---
title: "박테리아보다 작은 QR 코드: 이것이 차세대 Cold Storage의 미래일까?"
description: "박테리아보다 작은 QR 코드: 이것이 차세대 Cold Storage의 미래일까?"
pubDate: "2026-02-24T14:59:31Z"
---

QR 코드는 이제 우리 일상 어디에나 있습니다. 식당 메뉴판부터 결제까지, 솔직히 엔지니어 입장에서 QR 코드는 더 이상 '혁신'이라기보다는 '레거시(Legacy) 인터페이스'에 가깝게 느껴지곤 합니다. 그런데 최근 오스트리아의 비엔나 공과대학교(TU Wien)에서 발표한 연구 결과는 꽤 흥미롭습니다. 단순히 작게 만들어서 기네스북에 올랐다는 사실 때문만이 아닙니다. 이 기술이 **데이터 스토리지(Data Storage)**, 특히 장기 보존을 위한 아카이빙 기술의 새로운 가능성을 시사하고 있기 때문입니다.

오늘은 이 '세계에서 가장 작은 QR 코드'가 어떻게 만들어졌는지, 그리고 이것이 왜 단순한 기술 과시용이 아닌지 엔지니어링 관점에서 뜯어보겠습니다.

## 1.98 제곱마이크로미터의 세계

우선 스펙부터 살펴봅시다. 이번에 기록을 경신한 QR 코드의 크기는 **1.98 µm²** 입니다. 이전 기록이 독일 뮌스터 대학의 5.38 µm²였으니, 거의 3배 가까이 줄인 셈입니다. 이게 어느 정도 크기인지 감이 안 오신다면, 사람의 적혈구보다 훨씬 작고 일반적인 박테리아보다도 작다고 생각하시면 됩니다.

![The QR code is 'written' on to a thin ceramic film using focused ion beams](https://assets.newatlas.com/dims4/default/04215cc/2147483647/strip/true/crop/1200x800+0+0/resize/1200x800!/format/webp/quality/90/?url=https%3A%2F%2Fnewatlas-brightspot.s3.amazonaws.com%2Fdc%2F07%2F838738d84cf58f90bf7d94643f34%2Fthe-qr-code-is-written-on-to-a-thin-ceramic-film-using-focused-ion-beams.jpg)

이 정도 크기면 가시광선의 파장 한계 때문에 일반적인 광학 현미경으로는 절대 볼 수 없습니다. 읽으려면 **전자 현미경(SEM)** 이 필요하죠. 픽셀 하나의 크기가 **49나노미터(nm)** 에 불과하니까요.

### 어떻게 만들었나: FIB (Focused Ion Beam)

여기서 기술적으로 짚고 넘어가야 할 부분이 있습니다. 많은 매체에서 '전자 현미경 기술'이라고 뭉뚱그려 표현하지만, 실제 제작 방식은 **FIB(Focused Ion Beam)** 밀링입니다. 해커뉴스(Hacker News)의 한 유저가 지적했듯이, 전자가 아니라 이온 빔을 사용하여 물리적으로 세라믹 필름을 깎아낸 것입니다.

- **소재:** 얇은 세라믹 필름 (Ceramic Film)
- **방식:** 이온 빔을 이용한 에칭/밀링
- **협력:** Cerabyte (장기 데이터 스토리지 스타트업)

세라믹을 선택한 이유는 명확합니다. **내구성(Durability)** 때문입니다. 이 연구의 본질은 'QR 코드를 작게 만드는 것'이 아니라, 극한의 환경에서도 데이터가 손실되지 않는 **고밀도 아카이빙 매체** 를 만드는 데 있으니까요.

## 엔지니어의 시선: 이게 왜 중요한가?

솔직히 "QR 코드를 전자 현미경으로 읽어야 한다"는 말은 실용성 면에서 빵점처럼 들립니다. 편의점에서 결제하려고 전자 현미경을 들고 다닐 수는 없으니까요. 하지만 이 기술의 타겟은 컨슈머가 아니라 **데이터 센터의 Cold Storage** 입니다.

연구진은 이 기술을 확장하면 **A4 용지 크기의 세라믹 필름 한 장에 2TB의 데이터를 저장** 할 수 있다고 주장합니다. 이건 꽤 의미 있는 수치입니다.

![This is the smallest QR code in the world, seen here through an electron microscope](https://assets.newatlas.com/dims4/default/7dcc525/2147483647/strip/true/crop/1200x854+0+0/resize/1200x854!/format/webp/quality/90/?url=https%3A%2F%2Fnewatlas-brightspot.s3.amazonaws.com%2Fdd%2F66%2F68110d5f4d57986aab81e20e08f5%2Fthis-is-the-smallest-qr-code-in-the-world-seen-through-an-electron-microscope.jpg)

현재 우리는 '데이터 폭증'의 시대에 살고 있습니다. AWS Glacier나 Azure Archive 같은 서비스들이 있지만, 여전히 많은 데이터가 자기 테이프(Magnetic Tape)나 HDD에 의존합니다. 하지만 이들은 수명이 있고, 주기적인 마이그레이션이 필요합니다. 반면 세라믹이나 유리에 새겨진 데이터는 이론상 수천 년을 버틸 수 있습니다. 마이크로소프트의 **Project Silica** (유리에 레이저로 데이터 저장)와 궤를 같이하는 기술이라고 볼 수 있죠.

## 비판적 시각: 현실성은 있는가?

하지만 Principal Engineer로서 비판적인 시각을 거둘 수는 없습니다. 실제 이미지를 자세히 보면 몇 가지 우려 사항이 보입니다.

1.  **신호 대 잡음비 (SNR) 문제:**
    해커뉴스에서도 지적된 부분인데, 결과물을 보면 픽셀의 경계가 명확하지 않고 뭉개져(smudged/melted) 보입니다. QR 코드의 강력한 에러 보정(Error Correction) 알고리즘 덕분에 읽히는 것이지, Raw Data의 품질 자체는 아슬아슬해 보입니다. 대용량 데이터를 저장할 때 이런 노이즈는 심각한 **Bit Error Rate** 로 이어질 수 있습니다.

2.  **Read/Write Throughput:**
    FIB로 데이터를 쓰고, SEM으로 데이터를 읽는다? 이건 속도 면에서 끔찍하게 느릴 겁니다. 2TB를 저장할 수 있다 해도, 그걸 기록하고 읽어내는 데 며칠이 걸린다면 상용화는 요원합니다. Cerabyte와 같은 스타트업이 해결해야 할 가장 큰 숙제는 바로 이 **I/O 병목** 일 것입니다.

## 마치며: 장난감인가, 혁신인가?

이번 TU Wien의 성과는 재료 공학적으로는 놀라운 쾌거입니다. 49nm 픽셀을 안정적으로 세라믹에 새겼다는 건 분명 대단한 기술입니다. 하지만 이것이 당장 우리의 데이터 센터를 바꿀 수 있을까요? **아직은 아닙니다.**

DNA 스토리지, 유리에 새기는 홀로그래픽 스토리지, 그리고 이번 세라믹 스토리지까지. 차세대 아카이빙 전쟁은 이제 막 시작되었습니다. 개인적으로는 이 기술이 QR 코드라는 친숙한 포맷을 벗어나, 자체적인 고밀도 인코딩 스킴을 적용했을 때 어떤 퍼포먼스를 보여줄지 기대됩니다.

결국 승자는 '얼마나 많이 저장하느냐'가 아니라, **'얼마나 싸고 빠르게 읽고 쓸 수 있느냐'** 에서 판가름 날 테니까요.

---

**References:**
- [Original Article (New Atlas)](https://newatlas.com/technology/smallest-qr-code-bacteria-tu-wien/)
- [Hacker News Discussion](https://news.ycombinator.com/item?id=47097405)
