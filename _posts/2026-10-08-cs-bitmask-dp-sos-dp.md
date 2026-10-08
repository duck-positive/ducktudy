---
layout: post
title: "비트마스크 DP와 SOS DP 완전 정복: 부분집합을 정복하는 비트 기반 동적 프로그래밍"
date: 2026-10-08
categories: [cs, computer-science]
tags: [bitmask-dp, SOS-dp, subset, dynamic-programming, TSP, competitive-programming, algorithm]
---

`n`개의 원소로 이루어진 집합에서 부분집합은 2ⁿ개입니다. n=20이면 약 100만 개, n=30이면 10억 개입니다. 이 모든 부분집합을 다루는 알고리즘을 **O(n · 2ⁿ)**에 처리하는 마법 같은 기법이 있습니다. 바로 **비트마스크 DP(Bitmask Dynamic Programming)**와 **SOS DP(Sum Over Subsets DP)**입니다. 이 글에서는 두 기법의 원리를 수학적으로 이해하고, 대표적인 문제인 외판원 순회(TSP)와 부분집합 합 최적화를 직접 구현합니다.

---

## 개념 설명: 비트마스크로 집합을 표현하는 방법

### 집합과 비트마스크의 대응

n개의 원소를 가진 전체 집합 U = {0, 1, 2, ..., n-1}의 모든 부분집합은 n비트 정수 하나로 표현할 수 있습니다.

- 비트 i가 **1**이면: 원소 i가 집합에 **포함**됨
- 비트 i가 **0**이면: 원소 i가 집합에 **미포함**

예시 (n=4, 원소 {0,1,2,3}):

```
마스크  이진수  집합
  0    0000   {}
  1    0001   {0}
  2    0010   {1}
  3    0011   {0, 1}
  5    0101   {0, 2}
 15    1111   {0, 1, 2, 3}
```

### 핵심 비트 연산

```python
n = 4  # 원소 수

# 원소 i 추가: mask | (1 << i)
mask = 0b0011  # {0, 1}
mask |= (1 << 2)  # → 0b0111 = {0, 1, 2}

# 원소 i 제거: mask & ~(1 << i)
mask &= ~(1 << 0)  # → 0b0110 = {1, 2}

# 원소 i 포함 여부: (mask >> i) & 1
print((mask >> 1) & 1)  # 1 (원소 1 포함)
print((mask >> 0) & 1)  # 0 (원소 0 미포함)

# 마스크의 부분집합 열거 (가장 중요한 연산!)
mask = 0b1010  # {1, 3}
sub = mask
while sub > 0:
    print(bin(sub))  # 0b1010, 0b1000, 0b0010
    sub = (sub - 1) & mask  # 다음 부분집합
    # 수행 횟수: 2^popcount(mask) = 2^2 = 4회 (공집합 포함)
```

부분집합 열거의 시간 복잡도: 마스크가 k비트 켜져 있으면 2ᵏ번. 전체 2ⁿ개 마스크에 대해 합산하면 **Σ C(n,k)·2ᵏ = 3ⁿ**.

---

## 왜 필요한가: 조합 최적화 문제의 핵심

n이 작을 때(대개 n ≤ 20) 모든 부분집합의 상태를 DP 테이블에 저장하면, 지수 시간 알고리즘을 훨씬 효율적으로 만들 수 있습니다.

**대표 문제**: 외판원 순회(Travelling Salesman Problem, TSP)
- 완전 탐색: O(n!) — n=20이면 2.4 × 10¹⁸ 연산
- 비트마스크 DP: O(2ⁿ · n²) — n=20이면 약 4 × 10⁸ 연산 (충분히 실용적)

---

## 실제 구현 예제 1: 외판원 순회(TSP) — 비트마스크 DP

**문제**: n개의 도시를 모두 방문하고 출발지로 돌아오는 최소 비용 경로를 구하라.

**DP 정의**: `dp[mask][v]` = 집합 mask에 속한 도시들을 모두 방문하고 현재 도시 v에 있을 때의 최소 이동 비용.

**점화식**:
```
dp[mask | (1 << next)][next] = min(dp[mask][v] + dist[v][next])
  (next ∉ mask일 때)
```

**기저 조건**: `dp[1][0] = 0` (도시 0에서 출발, 도시 0만 방문)

**최종 답**: `min(dp[(1<<n)-1][v] + dist[v][0])` (모든 도시 방문 후 0으로 복귀)

```cpp
#include <bits/stdc++.h>
using namespace std;

const int INF = 1e9;

int tsp(int n, vector<vector<int>>& dist) {
    // dp[mask][v]: mask에 포함된 도시를 방문하고 v에 있는 최소 비용
    vector<vector<int>> dp(1 << n, vector<int>(n, INF));

    dp[1][0] = 0;  // 도시 0에서 출발 (비트 0이 켜진 상태)

    // mask를 작은 것부터 채워 나감 (방문 도시 수 증가 순서)
    for (int mask = 1; mask < (1 << n); mask++) {
        for (int v = 0; v < n; v++) {
            if (dp[mask][v] == INF) continue;
            if (!(mask & (1 << v))) continue;  // v가 mask에 없으면 스킵

            // 다음 방문 도시 선택
            for (int next = 0; next < n; next++) {
                if (mask & (1 << next)) continue;  // 이미 방문함
                if (dist[v][next] == INF) continue; // 연결 없음

                int new_mask = mask | (1 << next);
                int new_cost = dp[mask][v] + dist[v][next];
                if (new_cost < dp[new_mask][next]) {
                    dp[new_mask][next] = new_cost;
                }
            }
        }
    }

    // 모든 도시 방문 후 출발지(0)로 복귀
    int full_mask = (1 << n) - 1;
    int answer = INF;
    for (int v = 1; v < n; v++) {
        if (dp[full_mask][v] != INF && dist[v][0] != INF) {
            answer = min(answer, dp[full_mask][v] + dist[v][0]);
        }
    }
    return answer;
}


// 경로 복원을 포함한 완전한 구현
pair<int, vector<int>> tsp_with_path(int n, vector<vector<int>>& dist) {
    vector<vector<int>> dp(1 << n, vector<int>(n, INF));
    vector<vector<int>> parent(1 << n, vector<int>(n, -1));

    dp[1][0] = 0;

    for (int mask = 1; mask < (1 << n); mask++) {
        for (int v = 0; v < n; v++) {
            if (dp[mask][v] == INF || !(mask & (1 << v))) continue;
            for (int next = 0; next < n; next++) {
                if (mask & (1 << next)) continue;
                int new_mask = mask | (1 << next);
                int new_cost = dp[mask][v] + dist[v][next];
                if (new_cost < dp[new_mask][next]) {
                    dp[new_mask][next] = new_cost;
                    parent[new_mask][next] = v;  // 이전 도시 기록
                }
            }
        }
    }

    int full_mask = (1 << n) - 1;
    int best_cost = INF, last = -1;
    for (int v = 1; v < n; v++) {
        int cost = dp[full_mask][v] + dist[v][0];
        if (cost < best_cost) { best_cost = cost; last = v; }
    }

    // 경로 복원 (역추적)
    vector<int> path;
    int mask = full_mask, cur = last;
    while (cur != -1) {
        path.push_back(cur);
        int prev = parent[mask][cur];
        mask ^= (1 << cur);
        cur = prev;
    }
    path.push_back(0);
    reverse(path.begin(), path.end());

    return {best_cost, path};
}


int main() {
    int n = 5;
    // 5개 도시, 거리 행렬 (대칭)
    vector<vector<int>> dist = {
        {  0, 10, 15, 20, 25},
        { 10,  0, 35, 25, 30},
        { 15, 35,  0, 30, 10},
        { 20, 25, 30,  0, 15},
        { 25, 30, 10, 15,  0}
    };

    auto [cost, path] = tsp_with_path(n, dist);
    cout << "최소 비용: " << cost << "\n";  // 출력: 75
    cout << "경로: ";
    for (int city : path) cout << city << " ";
    cout << "→ 0\n";  // 0 → 1 → 3 → 4 → 2 → 0 등

    return 0;
}
```

시간 복잡도: O(2ⁿ · n²), 공간 복잡도: O(2ⁿ · n)  
n=20일 때: 약 4억 연산, 메모리 4GB → n=20까지 실용적 (n=25가 한계)

---

## 실제 구현 예제 2: SOS DP (Sum Over Subsets)

**문제**: 배열 A[0..2ⁿ-1]이 주어질 때, 모든 마스크 x에 대해 F(x) = Σ A[i] (i가 x의 부분집합인 i 전체 합)를 구하라.

**나이브 접근**: 각 x마다 부분집합을 열거하면 O(3ⁿ) (모든 마스크의 부분집합 수의 합).

**SOS DP**: O(n · 2ⁿ)에 가능합니다.

**핵심 아이디어**: 다차원 누적합으로 생각합니다. n개의 비트를 n개의 차원으로 보면, 비트 하나씩 처리하면서 "이 비트가 켜진 상태로 전이하는 것"을 누적합 방식으로 계산합니다.

```
dim 0: F[mask] += F[mask ^ (1<<0)]  (비트 0을 켠 것의 값을 더함)
dim 1: F[mask] += F[mask ^ (1<<1)]
...
dim n-1: F[mask] += F[mask ^ (1<<(n-1))]
```

각 차원 처리 후 F[mask]는 "비트 0..k 위치에서 0 또는 1을 가질 수 있는 모든 마스크의 원래 A값 합"이 됩니다.

```python
def sos_dp(n: int, A: list[int]) -> list[int]:
    """
    A[0..2^n-1]에 대해 F[mask] = sum(A[i] for all i ⊆ mask)를 계산.
    시간: O(n * 2^n), 공간: O(2^n)
    """
    F = A[:]  # 복사

    for bit in range(n):          # n개의 비트 차원 처리
        for mask in range(1 << n):
            if mask & (1 << bit): # 이 비트가 켜져 있으면
                # 이 비트를 끈 마스크(= 하나 작은 부분집합)의 값을 더함
                F[mask] += F[mask ^ (1 << bit)]

    return F


# 모비우스 변환 (역변환): SOS 결과에서 원래 A 복원
def inverse_sos(n: int, F: list[int]) -> list[int]:
    A = F[:]
    for bit in range(n):
        for mask in range(1 << n):
            if mask & (1 << bit):
                A[mask] -= A[mask ^ (1 << bit)]
    return A


# ── 응용: XOR 합산 (서로소 그룹 최대 XOR 합) ──
def max_xor_subset_sum(nums: list[int]) -> int:
    """
    리스트 nums에서 임의의 부분집합을 XOR했을 때 최대값.
    (Gaussian Elimination over GF(2) 기반)
    """
    basis = []
    for num in nums:
        cur = num
        for b in basis:
            cur = min(cur, cur ^ b)
        if cur > 0:
            basis.append(cur)
            basis.sort(reverse=True)
    return sum(basis)  # XOR로 만들 수 있는 최대값


# ── 응용 2: SOS DP로 비트 AND 부분집합 문제 풀기 ──
def count_pairs_with_zero_and(n_bits: int, arr: list[int]) -> int:
    """
    arr의 모든 쌍 (i, j)에서 arr[i] & arr[j] == 0인 쌍의 수.
    SOS DP 활용: cnt[mask] = arr에서 mask의 부분집합인 원소 수
    """
    size = 1 << n_bits
    cnt = [0] * size
    for x in arr:
        cnt[x] += 1

    # SOS: cnt[mask] = mask의 부분집합인 원소 개수 합산
    sos = sos_dp(n_bits, cnt)

    # arr[i] & arr[j] == 0 ↔ arr[j]는 arr[i]의 보수 집합의 부분집합
    result = 0
    for x in arr:
        complement = (size - 1) ^ x  # ~x (n비트 내에서)
        result += sos[complement]
        # (x, x) 쌍은 x & x = x. x가 0이면 중복 가산
    return result


# 테스트
n = 3
A = [1, 2, 3, 4, 5, 6, 7, 8]  # A[0]=1, A[1]=2, ..., A[7]=8
F = sos_dp(n, A)

print("원래 A:", A)
print("SOS F:", F)
# F[7] = F[0b111] = A[0]+A[1]+A[2]+A[3]+A[4]+A[5]+A[6]+A[7] = 36
# F[5] = F[0b101] = A[0]+A[1]+A[4]+A[5] = 1+2+5+6 = 14? 
# 실제로는 0b101의 부분집합: 0b000, 0b001, 0b100, 0b101
# = A[0]+A[1]+A[4]+A[5] = 1+2+5+6 = 14

# 역변환으로 원래 A 복원
A_restored = inverse_sos(n, F[:])
print("복원 A:", A_restored)  # 원래 A와 동일해야 함
```

**SOS DP의 시각적 이해** (n=3, 비트 처리 순서):

```
초기: F[mask] = A[mask]

dim=0 처리 후:
  F[0b001] += F[0b000]  → F[1] = A[1]+A[0]
  F[0b011] += F[0b010]  → F[3] = A[3]+A[2]
  F[0b101] += F[0b100]  → F[5] = A[5]+A[4]
  F[0b111] += F[0b110]  → F[7] = A[7]+A[6]

dim=1 처리 후:
  F[0b010] += F[0b000]  → F[2] = A[2]+A[0]
  F[0b011] += F[0b001]  → F[3] = (A[3]+A[2])+(A[1]+A[0]) = {0,1,2,3} 합
  ...

dim=2 처리 후:
  F[0b100] += F[0b000]  ...
  F[0b111] += F[0b011]  → F[7] = 전체 합
```

각 차원에서 해당 비트를 "0으로 자유롭게 할 수 있는" 방향으로 기여를 누적합니다.

---

## 주의사항 및 팁

### 1. 비트마스크 DP 적용 기준

| 조건 | 권장 사항 |
|------|---------|
| n ≤ 20 | 비트마스크 DP 적용 가능 |
| n ≤ 25 | 메모리가 넉넉하면 가능 (2⁵ × n × 4byte) |
| n > 30 | 분할 정복 + 만나는 중간(Meet in the Middle) 기법 필요 |

### 2. 공간 최적화: 비트 수 줄이기

원소 개수가 크더라도 중요한 비트 수만 마스크로 사용:

```cpp
// 원소 30개이지만 "그룹"이 15개 → 15비트 마스크 사용
// 원소를 그룹으로 클러스터링 후 그룹 단위로 비트마스크 DP
```

### 3. SOS DP의 확장: 상위집합 합(Sum Over Supersets)

부분집합 합 대신 **상위집합 합** F(x) = Σ A[i] (x ⊆ i)는 조건을 반전:

```python
def sos_superset(n: int, A: list[int]) -> list[int]:
    F = A[:]
    for bit in range(n):
        for mask in range(1 << n):
            if not (mask & (1 << bit)):  # 이 비트가 꺼져 있으면
                F[mask] += F[mask | (1 << bit)]
    return F
```

### 4. 부분집합 합 합성(Subset Sum Convolution)

h(S) = Σ f(T)·g(S\T) (T ⊆ S, S\T = S에서 T를 뺀 집합) 계산:
SOS DP를 popcount별로 n+1번 실행 → O(n² · 2ⁿ)

### 5. 비트마스크 DP 디버깅 팁

```python
# 마스크를 집합으로 출력
def mask_to_set(mask: int, n: int) -> set:
    return {i for i in range(n) if mask & (1 << i)}

# 사용 예
print(mask_to_set(0b1010, 4))  # {1, 3}
```

---

## 참고 자료
- [Sum over Subsets DP - USACO Guide](https://usaco.guide/adv/dp-sos)
- [SOS DP - Codeforces Blog by sidhant](https://codeforces.com/blog/entry/72488)
- [Bitmask DP Tutorial - Codeforces](https://codeforces.com/blog/entry/105247)
