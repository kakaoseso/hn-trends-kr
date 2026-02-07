---
title: "LLM이 작성한 코드를 Docker 없이 실행한다고? Pydantic Monty 딥다이브"
description: "LLM이 작성한 코드를 Docker 없이 실행한다고? Pydantic Monty 딥다이브"
pubDate: "2026-02-07T14:42:25Z"
---

최근 LLM 기반의 Agent 시스템을 설계해 본 엔지니어라면 누구나 마주하는 딜레마가 있습니다. 바로 **Code Execution(코드 실행)** 샌드박싱 문제입니다.

LLM에게 Python 코드를 짜게 하고 그걸 실행시키면 복잡한 로직을 훨씬 효율적으로 처리할 수 있다는 건 다들 압니다. 하지만 그 코드를 어디서 실행할까요? `exec()`를 쓰자니 보안이 걱정되고, Docker 컨테이너를 매번 띄우자니 Latency와 리소스 오버헤드가 감당이 안 됩니다. Firecracker 같은 MicroVM을 도입하자니 인프라 복잡도가 급상승하죠.

그런데 Pydantic 팀이 또 흥미로운 물건을 들고 나왔습니다. 이름은 **Monty**. Rust로 작성된, AI를 위한 초경량 Python 인터프리터입니다.

이 프로젝트가 단순한 장난감인지, 아니면 차세대 Agent 프레임워크의 핵심이 될지, 시니어 엔지니어의 관점에서 뜯어봤습니다.

## Monty가 도대체 뭔가?

한마디로 정의하면 **"CPython 의존성 없이 Rust로 처음부터 다시 짠 임베디드 Python 인터프리터"** 입니다.

기존의 CPython은 무겁습니다. 그리고 샌드박싱을 염두에 두고 설계되지 않았죠. Monty는 이 문제를 정면으로 돌파합니다.

### 핵심 스펙

- **Startup Time:** 1µs 미만 (Docker가 수백 ms 걸리는 것과 비교 불가)
- **Security:** 파일시스템, 네트워크 접근 원천 차단 (개발자가 허용한 함수만 호출 가능)
- **Snapshotting:** 실행 상태를 바이트로 직렬화하여 저장하고, 나중에 다른 프로세스에서 재개 가능
- **Compatibility:** Python, Rust, JavaScript 어디서든 호출 가능

README에 있는 예시를 보면 컨셉이 명확합니다.

```python
import pydantic_monty

code = """
data = fetch(url)
len(data)
"""

# fetch 함수만 외부에서 주입, 나머지는 격리
m = pydantic_monty.Monty(code, inputs=['url'], external_functions=['fetch'])

# 실행 시작 -> fetch 호출 시점에서 일시 정지
result = m.start(inputs={'url': 'https://example.com'})

# 실제 fetch는 호스트(Python/Rust)에서 안전하게 수행 후 결과 주입
result = result.resume(return_value='hello world')
print(result.output) # 11
```

## 왜 주목해야 하는가? (Engineering Perspective)

단순히 "빠르다"는 것보다 더 중요한 아키텍처적 함의가 있습니다.

### 1. 'Code Mode'의 대중화
Cloudflare나 Anthropic이 밀고 있는 **Code Mode** 는 LLM이 도구를 하나씩 호출하는 게 아니라, Python 스크립트를 짜서 한 번에 로직을 처리하는 방식입니다. 이렇게 하면 LLM과의 Round-trip 횟수를 획기적으로 줄일 수 있습니다. Monty는 이 패턴을 서버리스 환경이나 엣지 디바이스에서도 가볍게 돌릴 수 있게 해줍니다.

### 2. 진정한 의미의 'Stateful Serverless'
제가 가장 감탄한 부분은 **Snapshotting** 기능입니다. `dump()`와 `load()`를 통해 인터프리터의 메모리 상태를 그대로 얼렸다가 녹일 수 있습니다.

이게 왜 중요하냐면, 긴 시간이 걸리는 작업을 비동기로 처리할 때 유용하기 때문입니다. LLM이 짠 코드가 실행되다가 사용자 입력을 기다려야 한다면? 상태를 DB에 저장해두고 워커 프로세스를 죽인 뒤, 입력이 들어왔을 때 다른 서버에서 상태를 복원해 이어서 실행할 수 있습니다. Temporal 같은 Durable Execution 워크플로우를 Python 인터프리터 레벨에서 구현한 셈입니다.

## 한계점: 아직은 'Experimental'

물론 은탄환은 아닙니다. 현재 Monty는 명확한 한계가 있습니다.

- **제한된 기능:** 아직 Class 정의도 안 됩니다. (곧 추가된다고는 합니다)
- **라이브러리 부재:** Pandas, Numpy 같은 C 확장 모듈을 맘대로 `import` 할 수 없습니다. 순수 알고리즘 로직이나 데이터 가공 정도만 가능합니다.
- **생태계:** CPython의 방대한 생태계를 포기해야 합니다.

## Hacker News의 반응과 내 생각

[Hacker News 스레드](https://news.ycombinator.com/item?id=46918254)에서도 뜨거운 논쟁이 있었습니다. 인상 깊었던 의견들을 제 생각과 함께 정리해 봅니다.

### "AI가 생각하는 게 느린데 실행 속도가 무슨 상관?"
한 유저가 "배달 기사가 빙하(Glacier)처럼 느리게 걷는데, 자전거가 빠르다고 배달이 빨라지냐"며 LLM의 생성 속도가 병목이라고 지적했습니다.

**제 생각:** 반은 맞고 반은 틀립니다. 단발성 실행이라면 맞지만, Agent가 루프를 돌며 수십 번 코드를 실행하는 시나리오라면 200ms(Docker)와 0.001ms(Monty)의 차이는 UX에 큰 영향을 줍니다. 게다가 **Cold Start** 비용이 없다는 건 서버리스 아키텍처에서 비용 절감과 직결됩니다.

### "왜 JS/V8을 안 쓰고?"
"V8(JavaScript)은 이미 샌드박싱이 완벽한데 왜 굳이 Python 인터프리터를 새로 만드냐"는 의견도 많았습니다.

**제 생각:** 기술적으로는 JS가 샌드박싱에 더 유리한 게 사실입니다. 하지만 **Data Science와 AI의 Lingua Franca는 Python** 입니다. LLM 학습 데이터의 대다수가 Python이고, LLM이 가장 잘 짜는 코드도 Python입니다. 엔지니어링 효율성을 위해 언어적 정합성을 맞추는 시도는 타당해 보입니다.

## 결론: Production Ready인가?

아직은 **아니요(No)** 입니다. Pydantic 팀도 "Not ready for prime time"이라고 명시했습니다. Class 지원조차 없는 상태에서 복잡한 비즈니스 로직을 태우기는 무리입니다.

하지만 **방향성은 매우 정확합니다.**

1.  **Agentic Workflow** 가 대세가 되고 있고,
2.  안전하고 가벼운 **Code Execution Environment** 에 대한 갈증이 있으며,
3.  **Rust + Python** 의 결합이 성능과 생산성을 모두 잡는 트렌드를 보여줍니다.

만약 여러분이 사내에서 "LLM이 짠 코드를 안전하게 실행할 방법"을 고민하고 있다면, 무거운 Docker나 K8s Job을 띄우기 전에 Monty의 발전 과정을 주시해 볼 필요가 있습니다. 특히 Pydantic AI와 결합되었을 때 어떤 시너지가 날지 매우 기대됩니다.

가벼운 텍스트 처리나 제어 흐름(Control Flow) 로직을 LLM에게 위임하는 용도라면, 지금 당장 토이 프로젝트로 찍먹해보기엔 충분히 매력적인 도구입니다.
