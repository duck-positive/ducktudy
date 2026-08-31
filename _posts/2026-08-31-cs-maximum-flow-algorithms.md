---
layout: post
title: "최대 유량(Maximum Flow) 완전 정복: Ford-Fulkerson·Edmonds-Karp·Dinic 알고리즘과 최소 컷·이분 매칭 응용"
date: 2026-08-31
categories: [cs, computer-science]
tags: [maximum-flow, ford-fulkerson, edmonds-karp, dinic, min-cut, bipartite-matching, graph-algorithm, network-flow]
---

## 개념 설명

**최대 유량 문제(Maximum Flow Problem)**는 방향 그래프 G = (V, E)에서 각 간선 (u, v)에 용량(capacity) c(u, v)가 주어질 때, **소스(source) s**에서 **싱크(sink) t**로 흘릴 수 있는 최대 흐름을 구하는 문제입니다.

유량 함수 f(u, v)는 다음 세 가지 조건을 만족해야 합니다:

1. **용량 제한**: f(u, v) ≤ c(u, v) — 각 간선에 흐르는 유량은 용량을 초과할 수 없음
2. **반대칭성**: f(u, v) = -f(v, u) — 역방향 간선에는 음의 유량
3. **유량 보존**: 소스와 싱크를 제외한 모든 정점에서 들어오는 유량 합 = 나가는 유량 합

**잔여 그래프(Residual Graph)**는 현재 유량 기준으로 추가로 흘릴 수 있는 용량을 나타냅니다. 간선 (u, v)의 잔여 용량은 `c(u, v) - f(u, v)`이며, 역방향 간선 (v, u)의 잔여 용량은 `f(u, v)`입니다.

**최대 유량-최소 컷 정리(Max-Flow Min-Cut Theorem)**: 그래프에서의 최대 유량 = 소스와 싱크를 분리하는 최소 컷의 용량. 이 강력한 쌍대 정리가 최대 유량 알고리즘의 이론적 기반입니다.

---

## 왜 필요한가?

최대 유량은 표면적으로는 그래프 이론 문제처럼 보이지만, 놀라울 만큼 다양한 현실 문제로 환원됩니다:

| 응용 분야                      | 최대 유량으로 환원하는 방법                         |
|------------------------------|--------------------------------------------------|
| **이분 그래프 최대 매칭**       | 소스 → 좌측 정점, 우측 정점 → 싱크, 모든 용량 1    |
| **프로젝트 선택 문제**          | 최소 컷으로 이익/비용 구조 모델링                   |
| **네트워크 신뢰성**             | 최소 컷 = 끊어야 할 최소 간선 수                   |
| **작업 스케줄링**               | 시간 슬롯과 작업 사이 유량 모델링                   |
| **이미지 분할(Image Segmentation)**| 픽셀 간 유사도를 그래프로 모델링, 최소 컷으로 분할 |
| **다중 상품 유량**              | 물류·통신 네트워크 최적화                           |

---

## 실제 구현 예제

### 예제 1: Dinic 알고리즘 (Python)

Dinic 알고리즘은 **레벨 그래프(Level Graph)**를 BFS로 구축한 뒤 **블로킹 유량(Blocking Flow)**을 DFS로 찾기를 반복합니다. 시간 복잡도는 일반 그래프에서 **O(V²E)**, 단위 용량 그래프에서는 **O(E√V)**입니다.

```python
from collections import deque, defaultdict

class Dinic:
    """Dinic 최대 유량 알고리즘 O(V^2 * E)"""

    def __init__(self, n: int):
        self.n = n
        self.graph = [[] for _ in range(n)]  # graph[u] = [(v, rev_idx, cap), ...]
        # 실제로는 그래프를 인접 리스트로 표현; rev_idx로 역방향 간선 참조

    def add_edge(self, u: int, v: int, cap: int):
        """유향 간선 추가 (역방향 간선 자동 추가, 초기 용량 0)"""
        self.graph[u].append([v, len(self.graph[v]), cap])
        self.graph[v].append([u, len(self.graph[u]) - 1, 0])

    def _bfs(self, s: int, t: int) -> bool:
        """레벨 그래프 구축. 싱크 도달 가능 여부 반환"""
        self.level = [-1] * self.n
        self.level[s] = 0
        q = deque([s])
        while q:
            u = q.popleft()
            for v, _, cap in self.graph[u]:
                if cap > 0 and self.level[v] < 0:
                    self.level[v] = self.level[u] + 1
                    q.append(v)
        return self.level[t] >= 0

    def _dfs(self, u: int, t: int, pushed: int) -> int:
        """블로킹 유량 탐색 (iter 배열로 이미 탐색한 간선 스킵)"""
        if u == t:
            return pushed
        while self.iter[u] < len(self.graph[u]):
            v, rev, cap = self.graph[u][self.iter[u]]
            if cap > 0 and self.level[v] == self.level[u] + 1:
                d = self._dfs(v, t, min(pushed, cap))
                if d > 0:
                    self.graph[u][self.iter[u]][2] -= d
                    self.graph[v][rev][2] += d
                    return d
            self.iter[u] += 1
        return 0

    def max_flow(self, s: int, t: int) -> int:
        flow = 0
        while self._bfs(s, t):
            self.iter = [0] * self.n
            while True:
                f = self._dfs(s, t, float('inf'))
                if f == 0:
                    break
                flow += f
        return flow


# 테스트: 교과서 예제
# 정점: 0(소스), 1, 2, 3, 4, 5(싱크)
dinic = Dinic(6)
dinic.add_edge(0, 1, 16)
dinic.add_edge(0, 2, 13)
dinic.add_edge(1, 2,  4)
dinic.add_edge(1, 3, 12)
dinic.add_edge(2, 1,  0)
dinic.add_edge(2, 4, 14)
dinic.add_edge(3, 2,  9)
dinic.add_edge(3, 5, 20)
dinic.add_edge(4, 3,  7)
dinic.add_edge(4, 5,  4)

print("최대 유량:", dinic.max_flow(0, 5))  # 23
```

**핵심 최적화: `iter` 배열(Dead End Elimination)**

`iter[u]`는 정점 u에서 다음에 탐색할 간선의 인덱스를 기록합니다. 블로킹 유량을 찾는 과정에서 이미 막힌 간선을 다시 탐색하지 않아 DFS 탐색이 O(VE)로 제한됩니다.

---

### 예제 2: 이분 그래프 최대 매칭 (C++)

이분 매칭(Bipartite Matching)은 최대 유량의 가장 유명한 응용입니다. 소스에서 좌측 정점, 우측 정점에서 싱크로 용량 1의 간선을 추가하면 최대 유량 = 최대 매칭 수가 됩니다.

```cpp
#include <bits/stdc++.h>
using namespace std;

struct MaxFlow {
    struct Edge { int to, rev; int cap; };
    int n;
    vector<vector<Edge>> graph;
    vector<int> level, iter_;

    MaxFlow(int n) : n(n), graph(n), level(n), iter_(n) {}

    void add_edge(int u, int v, int cap) {
        graph[u].push_back({v, (int)graph[v].size(), cap});
        graph[v].push_back({u, (int)graph[u].size()-1, 0});
    }

    bool bfs(int s, int t) {
        fill(level.begin(), level.end(), -1);
        queue<int> q;
        level[s] = 0; q.push(s);
        while (!q.empty()) {
            int u = q.front(); q.pop();
            for (auto& e : graph[u])
                if (e.cap > 0 && level[e.to] < 0) {
                    level[e.to] = level[u] + 1;
                    q.push(e.to);
                }
        }
        return level[t] >= 0;
    }

    int dfs(int u, int t, int f) {
        if (u == t) return f;
        for (int& i = iter_[u]; i < (int)graph[u].size(); i++) {
            Edge& e = graph[u][i];
            if (e.cap > 0 && level[e.to] == level[u] + 1) {
                int d = dfs(e.to, t, min(f, e.cap));
                if (d > 0) {
                    e.cap -= d;
                    graph[e.to][e.rev].cap += d;
                    return d;
                }
            }
        }
        return 0;
    }

    int max_flow(int s, int t) {
        int flow = 0;
        while (bfs(s, t)) {
            fill(iter_.begin(), iter_.end(), 0);
            int f;
            while ((f = dfs(s, t, INT_MAX)) > 0) flow += f;
        }
        return flow;
    }
};

int bipartite_matching(int L, int R, vector<pair<int,int>>& edges) {
    // 정점: 0=소스, 1..L=좌측, L+1..L+R=우측, L+R+1=싱크
    int S = 0, T = L + R + 1;
    MaxFlow mf(T + 1);

    for (int i = 1; i <= L; i++)    mf.add_edge(S, i, 1);
    for (int j = 1; j <= R; j++)    mf.add_edge(L + j, T, 1);
    for (auto [u, v] : edges)       mf.add_edge(u, L + v, 1);

    return mf.max_flow(S, T);
}

int main() {
    // 좌측 4명, 우측 4명인 이분 그래프
    // (사람, 업무) 쌍: 1→1, 1→2, 2→1, 2→3, 3→2, 3→4, 4→3, 4→4
    int L = 4, R = 4;
    vector<pair<int,int>> edges = {
        {1,1},{1,2},{2,1},{2,3},{3,2},{3,4},{4,3},{4,4}
    };
    cout << "최대 이분 매칭: " << bipartite_matching(L, R, edges) << "\n";  // 4
    return 0;
}
```

**쾨닉의 정리(König's Theorem)**: 이분 그래프에서 최대 이분 매칭의 크기 = 최소 정점 커버(Minimum Vertex Cover)의 크기. 이것도 최소 컷의 직접적인 결과입니다.

---

## 주의사항 및 팁

### 1. 알고리즘 선택 기준

| 알고리즘           | 시간 복잡도       | 추천 상황                            |
|-------------------|-------------------|-------------------------------------|
| Ford-Fulkerson     | O(E × max_flow)  | 유량이 작거나 이론 공부 목적         |
| Edmonds-Karp      | O(VE²)            | 일반 네트워크 유량                   |
| Dinic             | O(V²E)            | 대부분의 경쟁 프로그래밍 상황        |
| Dinic (단위 용량) | O(E√V)            | 이분 매칭, 단위 용량 그래프          |
| HLPPS/Push-Relabel| O(V²√E)           | 매우 큰 용량, 밀집 그래프            |

경쟁 프로그래밍에서는 **Dinic이 사실상 표준**입니다. 구현이 간결하면서도 실제 동작 속도가 매우 빠릅니다.

### 2. 역방향 간선 구현 주의
역방향 간선을 그래프에 함께 저장할 때 인덱스 쌍으로 참조해야 합니다. 간선 추가 시 `graph[u].size()`와 `graph[v].size()`를 **추가 전에** 캡처해야 인덱스 오류가 없습니다.

### 3. 다중 소스/다중 싱크
소스가 여럿이면 **슈퍼 소스(super source)** S를 만들어 각 소스로 ∞ 용량의 간선을 연결합니다. 싱크도 마찬가지로 슈퍼 싱크를 만듭니다. 이 간단한 변환으로 단일 소스-싱크 알고리즘을 그대로 활용할 수 있습니다.

### 4. 정점 용량 제한 처리
정점 v에 용량 c가 있는 경우, v를 `v_in`과 `v_out` 두 정점으로 분리한 뒤 `v_in → v_out` 간선의 용량을 c로 설정합니다. 이 기법으로 간선 용량만 존재하는 표준 최대 유량 문제로 환원합니다.

### 5. 실수 용량과 무한 용량
실수 용량에서는 Ford-Fulkerson(DFS 기반)이 수렴하지 않을 수 있습니다. 반드시 BFS 기반의 Edmonds-Karp나 Dinic을 사용하세요. 무한 용량은 `INT_MAX / 2` 또는 `1e9`처럼 충분히 큰 값으로 설정해 오버플로를 방지합니다.

---

## 참고 자료

- [Maximum Flow - Ford-Fulkerson and Edmonds-Karp - CP-Algorithms](https://cp-algorithms.com/graph/edmonds_karp.html)
- [Maximum Flow - Dinic's Algorithm - CP-Algorithms](https://cp-algorithms.com/graph/dinic.html)
- [Ford–Fulkerson Algorithm - Wikipedia](https://en.wikipedia.org/wiki/Ford%E2%80%93Fulkerson_algorithm)
- [Dinic's Algorithm for Maximum Flow - GeeksforGeeks](https://www.geeksforgeeks.org/dsa/dinics-algorithm-maximum-flow/)
