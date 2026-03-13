---
title: "ARM NEON을 활용한 GB/s급 Prefix Sum 처리와 SVE2 파편화 문제"
description: "ARM NEON을 활용한 GB/s급 Prefix Sum 처리와 SVE2 파편화 문제"
pubDate: "2026-03-13T11:08:48Z"
---

최근 고성능 컴퓨팅 최적화의 대가인 Daniel Lemire의 블로그에 흥미로운 포스트가 하나 올라왔다. "Prefix sums at gigabytes per second with ARM NEON"이라는 도발적인 제목의 글이다. (현재 원본 서버 이슈로 접속이 원활하지 않지만, 제목과 커뮤니티의 반응만으로도 우리가 짚고 넘어가야 할 엔지니어링 포인트는 명확하다.)

데이터 파이프라인이나 데이터베이스 엔진을 밑바닥부터 짜본 엔지니어라면 Prefix sum(누적 합)이 얼마나 자주 등장하는 Bottleneck인지 공감할 것이다. Stream compaction, Lexing, 인덱스 구축 등 수많은 곳에서 쓰인다. 하지만 이전 요소의 결과가 다음 요소의 입력이 되는 본질적인 Data dependency 때문에 병렬화가 까다로운 대표적인 연산이기도 하다.

### Prefix Sum과 SIMD 병렬화의 딜레마

일반적인 루프를 돌며 누적 합을 구하는 것은 CPU의 브랜치 예측기나 파이프라인에 크게 의존하며, 메모리 대역폭을 한참 밑도는 Throughput을 보여준다. 이를 타파하기 위해 x86 진영에서는 AVX-512를 활용한 최적화가 널리 연구되어 왔다.

하지만 이제 서버룸의 대세는 AWS Graviton을 필두로 한 ARM 아키텍처다. ARM 환경에서 초당 수십 기가바이트(GB/s)의 Throughput을 뽑아내려면 결국 128-bit 폭의 ARM NEON 인스트럭션을 극한으로 쥐어짜야 한다. 레지스터 내에서 데이터를 시프트하고 더하는 방식을 통해 데이터 의존성 사슬을 끊어내고, CPU 캐시 라인에 맞춰 메모리 액세스 패턴을 최적화하는 것이 핵심이다.

솔직히 말해, 2024년이 넘은 시점에 여전히 128-bit NEON intrinsics를 수동으로 작성하고 있는 현실은 시스템 엔지니어로서 꽤나 피곤한 일이다. 과거 2018년경에 비슷한 최적화 작업을 하면서 "몇 년 뒤면 가변 길이 벡터(SVE)가 표준이 되어서 이런 노가다는 안 해도 되겠지"라고 생각했던 내 예상이 완전히 빗나갔기 때문이다.

### Hacker News의 반응: SVE2는 대체 어디에 있는가?

이 글이 Hacker News에 올라오자마자, 커뮤니티의 논의는 NEON 자체의 경이로움보다는 **ARM 생태계의 SIMD 파편화 문제** 로 불타올랐다.

가장 눈에 띄는 댓글은 단연 Apple Silicon에 대한 성토였다.
> "What's going on with SVE2 support in the ARM land? It's weird that even Apple's M5 still doesn't support it (other than SME2)."

나 역시 이 의견에 100% 동의한다. Apple은 M시리즈 칩셋으로 ARM의 데스크탑/랩톱 시대를 열었지만, 정작 개발자들이 목빠지게 기다리는 SVE(Scalable Vector Extension) 지원에는 인색하다. M5에서조차 SVE2를 건너뛰고 머신러닝 워크로드에 특화된 SME2(Scalable Matrix Extension)만 챙기는 모습은 철저히 자사 생태계와 AI 마케팅에만 집중하겠다는 의도로 읽힌다.

반면, Radxa Orion O6 같은 칩셋에서는 SVE2를 지원한다는 언급도 있었다. 이는 결국 ARM 기반 소프트웨어를 작성할 때, 하드웨어 파편화로 인해 동일한 코드베이스로 최적의 성능을 보장하기가 점점 더 어려워지고 있다는 것을 의미한다.

### 결론: 우리는 여전히 NEON을 써야 한다

Daniel Lemire가 굳이 최신 SVE2가 아닌 NEON을 타겟으로 GB/s급 Prefix sum을 구현한 이유는 명확하다. 지금 당장 프로덕션에서 신뢰하고 쓸 수 있는 가장 보편적인 ARM SIMD 표준은 **여전히 NEON** 이기 때문이다.

SVE2가 이상적인 미래임은 분명하지만, Apple과 클라우드 벤더들의 파편화된 도입 속도를 고려할 때, 앞으로 최소 3~5년은 NEON 기반의 최적화 코드가 현역으로 뛸 수밖에 없다. 고성능 데이터 처리 엔진을 개발 중이라면, SVE2가 모든 것을 해결해 줄 것이라는 환상을 버리고 당장 NEON intrinsics와 친해지거나, Google의 Highway 같은 SIMD 추상화 라이브러리를 적극 도입하는 것을 추천한다.

### References

- **원본 아티클:** https://lemire.me/blog/2026/03/08/prefix-sums-at-tens-of-gigabytes-per-second-with-arm-neon/
- **Hacker News 스레드:** https://news.ycombinator.com/item?id=47301017
