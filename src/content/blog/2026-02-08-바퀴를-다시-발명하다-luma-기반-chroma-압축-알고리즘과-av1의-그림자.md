---
title: "바퀴를 다시 발명하다: Luma 기반 Chroma 압축 알고리즘과 AV1의 그림자"
description: "바퀴를 다시 발명하다: Luma 기반 Chroma 압축 알고리즘과 AV1의 그림자"
pubDate: "2026-02-08T05:42:29Z"
---

엔지니어링 업계에는 "바퀴를 다시 발명하지 말라(Don't reinvent the wheel)"는 오래된 격언이 있습니다. 이미 검증된 라이브러리와 표준을 사용하라는 뜻이죠. 하지만 가끔은 그 바퀴를 직접 깎아보는 과정에서 우리가 당연하게 여기던 기술의 본질을 깨닫게 되기도 합니다.

오늘 소개할 글은 바로 그런 사례입니다. 한 개발자가 AI의 도움 없이, 순수하게 실험과 시행착오만으로 **Luma Dependent Chroma Compression** 알고리즘을 구현했습니다. 흥미로운 점은, 이 결과물이 최신 비디오 코덱인 AV1이나 H.266/VVC에서 사용하는 핵심 기술과 놀랍도록 닮아있다는 것입니다.

이 글에서는 이 알고리즘이 어떻게 작동하는지, 그리고 왜 현대 코덱들이 이 방식을 채택하고 있는지 'Principal Engineer'의 관점에서 뜯어보겠습니다.

## 핵심 아이디어: Luma는 Chroma를 알고 있다

이미지 압축의 기본은 RGB를 **YCbCr** 로 변환하는 것입니다. 인간의 눈은 밝기(Luma, Y)에는 민감하지만 색상(Chroma, Cb/Cr)의 해상도 저하에는 둔감하기 때문이죠. 그래서 보통 Chroma 서브샘플링(4:2:0 등)을 통해 용량을 줄입니다.

하지만 이 알고리즘의 저자는 여기서 한 걸음 더 나아갔습니다.

> "Chroma 채널의 텍스처는 Luma 채널과 매우 비슷하다. 그렇다면 Luma 값을 알면 Chroma 값을 예측할 수 있지 않을까?"

![](https://www.bitsnbites.eu/wp-content/uploads/2026/02/parrots-ycrcb-labled-1024x683.png)

위 이미지를 보면 명확합니다. 앵무새의 깃털 디테일(Edge)은 Luma 채널에 선명하게 드러나 있고, Chroma 채널에서도 같은 위치에 패턴이 존재합니다. 즉, Luma와 Chroma 사이에는 **상관관계(Correlation)** 가 존재합니다.

### 선형 회귀(Linear Regression)를 통한 예측

저자는 16x16 픽셀 블록 단위로 Luma(Y)와 Chroma(C) 사이의 관계를 1차 함수로 근사했습니다.

`C(Y) = a * Y + b`

각 블록마다 **최소자승법(Least Squares Fit)** 을 사용해 기울기 `a`와 절편 `b`를 구합니다. 이렇게 되면 Chroma 채널의 모든 픽셀을 저장할 필요 없이, 블록당 `a`, `b` 두 개의 값(또는 이를 유도할 수 있는 양 끝점 `C1`, `C2`)만 저장하면 됩니다. 복원 시에는 이미 디코딩된 Luma 값을 위 식에 대입해 Chroma를 만들어냅니다.

결과적으로 Chroma 채널을 블록당 단 32비트(Cr, Cb 합쳐서)로 표현할 수 있게 됩니다. 이는 엄청난 압축률입니다.

## 가변 블록 사이즈와 아티팩트 처리

물론 모든 영역이 선형 관계로 깔끔하게 떨어지진 않습니다. 복잡한 텍스처에서는 오차가 커지죠. 이를 해결하기 위해 저자는 **Quadtree** 방식의 가변 블록 사이즈를 도입했습니다.

- 기본 16x16 블록에서 오차가 크면 4개의 8x8 블록으로 쪼갭니다.
- 필요하면 4x4, 2x2까지 재귀적으로 들어갑니다.

![](https://www.bitsnbites.eu/wp-content/uploads/2026/02/parrots-variable-blocks-color.png)

### Deblocking: 단순하지만 효과적인 트릭

선형 근사의 치명적인 단점은 블록 경계가 눈에 띄는 'Blockiness' 현상입니다. 저자는 이를 해결하기 위해 인접 블록과의 경계 오차를 계산하고, 이를 보간(Interpolation)하여 스무딩 처리를 했습니다.

![](https://www.bitsnbites.eu/wp-content/uploads/2026/02/parrots-deblocked-1024x341.png)

이 과정에서 색 번짐(Smudging)이 발생하긴 하지만, 인간의 눈은 날카로운 블록 경계보다 부드러운 색 번짐을 더 자연스럽게 받아들입니다. 엔지니어링은 언제나 트레이드오프의 예술이니까요.

## Hacker News의 반응: "그거 AV1에 있는 건데요?"

이 글이 Hacker News에 올라오자, 비디오 코덱 전문가들이 등판했습니다. 반응은 냉소적이라기보단 **"놀랍다, 당신이 방금 최신 표준을 재발명했다"** 에 가까웠습니다.

한 유저는 이렇게 지적합니다:

> "당신의 알고리즘은 AV1의 **CfL(Chroma from Luma)** 기술과 거의 동일합니다. H.266/VVC의 CCLM(Cross Component Linear Model)이나 JPEG XL에도 포함된 표준 기술이죠."

실제로 AV1의 CfL은 Luma 픽셀들을 기반으로 DC 오프셋과 스케일링 팩터를 계산해 Chroma를 예측합니다. 저자가 구현한 `a * Y + b` 모델과 본질적으로 같습니다. 최신 코덱들은 여기에 더 복잡한 필터링과 문맥(Context) 기반 예측을 추가했을 뿐입니다.

또 다른 유저는 0.5bpp 결과물에서 Red/Yellow 계열의 손실과 Blue/Green의 이득이 보인다고 지적했습니다. 이는 Luma 계산 식(일반적으로 Green 채널의 가중치가 가장 높음)에 기인한 특성일 수 있습니다.

## 엔지니어로서의 단상

저는 이 프로젝트가 매우 가치 있다고 생각합니다. 물론 이 코드를 당장 프로덕션 이미지 압축 파이프라인에 넣을 수는 없습니다. 압축 효율이나 속도 면에서 고도로 최적화된 libavif나 libjxl을 이길 순 없으니까요.

하지만 이 프로젝트는 **"First Principles Thinking(제1원리 사고)"** 의 힘을 보여줍니다.

1.  **관찰:** Luma와 Chroma는 비슷하게 생겼다.
2.  **가설:** 수학적 모델로 변환 가능하다.
3.  **구현:** 직접 코드로 증명한다.

저자는 논문을 읽고 구현한 것이 아니라, 데이터 자체를 관찰하여 현대 압축 기술의 정수를 스스로 도출해냈습니다. 주니어 엔지니어들이 라이브러리 API만 호출하다가 놓치기 쉬운 **'밑바닥의 원리'** 를 꿰뚫은 것이죠.

### 결론

이 알고리즘은 **Chroma Subsampling** 의 훌륭한 대안이자, **Intra Prediction** 의 교과서적인 예시입니다. 만약 이미지/비디오 처리에 관심이 있다면, 복잡한 AV1 스펙 문서를 읽기 전에 [이 저자의 C++ 코드](https://codeberg.org/mbitsnbites/yuvlab)를 먼저 읽어보시길 권합니다.

때로는 바퀴를 다시 발명해보는 것이, 바퀴의 구조를 이해하는 가장 빠른 길입니다.

## References

- **Original Article:** [A spatial domain variable block size luma dependent chroma compression algorithm](https://www.bitsnbites.eu/a-spatial-domain-variable-block-size-luma-dependent-chroma-compression-algorithm/)
- **Hacker News Discussion:** [Hacker News Thread](https://news.ycombinator.com/item?id=46884412)
