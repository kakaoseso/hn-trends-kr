---
title: "QuickBEAM 리뷰: BEAM 위에서 돌아가는 JavaScript, 장난감인가 혁신인가"
description: "QuickBEAM 리뷰: BEAM 위에서 돌아가는 JavaScript, 장난감인가 혁신인가"
pubDate: "2026-03-29T20:22:36Z"
---

최근 Hacker News에서 꽤 도발적인 프로젝트를 하나 발견했다. 바로 Elixir/OTP 환경에서 JavaScript를 직접 실행하게 해주는 런타임, QuickBEAM이다. 백엔드 엔지니어로서 BEAM 생태계와 JS 생태계를 섞는 시도는 많이 보았지만, 대부분은 무거운 포트(Port) 통신이거나 별도의 마이크로서비스로 분리하는 식이었다. 하지만 QuickBEAM은 JS 런타임 자체를 OTP 프로세스 트리 안으로 끌고 들어왔다. 처음엔 "이게 무슨 끔찍한 혼종인가" 싶었지만, 아키텍처를 뜯어볼수록 감탄이 나왔다.

## JS 런타임이 GenServer라고?

가장 충격적이면서도 우아한 부분은 JS 런타임을 **GenServer** 로 취급한다는 점이다. Node.js 환경에서는 하나의 프로세스가 죽으면 전체 서버가 내려가는 끔찍한 경험을 종종 하게 된다. 하지만 QuickBEAM은 개별 JS 컨텍스트가 OTP Supervision tree 아래에 존재한다.

```elixir
{:ok, rt} = QuickBEAM.start()
{:ok, 3} = QuickBEAM.eval(rt, "1 + 2")
```

만약 JS 코드 내에서 치명적인 오류나 OOM(Out of Memory)이 발생해 런타임이 죽더라도, OTP 슈퍼바이저가 이를 즉시 새로운 컨텍스트로 재시작한다. 이는 Erlang의 "Let it crash" 철학을 JS 생태계에 그대로 이식한 것이다.

## 격리와 샌드박싱: 실무 도입의 핵심

Hacker News 댓글 쓰레드에서도 가장 뜨거운 주제는 단연 샌드박싱이었다. 신뢰할 수 없는 사용자의 코드를 서버에서 실행하는 것은 언제나 위험을 동반하기 때문이다. QuickBEAM은 두 가지 강력한 제어 장치를 제공한다.

- **memory_limit:** 컨텍스트 당 할당할 수 있는 메모리 제한 (예: 512KB). 이를 초과하면 프로세스가 죽는 것이 아니라 JS 예외가 발생하며 엘릭서 측에 에러를 반환한다.
- **max_reductions:** BEAM의 스케줄링 방식에서 영감을 받은 것으로, JS의 연산(Opcode) 횟수를 제한한다. 무한 루프에 빠지더라도 예산이 소진되면 실행을 강제로 인터럽트한다.

댓글에서 한 유저가 "JS 프로세스가 BEAM 스케줄러를 선점할 수 있는가?"라고 질문했는데, 작성자의 답변이 명쾌했다. JS는 BEAM 스케줄러 외부의 전용 OS 스레드에서 실행되며, 인터럽트 핸들러가 데드라인을 체크한다. 즉, 무거운 JS 연산이 Elixir 메인 시스템의 실시간성을 해치지 않는다.

## 생태계 통합: Node.js와 Webpack이 필요 없는 세계

또 하나 주목할 만한 점은 빌드 파이프라인의 단순화다. QuickBEAM은 내부적으로 Rust 기반의 OXC를 내장하고 있다. 덕분에 별도의 Node.js 설치나 Webpack 구성 없이도 런타임에 TypeScript 코드를 즉시 번들링하고 실행할 수 있다.

```elixir
{:ok, bundle} = QuickBEAM.JS.bundle(files)
```

게다가 브라우저 API의 구현 방식이 매우 실용적이다. 폴리필(Polyfill)로 떡칠한 것이 아니라, BEAM의 네이티브 모듈을 직접 연결했다.

- **fetch:** Erlang의 `:httpc` 백엔드 사용
- **WebSocket:** `:gun` 라이브러리 사용
- **DOM:** C 기반의 `lexbor` 사용

특히 Lexbor를 사용해 네이티브 DOM을 구현한 것은 SSR(Server-Side Rendering) 렌더링 속도에 엄청난 이점을 가져다줄 것이다.

## 아쉬운 점과 총평

솔직히 말해서 완벽한 것은 아니다. 예를 들어 `fetch` API의 백엔드로 `:httpc`를 사용한 점은 개인적으로 약간 우려스럽다. 실무에서 `:httpc`는 자잘한 버그나 한계가 많아 보통 `Req`나 `Finch`를 선호하기 때문이다. 또한 GC(Garbage Collection)가 레퍼런스 카운팅 기반으로 개별 컨텍스트마다 돈다고는 하지만, 수만 개의 커넥션이 열리는 LiveView 환경에서 ContextPool의 메모리 단편화가 어떻게 동작할지는 프로덕션 레벨의 검증이 더 필요해 보인다.

그럼에도 불구하고 QuickBEAM은 단순한 장난감을 넘어섰다. 기존 Elixir 서비스에서 npm 패키지의 비즈니스 로직(예: 마크다운 파싱, 특정 데이터 검증 로직 등)을 그대로 가져다 써야 할 때, 이보다 더 우아한 해결책은 당분간 찾기 힘들 것이다.

**결론:** 당장 메인스트림 프로덕션의 코어 로직을 대체하기엔 이르지만, 백오피스의 Rule Engine이나 LiveView의 SSR 보조 도구로는 지금 당장 도입을 검토해 볼 만한 훌륭한 기술이다.

---
- [원본 아티클 보기](https://github.com/elixir-volt/quickbeam)
- [Hacker News 토론 보기](https://news.ycombinator.com/item?id=47558094)
