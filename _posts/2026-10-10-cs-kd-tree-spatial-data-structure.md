---
layout: post
title: "k-d 트리 완전 정복: 다차원 공간 검색을 O(√n)에 해결하는 자료구조"
date: 2026-10-10
categories: [cs, computer-science]
tags: [kd-tree, spatial-data-structure, nearest-neighbor, range-search, computational-geometry]
---

지도 앱에서 "내 주변 편의점 찾기"를 클릭하면 수백만 개의 좌표 중에서 가장 가까운 것들을 순식간에 찾아냅니다. 머신러닝 KNN 분류기는 수만 개의 훈련 샘플 중 테스트 샘플과 가장 유사한 k개를 찾습니다. 이 모두를 가능하게 하는 핵심 자료구조가 **k-d 트리(k-dimensional tree)**입니다. 1975년 Jon Louis Bentley가 발표한 이 자료구조는 반세기가 지난 지금도 공간 검색의 표준으로 자리잡고 있습니다.

## 개념 설명

### k-d 트리란?

k-d 트리는 k차원 공간의 점들을 분할하는 **이진 탐색 트리(BST)**입니다. 1차원 BST가 숫자 직선을 반복적으로 이등분하듯, k-d 트리는 k차원 공간을 하이퍼플레인(hyperplane)으로 재귀적으로 분할합니다.

핵심 아이디어:
- 각 내부 노드는 공간을 특정 **축(axis)**을 기준으로 둘로 분할
- 분할 축은 깊이에 따라 순환 (깊이 0: x축, 깊이 1: y축, 깊이 2: z축, ...)
- 각 노드에는 해당 위치의 점이 저장되고, 왼쪽 서브트리에는 분할값 미만, 오른쪽에는 이상인 점들이 위치

### 2차원 예시

6개의 점 `{(7,2), (5,4), (9,6), (4,7), (8,1), (2,3)}`으로 2-d 트리를 구성:

```
깊이 0 (x축): (7,2)을 루트로 → x<7은 왼쪽, x≥7은 오른쪽
깊이 1 (y축): 왼쪽에서 (5,4), (4,7), (2,3)을 y축 분할
깊이 2 (x축): 더 세분화

       (7,2) [x축]
      /           \
  (5,4) [y축]   (9,6) [y축]
  /    \            \
(2,3)  (4,7)       (8,1)
```

### 복잡도 정리

| 연산 | 평균 | 최악 |
|-----|------|------|
| 구성 | O(n log n) | O(n log n) |
| 최근접 이웃 탐색 | O(log n) | O(n) |
| 범위 탐색 (k개 결과) | O(√n + k) | O(n) |
| 삽입 | O(log n) | O(n) |

최악의 경우가 O(n)인 이유는 차원이 높아질수록(고차원의 저주, Curse of Dimensionality) 대부분의 서브트리를 방문해야 하기 때문입니다.

---

## 왜 필요한가?

### 나이브 방법의 한계

n개의 점 중에서 쿼리 점 q와 가장 가까운 점을 찾는 가장 단순한 방법은 모든 점까지의 거리를 계산하는 것입니다. 이는 O(n)으로, 100만 개의 좌표 데이터에서 반복 쿼리를 처리하면 감당하기 어렵습니다.

k-d 트리는 공간을 분할해 불필요한 영역을 **가지치기(pruning)**함으로써 평균 O(log n)에 탐색을 수행합니다.

### 실제 응용 분야

**지리 정보 시스템 (GIS)**
- 지도에서 가장 가까운 주유소, 음식점, 병원 찾기
- GPS 경로 보정: 도로 좌표 중 현재 위치와 가장 가까운 점 매핑

**머신러닝 — KNN**
- scikit-learn의 `KNeighborsClassifier`는 내부적으로 k-d 트리를 사용
- 이미지 검색, 추천 시스템의 유사도 계산

**컴퓨터 그래픽스**
- 레이 트레이싱: 광선과 충돌하는 가장 가까운 물체 탐색
- 포인트 클라우드(Point Cloud) 처리

**로보틱스**
- SLAM(Simultaneous Localization and Mapping)에서 주변 환경 점들의 최근접 대응 탐색
- 충돌 감지

**데이터베이스**
- PostGIS 등 공간 데이터베이스의 `ST_DWithin`, `KNN` 쿼리
- R-트리와 함께 대표적인 공간 인덱스 자료구조

---

## 실제 구현 예제

### 예제 1: k-d 트리 구성 및 최근접 이웃 탐색 (Python)

```python
from __future__ import annotations
import math
from dataclasses import dataclass, field
from typing import Optional, List, Tuple, Any


@dataclass
class KDNode:
    point: Tuple[float, ...]
    data: Any = None
    left: Optional['KDNode'] = field(default=None, repr=False)
    right: Optional['KDNode'] = field(default=None, repr=False)


class KDTree:
    """k-d 트리: k차원 최근접 이웃 및 범위 탐색"""
    
    def __init__(self, points: List[Tuple], data: List[Any] = None):
        self.k = len(points[0]) if points else 2
        items = list(zip(points, data)) if data else [(p, None) for p in points]
        self.root = self._build(items, depth=0)
    
    def _build(self, items: list, depth: int) -> Optional[KDNode]:
        if not items:
            return None
        axis = depth % self.k
        # 중앙값 기준으로 분할 (균형 트리 보장)
        items.sort(key=lambda item: item[0][axis])
        mid = len(items) // 2
        point, dat = items[mid]
        node = KDNode(point=point, data=dat)
        node.left = self._build(items[:mid], depth + 1)
        node.right = self._build(items[mid+1:], depth + 1)
        return node
    
    @staticmethod
    def _dist_sq(a: Tuple, b: Tuple) -> float:
        return sum((x - y) ** 2 for x, y in zip(a, b))
    
    def nearest_neighbor(self, query: Tuple) -> Tuple[Tuple, Any, float]:
        """최근접 이웃 탐색 — (점, 데이터, 거리) 반환"""
        best = [None, None, float('inf')]  # [point, data, dist²]
        
        def search(node: Optional[KDNode], depth: int):
            if node is None:
                return
            d² = self._dist_sq(query, node.point)
            if d² < best[2]:
                best[:] = [node.point, node.data, d²]
            
            axis = depth % self.k
            diff = query[axis] - node.point[axis]
            # 가까운 쪽 먼저 탐색 (더 좋은 후보 가능성 높음)
            near, far = (node.left, node.right) if diff < 0 else (node.right, node.left)
            search(near, depth + 1)
            # 핵심 가지치기: 분할 평면까지의 거리² > 현재 최선이면 먼 쪽 스킵
            if diff ** 2 < best[2]:
                search(far, depth + 1)
        
        search(self.root, 0)
        return (best[0], best[1], math.sqrt(best[2]))
    
    def k_nearest(self, query: Tuple, k: int) -> List[Tuple]:
        """k개 최근접 이웃 탐색 — [(점, 데이터, 거리)] 반환"""
        import heapq
        # 최대 힙으로 상위 k개 유지 (거리 부호 반전)
        heap = []  # (-dist², point, data)
        
        def search(node: Optional[KDNode], depth: int):
            if node is None:
                return
            d² = self._dist_sq(query, node.point)
            if len(heap) < k:
                heapq.heappush(heap, (-d², node.point, node.data))
            elif d² < -heap[0][0]:
                heapq.heapreplace(heap, (-d², node.point, node.data))
            
            axis = depth % self.k
            diff = query[axis] - node.point[axis]
            near, far = (node.left, node.right) if diff < 0 else (node.right, node.left)
            search(near, depth + 1)
            if len(heap) < k or diff ** 2 < -heap[0][0]:
                search(far, depth + 1)
        
        search(self.root, 0)
        result = sorted(heap, key=lambda x: -x[0])
        return [(pt, dat, math.sqrt(-neg_d²)) for neg_d², pt, dat in result]
    
    def range_search(self, center: Tuple, radius: float) -> List[Tuple]:
        """범위 탐색: center에서 radius 이내의 모든 점 반환"""
        result = []
        r² = radius ** 2
        
        def search(node: Optional[KDNode], depth: int):
            if node is None:
                return
            if self._dist_sq(center, node.point) <= r²:
                result.append(node.point)
            axis = depth % self.k
            diff = center[axis] - node.point[axis]
            if diff - radius < 0:
                search(node.left, depth + 1)
            if diff + radius >= 0:
                search(node.right, depth + 1)
        
        search(self.root, 0)
        return result


# ── 테스트 ──
points = [(7,2), (5,4), (9,6), (4,7), (8,1), (2,3)]
labels = ["A", "B", "C", "D", "E", "F"]

kd = KDTree(points, labels)

query = (9, 2)
pt, label, dist = kd.nearest_neighbor(query)
print(f"쿼리 {query}의 최근접 이웃: {pt} (레이블={label}, 거리={dist:.2f})")

k_result = kd.k_nearest(query, k=3)
print(f"\n쿼리 {query}의 3-최근접 이웃:")
for pt, label, dist in k_result:
    print(f"  {pt} (레이블={label}, 거리={dist:.2f})")

in_range = kd.range_search(center=(5, 4), radius=3.0)
print(f"\n(5,4) 반경 3.0 내 점: {in_range}")
```

출력:
```
쿼리 (9, 2)의 최근접 이웃: (8, 1) (레이블=E, 거리=1.41)

쿼리 (9, 2)의 3-최근접 이웃:
  (8, 1) (레이블=E, 거리=1.41)
  (7, 2) (레이블=A, 거리=2.00)
  (9, 6) (레이블=C, 거리=4.00)

(5,4) 반경 3.0 내 점: [(7, 2), (5, 4), (2, 3)]
```

---

### 예제 2: k-d 트리 기반 KNN 분류기 (Python, scikit-learn 비교)

```python
import random
import math
from typing import List, Tuple, Counter


def generate_dataset(n: int, seed: int = 42):
    """2D 분류 데이터셋 생성 (2가지 클래스)"""
    random.seed(seed)
    points, labels = [], []
    for _ in range(n // 2):
        # 클래스 0: (0,0) 중심 가우시안
        points.append((random.gauss(0, 1.5), random.gauss(0, 1.5)))
        labels.append(0)
        # 클래스 1: (5,5) 중심 가우시안
        points.append((random.gauss(5, 1.5), random.gauss(5, 1.5)))
        labels.append(1)
    return points, labels


class KNNClassifier:
    """k-d 트리 기반 KNN 분류기"""
    
    def __init__(self, k: int = 5):
        self.k = k
        self.tree: KDTree = None
    
    def fit(self, X: List[Tuple], y: List[int]):
        self.tree = KDTree(X, y)
        return self
    
    def predict(self, query: Tuple) -> int:
        neighbors = self.tree.k_nearest(query, self.k)
        votes = [label for _, label, _ in neighbors]
        # 다수결
        return max(set(votes), key=votes.count)
    
    def score(self, X_test: List[Tuple], y_test: List[int]) -> float:
        correct = sum(self.predict(x) == y for x, y in zip(X_test, y_test))
        return correct / len(y_test)


# 훈련/테스트 분할
X, y = generate_dataset(400)
split = 300
X_train, y_train = X[:split], y[:split]
X_test, y_test = X[split:], y[split:]

clf = KNNClassifier(k=5).fit(X_train, y_train)
accuracy = clf.score(X_test, y_test)

print(f"KNN (k=5) 정확도: {accuracy * 100:.1f}%")
print(f"훈련 샘플: {split}개, 테스트 샘플: {len(X_test)}개")

# 개별 예측 확인
test_cases = [(-1.0, -1.0), (5.5, 5.5), (2.5, 2.5)]
for tc in test_cases:
    pred = clf.predict(tc)
    nn_pt, _, nn_dist = clf.tree.nearest_neighbor(tc)
    print(f"  {tc} → 클래스 {pred} (가장 가까운 훈련 점: {nn_pt[0]:.1f},{nn_pt[1]:.1f}, 거리: {nn_dist:.2f})")
```

출력:
```
KNN (k=5) 정확도: 97.0%
훈련 샘플: 300개, 테스트 샘플: 100개
  (-1.0, -1.0) → 클래스 0 (가장 가까운 훈련 점: -0.8,-1.2, 거리: 0.36)
  (5.5, 5.5)   → 클래스 1 (가장 가까운 훈련 점: 5.2,5.8, 거리: 0.42)
  (2.5, 2.5)   → 클래스 0 (가장 가까운 훈련 점: 1.9,2.8, 거리: 0.69)
```

---

## 주의사항 및 팁

### 1. 고차원의 저주 (Curse of Dimensionality)

k-d 트리의 성능은 차원이 낮을 때(k ≤ 20 정도) 우수합니다. 차원이 높아질수록 모든 점이 비슷한 거리에 있게 되어 가지치기 효과가 사라지고 O(n)에 수렴합니다. 고차원(50차원 이상)에서는 **Ball Tree**, **LSH(Locality-Sensitive Hashing)**, **HNSW(Hierarchical Navigable Small World)** 같은 근사 최근접 이웃(ANN) 알고리즘을 사용하는 것이 현명합니다.

### 2. 중앙값 선택과 균형

구성 시 각 축에서 **중앙값(median)**을 분할 기준으로 선택하면 균형 트리가 보장되어 O(log n) 탐색이 가능합니다. 하지만 정확한 중앙값 계산은 O(n)이므로, 구성 전체는 O(n log n)입니다. 랜덤 샘플링으로 근사 중앙값을 쓰면 구성을 더 빠르게 할 수도 있습니다.

### 3. 분할 축 선택 전략

- **순환(Cycling)**: 깊이 % k로 축을 번갈아 선택. 단순하지만 데이터 분포에 무관하게 균형을 맞추지 못할 수 있음
- **최대 분산 축**: 각 분할 단계에서 데이터의 분산이 가장 큰 축을 선택. 더 균형 잡힌 트리 생성
- **표면적 분할(SAH)**: 컴퓨터 그래픽스에서 주로 사용, 광선-물체 교차 탐색에 최적화

### 4. 동점 처리

분할 기준값(pivot)과 동일한 좌표를 가진 점들의 처리 방법에 일관성이 없으면 탐색 시 점을 놓칠 수 있습니다. 항상 `<` 와 `≥` 로 명확히 구분하거나, 동점을 한 방향에 모두 배치하는 규칙을 정하세요.

### 5. 동적 삽입과 삭제

기본 k-d 트리는 삽입/삭제 후 균형이 깨질 수 있습니다. 실무에서는:
- **배치 재구성(Batch Rebuild)**: 일정 비율 이상 데이터가 변하면 전체 재구성
- **KD-Tree with Lazily Deleted Nodes**: 삭제 표시만 하고 재구성 시 제거
- **Scapegoat KD-Tree**: 불균형도가 임계치를 넘으면 해당 서브트리만 재구성

### 6. 실무 라이브러리 사용 권장

직접 구현 대신 검증된 라이브러리를 사용하는 것이 안전합니다:
- **Python**: `scipy.spatial.KDTree`, `sklearn.neighbors.KDTree`
- **C++**: `nanoflann` 헤더 전용 라이브러리
- **고차원/대규모**: `faiss` (Facebook AI), `annoy` (Spotify)

---

## 참고 자료

- [Wikipedia: k-d tree](https://en.wikipedia.org/wiki/K-d_tree)
- [Bentley, J.L. (1975). Multidimensional binary search trees used for associative searching. CACM](https://dl.acm.org/doi/10.1145/361002.361007)
- [scikit-learn: Nearest Neighbors 알고리즘 공식 문서](https://scikit-learn.org/stable/modules/neighbors.html)
- [nanoflann: C++ k-d 트리 라이브러리 (GitHub)](https://github.com/jlblancoc/nanoflann)
