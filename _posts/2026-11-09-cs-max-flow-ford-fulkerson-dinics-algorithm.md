---
layout: post
title: "최대 유량(Max-Flow) 알고리즘 완전 정복: Ford-Fulkerson부터 Dinic's까지, 이분 매칭과 프로젝트 선택까지"
date: 2026-11-09
categories: [cs, computer-science]
tags: [max-flow, ford-fulkerson, edmonds-karp, dinics-algorithm, bipartite-matching, graph, algorithm]
---

## 최대 유량 문제란?

**최대 유량(Maximum Flow)** 문제는 그래프 이론에서 가장 널리 응용되는 문제 중 하나다. 파이프 네트워크를 통해 소스(source)에서 싱크(sink)까지 흘릴 수 있는 최대 유량을 구하는 문제이지만, 실제 응용은 훨씬 광범위하다.

**유량 네트워크(Flow Network)** 는 방향 그래프 G = (V, E)로, 각 간선 (u, v)에 용량(capacity) `c(u, v) ≥ 0`이 주어진다. 소스 `s`에서 싱크 `t`로 흐르는 유량 `f`는 세 가지 조건을 만족해야 한다:

1. **용량 제약**: `0 ≤ f(u, v) ≤ c(u, v)` — 용량을 초과할 수 없다.
2. **유량 보존**: `s, t`가 아닌 모든 정점에서 들어오는 유량 = 나가는 유량.
3. **대칭성**: `f(u, v) = -f(v, u)` — 역방향 유량이 존재한다.

**최대-유량 최소-컷 정리(Max-Flow Min-Cut Theorem)**: 네트워크의 최대 유량은 최소 컷의 용량과 같다. 이 정리는 최대 유량 문제와 최소 컷 문제가 동일한 문제의 양면임을 보여주며, 많은 최적화 문제를 유량 문제로 환원하는 토대가 된다.

---

## Ford-Fulkerson 방법

**Ford-Fulkerson 방법**(1956)은 최대 유량을 구하는 가장 기본적인 접근법이다. 핵심 아이디어는 **증가 경로(augmenting path)** 를 반복적으로 찾아 유량을 증가시키는 것이다.

**잔여 그래프(Residual Graph)**: 현재 유량이 흐르는 네트워크에서, 각 간선 (u, v)에 대해:
- **순방향 잔여 용량**: `c_f(u, v) = c(u, v) - f(u, v)` (더 보낼 수 있는 양)
- **역방향 잔여 용량**: `c_f(v, u) = f(u, v)` (취소할 수 있는 양)

```python
from collections import defaultdict, deque
from typing import Dict, List, Tuple

class MaxFlowFordFulkerson:
    """
    Ford-Fulkerson 방법: DFS로 증가 경로를 탐색.
    시간 복잡도: O(E * max_flow) — 정수 용량일 때만 종료 보장.
    """
    
    def __init__(self, num_vertices: int):
        self.n = num_vertices
        # 잔여 그래프: graph[u][v] = 잔여 용량
        self.graph = [[0] * num_vertices for _ in range(num_vertices)]
    
    def add_edge(self, u: int, v: int, capacity: int):
        self.graph[u][v] += capacity
        # 역방향 간선은 초기 용량 0 (잔여 그래프에서 취소 경로용)
    
    def _dfs(self, source: int, sink: int, visited: set, min_cap: float) -> float:
        """DFS로 source → sink 증가 경로 탐색, 최소 용량 반환."""
        if source == sink:
            return min_cap
        visited.add(source)
        for v in range(self.n):
            if v not in visited and self.graph[source][v] > 0:
                pushed = self._dfs(v, sink, visited, min(min_cap, self.graph[source][v]))
                if pushed > 0:
                    # 역방향에 유량 추가 (취소 가능성)
                    self.graph[source][v] -= pushed
                    self.graph[v][source] += pushed
                    return pushed
        return 0
    
    def max_flow(self, source: int, sink: int) -> int:
        total_flow = 0
        while True:
            pushed = self._dfs(source, sink, set(), float('inf'))
            if pushed == 0:
                break
            total_flow += pushed
        return total_flow


# 사용 예: 간단한 유량 네트워크
#
#       10        10
#   0 -----> 1 -----> 3
#   |        |        ^
# 10|      1 |        | 10
#   v        v        |
#   2 -----> 4 -----> 5
#       10        10
#
# source=0, sink=5, 최대 유량=20

ff = MaxFlowFordFulkerson(6)
ff.add_edge(0, 1, 10)
ff.add_edge(0, 2, 10)
ff.add_edge(1, 3, 10)
ff.add_edge(1, 4, 1)
ff.add_edge(2, 4, 10)
ff.add_edge(3, 5, 10)
ff.add_edge(4, 5, 10)

result = ff.max_flow(0, 5)
print(f"최대 유량 (Ford-Fulkerson): {result}")  # 20
```

**Ford-Fulkerson의 문제점**: DFS는 경로를 비효율적으로 선택할 수 있다. 최악의 경우 용량이 `C`인 그래프에서 `O(C)` 번의 증가가 필요하다. 용량이 무리수이면 무한 루프가 발생할 수도 있다.

---

## Edmonds-Karp 알고리즘: BFS로 최단 경로 선택

**Edmonds-Karp 알고리즘**(1972)은 Ford-Fulkerson에서 DFS 대신 **BFS**를 사용하여 항상 가장 짧은 증가 경로(최소 간선 수)를 선택한다. 이로써 시간 복잡도가 `O(VE²)`로 보장된다 — 용량과 무관하게!

```python
class MaxFlowEdmondsKarp:
    """
    Edmonds-Karp: BFS로 최단 증가 경로를 탐색.
    시간 복잡도: O(V * E²) — 용량과 무관하게 다항 시간 보장.
    """
    
    def __init__(self, num_vertices: int):
        self.n = num_vertices
        self.graph = [[0] * num_vertices for _ in range(num_vertices)]
    
    def add_edge(self, u: int, v: int, capacity: int):
        self.graph[u][v] += capacity
    
    def _bfs(self, source: int, sink: int, parent: list) -> bool:
        """BFS로 s → t 경로 탐색. parent 배열에 경로 기록."""
        visited = set([source])
        queue = deque([source])
        while queue:
            u = queue.popleft()
            for v in range(self.n):
                if v not in visited and self.graph[u][v] > 0:
                    visited.add(v)
                    parent[v] = u
                    if v == sink:
                        return True
                    queue.append(v)
        return False
    
    def max_flow(self, source: int, sink: int) -> int:
        total_flow = 0
        parent = [-1] * self.n
        
        while self._bfs(source, sink, parent):
            # 경로의 최소 잔여 용량(병목) 찾기
            path_flow = float('inf')
            v = sink
            while v != source:
                u = parent[v]
                path_flow = min(path_flow, self.graph[u][v])
                v = u
            
            # 경로 따라 유량 갱신
            v = sink
            while v != source:
                u = parent[v]
                self.graph[u][v] -= path_flow
                self.graph[v][u] += path_flow
                v = u
            
            total_flow += path_flow
            parent = [-1] * self.n  # reset
        
        return total_flow
```

---

## Dinic's 알고리즘: 레벨 그래프로 O(V²E)

**Dinic's 알고리즘**(Yefim Dinitz, 1970)은 현재 가장 실용적인 최대 유량 알고리즘이다. 핵심 아이디어는 **레벨 그래프(Level Graph)** 와 **블로킹 유량(Blocking Flow)** 이다.

**알고리즘 개요:**
1. BFS로 `s`에서 각 정점까지의 레벨(최단 거리)을 계산하여 **레벨 그래프** 구축.
2. 레벨 그래프에서 DFS로 블로킹 유량 찾기. (더 이상 s→t 경로 없을 때까지)
3. 잔여 그래프 갱신 후 1로 돌아감. `t`에 BFS가 도달 못하면 종료.

시간 복잡도: `O(V²E)`. 단위 용량 그래프에서는 `O(E√V)`.

```python
class MaxFlowDinics:
    """
    Dinic's Algorithm — 실용적 최대 유량 알고리즘.
    시간 복잡도: O(V² * E). 이분 그래프에서 O(E√V).
    """
    
    def __init__(self, num_vertices: int):
        self.n = num_vertices
        # 인접 리스트 방식 (대용량 그래프에 더 효율적)
        # graph[u] = [(v, cap, rev_idx), ...]
        self.graph: List[List[List]] = [[] for _ in range(num_vertices)]
    
    def add_edge(self, u: int, v: int, capacity: int):
        """순방향/역방향 간선 쌍으로 추가."""
        self.graph[u].append([v, capacity, len(self.graph[v])])
        self.graph[v].append([u, 0, len(self.graph[u]) - 1])  # 역방향 용량 0
    
    def _bfs(self, source: int, sink: int) -> bool:
        """BFS로 레벨 그래프 구축. 싱크에 도달 가능하면 True."""
        self.level = [-1] * self.n
        self.level[source] = 0
        queue = deque([source])
        while queue:
            u = queue.popleft()
            for v, cap, _ in self.graph[u]:
                if cap > 0 and self.level[v] < 0:
                    self.level[v] = self.level[u] + 1
                    queue.append(v)
        return self.level[sink] >= 0
    
    def _dfs(self, u: int, sink: int, pushed: float) -> float:
        """레벨 그래프에서 DFS로 블로킹 유량 탐색."""
        if u == sink:
            return pushed
        while self.iter[u] < len(self.graph[u]):
            v, cap, rev = self.graph[u][self.iter[u]]
            if cap > 0 and self.level[v] == self.level[u] + 1:
                d = self._dfs(v, sink, min(pushed, cap))
                if d > 0:
                    self.graph[u][self.iter[u]][1] -= d  # 순방향 용량 감소
                    self.graph[v][rev][1] += d            # 역방향 용량 증가
                    return d
            self.iter[u] += 1  # dead-end: 이 간선 건너뜀 (최적화)
        return 0
    
    def max_flow(self, source: int, sink: int) -> int:
        total_flow = 0
        while self._bfs(source, sink):
            self.iter = [0] * self.n  # 각 정점의 현재 간선 포인터
            while True:
                f = self._dfs(source, sink, float('inf'))
                if f == 0:
                    break
                total_flow += f
        return total_flow
```

---

## 응용: 이분 매칭(Bipartite Matching)

이분 그래프(bipartite graph)의 **최대 매칭(Maximum Bipartite Matching)** 은 최대 유량 문제로 환원할 수 있다.

```python
def bipartite_matching(n: int, m: int, edges: List[Tuple[int, int]]) -> int:
    """
    이분 매칭: 왼쪽 n개, 오른쪽 m개 정점.
    source(0) → 왼쪽(1..n) → 오른쪽(n+1..n+m) → sink(n+m+1)
    모든 내부 간선 용량 = 1 (각 정점은 하나의 매칭만 가능)
    
    예: 구직자(n명)와 직무(m개) 매칭. edges = [(구직자, 직무), ...]
    """
    total = n + m + 2
    source, sink = 0, total - 1
    
    flow_network = MaxFlowDinics(total)
    
    # source → 왼쪽 정점들
    for i in range(1, n + 1):
        flow_network.add_edge(source, i, 1)
    
    # 오른쪽 정점들 → sink
    for j in range(n + 1, n + m + 1):
        flow_network.add_edge(j, sink, 1)
    
    # 이분 간선
    for u, v in edges:
        flow_network.add_edge(u, n + v, 1)
    
    return flow_network.max_flow(source, sink)


# 예시: 4명의 구직자, 4개의 직무
# 구직자 1: 직무 1, 2 가능
# 구직자 2: 직무 1, 3 가능
# 구직자 3: 직무 2, 4 가능
# 구직자 4: 직무 3, 4 가능
edges = [(1, 1), (1, 2), (2, 1), (2, 3), (3, 2), (3, 4), (4, 3), (4, 4)]
result = bipartite_matching(4, 4, edges)
print(f"최대 매칭 수: {result}")  # 4 (모두 매칭 가능)
```

---

## 알고리즘별 복잡도 비교

| 알고리즘 | 시간 복잡도 | 조건 |
|---------|----------|------|
| Ford-Fulkerson (DFS) | O(E × |f*|) | 정수 용량, 종료 보장 불가(무리수) |
| Edmonds-Karp (BFS) | O(V × E²) | 항상 다항 시간 |
| Dinic's | O(V² × E) | 항상 다항 시간 |
| Dinic's (단위 용량) | O(E × √V) | 이분 매칭 최적 |
| Push-Relabel | O(V² × √E) | 조밀 그래프에서 Dinic's보다 빠름 |

---

## 다른 응용 문제들

**최소 컷(Min Cut)**: 최대 유량을 구한 뒤, 잔여 그래프에서 source로부터 BFS/DFS로 도달 가능한 정점 집합을 찾으면 된다. 도달 가능 집합과 불가능 집합을 나누는 간선이 최소 컷을 이룬다.

**프로젝트 선택(Project Selection)**: "프로젝트 선택 문제"는 프로젝트간 의존성과 이익/비용이 있을 때 최대 이익을 내는 프로젝트 집합을 선택하는 문제로, **최소 컷**으로 해결한다.

**이분 커버(Bipartite Vertex Cover)**: König's 정리에 의해 이분 그래프에서 최대 매칭 = 최소 정점 커버이므로, 최대 유량으로 해결 가능.

**주의사항:**
1. **int 오버플로우**: 용량이 크면 유량 합계가 int 범위를 초과할 수 있다. `long long` (C++) 또는 `int` (Python 기본값은 big integer)를 사용하라.
2. **다중 간선(Multi-edge)**: 같은 정점 쌍에 여러 간선이 있을 때는 `graph[u][v] += capacity`로 합산하거나, 인접 리스트 방식(Dinic's처럼)에서 개별 간선으로 처리한다.
3. **음수 용량**: 기본 알고리즘은 비음수 용량을 가정한다. 음수 하한(lower bound)이 있는 유량 문제는 변환 과정이 필요하다.
4. **실수 용량**: Ford-Fulkerson의 종료를 보장하려면 정수나 유리수 용량이 필요하다. Edmonds-Karp나 Dinic's는 실수도 가능하지만 수치 오차에 주의해야 한다.

최대 유량은 "흐름"이 있는 모든 최적화 문제의 토대다. 네트워크 설계, 스케줄링, 컴퓨터 비전(이미지 세그멘테이션), 화학 반응 모델링까지 — 최소 컷/최대 유량 패러다임으로 모델링되는 문제를 발견하는 눈이 생기면, 그 순간 복잡해 보이던 최적화 문제가 표준 알고리즘으로 해결 가능해진다.

## 참고 자료
- [Ford-Fulkerson Algorithm - Wikipedia](https://en.wikipedia.org/wiki/Ford%E2%80%93Fulkerson_algorithm)
- [Dinic's Algorithm - Wikipedia](https://en.wikipedia.org/wiki/Dinic%27s_algorithm)
- [Maximum flow - Ford-Fulkerson and Edmonds-Karp (CP-Algorithms)](https://cp-algorithms.com/graph/edmonds_karp.html)
- [Dinic's Algorithm (CP-Algorithms)](https://cp-algorithms.com/graph/dinic.html)
