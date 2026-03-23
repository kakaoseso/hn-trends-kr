---
title: "Raspberry Pi PIO vs BIO: I/O 코프로세서 설계의 CISC vs RISC 딜레마"
description: "Raspberry Pi PIO vs BIO: I/O 코프로세서 설계의 CISC vs RISC 딜레마"
pubDate: "2026-03-23T18:39:08Z"
---

최근 하드웨어와 임베디드 씬에서 가장 흥미로운 아키텍처적 시도 중 하나를 꼽으라면, 단연 Andrew "bunnie" Huang이 설계한 **BIO** (Bao I/O Coprocessor)를 들 수 있다. 

메인 CPU에서 Bitbanging으로 I/O를 제어해 본 엔지니어라면 누구나 공감할 것이다. 인터럽트 Jitter와 예측 불가능한 Latency는 실시간성이 중요한 시스템에서 끔찍한 악몽이다. 이 문제를 해결하기 위해 등장한 Raspberry Pi의 **PIO** 는 하드웨어 엔지니어들에게 그야말로 축복이자 흑마법 같은 존재였다. 하지만 완벽해 보이는 PIO에도 치명적인 단점이 존재했다. 

이 글에서는 Bunnie의 최신 포스트를 바탕으로, PIO의 한계점과 이를 RISC 아키텍처로 우아하게 풀어낸 BIO의 내부 구조를 딥다이브 해보려 한다. 개인적으로 이 설계는 하드웨어와 소프트웨어의 경계를 다루는 시니어 엔지니어들에게 엄청난 영감을 준다고 생각한다.

## PIO의 딜레마: 완벽한 타이밍, 무거운 비용

Raspberry Pi의 PIO는 4개의 State Machine으로 구성되며, 단 9개의 명령어만으로 동작한다. 언뜻 보면 굉장히 단순해 보이지만, 실상은 극단적인 **CISC** (Complex Instruction Set Computer) 아키텍처다.

명령어 하나가 단일 사이클 내에서 다음과 같은 작업들을 동시에 수행할 수 있다.
- 데이터 Shift 및 Masking
- FIFO 상태 확인 및 분기
- Program Counter 래핑 (Wrap-around)
- 핀 상태 변경 (Side-set)

Bunnie는 자신이 설계 중인 22nm 오픈소스 SoC(Baochip-1x)에 PIO를 도입하기 위해 이를 FPGA로 포팅했다. 그리고 경악스러운 결과를 마주했다.

![](https://bunniefoo.com/baochip/pio-utilization.png)

위 사진에서 마젠타 색상으로 표시된 영역이 PIO가 차지하는 공간이다. 고작 9개의 명령어를 처리하는 코어 4개가, 캐시를 제외한 VexRiscv CPU 코어 전체보다 더 많은 면적을 차지한 것이다. 심지어 타이밍 클로저(Timing Closure) 측면에서도 최악이었다. CPU 코어는 100MHz를 가볍게 달성했지만, PIO는 50MHz조차 맞추기 버거워했다.

원인은 명확하다. 명령어 하나에서 유연한 핀 매핑과 비트 조작을 동시에 지원하려면 거대한 **Barrel Shifter** 4개가 필요하다. FPGA의 LUT(Look-Up Table) 구조상, 이러한 거대한 멀티플렉서와 시프터 조합은 라우팅 리소스를 엄청나게 소모하며 크리티컬 패스를 길어지게 만든다. 

솔직히 나 역시 과거에 PIO 코드를 작성하며 그 기괴할 정도의 유연성에 감탄했지만, 이를 실리콘이나 FPGA로 직접 구현해야 하는 하드웨어 엔지니어 입장에서는 그야말로 재앙에 가까웠을 것이다.

## BIO의 탄생: RISC에 하드웨어 큐를 얹다

이 지점에서 Bunnie는 발상의 전환을 한다. "PIO의 CISC 방식을 버리고, 검증된 RISC 코어를 사용하면 어떨까?"

그는 Claire Xenia Wolf가 설계한 초소형 RISC-V 코어인 **PicoRV32** 를 기반으로 BIO를 설계했다. RV32E 모드를 사용하여 레지스터를 16개(r0-r15)로 줄이고, 남는 레지스터 공간(r16-r31)을 메모리가 아닌 **하드웨어 큐(Queue)와 동기화 프리미티브** 로 매핑해버렸다. 내가 이 아키텍처에서 가장 감탄한 부분이 바로 이 대목이다.

![](https://bunniefoo.com/baochip/bio-diagram.png)

Memory-Mapped I/O (MMIO)를 사용할 때 발생하는 버스 경합과 오버헤드를 피하기 위해, BIO는 CPU의 레지스터 자체를 블로킹 큐로 만들어버렸다. 

- **x16-x19 (FIFO):** 읽거나 쓸 때 FIFO가 비어있거나 꽉 차 있으면 CPU 실행이 즉시 Halt (대기) 상태에 빠진다.
- **x20 (Quantum):** 이 레지스터에 접근하면, 설정된 타이머 틱(Quantum)이나 외부 GPIO 이벤트가 발생할 때까지 CPU가 멈춘다.
- **x21-x26 (GPIO):** 데이터 핀을 위한 Clear-on-0 시맨틱을 적용해, 비트 인버전 없이 빠르게 데이터를 밀어 넣을 수 있도록 최적화했다.

### Snap to Quantum: 사이클 카운팅의 지옥에서 벗어나기

RISC 아키텍처의 가장 큰 약점은 명령어 실행 시간이 가변적일 수 있어 PIO처럼 정확한 Cycle-accurate 제어가 어렵다는 점이다. BIO는 이를 **Snap to Quantum** 이라는 개념으로 우아하게 해결했다.

![](https://bunniefoo.com/baochip/bio-quantum-halt.png)

루프 내에서 명령어 사이클을 완벽하게 계산하는 대신, 목표 시간보다 일찍 연산을 끝내놓고 `x20` 레지스터를 호출하여 다음 하드웨어 틱이 발생할 때까지 기다리는 방식이다. 이는 소프트웨어 엔지니어 관점에서 훨씬 유지보수하기 쉬운 멘탈 모델을 제공한다.

## C 컴파일러의 도입: 실용주의의 극치

PIO의 또 다른 진입 장벽은 난해한 커스텀 어셈블리 언어였다. BIO는 표준 RISC-V ISA를 사용하므로 기존 툴체인을 그대로 활용할 수 있다. Bunnie는 여기서 한 발 더 나아가, **Zig** 의 Clang 래퍼를 활용한 C 툴체인을 구축했다.

```c
// WS2812C LED 제어 코드의 일부
for (uint32_t i = 0; i < len; i++) {
    led = strip[i];
    for (uint32_t bit = 0; bit < 24; bit++) {
        if ((led & 0x800000) == 0) {
            // 2 hi
            set_gpio_pins(mask);
            wait_quantum();
            wait_quantum();
            // 5 lo
            clear_gpio_pins_n(antimask);
            wait_quantum();
            wait_quantum();
            wait_quantum();
            wait_quantum();
            wait_quantum();
        }
// ... 생략 ...
```

무거운 LLVM 백엔드를 새로 구축하는 대신, Python 스크립트로 Zig 툴체인을 다운로드하고 C 코드를 Rust 어셈블리 매크로로 변환하여 Xous OS 빌드 시스템에 통합했다. 시니어 엔지니어로서 이런 식의 실용적이고 유지보수 비용을 최소화하는 접근법은 언제 봐도 짜릿하다.

## Area vs Performance: 무엇을 선택할 것인가

![](https://bunniefoo.com/baochip/bio-pio-comparison.png)

동일한 FPGA에 올렸을 때, BIO(녹색)는 PIO(마젠타)의 절반 이하 면적을 차지한다. ASIC 공정에서는 PIO보다 4배 이상의 높은 클럭 속도(700MHz)를 달성했다. 

물론 트레이드오프는 존재한다. PicoRV32는 파이프라인이 없는 구조라 명령어 당 약 3사이클이 소모된다 (IPC가 낮음). 따라서 DVI 비디오 신호처럼 극단적인 속도의 비트뱅잉이 필요하다면 여전히 PIO가 우세하다. 하지만 SPI 통신, WS2812 제어, 복잡한 프로토콜 스택 처리 등 일반적인 임베디드 요구사항에서는 BIO의 압승이다. 4KB의 넉넉한 전용 명령 메모리를 통해 고정 소수점 연산 같은 상위 레벨 작업까지 오프로딩할 수 있기 때문이다.

## Hacker News의 반응과 나의 생각

Hacker News 스레드에서는 이 글을 두고 흥미로운 논쟁이 벌어졌다. 일부 유저들은 "PIO가 FPGA에 부적합한 것이 아니라, 오픈소스 구현체(fpga_pio)가 ASIC 수준으로 최적화되지 않았기 때문"이라며 Bunnie가 PIO를 부당하게 깎아내렸다고 지적했다.

하지만 Bunnie의 반론처럼, Barrel Shifter는 타겟 플랫폼이 무엇이든 본질적으로 비싼 하드웨어 블록이다. 게다가 PIO에는 폐쇄적인 특성과 특허 침해 리스크까지 존재한다.

다른 유저가 지적한 **Mental Model** 의 차이가 핵심을 찌른다. PIO는 완벽한 타이밍과 파형 중심(Timing-first)의 설계라면, BIO는 C 언어를 통한 명확한 로직 표현과 명시적인 하드웨어 동기화(Clarity-first)를 지향한다.

## 총평: 프로덕션 레벨에서의 효용성

결론적으로 나는 BIO의 접근 방식에 손을 들어주고 싶다. PIO는 분명 천재적인 발명품이지만, 현업에서 팀 단위로 코드를 유지보수하고 디버깅해야 하는 관점에서는 BIO의 RISC + C 툴체인 + 하드웨어 블로킹 레지스터 조합이 훨씬 합리적이다. 

특히 MMIO 대신 레지스터를 Queue로 맵핑하여 멀티 코어 간의 동기화를 하드웨어 레벨에서 강제한 설계는, 향후 다른 커스텀 SoC 설계에서도 널리 차용되어야 할 훌륭한 디자인 패턴이다. 

Raspberry Pi 재단이 PIO의 라이선스를 보수적으로 가져가는 상황에서, BIO와 같은 완벽한 오픈소스 대체재의 등장은 하드웨어 생태계에 큰 축복이 아닐 수 없다.

---

- **Reference:** [BIO: The Bao I/O Coprocessor](https://www.bunniestudios.com/blog/2026/bio-the-bao-i-o-coprocessor/)
- **Hacker News Discussion:** [Thread 47459363](https://news.ycombinator.com/item?id=47459363)
