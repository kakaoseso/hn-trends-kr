---
title: "스카치 테이프 하나로 카메라 렌즈를 대체하는 방법: Computational Imaging의 세계"
description: "스카치 테이프 하나로 카메라 렌즈를 대체하는 방법: Computational Imaging의 세계"
pubDate: "2026-02-16T19:41:26Z"
---

최근 Hacker News를 보다가 눈길을 끄는 제목을 발견했습니다. **"How to take a photo with scotch tape"**. 수천 달러짜리 렌즈를 사 모으는 카메라 애호가들이라면 기절초풍할 소리지만, 엔지니어링 관점에서 이 '장난감 같은 실험'은 현대 이미징 기술의 핵심을 꿰뚫고 있습니다.

오늘은 이 영상이 보여주는 **Lensless Imaging(렌즈리스 이미징)** 의 원리와, 이것이 왜 단순한 눈요기 이상의 기술적 가치를 지니는지 Principal Engineer의 시선에서 뜯어보겠습니다.

## 렌즈가 없는데 어떻게 상이 맺히나?

우리가 흔히 쓰는 카메라는 렌즈를 통해 빛을 굴절시켜 센서의 특정 픽셀에 1:1로 매핑합니다. 핀포인트에서 출발한 빛이 센서의 한 점에 모여야 선명한 이미지가 되죠. 초점이 안 맞으면 흐릿해지는 이유입니다.

그런데 영상에서처럼 센서 앞에 스카치 테이프를 붙이면 어떻게 될까요? 테이프의 불규칙한 표면 때문에 빛이 난반사(Scattering)됩니다. 센서에 들어오는 데이터는 이미지가 아니라, 마치 TV 노이즈 같은 뿌연 **Speckle Pattern** 이 됩니다. 육안으로 보면 그냥 쓰레기 데이터(Garbage Data)처럼 보이죠.

하지만 이 '쓰레기' 안에는 원본 이미지의 정보가 암호화되어 들어 있습니다. 바로 여기서 **Computational Photography** 의 마법이 시작됩니다.

## 핵심은 PSF (Point Spread Function)

이 기술의 핵심은 **Convolution(합성곱)** 입니다. 수학적으로 이미징 프로세스는 다음과 같이 표현됩니다.

> **Captured Image = Original Scene * PSF + Noise**

여기서 `*`는 Convolution 연산이고, **PSF(Point Spread Function)** 는 점 광원 하나가 센서에 어떻게 퍼지는지를 나타내는 함수입니다. 일반 렌즈의 PSF는 아주 작은 점(Delta function에 가까움)이지만, 스카치 테이프의 PSF는 복잡한 구름 모양의 패턴입니다.

### 엔지니어링 접근법: 역문제(Inverse Problem) 풀기

우리가 찍은 사진(Captured Image)은 엉망이지만, 우리가 **PSF** 를 정확히 알고 있다면? 수학적으로 거꾸로 연산하여 **Original Scene** 을 복원할 수 있습니다. 이를 **Deconvolution** 이라고 합니다.

1.  **Calibration:** 먼저 작은 점 광원(LED 등)을 찍어서 스카치 테이프 특유의 빛 번짐 패턴(PSF)을 기록합니다. 이 패턴은 테이프를 떼지 않는 한 고정값(Time-invariant)입니다.
2.  **Reconstruction:** 실제 피사체를 찍은 뒤, 기록해둔 PSF와 Cross-correlation(상호상관)을 돌리거나, 주파수 도메인(Fourier Transform)으로 변환하여 나눗셈 연산을 수행합니다.

결국 하드웨어(렌즈)가 해야 할 일을 소프트웨어(알고리즘)가 대신 처리하는 셈입니다. 이것이 바로 최근 스마트폰 카메라들이 물리적 센서 크기의 한계를 극복하는 방식이기도 합니다.

## 나의 생각: Hardware vs Software의 줄다리기

솔직히 처음 이 개념을 접했을 때는 "굳이 왜?"라는 생각이 들었습니다. 핀홀(Pinhole) 카메라만 해도 충분히 간단한데, 굳이 복잡한 연산까지 해가며 테이프를 써야 하나 싶었죠. 하지만 15년 넘게 시스템 설계를 해오다 보니 이 접근 방식이 주는 통찰이 보입니다.

-   **Cost Shifting:** 고정밀 광학 유리는 비쌉니다. 반면 컴퓨팅 파워는 점점 싸지고 있죠. 하드웨어 비용을 소프트웨어 복잡도로 전가하는 것은 현대 Tech 업계의 거대한 트렌드입니다.
-   **응용 가능성:** 가시광선 영역에서는 장난감 같지만, 렌즈를 만들기 어려운 **X-ray** 나 **Gamma-ray** 이미징, 혹은 렌즈를 넣을 공간이 없는 초소형 내시경 분야에서는 이 'Coded Aperture' 방식이 필수적입니다.

Hacker News의 댓글 반응은 다소 조용하거나 농담(Scotch tape니까 스코틀랜드 드립 등) 위주였지만, 이 실험이 내포한 'Hacker Spirit'—주변의 사물로 고도의 기술적 원리를 구현하는 태도—는 높이 살만합니다.

## 결론 (Verdict)

이 기술이 당장 여러분의 DSLR을 대체할까요? **절대 아닙니다.** 빛의 효율(Light Efficiency)도 떨어지고, 노이즈에 매우 취약하며, 실시간 처리에 리소스도 많이 듭니다.

하지만 **Signal Processing** 이나 **Deep Learning** 기반의 이미지 복원을 공부하는 주니어 엔지니어라면, 이보다 더 좋은 실습 프로젝트는 없을 겁니다. OpenCV 몇 줄로 '쓰레기 데이터'에서 '형체'를 끌어내는 경험은, 단순히 API를 호출하는 것과는 차원이 다른 엔지니어링 카타르시스를 줍니다.

**참고 링크:**
-   Original Video: [https://www.youtube.com/watch?v=97f0nfU5Px0](https://www.youtube.com/watch?v=97f0nfU5Px0)
-   Hacker News Discussion: [https://news.ycombinator.com/item?id=47037313](https://news.ycombinator.com/item?id=47037313)
