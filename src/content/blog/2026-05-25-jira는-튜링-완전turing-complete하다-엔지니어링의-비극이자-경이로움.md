---
title: "Jira는 튜링 완전(Turing-complete)하다: 엔지니어링의 비극이자 경이로움"
description: "Jira는 튜링 완전(Turing-complete)하다: 엔지니어링의 비극이자 경이로움"
pubDate: "2026-05-25T07:32:47Z"
---

솔직히 말해서, 우리 모두 Jira를 좋아하면서도 동시에 혐오합니다.

15년 넘게 엔지니어로 일하면서 수많은 조직을 거쳤지만, Jira가 없는 회사는 거의 없었고, 동시에 Jira 설정이 완벽하게 되어 있는 회사도 단 한 곳도 없었습니다. 그런데 최근 Hacker News에서 아주 흥미로운(그리고 약간은 끔찍한) 글을 하나 발견했습니다. 누군가 농담으로만 떠돌던 "Jira는 튜링 완전(Turing-complete)하다"는 가설을 수학적으로 증명해 낸 것입니다.

단순한 밈(Meme)이 아닙니다. 이 글의 저자는 Atlassian Automation을 이용해 Minsky Machine을 완벽하게 구현해 냈습니다. 오늘은 이 미친 짓이 어떻게 가능한지 기술적으로 파헤쳐보고, 이것이 우리의 엔지니어링 문화에 시사하는 바를 짚어보겠습니다.

### Minsky Machine을 Jira에 이식하다

어떤 시스템이 튜링 완전함을 증명하는 가장 우아하고 확실한 방법은, 이미 튜링 완전하다고 증명된 다른 모델을 해당 환경에서 구현해 내는 것입니다. 저자는 1967년 Marvin Minsky가 고안한 Minsky register machine을 선택했습니다.

이 머신은 단 두 개의 무한한 카운터(레지스터)와 유한한 명령어 셋(INC, DEC, 조건부 GOTO)만 있으면 동작합니다. 저자는 이 단순한 구조를 Jira의 도메인 모델에 기가 막히게 매핑했습니다.

- **Register A:** Epic에 링크된 'Bug' 타입 이슈의 총 개수
- **Register B:** Epic에 링크된 'Task' 타입 이슈의 총 개수
- **Program Counter:** 해당 Epic의 현재 Status (예: BACKLOG, TODO, DEV, PROD)
- **Dispatch Table:** 각 상태(Status)마다 트리거되는 Jira Automation Rules
- **Clock:** Automation이 트리거하는 상태 전이(Transition) 이벤트

예를 들어, 레지스터 A의 값을 레지스터 B로 더하는 프로그램은 논리적으로 다음과 같이 작성됩니다.

```text
1. DEC A; if A == 0 goto 3 else goto 2
2. INC B; goto 1
3. HALT
```

이걸 Jira Automation으로 구현하면 어떻게 될까요?
Epic의 상태가 `TODO`일 때, 링크된 Bug(A)를 하나 지우고(DEC), 만약 남은 Bug가 0개면 `PROD`(HALT)로 상태를 변경하고, 아니면 `DEV`로 변경합니다. `DEV` 상태에서는 Task(B)를 하나 생성(INC)하고 다시 `TODO`로 돌아가는 룰을 만듭니다.

결과적으로 Epic이 `PROD` 상태에 도달했을 때 링크된 Task의 개수를 세면 덧셈 연산이 완료된 것입니다. 심지어 저자는 이슈 타입을 즉시 변경하는 `CONVERT` 액션을 이용해 상태 전이 테이블을 최적화하고 피보나치 수열까지 구현해 냈습니다. Jira Cloud의 Automation 체인 제한(10회) 때문에 사람이 수동으로 다시 트리거(Clock tick)를 줘야 하긴 하지만, 수학적 환원(Reduction)의 관점에서는 완벽한 튜링 완전성의 증명입니다.

### "그래서 내 Jira가 그렇게 느렸군" (HN 반응과 나의 생각)

이 글을 읽고 Hacker News 커뮤니티는 그야말로 폭발했습니다. 한 유저의 댓글이 이 상황을 완벽하게 요약해 줍니다.

> "That explains why it's impossible to tell whether any given Jira operation is going to halt or not." (이래서 Jira의 특정 작업이 언제 끝날지 알 수 없었던 거군요.)

컴퓨터 과학의 난제인 정지 문제(Halting Problem)를 빗댄 최고의 농담이 아닐 수 없습니다.

현업에서 뛰는 시니어 엔지니어로서 저는 이 상황이 꽤 씁쓸하면서도 깊이 공감됩니다. Jira는 단순한 이슈 트래커로 시작했지만, 엔터프라이즈의 복잡한 요구사항(권한, 워크플로우, 감사 로그 등)을 모두 수용하다 보니 거대한 'Workflow Orchestration Engine'이자 사실상의 비주얼 프로그래밍 언어가 되어버렸습니다.

HN의 다른 유저가 지적했듯, Jira API를 이용해 자동화를 구축하는 것은 "프랙탈 구조의 똥 눈송이(fractal shit snowflake)"를 다루는 것과 같습니다. 수많은 마이그레이션과 커스텀 필드가 겹겹이 쌓여, UI에서는 보이지 않는 `custom_field_5537` 같은 매직 스트링을 API에 던져야만 겨우 동작하는 끔찍한 경험을 우리 모두 해본 적이 있을 겁니다.

최근에는 이 고통을 회피하기 위해 아예 AI를 도입하는 팀들이 늘고 있습니다. 저 역시 최근 사내에서 LLM과 연동된 MCP(Model Context Protocol)를 구축했습니다. 이제 개발자들은 끔찍하게 느리고 레이아웃이 튀는 Jira UI를 클릭하는 대신, 에디터에서 자연어로 "이 스펙 문서 기반으로 Jira 에픽 파고 하위 테스크 5개로 쪼개줘"라고 명령합니다. 사람이 쓰기엔 너무 무겁고 복잡한 시스템이 되었기에, 이제는 AI라는 추상화 레이어를 하나 더 얹어야만 쓸만한 도구가 된 셈입니다.

### Conclusion: 우리는 튜링 완전한 이슈 트래커가 필요한가?

기술적으로 Jira가 튜링 완전하다는 사실은 매우 훌륭한 지적 유희입니다. 하지만 엔지니어링 리더의 관점에서 보면, 이는 도구의 복잡성이 우리의 통제 범위를 넘어섰다는 적신호이기도 합니다.

우리가 티켓 하나를 '진행 중'으로 옮기는 데 튜링 완전한 상태 머신과 무한 루프 가능성을 내포한 룰 엔진이 정말 필요할까요? 때로는 단순함이 최고의 미덕입니다. 우리가 과거에 Trello나 최근의 Linear 같은 가볍고 오피니언이 강한(Opinionated) 도구들에 열광하는 이유도 시스템의 복잡도를 낮추고 싶어 하는 본능 때문일 것입니다.

물론, 내일 출근하면 저는 다시 Jira에 접속해 스프린트 백로그를 정리해야 할 겁니다. 하지만 적어도 이제는 Epic의 Status가 바뀔 때마다 백그라운드에서 보이지 않는 Minsky Machine이 돌아가고 있다는 사실에 묘한 경외감을 느낄 것 같네요. 누군가 Jira Automation 위에서 DOOM을 포팅하는 그날을 진심으로 기대해 봅니다.

### References
- **Original Article:** https://seriot.ch/computation/jira.html
- **Hacker News Thread:** https://news.ycombinator.com/item?id=48263253
