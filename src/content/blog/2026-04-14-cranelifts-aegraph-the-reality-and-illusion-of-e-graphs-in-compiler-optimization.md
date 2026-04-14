---
title: "Cranelift's aegraph: The Reality and Illusion of E-graphs in Compiler Optimization"
description: "Cranelift's aegraph: The Reality and Illusion of E-graphs in Compiler Optimization"
pubDate: "2026-04-14T15:15:34Z"
---

최근 몇 년간 컴파일러 엔지니어링 씬에서 e-graph와 Equality Saturation은 마치 모든 최적화 문제를 해결해 줄 은구슬(Silver bullet)처럼 여겨졌습니다. 하지만 Wasmtime의 JIT 컴파일러인 Cranelift 팀이 프로덕션에 이를 도입하며 겪은 삽질기, 그리고 최종적으로 도달한 결론을 보면 현실은 언제나 이론보다 훨씬 더 복잡하고 흥미롭습니다.

오늘은 Cranelift의 메인테이너 Chris Fallin이 작성한 블로그 포스트를 바탕으로, 그들이 어떻게 고전적인 e-graph의 한계를 극복하고 **aegraph** (acyclic e-graph)라는 실용적인 구조를 만들어냈는지 파헤쳐 보겠습니다. 솔직히 말해서, 이 아키텍처는 제가 최근 몇 년간 본 컴파일러 미들엔드 설계 중 가장 영리한 타협안입니다.

## 컴파일러 개발자의 영원한 숙제: Pass-Ordering Problem

컴파일러 최적화를 작성해 본 엔지니어라면 누구나 Pass-ordering 문제에 부딪히게 됩니다. 예를 들어 RLE(Redundant Load Elimination) 패스를 돌리고 나면 GVN(Global Value Numbering)을 돌릴 건덕지가 생기고, GVN을 돌리고 나면 다시 RLE를 돌릴 기회가 생깁니다. 완벽한 최적화를 하려면 이 패스들을 더 이상 코드가 변하지 않을 때까지(Fixpoint) 무한정 핑퐁을 쳐야 합니다.

이 문제를 해결하기 위해 등장한 개념이 바로 여러 최적화 규칙을 동시에 적용할 수 있게 해주는 e-graph입니다. 하지만 Cranelift 팀이 초기에 `egg` 크레이트를 사용해 실험해 본 결과, 무려 23%의 컴파일 타임 오버헤드가 발생했습니다. 심지어 이건 Rewrite 규칙을 하나도 적용하지 않고 단순히 IR을 e-graph로 변환했다가 다시 복구하기만 했을 때의 수치입니다. 밀리초 단위의 지연 시간이 중요한 JIT 컴파일러에서 이런 오버헤드는 프로덕션 도입 불가 판정을 의미합니다.

## 뼈대 있는 Sea-of-Nodes 설계

오버헤드를 줄이기 위해 Cranelift는 IR 구조 자체를 뜯어고칩니다. 여기서 첫 번째 천재적인 발상이 나옵니다. 바로 순수 연산(Pure operators)만 제어 흐름 그래프(CFG)에서 분리해 Sea-of-nodes 로 띄우고, 메모리 접근이나 함수 호출 같은 부수 효과(Side-effects)가 있는 명령어는 CFG 뼈대에 그대로 남겨두는 **Sea-of-nodes-with-CFG** 구조입니다.

최근 V8 팀이 디버깅의 어려움 등을 이유로 순수 Sea-of-nodes 구조를 버리기로 결정한 것과 묘하게 오버랩되는 부분입니다. (여담이지만, Hacker News 스레드에서 Sea-of-nodes의 창시자인 Cliff Click은 V8 팀이 애초에 구조를 잘못 사용한 것이라고 꼬집기도 했죠.) Cranelift의 방식은 기존 CFG의 직관성을 유지하면서도, 순수 연산들에 대해 위치(Location)의 제약 없이 해시 콘싱(Hash-consing)과 최적화를 수행할 수 있는 완벽한 절충안입니다.

## aegraph의 핵심: Eager Rewriting과 Union Node

고전적인 e-graph가 느린 이유는 명확합니다. 새로운 동치(Equivalence) 관계가 발견될 때마다 그래프 전체를 순회하며 부모 노드들을 갱신하는 Rebuilding 과정이 필요하고, 이를 위해 무거운 양방향 포인터를 유지해야 하기 때문입니다.

Cranelift는 이 무거운 과정을 버리고 **aegraph** (acyclic e-graph)를 도입했습니다. 핵심은 두 가지입니다.

- **Union Node:** SSA 값 공간 자체를 암묵적인 e-graph로 사용합니다. 별도의 e-class와 e-node 인덱스를 나누는 대신, 여러 동치 표현식을 하나로 묶는 Union 노드를 SSA IR에 직접 추가했습니다.
- **Eager Rewriting:** 이게 진짜 핵심입니다. 노드를 생성한 후 나중에 일괄적으로 Rewrite를 돌리는 게 아니라, 노드를 생성하는 **즉시** 모든 Rewrite 규칙을 적용합니다. 

```python
# Cranelift의 Eager Rewriting 의사 코드
def canonicalize_and_rewrite(basic_block):
    for inst in basic_block:
        if is_pure(inst):
            if inst in hashcons_map:
                # 이미 존재하는 노드 재사용
            else:
                optimized = rewrites(inst) # 생성 즉시 최적화 규칙 적용
                union = join_with_union_nodes(inst, optimized) # 원본과 최적화 결과를 Union으로 묶음
                optimized_form[inst.def] = union
```

이 구조는 데이터를 덮어쓰지 않고 추가만 하는(Append-only) 불변 자료구조처럼 동작합니다. 따라서 사이클이 절대 발생하지 않으며(Acyclic), 단일 패스(Single pass)만으로 최적화가 끝납니다. 컴파일 타임을 갉아먹는 주범인 Rebuilding을 아예 아키텍처 레벨에서 제거해 버린 것입니다.

## HN 커뮤니티의 반응과 나의 시선

이 글이 Hacker News에 올라왔을 때 현업 컴파일러 개발자들의 반응이 꽤 흥미로웠습니다. 한 익명의 컴파일러 엔지니어는 뼈 때리는 지적을 남겼습니다.

> "솔직히 Pass-ordering 문제는 과장되어 있습니다. GVN과 Load elimination은 원래 하나의 패스로 묶는 게 자연스럽습니다. 진짜 골치 아픈 문제는 고수준 IR과 저수준 IR 사이의 정보 손실이나, Tail duplication과 루프 최적화 사이의 순서 충돌 같은 것들이죠. e-graph는 이런 진짜 문제들을 해결해 주지 않습니다."

저 역시 15년 넘게 시스템 엔지니어링을 해오며 이 의견에 깊이 공감합니다. 때로는 우리가 아키텍처의 결함을 우아한 수학적 모델(Equality Saturation)로 덮으려는 경향이 있습니다. 

원작자인 Chris Fallin조차도 글 말미에 "가장 놀라운 결론은 다중 값 표현(Multi-value representation) 자체는 생각보다 중요하지 않았다는 것"이라고 고백합니다. 결국 Cranelift가 얻은 엄청난 성능 향상의 진짜 비결은 e-graph의 마법이 아니라, 탄탄하게 설계된 Sea-of-nodes 구조와 Scoped elaboration을 통한 영리한 메모이제이션이었습니다.

## 결론: 프로덕션 레벨의 엔지니어링이란 이런 것

Cranelift의 aegraph 설계는 이론적 이상향(순수 e-graph)을 프로덕션의 현실(컴파일 속도와 메모리 제약)에 맞게 깎고 다듬어낸 엔지니어링의 마스터클래스입니다. 

만약 당신이 HLS(High-Level Synthesis)나 초정밀 정적 분석 툴을 만들고 있다면 전통적인 e-graph가 맞을 수 있습니다. 하지만 Wasmtime이나 브라우저 엔진처럼 속도가 생명인 JIT 환경이라면, Cranelift가 보여준 Acyclic e-graph와 Eager rewriting의 조합이 현재로서는 가장 완벽한 해답이라고 생각합니다. 

기술의 본질은 논문의 화려한 수식이 아니라, 제약 조건 속에서 동작하는 시스템을 만들어내는 데 있다는 것을 다시 한번 느끼게 해주는 훌륭한 사례입니다.

---

**References:**
- **Original Article:** https://cfallin.org/blog/2026/04/09/aegraph/
- **Hacker News Thread:** https://news.ycombinator.com/item?id=47717192
