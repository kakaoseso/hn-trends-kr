---
title: "TypeScript를 네이티브로 컴파일한다고? Perry가 숨기고 있는 런타임의 진실"
description: "TypeScript를 네이티브로 컴파일한다고? Perry가 숨기고 있는 런타임의 진실"
pubDate: "2026-05-30T08:14:03Z"
---

TypeScript로 짠 코드를 V8이나 Node.js 없이 단일 네이티브 바이너리로 뽑아낼 수 있다면 어떨까? 15년 넘게 백엔드와 인프라를 굴려본 엔지니어라면 한 번쯤 꿈꿔봤을 법한 이야기다.

최근 Hacker News를 뜨겁게 달군 Perry라는 프로젝트가 정확히 이 포인트를 자극했다. SWC로 파싱하고 LLVM으로 최적화해 2~5MB짜리 실행 파일을 만든다는 것이다. "One Codebase. Every Platform. Native Performance." 마케팅 구호는 완벽하다. 하지만 뚜껑을 열어보면 과연 그럴까?

## 어떻게 동작한다고 주장하는가

Perry의 아키텍처 자체는 흥미롭다. 중간에 JavaScript로 변환하는 과정을 생략하고, TypeScript AST를 직접 LLVM IR로 낮춰버린다(lowering). 이론상으로는 JIT 컴파일러의 오버헤드를 AOT(Ahead-of-Time) 방식으로 날려버리는 셈이다.

```typescript
// hello.ts
const greeting = "Hello, World!";
console.log(greeting);
```

이 코드를 `perry compile main.ts`로 빌드하면 2MB 남짓한 단일 바이너리가 튀어나온다. 메모리 오버헤드도 최소화되고, Startup Time은 1ms에 불과하다고 주장한다.

## "No Runtime"이라는 달콤한 거짓말

하지만 시스템 엔지니어링에 공짜는 없다. 공식 홈페이지에서는 런타임이 없다고 강조하지만, 이는 마케팅 용어에 가깝다. TypeScript는 본질적으로 동적 타입 언어다. C나 Rust처럼 메모리를 수동으로 관리하지 않는다. 즉, 바이너리 안에 반드시 Garbage Collector(GC)가 포함되어야만 한다.

더 치명적인 것은 동적 타입의 처리 방식이다. HN 쓰레드에서도 지적되었듯, Perry는 TypeScript의 동적 특성을 유지하기 위해 JavaScriptCore와 동일한 **NaN-boxing** 기법을 사용한다. 정적 분석이 불가능한 `any` 타입이나 복잡한 객체 할당에서는 여전히 막대한 런타임 오버헤드가 발생한다.

실제로 Express 같은 평범한 Node.js 생태계의 패키지를 가져다 쓰려고 하면 어떻게 될까?

```typescript
import * as express from 'express';
const app = express();
```

이 코드를 컴파일하면 Perry는 에러를 뱉으며 `perry-jsruntime`(QuickJS 기반)을 활성화하라고 요구한다. 즉, 순수 TS로 작성된 코드가 아니면 결국 내장된 JS 엔진을 돌려야 한다는 뜻이다. 과일 바구니인 줄 알고 다가갔는데 플라스틱 모형을 발견한 기분이라는 한 유저의 비유가 뼈를 때린다.

## AI가 짜낸 수백만 줄의 코드, 감당할 수 있는가?

솔직히 말해, 내가 이 프로젝트를 프로덕션 레벨에서 극도로 경계하는 가장 큰 이유는 따로 있다. 프로젝트의 커밋 로그를 보면 시간에 15개씩 끝없는 AI 커밋이 이어지고 있다. 웹사이트의 카피라이팅부터 내부 로직까지, 이른바 Vibe coding의 결정체다.

수백만 줄의 AI 생성 Rust 코드로 이루어진 컴파일러와 GC를 우리가 신뢰할 수 있을까? 만약 프로덕션 환경에서 원인을 알 수 없는 Memory Corruption이나 세그폴트가 발생한다면, 이를 디버깅할 주체는 누구인가? AI 모델이 해결해 줄 때까지 프롬프트를 깎고 있어야 할지도 모른다.

## Monomorphization과 정적 분석의 한계

기술적으로 인상 깊은 부분도 분명 있다. 제네릭을 Rust처럼 단형화(Monomorphization) 처리하려는 시도나, 인터페이스 메서드 호출을 정적 분석해 vtable 대신 직접 점프(direct jump)로 변환하려는 최적화는 칭찬할 만하다.

- **Monomorphization:** 타입이 확정된 경우 정적 분석을 통해 C 수준의 성능을 낸다.
- **Fallback:** 하지만 정적 분석이 실패하면 결국 런타임 vtable 폴백으로 빠지게 된다.

TypeScript는 근본적으로 너무 동적이다. 언어의 스펙을 완전히 바꾸지 않는 이상, AOT 컴파일러가 런타임에 결정되는 모든 데이터를 컴파일 타임에 해결할 수는 없다.

## 결론: 프로덕션 레디인가, 장난감인가?

결론적으로 Perry는 TypeScript 생태계가 나아가야 할 방향을 보여주는 흥미로운 PoC(Proof of Concept)다. JS 엔진 없이 TS를 구동하려는 접근 자체는 가치가 있다. 하지만 당장 여러분의 회사 레포지토리에 도입할 수 있는 물건은 절대 아니다.

이 프로젝트는 AI가 혼자서 LLVM 백엔드, GC, 크로스 플랫폼 런타임을 뚝딱 만들어낼 수 있는 시대가 왔음을 보여주는 쇼케이스에 가깝다. 혁신적이지만, 동시에 기술적 부채가 얼마나 기괴한 형태로 쌓일 수 있는지 보여주는 경고장이기도 하다. 토이 프로젝트용으로는 훌륭하지만, 진짜 프로덕션 서버를 띄워야 한다면 여전히 Node.js나 Bun, 혹은 차라리 Rust나 Go를 쓰는 것을 강력히 권장한다.

## References
- Original Article: https://www.perryts.com/
- Hacker News Thread: https://news.ycombinator.com/item?id=48332151
