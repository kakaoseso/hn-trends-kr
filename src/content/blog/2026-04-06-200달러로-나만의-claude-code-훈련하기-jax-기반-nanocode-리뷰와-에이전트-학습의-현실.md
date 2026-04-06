---
title: "200달러로 나만의 Claude Code 훈련하기: JAX 기반 Nanocode 리뷰와 에이전트 학습의 현실"
description: "200달러로 나만의 Claude Code 훈련하기: JAX 기반 Nanocode 리뷰와 에이전트 학습의 현실"
pubDate: "2026-04-06T01:10:52Z"
---

최근 수많은 AI 코딩 어시스턴트가 쏟아져 나오고 있습니다. Copilot, Cursor, 그리고 최근의 Claude Code까지, 우리는 이미 이 도구들에 깊이 의존하고 있죠. 하지만 시니어 엔지니어로서 단순히 API를 호출하는 것을 넘어, **이 에이전트들이 도대체 내부적으로 어떻게 학습되고 동작하는지** 궁금증을 가져본 적이 있으실 겁니다.

솔직히 처음에 제목만 보고는 흔한 API 래퍼(wrapper) 프로젝트인 줄 알았습니다. 하지만 [Salman Mohammadi의 nanocode 프로젝트](https://github.com/salmanmohammadi/nanocode/discussions/1)는 제 예상을 완전히 빗나갔습니다. 이 프로젝트는 순수 JAX와 TPU를 이용해 1.3B 파라미터의 코딩 에이전트를 밑바닥부터(end-to-end) 학습시키는 과정을 적나라하게 보여줍니다. 200달러의 비용과 9시간의 학습 시간으로 말이죠.

오늘은 이 흥미로운 프로젝트를 해부해 보면서, 최신 에이전트 모델이 어떻게 도구를 사용하고(Tool-use), 어떻게 정렬되는지(Alignment), 그리고 그 과정에서 우리가 마주하는 현실적인 한계는 무엇인지 딥다이브해 보겠습니다.

## 1. JAX와 TPU: 효율적인 학습의 기반

이 프로젝트는 Andrej Karpathy의 `nanochat` 인프라를 기반으로 합니다. PyTorch가 지배하는 세상이지만, 저자 역시 언급했듯 JAX와 XLA 컴파일러의 조합은 분산 학습 환경(특히 TPU)에서 엄청난 퍼포먼스와 우아한 코드를 자랑합니다.

가장 먼저 눈에 띄는 것은 **데이터 믹스와 토크나이저 튜닝** 입니다. 범용 텍스트 모델을 코딩 에이전트로 탈바꿈시키기 위해 저자는 FineWeb-EDU와 The Stack-V2 데이터를 1:5 비율로 섞었습니다. 

결과는 명확했습니다. 코드 데이터에 대한 토크나이징 효율은 급격히 상승했지만, 일반 텍스트에 대한 효율은 떨어졌죠. 

- **Trade-off:** 범용적인 언어 추론 능력(CORE metric)은 GPT-2 XL(0.257)과 유사한 수준(0.261)으로 떨어졌지만, 코딩이라는 단일 도메인에서의 퍼포먼스는 극대화되었습니다.

이것은 매우 실용적인 접근입니다. 1.3B라는 작은 파라미터 제약 속에서 '모든 것을 잘하는' 모델을 만드는 것은 불가능합니다. 타겟 도메인에 리소스를 몰아주는 선택은 엔지니어링 관점에서 매우 타당합니다.

## 2. Agentic Interface: 텍스트 생성기를 에이전트로 만들기

LLM은 본질적으로 다음 토큰을 예측하는 텍스트 생성기(Next-token-generator)에 불과합니다. 이 녀석을 환경과 상호작용하는 '에이전트'로 만들기 위해서는 **템플릿(Templating)** 이라는 마법이 필요합니다.

저자는 모델이 툴을 호출하고 결과를 받을 수 있도록 특수 토큰들을 정의했습니다.

```text
<|assistant_start|>
I'll search for TODO.
<|tool_call_start|>Bash<|tool_arg|>command<|tool_val|>grep -rn "TODO" src/<|tool_call_end|>
<|assistant_end|>
```

이 구조를 보면 Anthropic이나 OpenAI가 내부적으로 도구 호출(Function Calling)을 어떻게 구현하는지 정확히 알 수 있습니다. 단순한 텍스트 스트림 사이에 `<|tool_call_start|>` 같은 구분자를 넣고, 에이전트 루프를 도는 얇은 CLI 래퍼가 이 토큰을 가로채서 실제 UNIX 명령어를 실행한 뒤, 그 결과를 다시 `<|tool_result_start|>` 토큰과 함께 모델에게 먹여주는 방식입니다.

저자는 모델의 표현력과 학습 가능성 사이의 균형을 맞추기 위해 `Read`, `Edit`, `Grep`, `Bash` 네 가지 도구만 정의했습니다. 작은 모델에게 복잡한 Bash 파이프라인을 처음부터 깨우치게 하는 것은 컴퓨팅 낭비이기 때문입니다.

## 3. Constitutional AI와 합성 데이터의 함정

이 프로젝트의 백미는 Anthropic의 **Constitutional AI (CAI)** 방법론을 적용한 부분입니다. 모델에게 특유의 성격(SOUL)을 부여하기 위해 저자는 '소문자만 사용할 것', '친근할 것', '지시사항을 엄격히 따를 것' 등의 규칙을 정의했습니다.

학습은 크게 두 단계로 진행됩니다.

1. **Constitutional SFT:** Generator 모델이 답변을 생성하면, Judge 모델이 SOUL 규칙에 따라 평가하고 비판(Critique)합니다. 이를 바탕으로 답변을 수정하는 루프를 돌아 고품질의 합성 데이터를 만듭니다.
2. **DPO (Direct Preference Optimisation):** 생성된 선호/비선호 데이터 쌍을 바탕으로 모델을 정렬합니다.

하지만 여기서 흥미로운 문제, 아니 **현실적인 한계** 가 발생합니다. Hacker News 커뮤니티에서 날카롭게 지적한 부분이기도 하죠.

저자가 블로그에 예시로 든 합성 데이터(SFT)의 프롬프트와 결과물을 보겠습니다.

**Prompt:**
> 리스트에서 falsey 값을 제거하는 Python 함수를 작성해라. 새로운 리스트를 만들지 말고 수정된 리스트를 반환해라. (Modify in place) 단, `*args`와 리스트 컴프리헨션을 사용해라.

**Assistant (Generated Code):**
```python
def remove_falsey_values(*args):
    return [val for val in args if val]
```

HN 유저들이 즉각적으로 지적했듯, 이 코드는 **틀렸습니다.** 
프롬프트는 명확히 "새로운 리스트를 만들지 말고(without creating a new one)" In-place 수정을 요구했지만, 모델은 리스트 컴프리헨션을 사용해 완전히 새로운 리스트를 반환하고 있습니다. 애초에 `*args` 튜플을 받아 In-place로 수정하라는 프롬프트 자체가 Python의 언어적 특성(튜플은 불변)과 충돌하는 모순을 안고 있습니다.

저자 역시 이 비판에 대해 인정했습니다. 기존 오픈소스 합성 데이터셋(evol-codealpaca 등)을 시드(seed)로 사용했는데, 이 데이터들 자체가 이미 다른 LLM에 의해 생성된 것이라 오류를 포함하고 있었던 것이죠.

**"Data is indeed the moat (데이터가 곧 해자다)"**

이 해프닝은 우리에게 아주 중요한 교훈을 줍니다. SFT 파이프라인을 아무리 정교하게 구축해도, 기반이 되는 데이터의 품질이 엉망이면 모델은 그 바보 같은 행동을 그대로 학습합니다(Garbage in, Garbage out). 최근 많은 스타트업들이 LLM 기반 합성 데이터에만 의존해 모델을 튜닝하려 하지만, 프로덕션 레벨에서는 반드시 도메인 전문가의 깐깐한 리뷰(Human-in-the-loop)가 동반되어야 하는 이유가 바로 여기에 있습니다.

## 4. DPO는 작은 모델에게 과연 유효한가?

저자는 RLHF 대신 더 가벼운 DPO를 사용했습니다. 결과적으로 선호도 정확도(Accuracy)는 0.45에서 0.88로 올랐지만, Validation BPB(Bits-per-byte)는 0.247에서 0.248로 사실상 변화가 없었습니다.

개인적인 경험에 비추어 볼 때, 1.3B 정도의 작은 파라미터와 제한된 토큰 예산 환경에서 DPO는 종종 플라시보에 가깝습니다. 모델은 이미 SFT 단계에서 제공된 SOUL 정렬 데이터에 심하게 오버피팅(over-tuned)되어 있기 때문입니다. DPO의 진가는 수많은 도메인에서 학습된 거대 모델의 원치 않는 행동을 '깎아낼 때(optimise away)' 발휘됩니다. 

HN 스레드에서도 "Claude Code"라는 용어 사용을 두고 논쟁이 있었습니다. 누군가는 "Claude Code는 단순한 래퍼(Harness)이지 학습 가능한 모델이 아니다"라고 지적했고, 저자는 "도구 사용을 위해 미세 조정된 모델이 백엔드에 있을 것"이라고 반박했죠. 저는 저자의 의견에 동의합니다. 단순한 프롬프팅만으로는 복잡한 환경에서의 안정적인 도구 체이닝(Tool chaining)을 보장하기 어렵습니다. 반드시 이 프로젝트처럼 Agentic SFT 과정이 수반되어야 합니다.

## 결론: 프로덕션 레벨은 아니지만, 완벽한 교보재

`nanocode`로 학습된 1.3B 모델을 현업 프로덕션에 당장 투입할 수 있을까요? 당연히 아닙니다. 복잡한 버그 픽스나 본 적 없는 코드베이스 앞에서는 처참하게 무너질 것입니다.

하지만 이 프로젝트의 가치는 결과물이 아니라 **과정** 에 있습니다. 단돈 200달러로, 5.5K 라인밖에 안 되는 깔끔한 JAX 코드를 통해 최신 프론티어 모델들이 사용하는 Constitutional AI, Tool-calling, SFT, DPO 파이프라인의 A to Z를 직접 돌려볼 수 있다는 것은 엄청난 축복입니다.

블랙박스처럼 느껴지는 AI 에이전트의 동작 원리를 뼛속까지 이해하고 싶다면, 주말을 투자해 Google TRC 크레딧으로 이 프로젝트를 직접 클론하고 돌려보시기를 강력히 권합니다. 직접 눈으로 Loss가 떨어지는 것을 보고, 모델이 엉뚱한 Bash 명령어를 날리는 것을 디버깅해 보는 것만큼 확실한 학습은 없으니까요.

---
**References:**
- [Original Article: Nanocode](https://github.com/salmanmohammadi/nanocode/discussions/1)
- [Hacker News Thread](https://news.ycombinator.com/item?id=47649742)
