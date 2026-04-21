---
title: "TypeScript로 작성된 CRDT 기반 그래프 데이터베이스: 장난감일까, 실전용일까?"
description: "TypeScript로 작성된 CRDT 기반 그래프 데이터베이스: 장난감일까, 실전용일까?"
pubDate: "2026-04-21T12:30:46Z"
---

그래프 데이터베이스(Graph Database)는 보통 무겁습니다. Neo4j 같은 녀석들을 운영해 본 분들이라면 샤딩과 메모리 관리의 고통을 잘 아실 겁니다. 그런데 최근 Hacker News에서 흥미로운 프로젝트를 하나 발견했습니다. 바로 TypeScript로 작성된, 그것도 Yjs 기반의 CRDT 위에서 돌아가는 실시간 협업 그래프 데이터베이스인 @codemix/graph 입니다.

처음엔 "TS로 DB를 만든다고? 성능 트랩 아닌가?" 싶었지만, 내부 구현과 유즈케이스를 뜯어보니 제법 영리한 접근이었습니다. 이건 대규모 인프라용 DB가 아니라, 엣지(Edge) 환경과 로컬 퍼스트(Local-first) 애플리케이션을 겨냥한 아주 날카로운 도구입니다.

## 왜 하필 TypeScript인가?

HN 댓글에서도 저와 똑같은 의문을 제기한 사람이 있었습니다. "TS가 빠르긴 하지만 그래프 DB의 요구사항을 감당할 수 있나?"

이에 대해 원작자인 Charles Pick은 명확하게 선을 긋습니다. 이 프로젝트는 수억 건의 노드를 저장하는 범용 DB가 아닙니다. 브라우저나 Cloudflare Workers 같은 환경에서 돌아가야 했고, End-to-End 타입 안정성이 필요했기 때문에 TS를 선택했다는 것이죠. 즉, 이 도구는 백엔드 인프라라기보다는 복잡한 도메인 모델을 다루기 위한 **In-memory Graph State Manager** 에 더 가깝습니다.

## Zod를 활용한 완벽한 타입 안정성

이 라이브러리의 가장 큰 장점은 스키마 정의부터 쿼리, 순회(Traversal)까지 타입이 100% 유지된다는 점입니다. Zod를 사용해 Vertex와 Edge를 정의하는 코드를 보겠습니다.

```typescript
import { Graph, GraphSchema, InMemoryGraphStorage } from "@codemix/graph";
import { z } from "zod";

const schema = {
  vertices: {
    User: {
      properties: {
        email: { type: z.email(), index: { type: "hash", unique: true } },
        name: { type: z.string() },
      },
    },
    Repo: {
      properties: {
        name: { type: z.string() },
        stars: { type: z.number() },
      },
    },
  },
  edges: {
    OWNS: { properties: {} },
    FOLLOWS: { properties: {} },
  },
} as const satisfies GraphSchema;

const graph = new Graph({ schema, storage: new InMemoryGraphStorage() });
```

보통 Gremlin 스타일의 순회 API를 사용할 때 런타임 캐스팅에 의존하는 경우가 많은데, 이 라이브러리는 `hasLabel("User")` 를 호출하는 순간 이후 체이닝되는 메서드들이 해당 스키마의 프로퍼티 타입을 정확히 추론합니다. HN에서 "메서드 체이닝 패턴은 유닛 테스트나 Mocking이 번거롭다"는 지적이 있었지만, 현재의 TS 생태계에서 완벽한 타입 안정성을 유지하려면 이 방식이 최선이라는 원작자의 의견에 저도 동의합니다.

## Yjs와 결합된 Local-first 협업

제가 가장 감탄한 부분은 저장소 계층의 추상화입니다. `InMemoryGraphStorage` 를 `YGraph` 로 교체하기만 하면 전체 그래프가 Yjs CRDT 문서로 동작합니다.

```typescript
import * as Y from "yjs";
import { WebsocketProvider } from "y-websocket";
import { YGraph } from "@codemix/y-graph-storage";

const doc = new Y.Doc();
const graph = new YGraph({ schema, doc });
const provider = new WebsocketProvider("wss://my-server", "graph-room", doc);
```

최근 로컬 퍼스트 아키텍처가 각광받으면서 상태 동기화 문제로 골머리를 앓는 팀이 많습니다. 단순한 JSON 트리 구조라면 일반적인 Yjs Map/Array로 충분하겠지만, 엔티티 간의 복잡한 관계(Reference)를 다뤄야 한다면 이야기가 다릅니다. 이 라이브러리는 그래프 구조 자체를 CRDT로 풀어내어 충돌 없는(conflict-free) 동기화를 제공합니다. 실시간 화이트보드나 노션(Notion) 같은 협업 툴의 기반 데이터 구조로 쓰기에 손색이 없습니다.

## LLM과 Cypher, 그리고 Knowledge Graph

최근 모든 기술이 LLM과 엮이는 것에 피로감을 느끼는 분들도 계실 겁니다. HN에서도 "왜 또 LLM 얘기냐"는 볼멘소리가 있었죠. 하지만 Cypher 쿼리를 파싱해서 내부적으로 Gremlin 스텝으로 변환해 실행하는 구조는 꽤나 실용적입니다.

```typescript
import { parseQueryToSteps, createTraverser } from "@codemix/graph";

const { steps, postprocess } = parseQueryToSteps(`
  MATCH (u:User)-[:OWNS]->(r:Repo)
  WHERE r.stars > 100
  RETURN u.name, r.name
  ORDER BY r.stars DESC
  LIMIT 10
`);
```

LLM 기반 에이전트에게 컨텍스트를 제공할 때, 그래프 형태의 Knowledge Graph는 환각(Hallucination)을 줄이고 다중 홉(n-hop) 추론을 돕는 강력한 무기입니다. 에이전트가 직접 Cypher 쿼리를 작성해 로컬 그래프 DB에서 필요한 정보만 쏙쏙 뽑아갈 수 있다면, MCP(Model Context Protocol) 서버를 구축하는 데 있어 이보다 가볍고 훌륭한 도구는 없을 겁니다.

HN 댓글 중 하나가 정확히 이 지점을 짚었습니다. "Cypher-over-Gremlin은 현명한 선택이다. LLM은 Cypher를 작성할 수 있으므로 MCP 관점에서 매우 유용하다."

## 총평 (Verdict)

결론적으로 `@codemix/graph` 는 수천만 건의 노드를 다루는 엔터프라이즈 백엔드 DB를 대체할 물건이 아닙니다. 이 도구를 전통적인 RDBMS나 네이티브 그래프 DB와 같은 선상에서 비교하면 실망할 수밖에 없습니다.

하지만 복잡한 관계형 데이터를 다루는 프론트엔드 애플리케이션, 실시간 협업 툴, 또는 엣지 환경에서 돌아가는 LLM 에이전트의 로컬 컨텍스트 저장소로는 매우 매력적인 선택지입니다. Alpha 버전이라는 점은 감안해야겠지만, 아이디어 자체는 최근 트렌드인 Local-first 및 Edge Computing을 정확히 관통하고 있습니다.

무거운 백엔드 인프라 없이 브라우저 단에서 강력한 관계형 데이터를 구축하고 동기화하고 싶다면, 한 번쯤 도입을 검토해 볼 만한 훌륭한 엔지니어링 결과물입니다.

---

### References
- **Original Article:** https://codemix.com/graph
- **Hacker News Thread:** https://news.ycombinator.com/item?id=47846946
