---
title: "Postgres의 오래된 족쇄 FPW, 분산 스토리지로 해결하다: 쓰기 성능 5배 향상의 비밀"
description: "Postgres의 오래된 족쇄 FPW, 분산 스토리지로 해결하다: 쓰기 성능 5배 향상의 비밀"
pubDate: "2026-05-10T18:22:09Z"
---

대규모 트래픽을 다루는 엔지니어라면 Postgres 환경에서 Write-heavy 워크로드를 운영할 때 겪는 고질적인 고통을 잘 알고 있을 것이다. Checkpoint가 발생할 때마다 미친 듯이 치솟는 디스크 I/O, 그리고 기하급수적으로 팽창하는 WAL(Write-Ahead Log) 파일들. 우리는 이를 완화하기 위해 디스크를 업그레이드하거나 Checkpoint 주기를 튜닝하는 등 온갖 똥꼬쇼를 해왔다.

최근 Databricks(정확히는 그들이 인수한 Neon 팀) 블로그에 매우 흥미로운 글이 올라왔다. **Postgres의 쓰기 성능(Throughput)을 5배 향상시키고, WAL 트래픽을 94% 감소시켰다** 는 내용이다. 처음 제목만 봤을 때는 흔한 마케팅용 벤치마크 뻥튀기인 줄 알았으나, 아키텍처를 뜯어보니 내가 2010년대 후반부터 고민해 오던 'Compute와 Storage의 완벽한 분리'가 가져다주는 구조적 이점을 극단적으로 잘 활용한 훌륭한 엔지니어링 사례였다.

오늘은 이들이 어떻게 Postgres의 10년 묵은 병목인 FPW(Full Page Write)를 제거했는지, 그리고 이것이 실제 프로덕션 레벨에서 어떤 의미를 가지는지 딥다이브 해보자.

## 모놀리식 Postgres의 원죄: Torn Page와 FPW

이 최적화를 이해하려면 먼저 Postgres가 왜 그렇게 많은 WAL을 생성하는지 알아야 한다.

Postgres는 데이터를 8KB 크기의 Page 단위로 메모리(Buffer Cache)에서 관리하고 디스크에 플러시한다. 문제는 하드웨어나 OS 레벨의 원자적 쓰기(Atomic Write) 단위가 보통 512바이트나 4KB라는 점이다. 만약 Postgres가 8KB 페이지를 디스크에 쓰는 도중 서버에 크래시가 발생하면 어떻게 될까? 페이지의 절반만 디스크에 기록되는 끔찍한 상황, 즉 **Torn Page(찢어진 페이지)** 현상이 발생한다.

이 상태에서 DB가 재시작되어 WAL의 작은 변경 사항(Delta)들을 Torn Page 위에 Replay 하려고 하면 데이터는 영구적으로 손상된다. 

Postgres는 이를 방지하기 위해 **FPW(Full Page Write)** 라는 무식하지만 확실한 방법을 사용한다. Checkpoint 이후 특정 페이지가 *처음* 수정될 때, 변경된 데이터만 WAL에 적는 것이 아니라 **8KB 페이지 전체를 통째로 WAL에 복사** 해 버린다. 크래시가 나면 디스크의 Torn Page를 무시하고 WAL에 백업된 온전한 8KB 페이지를 가져와 복구를 시작하는 것이다.

안전하다. 하지만 쓰기 볼륨이 높은 환경에서는 재앙이다. 고작 10바이트짜리 레코드를 업데이트했는데 WAL에는 8KB가 기록된다. WAL 볼륨이 최대 15배까지 팽창하며 시스템의 가장 큰 병목이 된다.

## Lakebase 아키텍처: 로컬 디스크가 없다면 Torn Page도 없다

Databricks와 Neon이 구축한 Lakebase 아키텍처의 핵심은 Compute와 Storage의 완전한 분리다. Postgres Compute 노드는 Stateless 상태이며, 로컬 데이터 디렉터리에 의존하지 않는다. 대신 WAL을 Paxos 기반의 Safekeeper 노드들로 스트리밍할 뿐이다.

여기서 천재적인 발상의 전환이 일어난다. **"Compute 노드에 로컬 디스크가 없으니, 디스크에 쓰다가 발생하는 Torn Page 문제도 애초에 존재하지 않는 것 아닌가?"**

맞다. 실패 모드 자체가 사라졌으니 Compute 노드에서 FPW를 켜둘 이유가 없다. 이론상 `full_page_writes = off`로 설정하기만 하면 WAL 팽창 문제는 즉시 해결된다.

## 새로운 문제: 무한한 Delta Chain과 Read Latency

하지만 시스템 설계에서 공짜 점심은 없다. FPW를 단순히 꺼버리면 읽기(Read) 성능에서 치명적인 문제가 발생한다.

Hacker News 스레드에서 한 유저가 날카로운 질문을 던졌다.
> "읽기 요청은 WAL이 아니라 Buffer Cache에서 처리되는데, FPW를 끄는 것이 왜 읽기 지연(Read Latency)을 유발한다는 건가요?"

모놀리식 Postgres에서는 이 유저의 말이 맞다. 하지만 Compute와 Storage가 분리된 환경에서는 이야기가 다르다. Compute 노드의 Buffer Cache에 원하는 페이지가 없다면(Cache Miss), Compute는 Storage 계층(Pageserver)에 해당 페이지를 요청해야 한다.

FPW가 켜져 있을 때는 WAL 안에 주기적으로 8KB 전체 이미지(Base Image)가 포함되어 있어, Storage가 페이지를 재구성할 때 최근의 Base Image를 찾고 그 이후의 짧은 Delta들만 Replay 하면 됐다. 즉, 복구 비용이 `O(Checkpoint 주기)`로 제한된다.

하지만 FPW를 꺼버리면 WAL에는 끝없는 Delta(작은 변경점)들만 남게 된다. Storage 계층이 특정 페이지를 서빙하려면 수천, 수만 개의 Delta Chain을 처음부터 끝까지 Replay 해야 한다. 결과적으로 Read Latency가 미친 듯이 튀고 리소스가 고갈된다.

## 해결책: Image Generation Pushdown

Databricks/Neon 팀은 이 딜레마를 **Image Generation Pushdown** 이라는 기법으로 해결했다. FPW의 책임을 Compute 노드에서 Storage 계층으로 밀어내버린 것이다.

이제 Compute 노드는 순수하게 변경된 Delta만 WAL로 스트리밍한다. WAL 트래픽은 극단적으로 가벼워진다. 반면 분산 스토리지인 Pageserver는 이 Delta들을 수신하다가, 특정 페이지에 대한 Delta가 **설정된 임계치(Threshold)를 초과하면 자체적으로 8KB 전체 이미지를 Materialize(생성) 해버린다.**

이 접근법이 기존 Postgres의 Checkpoint 기반 FPW보다 훨씬 우수한 이유는 명확하다. 단순히 시간이 지났다고 무의미하게 전체 페이지를 쓰는 것이 아니라, **해당 페이지가 실제로 얼마나 많이 변경되었는지** 를 기준으로 이미지를 생성하기 때문이다.

## 압도적인 벤치마크 결과

결과는 숫자가 증명한다. HammerDB TPROC-C 벤치마크 결과를 보면 입이 떡 벌어진다.

![](https://www.databricks.com/sites/default/files/inline-images/image_106.png?v=1778165943)

- **4-vCPU:** 20% 성능 향상
- **16-vCPU:** 2.8배 성능 향상
- **32-vCPU:** 4.5배 이상 성능 향상 (439,300 NOPM)

Compute 리소스가 커질수록 성능 향상 폭이 극적으로 증가한다. 기존에는 디스크 I/O와 WAL 락(Lock) 경합 때문에 CPU를 늘려도 Throughput이 선형적으로 증가하지 못했다. 병목이 제거되니 비로소 하드웨어의 잠재력을 100% 끌어다 쓸 수 있게 된 것이다.

![](https://www.databricks.com/sites/default/files/inline-images/image_107.png?v=1778165943)

트랜잭션당 평균 WAL 크기는 58Kb에서 4Kb 미만으로 **94% 감소** 했다. 실제 한 프로덕션 환경(56 vCPU)에서는 초당 WAL 생성량이 30MB/s에서 1MB/s로 줄었다고 한다. 네트워크 대역폭, 스토리지 인제스트 비용, 디스크 I/O 모두가 획기적으로 절약된 것이다.

게다가 Delta Chain이 최적화되면서 Storage 계층에서 페이지를 읽어올 때 적용해야 하는 WAL 레코드 수가 줄어들어, p99 Read Latency 마저 30~50% 감소하는 효과를 얻었다.

## 결론 및 엔지니어로서의 시각

솔직히 말해, 나는 AWS Aurora가 Redo Log 처리를 스토리지로 밀어내어(Log is the database) 혁신을 이뤘을 때와 비슷한 수준의 감명을 받았다. Aurora가 독점적인 인프라 위에서 마법을 부렸다면, Neon과 Databricks는 이를 오픈소스 생태계와 결합하여 우아하게 풀어냈다.

이 기능은 현재 모든 Lakebase Serverless 및 Neon 데이터베이스에 글로벌로 배포되었다고 한다. 더욱 인상적인 것은 기존 `XLOG_FPW_CHANGE` WAL 레코드 메커니즘을 활용해 고객의 다운타임이나 재시작 없이 컨트롤 플레인에서 이 변경 사항을 라이브로 적용했다는 점이다. 분산 시스템을 운영해 본 사람이라면 이것이 얼마나 까다로운 작업인지 알 것이다.

모놀리식 DB의 한계는 명확해지고 있다. 컴퓨팅은 컴퓨팅만 하고, 스토리지는 스토리지의 역할(데이터 재구성 및 스냅샷)만 수행하는 아키텍처. 만약 당신의 팀이 여전히 대규모 Write 워크로드 때문에 Postgres Checkpoint 튜닝에 밤을 새우고 있다면, 이제는 아키텍처의 패러다임을 바꿀 때가 온 것일지도 모른다. Postgres의 Write Tax는 이제 공식적으로 과거의 유물이 되었다.

---

**References:**
- [Databricks Blog: Solving the structural performance bottleneck in managed Postgres](https://www.databricks.com/blog/how-lakebase-architecture-delivers-5x-faster-postgres-writes)
- [Hacker News Discussion](https://news.ycombinator.com/item?id=48064789)
