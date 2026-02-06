---
title: "Tower of Fantasy Anti-Cheat Reversing: A BYOVD Toolkit That Was Never Even Loaded"
description: "Tower of Fantasy Anti-Cheat Reversing: A BYOVD Toolkit That Was Never Even Loaded"
pubDate: "2026-02-06T06:35:47Z"
---

최근 Hacker News와 기술 블로그들 사이에서 꽤나 충격적이면서도, 한편으로는 쓴웃음이 나오는 리버싱 사례가 화제가 되었습니다. 바로 **Tower of Fantasy(타워 오브 판타지)** 라는 게임의 안티치트 드라이버 분석기입니다.

보통 안티치트 드라이버라고 하면 VMProtect 등으로 떡칠된 난독화 코드를 떠올리지만, 이번 사례는 정반대였습니다. 심지어 이 드라이버는 게임이 실행될 때 로드조차 되지 않으면서, 시스템에 치명적인 백도어만 열어두고 있었습니다. 오늘은 이 '유령 드라이버'가 어떻게 완벽한 **BYOVD(Bring Your Own Vulnerable Driver)** 툴킷이 되었는지 기술적으로 파헤쳐 보겠습니다.

---

## 1. 시작은 계정 삭제였다

원작자인 Vespalec은 4년 전 만든 계정을 삭제하고 싶었을 뿐입니다. 하지만 웹 포털도 없고, 고객 지원도 없는 상황에서 계정을 지우려면 게임을 설치해야만 했습니다. 100GB가 넘는 게임을 다운로드하는 동안, 그는 습관적으로 설치 디렉터리를 뒤지기 시작했고 `GameDriverX64.sys`라는 커널 드라이버 파일을 발견합니다.

![Driver file in directory](https://vespalec.com/_astro/driver-directory.BRtOEZKm_k1drg.png)

보통 이런 파일은 IDA로 열면 가상화된 코드의 벽에 부딪히기 마련입니다. 하지만 놀랍게도, 이 파일은 아무런 난독화 없이 깨끗한 함수들을 그대로 노출하고 있었습니다.

![Driver entry in IDA](https://vespalec.com/_astro/ida.CDwpmWIX_297Hmn.png)

### 왜 난독화를 뺐을까? (HVCI의 역설)

이전 버전의 드라이버(`KSophon_x64.sys`)는 VMProtect로 보호되어 있었습니다. 그런데 왜 갑자기 보호를 풀었을까요? 범인은 Windows 11의 **HVCI(Hypervisor-Protected Code Integrity)** 입니다.

HVCI는 커널 메모리 페이지가 '쓰기 가능(Writable)'하면서 동시에 '실행 가능(Executable)'할 수 없도록 **W^X** 정책을 강제합니다. VMProtect의 패킹 방식은 이를 위반하기 때문에, HVCI가 켜진 최신 시스템에서는 드라이버 로드 자체가 실패합니다. 개발사는 이를 우회하기 위해 코드를 수정하는 대신, **그냥 보호 기능을 꺼버리는** 최악의 선택을 한 것으로 보입니다.

## 2. 보안? 그런 건 없습니다

이 드라이버는 커널 레벨에서 프로세스를 종료하거나 권한을 조작하는 강력한 기능을 제공합니다. 당연히 엄격한 인증이 필요하겠죠? 하지만 이들이 구현한 인증 메커니즘은 코미디 수준입니다.

### Magic Number 인증

모든 IOCTL 요청은 단 하나의 32비트 매직 넘버로 보호됩니다.

![Magic key check](https://vespalec.com/_astro/magic-key.BGgFfc0Q_Z8WkbG.png)

```c
if (InputBuffer->Magic != 0xFA123456)
    return STATUS_ACCESS_DENIED;
```

암호화 서명? 챌린지-리스폰스? 그런 건 없습니다. 바이너리만 열어보면 누구나 `0xFA123456`이라는 키를 알 수 있고, 이를 통해 커널 드라이버의 모든 기능을 제어할 수 있습니다.

## 3. 완벽한 멀웨어 툴킷 (BYOVD)

이 드라이버가 제공하는 기능은 공격자 입장에서는 '종합 선물 세트'나 다름없습니다. 크게 두 가지 치명적인 취약점이 존재합니다.

### 취약점 #1: 임의 프로세스 종료 (EDR 킬러)

IOCTL `0x222040`은 PID를 받아 해당 프로세스를 종료합니다. 문제는 내부적으로 사용하는 함수가 `ZwTerminateProcess`라는 점입니다.

```c
// 커널 모드 권한으로 프로세스 핸들을 획득
ZwOpenProcess(&Handle, GENERIC_ALL, &ObjectAttributes, &ClientId);
ZwTerminateProcess(Handle, 0);
```

윈도우 커널 API에서 `Nt` 접두사가 붙은 함수와 `Zw` 접두사가 붙은 함수는 큰 차이가 있습니다. `Zw` 함수는 **PreviousMode를 KernelMode로 설정** 하여 호출하기 때문에, 접근 권한 체크를 건너뜁니다. 즉, 이 드라이버를 통하면 PPL(Protected Process Light)로 보호받는 안티바이러스나 EDR 솔루션도 묻지도 따지지도 않고 종료시킬 수 있습니다.

### 취약점 #2: 임의 프로세스 보호 (셀프 방어)

반대로 IOCTL `0x222004`는 특정 PID를 '보호된 프로세스'로 등록합니다. `ObRegisterCallbacks`를 사용하여 해당 프로세스에 대한 핸들 접근 권한을 스트리핑(Stripping)해버립니다.

```c
// 권한 제거: VM_READ, VM_WRITE, TERMINATE 등
OperationInfo->Parameters->CreateHandleInformation.DesiredAccess &= 0xFFFFF587;
```

공격자는 자신의 멀웨어 프로세스를 여기에 등록함으로써, 보안 도구나 사용자가 작업 관리자로 강제 종료하는 것을 막을 수 있습니다. 심지어 이미 열려 있는 핸들조차 `ExEnumHandleTable`을 통해 소급적으로 권한을 박탈할 수 있습니다.

## 4. 가장 황당한 반전: "로드되지 않는다"

원작자가 PoC(개념 증명) 코드를 작성하고 실제 게임을 실행했을 때, 가장 충격적인 사실이 밝혀졌습니다.

**드라이버가 로드되지 않습니다.**

게임 프로세스는 보호받지 않고 있었고, 드라이버 파일은 그저 디스크에 존재할 뿐 메모리에 올라오지 않았습니다. 개발사는 HVCI 호환성 문제 때문인지, 아니면 다른 이유에서인지 드라이버 로드 기능을 뺐지만, **취약한 드라이버 파일 자체는 사용자 PC에 배포** 해버린 것입니다.

이는 공격자에게 최상의 시나리오입니다. 게임이 실행 중일 필요도 없이, 공격자는 디스크에 있는 이 서명된 드라이버를 로드하여 자신의 공격 도구로 활용할 수 있습니다.

## 5. Hacker News 및 업계 반응

이 글은 [Hacker News](https://news.ycombinator.com/item?id=46908671)에서도 뜨거운 반응을 얻었습니다. 특히 이미 이 드라이버가 랜섬웨어 그룹에 의해 악용되고 있다는 사실이 밝혀졌습니다.

- **실제 악용 사례:** 한 유저는 이 드라이버가 이미 'Interlock' 랜섬웨어 등에서 EDR을 무력화하는 데 사용되고 있다고 지적했습니다. (Fortinet 보고서 참조)
- **책임론:** "외과의사가 수술 능력이 없으면 메스를 들지 말아야 하듯, 안전한 커널 드라이버를 짤 능력이 없는 게임사는 커널에 손대지 말아야 한다"는 비판이 많은 공감을 얻었습니다.
- **OS의 역할:** 마이크로소프트가 OS 차원에서 안티치트를 제공해야 한다는 의견도 있었지만, 현실적으로 게임사들의 다양한 요구사항을 맞추기는 어렵다는 반론도 존재합니다.

## Principal Engineer's Verdict

이 사례는 **BYOVD(Bring Your Own Vulnerable Driver)** 공격이 왜 여전히 유효하고 위험한지를 적나라하게 보여줍니다. 공격자는 굳이 제로데이 취약점을 찾을 필요가 없습니다. 게임사가 친절하게 서명까지 해서 배포해 준 '만능 키'가 수백만 대의 PC에 잠들어 있으니까요.

기술적인 관점에서 볼 때, 이 드라이버는 '실수'의 수준을 넘어섰습니다.
1.  **HVCI 대응 실패:** 보안 기능을 끄는 것으로 대응함.
2.  **인증 부재:** 매직 넘버는 인증이 아닙니다.
3.  **배포 관리 실패:** 사용하지도 않는 위험한 커널 모듈을 사용자 PC에 방치함.

개발자로서 우리는 "코드가 동작하는가?"를 넘어 "이 코드가 악용되었을 때 어떤 파급력이 있는가?"를 항상 고민해야 합니다. 특히 Ring 0(커널) 레벨에서 노는 코드라면 더더욱 그렇습니다. Tower of Fantasy 팀은 이 드라이버를 즉시 제거하고, 이미 배포된 파일에 대해서도 MS 블록리스트 등록 등의 조치를 취해야 할 것입니다.
