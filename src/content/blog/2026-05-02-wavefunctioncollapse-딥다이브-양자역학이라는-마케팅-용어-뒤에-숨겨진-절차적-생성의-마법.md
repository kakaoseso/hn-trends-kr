---
title: "WaveFunctionCollapse 딥다이브: 양자역학이라는 마케팅 용어 뒤에 숨겨진 절차적 생성의 마법"
description: "WaveFunctionCollapse 딥다이브: 양자역학이라는 마케팅 용어 뒤에 숨겨진 절차적 생성의 마법"
pubDate: "2026-05-02T11:58:44Z"
---

최근 Hacker News를 뜨겁게 달군 10년 된 레포지토리가 하나 있습니다. 바로 mxgmn의 [WaveFunctionCollapse](https://github.com/mxgmn/WaveFunctionCollapse)입니다. Townscaper나 Caves of Qud 같은 게임을 해보셨거나 인디 게임 개발에 관심이 있다면 이미 이 알고리즘의 결과물을 맛보신 겁니다.

솔직히 고백하자면, 몇 년 전 처음 이 알고리즘의 이름을 들었을 때 저는 꽤나 비판적이었습니다. "Wave Function Collapse(파동 함수 붕괴)? 양자역학을 빙자한 어그로성 네이밍이군." 하지만 코드를 까보고 내부 원리를 이해한 뒤에는 제 오만함을 반성할 수밖에 없었습니다. 이 알고리즘은 단순하지만 강력하고, 무엇보다 엔지니어링 관점에서 대단히 우아합니다.

## 양자역학은 잊어라, 본질은 Constraint Satisfaction이다

HN 스레드의 한 유저가 날카롭게 지적했듯, 이 알고리즘은 실제 양자역학과는 아무런 관련이 없습니다. 본질적으로는 Constraint Satisfaction Problem (CSP)을 해결하기 위한 휴리스틱 기반의 Constraint Propagation 알고리즘입니다.

작동 방식은 크게 세 단계로 나뉩니다.

- **Superposition:** 초기 상태의 모든 타일은 가능한 모든 패턴의 확률적 '중첩' 상태로 존재합니다. (Boolean 배열이 모두 true인 상태)
- **Observation:** 가장 엔트로피(Shannon entropy)가 낮은, 즉 '가장 경우의 수가 적게 남은' 타일을 하나 골라 특정 상태로 확정(Collapse) 짓습니다.
- **Propagation:** 확정된 타일의 정보가 인접한 타일들로 전파되며 불가능한 상태들을 제거합니다.

여기서 핵심은 **Lowest Entropy Heuristic** 입니다. 사람이 캔버스에 그림을 그리거나 스도쿠를 풀 때, 가장 제약이 심한(채울 수 있는 경우의 수가 적은) 부분부터 해결해나가는 것과 정확히 같은 이치입니다. 이 휴리스틱 덕분에 생성 과정에서 방향성 편향(directional bias)이 사라지고 자연스러운 확장이 가능해집니다.

```cpp
// 엔트로피가 가장 낮은 노드를 찾는 핵심 로직의 개념적 수도코드
double min_entropy = INF;
int argmin = -1;

for (int i = 0; i < wave.size(); i++) {
    if (wave[i].is_collapsed()) continue;
    
    double entropy = calculate_shannon_entropy(wave[i]);
    if (entropy < min_entropy) {
        min_entropy = entropy;
        argmin = i;
    }
}
```

## Stable Diffusion의 조상인가?

HN 댓글 중 "이것이 Stable Diffusion 같은 모델의 아주 단순한 조상격인가요?"라는 질문이 있었습니다. 흥미로운 시각이지만, 두 기술의 철학은 완전히 다릅니다.

Diffusion 모델은 Latent space에서 노이즈를 깎아내며 통계적으로 '그럴싸한(plausible)' 결과물을 만들어내는 Connectionist 접근법입니다. 반면 WFC는 **엄격한 규칙(Symbolic)** 을 따릅니다. WFC의 가장 중요한 원칙(C1)은 "결과물은 반드시 입력값에 존재하는 NxN 패턴만을 포함해야 한다"는 것입니다.

즉, WFC는 환각(Hallucination)이 없습니다. 타일 간의 연결 조건이 완벽하게 맞아떨어지는 수학적 보장을 제공합니다. 제가 현업에서 레벨 디자인이나 에셋 생성 툴을 설계할 때 딥러닝 기반 생성을 꺼리는 가장 큰 이유가 바로 '제어 불가능성' 때문인데, WFC는 제약 조건(Constraints)을 통해 이 문제를 완벽하게 통제합니다.

## NP-Hard와 실무적 한계

물론 이 알고리즘이 은탄환은 아닙니다. WFC 알고리즘이 항상 모순(Contradiction) 없이 끝난다는 보장은 없습니다. 주어진 타일셋으로 그리드를 완벽하게 채우는 문제는 본질적으로 NP-hard이기 때문입니다. Propagation 과정에서 특정 타일의 가능한 상태가 0이 되어버리면 알고리즘은 막다른 길에 다다릅니다.

원작자는 "실제로는 모순에 빠지는 일이 놀라울 정도로 적다"고 말하지만, 프로덕션 환경에서 복잡한 3D 복셀 타일셋을 다루다 보면 이야기가 달라집니다. 현업에서 이를 해결하기 위해 Backtracking을 빡세게 구현하거나, 충돌이 난 청크만 도려내서 다시 돌리는(Restart) 방식의 엔지니어링 튜닝이 필수적입니다. 타일 대칭성(Symmetry) 시스템을 도입해 연산량을 줄이는 기법도 반드시 적용해야 합니다.

## 총평: 프로덕션 레벨인가?

결론부터 말하자면, **절대적으로 그렇습니다.**

Bad North, Caves of Qud 같은 훌륭한 게임들이 이미 증명했고, Unreal Engine 5나 Houdini 같은 산업 표준 툴에도 플러그인 형태로 깊숙이 침투해 있습니다. AI가 모든 것을 집어삼키는 블랙박스의 시대에, 이렇게 직관적이고 결정론적인(deterministic) 알고리즘이 주는 안정감은 시니어 엔지니어들에게 엄청난 무기입니다.

화려한 "생성형 AI" 마케팅 워드에 지치셨다면, 이번 주말에는 커피 한 잔 내려놓고 이 10년 된 C# 코드를 찬찬히 뜯어보시길 권합니다. 알고리즘이 주는 순수한 즐거움을 오랜만에 느끼실 수 있을 겁니다.

- **Original Article:** https://github.com/mxgmn/WaveFunctionCollapse
- **Hacker News Thread:** https://news.ycombinator.com/item?id=47961100
