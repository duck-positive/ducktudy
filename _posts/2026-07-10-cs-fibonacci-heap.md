---
layout: post
title: "피보나치 힙(Fibonacci Heap) 완전 정복: Dijkstra를 O(E + V log V)로 만드는 자료구조"
date: 2026-07-10
categories: [cs, computer-science]
tags: [fibonacci-heap, data-structure, priority-queue, dijkstra, amortized-analysis, algorithm]
---

## 피보나치 힙이란?

**피보나치 힙(Fibonacci Heap)**은 1984년 Michael L. Fredman과 Robert E. Tarjan이 발표한 자료구조로, 힙 연산에서 최고 수준의 분할 상환 시간 복잡도를 달성합니다. 이름의 "피보나치"는 이 구조의 핵심 분석에 피보나치 수열이 등장하기 때문에 붙여졌습니다.

피보나치 힙은 **최소 힙(min-heap) 성질을 만족하는 트리의 집합**으로 구성됩니다. 일반적인 이진 힙이 완전 이진 트리라는 엄격한 구조를 유지하는 것과 달리, 피보나치 힙은 트리의 형태에 거의 제약이 없어 삽입이나 합병(merge) 연산이 매우 느슨하게 이루어집니다. 이 "게으른(lazy)" 전략 덕분에 대부분의 연산이 O(1) 분할 상환 시간 안에 완료됩니다.

### 연산별 시간 복잡도 비교

| 연산 | 이진 힙 | 이항 힙 | 피보나치 힙 |
|------|---------|---------|------------|
| insert | O(log n) | O(log n) | **O(1)** 분할상환 |
| find-min | O(1) | O(log n) | **O(1)** |
| extract-min | O(log n) | O(log n) | **O(log n)** 분할상환 |
| decrease-key | O(log n) | O(log n) | **O(1)** 분할상환 |
| merge | O(n) | O(log n) | **O(1)** |
| delete | O(log n) | O(log n) | **O(log n)** 분할상환 |

이 중 `decrease-key`가 O(1)이라는 점이 피보나치 힙의 핵심입니다.

---

## 왜 피보나치 힙이 필요한가?

### Dijkstra 알고리즘 최적화

다익스트라 알고리즘의 시간 복잡도는 사용하는 우선순위 큐의 성능에 직결됩니다.

- 이진 힙 사용 시: **O((V + E) log V)**
- 피보나치 힙 사용 시: **O(E + V log V)**

두 복잡도의 차이는 `decrease-key` 연산 비용에 있습니다. 간선이 많은 밀집 그래프(E ≈ V²)에서는 이 차이가 매우 커집니다. 이진 힙 기반은 O(V² log V)가 되지만, 피보나치 힙 기반은 O(V²)로 줄어듭니다.

### Prim의 최소 신장 트리

마찬가지로 Prim 알고리즘도 피보나치 힙을 쓰면 O(E + V log V)로 개선됩니다. 네트워크 라우팅이나 VLSI 설계 같은 밀집 그래프 분야에서 실질적인 성능 차이를 가져옵니다.

---

## 내부 구조 이해

### 트리 컬렉션과 루트 리스트

피보나치 힙은 여러 개의 최소 힙 순서 트리(min-heap ordered tree)를 쌍방향 순환 연결 리스트로 연결한 구조입니다. 각 노드는 다음을 가집니다:

- `key`: 노드의 값
- `degree`: 자식 노드의 수
- `marked`: 자식을 잃은 적 있는지 여부 (decrease-key 최적화에 사용)
- `parent`, `child`, `left`, `right`: 포인터

최솟값 포인터(`min`)는 항상 루트 리스트에서 가장 작은 값을 가진 노드를 가리킵니다.

### 게으른 삽입 (Lazy Insert)

삽입은 단순히 새 트리(단일 노드)를 루트 리스트에 추가하고, 필요시 `min` 포인터를 갱신합니다. 루트 리스트를 정리하는 작업은 `extract-min` 때로 미룹니다.

### extract-min과 consolidate

`extract-min`은 가장 비용이 큰 연산으로, 이때 **consolidate(통합)** 과정이 발생합니다. 같은 차수(degree)를 가진 트리들을 하나로 합쳐 각 차수가 고유한 상태로 만듭니다. 이는 이항 힙의 정규화와 유사하며, 추후 연산의 효율을 보장합니다.

---

## 실제 구현 예제

### 예제 1: 피보나치 힙 핵심 연산 (Python)

```python
import math

class FibNode:
    def __init__(self, key):
        self.key = key
        self.degree = 0
        self.marked = False
        self.parent = None
        self.child = None
        self.left = self
        self.right = self

class FibonacciHeap:
    def __init__(self):
        self.min_node = None
        self.total = 0

    def _add_to_root_list(self, node):
        if self.min_node is None:
            self.min_node = node
            node.left = node
            node.right = node
        else:
            node.right = self.min_node
            node.left = self.min_node.left
            self.min_node.left.right = node
            self.min_node.left = node
            if node.key < self.min_node.key:
                self.min_node = node

    def insert(self, key):
        node = FibNode(key)
        self._add_to_root_list(node)
        self.total += 1
        return node

    def find_min(self):
        return self.min_node.key if self.min_node else None

    def merge(self, other):
        """두 피보나치 힙을 O(1)에 합병"""
        if other.min_node is None:
            return
        if self.min_node is None:
            self.min_node = other.min_node
        else:
            # 루트 리스트 연결
            self_last = self.min_node.left
            other_last = other.min_node.left
            self_last.right = other.min_node
            other.min_node.left = self_last
            other_last.right = self.min_node
            self.min_node.left = other_last
            if other.min_node.key < self.min_node.key:
                self.min_node = other.min_node
        self.total += other.total

    def _link(self, child, parent):
        """child를 parent의 자식으로 연결"""
        # 루트 리스트에서 child 제거
        child.left.right = child.right
        child.right.left = child.left
        child.parent = parent
        if parent.child is None:
            parent.child = child
            child.left = child
            child.right = child
        else:
            child.right = parent.child
            child.left = parent.child.left
            parent.child.left.right = child
            parent.child.left = child
        parent.degree += 1
        child.marked = False

    def _consolidate(self):
        max_degree = int(math.log2(self.total)) + 2
        degree_table = [None] * max_degree

        # 루트 리스트 순회
        nodes = []
        cur = self.min_node
        while True:
            nodes.append(cur)
            cur = cur.right
            if cur == self.min_node:
                break

        for node in nodes:
            d = node.degree
            while degree_table[d] is not None:
                other = degree_table[d]
                if node.key > other.key:
                    node, other = other, node
                self._link(other, node)
                degree_table[d] = None
                d += 1
            degree_table[d] = node

        self.min_node = None
        for node in degree_table:
            if node is not None:
                node.left = node
                node.right = node
                self._add_to_root_list(node)

    def extract_min(self):
        z = self.min_node
        if z is None:
            return None
        # z의 자식들을 루트 리스트로 이동
        if z.child:
            children = []
            cur = z.child
            while True:
                children.append(cur)
                cur = cur.right
                if cur == z.child:
                    break
            for child in children:
                self._add_to_root_list(child)
                child.parent = None
        # z를 루트 리스트에서 제거
        z.left.right = z.right
        z.right.left = z.left
        if z == z.right:
            self.min_node = None
        else:
            self.min_node = z.right
            self._consolidate()
        self.total -= 1
        return z.key

    def _cut(self, node, parent):
        """node를 parent에서 잘라 루트 리스트로 이동"""
        if node.right == node:
            parent.child = None
        else:
            node.left.right = node.right
            node.right.left = node.left
            if parent.child == node:
                parent.child = node.right
        parent.degree -= 1
        node.marked = False
        node.parent = None
        self._add_to_root_list(node)

    def _cascading_cut(self, node):
        """연쇄 절단: marked 노드를 위로 올려 트리 균형 유지"""
        parent = node.parent
        if parent is not None:
            if not node.marked:
                node.marked = True
            else:
                self._cut(node, parent)
                self._cascading_cut(parent)

    def decrease_key(self, node, new_key):
        """O(1) 분할상환 시간에 키 감소"""
        if new_key > node.key:
            raise ValueError("new_key must be smaller")
        node.key = new_key
        parent = node.parent
        if parent and node.key < parent.key:
            self._cut(node, parent)
            self._cascading_cut(parent)
        if node.key < self.min_node.key:
            self.min_node = node


# 사용 예시
fh = FibonacciHeap()
n1 = fh.insert(10)
n2 = fh.insert(3)
n3 = fh.insert(7)
n4 = fh.insert(1)

print(f"최솟값: {fh.find_min()}")          # 1
print(f"extract-min: {fh.extract_min()}")  # 1
print(f"extract-min: {fh.extract_min()}")  # 3

fh.decrease_key(n1, 2)
print(f"최솟값 (decrease-key 후): {fh.find_min()}")  # 2
```

### 예제 2: 피보나치 힙 기반 Dijkstra (Python)

```python
from collections import defaultdict

def dijkstra_fibonacci(graph, source, num_vertices):
    """
    피보나치 힙 기반 Dijkstra 알고리즘
    시간복잡도: O(E + V log V)
    """
    fh = FibonacciHeap()
    dist = {v: float('inf') for v in range(num_vertices)}
    node_map = {}  # vertex -> FibNode

    dist[source] = 0
    node_map[source] = fh.insert(0)

    for v in range(num_vertices):
        if v != source:
            node_map[v] = fh.insert(float('inf'))

    visited = set()

    while fh.total > 0:
        # O(log V) 분할상환
        min_key = fh.extract_min()
        if min_key == float('inf'):
            break

        # 현재 최솟값에 해당하는 정점 찾기
        u = next((v for v, d in dist.items() if d == min_key and v not in visited), None)
        if u is None:
            continue
        visited.add(u)

        for v, weight in graph[u]:
            if v not in visited:
                new_dist = dist[u] + weight
                if new_dist < dist[v]:
                    old_dist = dist[v]
                    dist[v] = new_dist
                    # O(1) 분할상환 — 이것이 피보나치 힙의 핵심!
                    fh.decrease_key(node_map[v], new_dist)

    return dist


# 그래프 예시
graph = defaultdict(list)
edges = [
    (0, 1, 4), (0, 2, 1),
    (2, 1, 2), (1, 3, 1),
    (2, 3, 5)
]
for u, v, w in edges:
    graph[u].append((v, w))
    graph[v].append((u, w))

distances = dijkstra_fibonacci(graph, source=0, num_vertices=4)
print("최단 거리:")
for v, d in distances.items():
    print(f"  0 → {v}: {d}")
# 0 → 0: 0
# 0 → 1: 3  (0→2→1)
# 0 → 2: 1
# 0 → 3: 4  (0→2→1→3)
```

---

## 분할 상환 분석(Amortized Analysis) 핵심

피보나치 힙의 성능 분석에는 **포텐셜 함수(potential function)** Φ를 사용합니다.

```
Φ = (루트 리스트의 트리 수) + 2 × (marked 노드의 수)
```

- **insert**: 루트 리스트에 트리 1개 추가 → ΔΦ = +1, 실제 비용 O(1), 분할상환 비용 O(1)
- **extract-min**: consolidate 후 트리 수 감소 → 분할상환 비용 O(log n)
- **decrease-key**: cascading cut으로 marked 노드 정리 → 분할상환 비용 O(1)

`decrease-key`의 cascading cut이 연쇄적으로 발생하더라도, 이는 실제 실행 비용이 marked 노드 수 감소(ΔΦ < 0)로 상쇄되기 때문에 분할상환 비용은 O(1)에 수렴합니다.

---

## 주의사항과 실전 팁

### 피보나치 힙을 쓰면 안 될 때

**상수 계수가 크다**: 포인터 조작이 복잡해 실제 실행 시간의 상수 계수가 이진 힙보다 훨씬 큽니다. 그래프가 밀집(dense)하지 않은 경우 이진 힙이 더 빠를 수 있습니다.

**캐시 비효율**: 피보나치 힙은 포인터 기반으로 메모리가 분산되어 캐시 지역성이 낮습니다. 이진 힙은 배열 기반이라 캐시에 친화적입니다.

**구현 복잡도**: decrease-key와 cascading cut 구현이 까다로워 버그가 생기기 쉽습니다.

### 실전에서의 대안

실제 프로덕션 시스템에서는 다음을 고려하세요:

1. **sparse graph (E ≈ V)**: 이진 힙이 충분
2. **dense graph (E ≈ V²)**: 피보나치 힙의 이론적 우위가 실현됨
3. **Java**: `PriorityQueue` (이진 힙) 대신 피보나치 힙 구현 라이브러리 활용
4. **Relaxed heap, Brodal queue**: 피보나치 힙의 복잡성을 줄인 대안 자료구조

### 언제 사용하면 좋은가?

- 네트워크 라우팅 알고리즘 (OSPF 내부 구현)
- VLSI 회로 배선 최적화
- 대형 그래프의 최단 경로 계산
- decrease-key가 매우 빈번하게 호출되는 모든 문제

---

## 참고 자료

- [Fibonacci Heap - Wikipedia](https://en.wikipedia.org/wiki/Fibonacci_heap)
- [Fibonacci Heap Introduction - GeeksforGeeks](https://www.geeksforgeeks.org/dsa/fibonacci-heap-set-1-introduction/)
- [Fibonacci Heaps - Princeton CS (Fredman & Tarjan)](https://www.cs.princeton.edu/~wayne/teaching/fibonacci-heap.pdf)
- [Fibonacci Heap - Programiz](https://www.programiz.com/dsa/fibonacci-heap)
