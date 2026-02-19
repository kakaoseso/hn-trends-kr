---
title: "Hindley-Milner vs Bidirectional Typing: 당신의 질문은 틀렸습니다"
description: "Hindley-Milner vs Bidirectional Typing: 당신의 질문은 틀렸습니다"
pubDate: "2026-02-19T12:37:59Z"
---

사이드 프로젝트로 프로그래밍 언어를 만들어본 경험, 엔지니어라면 한 번쯤 있으실 겁니다. 파서(Parser)를 작성하고 AST를 정의하는 것까진 즐겁죠. 그런데 **Type System** 이라는 거대한 벽을 마주하는 순간, 우리는 깊은 고민에 빠집니다.

"Hindley-Milner(HM)를 써서 완벽한 추론을 구현할까? 아니면 Bidirectional Typing(Bidir)을 써서 좀 더 명시적인 시스템을 만들까?"

최근 Hacker News와 기술 블로그들 사이에서 이 오래된 떡밥이 다시 불타올랐습니다. Thunderseethe의 아티클 **'How to choose between Hindley-Milner and bidirectional typing'** 이 그 주인공인데요. 이 글은 우리의 고민이 근본적으로 잘못되었다고 지적합니다. 15년 차 엔지니어로서, 그리고 수많은 언어의 흥망성쇠를 지켜본 입장에서 이 논쟁을 정리해보고자 합니다.

## HM과 Bidir는 대척점이 아니다

보통 우리는 이 두 가지를 양극단에 둡니다. 한쪽에는 타입 변수와 Unification(단일화)으로 무장한 **HM** 이 있고, 반대쪽에는 어노테이션(Annotation)에 의존하는 **Bidir** 가 있다고 생각하죠.

하지만 저자는 이것이 **'거짓 딜레마(False Dichotomy)'** 라고 말합니다. 핵심은 이겁니다:

> "당신의 언어에 제네릭(Generics)이 필요한가?"

제네릭이 필요하다면, 당신은 필연적으로 **Unification** 이 필요합니다. Rust에서 `Vec<T>`의 `T`를 추론하는 과정을 생각해보세요. 그게 바로 Unification 엔진이 돌아가는 소리입니다. HM을 선택한다는 건 Unification을 선택한다는 것과 같습니다.

## Bidir는 HM의 상위 집합이다 (Superset)

여기서 재미있는 통찰이 나옵니다. 많은 사람이 Bidir는 Unification을 안 쓴다고 오해하지만, 사실 Bidir는 HM을 포함할 수 있는 구조입니다. Rust 코드로 예시를 들어보죠.

HM 스타일의 추론 함수는 보통 이렇습니다:

```rust
fn infer(env: Env, ast: Ast) -> Result<Type, TypeError> {
    // ... HM의 마법이 일어나는 곳
}
```

Bidir 시스템은 여기에 `check` 함수를 추가합니다. 흥미로운 점은 `check` 함수 내부에서 `infer`와 `unify`를 호출할 수 있다는 겁니다.

```rust
fn check(env: Env, ast: Ast, ty: Type) -> Result<(), TypeError> {
    let infer_ty = infer(env, ast);
    unify(infer_ty, ty)
}
```

보이시나요? `infer`로 타입을 추론하고, 기대하는 타입(`ty`)과 `unify`를 시도합니다. 이렇게 하면 **Bidirectional 구조 안에서 Unification을 슬롯처럼 끼워 넣을 수 있습니다.** 즉, HM을 선택할 거라면 차라리 코드 몇 줄 더 써서 Bidir로 감싸는 게 낫다는 논리죠. "It's free real estate"라는 표현이 딱 맞습니다.

## 엔지니어의 관점: 왜 Bidir가 대세가 되었나?

이론적으로는 HM이 우아합니다. 어노테이션 하나 없이 모든 타입을 추론해내는 ML 계열 언어들을 보면 감탄이 나오죠. 하지만 **실무 레벨(Production Level)** 에서는 이야기가 다릅니다.

### 1. 에러 메시지의 품질 (Error Locality)
순수 HM 시스템의 가장 큰 단점은 에러의 **'비국소성(Non-locality)'** 입니다. 코드의 윗부분에서 잘못 추론된 타입 변수가 수십 라인 아래에서 터지는데, 컴파일러는 엉뚱한 곳을 가리키곤 합니다. 반면 Bidir는 `check` 단계에서 기대하는 타입(Expected Type)이 명확하기 때문에, 에러가 발생한 지점을 정확히 짚어낼 수 있습니다. TypeScript나 Rust가 훌륭한 에러 메시지를 보여주는 비결 중 하나가 바로 이 구조입니다.

### 2. 고급 기능 지원 (Subtyping & Higher-Rank Types)
Hacker News 댓글에서도 지적되었듯, 서브타이핑(Subtyping)이나 고차 타입(Higher-Rank Types)이 들어가는 순간 HM의 추론은 결정 불가능(Undecidable)해지거나 매우 복잡해집니다. 현대적인 언어들이 대부분 Bidir를 채택하는 이유가 여기에 있습니다. Haskell조차 초기엔 HM이었지만, 복잡한 기능들을 수용하며 Bidir 스타일로 진화했습니다.

## 커뮤니티의 목소리: "추론이 과연 정답인가?"

HN의 토론 스레드에서는 더 근본적인 회의론도 제기되었습니다.

> "복잡한 타입 추론이 정말 필요한가? 그냥 타입을 명시하게 하는 게 가독성과 에러 메시지 면에서 더 낫지 않은가?"

저도 이 의견에 어느 정도 동의합니다. C++의 `auto` 남발이나, 지나치게 생략된 코드는 읽는 사람에게 '인지적 부하(Mental Burden)'를 줍니다. IDE의 도움 없이는 타입을 알 수 없는 코드는 좋은 코드가 아닙니다. Go 언어가 제네릭 도입을 그토록 미루고, 타입 추론을 제한적으로만 허용한 것도 같은 맥락일 겁니다.

반면, **"타이핑(Typing)을 줄여주는 것이 아니라, 생각의 흐름을 끊지 않게 해주는 것이 추론의 핵심"** 이라는 반론도 인상적입니다. 프로토타이핑 단계에서는 엄격한 타입 명시가 방해물이 될 수 있으니까요.

## 결론: 무엇을 선택해야 할까?

만약 여러분이 범용 프로그래밍 언어를 만들고 있고, 제네릭을 지원할 계획이라면 답은 정해져 있습니다. **Unification을 품은 Bidirectional Typing** 으로 가세요.

1.  **확장성:** 나중에 서브타이핑이나 더 복잡한 기능을 넣기 쉽습니다.
2.  **DX (Developer Experience):** 더 나은 에러 메시지를 제공할 수 있습니다.
3.  **유연성:** 필요한 곳엔 추론을(infer), 명시가 필요한 곳엔 확인을(check) 적용할 수 있습니다.

하지만 단순히 학습용이거나, 아주 작은 DSL을 만든다면? 굳이 Unification이라는 복잡한 괴물을 끌어들일 필요 없습니다. 단순한 Bidir 시스템만으로도 충분히 강력하고, 구현은 훨씬 쉽습니다.

결국 **"어떤 알고리즘이 더 우월한가?"** 가 아니라 **"내 언어의 사용자가 얼마나 많은 타입을 직접 적게 할 것인가?"** 가 진짜 질문이어야 합니다.
