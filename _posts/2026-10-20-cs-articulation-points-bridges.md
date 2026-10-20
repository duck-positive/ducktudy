---
layout: post
title: "단절점과 단절선 완전 정복: Tarjan의 DFS로 그래프의 취약점을 O(V+E)에 찾아내기"
date: 2026-10-20
categories: [cs, computer-science]
tags: [articulation-point, bridge, graph, tarjan, dfs, connectivity, algorithm, network]
---

## 단절점과 단절선이란?

무방향 연결 그래프(undirected connected graph)에서 특정 구성 요소를 제거했을 때 그래프가 분리(disconnect)되는 경우가 있다.

- **단절점(Articulation Point / Cut Vertex)**: 제거하면 그래프가 두 개 이상의 연결 요소(connected component)로 분리되는 **정점(vertex)**
- **단절선(Bridge / Cut Edge)**: 제거하면 그래프가 분리되는 **간선(edge)**

직관적인 예를 들면, 두 네트워크 허브를 잇는 단 하나의 광케이블이 단절선이다. 그 케이블 하나가 끊어지면 두 네트워크가 완전히 단절된다.

### 왜 중요한가?

- **네트워크 신뢰성 분석**: 단절점·단절선을 제거하면 네트워크가 분할된다. 이를 미리 식별해 이중화(redundancy)를 추가한다.
- **회로 설계**: 인쇄 회로 기판(PCB)에서 단절선을 찾아 단일 장애점을 제거한다.
- **소셜 네트워크 분석**: 영향력의 중개자(broker) 역할을 하는 단절점 사용자를 찾는다.
- **컴파일러 최적화**: 제어 흐름 그래프(CFG)에서 지배 노드(dominator)를 분석할 때 관련 개념이 쓰인다.

---

## 브루트 포스 vs Tarjan 알고리즘

### 브루트 포스: O(V × (V + E))

각 정점(또는 간선)을 순서대로 제거한 뒤 BFS/DFS로 연결성을 확인한다. 정점 V개에 대해 각각 O(V + E)가 걸리므로 총 O(V(V + E))다.

### Tarjan의 알고리즘: O(V + E)

Robert Tarjan이 1972년에 제안한 알고리즘은 **단 한 번의 DFS**로 모든 단절점과 단절선을 찾는다. 핵심 아이디어는 두 가지 값을 각 노드에 부여하는 것이다.

| 값 | 의미 |
|----|------|
| `disc[u]` | DFS 탐색 시 노드 u가 처음 방문된 시간(순서) |
| `low[u]` | u의 서브트리에서 back edge를 통해 도달 가능한 가장 작은 disc 값 |

**back edge**란 DFS 트리에서 조상(ancestor)으로 향하는 간선이다. `low[u]`는 u의 서브트리가 트리 간선 없이 DFS로 거슬러 올라갈 수 있는 '최고 조상'의 발견 시간을 나타낸다.

### 단절점 판별 조건

노드 `u`의 자식 `v`에 대해:

1. **루트 노드인 경우**: DFS 트리에서 자식이 2개 이상이면 단절점
2. **루트가 아닌 경우**: `low[v] >= disc[u]`이면 단절점

조건 2의 의미: v의 서브트리가 u를 거치지 않고는 u의 조상에 도달할 수 없으므로, u를 제거하면 v의 서브트리가 분리된다.

### 단절선(브리지) 판별 조건

간선 `(u, v)` (u가 부모, v가 자식)에 대해:

`low[v] > disc[u]`이면 브리지

v의 서브트리에서 u 이상으로 거슬러 올라가는 back edge가 없다는 뜻으로, 간선 (u, v)를 제거하면 v가 분리된다.

---

## 실제 구현 예제

### 예제 1: 단절점 찾기 (Python)

```python
from typing import List, Set

class ArticulationPointFinder:
    """
    Tarjan의 DFS 기반 단절점 탐색.
    시간 복잡도: O(V + E)
    """

    def __init__(self, n: int):
        self.n = n
        self.graph = [[] for _ in range(n)]

    def add_edge(self, u: int, v: int):
        self.graph[u].append(v)
        self.graph[v].append(u)

    def find(self) -> Set[int]:
        disc = [-1] * self.n
        low = [0] * self.n
        parent = [-1] * self.n
        is_ap = [False] * self.n
        timer = [0]

        def dfs(u: int):
            disc[u] = low[u] = timer[0]
            timer[0] += 1
            children = 0

            for v in self.graph[u]:
                if disc[v] == -1:
                    children += 1
                    parent[v] = u
                    dfs(v)
                    low[u] = min(low[u], low[v])

                    if parent[u] == -1 and children > 1:
                        is_ap[u] = True
                    if parent[u] != -1 and low[v] >= disc[u]:
                        is_ap[u] = True

                elif v != parent[u]:
                    low[u] = min(low[u], disc[v])

        for i in range(self.n):
            if disc[i] == -1:
                dfs(i)

        return {i for i in range(self.n) if is_ap[i]}


# 예제 그래프:
#   0 - 1 - 2
#   |       |
#   +---3---+
#       |
#       4
finder = ArticulationPointFinder(5)
for u, v in [(0,1),(1,2),(2,3),(3,0),(3,4)]:
    finder.add_edge(u, v)

aps = finder.find()
print("단절점:", sorted(aps))  # 출력: [3]

# 다른 예제:
#   1 - 0 - 2
#       |
#       3
finder2 = ArticulationPointFinder(4)
for u, v in [(0,1),(0,2),(0,3)]:
    finder2.add_edge(u, v)
print("단절점:", sorted(finder2.find()))  # 출력: [0]
```

---

### 예제 2: 단절선(브리지) 찾기 (Python)

```python
from typing import List, Tuple

class BridgeFinder:
    """
    Tarjan의 DFS 기반 브리지(단절선) 탐색.
    중복 간선(멀티 그래프) 처리를 위해 간선 ID를 사용한다.
    시간 복잡도: O(V + E)
    """

    def __init__(self, n: int):
        self.n = n
        self.graph = [[] for _ in range(n)]
        self.edge_cnt = 0

    def add_edge(self, u: int, v: int) -> int:
        eid = self.edge_cnt
        self.graph[u].append((v, eid))
        self.graph[v].append((u, eid))
        self.edge_cnt += 1
        return eid

    def find(self) -> List[Tuple[int, int]]:
        disc = [-1] * self.n
        low = [0] * self.n
        bridges = []
        timer = [0]

        def dfs(u: int, parent_eid: int):
            disc[u] = low[u] = timer[0]
            timer[0] += 1

            for v, eid in self.graph[u]:
                if eid == parent_eid:
                    continue
                if disc[v] == -1:
                    dfs(v, eid)
                    low[u] = min(low[u], low[v])
                    if low[v] > disc[u]:
                        bridges.append((u, v))
                else:
                    low[u] = min(low[u], disc[v])

        for i in range(self.n):
            if disc[i] == -1:
                dfs(i, -1)

        return bridges


# 예제 그래프:
#   0 - 1 - 2 - 3
#       |       |
#       +---4---+
bf = BridgeFinder(5)
bf.add_edge(0, 1)
bf.add_edge(1, 2)
bf.add_edge(2, 3)
bf.add_edge(3, 4)
bf.add_edge(4, 1)

print("단절선:", bf.find())
# 출력: [(0, 1)]
# 1-2-3-4 는 사이클로 연결되어 있어 브리지 없음
# 0-1 간선만 브리지

# 일직선 그래프 (모두 브리지):
bf2 = BridgeFinder(4)
for u, v in [(0,1),(1,2),(2,3)]:
    bf2.add_edge(u, v)
print("단절선:", bf2.find())  # [(0,1), (1,2), (2,3)]
```

---

### 보충: 반복적(Iterative) DFS로 재귀 스택 오버플로 방지

노드 수가 수십만이면 재귀 DFS가 Python 기본 재귀 한계(`sys.setrecursionlimit`)를 초과한다. 스택(Stack) 자료구조로 DFS를 직접 구현하면 이를 피할 수 있다.

```python
import sys
sys.setrecursionlimit(300000)  # 임시방편

# 명시적 스택 기반 반복 DFS
def dfs_iterative(graph, root, n):
    disc = [-1] * n
    low = [0] * n
    parent = [-1] * n
    is_ap = [False] * n
    stack = [(root, iter(graph[root]))]
    timer = 0

    while stack:
        u, neighbors = stack[-1]
        if disc[u] == -1:
            disc[u] = low[u] = timer
            timer += 1

        try:
            v = next(neighbors)
            if disc[v] == -1:
                parent[v] = u
                stack.append((v, iter(graph[v])))
            elif v != parent[u]:
                low[u] = min(low[u], disc[v])
        except StopIteration:
            stack.pop()
            if stack:
                p = stack[-1][0]
                low[p] = min(low[p], low[u])
                children = sum(1 for nb in graph[p] if parent[nb] == p)
                if parent[p] == -1 and children > 1:
                    is_ap[p] = True
                if parent[p] != -1 and low[u] >= disc[p]:
                    is_ap[p] = True
```

---

## 주의사항 및 팁

### 1. 간선 ID로 부모 간선 구분 (멀티 그래프)

단순히 `v != parent[u]`로 부모 간선을 제외하면, 두 노드 사이에 평행 간선(parallel edge)이 있는 멀티 그래프에서 오류가 발생한다. **간선 ID**를 사용해 돌아온 간선을 정확히 필터링하는 것이 안전하다.

### 2. 비연결 그래프 처리

그래프가 연결되지 않은 경우, 모든 노드를 순회하며 미방문 노드에서 DFS를 시작한다. 위 구현에서는 `for i in range(n): if disc[i] == -1: dfs(i)` 패턴으로 처리한다.

### 3. 무방향 그래프만 해당

단절점·단절선 개념은 **무방향 그래프**에서 정의된다. 방향 그래프에서는 SCC(강결합 요소)와 응축 그래프(condensation graph)로 유사한 분석을 수행한다.

### 4. 단절점이 아닌 단절선이 존재할 수 있음

간선 (u, v)가 브리지라도 u, v 각각이 단절점이 아닐 수 있다. 예를 들어 단순 경로 `0-1-2`에서 간선 (0,1), (1,2) 모두 브리지지만, 단절점은 1뿐이다(0과 2는 제거해도 나머지가 연결됨).

### 5. Biconnected Component (이중 연결 요소)

단절점을 기준으로 그래프를 분해하면 **이중 연결 요소(BCC)**를 얻는다. BCC 내부에는 단절점이 없으며, BCC들을 노드로 하고 단절점을 간선으로 연결하면 트리(블록-컷 트리)가 만들어진다. 이를 이용해 경로 쿼리를 효율적으로 처리한다.

---

## 복잡도 정리

| 알고리즘 | 시간 복잡도 | 공간 복잡도 |
|---------|-----------|----------|
| 브루트 포스 | O(V × (V + E)) | O(V + E) |
| **Tarjan (단절점/브리지)** | **O(V + E)** | **O(V + E)** |

---

## 참고 자료

- [Bridges and Articulation Points — Swiss Olympiad in Informatics](https://soi.ch/wiki/bridges-articulationpoints/)
- [The DFS Tree and Its Applications — Codeforces Blog](https://codeforces.com/blog/entry/68138)
- [Articulation Points Tutorial — Codeforces](https://codeforces.net/blog/entry/71146)
- [Articulation Points and Bridges — TutorialsPoint](https://www.tutorialspoint.com/data_structures_algorithms/articulation_points.htm)
