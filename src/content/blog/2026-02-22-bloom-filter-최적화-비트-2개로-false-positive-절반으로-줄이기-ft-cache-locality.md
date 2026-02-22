---
title: "Bloom Filter 최적화: 비트 2개로 False Positive 절반으로 줄이기 (ft. Cache Locality)"
description: "Bloom Filter 최적화: 비트 2개로 False Positive 절반으로 줄이기 (ft. Cache Locality)"
pubDate: "2026-02-22T08:19:08Z"
---

데이터베이스 엔진이나 고성능 분산 시스템을 다루다 보면, 결국 마주치는 벽은 '알고리즘의 Time Complexity'가 아니라 **Memory Latency** 입니다. 아무리 $O(1)$이라도, 그게 L3 캐시 미스를 유발한다면 CPU 사이클 관점에서는 영겁의 시간이죠.

최근 FloeDB 팀에서 올린 블로그 포스트가 Hacker News에서 꽤 뜨거운 감자였습니다. **"Two Bits Are Better Than One"** 이라는 제목인데, 아주 단순한 아이디어로 Bloom Filter의 정확도를 2배 높이면서 성능 저하는 거의 없는 기법을 소개했습니다. 오늘은 이 기법을 뜯어보고, HN 커뮤니티의 날카로운 지적들까지 버무려 정리해 보겠습니다.

---

### The Problem: 이론과 현실의 괴리

교과서적인 Bloom Filter는 $k$개의 해시 함수를 사용해 비트 배열의 $k$군데 위치를 1로 세팅합니다. 이론적으로는 완벽합니다. $k$값을 조절해서 False Positive Rate(FPR)를 제어할 수 있으니까요.

하지만 **System Engineering** 의 세계에서는 이야기가 다릅니다.

1.  **Cache Miss:** $k$개의 비트 위치가 랜덤하게 퍼져 있다면, 최악의 경우 $k$번의 Cache Line 로딩이 필요합니다. 메모리 접근 비용이 비쌉니다.
2.  **Concurrency:** 멀티스레드 환경에서 여러 비트를 동시에 세팅하려면 atomic 연산이 $k$번 필요하거나, 락을 걸어야 합니다.

FloeDB 팀은 데이터베이스의 Hash Join 단계와 스토리지 엔진의 압축 해제 전 필터링(Pushdown)에 Bloom Filter를 사용합니다. 여기서 수십억 건의 row를 스캔해야 하는데, 기존의 $k=1$ 방식(성능 때문에 해시 1개만 사용)은 FPR이 10%가 넘어 효율이 떨어지는 문제가 있었습니다.

### The Solution: One `uint32`, Two Bits

이들이 찾아낸 해결책은 **"하나의 `uint32` 워드 안에 비트 2개를 다 때려 넣자"** 입니다.

기존 방식이 해시값 하나로 전체 비트 배열 중 한 곳을 찍었다면, 새로운 방식은 해시값 하나를 쪼개서 다음을 수행합니다:

1.  **Word Index:** 해시의 상위 비트를 사용해 배열의 어느 `uint32`에 접근할지 결정.
2.  **Bit 1 Offset:** 해시의 일부 5비트를 사용해 첫 번째 비트 위치 결정.
3.  **Bit 2 Offset:** 해시의 다른 5비트를 사용해 두 번째 비트 위치 결정.

코드로 보면 명확합니다.

```cpp
// FloeDB 구현체 발췌
void put(HashKey32 h) {
    const uint32_t idx = uint32Idx(h);
    // 하나의 해시(h)에서 두 개의 비트 위치를 추출
    const uint32_t mask = (1u << bitLoc1(h)) | (1u << bitLoc2(h));
    // 단 한 번의 Atomic OR 연산
    __sync_fetch_and_or(mBuf + idx, mask);
}
```

#### Why This Matters (Principal Engineer's View)

이 접근 방식의 핵심은 **Cache Locality** 와 **Atomic Operation** 의 최소화입니다.

*   **Single Memory Access:** 두 비트가 같은 `uint32` 안에 있으므로, CPU는 딱 한 번만 메모리(또는 캐시)를 읽으면 됩니다.
*   **Single Atomic Instruction:** `LOCK OR` 명령 하나로 두 비트를 동시에 세팅합니다. Lock-free 동시성 제어가 훨씬 가벼워집니다.

결과적으로 `put` 연산은 9.12 사이클에서 9.70 사이클로 약 6% 늘어났지만(무시할 수준), **FPR은 11.68%에서 5.69%로 절반 가까이 떨어졌습니다.**

테라바이트 단위의 테이블을 스캔할 때 FPR이 절반이 된다는 건, 불필요한 압축 해제(Decompression) 작업을 수십 GB 줄일 수 있다는 뜻입니다. 가성비가 미쳤죠.

![bloom_1vs2_bits](https://floedb.ai/hs-fs/hubfs/BlogCover/bloom_1vs2_bits.png?width=723&height=394&name=bloom_1vs2_bits.png)

---

### Hacker News: "그거 Blocked Bloom Filter잖아?"

역시 HN 형님들은 호락호락하지 않습니다. 이 글이 올라오자마자 댓글창에서는 "이거 휠(Wheel)을 재발명한 거 아니냐"는 토론이 이어졌습니다.

**1. Blocked Bloom Filter의 재발견**
한 유저는 이 방식이 사실상 **Blocked Bloom Filter** 의 변형이라고 지적했습니다. Blocked BF는 캐시 라인 크기(보통 64바이트)에 맞게 블록을 나누고, 그 안에서만 비트를 세팅하여 캐시 미스를 1회로 제한하는 기법입니다. FloeDB의 방식은 블록 크기를 `uint32` (4바이트)로 극단적으로 줄인 형태라고 볼 수 있습니다.

**2. 왜 `uint64`가 아니라 `uint32`인가?**
개인적으로도 의문이었고 HN에서도 나온 지적입니다. 64비트 레지스터가 일반적인 요즘 세상에 왜 굳이 `uint32`를 썼을까요? `uint64`를 쓰면 비트 밀도가 낮아져서 충돌(Collision) 확률이 더 줄어들 텐데 말이죠. 아마도 레거시 코드와의 호환성이나, 특정 SIMD 벡터화(AVX2 등)를 고려했을 때 32비트 정렬이 유리해서였을 것으로 추측됩니다만, 아쉬운 부분입니다.

**3. Atomics vs Partition Locks**
"Join을 할 거면 배치(Batch)로 처리가 가능할 텐데 굳이 Atomic을 써야 했나?"라는 의견도 있었습니다. Bloom Filter를 파티셔닝해서 각 파티션 별로 락을 걸고 Bulk Insert를 하는 게 더 빠를 수 있다는 거죠. 동의합니다. 하지만 구현 복잡도와 코드 유지보수 측면에서 `__sync_fetch_and_or` 한 줄로 끝내는 매력을 포기하긴 힘들었을 겁니다.

---

### My Verdict: Simple is Best

엔지니어링은 언제나 **Trade-off** 의 예술입니다.

이론적으로는 비트 간의 독립성(Independence)이 훼손되었으므로 수학적으로 '순수한' Bloom Filter보다는 효율이 떨어질 수 있습니다. 하지만 **CPU 아키텍처를 고려한 현실적인 성능** 은 압도적입니다. 복잡한 Cuckoo Filter나 XOR Filter를 도입하지 않고, 기존 코드에서 비트 마스킹 로직만 살짝 바꿔서 2배의 효율을 냈다는 점에 높은 점수를 주고 싶습니다.

**Key Takeaways:**
*   **Cache is King:** 알고리즘의 Big-O보다 메모리 접근 횟수가 더 중요할 때가 많습니다.
*   **Micro-optimization:** 수십억 건의 데이터를 다룰 때는 1나노초의 최적화가 수 시간의 배치 작업을 줄여줍니다.
*   **Don't Trust Default:** 라이브러리에 있는 기본 Bloom Filter(`k=3`, `k=7`)를 맹신하지 마세요. 여러분의 데이터 접근 패턴에 맞는 `k`와 구조를 직접 설계해야 합니다.

혹시 여러분의 시스템에서 Bloom Filter를 쓰고 있다면, 지금 당장 `k` 값과 메모리 접근 패턴을 확인해 보세요. 어쩌면 공짜 점심(Free Lunch)이 기다리고 있을지도 모릅니다.

**References:**
*   [Original Article: Two Bits Are Better Than One](https://floedb.ai/blog/two-bits-are-better-than-one-making-bloom-filters-2x-more-accurate)
*   [Hacker News Discussion](https://news.ycombinator.com/item?id=47046070)
