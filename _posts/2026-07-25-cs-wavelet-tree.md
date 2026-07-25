---
layout: post
title: "Wavelet Tree 완전 정복: 모든 범위 쿼리를 O(log N)에 해결하는 만능 자료구조"
date: 2026-07-25
categories: [cs, computer-science]
tags: [data-structures, wavelet-tree, range-queries, kth-smallest, algorithms, competitive-programming]
---

배열에서 구간 `[L, R]` 안의 **k번째 작은 수**를 구하는 문제를 생각해 보자. 가장 단순한 방법은 구간을 추출해서 정렬하는 것인데, 이는 쿼리당 O((R-L+1) log(R-L+1))이 걸린다. 쿼리가 수십만 개라면 사실상 불가능하다. **Wavelet Tree**는 이 문제를 전처리 O(N log N), 쿼리당 O(log N)으로 해결하면서, 그 외에도 구간 빈도 계산, 구간 k번째 최솟값, 역전(inversion) 개수 등 수십 가지 범위 쿼리를 모두 지원한다.

## Wavelet Tree란 무엇인가?

Wavelet Tree는 2002년 Grossi, Gupta, Vitter가 제안한 자료구조로, **배열을 재귀적으로 분할**하는 완전 이진 트리다. 원래는 압축 자료구조(succinct data structure) 분야에서 텍스트 인덱스를 위해 개발되�으나, 현재는 범위 쿼리 문제를 위한 강력한 도구로 경쟁 프로그래밍과 데이터베이스 엔진에 광범위하게 활용된다.

### 핵심 아이디어: 값을 기준으로 재귀 분할

배열 `A = [5, 3, 1, 4, 2, 7, 6]`이 있다고 하자. 값의 범위는 `[1, 7]`이다.

1. **루트 노드**: 중간값 `mid = 4`를 기준으로 각 원소가 `≤ mid` 인지 아닌지를 비트맵으로 기록한다.  
   비트맵: `[0, 1, 1, 0, 1, 0, 0]` (0=왼쪽, 1=오른쪽 — 값이 크면 0, 작으면 1로 하는 구현도 있음)

2. **왼쪽 서브트리**: `A`에서 `≤ mid`인 원소들만 추출 → `[3, 1, 4, 2]`로 재귀

3. **오른쪽 서브트리**: `A`에서 `> mid`인 원소들만 추출 → `[5, 7, 6]`으로 재귀

트리의 높이는 `O(log(max_value))`이고, 각 레벨에서 총 N개의 원소가 관리되므로 전체 공간 복잡도는 `O(N log(max_value))`이다.

## 왜 Wavelet Tree가 필요한가?

Wavelet Tree가 해결하는 대표적인 문제들:

| 쿼리 유형 | 시간 복잡도 |
|-----------|-------------|
| 구간 k번째 최솟값 `kth(L, R, k)` | O(log N) |
| 구간 내 값 `v`의 등장 횟수 `count(L, R, v)` | O(log N) |
| 구간 내 `≤ v`인 원소 개수 `rank(L, R, v)` | O(log N) |
| 구간 내 `k`번째 `≤ v`인 원소의 위치 `select(L, R, k, v)` | O(log N) |
| 구간 역전 수 (inversion count) | O(N log N) |
| 구간 원소의 합 (with augmentation) | O(log N) |

비교 대상인 **영속 세그먼트 트리(Persistent Segment Tree)** 도 동일한 쿼리를 O(log N)에 처리하지만, Wavelet Tree는 약 2배 가량 메모리 효율이 좋고 캐시 친화적이다.

## Wavelet Tree 완전 구현 (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

struct WaveletTree {
    int lo, hi;  // 이 노드가 담당하는 값 범위 [lo, hi]
    WaveletTree *left = nullptr, *right = nullptr;
    vector<int> b;  // b[i] = 처음 i개 원소 중 왼쪽(≤ mid)으로 간 개수 (prefix count)

    // [from, to) 범위의 배열로 트리 구축
    // arr은 정렬 안된 원본 배열 (현재 레벨에서 재배치된 순서)
    void build(int *from, int *to, int lo, int hi) {
        this->lo = lo;
        this->hi = hi;
        if (lo == hi) return;  // 리프 노드

        int mid = (lo + hi) / 2;
        // 각 원소가 왼쪽(≤mid)으로 가는지 비트맵 구성
        auto f = [mid](int x) { return x <= mid; };

        b.reserve(to - from + 1);
        b.push_back(0);  // prefix sum: b[i] = [0..i-1] 중 왼쪽으로 간 수
        for (auto it = from; it != to; ++it) {
            b.push_back(b.back() + f(*it));
        }

        // 안정 분할: 왼쪽(≤mid) 원소들을 먼저, 오른쪽 원소들을 뒤에
        auto pivot = stable_partition(from, to, f);

        left  = new WaveletTree();
        right = new WaveletTree();
        left->build(from, pivot, lo, mid);
        right->build(pivot, to, mid + 1, hi);
    }

    /**
     * 구간 [l, r] (1-indexed)에서 k번째 최솟값을 반환
     */
    int kth(int l, int r, int k) {
        if (lo == hi) return lo;

        // [l-1, r] 구간에서 왼쪽 서브트리로 간 원소 수
        int inLeft  = b[r] - b[l - 1];
        int inRight = (r - l + 1) - inLeft;

        if (k <= inLeft) {
            // k번째가 왼쪽 서브트리에 있음
            // 새로운 l, r: b[]를 이용해 변환
            return left->kth(b[l - 1] + 1, b[r], k);
        } else {
            // k번째가 오른쪽 서브트리에 있음
            int lb = l - 1 - b[l - 1];  // [0..l-1] 중 오른쪽으로 간 수
            int rb = r - b[r];           // [0..r] 중 오른쪽으로 간 수
            return right->kth(lb + 1, rb, k - inLeft);
        }
    }

    /**
     * 구간 [l, r]에서 값이 ≤ v인 원소의 개수 반환 (rank)
     */
    int rank(int l, int r, int v) {
        if (lo == hi) return (lo <= v) ? (r - l + 1) : 0;
        if (v >= hi)  return r - l + 1;  // 오른쪽 범위 전체가 ≤ v
        if (v < lo)   return 0;

        int mid = (lo + hi) / 2;
        int lb = b[l - 1], rb = b[r];

        if (v <= mid) {
            return left->rank(lb + 1, rb, v);
        } else {
            int leftCount = rb - lb;  // 왼쪽으로 간 원소 수
            int rlo = l - 1 - lb, rhi = r - rb;
            return leftCount + right->rank(rlo + 1, rhi, v);
        }
    }

    /**
     * 구간 [l, r]에서 값이 정확히 v인 원소의 개수
     */
    int count(int l, int r, int v) {
        return rank(l, r, v) - rank(l, r, v - 1);
    }
};

int main() {
    ios_base::sync_with_stdio(false);
    cin.tie(nullptr);

    vector<int> A = {5, 3, 1, 4, 2, 7, 6};
    int n = A.size();
    int lo = *min_element(A.begin(), A.end());
    int hi = *max_element(A.begin(), A.end());

    // 좌표 압축 (값이 매우 클 때 필요)
    vector<int> sorted_A = A;
    sort(sorted_A.begin(), sorted_A.end());
    sorted_A.erase(unique(sorted_A.begin(), sorted_A.end()), sorted_A.end());
    auto compress = [&](int x) {
        return lower_bound(sorted_A.begin(), sorted_A.end(), x) - sorted_A.begin() + 1;
    };
    vector<int> B(n);
    for (int i = 0; i < n; i++) B[i] = compress(A[i]);

    WaveletTree wt;
    wt.build(B.data(), B.data() + n, 1, sorted_A.size());

    // 쿼리 테스트 (1-indexed)
    cout << "A = {5, 3, 1, 4, 2, 7, 6}\n\n";

    // 구간 [2, 6] (A[1..5] = 3,1,4,2,7) 에서 2번째 최솟값
    int ans = wt.kth(2, 6, 2);
    cout << "kth(2, 6, 2) = " << sorted_A[ans - 1] << "\n"; // 기대: 2

    // 구간 [1, 5]에서 3번째 최솟값
    ans = wt.kth(1, 5, 3);
    cout << "kth(1, 5, 3) = " << sorted_A[ans - 1] << "\n"; // 기대: 3

    // 구간 [1, 7]에서 값 ≤ 4인 개수
    cout << "rank(1, 7, 4) = " << wt.rank(1, 7, compress(4)) << "\n"; // 기대: 4

    // 구간 [2, 6]에서 값이 3인 원소 개수
    cout << "count(2, 6, 3) = " << wt.count(2, 6, compress(3)) << "\n"; // 기대: 1

    return 0;
}
```

## 비트맵 기반 비재귀 구현 (메모리 효율화)

재귀 포인터 기반 구현 대신, 레벨별로 비트맵 배열을 연속 메모리에 저장하면 캐시 효율이 크게 향상된다.

```python
class WaveletTreeIterative:
    """
    비재귀 Wavelet Tree 구현 (Python)
    공간: O(N * log(max_val))
    쿼리: O(log(max_val))
    """
    def __init__(self, arr: list[int]):
        self.n = len(arr)
        if not arr:
            return

        # 좌표 압축
        self.vals = sorted(set(arr))
        self.val_to_idx = {v: i for i, v in enumerate(self.vals)}
        self.m = len(self.vals)  # 고유값 개수

        # log2(m) 레벨의 비트맵 저장
        self.levels = []  # levels[d][i] = 레벨 d의 i번째 원소가 왼쪽으로 가면 0, 오른쪽이면 1
        self.prefix = []  # prefix[d][i] = levels[d][0..i-1]에서 왼쪽으로 간 원소 수

        # 각 레벨에서 현재 배열 상태 유지
        cur = [self.val_to_idx[x] for x in arr]

        lo, hi = 0, self.m - 1
        while lo < hi:
            mid = (lo + hi) // 2
            nxt_left, nxt_right = [], []
            bits = []
            psum = [0]

            for v in cur:
                if v <= mid:
                    bits.append(0)
                    nxt_left.append(v)
                else:
                    bits.append(1)
                    nxt_right.append(v)
                psum.append(psum[-1] + (bits[-1] == 0))  # 왼쪽으로 간 수

            self.levels.append(bits)
            self.prefix.append(psum)
            cur = nxt_left + nxt_right
            lo_marker = lo  # 다음 레벨 범위 계산용
            lo, hi = lo, mid  # 실제로는 재귀 호출 대신 스택 사용 필요

            # 단순화를 위해 첫 레벨만 시뮬레이션
            break

    def kth_smallest(self, l: int, r: int, k: int) -> int:
        """
        구간 A[l..r] (0-indexed)에서 k번째 최솟값 (1-indexed k)
        완전한 구현은 재귀 버전 참조
        """
        lo, hi = 0, self.m - 1
        # 현재 레벨에서 [l, r] 범위의 인덱스 추적
        for d in range(len(self.levels)):
            if lo == hi:
                break
            mid = (lo + hi) // 2
            psum = self.prefix[d]
            # [l, r] 중 왼쪽으로 간 수
            left_count = psum[r + 1] - psum[l]
            if k <= left_count:
                # 왼쪽 서브트리로 이동
                l = psum[l]
                r = psum[r + 1] - 1
                hi = mid
            else:
                # 오른쪽 서브트리로 이동
                k -= left_count
                total_left_before_l = psum[l]
                total_left_before_r1 = psum[r + 1]
                l = (l - total_left_before_l)
                r = (r + 1 - total_left_before_r1) - 1
                # 오른쪽 구간의 오프셋 조정 필요 (레벨에 따라 다름)
                lo = mid + 1
        return self.vals[lo]


# 간단한 테스트
wt = WaveletTreeIterative([5, 3, 1, 4, 2, 7, 6])
print("고유값:", wt.vals)
print("좌표 압축:", [wt.val_to_idx[x] for x in [5, 3, 1, 4, 2, 7, 6]])
print("레벨 0 비트맵:", wt.levels[0] if wt.levels else "N/A")
print("레벨 0 prefix:", wt.prefix[0] if wt.prefix else "N/A")
```

## 핵심 쿼리 알고리즘 해설

### k번째 최솟값 `kth(l, r, k)` 알고리즘

재귀 방식으로 동작 원리를 이해해 보자:

```
A = [5, 3, 1, 4, 2, 7, 6], kth(1, 7, 3) = ?

레벨 0 (값 범위 [1,7], mid=4):
  비트맵: [1,1,1,0,1,0,0]  (1=오른쪽>4, 0=왼쪽≤4)
  prefix: [0,1,2,3,3,4,4,4]
  [1,7] 에서 왼쪽으로 간 수 = 4[3,1,4,2], 오른쪽 = 3[5,7,6]
  k=3 ≤ 4(왼쪽 수) → 왼쪽 서브트리로 이동
  새 [l',r'] = [1, 4] (왼쪽 서브트리 내에서의 인덱스)

레벨 1 (값 범위 [1,4], mid=2):
  해당 레벨의 [3,1,4,2] → 비트맵 [0,1,0,1] (0=≤2, 1=>2)
  prefix: [0,0,1,1,2]
  [1,4]에서 왼쪽 = 2[1,2], 오른쪽 = 2[3,4]
  k=3 > 2(왼쪽 수) → 오른쪽으로 이동, k=1
  새 [l',r'] = [1, 2] (오른쪽 내에서의 [3,4] 구간)

레벨 2 (값 범위 [3,4], mid=3):
  [3,4] → 비트맵 [1,0]
  k=1 ≤ 1(왼쪽 수=1) → 왼쪽 → 값=3

결과: 3 ✓ (A[1..7] = {1,2,3,4,5,6,7}의 3번째 = 3)
```

## Wavelet Tree의 실전 응용

**1. 문자열 검색 (FM-Index)**: BWT(Burrows-Wheeler Transform)와 Wavelet Tree를 결합하면 전체 게놈 데이터베이스에서 패턴 매칭을 수행할 수 있다. BWA(Burrows-Wheeler Aligner)가 이 원리를 활용한다.

**2. 역전 개수 세기**: 배열에서 `i < j`이지만 `A[i] > A[j]`인 쌍의 수를 O(N log N)에 계산할 수 있다.

**3. 범위 최빈값**: 구간에서 가장 많이 등장하는 값을 찾는 데 활용된다.

**4. 2D 범위 쿼리**: 2D 점 집합에서 특정 직사각형 영역 내의 k번째 점을 찾는 데 활용된다.

## 주의사항 및 팁

**1. 좌표 압축 필수**: 값의 범위가 크면 (`max_val = 10^9` 등) 반드시 좌표 압축으로 O(N)개의 고유 값으로 줄여야 한다.

**2. 영속 세그먼트 트리와 비교**: 같은 문제를 푸는 두 자료구조 중 Wavelet Tree는 **메모리 절약**에, 영속 세그먼트 트리는 **업데이트(포인트 업데이트 후 쿼리)** 가 필요한 경우에 유리하다.

**3. 경쟁 프로그래밍 활용**: PS에서는 C++ 기반의 재귀 포인터 구현 대신 비트맵 배열 기반 반복 구현이 캐시 효율 면에서 더 빠르다. 실제 제출 시 약 20~30% 속도 향상을 기대할 수 있다.

**4. 업데이트 불가**: 기본 Wavelet Tree는 **정적 자료구조**로 한 번 구축 후 원소 값 변경이 불가능하다. 동적 Wavelet Tree는 구현 복잡도가 크게 높아진다.

**5. 공간 복잡도 계산**: 값 범위가 `V`, 원소 수가 `N`일 때 총 공간은 `O(N log V)` 비트다. `N = 10^5`, `V = 10^5`라면 약 `10^5 * 17 = 1.7 * 10^6` 비트 ≈ 200KB 수준으로 상당히 작다.

## 참고 자료
- [Wavelet Tree - Wikipedia](https://en.wikipedia.org/wiki/Wavelet_tree)
- [Wavelet Trees Introduction - GeeksforGeeks](https://www.geeksforgeeks.org/dsa/wavelet-trees-introduction/)
- [Introduction to Wavelet Trees - Codeforces Blog](https://codeforces.com/blog/entry/52854)
- [Wavelet Trees for All - Gonzalo Navarro (PDF)](https://users.dcc.uchile.cl/~gnavarro/ps/cpm12.pdf)
