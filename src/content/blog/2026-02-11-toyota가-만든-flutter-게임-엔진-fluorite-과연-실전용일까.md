---
title: "Toyota가 만든 Flutter 게임 엔진 Fluorite: 과연 실전용일까?"
description: "Toyota가 만든 Flutter 게임 엔진 Fluorite: 과연 실전용일까?"
pubDate: "2026-02-11T17:20:38Z"
---

Flutter 생태계에 있다 보면 항상 마주치는 딜레마가 있습니다. **"2D UI는 환상적인데, 3D는 어떻게 하지?"**

물론 `Flame` 같은 훌륭한 게임 엔진이 있지만, 2D 중심입니다. 3D로 넘어가면 Unity를 `FlutterUnityWidget`으로 억지로 구겨 넣거나, 생소한 3D 라이브러리와 씨름해야 했죠. 그런데 뜬금없이 **Toyota(도요타)** 가 이 판에 등판했습니다. 이름하여 **Fluorite**.

처음엔 "자동차 회사가 무슨 게임 엔진이야?" 하고 넘기려 했는데, 스펙을 뜯어보니 이건 장난감이 아닙니다. 오늘은 이 'Console-grade'라고 주장하는 엔진이 기술적으로 어떤 의미를 갖는지, 그리고 왜 하필 도요타인지 엔지니어 관점에서 딥다이브 해보겠습니다.

---

## 1. 아키텍처: C++ ECS와 Dart의 하이브리드

가장 먼저 눈에 띄는 건 **ECS(Entity-Component-System)** 아키텍처입니다. Fluorite는 성능이 중요한 코어 로직을 **C++** 로 작성했습니다.

이건 기술적으로 매우 타당한 선택입니다. Dart는 훌륭한 언어지만, 가비지 컬렉션(GC)이 존재하고 JIT/AOT 컴파일 특성상 매 프레임 수천 개의 오브젝트를 생성하고 파괴하는 고사양 게임 로직에는 한계가 있습니다. Fluorite는 무거운 연산은 C++ 레벨에서 처리하고, 개발자는 Dart로 고수준 API만 건드리게 설계했습니다.

```dart
// Dart 코드 예시 (추정)
class PlayerController extends Component {
  void update(double dt) {
    // 여기서 C++로 최적화된 물리 엔진이나 트랜스폼을 호출
    entity.transform.position += velocity * dt;
  }
}
```

이 구조는 Unity(C++ 엔진 + C# 스크립트)나 Godot과 유사합니다. Flutter의 `Hot Reload` 기능을 게임 개발에 그대로 쓸 수 있다는 건 생산성 측면에서 엄청난 강점입니다. 코드를 수정하고 저장하면 게임 상태를 유지한 채로 로직만 바뀐다? 게임 개발 해보신 분들은 이게 얼마나 큰 축복인지 아실 겁니다.

## 2. 렌더링: Google Filament의 힘

"Console-grade"라는 마케팅 용어의 근거는 **Google Filament** 렌더러에 있습니다. Filament는 안드로이드 진영에서 이미 검증된 PBR(Physically Based Rendering) 엔진입니다. Vulkan을 지원하며, 모바일 하드웨어에서도 꽤 괜찮은 때깔을 뽑아줍니다.

자체 렌더러를 바닥부터 짜는 '바퀴의 재발명'을 하지 않고, 검증된 오픈소스를 가져다 Flutter 위젯 트리(`FluoriteView`)에 태운 전략은 영리합니다. 덕분에 조명, 그림자, 포스트 프로세싱 같은 고급 기능을 바로 사용할 수 있습니다.

## 3. 워크플로우: Blender와의 통합

개인적으로 가장 인상 깊었던 기능은 **"Model-defined touch trigger zones"** 입니다.

보통 3D 오브젝트에 클릭 이벤트를 달려면, 코드로 `BoxCollider` 좌표를 잡거나 별도의 에디터 작업을 해야 합니다. Fluorite는 3D 아티스트가 **Blender** 에서 특정 영역을 '클릭 가능'하게 설정해두면, 엔진이 이를 인식해서 이벤트를 뱉어줍니다.

> "Developers can then listen to onClick events with the specified tags to trigger all sorts of interactions!"

이건 개발자와 디자이너 사이의 커뮤니케이션 비용을 획기적으로 줄여줍니다. "거기 좌표 좀 5픽셀만 오른쪽으로 옮겨줘요" 같은 핑퐁을 할 필요가 없다는 거죠.

## 4. 왜 하필 Toyota인가?

Hacker News 댓글을 보면 다들 의아해합니다.

> User anon: "How is this related to Toyota? Toyota the car manufacturer?"

사실 도요타는 **차량용 인포테인먼트 시스템(IVI)** 에 Flutter를 도입하는 데 가장 적극적인 기업 중 하나입니다. 요즘 자동차 계기판이나 센터페시아를 보면 단순한 2D UI가 아니라, 차량 상태를 3D 모델로 보여주거나 화려한 인터랙션을 넣는 추세입니다 (언리얼 엔진이 자동차 업계에 진출한 이유기도 하죠).

도요타 입장에서 Unity나 Unreal은 라이선스 비용도 비싸고, 부팅 속도나 임베디드 리눅스 최적화 면에서 무거웠을 겁니다. 반면 Flutter는 가볍고 UI 그리기에 최적화되어 있죠. 여기에 **"고성능 3D 렌더링"** 만 얹으면 완벽한 차량용 OS 솔루션이 됩니다. 즉, Fluorite는 게임을 만들기 위해 태어났다기보다, **"게임 같은 UI"를 만들기 위해 태어난 엔진** 일 가능성이 높습니다.

## 5. 결론: 찍먹해볼 만한가?

솔직한 제 의견은 다음과 같습니다.

*   **본격적인 인디 게임 개발자라면?** 아직은 시기상조입니다. Unity나 Godot의 방대한 에셋 스토어와 커뮤니티를 이기긴 힘듭니다.
*   **앱에 3D 요소를 넣고 싶은 Flutter 개발자라면?** **강력 추천(Must Try)** 입니다. 기존에는 Unity 연동하느라 `MethodChannel` 파이프라인 뚫고 고생했는데, 이건 그냥 Dart 코드 안에서 다 해결됩니다.

아직 초기 단계(2026년 저작권 표기가 있는 걸로 보아 미래지향적 프로젝트인 듯)지만, FOSDEM에서도 발표될 정도면 오픈소스 생태계에 진심인 것 같습니다. 임베디드나 키오스크, 혹은 화려한 인터랙티브 앱을 만드는 분들이라면 이 프로젝트를 주시해야 합니다.

이건 단순한 게임 엔진이 아니라, **Flutter가 '앱 프레임워크'를 넘어 '인터랙티브 미디어 엔진'으로 진화하는 신호탄** 일지도 모릅니다.

---

**References:**
- [Fluorite Official Site](https://fluorite.game/)
- [Hacker News Discussion](https://news.ycombinator.com/item?id=46976911)
- [FOSDEM 2026 Talk](https://fosdem.org/2026/schedule/event/7ZJJWW-fluorite-game-...)
