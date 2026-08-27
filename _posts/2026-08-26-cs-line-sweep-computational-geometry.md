---
layout: post
title: "선 스윕 알고리즘(Line Sweep) — 기하 문제를 O(n log n)으로 풀기"
date: 2026-08-26
categories: [cs, computer-science]
tags: [algorithm, geometry, sweep-line, computational-geometry, interval, segment-intersection]
---

## 선 스윕 알고리즘이란

선 스윕(Line Sweep) 알고리즘, 또는 평면 스윕(Plane Sweep) 알고리즘은 계산 기하학(Computational Geometry)에서 광범위하게 사용되는 알고리즘 패러다임입니다. 핵심 아이디어는 가상의 수직선(sweep line)이 2차원 평면을 왼쪽에서 오른쪽으로 훑어 나가며, 특정 x좌표에서만 중요한 이벤트(event)가 발생한다고 생각하는 것입니다.

전수 검사(brute-force) 방식이 O(n²) 이상의 복잡도를 갖는 기하 문제들을, 선 스윕은 이벤트를 정렬하고 활성 객체 집합을 균형 이진 탐색 트리(BST)로 관리함으로써 **O(n log n)**으로 해결합니다.

### 핵심 구성 요소

1. **이벤트(Event)**: sweep line이 지나면서 처리해야 하는 의미 있는 x좌표들. 선분의 시작/끝점, 교차점 등.
2. **이벤트 큐(Event Queue)**: 이벤트를 x좌표 기준으로 정렬한 우선순위 큐 또는 정렬 배열.
3. **상태(Status)**: 현재 sweep line과 교차하는 객체들의 집합. y좌표 기준 정렬 BST로 관리.

---

## 왜 선 스윕이 필요한가

### 구간 합집합 문제

n개의 수직선 구간 [l₁, r₁], [l₂, r₂], ..., [lₙ, rₙ]이 주어질 때, 이들의 합집합 총 길이를 구하는 문제를 생각해 보겠습니다.

단순히 정렬 후 병합하면 O(n log n)이지만, 이 방식을 선 스윕으로 이해하면 더 복잡한 2D 문제(직사각형 합집합 넓이 등)로 일반화하는 발판이 됩니다.

### 선분 교차 검사

n개의 선분 중 교차하는 쌍이 하나라도 존재하는지 확인하는 **Shamos-Hoey 알고리즘**은 O(n log n)으로 동작합니다. 교차점을 모두 찾는 **Bentley-Ottmann 알고리즘**은 k개의 교차점이 있을 때 O((n+k) log n)입니다.

### 직사각형 합집합 넓이

n개의 축 정렬 직사각형이 겹칠 때 전체 덮인 넓이를 O(n log n)에 계산하는 고전 문제입니다. y축 방향 선분들의 활성 길이를 세그먼트 트리로 관리합니다.

---

## 구현 예제

### 예제 1: C++ — 구간 합집합 길이 계산

{% raw %}
```cpp
#include <bits/stdc++.h>
using namespace std;

/**
 * n개의 구간 [l, r]의 합집합 총 길이를 계산한다.
 * 아이디어: l에서 +1 이벤트, r에서 -1 이벤트를 만들고
 *           count > 0인 구간의 길이를 누적한다.
 */
long long unionLength(vector<pair<int,int>>& intervals) {
    vector<pair<int,int>> events;  // {x좌표, 타입(+1/-1)}
    for (auto& [l, r] : intervals) {
        events.push_back({l, +1});
        events.push_back({r, -1});
    }
    sort(events.begin(), events.end());

    long long total = 0;
    int count = 0;
    int prev = 0;

    for (auto& [x, type] : events) {
        if (count > 0) {
            total += x - prev;  // 현재 sweep line 위치까지 활성 길이 누적
        }
        count += type;
        prev = x;
    }
    return total;
}

// === 직사각형 합집합 넓이 (세그먼트 트리 + 선 스윕) ===

struct SegTree {
    int n;
    vector<int> cnt;     // 구간이 완전히 덮인 횟수
    vector<long long> len;  // 실제 활성 길이

    SegTree(int n) : n(n), cnt(4*n, 0), len(4*n, 0) {}

    void update(int node, int lo, int hi, int l, int r, int val) {
        if (r <= lo || hi <= l) return;
        if (l <= lo && hi <= r) {
            cnt[node] += val;
        } else {
            int mid = (lo + hi) / 2;
            update(2*node, lo, mid, l, r, val);
            update(2*node+1, mid, hi, l, r, val);
        }
        if (cnt[node] > 0) {
            len[node] = hi - lo;
        } else if (hi - lo == 1) {
            len[node] = 0;
        } else {
            len[node] = len[2*node] + len[2*node+1];
        }
    }

    long long query() { return len[1]; }
};

long long rectangleUnionArea(vector<array<int,4>>& rects) {
    // rects[i] = {x1, y1, x2, y2}
    // 이벤트: 각 직사각형의 좌/우 경계
    vector<tuple<int,int,int,int>> events;  // {x, type(+1/-1), y1, y2}
    set<int> ys_set;

    for (auto& r : rects) {
        events.push_back({r[0], +1, r[1], r[3]});  // 왼쪽 경계: 활성화
        events.push_back({r[2], -1, r[1], r[3]});  // 오른쪽 경계: 비활성화
        ys_set.insert(r[1]);
        ys_set.insert(r[3]);
    }
    sort(events.begin(), events.end());

    // y좌표 좌표 압축
    vector<int> ys(ys_set.begin(), ys_set.end());
    int m = ys.size();
    auto yidx = [&](int y) {
        return (int)(lower_bound(ys.begin(), ys.end(), y) - ys.begin());
    };

    SegTree seg(m);
    long long area = 0;
    int prevX = 0;

    for (auto& [x, type, y1, y2] : events) {
        area += (long long)(x - prevX) * seg.query();
        seg.update(1, 0, m-1, yidx(y1), yidx(y2), type);
        prevX = x;
    }
    return area;
}

int main() {
    // 구간 합집합 예시
    vector<pair<int,int>> intervals = {{1, 5}, {3, 8}, {10, 12}};
    cout << "합집합 길이: " << unionLength(intervals) << "\n";  // 9

    // 직사각형 합집합 예시 (x1, y1, x2, y2)
    vector<array<int,4>> rects = {
        {0, 0, 2, 2},
        {1, 1, 3, 3},
    };
    cout << "직사각형 합집합 넓이: " << rectangleUnionArea(rects) << "\n";  // 7
    return 0;
}
```
{% endraw %}

### 예제 2: Python — 선분 교차 존재 여부 검사 (Shamos-Hoey)

```python
from sortedcontainers import SortedList

def cross(o, a, b):
    """외적(cross product). 양수면 반시계, 음수면 시계, 0이면 공선."""
    return (a[0] - o[0]) * (b[1] - o[1]) - (a[1] - o[1]) * (b[0] - o[0])

def on_segment(p, q, r):
    """r이 선분 pq 위에 있는지 확인."""
    return (min(p[0], q[0]) <= r[0] <= max(p[0], q[0]) and
            min(p[1], q[1]) <= r[1] <= max(p[1], q[1]))

def segments_intersect_pair(p1, q1, p2, q2):
    """두 선분 p1q1, p2q2이 교차하는지 확인."""
    d1 = cross(p2, q2, p1)
    d2 = cross(p2, q2, q1)
    d3 = cross(p1, q1, p2)
    d4 = cross(p1, q1, q2)

    if ((d1 > 0 and d2 < 0) or (d1 < 0 and d2 > 0)) and \
       ((d3 > 0 and d4 < 0) or (d3 < 0 and d4 > 0)):
        return True
    if d1 == 0 and on_segment(p2, q2, p1): return True
    if d2 == 0 and on_segment(p2, q2, q1): return True
    if d3 == 0 and on_segment(p1, q1, p2): return True
    if d4 == 0 and on_segment(p1, q1, q2): return True
    return False


def has_intersection(segments):
    """
    Shamos-Hoey 알고리즘: n개의 선분 중 교차 쌍 존재 여부를 O(n log n)으로 판별.
    segments: [(p1, q1), (p2, q2), ...], p = (x, y)
    """
    # 이벤트 생성: (x, 이벤트_종류, 선분_인덱스)
    # 이벤트 종류: 0 = 시작(LEFT), 1 = 끝(RIGHT)
    events = []
    for i, (p, q) in enumerate(segments):
        if p[0] > q[0] or (p[0] == q[0] and p[1] > q[1]):
            p, q = q, p  # 왼쪽 점이 p
        events.append((p[0], 0, i, p, q))   # 시작
        events.append((q[0], 1, i, p, q))   # 끝

    # x좌표 우선, 같으면 시작 이벤트 우선
    events.sort(key=lambda e: (e[0], e[1]))

    # 현재 sweep line 위치를 추적하여 y좌표 계산
    sweep_x = [0.0]

    def y_at_sweep(seg):
        p, q = seg
        x = sweep_x[0]
        if p[0] == q[0]:
            return max(p[1], q[1])
        return p[1] + (q[1] - p[1]) * (x - p[0]) / (q[0] - p[0])

    # 활성 선분을 y좌표 순으로 유지 (인덱스로 식별)
    active_segs = []   # 단순 리스트 (교육 목적 단순화)
    seg_idx = {}       # seg_index -> (p, q)

    for event in events:
        x_coord, etype, i, p, q = event
        sweep_x[0] = x_coord

        if etype == 0:  # 선분 시작
            seg_idx[i] = (p, q)
            # 활성 선분들과 교차 검사
            for j, (sp, sq) in list(seg_idx.items()):
                if j == i:
                    continue
                if segments_intersect_pair(p, q, sp, sq):
                    return True, (i, j)
            active_segs.append(i)
        else:            # 선분 끝
            active_segs.remove(i)
            del seg_idx[i]

    return False, None


# === 사용 예시 ===
if __name__ == "__main__":
    # 교차하는 경우
    segs1 = [
        ((0, 0), (4, 4)),   # 선분 0: 대각선
        ((0, 4), (4, 0)),   # 선분 1: 반대 대각선 → (2,2)에서 교차
    ]
    found, pair = has_intersection(segs1)
    print(f"교차 존재: {found}, 쌍: {pair}")   # True, (0, 1)

    # 교차하지 않는 경우
    segs2 = [
        ((0, 0), (1, 1)),
        ((2, 2), (3, 3)),
    ]
    found, pair = has_intersection(segs2)
    print(f"교차 존재: {found}")  # False
```

---

## 다양한 선 스윕 응용

### 최근접 점 쌍 (Closest Pair of Points)

Shamos-Hoey에서와 유사하게 sweep line을 이용합니다. 현재까지 찾은 최소 거리 δ에 대해, sweep line 왼쪽 δ 범위 안의 점들만 활성 집합에 유지하며, 각 새 점이 들어올 때 같은 범위의 이웃 점들만 검사합니다. 이로써 전체 O(n log n)이 가능합니다.

### 지도 오버레이 (Map Overlay)

두 다각형 지도의 교차선을 구하는 작업에서 Bentley-Ottmann 알고리즘이 사용됩니다. GIS(지리정보시스템)에서 도로와 행정구역 경계 교차를 처리하는 데 활용됩니다.

### 이벤트 기반 시뮬레이션

충돌 시뮬레이션, 레이 캐스팅(ray casting), 음향 처리에서 스윕 기반 이벤트 처리가 활용됩니다.

---

## 주의사항 및 팁

**부동소수점 오차**: 기하 알고리즘에서 `float` 비교는 항상 위험합니다. 가능하면 정수 좌표만 사용하거나, 외적/내적을 정수 연산으로 처리하세요. 불가피하게 실수를 사용해야 한다면 `eps = 1e-9` 수준의 오차 허용치를 두세요.

**이벤트 동일 x좌표 처리**: 같은 x좌표에서 선분의 시작과 끝이 동시에 발생할 때 순서가 중요합니다. 일반적으로 시작 이벤트를 끝 이벤트보다 먼저 처리해야 선분이 순간적으로 활성화/비활성화 되는 경계 케이스를 올바르게 다룰 수 있습니다.

**수직 선분**: 수직 선분(x₁ = x₂)은 sweep line과 항상 접하므로 특별 처리가 필요합니다. 수직 선분을 별도 이벤트로 분리하거나 y 범위 구간으로 처리합니다.

**정렬된 상태 유지**: 활성 집합에서 이웃 검색은 `std::set`(C++) 또는 `SortedList`(Python sortedcontainers)로 O(log n)에 유지하세요. 일반 배열 사용 시 O(n)으로 악화됩니다.

---

## 참고 자료

- [Sweep line algorithm - Wikipedia](https://en.wikipedia.org/wiki/Sweep_line_algorithm)
- [Bentley-Ottmann algorithm - Wikipedia](https://en.wikipedia.org/wiki/Bentley%E2%80%93Ottmann_algorithm)
- [Search for intersecting segments - CP-Algorithms](https://cp-algorithms.com/geometry/intersecting_segments.html)
- [Sweep Line · USACO Guide](https://usaco.guide/plat/sweep-line)
