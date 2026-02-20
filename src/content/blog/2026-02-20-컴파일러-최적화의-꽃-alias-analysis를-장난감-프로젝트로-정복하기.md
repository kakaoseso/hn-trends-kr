---
title: "컴파일러 최적화의 꽃, Alias Analysis를 장난감 프로젝트로 정복하기"
description: "컴파일러 최적화의 꽃, Alias Analysis를 장난감 프로젝트로 정복하기"
pubDate: "2026-02-20T02:38:02Z"
---

안녕하세요. 오늘은 컴파일러 백엔드 최적화의 가장 골치 아픈, 하지만 가장 중요한 주제 중 하나인 **Alias Analysis(별칭 분석)** 에 대해 이야기해보려 합니다.

최근 Bernstein Bear 블로그에 올라온 'Toy Optimizer' 시리즈를 흥미롭게 보고 있는데, 이번에 올라온 **Type-based Alias Analysis (TBAA)** 편이 꽤 인상적이었습니다. 단순히 이론만 나열하는 게 아니라, 실제로 작동하는 파이썬 코드로 JIT 컴파일러나 정적 분석기가 어떻게 메모리 접근을 최적화하는지 보여줍니다.

시니어 엔지니어라면 한 번쯤 '이 코드가 왜 이렇게 느리지?' 하고 어셈블리를 까봤다가 불필요한 Load/Store가 남발되는 걸 보고 한숨 쉬어본 적이 있을 겁니다. 그 문제의 핵심에 바로 Alias Analysis가 있습니다.

## 왜 Alias Analysis가 필요한가?

이전 포스트에서 저자는 **Load-Store Forwarding** 을 구현했습니다. 쉽게 말해, 메모리에 값을 썼다가(Store) 바로 다시 읽는(Load) 경우, 메모리까지 가지 않고 레지스터 수준에서 값을 재사용하는 최적화입니다.

하지만 여기엔 치명적인 문제가 있습니다. 바로 **Aliasing** 입니다.

```python
# 예시 상황
p.x = 1
q.y = 2
return p.x
```

여기서 `p.x`는 여전히 1일까요? 만약 `p`와 `q`가 같은 객체를 가리키고 있다면(Aliasing), `q.y = 2`가 `p.x`의 값을 덮어썼을 수도 있습니다. 컴파일러가 `p`와 `q`가 서로 다른 메모리 영역이라는 것을 **증명** 하지 못하면, 안전을 위해 최적화를 포기하고 다시 메모리에서 읽어와야 합니다. 이게 성능 저하의 주범이죠.

## Type 정보로 힌트 얻기 (TBAA)

가장 단순한 해결책은 '오프셋'만 보는 겁니다. `p.x`는 오프셋 0번, `q.y`는 오프셋 8번이라면 둘은 겹치지 않겠죠. 하지만 오프셋이 같다면요?

여기서 **Type-based Alias Analysis (TBAA)** 가 등장합니다. "Array 객체와 String 객체는 절대 같은 메모리 공간을 공유하지 않는다"는 언어적 특성을 이용하는 것입니다.

이 글에서 소개하는 방식은 WebKit의 JavaScriptCore(JSC) 엔진을 만든 Fil Pizlo의 아이디어를 차용했습니다. 바로 **Hierarchical Heap Effect** 모델입니다.

### 힙을 계층적으로 나누기

힙 메모리 영역을 타입에 따라 트리 구조로 나눕니다.

- Any
  - Object
    - Array
    - String
  - Other

그리고 이 트리를 **DFS 순회 순서(Pre-order, Post-order)** 를 이용해 정수 범위(Range)로 매핑합니다. 이게 정말 천재적인 부분입니다. 복잡한 그래프 탐색 없이, 단순한 정수 비교만으로 포함 관계를 파악할 수 있으니까요.

- Any: [0, 3)
- Object: [0, 2)
- Array: [0, 1)
- String: [1, 2)

이제 두 포인터가 겹치는지 확인하는 `may_alias` 함수는 단순히 두 정수 범위가 겹치는지를 확인하는 `O(1)` 연산이 됩니다.

```python
class HeapRange:
    def overlaps(self, other: "HeapRange") -> bool:
        if self.is_empty() or other.is_empty():
            return False
        # 범위가 겹치는지 확인하는 단순 정수 비교
        return self.end > other.start and other.end > self.start
```

이 방식은 구현이 간단하면서도 강력합니다. 비트셋(Bitset)을 쓰는 방식도 있지만, 계층 구조를 표현하기엔 이 범위 기반 방식이 훨씬 직관적입니다.

## 실제 최적화 적용 결과

이 로직을 Load-Store 최적화 패스에 적용하면 드라마틱한 결과가 나옵니다.

```python
var0.info = Array
var1.info = String
store(var0, 0, 3)  # Array에 쓰기
store(var1, 0, 4)  # String에 쓰기
load(var0, 0)      # Array에서 읽기 -> 3으로 최적화 가능!
```

기존에는 오프셋 0번으로 같아서 최적화를 못 했겠지만, 이제는 `Array`와 `String`의 범위가 겹치지 않으므로 `var1`에 대한 쓰기가 `var0`의 값을 해치지 않는다는 것을 컴파일러가 확신할 수 있습니다.

## 더 나아가서: Allocation Site와 V8의 추억

저자는 여기서 멈추지 않고, **Allocation Site(할당 위치)** 정보를 활용하는 아이디어도 던집니다. 함수 내부에서 `malloc`으로 갓 생성된 객체는, 함수 인자로 넘어온 기존 객체들과 절대 겹칠 수 없다는 논리입니다.

이 부분에서 예전 V8 엔진의 IR인 **Hydrogen** 의 코드를 인용하는데, 개인적으로 향수에 젖게 하더군요. TurboFan으로 넘어가기 전 V8 코드는 C++로 짜인 교과서 같은 느낌이었습니다.

```cpp
// V8 Hydrogen 시절의 Alias 분석 로직
if (a->IsAllocate() || a->IsInnerAllocatedObject()) {
    // 새로 할당된 객체는 다른 할당이나 파라미터와 겹치지 않음
    if (b->IsAllocate()) return kNoAlias;
    if (b->IsParameter()) return kNoAlias;
}
```

이런 식의 'MustAlias', 'MayAlias', 'NoAlias' 3단 논법은 실무 컴파일러에서 아주 흔하게(그리고 유용하게) 쓰이는 패턴입니다.

## 현실적인 한계와 Trade-off

물론 이 'Toy Optimizer'는 한계가 명확합니다. 가장 큰 문제는 **함수 호출(Function Call)** 입니다. 함수 호출은 블랙박스라서, 그 함수가 내부에서 전역 메모리를 건드리는지, 힙을 엎어버리는지 알 길이 없습니다. 보수적인 컴파일러라면 함수 호출을 만나는 순간 캐싱해둔 모든 힙 정보를 날려버려야 합니다(Invalidate).

이를 해결하려면 함수에 Side-effect 정보를 어노테이션 하거나, 더 복잡한 Interprocedural Analysis가 필요합니다. 하지만 컴파일 시간과 런타임 성능 사이의 **Trade-off** 를 고려해야겠죠.

## 마치며

이 글은 'Toy'라는 제목을 달고 있지만, 내용은 결코 가볍지 않습니다. JIT 컴파일러나 고성능 런타임을 만드는 엔지니어들이 고민하는 **Precision vs Speed** 의 균형을 아주 잘 보여줍니다.

특히 타입 계층을 정수 범위로 매핑하여 Alias Check 비용을 최소화하는 기법은, 컴파일러뿐만 아니라 복잡한 권한 시스템이나 계층적 데이터 처리가 필요한 백엔드 시스템에서도 충분히 응용해볼 만한 패턴입니다.

여러분의 프로젝트에서는 불필요한 연산을 줄이기 위해 어떤 메타데이터를 활용하고 계신가요? 가끔은 코드의 '타입'이 생각보다 많은 힌트를 줄지도 모릅니다.
