---
title: "순수 AArch64 어셈블리로 웹 서버를 짰다고? ymawky 리뷰와 어셈블리 최적화의 현실"
description: "순수 AArch64 어셈블리로 웹 서버를 짰다고? ymawky 리뷰와 어셈블리 최적화의 현실"
pubDate: "2026-05-11T23:51:37Z"
---

요새 다들 Rust나 Go로 서버를 짜고, Kubernetes 위에서 마이크로서비스를 굴리느라 정신이 없습니다. Nginx나 Envoy 같은 훌륭한 웹 서버와 프록시가 널려있는 2025년에, 누군가 "내 인생의 무의미함을 달래기 위해" 순수 AArch64 어셈블리로 웹 서버를 만들었다고 하면 믿으시겠습니까?

이 글을 처음 접했을 때 저는 경악과 동시에 깊은 향수(?)를 느꼈습니다. 오늘은 [ymawky](https://imtomt.github.io/ymawky/)라는 이 미친 프로젝트를 해부해보고, Hacker News에서 벌어진 흥미로운 토론들을 바탕으로 시니어 엔지니어 관점에서의 인사이트를 나눠보겠습니다.

## 제약 조건: 타협 없는 하드코어

작성자는 이 프로젝트에 몇 가지 변태적인 제약 조건을 걸었습니다.

- **언어:** 오직 AArch64 어셈블리
- **OS:** macOS (Darwin)
- **의존성:** libc 래퍼 없이 Raw Syscall만 사용
- **라이브러리:** 외부 라이브러리 절대 금지 (사전 구현된 파서 없음)

C 언어조차 사치라고 생각한 이 접근법은, 우리가 평소에 얼마나 많은 Abstraction에 기대어 개발하고 있는지 뼈저리게 느끼게 해줍니다.

## Raw Syscall과 에러 처리의 민낯

보통 파일 하나를 열 때 우리는 `open()` 함수를 호출합니다. 하지만 libc가 없다면 커널과 직접 대화해야 합니다. Darwin에서 파일을 여는 코드는 다음과 같습니다.

```assembly
mov x16, #5 ; SYS_open syscall number
adrp x0, filename@PAGE
add x0, x0, filename@PAGEOFF
mov x1, #0x0 ; O_RDONLY is just 0x0000
svc #0x80
b.cs open_failed
```

Darwin 커널에서는 `x16` 레지스터에 syscall 번호(여기서는 5번)를 넣고, `x0`, `x1` 등에 인자를 세팅한 뒤 `svc #0x80`으로 커널에 제어권을 넘깁니다.

여기서 재미있는 부분은 에러 처리입니다. Exception이나 `try-catch` 같은 건 당연히 없습니다. 커널 호출이 실패하면 CPU의 Carry flag가 세팅되고, 우리는 `b.cs`(Branch if Carry Set) 명령어로 이를 캐치해서 수동으로 클린업 코드로 점프해야 합니다. 우아함과는 거리가 멀지만, CPU가 실제로 어떻게 동작하는지 이보다 투명하게 보여줄 순 없죠.

## 아키텍처: 1996년으로의 회귀

이 서버의 요청 처리 모델은 Nginx의 Event-driven async 모델이 아닙니다. 요청이 들어올 때마다 `fork()`를 때리는 **Fork-on-request** 방식을 사용합니다.

장점은 명확합니다. 프로세스 간 메모리가 격리되므로 상태 관리가 쉽고 코드가 단순해집니다. 하지만 단점은 치명적입니다. Context switching 오버헤드가 극심하고, 동시 접속자가 늘어날수록 메모리 Bloat이 발생합니다. HN의 한 유저가 지적했듯, 이건 사실상 1996년도 State-of-the-art 수준의 아키텍처입니다.

성능 최적화를 위해 어셈블리를 선택해놓고 아키텍처는 가장 무거운 모델을 택했다는 점이 아이러니하지만, 애초에 이 프로젝트의 목적이 Throughput 극대화가 아니었음을 감안하면 이해할 수 있는 트레이드오프입니다.

## 진짜 광기: 어셈블리로 HTTP 파싱하기

개인적으로 이 프로젝트에서 가장 경악스러웠던 부분은 문자열 파싱입니다. HTTP는 근본적으로 텍스트 프로토콜입니다. 파이썬이었다면 `text.split("GET ")[1].split(" ")[0]` 한 줄로 끝났을 URL 추출을 위해 작성자는 바이트 단위로 포인터를 옮기며 루프를 돕니다.

```assembly
streqn:
    ldrb w3, [x0]
    ldrb w4, [x1]
    cmp w3, w4
    b.ne Lstreqn_no_match
    cbz w3, Lstreqn_match ;; both equal and both NULL = end of string = match
    ;; if we've reached the end, it's a match yeah?
    subs x2, x2, #1
    b.eq Lstreqn_match
    add x0, x0, #1
    add x1, x1, #1
    b streqn
Lstreqn_match:
    mov x0, #1
    ret
Lstreqn_no_match:
    mov x0, #0
    ret
```

단순한 `strcmp`조차 직접 구현해야 합니다. 여기에 더해 URL Percent-decoding, `Range` 헤더 처리를 위한 커스텀 `atoi` (Integer overflow 방지 포함)까지 바닥부터 짰습니다. 개발자가 "어셈블리에서는 문자열 파싱을 혐오하게 된다"고 푸념한 것이 100% 공감되는 대목입니다.



## 엣지 케이스와 보안에 대한 고찰

이런 장난감(?) 프로젝트임에도 불구하고 시스템 프로그래밍의 엣지 케이스들을 꽤 진지하게 처리했습니다.

- **PUT 요청 안전성:** 파일 쓰기 도중 프로세스가 죽거나 연결이 끊길 경우를 대비해 `.ymawky_tmp_<pid>` 같은 임시 파일에 먼저 쓰고, 성공적으로 완료되었을 때만 `rename`을 수행합니다.
- **Slowloris 방어:** Content-Length와 최소 전송 속도(기본 16KB/s)를 기반으로 동적 Timeout을 계산하여 리소스 고갈 공격을 방어합니다.
- **Path Traversal 방어:** 단순히 `..`을 막는 것이 아니라, Percent-decoding 이후에 `..` 세그먼트를 검사하여 `/etc/shadow` 같은 시스템 파일 접근을 차단합니다. O_NOFOLLOW 플래그를 활용해 심볼릭 링크 공격도 대비했습니다.

## Hacker News 토론: 인간 vs 컴파일러

이 글이 HN에 올라온 후, 흥미로운 주제로 불판이 열렸습니다. **"과연 현대의 프로그래머가 어셈블리를 직접 짜면 최신 컴파일러(GCC, Clang)를 이길 수 있는가?"**

결론부터 말하자면, 일반적인 비즈니스 로직이나 IO 바운드 애플리케이션에서는 **절대 불가능** 합니다. 컴파일러는 우리가 상상하는 것 이상으로 영리하며, 레지스터 할당과 파이프라인 최적화에 있어서는 인간을 아득히 초월했습니다.

하지만 HN의 몇몇 시니어 시스템 프로그래머들이 지적한 예외 케이스들은 주목할 만합니다.

1. **Data Layout과 Vectorization:** 컴파일러는 데이터의 메모리 레이아웃을 스스로 뜯어고치지 못합니다. AVX-512 같은 벡터 인스트럭션을 극한으로 활용해야 하는 비디오 코덱(x264, ffmpeg 등)이나 암호화 알고리즘에서는 여전히 수제 어셈블리가 압도적인 성능을 냅니다.
2. **Interpreter Dispatch:** 거대한 Switch-case 기반의 인터프리터 루프에서는 Clang의 레지스터 할당이 바보같이 동작할 때가 있습니다. 이런 극단적인 분기 예측 환경에서는 인간이 직접 컨트롤 플로우를 짜는 것이 10~20%의 성능 향상을 가져오기도 합니다.

결국 어셈블리는 마법의 성능 향상 도구가 아닙니다. 병목이 CPU의 특정 인스트럭션 사이클에 있을 때만 꺼내 드는 마지막 수단이죠. 웹 서버처럼 IO와 네트워크 패킷 대기가 성능의 99%를 결정하는 도메인에서는, 어셈블리 최적화보다 Non-blocking IO 아키텍처 도입이 수백 배 더 중요합니다.

## Verdict: 그래서 이걸 왜 했는가?



작성자 스스로도 이 서버가 Nginx를 대체할 수 없다는 걸 잘 알고 있습니다. 하지만 이 프로젝트의 진정한 가치는 **교육과 통찰** 에 있습니다.

우리는 매일 파이썬, Go, Node.js로 수십 줄짜리 웹 서버를 띄우며, 그 아래에서 수천 줄의 C 코드와 OS 커널이 어떤 땀을 흘리고 있는지 잊고 삽니다. 소켓 바인딩, 문자열 인코딩, 버퍼 오버플로우 방지, 프로세스 관리 등 프레임워크가 공짜로 해주던 일들을 직접 마주하는 경험은 엔지니어의 시야를 완전히 바꿔놓습니다.

프로덕션에 당장 쓸 코드는 아니지만, 주말에 한 번쯤 이런 하드코어한 삽질을 해보는 것을 강력히 추천합니다. 아마 월요일 아침에 출근해서 IDE를 켜고 표준 라이브러리를 임포트할 때, 여러분은 이전과는 다른 깊은 감사함을 느끼게 될 것입니다.

---
**References:**
- 원문 블로그: [Building a web server in aarch64 assembly to give my life (a lack of) meaning](https://imtomt.github.io/ymawky/)
- Hacker News 스레드: [https://news.ycombinator.com/item?id=48062977](https://news.ycombinator.com/item?id=48062977)
