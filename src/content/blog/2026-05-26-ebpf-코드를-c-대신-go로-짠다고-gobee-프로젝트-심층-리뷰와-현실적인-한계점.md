---
title: "eBPF 코드를 C 대신 Go로 짠다고? gobee 프로젝트 심층 리뷰와 현실적인 한계점"
description: "eBPF 코드를 C 대신 Go로 짠다고? gobee 프로젝트 심층 리뷰와 현실적인 한계점"
pubDate: "2026-05-26T08:10:30Z"
---

최근 몇 년간 eBPF는 클라우드 네이티브 생태계의 치트키로 자리 잡았습니다. 네트워크 패킷 필터링부터 보안, 옵저버빌리티까지 커널을 수정하지 않고도 강력한 기능을 구현할 수 있게 해주었죠. 하지만 여전히 개발자들을 괴롭히는 장벽이 하나 있습니다. 유저스페이스(Userspace) 툴링은 Go로 훌륭하게 구축되어 있지만, 커널 스페이스 코드는 결국 "이제 C로 짜세요"라는 벽에 부딪힌다는 것입니다.

최근 Hacker News에서 눈길을 끄는 프로젝트를 발견했습니다. 바로 Go 언어로 eBPF 프로그램을 작성하게 해주는 `gobee`입니다. 15년 차 엔지니어로서 새로운 툴이 나오면 일단 의심부터 하고 보는 성격이지만, 이 프로젝트가 접근한 방식은 꽤나 흥미롭습니다. 오늘은 이 `gobee`가 어떻게 동작하는지, 그리고 과연 프로덕션에 도입할 만한 가치가 있는지 파헤쳐 보겠습니다.

## 어떻게 Go를 커널에서 돌리는가: Transpilation의 마법

결론부터 말하자면, `gobee`는 Go 코드를 직접 eBPF 바이트코드로 컴파일하지 않습니다. 대신 Go의 엄격한 서브셋(Subset)을 BPF C 코드로 트랜스파일(Transpile)합니다. 

왜 굳이 이런 우회로를 택했을까요? Go 컴파일러(`gc`)에는 LLVM 기반의 BPF 백엔드가 없기 때문입니다. 이를 밑바닥부터 새로 구현하는 것은 수년이 걸리는 대공사입니다. 반면 Rust 생태계의 `Aya` 프레임워크가 성공할 수 있었던 이유는 `rustc`가 이미 LLVM을 기반으로 동작하기 때문이죠. `gobee`는 영리하게도 코드를 C로 변환한 뒤, 이미 성숙할 대로 성숙한 `clang`의 BPF 백엔드를 재사용하는 방식을 택했습니다. 이를 통해 CO-RE(Compile Once - Run Everywhere)와 BTF 같은 최신 eBPF 기능들을 공짜로 얻을 수 있었습니다.

다음은 execve 시스템 콜을 추적하는 간단한 Go 코드와, `gobee`가 이를 변환한 C 코드입니다.

```go
// Your input (Go)
//go:build ignore
package main
import "github.com/boratanrikulu/gobee/bpf"

//bpf:license GPL
type Event struct {
	Pid  uint32
	Comm [16]byte
}

var Events = bpf.RingBuf[Event]{
	MaxEntries: 4096,
}

//bpf:section tracepoint/syscalls/sys_enter_execve
func OnExec(ctx *bpf.ExecveEnterCtx) bpf.TpReturn {
	e, ok := Events.Reserve()
	if !ok {
		return bpf.TpOk
	}
	e.Pid = bpf.GetCurrentPid()
	bpf.GetTaskComm(&e.Comm)
	Events.Submit(e)
	return bpf.TpOk
}
```

```c
// What gobee emits (BPF C)
#include "vmlinux.h"
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_core_read.h>

char _license[] SEC("license") = "GPL";

struct Event {
	__u32 Pid;
	__u8 Comm[16];
};

struct {
	__uint(type, BPF_MAP_TYPE_RINGBUF);
	__uint(max_entries, 4096);
} Events SEC(".maps");

SEC("tracepoint/syscalls/sys_enter_execve")
int OnExec(struct trace_event_raw_sys_enter *ctx) {
	struct Event *e = bpf_ringbuf_reserve(&Events, sizeof(struct Event), 0);
	if (!e) return 0;
	
	e->Pid = (__u32)(bpf_get_current_pid_tgid() >> 32);
	bpf_get_current_comm(&e->Comm, 16);
	bpf_ringbuf_submit(e, 0);
	return 0;
}
```

## 시니어 엔지니어의 시선: 이것은 진짜 Go가 아니다

코드를 보면 꽤 그럴싸합니다. 특히 유저스페이스와 커널스페이스가 동일한 타입 정의(struct)를 공유할 수 있다는 점은 엄청난 메리트입니다. 맵(Map)에서 데이터를 꺼낼 때마다 바이트 배열을 파싱하며 고통받던 기억을 떠올려보면, 타입 세이프(Type-safe)한 Go 바인딩을 자동 생성해 준다는 건 정말 매력적이죠.

하지만 냉정하게 평가해 봅시다. **이 코드는 겉모습만 Go일 뿐, 실제로는 C입니다.** 

eBPF 커널 환경의 특성상 고루틴(Goroutine), 채널, 가비지 컬렉터 같은 Go의 강력한 무기들은 전혀 사용할 수 없습니다. 반복문(Loop)조차 Verifier가 증명할 수 있는 제한적인 형태로만 작성해야 하죠. HN의 한 유저가 지적했듯, eBPF 코드를 짠다면 어차피 커널의 C 데이터 구조체와 친숙해져야 합니다. 

HN 커뮤니티의 반응도 크게 엇갈렸습니다.

- **언어 통합의 장점:** 프론트엔드와 백엔드를 JavaScript로 통일하려는 시도처럼, 하나의 언어 생태계(Go) 안에서 모든 빌드 파이프라인을 유지할 수 있다는 점에 열광하는 의견이 있었습니다.
- **도구의 한계:** 반면, "eBPF의 진짜 어려움은 코드 작성이 아니라 까다로운 Verifier를 통과하는 것"이라는 날카로운 지적도 많았습니다. Parca-Agent를 개발하는 한 엔지니어는 "C조차도 때로는 너무 고수준이라 Verifier를 달래기 위해 직접 어셈블리로 내려가기도 한다"고 말했죠.

저 역시 후자의 의견에 강하게 동의합니다. eBPF에서 C는 단순히 레거시가 아니라, 커널과 소통하는 가장 네이티브한 언어입니다. 추상화 계층이 하나 더 늘어난다는 것은, 디버깅 시 트랜스파일러가 뱉어낸 C 코드와 원래의 Go 코드 사이를 오가며 멘탈 체조를 해야 한다는 뜻이기도 합니다.

## 결론: 장난감인가, 프로덕션 툴인가?

`gobee`는 굉장히 잘 만들어진 프로젝트입니다. 소스맵(Sourcemap)을 통해 Verifier 에러를 Go 소스코드의 라인 넘버로 매핑해주는 디테일이나, `bpfvet`을 내장해 커널 버전을 로드 타임에 체크해주는 기능은 작성자가 실제 eBPF 개발의 페인 포인트를 정확히 이해하고 있음을 보여줍니다.

그럼에도 불구하고 당장 내일 회사 프로덕션 코드에 도입하겠냐고 묻는다면, 제 대답은 '아니오'입니다. 

간단한 네트워크 모니터링 툴이나 사이드 프로젝트라면 훌륭한 선택이 될 수 있습니다. 하지만 복잡한 커널 로직을 다루고 극한의 최적화가 필요한 환경이라면, 굳이 C를 피하기 위해 Go의 서브셋이라는 제약에 갇힐 필요는 없습니다. 만약 메모리 안전성과 강력한 타입 시스템이 정말 필요하다면, 현재로서는 LLVM 네이티브로 동작하는 Rust의 `Aya`가 훨씬 더 성숙한 대안이라고 봅니다.

하지만 C가 지배하던 eBPF 생태계에 이런 신선한 접근법이 등장했다는 것 자체는 매우 환영할 만한 일입니다. 앞으로 Go 진영의 eBPF 툴링이 어떤 방향으로 진화할지 지켜보는 것도 꽤나 즐거운 관전 포인트가 될 것 같네요.

---

### References
- **Original Article:** https://github.com/boratanrikulu/gobee
- **Hacker News Thread:** https://news.ycombinator.com/item?id=48225338
