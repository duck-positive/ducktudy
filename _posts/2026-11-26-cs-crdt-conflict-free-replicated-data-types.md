---
layout: post
title: "CRDT 완전 정복: 분산 시스템에서 충돌 없는 복제 데이터 타입으로 강한 최종 일관성 달성하기"
date: 2026-11-26
categories: [cs, computer-science]
tags: [CRDT, 분산시스템, 일관성, 동시성, 분산데이터베이스, 협업편집]
---

분산 시스템에서 여러 노드가 동일한 데이터를 동시에 수정하면 충돌이 발생합니다. 이를 해결하기 위한 기존 방법은 잠금(Lock)이나 중앙 조정자(Coordinator)에 의존했지만, 이는 가용성과 성능을 희생시킵니다. CRDT(Conflict-free Replicated Data Type)는 수학적 불변성을 이용해 충돌을 아예 발생하지 않게 설계된 자료구조입니다.

## CRDT란 무엇인가?

CRDT는 2011년 Marc Shapiro, Nuno Preguiça 등이 제안한 개념으로, 복수의 노드에서 독립적으로 수정된 복제본을 항상 병합할 수 있도록 수학적으로 설계된 자료구조입니다. 핵심 아이디어는 **단조 증가(monotonically increasing)** 하는 반격자(semilattice) 위에 연산을 정의하는 것입니다.

### CRDT의 두 가지 종류

**CvRDT(State-based CRDT)**: 전체 상태를 전송하고 병합 함수로 합칩니다. 병합 함수는 교환 법칙(Commutativity), 결합 법칙(Associativity), 멱등성(Idempotency)을 만족해야 합니다.

**CmRDT(Operation-based CRDT)**: 연산(operation)만 전달합니다. 연산이 교환 법칙을 만족하면 순서 무관하게 적용 가능합니다.

### 왜 CRDT가 필요한가?

CAP 정리에 따르면 분산 시스템은 네트워크 파티션 발생 시 일관성(C)과 가용성(A) 중 하나를 선택해야 합니다. CRDT는 **강한 최종 일관성(Strong Eventual Consistency, SEC)**을 보장합니다.

- 모든 올바른 복제본은 같은 업데이트 집합을 받으면 동일한 상태에 도달합니다.
- 잠금이나 중앙 조정자 없이 가용성을 극대화합니다.
- Google Docs, Figma, Notion 같은 실시간 협업 도구와 Redis, Riak 같은 분산 데이터베이스가 이를 활용합니다.

## 핵심 CRDT 구현 예제

### 1. G-Counter (Grow-only Counter)

G-Counter는 증가만 허용하는 가장 단순한 CRDT입니다. 각 노드는 자신의 카운터만 증가시키고, 전체 값은 모든 노드 카운터의 합으로 계산합니다.

```python
class GCounter:
    def __init__(self, node_id: str, num_nodes: int):
        self.node_id = node_id
        self.counters = [0] * num_nodes
        self.node_index = int(node_id.split('-')[1])

    def increment(self):
        self.counters[self.node_index] += 1

    def value(self) -> int:
        return sum(self.counters)

    def merge(self, other: 'GCounter') -> 'GCounter':
        # 병합: 각 노드의 최댓값을 선택 (반격자의 join 연산)
        result = GCounter(self.node_id, len(self.counters))
        result.counters = [
            max(a, b) for a, b in zip(self.counters, other.counters)
        ]
        return result

    def __repr__(self):
        return f"GCounter(node={self.node_id}, counters={self.counters}, value={self.value()})"


# 시뮬레이션: 두 노드가 동시에 카운터를 증가
node_a = GCounter("node-0", 3)
node_b = GCounter("node-1", 3)
node_c = GCounter("node-2", 3)

# 각 노드 독립 증가 (네트워크 파티션 상황 가정)
node_a.increment()
node_a.increment()  # node-0: 2번 증가
node_b.increment()  # node-1: 1번 증가
node_c.increment()
node_c.increment()
node_c.increment()  # node-2: 3번 증가

# 동기화: 순서 무관하게 병합해도 결과 동일
merged_ab = node_a.merge(node_b)
final = merged_ab.merge(node_c)
print(final)  # value = 6 (2 + 1 + 3)

# 반대 순서로 병합해도 동일한 결과 (교환 법칙)
merged_cb = node_c.merge(node_b)
final2 = merged_cb.merge(node_a)
print(final2)  # value = 6 (동일!)
```

### 2. 2P-Set (Two-Phase Set) — 삭제 가능한 집합

단순 G-Set(추가만 가능)에서 발전해, 추가 집합(A)과 삭제 집합(R)을 별도 관리하는 2P-Set입니다. 단, 한번 삭제된 원소는 다시 추가할 수 없다는 제약이 있습니다.

```typescript
class TwoPhaseSet<T> {
    private addSet: Set<T>;
    private removeSet: Set<T>;

    constructor(addSet: Set<T> = new Set(), removeSet: Set<T> = new Set()) {
        this.addSet = new Set(addSet);
        this.removeSet = new Set(removeSet);
    }

    add(element: T): void {
        this.addSet.add(element);
    }

    remove(element: T): void {
        // 삭제는 추가된 원소에만 가능
        if (this.addSet.has(element)) {
            this.removeSet.add(element);
        }
    }

    lookup(element: T): boolean {
        return this.addSet.has(element) && !this.removeSet.has(element);
    }

    // 반격자 join 연산 (합집합)
    merge(other: TwoPhaseSet<T>): TwoPhaseSet<T> {
        const newAdd = new Set([...this.addSet, ...other.addSet]);
        const newRemove = new Set([...this.removeSet, ...other.removeSet]);
        return new TwoPhaseSet(newAdd, newRemove);
    }

    getElements(): T[] {
        return [...this.addSet].filter(e => !this.removeSet.has(e));
    }
}

// 시뮬레이션
const setA = new TwoPhaseSet<string>();
const setB = new TwoPhaseSet<string>();

// 노드 A: "apple" 추가
setA.add("apple");
setA.add("banana");

// 노드 B: 동시에 "apple" 삭제 시도 + "cherry" 추가
// (setB는 아직 "apple"이 추가된 상태를 모름)
setB.add("apple");
setB.remove("apple"); // 삭제 집합에 추가
setB.add("cherry");

// 병합
const merged = setA.merge(setB);
console.log(merged.getElements());  // ["banana", "cherry"]
// "apple"은 removeSet에 있으므로 제외됨
console.log(merged.lookup("apple")); // false
```

### 3. OR-Set (Observed-Remove Set) — 더 직관적인 삭제

2P-Set의 한계(삭제 후 재추가 불가)를 해결한 OR-Set은 각 원소에 고유 태그를 부여합니다.

```python
import uuid
from typing import Dict, Set, Tuple

class ORSet:
    def __init__(self):
        # 원소별 고유 태그 집합
        self.entries: Dict[str, Set[str]] = {}     # 추가된 (원소, 태그) 쌍
        self.tombstones: Set[Tuple[str, str]] = set()  # 삭제된 (원소, 태그) 쌍

    def add(self, element: str) -> str:
        tag = str(uuid.uuid4())
        if element not in self.entries:
            self.entries[element] = set()
        self.entries[element].add(tag)
        return tag

    def remove(self, element: str):
        # 현재 보이는 모든 태그를 tombstone에 추가
        if element in self.entries:
            for tag in self.entries[element]:
                self.tombstones.add((element, tag))

    def lookup(self, element: str) -> bool:
        if element not in self.entries:
            return False
        active = self.entries[element] - {
            tag for (e, tag) in self.tombstones if e == element
        }
        return len(active) > 0

    def merge(self, other: 'ORSet') -> 'ORSet':
        result = ORSet()
        for elem, tags in self.entries.items():
            result.entries[elem] = set(tags)
        for elem, tags in other.entries.items():
            if elem not in result.entries:
                result.entries[elem] = set()
            result.entries[elem] |= tags
        result.tombstones = self.tombstones | other.tombstones
        return result

# 테스트: add-wins 의미론
node1 = ORSet()
node2 = ORSet()

node1.add("apple")
node2.add("apple")  # 별도 태그로 추가
node1.remove("apple")  # node1의 태그만 삭제

merged = node1.merge(node2)
print(merged.lookup("apple"))  # True! (node2의 태그는 살아있음)
```

## 실제 사용처와 선택 기준

### 언제 CRDT를 선택해야 하나?

| 사용 사례 | 적합한 CRDT | 이유 |
|---|---|---|
| 분산 카운터 (좋아요 수) | G-Counter / PN-Counter | 증감만 필요 |
| 실시간 협업 편집 | RGA, LSEQ, Logoot | 순서 있는 시퀀스 |
| 분산 태그/레이블 | OR-Set | 추가/삭제 가능 집합 |
| 분산 설정 저장 | LWW-Register | 최신 값이 우선 |
| 쇼핑 카트 | OR-Set | 항목 추가/삭제 |

## 주의사항과 팁

**1. 의미론적 충돌은 여전히 존재합니다.** CRDT는 구조적 충돌을 없애지만, 비즈니스 로직 충돌(예: 재고 0인데 동시에 2명이 주문)은 여전히 애플리케이션 레벨에서 처리해야 합니다.

**2. 삭제는 여전히 까다롭습니다.** Tombstone이 무한 증가하는 문제가 있어, 주기적인 가비지 컬렉션이 필요합니다. 모든 노드가 동기화된 후에만 tombstone을 제거할 수 있습니다.

**3. 상태 기반 CRDT는 대역폭을 많이 사용합니다.** 큰 자료구조일수록 전체 상태 전송 비용이 큽니다. Delta-CRDT를 사용해 변경분만 전송하세요.

**4. 성능 트레이드오프를 이해하세요.** CRDT는 최종 일관성을 제공하지만, 선형화 가능성(Linearizability)을 요구하는 금융 거래 등에는 적합하지 않습니다.

**5. 실용적인 라이브러리를 활용하세요.** 직접 구현보다는 Automerge(JavaScript/Rust), Yjs(JavaScript), Diamond Types 같은 검증된 라이브러리를 사용하는 것이 안전합니다.

## 참고 자료
- [Wikipedia: Conflict-free replicated data type](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type)
- [Conflict-free Replicated Data Types: An Overview (arXiv)](https://arxiv.org/pdf/1806.10254)
- [EnterpriseDB: CRDT Column Types in PGD](https://www.enterprisedb.com/docs/pgd/5.9/conflict-management/crdt/)
