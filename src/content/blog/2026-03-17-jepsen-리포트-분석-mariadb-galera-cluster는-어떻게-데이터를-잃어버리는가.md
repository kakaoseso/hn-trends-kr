---
title: "Jepsen 리포트 분석: MariaDB Galera Cluster는 어떻게 데이터를 잃어버리는가"
description: "Jepsen 리포트 분석: MariaDB Galera Cluster는 어떻게 데이터를 잃어버리는가"
pubDate: "2026-03-17T09:19:36Z"
---

# Jepsen 리포트 분석: MariaDB Galera Cluster는 어떻게 데이터를 잃어버리는가

분산 시스템을 설계하고 운영해 본 시니어 엔지니어라면, 벤더사의 공식 문서에 적힌 "완벽한 Active-Active", "Zero Data Loss", "동기식 복제(Synchronous Replication)" 같은 마케팅 용어를 볼 때마다 본능적인 의심이 들 겁니다. 세상에 공짜는 없고, CAP 정리와 PACELC 정리는 벤더사의 마케팅 부서가 무시한다고 해서 사라지는 물리 법칙이 아니니까요.

최근 분산 시스템 검증의 끝판왕인 Kyle Kingsbury(Aphyr)가 이끄는 Jepsen에서 [MariaDB Galera Cluster 12.1.2에 대한 분석 리포트](https://jepsen.io/analyses/mariadb-galera-cluster-12.1.2)를 공개했습니다. 결론부터 말씀드리자면, 참담한 수준입니다. 공식 문서의 화려한 약속과 달리, Galera Cluster는 기본적인 Read Uncommitted 격리 수준조차 제대로 보장하지 못했습니다.

오늘은 15년 차 엔지니어의 관점에서 이 리포트가 의미하는 바가 무엇인지, 그리고 우리가 왜 Multi-master RDBMS 도입에 극도로 보수적이어야 하는지 파헤쳐보겠습니다.

## 동기식 복제라는 환상

MariaDB Galera Cluster는 오랫동안 MySQL/MariaDB 생태계에서 Active-Active Multi-master 구성을 위한 은탄환처럼 여겨져 왔습니다. 공식 문서에서는 Galera가 트랜잭션이 모든 노드에서 인증(certification)을 통과해야만 커밋된 것으로 간주된다며 동기식 복제를 강조합니다.

하지만 Jepsen 리포트는 이 주장이 명백한 거짓(Obviously wrong)이라고 지적합니다. 만약 정말로 모든 노드의 동의가 필요하다면 단 하나의 노드만 죽어도 클러스터 전체의 쓰기 작업이 멈춰야 합니다. 실제로는 Quorum(정족수) 기반으로 동작하며, 소수 노드의 장애를 허용합니다. 여기까지는 일반적인 분산 합의 알고리즘(Raft, Paxos)과 다를 바 없습니다. 문제는 그 이면에 숨겨진 데이터 정합성 결함입니다.

## Jepsen이 발견한 치명적인 결함들

Jepsen은 Elle라는 강력한 트랜잭션 격리 수준 검증 도구를 사용해 Galera Cluster를 테스트했습니다. 그리고 다음과 같은 충격적인 결과들을 찾아냈습니다.

### 1. Coordinated Crash 시의 데이터 유실 (MDEV-38974)
MariaDB는 Galera Cluster 구성 시 성능을 위해 innodb_flush_log_at_trx_commit=0 설정을 권장해 왔습니다. 노드 하나가 죽어도 다른 노드에서 복구하면 되니까 로컬 디스크 동기화는 끄고 성능을 높이라는 논리죠.

하지만 데이터센터 전원 장애, 네트워크 스위치 버그 등으로 인해 클러스터의 모든 노드가 거의 동시에 죽는 Coordinated Crash가 발생하면 어떻게 될까요? Jepsen 테스트 결과, 클라이언트에게 성공적으로 커밋되었다고 응답한 트랜잭션들이 재시작 후 허무하게 증발해버렸습니다. 벤더사가 벤치마크 성능을 높이기 위해 얼마나 무책임한 기본 설정을 권장하는지 보여주는 전형적인 사례입니다.

### 2. 장애 상황에서의 추가적인 데이터 유실 (MDEV-38976)
그렇다면 안전하게 innodb_flush_log_at_trx_commit=1 로 설정하면 모든 문제가 해결될까요? 안타깝게도 아닙니다. 이 설정을 켜더라도 프로세스 크래시와 Network Partition이 겹치는 상황에서는 여전히 커밋된 데이터가 날아갔습니다. 발생 빈도가 낮다고는 하지만, 결제나 원장 시스템에서는 단 한 건의 데이터 유실도 치명적입니다.

### 3. 정상 상태에서의 Lost Update와 Stale Read (MDEV-38977, MDEV-38999)
제가 이번 리포트에서 가장 경악한 부분입니다. 네트워크 단절이나 노드 장애를 주입하지 않은 건강한 클러스터(Healthy cluster)에서도 데이터 정합성이 깨졌습니다.

- **Lost Update (P4):** 트랜잭션 T1이 데이터를 읽고, T2가 업데이트를 한 뒤, T1이 이전 읽기 결과를 바탕으로 업데이트를 덮어써버리는 현상입니다. ORM에서 흔히 사용하는 Read-Modify-Write 패턴이 Galera 위에서는 언제든 박살 날 수 있다는 뜻입니다.
- **Stale Read:** T1이 커밋 완료 응답을 받은 후 시작된 T2가 T1의 변경 사항을 읽지 못하는 현상입니다. Galera가 자랑하는 지연 없는 즉각적인 복제가 허구임이 드러났습니다.

Hacker News의 한 유저도 이 부분을 정확히 짚었습니다. "정상 클러스터에서 이런 현상이 발생한다는 건 아주 근본적인 문제(fundamental issue)로 보인다"라고요.

## 우리는 여기서 무엇을 배워야 하는가?

Hacker News 스레드에서 눈에 띄는 또 다른 댓글이 있었습니다. "MySQL은 애초에 분산 데이터베이스로 설계되지 않았고, 나중에 억지로 분산 기능을 끼워 맞추는 건 항상 어렵다."

전적으로 동의합니다. 단일 노드 RDBMS의 아키텍처 위에 Virtual Synchrony와 Certification 기반의 복제를 얹어 Multi-master를 흉내 내는 것은 필연적으로 한계에 부딪힙니다.

물론 누군가는 트래픽이 적은 사내 메일 서버 계정 관리에 10년째 Galera를 잘 쓰고 있다고 말합니다. 하지만 트래픽이 적다면 굳이 복잡하고 불안정한 Multi-master가 필요할까요? 단순한 Active-Standby 고가용성(HA) 구성으로 충분합니다. 반대로 트래픽이 많고 정합성이 중요하다면? Jepsen이 증명했듯 Galera는 그 부하와 예외 상황을 일관성 있게 처리하지 못합니다. 결국 Multi-master의 존재 의의 자체가 모순에 빠지는 셈입니다.

## 총평: 프로덕션 레벨에서의 도입은?

단호하게 말씀드립니다. 돈이 오가거나, 상태 머신의 정확성이 100% 보장되어야 하는 핵심 도메인에 MariaDB Galera Cluster를 도입하는 것은 시한폭탄을 안고 가는 것과 같습니다. Snapshot Isolation이나 Repeatable Read는커녕, Read Uncommitted보다도 못한 일관성을 제공하는 시스템을 믿고 비즈니스 로직을 짤 수는 없습니다.

강력한 일관성과 분산 확장이 필요하다면, 처음부터 분산 트랜잭션과 합의 알고리즘(Raft)을 염두에 두고 설계된 CockroachDB나 TiDB 같은 진정한 분산 SQL 데이터베이스를 검토하시기 바랍니다. 레거시 RDBMS에 날개를 달아준다는 달콤한 마케팅 용어에 더 이상 속지 맙시다.

---

- **Original Article:** https://jepsen.io/analyses/mariadb-galera-cluster-12.1.2
- **Hacker News Thread:** https://news.ycombinator.com/item?id=47408360
