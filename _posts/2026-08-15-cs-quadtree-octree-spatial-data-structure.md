---
layout: post
title: "쿼드트리(Quadtree)와 옥트리(Octree) 완전 정복: 공간 분할이 게임 엔진과 GIS를 바꾼 방법"
date: 2026-08-15
categories: [cs, computer-science]
tags: [quadtree, octree, spatial-index, data-structure, game-engine, collision-detection, gis]
---

지도 앱에서 화면을 확대할 때 새로운 건물들이 즉시 렌더링되고, 게임에서 수천 개의 오브젝트가 충돌 검사를 하면서도 60fps를 유지합니다. 이 모든 것의 배경에는 **쿼드트리(Quadtree)** 와 **옥트리(Octree)** 라는 공간 분할 자료구조가 있습니다. 이들은 2D 및 3D 공간을 계층적으로 분할하여 공간 기반 쿼리를 O(N)에서 O(log N) 수준으로 크게 줄여주는 핵심 자료구조입니다.

## 쿼드트리란 무엇인가?

쿼드트리(Quadtree)는 2D 공간을 재귀적으로 4개의 사분면으로 분할하는 트리 자료구조입니다. 각 내부 노드는 정확히 4개의 자식 노드를 가지며(Northwest, Northeast, Southwest, Southeast), 특정 영역에 포함된 점의 수가 임계값을 초과하면 해당 영역을 다시 분할합니다.

쿼드트리의 주요 유형은 다음과 같습니다.

- **포인트 쿼드트리(Point Quadtree)**: 2D 점 데이터를 저장. 삽입되는 점 기준으로 공간을 분할.
- **영역 쿼드트리(Region Quadtree)**: 영역을 균등하게 4분할. 이미지 압축·공간 인덱싱에 활용.
- **에지 쿼드트리(Edge Quadtree)**: 선분·다각형을 저장. 지리 정보 시스템(GIS)에 사용.

## 옥트리란 무엇인가?

옥트리(Octree)는 쿼드트리를 3D 공간으로 확장한 것입니다. 각 노드는 8개의 자식(옥탄트, Octant)으로 분할되며, 3D 게임 엔진, 포인트 클라우드, 의료 영상 분야에서 널리 사용됩니다.

## 왜 공간 분할 자료구조가 필요한가?

선형 구조로 공간 데이터를 관리하면 범위 검색(Range Query)이나 최근접 이웃 검색(Nearest Neighbor Query) 시 모든 원소를 순회해야 합니다. N개의 점이 있을 때 순진한 방법은 O(N)이지만, 쿼드트리를 사용하면 다음과 같은 개선이 가능합니다.

| 연산 | 순진한 방법 | 쿼드트리 (균형 시) |
|------|-----------|-----------------|
| 삽입 | O(1) | O(log N) |
| 점 검색 | O(N) | O(log N) |
| 범위 검색 | O(N) | O(√N + K) |
| 충돌 감지 | O(N²) | O(N log N) |

공간 데이터가 수십만~수백만 개가 되는 지도, 게임, 물리 시뮬레이션에서 이 차이는 실시간 처리 가능 여부를 결정합니다.

## 실제 구현 예제

### 예제 1: Python 포인트 쿼드트리 구현

```python
from dataclasses import dataclass, field
from typing import List, Optional

@dataclass
class Point:
    x: float
    y: float
    data: object = None


@dataclass
class Rect:
    """축 정렬 경계 직사각형 (AABB)"""
    cx: float  # 중심 x
    cy: float  # 중심 y
    hw: float  # 반폭 (half-width)
    hh: float  # 반높이 (half-height)

    def contains(self, point: Point) -> bool:
        return (self.cx - self.hw <= point.x <= self.cx + self.hw and
                self.cy - self.hh <= point.y <= self.cy + self.hh)

    def intersects(self, other: 'Rect') -> bool:
        return not (other.cx - other.hw > self.cx + self.hw or
                    other.cx + other.hw < self.cx - self.hw or
                    other.cy - other.hh > self.cy + self.hh or
                    other.cy + other.hh < self.cy - self.hh)


class Quadtree:
    CAPACITY = 4  # 노드당 최대 포인트 수 (초과 시 분할)

    def __init__(self, boundary: Rect, depth: int = 0, max_depth: int = 10):
        self.boundary = boundary
        self.depth = depth
        self.max_depth = max_depth
        self.points: List[Point] = []
        self.divided = False
        self.nw: Optional[Quadtree] = None
        self.ne: Optional[Quadtree] = None
        self.sw: Optional[Quadtree] = None
        self.se: Optional[Quadtree] = None

    def subdivide(self):
        """현재 영역을 4개의 사분면으로 분할"""
        cx, cy = self.boundary.cx, self.boundary.cy
        hw, hh = self.boundary.hw / 2, self.boundary.hh / 2
        depth = self.depth + 1
        max_d = self.max_depth

        self.nw = Quadtree(Rect(cx - hw, cy + hh, hw, hh), depth, max_d)
        self.ne = Quadtree(Rect(cx + hw, cy + hh, hw, hh), depth, max_d)
        self.sw = Quadtree(Rect(cx - hw, cy - hh, hw, hh), depth, max_d)
        self.se = Quadtree(Rect(cx + hw, cy - hh, hw, hh), depth, max_d)
        self.divided = True

        # 기존 포인트들을 자식 노드로 재삽입
        for p in self.points:
            self._insert_to_children(p)
        self.points = []

    def _insert_to_children(self, point: Point) -> bool:
        for child in (self.nw, self.ne, self.sw, self.se):
            if child.insert(point):
                return True
        return False

    def insert(self, point: Point) -> bool:
        if not self.boundary.contains(point):
            return False

        if self.divided:
            return self._insert_to_children(point)

        if len(self.points) < self.CAPACITY or self.depth >= self.max_depth:
            self.points.append(point)
            return True

        self.subdivide()
        return self._insert_to_children(point)

    def query(self, range_rect: Rect, found: List[Point] = None) -> List[Point]:
        """주어진 사각형 범위 안의 모든 포인트 반환"""
        if found is None:
            found = []

        if not self.boundary.intersects(range_rect):
            return found  # 겹치지 않으면 탐색 불필요 (가지치기)

        for p in self.points:
            if range_rect.contains(p):
                found.append(p)

        if self.divided:
            self.nw.query(range_rect, found)
            self.ne.query(range_rect, found)
            self.sw.query(range_rect, found)
            self.se.query(range_rect, found)

        return found


# 사용 예시
import random

qt = Quadtree(Rect(0, 0, 100, 100))

# 1000개의 무작위 점 삽입
for _ in range(1000):
    qt.insert(Point(random.uniform(-100, 100), random.uniform(-100, 100)))

# 특정 범위의 점 검색
search_area = Rect(0, 0, 20, 20)
results = qt.query(search_area)
print(f"범위 내 점 개수: {len(results)}")  # 약 100개 (전체의 10%)
```

### 예제 2: 충돌 감지에 쿼드트리 적용 (Java)

```java
import java.util.*;

public class CollisionDetection {

    record AABB(double cx, double cy, double hw, double hh, int id) {
        boolean intersects(AABB other) {
            return Math.abs(cx - other.cx) < hw + other.hw &&
                   Math.abs(cy - other.cy) < hh + other.hh;
        }
        boolean contains(double px, double py) {
            return Math.abs(cx - px) <= hw && Math.abs(cy - py) <= hh;
        }
    }

    static final int CAPACITY = 4;
    static final int MAX_DEPTH = 8;

    record QuadNode(AABB boundary, int depth,
                    List<AABB> objects,
                    QuadNode[] children) {

        static QuadNode create(AABB b, int depth) {
            return new QuadNode(b, depth, new ArrayList<>(), new QuadNode[4]);
        }
    }

    static void subdivide(QuadNode node) {
        double cx = node.boundary().cx(), cy = node.boundary().cy();
        double hw = node.boundary().hw() / 2, hh = node.boundary().hh() / 2;
        int d = node.depth() + 1;
        node.children()[0] = QuadNode.create(new AABB(cx - hw, cy + hh, hw, hh, -1), d);
        node.children()[1] = QuadNode.create(new AABB(cx + hw, cy + hh, hw, hh, -1), d);
        node.children()[2] = QuadNode.create(new AABB(cx - hw, cy - hh, hw, hh, -1), d);
        node.children()[3] = QuadNode.create(new AABB(cx + hw, cy - hh, hw, hh, -1), d);
    }

    static boolean insert(QuadNode node, AABB obj) {
        if (!node.boundary().contains(obj.cx(), obj.cy())) return false;

        if (node.children()[0] == null) {
            if (node.objects().size() < CAPACITY || node.depth() >= MAX_DEPTH) {
                node.objects().add(obj);
                return true;
            }
            subdivide(node);
            List<AABB> tmp = new ArrayList<>(node.objects());
            node.objects().clear();
            for (AABB o : tmp) insertToChildren(node, o);
        }
        return insertToChildren(node, obj);
    }

    static boolean insertToChildren(QuadNode node, AABB obj) {
        for (QuadNode child : node.children()) {
            if (child != null && insert(child, obj)) return true;
        }
        return false;
    }

    static List<AABB> retrieve(QuadNode node, AABB obj) {
        List<AABB> result = new ArrayList<>();
        if (!node.boundary().intersects(obj)) return result;
        result.addAll(node.objects());
        if (node.children()[0] != null) {
            for (QuadNode child : node.children())
                result.addAll(retrieve(child, obj));
        }
        return result;
    }

    // 충돌 감지: 쿼드트리를 이용해 O(N²)을 O(N log N)으로 최적화
    static Set<String> detectCollisions(List<AABB> objects) {
        QuadNode root = QuadNode.create(new AABB(0, 0, 500, 500, -1), 0);
        for (AABB obj : objects) insert(root, obj);

        Set<String> collisions = new HashSet<>();
        for (AABB obj : objects) {
            for (AABB candidate : retrieve(root, obj)) {
                if (candidate.id() != obj.id() && obj.intersects(candidate)) {
                    String key = Math.min(obj.id(), candidate.id()) + "-" +
                                 Math.max(obj.id(), candidate.id());
                    collisions.add(key);
                }
            }
        }
        return collisions;
    }

    public static void main(String[] args) {
        List<AABB> objects = new ArrayList<>();
        Random rand = new Random(42);
        for (int i = 0; i < 500; i++) {
            objects.add(new AABB(
                rand.nextDouble() * 900, rand.nextDouble() * 900,
                10, 10, i
            ));
        }

        long start = System.nanoTime();
        Set<String> cols = detectCollisions(objects);
        long elapsed = System.nanoTime() - start;

        System.out.printf("충돌 쌍: %d개, 소요 시간: %.2f ms%n",
                          cols.size(), elapsed / 1_000_000.0);
    }
}
```

## 옥트리(Octree)의 구조와 활용

옥트리는 3D 공간을 8개의 팔분체(Octant)로 분할합니다. 분할 기준은 X, Y, Z 세 축을 동시에 기준으로 하며, 구현은 쿼드트리와 동일한 패턴을 3차원으로 확장합니다.

```python
class OctreeNode:
    """3D 옥트리 노드"""
    CAPACITY = 8

    def __init__(self, cx, cy, cz, size, depth=0, max_depth=8):
        self.cx, self.cy, self.cz = cx, cy, cz
        self.size = size         # 정육면체 한 변 반길이
        self.depth = depth
        self.max_depth = max_depth
        self.points = []
        self.children = [None] * 8  # 8개 팔분체

    def _octant_index(self, px, py, pz):
        """점이 속하는 팔분체 인덱스 계산 (0~7)"""
        idx = 0
        if px >= self.cx: idx |= 4
        if py >= self.cy: idx |= 2
        if pz >= self.cz: idx |= 1
        return idx

    def _subdivide(self):
        half = self.size / 2
        offsets = [
            (-half, -half, -half), (-half, -half,  half),
            (-half,  half, -half), (-half,  half,  half),
            ( half, -half, -half), ( half, -half,  half),
            ( half,  half, -half), ( half,  half,  half),
        ]
        for i, (dx, dy, dz) in enumerate(offsets):
            self.children[i] = OctreeNode(
                self.cx + dx, self.cy + dy, self.cz + dz,
                half, self.depth + 1, self.max_depth
            )

    def insert(self, px, py, pz, data=None):
        """3D 점 삽입"""
        # 경계 검사
        s = self.size
        if not (self.cx - s <= px <= self.cx + s and
                self.cy - s <= py <= self.cy + s and
                self.cz - s <= pz <= self.cz + s):
            return False

        if self.children[0] is None:
            if len(self.points) < self.CAPACITY or self.depth >= self.max_depth:
                self.points.append((px, py, pz, data))
                return True
            self._subdivide()
            for p in self.points:
                self.children[self._octant_index(*p[:3])].insert(*p)
            self.points = []

        return self.children[self._octant_index(px, py, pz)].insert(px, py, pz, data)


# 사용 예시: 3D 포인트 클라우드 삽입
import random
root = OctreeNode(0, 0, 0, 100)
for _ in range(5000):
    root.insert(random.uniform(-100, 100),
                random.uniform(-100, 100),
                random.uniform(-100, 100))
print("3D 포인트 클라우드 구성 완료")
```

## 실제 사용 사례

**게임 엔진**: Unreal Engine과 Unity는 내부적으로 쿼드트리/옥트리를 사용해 가시성 결정(Frustum Culling), 충돌 감지, LOD(Level of Detail) 전환을 최적화합니다.

**지도 서비스**: 구글 맵, OpenStreetMap은 영역 쿼드트리(Region Quadtree)를 기반으로 지도 타일(Tile) 시스템을 구성합니다. 줌 레벨이 높아질수록 더 깊은 트리 노드에 해당하는 타일을 로드합니다.

**포인트 클라우드(LiDAR)**: 자율주행차의 LiDAR 센서는 초당 수십만 개의 3D 점을 생성합니다. 옥트리로 저장하면 인접 점 검색을 효율적으로 수행할 수 있습니다.

**의료 영상(CT/MRI)**: 3D 볼륨 데이터를 옥트리로 표현하면 균일한 빈 공간을 큰 노드 하나로 표현할 수 있어 메모리를 크게 절약합니다.

## 주의사항과 성능 팁

**1. 균형 문제**: 데이터가 극도로 편향되어 있으면(예: 모든 점이 한 구석에 몰림) 트리가 한쪽으로만 깊어져 최악의 경우 O(N)에 가까워질 수 있습니다. 이 경우 점 밀도에 기반한 적응적 분할을 고려하세요.

**2. CAPACITY 설정**: CAPACITY(노드당 최대 포인트 수)는 성능에 큰 영향을 미칩니다. 너무 작으면 트리가 지나치게 깊어지고, 너무 크면 리프 노드 선형 탐색이 증가합니다. 일반적으로 4~16 사이의 값이 좋습니다.

**3. 동적 삽입/삭제**: 점이 이동하는 경우(게임 오브젝트)에는 매 프레임 전체 트리를 재구성하거나, 점이 현재 노드를 벗어날 때만 재배치하는 전략을 사용합니다.

**4. 메모리 레이아웃**: 캐시 효율을 위해 노드를 배열에 선형으로 저장하고 인덱스로 자식을 참조하는 인덱스 기반 쿼드트리가 포인터 기반보다 캐시 친화적입니다.

**5. 루스 쿼드트리(Loose Quadtree)**: 각 사분면 경계를 실제 크기의 2배로 확장하면, 경계에 걸치는 오브젝트를 여러 노드에 중복 삽입하지 않아도 됩니다. 게임 충돌 감지에 자주 사용됩니다.

쿼드트리와 옥트리는 KD-트리, R-트리와 함께 공간 인덱싱의 핵심 자료구조 군을 형성합니다. 각각의 장단점을 이해하고, 데이터의 차원과 쿼리 패턴에 맞는 자료구조를 선택하는 것이 중요합니다.

## 참고 자료
- [d3/d3-quadtree — JavaScript 쿼드트리 구현 (D3.js 생태계)](https://github.com/d3/d3-quadtree)
- [pvigier/Quadtree — 현대적 C++17 쿼드트리 구현](https://github.com/pvigier/Quadtree)
- [williamfiset/Algorithms — 다양한 공간 알고리즘 컬렉션](https://github.com/williamfiset/Algorithms)
