---
layout: post
title: "Sparse Table과 범위 최솟값 쿼리(RMQ) 완전 정복: O(1) 쿼리로 정적 배열 범위 연산 마스터하기"
date: 2026-07-19
categories: [cs, computer-science]
tags: [sparse-table, rmq, range-minimum-query, data-structures, algorithms, competitive-programming]
---

## 개념 설명

**범위 최솟값 쿼리(Range Minimum Query, RMQ)** 는 배열 `A`가 주어졌을 때, 임의의 구간 `[l, r]`에서의 최솟값을 묻는 연산입니다. 쿼리 한 번을 O(N)으로 처리할 수 있지만, 쿼리가 수백만 번 주어지면 전체 시간이 O(N × Q)가 되어 실용적이지 않습니다.

**Sparse Table**은 다음을 달성합니다.

- **전처리**: O(N log N) 시간, O(N log N) 공간
- **쿼리**: O(1) 시간 (멱등함수, idempotent function에 한정)

핵심 아이디어는 **겹침을 허용하는 두 구간으로 [l, r]을 덮는 것**입니다. 구간 `[l, r]`의 길이를 `len = r - l + 1`이라 할 때, `k = ⌊log₂(len)⌋`을 구하면 두 구간 `[l, l + 2^k - 1]`과 `[r - 2^k + 1, r]`이 `[l, r]`을 완전히 덮습니다. 최솟값은 멱등 연산(min(a, a) = a)이므로 겹치는 부분이 있어도 결과에 영향이 없습니다.

### 자료구조 정의

`sparse[j][i]`를 인덱스 `i`에서 시작하는 길이 `2^j`인 구간의 최솟값으로 정의합니다.

```
sparse[j][i] = min(A[i], A[i+1], ..., A[i + 2^j - 1])
```

**점화식**:
```
sparse[0][i] = A[i]
sparse[j][i] = min(sparse[j-1][i], sparse[j-1][i + 2^(j-1)])
```

즉, 길이 `2^j` 구간의 최솟값은, 길이 `2^(j-1)`인 두 구간(절반씩)의 최솟값 중 더 작은 것입니다.

---

## 왜 필요한가?

### 실제 사용 사례

1. **LCA(Lowest Common Ancestor)** 탐색: 오일러 경로(Euler Tour)를 통해 트리 문제를 RMQ로 환원합니다. 두 노드의 LCA를 찾을 때 오일러 경로 배열에서 깊이 최솟값을 찾으면 됩니다.

2. **Suffix Array + LCP Array**: 두 접미사 사이의 최장 공통 접두사(LCP)를 구할 때 LCP 배열에서 RMQ를 수행합니다.

3. **슬라이딩 윈도우 최솟값**: 정적 배열이라면 Sparse Table이 단조 덱보다 구현이 단순합니다.

4. **2D RMQ**: 이미지 처리, 지형 데이터 분석에서 직사각형 영역의 최솟값을 O(1)에 찾습니다.

### 다른 RMQ 방법과의 비교

| 방법 | 전처리 | 쿼리 | 갱신 지원 |
|------|--------|------|-----------|
| 선형 탐색 | O(1) | O(N) | O(1) |
| Segment Tree | O(N) | O(log N) | O(log N) |
| Sparse Table | O(N log N) | **O(1)** | X |
| Farach-Colton & Bender | O(N) | O(1) | X |

배열이 변하지 않고 쿼리가 많다면 Sparse Table이 최선입니다.

---

## 실제 구현 예제

### 예제 1: C++로 구현하는 Sparse Table

```cpp
#include <bits/stdc++.h>
using namespace std;

struct SparseTable {
    int n, LOG;
    vector<vector<int>> sparse;
    vector<int> log2_floor;

    SparseTable(const vector<int>& a) {
        n = a.size();
        LOG = __lg(n) + 1;  // floor(log2(n)) + 1
        sparse.assign(LOG, vector<int>(n));
        log2_floor.resize(n + 1);

        // log2 사전 계산: O(N)
        log2_floor[1] = 0;
        for (int i = 2; i <= n; i++) {
            log2_floor[i] = log2_floor[i / 2] + 1;
        }

        // 전처리: sparse[0] = 원본 배열
        for (int i = 0; i < n; i++) sparse[0][i] = a[i];

        // 점화식으로 채우기: O(N log N)
        for (int j = 1; j < LOG; j++) {
            for (int i = 0; i + (1 << j) <= n; i++) {
                sparse[j][i] = min(sparse[j-1][i],
                                   sparse[j-1][i + (1 << (j-1))]);
            }
        }
    }

    // O(1) 쿼리: [l, r] 범위의 최솟값
    int query(int l, int r) {
        int k = log2_floor[r - l + 1];
        return min(sparse[k][l], sparse[k][r - (1 << k) + 1]);
    }
};

int main() {
    ios_base::sync_with_stdio(false);
    cin.tie(nullptr);

    int n, q;
    cin >> n >> q;

    vector<int> a(n);
    for (int& x : a) cin >> x;

    SparseTable st(a);

    while (q--) {
        int l, r;
        cin >> l >> r;
        // 1-indexed 입력을 0-indexed로 변환
        cout << st.query(l - 1, r - 1) << "\n";
    }
    return 0;
}
```

**동작 예시**:
```
입력:
5 3
3 1 4 1 5
1 3
2 5
1 5

출력:
1
1
1
```

### 예제 2: Python으로 구현하는 Sparse Table (범위 GCD)

min뿐 아니라 GCD처럼 멱등성을 가지는 모든 연산에 Sparse Table을 적용할 수 있습니다. (`gcd(a, a) = a`)

```python
import math
from typing import Callable

class SparseTable:
    """멱등 연산(idempotent)을 지원하는 Sparse Table"""

    def __init__(self, arr: list[int], op: Callable[[int, int], int]):
        self.n = len(arr)
        self.op = op
        self.LOG = max(1, self.n.bit_length())

        # sparse[j][i] = arr[i..i+2^j-1]에 대한 연산 결과
        self.sparse = [[0] * self.n for _ in range(self.LOG)]
        self.log2_floor = [0] * (self.n + 1)

        # log2 테이블 사전 계산
        for i in range(2, self.n + 1):
            self.log2_floor[i] = self.log2_floor[i // 2] + 1

        # 전처리
        self.sparse[0] = arr[:]
        for j in range(1, self.LOG):
            for i in range(self.n - (1 << j) + 1):
                self.sparse[j][i] = op(
                    self.sparse[j-1][i],
                    self.sparse[j-1][i + (1 << (j-1))]
                )

    def query(self, l: int, r: int) -> int:
        """[l, r] (0-indexed, inclusive) 범위에 연산 적용"""
        k = self.log2_floor[r - l + 1]
        return self.op(self.sparse[k][l], self.sparse[k][r - (1 << k) + 1])


# 사용 예시 1: 범위 최솟값
arr = [2, 4, 3, 1, 6, 7, 8, 9, 1, 7]
st_min = SparseTable(arr, min)
print(st_min.query(0, 4))   # min(2,4,3,1,6) = 1
print(st_min.query(2, 7))   # min(3,1,6,7,8,9) = 1
print(st_min.query(5, 9))   # min(7,8,9,1,7) = 1

# 사용 예시 2: 범위 GCD
st_gcd = SparseTable(arr, math.gcd)
print(st_gcd.query(0, 3))   # gcd(2,4,3,1) = 1
print(st_gcd.query(0, 1))   # gcd(2,4) = 2
```

---

### 예제 3: LCA를 Sparse Table + 오일러 투어로 O(1)에 구하기

```cpp
#include <bits/stdc++.h>
using namespace std;

const int MAXN = 100005;
vector<int> adj[MAXN];
int euler[2 * MAXN];  // 오일러 경로
int depth[MAXN];
int first[MAXN];      // 오일러 경로에서 노드 i의 첫 등장 위치
int idx = 0;

void dfs(int u, int parent, int d) {
    depth[u] = d;
    first[u] = idx;
    euler[idx++] = u;
    for (int v : adj[u]) {
        if (v != parent) {
            dfs(v, u, d + 1);
            euler[idx++] = u;  // 돌아올 때 다시 기록
        }
    }
}

// Sparse Table on euler array (depth 기준 최솟값)
// query(first[u], first[v])의 depth 최솟값 위치 = LCA

int main() {
    int n; cin >> n;
    for (int i = 0; i < n - 1; i++) {
        int u, v; cin >> u >> v;
        adj[u].push_back(v);
        adj[v].push_back(u);
    }
    dfs(1, 0, 0);
    // euler 배열로 Sparse Table 구성 후 쿼리하면 O(1) LCA 달성
    return 0;
}
```

---

## 주의사항 및 팁

### 1. 멱등성(Idempotency) 필수 조건

Sparse Table이 O(1) 쿼리를 달성하는 비결은 **겹치는 두 구간으로 전체 구간을 덮는 것**입니다. 이 방식이 올바르게 동작하려면 연산이 반드시 **멱등성(idempotent)** 을 만족해야 합니다.

- **가능**: `min`, `max`, `gcd`, `bitwise AND/OR`
- **불가능**: `sum`, `product`, `XOR` (겹치는 원소가 두 번 계산됨)

`sum`처럼 멱등성이 없는 연산에는 **Segment Tree** 또는 **Prefix Sum**을 사용하세요.

### 2. 메모리 레이아웃 최적화

기본적으로 `sparse[j][i]` 형태(행 = 레벨, 열 = 인덱스)로 저장하면 전처리 시 캐시 친화적입니다. 반면 `sparse[i][j]` (행 = 인덱스)로 저장하면 쿼리 시 캐시 미스가 증가합니다.

```cpp
// 좋음: 같은 레벨 j의 원소들이 메모리에 연속 배치
vector<vector<int>> sparse(LOG, vector<int>(n));
```

### 3. LOG 크기 설정

`LOG = floor(log2(N)) + 1`이면 충분합니다. `N = 10^6`이면 `LOG = 20`으로 충분합니다. 안전하게 `LOG = 21` 또는 `__lg(n) + 2`를 사용하는 것을 권장합니다.

### 4. 0-indexed vs 1-indexed

경쟁 프로그래밍에서 입력이 1-indexed로 주어지는 경우가 많습니다. 쿼리 시 `query(l-1, r-1)`로 변환하거나, 배열을 1-indexed로 구성하세요.

### 5. 2D Sparse Table

2차원 배열에서도 적용 가능합니다. 전처리가 O(NM log N log M)이고 쿼리가 O(1)이지만 메모리 사용량이 크므로 `N, M ≤ 1000` 정도에서 사용하는 것이 현실적입니다.

---

## 참고 자료

- [Sparse Table - Algorithms for Competitive Programming](https://cp-algorithms.com/data_structures/sparse-table.html)
- [RMQ task - Algorithms for Competitive Programming](https://cp-algorithms.com/sequences/rmq.html)
- [Tutorial: Ultimate RMQ + Sparse Table - Codeforces](https://codeforces.com/blog/entry/140101)
- [Tutorial: Range minimum query in O(1) with linear time construction - Codeforces](https://codeforces.com/blog/entry/78931)
