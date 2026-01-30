---
title: "DuckDB와 Ray의 만남: Quack-Cluster로 나만의 분산 SQL 엔진 맛보기"
description: "DuckDB와 Ray의 만남: Quack-Cluster로 나만의 분산 SQL 엔진 맛보기"
pubDate: "2026-01-30T15:50:31Z"
---

최근 데이터 엔지니어링 씬에서 가장 뜨거운 감자를 꼽으라면 단연 **DuckDB** 일 겁니다. 로컬에서 믿기 힘들 정도로 빠른 속도를 보여주는 이 'In-process OLAP' DB는 많은 엔지니어들의 사랑을 받고 있죠. 하지만 DuckDB를 쓸 때마다 마주하는 필연적인 질문이 하나 있습니다.

"이거 데이터 커지면 어떻게 스케일아웃(Scale-out) 하죠?"

보통 이 시점에서 우리는 무거운 Spark를 꺼내들거나, 운영이 까다로운 Trino 클러스터를 구축하곤 합니다. 하지만 오늘 소개할 **Quack-Cluster** 는 조금 다른, 훨씬 'Python스러운' 접근 방식을 보여줍니다. 바로 분산 컴퓨팅 프레임워크인 **Ray** 위에 DuckDB를 태운 것이죠.

Github 트렌딩을 보다가 흥미로운 프로젝트가 있어 뜯어봤습니다. 과연 이게 '장난감' 수준일지, 아니면 차세대 경량 데이터 웨어하우스의 청사진일지, Principal Engineer 관점에서 깊게 파헤쳐 보겠습니다.

## Quack-Cluster가 해결하려는 문제

핵심은 간단합니다. **"ETL 없이, 인프라 관리 없이, S3에 있는 대용량 데이터를 SQL로 쿼리하고 싶다."**

기존 빅데이터 시스템들은 너무 무겁습니다(JVM 기반이 대다수죠). 반면 Quack-Cluster는 Python 생태계의 강자인 **Ray** 를 오케스트레이터로 쓰고, 개별 워커 노드에서 **DuckDB** 엔진을 돌리는 방식을 택했습니다. 이렇게 하면 Python 개발자들에게 익숙한 환경을 유지하면서도, DuckDB의 벡터화 연산 능력과 Ray의 분산 처리 능력을 동시에 가져갈 수 있습니다.

## 아키텍처 딥다이브: 어떻게 작동하나?

이 프로젝트의 아키텍처는 꽤 직관적이면서도 영리합니다. 크게 세 가지 컴포넌트로 나뉩니다.

1. **Coordinator (두뇌):** FastAPI로 구현된 API 서버입니다. 사용자가 SQL을 던지면, **SQLGlot** 을 사용해 파싱하고 실행 계획(Execution Plan)을 짭니다. 여기서 어떤 파일(예: `s3://bucket/*.parquet`)을 읽어야 할지 결정합니다.
2. **Ray Cluster (근육):** 실제 연산이 일어나는 곳입니다. Coordinator가 쪼개준 작업(Task)을 여러 워커 노드에 분배합니다.
3. **Worker Nodes (심장):** 각 Ray Actor 내부에는 DuckDB 인스턴스가 임베딩되어 있습니다. 할당받은 데이터 파티션을 읽어서 로컬에서 쿼리를 수행하고, 그 결과(Partial Result)를 Arrow 포맷으로 반환합니다.

```mermaid
graph TD
    User[Client] -->|SQL Query| Coord[Coordinator (FastAPI + SQLGlot)]
    Coord -->|Plan & Distribute| Ray[Ray Cluster]
    subgraph Ray Cluster
        W1[Worker 1 (DuckDB)]
        W2[Worker 2 (DuckDB)]
    end
    W1 -->|Read| S3[Object Storage]
    W2 -->|Read| S3
    W1 -->|Arrow Result| Coord
    W2 -->|Arrow Result| Coord
    Coord -->|Aggregation| User
```

이 구조의 장점은 **MPP(Massively Parallel Processing)** 데이터베이스의 흉내를 아주 적은 코드로 낼 수 있다는 점입니다. 특히 데이터 교환 포맷으로 Apache Arrow를 사용한다는 점은 메모리 복사 비용을 줄이는 데 필수적인 선택이었습니다.

## 직접 돌려보며 느낀 점

설치는 매우 간단합니다. Docker와 Make만 있으면 로컬에서 바로 클러스터를 띄울 수 있습니다.

```bash
# 클러스터 실행 (워커 2개)
make up scale=2
```

쿼리 테스트를 위해 `curl`을 날려봤습니다. 와일드카드(`*`)를 지원해서 S3나 로컬 디렉토리의 파티션된 파일들을 한 번에 긁어모으는 게 인상적입니다.

```bash
curl -X 'POST' 'http://localhost:8000/query' \
-d '{
  "sql": "SELECT product, SUM(sales) FROM \"data_part_*.parquet\" GROUP BY product"
}'
```

### Principal Engineer의 시선: 좋았던 점

- **Simplicity:** 코드를 뜯어보니 생각보다 복잡하지 않습니다. Ray가 분산 처리의 복잡한 부분(Task Scheduling, Fault Tolerance)을 다 처리해주기 때문입니다. 덕분에 개발자는 '쿼리 로직'에만 집중할 수 있습니다.
- **Python Native:** Java 힙 메모리 튜닝하느라 고생할 필요가 없습니다. 이미 ML 파이프라인 때문에 Ray를 쓰고 있는 조직이라면, 별도 인프라 추가 없이 바로 SQL 엔진을 얹을 수 있다는 게 큰 매력입니다.
- **SQLGlot 활용:** SQL 파서로 SQLGlot을 선택한 건 훌륭합니다. 다양한 SQL 방언(Dialect)을 처리할 수 있는 유연성을 확보했습니다.

### 우려되는 점 (Critical Critique)

하지만 냉정하게 봐야 할 부분들도 분명히 존재합니다.

- **Distributed JOIN의 악몽:** README에는 Distributed JOIN을 지원한다고 되어 있습니다. 하지만 분산 시스템에서 조인은 단순히 "나눠서 처리"하는 게 아닙니다. 데이터 셔플링(Shuffling)이 발생해야 하는데, Ray의 Object Store를 통해 대량의 데이터를 노드 간에 이동시키는 비용이 만만치 않을 겁니다. 단순 집계(Aggregation)는 빠르겠지만, 복잡한 조인 쿼리에서는 네트워크 병목이 올 가능성이 높습니다.
- **Optimizer의 부재:** 진짜 MPP 데이터베이스(Snowflake, BigQuery 등)의 핵심은 쿼리 옵티마이저입니다. "어떤 순서로 조인할지", "데이터를 어떻게 브로드캐스트 할지" 결정하는 두뇌가 필요한데, 이 프로젝트는 DuckDB의 로컬 옵티마이저에 의존하고 분산 레벨의 최적화는 아직 초기 단계로 보입니다.
- **AI Generated Code:** 프로젝트 설명에 "AI 도구를 활용해 개발했다"고 명시되어 있습니다. 솔직해서 좋긴 합니다만, 코어 아키텍처의 엣지 케이스(Edge Case)들이 얼마나 견고하게 처리되었는지는 코드를 꼼꼼히 리뷰해봐야 할 것 같습니다.

## Hacker News의 반응

아쉽게도 제가 글을 쓰는 시점에는 아직 HN에 댓글이 달리지 않았습니다. 하지만 최근 DuckDB와 관련된 분산 처리 시도들(예: MotherDuck 등)이 워낙 핫하기 때문에, 곧 엔지니어들 사이에서 "Ray vs Spark" 혹은 "DuckDB의 확장성"을 두고 토론이 벌어질 것으로 예상됩니다.

## 결론: Production Ready인가?

**아직은 아닙니다.** 하지만 **강력한 프로토타입** 임은 분명합니다.

당장 회사의 메인 데이터 웨어하우스를 이걸로 교체하라고 하진 않겠습니다. 하지만 다음과 같은 상황이라면 도입을 고려해볼 만합니다.

1. 이미 **Ray 클러스터** 를 운영 중인 ML 조직.
2. **수 TB 이하** 의 데이터를 S3에 쌓아두고 가끔씩 무거운 쿼리를 돌려야 하는 경우.
3. Spark를 띄우기엔 너무 무겁고, AWS Athena 비용은 부담스러운 경우.

개인적으로는 이 프로젝트가 **"Python으로 나만의 분산 데이터 엔진을 만드는 교과서"** 같다는 느낌을 받았습니다. 소스 코드가 공개되어 있으니, 분산 시스템이 어떻게 돌아가는지 궁금한 주니어~시니어 엔지니어들에게 일독을 권합니다.

오픈소스의 매력은 이런 실험적인 시도들이 계속 나온다는 점이죠. DuckDB가 어디까지 날아오를지, 계속 지켜봐야겠습니다.

---

**References:**
- [Quack-Cluster GitHub Repository](https://github.com/kristianaryanto/Quack-Cluster)
- [Hacker News Discussion](https://news.ycombinator.com/item?id=46773793)
