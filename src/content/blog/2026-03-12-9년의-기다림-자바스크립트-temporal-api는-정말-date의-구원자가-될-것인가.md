---
title: "9년의 기다림, 자바스크립트 Temporal API는 정말 Date의 구원자가 될 것인가?"
description: "9년의 기다림, 자바스크립트 Temporal API는 정말 Date의 구원자가 될 것인가?"
pubDate: "2026-03-12T06:35:35Z"
---

지난 주말 새벽 3시, 서머타임(DST) 전환 버그로 PagerDuty 알람을 받아본 적이 있는가? 글로벌 서비스를 운영하며 프론트엔드와 백엔드를 오가는 타임존 이슈로 디버깅 지옥에 빠져본 시니어 엔지니어라면, 자바스크립트의 내장 `Date` 객체가 얼마나 끔찍한 레거시인지 굳이 설명할 필요가 없을 것이다.

최근 Bloomberg 엔지니어링 블로그에 [Temporal: The 9-Year Journey to Fix Time in JavaScript](https://bloomberg.github.io/js-blog/post/temporal/)라는 글이 올라왔고, Hacker News에서도 뜨거운 논쟁이 벌어졌다. 장장 9년의 스펙 논의 끝에 드디어 Stage 4(ES2026 표준)에 도달한 Temporal API. 과연 이것이 우리의 야근을 줄여줄 은탄환이 될 수 있을지, 아니면 또 다른 무거운 추상화 레이어에 불과할지 엔지니어의 시각에서 깊게 파헤쳐보자.

## 10일 만에 만들어진 30년짜리 기술 부채

1995년, 브렌던 아이크(Brendan Eich)가 10일 만에 자바스크립트(당시 Mocha)를 만들 때 시간 압박 속에서 내린 실용적인, 그러나 치명적인 결정이 하나 있었다. 바로 당시 잘 나가던 Java의 `java.util.Date` 코드를 그대로 C로 포팅한 것이다. 당시 내부 철학이 **MILLJ** (Make It Look Like Java)였다고 하니 이해는 가지만, 우리는 이 결정 때문에 30년 가까이 고통받았다.



기존 `Date` 객체의 문제점은 명확하다.

- **Mutability (가변성):** `d.setDate(d.getDate() + 1)`을 호출하면 원본 객체가 변경된다. 순수 함수를 지향하는 모던 자바스크립트 생태계에서 이는 재앙에 가깝다.
- **Inconsistent Arithmetic:** `setMonth`를 호출할 때 0-indexed month의 함정이나, 1월 31일에 한 달을 더하면 3월 2일이 되어버리는 암묵적 오버플로우는 수많은 결제 시스템 버그를 양산했다.
- **Ambiguous Parsing:** ISO 8601과 유사한 문자열을 넣었을 때 브라우저마다 로컬 타임, UTC, 혹은 `RangeError`를 뱉어내는 파편화가 존재했다.

이러한 문제를 해결하기 위해 Moment.js, date-fns, Luxon 같은 라이브러리들이 등장했지만, 이는 또 다른 문제를 낳았다. 바로 거대한 Bundle Size다.



Tree-shaking이 아무리 발전해도, 사용자가 어떤 타임존 데이터를 필요로 할지 빌드 타임에 알 수 없기 때문에 결국 수 MB에 달하는 tzdata(Time Zone Database)를 클라이언트에 통째로 내려보내야 했다. 이는 성능 최적화 관점에서 절대 용납할 수 없는 타협이었다.

## Temporal API 딥다이브: 시간의 본질을 분리하다

Temporal은 단순히 `Date`에 유틸리티 메서드 몇 개를 붙인 것이 아니다. 시간(Time)이라는 도메인을 완전히 재설계했다. Java가 Joda-Time의 철학을 받아들여 JSR 310(Java 8 `java.time`)으로 진화했듯, 자바스크립트도 드디어 시간의 본질을 타입으로 분리하기 시작했다.

### Temporal.Instant: 기계의 시간

`Instant`는 타임존이나 달력의 개념이 없는, 1970년 1월 1일 자정(UTC)부터 흐른 절대적인 시간이다. 기존 `Date`와 유사하지만 밀리초가 아닌 나노초(Nanoseconds) 정밀도를 가진다.

```javascript
const instant = Temporal.Instant.from("2026-02-25T15:15:00Z");
console.log(instant.toString()); 
// "2026-02-25T15:15:00Z"
```

개인적으로 백엔드 DB에 타임스탬프를 저장하거나, 분산 시스템 간에 이벤트를 전달할 때는 무조건 `Instant` 형태(혹은 나노초 정수)를 사용해야 한다고 생각한다. 시스템 바운더리를 넘나들 때 타임존 컨텍스트가 묻어있으면 언젠가 반드시 사고가 난다.

### Temporal.ZonedDateTime: 인간의 시간

프론트엔드에서 유저에게 시간을 보여주거나, 특정 지역의 비즈니스 로직(예: 런던 주식 시장 개장 시간)을 처리할 때는 `ZonedDateTime`이 핵심이다.

```javascript
// 런던 서머타임 시작: 2026-03-29 01:00 -> 02:00
const zdt = Temporal.ZonedDateTime.from(
  "2026-03-29T00:30:00+00:00[Europe/London]"
);

const plus1h = zdt.add({ hours: 1 });
console.log(plus1h.toString());
// "2026-03-29T02:30:00+01:00[Europe/London]" 
// (01:30은 존재하지 않으므로 자동으로 02:30으로 스킵됨)
```

위 코드를 보라. 서머타임 전환 시기에 1시간을 더했을 때, 존재하지 않는 시간을 건너뛰고 정확히 02:30을 반환한다. 기존 `Date`로 이 로직을 짠다고 생각해보라. 끔찍하지 않은가?

### Temporal.PlainDate: Wall Time의 도입

단순히 '2026년 3월 11일'이라는 정보만 필요할 때가 있다. 생일이나 신용카드 만료일 같은 데이터가 그렇다. 이때 타임존이 개입하면 한국에서 입력한 생일이 미국 서버에서 파싱될 때 하루 전날로 밀려버리는 고전적인 버그가 발생한다. `PlainDate`는 이러한 Wall Time(벽시계 시간)을 완벽하게 표현한다.

## Hacker News의 뜨거운 감자: 직렬화(Serialization)와 UX

Hacker News 스레드에서 가장 격렬하게 논의된 주제는 다름 아닌 Serialization 문제였다.

한 유저는 다음과 같이 비판했다.
> "클라이언트와 서버 간에 데이터를 주고받을 때, 나는 데이터와 로직이 엄격히 분리된 Plain JSON을 선호한다. Temporal 객체는 내부에 메서드를 가진 클래스 인스턴스이기 때문에 `JSON.parse`를 거치면 프로토타입이 날아가서 `.subtract()` 같은 메서드를 호출할 수 없다. 차라리 `date-fns`처럼 순수 함수에 데이터 객체를 넘기는 방식이 낫지 않은가?"

이 지적은 실무적으로 매우 날카롭다. tRPC나 REST API 바운더리에서 데이터를 받을 때, 우리는 종종 DTO를 매핑하는 과정을 극도로 귀찮아한다. `JSON.parse` 직후에 이것이 일반 객체인지 Temporal 인스턴스인지 타입 시스템(TypeScript)은 속을지언정 런타임은 속지 않기 때문이다.

하지만 나는 **Temporal 팀의 설계 결정에 전적으로 동의** 한다.

시간 데이터는 컨텍스트(타임존, 캘린더 시스템 등)가 생명이다. `date-fns` 같은 'Bag of Data + Free Functions' 접근 방식은 개발자가 실수로 타임존이 다른 객체를 함수에 밀어넣었을 때 런타임에서 이를 방어해주지 못한다. 객체지향적인 바인딩을 통해 타입 시스템이 `PlainDate`와 `ZonedDateTime`이 섞이는 것을 원천 차단하는 것이 훨씬 안전하다.

바운더리에서 `Temporal.Instant.from(jsonDate)`를 호출하는 보일러플레이트가 귀찮은가? 물론 귀찮다. 하지만 이 약간의 수고로움은 글로벌 서비스에서 타임존 엣지 케이스로 인해 발생하는 크리티컬 버그를 막아주는 매우 값싼 보험이다.

## 엔진 구현의 혁신: temporal_rs

이번 Temporal 스펙 구현 과정에서 엔지니어링 적으로 가장 흥미로웠던 부분은 브라우저 엔진들의 협업 방식이다.

Temporal은 ECMAScript 역사상 가장 거대한 스펙 추가다. Test262(공식 테스트 스위트) 기준으로 무려 4,500개의 테스트를 통과해야 한다. 이를 각 브라우저 벤더(V8, SpiderMonkey, JavaScriptCore)가 밑바닥부터 따로 구현하는 것은 엄청난 리소스 낭비다.



구글 Internationalization 팀과 Boa 엔진 팀은 이를 해결하기 위해 `temporal_rs`라는 Rust 기반의 공통 라이브러리를 만들었다. 서로 다른 철학과 아키텍처를 가진 자바스크립트 엔진들이 표준 스펙 구현을 위해 Rust 코어를 공유한다는 것은 웹 플랫폼 생태계에 시사하는 바가 크다. 이는 향후 복잡한 Web API들이 어떻게 구현되어야 하는지 보여주는 훌륭한 레퍼런스 아키텍처다.

## 결론: 그래서 실무에 도입할 것인가?

Temporal은 현재 Firefox v139, Chrome/Edge v144에서 지원되며, Node.js v26에도 탑재될 예정이다. TypeScript 6.0 Beta에서도 이미 지원을 시작했다.

솔직히 말해, Temporal API는 기존 `Date`보다 훨씬 Verbose(장황)하다. 무언가를 하려면 명시적으로 타입을 선택해야 하고, 파싱과 포맷팅 규칙을 엄격하게 지켜야 한다. 하지만 시간 다루기(Time Management) 도메인에서 **장황함은 버그가 아니라 기능(Feature)이다.**

Temporal은 개발자에게 시간의 복잡성을 회피하지 말고 정면으로 마주할 것을 강제한다. 당신이 작성하는 코드가 절대적인 순간(Instant)인지, 특정 지역의 달력 시간(ZonedDateTime)인지, 아니면 단순한 벽시계 시간(PlainDate)인지 명확히 정의해야만 코드가 컴파일된다.

거의 30년이 걸렸다. 마침내 자바스크립트는 프로덕션 레벨에서 신뢰하고 사용할 수 있는 모던 시간 API를 갖게 되었다. 기존의 수많은 유틸리티 함수와 라이브러리들을 걷어내고 Temporal로 마이그레이션하는 작업은 분명 고통스럽겠지만, 그 결과물은 훨씬 견고하고 우아할 것이다.

이제 더 이상 서머타임 버그로 새벽에 깨는 일은 없기를 바란다.

---
- **참고 문서:** [Temporal: The 9-Year Journey to Fix Time in JavaScript](https://bloomberg.github.io/js-blog/post/temporal/)
- **커뮤니티 반응:** [Hacker News Discussion](https://news.ycombinator.com/item?id=47336989)
