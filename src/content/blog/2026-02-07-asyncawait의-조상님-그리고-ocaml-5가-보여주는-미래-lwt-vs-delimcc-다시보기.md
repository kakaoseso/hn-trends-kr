---
title: "Async/Await의 조상님, 그리고 OCaml 5가 보여주는 미래: Lwt vs Delimcc 다시보기"
description: "Async/Await의 조상님, 그리고 OCaml 5가 보여주는 미래: Lwt vs Delimcc 다시보기"
pubDate: "2026-02-07T10:53:18Z"
---

안녕하세요. 오늘은 조금 '고고학적'이면서도, 현재의 백엔드 엔지니어링 트렌드를 관통하는 흥미로운 주제를 가져왔습니다.

우리가 흔히 쓰는 Node.js의 Promise나 Python의 Asyncio, 그리고 Go의 Goroutine. 이 모든 비동기(Asynchronous) 모델의 근본적인 고민은 **"어떻게 하면 Non-blocking I/O를 다루면서도 코드는 동기(Synchronous) 방식처럼 깔끔하게 짤 수 있을까?"** 입니다.

최근 Hacker News에 2011년 작성된 OCaml의 스레딩 모델 비교 글이 다시 올라왔습니다. 무려 13년 전 글이지만, 당시의 고민이 지금 OCaml 5의 'Effect Handlers'로 어떻게 진화했는지 살펴보면 꽤나 깊은 기술적 통찰을 얻을 수 있습니다. 시니어 엔지니어로서 이 흐름을 짚어보는 건 꽤 의미 있는 일입니다.

### 1. Monad의 지옥 vs. Direct Style의 꿈

2011년 당시 OCaml 생태계의 비동기 표준은 `Lwt`라는 라이브러리였습니다. 쉽게 말해 **Monadic Threading** 입니다. JavaScript로 치면 Promise 체이닝과 비슷하고, Haskell의 IO Monad와도 닮았죠.

`Lwt`의 코드는 대략 이렇습니다.

```ocaml
(* Lwt 스타일: Monad Bind의 향연 *)
let x = sleep 5 in
let y = bind x (fun () -> print_endline "awake!") in
run y
```

엔지니어로서 솔직히 말하자면, 이런 스타일은 **'Function Coloring'** 문제를 야기합니다. 비동기 함수는 일반 함수와 섞일 수 없고, 모든 호출 체인이 `bind`나 `return`으로 오염되죠. 당시 Anil Madhavapeddy(MirageOS 창시자)는 이 문제의 대안으로 `delimcc`(Delimited Continuations)를 주목했습니다.

**Delimited Continuation** 은 프로그램의 실행 흐름(Stack)을 캡처해서 나중에 재개할 수 있게 해주는 기법입니다. 이걸 사용하면 비동기 코드도 마치 동기 코드처럼 짤 수 있습니다. (지금의 Fiber나 Coroutine의 원형이라고 보시면 됩니다.)

### 2. 성능 벤치마크: 힙(Heap) 할당이냐, 스택(Stack) 복사냐

이 글의 핵심은 `Lwt`(Monad)와 `delimcc`(Continuation)의 성능 비교입니다. 결과가 아주 흥미롭습니다.

#### Round 1: 얕은 호출 스택 + 잦은 Context Switch
`Lwt`가 더 빨랐습니다. `delimcc`는 Continuation을 캡처할 때마다 예외 처리 매커니즘을 사용하는데, 이게 오버헤드가 꽤 컸던 모양입니다. 반면 `Lwt`는 단순히 힙에 클로저(Closure)를 할당하는 방식이라, OCaml의 빠른 힙 할당 성능 덕을 봤죠.

#### Round 2: 깊은 재귀 호출 (Deep Recursion)
여기서 재미있는 현상이 발생합니다. 호출 스택이 깊어질수록 `Lwt`는 느려집니다. 왜냐고요? **모든 단계마다 `bind`를 위한 클로저를 생성해야 하니까요.** 반면 `delimcc`는 Yield가 발생하기 전까지는 그냥 Native Stack을 쓰는 일반 함수 호출입니다. 당연히 빠를 수밖에 없습니다.

하지만 `delimcc`에도 치명적인 약점이 있었습니다. **깊은 스택에서 Yield를 자주 하면**, 그 거대한 스택을 계속 복사하고 복구해야 하므로 성능이 $O(N^2)$로 나락을 갑니다. (Jake Donham이 지적했듯이 말이죠.)

### 3. Principal Engineer의 시선: "결국은 추상화 비용이다"

이 벤치마크를 보면서 제가 느낀 점은, **"공짜 점심은 없다"** 는 겁니다.

- **Lwt (Monad):** 런타임 스택 트릭 없이 순수 언어 레벨에서 구현 가능하지만, **Garbage Collector(GC)에 엄청난 부하** 를 줍니다. 수만 개의 I/O 요청이 들어오면 그만큼의 클로저 객체가 힙에 쌓이니까요.
- **Delimcc (Continuation):** 코드는 깔끔해지지만, 스택 스위칭 비용을 런타임이 얼마나 효율적으로 처리하느냐에 따라 성능이 천지차이입니다.

2011년 당시 Anil의 결론은 "성능 차이는 미미하니, JS 호환성이 좋은 `Lwt`를 쓰자"였습니다. 당시로서는 합리적인 선택이었죠. 하지만 10년이 지난 지금은 어떨까요?

### 4. 2025년의 반전: OCaml 5와 Effect Handlers

최근 HN 댓글에 등판한 원작자(Anil)의 코멘트가 이 글의 하이라이트입니다.

> "This is a blast from the past! ... I use Eio these days to do direct-style IO ... and there's nary a monad in sight!"

OCaml 5가 멀티코어와 함께 **Effect Handlers** 를 도입하면서 판도가 완전히 뒤집혔습니다. Effect Handler는 `delimcc`가 하려던 '스택 스위칭'을 언어 런타임 레벨에서 초고속으로 지원합니다. 이제 `Lwt` 같은 Monad 라이브러리 없이도, Go의 Goroutine처럼 **Direct Style** 로 코드를 짜면서 성능은 Non-blocking 수준으로 뽑아낼 수 있게 된 겁니다.

`Eio` 라이브러리를 사용한 HTTP 파서를 보면 Monad(`>>=`)가 싹 사라졌습니다. 코드는 동기식처럼 직관적인데, 내부적으로는 런타임이 알아서 스케줄링을 해줍니다.

### 5. 마치며: 기술의 나선형 발전

우리는 지난 10년간 비동기 처리를 위해 수많은 `Promise`, `Future`, `Observable` 래퍼(Wrapper)들을 만들어왔습니다. 하지만 결국 기술은 **"가장 자연스러운 코드(Direct Style)를 유지하면서, 하부 구조(Runtime)가 복잡성을 떠안는 방향"** 으로 회귀하고 있습니다.

Java의 Project Loom(Virtual Threads), OCaml 5의 Effects, 그리고 Go의 Goroutine이 증명하듯 말이죠.

만약 여러분이 아직도 "어떤 비동기 라이브러리가 더 빠를까?"를 고민하고 있다면, 이제는 시야를 넓혀 **"언어 런타임이 동시성을 어떻게 추상화하고 있는가?"** 를 봐야 할 때입니다. 2011년의 `delimcc`는 시대를 너무 앞서간 '해킹'이었지만, 2025년의 Effect Handler는 '표준'이 되었습니다.

**Verdict:** 레거시 코드나 JS 상호운용성이 필수라면 여전히 Promise/Monad 패턴이 유효합니다. 하지만 새로운 시스템을 설계한다면, 런타임 레벨의 Concurrency 모델(Fiber, Coroutine, Effects)을 적극 활용하세요. 코드는 사람이 읽기 위해 쓰는 것이니까요.

---

**References:**
- [Original Article: Delimited Continuations vs. Lwt for Threads](https://mirageos.org/blog/delimcc-vs-lwt)
- [Hacker News Discussion](https://news.ycombinator.com/item?id=46859267)
