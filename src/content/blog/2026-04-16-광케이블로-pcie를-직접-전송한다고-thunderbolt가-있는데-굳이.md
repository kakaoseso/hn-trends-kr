---
title: "광케이블로 PCIe를 직접 전송한다고? (Thunderbolt가 있는데 굳이?)"
description: "광케이블로 PCIe를 직접 전송한다고? (Thunderbolt가 있는데 굳이?)"
pubDate: "2026-04-16T19:41:26Z"
---

최근 해커뉴스(Hacker News)에서 아주 흥미로운, 그리고 다분히 '긱(Geek)'스러운 프로젝트를 하나 발견했다. 바로 PCIe 신호를 광케이블(Fiber)에 직접 태워서 전송하는 하드웨어 해킹 영상이다.

서버실이나 랩실에서 비행기 이륙 소리를 내는 GPU 랙을 보며 '이놈의 GPU들만 저 멀리 창고에 박아두고 싶다'는 생각, 인프라나 하드웨어를 다루는 엔지니어라면 한 번쯤 해봤을 거다. 실제로 HN 댓글 창에서도 "미시간주 눈밭에 4090 여러 대를 던져두고, 지하실로 광케이블만 끌어와서 무소음 천연 쿨링을 하고 싶다"는 농담이 많은 공감을 얻었다.

하지만 15년 차 엔지니어의 시각에서 볼 때, 기술적 호기심을 넘어 "잠깐, 우리에겐 이미 Thunderbolt나 USB4가 있지 않나? 왜 굳이 raw PCIe를 SFP 모듈에 밀어 넣는 삽질을 하는 걸까?"라는 의문이 드는 게 사실이다. 이 프로젝트가 기술적으로 왜 어렵고, 어떤 의미를 가지는지 밑바닥부터 파헤쳐보자.

## SFP의 본질: 멍청하지만(Dumb) 미친 듯이 빠른 파이프

이 프로젝트를 이해하려면 먼저 SFP/QSFP 모듈의 본질을 알아야 한다. 많은 사람들이 SFP를 네트워크 전용 장비로 오해하지만, 사실 얘네는 프로토콜을 따지지 않는다.

(Q)SFP는 기본적으로 초고속 직렬(Serial) 인터페이스다. FPGA를 다뤄본 사람들은 알겠지만, QSFP 인터페이스에 이더넷뿐만 아니라 SATA, HDMI, 심지어 커스텀 프로토콜까지 다 쏠 수 있다. AliExpress에 굴러다니는 흙수저용 USB3/HDMI over SFP 컨버터들이 그토록 저렴한 이유도 결국 SFP가 물리 계층(PHY) 역할만 하고, 나머지 복잡한 처리는 저렴한 칩셋이 다 해주기 때문이다.

따라서 이론적으로는 PCIe의 TX/RX 핀을 SFP의 TX/RX에 물리적으로 매핑하기만 하면 데이터 자체는 넘어간다. 하지만 진짜 문제는 그다음부터 시작된다.

## 왜 Raw PCIe over Fiber는 지옥인가?

PCIe는 단순한 데이터 파이프가 아니다. Host와 Device 간의 상태를 맞추기 위한 극도로 복잡한 메커니즘이 존재한다. 영상을 보면 이 프로젝트가 직면한 끔찍한 호환성 문제들이 나오는데, 크게 두 가지로 요약할 수 있다.

- **Device Detection 및 Clocking:** PCIe는 데이터 라인 외에도 호스트와 디바이스가 상태를 주고받는 Side-band signal(Reset, Wake 등)이 필수적이다. 또한, SRIS(Separate Reference Independent SSC)를 지원하지 않는 환경이라면 호스트의 레퍼런스 클럭을 디바이스와 정확히 동기화해야 한다. 광케이블은 오직 데이터(TX/RX)만 넘겨주므로 이 side-channel들이 완전히 끊겨버린다.
- **Equalization Training (Gen 3 이상):** 개인적으로 가장 치명적이라고 보는 부분이다. PCIe Gen 3부터는 신호 무결성을 위해 양단이 전기적 특성(Electrical characteristics)을 측정하고 파라미터를 튜닝하는 EQ Training 과정을 거친다. 그런데 중간에 광 모듈(O-E/E-O 변환)이 껴버리면? 호스트와 디바이스 사이의 전기적 피드백 루프가 완전히 박살 난다. SFP 링크 위에서 EQ Training은 바보가 될 수밖에 없다.

## Thunderbolt라는 훌륭한 대안

HN의 한 유저가 날카로운 지적을 했다.
> "Thunderbolt 같은 Encapsulating Protocol을 쓰면 이런 호환성 문제 다 해결되는데, 굳이 왜 생고생을 함? Thunderbolt 오버헤드가 그렇게 큼?"

솔직히 내 생각도 정확히 일치한다. Thunderbolt 3/4는 PCIe 데이터를 캡슐화(Encapsulation)해서 전송하며, 앞서 말한 클럭 동기화, 리셋, EQ 트레이닝 같은 골치 아픈 물리 계층의 문제들을 컨트롤러 레벨에서 싹 다 추상화해 준다. 

물론 캡슐화 과정에서 약간의 Latency와 Throughput 손실(Overhead)이 발생하긴 하지만, 일반적인 GPU 연산이나 데이터 전송에서는 체감하기 힘든 수준이다. 게다가 이미 시중에는 TB3/TB4용 광학 모듈과 케이블이 상용화되어 있다. 굳이 호환성 이슈를 감수하며 Raw PCIe를 고집할 실용적인 이유는 거의 없다고 본다.

## 미래는 이미 정해져 있다: Optical Retimer와 CXL

그럼에도 불구하고 이 프로젝트가 시사하는 바는 크다. 미래의 데이터센터 아키텍처는 결국 'Disaggregation(자원 분리)'으로 가고 있기 때문이다.

재미있게도 PCIe SIG(표준화 기구)는 이미 이런 수요를 파악하고 PCIe 스펙 내에 **Optical Retimer** 에 대한 조항을 추가했다. 즉, 이런 해킹 꼼수가 아니라 네이티브하게 광통신을 지원하는 PCIe 규격이 표준으로 자리 잡고 있다는 뜻이다. 여기에 CXL(Compute Express Link) 프로토콜이 결합되면 어떨까? CPU는 랙 A에, GPU 팜은 랙 B에, 메모리 풀은 랙 C에 두고 광케이블로 묶어버리는 진정한 Composable Infrastructure가 완성된다. 2018년쯤 우리가 NVMe-oF로 스토리지 병목을 풀었던 것과 정확히 같은 패러다임 시프트가 이제 PCIe 버스 자체에서 일어나고 있는 것이다.

## 결론 (Verdict)

그래서, 당장 집에 있는 4090을 눈 덮인 앞마당으로 쫓아낼 수 있을까?

만약 당신이 안정적인 프로덕션 환경을 원하거나, 정신 건강을 소중히 여긴다면 그냥 얌전히 컴퓨터 본체째로 밖에 내놓거나 Thunderbolt 광 도킹 스테이션을 사라. 현재로서는 Raw PCIe over SFP는 너무 Brittle(깨지기 쉬운)한 장난감이다.

하지만 SFP 모듈의 본질을 꿰뚫어 보고 고속 시리얼 통신의 밑바닥 물리 계층을 직접 파헤쳐본 이 해커의 집요함에는 기립 박수를 보낸다. 엔지니어링의 위대한 발전은 항상 이런 '쓸데없어 보이는 잉여로움'에서 시작되는 법이니까.

---
**References**
- **Original Article:** https://www.youtube.com/watch?v=XaDa9bBucEI
- **Hacker News Thread:** https://news.ycombinator.com/item?id=47733059
