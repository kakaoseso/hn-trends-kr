---
title: "코드 리뷰의 패러다임을 바꿀 수 있을까? AST 기반 Semantic Diff 툴 'Sem' 분석"
description: "코드 리뷰의 패러다임을 바꿀 수 있을까? AST 기반 Semantic Diff 툴 'Sem' 분석"
pubDate: "2026-03-08T12:01:35Z"
---

15년 넘게 엔지니어로 일하면서 수만 번의 PR(Pull Request)을 리뷰했지만, 여전히 가장 짜증 나는 순간이 있다. 누군가 거대한 파일에서 함수 위치를 위에서 아래로 옮겼거나, Prettier 설정이 바뀌어서 수백 줄의 들여쓰기가 변경되었을 때다. Git은 이럴 때 무자비하게 빨간 줄과 초록 줄을 수백 개씩 뱉어낸다. 우리는 라인(Line)이 아니라 함수나 클래스 같은 의미(Entity) 단위로 코드를 읽고 생각하는데, 도구는 여전히 70년대의 텍스트 라인 비교 방식에 머물러 있는 것이다.

최근 Hacker News에서 눈길을 끄는 프로젝트 하나를 발견했다. Git 위에 얹어서 사용하는 엔티티 레벨의 Semantic 버전 관리 도구, 바로 [Sem](https://github.com/ataraxy-labs/sem)이다.

## Sem은 어떻게 작동하는가?

단순히 정규표현식으로 함수 이름을 때려 맞추는 조잡한 스크립트가 아니다. Sem은 Rust로 작성되었으며, 내부적으로 `tree-sitter`를 사용해 코드를 AST(Abstract Syntax Tree)로 파싱한다. 현재 TypeScript, Python, Rust, Go, C++ 등 13개의 주요 프로그래밍 언어와 JSON, YAML 같은 구조화된 데이터 포맷을 지원한다.

가장 흥미로운 부분은 Sem이 변경 사항을 추적하는 **3단계 엔티티 매칭(Three-phase entity matching)** 알고리즘이다.

- **Exact ID match:** 변경 전후에 동일한 엔티티 ID가 존재하는지 확인한다.
- **Structural hash match:** AST 구조를 해싱하여 비교한다. 이름이 바뀌었더라도 구조가 같으면 Rename이나 Move로 인식한다. 공백이나 주석 같은 코스메틱 변경은 무시된다.
- **Fuzzy similarity:** 토큰의 80% 이상이 겹치면 높은 확률로 Rename된 것으로 간주한다.

이러한 구조 덕분에 단순한 라인 추가/삭제가 아니라, "`src/auth.ts` 파일에서 `validateToken` 함수가 추가되었고 `legacyAuth` 함수가 삭제되었다"는 식의 인간 친화적인 결과를 얻을 수 있다.

```bash
# 일반적인 Git Diff 대신 Sem을 사용한 결과
$ sem diff
┌─ src/auth/login.ts ──────────────────────────────────
│
│ ⊕ function validateToken [added]
│ ∆ function authenticateUser [modified]
│ ⊖ function legacyAuth [deleted]
│
└──────────────────────────────────────────────────────
```

또한 CI 파이프라인이나 AI 에이전트가 쉽게 파싱할 수 있도록 JSON 포맷의 출력도 지원한다.

```json
{
  "summary": {
    "fileCount": 2,
    "added": 1,
    "modified": 1,
    "deleted": 1,
    "total": 3
  },
  "changes": [
    {
      "entityId": "src/auth.ts::function::validateToken",
      "changeType": "added",
      "entityType": "function",
      "entityName": "validateToken",
      "filePath": "src/auth.ts"
    }
  ]
}
```

## 이것이 진짜 'Semantic'인가?

솔직히 말하자면, 나는 이런 류의 도구에 약간의 회의감을 가지고 있다. HN 스레드에서도 나와 비슷한 생각을 가진 엔지니어의 날카로운 지적이 있었다.

> "단순히 AST를 비교하는 것은 'Semantic diffing'이라기보다는 'Syntax-aware diffing'에 가깝다. 컴파일러는 AST가 달라도 최적화 패스를 거치면 결국 같은 머신 바이트코드를 생성할 수 있기 때문이다."

정확한 지적이다. 진정한 의미의 Semantic(의미론적) 변경을 추적하려면 프로그램의 제어 흐름(Control Flow)이나 데이터 흐름(Data Flow)까지 분석해야 한다. Sem은 AST 노드의 구조적 해시를 비교하는 구문(Syntax) 기반 도구다. 따라서 컴파일러 관점에서의 '의미'를 완벽히 이해한다고 볼 수는 없다.

하지만 실용적인 관점에서 보자면, 이 정도의 **Syntax-aware diff** 만으로도 엔지니어의 생산성은 비약적으로 상승한다. YAML이나 JSON 배열의 순서가 바뀐 것을 무시하거나, 함수 순서가 뒤섞인 리팩토링 PR을 리뷰할 때 Sem의 진가가 발휘될 것이다.

## AI 에이전트 시대의 필수재

이번 HN 논의에서 가장 내 무릎을 탁 치게 만든 코멘트는 따로 있었다.

> "라인 레벨 Diff는 인간이 모든 라인을 직접 작성할 때나 의미가 있었다. AI 에이전트가 한 세션에 수백 개의 변경을 커밋하는 지금, 우리는 어떤 라인이 바뀌었는지가 아니라 '어떤 함수'가 바뀌었는지를 알아야 한다."

최근 Cursor나 GitHub Copilot Workspace 같은 AI 코딩 어시스턴트를 도입하면서, PR의 규모가 예전과는 비교도 안 되게 커지고 있다. AI가 작성한 수백 줄의 보일러플레이트와 리팩토링 코드를 기존의 Git Diff로 리뷰하는 것은 고문에 가깝다. Sem이 제공하는 JSON 아웃풋과 엔티티 레벨의 변경 요약은, 다른 AI가 코드를 리뷰하거나 시스템의 Impact Analysis를 수행하는 데 있어 완벽한 중간 매개체가 될 수 있다.

## 총평: 프로덕션에 도입할 가치가 있는가?

결론부터 말하자면, **충분히 도입할 가치가 있다.**

Sem은 Git을 완전히 대체하려는 멍청한 시도를 하지 않았다. 대신 Git 위에 가볍게 얹어서 사용할 수 있는 CLI 도구이자 Rust 라이브러리로 스스로를 포지셔닝했다. `git2`와 `tree-sitter`를 결합해 빠르고 안정적으로 동작하며, 별도의 설정 없이 바로 저장소에서 실행할 수 있다는 점이 매우 훌륭하다.

물론 HN 스레드에서 언급된 것처럼 최근 비슷한 오픈소스 프로젝트들이 이름만 바꿔가며 쏟아지고 있는 피로감은 존재한다. (Beagle 같은 RocksDB 기반의 AST 저장소 프로젝트도 흥미로운 대안이다). 하지만 당장 내일 아침 거대한 레거시 코드를 리팩토링한 후배의 PR을 리뷰해야 한다면, 나는 주저 없이 `sem diff`를 터미널에 입력할 것이다.

우리는 더 이상 라인 단위로 코딩하지 않는다. 이제는 도구도 진화해야 할 때다.

---
**References**
- [Sem GitHub Repository](https://github.com/ataraxy-labs/sem)
- [Hacker News Discussion](https://news.ycombinator.com/item?id=47294924)
