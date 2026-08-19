---
layout: post
title: "최장 증가 부분 수열(LIS) 완전 정복: O(N log N)부터 CDQ 분할 정복까지"
date: 2026-08-15
categories: [cs, computer-science]
tags: [lis, longest-increasing-subsequence, patience-sorting, dynamic-programming, cdq, divide-conquer, algorithm]
---

최장 증가 부분 수열(LIS, Longest Increasing Subsequence)은 컴퓨터 과학에서 가장 고전적이면서도 심오한 문제 중 하나입니다. 주어진 수열에서 원소들을 순서를 유지하면서 선택했을 때, 선택된 원소들이 순증가하는 가장 긴 부분 수열을 찾는 문제입니다. 단순해 보이는 이 문제는 O(N²) 동적 프로그래밍 풀이부터, 카드 게임에서 영감을 얻은 O(N log N) 인내심 정렬(Patience Sorting), 그리고 2D 문제로의 확장까지 아름다운 이론적 체계를 형성합니다.

## 문제 정의와 O(N²) DP 풀이

배열 `A = [A₀, A₁, ..., A_{N-1}]`이 주어질 때, 인덱스 `i₁ < i₂ < ... < iₖ`를 선택해서 `A[i₁] < A[i₂] < ... < A[iₖ]`인 가장 긴 부분 수열의 길이를 구합니다.

**예시**: `A = [3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5]`  
LIS = `[1, 4, 5, 9]` 또는 `[1, 2, 6]` 또는 `[1, 2, 5]` → 길이 4

### 기본 O(N²) DP

```python
def lis_n2(arr):
    """O(N²) DP: dp[i] = arr[i]로 끝나는 LIS 길이"""
    n = len(arr)
    dp = [1] * n
    parent = [-1] * n  # 경로 복원을 위한 부모 인덱스

    for i in range(1, n):
        for j in range(i):
            if arr[j] < arr[i] and dp[j] + 1 > dp[i]:
                dp[i] = dp[j] + 1
                parent[i] = j

    # LIS 길이와 끝 인덱스 찾기
    lis_len = max(dp)
    end_idx = dp.index(lis_len)

    # 경로 복원
    path = []
    idx = end_idx
    while idx != -1:
        path.append(arr[idx])
        idx = parent[idx]
    path.reverse()

    return lis_len, path


arr = [3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5]
length, sequence = lis_n2(arr)
print(f"LIS 길이: {length}, 수열: {sequence}")
# LIS 길이: 4, 수열: [1, 4, 5, 9]
```

O(N²) 풀이는 이해하기 쉽지만 N이 10만을 넘으면 10^10 연산이 필요해 비현실적입니다.

## 인내심 정렬(Patience Sorting)과 O(N log N) 알고리즘

### 카드 게임에서 탄생한 알고리즘

인내심 정렬은 솔리테어(Solitaire) 카드 게임의 규칙에서 영감을 받았습니다.

**규칙**: 
1. 카드 더미(pile)들을 왼쪽부터 유지한다.
2. 새 카드를 놓을 때는, 새 카드보다 크거나 같은 맨 위 카드를 가진 더미 중 **가장 왼쪽** 더미 위에 놓는다.
3. 그런 더미가 없으면 새 더미를 오른쪽에 만든다.

이 과정을 마친 뒤 **더미의 수 = LIS의 길이**가 됩니다. 각 더미에서 가장 위 카드를 binary search로 찾으면 O(N log N)이 됩니다.

**핵심 불변식**: 더미의 맨 위 카드 배열은 항상 비내림차순을 유지합니다. 따라서 이분 탐색이 가능합니다.

```python
import bisect

def lis_patience_sort(arr):
    """
    O(N log N) LIS 길이 계산 — 인내심 정렬 기반
    tails[i] = 길이 (i+1)의 LIS를 끝낼 수 있는 최솟값
    """
    tails = []  # 각 더미의 맨 위 카드(최솟값)

    for x in arr:
        # tails에서 x 이상인 가장 왼쪽 위치를 이분 탐색
        pos = bisect.bisect_left(tails, x)

        if pos == len(tails):
            tails.append(x)   # 새 더미 생성
        else:
            tails[pos] = x    # 기존 더미의 맨 위 카드 교체

    return len(tails)


# 예시
arr = [3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5]
print(f"LIS 길이: {lis_patience_sort(arr)}")  # 4

# 더 긴 예시로 성능 확인
import random
import time
large_arr = random.sample(range(1, 10**6), 100000)

start = time.time()
result = lis_patience_sort(large_arr)
print(f"N=100,000: LIS={result}, 소요={time.time()-start:.4f}s")
```

### 핵심 불변식 증명

`tails[i]`가 증가 순서를 유지하는 이유:

1. `tails[pos] = x`로 교체될 때, `x < tails[pos]`이므로 `tails[pos-1] < x < tails[pos]`가 유지됩니다.
2. 새 더미를 추가할 때(`pos == len(tails)`) `x > tails[-1]`이 보장됩니다.

이로써 `tails`는 항상 엄격하게 증가하는 배열로 유지됩니다.

### LIS 경로 복원 (O(N log N))

길이뿐 아니라 실제 LIS 수열을 복원하려면 각 원소가 삽입된 더미 위치를 기록해야 합니다.

```python
import bisect

def lis_with_reconstruction(arr):
    """O(N log N) LIS 길이 계산 + 경로 복원"""
    n = len(arr)
    tails = []       # tails[i]: 길이 (i+1) LIS의 최솟값 원소
    dp = [0] * n     # dp[i]: arr[i]로 끝나는 LIS 길이
    parent = [-1] * n
    tail_idx = []    # tail_idx[i]: tails[i]를 구성하는 원소의 인덱스

    for i, x in enumerate(arr):
        pos = bisect.bisect_left(tails, x)
        dp[i] = pos + 1

        if pos == len(tails):
            tails.append(x)
            tail_idx.append(i)
        else:
            tails[pos] = x
            tail_idx[pos] = i

        # 이전 더미의 마지막 원소를 부모로 설정
        if pos > 0:
            parent[i] = tail_idx[pos - 1]

    # 경로 복원: 마지막 더미의 원소부터 역추적
    lis_len = len(tails)
    path = []
    cur = tail_idx[-1]
    while cur != -1:
        path.append(arr[cur])
        cur = parent[cur]
    path.reverse()

    return lis_len, path


arr = [3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5]
length, seq = lis_with_reconstruction(arr)
print(f"LIS 길이: {length}, 수열: {seq}")
# LIS 길이: 4, 수열: [1, 4, 5, 9]

# 검증
arr2 = [10, 9, 2, 5, 3, 7, 101, 18]
length2, seq2 = lis_with_reconstruction(arr2)
print(f"LIS 길이: {length2}, 수열: {seq2}")
# LIS 길이: 4, 수열: [2, 3, 7, 101] 또는 [2, 5, 7, 18] 등
```

## CDQ 분할 정복과 2D LIS 문제

### 2D LIS: 두 기준으로 동시에 증가하는 부분 수열

2D LIS는 점 `(x, y)`들의 집합에서, x와 y가 동시에 순증가하는 가장 긴 부분 수열을 찾는 문제입니다. 1D LIS를 기반으로 하되, 두 번째 차원 처리가 추가됩니다.

**CDQ 분할 정복(CDQ Divide and Conquer)**은 이 문제를 O(N log² N)에 해결합니다.

```python
def lis_2d_cdq(points):
    """
    2D LIS: CDQ 분할 정복
    points: [(x, y)] 쌍의 리스트
    모든 점 (x_i, y_i)에 대해 i < j이고 x_i < x_j, y_i < y_j인 최장 체인 반환
    """
    n = len(points)
    # 인덱스와 함께 저장, x 기준 정렬
    indexed = sorted(enumerate(points), key=lambda t: (t[1][0], t[1][1]))
    dp = [1] * n

    def bit_query(tree, i):
        """Fenwick Tree 최대값 쿼리"""
        res = 0
        while i > 0:
            res = max(res, tree[i])
            i -= i & (-i)
        return res

    def bit_update(tree, i, val, size):
        """Fenwick Tree 업데이트"""
        while i <= size:
            tree[i] = max(tree[i], val)
            i += i & (-i)

    def bit_clear(tree, i, size):
        """사용한 Fenwick Tree 위치 초기화"""
        while i <= size:
            if tree[i] == 0:
                break
            tree[i] = 0
            i += i & (-i)

    # y값 좌표 압축
    ys = sorted(set(y for _, (x, y) in indexed))
    y_rank = {v: i + 1 for i, v in enumerate(ys)}
    max_y = len(ys)

    bit = [0] * (max_y + 1)

    def cdq(items):
        if len(items) <= 1:
            return

        mid = len(items) // 2
        left, right = items[:mid], items[mid:]

        cdq(left)  # 좌반부 먼저 해결

        # 좌반부의 결과로 우반부 dp 갱신
        # x 기준으로 이미 정렬되어 있으므로 y만 처리
        li, ri = 0, 0
        left_sorted = sorted(left, key=lambda t: t[1][1])  # y 기준 정렬
        right_sorted = sorted(right, key=lambda t: t[1][1])

        for orig_idx, (_, (rx, ry)) in right_sorted:
            # 좌반부에서 y < ry인 모든 점의 dp 최대값 쿼리
            yr = y_rank[ry]
            val = bit_query(bit, yr - 1)
            if val + 1 > dp[orig_idx]:
                dp[orig_idx] = val + 1

        # Fenwick Tree 초기화
        for orig_idx, (_, (lx, ly)) in left_sorted:
            bit_clear(bit, y_rank[ly], max_y)

        # 우반부 재귀
        cdq(right)

    # CDQ 실행: x 기준으로 정렬된 items로 분할 정복
    cdq(list(indexed))

    return max(dp)


# 예시
points = [(1, 3), (2, 1), (3, 4), (4, 2), (5, 5), (6, 6)]
print(f"2D LIS 길이: {lis_2d_cdq(points)}")  # 4: (1,3)→(3,4)→(5,5)→(6,6) 또는 (2,1)→(4,2)→(5,5)→(6,6)
```

## Dilworth 정리와 LIS의 쌍대성

LIS에는 아름다운 수학적 성질이 있습니다.

**Dilworth 정리**: 부분 순서 집합(poset)에서 최대 반사슬(antichain)의 크기는 최소 체인 분할의 크기와 같습니다.

LIS 문제에 적용하면:
- **LIS 길이** = 수열을 비증가 부분 수열로 분할하는 데 필요한 최소 개수 (Patience Sorting의 더미 수!)
- **LDS(최장 감소 부분 수열) 길이** = 수열을 증가 부분 수열로 분할하는 데 필요한 최소 개수

이를 활용한 응용:
- **인내심 정렬의 더미 수 = LIS 길이** (이분 탐색과 결합해 O(N log N) 달성)
- 2D 점에서의 LIS = 2D Dilworth 정리로 확장 가능

## 주의사항과 팁

**1. 엄격 증가 vs 비감소**  
`bisect_left`는 엄격한 증가 LIS를 구합니다. 비감소(≤) LIS를 구하려면 `bisect_right`를 사용하세요. 헷갈리기 쉬운 부분이니 문제 조건을 반드시 확인하세요.

```python
import bisect

def lis_non_decreasing(arr):
    tails = []
    for x in arr:
        pos = bisect.bisect_right(tails, x)  # <=를 허용
        if pos == len(tails):
            tails.append(x)
        else:
            tails[pos] = x
    return len(tails)
```

**2. 경로 복원의 복잡성**  
경로 복원은 단순 `tails` 배열만으로는 올바르게 되지 않습니다. 위 구현처럼 각 원소가 삽입된 위치와 이전 더미의 마지막 인덱스를 추적해야 합니다. 잘못된 경로 복원은 길이는 맞지만 수열 자체가 틀리는 버그를 유발합니다.

**3. 좌표 압축과 세그먼트 트리 / BIT 활용**  
2D LIS처럼 값 범위가 크고 복잡한 경우, Fenwick Tree(BIT)와 좌표 압축을 결합하면 효율적인 구현이 가능합니다. CDQ 분할 정복은 이를 O(N log² N)에 처리하는 강력한 도구입니다.

**4. LIS = 편집 거리의 특수 케이스**  
LIS 길이 = N - LCS(A, sorted(A))임을 활용하면, 두 수열 간 LCS 알고리즘의 특수 케이스로도 풀 수 있습니다. 단 이 방법은 중복 원소가 없을 때만 유효합니다.

**5. 온라인 vs 오프라인**  
tails 배열 기반 O(N log N) 알고리즘은 배열을 왼쪽부터 오른쪽으로 순차 처리하는 **온라인 알고리즘**입니다. 원소가 동적으로 추가될 때도 그대로 적용할 수 있습니다.

**6. 응용 문제 유형**  
- **가장 긴 바이토닉 부분 수열**: 앞에서의 LIS + 뒤에서의 LIS 합산.
- **LIS를 이용한 Patience Sorting 전체 구현**: 카드 게임 시뮬레이션.
- **2D 문제에서 LIS 변환**: 좌표로 정렬 후 1D LIS 문제로 환원.
- **최장 공통 부분 수열(LCS) 연계**: 한 수열이 정렬된 고유 원소일 때 LCS → LIS 변환.

LIS는 단순한 알고리즘 문제를 넘어 Dilworth 정리, 인내심 정렬, RSK 대응(Robinson-Schensted-Knuth)으로 이어지는 깊은 수학적 체계의 시작점입니다. O(N log N) 인내심 정렬의 동작 원리를 손으로 직접 시뮬레이션해보면, 알고리즘이 얼마나 우아하게 최적성을 보장하는지 체감할 수 있습니다.

## 참고 자료
- [CP-Algorithms: Longest Increasing Subsequence (O(N log N) 포함)](https://raw.githubusercontent.com/cp-algorithms/cp-algorithms/main/src/dynamic_programming/longest_increasing_subsequence.md)
- [trekhleb/javascript-algorithms — LIS README (다중 언어 구현)](https://github.com/trekhleb/javascript-algorithms/blob/master/src/algorithms/sets/longest-increasing-subsequence/README.md)
- [williamfiset/Algorithms — LIS Java 구현 및 영상 강의](https://github.com/williamfiset/Algorithms)
