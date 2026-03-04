---
title: "Spiking Neural Networks and the Neuromorphic Hype: Why I am Skeptical About Ditching Matrix Math"
description: "Spiking Neural Networks and the Neuromorphic Hype: Why I am Skeptical About Ditching Matrix Math"
pubDate: "2026-03-04T14:02:59Z"
---



매일같이 PyTorch 코드를 최적화하고, CUDA OOM 에러와 싸우며, 트랜스포머 모델에 더 많은 H100 GPU를 쏟아붓는 것이 우리 엔지니어들의 일상이다. 우리는 인공신경망을 역전파(Backpropagation)와 거대한 행렬 곱셈(Matrix Multiplication)으로 구동되는 정적인 수학적 그래프로 취급하는 데 익숙하다.

그런데 최근 뇌과학과 AI의 교차점을 다룬 흥미로운 글을 읽었다. 결론부터 말하자면, 우리 두개골 안의 생물학적 하드웨어인 '뇌'는 우리가 GPU 위에서 돌리는 모델들과는 완전히 다른 방식으로 작동한다. 미적분학(Calculus) 없이 학습하고, 연속적인 실수가 아닌 이산적인 스파이크(Spike)로 통신한다.

이론은 정말 우아하고 매력적이다. 하지만 15년 차 엔지니어의 관점에서, 이것이 당장 우리의 프로덕션 환경이나 행렬 연산의 종말을 의미하는지에 대해서는 상당히 회의적이다. 오늘은 뇌가 정보를 처리하는 방식의 기술적 디테일을 파헤쳐보고, 왜 내가 당분간은 계속 GPU를 사 모을 계획인지 솔직한 생각을 공유하고자 한다.

## 1. Predictive Coding: 뇌는 거대한 환각 머신이다

AI에서 시각적 인지(Perception)는 철저한 Bottom-up 프로세스다. CNN에 사과 이미지를 넣으면, 초기 레이어에서 엣지와 색상을 추출하고, 최종 레이어에서 '사과'라는 추상적 개념을 나타내는 임베딩 벡터를 출력한다.

하지만 인간의 인지는 이와 다르다. 오히려 Top-down에 가깝다. 뇌는 단순히 시신경에서 픽셀이 올라오기를 기다리는 수동적인 컨베이어 벨트가 아니다. 뇌는 끊임없이 세상을 시뮬레이션하는 환각 머신(Hallucination Machine)이다.



이것이 바로 Predictive Coding의 핵심이다. 당신이 나무를 볼 때, 뇌는 자신이 무엇을 보게 될지 미리 예측(Prediction)한다. 그리고 그 Top-down 예측값과 눈에서 들어오는 Bottom-up 감각 데이터를 비교한다. 뇌는 전체 이미지를 상위 레이어로 올려보내지 않는다. 오직 예측 오차(Prediction Error), 즉 'Diff'만을 위로 전달한다. 우리의 의식적 경험은 뇌가 만들어낸 현실에 대한 최선의 추측이며, 감각적 Diff에 의해 실시간으로 패치(Patch)되는 과정일 뿐이다.

솔직히 이 구조는 비디오 스트리밍의 압축 알고리즘이나 React의 Virtual DOM이 상태 변화(Diff)만 렌더링하는 방식과 놀랍도록 유사하다. 자연은 이미 수백만 년 전에 대역폭 최적화의 끝판왕을 만들어둔 셈이다.

## 2. 역전파(Backpropagation)가 생물학적으로 불가능한 이유

우리가 사랑하는 Backpropagation은 전지전능한 글로벌 코치와 같다. 네트워크가 실수를 하면 글로벌 에러 그래디언트를 계산하고, 미적분학을 사용해 모든 가중치를 완벽하게 뒤에서부터 업데이트한다.

하지만 생물학적 뉴런은 이를 수학적으로 수행할 수 없다. 시냅스는 일방통행이며, 연속적인 숫자(예: 0.75)를 출력하는 대신 이산적인 이진 전기 신호(Spike)를 시간에 따라 발사한다. 여기서 뇌가 사용하는 로컬 학습 방식이 바로 STDP(Spike-Timing-Dependent Plasticity)다.



- **인과관계 (Cause and Effect):** 뉴런 A가 뉴런 B보다 몇 밀리초 먼저 발사되면, 뇌는 A가 B를 유발했다고 가정하고 시냅스를 강화한다.
- **우연의 일치 (Coincidence):** A가 B보다 늦게 발사되면 연결은 약화된다.

모든 시냅스가 글로벌 에러 없이 철저히 로컬 타이밍에 의존해 스스로 가중치를 업데이트하는 완전한 분산 시스템이다.

## 3. 도파민과 TD Error: 컴퓨터 과학과 신경과학의 소름 돋는 교차점

STDP만으로는 목표를 향한 복잡한 학습을 할 수 없다. 여기서 뇌는 도파민이라는 세 번째 요소를 도입한다.

흥미로운 점은 80년대 AI 엔지니어들이 에이전트를 학습시키기 위해 발명한 강화학습(Reinforcement Learning) 알고리즘인 TD Learning과, 90년대 신경과학자들이 원숭이 뇌에서 측정한 도파민 뉴런의 발사 패턴이 수학적으로 완벽하게 일치했다는 사실이다.



도파민은 단순한 '쾌락' 호르몬이 아니다. 그것은 보상 예측 오차(Reward Prediction Error, RPE)다. 예상치 못한 보상이 주어지면 도파민이 폭발적으로 분비되고(Positive TD Error), 예측된 보상이 정확히 주어지면 도파민은 베이스라인을 유지한다(Zero TD Error). AI 엔지니어들이 Q-Learning을 통해 로봇을 학습시키는 코드가 우리 뇌 안에서 화학적으로 실행되고 있는 것이다.

## 4. Neuromorphic Hardware와 현실의 벽

이론이 이렇게 완벽하다면 왜 우리는 ChatGPT를 Spiking Neural Networks(SNN) 위에서 돌리지 않을까? 

우리는 철저히 GPU의 세계에 살고 있기 때문이다. GPU는 결정론적인 행렬 곱셈에는 괴물같이 강하지만, 생물학의 이산적이고 시간 기반의 이진 스파이크를 처리하는 데는 끔찍하게 비효율적이다. 이를 해결하기 위해 Intel의 Loihi 2나 SpiNNaker 같은 뉴로모픽 칩(Neuromorphic chips)이 등장했다. 이들은 행렬 연산을 하지 않고, 인공 실리콘 시냅스를 사용해 전기 스파이크를 보내며 하드웨어 네이티브로 STDP를 수행한다.



하지만 여기서 내 Principal Engineer로서의 회의감이 발동한다. Hacker News의 한 유저가 정확히 지적했듯, 뉴로모픽 칩은 지난 15년 동안 항상 "5년 뒤에 상용화될 기술"이었다.

가장 큰 문제는 SNN의 토폴로지(Topology)를 탐색하는 것이 지옥에 가깝다는 점이다. 우리는 어떤 네트워크 구조가 의미 있는지 아직 모른다. 토폴로지를 찾기 위해 실제 문제에 시뮬레이션을 돌려야 하는데, 스파이킹 활동의 멱법칙(Power law)이 개입하는 순간 이 시뮬레이션은 거대한 늪으로 변한다. 

## 5. 결론: 컴퓨터가 유용한 이유는 인간과 다르기 때문이다

Target Propagation이나 Feedback Alignment 같은 생물학적 역전파 대안 이론들이 나오고 있고, AI와 신경과학이 융합되는 청사진은 분명 경이롭다.



하지만 우리가 반드시 생물학을 맹목적으로 모방해야 할까? Hacker News 스레드에서 가장 내 뼈를 때린 코멘트는 이것이었다. 

> "컴퓨터가 유용한 이유는 정확히 우리와 다르기 때문이다. LLM이 유용한 이유도 인간과 다르기 때문이다."

내 CPU는 전하 누출(leak charge)을 시뮬레이션할 필요가 없다. 실리콘은 생물학적 타당성(Biological plausibility)을 훨씬 뛰어넘는 속도로 정보를 손실 없이 복사하고 처리할 수 있다. 비행기를 발명할 때 새의 펄럭이는 날개를 완벽하게 모방하는 대신 고정익과 제트 엔진을 만든 것처럼, 뇌의 구조를 1:1로 복제하는 것이 AGI로 가는 유일한 길이라고 생각하지 않는다.

SNN은 전력 효율성 측면에서 엣지 디바이스(Edge devices)에 혁명을 일으킬 잠재력이 있다. 하지만 SNN 소프트웨어 진영에서 'AlexNet' 모먼트가 터지기 전까지는, 전용 실리콘 하드웨어에 베팅하는 것은 너무 이르다. 

생물학적 영감은 훌륭한 나침반이지만, 그것이 우리의 엔지니어링 제약사항이 되어서는 안 된다. 나는 당분간 계속해서 행렬 곱셈의 세계에 머물며, PyTorch와 GPU 클러스터의 영점을 맞추는 데 집중할 것이다.

**References:**
- Original Article: https://metaduck.com/reverse-engineering-the-wetware-spiking-networks-td-errors-and-the-end-of-matrix-math/
- Hacker News Thread: https://news.ycombinator.com/item?id=47211034
