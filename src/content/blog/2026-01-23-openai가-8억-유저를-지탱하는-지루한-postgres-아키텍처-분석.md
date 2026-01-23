---
title: "OpenAI가 8억 유저를 지탱하는 지루한 Postgres 아키텍처 분석"
description: "OpenAI가 8억 유저를 지탱하는 지루한 Postgres 아키텍처 분석"
pubDate: "2026-01-23T10:14:47Z"
---

최근 OpenAI 엔지니어링 블로그에 꽤 흥미로운 글이 하나 올라왔습니다. 제목부터 자극적인 **Scaling PostgreSQL to power 800M users** 입니다. 8억 명의 ChatGPT 유저 트래픽을 처리하는데, 여전히 단일(Single) Postgres Primary를 메인으로 쓰고 있다는 내용이죠.

주니어 시절엔 무조건 'Sharding'이나 'Microservices'가 정답인 줄 알지만, 시니어 레벨로 올라갈수록 **Monolith** 와 **Vertical Scaling** 의 위대함을 깨닫게 됩니다. 이번 OpenAI의 사례는 그 'Simple is Best'라는 철학을 아주 극단적으로, 그리고 성공적으로 증명해 낸 케이스라 할 수 있겠습니다. Hacker News에서도 뜨거운 논쟁이 오가고 있는데, 엔지니어 관점에서 핵심만 씹고 뜯어보겠습니다.

## 1. 8억 명을 태운 단일 Primary, 그리고 50개의 Read Replica

많은 스타트업들이 유저 100만 명만 넘어도 "이제 샤딩해야 하나요?"라고 묻습니다. 하지만 OpenAI는 8억 명 규모에서도 **Single Primary** 구조를 유지하고 있습니다. 이게 어떻게 가능할까요?

핵심은 철저한 **Read/Write 분리** 입니다. OpenAI는 하나의 Primary 인스턴스에 무려 50개의 Read Replica를 붙여서 운영하고 있습니다. 보통 Replica가 많아지면 Replication Lag이나 Primary의 부하를 걱정하기 마련인데, 이들은 Azure의 인프라 위에서 이를 해결했습니다.

Hacker News의 한 유저는 이를 두고 "트위터 초기 시절(2011년)의 MySQL 구조가 생각난다"고 회상하더군요. 당시 트위터도 단일 마스터에 수십 개의 리플리카를 붙여서 버텼습니다. 10년이 지난 지금, 하드웨어 스펙이 깡패가 되면서 이 전략의 유효기간이 훨씬 길어진 겁니다.

- **Insight:** 분산 데이터베이스나 샤딩은 복잡도를 기하급수적으로 늘립니다. 돈(하드웨어 비용)으로 해결할 수 있는 문제라면, 엔지니어링 리소스를 갈아 넣는 것보다 **Scale-up** 이 훨씬 저렴합니다.

## 2. 스키마 마이그레이션과 Lock 관리의 현실적인 팁

개인적으로 이 글에서 가장 실무적인 팁이라고 느낀 부분은 스키마 변경 전략입니다. 대용량 테이블에 `ALTER TABLE`을 날리는 건 서비스 장애를 예약하는 것과 다름없습니다. Postgres의 **Heavyweight Lock** 때문이죠.

OpenAI는 마이그레이션 스크립트가 돌 때, 별도의 스크립트를 병렬로 실행해서 **해당 테이블에 락을 잡으려 하거나 충돌하는 트랜잭션을 강제로 Kill** 해버린다고 합니다. 

> "While running the schema rollout, run a script alongside it that kills any workload conflicting with the aggressive locks..."

이건 정말 현업에서 피 땀 흘려본 사람들만 아는 팁입니다. 보통은 `lock_timeout` 설정 정도만 하고 기도 메타로 가는데, 아예 능동적으로 블로킹 세션을 죽여버려서 마이그레이션 성공률을 높이는 전략이죠. 잠깐의 에러율 상승이 장시간의 락 대기보다 낫다는 판단입니다.

## 3. 왜 결국 CosmosDB인가? (Azure 광고인가, 현실인가)

글의 중반부부터는 "새로운 기능은 **Azure CosmosDB** 를 사용한다"는 내용이 나옵니다. 기존 데이터를 샤딩하는 건 애플리케이션 수백 군데를 고쳐야 하는 대공사(수개월~수년 소요)이기 때문에, 신규 기능부터 NoSQL 기반의 샤딩을 적용한다는 것이죠.

Hacker News 댓글 창은 이 부분에서 의견이 갈립니다.

- **비판:** "결국 Azure 광고 아니냐? Postgres도 샤딩(Partitioning + FDW) 되는데 굳이 비싼 CosmosDB를?"
- **옹호:** "이미 MS가 대주주인데 Azure 쓰는 게 당연하다. 그리고 Postgres 샤딩 관리 포인트보다 Managed NoSQL이 운영상 편한 건 팩트다."

제 생각은 이렇습니다. OpenAI 정도의 자금력이면 **CosmosDB** 의 살인적인 비용은 문제가 되지 않습니다. 오히려 그들에게 가장 비싼 자원은 '엔지니어의 시간'입니다. Postgres 샤딩을 직접 운영하며 겪을 운영 이슈(Resharding, Hotspot handling)를 돈으로 퉁칠 수 있다면 그게 남는 장사입니다.

## 4. 하드웨어 스펙에 대한 고찰

댓글 중 흥미로운 분석이 있었는데, OpenAI가 쓰고 있을 것으로 추정되는 Azure VM 스펙에 대한 이야기입니다. `Standard_M896ixds_24_v3` 같은 괴물 인스턴스는 코어만 896개, 메모리는 32TB에 달합니다. 월 비용만 수천만 원에서 억 단위가 넘어가죠.

우리가 여기서 얻어야 할 교훈은 명확합니다. **"소프트웨어 최적화보다 하드웨어 업그레이드가 싸다"** 는 격언은 여전히 유효합니다. 복잡한 아키텍처를 도입하기 전에, 지금 쓰고 있는 DB 인스턴스의 CPU와 메모리를 최대로 올렸는지부터 자문해봐야 합니다. 8억 명도 단일 노드로 버티는데, 우리 서비스가 벌써 마이크로서비스와 샤딩을 도입해야 할까요?

## 결론: 지루한 기술의 승리

이 아티클은 화려한 신기술 자랑이 아닙니다. 오히려 **기본기(Fundamental)** 에 대한 이야기입니다.

1.  **Postgres** 는 우리가 생각하는 것보다 훨씬 강력하다.
2.  **Pgbouncer** 와 같은 커넥션 풀링, 적절한 인덱싱, 그리고 쿼리 최적화로 웬만한 규모는 다 커버된다.
3.  샤딩은 최후의 최후까지 미뤄라.

OpenAI가 AI 모델은 최첨단을 달리지만, 인프라는 가장 보수적이고 검증된 **Postgres** 를 메인으로 쓴다는 점. 이게 바로 시니어 엔지니어가 지향해야 할 **Pragmatism(실용주의)** 아닐까요?

**Reference:**
- [Scaling PostgreSQL to power 800M users](https://openai.com/index/scaling-postgresql/)
- [Hacker News Discussion](https://news.ycombinator.com/item?id=46725300)
