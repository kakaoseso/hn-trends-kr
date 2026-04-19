---
title: "Apple Silicon에서 WebAssembly로 Zero-Copy GPU 추론하기: 낭만과 현실"
description: "Apple Silicon에서 WebAssembly로 Zero-Copy GPU 추론하기: 낭만과 현실"
pubDate: "2026-04-19T10:39:20Z"
---

15년 넘게 시스템 엔지니어링과 아키텍처 설계를 해오면서 가장 지겹도록 마주친 딜레마 중 하나는 바로 '격리(Isolation)냐, 성능(Performance)이냐'입니다. 보안과 이식성을 위해 샌드박스를 도입하면 필연적으로 직렬화(Serialization)와 메모리 복사라는 엄청난 오버헤드가 발생합니다. 특히 LLM 추론처럼 수백 MB의 데이터를 GPU로 밀어 넣어야 하는 상황에서, PCIe 버스를 타는 메모리 복사는 그 자체로 병목이 됩니다.

최근 Hacker News를 뜨겁게 달군 [Zero-Copy GPU Inference from WebAssembly on Apple Silicon](https://abacusnoir.com/2026/04/18/zero-copy-gpu-inference-from-webassembly-on-apple-silicon/) 아티클은 이 오래된 난제를 Apple Silicon의 UMA(Unified Memory Architecture)를 이용해 우아하게 우회하는 방법을 보여줍니다. 요즘 기술 블로그 생태계에 LLM이 대충 요약해 주는 영혼 없는 글들이 넘쳐나서 피곤하던 차에, 이런 진짜 '엔지니어링' 냄새가 나는 하드코어한 실험은 참 반갑습니다.

이 글에서는 해당 아티클이 제시한 기술적 기반을 해부해 보고, 이것이 실제 프로덕션 레벨에서 어떤 의미를 가지는지, 그리고 HN 커뮤니티에서 오갔던 날선 비판들까지 제 개인적인 경험을 곁들여 파헤쳐 보겠습니다.

## 왜 이것이 평소에는 지옥 같은 작업인가?

기본적으로 WebAssembly(Wasm)는 샌드박스입니다. Wasm 모듈에게 세상은 자신에게 할당된 플랫한 바이트 배열(Linear Memory)이 전부입니다. 반면, NVIDIA나 AMD 같은 외장 GPU(Discrete GPU)는 PCIe 버스 건너편에 자신만의 VRAM을 가지고 있습니다.

이 둘을 연결하려면 다음의 끔찍한 과정을 거쳐야 합니다.
1. Wasm 샌드박스 내부의 데이터를 호스트 메모리로 복사 (Copy 1)
2. 호스트 메모리에서 PCIe 버스를 태워 GPU VRAM으로 복사 (Copy 2)

이 임피던스 불일치(Impedance mismatch)는 지연시간(Latency)을 늘리고 메모리 사용량을 두 배로 만듭니다. 하지만 Apple Silicon은 CPU와 GPU가 물리적으로 동일한 메모리를 공유하는 UMA 구조를 채택하면서 이 물리적 한계를 지워버렸습니다.

## 마법을 구현하는 3단계 체인

솔직히 처음 이 글의 제목을 봤을 때 '또 드라이버 레벨에서 위험한 해킹을 했겠구나' 싶었습니다. 과거에도 이런 류의 Zero-copy 시도들은 OS 업데이트 한 번에 박살 나는 경우가 허다했으니까요. 하지만 원작자의 접근 방식은 OS가 보장하는 컨트랙트를 영리하게 조합한, 매우 우아한 방식이었습니다.

- **Link 1: mmap을 통한 메모리 정렬**
ARM64 macOS에서 `mmap`에 `MAP_ANON | MAP_PRIVATE` 플래그를 주면 16KB 단위로 정렬된 주소를 반환합니다. 우연이 아니라 ARM64의 페이지 크기이며, Apple의 Metal API는 바로 이 정렬된 메모리를 요구합니다.

- **Link 2: Metal의 Zero-copy 버퍼 생성**
`MTLDevice.makeBuffer(bytesNoCopy:)` API를 사용하면 앞서 만든 메모리 포인터를 복사 없이 Metal 버퍼로 래핑할 수 있습니다. 원작자의 벤치마크에 따르면 명시적 복사를 할 때는 16.78MB의 RSS 증가가 있었지만, 이 방식을 쓰면 0.03MB(측정 노이즈 수준)만 증가했습니다.

- **Link 3: Wasmtime에 커스텀 Allocator 주입**
Wasmtime의 `MemoryCreator` 트레이트를 구현하여, Wasmtime이 자체적으로 메모리를 할당하는 대신 우리가 만든 `mmap` 영역을 Linear Memory로 사용하도록 강제합니다.

결과적으로 Wasm 모듈이 자신의 메모리에 행렬 데이터를 쓰면, GPU가 동일한 물리 메모리에서 곧바로 읽어 연산을 수행합니다. 128x128 GEMM 테스트에서 단 하나의 데이터 오염도 없었다는 것은 이 파이프라인이 완벽히 정렬되었음을 의미합니다.

## KV 캐시의 이식성: 진정한 게임 체인저

제가 이 아티클에서 가장 흥미롭게 본 부분은 단순한 속도 향상이 아니라, 상태를 가진(Stateful) 추론의 이식성입니다.

LLM 추론에서 컨텍스트가 길어질수록 KV 캐시는 수백 MB에서 수 GB까지 비대해집니다. 원작자는 이 KV 캐시가 Wasm이 통제하는 메모리에 존재한다는 점을 이용해, 이를 `safetensors` 포맷으로 디스크에 직렬화(Serialization)하고 나중에 복원하는 실험을 했습니다.

- **Serialize (24 tokens):** 1.1 ms
- **Restore from disk:** 1.4 ms
- **Re-prefill from scratch:** 67.7 ms

단 24개의 토큰만으로도 복원 속도가 5.45배 빠릅니다. 컨텍스트가 4,096 토큰 수준으로 길어지면 복원 속도는 재연산 대비 100배 이상 빠를 것으로 추정됩니다. 

이 패턴은 2010년대 초반 유행했던 '프로세스 마이그레이션(Process Migration)' 연구를 떠올리게 합니다. 대화형 AI의 상태(State)를 그대로 얼려서(Freeze) 다른 머신으로 옮기거나 나중에 재개할 수 있다는 것은, 향후 Edge AI와 분산 추론 아키텍처에서 엄청난 무기가 될 수 있습니다.

## Hacker News의 비판적 시각: 현실 체크

물론 HN 커뮤니티의 시니어 엔지니어들은 호락호락하지 않았습니다. 몇 가지 날카로운 지적들이 오갔고, 저 역시 상당 부분 동의합니다.

첫째, 'x86도 예전부터 통합 메모리를 썼다'는 지적입니다. 맞습니다. Intel이나 AMD의 내장 그래픽(iGPU)도 CPU와 RAM을 공유해 왔습니다. 하지만 Apple Silicon이 특별한 이유는 대역폭과 용량입니다. M4 Max는 최대 128GB의 LPDDR5X 메모리를 사용하며 대역폭이 ~500GB/s에 달합니다. 일반적인 x86 노트북의 옹색한 대역폭으로는 수십 GB의 LLM 모델을 쾌적하게 돌리기 어렵습니다. Apple이 특별한 기술을 발명했다기보다는, UMA 구조에 무식할 정도로 자본을 때려 박아 하이엔드급 성능을 뽑아낸 것이 핵심입니다.

둘째, 보안(Security) 문제입니다. 'Wasm의 샌드박스 보안은 끝났다'는 냉소적인 댓글이 있었죠. Wasm 메모리의 raw 포인터를 GPU 드라이버에 직접 넘기는 행위는, 만약 GPU 드라이버에 취약점이 있다면 샌드박스 탈출로 이어질 수 있는 거대한 공격 표면(Attack Surface)을 엽니다. 이 기술이 웹 브라우저(Chrome, Safari 등)에서 당장 쓰일 수 없는 이유이기도 합니다. 원작자도 Wasmtime이라는 서버/호스트 런타임을 사용했습니다.

## 총평: 프로덕션 레디인가, 장난감인가?

결론부터 말하자면, 원작자가 만들고 있는 Driftwood 프로젝트는 아직 실험실 수준의 장난감(Toy)입니다. 당장 내일 회사 프로덕션에 도입할 기술은 아닙니다.

하지만 이 실험이 증명한 'Primitive(원시 기반 기술)'의 가치는 엄청납니다. Wasm의 뛰어난 이식성과 샌드박싱 능력을 유지하면서도, 하드웨어 가속기의 성능을 100% 끌어다 쓸 수 있다는 것을 증명했으니까요. 특히 로컬 LLM 구동이 트렌드가 되어가는 지금, 모바일 기기나 Edge 서버에서 AI 에이전트를 구동할 때 이와 같은 Zero-copy 아키텍처는 선택이 아닌 필수가 될 것입니다.

오랜만에 기술의 본질을 파고드는, 엔지니어의 가슴을 뛰게 하는 훌륭한 해킹이었습니다. 모델 스왑 테스트나 대규모 스케일업 등 원작자의 다음 후속 글이 벌써부터 기다려집니다.

---
**References:**
- Original Article: [Zero-Copy GPU Inference from WebAssembly on Apple Silicon](https://abacusnoir.com/2026/04/18/zero-copy-gpu-inference-from-webassembly-on-apple-silicon/)
- Hacker News Thread: [https://news.ycombinator.com/item?id=47820195](https://news.ycombinator.com/item?id=47820195)
