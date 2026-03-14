---
title: "32GB OOM의 악몽을 끝내다: Lock-Free의 한계와 Kovan의 Wait-Free 메모리 관리"
description: "32GB OOM의 악몽을 끝내다: Lock-Free의 한계와 Kovan의 Wait-Free 메모리 관리"
pubDate: "2026-03-14T02:38:23Z"
---

새벽 3시, 잘 돌아가던 서버의 힙 메모리가 갑자기 32GB를 돌파하며 OOM(Out of Memory)으로 죽어버린 경험이 있는가? 고성능 동시성 시스템을 운영해본 엔지니어라면 한 번쯤 겪어봤을 악몽이다.

Rust 생태계에서 Lock-Free 자료구조를 다룰 때 대부분의 개발자는 자연스럽게 `crossbeam-epoch`을 선택한다. 훌륭한 라이브러리이고, 빠르며, 검증된 기본값이다. 하지만 여기에 치명적인 함정이 있다. 바로 **Lock-Free** 이지 **Wait-Free** 가 아니라는 점이다.

최근 Hacker News에 올라온 Kovan: From Production MVCC Systems to Wait-Free Memory Reclamation 글은 이 문제를 정면으로 다루며, 시스템 엔지니어들이 왜 Wait-Free 메모리 회수(Memory Reclamation)에 주목해야 하는지 아주 날카롭게 짚어냈다. 15년 넘게 백엔드와 데이터베이스 코어 시스템을 만져온 입장에서, 이 글은 근래 읽은 기술 블로그 중 단연 최고다.

### Lock-Free의 배신: Epoch-based Reclamation의 한계

Lock-Free 시스템은 전체 시스템이 진행(Progress)됨을 보장하지만, 개별 스레드의 기아(Starvation) 상태까지 막아주진 않는다. `crossbeam`이 사용하는 Epoch 기반 메모리 회수 방식의 가장 큰 문제는, 단 하나의 스레드라도 지연되면 전체 프로세스의 메모리 회수가 블로킹된다는 것이다.

데이터베이스를 예로 들어보자. OLTP 워크로드 중간에 500ms 정도 걸리는 긴 분석용 스캔 쿼리가 들어왔다고 가정하자. 이 스레드가 Epoch를 잡고 있는 동안, 다른 모든 스레드에서 발생하는 메모리 해제(Retirement)는 실제로 OS에 반환되지 못하고 쌓이게 된다. 쓰기 작업이 많을수록 쓰레기 객체들은 걷잡을 수 없이 증식하고, 결국 OOM을 유발한다. 개발자들은 보통 자신의 라이프타임 관리나 메모리 할당 로직을 의심하며 밤을 새우지만, 사실 이건 당신 잘못이 아니다. 의존하고 있는 라이브러리의 구조적 한계다.

### Wait-Free의 등장: Kovan과 Crystalline

저자인 vertexclique가 내놓은 해답은 **Wait-Free** 다. Wait-Free는 Lock-Free의 상위 호환으로, 다른 스레드의 상태와 무관하게 모든 연산이 제한된 단계(Bounded steps) 내에 완료됨을 보장한다. 스케줄러의 공정성에 의존하지 않으며, 메모리 누적도 없다.

Kovan은 2021년 DISC에 발표된 "Crystalline: Fast and Memory Efficient Wait-Free Reclamation" 논문을 바탕으로 구현된 Rust 라이브러리다. 논문을 실제 프로덕션 수준의 ARM64/x86-64 코드로 옮기는 것은 Memory Ordering이나 고경합 상황의 ABA 문제를 해결해야 하는 극도로 고된 작업인데, 이를 해냈다는 점이 인상적이다.

Kovan의 핵심 설계는 다음과 같다.

- **Slot-based architecture:** 스레드 단위의 구조 대신 슬롯 기반 아키텍처를 사용하여 Wait-Free의 바운드를 관리 가능하게 만듦.
- **Batch retirement:** 연산 전반에 걸쳐 회수 비용을 상각(Amortize)하기 위한 일괄 처리.
- **no_std 지원:** 임베디드 환경에서도 사용 가능.

벤치마크 결과도 흥미롭다. 데이터베이스처럼 읽기 작업이 압도적으로 많은(Read-heavy) 워크로드에서 Kovan은 `crossbeam-epoch` 대비 1.3~1.4배 빠른 성능을 보여준다. 읽기 오버헤드가 단일 Atomic Load 하나로 끝난다는 점이 이 차이를 만든다.

### 개인적인 시선: TLA+와 프로덕션의 무게

이 프로젝트에서 내가 가장 높게 평가하는 부분은 단순히 코드를 짠 것을 넘어 **TLA+** 를 이용한 정형 검증(Formal Verification)을 거쳤다는 점이다.

동시성 버그는 테스트 코드를 아무리 돌려도 프로덕션의 기괴한 타이밍과 인터리빙(Interleaving) 상황에서 반드시 터진다. Kovan은 Leslie Lamport의 TLA+를 이용해 모델 체커로 도달 가능한 모든 상태를 탐색했다. 구조적 정합성, Use-after-free 방지(Safety), 그리고 제한된 메모리 보장(Liveness)을 수학적으로 증명한 것이다. 인프라 코어 시스템을 만드는 사람이라면 이 접근 방식이 얼마나 든든한지 알 것이다.

API 설계도 영리하다. 기존 `crossbeam-epoch`과 의도적으로 유사하게 만들어 마이그레이션 비용을 낮췄다.

```rust
// Kovan API — crossbeam-epoch과 매우 유사함
use kovan::{pin, retire, Atomic, Shared};

let atomic = Atomic::new(Box::into_raw(Box::new(42)));
let guard = pin();

// 단일 Atomic Load로 제로 오버헤드 읽기
let ptr = atomic.load(Ordering::Acquire, &guard);

atomic.store(Shared::from(new_ptr), Ordering::Release);
retire(old_ptr.as_raw()); // Wait-free 메모리 회수
```

### 총평 (Verdict)

Hacker News에서는 종종 AI의 화려함에 가려져 이런 하드코어 시스템 엔지니어링 주제가 묻히곤 하지만, 진짜 돈을 벌어다 주고 시스템을 지탱하는 것은 결국 이런 근본적인 기술이다.

클라우드 서비스의 SLA, 금융 트레이딩의 Tail Latency, 혹은 분산 데이터베이스를 다루고 있다면 Epoch 지연으로 인한 OOM이나 꼬리 지연 시간은 치명적이다.

Kovan은 단순한 장난감이 아니다. 저자 본인도 SpireDB라는 프로덕션 환경에서 이를 돌리고 있으며, TLA+ 검증까지 마쳤다. Rust로 고성능 동시성 시스템을 구축하고 있거나 `crossbeam-epoch`의 메모리 스파이크에 지쳤다면, 당장 도입을 검토해 볼 가치가 충분하다. 오랜만에 가슴이 뛰는 오픈소스를 만났다.

### References
- **Original Article:** [Kovan: From Production MVCC Systems to Wait-Free Memory Reclamation](https://vertexclique.com/blog/kovan-from-prod-to-mr/)
- **Hacker News Thread:** [https://news.ycombinator.com/item?id=47321850](https://news.ycombinator.com/item?id=47321850)
