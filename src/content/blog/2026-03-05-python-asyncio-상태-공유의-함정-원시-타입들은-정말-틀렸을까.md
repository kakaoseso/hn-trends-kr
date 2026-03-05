---
title: "Python asyncio 상태 공유의 함정: 원시 타입들은 정말 틀렸을까"
description: "Python asyncio 상태 공유의 함정: 원시 타입들은 정말 틀렸을까"
pubDate: "2026-03-05T04:11:22Z"
---



비동기 프로그래밍을 처음 접하는 주니어 엔지니어들이 흔히 하는 착각이 하나 있습니다. 바로 "`asyncio`는 싱글 스레드 기반이니까 동시성 문제(Concurrency hazards)나 Race condition에서 자유롭겠지?"라는 생각입니다. 

천만의 말씀입니다. 코루틴은 매 `await` 지점마다 실행을 교차(interleave)합니다. OS 스레드 레벨의 컨텍스트 스위칭이 협력적 양보(cooperative yield) 지점으로 이동했을 뿐, 공유된 가변 상태(Shared mutable state)가 오염되는 문제는 멀티스레딩 환경만큼이나 쉽게 발생합니다.

최근 Inngest 팀에서 [What Python's asyncio primitives get wrong about shared state](https://www.inngest.com/blog/no-lost-updates-python-asyncio)라는 흥미로운 아티클을 발행했습니다. Python SDK를 개발하며 WebSocket 연결 상태를 동시성 환경에서 제어할 때 겪은 문제와 해결책을 다루고 있죠. 

하지만 이 아티클의 도발적인 제목과 달리, 저는 이 문제의 본질이 "Python의 원시 타입(primitives)이 틀렸다"기보다는 **"설계 요구사항에 맞지 않는 도구를 선택했을 때 벌어지는 참사"** 에 가깝다고 봅니다. 오늘 포스팅에서는 이들이 겪은 'Lost Update' 문제가 무엇인지 기술적으로 파고들어 보고, 왜 결국 큐(Queue) 기반의 메시지 패싱으로 귀결될 수밖에 없었는지, 그리고 이 아티클이 간과하고 있는 치명적인 위험성에 대해 제 개인적인 시각을 더해 리뷰해 보겠습니다.

---

## 문제의 발단: 상태 변화 기다리기

시스템이 `disconnected` → `connecting` → `connected` → `closing` → `closed` 와 같은 상태 머신을 가진다고 가정해 봅시다. 특정 비동기 핸들러는 연결이 종료되는 시점(`closing`)을 감지하고 대기 중인 요청들을 안전하게 Drain(비우기) 해야 합니다.

가장 단순 무식한 방법은 **Polling** 입니다.

```python
async def drain_requests():
    while state != "closing":
        await asyncio.sleep(0.1)
    print("draining pending requests")
```

당연히 실무에서 이런 코드를 작성하면 PR 리뷰에서 살아남지 못합니다. 짧은 sleep은 CPU 사이클을 낭비하고, 긴 sleep은 Latency를 증가시킵니다. 

그래서 우리는 `asyncio.Event`나 `asyncio.Condition` 같은 표준 라이브러리의 동기화 원시 타입들을 찾게 됩니다.

## asyncio.Condition과 'Lost Update'의 늪

`asyncio.Event`는 boolean 상태만 가지므로 여러 상태를 표현하기엔 Boilerplate가 너무 많이 생깁니다. 자연스럽게 임의의 조건식을 평가할 수 있는 `asyncio.Condition`으로 눈을 돌리게 되죠.

```python
async def drain_requests():
    async with condition:
        await condition.wait_for(lambda: state == "closing")
    print("draining pending requests")
```

완벽해 보입니다. 상태가 변경될 때마다 `notify_all()`을 호출하면, 대기 중인 코루틴이 깨어나 조건을 재평가합니다. 하지만 여기에 **치명적인 함정** 이 숨어 있습니다. 바로 상태가 너무 빨리 변할 때 발생하는 **Lost Update (상태 누락)** 현상입니다.

이 현상은 Event loop tick의 동작 방식 때문에 발생합니다. 상태 변경자가 `notify_all()`을 호출하면, 대기 중인 코루틴들의 'Wakeup'이 스케줄링됩니다. 하지만 싱글 스레드 이벤트 루프에서는 **현재 실행 중인 코루틴이 제어권을 양보(`await`)하기 전까지는 깨어난 코루틴이 실제로 실행되지 않습니다.**

만약 제어권이 넘어가기 전에 상태가 `closing`에서 `closed`로 한 번 더 바뀌어 버린다면 어떻게 될까요?

```python
await set_state("closing") # Wakeup 스케줄링됨
await set_state("closed")  # 코루틴이 실행되기 전에 상태가 또 바뀜

# drain_requests가 드디어 실행되어 조건을 평가함.
# state는 이미 "closed"이므로 lambda: state == "closing" 은 False가 됨.
# 영원히 대기 상태에 빠짐.
```

이것은 전형적인 TOCTOU (Time of Check to Time of Use) 버그의 변형입니다. `drain_requests`는 찰나의 `closing` 상태를 보지 못하고 영원히 잠들어버리며, In-flight 요청들은 허공으로 증발합니다.

## 해결책: 상태를 묻지 말고, 변화를 버퍼링하라

Inngest 팀은 이 문제를 해결하기 위해 상태를 덮어쓰는 대신, 모든 상태 변화를 각 컨슈머별 `asyncio.Queue`에 `(old, new)` 튜플 형태로 밀어 넣는 `ValueWatcher` 패턴을 구현했습니다.

```python
class ValueWatcher:
    # ... 초기화 코드 생략 ...
    @value.setter
    def value(self, new_value):
        old_value = self._value
        self._value = new_value
        for queue in self._watch_queues:
            queue.put_nowait((old_value, new_value))
```

이제 컨슈머는 이벤트 루프 타이밍에 의존하여 '현재 상태'를 폴링하는 것이 아니라, 큐에 쌓인 상태 변화의 '스트림'을 순차적으로 소비합니다. 상태를 절대 놓치지 않게 되는 것이죠.

---

## Principal Engineer의 시선: 무엇이 문제인가?

솔직히 말해서, 이 패턴은 꽤 유용하고 실무에서도 자주 쓰이는 방식입니다. 하지만 이 아티클과 코드를 프로덕션에 그대로 복사해 넣기 전에 몇 가지 짚고 넘어가야 할 비판점이 있습니다. Hacker News 커뮤니티에서도 비슷한 지적들이 쏟아졌죠.

### 1. Primitives는 틀리지 않았다. 요구사항이 바뀌었을 뿐이다.

글의 제목은 마치 `asyncio` 설계자들이 바보라서 무언가를 잘못 만든 것처럼 묘사합니다. 하지만 글을 읽다 보면 은근슬쩍 새로운 요구사항이 추가됩니다.

> "모든 상태 전환을 컨슈머별 큐에 버퍼링해야 한다. 컨슈머는 어떤 상태도 놓쳐서는 안 된다."

이건 동기화(Synchronization)의 문제가 아니라 **이벤트 소싱(Event Sourcing) 혹은 메시지 패싱(Message Passing)** 의 문제입니다. 만약 모든 상태 변화의 히스토리를 순차적으로 처리해야 한다면, 처음부터 Queue를 써야 하는 것이 당연합니다. Event나 Condition은 '엣지 트리거(Edge-triggered)' 방식으로 현재의 스냅샷 상태를 기반으로 흐름을 제어하기 위해 만들어진 도구이지, 이벤트 스트림을 처리하기 위한 도구가 아닙니다.

### 2. Unbounded Queue의 공포 (메모리 누수)

제가 이 코드를 코드 리뷰에서 보았다면 가장 먼저 반려(Reject)했을 포인트입니다. 

- **문제점:** 코드에서 `queue = asyncio.Queue()`를 생성할 때 최대 크기(`maxsize`)를 지정하지 않았습니다.

만약 상태를 생산하는 속도(Producer)가 컨슈머가 이벤트를 처리하는 속도보다 빠르다면 어떻게 될까요? 큐는 무한정 커지고 결국 OOM(Out of Memory) 크래시를 유발합니다. 특히 네트워크 상태 변경처럼 외부 요인에 의해 이벤트 스파이크가 발생할 수 있는 시스템에서 Unbounded Queue는 시한폭탄과 같습니다. 반드시 `maxsize`를 설정하고, 큐가 가득 찼을 때 이벤트를 버릴지(Drop), 아니면 생산자를 블로킹할지(Backpressure) 결정하는 로직이 필요합니다.

### 3. asyncio.Lock vs threading.Lock

저자는 `ValueWatcher` 내부 상태를 보호하기 위해 `threading.Lock`을 사용했습니다. 비동기 코드에서 동기 락을 사용하는 것은 일반적으로 안티 패턴입니다. 

저자의 변명은 "다른 스레드에서 `loop.call_soon_threadsafe`를 통해 값을 세팅할 수 있도록 하기 위해서"입니다. 즉, 이 클래스는 순수 `asyncio` 컴포넌트라기보다는 멀티스레드와 비동기 루프 간의 브릿지 역할을 겸하고 있습니다. 만약 여러분의 애플리케이션이 단일 이벤트 루프 내에서만 동작한다면, 이런 오버헤드를 감수할 필요 없이 순수하게 `asyncio` 원시 타입들로만 메시지 패싱 루프를 구성하는 것이 훨씬 깔끔합니다.

### 4. Erlang이 이미 30년 전에 푼 문제

이 모든 구조를 보고 있으면 자연스럽게 Erlang의 Actor 모델과 `gen_server`의 Mailbox가 떠오릅니다. 비동기 환경에서 공유 가변 상태를 안전하게 다루는 가장 확실한 방법은, **상태를 공유하지 않는 것** 입니다. 상태를 소유한 단일 Actor(또는 Task)를 띄워두고, 다른 코루틴들은 Queue를 통해 메시지만 던지는 구조(Message-passing)로 설계했다면, 굳이 이런 복잡한 `ValueWatcher` 클래스를 만들지 않아도 TOCTOU 문제를 원천 차단할 수 있었을 것입니다.

---

## Verdict: 그래서 이 코드를 써도 될까?

이 아티클은 Python `asyncio` 환경에서 코루틴 스케줄링의 미묘한 타이밍 이슈(Lost Update)를 아주 훌륭하게 시각화하고 설명했습니다. 이 부분만으로도 비동기 프로그래밍을 다루는 엔지니어라면 한 번쯤 정독할 가치가 있습니다.

하지만 `ValueWatcher` 코드를 그대로 가져다 쓰는 것은 추천하지 않습니다. 앞서 언급한 메모리 누수 위험(Unbounded Queue)이 존재하며, 대부분의 경우 특정 상태를 모니터링하기 위해 이렇게 무거운 제네릭 옵저버 패턴을 도입하기보다는, 아키텍처 자체를 메시지 패싱 기반으로 리팩토링하는 것이 장기적인 유지보수 관점에서 훨씬 유리합니다.

`asyncio`는 마법이 아닙니다. 스레드가 하나라고 해서 상태 관리의 복잡성이 사라지는 것은 아니며, 동시성 프로그래밍의 기본 원칙들은 비동기 환경에서도 여전히, 아니 오히려 더 엄격하게 적용되어야 합니다.

---

**References:**
- **Original Article:** [What Python's asyncio primitives get wrong about shared state](https://www.inngest.com/blog/no-lost-updates-python-asyncio)
- **Hacker News Thread:** [https://news.ycombinator.com/item?id=47256923](https://news.ycombinator.com/item?id=47256923)
