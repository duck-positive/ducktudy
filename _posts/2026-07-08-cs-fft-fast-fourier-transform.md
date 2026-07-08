---
layout: post
title: "고속 푸리에 변환(FFT) 완전 정복: O(N²)을 O(N log N)으로 줄이는 알고리즘의 마법"
date: 2026-07-08
categories: [cs, computer-science]
tags: [fft, algorithm, polynomial, dft, cooley-tukey, divide-and-conquer, signal-processing]
---

## 개념 설명

고속 푸리에 변환(Fast Fourier Transform, FFT)은 이산 푸리에 변환(Discrete Fourier Transform, DFT)을 효율적으로 계산하는 알고리즘입니다. 1965년 Cooley와 Tukey가 발표했지만, 사실 Gauss가 1805년에 이미 유사한 방법을 발견한 것으로 알려져 있습니다.

DFT는 N개의 복소수 입력을 받아 N개의 복소수 주파수 성분을 출력합니다. 직관적으로 구현하면 O(N²)이지만, FFT는 이를 **O(N log N)**으로 줄입니다. 이 차이는 N = 10^6일 때 약 5만 배의 속도 차이를 만듭니다.

### 핵심 수학: N제곱근과 단위근

FFT의 핵심은 **단위근(roots of unity)**입니다. x^N = 1의 복소수 해를 N제곱근이라 하며, 이 중 주원시근(principal N-th root)은 다음과 같습니다.

```
ω_N = e^(2πi/N) = cos(2π/N) + i·sin(2π/N)
```

모든 N제곱근은 ω_N^k (k = 0, 1, ..., N-1) 형태로 표현됩니다. DFT는 이 단위근들을 점 집합으로 사용하여 다항식을 평가하는 연산입니다.

### 쿨리-터키(Cooley-Tukey) 알고리즘

가장 널리 쓰이는 FFT 구현은 **분할 정복(Divide and Conquer)** 방식입니다. N개 계수를 가진 다항식 A(x)를 짝수 인덱스와 홀수 인덱스로 분리합니다.

```
A(x) = A_even(x²) + x · A_odd(x²)
```

여기서 A_even은 짝수 번째 계수로, A_odd는 홀수 번째 계수로 구성된 다항식입니다. 이 분리를 재귀적으로 적용하면 각 단계에서 문제 크기가 절반으로 줄어 O(N log N) 시간이 됩니다.

---

## 왜 필요한가

FFT는 단순히 신호처리에만 쓰이는 게 아닙니다. 컴퓨터 과학의 여러 분야에서 핵심적인 역할을 합니다.

**1. 큰 정수 곱셈**
두 N자리 정수의 곱셈을 일반 방법으로 하면 O(N²)이 걸립니다. FFT를 이용한 다항식 곱셈으로 O(N log N)에 처리할 수 있으며, 이는 Python의 내장 big integer 곱셈에도 활용됩니다.

**2. 문자열 패턴 매칭**
와일드카드를 포함한 문자열 매칭을 FFT로 O(N log N)에 해결할 수 있습니다. 두 문자열의 역전 합성곱(convolution)을 구하면 각 위치에서의 일치 여부를 한 번에 계산합니다.

**3. 경쟁 프로그래밍**
N ≤ 10^5 범위의 다항식 곱셈 문제에서 FFT 없이는 시간 초과가 발생합니다. 배낭 문제의 최적화, 부분집합 합(subset sum) 계산 등에 응용됩니다.

**4. 이미지 처리**
2D FFT는 이미지 필터링, 에지 감지, 압축(JPEG 기반 기술)에 사용됩니다.

---

## 실제 구현 예제

### 예제 1: 재귀 FFT 구현 (Python)

```python
import cmath
import math

def fft(a):
    n = len(a)
    if n == 1:
        return a
    
    # 짝수/홀수 분리
    a_even = fft(a[0::2])
    a_odd  = fft(a[1::2])
    
    # 단위근 계산
    result = [0] * n
    for k in range(n // 2):
        # ω = e^(-2πi·k/n)  (역변환은 + 부호)
        w = cmath.exp(-2j * cmath.pi * k / n)
        result[k]           = a_even[k] + w * a_odd[k]
        result[k + n // 2]  = a_even[k] - w * a_odd[k]
    
    return result

def ifft(a):
    n = len(a)
    # 켤레복소수 적용 후 FFT, 마지막에 n으로 나누면 역변환
    a_conj = [x.conjugate() for x in a]
    result = fft(a_conj)
    return [x.conjugate() / n for x in result]

def poly_multiply(a, b):
    """두 다항식의 곱: a와 b는 계수 리스트 (낮은 차수부터)"""
    result_size = len(a) + len(b) - 1
    # FFT는 2의 거듭제곱 크기가 필요
    n = 1
    while n < result_size:
        n <<= 1
    
    # 0 패딩
    fa = fft(a + [0] * (n - len(a)))
    fb = fft(b + [0] * (n - len(b)))
    
    # 점별 곱셈 (O(N))
    fc = [x * y for x, y in zip(fa, fb)]
    
    # 역변환 후 실수부만 반올림
    result = ifft(fc)
    return [round(x.real) for x in result[:result_size]]

# 사용 예: (x² + 2x + 1) × (x + 3) = x³ + 5x² + 7x + 3
a = [1, 2, 1]  # x² + 2x + 1
b = [3, 1]     # 3 + x
print(poly_multiply(a, b))  # [3, 7, 5, 1]
```

이 재귀 구현은 이해하기 쉽지만 함수 호출 오버헤드로 인해 실제로는 느립니다. 실전에서는 반복 버전을 씁니다.

---

### 예제 2: 반복 FFT + 다항식 곱셈 (C++)

경쟁 프로그래밍에서 실제로 사용하는 반복(iterative) FFT입니다. 비트 역순 치환(bit-reversal permutation)으로 재귀 없이 in-place로 처리합니다.

```cpp
#include <bits/stdc++.h>
using namespace std;

using cd = complex<double>;
const double PI = acos(-1);

// 반복 FFT (Cooley-Tukey)
// inv=true 이면 역FFT
void fft(vector<cd>& a, bool inv) {
    int n = a.size();
    
    // 비트 역순 치환: 재귀 순서를 반복으로 시뮬레이션
    for (int i = 1, j = 0; i < n; i++) {
        int bit = n >> 1;
        for (; j & bit; bit >>= 1)
            j ^= bit;
        j ^= bit;
        if (i < j) swap(a[i], a[j]);
    }
    
    // 나비(butterfly) 연산: 길이 2부터 n까지 배증
    for (int len = 2; len <= n; len <<= 1) {
        double ang = 2 * PI / len * (inv ? -1 : 1);
        cd wlen(cos(ang), sin(ang));
        for (int i = 0; i < n; i += len) {
            cd w(1);
            for (int j = 0; j < len / 2; j++) {
                cd u = a[i + j];
                cd v = a[i + j + len/2] * w;
                a[i + j]          = u + v;
                a[i + j + len/2]  = u - v;
                w *= wlen;
            }
        }
    }
    
    if (inv) {
        for (auto& x : a) x /= n;
    }
}

// 두 다항식의 곱
vector<long long> multiply(vector<int>& a, vector<int>& b) {
    vector<cd> fa(a.begin(), a.end());
    vector<cd> fb(b.begin(), b.end());
    
    int result_size = a.size() + b.size() - 1;
    int n = 1;
    while (n < result_size) n <<= 1;
    
    fa.resize(n); fb.resize(n);
    
    fft(fa, false);
    fft(fb, false);
    
    for (int i = 0; i < n; i++) fa[i] *= fb[i];
    
    fft(fa, true);  // 역FFT
    
    vector<long long> result(result_size);
    for (int i = 0; i < result_size; i++)
        result[i] = llround(fa[i].real());
    
    return result;
}

int main() {
    // (3x² + 2x + 1) × (x² + 4)
    // = 3x⁴ + 2x³ + 13x² + 8x + 4
    vector<int> a = {1, 2, 3};  // 1 + 2x + 3x²
    vector<int> b = {4, 0, 1};  // 4 + x²
    
    auto res = multiply(a, b);
    for (int i = 0; i < (int)res.size(); i++)
        cout << "x^" << i << ": " << res[i] << "\n";
    // x^0: 4, x^1: 8, x^2: 13, x^3: 2, x^4: 3
    return 0;
}
```

핵심은 **나비(butterfly) 연산**입니다. 각 단계에서 인접한 두 원소를 합(+)과 차(-)로 합치며, 이것이 log N단계에 걸쳐 진행되어 전체 O(N log N)이 됩니다.

---

### NTT (수론 변환): 정수 오버플로 방지

부동소수점 오차가 문제라면 **Number Theoretic Transform(NTT)**을 씁니다. 소수 mod p = 998244353 (= 119 × 2^23 + 1) 아래에서 원시근을 이용해 FFT와 동일한 구조로 정확한 정수 계산을 합니다.

```python
MOD = 998244353
g = 3  # 원시근

def power(base, exp, mod):
    result = 1
    base %= mod
    while exp > 0:
        if exp & 1:
            result = result * base % mod
        base = base * base % mod
        exp >>= 1
    return result

def ntt(a, inv=False):
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
        # MOD-1을 2^k으로 나눈 원시근 계산
        w = power(g, (MOD - 1) // length, MOD)
        if inv:
            w = power(w, MOD - 2, MOD)
        for i in range(0, n, length):
            wn = 1
            for k in range(length // 2):
                u = a[i + k]
                v = a[i + k + length // 2] * wn % MOD
                a[i + k]                = (u + v) % MOD
                a[i + k + length // 2]  = (u - v + MOD) % MOD
                wn = wn * w % MOD
        length <<= 1
    
    if inv:
        n_inv = power(n, MOD - 2, MOD)
        for i in range(n):
            a[i] = a[i] * n_inv % MOD
```

---

## 주의사항 및 팁

**1. 크기는 반드시 2의 거듭제곱**

쿨리-터키 알고리즘은 입력 크기 N이 2의 거듭제곱이어야 합니다. 결과 다항식 크기 이상의 가장 작은 2의 거듭제곱으로 0 패딩(zero-padding)하세요.

```python
n = 1
while n < len(a) + len(b) - 1:
    n <<= 1
```

**2. 부동소수점 오차**

`double`형 FFT는 계수가 10^4을 넘으면 반올림 오차가 쌓입니다. 경쟁 프로그래밍에서 계수가 크다면 NTT를 쓰거나, `long double`을 사용하거나, 입력을 절반으로 쪼개는 split 기법을 씁니다.

**3. 나비 연산의 방향**

정방향 FFT는 `e^(-2πi/n)`, 역방향은 `e^(+2πi/n)` (또는 그 반대)를 씁니다. 부호를 혼동하면 역변환이 틀립니다. 선택한 부호 관례를 코드 전체에서 일관되게 유지하세요.

**4. 실전 최적화**

- 두 실수 다항식의 곱은 복소수 한 번으로 처리 가능: `fft(a + bi)` 방식으로 FFT를 한 번만 호출
- SSE/AVX SIMD를 활용하면 실제 하드웨어에서 2~8배 추가 가속
- 배열을 미리 할당해두고 재사용하면 캐시 효율이 올라감

**5. 시간/공간 복잡도 정리**

| 연산 | 복잡도 |
|------|--------|
| DFT (브루트포스) | O(N²) |
| FFT (쿨리-터키) | O(N log N) |
| 역FFT | O(N log N) |
| 다항식 곱셈 (FFT 이용) | O(N log N) |
| 공간 | O(N) |

---

## 참고 자료
- [Fast Fourier transform - Algorithms for Competitive Programming](https://cp-algorithms.com/algebra/fft.html)
- [Fast Fourier transform - Wikipedia](https://en.wikipedia.org/wiki/Fast_Fourier_transform)
- [Tutorial on FFT/NTT — Codeforces](https://codeforces.com/blog/entry/43499)
- [Understanding the FFT Algorithm — Jake VanderPlas](https://jakevdp.github.io/blog/2013/08/28/understanding-the-fft/)
