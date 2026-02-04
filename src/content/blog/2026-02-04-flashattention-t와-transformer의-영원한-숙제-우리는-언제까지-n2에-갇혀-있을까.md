---
title: "FlashAttention-T와 Transformer의 영원한 숙제: 우리는 언제까지 N^2에 갇혀 있을까?"
description: "FlashAttention-T와 Transformer의 영원한 숙제: 우리는 언제까지 N^2에 갇혀 있을까?"
pubDate: "2026-02-04T01:15:48Z"
---

솔직히 고백하자면, 요즘 "새로운 Attention 최적화 기법"이라는 논문이 나올 때마다 피로감을 느낍니다. Transformer가 세상을 집어삼킨 지 벌써 수년이 지났고, 우리는 여전히 이 $O(N^2)$의 저주를 풀기 위해 마른 수건을 쥐어짜고 있으니까요.

최근 ACM에 등재된 **FlashAttention-T: Towards Tensorized Attention** 이라는 논문이 Hacker News에서 꽤 뜨거운 논쟁을 불러일으켰습니다. 오늘은 이 논문이 던지는 화두와, 엔지니어들이 댓글 창에서 벌이고 있는 "Transformer vs. The World" 전쟁에 대해 이야기해보려 합니다.

## FlashAttention-T: 마른 수건에서 17%를 더 짜내는 법

우선 이 논문의 핵심부터 짚고 넘어갑시다. 원문은 유료 장벽 뒤에 있지만, 공개된 정보와 커뮤니티의 분석을 종합해보면 핵심은 **"Tensorized"** 에 있습니다.

기존의 FlashAttention(Tri Dao의 걸작)이 메모리 I/O 병목을 해결하는 데 집중했다면, FlashAttention-T는 GPU의 Compute Core 활용률, 특히 **Pipeline Stall** 을 줄이는 데 초점을 맞춥니다. 간단히 말해, GPU가 데이터를 기다리느라 멍하니 있는(stall) 시간을 줄이고, 텐서 코어(Tensor Core)가 쉴 새 없이 돌아가도록 명령어 스케줄링을 기가 막히게 조절했다는 뜻입니다.

결과적으로 **5%에서 최대 17%의 속도 향상** 을 주장합니다. "겨우 그 정도?"라고 할 수 있겠지만, 수천 대의 H100을 돌리는 인프라 팀 입장에서는 수십억 원이 왔다 갔다 하는 숫자입니다.

흥미로운 점은 Hacker News 댓글 중 하나가 지적했듯, 이 논문에 원조 FlashAttention의 저자인 Tri Dao가 포함되어 있지 않다는 겁니다. 이름만 빌려 쓴 것 아니냐는 의혹도 있지만, 기술적으로는 Ampere 아키텍처(A100 등)에서 쥐어짜 낼 수 있는 마지막 성능을 뽑아낸 것으로 보입니다.

## 왜 우리는 여전히 Quadratic($N^2$)에 집착하는가?

이 글을 쓰게 된 진짜 이유는 논문 그 자체보다, 이어진 엔지니어들의 토론 때문입니다. 한 유저(User anon)가 던진 질문이 제 엔지니어링 본능을 자극했습니다.

> "도대체 왜 우리는 전체 컨텍스트를 다 봐야 하는 Attention에만 매달려 있나? 인간은 이렇게 작동하지 않잖아?"

맞는 말입니다. $N^2$ 복잡도는 시퀀스 길이가 길어질수록 재앙이 됩니다. 그래서 RWKV, Mamba(State Space Models), Linear Attention 같은 $O(N)$ 아키텍처들이 끊임없이 도전장을 내밀고 있죠. 하지만 **Industry(현업)는 여전히 Transformer에 올인** 하고 있습니다. 왜일까요?

### 1. 학습 효율성 (Parallelism)
RNN 계열이나 일부 순차적 모델들은 학습 시 병렬화가 어렵습니다. 반면 Transformer의 Causal Masked Attention은 시퀀스 전체를 한 번에 쏟아부어 학습시킬 수 있습니다. GPU를 100% 활용해서 빠르게 학습시킬 수 없다면, 모델 구조가 아무리 우아해도 탈락입니다.

### 2. "Deep"의 진짜 의미
댓글 중 통찰력 있는 의견이 있었습니다. 단순히 모든 토큰을 비교하는 게 낭비처럼 보이지만, **Layer가 깊어질수록(Deep) 이 $N^2$ 연산은 단순 비교를 넘어섭니다.**

- **Layer 1:** 단어 vs 단어 비교
- **Layer 20:** 문맥 vs 문맥, 추상적 패턴 vs 관계성 비교

깊이가 깊어질수록 Attention은 고차원적인 상호작용(Higher-order interactions)을 모델링합니다. 얕고 넓은 네트워크보다 깊고 좁은 네트워크가 더 효율적이라는 것은 이미 딥러닝의 정설이죠. Transformer는 이 "깊이"를 쌓기에 가장 최적화된 블록입니다.

## 하드웨어: Consumer vs. Cloud의 격차

또 하나 흥미로운 주제는 하드웨어 최적화입니다. 한 유저가 "AI 시스템 프로그래밍으로 전직하고 싶은데, 집에 있는 GPU로 공부해도 되냐"고 묻자, 현직 CUDA 엔지니어가 뼈 때리는 조언을 합니다.

- **Cloud GPU:** H100, B200 등은 TMEM MMA 같은 Cooperative Execution 기능을 지원합니다. 이는 칩 간, 혹은 SM(Streaming Multiprocessor) 간의 초고속 협업을 가능하게 합니다.
- **Consumer GPU:** RTX 4090 같은 카드는 이런 기능이 제한적이거나 아키텍처가 다릅니다.

결국 최첨단 커널 최적화(Kernel Optimization)를 공부하려면 클라우드 인스턴스를 빌려야 하는 시대가 왔습니다. FlashAttention-T 같은 기술도 결국 이런 엔터프라이즈급 하드웨어의 특성을 극한으로 이용하는 것이니까요.

## Principal Engineer's Verdict

FlashAttention-T는 혁명이라기보다는 **"극한의 최적화"** 입니다. 하지만 이런 최적화들이 모여서 지금의 LLM 시대를 지탱하고 있습니다. AI가 코드를 짜주는 시대라지만, 결국 하드웨어의 밑바닥(Instruction Level)까지 긁어서 병목을 해결하는 건 아직 인간 엔지니어들의 몫인 것 같습니다.

하지만 마음 한구석에서는 여전히 $O(N^2)$를 벗어날 "Next Transformer"를 기다립니다. 리튬 배터리가 나트륨 배터리보다 효율이 좋아서 계속 쓰이는 것처럼, Transformer도 당분간은 왕좌를 지킬 것입니다. 하지만 그 왕좌가 영원할 것이라고 믿는 건 엔지니어로서 너무 안일한 태도겠죠.

**Reference:**
- **Original Article:** https://dl.acm.org/doi/10.1145/3774934.3786425
- **Hacker News Thread:** https://news.ycombinator.com/item?id=46877403
