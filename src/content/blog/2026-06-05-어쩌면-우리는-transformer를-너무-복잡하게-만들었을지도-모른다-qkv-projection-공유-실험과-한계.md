---
title: "어쩌면 우리는 Transformer를 너무 복잡하게 만들었을지도 모른다: QKV Projection 공유 실험과 한계"
description: "어쩌면 우리는 Transformer를 너무 복잡하게 만들었을지도 모른다: QKV Projection 공유 실험과 한계"
pubDate: "2026-06-05T03:17:46Z"
---

LLM 인퍼런스 최적화나 서빙을 프로덕션 레벨에서 고민해 본 엔지니어라면 누구나 공감할 만한 골칫거리가 하나 있습니다. 바로 KV Cache입니다. 모델 파라미터가 차지하는 VRAM은 정적이지만, 컨텍스트 길이가 길어질수록 선형적으로 폭발하는 KV Cache는 배치 사이즈를 제한하고 결국 Throughput을 깎아먹는 주범이 됩니다.

이를 해결하기 위해 우리는 MQA(Multi-Query Attention)나 GQA(Grouped-Query Attention) 같은 아키텍처 레벨의 타협을 받아들여 왔습니다. 그런데 최근 arXiv에 올라온 [Do Transformers Need Three Projections? Systematic Study of QKV Variants](https://arxiv.org/abs/2606.04032) 논문은 아주 근본적이면서도 도발적인 질문을 던집니다. 

"애초에 Query, Key, Value를 전부 다 따로 Projection 할 필요가 있을까?"

오늘은 이 논문이 제안하는 QKV 가중치 공유(Weight Tying) 기법의 원리와 성과, 그리고 시니어 엔지니어 관점에서 바라본 명확한 한계점에 대해 딥다이브 해보겠습니다.

## 끔찍한 네이밍, 하지만 흥미로운 아이디어

본격적인 내용에 앞서 논문의 네이밍(Notation)부터 짚고 넘어가야겠습니다. Hacker News에서도 가장 많은 비판을 받은 부분인데, 저자들은 QKV 공유 방식을 `Q-K=V`, `Q=K-V` 같은 식으로 표기했습니다.

선형대수를 조금이라도 다뤄본 사람이라면 `Q-K=V`를 보는 순간 "Q 행렬에서 K 행렬을 빼면 V가 나온다고?"라고 생각할 겁니다. 저도 처음엔 그렇게 읽고 머리를 쥐어뜯었습니다. 하지만 이 논문에서 `-` 기호는 뺄셈이 아니라 그냥 **구분자(and)** 였습니다. 

즉, 논문에서 말하는 `Q-K=V`는 수학적으로 `(Q, K=V)`를 의미합니다. Query는 따로 두고 Key와 Value의 Projection 가중치를 동일하게 공유하겠다는 뜻이죠. 이 글에서는 혼란을 막기 위해 `K=V` 모델이라고 부르겠습니다.

## K=V 아키텍처: 왜 동작하는가?

전통적인 Attention 메커니즘에서 QKV는 각각 고유한 역할을 수행합니다. 기하학적으로 비유하자면, 수많은 벡터들을 고차원 공간에서 회전시키고 찌그러뜨려서 우리가 원하는 정보가 지나갈 수 있는 '틈'을 찾는 과정이죠.

그런데 Key와 Value의 가중치를 하나로 통일해버리면 어떻게 될까요? 직관적으로는 모델의 표현력(Expressiveness)이 크게 떨어질 것 같습니다. Query가 찾고자 하는 대상(Key)과 실제 반환되는 내용(Value)이 동일한 벡터 공간에 강제로 묶이기 때문입니다. 

하지만 논문의 실험 결과는 놀랍습니다. `K=V` 공유 방식은 언어 모델링에서 퍼플렉서티(Perplexity)를 불과 3.1%만 희생하면서 KV Cache를 50%나 줄여냅니다. 게다가 이 기법은 기존의 GQA나 MQA와 직교(Orthogonal)하는 성질을 가집니다. 즉, 둘을 결합할 수 있다는 뜻입니다.

- **K=V + GQA-4:** 87.5% 캐시 감소
- **K=V + MQA:** 무려 96.9% 캐시 감소

이게 가능한 이유는 Attention이 본질적으로 **Low-rank regime** 에서 동작하기 때문입니다. 고차원 공간에는 우리가 생각하는 것보다 훨씬 더 많은 '여유 공간'이 존재하며, Key와 Value가 유사한 표현 공간(Representational space)을 공유하더라도 모델이 유의미한 패턴을 학습하는 데는 충분하다는 것이 저자들의 주장입니다. 실제로 컨텍스트 길이가 길어질수록(512 -> 2048) 성능 저하 폭이 오히려 감소(5.4% -> 2.2%)한다는 데이터는, 이 방식이 단순히 시퀀스가 짧아서 통했던 요행이 아님을 시사합니다.

## 스케일링 커브 없이는 믿을 수 없다 (Scaling curves or GTFO)

여기까지만 보면 Edge 디바이스 배포를 위한 혁명적인 발견 같지만, 현업에서 대규모 모델을 다루는 엔지니어로서 저는 이 결과를 100% 신뢰하기 어렵습니다. Hacker News의 한 유저가 남긴 코멘트가 제 심정을 정확히 대변합니다.

> "I'm terribly sorry, but scaling curves or GTFO. Any random pile of linear algebra works fine-ish at small scales. Very few random piles of linear algebra push the Pareto envelope at large scales."

논문에서 테스트한 언어 모델의 최대 사이즈는 1.2B 파라미터이며, 학습 토큰은 고작 10B(100억) 개에 불과합니다. 2026년 현재, 제대로 된 1B 모델들은 최소 1T(1조)에서 많게는 10T 토큰으로 과학습(Over-training)을 진행합니다. 10B 토큰은 Chinchilla 최적화 기준의 절반에도 못 미치는, 심각한 Under-trained 상태입니다.

제 경험상, Attention 메커니즘을 단순화하거나 변형하는 기법들은 학습 초기(Under-trained regime)에는 기존 방식과 비슷하거나 심지어 더 나은 성능을 보여주곤 합니다. Attention 자체의 Inductive bias가 부족하기 때문에, 구조적 제약을 가하는 것이 초기 수렴에 도움을 줄 수 있기 때문이죠. 하지만 수조 개의 토큰을 밀어 넣으며 모델의 표현력을 극한까지 쥐어짜는 Over-training 단계에 진입하면, 이러한 구조적 제약(K=V)은 결국 성능의 천장(Ceiling)으로 작용하여 표준 QKV 모델에 크게 뒤처지게 됩니다.

## 결론: 그래서 프로덕션에 쓸 수 있는가?

이 논문은 분명 훌륭한 Ablation study입니다. "아무도 QKV를 쓴다고 해고당하지 않으니까" 관성적으로 써왔던 Transformer의 기본 구조에 의문을 제기했다는 점만으로도 가치가 있습니다. Gemma-4가 레이어 간(Cross-layer) KV 캐시를 공유하는 방식을 택했다면, 이 논문은 레이어 내(Intra-layer) 공유라는 또 다른 방향성을 제시했습니다.

하지만 당장 이 아키텍처를 메인스트림 Foundation Model 학습에 도입하기엔 리스크가 너무 큽니다. 수십억 원이 깨지는 대규모 학습 클러스터에서, 스케일링 법칙이 증명되지 않은 아키텍처를 채택할 CTO는 없습니다.

다만, **하드웨어 제약이 극심한 Edge AI나 On-device 모델링** 환경이라면 이야기가 다릅니다. VRAM이 몇 MB 단위로 아쉬운 환경에서, 퍼플렉서티 3%를 내어주고 메모리 50%를 얻는 것은 대단히 매력적인 트레이드오프입니다. 

결국 우리가 던져야 할 다음 질문은 명확합니다. "이 기법이 10B 파라미터, 1T 토큰 스케일에서도 여전히 유효할 것인가?" 누군가 GPU 클러스터를 태워 이 스케일링 커브를 증명해 주기 전까지, 이 매력적인 아이디어는 당분간 흥미로운 실험실의 결과물로 남을 것 같습니다.

---
**References:**
- Original Paper: [Do transformers need three projections? Systematic study of QKV variants](https://arxiv.org/abs/2606.04032)
- Hacker News Discussion: [https://news.ycombinator.com/item?id=48405931](https://news.ycombinator.com/item?id=48405931)
