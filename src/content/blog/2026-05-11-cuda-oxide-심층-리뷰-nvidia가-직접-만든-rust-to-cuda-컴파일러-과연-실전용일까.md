---
title: "Cuda-oxide 심층 리뷰: NVIDIA가 직접 만든 Rust to CUDA 컴파일러, 과연 실전용일까?"
description: "Cuda-oxide 심층 리뷰: NVIDIA가 직접 만든 Rust to CUDA 컴파일러, 과연 실전용일까?"
pubDate: "2026-05-11T18:56:36Z"
---

GPU 커널을 작성하는 일은 오랫동안 C++의 전유물이자, 동시에 수많은 엔지니어들에게 고통의 연속이었다. 15년 넘게 현업에서 백엔드와 인프라를 굴리면서 수많은 CUDA 코드를 디버깅해봤지만, 포인터 에일리어싱(Pointer aliasing)이나 메모리 누수 같은 Undefined Behavior(UB)는 언제나 골칫거리였다.

최근 몇 년간 Rust 생태계에서 cudarc 같은 훌륭한 크레이트들이 등장하며 숨통이 트이나 싶었지만, 결국 그 밑바탕에는 무겁고 느린 nvcc 컴파일러와 FFI(Foreign Function Interface)라는 구조적 한계가 존재했다. 그런데 이번에 NVIDIA가 공식적으로  **cuda-oxide**  라는 프로젝트를 공개했다. 결론부터 말하자면 이건 단순한 라이브러리나 래퍼(Wrapper)가 아니다. Rust를 PTX로 직접 컴파일하는 진짜 컴파일러다.



## 어떻게 작동하는가? (Not a DSL, Pure Rust)

cuda-oxide의 가장 큰 특징은 별도의 DSL이나 C++ 바인딩 없이, 표준 Rust 코드를 그대로 PTX로 컴파일한다는 점이다. 내부적으로는 rustc-codegen-cuda를 사용해 Rust -> MIR -> pliron IR -> LLVM IR -> PTX의 파이프라인을 거친다.

공개된 Quick Start 코드를 한 번 살펴보자.

```rust
use cuda_device::{cuda_module, kernel, thread, DisjointSlice};
use cuda_core::{CudaContext, DeviceBuffer, LaunchConfig};

#[cuda_module]
mod kernels {
    use super::*;

    #[kernel]
    fn vecadd(a: &[f32], b: &[f32], mut c: DisjointSlice<f32>) {
        let idx = thread::index_1d();
        let i = idx.get();
        if let Some(c_elem) = c.get_mut(idx) {
            *c_elem = a[i] + b[i];
        }
    }
}
// ... (main 생략)
```

여기서 눈여겨봐야 할 부분은  **DisjointSlice**  이다. SIMT(Single Instruction, Multiple Threads) 환경인 GPU에서 Rust의 소유권(Ownership)과 대여(Borrowing) 규칙을 완벽하게 적용하는 것은 불가능에 가깝다. C++에서는 여러 스레드가 같은 메모리 주소에 동시에 쓰는 것을 막아줄 안전장치가 전혀 없지만, cuda-oxide는 DisjointSlice와 ThreadIndex를 강제함으로써 컴파일 타임 혹은 안전한 런타임 추상화 레벨에서 Mutable Aliasing을 방지한다.

## 실무자의 시선: 이게 왜 중요한가?

솔직히 처음 이 소식을 접했을 때 내 반응은 "또 다른 장난감 아닐까?" 였다. 하지만 아키텍처를 뜯어볼수록 NVIDIA가 꽤 진심이라는 걸 알 수 있었다.

- **빌드 타임:** 기존 Rust에서 CUDA를 쓸 때는 build.rs에서 CMake나 nvcc를 호출해야 했다. 이는 CI 파이프라인을 무겁게 만들고 로컬 개발 경험을 최악으로 만든다. cuda-oxide는 호스트 바이너리에 생성된 디바이스 아티팩트를 바로 임베딩한다. nvcc를 거치지 않는다는 것만으로도 빌드 속도와 캐싱 효율에서 엄청난 이득이다.
- **메모리 안전성:** C++에서는 cudaFree를 잊어버리거나 UAF(Use-After-Free)가 발생하는 일이 흔하다. Rust의 Drop 시맨틱을 GPU 메모리에 그대로 적용할 수 있다는 건, 인프라 레벨의 안정성을 크게 끌어올릴 수 있음을 의미한다.

하지만 우려되는 점도 분명히 있다. Rust 특유의 바운드 체크(Bounds checking)가 GPU 커널의 레지스터 사용량을 늘리지 않을까? GPU 프로그래밍에서는 레지스터 스필(Register spill)이 조금만 발생해도 Occupancy가 급락하고 성능이 처참해진다. 이 오버헤드를 컴파일러 백엔드가 얼마나 똑똑하게 최적화해 줄지가 관건이다.

## Hacker News 커뮤니티의 반응

이번 발표에 대해 HN에서도 심도 있는 토론이 오갔다. 몇 가지 흥미로운 포인트들을 짚어본다.

- **cudarc와의 비교:** 많은 엔지니어들이 이것이 cudarc를 완전히 대체할 수 있는지에 대해 논의했다. 결론적으로 워크플로우는 비슷해 보일지 몰라도 근본이 다르다. 기존의 C++ CUDA 커널을 단순히 Rust에서 호출만 하고 싶다면 여전히 cudarc가 가볍고 좋은 선택이다. 하지만 커널 자체를 Rust로 작성하고 싶다면 cuda-oxide가 정답이 될 것이다.
- **Autodiff 지원 여부:** 머신러닝 엔지니어들은 슬랭(Slang)이나 줄리아(Julia)처럼 AD(Automatic Differentiation)가 1급 시민으로 지원되지 않는 점을 아쉬워했다. 흥미롭게도 현재 Rust 진영에서는 std::autodiff가 pre-RFC 단계로 논의되고 있다. 만약 이것이 cuda-oxide와 결합된다면, ML 생태계에 엄청난 파장을 일으킬 것이다.
- **오픈소스 논쟁:** 여전히 밑단에는 NVIDIA의 폐쇄적인 생태계가 존재한다는 비판도 있다. Mojo처럼 여러 가속기를 지원하는 오픈 컴파일러를 지향하는 대신, 철저히 NVIDIA 하드웨어에 종속된다는 점은 아킬레스건이다. 하지만 실용주의적인 관점에서 당장의 프로덕션 문제를 해결해 주는 툴임에는 틀림없다.

## 결론 (Verdict)

cuda-oxide는 아직 v0.1.0 알파 버전이다. API는 계속 바뀔 것이고 버그도 많을 것이다. 당장 내일 회사 프로덕션의 PyTorch나 C++ CUDA 코드를 걷어내고 이걸 도입하라고 권하지는 않겠다.

하지만, 시스템 프로그래밍의 미래가 Rust로 기울고 있다는 점을 NVIDIA가 공식적으로 인정하고 투자하기 시작했다는 사실 자체가 매우 상징적이다. 하드웨어의 한계를 쥐어짜야 하는 커스텀 커널 개발자라면, 주말 토이 프로젝트로 당장 cargo oxide run을 돌려볼 가치가 충분하다.

References:
- Original Article: https://nvlabs.github.io/cuda-oxide/index.html
- Hacker News Thread: https://news.ycombinator.com/item?id=48096692
