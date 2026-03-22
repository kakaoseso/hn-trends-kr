---
title: "Ruby JIT 전쟁의 새로운 국면: ZJIT은 어떻게 YJIT을 뛰어넘었나 (Load-Store 최적화 심층 분석)"
description: "Ruby JIT 전쟁의 새로운 국면: ZJIT은 어떻게 YJIT을 뛰어넘었나 (Load-Store 최적화 심층 분석)"
pubDate: "2026-03-22T00:02:55Z"
---

Ruby 성능 최적화는 항상 뜨거운 감자다. 15년 넘게 백엔드와 인프라를 다루면서 수많은 언어의 JIT 컴파일러가 발전하는 과정을 지켜봤지만, 동적 타입 언어인 Ruby에서 JIT을 제대로 구현하는 것은 완전히 다른 차원의 난제다. 최근 YJIT이 기본 JIT으로 자리 잡으며 큰 진전이 있었지만, ZJIT의 추격이 심상치 않다.

최근 Rails at Scale 블로그에 올라온 글을 보며 꽤 흥미로운 최적화 기법을 발견했다. ZJIT이 특정 마이크로벤치마크(`setivar`)에서 YJIT을 2배 이상 앞섰다는 소식이다. 인터프리터와 비교하면 무려 25배나 빠르다. 오늘은 단순한 벤치마크 수치를 넘어, ZJIT이 컴파일러 레벨에서 어떻게 중복된 객체 로드와 스토어를 제거했는지 그 내부를 파헤쳐보자.

## JIT 컴파일러와 잉여 인스트럭션의 딜레마

코드를 작성하다 보면 무의미하게 중복된 변수 할당이나 참조가 발생하기 마련이다. 개발자가 명시적으로 중복을 작성하지 않더라도, JIT 컴파일러가 인터프리터의 Opcode를 중간 표현체(IR)로 번역하는 과정에서 필연적으로 중복된 메모리 접근 인스트럭션이 생성된다.

ZJIT 팀은 이 문제를 해결하기 위해 HIR(High-level Intermediate Representation) 파이프라인에 `optimize_load_store`라는 새로운 최적화 패스를 추가했다.



위 그래프에서 볼 수 있듯, 이 단일 최적화 패스가 추가된 기점으로 ZJIT(노란색)의 성능이 YJIT(초록색)을 극적으로 역전했다.

## Load-Store 최적화의 핵심 원리

ZJIT의 HIR은 인스턴스 변수와 객체 Shape을 다루기 위해 `LoadField`와 `StoreField` 인스트럭션을 사용한다. 가장 단순한 형태의 중복 스토어(Redundant Store) 예제를 보자.

```ruby
class C
  def initialize
    value = 1
    @a = value
    # 이 스토어는 중복이므로 HIR에서 제거되어야 한다
    @a = value
  end
end
```

이런 코드를 최적화하기 위해 ZJIT은 가벼운 **Abstract Interpretation** 방식을 채택했다. 알고리즘의 핵심은 Basic Block 내의 인스트럭션을 순회하면서 객체, 오프셋, 값으로 구성된 트리플(Triple)을 해시맵에 캐싱하는 것이다.

- **LoadField:** 캐시에 이미 해당 객체와 오프셋의 값이 있다면 인스트럭션을 삭제하고, 기존 캐시된 값을 참조하도록 변경한다.
- **StoreField:** 캐시에 동일한 트리플이 있다면 불필요한 스토어이므로 인스트럭션 자체를 날려버린다.

단순해 보이지만, 컴파일러 엔지니어링에서 이런 최적화가 언제나 발목을 잡는 지점은 바로 엣지 케이스다.

## 현실 세계의 장벽: Aliasing과 Side Effects

내가 이 글에서 가장 유심히 본 부분은 ZJIT 팀이 Aliasing과 Effectful operation을 어떻게 다루느냐였다. 컴파일러를 조금이라도 만져본 엔지니어라면 Aliasing이 얼마나 끔찍한 버그를 유발하는지 알 것이다.

```ruby
def multi_object_test
  x = C.new(1)
  y = x
  new_x_val = 2
  new_y_val = 3
  
  x.a = new_x_val
  y.a = new_y_val
  # y가 x와 같은 객체를 참조하므로, 이 스토어는 제거하면 안 된다!
  x.a = new_x_val
end
```

현재 ZJIT은 매우 보수적이고 안전한 길을 택했다. 동일한 오프셋을 조작하는 스토어가 발생하면 관련 캐시를 지워버리고, 객체의 상태를 변경할 가능성이 있는 임의의 Ruby 메서드 호출이나 C 확장 호출이 발생하면 전체 캐시를 플러시(Flush)해버린다. 가비지 컬렉터를 위한 `WriteBarrier` 인스트럭션 역시 마찬가지로 캐시 무효화를 트리거한다.

프로그램의 의미가 변하지 않음을 보장(Soundness)하기 위한 현실적인 타협이다. 성능을 극한으로 쥐어짜기 위해 여기서 Type-Based Alias Analysis (TBAA) 같은 고급 기법을 성급하게 도입했다가는, JIT 컴파일러의 고질병인 Type Confusion 취약점이 터지기 십상이다. 개인적으로 이 보수적인 접근 방식은 매우 현명한 엔지니어링 결정이라고 생각한다.

## 아키텍처적 결단: 가벼운 SSA의 선택

ZJIT 팀은 객체 레벨에서 완전한 SSA(Static Single Assignment) 폼을 구축하는 대신, 현재의 가벼운 형태를 유지하기로 했다. V8이나 Jikes RVM 같은 컴파일러의 역사를 아는 시니어 엔지니어라면 이 결정의 무게를 이해할 것이다.

Sea of Nodes 같은 복잡한 구조를 도입하면 최적화의 한계점은 높아지지만, 코드베이스의 복잡도와 유지보수 비용이 기하급수적으로 증가한다. ZJIT은 HIR의 구조적 단순함을 유지하면서도, 단일 패스 최적화를 통해 실질적인 성능 향상을 이끌어내는 실용주의를 보여줬다.

## Hacker News의 시선과 YJIT vs ZJIT

이번 소식에 대한 Hacker News 커뮤니티의 반응도 꽤 흥미로웠다. 특히 YJIT의 핵심 설계인 Basic Block Versioning(BBV)을 주도했던 Maxime Chevalier-Boisvert가 Shopify를 떠났다는 사실에 커뮤니티 일부가 우려를 표하기도 했다. 하지만 현재 Max Bernstein이 팀을 훌륭하게 이끌고 있으며, ZJIT의 발전 속도를 보면 그 우려는 기우에 불과해 보인다.

- **YJIT:** Basic Block Versioning이라는 다소 실험적이고 혁신적인 논문 기반의 접근을 취한다.
- **ZJIT:** 기존의 전통적인 컴파일러 최적화 기법들을 우직하게 구현해 나가는 방식을 택하고 있다.

이 두 가지 서로 다른 철학이 Ruby 생태계 안에서 선의의 경쟁을 펼치고 있다는 사실 자체가 개발자들에게는 엄청난 축복이다.

## 결론

ZJIT의 이번 Load-Store 최적화는 단순한 벤치마크용 트릭이 아니라, 견고한 컴파일러 엔지니어링의 결과물이다. 아직은 Basic Block 내부에 국한된 로컬 최적화에 머물러 있지만, 향후 Dead Store Elimination 같은 추가적인 패스가 더해지면 YJIT을 기본 JIT 자리에서 완전히 밀어낼 잠재력이 충분하다.

물론 프로덕션 환경의 거대한 Rails 앱에 당장 ZJIT을 전면 도입하기엔 아직 지켜볼 여지가 있다. 하지만 전통적인 컴파일러 최적화 기법이 동적 언어 JIT에 어떻게 성공적으로 이식될 수 있는지 보여준 훌륭한 사례임은 분명하다. Ruby의 성능 최적화 전쟁은 이제 막 새로운 챕터에 진입했다.

---
**References:**
- [How ZJIT removes redundant object loads and stores](https://railsatscale.com/2026-03-18-how-zjit-removes-redundant-object-loads-and-stores/)
- [Hacker News Discussion](https://news.ycombinator.com/item?id=47431625)
