---
title: "정적 타입 Lisp의 등장: Coalton은 Common Lisp와 Haskell의 완벽한 혼합일까?"
description: "정적 타입 Lisp의 등장: Coalton은 Common Lisp와 Haskell의 완벽한 혼합일까?"
pubDate: "2026-05-28T23:01:35Z"
---

## 시작하며: Lisp의 자유로움, 그리고 대규모 시스템에서의 한계

15년 넘게 백엔드와 분산 시스템을 설계해 오면서 수많은 프로그래밍 언어의 흥망성쇠를 지켜봤습니다. 그중에서도 Lisp은 항상 엔지니어들의 마음 한구석에 자리 잡은 로망 같은 언어입니다. Homoiconicity(동형성)와 매크로가 제공하는 메타 프로그래밍의 강력함은 타의 추종을 불허하죠. 하지만 현실로 돌아와 대규모 프로덕션 환경에서 동적 타입 언어인 Common Lisp(CL)로 거대한 코드베이스를 유지보수하는 것은 종종 지뢰찾기 게임이 되곤 합니다.

최근 Hacker News에서 제 눈길을 사로잡은 프로젝트가 하나 있었습니다. 바로 **Coalton** 입니다. 2025년 ELS(European Lisp Symposium)에서 Robert Smith가 발표하기도 한 이 언어는, Common Lisp 위에 Haskell과 OCaml의 강력한 정적 타입 시스템을 얹어버린 흥미로운 결과물입니다.

## Coalton: 단순한 타입 힌팅이 아닌 진짜 정적 타입 시스템

솔직히 처음엔 회의적이었습니다. "Python이나 TypeScript처럼 기존 동적 언어에 적당히 타입 어노테이션만 추가한 또 다른 시도겠지"라고 생각했거든요. 하지만 Coalton의 코드를 뜯어보니 제 예상과는 완전히 달랐습니다. 

이 녀석은 단순한 린터(Linter) 수준이 아니라, Common Lisp 내부에 완벽한 Hindley-Milner 타입 시스템을 임베딩한 **DSL(Domain Specific Language)** 에 가깝습니다.

기본적인 문법을 한번 볼까요?

```lisp
(declare add-two-ints (Integer * Integer -> Integer))
(define (add-two-ints a b)
  (+ a b))
```

위 코드를 보면 `(add-two-ints 3.5 5)`처럼 Float 값을 넘기려 하면 런타임이 아니라 **컴파일 타임** 에 Type Mismatch 에러가 발생합니다. TypeScript나 PHP에 타입을 얹은 것과 비슷한 느낌을 주지만, Coalton의 진가는 Algebraic Data Types(ADT)와 Type Class에서 드러납니다.

### 대수적 데이터 타입 (ADT)의 도입

도메인 주도 설계(DDD)를 해본 시니어 엔지니어라면, 비즈니스 로직을 모델링할 때 Sum Type(합 타입)과 Product Type(곱 타입)이 얼마나 중요한지 아실 겁니다. Coalton은 이를 `define-type`과 `define-struct`로 우아하게 풀어냅니다.

```lisp
(define-type PaymentMethod
  (CreditCard CreditCardInfo)
  (PayPal     Email)
  (Check      CheckNumber)
  StoreCredit
  Cash)
```

이 코드를 보면 Haskell이나 F#의 냄새가 강하게 납니다. 결제 수단이라는 도메인을 명확하게 정의하고, 이후 로직에서는 `match` 패턴 매칭을 통해 각 케이스를 안전하게 처리할 수 있습니다. `StoreCredit`이나 `Cash`처럼 추가 데이터가 필요 없는 경우도 깔끔하게 표현됩니다. 

이 패턴을 보니 예전 2018년경 복잡한 결제 상태 머신을 설계할 때, Enum과 Nullable 필드들을 덕지덕지 붙여가며 고생했던 기억이 납니다. 이런 강력한 타입 시스템이 Lisp 환경에 존재했다면 훨씬 우아하게 해결했을 텐데 말이죠.

### 인터페이스와 다형성: Type Class

Haskell의 Type Class 개념도 그대로 이식되었습니다.

```lisp
(declare calculate-total-tax ((Taxable :a) => :a * Fraction -> Cents))
```

여기서 `Taxable`은 Type Class Constraint입니다. 과세 대상인 `Widget`이든, 비과세 대상인 `GiftCard`든 `Taxable` 인터페이스(Type Class)를 구현(`define-instance`)하기만 하면 동일한 다형성 함수를 안전하게 사용할 수 있습니다.

## Hacker News의 반응과 나의 생각

이 프로젝트에 대한 Hacker News 커뮤니티의 반응은 꽤 뜨거웠습니다. 몇 가지 흥미로운 논점들을 짚어보겠습니다.

### 1. "그냥 OCaml에 S-expression 껍데기를 씌우면 되는 거 아님?"

한 유저가 "OCaml에 S-expression 문법을 얹어서 매크로를 쓰면 되지, 굳이 왜 CL 위에 이걸 만드냐"는 날카로운 질문을 던졌습니다. 

제 생각은 다릅니다. Coalton의 핵심 가치는 **Common Lisp 생태계와의 완벽한 상호 운용성** 에 있습니다. OCaml로 넘어가면 기존 CL 라이브러리, 강력한 CLOS(Common Lisp Object System), 그리고 수십 년간 축적된 Lisp의 자산들을 버려야 합니다. Coalton은 핵심 비즈니스 로직은 안전하게 정적 타입으로 작성하고, I/O나 동적인 처리가 필요한 부분은 기존 CL 코드를 그대로 호출할 수 있게 해줍니다. 심지어 같은 파일 안에서 섞어 쓰는 것도 가능하죠. 이것은 엔지니어링 관점에서 엄청난 유연성을 제공합니다.

### 2. 가파른 학습 곡선과 툴링의 부재

많은 유저들이 Coalton의 진입 장벽을 지적했습니다. 한 유저는 "Common Lisp도 모르는데 Coalton부터 시작하려니 막막하다"며 튜토리얼의 부재를 아쉬워했습니다.

커뮤니티의 조언처럼, 저 역시 **Common Lisp와 Coalton은 분리해서 학습해야 한다** 고 봅니다. CL을 먼저 배우고 싶다면 여전히 [Practical Common Lisp (PCL)](https://gigamonkeys.com/book/)가 바이블입니다. 

툴링 측면에서는 VS Code 확장 프로그램들이 나오고 있지만(최근 안정화된 새로운 확장이 Reddit에 발표되기도 했죠), 본질적인 Lisp 개발 경험을 원한다면 Emacs + SLIME이나 Vim + slimv 환경을 구축하는 것을 추천합니다. REPL과 에디터가 한 몸처럼 움직이는 경험은 VS Code가 아직 완전히 따라잡지 못한 영역입니다.

## 결론: 그래서 프로덕션에 쓸 만한가?

솔직히 말해, 내일 당장 여러분의 회사의 메인 백엔드를 Coalton으로 재작성하라고 권하고 싶지는 않습니다. 아직 생태계가 작고, 베스트 프랙티스나 레퍼런스 프로젝트가 턱없이 부족합니다. 

하지만 이미 Common Lisp를 프로덕션에서 사용 중이거나, 팀 내에 Lisp 해커들이 많은 조직이라면? **Coalton은 게임 체인저가 될 수 있습니다.** 특히 금융 시스템이나 복잡한 룰 엔진처럼 컴파일 타임의 타입 안정성이 극도로 중요한 코어 도메인을 분리해 낼 때, Coalton은 최고의 선택지가 될 수 있습니다.

결국 Coalton은 언어 설계의 훌륭한 타협점을 보여줍니다. 동적 언어의 유연함과 정적 언어의 안전성이 어떻게 한 지붕 아래에서 평화롭게 공존할 수 있는지 증명한 멋진 아키텍처적 실험입니다.

---

- **원문 링크:** [Coalton Official Website](https://coalton-lang.github.io/)
- **Hacker News 토론:** [HN Thread](https://news.ycombinator.com/item?id=48280451)
