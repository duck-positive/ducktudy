---
layout: post
title: "네트워크 플로우 완전 정복: Ford-Fulkerson, Edmonds-Karp로 최대 유량 문제 해결하기"
date: 2026-06-29
categories: [cs, computer-science]
tags: [network-flow, max-flow, ford-fulkerson, edmonds-karp, min-cut, bipartite-matching, graph-algorithm]
---

## 개념 설명: 네트워크 플로우란 무엇인가

**네트워크 플로우(Network Flow)**는 그래프에서 소스(source)에서 싱크(sink)까지 흘릴 수 있는 최대 "양"을 구하는 문제다. 파이프를 통한 물의 흐름, 도로 네트워크의 교통량, 통신 네트워크의 처리량 등 현실 세계의 수많은 문제가 네트워크 플로우로 모델링된다.

### 핵심 용어 정의

**플로우 네트워크(Flow Network)**: 가중치 방향 그래프 G = (V, E)로, 각 간선 (u, v)에는 **용량(capacity)** c(u, v) ≥ 0이 부여된다. 특별한 두 노드 **소스 s**와 **싱크 t**가 존재한다.

**플로우(Flow)**: 각 간선에 할당된 값 f(u, v)로, 두 가지 조건을 만족해야 한다:
- **용량 제약**: 0 ≤ f(u, v) ≤ c(u, v) — 용량을 초과할 수 없다.
- **흐름 보존**: s와 t를 제외한 모든 노드에서 들어오는 플로우의 합 = 나가는 플로우의 합.

**최대 플로우**: |f| = Σ f(s, v) — 소스에서 나가는 플로우의 총합을 최대화하는 것이 목표다.

**잔여 그래프(Residual Graph)**: 현재 플로우 f가 주어졌을 때, 추가로 흘릴 수 있는 양을 나타내는 그래프. 정방향 간선에는 잔여 용량 c(u,v) − f(u,v), 역방향 간선에는 f(u,v) 용량이 부여된다. 역방향 간선은 이미 보낸 플로우를 "취소"하는 개념으로 이해하면 된다.

**증가 경로(Augmenting Path)**: 잔여 그래프에서 s→t로 가는 경로. 이 경로를 따라 추가로 플로우를 흘릴 수 있다. 더 이상 증가 경로가 없으면 최대 플로우에 도달한 것이다.

### 최대 유량-최소 컷 정리 (Max-Flow Min-Cut Theorem)

이 정리는 그래프 이론에서 가장 우아한 결과 중 하나다:

> **최대 플로우의 값 = 최소 컷의 용량**

**컷(Cut)**이란 V를 두 집합 S(s 포함)와 T(t 포함)로 분리하는 것이다. 컷의 용량은 S에서 T로 향하는 간선들의 용량 합이다. **최소 컷**은 이 값이 최소인 컷으로, 네트워크의 병목 구간을 나타낸다.

이 정리의 실용적 의미: 네트워크에서 최대 처리량을 높이려면 최소 컷에 포함된 간선의 용량을 늘려야 한다. 즉 최대 플로우 알고리즘이 동시에 네트워크의 병목을 찾아주는 것이다.

---

## 왜 이 알고리즘이 필요한가

네트워크 플로우는 언뜻 단순해 보이지만 놀랍도록 다양한 문제로 환원된다:

- **이분 매칭(Bipartite Matching)**: 이분 그래프에서 최대 매칭 = 최대 플로우 (소스에서 왼쪽 노드, 오른쪽 노드에서 싱크에 용량 1 간선 추가)
- **작업 할당(Assignment Problem)**: 직원을 업무에 배정하는 최대 효율 할당
- **프로젝트 선택(Project Selection)**: 이익/비용이 있는 의존성 그래프에서 순이익 최대화
- **선박 화물 최적화**: 항구-항구 경로의 최대 화물 처리량
- **이미지 분할(Image Segmentation)**: 픽셀 그래프에서 전경/배경 분리 (최소 컷 활용)

---

## 실제 구현 예제

### 예제 1: Ford-Fulkerson 알고리즘 (DFS 기반)

Ford-Fulkerson은 잔여 그래프에서 증가 경로를 반복적으로 찾아 플로우를 늘린다. 경로 탐색에 DFS를 사용하면 구현이 단순하지만, 정수가 아닌 용량이나 비효율적인 경로 선택 시 수렴을 보장하지 못한다.

```python
from collections import defaultdict

class MaxFlowFordFulkerson:
    """Ford-Fulkerson 최대 유량 (DFS 기반 증가 경로 탐색)"""

    def __init__(self, n: int):
        self.n = n
        # capacity[u][v] = 잔여 용량
        self.capacity: dict = defaultdict(lambda: defaultdict(int))

    def add_edge(self, u: int, v: int, cap: int):
        """간선 추가 (역방향 간선도 0 용량으로 자동 초기화)"""
        self.capacity[u][v] += cap
        # 역방향 간선을 명시적으로 참조해 초기화 보장
        _ = self.capacity[v][u]

    def _dfs(self, u: int, t: int, pushed: float, visited: set) -> float:
        """DFS로 증가 경로 탐색. pushed = 현재까지 흘릴 수 있는 최소 용량."""
        if u == t:
            return pushed
        visited.add(u)
        for v in list(self.capacity[u]):
            cap = self.capacity[u][v]
            if v not in visited and cap > 0:
                d = self._dfs(v, t, min(pushed, cap), visited)
                if d > 0:
                    self.capacity[u][v] -= d
                    self.capacity[v][u] += d
                    return d
        return 0

    def max_flow(self, s: int, t: int) -> float:
        total = 0.0
        while True:
            pushed = self._dfs(s, t, float('inf'), set())
            if pushed == 0:
                break
            total += pushed
        return total


# ─── 예시: CLRS 교과서 6-노드 예제 ───
#
#           16        12
#    s(0) ──────► 1 ──────► t(5)
#     │       ↗   │   ↘        ↑
#    13     4    9    7      20
#     ▼   ↗      ▼      ↘    │
#     2 ──────► 3 ──────► 4
#          14          4
#
mf = MaxFlowFordFulkerson(6)
edges = [
    (0, 1, 16), (0, 2, 13),
    (1, 2, 4),  (1, 3, 12),
    (2, 4, 14), (3, 2, 9),
    (3, 5, 20), (4, 3, 7),
    (4, 5, 4),
]
for u, v, c in edges:
    mf.add_edge(u, v, c)

result = mf.max_flow(0, 5)
print(f"최대 유량 (Ford-Fulkerson DFS): {result}")  # 기대값: 23
```

### 예제 2: Edmonds-Karp 알고리즘 + 이분 매칭 응용

Edmonds-Karp는 Ford-Fulkerson과 동일하지만 **BFS**로 **최단 증가 경로**(hop 수 기준)를 탐색한다. 이로써 시간 복잡도가 O(VE²)로 엄밀하게 보장된다. 이분 그래프 매칭에 바로 적용할 수 있다.

```python
from collections import deque

class MaxFlowEdmondsKarp:
    """Edmonds-Karp 최대 유량 (BFS 기반, O(VE²) 보장)"""

    def __init__(self, n: int):
        self.n = n
        # graph[u] = u와 인접한 노드 목록 (역방향 포함)
        self.graph: list = [[] for _ in range(n)]
        self.capacity: list = [[0] * n for _ in range(n)]

    def add_edge(self, u: int, v: int, cap: int):
        self.graph[u].append(v)
        self.graph[v].append(u)   # 역방향 간선 (초기 용량 0)
        self.capacity[u][v] += cap

    def _bfs(self, s: int, t: int, parent: list) -> bool:
        """BFS로 최단 증가 경로 탐색 (잔여 용량 > 0인 간선만 사용)"""
        visited = [False] * self.n
        visited[s] = True
        queue = deque([s])
        while queue:
            u = queue.popleft()
            for v in self.graph[u]:
                if not visited[v] and self.capacity[u][v] > 0:
                    visited[v] = True
                    parent[v] = u
                    if v == t:
                        return True
                    queue.append(v)
        return False

    def max_flow(self, s: int, t: int) -> int:
        parent = [-1] * self.n
        total = 0
        while self._bfs(s, t, parent):
            # 경로를 역추적하여 최소 잔여 용량 탐색
            path_flow = float('inf')
            v = t
            while v != s:
                u = parent[v]
                path_flow = min(path_flow, self.capacity[u][v])
                v = u
            # 잔여 용량 업데이트
            v = t
            while v != s:
                u = parent[v]
                self.capacity[u][v] -= path_flow
                self.capacity[v][u] += path_flow
                v = u
            total += path_flow
            parent = [-1] * self.n
        return total


# ─── 응용: 직원-업무 이분 매칭 ───
# 노드 구성: 직원(0,1,2) | 업무(3,4,5) | 소스(6) | 싱크(7)
def bipartite_matching_demo():
    mf = MaxFlowEdmondsKarp(8)
    s, t = 6, 7

    # 소스 → 직원 (용량 1: 각 직원은 1개 업무만)
    for emp in range(3):
        mf.add_edge(s, emp, 1)

    # 직원 → 수행 가능 업무
    # 직원0: 업무A, 업무B
    mf.add_edge(0, 3, 1)
    mf.add_edge(0, 4, 1)
    # 직원1: 업무A, 업무C
    mf.add_edge(1, 3, 1)
    mf.add_edge(1, 5, 1)
    # 직원2: 업무B, 업무C
    mf.add_edge(2, 4, 1)
    mf.add_edge(2, 5, 1)

    # 업무 → 싱크 (용량 1: 각 업무는 1명만)
    for job in range(3, 6):
        mf.add_edge(job, t, 1)

    result = mf.max_flow(s, t)
    print(f"\n최대 이분 매칭: {result}쌍")  # 기대값: 3

    # 어떤 매칭이 이루어졌는지 확인
    job_names = {3: "업무A", 4: "업무B", 5: "업무C"}
    emp_names = {0: "직원0", 1: "직원1", 2: "직원2"}
    print("매칭 결과:")
    for emp in range(3):
        for job in range(3, 6):
            # 용량이 0으로 줄었다 = 해당 간선에 플로우가 흘렀다
            if mf.capacity[emp][job] == 0:
                print(f"  {emp_names[emp]} ↔ {job_names[job]}")


bipartite_matching_demo()
```

---

## 주의사항과 실전 팁

### 1. 알고리즘 선택 가이드

| 알고리즘 | 시간 복잡도 | 특징 |
|---------|------------|------|
| Ford-Fulkerson (DFS) | O(E × max_flow) | 정수 용량에서만 안전, 구현 간단 |
| Edmonds-Karp (BFS) | O(VE²) | 정수/실수 모두 안전, 범용 |
| Dinic's | O(V²E) | 이분 그래프에서 O(E√V), 고성능 |
| Push-Relabel | O(V²√E) | 대규모 그래프에서 실용적 |

노드 수 100 이하의 그래프라면 Edmonds-Karp로 충분하다. 경쟁 프로그래밍에서는 Dinic's 알고리즘이 가장 선호된다.

### 2. 역방향 간선을 반드시 추가하라

Ford-Fulkerson/Edmonds-Karp에서 역방향 간선은 핵심이다. 이미 보낸 플로우를 취소(undo)할 수 있어야만 글로벌 최적 해를 보장한다. `add_edge(u, v, cap)` 시 역방향 간선 초기화를 빠뜨리면 서브옵티멀 해가 나온다.

예를 들어 두 경로 s→a→b→t와 s→a→c→t가 있고 s→a 용량이 1이라면, DFS가 먼저 s→a→b→t를 선택하더라도 역방향 간선 b→a를 통해 s→a→c→t + s→b→t(역방향)로 재최적화할 수 있다.

### 3. 부동소수점 오차 처리

실수 용량을 사용할 때는 비교에 엡실론을 사용한다:

```python
EPSILON = 1e-9

# 잔여 용량이 있는지 확인할 때
if self.capacity[u][v] > EPSILON:  # 올바름
    ...
if self.capacity[u][v] > 0:        # 부동소수점 오차로 무한 루프 가능
    ...
```

### 4. 다중 소스/싱크 처리

소스나 싱크가 여러 개인 경우, 가상의 슈퍼 소스(super source)와 슈퍼 싱크(super sink)를 추가하고 각 실제 소스/싱크에 무한 용량 간선을 연결한다. 코드 변경 없이 바로 적용할 수 있다.

### 5. 실전에서의 응용 패턴

- **이분 매칭 문제**: 호텔 객실 예약, 시험 감독관 배정
- **최소 비용 최대 유량(MCMF)**: 비용이 있는 간선에서 최대 유량의 최소 비용을 구한다. Bellman-Ford 기반 SPFA나 Successive Shortest Path 알고리즘을 사용한다
- **순환 플로우**: 소스/싱크가 없는 순환 그래프에서 하한(lower bound)이 있는 플로우 문제

---

## 참고 자료
- [Ford–Fulkerson algorithm — Wikipedia](https://en.wikipedia.org/wiki/Ford%E2%80%93Fulkerson_algorithm)
- [Edmonds–Karp algorithm — Wikipedia](https://en.wikipedia.org/wiki/Edmonds%E2%80%93Karp_algorithm)
- [Maximum flow - Ford-Fulkerson and Edmonds-Karp — cp-algorithms.com](https://cp-algorithms.com/graph/edmonds_karp.html)
- [Network flow problem — Wikipedia](https://en.wikipedia.org/wiki/Network_flow_problem)
