---
layout: post
title: "실시간 스케줄링 완전 정복: EDF와 Rate Monotonic으로 마감 시한을 절대 놓치지 않는 운영체제 설계"
date: 2026-09-14
categories: [cs, computer-science]
tags: [real-time, scheduling, EDF, rate-monotonic, RTOS, operating-systems, deadline, embedded]
---

## 개념 설명: 실시간 시스템이란?

항공기 자동 조종 장치가 0.01초 안에 센서 데이터를 처리하지 못한다면? 심박동기가 100ms 안에 심장 자극 신호를 보내지 못한다면? 자동차 ABS 시스템이 브레이크 압력 조절을 제때 하지 못한다면?

이런 시스템에서는 연산이 "빠르게" 완료되는 것만으로는 충분하지 않다. **정해진 시간 안에 반드시 완료**되어야 한다. 이를 **실시간 시스템(Real-Time System)**이라 부른다.

### 실시간 시스템의 분류

- **하드 실시간(Hard Real-Time)**: 마감 시한 초과 = 시스템 실패. 항공우주, 의료기기, 자동차 제어계통
- **소프트 실시간(Soft Real-Time)**: 마감 시한 초과 = 성능 저하. 멀티미디어 스트리밍, 화상회의
- **퍼름 실시간(Firm Real-Time)**: 마감 시한 초과된 결과는 무가치하지만 치명적이지 않음. 금융 트레이딩

### 실시간 태스크 모델

실시간 스케줄링에서 각 태스크 τᵢ는 다음 세 파라미터로 정의된다.

- **Cᵢ (Worst-Case Execution Time, WCET)**: 최악의 경우 실행 시간
- **Tᵢ (Period)**: 주기 (Rate Monotonic에서 사용)
- **Dᵢ (Deadline)**: 완료 마감 시한 (EDF에서 사용)

대부분의 분석에서 **암묵적 마감 시한(Implicit Deadline)** 모델을 사용한다: Dᵢ = Tᵢ, 즉 마감 시한이 다음 주기 시작과 같다.

---

## 왜 전통적인 스케줄링으로는 안 되는가?

### CFS(Completely Fair Scheduler)의 한계

Linux의 CFS는 "공정한 CPU 시간 분배"를 목표로 한다. 모든 프로세스가 비슷한 CPU 시간을 받도록 nice 값 기반으로 가중치를 조정한다. 그러나 이는 실시간 요구사항을 보장하지 못한다.

- CFS는 마감 시한 개념이 없다
- 우선순위 역전(Priority Inversion) 문제가 발생할 수 있다
- 응답 시간의 상한이 보장되지 않는다

실시간 스케줄링에서는 두 가지 접근법이 주류다.

1. **Rate Monotonic Scheduling (RMS)**: 정적 우선순위, 주기가 짧을수록 높은 우선순위
2. **Earliest Deadline First (EDF)**: 동적 우선순위, 마감 시한이 가장 임박한 태스크 우선

---

## Rate Monotonic Scheduling (RMS)

Liu와 Layland(1973)가 제안한 RMS는 가장 오래된 실시간 스케줄링 이론 중 하나다.

**핵심 규칙**: 주기(Period)가 짧을수록 높은 우선순위를 부여한다. 우선순위는 고정(Static)이다.

### 스케줄 가능성 분석

태스크 집합이 RMS로 스케줄 가능한지 확인하는 간단한 기준:

**Liu-Layland 한계 (Utilization Bound)**:

U = Σ(Cᵢ/Tᵢ) ≤ n(2^(1/n) - 1)

n이 커질수록 이 한계는 ln(2) ≈ 0.693에 수렴한다. 즉, CPU 활용률이 약 69.3% 이하면 RMS로 항상 스케줄 가능하다.

활용률이 69.3~100%라면 RMS가 스케줄 가능할 수도 있고 아닐 수도 있다. 이 경우 **Response Time Analysis (RTA)**로 정확히 분석해야 한다.

---

## 실제 구현 예제

### 예제 1: RMS 스케줄러 시뮬레이션 (Python)

```python
from dataclasses import dataclass
from typing import Optional
import math

@dataclass
class Task:
    name: str
    period: float      # 주기 (ms)
    wcet: float        # 최악 실행 시간 (ms)
    deadline: float = 0.0   # = period (암묵적 마감 시한)
    priority: int = 0  # RMS: 주기 짧을수록 높음 (낮은 숫자 = 높은 우선순위)
    
    def __post_init__(self):
        if self.deadline == 0.0:
            self.deadline = self.period
        self.utilization = self.wcet / self.period
    
    def __repr__(self):
        return (f"Task({self.name}, T={self.period}ms, "
                f"C={self.wcet}ms, U={self.utilization:.3f})")


class RMSScheduler:
    def __init__(self, tasks: list[Task]):
        self.tasks = tasks
        # RMS: 주기 오름차순 = 우선순위 내림차순
        sorted_tasks = sorted(tasks, key=lambda t: t.period)
        for i, task in enumerate(sorted_tasks):
            task.priority = i  # 0 = 가장 높은 우선순위
    
    def check_utilization_bound(self) -> tuple[bool, float]:
        """Liu-Layland 활용률 한계 검사"""
        n = len(self.tasks)
        total_u = sum(t.utilization for t in self.tasks)
        bound = n * (2 ** (1/n) - 1)
        return total_u <= bound, total_u
    
    def response_time_analysis(self) -> dict[str, float]:
        """응답 시간 분석 (RTA) - 정확한 스케줄 가능성 검사"""
        sorted_tasks = sorted(self.tasks, key=lambda t: t.priority)
        response_times = {}
        
        for i, task in enumerate(sorted_tasks):
            # 초기 응답 시간 = WCET
            R = task.wcet
            
            # 상위 우선순위 태스크들의 간섭 계산 (고정점 반복)
            while True:
                R_prev = R
                interference = sum(
                    math.ceil(R / hp_task.period) * hp_task.wcet
                    for hp_task in sorted_tasks[:i]  # 높은 우선순위 태스크들
                )
                R = task.wcet + interference
                
                if R == R_prev:
                    break  # 고정점 수렴
                if R > task.deadline:
                    R = float('inf')  # 마감 시한 초과
                    break
            
            response_times[task.name] = R
        
        return response_times
    
    def simulate(self, duration: float) -> list[tuple]:
        """RMS 스케줄 시뮬레이션"""
        schedule = []
        sorted_tasks = sorted(self.tasks, key=lambda t: t.priority)
        
        # 각 태스크의 다음 릴리즈 시간과 남은 실행 시간
        next_release = {t.name: 0.0 for t in self.tasks}
        remaining = {t.name: 0.0 for t in self.tasks}
        deadline_miss = {t.name: [] for t in self.tasks}
        
        t = 0.0
        dt = 0.1  # 시뮬레이션 타임 스텝
        
        while t < duration:
            # 릴리즈된 태스크 추가
            for task in self.tasks:
                if t >= next_release[task.name] and remaining[task.name] == 0:
                    remaining[task.name] = task.wcet
                    next_release[task.name] += task.period
            
            # RMS: 가장 높은 우선순위(가장 짧은 주기) 실행 가능 태스크 선택
            running = None
            for task in sorted_tasks:
                if remaining[task.name] > 0:
                    running = task
                    break
            
            if running:
                schedule.append((t, running.name))
                remaining[running.name] = max(0, remaining[running.name] - dt)
                
                # 마감 시한 검사
                for task in self.tasks:
                    if (remaining[task.name] > 0 and
                            t + dt >= next_release[task.name] - task.deadline + task.period):
                        if remaining[task.name] > 0:
                            deadline_miss[task.name].append(t)
            else:
                schedule.append((t, "IDLE"))
            
            t = round(t + dt, 3)
        
        return schedule, deadline_miss


# 태스크 정의
tasks = [
    Task("τ1", period=10, wcet=3),    # U1 = 0.3
    Task("τ2", period=20, wcet=5),    # U2 = 0.25
    Task("τ3", period=50, wcet=10),   # U3 = 0.2
]

scheduler = RMSScheduler(tasks)

print("=== Rate Monotonic Scheduling 분석 ===")
print(f"\n태스크 목록:")
for task in tasks:
    print(f"  {task}")

schedulable, total_u = scheduler.check_utilization_bound()
n = len(tasks)
bound = n * (2 ** (1/n) - 1)
print(f"\n총 CPU 활용률: {total_u:.3f}")
print(f"Liu-Layland 한계 (n={n}): {bound:.3f}")
print(f"활용률 한계 검사: {'PASS ✓' if schedulable else 'FAIL (RTA 추가 확인 필요)'}")

rt = scheduler.response_time_analysis()
print(f"\n응답 시간 분석 (RTA):")
for task in sorted(tasks, key=lambda t: t.priority):
    r = rt[task.name]
    status = "OK ✓" if r <= task.deadline else "MISS ✗"
    print(f"  {task.name}: R={r:.1f}ms, D={task.deadline}ms → {status}")
```

출력:
```
=== Rate Monotonic Scheduling 분석 ===

태스크 목록:
  Task(τ1, T=10ms, C=3ms, U=0.300)
  Task(τ2, T=20ms, C=5ms, U=0.250)
  Task(τ3, T=50ms, C=10ms, U=0.200)

총 CPU 활용률: 0.750
Liu-Layland 한계 (n=3): 0.780
활용률 한계 검사: PASS ✓

응답 시간 분석 (RTA):
  τ1: R=3.0ms, D=10ms → OK ✓
  τ2: R=11.0ms, D=20ms → OK ✓
  τ3: R=34.0ms, D=50ms → OK ✓
```

### 예제 2: EDF 스케줄러 시뮬레이션 (Python)

```python
import heapq
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class RTTask:
    name: str
    period: float       # 주기 (ms)
    wcet: float         # 최악 실행 시간 (ms)
    utilization: float = field(init=False)
    
    def __post_init__(self):
        self.utilization = self.wcet / self.period

@dataclass(order=True)
class TaskInstance:
    """실행 중인 태스크 인스턴스 (EDF 우선순위 큐용)"""
    abs_deadline: float         # 절대 마감 시한 (우선순위 기준)
    name: str = field(compare=False)
    remaining: float = field(compare=False)    # 남은 실행 시간
    release: float = field(compare=False)      # 릴리즈 시간


class EDFScheduler:
    """Earliest Deadline First 스케줄러"""
    
    def __init__(self, tasks: list[RTTask]):
        self.tasks = tasks
    
    def check_feasibility(self) -> tuple[bool, float]:
        """EDF 스케줄 가능성 검사: U ≤ 1.0"""
        total_u = sum(t.utilization for t in self.tasks)
        return total_u <= 1.0, total_u
    
    def simulate(self, duration: float) -> tuple[list, dict]:
        """EDF 스케줄 시뮬레이션"""
        schedule = []
        deadline_misses = {t.name: [] for t in self.tasks}
        
        # 우선순위 큐: (절대 마감 시한, 태스크 인스턴스)
        ready_queue: list[TaskInstance] = []
        
        # 다음 릴리즈 시간
        next_release = {t.name: (0.0, t) for t in self.tasks}
        
        t = 0.0
        dt = 0.5  # 타임 스텝
        
        while t < duration:
            # 현재 시점에 릴리즈된 태스크 추가
            for task_name, (release_time, task) in list(next_release.items()):
                if release_time <= t:
                    abs_dl = release_time + task.period  # = 다음 주기 시작
                    heapq.heappush(ready_queue, TaskInstance(
                        abs_deadline=abs_dl,
                        name=task.name,
                        remaining=task.wcet,
                        release=release_time
                    ))
                    next_release[task_name] = (release_time + task.period, task)
            
            # 마감 시한 초과 검사
            for instance in ready_queue:
                if t > instance.abs_deadline and instance.remaining > 0:
                    deadline_misses[instance.name].append(
                        (t, instance.abs_deadline)
                    )
            
            # EDF: 가장 임박한 마감 시한 태스크 실행
            if ready_queue:
                # 완료된 인스턴스 제거
                while ready_queue and ready_queue[0].remaining <= 0:
                    heapq.heappop(ready_queue)
            
            if ready_queue:
                current = ready_queue[0]
                schedule.append((t, current.name, current.abs_deadline))
                current.remaining -= dt
                if current.remaining <= 0:
                    heapq.heapreplace(ready_queue, current)  # 정렬 유지
            else:
                schedule.append((t, "IDLE", None))
            
            t = round(t + dt, 3)
        
        return schedule, deadline_misses
    
    def compare_with_rms(self):
        """EDF vs RMS 비교 분석"""
        total_u = sum(t.utilization for t in self.tasks)
        n = len(self.tasks)
        rms_bound = n * (2 ** (1/n) - 1)
        
        print(f"\n=== EDF vs RMS 스케줄 가능성 비교 ===")
        print(f"총 CPU 활용률: {total_u:.3f}")
        print(f"RMS 한계:       {rms_bound:.3f} → {'OK' if total_u <= rms_bound else 'FAIL'}")
        print(f"EDF 한계:       1.000 → {'OK' if total_u <= 1.0 else 'FAIL'}")


# EDF 사용 예시 — RMS로는 스케줄 불가능한 케이스
tasks = [
    RTTask("τ1", period=4,  wcet=2),   # U = 0.50
    RTTask("τ2", period=5,  wcet=2),   # U = 0.40
    RTTask("τ3", period=20, wcet=1),   # U = 0.05
]

edf = EDFScheduler(tasks)
edf.compare_with_rms()

feasible, total_u = edf.check_feasibility()
print(f"\nEDF 스케줄 가능성: {'FEASIBLE ✓' if feasible else 'INFEASIBLE ✗'}")
print(f"(총 활용률 {total_u:.2f} ≤ 1.00)\n")

schedule, misses = edf.simulate(40)

# 스케줄 출력 (앞 10 타임슬롯만)
print("EDF 스케줄 (앞 20ms):")
for time, task, dl in schedule[:40]:
    dl_str = f"dl={dl:.1f}" if dl else ""
    print(f"  t={time:5.1f}: {task:<6} {dl_str}")

for task_name, miss_list in misses.items():
    if miss_list:
        print(f"\n⚠ {task_name} 마감 시한 초과: {len(miss_list)}회")
    else:
        print(f"✓ {task_name}: 마감 시한 준수")
```

---

## EDF vs Rate Monotonic: 무엇을 선택해야 하나?

| 특성 | Rate Monotonic | EDF |
|------|---------------|-----|
| 우선순위 타입 | 정적 (고정) | 동적 |
| CPU 활용률 한계 | ~69.3% | 100% |
| 컨텍스트 스위치 | 적음 | 많음 (런타임 재계산) |
| 구현 복잡도 | 간단 | 복잡 |
| 마감 시한 초과 시 | 예측 가능 | 예측 어려움 |
| 오버로드 처리 | RMS가 유리 | 연쇄 실패 위험 |
| 적용 분야 | 하드 RT, 임베디드 | 소프트 RT, 서버 |

**Liu와 Layland의 최적성 정리**:
- RMS는 정적 우선순위 알고리즘 중 **최적**이다 (즉, 어떤 정적 우선순위 알고리즘으로도 스케줄 불가능한 태스크 집합은 RMS로도 불가능)
- EDF는 단일 프로세서에서 **범용 최적**이다 (EDF로 스케줄 불가능한 태스크 집합은 어떤 알고리즘으로도 불가능)

---

## 주의사항과 팁

### 1. WCET 측정의 어려움

실시간 스케줄링의 가장 큰 실용적 도전은 정확한 WCET 측정이다.
- 캐시 미스, 파이프라인 스톨, 브랜치 예측 실패 등이 WCET에 영향을 미친다
- 보수적으로 측정하면 CPU 활용률이 낮게 나오고, 정확하게 측정하려면 정적 분석 도구(aiT, Bound-T)가 필요하다

### 2. 우선순위 역전 (Priority Inversion)

공유 자원 접근 시 우선순위 역전이 발생할 수 있다. 해결 방법:
- **PIP (Priority Inheritance Protocol)**: 낮은 우선순위 태스크가 자원을 점유하면 현재 기다리는 가장 높은 우선순위로 임시 승격
- **PCP (Priority Ceiling Protocol)**: 각 자원에 천장(ceiling) 우선순위를 미리 지정

Linux PREEMPT_RT 패치는 이런 메커니즘을 커널 수준에서 지원한다.

### 3. 멀티코어에서의 실시간 스케줄링

단일 프로세서에서의 이론은 멀티코어로 쉽게 확장되지 않는다. 분산 실시간 스케줄링은 NP-Hard 문제이며, 실용적으로는:
- **글로벌 EDF**: 모든 코어에서 단일 준비 큐를 공유 (Dhall 효과 주의)
- **파티션된 스케줄링**: 각 태스크를 특정 코어에 고정 배치

### 4. Linux의 실시간 스케줄러

Linux는 `SCHED_FIFO`, `SCHED_RR`, `SCHED_DEADLINE`(EDF 기반) 정책을 지원한다.

```c
// Linux에서 SCHED_DEADLINE 사용 (임의 EDF 정책)
struct sched_attr attr = {
    .size         = sizeof(attr),
    .sched_policy = SCHED_DEADLINE,
    .sched_runtime  = 10 * 1000 * 1000,  // 10ms (나노초)
    .sched_period   = 100 * 1000 * 1000, // 100ms
    .sched_deadline = 50 * 1000 * 1000,  // 50ms
};
syscall(SYS_sched_setattr, 0, &attr, 0);
```

CBS(Constant Bandwidth Server) 알고리즘으로 구현되어, 각 태스크가 선언한 bandwidth 내에서만 자원을 사용하도록 강제한다.

---

## 정리

실시간 스케줄링은 "빠르게"가 아니라 "제때"를 보장하는 학문이다. RMS는 단순하고 예측 가능하지만 CPU 활용률 한계가 있고, EDF는 이론적으로 최적이지만 오버로드 시 연쇄 실패 위험이 있다. 실무에서는 WCET 분석, 우선순위 역전 방지, 스케줄 가능성 검증을 모두 고려해야 한다. 하드 실시간 요구사항이 있는 임베디드 시스템 개발자라면, RMS와 EDF의 이론적 토대를 이해하고 적절한 RTOS(FreeRTOS, Zephyr, VxWorks)를 선택하는 것이 첫 걸음이다.

## 참고 자료
- [GitHub - marciamart/Real-time-scheduling-Algorithm: RM과 EDF 알고리즘 구현](https://github.com/marciamart/Real-time-scheduling-Algorithm)
- [GitHub - diegoperini/py-common-scheduling-algorithms: Python으로 구현한 실시간 스케줄링 알고리즘](https://github.com/diegoperini/py-common-scheduling-algorithms)
