---
layout: post
title: "데드락 완전 정복: 탐지·예방·회피 알고리즘과 Banker's Algorithm 구현"
date: 2026-11-14
categories: [cs, computer-science]
tags: [deadlock, operating-systems, concurrency, bankers-algorithm, synchronization, resource-management]
---

## 개념 설명

**데드락(Deadlock)**은 두 개 이상의 프로세스(또는 스레드)가 서로가 가진 자원을 기다리며 영원히 진행하지 못하는 상태입니다. 1965년 Dijkstra가 식사하는 철학자 문제로 이를 대중화했고, 이후 운영체제와 데이터베이스 시스템의 핵심 연구 주제가 되었습니다.

### 데드락의 4가지 필요충분 조건 (Coffman Conditions)

다음 네 가지 조건이 **동시에** 성립할 때만 데드락이 발생합니다. 하나라도 성립하지 않으면 데드락은 발생하지 않습니다.

1. **상호 배제(Mutual Exclusion)**: 자원은 한 번에 하나의 프로세스만 사용 가능
2. **점유 대기(Hold and Wait)**: 자원을 보유한 채로 다른 자원을 대기
3. **비선점(No Preemption)**: 자원을 강제로 빼앗을 수 없음
4. **순환 대기(Circular Wait)**: 프로세스들이 원형으로 서로 자원을 대기

---

## 왜 중요한가?

데드락은 개발 환경에서는 잘 드러나지 않고 프로덕션에서 갑작스럽게 나타납니다. 타이밍에 의존하기 때문에 재현이 매우 어렵습니다.

실제 사례:

- **데이터베이스**: 두 트랜잭션이 서로 다른 테이블의 행 락을 순서를 달리해 획득할 때 (MySQL, PostgreSQL 모두 데드락 감지 기능 내장)
- **JVM**: `synchronized` 블록을 중첩할 때 락 획득 순서가 다른 스레드 간에 역전되면 발생
- **운영체제**: 커널 레벨 자원(세마포어, 파일 핸들)에서도 발생 가능

데드락을 처리하는 전략은 크게 **예방(Prevention)**, **회피(Avoidance)**, **탐지 후 복구(Detection & Recovery)**, **무시(Ostrich Algorithm)** 네 가지입니다.

---

## 전략 1: 데드락 예방 (Prevention)

Coffman 조건 중 하나를 아예 성립하지 않게 설계하는 방법입니다.

### 순환 대기 제거 — 락 순서 고정

가장 실용적인 예방 기법입니다. 모든 자원에 전역 순서를 부여하고, 항상 순서에 따라 자원을 획득합니다.

```python
import threading

# 모든 락에 고유 ID를 부여하고 ID 오름차순으로만 획득
class OrderedLock:
    _next_id = 0
    _id_lock = threading.Lock()

    def __init__(self, name: str):
        self.name = name
        with OrderedLock._id_lock:
            self.id = OrderedLock._next_id
            OrderedLock._next_id += 1
        self._lock = threading.Lock()

    def acquire(self):
        self._lock.acquire()

    def release(self):
        self._lock.release()


def transfer_funds_safe(lock_a: OrderedLock, lock_b: OrderedLock, amount: float):
    """
    두 계좌 간 이체 — 항상 ID 순서로 락을 획득해 순환 대기를 제거
    """
    # 항상 ID가 작은 락부터 획득 → 순환 대기 불가
    first, second = (lock_a, lock_b) if lock_a.id < lock_b.id else (lock_b, lock_a)

    first.acquire()
    try:
        second.acquire()
        try:
            print(f"  {lock_a.name} → {lock_b.name}: {amount}원 이체 완료")
        finally:
            second.release()
    finally:
        first.release()


# 테스트 — 두 스레드가 반대 순서로 이체를 시도해도 데드락 없음
account_A = OrderedLock("계좌A")
account_B = OrderedLock("계좌B")

threads = [
    threading.Thread(target=transfer_funds_safe, args=(account_A, account_B, 100)),
    threading.Thread(target=transfer_funds_safe, args=(account_B, account_A, 200)),
]
for t in threads:
    t.start()
for t in threads:
    t.join()
print("데드락 없이 완료!")
```

### 점유 대기 제거 — 모든 자원을 한 번에 요청

프로세스 시작 시 필요한 모든 자원을 한꺼번에 요청합니다. 하나라도 얻지 못하면 전부 반납하고 재시도합니다.

```python
import threading
import time
import random

class AllOrNothingResourceManager:
    """모든 자원을 원자적으로 획득하거나 하나도 획득하지 않음"""

    def __init__(self):
        self._meta_lock = threading.Lock()
        self._resources: dict[str, threading.Lock] = {}

    def register(self, name: str):
        self._resources[name] = threading.Lock()

    def acquire_all(self, names: list[str], timeout: float = 1.0) -> bool:
        """모든 자원을 한 번에 획득 시도. 실패 시 획득한 것도 반납."""
        acquired = []
        with self._meta_lock:  # 한 번에 하나의 스레드만 획득 시도
            for name in names:
                if self._resources[name].acquire(blocking=True, timeout=timeout):
                    acquired.append(name)
                else:
                    # 하나라도 실패 시 전부 반납
                    for n in acquired:
                        self._resources[n].release()
                    return False
        return True

    def release_all(self, names: list[str]):
        for name in names:
            self._resources[name].release()


mgr = AllOrNothingResourceManager()
for res in ["CPU", "메모리", "디스크"]:
    mgr.register(res)

def worker(worker_id: int, needed: list[str]):
    for attempt in range(5):
        if mgr.acquire_all(needed):
            try:
                print(f"Worker {worker_id}: {needed} 획득 → 작업 중...")
                time.sleep(random.uniform(0.05, 0.1))
            finally:
                mgr.release_all(needed)
            print(f"Worker {worker_id}: 완료")
            return
        print(f"Worker {worker_id}: 획득 실패, 재시도 {attempt + 1}")
        time.sleep(random.uniform(0.01, 0.05))

threads = [
    threading.Thread(target=worker, args=(1, ["CPU", "메모리"])),
    threading.Thread(target=worker, args=(2, ["메모리", "디스크"])),
    threading.Thread(target=worker, args=(3, ["CPU", "디스크"])),
]
for t in threads:
    t.start()
for t in threads:
    t.join()
```

---

## 전략 2: 데드락 회피 — Banker's Algorithm

**Banker's Algorithm**(은행가 알고리즘)은 Dijkstra가 1965년 제안한 알고리즘으로, 자원 요청이 들어올 때마다 그 요청을 허용했을 때 **안전 상태(safe state)**가 유지되는지 검사합니다.

안전 상태란, 모든 프로세스가 순차적으로 완료될 수 있는 순서(**안전 순열**)가 존재하는 상태입니다.

### 핵심 자료구조

- **Available**: 현재 사용 가능한 자원 수
- **Max**: 각 프로세스가 최대로 필요한 자원 수
- **Allocation**: 각 프로세스에 현재 할당된 자원 수
- **Need**: 각 프로세스가 추가로 필요한 자원 수 (= Max − Allocation)

```python
class BankersAlgorithm:
    """Dijkstra의 은행가 알고리즘 구현"""

    def __init__(self, num_processes: int, num_resources: int,
                 available: list[int], max_demand: list[list[int]],
                 allocation: list[list[int]]):
        self.n = num_processes
        self.m = num_resources
        self.available = available[:]
        self.max_demand = [row[:] for row in max_demand]
        self.allocation = [row[:] for row in allocation]
        # Need = Max - Allocation
        self.need = [
            [max_demand[i][j] - allocation[i][j] for j in range(num_resources)]
            for i in range(num_processes)
        ]

    def is_safe(self) -> tuple[bool, list[int]]:
        """
        안전 상태 검사 — 안전 순열 반환
        O(n² × m) 시간 복잡도
        """
        work = self.available[:]
        finish = [False] * self.n
        safe_sequence = []

        while len(safe_sequence) < self.n:
            found = False
            for i in range(self.n):
                if not finish[i] and all(self.need[i][j] <= work[j] for j in range(self.m)):
                    # 프로세스 i가 완료 가능 → 자원 반납
                    work = [work[j] + self.allocation[i][j] for j in range(self.m)]
                    finish[i] = True
                    safe_sequence.append(i)
                    found = True
                    break

            if not found:
                return False, []  # 데드락 가능성 — 안전 순열 없음

        return True, safe_sequence

    def request_resources(self, process_id: int, request: list[int]) -> bool:
        """
        자원 요청 처리
        요청이 허용 가능한 경우에만 할당하고 안전 상태 재확인
        """
        pid = process_id

        # 1단계: 요청이 Need를 초과하지 않는지 확인
        if any(request[j] > self.need[pid][j] for j in range(self.m)):
            print(f"프로세스 {pid}: 최대 수요 초과 요청 — 거부")
            return False

        # 2단계: 요청이 Available을 초과하지 않는지 확인
        if any(request[j] > self.available[j] for j in range(self.m)):
            print(f"프로세스 {pid}: 자원 부족 — 대기")
            return False

        # 3단계: 임시로 자원 할당 후 안전 상태 검사
        for j in range(self.m):
            self.available[j] -= request[j]
            self.allocation[pid][j] += request[j]
            self.need[pid][j] -= request[j]

        is_safe, sequence = self.is_safe()

        if is_safe:
            print(f"프로세스 {pid}: 자원 {request} 할당 승인")
            print(f"  안전 순열: {' → '.join(f'P{s}' for s in sequence)}")
            return True
        else:
            # 안전하지 않으면 롤백
            for j in range(self.m):
                self.available[j] += request[j]
                self.allocation[pid][j] -= request[j]
                self.need[pid][j] += request[j]
            print(f"프로세스 {pid}: 자원 {request} 요청 — 불안전 상태 유발 → 거부")
            return False

    def print_state(self):
        """현재 시스템 상태 출력"""
        print("\n--- 시스템 상태 ---")
        print(f"{'프로세스':>8} | {'Allocation':^18} | {'Need':^18} | {'Max':^18}")
        for i in range(self.n):
            print(f"P{i:>7} | {str(self.allocation[i]):^18} | {str(self.need[i]):^18} | {str(self.max_demand[i]):^18}")
        print(f"Available: {self.available}")


# --- 테스트 시나리오 ---
# 자원 타입: [A, B, C] = [10, 5, 7] 단위
available   = [3, 3, 2]
max_demand  = [[7, 5, 3], [3, 2, 2], [9, 0, 2], [2, 2, 2], [4, 3, 3]]
allocation  = [[0, 1, 0], [2, 0, 0], [3, 0, 2], [2, 1, 1], [0, 0, 2]]

banker = BankersAlgorithm(
    num_processes=5,
    num_resources=3,
    available=available,
    max_demand=max_demand,
    allocation=allocation
)
banker.print_state()

safe, seq = banker.is_safe()
print(f"\n초기 상태 안전 여부: {safe}")
if safe:
    print(f"안전 순열: {' → '.join(f'P{s}' for s in seq)}")

print("\n--- 자원 요청 테스트 ---")
# P1이 [1, 0, 2] 요청 → 허용되어야 함
banker.request_resources(1, [1, 0, 2])

# P4가 [3, 3, 0] 요청 → 거부되어야 함 (불안전 상태 유발)
banker.request_resources(4, [3, 3, 0])
```

---

## 전략 3: 데드락 탐지 (Detection)

데드락이 발생하도록 허용하고, 주기적으로 **자원 할당 그래프(Resource Allocation Graph)**에서 사이클을 탐지합니다.

```python
from collections import defaultdict, deque

class DeadlockDetector:
    """자원 할당 그래프에서 데드락(사이클) 탐지"""

    def __init__(self):
        self.graph: dict[str, list[str]] = defaultdict(list)

    def add_edge(self, from_node: str, to_node: str):
        """
        P→R: 프로세스가 자원을 요청 중
        R→P: 자원이 프로세스에 할당됨
        """
        self.graph[from_node].append(to_node)

    def detect_cycle(self) -> tuple[bool, list[str]]:
        """DFS로 사이클 탐지. 사이클이 있으면 데드락."""
        visited = set()
        rec_stack = set()
        cycle_path: list[str] = []

        def dfs(node: str, path: list[str]) -> bool:
            visited.add(node)
            rec_stack.add(node)
            path.append(node)

            for neighbor in self.graph.get(node, []):
                if neighbor not in visited:
                    if dfs(neighbor, path):
                        return True
                elif neighbor in rec_stack:
                    # 사이클 발견 — 사이클 구간 추출
                    cycle_start = path.index(neighbor)
                    cycle_path.extend(path[cycle_start:] + [neighbor])
                    return True

            path.pop()
            rec_stack.remove(node)
            return False

        for node in list(self.graph.keys()):
            if node not in visited:
                if dfs(node, []):
                    return True, cycle_path

        return False, []


# 시나리오: P1→R1→P2→R2→P1 (데드락)
detector = DeadlockDetector()
detector.add_edge("P1", "R1")  # P1이 R1을 요청
detector.add_edge("R1", "P2")  # R1이 P2에 할당됨
detector.add_edge("P2", "R2")  # P2가 R2를 요청
detector.add_edge("R2", "P1")  # R2가 P1에 할당됨 → 순환!

has_deadlock, cycle = detector.detect_cycle()
print(f"데드락 감지: {has_deadlock}")
if has_deadlock:
    print(f"데드락 사이클: {' → '.join(cycle)}")
    print("조치: 사이클에 포함된 프로세스 중 하나를 종료하고 자원 반납")
```

---

## 주의사항과 팁

### 전략 선택 가이드

| 전략 | 장점 | 단점 | 적합한 상황 |
|------|------|------|-----------|
| 예방 | 간단, 오버헤드 적음 | 자원 활용률 저하 | 락 순서 고정이 쉬운 경우 |
| 회피 (Banker's) | 자원 활용률 높음 | 최대 수요 선언 필요, O(n²) | 임베디드, 실시간 시스템 |
| 탐지 + 복구 | 자원 활용률 최고 | 복구 비용, 작업 손실 가능 | 데이터베이스 시스템 |
| 무시 (Ostrich) | 오버헤드 없음 | 데드락 발생 시 재시작 필요 | 범용 OS (Unix, Windows) |

### 핵심 주의사항

**1. Banker's Algorithm의 현실적 한계**

프로세스가 미리 최대 자원 수요를 선언해야 하고, 자원 수와 프로세스 수가 고정되어 있어야 합니다. 동적으로 생성되는 스레드가 많은 현대 서버 애플리케이션에 직접 적용하기 어렵습니다.

**2. 락 획득 타임아웃**

Java의 `ReentrantLock.tryLock(timeout)`, Python의 `threading.Lock.acquire(timeout=n)` 등을 활용해 무한 대기를 방지하세요. 타임아웃 후 획득한 락을 모두 반납하고 재시도하는 패턴이 실용적입니다.

**3. 데이터베이스의 데드락 처리**

MySQL InnoDB와 PostgreSQL은 내부적으로 대기 그래프(wait-for graph)를 유지하고 주기적으로 사이클을 탐지합니다. 탐지 시 가장 비용이 적은 트랜잭션을 롤백합니다. 애플리케이션 레벨에서는 항상 같은 순서로 테이블/행에 접근하도록 쿼리를 작성해야 합니다.

**4. 철학자 식사 문제의 현실적 해법**

홀수 번호 철학자는 왼쪽 → 오른쪽, 짝수 번호 철학자는 오른쪽 → 왼쪽 순서로 포크를 집으면 순환 대기가 깨져 데드락이 발생하지 않습니다.

---

## 참고 자료

- [Wikipedia: Deadlock](https://en.wikipedia.org/wiki/Deadlock)
- [Wikipedia: Banker's algorithm](https://en.wikipedia.org/wiki/Banker%27s_algorithm)
- [Wikipedia: Dining philosophers problem](https://en.wikipedia.org/wiki/Dining_philosophers_problem)
- [GeeksforGeeks: Deadlock in Operating System](https://www.geeksforgeeks.org/introduction-of-deadlock-in-operating-system/)
