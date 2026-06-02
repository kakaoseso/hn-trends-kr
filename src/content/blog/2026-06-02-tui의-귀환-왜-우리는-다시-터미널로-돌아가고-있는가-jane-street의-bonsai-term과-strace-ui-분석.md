---
title: "TUI의 귀환: 왜 우리는 다시 터미널로 돌아가고 있는가? (Jane Street의 Bonsai_term과 strace-ui 분석)"
description: "TUI의 귀환: 왜 우리는 다시 터미널로 돌아가고 있는가? (Jane Street의 Bonsai_term과 strace-ui 분석)"
pubDate: "2026-06-02T15:44:30Z"
---

15년 넘게 백엔드와 인프라를 다루면서 수없이 많은 디버깅 툴을 써봤다. 그중에서도 `strace`는 시스템 콜을 추적할 때 없어서는 안 될 강력한 무기지만, 솔직히 말해 그 원시적인 텍스트 출력은 매트릭스 코드를 읽는 것만큼이나 고통스럽다.

최근 Jane Street에서 흥미로운 글을 하나 올렸다. `strace`의 출력을 인터랙티브하게 탐색할 수 있는 `strace-ui`와, 이를 구축하는 데 사용된 OCaml 기반의 TUI 프레임워크 `Bonsai_term`에 대한 이야기다. 단순한 툴 소개를 넘어, 왜 2026년인 지금 **TUI (Terminal UI)** 르네상스가 일어나고 있는지에 대한 꽤 날카로운 통찰을 담고 있다.

## strace-ui: 우리가 원했던 진짜 디버깅 경험

`strace`로 멀티스레드나 비동기 프로세스를 디버깅해 본 사람이라면 알 것이다. 수많은 로그가 터미널을 뒤덮고, 특정 파일 디스크립터(FD)나 PID를 추적하려면 매번 플래그를 바꿔가며 명령어를 다시 실행해야 한다.

Jane Street의 개발자 Ian Henry가 만든 `strace-ui`는 이 과정을 완전히 뒤집었다.



- **가독성:** PID에 짧은 ID를 할당하고, 버퍼를 단순 문자열이 아닌 hexdump로 렌더링한다.
- **인터랙티브 필터링:** 실행 중에 `h` 키를 눌러 관심 없는 시스템 콜을 숨기거나, 특정 FD를 참조하는 다음 시스템 콜로 점프(`<`, `>`)할 수 있다.
- **통합 컨텍스트:** `rt_sigprocmask`가 뭔지 기억나지 않는다면? `m`을 눌러 바로 man page를 띄운다. DNS 해상도 결과도 훨씬 직관적으로 파싱해 보여준다.

이런 툴을 보면 "왜 진작 없었지?"라는 생각이 든다. 답은 간단하다. 과거에는 이런 인터랙티브 TUI 환경을 만드는 것 자체가 엄청난 고역이었기 때문이다.

## Bonsai_term: 반응형 패러다임이 터미널을 만났을 때

Jane Street는 원래 웹 애플리케이션을 위해 `Bonsai`라는 OCaml 라이브러리를 개발해 쓰고 있었다. Elm에서 영감을 받은 이 프레임워크는 순수 함수형 상태 머신으로 컴포넌트를 구현한다.

```ocaml
module Dice = struct
  let faces = ...
  let component (graph @ local) =
    let face, set_face = Bonsai.state (List.hd_exn faces) graph in
    let%arr face and set_face in
    {%html|
      <div>
        You rolled a #{face}
        <button
          style=""
          on_click=%{fun _ ->
            let index = Random.int (List.length faces) in
            set_face (List.nth_exn faces index)}
        >Roll the dice</button>
      </div>
    |}
end
```

React의 Hook에 익숙하다면 위 코드가 꽤 친숙하게 느껴질 것이다. Bonsai의 핵심은 뷰뿐만 아니라 비즈니스 로직의 상태와 점진적 연산(incremental computation)을 효율적으로 관리한다는 점이다.

재미있는 점은 이 Bonsai 구조가 특정 프론트엔드에 종속되지 않는다는 것이다. UI란 본질적으로 상태에 따라 변하는 점진적 연산의 결과물일 뿐이다. 이 철학을 터미널로 가져온 것이 바로 `Bonsai_term`이다. 과거 C나 ncurses로 상태 관리를 하며 화면 깜빡임과 싸워본 엔지니어로서, 이런 선언적이고 타입 안전한 프레임워크의 등장은 쌍수를 들고 환영할 만한 일이다.

## AI와 스크린샷 테스트: TUI 폭발의 기폭제

가장 흥미로운 부분은 **AI 에이전트** 도구들이 TUI 생태계 확장에 불을 지폈다는 것이다. 최근 Claude Code 같은 AI 코딩 에이전트가 등장하면서, 무거운 IDE 플러그인보다 터미널 환경이 AI와 협업하기 훨씬 좋다는 사실이 증명되고 있다.



Bonsai_term은 터미널 화면 자체를 텍스트로 캡처하여 검증하는 **expect test** 프레임워크를 제공한다. AI가 코드를 작성하고, 테스트를 실행한 뒤, 텍스트로 된 스크린샷 상태를 직접 읽고 버그를 수정할 수 있다. 피드백 루프가 완벽하게 닫히는 것이다.

## Hacker News의 반응: Electron의 비대함에 대한 반작용

이 글이 Hacker News에 올라온 후, 커뮤니티의 반응은 매우 뜨거웠다. 가장 공감 갔던 의견들은 현재의 GUI 생태계에 대한 피로감에서 비롯되었다.

- **Electron의 저주:** 브라우저 엔진을 통째로 띄우는 데스크톱 앱들에 지친 개발자들이 많다. 한 유저는 k8s 관리를 위해 무거운 Lens 대신 `k9s`를 쓰면서 메모리 스왑 지옥에서 해방되었다고 극찬했다.
- **생태계의 진화:** Go 진영의 Charm (Lipgloss, Bubbletea) 생태계나, Kitty 이미지 프로토콜을 활용해 터미널에서 아바타와 이미지를 렌더링하는 Slack 클라이언트(`slk`) 사례도 언급되었다.
- **가짜 TUI의 아이러니:** 한 유저가 지적했듯, Anthropic의 Claude Code CLI조차 실제로는 Headless Electron 엔진을 띄워 초당 60프레임으로 터미널에 렌더링하는 기괴한 구조를 가지고 있다. 우리가 얼마나 웹 기술에 과도하게 의존하고 있는지 보여주는 씁쓸한 단면이다.

물론 "왜 굳이 폰트 크기 하나 마음대로 못 바꾸는 터미널을 고집하느냐"는 회의적인 시각도 존재한다. 하지만 TUI의 핵심은 화려함이 아니다. 키보드 중심의 극단적인 Latency 최적화, 그리고 SSH나 tmux 같은 기존 CLI 도구들과의 완벽한 조합성 (composability) 에 있다.

## 나의 결론: TUI는 프로 개발자의 '맞춤형 작업복'이다

솔직히 말해 모든 앱이 TUI 환경으로 돌아가야 한다고 생각하지는 않는다. 하지만 개발자 도구, 모니터링, 인프라 관리 영역에서는 이야기가 다르다.

과거에는 쓸만한 TUI 앱을 만드는 비용이 너무 컸다. 하지만 이제는 Bonsai_term이나 Go의 Bubbletea 같은 훌륭한 프레임워크가 있고, 심지어 AI가 초기 보일러플레이트를 10분 만에 짜주는 시대다. 무거운 GUI 프레임워크와 씨름하느니, 가볍고 빠르며 내 손끝에서 즉각적으로 반응하는 TUI 기반의 툴을 만드는 것이 훨씬 실용적이다.

Jane Street가 내부적으로 수많은 TUI 앱을 쏟아내고 있는 것은 우연이 아니다. 웹 기술이 모든 것을 집어삼킨 줄 알았던 2026년, 우리는 다시 터미널이라는 가장 강력하고 본질적인 인터페이스로 돌아오고 있다. 그리고 이번에는 과거보다 훨씬 더 우아한 무기를 들고 말이다.

### References
- Original Article: https://blog.janestreet.com/strace-ui-bonsai-term-and-the-tui-renaissance/
- Hacker News Thread: https://news.ycombinator.com/item?id=48365904
