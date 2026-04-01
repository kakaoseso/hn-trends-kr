---
title: "Flaky 테스트의 악몽을 끝내다: Git Bayesect와 베이지안 추론의 만남"
description: "Flaky 테스트의 악몽을 끝내다: Git Bayesect와 베이지안 추론의 만남"
pubDate: "2026-04-01T19:40:27Z"
---

15년 넘게 엔지니어로 일하면서 수많은 버그를 잡아왔지만, 가장 멘탈을 갉아먹는 상황을 하나 꼽으라면 단연코 **Flaky** 테스트를 디버깅할 때입니다. 코드가 명확하게 깨진다면 `git bisect`를 돌려놓고 커피 한 잔 마시고 오면 그만입니다. 하지만 성공률이 90%에서 30%로 떨어진 애매한 상황이라면 어떨까요? 기존의 Binary Search 기반 이진 탐색은 단 한 번의 잘못된 Pass/Fail 판단으로 완전히 엉뚱한 커밋을 범인으로 지목하게 됩니다.

결국 우리는 수동으로 스크립트를 짜서 각 커밋마다 테스트를 100번씩 돌려보고 통계적으로 유의미한 차이를 찾아내는 삽질을 반복해왔습니다. 그런데 최근 Hacker News에서 이 문제를 수학적으로 우아하게 풀어낸 프로젝트를 발견했습니다. 바로 **Git Bayesect** 입니다.

## Git Bayesect: 베이지안 추론으로 커밋 찾기

[Git Bayesect](https://github.com/hauntsaninja/git_bayesect)는 이름에서 알 수 있듯이, 기존의 `git bisect`에 베이지안 추론(Bayesian inference)을 결합한 도구입니다. 이 도구의 핵심 철학은 **테스트 결과가 Deterministic하지 않다는 것을 기본 전제로 깔고 간다는 점** 입니다.

단순히 '성공' 혹은 '실패'라는 이분법적 사고에서 벗어나, 특정 커밋에서 버그가 발생할 **확률(Likelihood)** 의 변화를 감지합니다. 

### 내부 동작 원리 (The Math Magic)

이 툴이 단순히 확률을 계산하는 껍데기 스크립트였다면 제가 이렇게 블로그에 소개하지도 않았을 겁니다. 내부를 들여다보면 꽤 흥미로운 수학적 트릭들이 숨어 있습니다.

- **Beta-Bernoulli Conjugacy:** 이 도구는 실패 확률을 알 수 없는 상황을 처리하기 위해 베타-베르누이 켤레 사전 분포(Conjugate prior) 트릭을 사용합니다. 즉, 우리가 정확한 실패 확률을 모르더라도, 테스트를 한 번씩 실행할 때마다 사후 확률(Posterior probability)을 효율적으로 업데이트할 수 있습니다.
- **Expected Entropy Minimization:** 다음으로 테스트할 커밋을 고를 때 단순히 중간 지점을 선택하지 않습니다. 대신, 기대 엔트로피(Expected entropy)를 최소화하는 탐욕적(Greedy) 방식을 사용합니다. 쉽게 말해, **현재 상태에서 가장 많은 정보를 얻을 수 있는 커밋** 을 수학적으로 계산해서 다음 타겟으로 지정한다는 뜻입니다. 

이 구조 덕분에 Sample efficiency가 극대화됩니다. 테스트 실행 비용이 높은 환경에서는 정말 엄청난 강점이 아닐 수 없습니다.

## 어떻게 사용하는가?

파이썬 환경에서 간단하게 설치하고 사용할 수 있습니다. 기존 `git bisect`의 워크플로우를 크게 해치지 않으면서도 훨씬 유연합니다.

```bash
# 설치
pip install git_bayesect

# 탐색 시작
git bayesect start --old $COMMIT

# 현재 커밋에 대한 관측 결과 기록
git bayesect pass
git bayesect fail

# 자동화된 스크립트 실행
git bayesect run python flaky_test.py
```

제가 가장 감탄한 기능은 **Prior(사전 확률)를 주입할 수 있다는 점** 입니다. 대규모 Monorepo 환경에서는 모든 커밋이 동일한 용의선상에 있지 않습니다. 파일 이름이나 커밋 메시지를 기반으로 가중치를 줄 수 있습니다.

```bash
# 의심스러운 파일이 변경된 커밋에 가중치 부여
git bayesect priors_from_filenames --filenames-callback "return 10 if any('suspicious' in f for f in filenames) else 1"
```

이건 정말 현업의 Pain point를 정확히 이해하고 만든 기능입니다. 타임아웃 관련 버그를 찾을 때, 커밋 메시지에 'timeout'이나 'race condition'이 들어간 커밋부터 집중적으로 파고들게 만들 수 있으니까요.

## Hacker News 커뮤니티의 반응과 나의 생각

이 프로젝트가 [Hacker News](https://news.ycombinator.com/item?id=47557921)에 올라왔을 때, 시니어 엔지니어들의 반응은 폭발적이었습니다. 저 역시 몇 가지 흥미로운 토론에 깊게 공감했습니다.

가장 눈에 띄었던 피드백은 **Performance Regression** 에 대한 적용 가능성이었습니다. 한 유저는 벤치마크 점수처럼 연속적인(Continuous) 값을 다룰 때, 단순히 임계값을 정해 Pass/Fail로 나누면 정보의 손실이 발생한다고 지적했습니다. 

솔직히 저도 이 부분은 아쉽습니다. 현재의 모델은 이산적인(Discrete) 베르누이 시행에 맞춰져 있습니다. 만약 Latency나 Throughput 같은 연속적인 지표의 분포 변화(예: 평균이나 분산의 이동)를 감지하도록 가우시안 모델 등을 플러그인 형태로 지원한다면, 성능 튜닝의 판도를 바꿀 툴이 될 수 있을 겁니다.

또한 테스트를 여러 번 돌려 배치(Batch)로 결과를 입력하는 기능에 대한 논의도 있었습니다. 작성자는 도구 자체가 가장 정보량이 많은 커밋을 찾아주기 때문에 굳이 한 커밋에서 여러 번 테스트하는 것보다 도구가 지시하는 대로 이동하는 것이 효율적이라고 답변했습니다. 이론적으로는 맞지만, 테스트 환경 셋업(Setup/Teardown) 비용이 매우 큰 E2E 테스트 환경이라면 배치 입력을 지원하는 것이 실용적일 수 있습니다.

## 총평 (Verdict)

Git Bayesect는 모든 버그에 필요한 도구는 아닙니다. 99%의 Deterministic한 버그는 여전히 `git bisect`로 충분합니다. 

하지만 남은 1%의 악랄한 Non-deterministic 버그, 즉 시스템의 부하나 타이밍 이슈로 인해 간헐적으로 발생하는 버그를 추적해야 할 때, 이 도구는 여러분의 주말을 지켜줄 강력한 무기가 될 것입니다. 단순한 휴리스틱이 아닌 수학적 기반 위에서 동작한다는 점이 엔지니어로서 매우 신뢰가 갑니다. 

당장 도입하지 않더라도, 팀의 사내 위키나 개인 `.gitconfig` 툴킷 리스트에 반드시 메모해 두시길 강력히 권장합니다.
