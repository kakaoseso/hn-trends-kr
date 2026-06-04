---
title: "엔터프라이즈 장비 없이 95Gbps RDMA 구현하기: Thunderbolt-ibverbs 리뷰"
description: "엔터프라이즈 장비 없이 95Gbps RDMA 구현하기: Thunderbolt-ibverbs 리뷰"
pubDate: "2026-06-04T17:28:31Z"
---

LLM 시대에 접어들면서 홈 랩이나 소규모 스타트업 환경에서 멀티 노드 GPU 클러스터를 구축하려는 시도가 많아졌다. 하지만 늘 그렇듯 진짜 병목은 컴퓨팅 파워가 아니라 네트워킹에서 발생한다. 노드 간 Tensor Parallelism이나 FSDP(Fully Sharded Data Parallel)를 돌리려면 InfiniBand나 최소 100G 이상의 고속 이더넷과 RDMA 환경이 필수적인데, 엔터프라이즈급 스위치와 NIC(Network Interface Card)의 가격은 그야말로 절망적이다.

최근 Hacker News에서 내 눈길을 사로잡은 프로젝트가 하나 있다. AMD Strix Halo 미니 PC의 평범한 USB4/Thunderbolt 포트를 InfiniBand 디바이스처럼 속여서 사용하는 thunderbolt-ibverbs 프로젝트다.

![Two Strix Halo mini-PCs (strix-1, strix-2) connected by USB4](https://blog.hellas.ai/blog/thunderbolt-ibverbs/./strix-strix.jpeg)

## 어떻게 구현했는가? (그리고 왜 빠른가)

일반적으로 Thunderbolt 포트끼리 연결해서 네트워크를 구성한다고 하면 IP-over-Thunderbolt (또는 thunderbolt-net)를 떠올린다. 하지만 이 방식은 커널의 표준 네트워크 스택을 타기 때문에 오버헤드가 극심하다. 실제로 저자도 이 방식으로는 Throughput이 ~9 Gb/s에 그쳤고 Latency는 ~65 µs나 나왔다고 언급했다. 이 정도면 분산 학습은 커녕 단순 파일 전송용으로나 쓸 법한 수치다.

저자가 선택한 방식은 TCP/IP 스택을 완전히 우회하는 것이다. Thunderbolt 컨트롤러의 NHI(Node Hardware Interface) 포트에서 직접 DMA 링을 할당받아, 이를 InfiniBand Verbs API로 에뮬레이션하는 커널 모듈을 밑바닥부터(물론 AI의 도움을 받아) 작성했다.

결과는 꽤나 충격적이다.

- **Throughput:** 양방향 기준 ~95 Gb/s (단방향 ~48 Gb/s)
- **Latency:** 64B 메시지 기준 ~7 µs (One-way)

이 정도 스펙이면 어지간한 구형 Mellanox ConnectX NIC를 직결한 것과 유사한 성능이다. 실제로 저자는 이 셋업으로 두 대의 미니 PC를 묶어 단일 노드 메모리에 올라가지 않는 MiniMax-M2.7 모델의 TP=2 추론을 성공시켰고, Gemma 3 27B LoRA FSDP 스텝 처리 시간을 기존 이더넷 환경의 1359초에서 126초로 극적으로 단축시켰다.

## 시니어 엔지니어의 시선: 무식하지만 우아한 해킹

15년 넘게 인프라와 백엔드를 다뤄온 입장에서, 이런 류의 저비용 고효율 네트워크 해킹은 항상 흥미롭다. 과거 PCIe Non-Transparent Bridge(NTB)를 이용해 노드 간 메모리를 직접 매핑하려던 시도들이 떠오르기도 한다. 

솔직히 말해, 처음 제목만 봤을 때는 '또 그저 그런 소프트웨어 RoCE(Soft-RoCE) 래퍼겠지'라고 생각했다. 하지만 커널 레벨에서 NHI DMA를 직접 건드려 Verbs API를 노출시켰다는 대목에서 감탄했다. AI 런타임(vLLM, RCCL 등) 입장에서는 이 장비가 진짜 InfiniBand인지 USB 케이블인지 알 길이 없다. 그저 표준 RDMA 인터페이스로 통신할 뿐이다. Zero-copy 네트워킹의 본질을 정확히 꿰뚫은 접근이다.

물론 저자 스스로도 밝혔듯 이는 언제든 커널 패닉을 일으킬 수 있는 실험적인 코드다. AI가 생성한 코드가 다수 포함되어 있고, IOMMU를 끄고 테스트하는 등 프로덕션 환경에서는 상상도 할 수 없는 타협들이 들어있다. 하지만 PoC(Proof of Concept)로서 이 프로젝트가 가지는 가치는 엄청나다.

## Hacker News 커뮤니티의 반응과 Linux 7.2

Hacker News 스레드에서도 꽤 심도 있는 논의가 오갔다. 가장 눈에 띄는 부분은 다가오는 Linux v7.2에 추가될 예정인 USB4STREAM에 대한 언급이었다. 

한 유저가 곧 커널에 통합될 USB4STREAM을 사용하면 로우 레벨 Thunderbolt 패킷을 쉽게 다룰 수 있을 것이라고 지적했다. 이에 대해 저자는 본인의 구현체가 기존의 thunderbolt-net 위에 올라간 것이 아니라, 해당 네트워킹 스택과 동일한 원시(primitive) 레벨에서 Verbs 디바이스를 에뮬레이션하도록 재조립된 것이라고 명확히 선을 그었다.

커뮤니티의 반응 중 내가 가장 공감했던 것은 저자의 태도에 대한 칭찬이다. '이 코드는 AI가 짰고, 당신의 머신을 크래시 낼 수 있다'고 투명하게 밝힌 점이 오히려 엔지니어들의 신뢰를 얻었다. 기술의 한계와 현재 상태를 명확히 인지하고 공유하는 것은 시니어리티의 중요한 덕목 중 하나다.

## 결론: 그래서 쓸 만한가?

프로덕션 환경의 데이터센터에 이 방식을 도입하겠다고 한다면 당장 말리고 싶다. 엔터프라이즈 환경에서 InfiniBand를 비싼 돈 주고 사는 이유는 단순히 속도 때문만이 아니다. 안정성, 스위치 토폴로지 확장성, 그리고 벤더의 든든한 기술 지원이 포함된 가격이다.

하지만 홈 랩을 운영하는 AI 연구자나 예산이 극도로 제한된 초기 스타트업이라면? 이야기가 다르다. 비싼 PCIe 스위치나 중고 Mellanox 카드를 구하러 이베이를 뒤지는 대신, 책상 위 미니 PC 두 대를 썬더볼트 케이블 하나로 묶어 당장 FSDP를 태워볼 수 있다는 것은 엄청난 혁신이다.

앞으로 Linux 7.2에 USB4STREAM이 정식으로 도입되고 이러한 유저 스페이스/커널 레벨의 RDMA 에뮬레이션이 안정화된다면, Consumer 하드웨어를 활용한 엣지 컴퓨팅이나 소규모 분산 AI 클러스터 구축의 패러다임이 크게 바뀔 수 있을 것이라 생각한다.

---

- **Original Article:** https://blog.hellas.ai/blog/thunderbolt-ibverbs/
- **Hacker News Thread:** https://news.ycombinator.com/item?id=48371008
