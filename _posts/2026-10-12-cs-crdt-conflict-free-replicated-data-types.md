---
layout: post
title: "CRDT 완전 정복: 분산 시스템에서 충돌 없이 데이터를 병합하는 수학적 기법"
date: 2026-10-12
categories: [cs, computer-science]
tags: [crdt, distributed-systems, eventual-consistency, cap-theorem, collaborative-editing, conflict-resolution]
---

## CRDT란 무엇인가?

Google Docs에서 두 명이 동시에 같은 문서를 편집할 때 어떻게 충돌 없이 변경사항이 합쳐지는 걸까? Figma에서 여러 디자이너가 동시에 요소를 수정해도 왜 덮어쓰기가 일어나지 않을까? 그 핵심에는 **CRDT(Conflict-free Replicated Data Type, 충돌 없는 복제 데이터 타입)**가 있다.

CRDT는 **분산 시스템에서 여러 노드가 독립적으로 데이터를 수정한 후, 통신이 재개되면 자동으로 일관된 상태로 수렴하도록 설계된 데이터 타입**이다. 락(lock)도, 중앙 조정자도, 충돌 해결 로직도 필요 없다. 수학적으로 보장된 수렴이 이루어진다.

2011년 Marc Shapiro, Nuno Preguiça 등이 발표한 이 개념은 현재 Redis, Riak, Apple Notes, Atom 에디터, 협업 코딩 도구 등 수많은 시스템의 핵심 기술이 되었다.

---

## 왜 CRDT가 필요한가?

### CAP 정리와 분산 시스템의 딜레마

CAP 정리에 따르면 네트워크 파티션이 발생한 분산 시스템은 **일관성(Consistency)**과 **가용성(Availability)** 중 하나를 선택해야 한다. 은행 계좌나 재고 시스템처럼 강한 일관성이 필요한 경우는 Paxos나 Raft 같은 합의 알고리즘을 사용하지만, 이는 **레이턴시**와 **가용성**을 희생한다. 파티션이 발생하면 쓰기가 차단된다.

반면 협업 문서 편집, 쇼핑 카트, 소셜 미디어 좋아요 수 같은 경우는 **최종 일관성(Eventual Consistency)**으로 충분하다. 잠깐 다른 상태를 보더라도 결국 모든 복제본이 동일한 값으로 수렴하면 된다.

### 기존 최종 일관성의 문제점

단순한 최종 일관성 시스템은 **동시 수정 시 충돌**을 야기한다:

```
시나리오: 두 노드가 동시에 카운터를 증가
초기값: counter = 5

노드 A: counter = counter + 2  → counter = 7
노드 B: counter = counter + 3  → counter = 8

병합 전략이 없으면: counter = 7 또는 8 (하나가 유실됨)
올바른 결과: counter = 10
```

Last-Write-Wins(LWW) 같은 단순 전략은 데이터 유실을 일으킨다. CRDT는 **수학적 구조**를 통해 이 문제를 원천적으로 해결한다.

---

## CRDT의 두 가지 종류

### 1. State-based CRDT (CvRDT — Convergent)
노드 간에 **완전한 상태(state)**를 전송하고 병합한다. 병합 함수는 **결합법칙(associative), 교환법칙(commutative), 멱등법칙(idempotent)**을 만족해야 한다. 수학적으로는 **조인 반격자(join-semilattice)** 구조를 따른다.

- 장점: 메시지 손실에 강함 (재전송만 하면 됨)
- 단점: 전체 상태를 전송하므로 네트워크 트래픽이 큼

### 2. Operation-based CRDT (CmRDT — Commutative)
노드 간에 **연산(operation)**을 전송한다. 연산들이 **교환 가능(commutative)**해야 최종 결과가 수렴한다.

- 장점: 작은 메시지 크기
- 단점: 메시지 정확히 한 번 전달(exactly-once delivery) 보장이 필요

---

## 실제 구현 예제

### 예제 1: G-Counter, PN-Counter, G-Set 구현 (Python)

```python
from __future__ import annotations
from dataclasses import dataclass, field
from typing import Dict, Set, TypeVar

# ────────────────────────────────────────
# G-Counter (Grow-only Counter)
# 증가만 가능한 카운터. 각 노드 ID를 키로, 해당 노드의 최대 증가량을 저장.
# ────────────────────────────────────────
@dataclass
class GCounter:
    node_id: str
    counts: Dict[str, int] = field(default_factory=dict)

    def increment(self, amount: int = 1):
        self.counts[self.node_id] = self.counts.get(self.node_id, 0) + amount

    def value(self) -> int:
        return sum(self.counts.values())

    def merge(self, other: GCounter) -> GCounter:
        """두 G-Counter 병합: 각 노드별 최댓값을 취함 (교환·결합·멱등 법칙 모두 만족)"""
        merged_counts = {**self.counts}
        for node, count in other.counts.items():
            merged_counts[node] = max(merged_counts.get(node, 0), count)
        result = GCounter(self.node_id)
        result.counts = merged_counts
        return result


# ────────────────────────────────────────
# PN-Counter (Positive-Negative Counter)
# 증가와 감소 모두 지원. 내부적으로 P (positive) + N (negative) 두 G-Counter 사용.
# ────────────────────────────────────────
@dataclass
class PNCounter:
    node_id: str
    positive: GCounter = None
    negative: GCounter = None

    def __post_init__(self):
        if self.positive is None:
            self.positive = GCounter(self.node_id)
        if self.negative is None:
            self.negative = GCounter(self.node_id)

    def increment(self, amount: int = 1):
        self.positive.increment(amount)

    def decrement(self, amount: int = 1):
        self.negative.increment(amount)

    def value(self) -> int:
        return self.positive.value() - self.negative.value()

    def merge(self, other: PNCounter) -> PNCounter:
        result = PNCounter(self.node_id)
        result.positive = self.positive.merge(other.positive)
        result.negative = self.negative.merge(other.negative)
        return result


# ────────────────────────────────────────
# G-Set (Grow-only Set)
# 원소 추가만 지원하는 집합. 삭제 불가.
# ────────────────────────────────────────
T = TypeVar('T')

@dataclass
class GSet:
    elements: Set = field(default_factory=set)

    def add(self, element) -> None:
        self.elements.add(element)

    def contains(self, element) -> bool:
        return element in self.elements

    def merge(self, other: GSet) -> GSet:
        result = GSet()
        result.elements = self.elements | other.elements  # 합집합: 교환·결합·멱등 만족
        return result


# ────────────────────────────────────────
# 2P-Set (Two-Phase Set)
# 추가와 삭제 지원. 단, 한번 삭제된 원소는 재추가 불가.
# ────────────────────────────────────────
@dataclass
class TwoPhaseSet:
    added: GSet = field(default_factory=GSet)
    removed: GSet = field(default_factory=GSet)

    def add(self, element) -> None:
        if not self.removed.contains(element):
            self.added.add(element)

    def remove(self, element) -> None:
        if self.added.contains(element):
            self.removed.add(element)

    def contains(self, element) -> bool:
        return self.added.contains(element) and not self.removed.contains(element)

    def merge(self, other: TwoPhaseSet) -> TwoPhaseSet:
        result = TwoPhaseSet()
        result.added = self.added.merge(other.added)
        result.removed = self.removed.merge(other.removed)
        return result


# ── 시나리오 테스트 ──────────────────────────
print("=== G-Counter 테스트 ===")
counter_a = GCounter("node-A")
counter_b = GCounter("node-B")

# 오프라인 상태에서 각자 증가
counter_a.increment(5)
counter_b.increment(3)
counter_b.increment(2)

print(f"노드 A의 카운터: {counter_a.value()}")  # 5
print(f"노드 B의 카운터: {counter_b.value()}")  # 5

# 재연결 후 병합
merged = counter_a.merge(counter_b)
print(f"병합 후 카운터: {merged.value()}")  # 10 (데이터 유실 없음!)

print("\n=== PN-Counter 테스트 ===")
likes_a = PNCounter("node-A")  # 서울 서버
likes_b = PNCounter("node-B")  # 도쿄 서버

likes_a.increment(10)  # 좋아요 10개 추가
likes_b.increment(5)   # 좋아요 5개 추가
likes_a.decrement(2)   # 좋아요 취소 2개

merged_likes = likes_a.merge(likes_b)
print(f"최종 좋아요 수: {merged_likes.value()}")  # 13

print("\n=== 2P-Set 테스트 ===")
cart_a = TwoPhaseSet()  # 클라이언트 A
cart_b = TwoPhaseSet()  # 클라이언트 B (오프라인)

cart_a.add("apple")
cart_a.add("banana")
cart_b.add("banana")   # 동시에 추가 (중복 없음)
cart_b.add("cherry")
cart_a.remove("banana")

merged_cart = cart_a.merge(cart_b)
print(f"apple: {merged_cart.contains('apple')}")   # True
print(f"banana: {merged_cart.contains('banana')}")  # False (삭제됨)
print(f"cherry: {merged_cart.contains('cherry')}")  # True
```

---

### 예제 2: LWW-Register와 OR-Set으로 협업 편집기 구현 (Python)

```python
import time
from dataclasses import dataclass, field
from typing import Dict, Tuple, Set
import uuid

# ────────────────────────────────────────
# LWW-Register (Last-Write-Wins Register)
# 타임스탬프가 가장 큰 값이 최종 값이 됨
# ────────────────────────────────────────
@dataclass
class LWWRegister:
    node_id: str
    _value: object = None
    _timestamp: float = 0.0

    def write(self, value) -> None:
        self._value = value
        self._timestamp = time.time()

    def read(self):
        return self._value

    def merge(self, other: LWWRegister) -> LWWRegister:
        result = LWWRegister(self.node_id)
        if self._timestamp >= other._timestamp:
            result._value = self._value
            result._timestamp = self._timestamp
        else:
            result._value = other._value
            result._timestamp = other._timestamp
        return result


# ────────────────────────────────────────
# OR-Set (Observed-Remove Set)
# 2P-Set의 "한번 삭제하면 재추가 불가" 제한을 해결.
# 각 추가 연산에 고유 태그(UUID)를 부여. 삭제는 해당 태그만 제거.
# ────────────────────────────────────────
@dataclass
class ORSet:
    """
    내부: {element: {tag1, tag2, ...}}
    태그 집합이 비어있으면 원소가 없는 것.
    병합: 각 원소의 태그 집합의 합집합.
    """
    _elements: Dict[str, Set[str]] = field(default_factory=dict)

    def add(self, element: str) -> str:
        """새 태그(UUID)로 원소를 추가. 재추가 시에도 새 태그가 생성됨."""
        tag = str(uuid.uuid4())
        if element not in self._elements:
            self._elements[element] = set()
        self._elements[element].add(tag)
        return tag

    def remove(self, element: str) -> None:
        """현재 알려진 태그들을 모두 제거 (현재 관찰된 것만 삭제)"""
        if element in self._elements:
            self._elements[element].clear()

    def contains(self, element: str) -> bool:
        return bool(self._elements.get(element))

    def elements(self) -> Set[str]:
        return {e for e, tags in self._elements.items() if tags}

    def merge(self, other: ORSet) -> ORSet:
        result = ORSet()
        all_keys = set(self._elements) | set(other._elements)
        for key in all_keys:
            tags_a = self._elements.get(key, set())
            tags_b = other._elements.get(key, set())
            result._elements[key] = tags_a | tags_b
        return result


# ── 시나리오: OR-Set이 2P-Set의 한계를 극복하는 방법 ──
print("=== OR-Set: 삭제 후 재추가 테스트 ===")
set_a = ORSet()  # 사용자 A (온라인)
set_b = ORSet()  # 사용자 B (오프라인)

# 초기 동기화 상태: 둘 다 "task-1"을 갖고 있음
set_a.add("task-1")
set_b._elements = {k: set(v) for k, v in set_a._elements.items()}  # 동기화

# 네트워크 파티션 발생
# A: task-1 삭제 후 재추가
set_a.remove("task-1")
set_a.add("task-1")  # 새 UUID 태그로 재추가

# B: task-1에 새 태그 추가 (삭제 모름)
set_b.add("task-1")

# 재연결 후 병합
merged = set_a.merge(set_b)
print(f"task-1 존재: {merged.contains('task-1')}")   # True (두 노드 모두 "있음"으로 수렴)
print(f"현재 원소들: {merged.elements()}")


print("\n=== LWW-Register: 협업 편집기 필드 ===")
# 두 편집자가 동시에 문서 제목 수정
editor_a = LWWRegister("editor-A")
editor_b = LWWRegister("editor-B")

editor_a.write("제목: 분산 시스템 가이드")
time.sleep(0.001)  # 약간의 시간 차이
editor_b.write("제목: 분산 시스템 완전 정복")

merged_title = editor_a.merge(editor_b)
print(f"최종 제목: {merged_title.read()}")
# "제목: 분산 시스템 완전 정복" (더 나중에 쓴 B의 값이 우선)


print("\n=== CRDT vs 낙관적 잠금 비교 ===")
# CRDT: 오프라인에서 각자 카운터 증가 후 병합
c1, c2 = GCounter("region-us"), GCounter("region-eu")
c1.increment(1000); c2.increment(2000)
total = c1.merge(c2).value()
print(f"CRDT 합산 (데이터 유실 없음): {total}")  # 3000

# 낙관적 잠금: 동시 수정 시 충돌 발생
last_known = 0
result_a = last_known + 1000  # CAS 성공
result_b = last_known + 2000  # CAS 실패 → 재시도 필요
print(f"낙관적 잠금 (한쪽만 성공): {max(result_a, result_b)}")  # 2000 (1000 유실!)
```

---

## 주의사항 및 실전 팁

### 1. 모든 데이터 타입이 CRDT가 될 수 있는 건 아니다
CRDT는 교환·결합·멱등 법칙을 만족하는 병합 함수가 존재해야 한다. 예를 들어 "순서가 보장된 작업 큐"나 "유일한 ID 할당"은 CRDT로 표현하기 매우 어렵다. 강한 순서 보장이 필요하다면 Paxos/Raft를 사용하라.

### 2. OR-Set의 "동시 추가/삭제" 의미론
OR-Set에서 동시 추가와 삭제는 **추가 우선(add-wins)** 정책을 따른다. 이는 데이터 손실보다 데이터 유지를 선호하는 협업 도구에 적합하다. 반대 정책이 필요하면 RR-Set(Remove-wins)을 사용하면 되지만, 설계가 더 복잡해진다.

### 3. 메모리와 네트워크 오버헤드
OR-Set의 UUID 태그, G-Counter의 노드별 카운터 등 CRDT는 **메타데이터(tombstone)** 를 영구 보존한다. 장기 운영 시스템에서는 가비지 컬렉션 메커니즘이 필요하다. 이를 위해 정기적인 스냅샷과 압축(compaction) 정책을 설계해야 한다.

### 4. Delta-CRDT로 네트워크 효율 개선
State-based CRDT의 전체 상태 전송 문제를 해결하기 위해 **Delta-CRDT**가 제안되었다. 마지막 동기화 이후 변경된 부분(delta)만 전송한다. 이는 State-based의 단순함과 Operation-based의 효율성을 결합한다.

### 5. 실제 사용처
- **Redis**: CRDT 기반의 Multi-master 복제 (Redis Enterprise)
- **Riak**: Riak DT (G-Counter, PN-Counter, OR-Set, Map)
- **Apple Notes**: iCloud 동기화
- **Automerge/Yjs**: JSON CRDT 라이브러리 (Google Docs 스타일 협업 편집)
- **Figma**: 디자인 도구의 실시간 협업

---

## 마무리

CRDT는 분산 시스템에서 합의(consensus)라는 값비싼 연산 없이 일관성을 달성하는 우아한 수학적 해법이다. 락이나 트랜잭션 없이 여러 노드가 자유롭게 쓰고 나중에 자동으로 수렴하는 이 접근 방식은, 오프라인 지원이 필요한 모바일 앱, 지리적으로 분산된 멀티 리전 서비스, 실시간 협업 도구에서 특히 빛을 발한다.

핵심은 **"충돌을 해결하지 않고, 애초에 충돌이 발생하지 않는 자료구조를 설계한다"**는 사고의 전환이다.

---

## 참고 자료
- [Conflict-free Replicated Data Types: An Overview (arXiv, 2018)](https://arxiv.org/abs/1806.10254)
- [Conflict-free Replicated Data Types (arXiv, 2011 — Shapiro et al.)](https://arxiv.org/abs/1805.06358)
- [CRDT — TechTarget 정의](https://www.techtarget.com/it-strategy/definition/conflict-free-replicated-data-type-CRDT)
- [Automerge: JSON CRDT 라이브러리](https://automerge.org/)
