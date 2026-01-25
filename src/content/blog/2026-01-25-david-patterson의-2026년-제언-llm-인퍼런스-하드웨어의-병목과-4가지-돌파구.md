---
title: "David Patterson의 2026년 제언: LLM 인퍼런스 하드웨어의 병목과 4가지 돌파구"
description: "David Patterson의 2026년 제언: LLM 인퍼런스 하드웨어의 병목과 4가지 돌파구"
pubDate: "2026-01-25T06:07:57Z"
---

엔지니어링 커리어에서 가끔 "아, 우리가 지금까지 잘못된 방향으로 최적화를 하고 있었구나"라는 깨달음을 주는 페이퍼를 만날 때가 있습니다. 오늘 소개할 논문이 딱 그렇습니다.

RISC 아키텍처의 아버지이자 튜링상 수상자인 **David Patterson** 이 2026년 1월, 따끈따끈한 아카이브 논문을 들고 나왔습니다. 제목은 [Challenges and Research Directions for Large Language Model Inference Hardware](https://arxiv.org/abs/2601.05047)입니다.

우리는 지난 몇 년간 NVIDIA GPU를 구하기 위해 전쟁을 치렀고, HBM(High Bandwidth Memory) 용량에 목을 매달았습니다. 하지만 Patterson은 이 논문에서 **"LLM 인퍼런스는 트레이닝과 근본적으로 다르다"** 며, 현재의 하드웨어 발전 방향에 제동을 겁니다. 시니어 엔지니어로서 이 논문을 씹고 뜯고 맛본 내용을 정리해 봅니다.

## 왜 인퍼런스는 트레이닝과 다른가?

많은 분들이 '트레이닝용 GPU'와 '인퍼런스용 GPU'를 구분 없이 사용합니다. 하지만 LLM의 동작 방식을 들여다보면 이 둘은 완전히 다른 workload를 가집니다.

논문에서 지적하는 핵심은 Transformer 모델의 **Autoregressive Decode** 단계입니다.

- **Prefill 단계:** 프롬프트를 한 번에 처리하므로 병렬화가 쉽고 Compute-bound입니다. 기존 GPU가 잘합니다.
- **Decode 단계:** 토큰을 하나씩 순차적으로 생성합니다. 이전 토큰이 나와야 다음 토큰을 계산할 수 있죠. 병렬 처리가 극도로 제한적이며, 매 토큰 생성 시마다 거대한 모델 가중치(Weights)와 KV Cache를 메모리에서 불러와야 합니다.

즉, 인퍼런스(특히 Decode)는 **Memory Wall** 과 **Interconnect** 의 싸움입니다. 아무리 FLOPS가 높은 칩을 가져다 놔도, 메모리 대역폭이 받쳐주지 않으면 GPU는 놀게 됩니다. Patterson은 이 병목을 해결하기 위해 4가지 연구 방향을 제시합니다.

## 4가지 하드웨어 돌파구 (Research Opportunities)

### 1. High Bandwidth Flash (HBF): 게임 체인저가 될 것인가?

개인적으로 가장 흥미로웠던 부분입니다. 보통 우리는 "Flash 메모리(SSD)는 느리고, DRAM은 빠르다"고 생각합니다. 그런데 Patterson은 **HBM 수준의 대역폭을 가진 Flash** 를 제안합니다.

- **아이디어:** 수십, 수백 개의 채널을 병렬로 연결하여 Throughput을 극대화한 Flash 메모리입니다.
- **장점:** HBM 대비 **10배 이상의 용량** 을 제공하면서도 대역폭을 맞출 수 있습니다.

Hacker News의 [관련 코멘트](https://news.ycombinator.com/item?id=46750214)에서도 언급되었듯, HBF 기술은 이미 업계에서 태동하고 있습니다. 만약 이것이 상용화된다면, 수천만 원짜리 HBM GPU 수십 장을 연결해야만 돌릴 수 있었던 거대 모델을 단일 노드나 소규모 클러스터에서 돌릴 수 있게 됩니다. 비용 효율성(Unit Economics) 측면에서 엄청난 파괴력을 가질 겁니다.

### 2. Processing-Near-Memory (PNM)

"데이터를 프로세서로 가져오지 말고, 프로세서를 데이터 옆으로 보내라."

오래된 아이디어지만, LLM 시대에 비로소 빛을 발하고 있습니다. 모델 사이즈가 TB 단위로 커지는 상황에서, 데이터를 이동시키는 에너지와 Latency가 전체 성능을 좌우하기 때문입니다. 특히 HBF와 결합하여 Flash 메모리 컨트롤러 근처에서 간단한 연산을 수행한다면 데이터 이동을 획기적으로 줄일 수 있습니다.

### 3. 3D Memory-Logic Stacking

이미 HBM에서 보고 있는 기술이지만, 로직과 메모리를 더 공격적으로 수직 적층(Stacking)하는 것을 의미합니다. TSV(Through-Silicon Via) 기술의 발전이 필수적인데, 이는 칩 간의 물리적 거리를 줄여 Latency를 최소화하는 가장 확실한 방법입니다.

### 4. Low-Latency Interconnect

모델이 하나의 칩에 들어가지 않을 때, 우리는 Model Parallelism을 사용합니다. 이때 칩 간 통신(Interconnect) 속도가 전체 인퍼런스 속도를 결정짓습니다. Patterson은 대역폭(Throughput)보다는 **Latency** 에 집중한 인터커넥트가 필요하다고 역설합니다. 작은 패킷을 얼마나 빨리 옆 칩으로 보내느냐가 관건이라는 것이죠.

## 엔지니어의 시선: 현실적인 과제들

솔직히 말해서, 이 제안들이 당장 내년 상용 제품에 적용되기는 쉽지 않아 보입니다.

1.  **HBF의 내구성과 발열:** Flash를 HBM처럼 긁어대면 셀 수명(Endurance)과 컨트롤러 발열을 어떻게 감당할 것인가? 이에 대한 명확한 해법이 필요합니다.
2.  **소프트웨어 스택:** NVIDIA의 해자(Moat)는 CUDA입니다. 새로운 아키텍처(PNM 등)가 나오더라도, PyTorch나 vLLM 같은 프레임워크에서 투명하게 지원되지 않으면 '예쁜 쓰레기'가 될 수 있습니다.

## 결론 (Verdict)

David Patterson의 이번 논문은 **"더 이상 무식하게 HBM만 늘리는 방식으로는 한계가 있다"** 는 강력한 메시지입니다. AI 인프라를 다루는 엔지니어라면, 단순히 GPU 스펙 시트의 FLOPS만 볼 것이 아니라 **Memory Hierarchy** 전체를 다시 설계해야 할 시점이 왔습니다.

특히 **High Bandwidth Flash** 는 주의 깊게 지켜봐야 할 기술입니다. 만약 이것이 성공한다면, 우리는 지금의 1/10 비용으로 GPT-5급 모델을 서빙하는 시대를 맞이할지도 모릅니다. 하드웨어 아키텍처의 황금기는 아직 끝나지 않았습니다.
