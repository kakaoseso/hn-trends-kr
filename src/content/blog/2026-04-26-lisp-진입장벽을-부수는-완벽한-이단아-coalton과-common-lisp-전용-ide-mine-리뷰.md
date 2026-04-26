---
title: "Lisp 진입장벽을 부수는 완벽한 이단아: Coalton과 Common Lisp 전용 IDE 'mine' 리뷰"
description: "Lisp 진입장벽을 부수는 완벽한 이단아: Coalton과 Common Lisp 전용 IDE 'mine' 리뷰"
pubDate: "2026-04-26T20:07:35Z"
---

시니어 엔지니어라면 누구나 한 번쯤 "Lisp이 그렇게 좋다는데 나도 한번 써볼까?"라는 생각을 해본 적이 있을 것이다. 하지만 현실은 어떤가? Common Lisp을 찍어 먹어보려다 Emacs 단축키를 외우고, SLIME을 설정하고, ASDF와 Quicklisp의 늪에서 허우적대다 주말을 다 날리고 포기하는 것이 국룰이다. 2020년대에 이런 OOBE(Out of Box Experience)는 솔직히 끔찍하다.

이런 상황에서 Robert Smith가 내놓은 mine이라는 새로운 IDE는 꽤나 신선한 충격이다. Coalton과 Common Lisp만을 위해 바닥부터 만들어진 이 툴은, 우리가 당연하게 여겼던 '에디터 깎는 노인' 메타를 정면으로 부정한다.

## 범용성을 버리고 특수성을 취하다

mine은 단일 다운로드로 끝나는 독립형 애플리케이션이다. 설치 직후 파일을 열고 코드를 작성하면 즉시 REPL로 코드를 쏠 수 있다. Lisp 생태계의 핵심인 인터랙티브 개발, Hot-reloading, On-the-fly 디버깅이 그냥 '된다'.

내가 가장 흥미롭게 본 부분은 이 IDE의 철학이다. mine은 확장성이 없다. 플러그인도 없고, 커스터마이징도 거의 불가능하다. 텔레메트리도, 숨겨진 서버 연결도 없다. 오직 초보자와 Lisp 개발자만을 위한 '완성된 도구'를 지향한다.

- **Zero Config:** ASDF, Quicklisp, SLIME 설정이 내장되어 있어 환경 구성에 시간을 낭비할 필요가 없다.
- **Structural Editing:** Lisp 특유의 괄호 문법을 다루기 위한 구조적 편집 기능과 그 튜토리얼이 내장되어 있다.
- **Familiarity:** Emacs나 vi 에뮬레이션 따위는 없다. Ctrl+C, Ctrl+V 같은 지극히 일반적인 단축키를 사용한다.



## Borland Turbo 시절의 향수

Hacker News 스레드에서 한 유저가 "과거 QBASIC이나 Borland Turbo Pascal을 떠올리게 한다"고 언급했는데, 이 말에 100% 공감한다. 나 역시 예전 Turbo C++로 코딩을 처음 배우던 시절, 파란 화면에 코드만 치면 바로 컴파일되고 실행되던 그 직관적인 경험을 잊지 못한다.

최근 몇 년간 Light Table 같은 시도들이 있었고, 현재 Lem 같은 훌륭한 대안도 있지만, 대부분은 결국 "차세대 Emacs"를 지향하며 범용성을 챙기려다 이도 저도 아니게 되는 경우가 많았다. 반면 mine은 철저하게 Coalton과 Common Lisp에만 종속된다. 범용성을 버리고 특수성을 취함으로써 극단적인 사용성을 확보한 셈이다.

## 시니어 엔지니어의 시선: 프로덕션 레벨인가?

솔직히 말해서, 당장 내일 회사 프로덕션 코드를 작성해야 한다면 나는 여전히 Emacs와 SLIME 조합을 쓸 것이다. mine은 아직 알파 버전(v0.1.0)이고, 개발자 본인도 버그와 부족한 점이 많다고 인정하고 있다.

하지만 온보딩 관점에서는 얘기가 다르다. 새로운 팀원에게 Lisp이나 Coalton을 가르칠 때, "일단 Emacs부터 깔고 이 설정 파일 복사해"라고 말하는 것과 "mine 다운받아서 실행해"라고 말하는 것은 천지차이다. 도구의 진입장벽이 언어의 진입장벽으로 둔갑하는 현상을 mine이 완벽하게 해결해주고 있다.

## 총평

mine은 단순한 텍스트 에디터가 아니라, Lisp 생태계가 오랫동안 외면해왔던 **사용자 경험(UX)** 에 대한 반성문이자 훌륭한 해결책이다. 당장 메인 IDE로 쓰기엔 무리가 있더라도, 주말에 Coalton이나 Common Lisp을 가볍게 맛보고 싶은 엔지니어라면 주저 없이 다운로드해보길 권한다. 1.0.0 버전이 릴리즈될 즈음에는 Lisp 생태계의 판도가 조금은 바뀌어 있지 않을까 기대해본다.

### References
- **Original Article:** https://coalton-lang.github.io/20260424-mine/
- **Hacker News Thread:** https://news.ycombinator.com/item?id=47894014
