---
title: "정적 타입 체킹 없는 Borrow Checker: 미친 아이디어인가, 새로운 패러다임인가?"
description: "정적 타입 체킹 없는 Borrow Checker: 미친 아이디어인가, 새로운 패러다임인가?"
pubDate: "2026-04-23T07:03:44Z"
---

Rust의 Borrow Checker는 메모리 안전성을 보장하는 훌륭한 도구지만, 특유의 빡빡한 정적 타입 시스템 때문에 학습 곡선이 악명 높다. 그렇다면 만약 이 **Borrow Checking 로직을 런타임으로 끌고 와서 동적 타입 언어에 적용한다면** 어떨까? 

처음 이 아티클의 제목을 봤을 때 내 머릿속에 든 생각은 "대체 왜 그런 짓을?" 이었다. 정적 타입의 컴파일 타임 최적화를 포기하고 굳이 런타임 오버헤드를 지불하면서까지 소유권(Ownership) 모델을 가져갈 이유가 있을까? 

하지만 Jamie Brandon이 작성한 이 실험적인 장난감 언어(Toy language)의 내부를 뜯어보니, 단순한 기행이 아니라 언어 디자인의 아주 흥미로운 틈새를 찌르고 있었다. 오늘은 이 언어가 어떻게 정적 타입 없이 Borrow Checking을 구현했는지, 그리고 이것이 실무 레벨에서 어떤 의미를 가지는지 시니어 엔지니어의 관점에서 파헤쳐보자.

## 왜 동적 언어에 Borrow Checker가 필요한가?

이 프로젝트의 배경을 이해하려면 저자가 만들고 있는 언어인 Zest의 철학을 알아야 한다. 저자는 Julia나 Zig처럼 동적 타입과 정적 타입을 넘나드는 시스템을 구상 중이다. 

대부분의 코드는 정적 타입으로 컴파일되어 성능을 내지만, REPL, 핫 리로딩(Live Code Reloading), 런타임 메타프로그래밍이 필요한 영역(Glue code)에서는 동적 타입의 유연함이 필요하다. 문제는 여기서 **Mutable Value Semantics** 를 어떻게 유지하느냐다.

기존의 동적 언어들은 이 문제를 두 가지 방식으로 우회했다.
- **GC 또는 무거운 레퍼런스 카운팅(RC):** 성능 오버헤드가 크고, 캐시 친화적인 스택 할당이나 Interior Pointer를 쓰기 어렵다.
- **정적 타입 시스템에 의존:** 동적 언어라는 전제 자체를 위반한다.

저자는 이 딜레마를 타파하기 위해 **스택 기반의 가벼운 런타임 Borrow Checker** 를 고안해냈다.

## 어떻게 동작하는가: 스택과 4-State Ref-Count

이 시스템의 핵심은 레퍼런스 카운트(Ref-count)를 힙(Heap)이 아닌 **스택(Stack)** 에 유지한다는 점이다. 스레드 간에 공유되지 않으므로 Atomic 연산이 필요 없고, 캐시 히트율도 높다.

코드를 보면 세 가지 방식의 참조를 지원한다.
- **Move (`^`):** 값을 이동시키고 원본을 파괴한다.
- **Borrow (`!`):** Rust의 `&mut`처럼 값을 빌려오며, 이 동안 다른 참조는 불가능하다.
- **Share (`&`):** Rust의 `&`처럼 읽기 전용으로 여러 곳에서 공유한다.

![Box reference](https://www.scattered-thoughts.net/writing/borrow-checking-without-type-checking/box.webp)

가장 흥미로운 부분은 이 소유권 규칙을 런타임에 검사하는 로직이다. 저자는 변수마다 레퍼런스 카운트를 두는데, 이 카운트 값의 범위로 상태를 기가 막히게 표현했다.

```zig
const Count = std.math.IntFittingRange(-stack_size, stack_size);
const available = 0;
const moved = std.math.minInt(Count);

fn isMoved(ref_count: RefCount) bool {
    return ref_count.count == moved;
}
fn canMove(ref_count: *RefCount) bool {
    return ref_count.count == available;
}
fn canBorrow(ref_count: *RefCount) bool {
    return ref_count.count == available;
}
fn canShare(ref_count: *RefCount) bool {
    return ref_count.count >= available;
}
```

- **음수 최솟값:** 값이 Move 됨 (사용 불가)
- **음수:** Borrow 된 횟수 (오직 1개의 Borrow만 허용되므로 보통 -1)
- **0:** 사용 가능 (Available)
- **양수:** Share 된 횟수

이 구조 덕분에 런타임에 값의 소유권 상태를 확인하는 비용이 **단일 정수 비교(Single integer comparison)** 하나로 끝난다. 꽤나 우아한 엔지니어링이다.

### 참조의 추적: Lender와 Owner

단순히 카운트만 세는 것으로는 복잡한 라이프타임 문제를 해결할 수 없다. 참조가 또 다른 참조를 낳는 Reborrowing 상황을 처리하기 위해, 모든 참조 객체는 16바이트 크기로 다음 세 가지 정보를 담는다.

- **Lease:** 참조의 종류 (Owned, Borrowed, Shared)
- **Lender:** 이 값을 빌려준 직계 부모 변수 (스코프 종료 시 카운트를 깎을 대상)
- **Owner:** 이 값의 최초 소유자 (이 값보다 수명이 짧은 변수를 할당하려 할 때 에러를 뱉기 위한 기준)

## 엔지니어의 시선: 훌륭한 해킹, 그러나 실용성은?

이 아티클을 읽으며 나는 저자의 창의적인 접근에 감탄했다. 특히 정적 타입 영역과 동적 타입 영역 사이의 경계를 넘나들 때, 스택 복사(`with_new_stack`)를 통해 레퍼런스 카운트 오염을 막는 아이디어는 실무적으로도 꽤 견고한 설계다.

하지만 내가 CTO로서 이 언어를 프로덕션에 도입하겠냐고 묻는다면, 대답은 **"아니오"** 다.

가장 큰 문제는 **개발자 경험(DX)** 이다. Rust의 Borrow Checker가 그나마 쓸만한 이유는 컴파일러가 뒤에서 엄청난 마법을 부려주기 때문이다. Deref Coercion, Autoborrow 같은 기능 덕분에 개발자는 `*`나 `&`를 매번 타이핑하지 않아도 된다. 

하지만 이 장난감 언어는 동적 타입이기 때문에 컴파일러가 미리 타입을 추론해서 이런 편의를 제공할 수 없다. 저자 본인도 "Fiddly(번거롭다)"라고 표현했듯, `a^**[index*]!` 같은 괴랄한 문법이 튀어나오게 된다. 명시적인 것은 좋지만, 이 정도의 인지 부하를 런타임 에러를 잡기 위해 견뎌야 한다면 차라리 처음부터 정적 타입을 쓰는 게 낫다.

## Hacker News의 반응과 타입 이론가들의 반발

HN 스레드를 보면 예상대로 타입 이론가(Type Theorists)들의 태클이 이어졌다.

> "Dynamic typing is no typing. The correct term is 'untyped' or 'unityped'." (동적 타입은 타입이 없는 거다. 정확한 용어는 무타입 혹은 단일타입이다.)

컴퓨터 과학의 관점에서 보면 맞는 말이다. 하지만 실무를 하는 엔지니어 입장에서는 다소 피곤한 논쟁이다. 중요한 건 용어가 아니라 **"왜 런타임 비용을 지불하면서 이 짓을 하느냐"** 에 대한 저자의 대답이다.

저자는 명확하게 선을 긋는다. 핫패스(Hot path)는 어차피 정적 컴파일을 할 것이고, 이 동적 Borrow Checking은 REPL이나 라이브 리로딩 같은 **메타프로그래밍 글루 코드** 를 위한 옵트인(Opt-in) 기능이라는 것이다. 이 관점에서 보면 16바이트의 레퍼런스 오버헤드는 충분히 지불할 만한 비용이다.

## 결론: 언어 디자인의 훌륭한 레퍼런스

이 프로젝트는 당장 실무에 쓸 수 있는 기술은 아니다. 하지만 **"소유권 모델(Ownership)은 반드시 정적 컴파일러의 전유물이어야 하는가?"** 라는 질문에 대한 훌륭한 PoC(Proof of Concept)다.

Val(현재 Hylo)이나 Swift가 제안하는 소유권 모델, 혹은 C#의 `ref struct` 같은 최신 언어 스펙의 진화에 관심이 있는 엔지니어라면 이 코드를 한 번쯤 뜯어볼 가치가 있다. 런타임과 컴파일 타임, 유연성과 안전성 사이의 트레이드오프를 이토록 적나라하게 보여주는 예제는 흔치 않기 때문이다.

**References**
- 원문: [Borrow-checking without type-checking](https://www.scattered-thoughts.net/writing/borrow-checking-without-type-checking/)
- Hacker News 토론: [HN Thread](https://news.ycombinator.com/item?id=47871817)
