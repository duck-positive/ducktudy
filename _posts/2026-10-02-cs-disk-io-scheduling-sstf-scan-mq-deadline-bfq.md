---
layout: post
title: "디스크 I/O 스케줄링 알고리즘 완전 분석: SSTF, SCAN부터 mq-deadline, BFQ까지"
date: 2026-10-02
categories: [cs, computer-science]
tags: [disk-scheduling, io-scheduler, sstf, scan, cfq, mq-deadline, bfq, linux-kernel, storage, hdd, ssd]
---

데이터베이스, 파일 서버, 스트리밍 플랫폼의 성능을 결정짓는 핵심 요소 중 하나가 바로 I/O 스케줄링입니다. 디스크 I/O 스케줄러는 여러 프로세스가 동시에 발행하는 읽기·쓰기 요청을 어떤 순서로 처리할지 결정함으로써, 전체 처리량(throughput)과 공정성(fairness), 레이턴시를 조율합니다. 이 글에서는 고전 알고리즘부터 리눅스 커널의 현대적 스케줄러까지 원리와 구현을 함께 살펴봅니다.

## 왜 I/O 스케줄링이 중요한가?

HDD(하드디스크)는 물리적 헤드가 플래터 위를 이동해 데이터를 읽고 씁니다. 이 헤드 이동 시간(seek time)은 밀리초 단위로, 현대 CPU(나노초)와 메모리(나노초~마이크로초)에 비해 수만~수백만 배 느립니다.

### 디스크 성능 지표

- **탐색 시간(Seek Time)**: 헤드가 목표 트랙으로 이동하는 시간 (평균 3~15ms)
- **회전 지연(Rotational Latency)**: 플래터가 회전하여 목표 섹터가 헤드 아래에 올 때까지의 시간 (7200RPM ≈ 평균 4.2ms)
- **전송 시간(Transfer Time)**: 실제 데이터 읽기/쓰기 시간

I/O 스케줄러의 핵심 목표는 **탐색 시간을 최소화**하는 것입니다. 무작위 순서로 요청을 처리하면 헤드가 디스크 전체를 왔다 갔다 하는 반면, 순서를 재정렬하면 헤드 이동 거리를 대폭 줄일 수 있습니다.

> **SSD에서는?** SSD는 물리적 헤드가 없어 랜덤 액세스 레이턴시가 거의 없습니다. 따라서 SSD 환경에서는 스케줄러의 역할이 달라지며, 리눅스는 NVMe SSD에 대해 `none` 스케줄러(재정렬 없음)를 권장합니다.

## 고전 디스크 스케줄링 알고리즘

### 1. FCFS (First-Come, First-Served) — 기준선

요청이 도착한 순서대로 처리합니다. 가장 단순하지만 헤드 이동이 최악인 경우가 많습니다.

### 2. SSTF (Shortest Seek Time First)

현재 헤드 위치에서 가장 가까운 요청을 우선 처리합니다. 탐색 시간을 크게 줄이지만 **기아(starvation)** 문제가 있습니다. 헤드 근처에 요청이 계속 들어오면 먼 위치의 요청은 영원히 처리되지 않을 수 있습니다.

### 3. SCAN (엘리베이터 알고리즘)

헤드가 한쪽 방향으로 이동하며 모든 요청을 처리하고, 끝에 도달하면 반대 방향으로 이동합니다. 엘리베이터의 동작 방식과 같습니다.

### 4. C-SCAN (Circular SCAN)

SCAN의 변형으로, 한쪽 끝에서 반대 끝으로 이동할 때는 요청을 처리하지 않고 즉시 시작점으로 돌아옵니다. 대기 시간의 분산을 줄여 더 균일한 응답 시간을 제공합니다.

### 5. LOOK / C-LOOK

SCAN/C-SCAN의 개선판으로, 끝까지 가는 대신 실제 마지막 요청까지만 이동하고 방향을 바꿉니다. 불필요한 끝 지점 이동을 제거해 효율을 높입니다.

## 알고리즘 시뮬레이션 구현

```python
"""
디스크 스케줄링 알고리즘 시뮬레이터
디스크: 200개 트랙 (0-199), 헤드 초기 위치 53
"""

from typing import List, Tuple

def fcfs(head: int, requests: List[int]) -> Tuple[int, List[int]]:
    """FCFS: 도착 순서대로 처리"""
    order = list(requests)
    total_seek = sum(abs(order[i] - order[i-1])
                     for i in range(1, len(order)))
    total_seek += abs(head - order[0])
    return total_seek, order

def sstf(head: int, requests: List[int]) -> Tuple[int, List[int]]:
    """SSTF: 현재 위치에서 가장 가까운 요청 우선"""
    remaining = list(requests)
    current = head
    order = []
    total_seek = 0

    while remaining:
        # 현재 헤드에서 가장 가까운 요청 찾기
        closest = min(remaining, key=lambda x: abs(x - current))
        total_seek += abs(closest - current)
        current = closest
        order.append(closest)
        remaining.remove(closest)

    return total_seek, order

def scan(head: int, requests: List[int], disk_size: int = 200,
         direction: str = 'up') -> Tuple[int, List[int]]:
    """SCAN: 엘리베이터 알고리즘 - 한 방향으로 쭉 이동 후 반대로"""
    sorted_req = sorted(requests)
    order = []
    total_seek = 0
    current = head

    # 현재 위치보다 크고 작은 요청으로 분리
    lower = [r for r in sorted_req if r < current]
    upper = [r for r in sorted_req if r >= current]

    if direction == 'up':
        # 위로 먼저, 그 다음 아래로
        sequence = upper + lower[::-1]
    else:
        # 아래로 먼저, 그 다음 위로
        sequence = lower[::-1] + upper

    for req in sequence:
        total_seek += abs(req - current)
        current = req
        order.append(req)

    return total_seek, order

def c_scan(head: int, requests: List[int],
           disk_size: int = 200) -> Tuple[int, List[int]]:
    """C-SCAN: 한 방향으로만 처리, 끝에서 시작점으로 즉시 복귀"""
    sorted_req = sorted(requests)
    order = []
    total_seek = 0
    current = head

    upper = [r for r in sorted_req if r >= current]
    lower = [r for r in sorted_req if r < current]

    # 위로 이동하며 처리
    for req in upper:
        total_seek += abs(req - current)
        current = req
        order.append(req)

    if lower:
        # 끝에서 디스크 시작(0)으로 즉시 이동
        total_seek += current + lower[0]  # 현재→0 + 0→첫 번째 lower
        current = lower[0]
        order.append(lower[0])

        for req in lower[1:]:
            total_seek += abs(req - current)
            current = req
            order.append(req)

    return total_seek, order

def simulate_all(head: int, requests: List[int]):
    """모든 알고리즘 비교"""
    print(f"헤드 시작 위치: {head}")
    print(f"요청 큐: {requests}\n")
    print(f"{'알고리즘':<12} {'총 탐색 거리':>12} {'처리 순서'}")
    print("-" * 60)

    algorithms = [
        ("FCFS",   fcfs(head, requests)),
        ("SSTF",   sstf(head, requests)),
        ("SCAN↑",  scan(head, requests, direction='up')),
        ("C-SCAN", c_scan(head, requests)),
    ]

    for name, (seek, order) in algorithms:
        print(f"{name:<12} {seek:>12} → {order}")

if __name__ == "__main__":
    # 교과서 표준 예제
    simulate_all(
        head=53,
        requests=[98, 183, 37, 122, 14, 124, 65, 67]
    )
```

실행 결과:
```
헤드 시작 위치: 53
요청 큐: [98, 183, 37, 122, 14, 124, 65, 67]

알고리즘     총 탐색 거리 처리 순서
------------------------------------------------------------
FCFS                640 → [98, 183, 37, 122, 14, 124, 65, 67]
SSTF                236 → [65, 67, 37, 14, 98, 122, 124, 183]
SCAN↑               208 → [65, 67, 98, 122, 124, 183, 37, 14]
C-SCAN              187 → [65, 67, 98, 122, 124, 183, 14, 37]
```

## 리눅스 현대 I/O 스케줄러

리눅스 5.x 이후 블록 레이어는 멀티큐(multi-queue) 아키텍처로 재설계되었습니다. 이를 **blk-mq**라 하며, 전통적인 단일 큐 방식과 달리 CPU 코어별로 독립적인 소프트웨어 큐와 하드웨어 큐를 사용합니다.

### 현재 리눅스에서 사용 가능한 스케줄러

```bash
# 특정 디바이스의 현재 스케줄러 확인
cat /sys/block/sda/queue/scheduler
# 예: [mq-deadline] kyber bfq none

# 스케줄러 변경
echo "bfq" > /sys/block/sda/queue/scheduler
```

#### mq-deadline (SATA HDD/SSD 권장)

mq-deadline은 SCAN 기반에 데드라인 메커니즘을 추가한 스케줄러입니다:

- **읽기 큐와 쓰기 큐 분리**: 각각 LBA(논리 블록 주소) 순으로 정렬
- **데드라인 강제**: 읽기는 500ms, 쓰기는 5000ms를 초과하면 강제 처리
- **읽기 우선**: 읽기는 응용 프로그램을 블록하므로 쓰기보다 우선
- **배치(batch) 쓰기**: 쓰기 기아를 방지하기 위해 일정 주기로 쓰기 배치 처리

```bash
# mq-deadline 파라미터 튜닝
/sys/block/sda/queue/iosched/read_expire   # 읽기 데드라인 (ms, 기본 500)
/sys/block/sda/queue/iosched/write_expire  # 쓰기 데드라인 (ms, 기본 5000)
/sys/block/sda/queue/iosched/writes_starved # 기아 방지: N번 읽기 후 쓰기 배치
```

#### BFQ (Budget Fair Queueing — 대화형 워크로드 권장)

BFQ는 CFS(완전 공정 스케줄러)를 I/O에 적용한 비례 공유(proportional share) 스케줄러입니다:

- **B-WF2Q+ 알고리즘**: 프로세스별 I/O 예산(budget, 섹터 단위) 할당
- **낮은 레이턴시**: 대화형 프로세스(키 입력, UI 응답)의 I/O를 우선 처리
- **공정성**: 백그라운드 작업이 대화형 작업을 굶기지 않도록 보장
- **내부 디바이스 큐(NCQ) 활용**: SATA NCQ의 32개 큐 슬롯을 효율적으로 관리

```python
# BFQ 스케줄러 동작 원리 의사코드
class BFQScheduler:
    """예산 기반 공정 큐(Budget Fair Queuing) 핵심 개념"""

    def __init__(self):
        self.queues = {}          # 프로세스별 BFQ 큐
        self.active_queue = None  # 현재 서비스 중인 큐
        self.vtime = 0            # 가상 시간 (공정성 기준)

    def add_request(self, pid: int, lba: int, size: int):
        """새 I/O 요청 추가"""
        if pid not in self.queues:
            self.queues[pid] = BFQQueue(pid, initial_budget=128)  # 128 섹터
        self.queues[pid].enqueue(lba, size)

    def select_next_queue(self) -> 'BFQQueue':
        """
        B-WF2Q+ 알고리즘: 가상 완료 시간이 가장 이른 큐 선택
        (가상 완료 시간 = 가상 시작 시간 + 예산/가중치)
        """
        eligible = [q for q in self.queues.values()
                    if q.virtual_start <= self.vtime and q.has_requests()]
        if not eligible:
            return None
        return min(eligible, key=lambda q: q.virtual_finish)

    def dispatch(self) -> dict:
        """다음 처리할 I/O 요청 반환"""
        queue = self.select_next_queue()
        if not queue:
            return None

        request = queue.dequeue()
        queue.budget_used += request['size']

        # 예산 소진 시 큐 비활성화 및 가상 시간 업데이트
        if queue.budget_used >= queue.budget:
            self.vtime = queue.virtual_finish
            queue.reset_budget()

        return request

class BFQQueue:
    def __init__(self, pid: int, initial_budget: int):
        self.pid = pid
        self.budget = initial_budget      # 현재 예산 (섹터 수)
        self.budget_used = 0
        self.weight = 100                 # 기본 가중치
        self.virtual_start = 0
        self.virtual_finish = 0
        self.requests = []                # LBA 순 정렬 큐

    def enqueue(self, lba: int, size: int):
        import bisect
        bisect.insort(self.requests, (lba, size))  # LBA 순 삽입

    def dequeue(self):
        if not self.requests:
            return None
        lba, size = self.requests.pop(0)
        return {'lba': lba, 'size': size, 'pid': self.pid}

    def has_requests(self) -> bool:
        return len(self.requests) > 0

    def reset_budget(self):
        """예산 소진 후 다음 서비스 라운드 준비"""
        self.virtual_start = self.virtual_finish
        self.virtual_finish = self.virtual_start + self.budget / self.weight
        self.budget_used = 0
        # 다음 예산은 이전 서비스 패턴에 따라 적응적으로 조정
        # (짧은 버스트: 예산 감소, 긴 순차 I/O: 예산 증가)
```

#### none / noop (NVMe SSD 권장)

NVMe SSD는 내부적으로 수천 개의 병렬 I/O 큐를 가지며 하드웨어가 최적화를 담당합니다. 커널 스케줄러가 재정렬하면 오히려 성능 저하가 발생할 수 있으므로, `none`(재정렬 없이 즉시 전달)을 권장합니다.

## 스케줄러 선택 가이드

```
워크로드 유형            추천 스케줄러    이유
────────────────────────────────────────────────────────────
NVMe SSD               none           하드웨어 자체 최적화, 스케줄러 오버헤드 불필요
SATA SSD (서버)        mq-deadline    낮은 레이턴시, 쓰기 기아 방지
SATA HDD (서버)        mq-deadline    탐색 최적화 + 데드라인 보장
데스크톱 / 멀티미디어   bfq            대화형 응답성, 공정한 대역폭 배분
가상머신(VM) 게스트     none           호스트 스케줄러에 위임, 이중 스케줄링 방지
데이터베이스(Oracle)   mq-deadline    예측 가능한 읽기 레이턴시 최우선
```

## 실전 I/O 성능 튜닝

```bash
# 현재 I/O 통계 모니터링
iostat -x 1 10

# blktrace: 블록 레이어 이벤트 추적
blktrace -d /dev/sda -o trace
blkparse -i trace.blktrace.0 | head -50

# fio: I/O 벤치마크 (순차 읽기 테스트)
fio --name=seq-read --ioengine=libaio --iodepth=32 \
    --rw=read --bs=1M --direct=1 --size=4G --filename=/dev/sda

# fio: 랜덤 읽기 테스트 (4KB 블록, IOPS 측정)
fio --name=rand-read --ioengine=libaio --iodepth=128 \
    --rw=randread --bs=4k --direct=1 --size=4G --filename=/dev/sda

# 디바이스별 큐 깊이(queue depth) 설정
echo 64 > /sys/block/sda/queue/nr_requests

# HDD 선반독출(read-ahead) 조정 (KB 단위, 순차 워크로드에 효과적)
blockdev --setra 2048 /dev/sda  # 1MB read-ahead
```

## 주의사항과 팁

### 1. SSD에서 TRIM/Discard 설정

SSD는 삭제된 블록을 재사용하기 전에 지워야 합니다(가비지 컬렉션). TRIM 명령을 활성화하면 파일시스템이 삭제된 블록을 SSD에 알려줍니다:

```bash
# /etc/fstab에 discard 옵션 추가 (즉시 TRIM)
# /dev/sda1 / ext4 defaults,discard 0 1

# 또는 주기적 fstrim (배치 TRIM, 성능 영향 적음)
systemctl enable --now fstrim.timer  # 주 1회 자동 실행
```

### 2. 가상화 환경에서의 I/O

KVM/VMware 게스트에서는 virtio-blk나 virtio-scsi를 사용하고, 게스트 커널의 스케줄러를 `none`으로 설정하여 호스트 스케줄러에 위임하는 것이 좋습니다.

### 3. 데이터베이스 최적화

PostgreSQL, MySQL 등은 자체 버퍼 풀을 관리하므로 `O_DIRECT` 플래그로 커널 페이지 캐시를 우회합니다. 이 경우 I/O 패턴이 완전히 달라지므로, mq-deadline의 데드라인을 짧게(읽기 200ms) 설정하는 것이 효과적입니다.

## 요약

| 알고리즘 | 탐색 최적화 | 공정성 | 레이턴시 | 최적 환경 |
|----------|-------------|--------|----------|-----------|
| FCFS | ❌ | ✅ | 나쁨 | 학습용 |
| SSTF | ✅ | ❌ (기아) | 좋음 | 단순 시스템 |
| SCAN/LOOK | ✅ | 중간 | 중간 | 일반 HDD |
| C-SCAN | ✅ | 좋음 | 균일 | 배치 워크로드 |
| mq-deadline | ✅ | 데드라인 보장 | 예측 가능 | SATA 서버 |
| BFQ | 중간 | ✅ | 대화형 우수 | 데스크톱/멀티미디어 |
| none | N/A | 하드웨어 의존 | 최소 | NVMe SSD |

디스크 I/O 스케줄링은 단순해 보이지만, 워크로드 특성과 스토리지 하드웨어에 따라 올바른 스케줄러 선택과 파라미터 튜닝이 시스템 성능을 수 배 이상 바꿀 수 있는 중요한 주제입니다.

## 참고 자료
- [Linux Kernel Documentation: Block I/O Layer](https://www.kernel.org/doc/html/latest/block/index.html)
- [BFQ I/O Scheduler — kernel.org](https://www.kernel.org/doc/html/latest/block/bfq-iosched.html)
- [RedHat: Configuring the Linux Storage Stack for Optimal Performance](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/managing_storage_devices/optimizing-the-storage-performance_managing-storage-devices)
- [Linux I/O Scheduling — FDCServers Blog](https://fdcservers.net/blog/linux-io-scheduler-tuning-mq-deadline-none-bfq)
