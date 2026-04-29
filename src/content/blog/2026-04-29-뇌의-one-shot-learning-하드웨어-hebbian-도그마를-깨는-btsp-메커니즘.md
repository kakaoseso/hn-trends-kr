---
title: "뇌의 One-Shot Learning 하드웨어: Hebbian 도그마를 깨는 BTSP 메커니즘"
description: "뇌의 One-Shot Learning 하드웨어: Hebbian 도그마를 깨는 BTSP 메커니즘"
pubDate: "2026-04-29T01:04:07Z"
---

최근 LLM이나 거대한 딥러닝 모델들을 튜닝하다 보면 항상 부딪히는 근본적인 의문이 있습니다. "왜 AI는 단순한 패턴 하나를 학습하기 위해 수만 번의 Epoch와 엄청난 컴퓨팅 리소스가 필요한데, 인간은 뜨거운 난로를 단 한 번만 만져보고도 평생 그 위험성을 학습할까?"

수십 년 동안 신경과학과 이를 모방한 컴퓨터 과학(Artificial Neural Networks)은 하나의 거대한 도그마에 지배받아 왔습니다. 바로 1949년에 제안된 Hebbian plasticity입니다. "Neurons that fire together, wire together"라는 이 유명한 문구는 오늘날 우리가 사용하는 머신러닝의 근간이 되었습니다. 하지만 최근 신경과학계에서 이 70년 된 패러다임을 뒤흔드는 새로운 하드웨어적 메커니즘이 발견되었습니다. 

솔직히 엔지니어로서 이 기사를 읽고 머리를 한 대 맞은 것 같았습니다. 우리가 지금까지 뇌를 모방한다고 만들었던 아키텍처가, 사실은 뇌의 아주 일부분(그것도 꽤 비효율적인 부분)에 불과했을지도 모르기 때문입니다. 오늘은 이 새로운 발견인 BTSP(Behavioral timescale synaptic plasticity)에 대해 엔지니어링 관점에서 딥다이브 해보겠습니다.

## 70년 된 레거시 시스템: Hebbian Learning의 한계

전통적인 Hebbian 모델은 두 뉴런이 밀리초(milliseconds) 단위로 동시에 활성화될 때 그 연결이 강해진다고 설명합니다. 우리가 흔히 아는 Gradient Descent를 통한 Weight 업데이트와 매우 유사한 개념입니다. 

![Santiago Ramón y Cajal’s drawings of neurons](https://www.quantamagazine.org/wp-content/uploads/2026/04/C-Pyramidal-cells-cr.Santiago-Ramon-y-Cajal_Publiic-Domain.webp)

이 방식은 새로운 언어를 배우거나 도시의 지도를 외우는 데는 적합합니다. 수많은 반복(Iteration)을 통해 서서히 네트워크를 강화하니까요. 하지만 심각한 문제가 하나 있습니다. 바로 Latency와 One-shot learning의 부재입니다. 야생에서 포식자를 만났을 때, "아, 이 패턴이 10번 반복되었으니 이제 도망가야지"라고 학습한다면 그 개체는 이미 유전자 풀에서 삭제되었을 것입니다.

기존의 Hebbian 이론은 인간의 '행동'이라는 상대적으로 느린 스케일(수 초)을 밀리초 단위의 시냅스 변화로 어떻게 캡처하는지 설명하지 못했습니다. 엔지니어링으로 치면, 시스템 로그는 마이크로초 단위로 남는데 유저의 세션은 초 단위로 유지될 때, 이 둘을 매핑할 방법이 없었던 셈입니다.

## 패러다임 시프트: BTSP와 Dendritic Plateau

2014년 Jeffrey Magee 연구팀은 살아있는 쥐의 해마(Hippocampus)를 관찰하다가 놀라운 현상을 발견합니다. 뉴런의 팔 역할을 하는 수상돌기(Dendrite)에서 발생하는 고원 전위(Plateau potential)라는 현상입니다.

![Dendrites](https://www.quantamagazine.org/wp-content/uploads/2026/04/Dendrite-cr.Jose-Calvo_Alamy.webp)

이들은 쥐가 특정 위치에 갔을 때, 단 한 번의 Dendritic plateau 발생만으로도 해당 위치를 기억하는 장소 세포(Place cell)가 형성되는 것을 확인했습니다. 여러 번의 반복 없이, 단 한 번의 강력한 전기적 신호로 시냅스가 재배선된 것입니다.

이것이 바로 **BTSP** (Behavioral timescale synaptic plasticity)입니다. 여기서 가장 중요한 차이는 '시간'입니다. 밀리초 단위가 아니라, 무려 수 초(seconds)에 걸쳐 일어납니다. Plateau 이벤트가 발생하기 6~8초 전후로 활성화된 시냅스들이 모두 강화됩니다. 

## 메커니즘 딥다이브: TTL 캐시와 Batch Update

어떻게 8초 전에 일어난 일을 뉴런이 기억하고 업데이트할까요? 이 부분의 메커니즘을 읽으면서 저는 분산 시스템의 Eventual Consistency나 캐시 시스템이 떠올랐습니다.

연구자들은 이를 Eligibility traces라는 생화학적 서명으로 설명합니다.

- **Eligibility Traces:** 특정 경험으로 인해 활성화된 시냅스에 남는 일종의 마커입니다. 이 마커는 수 초 동안 유지됩니다. (마치 TTL이 설정된 캐시 키 같습니다)
- **Dendritic Plateau:** 이후 강력한 전위가 수상돌기 전체에 퍼집니다. (일종의 Global Trigger 신호입니다)
- **Batch Update:** Plateau 신호가 퍼질 때, Eligibility trace가 남아있는(TTL이 만료되지 않은) 모든 시냅스의 연결 강도가 일괄적으로 강화됩니다. 

최근 연구에 따르면 이 과정에서 CaMKII라는 단백질이 수용체의 표면적과 개수를 물리적으로 늘려버린다고 합니다. 소프트웨어적인 Weight 업데이트가 아니라, 대역폭(Bandwidth) 자체를 하드웨어적으로 넓혀버리는 것입니다.

## 머신러닝의 'Credit Assignment Problem'에 대한 해답?

AI 분야에서 가장 골치 아픈 문제 중 하나가 Credit assignment problem입니다. 수많은 뉴런 중 어떤 뉴런이 현재의 성공(또는 실패)에 기여했는지 파악하고 보상을 분배하는 문제입니다. 현재 우리는 Backpropagation이라는 무식하지만 수학적으로 우아한 방법을 씁니다.

하지만 뇌는 BTSP를 통해 이 문제를 극복하는 것으로 보입니다. 모든 활성 뉴런을 강화하는 것이 아니라, Eligibility trace가 남은 '관련성 있는' 뉴런만 선택적으로 강화하기 때문입니다. 이는 향후 새로운 형태의 딥러닝 아키텍처나 Spiking Neural Networks(SNN)를 설계할 때 엄청난 영감을 줄 수 있습니다.

## 결론: 엔지니어의 시선으로 본 뇌의 최적화

Hacker News 커뮤니티나 테크 씬에서는 종종 AGI(Artificial General Intelligence)가 언제 올 것인가에 대해 격렬한 토론이 벌어집니다. 하지만 이번 BTSP 발견을 보면, 우리는 아직 우리가 모방하려는 타겟 시스템(뇌)의 기초적인 하드웨어 스펙조차 다 파악하지 못했다는 겸손함을 배우게 됩니다.

물론 BTSP가 기존의 Hebbian learning을 완전히 대체하는 것은 아닙니다. 초기 뇌의 발달이나 기본적인 배선은 여전히 Hebbian 방식이 주도하겠지만, 성인의 에피소드 기억이나 즉각적인 생존 반응(One-shot learning)은 BTSP가 담당하는 하이브리드 아키텍처일 확률이 높습니다.

개인적으로는 이 메커니즘이 향후 강화학습(RL) 모델의 새로운 보상 함수 설계에 어떻게 응용될 수 있을지 매우 기대됩니다. 뇌는 우리가 생각했던 것보다 훨씬 더 정교한 캐싱 시스템과 배치 업데이트 로직을 가진 궁극의 분산 시스템이니까요.

---
**References:**
- Quanta Magazine: [A New Type of Neuroplasticity Rewires the Brain After a Single Experience](https://www.quantamagazine.org/a-new-type-of-neuroplasticity-rewires-the-brain-after-a-single-experience-20260424/)
- Hacker News Discussion: [Behavioral timescale synaptic plasticity rewires the brain after an experience](https://news.ycombinator.com/item?id=47921610)
