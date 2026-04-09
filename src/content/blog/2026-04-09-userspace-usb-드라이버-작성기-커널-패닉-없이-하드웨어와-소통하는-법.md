---
title: "Userspace USB 드라이버 작성기: 커널 패닉 없이 하드웨어와 소통하는 법"
description: "Userspace USB 드라이버 작성기: 커널 패닉 없이 하드웨어와 소통하는 법"
pubDate: "2026-04-09T06:33:51Z"
---

솔직히 말해서, "새로운 USB 기기의 드라이버를 작성해 달라"는 요구를 받으면 대부분의 엔지니어는 본능적으로 방어적인 태도를 취하게 됩니다. 커널 코드를 건드려야 하고, 끝없는 BSOD(블루스크린)나 커널 패닉과 싸워야 하며, Windows 환경이라면 그 악명 높은 드라이버 서명(Driver Signing) 인증서 문제까지 겹치기 때문입니다. 

하지만 우리가 굳이 커널 영역(Kernel space)에서 놀아야 할 이유가 있을까요? 최근 WerWolv의 블로그에 올라온 [USB for Software Developers](https://werwolv.net/posts/usb_for_sw_devs/)라는 글을 읽고, 제가 예전에 커스텀 하드웨어 디버깅 툴을 만들며 느꼈던 Userspace 기반 접근법의 우수성을 다시금 떠올리게 되었습니다. 오늘은 이 글과 Hacker News의 논의를 바탕으로, 왜 우리가 Userspace에서 USB를 다루어야 하는지, 그리고 그것이 기술적으로 어떻게 동작하는지 깊이 파헤쳐 보겠습니다.

## 왜 Userspace인가?

제가 15년 넘게 시스템 프로그래밍을 해오며 얻은 교훈 중 하나는, **"가능하다면 커널 밖으로 나와라"** 입니다. 커널 드라이버는 시스템 전체의 안정성을 담보로 잡습니다. 포인터 하나 잘못 건드리면 시스템 전체가 다운되죠. 반면 Userspace에서 애플리케이션 레벨로 하드웨어와 통신하면, 버그가 발생해도 해당 프로세스만 죽고 끝납니다. 디버거를 붙이기도 훨씬 수월하죠.

Windows에서는 WinUSB를 통해 서명 없이도 범용 USB 기기와 통신할 수 있고, Linux나 macOS에서도 `libusb` 같은 훌륭한 라이브러리를 통해 소켓 프로그래밍하듯 USB 기기를 다룰 수 있습니다.

## USB 통신의 기초: Enumeration과 Endpoints

네트워크 프로그래밍을 할 때 IP 주소와 포트 번호가 필요하듯, USB 통신에도 고유의 주소 체계가 있습니다.

기기를 PC에 꽂으면 OS는 가장 먼저 **Enumeration** (열거) 과정을 거칩니다. 기기가 자신의 Vendor ID(VID)와 Product ID(PID)를 호스트에 보고하는 과정이죠. Linux의 `lsusb` 명령어로 흔히 보는 `18d1:4ee0` 같은 값들이 바로 이것입니다.

기기를 식별했다면 이제 데이터를 주고받을 차례입니다. USB는 **Endpoint** 라는 개념을 사용합니다. 네트워크의 포트와 비슷하지만, 철저하게 단방향(Unidirectional)이며 호스트(PC) 중심적이라는 특징이 있습니다.

- **IN Endpoint:** 호스트가 기기로부터 데이터를 '가져올 때' 사용합니다.
- **OUT Endpoint:** 호스트가 기기로 데이터를 '보낼 때' 사용합니다.

기기는 스스로 호스트에게 먼저 데이터를 쏠 수 없습니다. 철저한 Master-Slave 구조이며, 호스트가 폴링(Polling)하거나 요청해야만 기기가 응답할 수 있습니다.

### 4가지 Transfer Types

USB 스펙은 데이터의 특성에 따라 4가지 전송 방식을 정의합니다.

1. **Control:** 0x00 주소에 고정된 기본 엔드포인트입니다. 기기의 설정값을 읽거나 Descriptor를 요청할 때 사용됩니다. 닭과 달걀의 문제를 해결해 주는 초기화 채널이죠.
2. **Bulk:** 대용량 데이터를 보낼 때 사용합니다. 대역폭은 넓지만 우선순위가 낮아 남는 대역폭을 씁니다. Mass Storage나 Fastboot 프로토콜이 이를 사용합니다.
3. **Interrupt:** 마우스나 키보드 같은 HID 기기에서 짧은 지연 시간(Low latency)을 요구할 때 씁니다. 이름은 인터럽트지만 실제로는 호스트가 매우 빠른 주기로 폴링하는 방식입니다.
4. **Isochronous:** 오디오나 비디오 스트리밍처럼 데이터 유실보다 타이밍(지연)이 더 중요한 경우에 사용합니다.

## C++와 libusb로 Fastboot 기기 제어하기

원문에서는 Android 기기의 Fastboot 모드를 예제로 삼았습니다. Fastboot은 매우 단순한 텍스트 기반 프로토콜을 Bulk 전송으로 주고받습니다. 

아래는 `libusb`를 사용해 기기에 `getvar:version` 명령을 보내는 핵심 코드입니다.

```cpp
// 64바이트 버퍼 준비
std::vector<uint8_t> bytes(64);
std::ranges::copy("getvar:version", bytes.begin());

// OUT Endpoint(0x02)로 Bulk 전송
int num_bytes_transferred = 0;
libusb_bulk_transfer(
    handle,
    LIBUSB_ENDPOINT_OUT | 0x02, // 0x02번 OUT 엔드포인트
    bytes.data(), bytes.size(), 
    &num_bytes_transferred,
    1000 // 1000ms 타임아웃
);

// IN Endpoint(0x01)로부터 응답 수신
std::ranges::fill(bytes, 0x00);
libusb_bulk_transfer(
    handle,
    LIBUSB_ENDPOINT_IN | 0x01,  // 0x01번 IN 엔드포인트
    bytes.data(), bytes.size(),
    &num_bytes_transferred,
    1000
);
```

이 코드를 보면 커널 프로그래밍의 흔적은 전혀 찾아볼 수 없습니다. 그저 라이브러리 함수를 호출해 버퍼를 밀어 넣고 빼내는 것뿐이죠. 소켓 프로그래밍과 다를 바가 없습니다.

*여담이지만, 원문 코드에 등장한 `auto main() -> int` 문법 때문에 Hacker News 댓글 창에서 한바탕 소동이 있었습니다. 프로그래밍 폰트의 리거처(Ligature) 기능 때문에 `->`가 `→` 화살표 기호로 렌더링되었기 때문이죠. C++의 Trailing return type을 처음 본 일부 개발자들이 혼란을 겪은 재밌는 해프닝이었습니다.*

## Hacker News 커뮤니티의 인사이트

이 글에 대한 [Hacker News의 반응](https://news.ycombinator.com/item?id=47695012)은 꽤나 실용적이고 깊이가 있었습니다.

특히 인상 깊었던 논의는 **OS 서브시스템과의 통합 문제** 였습니다. 만약 Userspace에서 USB-to-Ethernet 드라이버를 만들었다면, 이를 어떻게 OS의 네트워크 스택에 연결할까요? 

가장 현실적인 답변은 Linux의 TUN/TAP 인터페이스를 활용하는 것입니다. Userspace 애플리케이션이 가상 네트워크 인터페이스를 만들고, OS 네트워크 스택에서 내려오는 패킷을 가로채어 USB Bulk 전송으로 변환하는 브릿지 역할을 하는 것이죠. 고빈도 거래(HFT) 시스템에서 커널 오버헤드를 줄이기 위해 커널 바이패스(Kernel Bypass) 네트워크 스택을 사용하는 것과 철학적으로 맞닿아 있습니다.

또한, 최근 트렌드에 맞게 C++ 대신 Rust의 `nusb`나 Go의 `go-usb`를 사용해 메모리 안전성까지 챙기는 방식도 널리 추천되었습니다. 

다만 한 가지 주의할 점은, **최신 macOS(M3 등) 환경에서의 보안 제약** 입니다. 시스템이 이미 자체 드라이버로 인식한 USB 기기를 Userspace에서 강제로 가로채는(Override) 행위가 보안 정책상 점점 까다로워지고 있다는 현실적인 지적도 있었습니다.

## 나의 결론 (Verdict)

Userspace USB 프로그래밍은 단순한 장난감이 아닙니다. 

기존에 존재하는 클래스(Mass Storage, HID 등)를 구현하는 상용 제품이라면 당연히 OS 표준 드라이버에 맞춰야겠지만, **사내 디버깅 툴, DFU(Device Firmware Upgrade) 플래셔, 혹은 프로토콜이 공개되지 않은 구형 하드웨어(예: 구형 MIDI 장비)를 리버스 엔지니어링하여 살려내는 작업** 이라면 이 방식이 압도적으로 우수합니다.

커널 패닉의 두려움 없이, 익숙한 Userspace 환경에서 현대적인 언어(C++23, Rust, Go)의 풍부한 생태계를 누리며 하드웨어와 소통해 보시길 권장합니다. 하드웨어 개발자와 소프트웨어 개발자 사이의 보이지 않는 벽을 허무는 아주 훌륭한 도구가 될 것입니다.

---

**References:**
- Original Article: [USB for Software Developers | WerWolv](https://werwolv.net/posts/usb_for_sw_devs/)
- Hacker News Thread: [Discussion on HN](https://news.ycombinator.com/item?id=47695012)
