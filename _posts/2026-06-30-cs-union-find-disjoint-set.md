---
layout: post
title: "Union-Find(Disjoint Set Union) 완전 정복: 경로 압축과 크루스칼 알고리즘의 핵심"
date: 2026-06-30
categories: [cs, computer-science]
tags: [algorithms, data-structure, union-find, disjoint-set, graph, kruskal, spanning-tree]
---

## 개념 설명

**Union-Find** (또는 Disjoint Set Union, DSU)는 여러 개의 원소를 서로소(Disjoint)인 집합들로 분할하고, 다음 두 연산을 효율적으로 지원하는 자료구조다:

- **Find(x)**: 원소 x가 속한 집합의 대표 원소(루트)를 반환한다.
- **Union(x, y)**: 원소 x와 y가 속한 두 집합을 하나로 합친다.

직관적으로는 각 집합을 **역트리(inverted tree, 루트로 올라가는 방향)**로 표현한다. 각 노드는 부모를 가리키고, 루트 노드는 자기 자신을 부모로 가진다.

```
초기 상태 (모든 노드가 독립 집합):
0   1   2   3   4
↑   ↑   ↑   ↑   ↑

Union(0,1), Union(2,3), Union(1,2) 이후:
    0           4
   / \
  1   2
       \
        3
루트: 0이 {0,1,2,3}의 대표, 4는 {4}의 대표
```

---

## 왜 필요한가

### 핵심 문제: 연결성 판별

다음 유형의 문제에서 Union-Find는 필수적이다:

- **그래프 사이클 탐지**: 간선을 추가할 때 두 정점이 이미 같은 집합이면 사이클이 생긴다.
- **최소 신장 트리(MST)**: 크루스칼(Kruskal) 알고리즘의 핵심 구성 요소.
- **네트워크 연결성**: n개의 컴퓨터가 연결되는 시점 판단.
- **이미지 레이블링**: 픽셀 군집 탐지.
- **계정 병합 문제**: 동일 사용자를 여러 이메일로 식별.

### 최적화 없이는 느리다

순진한 구현에서 `find`는 트리 높이에 비례하여 O(n) 시간이 걸릴 수 있다. 1,000만 개의 Union 연산을 처리하면 사실상 O(n²)이 된다. 두 가지 최적화가 이를 해결한다.

---

## 두 가지 핵심 최적화

### 1. 경로 압축(Path Compression)

`find` 연산 중 방문한 모든 노드의 부모를 루트로 직접 연결한다. 다음 번 `find` 호출 시 즉시 O(1)에 루트를 찾을 수 있게 된다.

```
경로 압축 전:                 경로 압축 후 (find(3) 호출):
0 ← 1 ← 2 ← 3              0 ← 1
                              0 ← 2
(find(3): 0→1→2→3 탐색)       0 ← 3
```

### 2. 랭크/크기 기반 합치기(Union by Rank/Size)

두 트리를 합칠 때, 항상 **랭크(높이 추정치)가 낮은 트리를 높은 트리 아래로** 붙인다. 이렇게 하면 트리 높이가 O(log n)을 넘지 않는다.

두 최적화를 함께 적용하면 `find`와 `union`의 시간 복잡도가 사실상 O(α(n))이 된다. α(n)은 **역 아커만 함수(Inverse Ackermann Function)**로, 실용적인 범위(n ≤ 10^80)에서 항상 4 이하이므로 사실상 상수 시간이다.

---

## 실제 구현 예제

### 예제 1: Union-Find 완전 구현 (Python)

```python
class UnionFind:
    def __init__(self, n: int):
        # 초기에는 자기 자신이 부모이자 루트
        self.parent = list(range(n))
        # 랭크: 트리 높이의 상한 추정치
        self.rank = [0] * n
        # 각 집합의 원소 수
        self.size = [1] * n
        # 독립된 집합의 수
        self.num_components = n

    def find(self, x: int) -> int:
        """경로 압축: x의 루트를 찾으면서 경로상 모든 노드를 루트에 직접 연결"""
        if self.parent[x] != x:
            self.parent[x] = self.find(self.parent[x])  # 재귀적 경로 압축
        return self.parent[x]

    def union(self, x: int, y: int) -> bool:
        """랭크 기반 합치기. 이미 같은 집합이면 False 반환 (사이클 감지)"""
        root_x = self.find(x)
        root_y = self.find(y)

        if root_x == root_y:
            return False  # 이미 같은 집합 → 사이클

        # 랭크가 낮은 트리를 높은 트리 아래로 붙임
        if self.rank[root_x] < self.rank[root_y]:
            root_x, root_y = root_y, root_x

        self.parent[root_y] = root_x
        self.size[root_x] += self.size[root_y]

        # 두 랭크가 같으면 합친 쪽 랭크를 1 증가
        if self.rank[root_x] == self.rank[root_y]:
            self.rank[root_x] += 1

        self.num_components -= 1
        return True

    def connected(self, x: int, y: int) -> bool:
        return self.find(x) == self.find(y)

    def component_size(self, x: int) -> int:
        return self.size[self.find(x)]


# --- 사용 예시 ---
uf = UnionFind(6)  # 노드 0~5

print(f"초기 집합 수: {uf.num_components}")  # 6

uf.union(0, 1)
uf.union(2, 3)
uf.union(4, 5)
print(f"3번의 Union 후 집합 수: {uf.num_components}")  # 3

print(f"0과 1은 연결됨: {uf.connected(0, 1)}")  # True
print(f"0과 2는 연결됨: {uf.connected(0, 2)}")  # False

uf.union(1, 3)
print(f"union(1,3) 후 집합 수: {uf.num_components}")  # 2
print(f"0과 3은 연결됨: {uf.connected(0, 3)}")  # True
print(f"{0}의 집합 크기: {uf.component_size(0)}")  # 4

# 사이클 감지: union 반환값이 False면 사이클
is_cycle = not uf.union(0, 2)
is_cycle2 = not uf.union(0, 3)  # 이미 같은 집합
print(f"0-3 간선이 사이클을 형성하나: {is_cycle2}")  # True
```

### 예제 2: 크루스칼(Kruskal) 알고리즘으로 MST 구하기

크루스칼 알고리즘은 간선을 가중치 오름차순으로 정렬하고, 사이클을 만들지 않는 간선을 탐욕적으로 선택하여 MST를 구성한다. Union-Find가 사이클 판별을 O(α(n))에 처리해 전체 복잡도가 O(E log E)가 된다.

```python
from typing import List, Tuple


def kruskal_mst(n: int, edges: List[Tuple[int, int, int]]) -> Tuple[int, List]:
    """
    크루스칼 알고리즘으로 최소 신장 트리(MST)를 구한다.

    Args:
        n: 정점 수 (0 ~ n-1)
        edges: (가중치, u, v) 형식의 간선 리스트

    Returns:
        (MST 총 가중치, MST를 구성하는 간선 리스트)
    """
    uf = UnionFind(n)
    mst_edges = []
    total_weight = 0

    # 간선을 가중치 기준 오름차순 정렬 — O(E log E)
    sorted_edges = sorted(edges, key=lambda e: e[0])

    for weight, u, v in sorted_edges:
        # union이 True이면 사이클 없이 합쳐진 것 → MST에 포함
        if uf.union(u, v):
            mst_edges.append((u, v, weight))
            total_weight += weight

        # MST에 n-1개의 간선이 모이면 완성
        if len(mst_edges) == n - 1:
            break

    return total_weight, mst_edges


# 예시 그래프
#    2     3
# 0----1----2
# |    |    |
# 6    8    5
# |    |    |
# 3----4----5
#    7     9
edges = [
    (2, 0, 1), (3, 1, 2), (6, 0, 3), (8, 1, 4), (5, 2, 5),
    (7, 3, 4), (9, 4, 5), (4, 1, 3), (1, 3, 4),  # (가중치, u, v)
]

total, mst = kruskal_mst(6, edges)
print(f"MST 총 가중치: {total}")
print("MST 간선:")
for u, v, w in mst:
    print(f"  {u} -- {v}: {w}")

# 연결 컴포넌트 분석: 모든 정점이 하나의 집합인지 확인
uf_check = UnionFind(6)
for _, u, v in mst:
    uf_check.union(u, v)
print(f"MST로 모든 정점이 연결됨: {uf_check.num_components == 1}")
```

---

## 주의사항 및 팁

### 1. 재귀 깊이 한계

Python의 기본 재귀 깊이는 1,000이다. 경로 압축을 재귀로 구현하면 큰 입력에서 `RecursionError`가 발생할 수 있다. 반복문(iterative) 방식으로 바꾸거나 `sys.setrecursionlimit`을 활용하자.

```python
def find_iterative(self, x: int) -> int:
    """반복문을 사용한 경로 압축 — 재귀 깊이 제한 없음"""
    root = x
    # 루트 탐색
    while self.parent[root] != root:
        root = self.parent[root]
    # 경로상 모든 노드를 루트로 직접 연결
    while self.parent[x] != root:
        next_node = self.parent[x]
        self.parent[x] = root
        x = next_node
    return root
```

### 2. 랭크(Rank) vs 크기(Size)

랭크는 트리 높이의 추정치이고, 크기는 집합의 원소 수다. 경로 압축을 적용하면 랭크가 실제 높이와 달라질 수 있지만, 이는 상관없다 — 랭크는 오직 비교를 위한 추정치이기 때문이다. 크기 기반 합치기는 전체 원소 수가 중요할 때 유용하다.

### 3. Rollback Union-Find (오프라인 알고리즘)

간선 제거가 필요한 문제(오프라인 쿼리)에서는 **Union by Rank + 스택**으로 롤백을 구현할 수 있다. 단, 경로 압축과 함께 사용하면 롤백이 어려우므로 경로 압축 없이 Union by Rank만 사용한다. 이 경우 `find`는 O(log n)이 된다.

### 4. 간선 수가 정점 수보다 훨씬 많다면 (Dense Graph)

MST의 경우 밀집 그래프(Dense Graph)에서는 프림(Prim) 알고리즘 + 우선순위 큐가 O((V + E) log V)로 더 빠를 수 있다. 크루스칼은 O(E log E)이므로 E가 V²에 가까우면 프림이 우세하다.

---

## 요약

Union-Find는 구현이 30줄 미만이지만, 경로 압축과 랭크 기반 합치기를 통해 역 아커만 함수 시간 복잡도라는 사실상 상수에 가까운 성능을 달성한다. 그래프 문제에서 연결성 판별이 등장한다면 Union-Find가 가장 먼저 고려해야 할 도구다.

---

## 참고 자료

- [Introduction to Disjoint Set — GeeksforGeeks](https://www.geeksforgeeks.org/dsa/introduction-to-disjoint-set-data-structure-or-union-find-algorithm/)
- [Union by Rank and Path Compression — GeeksforGeeks](https://www.geeksforgeeks.org/dsa/union-by-rank-and-path-compression-in-union-find-algorithm/)
- [Disjoint Set Union — cp-algorithms.com](https://cp-algorithms.com/data_structures/disjoint_set_union.html)
- [CS124 Lecture 6: Disjoint Set (Union-Find) — Georgetown University](https://people.cs.georgetown.edu/jthaler/ANLY550/lec6.pdf)
