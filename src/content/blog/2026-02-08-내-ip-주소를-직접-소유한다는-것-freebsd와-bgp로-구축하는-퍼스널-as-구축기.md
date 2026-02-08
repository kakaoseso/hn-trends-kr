---
title: "내 IP 주소를 직접 소유한다는 것: FreeBSD와 BGP로 구축하는 퍼스널 AS 구축기"
description: "내 IP 주소를 직접 소유한다는 것: FreeBSD와 BGP로 구축하는 퍼스널 AS 구축기"
pubDate: "2026-02-08T15:46:31Z"
---

네트워크 엔지니어들의 영원한 '로망'이 하나 있습니다. 바로 내 집 마련은 못 해도 내 IP 대역(Prefix)은 갖고 싶다는 것이죠. ISP가 할당해 주는 IP가 아니라, 내가 전 세계 어디로 이사를 가든 나를 따라다니는 고유한 IP 주소 말입니다.

최근 Hacker News에서 꽤 흥미로운 아티클이 올라왔습니다. 단순히 BGP를 돌리는 것을 넘어서, FreeBSD의 강력한 네트워킹 스택(Dual-FIB, PF)을 활용해 **Provider-Independent** 한 환경을 구축한 사례입니다. 오늘은 이 아티클을 바탕으로, 왜 우리가 굳이 이 '삽질'을 해야 하며, 기술적으로 어떤 우아함이 숨어 있는지 딥다이브 해보겠습니다.

### 왜 굳이 내 AS(Autonomous System)를 구축하는가?

클라우드 시대에 이게 무슨 의미가 있냐고 물을 수 있습니다. 하지만 시니어 엔지니어라면 **Vendor Lock-in** 의 공포를 알 겁니다. 호스팅 업체를 옮길 때마다 IP가 바뀌고, DNS 레코드를 수정하고, 방화벽 룰을 갱신하고, 평판(Reputation)이 초기화되는 고통 말이죠.

자신의 AS 번호와 IP Prefix(IPv6 /48 등)를 가지면 이 문제가 사라집니다. 물리적 서버를 어디로 옮기든, 터널링만 다시 맺으면 내 IP는 그대로 유지됩니다. 아키텍처 관점에서 보면 인프라와 네트워크 아이덴티티를 완벽하게 분리하는 셈입니다.

### 아키텍처: Hub-and-Spoke의 정석

이 글의 필자는 전형적인 Hub-and-Spoke 구조를 택했습니다.

- **Hub (BGP Router):** Upstream Provider와 BGP 세션을 맺는 FreeBSD VM입니다. 여기가 인터넷 세상으로 나가는 관문입니다.
- **Spoke (Downstream):** 실제 서비스가 돌아가는 VPS나 홈 서버입니다. Hub와는 터널(GRE/GIF)로 연결됩니다.

여기서 핵심은 **FreeBSD** 를 선택했다는 점입니다. 리눅스로도 가능하지만, 라우팅 테이블 분리나 터널링 처리에 있어서 FreeBSD가 보여주는 일관성은 확실히 매력적입니다.

### 1. FRR을 이용한 BGP 설정과 Bogon 필터링

라우팅 데몬으로는 업계 표준이나 다름없는 **FRR(Free Range Routing)** 을 사용했습니다. 설정 파일은 Cisco IOS나 Juniper Junos를 다뤄본 분들에게는 매우 친숙할 겁니다.

여기서 주목할 점은 **Bogon 필터링** 입니다. 인터넷은 야생입니다. 내 라우터가 엉뚱한 라우팅 정보를 받아들이면 트래픽 블랙홀이 되거나, 최악의 경우 라우팅 하이재킹의 공범이 될 수 있습니다.

```text
ipv6 prefix-list PL-BOGONS seq 5 deny ::/0 le 7
ipv6 prefix-list PL-BOGONS seq 100 deny 2a06:9801:1c::/48
ipv6 prefix-list PL-BOGONS seq 110 permit ::/0 le 48
```

필자는 자신의 Prefix까지 포함하여 엄격한 Inbound 필터를 적용했습니다. 이는 BGP 운영의 기본이자 핵심입니다. 또한 Upstream에 따라 Community 값을 조정하거나 AS-Path Prepending을 통해 트래픽 엔지니어링을 시도하는 모습은 교과서적입니다.

### 2. Dual-FIB: 이 글의 기술적 하이라이트

제가 이 포스트에서 가장 감탄한 부분은 Downstream 서버의 라우팅 처리 방식입니다. 상황은 이렇습니다:

1. VPS는 이미 호스팅 업체로부터 받은 공인 IP가 있습니다 (Default Gateway 존재).
2. 동시에 터널을 통해 받아온 내 BGP IP 대역도 사용해야 합니다.
3. **문제:** BGP IP를 소스(Source)로 하는 패킷이 호스팅 업체의 게이트웨이로 나가면? **Spoofing** 으로 간주되어 드랍됩니다.

리눅스라면 `ip rule`과 `fwmark`를 써서 복잡하게 해결했겠지만, FreeBSD는 **FIB(Forwarding Information Base)** 라는 다중 라우팅 테이블 기능을 OS 레벨에서 아주 깔끔하게 지원합니다.

```text
# FIB 1 (BGP 트래픽용) 설정
route_fib1default="-6 default -interface gif0 -fib 1"

# 인터페이스 설정
ifconfig_gif0="fib 1 tunnel 203.0.113.10 198.51.100.10 tunnelfib 0"
```

이 설정이 예술입니다. `fib 1`은 터널 내부의 트래픽이 사용할 라우팅 테이블을 지정하고, `tunnelfib 0`는 터널 자체(Encapsulation된 패킷)가 사용할 라우팅 테이블을 지정합니다. 즉, **"내용물은 내 전용 경로(FIB 1)로 다니고, 포장지는 호스팅 업체의 경로(FIB 0)를 탄다"** 는 로직이 단 한 줄로 정의됩니다.

### 3. PF(Packet Filter)의 `reply-to` 매직

비대칭 라우팅(Asymmetric Routing)은 네트워크 엔지니어의 주적입니다. 들어올 때는 터널로 들어왔는데, 나갈 때는 Default Gateway로 나가버리면 연결이 성립되지 않습니다.

FreeBSD의 PF는 `reply-to` 구문으로 이를 우아하게 해결합니다.

```text
pass in quick on $tun_if reply-to ($tun_if $bgp_hub_ip) inet6 ...
```

"이 인터페이스로 들어온 패킷의 응답은 반드시 들어온 곳으로 다시 내보내라"는 강제성을 부여합니다. 상태 기반 방화벽(Stateful Firewall)의 이점을 십분 활용한 것이죠.

### Hacker News의 반응과 개인적 생각

Hacker News 댓글 중 "제목의 As는 AS(Autonomous System)로 대문자 표기해야 한다"는 지적이 있었습니다. 전형적인 개발자 유머 같지만, 사실 이쪽 바닥에서는 중요한 디테일입니다.

개인적으로 이 아티클은 **"왜 리눅스 대신 FreeBSD를 네트워크 장비로 쓰는가?"** 에 대한 훌륭한 대답이라고 생각합니다. 리눅스의 네트워크 스택도 훌륭하지만, `netfilter`, `iptables`, `nftables`, `bpfilter`로 이어지는 파편화에 비해, FreeBSD의 네트워크 스택은 거대한 하나의 잘 설계된 건축물 같은 일관성을 보여줍니다.

### Verdict: 엔지니어의 주말을 바칠 가치가 있는가?

- **난이도:** 상 (BGP, 터널링, 정책 기반 라우팅에 대한 이해 필수)
- **효용성:** 중 (개인 블로그용으로는 오버엔지니어링이지만, 학습 효과는 최상)
- **재미:** 최상

솔직히 말해서, 블로그 하나 돌리자고 LIR을 찾고, RIPE에 서류를 내고, BGP 세션을 맺는 건 "미친 짓"에 가깝습니다. 하지만 우리는 그 미친 짓을 즐기는 사람들이죠. 인프라의 밑바닥부터 끝까지 완전히 제어해보고 싶은 욕구가 있다면, 이번 주말은 FreeBSD ISO를 다운로드하는 것으로 시작해보시길 권합니다.

**References:**
- **Original Article:** https://blog.hofstede.it/running-your-own-as-bgp-on-freebsd-with-frr-gre-tunnels-and-policy-routing/
- **Hacker News Thread:** https://news.ycombinator.com/item?id=46934266
