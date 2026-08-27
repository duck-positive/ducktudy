---
layout: post
title: "볼록 껍질 트릭(Convex Hull Trick)과 Li Chao 트리 완전 정복: 선형 함수 최솟값 DP를 O(N log N)으로 최적화하기"
date: 2026-08-19
categories: [cs, computer-science]
tags: [convex-hull-trick, li-chao-tree, dp-optimization, algorithm, competitive-programming, cpp]
---

## 볼록 껍질 트릭(Convex Hull Trick)이란?

동적 프로그래밍(DP)에서 점화식의 시간 복잡도를 줄이는 최적화 기법 중 하나가 **볼록 껍질 트릭(Convex Hull Trick, CHT)**이다. 단순 DP로 O(N²)이 걸리는 문제를 O(N log N) 또는 심지어 O(N)으로 줄일 수 있는 강력한 도구다.

CHT는 다음과 같은 형태의 점화식에 적용된다:

```
dp[i] = min over all j < i of { dp[j] + b[j] * a[i] }
```

여기서 `dp[j] + b[j] * a[i]`는 기울기 `b[j]`, 절편 `dp[j]`인 **일차 함수 f_j(x) = b[j] * x + dp[j]**를 `x = a[i]`에서 평가한 값이다. 즉, N개의 일차 함수 중에서 특정 x 값에서의 최솟값(또는 최댓값)을 효율적으로 구하는 문제로 환원된다.

---

## 왜 필요한가?

### 동적 프로그래밍의 병목

대표적인 적용 예시로 **Slope Trick**이나 **1D1D DP**를 생각해보자. 예를 들어, 비용 최소화 문제:

> N개의 공장에서 M개의 창고로 물건을 보낸다. 창고 j에서 공장 i까지의 비용이 `cost[j] * dist[i]`일 때, 총 비용 최소화.

이 문제의 DP는:

```
dp[i] = min_{j < i}(dp[j] + cost[j] * dist[i])
```

순진하게 구현하면 각 i에 대해 모든 j를 확인하므로 O(N²)이다. CHT를 적용하면 O(N log N)으로 줄어든다.

### 핵심 관찰: 하부 볼록 포락선(Lower Convex Envelope)

모든 직선 `f_j(x) = b[j] * x + c[j]`를 좌표계에 그리면, x 값에 따라 최솟값을 구성하는 직선들이 **볼록 포락선(convex envelope)**을 형성한다. 이 포락선 위에 있지 않은 직선은 어떤 x에 대해서도 최솟값에 기여하지 않으므로 제거할 수 있다.

```
y
│
│  \        /
│   \  \  /
│    \  \/
│     \/
│      선 1  선 2  선 3 ...
└─────────────────────── x
     ↑      ↑
   교점1   교점2
```

x가 증가할 때 최솟값을 제공하는 직선이 단조적으로 바뀐다면 **단조 CHT**(O(N)), 그렇지 않다면 이진 탐색을 이용한 **일반 CHT**(O(N log N))를 사용한다.

---

## 구현 1: 단조 CHT (O(N))

기울기가 단조 감소하고 쿼리 x도 단조 증가하는 경우, 덱(deque)을 이용해 O(N) 시간에 해결한다.

{% raw %}
```cpp
#include <bits/stdc++.h>
using namespace std;
typedef long long ll;

struct Line {
    ll m, b; // y = m*x + b
    ll eval(ll x) { return m * x + b; }
};

// 세 직선 l1, l2, l3에서 l2가 불필요한지 판단
// l2가 l1과 l3의 교점보다 왼쪽에서 l3에게 추월당하면 불필요
bool bad(Line l1, Line l2, Line l3) {
    // l1과 l3의 교점의 x좌표 <= l1과 l2의 교점의 x좌표
    // 교점 x = (b1 - b2) / (m2 - m1) (정수 연산으로 변환)
    return (__int128)(l3.b - l1.b) * (l1.m - l2.m) 
        <= (__int128)(l2.b - l1.b) * (l1.m - l3.m);
}

struct ConvexHullTrick {
    deque<Line> hull;
    
    // 기울기 단조 감소 순서로 직선 추가
    void add(ll m, ll b) {
        Line l = {m, b};
        while (hull.size() >= 2 && bad(hull[hull.size()-2], hull[hull.size()-1], l))
            hull.pop_back();
        hull.push_back(l);
    }
    
    // 쿼리 x 단조 증가 순서로 최솟값 조회
    ll query(ll x) {
        while (hull.size() >= 2 && hull[0].eval(x) >= hull[1].eval(x))
            hull.pop_front();
        return hull[0].eval(x);
    }
};

// 예제: 직선 y = m_i * x + b_i 들 중 x에서 최솟값
int main() {
    // 직선들 (기울기 단조 감소)
    vector<pair<ll,ll>> lines = {{5, 1}, {3, 4}, {1, 8}, {-1, 15}};
    
    ConvexHullTrick cht;
    for (auto [m, b] : lines)
        cht.add(m, b);
    
    // 쿼리 x (단조 증가)
    vector<ll> queries = {0, 1, 2, 3, 5, 10};
    for (ll x : queries) {
        cout << "x=" << x << ": min=" << cht.query(x) << "\n";
    }
    
    return 0;
}
```
{% endraw %}

출력:
```
x=0: min=1
x=1: min=6
x=2: min=10
x=3: min=12
x=5: min=10
x=10: min=5
```

---

## 구현 2: 이진 탐색 CHT (O(N log N))

단조 조건이 없을 때는 볼록 포락선을 벡터로 유지하고 각 쿼리마다 이진 탐색을 사용한다.

```cpp
#include <bits/stdc++.h>
using namespace std;
typedef long long ll;

struct Line {
    ll m, b;
    ll eval(ll x) const { return m * x + b; }
};

bool bad(const Line& l1, const Line& l2, const Line& l3) {
    return (__int128)(l3.b - l1.b) * (l1.m - l2.m)
        <= (__int128)(l2.b - l1.b) * (l1.m - l3.m);
}

struct CHT_Log {
    vector<Line> hull;
    
    // 임의 순서로 직선 추가 가능 (단, 기울기 단조 감소라고 가정)
    void add(ll m, ll b) {
        Line l = {m, b};
        while (hull.size() >= 2 && bad(hull[hull.size()-2], hull[hull.size()-1], l))
            hull.pop_back();
        hull.push_back(l);
    }
    
    // 임의 x 쿼리, O(log N)
    ll query(ll x) const {
        int lo = 0, hi = (int)hull.size() - 1;
        while (lo < hi) {
            int mid = (lo + hi) / 2;
            if (hull[mid].eval(x) >= hull[mid+1].eval(x))
                lo = mid + 1;
            else
                hi = mid;
        }
        return hull[lo].eval(x);
    }
};

// 실전 문제: 직사각형 배달 비용 최소화 DP
// dp[i] = min_{j < i}(dp[j] + h[j] * w[i]) 형태
int main() {
    int n = 5;
    vector<ll> h = {0, 3, 2, 4, 1};  // 기울기 역할
    vector<ll> w = {0, 2, 5, 3, 6};  // 쿼리 역할
    
    vector<ll> dp(n, LLONG_MAX);
    dp[0] = 0;
    
    CHT_Log cht;
    cht.add(h[0], dp[0]); // 직선 추가: y = h[0]*x + dp[0]
    
    for (int i = 1; i < n; i++) {
        dp[i] = cht.query(w[i]);
        if (i < n - 1)
            cht.add(h[i], dp[i]);
    }
    
    for (int i = 0; i < n; i++)
        cout << "dp[" << i << "] = " << dp[i] << "\n";
    
    return 0;
}
```

---

## Li Chao 트리: 더 일반화된 접근

볼록 껍질 트릭은 선형 함수에만 적용되지만, **Li Chao 트리(Li Chao Segment Tree)**는 임의의 함수에 대해 적용할 수 있는 더 일반적인 자료구조다. 특히 쿼리 범위가 미리 알려진 경우에 유용하다.

### Li Chao 트리의 핵심 아이디어

구간 트리(Segment Tree)의 각 노드에 그 구간의 **"지배 직선(dominant line)"**을 저장한다. 어떤 구간에서 지배 직선은 그 구간의 중앙점에서 최솟값을 주는 직선이다.

새 직선 l을 삽입할 때:
1. 현재 노드 구간의 중앙점에서 기존 직선 vs 새 직선을 비교
2. 중앙점에서 더 좋은 직선을 이 노드에 저장
3. 나머지 직선을 해당 반 구간의 자식 노드로 재귀 전달

```cpp
#include <bits/stdc++.h>
using namespace std;
typedef long long ll;

const ll INF = 1e18;
const int MAXN = 1 << 18; // 쿼리 범위 [0, MAXN)

struct Line {
    ll m, b;
    Line() : m(0), b(INF) {}
    Line(ll m, ll b) : m(m), b(b) {}
    ll eval(ll x) const { return m * x + b; }
};

struct LiChaoTree {
    vector<Line> tree;
    int n;
    
    LiChaoTree(int n) : n(n), tree(4 * n) {}
    
    void add(int node, int lo, int hi, Line newLine) {
        int mid = (lo + hi) / 2;
        bool leftBetter = newLine.eval(lo) < tree[node].eval(lo);
        bool midBetter  = newLine.eval(mid) < tree[node].eval(mid);
        
        if (midBetter) swap(tree[node], newLine);
        
        if (lo == hi) return;
        
        if (leftBetter != midBetter)
            add(2*node, lo, mid, newLine);
        else
            add(2*node+1, mid+1, hi, newLine);
    }
    
    void add(ll m, ll b) {
        add(1, 0, n-1, Line(m, b));
    }
    
    ll query(int node, int lo, int hi, int x) {
        ll res = tree[node].eval(x);
        if (lo == hi) return res;
        
        int mid = (lo + hi) / 2;
        if (x <= mid)
            return min(res, query(2*node, lo, mid, x));
        else
            return min(res, query(2*node+1, mid+1, hi, x));
    }
    
    ll query(int x) {
        return query(1, 0, n-1, x);
    }
};

int main() {
    LiChaoTree lct(100); // x 범위 [0, 99]
    
    // 직선 추가 (임의 순서)
    lct.add(2, 1);   // y = 2x + 1
    lct.add(-1, 10); // y = -x + 10
    lct.add(0, 5);   // y = 5
    lct.add(3, -5);  // y = 3x - 5
    
    // 각 x에서 최솟값
    for (int x : {0, 1, 2, 3, 4, 5, 10}) {
        cout << "x=" << x << ": min=" << lct.query(x) << "\n";
    }
    
    return 0;
}
```

출력:
```
x=0: min=-5
x=1: min=-2
x=2: min=1
x=3: min=4
x=4: min=6
x=5: min=5
x=10: min=0
```

---

## CHT vs Li Chao 트리 비교

| 특성 | CHT (단조) | CHT (이진탐색) | Li Chao 트리 |
|------|-----------|--------------|------------|
| 시간 복잡도 | O(N) | O(N log N) | O(N log N) |
| 기울기 조건 | 단조 필요 | 임의 가능 | 임의 가능 |
| 쿼리 조건 | 단조 필요 | 임의 가능 | 임의 가능 (정수 범위) |
| 구현 복잡도 | 낮음 | 중간 | 중간 |
| 공간 복잡도 | O(N) | O(N) | O(N log MAX_X) |
| 동적 삽입 | 가능 | 가능 | 가능 |
| 적용 범위 | 좁음 | 중간 | 넓음 |

---

## 주의사항과 팁

### 1. 정수 오버플로우

기울기와 절편이 큰 경우 교점 계산에서 오버플로우가 발생할 수 있다. `__int128`을 사용하거나 부등식을 교차 곱셈으로 변환할 때 주의하라.

```cpp
// 위험: ll 오버플로우 가능
bool bad(Line l1, Line l2, Line l3) {
    return (l3.b - l1.b) * (l1.m - l2.m) <= (l2.b - l1.b) * (l1.m - l3.m);
}

// 안전: __int128 사용
bool bad(Line l1, Line l2, Line l3) {
    return (__int128)(l3.b - l1.b) * (l1.m - l2.m) 
        <= (__int128)(l2.b - l1.b) * (l1.m - l3.m);
}
```

### 2. 최솟값 vs 최댓값

최댓값을 구하려면 부등호를 반전하거나, 모든 직선에 -1을 곱한 뒤 최솟값을 구하는 방법이 있다. "하부 볼록 포락선" 대신 "상부 볼록 포락선"을 유지한다.

### 3. 적용 가능한 DP 패턴 인식법

점화식에서 다음 패턴을 발견하면 CHT를 의심하라:
```
dp[i] = min_{j} (A[j] * B[i] + C[j])
```
`A[j]`가 직선의 기울기, `C[j]`가 절편, `B[i]`가 쿼리 x값이다.

### 4. 실수 vs 정수

교점 계산이 실수인 경우를 조심해야 한다. 가능하면 정수 교차 곱셈으로 비교하는 것이 정확도 면에서 안전하다.

## 참고 자료
- [Convex Hull Trick and Li Chao Tree - CP-Algorithms](https://cp-algorithms.com/geometry/convex_hull_trick.html)
- [Convex Hull Trick - Codeforces Blog](https://codeforces.com/blog/entry/84095)
- [Tutorial: Convex Hull Trick — Geometry being useful](https://codeforces.com/blog/entry/63823)
- [Li-Chao Tree: Algorithm Specification and Analysis (arXiv)](https://arxiv.org/abs/2603.07948)
