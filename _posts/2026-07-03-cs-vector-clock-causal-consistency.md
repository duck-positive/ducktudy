---
layout: post
title: "벡터 클록(Vector Clock)과 인과적 일관성 완전 정복: 분산 시스템에서 시간의 흐름을 추적하는 법"
date: 2026-07-03
categories: [cs, computer-science]
tags: [vector-clock, lamport-clock, distributed-systems, causality, happens-before, causal-consistency]
---

## 분산 시스템에서 "시간"이란?

단일 컴퓨터에서는 `System.currentTimeMillis()`를 호출하면 절대적인 시간을 알 수 있습니다. 하지만 분산 시스템에서는 이것이 불가능합니다. 각 노드는 독립적인 시계를 갖고 있고, NTP 동기화를 해도 수십 밀리초의 오차가 있습니다. 게다가 네트워크 지연 때문에 메시지 전달 순서가 실제 발생 순서와 다를 수 있습니다.

이 문제를 해결하기 위해 Leslie Lamport는 1978년 "Time, Clocks, and the Ordering of Events in a Distributed System"에서 **논리 클록(Logical Clock)** 개념을 제안했습니다. 이것이 오늘날 분산 데이터베이스, 이벤트 소싱, 버전 관리 시스템에서 널리 사용되는 **벡터 클록(Vector Clock)**의 기원입니다.

## 왜 물리적 시계로는 부족한가?

**Amazon DynamoDB 충돌 시나리오**를 생각해봅시다:

1. 사용자 A가 서울 리전에서 `name = "김철수"` 로 업데이트 (시각: 14:00:00.001)
2. 동시에 사용자 B가 도쿄 리전에서 `name = "이영희"` 로 업데이트 (시각: 14:00:00.003)
3. 두 업데이트가 네트워크를 통해 상대 리전으로 전파

어느 것이 "나중" 업데이트인가요? 물리적 시계를 보면 도쿄 업데이트가 2ms 늦지만, 서버 클록 오차가 10ms라면 이 순서를 신뢰할 수 없습니다. 두 업데이트가 완전히 독립적으로 발생했다면 어느 것도 다른 것을 "알지 못합니다" — 이것이 **인과적 관계(Causal Relationship)**가 없는 **동시적(Concurrent)** 이벤트입니다.

## Lamport 타임스탬프: 첫 번째 해결책

Lamport 타임스탬프는 단순한 규칙으로 **happened-before(선행)** 관계를 포착합니다.

**규칙:**
1. 각 프로세스는 단조 증가하는 카운터를 유지합니다.
2. 이벤트 발생 시마다 카운터를 1 증가시킵니다.
3. 메시지를 보낼 때 현재 카운터 값을 함께 전송합니다.
4. 메시지를 받을 때 `자신의 카운터 = max(자신, 수신값) + 1`로 업데이트합니다.

```python
class LamportClock:
    def __init__(self, process_id: str):
        self.pid = process_id
        self.time = 0

    def tick(self) -> int:
        """로컬 이벤트 발생"""
        self.time += 1
        return self.time

    def send(self) -> tuple:
        """메시지 전송 시 타임스탬프 포함"""
        self.time += 1
        return (self.time, self.pid)

    def receive(self, timestamp: int) -> int:
        """메시지 수신 시 시계 업데이트"""
        self.time = max(self.time, timestamp) + 1
        return self.time


# 3-프로세스 시스템 시뮬레이션
p1 = LamportClock("P1")
p2 = LamportClock("P2")
p3 = LamportClock("P3")

# P1에서 이벤트 발생
t1 = p1.tick()  # P1: time=1
print(f"P1 로컬 이벤트: t={t1}")

# P1이 P2에게 메시지 전송
msg_ts, msg_pid = p1.send()  # P1: time=2
print(f"P1 → P2 메시지 전송: t={msg_ts}")

# P2가 메시지 수신
t2 = p2.receive(msg_ts)  # P2: time=max(0,2)+1=3
print(f"P2 메시지 수신: t={t2}")

# P2가 P3에게 전달
msg2_ts, _ = p2.send()   # P2: time=4
t3 = p3.receive(msg2_ts) # P3: time=max(0,4)+1=5

print(f"\n최종 타임스탬프:")
print(f"  P1={p1.time}, P2={p2.time}, P3={p3.time}")
print(f"\nP1의 로컬 이벤트(t=1) → P2 수신(t=3): happened-before 관계 성립")
```

**Lamport 타임스탬프의 한계**: A → B (A가 B보다 먼저)이면 `ts(A) < ts(B)`가 보장됩니다. 하지만 역은 성립하지 않습니다. `ts(A) < ts(B)`라고 해서 반드시 A → B인 것은 아닙니다. 두 이벤트가 동시적(concurrent)인지 인과관계가 있는지 **구분할 수 없습니다**.

## 벡터 클록: 완전한 인과관계 추적

이 문제를 해결하기 위해 Colin Fidge와 Friedemann Mattern이 독립적으로 1988년에 **벡터 클록**을 제안했습니다.

**아이디어**: 각 프로세스가 전체 시스템의 모든 프로세스에 대한 카운터를 벡터로 유지합니다.

```
N개 프로세스 시스템에서 각 프로세스 i는:
VC_i = [c_0, c_1, ..., c_{N-1}] 를 유지
c_i: 자신이 본 자신의 이벤트 수
c_j: 자신이 마지막으로 들은 프로세스 j의 이벤트 수
```

**규칙:**
1. 로컬 이벤트: `VC_i[i] += 1`
2. 메시지 전송: 현재 벡터를 함께 전송 (전송 전 `VC_i[i] += 1`)
3. 메시지 수신: `VC_i[k] = max(VC_i[k], VC_msg[k])` for all k, then `VC_i[i] += 1`

```python
class VectorClock:
    def __init__(self, process_id: int, num_processes: int):
        self.pid = process_id
        self.n = num_processes
        self.vc = [0] * num_processes  # 벡터 클록 초기화

    def tick(self) -> list:
        """로컬 이벤트"""
        self.vc[self.pid] += 1
        return self.vc.copy()

    def send_message(self) -> list:
        """메시지 전송 — 전송 전에 자신의 카운터 증가"""
        self.vc[self.pid] += 1
        return self.vc.copy()  # 불변 스냅샷 전달

    def receive_message(self, recv_vc: list) -> list:
        """메시지 수신 — 두 벡터의 element-wise max + 자신 증가"""
        for k in range(self.n):
            self.vc[k] = max(self.vc[k], recv_vc[k])
        self.vc[self.pid] += 1
        return self.vc.copy()

    @staticmethod
    def happens_before(vc_a: list, vc_b: list) -> bool:
        """vc_a → vc_b (a가 b보다 먼저)인지 판단"""
        # 모든 원소가 ≤ 이고, 적어도 하나는 <
        less_or_equal = all(a <= b for a, b in zip(vc_a, vc_b))
        strictly_less = any(a < b for a, b in zip(vc_a, vc_b))
        return less_or_equal and strictly_less

    @staticmethod
    def concurrent(vc_a: list, vc_b: list) -> bool:
        """두 이벤트가 동시적(인과관계 없음)인지 판단"""
        a_before_b = VectorClock.happens_before(vc_a, vc_b)
        b_before_a = VectorClock.happens_before(vc_b, vc_a)
        return not a_before_b and not b_before_a


# 실제 분산 시나리오 시뮬레이션
print("=== 3-노드 분산 시스템 시뮬레이션 ===\n")

p0 = VectorClock(0, 3)  # 서울
p1 = VectorClock(1, 3)  # 도쿄
p2 = VectorClock(2, 3)  # 싱가포르

# 이벤트 A: 서울에서 주문 생성
vc_A = p0.tick()
print(f"[A] 서울 주문 생성: VC={vc_A}")

# 이벤트 B: 서울이 도쿄에 재고 확인 요청
msg_vc = p0.send_message()
print(f"[→] 서울→도쿄 메시지: VC={msg_vc}")

vc_B = p1.receive_message(msg_vc)
print(f"[B] 도쿄 재고 확인 수신: VC={vc_B}")

# 이벤트 C: 싱가포르에서 독립적으로 가격 변경 (동시적 이벤트)
vc_C = p2.tick()
print(f"[C] 싱가포르 가격 변경: VC={vc_C}")

# 인과관계 분석
print(f"\n=== 인과관계 분석 ===")
print(f"A → B? {VectorClock.happens_before(vc_A, vc_B)}")  # True
print(f"A와 C는 동시적? {VectorClock.concurrent(vc_A, vc_C)}")  # True
print(f"B와 C는 동시적? {VectorClock.concurrent(vc_B, vc_C)}")  # True
```

## 벡터 클록의 핵심 성질

벡터 클록은 다음을 완전하게 보장합니다:

- **A → B** ⟺ `VC(A) < VC(B)` (모든 원소가 ≤이고 적어도 하나는 <)
- **A ∥ B** (동시적) ⟺ `VC(A) ≮ VC(B)` AND `VC(B) ≮ VC(A)`

이것이 Lamport 타임스탬프와의 결정적 차이입니다. 벡터 클록은 인과관계를 **완전히(completely)** 포착합니다.

## 실제 활용: 충돌 감지와 해결

벡터 클록의 가장 중요한 실제 용도는 **분산 데이터 충돌 감지**입니다. Amazon DynamoDB와 Riak이 사용하는 방식입니다.

```python
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class VersionedValue:
    """벡터 클록으로 버전 관리되는 데이터"""
    value: str
    vector_clock: list
    node_id: int

class DistributedStore:
    """벡터 클록 기반 분산 키-값 저장소"""
    
    def __init__(self, node_id: int, num_nodes: int):
        self.node_id = node_id
        self.num_nodes = num_nodes
        self.vc = VectorClock(node_id, num_nodes)
        self.data: dict = {}  # key → VersionedValue

    def put(self, key: str, value: str) -> VersionedValue:
        """값 저장 — 항상 새 버전 생성"""
        clock = self.vc.tick()
        versioned = VersionedValue(value, clock.copy(), self.node_id)
        self.data[key] = versioned
        print(f"[노드 {self.node_id}] PUT {key}={value}, VC={clock}")
        return versioned

    def receive_update(self, key: str, remote: VersionedValue) -> Optional[str]:
        """
        다른 노드로부터 업데이트 수신.
        반환: None(정상 병합) 또는 충돌 설명 문자열
        """
        local = self.data.get(key)

        if local is None:
            # 로컬에 없음 → 그냥 저장
            self.data[key] = remote
            self.vc.receive_message(remote.vector_clock)
            print(f"[노드 {self.node_id}] 신규 수신: {key}={remote.value}")
            return None

        local_vc = local.vector_clock
        remote_vc = remote.vector_clock

        if VectorClock.happens_before(local_vc, remote_vc):
            # 원격이 더 최신 → 덮어쓰기
            self.data[key] = remote
            self.vc.receive_message(remote_vc)
            print(f"[노드 {self.node_id}] 원격 버전 채택: {key}={remote.value}")
            return None
        elif VectorClock.happens_before(remote_vc, local_vc):
            # 로컬이 더 최신 → 원격 무시
            print(f"[노드 {self.node_id}] 로컬 버전 유지: {key}={local.value}")
            return None
        else:
            # 동시 업데이트 → 충돌!
            conflict_msg = (
                f"충돌 감지! key={key}\n"
                f"  로컬 값='{local.value}' VC={local_vc}\n"
                f"  원격 값='{remote.value}' VC={remote_vc}\n"
                f"  → 애플리케이션 수준 해결 필요 (LWW, 사용자 선택, CRDT 등)"
            )
            print(f"[노드 {self.node_id}] {conflict_msg}")
            return conflict_msg


# 충돌 시나리오 시뮬레이션
print("=== 동시 업데이트 충돌 시뮬레이션 ===\n")
store_seoul = DistributedStore(0, 2)
store_tokyo = DistributedStore(1, 2)

# 서울에서 초기 값 설정
initial = store_seoul.put("cart:user123", "item_A")

# 도쿄가 동기화 수신
store_tokyo.receive_update("cart:user123", initial)

# 네트워크 분리 상황: 두 노드 동시 업데이트
print("\n--- 네트워크 분리 중 동시 업데이트 ---")
v_seoul = store_seoul.put("cart:user123", "item_A, item_B")  # 서울: 아이템 추가
v_tokyo = store_tokyo.put("cart:user123", "item_C")          # 도쿄: 다른 아이템으로 교체

# 네트워크 복구 후 교환
print("\n--- 네트워크 복구 후 동기화 ---")
store_tokyo.receive_update("cart:user123", v_seoul)
store_seoul.receive_update("cart:user123", v_tokyo)
```

## 인과적 일관성(Causal Consistency)

벡터 클록이 제공하는 **인과적 일관성**은 CAP 정리의 맥락에서 중요한 위치를 차지합니다.

- **강한 일관성**: 모든 읽기가 가장 최근 쓰기를 본다 → 느리지만 안전
- **인과적 일관성**: 인과관계 있는 쓰기만 순서 보장, 독립적인 쓰기는 순서 무관 → 균형점
- **최종 일관성**: 언젠가는 모든 노드가 같은 값 → 빠르지만 일시적 불일치 허용

실제로 Facebook의 Cassandra, Amazon DynamoDB, MongoDB 등은 인과적 일관성을 기반으로 설계되어 높은 가용성과 적절한 일관성을 동시에 제공합니다.

## 벡터 클록의 단점과 발전

**공간 복잡도 O(N)**: 노드 수에 비례하여 벡터 크기가 증가합니다. 수천 개 노드가 있는 시스템에서는 오버헤드가 큽니다.

**Dotted Version Vectors**: Riak이 사용하는 방식으로, 벡터 클록의 크기 문제를 개선합니다.

**Hybrid Logical Clocks (HLC)**: CockroachDB가 사용하는 방식으로, 물리적 시계와 논리적 시계를 결합하여 인과관계 추적과 직관적인 시간 표현을 동시에 달성합니다.

```python
# Hybrid Logical Clock 간단 구현
class HybridLogicalClock:
    """물리적 시계 + 논리 카운터의 조합"""
    
    def __init__(self, node_id: str):
        self.node_id = node_id
        self.l = 0  # 마지막 관찰된 최대 물리 시각
        self.c = 0  # 타이브레이커 카운터

    def _physical_time(self) -> int:
        """밀리초 단위 물리 시각 (실제로는 time.time_ns() 사용)"""
        import time
        return int(time.time() * 1000)

    def now(self) -> tuple:
        """로컬 이벤트 발생"""
        pt = self._physical_time()
        if pt > self.l:
            self.l = pt
            self.c = 0
        else:
            self.c += 1
        return (self.l, self.c, self.node_id)

    def receive(self, msg_l: int, msg_c: int) -> tuple:
        """메시지 수신"""
        pt = self._physical_time()
        old_l = self.l
        self.l = max(self.l, msg_l, pt)
        if self.l == old_l == msg_l:
            self.c = max(self.c, msg_c) + 1
        elif self.l == old_l:
            self.c += 1
        elif self.l == msg_l:
            self.c = msg_c + 1
        else:
            self.c = 0
        return (self.l, self.c, self.node_id)
```

## 주의사항과 실무 팁

**벡터 클록은 "무엇이 더 최신인가"를 알려주지 않는다**: 동시적 이벤트를 감지하면 충돌을 **해결**하는 것은 애플리케이션의 책임입니다. LWW(Last Writer Wins), 사용자 선택, 또는 CRDT를 사용합니다.

**가비지 컬렉션**: 벡터 클록은 계속 증가하므로 주기적으로 불필요한 버전을 정리해야 합니다. Riak은 주기적 클린업과 dotted version vectors를 함께 사용합니다.

**전달 보장**: 벡터 클록은 메시지가 반드시 전달된다는 가정을 합니다. 네트워크 분리 시 pending 메시지를 버퍼링하고 재전송하는 로직이 필요합니다.

벡터 클록은 분산 시스템의 "진실의 순서"를 추적하는 강력한 도구입니다. 물리적 시계가 거짓말을 하더라도 벡터 클록은 인과관계의 진실을 말합니다. Git의 병합 기록, DynamoDB의 충돌 해결, CRDTs의 동기화까지 — 분산 시스템의 데이터 일관성 어디에나 그 원리가 숨어있습니다.

## 참고 자료
- [Time, Clocks, and the Ordering of Events — Leslie Lamport (1978)](https://lamport.azurewebsites.net/pubs/time-clocks.pdf)
- [Vector Clocks and Lamport Timestamps — Medium](https://hosseinnejati.medium.com/vector-clocks-and-lamport-timestamps-ordering-events-in-distributed-systems-4f6df4c284bb)
- [Lamport & Vector Clocks — explain.technical.li](https://explain.technical.li/lamport-vector-clocks/)
- [Wikipedia: Vector clock](https://en.wikipedia.org/wiki/Vector_clock)
