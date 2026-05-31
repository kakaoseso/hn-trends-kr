---
title: "리눅스 커널의 숨겨진 무기 rseq: Lock과 Atomic 없이 동시성 문제 해결하기"
description: "리눅스 커널의 숨겨진 무기 rseq: Lock과 Atomic 없이 동시성 문제 해결하기"
pubDate: "2026-05-31T18:28:29Z"
---

최근 96코어, 128코어 ARM(Ampere) 워크스테이션이 시장에 보급되면서, 시스템 프로그래밍의 병목 지점이 완전히 달라지고 있다. 과거에는 디스크 I/O나 네트워크 Latency가 문제였다면, 이제는 수백 개의 코어가 동일한 메모리 캐시라인을 두고 다투는 **Cacheline Bouncing** 현상이 가장 큰 적이다.

이 문제를 해결하기 위해 Linux 4.18부터 조용히 도입된 강력한 무기가 있다. 바로 rseq (Restartable Sequences) 기술이다. tcmalloc, jemalloc, glibc 같은 최상위 시스템 라이브러리들은 이미 이 기술을 사용해 성능을 극적으로 끌어올리고 있다.

## 왜 Mutex와 Atomic으로는 부족한가?

전통적인 멀티스레딩 환경에서 공유 자원을 보호하는 방법은 Mutex를 쓰거나 Atomic 연산을 사용하는 것이다. 하지만 코어 수가 100개를 넘어가면 이야기가 달라진다.

- **Mutex:** 아무리 최적화된 라이브러리를 써도 Uncontended 상태에서 약 15ns, Contended 상태에서는 200ns 이상의 오버헤드가 발생한다.
- **Atomic:** Lock-free 자료구조를 만들기 위해 Compare-And-Swap(CAS)을 사용하지만, 여러 코어가 동일한 64바이트 캐시라인을 갱신하려 들면 CPU 내부 버스에서 병목이 발생해 사실상 Mutex와 다를 바 없는 속도 저하가 일어난다.

결국 데이터를 CPU 코어별로 쪼개는(Sharding) 방식을 택해야 한다. sched_getcpu() 를 사용해 CPU별로 독립적인 배열을 할당하면 경합을 줄일 수 있다. 하지만 이마저도 스레드가 데이터를 수정하는 찰나의 순간에 OS 커널이 컨텍스트 스위칭을 해버리거나 다른 CPU로 스레드를 마이그레이션해버리면 데이터 정합성이 깨진다. 결국 또 Lock이 필요해진다.

## Restartable Sequences (rseq)의 동작 원리

rseq는 이 문제를 OS 커널과 유저스페이스 간의 가벼운 약속으로 해결한다. 핵심 철학은 내 크리티컬 섹션을 방해하지 마가 아니라, 방해할 거면 차라리 처음부터 다시 하게 해줘에 가깝다. Hacker News의 한 유저가 언급했듯, 이는 일종의 가벼운 STM(Software Transactional Memory) 패턴이다.

1. 스레드가 생성될 때 커널과 32바이트 크기의 TLS(Thread Local Storage) 메모리를 공유한다.
2. 유저스페이스 코드는 크리티컬 섹션(보통 10개 내외의 어셈블리 명령어)에 진입하기 전, 이 TLS 메모리의 rseq_cs 필드에 현재 실행 구간의 정보를 기록한다.
3. 만약 이 짧은 구간을 실행하는 도중 커널이 스레드를 선점(Preempt)하거나 다른 CPU로 옮겨버리면, 커널은 rseq_cs를 확인하고 스레드의 Program Counter(PC)를 미리 정의된 Abort Handler 로 강제 이동시킨다.
4. Abort Handler는 단순히 크리티컬 섹션의 처음으로 돌아가 연산을 재시도(Restart)한다.

이 방식은 Lock도, Atomic 명령어(메모리 배리어)도 필요로 하지 않는다. 그저 일반적인 메모리 읽기/쓰기 명령어만으로 CPU 로컬 데이터를 안전하게 수정할 수 있으며, 오버헤드는 고작 1ns 수준이다.

## 코드 레벨에서의 접근

Justine의 블로그에서 발췌한 CPU Sharded Linked List의 Push 연산 일부를 살펴보자.

```c
static inline void push(struct List *chunk) {
#ifdef __x86_64__
  asm volatile(
    ".pushsection .rodata.rseq,\"a\",@progbits\n"
    "300: .long 0\n" // rseq_cs::version
    // ... (중략) ...
    "301: lea 300b(%%rip),%%rcx\n"
    "  mov %%rcx,8(%1)\n" // rseq->rseq_cs 설정
    "303: jmp 301b\n" // Abort 발생 시 301로 재시도
  );
#endif
}
```

인라인 어셈블리를 사용해 명시적으로 rseq_cs 구조체를 레지스터에 올리고, 작업이 실패했을 때 jmp 301b 를 통해 재시도하는 로직을 볼 수 있다.

## Hacker News의 반응과 개인적인 생각

솔직히 말해서, Justine의 원본 글을 읽으면서 눈살이 찌푸려지는 대목이 있었다. $20,000짜리 96코어 워크스테이션이 없으면 공룡처럼 도태될 것이라는 도발적인 문구 때문이다. HN 커뮤니티에서도 이 점을 강하게 비판했다. 멀티코어 최적화 패턴을 이해하기 위해 굳이 비싼 장비를 살 필요는 없다. AWS에서 시간당 몇 달러면 128코어 인스턴스를 빌릴 수 있고, 심지어 Raspberry Pi에서도 3배 이상의 성능 향상을 확인할 수 있다.

하지만 저자의 오만한 태도를 걷어내고 나면, 기술적 통찰 자체는 매우 훌륭하다.

다만 한 가지 짚고 넘어갈 점이 있다. 2026년 현재, 비즈니스 로직을 작성하는 엔지니어가 직접 인라인 어셈블리로 rseq를 구현해야 할까? 절대 아니다. HN 댓글에서도 지적되었듯, rseq의 원작자가 유지보수하는 librseq 같은 라이브러리가 이미 존재한다. 우리가 직접 어셈블리를 짤 일은 없겠지만, 우리가 매일 사용하는 malloc() 내부에서 이런 마법이 일어나고 있다는 사실을 이해하는 것은 Senior 엔지니어로서 필수적인 소양이다.

흥미로운 점은 이 개념이 완전히 새로운 것이 아니라는 것이다. 과거 Sun Microsystems에서 이미 10여 년 전에 이와 유사한 개념을 논문으로 발표한 바 있다. 하드웨어의 발전이 과거의 이론을 현실의 필수 불가결한 기술로 끌어올린 셈이다.

## 결론: 프로덕션 레벨에서 쓸 만한가?

rseq는 단순한 장난감이 아니다. 이미 구글의 tcmalloc이나 최신 glibc 등에서 프로덕션 레벨로 굴러가고 있는 검증된 기술이다.

만약 당신이 초고성능 트레이딩 시스템(HFT)이나 대규모 동시성을 처리하는 데이터베이스 엔진을 개발하고 있다면, rseq 기반의 자료구조 도입을 심각하게 고려해 보아야 한다. Lock과 Atomic을 제거함으로써 얻는 Throughput 향상은 기존의 상식을 파괴하는 수준이기 때문이다. 하지만 일반적인 웹 애플리케이션 레벨이라면, 그저 최신 버전의 OS와 메모리 할당자를 사용하는 것만으로도 이 기술의 혜택을 공짜로 누릴 수 있을 것이다.

## References
- Original Article: https://justine.lol/rseq/
- Hacker News Thread: https://news.ycombinator.com/item?id=48346019
