---
title: "Verilog로 팩토리오 회로를 굽는다: v2f 컴파일러와 RISC-V 구현 딥다이브"
description: "Verilog로 팩토리오 회로를 굽는다: v2f 컴파일러와 RISC-V 구현 딥다이브"
pubDate: "2026-03-29T06:22:40Z"
---

가끔 해커뉴스(Hacker News)를 보다 보면, 엔지니어들의 순수한 광기(혹은 잉여력)에 압도당할 때가 있다. 15년 넘게 현업에서 구르면서 수많은 'X에서 Y 돌리기' 프로젝트(예: 임신 테스트기에서 둠 돌리기)를 봐왔지만, 이번에 내 눈길을 사로잡은 프로젝트는 그 궤를 달리한다. 바로 **v2f** (Verilog to Factorio) 프로젝트다.

단순히 게임 안에서 노가다로 논리 게이트를 이어 붙여 계산기를 만든 수준이 아니다. 이 툴은 실제 하드웨어 기술 언어인 Verilog를 입력으로 받아, 팩토리오(Factorio) 2.0 환경에서 동작하는 조합기(Combinator) 블루프린트 JSON으로 컴파일해버린다. 심지어 그 결과물로 완전하게 동작하는 **RV32IM RISC-V CPU** 를 구현해냈다.

오늘은 이 미친 프로젝트가 기술적으로 어떻게 동작하는지, 그리고 왜 이것이 단순한 장난감을 넘어선 훌륭한 엔지니어링인지 파헤쳐보자.

## Yosys를 활용한 진짜 RTL 합성

이 프로젝트가 범상치 않은 이유는 백엔드 파이프라인에 있다. 저자는 밑바닥부터 파서를 만든 것이 아니라, 실제 FPGA 및 ASIC 설계에 널리 쓰이는 오픈소스 합성 툴인 **Yosys** 를 적극 활용했다.

- **Front-end:** Verilog 코드를 Yosys가 읽어들여 RTL(Register-Transfer Level) 및 워드 레벨(word-level) 논리로 합성한다.
- **Mapping:** Yosys가 생성한 범용 논리 게이트들을 팩토리오의 조합기(Combinator) 특성에 맞게 매핑한다.
- **Back-end (v2f):** 매핑된 데이터를 바탕으로 Factorio 2.0에서 임포트할 수 있는 JSON 블루프린트 문자열을 생성한다.

사실 팩토리오의 조합기는 일종의 거대한 룩업 테이블(LUT)이나 다름없다. 이를 타겟 아키텍처로 삼아 EDA(Electronic Design Automation) 플로우를 구축했다는 점이 정말 놀랍다. 예전에 사내에서 커스텀 가속기용 사내 컴파일러를 만들 때 Yosys를 백엔드로 연동하며 고생했던 기억이 나는데, 이걸 브라우저(Dev Container) 환경과 Lua API로 깔끔하게 래핑한 저자의 실력에 박수를 보낸다.

## Lua API와 물리적 렌더링 (Physical Design Rendering)

단순히 Verilog를 변환하는 것에 그치지 않고, 저자는 Rust와 Lua를 이용해 코드로 직접 설계를 조작하고 시뮬레이션할 수 있는 API를 제공한다.

```bash
# 일반적인 사용 예시
$ v2f -i xyz.lua -o blueprint.json
```

특히 인상 깊었던 기능은 **물리적 설계 렌더링(Physical design rendering)** 이다. 팩토리오에 블루프린트를 임포트하기 전에도, SVG 포맷으로 회로의 배치와 라우팅 상태를 미리 시뮬레이션하고 확인할 수 있다.

이 기능은 디버깅 관점에서 엄청난 이점을 가진다. 실제 하드웨어 설계에서 Place & Route 이후의 타이밍 이슈나 배선 문제를 시각화하는 것은 매우 고통스러운 작업인데, 이 툴은 시뮬레이션 상태를 SVG에 주석으로 달아 전역적인 가시성을 제공한다.

## 팩토리오 안에서 돌아가는 RISC-V (RV32IM)

이 툴체인의 강력함을 증명하기 위해 저자는 Ultraembedded의 RV32IM 코어를 팩토리오에 올렸다.

![Image of the RV32IM core](https://github.com/ben-j-c/riscv_v2f_optimized/raw/7f76b0459d402eee0993eac75e75c2aa3ca6af48/doc/overview.png)

위 다이어그램은 원본 코어의 구조인데, 저자는 이를 그대로 가져오되 팩토리오 환경의 제약에 맞춰 곱셈과 나눗셈 로직을 더욱 효율적으로 재합성(synthesize)했다. 이런 타겟 기반의 최적화야말로 실력 있는 하드웨어 엔지니어의 기본 소양이다.

![System diagram](https://raw.githubusercontent.com/ben-j-c/riscv_v2f_optimized/refs/heads/master/top_v2f/doc/system_diagram.svg)

코어 주변에는 디스플레이 출력과 프로그래밍 가능한 RAM에 접근하기 위한 패브릭이 래핑되어 있다. 더 충격적인 것은, 이 위에서 돌아가는 프로그램을 작성하기 위해 **GNU Toolchain** (freestanding RV32IM 크로스 컴파일러)을 그대로 사용한다는 점이다.

![hello world](https://raw.githubusercontent.com/ben-j-c/riscv_v2f_optimized/refs/heads/master/top_v2f/doc/hello_world.png)

C언어로 `hello_world.c`를 짜고, GCC로 컴파일해서 나온 바이너리를 팩토리오 안의 RAM에 올려 실행한다? 이건 정말 경이로운 수준의 풀스택(Full-stack) 엔지니어링이다.

실제 게임 내에서 구현된 거대한 CPU의 모습은 아래와 같다.

![In game view](https://raw.githubusercontent.com/ben-j-c/riscv_v2f_optimized/refs/heads/master/top_v2f/doc/1000ft_view.png)

## 해커뉴스 반응과 나의 생각

해커뉴스 커뮤니티의 반응도 뜨겁다. 한 유저는 "정말 기발한 아이디어다. 아직 팩토리오를 해보진 않았지만 시도해봐야겠다"라고 남겼는데, 이에 대한 대댓글이 압권이다.

> "수면 시간과 여가 시간을 소중히 여긴다면 절대 하지 마세요. 농담 섞인 말이지만, 진심으로 엄청나게 중독성이 강합니다."

나 역시 동의한다. 팩토리오는 그 자체로 거대한 분산 시스템 설계 시뮬레이터와 같다. Latency 와 Throughput 의 밸런스를 맞추고, 병목(Bottleneck)을 찾아 해결하는 과정이 실제 소프트웨어 아키텍처나 하드웨어 설계와 놀랍도록 닮아있다.

**그래서, 이게 실용성이 있는가?**
당연히 프로덕션 레벨의 툴은 아니다. 하지만 **교육용 도구** 혹은 **컴퓨터 아키텍처 시각화 도구** 로서는 현존하는 수많은 상용 EDA 툴보다 훨씬 직관적이고 훌륭하다. 우리가 보통 터미널 창의 파형(Waveform) 뷰어로만 보던 레지스터의 상태 변화를, 게임 내의 시각적인 신호 흐름으로 직접 볼 수 있다는 것은 학생들과 주니어 엔지니어들에게 엄청난 영감을 줄 것이다.

가끔은 이런 순수한 호기심과 열정으로 만들어진 프로젝트가, 매일 JIRA 티켓을 쳐내며 잊고 지냈던 '엔지니어링의 본질적인 즐거움'을 일깨워주곤 한다. 주말에 시간을 내어 Dev Container를 띄우고 Lua 스크립트를 만지작거려볼 핑계가 생겼다. (물론, 팩토리오를 켜는 순간 주말이 삭제될 것은 각오해야겠지만.)

---

### References
- **GitHub Repo:** [ben-j-c/verilog2factorio](https://github.com/ben-j-c/verilog2factorio)
- **Hacker News Thread:** [https://news.ycombinator.com/item?id=47528853](https://news.ycombinator.com/item?id=47528853)
