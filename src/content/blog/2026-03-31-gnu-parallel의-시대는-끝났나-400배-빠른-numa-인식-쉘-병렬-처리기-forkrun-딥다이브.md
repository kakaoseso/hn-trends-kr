---
title: "GNU Parallel의 시대는 끝났나? 400배 빠른 NUMA 인식 쉘 병렬 처리기 forkrun 딥다이브"
description: "GNU Parallel의 시대는 끝났나? 400배 빠른 NUMA 인식 쉘 병렬 처리기 forkrun 딥다이브"
pubDate: "2026-03-31T19:25:33Z"
---

데이터 엔지니어나 인프라를 다루는 시니어 엔지니어라면, 수십에서 수백 GB에 달하는 텍스트 로그나 TSV 파일을 쉘 스크립트로 파싱해본 경험이 있을 것이다. 이때 우리의 오랜 친구는 항상 `xargs -P`나 `GNU Parallel`이었다.

하지만 최신 다중 소켓(Multi-socket) EPYC이나 Xeon 서버에서 GNU Parallel을 돌려보면 뭔가 이상함을 느끼게 된다. htop을 켜보면 코어는 64개인데, 전체 CPU 사용률은 5~10% 언저리에서 맴돌고, 1~2개의 코어만 100%를 찍으며 병목을 일으킨다. 무거운 정규식 파싱과 IPC(Inter-Process Communication) 오버헤드, 그리고 중앙 집중식 디스패처가 최신 하드웨어의 발목을 잡는 것이다.

최근 Hacker News에서 흥미로운 프로젝트를 하나 발견했다. GNU Parallel 대비 최대 400배 빠르고, NUMA 아키텍처를 완벽하게 인식하여 CPU 사용률을 95% 이상으로 끌어올린다는 쉘 병렬 처리기, [forkrun](https://github.com/jkool702/forkrun)이다.

처음엔 "또 벤치마크 숫자 놀음이겠지" 하고 넘기려 했으나, 내부 아키텍처를 뜯어보고 나니 꽤나 감탄이 나왔다. 이 툴은 단순히 코딩을 잘한 수준이 아니라, 현대 Linux 시스템 프로그래밍의 정수를 쉘 유틸리티에 우겨넣은 괴물이다.

## 어떻게 400배의 속도 향상을 이루어냈는가?

GNU Parallel의 가장 큰 문제는 Perl 기반의 무거운 런타임과, 마스터 프로세스가 모든 작업을 분배하는 구조에 있다. forkrun은 이 병목을 해결하기 위해 철저하게 물리적 지역성(Physical Locality)을 유지하는 4단계 파이프라인을 설계했다.

### 1. Ingest: NUMA-Aware "Born-Local" 설계
가장 인상 깊은 부분이다. forkrun은 표준 입력(stdin)에서 데이터를 읽을 때 단순한 `read` 시스템 콜을 쓰지 않는다. `splice()`를 사용해 데이터를 커널 공간에서 공유 `memfd`로 Zero-copy 전송한다. 

더 놀라운 것은 NUMA 처리 방식이다. 다중 소켓 시스템에서 `set_mempolicy(MPOL_BIND)`를 호출해, 워커 스레드가 데이터를 건드리기 전에 각 청크의 메모리 페이지를 타겟 NUMA 노드에 강제로 할당한다. 크로스 소켓 메모리 트래픽은 최신 서버에서 성능을 갉아먹는 주범인데, 이를 원천 차단한 것이다.

### 2. Index: SIMD를 활용한 메모리 대역폭 스캐닝
레코드의 경계를 찾는 작업(보통 줄바꿈 문자 찾기)은 AVX2/NEON SIMD 명령어를 사용해 메모리 대역폭 한계치까지 속도를 끌어올린다. 런타임 상태에 따라 동적으로 배치 크기를 조절하고, 그 오프셋 마커를 Lock-free 링 버퍼에 기록한다.

### 3. Claim: Contention-Free 작업 할당
보통 이런 병렬 처리에서는 CAS(Compare-And-Swap) 재시도 루프를 돌며 락 경합이 발생하기 마련이다. forkrun은 단일 `atomic_fetch_add` 연산만으로 워커가 배치를 가져가게 만들었다. 락도 없고, 경합도 없다. 남는 작업은 에스크로 파이프에 넣어 유휴 워커가 훔쳐갈 수 있게(Work-stealing) 설계했다.

### 4. Reclaim: PUNCH_HOLE을 이용한 메모리 관리
작업이 끝난 메모리는 어떻게 처리할까? 여기서 개발자의 짬바가 느껴진다. `fallocate(PUNCH_HOLE)`을 사용해 처리 완료된 데이터의 메모리에 구멍을 뚫어버린다(해제한다). 이렇게 하면 오프셋 좌표계는 그대로 유지하면서도 메모리 사용량을 완벽하게 통제할 수 있다. 데이터베이스 스토리지 엔진에서나 볼 법한 기법을 쉘 스크립트 도구에 적용한 것이다.

## 배포 방식: 천재적인가, 보안 부채인가?

기술적으로는 훌륭하지만, 이 도구의 배포 방식은 호불호가 극명하게 갈릴 것이다.

```bash
source <(curl -sL https://raw.githubusercontent.com/jkool702/forkrun/main/frun.bash)
```

forkrun은 의존성(Perl, Python 등)을 완전히 없애기 위해 단일 Bash 파일 내부에 컴파일된 C 익스텐션을 Base64로 인코딩하여 박아넣었다. 스크립트를 `source` 하면 이 C 코드가 동적으로 로드되는 방식이다.

Hacker News 커뮤니티에서도 이 부분에 대한 갑론을박이 뻔히 예상된다. "curl 파이프 스크립트 실행은 보안의 적이다"라는 비판을 피하기 위해, 개발자는 GitHub Actions CI 워크플로우를 통해 빌드된 Base64 블롭의 출처를 투명하게 추적할 수 있도록 만들었다. 

솔직한 내 의견을 말하자면, 인프라 보안 팀이 이 스크립트를 프로덕션 서버에 그대로 올리는 것을 허락할 리 없다. 하지만 로컬 개발 환경이나, 격리된 데이터 전처리 클러스터에서는 이만한 드롭인(Drop-in) 대체제가 없을 것이다.

## 사용 예시

사용법은 기존 도구들과 비슷하게 매우 직관적이다.

```bash
# 커스텀 bash 함수를 네이티브로 병렬 실행
frun my_bash_func < inputs.txt 

# 파이프 기반 입력 및 순서 보장 출력 (-k)
cat file_list | frun -k sed 's/old/new/' 

# 고속 스트리밍 처리 (-s)
frun -k -s sort < records.tsv 
```

## 결론: 그래서 써야 할까?

- **로컬/개발 환경에서의 대규모 데이터 랭글링:** 적극 권장한다. 수십 기가의 로그를 `grep`이나 `awk`로 파싱할 때 퇴근 시간을 1시간 이상 앞당겨 줄 것이다.
- **크리티컬한 프로덕션 파이프라인:** 아직은 보류. 개발자도 로드맵에서 언급했듯, 아직 워커 크래시 시 재시도(Retries)나 중단 후 재개(Resume) 같은 결함 허용(Fault-tolerance) 기능이 부족하다.

결론적으로 forkrun은 최신 하드웨어의 잠재력을 100% 끌어내지 못하는 레거시 도구들에 대한 통쾌한 일격이다. Bash 스크립트의 탈을 쓴 고성능 C 애플리케이션이며, NUMA와 메모리 관리 최적화에 관심이 있는 엔지니어라면 소스 코드를 한 번쯤 정독해 볼 가치가 충분하다.

**References:**
- Original Article: https://github.com/jkool702/forkrun
- Hacker News Thread: https://news.ycombinator.com/item?id=47541746
