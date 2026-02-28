---
title: "The Ultimate Quine: How 45 Nucleotides Solved the Bootstrapping Problem"
description: "The Ultimate Quine: How 45 Nucleotides Solved the Bootstrapping Problem"
pubDate: "2026-02-28T07:14:12Z"
---

개발자라면 누구나 한 번쯤 **Quine(콰인)** 을 짜본 경험이 있을 겁니다. 자신의 소스 코드를 그대로 출력하는 프로그램 말이죠. 이건 일종의 지적 유희이자 퍼즐이지만, 자연(Nature)에게 있어서 자기 복제(Self-replication)는 생존 그 자체입니다.

최근 *Science*에 실린 논문 하나가 Hacker News를 뜨겁게 달구고 있습니다. 바로 **Qt45** 라는 RNA 효소(Ribozyme)에 관한 이야기입니다. 이 녀석은 단 45개의 뉴클레오타이드(nucleotide)만으로 구성되어 있는데, 놀랍게도 **스스로를 합성(synthesize itself)** 할 수 있습니다.

엔지니어링 관점에서 보면, 이건 생명이라는 거대한 레거시 시스템의 **Bootstrapping** 코드를 발견한 것과 다름없습니다. 오늘은 이 발견이 왜 기술적으로 흥미로운지, 그리고 확률론적 관점에서 우리가 '생명의 기원'을 어떻게 다시 바라봐야 하는지 이야기해 보려 합니다.

## The MVP of Life: 45 Bytes

우리가 흔히 생명체를 복잡한 마이크로서비스 아키텍처에 비유한다면, Qt45는 그야말로 **Minimum Viable Product (MVP)** 입니다. 단 45개의 염기서열. 컴퓨터로 치면 45바이트도 안 되는 코드가 자기 자신을 복제하는 컴파일러이자 소스 코드인 셈입니다.

Hacker News의 한 유저가 지적했듯이, 이 정도 길이의 시퀀스가 우연히 만들어질 확률은 약 $2^{-90}$입니다. 숫자로만 보면 "말도 안 되는 확률" 같죠? 하지만 엔지니어링에서 **Scale** 은 언제나 확률을 압도합니다.

> "Getting this sequence by random chance out of a pile of nucleotides is a 1 in 2^90 chance... Not at all an impossible number."

지구상의 바다, 혹은 소행성(Bennu 같은)에 존재하는 뉴클레오타이드의 양(Mole 단위)을 대입해 보면 이야기가 달라집니다. 소행성 하나만 해도 약 20,000 몰(moles)의 뉴클레오타이드를 전달할 수 있다고 합니다. 이걸 전체 우주, 혹은 초기 지구의 바다라는 스케일로 확장하면 $2^{90}$분의 1이라는 확률은 **Brute-force Attack** 으로 충분히 뚫을 수 있는 수준이 됩니다.

즉, 생명의 탄생은 '기적'이 아니라, 충분한 컴퓨팅 파워(화학 반응)와 시간만 있으면 필연적으로 발생하는 **Deterministic** 한 결과일 수 있다는 겁니다.

## Performance vs. Functionality

하지만 이 Qt45에도 치명적인 단점이 있습니다. 바로 **Throughput** 입니다.

논문에 따르면 이 친구가 자신을 복제하는 데 걸리는 시간은 엄청나게 깁니다. 72일(약 1,700시간) 동안 고작 0.2%의 수율(yield)을 보였다고 합니다. 2009년에 나왔던 다른 연구(Lincoln & Joyce)가 1시간 내에 배가(doubling)되었던 것과 비교하면 처참한 성능이죠.

하지만 여기서 중요한 차이점이 있습니다. 2009년의 연구는 이미 만들어진 덩어리들을 이어 붙이는(ligation) 방식에 가까웠다면, Qt45는 **"Drawing the whole owl"**, 즉 바닥부터 진짜 합성을 해내는 것에 가깝습니다.

엔지니어로서 저는 이 부분에서 전율을 느꼈습니다. 초기 구현체(Prototype)는 당연히 느립니다. 최적화(Optimization)는 나중 문제입니다. 일단 **동작(Working)** 하는 최소한의 모델이 나왔다는 것, 그것도 45라는 믿기지 않을 정도로 작은 크기로 구현됐다는 것이 핵심입니다. 일단 복제가 시작되면, 그 뒤의 성능 개선은 '진화(Evolution)'라는 강력한 알고리즘이 해결해 줄 테니까요.

## Cloud vs. On-Premise: 기원은 어디인가?

댓글 타래에서는 이 재료들이 우주에서 왔느냐(Panspermia), 지구에서 자연 발생했느냐에 대한 토론도 활발합니다. 한 유저는 "우주가 지구보다 훨씬 오래되었으니, 행성 간 화학 반응이 주된 공급원일 수 있다"라고 주장합니다.

개인적으로 이 논쟁은 **Cloud vs. On-Premise** 논쟁처럼 느껴집니다. 소스가 어디에 배포되어 있었든(우주 공간이든 원시 지구의 바다든), 중요한 건 실행 환경(Runtime Environment)이 갖춰졌을 때 코드가 돌기 시작했다는 사실입니다. 재료(뉴클레오타이드)는 우주 어디에나 널려 있습니다. 병목(Bottleneck)은 재료가 아니라, 그 재료들이 유의미한 기능을 하는 구조로 조립되는 **확률** 이었는데, Qt45는 그 문턱이 우리가 생각했던 것보다 훨씬 낮을 수 있음을 시사합니다.

## Verdict: It's Just Chemistry

솔직히 말해서, 저는 오랫동안 생명의 기원이 일종의 '스파게티 코드' 덩어리에서 우연히 시작된 복잡한 기적이라고 생각했습니다. 하지만 Qt45는 생명의 시작이 **Elegant** 한 몇 줄의 코드였을 수 있음을 보여줍니다.

이 발견은 우리에게 두 가지를 시사합니다:
1.  **Simplicity:** 복잡해 보이는 시스템도 그 코어(Core)는 놀라울 정도로 단순할 수 있다.
2.  **Inevitability:** 충분한 리소스와 시간이 주어진다면, 복잡계는 필연적으로 자기 조직화(Self-organization)한다.

우리는 지금 생물학적 **Singularity** 의 흔적을 보고 있는 것일지도 모릅니다. 45개의 염기서열이 30억 년 뒤에 이 글을 쓰고 있는 인간으로 진화했다니, 어떤 면에서는 지구상에서 가장 성공한 오픈소스 프로젝트가 아닐까요?

---

**References:**
- Original Article: [Science.org](https://www.science.org/doi/10.1126/science.adt2760)
- Hacker News Thread: [Hacker News](https://news.ycombinator.com/item?id=47187649)
