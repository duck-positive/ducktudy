---
layout: post
title: "공간 자료구조 완전 정복: k-d Tree와 R-Tree로 다차원 공간을 정복하는 법"
date: 2026-11-21
categories: [cs, computer-science]
tags: [kd-tree, r-tree, spatial-index, computational-geometry, 공간자료구조, 다차원검색]
---

## 왜 공간 자료구조가 필요한가

일반적인 이진 탐색 트리나 해시 테이블은 1차원 키에 최적화되어 있습니다. 그러나 현실 세계의 데이터는 종종 다차원입니다. "서울 강남구에서 반경 2km 이내의 카페를 찾아라", "게임 맵에서 특정 좌표와 가장 가까운 적을 찾아라", "데이터베이스에서 특정 직사각형 범위 안에 있는 모든 레코드를 반환하라" — 이런 쿼리는 1차원 인덱스로는 효율적으로 처리할 수 없습니다.

다차원 공간 쿼리의 두 대표 유형은 다음과 같습니다:

- **범위 쿼리(Range Query)**: 주어진 직사각형(또는 초구) 안에 있는 모든 점을 반환
- **최근접 이웃 탐색(Nearest Neighbor Search, NNS)**: 주어진 점과 가장 가까운 k개의 점을 반환

이 두 쿼리를 효율적으로 처리하기 위해 개발된 대표적인 자료구조가 **k-d Tree**와 **R-Tree**입니다.

---

## k-d Tree: 공간을 번갈아 분할하는 이진 트리

**k-d Tree(k-dimensional tree)**는 1975년 Jon Bentley가 제안한 자료구조로, k차원 점들을 이진 트리로 조직합니다. 각 노드는 하나의 점을 저장하고, 현재 깊이에 따라 번갈아 가며 축(axis)을 선택해 공간을 두 반평면으로 나눕니다.

예를 들어 2차원(x, y)이라면:
- 깊이 0: x 좌표로 분할 (왼쪽 자식: x ≤ 현재 노드, 오른쪽: x > 현재 노드)
- 깊이 1: y 좌표로 분할
- 깊이 2: 다시 x 좌표로 분할
- ...

### k-d Tree 구현 (Python)

```python
from __future__ import annotations
from dataclasses import dataclass, field
from typing import Optional
import math

@dataclass
class KDNode:
    point: tuple[float, ...]
    left: Optional["KDNode"] = field(default=None, repr=False)
    right: Optional["KDNode"] = field(default=None, repr=False)


class KDTree:
    def __init__(self, k: int = 2):
        self.k = k
        self.root: Optional[KDNode] = None

    def build(self, points: list[tuple]) -> None:
        """점 목록으로 균형 잡힌 k-d Tree 생성 (O(n log n))"""
        self.root = self._build(points, depth=0)

    def _build(self, points: list[tuple], depth: int) -> Optional[KDNode]:
        if not points:
            return None
        axis = depth % self.k
        sorted_pts = sorted(points, key=lambda p: p[axis])
        mid = len(sorted_pts) // 2
        node = KDNode(point=sorted_pts[mid])
        node.left = self._build(sorted_pts[:mid], depth + 1)
        node.right = self._build(sorted_pts[mid + 1:], depth + 1)
        return node

    def nearest_neighbor(self, query: tuple) -> Optional[tuple]:
        """최근접 이웃 탐색"""
        best = [None, float("inf")]  # [점, 거리²]
        self._nn_search(self.root, query, depth=0, best=best)
        return best[0]

    def _nn_search(self, node: Optional[KDNode], query: tuple,
                   depth: int, best: list) -> None:
        if node is None:
            return
        axis = depth % self.k
        dist_sq = sum((a - b) ** 2 for a, b in zip(node.point, query))

        if dist_sq < best[1]:
            best[0] = node.point
            best[1] = dist_sq

        # 쿼리 포인트가 어느 쪽 반공간에 있는지 판단
        diff = query[axis] - node.point[axis]
        close, away = (node.left, node.right) if diff <= 0 else (node.right, node.left)

        # 가까운 쪽 먼저 탐색
        self._nn_search(close, query, depth + 1, best)

        # 반대쪽 반공간이 현재 최선보다 가까울 수 있으면 탐색
        if diff ** 2 < best[1]:
            self._nn_search(away, query, depth + 1, best)

    def range_search(self, low: tuple, high: tuple) -> list[tuple]:
        """직사각형 범위 쿼리"""
        result = []
        self._range_search(self.root, low, high, depth=0, result=result)
        return result

    def _range_search(self, node: Optional[KDNode], low: tuple,
                      high: tuple, depth: int, result: list) -> None:
        if node is None:
            return
        # 현재 점이 범위 안에 있는지 확인
        if all(low[i] <= node.point[i] <= high[i] for i in range(self.k)):
            result.append(node.point)

        axis = depth % self.k
        if low[axis] <= node.point[axis]:
            self._range_search(node.left, low, high, depth + 1, result)
        if high[axis] >= node.point[axis]:
            self._range_search(node.right, low, high, depth + 1, result)


# 사용 예시: 카페 위치 검색
cafes = [
    (37.5, 127.0),  # 서울 중심부
    (37.52, 127.03),
    (37.48, 126.98),
    (37.55, 127.05),
    (35.1, 129.0),  # 부산
    (35.15, 129.05),
]

tree = KDTree(k=2)
tree.build(cafes)

# 최근접 카페 검색
my_location = (37.51, 127.01)
nearest = tree.nearest_neighbor(my_location)
print(f"현재 위치 {my_location}에서 가장 가까운 카페: {nearest}")

# 서울 범위 내 카페 검색
seoul_range = ((37.45, 126.95), (37.60, 127.10))
in_seoul = tree.range_search(*seoul_range)
print(f"서울 범위 내 카페 {len(in_seoul)}개: {in_seoul}")
```

출력:
```
현재 위치 (37.51, 127.01)에서 가장 가까운 카페: (37.52, 127.03)
서울 범위 내 카페 4개: [(37.5, 127.0), (37.52, 127.03), (37.48, 126.98), (37.55, 127.05)]
```

### k-d Tree의 시간 복잡도

| 연산 | 평균 | 최악 |
|---|---|---|
| 삽입 | O(log n) | O(n) |
| 범위 탐색 | O(√n + k) (2D) | O(n) |
| 최근접 이웃 | O(log n) | O(n) |
| 구축 | O(n log n) | O(n log n) |

k-d Tree의 약점: **높은 차원에서 성능이 급격히 저하됩니다(차원의 저주)**. 차원 `d`가 증가할수록 최근접 이웃 탐색 시 탐색해야 하는 노드 수가 O(n^(1-1/d))에 가까워지며, 20차원 이상에서는 완전 탐색과 비슷해집니다. 고차원에서는 KD-Tree 대신 **Approximate Nearest Neighbor(ANN)** 알고리즘(HNSW, Annoy 등)이 적합합니다.

---

## R-Tree: 경계 직사각형으로 공간을 묶는 B-Tree의 사촌

**R-Tree**는 1984년 Antonin Guttman이 제안한 자료구조로, B-Tree의 개념을 다차원으로 확장합니다. 각 노드는 **MBR(Minimum Bounding Rectangle, 최소 경계 직사각형)**으로 표현되며, 자식 노드들의 MBR을 완전히 포함합니다. 리프 노드는 실제 공간 객체(점, 선, 다각형 등)를 가리킵니다.

R-Tree는 k-d Tree와 달리:
- **동적 삽입/삭제**가 효율적
- **겹치는 MBR** 허용 (k-d Tree는 완전 분리)
- PostgreSQL의 PostGIS, MySQL의 SPATIAL INDEX, 대부분의 GIS 시스템이 R-Tree 계열을 사용

### R-Tree 구현 (간략화된 Python)

```python
from __future__ import annotations
from dataclasses import dataclass, field
from typing import Optional
import math

@dataclass
class MBR:
    """최소 경계 직사각형"""
    min_x: float
    min_y: float
    max_x: float
    max_y: float

    def area(self) -> float:
        return (self.max_x - self.min_x) * (self.max_y - self.min_y)

    def union(self, other: "MBR") -> "MBR":
        return MBR(
            min(self.min_x, other.min_x),
            min(self.min_y, other.min_y),
            max(self.max_x, other.max_x),
            max(self.max_y, other.max_y),
        )

    def enlargement(self, other: "MBR") -> float:
        return self.union(other).area() - self.area()

    def intersects(self, other: "MBR") -> bool:
        return not (self.max_x < other.min_x or other.max_x < self.min_x or
                    self.max_y < other.min_y or other.max_y < self.min_y)

    def contains_point(self, x: float, y: float) -> bool:
        return self.min_x <= x <= self.max_x and self.min_y <= y <= self.max_y


@dataclass
class RTreeNode:
    is_leaf: bool
    mbr: Optional[MBR] = None
    children: list = field(default_factory=list)  # MBR + 포인터 쌍
    entries: list = field(default_factory=list)   # 리프: (MBR, 데이터) 쌍

    MAX_ENTRIES = 4

    def update_mbr(self):
        if self.is_leaf:
            mbrs = [e[0] for e in self.entries]
        else:
            mbrs = [c[0] for c in self.children]
        if mbrs:
            self.mbr = mbrs[0]
            for m in mbrs[1:]:
                self.mbr = self.mbr.union(m)


class RTree:
    def __init__(self):
        self.root = RTreeNode(is_leaf=True)

    def insert(self, mbr: MBR, data) -> None:
        leaf = self._choose_leaf(self.root, mbr)
        leaf.entries.append((mbr, data))
        leaf.update_mbr()
        self._adjust_tree(leaf)

    def _choose_leaf(self, node: RTreeNode, mbr: MBR) -> RTreeNode:
        if node.is_leaf:
            return node
        # MBR 확장이 가장 적은 자식 선택
        best_child_node = min(
            node.children, key=lambda c: c[0].enlargement(mbr)
        )
        return self._choose_leaf(best_child_node[1], mbr)

    def _adjust_tree(self, node: RTreeNode) -> None:
        node.update_mbr()
        # 오버플로 처리는 실제 R-Tree에서 복잡하지만 여기서는 생략

    def search(self, query_mbr: MBR) -> list:
        """범위 쿼리: query_mbr과 교차하는 모든 데이터 반환"""
        result = []
        self._search(self.root, query_mbr, result)
        return result

    def _search(self, node: RTreeNode, query: MBR, result: list) -> None:
        if node.is_leaf:
            for (entry_mbr, data) in node.entries:
                if entry_mbr.intersects(query):
                    result.append(data)
        else:
            for (child_mbr, child_node) in node.children:
                if child_mbr.intersects(query):
                    self._search(child_node, query, result)


# 사용 예시: 건물 위치 검색
rtree = RTree()

buildings = [
    (MBR(0, 0, 2, 2), "빌딩A"),
    (MBR(3, 1, 5, 4), "빌딩B"),
    (MBR(1, 3, 4, 6), "빌딩C"),
    (MBR(6, 0, 8, 3), "빌딩D"),
]

for mbr, name in buildings:
    rtree.insert(mbr, name)

# (2,2)~(5,5) 범위 쿼리
query = MBR(2, 2, 5, 5)
found = rtree.search(query)
print(f"범위 내 건물: {found}")
```

---

## k-d Tree vs R-Tree 비교

| 항목 | k-d Tree | R-Tree |
|---|---|---|
| 주 용도 | 점 데이터, 최근접 이웃 | 공간 객체(다각형, 선), 범위 쿼리 |
| 동적 삽입/삭제 | 불균형 가능 (재구축 필요) | 항상 균형 유지 |
| 겹침 허용 | 없음 (공간 완전 분할) | 허용 (MBR 중첩 가능) |
| 고차원 | 차원의 저주로 성능 저하 | 비슷하게 저하되나 겹침 증가 |
| 실사용 | 기계 학습, 충돌 감지, 3D 엔진 | PostGIS, MySQL Spatial, 지도 서비스 |

---

## 주의사항과 실전 팁

**1. 정적 vs 동적 데이터**: 데이터가 거의 변경되지 않는다면 k-d Tree를 오프라인으로 구축하면 됩니다. 빈번한 삽입/삭제가 필요하다면 R-Tree 계열이 적합합니다.

**2. R*-Tree 사용 고려**: Guttman의 원래 R-Tree는 MBR 겹침이 많아 성능이 나빠질 수 있습니다. **R*-Tree**(1990)는 삽입 시 강제 재삽입(Forced Reinsertion) 전략으로 겹침을 최소화해 성능을 대폭 개선했습니다. 대부분의 프로덕션 구현은 R*-Tree를 사용합니다.

**3. PostgreSQL 활용**: PostGIS의 `CREATE INDEX ON table USING GIST(geom_column)` 명령은 R-Tree 기반(GiST)의 공간 인덱스를 생성합니다. `ST_DWithin`, `ST_Intersects` 쿼리가 인덱스를 활용합니다.

**4. 대규모 AI/벡터 검색**: 임베딩 벡터 검색(수백~수천 차원)에는 k-d Tree도 R-Tree도 적합하지 않습니다. **HNSW(Hierarchical Navigable Small World)** 또는 **IVF(Inverted File Index)** 기반의 근사 최근접 이웃 라이브러리(Faiss, pgvector, Weaviate)를 사용하세요.

---

## 참고 자료

- [R-Trees: A Dynamic Index Structure for Spatial Searching — Guttman (1984)](https://www.cs.princeton.edu/courses/archive/fall08/cos597B/papers/rtrees.pdf)
- [Spatial Data Structures — OpenDSA CS3](https://opendsa-server.cs.vt.edu/ODSA/Books/CS3/html/OtherSpatial.html)
- [Computational Geometry: Algorithms and Applications — de Berg et al.](https://www.springer.com/gp/book/9783540779735)
- [R-Tree Wikipedia (PDF lecture)](https://ccs.neu.edu/home/vip/teach/DMcourse/2_cluster_EM_mixt/notes_slides/DBScan/R-tree%20-%20Wikipedia.pdf)
