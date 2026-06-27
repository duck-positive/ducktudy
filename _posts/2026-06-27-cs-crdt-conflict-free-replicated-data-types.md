---
layout: post
title: "CRDT 완전 정복: 분산 시스템에서 잠금 없이 충돌 없는 동기화를 구현하는 법"
date: 2026-06-27
categories: [cs, computer-science]
tags: [crdt, distributed-systems, eventual-consistency, conflict-free, replication, collaborative-editing, cap-theorem]
---

구글 Docs에서 여러 사람이 동시에 같은 문서를 편집한다. 두 사람이 동시에 같은 줄을 수정하면 어떻게 될까? 중앙 서버가 항상 살아있다면 락(Lock)으로 해결할 수 있지만, 네트워크가 분단(Partition)되거나 오프라인 편집을 지원해야 한다면 락 기반 방식은 무너진다. CRDT(Conflict-free Replicated Data Type)는 이 문제를 수학적 구조로 해결한다. 어떤 순서로 연산을 적용하든, 어떤 레플리카가 먼저 수렴하든 **결과가 반드시 같아진다**는 보장이 핵심이다.

---

## 개념 설명

### CAP 정리의 맥락

Eric Brewer의 CAP 정리는 분산 시스템이 Consistency(일관성), Availability(가용성), Partition tolerance(분단 내성) 세 가지를 동시에 만족할 수 없다고 말한다. 네트워크 분단이 발생하면 CP(일관성 우선)와 AP(가용성 우선) 중 하나를 택해야 한다.

CRDT는 **강한 최종 일관성(Strong Eventual Consistency, SEC)** 모델을 통해 AP 시스템에서도 안전하게 동작하도록 설계된다:

- **Eventual Consistency**: 네트워크가 복구되면 모든 레플리카가 동일한 상태에 수렴
- **Strong**: 같은 연산 집합을 받은 레플리카는 **수신 순서에 무관하게** 항상 동일한 상태

### 두 가지 CRDT 설계 전략

**1. State-based CRDT (CvRDT — Convergent CRDT)**

전체 상태를 주기적으로 브로드캐스트하고, 병합 함수(merge)로 상태를 합친다. 병합은 반드시 **교환적(commutative), 결합적(associative), 멱등적(idempotent)** 이어야 한다. 수학적으로 반격자(join-semilattice) 구조다.

```
상태 S1 병합 S2 = S1 ∪ S2 (항상 더 큰 상태로 수렴)
merge(S1, S2) = merge(S2, S1)         ← 교환적
merge(merge(S1, S2), S3) = merge(S1, merge(S2, S3))  ← 결합적
merge(S1, S1) = S1                    ← 멱등적
```

**2. Operation-based CRDT (CmRDT — Commutative CRDT)**

연산 자체를 브로드캐스트하며, 연산들이 **교환적(commutative)** 이기만 하면 순서가 달라도 결과가 같다. 상태 전체를 전송하지 않아도 되므로 대역폭 효율이 높다.

### 대표적인 CRDT 자료구조

| 자료구조 | 특징 | 사용 사례 |
|----------|------|-----------|
| G-Counter | 증가만 가능한 카운터 | 방문자 수, 좋아요 수 |
| PN-Counter | 증가/감소 가능 카운터 | 재고 수량, 잔고 |
| G-Set | 원소 추가만 가능한 집합 | 태그, 완료된 작업 목록 |
| 2P-Set | 추가·제거 가능 집합 | 장바구니 |
| OR-Set | 동시 추가·제거 처리 집합 | 협업 편집 태그 |
| LWW-Register | 마지막 쓰기 승리 레지스터 | 설정값, 프로필 |
| RGA | 협업 텍스트 편집용 배열 | 구글 Docs 방식 |

---

## 왜 필요한가

### 오프라인·분단 환경에서의 안정성

모바일 앱, IoT 기기, 지리적으로 분산된 멀티 리전 데이터베이스에서는 네트워크 연결이 언제든 끊길 수 있다. 기존 OT(Operational Transformation) 방식은 중앙 서버가 연산 변환을 중재해야 하므로 오프라인 환경에서 사용하기 어렵다. CRDT는 중앙 조정자 없이 각 레플리카가 독립적으로 동작할 수 있다.

### 실제 채택 사례

- **Redis Enterprise**: CRDT 기반 액티브-액티브 지역 분산 데이터베이스
- **Riak**: League of Legends의 채팅 시스템(7.5백만 동시 접속, 초당 11,000 메시지)
- **Apple Notes**: 오프라인 편집 후 동기화
- **Figma, Linear**: 협업 도구의 실시간 충돌 없는 편집
- **Automerge / Yjs**: 오픈소스 CRDT 라이브러리

---

## 실제 구현 예제

### 예제 1: G-Counter와 PN-Counter 직접 구현 (Python)

```python
from dataclasses import dataclass, field
from typing import Dict
import uuid

@dataclass
class GCounter:
    """
    G-Counter (Grow-only Counter): 증가만 가능한 분산 카운터.
    각 노드는 자신의 카운터만 증가시키며, 전체 값은 모든 노드 합산.
    """
    node_id: str
    counts: Dict[str, int] = field(default_factory=dict)

    def __post_init__(self):
        self.counts[self.node_id] = 0

    def increment(self, amount: int = 1):
        self.counts[self.node_id] = self.counts.get(self.node_id, 0) + amount

    def value(self) -> int:
        return sum(self.counts.values())

    def merge(self, other: 'GCounter') -> 'GCounter':
        """두 카운터 병합: 각 노드별 최댓값 취득 (join-semilattice)"""
        merged = GCounter(node_id=self.node_id)
        all_nodes = set(self.counts.keys()) | set(other.counts.keys())
        merged.counts = {
            node: max(self.counts.get(node, 0), other.counts.get(node, 0))
            for node in all_nodes
        }
        return merged


@dataclass
class PNCounter:
    """
    PN-Counter: 증가·감소 모두 가능한 분산 카운터.
    내부적으로 GCounter 두 개(P: 증가, N: 감소)로 구성.
    """
    node_id: str
    p: GCounter = field(init=False)
    n: GCounter = field(init=False)

    def __post_init__(self):
        self.p = GCounter(node_id=self.node_id)
        self.n = GCounter(node_id=self.node_id)

    def increment(self, amount: int = 1):
        self.p.increment(amount)

    def decrement(self, amount: int = 1):
        self.n.increment(amount)

    def value(self) -> int:
        return self.p.value() - self.n.value()

    def merge(self, other: 'PNCounter') -> 'PNCounter':
        merged = PNCounter(node_id=self.node_id)
        merged.p = self.p.merge(other.p)
        merged.n = self.n.merge(other.n)
        return merged


# 시뮬레이션: 두 노드가 독립적으로 카운터를 조작한 후 병합
node_a = PNCounter(node_id="node_A")
node_b = PNCounter(node_id="node_B")

# 노드 A: +3
node_a.increment(3)

# 노드 B: +5, -1 (독립적으로 수행)
node_b.increment(5)
node_b.decrement(1)

print(f"노드 A 로컬 값: {node_a.value()}")  # 3
print(f"노드 B 로컬 값: {node_b.value()}")  # 4

# 네트워크 복구 후 병합 (순서 무관)
merged_ab = node_a.merge(node_b)
merged_ba = node_b.merge(node_a)  # 반대 순서로 병합

print(f"A←B 병합 결과: {merged_ab.value()}")  # 7 (교환법칙 성립)
print(f"B←A 병합 결과: {merged_ba.value()}")  # 7 (동일한 결과 보장)
```

### 예제 2: OR-Set (Observed-Remove Set) 구현 (Python)

2P-Set의 문제점 — 한 레플리카가 원소를 제거하고 다른 레플리카가 동시에 같은 원소를 추가하면 충돌이 발생한다. OR-Set은 각 원소에 **고유 태그(UID)** 를 부여해 이 문제를 해결한다. "보았던(observed) 태그만 제거"하므로 동시 추가·제거 충돌에서 **추가가 우선**된다.

```python
from typing import Set, Tuple
from dataclasses import dataclass, field
import uuid

@dataclass
class ORSet:
    """
    OR-Set (Observed-Remove Set): 동시 추가·제거 충돌을 해결하는 CRDT 집합.
    내부적으로 (값, 고유_태그) 쌍으로 원소를 관리한다.
    """
    added: Set[Tuple] = field(default_factory=set)    # {(value, uid), ...}
    removed: Set[Tuple] = field(default_factory=set)  # {(value, uid), ...}

    def add(self, value) -> str:
        uid = str(uuid.uuid4())
        self.added.add((value, uid))
        return uid

    def remove(self, value):
        # '현재 보이는' 태그들만 제거 대상으로 표시
        to_remove = {(v, uid) for (v, uid) in self.added if v == value}
        self.removed.update(to_remove)

    def contains(self, value) -> bool:
        effective = self.added - self.removed
        return any(v == value for (v, _) in effective)

    def elements(self) -> set:
        effective = self.added - self.removed
        return {v for (v, _) in effective}

    def merge(self, other: 'ORSet') -> 'ORSet':
        merged = ORSet()
        merged.added = self.added | other.added
        merged.removed = self.removed | other.removed
        return merged


# 시뮬레이션: 동시 추가·제거 충돌
replica1 = ORSet()
replica2 = ORSet()

# 두 레플리카에 "apple" 추가 (네트워크 분단 전 초기화)
uid = replica1.add("apple")
replica2.added = replica1.added.copy()  # 동기화된 초기 상태

# 네트워크 분단: 두 레플리카가 독립적으로 동작
replica1.remove("apple")                 # 레플리카1: apple 제거
replica2.add("apple")                    # 레플리카2: apple 다시 추가 (새 UID 발급)

print(f"레플리카1 'apple' 존재?: {replica1.contains('apple')}")  # False
print(f"레플리카2 'apple' 존재?: {replica2.contains('apple')}")  # True

# 병합 — OR-Set은 "추가 우선" 원칙
merged = replica1.merge(replica2)
print(f"병합 후 'apple' 존재?: {merged.contains('apple')}")  # True (추가 우선)
print(f"병합 후 원소 집합: {merged.elements()}")
```

---

## 주의사항 및 팁

### 1. CRDT의 단조 증가(Monotonic Growth) 문제

State-based CRDT는 상태가 절대 줄어들지 않는다. OR-Set의 `removed` 집합, PN-Counter의 `n` 카운터는 시간이 지날수록 커진다. 장기 운영 시스템에서는 **가비지 컬렉션(Tombstone GC)** 전략이 필요하다. 예를 들어 모든 레플리카가 제거를 확인한 tombstone은 안전하게 삭제할 수 있다.

### 2. 의미론적 충돌은 여전히 발생한다

CRDT는 **구조적 충돌**(동시 쓰기로 인한 데이터 불일치)은 해결하지만 **의미론적 충돌**(두 사람이 같은 제품의 가격을 동시에 다른 값으로 수정)은 해결하지 않는다. LWW(Last-Write-Wins) 전략은 하나를 버리고 하나를 선택할 뿐이다. 도메인에 맞는 병합 전략이 필요하다.

### 3. Delta-CRDT: 상태 기반 CRDT의 대역폭 최적화

전체 상태를 브로드캐스트하는 State-based CRDT는 상태가 커질수록 네트워크 부하가 증가한다. **Delta-CRDT**는 전체 상태 대신 "변경된 부분(delta)"만 전송한다. Riak, Akka Distributed Data가 이 방식을 사용한다.

### 4. CRDT와 OT의 선택 기준

| 기준 | CRDT | OT |
|------|------|----|
| 중앙 서버 필요 | 불필요 | 필요 |
| 오프라인 지원 | 우수 | 어려움 |
| 복잡도 | 낮음 (이론적) | 높음 |
| 텍스트 편집 | RGA/Logoot | 구글 Docs 방식 |
| 메모리 증가 | 있음 | 없음 |

---

## 참고 자료
- [CRDT.tech — About CRDTs 공식 사이트](https://crdt.tech/)
- [Wikipedia: Conflict-free replicated data type](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type)
- [Shapiro et al. (2011) — A Comprehensive Study of CRDTs (논문)](https://hal.inria.fr/inria-00555588/document)
- [Automerge — CRDT 기반 협업 편집 라이브러리](https://automerge.org/)
