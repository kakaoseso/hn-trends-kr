---
title: "Rust 컴파일러가 데드락을 증명해준다면? Hibana와 Affine MPST 찍먹해보기"
description: "Rust 컴파일러가 데드락을 증명해준다면? Hibana와 Affine MPST 찍먹해보기"
pubDate: "2026-02-11T22:26:11Z"
---

분산 시스템을 설계할 때 가장 골치 아픈 게 뭘까요? 네트워크 파티션? 레이턴시? 제 경험상 가장 끈질기게 괴롭히는 건 바로 **Protocol Drift(프로토콜 불일치)** 입니다.

화이트보드에 예쁘게 그려둔 시퀀스 다이어그램은 완벽하죠. 하지만 6개월 뒤, 클라이언트 팀은 v1.2를 구현하고 서버 팀은 v1.3 스펙을 보고 있다면? 혹은 에러 처리 분기 하나를 빼먹어서 특정 상태에서 시스템이 멈춰버린다면?

오늘 소개할 **Hibana** 는 이런 문제를 **컴파일 타임** 에 잡아내겠다는 야심 찬 Rust 프로젝트입니다. Hacker News에서 꽤 흥미로운 논의가 오가고 있는데, 단순히 "새로운 라이브러리"가 아니라 우리가 분산 시스템을 코딩하는 방식 자체를 건드리고 있습니다.

## MPST? Choreography? 그게 뭔데?

이 프로젝트의 핵심 키워드는 **Affine MPST(Multiparty Session Types)** 입니다. 이름만 들어도 벌써 논문 냄새가 나고 머리가 아프죠? 사실 저도 처음엔 그랬습니다.

하지만 HN 댓글에서도 지적되었듯, 이 기술의 본질은 "학술적 용어"가 아니라 **"단일 진실 공급원(Single Source of Truth)"** 에 있습니다. 우리가 흔히 IDL(Protobuf, gRPC)로 데이터의 *모양*을 정의한다면, Hibana는 통신의 *순서와 흐름*을 정의합니다.

### 코드로 보는 "Global Choreography"

Hibana에서는 전체 시스템의 동작(Choreography)을 하나의 코드로 작성합니다. 그리고 이를 각 참여자(Role)의 코드로 "투영(Projection)"해냅니다.

```rust
// 1. Global Choreography (전체 흐름 정의)
const PING_PONG: g::Program<_> = g::seq(
    g::send::<Client, Server, Ping>(), // 클라이언트가 서버에게 Ping 전송
    g::send::<Server, Client, Pong>(), // 서버가 클라이언트에게 Pong 응답
);

// 2. Compile-Time Projection (클라이언트 관점의 코드 자동 생성)
const CLIENT: g::RoleProgram<0, _> = g::project(&PING_PONG);
```

이게 왜 대단하냐고요? `CLIENT` 코드는 `PING_PONG`이라는 전역 정의에서 **컴파일 타임에** 도출됩니다. 즉, 내가 클라이언트 코드를 짜다가 실수로 `Ping`을 보내지 않거나, `Pong`을 기다리지 않고 종료해버리면 **컴파일러가 에러를 뱉습니다.**

## Affine Types: 실수는 허용되지 않는다

Rust의 소유권(Ownership) 모델은 이 개념과 찰떡궁합입니다. Hibana는 "Affine Cursors"라는 개념을 사용하는데, 쉽게 말해 프로토콜의 각 단계가 **"한 번만 사용되고 소비되는 자원"** 이라는 뜻입니다.

```rust
// 3. Affine Execution
// 컴파일러가 프로토콜 준수 여부를 강제함
let (client, _) = client.flow::<Ping>()?.send(&42u32).await?;
let (client, pong) = client.recv::<Pong>().await?;
```

여기서 `client` 객체는 상태가 변할 때마다 소비되고 새로운 상태의 객체로 반환됩니다. 만약 중간에 `recv`를 빼먹고 세션을 종료하려 하거나, 이미 보낸 상태를 재사용하려 하면 Rust 컴파일러가 막아섭니다. 런타임 에러가 아니라 컴파일 타임 에러로요. 이건 정말 강력합니다.

## Embedded First: `no_std`와 `no_alloc`

개인적으로 가장 놀라웠던 점은 이 라이브러리가 **Embedded First** 를 지향한다는 겁니다. 보통 이런 고수준 추상화는 런타임 오버헤드가 크거나 힙 메모리를 펑펑 쓰기 마련입니다.

하지만 Hibana는 `const fn`을 적극 활용해서 프로토콜 투영을 컴파일 타임에 끝내버립니다. 런타임에는 사실상 상태 머신을 돌리는 비용밖에 들지 않죠. `#![no_std]`와 `#![no_alloc]`을 지원한다는 건, 이 복잡한 프로토콜 검증 로직을 베어메탈 마이크로컨트롤러에서도 돌릴 수 있다는 뜻입니다. IoT 디바이스 간의 통신 프로토콜을 이렇게 엄격하게 관리할 수 있다면, 펌웨어 엔지니어들의 수명을 꽤나 늘려줄 수 있을 겁니다.

## Hacker News의 반응과 나의 생각

HN의 반응 중 가장 공감 갔던 건 *"기술적으로는 멋진데, 왜 이게 좋은지 더 쉽게 설명해달라"*는 피드백이었습니다. 개발자인 작성자(Author)도 이를 인정하며 **"분산 시스템에서의 프로토콜 드리프트(Drift) 방지"** 가 핵심 가치라고 정리했죠.

솔직히 제 의견을 보태자면, **아직은 시기상조(Preview) 단계** 입니다. 
- `hibana-quic` 데모가 있다고는 하지만, 실제 프로덕션 레벨의 복잡한 비즈니스 로직을 이 방식(Choreography)으로 모두 표현할 수 있을지는 검증이 필요합니다.
- Rust의 타입 시스템이 워낙 강력하다 보니, 프로토콜이 복잡해질수록 제네릭 타입 에러가 얼마나 끔찍하게 나올지 벌써부터 걱정되기도 합니다.

하지만 방향성은 확실히 매력적입니다. 우리는 언제까지 `if (msg.type == "PING")` 같은 코드를 수동으로 짜면서, 상대방이 스펙대로 보내주길 기도만 하고 있을 순 없으니까요.

## Verdict: 찍먹해볼 만한가?

**YES.** 특히 Rust를 사용해서 신뢰성이 중요한 통신 모듈을 개발하고 있다면 반드시 살펴봐야 할 패턴입니다. 당장 프로덕션에 넣지 않더라도, "타입 시스템으로 프로토콜 상태 머신을 제어한다"는 아이디어 자체만으로도 엔지니어링 시야를 넓혀줄 겁니다.

미래에는 우리가 API 문서를 작성하는 대신, 이런 `Choreography` 코드를 공유하고 각 언어별 컴파일러가 알아서 클라이언트/서버 스텁을 검증해 주는 세상이 오지 않을까요? Hibana가 그 시작점이 될 수도 있겠습니다.

---

**References:**
- [Hibana Official Site](https://hibanaworks.dev)
- [Hacker News Discussion](https://news.ycombinator.com/item?id=46913747)
