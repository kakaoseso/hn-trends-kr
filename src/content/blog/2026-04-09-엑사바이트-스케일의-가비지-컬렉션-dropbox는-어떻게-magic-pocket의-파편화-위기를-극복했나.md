---
title: "엑사바이트 스케일의 가비지 컬렉션: Dropbox는 어떻게 Magic Pocket의 파편화 위기를 극복했나"
description: "엑사바이트 스케일의 가비지 컬렉션: Dropbox는 어떻게 Magic Pocket의 파편화 위기를 극복했나"
pubDate: "2026-04-09T11:15:40Z"
---

데이터의 불변성(Immutability)은 현대 스토리지 아키텍처에서 거의 종교에 가까운 개념이 되었습니다. 상태를 덮어쓰지 않고 항상 새로운 데이터를 쓰는 방식은 동시성 제어, 데이터 복구, 캐싱 등 수많은 문제를 우아하게 해결해 줍니다. 하지만 청구서가 날아오는 시점이 되면 이야기가 달라집니다. 불변성은 필연적으로 '쓰레기(Garbage)'를 남기기 때문입니다.

최근 Dropbox 엔지니어링 블로그에 올라온 Magic Pocket(Dropbox의 자체 구축 엑사바이트 스케일 Immutable Blob Store)의 스토리지 효율성 개선기에 대한 글을 읽었습니다. 이 글은 단순히 "우리 시스템이 이렇게 빠르고 좋습니다"라고 자랑하는 뻔한 PR 용도의 글이 아닙니다. 새로운 서비스를 배포했다가 스토리지 파편화(Fragmentation)가 폭증하여 발등에 불이 떨어졌고, 이를 해결하기 위해 Compaction(압축/병합) 전략을 밑바닥부터 다시 설계해야만 했던 치열한 엔지니어링 생존기입니다.

개인적으로 과거에 S3와 로컬 디스크를 혼합하여 대규모 Immutable 스토리지 엔진을 설계해 본 경험이 있기에, 이들이 겪은 고통이 남 일 같지 않았습니다. 이번 포스트에서는 Dropbox가 엑사바이트 스케일에서 어떻게 물리적 공간을 회수하는지, 그리고 그 과정에서 우리가 배울 수 있는 시스템 설계의 교훈을 깊이 파헤쳐 보겠습니다.

## 불변성의 대가: 파편화와 스토리지 오버헤드

Magic Pocket은 데이터가 한 번 기록되면 절대 제자리에서 수정(in-place update)되지 않습니다. 사용자가 파일을 수정하거나 삭제하면 새로운 Blob이 기록되고, 기존 데이터는 논리적으로 삭제된 것으로 마킹될 뿐 물리적 디스크 공간은 그대로 차지합니다.

문제는 여기서 시작됩니다. 논리적으로 삭제된 데이터가 쌓이면, 스토리지 볼륨(Volume) 내부에는 유효한 데이터(Live Data)와 쓰레기 데이터가 섞이게 됩니다. Dropbox는 이를 해결하기 위해 백그라운드에서 Compaction 프로세스를 돌립니다. 유효한 데이터만 골라내어 새로운 볼륨에 촘촘하게 눌러 담고, 기존 볼륨은 폐기하여 공간을 회수하는 방식입니다.

![Compaction lifecycle](https://dropbox.tech/cms/content/dam/dropbox/tech-blog/en-us/2026/march/magic-pocket-compaction/diagram/Diagram%201.png/_jcr_content/renditions/Diagram%201.webp)

그런데 최근 Dropbox가 Write Amplification을 줄이기 위해 도입한 'Live Coder'(실시간 Erasure Coding 서비스)가 예상치 못한 나비효과를 일으켰습니다. 백그라운드 쓰기 작업의 효율은 좋아졌지만, 이 경로를 통해 생성된 볼륨들의 유효 데이터 비율이 5% 미만으로 떨어지는 끔찍한 파편화가 발생한 것입니다. 1TB 볼륨에 실제 유효한 데이터는 50GB도 안 되는데, 디스크 공간은 1TB를 온전히 점유하는 상황이 엑사바이트 스케일로 벌어졌다고 상상해 보십시오. 인프라 비용이 수직 상승하는 소리가 들릴 것입니다.

## 단일 전략의 실패와 다층적 방어선 구축

기존에 Dropbox가 사용하던 Compaction 전략(L1)은 지극히 상식적인 방식이었습니다. 대부분의 볼륨이 꽉 차 있다는 가정하에, 약간의 빈 공간이 있는 'Host' 볼륨에 파편화된 'Donor' 볼륨의 데이터를 채워 넣는 식입니다. 

하지만 5% 미만으로 채워진 극단적인 볼륨이 수만 개씩 쏟아지는 상황에서 L1 전략은 무용지물이었습니다. 이를 해결하기 위해 그들은 볼륨의 파편화 정도에 따라 세 가지 계층화된 전략(L1, L2, L3)을 도입했습니다.

### L2: Dynamic Programming을 활용한 볼륨 패킹

내가 이 아티클에서 가장 흥미롭게 읽은 부분은 L2 전략입니다. 적당히 비어있는 볼륨들을 모아 하나의 꽉 찬 새로운 볼륨으로 만드는 과정은 전형적인 배낭 문제(Knapsack Problem)입니다. Dropbox 엔지니어들은 이를 Dynamic Programming(DP)으로 해결했습니다.

```python
# Dropbox L2 Compaction의 개념적 DP 모델
def plan_l2_compaction(volumes, max_vol_bytes, max_vols, granularity):
    # 1. 탐색 공간(Search Space)을 줄이기 위해 바이트와 용량을 granularity로 스케일링
    scaled_volumes = scale_down(volumes, granularity)
    scaled_capacity = max_vol_bytes / granularity
    
    # 2. i(인덱스), k(사용할 볼륨 수 제한), c(용량)에 대한 3차원 DP 테이블 구성
    dp_table = initialize_3d_table(len(volumes), max_vols, scaled_capacity)
    choice_table = parallel_table_for_tracking()
    
    # 3. 최대 패킹 바이트를 찾는 DP 루프 실행
    for i, vol in enumerate(scaled_volumes):
        # ... DP 로직 수행 및 choice_table 업데이트 ...
        
    # 4. 최적의 (k, capacity) 조합에서 백트래킹하여 선택된 볼륨 도출
    return backtrack(choice_table, best_k, best_c)
```

여기서 주목할 점은 granularity(확장 계수)를 도입해 DP 테이블의 크기를 줄였다는 것입니다. 프로덕션 환경에서 수많은 볼륨을 조합해야 하는데, 바이트 단위로 DP를 돌리면 메모리와 연산량이 폭발합니다. 적절한 단위로 청크를 나누어 근사치(Approximation)를 구하는 이 접근법은 실무 엔지니어링의 정수라고 할 수 있습니다. 완벽한 100% 패킹을 위해 CPU를 태우느니, 98% 패킹을 10배 빠르게 수행하는 것이 분산 시스템에서는 훨씬 현명한 선택입니다.

### L3: 극단적 파편화를 위한 스트리밍 파이프라인

유효 데이터가 1%도 남지 않은 극단적인 꼬리(Tail) 분포의 볼륨들은 L2로도 처리하기 비효율적입니다. 그래서 도입한 L3 전략은 아예 기존의 'Live Coder'를 스트리밍 파이프라인으로 재활용합니다. 텅텅 빈 볼륨들에 남아있는 소수의 Blob들을 긁어모아 실시간으로 새로운 Erasure Coded 볼륨으로 인코딩해 버리는 것입니다.

이 방식은 공간 회수 속도는 압도적으로 빠르지만 치명적인 트레이드오프가 있습니다. 이동하는 모든 Blob이 완전히 새로운 볼륨에 기록되므로, 메타데이터 시스템(Panda)에 엄청난 쓰기 부하를 발생시킵니다. 기존 L1/L2는 Donor 볼륨의 메타데이터만 일부 수정하면 되지만, L3는 사실상 전체 데이터의 주소를 다시 매핑해야 합니다.

## 메타데이터: 스토리지 최적화의 숨겨진 병목

현업에서 대규모 스토리지를 운영해 본 엔지니어라면 깊이 공감할 대목이 바로 이 부분입니다. 스토리지를 최적화하려고 데이터를 움직이면, 결국 병목은 디스크 I/O나 네트워크 대역폭이 아니라 메타데이터 DB에서 터집니다.

Dropbox 역시 이 문제를 피할 수 없었습니다. 그들은 메타데이터 시스템이 무너지는 것을 막기 위해 정적 튜닝(Static Tuning)을 버리고 동적 제어 루프(Dynamic Control Loop)를 도입했습니다. 시스템의 오버헤드 지표를 실시간으로 모니터링하면서, Compaction 대상이 되는 볼륨의 기준치(Threshold)를 동적으로 조절하고, 트래픽을 데이터센터(Cell) 내부로 국한시켜 Cross-cluster 대역폭을 보호했습니다.

"수동 튜닝은 엑사바이트 스케일에서 작동하지 않는다"는 그들의 결론은, 인프라 규모가 일정 수준을 넘어서면 결국 모든 운영 로직이 자동화된 Control Plane에 의해 관리되어야 함을 다시 한번 증명합니다.

## 총평 및 Hacker News 반응

현재 이 주제에 대해 Hacker News 커뮤니티에서는 아직 활발한 토론이 이루어지지 않고 있습니다. 아마도 엑사바이트 단위의 자체 스토리지 엔진을 운영하며 파편화로 고통받는 회사가 전 세계에 손꼽힐 정도로 적기 때문일 것입니다. 대부분의 개발자는 AWS S3의 Lifecycle Rule을 설정하는 것으로 이 문제를 외주화하고 있으니까요.

하지만 이 아티클은 분산 시스템을 설계하는 시니어 엔지니어라면 반드시 정독해야 할 가치가 있습니다. 시스템의 한쪽(Throughput)을 최적화했을 때 다른 쪽(Storage Overhead)이 어떻게 망가질 수 있는지, 그리고 장애 상황에서 단일 휴리스틱에 의존하는 것이 얼마나 위험한지 명확히 보여주기 때문입니다.

결론적으로 Immutable Architecture는 결코 공짜가 아닙니다. 불변성이 주는 시스템적 안정성의 이면에는, 끊임없이 쓰레기를 치우고 데이터를 재배치해야 하는 가혹한 Compaction Lifecycle이 존재합니다. Dropbox의 이번 L1-L2-L3 다층 전략은 이 복잡한 생태계를 안정적으로 유지하기 위한 매우 우아하고 실용적인 해답입니다.

---
- **원문 블로그:** [Improving storage efficiency in Magic Pocket, Dropbox's immutable blob store](https://dropbox.tech/infrastructure/improving-storage-efficiency-in-magic-pocket-our-immutable-blob-store)
- **Hacker News 토론:** [https://news.ycombinator.com/item?id=47632157](https://news.ycombinator.com/item?id=47632157)
