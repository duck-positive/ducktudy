---
layout: post
title: "행렬 거듭제곱(Matrix Exponentiation) 완전 정복: O(N)을 O(log N)으로 줄이는 알고리즘 마법"
date: 2026-07-11
categories: [cs, computer-science]
tags: [matrix-exponentiation, algorithms, dynamic-programming, fibonacci, linear-recurrence, competitive-programming]
---

## 행렬 거듭제곱이란?

행렬 거듭제곱(Matrix Exponentiation)은 **빠른 거듭제곱(Fast Exponentiation, 분할 정복 기반)**을 행렬로 확장한 알고리즘이다. 스칼라 값의 n제곱을 O(log n) 시간에 구하듯, n × n 행렬의 k제곱을 O(n³ log k) 시간에 구한다.

이 기법의 핵심 통찰은 다음과 같다:

```
A^k = A^(k/2) × A^(k/2)            (k가 짝수일 때)
A^k = A^(k/2) × A^(k/2) × A        (k가 홀수일 때)
```

k를 절반으로 줄여가며 재귀적으로 계산하므로, 총 행렬 곱셈 횟수가 O(log k)가 된다.

---

## 왜 행렬 거듭제곱이 필요한가?

### 피보나치 수열의 O(N) 한계

피보나치 F(n)을 단순 반복문으로 구하면 O(N)이다. N이 10¹⁸이라면? 반복문은 수십 년이 걸린다.

```
F(n) = F(n-1) + F(n-2)
```

이 점화식을 행렬로 표현하면:

```
[F(n+1)]   [1 1]^n   [F(1)]
[F(n)  ] = [1 0]   × [F(0)]
```

행렬 `[[1,1],[1,0]]`의 n제곱을 O(log n) 만에 구하면 F(n)을 O(log n)에 계산할 수 있다.

### 선형 점화식의 일반화

행렬 거듭제곱이 적용되는 문제 유형:

1. **선형 점화식**: F(n) = aF(n-1) + bF(n-2) + ... (피보나치의 일반화)
2. **그래프의 경로 수 계산**: k번 이동으로 정점 u에서 v로 가는 경로 수
3. **최단 경로 행렬 거듭제곱**: 정확히 k개 간선을 사용하는 최단 경로
4. **동적 프로그래밍 가속**: 특정 DP 점화식을 행렬로 변환하여 가속

---

## 구현 1: 스칼라 빠른 거듭제곱 (기반 이해)

행렬 거듭제곱을 이해하기 전에 스칼라 버전부터 살펴보자.

```cpp
#include <bits/stdc++.h>
using namespace std;

typedef long long ll;
const ll MOD = 1e9 + 7;

// 기본: O(k) 방법
ll slow_power(ll base, ll k) {
    ll result = 1;
    for (ll i = 0; i < k; i++) {
        result = result * base % MOD;
    }
    return result;
}

// 빠른 거듭제곱: O(log k)
ll fast_power(ll base, ll k, ll mod = MOD) {
    ll result = 1;
    base %= mod;

    while (k > 0) {
        if (k & 1) {  // k가 홀수
            result = result * base % mod;
        }
        base = base * base % mod;  // base를 제곱
        k >>= 1;                   // k를 절반으로
    }
    return result;
}

// 동작 과정 추적 (k=13 예시)
// k=13(1101₂): base^1 * base^4 * base^8 = base^13
void trace_fast_power(ll base, ll k) {
    ll result = 1;
    ll cur = base;
    cout << k << "의 이진수: ";
    for (int bit = 0; bit < 4; bit++) {
        cout << ((k >> bit) & 1);
    }
    cout << " (LSB 먼저)" << endl;

    ll temp_k = k;
    int bit_pos = 0;
    while (temp_k > 0) {
        if (temp_k & 1) {
            cout << "비트 " << bit_pos << ": base^" << (1LL << bit_pos) << " 곱함" << endl;
            result = result * cur % MOD;
        }
        cur = cur * cur % MOD;
        temp_k >>= 1;
        bit_pos++;
    }
    cout << "결과: " << result << endl;
}

int main() {
    cout << "2^10 = " << fast_power(2, 10) << endl;   // 1024
    cout << "2^62 mod (1e9+7) = " << fast_power(2, 62) << endl;
    trace_fast_power(2, 13);
    return 0;
}
```

---

## 구현 2: 행렬 거듭제곱 — 피보나치 O(log N)

```cpp
#include <bits/stdc++.h>
using namespace std;

typedef long long ll;
typedef vector<vector<ll>> Matrix;
const ll MOD = 1e9 + 7;

// 행렬 곱셈: O(n^3) — n은 행렬 크기
Matrix mat_mul(const Matrix& A, const Matrix& B) {
    int n = A.size();
    Matrix C(n, vector<ll>(n, 0));
    for (int i = 0; i < n; i++)
        for (int k = 0; k < n; k++) if (A[i][k])
            for (int j = 0; j < n; j++)
                C[i][j] = (C[i][j] + A[i][k] * B[k][j]) % MOD;
    return C;
}

// 행렬 거듭제곱: O(n^3 * log k)
Matrix mat_pow(Matrix A, ll k) {
    int n = A.size();
    // 단위 행렬 (곱셈의 항등원)
    Matrix result(n, vector<ll>(n, 0));
    for (int i = 0; i < n; i++) result[i][i] = 1;

    while (k > 0) {
        if (k & 1) result = mat_mul(result, A);
        A = mat_mul(A, A);
        k >>= 1;
    }
    return result;
}

// F(n) = [[1,1],[1,0]]^n 의 [0][1] 원소
ll fibonacci(ll n) {
    if (n <= 0) return 0;
    if (n == 1) return 1;

    // 점화식: [F(n+1), F(n)] = [[1,1],[1,0]] * [F(n), F(n-1)]
    Matrix base = {{1, 1}, {1, 0}};
    Matrix result = mat_pow(base, n - 1);

    // result * [F(1), F(0)] = result * [1, 0]
    // F(n) = result[0][0] * 1 + result[0][1] * 0 = result[0][0]
    return result[0][0];
}

// 검증: F(1)=1, F(2)=1, F(3)=2, ..., F(10)=55
int main() {
    cout << "피보나치 수열:" << endl;
    for (int i = 1; i <= 15; i++) {
        cout << "F(" << i << ") = " << fibonacci(i) << endl;
    }

    // 대규모 테스트
    ll n = 1000000000000000000LL; // 10^18
    cout << "\nF(10^18) mod (10^9+7) = " << fibonacci(n) << endl;
    // 이 계산은 O(log(10^18)) ≈ 60번의 행렬 곱셈으로 완료
    return 0;
}
```

### 피보나치 점화식의 행렬 유도

```
F(n)   = 1*F(n-1) + 1*F(n-2)
F(n-1) = 1*F(n-1) + 0*F(n-2)

⟹ [F(n)  ] = [1 1] × [F(n-1)]
   [F(n-1)]   [1 0]   [F(n-2)]

⟹ [F(n+1)] = [1 1]^n × [F(1)] = [1 1]^n × [1]
   [F(n)  ]   [1 0]     [F(0)]   [1 0]     [0]
```

---

## 구현 3: 일반 선형 점화식 자동화

k차 선형 점화식 `a(n) = c1*a(n-1) + c2*a(n-2) + ... + ck*a(n-k)`를 행렬로 변환하는 일반적인 방법이다.

```cpp
#include <bits/stdc++.h>
using namespace std;
typedef long long ll;
typedef vector<vector<ll>> Matrix;
const ll MOD = 1e9 + 7;

Matrix mat_mul(const Matrix& A, const Matrix& B) {
    int n = A.size();
    Matrix C(n, vector<ll>(n, 0));
    for (int i = 0; i < n; i++)
        for (int k = 0; k < n; k++) if (A[i][k])
            for (int j = 0; j < n; j++)
                C[i][j] = (C[i][j] + A[i][k] * B[k][j]) % MOD;
    return C;
}

Matrix mat_pow(Matrix A, ll k) {
    int n = A.size();
    Matrix result(n, vector<ll>(n, 0));
    for (int i = 0; i < n; i++) result[i][i] = 1;
    while (k > 0) {
        if (k & 1) result = mat_mul(result, A);
        A = mat_mul(A, A);
        k >>= 1;
    }
    return result;
}

/**
 * k차 선형 점화식 a(n) = c[0]*a(n-1) + c[1]*a(n-2) + ... + c[k-1]*a(n-k) 에 대해
 * a(n)을 O(k^3 log n) 시간에 계산
 *
 * @param coeffs 점화식 계수 [c1, c2, ..., ck]
 * @param init   초기값 [a(1), a(2), ..., a(k)] (인덱스 1부터)
 * @param n      구하려는 항 번호
 */
ll linear_recurrence(vector<ll> coeffs, vector<ll> init, ll n) {
    int k = coeffs.size();
    if (n <= k) return init[n - 1];

    // 동반 행렬 (Companion Matrix) 구성
    // [c1 c2 c3 ... ck]   [a(n-1)]   [a(n)  ]
    // [1  0  0  ...  0] × [a(n-2)] = [a(n-1)]
    // [0  1  0  ...  0]   [a(n-3)]   [a(n-2)]
    // [0  0  1  ...  0]   [ ...  ]   [ ...  ]
    Matrix companion(k, vector<ll>(k, 0));
    for (int j = 0; j < k; j++)
        companion[0][j] = coeffs[j] % MOD;
    for (int i = 1; i < k; i++)
        companion[i][i - 1] = 1;

    // companion^(n-k) 계산
    Matrix powered = mat_pow(companion, n - k);

    // 초기 상태 벡터: [a(k), a(k-1), ..., a(1)]
    ll result = 0;
    for (int j = 0; j < k; j++) {
        result = (result + powered[0][j] * init[k - 1 - j]) % MOD;
    }
    return result;
}

int main() {
    // 예제 1: 피보나치 (k=2, c=[1,1], init=[1,1])
    // a(n) = a(n-1) + a(n-2), a(1)=1, a(2)=1
    vector<ll> fib_c = {1, 1};
    vector<ll> fib_init = {1, 1};
    cout << "피보나치:" << endl;
    for (int i = 1; i <= 10; i++) {
        cout << "F(" << i << ") = " << linear_recurrence(fib_c, fib_init, i) << endl;
    }

    // 예제 2: Tribonacci (k=3, c=[1,1,1], init=[0,1,1])
    // T(n) = T(n-1) + T(n-2) + T(n-3), T(1)=0, T(2)=1, T(3)=1
    vector<ll> trib_c = {1, 1, 1};
    vector<ll> trib_init = {0, 1, 1};
    cout << "\nTribonacci:" << endl;
    for (int i = 1; i <= 10; i++) {
        cout << "T(" << i << ") = " << linear_recurrence(trib_c, trib_init, i) << endl;
    }
    // T: 0, 1, 1, 2, 4, 7, 13, 24, 44, 81

    // 예제 3: a(n) = 3a(n-1) - 2a(n-2), a(1)=1, a(2)=3 → 2^n - 1
    vector<ll> c3 = {3, -2};
    vector<ll> init3 = {1, 3};
    cout << "\na(n) = 3a(n-1) - 2a(n-2) (결과: 2^n - 1):" << endl;
    for (int i = 1; i <= 8; i++) {
        ll val = linear_recurrence(c3, init3, i);
        // 음수 처리
        val = (val % MOD + MOD) % MOD;
        cout << "a(" << i << ") = " << val << " (예상: " << ((1LL << i) - 1) << ")" << endl;
    }

    return 0;
}
```

---

## 구현 4: 그래프 경로 수 계산

인접 행렬 A에서 A^k의 `[i][j]` 원소는 **정점 i에서 j까지 정확히 k번의 간선을 사용하는 경로 수**다.

```cpp
#include <bits/stdc++.h>
using namespace std;
typedef long long ll;
typedef vector<vector<ll>> Matrix;
const ll MOD = 1e9 + 7;

Matrix mat_mul(const Matrix& A, const Matrix& B) {
    int n = A.size();
    Matrix C(n, vector<ll>(n, 0));
    for (int i = 0; i < n; i++)
        for (int k = 0; k < n; k++) if (A[i][k])
            for (int j = 0; j < n; j++)
                C[i][j] = (C[i][j] + A[i][k] * B[k][j]) % MOD;
    return C;
}

Matrix mat_pow(Matrix A, ll k) {
    int n = A.size();
    Matrix result(n, vector<ll>(n, 0));
    for (int i = 0; i < n; i++) result[i][i] = 1;
    while (k > 0) {
        if (k & 1) result = mat_mul(result, A);
        A = mat_mul(A, A);
        k >>= 1;
    }
    return result;
}

ll count_paths(int n_vertices, vector<pair<int,int>>& edges, int from, int to, ll k) {
    /**
     * 정점 수 n_vertices의 방향 그래프에서
     * from → to 로 정확히 k번 이동하는 경로 수
     */
    Matrix adj(n_vertices, vector<ll>(n_vertices, 0));
    for (auto [u, v] : edges) adj[u][v]++;

    Matrix powered = mat_pow(adj, k);
    return powered[from][to];
}

int main() {
    // 예제: 4개 정점 방향 그래프
    // 0→1, 1→2, 2→3, 3→0, 0→2 (두 경로 존재)
    int n = 4;
    vector<pair<int,int>> edges = {{0,1},{1,2},{2,3},{3,0},{0,2}};

    cout << "정점 0 → 3 까지의 경로 수:" << endl;
    for (int k = 1; k <= 6; k++) {
        ll paths = count_paths(n, edges, 0, 3, k);
        cout << "  정확히 " << k << "번 이동: " << paths << "가지" << endl;
    }

    // 응용: 최소 간선 개수로 갈 수 있는지 BFS로 확인 후
    // 정확히 K번 이동 경로 수를 행렬 거듭제곱으로 계산
    ll K = 1000000000LL;
    cout << "\n정점 0 → 3 까지 정확히 10^9번 이동하는 경로 수 mod (10^9+7):" << endl;
    cout << count_paths(n, edges, 0, 3, K) << endl;

    return 0;
}
```

---

## 행렬 거듭제곱으로 DP 가속하기

타일링 문제처럼 열별로 독립적인 DP는 행렬 거듭제곱으로 O(m³ log n) 시간에 계산할 수 있다.

```python
MOD = 10**9 + 7

def mat_mul(A, B):
    n = len(A)
    C = [[0] * n for _ in range(n)]
    for i in range(n):
        for k in range(n):
            if A[i][k]:
                for j in range(n):
                    C[i][j] = (C[i][j] + A[i][k] * B[k][j]) % MOD
    return C

def mat_pow(A, k):
    n = len(A)
    result = [[1 if i == j else 0 for j in range(n)] for i in range(n)]
    while k > 0:
        if k & 1:
            result = mat_mul(result, A)
        A = mat_mul(A, A)
        k >>= 1
    return result

def tile_ways(n):
    """
    2×n 격자를 1×2 또는 2×1 타일로 채우는 방법 수
    점화식: dp[n] = dp[n-1] + dp[n-2]  (피보나치!)
    이를 행렬 거듭제곱으로 O(log n) 계산
    """
    if n == 0: return 1
    if n == 1: return 1

    base = [[1, 1], [1, 0]]
    result = mat_pow(base, n)
    return result[0][0]  # F(n+1) = dp[n]

# 검증
expected = [1, 1, 2, 3, 5, 8, 13, 21, 34, 55]
for i in range(10):
    val = tile_ways(i)
    print(f"tile_ways({i}) = {val}, 예상: {expected[i]}, {'✅' if val == expected[i] else '❌'}")

# 대규모 계산
print(f"\ntile_ways(10^18) mod (10^9+7) = {tile_ways(10**18)}")
```

---

## 시간 복잡도 분석

| 방법 | 피보나치 F(n) | k차 점화식 a(n) | 경로 수 (V정점, k간선) |
|------|-------------|--------------|---------------------|
| 단순 반복 | O(N) | O(kN) | O(V²k) |
| 행렬 거듭제곱 | O(log N) | O(k³ log N) | O(V³ log k) |

N = 10¹⁸일 때 단순 반복은 **수십억 년**, 행렬 거듭제곱은 **0.001초** 이내에 완료된다.

---

## 주의사항과 팁

### 1. 모듈러 연산 일관성

행렬의 각 원소 크기가 MOD에 가까워지면 곱셈 시 오버플로가 발생한다. C++에서는 `long long`(최대 약 9.2×10¹⁸)으로 `(10⁹)²`까지는 안전하다.

```cpp
// 안전: long long으로 곱하기 전 MOD 적용
C[i][j] = (C[i][j] + (A[i][k] % MOD) * (B[k][j] % MOD)) % MOD;

// 위험: __int128이 없으면 오버플로 가능
// C[i][j] += A[i][k] * B[k][j];
```

### 2. 음수 계수 처리

점화식에 음수 계수가 있으면 결과가 음수가 될 수 있다. 모듈러 연산 전에 MOD를 더해야 한다.

```cpp
ll val = (result[0][0] % MOD + MOD) % MOD;
```

### 3. 행렬 크기 최소화

행렬 크기 n이 클수록 O(n³)의 행렬 곱셈 비용이 급격히 늘어난다. 점화식의 실제 차수(order)에 맞는 최소 크기 행렬을 사용해야 한다. 예를 들어 k=50차 점화식은 50×50 행렬로 충분하다.

### 4. Cayley-Hamilton 정리를 이용한 최적화

k차 특성 다항식을 구하면 k×k보다 더 작은 행렬로 풀 수 있는 경우도 있다. 하지만 대부분의 경우 k×k 동반 행렬로 충분하다.

---

## 마치며

행렬 거듭제곱은 **선형 점화식을 다루는 문제에서 O(N)을 O(log N)으로 줄이는 강력한 도구**다. 단순히 피보나치를 빠르게 계산하는 것을 넘어, 선형 점화식을 갖는 모든 수열, 그래프의 경로 수, 특정 형태의 DP 전이를 가속하는 데 적용할 수 있다.

핵심 아이디어는 세 가지다: (1) 점화식을 행렬 형태로 변환, (2) 행렬 거듭제곱을 분할 정복으로 O(log k) 수행, (3) 모듈러 연산을 일관되게 적용. 이 세 가지를 마스터하면 10¹⁸ 크기의 N도 순식간에 처리할 수 있다.

## 참고 자료
- [Exponentiation by squaring - Wikipedia](https://en.wikipedia.org/wiki/Exponentiation_by_squaring)
- [Matrix Exponentiation - GeeksforGeeks](https://www.geeksforgeeks.org/dsa/matrix-exponentiation/)
- [Binary Exponentiation - cp-algorithms.com](https://cp-algorithms.com/algebra/binary-exp.html)
- [Linear Recurrence and Berlekamp-Massey - cp-algorithms.com](https://cp-algorithms.com/algebra/linear-recurrence.html)
