---
title: "io_uring vs libaio 성능 비교와 숨겨진 IOMMU 함정: NVMe I/O의 진짜 병목은 어디인가"
description: "io_uring vs libaio 성능 비교와 숨겨진 IOMMU 함정: NVMe I/O의 진짜 병목은 어디인가"
pubDate: "2026-03-24T18:01:27Z"
---

최근 YDB 엔지니어링 팀에서 공개한 io_uring과 libaio의 커널 버전별 성능 비교 글을 읽으며, 과거 고성능 스토리지 시스템을 설계하며 겪었던 수많은 삽질들이 떠올랐다.

결론부터 말하자면 io_uring은 예상대로 libaio 대비 약 2배의 성능 향상을 보여주었다. 하지만 이 벤치마크의 진짜 묘미는 io_uring의 승리에 있는 것이 아니다. 최신 커널 환경에서 기본적으로 활성화되는 IOMMU가 무려 30%의 성능 저하(Regression)를 일으키는 주범이었다는 사실을 짚어낸 점이 이 글의 핵심이다.

15년 넘게 시스템 엔지니어링을 해오며 뼈저리게 느낀 점 중 하나는, "최신 기술 스택을 도입한다고 해서 무조건 성능이 오르지는 않는다"는 것이다. 이번 YDB의 벤치마크와 Hacker News 커뮤니티의 심도 있는 토론을 바탕으로, 현대 Linux I/O 스택의 현실과 우리가 프로덕션 환경에서 주의해야 할 점들을 파헤쳐보자.

## 왜 4K Random Write, 그리고 Single Thread인가?

벤치마크 결과를 분석하기 전에, 그들이 왜 4K Random Write 워크로드와 단일 fio Job(Single Thread)을 선택했는지 이해할 필요가 있다.

일부 개발자들은 "왜 Read/Write Mixed 워크로드가 아니냐"고 반문할 수 있다. 하지만 NVMe 디바이스에서 Mixed 워크로드를 테스트하면 디바이스 자체의 Garbage Collection이나 FTL(Flash Translation Layer) 상태에 따라 Latency가 널뛰기 십상이다. YDB 팀의 목표는 디바이스의 물리적 한계가 아니라 소프트웨어 스택(커널 I/O 경로)의 오버헤드를 측정하는 것이었다. 따라서 소프트웨어 Latency를 가장 명확하게 측정할 수 있고 DBMS에서 흔히 발생하는 패턴인 4K Random Write를 선택한 것은 매우 합리적인 접근이다.

또한, Throughput을 극한으로 끌어올리기 위해 수십 개의 스레드를 띄우는 대신 Single Job으로 테스트한 점도 눈여겨볼 만하다. 고성능 데이터베이스 엔진들은 보통 디스크당 1개의 스레드(Polling을 사용할 경우 최대 2개)를 할당하여 Context Switching을 최소화하는 아키텍처를 지향한다. 즉, 이 벤치마크는 철저히 **Latency-bound** 상황에서의 커널 오버헤드를 보여준다.

## io_uring이 libaio를 압도하는 진짜 이유

Linux 5.4에서 7.0-rc3에 이르는 커널 버전을 테스트한 결과, io_uring은 libaio 대비 거의 2배에 달하는 IOPS를 기록했다. 

단순히 "최신 API라서 빠르다"고 퉁치고 넘어갈 문제가 아니다. 시스템 내부를 들여다보면 명확한 구조적 차이가 존재한다.

- **Control Structure Copy:** libaio는 I/O submission과 completion 과정에서 컨트롤 구조체를 사용자 공간과 커널 공간 사이에서 복사해야 한다. 반면 io_uring은 Shared Ring Buffer와 Pre-registered Resources를 활용해 이 복사 오버헤드를 원천적으로 제거한다.
- **Syscall Latency:** 내 경험상으로도 AIO syscall의 Latency는 아무리 튜닝을 잘해도 예측 불가능하게 튀는(Spike) 구간이 존재한다. io_uring은 링 버퍼를 통한 비동기 처리로 이러한 병목을 우회한다.

솔직히 말해, 아직도 신규 고성능 프로젝트에서 libaio를 고집할 이유는 전혀 없다. io_uring은 이제 실험적인 장난감이 아니라 엔터프라이즈의 표준으로 자리 잡았다.

## 예상치 못한 함정: IOMMU가 훔쳐간 30%의 성능

이 스토리의 가장 흥미로운 부분은 커널 버전을 업그레이드하면서 발생한 성능 회귀(Regression) 현상이다. 원인은 놀랍게도 **IOMMU(Input/Output Memory Management Unit)** 였다. 최신 커널 릴리즈에서는 보안과 격리를 위해 IOMMU가 기본적으로 활성화되는 추세인데, 이것이 30%의 성능 하락을 가져왔다.

IOMMU는 디바이스가 물리 메모리에 직접 접근(DMA)할 때 메모리 주소를 변환하고 보호하는 역할을 한다. 가상화 환경이나 보안이 극도로 중요한 환경에서는 필수적이지만, 천만 단위의 IOPS를 처리해야 하는 베어메탈 스토리지 노드에서는 치명적인 병목이 된다.

Hacker News에서 성능 분석의 대가인 Tanel Poder(anon)가 남긴 코멘트는 이 현상의 본질을 정확히 꿰뚫고 있다. IOMMU가 활성화되면 DMA Translation 과정 자체의 오버헤드도 문제지만, 인터럽트 기반의 I/O Completion 워크로드에서 엄청난 **Interrupt Handling Overhead** 가 발생한다. 

이러한 인터럽트 오버헤드는 일반적인 CPU Flamegraph에서는 무작위로 흩어져 나타나기 때문에 시각적으로 찾아내기 매우 까다롭다. 이를 제대로 추적하려면 다음과 같은 도구들이 필요하다.

- `/proc/interrupts` 를 통한 IOPS 대비 인터럽트 발생 횟수 정규화
- `bcc-tools` 의 `hardirqs -d` 를 활용한 IRQ 처리 Latency 히스토그램 분석
- `perf record -g` 로 인터럽트 핸들링 코드 경로의 병목 확인

## 결론 및 실무 적용 가이드

이번 사례는 우리가 인프라를 다룰 때 "기본 설정(Default)"을 얼마나 의심해봐야 하는지 보여주는 완벽한 교훈이다. 클라우드 벤더가 제공하는 최신 OS 이미지나 최신 커널을 적용했다고 해서 안심할 수 없다.

만약 당신이 NVMe 기반의 고성능 데이터베이스나 스토리지 시스템을 베어메탈에서 운영 중이라면, 당장 커널 파라미터를 확인해보기 바란다. 신뢰할 수 있는 디바이스 환경이라면 부팅 파라미터에 `iommu=pt` (Passthrough mode)를 설정하여 IOMMU의 주소 변환 과정을 우회하는 것을 강력히 권장한다. 이렇게 하면 보안(Device Isolation)의 이점을 일부 포기하는 대신, 디바이스 본연의 성능을 100% 끌어낼 수 있다.

기술의 발전은 우리에게 io_uring이라는 강력한 무기를 쥐여주었지만, 동시에 IOMMU와 같은 복잡한 추상화 레이어를 겹겹이 쌓아 올리고 있다. 진정한 시니어 엔지니어라면 새 API의 사용법을 익히는 것을 넘어, 그 아래 커널과 하드웨어가 어떻게 상호작용하는지 꿰뚫어 볼 수 있어야 한다.

---
**References:**
- Original Article: [io_uring, libaio performance across Linux kernels and an unexpected IOMMU trap](https://blog.ydb.tech/how-io-uring-overtook-libaio-performance-across-linux-kernels-and-an-unexpected-iommu-trap-ea6126d9ef14)
- Hacker News Thread: [https://news.ycombinator.com/item?id=47502193](https://news.ycombinator.com/item?id=47502193)
