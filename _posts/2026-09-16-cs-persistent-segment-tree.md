---
layout: post
title: "영속성 세그먼트 트리(Persistent Segment Tree) 완전 정복: 과거 버전에 O(log N)으로 쿼리하기"
date: 2026-09-16
categories: [cs, computer-science]
tags: [data-structure, segment-tree, persistent, competitive-programming, algorithm]
---

## 개념 설명

**영속성 세그먼트 트리(Persistent Segment Tree)**는 세그먼트 트리에 **버전 관리** 기능을 더한 자료구조입니다. 일반 세그먼트 트리는 업데이트 시 기존 상태를 덮어쓰지만, 영속성 버전은 **과거 버전을 그대로 보존**하면서 새로운 버전을 생성합니다.

핵심 아이디어는 **경로 복사(Path Copying)** 기법입니다. 값을 업데이트할 때 변경이 필요한 노드만 새로 생성하고, 변경이 필요 없는 노드는 이전 버전과 **공유**합니다. 크기 N인 세그먼트 트리에서 하나의 포인트 업데이트는 루트에서 리프까지 O(log N)개의 노드를 거치므로, 버전 하나를 생성하는 데 O(log N)개의 노드만 추가됩니다.

```
버전 0 (초기):
        [1,8]
       /     \
   [1,4]    [5,8]
   /   \    /   \
[1,2][3,4][5,6][7,8]

버전 1 (인덱스 3 업데이트 후):
    [1,8]' (새 노드)
   /       \
[1,4]'   [5,8] ← 공유
/   \
[1,2][3,4]' (새 노드)
```

이 구조를 통해 버전 v의 루트 포인터를 기억해두면, 언제든 O(log N)에 그 시점의 상태를 조회할 수 있습니다.

---

## 왜 필요한가

**일반 세그먼트 트리의 한계**: 업데이트마다 N 크기의 트리를 복사하면 O(N)의 공간이 필요합니다. Q개의 업데이트가 있다면 O(NQ)로 폭발적으로 증가합니다.

영속성 세그먼트 트리는 이를 해결합니다:
- **공간 복잡도**: 버전당 O(log N)개 노드 추가 → 총 O(N + Q log N)
- **시간 복잡도**: 버전 생성 O(log N), 쿼리 O(log N)

### 대표적인 활용 사례

**1. K번째 수 쿼리 (Offline K-th Query)**  
구간 [l, r]에서 K번째로 작은 수를 찾는 문제를 O(log N)에 처리합니다. 좌표 압축 후 값을 하나씩 삽입하면서 버전을 생성하면, `version[r] - version[l-1]`로 구간 내 각 값의 빈도를 구할 수 있습니다.

**2. 정적 구간 쿼리 (Static Range Queries)**  
오프라인 처리 없이 순서대로 쌓이는 배열의 누적 상태를 각 시점에서 조회할 때.

**3. 동적 트리 경로 쿼리**  
Heavy-Light Decomposition과 결합하여 트리 경로의 K번째 원소를 쿼리하는 데 활용.

---

## 실제 구현 예제

### 예제 1: 기본 영속성 세그먼트 트리 구현 (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

const int MAXN = 200005;

struct Node {
    int left, right, val;
} tree[MAXN * 40]; // 충분한 노드 공간

int roots[MAXN]; // 각 버전의 루트 인덱스
int cnt = 0;     // 노드 카운터

// 새 노드를 할당하고 이전 노드를 복사
int newNode(int prev) {
    tree[++cnt] = tree[prev];
    return cnt;
}

// 초기 트리 빌드: [l, r] 범위를 0으로 초기화
int build(int l, int r) {
    int node = ++cnt;
    tree[node].val = 0;
    if (l == r) return node;
    int mid = (l + r) / 2;
    tree[node].left = build(l, mid);
    tree[node].right = build(mid + 1, r);
    return node;
}

// 포인트 업데이트: 버전 prev의 트리에서 pos 위치에 val 추가
int update(int prev, int l, int r, int pos, int val) {
    int node = newNode(prev); // 현재 노드를 복사하여 새 버전 생성
    if (l == r) {
        tree[node].val += val;
        return node;
    }
    int mid = (l + r) / 2;
    if (pos <= mid)
        tree[node].left = update(tree[prev].left, l, mid, pos, val);
    else
        tree[node].right = update(tree[prev].right, mid + 1, r, pos, val);
    tree[node].val = tree[tree[node].left].val + tree[tree[node].right].val;
    return node;
}

// 두 버전의 차이로 구간 합 계산
int query(int lNode, int rNode, int l, int r, int ql, int qr) {
    if (ql <= l && r <= qr)
        return tree[rNode].val - tree[lNode].val;
    int mid = (l + r) / 2;
    int res = 0;
    if (ql <= mid)
        res += query(tree[lNode].left, tree[rNode].left, l, mid, ql, qr);
    if (qr > mid)
        res += query(tree[lNode].right, tree[rNode].right, mid + 1, r, ql, qr);
    return res;
}

int main() {
    ios_base::sync_with_stdio(false);
    cin.tie(nullptr);

    int n, q;
    cin >> n >> q;

    vector<int> arr(n + 1);
    for (int i = 1; i <= n; i++) cin >> arr[i];

    // 좌표 압축
    vector<int> sorted_arr(arr.begin() + 1, arr.end());
    sort(sorted_arr.begin(), sorted_arr.end());
    sorted_arr.erase(unique(sorted_arr.begin(), sorted_arr.end()), sorted_arr.end());
    int m = sorted_arr.size();

    auto compress = [&](int x) {
        return lower_bound(sorted_arr.begin(), sorted_arr.end(), x) - sorted_arr.begin() + 1;
    };

    roots[0] = build(1, m);

    for (int i = 1; i <= n; i++) {
        roots[i] = update(roots[i - 1], 1, m, compress(arr[i]), 1);
    }

    // 쿼리: [l, r] 구간에서 K번째 작은 수
    while (q--) {
        int l, r, k;
        cin >> l >> r >> k;

        // 이진 탐색으로 K번째 수 찾기
        int lo = 1, hi = m, lNode = roots[l - 1], rNode = roots[r];
        while (lo < hi) {
            int mid = (lo + hi) / 2;
            int leftCount = tree[tree[rNode].left].val - tree[tree[lNode].left].val;
            if (leftCount >= k) {
                lNode = tree[lNode].left;
                rNode = tree[rNode].left;
                hi = mid;
            } else {
                k -= leftCount;
                lNode = tree[lNode].right;
                rNode = tree[rNode].right;
                lo = mid + 1;
            }
        }
        cout << sorted_arr[lo - 1] << '\n';
    }
    return 0;
}
```

이 구현에서 핵심은 `roots[r] - roots[l-1]`을 통해 구간 [l, r]의 값 분포를 O(log N)에 파악하고, 이진 탐색으로 K번째 수를 찾는 것입니다.

---

### 예제 2: 구간 K번째 수 쿼리 (Java 구현)

```java
import java.util.*;

public class PersistentSegTree {
    static int[] left, right, val;
    static int cnt = 0;

    static int build(int l, int r) {
        int node = ++cnt;
        val[node] = 0;
        if (l == r) return node;
        int mid = (l + r) / 2;
        left[node] = build(l, mid);
        right[node] = build(mid + 1, r);
        return node;
    }

    static int update(int prev, int l, int r, int pos) {
        int node = ++cnt;
        left[node] = left[prev];
        right[node] = right[prev];
        val[node] = val[prev] + 1;
        if (l == r) return node;
        int mid = (l + r) / 2;
        if (pos <= mid)
            left[node] = update(left[prev], l, mid, pos);
        else
            right[node] = update(right[prev], mid + 1, r, pos);
        return node;
    }

    static int kth(int lv, int rv, int l, int r, int k) {
        if (l == r) return l;
        int mid = (l + r) / 2;
        int leftCount = val[left[rv]] - val[left[lv]];
        if (leftCount >= k)
            return kth(left[lv], left[rv], l, mid, k);
        else
            return kth(right[lv], right[rv], mid + 1, r, k - leftCount);
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int n = sc.nextInt(), q = sc.nextInt();

        int[] arr = new int[n + 1];
        for (int i = 1; i <= n; i++) arr[i] = sc.nextInt();

        // 좌표 압축
        int[] sorted = Arrays.copyOfRange(arr, 1, n + 1);
        Arrays.sort(sorted);
        int m = (int) Arrays.stream(sorted).distinct().count();
        int[] unique = Arrays.stream(sorted).distinct().toArray();

        int MAX_NODES = (n + 1) * 40;
        left = new int[MAX_NODES];
        right = new int[MAX_NODES];
        val = new int[MAX_NODES];

        int[] roots = new int[n + 1];
        roots[0] = build(1, m);

        for (int i = 1; i <= n; i++) {
            int compressed = Arrays.binarySearch(unique, arr[i]) + 1;
            roots[i] = update(roots[i - 1], 1, m, compressed);
        }

        StringBuilder sb = new StringBuilder();
        while (q-- > 0) {
            int l = sc.nextInt(), r = sc.nextInt(), k = sc.nextInt();
            int idx = kth(roots[l - 1], roots[r], 1, m, k);
            sb.append(unique[idx - 1]).append('\n');
        }
        System.out.print(sb);
    }
}
```

---

## 주의사항과 팁

### 1. 메모리 계산을 미리 하라
업데이트마다 O(log N)개의 노드가 추가됩니다. N = 200,000이고 쿼리가 200,000개라면 최대 약 200,000 × 18 = 3,600,000개의 노드가 필요합니다. 배열을 충분히 크게 잡아야 합니다 (보통 N × 40이 안전한 상수).

### 2. 포인터 방식 vs 배열 방식
동적 메모리 할당(포인터)보다 정적 배열 방식이 캐시 친화적이고 실제로 빠릅니다. 경쟁 프로그래밍에서는 배열 방식을 권장합니다.

### 3. 좌표 압축 필수
값의 범위가 크면(최대 10^9) 좌표 압축(coordinate compression)을 먼저 적용해야 합니다. 중복을 제거하고 1부터 M까지 재매핑합니다.

### 4. 범위 업데이트는 불가
영속성 세그먼트 트리는 포인트 업데이트에 적합합니다. 범위 업데이트에 Lazy Propagation을 적용하면 영속성을 유지하기 매우 복잡해집니다. 이 경우 영속성 평형 BST(예: 영속성 Treap)를 고려하세요.

### 5. 함수형 세그먼트 트리
순수 함수형 언어(Haskell, Scala)에서는 변경 불가능한 데이터 구조 특성상 영속성 세그먼트 트리가 자연스럽게 구현됩니다. 구조 공유(structural sharing)가 자동으로 이루어집니다.

### 6. 관련 응용
- **2D 오프라인 쿼리**: 하나의 축을 정렬하여 버전으로 활용
- **HLD + 영속성 세그먼트 트리**: 트리 경로의 K번째 원소 쿼리
- **순위 트리(Rank Tree)**: 동적 순위 관리

---

## 시간/공간 복잡도 요약

| 연산 | 시간 복잡도 | 공간 복잡도 |
|------|-----------|-----------|
| 초기화 | O(N) | O(N) |
| 버전 생성(업데이트) | O(log N) | O(log N) |
| 구간 쿼리 | O(log N) | O(1) |
| Q개 업데이트 후 총 공간 | — | O(N + Q log N) |

영속성 세그먼트 트리는 "과거 버전에 접근"이라는 단 하나의 아이디어로 놀랍도록 다양한 문제를 해결합니다. 경로 복사의 핵심을 이해하면 영속성 트라이, 영속성 Union-Find 등 다양한 영속성 자료구조로 확장할 수 있습니다.

## 참고 자료
- [Segment Tree - Algorithms for Competitive Programming](https://cp-algorithms.com/data_structures/segment_tree.html)
- [Persistent Data Structures - USACO Guide](https://usaco.guide/adv/persistent)
- [Persistent Segment Tree - Codeforces Blog](https://codeforces.com/blog/entry/56760)
- [Algorithm Gym :: Everything About Segment Trees](https://codeforces.com/blog/entry/15890)
