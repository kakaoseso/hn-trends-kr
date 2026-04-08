---
title: "JSIR: 구글이 자바스크립트 AST를 버리고 MLIR을 선택한 이유"
description: "JSIR: 구글이 자바스크립트 AST를 버리고 MLIR을 선택한 이유"
pubDate: "2026-04-08T06:19:28Z"
---

최근 LLVM Discourse에 꽤 흥미로운 RFC가 하나 올라왔다. 구글이 자바스크립트를 위한 고수준 IR(Intermediate Representation)인 JSIR을 오픈소스로 공개했다는 소식이다. 

15년 넘게 엔지니어로 구르면서 수많은 정적 분석 도구와 트랜스파일러를 뜯어봤지만, 자바스크립트 생태계는 유독 **AST** (Abstract Syntax Tree)에 병적으로 집착하는 경향이 있었다. Babel, ESLint, Webpack 등 우리가 매일 쓰는 도구들이 전부 ESTree 기반의 AST를 조작해서 돌아간다. 

하지만 컴파일러를 조금이라도 다뤄본 사람이라면 알 것이다. AST는 단순한 구문 변환이나 린팅에는 훌륭하지만, 복잡한 제어 흐름 그래프(CFG)를 그리거나 데이터플로우 분석을 수행하기에는 끔찍하게 불편한 구조다. 구글은 이 한계를 극복하기 위해 자바스크립트를 아예 MLIR 생태계로 끌고 들어왔다. 솔직히 말해, 처음 이 소식을 들었을 때는 '동적 타입 언어에 MLIR은 너무 오버엔지니어링 아닌가?' 싶었지만, 내부 구조를 뜯어보니 고개가 끄덕여졌다.

### 왜 AST로는 부족한가?

대부분의 자바스크립트 도구는 소스 코드를 생성해 내야 한다. 

- **Transpilation:** Babel처럼 최신 문법을 구형 브라우저용으로 변환.
- **Optimization:** Closure Compiler처럼 코드를 짧고 빠르게 압축.
- **Bundling:** Webpack처럼 여러 파일을 하나로 병합.

이 도구들은 코드를 다시 자바스크립트로 출력해야 하므로, 원본 코드의 형태를 유지하는 AST를 사용할 수밖에 없었다. 하지만 AST 위에서 데이터플로우 분석을 구현하려면 개발자가 직접 조잡한 CFG를 만들고 상태를 추적해야 한다. 

구글의 JSIR은 이 간극을 메운다. MLIR의 강력한 분석 프레임워크를 사용하면서도, `Source ↔ AST ↔ JSIR` 간의 무손실 양방향 변환(Round-trip)을 99.9% 이상의 성공률로 지원한다. 즉, 최적화나 분석은 IR에서 수행하고, 결과물은 다시 완벽한 자바스크립트 코드로 뽑아낼 수 있다는 뜻이다.

### JSIR의 핵심 설계: 고해상도 Round-trip

단순히 코드를 SSA(Static Single Assignment) 형태로 바꾸는 것은 쉽다. 문제는 다시 AST로 되돌릴 때 발생한다. 다음 자바스크립트 코드를 보자.

```javascript
1 + 2 + 3;
4 * 5;
```

이 코드가 JSIR로 변환되면 다음과 같은 형태가 된다.

```mlir
%1 = jsir.numeric_literal {1}
%2 = jsir.numeric_literal {2}
%1_plus_2 = jsir.binary_expression {'+'} (%1, %2)
%3 = jsir.numeric_literal {3}
%1_plus_2_plus_3 = jsir.binary_expression {'+'} (%1_plus_2, %3)
jsir.expression_statement (%1_plus_2_plus_3)

%4 = jsir.numeric_literal {4}
%5 = jsir.numeric_literal {5}
%4_mult_5 = jsir.binary_expression {'*'} (%4, %5)
jsir.expression_statement (%4_mult_5)
```

이 IR을 순진하게 다시 자바스크립트로 변환하면, 모든 SSA 값(`%1`, `%2` 등)이 지역 변수가 되어버려 원본과 전혀 다른 쓰레기 코드가 생성된다. 

구글 팀은 이 문제를 해결하기 위해 `jsir.expression_statement` 같은 구문 수준의 연산자를 식별하고, 해당 연산자의 **Use-def chain** 을 역추적하여 AST를 재구성하는 방식을 택했다. 컴파일러 이론을 실무에 아주 우아하게 적용한 사례다.

### 제어 흐름과 MLIR Region의 활용

내가 JSIR 설계에서 가장 감탄한 부분은 제어 흐름(Control Flow)을 처리하는 방식이다. 일반적인 저수준 IR은 `if`나 `while` 문을 단순히 블록과 `br`(branch) 명령어로 쪼개버린다. 이렇게 되면 원본 코드의 구조적 정보가 날아가서 다시 자바스크립트로 복원하기가 매우 까다로워진다.

JSIR은 MLIR의 Region 기능을 적극 활용하여 중첩 구조를 그대로 보존한다. `while` 문의 변환 결과를 보자.

```javascript
while (cond())
  x++;
```

```mlir
jshir.while_statement ({
  %cond_id = jsir.identifier {"cond"}
  %cond_call = jsir.call_expression (%cond_id)
  jsir.expr_region_end (%cond_call)
}, {
  %x_ref = jsir.identifier_ref {"x"}
  %update = jsir.update_expression {"++"} (%x_ref)
  jsir.expression_statement (%update)
})
```

여기서 주목할 점은 `while` 문의 조건부가 일반적인 SSA 값이 아니라 **Region** 으로 표현되었다는 것이다. `if` 문의 조건은 한 번만 평가되지만, `while` 문의 조건은 매 반복마다 평가되어야 하기 때문이다. 자바스크립트의 시맨틱을 MLIR이라는 범용 프레임워크 위에 얼마나 세밀하게 매핑했는지 엿볼 수 있는 대목이다.

### 해커뉴스 커뮤니티의 반응과 나의 생각

해커뉴스(Hacker News)에서도 이 RFC를 두고 흥미로운 토론이 오갔다. 

한 유저는 ARIA(AI가 작성하고 컴파일러가 소비하는 IR)와 JSIR을 비교하며, "JSIR은 인간이 작성한 코드를 이해하고 변환할 때 적합한 도구"라고 평가했다. 전적으로 동의한다. 구글이 논문에서 밝혔듯, Gemini LLM과 JSIR을 결합하여 난독화된 코드를 해제(Deobfuscation)하는 작업은 소스 레벨의 정보가 완벽히 보존되는 JSIR이 아니었다면 불가능했을 것이다.

또한, 컴파일러 엔지니어들 사이의 해묵은 논쟁인 'SSA vs Expression-based 구조'에 대한 지적도 있었다. 일부 컴파일러 엔지니어들은 SSA만이 진정한 최적화를 위한 유일한 자료구조라고 맹신하는 경향이 있다. 하지만 자바스크립트 같은 언어에서는 AST나 스택 기반 IR로도 충분히 훌륭한 최적화가 가능하다. 그럼에도 불구하고 구글이 MLIR을 선택한 이유는 단순한 최적화가 아니라, LLVM 생태계가 가진 거대한 분석 도구들을 자바스크립트 진영으로 가져오기 위함이라고 본다.

### 결론: 프로덕션 레벨인가?

현재 구글 내부에서는 Hermes 바이트코드 디컴파일과 코드 난독화 해제 등 꽤 무거운 작업에 JSIR을 실무 투입하고 있다. 99.9%의 Round-trip 성공률을 달성했다는 것은 이 도구가 단순한 장난감이 아님을 증명한다.

다만, 이것이 당장 내일 당신의 프론트엔드 빌드 파이프라인(Webpack이나 Vite)을 대체할 것이라 기대하진 마라. 상류(Upstream) LLVM에 병합되기에는 QuickJS나 Babel 같은 외부 의존성 문제가 아직 남아있다. 

하지만 SWC나 Biome, Rolldown 같은 차세대 자바스크립트 툴체인을 개발하는 엔지니어라면 이 프로젝트를 예의주시해야 한다. AST의 한계에 부딪혀 복잡한 타입 추론이나 데드 코드 제거(Tree-shaking)에 고통받고 있다면, JSIR이 보여준 MLIR 기반의 접근 방식이 훌륭한 돌파구가 될 수 있을 것이다.

---
**References:**
- [RFC: JSIR, a high-level IR for JavaScript (LLVM Discourse)](https://discourse.llvm.org/t/rfc-jsir-a-high-level-ir-for-javascript/90456)
- [Hacker News Discussion](https://news.ycombinator.com/item?id=47683376)
