---
layout: post
title: "보로노이 다이어그램과 들로네 삼각분할 완전 정복: Fortune 알고리즘으로 O(n log n)에 근접 이웃 구조 만들기"
date: 2026-10-18
categories: [cs, computer-science]
tags: [voronoi-diagram, delaunay-triangulation, fortune-algorithm, computational-geometry, sweep-line]
---

## 개요

지도 위에 편의점 100개가 있을 때, "내 위치에서 가장 가까운 편의점은 어디인가?"라는 질문에 O(log n)으로 답하는 공간 자료구조가 **보로노이 다이어그램(Voronoi Diagram)**이다. 이와 쌍대(dual) 관계에 있는 **들로네 삼각분할(Delaunay Triangulation)**은 점집합을 삼각형으로 분할하는 방법 중 삼각형의 최소 내각을 최대화하는 '가장 고른' 분할을 제공한다.

이 두 구조는 컴퓨터 그래픽스(메쉬 생성), GIS(지리 분석), 로보틱스(경로 계획), 도시 계획, 분산 시스템(데이터 파티셔닝) 등 수십 개의 분야에서 핵심 도구로 활용된다. 이 글에서는 두 구조의 수학적 원리와 Fortune의 스위프라인 알고리즘, 그리고 Python·C++ 구현까지 완전히 다룬다.

---

## 보로노이 다이어그램이란

### 정의

평면 위의 n개의 **씨앗 점(site)** 집합 P = {p₁, p₂, ..., pₙ}에 대해, 보로노이 다이어그램 Vor(P)는 평면을 n개의 **보로노이 영역(Voronoi cell)**으로 분할한다. 씨앗 pᵢ에 대응하는 보로노이 영역 V(pᵢ)는 다음과 같이 정의된다.

```
V(pᵢ) = { q ∈ ℝ² | d(q, pᵢ) ≤ d(q, pⱼ) for all j ≠ i }
```

즉, pᵢ가 다른 어떤 씨앗보다 가까운 점들의 집합이다. 유클리드 거리를 쓰면 각 경계선은 두 씨앗의 **수직 이등분선(perpendicular bisector)**의 일부가 된다.

### 성질

- **꼭짓점(Voronoi vertex)**: 세 개의 보로노이 영역이 만나는 점. 세 씨앗으로부터 등거리(외접원의 중심).
- **간선(Voronoi edge)**: 두 씨앗의 수직 이등분선의 일부.
- **복잡도**: n개 씨앗의 보로노이 다이어그램은 최대 2n-5개의 꼭짓점, 3n-6개의 간선, n개의 영역을 가진다 (오일러 공식 적용).
- **무한 영역**: 볼록 껍질(convex hull) 위의 씨앗들은 무한한 보로노이 영역을 가진다.

---

## 들로네 삼각분할이란

### 쌍대 그래프

보로노이 다이어그램의 **쌍대 그래프**가 들로네 삼각분할이다. 구체적으로:

- 보로노이 꼭짓점 → 들로네 삼각형
- 보로노이 간선 → 들로네 간선
- 보로노이 영역 → 들로네 점(씨앗)

두 씨앗 pᵢ, pⱼ는 그들의 보로노이 영역이 간선을 공유할 때 들로네 그래프에서 연결된다.

### 들로네 조건

삼각분할 T가 들로네 삼각분할이려면 모든 삼각형의 **외접원** 내부에 다른 점이 없어야 한다.

```
삼각형 abc의 외접원 안에 다른 어떤 점 d도 존재하지 않는다.
```

이 조건은 삼각형의 최소 내각을 최대화한다는 의미에서 "가장 고른" 삼각분할을 제공한다. 실제로 들로네 삼각분할은 모든 삼각분할 중 최소 각도를 최대화(lexicographic maximum of sorted angles)함이 증명되어 있다.

### 쌍대 관계의 의미

| 보로노이 | 들로네 |
|---------|-------|
| 각 씨앗의 영역 | 각 씨앗(꼭짓점) |
| 공유 경계 간선 | 두 씨앗을 잇는 간선 |
| 세 영역의 꼭짓점 | 세 씨앗을 잇는 삼각형 |
| 외접원 중심 | 보로노이 꼭짓점 |

---

## Fortune의 스위프라인 알고리즘 O(n log n)

### 나이브 알고리즘의 한계

n개 씨앗에 대해 직접 수직 이등분선을 계산하는 브루트-포스는 O(n²). n = 10⁶일 때 실용적이지 않다.

### Fortune 알고리즘 개요

1987년 Steven Fortune이 고안한 이 알고리즘은 **스위프라인(sweep line)** l을 y축에 평행하게 위에서 아래로 이동시키며 보로노이 다이어그램을 구성한다.

핵심 개념:

- **해변선(beach line)**: 스위프라인이 지나간 씨앗들에 의해 생성되는 포물선들의 아래쪽 포락선(lower envelope). 씨앗 pᵢ에서 스위프라인 l까지의 거리와 임의의 점 q에서 pᵢ까지의 거리가 같은 포물선들이 겹쳐져 해변선을 형성한다.
- **이벤트 큐**: (1) **씨앗 이벤트(site event)**: 스위프라인이 새 씨앗을 만날 때. (2) **원 이벤트(circle event)**: 해변선에서 포물선 호(arc)가 수축해 사라질 때.

알고리즘은 이 두 종류의 이벤트를 우선순위 큐로 처리하며 보로노이 간선과 꼭짓점을 생성한다.

---

## 구현 예제 1 — Python: scipy를 활용한 들로네 삼각분할

완전한 Fortune 알고리즘 구현은 수백 줄이므로, 먼저 `scipy`의 들로네 삼각분할을 활용해 시각화하는 예제를 보자.

```python
import numpy as np
from scipy.spatial import Delaunay, Voronoi, voronoi_plot_2d
import matplotlib.pyplot as plt

def demo_delaunay_voronoi():
    rng = np.random.default_rng(42)
    points = rng.uniform(0, 10, size=(20, 2))

    # 들로네 삼각분할
    tri = Delaunay(points)

    fig, axes = plt.subplots(1, 2, figsize=(12, 5))

    # 들로네 시각화
    ax1 = axes[0]
    ax1.triplot(points[:, 0], points[:, 1], tri.simplices, color='steelblue', lw=0.8)
    ax1.scatter(points[:, 0], points[:, 1], color='tomato', zorder=5, s=40)
    ax1.set_title("Delaunay Triangulation")
    ax1.set_aspect('equal')

    # 보로노이 시각화
    vor = Voronoi(points)
    ax2 = axes[1]
    voronoi_plot_2d(vor, ax=ax2, show_vertices=False,
                    line_colors='steelblue', line_width=0.8,
                    point_size=4)
    ax2.set_title("Voronoi Diagram")
    ax2.set_aspect('equal')
    ax2.set_xlim(-1, 11)
    ax2.set_ylim(-1, 11)

    plt.tight_layout()
    plt.savefig("voronoi_delaunay.png", dpi=100)
    print("그림 저장 완료")

demo_delaunay_voronoi()

# --- 들로네 삼각분할로 최근접 이웃 탐색 ---
def nearest_neighbor_voronoi(points, query):
    """보로노이/들로네를 활용한 최근접 이웃 탐색."""
    from scipy.spatial import cKDTree
    # 직접 KD-tree 대신, 들로네 인접 관계를 이용한 워킹 탐색
    tree = cKDTree(points)
    dist, idx = tree.query(query)
    return idx, dist

points = np.array([[1,1],[2,5],[3,2],[6,3],[8,7],[5,9],[4,4]], dtype=float)
query = np.array([4.5, 4.5])
idx, dist = nearest_neighbor_voronoi(points, query)
print(f"질의점 {query}의 최근접 이웃: 점 {idx} = {points[idx]}, 거리: {dist:.3f}")
```

### 들로네 삼각분할의 외접원 조건 검증

```python
import numpy as np

def circumcircle(a, b, c):
    """세 점의 외접원 중심과 반지름 반환."""
    ax, ay = a
    bx, by = b
    cx, cy = c

    D = 2 * (ax * (by - cy) + bx * (cy - ay) + cx * (ay - by))
    if abs(D) < 1e-10:
        return None, float('inf')  # 공선(collinear)

    ux = ((ax**2 + ay**2) * (by - cy) +
          (bx**2 + by**2) * (cy - ay) +
          (cx**2 + cy**2) * (ay - by)) / D
    uy = ((ax**2 + ay**2) * (cx - bx) +
          (bx**2 + by**2) * (ax - cx) +
          (cx**2 + cy**2) * (bx - ax)) / D

    center = np.array([ux, uy])
    radius = np.linalg.norm(a - center)
    return center, radius

def check_delaunay_condition(points, triangles):
    """들로네 조건 검사: 각 삼각형 외접원 안에 다른 점이 없어야 함."""
    for tri in triangles:
        a, b, c = points[tri[0]], points[tri[1]], points[tri[2]]
        center, radius = circumcircle(a, b, c)
        if center is None:
            continue
        for i, p in enumerate(points):
            if i in tri:
                continue
            dist = np.linalg.norm(p - center)
            if dist < radius - 1e-9:
                print(f"삼각형 {tri}: 들로네 조건 위반! 점 {i} ({p})가 외접원 내부에 있음.")
                return False
    print("들로네 조건 만족!")
    return True

from scipy.spatial import Delaunay
pts = np.array([[0,0],[4,0],[2,3],[2,1],[1,2]], dtype=float)
tri = Delaunay(pts)
check_delaunay_condition(pts, tri.simplices)
```

---

## 구현 예제 2 — Fortune 알고리즘 핵심 스케치 (Python)

완전 구현은 복잡하지만 핵심 구조를 이해하기 위한 스케치다.

```python
import heapq
from dataclasses import dataclass, field
from typing import Optional

@dataclass(order=True)
class Event:
    y: float
    x: float
    is_site: bool
    site_idx: int = field(compare=False, default=-1)
    arc: Optional[object] = field(compare=False, default=None)
    valid: bool = field(compare=False, default=True)

class Arc:
    """해변선 위의 포물선 호."""
    def __init__(self, site):
        self.site = site
        self.prev = None
        self.next = None
        self.event = None   # 이 호가 사라지는 원 이벤트

class FortuneVoronoi:
    """Fortune 알고리즘 스켈레톤."""

    def __init__(self, sites):
        self.sites = sites
        self.edges = []
        self.vertices = []
        self.beach = None   # 해변선 (연결 리스트)
        self.pq = []        # 이벤트 큐 (min-heap by y)

    def compute(self):
        # 1. 씨앗 이벤트 초기화
        for i, (x, y) in enumerate(self.sites):
            heapq.heappush(self.pq, Event(y=-y, x=x, is_site=True, site_idx=i))

        while self.pq:
            event = heapq.heappop(self.pq)
            if not event.valid:
                continue
            if event.is_site:
                self._handle_site_event(event)
            else:
                self._handle_circle_event(event)

        # 2. 무한 간선 마무리 처리 (bounding box 클리핑)
        self._finish_edges()

    def _handle_site_event(self, event):
        """
        새 씨앗이 스위프라인과 만날 때:
        - 해변선에서 이 씨앗 바로 위의 호(arc)를 찾는다.
        - 해당 호를 세 개로 분할한다.
        - 새 보로노이 간선의 반무한 선분을 시작한다.
        - 기존 원 이벤트를 취소하고 새 원 이벤트를 추가한다.
        """
        # (간략화된 의사코드)
        pass

    def _handle_circle_event(self, event):
        """
        호가 수축해 사라질 때 (세 씨앗이 공통 외접원을 가질 때):
        - 보로노이 꼭짓점을 추가한다.
        - 이전/다음 호를 연결한다.
        - 새 원 이벤트를 검사한다.
        """
        pass

    def _finish_edges(self):
        """해변선에 남은 반무한 간선들을 경계 상자에서 자른다."""
        pass

# Fortune 알고리즘은 구현 상 데이터 구조 조율이 복잡하다.
# 실무에서는 scipy.spatial.Voronoi 혹은 CGAL 라이브러리를 활용한다.
print("Fortune 알고리즘 스켈레톤 완료 — 실제 구현은 scipy/CGAL 활용 권장")
```

---

## 들로네 삼각분할의 증분(Incremental) 알고리즘

Fortune보다 구현이 간단한 **증분 삽입 알고리즘**:

1. 모든 점을 포함하는 슈퍼삼각형(super-triangle)으로 시작.
2. 점을 하나씩 삽입. 해당 점이 들어가는 삼각형의 외접원에 점이 있으면 그 삼각형들을 제거하고 새 삼각형으로 재분할.
3. **Lawson Flip**: 들로네 조건을 위반하는 간선을 플립(flip)해 조건을 만족시킴. 각 플립은 인접 두 삼각형을 다른 대각선으로 재연결하는 것.
4. 마지막에 슈퍼삼각형의 꼭짓점과 연결된 삼각형 제거.

```python
def delaunay_incremental_simple(points):
    """
    Bowyer-Watson 증분 알고리즘 (간략화).
    실제 구현에서는 수치 정밀도 처리가 필요하다.
    """
    import numpy as np

    def in_circumcircle(a, b, c, d):
        """점 d가 삼각형 abc의 외접원 내부에 있는지 판별."""
        ax, ay = a[0]-d[0], a[1]-d[1]
        bx, by = b[0]-d[0], b[1]-d[1]
        cx, cy = c[0]-d[0], c[1]-d[1]
        return (ax*(by*1 - cy*1) - bx*(ay*1 - cy*1) + cx*(ay*1 - by*1) > 0 and
                np.linalg.det([[ax, ay, ax**2+ay**2],
                                [bx, by, bx**2+by**2],
                                [cx, cy, cx**2+cy**2]]) > 0)

    pts = list(points)
    n = len(pts)

    # 슈퍼삼각형 추가
    M = 1e5
    pts += [(-M, -M), (M, -M), (0, M)]
    triangles = {(n, n+1, n+2)}  # 슈퍼삼각형

    for i in range(n):
        p = np.array(pts[i])
        bad = set()
        for tri in triangles:
            a, b, c = np.array(pts[tri[0]]), np.array(pts[tri[1]]), np.array(pts[tri[2]])
            if in_circumcircle(a, b, c, p):
                bad.add(tri)

        # 경계 다각형(boundary polygon) 찾기
        boundary = []
        for tri in bad:
            for edge in [(tri[0],tri[1]), (tri[1],tri[2]), (tri[2],tri[0])]:
                shared = any(
                    (edge[0] in t and edge[1] in t) for t in bad if t != tri
                )
                if not shared:
                    boundary.append(edge)

        # bad 삼각형 제거 후 재삼각화
        triangles -= bad
        for e0, e1 in boundary:
            triangles.add((e0, e1, i))

    # 슈퍼삼각형 꼭짓점 관련 제거
    triangles = {
        tri for tri in triangles
        if n not in tri and n+1 not in tri and n+2 not in tri
    }
    return list(triangles)

pts = [(0,0),(4,0),(2,3),(2,1),(1,2)]
triangles = delaunay_incremental_simple(pts)
print("들로네 삼각분할 결과:", triangles)
```

---

## 응용: 최근접 이웃 탐색과 공간 분할

보로노이 다이어그램의 가장 직접적인 응용은 **최근접 이웃(Nearest Neighbor) 탐색**이다.

- 질의점 q가 속한 보로노이 셀을 찾으면 → 해당 셀의 씨앗이 최근접 이웃.
- 점 위치(point location) 문제로 환원되어 O(log n)에 해결 가능 (슬랩 분해 또는 Kirk-Patric's hierarchy 사용).
- k-근접 이웃은 인접 셀까지 확장 탐색.

들로네 삼각분할은 **Ruppert의 메쉬 정제(refinement) 알고리즘**에도 사용된다. 모든 삼각형의 최소 내각이 특정 임계값 이상이 되도록 점을 추가/이동해 고품질 삼각형 메쉬를 생성한다. 이는 FEM(유한요소법), 게임 지형 렌더링, 물리 시뮬레이션에 필수다.

---

## 복잡도 요약

| 알고리즘 | 시간 | 공간 | 비고 |
|---------|------|------|------|
| 나이브 보로노이 | O(n²) | O(n) | 수직 이등분선 직접 교차 |
| Fortune 스위프라인 | O(n log n) | O(n) | 최적 |
| 증분 삽입(flip) | O(n²) worst, O(n log n) avg | O(n) | 랜덤 삽입 순서 시 |
| Bowyer-Watson | O(n log n) avg | O(n) | 간단한 구현 |

---

## 주의사항 및 팁

1. **퇴화(degenerate) 경우**: 4점이 공원(co-circular)이면 외접원 조건이 애매해진다. ε-perturbation이나 심볼릭 퍼터베이션(SOS)으로 처리한다.

2. **수치 정밀도**: 부동소수점 산술로 인한 외접원 검사 오류는 틀린 삼각분할을 만들 수 있다. 정확한 산술(exact arithmetic) 라이브러리(CGAL, libigl, Shewchuk's predicates)를 사용하라.

3. **경계 처리**: 무한 영역을 가진 보로노이 셀은 실용적으로 바운딩 박스(bounding box)로 클리핑해야 한다.

4. **고차원**: n차원 들로네 삼각분할은 `scipy.spatial.Delaunay`가 지원하며, Quickhull 기반으로 구현되어 있다.

5. **가중 보로노이**: 씨앗마다 가중치를 부여하는 Power Diagram(파워 다이어그램)은 가중 들로네 삼각분할의 쌍대로, 곱셈 가중치나 거듭제곱 거리로 확장된다.

---

## 참고 자료
- [Voronoi Diagram and Delaunay Triangulation in O(n log n) with Fortune's Algorithm - Codeforces Blog](https://codeforces.com/blog/Gay_Lord)
- [Fortune's Algorithm - Wikipedia](https://en.wikipedia.org/wiki/Fortune%27s_algorithm)
- [Delaunay triangulation - Wikipedia](https://en.wikipedia.org/wiki/Delaunay_triangulation)
- [Computational Geometry: Algorithms and Applications (de Berg et al.) - Springer](https://link.springer.com/book/10.1007/978-3-540-77974-2)
