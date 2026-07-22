---
layout: post
title: "컨벡스 헐 알고리즘 심화: Graham Scan과 Chan's Algorithm"
date: 2026-07-22
categories: [cs, computer-science]
tags: [convex-hull, graham-scan, chan-algorithm, computational-geometry, algorithm, geometry]
---

**컨벡스 헐(Convex Hull, 볼록 껍질)**은 주어진 점 집합을 모두 포함하는 가장 작은 볼록 다각형입니다. 고무줄로 모든 못을 감싸면 자연스럽게 형성되는 모양이라고 생각할 수 있습니다. 충돌 감지, 패턴 인식, GIS 시스템, 로봇공학 등 다양한 분야에서 핵심 알고리즘으로 활용됩니다.

## 개념 설명

### 볼록 집합이란?

집합 S가 볼록(convex)하다는 것은, S 내의 임의의 두 점 p, q에 대해 선분 pq 위의 모든 점도 S에 속하는 것을 의미합니다. 컨벡스 헐은 점 집합 P를 포함하는 가장 작은 볼록 집합입니다.

### 핵심 개념: 교차곱 (Cross Product)

컨벡스 헐 알고리즘의 핵심은 세 점 A, B, C가 **반시계 방향(CCW)**인지 **시계 방향(CW)**인지 판별하는 것입니다.

```
교차곱 = (B.x - A.x) * (C.y - A.y) - (B.y - A.y) * (C.x - A.x)
> 0 : 반시계 방향 (CCW)
= 0 : 일직선 (collinear)
< 0 : 시계 방향 (CW)
```

---

## Graham Scan 알고리즘

Ronald Graham이 1972년에 발표한 알고리즘으로, **O(n log n)** 시간에 컨벡스 헐을 구합니다.

### 동작 원리

1. y좌표가 가장 낮은 점(동률이면 x좌표 기준)을 기준점 P0으로 선택합니다.
2. 나머지 점들을 P0에 대한 **극각(polar angle)** 기준으로 정렬합니다.
3. 스택을 이용해 순서대로 점을 추가하면서, 왼쪽 회전(CCW)이 아닌 경우 스택에서 제거합니다.

```python
from functools import cmp_to_key

def cross_product(O, A, B):
    """양수: CCW, 음수: CW, 0: 일직선"""
    return (A[0] - O[0]) * (B[1] - O[1]) - (A[1] - O[1]) * (B[0] - O[0])

def graham_scan(points):
    n = len(points)
    if n < 3:
        return points

    # 기준점: y좌표 최소, 동률이면 x좌표 최소
    pivot = min(points, key=lambda p: (p[1], p[0]))

    def compare(a, b):
        cp = cross_product(pivot, a, b)
        if cp != 0:
            return -1 if cp > 0 else 1  # CCW 순서
        # 같은 각도면 거리 기준 정렬
        da = (a[0]-pivot[0])**2 + (a[1]-pivot[1])**2
        db = (b[0]-pivot[0])**2 + (b[1]-pivot[1])**2
        return -1 if da < db else 1

    sorted_pts = sorted(
        [p for p in points if p != pivot],
        key=cmp_to_key(compare)
    )
    sorted_pts = [pivot] + sorted_pts

    # 동일 각도 중 가장 먼 점만 남기기 (선택적)
    # 이 구현에서는 일직선 상 점도 포함

    stack = []
    for p in sorted_pts:
        # 스택의 마지막 두 점과 현재 점이 CCW 방향이 아니면 제거
        while len(stack) >= 2 and cross_product(stack[-2], stack[-1], p) <= 0:
            stack.pop()
        stack.append(p)

    return stack


# 테스트
points = [
    (0, 0), (1, 1), (2, 2), (0, 2),
    (2, 0), (1, 3), (3, 1), (0.5, 1)
]
hull = graham_scan(points)
print("Convex Hull vertices:")
for p in hull:
    print(f"  ({p[0]}, {p[1]})")

# 출력:
# Convex Hull vertices:
#   (0, 0)
#   (2, 0)
#   (3, 1)
#   (1, 3)  (근사값)
#   (0, 2)
```

Graham Scan의 핵심은 스택입니다. 점을 추가할 때 **오른쪽 회전(CW)이 발생하면** 스택에서 이전 점을 제거합니다. 이 과정이 볼록성을 유지하는 핵심 조작입니다.

---

## Chan's Algorithm: 출력 민감 최적화

Timothy Chan이 1996년에 발표한 알고리즘으로, **O(n log h)** 시간에 동작합니다. 여기서 h는 컨벡스 헐의 꼭짓점 수입니다. h가 작을수록 Graham Scan의 O(n log n)보다 훨씬 빠릅니다.

### 핵심 아이디어

1. 점 집합을 m개씩의 부분집합으로 나눕니다.
2. 각 부분집합에 Graham Scan을 적용합니다.
3. Jarvis March를 적용하되, 각 부분집합의 컨벡스 헐에서 이진 탐색으로 다음 점을 O(log m)에 찾습니다.
4. m을 잘못 추측했으면 m을 키워 반복합니다 (m = 2^(2^t), t = 1, 2, 3, ...).

```python
import math

def cross_product(O, A, B):
    return (A[0]-O[0])*(B[1]-O[1]) - (A[1]-O[1])*(B[0]-O[0])

def graham_scan_sub(points):
    """부분집합에 대한 Graham Scan — 정렬된 CCW 순서 반환"""
    if len(points) <= 1:
        return points[:]
    pivot = min(points, key=lambda p: (p[1], p[0]))
    others = sorted(
        [p for p in points if p != pivot],
        key=lambda p: (math.atan2(p[1]-pivot[1], p[0]-pivot[0]),
                       (p[0]-pivot[0])**2 + (p[1]-pivot[1])**2)
    )
    stk = [pivot]
    for p in others:
        while len(stk) >= 2 and cross_product(stk[-2], stk[-1], p) <= 0:
            stk.pop()
        stk.append(p)
    return stk

def tangent_point(hull, p):
    """
    볼록 다각형 hull에서 점 p로부터 가장 오른쪽(CW 방향) 접선을 이루는 점의 인덱스 반환.
    이진 탐색으로 O(log |hull|)에 수행.
    """
    n = len(hull)
    if n == 1:
        return 0

    def left_turn(a, b, c):
        return cross_product(a, b, c) > 0

    lo, hi = 0, n
    while lo < hi:
        mid = (lo + hi) // 2
        e1 = left_turn(p, hull[mid], hull[(mid+1) % n])
        e2 = left_turn(p, hull[(mid-1) % n], hull[mid])
        if not e1 and not e2:
            return mid
        # 탐색 범위 결정
        if e1 and left_turn(p, hull[lo % n], hull[mid]):
            hi = mid
        else:
            lo = mid + 1
    return lo % n

def chans_algorithm(points):
    """
    Chan's Algorithm: O(n log h) 컨벡스 헐
    """
    n = len(points)
    if n <= 3:
        return graham_scan_sub(points)

    # t를 증가시키며 m = min(2^(2^t), n) 시도
    for t in range(1, int(math.log2(n)) + 2):
        m = min(2 ** (2 ** t), n)

        # Step 1: 점들을 m개짜리 부분집합으로 나누어 각각 Graham Scan
        subsets = []
        for i in range(0, n, m):
            chunk = points[i:i+m]
            subsets.append(graham_scan_sub(chunk))

        # Step 2: Jarvis March with tangent search
        # 시작점: 전체 점 중 y좌표 최소 (x 최소 우선)
        start = min(points, key=lambda p: (p[1], p[0]))
        hull = [start]
        prev = (start[0] - 1, start[1])  # 왼쪽 방향 가상의 이전 점

        for _ in range(m):  # 최대 m번 반복
            best = None
            for sub in subsets:
                # 각 부분집합의 컨벡스 헐에서 접선점 찾기 (이진 탐색)
                idx = tangent_point(sub, hull[-1])
                candidate = sub[idx]
                if best is None or cross_product(hull[-1], best, candidate) < 0:
                    best = candidate

            if best == hull[0]:
                return hull  # 한 바퀴 완성
            hull.append(best)

    return graham_scan_sub(points)  # fallback


# 테스트
import random
random.seed(42)
pts = [(random.uniform(0, 100), random.uniform(0, 100)) for _ in range(100)]
hull = chans_algorithm(pts)
print(f"Chan's Algorithm: {len(pts)}개 점에서 {len(hull)}개 꼭짓점의 컨벡스 헐 계산 완료")
```

### 시간 복잡도 분석

- **Graham Scan**: O(n log n) — 정렬이 병목
- **Jarvis March**: O(nh) — h: 헐의 꼭짓점 수
- **Chan's Algorithm**: O(n log h) — h를 모르는 상태에서 최적

Chan의 알고리즘이 O(n log h)를 달성하는 핵심은 **m을 지수적으로 증가**시키는 전략입니다. m을 잘못 추측하더라도 총 비용이 O(n log h)로 수렴하는 것이 수학적으로 증명되어 있습니다.

---

## 3D 컨벡스 헐 (개념)

3D로 확장하면 **Quickhull** 알고리즘이 일반적으로 사용됩니다. 평균 O(n log n), 최악 O(n²)이며, `scipy.spatial.ConvexHull`이 내부적으로 Quickhull을 사용합니다.

```python
import numpy as np
from scipy.spatial import ConvexHull

# 3D 점들의 컨벡스 헐
rng = np.random.default_rng(42)
points_3d = rng.random((50, 3)) * 100
hull_3d = ConvexHull(points_3d)

print(f"3D 컨벡스 헐:")
print(f"  꼭짓점 수: {len(hull_3d.vertices)}")
print(f"  면(faces) 수: {len(hull_3d.simplices)}")
print(f"  표면적: {hull_3d.area:.2f}")
print(f"  부피: {hull_3d.volume:.2f}")
```

---

## 주의사항 및 팁

### 1. 수치 안정성 (Numerical Stability)

부동소수점 연산에서 교차곱이 0에 매우 가까운 값을 반환할 때 일직선 여부 판단이 불안정합니다. 정수 좌표를 사용하거나, Python의 `Fraction`이나 Java의 `BigDecimal`로 정확 산술을 적용할 수 있습니다.

### 2. 일직선 점 처리

요구사항에 따라 컨벡스 헐 경계 위에 있는 일직선 점을 포함할지 제외할지 결정해야 합니다. `cross_product == 0` 조건을 `<= 0` 또는 `< 0`으로 조정해 처리합니다.

### 3. 동적 컨벡스 헐

점이 실시간으로 추가·삭제되는 경우 **동적 컨벡스 헐(Dynamic Convex Hull)** 자료구조를 사용합니다. 균형 이진 탐색 트리 기반으로 삽입/삭제 O(log² n)에 처리 가능합니다.

### 4. 응용 분야

- **충돌 감지**: 게임 물리 엔진에서 GJK 알고리즘과 함께 사용
- **최소 경계 사각형**: 컨벡스 헐 위를 회전 캘리퍼스로 순회해 O(n)에 계산
- **직경 계산**: 가장 먼 두 점 쌍(Rotating Calipers)

컨벡스 헐은 기하학적 알고리즘의 근간이 되는 중요한 문제입니다. Graham Scan으로 기초를 다지고, Chan's Algorithm으로 출력 민감 최적화의 아름다움을 이해하는 것이 전산기하학 학습의 좋은 출발점이 됩니다.

## 참고 자료
- [Chan's Algorithm — Wikipedia](https://en.wikipedia.org/wiki/Chan%27s_algorithm)
- [Convex Hull using Graham Scan — GeeksforGeeks](https://www.geeksforgeeks.org/dsa/convex-hull-using-graham-scan/)
- [Graham Scan Convex Hull — OpenGenus IQ](https://iq.opengenus.org/graham-scan-convex-hull/)
- [Understanding Graham Scan — Muthu.co](https://muthu.co/understanding-graham-scan-algorithm-for-finding-the-convex-hull-of-a-set-of-points/)
