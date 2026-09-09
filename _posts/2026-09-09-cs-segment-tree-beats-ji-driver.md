---
layout: post
title: "세그먼트 트리 Beats (Ji Driver Segmentation) 완전 정복: 구간 chmin/chmax와 구간 합을 동시에 처리하는 고급 기법"
date: 2026-09-09
categories: [cs, computer-science]
tags: [segment-tree, segment-tree-beats, ji-driver, range-query, data-structure, competitive-programming]
---

## 개념 설명

**세그먼트 트리 Beats**는 2016년 Ji (吉如一)가 논문 "Chtholly Tree and Segment Tree Beats"에서 발표한 고급 세그먼트 트리 기법이다. 흔히 "Ji Driver Segmentation"이라고도 불린다. 이 기법은 기존 Lazy Propagation 세그먼트 트리로는 효율적으로 처리할 수 없었던, **구간 최솟값 갱신(range chmin)과 구간 합 쿼리를 O((n + q) log² n) 시간에 처리**하는 획기적인 방법이다.

### 문제 설정

다음 연산들을 O(log² n) 또는 O(log n) 시간에 처리하고 싶다:

1. `range_chmin(l, r, v)`: 구간 [l, r]의 모든 원소를 min(원소, v)로 갱신
2. `range_chmax(l, r, v)`: 구간 [l, r]의 모든 원소를 max(원소, v)로 갱신  
3. `range_add(l, r, v)`: 구간 [l, r]의 모든 원소에 v 더하기
4. `range_sum(l, r)`: 구간 [l, r]의 합 반환
5. `range_max(l, r)`: 구간 [l, r]의 최댓값 반환

문제는 `range_chmin`이 비선형 연산이라는 점이다. 일반 Lazy Propagation은 "모든 원소에 동일한 변환을 적용"할 때만 효율적이며, 각 원소가 다른 값으로 클램프되는 경우를 처리하지 못한다.

### 핵심 아이디어: 두 번째 최댓값 추적

세그먼트 트리 Beats의 핵심은 각 노드에 **구간의 최댓값(max1)과 두 번째로 큰 값(max2)을 함께 저장**하는 것이다.

`range_chmin(l, r, v)` 연산을 노드 [nl, nr]에 적용할 때:
- `v >= max1[node]`: 아무것도 변하지 않으므로 종료 (불필요 조건)
- `max2[node] < v < max1[node]`: 이 노드의 최댓값들만 v로 교체하고, lazy 태그로 기록 (태그 조건)
- `v <= max2[node]`: 이 범위에서는 더 깊이 내려가야 한다 (재귀 분할)

태그 조건이 만족되면 전체 구간을 O(1)에 처리할 수 있다. 이 분리 덕분에 전체 시간 복잡도가 O((n + q) log² n)로 유지된다.

---

## 왜 필요한가

### 기존 방법의 한계

구간 chmin 연산을 naive하게 처리하면 O(n) 시간이 소요된다. 일반 Lazy Propagation으로는 "v보다 큰 원소만 v로 교체"라는 조건부 갱신을 태그로 표현하고 올바르게 합성하기가 불가능하다.

이 문제는 **Segment Tree Beats 이전에는 sqrt decomposition(O(q√n))이나 직접 업데이트(O(qn))로만 해결**할 수 있었다. Beats 기법은 이를 polylogarithmic으로 개선했다.

### 실제 응용 사례

- 경쟁 프로그래밍의 고급 구간 쿼리 문제 (BZOJ 4695 등)
- 실시간 데이터 스트리밍에서의 이상값 클램핑
- 게임 엔진의 HP 시스템 (구간 내 캐릭터 HP를 최솟값으로 제한)
- 금융 데이터 처리: 구간 내 가격 시퀀스에 상하한 적용 후 합산

---

## 실제 구현 예제

### 예제 1: 구간 chmin + 구간 합 세그먼트 트리 Beats

```cpp
#include <bits/stdc++.h>
using namespace std;

const int MAXN = 100005;
const long long INF = 1e18;

struct Node {
    long long sum;   // 구간 합
    long long max1;  // 구간 최댓값
    long long max2;  // 구간 두 번째 최댓값 (strictly less than max1)
    int max_cnt;     // max1의 등장 횟수
    long long lazy_add;   // 구간 덧셈 lazy 태그
    long long lazy_chmin; // 구간 chmin lazy 태그 (INF = 없음)
} tree[MAXN * 4];

int n;
long long a[MAXN];

void push_up(int node) {
    int l = node * 2, r = node * 2 + 1;
    tree[node].sum = tree[l].sum + tree[r].sum;
    if (tree[l].max1 == tree[r].max1) {
        tree[node].max1 = tree[l].max1;
        tree[node].max_cnt = tree[l].max_cnt + tree[r].max_cnt;
        tree[node].max2 = max(tree[l].max2, tree[r].max2);
    } else if (tree[l].max1 > tree[r].max1) {
        tree[node].max1 = tree[l].max1;
        tree[node].max_cnt = tree[l].max_cnt;
        tree[node].max2 = max(tree[l].max2, tree[r].max1);
    } else {
        tree[node].max1 = tree[r].max1;
        tree[node].max_cnt = tree[r].max_cnt;
        tree[node].max2 = max(tree[l].max1, tree[r].max2);
    }
}

// max1에만 chmin 태그 적용 (max2 < v < max1 조건이 이미 확인된 후 호출)
void apply_chmin(int node, long long v) {
    if (v >= tree[node].max1) return;
    tree[node].sum -= (tree[node].max1 - v) * tree[node].max_cnt;
    tree[node].max1 = v;
    tree[node].lazy_chmin = min(tree[node].lazy_chmin, v);
}

// 구간 덧셈 적용
void apply_add(int node, long long v, int len) {
    tree[node].sum += v * len;
    tree[node].max1 += v;
    if (tree[node].max2 != -INF) tree[node].max2 += v;
    tree[node].lazy_add += v;
    if (tree[node].lazy_chmin != INF) tree[node].lazy_chmin += v;
}

void push_down(int node, int l, int r) {
    int mid = (l + r) / 2;
    int lc = node * 2, rc = node * 2 + 1;

    if (tree[node].lazy_add != 0) {
        apply_add(lc, tree[node].lazy_add, mid - l + 1);
        apply_add(rc, tree[node].lazy_add, r - mid);
        tree[node].lazy_add = 0;
    }
    if (tree[node].lazy_chmin != INF) {
        apply_chmin(lc, tree[node].lazy_chmin);
        apply_chmin(rc, tree[node].lazy_chmin);
        tree[node].lazy_chmin = INF;
    }
}

void build(int node, int l, int r) {
    tree[node].lazy_add = 0;
    tree[node].lazy_chmin = INF;
    if (l == r) {
        tree[node].sum = tree[node].max1 = a[l];
        tree[node].max2 = -INF;
        tree[node].max_cnt = 1;
        return;
    }
    int mid = (l + r) / 2;
    build(node * 2, l, mid);
    build(node * 2 + 1, mid + 1, r);
    push_up(node);
}

// range chmin: [ql, qr]의 모든 원소를 min(원소, v)로
void update_chmin(int node, int l, int r, int ql, int qr, long long v) {
    if (ql > r || qr < l || v >= tree[node].max1) return;
    if (ql <= l && r <= qr && v > tree[node].max2) {
        // 태그 조건: 이 노드의 max1만 v로 교체
        apply_chmin(node, v);
        return;
    }
    push_down(node, l, r);
    int mid = (l + r) / 2;
    update_chmin(node * 2, l, mid, ql, qr, v);
    update_chmin(node * 2 + 1, mid + 1, r, ql, qr, v);
    push_up(node);
}

// range add: [ql, qr]에 v 더하기
void update_add(int node, int l, int r, int ql, int qr, long long v) {
    if (ql > r || qr < l) return;
    if (ql <= l && r <= qr) {
        apply_add(node, v, r - l + 1);
        return;
    }
    push_down(node, l, r);
    int mid = (l + r) / 2;
    update_add(node * 2, l, mid, ql, qr, v);
    update_add(node * 2 + 1, mid + 1, r, ql, qr, v);
    push_up(node);
}

// range sum 쿼리
long long query_sum(int node, int l, int r, int ql, int qr) {
    if (ql > r || qr < l) return 0;
    if (ql <= l && r <= qr) return tree[node].sum;
    push_down(node, l, r);
    int mid = (l + r) / 2;
    return query_sum(node * 2, l, mid, ql, qr) +
           query_sum(node * 2 + 1, mid + 1, r, ql, qr);
}

// range max 쿼리
long long query_max(int node, int l, int r, int ql, int qr) {
    if (ql > r || qr < l) return -INF;
    if (ql <= l && r <= qr) return tree[node].max1;
    push_down(node, l, r);
    int mid = (l + r) / 2;
    return max(query_max(node * 2, l, mid, ql, qr),
               query_max(node * 2 + 1, mid + 1, r, ql, qr));
}

int main() {
    ios_base::sync_with_stdio(false);
    cin.tie(nullptr);

    n = 8;
    // 초기 배열: [3, 1, 4, 1, 5, 9, 2, 6]
    long long arr[] = {3, 1, 4, 1, 5, 9, 2, 6};
    for (int i = 1; i <= n; i++) a[i] = arr[i - 1];

    build(1, 1, n);

    printf("초기 합 [1,8]: %lld\n", query_sum(1, 1, n, 1, n));  // 31
    printf("초기 최댓값 [1,8]: %lld\n", query_max(1, 1, n, 1, n));  // 9

    // [1,8] 전체에 chmin(4) 적용: 5,9,6 → 4
    update_chmin(1, 1, n, 1, 8, 4);
    printf("chmin(4) 후 합: %lld\n", query_sum(1, 1, n, 1, n));   // 25
    printf("chmin(4) 후 최댓값: %lld\n", query_max(1, 1, n, 1, n)); // 4

    // [3,6]에 +2 적용
    update_add(1, 1, n, 3, 6, 2);
    printf("+2 후 [3,6] 합: %lld\n", query_sum(1, 1, n, 3, 6));  // 22

    return 0;
}
```

### 예제 2: 파이썬으로 핵심 로직 시각화

```python
class SegTreeBeats:
    """
    세그먼트 트리 Beats: range_chmin + range_sum 지원
    Python은 속도가 느리므로 개념 이해용으로 사용
    """
    INF = float('inf')

    def __init__(self, arr):
        self.n = len(arr)
        size = self.n
        self.max1 = [-self.INF] * (4 * size)
        self.max2 = [-self.INF] * (4 * size)
        self.max_cnt = [0] * (4 * size)
        self.s = [0] * (4 * size)
        self.lazy_chmin = [self.INF] * (4 * size)
        self._build(arr, 1, 0, self.n - 1)

    def _build(self, arr, node, l, r):
        self.lazy_chmin[node] = self.INF
        if l == r:
            self.max1[node] = arr[l]
            self.max2[node] = -self.INF
            self.max_cnt[node] = 1
            self.s[node] = arr[l]
            return
        mid = (l + r) // 2
        self._build(arr, 2*node, l, mid)
        self._build(arr, 2*node+1, mid+1, r)
        self._push_up(node)

    def _push_up(self, node):
        lc, rc = 2*node, 2*node+1
        self.s[node] = self.s[lc] + self.s[rc]
        if self.max1[lc] == self.max1[rc]:
            self.max1[node] = self.max1[lc]
            self.max_cnt[node] = self.max_cnt[lc] + self.max_cnt[rc]
            self.max2[node] = max(self.max2[lc], self.max2[rc])
        elif self.max1[lc] > self.max1[rc]:
            self.max1[node] = self.max1[lc]
            self.max_cnt[node] = self.max_cnt[lc]
            self.max2[node] = max(self.max2[lc], self.max1[rc])
        else:
            self.max1[node] = self.max1[rc]
            self.max_cnt[node] = self.max_cnt[rc]
            self.max2[node] = max(self.max1[lc], self.max2[rc])

    def _apply_chmin(self, node, v):
        if v >= self.max1[node]:
            return
        self.s[node] -= (self.max1[node] - v) * self.max_cnt[node]
        self.max1[node] = v
        self.lazy_chmin[node] = min(self.lazy_chmin[node], v)

    def _push_down(self, node):
        if self.lazy_chmin[node] < self.INF:
            self._apply_chmin(2*node, self.lazy_chmin[node])
            self._apply_chmin(2*node+1, self.lazy_chmin[node])
            self.lazy_chmin[node] = self.INF

    def range_chmin(self, node, l, r, ql, qr, v):
        if ql > r or qr < l or v >= self.max1[node]:
            return
        if ql <= l and r <= qr and v > self.max2[node]:
            self._apply_chmin(node, v)  # 태그 조건 성공
            return
        self._push_down(node)
        mid = (l + r) // 2
        self.range_chmin(2*node, l, mid, ql, qr, v)
        self.range_chmin(2*node+1, mid+1, r, ql, qr, v)
        self._push_up(node)

    def range_sum(self, node, l, r, ql, qr):
        if ql > r or qr < l:
            return 0
        if ql <= l and r <= qr:
            return self.s[node]
        self._push_down(node)
        mid = (l + r) // 2
        return (self.range_sum(2*node, l, mid, ql, qr) +
                self.range_sum(2*node+1, mid+1, r, ql, qr))

    def chmin(self, l, r, v):
        self.range_chmin(1, 0, self.n-1, l, r, v)

    def query_sum(self, l, r):
        return self.range_sum(1, 0, self.n-1, l, r)


# 테스트
arr = [3, 1, 4, 1, 5, 9, 2, 6]
seg = SegTreeBeats(arr)

print(f"초기 배열: {arr}")
print(f"전체 합: {seg.query_sum(0, 7)}")  # 31

seg.chmin(0, 7, 4)  # 5→4, 9→4, 6→4
print(f"chmin(0,7,4) 후 전체 합: {seg.query_sum(0, 7)}")  # 25
print(f"chmin(0,7,4) 후 [4,5] 합: {seg.query_sum(4, 5)}")  # 8 (4+4)

seg.chmin(2, 5, 3)  # [4,1,4,4] → [3,1,3,3]
print(f"chmin(2,5,3) 후 전체 합: {seg.query_sum(0, 7)}")  # 23
```

---

## 시간 복잡도 분석

세그먼트 트리 Beats의 시간 복잡도 증명은 **势 함수(potential function)**를 사용한 분할 상환 분석에 기반한다. 각 노드에서 발생하는 "태그 조건 실패(break condition hit)"의 총 횟수가 O((n + q) log n)임을 势 함수로 증명한다.

势 함수 Φ = Σ (노드의 고유 최댓값 개수), 즉 세그먼트 트리 전체에서 서로 다른 최댓값을 갖는 노드의 수를 추적하면, 각 연산이 势를 최대 O(log n)만큼 증가시키고 각 "깊이 내려감" 이벤트가 势를 1씩 감소시킴을 보일 수 있다.

결론: `range_chmin`, `range_chmax` 각각 O(log² n), `range_add` / `range_sum` / `range_max`는 O(log n).

---

## 주의사항과 팁

1. **max2 처리에 주의**: `max2`는 strictly second maximum이다. 같은 값이 여러 개 있을 때 max2는 여전히 그보다 작은 값이어야 한다. max_cnt로 최댓값의 개수를 별도 추적하라.

2. **lazy 태그 합성 순서**: add와 chmin 두 태그가 공존할 때, push_down 시 add를 먼저 적용한 뒤 chmin을 적용해야 한다. 순서가 틀리면 결과가 달라진다.

3. **chmin + chmax 동시 지원**: 같은 구조에 chmax도 추가하려면 min1(최솟값), min2(두 번째 최솟값)를 별도로 유지해야 하며, 태그 합성이 크게 복잡해진다. 둘 중 하나만 필요하면 그것만 구현하라.

4. **메모리**: 노드당 저장 필드가 많아 4n 배열 크기에도 불구하고 캐시 미스가 빈번하다. 경쟁 프로그래밍에서는 구조체 배열보다 필드별 배열이 더 빠른 경우가 있다.

5. **Python에서의 사용**: 재귀 깊이 제한(sys.setrecursionlimit)과 속도 문제 때문에 실전에서는 C++ 구현이 필수다.

---

## 참고 자료
- [Ji Driver Segmentation — Codeforces Blog (Ji의 원본 블로그)](https://codeforces.com/blog/entry/57319)
- [Segment Tree Beats — cp-algorithms](https://cp-algorithms.com/data_structures/segment_tree_beats.html)
- [Chtholly Tree and Segment Tree Beats (원논문 번역)](https://codeforces.com/blog/entry/89399)
- [AtCoder Library: Segment Tree (참고 구현)](https://atcoder.github.io/ac-library/document_en/segtree.html)
