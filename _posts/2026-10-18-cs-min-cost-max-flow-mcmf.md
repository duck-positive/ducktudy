---
layout: post
title: "최소 비용 최대 유량(MCMF) 완전 정복: Successive Shortest Paths 알고리즘으로 비용 최적화 네트워크 유량 해결하기"
date: 2026-10-18
categories: [cs, computer-science]
tags: [mcmf, min-cost-flow, network-flow, graph-algorithm, competitive-programming, optimization]
---

## 개요

네트워크 유량(network flow) 문제에서 **최대 유량(maximum flow)**만을 구하는 것은 Ford-Fulkerson, Dinic 등의 알고리즘으로 해결할 수 있다. 그런데 현실의 많은 문제는 단순히 많이 보내는 것뿐만 아니라 **가장 저렴한 비용으로** 유량을 보내야 한다. 이것이 **최소 비용 최대 유량(Minimum Cost Maximum Flow, MCMF)** 문제다.

MCMF는 공급망 최적화, 작업 배정(assignment problem), 최소 비용 매칭, 교통 흐름 계획 등 실용적인 분야에서 핵심 알고리즘으로 활용된다. 이 글에서는 **Successive Shortest Paths(SSP) 알고리즘**을 중심으로 MCMF를 완전히 분해한다.

---

## 왜 MCMF가 필요한가

### 최대 유량만으로 부족한 경우

예를 들어 물류 센터 문제를 생각해보자. 여러 공급 지점(source)에서 수요 지점(sink)으로 물건을 보내야 한다. 각 경로에는 운반 가능한 최대 용량(capacity)과 단위당 비용(cost)이 있다. 목표는 "최대한 많이 보내면서 총 비용을 최소화하는" 것이다.

최대 유량 알고리즘은 비용을 고려하지 않는다. 그냥 통과할 수 있는 모든 경로를 채운다. MCMF 알고리즘은 유량을 증가시킬 때 항상 가장 싼 경로(증가 경로 중 최소 비용 경로)부터 채워 총 비용을 최소화한다.

### 대표 응용 사례

| 문제 | MCMF 모델링 |
|------|------------|
| 이분 매칭 최소 비용 | 좌측 노드 → 우측 노드 간선에 비용 부여 |
| 작업 배정 문제 (Assignment Problem) | 작업자-작업 이분 그래프로 변환 |
| 최소 비용 순환 (min-cost circulation) | 음수 비용 간선 처리 |
| BOJ 11408 (열혈강호 5), BOJ 13166 (고려대학교) | 전형적인 MCMF 문제 |

---

## Successive Shortest Paths (SSP) 알고리즘

### 핵심 아이디어

SSP는 Dijkstra/Bellman-Ford 기반의 **반복적 최단 경로 탐색**을 사용한다.

1. 잔여 그래프(residual graph)에서 소스(S)에서 싱크(T)까지의 **최소 비용 경로(최단 경로)**를 찾는다.
2. 해당 경로를 따라 최대한 유량을 흘린다 (병목 용량만큼).
3. 잔여 그래프를 업데이트하고, 더 이상 경로가 없을 때까지 반복한다.

단계마다 최소 비용 경로를 선택하기 때문에 최종 유량이 최대가 되었을 때 총 비용도 최소가 된다.

### 왜 Bellman-Ford가 필요한가

잔여 그래프에서 역방향 간선의 비용은 **원래 비용의 음수**가 된다 (이미 흐른 유량을 취소하는 비용). 따라서 음수 가중치 간선이 생기고, Dijkstra를 직접 적용할 수 없다. 이를 해결하는 방법은 두 가지다.

- **SPFA (Shortest Path Faster Algorithm)**: Bellman-Ford의 큐 기반 최적화. 구현이 간단하고 음수 간선을 처리할 수 있다.
- **Johnson's Algorithm (Potential Trick)**: 처음 한 번 Bellman-Ford로 포텐셜(potential)을 구한 후, 이후 반복에서 Dijkstra를 쓸 수 있도록 비용을 재가중치한다. 더 빠르지만 구현이 복잡하다.

---

## 구현 예제 1 — Python: SPFA 기반 MCMF

```python
from collections import defaultdict, deque

INF = float('inf')

class MCMF:
    def __init__(self, n: int):
        self.n = n
        self.graph = defaultdict(list)  # graph[u] = [(v, cap, cost, rev_idx)]

    def add_edge(self, u: int, v: int, cap: int, cost: int):
        """유향 간선 추가. 역방향 간선도 함께 추가."""
        forward = [v, cap, cost, len(self.graph[v])]
        backward = [u, 0, -cost, len(self.graph[u])]
        self.graph[u].append(forward)
        self.graph[v].append(backward)

    def _spfa(self, s: int, t: int):
        """SPFA로 최소 비용 경로 탐색. (dist, prev_node, prev_edge_idx) 반환."""
        dist = [INF] * self.n
        in_queue = [False] * self.n
        prev_v = [-1] * self.n
        prev_e = [-1] * self.n

        dist[s] = 0
        q = deque([s])
        in_queue[s] = True

        while q:
            u = q.popleft()
            in_queue[u] = False
            for i, (v, cap, cost, _) in enumerate(self.graph[u]):
                if cap > 0 and dist[u] + cost < dist[v]:
                    dist[v] = dist[u] + cost
                    prev_v[v] = u
                    prev_e[v] = i
                    if not in_queue[v]:
                        q.append(v)
                        in_queue[v] = True

        if dist[t] == INF:
            return False, 0, 0

        # 경로를 따라 최대 유량(병목) 계산
        flow = INF
        v = t
        while v != s:
            u = prev_v[v]
            flow = min(flow, self.graph[u][prev_e[v]][1])
            v = u

        # 경로 업데이트
        total_cost = flow * dist[t]
        v = t
        while v != s:
            u = prev_v[v]
            e = prev_e[v]
            self.graph[u][e][1] -= flow          # 정방향 잔여 용량 감소
            rev = self.graph[u][e][3]
            self.graph[v][rev][1] += flow         # 역방향 잔여 용량 증가
            v = u

        return True, flow, total_cost

    def min_cost_max_flow(self, s: int, t: int):
        """(최대 유량, 최소 비용) 반환."""
        total_flow = 0
        total_cost = 0
        while True:
            ok, flow, cost = self._spfa(s, t)
            if not ok:
                break
            total_flow += flow
            total_cost += cost
        return total_flow, total_cost


# --- 작업 배정 문제 예시 ---
# 작업자 3명, 작업 3개, 각 작업자-작업 쌍의 비용
def assignment_problem():
    # 노드: 0=source, 1~3=작업자, 4~6=작업, 7=sink
    N = 8
    mcmf = MCMF(N)
    S, T = 0, 7

    cost_matrix = [
        [3, 2, 7],  # 작업자1이 작업1,2,3을 수행하는 비용
        [5, 1, 3],  # 작업자2
        [2, 4, 6],  # 작업자3
    ]

    # source → 작업자 (용량 1, 비용 0)
    for i in range(3):
        mcmf.add_edge(S, i + 1, 1, 0)

    # 작업자 → 작업 (용량 1, 각 비용)
    for i in range(3):
        for j in range(3):
            mcmf.add_edge(i + 1, j + 4, 1, cost_matrix[i][j])

    # 작업 → sink (용량 1, 비용 0)
    for j in range(3):
        mcmf.add_edge(j + 4, T, 1, 0)

    flow, cost = mcmf.min_cost_max_flow(S, T)
    print(f"최대 매칭: {flow}, 최소 비용: {cost}")
    # 최적 배정: 작업자1→작업2(2), 작업자2→작업2(1), 작업자3→작업1(2)
    # 실제: 작업자1→작업2, 작업자2→작업3, 작업자3→작업1 → 2+3+2 = 7

assignment_problem()
```

---

## 구현 예제 2 — C++: Potential Trick(Johnson's)을 활용한 Dijkstra 기반 MCMF

Dijkstra 기반은 SPFA보다 최악의 경우 훨씬 빠르다(O(F · (E log V)) vs O(F · VE)).

```cpp
#include <bits/stdc++.h>
using namespace std;

const int INF = 1e9;
struct Edge { int to, cap, cost, flow; };

struct MCMF {
    int n;
    vector<Edge> edges;
    vector<vector<int>> g;
    vector<int> d, p, a;
    vector<bool> inq;

    MCMF(int n) : n(n), g(n), d(n), p(n), a(n), inq(n) {}

    void addEdge(int from, int to, int cap, int cost) {
        g[from].push_back(edges.size());
        edges.push_back({to, cap, cost, 0});
        g[to].push_back(edges.size());
        edges.push_back({from, 0, -cost, 0});
    }

    // SPFA로 최단 거리(최소 비용 경로) 탐색
    bool spfa(int s, int t) {
        fill(d.begin(), d.end(), INF);
        fill(inq.begin(), inq.end(), false);
        d[s] = 0; a[s] = INF;
        deque<int> q;
        q.push_back(s);
        inq[s] = true;

        while (!q.empty()) {
            int u = q.front(); q.pop_front();
            inq[u] = false;
            for (int id : g[u]) {
                auto& e = edges[id];
                if (e.cap > e.flow && d[u] + e.cost < d[e.to]) {
                    d[e.to] = d[u] + e.cost;
                    p[e.to] = id;
                    a[e.to] = min(a[u], e.cap - e.flow);
                    if (!inq[e.to]) {
                        inq[e.to] = true;
                        // SLF 최적화: deque 앞쪽에 더 작은 거리 삽입
                        if (!q.empty() && d[e.to] < d[q.front()])
                            q.push_front(e.to);
                        else
                            q.push_back(e.to);
                    }
                }
            }
        }
        return d[t] != INF;
    }

    pair<int,int> minCostFlow(int s, int t) {
        int flow = 0, cost = 0;
        while (spfa(s, t)) {
            flow += a[t];
            cost += a[t] * d[t];
            int u = t;
            while (u != s) {
                edges[p[u]].flow += a[t];
                edges[p[u] ^ 1].flow -= a[t];
                u = edges[p[u] ^ 1].to;
            }
        }
        return {flow, cost};
    }
};

int main() {
    // 예시: 4x4 할당 문제
    // 노드 0=소스, 1~4=작업자, 5~8=작업, 9=싱크
    MCMF mcmf(10);
    int S = 0, T = 9;

    int cost[4][4] = {
        {9, 2, 7, 8},
        {6, 4, 3, 7},
        {5, 8, 1, 8},
        {7, 6, 9, 4}
    };

    for (int i = 0; i < 4; i++) mcmf.addEdge(S, i+1, 1, 0);
    for (int i = 0; i < 4; i++)
        for (int j = 0; j < 4; j++)
            mcmf.addEdge(i+1, j+5, 1, cost[i][j]);
    for (int j = 0; j < 4; j++) mcmf.addEdge(j+5, T, 1, 0);

    auto [flow, total_cost] = mcmf.minCostFlow(S, T);
    cout << "최대 매칭: " << flow
         << ", 최소 비용: " << total_cost << "\n";
    // 최적: 0+1→4, 2→3, 3→4 처럼 최소 합
    return 0;
}
```

---

## 알고리즘 정확성 증명 스케치

SSP의 정확성은 **최소 비용 순환 보조 정리(min-cost circulation lemma)**에 기반한다. 핵심은 다음과 같다.

> 잔여 그래프에 음수 비용 순환(negative-cost cycle)이 없으면 현재 유량이 최소 비용이다.

SSP는 매 반복마다 음수 비용 순환을 만들지 않는 경로(최소 비용 경로)를 선택해 유량을 증가시키므로, 종료 시 음수 비용 순환이 없고 따라서 최소 비용 최대 유량이 된다.

---

## 복잡도 분석

| 알고리즘 | 시간 복잡도 | 비고 |
|---------|------------|------|
| SPFA 기반 SSP | O(F · V · E) | F = 최대 유량값, 음수 간선 처리 가능 |
| SPFA+SLF 최적화 | 실무에서 훨씬 빠름 | 최악 동일 |
| Dijkstra+Potential | O(F · E log V) | 첫 번째 Bellman-Ford 후 이후 Dijkstra |
| MCMF(일반) 하한 | Ω(VE log V · log(VW)) | 최신 최고 복잡도 |

여기서 F는 최대 유량값이며, 유량값이 클수록 느려지는 구조임에 주의하라. 비용이 정수이고 작은 경우(경쟁 프로그래밍 환경) SPFA+SLF는 충분히 빠르다.

---

## 주의사항 및 팁

1. **음수 비용 초기 간선 처리**: 초기 그래프에 음수 비용 간선이 있으면 Bellman-Ford로 초기 포텐셜을 한 번 계산한 후 Dijkstra를 써야 한다. 구현이 복잡해지므로 경쟁 프로그래밍에서는 보통 SPFA만 사용한다.

2. **유량 상한 주의**: 유량값 F가 매우 크면 SSP 반복 횟수도 많아진다. 이런 경우 scaling 기반 알고리즘(Cost Scaling, Hungarian)을 고려하라.

3. **정수형 오버플로우**: 비용과 유량을 곱해 총 비용을 계산할 때 `int` 범위를 초과할 수 있다. `long long`을 사용하라.

4. **간선 인덱스 XOR 트릭**: C++ 구현에서 간선을 짝수/홀수 인덱스로 저장하면 역방향 간선을 `id ^ 1`로 O(1)에 접근할 수 있다.

5. **실수 비용**: SPFA 기반은 부동소수점 비용도 처리할 수 있지만 정밀도 문제가 생긴다. 가능하면 정수로 변환하라.

---

## 참고 자료
- [Min-cost flow - CP-Algorithms](https://cp-algorithms.com/graph/min_cost_flow.html)
- [Successive Shortest Paths - TU Darmstadt Algorithm Wiki](https://wiki.algo.informatik.tu-darmstadt.de/Successive_shortest_paths)
- [Min Cost Flow - MIT 6.854 Lecture Notes](https://courses.csail.mit.edu/6.854/21/Scribe/s10-minCostFlowAlg/s10-minCostFlowAlg.html)
- [Successive Shortest Path for Min Cost Max Flow - Boost Graph Library](https://519.graph.prtest3.cppalliance.org/graph/algorithms/network_flow/successive_shortest_path_nonnegative_weights.html)
