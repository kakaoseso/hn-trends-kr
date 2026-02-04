---
title: "Transformers Without the Memory Wall? A Look at SATA and Constant Cost Attention"
description: "Transformers Without the Memory Wall? A Look at SATA and Constant Cost Attention"
pubDate: "2026-02-04T15:39:06Z"
---

## 또 하나의 'Linear Attention' 논문일까요, 아니면 진짜 게임 체인저일까요?

솔직히 말해서, "Linear Attention"이나 "Constant Cost Attention"을 주장하는 논문 제목을 볼 때마다 저는 기대보다 의심부터 앞섭니다. 지난 몇 년간 우리는 수많은 $O(N)$ Attention 논문들이 쏟아져 나오는 것을 목격했습니다. 하지만 여전히 우리는 프로덕션에서 `FlashAttention`을 쓴 표준 Transformer를 돌리고 있죠. 왜냐고요? **'True love is quadratic'** 이라는 농담이 있을 정도로, Full Attention의 성능을 온전히 따라잡는 근사(Approximation) 기법이 드물기 때문입니다.

하지만 최근 arXiv에 올라온 **"Self-Attention at Constant Cost per Token via Symmetry-Aware Taylor Approximation (SATA)"** 논문은 꽤 흥미로운 지점을 건드리고 있습니다. 단순히 "빠르다"고 주장하는 것을 넘어, 기존 Taylor Expansion 기반 접근법들의 고질적인 문제였던 연산 비용과 불안정성을 '대칭성(Symmetry)'을 이용해 해결했다고 주장합니다.

오늘은 이 논문이 주장하는 바와 엔지니어로서 우리가 가져야 할 현실적인 의구심, 그리고 Hacker News 커뮤니티의 반응을 섞어 깊이 있게 들여다보겠습니다.

---

### 1. 문제는 언제나 KV Cache와 $O(N^2)$ 입니다

모두 아시다시피, 표준 Transformer의 Self-Attention은 입력 시퀀스 길이 $N$에 대해 $O(N^2)$의 연산량과 메모리를 요구합니다. Inference 시에는 KV Cache가 $O(N)$으로 증가하죠. Context Window가 100k, 1M 토큰으로 늘어나는 요즘, 이 선형적 메모리 증가는 치명적인 병목입니다.

이 논문(SATA)의 핵심 주장은 매력적입니다:

> "토큰당 비용을 상수로 고정(Constant Cost)하면서도, 임의의 정밀도로 Self-Attention을 계산할 수 있다."

즉, 무한한 길이의 텍스트를 생성하더라도 메모리 사용량이 늘어나지 않는다는 뜻입니다. 마치 RNN이나 SSM(State Space Models)처럼 고정된 크기의 State만 유지하면 된다는 것이죠.

### 2. 어떻게 가능할까요? (Symmetry-Aware Taylor Approximation)

기존에도 Softmax Attention을 선형화하려는 시도는 많았습니다 (예: Performer, Linear Transformer). 핵심 아이디어는 $e^{QK^T}$라는 수식을 Taylor Series로 전개하여 $Q$와 $K$를 먼저 분리하는 것입니다. 이렇게 하면 $Q \times (K^T \times V)$ 순서로 계산할 수 있어 $N^2$ 행렬을 만들 필요가 없어집니다.

하지만 Taylor 급수는 항(Term)을 많이 쓸수록 정확해지지만, 연산량이 급증하는 문제가 있었습니다. 이 논문은 여기서 **Tensor Product의 대칭성(Symmetry)** 을 활용합니다.

- **핵심 기술:** Taylor 전개식을 대칭적인 텐서 곱의 체인으로 분해합니다.
- **Polynomial Feature Basis:** 이를 통해 $Q$와 $K$를 효율적으로 매핑하는 Feed-forward 변환을 유도합니다.
- **결과:** Head Size에 반비례하는 고정 비용으로 계산이 가능해집니다. 즉, 더 많은 Head를 써도 비용 부담이 적다는 뜻입니다.

저자들은 4차 Taylor 항($P=4$)만 사용해도 Float16 해상도 수준의 오차 범위 내에서 기존 Attention을 복원할 수 있다고 주장합니다.

### 3. Hacker News의 반응: "Attention이 흐릿해지지 않을까?"

이 논문에 대해 엔지니어들이 가장 우려하는 부분은 **'Softmax의 날카로움(Sharpness)'** 을 잃는 것입니다. Hacker News의 한 유저는 이렇게 지적했습니다:

> "Attention의 본질은 'Needle-in-a-haystack'이나 'Winner-takes-all'처럼 특정 정보에 강하게 집중하는 능력입니다. Taylor 근사는 이 뾰족한 분포를 부드럽게(Soften) 뭉개버릴 위험이 있습니다."

저도 이 부분에 동의합니다. Softmax는 $e^x$의 특성상 가장 큰 값에 확률을 몰아주는 경향이 있는데, 다항식 근사(Polynomial Approximation)가 과연 그 'Spike'를 제대로 표현할 수 있을까요? 만약 문맥 속에 아주 미묘한 차이를 가진 4~8개의 비슷한 정보가 있을 때, 진짜 정답 하나를 골라낼 수 있을지는 미지수입니다.

물론 논문에서는 "Float16 수준의 오차"라고 방어하지만, **Perplexity가 비슷하다고 해서 Downstream Task(특히 복잡한 Reasoning이나 Retrieval) 성능이 동일하다는 보장은 없습니다.**

### 4. 하드웨어 관점: 이론 vs 현실

이론적으로 $O(1)$이라도 실제 GPU에서는 이야기가 다릅니다. Hacker News의 다른 코멘트가 지적했듯, Taylor 급수 연산은 생각보다 복잡합니다.

- **Kernel Complexity:** 표준 Softmax Attention은 `FlashAttention` 등을 통해 하드웨어 레벨에서 극한으로 최적화되어 있습니다. 반면, 복잡한 다항식 연산을 포함하는 커스텀 커널은 메모리 대역폭(Memory Bandwidth) 효율이 떨어지거나, 연산 자체의 오버헤드가 클 수 있습니다.
- **Numerical Stability:** Taylor 급수는 $x$ 값이 클 때 발산하기 쉽습니다. 논문에서는 이를 제어한다고 하지만, 대규모 모델 학습 시 어떤 발산 이슈가 터질지는 아무도 모릅니다.

### 5. 결론: 아직은 '시기상조'지만, 방향은 맞다

개인적으로 이 논문이 당장 Llama 4나 GPT-5의 아키텍처를 바꿀 것이라고 보지는 않습니다. **"True love is quadratic"** 이라는 말처럼, Full Attention의 표현력은 아직 대체 불가능한 영역에 있습니다. 특히 '정확한 기억'이 필요한 작업에서는 더욱 그렇습니다.

하지만 **Inference Cost** 가 비즈니스의 생존을 결정하는 시대에, 이런 시도는 매우 중요합니다. 만약 이 기법이 'Hybrid' 방식(예: 최근 윈도우는 Full Attention, 먼 과거는 SATA로 압축)으로 사용된다면, 무한에 가까운 Context를 현실적인 비용으로 제공하는 돌파구가 될 수도 있습니다.

**엔지니어로서의 제안:**
당장 프로덕션에 적용하기보다는, 이 논문의 [GitHub 구현체](https://github.com/glassroom/sata_attention)를 가지고 **'Needle In A Haystack'** 테스트를 직접 돌려보세요. PPL(Perplexity) 수치만 보고 환호하기엔, 우리는 이미 너무 많은 'Linear Attention' 논문의 무덤을 보아왔으니까요.

---

**References:**
- [Original Paper (arXiv)](https://arxiv.org/abs/2602.00294)
- [Hacker News Discussion](https://news.ycombinator.com/item?id=46886265)
