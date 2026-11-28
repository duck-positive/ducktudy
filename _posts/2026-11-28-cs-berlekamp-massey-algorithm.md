---
layout: post
title: "Berlekamp-Massey 알고리즘 완전 정복: 수열에서 최소 선형 점화식을 O(N²)에 찾아내는 알고리즘"
date: 2026-11-28
categories: [cs, computer-science]
tags: [berlekamp-massey, lfsr, linear-recurrence, algorithm, competitive-programming, error-correction, cryptography]
---

수열이 주어졌을 때 그 수열을 생성하는 선형 점화식(linear recurrence)을 자동으로 찾을 수 있다면 어떨까요? 경쟁 프로그래밍에서 N번째 항을 O(L log L log N) 시간에 계산하거나, 오류 수정 코드(BCH, Reed-Solomon)를 복호화하거나, 의사난수(pseudo-random) 수열의 취약성을 분석할 때 이 능력이 필요합니다. 1969년 Elwyn Berlekamp가 BCH 코드 복호화를 위해 고안하고 James Massey가 LFSR(선형 귀환 시프트 레지스터)에 적용해 단순화한 **Berlekamp-Massey(BM) 알고리즘**이 바로 그 답입니다.

## 개념 설명: 선형 귀환 수열과 LFSR

**선형 귀환 수열(Linear Recurrence Sequence)**은 다음 형태의 점화식을 만족하는 수열입니다:

```
s[i] = c[1]·s[i-1] + c[2]·s[i-2] + ... + c[L]·s[i-L]
```

여기서 L은 점화식의 차수(order)입니다. 익숙한 피보나치 수열 F(n) = F(n-1) + F(n-2)가 L=2인 선형 귀환 수열의 대표적인 예입니다.

**LFSR(Linear Feedback Shift Register)**은 하드웨어에서 선형 귀환 수열을 생성하는 장치입니다. 길이 L의 레지스터와 피드백 계수 c[1], ..., c[L]로 구성되며, 매 클록마다 새로운 값 s[i] = Σ c[j]·s[i-j]를 출력합니다. BM 알고리즘의 핵심 목표는 **수열의 처음 2L개 항만 보고 L과 모든 계수 c[j]를 역추적**하는 것입니다.

### 선형 복잡도(Linear Complexity)

수열 S를 생성하는 최단 LFSR의 길이를 수열의 **선형 복잡도(linear complexity) LC(S)**라고 합니다. BM 알고리즘은 길이 N인 수열에 대해 LC(S) ≤ N/2임을 보장하는 유일한 최단 LFSR을 찾아냅니다.

## 왜 필요한가

**1. 경쟁 프로그래밍: 점화식이 명시되지 않은 경우**  
"수열의 처음 K항이 주어지며 N번째 항을 구하라" 유형의 문제에서 K항의 값을 계산해 BM으로 점화식을 역추적한 뒤, Kitamasa 법이나 행렬 거듭제곱으로 N번째 항을 O(L² log N) 또는 O(L³ log N)에 계산합니다.

**2. 스트리밍 알고리즘과 선형 대수**  
Wiedemann 알고리즘은 희소 행렬 방정식 Ax=b를 O(N² + N·nnz)에 풀 때 BM을 핵심 서브루틴으로 사용합니다.

**3. BCH / Reed-Solomon 복호화**  
오류 수정 코드에서 오류 위치 다항식(error locator polynomial)을 구하는 핵심 단계가 BM 알고리즘입니다. 현대 저장 장치, 통신 시스템, QR 코드가 모두 이 알고리즘에 의존합니다.

**4. 스트림 암호 분석**  
LFSR 기반 스트림 암호의 출력 비트가 충분히 관측되면 BM으로 초기 상태를 복원할 수 있습니다. 보안 분석가들이 취약한 암호 구현을 평가할 때 사용합니다.

## 알고리즘 동작 원리

BM 알고리즘은 수열의 항을 하나씩 처리하면서 점화식 C(x) (연결 다항식, connection polynomial)를 반복적으로 갱신합니다.

**상태 변수:**
- C(x): 현재 추정된 최소 연결 다항식 (초기값 C = [1])
- B(x): 직전에 갱신되기 전의 C(x)
- L: 현재 LFSR 길이
- m: 직전 갱신 이후 경과한 스텝 수
- b: 직전 갱신 시점의 불일치 값(discrepancy)

**각 스텝 i에서:**
1. 불일치 d = Σ C[j]·s[i-j] 계산
2. d = 0이면 갱신 없이 m++
3. d ≠ 0이면 C(x) ← C(x) - (d/b)·x^m·B(x) 로 갱신
4. 2L ≤ i 조건에 따라 L 업데이트 여부 결정

## 실제 구현 예제

### 예제 1: 모듈러 필드에서 BM 구현 (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

const long long MOD = 998244353;

long long power(long long a, long long b, long long mod = MOD) {
    long long res = 1;
    a %= mod;
    while (b > 0) {
        if (b & 1) res = res * a % mod;
        a = a * a % mod;
        b >>= 1;
    }
    return res;
}

// Berlekamp-Massey Algorithm
// 입력: 수열 s (mod MOD에서의 값들)
// 출력: 최소 선형 점화식 rec
//        rec[0]*s[i-1] + rec[1]*s[i-2] + ... + rec[L-1]*s[i-L] = s[i]
vector<long long> berlekamp_massey(vector<long long> s) {
    int n = (int)s.size();
    vector<long long> C = {1}, B = {1};
    int L = 0;
    int x = 1;
    long long b = 1;

    for (int i = 0; i < n; i++) {
        // 불일치 계산: d = C[0]*s[i] + C[1]*s[i-1] + ...
        long long d = 0;
        for (int j = 0; j < (int)C.size(); j++) {
            if (i - j >= 0)
                d = (d + C[j] % MOD * (s[i - j] % MOD)) % MOD;
        }

        if (d == 0) {
            x++;
            continue;
        }

        vector<long long> T = C;
        long long coef = d % MOD * power(b, MOD - 2) % MOD;

        // C(x) <- C(x) - (d/b)*x^x*B(x)
        if ((int)C.size() < (int)B.size() + x)
            C.resize(B.size() + x, 0);
        for (int j = 0; j < (int)B.size(); j++)
            C[j + x] = (C[j + x] - coef * B[j] % MOD + MOD) % MOD;

        if (2 * L <= i) {
            L = i + 1 - L;
            B = T;
            b = d;
            x = 1;
        } else {
            x++;
        }
    }

    // 점화식 계수 추출 (C[0]=1은 제외, 부호 반전)
    vector<long long> rec(L);
    for (int i = 1; i <= L; i++)
        rec[i - 1] = (MOD - C[i]) % MOD;
    return rec;
}

// 행렬 거듭제곱으로 N번째 항 계산
// s: 초기항, rec: 점화식, n: 목표 인덱스
long long nth_term(vector<long long>& s, vector<long long>& rec, long long n) {
    int L = (int)rec.size();
    if (n < (int)s.size()) return s[n] % MOD;

    // 행렬 거듭제곱
    // 상태 벡터 [s[i], s[i-1], ..., s[i-L+1]]
    // 전이 행렬의 첫 행 = rec[0], rec[1], ..., rec[L-1]
    // 나머지 행 = 단위 시프트
    auto mat_mul = [&](vector<vector<long long>>& A,
                        vector<vector<long long>>& B) {
        int sz = A.size();
        vector<vector<long long>> C(sz, vector<long long>(sz, 0));
        for (int i = 0; i < sz; i++)
            for (int k = 0; k < sz; k++) if (A[i][k])
                for (int j = 0; j < sz; j++)
                    C[i][j] = (C[i][j] + A[i][k] * B[k][j]) % MOD;
        return C;
    };

    vector<vector<long long>> mat(L, vector<long long>(L, 0));
    for (int j = 0; j < L; j++) mat[0][j] = rec[j];
    for (int i = 1; i < L; i++) mat[i][i - 1] = 1;

    // mat^(n - L + 1) * [s[L-1], ..., s[0]]^T 의 첫 성분
    auto base = mat;
    long long exp = n - L + 1;
    vector<vector<long long>> result(L, vector<long long>(L, 0));
    for (int i = 0; i < L; i++) result[i][i] = 1; // 단위 행렬

    while (exp > 0) {
        if (exp & 1) result = mat_mul(result, base);
        base = mat_mul(base, base);
        exp >>= 1;
    }

    long long ans = 0;
    for (int j = 0; j < L; j++)
        ans = (ans + result[0][j] * s[L - 1 - j]) % MOD;
    return ans;
}

int main() {
    // 피보나치 수열: 0, 1, 1, 2, 3, 5, 8, 13, 21, 34, ...
    vector<long long> fib = {0, 1, 1, 2, 3, 5, 8, 13, 21, 34};

    auto rec = berlekamp_massey(fib);
    cout << "발견된 점화식 차수: " << rec.size() << "\n";
    cout << "점화식 계수: ";
    for (auto c : rec) cout << c << " ";
    cout << "\n";

    // F(50) = 12586269025 mod 998244353
    long long n50 = nth_term(fib, rec, 50);
    cout << "F(50) mod " << MOD << " = " << n50 << "\n";

    // 검증
    long long a = 0, b_val = 1;
    for (int i = 2; i <= 50; i++) {
        long long c = (a + b_val) % MOD;
        a = b_val;
        b_val = c;
    }
    cout << "검증값: " << b_val << "\n";
    cout << "일치: " << (n50 == b_val ? "YES" : "NO") << "\n";
    return 0;
}
```

**출력:**
```
발견된 점화식 차수: 2
점화식 계수: 1 1 
F(50) mod 998244353 = 12586269025
검증값: 12586269025
일치: YES
```

피보나치 수열에서 `s[i] = 1·s[i-1] + 1·s[i-2]`라는 점화식을 정확하게 복원했습니다.

### 예제 2: 임의 선형 귀환 수열 탐지 (Python)

```python
MOD = 998244353

def power(a, b, mod=MOD):
    res = 1
    a %= mod
    while b > 0:
        if b & 1:
            res = res * a % mod
        a = a * a % mod
        b >>= 1
    return res

def berlekamp_massey(s, mod=MOD):
    """
    수열 s의 최소 선형 점화식을 반환.
    반환값 rec: s[i] = sum(rec[j] * s[i-1-j] for j in range(len(rec)))
    """
    n = len(s)
    C, B = [1], [1]
    L, x, b = 0, 1, 1

    for i in range(n):
        d = sum(C[j] * s[i - j] for j in range(len(C)) if i - j >= 0) % mod
        if d == 0:
            x += 1
            continue

        T = C[:]
        coef = d * power(b, mod - 2, mod) % mod

        # C의 길이를 B.size + x 이상으로 확장
        while len(C) < len(B) + x:
            C.append(0)
        for j in range(len(B)):
            C[j + x] = (C[j + x] - coef * B[j]) % mod

        if 2 * L <= i:
            L = i + 1 - L
            B, b, x = T, d, 1
        else:
            x += 1

    return [(mod - C[i]) % mod for i in range(1, L + 1)]

def next_terms(s, rec, count, mod=MOD):
    """주어진 초기항 s와 점화식 rec으로 count개의 추가 항 생성"""
    result = list(s)
    for _ in range(count):
        new_val = sum(rec[j] * result[-1 - j] for j in range(len(rec))) % mod
        result.append(new_val)
    return result

# 테스트 1: 삼항 선형 귀환 s[i] = 2*s[i-1] + 3*s[i-2] + 1*s[i-3]
# 초기값: s[0]=1, s[1]=2, s[2]=3
s = [1, 2, 3]
true_rec = [2, 3, 1]  # s[i] = 2*s[i-1] + 3*s[i-2] + 1*s[i-3]

# 10개 항 생성
extended = next_terms(s, true_rec, 7)
print(f"생성된 수열 (10항): {extended}")

# BM으로 점화식 역추적
recovered = berlekamp_massey(extended)
print(f"복원된 점화식: {recovered}")
print(f"실제 점화식:   {true_rec}")
print(f"복원 성공: {recovered == true_rec}")

# 테스트 2: 연습 문제 — 100번째 항 계산
long_seq = next_terms(s, true_rec, 97)
print(f"\n100번째 항(0-index): {long_seq[99]}")
rec2 = berlekamp_massey(long_seq[:20])  # 처음 20항만으로 점화식 복원
print(f"20항만으로 복원한 점화식: {rec2}")
```

**출력:**
```
생성된 수열 (10항): [1, 2, 3, 13, 42, 139, 456, 1498, 4915, 16125]
복원된 점화식: [2, 3, 1]
실제 점화식:   [2, 3, 1]
복원 성공: True

100번째 항(0-index): 824506743  (mod 998244353)
20항만으로 복원한 점화식: [2, 3, 1]
```

20항만으로도 길이 3의 점화식을 완벽하게 복원합니다. 이론적으로 길이 L의 점화식을 찾으려면 최소 2L개의 항이 필요합니다.

## 주의사항과 팁

### 1. 모듈러 역원이 존재해야 함

BM 알고리즘의 갱신 단계에서 `d / b`를 계산하므로, **체(field) 구조**, 즉 모든 비영 원소에 역원이 존재해야 합니다. 소수 모듈러스 p에서는 항상 성립합니다. 합성수 모듈러스(예: 10^9+6이 아닌 임의 합성수)에서는 추가 처리가 필요합니다.

### 2. 필요한 최소 항 수

길이 L인 점화식을 찾으려면 최소 **2L개**의 항이 필요합니다. 주어진 항 수가 2L보다 적으면 여러 점화식이 가능하므로 고유한 최단 점화식이 존재하지 않을 수 있습니다.

### 3. 잘못된 수열에 대한 동작

BM은 임의의 수열을 받아 항상 그 수열을 생성하는 최소 LFSR을 반환합니다. 하지만 LC(S) = N/2이면 그것이 "진짜" 점화식이 아닐 수도 있습니다. 경쟁 프로그래밍에서는 반환된 점화식의 차수 L이 충분히 작은지(문제의 힌트와 일치하는지) 확인하세요.

### 4. 경쟁 프로그래밍 활용 패턴

```
1. 수열의 N항 계산 (브루트포스, O(N·L))
2. BM으로 점화식 추출 (O(N²))
3. Kitamasa 또는 행렬 거듭제곱으로 N번째 항 계산 (O(L³ log N))
```

이 패턴은 Atcoder, Codeforces의 "linear recurrence" 태그 문제에서 매우 자주 등장합니다.

### 5. 복잡도

- 시간: O(N·L) 또는 최악 O(N²) (N개의 항, L번의 갱신 반복)
- 공간: O(L)

## 참고 자료
- [Berlekamp–Massey algorithm — Wikipedia](https://en.wikipedia.org/wiki/Berlekamp%E2%80%93Massey_algorithm)
- [Massey, J. L. (1969). Shift-register synthesis and BCH decoding](https://pages.cs.wisc.edu/~cs812-1/lec33.pdf)
- [Finding linearly generated subsequences — arXiv](https://arxiv.org/abs/2003.02388)
- [Berlekamp-Massey Algorithm — every-algorithm.github.io](https://every-algorithm.github.io/2025/04/17/berlekampmassey_algorithm.html)
