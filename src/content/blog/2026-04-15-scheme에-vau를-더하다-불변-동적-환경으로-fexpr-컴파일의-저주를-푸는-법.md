---
title: "Scheme에 vau를 더하다: 불변 동적 환경으로 fexpr 컴파일의 저주를 푸는 법"
description: "Scheme에 vau를 더하다: 불변 동적 환경으로 fexpr 컴파일의 저주를 푸는 법"
pubDate: "2026-04-15T09:44:54Z"
---

Lisp나 Scheme 같은 언어를 깊게 파본 엔지니어라면, 매크로(Macro)와 함수(Procedure) 사이의 견고한 벽을 느껴본 적이 있을 것이다. 우리는 런타임과 컴파일 타임을 분리하고, 코드를 생성하는 코드(Macro)를 작성하기 위해 별도의 메타 언어(DSL)를 배운다. `syntax-rules`나 `syntax-case` 같은 것들 말이다.

솔직히 말해서, 나는 이 두 세계가 분리되어 있다는 사실이 항상 불만이었다. 트리 구조를 변환하는 작업이라는 본질은 같은데, 왜 우리는 두 개의 다른 메타 언어 시스템을 유지해야 하는가?

최근 커뮤니티에서 흥미로운 프로젝트를 하나 발견했다. 바로 Chez Scheme에 Kernel 언어의 핵심인 `vau`를 구현한 **Seed** 라는 프로젝트다. 단순한 장난감 프로젝트로 치부하기엔, 이들이 fexpr의 오랜 난제를 해결한 방식이 엔지니어링 적으로 꽤나 우아하다.

### Fexpr의 저주와 컴파일러의 딜레마

Kernel 언어의 창시자 John Nathan Shutt는 2010년 논문을 통해 매크로와 함수를 통합하는 궁극의 추상화로 `vau`를 제시했다. `vau`는 쉽게 말해 인자를 평가(evaluate)하지 않은 상태로, 호출자의 동적 환경(Dynamic Environment)과 함께 받아오는 연산자(Operative)다.

아이디어는 환상적이지만, 치명적인 문제가 있었다. 1998년 Mitchell Wand는 논문을 통해 fexpr이 등식 추론(equational reasoning)을 불가능하게 만든다고 증명했다.

컴파일러 개발자 입장에서 생각해보자. 만약 어떤 연산자가 호출자의 동적 환경을 런타임에 마음대로 뒤집어엎고 변수를 재할당(`set!`)할 수 있다면? 컴파일러는 특정 변수가 호출 시점에 어떤 값을 가질지 정적으로 예측할 수 없다. Inlining도, Substitution도 불가능해진다. 결국 컴파일은 포기하고 느려터진 인터프리터로 돌아가야 한다. 이것이 수십 년간 fexpr이 주류로 올라서지 못한 이유다.

### 우아하고 외과적인 해결책: 불변성(Immutability)

Seed 프로젝트는 이 문제를 아주 심플하고 외과적인(surgical) 방법으로 해결했다. `vau` 연산자에게 동적 환경을 전달하되, 이를 **읽기 전용(Read-only)** 으로 강제한 것이다.

연산자는 여전히 호출자의 바인딩을 들여다보고 인자를 언제 어떻게 평가할지 결정할 수 있다. 하지만 환경 자체를 변이(mutate)시킬 수는 없다. 즉, 동적 환경 내에서 새로운 변수를 `define` 하거나 `set!` 할 수 없다.

단 하나의 제약, 즉 환경의 불변성을 보장했을 뿐인데 컴파일러는 다시 코드를 정적으로 추론할 수 있는 지식을 얻었다. 함수 호출을 인라인화하고, Chez Scheme의 `syntax-case`가 만들어내는 것과 동일한 수준의 고성능 네이티브 코드를 뱉어낼 수 있게 된 것이다. 과거 우리가 상태 관리의 복잡성을 줄이기 위해 Redux나 불변 객체 패턴을 도입했던 것과 정확히 같은 철학이다.

### 두 세계의 통합: 코드로 보는 차이

`syntax-rules`와 `vau`를 사용해 Short-circuit OR(`myor`)를 구현한 코드를 비교해보면 이 접근법의 진가를 알 수 있다.

```scheme
;;; 1. syntax-rules (R5RS / R7RS)
;;; 매크로 언어: ellipsis(...)를 사용하는 패턴 템플릿.
(define-syntax myor
  (syntax-rules ()
    ((_) #f)
    ((_ e) e)
    ((_ e1 e2 ...)
     (let ((t e1))
       (if t t (myor e2 ...))))))
```

`syntax-rules`는 코드를 생성하기 위해 템플릿 언어를 강제한다. 런타임에 중복 평가를 막으려면 매크로 확장기(expander)가 `let` 바인딩을 생성하도록 지시해야 한다.

반면 `vau`를 사용한 코드를 보자.

```scheme
;;; 3. vau (Kernel / Seed)
;;; 템플릿 언어도, phase 분리도 없다.
(define myor
  (vau args env
    (if (null? args) #f
        (let ((v (eval (car args) env)))
          (if v v
              (eval (cons myor (cdr args)) env))))))
```

놀랍도록 직관적이다. 매크로 확장을 위한 별도의 DSL이 필요 없다. 그저 인자(`args`)와 환경(`env`)을 받아, 평범한 Scheme 코드로 언제 무엇을 평가할지 직접 제어할 뿐이다. 프로그래머는 코드를 작성하는 프로그램이 아니라, 그냥 '프로그램'을 작성하면 된다.

### 벤치마크와 커뮤니티의 반응

가장 놀라운 점은 성능이다. N-Queens, Collatz, Abacus 등 다양한 벤치마크에서 Seed(`seedink`)는 기존 Chez Scheme(`syntax-case` 사용)과 거의 동일하거나 미세하게 더 빠른 실행 시간(Wall Clock)을 보여주었다. 불변 환경을 통한 정적 분석이 실제로 통했다는 증거다.

물론 Reddit과 Lobsters 등 커뮤니티에서 지적된 한계점들도 존재한다.

- **환경 확장 문제:** `define-record-type`처럼 호출자의 환경에 새로운 바인딩을 주입해야 하는 경우, 기존의 가변 환경이 없으면 처리가 까다롭다. Seed는 이를 `call-with-values`를 통해 람다 파라미터로 불변하게 주입하는 방식으로 우회했다.
- **Applicative Dispatch 버그:** 람다로 바인딩된 파라미터에 대해 operative/applicative 디스패치가 제대로 컴파일되지 않는 버그가 보고되었다. 저자도 이를 인지하고 있다.

### 총평 (Verdict)

그래서 이 Seed 프로젝트가 당장 프로덕션에 도입할 만한 기술인가? 당연히 아니다. 저자 스스로도 밝혔듯 이는 Claude의 도움을 받아 작성된 탐색적 성격의 PoC(Proof of Concept)다.

하지만 **메타프로그래밍의 복잡성을 어떻게 줄일 것인가** 라는 질문에 대해, 별도의 매크로 DSL을 덧붙이는 대신 핵심 평가 모델(`vau`)을 수정하고 불변성(Immutability)이라는 현대적인 제약을 더해 컴파일러의 최적화 길을 열어준 접근은 박수받아 마땅하다.

복잡한 시스템을 설계하다 보면 종종 "더 강력한 기능"을 추가하려는 유혹에 빠진다. 하지만 때로는 이 프로젝트가 보여주듯, 기존의 강력한 기능(가변 동적 환경)에 **올바른 제약(Read-only)** 을 가하는 것이 전체 시스템을 훨씬 우아하고 빠르고 예측 가능하게 만든다. 컴파일러나 프레임워크를 설계하는 모든 시니어 엔지니어들이 곱씹어볼 만한 훌륭한 사례다.

---

**References:**
- Seed GitHub Repository: https://github.com/amirouche/seed
- Hacker News Thread: https://news.ycombinator.com/item?id=47709203
