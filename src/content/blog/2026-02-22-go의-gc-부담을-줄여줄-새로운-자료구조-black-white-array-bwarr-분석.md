---
title: "Go의 GC 부담을 줄여줄 새로운 자료구조: Black-White Array (BWArr) 분석"
description: "Go의 GC 부담을 줄여줄 새로운 자료구조: Black-White Array (BWArr) 분석"
pubDate: "2026-02-22T22:25:14Z"
---

Go 언어로 고성능 시스템을 설계하다 보면 결국 마주하게 되는 최종 보스는 언제나 **Garbage Collector (GC)** 입니다. 특히 수백만 개의 객체를 힙(Heap)에 띄워놓고 관리해야 하는 인메모리 캐시나 데이터베이스를 구현할 때, 포인터 추적 비용은 시스템의 전체적인 Latency에 치명적인 영향을 줍니다.

우리는 보통 `google/btree` 같은 검증된 B-Tree 구현체를 사용해왔지만, 최근 Hacker News에서 꽤 흥미로운 프로젝트가 눈에 띄어 깊게 파보았습니다. 바로 **Black-White Array (BWArr)** 입니다.

## BWArr란 무엇인가?

이 프로젝트는 2020년에 Z. George Mou 교수가 발표한 논문 [Black-White Array: A New Data Structure for Dynamic Data Sets](https://arxiv.org/abs/2004.09051)를 기반으로 한 Go 구현체입니다. GitHub 레포지토리에 따르면 이것이 "최초의 공개 구현"이라고 합니다.

핵심 컨셉은 간단하면서도 강력합니다. **정렬된 상태를 유지하면서도 메모리 할당을 획기적으로 줄이는 것** 입니다. 보통의 트리 구조가 노드 하나를 추가할 때마다 메모리 할당을 요구하는 것과 달리, BWArr는 배열 기반으로 $O(\log N)$의 메모리 할당만으로 Insert를 처리합니다.

### 기술적 특징과 장점

엔지니어 관점에서 가장 매력적인 부분은 **Pointerless** 구조라는 점입니다.

1.  **GC Pressure 감소:** 요소마다 포인터를 가지지 않습니다. Go GC가 마킹(Marking) 단계에서 스캔해야 할 포인터가 줄어든다는 뜻입니다. 이는 Stop-the-world 시간을 줄이는 데 직접적인 도움이 됩니다.
2.  **Cache Locality:** 배열 기반이므로 메모리 상에 데이터가 연속적으로 위치합니다. CPU 캐시 히트율이 높아져 순차 탐색(Iteration) 속도가 비약적으로 상승합니다.
3.  **Drop-in Replacement:** `google/btree`나 `GoLLRB`와 유사한 인터페이스를 제공하여 교체가 용이합니다.

## 벤치마크: B-Tree와의 비교

제작자가 공개한 벤치마크를 보면, 특히 순차 탐색(Iteration)에서 배열 기반 구조의 이점이 드러납니다.

![Ordered Iterate performance](https://github.com/dronnix/bwarr-bench/raw/main/images/ordered_iteration_over_all_values.png?raw=true)

정렬된 데이터를 순회할 때, 포인터를 타고 점프해야 하는 B-Tree보다 훨씬 안정적이고 빠른 성능을 보여줍니다. 무작위 순서(Unordered) 탐색에서도 준수한 성능을 보입니다.

![Unordered Iterate performance](https://github.com/dronnix/bwarr-bench/raw/main/images/unordered_iteration_over_all_values.png?raw=true)

## 하지만, 은탄환(Silver Bullet)은 없다

솔직히 말해, 처음 README를 읽으면서 "너무 좋은데?" 싶었지만, **Trade-off** 섹션을 읽고 나서야 고개를 끄덕였습니다. 이 자료구조는 명확한 약점이 존재합니다.

### 1. Latency Spikes
README에도 명시되어 있듯이, $N$번의 Insert 중 한 번은 복잡도가 $O(N)$으로 튈 수 있습니다. 물론 Amortized Complexity(분할 상환 복잡도)는 $O(\log N)$으로 유지되지만, **Real-time System** 에서는 치명적일 수 있습니다. HFT(고빈도 매매)나 실시간 광고 입찰 서버에서 갑자기 Insert 하나가 전체 배열을 재배치하느라 튀어버리면 곤란합니다. 저자는 이를 `async/background inserts`로 완화할 수 있다고 제안하지만, 이는 시스템 복잡도를 높이는 요인이 됩니다.

### 2. 특정 연산의 비효율성
작은 규모의 데이터셋이나 특정 삭제 패턴에서 성능 저하가 발생할 수 있습니다.
- **Search/Delete:** 데이터가 적을 때 오히려 $O((\log N)^2)$가 소요될 수 있습니다.
- **Min/Max:** 대량의 삭제(Delete)가 일어난 직후에는 최솟값/최댓값을 찾는 연산이 $O(N/4)$까지 느려질 수 있습니다.

## 코드 사용 예시

사용법은 Go의 표준적인 컬렉션 패턴을 따릅니다. Go 1.22 이상을 요구하며, 제네릭을 지원합니다.

```go
package main

import (
    "fmt"
    "github.com/dronnix/bwarr"
)

func main() {
    // 비교 함수와 초기 용량(Capacity Hint)을 설정
    bwa := bwarr.New(func(a, b int64) int {
        return int(a - b)
    }, 10)

    bwa.Insert(42)
    bwa.Insert(17)
    
    // 정렬된 순서로 순회
    bwa.Ascend(func(item int64) bool {
        fmt.Printf(" %d\n", item)
        return true
    })
}
```

## Hacker News의 반응과 개인적 견해

Hacker News의 반응은 아직 초기 단계라 구체적인 검증보다는 신기하다는 반응이 주를 이룹니다. 개인적으로 이 프로젝트는 **"Read-Heavy, Write-Batch"** 워크로드에 최적화되어 있다고 봅니다.

만약 여러분이 시계열 데이터를 메모리에 쌓아두고 분석하거나, 변경이 잦지 않은 대용량 인메모리 인덱스를 구축한다면 BWArr는 훌륭한 대안이 될 수 있습니다. 포인터 기반 자료구조가 주는 GC 오버헤드에서 벗어날 수 있기 때문입니다.

하지만, **Write-Heavy** 하거나 **Latency Jitter** 에 민감한 서비스라면 도입에 신중해야 합니다. $O(N)$ 스파이크는 운영 환경에서 모니터링 알람을 울리게 만드는 주범이 될 테니까요.

## 결론: Production에 쓸 수 있을까?

아직은 **시기상조** 라고 판단됩니다. 학술적으로 증명된 아이디어의 첫 구현체인 만큼, 엣지 케이스에서의 버그나 최적화 이슈가 남아있을 가능성이 큽니다. 하지만 Go 생태계에서 GC 문제를 자료구조 레벨에서 해결하려는 시도는 매우 환영할 만합니다.

**추천 대상:**
- GC 튜닝에 지친 엔지니어
- 실험적인 프로젝트에서 극한의 조회 성능을 뽑아내고 싶은 분
- 새로운 알고리즘 구현체 분석을 즐기는 분

앞으로 이 라이브러리가 어떻게 발전할지, 특히 `Serialization` 지원과 배치 작업 최적화가 이루어진다면 Redis나 여타 인메모리 DB의 내부 구현체로도 고려해볼 법합니다.
