---
title: "GPU에서 Rust std::thread를 돌린다고? VectorWare의 과감한 시도와 그 이면"
description: "GPU에서 Rust std::thread를 돌린다고? VectorWare의 과감한 시도와 그 이면"
pubDate: "2026-04-14T04:56:01Z"
---



GPU 프로그래밍은 15년 전이나 지금이나 여전히 고통스럽다. CUDA C/C++를 다루면서 겪는 메모리 버그와 디버깅 지옥은 시니어 엔지니어들에게도 기피 대상 1호다. 메모리 안전성을 보장하는 Rust가 GPU 생태계의 구원자가 될 수 있을까 기대했지만, 현실은 녹록지 않았다.

최근 VectorWare라는 스타트업이 GPU 위에서 Rust의 `std::thread`를 구동하는 데 성공했다는 글을 발표했다. 처음 이 소식을 접했을 때 내 머릿속을 스친 생각은 "이게 무슨 끔찍한 혼종인가?"였다. GPU를 단순히 코어가 엄청나게 많은 CPU처럼 취급하려는 순진한 발상처럼 보였기 때문이다. 하지만 그들의 구현 방식을 뜯어보니, 생각보다 훨씬 영리하고 실용적인 타협점을 찾았다는 것을 알 수 있었다.

### 왜 GPU에서 Rust를 쓰기가 그토록 어려웠나?

CPU와 GPU는 실행 모델 자체가 다르다. CPU는 단일 스레드에서 시작해 필요에 따라 명시적으로 동시성을 제어하지만, GPU는 수천 개의 인스턴스가 암시적으로 병렬 실행된다.

이 차이 때문에 기존 Rust 코드를 GPU 커널로 작성하면 다음과 같은 처참한 꼴이 된다.

```rust
use core::arch::nvptx::*;
pub unsafe extern "ptx-kernel" fn scale(data: *mut f32) {
    let i = (_block_idx_x() * _block_dim_x() + _thread_idx_x()) as usize;
    *data.add(i) *= 2.0;
}
```

`unsafe` 블록과 raw pointer가 난무한다. 수천 개의 GPU 인스턴스가 동일한 포인터를 동시에 참조하기 때문에, Rust의 자랑인 소유권 모델과 Borrow checker가 무용지물이 되어버린다. 사실상 컴파일러의 보호를 전혀 받지 못하는 FFI 경계나 다름없다.

### VectorWare의 해결책: Warp를 Thread로 취급하기

이 문제를 해결하기 위해 VectorWare는 `std::thread`를 GPU에 구현했다. 여기서 핵심은 Rust의 스레드를 개별 GPU 스레드(lane)가 아닌 **Warp** 단위에 매핑했다는 점이다.

이게 왜 중요한 발견일까? 만약 `std::thread`를 개별 lane에 매핑했다면 심각한 성능 저하를 피할 수 없었을 것이다. 같은 Warp 내의 lane들이 서로 다른 분기를 타게 되면 하드웨어는 이를 직렬화하여 실행한다. 이를 Divergence라고 부르는데, GPU 성능을 깎아먹는 주범이다.

VectorWare는 다음과 같은 방식으로 이를 우회했다.

- **Warp 0:** 커널이 시작되면 Warp 0만 활성화되어 `main` 함수를 실행하고 나머지 Warp는 대기 상태로 둔다.
- **Spawn:** `thread::spawn()`이 호출되면 대기 중이던 다른 Warp를 깨워 해당 클로저를 실행하게 한다.
- **Join:** 부모 Warp는 자식 Warp가 끝날 때까지 블로킹된다.

결과적으로 우리가 아는 평범한 Rust 코드가 GPU에서 그대로 돌아간다.

```rust
use std::thread;
fn main() {
    let a = thread::spawn(|| {
        let mut sum = 0u64;
        for i in 0..1000 {
            sum += i;
        }
        sum
    });
    let sum = a.join().unwrap();
    println!("sum: {sum}");
}
```

이 방식의 가장 큰 장점은 Divergence가 원천적으로 차단된다는 것이다. 하나의 스레드 내에 있는 모든 lane은 항상 동일한 코드를 실행하기 때문이다. 게다가 Rust의 라이프타임과 Borrow checker가 CPU에서처럼 완벽하게 작동한다.

### 개인적인 시선: 혁신인가, 낭비인가?

솔직히 말하자면, 나는 이 접근법에 대해 기대반 우려반이다.

Warp 단위로 스레드를 할당하는 것은 프로그래밍 편의성을 극대화하지만, 하드웨어 활용도 측면에서는 심각한 낭비를 초래할 수 있다. 32개의 lane을 가진 Warp에 단순한 스칼라 연산을 던져주면, 1개의 lane만 일하고 31개의 lane은 놀게 된다. 결국 성능을 제대로 뽑아내려면 클로저 내부에서 `warp_shuffle_idx` 같은 Warp 레벨의 intrinsics를 명시적으로 사용해 데이터를 병렬로 처리해야 한다.

"GPU 코드를 CPU 코드처럼 짜게 해줄게"라고 말하지만, 최상의 성능을 내려면 결국 개발자가 GPU의 하드웨어 특성을 깊이 이해하고 있어야 한다는 모순이 발생한다.

### Hacker News의 반응과 창업자의 등판

Hacker News에서도 이 글을 두고 격렬한 토론이 벌어졌다.

가장 많은 공감을 받은 의견은 역시 "이건 GPU를 느린 CPU로 만드는 것 아니냐"는 비판이었다. GPU의 본질적인 병렬성을 무시한 채 CPU의 패러다임을 강요하면, 애초에 GPU를 사용하는 이유가 퇴색된다는 것이다.

이에 대해 VectorWare의 창업자가 직접 등판해 남긴 코멘트가 꽤 인상적이었다.

- **문제의 본질:** GPU 프로그래머가 턱없이 부족한 이유는 GPU 도구들이 너무 기괴하기 때문이다.
- **목표:** 기존 바이너리를 그대로 돌리는 것이 아니라, 풍부한 기존 Rust 라이브러리 생태계를 GPU 위에서 활용하는 것이 목적이다.

추가로 댓글란에서는 Warp, Wavefront, Subgroup 등 벤더마다 제각각인 용어의 파편화에 대한 푸념도 이어졌는데, 10년 넘게 그래픽스 API를 다뤄온 나로서는 격하게 공감할 수밖에 없었다.

### 결론: 프로덕션 레벨인가?

이 기술이 당장 CUDA 기반의 딥러닝 프레임워크나 극단적으로 최적화된 행렬 연산을 대체할까? 절대 아니다. 그런 작업들은 여전히 전통적인 SIMT 모델이 압도적으로 유리하다.

하지만 이 시도는 단순한 장난감을 넘어선 유의미한 실험이다. I/O가 복잡하게 얽혀 있거나, 비정형적인 상태 관리가 필요한 새로운 유형의 GPU 애플리케이션을 구축할 때 이 모델은 강력한 무기가 될 수 있다.

GPU는 점차 범용 컴퓨팅 유닛으로 진화하고 있다. VectorWare의 시도는 그 진화 과정에서 소프트웨어 스택이 어떻게 하드웨어의 복잡성을 추상화해야 하는지 보여주는 훌륭한 사례다. 앞으로 이들이 어떻게 Underutilization 문제를 해결하고 최적화해 나갈지 관심 있게 지켜볼 생각이다.

### References
- Original Article: https://www.vectorware.com/blog/threads-on-gpu/
- Hacker News Thread: https://news.ycombinator.com/item?id=47708457
