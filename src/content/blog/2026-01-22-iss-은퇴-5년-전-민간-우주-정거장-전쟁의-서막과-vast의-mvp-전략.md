---
title: "ISS 은퇴 5년 전, 민간 우주 정거장 전쟁의 서막과 Vast의 MVP 전략"
description: "ISS 은퇴 5년 전, 민간 우주 정거장 전쟁의 서막과 Vast의 MVP 전략"
pubDate: "2026-01-22T14:46:36Z"
---

엔지니어로서 'Legacy System Migration'이라는 단어만 들어도 머리가 지끈거리는 분들 많으실 겁니다. 그런데 그 레거시 시스템이 지구 궤도를 돌고 있는 400톤짜리 구조물이고, 5년 뒤에는 폐기(Decommission)해야 한다면 어떨까요?

바로 국제우주정거장(ISS) 이야기입니다. NASA는 지금 발등에 불이 떨어졌습니다. ISS는 노후화되었고, 이를 대체할 'Commercial LEO Destinations (CLD)' 프로그램은 시간과의 싸움을 벌이고 있습니다. 오늘은 그 경쟁의 최전선에 있는 **Vast Space** 와 그들의 첫 번째 상용 우주 정거장 **Haven-1** 에 대해 이야기해볼까 합니다. Ars Technica의 인터뷰와 Hacker News의 반응을 바탕으로 엔지니어링 관점에서 뜯어보겠습니다.

## Monolith vs Microservices: 우주 정거장의 접근 방식

현재 ISS 대체 프로젝트에는 여러 플레이어가 있습니다. Voyager(Starlab), Axiom Space, Blue Origin(Orbital Reef), 그리고 Vast Space입니다.

재미있는 점은 각 회사의 접근 방식입니다. Blue Origin이나 Voyager가 거대한 'Next-Gen Platform'을 기획하고 있다면, Vast는 철저하게 **MVP(Minimum Viable Product)** 전략을 취하고 있습니다.

### Vast의 전략: Haven-1
- **Form Factor:** Falcon 9 로켓 하나에 실려 올라갈 수 있는 단일 모듈입니다.
- **Goal:** 장기 거주가 아닌 단기 체류(Short-duration stays)를 위한 'Interim(임시)' 정거장입니다.
- **Timeline:** 원래 2026년 중반이었으나, 2027년 1분기로 연기되었습니다.

Ars Technica의 인터뷰에서 Vast CEO Max Haot은 이 연기에 대해 "안전과 속도의 균형"이라고 설명했습니다. 하지만 우리 엔지니어들은 알죠. 하드웨어, 그것도 우주로 쏘아 올리는 하드웨어의 **Integration Hell** 이 얼마나 끔찍한지 말입니다. 2026년 런칭은 애초에 공격적인 마케팅용 날짜였을 가능성이 높습니다.

## "Build from Scratch"의 함정

Max Haot은 인터뷰에서 "빈 건물에서 시작해 팀도 없이 4년 만에 세계 최초의 상용 우주 정거장을 짓는 것"이라고 강조했습니다. 스타트업 씬에서는 멋진 피치(Pitch)지만, 엔지니어링 관점에서는 엄청난 리스크입니다.

NASA의 요구사항(Requirements)조차 아직 완전히 확정되지 않은 상태에서 개발을 진행한다는 것은, 소프트웨어로 치면 Spec도 없이 코딩부터 시작하는 것과 같습니다. 나중에 NASA가 "연속 거주(Continuous Habitation) 기능이 필수"라고 규정해버리면 Haven-1은 그냥 비싼 깡통이 될 수도 있습니다.

## Hacker News의 시선: eDonkey에서 우주까지?

Hacker News의 반응은 역시나 날카롭고 흥미롭습니다. 특히 창업자와 자본의 출처에 대한 이야기가 많았습니다.

> "eDonkey 개발에서 우주 정거장 발사까지의 커리어 패스는 정말 놀랍다."

이 댓글은 Vast의 설립자 Jed McCaleb을 두고 하는 말입니다. eDonkey(P2P), Mt.Gox(초기), Ripple, Stellar를 거쳐 이제는 우주 산업입니다. 소프트웨어 엔지니어가 하드웨어, 그것도 우주 산업으로 넘어가는 이 거대한 Pivot은 실리콘밸리 엔지니어들에게 묘한 희망(?)과 경외감을 줍니다.

하지만 비판적인 시각도 존재합니다. 우주 인프라의 민영화(Privatization)가 가져올 부작용에 대한 우려입니다.

> "유럽의 철도나 텍사스의 전력망처럼, 인프라 민영화가 서비스 질 하락과 비용 상승을 불러올 수 있다. 만약 중국이 비영리 목적으로 치고 나간다면 미국 기업들은 어떻게 될까?"

기술적으로 SpaceX가 발사 비용(Cost per kg)을 획기적으로 낮춘 것은 사실이지만, 우주 거주 공간(Habitat)까지 민간 자본의 논리로 돌아갈 때 발생할 수 있는 'SLA(Service Level Agreement) 위반'은 단순히 서버 다운타임 정도의 문제가 아닐 겁니다. 생명 유지 장치(ECLSS)가 구독형 모델이 되지 않기를 바랄 뿐입니다.

## 개인적인 견해: Done is Better than Perfect

솔직히 말해서, 저는 Blue Origin의 화려한 렌더링보다 Vast의 투박한 접근 방식에 더 점수를 주고 싶습니다.

소프트웨어 개발에서도 거창한 아키텍처를 설계하느라 3년을 보내고 결국 런칭도 못 하는 프로젝트를 수없이 봐왔습니다. 반면, 버그가 좀 있어도 일단 돌아가는 MVP를 내놓고 피드백을 받는 팀이 결국 시장을 장악합니다.

Vast가 2027년에 정말로 Haven-1을 궤도에 올린다면, 그들은 **First Mover Advantage** 를 확실히 가져갈 겁니다. NASA 입장에서도 ISS 폐기 시점은 다가오는데 대안이 없다면, 완벽하지 않아도 당장 쓸 수 있는 Vast의 모듈을 선택할 수밖에 없을 테니까요.

물론, "Integration Phase"에서 발생할 수많은 이슈들을 그들이 어떻게 해결할지는 지켜봐야 합니다. 우주에는 `git revert`가 없으니까요.

**Verdict:**
기술적으로는 무모해 보이지만, 비즈니스 전략적으로는 가장 현실적인 접근입니다. 다만, 2027년 1분기 런칭? 글쎄요. 저는 2027년 하반기에 겁니다.

---

**References:**
- [Original Article (Ars Technica)](https://arstechnica.com/space/2026/01/the-first-commercial-space-station-haven-1-is-now-undergoing-assembly-for-launch/)
- [Hacker News Discussion](https://news.ycombinator.com/item?id=46718330)
