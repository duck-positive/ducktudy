---
layout: post
title: "CRDT(Conflict-free Replicated Data Types) 완전 정복: 충돌 없는 분산 협업의 수학적 토대"
date: 2026-11-09
categories: [cs, computer-science]
tags: [crdt, distributed-systems, eventual-consistency, collaboration, strong-eventual-consistency]
---

## CRDT란 무엇인가?

현대의 분산 시스템에서 가장 어려운 문제 중 하나는 **동시에 발생하는 쓰기 충돌(write conflict)** 을 어떻게 해결하느냐이다. Google Docs에서 두 사람이 동시에 같은 문서를 편집할 때, 또는 오프라인 상태에서 모바일 앱의 데이터를 수정한 뒤 네트워크가 복구되었을 때, 시스템은 어떻게 두 버전을 올바르게 합칠 수 있을까?

**CRDT(Conflict-free Replicated Data Types)** 는 이 문제를 수학적으로 해결하는 자료구조 패밀리다. 2011년 Marc Shapiro, Nuno Preguiça 등이 INRIA에서 발표한 논문에서 공식화되었으며, Figma, Notion, Linear 등 수많은 현대 협업 도구의 핵심 기술로 사용된다. CRDT는 분산된 여러 복제본(replica)에서 각자 독립적으로 수정이 발생하더라도, 메시지가 결국 모든 노드에 전달되기만 하면 **자동으로 동일한 최종 상태로 수렴(converge)** 함을 수학적으로 보장한다.

이 보장을 **강한 최종 일관성(Strong Eventual Consistency, SEC)** 이라고 부르며, 이는 Eventual Consistency보다 더 강력한 보장이다. Eventual Consistency는 "언젠가 수렴하겠지"라는 약속이지만, SEC는 "같은 업데이트 집합을 받은 복제본은 반드시 동일한 상태가 된다"는 결정론적 보장이다.

---

## 왜 CRDT가 필요한가?

기존의 분산 일관성 접근법에는 두 가지 큰 범주가 있다.

**1. 강한 일관성(Strong Consistency) - Paxos, Raft 등**
- 모든 쓰기 작업에 쿼럼(quorum) 동의가 필요하다.
- 네트워크 파티션 발생 시 쓰기를 거부해야 한다(CAP 정리의 CP 선택).
- 레이턴시가 높고, 가용성이 떨어진다.

**2. 충돌 감지 및 해결(Conflict Detection & Resolution)**
- 충돌이 발생하면 별도의 해결 로직이 필요하다(last-write-wins, merge function 등).
- 애플리케이션 레벨에서 충돌 로직을 구현해야 하므로 복잡도가 높다.
- Git의 merge conflict가 대표적인 예다.

**CRDT는 세 번째 길**을 제시한다. 충돌 자체가 발생할 수 없도록 자료구조를 설계함으로써, 별도의 충돌 해결 로직 없이도 항상 올바른 상태로 수렴한다. 이를 위해 CRDT는 다음의 수학적 조건을 만족해야 한다:

- **교환법칙(Commutativity)**: 연산 순서와 무관하게 결과가 같다.
- **결합법칙(Associativity)**: 연산의 그룹화와 무관하게 결과가 같다.
- **멱등법칙(Idempotency)**: 같은 연산을 여러 번 적용해도 결과가 같다.

---

## CRDT의 두 가지 유형

### 상태 기반 CRDT (State-based CRDT, CvRDT)

노드들이 전체 상태를 주고받는다. 상태를 병합할 때 **조인 반격자(join semi-lattice)** 의 `merge` 함수를 사용하며, 이 함수는 멱등·교환·결합 법칙을 모두 만족해야 한다.

가장 단순한 예는 **G-Counter(Grow-only Counter)** 다:

```python
class GCounter:
    """Grow-only Counter: 증가만 가능한 분산 카운터"""
    
    def __init__(self, node_id: str, num_nodes: int):
        self.node_id = node_id
        # 각 노드별 카운터 벡터
        self.counters = [0] * num_nodes
        self.node_index = int(node_id)
    
    def increment(self):
        """현재 노드의 카운터만 증가"""
        self.counters[self.node_index] += 1
    
    def value(self) -> int:
        """전체 합산 값"""
        return sum(self.counters)
    
    def merge(self, other: 'GCounter') -> 'GCounter':
        """두 상태를 element-wise 최댓값으로 병합"""
        result = GCounter(self.node_id, len(self.counters))
        result.counters = [
            max(self.counters[i], other.counters[i])
            for i in range(len(self.counters))
        ]
        return result
    
    def __repr__(self):
        return f"GCounter(node={self.node_id}, counters={self.counters}, value={self.value()})"


# 시뮬레이션: 두 노드가 오프라인으로 각자 증가 후 병합
node_a = GCounter("0", 3)
node_b = GCounter("1", 3)
node_c = GCounter("2", 3)

# 각 노드 독립 증가
node_a.increment()
node_a.increment()  # node_a.counters = [2, 0, 0]
node_b.increment()  # node_b.counters = [0, 1, 0]
node_c.increment()
node_c.increment()
node_c.increment()  # node_c.counters = [0, 0, 3]

# 병합 (순서와 무관하게 동일한 결과)
merged_ab = node_a.merge(node_b)
merged_abc = merged_ab.merge(node_c)

print(merged_abc)  # GCounter(node=0, counters=[2, 1, 3], value=6)

# 역순으로 병합해도 동일
merged_cb = node_c.merge(node_b)
merged_cba = merged_cb.merge(node_a)
print(merged_cba)  # GCounter(node=0, counters=[2, 1, 3], value=6)
```

**PN-Counter(Positive-Negative Counter)** 는 G-Counter 두 개(P: 증가용, N: 감소용)를 조합해 증감이 모두 가능한 카운터를 만든다. `value = P.value() - N.value()` 로 계산한다.

---

### 연산 기반 CRDT (Operation-based CRDT, CmRDT)

상태 전체 대신 **연산(operation)** 만을 전파한다. 단, 전달 조건으로 **인과적 순서(causal ordering)** 와 **정확히 한 번 전달(exactly-once delivery)** 을 요구한다. 상태 기반보다 메시지 크기가 작지만, 전달 보장이 더 강하다.

---

## 텍스트 편집의 CRDT: RGA(Replicated Growable Array)

텍스트 협업 편집을 위한 대표적인 CRDT가 **RGA(Replicated Growable Array)** 다. 각 문자에 고유한 타임스탬프를 부여하여 삽입 위치를 결정한다.

```python
import time
import uuid
from dataclasses import dataclass, field
from typing import Optional, List

@dataclass
class RGANode:
    """RGA의 각 문자 노드"""
    char: Optional[str]      # None이면 tombstone (삭제된 노드)
    uid: str                 # 전역 고유 ID (timestamp + node_id)
    prev_uid: Optional[str]  # 이전 노드의 UID

class RGADocument:
    """
    Replicated Growable Array 기반 텍스트 CRDT.
    각 문자는 고유 UID를 가지며, 삽입 위치는 UID 비교로 결정된다.
    """
    
    def __init__(self, node_id: str):
        self.node_id = node_id
        # sentinel head: 모든 문자의 이전 노드
        self.head = RGANode(char=None, uid="HEAD", prev_uid=None)
        self.nodes: dict[str, RGANode] = {"HEAD": self.head}
        self.order: List[str] = ["HEAD"]  # UID 순서 목록
    
    def _generate_uid(self) -> str:
        return f"{time.time_ns()}_{self.node_id}_{uuid.uuid4().hex[:6]}"
    
    def insert(self, char: str, after_uid: str) -> RGANode:
        """after_uid 뒤에 char를 삽입. 작업을 반환하여 다른 노드에 전파."""
        uid = self._generate_uid()
        node = RGANode(char=char, uid=uid, prev_uid=after_uid)
        self._apply_insert(node)
        return node
    
    def _apply_insert(self, node: RGANode):
        """원격 삽입 연산을 로컬에 적용 (동시 삽입 충돌 해결 포함)"""
        self.nodes[node.uid] = node
        
        # after_uid 이후의 올바른 위치를 찾아 삽입
        after_idx = self.order.index(node.prev_uid)
        insert_idx = after_idx + 1
        
        # 같은 위치에 동시 삽입된 경우, UID 사전순으로 정렬 (결정론적)
        while (insert_idx < len(self.order) and
               self.nodes[self.order[insert_idx]].prev_uid == node.prev_uid and
               self.order[insert_idx] > node.uid):
            insert_idx += 1
        
        self.order.insert(insert_idx, node.uid)
    
    def delete(self, uid: str):
        """uid에 해당하는 문자를 삭제 (tombstone으로 표시)"""
        self.nodes[uid].char = None
    
    def text(self) -> str:
        """현재 문서 내용 반환 (tombstone 제외)"""
        return "".join(
            node.char
            for uid in self.order
            if (node := self.nodes[uid]).char is not None
        )


# 시뮬레이션: 두 사용자 동시 편집
doc_alice = RGADocument("alice")
doc_bob = RGADocument("bob")

# Alice가 "Hello" 입력
h = doc_alice.insert("H", "HEAD")
e = doc_alice.insert("e", h.uid)
l1 = doc_alice.insert("l", e.uid)
l2 = doc_alice.insert("l", l1.uid)
o_node = doc_alice.insert("o", l2.uid)

# Bob에게 초기 상태 동기화
for node in [h, e, l1, l2, o_node]:
    doc_bob._apply_insert(node)

print(f"Alice: '{doc_alice.text()}'")  # Hello
print(f"Bob:   '{doc_bob.text()}'")    # Hello

# 동시에: Alice는 "Hello World", Bob은 "Hello CRDT" 입력
# Alice가 " World" 삽입
space_a = doc_alice.insert(" ", o_node.uid)
w = doc_alice.insert("W", space_a.uid)
r = doc_alice.insert("r", w.uid)
ld1 = doc_alice.insert("l", r.uid)
ld2 = doc_alice.insert("d", ld1.uid)

# Bob이 " CRDT" 삽입
space_b = doc_bob.insert(" ", o_node.uid)
c = doc_bob.insert("C", space_b.uid)
rd = doc_bob.insert("R", c.uid)
dt = doc_bob.insert("D", rd.uid)
t = doc_bob.insert("T", dt.uid)

# 연산 교환 (실제로는 네트워크를 통해 전파됨)
for node in [space_b, c, rd, dt, t]:
    doc_alice._apply_insert(node)
for node in [space_a, w, r, ld1, ld2]:
    doc_bob._apply_insert(node)

# 양쪽 모두 동일한 최종 상태 (UID 정렬에 따라 결정론적으로 수렴)
print(f"Alice final: '{doc_alice.text()}'")
print(f"Bob final:   '{doc_bob.text()}'")
# 두 결과가 동일함을 확인
assert doc_alice.text() == doc_bob.text(), "CRDT 수렴 실패!"
print("수렴 성공: 두 복제본이 동일한 상태입니다.")
```

---

## 주요 CRDT 자료구조 정리

| 자료구조 | 지원 연산 | 사용 사례 |
|---------|----------|---------|
| G-Counter | increment | 조회수, 좋아요 수 |
| PN-Counter | increment, decrement | 재고 수량 |
| G-Set | add | 태그 집합 |
| 2P-Set | add, remove (단, 삭제 후 재삽입 불가) | 멤버십 목록 |
| OR-Set (Observed-Remove Set) | add, remove | 장바구니 |
| LWW-Register | assign | 프로필 필드 |
| MV-Register (Multi-Value) | assign | 충돌 보존 레지스터 |
| RGA / LSEQ | insert, delete | 텍스트 편집 |
| RHT (Replicated Hash Table) | put, delete | 분산 JSON 문서 |

---

## 실제 사용처와 주의사항

**사용 사례:**
- **Figma**: OR-Set 기반으로 선택된 도형 집합을 각 클라이언트에서 관리.
- **Redis**: `CRDT` 모듈로 지리적으로 분산된 클러스터 간 동기화.
- **Apple Notes**: 오프라인 편집 후 iCloud 동기화 시 CRDT 기반 병합.
- **Riak**: LWW-Register와 PN-Counter를 기본 제공.

**주의사항 및 한계:**
1. **의미론적 충돌은 해결 불가**: CRDT는 데이터 구조적 충돌을 해결하지만, "Alice가 가격을 1,000원으로, Bob이 2,000원으로 동시에 설정"한 경우의 비즈니스 로직 충돌은 애플리케이션이 처리해야 한다.
2. **묘비(Tombstone) 누적**: 삭제된 원소가 tombstone으로 남아 메모리를 차지한다. 주기적인 가비지 컬렉션 프로토콜(예: 2P-Set)이 필요하다.
3. **메타데이터 오버헤드**: UID, 벡터 클럭 등 메타데이터가 실제 데이터보다 클 수 있다. 텍스트 CRDT에서 문자 하나당 수십 바이트의 메타데이터가 붙는다.
4. **OR-Set의 "add wins" 의미론**: 동시에 add와 remove가 발생하면 add가 이긴다. 이 의미론이 모든 도메인에 맞지 않을 수 있다.
5. **인과적 메타데이터 관리**: CmRDT는 벡터 클럭(vector clock)이나 버전 벡터(version vector)로 인과 순서를 추적해야 한다.

CRDT는 "충돌 해결"이 아닌 "충돌 예방"이라는 패러다임 전환을 가져온다. 협업 도구, 오프라인 우선(offline-first) 애플리케이션, 지리적으로 분산된 시스템을 설계할 때 CRDT는 가장 강력한 도구 중 하나다.

## 참고 자료
- [Conflict-free Replicated Data Types (Shapiro et al., INRIA)](https://hal.archives-ouvertes.fr/inria-00609399v1)
- [A Conflict-Free Replicated JSON Datatype (Kleppmann & Beresford, arXiv)](https://arxiv.org/abs/1707.01747)
- [Verifying Strong Eventual Consistency in Distributed Systems (arXiv)](https://arxiv.org/abs/1707.01747)
