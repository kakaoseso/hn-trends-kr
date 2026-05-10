---
title: "WebGPU와 Surfel로 구현한 브라우저 위 실시간 GI: 혁신일까, 아직은 장난감일까?"
description: "WebGPU와 Surfel로 구현한 브라우저 위 실시간 GI: 혁신일까, 아직은 장난감일까?"
pubDate: "2026-05-10T09:02:16Z"
---

최근 웹 그래픽스 생태계를 보면 참 격세지감을 느낀다. WebGL 시절의 제한적인 렌더링 파이프라인을 넘어, 이제는 WebGPU를 통해 브라우저 위에서 수십 개의 Compute Pass를 돌리며 AAA급 게임에서나 쓰던 기술을 실험하는 시대가 왔다. 최근 Hacker News를 뜨겁게 달군 Jure Triglav의 아티클은 이 변화의 정점을 보여준다.

저자는 EA SEED의 SIGGRAPH 발표에서 영감을 받아, WebGPU 환경에서 Surfel(Surface Element)을 활용한 실시간 글로벌 일루미네이션(GI)을 구현했다. 무려 3,000줄이 넘는 TypeScript와 WGSL 코드로 말이다. 15년 차 엔지니어의 시각에서 이 파이프라인을 뜯어보고, 이것이 지니는 기술적 의미와 현재 웹 플랫폼이 가진 뼈아픈 한계들을 짚어보려 한다.

## 왜 픽셀이 아니라 Surfel인가?

실시간 렌더링에서 가장 큰 난제는 해상도다. 4K 화면의 수백만 픽셀 각각에 대해 Path Tracing을 돌리는 것은 최신 고성능 GPU로도 벅차다. 하물며 모바일 기기나 브라우저에서는 어림도 없는 소리다.

여기서 Surfel이 등장한다. Surfel은 3D 공간에 떠 있는 작은 원반(Disc) 형태의 표면 패치다. 조명 연산을 화면 해상도(Screen Resolution)에서 분리하여, 화면에 200만 개의 픽셀이 있더라도 실제로는 5만 개의 Surfel만 셰이딩하면 되도록 캐싱(Caching)하는 역할을 한다.

이 프로젝트의 파이프라인은 크게 4단계로 나뉜다.

- **Surfelization:** G-buffer(Normal, Depth, Albedo)를 렌더링하고, 화면 공간을 분석해 Surfel이 부족한 곳에 새로 생성한다.
- **Hash Grid:** Surfel들을 공간 데이터 구조(Cascaded 3D Grid)에 밀어 넣는다. 이때 Atomic operation과 Prefix-sum scan을 사용해 O(1) 수준의 룩업(Lookup)을 보장한다.
- **Integrator (Ray Tracing):** 각 Surfel에서 광선을 쏴서 주변의 빛을 수집한다.
- **Resolver:** 최종 화면의 픽셀을 그릴 때, 픽셀 주변의 Surfel 데이터를 긁어와 합성한다.

## WebGPU의 한계를 우회하는 꼼수들 (The Hacks)

이 글에서 가장 흥미로운 부분은 WebGPU라는 환경적 제약을 어떻게 엔지니어링으로 극복했는지다.

첫째, WebGPU에는 아직 하드웨어 Ray Tracing(RT) 확장이 없다. 저자는 이를 극복하기 위해 `three-mesh-bvh` 라이브러리를 사용해 Compute Shader 위에서 BVH(Bounding Volume Hierarchy) 순회를 소프트웨어로 구현했다. 텍스처 바인딩 제한 때문에 모든 재질을 하나의 2D Array Texture로 패킹하는 꼼수까지 동원했다. 우아하진 않지만, 실전에서는 이런 방식이 통한다.

둘째, 노이즈와 Temporal Lag의 딜레마다. Surfel이 사방으로 광선을 쏘면(Uniform sampling) 허공이나 어두운 벽만 맞추느라 정작 중요한 광원을 놓친다. 저자는 Hemi-oct-square 매핑을 통해 가중치 그리드를 만들어 중요도 샘플링(Guided Sampling)을 구현했다.

하지만 이것만으로는 부족하다. 필연적으로 발생하는 노이즈를 잡기 위해 이전 프레임의 데이터를 누적해야 하는데, 단순 지수 이동 평균(EMA)을 쓰면 화면이 움직일 때 빛이 질척거리는 고스트(Ghosting) 현상이 발생한다. 이를 해결하기 위해 저자는 EA SEED의 PICA PICA 엔진에서 쓰인 **MSME(Multi-scale mean estimator)** 를 도입했다. 단기 평균과 장기 평균을 동시에 추적해 변화에 빠르게 반응하면서도 노이즈를 억제하는 훌륭한 기법이다.

Hacker News의 한 유저는 이 대목에서 매우 날카로운 지적을 했다.
> "대부분의 GI 기술이 Temporal Caching과 Denoising에 의존해야 한다는 게 슬프다. 우리는 아마 영원히 노이즈 없고 즉각적인(Crisp) 그래픽으로 돌아가지 못할 것이다."

나 역시 100% 동의한다. 현대 그래픽스 파이프라인은 본질적으로 거대한 '노이즈 제거기'가 되어버렸다. 우리는 공간적 해상도를 포기하고 시간적 아티팩트(Temporal Artifacts)를 감수하는 방향으로 타협하고 있다.

## Light Leaks: 얇은 벽의 저주

Surfel은 기하학적 구조(Geometry)에 종속되지 않고 공간에 떠 있다. 그렇다 보니 얇은 벽을 사이에 두고 반대편 방의 빛을 끌어다 쓰는 '빛 샘(Light Leaks)' 현상이 발생한다.

저자는 이를 해결하기 위해 각 Surfel에 4x4 픽셀 크기의 초소형 360도 방사형 깊이 아틀라스(Radial Depth Atlas)를 쥐여주었다. 그리고 MSM(Moment Shadow Mapping)을 통해 통계적 모멘트(E[z], E[z²] 등)를 저장하여 부드러운 가시성 함수를 재구성했다. 

이건 정말 미친 짓(좋은 의미로)이다. 브라우저에서 이 정도의 수학적 연산과 메모리 대역폭을 감당하며 초당 60프레임을 뽑아낸다는 것은 WebGPU의 Compute 성능이 생각보다 훨씬 강력하다는 것을 증명한다.

## The Ugly Truth: WebGPU의 뼈아픈 한계

HN 커뮤니티에서 누군가 "왜 Surfel 대신 Light Probe를 쓰지 않았냐"고 물었다. 저자의 답변에서 현재 웹 그래픽스의 가장 큰 병목이 드러난다.

AAA 게임들은 즉각적인 빛 정보를 얻기 위해 Probe를, 디테일한 표현과 누수를 막기 위해 Surfel을 혼합해서 사용한다. 하지만 저자는 이 시스템에 Probe를 추가할 수 없었다. **Chrome의 WebGPU 구현체가 Compute Pass당 바인딩할 수 있는 Storage Buffer의 개수를 10개로 하드코딩해 두었기 때문이다.**

이 파이프라인은 이미 10개의 버퍼를 꽉 채워 쓰고 있다. 더 고도화된 렌더링 기법(예: ReSTIR PT)을 웹에서 구현하려는 수많은 엔지니어들이 이 황당한 10개 제한에 가로막혀 있다. 표준이 하드웨어의 가능성을 인위적으로 억누르고 있는 셈이다.

## 총평: 그래서 실무에 쓸 수 있는가?

결론부터 말하자면, 이 파이프라인은 아직 프로덕션 레벨이 아니다. 동적 객체(Dynamic Geometry)나 광택(Specular) 반사, 투명도 처리가 불가능하며, 얇은 메쉬나 빠르게 움직이는 광원 앞에서는 여전히 시스템이 무너진다.

하지만 이 프로젝트는 '장난감' 이상의 가치가 있다. WebGPU가 단순히 WebGL의 문법적 설탕(Syntactic Sugar)이 아니라, 브라우저를 하나의 거대한 GPGPU 플랫폼으로 만들어주었다는 것을 완벽하게 증명한 사례다. 13개 이상의 Compute Pass를 유기적으로 연결해 Spatial Grid를 빌드하고 Ray Tracing을 에뮬레이션하는 과정은 그 자체로 훌륭한 엔지니어링 마스터클래스다.

프론트엔드 생태계가 React 컴포넌트 렌더링 속도로 논쟁하는 동안, 웹의 한구석에서는 우주 빅뱅의 광자를 시뮬레이션하며 빛의 물리학을 렌더링하고 있다. 웹의 미래는 우리가 생각하는 것보다 훨씬 더 로우레벨(Low-level)에 있을지도 모른다.

---
**References:**
- 원문: [Surfel-based global illumination on the web](https://juretriglav.si/surfel-based-global-illumination-on-the-web/)
- Hacker News 토론: [HN Thread](https://news.ycombinator.com/item?id=48077395)
