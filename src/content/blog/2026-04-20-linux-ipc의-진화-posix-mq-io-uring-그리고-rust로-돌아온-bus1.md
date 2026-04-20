---
title: "Linux IPC의 진화: POSIX MQ, io_uring, 그리고 Rust로 돌아온 bus1"
description: "Linux IPC의 진화: POSIX MQ, io_uring, 그리고 Rust로 돌아온 bus1"
pubDate: "2026-04-20T16:57:09Z"
---

시스템 프로그래밍이나 백엔드 아키텍처를 깊게 파고들다 보면, 결국 마주치는 거대한 벽이 하나 있다. 바로 IPC(Inter-Process Communication)다. 마이크로서비스 아키텍처가 대세가 되면서 다들 네트워크 기반의 gRPC나 REST에 익숙해졌지만, 단일 노드 내에서 극한의 Latency와 Throughput을 뽑아내야 하는 상황에서는 여전히 커널 레벨의 IPC 메커니즘이 필수적이다.

최근 LWN에 흥미로운 기사가 하나 올라왔다. Linux 커널 메일링 리스트(LKML)에서 논의 중인 세 가지 IPC 관련 패치에 대한 내용이다. 15년 넘게 현업에서 온갖 시스템 병목과 싸워온 엔지니어 입장에서, 이 세 가지 제안은 현재 Linux 커널 커뮤니티가 겪고 있는 고민과 기술적 트렌드를 아주 적나라하게 보여준다.

## POSIX Message Queue의 생명 연장: mq_timedreceive2

솔직히 말해서, 요즘 신규 프로젝트에서 POSIX Message Queue를 주력 IPC로 선택하는 미친(?) 엔지니어는 거의 없을 것이다. 대부분 Redis, Kafka를 쓰거나 로컬에서는 Unix Domain Socket을 쓴다. 하지만 레거시 시스템이나 특정 실시간(RT) 요구사항이 있는 임베디드 환경에서는 여전히 무시할 수 없는 존재다.

이번에 Mathura Kumar가 제안한 패치는 기존 `mq_timedreceive()` 시스템 콜의 한계를 극복하기 위해 `mq_timedreceive2()`를 추가하는 내용이다.

```c
struct mq_timedreceive2_args {
    size_t msg_len;
    unsigned int *msg_prio;
    char *msg_ptr;
};

ssize_t mq_timedreceive2(mqd_t mqdes, struct mq_timedreceive2_args *uargs, unsigned int flags, unsigned long index, const struct timespec *abs_timeout);
```

핵심은 시스템 콜의 인자 개수 제한(아키텍처에 따라 보통 6개)을 우회하기 위해, 일부 인자들을 `struct mq_timedreceive2_args` 구조체로 묶어서 포인터로 넘겼다는 점이다. 이를 통해 확보한 슬롯으로 `flags`와 `index`를 추가했다.

새로 추가된 `MQ_PEEK` 플래그를 사용하면 큐에서 메시지를 소비하지 않고 읽기만 할 수 있다. 또한 `index`를 통해 큐의 맨 앞(우선순위가 가장 높은) 메시지가 아닌, 더 뒤쪽에 있는 메시지에도 접근할 수 있다.

- **나의 생각:** 이 패턴은 과거 다른 시스템 콜들이 진화해 온 전형적인 방식이다. 구조체 포인터를 넘기는 방식은 향후 확장성을 위해 매우 현명한 선택이다. 특히 CRIU(Checkpoint/Restore In Userspace) 같은 툴에서 큐의 상태를 덤프 뜰 때 파괴적인 읽기(destructive read)를 하지 않아도 된다는 점은 매우 실용적이다. 다만, 커뮤니티의 관심도가 너무 낮아서 메인라인에 머지되기까지는 꽤 오랜 시간이 걸릴 것 같다.

## io_uring 기반 IPC: 야심차지만 험난한 길

개인적으로 가장 눈길이 갔던 부분이다. `io_uring`은 이제 단순한 비동기 I/O 인터페이스를 넘어, Linux 커널의 비동기 실행 엔진 그 자체가 되어가고 있다. Daniel Hodges는 이 `io_uring` 위에 D-Bus 스타일의 고대역폭 IPC 채널을 올리는 패치를 제안했다.

Shared ring buffer를 사용해 프로세스 간 통신 시 데이터 복사를 최소화(Zero-copy 지향)하는 것이 목표다. `IORING_REGISTER_IPC_CHANNEL_CREATE`로 채널을 열고, `IORING_OP_IPC_SEND`와 `IORING_OP_IPC_RECV`로 통신하는 구조다.

- **나의 생각:** 아이디어 자체는 훌륭하다. 하지만 과거 `kdbus`가 커널에 들어가려다 복잡성과 보안 문제로 처참하게 실패했던 슬픈 역사가 떠오른다. 커널 내부에 복잡한 IPC 브로커를 두는 것은 언제나 논란의 대상이다.

게다가 Jens Axboe(io_uring 메인테이너)가 이 패치가 LLM의 도움을 받아 작성된 것을 눈치챘고, 작성자도 이를 인정했다는 점이 커뮤니티에서 꽤 화제가 되었다. LLM이 보일러플레이트를 짜주는 건 좋지만, 커널 코어 서브시스템의 IPC 메커니즘은 Credential management 같은 극도로 예민한 보안 이슈를 다뤄야 한다. Axboe가 방향성에는 동의했지만, 프로덕션 레벨로 끌어올리려면 작성자가 뼈를 깎는 리팩토링과 엣지 케이스 처리를 해야 할 것이다. 당장 써먹을 수 있는 물건은 아니다.

## 10년 만에 Rust로 부활한 bus1

2016년에 David Rheinsberg가 제안했던 `bus1`을 기억하는 시니어들이 있을 것이다. D-Bus의 성능 문제를 해결하겠다며 커널 내부에 Capability 기반의 IPC를 구현하려 했던 시도였으나, 결국 메인라인에 들어가지 못하고 잊혀졌다.

그런데 10년이 지난 지금, 이 `bus1`이 **Rust** 로 재작성되어 돌아왔다. 작성자의 코멘트가 예술이다.

> "가장 큰 변화는 모든 것을 기본으로 깎아내고 모듈을 Rust로 재구현했다는 것입니다. Refcount 소유권과 객체 수명에 대해 걱정하지 않아도 된다는 것은 정말 큰 기쁨입니다. (후략)"

- **나의 생각:** C언어로 복잡한 커널 객체의 생명주기를 관리하는 것은 지옥 그 자체다. Use-after-free 버그 하나 잡으려고 밤새워 커널 덤프를 까본 사람이라면 누구나 공감할 것이다. Rust의 오너십 모델이 커널 내 복잡한 IPC 상태 관리에 얼마나 적합한지 보여주는 완벽한 유즈케이스라고 본다.

물론 C와 Rust 사이의 FFI 브릿지 오버헤드가 성능에 어떤 영향을 미칠지가 관건이다. 하지만 현재 Rust for Linux 커뮤니티가 이 패치를 통해 커널 내 Rust 도입의 타당성을 증명하려 할 것이기 때문에, 최적화는 빠르게 이루어질 것으로 기대한다.

## 결론 및 요약

이번 LWN 기사를 통해 본 Linux IPC의 현주소는 다음과 같다.

1. **POSIX MQ:** 레거시와 엣지 케이스를 위한 생명 연장 중.
2. **io_uring IPC:** 성능 잠재력은 최고지만, 커널 내 복잡도 증가와 보안(Credential) 문제라는 큰 산을 넘어야 함.
3. **bus1 (Rust):** 커널 레벨 Rust의 강력함을 증명할 시험대. 가장 기대되는 프로젝트.

아직 세 가지 모두 당장 내일 프로덕션에 적용할 수 있는 단계는 아니다. 하지만 시스템의 한계를 쥐어짜내야 하는 엔지니어라면, `io_uring` IPC의 발전 과정과 `bus1`의 Rust 기반 성능 벤치마크는 반드시 팔로우업 할 가치가 있다.

**References**
- Original Article: https://lwn.net/Articles/1065490/
- Hacker News Thread: https://news.ycombinator.com/item?id=47801923
