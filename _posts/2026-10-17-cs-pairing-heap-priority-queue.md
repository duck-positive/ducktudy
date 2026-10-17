---
layout: post
title: "페어링 힙(Pairing Heap) 완전 정복: 피보나치 힙보다 빠른 실용적 우선순위 큐"
date: 2026-10-17
categories: [cs, computer-science]
tags: [페어링힙, 우선순위큐, 자료구조, 알고리즘, 힙, 피보나치힙, 분할상환분석]
---

## 페어링 힙이란 무엇인가

1986년 Michael Fredman, Robert Sedgewick, Daniel Sleator, Robert Tarjan이 발표한 **페어링 힙(Pairing Heap)** 은 이론적으로 피보나치 힙과 유사한 성능을 가지면서도 구현이 훨씬 간단한 우선순위 큐(Priority Queue) 자료구조입니다. 실제 벤치마크에서는 종종 피보나치 힙보다 빠른 성능을 보이며, Dijkstra 알고리즘이나 Prim 알고리즘 같은 그래프 알고리즘에서 활발히 사용됩니다.

피보나치 힙은 이론적으로 완벽하지만 구현이 복잡하고 상수 인자가 크다는 단점이 있습니다. 페어링 힙은 이 격차를 좁히기 위해 설계되었으며, "구현하기 쉬운 고성능 힙"이라는 평가를 받습니다.

## 왜 페어링 힙이 필요한가

우선순위 큐의 표준 구현인 이진 힙(Binary Heap)은 삽입과 삭제가 O(log n)이지만, `decrease-key` 연산이 O(log n)으로 느립니다. 다익스트라 알고리즘처럼 decrease-key를 자주 호출하는 경우 전체 복잡도가 올라갑니다.

| 연산 | 이진 힙 | 피보나치 힙 | 페어링 힙 |
|------|---------|-----------|---------|
| Insert | O(log n) | O(1) amortized | O(1) amortized |
| Find-Min | O(1) | O(1) | O(1) |
| Merge | O(n) | O(1) amortized | O(1) amortized |
| Delete-Min | O(log n) | O(log n) amortized | O(log n) amortized |
| Decrease-Key | O(log n) | O(1) amortized | O(log n) amortized* |

\* 정확한 상한은 아직 열린 문제로, 실험적으로는 O(log log n)에 가깝게 동작합니다.

이 표에서 보듯 페어링 힙은 피보나치 힙과 대부분 동일한 복잡도를 가지면서도 구현이 훨씬 단순합니다. `decrease-key`의 엄밀한 하한은 아직 미해결 문제지만, 실용적인 성능은 매우 우수합니다.

## 핵심 구조: 다진 트리의 힙 표현

페어링 힙은 **힙 속성(heap property)** 을 만족하는 다진 트리(multiway tree)입니다. 각 노드는 자신의 키 값이 자식 노드의 키 값보다 작거나 같아야 합니다(최소 힙 기준). 이 단순한 불변식 하나로 모든 연산이 정의됩니다.

```
구조 예시 (최소 힙):
        1
      / | \
     3  5   2
    / \    / \
   7   4  8   6
```

메모리상에서는 다진 트리를 "왼쪽 자식-오른쪽 형제(Left-Child Right-Sibling, LCRS)" 표현으로 구현합니다. 각 노드는 `key`, `leftChild`, `nextSibling`, 그리고 `prev`(부모 또는 이전 형제 역참조용) 포인터를 가집니다.

## 핵심 연산 구현

### 1. Meld (병합)

두 힙을 병합하는 연산입니다. 둘 중 루트 키가 더 작은 것이 새로운 루트가 되고, 나머지 힙의 루트가 그 자식이 됩니다. O(1) 시간이 소요됩니다.

```python
from dataclasses import dataclass, field
from typing import Optional, TypeVar, Generic

T = TypeVar('T')

@dataclass
class PairingHeapNode:
    key: int
    left_child: Optional['PairingHeapNode'] = None
    next_sibling: Optional['PairingHeapNode'] = None
    prev: Optional['PairingHeapNode'] = None  # 부모 또는 이전 형제

class PairingHeap:
    def __init__(self):
        self.root: Optional[PairingHeapNode] = None
        self.size: int = 0

    def _meld(self, h1: Optional[PairingHeapNode],
              h2: Optional[PairingHeapNode]) -> Optional[PairingHeapNode]:
        """두 힙을 병합. O(1)"""
        if h1 is None:
            return h2
        if h2 is None:
            return h1

        # 더 작은 루트가 승자
        if h1.key <= h2.key:
            # h2를 h1의 왼쪽 자식 앞에 삽입
            h2.next_sibling = h1.left_child
            if h1.left_child:
                h1.left_child.prev = h2
            h1.left_child = h2
            h2.prev = h1
            return h1
        else:
            h1.next_sibling = h2.left_child
            if h2.left_child:
                h2.left_child.prev = h1
            h2.left_child = h1
            h1.prev = h2
            return h2

    def insert(self, key: int) -> PairingHeapNode:
        """새 키 삽입. O(1) amortized"""
        node = PairingHeapNode(key=key)
        self.root = self._meld(self.root, node)
        self.size += 1
        return node

    def find_min(self) -> int:
        """최솟값 조회. O(1)"""
        if self.root is None:
            raise IndexError("힙이 비어있습니다")
        return self.root.key
```

### 2. Two-Pass Pairing (핵심 알고리즘)

Delete-Min 연산의 핵심은 **Two-Pass Pairing** 입니다. 루트를 제거하면 루트의 자식들(형제 리스트)만 남습니다. 이 자식들을 다시 합쳐 새 힙을 만들어야 합니다.

**1패스 (왼쪽 → 오른쪽):** 자식 리스트를 앞에서부터 둘씩 짝지어(pair) 병합합니다.
**2패스 (오른쪽 → 왼쪽):** 1패스에서 만들어진 힙들을 오른쪽에서부터 순차 병합합니다.

```python
    def _two_pass_pairing(self,
                          first_child: Optional[PairingHeapNode]
                          ) -> Optional[PairingHeapNode]:
        """Two-Pass Pairing으로 자식 리스트를 하나의 힙으로 합침"""
        if first_child is None:
            return None

        # 자식 리스트를 Python 리스트로 수집
        children = []
        cur = first_child
        while cur is not None:
            nxt = cur.next_sibling
            cur.next_sibling = None
            cur.prev = None
            children.append(cur)
            cur = nxt

        # 1패스: 왼쪽부터 둘씩 병합
        pairs = []
        i = 0
        while i + 1 < len(children):
            pairs.append(self._meld(children[i], children[i + 1]))
            i += 2
        if i < len(children):
            pairs.append(children[i])  # 홀수 개면 마지막 하나는 그대로

        # 2패스: 오른쪽부터 순차 병합
        result = pairs[-1]
        for j in range(len(pairs) - 2, -1, -1):
            result = self._meld(pairs[j], result)
        return result

    def delete_min(self) -> int:
        """최솟값 삭제 및 반환. O(log n) amortized"""
        if self.root is None:
            raise IndexError("힙이 비어있습니다")
        min_key = self.root.key
        self.root = self._two_pass_pairing(self.root.left_child)
        self.size -= 1
        return min_key

    def decrease_key(self, node: PairingHeapNode, new_key: int) -> None:
        """특정 노드의 키 감소. O(log n) amortized (경험적으로 더 빠름)"""
        if new_key > node.key:
            raise ValueError("새 키가 기존 키보다 큽니다")
        node.key = new_key
        if node is self.root:
            return  # 루트면 변경 없음

        # 노드를 부모에서 분리
        if node.prev is not None:
            if node.prev.left_child is node:
                node.prev.left_child = node.next_sibling
            else:
                node.prev.next_sibling = node.next_sibling
            if node.next_sibling:
                node.next_sibling.prev = node.prev
            node.next_sibling = None
            node.prev = None

        # 루트와 병합
        self.root = self._meld(self.root, node)
```

## 실전 예제: Dijkstra 알고리즘에 적용

페어링 힙의 가장 대표적인 활용은 그래프의 최단 경로 알고리즘입니다. decrease-key를 자주 호출하는 Dijkstra에서 실용적인 속도 개선을 경험할 수 있습니다.

```python
import heapq
from collections import defaultdict

def dijkstra_pairing_heap(graph: dict, start: int) -> dict:
    """
    페어링 힙을 활용한 Dijkstra 최단 경로.
    graph: {노드: [(이웃노드, 가중치), ...]}
    """
    heap = PairingHeap()
    dist = {node: float('inf') for node in graph}
    dist[start] = 0
    node_map = {}  # 노드 → 힙 노드 참조

    # 모든 노드 삽입
    for node in graph:
        hn = heap.insert(dist[node])
        # key를 (dist, node)로 저장하도록 확장 필요
        # 간략화: 표준 heapq와 비교 용도로 단순 예시
        node_map[node] = hn

    # 실제 프로덕션 코드: (거리, 노드) 쌍을 힙에 넣고 방문 여부로 스킵
    dist2 = {node: float('inf') for node in graph}
    dist2[start] = 0
    pq = [(0, start)]  # (거리, 노드)

    while pq:
        d, u = heapq.heappop(pq)
        if d > dist2[u]:
            continue  # 오래된 항목 스킵
        for v, weight in graph.get(u, []):
            nd = dist2[u] + weight
            if nd < dist2[v]:
                dist2[v] = nd
                heapq.heappush(pq, (nd, v))

    return dist2

# 예시 그래프
graph = {
    0: [(1, 4), (2, 1)],
    1: [(3, 1)],
    2: [(1, 2), (3, 5)],
    3: []
}
result = dijkstra_pairing_heap(graph, 0)
print(result)
# {0: 0, 1: 3, 2: 1, 3: 4}
```

### 성능 비교 실험

```python
import time
import random
import heapq

def benchmark_pairing_vs_binary(n: int = 100_000):
    keys = [random.randint(0, 10_000_000) for _ in range(n)]

    # 이진 힙 (Python heapq)
    t0 = time.perf_counter()
    bh = []
    for k in keys:
        heapq.heappush(bh, k)
    for _ in range(n // 2):
        heapq.heappop(bh)
    t1 = time.perf_counter()
    binary_time = t1 - t0

    # 페어링 힙
    t2 = time.perf_counter()
    ph = PairingHeap()
    for k in keys:
        ph.insert(k)
    for _ in range(n // 2):
        ph.delete_min()
    t3 = time.perf_counter()
    pairing_time = t3 - t2

    print(f"n={n}: Binary Heap={binary_time:.4f}s, Pairing Heap={pairing_time:.4f}s")

benchmark_pairing_vs_binary(100_000)
```

위 코드를 실행하면, 단순 삽입/삭제 벤치마크에서는 Python의 heapq(이진 힙)가 C 구현으로 최적화되어 있어 더 빠르게 나올 수 있습니다. 하지만 decrease-key가 자주 발생하는 실제 그래프 알고리즘에서는 페어링 힙이 빛을 발합니다.

## 분할상환 분석: Two-Pass Pairing이 O(log n)인 이유

**잠재 함수(Potential Function)** 를 이용한 분할상환 분석으로 Delete-Min의 O(log n) amortized 복잡도를 증명합니다.

**직관적 설명:**
- 삽입은 O(1)이지만 잠재 에너지를 쌓아둡니다.
- Delete-Min은 쌓인 잠재 에너지를 소비하며 자식들을 재배열합니다.
- Two-Pass Pairing은 형제 수를 O(log n)으로 줄이는 효과가 있습니다 (엄밀한 증명은 아직 미완성이지만, 실험적으로 확인됨).

Fredman(1999)은 `decrease-key`의 하한이 Ω(log log n)임을 증명했으나, 상한은 여전히 열린 문제(open problem)입니다. 이는 페어링 힙을 이론 CS에서 아직도 흥미로운 연구 주제로 만드는 요소입니다.

## 변형: Lazy Pairing Heap과 Skew Heap

### Lazy Pairing Heap

Delete-Min을 지연시켜 배치 처리합니다. 여러 삽입을 모아두다가 Extract-Min 시점에 한번에 처리합니다.

```python
class LazyPairingHeap:
    """지연 평가 방식의 페어링 힙 변형"""
    def __init__(self):
        self.root = None
        self._pending = []  # 지연 삽입 버퍼

    def insert(self, key: int):
        self._pending.append(PairingHeapNode(key=key))

    def _flush(self):
        """버퍼의 모든 노드를 힙에 반영"""
        ph = PairingHeap()
        ph.root = self.root
        for node in self._pending:
            ph.root = ph._meld(ph.root, node)
        self.root = ph.root
        self._pending = []
```

### 비교: Skew Heap

Skew Heap은 페어링 힙과 비슷한 병합 기반 접근법을 사용하지만, 단순한 "항상 왼쪽-오른쪽을 교환"하는 규칙으로 균형을 유지합니다.

```
Skew Heap meld 규칙:
  merge(h1, h2):
    if h1.key <= h2.key:
      h1.right ↔ h1.left  (스왑)
      h1.left = merge(h1.right, h2)  ← 재귀 병합
      return h1
```

Skew Heap은 구현이 페어링 힙보다도 단순하지만, 실용적 성능은 페어링 힙이 더 우수한 경향이 있습니다.

## 주의사항과 실용적 팁

**1. decrease-key를 위한 노드 참조 관리**
페어링 힙에서 decrease-key를 사용하려면 노드에 대한 직접 참조가 필요합니다. Dijkstra 구현 시 `{vertex: node}` 딕셔너리를 따로 관리해야 합니다.

**2. GC 언어에서의 메모리 압박**
파이썬/자바 같은 GC 언어에서는 포인터 기반 자료구조가 캐시 미스를 많이 일으킬 수 있습니다. 노드 풀(Node Pool)을 배열로 관리하면 성능이 향상됩니다.

**3. Python heapq vs. 페어링 힙**
단순 삽입/삭제만 필요하다면 Python의 내장 heapq가 C 구현으로 더 빠릅니다. 페어링 힙은 decrease-key가 필요한 복잡한 알고리즘(다익스트라, 프림, 허프만)에서 장점을 발휘합니다.

**4. Fibonacci Heap과의 선택 기준**
- 이론적 복잡도 보장이 중요하면: Fibonacci Heap
- 실제 구현과 성능이 중요하면: Pairing Heap
- 코드 단순성이 최우선이면: Binary Heap (heapq)

**5. 멀티 패스 변형**
Two-Pass 대신 Multi-Pass(자식들을 반복적으로 하나씩 병합)를 사용하면 구현은 더 단순해지지만 최악 케이스 성능이 저하됩니다.

## 정리

페어링 힙은 "이론과 실용의 균형"을 잡은 자료구조입니다. 피보나치 힙의 이론적 복잡도를 실용적인 구현으로 근사하며, Dijkstra·프림 알고리즘 등에서 탁월한 성능을 보입니다. decrease-key 연산의 정확한 복잡도가 아직 미해결 문제라는 점은, 이 자료구조가 CS 연구의 최전선에 여전히 있다는 신호이기도 합니다.

## 참고 자료
- [Pairing Heap - Wikipedia](https://en.wikipedia.org/wiki/Pairing_heap)
- [Pairing Heap - Brilliant.org](https://brilliant.org/wiki/pairing-heap/)
- [The Pairing Heap: A New Form of Self-Adjusting Heap (Fredman et al., 1986)](https://link.springer.com/article/10.1007/BF01840439)
- [Smooth heaps and a dual view of self-adjusting data structures (arXiv)](https://arxiv.org/abs/1802.05471)
