---
title: "프론트엔드 상태 관리의 종착지? Signals의 Push-Pull 알고리즘 파헤치기"
description: "프론트엔드 상태 관리의 종착지? Signals의 Push-Pull 알고리즘 파헤치기"
pubDate: "2026-04-06T15:27:28Z"
---

React를 오랫동안 사용해 온 엔지니어라면 누구나 `useEffect`와 `useMemo`의 의존성 배열 때문에 골머리를 앓아본 경험이 있을 것입니다. 배열에 값을 빼먹으면 stale closure 버그가 발생하고, 너무 많이 넣으면 무한 루프에 빠지죠.

최근 몇 년간 프론트엔드 생태계를 지켜보면서 가장 흥미로웠던 변화는 Solid, Vue, Preact, Angular, 심지어 Svelte까지 거의 모든 모던 프레임워크가 동일한 반응형 모델인 **Signals** 기술로 대동단결하고 있다는 점입니다.

하지만 우리 중 "Signal이 내부적으로 어떻게 동작하는가?"를 명확히 설명할 수 있는 사람은 얼마나 될까요? 최근 Willy Brauner가 작성한 [Signals, the push-pull based algorithm](https://willybrauner.com/journal/signal-the-push-pull-based-algorithm) 아티클과 Hacker News의 열띤 토론을 바탕으로, 이 마법 같은 시스템의 내부를 시니어 엔지니어의 시각에서 파헤쳐 보려 합니다.

## Push-Pull 알고리즘: 우아한 타협점

Signal의 핵심은 그 이름에서 알 수 있듯 상태의 변화를 전파하는 데 있습니다. 하지만 단순히 이벤트 에미터처럼 동작하는 것은 아닙니다. Signal은 **Push** 방식과 **Pull** 방식을 영리하게 결합합니다.

- **Push (Eager Invalidation):** Signal의 값이 변경되면, 즉시 구독자들에게 "내 값이 변했어!"라고 알립니다. 하지만 여기서 중요한 점은 절대 **새로운 값을 계산해서 넘겨주지 않는다** 라는 것입니다. 그저 자신에 의존하는 Computed 노드들을 `dirty = true` 상태로 마킹할 뿐입니다.
- **Pull (Lazy Evaluation):** Computed 값은 누군가 그 값을 실제로 읽으려고 할 때만 재계산됩니다. 만약 `dirty` 플래그가 켜져 있다면 그때서야 의존성 트리를 따라 올라가며 최신 값을 가져와 계산하고 캐싱합니다.

개인적으로 이 구조는 성능 최적화의 교과서 같다고 생각합니다. 상태가 변할 때마다 연쇄적으로 무거운 연산을 수행하는 대신, 무효화만 시켜두고 실제 연산은 값이 필요한 시점까지 지연시키는 것이죠. 불필요한 렌더링을 막는 핵심 비결이 바로 여기에 있습니다.

## 마법의 연결고리: Global STACK과 Auto-Tracking

React 개발자들이 Signal을 처음 접할 때 가장 놀라는 부분은 의존성 배열을 명시할 필요가 없다는 것입니다. 시스템이 알아서 어떤 Signal이 참조되었는지 추적하죠. 이 기술은 어떻게 구현될까요?

핵심은 하나의 전역 배열, 즉 `STACK`에 있습니다.

```typescript
type ComputeContext = {
  setDirty: () => void
  addSource: (cleanup: () => void) => void
}
const STACK: Array<ComputeContext> = []
```

`computed` 함수가 실행될 때 벌어지는 일련의 과정은 다음과 같습니다.

1. `computed` 내부의 함수가 실행되기 직전, 자신을 전역 `STACK`에 푸시합니다.
2. 내부 함수가 실행되면서 특정 `signal`의 `getter`를 호출하게 됩니다.
3. `signal`의 `getter`는 전역 `STACK`의 최상단 값을 확인합니다. 만약 누군가 있다면, 그 객체를 자신의 구독자 목록에 추가합니다.
4. 함수 실행이 끝나면 `STACK`에서 자신을 팝합니다.

솔직히 처음 이 패턴을 보았을 때, "전역 상태를 이렇게 대놓고 쓴다고?"라며 약간의 거부감이 들었습니다. Hacker News의 한 유저도 비슷한 지적을 했죠.

> "전역 스택을 사용하는 것은 데이터 레이스를 유발할 수 있습니다! 멀티스레드 환경이라면 글로벌 뮤텍스가 필요할 텐데 이건 끔찍하죠."

하지만 JavaScript가 싱글 스레드 기반이라는 점을 떠올리면, 이 전역 스택 트릭은 매우 실용적이고 우아한 해결책입니다. 복잡한 컨텍스트 패싱 없이도 런타임에 동적으로 의존성 그래프를 그려낼 수 있으니까요.

## 아티클이 놓친 중요한 퍼즐: Glitch-Freedom

Willy Brauner의 아티클은 알고리즘의 뼈대를 훌륭하게 설명했지만, Hacker News 커뮤니티의 시니어 엔지니어들은 날카로운 지적을 놓치지 않았습니다. 바로 **Glitch-Freedom** 개념에 대한 언급이 빠졌다는 것이죠.

여러 개의 파생 상태가 동일한 원본 Signal을 참조하는 다이아몬드 의존성 구조를 상상해 봅시다. 원본이 변경되었을 때, 이를 참조하는 중간 노드들이 무효화되고, 최종 노드가 재계산되어야 합니다.

만약 단순한 Push-Pull로만 동작한다면, 최종 노드가 재계산되는 시점에 일부 중간 노드는 최신 값으로 업데이트되었지만 다른 중간 노드는 아직 과거의 값을 가지는 **불가능한 중간 상태** 현상이 발생할 수 있습니다. 이를 "Glitch"라고 부릅니다.

HN의 한 유저는 이렇게 회고했습니다.
> "글리치는 실제로 겪어보기 전까지는 문제인지도 모릅니다. 저는 대시보드 프로젝트에서 단일 프레임 동안 퍼센티지가 잠깐 음수로 표시되는 버그를 잡느라 하루를 꼬박 날린 적이 있습니다."

이를 해결하기 위해 상용 라이브러리들은 Pull 단계에서 **위상 정렬** 알고리즘을 수행합니다. 의존성 그래프의 깊이에 따라 순서를 매겨, 모든 부모 노드가 최신화된 이후에만 자식 노드가 평가되도록 보장하는 것이죠. 우리가 직접 Signal 시스템을 바닥부터 구현해서는 안 되는 가장 큰 이유가 바로 이 복잡성에 있습니다.

## 15년을 돌고 돌아 안착한 곳

Hacker News 댓글 창에서 흥미로웠던 또 다른 사실은, 이 반응형 패러다임이 결코 새로운 것이 아니라는 점입니다. 무려 2008년에 발표된 [Flapjax](https://www.flapjax-lang.org/)라는 라이브러리가 이미 이와 유사한 개념을 구현했었다고 합니다.

프론트엔드 생태계는 지난 15년 동안 수많은 상태 관리 패턴을 거쳐 결국 더 나은 인체공학적 API를 갖춘 "오래된 아이디어"로 회귀하고 있습니다.

## 결론: 그래서 우리는 무엇을 해야 할까?

현재 TC39에서는 Signal을 JavaScript의 네이티브 스펙으로 편입시키려는 [proposal-signals](https://github.com/tc39/proposal-signals) 작업이 진행 중입니다. 만약 이것이 통과된다면, 프레임워크들은 각자의 렌더링 로직에만 집중하고 상태 관리의 근간은 브라우저 엔진에 맡기게 될 것입니다.

엔지니어로서 우리는 당장 바닐라 JS로 Signal을 짤 필요는 없습니다. 하지만 우리가 매일 사용하는 도구의 기저에 깔린 **Eager Invalidation** 및 **Lazy Evaluation** 철학이 어떻게 맞물려 돌아가는지, 그리고 **위상 정렬** 기법을 통해 어떻게 데이터 정합성을 보장하는지 이해하는 것은 매우 중요합니다.

마법처럼 보이는 기술일수록, 그 모자를 들춰보고 트릭을 파악하는 것이 시니어 엔지니어의 숙명이니까요.

### References
- Original Article: [Signals, the push-pull based algorithm](https://willybrauner.com/journal/signal-the-push-pull-based-algorithm)
- Hacker News Discussion: [https://news.ycombinator.com/item?id=47637116](https://news.ycombinator.com/item?id=47637116)
