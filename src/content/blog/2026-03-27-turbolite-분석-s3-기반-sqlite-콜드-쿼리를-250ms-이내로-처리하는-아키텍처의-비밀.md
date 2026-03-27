---
title: "Turbolite 분석: S3 기반 SQLite 콜드 쿼리를 250ms 이내로 처리하는 아키텍처의 비밀"
description: "Turbolite 분석: S3 기반 SQLite 콜드 쿼리를 250ms 이내로 처리하는 아키텍처의 비밀"
pubDate: "2026-03-27T09:08:30Z"
---

최근 멀티 테넌트 아키텍처나 Agentic AI 애플리케이션을 설계하다 보면 "테넌트당 하나의 SQLite DB"라는 아이디어에 매료되곤 합니다. 수천 개의 DB 파일을 영구 볼륨(Persistent Volume)에 주렁주렁 매달아 관리하는 건 악몽에 가깝죠. 그래서 자연스럽게 S3 같은 Object Storage에 DB를 올려두고 필요할 때만 쿼리하는 방식을 상상하게 됩니다.

하지만 과거에는 네트워크 Latency 때문에 이런 접근이 사실상 불가능했습니다. 그런데 S3 Express One Zone이나 Tigris 같은 초저지연 스토리지가 등장하면서 판도가 바뀌고 있습니다.

이번에 해커뉴스(HN)에 올라온 **Turbolite** 는 바로 이 지점을 파고든 프로젝트입니다. Rust로 작성된 SQLite VFS(Virtual File System)로, S3에서 직접 콜드 JOIN 쿼리를 250ms 이하로 서빙하는 것을 목표로 합니다.

단순한 장난감 프로젝트가 아닙니다. 아키텍처를 뜯어보면 분산 시스템과 스토리지 엔진에 대한 깊은 이해가 돋보입니다. 15년 차 엔지니어의 시각에서 이 프로젝트가 왜 흥미로운지, 그리고 어떤 한계가 있는지 파헤쳐 보겠습니다.

## 아키텍처 Deep Dive: 파일이 아닌 B-Tree를 이해하는 스토리지

기존에도 S3 위에서 SQLite를 돌리려는 시도(sql.js-httpvfs 등)는 많았습니다. 대부분은 단순히 원본 `.db` 파일에 대해 HTTP Range GET 요청을 날리는 방식이었습니다. 하지만 이 방식은 치명적인 단점이 있습니다. SQLite의 페이지들이 파일 전체에 흩어져 있기 때문에, 쿼리 한 번에 수십 번의 랜덤 네트워크 I/O가 발생한다는 점입니다.

Turbolite의 진정한 무기는 **B-Tree Aware Grouping** 에 있습니다.

이 VFS는 단순히 바이트 오프셋을 읽는 게 아니라, SQLite의 내부 구조를 파악합니다. 페이지를 Interior B-tree, Index leaf, Data leaf 페이지로 분류하고, 같은 테이블이나 인덱스에 속한 페이지들을 모아서 하나의 S3 객체(~16MB 단위)로 묶어버립니다.

- **Interior Pages:** 트리를 탐색할 때 무조건 거쳐야 하는 핵심 페이지입니다. Turbolite는 이를 별도로 압축하여 VFS가 열릴 때 메모리에 통째로 올려버립니다(Eager load).
- **Index & Data Pages:** 백그라운드에서 지연 로딩(Lazy load)되며, 캐시 미스가 발생하면 해당 서브 청크만 S3 Range GET으로 가져옵니다.

결과적으로 콜드 쿼리 상황에서도 S3 요청 횟수를 극단적으로 줄일 수 있습니다. S3 환경에서는 대역폭(Bandwidth)보다 요청 횟수(Round Trips)가 훨씬 비싼 병목이라는 점을 정확히 찌른 설계입니다.

## Query-Plan Frontrunning: 쿼리 실행 전에 미리 가져오기

개인적으로 가장 감탄한 부분은 쿼리 계획을 가로채는 방식입니다.

단순히 캐시 미스가 났을 때 다음 페이지를 가져오는 Reactive Prefetching은 한계가 있습니다. Turbolite는 쿼리가 실행되기 직전에 SQLite의 `EXPLAIN QUERY PLAN`을 인터셉트합니다. 쿼리가 어떤 테이블과 인덱스를 풀 스캔할지 미리 알아내고, 첫 번째 페이지를 읽기도 전에 병렬로 S3에 Fetch 요청을 때려버립니다.

과거에 분산 데이터베이스 엔진을 튜닝할 때 클라이언트 사이드에서 비슷한 최적화를 구현하느라 피를 토했던 기억이 나는데, 이걸 SQLite VFS 레벨에서 깔끔하게 추상화했다는 점이 매우 인상적입니다.

```python
import turbolite

# S3 호환 스토리지를 백엔드로 사용하는 연결
conn = turbolite.connect("my.db", mode="s3",
                         bucket="my-bucket",
                         endpoint="https://t3.storage.dev")
                         
conn.execute("CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT)")
conn.execute("INSERT INTO users VALUES (1, 'alice')")
conn.commit()

# 콜드 상태에서도 캐시와 Prefetch를 통해 빠르게 결과를 반환
alice = conn.cursor().execute("SELECT * FROM users").fetchone()
```

## 현업 엔지니어의 시선: 트레이드오프와 한계점

기술적으로 훌륭하지만, 만병통치약은 아닙니다. 현업에서 도입을 검토한다면 다음의 **트레이드오프** 를 반드시 인지해야 합니다.

첫째, **쓰기 증폭 (Write Amplification)** 문제입니다. Turbolite는 페이지가 하나라도 변경되면 해당 페이지가 속한 그룹(~16MB) 전체를 새로운 버전으로 S3에 PUT 해야 합니다. S3의 특성상 In-place 업데이트가 불가능하기 때문입니다. 따라서 이 솔루션은 트랜잭션이 빈번한 OLTP 환경에는 절대 쓸 수 없습니다. 철저하게 Read-heavy 워크로드나 하루 한두 번 배치로 쓰기가 일어나는 환경에 적합합니다.

둘째, **단일 쓰기 (Single Writer)** 제약입니다. 다중 노드에서 동시에 쓰기를 시도하면 매니페스트 파일이 꼬이게 됩니다. HN 댓글에서도 분산 락이나 조건부 PUT(Conditional PUT)을 활용한 Multi-writer 지원에 대한 토론이 있었지만, 원작자 역시 데이터 정합성을 보장하는 것이 매우 까다롭다고 인정했습니다.

Litestream이나 LiteFS와 비교하는 시각도 많습니다만, 두 진영은 푸는 문제가 다릅니다. Litestream은 Local SSD의 속도를 누리면서 S3를 "백업 및 복구용"으로 쓰는 HA 솔루션입니다. 반면 Turbolite는 로컬 스토리지를 아예 배제하고, Ephemeral Compute(Lambda, Fly.io 임시 인스턴스 등)에서 S3를 "주 스토리지"로 직접 쿼리하기 위한 솔루션입니다.

## Hacker News 커뮤니티의 반응

해커뉴스 스레드에서도 상당히 수준 높은 토론이 오갔습니다.

가장 눈에 띄는 의견은 Turbolite의 B-Tree 기반 그룹핑이 Litestream의 시간적 지역성(Temporal Locality)과 정반대의 길을 걷고 있다는 점이었습니다. Litestream은 트랜잭션 순서대로 WAL을 쌓기 때문에 시점 복구(Point-in-time restore)에 유리하지만, Turbolite는 철저하게 "읽기 성능"을 위해 데이터를 논리적 구조로 재배치합니다.

또한 S3 GET 요청 비용에 대한 현실적인 우려도 있었습니다. 캐시 축출(Eviction) 주기를 너무 짧게 잡으면 S3 API 호출 비용이 기하급수적으로 늘어날 수 있습니다. 원작자는 S3 Express를 활용하고 적절한 체크포인트 주기를 가져가면 월 몇 달러 수준으로 방어할 수 있다고 방어 논리를 펼쳤습니다.

## 총평: 프로덕션 레디인가?

솔직히 말해, 아직 프로덕션의 메인 DB로 쓰기에는 무리가 있습니다. 원작자 본인도 저장소에 "데이터가 손상될 수 있으니 주의하라"고 명시해 두었죠.

하지만 클라우드 네이티브 환경에서 SQLite가 나아가야 할 또 다른 방향성을 제시했다는 점에서 극찬하고 싶습니다. 특히 수만 개의 Agent가 각각 독립적인 상태(Memory/Context)를 유지해야 하는 AI 애플리케이션이나, 테넌트별로 격리된 콜드 데이터를 가끔 분석해야 하는 Serverless 환경에서는 매우 강력한 무기가 될 잠재력이 있습니다.

당장 회사 프로덕트에 넣지는 않겠지만, 주말 사이드 프로젝트로 Tigris와 엮어서 Agentic 시스템의 상태 저장소로 꼭 한번 테스트해 볼 생각입니다.

- **Original Repository:** https://github.com/russellromney/turbolite
- **Hacker News Thread:** https://news.ycombinator.com/item?id=47534283
