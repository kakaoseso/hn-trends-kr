---
title: "Z3 Theorem Prover 실전 도입기: 우리가 Constraint Solver를 알아야 하는 이유"
description: "Z3 Theorem Prover 실전 도입기: 우리가 Constraint Solver를 알아야 하는 이유"
pubDate: "2026-04-18T15:21:28Z"
---

최근 Hacker News에서 흥미로운 아티클을 하나 발견했다. "Many Hard Leetcode Problems are Easy Constraint Problems"라는 아이디어에서 출발해 Rust로 Z3 바인딩을 다뤄본 "A Dumb Introduction to Z3"라는 글이다. 15년 넘게 현업에서 구르다 보면, 스케줄링, 리소스 할당, 레이아웃 계산 같은 문제를 풀기 위해 수백 줄의 휴리스틱이나 Dynamic Programming 코드를 짜는 주니어들을 자주 보게 된다. 솔직히 말해서, 이런 문제의 80%는 Constraint Solver를 쓰면 10줄 컷이다.

오늘은 이 아티클과 HN 커뮤니티의 반응을 바탕으로, Senior Engineer 관점에서 Z3 같은 SMT Solver를 실무에 어떻게 적용할 수 있을지, 그리고 왜 우리가 이런 도구를 알아야 하는지 파헤쳐보자.

## Constraint Solver란 무엇인가?

간단히 말해, 우리가 규칙을 정의하면 도구가 알아서 해답을 찾아주는 라이브러리다. 내부적으로는 SMT-LIB2라는 언어를 사용하며, 우리가 작성한 코드는 이 언어로 번역되어 Solver 엔진으로 들어간다.

아티클에서 소개된 Z3는 Microsoft Research에서 만든 아주 강력하고 성숙한 SMT Solver다. 여기서 알아둬야 할 Z3만의 독특한 용어가 있다.

- **Sort:** Z3 세계에서의 Type을 의미한다.
- **Constant:** 우리가 흔히 아는 상수가 아니라, Solver가 문제를 풀기 위해 조작하는 Variable 변수에 가깝다.

## Rust와 Z3: 우아한 바인딩

원문 작성자는 C++ 대신 Rust 바인딩을 선택했다. HN 댓글란에서는 "Z3 본체가 C++인데 왜 굳이 Rust를 쓰냐"는 태클이 있었지만, 나는 작성자의 선택에 100% 동의한다.

과거에 C++ API로 Z3를 다뤄본 적이 있는데, Z3 내부에 자체적인 Garbage Collection 메커니즘이 있어서 C++의 메모리 관리 모델과 충돌하는 경우가 잦았다. 오히려 Rust나 Python 바인딩을 거치는 것이 정신 건강에 훨씬 이롭다. Rust 바인딩은 Operator Overloading을 적극적으로 활용해 수학 공식을 코드에 그대로 옮길 수 있는 훌륭한 Ergonomics 환경을 제공한다.

다음은 아주 단순한 방정식 `x + 4 = 7`을 푸는 코드다.

```rust
use z3::{Solver, ast::Int};

fn main() {
    let solver = Solver::new();
    let x = Int::new_const("x");
    solver.assert((&x + 4).eq(7));
    
    _ = solver.check();
    let model = solver.get_model().unwrap();
    println!("{model:?}"); // x -> 3
}
```

## 최적화 문제와 함정: Coin Change

단순한 방정식 풀이보다 실무에서 더 자주 마주치는 것은 최적화 문제다. 아티클에서는 1, 5, 10 단위의 동전으로 37을 만드는 최소 동전 개수를 구하는 문제를 다룬다. Z3의 `Optimize` 객체를 사용하면 된다.

하지만 여기서 초보자들이 흔히 겪는 함정이 등장한다. 제약 조건을 단순히 총합과 최소 개수로만 주면, Z3는 "37개의 1원짜리 동전과 0개의 5원, 0개의 10원"이라는 기괴한 답을 내놓거나, 아예 음수 개수의 동전을 사용해버린다.

왜 그럴까? Z3의 `Int` Sort는 프로그래밍 언어의 부호 없는 정수가 아니라 수학적인 정수 전체를 의미하기 때문이다. 따라서 동전의 개수가 0 이상이라는 제약 조건을 명시적으로 추가해야 한다.

```rust
opt.assert(&c1.ge(0));
opt.assert(&c5.ge(0));
opt.assert(&c10.ge(0));
```

이런 사소한 제약 조건 누락이 프로덕션 환경에서는 치명적인 버그로 이어진다. Solver가 멍청한 게 아니라, 우리가 문제를 정확히 모델링하지 못한 탓이다.

## 블랙박스 논쟁과 도구의 가치

HN 댓글란에서 한 유저가 뼈 있는 비판을 남겼다. "문제를 블랙박스에 넣고 숫자만 얻어낼 뿐, 아무것도 배우지 못한다."

솔직히 이 의견에는 전혀 동의할 수 없다. 우리가 RDBMS를 사용할 때 B-Tree가 어떻게 밸런싱되는지 매번 신경 쓰면서 SQL을 작성하는가? 아니다. 도구를 사용해 문제를 선언적으로 해결하는 방법을 배우는 것 자체가 훌륭한 엔지니어링 스킬이다. Sudoku 풀이나 UI 레이아웃 계산 규칙을 Z3 제약 조건으로 모델링하는 과정에서 우리는 문제의 본질을 더 명확하게 이해하게 된다.

## Z3 vs CVC5: 실무에서의 선택

HN 스레드에서 또 하나 주목할 만한 논의는 Z3와 CVC5의 비교였다. 실무에서 Software Verification 업무를 주로 하는 엔지니어들의 의견을 종합해보면 다음과 같다.

- **Z3:** Term Simplification 기능이 압도적으로 우수하다. 복잡한 조건을 단순화해야 하는 검증 작업에서는 여전히 원탑이다.
- **CVC5:** Proof Generation 영역에서 버그가 적고 안정적이다. Lean이나 Coq 같은 Proof Assistant와 통합할 때는 CVC5가 더 선호된다.

규모가 큰 문제를 풀 때는 하나에만 의존하지 않고, 두 Solver를 모두 찔러볼 수 있도록 Verification 툴의 아키텍처를 유연하게 가져가는 것이 요즘의 트렌드다.

## 결론: 그래서 실무에 쓸 만한가?

결론부터 말하자면, 무조건 알아둬야 할 무기 중 하나다.

모든 로직을 Z3로 짤 필요는 없지만, 복잡한 스케줄링 로직, 권한 검증, 리소스 할당 등 수많은 Edge Case 처리가 필요한 도메인 로직을 구현할 때 Constraint Solver는 빛을 발한다. 며칠 밤을 새워가며 짜야 할 엣지 케이스 처리 코드를 단 몇 십 줄의 제약 조건 선언으로 대체할 수 있다.

물론 Z3가 마법의 지팡이는 아니다. 비선형 방정식에 취약하고, 외부 API를 찔러서 값을 가져오는 식의 동적인 작업은 불가능하다. 하지만 문제의 제약 조건을 수학적으로 모델링하는 사고방식은 Senior Engineer로 성장하기 위해 반드시 갖춰야 할 역량이다. 이번 주말에는 직접 Z3 바인딩을 설치하고 사내의 골칫거리 로직 하나를 Solver로 우아하게 풀어보는 것은 어떨까.

**References:**
- Original Article: https://ar-ms.me/thoughts/a-gentle-introduction-to-z3/
- Hacker News Thread: https://news.ycombinator.com/item?id=47760648
