---
title: "Java로 GPU를 갈궈보자: Babylon과 HAT이 보여주는 새로운 가능성"
description: "Java로 GPU를 갈궈보자: Babylon과 HAT이 보여주는 새로운 가능성"
pubDate: "2026-01-26T00:47:22Z"
---

## "Java로 AI 개발? 그게 되나요?"

솔직히 말해보자. 지난 10년 동안 "Java로 GPU 프로그래밍 하기"라는 주제는 일종의 **성배(Holy Grail)** 이자, 동시에 **희망 고문** 이었다. Sumatra 프로젝트부터 Aparapi, RootBeer까지... 시도는 많았지만, 결국 우리는 Python과 C++(CUDA)의 조합 앞에 무릎을 꿇었다. Java 개발자가 GPU를 쓰려면 JNI로 C++ 라이브러리를 호출하는 '껍데기'를 만드는 게 최선이었던 게 사실이다.

그런데 최근 OpenJDK 진영에서 꽤 흥미로운 움직임이 포착됐다. 바로 **Project Babylon** 과 **HAT(Heterogeneous Accelerator Toolkit)** 이다. 단순히 "Java에서 CUDA를 부른다" 수준이 아니다. Java 코드를 런타임에 분석해서(Code Reflection), CUDA나 OpenCL 커널로 트랜스파일링하고, 데이터 이동까지 처리하겠다는 야심 찬 프로젝트다.

오늘은 OpenJDK 팀이 공개한 **[Matrix Multiplication 최적화 사례](https://openjdk.org/projects/babylon/articles/hat-matmul/hat-matmul)** 를 씹고 뜯고 맛보고 즐겨보려 한다. 과연 이번엔 진짜일까?

---

### HAT: Java와 GPU 사이의 미싱 링크

HAT은 기본적으로 **Project Babylon(Code Reflection)** 과 **Project Panama(Foreign Function & Memory API)** 위에서 돌아간다. 이 기술 스택의 조합이 주는 의미는 크다.

1.  **Code Reflection:** Java 메서드의 바이트코드가 아니라, 그 코드의 **구조(Code Model)** 를 읽어낸다. 즉, `for` 루프가 어떻게 생겼는지, 어떤 연산을 하는지 심볼릭하게 파악해서 GPU가 이해할 수 있는 PTX나 SPIR-V로 변환한다.
2.  **Panama:** JVM 힙 메모리와 GPU 메모리 사이의 데이터 이동을 효율적으로 처리한다.

이론상으로는 Java 문법만으로 CUDA C++ 수준의 최적화를 달성할 수 있다는 건데, 실제 코드를 보자.

### 1. 맨땅의 헤딩: CPU vs GPU 1D Kernel

우선 베이스라인이다. Java Parallel Stream으로 짠 행렬 곱셈(Matrix Multiplication)은 최신 인텔 CPU(15코어)에서 약 **7 GFLOP/s** 가 나온다. 나쁘진 않지만, AI 워크로드엔 턱도 없다.

이걸 HAT을 이용해 가장 단순한 1D 커널로 GPU(NVIDIA A10)에 올리면 어떻게 될까?

```java
@Reflect
public static void matrixMultiplyKernel1D(
    @RO KernelContext kc,
    @RO F32Array matrixA, 
    @RO F32Array matrixB, 
    @RW F32Array matrixC, int size) {
    if (kc.gix < size) {
        // ... 행렬 곱셈 로직 ...
    }
}
```

결과는 **95ms**. CPU보다 3배 빠르다. 하지만 `ncu`(Nsight Compute) 프로파일러를 돌려보면 처참하다. GPU Compute Throughput이 겨우 **4.4%** 다. A10 같은 괴물 GPU를 데려와서 4%만 쓰는 건 자원 낭비다.

### 2. 2D 커널과 메모리 최적화의 시작

GPU는 수천 개의 스레드가 동시에 돈다. 1D로 펴서 돌리는 것보다 2D 그리드로 매핑하는 게 인지적으로나 성능적으로나 유리하다. HAT에서는 `kc.gix`, `kc.giy` 같은 직관적인 API로 2D 인덱스에 접근한다.

단순히 2D로 바꿨더니 **9.05ms** 로 10배 빨라졌다. 하지만 진짜 핵심은 그다음이다. 바로 **Memory Coalescing(메모리 병합 액세스)** 이다.

Java 개발자들은 보통 메모리 레이아웃을 깊게 고민하지 않는다. 하지만 GPU에서는 **"인접한 스레드가 인접한 메모리를 읽느냐"** 가 성능을 좌우한다. 원본 코드에서는 `matrixA`를 읽을 때 스레드들이 띄엄띄엄 메모리를 읽는(strided access) 문제가 있었다.

이걸 해결하기 위해 `gix`와 `giy`의 루프 순서를 바꿔서, 워프(Warp) 내의 스레드들이 연속된 메모리 주소를 긁어오도록 수정했다.

```java
// Coalescing을 위한 인덱스 스왑
if (kc.giy < kc.gsy) {
    if (kc.gix < kc.gsx) {
        // ...
        acc += (matrixA.array(kc.giy * size + k) * ...);
    }
}
```

결과는 **2.19ms**. 또다시 4배가 빨라졌다. 이제 메모리 대역폭의 96%를 쓴다. 여기서부터는 "Java라서 느리다"는 핑계가 안 통하는 영역이다.

### 3. 끝판왕: Shared Memory와 Register Tiling

여기서부터가 진짜다. C++ CUDA 프로그래머들이 성능을 쥐어짜 낼 때 쓰는 기법인 **Tiling** 을 Java로 구현한다. GPU의 Global Memory는 느리다. 그래서 자주 쓰는 데이터를 L1 캐시 역할인 **Shared Memory** 로 가져오고, 더 나아가 **Register** 에 박아두고 연산해야 한다.

HAT은 `SharedMemory`라는 인터페이스를 통해 이를 지원한다. Java 코드 안에서 명시적으로 "이건 Shared Memory에 할당해"라고 선언하는 것이다.

```java
private interface SharedMemory extends DeviceType {
    // ... schema definition ...
}

// 커널 내부
SharedMemory tileA = SharedMemory.createLocal();
// Global -> Shared -> Register 로 데이터 이동 및 연산
```

이 과정은 솔직히 Java 코드치고는 매우 장황(verbose)하고 복잡하다. 하지만 결과는 **0.37ms**. 처음 1D 커널 대비 **250배**, CPU 대비로는 수천 배의 성능 향상이다. 최종적으로 **14 TFLOP/s** 를 찍었다. 이는 Native cuBLAS 성능에 거의 근접한 수치다.

### Hacker News의 반응과 나의 생각

이 글에 대한 [Hacker News의 반응](https://news.ycombinator.com/item?id=46689895)은 꽤 냉소적이면서도 기대가 섞여 있다. 한 유저는 이렇게 말했다.

> "Java GPU 프로그래밍 시도는 많았지만(Aparapi 등), 입양(Adoption)되는 꼴을 못 봤다. TornadoVM 같은 멋진 프로젝트도 있지만, 메이저 프로젝트에서 쓰이기 전까진 투자하기 망설여진다."

매우 공감 가는 말이다. 기술적으로 가능하다는 것과 생태계가 만들어진다는 건 천지 차이다. Python이 AI를 지배한 건 언어가 빨라서가 아니라, `import torch` 한 줄이면 끝나기 때문이다.

하지만 또 다른 유저의 코멘트처럼 **"이것이 JVM 위의 Native Torch로 이어질 수도 있다"** 는 희망도 있다. HAT이 로우레벨 빌딩 블록이 되어주고, 그 위에 사용하기 쉬운 라이브러리가 얹어진다면? Java의 강력한 서버 사이드 생태계와 결합했을 때 파괴력은 무시할 수 없다.

### 결론: 아직은 '원석', 하지만 빛난다

Babylon과 HAT을 보며 느낀 점을 정리하자면:

1.  **기술적 완성도는 놀랍다:** Java의 어노테이션과 리플렉션만으로 CUDA 커널을 생성하고, 메모리 계층(Shared/Register)까지 제어할 수 있다는 건 대단한 진보다.
2.  **생산성은 아직 글쎄:** 위에서 본 Tiling 최적화 코드를 직접 짜고 싶은 Java 개발자는 많지 않을 것이다. 이건 라이브러리 개발자들을 위한 도구다.
3.  **가능성:** 만약 Spring Boot 애플리케이션에서 어노테이션 하나로 비즈니스 로직의 일부를 GPU로 오프로딩할 수 있는 날이 온다면, 그건 게임 체인저가 될 것이다.

아직 프로덕션에 투입할 단계는 아니다. 하지만 15년 차 엔지니어의 감으로 볼 때, 이번 시도는 과거의 '실패한 시도들'과는 결이 다르다. JDK 차원에서 밀고 있는 **Code Reflection** 이라는 강력한 무기가 있기 때문이다. Java 개발자라면 이 프로젝트, 계속 주시해야 한다.
