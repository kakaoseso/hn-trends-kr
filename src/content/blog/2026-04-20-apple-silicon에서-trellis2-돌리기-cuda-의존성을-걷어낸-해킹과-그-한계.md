---
title: "Apple Silicon에서 TRELLIS.2 돌리기: CUDA 의존성을 걷어낸 해킹과 그 한계"
description: "Apple Silicon에서 TRELLIS.2 돌리기: CUDA 의존성을 걷어낸 해킹과 그 한계"
pubDate: "2026-04-20T01:56:32Z"
---

최근 Microsoft에서 공개한 Image-to-3D 모델인 TRELLIS.2는 꽤나 인상적인 결과물을 보여줍니다. 단일 이미지에서 고품질의 3D 메쉬를 뽑아내는 속도와 디테일이 훌륭하죠. 하지만 새로운 SOTA(State-of-the-Art) 모델이 릴리즈될 때마다 우리 Mac 유저들이 겪는 익숙한 좌절감이 있습니다. 바로 소스코드 곳곳에 하드코딩된 `.cuda()` 호출과 NVIDIA 전용 커널 라이브러리들입니다.

로컬에서 빠르게 테스트해보고 싶은데, 결국 클라우드 GPU 인스턴스를 띄워야 하는 상황. 다들 익숙하실 겁니다. 그런데 최근 GitHub에 아주 흥미로운 프로젝트가 올라왔습니다. **TRELLIS.2를 Apple Silicon(MPS)에서 네이티브로 구동** 할 수 있게 포팅한 `trellis-mac` 레포지토리입니다.

솔직히 처음 이 프로젝트를 봤을 때는 그저 PyTorch의 device 설정을 `mps`로 바꾼 단순한 래퍼(wrapper) 스크립트일 거라고 생각했습니다. 하지만 코드를 뜯어보니 생각보다 훨씬 깊은 수준의 엔지니어링 삽질이 들어가 있었습니다. 오늘은 이 포팅이 기술적으로 어떻게 이루어졌는지, 그리고 그 이면에 숨겨진 Trade-off는 무엇인지 파헤쳐 보겠습니다.

## 어떻게 CUDA 의존성을 걷어냈는가?

단순히 `device="cuda"`를 `device="mps"`로 바꾼다고 모델이 돌아가지 않습니다. 현대의 딥러닝 모델들은 퍼포먼스를 쥐어짜내기 위해 C++과 CUDA로 작성된 커스텀 커널에 심하게 의존하고 있기 때문입니다. TRELLIS.2 역시 예외는 아니었으며, 이 프로젝트의 저자는 핵심 병목들을 순수 PyTorch와 Python으로 재구현(Re-implementation)하는 방식을 택했습니다.

### 1. Sparse 3D Convolution (`flex_gemm`의 대체)

가장 큰 난관은 Sparse 3D Convolution이었습니다. 원본 코드는 이를 위해 CUDA 기반의 `flex_gemm`을 사용합니다. 저자는 이를 `backends/conv_none.py`에서 **Gather-Scatter** 방식을 사용한 순수 PyTorch 코드로 대체했습니다.

활성화된 복셀(voxel)들의 공간 해시(spatial hash)를 만들고, 각 커널 위치에 대해 이웃 피처들을 Gather한 뒤, 행렬 곱셈(Matrix Multiplication)으로 가중치를 적용하고, 그 결과를 다시 Scatter-add 하는 방식입니다. 중복 연산을 피하기 위해 이웃 맵을 텐서 단위로 캐싱하는 최적화도 들어갔습니다. 이론적으로는 훌륭하지만, 여기서 엄청난 성능 저하가 발생합니다. CUDA 커널 대비 약 10배 느려지는 메인 병목 지점입니다.

### 2. Attention Module의 현대화 (`flash_attn` -> SDPA)

과거에는 `flash_attn` 라이브러리가 필수였지만, 이제는 다행히 PyTorch가 네이티브로 `scaled_dot_product_attention`(SDPA)을 지원합니다. 저자는 Sparse Attention 모듈에 SDPA 백엔드를 추가했습니다. 가변 길이의 시퀀스를 배치로 패딩(Padding)한 뒤 SDPA를 통과시키고 다시 언패딩하는 식입니다. 이 부분은 꽤나 우아하게 해결되었습니다.

### 3. Mesh Extraction의 Pythonic한 접근

원본은 듀얼 복셀 그리드에서 메쉬를 추출할 때 CUDA 해시맵(`o_voxel._C hashmap`)을 사용합니다. 포팅 버전에서는 이를 순수 Python 딕셔너리로 대체했습니다. 좌표-인덱스 룩업 테이블을 만들고 연결된 복셀을 찾아 쿼드(quad)를 삼각화(triangulation)합니다. C++의 포인터 연산과 해시맵을 Python 딕셔너리로 바꿨으니 오버헤드는 뻔하지만, 어쨌든 '돌아가게' 만들었다는 점에 점수를 주고 싶습니다.

## 과감한 포기: Graceful Degradation

이 프로젝트가 영리한 점은, 모든 것을 완벽하게 포팅하려다 실패하는 대신 **핵심이 아닌 기능은 과감히 버렸다** 는 것입니다.

- **Texture Export (텍스처 베이킹):** `nvdiffrast`라는 미분 가능 래스터라이저(Differentiable Rasterizer)가 필요한데, 이는 순수 CUDA 라이브러리입니다. 저자는 이를 Stub 처리하여 무시하고, 대신 정점 색상(Vertex colors)만 있는 OBJ/GLB를 출력하도록 타협했습니다.
- **Hole Filling (메쉬 구멍 메우기):** `cumesh`라는 또 다른 CUDA 라이브러리가 필요합니다. 이 역시 비활성화하여, 결과물에 작은 구멍이 생길 수 있는 한계를 인정했습니다.

## 성능과 한계: 10배의 페널티

M4 Pro (24GB Unified Memory) 기준으로 512 파이프라인 파라미터를 돌렸을 때의 벤치마크는 다음과 같습니다.

- **Model loading:** ~45s
- **Shape SLat sampling:** ~90s
- **Texture SLat sampling:** ~50s
- **Total Time:** 약 3.5분

생성 중 메모리 사용량은 약 18GB에 달합니다. 4B 모델을 로드하고 연산하기에는 24GB 메모리가 거의 마지노선으로 보입니다.

Hacker News의 한 유저(anon)는 이렇게 꼬집었습니다. *"MPS 백엔드로 돌리는 건 원래 가능했습니다. 사람들이 HuggingFace 스페이스나 데모에서 굳이 안 쓰는 이유는, 단지 호환성을 위해 속도를 10배나 희생하고 싶지 않기 때문이죠."*

저도 이 의견에 동의합니다. 프로덕션 환경에서 3.5분이라는 Latency는 사실상 사용 불가능한 수치입니다. 순수 PyTorch로 구현된 Sparse Convolution은 유연하지만, 하드웨어 가속을 제대로 받지 못해 Throughput을 심각하게 깎아먹습니다.

## 총평: NVIDIA의 진정한 해자(Moat)

이 프로젝트를 보며 2018년쯤 우리가 무거운 비전 모델을 모바일 디바이스로 포팅하기 위해 온갖 C++ 커널을 깎던 시절이 떠올랐습니다. 그때나 지금이나 하드웨어의 파편화는 엔지니어들의 몫입니다.

이 포팅은 훌륭한 엔지니어링 쇼케이스이자, 로컬에서 모델 구조를 뜯어보고 실험하려는 연구자들에게는 가뭄의 단비 같은 도구입니다. 하지만 실무에 투입할 수 있는 수준은 아닙니다. 텍스처 베이킹이 안 되고 메쉬에 구멍이 뚫려 나오는 3D 에셋을 그대로 쓸 파이프라인은 없으니까요.

무엇보다 이 레포지토리는 **NVIDIA의 진정한 해자가 GPU 하드웨어 그 자체가 아니라, `flex_gemm`, `nvdiffrast` 같이 수년간 축적된 CUDA 생태계와 커스텀 커널들** 에 있다는 것을 적나라하게 보여줍니다. Apple Silicon의 깡성능이 아무리 좋아져도, SOTA 모델들이 이 CUDA 생태계 위에서 작성되는 한 Mac에서의 AI는 당분간 '10배 느린 해킹'에 머물 수밖에 없을 것 같습니다.

---

### References
- Original Article (GitHub): https://github.com/shivampkumar/trellis-mac
- Hacker News Thread: https://news.ycombinator.com/item?id=47828896
