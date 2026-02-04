---
title: "AliSQL: MySQL에 DuckDB와 Vector 엔진을 탑재한 알리바바의 야심작, 과연 실전용일까?"
description: "AliSQL: MySQL에 DuckDB와 Vector 엔진을 탑재한 알리바바의 야심작, 과연 실전용일까?"
pubDate: "2026-02-04T06:29:33Z"
---

엔지니어링 생활 15년 차, 우리가 가장 싫어하는 작업 중 하나는 바로 '데이터 이동'입니다. OLTP(MySQL)에서 데이터를 뽑아내서, Kafka 태우고, 변환해서, 결국 OLAP(Snowflake나 BigQuery)에 넣는 그 지루하고 깨지기 쉬운 파이프라인 말이죠.

최근 Hacker News에서 꽤 흥미로운 프로젝트가 화제가 되었습니다. 바로 알리바바(Alibaba)가 오픈소스로 공개한 **AliSQL** 입니다. 단순히 MySQL의 튜닝 버전이 아닙니다. 이 녀석은 MySQL 안에 **DuckDB** 와 **Vector Search** 엔진을 통째로 심어버렸습니다.

오늘은 이 프로젝트가 단순한 '기능 자랑'인지, 아니면 우리가 기다려온 진정한 **HTAP(Hybrid Transactional/Analytical Processing)** 솔루션인지 딥다이브 해보겠습니다.

## 1. AliSQL이 도대체 뭔가요?

기본적으로 AliSQL은 **MySQL 8.0.44** 를 기반으로 한 포크(Fork) 버전입니다. 알리바바 그룹 내부 프로덕션 환경에서 이미 하드하게 구르고 있는 녀석이죠. 하지만 이번 공개가 충격적인 이유는 두 가지 강력한 엔진을 내장했기 때문입니다.

### DuckDB Storage Engine
MySQL의 플러그인 아키텍처(Pluggable Storage Engine)를 극한으로 활용했습니다. InnoDB 옆에 DuckDB를 붙여버린 겁니다.

- **작동 방식:** 사용자는 일반적인 MySQL 쿼리를 날립니다. 하지만 분석이 필요한 무거운 쿼리는 내부적으로 DuckDB 엔진이 처리합니다.
- **장점:** 별도의 ETL 파이프라인 없이, MySQL 프로토콜 그대로 컬럼 기반(Columnar) 분석이 가능합니다. Hacker News의 한 유저가 지적했듯, "기존 MySQL 연결과 툴링을 유지하면서 분석 쿼리만 라우팅 할 수 있다는 점"은 운영 측면에서 엄청난 세일즈 포인트입니다.

### Vector Storage (HNSW)
요즘 AI 트렌드에 맞춰 벡터 검색도 지원합니다. 최대 16,383차원의 벡터를 처리하며, **HNSW(Hierarchical Navigable Small World)** 알고리즘을 내장했습니다. 이제 RAG(Retrieval-Augmented Generation) 구현을 위해 별도의 Vector DB를 띄우지 않고 MySQL 하나로 퉁칠 수 있다는 이야기입니다.

## 2. 기술적 딥다이브: 진짜 HTAP인가, 아니면 그저 '접착제'인가?

Hacker News 댓글 창은 이 부분에서 논쟁이 뜨겁습니다. "이건 진정한 HTAP다"라는 의견과 "그저 두 개의 DB를 인터페이스로 묶은 것뿐"이라는 의견이 맞서고 있죠.

### 데이터 일관성 (Consistency)의 문제
가장 큰 기술적 난관은 **InnoDB의 트랜잭션 데이터와 DuckDB의 분석 데이터 간의 동기화** 입니다.

- **User anon의 지적:** "모든 하이브리드 OLTP/OLAP 시스템이 빛을 발하거나 조용히 데이터를 잃어버리는 지점이 바로 여기다."
- AliSQL은 Binlog Parallel Flush와 같은 기술을 통해 복제 지연(Replication Lag)을 최소화한다고 주장합니다. 하지만, 한 세션 내에서 트랜잭션을 처리하면서 동시에 분석 쿼리를 날릴 때의 격리 수준(Isolation Level)을 어떻게 보장하는지는 코드를 더 까봐야 알 수 있을 것 같습니다.

### PostgreSQL 진영과의 비교
Postgres는 `pg_duckdb`나 `pgvector` 같은 강력한 익스텐션 생태계가 이미 자리를 잡았습니다. 반면 MySQL은 구조적으로 이런 확장이 쉽지 않았는데, 알리바바는 아예 코어 레벨에서 이를 통합하는 방식을 택했습니다. 이는 성능 면에서는 이점이 있을 수 있지만, 유지보수 측면에서는 '벤더 종속'의 위험이 있습니다.

## 3. 엔지니어의 시선: 찜찜한 구석들

솔직히 말해서, 기술적으로 흥미롭지만 당장 프로덕션에 도입하기엔 몇 가지 걸리는 점들이 있습니다.

### 1. 수상한 Git History
GitHub 커밋 로그를 보면 2022년에 2개, 2026년(?)에 5개 같은 식으로 되어 있습니다. 이는 전형적인 **"내부 소스코드 덤프(Dump)"** 현상입니다. 내부 Jira 티켓이나 중국어 주석, 보안 이슈 등을 제거하고 스냅샷만 공개했을 가능성이 큽니다. 오픈소스 커뮤니티가 주도하는 프로젝트라기보다는, 알리바바가 "우리 기술력 좀 봐라" 하고 던져놓은 느낌을 지울 수 없습니다.

### 2. 빌드 환경의 복잡성
C++17 컴파일러와 CMake 3.x가 필요하며, 빌드 과정이 꽤 무겁습니다.

```bash
# 릴리즈 빌드 명령어
sh build.sh -t release -d /path/to/install/dir
```

단순해 보이지만, MySQL 커스텀 빌드는 의존성 지옥(Dependency Hell)에 빠지기 딱 좋습니다. Docker 이미지가 제공되겠지만, 직접 운영하려면 상당한 삽질이 예상됩니다.

## 4. 결론: 찍먹은 OK, 도입은 신중하게

AliSQL은 **"MySQL 하나로 모든 걸 해결하고 싶은"** 엔지니어들의 로망을 정확히 타격했습니다. 특히 이미 MySQL을 메인으로 쓰고 있고, 가벼운 분석 니즈 때문에 복잡한 데이터 파이프라인을 구축하기 싫은 스타트업 CTO들에게는 매력적인 제안입니다.

하지만 저는 아직 보수적인 입장을 취하고 싶습니다.

1.  **커뮤니티 검증 부족:** 아직 공개된 지 얼마 안 되었고, 이슈 트래커가 활성화되지 않았습니다.
2.  **호환성:** 오라클의 공식 MySQL 8.0과 얼마나 벌어질지 알 수 없습니다. 나중에 순정 MySQL로 돌아가고 싶을 때 발목을 잡을 수 있습니다.

**제 Verdict:**
사이드 프로젝트나 내부 어드민 툴의 백엔드로는 **강력 추천** 합니다. DuckDB의 성능을 MySQL 프로토콜로 쓰는 건 확실히 쾌적하니까요. 하지만, 회사의 핵심 결제 데이터를 다루는 메인 DB로 쓰기엔 아직 '시기상조'입니다. 조금 더 지켜봅시다.

**참고 링크:**
- [AliSQL GitHub Repository](https://github.com/alibaba/AliSQL)
- [Hacker News Discussion](https://news.ycombinator.com/item?id=46875228)
