---
title: "RISC-V Vector Extension: SIMD의 파편화 지옥을 끝낼 수 있을까?"
description: "RISC-V Vector Extension: SIMD의 파편화 지옥을 끝낼 수 있을까?"
pubDate: "2026-02-12T08:41:07Z"
---

솔직히 고백하자면, 저는 x86의 SIMD 파편화에 지칠 대로 지친 상태입니다. MMX부터 시작해서 SSE, AVX, AVX2, 그리고 악명 높은 AVX-512까지. 인텔이 새로운 명령어 세트를 내놓을 때마다 컴파일러 옵션을 손보고, 런타임에 CPU 기능을 체크해서 분기하는 코드를 짜는 건 엔지니어로서 정말 고역이었습니다.

그런 와중에 오늘 Hacker News에서 **RISC-V Vector Primer** 라는 문서를 발견했습니다. 처음에는 "또 다른 아키텍처의 매뉴얼이겠거니" 하고 넘기려다, 내용을 훑어보고는 꽤나 충격을 받았습니다. 오늘은 이 문서가 왜 흥미로운지, 그리고 RISC-V의 벡터 접근 방식이 왜 기존 SIMD의 패러다임을 바꿀 잠재력이 있는지 이야기해보려 합니다.

## SIMD가 아니라 'Vector'다

많은 분들이 RISC-V Vector(RVV)를 단순히 "RISC-V 버전의 NEON이나 AVX"라고 생각합니다. 하지만 이 Primer를 읽어보면 근본적인 철학이 다르다는 것을 알 수 있습니다. RVV는 **Cray 스타일의 벡터 아키텍처** 를 현대적으로 계승했습니다.

핵심은 **Vector Length Agnostic (VLA)** 입니다.

기존 x86이나 ARM NEON은 레지스터 크기가 고정되어 있습니다(128-bit, 256-bit 등). 하드웨어가 바뀌면 소프트웨어도 다시 컴파일하거나 최적화해야 했죠. 하지만 RVV는 하드웨어 구현체가 벡터 레지스터를 128비트로 만들든, 512비트로 만들든, 소프트웨어는 동일한 바이너리로 돌아갑니다.

### `vsetvli`의 마법

이 Primer에서 가장 잘 설명하고 있는 부분이 바로 `vsetvli` 명령어입니다. 보통 SIMD 루프를 짤 때는 남은 데이터(remainder) 처리를 위해 지저분한 코드가 들어가기 마련입니다. 하지만 RVV에서는 하드웨어가 "이번 턴에 몇 개의 요소를 처리할 수 있는지"를 알려줍니다.

```assembly
# 의사 코드(Pseudo-code) 예시
loop:
    vsetvli t0, a0, e32, m1  # 남은 요소 개수(a0)에 맞춰 벡터 길이 설정
    vle32.v v0, (a1)         # 메모리 로드
    vadd.vv v0, v0, v1       # 벡터 덧셈
    vse32.v v0, (a2)         # 메모리 저장
    sub a0, a0, t0           # 남은 개수 업데이트
    add a1, a1, t0           # 포인터 이동
    bnez a0, loop            # 반복
```

보시다시피 레지스터 폭(Width)에 대한 하드코딩이 전혀 없습니다. 이 코드는 소형 임베디드 칩에서도, 거대한 HPC용 코어에서도 수정 없이 돌아갑니다. 이것이 바로 우리가 꿈꾸던 **Binary Compatibility** 입니다.

## 문서 퀄리티와 커뮤니티의 반응

이 GitHub 리포지토리(Primer)는 RVV의 복잡한 스펙을 엔지니어가 이해하기 쉽게 잘 풀어냈습니다. 스펙 문서(Specification)는 법전처럼 딱딱해서 읽기 힘든데, 이 문서는 실전 가이드에 가깝습니다.

Hacker News의 댓글을 보니 재미있는 반응이 있더군요. 한 유저(anon)는 마이크로소프트가 GitHub UI를 너무 무겁게 만들어서 리포지토리를 보려면 클론을 떠야 한다고 불평하더군요.

> "impossible to have a 'classic web' look a the repo. Must clone it now..."

저도 공감합니다. 요즘 웹들이 너무 SPA(Single Page Application)로 넘어가면서 기본기를 잃어가는 느낌인데, 역설적으로 그 무거운 UI 속에 담긴 내용은 "군더더기 없는 로우레벨 최적화"를 다루고 있다는 점이 아이러니합니다.

## 현실적인 한계: 하드웨어는 어디에?

기술적으로는 RVV가 압도적으로 우아합니다. 하지만 Principal Engineer로서 냉정하게 평가하자면, **"그래서 이걸 지금 프로덕션에 쓸 수 있는가?"** 라는 질문에는 아직 물음표가 붙습니다.

1.  **하드웨어 보급:** RVV 1.0 스펙을 제대로 지원하는 고성능 칩을 구하기가 여전히 어렵습니다. 알리바바의 T-Head나 일부 보드들이 나오고 있지만, x86이나 ARM을 대체할 수준의 퍼포먼스와 가용성을 보여주진 못하고 있습니다.
2.  **생태계:** LLVM과 GCC의 지원은 훌륭하지만, 오토 벡터라이제이션(Auto-vectorization)이 인텔만큼 성숙하려면 시간이 더 필요합니다.

## 결론: 미래는 밝지만, 아직은 'Early Access'

RISC-V Vector Extension은 하드웨어 설계자와 컴파일러 개발자들에게는 축복과 같은 아키텍처입니다. **Strip-mining** 을 하드웨어 레벨에서 우아하게 처리하는 방식은 확실히 미래지향적입니다.

시스템 엔지니어라면 이 Primer를 일독하시길 권합니다. 당장 회사 코드에 적용하진 못하더라도, **"제대로 된 벡터 컴퓨팅이란 무엇인가"** 에 대한 통찰을 얻을 수 있습니다. x86의 덕지덕지 붙은 레거시에서 벗어나, 깔끔한 설계를 보는 것만으로도 머리가 맑아지는 기분이 드실 겁니다.

**참고 링크:**
- [RISC-V Vector Primer (GitHub)](https://github.com/simplex-micro/riscv-vector-primer/blob/main/index.md)
- [Hacker News Discussion](https://news.ycombinator.com/item?id=46923051)
