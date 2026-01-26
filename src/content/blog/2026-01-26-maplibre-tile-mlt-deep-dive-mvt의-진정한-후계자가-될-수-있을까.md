---
title: "MapLibre Tile (MLT) Deep Dive: MVT의 진정한 후계자가 될 수 있을까?"
description: "MapLibre Tile (MLT) Deep Dive: MVT의 진정한 후계자가 될 수 있을까?"
pubDate: "2026-01-26T15:16:11Z"
---

솔직히 말해서, 지난 10년 동안 웹 지도 생태계는 **Mapbox Vector Tile (MVT)** 가 사실상 독재해왔다고 해도 과언이 아닙니다. GeoJSON보다는 훨씬 효율적이고, PBF(Protocol Buffers) 기반이라 작고 빠르죠. 하지만 2014년에 나온 기술입니다. 그동안 브라우저는 WebGL을 넘어 WebGPU 시대로 가고 있고, 우리가 다루는 데이터의 양은 기하급수적으로 늘어났습니다.

최근 MapLibre에서 **MapLibre Tile (MLT)** 이라는 새로운 포맷을 발표했습니다. 단순히 "압축률 좀 좋아진 포맷" 정도가 아닙니다. 내부 구조를 뜯어보면 이건 지도 데이터의 **Apache Arrow** 화(化)라고 볼 수 있습니다. 

오늘 포스팅에서는 MLT의 기술적 특징과 Hacker News에서 오간 엔지니어들의 리얼한 반응, 그리고 현업 엔지니어로서 제가 느끼는 장단점을 깊게 파보겠습니다.

## 1. MVT는 무엇이 문제였나?

MVT는 훌륭하지만, 근본적으로 **Row-oriented** 방식에 가깝습니다. Feature 단위로 데이터를 직렬화하죠. 하지만 렌더링 파이프라인(GPU) 입장에서 생각해보세요. GPU는 같은 속성끼리 묶여있는 배열(Array)을 좋아합니다. 

기존 MVT를 렌더링하려면:
1. PBF 디코딩
2. 객체 파싱 (JS 힙 메모리 사용)
3. WebGL 버퍼로 변환 (Tessellation)
4. GPU 업로드

이 과정에서 메인 스레드 부하가 상당합니다. 특히 복잡한 Basemap을 줌인/줌아웃할 때 버벅거리는 원인이 되기도 하죠.

## 2. MLT의 핵심: Columnar & Modern Encoding

MLT의 설계 철학은 명확합니다. **"Decoding 성능을 극대화하고 GPU 친화적으로 만들자."**

### Columnar Structure (컬럼 지향 구조)
MLT는 데이터 분석 진영의 **Parquet** 나 **Arrow** 처럼 데이터를 컬럼 단위로 저장합니다. 
- **Key:** 같은 타입의 데이터가 모여 있으니 압축 효율이 급격히 올라갑니다.
- **Key:** 필요한 속성만 읽거나, SIMD 같은 최신 CPU 명령어를 활용하기 유리합니다.

Hacker News의 한 유저가 정확히 지적했더군요:
> "Another thing worth mentioning is it's very similar to the structure of columnar formats like Arrow and Parquet. Anyone with familiarity with these formats could build a decoder in a couple of days."

### FastPFOR 압축 알고리즘
여기에 정수 압축을 위해 `FastPFOR`를 사용했습니다. 다만, 이 부분은 논란이 좀 있습니다. 성능은 확실하지만 알고리즘 자체가 꽤 복잡하고 불투명하다(opaque)는 비판이 있죠. 구현 복잡도가 올라가면 생태계 확장에 걸림돌이 될 수 있거든요.

## 3. 압축률: 10%의 함정?

공개된 데모에서 MVT 대비 약 **10% 정도 용량이 줄어든 것** 으로 나옵니다.

- **Before (MVT):** 261.62kb
- **After (MLT):** 237.67kb

"겨우 10%?"라고 생각할 수 있습니다. 하지만 여기서 중요한 건 **Wire Size(전송 용량)** 보다 **Decoding Speed** 입니다. 게다가 아직 초기 단계라 최적화 여지가 많습니다. AWS가 올해 MLT 최적화에 자금을 지원한다고 하니, 이 수치는 더 개선될 겁니다. 

실제로 Planetiler 개발자는 기본 설정만으로도 10% 감소를 확인했고, 추가 최적화를 진행 중이라고 밝혔습니다.

## 4. 생태계 반응: PMTiles와의 궁합

개인적으로 가장 기대되는 부분은 **PMTiles** 와의 결합입니다. PMTiles는 제가 최근 몇 년간 본 가장 우아한(Serverless) 지도 호스팅 솔루션인데, 단일 파일 아카이브에서 Range Request로 필요한 타일만 가져오는 방식이죠.

PMTiles는 컨테이너일 뿐이라 내부에 MVT를 담든 MLT를 담든 상관없습니다. 이미 스펙 업데이트가 논의되고 있고, 이렇게 되면 **"S3 + CloudFront + PMTiles(MLT)"** 조합으로 비용 효율적이면서도 엄청나게 빠른 글로벌 지도 서비스가 가능해집니다.

반면, **Tilemaker** 같은 주요 툴이 당장 지원 계획이 없다는 건 아쉬운 점입니다. 오픈소스 생태계가 파편화될 위험이 있다는 뜻이니까요.

## 5. 어떻게 사용할 수 있나?

이미 MapLibre GL JS 최신 버전은 MLT를 지원합니다. 스타일 JSON에서 `encoding` 속성만 바꿔주면 됩니다.

```json
{
  "sources": {
    "openmaptiles": {
      "type": "vector",
      "url": "https://...",
      "encoding": "mlt" // 여기가 핵심입니다
    }
  }
}
```

## 6. Hacker News의 딴지들 (feat. Mercator Projection)

재미있는 건, HN 스레드의 절반은 기술 이야기 대신 **"왜 아직도 메르카토르 도법(Web Mercator)을 쓰냐, 그린란드가 너무 커 보이지 않냐"** 는 논쟁으로 빠졌다는 점입니다. (개발자 커뮤니티 특유의 현상이죠.)

물론 맞는 말이지만, 엔지니어링 관점에서 메르카토르 도법은 사각형 타일링(Tiling)과 줌 레벨 관리에 최적화되어 있습니다. Apple Maps처럼 줌 레벨에 따라 구체(Globe)에서 평면으로 전환하는 게 베스트지만, 그건 클라이언트 렌더링의 영역이지 파일 포맷(MLT)이 해결할 문제는 아닙니다.

## Verdict: 아직은 'Early Adopter' 단계

MLT는 분명 벡터 타일의 미래입니다. 데이터 구조가 현대적인 하드웨어 트렌드와 일치하기 때문입니다. 하지만 지금 당장 프로덕션의 모든 MVT를 MLT로 변환하는 건 추천하지 않습니다.

1. **Tooling:** 타일을 생성하는 도구(Planetiler 등)가 이제 막 지원을 시작했습니다.
2. **Compatibility:** MapLibre 외의 다른 클라이언트(OpenLayers, Leaflet 플러그인 등) 지원이 미지수입니다.

하지만 **사내 대시보드나 무거운 데이터 시각화 프로젝트** 를 하고 있다면, Planetiler와 MapLibre GL JS 조합으로 PoC를 해볼 가치는 충분합니다. 특히 클라이언트 측 디코딩 부하를 줄여야 하는 모바일 환경이라면 더더욱요.

지도 데이터의 'Arrow 모먼트'가 오고 있습니다. 준비해두시죠.
