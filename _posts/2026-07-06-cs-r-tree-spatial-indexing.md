---
layout: post
title: "R-트리(R-Tree) 완전 정복: 지도 앱과 공간 데이터베이스가 선택한 공간 인덱싱"
date: 2026-07-06
categories: [cs, computer-science]
tags: [r-tree, spatial-indexing, database, gis, nearest-neighbor, mbr, guttman]
---

"내 위치 반경 1km 안에 있는 카페를 찾아줘"라는 요청을 처리할 때, 데이터베이스는 수백만 개의 카페 좌표를 모두 뒤지지 않습니다. 바로 **R-트리(R-Tree)** 덕분입니다. 1984년 Antonin Guttman이 제안한 R-트리는 PostgreSQL/PostGIS, SQLite, MySQL 등 대부분의 공간 데이터베이스와 구글 지도, 우버, 에어비앤비 같은 위치 기반 서비스의 핵심 인덱싱 구조입니다.

---

## 1. 공간 데이터가 일반 B-트리로는 불가능한 이유

B-트리와 해시 인덱스는 "값이 X인 것을 찾아라" 같은 1차원 쿼리에 최적화되어 있습니다. 하지만 공간 쿼리는 다릅니다:

- **범위 쿼리:** 위도 37.4~37.6, 경도 126.8~127.0 안에 있는 모든 음식점
- **최근접 이웃:** 현재 위치에서 가장 가까운 병원 5개
- **교차 쿼리:** 이 폴리곤과 겹치는 행정구역은?

B-트리는 1차원 키를 기준으로 정렬합니다. 좌표를 (위도, 경도) 쌍으로 인덱싱하면 위도 하나만으로는 정렬되므로, 경도 조건을 추가하면 전체 스캔에 가까워집니다. **R-트리는 다차원 공간을 직접 인덱싱**하여 이 문제를 해결합니다.

---

## 2. R-트리의 핵심 개념: MBR과 계층 구조

### 2.1 최소 경계 사각형(MBR, Minimum Bounding Rectangle)

R-트리의 핵심 아이디어는 **공간 객체들을 가장 작은 사각형으로 묶는 것**입니다. 이 사각형을 MBR(Minimum Bounding Rectangle)이라 합니다.

```
예시: 서울 주요 시설의 MBR

카페 A (37.50, 127.02)  ───┐
카페 B (37.52, 127.05)     ├→ MBR1: [37.50~37.52, 127.02~127.05]
카페 C (37.51, 127.03)  ───┘

식당 D (37.56, 126.97)  ───┐
식당 E (37.58, 126.99)     ├→ MBR2: [37.56~37.58, 126.97~126.99]
식당 F (37.57, 126.98)  ───┘

MBR1과 MBR2를 묶으면 → 루트 MBR: [37.50~37.58, 126.97~127.05]
```

R-트리 노드는 **자식 노드들의 MBR과 자식 포인터**를 저장합니다. 리프 노드는 실제 공간 객체(포인트, 폴리곤 등)의 MBR과 데이터 포인터를 저장합니다.

### 2.2 R-트리 구조

R-트리는 B-트리와 마찬가지로 **높이 균형(Height-Balanced)** 트리입니다:

- 루트 노드를 제외한 모든 노드는 `m ≤ 엔트리 수 ≤ M` 조건을 만족 (m ≈ M/2)
- 리프 노드는 모두 같은 깊이에 위치
- 노드 하나가 디스크 페이지 크기(보통 4KB~16KB)에 맞게 설계됨

```
루트:     [전체 서울 MBR]
           /          \
내부:  [강남구 MBR]  [강북구 MBR]
        /     \          /     \
리프: [카페A,B,C] [식당D,E,F] [공원G,H] [병원I,J]
```

---

## 3. R-트리의 핵심 연산 구현

### 3.1 Python으로 구현하는 R-트리

```python
from dataclasses import dataclass, field
from typing import Optional, List, Tuple
import math

@dataclass
class MBR:
    """2D Minimum Bounding Rectangle"""
    min_x: float
    min_y: float
    max_x: float
    max_y: float

    def area(self) -> float:
        return (self.max_x - self.min_x) * (self.max_y - self.min_y)

    def enlargement_to_contain(self, other: 'MBR') -> float:
        """이 MBR이 other를 포함하려면 얼마나 넓어져야 하는가"""
        enlarged = self.union(other)
        return enlarged.area() - self.area()

    def union(self, other: 'MBR') -> 'MBR':
        return MBR(
            min(self.min_x, other.min_x),
            min(self.min_y, other.min_y),
            max(self.max_x, other.max_x),
            max(self.max_y, other.max_y)
        )

    def intersects(self, other: 'MBR') -> bool:
        return not (self.max_x < other.min_x or other.max_x < self.min_x or
                    self.max_y < other.min_y or other.max_y < self.min_y)

    def contains_point(self, x: float, y: float) -> bool:
        return self.min_x <= x <= self.max_x and self.min_y <= y <= self.max_y

    def min_dist_to_point(self, x: float, y: float) -> float:
        """점에서 MBR까지의 최소 거리"""
        dx = max(self.min_x - x, 0, x - self.max_x)
        dy = max(self.min_y - y, 0, y - self.max_y)
        return math.sqrt(dx*dx + dy*dy)


@dataclass
class RTreeNode:
    is_leaf: bool
    entries: List = field(default_factory=list)  # (MBR, child_or_data) 쌍
    MAX_ENTRIES = 4
    MIN_ENTRIES = 2

    def mbr(self) -> Optional[MBR]:
        """이 노드의 전체 MBR"""
        if not self.entries:
            return None
        result = self.entries[0][0]
        for mbr, _ in self.entries[1:]:
            result = result.union(mbr)
        return result

    def is_full(self) -> bool:
        return len(self.entries) >= self.MAX_ENTRIES


class RTree:
    def __init__(self):
        self.root = RTreeNode(is_leaf=True)

    def insert(self, mbr: MBR, data):
        """공간 객체 삽입"""
        leaf = self._choose_leaf(self.root, mbr)
        leaf.entries.append((mbr, data))

        # 오버플로우 처리
        split_node = None
        if leaf.is_full():
            leaf, split_node = self._split_node(leaf)

        self._adjust_tree(leaf, split_node)

    def _choose_leaf(self, node: RTreeNode, mbr: MBR) -> RTreeNode:
        """삽입할 리프 노드 선택: MBR 확장이 최소인 자식 선택"""
        if node.is_leaf:
            return node

        best_child = None
        best_enlargement = float('inf')
        best_area = float('inf')

        for child_mbr, child in node.entries:
            enlargement = child_mbr.enlargement_to_contain(mbr)
            area = child_mbr.area()
            # 확장이 최소인 것, 동점이면 면적이 최소인 것 선택
            if (enlargement < best_enlargement or
                    (enlargement == best_enlargement and area < best_area)):
                best_enlargement = enlargement
                best_area = area
                best_child = child

        return self._choose_leaf(best_child, mbr)

    def _split_node(self, node: RTreeNode):
        """Linear Split: 가장 멀리 떨어진 두 항목을 시드로 분할"""
        entries = node.entries[:]

        # 시드 선택: x축/y축에서 가장 멀리 분리된 두 항목
        seed1, seed2 = self._pick_seeds(entries)

        group1 = RTreeNode(is_leaf=node.is_leaf)
        group2 = RTreeNode(is_leaf=node.is_leaf)
        group1.entries.append(entries[seed1])
        group2.entries.append(entries[seed2])

        remaining = [e for i, e in enumerate(entries)
                     if i != seed1 and i != seed2]

        for entry in remaining:
            mbr, _ = entry
            mbr1 = group1.mbr()
            mbr2 = group2.mbr()

            # 어느 그룹의 MBR을 덜 확장하는지 비교
            d1 = mbr1.enlargement_to_contain(mbr) if mbr1 else 0
            d2 = mbr2.enlargement_to_contain(mbr) if mbr2 else 0

            if d1 <= d2:
                group1.entries.append(entry)
            else:
                group2.entries.append(entry)

        return group1, group2

    def _pick_seeds(self, entries):
        """가장 낭비가 큰 두 항목을 시드로 선택"""
        best_waste = -float('inf')
        seed1, seed2 = 0, 1

        for i in range(len(entries)):
            for j in range(i+1, len(entries)):
                combined = entries[i][0].union(entries[j][0])
                waste = combined.area() - entries[i][0].area() - entries[j][0].area()
                if waste > best_waste:
                    best_waste = waste
                    seed1, seed2 = i, j

        return seed1, seed2

    def _adjust_tree(self, node: RTreeNode, split: Optional[RTreeNode]):
        """루트까지 MBR 갱신, 필요시 부모 노드 분할"""
        # 단순화된 구현: 실제로는 부모 포인터 추적 필요
        if node is self.root and split:
            new_root = RTreeNode(is_leaf=False)
            new_root.entries = [
                (node.mbr(), node),
                (split.mbr(), split)
            ]
            self.root = new_root

    def search(self, query_mbr: MBR) -> List:
        """MBR과 교차하는 모든 객체 반환"""
        results = []
        self._search_recursive(self.root, query_mbr, results)
        return results

    def _search_recursive(self, node: RTreeNode, query: MBR, results: List):
        for child_mbr, child in node.entries:
            if child_mbr.intersects(query):
                if node.is_leaf:
                    results.append(child)  # 실제 데이터
                else:
                    self._search_recursive(child, query, results)

    def nearest_neighbor(self, x: float, y: float, k: int = 1) -> List:
        """k-최근접 이웃 탐색 (Best-First Search)"""
        import heapq

        heap = []  # (거리, 노드_또는_객체)
        results = []

        if self.root.mbr():
            heapq.heappush(heap, (0.0, id(self.root), self.root, False))

        while heap and len(results) < k:
            dist, _, item, is_leaf_entry = heapq.heappop(heap)

            if is_leaf_entry:
                results.append((dist, item))
                continue

            node = item
            for child_mbr, child in node.entries:
                min_dist = child_mbr.min_dist_to_point(x, y)
                if node.is_leaf:
                    heapq.heappush(heap, (min_dist, id(child), child, True))
                else:
                    heapq.heappush(heap, (min_dist, id(child), child, False))

        return results


# 사용 예시: 서울 카페 검색
rtree = RTree()

cafes = [
    (MBR(37.501, 127.024, 37.501, 127.024), "스타벅스 강남점"),
    (MBR(37.523, 127.047, 37.523, 127.047), "블루보틀 성수"),
    (MBR(37.557, 126.972, 37.557, 126.972), "카페 홍대"),
    (MBR(37.579, 126.977, 37.579, 126.977), "이디야 신촌"),
]

for mbr, name in cafes:
    rtree.insert(mbr, name)

# 강남구 범위 검색
gangnam_area = MBR(37.49, 127.00, 37.54, 127.10)
found = rtree.search(gangnam_area)
print(f"강남 범위 내 카페: {found}")

# 최근접 이웃: 현재 위치 (37.53, 127.02) 기준 가장 가까운 카페
nearest = rtree.nearest_neighbor(37.53, 127.02, k=2)
print(f"가장 가까운 카페 2개: {nearest}")
```

---

## 4. R*-트리: 실전에서 쓰이는 개선판

1990년 제안된 **R*-트리**는 Guttman의 원본 R-트리를 세 가지 핵심 방식으로 개선합니다:

1. **강제 재삽입(Forced Reinsertion):** 노드가 가득 찰 때 바로 분할하지 않고, 일부 항목을 트리에서 제거 후 재삽입합니다. 이는 트리의 구조를 더 컴팩트하게 유지합니다.

2. **개선된 분할 알고리즘:** 세 가지 기준(면적, 오버랩, 둘레)을 함께 고려하여 분할 축과 분할점을 결정합니다.

3. **삽입 시 오버랩 최소화:** ChooseLeaf에서 면적 확장 외에 오버랩 증가를 추가 기준으로 사용합니다.

```
R-트리 vs R*-트리 성능 비교 (일반적인 경우):
                R-트리   R*-트리
검색 성능:       기준     +20~40% 향상
삽입 속도:       기준     약간 느림 (재삽입 비용)
저장 공간 효율:  기준     더 높음 (MBR 겹침 최소)
```

---

## 5. PostGIS와 R-트리의 실전 활용

### 5.1 PostgreSQL + PostGIS 공간 인덱스

```sql
-- 공간 데이터 테이블 생성
CREATE TABLE places (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    location GEOMETRY(POINT, 4326)  -- WGS84 좌표계
);

-- R-트리 기반 GiST 인덱스 생성
CREATE INDEX places_location_idx ON places USING GIST(location);

-- 데이터 삽입
INSERT INTO places (name, location) VALUES
    ('스타벅스 강남', ST_SetSRID(ST_Point(127.0244, 37.4979), 4326)),
    ('블루보틀 성수', ST_SetSRID(ST_Point(127.0557, 37.5443), 4326)),
    ('카페 홍대',     ST_SetSRID(ST_Point(126.9243, 37.5563), 4326));

-- 반경 1km 이내 카페 검색 (인덱스 활용)
SELECT name,
       ST_Distance(
           location::geography,
           ST_SetSRID(ST_Point(127.02, 37.50), 4326)::geography
       ) AS dist_meters
FROM places
WHERE ST_DWithin(
    location::geography,
    ST_SetSRID(ST_Point(127.02, 37.50), 4326)::geography,
    1000  -- 1000미터
)
ORDER BY dist_meters;

-- EXPLAIN ANALYZE로 인덱스 사용 확인
EXPLAIN ANALYZE
SELECT * FROM places
WHERE ST_Within(
    location,
    ST_MakeEnvelope(127.0, 37.49, 127.1, 37.55, 4326)
);
-- "Index Scan using places_location_idx" 확인 가능
```

---

## 6. R-트리 변형들과 적용 분야

| 변형 | 특징 | 적용 |
|------|------|------|
| **R+-트리** | 오버랩 없음, 객체를 여러 노드에 분산 저장 | 오버랩 없는 2D 레이어 |
| **R*-트리** | 재삽입, 오버랩 최소화 | PostGIS, SQLite R*Tree |
| **Hilbert R-트리** | 힐베르트 커브로 공간 배치 최적화 | 정적 데이터 |
| **PR-트리** | 각 좌표에서 최적 성능 보장 | 이론적 최적 |

**주요 사용처:**
- **PostGIS:** PostgreSQL 공간 인덱스 (GiST 위에 R-트리 구현)
- **SQLite:** `fts5` 및 `rtree` 가상 테이블 모듈
- **게임 엔진:** Unity/Unreal의 충돌 감지 시스템
- **지도 서비스:** 타일 캐싱, 경계선 교차 판별

---

## 7. 주의사항과 팁

- **고차원에서의 한계:** R-트리는 2~3차원에서 가장 효과적입니다. 10차원 이상에서는 차원의 저주로 성능이 급격히 저하됩니다 (이 경우 KD-트리, 구형 LSH 사용 권장).
- **삽입 순서의 영향:** 최악의 경우 모든 MBR이 전체 공간을 덮어 검색 효율이 크게 떨어질 수 있습니다. **STR(Sort-Tile-Recursive)** 로딩을 사용하면 초기 구조가 훨씬 균형잡힙니다.
- **PostgreSQL 팁:** `SET enable_seqscan = off`으로 인덱스 사용을 강제하여 EXPLAIN 결과를 확인할 수 있습니다.

---

## 참고 자료

- [Guttman (1984) — R-Trees: A Dynamic Index Structure for Spatial Searching (ACM DL)](https://dl.acm.org/doi/10.1145/971697.602266)
- [Wikipedia — R-tree](https://en.wikipedia.org/wiki/R-tree)
- [PostGIS Documentation — Spatial Indexing](https://postgis.net/workshops/postgis-intro/indexing.html)
- [SQLite R*Tree Module Documentation](https://www.sqlite.org/rtree.html)
