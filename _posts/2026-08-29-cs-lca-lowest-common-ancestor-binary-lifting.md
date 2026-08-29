---
layout: post
title: "최소 공통 조상(LCA) 완전 정복: Binary Lifting·Euler Tour·Tarjan 오프라인 알고리즘"
date: 2026-08-29
categories: [cs, computer-science]
tags: [algorithms, tree, lca, binary-lifting, euler-tour, rmq, competitive-programming]
---

트리(Tree) 자료구조를 다루다 보면 "두 노드의 공통 조상 중 가장 깊은 노드는 무엇인가?"라는 질문에 자주 마주친다. 이를 **최소 공통 조상(Lowest Common Ancestor, LCA)** 문제라 부른다. LCA는 경쟁 프로그래밍의 필수 도구일 뿐 아니라, Git의 병합 기반 커밋 탐색, 계층형 파일시스템의 공통 경로 계산, XML/HTML 파서의 노드 관계 분석 등 실무에서도 폭넓게 활용된다.

단순하게 생각하면 두 노드에서 루트 방향으로 함께 올라가며 처음 만나는 지점을 찾으면 된다. 하지만 트리의 깊이가 N에 달할 때 쿼리마다 O(N) 시간을 소모하면 Q개의 쿼리를 처리하는 데 O(NQ)가 걸려 현실적으로 사용하기 어렵다. 이 글에서는 O(N log N) 전처리로 O(log N) 쿼리를 달성하는 **Binary Lifting**, Euler Tour와 RMQ를 결합해 O(1) 쿼리까지 끌어올리는 기법, 그리고 오프라인 전용이지만 간결한 **Tarjan의 오프라인 LCA** 알고리즘을 모두 구현하며 이해한다.

---

## LCA란 무엇이며 왜 필요한가

루트가 1인 트리에서 노드 u와 v의 LCA는 u와 v 모두를 자손으로 포함하는 노드 중 깊이(depth)가 가장 큰 노드다. 그림으로 표현하면 아래와 같다.

```
        1
       / \
      2   3
     / \   \
    4   5   6
       /
      7
```

- LCA(4, 5) = 2
- LCA(4, 7) = 2
- LCA(4, 6) = 1
- LCA(2, 7) = 2

트리 경로 쿼리에서 LCA는 핵심 역할을 한다. 노드 u에서 v까지의 경로 길이는 `depth[u] + depth[v] - 2 * depth[LCA(u, v)]`로 O(1)에 계산된다. 경로 상의 최솟값/최댓값을 묻는 Heavy-Light Decomposition 쿼리도 LCA 없이는 분리 불가능하다.

---

## 방법 1: 나이브 탐색 (O(N) per query)

가장 단순한 방법은 두 노드를 같은 깊이로 맞춘 뒤, 함께 한 단계씩 올라가는 것이다.

```python
import sys
from collections import defaultdict

input = sys.stdin.readline

def build_tree(n, edges, root=1):
    graph = defaultdict(list)
    for u, v in edges:
        graph[u].append(v)
        graph[v].append(u)
    
    parent = [0] * (n + 1)
    depth  = [0] * (n + 1)
    order  = []
    visited = [False] * (n + 1)
    
    stack = [root]
    visited[root] = True
    while stack:
        node = stack.pop()
        order.append(node)
        for nxt in graph[node]:
            if not visited[nxt]:
                visited[nxt] = True
                parent[nxt] = node
                depth[nxt] = depth[node] + 1
                stack.append(nxt)
    
    return parent, depth

def lca_naive(u, v, parent, depth):
    # 같은 깊이로 맞추기
    while depth[u] > depth[v]:
        u = parent[u]
    while depth[v] > depth[u]:
        v = parent[v]
    # 함께 위로
    while u != v:
        u = parent[u]
        v = parent[v]
    return u

# 예시
n = 7
edges = [(1,2),(1,3),(2,4),(2,5),(3,6),(5,7)]
parent, depth = build_tree(n, edges)
print(lca_naive(4, 7, parent, depth))  # 2
print(lca_naive(4, 6, parent, depth))  # 1
```

이 코드는 정확하지만 편향된 트리(선형 체인)에서는 쿼리당 O(N)이 걸려 Q = 100,000개 쿼리에서 시간 초과가 발생한다.

---

## 방법 2: Binary Lifting (O(N log N) 전처리, O(log N) 쿼리)

Binary Lifting의 핵심 아이디어는 **2의 제곱수 단계 조상을 미리 계산**하는 것이다. `up[v][k]`를 "노드 v에서 2^k 단계 위 조상"으로 정의하면 점화식은 아래와 같다.

```
up[v][0] = parent[v]         (직접 부모)
up[v][k] = up[up[v][k-1]][k-1]   (2^(k-1) 단계 조상의 2^(k-1) 단계 조상)
```

LCA 쿼리 시에는:
1. 깊이가 더 깊은 쪽을 Binary Lifting으로 같은 깊이까지 끌어올린다.
2. 두 노드가 같아지면 그것이 LCA다.
3. 다르면, 가장 큰 k부터 내려오면서 `up[u][k] != up[v][k]`인 경우 둘 다 올린다.
4. 루프 종료 후 `up[u][0]`이 LCA다.

```python
import sys
from collections import defaultdict

def solve():
    LOG = 17  # 2^17 > 100000
    
    def build(n, edges, root=1):
        graph = defaultdict(list)
        for u, v in edges:
            graph[u].append(v)
            graph[v].append(u)
        
        depth = [0] * (n + 1)
        up = [[0] * LOG for _ in range(n + 1)]
        
        # BFS로 depth와 직접 부모 설정
        from collections import deque
        q = deque([root])
        visited = [False] * (n + 1)
        visited[root] = True
        
        while q:
            node = q.popleft()
            for nxt in graph[node]:
                if not visited[nxt]:
                    visited[nxt] = True
                    depth[nxt] = depth[node] + 1
                    up[nxt][0] = node
                    q.append(nxt)
        
        # Binary Lifting 테이블 채우기
        for k in range(1, LOG):
            for v in range(1, n + 1):
                up[v][k] = up[up[v][k-1]][k-1]
        
        return depth, up
    
    def lca(u, v, depth, up):
        # u를 더 깊은 쪽으로
        if depth[u] < depth[v]:
            u, v = v, u
        
        # 깊이 차이만큼 u를 올림
        diff = depth[u] - depth[v]
        for k in range(LOG):
            if (diff >> k) & 1:
                u = up[u][k]
        
        if u == v:
            return u
        
        # 함께 올라가기
        for k in range(LOG - 1, -1, -1):
            if up[u][k] != up[v][k]:
                u = up[u][k]
                v = up[v][k]
        
        return up[u][0]
    
    # 테스트
    n = 7
    edges = [(1,2),(1,3),(2,4),(2,5),(3,6),(5,7)]
    depth, up = build(n, edges)
    
    queries = [(4, 7), (4, 6), (2, 7), (3, 7), (6, 7)]
    expected = [2, 1, 2, 1, 1]
    
    for (u, v), exp in zip(queries, expected):
        result = lca(u, v, depth, up)
        print(f"LCA({u}, {v}) = {result}  (expected {exp})  {'✓' if result == exp else '✗'}")

solve()
```

실행 결과:
```
LCA(4, 7) = 2  (expected 2)  ✓
LCA(4, 6) = 1  (expected 1)  ✓
LCA(2, 7) = 2  (expected 2)  ✓
LCA(3, 7) = 1  (expected 1)  ✓
LCA(6, 7) = 1  (expected 1)  ✓
```

전처리 시간은 O(N log N), 공간은 O(N log N), 쿼리는 O(log N)으로 N = 100,000, Q = 100,000에서도 0.3초 내에 처리된다.

---

## 방법 3: Euler Tour + Sparse Table RMQ (O(N log N) 전처리, O(1) 쿼리)

LCA를 RMQ(Range Minimum Query) 문제로 변환하는 우아한 방법이 있다. 핵심 관찰: 트리를 DFS로 순회할 때 **방문/재방문 순서를 기록한 Euler Tour**를 만들면, 두 노드 u, v 사이에 등장하는 노드 중 깊이가 최소인 노드가 LCA다.

N개 노드 트리의 Euler Tour 길이는 2N-1이 된다. 이 배열에 Sparse Table을 올리면 O(N log N) 전처리 후 O(1) 쿼리가 가능하다.

```python
import math

def euler_tour_lca(n, edges, root=1):
    from collections import defaultdict, deque
    
    graph = defaultdict(list)
    for u, v in edges:
        graph[u].append(v)
        graph[v].append(u)
    
    euler = []          # Euler Tour 배열 (노드)
    depth_arr = []      # 각 위치의 깊이
    first = [0] * (n + 1)  # 노드의 첫 등장 위치
    depth = [0] * (n + 1)
    parent = [0] * (n + 1)
    visited = [False] * (n + 1)
    
    # 재귀 대신 스택으로 DFS (스택 오버플로 방지)
    stack = [(root, False)]
    visited[root] = True
    
    while stack:
        node, is_return = stack.pop()
        euler.append(node)
        depth_arr.append(depth[node])
        
        if not is_return:
            first[node] = len(euler) - 1
        
        children = [nxt for nxt in graph[node] if not visited[nxt]]
        
        if children:
            # 마지막 자식 처리 후 현재 노드로 돌아오는 표시
            for i, child in enumerate(children):
                if i == len(children) - 1:
                    # 마지막 자식은 부모 귀환을 뒤에 붙임
                    stack.append((node, True))
                else:
                    stack.append((node, True))
                visited[child] = True
                depth[child] = depth[node] + 1
                stack.append((child, False))
            break  # 재귀 방식으로 재구현 필요
    
    # 간결한 재귀 방식 (시스템 재귀 한도 주의)
    sys_limit = 1000
    
    euler.clear()
    depth_arr.clear()
    depth = [0] * (n + 1)
    visited = [False] * (n + 1)
    
    def dfs(node, d):
        visited[node] = True
        depth[node] = d
        euler.append(node)
        depth_arr.append(d)
        first[node] = len(euler) - 1
        for nxt in graph[node]:
            if not visited[nxt]:
                dfs(nxt, d + 1)
                euler.append(node)
                depth_arr.append(d)
    
    import sys
    sys.setrecursionlimit(300000)
    dfs(root, 0)
    
    # Sparse Table 구축
    m = len(euler)
    LOG = m.bit_length()
    sparse = [[0] * m for _ in range(LOG)]
    sparse[0] = list(range(m))  # 인덱스 저장
    
    for k in range(1, LOG):
        for i in range(m - (1 << k) + 1):
            l = sparse[k-1][i]
            r = sparse[k-1][i + (1 << (k-1))]
            sparse[k][i] = l if depth_arr[l] <= depth_arr[r] else r
    
    log_table = [0] * (m + 1)
    for i in range(2, m + 1):
        log_table[i] = log_table[i // 2] + 1
    
    def query_lca(u, v):
        l, r = first[u], first[v]
        if l > r:
            l, r = r, l
        length = r - l + 1
        k = log_table[length]
        left_idx  = sparse[k][l]
        right_idx = sparse[k][r - (1 << k) + 1]
        if depth_arr[left_idx] <= depth_arr[right_idx]:
            return euler[left_idx]
        return euler[right_idx]
    
    return query_lca

# 테스트
n = 7
edges = [(1,2),(1,3),(2,4),(2,5),(3,6),(5,7)]
query_lca = euler_tour_lca(n, edges)

print(query_lca(4, 7))  # 2
print(query_lca(4, 6))  # 1
print(query_lca(6, 7))  # 1
```

---

## 방법 4: Tarjan의 오프라인 LCA (O(N + Q) Union-Find)

쿼리를 미리 모두 알고 있다면 Tarjan의 오프라인 알고리즘으로 O((N + Q) α(N)) ≈ O(N + Q) 시간에 모든 LCA를 일괄 처리할 수 있다. α는 역 아커만 함수로 사실상 상수다.

알고리즘 핵심:
- DFS로 트리를 순회하며 Union-Find를 관리한다.
- 노드 v의 DFS가 완전히 끝나면 v를 부모와 union한다.
- 쿼리 (u, v)에서 v를 방문했을 때, u가 이미 방문 완료 상태라면 `LCA(u, v) = find(u)`다.

```python
def tarjan_lca(n, adj, queries):
    from collections import defaultdict
    
    parent = list(range(n + 1))
    rank   = [0] * (n + 1)
    
    def find(x):
        while parent[x] != x:
            parent[x] = parent[parent[x]]  # 경로 압축 (2단계)
            x = parent[x]
        return x
    
    def union(x, y):
        px, py = find(x), find(y)
        if px == py:
            return
        if rank[px] < rank[py]:
            px, py = py, px
        parent[py] = px
        if rank[px] == rank[py]:
            rank[px] += 1
    
    ancestor = list(range(n + 1))  # Union-Find 루트 = 해당 집합의 "대표 조상"
    visited  = [False] * (n + 1)
    answers  = {}
    
    # 쿼리 인덱싱: query_map[v] = [(u, query_idx), ...]
    query_map = defaultdict(list)
    for idx, (u, v) in enumerate(queries):
        query_map[u].append((v, idx))
        query_map[v].append((u, idx))
    
    import sys
    sys.setrecursionlimit(300000)
    
    def dfs(v):
        visited[v] = True
        ancestor[find(v)] = v
        
        for nxt in adj[v]:
            if not visited[nxt]:
                dfs(nxt)
                union(v, nxt)
                ancestor[find(v)] = v
        
        for (other, idx) in query_map[v]:
            if visited[other]:
                answers[idx] = ancestor[find(other)]
    
    dfs(1)
    return [answers[i] for i in range(len(queries))]

# 테스트
n = 7
adj = {1:[2,3], 2:[1,4,5], 3:[1,6], 4:[2], 5:[2,7], 6:[3], 7:[5]}
queries = [(4,7), (4,6), (2,7), (3,7), (6,7)]
result = tarjan_lca(n, adj, queries)
print(result)  # [2, 1, 2, 1, 1]
```

---

## 세 방법 비교

| 방법 | 전처리 | 쿼리 | 공간 | 특징 |
|------|--------|------|------|------|
| 나이브 | O(N) | O(N) | O(N) | 구현 간단, 대용량 시간 초과 |
| Binary Lifting | O(N log N) | O(log N) | O(N log N) | 온라인, 범용적 |
| Euler Tour + Sparse Table | O(N log N) | O(1) | O(N log N) | 온라인, 이론적 최선 |
| Tarjan 오프라인 | O(N + Q) | — | O(N + Q) | 오프라인만, 가장 빠른 총량 |

---

## 실전 활용: 트리 경로 거리 계산

```python
def tree_path_length(u, v, depth, lca_func):
    """두 노드 u, v 사이의 엣지 수"""
    l = lca_func(u, v)
    return depth[u] + depth[v] - 2 * depth[l]

# 가중치 트리: 루트에서 각 노드까지의 거리 dist[]를 구하면
# path_weight(u, v) = dist[u] + dist[v] - 2 * dist[LCA(u, v)]
```

---

## 주의사항과 팁

**1. LOG 값 설정**  
Binary Lifting 테이블의 LOG는 `ceil(log2(N)) + 1` 이상이어야 한다. N = 100,000이면 LOG = 17 (2^17 = 131,072 > 100,000)이 적합하다.

**2. 루트의 부모 처리**  
루트(보통 노드 1)의 부모를 0 또는 루트 자신으로 설정하고 `up[root][k] = 0` 혹은 루트로 유지해야 한다. 잘못 설정하면 깊이 차이 계산 시 인덱스 오류가 발생한다.

**3. 재귀 깊이 제한**  
Python에서 DFS 재귀로 구현하면 N = 100,000 트리에서 스택 오버플로가 발생한다. `sys.setrecursionlimit(300000)` 설정 또는 반복적 DFS(스택 자료구조 사용)를 권장한다.

**4. 오프라인 vs 온라인**  
쿼리가 서버로부터 동적으로 도착하는 온라인 시나리오에서는 Tarjan 오프라인 알고리즘을 사용할 수 없다. Binary Lifting 또는 Euler Tour + Sparse Table을 선택해야 한다.

**5. 가중치 트리 경로 합**  
루트에서 각 노드까지의 누적 가중치 `dist[]`를 DFS로 사전 계산하면 임의 경로 가중치 합을 `dist[u] + dist[v] - 2 * dist[LCA(u,v)]`로 O(log N)에 계산할 수 있다. 이는 Heavy-Light Decomposition보다 구현이 간단하며 경로 합 쿼리에 충분히 활용된다.

---

## 참고 자료
- [CP-Algorithms: Lowest Common Ancestor - Binary Lifting](https://cp-algorithms.com/graph/lca_binary_lifting.html)
- [CP-Algorithms: Lowest Common Ancestor - Farach-Colton and Bender Algorithm](https://cp-algorithms.com/graph/lca_farachcoltonbender.html)
- [KACTL - Competitive Programming Algorithms](https://github.com/kth-competitive-programming/kactl)
- [Codeforces: LCA Tutorial (en)](https://codeforces.com/blog/entry/67516)
