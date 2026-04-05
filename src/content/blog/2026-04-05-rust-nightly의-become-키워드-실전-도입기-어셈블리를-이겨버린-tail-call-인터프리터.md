---
title: "Rust nightly의 become 키워드 실전 도입기: 어셈블리를 이겨버린 Tail-call 인터프리터"
description: "Rust nightly의 become 키워드 실전 도입기: 어셈블리를 이겨버린 Tail-call 인터프리터"
pubDate: "2026-04-05T20:30:50Z"
---

15년 넘게 시스템 엔지니어링을 하면서 수많은 VM과 인터프리터를 작성해 보았습니다. 직렬화(Serialization) 프로토콜, 커스텀 DSL, 혹은 에뮬레이터까지 그 목적은 다양하지만, 결국 우리가 도달하는 종착지는 항상 비슷합니다. 처음에는 거대한 `match` 루프로 시작했다가, 성능의 한계에 부딪혀 C나 어셈블리로 넘어가곤 하죠.

솔직히 말해서, 성능을 쥐어짜기 위해 수천 줄의 어셈블리 코드를 직접 유지보수하는 것은 현대 소프트웨어 엔지니어링 관점에서 꽤나 고통스러운 일입니다. 그런데 최근 Matt Keeter가 작성한 [A tail-call interpreter in (nightly) Rust](https://www.mattkeeter.com/blog/2026-04-05-tailcall/) 아티클을 읽고 꽤 신선한 충격을 받았습니다. Rust nightly에 도입된 `become` 키워드를 사용해 순수 Rust 코드로 수제 ARM64 어셈블리를 이겨버렸기 때문입니다.

오늘은 이 흥미로운 사례를 바탕으로, Tail-call 최적화가 시스템 프로그래밍에서 어떤 의미를 가지는지, 그리고 왜 때로는 인터프리터가 직접 컴파일된 코드보다 빠를 수 있는지 깊게 파헤쳐 보겠습니다.

## 거대한 match 루프의 한계와 Token Threading

일반적으로 가벼운 VM(여기서는 Uxn CPU 에뮬레이터)을 구현할 때 가장 먼저 떠올리는 구조는 다음과 같습니다.

```rust
fn run(core: &mut Uxn, dev: &mut Device, mut pc: u16) -> u16 {
    loop {
        let op = core.next(&mut pc);
        let Some(next) = core.op(op, dev, pc) else {
            break pc;
        };
        pc = next;
    }
}
```

이 코드는 직관적이지만 하드웨어 관점에서는 최악에 가깝습니다. 메인 루프의 `match` 문은 CPU의 **Branch Predictor** 를 완전히 바보로 만듭니다. 다음 명령어가 무엇일지 예측할 수 없기 때문에 파이프라인 스톨(Pipeline stall)이 빈번하게 발생하죠.

이를 해결하기 위해 Matt는 이전에 **Token Threading** 기법을 사용해 ARM64 어셈블리로 엔진을 재작성했습니다. 각 명령어(Opcode) 구현체의 마지막에서 다음 명령어로 직접 점프(`br x10`)하는 방식입니다. 이 방식은 분기 예측기가 명령어 시퀀스를 학습하기 쉽게 만들어 성능을 40~50% 끌어올렸지만, 2000줄이 넘는 Unsafe한 어셈블리를 관리해야 하는 끔찍한 기술 부채를 낳았습니다.

## Rust의 구원투수: become 키워드

어셈블리의 성능을 유지하면서 Rust의 안전성을 취할 수는 없을까요? 여기서 등장하는 개념이 바로 **Tail-call Interpreter** 입니다. 

아이디어는 단순합니다.
1. VM의 상태(State)를 구조체가 아닌 함수의 인자(Arguments)로 전달하여 레지스터에 머물게 합니다.
2. 각 명령어 함수의 마지막에 다음 명령어 함수를 호출합니다.

하지만 일반적인 함수 호출은 스택 프레임을 지속적으로 쌓기 때문에 결국 Stack Overflow를 발생시킵니다. 이를 해결하기 위해 Rust nightly에 추가된 `become` 키워드를 사용합니다.

```rust
match core.inc::<FLAGS>(pc) {
    Some(pc) => {
        let op = core.next(&mut pc);
        // 호출자의 스택 프레임을 피호출자의 프레임으로 교체합니다.
        become TABLE.0[op as usize](
            core.stack.data,
            core.stack.index,
            core.ret.data,
            core.ret.index,
            core.dev,
            core.ram,
            pc,
            vdev,
        )
    }
    None => (core, pc)
}
```

`become`은 일반적인 함수 호출(`bl` 명령어)이 아닌 레지스터로의 직접 분기(`br` 명령어)를 강제합니다. 스택을 소모하지 않는 진정한 의미의 **Tail Call Optimization(TCO)** 이 이루어지는 것입니다.

## 벤치마크: 어셈블리를 이기다

결과는 놀라웠습니다. Apple M1(ARM64) 환경에서 Mandelbrot 렌더링 벤치마크를 돌렸을 때의 결과입니다.

- **VM (match loop):** 125ms
- **Assembly:** 87ms
- **Tailcall (become):** 76ms

순수 Rust 코드가 장인이 한땀 한땀 깎은 어셈블리보다 빠릅니다. 컴파일러가 UxnCore 객체의 생성과 해체 보일러플레이트를 완전히 최적화(Inline)하고, 레지스터 마스킹을 더 효율적으로 배치한 덕분입니다.

### 하지만 은환탄은 없다: x86과 WASM의 굴욕

이 대목에서 저는 "과연 x86에서도 똑같이 훌륭할까?"라는 의심이 들었습니다. 아니나 다를까, x86-64 환경에서는 Tailcall(175ms)이 어셈블리(168ms)를 이기지 못했습니다.

Matt가 분석한 생성된 어셈블리를 보면, LLVM의 x86 백엔드가 아직 이 패턴을 완벽히 소화하지 못하고 있습니다. 64비트 레지스터(rbp, r11)를 불필요하게 스택에 스필(Spill)하고 복원하는 끔찍한 코드 생성(Codegen)을 보여줍니다. WASM 환경에서는 상황이 더 심각해서 기존 VM 방식보다 최대 4.6배나 느려졌습니다. WASM의 스택 머신 구조와 JIT 컴파일러가 레지스터 위주의 Tail-call 패턴을 효율적인 기계어로 낮추지(Lowering) 못하기 때문입니다.

## Hacker News 커뮤니티의 시선: 왜 간접 호출이 더 빠른가?

이 아티클이 [Hacker News](https://news.ycombinator.com/item?id=47650312)에 올라왔을 때, 가장 흥미로웠던 토론 주제는 "어떻게 추상화 계층이 추가된 인터프리터가 직접 컴파일된 코드보다 빠를 수 있는가?"였습니다.

이 현상은 시스템 프로그래밍에서 주기적으로 재발견되는 진리입니다. 코드가 고도로 인라인화(Monomorphization)되면 실행 파일의 크기가 커지고, 이는 **Instruction Cache (i-cache)** 미스를 유발합니다. 반면, 잘 설계된 바이트코드 인터프리터는 명령어의 밀도가 매우 높아 CPU 캐시에 쏙 들어가며, 분기 예측기만 잘 달래주면 훨씬 높은 처리량(Throughput)을 보여줍니다.

실제로 Apple의 Swift 팀도 Value Witness(다형성 값 타입을 위한 vtable)를 처음에는 네이티브 코드로 생성하다가 바이너리 크기 문제에 짓눌려 결국 바이트코드 기반으로 전환했습니다. Protocol Buffers의 UPB(Table-driven parsing) 역시 동일한 철학을 공유합니다.

## 나의 결론 (Verdict)

Rust의 `become` 키워드는 아직 Nightly 기능이며, x86과 WASM에서의 코드 생성 품질을 볼 때 당장 프로덕션에 도입하기에는 무리가 있습니다. 특히 `extern "rust-preserve-none"` 같은 비표준 호출 규약(Calling convention)에 의존해야 하는 점도 찝찝합니다.

하지만 이 기술의 잠재력은 엄청납니다. 우리가 고성능 직렬화 라이브러리, 네트워크 패킷 파서, 혹은 커스텀 VM을 만들 때 더 이상 어셈블리라는 심연을 들여다보지 않아도 된다는 뜻이니까요. 컴파일러 백엔드가 조금만 더 성숙해진다면, 앞으로 Rust 생태계에서 고성능 State Machine을 작성하는 표준 패턴으로 자리 잡을 것이라 확신합니다.

---
**References:**
- Original Article: [A tail-call interpreter in (nightly) Rust](https://www.mattkeeter.com/blog/2026-04-05-tailcall/)
- Hacker News Thread: [News YCombinator](https://news.ycombinator.com/item?id=47650312)
