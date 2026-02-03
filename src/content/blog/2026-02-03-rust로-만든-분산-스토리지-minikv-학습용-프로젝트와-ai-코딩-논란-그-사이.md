---
title: "Rust로 만든 분산 스토리지 Minikv: 학습용 프로젝트와 AI 코딩 논란 그 사이"
description: "Rust로 만든 분산 스토리지 Minikv: 학습용 프로젝트와 AI 코딩 논란 그 사이"
pubDate: "2026-02-03T15:47:31Z"
---

분산 시스템(Distributed Systems)은 모든 백엔드 엔지니어들의 '최종 보스'와도 같습니다. 특히 요즘처럼 Rust가 인프라 스트럭처 언어의 표준으로 자리 잡으면서, 직접 KV Store나 DB를 밑바닥부터 구현해보는 것이 일종의 통과의례처럼 되었죠.

오늘은 최근 Hacker News에서 뜨거운 감자로 떠오른 **Minikv** 라는 프로젝트를 뜯어보려고 합니다. 기능 명세만 보면 엔터프라이즈급인데, 그 이면에 있는 구현 방식과 커뮤니티의 논란이 꽤나 흥미롭습니다. 시니어 엔지니어의 관점에서 냉정하게 분석해 보겠습니다.

## Minikv: 무엇을 만들었나?

Minikv는 Rust로 작성된 분산 키-값(Key-Value) 및 객체 스토리지입니다. 작성자는 이 프로젝트를 "Rust를 배운 지 82일 만에 시작한 학습용 프로젝트"라고 소개했지만, 스펙을 보면 단순한 토이 프로젝트 수준을 넘어서려 노력한 흔적이 보입니다.

주요 스펙은 다음과 같습니다.

- **Consistency:** Raft Consensus + 2PC(Two-Phase Commit)를 통한 강력한 일관성 보장
- **API:** gRPC, HTTP REST, 그리고 **S3 호환 API**
- **Sharding:** 256개의 Virtual Shard를 통한 데이터 분산
- **Durability:** WAL(Write-Ahead Log) 지원
- **Storage Engine:** In-memory, RocksDB, Sled 중 선택 가능

### 아키텍처 Deep Dive: Raft와 2PC의 조합?

여기서 제 눈을 사로잡은(혹은 의심하게 만든) 부분은 바로 **Raft와 2PC의 혼용** 입니다.

일반적으로 분산 스토리지에서 Raft는 메타데이터(Membership, Shard map 등) 관리에 주로 사용됩니다. 데이터 복제(Replication) 경로에 Raft를 직접 태우는 경우는 CockroachDB나 TiKV 같은 케이스가 있지만, 구현 난이도가 상당히 높고 오버헤드가 큽니다. 그런데 Minikv는 여기에 트랜잭션을 위해 2PC까지 얹었습니다.

Hacker News의 한 유저는 이렇게 지적했습니다.

> "데이터 샤딩을 하고 크로스 샤드 트랜잭션을 하는 게 아니라면, 왜 Raft와 2PC를 같이 쓰나요? 단순히 데이터 복제를 위해서라면 이는 과도한 오버헤드(Overhead)입니다."

저도 이 의견에 동의합니다. 보통 데이터 복제는 Chain Replication이나 Primary-Backup 방식을 사용해 Latency를 줄이는 것이 일반적입니다. 모든 쓰기 작업에 Raft 합의를 거치고 2PC까지 수행한다면, **Write Throughput** 과 **Latency** 에서 상당한 손해를 볼 수밖에 없습니다. 학습 목적이라면 훌륭한 시도지만, 프로덕션 관점에서는 "굳이?"라는 의문이 드는 아키텍처입니다.

## 논란의 중심: "100% Hand-written" vs "AI Generated"

사실 기술적인 부분보다 더 뜨거웠던 건 **"이걸 정말 혼자 짰느냐"** 에 대한 논쟁이었습니다.

작성자는 "모든 코드를 100% 손으로 짰다"고 주장했지만, 공개된 Git Log가 발목을 잡았습니다. 커밋 로그를 보면, 복잡한 모듈들이 **30초 간격** 으로 완성되어 커밋된 기록이 발견되었습니다. 해싱 유틸리티, 설정 구조체, 에러 타입 정의 등이 분 단위로 쏟아져 나왔죠.

- **작성자의 주장:** "나는 집중하면 30초마다 커밋한다. CI가 통과될 때까지 계속 수정해서 올리는 스타일이다."
- **커뮤니티의 반응:** "전체 프로젝트 구조를 알고 있는 상태에서 AI가 생성한 코드를 붙여넣기 한 것이 명백하다. `fix_everything.sh` 같은 커밋이 중간에 섞여 있는 것도 수상하다."

솔직히 말해서, 시니어 엔지니어로서 코드를 리뷰할 때 가장 경계하는 것이 **"이해하지 못하고 복붙한 코드"** 입니다. Rust는 컴파일러가 엄격해서 아무 코드나 갖다 붙인다고 돌아가지 않습니다. 만약 AI의 도움을 받았다면, 차라리 "Copilot과 함께 Pair Programming을 했다"고 솔직하게 말했으면 더 좋았을 겁니다. 학습용 프로젝트에서 AI를 쓰는 건 죄가 아니지만, 그것을 "순수 인간의 노력"으로 포장하려다 신뢰를 잃은 케이스라 아쉽습니다.

## 그래도 배울 점은 있다

논란을 차치하고 코드 자체만 보면, Rust로 분산 시스템을 구축할 때 필요한 요소들을 종합 선물 세트처럼 모아두긴 했습니다.

```rust
// config.example.toml 예시
[server]
id = 1
addr = "127.0.0.1:8080"

[raft]
heartbeat_interval = 100
election_timeout = 1000

[storage]
engine = "rocksdb" # or "memory", "sled"
path = "./data"
```

특히 **Pluggable Storage** 구조나 **Virtual Sharding** 을 구현한 방식은 Rust 입문자들이 참고하기에 나쁘지 않습니다. 또한 S3 API를 흉내 내어 구현해 본 것은, 실제 현업에서 Object Store가 어떻게 동작하는지 이해하는 데 큰 도움이 되었을 겁니다.

하지만 성능 지표(단일 노드 인메모리 50k ops/sec)는 Rust의 성능을 극한으로 끌어올렸다고 보기엔 다소 평범합니다. 잘 튜닝된 Redis나 다른 Rust 기반 스토리지(예: Garage, Dragonfly)들은 이보다 훨씬 높은 처리량을 보여줍니다.

## 결론: Production Ready? No. Educational? Yes.

Minikv는 **"Rust로 분산 시스템 맛보기"** 에는 적합할지 몰라도, 아키텍처의 효율성이나 프로젝트의 투명성 측면에서 프로덕션 도입을 논하기엔 시기상조입니다.

**제 평가는요:**
1.  **기술적 시도:** Raft와 2PC를 직접 구현해 본 깡(?)은 높이 삽니다. 하지만 실무에서는 etcd나 ZooKeeper를 쓰거나, 데이터 패스에서 합의 알고리즘을 덜어내는 설계를 고민해야 합니다.
2.  **AI 논란:** 30초마다 완벽한 모듈을 짜내는 개발자는 없습니다. 우리는 결과물뿐만 아니라 그 과정의 정직함도 엔지니어링의 일부로 봅니다.
3.  **대안:** Rust로 작성된 S3 호환 Object Store에 관심이 있다면, 이미 오픈소스로 잘 관리되고 있는 **Garage** 프로젝트를 먼저 살펴보는 것을 추천합니다.

엔지니어링은 단순히 코드를 돌아가게 만드는 것이 아니라, **트레이드오프(Trade-off)** 를 관리하고 **신뢰** 를 쌓아가는 과정임을 다시 한번 상기하게 되는 프로젝트였습니다.

---

**References:**
- [Minikv GitHub Repository](https://github.com/whispem/minikv)
- [Hacker News Discussion](https://news.ycombinator.com/item?id=46867947)
