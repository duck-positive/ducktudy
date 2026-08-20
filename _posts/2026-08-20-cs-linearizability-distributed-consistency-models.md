---
layout: post
title: "선형화 가능성과 분산 일관성 모델 완전 정복: Linearizability부터 Eventual Consistency까지"
date: 2026-08-20
categories: [cs, computer-science]
tags: [linearizability, consistency-models, distributed-systems, sequential-consistency, causal-consistency, eventual-consistency, jepsen, CAP-theorem, PACELC]
---

분산 시스템에서 "데이터 일관성"이란 과연 무엇인가? 단일 노드 시스템에서는 자명한 이 질문이, 여러 서버가 네트워크로 연결된 환경에서는 놀랍도록 복잡해진다. 어떤 노드에서 값을 썼는데 다른 노드에서 곧바로 읽으면 새 값을 볼 수 있는가? 네트워크 지연이 있다면? 노드 중 하나가 죽었다면? 이 질문들에 대한 답이 바로 **일관성 모델(Consistency Model)**이다. 1990년 Herlihy와 Wing이 정식으로 정의한 **선형화 가능성(Linearizability)**은 분산 시스템에서 달성 가능한 가장 강력한 일관성 모델이며, 이를 기준점으로 다양한 약한 모델들이 파생되었다.

## 일관성이란 무엇인가

단순히 말하면, 일관성 모델은 **연산 이력(History)**이 어떤 조건을 만족해야 "올바르다"고 볼 수 있는지를 정의한다.

예를 들어, 레지스터(초기값 0)에 대해 두 클라이언트가 동시에 다음 연산을 수행한다고 하자:

```
클라이언트 A: write(1) → ok
클라이언트 B: read()   → 0 또는 1?
```

클라이언트 B가 read를 A의 write 이후에 시작했지만 0을 읽었다면, 이것이 허용되는가? 일관성 모델에 따라 답이 다르다.

## 일관성 모델 스펙트럼

일관성 모델은 강도(Strength) 순으로 계층 구조를 형성한다:

```
Strict Consistency
    ↓  (약화)
Linearizability (선형화 가능성)
    ↓  (약화)
Sequential Consistency (순차 일관성)
    ↓  (약화)
Causal Consistency (인과 일관성)
    ↓  (약화)
PRAM / Pipeline Consistency
    ↓  (약화)
Eventual Consistency (최종 일관성)
```

강한 모델일수록 정확성은 높지만 성능/가용성 비용이 크다. 약한 모델일수록 높은 가용성과 낮은 레이턴시를 달성하기 쉽지만, 애플리케이션이 이상한 동작에 대비해야 한다.

## 선형화 가능성: 가장 강력한 실용적 일관성

### 정식 정의

Maurice Herlihy와 Jeannette Wing이 1990년 논문 "Linearizability: A Correctness Condition for Concurrent Objects"에서 정의했다.

**연산 이력 H**는 다음 조건을 만족하면 **선형화 가능(Linearizable)**하다:
1. H를 완전한 이력 H'로 확장할 수 있다 (완료되지 않은 연산을 응답으로 보완하거나 제거)
2. H'와 등가인(equivalent) 순차 이력 S가 존재한다
3. S는 해당 객체의 순차 명세(Sequential Specification)를 만족한다
4. H'에서 연산 A의 응답이 B의 호출보다 앞서면(A → B), S에서도 A가 B보다 앞선다

핵심은 **선형화 포인트(Linearization Point)**: 각 연산이 호출(invocation)과 응답(response) 사이의 어느 한 순간에 원자적으로 실행된 것처럼 보여야 한다.

### 직관적 이해

선형화 가능성은 "외부에서 볼 때 연산이 순간적으로 일어난 것처럼 보인다"는 것이다. 연산이 시간적으로 겹치더라도, 어느 한 순간에 딱 실행된 것처럼 전체 시스템이 일관된 뷰를 제공해야 한다.

```
시간 -→
클라이언트 A: [----write(1)----]
클라이언트 B:        [----read()----]
                         ↑
                    선형화 포인트: 이 순간에 write(1)이 실행된 것으로 간주
                    따라서 B의 read()는 반드시 1을 반환해야 함
```

### 구현 방법

선형화 가능성은 주로 **Raft, Paxos, 2PC** 같은 합의 프로토콜로 구현된다. 리더(Leader)가 모든 쓰기를 직렬화하고, 읽기도 리더를 통하거나 쿼럼을 확인한다.

```python
# 선형화 가능성을 위반하는 잘못된 분산 레지스터 구현
class BrokenDistributedRegister:
    def __init__(self, node_id, replicas):
        self.node_id = node_id
        self.replicas = replicas
        self.value = 0

    def write(self, v):
        self.value = v
        for replica in self.replicas:
            replica.async_update(v)  # 비동기 → 선형화 위반 가능

    def read(self):
        return self.value  # 전파 전이면 구 값 반환 → 선형화 위반!


# 선형화 가능한 올바른 구현 (단순화된 Raft 스타일)
import threading
from typing import List

class LinearizableRegister:
    def __init__(self, node_id: str, all_nodes: list):
        self.node_id = node_id
        self.all_nodes = all_nodes
        self.value = 0
        self.lock = threading.Lock()

    def write(self, v: int) -> bool:
        """
        쿼럼 쓰기: 과반수 노드에 성공해야 완료.
        모든 쓰기는 단조 증가하는 버전 번호와 함께.
        """
        quorum_size = len(self.all_nodes) // 2 + 1
        acks = 0
        for node in self.all_nodes:
            try:
                node.apply_write(v)  # 동기 RPC
                acks += 1
            except Exception:
                pass
        if acks >= quorum_size:
            return True  # 쿼럼 확보 → 선형화 포인트
        return False

    def read(self) -> int:
        """
        쿼럼 읽기: 과반수에서 읽어 최신 값을 반환.
        write-quorum과 read-quorum이 반드시 겹침을 보장.
        """
        quorum_size = len(self.all_nodes) // 2 + 1
        values = []
        for node in self.all_nodes:
            try:
                v = node.get_value()
                values.append(v)
                if len(values) >= quorum_size:
                    break
            except Exception:
                pass
        if len(values) >= quorum_size:
            return max(values)
        raise Exception("읽기 쿼럼 실패")
```

## 순차 일관성: 실시간 순서를 포기하다

**순차 일관성(Sequential Consistency)**은 Lamport이 1979년 정의했다. 선형화 가능성과 거의 같지만, 하나가 다르다: **실시간 순서(real-time order)를 보장하지 않는다.**

선형화 가능성에서는 "연산 A가 실제 시간에서 연산 B보다 먼저 끝났으면, 순차 이력에서도 A가 먼저다." 순차 일관성에서는 이 제약이 없다. 단지 "모든 프로세스가 동일한 순서를 본다"만 보장한다.

```
# 순차 일관성을 만족하지만 선형화 가능성을 위반하는 예시

# 시간 흐름:
# P1: write(x=1)이 완료됨
# P2: write(y=1)이 완료됨
# P3: read(x)→0, read(y)→1  ← x가 0? write(x=1)이 먼저 완료됐는데!
# P4: read(x)→1, read(y)→0

# P3와 P4가 다른 순서를 봄 → 순차 일관성 위반
# 모든 프로세스가 동일한 순서를 본다면 순차 일관성 만족
# 하지만 write(x=1)이 실제 완료 후에도 read(x)→0이면 → 선형화 위반
```

순차 일관성은 일부 멀티프로세서 메모리 모델(x86 TSO의 부분 집합)에서 나타나며, 엄격한 시간 보장 없이 프로그래밍 직관성을 제공한다.

## 인과 일관성: 관련된 연산만 순서 보장

**인과 일관성(Causal Consistency)**은 인과 관계가 있는 연산들 사이에서만 순서를 보장한다.

두 연산 A와 B가 인과 관계를 가지는 경우:
- A → B: 동일 프로세스에서 A가 B보다 먼저 실행
- A → B: A가 값을 쓰고 B가 그 값을 읽음
- A → B: 어떤 연산 C에 대해 A → C이고 C → B (이행성)

```python
# 벡터 클록 기반 인과 일관성 구현 (개념 코드)
from typing import Dict, Optional

class VectorClock:
    def __init__(self, node_id: str, nodes: list):
        self.node_id = node_id
        self.clock: Dict[str, int] = {n: 0 for n in nodes}

    def increment(self):
        self.clock[self.node_id] += 1

    def merge(self, other: 'VectorClock'):
        for node, ts in other.clock.items():
            self.clock[node] = max(self.clock.get(node, 0), ts)

    def happens_before(self, other: 'VectorClock') -> bool:
        """self → other (self가 other보다 인과적으로 앞서는지)"""
        all_nodes = set(list(self.clock) + list(other.clock))
        return (all(self.clock.get(n, 0) <= other.clock.get(n, 0) for n in all_nodes)
                and any(self.clock.get(n, 0) < other.clock.get(n, 0) for n in all_nodes))

    def concurrent(self, other: 'VectorClock') -> bool:
        """두 이벤트가 동시 발생 (인과 관계 없음)"""
        return (not self.happens_before(other) and not other.happens_before(self))


class CausalKVStore:
    """인과 일관성을 보장하는 간단한 키-값 저장소"""

    def __init__(self, node_id: str, nodes: list):
        self.node_id = node_id
        self.nodes = nodes
        self.store: Dict[str, tuple] = {}
        self.vc = VectorClock(node_id, nodes)

    def write(self, key: str, value: str) -> VectorClock:
        self.vc.increment()
        vc_snapshot = VectorClock(self.node_id, self.nodes)
        vc_snapshot.clock = dict(self.vc.clock)
        self.store[key] = (value, vc_snapshot)
        return vc_snapshot

    def read(self, key: str, context: Optional[VectorClock] = None):
        if key not in self.store:
            return None, self.vc
        stored_value, stored_vc = self.store[key]
        if context and not context.happens_before(stored_vc):
            raise Exception("인과 의존성 미충족: 잠시 후 재시도")
        return stored_value, stored_vc
```

## 최종 일관성: 언젠가 같아진다

**최종 일관성(Eventual Consistency)**은 가장 약한 유용한 모델이다. "업데이트가 없으면, 언젠가 모든 복제본이 같은 값으로 수렴한다." 언제? 보장 없다. 수렴 중간에 어떤 값을 볼지? 보장 없다. 단지 수렴한다는 것만 보장한다.

Amazon DynamoDB(기본), Apache Cassandra(기본), DNS가 최종 일관성을 제공한다.

**Strong Eventual Consistency(강한 최종 일관성)**: 동일한 업데이트 집합을 받은 노드들은 동일한 상태에 있다. CRDT가 이를 달성한다.

## 실제 시스템의 일관성 모델

```python
# 각 시스템의 일관성 모델 정리
consistency_matrix = {
    "ZooKeeper": ("Linearizable (읽기는 리더에서)", None, "Zab 합의"),
    "etcd": ("Linearizable", "항상 linearizable", "Raft"),
    "DynamoDB": ("Eventual", "Strongly Consistent Read", "쿼럼 기반"),
    "Cassandra": ("Eventual", "QUORUM 읽기/쓰기", "Gossip + 쿼럼"),
    "Redis Cluster": ("Eventual (복제 비동기)", "WAIT 명령어", "비동기 복제"),
    "PostgreSQL (단일)": ("Serializable", "Serializable", "MVCC + 2PL"),
    "CockroachDB": ("Serializable", "Serializable", "Raft + HLC"),
    "Spanner": ("External Consistency (≈Linearizable)", "항상", "TrueTime API"),
}

# Jepsen 테스트 스타일: 선형화 가능성 검증 (간단한 시뮬레이터)
from dataclasses import dataclass, field
from typing import Any

@dataclass
class Operation:
    client: int
    op_type: str   # 'write' or 'read'
    key: str
    value: Any
    start_time: float
    end_time: float
    result: Any = None

def check_linearizability(history: list) -> bool:
    """
    간단한 선형화 가능성 체커 (레지스터 한정).
    실제 Knossos/Elle는 훨씬 정교한 알고리즘 사용.
    """
    writes = [(op.end_time, op.value)
              for op in history if op.op_type == 'write']
    reads = [(op.start_time, op.end_time, op.result)
             for op in history if op.op_type == 'read']

    for r_start, r_end, r_result in reads:
        valid_values = {v for t, v in writes if t <= r_end}
        valid_values.add(0)  # 초기값
        if r_result not in valid_values:
            return False
    return True

# 올바른 이력 예시
correct_history = [
    Operation(1, 'write', 'x', 1, 0.0, 0.5),
    Operation(2, 'read',  'x', None, 0.6, 0.9, result=1),
]
print("올바른 이력:", check_linearizability(correct_history))  # True

# 잘못된 이력 예시
bad_history = [
    Operation(1, 'write', 'x', 1, 0.0, 0.5),
    Operation(2, 'read',  'x', None, 0.6, 0.9, result=0),  # 위반
]
print("잘못된 이력:", check_linearizability(bad_history))  # False
```

## CAP 정리와 일관성 모델의 관계

CAP 정리의 C는 선형화 가능성(Linearizability)을 의미한다. 파티션(P) 상황에서:
- **CP 시스템** (ZooKeeper, HBase, etcd): 파티션 시 일부 노드가 응답 거부 → 가용성 희생
- **AP 시스템** (Cassandra, DynamoDB, CouchDB): 파티션 시에도 응답 → 일관성(선형화 가능성) 희생

그러나 PACELC 모델은 더 현실적이다: 파티션 없을 때도 레이턴시(L)와 일관성(C) 사이의 트레이드오프가 존재한다.

```
PACELC:
- Partition 시: Available vs Consistent (CAP)
- Else (정상 운영 시): Latency vs Consistent

예:
- DynamoDB: PA/EL (파티션→가용성, 정상→저레이턴시)
- ZooKeeper: PC/EC (파티션→일관성, 정상→일관성 우선)
- Cassandra: PA/EL (설정에 따라 다름)
- Spanner: PC/EC (TrueTime으로 낮은 레이턴시도 달성)
```

## 주의사항과 실무 팁

**선형화 가능성은 비싸다**: Raft 리더를 통한 직렬화는 리더 위치에서 수백 km 떨어진 클라이언트에게는 큰 레이턴시가 된다. Google Spanner는 TrueTime API (GPS + 원자시계)로 이를 완화한다.

**"강한 일관성"의 오용**: MySQL의 기본 트랜잭션 격리 수준은 `REPEATABLE READ`이며, Serializable도 선형화 가능성과 다르다. "강한 일관성"이 무엇인지 정확히 정의하는 것이 중요하다.

**Jepsen 테스트**: Kyle Kingsbury가 개발한 Jepsen 프레임워크는 실제 분산 시스템이 주장하는 일관성 모델을 실제로 만족하는지 혼돈 엔지니어링으로 검증한다. 많은 DB가 Jepsen 테스트에서 일관성 위반이 발견되었다.

**세션 일관성**: 단일 클라이언트 세션 내에서만 일관성을 보장하는 모델. 클라이언트가 직전에 쓴 값을 이후 읽기에서 반드시 볼 수 있다. Amazon S3, DynamoDB가 세션 일관성을 제공한다.

## 참고 자료
- [Herlihy & Wing, "Linearizability: A Correctness Condition for Concurrent Objects" (1990)](https://dl.acm.org/doi/10.1145/78969.78972)
- [Martin Kleppmann, "Designing Data-Intensive Applications" (O'Reilly)](https://dataintensive.net/)
- [Jepsen 분산 시스템 안전성 분석](https://jepsen.io/consistency)
- [Princeton COS418 일관성 모델 강의자료](https://www.cs.princeton.edu/courses/archive/fall22/cos418/docs/L14-consistency.pdf)
