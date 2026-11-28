---
layout: post
title: "수론 변환(NTT, Number Theoretic Transform) 완전 정복: 모듈러 산술로 다항식 곱셈을 O(N log N)에 정확하게 해결하기"
date: 2026-11-28
categories: [cs, computer-science]
tags: [ntt, number-theoretic-transform, fft, polynomial-multiplication, modular-arithmetic, algorithm, competitive-programming]
---

다항식 곱셈은 현대 알고리즘의 핵심 연산 중 하나입니다. 두 다항식 A(x)와 B(x)를 곱하는 단순한 방법은 O(N²) 시간을 소모하지만, FFT(고속 푸리에 변환)를 활용하면 O(N log N)에 해결할 수 있습니다. 그런데 경쟁 프로그래밍이나 암호학에서는 정수 계수 다항식을 **정확하게** 곱해야 하는 상황이 자주 등장합니다. 부동소수점 연산의 FFT는 반올림 오차로 인해 큰 계수에서 오답을 낼 수 있습니다. 이 문제를 완벽하게 해결하는 것이 바로 **수론 변환(Number Theoretic Transform, NTT)**입니다.

## 개념 설명: FFT의 정수 버전

FFT는 N차 단위근 ω = e^(2πi/N)을 이용해 다항식을 N개의 점에서 평가하고, 역변환으로 복원합니다. NTT는 이 개념을 그대로 가져오되, 복소수 단위근 대신 **소수 모듈러스 p에서의 정수 단위근**을 사용합니다.

소수 p에 대해 Zp = {0, 1, ..., p-1}에는 원시근(primitive root) g가 존재합니다. 이때 N차 NTT를 정의하려면 다음 조건이 필요합니다:

```
N | (p - 1)
```

이 조건이 충족되면, **NTT의 단위근**은 다음과 같이 정의됩니다:

```
ω_N = g^((p-1)/N) mod p
```

이 ω_N은 복소수 단위근과 동일한 대수적 성질을 가집니다:
- ω_N^N ≡ 1 (mod p)
- ω_N^(N/2) ≡ -1 ≡ p-1 (mod p)
- ω_N^k ≠ 1 (1 ≤ k < N)

따라서 FFT와 완전히 동일한 Cooley-Tukey 버터플라이 연산 구조를 정수 모듈러 산술로 그대로 적용할 수 있습니다.

### 자주 쓰이는 NTT 모듈러스

경쟁 프로그래밍에서 가장 많이 쓰이는 소수는 다음과 같습니다:

| 소수 p | 인수 분해 | 원시근 g | 최대 N |
|--------|-----------|----------|--------|
| 998244353 | 119 × 2^23 + 1 | 3 | 2^23 |
| 1004535809 | 479 × 2^21 + 1 | 3 | 2^21 |
| 469762049 | 7 × 2^26 + 1 | 3 | 2^26 |

998244353 = 119 × 2^23 + 1은 2^23 ≈ 8백만 크기의 NTT를 지원하므로 사실상 대부분의 문제를 커버합니다.

## 왜 필요한가

**1. 경쟁 프로그래밍의 다항식 계산**: 두 큰 수의 곱셈, 대용량 배열의 컨볼루션, 집합의 합성곱 등을 O(N log N)에 정확하게 처리해야 할 때 필수입니다.

**2. 암호학 (동형 암호, 격자 기반 암호)**: CRYSTALS-Kyber, CRYSTALS-Dilithium 같은 차세대 양자 내성 암호는 링 LWE(Ring Learning With Errors) 위에 구축되며, 여기서 다항식 곱셈이 핵심 연산입니다. NTT는 이를 효율적으로 수행합니다.

**3. FFT 대비 정확성**: 부동소수점 FFT는 계수가 10^9 수준이 되면 반올림 오차로 인해 ±1 범위의 오차가 발생해 오답이 나옵니다. NTT는 모듈러 산술이므로 이런 문제가 없습니다.

## 실제 구현 예제

### 예제 1: 기본 NTT 구현 (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

const long long MOD = 998244353;
const long long G = 3;  // primitive root

long long power(long long a, long long b, long long mod) {
    long long res = 1;
    a %= mod;
    while (b > 0) {
        if (b & 1) res = res * a % mod;
        a = a * a % mod;
        b >>= 1;
    }
    return res;
}

// inverse == false: forward NTT, inverse == true: inverse NTT
void ntt(vector<long long>& a, bool inverse) {
    int n = a.size();

    // bit-reversal permutation
    for (int i = 1, j = 0; i < n; i++) {
        int bit = n >> 1;
        for (; j & bit; bit >>= 1) j ^= bit;
        j ^= bit;
        if (i < j) swap(a[i], a[j]);
    }

    // butterfly operations
    for (int len = 2; len <= n; len <<= 1) {
        // unit root for this stage
        long long w = inverse
            ? power(G, MOD - 1 - (MOD - 1) / len, MOD)
            : power(G, (MOD - 1) / len, MOD);

        for (int i = 0; i < n; i += len) {
            long long wn = 1;
            for (int j = 0; j < len / 2; j++) {
                long long u = a[i + j];
                long long v = a[i + j + len / 2] * wn % MOD;
                a[i + j]           = (u + v) % MOD;
                a[i + j + len / 2] = (u - v + MOD) % MOD;
                wn = wn * w % MOD;
            }
        }
    }

    // scale by 1/n when doing inverse NTT
    if (inverse) {
        long long n_inv = power(n, MOD - 2, MOD);
        for (auto& x : a) x = x * n_inv % MOD;
    }
}

// multiply two polynomials mod MOD
vector<long long> poly_multiply(vector<long long> a, vector<long long> b) {
    int result_size = (int)a.size() + (int)b.size() - 1;
    int n = 1;
    while (n < result_size) n <<= 1;

    a.resize(n, 0);
    b.resize(n, 0);

    ntt(a, false);
    ntt(b, false);
    for (int i = 0; i < n; i++)
        a[i] = a[i] * b[i] % MOD;
    ntt(a, true);

    a.resize(result_size);
    return a;
}

int main() {
    // (1 + 2x + 3x²) * (4 + 5x + 6x²)
    // = 4 + 13x + 28x² + 27x³ + 18x⁴
    vector<long long> A = {1, 2, 3};
    vector<long long> B = {4, 5, 6};

    auto C = poly_multiply(A, B);
    // C = [4, 13, 28, 27, 18]
    for (int i = 0; i < (int)C.size(); i++)
        cout << "x^" << i << " 계수: " << C[i] << "\n";
    return 0;
}
```

**출력:**
```
x^0 계수: 4
x^1 계수: 13
x^2 계수: 28
x^3 계수: 27
x^4 계수: 18
```

### 예제 2: 큰 정수 곱셈에 NTT 적용 (Python)

NTT를 사용하면 임의 정밀도 정수 곱셈도 구현할 수 있습니다. 각 자리수를 계수로 보고 다항식 곱셈 후, 올림수(carry) 처리를 합니다.

```python
MOD = 998244353
G = 3

def power(a, b, mod=MOD):
    res = 1
    a %= mod
    while b > 0:
        if b & 1:
            res = res * a % mod
        a = a * a % mod
        b >>= 1
    return res

def ntt(a, inverse=False):
    n = len(a)
    j = 0
    for i in range(1, n):
        bit = n >> 1
        while j & bit:
            j ^= bit
            bit >>= 1
        j ^= bit
        if i < j:
            a[i], a[j] = a[j], a[i]

    length = 2
    while length <= n:
        if inverse:
            w = power(G, MOD - 1 - (MOD - 1) // length)
        else:
            w = power(G, (MOD - 1) // length)
        for i in range(0, n, length):
            wn = 1
            for jj in range(length // 2):
                u = a[i + jj]
                v = a[i + jj + length // 2] * wn % MOD
                a[i + jj] = (u + v) % MOD
                a[i + jj + length // 2] = (u - v + MOD) % MOD
                wn = wn * w % MOD
        length <<= 1

    if inverse:
        n_inv = power(n, MOD - 2)
        for i in range(n):
            a[i] = a[i] * n_inv % MOD

def big_multiply(x: int, y: int) -> int:
    """NTT를 이용한 큰 정수 곱셈"""
    # 각 자리수를 추출 (10진수 각 자리)
    digits_x = [int(d) for d in str(x)][::-1]
    digits_y = [int(d) for d in str(y)][::-1]

    n = 1
    while n < len(digits_x) + len(digits_y):
        n <<= 1

    a = digits_x + [0] * (n - len(digits_x))
    b = digits_y + [0] * (n - len(digits_y))

    ntt(a)
    ntt(b)
    for i in range(n):
        a[i] = a[i] * b[i] % MOD
    ntt(a, inverse=True)

    # carry 처리
    result_digits = a[:len(digits_x) + len(digits_y) - 1]
    carry = 0
    for i in range(len(result_digits)):
        result_digits[i] += carry
        carry = result_digits[i] // 10
        result_digits[i] %= 10
    while carry:
        result_digits.append(carry % 10)
        carry //= 10

    return int(''.join(str(d) for d in reversed(result_digits)))

# 검증
x = 123456789
y = 987654321
result = big_multiply(x, y)
print(f"{x} × {y} = {result}")
print(f"검증: {x * y}")
print(f"일치: {result == x * y}")
```

**출력:**
```
123456789 × 987654321 = 121932631112635269
검증: 121932631112635269
일치: True
```

## 주의사항과 팁

### 1. 모듈러스 선택이 핵심

NTT 소수를 선택할 때는 반드시 `N | (p - 1)` 조건을 확인하세요. 998244353은 2^23까지의 N을 지원하므로 사실상 모든 경쟁 프로그래밍 문제에 충분합니다. 단, **정답 자체가 998244353을 넘는 경우** (예: 계수가 10^18 수준)에는 중국인의 나머지 정리(CRT)를 여러 NTT 소수와 조합해야 합니다.

### 2. 비트 역전 순열(Bit-Reversal Permutation)

Cooley-Tukey 버터플라이 알고리즘은 입력 배열을 비트 역순으로 재배열한 뒤 연산합니다. 이 과정은 O(N)이므로 전체 복잡도에 영향을 주지 않습니다.

### 3. 길이를 2의 제곱으로 맞추기

NTT는 배열 길이가 2의 거듭제곱이어야 합니다. 두 다항식 A(차수 n)와 B(차수 m)를 곱할 때 결과 다항식의 최대 차수는 n+m이므로, `n+m+1`보다 큰 가장 작은 2의 거듭제곱으로 패딩해야 합니다.

### 4. 거듭제곱 단계별 단위근 사전 계산

매 버터플라이 단계마다 `power()` 를 호출하면 불필요한 반복 계산이 발생합니다. 단계별 단위근을 사전에 계산해 테이블로 저장하면 약 2배의 속도 향상을 얻을 수 있습니다.

### 5. NTT vs FFT 선택 가이드

| 상황 | 추천 |
|------|------|
| 정수 계수, 모듈러 답 필요 | NTT |
| 실수/복소수 계수 | FFT |
| 계수 범위가 작고 정확도 중요 | NTT |
| 신호 처리, 이미지 | FFT |

NTT는 경쟁 프로그래밍에서 "poly"라는 태그가 붙은 문제, 즉 다항식 연산이 필요한 고난이도 문제에서 거의 필수입니다. 단순히 외워서 쓰기보다 FFT와의 대응 관계를 이해하면 응용 범위가 훨씬 넓어집니다.

## 참고 자료
- [Number Theoretic Transform — Wikipedia](https://en.wikipedia.org/wiki/Number-theoretic_transform)
- [Fast Fourier Transform & NTT — cp-algorithms.com](https://cp-algorithms.com/algebra/fft.html)
- [NTT for Polynomial Multiplication in Lattice Cryptography — Microsoft Research](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/05/RLWE-1.pdf)
- [A Tutorial on Number Theoretic Transform — arXiv](https://arxiv.org/abs/2306.12519)
