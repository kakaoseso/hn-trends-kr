---
title: "Git 이전의 세계: Weave와 Interleaved Deltas의 우아함에 대하여"
description: "Git 이전의 세계: Weave와 Interleaved Deltas의 우아함에 대하여"
pubDate: "2026-05-27T22:23:00Z"
---

우리 대부분은 매일 Git을 사용한다. Git의 내부 구조, 즉 Content-addressable 스토리지와 Commit들의 DAG(Directed Acyclic Graph) 모델은 그 자체로 놀랍도록 단순하고 우아하다. 이 단순함 덕분에 우리는 수많은 복잡한 브랜칭과 병합 작업을 일상적으로 처리할 수 있다. 

하지만 시간을 조금 더 거슬러 올라가 보자. Git 이전에 Linux 커널 개발을 지탱했던 BitKeeper, 그리고 그보다 훨씬 전인 1970년대에 Marc J. Rochkind가 개발한 SCCS(Source Code Control System)는 완전히 다른 패러다임을 사용했다. 바로 **Weave** 또는 Interleaved Deltas라고 불리는 자료구조다.

솔직히 말해서, 15년 넘게 백엔드와 분산 시스템을 다뤄오면서 수많은 자료구조를 봤지만 Weave만큼 매력적이면서도 동시에 사람을 주눅 들게 만드는 녀석은 드물었다. 오늘 다뤄볼 주제는 바로 이 잊혀진, 하지만 현대의 CRDT(Conflict-free Replicated Data Type)와 Pijul 같은 차세대 VCS에서 다시 부활하고 있는 Interleaved Deltas에 대한 딥다이브다.

### Weave란 무엇인가? (The Weave Structure)

Git이 각 버전의 전체 스냅샷을 저장하고 필요할 때 Diff를 계산한다면, Weave는 파일의 **모든 버전의 변경 사항을 단일 스트림으로 교차(Interleave)시켜 저장** 한다. 

Weave는 파일 리비전을 재구성하기 위한 일련의 명령어(Instruction) 시퀀스다. 명령어는 크게 4가지로 나뉜다.

```go
type InstructionType int

const (
	Line InstructionType = iota
	BeginInsert
	BeginDelete
	End
)

// Instruction is a weave instruction.
type Instruction struct {
	Type InstructionType
	// Payload contains the instruction payload.
	// For Line instructions, it's the line number.
	// For other instructions, it's the version ID.
	Payload int
}
```

모든 `Line` 명령어는 반드시 `BeginInsert` 또는 `BeginDelete` 블록 안에 존재해야 한다. 여기서 흥미로운 점은 문자열(String)을 직접 저장하는 대신 전역 라인 풀(Global line pool)의 인덱스를 페이로드로 사용한다는 것이다. 이는 Weave의 크기를 줄이고 명령어를 균일하게 만든다.

하지만 Weave를 정말 까다롭게 만드는 것은 블록의 **오버랩(Overlap)** 이다. XML이나 JSON의 엄격한 트리 구조와 달리 Weave의 블록은 서로 겹칠 수 있다. 예를 들어 v1이 라인 1, 2를 추가하고, v2가 라인 3~6을 추가했는데, v3가 라인 2~4를 삭제한다면 어떻게 될까? v3의 Delete 델타는 v1과 v2의 Insert 델타에 걸쳐서 존재하게 된다. 

### 버전 복원의 핵심: Active Set과 Priority Queue

특정 리비전을 체크아웃(복원)하려면 먼저 해당 리비전에 기여하는 모든 델타의 집합, 즉 **Active Set** 을 계산해야 한다. 버전 간의 의존성 때문에 특정 버전을 활성화하면 그 부모 버전들도 모두 활성화되어야 하므로 간단한 그래프 순회(Graph traversal)가 필요하다.

진짜 마법은 `Reconstruct` 함수에서 일어난다. 위에서 언급한 '오버랩' 문제 때문에 단순히 Stack을 사용해서는 열려있는 블록들을 추적할 수 없다. 대신 Priority Queue(우선순위 큐)가 필요하다.

```go
// Reconstruct locates the lines enabled in the given version set.
func Reconstruct(
	instructions []Instruction,
	activeSet ActiveSet,
) (mask []bool, versions []VersionID, err error) {
	mask = make([]bool, len(instructions))
	openBlocksByVersion := make(map[VersionID]int)
	
	// activeBlocks are sorted by VersionID (ascending).
	var activeBlocks []Instruction

	for offset, instr := range instructions {
		switch instr.Type {
		case Line:
			top := activeBlocks[len(activeBlocks)-1]
			v := top.VersionID()
			if activeSet.Contains(v) && top.Type == BeginInsert {
				mask[offset] = true
				versions = append(versions, v)
			}
		case BeginInsert, BeginDelete:
			v := instr.VersionID()
			if instr.Type == BeginInsert || activeSet.Contains(v) {
				at, _ := activeBlock(v)
				activeBlocks = slices.Insert(activeBlocks, at, instr)
			}
			openBlocksByVersion[v] = offset
		// ... (End handling omitted for brevity)
		}
	}
	return mask, versions, nil
}
```

여기서 원본 글의 저자가 남긴 아주 훌륭한 연습문제가 하나 있다. 

> "왜 `Reconstruct`는 비활성화된(inactive) `BeginInsert` 명령어는 큐에 푸시하면서, 비활성화된 `BeginDelete` 명령어는 무시할까?"

이 질문에 대한 답이 Weave 구조를 이해하는 핵심이다. 비활성화된 Insert 블록이라도 큐에 유지해야 하는 이유는 **상대적인 위치(Relative ordering)** 를 보존하기 위해서다. 특정 라인이 현재 버전에선 비활성화되어 있더라도, 후속 버전들이 그 라인을 기준으로 삽입/삭제를 수행했을 수 있다. 공간적 앵커(Spatial anchor) 역할을 하는 것이다. 반면 비활성화된 Delete는 이미 삭제되지 않은 상태로 간주되므로 추적할 필요가 없다.

### LCS를 이용한 Delta 계산과 병합

새로운 변경사항을 Weave에 추가하려면 먼저 Diff를 계산해야 한다. 저자는 J. W. Hunt와 M. D. McIlroy의 고전적인 LCS(Longest Common Subsequence) 알고리즘을 사용하여 Diff Script를 추출했다. 

실무 환경이었다면 성능 문제로 Myers Diff나 Bram Cohen의 Patience Diff를 사용했겠지만, 개념 증명(PoC) 용도로는 LCS 기반의 2차원 동적 계획법(Dynamic Programming)도 훌륭한 선택이다. 추출된 Delta(Insert, Delete, Keep)는 `Interleave` 함수를 통해 기존 Weave 스트림과 병렬로 순회하며 새로운 `BeginInsert` / `BeginDelete` 블록으로 짜여 들어간다.

### Principal Engineer의 시선: 왜 지금 Weave를 알아야 하는가?

이 글을 읽고 "아, 옛날엔 저렇게 복잡하게 버전 관리를 했구나. Git 최고!" 하고 넘긴다면 엔지니어로서 큰 통찰을 놓치는 것이다.

최근 몇 년간 나는 분산 시스템에서의 동시 편집(Collaborative editing)과 상태 동기화 문제를 해결하기 위해 CRDT를 깊게 연구해 왔다. 흥미롭게도 CRDT의 텍스트 편집 알고리즘(예: Yjs, Automerge)을 뜯어보면 Weave의 구조와 소름 돋을 정도로 닮아있다.

Git의 스냅샷 모델은 직관적이고 빠르지만, 복잡한 Merge Conflict를 해결할 때는 문맥(Context)을 잃어버리는 경우가 많다. 반면 Pijul 같은 차세대 VCS나 CRDT 시스템은 Weave처럼 변경 사항 자체의 역사와 의존성을 데이터 구조 레벨에서 유지한다. 이 방식은 충돌(Conflict)을 본질적으로 더 우아하게 해결할 수 있는 잠재력을 제공한다. (아이러니하게도 SCCS 자체는 Merge를 제대로 지원하지 않았지만 말이다.)

### 마치며

오늘날 프로덕션 환경에서 SCCS나 순수한 형태의 Weave를 직접 구현해 사용할 일은 없을 것이다. 하지만 Marc J. Rochkind가 70년대에 고안한 이 아이디어는 소프트웨어 진화를 추적한다는 개념 자체를 정립했다. 

단순함에서 출발해 복잡한 문제를 해결하는 Git의 철학도 훌륭하지만, 데이터 구조 자체에 변경의 역사를 직조(Weave)해 넣는 이 고전적인 접근 방식은 오늘날의 분산 시스템 엔지니어들에게 여전히 강력한 영감을 준다. 기술의 유행은 돌고 돈다. 오래된 논문과 잊혀진 자료구조 속에 우리가 직면한 현대 아키텍처 문제의 해답이 숨어있을지도 모른다.

---

### References
- **Original Article:** [Interleaved Deltas (mmapped.blog)](https://mmapped.blog/posts/51-interleaved-deltas)
- **Hacker News Thread:** [Discussion on YCombinator](https://news.ycombinator.com/item?id=48280356)
