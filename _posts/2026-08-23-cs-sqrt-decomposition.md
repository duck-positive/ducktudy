---
layout: post
title: "제곱근 분해(Square Root Decomposition) 완전 정복: √N으로 범위 쿼리를 해결하는 범용 테크닉"
date: 2026-08-23
categories: [cs, computer-science]
tags: [sqrt-decomposition, range-query, block-decomposition, algorithm, competitive-programming, data-structure]
---

세그먼트 트리나 펜윅 트리를 구현하기에는 너무 복잡하거나, 해당 자료구조로 처리하기 어려운 연산이 필요할 때 선택할 수 있는 강력한 대안이 있다. 바로 **제곱근 분해(Square Root Decomposition, 또는 Sqrt Decomposition)**다.

이 기법은 배열을 크기 `√N`인 블록으로 나눠 각 블록의 집계 값을 별도로 관리함으로써, 범위 쿼리와 포인트 업데이트를 O(√N)에 처리한다. 세그먼트 트리의 O(log N)보다는 느리지만, **구현이 훨씬 간단**하고 세그먼트 트리로 처리하기 까다로운 다양한 문제에 유연하게 적용할 수 있다.

---

## 왜 제곱근 분해인가?

### 직관적인 시간 복잡도 도출

배열의 크기를 N이라 하고, 이를 크기 B인 블록으로 나눈다고 하자.

- **쿼리**: 최대 `B-1`개의 좌측 끝 단일 원소 + `N/B`개의 완전 블록 + `B-1`개의 우측 끝 단일 원소를 처리 → O(B + N/B)
- **업데이트**: 해당 원소와 해당 원소가 속한 블록의 집계 값만 갱신 → O(1)

쿼리 시간 `T(B) = B + N/B`를 최소화하려면 미분을 0으로 놓으면 된다.

```
dT/dB = 1 - N/B² = 0
B² = N
B = √N
```

B = √N일 때 T(B) = √N + √N = 2√N = O(√N)으로 최적이다. 이것이 이 기법을 "제곱근 분해"라 부르는 이유다.

### 세그먼트 트리 대비 장점

| 특성 | 세그먼트 트리 | 제곱근 분해 |
|---|---|---|
| 쿼리/업데이트 | O(log N) | O(√N) |
| 구현 복잡도 | 높음 | 낮음 |
| 범용성 | 특정 연산에 최적 | 다양한 연산 지원 |
| 오프라인 문제 | 부분적 지원 | Mo's Algorithm과 결합 |

N = 100,000일 때 √N ≈ 316, log N ≈ 17이므로 세그먼트 트리가 약 18배 빠르다. 그러나 상수 배 차이이며, O(N√N) 솔루션도 실전에서 충분히 통과하는 경우가 많다.

---

## 기본 구현: 범위 합 쿼리

### 배열 분할과 블록 초기화

```cpp
#include <bits/stdc++.h>
using namespace std;

const int MAXN = 100005;
int arr[MAXN];
long long block[320];  // 블록 크기 √N ≈ 316
int n, B;             // B = 블록 크기

void build(int a[], int sz) {
    n = sz;
    B = (int)sqrt(n) + 1;
    for (int i = 0; i < n; i++) {
        arr[i] = a[i];
        block[i / B] += a[i];
    }
}

// 포인트 업데이트: arr[idx] = val
void update(int idx, int val) {
    block[idx / B] -= arr[idx];
    arr[idx] = val;
    block[idx / B] += val;
}

// 범위 합 쿼리: [l, r]의 합
long long query(int l, int r) {
    long long sum = 0;
    int bl = l / B, br = r / B;

    if (bl == br) {
        // 같은 블록 안에 있으면 직접 합산
        for (int i = l; i <= r; i++) sum += arr[i];
    } else {
        // 왼쪽 끝 부분
        for (int i = l; i < (bl + 1) * B; i++) sum += arr[i];
        // 완전히 포함되는 블록들
        for (int b = bl + 1; b < br; b++) sum += block[b];
        // 오른쪽 끝 부분
        for (int i = br * B; i <= r; i++) sum += arr[i];
    }
    return sum;
}

int main() {
    int a[] = {1, 3, 5, 7, 9, 11, 13, 15, 17, 19};
    build(a, 10);

    cout << query(2, 7) << "\n";  // 5+7+9+11+13+15 = 60
    update(4, 100);               // arr[4] = 100
    cout << query(2, 7) << "\n";  // 5+7+100+11+13+15 = 151
    return 0;
}
```

### 시각화: 블록 구조

```
배열:  [1][3][5][7][9][11][13][15][17][19]
인덱스: 0   1  2  3  4   5   6   7   8   9
블록:  [  블록0  ][   블록1    ][    블록2  ]
       (크기 3일 때 예시)
블록합: 9        33            36
```

---

## 심화 구현: 범위 업데이트 + 범위 최댓값 쿼리

레이지 프로파게이션과 유사하게, 제곱근 분해에서도 범위 업데이트를 블록 단위로 지연 처리할 수 있다.

```python
import math

class SqrtDecomposition:
    def __init__(self, arr):
        self.n = len(arr)
        self.B = int(math.isqrt(self.n)) + 1
        self.arr = arr[:]
        self.block_max = [float('-inf')] * ((self.n + self.B - 1) // self.B)
        self.lazy = [0] * len(self.block_max)  # 범위 더하기 지연 처리

        for i in range(self.n):
            b = i // self.B
            self.block_max[b] = max(self.block_max[b], self.arr[i])

    def _rebuild_block(self, b):
        """블록 b의 최댓값을 arr에서 재계산"""
        start = b * self.B
        end = min(start + self.B, self.n)
        self.block_max[b] = max(self.arr[i] for i in range(start, end))

    def range_add(self, l, r, val):
        """[l, r] 구간에 val을 더한다"""
        bl, br = l // self.B, r // self.B

        if bl == br:
            for i in range(l, r + 1):
                self.arr[i] += val
            self._rebuild_block(bl)
        else:
            # 왼쪽 끝 부분
            for i in range(l, (bl + 1) * self.B):
                self.arr[i] += val
            self._rebuild_block(bl)

            # 완전히 포함된 블록: lazy에 더함
            for b in range(bl + 1, br):
                self.lazy[b] += val
                self.block_max[b] += val  # 최댓값도 일괄 증가

            # 오른쪽 끝 부분
            for i in range(br * self.B, r + 1):
                self.arr[i] += val
            self._rebuild_block(br)

    def range_max(self, l, r):
        """[l, r] 구간의 최댓값을 반환한다"""
        bl, br = l // self.B, r // self.B
        result = float('-inf')

        if bl == br:
            for i in range(l, r + 1):
                result = max(result, self.arr[i] + self.lazy[bl])
        else:
            # 왼쪽 끝
            for i in range(l, (bl + 1) * self.B):
                result = max(result, self.arr[i] + self.lazy[bl])
            # 완전한 블록
            for b in range(bl + 1, br):
                result = max(result, self.block_max[b])
            # 오른쪽 끝
            for i in range(br * self.B, r + 1):
                result = max(result, self.arr[i] + self.lazy[br])

        return result


# 사용 예시
arr = [3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5]
sq = SqrtDecomposition(arr)

print(sq.range_max(1, 8))   # 9
sq.range_add(2, 7, 10)      # [2, 7] 구간에 +10
print(sq.range_max(1, 8))   # 19 (9+10=19)
print(sq.range_max(0, 2))   # max(3, 1, 14) = 14
```

---

## 제곱근 분해의 다양한 응용

### 1. 정렬된 블록으로 k번째 원소 탐색

각 블록을 정렬된 상태로 유지하면, 특정 값보다 작은 원소의 개수를 O(√N log √N)에 계산할 수 있다.

```python
import math
import bisect

class SortedBlock:
    """각 블록을 정렬 상태로 유지하는 제곱근 분해"""

    def __init__(self, arr):
        self.n = len(arr)
        self.B = int(math.isqrt(self.n)) + 1
        self.arr = arr[:]
        self.sorted_blocks = []
        for b in range(0, self.n, self.B):
            self.sorted_blocks.append(sorted(arr[b:b + self.B]))

    def count_less_than(self, l, r, val):
        """[l, r] 구간에서 val보다 작은 원소의 수"""
        bl, br = l // self.B, r // self.B
        count = 0

        if bl == br:
            for i in range(l, r + 1):
                if self.arr[i] < val:
                    count += 1
        else:
            for i in range(l, (bl + 1) * self.B):
                if self.arr[i] < val:
                    count += 1
            for b in range(bl + 1, br):
                count += bisect.bisect_left(self.sorted_blocks[b], val)
            for i in range(br * self.B, r + 1):
                if self.arr[i] < val:
                    count += 1
        return count

    def update(self, idx, val):
        b = idx // self.B
        old = self.arr[idx]
        pos = bisect.bisect_left(self.sorted_blocks[b], old)
        self.sorted_blocks[b].pop(pos)
        bisect.insort(self.sorted_blocks[b], val)
        self.arr[idx] = val


arr = [5, 3, 1, 8, 2, 7, 4, 6, 9, 0]
sb = SortedBlock(arr)
print(sb.count_less_than(1, 8, 5))  # [1,8] 구간에서 5보다 작은 수: 3,1,2,4 = 4개
```

### 2. Mo's Algorithm과 제곱근 분해

**Mo's Algorithm**은 오프라인 범위 쿼리를 제곱근 분해를 활용해 O((N + Q)√N)에 처리하는 기법이다. 쿼리를 블록 기준으로 정렬해 포인터 이동 횟수를 최소화한다.

---

## 제곱근 분해가 적합한 문제 유형

1. **범위 합/최솟값/최댓값 + 포인트 업데이트**: 구현이 간단해 빠른 코딩이 필요할 때
2. **범위 쿼리 + 범위 업데이트**: 레이지 프로파게이션 없이 처리 가능
3. **오프라인 쿼리**: Mo's Algorithm으로 복잡한 범위 쿼리 처리
4. **k번째 원소 탐색**: 정렬된 블록과 이분 탐색 결합
5. **2D 배열 쿼리**: 2D 세그먼트 트리 대신 간단한 구현
6. **세그먼트 트리로 표현하기 어려운 연산**: 블록 단위 재계산으로 대응

---

## 주의사항과 팁

### 1. 블록 크기 최적화

블록 크기를 반드시 `int(sqrt(N))`으로 고정할 필요는 없다. 쿼리 수 Q와 업데이트 수 U의 비율에 따라 최적 블록 크기가 달라진다.

- 쿼리가 많고 업데이트가 적은 경우: 블록을 더 크게
- 업데이트가 많고 쿼리가 적은 경우: 블록을 더 작게

일반적으로 `B = max(1, (int)(sqrt(N * log2(N))))`처럼 상황에 맞게 조정한다.

### 2. 경계 처리 주의

범위의 왼쪽 끝과 오른쪽 끝이 같은 블록에 속하는 경우(`bl == br`)를 반드시 따로 처리해야 한다. 이를 빠뜨리면 완전한 블록으로 취급되어 잘못된 결과가 나온다.

### 3. 블록 재계산 비용

범위 업데이트 후 부분 블록의 집계 값을 재계산할 때, 블록 전체를 순회해야 한다. 이 비용이 O(B) = O(√N)이므로 전체 복잡도는 유지된다.

### 4. 정수 제곱근 계산

Python의 `math.isqrt()`나 C++의 `(int)sqrt(n)`은 부동소수점 오차로 정확하지 않을 수 있다. `while (B * B < n) B++;`처럼 정수 연산으로 확인하는 것이 안전하다.

### 5. 메모리 지역성

블록 크기를 캐시 라인(64바이트) 크기에 맞게 조정하면 실제 성능이 향상될 수 있다. `int` 배열이면 16개(64 / 4), `long long`이면 8개(64 / 8)가 한 캐시 라인에 들어간다.

---

## 성능 비교: 실전에서의 판단 기준

| N | O(√N) | O(log N) | 비율 |
|---|---|---|---|
| 1,000 | 31.6 | 10.0 | 3.2× |
| 10,000 | 100 | 13.3 | 7.5× |
| 100,000 | 316 | 16.6 | 19× |
| 1,000,000 | 1,000 | 19.9 | 50× |

N이 클수록 세그먼트 트리와의 차이가 커진다. 그러나 제곱근 분해는 구현 복잡도가 낮고 상수 계수가 작아, 시간 제한이 여유롭거나 N이 작은 경우에는 충분히 실용적이다. 특히 Mo's Algorithm처럼 제곱근 분해만으로 표현할 수 있는 테크닉은 세그먼트 트리로 대체하기 어렵다.

---

## 마무리

제곱근 분해는 "복잡한 자료구조 없이 O(√N)으로 범위 쿼리를 해결하고 싶을 때" 가장 먼저 떠올려야 할 도구다. 구현이 직관적이고, 다양한 연산과 유연하게 결합할 수 있으며, Mo's Algorithm과 같은 고급 테크닉의 기반이 된다. 세그먼트 트리와 펜윅 트리만 알고 있다면 제곱근 분해를 도구함에 추가해 문제 해결 범위를 크게 넓힐 수 있다.

## 참고 자료
- [GeeksforGeeks: Square Root (Sqrt) Decomposition Algorithm](https://www.geeksforgeeks.org/dsa/square-root-sqrt-decomposition-algorithm/)
- [Codeforces: Tutorial - Square root decomposition and applications](https://codeforces.com/blog/entry/83248)
- [CP-Algorithms: Sqrt Decomposition](https://cp-algorithms.com/data_structures/sqrt_decomposition.html)
- [Wikipedia: Block decomposition (data structures)](https://en.wikipedia.org/wiki/Block_decomposition_(data_structures))
