---
title: "Bloom Filter는 이제 놔줍시다: Binary Fuse Filters 심층 분석"
description: "Bloom Filter는 이제 놔줍시다: Binary Fuse Filters 심층 분석"
pubDate: "2026-01-22T04:38:41Z"
---

엔지니어링 생활을 15년 넘게 하다 보면, "이건 그냥 원래 그런 거야"라고 무심코 넘어가는 기술들이 있습니다. 대표적인 것이 바로 **Bloom Filter** 입니다. 데이터베이스의 불필요한 Disk I/O를 줄이거나, 캐시 미스를 방지하기 위해 우리는 습관적으로 Bloom Filter를 사용합니다. Redis, Cassandra, RocksDB 등 우리가 사랑하는 인프라의 기저에는 항상 이 친구가 있었죠.

하지만 솔직해져 봅시다. Bloom Filter는 완벽하지 않습니다. False Positive(위양성)를 줄이려면 메모리를 더 써야 하고, 해시 함수를 여러 번 돌려야 하니 CPU 사이클도 은근히 잡아먹습니다. Cache Locality 관점에서도 그리 친화적이지 않죠.

오늘 소개할 **Binary Fuse Filters** 는 바로 이 오래된 기술적 부채를 청산할 강력한 대안입니다. 2022년에 발표된 이 논문은 기존의 XOR Filter를 한 단계 더 진화시켰는데, 결론부터 말하자면 **"더 빠르고, 더 작습니다."**

## Bloom Filter, 그리고 XOR Filter의 한계

전통적인 Bloom Filter가 이론적 하한선(Theoretical Lower Bound) 대비 약 44%의 오버헤드를 가진다는 사실, 알고 계셨나요? 쉽게 말해, 우리가 필요 이상으로 메모리를 낭비하고 있다는 뜻입니다.

이 문제를 해결하기 위해 몇 년 전 **XOR Filter** 가 등장했습니다. XOR Filter는 오버헤드를 23%까지 줄였고, 쿼리 속도도 획기적으로 개선했습니다. 하지만 치명적인 단점이 하나 있었는데, 바로 **Construction Time(생성 시간)** 입니다. 필터를 구축하는 과정이 복잡해서, 데이터가 자주 바뀌거나 빠르게 빌드해야 하는 환경에서는 도입이 꺼려졌던 게 사실입니다.

## Binary Fuse Filters: 게임 체인저의 등장

이 논문(Binary Fuse Filters: Fast and Smaller Than Xor Filters)의 저자들은 XOR Filter의 아이디어를 계승하면서도, 구조적인 병목을 해결했습니다. 핵심 스펙을 정리하면 다음과 같습니다.

- **메모리 효율:** 이론적 하한선 대비 오버헤드가 **13%** 에 불과합니다. (옵션을 타협하면 8%까지 가능)
- **생성 속도:** 기존 XOR Filter보다 **2배 이상** 빠릅니다.
- **조회 속도:** Bloom Filter보다 훨씬 빠르며, XOR Filter와 대동소이합니다.

### 기술적 작동 원리 (Deep Dive)

Binary Fuse Filter는 기본적으로 **Probabilistic Data Structure** 입니다. 핵심은 데이터(Key)를 3개의 세그먼트로 매핑하고, 이를 XOR 연산을 통해 검증하는 구조입니다. 기존 XOR Filter가 전체 데이터셋에 대해 복잡한 매핑을 수행해야 했다면, Binary Fuse Filter는 이를 더 작은 'Fuse' 단위로 쪼개어 처리합니다.

이 구조가 가져오는 엔지니어링적 이점은 명확합니다.

1. **Cache Friendly:** 데이터 구조가 작고 인접해 있어 CPU 캐시 적중률이 높습니다.
2. **Zero-Allocation:** 구현체에 따라 다르지만, 기본적으로 쿼리 시점에 별도의 메모리 할당이 거의 없습니다.
3. **Batch Build 최적화:** 생성 속도가 빨라졌다는 것은, 주기적으로 덤프를 떠야 하는 **Immutable Dataset** (예: LSM Tree의 SSTable)에 적용하기 훨씬 수월해졌다는 뜻입니다.

## Hacker News의 반응과 팩트 체크

이 기술이 소개되었을 때 Hacker News 커뮤니티의 반응도 흥미로웠습니다. 특히 한 유저는 "제목이 오타 아니냐? 빠르면서 더 작다는 게 말이 되냐?"라는 반응을 보였습니다.

> "Fast, but not faster than XOR filters. I was wondering if the title was a typo..."

여기서 명확히 짚고 넘어가야 할 점이 있습니다. 논문에서 말하는 "Fast"는 **Construction Speed(생성 속도)** 를 강조한 것입니다. 조회(Query) 속도는 XOR Filter와 거의 비슷하거나, 압축률을 극단적으로 높일 경우 아주 미세하게 느려질 수 있습니다. 하지만 Bloom Filter와 비교하면 조회 속도 역시 압도적으로 빠릅니다.

또 다른 중요한 지적은 이 기술이 **Static Set** 에 최적화되어 있다는 점입니다. Bloom Filter처럼 데이터를 동적으로 하나씩 `add()` 하는 구조가 아닙니다. 전체 데이터를 한 번에 빌드해야 합니다. 따라서 실시간으로 데이터가 추가되는 스트림 처리보다는, **일별 배치 작업이나 Read-Only 데이터베이스** 에 적합합니다.

## Principal Engineer's Verdict

그래서, 이걸 지금 프로덕션에 써야 할까요?

제 의견은 **"Immutable Data를 다룬다면 무조건 교체하라"** 입니다.

만약 여러분이 RocksDB나 LevelDB 같은 LSM-Tree 기반 스토리지를 튜닝하고 있거나, 대용량 데이터의 존재 여부를 빠르게 체크해야 하는 검색 엔진을 만들고 있다면 Binary Fuse Filter는 선택이 아니라 필수입니다. Daniel Lemire 교수(저자)가 공개한 C, Go, Rust 구현체들은 이미 퀄리티가 상당히 높고 안정적입니다.

**장점:**
- Bloom Filter 대비 메모리 사용량 30% 이상 절감.
- 압도적인 조회 성능 (Hash 연산 횟수 감소).

**단점:**
- **Immutable:** 한 번 만들면 수정이 불가능함 (재빌드 필요).
- 구현 복잡도가 높음 (라이브러리 의존성).

결론적으로, 2024년 현재 시점에서 정적인 데이터셋에 대해 여전히 Standard Bloom Filter를 쓰고 있다면, 그건 게으름입니다. 이제 더 효율적인 도구를 손에 쥘 때가 되었습니다.

---

**References:**
- Original Article: [https://arxiv.org/abs/2201.01174](https://arxiv.org/abs/2201.01174)
- Hacker News Discussion: [https://news.ycombinator.com/item?id=46658128](https://news.ycombinator.com/item?id=46658128)
