---
title: "브라우저 샌드박스의 허상: OPFS를 이용한 SSD 타이밍 사이드 채널 공격(FROST) 분석"
description: "브라우저 샌드박스의 허상: OPFS를 이용한 SSD 타이밍 사이드 채널 공격(FROST) 분석"
pubDate: "2026-05-31T23:26:27Z"
---

최근 프론트엔드 생태계를 보면 브라우저가 단순한 문서 뷰어를 넘어 거대한 운영체제가 되어가고 있다는 것을 실감한다. WebAssembly, WebGL, 그리고 최근의 OPFS(Origin Private File System)까지. 하지만 기능이 강력해질수록 공격 표면(Attack Surface)은 필연적으로 넓어진다.

최근 Ars Technica에 흥미로운(동시에 섬뜩한) 기사가 하나 올라왔다. 웹사이트가 방문자의 SSD I/O 타이밍을 분석해 현재 실행 중인 다른 앱이나 탭을 알아내는 FROST라는 기법에 대한 내용이다. 해커뉴스(Hacker News)에서도 꽤 뜨거운 논쟁이 있었는데, 15년 차 엔지니어의 시각에서 이 기술이 왜 흥미로운지, 그리고 우리가 왜 브라우저 샌드박스를 맹신하면 안 되는지 파헤쳐보자.

## 브라우저에 파일 시스템이 왜 필요해졌나?

전통적으로 웹은 LocalStorage나 IndexedDB를 사용해왔다. 하지만 WASM이 도입되고 브라우저 위에서 SQLite 같은 무거운 C/C++ 라이브러리를 돌리려다 보니, 기존의 Key-Value 스토어로는 성능(Latency와 Throughput)을 감당할 수 없었다.

그래서 등장한 것이 **OPFS** 다. 이는 브라우저 샌드박스 내부에 격리된, 바이트 단위의 랜덤 액세스가 가능한 진짜 파일 시스템 API다. 문제는 여기서 시작된다. 사용자의 명시적인 권한 승인 없이도 기가바이트 단위의 스토리지를 할당받고 디스크에 직접 I/O를 발생시킬 수 있다는 점이다.

## FROST: SSD 경합을 이용한 사이드 채널

FROST(Fingerprinting Remotely using OPFS-based SSD Timing)는 전형적인 사이드 채널 공격이다. 그중에서도 자원을 두고 경쟁할 때 발생하는 지연을 측정하는 **Contention Side Channel** 기법을 사용한다.

원리는 이렇다.
1. 공격자가 심어둔 자바스크립트가 OPFS를 통해 SSD에 더미 파일을 생성하고 지속적으로 랜덤 읽기/쓰기를 수행한다.
2. 동시에 사용자가 띄워둔 다른 탭(예: 백그라운드에서 메일을 폴링하는 Gmail, 채팅을 동기화하는 Reddit)이나 로컬 앱이 자신만의 SSD I/O를 발생시킨다.
3. SSD 컨트롤러의 큐(Queue)와 리소스는 한정되어 있으므로, I/O 경합이 발생하면서 미세한 지연이 생긴다.
4. 공격자는 API를 통해 이 지연 패턴을 수집하고, 사전에 학습된 머신러닝(CNN) 모델에 넣어 현재 어떤 앱이 I/O를 발생시키고 있는지 추론한다.

```javascript
// 개념적인 공격 코드의 형태
async function measureSSDLatency(fileHandle) {
  const syncAccessHandle = await fileHandle.createSyncAccessHandle();
  const buffer = new DataView(new ArrayBuffer(4096));
  
  const start = performance.now();
  // 동기식(Synchronous) API를 사용하여 정확한 디스크 I/O 시간을 측정
  syncAccessHandle.read(buffer, { at: Math.random() * MAX_OFFSET });
  const end = performance.now();
  
  return end - start; // 이 미세한 지연 시간을 수집하여 패턴화
}
```

## 노이즈 속에서 시그널 찾기: 과연 실용적인가?

해커뉴스 스레드를 보면 나와 비슷한 의문을 품은 엔지니어들이 많다. "요즘 OS 환경이 얼마나 노이지(Noisy)한데 그게 가능해?"

솔직히 처음 논문 요약을 읽었을 때는 나 역시 회의적이었다. 실제 유저의 백그라운드에서는 Docker 데몬이 돌고, Slack 같은 Electron 앱들이 쉴 새 없이 메모리와 디스크를 스래싱(Thrashing)하며, OS 자체의 캐싱 레이어도 존재한다. 이 모든 노이즈를 뚫고 특정 웹사이트가 열려 있는지 100% 정확도로 짚어내는 범용적인 공격 벡터로 쓰기에는 무리가 있다.

하지만 관점을 바꿔보면 이야기가 달라진다. 이 기술을 **브라우저 핑거프린팅** 의 새로운 엔트로피 소스로 사용한다면 어떨까?
정확히 '어떤 앱'을 쓰는지 맞출 필요가 없다. 사용자의 하드웨어 성능, SSD 컨트롤러의 특성, 그리고 평소 백그라운드에 띄워두는 앱들의 조합이 만들어내는 '고유한 I/O 노이즈 패턴' 그 자체가 서드파티 쿠키를 대체할 강력하고 은밀한 식별자가 될 수 있다.

## 샌드박스라는 허상

해커뉴스의 한 유저가 남긴 코멘트가 뼈를 때린다.
> "내 기기에서 돌아가는 샌드박스란 사실 존재하지 않는다. 코드는 결국 같은 하드웨어 위에서 실행되며, 하드웨어를 건드리는 수많은 방법은 결국 익스플로잇된다."

과거 Meltdown이나 Spectre 사태 때도 뼈저리게 느꼈지만, 물리적인 자원(CPU 캐시, RAM, SSD 등)을 공유하는 이상 소프트웨어 레벨의 완벽한 격리란 불가능에 가깝다. 브라우저가 네이티브 앱의 영역을 넘보며 로우레벨 API를 개방할수록, 이러한 하드웨어 레벨의 사이드 채널 공격은 계속 쏟아져 나올 것이다. (예전부터 Firefox 프로필을 램디스크에 올려 쓰던 Arch Linux 유저들이 다시 한번 1승을 챙긴 셈이다.)

## 결론: 그래서 우리는 무엇을 해야 하나?

당장 내일 회사 프로덕트에 긴급 패치를 해야 할 수준의 치명적인 제로데이는 아니다. 구형 HDD를 쓰는 유저라면 너무 느려서 오히려 이 공격이 통하지 않는다는 웃지 못할 아이러니도 있다. 하지만 프라이버시를 중시하는 유저나, 고도의 보안이 필요한 환경에서는 충분히 껄끄러운 문제다.

아마도 조만간 브라우저 벤더(Chrome, Firefox 등)들은 Spectre 때 그랬던 것처럼 OPFS의 I/O 타이밍이나 타이머 API의 해상도를 의도적으로 뭉개는(Fuzzing) 패치를 내놓을 것이다.

엔지니어로서 우리는 웹의 성능을 극대화하기 위해 너무 많은 권한을 브라우저에 쥐여주고 있는 것은 아닌지 고민해 봐야 한다. 가끔은 이 모든 혁신이 진정으로 유저를 위한 것인지, 아니면 그저 벤더들이 '할 수 있으니까' 만들어내는 오버엔지니어링인지 되돌아볼 필요가 있다.

## References
- **Original Article:** https://arstechnica.com/security/2026/05/websites-have-a-new-way-to-spy-on-visitors-analyzing-their-ssd-activity/
- **Hacker News Thread:** https://news.ycombinator.com/item?id=48309492
