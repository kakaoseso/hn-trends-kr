---
title: "ClickHouse의 새로운 Fiber 스케줄러 Silk 리뷰: 스레드 당 커넥션 모델의 한계를 넘어서"
description: "ClickHouse의 새로운 Fiber 스케줄러 Silk 리뷰: 스레드 당 커넥션 모델의 한계를 넘어서"
pubDate: "2026-05-24T12:02:49Z"
---

15년 넘게 백엔드 엔지니어로 구르면서 가장 골치 아팠던 문제 중 하나는 C++ 서버의 동시성 모델을 결정하는 것이었습니다. 보통 Thread-per-connection 모델로 시작하죠. 구현이 직관적이니까요. 하지만 동시 접속자가 1,000명을 넘어가는 순간 OS의 컨텍스트 스위칭 오버헤드에 시스템이 비명을 지르기 시작합니다. 이를 해결하려고 epoll 기반의 Event-driven 아키텍처나 C++20의 Stackless 코루틴을 도입해보지만, 콜백 지옥이나 Function color 문제로 코드베이스가 엉망이 되곤 합니다.

최근 ClickHouse에서 흥미로운 오픈소스를 공개했습니다. 바로 Linux용 Cooperative fiber 스케줄러인 Silk입니다. ClickHouse가 왜 갑자기 자체 스케줄러를 만들었을까요?

## Silk의 핵심 아키텍처

Silk는 철저하게 성능과 확장성에 초점을 맞춘 스케줄러입니다. 문서를 살펴보면서 가장 인상 깊었던 기술적 특징들은 다음과 같습니다.

- **Stackful Coroutine:** Silk는 경량 Stackful 코루틴을 사용합니다. OS 스레드를 블로킹하는 대신 Fiber 단위에서 Suspend를 수행하죠. C++20의 Stackless 코루틴과 달리 기존의 동기식 코드를 거의 수정 없이 비동기처럼 돌릴 수 있다는 게 엄청난 장점입니다.
- **io_uring 통합:** 최신 Linux 비동기 I/O의 표준이 된 io_uring을 깊게 통합했습니다. 블로킹 I/O를 완전히 배제하고 진정한 의미의 비동기 런타임을 구현했습니다.
- **Work-stealing:** Per-CPU 스케줄러 스레드 기반에 Topology-aware work-stealing 알고리즘을 적용했습니다. CPU 캐시 친화성을 유지하면서 유휴 코어에 작업을 분산시키는, 성능의 극한을 쥐어짜는 설계입니다.

단순히 스케줄러만 던져주는 것이 아니라, 실무에서 당장 필요한 동기화 원시 타입들(FiberFuture, FiberMutex 등)과 Lock-free 자료구조까지 유틸리티로 묶어 제공합니다.

## 개발 및 디버깅 경험에 대한 집착

엔지니어로서 가장 마음에 들었던 부분은 이들이 툴링에 얼마나 진심인지 보여주는 대목입니다.

- **Perf Tools:** net-perf, file-perf, s3-perf, http-perf 등 인프라의 각 계층을 테스트할 수 있는 벤치마크 툴을 내장했습니다. 심지어 `--flamegraph` 플래그 하나로 Flamegraph SVG를 뽑아낼 수 있습니다.
- **GDB Extension:** Fiber를 디버깅하는 건 지옥 같은 일입니다. Silk는 전용 GDB 익스텐션을 제공하여 `fiber-list`나 `fiber-switchcontext` 같은 명령어로 컨텍스트 스위칭 상태를 직접 추적할 수 있게 해줍니다.

## Hacker News 커뮤니티의 반응과 개인적인 생각

Hacker News에서도 꽤 흥미로운 토론이 오갔습니다. 몇 가지 주요 쟁점과 제 생각을 덧붙여보겠습니다.

첫째, ScyllaDB의 Seastar와의 비교입니다. Seastar의 Share-nothing 아키텍처는 극단적인 성능을 내지만 러닝 커브가 너무 가파르고 모든 코드를 다시 작성해야 합니다. 반면 Silk는 Work-stealing을 지원하는 Fiber 모델이라 기존 멀티스레드 코드베이스를 포팅하기가 훨씬 수월합니다. 실용성 면에서는 Silk의 손을 들어주고 싶습니다.

둘째, ClickHouse의 근본적인 아키텍처 개선입니다. 한 유저가 정확히 짚었듯, ClickHouse는 그동안 Thread-per-connection 모델의 한계 때문에 대량의 Async INSERT 처리에 병목이 있었습니다. Silk는 이 네트워크와 I/O 병목을 해결하기 위한 ClickHouse의 전략적 무기임이 분명합니다.

셋째, 예외 처리(Exception safety)에 대한 우려입니다. Task switching 중 Unwind가 발생할 때 안전하지 않을 수 있다는 지적이 있었습니다. C++에서 Fiber를 다뤄본 분들이라면 이 골칫거리에 깊이 공감하실 겁니다. 코루틴 경계를 넘나드는 예외 처리는 여전히 C++ 생태계에서 완벽하게 풀리지 않은 숙제입니다.

## 총평: 프로덕션에 도입할 만한가?

솔직히 말해서, 당장 내일 여러분의 프로젝트에 Silk를 Drop-in replacement로 넣으라고 권하진 않겠습니다. Clang 21을 요구하고 Boost.Context를 벤더링해서 쓰는 등 빌드 제약사항이 꽤 빡빡하기 때문입니다.

하지만 Silk는 장난감이 아닙니다. ClickHouse라는 거인의 어깨 위에서 탄생한 만큼, 극한의 I/O 성능이 필요한 시스템에서는 훌륭한 레퍼런스가 될 것입니다. 새로운 고성능 C++ 네트워크 서버를 바닥부터 설계해야 하거나, 기존의 무거운 스레드 모델에 한계를 느끼고 있다면 Silk의 코드를 한 번 깊게 분석해 보시길 강력히 추천합니다.

## References
- **GitHub:** https://github.com/ClickHouse/silk
- **Hacker News:** https://news.ycombinator.com/item?id=48210937
