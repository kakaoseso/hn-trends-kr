---
title: "Apple M4 Neural Engine 리버스 엔지니어링: CoreML을 버리고 하드웨어와 직접 통신하기"
description: "Apple M4 Neural Engine 리버스 엔지니어링: CoreML을 버리고 하드웨어와 직접 통신하기"
pubDate: "2026-03-03T07:54:57Z"
---

최근 시스템 엔지니어링 커뮤니티를 뜨겁게 달군 흥미로운 리버스 엔지니어링 사례가 하나 있습니다. 바로 Apple의 M4 Neural Engine(ANE)을 밑바닥부터 파헤친 Maderix의 작업입니다. 

15년 넘게 다양한 하드웨어 가속기와 커널 레벨 코드를 다뤄온 제 입장에서, 이번 분석은 단순한 해킹을 넘어 Apple의 실리콘 설계 철학과 폐쇄적인 생태계의 민낯을 엿볼 수 있는 훌륭한 레퍼런스였습니다. 우리는 항상 CoreML이라는 거대한 추상화 레이어 뒤에 숨겨진 ANE의 진짜 성능이 궁금했죠. 오늘 그 껍데기를 한 번 벗겨보겠습니다.

## CoreML이라는 블랙박스

Apple은 개발자가 ANE를 직접 제어하는 것을 원치 않습니다. 공식적인 ISA(명령어 집합) 문서도 없고, 내부 아키텍처에 대한 설명도 전무합니다. 모든 것은 CoreML을 통해서만 이루어집니다. 

하지만 시스템 엔지니어라면 누구나 공감하듯, 이런 고수준의 프레임워크는 수많은 최적화 패스와 추상화 오버헤드를 동반합니다. 내가 작성한 모델이 하드웨어에서 정확히 어떻게 스케줄링되고 실행되는지 알 길이 없다는 건 정말 답답한 일입니다. 

작성자는 이 블랙박스를 우회하기 위해 `AppleNeuralEngine.framework` 내부의 프라이빗 API인 `_ANEClient`를 찾아냈습니다. 이를 통해 CoreML을 완전히 배제하고 직접 컴파일, 로드, 실행 파이프라인을 구축하는 데 성공했습니다.

### 프라이빗 API를 통한 직접 제어

발견된 파이프라인의 핵심은 다음과 같습니다.

```objc
// 1. 클라이언트 연결
id client = [_ANEClient sharedConnection];

// 2. 모델 참조 및 컴파일
id model = [_ANEModel modelAtURL:compiledURL key:@"mykey"];
[client compileModel:model options:...];

// 3. 하드웨어에 로드 (queueDepth = 127)
[client loadModel:model options:...];

// 4. IOSurface를 이용한 I/O 버퍼 생성
IOSurfaceRef surface = IOSurfaceCreate(props);
id wrapped = [_ANEIOSurfaceObject objectWithIOSurface:surface];

// 5. 실행 및 결과 읽기
[client evaluateWithModel:model options:{} request:req ...];
```

여기서 가장 눈에 띄는 것은 I/O에 `IOSurface`를 사용한다는 점입니다. GPU 텍스처 공유에 사용되는 바로 그 메커니즘입니다. 이론적으로 GPU와 ANE가 동일한 메모리 포인터를 공유하는 **Zero-copy** 파이프라인 구축이 가능하다는 뜻입니다. 과거 비디오 프로세싱 파이프라인을 최적화할 때 이 방식을 써서 Latency를 극적으로 줄였던 기억이 나는데, 머신러닝 추론에서도 동일한 아키텍처를 적용할 수 있다는 건 엄청난 이점입니다.

## ANE는 GPU가 아니다

이번 분석에서 가장 흥미로웠던 기술적 발견은 E5 컴파일 바이너리의 구조입니다. 

CoreML은 네트워크를 MIL(Machine Learning Intermediate Language)이라는 SSA 포맷으로 변환한 뒤, 이를 ANECompiler를 통해 E5 바이너리로 만듭니다. 놀랍게도 1024x1024 행렬 곱셈이나 128x128 행렬 곱셈이나 컴파일된 바이너리의 크기는 약 2.6KB로 거의 동일했습니다.

이게 무슨 의미일까요? ANE는 GPU처럼 스레드가 개별 명령어를 패치하고 실행하는 구조가 아닙니다. 텐서의 형태(Shape)를 파라미터로 받아 실행되는 고정 기능(fixed-function) 하드웨어, 즉 거대한 Systolic Array에 가깝다는 뜻입니다. E5 바이너리는 전통적인 기계어라기보다는 하드웨어 파이프라인을 세팅하는 Configuration 파일에 불과합니다.

## 마케팅의 허상: 38 TOPS의 진실

Apple은 M4 ANE가 38 TOPS의 성능을 낸다고 광고합니다. 하지만 하드웨어 레벨에서 측정해 본 결과, 이는 철저히 마케팅 부서의 계산법이었습니다. 

업계 관행에 따라 INT8 연산 속도를 FP16의 2배로 계산하여 부풀린 수치일 뿐, 실제 하드웨어가 INT8을 두 배 빠르게 처리하는 것은 아니었습니다. Hacker News에서도 이 부분에 대한 비판이 많았죠. 칩 설계 엔지니어들이 아무리 훌륭한 하드웨어를 만들어도, 마케팅 부서가 숫자를 뻥튀기하는 건 Apple이나 다른 벤더나 매한가지인 것 같습니다.

## Hacker News 커뮤니티의 반응

이 글이 올라오고 Hacker News는 말 그대로 난리가 났습니다. 특히 인상 깊었던 두 가지 논쟁거리가 있었습니다.

- **실제 활용 사례:** 한 유저는 이 프라이빗 API를 활용해 Karpathy의 NanoGPT 학습 코드를 ANE에 오프로드하는 데 성공했습니다. Classifier 레이어에서 10배, Softmax에서 34배의 속도 향상을 얻어냈다고 합니다. ANE가 단순한 추론용 장난감이 아니라, 특정 워크로드에서는 엄청난 Throughput을 낼 수 있다는 걸 증명한 셈입니다.
- **AI와의 협업:** 원작자는 이 모든 리버스 엔지니어링 과정을 Claude Opus 4.6과 함께 진행했다고 밝혔습니다. 일부 시니어 엔지니어들은 "AI가 짠 코드를 어떻게 믿느냐"며 강한 거부감을 보였지만, 솔직히 저는 동의하기 어렵습니다. 복잡한 Objective-C 덤프를 파싱하고 보일러플레이트를 짜는 건 AI에게 맡기고, 인간은 아키텍처의 큰 그림과 가설 검증에 집중하는 것. 이것이 바로 현재이자 미래의 시스템 리서치 방식입니다.

## 총평: 프로덕션에 쓸 수 있을까?

결론부터 말씀드리면, **절대 프로덕션 코드에 넣지 마십시오.**

이것은 철저히 문서화되지 않은 프라이빗 API입니다. Apple은 다음 macOS 업데이트에서 클래스 이름을 바꾸거나 샌드박스 권한을 비틀어버릴 수 있습니다. MLX 팀조차 ANE의 소스 코드에 접근하지 못해 OSS 통합에 어려움을 겪는다는 루머가 있을 정도로 Apple의 통제는 엄격합니다.

하지만 Local LLM을 개발하거나 극한의 성능 튜닝을 즐기는 해커들에게 이 연구는 엄청난 가치를 지닙니다. CoreML의 오버헤드를 걷어내고 하드웨어의 바닥까지 내려가 본 경험은, 향후 Apple Silicon 위에서 돌아가는 모든 ML 프레임워크의 한계를 이해하는 데 큰 도움이 될 것입니다.

다음 파트에서는 실제 벤치마크와 모델 학습 과정이 공개된다고 하니, 벌써부터 기대가 됩니다. 

**References:**
- Original Article: [Inside the M4 Apple Neural Engine, Part 1: Reverse Engineering](https://maderix.substack.com/p/inside-the-m4-apple-neural-engine)
- Hacker News Thread: [https://news.ycombinator.com/item?id=47208573](https://news.ycombinator.com/item?id=47208573)
