---
layout: post
title: "세그먼트 트리와 펜윅 트리(BIT): 구간 쿼리의 두 거장"
date: 2026-06-22
categories: [cs, computer-science]
tags: [segment-tree, fenwick-tree, BIT, range-query, data-structure, algorithm, competitive-programming]
---

"배열의 i번째부터 j번째까지 합을 구하라. 단, 중간에 원소 값이 바뀐다." 이 문제는 단순히 구현만 하면 O(N) × 쿼리 수 = 수억 번 연산이 된다. 세그먼트 트리(Segment Tree)와 펜윅 트리(Fenwick Tree, BIT)는 이를 O(log N)으로 해결하는 두 가지 핵심 자료구조다.

## 개념: 구간 쿼리 문제

**구간 쿼리(Range Query)** 란 배열의 특정 구간에 대해 합, 최솟값, 최댓값, GCD 등을 구하는 연산이다. 동시에 **점 업데이트(Point Update)** 또는 **구간 업데이트(Range Update)** 가 섞이면 단순 prefix sum으로는 대응하기 어렵다.

| 자료구조 | 구간 쿼리 | 점 업데이트 | 구간 업데이트 | 메모리 |
|--------|---------|----------|------------|------|
| 브루트포스 | O(N) | O(1) | O(N) | O(N) |
| Prefix Sum | O(1) | O(N) | O(N) | O(N) |
| **세그먼트 트리** | **O(log N)** | **O(log N)** | **O(log N)** 레이지 | **O(4N)** |
| **펜윅 트리(BIT)** | **O(log N)** | **O(log N)** | O(log²N) | **O(N)** |

---

## 세그먼트 트리 (Segment Tree)

### 원리

배열을 반씩 나눠 완전 이진 트리로 구성한다. 각 노드는 자신이 담당하는 구간의 집계값(합, 최솟값 등)을 저장한다.

```
배열: [1, 3, 5, 7, 9, 11]
인덱스:  0  1  2  3  4   5

세그먼트 트리 (구간 합):
                  [0,5] = 36
               /              \
         [0,2] = 9          [3,5] = 27
        /       \           /       \
    [0,1]=4  [2,2]=5  [3,4]=16  [5,5]=11
    /    \           /      \
[0,0]=1 [1,1]=3 [3,3]=7 [4,4]=9
```

- **쿼리**: 구간 [l, r]을 O(log N)개의 노드로 커버
- **업데이트**: 리프를 갱신하고 부모 방향으로 O(log N)개 노드 갱신

### 구현 예제 1: Python — 구간 합 세그먼트 트리

```python
class SegmentTree:
    def __init__(self, arr: list[int]):
        self.n = len(arr)
        self.tree = [0] * (4 * self.n)
        self._build(arr, 0, 0, self.n - 1)

    def _build(self, arr, node, start, end):
        if start == end:
            self.tree[node] = arr[start]
            return
        mid = (start + end) // 2
        self._build(arr, 2 * node + 1, start, mid)
        self._build(arr, 2 * node + 2, mid + 1, end)
        self.tree[node] = self.tree[2 * node + 1] + self.tree[2 * node + 2]

    def update(self, idx: int, val: int, node=0, start=0, end=None):
        if end is None:
            end = self.n - 1
        if start == end:
            self.tree[node] = val
            return
        mid = (start + end) // 2
        if idx <= mid:
            self.update(idx, val, 2 * node + 1, start, mid)
        else:
            self.update(idx, val, 2 * node + 2, mid + 1, end)
        self.tree[node] = self.tree[2 * node + 1] + self.tree[2 * node + 2]

    def query(self, l: int, r: int, node=0, start=0, end=None) -> int:
        """구간 [l, r]의 합 반환"""
        if end is None:
            end = self.n - 1
        if r < start or end < l:
            return 0  # 범위 밖
        if l <= start and end <= r:
            return self.tree[node]  # 완전히 포함
        mid = (start + end) // 2
        left_sum = self.query(l, r, 2 * node + 1, start, mid)
        right_sum = self.query(l, r, 2 * node + 2, mid + 1, end)
        return left_sum + right_sum


# 사용 예시
arr = [1, 3, 5, 7, 9, 11]
st = SegmentTree(arr)

print(st.query(1, 3))   # 3+5+7 = 15
print(st.query(0, 5))   # 1+3+5+7+9+11 = 36

st.update(2, 10)         # arr[2] = 10으로 변경
print(st.query(1, 3))   # 3+10+7 = 20
```

### 레이지 프로파게이션 (Lazy Propagation)

구간 업데이트(예: [l, r] 전체에 +5)를 O(log N)으로 처리하려면 레이지 배열을 추가한다. 노드 방문 시점까지 업데이트를 미루는 기법이다.

```python
class LazySegmentTree:
    def __init__(self, n: int):
        self.n = n
        self.tree = [0] * (4 * n)
        self.lazy = [0] * (4 * n)

    def _push_down(self, node, start, end):
        if self.lazy[node] != 0:
            mid = (start + end) // 2
            self.tree[2*node+1] += self.lazy[node] * (mid - start + 1)
            self.tree[2*node+2] += self.lazy[node] * (end - mid)
            self.lazy[2*node+1] += self.lazy[node]
            self.lazy[2*node+2] += self.lazy[node]
            self.lazy[node] = 0

    def range_update(self, l, r, val, node=0, start=0, end=None):
        if end is None:
            end = self.n - 1
        if r < start or end < l:
            return
        if l <= start and end <= r:
            self.tree[node] += val * (end - start + 1)
            self.lazy[node] += val
            return
        self._push_down(node, start, end)
        mid = (start + end) // 2
        self.range_update(l, r, val, 2*node+1, start, mid)
        self.range_update(l, r, val, 2*node+2, mid+1, end)
        self.tree[node] = self.tree[2*node+1] + self.tree[2*node+2]

    def query(self, l, r, node=0, start=0, end=None) -> int:
        if end is None:
            end = self.n - 1
        if r < start or end < l:
            return 0
        if l <= start and end <= r:
            return self.tree[node]
        self._push_down(node, start, end)
        mid = (start + end) // 2
        return (self.query(l, r, 2*node+1, start, mid) +
                self.query(l, r, 2*node+2, mid+1, end))
```

---

## 펜윅 트리 / BIT (Binary Indexed Tree)

### 원리

펜윅 트리의 핵심은 **비트 조작**으로 구간을 분해하는 것이다. 인덱스 i의 최하위 비트(Lowest Set Bit, LSB)를 이용해 각 노드가 담당하는 구간 길이를 결정한다.

```
i의 LSB: i & (-i)

i = 6 (0110₂) → LSB = 2 (0010₂) → tree[6]은 arr[5..6] 구간 담당
i = 4 (0100₂) → LSB = 4 (0100₂) → tree[4]은 arr[1..4] 구간 담당
```

- **prefix_sum(i)**: i에서 시작해 LSB를 빼면서 누적 (i -= i & (-i))
- **update(i, delta)**: i에서 시작해 LSB를 더하면서 전파 (i += i & (-i))

### 구현 예제 2: C++ — 1-indexed BIT

```cpp
#include <bits/stdc++.h>
using namespace std;

class BIT {
    int n;
    vector<long long> tree;
public:
    BIT(int n) : n(n), tree(n + 1, 0) {}

    // arr[i] += delta (1-indexed)
    void update(int i, long long delta) {
        for (; i <= n; i += i & (-i))
            tree[i] += delta;
    }

    // prefix sum [1, i]
    long long query(int i) {
        long long sum = 0;
        for (; i > 0; i -= i & (-i))
            sum += tree[i];
        return sum;
    }

    // range sum [l, r]
    long long query(int l, int r) {
        return query(r) - query(l - 1);
    }
};

int main() {
    vector<int> arr = {0, 1, 3, 5, 7, 9, 11};  // 1-indexed (0번은 더미)
    int n = 6;
    BIT bit(n);

    // 초기 배열 삽입
    for (int i = 1; i <= n; i++)
        bit.update(i, arr[i]);

    cout << bit.query(2, 4) << "\n";  // 3+5+7 = 15
    cout << bit.query(1, 6) << "\n";  // 1+3+5+7+9+11 = 36

    // arr[3] = 10 으로 변경: delta = 10 - 5 = 5
    bit.update(3, 10 - arr[3]);
    arr[3] = 10;

    cout << bit.query(2, 4) << "\n";  // 3+10+7 = 20
    return 0;
}
```

### 왜 BIT는 이렇게 작동하는가?

```
tree 배열 (1-indexed):
index:  1    2    3    4    5    6    7    8
binary: 001  010  011  100  101  110  111  1000
LSB:     1    2    1    4    1    2    1    8

tree[i]가 담당하는 구간 길이 = LSB(i)
- tree[1] → arr[1..1]  (길이 1)
- tree[2] → arr[1..2]  (길이 2)
- tree[3] → arr[3..3]  (길이 1)
- tree[4] → arr[1..4]  (길이 4)
- tree[6] → arr[5..6]  (길이 2)

prefix_sum(6) = tree[6] + tree[4]
             = (arr[5]+arr[6]) + (arr[1]+arr[2]+arr[3]+arr[4])
             = arr[1..6]  ✓
```

이렇게 하면 prefix sum 계산 시 최대 log₂(N)번의 덧셈만 필요하다.

---

## 세그먼트 트리 vs 펜윅 트리

| 기준 | 세그먼트 트리 | 펜윅 트리 |
|-----|------------|---------|
| 구현 난이도 | 보통 | 쉬움 |
| 코드 길이 | 길다 | 짧다 (10줄 내외) |
| 지원 연산 | 합, min, max, GCD, 임의 결합 법칙 | 합(역연산 가능한 연산) |
| 구간 업데이트 | O(log N) (레이지) | O(log²N) |
| 메모리 | O(4N) | O(N) |
| 캐시 효율 | 낮음 (포인터 기반) | 높음 (배열 기반) |
| 실무 사용 | 게임 서버, DB 통계 | 경쟁 프로그래밍 단골 |

**실무 선택 기준:**
- 구간 최솟값(RMQ), 구간 GCD → 세그먼트 트리
- 단순 구간 합/차 + 점 업데이트 → 펜윅 트리 (더 빠르고 간결)
- 구간 업데이트 + 구간 쿼리 → Lazy Segment Tree

---

## 응용: 2D 펜윅 트리

2D 배열에서 직사각형 구간 합을 구할 때도 BIT를 중첩해 사용할 수 있다. `tree[i][j]`로 2중 배열을 쓰며, 업데이트/쿼리 모두 O(log N × log M)이다.

```python
class BIT2D:
    def __init__(self, n, m):
        self.n, self.m = n, m
        self.tree = [[0] * (m + 1) for _ in range(n + 1)]

    def update(self, x, y, delta):
        i = x
        while i <= self.n:
            j = y
            while j <= self.m:
                self.tree[i][j] += delta
                j += j & (-j)
            i += i & (-i)

    def query(self, x, y):
        s = 0
        i = x
        while i > 0:
            j = y
            while j > 0:
                s += self.tree[i][j]
                j -= j & (-j)
            i -= i & (-i)
        return s

    def rect_query(self, x1, y1, x2, y2):
        return (self.query(x2, y2)
                - self.query(x1 - 1, y2)
                - self.query(x2, y1 - 1)
                + self.query(x1 - 1, y1 - 1))
```

---

## 주의사항 및 팁

### 1. 세그먼트 트리 크기

배열 크기 N에 대해 트리 배열은 **4N**으로 잡는 것이 안전하다. 2^⌈log₂N⌉ × 2로 계산하면 정확하지만 실수가 잦다. 대회에서는 `4 * MAXN`이 관행이다.

### 2. 펜윅 트리는 1-indexed

BIT의 `i -= i & (-i)` 루프는 i=0에서 종료하므로, 0번 인덱스를 쓰면 무한루프가 발생한다. 반드시 1-indexed로 사용하거나, 래퍼에서 인덱스를 +1 보정해야 한다.

### 3. 오버플로 주의

값의 범위가 크면(배열 원소 최대 10^9, 크기 10^5) 구간 합이 int 범위를 초과한다. C++에서는 `long long`, Python에서는 기본적으로 임의 정밀도라 문제없다.

### 4. 세그먼트 트리 최솟값은 레이지 없이도 구간 업데이트 가능

구간 최솟값은 레이지 없이도 "구간 업데이트 + 구간 최솟값 쿼리"를 Ji Driver Segment Tree(Segment Tree Beats)로 O(N log²N)에 처리할 수 있다.

---

## 마무리

세그먼트 트리와 펜윅 트리는 PS(Problem Solving)뿐만 아니라 데이터베이스 통계 집계, 실시간 랭킹 시스템, 게임 서버의 구역별 데이터 처리 등 실무에서도 핵심 패턴으로 쓰인다. 두 자료구조의 차이를 명확히 이해하고, 문제의 요구사항에 맞는 것을 선택하는 판단력이 중요하다.

## 참고 자료
- [Segment Tree - cp-algorithms.com](https://cp-algorithms.com/data_structures/segment_tree.html)
- [Fenwick Tree - cp-algorithms.com](https://cp-algorithms.com/data_structures/fenwick.html)
- [Binary Indexed Tree - GeeksforGeeks](https://www.geeksforgeeks.org/dsa/binary-indexed-tree-or-fenwick-tree-2/)
- [Understanding Fenwick Trees - Codeforces Blog](https://codeforces.com/blog/entry/57292)
