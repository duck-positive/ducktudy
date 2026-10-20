---
layout: post
title: "스파스 테이블(Sparse Table)과 RMQ 완전 정복: 구간 최솟값을 O(1)에 처리하는 비밀"
date: 2026-10-20
categories: [cs, computer-science]
tags: [sparse-table, rmq, range-minimum-query, algorithm, data-structure, idempotent, lca]
---

## 스파스 테이블이란?

**스파스 테이블(Sparse Table)**은 정적 배열에서 **구간 최솟값·최댓값·GCD** 같은 **멱등(idempotent) 연산**을 전처리 O(N log N), 쿼리 O(1)에 답하는 자료구조다. '희소(sparse)'라는 이름은 2의 거듭제곱 길이의 구간만 저장한다는 데서 왔다.

배열이 변경되지 않는다는 전제 하에 세그먼트 트리(쿼리 O(log N))나 펜윅 트리보다 **쿼리 속도가 훨씬 빠르다**. 이 덕분에 온라인 알고리즘 대회와 오프라인 LCA 쿼리, 데이터베이스 분석 등 다양한 곳에서 활용된다.

### 멱등 연산(Idempotent Operation)이란?

연산 `f`가 멱등하다는 것은 같은 원소를 두 번 이상 적용해도 결과가 바뀌지 않는다는 의미다.

```
f(a, a) = a
```

- **min(3, 3) = 3** → 멱등 ✓
- **max(5, 5) = 5** → 멱등 ✓
- **gcd(6, 6) = 6** → 멱등 ✓
- **sum(3, 3) = 6** → 멱등 **✗** (합은 겹치면 틀림)

스파스 테이블은 두 구간이 겹쳐도 되는 경우에만 O(1) 쿼리가 성립한다. 합(sum) 같은 비멱등 연산은 세그먼트 트리를 사용해야 한다.

---

## 왜 스파스 테이블이 필요한가?

RMQ(Range Minimum Query) 문제를 예로 들어보자.

- **브루트 포스**: 쿼리마다 구간을 순회 → O(N) 쿼리, 전처리 불필요
- **세그먼트 트리**: 전처리 O(N), 쿼리 O(log N), 업데이트 O(log N)
- **스파스 테이블**: 전처리 O(N log N), 쿼리 **O(1)**, 업데이트 불가

배열이 고정된 채로 수백만 번의 구간 최솟값 쿼리가 들어올 때, 스파스 테이블은 세그먼트 트리보다 훨씬 빠른 응답을 보장한다.

대표적 활용 사례:
- **LCA(최소 공통 조상)**: Euler Tour + 스파스 테이블로 O(1) LCA
- **비교 기반 알고리즘 최적화**: 접미사 배열(Suffix Array) 구축 후 LCP 쿼리
- **경쟁 프로그래밍**: 오프라인 RMQ 쿼리가 많은 경우

---

## 핵심 아이디어: 이진 거듭제곱 구간의 전처리

**전처리 정의**

`table[k][i]` = 인덱스 `i`에서 시작하는 길이 `2^k`인 구간의 최솟값

```
table[0][i] = nums[i]                  (길이 1 구간 = 원소 자체)
table[k][i] = min(table[k-1][i],
                  table[k-1][i + 2^(k-1)])
```

길이 `2^k` 구간을 절반으로 나눠 두 개의 `2^(k-1)` 구간의 답을 합치는 점화식이다. 이 방식으로 O(N log N) 번의 연산으로 모든 2의 거듭제곱 길이 구간을 채운다.

**쿼리 처리**

구간 `[l, r]`의 최솟값을 구하려면, `2^k ≤ (r - l + 1)`인 최대의 k를 찾는다.

```
k = floor(log2(r - l + 1))
answer = min(table[k][l], table[k][r - 2^k + 1])
```

두 구간 `[l, l + 2^k - 1]`과 `[r - 2^k + 1, r]`은 서로 겹칠 수 있지만, 최솟값은 멱등 연산이라 겹쳐도 결과가 정확하다.

---

## 실제 구현 예제

### 예제 1: 스파스 테이블 — 구간 최솟값 쿼리

```python
import math
from typing import List

class SparseTable:
    """
    정적 배열에 대해 구간 최솟값을 O(1)에 답하는 스파스 테이블.
    전처리: O(N log N), 쿼리: O(1)
    """

    def __init__(self, nums: List[int]):
        n = len(nums)
        LOG = max(1, int(math.log2(n)) + 1) if n > 0 else 1

        # table[k][i]: i에서 시작하는 길이 2^k 구간의 최솟값
        self.table = [[float('inf')] * n for _ in range(LOG)]
        self.log2 = [0] * (n + 1)

        # log2 테이블 전처리 (쿼리 시 O(1) 로그 계산용)
        for i in range(2, n + 1):
            self.log2[i] = self.log2[i // 2] + 1

        # 초기화: 길이 1 구간
        self.table[0] = nums[:]

        # DP로 나머지 채우기
        for k in range(1, LOG):
            for i in range(n - (1 << k) + 1):
                self.table[k][i] = min(
                    self.table[k - 1][i],
                    self.table[k - 1][i + (1 << (k - 1))]
                )

    def query(self, l: int, r: int) -> int:
        """[l, r] 구간의 최솟값 반환 (0-indexed, inclusive)."""
        k = self.log2[r - l + 1]
        return min(self.table[k][l], self.table[k][r - (1 << k) + 1])


# 테스트
nums = [7, 2, 3, 0, 5, 10, 3, 12, 18]
st = SparseTable(nums)

print(st.query(0, 4))   # 0 → min(7,2,3,0,5)
print(st.query(4, 7))   # 3 → min(5,10,3,12)
print(st.query(0, 8))   # 0 → 전체 최솟값
print(st.query(2, 5))   # 0 → min(3,0,5,10)
```

출력:
```
0
3
0
0
```

---

### 예제 2: 스파스 테이블을 활용한 O(1) LCA

Euler Tour를 이용해 트리의 LCA(Lowest Common Ancestor)를 O(1)에 구할 수 있다. 트리를 DFS하며 방문 순서(Euler Tour)를 기록하고, 두 노드 사이의 구간에서 깊이가 가장 낮은(작은) 노드를 스파스 테이블로 찾으면 된다.

```python
from typing import List

class LCASparseTable:
    """
    Euler Tour + Sparse Table로 O(N log N) 전처리, O(1) LCA 쿼리.
    """

    def __init__(self, n: int, edges: List[tuple]):
        self.graph = [[] for _ in range(n)]
        for u, v in edges:
            self.graph[u].append(v)
            self.graph[v].append(u)

        self.depth = [0] * n
        self.first = [-1] * n
        self.euler = []
        self.euler_depth = []

        self._dfs(0, -1, 0)

        self.st = SparseTable(self.euler_depth)

    def _dfs(self, node: int, parent: int, d: int):
        self.depth[node] = d
        self.first[node] = len(self.euler)
        self.euler.append(node)
        self.euler_depth.append(d)
        for neighbor in self.graph[node]:
            if neighbor != parent:
                self._dfs(neighbor, node, d + 1)
                self.euler.append(node)
                self.euler_depth.append(d)

    def lca(self, u: int, v: int) -> int:
        l, r = self.first[u], self.first[v]
        if l > r:
            l, r = r, l
        min_depth = self.st.query(l, r)
        for idx in range(l, r + 1):
            if self.euler_depth[idx] == min_depth:
                return self.euler[idx]
        return -1


# 5개 노드, 루트 0인 트리 예제
#      0
#    / | \
#   1  2  3
#      |
#      4
edges = [(0,1),(0,2),(0,3),(2,4)]
lca_solver = LCASparseTable(5, edges)
print(lca_solver.lca(1, 4))  # 0
print(lca_solver.lca(2, 4))  # 2
print(lca_solver.lca(1, 3))  # 0
```

실제 대회에서는 Euler Tour 인덱스 기반 직접 접근 방식으로 더 단순하게 구현한다. 위 예제는 개념 흐름 이해를 위한 것이다.

---

## 주의사항 및 팁

### 1. 멱등 연산에만 O(1) 쿼리가 성립

합(sum), 곱(product) 등 겹치면 값이 달라지는 연산은 스파스 테이블로 O(1) 쿼리가 불가능하다. 이때는 세그먼트 트리나 펜윅 트리를 쓴다.

### 2. 정적 배열 전용

스파스 테이블은 업데이트(값 변경)를 지원하지 않는다. 배열이 자주 바뀌는 경우에는 세그먼트 트리가 더 적합하다. "정적이고 쿼리가 많을 때" 스파스 테이블을 선택한다.

### 3. 공간 복잡도 O(N log N)

최대 N = 10^6, log N ≈ 20이면 테이블 크기는 약 2×10^7 엔트리다. 자료형이 int(4바이트)라면 약 80MB로 메모리 제한에 주의해야 한다.

### 4. log2 테이블 사전 계산

쿼리마다 `math.log2`를 호출하면 상수 인자가 늘어난다. 정수 배열로 log2를 미리 계산해두면 실제 O(1)에 가까운 쿼리가 된다.

### 5. 비트 연산으로 2의 거듭제곱 처리

`1 << k`는 `2^k`를, `i >> 1`은 `i // 2`를 의미한다. C++ 같은 언어에서는 `__builtin_clz`나 `__lg`로 더 빠르게 log2를 구할 수 있다.

---

## 복잡도 정리

| | 전처리 | 쿼리 | 업데이트 | 공간 |
|---|--------|------|---------|------|
| 브루트 포스 | O(1) | O(N) | O(1) | O(1) |
| 세그먼트 트리 | O(N) | O(log N) | O(log N) | O(N) |
| **스파스 테이블** | **O(N log N)** | **O(1)** | **불가** | **O(N log N)** |

---

## 참고 자료

- [Sparse Tables — Baeldung on Computer Science](https://www.baeldung.com/cs/sparse-tables)
- [Sparse Table — GeeksforGeeks](https://www.geeksforgeeks.org/sparse-table/)
- [Ultimate RMQ + Sparse Table Tutorial — Codeforces](https://codeforces.net/topic/140727/en1)
- [RMQ and LCA — University of Illinois Lecture Notes](https://jeffe.cs.illinois.edu/teaching/225H/notes/RMQ-LCA.pdf)
