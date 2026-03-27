---
title: "Redox OS의 실험적 행보: Namespace와 CWD를 Capability로 다루기"
description: "Redox OS의 실험적 행보: Namespace와 CWD를 Capability로 다루기"
pubDate: "2026-03-27T23:25:39Z"
---

15년 넘게 시스템 엔지니어링과 백엔드 아키텍처를 다루면서, 보안과 권한 관리(Access Control)는 언제나 골칫거리였습니다. 우리가 매일 사용하는 Linux나 Unix 기반 시스템은 기본적으로 ACL(Access Control List) 모델을 따릅니다. "너 누구야? 이 파일에 접근할 권한 있어?"를 매번 확인하는 방식이죠.

하지만 최근 Redox OS 블로그에 올라온 글 하나가 제 눈길을 끌었습니다. 바로 Namespace와 CWD(Current Working Directory)를 Capability로 취급하겠다는 내용입니다. 원문 데이터가 깨져서 보일 정도로 딥한 내용들이 오가고 있지만, 그 핵심 아키텍처 철학은 명확합니다.

### ACL의 한계와 Capability-Based Security

전통적인 Unix 권한 모델에서 가장 큰 문제는 권한이 '사용자'에게 부여된다는 점입니다. 프로세스는 사용자의 권한을 위임받아 실행되죠. 이로 인해 악명 높은 Confused Deputy Problem이 발생합니다. 시스템은 프로세스가 요청하는 작업이 정말로 안전한지 확신하지 못한 채, 단지 'root가 실행했으니 통과'시키는 식의 허점을 노출하게 됩니다.

반면 Capability-Based Security는 접근 방식을 완전히 뒤집습니다. 신분증을 검사하는 문지기 대신, 특정 문을 열 수 있는 위조 불가능한 '열쇠(Token)'를 프로세스에 직접 쥐어줍니다. 열쇠가 있으면 열고, 없으면 못 엽니다. 아주 직관적이고 단순하죠.

### Redox OS는 무엇을 바꾸었나?

Redox는 Rust로 작성된 마이크로커널 OS입니다. 이번 업데이트의 핵심은 파일 시스템의 근간이 되는 CWD와 Namespace를 단순한 상태나 경로 문자열이 아니라, 커널이 보장하는 Capability 객체로 만들었다는 것입니다.

- **CWD as a Capability:** 기존 Unix에서 CWD는 프로세스가 위치한 경로일 뿐입니다. 하지만 Redox에서는 특정 디렉토리에 대한 접근 권한을 캡슐화한 객체가 됩니다.
- **Namespace as a Capability:** 컨테이너 환경에서 격리를 위해 사용하는 네임스페이스 역시 하나의 권한 토큰으로 취급됩니다.

이게 왜 중요할까요? 프로세스를 샌드박싱할 때 복잡한 seccomp 룰이나 AppArmor 프로파일을 작성할 필요가 없어집니다. 그저 프로세스를 띄울 때, 루트 파일 시스템의 Capability 대신 특정 하위 디렉토리의 Capability만 CWD로 넘겨주면 끝입니다. 프로세스 입장에서는 그 CWD 밖으로 나갈 방법이 원천적으로 존재하지 않게 됩니다.

### 시니어 엔지니어의 시선

솔직히 말씀드리면, 수년 전 처음 Redox OS를 봤을 때는 그저 "Rust로 만든 장난감 OS" 정도로 생각했습니다. 하지만 이번 아키텍처 결정은 꽤나 뼈가 있습니다.

과거 Kubernetes 환경에서 복잡한 RBAC과 Pod Security Policy를 설정하며 밤을 새웠던 기억이 납니다. Linux 커널 위에 덕테이프를 발라가며 샌드박싱을 구현하는 현대의 방식은 분명 한계에 다다르고 있습니다. OS 커널 레벨에서 Capability 모델을 네이티브하게 지원한다면, 컨테이너 런타임이나 샌드박스 구현이 아찔할 정도로 단순해질 것입니다.

Hacker News의 반응도 흥미롭습니다. 한 유저(anon)는 이렇게 코멘트를 남겼더군요. "이제 Capability 구현체들이 꽤 많아졌고, 이 개념이 접근 제어와 샌드박싱을 정말로 단순화한다는 것을 증명하고 있다." 저 역시 이 의견에 전적으로 동의합니다. 이론적으로만 훌륭했던 개념들이 이제 실제 구현체로 증명되는 단계에 왔습니다.

### 결론

당장 내일 우리가 서비스하는 프로덕션 서버를 Linux에서 Redox로 바꿀 일은 없을 겁니다. 아직 생태계나 안정성 면에서 갈 길이 멀죠.

하지만 기술의 방향성이라는 측면에서, Redox OS의 이번 행보는 매우 가치 있는 실험입니다. 특히 보안 격리가 극도로 중요한 Edge Computing이나 WebAssembly 런타임 환경에서는 이러한 Capability 기반의 OS 아키텍처가 결국 표준으로 자리 잡을지도 모릅니다. 시스템 프로그래밍이나 보안 아키텍처에 관심이 있는 엔지니어라면, 이들이 이 개념을 어떻게 고도화해 나가는지 계속 지켜볼 필요가 있습니다.

### References
- Original Article: https://www.redox-os.org/news/nlnet-cap-nsmgr-cwd/
- Hacker News Thread: https://news.ycombinator.com/item?id=47546911
