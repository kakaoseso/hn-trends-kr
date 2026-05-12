---
title: "DuckDB의 클라이언트-서버 프로토콜 Quack: 왜 그들은 Arrow Flight를 버렸을까?"
description: "DuckDB의 클라이언트-서버 프로토콜 Quack: 왜 그들은 Arrow Flight를 버렸을까?"
pubDate: "2026-05-12T19:22:27Z"
---

DuckDB는 원래 in-process 분석용 데이터베이스로 시작했습니다. SQLite의 OLAP 버전이라고 불리며 데이터 엔지니어와 분석가들 사이에서 엄청난 인기를 끌었죠. 하지만 현업에서 시스템을 운영하다 보면 필연적으로 벽에 부딪히게 됩니다. "여러 프로세스에서 동시에 데이터를 밀어 넣고 싶은데 어떡하지?"

지금까지 우리는 이 문제를 해결하기 위해 DuckDB 앞에 REST API를 붙이거나, Arrow Flight SQL을 억지로 얹거나, 심지어 PostgreSQL 안에 DuckDB를 기생시키는(pg_duckdb) 기괴한 아키텍처를 만들어왔습니다. DuckDB 팀도 결국 커뮤니티의 이러한 삽질을 외면할 수 없었나 봅니다. 2026년 5월, 그들은 공식적인 클라이언트-서버 프로토콜인 **Quack** 을 발표했습니다.

솔직히 처음 이 소식을 들었을 때 저는 약간 회의적이었습니다. "잘 돌아가는 in-process DB에 굳이 네트워크 오버헤드를 얹는다고?" 하지만 아키텍처 결정과 벤치마크 결과를 뜯어보니, 꽤나 합리적이고 엔지니어링 적으로 흥미로운 포인트들이 많았습니다.

## 마법 같은 설정 경험

DuckDB의 철학답게 설정은 놀라울 정도로 간단합니다. 별도의 서버 데몬을 띄울 필요 없이, 기존 DuckDB 인스턴스에서 확장을 로드하기만 하면 됩니다.

```sql
-- Server (DuckDB #1)
INSTALL quack FROM core_nightly;
LOAD quack;
CALL quack_serve('quack:localhost', token = 'super_secret');
```

```sql
-- Client (DuckDB #2)
INSTALL quack FROM core_nightly;
LOAD quack;
CREATE SECRET (TYPE quack, TOKEN 'super_secret');
ATTACH 'quack:localhost' AS remote;
```

이게 끝입니다. 클라이언트에서는 `remote.table_name` 형태로 원격 데이터에 바로 쿼리를 날릴 수 있습니다.

## 아키텍처 딥다이브: 왜 HTTP와 독자 규격인가?

이 발표에서 가장 논쟁적인 부분이자, Hacker News에서도 갑론을박이 오갈 만한 주제는 바로 **"왜 기존의 Arrow Flight SQL을 쓰지 않았는가?"** 입니다.

### 1. HTTP 기반의 프로토콜
Quack은 기본적으로 HTTP 위에서 동작합니다. 2026년에 새로운 데이터베이스 프로토콜을 바닥부터 TCP로 짜는 것은 바보 같은 짓입니다. HTTP를 선택함으로써 얻는 이점은 명확합니다. Nginx나 HAProxy 같은 기존 인프라를 그대로 활용해 로드 밸런싱과 SSL Termination을 처리할 수 있고, 무엇보다 **DuckDB-Wasm** 이 브라우저에서 서버의 DuckDB와 직접 통신할 수 있게 됩니다. 이는 웹 기반 데이터 애플리케이션 아키텍처에 큰 변화를 가져올 수 있습니다.

### 2. Arrow Flight를 버린 이유: Latency와 통제권
업계 표준으로 자리 잡고 있는 Arrow Flight SQL을 채택하지 않은 것은 꽤 과감한 결정입니다. DuckDB 팀이 밝힌 이유는 두 가지입니다.

- **Round-Trip 오버헤드:** Arrow Flight SQL의 치명적인 단점은 쿼리 하나를 실행할 때 최소 두 번의 네트워크 왕복(CommandStatementQuery 요청 후 DoGet 요청)이 필요하다는 것입니다. Quack은 이를 단일 Round-Trip으로 최적화했습니다. Latency에 민감한 환경에서는 이 차이가 극심한 성능 격차를 만듭니다.
- **직렬화 통제권:** 외부 표준에 얽매이면 새로운 데이터 타입이나 내부 최적화를 적용할 때마다 표준이 업데이트되기를 기다려야 합니다. Quack은 `application/duckdb`라는 독자적인 MIME 타입을 사용해 DuckDB의 내부 WAL(Write-Ahead Log) 직렬화 포맷을 그대로 네트워크로 쏴버립니다.

개인적으로 이 결정에 전적으로 동의합니다. 범용 호환성보다 극단적인 성능과 제어권이 필요한 구간에서는 독자 규격이 맞습니다.

## 성능: 압도적인 Bulk Transfer, 하지만 한계도 명확함

벤치마크 결과를 보면 Quack의 설계 의도가 명확히 드러납니다.

- **Bulk Transfer:** 6천만 건의 TPC-H 데이터를 전송하는 데 단 5초가 걸렸습니다. Arrow Flight(17초)나 PostgreSQL(158초)을 압도적으로 짓밟는 수치입니다. Zero-copy에 가까운 내부 직렬화 포맷 전송이 빛을 발하는 순간입니다.
- **Small Writes:** 8스레드 환경에서 초당 약 5,500 트랜잭션을 기록하며 PostgreSQL을 이겼다고 자랑합니다. 하지만 **여기서 속으시면 안 됩니다.** 8스레드를 넘어가는 순간 DuckDB 내부의 동시성 제어 한계로 인해 성능이 꺾입니다. 반면 PostgreSQL은 훨씬 더 높은 동시성에서도 안정적으로 스케일링되죠.

## Principal Engineer의 시선: 프로덕션에 쓸 만한가?

Quack은 DuckDB를 단순한 개인용 분석 장난감에서 데이터 인프라의 핵심 컴포넌트로 끌어올리는 중요한 마일스톤입니다.

하지만 분명히 짚고 넘어갑시다. **DuckDB는 여전히 OLTP 데이터베이스가 아닙니다.** Quack이 초당 수천 건의 쓰기를 지원한다고 해서, 여러분의 메인 서비스 백엔드 DB를 DuckDB로 교체하려 든다면 끔찍한 결말을 맞이할 것입니다.

이 기술이 진짜 빛을 발하는 유스케이스는 다음과 같습니다:
1. **분산된 텔레메트리 수집:** 여러 애플리케이션 서버에서 발생하는 로그를 중앙 DuckDB 인스턴스로 가볍게 밀어 넣고, 대시보드에서는 이를 실시간으로 집계할 때.
2. **데이터 레이크 카탈로그:** DuckLake와 결합하여 중앙화된 메타데이터 및 카탈로그 서버로 활용할 때.

올가을 DuckDB v2.0과 함께 Quack의 정식 버전이 출시된다고 하니, 그때쯤이면 사내 내부용 대시보드나 가벼운 데이터 파이프라인에는 충분히 프로덕션 레벨로 도입해 볼 만할 것 같습니다.

## References
- Original Article: https://duckdb.org/2026/05/12/quack-remote-protocol
- Hacker News Discussion: https://news.ycombinator.com/item?id=48111765
