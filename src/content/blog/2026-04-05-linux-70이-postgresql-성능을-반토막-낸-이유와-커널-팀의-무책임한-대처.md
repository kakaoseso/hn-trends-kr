---
title: "Linux 7.0이 PostgreSQL 성능을 반토막 낸 이유와 커널 팀의 무책임한 대처"
description: "Linux 7.0이 PostgreSQL 성능을 반토막 낸 이유와 커널 팀의 무책임한 대처"
pubDate: "2026-04-05T01:57:00Z"
---

최근 Phoronix 기사를 읽다가 제 눈을 의심했습니다. 인프라 엔지니어에게 커널 업데이트는 보통 성능 향상이나 보안 패치를 의미합니다. 그런데 최신 커널이 세계에서 가장 많이 쓰이는 관계형 데이터베이스의 성능을 정확히 반토막 낸다면 어떨까요?



AWS의 Salvatore Dipietro가 보고한 바에 따르면, Graviton4 서버 환경의 Linux 7.0에서 PostgreSQL의 Throughput이 이전 커널 대비 **0.51배** 수준으로 폭락했습니다. Latency 역시 심각하게 튀었고요. 15년 넘게 백엔드와 인프라 아키텍처를 다뤄온 제 입장에서, 이건 단순한 성능 저하(Regression)가 아니라 사실상 장애(Outage)입니다.

## 왜 이런 일이 발생했는가?

문제의 원인은 Linux 7.0 스케줄러 업데이트에 포함된 Preemption(선점) 모델 변경입니다. 

기존 서버 환경, 특히 데이터베이스처럼 Throughput이 극도로 중요한 워크로드에서는 주로 PREEMPT_NONE을 사용해 왔습니다. 불필요한 컨텍스트 스위칭을 막고 CPU 사이클을 최대한 애플리케이션에 몰아주기 위해서죠. 그런데 커널 팀이 최신 CPU 아키텍처에 맞춰 Full & Lazy Preemption 모델에 집중하기로 하면서 상황이 꼬였습니다.

결과적으로 PostgreSQL처럼 User-space spinlock을 하드코어하게 사용하는 애플리케이션에서 Lock holder preemption이 빈번하게 발생하기 시작했습니다. 스레드가 락을 쥔 상태에서 선점당해버리니, 다른 스레드들은 의미 없이 엄청난 시간을 Spinlock 대기에 낭비하게 된 것입니다.

## 커널 팀의 황당한 대처: "너희가 코드를 고쳐라"

이 치명적인 문제를 해결하기 위해 PREEMPT_NONE을 다시 기본값으로 되돌리자는 패치가 LKML(Linux Kernel Mailing List)에 올라왔습니다. 상식적으로 당연한 수순이죠.

하지만 기존 코드를 단순화했던 커널 개발자 Peter Zijlstra의 답변은 가관이었습니다. 그는 PostgreSQL이 Linux 7.0에 추가된 **RSEQ (Restartable Sequences)** time slice extension을 사용하도록 코드를 수정하는 것이 올바른 Fix라고 주장했습니다.

솔직히 말씀드리면, 이건 전형적인 상아탑 엔지니어링의 오만입니다. Linus Torvalds가 과거부터 그토록 강조하던 "Never break userspace" 원칙은 도대체 어디로 간 걸까요? 

거대한 레거시를 가진 PostgreSQL이 하루아침에 RSEQ 기반으로 락킹 메커니즘을 갈아엎을 수 있을 리 만무합니다. 새로운 기술이 더 우수할 수는 있지만, 충분한 Deprecation 기간과 하위 호환성(Fallback) 없이 프로덕션 생태계를 절벽으로 밀어붙이는 것은 최악의 안티 패턴입니다.

## 커뮤니티의 반응과 현실적인 여파

Hacker News 스레드 역시 이 문제로 뜨겁게 달아올랐습니다. 

- **Andres Freund:** PostgreSQL 코어 개발자인 그는 LKML에서 커널 팀의 접근 방식을 강하게 비판하며 현실적인 문제점들을 조목조목 지적했습니다.
- **Ubuntu 26.04 LTS:** 가장 큰 문제는 타이밍입니다. 4월에 출시될 이 LTS 버전이 바로 Linux 7.0 커널을 탑재합니다. 수많은 기업들이 "안정적인 LTS"라는 이유로 업그레이드를 진행했다가 DB 성능이 반토막 나는 악몽을 겪게 될 것입니다.
- **Workaround:** 결국 시스템 관리자들이 부팅 파라미터나 커널 패치를 통해 강제로 Preemption을 끄는 해프닝이 벌어질 것입니다.

## 결론: 당분간 커널 업데이트는 보류하십시오

만약 여러분이 대규모 트래픽을 처리하는 PostgreSQL 클러스터를 운영 중이라면, 당분간 Linux 7.0(혹은 이를 기반으로 하는 배포판) 도입은 절대적으로 피해야 합니다. 

기술의 진보는 언제나 환영하지만, 그 과정이 사용자에게 재앙을 초래해서는 안 됩니다. RSEQ가 훌륭한 메커니즘임은 부정하지 않겠습니다. 하지만 커널은 인프라의 근간이며, 근간이 흔들리면 그 위에서 돌아가는 모든 비즈니스가 타격을 받습니다. 

커널 팀이 고집을 꺾고 현실적인 유예 기간을 제공할지, 아니면 결국 DB 엔지니어들이 밤을 새워가며 커스텀 튜닝을 해야 할지 당분간 LKML의 동향을 예의주시해야 할 것 같습니다.

## References
- Original Article: https://www.phoronix.com/news/Linux-7.0-AWS-PostgreSQL-Drop
- Hacker News Thread: https://news.ycombinator.com/item?id=47644864
