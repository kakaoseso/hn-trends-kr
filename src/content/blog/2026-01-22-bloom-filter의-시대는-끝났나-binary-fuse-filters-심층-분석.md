---
title: "Bloom Filter의 시대는 끝났나? Binary Fuse Filters 심층 분석"
description: "Bloom Filter의 시대는 끝났나? Binary Fuse Filters 심층 분석"
pubDate: "2026-01-22T02:11:02Z"
---

엔지니어링 면접에서 "존재 여부를 빠르게 확인하려면 어떻게 해야 하죠?"라는 질문을 받으면, 십중팔구는**Bloom Filter**를 답합니다. 물론 정답입니다. 하지만 2024년 현재, 실제 고성능 시스템을 설계하는 Principal Engineer로서 저는 되묻고 싶습니다. "정말 Bloom Filter가 최선입니까?"

오늘은 오랫동안 왕좌를 지켜온 Bloom Filter와 그 대안으로 떠올랐던 XOR Filter를 넘어, 더 빠르고 더 작은**Binary Fuse Filters**에 대해 이야기해보려 합니다. 특히 불변(Immutable) 데이터셋을 다루는 시스템을 설계 중이라면, 이 글이 꽤 흥미로울 것입니다.

## 1. Bloom Filter, 무엇이 문제인가?

Bloom Filter는 1970년에 고안된 이래로 놀라운 효율성을 보여주었습니다. 하지만 현대적인 하드웨어 관점에서는 몇 가지 아쉬움이 있습니다.

1.**공간 효율성:**Bloom Filter는 이론적인 정보 엔트로피 하한선(Shannon limit)보다 약**44%**더 많은 공간을 사용합니다. 수십억 개의 키를 다루는 대규모 스토리지 엔진에서는 이 44%가 수 기가바이트의 RAM 낭비로 이어집니다.
2.**Cache Locality:**표준 Bloom Filter는 여러 해시 함수가 가리키는 비트들이 메모리 여기저기에 흩어져 있습니다. 이는 CPU Cache Miss를 유발하여 조회 성능을 떨어뜨립니다. (물론 Blocked Bloom Filter가 이를 어느 정도 해결하지만, 공간 효율성은 더 떨어집니다.)

## 2. XOR Filter의 등장, 그리고 한계

몇 년 전, 이 문제를 해결하기 위해**XOR Filter**가 등장했습니다. `3-way XOR` 연산을 통해 멤버십을 확인하는 이 구조는 Bloom Filter보다 공간 효율적(이론적 하한선의 23% 수준)이며, Cache 친화적입니다.

하지만 XOR Filter도 완벽하지 않았습니다. 특히**필터 생성(Construction) 속도**가 느리다는 단점이 있었죠. 대용량 데이터를 메모리에 올리고 필터를 빌드할 때, 이 시간은 배포 파이프라인의 병목이 되곤 합니다.

## 3. Binary Fuse Filters: Game Changer

2022년 arXiv에 공개된 논문 *"Binary Fuse Filters: Fast and Smaller Than Xor Filters"*는 이 게임의 판도를 바꿨습니다. Daniel Lemire 교수팀(이분은 고성능 파싱/인덱싱 분야의 락스타입니다)이 제안한 이 구조는 XOR Filter의 아이디어를 기반으로 하되,**Fuse Graph**라는 개념을 도입했습니다.

### 핵심 성능 요약 (논문 기반)

***공간 효율성:**이론적 하한선의**13%**이내로 접근했습니다. (Bloom Filter: 44%, XOR Filter: 23%)
***생성 속도:**XOR Filter보다**2배 이상**빠릅니다.
***조회 속도:**Bloom Filter보다 빠르며, XOR Filter와 대등합니다.

### 어떻게 작동하는가?

기술적으로 깊게 들어가자면, Binary Fuse Filter는**Probabilistic Data Structure**이면서**Static**합니다. 즉, 데이터가 한 번 빌드되면 추가/삭제가 불가능합니다. (이 부분은 뒤에서 다시 다루겠습니다.)

기본 원리는 키(Key)를 세 개의 해시 위치(h0, h1, h2)에 매핑하고, 해당 위치의 값들을 XOR 연산했을 때 키의 `Fingerprint`가 나오도록 만드는 것입니다.

```c
// 개념적인 조회 로직 (Pseudo-code)
bool contains(key) {
    fingerprint = hash_fingerprint(key);
    h0 = hash0(key);
    h1 = hash1(key);
    h2 = hash2(key);
    // 세 위치의 값을 XOR한 결과가 fingerprint와 같으면 존재한다고 판단
    return (B[h0] ^ B[h1] ^ B[h2]) == fingerprint;
}
```

이 구조를 만들기 위해 내부적으로는 하이퍼그래프(Hypergraph)의 Peeling Algorithm을 사용하는데, Binary Fuse Filter는 이 그래프 구조를 더 작고 캐시 친화적인 세그먼트로 나누어(Fuse) 생성 속도와 공간 효율을 극대화했습니다.

## 4. Production 환경에서의 고려사항

이 기술이 아무리 좋아도, 무조건 Bloom Filter를 갖다 버려야 하는 것은 아닙니다. Principal Engineer로서 판단해야 할 Trade-off는 명확합니다.

### 언제 써야 하는가? (Use Cases)

***Immutable Data:**데이터가 한 번 쓰이면 변하지 않는 경우. 예를 들어,**RocksDB의 SSTable**이나**검색 엔진의 인덱스 파일**같은 경우입니다. 실제로 Binary Fuse Filter는 이러한 'Write-Once, Read-Many' 시나리오를 위해 설계되었습니다.
***Memory Constraint:**임베디드 장비나, 수십억 개의 키를 메모리에 올려야 하는 서버.

### 언제 쓰면 안 되는가?

***Dynamic Sets:**데이터가 수시로 추가(`add`)되거나 삭제(`delete`)되어야 한다면 사용할 수 없습니다. 이 경우엔**Cuckoo Filter**나 전통적인**Counting Bloom Filter**가 맞습니다. Binary Fuse Filter는 전체 데이터셋을 가지고 처음부터 다시 빌드해야 합니다.

## 5. 커뮤니티와 현장의 목소리

이 논문과 관련 기술에 대해 Hacker News나 엔지니어 커뮤니티에서는 흥미로운 논의들이 오갑니다. (비록 이번 링크에는 댓글이 없었지만, 이 분야의 일반적인 담론을 반영하자면 다음과 같습니다.)

많은 엔지니어들이**"복잡도 대비 이득"**에 대해 의문을 가집니다. *"Bloom Filter는 구현이 10줄이면 끝나는데, 굳이 복잡한 Fuse Filter를 써야 하나?"* 라는 것이죠.

제 생각은 이렇습니다.**"Scale에 따라 다르다."**
작은 서비스, 키가 수백만 개 수준이라면 Bloom Filter로 충분합니다. 하지만 키가 수십억 개가 되고, 인스턴스 비용을 줄여야 하는 상황이라면 10~20%의 메모리 절감은 엄청난 비용 차이를 만들어냅니다. 또한, 최신 라이브러리들(Rust, C++, Go 등)이 이미 구현체를 제공하고 있으므로 구현 복잡도는 라이브러리 사용자 입장에서 큰 문제가 되지 않습니다.

## 6. 결론 (Verdict)

**Binary Fuse Filter는 Static Set Membership 문제의 현재 'State-of-the-Art'입니다.**

만약 여러분이 데이터베이스 엔진, 대규모 캐시 시스템, 혹은 불변 데이터 저장소를 만들고 있다면, Bloom Filter 대신 Binary Fuse Filter를 기본값으로 고려해야 합니다. 메모리는 더 적게 쓰고, 빌드는 더 빠르며, 조회 성능은 뛰어납니다. 쓰지 않을 이유가 없습니다.

하지만 동적인 데이터 변경이 잦은 일반적인 애플리케이션 레벨의 캐싱이라면? 여전히 Cuckoo Filter나 Redis의 내장 기능들이 더 실용적일 것입니다. 도구는 목적에 맞게 써야 하니까요.

---

**References:**
*   Original Article: [Binary Fuse Filters: Fast and Smaller Than Xor Filters](https://arxiv.org/abs/2201.01174)
*   Hacker News Thread: [HN Discussion](https://news.ycombinator.com/item?id=46658128)
