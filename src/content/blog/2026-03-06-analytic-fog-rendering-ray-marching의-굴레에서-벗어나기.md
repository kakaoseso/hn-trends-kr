---
title: "Analytic Fog Rendering: Ray-marching의 굴레에서 벗어나기"
description: "Analytic Fog Rendering: Ray-marching의 굴레에서 벗어나기"
pubDate: "2026-03-06T15:44:05Z"
---

최근 렌더링 엔지니어링 씬을 보면 볼류메트릭(Volumetric) 효과는 거의 Ray-marching의 독무대가 된 것 같다. 하지만 성능 예산(Frame budget)이 빡빡한 환경에서 Ray-marching은 여전히 골칫거리다. 최근 Matej Lou의 블로그에 올라온 'Analytic Fog Rendering with Volumetric Primitives'라는 글을 읽고, 우리가 너무 수치적 적분(Numerical Integration)에만 매몰되어 있던 것은 아닌지 다시 생각해보게 되었다.

## Ray-marching의 딜레마

안개나 연기 같은 이질적인 매질(Heterogenous Volumes)을 렌더링하는 기본 원리는 Beer-Lambert Law를 따른다. 빛이 매질을 통과하며 에너지를 잃는 과정(Absorption)을 계산하는 것인데, 밀도가 일정하지 않기 때문에 광선을 따라 적분해야 한다.

![Ray-marching](https://matejlou.blog/wp-content/uploads/2025/01/screenshot-2025-01-20-at-20.59.36.png?w=1012)

현재 업계 표준은 Ray-marching이다. 볼륨 텍스처를 따라 작은 스텝으로 이동하며 밀도를 누적하는 방식이다. 유연성은 최고지만, 치명적인 단점이 있다. 샘플링 횟수가 부족하면 여지없이 Aliasing이 발생한다는 것이다. 솔직히 말해서, 매 프레임마다 수십 번씩 텍스처를 샘플링하며 스텝을 밟는 건 무식한 방법이다. 최신 GPU 성능에 기대어 밀어붙이고 있을 뿐, 모바일 환경이나 인디 게임 엔진에서 이는 너무 무거운 무기다.

## Analytic Integration: 우아한 수학적 접근

Matej가 제시한 방법은 고전적이지만 매우 우아하다. Ray-marching처럼 스텝을 밟는 대신, 밀도 함수 자체의 역도함수(Antiderivative)를 구해 광선의 시작점과 끝점 사이의 정적분 값을 한 번에 계산해버리는 것이다.

![Cross Section](https://matejlou.blog/wp-content/uploads/2025/01/crosssection-1.png?w=800)

중심에서 멀어질수록 밀도가 변하는 방사형 함수(Radial Functions)를 사용하면, 광선이 이 Primitive를 관통할 때의 단면을 수식으로 정의할 수 있다. 이렇게 하면 Aliasing이 아예 발생하지 않으며(수학적으로 완벽한 연속 곡선이므로), 계산 속도도 압도적으로 빠르다.

## 코드 레벨의 우아함 (Quartic Density)

저자가 소개한 여러 함수(Linear, Quadratic, Spiky, Gaussian 등) 중 개인적으로 가장 마음에 들었던 것은 Quartic Density 함수다. 경계면에서 도함수가 0이 되기 때문에 Smoothstep처럼 매우 자연스러운 감쇠(Falloff)를 보여준다.

![Quartic Density](https://matejlou.blog/wp-content/uploads/2025/01/desmos-graph-3.png?w=1024)

```glsl
float antiderivativeQuartic(float a, float b, float c, float t)
{
    return
    t * (t * (t * (t * (a * a * t / 5.0f + a * b) +
    (2.0f * b * b + a * c - a) * 2.0f / 3.0f) +
    (b * c - b) * 2.0f) + (c * c - 2.0f * c + 1.0f));
}
```

루프(Loop)도 없고, 텍스처 페치(Texture fetch)도 없다. 그저 순수한 다항식 연산뿐이다. GPU ALU 관점에서 이보다 더 사랑스러운 코드가 있을까?

## 한계점과 현실적인 타협

하지만 15년 넘게 렌더링 파이프라인을 만져본 입장에서 냉정하게 평가하자면, 이 기술이 당장 AAA 게임의 구름 렌더링을 대체하지는 못할 것이다. 가장 큰 아킬레스건은 **빛과 그림자** (Lighting and Shadow) 처리다.

밀도를 해석적으로 구하는 것은 완벽하지만, 매질 내부에서 발생하는 다중 산란(Multiple scattering)이나 자체 그림자(Self-shadowing)를 해석적 방정식으로 풀어내는 것은 현재 실시간 렌더링 환경에서는 사실상 불가능에 가깝다. 저자 역시 개별 Primitive에 단순한 셰이딩 모델을 적용하는 꼼수(Hack)를 제안하고 있으며, 이는 물리 기반 렌더링(PBR) 환경에서는 이질감을 줄 수 있다.

## 나의 결론

그럼에도 불구하고 이 접근법은 여전히 강력한 무기다. 복잡한 형태의 Bounding Volume을 Constructive Solid Geometry(CSG) 연산으로 엮어서 아래와 같은 Fog of War 효과를 만들거나, 파티클 시스템에 연동하여 동적인 연기 효과를 만들 때 Ray-marching보다 훨씬 가볍고 깔끔한 결과를 낼 수 있다.

![Fog of War](https://matejlou.blog/wp-content/uploads/2025/02/screenshot-2025-02-09-at-23.33.39.png?w=1024)

- **Verdict:** 실사풍 렌더링의 메인 솔루션으로는 무리가 있지만, 모바일 게임, 스타일라이즈드 아트, 또는 최적화가 극도로 중요한 특수 VFX 구현에는 완벽한 기술이다. 무작정 무거운 볼륨 렌더링을 얹기 전에, 이런 해석적(Analytic) 접근을 먼저 고민해보는 엔지니어가 더 많아지기를 바란다.

### References
- **Original Article:** https://matejlou.blog/2025/02/11/analytic-fog-rendering-with-volumetric-primitives/
- **Hacker News Thread:** https://news.ycombinator.com/item?id=47263441
