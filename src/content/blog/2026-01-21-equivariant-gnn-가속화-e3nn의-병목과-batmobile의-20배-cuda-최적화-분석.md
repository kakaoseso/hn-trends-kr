---
title: "Equivariant GNN 가속화: e3nn의 병목과 Batmobile의 20배 CUDA 최적화 분석"
description: "Equivariant GNN 가속화: e3nn의 병목과 Batmobile의 20배 CUDA 최적화 분석"
pubDate: "2026-01-21T13:22:34Z"
---

최근 AI 분야, 특히 신약 개발이나 재료 과학(Materials Science) 쪽에서는 **Equivariant Graph Neural Networks (EGNN)**가 사실상의 표준(Standard)으로 자리 잡고 있습니다. MACE, NequIP, Allegro 같은 모델들이 분자 동역학 시뮬레이션에서 보여주는 정확도는 정말 놀랍습니다.

하지만 현업에서 이 모델들을 서빙하거나 대규모 학습을 돌려보신 분들은 공감하실 겁니다. **"아름다운 수학인데, 너무 느리다."**

오늘은 이 문제를 해결하기 위해 등장한 **Batmobile**이라는 프로젝트를 깊게 파보려 합니다. 기존의 표준 라이브러리인 e3nn 대비 10~20배의 속도 향상을 이뤄낸 커스텀 CUDA 커널 구현체입니다. 단순히 "빨라졌다"는 결과보다, **왜 기존 방식이 느렸고 어떻게 하드웨어 레벨에서 최적화했는지** 그 엔지니어링 과정이 매우 흥미롭습니다.

---

## 왜 Equivariant GNN은 느린가?

먼저 문제의 본질을 이해해야 합니다. Equivariant GNN의 핵심은 물리적 대칭성(회전, 이동, 반사 불변성)을 지키는 것입니다. 이를 위해 두 가지 무거운 연산을 수행합니다.

1.  **Spherical Harmonics ($Y_{lm}$):** 3D 공간의 방향성을 인코딩합니다.
2.  **Tensor Products:** Clebsch-Gordan 계수를 사용해 피처들을 결합합니다.

MACE 같은 모델은 Forward Pass 시간의 약 **80%**를 이 두 연산에 쏟아붓습니다. 분자 시뮬레이션은 수십억 타임스텝을 돌려야 하는데, 한 스텝이 마이크로초(µs)가 아니라 밀리초(ms) 단위라면 실무에서는 사용이 불가능합니다.

### e3nn의 구조적 한계

현재 학계와 산업계에서 가장 많이 쓰이는 라이브러리는 `e3nn`입니다. 추상화가 잘 되어 있고 사용하기 편하지만, 성능 면에서는 명확한 한계가 있습니다.

*   **Python/PyTorch 오버헤드:** Spherical Harmonics의 각 컴포넌트를 별도의 PyTorch 연산으로 처리합니다. $L_{max}=3$인 경우 16개의 컴포넌트가 나오는데, 이걸 계산하기 위해 커널 런칭이 16번 일어난다는 뜻입니다.
*   **Memory Bandwidth 낭비:** 중간 계산 결과(Intermediate results)를 계속 GPU의 Global Memory에 썼다가 다시 읽어옵니다. 레지스터에서 끝낼 수 있는 일을 VRAM까지 갔다 오는 비용을 치르는 셈입니다.
*   **Fusion의 부재:** Spherical Harmonics 계산과 Tensor Product가 분리되어 있어 데이터 로컬리티를 활용하지 못합니다.

---

## Batmobile: 20배 빨라진 비결

Batmobile은 범용성을 포기하고 **속도에 올인**한 프로젝트입니다. 핵심 전략은 세 가지로 요약됩니다.

![Batmobile benchmark results showing 10-20x speedup over e3nn](/blog/batmobile/benchmark_hero.png)

### 1. Compile-Time Constants (컴파일 타임 상수화)

`e3nn`은 런타임에 다양한 $L$ 값(차수)을 처리할 수 있도록 유연하게 설계되었습니다. 반면 Batmobile은 $L_{max}=3$과 같이 특정 차수에 대해 루프와 계수를 하드코딩해버립니다.

Clebsch-Gordan 계수는 수학적으로 고정된 상수입니다. 이를 메모리에서 읽어오는 대신, CUDA 커널 코드 안에 박아버리면(bake in) 레지스터 조차 아낄 수 있고 Instruction Cache 효율이 극대화됩니다.

```cpp
// 34개의 모든 CG 경로를 명시적으로 Unroll (Loop Unrolling)
// Path (0,0)->0: trivial identity
out[0] += cg_0_0_0 * in1[0] * in2[0];
// Path (1,1)->0: scalar from two vectors
out[0] += cg_1_1_0_m1m1 * in1[1] * in2[1];
// ... 이런 식으로 34개 경로를 전부 펼침
```

### 2. Register-Only Intermediates

이 부분이 성능 향상의 핵심입니다. Spherical Harmonics 계산 결과를 Global Memory에 쓰지 않고, **모두 레지스터(Register) 안에서만 처리**합니다.

```cpp
__device__ __forceinline__ void compute_sh_registers(
    float x, float y, float z,
    float* __restrict__ sh // sh[16]이 레지스터에 매핑됨
) {
    // L=0
    sh[0] = 1.0f;
    // L=1: sqrt(3) * (x, y, z)
    constexpr float c1 = 1.7320508075688772f;
    sh[1] = c1 * x;
    sh[2] = c1 * y;
    sh[3] = c1 * z;
    // L=2, L=3 도 마찬가지로 레지스터 연산으로 처리
}
```

GPU 프로그래밍에서 Global Memory 접근은 가장 비싼 비용 중 하나입니다. 이를 제거함으로써 엄청난 Throughput 향상을 얻을 수 있습니다.

### 3. Kernel Fusion (커널 퓨전)

Spherical Harmonics 계산과 Tensor Product를 하나의 커널로 합쳤습니다. 입력 벡터가 들어오면, 중간 결과를 메모리에 저장할 필요 없이 바로 다음 연산의 입력으로 사용되어 최종 피처만 뱉어냅니다.

---

## 벤치마크: 숫자가 말해주는 것

RTX 3090에서의 벤치마크 결과는 압도적입니다.

| Operation | e3nn | Batmobile | Speedup |
|---|---|---|---|
| Spherical Harmonics ($L=3$) | 0.142 ms | **0.012 ms** | **11.8x** |
| Tensor Product | 1.847 ms | **0.089 ms** | **20.8x** |
| TP Backward | 3.21 ms | **0.156 ms** | **20.6x** |

특히 **Tensor Product에서 20배 이상의 속도 향상**이 있었다는 점이 고무적입니다. 이는 MACE 같은 모델의 학습 및 추론 속도를 획기적으로 단축시킬 수 있다는 뜻입니다.

![Detailed benchmark comparison between e3nn and Batmobile](/blog/batmobile/benchmark_chart.png)

---

## Community Insight: 학계 코드 vs 벤더 라이브러리

이 프로젝트에 대해 Hacker News에서 매우 흥미로운 논쟁이 있었습니다. 한 유저(anon)가 다음과 같은 댓글을 남겼는데요:

> "e3nn은 학술용 소프트웨어라 다소 하이레벨로 설계되었습니다. 비교 대상으로는 NVIDIA의 `cuEquivariance`가 더 적절할 것입니다. HPC 개발자로서, 학계 소프트웨어의 성능이 벤더 라이브러리(Intel, Nvidia)에 비해 얼마나 떨어지는지 볼 때마다 가슴이 아픕니다. 우리는 목표를 더 높게 잡아야 합니다."

이 코멘트는 뼈아프지만 정확한 지적입니다. `e3nn`은 연구자들이 수식을 코드로 옮기기 쉽게 만든 **'탐색용 도구'**에 가깝습니다. 반면 Batmobile이나 NVIDIA의 `cuEquivariance`는 **'프로덕션용 엔진'**입니다.

하지만 저는 Batmobile 같은 시도가 매우 중요하다고 봅니다. NVIDIA가 모든 로직을 최적화해주길 기다릴 수는 없습니다. 연구자가 만든 알고리즘을 엔지니어가 하드웨어 레벨까지 뜯어고쳐 최적화하는 이런 사이클이야말로 기술 발전의 정석이기 때문입니다.

## Principal Engineer's Verdict

Batmobile은 이름값을 합니다. 범용성을 포기하고 특정 워크로드($L_{max}=3$, 고정된 Path)에 집중함으로써 극한의 성능을 뽑아냈습니다. 이것이 바로 **Domain Specific Optimization**의 정수입니다.

**제언:**
1.  **MACE/NequIP 사용자라면:** 당장 도입을 고려해보세요. 추론 속도 20배 향상은 비즈니스 임팩트가 다릅니다.
2.  **커스텀 모델 연구자라면:** `e3nn`으로 프로토타이핑을 하되, 배포 단계에서는 Batmobile이나 `cuEquivariance`로 포팅하는 파이프라인을 구축해야 합니다.
3.  **엔지니어라면:** 파이썬 for loop를 줄이는 것을 넘어, GPU 레지스터와 메모리 계층 구조를 이해하는 것이 얼마나 큰 차이를 만드는지 보여주는 훌륭한 케이스 스터디로 삼으시기 바랍니다.

**References:**
*   Original Article: [Batmobile: 10-20x Faster CUDA Kernels](https://elliotarledge.com/blog/batmobile)
*   Hacker News Discussion: [Hacker News Thread](https://news.ycombinator.com/item?id=46663931)
