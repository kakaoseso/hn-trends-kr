---
title: "Airbnb의 StatsD에서 OpenTelemetry 전환기: 왜 우리는 여전히 메트릭 파이프라인으로 고통받는가"
description: "Airbnb의 StatsD에서 OpenTelemetry 전환기: 왜 우리는 여전히 메트릭 파이프라인으로 고통받는가"
pubDate: "2026-04-16T15:15:50Z"
---

대규모 트래픽을 다루는 시스템에서 메트릭 파이프라인을 교체하는 것은 비행 중에 엔진을 교체하는 것과 같다. 최근 Airbnb에서 기존의 오래된 StatsD 기반 파이프라인을 OpenTelemetry와 vmagent 기반으로 전환한 경험을 공유했다. 원문 기사는 접근이 좀 까다로웠지만, Hacker News에서 오간 토론들을 보면 시니어 엔지니어들이 현업에서 겪는 진짜 고통(pain points)들이 고스란히 묻어난다.

15년 넘게 인프라와 백엔드 아키텍처를 설계해 온 입장에서, 이번 Airbnb의 결정과 커뮤니티의 반응은 꽤 흥미로웠다. 겉보기엔 단순한 도구 교체 같지만, 그 이면에는 분산 시스템의 복잡성과 기술적 부채가 얽혀 있기 때문이다.

### Sparse Counters와 Zero Injection: 우아한 문제 해결

이번 마이그레이션에서 가장 눈에 띄는 기술적 해결책은 'Sparse counter' 문제를 다룬 방식이다.

StatsD 같은 Delta 기반 시스템에서 Prometheus 같은 Cumulative 기반 시스템으로 넘어갈 때 거의 모든 팀이 피를 보는 구간이 바로 여기다. 이벤트가 간헐적으로 발생하는(sparse) 카운터의 경우, 이벤트가 없으면 메트릭이 아예 방출되지 않는다. Prometheus는 이전 baseline을 모르기 때문에 rate 계산이 완전히 꼬여버린다.

Airbnb는 이를 해결하기 위해 Aggregation tier에서 첫 flush 시점에 인위적으로 '0'을 주입(Synthetic zero injection)하는 방식을 택했다.

솔직히 말해, 이건 정말 우아한(elegant) 접근이다. 보통 이런 문제가 터지면 인프라 팀은 각 서비스 개발팀에게 "애플리케이션 초기화 시점에 카운터를 0으로 세팅해서 쏴주세요"라고 가이드를 내린다. 그리고 이건 100% 실패하는 방식이다. 수백 개의 마이크로서비스를 일일이 수정하는 것은 불가능에 가깝기 때문이다. 인프라 레벨에서 중앙집중적으로 이 엣지 케이스를 처리한 것은 Principal급의 훌륭한 의사결정이다.

HN의 한 유저가 이 이슈에 대해 "Wait, everyone just lives with this? What the fuck?!"이라고 반응한 것도 십분 이해가 간다. Prometheus 생태계에는 이런 '다들 그냥 참고 쓰는' 기묘한 관행들이 꽤 많고, 이를 정면으로 돌파한 Airbnb의 방식은 칭찬받아 마땅하다.

### Pull 모델의 한계와 OTLP Push의 선택

Prometheus 생태계의 철학은 기본적으로 Pull(Scrape) 방식이다. 하지만 Airbnb는 OTel receiver가 엔드포인트를 스크랩하게 두는 대신, 애플리케이션에서 OTLP로 직접 메트릭을 Push하는 방식을 선택했다.

Kubernetes 환경에서는 `/metrics` 엔드포인트를 열어두고 스크랩하는 게 국룰처럼 여겨지지만, 레거시가 방대한 조직에서는 이야기가 다르다. HN 댓글 중 10년 전에 Airbnb에 dogstatsd를 도입했던 엔지니어의 회고가 매우 인상 깊었다. 이미 수많은 서비스가 Push 방식(StatsD)으로 메트릭을 쏘고 있는 상황에서, 모든 앱의 코드를 수정해 Scrape 엔드포인트를 열게 하는 것은 미친 짓이다.

기술적 결정은 시간이 지남에 따라 복리(compound)로 돌아온다. OTLP Push는 레거시의 관성을 인정하면서도 현대적인 표준으로 나아가기 위해 찾은 최적의 타협점이었을 것이다.

### The Elephant in the Room: Prometheus vs VictoriaMetrics

이 글의 백미는 사실 HN 커뮤니티에서 벌어진 TSDB(Time Series Database) 논쟁이다. Airbnb는 수집기로 `vmagent`(VictoriaMetrics의 agent)를 썼고, 백엔드로는 Grafana Mimir를 선택한 것으로 보인다.

여기서 많은 엔지니어들이 공감하는 Prometheus의 치명적인 단점이 지적된다. 바로 **Startup Time** 이다.

"Prometheus는 안정적이다"라는 명제에 대해 한 유저는 강하게 반발했다. 나 역시 동의한다. 2TB 사이즈의 메트릭 데이터를 가진 Prometheus 인스턴스를 재시작해 본 적이 있는가? SSD를 써도 WAL(Write-Ahead Log)을 리플레이하며 기동하는 데 최대 2시간이 걸린다. 장애 상황에서 2시간의 복구 시간은 치명적이다.

반면 VictoriaMetrics(VM)는 어떨까? HN 유저의 말마따나 "그냥 켜진다(just... starts)".

- **성능:** 초당 200만 개의 샘플을 긁어오는 상황에서도 VM은 놀라운 퍼포먼스와 안정성을 보여준다.
- **운영 비용:** 유지보수 비용(support investment) 측면에서 기존 Prometheus와는 비교가 안 될 정도로 낮다.

나 역시 최근 구축하는 대규모 클러스터에서는 Prometheus 대신 VictoriaMetrics를 기본으로 채택하고 있다. 그렇다면 Airbnb는 왜 VM Cluster 대신 Mimir를 썼을까? Mimir의 강력한 멀티테넌시(Multi-tenancy) 기능이나 Grafana 생태계와의 완벽한 통합 때문일 확률이 높지만, 운영 복잡도를 생각하면 VM Cluster가 더 나은 선택이 아니었을까 하는 아쉬움은 남는다.

### 결론: 우리는 어디로 가야 하는가

메트릭 파이프라인은 겉보기엔 단순해 보이지만, Scale이 커지는 순간 온갖 분산 시스템의 난제들이 튀어나오는 마굴이다.

Airbnb의 이번 사례는 단순히 "새로운 툴을 도입했다"가 아니라, Delta에서 Cumulative로의 패러다임 전환, 그리고 레거시 Push 모델을 OTel이라는 현대적인 표준으로 매끄럽게(seamlessly) 이어붙인 훌륭한 엔지니어링 사례다.

결론적으로, OpenTelemetry는 이제 완벽히 Production-ready 상태다. 다만, 백엔드 스토리지를 선택할 때는 Prometheus의 과거 명성에만 기대지 말고, VictoriaMetrics나 Mimir 같은 차세대 TSDB들을 여러분의 실제 워크로드에 맞게 면밀히 벤치마킹해보길 권한다. 맹목적인 기술 도입만큼 위험한 것은 없다.

---

### References
- **Original Article:** [Building a High-Volume Metrics Pipeline with OpenTelemetry and vmagent](https://medium.com/airbnb-engineering/building-a-high-volume-metrics-pipeline-with-opentelemetry-and-vmagent-c714d6910b45)
- **Hacker News Thread:** [Discussion on Airbnb's Metrics Pipeline](https://news.ycombinator.com/item?id=47788818)
