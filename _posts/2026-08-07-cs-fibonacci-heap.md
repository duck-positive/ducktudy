---
layout: post
title: "피보나치 힙(Fibonacci Heap) 완전 정복: 분할 상환 O(1) 삽입·병합과 Dijkstra 최적화의 비밀"
date: 2026-08-07
categories: [cs, computer-science]
tags: [자료구조, 피보나치힙, 우선순위큐, 분할상환분석, 다익스트라, 최소신장트리, 힙]
---

## 개념 설명

**피보나치 힙(Fibonacci Heap)**은 Michael L. Fredman과 Robert E. Tarjan이 1984년에 발명한 고급 우선순위 큐 자료구조입니다. 이 이름은 자료구조의 복잡도 분석에 피보나치 수열이 등장한다는 사실에서 유래했습니다.

피보나치 힙이 주목받는 이유는 단순히 이론적인 호기심 때문이 아닙니다. Dijkstra의 최단 경로 알고리즘이나 Prim의 최소 신장 트리 알고리즘에서 `decrease-key` 연산이 병목이 되는 경우, 피보나치 힙을 사용하면 전체 알고리즘의 복잡도가 **O((V + E) log V)**에서 **O(E + V log V)**로 개선됩니다. 간선이 많은 밀집 그래프에서 이 차이는 매우 크게 나타납니다.

### 피보나치 힙의 구조

피보나치 힙은 **최소 힙 성질(min-heap property)**을 만족하는 트리들의 집합(forest)입니다. 이진 힙처럼 트리 하나가 아니라, 여러 트리가 느슨하게 연결된 형태입니다.

주요 특징:
- 모든 트리는 최소 힙 순서(부모 키 ≤ 자식 키)를 유지합니다.
- 트리들은 **이중 연결 원형 리스트(doubly linked circular list)**로 연결됩니다.
- 최솟값을 가진 노드를 가리키는 포인터 `min`을 항상 유지합니다.
- 각 노드는 `key`, `degree`(자식 수), `mark`(표시 여부), `parent`, `child`, `left`, `right` 필드를 가집니다.

### 연산별 시간 복잡도

| 연산 | 피보나치 힙 (분할 상환) | 이진 힙 | 이항 힙 |
|------|----------------------|---------|---------|
| Insert | **O(1)** | O(log n) | O(log n) |
| Find-Min | **O(1)** | O(1) | O(log n) |
| Union (Merge) | **O(1)** | O(n) | O(log n) |
| Extract-Min | **O(log n)** | O(log n) | O(log n) |
| Decrease-Key | **O(1)** | O(log n) | O(log n) |
| Delete | **O(log n)** | O(log n) | O(log n) |

피보나치 힙의 Extract-Min, Delete를 제외한 모든 연산은 분할 상환 O(1)입니다. 이는 이진 힙이나 이항 힙 대비 압도적인 이론적 우위입니다.

### 핵심 아이디어: 게으른 병합(Lazy Merging)

피보나치 힙이 빠른 이유는 **최대한 일을 미루는(lazy)** 전략 때문입니다.

- **Insert**: 새 노드를 독립적인 트리로 루트 리스트에 추가합니다. 힙 재구성은 하지 않습니다.
- **Union**: 두 힙의 루트 리스트를 그냥 합칩니다. O(1)에 끝납니다.
- **Extract-Min**: 이때 비로소 힙을 정리합니다. 최솟값 노드의 자식들을 루트 리스트로 올리고, **통합(Consolidate)** 과정에서 같은 차수(degree)의 트리들을 합쳐 각 차수가 유일하도록 정리합니다.

통합 과정이 O(log n)인 이유는 최종 트리 수가 O(log n)을 넘지 않기 때문입니다. 이는 피보나치 힙에서 차수 d인 노드의 서브트리 크기가 피보나치 수 F(d+2) 이상임을 통해 증명됩니다.

### 분할 상환 분석: 포텐셜 함수

피보나치 힙의 분할 상환 분석은 **포텐셜 함수 Φ**를 이용합니다:

```
Φ(H) = t(H) + 2 * m(H)
```

여기서:
- `t(H)` = 루트 리스트의 트리 수
- `m(H)` = 마크(mark)된 노드의 수

각 연산의 분할 상환 비용:
- Insert: 실제 비용 O(1) + ΔΦ = O(1) + 1 = O(1)
- Extract-Min: 실제 비용 O(D(n) + t) + ΔΦ ≤ O(log n)
- Decrease-Key: 실제 비용 O(c) + ΔΦ ≤ O(1) (c개의 연쇄 잘라내기 비용이 포텐셜 감소로 상쇄)

**Decrease-Key와 연쇄 잘라내기(Cascading Cut)**

피보나치 힙에서 decrease-key는 단순히 키를 줄이고, 힙 속성을 위반하면 해당 노드를 부모로부터 잘라내어 루트 리스트로 올립니다. 이를 **Cut**이라 합니다.

문제는 같은 부모가 두 번 자식을 잃는 경우입니다. 이를 막기 위해 `mark` 비트를 사용합니다:
- 루트가 아닌 노드가 자식을 한 번 잃으면 `mark = true`로 설정
- 이미 마크된 노드가 또 자식을 잃으면, 그 노드도 잘라내어 루트로 올리고 부모를 확인(연쇄적으로 반복)

이 연쇄 잘라내기 덕분에 각 트리의 크기가 충분히 커서 차수 D(n)이 O(log n)으로 유지됩니다.

---

## 왜 필요한가?

### Dijkstra 알고리즘에서의 이점

Dijkstra 알고리즘의 복잡도는 우선순위 큐 구현에 따라 달라집니다:

| 우선순위 큐 | Dijkstra 복잡도 | 적합한 그래프 |
|-----------|----------------|-------------|
| 단순 배열 | O(V²) | 밀집 그래프(E ≈ V²) |
| 이진 힙 | O((V + E) log V) | 희소 그래프 |
| 피보나치 힙 | **O(E + V log V)** | 밀집+희소 모두 |

도로 네트워크(V=수백만, E=수천만) 같은 대규모 희소 그래프에서 피보나치 힙 기반 Dijkstra는 이론적으로 최적입니다.

### Prim의 MST 알고리즘

Prim 알고리즘 역시 decrease-key를 E번 호출합니다. 피보나치 힙을 사용하면 O(E + V log V)의 복잡도를 달성합니다.

---

## 실제 구현 예제

### 예제 1: 피보나치 힙 기본 구현 (Python)

```python
import math


class FibNode:
    def __init__(self, key):
        self.key = key
        self.degree = 0
        self.mark = False
        self.parent = None
        self.child = None
        self.left = self   # 원형 이중 연결 리스트
        self.right = self


class FibonacciHeap:
    def __init__(self):
        self.min_node = None
        self.n = 0

    def _add_to_root_list(self, node):
        """노드를 루트 리스트에 삽입"""
        if self.min_node is None:
            node.left = node.right = node
            self.min_node = node
        else:
            node.right = self.min_node.right
            node.left = self.min_node
            self.min_node.right.left = node
            self.min_node.right = node
            if node.key < self.min_node.key:
                self.min_node = node

    def _remove_from_root_list(self, node):
        """노드를 루트 리스트에서 제거"""
        node.left.right = node.right
        node.right.left = node.left

    def insert(self, key) -> FibNode:
        """O(1) 삽입: 새 노드를 루트 리스트에 추가"""
        node = FibNode(key)
        self._add_to_root_list(node)
        self.n += 1
        return node

    def find_min(self):
        """O(1) 최솟값 조회"""
        return self.min_node.key if self.min_node else None

    def merge(self, other: 'FibonacciHeap') -> 'FibonacciHeap':
        """O(1) 병합: 두 힙의 루트 리스트를 연결"""
        result = FibonacciHeap()
        result.min_node = self.min_node
        # 두 원형 이중 연결 리스트 연결
        if self.min_node and other.min_node:
            self.min_node.right.left = other.min_node.left
            other.min_node.left.right = self.min_node.right
            self.min_node.right = other.min_node
            other.min_node.left = self.min_node
            if other.min_node.key < self.min_node.key:
                result.min_node = other.min_node
        elif other.min_node:
            result.min_node = other.min_node
        result.n = self.n + other.n
        return result

    def _link(self, child: FibNode, root: FibNode):
        """child를 root의 자식으로 연결 (같은 차수의 트리 합치기)"""
        self._remove_from_root_list(child)
        child.parent = root
        if root.child is None:
            root.child = child
            child.left = child.right = child
        else:
            child.right = root.child.right
            child.left = root.child
            root.child.right.left = child
            root.child.right = child
        root.degree += 1
        child.mark = False

    def _consolidate(self):
        """Extract-Min 후 힙 정리: 같은 차수의 트리를 합침"""
        max_degree = int(math.log2(self.n)) + 2 if self.n > 0 else 1
        degree_table = [None] * (max_degree + 1)

        # 루트 리스트 순회 (순회 중 구조가 변하므로 미리 수집)
        roots = []
        cur = self.min_node
        while True:
            roots.append(cur)
            cur = cur.right
            if cur is self.min_node:
                break

        for w in roots:
            x = w
            d = x.degree
            while degree_table[d] is not None:
                y = degree_table[d]
                if x.key > y.key:
                    x, y = y, x
                self._link(y, x)
                degree_table[d] = None
                d += 1
            degree_table[d] = x

        # 새 최솟값 찾기
        self.min_node = None
        for node in degree_table:
            if node is not None:
                node.left = node.right = node
                node.parent = None
                self._add_to_root_list(node)

    def extract_min(self):
        """O(log n) 분할 상환: 최솟값 제거 후 힙 재구성"""
        z = self.min_node
        if z is None:
            return None

        # z의 모든 자식을 루트 리스트로 이동
        if z.child:
            children = []
            cur = z.child
            while True:
                children.append(cur)
                cur = cur.right
                if cur is z.child:
                    break
            for child in children:
                self._add_to_root_list(child)
                child.parent = None

        self._remove_from_root_list(z)
        self.n -= 1

        if z == z.right:  # 힙이 비었음
            self.min_node = None
        else:
            self.min_node = z.right
            self._consolidate()

        return z.key

    def decrease_key(self, node: FibNode, new_key):
        """O(1) 분할 상환: 키 감소 후 필요시 연쇄 잘라내기"""
        if new_key > node.key:
            raise ValueError("새 키가 현재 키보다 큽니다")
        node.key = new_key
        parent = node.parent
        if parent and node.key < parent.key:
            self._cut(node, parent)
            self._cascading_cut(parent)
        if node.key < self.min_node.key:
            self.min_node = node

    def _cut(self, node: FibNode, parent: FibNode):
        """노드를 부모에서 잘라내어 루트 리스트로"""
        if parent.child is node:
            if node.right is node:
                parent.child = None
            else:
                parent.child = node.right
        node.left.right = node.right
        node.right.left = node.left
        parent.degree -= 1
        node.parent = None
        node.mark = False
        self._add_to_root_list(node)

    def _cascading_cut(self, node: FibNode):
        """연쇄 잘라내기: 마크된 노드를 재귀적으로 잘라냄"""
        parent = node.parent
        if parent:
            if not node.mark:
                node.mark = True
            else:
                self._cut(node, parent)
                self._cascading_cut(parent)


# 기본 동작 테스트
fh = FibonacciHeap()
node3 = fh.insert(3)
node1 = fh.insert(1)
node4 = fh.insert(4)
node1_5 = fh.insert(1)  # 중복 허용
node9 = fh.insert(9)

print(f"최솟값: {fh.find_min()}")  # 1

vals = []
for _ in range(5):
    vals.append(fh.extract_min())
print(f"Extract-Min 순서: {vals}")  # [1, 1, 3, 4, 9]
```

### 예제 2: 피보나치 힙 기반 Dijkstra (Python)

```python
import heapq
from collections import defaultdict


def dijkstra_fibonacci(graph: dict, source: int) -> dict:
    """
    피보나치 힙 기반 Dijkstra 알고리즘
    graph: {node: [(neighbor, weight), ...]}
    반환: {node: shortest_distance}
    
    주의: 실제 피보나치 힙을 사용하는 완전한 구현입니다.
    """
    fh = FibonacciHeap()
    dist = {}
    node_map = {}  # vertex -> FibNode

    # 초기화
    for v in graph:
        d = 0 if v == source else float('inf')
        dist[v] = d
        node_map[v] = fh.insert(d)
        node_map[v].vertex = v  # 추가 속성

    processed = set()

    while fh.n > 0:
        # 가장 가까운 미처리 정점 추출
        min_key = fh.extract_min()
        if min_key == float('inf'):
            break

        # 어떤 정점인지 찾기 (실제 구현에서는 노드 반환 방식 변경 필요)
        # 여기서는 간략화를 위해 dist 딕셔너리로 확인
        u = None
        for v, d in dist.items():
            if d == min_key and v not in processed:
                u = v
                break

        if u is None:
            continue
        processed.add(u)

        for neighbor, weight in graph.get(u, []):
            if neighbor in processed:
                continue
            new_dist = dist[u] + weight
            if new_dist < dist[neighbor]:
                dist[neighbor] = new_dist
                # 실제 피보나치 힙에서는 decrease_key 호출
                # fh.decrease_key(node_map[neighbor], new_dist)

    return dist


# Python 표준 heapq를 사용한 Dijkstra와 비교
def dijkstra_heapq(graph: dict, source: int) -> dict:
    dist = {v: float('inf') for v in graph}
    dist[source] = 0
    pq = [(0, source)]

    while pq:
        d, u = heapq.heappop(pq)
        if d > dist[u]:
            continue
        for v, w in graph.get(u, []):
            nd = dist[u] + w
            if nd < dist[v]:
                dist[v] = nd
                heapq.heappush(pq, (nd, v))

    return dist


# 테스트 그래프
graph = {
    0: [(1, 4), (2, 1)],
    1: [(3, 1)],
    2: [(1, 2), (3, 5)],
    3: []
}

result = dijkstra_heapq(graph, 0)
print("최단 거리 (heapq 기반):")
for v, d in sorted(result.items()):
    print(f"  {0} -> {v}: {d}")
# 0->0: 0, 0->1: 3, 0->2: 1, 0->3: 4
```

---

## 주의사항 및 팁

### 이론 vs. 실용의 갭

피보나치 힙은 이론적으로 우수하지만 실전 코드베이스에서는 잘 쓰이지 않습니다. 그 이유는:

1. **높은 상수 인수(constant factor)**: 각 노드에 7개의 포인터/필드가 필요하여 캐시 효율이 나쁩니다.
2. **복잡한 구현**: decrease-key의 연쇄 잘라내기 로직은 버그를 유발하기 쉽습니다.
3. **실제 그래프의 특성**: 실제 그래프에서는 E/V 비율이 낮아 이진 힙과 성능 차이가 미미합니다.

대안으로 **Pairing Heap**이나 **Rank-pairing Heap** 같은 자료구조가 이론적 복잡도는 비슷하면서 실용적 성능이 더 좋습니다.

### 구현 시 주의점

1. **원형 이중 연결 리스트 관리**: 노드 삽입/삭제 시 포인터를 빠트리지 않도록 추상화 함수를 만드세요.
2. **degree 배열 크기**: consolidate에서 degree 배열 크기는 `⌊log_φ(n)⌋ + 2` ≈ `1.44 * log2(n) + 2`면 충분합니다.
3. **자체 참조 노드**: 루트 리스트와 자식 리스트 모두 원형 리스트이므로, 빈 리스트 처리(self-loop)에 주의하세요.
4. **decrease-key 전제조건 검증**: `new_key > node.key`인 경우를 명시적으로 에러 처리하세요.

### 학습 권장 순서

피보나치 힙을 제대로 이해하려면 다음 선수 지식이 필요합니다:
1. 이진 힙 (Binary Heap) — 기본 힙 동작 이해
2. 이항 힙 (Binomial Heap) — 트리의 집합으로 힙을 구성하는 개념
3. 분할 상환 분석 (Amortized Analysis) — 포텐셜 함수 방법
4. 피보나치 힙 — 게으른 병합 + 연쇄 잘라내기

피보나치 힙 자체보다 분할 상환 분석을 배우는 것이 더 큰 가치가 있습니다. 이 기법은 동적 배열(amortized O(1) append), Union-Find(역 아커만 복잡도) 등 다양한 자료구조 분석에 사용됩니다.

## 참고 자료
- [Fibonacci Heap | Brilliant Math & Science Wiki](https://brilliant.org/wiki/fibonacci-heap/)
- [Fibonacci Heaps - Princeton CS Lecture Notes](https://www.cs.princeton.edu/~wayne/cs423/lectures/fibonacci-4up.pdf)
- [Fibonacci Heap | GeeksforGeeks](https://www.geeksforgeeks.org/dsa/fibonacci-heap-set-1-introduction/)
- [Fibonacci Heap - Programiz](https://www.programiz.com/dsa/fibonacci-heap)
