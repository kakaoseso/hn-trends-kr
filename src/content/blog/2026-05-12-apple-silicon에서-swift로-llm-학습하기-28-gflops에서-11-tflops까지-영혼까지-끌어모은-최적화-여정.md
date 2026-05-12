---
title: "Apple Silicon에서 Swift로 LLM 학습하기: 2.8 Gflop/s에서 1.1 Tflop/s까지 영혼까지 끌어모은 최적화 여정"
description: "Apple Silicon에서 Swift로 LLM 학습하기: 2.8 Gflop/s에서 1.1 Tflop/s까지 영혼까지 끌어모은 최적화 여정"
pubDate: "2026-05-12T04:40:58Z"
---

안드레 카파시(Andrej Karpathy)가 1000줄 남짓한 순수 C 코드로 GPT-2를 학습시키는 llm.c를 공개했을 때, 많은 엔지니어들이 환호했다. 파이썬과 거대한 프레임워크 뒤에 숨겨져 있던 블랙박스가 투명하게 걷히는 느낌이었기 때문이다.

최근 초기 iOS 개발 생태계의 전설적인 블로그인 CocoaWithLove의 Matt Gallagher가 이 llm.c를 프레임워크 없이 순수 Swift로 포팅하며 작성한 글을 읽었다. 단순히 포팅에 그치지 않고, 2.8 Gflop/s에 불과했던 처참한 초기 성능을 1.1 Tflop/s까지 382배 끌어올리는 집요한 최적화 과정을 담고 있다.

15년 이상 개발을 해오면서 수많은 성능 최적화 사례를 봤지만, 이 정도로 언어의 로우레벨 특성과 하드웨어 아키텍처를 밑바닥부터 파고든 글은 오랜만이다. 오늘은 이 흥미로운 여정을 따라가며, 시니어 엔지니어 관점에서 주목할 만한 기술적 포인트와 해커뉴스(Hacker News) 커뮤니티의 날카로운 인사이트를 더해 심층적으로 분석해 보려 한다.

## 1. 안전함이 부른 참사: COW (Copy-On-Write) 오버헤드

최초로 작성된 순수 Swift 버전의 행렬 곱셈 코드는 C 버전보다 무려 15~20배나 느렸다. 초당 2.8 Gflop/s라는 수치는 1999년 PowerMac G4 시절에나 자랑할 만한 속도다.

프로파일러를 돌려본 결과, 원인은 _ArrayBuffer.beginCOWMutation()이었다. Swift는 배열의 복사를 방지하고 고유성을 보장하기 위해 끊임없이 참조 카운트를 확인하는데, 이 고유성 검사 자체가 엄청난 병목을 일으킨 것이다.

솔직히 이 부분은 Swift 컴파일러의 한계 혹은 옵티마이저의 버그라고 본다. 루프 내에서 배열이 수정되지 않음에도 불구하고 불필요한 검사가 반복되는 현상은 고성능 연산에서 치명적이다. 다행히 Swift 6.2에서 도입된 MutableSpan을 사용해 배열의 메모리 영역을 직접 참조함으로써 이 오버헤드를 완전히 제거할 수 있었다. 이 단순한 변경만으로 전체 학습 속도가 3배 이상 빨라졌다.

## 2. FMA와 잃어버린 컴파일러 플래그

메모리 병목을 해결한 뒤 마주한 다음 장벽은 연산 그 자체였다. C 코드는 내부 루프에서 8번 언롤링된 fmadd (Fused-Multiply-Add) 어셈블리 명령어를 사용하고 있었다. 이는 곱셈과 덧셈을 단일 명령어로 처리하는 강력한 하드웨어 가속 기능이다.

C 컴파일러가 이를 사용할 수 있었던 이유는 -ffast-math 플래그 덕분이다. 반면 Swift에는 이 플래그가 없어서 별도의 fmul과 fadd 명령어로 쪼개져 실행되고 있었다. 이를 해결하기 위해 Apple의 Swift-Numerics 라이브러리에서 제공하는 Relaxed.multiplyAdd를 사용하여 강제로 FMA를 유도했다.

여기서 해커뉴스 스레드의 한 유저가 굉장히 예리한 지적을 남겼다.

> "ML/AI 분야가 아니면 -ffast-math는 절대 함부로 쓰면 안 됩니다. FMA만 원한다면 -ffp-contract=fast를 쓰는 것이 맞습니다. -ffast-math는 수치적 정확도가 중요한 애플리케이션에서 원치 않는 코드 변환을 일으킬 수 있습니다."

개인적으로 이 의견에 100% 동의한다. 과거 금융권이나 정밀한 물리 엔진 코드를 다룰 때 컴파일러 플래그가 만들어낸 미세한 부동소수점 오차 때문에 며칠 밤을 샌 기억이 있다. 맹목적인 플래그 사용은 기술 부채로 돌아온다.

## 3. 스택 할당과 InlineArray

C 언어 최적화의 핵심 중 하나는 내부 루프에서 float result[8]과 같이 작은 버퍼를 스택에 할당하여 레지스터 활용도를 높이는 것이다. 기존 Swift에서는 힙 할당 없이 이런 가변 길이의 로컬 배열을 만드는 것이 사실상 불가능했다.

하지만 Swift 6.2의 InlineArray 덕분에 C의 스택 할당 배열과 완벽히 동일한 구조를 구현할 수 있게 되었다. 언어의 발전이 로우레벨 제어력을 어떻게 회복시켜 주는지 보여주는 훌륭한 사례다.

## 4. 멀티스레딩과 Swift Concurrency의 민낯

C 버전은 #pragma omp parallel for 한 줄로 우아하게 OpenMP 멀티스레딩을 구현했다. 반면 Swift에서 이를 구현하는 과정은 상당히 고통스럽다.

```swift
out.withUnsafeMutableBufferPointer { outBuffer in
    let outStorage = SendableUnsafeMutableBuffer(baseAddress: outBuffer.baseAddress!)
    DispatchQueue.concurrentPerform(iterations: chunkCount) { chunk in
        // 병렬 타일 연산 수행...
    }
}
```

배열을 슬라이싱하여 병렬로 접근하려면 DispatchQueue.concurrentPerform을 사용해야 하는데, 위 코드처럼 withUnsafeMutableBufferPointer를 열고 @unchecked Sendable 래퍼를 씌워 컴파일러의 눈을 가려야만 했다.

앱 개발 관점에서는 Swift의 엄격한 데이터 레이스 방지가 훌륭하지만, 고성능 컴퓨팅(HPC) 영역에서는 이처럼 코드를 극도로 지저분하게 만든다. Rust의 split_at_mut 같은 우아한 병렬 슬라이싱 API가 Swift에도 절실히 필요하다.

## 5. 숨겨진 AMX 명령어와 GPU의 벽

CPU 최적화의 끝판왕으로 Matt은 Apple Silicon의 비공개 행렬 보조 프로세서인 AMX (Apple Matrix Coprocessor) 명령어를 리버스 엔지니어링하여 사용했다. 타일 기반의 행렬 연산을 통해 성능을 비약적으로 끌어올렸지만, 이는 언제든 깨질 수 있는 위험한 접근이다. (해커뉴스 댓글에 따르면 M4 칩부터는 AMX 대신 SME 아키텍처가 도입되었다고 한다.)

결국 진정한 성능을 위해 Metal을 이용한 GPU 컴퓨트 셰이더로 넘어간다. 단순한 1D 스레드 분배에서 시작해, 타일 메모리를 활용하는 복잡한 커널(Tiled Kernel)을 작성하여 마침내 1.1 Tflop/s의 벽을 돌파한다.



이 대목에서 왜 엔비디아(Nvidia)의 CUDA 생태계가 강력한 해자(Moat)를 가지는지 다시 한번 깨닫게 된다. 해커뉴스 유저의 말처럼, GPU에서 이론적 최대 성능(M3 Max 기준 약 15 Tflop/s)을 뽑아내는 것은 CPU 최적화와는 차원이 다른 문제다. 하드웨어 아키텍처에 완벽히 맞춰진 수천 개의 튜닝된 커널을 기본 제공하는 CUDA를 순수 커스텀 셰이더로 따라잡는 것은 사실상 불가능에 가깝다.

## 총평: 그래서 프로덕션에 쓸 수 있는가?

결론부터 말하자면 프로덕션 레벨에서는 이 코드를 사용하면 안 된다.

- **퍼포먼스:** 1.1 Tflop/s는 훌륭한 성과지만, Apple이 수년간 깎고 다듬은 Accelerate 프레임워크나 MPSGraph를 사용하는 것이 훨씬 빠르고 안정적이다.
- **유지보수성:** Unsafe 포인터와 비공개 명령어가 난무하는 코드는 기술 부채의 시한폭탄이다.

하지만 이 여정이 무가치하다는 뜻은 결코 아니다. 추상화의 장막을 걷어내고 메모리 레이아웃, 레지스터, SIMD, 병렬 처리의 본질을 바닥부터 마주하는 경험은 시니어 엔지니어의 통찰력을 한 차원 높여준다. 프레임워크 없이 극한의 최적화를 이뤄낸 Matt Gallagher에게 찬사를 보낸다.

## References
- **Original Article:** https://www.cocoawithlove.com/blog/matrix-multiplications-swift.html
- **Hacker News Thread:** https://news.ycombinator.com/item?id=48085685
