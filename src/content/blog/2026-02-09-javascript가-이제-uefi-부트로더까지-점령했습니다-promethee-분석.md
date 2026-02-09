---
title: "JavaScript가 이제 UEFI 부트로더까지 점령했습니다 (Promethee 분석)"
description: "JavaScript가 이제 UEFI 부트로더까지 점령했습니다 (Promethee 분석)"
pubDate: "2026-02-09T17:09:22Z"
---

솔직히 말하겠습니다. 처음 이 프로젝트를 봤을 때 저는 2014년 Gary Bernhardt의 전설적인 발표, **"The Birth and Death of JavaScript"** 가 떠올랐습니다. 그 영상에서 그는 농담 반 진담 반으로 "모든 것이 JS로 재작성되는 미래(Metal stage)"를 예언했었죠. 그리고 오늘, 우리는 그 예언이 현실이 되는 순간을 목격하고 있습니다.

바로 **Promethee** 라는 프로젝트입니다. UEFI 환경에서 JavaScript를 실행할 수 있게 해주는 바인딩이죠.

단순히 "JS가 어디서든 돈다"는 밈(meme)으로 치부하기엔, 이 프로젝트가 가진 기술적 접근 방식이 꽤 흥미롭습니다. 15년 차 엔지니어 입장에서 이 "끔찍하지만 천재적인" 프로젝트를 뜯어보았습니다.

## 이게 도대체 뭔가요?

Promethee는 아주 간단한 컨셉입니다. 컴퓨터가 부팅될 때(UEFI 단계), C로 작성된 로더가 JavaScript 엔진을 초기화하고 `script.js` 파일을 읽어 실행합니다. 즉, 여러분의 부트로더 로직을 JavaScript로 짤 수 있다는 뜻입니다.

Node.js를 통째로 부트 파티션에 넣은 것이 아닙니다. 그랬다면 바이너리 크기 때문에라도 불가능했을 겁니다. 핵심은 **Duktape** 라는 임베디드용 경량 JS 엔진을 사용했다는 점입니다.

## 기술적 Deep Dive: 왜 V8이 아니라 Duktape인가?

Hacker News의 한 유저(anon)가 정확히 지적했듯이, 이 선택은 매우 영리했습니다.

- **V8/SpiderMonkey:** JIT 컴파일러와 복잡한 의존성 때문에 OS가 없는 Freestanding 환경(Bare-metal)에서 구동하기엔 악몽에 가깝습니다.
- **Duktape:** 애초에 임베디드 시스템을 위해 설계되었고, 최소한의 libc stub만 있으면 어디서든 돌아갑니다. 스택 기반의 C API를 제공해서 C 코드와 JS를 연결(Binding)하기가 매우 수월합니다.

이 프로젝트의 구조를 보면 C 코드가 약 70%를 차지하는데, 대부분이 UEFI의 시스템 테이블(System Table)과 프로토콜을 JS 객체로 매핑해주는 접착제 코드(Glue code)입니다.

## 실제 코드는 어떻게 생겼나?

가장 인상적인 부분은 하드웨어 추상화 계층(HAL)을 억지로 만들지 않았다는 점입니다. 대신 UEFI의 **Raw Protocol** 을 거의 그대로 JS에 노출시켰습니다. 아래는 화면에 빨간 사각형을 그리는 예제입니다.

```javascript
// UEFI의 Graphics Output Protocol(GOP)을 찾습니다.
var gop = efi.SystemTable.BootServices.LocateProtocol(efi.guid.GraphicsOutput);

if (gop) {
    var red = { r: 255, g: 0, b: 0 };
    // Blt(Block Transfer) 함수를 직접 호출하여 메모리에 픽셀을 씁니다.
    gop.Blt(red, 'EfiBltVideoFill', 0, 0, 50, 50, 200, 120, 0);
}
```

보시다시피 `efi.SystemTable`이나 `LocateProtocol` 같은 UEFI 스펙의 용어를 그대로 사용합니다. 이는 C로 UEFI 애플리케이션을 짜본 사람이라면 아주 익숙한 패턴입니다. 차이점이라면, 포인터 연산이나 메모리 할당의 고통 없이 JS의 유연함을 누릴 수 있다는 것이죠.

## Hacker News의 반응: "Cursed" vs "Genius"

커뮤니티의 반응은 예상대로 뜨겁습니다. "경이롭다"는 반응과 "저주받은 물건(Cursed)"이라는 반응이 공존합니다.

- **OS 개발 가능성:** 누군가는 "이제 JS로 OS 커널을 만들 수 있냐"고 묻더군요. 이론적으로는 부트로더 단계에서 메모리 맵을 제어하고 하드웨어를 초기화할 수 있으니, 아주 원시적인 형태의 OS 로직을 구현할 수는 있습니다. 하지만 GC(Garbage Collection)가 커널 레벨에서 돌 때 발생할 'Stop-the-world' 현상은 시스템 안정성에 치명적일 겁니다.
- **네트워킹:** 어떤 유저는 "이제 부트로더에서 `npm install left-pad`를 할 수 있는 거냐"며 비꼬기도 했지만, 실제로 UEFI에는 네트워크 스택이 포함되어 있습니다. JS로 PXE 부팅 스크립트를 짠다면 꽤 유용할 수도 있습니다.
- **CSS로 스플래시 스크린:** "드디어 순수 CSS로 부팅 애니메이션을 만들 수 있다"는 농담은 꽤 그럴듯하게 들립니다.

## Principal Engineer의 시선: 실용성이 있을까?

냉정하게 평가하자면, 이 프로젝트를 프로덕션 레벨의 펌웨어에 당장 도입할 일은 없을 겁니다. 보안(Secure Boot 서명 문제)이나 성능, 그리고 디버깅의 어려움 때문입니다.

하지만 **Prototyping 도구** 로서는 엄청난 잠재력이 있습니다.

1.  **진입 장벽 완화:** UEFI 개발은 설정이 까다롭기로 유명합니다. EDK2 빌드 환경을 세팅하다가 포기하는 경우가 다반사죠. 하지만 이 프로젝트는 `script.js`만 수정해서 바로 QEMU로 띄워볼 수 있습니다.
2.  **진단 도구:** 하드웨어 진단 로직을 굳이 C로 컴파일해서 펌웨어에 굽는 대신, 외부 스토리지에 있는 JS 스크립트를 로드해서 실행하는 방식은 유연성 면에서 큰 장점입니다.

## 결론

Promethee는 기술적 유희(Toy Project)에 가깝지만, 우리가 "당연히 C로 해야 한다"고 믿었던 영역(Bare-metal)에 균열을 내는 흥미로운 시도입니다. 만약 여러분이 UEFI 프로그래밍에 관심이 있었지만 C 언어의 장벽 때문에 망설였다면, 이 프로젝트는 아주 훌륭한 입문서가 될 것입니다.

물론, 부트로더에 `is-odd` 패키지를 import 하는 날이 오지 않기만을 바랄 뿐입니다.

---

**References:**
- **Original Article:** https://codeberg.org/smnx/promethee
- **Hacker News Thread:** https://news.ycombinator.com/item?id=46945348
