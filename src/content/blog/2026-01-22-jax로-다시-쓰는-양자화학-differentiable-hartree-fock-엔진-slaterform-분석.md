---
title: "JAX로 다시 쓰는 양자화학: Differentiable Hartree-Fock 엔진 slaterform 분석"
description: "JAX로 다시 쓰는 양자화학: Differentiable Hartree-Fock 엔진 slaterform 분석"
pubDate: "2026-01-22T10:18:41Z"
---

최근 몇 년간 **Differentiable Programming** (미분 가능한 프로그래밍)은 단순한 ML 트렌드를 넘어 과학 계산(Scientific Computing) 전반을 집어삼키고 있습니다. 렌더링 엔진이 미분 가능해지더니, 유체 역학이 들어오고, 이제는 **양자화학(Quantum Chemistry)** 까지 그 영역이 확장되었습니다.

오늘 소개할 프로젝트는 Hacker News에서 꽤 흥미로운 논의를 불러일으킨 **slaterform** 입니다. JAX로 작성된 Differentiable Hartree-Fock 솔버인데, 솔직히 처음엔 "또 JAX로 만든 장난감 하나 나왔네"라고 생각했습니다. 하지만 코드를 뜯어보고 데모를 돌려보니, 이건 단순한 장난감이 아니라 **Computational Chemistry의 미래가 어떻게 변할지 보여주는 중요한 이정표** 라는 생각이 들더군요.

오늘은 이 `slaterform`이 왜 중요한지, 그리고 엔지니어링 관점에서 어떤 의미가 있는지 깊게 파고들어 보겠습니다.

## Hartree-Fock이 뭔데? (짧은 요약)

양자화학을 전공하지 않은 분들을 위해 아주 짧게 설명하자면, **Hartree-Fock(하트리-폭)** 방법은 다체 시스템(원자, 분자 등)의 바닥 상태 에너지를 근사적으로 계산하는 가장 기초적인 알고리즘입니다. 슈뢰딩거 방정식을 풀고 싶은데 너무 복잡하니, "전자들이 서로 평균적인 전기장 안에서 움직인다"고 가정하고 푸는 것이죠.

전통적으로 이 분야는 **Fortran** 의 텃밭이었습니다. Gaussian, GAMESS, Molpro 같은 툴들은 수십 년 된 레거시 코드 위에 최적화의 탑을 쌓아 올린 괴물들입니다. 빠르고 정확하지만, 코드를 수정하거나 최신 ML 파이프라인에 통합하는 건 고문에 가깝습니다.

## slaterform: JAX가 가져온 마법

`slaterform`의 핵심은 간단합니다. Hartree-Fock 알고리즘 전체를 **Pure JAX** 로 구현했다는 겁니다. 이것이 의미하는 바는 엄청납니다.

### 1. Auto-Differentiation을 통한 Geometry Optimization

분자의 구조를 최적화하려면(예: 에너지가 가장 낮은 안정된 형태 찾기), 에너지 함수를 원자 핵의 위치에 대해 미분해야 합니다. 이를 통해 원자들에 작용하는 힘(Force)을 구하고, 그 방향으로 원자를 조금씩 움직입니다.

기존 방식에서는 **Hellmann-Feynman 정리** 를 이용해 해석적인 그래디언트(Analytical Gradient)를 직접 구현해야 했습니다. 수식이 복잡하고 구현 실수가 잦은 부분이죠.

하지만 `slaterform`에서는 JAX의 `grad`를 쓰면 끝납니다. 아래 코드를 보시죠.

```python
import jax
import slaterform as sf
import slaterform.hartree_fock.scf as scf

def total_energy(molecule: sf.Molecule):
    """Hartree-Fock으로 분자의 전체 에너지를 계산"""
    options = scf.Options(
        max_iterations=20,
        execution_mode=scf.ExecutionMode.FIXED,
        integral_strategy=scf.IntegralStrategy.CACHED,
        perturbation=1e-10,
    )
    result = scf.solve(molecule, options)
    return result.total_energy

# 마법이 일어나는 곳: 에너지 함수를 미분하여 힘(Gradient)을 자동 계산
total_energy_and_grad = jax.jit(jax.value_and_grad(total_energy))
```

이 몇 줄의 코드로 우리는 분자 시뮬레이터를 만들 수 있습니다. 실제로 저자가 공개한 [Colab 노트북](https://colab.research.google.com/github/lowdanie/hartree-fock-solver/blob/main/notebooks/geometry_optimization.ipynb)을 보면, 평면에 납작하게 놓인 메탄(CH4) 분자가 최적화 과정을 거쳐 우리가 아는 정사면체(Tetrahedral) 구조로 '팝'하고 튀어 오르는 것을 볼 수 있습니다. 시각적으로 정말 쾌감 쩌는 장면입니다.

### 2. Native Integral Implementation

보통 이런 라이브러리들이 나오면 "적분 계산(Electron Integrals)은 `libcint` 같은 C 라이브러리 바인딩해서 썼겠지"라고 생각하기 쉽습니다. 하지만 이 프로젝트는 **전자 적분까지 JAX로 네이티브 구현** 했습니다. 즉, 적분 과정 자체도 미분 가능하다는 뜻입니다. 이는 추후에 기저 함수(Basis Set) 자체를 최적화하거나, 더 복잡한 파라미터를 학습시키는 데 엄청난 유연성을 제공합니다.

## Hacker News의 반응과 기술적 통찰

Hacker News의 반응도 꽤 뜨거웠습니다. 특히 인상 깊었던 몇 가지 코멘트를 제 경험과 섞어 해석해 보겠습니다.

### "Hellmann-Feynman vs JAX Magic"

한 유저(anon)는 저자가 Hellmann-Feynman 정리를 쓰는 대신 JAX의 자동 미분을 사용해 원자 힘(Atomic Forces)을 구하는 방식에 주목했습니다. 또한, **"핵 전하(Nuclear Charge)에 대해 미분하면 연금술(Alchemy)도 가능하겠네?"** 라는 농담 반 진담 반의 코멘트를 남겼는데요.

이게 기술적으로 꽤 재미있는 포인트입니다. 이론상으로는 원자핵의 전하량을 연속적인 변수(float)로 취급하여 미분할 수 있습니다. 즉, 탄소(C)에서 질소(N)로 가는 경로를 미분 가능한 공간에서 탐색할 수 있다는 뜻이죠. 물론 물리적으로는 말이 안 되지만, 신소재 탐색을 위한 Latent Space에서의 최적화 기법으로는 충분히 연구 가치가 있습니다. 이것이 바로 "Differentiable Physics"가 주는 새로운 시각입니다.

### "Fortran의 악몽에서 벗어나다"

다른 유저는 첫 직장에서 **Molpro** 를 썼던 기억을 떠올리며, "오래되고 방대하지만, 그 끔찍한 Fortran 코드는 정말..."이라며 치을 떨더군요. 저도 공감합니다. 과학 계산 분야는 성능을 위해 가독성과 유지보수성을 희생한 역사가 깁니다. `slaterform`처럼 Python/JAX 생태계 위에서 돌아가는 코드는 성능이 조금 떨어지더라도, 연구 속도(Iteration Speed)와 확장성 면에서 압도적인 우위를 가집니다. 최신 GPU/TPU 가속을 공짜로 얻는 건 덤이고요.

## 한계점과 나의 평결 (Verdict)

물론 칭찬만 할 수는 없습니다. 현업 엔지니어 관점에서 냉정하게 보자면:

1.  **성능:** 아직 `libint`나 `libcint`를 사용하는 C++ 기반 솔버들에 비해 속도 면에서 경쟁력이 있을지는 의문입니다. JAX가 빠르긴 하지만, 양자화학의 적분 계산은 워낙 특수한 최적화가 많이 들어가는 분야라 단순 GPU 병렬화만으로는 한계가 있습니다.
2.  **기능:** Hartree-Fock은 가장 기초적인 방법론입니다. 실제 화학적 정확도를 얻으려면 DFT(Density Functional Theory)나 Post-HF 방법론(CCSD(T) 등)이 필요한데, 여기까지 순수 JAX로 구현하려면 갈 길이 멉니다.

**하지만,** 이 프로젝트는 **"AI for Science"** 가 나아가야 할 방향을 명확히 보여줍니다. 기존의 블랙박스 시뮬레이터를 단순히 ML 모델로 근사(Approximation)하는 것을 넘어, **시뮬레이터 자체를 미분 가능한 구조로 재작성** 하여 ML 파이프라인의 일부로 편입시키는 흐름입니다.

만약 여러분이 딥러닝을 이용한 신약 개발이나 소재 탐색에 관심이 있다면, 이 코드를 뜯어보는 것을 강력히 추천합니다. 단순히 API를 호출하는 것을 넘어, 물리 엔진 내부가 어떻게 그래디언트를 흘려보내는지 이해하는 데 이보다 좋은 교보재는 없을 겁니다.

**결론:** Production 레벨의 양자화학 계산용은 아니지만, **Deep Learning + Physics** 연구자들에게는 보물 같은 코드베이스.
