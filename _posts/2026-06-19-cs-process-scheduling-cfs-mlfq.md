---
layout: post
title: "프로세스 스케줄링 심화: CFS와 MLFQ로 이해하는 공정한 CPU 분배"
date: 2026-06-19
categories: [cs, computer-science]
tags: [process-scheduling, CFS, MLFQ, operating-system, linux-kernel, vruntime, red-black-tree]
---

## 프로세스 스케줄러란?

운영체제의 핵심 임무 중 하나는 수백~수천 개의 프로세스가 단 몇 개의 CPU 코어를 나누어 쓸 수 있도록 조율하는 것입니다. 이 역할을 담당하는 것이 **프로세스 스케줄러(Process Scheduler)**입니다.

스케줄러는 매우 짧은 주기(수 밀리초~수십 마이크로초)마다 "다음에 어떤 프로세스를 실행할 것인가?"를 결정합니다. 잘못된 스케줄링은 다음 문제를 유발합니다.

- **기아(Starvation)**: 일부 프로세스가 CPU를 영원히 배정받지 못함
- **응답 지연**: 사용자 인터랙션에 반응이 느림
- **처리량 저하**: 전체 작업 완료 시간이 길어짐

현대 OS가 어떻게 이 문제를 해결하는지, 대표적인 두 알고리즘인 **MLFQ**와 **CFS**를 중심으로 살펴봅니다.

---

## MLFQ: Multi-Level Feedback Queue

MLFQ는 1962년 Fernando J. Corbató가 Compatible Time-Sharing System(CTSS)에서 처음 제안한 알고리즘으로, 현재도 많은 OS의 기반 아이디어로 쓰입니다.

### 핵심 아이디어

MLFQ는 여러 개의 우선순위 큐를 두고, 프로세스의 **과거 행동**에 따라 동적으로 우선순위를 조정합니다.

**기본 규칙:**

1. 우선순위가 높은 큐의 프로세스를 먼저 실행한다
2. 동일 우선순위 내에서는 Round-Robin으로 실행한다
3. 새 프로세스는 최고 우선순위 큐에 배치한다
4. 프로세스가 할당된 타임 퀀텀(time quantum)을 다 쓰면 한 단계 낮은 큐로 이동한다 (CPU 집중 프로세스 판단)
5. 타임 퀀텀 내에 자발적으로 CPU를 반납하면(I/O 대기 등) 현재 우선순위를 유지한다 (I/O 집중 프로세스 = 대화형)
6. 주기적(aging)으로 모든 프로세스를 최고 우선순위 큐로 올려 기아를 방지한다

### MLFQ가 해결하는 문제

| 문제 | MLFQ의 해법 |
|---|---|
| CPU 집중 vs I/O 집중 구분 | 행동 관찰로 자동 분류 |
| 대화형 프로세스 응답성 | I/O 집중 = 높은 우선순위 유지 |
| 기아(Starvation) | 주기적 aging으로 모두 최고 큐로 승격 |

---

## CFS: Completely Fair Scheduler

CFS는 2007년 Ingo Molnár가 개발해 Linux 커널 2.6.23에 도입된 스케줄러입니다. 현재 Linux의 기본 스케줄러로, 거의 모든 Android 기기와 서버에서 동작합니다(커널 6.6부터 EEVDF로 점진적 대체 중).

### 핵심 아이디어: 가상 런타임(vruntime)

CFS의 목표는 "모든 프로세스가 이상적인 멀티태스킹 CPU에서 동시에 실행되는 것처럼" 행동하는 것입니다.

각 프로세스는 **vruntime(virtual runtime)**을 가집니다. vruntime은 실제 실행 시간을 **프로세스 가중치로 나눈 정규화된 시간**입니다.

$$vruntime_i += \frac{실제\_실행\_시간 \times nice\_0\_weight}{weight_i}$$

- `nice` 값이 낮을수록(우선순위 높을수록) weight가 크고, 따라서 vruntime이 천천히 증가합니다
- CFS는 항상 **vruntime이 가장 작은 프로세스**(가장 덜 실행된 프로세스)를 다음으로 선택합니다

### 레드-블랙 트리로 O(log n) 스케줄링

CFS는 모든 실행 가능한 프로세스를 **vruntime을 키로 하는 레드-블랙 트리**에 저장합니다. 트리의 가장 왼쪽 노드(min-vruntime)가 다음 실행 프로세스입니다.

- **다음 프로세스 선택**: O(1) — 캐시된 leftmost 포인터 사용
- **삽입/삭제**: O(log n)

---

## 실제 구현 예제

### 예제 1: Python으로 MLFQ 시뮬레이터 구현

```python
from collections import deque
from dataclasses import dataclass, field


@dataclass
class Process:
    pid: int
    name: str
    burst_time: int       # 총 CPU 필요 시간 (ms)
    remaining: int = 0
    priority_level: int = 0  # 현재 큐 레벨 (0=최고)
    total_wait: int = 0

    def __post_init__(self):
        self.remaining = self.burst_time


class MLFQ:
    def __init__(self, num_levels: int = 3, aging_period: int = 50):
        # 각 레벨별 타임 퀀텀: 레벨이 낮을수록 더 긴 퀀텀
        self.quantums = [4, 8, 16][:num_levels]
        self.queues: list[deque[Process]] = [deque() for _ in range(num_levels)]
        self.aging_period = aging_period
        self.time = 0

    def add_process(self, proc: Process):
        proc.priority_level = 0
        self.queues[0].append(proc)
        print(f"[t={self.time:3d}] 프로세스 {proc.name}(PID={proc.pid}) 도착 → Q0")

    def _age_processes(self):
        """기아 방지: 하위 큐 프로세스를 상위 큐로 승격"""
        for level in range(1, len(self.queues)):
            promoted = []
            for proc in self.queues[level]:
                promoted.append(proc)
            for proc in promoted:
                self.queues[level].remove(proc)
                proc.priority_level = 0
                self.queues[0].appendleft(proc)
                print(f"[t={self.time:3d}] [AGING] {proc.name} Q{level} → Q0 승격")

    def run(self, total_time: int = 100):
        last_age = 0

        while self.time < total_time:
            # Aging 주기 체크
            if self.time - last_age >= self.aging_period:
                self._age_processes()
                last_age = self.time

            # 실행할 프로세스 탐색 (우선순위 높은 큐부터)
            chosen = None
            chosen_level = -1
            for level, q in enumerate(self.queues):
                if q:
                    chosen = q.popleft()
                    chosen_level = level
                    break

            if chosen is None:
                self.time += 1  # idle
                continue

            quantum = self.quantums[chosen_level]
            run_time = min(quantum, chosen.remaining)

            print(f"[t={self.time:3d}] 실행: {chosen.name} "
                  f"(Q{chosen_level}, quantum={quantum}, run={run_time}ms, "
                  f"remaining={chosen.remaining - run_time}ms)")

            self.time += run_time
            chosen.remaining -= run_time

            if chosen.remaining <= 0:
                print(f"[t={self.time:3d}] 완료: {chosen.name} (총 {chosen.burst_time}ms)")
            else:
                # 타임 퀀텀 소진 → 하위 큐로 강등
                next_level = min(chosen_level + 1, len(self.queues) - 1)
                chosen.priority_level = next_level
                self.queues[next_level].append(chosen)
                print(f"[t={self.time:3d}] {chosen.name} Q{chosen_level} → Q{next_level} 강등")


# --- 실행 ---
if __name__ == "__main__":
    mlfq = MLFQ(num_levels=3, aging_period=30)

    # 대화형(짧은 burst) 프로세스들
    mlfq.add_process(Process(1, "Browser", burst_time=6))
    mlfq.add_process(Process(2, "Editor", burst_time=4))
    # CPU 집중 프로세스
    mlfq.add_process(Process(3, "Compiler", burst_time=40))

    mlfq.run(total_time=80)
```

---

### 예제 2: Python으로 CFS vruntime 시뮬레이터 구현

```python
import heapq
from dataclasses import dataclass, field

# nice 값에 따른 가중치 (Linux 커널 실제 값에서 발췌, 간소화)
NICE_TO_WEIGHT = {
    -20: 88761, -10: 9548, 0: 1024, 5: 335, 10: 110, 19: 15
}


@dataclass(order=True)
class Task:
    vruntime: float          # 정렬 기준
    pid: int = field(compare=False)
    name: str = field(compare=False)
    nice: int = field(compare=False, default=0)
    remaining_ms: int = field(compare=False, default=100)

    @property
    def weight(self) -> int:
        # 가장 가까운 nice 값의 weight 선택
        return NICE_TO_WEIGHT.get(self.nice, 1024)


class CFSScheduler:
    def __init__(self, tick_ms: int = 4):
        self.run_queue: list[Task] = []  # min-heap (vruntime 기준)
        self.tick_ms = tick_ms           # 기본 타임슬라이스
        self.min_vruntime = 0.0
        self.clock = 0

    def add_task(self, task: Task):
        # 새 태스크는 현재 min_vruntime부터 시작 (불이익 없음)
        task.vruntime = self.min_vruntime
        heapq.heappush(self.run_queue, task)
        print(f"[t={self.clock:4d}ms] 추가: {task.name}(nice={task.nice}, "
              f"weight={task.weight}, vruntime={task.vruntime:.1f})")

    def _calc_timeslice(self, task: Task) -> int:
        """가중치에 비례한 타임슬라이스 계산"""
        total_weight = sum(t.weight for t in self.run_queue) + task.weight
        return max(1, int(self.tick_ms * task.weight / total_weight * len(self.run_queue)))

    def run(self, duration_ms: int = 200):
        while self.clock < duration_ms and self.run_queue:
            task = heapq.heappop(self.run_queue)
            timeslice = self._calc_timeslice(task)
            actual_run = min(timeslice, task.remaining_ms)

            # vruntime 증가: 실제 실행시간 * (기준 weight / 태스크 weight)
            delta_vruntime = actual_run * (NICE_TO_WEIGHT[0] / task.weight)
            task.vruntime += delta_vruntime
            task.remaining_ms -= actual_run
            self.clock += actual_run

            print(f"[t={self.clock:4d}ms] 실행: {task.name:12s} "
                  f"slice={actual_run}ms, vruntime={task.vruntime:.1f}, "
                  f"remaining={task.remaining_ms}ms")

            if task.remaining_ms > 0:
                self.min_vruntime = min(t.vruntime for t in self.run_queue) \
                    if self.run_queue else task.vruntime
                heapq.heappush(self.run_queue, task)
            else:
                print(f"           ✓ {task.name} 완료")


if __name__ == "__main__":
    cfs = CFSScheduler(tick_ms=4)

    # 일반 우선순위 태스크 (nice=0)
    cfs.add_task(Task(vruntime=0, pid=1, name="WebServer",   nice=0,  remaining_ms=80))
    # 높은 우선순위 태스크 (nice=-10)
    cfs.add_task(Task(vruntime=0, pid=2, name="RealTimeTask", nice=-10, remaining_ms=40))
    # 낮은 우선순위 태스크 (nice=10)
    cfs.add_task(Task(vruntime=0, pid=3, name="Backup",       nice=10,  remaining_ms=120))

    print("\n=== CFS 스케줄링 시뮬레이션 ===\n")
    cfs.run(duration_ms=300)
```

---

## CFS vs MLFQ 비교

| 항목 | MLFQ | CFS |
|---|---|---|
| 설계 목표 | 대화형 응답성 + CPU 효율 | 완전한 공정성(Fairness) |
| 우선순위 조정 | 동적(행동 기반) | 가중치(nice 값) 기반 |
| 기아 방지 | Aging | vruntime 추적으로 자연 방지 |
| 구현 복잡도 | 중간 | 높음 (레드-블랙 트리) |
| 실제 사용 | Windows, 구형 Unix | Linux (Android 포함) |

---

## 주의사항과 팁

### 1. nice 값은 상대적이다

Linux에서 `nice -n -5 ./myapp`으로 우선순위를 높이더라도, 다른 모든 프로세스가 같은 nice 값이면 효과가 없습니다. CFS는 **상대적인 가중치 비율**로 동작합니다.

### 2. 스케줄링 정책 선택

Linux는 `SCHED_NORMAL`(CFS) 외에도 `SCHED_FIFO`, `SCHED_RR`(실시간)을 제공합니다. 실시간성이 중요한 오디오/영상 처리에는 `SCHED_FIFO`를 검토하세요. 단, 잘못 사용하면 시스템 전체를 멈출 수 있습니다.

### 3. CPU 어피니티(Affinity) 설정

NUMA 아키텍처에서 프로세스를 특정 CPU 코어에 고정하면 캐시 히트율이 올라갑니다. `taskset` 명령어나 `pthread_setaffinity_np`를 활용하세요.

### 4. 커널 6.6 이후: EEVDF 스케줄러

Linux 6.6부터는 CFS를 점진적으로 대체하는 **EEVDF(Earliest Eligible Virtual Deadline First)** 스케줄러가 도입되었습니다. 레이턴시 민감 워크로드에서 더 나은 응답성을 제공합니다.

---

## 참고 자료

- [CFS Scheduler — The Linux Kernel Documentation](https://docs.kernel.org/scheduler/sched-design-CFS.html)
- [Completely Fair Scheduler — Wikipedia](https://en.wikipedia.org/wiki/Completely_Fair_Scheduler)
- [Inside the Linux 2.6 Completely Fair Scheduler — IBM Developer](https://developer.ibm.com/tutorials/l-completely-fair-scheduler/)
- [Multi-Level Feedback Queues — USFCA CS326 Lecture](https://www.cs.usfca.edu/~mmalensek/cs326/schedule/lectures/326-L11.pdf)
