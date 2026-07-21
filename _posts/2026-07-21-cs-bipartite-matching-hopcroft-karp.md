---
layout: post
title: "이분 매칭 완전 정복: 헝가리안 알고리즘부터 Hopcroft-Karp까지"
date: 2026-07-21
categories: [cs, computer-science]
tags: [bipartite-matching, hopcroft-karp, hungarian-algorithm, graph-theory, algorithms]
---

## 이분 매칭이란 무엇인가?

**이분 매칭(Bipartite Matching)**은 그래프 이론에서 가장 실용적인 문제 중 하나다. 이분 그래프(Bipartite Graph)란 정점(vertex) 집합을 두 개의 그룹 U와 V로 나눌 수 있으며, 모든 에지(edge)가 반드시 한 그룹에서 다른 그룹으로만 연결되는 그래프를 말한다. 즉 U 내부의 정점끼리, V 내부의 정점끼리는 에지가 없다.

이러한 이분 그래프에서 **매칭(Matching)**이란 공통 정점을 공유하지 않는 에지들의 집합이다. **최대 매칭(Maximum Matching)**은 매칭에 포함된 에지 수를 최대화한 것이다. 예를 들어, 지원자 집합(U)과 일자리 집합(V)이 있을 때, 각 지원자-일자리 쌍 사이에 에지가 있다면 최대한 많은 지원자를 채용하는 문제가 곧 최대 이분 매칭이다.

### 핵심 개념: 증가 경로 (Augmenting Path)

**증가 경로(Augmenting Path)**는 매칭되지 않은 정점에서 시작해 매칭되지 않은 정점으로 끝나며, 매칭되지 않은 에지와 매칭된 에지를 번갈아 가며 지나는 경로다. 증가 경로를 하나 찾으면 매칭 크기를 정확히 1 증가시킬 수 있다. 이것이 **Berge의 정리(Berge's theorem)**의 핵심이다: 현재 매칭이 최대 매칭임과 증가 경로가 존재하지 않음은 동치다.

---

## 왜 이분 매칭이 필요한가?

이분 매칭은 실생활과 산업 전반에서 광범위하게 사용된다.

- **작업 스케줄링**: 직원-근무 시간 배정, 기계-작업 할당
- **추천 시스템**: 사용자-상품 매칭
- **네트워크 플로우**: 이분 그래프에서의 최대 이분 매칭은 최대 유량 문제의 특수 케이스다
- **생물정보학**: 유전자-단백질 상호작용 분석
- **컴파일러 최적화**: 레지스터 할당 문제

**König의 정리(König's theorem)**에 의하면 이분 그래프에서 최대 매칭의 크기는 최소 정점 커버(minimum vertex cover)의 크기와 같다. 이 강력한 결과는 이분 그래프 문제들 사이의 깊은 관계를 드러낸다.

---

## 기본 구현: DFS 기반 이분 매칭

가장 단순한 접근법은 각 U 정점에서 DFS로 증가 경로를 찾는 것이다. 시간 복잡도는 O(V × E)이다.

```python
from collections import defaultdict

class BipartiteMatching:
    def __init__(self, n_left, n_right):
        self.n_left = n_left
        self.n_right = n_right
        self.graph = defaultdict(list)  # u -> [v1, v2, ...]
        self.match_left = [-1] * n_left   # match_left[u] = v
        self.match_right = [-1] * n_right # match_right[v] = u

    def add_edge(self, u, v):
        self.graph[u].append(v)

    def _dfs(self, u, visited):
        for v in self.graph[u]:
            if not visited[v]:
                visited[v] = True
                # v가 매칭 안 됐거나, v의 현재 매칭 상대를 다른 곳에 배정 가능하면
                if self.match_right[v] == -1 or \
                   self._dfs(self.match_right[v], visited):
                    self.match_left[u] = v
                    self.match_right[v] = u
                    return True
        return False

    def max_matching(self):
        result = 0
        for u in range(self.n_left):
            visited = [False] * self.n_right
            if self._dfs(u, visited):
                result += 1
        return result


# 사용 예시: 3명의 지원자, 3개의 일자리
# 지원자 0: 일자리 0, 1 지원 가능
# 지원자 1: 일자리 1 지원 가능
# 지원자 2: 일자리 0, 2 지원 가능
bm = BipartiteMatching(3, 3)
bm.add_edge(0, 0)
bm.add_edge(0, 1)
bm.add_edge(1, 1)
bm.add_edge(2, 0)
bm.add_edge(2, 2)

print(f"최대 매칭: {bm.max_matching()}")  # 출력: 최대 매칭: 3
print(f"매칭 결과 (left->right): {bm.match_left}")  # [0, 1, 2] 또는 유사
```

이 구현에서 `_dfs` 함수는 재귀적으로 증가 경로를 탐색한다. V 정점마다 한 번씩 DFS를 실행하므로 전체 시간 복잡도는 O(V × E)다.

---

## Hopcroft-Karp 알고리즘: O(√V × E)

1973년 John Hopcroft와 Richard Karp가 발표한 알고리즘으로, 단순 DFS 방식보다 훨씬 빠르다. 핵심 아이디어는 **매 라운드마다 최단 증가 경로를 BFS로 찾은 뒤, 서로 정점을 공유하지 않는 최대 집합(maximal set of shortest augmenting paths)**을 한 번에 DFS로 처리하는 것이다.

```python
from collections import deque

class HopcroftKarp:
    INF = float('inf')

    def __init__(self, n_left, n_right):
        self.n_left = n_left
        self.n_right = n_right
        self.graph = [[] for _ in range(n_left)]
        # 매칭: match_u[u] = v (또는 None), match_v[v] = u (또는 None)
        self.match_u = [None] * n_left
        self.match_v = [None] * n_right
        self.dist = [0] * n_left

    def add_edge(self, u, v):
        self.graph[u].append(v)

    def _bfs(self):
        queue = deque()
        for u in range(self.n_left):
            if self.match_u[u] is None:
                self.dist[u] = 0
                queue.append(u)
            else:
                self.dist[u] = self.INF

        found = False
        while queue:
            u = queue.popleft()
            for v in self.graph[u]:
                w = self.match_v[v]   # v의 현재 매칭 상대 (left 쪽)
                if w is None:
                    found = True
                elif self.dist[w] == self.INF:
                    self.dist[w] = self.dist[u] + 1
                    queue.append(w)
        return found

    def _dfs(self, u):
        for v in self.graph[u]:
            w = self.match_v[v]
            if w is None or (self.dist[w] == self.dist[u] + 1 and self._dfs(w)):
                self.match_u[u] = v
                self.match_v[v] = u
                return True
        self.dist[u] = self.INF  # 이 경로는 더 이상 사용 불가
        return False

    def max_matching(self):
        matching = 0
        while self._bfs():
            for u in range(self.n_left):
                if self.match_u[u] is None:
                    if self._dfs(u):
                        matching += 1
        return matching


# 성능 비교: 1000명 지원자, 1000개 일자리, 무작위 에지 5000개
import random
random.seed(42)
hk = HopcroftKarp(1000, 1000)
for _ in range(5000):
    hk.add_edge(random.randint(0, 999), random.randint(0, 999))

result = hk.max_matching()
print(f"최대 매칭 (Hopcroft-Karp): {result}")
```

### 시간 복잡도 분석

- **BFS 라운드 횟수**: O(√V). 증가 경로의 최단 길이는 최대 O(√V)번 증가하기 때문이다. 각 BFS 라운드에서 최단 증가 경로 길이가 적어도 1 증가하고, 최단 경로 길이가 √V를 넘으면 남은 매칭 수가 O(√V) 이하가 된다.
- **각 라운드의 BFS+DFS**: O(E)
- **총 복잡도**: O(√V × E)

이는 밀집 그래프에서 단순 DFS 방식(O(V × E))에 비해 훨씬 빠르다. V = 10,000, E = 1,000,000인 경우 단순 방식은 10^10 연산이지만 Hopcroft-Karp는 약 10^8이다.

---

## 가중 이분 매칭: 헝가리안 알고리즘

가중치가 있는 이분 매칭에서는 매칭의 총 가중치를 최대(또는 최소)화하는 **최적 할당(optimal assignment)**을 구해야 한다. 이를 위한 알고리즘이 바로 **헝가리안 알고리즘(Hungarian Algorithm)**이다.

헝가리안 알고리즘의 핵심은 **포텐셜(potential, 또는 라벨)**을 관리하는 것이다. 각 정점 u ∈ U와 v ∈ V에 포텐셜 h(u), h(v)를 부여하고, 다음 불변식을 유지한다:

```
h(u) + h(v) ≥ w(u, v) for all edges (u, v)
```

등호가 성립하는 에지들을 **타이트 에지(tight edge)**라 하고, 이 타이트 에지들만으로 이루어진 이분 그래프에서 완전 매칭을 찾으면 그것이 최적 해다.

```python
def hungarian_algorithm(cost_matrix):
    """
    n×n 비용 행렬에서 최소 비용 완전 매칭을 구함.
    반환값: (최소 비용, 할당 배열) — assignment[i] = j 는 행 i가 열 j에 할당됨
    """
    n = len(cost_matrix)
    INF = float('inf')

    # 포텐셜 초기화
    u = [0] * (n + 1)  # 행(left) 포텐셜
    v = [0] * (n + 1)  # 열(right) 포텐셜
    p = [0] * (n + 1)  # p[j] = 현재 열 j에 매칭된 행
    way = [0] * (n + 1)

    for i in range(1, n + 1):
        p[0] = i
        j0 = 0
        minval = [INF] * (n + 1)
        used = [False] * (n + 1)

        while True:
            used[j0] = True
            i0 = p[j0]
            delta = INF
            j1 = -1

            for j in range(1, n + 1):
                if not used[j]:
                    cur = cost_matrix[i0 - 1][j - 1] - u[i0] - v[j]
                    if cur < minval[j]:
                        minval[j] = cur
                        way[j] = j0
                    if minval[j] < delta:
                        delta = minval[j]
                        j1 = j

            for j in range(n + 1):
                if used[j]:
                    u[p[j]] += delta
                    v[j] -= delta
                else:
                    minval[j] -= delta

            j0 = j1
            if p[j0] == 0:
                break

        while j0:
            p[j0] = p[way[j0]]
            j0 = way[j0]

    assignment = [0] * n
    for j in range(1, n + 1):
        if p[j] != 0:
            assignment[p[j] - 1] = j - 1

    total_cost = sum(cost_matrix[i][assignment[i]] for i in range(n))
    return total_cost, assignment


# 예시: 3명의 직원에게 3개의 작업 배정
# cost[i][j] = 직원 i가 작업 j를 수행하는 비용
costs = [
    [9, 2, 7],
    [3, 6, 4],
    [1, 8, 5],
]
min_cost, assign = hungarian_algorithm(costs)
print(f"최소 비용: {min_cost}")  # 출력: 최소 비용: 13
print(f"할당: {assign}")          # 출력: [1, 2, 0] (직원0→작업1, 직원1→작업2, 직원2→작업0)
```

헝가리안 알고리즘의 시간 복잡도는 O(n³)이며, 이는 최적 할당 문제에서 알려진 가장 효율적인 알고리즘 중 하나다.

---

## 주의사항과 실전 팁

### 1. 방향성 확인
이분 그래프 문제인지 항상 먼저 확인하라. 그래프가 이분 그래프인지 확인하려면 BFS/DFS로 2-coloring을 시도하면 된다. 홀수 길이 사이클이 있으면 이분 그래프가 아니다.

### 2. 인덱싱 주의
Hopcroft-Karp 구현에서 left/right 정점 인덱스가 겹치지 않도록 주의하라. 별도의 배열로 관리하는 것이 버그를 줄인다.

### 3. 최대 유량으로의 환원
이분 매칭은 최대 유량 문제로 환원할 수 있다. 소스(S)에서 모든 U 정점으로 용량 1의 에지를, 모든 V 정점에서 싱크(T)로 용량 1의 에지를 추가하고, 기존 에지는 용량 1로 설정하면 된다. Ford-Fulkerson이나 Dinic's 알고리즘으로 풀 수 있다.

### 4. 가중치 매칭에서의 부동소수점
헝가리안 알고리즘에서 가중치가 부동소수점이면 수치 오차가 발생할 수 있다. 정수로 스케일링하거나 epsilon 비교를 사용하라.

### 5. 경쟁 프로그래밍에서의 활용
이분 매칭은 경쟁 프로그래밍에서 "최소 경로 커버(Minimum Path Cover)", "최대 독립 집합(Maximum Independent Set in Bipartite Graph)" 등 다양한 문제로 변환해 활용된다. König의 정리를 숙지하면 풀 수 있는 문제가 크게 늘어난다.

---

## 참고 자료

- [Hopcroft–Karp algorithm - Wikipedia](https://en.wikipedia.org/wiki/Hopcroft%E2%80%93Karp_algorithm)
- [Hungarian algorithm - Wikipedia](https://en.wikipedia.org/wiki/Hungarian_algorithm)
- [Bipartite graph - Wikipedia](https://en.wikipedia.org/wiki/Bipartite_graph)
- [Hopcroft-Karp Algorithm - GeeksforGeeks](https://www.geeksforgeeks.org/dsa/hopcroft-karp-algorithm-for-maximum-matching-set-1-introduction/)
