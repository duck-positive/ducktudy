---
layout: post
title: "KD-트리와 공간 인덱싱 완전 정복: 최근접 이웃 탐색과 공간 쿼리의 핵심 알고리즘"
date: 2026-07-03
categories: [cs, computer-science]
tags: [kd-tree, spatial-indexing, nearest-neighbor, r-tree, geospatial, machine-learning, computational-geometry]
---

## KD-트리(K-Dimensional Tree)란?

KD-트리는 **K차원 공간의 점들을 효율적으로 탐색**하기 위한 이진 트리 자료구조입니다. 1975년 Jon Louis Bentley가 제안했으며, 이름 그대로 K차원 데이터를 분할하여 저장합니다.

주요 활용 분야를 살펴보면 KD-트리의 중요성이 드러납니다:

- **머신러닝**: KNN(K-Nearest Neighbors) 분류/회귀, scikit-learn이 기본적으로 사용
- **컴퓨터 그래픽스**: 레이트레이싱에서 물체와 광선의 교차 테스트
- **지리정보시스템(GIS)**: "반경 5km 이내 음식점 찾기" 같은 공간 쿼리
- **로보틱스**: 포인트 클라우드 처리, SLAM(동시 위치 추정 및 지도 작성)
- **추천 시스템**: 유사한 사용자나 아이템 찾기 (임베딩 공간에서의 ANN)

## 왜 KD-트리가 필요한가?

N개의 2D 점들 중에서 주어진 점과 가장 가까운 점을 찾는 문제를 생각해봅시다.

- **브루트포스**: 모든 점과 거리 계산 → O(N) 시간. N=10억이면 실용적이지 않습니다.
- **KD-트리**: 평균 O(log N), 최악 O(√N) (2D 기준). 공간을 재귀적으로 분할하여 탐색 범위를 극적으로 줄입니다.

실제로 scikit-learn의 `KNeighborsClassifier`는 내부적으로 KD-트리 또는 Ball Tree를 사용하며, 수백만 개의 학습 데이터에서도 밀리초 단위로 최근접 이웃을 찾습니다.

## KD-트리의 구조와 구축 알고리즘

### 핵심 아이디어: 초평면으로 공간 분할

KD-트리는 각 레벨마다 **하나의 축(dimension)을 기준으로 공간을 반으로 분할**합니다.

```
레벨 0: x축 기준 분할
레벨 1: y축 기준 분할
레벨 2: z축 기준 분할
레벨 3: x축 기준 분할 (순환)
...
```

각 노드는 분할 기준이 되는 **중간값 점(median point)**을 저장하고, 왼쪽 서브트리는 그 기준값보다 작은 점들, 오른쪽 서브트리는 큰 점들을 담습니다.

```python
from dataclasses import dataclass
from typing import Optional, List, Tuple
import math

@dataclass
class KDNode:
    point: tuple          # 이 노드가 저장하는 점
    axis: int             # 분할 축 (0=x, 1=y, ...)
    left: Optional['KDNode'] = None
    right: Optional['KDNode'] = None


class KDTree:
    def __init__(self, points: List[tuple]):
        """
        points: [(x0, y0), (x1, y1), ...] 형태의 좌표 목록
        """
        self.k = len(points[0]) if points else 0
        self.root = self._build(points, depth=0)

    def _build(self, points: List[tuple], depth: int) -> Optional[KDNode]:
        if not points:
            return None

        # 현재 깊이에서 분할할 축
        axis = depth % self.k

        # 해당 축 기준으로 정렬 후 중간값 선택
        sorted_points = sorted(points, key=lambda p: p[axis])
        median = len(sorted_points) // 2

        node = KDNode(
            point=sorted_points[median],
            axis=axis
        )
        # 중간값 기준으로 좌우 재귀 구축
        node.left  = self._build(sorted_points[:median],    depth + 1)
        node.right = self._build(sorted_points[median+1:], depth + 1)
        return node

    def _distance(self, p1: tuple, p2: tuple) -> float:
        """유클리드 거리 (제곱근 없이 비교용으로는 거리^2 사용)"""
        return math.sqrt(sum((a - b) ** 2 for a, b in zip(p1, p2)))

    def nearest_neighbor(self, target: tuple) -> Tuple[tuple, float]:
        """
        target에 가장 가까운 점과 그 거리를 반환.
        핵심 최적화: 초구(hypersphere) 교차 검사로 불필요한 서브트리 가지치기.
        """
        best = [None, float('inf')]  # [최근접 점, 최소 거리]

        def search(node: Optional[KDNode]):
            if node is None:
                return

            # 현재 노드와의 거리 갱신
            dist = self._distance(target, node.point)
            if dist < best[1]:
                best[0] = node.point
                best[1] = dist

            # 분할 초평면 기준으로 더 가까운 쪽을 먼저 탐색
            axis = node.axis
            diff = target[axis] - node.point[axis]
            closer    = node.left  if diff <= 0 else node.right
            farther   = node.right if diff <= 0 else node.left

            search(closer)

            # 핵심 가지치기: 현재 최소 거리 원이 분할 초평면을 넘는 경우에만
            # 반대쪽 서브트리를 탐색할 필요가 있음
            if abs(diff) < best[1]:
                search(farther)

        search(self.root)
        return best[0], best[1]

    def range_search(self, target: tuple, radius: float) -> List[tuple]:
        """
        target 기준 radius 이내의 모든 점 반환.
        GIS에서 "반경 R km 이내 검색"에 해당.
        """
        result = []

        def search(node: Optional[KDNode]):
            if node is None:
                return
            if self._distance(target, node.point) <= radius:
                result.append(node.point)

            axis = node.axis
            diff = target[axis] - node.point[axis]

            # 탐색할 방향 결정 (가까운 쪽 먼저)
            if diff <= 0:
                search(node.left)
                if abs(diff) <= radius:  # 분할선이 반경 원 안에 있으면 반대쪽도 탐색
                    search(node.right)
            else:
                search(node.right)
                if abs(diff) <= radius:
                    search(node.left)

        search(self.root)
        return result


# === 실행 예시 ===
points_2d = [
    (2, 3), (5, 4), (9, 6),
    (4, 7), (8, 1), (7, 2)
]

tree = KDTree(points_2d)

# 최근접 이웃 탐색
query = (6, 5)
nn_point, nn_dist = tree.nearest_neighbor(query)
print(f"쿼리 점: {query}")
print(f"최근접 이웃: {nn_point}, 거리: {nn_dist:.3f}")

# 반경 탐색
radius = 3.0
nearby = tree.range_search(query, radius)
print(f"\n반경 {radius} 이내 점들: {nearby}")

# 브루트포스와 결과 비교 (검증)
brute_force = min(points_2d, key=lambda p: math.dist(query, p))
print(f"\n브루트포스 결과: {brute_force} (KD-트리와 일치: {brute_force == nn_point})")
```

## KD-트리 시각적 이해: 분할 과정

6개 점에 대한 KD-트리 구축 과정:

```
초기 점들: (2,3), (5,4), (9,6), (4,7), (8,1), (7,2)

깊이 0 (x축 기준 분할):
  정렬: [(2,3), (4,7), (5,4), (7,2), (8,1), (9,6)]
  중간값: (7,2) → 루트 노드
  
  왼쪽: [(2,3), (4,7), (5,4)]  (x ≤ 7)
  오른쪽: [(8,1), (9,6)]        (x > 7)

깊이 1 (y축 기준 분할):
  왼쪽 그룹 y 정렬: [(2,3), (5,4), (4,7)]
  중간값: (5,4) → 루트의 왼쪽 자식
  
  오른쪽 그룹 y 정렬: [(8,1), (9,6)]
  중간값: (8,1) → 루트의 오른쪽 자식

최종 트리:
            (7,2)[x축]
           /          \
        (5,4)[y축]   (8,1)[y축]
        /    \           \
     (2,3)  (4,7)       (9,6)
```

## K-최근접 이웃(K-NN) 탐색

단 하나가 아닌 **K개의 최근접 이웃**을 효율적으로 찾는 방법입니다. 힙(최대 힙)을 사용하여 상위 K개를 유지합니다.

```python
import heapq

class KDTreeKNN(KDTree):
    def knn(self, target: tuple, k: int) -> List[Tuple[float, tuple]]:
        """
        K-최근접 이웃 탐색.
        반환: [(거리, 점), ...] k개, 거리 오름차순
        """
        # 최대 힙으로 상위 k개 유지 (Python heapq는 최소 힙이므로 음수 사용)
        heap = []  # [(-거리, 점)]

        def search(node: Optional[KDNode]):
            if node is None:
                return

            dist = self._distance(target, node.point)

            if len(heap) < k:
                heapq.heappush(heap, (-dist, node.point))
            elif dist < -heap[0][0]:
                heapq.heapreplace(heap, (-dist, node.point))

            axis = node.axis
            diff = target[axis] - node.point[axis]
            closer  = node.left  if diff <= 0 else node.right
            farther = node.right if diff <= 0 else node.left

            search(closer)

            # 힙이 k개 미만이거나, 분할선이 현재 최대 거리보다 가까우면 탐색
            if len(heap) < k or abs(diff) < -heap[0][0]:
                search(farther)

        search(self.root)

        # 거리 오름차순 정렬하여 반환
        return sorted([(-d, p) for d, p in heap])


# KNN 예시
knn_tree = KDTreeKNN(points_2d)
query = (6, 5)
neighbors = knn_tree.knn(query, k=3)
print(f"쿼리: {query}")
print("3-최근접 이웃:")
for dist, point in neighbors:
    print(f"  {point}, 거리: {dist:.3f}")
```

## 실제 지리정보 활용 예시

```python
import math
from typing import List, Tuple

def haversine_distance(lat1: float, lon1: float,
                       lat2: float, lon2: float) -> float:
    """
    두 위도/경도 좌표 간 거리(km) 계산 - Haversine 공식
    실제 GIS 시스템에서 지구 곡률을 고려한 거리 계산
    """
    R = 6371  # 지구 반지름 (km)
    lat1, lon1, lat2, lon2 = map(math.radians, [lat1, lon1, lat2, lon2])
    dlat = lat2 - lat1
    dlon = lon2 - lon1
    a = math.sin(dlat/2)**2 + math.cos(lat1)*math.cos(lat2)*math.sin(dlon/2)**2
    return 2 * R * math.asin(math.sqrt(a))


class GeoKDTree:
    """위도/경도 기반 위치 검색 KD-트리"""
    
    def __init__(self, locations: List[dict]):
        """
        locations: [{'name': '장소명', 'lat': 위도, 'lon': 경도}, ...]
        """
        self.locations = {(loc['lat'], loc['lon']): loc for loc in locations}
        points = [(loc['lat'], loc['lon']) for loc in locations]
        self.tree = KDTree(points) if points else None

    def find_nearest(self, lat: float, lon: float,
                     max_results: int = 5,
                     radius_km: float = 10.0) -> List[dict]:
        """
        주어진 위치에서 가장 가까운 장소들을 반환.
        위도/경도 1도 ≈ 111km이므로 km를 도 단위로 환산하여 KD-트리 검색.
        """
        if self.tree is None:
            return []

        # km → 도 단위 근사 변환 (적도 기준, 실제로는 위도에 따라 다름)
        radius_deg = radius_km / 111.0

        nearby_coords = self.tree.range_search((lat, lon), radius_deg)

        results = []
        for coord in nearby_coords:
            loc = self.locations.get(coord)
            if loc:
                exact_dist = haversine_distance(lat, lon, coord[0], coord[1])
                results.append({**loc, 'distance_km': round(exact_dist, 2)})

        # 실제 Haversine 거리로 정렬
        results.sort(key=lambda x: x['distance_km'])
        return results[:max_results]


# 서울 주요 장소 데이터
seoul_places = [
    {'name': '경복궁',   'lat': 37.5796, 'lon': 126.9770},
    {'name': '남산타워', 'lat': 37.5512, 'lon': 126.9882},
    {'name': '강남역',   'lat': 37.4980, 'lon': 127.0276},
    {'name': '홍대입구', 'lat': 37.5573, 'lon': 126.9243},
    {'name': '동대문',   'lat': 37.5714, 'lon': 127.0097},
    {'name': '인사동',   'lat': 37.5743, 'lon': 126.9854},
    {'name': '이태원',   'lat': 37.5345, 'lon': 126.9945},
    {'name': '잠실역',   'lat': 37.5132, 'lon': 127.1001},
]

geo_tree = GeoKDTree(seoul_places)
# 광화문 근처에서 5km 이내 장소 검색
my_lat, my_lon = 37.5755, 126.9769
nearby = geo_tree.find_nearest(my_lat, my_lon, max_results=5, radius_km=5.0)
print(f"광화문({my_lat}, {my_lon}) 기준 5km 이내 장소:")
for place in nearby:
    print(f"  {place['name']}: {place['distance_km']}km")
```

## R-트리: 영역 기반 공간 인덱싱

KD-트리는 점(Point) 데이터에 최적화되어 있지만, 현실에서는 **폴리곤(건물 경계), 라인(도로), 사각형(구역)** 같은 영역 데이터를 검색해야 하는 경우가 많습니다.

이때 사용하는 것이 **R-트리(R-Tree)**입니다. PostgreSQL의 PostGIS, SQLite의 SpatiaLite, 그리고 대부분의 공간 데이터베이스가 R-트리를 기본 인덱스로 사용합니다.

```sql
-- PostGIS + R-트리 인덱스 활용 예시
CREATE TABLE restaurants (
    id      SERIAL PRIMARY KEY,
    name    VARCHAR(100),
    location GEOGRAPHY(POINT, 4326)
);

-- R-트리 기반 GIST 인덱스 생성
CREATE INDEX idx_restaurants_location
    ON restaurants USING GIST(location);

-- 서울 광화문 기준 반경 1km 식당 검색 (인덱스 활용)
SELECT name,
       ST_Distance(location, ST_MakePoint(126.9769, 37.5755)::geography) AS dist_m
FROM   restaurants
WHERE  ST_DWithin(
           location,
           ST_MakePoint(126.9769, 37.5755)::geography,
           1000  -- 1000m = 1km
       )
ORDER BY dist_m
LIMIT 10;

-- EXPLAIN으로 인덱스 사용 확인
EXPLAIN ANALYZE
SELECT * FROM restaurants
WHERE ST_DWithin(location, ST_MakePoint(126.9769, 37.5755)::geography, 1000);
```

## 고차원에서의 한계: 차원의 저주

KD-트리는 **차원이 낮을수록(D < 20) 효율적**입니다. 차원이 높아질수록 "차원의 저주(Curse of Dimensionality)" 때문에 성능이 급격히 저하됩니다.

**왜 그럴까요?** 고차원 공간에서는 모든 점들이 서로 비슷한 거리를 갖게 됩니다. KD-트리의 분할 기준(초평면)이 무의미해지고, 결국 브루트포스와 비슷한 시간이 걸립니다.

해결책:
- **차원 축소**: PCA, UMAP으로 50→2차원으로 줄인 후 KD-트리 적용
- **Ball Tree**: 초평면 대신 초구(hypersphere)로 분할, 고차원에서 KD-트리보다 나음
- **HNSW (Hierarchical Navigable Small World)**: 현재 ANN 탐색 최고 성능. Spotify, Weaviate가 사용
- **Product Quantization**: 벡터를 서브벡터로 분할하여 근사 탐색

```python
# scikit-learn으로 실제 KNN 분류 — KD-트리와 Ball Tree 비교
from sklearn.neighbors import KNeighborsClassifier
from sklearn.datasets import make_classification
from sklearn.model_selection import train_test_split
import time

X, y = make_classification(n_samples=50000, n_features=10, random_state=42)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2)

for algorithm in ['kd_tree', 'ball_tree', 'brute']:
    clf = KNeighborsClassifier(n_neighbors=5, algorithm=algorithm)

    start = time.time()
    clf.fit(X_train, y_train)
    fit_time = time.time() - start

    start = time.time()
    accuracy = clf.score(X_test, y_test)
    query_time = time.time() - start

    print(f"{algorithm:10s}: 학습={fit_time:.3f}s, "
          f"예측={query_time:.3f}s, 정확도={accuracy:.4f}")
```

## 주의사항과 실무 팁

**동적 삽입의 비효율**: KD-트리는 대량의 데이터를 미리 알고 일괄 구축할 때 최적입니다. 하나씩 삽입하면 트리가 불균형해져 성능이 저하됩니다. 이때는 R-트리나 주기적 재구축을 고려합니다.

**삭제 연산**: KD-트리에서 노드 삭제는 복잡합니다. 단순히 노드를 제거하면 트리가 유효하지 않을 수 있어, 서브트리를 재구축하거나 "묘지 표시(tombstone)" 방식을 사용합니다.

**float 정밀도**: 부동소수점 비교에서 등호(=)를 사용할 때 epsilon 처리가 필요합니다.

**최적 축 선택**: 항상 `depth % k`로 축을 순환하는 대신, **분산이 가장 큰 축**을 선택하면 더 균형 잡힌 트리를 만들 수 있습니다(Variance-based splitting).

KD-트리는 공간 데이터를 다루는 모든 분야의 기반입니다. GPS 앱이 주변 장소를 밀리초 안에 찾고, 얼굴인식 시스템이 수백만 개의 얼굴 임베딩에서 일치하는 얼굴을 찾는 것 — 그 모든 것의 출발점이 이 우아한 트리 구조입니다.

## 참고 자료
- [KD-Tree in Python — John Lekberg](https://johnlekberg.com/blog/2020-04-17-kd-tree.html)
- [Approximate Nearest Neighbor with KD-Trees — PyImageSearch](https://pyimagesearch.com/2024/12/23/implementing-approximate-nearest-neighbor-search-with-kd-trees/)
- [Nearest Neighbors — scikit-learn documentation](https://scikit-learn.org/stable/modules/neighbors.html)
- [Wikipedia: k-d tree](https://en.wikipedia.org/wiki/K-d_tree)
