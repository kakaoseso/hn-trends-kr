---
title: "JSON의 한계와 새로운 대안: RX(REXC) 심층 분석 및 리뷰"
description: "JSON의 한계와 새로운 대안: RX(REXC) 심층 분석 및 리뷰"
pubDate: "2026-03-19T05:06:04Z"
---

최근 프로덕션 환경에서 수십 MB에 달하는 설정 파일이나 배포 매니페스트를 로드하다가 Node.js의 메인 스레드가 블로킹되는 현상을 겪어본 적이 있는가? JSON은 현대 웹의 공용어와 같지만, 데이터 크기가 커지면 파싱(Parse) 단계에서 엄청난 Memory와 CPU를 소모한다는 치명적인 단점이 있다.

이러한 문제를 해결하기 위해 등장한 수많은 대안 중, 최근 Hacker News에서 뜨거운 감자로 떠오른 프로젝트가 있다. 바로 nvm.sh와 luvit.io의 창시자인 Tim Caswell이 공개한 RX(REXC)다.

단순한 직렬화 라이브러리가 아니다. RX는 파싱 과정을 통째로 건너뛰고, 인코딩된 바이트 배열 위에서 직접 Random-access를 수행한다. 15년 넘게 백엔드와 인프라를 다루면서 Protobuf, FlatBuffers 등 수많은 바이너리 포맷을 써봤지만, RX가 취한 접근 방식은 상당히 독특하고 흥미롭다.

## RX는 어떻게 동작하는가?

RX가 압도적인 성능(단일 키 조회 시 23,000배 빠름)을 내는 핵심 비결은 크게 두 가지다. Zero-allocation과 Proxy 패턴이다.

일반적인 `JSON.parse`는 문자열을 읽어 메모리(Heap)에 거대한 자바스크립트 객체 트리를 생성한다. 반면 RX는 파싱을 하지 않는다. 대신 Flat byte buffer 위에 가벼운 Proxy 객체를 씌워 반환할 뿐이다.

```javascript
import { stringify, parse } from "@creationix/rx";

const payload = stringify({ users: ["alice", "bob"], version: 3 });
const data = parse(payload) as any;

console.log(data.users[0]); // "alice"
```

코드는 기존 JSON을 사용할 때와 동일해 보이지만, 내부 동작은 완전히 다르다. `data.users[0]`에 접근하는 그 순간, Proxy가 바이트 배열에서 해당 위치를 찾아 값을 디코딩한다. Garbage Collection이 추적해야 할 거대한 객체 트리가 애초에 존재하지 않으므로 Heap pressure가 거의 0에 수렴한다.

## 성능의 비밀: O(log n) 탐색과 Cursor API

RX는 객체나 배열의 크기가 일정 수준을 넘어가면 내부에 정렬된 인덱스를 자동으로 생성한다. 특정 키를 찾을 때 처음부터 순차 탐색을 하는 것이 아니라, 인코딩된 바이트 위에서 직접 이진 탐색(Binary search)을 수행한다.

극단적인 성능이 필요하거나 메모리 할당을 극도로 통제해야 하는 경우, Proxy조차 거치지 않고 Cursor API를 직접 사용할 수 있다.

```javascript
import { makeCursor, read, findKey } from "@creationix/rx";

const c = makeCursor(data); // 단 한 번의 객체 할당
if (findKey(c, container, "myKey")) {
  // c는 이제 myKey의 값 노드를 가리킴
}
```

이 패턴은 C++이나 Rust에서 FlatBuffers를 다룰 때의 감각과 매우 유사하다. 메모리 할당 없이 포인터(커서)만 이동시키며 데이터를 읽어내는 방식은 Throughput과 Latency 최적화에 있어 타의 추종을 불허한다.

## 논쟁의 중심: 정말 Drop-in Replacement인가?

Hacker News 스레드에서도 가장 열띤 토론이 벌어진 부분이다. 제작자는 이 라이브러리를 `JSON.stringify`와 `JSON.parse`의 Drop-in replacement라고 소개하지만, 실무를 다루는 엔지니어 입장에서 이는 다소 위험한 마케팅 용어다.

- **Read-only 제약:** RX의 `parse`가 반환하는 Proxy 객체는 읽기 전용이다. 기존 레거시 코드 중 파싱된 객체의 값을 수정(Mutation)하는 로직이 있다면 `TypeError`가 발생하며 애플리케이션이 크래시될 것이다. 100% 완벽한 호환성을 기대하고 코드를 교체하면 낭패를 볼 수 있다.
- **가독성 상실:** RX는 Base64 기반의 ASCII 텍스트로 인코딩된다. 한 유저는 커뮤니티에서 "공간 효율은 완전 바이너리보다 떨어지면서, 사람 눈으로 읽지도 못하는 Worst of both worlds"라고 강하게 비판했다.
- **스펙의 부재:** JSON이 천하통일을 할 수 있었던 이유는 json.org라는 명확하고 단순한 스펙이 있었기 때문이다. 현재 RX는 코드가 곧 스펙인 상태다. 공식적인 포맷 명세서(Specification)가 없다면 엔터프라이즈 환경에서 표준으로 채택하기엔 리스크가 크다.

하지만 RX를 옹호하는 의견도 충분히 일리가 있다. 100MB짜리 Minified JSON은 어차피 사람 눈으로 읽을 수 없다. 게다가 Protobuf 같은 완전한 바이너리 포맷과 달리, ASCII 텍스트 기반이라 환경변수나 클립보드 복사-붙여넣기 등 기존 텍스트 기반 파이프라인에 그대로 태울 수 있다는 점은 현업에서 엄청난 물류적 이점을 제공한다.

## 총평 (Verdict)

솔직히 말해, RX를 일반적인 마이크로서비스 간의 REST API 통신 포맷으로 사용하는 것은 추천하지 않는다. 그 영역은 이미 JSON과 gRPC가 각자의 생태계를 확고히 구축하고 있다.

하지만 Edge Computing 환경(Cloudflare Workers 등)에서 수십 MB의 읽기 전용 라우팅 테이블이나 설정 파일을 메모리 제약 하에 로드해야 한다면 어떨까? 혹은 SQLite를 띄우기엔 오버헤드가 부담스럽지만, 거대한 데이터셋에서 일부 데이터만 빠르게 쿼리해야 하는 상황이라면? 이럴 때 RX는 완벽한 해결책이 될 수 있다.

RX는 모든 곳에 쓰일 마법의 탄환은 아니지만, 메모리와 파싱 속도가 극단적인 병목을 일으키는 특정 구간을 우회하는 데 있어서는 매우 날카롭고 우아한 메스 역할을 할 것이다.

## References
- **Original Article:** https://github.com/creationix/rx
- **Hacker News Thread:** https://news.ycombinator.com/item?id=47432915
