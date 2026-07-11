---
layout: post
title: "데드락(Deadlock) 완전 정복: 탐지·예방·회피와 Banker's Algorithm까지"
date: 2026-07-11
categories: [cs, computer-science]
tags: [deadlock, operating-systems, concurrency, banker-algorithm, resource-allocation, mutex, semaphore]
---

## 데드락이란?

데드락(Deadlock)은 두 개 이상의 프로세스가 서로 상대방이 보유한 자원을 기다리며 **영원히 진행되지 못하는 상태**다. 1971년 E.W. Dijkstra가 처음 공식화한 이 문제는 오늘날 운영체제, 데이터베이스, 분산 시스템 등 모든 동시성 환경에서 여전히 중요한 도전 과제다.

교통 상황으로 비유하자면, 교차로에서 네 방향의 차량이 서로 진행하지 못하고 막혀 있는 상태다. 각 차량은 앞에 있는 차가 빠져나가길 기다리지만, 앞 차도 마찬가지로 다른 차를 기다리고 있어 어느 차도 움직일 수 없다.

---

## 데드락의 4가지 필요 조건

데드락이 발생하려면 다음 네 가지 조건이 **동시에** 성립해야 한다. Coffman(1971)이 제시한 이 조건들 중 하나라도 깨면 데드락은 발생하지 않는다.

### 1. 상호 배제 (Mutual Exclusion)
자원은 한 번에 하나의 프로세스만 사용할 수 있다. 뮤텍스(mutex), 파일 잠금 등이 여기에 해당한다.

### 2. 점유와 대기 (Hold and Wait)
프로세스가 자원을 보유하면서 다른 자원을 기다린다. A가 mutex1을 잠그고 mutex2를 기다리는 상황이다.

### 3. 비선점 (No Preemption)
자원은 프로세스가 자발적으로 해제하기 전까지 강제로 빼앗을 수 없다. 이미 잠긴 뮤텍스를 OS가 강제로 해제하지 않는 것이 전형적인 예다.

### 4. 순환 대기 (Circular Wait)
P1 → P2 → P3 → P1과 같이 프로세스들이 원형으로 서로의 자원을 기다린다.

---

## 왜 데드락이 위험한가?

데드락은 세 가지 이유로 특히 위험하다:

1. **무증상 잠복**: 데드락은 평상시에는 발생하지 않다가 특정 타이밍 조건이 맞을 때만 재현된다. 개발 환경에서는 절대 발생하지 않지만 프로덕션에서 갑자기 나타나는 경우가 흔하다.

2. **파급 효과**: 하나의 데드락이 연쇄적으로 다른 스레드/프로세스를 블록시킬 수 있다.

3. **무한 대기**: 타임아웃이 없으면 시스템은 영원히 복구되지 않는다.

---

## 데드락 발생 시뮬레이션

```python
import threading
import time
from typing import Optional

def simulate_deadlock():
    """
    두 스레드가 서로 다른 순서로 두 개의 락을 획득하려는 데드락 상황 시뮬레이션
    실제로 실행하면 영원히 블록됨 — 교육용 예제
    """
    lock_a = threading.Lock()
    lock_b = threading.Lock()
    deadlock_detected = threading.Event()

    def thread_1():
        print("[T1] lock_a 획득 시도...")
        with lock_a:
            print("[T1] lock_a 획득!")
            time.sleep(0.1)  # 타이밍 조건 유도
            print("[T1] lock_b 획득 시도...")
            if lock_b.acquire(timeout=1.0):  # 타임아웃으로 데드락 방지
                print("[T1] lock_b 획득!")
                lock_b.release()
            else:
                print("[T1] lock_b 타임아웃 — 데드락 감지!")
                deadlock_detected.set()

    def thread_2():
        print("[T2] lock_b 획득 시도...")
        with lock_b:
            print("[T2] lock_b 획득!")
            time.sleep(0.1)
            print("[T2] lock_a 획득 시도...")
            if lock_a.acquire(timeout=1.0):  # 타임아웃으로 데드락 방지
                print("[T2] lock_a 획득!")
                lock_a.release()
            else:
                print("[T2] lock_a 타임아웃 — 데드락 감지!")
                deadlock_detected.set()

    t1 = threading.Thread(target=thread_1)
    t2 = threading.Thread(target=thread_2)
    t1.start()
    t2.start()
    t1.join(timeout=3.0)
    t2.join(timeout=3.0)

    if deadlock_detected.is_set():
        print("\n⚠️  데드락이 발생했습니다!")
        print("해결책: 두 스레드가 락을 동일한 순서로 획득해야 합니다.")
    else:
        print("\n✅ 정상 완료")


def demonstrate_deadlock_prevention():
    """
    락 순서 일관성으로 데드락 예방
    두 스레드가 항상 lock_a → lock_b 순서로 획득
    """
    lock_a = threading.Lock()
    lock_b = threading.Lock()

    def safe_thread_1():
        with lock_a:          # 항상 A 먼저
            time.sleep(0.1)
            with lock_b:      # 그 다음 B
                print("[T1] 두 락 모두 획득 완료")

    def safe_thread_2():
        with lock_a:          # T2도 항상 A 먼저
            time.sleep(0.05)
            with lock_b:      # 그 다음 B
                print("[T2] 두 락 모두 획득 완료")

    t1 = threading.Thread(target=safe_thread_1)
    t2 = threading.Thread(target=safe_thread_2)
    t1.start()
    t2.start()
    t1.join()
    t2.join()
    print("데드락 없이 완료!")


if __name__ == "__main__":
    print("=== 데드락 발생 시뮬레이션 ===")
    simulate_deadlock()
    print("\n=== 데드락 예방 (락 순서 일관성) ===")
    demonstrate_deadlock_prevention()
```

---

## 데드락 처리 전략

### 전략 1: 데드락 예방 (Deadlock Prevention)

4가지 필요 조건 중 하나를 원천적으로 차단한다.

**순환 대기 제거** — 가장 실용적인 방법. 모든 자원에 전역 번호를 부여하고, 프로세스는 항상 번호 오름차순으로 자원을 요청하도록 강제한다.

```python
class Resource:
    def __init__(self, name: str, order: int):
        self.name = name
        self.order = order  # 전역 자원 순서
        self._lock = threading.Lock()

    def acquire(self):
        self._lock.acquire()

    def release(self):
        self._lock.release()

class OrderedLockAcquisition:
    """순서 기반 락 획득으로 순환 대기 제거"""

    def __init__(self, resources: list):
        # 자원을 order 기준으로 정렬하여 항상 같은 순서로 획득
        self.resources = sorted(resources, key=lambda r: r.order)

    def __enter__(self):
        for r in self.resources:
            r.acquire()
        return self

    def __exit__(self, *args):
        # 역순으로 해제
        for r in reversed(self.resources):
            r.release()

# 사용 예
db_lock = Resource("database", order=1)
cache_lock = Resource("cache", order=2)
file_lock = Resource("file", order=3)

# 어느 스레드에서든 order 오름차순으로 획득되므로 데드락 불가
def thread_work(resources_needed):
    with OrderedLockAcquisition(resources_needed):
        print(f"자원 {[r.name for r in resources_needed]} 사용 중")
        time.sleep(0.01)
```

**점유 대기 제거** — 필요한 모든 자원을 한꺼번에 요청하거나, 실패하면 모두 해제하고 재시도한다.

```python
def try_acquire_all(locks: list, timeout: float = 1.0) -> bool:
    """모든 락을 원자적으로 획득 시도 (일부 실패 시 전체 해제)"""
    acquired = []
    for lock in locks:
        if lock.acquire(timeout=timeout):
            acquired.append(lock)
        else:
            # 일부만 획득 성공한 경우 모두 해제
            for l in acquired:
                l.release()
            return False
    return True
```

### 전략 2: 데드락 회피 (Deadlock Avoidance) — Banker's Algorithm

자원 요청이 들어올 때마다 **시스템이 안전 상태(safe state)인지 검사**하고, 안전하지 않으면 요청을 거부한다. Dijkstra의 은행가 알고리즘이 대표적이다.

**안전 상태**: 모든 프로세스가 최대 필요 자원을 요청하더라도 순차적으로 완료 가능한 순서(safe sequence)가 존재하는 상태.

```python
from typing import List, Optional, Tuple

class BankersAlgorithm:
    """
    Banker's Algorithm 구현
    n: 프로세스 수
    m: 자원 종류 수
    """

    def __init__(self, n: int, m: int,
                 max_need: List[List[int]],
                 allocation: List[List[int]],
                 available: List[int]):
        self.n = n
        self.m = m
        self.max_need = max_need       # max_need[i][j]: 프로세스 i의 자원 j 최대 요구량
        self.allocation = allocation   # allocation[i][j]: 프로세스 i에 할당된 자원 j
        self.available = available[:]  # 현재 가용 자원

        # need[i][j] = max_need[i][j] - allocation[i][j]
        self.need = [
            [max_need[i][j] - allocation[i][j] for j in range(m)]
            for i in range(n)
        ]

    def is_safe(self) -> Tuple[bool, List[int]]:
        """안전 상태 검사 — 안전 순서열 반환"""
        work = self.available[:]
        finish = [False] * self.n
        safe_sequence = []

        for _ in range(self.n):
            found = False
            for i in range(self.n):
                if not finish[i] and all(self.need[i][j] <= work[j] for j in range(self.m)):
                    # 프로세스 i가 완료 가능: 할당 자원 반환
                    work = [work[j] + self.allocation[i][j] for j in range(self.m)]
                    finish[i] = True
                    safe_sequence.append(i)
                    found = True
                    break

            if not found:
                break

        is_safe_state = all(finish)
        return is_safe_state, safe_sequence

    def request_resources(self, process_id: int, request: List[int]) -> Tuple[bool, str]:
        """
        프로세스가 자원을 요청할 때 허용 여부 결정
        반환: (허용 여부, 메시지)
        """
        pid = process_id

        # 조건 1: request <= need
        if any(request[j] > self.need[pid][j] for j in range(self.m)):
            return False, f"오류: 최대 필요량({self.need[pid]})을 초과하는 요청({request})"

        # 조건 2: request <= available
        if any(request[j] > self.available[j] for j in range(self.m)):
            return False, f"자원 부족: 가용({self.available}) < 요청({request}), 대기 필요"

        # 가상으로 자원 할당 후 안전 상태 검사
        self.available = [self.available[j] - request[j] for j in range(self.m)]
        self.allocation[pid] = [self.allocation[pid][j] + request[j] for j in range(self.m)]
        self.need[pid] = [self.need[pid][j] - request[j] for j in range(self.m)]

        safe, sequence = self.is_safe()

        if safe:
            return True, f"요청 승인. 안전 순서열: P{sequence}"
        else:
            # 롤백
            self.available = [self.available[j] + request[j] for j in range(self.m)]
            self.allocation[pid] = [self.allocation[pid][j] - request[j] for j in range(self.m)]
            self.need[pid] = [self.need[pid][j] + request[j] for j in range(self.m)]
            return False, "요청 거부: 할당 시 안전하지 않은 상태가 됨"


# 예제: 5개 프로세스, 3종 자원 (A, B, C)
n_processes = 5
n_resources = 3

# 각 프로세스의 최대 자원 요구량 [A, B, C]
max_need = [
    [7, 5, 3],  # P0
    [3, 2, 2],  # P1
    [9, 0, 2],  # P2
    [2, 2, 2],  # P3
    [4, 3, 3],  # P4
]

# 현재 할당된 자원
allocation = [
    [0, 1, 0],  # P0
    [2, 0, 0],  # P1
    [3, 0, 2],  # P2
    [2, 1, 1],  # P3
    [0, 0, 2],  # P4
]

# 가용 자원 [A=3, B=3, C=2]
available = [3, 3, 2]

banker = BankersAlgorithm(n_processes, n_resources, max_need, allocation, available)

safe, seq = banker.is_safe()
print(f"현재 상태 안전 여부: {safe}")
print(f"안전 순서열: P{seq}")

# P1이 자원 [1, 0, 2] 요청
ok, msg = banker.request_resources(1, [1, 0, 2])
print(f"\nP1이 [1,0,2] 요청: {msg}")
```

### 전략 3: 데드락 탐지 및 복구 (Detection and Recovery)

데드락 발생을 허용하되 주기적으로 자원 할당 그래프를 검사하고, 발견 시 복구한다.

```python
class DeadlockDetector:
    """
    자원 할당 그래프에서 데드락 탐지
    Banker's Algorithm의 탐지 변형: need 대신 request 사용
    """

    def __init__(self, n: int, m: int,
                 allocation: List[List[int]],
                 request: List[List[int]],
                 available: List[int]):
        self.n = n
        self.m = m
        self.allocation = allocation
        self.request = request   # 현재 대기 중인 자원 요청
        self.available = available[:]

    def detect(self) -> Tuple[bool, List[int]]:
        """
        데드락 탐지 알고리즘
        반환: (데드락 여부, 데드락에 걸린 프로세스 목록)
        """
        work = self.available[:]
        finish = [all(self.allocation[i][j] == 0 for j in range(self.m)) for i in range(self.n)]

        while True:
            found = False
            for i in range(self.n):
                if not finish[i] and all(self.request[i][j] <= work[j] for j in range(self.m)):
                    work = [work[j] + self.allocation[i][j] for j in range(self.m)]
                    finish[i] = True
                    found = True

            if not found:
                break

        deadlocked = [i for i in range(self.n) if not finish[i]]
        return len(deadlocked) > 0, deadlocked

    def recover_by_preemption(self, deadlocked: List[int]) -> int:
        """
        자원 선점으로 복구: 데드락 프로세스 중 가장 적은 자원을 보유한 프로세스 종료
        반환: 종료된 프로세스 ID
        """
        if not deadlocked:
            return -1

        # 보유 자원 합계가 가장 작은 프로세스 선택 (최소 비용 복구)
        victim = min(deadlocked, key=lambda i: sum(self.allocation[i]))
        print(f"데드락 복구: P{victim} 강제 종료 (자원 선점)")

        # 자원 반환
        for j in range(self.m):
            self.available[j] += self.allocation[victim][j]
            self.allocation[victim][j] = 0
            self.request[victim][j] = 0

        return victim
```

---

## 데이터베이스의 데드락: 교착 그래프와 탐지

RDBMS(MySQL, PostgreSQL 등)는 트랜잭션 간 락 대기 그래프(Wait-for Graph)에서 **사이클을 탐지**하여 데드락을 처리한다.

```python
class WaitForGraph:
    """
    데이터베이스 트랜잭션 대기 그래프
    트랜잭션이 보유한 락과 대기 중인 락을 추적
    """

    def __init__(self):
        self.holds = {}   # {txn_id: set of resource_ids}
        self.waits = {}   # {txn_id: resource_id}

    def acquire_lock(self, txn_id: int, resource_id: int) -> bool:
        """락 획득 시도. 이미 다른 트랜잭션이 보유 중이면 대기"""
        # 누가 이 자원을 보유하고 있는지 확인
        holder = None
        for tid, resources in self.holds.items():
            if resource_id in resources and tid != txn_id:
                holder = tid
                break

        if holder is None:
            # 자원 획득
            self.holds.setdefault(txn_id, set()).add(resource_id)
            return True
        else:
            # 대기 등록
            self.waits[txn_id] = resource_id
            return False

    def detect_deadlock(self) -> Optional[List[int]]:
        """
        Wait-for Graph에서 사이클 탐지
        반환: 사이클을 구성하는 트랜잭션 ID 목록
        """
        # 대기 그래프 구성: txn_a → txn_b (a가 b가 보유한 자원 대기)
        wait_graph = {}
        for waiting_txn, resource in self.waits.items():
            for holding_txn, resources in self.holds.items():
                if resource in resources and holding_txn != waiting_txn:
                    wait_graph.setdefault(waiting_txn, []).append(holding_txn)

        # DFS로 사이클 탐지
        visited = set()
        path = []
        path_set = set()

        def dfs_cycle(node):
            visited.add(node)
            path.append(node)
            path_set.add(node)

            for neighbor in wait_graph.get(node, []):
                if neighbor in path_set:
                    # 사이클 발견
                    cycle_start = path.index(neighbor)
                    return path[cycle_start:]
                if neighbor not in visited:
                    result = dfs_cycle(neighbor)
                    if result:
                        return result

            path.pop()
            path_set.discard(node)
            return None

        for txn in list(wait_graph.keys()):
            if txn not in visited:
                cycle = dfs_cycle(txn)
                if cycle:
                    return cycle

        return None

    def resolve_deadlock(self):
        """데드락 탐지 및 희생자 트랜잭션 롤백"""
        cycle = self.detect_deadlock()
        if cycle:
            # 가장 최근 트랜잭션(높은 ID)을 롤백
            victim = max(cycle)
            print(f"데드락 탐지: {cycle}. 트랜잭션 T{victim} 롤백")
            # 자원 해제
            self.holds.pop(victim, None)
            self.waits.pop(victim, None)
            return victim
        return None
```

---

## 실무 가이드: 데드락 없는 코드 작성

### 1. 락 획득 순서 문서화

팀 전체가 공유하는 락 순서 규약을 문서로 명시한다.

```python
# 규약: DB 연결 → 캐시 → 파일 순으로 획득
LOCK_ORDER = {
    "db_connection": 1,
    "cache": 2,
    "file_system": 3,
}
```

### 2. `try_lock` + 타임아웃

블로킹 락 대신 타임아웃이 있는 `try_lock`을 사용하고, 실패 시 지수 백오프로 재시도한다.

```python
import random

def acquire_with_backoff(lock, max_retries=5, base_delay=0.01):
    for attempt in range(max_retries):
        if lock.acquire(blocking=False):
            return True
        wait = base_delay * (2 ** attempt) + random.uniform(0, base_delay)
        time.sleep(wait)
    return False
```

### 3. 데이터베이스: 인덱스 순서 일관성

SQL 트랜잭션에서 여러 행을 업데이트할 때 항상 **기본키 오름차순**으로 처리하면 순환 대기를 방지할 수 있다.

```sql
-- 데드락 유발 가능 (순서 불일치)
-- T1: UPDATE orders WHERE id=100; UPDATE orders WHERE id=200;
-- T2: UPDATE orders WHERE id=200; UPDATE orders WHERE id=100;

-- 데드락 방지 (항상 id 오름차순)
-- T1: UPDATE orders WHERE id IN (100, 200) ORDER BY id;
-- T2: UPDATE orders WHERE id IN (100, 200) ORDER BY id;
```

---

## 주의사항과 팁

### Livelock vs Starvation

- **라이브락(Livelock)**: 프로세스들이 계속 상태를 바꾸지만 진전이 없다. 예: 두 사람이 복도에서 서로 비켜주다 계속 같은 방향으로 움직이는 상황.
- **기아(Starvation)**: 특정 프로세스가 자원을 영원히 얻지 못한다. 낮은 우선순위 프로세스가 높은 우선순위 프로세스에 계속 밀리는 경우.

### Banker's Algorithm의 한계

- **최대 필요량 사전 선언**: 현실에서 프로세스가 필요한 자원 최대량을 미리 알기 어렵다.
- **고정 자원 수**: 동적으로 자원이 추가/제거되는 환경에서 적용이 복잡하다.
- **오버헤드**: 매 자원 요청마다 O(n²m)의 안전성 검사를 수행해야 한다.

이런 이유로 실제 범용 OS는 Banker's Algorithm을 사용하지 않고, 대신 데드락 발생을 허용한 뒤 **타임아웃 기반 탐지와 재시작** 전략을 취한다.

---

## 마치며

데드락은 동시성 프로그래밍의 가장 악명 높은 버그 중 하나다. 재현이 어렵고 영향이 치명적이지만, 4가지 필요 조건을 이해하면 체계적으로 다룰 수 있다.

가장 실용적인 예방책은 **일관된 락 획득 순서**(순환 대기 제거)다. Banker's Algorithm은 개념적으로 우아하지만 실용적 제약이 있어 주로 임베디드 시스템이나 특수 환경에서 사용된다. 범용 소프트웨어에서는 타임아웃, 재시도, 락 순서 규약의 조합이 현실적인 해법이다.

## 참고 자료
- [Deadlock (computer science) - Wikipedia](https://en.wikipedia.org/wiki/Deadlock_(computer_science))
- [Banker's Algorithm - Wikipedia](https://en.wikipedia.org/wiki/Banker%27s_algorithm)
- [Handling Deadlocks - GeeksforGeeks](https://www.geeksforgeeks.org/operating-systems/handling-deadlocks/)
- [Deadlock Prevention And Avoidance - GeeksforGeeks](https://www.geeksforgeeks.org/deadlock-prevention/)
