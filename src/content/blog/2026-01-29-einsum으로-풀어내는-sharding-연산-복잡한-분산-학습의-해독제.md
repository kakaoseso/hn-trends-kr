---
title: "Einsum으로 풀어내는 Sharding 연산: 복잡한 분산 학습의 해독제"
description: "Einsum으로 풀어내는 Sharding 연산: 복잡한 분산 학습의 해독제"
pubDate: "2026-01-29T20:12:44Z"
---

분산 학습(Distributed Training) 인프라를 구축해 본 엔지니어라면 공감할 겁니다. 텐서(Tensor)가 쪼개지고, GPU 클러스터 사이를 날아다니는 과정을 머릿속으로 그리는 게 얼마나 고통스러운지 말이죠.

보통 우리는 화이트보드 앞에 서서 네모난 행렬을 그리고, "자, 이걸 칼로 자르듯이 쪼개면..." 하며 그림을 그립니다. 직관적이긴 하죠. 하지만 모델이 커지고 병렬화 전략(Parallelism Strategy)이 복잡해지면 이 '그림 그리기' 방식은 금방 한계에 부딪힙니다. 계산은 느리고, 실수는 잦아지죠.

최근 PyTorch의 리드 엔지니어인 Edward Z. Yang이 쓴 [Computing sharding with einsum](https://blog.ezyang.com/2026/01/computing-sharding-with-einsum/)이라는 글을 읽었는데, 무릎을 탁 쳤습니다. **"왜 여태까지 그림을 그리고 있었지? 수식 한 줄이면 되는데."**

오늘은 이 글을 바탕으로, `einsum`이 어떻게 분산 처리를 위한 '암산 도구'가 될 수 있는지, 그리고 왜 시니어 엔지니어들이 이 표기법에 익숙해져야 하는지 딥다이브 해보겠습니다.

## Einsum: 단순히 코드를 줄이는 게 아니다

주니어 시절엔 `torch.einsum`을 보면 "그냥 가독성 떨어지는 암호문 아닌가?"라고 생각했습니다. `torch.matmul`, `torch.bmm`, `torch.mm`... 함수 이름만 봐도 뭘 하는지 알 수 있는데 굳이 왜 `"ij,jk->ik"` 같은 문자열을 써야 하나 싶었죠.

하지만 차원이 4개, 5개로 늘어나고 Sharding을 고민해야 하는 시점이 오면 이야기가 달라집니다. `einsum`은 연산의 **Topology(위상)** 를 가장 명확하게 보여주는 도구입니다.

기본적인 Matrix Multiplication을 봅시다.

```python
# 수식: C = AB
torch.einsum("ij,jk->ik", A, B)
```

여기서 중요한 개념 두 가지가 나옵니다.
1.  **Free Dimension (자유 차원):** 입력과 출력 모두에 존재하는 인덱스 (`i`, `k`).
2.  **Contraction Dimension (축소 차원):** 입력에는 있지만 출력에서는 사라지는 인덱스 (`j`). 즉, 합(Summation)이 일어나는 차원입니다.

이 두 가지만 구별할 줄 알면, 복잡한 Sharding 규칙을 아주 우아하게 정의할 수 있습니다.

## Sharding 규칙의 4가지 패턴

Edward가 정리한 룰은 놀라울 정도로 심플합니다. 복잡한 `AllReduce`, `AllGather`를 생각하기 전에, 인덱스가 어떻게 변하는지만 보면 됩니다.

예를 들어 `"abi,aoi->abo"`라는 연산이 있다고 칩시다.

1.  **Replicate + Replicate -> Replicate**: 당연한 이야기입니다. 모든 곳에 복제되어 있으면 결과도 복제됩니다.
2.  **Shard(Batch) + Shard(Batch) -> Shard(Batch)**: 배치 차원(`a`)이 쪼개져 있다면, 결과물의 배치 차원도 쪼개진 상태로 유지됩니다.
3.  **Shard(Free) + Replicate -> Shard(Free)**: 자유 차원(`b`)이 쪼개져 있고 다른 입력은 복제되어 있다면, 결과도 해당 차원이 쪼개진 채로 나옵니다.
4.  **Shard(Contraction) + Shard(Contraction) -> Partial**: 이게 핵심입니다. 축소 차원(`i`)이 쪼개져 있다면, 각 디바이스는 전체 합의 '일부분(Partial Sum)'만 가지게 됩니다. **즉, 이 시점에서 `AllReduce`가 필요하다는 신호입니다.**

이 규칙들이 왜 강력하냐고요? 코드를 돌려보지 않고도 통신 오버헤드가 어디서 발생할지 '암산'이 가능해지기 때문입니다.

## 실전 검증: Megatron-LM의 Tensor Parallelism

이론은 지루하니 실제 사례를 봅시다. NVIDIA의 Megatron-LM을 뜯어보신 분들은 `ColumnParallelLinear` 같은 모듈에서 `CopyToModelParallelRegion`이 왜 호출되는지, 왜 Backward에서 `AllReduce`가 일어나는지 헷갈린 적이 있을 겁니다. 2019년에 깃허브 이슈로도 올라왔던 질문이죠.

이걸 `einsum`으로 해석하면 명쾌해집니다.

**Forward Pass:**
- Input: `[sequence(s), batch(b), in_features(i)]`
- Weight: `[in_features(i), out_features(o)]`
- 연산: `torch.einsum("sbi,io->sbo", input, weight)`

Tensor Parallelism(TP)에서는 Weight를 `out_features(o)` 차원으로 쪼갭니다(Shard). Input은 복제(Replicate)된 상태라고 가정합시다.
- Input: Replicate
- Weight: Shard("o")
- Output: Shard("o") (규칙 3에 의해)

여기까진 쉽습니다. 문제는 **Backward Pass** 입니다.

**Backward Pass (Gradient Computation):**
역전파를 할 때 `grad_input`을 구하는 식은 `einsum`에서 인덱스만 바꾸면 바로 나옵니다.

```python
grad_input = torch.einsum("sbo,io->sbi", grad_output, weight)
```

자, 여기서 Sharding 상태를 대입해 봅시다.
- `grad_output`: Forward의 Output이 Shard("o")였으므로, 얘도 Shard("o")입니다.
- `weight`: 여전히 Shard("o")입니다.
- **핵심:** 여기서 `o`는 `grad_input` 입장에서 입력엔 있고 출력(`sbi`)엔 없는 **Contraction Dimension** 입니다.

규칙 4번 기억나시나요? **Shard(Contraction) + Shard(Contraction) -> Partial**.

즉, `grad_input`은 각 GPU가 전체 그라디언트의 일부만 들고 있는 `Partial` 상태가 됩니다. 다음 레이어로 넘어가려면 온전한 값이 필요하죠. 그래서 여기서 필연적으로 **All-Reduce** 가 발생해야 하는 겁니다. Megatron 코드를 보며 "여기서 왜 통신을 하지?"라고 고민할 필요가 없습니다. 수식이 그렇게 말하고 있으니까요.

## Sequence Parallelism의 경우

반대로 Sequence Parallelism(SP)을 볼까요? 여기선 `sequence(s)` 차원을 쪼갭니다.

```python
# Forward: s가 Shard, h는 Replicate
torch.einsum("sbh,h->sbh", input, weight)
```

Backward에서 `grad_weight`를 구할 때를 봅시다.

```python
grad_weight = torch.einsum("sbh,sbh->h", input, grad_output)
```

여기서 `s`는 입력(`input`, `grad_output`) 모두에서 Shard 상태입니다. 그리고 `grad_weight`(`h`)에는 `s`가 없습니다. 즉, `s`는 **Contraction Dimension** 입니다.

다시 규칙 4번. Shard 된 차원이 축소되므로 `grad_weight`는 `Partial` 상태가 됩니다. 따라서 SP에서는 Weight의 그라디언트를 합치기 위해 `AllReduce`가 필요합니다.

## 엔지니어의 관점: 도구의 차이가 생각의 깊이를 만든다

솔직히 고백하자면, 저도 처음엔 DTensor나 Sharding 관련 코드를 짤 때 머릿속으로 GPU 8개를 그려놓고 데이터를 이리저리 옮기는 상상을 했습니다. 하지만 이 방식은 GPU가 1000개가 넘어가고, 3D Parallelism(TP+PP+DP)이 섞이는 순간 뇌 용량을 초과합니다.

Edward의 접근법이 훌륭한 이유는 **'물리적인 배치'를 '수학적인 속성'으로 추상화** 했기 때문입니다. `einsum`을 통해 우리는 데이터가 어디에 있는지(Where)보다, 데이터가 어떻게 상호작용하는지(How)에 집중할 수 있게 됩니다.

### 결론

아직도 분산 학습 코드를 짤 때 `view`, `permute`, `reshape`으로 차원 맞추기 놀이를 하고 계신가요? 아니면 복잡한 파이프라인 병렬화 로직을 주석으로만 설명하고 계신가요?

이제 `einsum`을 적극적으로 도입해 보세요. 처음엔 낯설겠지만, 익숙해지면 분산 시스템의 복잡도가 놀라울 정도로 단순해 보이는 경험을 하게 될 겁니다. **"Sharding은 곧 Einsum이다"** 라는 관점, 2026년의 엔지니어링에서는 선택이 아니라 필수 교양입니다.
