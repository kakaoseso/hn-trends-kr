---
title: "BitLocker가 뚫렸다: YellowKey 제로데이와 TPM 보안의 환상"
description: "BitLocker가 뚫렸다: YellowKey 제로데이와 TPM 보안의 환상"
pubDate: "2026-05-14T04:45:11Z"
---

Windows 11이 출시될 때 Microsoft가 TPM 2.0을 강제하면서 내세운 명분을 기억하십니까? 보안이 그 이유였습니다. 저는 15년 넘게 인프라와 보안 아키텍처를 다뤄오면서, Pre-boot 인증 없는 하드웨어 종속 암호화의 실효성에 늘 의문을 품어왔습니다. 그리고 오늘, 그 의심이 현실이 되었습니다. 한 보안 연구자가 USB 스틱 하나만으로 BitLocker를 완전히 우회하는 데 성공했습니다.

## YellowKey: 너무나도 허술한 우회 기법
이번에 공개된 **YellowKey** 제로데이 익스플로잇은 그 방식이 충격적일 정도로 단순합니다. USB 드라이브의 'System Volume Information' 디렉토리에 'FsTx'라는 특정 폴더와 파일들을 복사해 넣기만 하면 됩니다.

그 후 USB를 꽂은 상태에서 Shift 키를 누른 채 시스템을 다시 시작하여 Windows Recovery Environment (WinRE)로 진입합니다. 이때 Control 키를 꾹 누르고 있으면, 비밀번호 입력이나 복구 키 요구 없이 즉시 최고 관리자 권한(Elevated command line)의 프롬프트가 떨어집니다. BitLocker로 암호화된 드라이브는 이미 완전히 복호화되어 마운트된 상태입니다.

이게 어떻게 가능할까요? 핵심은 **TPM-only** 모드의 구조적 한계에 있습니다. 기본 설정에서 BitLocker는 사용자가 누구인지 검증하지 않습니다. 단지 부트스트랩(WinRE 포함)이 Microsoft의 신뢰할 수 있는 서명을 가졌는지만 확인하고, TPM은 기꺼이 Volume Master Key (VMK)를 메모리에 풀어버립니다. 공격자는 이 신뢰된 복구 환경의 초기화 프로세스 어딘가를 가로채어 셸을 띄운 것입니다.

## 백도어인가, 끔찍한 버그인가?
원문 기사와 Hacker News의 일부 엔지니어들은 익스플로잇에 사용된 파일들이 실행 후 자폭(Self-delete)한다는 점을 들어 이를 명백한 백도어(Backdoor)라고 주장하고 있습니다.

솔직히 말씀드리면, 저는 이것을 의도된 악의적 백도어라고 생각하지 않습니다. 제 경험상 이것은 Microsoft가 OEM 장비 복구나 내부 디버깅을 위해 만들어둔 WinRE의 자동화 스크립트 실행 로직이 방치된 결과일 확률이 높습니다. USB에서 특정 패턴을 읽어들여 진단 모드로 진입하는 기능(마치 과거의 공장 초기화 트리거처럼)을 만들어두고, 이를 프로덕션 빌드에서 제대로 막지 않은 전형적인 부채(Technical Debt)의 폭발입니다.

물론 의도가 무엇이든, 결과적으로 물리적 접근이 가능한 공격자에게 시스템을 활짝 열어주는 마스터키 역할을 한다는 점에서는 변명의 여지가 없습니다.

## GreenPlasma와 권한 상승 문제
YellowKey와 함께 공개된 **GreenPlasma** 역시 주목해야 합니다. 이는 CTFMon 프로세스를 조작하여 SYSTEM 권한을 얻어내는 Local Privilege Escalation (LPE) 취약점입니다.

메모리 섹션 객체(Memory section object)를 조작하여 정상적인 접근 제어를 우회하는 방식인데, 데스크톱 환경에서도 문제지만 특히 서버 환경에서는 치명적입니다. 일반 유저 권한만 탈취해도 서버 전체의 통제권을 넘겨주게 되기 때문입니다.

## Hacker News 동향: TPM+PIN은 안전한가?
Hacker News 커뮤니티에서도 이 사안을 두고 격렬한 토론이 벌어지고 있습니다. 대다수의 시니어 엔지니어들이 가장 우려하는 질문은 바로 이것입니다. "Pre-boot PIN을 설정한 BitLocker도 뚫리는가?"

익스플로잇 개발자인 Chaotic Eclipse는 TPM+PIN 모드에 대한 우회 방법도 가지고 있다고 주장하지만, 아직 PoC(Proof of Concept)는 공개하지 않았습니다. 저는 이 주장에 대해서는 매우 회의적인 입장입니다. PIN 자체가 TPM에서 키를 Unseal하기 위한 엔트로피로 작용하도록 올바르게 설계되었다면, 물리적으로 키가 메모리에 존재하지 않기 때문에 OS 단의 셸 탈취만으로는 이를 우회할 수 없어야 정상입니다. 만약 PIN 모드마저 뚫린다면, 그것은 Windows의 암호학적 구현 자체가 근본적으로 붕괴되었음을 의미합니다.

## 결론 및 대응 방안
기본 설정(TPM-only)으로 구성된 BitLocker는 이제 물리적 탈취에 대해 완전히 무력화되었다고 보아야 합니다. Microsoft가 Windows 11에서 TPM을 강제하며 약속했던 보안은 환상에 불과했습니다.

프로덕션 환경을 책임지는 엔지니어라면 당장 다음 조치를 취해야 합니다.
- **GPO 설정:** 그룹 정책(Group Policy)을 통해 BitLocker Pre-boot PIN 인증을 즉시 강제하십시오.
- **서버 패치:** GreenPlasma와 같은 LPE 취약점에 대비해 Windows Server 2022/2025 환경의 모니터링을 강화하고, 공식 패치를 예의주시하십시오.

보안은 하드웨어 칩 하나에 의존해서 완성되지 않습니다. 이번 사태는 우리가 잊고 있던 '심층 방어(Defense in Depth)'의 중요성을 다시 한번 뼈저리게 일깨워줍니다.

## References
- **Original Article:** https://www.tomshardware.com/tech-industry/cyber-security/microsoft-bitlocker-protected-drives-can-now-be-opened-with-just-some-files-on-a-usb-stick-yellowkey-zero-day-exploit-demonstrates-an-apparent-backdoor
- **Hacker News Thread:** https://news.ycombinator.com/item?id=48130519
