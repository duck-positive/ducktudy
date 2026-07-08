---
layout: post
title: "최소 신장 트리 완전 정복: Kruskal과 Prim 알고리즘으로 최소 비용 연결 네트워크 설계하기"
date: 2026-07-08
categories: [cs, computer-science]
tags: [minimum-spanning-tree, kruskal, prim, graph, greedy, union-find, heap, network-design]
---

## 개념 설명

**최소 신장 트리(Minimum Spanning Tree, MST)**는 연결된 가중치 비방향 그래프에서 모든 정점을 연결하면서 간선 가중치의 합이 최소가 되는 트리입니다. N개의 정점을 연결하는 MST는 정확히 N-1개의 간선을 가지며 사이클이 없습니다.

MST는 두 가지 핵심 특성을 만족합니다.

- **컷 특성(Cut Property)**: 그래프를 두 집합으로 나누는 모든 컷에서, 컷을 가로지르는 가장 가벼운 간선은 반드시 MST에 포함됩니다.
- **사이클 특성(Cycle Property)**: 그래프의 임의 사이클에서 가장 무거운 간선은 MST에 포함되지 않습니다.

이 두 특성이 그리디(greedy) 알고리즘의 정당성을 보장합니다.

### Kruskal vs. Prim 비교

| 특성 | Kruskal | Prim |
|------|---------|------|
| 전략 | 간선 기준 | 정점 기준 |
| 처리 순서 | 전체 간선 정렬 후 선택 | 현재 트리 인접 간선 중 최소 선택 |
| 자료구조 | Union-Find | 우선순위 큐(힙) |
| 시간복잡도 | O(E log E) | O(E log V) |
| 유리한 상황 | 희소 그래프 (E ≈ V) | 밀집 그래프 (E ≈ V²) |
| 병렬화 | 용이 | 어려움 |

---

## 왜 필요한가

MST는 "최소 비용으로 모든 지점을 연결하라"는 문제의 핵심 알고리즘입니다.

**실제 응용 분야:**

1. **네트워크 설계**: 도시 간 광케이블, 전력망, 수도관 설치 시 최소 비용 경로 계산
2. **클러스터링**: 머신러닝에서 MST 기반 군집화 — 가장 긴 간선을 끊어 k개 클러스터 생성
3. **근사 알고리즘**: 외판원 문제(TSP)의 2-근사 알고리즘이 MST를 기반으로 함
4. **이미지 분할**: 픽셀을 노드로, 색상 차이를 가중치로 한 그래프의 MST로 이미지 영역 분리
5. **컴파일러 최적화**: 레지스터 할당에서 간섭 그래프의 스패닝 트리 활용
6. **경쟁 프로그래밍**: 도로 건설, 통신망 구축 등 수많은 문제의 직접 풀이 또는 서브태스크

---

## 실제 구현 예제

### 예제 1: Kruskal 알고리즘 (Union-Find 기반)

Kruskal은 간선을 가중치 순으로 정렬한 뒤, 사이클을 만들지 않는 간선을 하나씩 추가합니다. 사이클 검사에는 Union-Find(Disjoint Set Union)를 사용합니다.

```python
from typing import List, Tuple

class UnionFind:
    def __init__(self, n: int):
        self.parent = list(range(n))
        self.rank = [0] * n
        self.components = n
    
    def find(self, x: int) -> int:
        # 경로 압축 (Path Compression)
        if self.parent[x] != x:
            self.parent[x] = self.find(self.parent[x])
        return self.parent[x]
    
    def union(self, x: int, y: int) -> bool:
        """두 집합을 합침. 이미 같은 집합이면 False 반환 (사이클 감지)"""
        rx, ry = self.find(x), self.find(y)
        if rx == ry:
            return False  # 같은 컴포넌트 → 사이클 발생
        # 랭크 기반 합치기 (Union by Rank)
        if self.rank[rx] < self.rank[ry]:
            rx, ry = ry, rx
        self.parent[ry] = rx
        if self.rank[rx] == self.rank[ry]:
            self.rank[rx] += 1
        self.components -= 1
        return True

def kruskal(n: int, edges: List[Tuple[int, int, int]]) -> Tuple[int, List[Tuple]]:
    """
    n: 정점 수 (0-indexed)
    edges: (가중치, u, v) 리스트
    반환: (MST 총 가중치, MST 간선 리스트)
    """
    # 1단계: 간선을 가중치 기준 오름차순 정렬 — O(E log E)
    edges_sorted = sorted(edges, key=lambda e: e[0])
    
    uf = UnionFind(n)
    mst_weight = 0
    mst_edges = []
    
    # 2단계: 간선을 순서대로 검토 — O(E·α(N)) ≈ O(E)
    for weight, u, v in edges_sorted:
        if uf.union(u, v):  # 사이클 없으면 MST에 추가
            mst_weight += weight
            mst_edges.append((u, v, weight))
            if len(mst_edges) == n - 1:
                break  # N-1개 간선 모으면 완료
    
    # 연결 그래프라면 len(mst_edges) == n-1 보장
    if len(mst_edges) < n - 1:
        return -1, []  # 연결되지 않은 그래프
    
    return mst_weight, mst_edges

# ============ 테스트 ============
# 정점 5개, 간선 7개인 그래프
n = 5
edges = [
    (1, 0, 1),
    (3, 0, 3),
    (2, 1, 2),
    (4, 1, 3),
    (5, 2, 3),
    (6, 2, 4),
    (7, 3, 4),
]
# 위 그래프의 MST: (0-1):1, (1-2):2, (0-3):3, (2-4):6 → 합 = 12

total, mst = kruskal(n, edges)
print(f"MST 총 가중치: {total}")   # 12
for u, v, w in mst:
    print(f"  간선 {u}-{v}: 가중치 {w}")
```

Kruskal의 실질적 병목은 정렬 O(E log E)이며, Union-Find 연산은 역아커만 함수 α(N) ≈ 상수로 사실상 무시할 수 있습니다.

---

### 예제 2: Prim 알고리즘 (우선순위 큐 기반, C++)

Prim은 하나의 시작 정점에서 MST를 점진적으로 키워나갑니다. 현재 트리에서 인접한 간선 중 가장 가벼운 것을 선택하며, 이를 위해 최소 힙(min-heap)을 사용합니다.

```cpp
#include <bits/stdc++.h>
using namespace std;

// Prim's Algorithm — O(E log V)
// adj[u] = {(가중치, 인접정점 v)} 형태의 인접 리스트
pair<long long, vector<pair<int,int>>> prim(
    int n, vector<vector<pair<int,int>>>& adj
) {
    // min-heap: (가중치, 현재정점, 이전정점)
    priority_queue<tuple<int,int,int>,
                   vector<tuple<int,int,int>>,
                   greater<>> pq;
    
    vector<bool> inMST(n, false);
    long long total = 0;
    vector<pair<int,int>> mst_edges;
    
    // 정점 0에서 시작 (가중치 0, parent=-1)
    pq.push({0, 0, -1});
    
    while (!pq.empty() && (int)mst_edges.size() < n - 1) {
        auto [w, u, parent] = pq.top();
        pq.pop();
        
        if (inMST[u]) continue;  // 이미 MST에 포함된 정점 스킵
        
        inMST[u] = true;
        total += w;
        if (parent != -1) {
            mst_edges.push_back({parent, u});
        }
        
        // 인접 간선을 힙에 추가
        for (auto [next_w, v] : adj[u]) {
            if (!inMST[v]) {
                pq.push({next_w, v, u});
            }
        }
    }
    
    if ((int)mst_edges.size() < n - 1)
        return {-1, {}};  // 연결되지 않은 그래프
    
    return {total, mst_edges};
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    int n, m;
    cin >> n >> m;
    
    vector<vector<pair<int,int>>> adj(n);
    for (int i = 0; i < m; i++) {
        int u, v, w;
        cin >> u >> v >> w;
        u--; v--;  // 0-indexed 변환
        adj[u].push_back({w, v});
        adj[v].push_back({w, u});
    }
    
    auto [total, mst] = prim(n, adj);
    
    if (total == -1) {
        cout << "연결되지 않은 그래프\n";
        return 0;
    }
    
    cout << "MST 총 가중치: " << total << "\n";
    for (auto [u, v] : mst) {
        cout << "  간선 " << u+1 << "-" << v+1 << "\n";
    }
    return 0;
}
```

**Prim vs. Kruskal 선택 기준:**

밀집 그래프(dense graph, E ≈ V²)에서는 Prim의 O(E log V)가 Kruskal의 O(E log E) ≈ O(E log V²) = O(2E log V)보다 실제로 빠릅니다. 반면 희소 그래프(sparse graph, E ≈ V)에서는 정렬이 빠른 Kruskal이 더 단순하고 효율적입니다.

---

### MST 응용: 클러스터링 (Single-Linkage Clustering)

MST의 가장 긴 k-1개 간선을 제거하면 k개의 클러스터로 분리됩니다. 이것이 **Single-Linkage Clustering**으로, 가장 가까운 두 점 간 거리를 기반으로 군집을 형성합니다.

```python
def mst_clustering(points: List[Tuple[float, float]], k: int):
    """
    2D 점들을 MST 기반으로 k개 클러스터로 분리
    points: [(x, y), ...]
    k: 원하는 클러스터 수
    """
    import math
    n = len(points)
    
    # 모든 쌍의 유클리드 거리로 간선 생성
    edges = []
    for i in range(n):
        for j in range(i + 1, n):
            dx = points[i][0] - points[j][0]
            dy = points[i][1] - points[j][1]
            dist = math.sqrt(dx*dx + dy*dy)
            edges.append((dist, i, j))
    
    # Kruskal로 MST 구성
    _, mst_edges = kruskal(n, edges)
    
    # MST 간선 중 가중치 상위 k-1개 제거
    mst_sorted = sorted(mst_edges, key=lambda e: e[2], reverse=True)
    removed = set()
    for u, v, w in mst_sorted[:k-1]:
        removed.add((u, v))
        removed.add((v, u))
    
    # 제거 후 남은 간선으로 연결 컴포넌트 탐색 (BFS)
    adj = [[] for _ in range(n)]
    for u, v, w in mst_edges:
        if (u, v) not in removed:
            adj[u].append(v)
            adj[v].append(u)
    
    cluster = [-1] * n
    cluster_id = 0
    for start in range(n):
        if cluster[start] == -1:
            queue = [start]
            cluster[start] = cluster_id
            for node in queue:
                for neighbor in adj[node]:
                    if cluster[neighbor] == -1:
                        cluster[neighbor] = cluster_id
                        queue.append(neighbor)
            cluster_id += 1
    
    return cluster

# 사용 예
points = [(0,0),(1,0),(0,1),(1,1),   # 클러스터 A
          (5,5),(6,5),(5,6),(6,6)]    # 클러스터 B
labels = mst_clustering(points, k=2)
print(labels)  # [0,0,0,0,1,1,1,1]
```

---

## 주의사항 및 팁

**1. MST의 유일성**

모든 간선 가중치가 서로 다르면 MST는 유일합니다. 동일한 가중치 간선이 있으면 여러 MST가 존재할 수 있지만, 총 가중치는 동일합니다.

**2. Kruskal에서 정렬 안정성**

같은 가중치 간선이 많을 때, 정렬 순서에 따라 다른 MST가 나올 수 있습니다. 결과의 일관성이 필요하다면 동일 가중치 간선에 추가 정렬 기준을 두세요.

**3. Prim에서 지연 삭제(Lazy Deletion)**

위의 Prim 구현은 "lazy deletion" 방식입니다. 힙에서 이미 MST에 포함된 정점을 꺼내도 skip합니다. 정점 수가 매우 많다면 힙 크기가 O(E)까지 커질 수 있으므로, 힙을 decrease-key 지원 구조(Fibonacci Heap)로 교체하면 O(E + V log V)로 줄일 수 있습니다. 하지만 실제로는 Fibonacci Heap의 상수 인자가 커서 실용적이지 않습니다.

**4. 방향 그래프의 MST: 최소 신장 수형도(MDST)**

방향 그래프의 경우 Kruskal/Prim을 직접 쓸 수 없고, **Chu-Liu/Edmonds 알고리즘** (O(EV) 또는 O(E log V))을 사용해야 합니다.

**5. 온라인 MST (동적 간선 추가)**

간선이 동적으로 추가되는 경우, Link-Cut Tree를 사용하면 O(log² N) per update로 MST를 유지할 수 있습니다.

**6. 시간복잡도 요약**

| 알고리즘 | 시간복잡도 | 공간복잡도 | 적합 상황 |
|----------|------------|------------|-----------|
| Kruskal | O(E log E) | O(V) | 희소 그래프, 병렬 처리 |
| Prim (힙) | O(E log V) | O(V + E) | 밀집 그래프, 인접 리스트 |
| Prim (피보나치 힙) | O(E + V log V) | O(V + E) | 이론적 최적, 실용성 낮음 |
| Borůvka | O(E log V) | O(V + E) | 병렬 MST 계산 |

---

## 참고 자료
- [Kruskal's algorithm - Wikipedia](https://en.wikipedia.org/wiki/Kruskal%27s_algorithm)
- [Minimum Spanning Tree - Prim's and Kruskal's algorithm (Medium)](https://sethuram52001.medium.com/minimum-spanning-tree-prims-and-kruskal-s-algorithm-134cb0a32212)
- [Minimum Spanning Tree (Prim's, Kruskal's) - VisuAlgo](https://visualgo.net/en/mst)
- [MST Kruskal - Algorithms for Competitive Programming](https://cp-algorithms.com/graph/mst_kruskal.html)
