---
title: "Bloom Filter의 시대는 끝났나? Binary Fuse Filter 심층 분석"
description: "Bloom Filter의 시대는 끝났나? Binary Fuse Filter 심층 분석"
pubDate: "2026-01-22T01:55:22Z"
---

시스템 설계를 하다 보면 "이 데이터가 DB에 존재하는지 확인해라, 단 디스크 I/O는 최소화해야 한다"라는 요구사항을 자주 마주칩니다. 지난 수십 년간 엔지니어들의 모범 답안은 항상**Bloom Filter**였습니다. 구현하기 쉽고, 라이브러리도 많으니까요.

하지만 고성능 시스템, 특히 대규모 데이터를 다루는 스토리지 엔진 레벨에서는 이야기가 달라지고 있습니다. 오늘은 Bloom Filter보다 더 작고, 더 빠르며, 심지어 생성 속도까지 개선된**Binary Fuse Filter**에 대해 깊이 파헤쳐보겠습니다. arXiv에 올라온 논문 [Binary Fuse Filters: Fast and Smaller Than Xor Filters](https://arxiv.org/abs/2201.01174)를 바탕으로, 왜 우리가 이 기술에 주목해야 하는지 정리해 드립니다.

## 1. Probabilistic Data Structure의 진화: Bloom에서 Fuse까지

먼저 맥락을 짚어봅시다. 우리는 왜 확률적 자료구조(Probabilistic Data Structure)를 사용할까요? 정답은**False Positive(오탐)를 허용하는 대신 메모리를 획기적으로 아끼기 위함**입니다. "없다"는 확실하게 보장하고, "있다"는 "있을 수도 있다"라고 답하는 구조죠.

***Bloom Filter:**고전입니다. 여러 해시 함수를 써서 비트 배열을 세팅합니다. 하지만 이론적 한계(Shannon limit) 대비 약 44%의 오버헤드가 있습니다. 또한, 해시 함수가 여러 개라 메모리 접근이 랜덤하게 발생해 CPU Cache Miss를 유발하기 쉽습니다.
***Cuckoo Filter:**삭제(delete)가 가능하고 공간 효율이 더 좋지만, 여전히 최적은 아닙니다.
***XOR Filter:**획기적이었습니다. Bloom Filter보다 약 15~20% 더 작습니다. 하지만 필터를 생성(Construction)하는 과정이 복잡하고 느리다는 단점이 있었죠.

그리고 등장한 것이 바로**Binary Fuse Filter**입니다.

## 2. Binary Fuse Filter: 무엇이 다른가?

이 논문의 핵심 주장은 간단합니다.**"XOR Filter의 공간 효율성을 유지하면서, 생성 속도를 2배 이상 끌어올리고, 조회 속도도 희생하지 않겠다"**는 것입니다.

### 압도적인 공간 효율성
데이터를 저장하는 데 필요한 이론적 최소 비트 수(Lower bound)가 있다고 칠 때, 각 필터들의 오버헤드는 다음과 같습니다.

***Bloom Filter:**~44% 오버헤드
***XOR Filter:**~23% 오버헤드
***Binary Fuse Filter:****~13% 오버헤드**

심지어 조회 속도를 아주 약간만 희생하면 오버헤드를**8%**까지 줄일 수 있다고 합니다. 수 페타바이트 단위의 데이터를 다루는 시스템에서 10% 이상의 메모리 절감은 엄청난 비용 차이를 만들어냅니다.

### 동작 원리: Peeling과 Segment
기술적으로 들어가 보자면, Fuse Filter는 XOR Filter와 마찬가지로 그래프 이론의**Peeling**기법을 사용합니다. 하지만 핵심적인 차이는 데이터를 처리하는 구조에 있습니다.

기존 XOR Filter는 전체 데이터셋에 대해 매핑을 시도하다 보니 실패 확률이 있고, 이를 해결하기 위한 연산이 무거웠습니다. 반면, Fuse Filter는 데이터셋을 여러 개의 작은**Segment**로 나누어 처리합니다. 

이 Segment 구조 덕분에:
1.**Cache Locality:**데이터가 작게 쪼개져 L1/L2 캐시에 쏙 들어갑니다. 이는 필터 생성 속도를 비약적으로 높여줍니다.
2.**병렬성:**각 Segment를 독립적으로 처리할 수 있어 최신 CPU의 성능을 온전히 활용할 수 있습니다.

## 3. Production 환경에서의 의미 (Principal Engineer's View)

이 논문을 읽으면서 제가 현업에서 느낀 시사점은 다음과 같습니다.

### "Immutable"이라는 제약 조건
Binary Fuse Filter(그리고 XOR Filter)의 가장 큰 특징이자 단점은**Immutable(불변)**이라는 점입니다. 한 번 만들면 새로운 데이터를 추가(Add)할 수 없습니다. Bloom Filter나 Cuckoo Filter는 동적으로 데이터를 넣을 수 있죠.

하지만 생각해보십시오. 우리가 Bloom Filter를 사용하는 대다수의 고성능 시나리오, 예컨대**LSM-Tree 기반의 DB (RocksDB, Cassandra 등)**에서 SSTable은 어차피 불변(Immutable) 파일입니다. 디스크에 한 번 쓰면 끝이죠. 이런 Use Case에서는 `add()` 기능이 전혀 필요 없습니다.

### Build Time이 왜 중요한가?
"어차피 한 번 만들어서 읽기만 할 건데 생성 속도(Build time)가 중요한가요?"라고 물을 수 있습니다. 중요합니다. 

대규모 데이터 파이프라인이나 DB의 Compaction 과정에서 필터를 생성하는 시간은 전체 Write Throughput에 병목이 될 수 있습니다. 논문에 따르면 Binary Fuse Filter는 XOR Filter보다 생성 속도가 2배 이상 빠릅니다. 이는 시스템의**Write Stall**을 줄이는 데 직접적인 기여를 합니다.

## 4. 커뮤니티의 반응과 논쟁

이 기술에 대해 엔지니어링 커뮤니티에서는 흥미로운 논의들이 오가고 있습니다.

***"Bloom Filter는 이제 레거시인가?"**: 많은 전문가들이 Immutable 데이터셋에 대해서는 "그렇다"고 동의합니다. Rust나 C++ 같은 시스템 언어 생태계에서는 이미 `ublast`나 `ribbon filter` 같은 구현체들이 Bloom Filter를 대체하고 있습니다.
***구현 복잡도**: Bloom Filter는 주니어 개발자도 30분이면 짜지만, Fuse Filter는 구현 난이도가 꽤 높습니다. 검증된 라이브러리 사용이 필수적입니다.
***Ribbon Filter와의 비교**: RocksDB는 최근 Ribbon Filter를 도입했습니다. Fuse Filter와 유사한 목표를 가지지만, 메모리 레이아웃 측면에서 또 다른 트레이드오프가 있습니다. 하지만 범용적인 성능 면에서 Fuse Filter는 여전히 강력한 경쟁자입니다.

## 5. 결론: 언제 도입해야 할까?

만약 여러분이 Python으로 간단한 웹 서버를 짜고 있다면 그냥 내장된 `set`이나 Redis를 쓰세요. 굳이 이걸 도입할 필요 없습니다.

하지만 다음과 같은 상황이라면**Binary Fuse Filter**검토는 필수입니다.

1.**Static Dataset:**데이터가 자주 변하지 않거나, 일별/월별로 배치 업데이트되는 경우.
2.**Memory Constraint:**임베디드 장비나, 램 용량이 비용과 직결되는 클라우드 인스턴스를 운영 중인 경우.
3.**High Throughput:**초당 수십만 건 이상의 조회가 발생하는 검색/광고 시스템.

**Verdict:**
Binary Fuse Filter는 단순한 학술적 호기심의 산물이 아닙니다. 이는 하드웨어의 특성(Cache Hierarchy)을 극한으로 활용한**Modern Engineering의 정수**입니다. 여러분의 시스템이 'Read-Heavy'하고 데이터가 'Immutable'하다면, Bloom Filter를 걷어내고 Fuse Filter를 끼우는 것만으로도 인프라 비용을 10% 이상 아낄 수 있을 것입니다.

---

**References:**
*   Original Paper: [Binary Fuse Filters: Fast and Smaller Than Xor Filters](https://arxiv.org/abs/2201.01174)
*   Hacker News Context: [Discussion](https://news.ycombinator.com/item?id=46658128)
