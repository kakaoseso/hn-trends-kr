---
title: "Leanstral 리뷰: AI 코드 생성의 진짜 병목은 인간의 리뷰다"
description: "Leanstral 리뷰: AI 코드 생성의 진짜 병목은 인간의 리뷰다"
pubDate: "2026-03-17T04:03:47Z"
---

최근 몇 년간 수많은 AI 코딩 어시스턴트를 써오면서 느낀 가장 큰 딜레마가 있습니다. AI가 1초 만에 짜준 1000줄짜리 코드를, 제가 1시간 동안 눈이 빠져라 리뷰해야 한다는 사실이죠. "이 코드가 진짜 돌아갈까? 엣지 케이스는 없나?" 결국 책임은 시니어 엔지니어인 제 몫입니다. AI가 아무리 발전해도 **Human in the loop** 자체가 엔지니어링 속도의 가장 큰 병목이 되어버렸죠.

그런데 며칠 전 Mistral에서 꽤 흥미로운 접근법을 내놓았습니다. 바로 Lean 4 전용 오픈소스 코드 에이전트, Leanstral입니다. 단순한 코드 생성을 넘어 정형 증명(Formal verification)의 영역으로 AI를 끌어들인 이 시도가 왜 중요한지, 그리고 해커뉴스 커뮤니티의 반응은 어떤지 깊게 파헤쳐보겠습니다.

### Leanstral: 코딩이 아니라 증명을 생성하는 AI

Lean 4는 단순한 범용 프로그래밍 언어가 아니라 Proof assistant(정형 증명기)입니다. 수학적 정리나 소프트웨어 스펙을 엄밀하게 증명하는 데 쓰이죠. Mistral은 6B 파라미터의 작고 가벼운 모델을 만들어 이 Lean 4 환경에 특화시켰습니다.

핵심 아이디어는 이렇습니다. 인간이 "어떻게(How)" 동작할지 디버깅하는 대신, "무엇을(What)" 원하는지 스펙만 작성하면, AI가 구현체와 그 구현체가 스펙을 만족한다는 증명을 함께 작성합니다.

해커뉴스 스레드에서 한 유저가 핵심적인 질문을 던졌습니다. "AI가 스펙을 짜면 그 스펙은 누가 검증하냐? 결국 다시 원점 아닌가?"

여기에 대한 답이 바로 Lean의 강력함입니다. Lean의 타입 체커는 약 1만 줄 정도의 매우 작고 검증된 커널(Kernel)입니다. 우리는 AI 에이전트를 믿는 게 아니라, 수학적으로 증명된 커널을 믿는 겁니다. AI가 아무리 환각(Hallucination)을 일으켜도, 증명이 타입 체커를 통과하지 못하면 가차 없이 거부됩니다. 코드 리뷰의 부담이 기계에게 넘어가는 순간이죠.

### 비용과 성능: pass@k가 빛을 발하는 순간

일반적인 LLM 벤치마크에서 pass@k(k번 시도해서 한 번이라도 맞추면 정답 처리)는 꽤 기만적인 지표입니다. 10번 시도해서 나온 10개의 코드 중 뭐가 맞는지 결국 인간이 확인해야 하니까요. 하지만 Lean 환경에서는 다릅니다. 컴파일러가 알아서 정답을 걸러주기 때문에, 무한 루프를 돌리듯 에이전트에게 계속 시도하게 만들 수 있습니다.

결과는 놀랍습니다.

- **Leanstral pass@2:** 26.3점 (비용 $36)
- **Claude 3.5 Sonnet:** 23.7점 (비용 $549)
- **Claude 3 Opus:** 39.6점 (비용 $1,650)

Claude Opus가 압도적인 품질을 보여주긴 하지만, 비용이 무려 92배 차이가 납니다. 자본이 넘치는 빅테크라면 Opus를 쓰겠지만, 로컬에서 Apache 2.0 라이선스로 돌릴 수 있는 6B 모델이 Sonnet을 이긴다는 건 엄청난 가성비입니다.

![Leanstrall Normalized Model Cost Vs Flt Eval Score](https://cms.mistral.ai/assets/78aaa774-c717-4c60-b71b-1161487b28e5.jpg?width=1861&height=1245)

### 실제 코드 사례: Rocq에서 Lean으로의 변환

블로그 포스트에 소개된 사례 중 가장 인상 깊었던 것은 Rocq(Coq)의 정의를 Lean으로 변환하고 증명하는 과정이었습니다. 아래는 Leanstral이 변환한 명령어 평가(Command evaluation)의 일부입니다.

```lean
inductive ceval : com → state → state → Prop where
| E_Skip (st : state) : ceval .CSkip st st
| E_Ass (st : state) (a1 : aexp) (n : Nat) (l : ident) (h : aeval a1 st = n) :
  ceval (.CAss l a1) st (update st l n)
```

더 놀라운 것은 증명 과정입니다. X 변수에 2를 더하는 `plus2` 명령어에 대해, 초기 상태에서 n이었던 값이 실행 후 n+2가 된다는 것을 완벽하게 증명해냅니다.

```lean
theorem plus2_spec (st : state) (n : Nat) (st' : state) (h1 : st "X" = n) (h2 : plus2 / st ⇒ st') :
  st' "X" = n + 2 := by
  change ceval (.CAss "X" (.APlus (.AId "X") (.ANum 2))) st st' at h2
  cases h2 with
  | E_Ass _ _ n l h =>
    have : aeval (.APlus (.AId "X") (.ANum 2)) st = n := h
    simp only [aeval] at this
    rw [update]
    simp [← this, h1]
```

이런 코드는 일반적인 개발자가 처음부터 짜기엔 러닝 커브가 너무 가파릅니다. 하지만 AI가 초안을 잡고 증명까지 해준다면 이야기가 달라지죠.

### 해커뉴스 커뮤니티의 반응과 나의 생각

이번 발표를 보며 커뮤니티에서는 **TDD의 재발견** 이라는 키워드가 화두로 떠올랐습니다. "AI가 TDD의 이상을 현실로 만들고 있다"는 한 유저의 의견에 전적으로 동의합니다. 과거에는 테스트 코드 짜는 게 본 코드 짜는 것보다 오래 걸려서 포기했지만, 이제는 인간이 Property-based spec만 던져주고 나머지는 AI가 증명하게 만드는 흐름으로 갈 것입니다.

또한, LLM Alloy(혼합 모델)에 대한 아이디어도 흥미로웠습니다. pass@k를 할 때 Leanstral만 여러 번 돌릴 게 아니라 Qwen, Kimi 등 다른 모델을 섞어 쓰면 정답률이 훨씬 올라갈 수 있다는 엔지니어링적 접근은 당장 실무 파이프라인에 적용해볼 만한 팁입니다.

### 결론: 당장 쓸 수 있을까?

솔직히 말해서, 당장 내일 우리가 짜는 React나 Spring Boot 웹 서버에 이걸 도입할 수 있을까요? 아닙니다. 여전히 너무 학술적이고 생소한 영역입니다. 

하지만 결제 코어, DB 엔진, 인프라 제어 같은 Mission-critical 시스템에서는 이 패러다임이 결국 표준이 될 것이라 확신합니다. "김 대리에게 물어봐" 식의 구전 지식에 의존하는 코딩이 아니라, 수학적으로 증명된 코드를 배포하는 시대. Leanstral은 그 미래를 로컬 환경으로 가져온 훌륭한 마일스톤입니다.

---
- **원본 글:** https://mistral.ai/news/leanstral
- **해커뉴스 스레드:** https://news.ycombinator.com/item?id=47404796
