---
title: "C와 LLVM 없이 eBPF 프로그래밍하기: Whistler와 Common Lisp의 기묘한 만남"
description: "C와 LLVM 없이 eBPF 프로그래밍하기: Whistler와 Common Lisp의 기묘한 만남"
pubDate: "2026-03-27T03:55:45Z"
---

현업에서 eBPF를 다뤄본 엔지니어라면 누구나 공감할 만한 고통이 있습니다. 커널 공간에서 실행될 C 코드를 작성하고, 특정 버전의 clang과 LLVM 툴체인으로 컴파일한 뒤, 다시 유저 공간에서 Go나 Rust로 로더(loader)를 작성하는 과정 말입니다. 이 지루한 컨텍스트 스위칭과 장황한 빌드 파이프라인은 eBPF의 강력함을 반감시키는 주범이었습니다.

그런데 최근 아주 흥미로운 프로젝트를 하나 발견했습니다. Common Lisp REPL에서 실시간으로 eBPF를 컴파일하고 커널에 주입하는 [Whistler](https://github.com/atgreen/whistler)라는 툴입니다. 솔직히 처음엔 '또 Lisp 매니아들의 장난감이 하나 나왔군' 하고 넘기려 했지만, 아키텍처를 뜯어볼수록 감탄이 나왔습니다.

### 매크로 확장을 이용한 우아한 컴파일러

Whistler의 핵심은 Lisp의 매크로 확장(Macroexpansion) 단계에서 eBPF 바이트코드를 생성한다는 점입니다. 다음 코드를 보시죠.

```lisp
(with-bpf-session ()
  (bpf:map counter :type :hash :key-size 4 :value-size 8 :max-entries 1)
  (bpf:prog trace (:type :kprobe :section "kprobe/__x64_sys_execve" :license "GPL")
    (incf (getmap counter 0))
    0)
  (bpf:attach trace "__x64_sys_execve")
  (loop (sleep 1)
        (format t "execve count: ~d~%" (bpf:map-ref counter 0))))
```

이 짧은 코드가 완전한 eBPF 프로그램이자 유저 공간 애플리케이션입니다. `bpf:` 접두사가 붙은 폼들은 SBCL(Steel Bank Common Lisp)이 코드를 컴파일하는 매크로 확장 시점에 이미 eBPF 바이트코드로 변환되어 리터럴 바이트 배열로 박힙니다. 즉, 런타임에는 디스크에 오브젝트 파일을 떨굴 필요 없이 메모리 상에서 바로 커널로 로드되는 것이죠. 피드백 루프가 말도 안 되게 짧아집니다.

### 하나의 구조체, 두 개의 세계

제가 가장 높게 평가하는 부분은 커널과 유저 공간 사이의 데이터 구조 동기화입니다. 기존에는 C에서 정의한 구조체를 Go에서 파싱하기 위해 수동으로 바이트 오프셋을 계산하거나 cgo에 의존해야 했습니다. Whistler는 `whistler:defstruct` 하나로 이 문제를 우아하게 해결합니다.

BPF 측에서는 컴파일 타임 오프셋을 활용한 직접적인 메모리 로드/스토어로 변환되고, 유저 공간 측에서는 동일한 필드명을 사용하는 디코딩 함수가 자동 생성됩니다. 수동으로 바이트를 파싱할 일이 아예 사라집니다.

게다가 `import-kernel-struct`를 통해 실행 중인 커널의 BTF(BPF Type Format)를 직접 읽어옵니다. 무겁고 관리하기 까다로운 `vmlinux.h` 파일이나 커널 헤더가 전혀 필요 없다는 뜻입니다. 이건 정말 실무에서 겪는 가장 큰 페인 포인트를 정확히 짚어낸 설계입니다.

### 현업 엔지니어의 시선: 이거 프로덕션에 쓸 수 있나?

아키텍처는 훌륭하지만, 냉정하게 말해 당장 프로덕션의 관측성 에이전트를 Common Lisp로 전면 재작성할 회사는 거의 없을 겁니다. 생태계와 채용의 문제가 가장 크니까요.

작성자도 이 점을 정확히 인지하고 있습니다. 그래서 Whistler에는 흥미로운 기능이 하나 포함되어 있습니다. 바로 Polyglot 유저 공간 지원입니다. Lisp로 작성한 프로브 코드를 컴파일할 때 `--gen go`나 `--gen rust` 플래그를 주면, BPF 측과 완벽히 일치하는 구조체 레이아웃을 가진 코드를 생성해 줍니다. 즉, BPF 로직은 Lisp의 강력한 REPL 환경에서 실시간으로 테스트하며 개발하고, 실제 프로덕션 배포용 로더는 팀에 익숙한 Go나 Rust로 작성할 수 있다는 의미입니다.

### Hacker News의 반응과 나의 결론

Hacker News의 한 유저는 "이건 정말 멋지다. 너드 스나이핑(nerd sniped) 당할 위기다"라고 평했습니다. 전적으로 동의합니다. 실용성을 떠나서, 언어의 경계를 허물고 커널 프로그래밍에 REPL 주도 개발을 도입했다는 것 자체만으로도 엔지니어의 가슴을 뛰게 만듭니다.

**결론적으로:** Whistler는 eBPF 개발 경험(DX)이 나아가야 할 방향을 제시하는 훌륭한 개념 증명입니다. 당장 Cilium이나 Tetragon을 대체할 수는 없겠지만, 복잡한 커널 트레이싱 로직을 프로토타이핑하거나 내부 툴을 빠르게 만들 때 이보다 강력한 무기는 찾기 힘들 겁니다. 기존의 장황한 C 코드 툴체인에 지쳤다면, 주말을 투자해 이 Lisp 토끼굴에 빠져보는 것을 강력히 추천합니다.

---
### References
- **Original Article:** https://atgreen.github.io/repl-yell/posts/whistler/
- **Hacker News Thread:** https://news.ycombinator.com/item?id=47495190
