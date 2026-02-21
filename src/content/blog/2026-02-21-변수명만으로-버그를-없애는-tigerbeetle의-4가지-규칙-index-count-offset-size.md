---
title: "변수명만으로 버그를 없애는 TigerBeetle의 4가지 규칙 (Index, Count, Offset, Size)"
description: "변수명만으로 버그를 없애는 TigerBeetle의 4가지 규칙 (Index, Count, Offset, Size)"
pubDate: "2026-02-21T03:04:23Z"
---

개발자 경력이 15년을 넘어가면서 깨달은 재미있는 사실이 하나 있습니다. 주니어 시절에는 복잡한 알고리즘이나 아키텍처 때문에 밤을 샜다면, **시니어, 아니 프린시펄 레벨이 되어서는 '멍청한 실수(Stupid bugs)'와 싸우는 시간이 더 많아진다는 점입니다.**

머릿속에 있는 로직은 완벽한데, 손가락이 코드로 옮길 때 미묘하게 어긋납니다. 섀도잉(Shadowing)된 변수를 잘못 참조하거나, 배열 인덱스를 1 차이로 틀리는(Off-by-one) 그런 실수들 말이죠. Rust 같은 언어가 강력한 타입 시스템으로 메모리 안전성을 보장해주지만, **'논리적 인덱싱 오류'** 까지 컴파일러가 잡아주지는 못합니다.

최근 고성능 금융 데이터베이스인 **TigerBeetle** 팀이 그들의 블로그에 이 문제를 해결하기 위한 아주 흥미로운 컨벤션을 공유했습니다. 거창한 기술이 아니라, **'변수 이름을 짓는 규칙'** 만으로 버그를 예방하는 방법론입니다. 꽤 인상 깊어서 제 시각을 섞어 정리해 봅니다.

## 타입 시스템의 한계와 '이름 짓기'의 미학

우리는 흔히 "타입이 곧 명세(Specification)이자 증명(Proof)"이라고 말합니다. 하지만 타입 시스템이 닿지 않는 사각지대가 존재합니다. 바로 **인덱싱(Indexing)** 과 관련된 영역입니다.

`usize` 타입의 변수 두 개가 있을 때, 하나는 배열의 인덱스고 하나는 배열의 길이일 수 있습니다. 타입 체커 입장에서는 둘 다 같은 정수형일 뿐이니, 이 둘을 더하든 빼든 막을 방법이 없습니다. 여기서 그 유명한 **Off-by-one error** 가 탄생합니다.

TigerBeetle 팀은 이 문제를 해결하기 위해 **Index, Count, Offset, Size** 라는 네 가지 엄격한 명명 규칙을 도입했습니다.

### 1. Element Space: Index와 Count

배열의 **'요소(Item)'** 를 다룰 때는 무조건 이 두 단어만 사용합니다.

- **Index:** 특정 아이템을 가리키는 위치 포인터 (0부터 시작)
- **Count:** 아이템의 총개수

여기서 중요한 불변식(Invariant)은 `index < count`입니다. 코드를 읽을 때 변수명 뒤에 `_index`나 `_count` 접미사가 붙어 있으면, 잘못된 연산이 시각적으로 튀어 오르게 됩니다. 예를 들어 `xxx_index + yyy_count` 같은 연산이 보이면 "잠깐, 인덱스에 개수를 더한다고?"라며 본능적인 거부감을 느끼게 되는 것이죠.

### 2. Byte Space: Offset과 Size

시스템 프로그래밍, 특히 로우 레벨에서는 타입이 있는 배열(`[]T`)과 로우 바이트(`[]u8`)를 오가야 할 때가 많습니다. 이 두 공간(Space)을 혼동하면 재앙이 일어납니다. 그래서 바이트 단위에는 별도의 용어를 씁니다.

- **Offset:** 바이트 단위의 위치 (Index의 대응 개념)
- **Size:** 바이트 단위의 크기 (Count의 대응 개념)

### 3. 금지어: Length

개인적으로 가장 공감했던 부분인데, **TigerBeetle은 `length`라는 단어 사용을 피합니다.**

이유는 **'모호함(Ambiguity)'** 때문입니다. Rust에서 `String::len()`은 바이트 크기를 리턴하지만, Python의 `len()`은 유니코드 코드 포인트(글자 수)를 리턴합니다. `len`이나 `length`라고 쓰여 있으면 "이게 바이트 크기야, 아니면 요소 개수야?"라고 한 번 더 생각해야 합니다. 이 1초의 망설임, 혹은 잘못된 가정이 버그를 낳습니다.

## 코드로 보는 차원 분석 (Dimensional Analysis)

이 규칙이 실제로 어떻게 작동하는지 TigerBeetle의 `NodePool` 코드를 통해 살펴보겠습니다.

```zig
pub fn release(pool: *NodePool, node: Node) void {
    // ... (생략)
    const node_offset =
        @intFromPtr(node) - @intFromPtr(pool.buffer.ptr);
        
    const node_index =
        @divExact(node_offset, node_size);
        
    assert(!pool.free.isSet(node_index));
    pool.free.set(node_index);
}
```

위 코드를 보면 물리학의 **차원 분석(Dimensional Analysis)** 이 떠오르지 않나요?

- `node_offset` (거리) / `node_size` (단위 크기) = `node_index` (순서)

변수명만 봐도 수식이 논리적으로 타당한지 검증이 됩니다. 만약 누군가 실수로 `node_offset / node_count`라고 썼다면, 코드 리뷰어는 변수명만 보고도 "바이트 오프셋을 개수로 나눈다고? 단위가 안 맞잖아!"라고 즉시 지적할 수 있습니다.

## 시각적 정렬 (Visual Alignment)

TigerBeetle 스타일(TigerStyle)의 또 다른 특징은 **'Big Endian Naming'** 입니다. 중요한 명사(Subject)를 앞에 두고, 수식어(Qualifier)를 뒤에 붙이는 방식입니다.

- `source`
- `source_words`
- `source_index`

그리고 대칭되는 개념의 변수명 길이를 맞추려고 노력합니다. 예를 들어 `source`와 `target`처럼 말이죠. 이렇게 하면 코드가 시각적으로 정렬(Align)되어, 패턴에서 벗어난 버그가 눈에 확 띕니다.

```zig
source_index += marker.literal_word_count;
target_index += marker.literal_word_count;
```

위 코드에서 만약 `target_index` 줄에 `source_index`가 섞여 있다면, 글자 길이와 모양이 달라서 위화감이 들 겁니다.

## 마치며: 모래알이 모여 사막이 된다

솔직히 처음 이 글을 읽었을 때는 "변수명 가지고 너무 유난 떠는 거 아닌가?" 싶을 수도 있습니다. 하지만 저는 이 접근 방식에 **전적으로 동의합니다.**

엔지니어링의 복잡도가 임계치를 넘어가면, 인간의 인지 능력만으로는 모든 상태를 추적할 수 없습니다. 그때 우리를 구해주는 건 천재적인 두뇌가 아니라, **실수를 원천 차단하는 지루한 습관(Convention)** 들입니다.

- `length` 대신 `count`와 `size`를 구분해 쓰는 것.
- 인덱스와 오프셋을 명확히 분리하는 것.

이런 작은 '모래알' 같은 규칙들이 쌓여서, TigerBeetle처럼 결함 없는 거대한 '사막(Dune)'을 만드는 것이겠죠. 당장 내일 코드 리뷰부터 팀원들에게 `length` 대신 `count`와 `size`를 써보자고 제안해볼 생각입니다. 모호함을 없애는 비용은 0에 가깝지만, 그로 인해 예방할 수 있는 버그의 비용은 무한대니까요.

**Reference:**
- [Index, Count, Offset, Size - TigerBeetle Blog](https://tigerbeetle.com/blog/2026-02-16-index-count-offset-size/)
- [Hacker News Discussion](https://news.ycombinator.com/item?id=47058584)
