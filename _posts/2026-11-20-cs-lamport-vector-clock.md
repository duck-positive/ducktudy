---
layout: post
title: "Lamport Clock와 Vector Clock 완전 정복: 분산 시스템에서 인과성을 추적하는 논리적 시계"
date: 2026-11-20
categories: [cs, computer-science]
tags: [distributed-systems, lamport-clock, vector-clock, causality, logical-clock, consistency]
---

분산 시스템에서 "무엇이 먼저 일어났는가?"라는 질문은 생각보다 훨씬 어렵다. 여러 서버가 각자 다른 물리적 시계를 가지고 있고, 네트워크 지연은 불규칙하며, NTP로 동기화해도 수 밀리초의 오차가 항상 존재한다. 이 문제를 해결하기 위해 Leslie Lamport는 1978년 혁명적인 아이디어를 제안했다: **물리적 시간 대신 논리적 인과관계를 추적하자**. 이것이 Lamport Clock의 탄생이다.

## 왜 물리적 시계로는 부족한가

분산 시스템에서 시간 동기화의 어려움을 구체적인 예로 살펴보자.

- 서버 A가 09:00:00.100에 메시지 M을 전송
- 서버 B가 09:00:00.095에 메시지 M을 수신

물리 시계상으로는 B가 M을 받기 전에 M이 존재하지 않는 것처럼 보인다. 클록 드리프트(clock drift)와 네트워크 비결정성 때문에 발생하는 이 역설은 분산 시스템 디버깅을 극도로 어렵게 만든다. 더 큰 문제는 **인과성(causality)** — "A가 B의 원인인가?" — 을 물리 시계로는 판단할 수 없다는 점이다.

Lamport의 핵심 통찰은 단순하다: **인과관계가 있는 이벤트들 사이에서만 순서가 의미 있다**. 서로 아무 관련 없는 두 이벤트는 순서를 정할 필요가 없다.

## 핵심 개념: happens-before 관계

Lamport는 `→` (happens-before) 관계를 세 가지 규칙으로 정의했다:

1. 같은 프로세스 내에서 a가 b보다 먼저 발생하면, `a → b`
2. 프로세스 i가 메시지 m을 전송하고 프로세스 j가 m을 수신하면, `send(m) → receive(m)`
3. 이행성: `a → b`이고 `b → c`이면 `a → c`

이 관계가 성립하지 않으면 두 이벤트는 **동시적(concurrent)** 이다 (`a ∥ b`).

## Lamport Clock: 스칼라 논리 시계

### 알고리즘

각 프로세스는 정수형 카운터 `LC`를 유지한다:

- **내부 이벤트 발생 시**: `LC = LC + 1`
- **메시지 전송 시**: `LC = LC + 1`, 메시지에 현재 `LC` 값 첨부
- **메시지 수신 시**: `LC = max(LC, LC_received) + 1`

### Python 구현

```python
import threading
from dataclasses import dataclass, field
from typing import Any

@dataclass
class LamportClock:
    process_id: str
    _counter: int = field(default=0, init=False)
    _lock: threading.Lock = field(default_factory=threading.Lock, init=False)

    def tick(self) -> int:
        """내부 이벤트 발생 시 카운터 증가"""
        with self._lock:
            self._counter += 1
            return self._counter

    def send(self) -> int:
        """메시지 전송 시 타임스탬프 반환"""
        with self._lock:
            self._counter += 1
            return self._counter

    def receive(self, received_ts: int) -> int:
        """메시지 수신 시 카운터 업데이트"""
        with self._lock:
            self._counter = max(self._counter, received_ts) + 1
            return self._counter

    @property
    def value(self) -> int:
        return self._counter


def simulate_lamport():
    """Lamport Clock 동작 시뮬레이션"""
    clock_a = LamportClock("A")
    clock_b = LamportClock("B")

    # 프로세스 A: 이벤트 1
    ts_a1 = clock_a.tick()
    print(f"A 이벤트1: LC={ts_a1}")

    # 프로세스 A: 메시지 전송
    ts_send = clock_a.send()
    print(f"A 전송: LC={ts_send}")

    # 프로세스 B: 독립 이벤트
    ts_b1 = clock_b.tick()
    print(f"B 이벤트1: LC={ts_b1}")

    # 프로세스 B: 메시지 수신
    ts_recv = clock_b.receive(ts_send)
    print(f"B 수신: LC={ts_recv}")

    # 결과: ts_send < ts_recv (인과관계 보장)
    print(f"\n인과관계 검증: 전송({ts_send}) < 수신({ts_recv}) = {ts_send < ts_recv}")


simulate_lamport()
```

**출력:**
```
A 이벤트1: LC=1
A 전송: LC=2
B 이벤트1: LC=1
B 수신: LC=3

인과관계 검증: 전송(2) < 수신(3) = True
```

### Lamport Clock의 한계

Lamport Clock은 `a → b`이면 `LC(a) < LC(b)`를 보장하지만, **역은 성립하지 않는다**. 즉, `LC(a) < LC(b)`여도 `a → b`가 아닐 수 있다. 두 이벤트가 단순히 큰 타임스탬프를 가졌다는 것이 인과 관계를 의미하지는 않는다.

이것이 Vector Clock가 필요한 이유다.

## Vector Clock: 완전한 인과성 추적

Vector Clock은 Fidge와 Mattern이 1988년 독립적으로 제안한 자료구조로, N개 프로세스로 구성된 시스템에서 각 프로세스는 N차원 정수 벡터를 유지한다.

### 알고리즘

프로세스 i는 벡터 `VC[0..N-1]`을 유지한다:

- **내부 이벤트**: `VC[i] += 1`
- **메시지 전송**: `VC[i] += 1`, 전체 벡터를 메시지에 첨부
- **메시지 수신**: `VC[j] = max(VC[j], received_VC[j])` (모든 j에 대해), `VC[i] += 1`

### 비교 규칙

두 벡터 타임스탬프 `V1`, `V2`에 대해:
- `V1 < V2` (V1 happens-before V2): 모든 k에 대해 `V1[k] ≤ V2[k]`이고 적어도 하나는 `V1[k] < V2[k]`
- `V1 ∥ V2` (동시적): `V1 < V2`도 `V2 < V1`도 아닌 경우

### Python 구현

```python
from copy import deepcopy
from typing import List, Dict

class VectorClock:
    def __init__(self, process_id: int, num_processes: int):
        self.pid = process_id
        self.clock: List[int] = [0] * num_processes

    def tick(self) -> List[int]:
        self.clock[self.pid] += 1
        return deepcopy(self.clock)

    def send(self) -> List[int]:
        self.clock[self.pid] += 1
        return deepcopy(self.clock)

    def receive(self, received_vc: List[int]) -> List[int]:
        # 각 차원의 최댓값으로 업데이트
        for i in range(len(self.clock)):
            self.clock[i] = max(self.clock[i], received_vc[i])
        self.clock[self.pid] += 1
        return deepcopy(self.clock)

    @staticmethod
    def happens_before(vc1: List[int], vc2: List[int]) -> bool:
        """vc1 → vc2 (vc1 happens-before vc2)"""
        leq = all(vc1[i] <= vc2[i] for i in range(len(vc1)))
        lt = any(vc1[i] < vc2[i] for i in range(len(vc1)))
        return leq and lt

    @staticmethod
    def concurrent(vc1: List[int], vc2: List[int]) -> bool:
        """vc1 ∥ vc2 (동시적 이벤트)"""
        return (not VectorClock.happens_before(vc1, vc2) and
                not VectorClock.happens_before(vc2, vc1))


def simulate_vector_clock():
    """Vector Clock 시뮬레이션: 3개 프로세스"""
    p0 = VectorClock(0, 3)  # 프로세스 0
    p1 = VectorClock(1, 3)  # 프로세스 1
    p2 = VectorClock(2, 3)  # 프로세스 2

    # p0: 이벤트 a
    vc_a = p0.tick()
    print(f"p0 이벤트a: {vc_a}")  # [1, 0, 0]

    # p1: 이벤트 b (독립적)
    vc_b = p1.tick()
    print(f"p1 이벤트b: {vc_b}")  # [0, 1, 0]

    # p0 → p1 메시지 전송
    vc_send = p0.send()
    print(f"p0 전송:    {vc_send}")  # [2, 0, 0]

    vc_recv = p1.receive(vc_send)
    print(f"p1 수신:    {vc_recv}")  # [2, 2, 0]

    # p1 → p2 메시지 전송
    vc_send2 = p1.send()
    print(f"p1 전송:    {vc_send2}")  # [2, 3, 0]

    vc_recv2 = p2.receive(vc_send2)
    print(f"p2 수신:    {vc_recv2}")  # [2, 3, 1]

    # 인과관계 검증
    print(f"\n인과관계 분석:")
    print(f"a → p1수신? {VectorClock.happens_before(vc_a, vc_recv)}")   # True
    print(f"a ∥ b?      {VectorClock.concurrent(vc_a, vc_b)}")           # True (동시적)
    print(f"p1수신 → p2수신? {VectorClock.happens_before(vc_recv, vc_recv2)}")  # True


simulate_vector_clock()
```

**출력:**
```
p0 이벤트a: [1, 0, 0]
p1 이벤트b: [0, 1, 0]
p0 전송:    [2, 0, 0]
p1 수신:    [2, 2, 0]
p1 전송:    [2, 3, 0]
p2 수신:    [2, 3, 1]

인과관계 분석:
a → p1수신? True
a ∥ b?      True (동시적)
p1수신 → p2수신? True
```

이제 `a ∥ b` (이벤트 a와 b는 서로 인과관계가 없는 동시적 이벤트)라는 것을 정확히 감지할 수 있다. Lamport Clock으로는 이를 판단할 수 없었다.

## 실제 활용 사례

### 1. Amazon DynamoDB의 충돌 해결
DynamoDB는 Vector Clock(실제로는 version vector)을 사용해 여러 복제본에 동시 쓰기가 발생했을 때 충돌을 감지한다. `concurrent` 이벤트가 감지되면 "last writer wins" 대신 애플리케이션에 충돌 해결을 위임한다.

### 2. 분산 디버깅
Google의 Dapper, Zipkin 등 분산 추적 시스템은 Lamport Clock 변형을 사용해 마이크로서비스 간 요청 흐름을 재구성한다.

### 3. CRDTs
Conflict-free Replicated Data Types에서 Vector Clock은 어느 연산이 더 "최신"인지, 어느 연산이 동시에 발생했는지를 판단하는 핵심 메커니즘이다.

## 한계와 대안

**Vector Clock의 단점:**
- 프로세스 수 N에 비례하는 메모리와 네트워크 오버헤드 (`O(N)` 크기)
- 프로세스가 동적으로 추가/제거되는 환경에서 관리 복잡도 증가

**대안:**
- **Interval Tree Clocks (ITC)**: 프로세스 수 고정 없이 동적 확장 가능
- **Hybrid Logical Clocks (HLC)**: 물리 시간과 논리 시간을 결합, NTP 호환성 유지하면서 인과성 보장
- **Plausible Clocks**: N보다 작은 고정 크기 벡터로 근사적 인과성 추적

## 주의사항과 팁

1. **Vector Clock는 완전 순서(total order)를 제공하지 않는다.** 동시적 이벤트들 사이에는 순서가 없다. 완전 순서가 필요하면 Lamport Clock + 프로세스 ID로 tie-breaking해야 한다.

2. **Clock의 단조 증가를 보장하라.** 카운터가 절대 감소하지 않도록, 멀티스레드 환경에서는 반드시 동기화해야 한다.

3. **메시지 전달 보장이 필요하다.** Vector Clock은 메시지가 순서대로(causally) 전달된다고 가정한다. 메시지 손실이나 재정렬이 발생하면 인과성이 깨진다.

4. **대규모 시스템에서는 Version Vector 사용을 고려하라.** 오브젝트별로 쓰기에 참여한 복제본만 추적하는 Version Vector는 전체 Vector Clock보다 훨씬 작은 크기를 유지한다.

분산 시스템을 설계할 때 "언제 발생했는가" 보다 "무엇이 무엇을 야기했는가"에 집중하는 것이 훨씬 더 견고한 시스템을 만드는 열쇠다. Lamport Clock과 Vector Clock은 그 첫 번째 도구다.

## 참고 자료
- [Lamport Timestamp - Wikipedia](https://en.wikipedia.org/wiki/Lamport_timestamp)
- [Vector Clock - Wikipedia](https://en.wikipedia.org/wiki/Vector_clock)
- [Logical Clocks - Rutgers University CS417](https://people.cs.rutgers.edu/pxk/417/notes/logical-clocks.html)
- [Vector Clocks - Princeton COS418 Lecture](https://www.cs.princeton.edu/courses/archive/spring22/cos418/docs/L6-vc.pdf)
