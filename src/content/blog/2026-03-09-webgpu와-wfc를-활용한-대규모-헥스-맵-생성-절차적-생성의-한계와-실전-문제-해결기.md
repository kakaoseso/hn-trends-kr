---
title: "WebGPU와 WFC를 활용한 대규모 헥스 맵 생성: 절차적 생성의 한계와 실전 문제 해결기"
description: "WebGPU와 WFC를 활용한 대규모 헥스 맵 생성: 절차적 생성의 한계와 실전 문제 해결기"
pubDate: "2026-03-09T21:11:09Z"
---

절차적 생성(Procedural Generation)은 밖에서 보면 마법 같지만, 막상 직접 구현해보면 끝없는 상태 공간 폭발(Combinatorial Explosion) 및 예외 처리와 싸우는 지루하고 고통스러운 과정이다.

최근 Felix Turner가 공개한 [Building a Procedural Hex Map with Wave Function Collapse](https://felixturner.github.io/hex-map-wfc/article/) 프로젝트를 흥미롭게 살펴봤다. WebGPU와 Three.js를 사용해 4,100개의 헥스(Hex) 타일로 이루어진 3D 중세 섬을 60fps로 생성해내는 이 프로젝트는, 단순히 시각적으로 아름다울 뿐만 아니라 엔지니어링 관점에서 굉장히 실용적인 문제 해결 방식을 보여준다.

오늘은 이 프로젝트가 어떻게 WFC의 한계를 극복했는지, 그리고 렌더링 파이프라인을 어떻게 최적화했는지 시니어 엔지니어의 시각에서 파헤쳐보자.

## WFC의 본질과 헥스 그리드의 저주

Wave Function Collapse(WFC)는 보드게임 '카르카손(Carcassonne)'을 생각하면 이해하기 쉽다. 인접한 타일의 엣지(Edge) 조건이 서로 일치하도록 타일을 배치하는 제약 충족 문제(Constraint Satisfaction Problem)다.

문제는 이 프로젝트가 사각형이 아닌 헥스(Hex) 그리드를 사용한다는 점이다. 타일의 엣지가 4개에서 6개로 늘어나면 제약 조건은 50% 증가하지만, 조합 가능한 경우의 수는 기하급수적으로 폭발한다. 여기에 5단계의 고도(Elevation) 개념까지 추가되면서 2D 제약 문제가 3D 문제로 변질되었다.

```javascript
// 3방향 교차로 타일의 정의 예시
{ 
  name: 'ROAD_D', 
  mesh: 'hex_road_D',
  edges: { NE: 'road', E: 'grass', SE: 'road', SW: 'grass', W: 'road', NW: 'grass' },
  weight: 2 
}
```

저자는 이 복잡도를 해결하기 위해 맵을 19개의 독립적인 서브 그리드로 나누어 처리하는 Multi-Grid 방식을 채택했다. 하지만 그리드를 나누면 필연적으로 '경계면 문제'가 발생한다.

## WFC의 불편한 진실: 실패를 우회하는 3단계 복구 시스템

WFC 알고리즘의 가장 큰 비밀은 **"자주 실패한다"** 는 것이다. 엔트로피가 가장 낮은 셀을 찾아 붕괴(Collapse)시키고 제약을 전파(Propagate)하다 보면, 결국 어떤 셀에도 유효한 타일을 놓을 수 없는 막다른 골목(Dead end)에 다다르게 된다.

저자는 단순한 백트래킹(Backtracking)만으로는 교착 상태를 해결할 수 없음을 깨닫고, 다음과 같은 3단계 복구(Recovery) 시스템을 구축했다. 나는 이 부분이 전체 아키텍처에서 가장 빛나는 실용주의적 접근이라고 생각한다.

- **Layer 1 (Unfixing):** 충돌을 유발한 인접 셀의 고정 상태를 풀고 다시 계산 가능한 상태로 되돌린다.
- **Layer 2 (Local-WFC):** 실패 지점 주변의 반경 2칸(Radius-2) 영역만 떼어내어 미니 WFC를 다시 돌린다. 불가능한 전체를 풀려 하지 않고, 국소적인 경계 조건을 변경해버리는 훌륭한 휴리스틱이다.
- **Layer 3 (Drop and hide):** 최후의 수단. 충돌이 난 셀을 아예 날려버리고 그 자리에 '산(Mountain)' 타일을 덮어버린다.

"아무도 산이 왜 거기 있는지 의심하지 않는다(Nobody questions a mountain)."

이 문장에서 헛웃음이 나왔다. 엄격한 엔터프라이즈 시스템이었다면 제약 충돌은 Fatal Error를 뱉고 뻗어버릴 문제다. 하지만 게임 맵 생성에서는 그저 절벽이나 산을 하나 스폰하면 그만이다. 완벽한 수학적 해를 구하는 것보다 '그럴듯한 결과물'을 내는 것이 더 중요하다는 도메인의 특성을 완벽히 이해한 엔지니어링이다.

## 도구의 한계 인정하기: WFC와 Perlin Noise의 분업

주니어 시절 흔히 하는 실수 중 하나는 새롭고 멋진 알고리즘 하나로 모든 문제를 해결하려 드는 것이다. 저자 역시 초기에는 나무나 건물의 배치까지 WFC로 해결하려 했으나 실패했다고 고백한다.

WFC는 국소적인 엣지 매칭(Local constraints)에는 탁월하지만, 숲이나 마을 같은 거시적인 군집 패턴(Global patterns)을 만드는 데는 끔찍하게 무능하다. 결국 지형 생성은 WFC에 맡기고, 나무와 건물의 배치는 전통적이고 검증된 Perlin Noise에 맡겼다. 각 도구가 가장 잘하는 일만 하도록 책임을 분리한 것이다.



## 성능 최적화: WebGPU와 BatchedMesh

4,100개가 넘는 타일과 수많은 장식물들을 브라우저에서 60fps로 렌더링하는 것은 쉽지 않다. 저자는 여기서 두 가지 핵심 최적화 기법을 사용했다.

- **BatchedMesh:** 19개의 각 그리드마다 타일용, 장식물용으로 단 2개의 BatchedMesh만 생성했다. GPU가 인스턴스별 트랜스폼을 처리하므로 CPU 오버헤드가 거의 0에 수렴한다.
- **Shared Material:** 씬 내의 모든 메시는 단 하나의 단일 재질(Material)을 공유하며, UV 좌표를 통해 팔레트 텍스처에서 색상을 가져온다.

렌더링 파이프라인에서 State Switch(상태 변경)는 Throughput을 갉아먹는 주적이다. 단일 재질과 BatchedMesh의 조합으로 전체 씬을 단 몇 번의 Draw call만으로 처리해낸 점은 매우 교과서적이고 훌륭한 WebGL/WebGPU 최적화 사례다. 이렇게 아낀 GPU 자원은 GTAO(Global Texture Ambient Occlusion)나 DoF(Depth of Field) 같은 포스트 프로세싱에 투자하여 비주얼 퀄리티를 끌어올렸다.

## Hacker News의 시선: 더 나은 알고리즘이 있을까?

[Hacker News 스레드](https://news.ycombinator.com/item?id=47311815)에서는 이 문제에 대한 흥미로운 학구적 토론이 이어졌다.

많은 개발자들이 단순 백트래킹 대신 Knuth의 Algorithm X(Dancing Links)를 사용하거나, Minizinc, Clingo 같은 SAT Solver 기반의 제약 프로그래밍(Constraint Programming)을 도입하면 86%의 성공률을 훨씬 끌어올릴 수 있을 것이라고 지적했다.

이론적으로는 전적으로 동의한다. Constraint Solver를 사용하면 더 우아하게 해를 찾을 수 있다. 하지만 실무자의 관점에서는 약간 생각이 다르다. SAT Solver는 구현과 디버깅이 까다롭고 무겁다. 반면 WFC는 로직이 단순하고(Brute-force-simple), 연산 비용이 저렴하다. 게임 프로그래밍에서는 완벽한 해를 찾기 위해 몇 분을 기다리는 것보다, 가끔 산(Mountain)을 덮어씌우는 꼼수를 쓰더라도 20초 만에 맵을 뽑아내는 단순한 아키텍처가 훨씬 가치 있다.

## 결론

Felix Turner의 이번 프로젝트는 단순히 예쁜 데모를 넘어, 절차적 생성 시스템을 구축할 때 마주하는 현실적인 문제들을 어떻게 실용적으로 우회할 수 있는지 보여주는 훌륭한 레퍼런스다.

특히 WebGPU와 TSL(Three.js Shading Language)의 조합이 이제 브라우저 위에서도 네이티브 수준의 복잡한 렌더링 파이프라인을 충분히 감당할 수 있을 만큼 성숙했다는 것을 증명했다. 3D 웹 인터랙션이나 절차적 생성에 관심이 있는 엔지니어라면, 반드시 코드를 클론해서 뜯어볼 가치가 있다.

**References**
- Original Article: https://felixturner.github.io/hex-map-wfc/article/
- Hacker News Thread: https://news.ycombinator.com/item?id=47311815
