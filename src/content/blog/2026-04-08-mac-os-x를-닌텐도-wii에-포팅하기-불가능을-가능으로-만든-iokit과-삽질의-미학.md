---
title: "Mac OS X를 닌텐도 Wii에 포팅하기: 불가능을 가능으로 만든 IOKit과 삽질의 미학"
description: "Mac OS X를 닌텐도 Wii에 포팅하기: 불가능을 가능으로 만든 IOKit과 삽질의 미학"
pubDate: "2026-04-08T20:47:44Z"
---

"There is a zero percent chance of this ever happening."

2021년 레딧에 달린 이 단호한 댓글은 이번 프로젝트의 완벽한 시작점입니다. 우리 엔지니어들은 누군가 "절대 불가능하다"고 선언할 때 묘한 전투력을 얻곤 하죠. 저 역시 예전 직장에서 Ops 매니저가 "스택 트레이스만으로 어느 DB 인스턴스가 죽었는지 아는 건 불가능하다"고 했을 때, 오기로 Ruby 라이브러리를 깎아내던 기억이 납니다.

최근 닌텐도 Wii에 Mac OS X 10.0 (Cheetah)를 네이티브로 구동한 Bryan Keller의 글을 읽으며, 순수하게 '그냥 해보고 싶어서' 밤을 새우던 그 시절의 해커 정신을 다시 느꼈습니다. 요즘처럼 AI가 생성한 영혼 없는 아티클이 넘쳐나는 시대에, 이런 딥다이브 하드코어 엔지니어링 포스트는 그야말로 가뭄의 단비 같습니다.



## 아키텍처와 커스텀 부트로더

Wii는 PowerPC 750CL 프로세서를 사용합니다. 이는 과거 G3 iBook이나 iMac에 탑재되었던 CPU와 아키텍처가 거의 동일하죠. CPU 자체는 호환성이 보장된다는 뜻입니다. 하지만 부팅 과정은 완전히 다릅니다. 일반적인 올드 Mac은 Open Firmware와 BootX를 거쳐 커널을 로드하지만, 저자는 과감하게 이들을 버리고 커스텀 부트로더를 밑바닥부터 작성했습니다.

솔직히 매우 현명한 선택입니다. 불필요한 레거시 하드웨어 호환성을 챙길 필요 없이, 딱 Wii 하드웨어만 초기화하고 Mach-O 커널을 메모리에 올린 뒤 Device Tree만 넘겨주면 되니까요.

하지만 커널 엔트리 포인트로 점프한 직후 화면이 까맣게 변하고 시리얼 디버그 로그마저 멈췄을 때의 그 막막함은 로우레벨 개발자라면 다들 공감하실 겁니다. 여기서 저자가 선택한 디버깅 방식은 꽤나 예술적입니다.

```assembly
lis r5, 0xd80
ori r5, r5, 0xc0
lwz r4, (r5)
sync
xori r4, r4, 0x20
stw r4, (r5)
```

커널 어셈블리를 직접 바이너리 패치해서, 특정 초기화 단계를 통과할 때마다 Wii 전면 패널의 LED를 깜빡이게 만든 것입니다. JTAG 같은 고급 장비 없이 맨땅에 헤딩할 때 쓸 수 있는 가장 원초적이면서도 확실한 방법이죠.

## IOKit의 우아함과 고통

부팅이 진행되면서 마주한 가장 큰 벽은 드라이버 작성입니다. Mac OS X의 드라이버 모델인 IOKit은 철저하게 객체지향적인 C++ 기반으로 설계되었습니다. HN 커뮤니티에서도 NeXT 시절의 DriverKit과 비교하며 많은 이야기가 오갔는데, IOKit의 Provider-Client 추상화 계층은 20년이 지난 지금 봐도 상당히 우아합니다.

Wii는 표준 PCI 버스 대신 Hollywood SoC를 사용합니다. 따라서 저자는 기존의 IOPCIFamily에 묻어가는 대신, 자체적인 Hollywood 드라이버를 작성하여 하위 디바이스들을 위한 Nub(attach-point)을 직접 생성해야 했습니다.

```cpp
IOService *NintendoWiiHollywood::createNub(IORegistryEntry *from) {
    NintendoWiiHollywoodDevice *nub = new NintendoWiiHollywoodDevice;
    if (nub && nub->init(from, gIODTPlane)) {
        nub->hollywood = this;
        return nub;
    }
    if (nub) {
        nub->release();
    }
    return 0;
}
```

이후 Starlet 코프로세서와의 IPC 통신을 통해 SD 카드 드라이버를 구현하고 마침내 "Still waiting for root device" 에러를 넘어섰을 때의 쾌감은 글을 읽는 저에게도 고스란히 전해졌습니다.

## 프레임버퍼와 Endianness 악몽

GUI를 띄우기 위한 프레임버퍼 작업에서도 흥미로운 꼼수가 등장합니다. Wii의 비디오 하드웨어는 아날로그 TV 출력을 위해 16-bit YUV 포맷을 기대하지만, Mac OS X는 RGB를 뱉어냅니다. 이 때문에 화면이 온통 마젠타(Magenta) 색으로 덮이는 문제가 발생했죠.

해결책은 무식하지만 확실했습니다. RGB 프레임버퍼와 YUV 프레임버퍼 두 개를 메모리에 잡고, 초당 60번씩 소프트웨어로 RGB 데이터를 YUV로 변환해버린 것입니다. 퍼포먼스 오버헤드가 있겠지만, 일단 돌아가게 만드는 것이 최우선이니까요.



가장 끔찍했던 부분은 USB 지원이었습니다. 마우스와 키보드를 잡기 위해 AppleUSBOHCI 드라이버를 로드해야 했는데, 여기서 Endianness 문제가 터집니다. Wii 하드웨어는 Reversed-Little-Endian 방식을 사용해 하드웨어 단에서 바이트 스왑을 처리하는데, Apple의 USB 스택은 소프트웨어 단에서 또 스왑을 해버려 데이터가 완전히 망가지는 상황이었습니다.

소스 코드가 없어서 비행기 안에서 Ghidra로 어셈블리를 뜯어고치다 실패하고, 결국 고대 유물 같은 IRC 채널을 뒤져 Mac OS X Cheetah 시절의 IOUSBFamily 소스코드를 찾아내 빌드에 성공하는 대목은 이 글의 클라이맥스입니다.

## 커뮤니티의 반응과 나의 생각

Hacker News 스레드를 보면 다들 비슷한 향수를 느끼고 있습니다. "이게 바로 내가 처음 HN에 접속했던 이유다", "요즘 넘쳐나는 AI 생성 슬롭(Slop)들 사이에서 이런 진짜 해커의 글을 보니 눈이 정화된다"는 반응이 지배적입니다.

- **핵심 포인트:** 불가능을 선언하는 것은 작업 자체의 난이도라기보다, 선언하는 개인의 인지적 한계에 불과합니다.

시니어 엔지니어로서 이 프로젝트를 평가하자면, 프로덕션 레벨의 실용성은 제로입니다. 하지만 이 과정에서 얻은 OS 부트스트래핑, 커널 패칭, IOKit 드라이버 작성, 하드웨어 레지스터 디버깅 경험은 그 어떤 튜토리얼에서도 배울 수 없는 진짜 엔지니어링 근육을 키워줍니다.

가끔은 우리도 당장 돈이 되거나 KPI를 달성하는 업무에서 벗어나, 순수하게 "이게 될까?"라는 호기심만으로 밤을 새워보는 건 어떨까요? 적어도 저는 오늘 주말에 묵혀뒀던 라즈베리 파이 토이 프로젝트를 다시 꺼내볼 생각입니다.

**References:**
- Original Article: https://bryankeller.github.io/2026/04/08/porting-mac-os-x-nintendo-wii.html
- Hacker News Thread: https://news.ycombinator.com/item?id=47691730
