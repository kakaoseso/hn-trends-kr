---
title: "PostgreSQL 안에서 돌아가는 6502 CPU: 관계형 데이터베이스로 에뮬레이터를 만드는 법"
description: "PostgreSQL 안에서 돌아가는 6502 CPU: 관계형 데이터베이스로 에뮬레이터를 만드는 법"
pubDate: "2026-04-14T10:12:32Z"
---

15년 넘게 백엔드와 인프라 엔지니어링을 하면서 수많은 '기상천외한' 토이 프로젝트들을 봐왔다. "Doom이 여기서도 돌아가네?" 같은 밈(meme)은 이제 지겨울 정도다. 하지만 가끔은 단순한 장난을 넘어 아키텍처 관점에서 꽤 흥미로운 질문을 던지는 프로젝트들이 있다.

최근 Hacker News에서 내 눈길을 끈 프로젝트가 바로 [pg_6502](https://github.com/lasect/pg_6502)다. 무려 MOS Technology 6502 8-bit 마이크로프로세서를 PostgreSQL 위에서 구현했다. 그것도 별도의 외부 C 플러그인 없이 '순수 SQL과 PL/pgSQL'만으로 말이다.

## 아키텍처: 데이터베이스가 CPU가 되는 법

폰 노이만 아키텍처의 핵심은 결국 상태(State)와 연산(Compute)이다. 이 프로젝트는 이 두 가지를 RDBMS의 기본 구성 요소로 1:1 매핑했다.

- **상태 저장소:** 레지스터와 메모리는 테이블이 된다.
- **연산 유닛:** Opcode 실행은 Stored Procedure가 담당한다.

구체적인 스키마를 보면 꽤 직관적이다.
`pg6502.cpu` 테이블은 단 하나의 행(row)을 가지며 A, X, Y, SP, PC 레지스터와 상태 플래그를 저장한다.
`pg6502.mem` 테이블은 64KB의 메모리를 나타내며, 각 바이트가 하나의 행으로 구성된다.

```sql
-- 대략적인 개념도
CREATE TABLE pg6502.cpu (
    a smallint,
    x smallint,
    y smallint,
    pc integer,
    sp smallint,
    flags jsonb
);
```

## Principal Engineer의 시선: 이것은 진정한 SQL인가?

솔직히 처음 저장소를 열어보고 `init/05_execute.sql` 코드를 읽었을 때 내 반응은 "음..." 이었다. 개발자의 엄청난 끈기와 집념(Perseverance)에는 기립 박수를 보낸다. 6502의 모든 Opcode를 PL/pgSQL로 구현하고 Klaus 6502 Functional Test까지 통과했다는 건 정말 대단한 일이다.

하지만 기술적인 관점에서 묻고 싶다. **이게 정말 SQL의 강점을 살린 아키텍처일까?**

결론부터 말하자면 아니다. 이건 사실상 데이터베이스의 테이블을 단순히 전역 변수(Global Variable)처럼 사용하고, PL/pgSQL이라는 절차적 언어(Procedural Language)로 무한 루프를 돌리는 것에 불과하다. 관계형 데이터베이스(RDBMS)의 진정한 강점인 집합 기반 연산(Set-based operation)은 전혀 활용되지 않았다.

## 만약 진짜 '관계형 CPU'를 만든다면?

Hacker News의 한 유저(anon)가 스레드에서 정확히 내가 생각했던 아쉬움을 지적했다. 만약 정말 SQL과 RDBMS의 특성을 극한으로 끌어올린 CPU 에뮬레이터를 만들고 싶었다면 접근 방식을 완전히 바꿔야 했다.

나라면 이렇게 설계했을 것이다.

1. **Decode ROM 테이블화:** 6502의 Decode ROM 자체를 하나의 거대한 매핑 테이블로 만든다.
2. **Trigger 기반 상태 전이:** 메인 루프가 이 ROM 테이블을 Lookup 하여 출력 비트를 가져오고, 각 마이크로 오퍼레이션(micro-op)을 Trigger로 처리한다.
3. **논리 게이트 모델링:** 더 극단적으로 간다면, 오리지널 CPU의 모든 논리 게이트(Logic Gate)와 와이어(Wire)를 개별 테이블로 모델링한다. 신호가 이동할 때마다 Trigger 연쇄 반응을 일으켜 상태를 변경하는 것이다.

이렇게 구현했다면 PostgreSQL의 Trigger 엔진과 동시성 제어(Concurrency Control)가 수천 개의 마이크로 오퍼레이션을 병렬로 처리할 때 어디까지 버틸 수 있는지, Race Condition을 어떻게 회피하는지 테스트할 수 있는 훌륭한 스트레스 테스트 도구가 되었을 것이다.

## 결론: 잉여력이 만들어낸 훌륭한 사고 실험

당연하지만 이 프로젝트를 프로덕션 레벨의 기술로 진지하게 평가하는 것은 넌센스다. HN 스레드에서 누군가 "Microsoft 6502 BASIC 위에서 Postgres를 돌려보는 건 어때?"라고 농담한 것처럼, 이건 순수한 지적 유희다.

하지만 이런 잉여력 넘치는 토이 프로젝트는 가치가 있다. 우리가 매일 사용하는 도구인 PostgreSQL이 얼마나 유연한지 보여주고, 동시에 '절차적 사고'와 '관계형 사고'의 차이가 무엇인지 극명하게 드러내기 때문이다. 데이터를 다루는 엔지니어라면 한 번쯤 코드를 열어보고 "나라면 이걸 어떻게 쿼리로 풀었을까?" 고민해 볼 만한 재미있는 주제다.

---
- **Original Article:** [pg_6502 on GitHub](https://github.com/lasect/pg_6502)
- **Hacker News Thread:** [HN Comments](https://news.ycombinator.com/item?id=47761723)
