---
title: "Docker Sandbox의 숨겨진 MicroVM API: 리버스 엔지니어링 분석과 실무적 고찰"
description: "Docker Sandbox의 숨겨진 MicroVM API: 리버스 엔지니어링 분석과 실무적 고찰"
pubDate: "2026-05-21T16:12:53Z"
---

![We Reverse-Engineered Docker Sandbox's Undocumented MicroVM API](https://rivet.dev/_astro/image.DdB6C2th_ZvQbpH.webp)

최근 AI 코딩 에이전트들이 쏟아져 나오면서, 엔지니어들 사이에서 다시금 뜨거운 감자로 떠오른 주제가 있습니다. 바로 '신뢰할 수 없는 코드(Untrusted code)를 어디서, 어떻게 안전하게 실행할 것인가'입니다.

우리 모두 알고 있듯, 일반적인 Docker 컨테이너는 진정한 의미의 보안 샌드박스가 아닙니다. 컨테이너는 호스트의 커널을 공유하기 때문에, 권한이 탈취되거나 취약점이 발생하면 호스트 머신 전체가 위험해질 수 있습니다. AWS Lambda나 Fly.io 같은 서비스들이 컨테이너 대신 Firecracker 같은 MicroVM을 사용하는 이유도 바로 이 때문이죠.

그런데 최근 흥미로운 소식이 있었습니다. Docker가 AI 에이전트를 안전하게 실행하기 위해 'Docker Sandboxes'라는 기능을 출시했는데, 그 이면에 **문서화되지 않은 MicroVM API** 를 조용히 숨겨두었다는 것입니다. Rivet 팀이 이를 리버스 엔지니어링하여 공개한 글을 읽고, 15년 차 엔지니어의 시각에서 이 기술이 어떻게 동작하며 어떤 의미를 가지는지 딥다이브 해보겠습니다.

## 왜 컨테이너 대신 MicroVM인가?

이 API를 뜯어보기 전에, Docker가 왜 굳이 익숙한 컨테이너 기술을 두고 MicroVM을 도입했는지 짚고 넘어갈 필요가 있습니다.

Claude Code나 Gemini 같은 AI 에이전트들은 임의의 코드를 작성하고, 패키지를 설치하고, 파일을 수정해야 합니다. 이를 위해 `--dangerously-skip-permissions` 같은 플래그를 남발하며 컨테이너를 실행하는 것은 자살 행위나 다름없습니다.

- **Docker Container:** 호스트 커널을 공유(네임스페이스와 cgroups 사용). 빠르고 가볍지만 악성 코드 격리에는 부적합.
- **Docker Sandbox (MicroVM):** 별도의 커널을 구동. 가상 머신 수준의 격리를 제공하면서도 기존 VM보다는 훨씬 가벼움.

Docker는 10년 전 컨테이너 생태계를 통일했던 것처럼, 이번에는 로컬 인프라에서의 MicroVM 오케스트레이션을 통일하려는 야심을 품고 있는 것 같습니다.

## 숨겨진 API 파헤치기: 어떻게 동작하는가?

Docker는 `docker sandbox run claude ~/project`라는 단순한 CLI를 제공하지만, 그 밑단에서는 `sandboxd`라는 데몬이 백그라운드에서 MicroVM을 관리하고 있습니다.

이 데몬은 `~/.docker/sandboxes/sandboxd.sock`라는 유닉스 소켓을 통해 통신합니다. Rivet 팀이 밝혀낸 VM 생성 요청은 다음과 같습니다.

```bash
curl -X POST --unix-socket ~/.docker/sandboxes/sandboxd.sock \
  http://localhost/vm \
  -H "Content-Type: application/json" \
  -d '{"agent_name": "my-sandbox", "workspace_dir": "/path/to/project"}'
```

이 요청을 보내면 아래와 같은 응답이 돌아옵니다.

```json
{
  "vm_id": "abc123",
  "vm_config": {
    "socketPath": "/Users/you/.docker/sandboxes/vm/my-sandbox-vm/docker.sock",
    "fileSharingDirectories": ["/path/to/project"],
    "stateDir": "/Users/you/.docker/sandboxes/vm/my-sandbox-vm"
  },
  "ca_cert_path": "/Users/you/.docker/sandboxes/vm/my-sandbox-vm/proxy_cacerts/proxy-ca.crt"
}
```

### 아키텍처의 핵심: 격리된 Docker Daemon

제가 이 구조에서 가장 감탄한 부분은 `socketPath`입니다. 보통 호스트의 모든 컨테이너는 `/var/run/docker.sock`를 공유합니다. 소켓 권한만 얻으면 다른 모든 컨테이너를 제어할 수 있죠.

하지만 이 Sandbox 아키텍처에서는 **각 MicroVM마다 완전히 독립된 Docker 데몬** 이 부여됩니다. VM 내부에 격리된 데몬을 띄우고, 호스트에서는 `--host "unix://$VM_SOCK"` 플래그를 사용해 해당 VM 내부의 데몬과 직접 통신하는 방식입니다. 매우 깔끔하고 우아한 격리 방식입니다.

### 이미지 로딩과 네트워크의 한계

VM이 완전히 격리되어 있기 때문에, 호스트에 있는 Docker 이미지를 바로 사용할 수 없습니다. `docker save`로 이미지를 아카이빙한 뒤, 특정 VM의 소켓을 타겟으로 `docker load`를 해줘야 합니다.

```bash
# 호스트에서 빌드 및 아카이브
docker build -t my-image:latest .
docker save my-image:latest > /tmp/image.tar

# MicroVM 내부로 로드
docker --host "unix://$VM_SOCK" load < /tmp/image.tar
```

네트워크 처리 방식도 흥미롭습니다. 모든 아웃바운드 트래픽은 `host.docker.internal:3128`에 위치한 필터링 프록시를 거치게 됩니다. 네트워크 정책 강제를 위해 HTTPS 트래픽에 대해 MITM(Man-in-the-Middle) 방식을 사용하므로, 컨테이너 실행 시 `NODE_TLS_REJECT_UNAUTHORIZED=0`을 설정하거나 응답으로 받은 CA 인증서를 주입해야 합니다.

솔직히 현업 엔지니어 입장에서 이 프록시 구조는 프로덕션 환경에서 다루기 꽤나 까다롭고 귀찮은 포인트입니다. 하지만 '로컬 샌드박스'라는 목적을 생각하면 보안을 위한 합리적인 트레이드오프라고 봅니다.

## 총평: 왜 숨겼으며, 실무에 쓸만한가?

이 훌륭한 API를 Docker는 왜 공식 문서에서 숨겼을까요?

가장 큰 이유는 **플랫폼 의존성** 때문일 것입니다. 현재 이 기능은 macOS(Apple Virtualization.framework)와 Windows(Hyper-V)에서만 동작하며, 정작 서버 환경의 표준인 Linux는 지원하지 않습니다. Docker Desktop에 강하게 결합된 기능이기 때문에, 서버사이드 프로덕션용으로 섣불리 공개하기엔 부담스러웠을 것입니다.

그렇다면 이 기술은 그저 장난감일까요? 절대 그렇지 않습니다.

로컬 환경에서 AI 에이전트를 개발하거나, CI/CD 파이프라인 중 macOS/Windows 노드에서 신뢰할 수 없는 빌드 스크립트를 격리 실행해야 할 때 이 API는 엄청난 무기가 될 수 있습니다. Rivet 팀이 만든 [Sandbox Agent SDK](https://sandboxagent.dev)처럼 이 API를 래핑한 도구들이 발전한다면, 로컬 샌드박싱의 새로운 표준이 될 가능성도 충분합니다.

물론 서버 환경에서 대규모 멀티테넌트 샌드박스를 구축해야 한다면 여전히 Firecracker나 gVisor를 직접 다루는 것이 맞습니다. 하지만 개발자 경험(DX) 측면에서 Docker가 제공하는 이 마이크로VM 통합은 매우 훌륭한 시도이며, 향후 Linux 지원이 추가된다면 생태계에 꽤 큰 파장을 일으킬 것이라 확신합니다.

---

**References:**
- 원문 블로그: [We Reverse-Engineered Docker Sandbox's Undocumented MicroVM API](https://rivet.dev/blog/2026-02-04-we-reverse-engineered-docker-sandbox-undocumented-microvm-api/)
- Hacker News 토론: [https://news.ycombinator.com/item?id=48223693](https://news.ycombinator.com/item?id=48223693)
