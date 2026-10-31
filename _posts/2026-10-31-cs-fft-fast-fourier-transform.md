---
layout: post
title: "FFT(고속 푸리에 변환) 완전 정복: Cooley-Tukey 알고리즘과 다항식 곱셈 최적화"
date: 2026-10-31
categories: [cs, computer-science]
tags: [fft, 알고리즘, 수학, 신호처리, 다항식, divide-and-conquer, 분할정복]
---

## 개요

FFT(Fast Fourier Transform, 고속 푸리에 변환)는 이산 푸리에 변환(DFT)을 O(n²)에서 O(n log n)으로 가속하는 알고리즘이다. 단순한 수학적 변환 공식처럼 보이지만, 실제로는 문자열 검색, 정수 곱셈, 이미지 압축, 오디오 처리, 경쟁 프로그래밍의 다항식 곱셈 등 컴퓨터 과학 전반에 걸쳐 활용된다. IEEE Scientific Computing의 20세기 10대 알고리즘 중 하나로 선정된 이 알고리즘의 내부 구조를 완전히 이해해 보자.

---

## 왜 FFT가 필요한가?

### DFT란 무엇인가?

크기 n인 수열 x[0], x[1], ..., x[n-1]의 이산 푸리에 변환은 다음과 같이 정의된다.

$$X[k] = \sum_{j=0}^{n-1} x[j] \cdot \omega_n^{jk}, \quad k = 0, 1, \ldots, n-1$$

여기서 $\omega_n = e^{-2\pi i / n}$은 **n차 원시 단위근(primitive n-th root of unity)**이다.

이 공식을 직접 계산하면 k와 j 두 루프가 중첩되어 O(n²)이 된다. n = 10⁶이면 연산 수는 10¹²에 달해 실용성이 없다.

### 다항식 곱셈에서 FFT의 역할

차수 n인 두 다항식 A(x), B(x)를 곱하면 결과 다항식 C(x)는 차수 2n이 된다. 단순 계수 곱셈은 O(n²)이다.

FFT를 이용하면:
1. A(x), B(x)를 2n+1개의 점에서 평가(evaluation) — O(n log n)
2. 같은 점에서 A·B 점별 곱셈 — O(n)
3. 역 FFT(IFFT)로 계수 복원(interpolation) — O(n log n)

합계 O(n log n). 이는 RSA 같은 큰 수 곱셈, ZIP 압축의 핵심인 LZ77+Huffman 압축, 음성 인식의 컨볼루션 연산에서 결정적인 차이를 만든다.

---

## Cooley-Tukey 알고리즘: 분할 정복의 정수

1965년 James Cooley와 John Tukey가 발표(실제로는 Gauss가 1805년에 이미 발견)한 Radix-2 DIT(Decimation In Time) 알고리즘은 n = 2^k일 때 DFT를 재귀적으로 절반씩 분할한다.

### 핵심 수식 유도

n이 짝수일 때 DFT를 짝수 인덱스와 홀수 인덱스로 분리한다.

```
X[k] = Σ x[j] · ω_n^{jk}
     = Σ x[2m] · ω_n^{2mk}   +   Σ x[2m+1] · ω_n^{(2m+1)k}
     = Σ x[2m] · ω_{n/2}^{mk}  +  ω_n^k · Σ x[2m+1] · ω_{n/2}^{mk}
     = E[k]  +  ω_n^k · O[k]
```

E[k]는 짝수 인덱스의 n/2 크기 DFT, O[k]는 홀수 인덱스의 n/2 크기 DFT다.

k와 k + n/2에 대해:
- X[k]     = E[k] + ω_n^k · O[k]
- X[k+n/2] = E[k] - ω_n^k · O[k]   ← ω_n^{k+n/2} = -ω_n^k

이 **버터플라이 연산(butterfly operation)**이 FFT의 핵심이다. 매 단계마다 결과를 재사용하여 절반의 연산만으로 두 배 크기의 DFT를 계산한다.

---

## 구현 예제 1: 재귀 FFT (Python)

```python
import cmath
import math

def fft(a):
    n = len(a)
    if n == 1:
        return [a[0]]
    
    # 짝수/홀수 분리
    even = fft(a[0::2])
    odd  = fft(a[1::2])
    
    result = [0] * n
    half = n // 2
    
    for k in range(half):
        # n차 원시 단위근의 k제곱
        w = cmath.exp(-2j * cmath.pi * k / n)
        
        # 버터플라이 연산
        result[k]        = even[k] + w * odd[k]
        result[k + half] = even[k] - w * odd[k]
    
    return result

def ifft(a):
    """역 FFT: 켤레 복소수 취한 뒤 FFT, 결과를 n으로 나눔"""
    n = len(a)
    conjugated = [x.conjugate() for x in a]
    result = fft(conjugated)
    return [x.conjugate() / n for x in result]

def poly_multiply(A, B):
    """두 다항식의 계수 배열을 FFT로 곱셈"""
    result_len = len(A) + len(B) - 1
    # 2의 거듭제곱으로 패딩
    n = 1
    while n < result_len:
        n <<= 1
    
    fa = A + [0] * (n - len(A))
    fb = B + [0] * (n - len(B))
    
    fa_fft = fft(fa)
    fb_fft = fft(fb)
    
    # 점별 곱셈
    fc_fft = [a * b for a, b in zip(fa_fft, fb_fft)]
    
    # 역 FFT로 계수 복원
    fc = ifft(fc_fft)
    return [round(x.real) for x in fc[:result_len]]

# 예시: (1 + 2x + 3x²) × (4 + 5x) = 4 + 13x + 22x² + 15x³
A = [1, 2, 3]  # 1 + 2x + 3x²
B = [4, 5]     # 4 + 5x
result = poly_multiply(A, B)
print(result)  # [4, 13, 22, 15]
```

---

## 구현 예제 2: 반복적 FFT와 비트 역전 (Python - O(n log n) 최적화)

재귀 FFT는 함수 호출 오버헤드가 크다. **Bit-reversal permutation**을 사용한 반복적(iterative) FFT는 상수 계수가 훨씬 작다.

```python
def bit_reverse(n, log2_n):
    """n비트 정수를 log2_n 비트 기준으로 비트 역전"""
    result = 0
    for _ in range(log2_n):
        result = (result << 1) | (n & 1)
        n >>= 1
    return result

def fft_iterative(a, invert=False):
    n = len(a)
    log2_n = int(math.log2(n))
    
    # 비트 역전 순열로 입력 배열 재배열
    a = list(a)
    for i in range(n):
        j = bit_reverse(i, log2_n)
        if i < j:
            a[i], a[j] = a[j], a[i]
    
    # 버터플라이 연산: 크기 2, 4, 8, ..., n 순으로
    length = 2
    while length <= n:
        # 현재 레벨의 단위근 (역변환이면 켤레 방향)
        angle = 2 * math.pi / length * (-1 if invert else 1)
        wlen = complex(math.cos(angle), math.sin(angle))
        
        for i in range(0, n, length):
            w = complex(1, 0)
            half = length // 2
            for j in range(half):
                u = a[i + j]
                v = a[i + j + half] * w
                a[i + j]        = u + v
                a[i + j + half] = u - v
                w *= wlen
        
        length <<= 1
    
    if invert:
        a = [x / n for x in a]
    
    return a

def poly_multiply_fast(A, B):
    result_len = len(A) + len(B) - 1
    n = 1
    while n < result_len:
        n <<= 1
    
    fa = A + [0] * (n - len(A))
    fb = B + [0] * (n - len(B))
    
    fa_f = fft_iterative(fa)
    fb_f = fft_iterative(fb)
    fc_f = [a * b for a, b in zip(fa_f, fb_f)]
    fc   = fft_iterative(fc_f, invert=True)
    
    return [round(x.real) for x in fc[:result_len]]

# 정확도 검증
import random
A = [random.randint(0, 9) for _ in range(500)]
B = [random.randint(0, 9) for _ in range(500)]
expected = [
    sum(A[i] * B[j] for i in range(len(A)) for j in range(len(B)) if i + j == k)
    for k in range(len(A) + len(B) - 1)
]
result = poly_multiply_fast(A, B)
assert result == expected, "FFT 곱셈 결과 불일치"
print("검증 통과! 계수 수:", len(result))  # 999
```

**비트 역전 순열의 의미:** 재귀 분할에서 짝수/홀수를 반복하면, 최종 배열 순서는 원본 인덱스를 이진수로 쓴 뒤 비트를 뒤집은 순서와 같다. 예를 들어 n=8이면 [0,4,2,6,1,5,3,7] — 이를 미리 계산해두면 재귀 없이 한 번의 순열로 시작할 수 있다.

---

## 수치 정밀도와 NTT

부동소수점 FFT는 계수가 크면 반올림 오차가 누적된다. 계수가 정수이고 결과를 특정 소수 p로 modulo 할 때는 **NTT(Number Theoretic Transform)**를 사용한다.

NTT는 복소수 단위근 대신 모듈러 원시근 g를 사용한다. 흔히 쓰이는 소수는 p = 998,244,353 = 119 × 2²³ + 1로, 이 소수의 원시근 g = 3이 2²³ 크기의 NTT를 지원한다.

```python
MOD = 998_244_353
g   = 3  # 원시근

def power(base, exp, mod):
    result = 1
    base %= mod
    while exp > 0:
        if exp & 1:
            result = result * base % mod
        base = base * base % mod
        exp >>= 1
    return result

def ntt(a, invert=False):
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
        w = power(g, (MOD - 1) // length, MOD)
        if invert:
            w = power(w, MOD - 2, MOD)  # 모듈러 역원
        for i in range(0, n, length):
            wn = 1
            for k in range(length // 2):
                u = a[i + k]
                v = a[i + k + length // 2] * wn % MOD
                a[i + k]               = (u + v) % MOD
                a[i + k + length // 2] = (u - v + MOD) % MOD
                wn = wn * w % MOD
        length <<= 1
    
    if invert:
        inv_n = power(n, MOD - 2, MOD)
        for i in range(n):
            a[i] = a[i] * inv_n % MOD

def poly_multiply_ntt(A, B):
    result_len = len(A) + len(B) - 1
    n = 1
    while n < result_len:
        n <<= 1
    fa = A + [0] * (n - len(A))
    fb = B + [0] * (n - len(B))
    ntt(fa); ntt(fb)
    for i in range(n):
        fa[i] = fa[i] * fb[i] % MOD
    ntt(fa, invert=True)
    return fa[:result_len]

# 예: 100! 의 자릿수 합 계산에도 사용 가능
A = [3, 1, 4]   # 4x² + x + 3
B = [1, 5, 9]   # 9x² + 5x + 1
print(poly_multiply_ntt(A, B))  # [3, 16, 48, 29, 36] 모두 MOD 적용
```

---

## 응용 분야

| 분야 | 사용 사례 |
|------|-----------|
| **신호 처리** | 오디오 스펙트럼 분석, 노이즈 필터링 |
| **이미지 처리** | JPEG 압축의 DCT(코사인 변환) |
| **정수 곱셈** | Python 큰 수 곱셈, GMP 라이브러리 |
| **경쟁 프로그래밍** | 배열 합성곱, 문자열 매칭 |
| **통신** | OFDM(4G/5G 무선 변조 방식) |

---

## 주의사항과 팁

1. **n이 2의 거듭제곱이어야 한다**: Cooley-Tukey Radix-2는 이 조건이 필수다. 그렇지 않으면 0으로 패딩하거나 Mixed-Radix FFT 혹은 Bluestein's 알고리즘을 사용한다.

2. **부동소수점 오차**: 계수가 10⁶ 이상이거나 배열 크기가 10⁶ 이상이면 NTT를 쓰거나 `long double`을 사용한다.

3. **캐시 친화성**: 반복적 FFT는 메모리 접근 패턴이 시퀀셜에 가까워 재귀보다 실제 수행 시간이 2~5배 빠르다.

4. **실수 FFT 최적화**: 입력이 실수일 경우 두 배열을 허수부와 실수부에 묶어 한 번의 복소수 FFT로 처리하면 시간을 절반으로 줄인다.

5. **n의 크기**: 경쟁 프로그래밍에서 두 배열 곱셈 시 `n ≥ 2*(max_len)` 이 되도록 패딩해야 circular convolution을 방지한다.

---

## 참고 자료
- [Fast Fourier transform — Wikipedia](https://en.wikipedia.org/wiki/Fast_Fourier_transform)
- [Cooley–Tukey FFT algorithm — Wikipedia](https://en.wikipedia.org/wiki/Cooley%E2%80%93Tukey_FFT_algorithm)
- [FFT — cp-algorithms.com](https://cp-algorithms.com/algebra/fft.html)
- [FFT — nayuki.io](https://www.nayuki.io/page/fast-fourier-transform-in-x86-assembly)
