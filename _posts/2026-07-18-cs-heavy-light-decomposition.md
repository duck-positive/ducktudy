---
layout: post
title: "Heavy-Light Decomposition: 트리 경로 쿼리를 O(log²N)으로 해결하기"
date: 2026-07-18
categories: [cs, computer-science]
tags: [heavy-light-decomposition, hld, tree, segment-tree, competitive-programming, data-structure, algorithm]
---

트리(Tree) 자료구조에서 임의의 두 노드를 잇는 경로에 대해 구간 합, 최솟값, 최댓값 등을 빠르게 구해야 하는 문제가 있다. 단순 DFS로는 매 쿼리마다 O(N)이 걸리지만, **Heavy-Light Decomposition(HLD, 무거운 경로 분해)**을 사용하면 O(log²N)으로 줄일 수 있다. 이 글에서는 HLD의 핵심 아이디어부터 구현, 실전 활용까지 단계적으로 살펴본다.

---

## 개념 설명

### 트리 경로 쿼리 문제

N개의 노드로 구성된 트리가 있고, 각 노드에 가중치가 부여되어 있다. 다음 두 종류의 연산을 빠르게 처리하고 싶다.

- **업데이트**: 특정 노드의 가중치를 변경
- **쿼리**: 두 노드 u, v를 잇는 경로 위의 모든 노드 값에 대해 합/최솟값/최댓값 계산

### HLD의 핵심 아이디어

HLD는 트리의 모든 간선을 **Heavy Edge**와 **Light Edge**로 분류한다.

- **Heavy Child**: 각 노드의 자식 중 서브트리 크기가 가장 큰 자식
- **Heavy Edge**: 노드와 Heavy Child를 잇는 간선
- **Light Edge**: 나머지 간선

이렇게 분류하면 루트에서 임의의 리프 노드까지 가는 경로에서 **Light Edge는 최대 O(log N)번만 등장**한다는 핵심 성질이 성립한다. 이는 Light Edge를 통해 서브트리로 내려갈 때마다 서브트리 크기가 최소 2배 이상 줄어들기 때문이다.

Heavy Edge로 이루어진 연속된 체인(Chain)들을 하나의 구간으로 표현하고, 세그먼트 트리 등의 자료구조로 관리하면 경로 쿼리를 O(log N)개의 구간 쿼리로 분해할 수 있다.

---

## 왜 필요한가

트리 경로 쿼리는 다양한 실전 문제에서 등장한다.

- 네트워크 라우팅에서 두 호스트 간 경로의 최대 병목 대역폭 계산
- 게임 트리에서 두 노드 간 경로의 가중치 합/최댓값 업데이트
- 컴파일러의 지배자 트리(Dominator Tree)에서 구간 질의
- 조상-후손 관계 쿼리, LCA(최소 공통 조상) 응용

순수 DFS(O(N)/쿼리), 오일러 투어 + 세그먼트 트리(LCA까지만 지원) 등 다른 방법보다 HLD는 **경로 업데이트와 쿼리를 모두 효율적으로** 처리한다.

---

## 실제 구현 예제

### 예제 1: HLD 기본 구현 (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

const int MAXN = 100005;
vector<int> adj[MAXN];
int parent[MAXN], depth[MAXN], subtree_size[MAXN];
int heavy[MAXN];      // heavy child
int head[MAXN];       // 체인의 최상단 노드
int pos[MAXN];        // DFS 순서 기반 배열 인덱스
int cur_pos;

// 1단계: DFS로 서브트리 크기 계산 및 heavy child 지정
int dfs(int v, int p, int d) {
    parent[v] = p;
    depth[v] = d;
    subtree_size[v] = 1;
    int max_size = 0;
    heavy[v] = -1;
    for (int u : adj[v]) {
        if (u == p) continue;
        subtree_size[v] += dfs(u, v, d + 1);
        if (subtree_size[u] > max_size) {
            max_size = subtree_size[u];
            heavy[v] = u;  // 가장 큰 서브트리의 자식이 heavy child
        }
    }
    return subtree_size[v];
}

// 2단계: 체인 분해 및 DFS 순서 배열 위치 할당
void decompose(int v, int h) {
    head[v] = h;
    pos[v] = cur_pos++;
    if (heavy[v] != -1) {
        // heavy child는 같은 체인 유지
        decompose(heavy[v], h);
    }
    for (int u : adj[v]) {
        if (u == parent[v] || u == heavy[v]) continue;
        // light child는 새 체인 시작
        decompose(u, u);
    }
}

// 세그먼트 트리 (구간 합)
long long seg[4 * MAXN];
int arr[MAXN];

void build(int node, int l, int r) {
    if (l == r) { seg[node] = arr[l]; return; }
    int mid = (l + r) / 2;
    build(2*node, l, mid);
    build(2*node+1, mid+1, r);
    seg[node] = seg[2*node] + seg[2*node+1];
}

void update(int node, int l, int r, int idx, int val) {
    if (l == r) { seg[node] = val; return; }
    int mid = (l + r) / 2;
    if (idx <= mid) update(2*node, l, mid, idx, val);
    else update(2*node+1, mid+1, r, idx, val);
    seg[node] = seg[2*node] + seg[2*node+1];
}

long long query(int node, int l, int r, int ql, int qr) {
    if (qr < l || r < ql) return 0;
    if (ql <= l && r <= qr) return seg[node];
    int mid = (l + r) / 2;
    return query(2*node, l, mid, ql, qr)
         + query(2*node+1, mid+1, r, ql, qr);
}

// 3단계: 경로 쿼리 (u ~ v 경로 합)
long long path_query(int u, int v, int n) {
    long long result = 0;
    while (head[u] != head[v]) {
        // 더 깊은 쪽의 체인 머리까지 쿼리
        if (depth[head[u]] < depth[head[v]]) swap(u, v);
        result += query(1, 0, n-1, pos[head[u]], pos[u]);
        u = parent[head[u]];  // 체인 머리의 부모로 이동
    }
    // 같은 체인 안에 있을 때
    if (depth[u] > depth[v]) swap(u, v);
    result += query(1, 0, n-1, pos[u], pos[v]);
    return result;
}

int main() {
    int n, q;
    cin >> n >> q;
    
    for (int i = 0; i < n - 1; i++) {
        int u, v; cin >> u >> v;
        adj[u].push_back(v);
        adj[v].push_back(u);
    }
    
    for (int i = 1; i <= n; i++) cin >> arr[i];
    
    cur_pos = 0;
    dfs(1, 0, 0);
    decompose(1, 1);
    
    // DFS 순서로 배열 재배치
    int new_arr[MAXN];
    for (int i = 1; i <= n; i++) new_arr[pos[i]] = arr[i];
    build(1, 0, n-1);
    
    while (q--) {
        int type; cin >> type;
        if (type == 1) {
            int v, val; cin >> v >> val;
            update(1, 0, n-1, pos[v], val);
        } else {
            int u, v; cin >> u >> v;
            cout << path_query(u, v, n) << "\n";
        }
    }
    return 0;
}
```

### 예제 2: 경로 최솟값 쿼리 (Python 간략 버전)

Python에서는 대규모 트리보다 개념 검증 수준으로 활용할 수 있다.

```python
import sys
from collections import defaultdict
sys.setrecursionlimit(300000)

class HLD:
    def __init__(self, n):
        self.n = n
        self.adj = defaultdict(list)
        self.parent = [0] * (n + 1)
        self.depth = [0] * (n + 1)
        self.size = [1] * (n + 1)
        self.heavy = [-1] * (n + 1)
        self.head = [0] * (n + 1)
        self.pos = [0] * (n + 1)
        self.pos_node = [0] * (n + 1)
        self.cur = 0
        # 세그먼트 트리 (최솟값)
        self.seg = [float('inf')] * (4 * n + 4)
    
    def add_edge(self, u, v):
        self.adj[u].append(v)
        self.adj[v].append(u)
    
    def _dfs_size(self, v, p, d):
        self.parent[v] = p
        self.depth[v] = d
        self.size[v] = 1
        max_size, heavy_child = 0, -1
        for u in self.adj[v]:
            if u == p: continue
            self._dfs_size(u, v, d + 1)
            self.size[v] += self.size[u]
            if self.size[u] > max_size:
                max_size = self.size[u]
                heavy_child = u
        self.heavy[v] = heavy_child
    
    def _decompose(self, v, h):
        self.head[v] = h
        self.pos[v] = self.cur
        self.pos_node[self.cur] = v
        self.cur += 1
        if self.heavy[v] != -1:
            self._decompose(self.heavy[v], h)
        for u in self.adj[v]:
            if u == self.parent[v] or u == self.heavy[v]: continue
            self._decompose(u, u)
    
    def build(self, values):
        self._dfs_size(1, 0, 0)
        self._decompose(1, 1)
        for v in range(1, self.n + 1):
            self._seg_update(1, 0, self.n - 1, self.pos[v], values[v])
    
    def _seg_update(self, node, l, r, idx, val):
        if l == r:
            self.seg[node] = val; return
        mid = (l + r) // 2
        if idx <= mid: self._seg_update(2*node, l, mid, idx, val)
        else: self._seg_update(2*node+1, mid+1, r, idx, val)
        self.seg[node] = min(self.seg[2*node], self.seg[2*node+1])
    
    def _seg_query(self, node, l, r, ql, qr):
        if qr < l or r < ql: return float('inf')
        if ql <= l <= r <= qr: return self.seg[node]
        mid = (l + r) // 2
        return min(self._seg_query(2*node, l, mid, ql, qr),
                   self._seg_query(2*node+1, mid+1, r, ql, qr))
    
    def path_min(self, u, v):
        result = float('inf')
        while self.head[u] != self.head[v]:
            if self.depth[self.head[u]] < self.depth[self.head[v]]:
                u, v = v, u
            result = min(result, self._seg_query(
                1, 0, self.n-1, self.pos[self.head[u]], self.pos[u]))
            u = self.parent[self.head[u]]
        if self.depth[u] > self.depth[v]:
            u, v = v, u
        result = min(result, self._seg_query(
            1, 0, self.n-1, self.pos[u], self.pos[v]))
        return result

# 사용 예
hld = HLD(7)
edges = [(1,2),(1,3),(2,4),(2,5),(3,6),(3,7)]
for u, v in edges: hld.add_edge(u, v)
values = [0, 10, 5, 8, 3, 7, 2, 9]  # 1-indexed
hld.build(values)
print(hld.path_min(4, 6))  # 노드 4 ~ 6 경로의 최솟값
```

---

## 시간 복잡도 분석

| 연산 | 시간 복잡도 |
|------|------------|
| 전처리 (DFS + 분해) | O(N) |
| 단일 노드 업데이트 | O(log N) |
| 경로 쿼리 | O(log²N) |
| 경로 업데이트 | O(log²N) |

경로 쿼리가 O(log²N)인 이유는 Light Edge를 최대 O(log N)번 만나고, 각 체인 내 세그먼트 트리 쿼리가 O(log N)이기 때문이다.

---

## 주의사항 및 팁

### 엣지 가중치 vs 노드 가중치

HLD의 기본 구현은 **노드 가중치**를 다룬다. 엣지 가중치가 주어지는 문제라면, 각 엣지의 가중치를 자식 노드에 부여하는 방식으로 변환한다(루트 노드는 가중치 0).

### 체인 머리(head) 깊이 비교

`path_query` 함수에서 두 노드의 체인 머리 중 더 깊은 쪽부터 올라가야 한다. 이 방향을 틀리면 무한 루프에 빠진다.

### LCA와의 관계

HLD를 구현하면 LCA(Lowest Common Ancestor)를 O(log N)에 부수적으로 구할 수 있다. `path_query`의 종료 조건(같은 체인에 올라온 순간의 상위 노드)이 곧 LCA다.

### 재귀 깊이 제한

Python에서 DFS 기반 구현은 N이 클 때 재귀 한도를 초과한다. `sys.setrecursionlimit`를 충분히 크게 설정하거나, 반복적(iterative) DFS로 전환하는 것이 안전하다.

### 세그먼트 트리 대안

구간 합만 필요하다면 **Fenwick Tree(BIT)**로도 구현 가능하다. 코드가 짧아지지만 구간 최솟값/최댓값은 지원하지 않는다.

---

## 참고 자료
- [Heavy-light decomposition - CP-Algorithms](https://cp-algorithms.com/graph/hld.html)
- [Heavy-Light Decomposition - USACO Guide](https://usaco.guide/plat/hld)
- [Introduction to Heavy Light Decomposition - GeeksforGeeks](https://www.geeksforgeeks.org/dsa/introduction-to-heavy-light-decomposition/)
- [cp-algorithms/hld.md - GitHub](https://raw.githubusercontent.com/cp-algorithms/cp-algorithms/main/src/graph/hld.md)
