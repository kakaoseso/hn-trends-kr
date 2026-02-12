---
title: "Parquet와 Iceberg의 한계를 넘어서: Lance 포맷이 주목받는 이유"
description: "Parquet와 Iceberg의 한계를 넘어서: Lance 포맷이 주목받는 이유"
pubDate: "2026-02-12T13:08:42Z"
---

최근 데이터 엔지니어링 씬, 특히 Object Storage 기반의 Big Data 생태계가 2025년에 들어서며 아주 흥미롭게 돌아가고 있습니다. Iceberg V3가 나왔고, Databricks가 Neon을 인수했죠. 하지만 제 엔지니어링 레이더에 가장 강하게 잡힌 신호는 바로 **Lance** 입니다.

솔직히 말해, 우리가 Parquet를 표준으로 쓴 지 10년이 넘었습니다. 훌륭한 포맷이지만, 시대가 변했습니다. 오늘은 이 Lance가 도대체 뭐길래 Parquet와 Iceberg의 후계자를 자처하는지, 그리고 실무 관점에서 이게 말이 되는 이야기인지 뜯어보겠습니다.

## 1. Parquet가 못하는 것: Random Read

우리는 Parquet를 사랑합니다. 압축률 좋고, 컬럼 기반이라 집계(Aggregation) 쿼리에 최적화되어 있으니까요. 하지만 엔지니어로서 솔직해집시다. Parquet 파일에서 특정 `ID` 하나를 찾으려 할 때(Point Lookup) 성능이 어땠나요? 끔찍합니다. Parquet는 기본적으로 스캔(Scan)을 위한 포맷이기 때문입니다.

원문 기사에서 설명하듯, Lance는 이 지점을 파고듭니다.
- **Random Access 최적화:** Lance는 `WHERE id = 123` 같은 쿼리에서 Parquet보다 훨씬 빠릅니다. Parquet가 1MB 단위의 페이지를 쓴다면, Lance는 더 작은 단위나 효율적인 인덱싱을 통해 필요한 부분만 콕 집어 가져옵니다.
- **Scan 성능 유지:** 그러면서도 대량의 데이터를 읽을 때의 성능은 Parquet와 비슷하게 유지한다고 주장합니다. 이게 사실이라면, OLAP과 OLTP의 경계에 있는 워크로드에 엄청난 이점입니다.

## 2. 데이터 복사 없는 스키마 변경 (Schema Evolution)

이게 제가 가장 놀란 부분입니다. 보통 Iceberg나 Delta Lake 같은 Table Format을 쓸 때, 컬럼을 추가하거나 데이터 타입을 바꾸면 꽤 골치 아픈 일이 벌어집니다. 메타데이터만 바꾸는 경우도 있지만, 특정 상황에서는 데이터 파일을 다시 써야(Rewrite) 하는 비용이 발생합니다. 데이터가 페타바이트 단위라면 이건 재앙에 가깝죠.

Lance는 **데이터 전체를 복사하지 않고도 컬럼을 추가(Ad-hoc column addition)** 할 수 있다고 합니다. 아마도 기존 데이터 파일은 그대로 두고, 새로운 컬럼 값만 별도의 파일이나 세그먼트로 저장한 뒤 읽을 때 합치는 방식을 쓸 것으로 보입니다. 중요한 건 이 과정에서 **MVCC(다중 버전 동시성 제어)** 를 유지한다는 점입니다.

## 3. 왜 지금인가? 결국 AI와 Vector Search

갑자기 왜 이런 포맷이 튀어나왔을까요? 답은 명확합니다. **AI** 때문입니다.

요즘 RAG(Retrieval-Augmented Generation)나 멀티모달 데이터를 다루다 보면, 벡터 검색(Vector Search)과 일반적인 메타데이터 필터링을 동시에 해야 합니다. 기존 Parquet 기반의 Data Lake는 이 '검색' 패턴에 맞지 않습니다. 그렇다고 비싼 Vector DB를 따로 구축하자니 데이터 사일로(Silo)가 생기죠.

Lance는 아예 인덱스를 내장하고 있습니다.
- **지원 인덱스:** BTree(스칼라), Inverted Index(Full-text search), Vector Index(HNSW 등).

즉, Data Lake 위에 바로 인덱스를 걸어서 Vector DB처럼 쓰겠다는 야심 찬 계획입니다. 2018년쯤 우리가 Hadoop 위에 SQL 엔진을 얹으려고 고생했던 역사가 이제는 Object Storage 위에 AI 엔진을 얹는 형태로 반복되는 느낌이네요.

## Hacker News의 반응과 내 생각

커뮤니티의 반응도 흥미롭습니다. 특히 한 유저(anon)가 지적한 부분이 핵심을 찌릅니다.

> "데이터 타입이 Apache Arrow와 1-1 대응된다는 점이 흥미롭다. LanceDB와 Arrow Flight 엔드포인트를 엮으면 아주 간단하게 고성능 API를 만들 수 있을 것이다."

전적으로 동의합니다. 제가 예전에 고성능 데이터 시스템을 설계할 때 가장 병목이 생겼던 곳이 스토리지 포맷(Disk)과 메모리 포맷(RAM) 간의 변환(SerDe) 비용이었습니다. Lance는 Arrow 구조를 그대로 디스크에 내리기 때문에 **Zero-copy** 읽기가 가능합니다. Python/Pandas/Polars 생태계와의 통합 속도가 엄청날 것이라는 뜻입니다.

## 결론: 아직은 시기상조? 아니면 도입각?

Lance는 기술적으로 매우 매력적입니다. 특히 **AI/ML 파이프라인** 을 운영하고 있거나, 대용량 데이터에서 **Vector Search** 를 해야 한다면 당장 PoC를 해볼 가치가 충분합니다.

하지만 기존의 전통적인 BI/Data Warehousing 용도로 Parquet를 전면 교체하기엔 아직 생태계가 젊습니다. Parquet는 세상 모든 도구(Spark, Trino, Snowflake, BigQuery 등)가 지원하지만, Lance는 아직 그 정도는 아닙니다.

개인적으로는 **Vector Search나 Random Access가 빈번한 'Hot' 데이터 계층부터 Lance로 교체** 하고, 기존의 무거운 Batch성 데이터는 Parquet로 유지하는 하이브리드 전략이 현시점에서는 정답에 가깝다고 봅니다. 하지만 Vortex 같은 경쟁자들도 나오고 있는 걸 보면, 차세대 포맷 전쟁은 이제 막 시작된 것 같습니다.

---
*References:*
- [Original Article](https://tontinton.com/posts/lance/)
- [Hacker News Discussion](https://news.ycombinator.com/item?id=46939200)
