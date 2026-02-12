---
title: "Python 3.14 Zstd로 ML 모델 없이 텍스트 분류하기: 장난감인가, 실전 도구인가?"
description: "Python 3.14 Zstd로 ML 모델 없이 텍스트 분류하기: 장난감인가, 실전 도구인가?"
pubDate: "2026-02-12T03:16:40Z"
---

요즘 테크 업계는 온통 LLM(Large Language Model) 이야기뿐입니다. 수십억 개의 파라미터, 엄청난 GPU 비용, 그리고 복잡한 파이프라인이 'AI'의 표준처럼 여겨지고 있죠. 그런데 최근 Python 3.14 릴리즈 노트에서 꽤 흥미로운 모듈 하나가 눈에 띄었습니다. 바로 `compression.zstd`입니다.

Facebook이 개발한 **Zstandard(Zstd)** 알고리즘이 이제 파이썬 표준 라이브러리에 들어왔다는 소식인데, 단순히 "압축이 빨라졌다"는 이야기를 하려는 게 아닙니다. 이 모듈을 이용해 **행렬 연산도, 역전파(Backpropagation)도 없이 텍스트 분류기를 만드는 방법** 이 화제가 되고 있기 때문입니다.

Max Halford가 작성한 블로그 포스트와 이에 대한 Hacker News의 논쟁을 바탕으로, 이 기법이 단순한 '장난감'인지 아니면 실무에서 써먹을 만한 '도구'인지 Principal Engineer의 관점에서 뜯어보겠습니다.

## 압축으로 텍스트를 분류한다고?

기본 아이디어는 정보 이론의 **콜모고로프 복잡도(Kolmogorov complexity)** 에 뿌리를 두고 있습니다. 쉽게 말해, **"같은 주제의 텍스트끼리는 서로를 더 잘 압축한다"** 는 원리입니다.

예를 들어보죠. '타코', '부리또', '살사'가 가득한 텍스트 뭉치(Dictionary)를 가지고 있다면, 새로운 문장 "I love guacamole"를 압축할 때 압축률이 매우 높을 겁니다. 반면, '테니스', '라켓', '경기'가 가득한 텍스트 뭉치로 같은 문장을 압축하면 압축률이 떨어지겠죠.

이 원리를 이용해 가장 압축이 잘 되는 카테고리를 찾아내는 것이 이 분류기의 핵심입니다. 2023년에 ACL에 발표된 논문(gzip과 k-NN을 이용한 분류)이 화제가 된 적이 있었는데, 이번 접근 방식은 Python 3.14의 `zstd` 모듈을 활용해 이를 훨씬 더 우아하게, 그리고 **Incremental(점진적)** 하게 풀어냈습니다.

## Python 3.14의 Zstd가 특별한 이유

사실 `gzip`이나 `LZW` 같은 알고리즘도 이론적으로는 스트리밍 압축이 가능합니다. 하지만 Hacker News의 한 유저가 지적했듯, 파이썬의 기존 `gzip` 모듈은 내부 상태(State)를 유지하며 점진적으로 압축하는 API를 제대로 노출하지 않았습니다. 매번 데이터를 처음부터 다시 압축해야 하니 성능이 나올 리가 없었죠.

반면, Python 3.14의 `compression.zstd`는 다릅니다. **ZstdDict** 와 **ZstdCompressor** 를 통해 압축기의 상태를 효율적으로 관리할 수 있습니다.

Max Halford가 제시한 직관적인 예시 코드를 봅시다:

```python
from compression.zstd import ZstdCompressor, ZstdDict

# 1. '타코' 관련 데이터로 딕셔너리 생성
tacos = b"taco burrito tortilla salsa guacamole cilantro lime " * 50
zd_tacos = ZstdDict(tacos, is_raw=True)
comp_tacos = ZstdCompressor(zstd_dict=zd_tacos)

# 2. '테니스' 관련 데이터로 딕셔너리 생성
padel = b"racket court serve volley smash lob match game set " * 50
zd_padel = ZstdDict(padel, is_raw=True)
comp_padel = ZstdCompressor(zstd_dict=zd_padel)

# 3. 새로운 입력 데이터
input_text = b"I ordered three tacos with extra guacamole"

# 4. 압축 결과 비교
print(len(comp_tacos.compress(input_text, mode=ZstdCompressor.FLUSH_FRAME)))
# 결과: 43 (더 작음 -> 타코 클래스로 분류)

print(len(comp_padel.compress(input_text, mode=ZstdCompressor.FLUSH_FRAME)))
# 결과: 51
```

코드가 정말 심플합니다. `comp_tacos` 압축기가 `input_text`를 더 작게 압축했으므로, 이 문장은 '타코' 관련 글이라고 판단하는 겁니다.

## 성능: 생각보다 빠르고 정확하다

저자는 20 Newsgroups 데이터셋으로 벤치마크를 돌렸습니다. 결과는 놀랍습니다.

- **정확도:** 91.0%
- **소요 시간:** 1.9초 (학습 및 분류 포함)

비교군인 **TF-IDF + Logistic Regression** 모델이 91.8%의 정확도를 보였지만 학습/재학습 파이프라인이 훨씬 무겁다는 점을 감안하면, Zstd 방식은 '가성비'가 미쳤다고 볼 수 있습니다. 5년 전 LZW로 구현했을 때 32분이 걸리던 작업이 2초 만에 끝난 것입니다.

특히 이 방식은 **Online Learning** 에 최적화되어 있습니다. 데이터가 들어올 때마다 버퍼에 추가하고 압축기만 다시 빌드하면 끝입니다. 복잡한 경사 하강법(Gradient Descent)도, 하이퍼파라미터 튜닝의 지옥도 없습니다.

## 엔지니어 관점에서의 비판과 한계

하지만 냉정하게 바라봐야 합니다. Hacker News의 반응들도 꽤 날카롭습니다. 몇 가지 포인트에서 이 기술의 한계를 짚어보겠습니다.

### 1. 이것은 결국 '복잡한 단어 매칭'일 뿐이다
한 HN 유저는 이렇게 말했습니다. "Zstd(A+B)의 크기를 Zstd(A) + Zstd(B)와 비교하는 건, 결국 두 문서가 공통된 단어와 구문을 얼마나 공유하는지 측정하는 복잡한 방법에 불과하다." 맞는 말입니다. 이 방식은 문맥(Context)이나 의미(Semantics)를 이해하는 게 아니라, 바이트 패턴의 반복을 찾는 것입니다. '은행(Bank)'이 돈을 넣는 곳인지 강둑인지 구분하는 건 불가능에 가깝습니다.

### 2. 프로덕션 레벨의 확장성?
데이터가 커지면 어떻게 될까요? `ZstdDict`를 생성하고 유지하는 비용은 데이터 양에 비례하지 않지만, 분류를 위해 **모든 클래스의 압축기를 돌려봐야 한다** 는 점은 치명적입니다. 클래스가 1,000개라면? 입력 텍스트 하나를 분류하기 위해 1,000번의 압축 연산을 수행해야 합니다. 전형적인 $O(N)$ 복잡도입니다. 반면, 잘 학습된 신경망 모델은 추론 비용이 클래스 수에 크게 비례하지 않습니다.

### 3. 스피커를 마이크로 쓰는 격
"압축기를 분류기로 쓰는 건 스피커를 마이크로 쓰는 것과 같다. 가능은 하지만, 그 용도로 만들어진 물건은 아니다."라는 비유가 와닿습니다. 재미있는 해킹이고 정보 이론적으로 의미가 있지만, 전문적인 NLP 문제 해결을 위해서는 한계가 명확합니다.

## 결론: 언제 써먹어야 할까?

이 기법을 당장 회사의 메인 추천 시스템이나 스팸 필터에 적용하라고 권하고 싶지는 않습니다. 하지만 **특정 상황** 에서는 매우 강력한 도구가 될 수 있습니다.

1.  **Cold Start 문제 해결:** 데이터가 거의 없는 초기 단계에서, 복잡한 모델 학습 없이 빠르게 베이스라인을 잡아야 할 때.
2.  **임베디드/엣지 환경:** PyTorch나 TensorFlow를 올리기 부담스러운 저사양 환경에서 텍스트 분류가 필요할 때.
3.  **데이터 드리프트 감지:** 메인 모델 옆에서 가볍게 돌리면서, 텍스트 분포가 급격히 변하는지 모니터링하는 용도.

Python 3.14가 정식 릴리즈되면, `import compression.zstd` 한 줄로 이 모든 실험을 해볼 수 있다는 점은 분명 매력적입니다. 엔지니어라면 이런 'Low-Resource' 접근법 하나쯤은 주머니에 넣어두는 것이 좋지 않을까요?

**참고 자료:**
- [Text classification with Python 3.14's zstd module](https://maxhalford.github.io/blog/text-classification-zstd/)
- [Hacker News Discussion](https://news.ycombinator.com/item?id=46942864)
