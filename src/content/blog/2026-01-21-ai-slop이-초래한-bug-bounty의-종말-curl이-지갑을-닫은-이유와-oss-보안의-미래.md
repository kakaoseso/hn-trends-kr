---
title: "AI Slop이 초래한 Bug Bounty의 종말: cURL이 지갑을 닫은 이유와 OSS 보안의 미래"
description: "AI Slop이 초래한 Bug Bounty의 종말: cURL이 지갑을 닫은 이유와 OSS 보안의 미래"
pubDate: "2026-01-21"
---

### 서론: 인센티브 구조의 붕괴

오픈 소스 생태계에서 가장 성공적인 보안 모델 중 하나였던 '공개 버그 바운티(Bug Bounty)'가 생성형 AI(GenAI)라는 변수를 만나 붕괴하고 있습니다. 전 세계 수십억 개의 디바이스에서 사용되는 핵심 라이브러리인 **cURL**이 1월 말을 기점으로 금전적 보상을 지급하는 버그 바운티 프로그램을 종료한다고 선언했습니다.

이는 단순한 정책 변경이 아닙니다. **LLM(Large Language Model)이 만들어낸 낮은 품질의 'AI Slop(AI가 쏟아내는 쓰레기 데이터)'이 인간 메인테이너의 검증 대역폭(Bandwidth)을 초과해버린, 일종의 '인적 자원에 대한 DoS(Denial of Service)' 사태**입니다.

본 아티클에서는 cURL의 메인테이너 Daniel Stenberg와 보안 연구원 Joshua Rogers의 분석을 바탕으로, 왜 금전적 인센티브가 오히려 독이 되었는지, 그리고 고밀도 기술 조직이 이를 어떻게 바라봐야 하는지 심층 분석합니다.

---

### 1. 현상: "Death by a thousand slops"

Daniel Stenberg는 현재 상황을 "수천 개의 오물에 의한 죽음([Death by a thousand slops](https://daniel.haxx.se/blog/2025/07/14/death-by-a-thousand-slops/))"이라고 표현했습니다. 핵심 문제는 **신호 대 잡음비(SNR)의 급격한 하락**입니다.

*   **기존 모델:** 보안 연구원이 코드를 분석(높은 비용) → 유효한 버그 발견 → 리포트 → 보상. (리포트 비용이 높으므로 신중함)
*   **현재 모델:** AI에게 코드 주입 → 환각(Hallucination)이 섞인 리포트 자동 생성(비용 0에 수렴) → 무차별 제출 → 메인테이너의 검증(높은 비용).

메인테이너는 제출된 리포트가 AI가 쓴 헛소리인지, 실제 심각한 힙 오버플로우(Heap Overflow)인지 판단하기 위해 여전히 동일한 수준의 고도화된 디버깅 시간을 투입해야 합니다. Stenberg는 "AI가 생성한 리포트의 대다수는 완전히 넌센스(pure nonsense)이지만, 이를 판별하는 데 너무 많은 시간이 소요된다"고 토로했습니다.

![]()
*(Daniel Stenberg)*

물론 모든 AI 리포트가 무의미한 것은 아닙니다. Stenberg 역시 100건 이상의 AI 보조 리포트가 실제 수정으로 이어졌음을 인정합니다. 그러나 '쓰레기 리포트'를 걸러내는 비용이 '유효한 버그'를 찾는 이득을 상회하는 **임계점(Tipping Point)**을 넘어섰다는 것이 이번 결정의 본질입니다.

---

### 2. 경제적 비대칭성 (Economic Asymmetry)

유명 보안 연구원이자 버그 헌터인 Joshua Rogers는 이번 cURL의 결정을 두고 "진작에 이루어졌어야 할 조치"라며 강력히 지지했습니다. 그는 이 문제의 근본 원인을 **글로벌 경제적 비대칭성**에서 찾습니다.

> "개발자와 소위 '보안 연구원' 사이에는 비대칭적 관계가 존재합니다. 저소득 국가의 리포터에게는 스웨덴(cURL 본거지)에서의 점심값 정도 되는 보상이 거대한 금액일 수 있습니다."

이러한 경제적 격차는 다음과 같은 악순환을 만듭니다:
1.  **Low Effort, High Variance:** 리포터는 AI 도구를 돌려 수백 개의 리포트를 보냅니다. 99%가 거절당해도 1%만 수용되면 그들에게는 막대한 이득입니다.
2.  **검증의 병목:** 반면, 메인테이너에게는 모든 리포트가 고도의 기술적 검토를 요구하는 '일감'입니다.

Rogers는 cURL의 최대 보상금인 $10,000조차, 실제로 cURL 내부의 심각한 취약점을 찾아낼 수 있는 역량을 가진 톱티어 연구원들에게는 큰 유인이 되지 않는다고 지적합니다. 그들에게 진짜 인센티브는 돈이 아니라 **'명성(Fame)'과 '브랜드(Brand)'**입니다.

![]()
*(Joshua Rogers)*

---

### 3. 기술적 심층 분석: 무엇이 'Slop'인가?

커뮤니티(Hacker News)에서 공유된 'AI Slop'의 구체적인 예시들은 LLM이 C언어의 메모리 모델과 컨텍스트를 어떻게 오해하는지 적나라하게 보여줍니다. (참고: [User anon의 Gist](https://gist.github.com/bagder/07f7581f6e3d78ef37dfbfc81fd1d...))

주요 패턴은 다음과 같습니다:

1.  **컨텍스트 없는 경계 검사 오류:**
    *   AI는 `memcpy`나 `strcpy`가 보이면 무조건적으로 버퍼 오버플로우 가능성을 제기합니다.
    *   하지만 해당 코드가 이미 상위 로직에서 엄격하게 길이를 검증(`size_t` 체크 등)하고 들어왔다는 제어 흐름(Control Flow)을 이해하지 못합니다.
2.  **비현실적인 위협 모델(Threat Model):**
    *   공격자가 물리적으로 서버의 메모리에 접근해야만 가능한 시나리오를 '원격 코드 실행(RCE)' 취약점으로 포장합니다.
3.  **거짓된 함수 동작:**
    *   라이브러리 내부 함수가 리턴하지 않는 값(예: NULL)을 리턴한다고 가정하고, NULL 포인터 역참조 버그를 주장합니다.

이러한 리포트들은 겉보기에는 매우 그럴듯한(plausible) 기술 용어와 마크다운 포맷을 갖추고 있어, 메인테이너가 코드를 직접 까보고 로직을 따라가기 전까지는 기각하기 어렵습니다.

---

### 4. 결론 및 인사이트

**cURL의 결정은 OSS 보안의 분기점입니다.**
지금까지 총 87건의 버그에 대해 약 $101,020(약 1억 3천만 원)가 지급되었습니다. 하지만 이제 '누구나 참여 가능한' 금전적 인센티브 모델은 AI 스팸의 홍수 속에서 그 수명을 다했습니다.

**CTO 및 시니어 엔지니어를 위한 제언:**
1.  **공개 바운티의 폐기 검토:** 프로젝트의 인지도가 높을수록, 공개 바운티는 'AI 스크립트 키디'들의 타겟이 될 확률이 높습니다. 노이즈가 감당 불가능하다면 과감히 중단하십시오.
2.  **검증된 연구원 중심의 비공개 프로그램(Private Bounty):** 신뢰할 수 있는 실력을 갖춘 연구원들만을 초대하여 운영하는 모델로 전환이 필요합니다. Joshua Rogers의 말처럼, "진짜 중요한 버그를 찾는 사람들은 점심값을 벌려고 리포팅하지 않습니다."
3.  **명성(Fame) 기반 인센티브 강화:** 금전적 보상 대신, 기여자 목록 상단 노출, 기술 블로그 인터뷰, 컨퍼런스 초청 등 '커리어 자산'을 제공하는 것이 고품질 리포트를 유도하는 더 강력한 필터가 될 수 있습니다.

AI는 생산성을 높여주지만, 검증되지 않은 AI 산출물은 타인의 시간을 갉아먹는 기생적인 요소가 되었습니다. cURL은 그 연결고리를 끊어냄으로써 엔지니어링 리소스를 보호하는 선택을 한 것입니다.

---

### References

*   **Original Article:** [Curl removes bug bounties because of AI slop](https://etn.se/index.php/nyheter/72808-curl-removes-bug-bounties.html)
*   **Hacker News Thread:** [Discussion on Y Combinator](https://news.ycombinator.com/item?id=46701733)
*   **Daniel Stenberg's Blog:** [Death by a thousand slops](https://daniel.haxx.se/blog/2025/07/14/death-by-a-thousand-slops/)
*   **Joshua Rogers' Analysis:** [2025 Bug Bounty Stories Fail](https://joshua.hu/2025-bug-bounty-stories-fail)
*   **List of AI Slop Examples (Gist):** [GitHub Gist by bagder](https://gist.github.com/bagder/07f7581f6e3d78ef37dfbfc81fd1d...)
