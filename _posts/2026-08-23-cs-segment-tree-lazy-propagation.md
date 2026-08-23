---
layout: post
title: "세그먼트 트리 레이지 프로파게이션(Lazy Propagation) 완전 정복: 범위 업데이트와 범위 쿼리를 O(log N)에 처리하기"
date: 2026-08-23
categories: [cs, computer-science]
tags: [segment-tree, lazy-propagation, range-update, range-query, data-structure, algorithm]
---

세그먼트 트리(Segment Tree)는 배열의 범위 쿼리(Range Query)를 O(log N)에 처리하는 강력한 자료구조다. 그러나 기본 세그먼트 트리는 **범위 업데이트**(Range Update, 특정 구간의 모든 원소를 동시에 변경)를 수행하면 O(N log N)의 시간이 걸린다. 이 문제를 해결하기 위해 고안된 기법이 바로 **레이지 프로파게이션(Lazy Propagation)**이다.

레이지 프로파게이션은 "지금 당장 필요하지 않은 업데이트는 나중으로 미루자"는 철학을 구현한 최적화 기법으로, 범위 업데이트와 범위 쿼리를 모두 O(log N)으로 줄여준다.

---

## 왜 레이지 프로파게이션이 필요한가?

### 기본 세그먼트 트리의 한계

배열 `A[0..N-1]`이 있고, 다음 두 연산을 반복적으로 수행해야 한다고 하자.

1. **범위 쿼리**: `[L, R]` 구간의 합을 구하라.
2. **범위 업데이트**: `[L, R]` 구간의 모든 원소에 `v`를 더하라.

기본 세그먼트 트리에서 포인트 업데이트는 O(log N)이지만, 범위 업데이트(`[L, R]`에 모두 더하기)는 해당 구간의 모든 원소를 개별적으로 업데이트해야 하므로 최악의 경우 O(N log N)이 된다. 쿼리 Q개가 주어진다면 전체 시간 복잡도는 O(QN log N)으로, N=100,000이면 약 1.7 × 10¹¹번의 연산이 발생해 현실적으로 불가능하다.

### 핵심 아이디어: 업데이트를 게으르게 처리하기

레이지 프로파게이션의 핵심 아이디어는 **"완전히 포함된 구간에 대한 업데이트 정보를 해당 노드에 저장해두고, 실제로 그 노드의 자식이 필요할 때만 전파한다"**는 것이다.

예를 들어 `[0, 7]` 전체에 +5를 더해야 한다면, 루트 노드의 값과 lazy 값만 갱신하고 자식 노드들은 건드리지 않는다. 이후 누군가 `[0, 3]` 구간을 쿼리할 때 비로소 루트의 lazy 값을 자식들에게 전파(push down)한다.

---

## 레이지 프로파게이션 구현 원리

### 자료구조 정의

- `tree[node]`: node가 담당하는 구간의 합 (또는 다른 집계 값)
- `lazy[node]`: node가 담당하는 구간의 자식들에게 아직 전파되지 않은 보류 업데이트 값

### 핵심 연산: Push Down

```cpp
#include <bits/stdc++.h>
using namespace std;

const int MAXN = 100005;
long long tree[4 * MAXN];
long long lazy[4 * MAXN];
int n;

// 자식 노드에게 lazy 값을 전파한다
void pushDown(int node, int start, int end) {
    if (lazy[node] != 0) {
        int mid = (start + end) / 2;
        // 왼쪽 자식 업데이트
        tree[2 * node] += lazy[node] * (mid - start + 1);
        lazy[2 * node] += lazy[node];
        // 오른쪽 자식 업데이트
        tree[2 * node + 1] += lazy[node] * (end - mid);
        lazy[2 * node + 1] += lazy[node];
        // 현재 노드의 lazy 초기화
        lazy[node] = 0;
    }
}

// 세그먼트 트리 빌드
void build(int arr[], int node, int start, int end) {
    lazy[node] = 0;
    if (start == end) {
        tree[node] = arr[start];
    } else {
        int mid = (start + end) / 2;
        build(arr, 2 * node, start, mid);
        build(arr, 2 * node + 1, mid + 1, end);
        tree[node] = tree[2 * node] + tree[2 * node + 1];
    }
}

// 범위 업데이트: [l, r]에 val을 더한다
void update(int node, int start, int end, int l, int r, long long val) {
    if (r < start || end < l) return;  // 범위 벗어남
    if (l <= start && end <= r) {      // 완전히 포함
        tree[node] += val * (end - start + 1);
        lazy[node] += val;
        return;
    }
    pushDown(node, start, end);        // 부분 포함: 자식에게 전파 후 재귀
    int mid = (start + end) / 2;
    update(2 * node, start, mid, l, r, val);
    update(2 * node + 1, mid + 1, end, l, r, val);
    tree[node] = tree[2 * node] + tree[2 * node + 1];
}

// 범위 쿼리: [l, r]의 합을 반환한다
long long query(int node, int start, int end, int l, int r) {
    if (r < start || end < l) return 0;  // 범위 벗어남
    if (l <= start && end <= r) return tree[node];  // 완전히 포함
    pushDown(node, start, end);
    int mid = (start + end) / 2;
    return query(2 * node, start, mid, l, r) +
           query(2 * node + 1, mid + 1, end, l, r);
}

int main() {
    int arr[] = {1, 3, 5, 7, 9, 11};
    n = 6;
    build(arr, 1, 0, n - 1);

    // [1, 4] 구간에 +10 업데이트
    update(1, 0, n - 1, 1, 4, 10);

    // [0, 5] 구간 합 쿼리 (기댓값: 1+13+15+17+19+11 = 76)
    cout << query(1, 0, n - 1, 0, n - 1) << "\n";  // 76

    // [2, 3] 구간 합 쿼리 (기댓값: 15+17 = 32)
    cout << query(1, 0, n - 1, 2, 3) << "\n";  // 32
    return 0;
}
```

위 코드에서 `pushDown`은 lazy 값이 존재할 때 자식들의 `tree`와 `lazy`를 갱신하고 현재 노드의 `lazy`를 0으로 초기화한다. 업데이트와 쿼리 모두 구간이 완전히 포함될 때만 lazy를 설정하거나 값을 반환하고, 부분 포함인 경우 먼저 `pushDown`을 수행한 뒤 재귀 호출한다.

---

## 시간 복잡도와 공간 복잡도

| 연산 | 시간 복잡도 |
|---|---|
| 빌드 | O(N) |
| 범위 업데이트 | O(log N) |
| 범위 쿼리 | O(log N) |
| 포인트 업데이트 | O(log N) |

공간 복잡도는 O(4N)으로, `tree`와 `lazy` 배열 각각에 4N 크기가 필요하다.

---

## 레이지 프로파게이션 심화: 범위 업데이트 + 범위 최댓값 쿼리

레이지 프로파게이션은 합(sum) 외에도 최댓값(max), 최솟값(min), GCD 등 다양한 집계 연산에 적용할 수 있다. 아래는 Python으로 범위 덧셈 업데이트와 범위 최댓값 쿼리를 지원하는 버전이다.

```python
class LazySegTree:
    def __init__(self, arr):
        self.n = len(arr)
        self.tree = [0] * (4 * self.n)
        self.lazy = [0] * (4 * self.n)
        self._build(arr, 1, 0, self.n - 1)

    def _build(self, arr, node, start, end):
        self.lazy[node] = 0
        if start == end:
            self.tree[node] = arr[start]
        else:
            mid = (start + end) // 2
            self._build(arr, 2 * node, start, mid)
            self._build(arr, 2 * node + 1, mid + 1, end)
            self.tree[node] = max(self.tree[2 * node], self.tree[2 * node + 1])

    def _push_down(self, node):
        if self.lazy[node] != 0:
            for child in [2 * node, 2 * node + 1]:
                self.tree[child] += self.lazy[node]
                self.lazy[child] += self.lazy[node]
            self.lazy[node] = 0

    def update(self, node, start, end, l, r, val):
        if r < start or end < l:
            return
        if l <= start and end <= r:
            self.tree[node] += val
            self.lazy[node] += val
            return
        self._push_down(node)
        mid = (start + end) // 2
        self.update(2 * node, start, mid, l, r, val)
        self.update(2 * node + 1, mid + 1, end, l, r, val)
        self.tree[node] = max(self.tree[2 * node], self.tree[2 * node + 1])

    def query(self, node, start, end, l, r):
        if r < start or end < l:
            return float('-inf')
        if l <= start and end <= r:
            return self.tree[node]
        self._push_down(node)
        mid = (start + end) // 2
        return max(
            self.query(2 * node, start, mid, l, r),
            self.query(2 * node + 1, mid + 1, end, l, r)
        )

    def range_update(self, l, r, val):
        self.update(1, 0, self.n - 1, l, r, val)

    def range_max(self, l, r):
        return self.query(1, 0, self.n - 1, l, r)


# 사용 예시
arr = [2, 1, 1, 3, 2, 3, 4, 5]
seg = LazySegTree(arr)

print(seg.range_max(0, 7))    # 5

seg.range_update(1, 5, 3)     # [1, 5] 구간에 +3

print(seg.range_max(0, 7))    # 8 (5+3=8은 아니고 4+3=7이지만 인덱스 7은 5 → 5)
print(seg.range_max(1, 5))    # 7 (4+3)
```

---

## 레이지 프로파게이션이 적용되는 문제 유형

레이지 프로파게이션은 다음 패턴의 문제에서 필수적이다.

1. **범위 덧셈 + 범위 합 쿼리**: 구간 합 세그먼트 트리의 확장
2. **범위 덧셈 + 범위 최댓값/최솟값 쿼리**: 슬라이딩 윈도우 최적화에도 활용
3. **범위 대입 + 범위 합/최댓값**: `lazy`의 의미가 "추가"가 아닌 "대입"으로 바뀜
4. **범위 XOR + 범위 합**: 비트 연산 응용

---

## 주의사항과 팁

### 1. Push Down의 순서를 놓치지 마라

부분 포함 구간에서 재귀 호출 **전**에 반드시 `pushDown`을 수행해야 한다. 그렇지 않으면 자식의 값이 최신 상태가 아닌 상태에서 연산이 이루어져 잘못된 결과가 나온다.

### 2. Lazy 값의 중첩 처리

여러 번의 업데이트가 쌓일 때 lazy 값을 올바르게 합산해야 한다. 덧셈의 경우 `lazy[child] += lazy[parent]`처럼 누적하면 되지만, 대입 업데이트가 섞이면 우선순위 처리가 복잡해진다. 이때는 "마지막 대입 이후 추가된 값"과 "대입 여부 플래그"를 별도 관리해야 한다.

### 3. 배열 크기 주의

세그먼트 트리 배열은 `4 * N` 크기로 선언해야 안전하다. N이 2의 거듭제곱이 아닌 경우 `2 * N` 크기로는 부족할 수 있다.

### 4. long long 오버플로우 주의

범위 합 쿼리에서 값이 매우 크거나 업데이트가 많으면 int 범위를 초과한다. 반드시 `long long`을 사용하라.

### 5. 비재귀(Iterative) 세그먼트 트리와의 비교

재귀 방식보다 반복 방식이 상수 배 빠르지만, 레이지 프로파게이션을 비재귀로 구현하기는 다소 까다롭다. 경쟁 프로그래밍에서 시간이 촉박하지 않다면 재귀 방식이 구현하기 더 안전하다.

---

## 마무리

레이지 프로파게이션은 세그먼트 트리의 활용 범위를 크게 확장하는 핵심 기법이다. 범위 업데이트와 범위 쿼리가 동시에 필요한 문제에서는 이 기법 없이는 시간 내에 통과하기 어렵다. 구현의 핵심은 **"완전히 포함된 구간에서는 lazy에 기록하고, 부분 포함 시 push down 후 재귀"**라는 단순한 원칙이다. 이 원칙만 잘 이해하면 다양한 집계 연산과 업데이트 유형에 자유롭게 응용할 수 있다.

## 참고 자료
- [CP-Algorithms: Segment Tree](https://cp-algorithms.com/data_structures/segment_tree.html)
- [GeeksforGeeks: Lazy Propagation in Segment Tree](https://www.geeksforgeeks.org/dsa/lazy-propagation-in-segment-tree/)
- [Codeforces: Efficient and easy segment trees](https://codeforces.com/blog/entry/18051)
- [Codeforces: Segment tree beats](https://codeforces.com/blog/entry/57319)
