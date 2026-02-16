---
title: "WebGL의 한계를 시험하다: 브라우저 기반 실시간 Path Tracing 구현과 기술적 함의"
description: "WebGL의 한계를 시험하다: 브라우저 기반 실시간 Path Tracing 구현과 기술적 함의"
pubDate: "2026-02-16T14:30:17Z"
---

솔직히 말해, 처음 이 프로젝트를 접했을 때 내 반응은 "굳이?"였습니다. 브라우저에서, 그것도 WebGPU가 아닌 WebGL 2.0 위에서 실시간 패스 트레이싱(Path Tracing)을 구현한다니. 이건 마치 자전거로 고속도로를 달리겠다는 소리처럼 들렸거든요. 하지만 Erich Lof가 공개한 **THREE.js PathTracing Renderer** 데모들을 직접 돌려보고 코드를 뜯어본 뒤, 제 생각은 완전히 바뀌었습니다.

이것은 단순한 그래픽스 데모가 아닙니다. 제약 사항이 극심한 환경에서 엔지니어링의 극한을 짜내면 어떤 결과물이 나올 수 있는지를 보여주는, 일종의 '최적화 차력쇼'에 가깝습니다. 오늘은 이 프로젝트가 어떻게 모바일 디바이스에서조차 60fps로 Global Illumination(GI)을 뽑아내는지, 그 기술적 배경과 함의를 깊게 파보려 합니다.

### 1. 왜 지금 WebGL인가? (WebGPU가 아니고?)

Hacker News의 댓글들을 보면 예상대로 "WebGPU가 나왔는데 왜 WebGL로 고생하냐"는 반응이 꽤 있습니다. 기술적으로 타당한 지적입니다. Compute Shader에 대한 접근성이나 스토리지 버퍼 제한 등을 고려하면 WebGPU가 패스 트레이싱에는 훨씬 적합하니까요.

하지만 이 프로젝트의 진가는 바로 **하위 호환성** 과 **최적화** 에 있습니다. 현재 WebGPU는 아직 모든 디바이스(특히 구형 모바일)에서 완벽하게 지원되지 않습니다. 이 렌더러는 WebGL 2.0을 사용하여, 소위 '감자(potato)'라고 불리는 저사양 폰에서도 돌아갑니다. 이는 엔지니어로서 "최신 기술 스택"에 의존하지 않고 알고리즘 레벨에서 문제를 해결하려 했다는 증거입니다.

### 2. 핵심 기술: Triangle이 아닌 'Quadric Shapes' BVH

보통의 레이 트레이싱 엔진은 수만 개의 삼각형(Triangle)으로 이루어진 메쉬를 처리하기 위해 BVH(Bounding Volume Hierarchy)를 구축합니다. 하지만 모바일 환경에서 깊은 BVH 트리를 순회하는 것은 엄청난 메모리 대역폭과 연산 비용을 요구합니다.

이 프로젝트의 가장 흥미로운 지점은 **Shapes BVH** 시스템입니다. 작성자는 복잡한 지오메트리를 수많은 삼각형으로 쪼개는 대신, 구(Sphere), 상자(Box), 원기둥(Cylinder) 같은 수학적 프리미티브(Quadric Shapes)로 근사하여 BVH를 구성했습니다.

> "Instead of handling thousands of small triangles... it works with larger, simpler shapes... reducing geometry count exponentially!"

예를 들어 'Invisible Date' 데모의 경우, 수천 개의 삼각형을 단 54개의 셰이프(Shape)로 압축했습니다. 이렇게 되면 BVH 트리의 깊이가 획기적으로 얕아지고, 레이(Ray) 교차 연산 비용이 급격히 줄어듭니다. 이것이 바로 모바일에서 60fps가 나오는 비결입니다. 수학적 최적화가 하드웨어의 한계를 뚫어버린 케이스죠.

![Cornell Box Demo](https://erichlof.github.io/THREE.js-PathTracing-Renderer/readme-Images/CornellBox-Render0.png)

### 3. Ray Marching과 Path Tracing의 하이브리드

또 하나 눈여겨볼 점은 **Ray Marching** 과 **Path Tracing** 의 혼용입니다. [Ocean and Sky Demo](https://erichlof.github.io/THREE.js-PathTracing-Renderer/Ocean_And_Sky_Rendering.html)를 보면, 무한히 펼쳐진 바다와 절차적(Procedural) 구름은 Ray Marching으로 처리하고, 그 위의 객체들은 Path Tracing으로 처리합니다.

순수 Path Tracing만으로는 광활한 지형이나 볼륨 효과를 실시간으로 처리하기 버겁습니다. 이 프로젝트는 두 기법을 적재적소에 섞어 사용함으로써, 물리 기반의 정확한 라이팅(Caustics, Soft Shadows)과 거대 스케일의 환경 렌더링이라는 두 마리 토끼를 잡았습니다. 특히 물 표면의 웨이브 시뮬레이션이 30-60fps로 유지되는 것은 WebGL 쉐이더 코드가 얼마나 타이트하게 짜여있는지를 방증합니다.

### 4. 게임 엔진으로서의 가능성: 물리 엔진과의 통합

단순 렌더링 데모를 넘어, 작성자는 이를 이용해 'Glider Ball 3D' 같은 게임을 만들었습니다. 여기서 주목할 점은 **물리 연산의 효율성** 입니다.

렌더링에 사용된 Quadric Shapes 수학 모델을 물리 충돌(Collision) 감지에도 그대로 재사용했습니다. 곡면(Curved surface) 위를 달리는 차량의 물리 연산을 위해 별도의 물리 메쉬를 굽는 것이 아니라, 렌더링에 쓰이는 수학적 정의(Implicit definition)를 그대로 활용해 충돌 체크를 수행합니다. 이는 메모리 사용량을 줄일 뿐만 아니라, 폴리곤 기반 물리 엔진에서 발생하는 '각진' 이동 문제를 원천적으로 해결합니다.

### 5. 한계점과 비판적 시각

물론 단점이 없는 것은 아닙니다.

- **노이즈(Noise)와 수렴 속도:** 실시간이라고는 하지만, 카메라를 움직이는 순간 샘플링이 초기화되면서 자글거리는 노이즈가 발생합니다. 커스텀 디노이저(Denoiser)가 탑재되어 있지만, 네이티브 RTX 앱들의 AI 기반 디노이징(DLSS 등)에 비하면 퀄리티가 떨어지는 것은 어쩔 수 없습니다.
- **UX/컨트롤:** HN 댓글에서도 지적되었듯, 카메라 컨트롤이 다소 직관적이지 않습니다. Orbit 컨트롤 대신 WASD와 마우스 캡처 방식을 사용하는데, 이는 기술 데모에는 적합할지 몰라도 일반 사용자 경험에는 마이너스 요소입니다.
- **코드 복잡도:** WebGL 쉐이더(GLSL)로 이 모든 로직을 구현하다 보니, 유지보수 난이도가 상당히 높을 것으로 보입니다. WebGPU로 넘어간다면 WGSL의 혜택을 받아 좀 더 구조적인 코딩이 가능할 것입니다.

### 결론: WebGPU 시대를 맞이하기 전의 위대한 유산

많은 엔지니어들이 "WebGPU가 나오면 공부해야지" 하며 기다릴 때, Erich Lof는 현재 주어진 도구(WebGL 2.0, Three.js)를 극한까지 밀어붙여 불가능해 보이는 것을 구현해냈습니다. 

이 프로젝트는 **"제약 사항이 혁신의 어머니"** 라는 엔지니어링의 격언을 다시 한번 상기시켜 줍니다. 물론 장기적으로 웹 그래픽스의 미래는 WebGPU에 있겠지만, 이 프로젝트에서 사용된 **Shapes BVH 최적화 기법** 이나 **하이브리드 렌더링 아키텍처** 는 플랫폼이 바뀌어도 여전히 유효한, 아니 더욱 빛을 발할 고급 기술들입니다.

지금 당장 여러분의 폰으로 [데모 링크](https://erichlof.github.io/THREE.js-PathTracing-Renderer/Geometry_Showcase.html)를 열어보세요. 그 작은 화면 속에서 빛이 튕겨 나가는 모습을 보고 있으면, 엔지니어로서 묘한 전율을 느끼게 될 겁니다.
