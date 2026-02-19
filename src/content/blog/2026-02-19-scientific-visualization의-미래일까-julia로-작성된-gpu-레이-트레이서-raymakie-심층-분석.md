---
title: "Scientific Visualization의 미래일까? Julia로 작성된 GPU 레이 트레이서 RayMakie 심층 분석"
description: "Scientific Visualization의 미래일까? Julia로 작성된 GPU 레이 트레이서 RayMakie 심층 분석"
pubDate: "2026-02-19T22:14:48Z"
---

솔직히 말해봅시다. 엔지니어링이나 과학 시뮬레이션 데이터를 다루다 보면 '시각화' 단계에서 항상 병목이 발생합니다. Python의 Matplotlib이나 Julia의 Plots.jl로 데이터를 확인하다가, 논문이나 프레젠테이션을 위해 '진짜' 퀄리티가 필요해지면 어떻게 하시나요? 보통 데이터를 OBJ나 VDB로 export하고, Blender나 Houdini를 켜서, 셰이더를 다시 짜고 렌더링을 겁니다. 이 과정에서 인터랙티브함은 사라지고, 데이터 파이프라인은 끊어집니다.

최근 Hacker News에서 꽤 흥미로운 프로젝트가 올라왔습니다. 바로 **RayMakie** 와 **Hikari** 입니다. 요약하자면, Julia 언어 하나로 작성된 물리 기반(Physically-based) GPU 레이 트레이서가 Julia의 시각화 생태계인 Makie에 직접 통합되었다는 소식입니다.

15년 넘게 그래픽스와 컴퓨팅 파이프라인을 봐온 입장에서, 이건 단순한 "예쁜 그림 그리기 도구"가 아닙니다. 이건 **High-Performance Computing(HPC) 언어가 그래픽스 파이프라인을 어떻게 흡수할 수 있는지 보여주는 중요한 사례** 입니다.

### 1. RayMakie와 Hikari: 무엇이 다른가?

기존의 OpenGL/Vulkan 기반 래스터라이저와 달리, Hikari는 **pbrt-v4** (물리 기반 렌더링의 교과서적인 레퍼런스 구현체)를 Julia로 포팅한 Wavefront Path Tracer입니다. 핵심은 다음과 같습니다.

- **Pure Julia & GPU Backend:** C++ 코드를 호출하는 게 아니라, Julia 코드가 GPU 커널로 컴파일됩니다. `KernelAbstractions.jl`을 사용하여 단일 코드베이스로 NVIDIA(CUDA), AMD(ROCm), 그리고 CPU까지 지원합니다.
- **Makie 통합:** 기존에 `mesh!`, `surface!` 등으로 그리던 코드를 그대로 두고, 백엔드만 RayMakie로 교체하면 Path Tracing이 적용됩니다.

![BOMEX cumulus clouds rendered with volumetric path tracing](https://makie.org/website/bonito/png/breeze7235631823897871785.png)

위 이미지는 MIT의 기후 모델링 얼라이언스(CliMA)에서 시뮬레이션한 구름 데이터를 렌더링한 것입니다. 이걸 Blender로 가져가서 렌더링하는 게 아니라, 시뮬레이션이 돌아가는 Julia 환경 안에서 바로 렌더링했다는 점이 핵심입니다.

### 2. 기술적 깊이: Multiple Dispatch의 마법

제가 이 프로젝트에서 가장 주목한 부분은 **커스텀 피직스(Custom Physics)** 구현 방식입니다. 보통 상용 렌더러에서 빛의 굴절 방식을 바꾸려면 C++로 셰이더를 짜거나 플러그인을 만들어야 합니다. 하지만 Julia의 **Multiple Dispatch** 덕분에, 사용자는 렌더러 코드를 건드리지 않고도 새로운 물리 법칙을 주입할 수 있습니다.

블로그 포스트에 소개된 블랙홀 렌더링 예시를 봅시다. `Hikari.Medium`을 상속받아 `SpacetimeMedium`을 정의하고, 빛이 휘어지는 `apply_deflection` 함수만 오버로딩하면 끝입니다. 이 코드는 기존 렌더러 커널과 함께 GPU 코드로 컴파일되어 돌아갑니다.

![Black hole with accretion disk and gravitational lensing](https://makie.org/website/bonito/png/blackhole2083336987488631976.png)

이건 과학자들에게 엄청난 무기입니다. 단순히 데이터를 보여주는 게 아니라, **시각화 엔진 자체를 시뮬레이션 도구로 쓸 수 있다** 는 뜻이니까요. 식물학 연구자가 잎의 투과율을 계산하거나, 입자 물리학자가 검출기 내부를 시각화할 때 별도의 그래픽스 엔지니어가 필요 없게 됩니다.

### 3. Hacker News의 반응: 기대와 우려

이 프로젝트에 대한 커뮤니티의 반응은 뜨거우면서도 냉정했습니다. 특히 **AMD GPU 지원** 에 대한 언급이 많았습니다.

> "Cross-vendor GPU support... This is why I wish Julia were the language for ML and sci comp..."

보통 이런 프로젝트는 NVIDIA CUDA 전용으로 시작하기 마련인데, AMD ROCm을 1st-party로 지원한다는 점이 인상적입니다. 이는 Julia의 GPU 컴파일러 스택이 꽤 성숙했다는 증거이기도 합니다.

하지만, Julia의 고질적인 문제인 **TTFX (Time To First X)**, 즉 초기 컴파일 지연 문제는 여전히 논쟁거리입니다. HN 댓글에서도 "Julia는 멋지지만, 컴파일 기다리다가 지친다", "시스템 이미지(Sysimage) 빌드하는 데 시간이 너무 걸린다"는 불만이 터져 나왔습니다. 저도 동의합니다. 인터랙티브한 탐색이 목표라지만, 처음 창을 띄우기까지 수십 초가 걸린다면 그 '인터랙티브'의 의미가 퇴색되니까요.

또한, **"왜 Python이 아닌가?"** 라는 질문도 빠지지 않았습니다. Python 생태계가 압도적이긴 하지만, Python만으로는 이런 수준의 GPU 커널 통합을 '단일 언어'로 해내기 어렵습니다. Python은 결국 C++/CUDA를 감싸는 접착제(Glue) 역할을 하지만, Julia는 그 자체로 고성능 커널이 되기 때문입니다.

### 4. 내 생각: 아직은 '얼리 어답터'의 영역

RayMakie는 아직 Alpha 단계입니다. 블로그에서도 메모리 관리 문제(VRAM 부족), BVH 생성 최적화 등 해결해야 할 과제들을 솔직하게 인정하고 있습니다. 프로덕션 레벨의 렌더링을 기대하고 접근한다면 실망할 수 있습니다.

하지만 **Scientific Digital Twin** 이나 **복잡한 시뮬레이션 검증** 이 필요한 시니어 엔지니어라면, 이 프로젝트를 주시해야 합니다. 데이터와 렌더링 파이프라인이 하나로 합쳐졌을 때의 생산성 향상은 상상 이상일 수 있습니다.

**결론:**
만약 여러분이 Julia를 사용하는 연구 그룹에 있거나, 고성능 컴퓨팅(HPC) 시각화에 관심이 있다면 RayMakie는 게임 체인저가 될 잠재력이 있습니다. 하지만 일반적인 데이터 사이언티스트라면? 아직은 Python 생태계에 머무르셔도 좋습니다. 다만, **"언어 자체가 렌더러가 되는"** 이 패러다임이 어디까지 발전할지는 꼭 지켜보시길 바랍니다.

![Alpine terrain with volumetric clouds](https://makie.org/website/bonito/png/terrain794547408289448833.png)

*참고: 이 프로젝트는 오픈소스이며 [GitHub](https://github.com/MakieOrg/Makie.jl)에서 확인할 수 있습니다.*
