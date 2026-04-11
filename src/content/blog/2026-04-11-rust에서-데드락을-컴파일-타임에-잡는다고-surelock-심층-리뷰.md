---
title: "Rust에서 데드락을 컴파일 타임에 잡는다고? Surelock 심층 리뷰"
description: "Rust에서 데드락을 컴파일 타임에 잡는다고? Surelock 심층 리뷰"
pubDate: "2026-04-11T19:39:55Z"
---

데드락(Deadlock)만큼 엔지니어를 피 말리게 하는 버그도 없습니다. 코드 리뷰에서는 완벽하게 숨어 있고, CI 테스트도 수천 번 가뿐히 통과하죠. 그러다 아무도 예상치 못한 트래픽 패턴이 몰리는 새벽 3시에 시스템 전체를 멈춰버립니다.

Rust는 데이터 레이스(Data race)를 컴파일 타임에 잡아내는 것으로 유명합니다. 하지만 데드락은 어떨까요? 컴파일러는 그저 `Mutex` 하나 쥐여주고 "행운을 빈다"며 등을 떠밀 뿐입니다. 물론 Actor 모델이나 Lock-free 프로그래밍을 지향해야 한다는 원론적인 이야기는 많지만, 현실의 Rust 코드베이스에서 Mutex는 숨 쉬듯 쓰입니다.

최근 Hacker News에서 제 눈길을 끈 프로젝트가 하나 있습니다. 바로 컴파일 타임에 데드락을 원천 차단하겠다는 목표로 만들어진 [surelock](https://crates.io/crates/surelock)이라는 크레이트입니다. 15년 넘게 백엔드와 분산 시스템을 굴려보며 온갖 동시성 이슈에 데여본 입장에서, 이 라이브러리가 취한 접근 방식과 한계를 깊게 파헤쳐 보려고 합니다.

## 데드락의 근본 원인과 Surelock의 접근법

1971년 발표된 Coffman 조건에 따르면, 데드락이 발생하려면 4가지 조건(상호 배제, 점유 대기, 비선점, 순환 대기)이 동시에 충족되어야 합니다. Mutex의 존재 이유가 상호 배제(Mutual exclusion)이고, 선점형 락은 또 다른 버그의 온상이 되기 쉬우니 우리가 깰 수 있는 건 결국 **순환 대기(Circular wait)** 뿐입니다.

Surelock은 이 순환 대기를 끊기 위해 두 가지 상호 보완적인 메커니즘을 사용합니다. 그리고 이 모든 것을 가능하게 하는 핵심 메타포가 바로 `MutexKey`입니다.

### 1. MutexKey: 락의 상태를 증명하는 Linear Token

이 라이브러리의 가장 영리한 부분입니다. 락을 획득하려면 `MutexKey`라는 토큰이 필요합니다. `.lock()`을 호출하면 기존 키가 소비(consume)되고, 현재 락 상태가 타입 레벨에 기록된 새로운 키가 반환됩니다.

이 구조는 Rust의 소유권(Ownership) 모델과 완벽하게 맞아떨어집니다. 글로벌 분석이나 런타임 오버헤드 없이, 단순히 키를 스레드 로컬에 가두고 타입 체커가 순서를 강제하도록 만드는 것이죠.

### 2. LockSet: 동일 레벨의 원자적 락 획득

여러 개의 락을 한 번에 잡아야 할 때, Surelock은 `LockSet`을 통해 런타임에 락을 정렬합니다. 각 Mutex는 생성 시점에 고유한 `LockId`를 부여받습니다.

```rust
let alice = Mutex::new("alice");
let bob = Mutex::new("bob");

// 인자 순서와 무관하게 결정론적(Deterministic)으로 정렬됨
let set = LockSet::new((&alice, &bob));
key_handle.scope(|key| {
    let (a, b) = set.lock(key);
    // 둘 다 안전하게 락 획득 완료
});
```

이 부분에 대해 HN 코멘트에서는 꽤 흥미로운 논쟁이 있었습니다. 기존의 `happylock` 라이브러리처럼 메모리 주소값을 기준으로 정렬하면 되지 않냐는 의견이었죠. 누군가는 "Mutex가 이동(move)되면 어차피 락을 걸 수 없으니 주소값 정렬도 안전하다"고 주장했습니다. 

하지만 제 생각은 다릅니다. Rust에서 메모리 주소 안정성을 보장하려면 필연적으로 `unsafe` (예: `Box::leak`, `transmute`)가 전염됩니다. 저자는 이 문제를 피하기 위해 단순하지만 확실한 `AtomicU64` 기반의 ID 부여 방식을 택했습니다. 라이브러리 설계 관점에서 훨씬 우아하고 안전한 선택입니다.

### 3. Level<N>: 컴파일 타임 순서 강제

서로 다른 종류의 리소스(예: 전역 Config와 개별 Account)를 잠글 때는 `Level<N>` 타입과 `LockAfter` 트레잇을 사용합니다.

```rust
use surelock::level::{Level, lock_scope};

type Config = Level<1>;
type Account = Level<2>;

let config: Mutex<_, Config> = Mutex::new(AppConfig::default());
let account: Mutex<_, Account> = Mutex::new(Balance(0));

lock_scope(|key| {
    let (cfg, key) = config.lock(key);   // Level 1 -> 통과
    let (acct, key) = account.lock(key); // Level 2 -> 통과
    // 여기서 다시 config.lock(key)를 호출하면 컴파일 에러 발생!
});
```

Google Fuchsia 팀의 `lock_tree`가 DAG(방향 비순환 그래프)를 사용한 것과 달리, Surelock은 **엄격한 전체 순서(Total Order)** 를 강제합니다. DAG는 유연해 보이지만 두 개의 독립적인 브랜치를 서로 다른 스레드가 역순으로 획득하면 여전히 데드락이 발생할 수 있습니다. 

HN의 한 유저가 지적했듯, 이는 데이터베이스 엔진 설계에서 자주 쓰이는 패턴입니다. DB 세계의 엄격한 동시성 제어 기법이 일반 애플리케이션 레벨의 라이브러리로 넘어온 아주 좋은 사례라고 봅니다.

## 현실적인 한계: Async와 인지적 과부하

그렇다면 당장 내일 회사 코드베이스에 Surelock을 도입해야 할까요? 안타깝게도 몇 가지 치명적인 허들이 있습니다.

- **Async 생태계와의 충돌**: HN 코멘트에서 가장 날카롭게 지적된 부분입니다. `MutexKey`와 `MutexGuard`는 `!Send`입니다. 즉, `.await` 포인트를 가로질러 락을 유지할 수 없습니다. Tokio 같은 런타임에서는 Future가 `Send`여야 하는 경우가 많은데, Surelock을 쓰면 이 제약에 정면으로 부딪힙니다. 비동기 웹 백엔드에서 쓰기에는 아직 무리가 있습니다.
- **인지적 부담 (Cognitive Load)**: 락의 레벨을 설계하고 관리하는 것은 꽤 피곤한 일입니다. 시스템이 커질수록 "이 락이 레벨 3인가 4인가?"를 고민해야 합니다. 잘못 설계하면 불필요하게 상위 락을 오래 잡고 있어 성능 저하(Priority Inversion)를 유발할 수 있습니다.

## 총평: 누구를 위한 기술인가?

이 라이브러리는 모든 프로젝트를 위한 은탄환은 아닙니다. 일반적인 CRUD 웹 서버라면 차라리 락의 범위를 최소화하고 아키텍처를 단순하게 가져가는 것이 낫습니다.

하지만 **임베디드(no_std)** 환경이나, 디버거를 붙일 수 없는 **베어메탈 펌웨어**, 혹은 고도로 복잡한 **동기식 코어 엔진** 을 개발 중이라면 이야기가 다릅니다. 이런 환경에서는 데드락이 곧 치명적인 장애로 이어지기 때문에, 컴파일 타임에 순서를 강제하는 Surelock의 철학이 빛을 발할 것입니다.

데이터베이스 세계의 지혜를 Rust의 타입 시스템으로 우아하게 녹여낸 저자의 통찰력에 박수를 보냅니다. 당장 프로덕션의 비동기 서버에 넣진 못하더라도, 동시성을 다루는 엔지니어라면 이 라이브러리의 설계 철학만큼은 반드시 곱씹어볼 가치가 있습니다.

---
**참고 링크**
- 원문: [Surelock: Deadlock-Free Mutexes for Rust](https://notes.brooklynzelenka.com/Blog/Surelock)
- Hacker News 토론: [https://news.ycombinator.com/item?id=47693559](https://news.ycombinator.com/item?id=47693559)
