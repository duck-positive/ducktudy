---
layout: post
title: "중국인의 나머지 정리(CRT) 완전 정복: 연립 합동식 풀기부터 경쟁 프로그래밍 활용까지"
date: 2026-08-29
categories: [cs, computer-science]
tags: [algorithms, number-theory, crt, modular-arithmetic, extended-euclidean, competitive-programming]
---

2000년 전 중국의 수학서 『손자산경(孫子算經)』에는 이런 문제가 등장한다. "어떤 수를 3으로 나누면 2가 남고, 5로 나누면 3이 남고, 7로 나누면 2가 남는다. 이 수는 무엇인가?" 답은 23이다. 이 고전 문제를 현대적으로 정형화한 것이 바로 **중국인의 나머지 정리(Chinese Remainder Theorem, CRT)**다.

CRT는 단순한 퍼즐에서 출발했지만 현대 CS에서 핵심 역할을 맡고 있다. RSA 암호화의 복호화 속도를 4배 높이는 CRT 최적화, 다항식 곱셈을 빠르게 하는 수론적 변환(NTT), 대용량 정수 연산을 작은 모듈의 병렬 계산으로 분해하는 기법 — 모두 CRT를 기반으로 한다. 경쟁 프로그래밍에서는 답이 매우 커서 두세 개의 소수 모듈로 나눈 나머지로만 관리해야 할 때 CRT로 원래 답을 복원한다.

---

## 정리의 정확한 진술

**정리:** m₁, m₂, …, mₖ가 쌍마다 서로소(pairwise coprime, gcd(mᵢ, mⱼ) = 1 for i ≠ j)이면, 연립 합동식

```
x ≡ a₁ (mod m₁)
x ≡ a₂ (mod m₂)
    …
x ≡ aₖ (mod mₖ)
```

는 0 ≤ x < M = m₁·m₂·…·mₖ 범위에서 **유일한 해**를 가지며, 해는 M을 주기로 반복된다.

---

## 핵심 도구: 확장 유클리드 알고리즘

CRT 풀이의 열쇠는 **확장 유클리드 알고리즘(Extended Euclidean Algorithm)**이다. gcd(a, b) = 1이면 `ax + by = 1`을 만족하는 정수 x, y가 존재하며, 이 x가 a의 b에 대한 **모듈러 역원**이다.

```python
def extended_gcd(a, b):
    """
    ax + by = gcd(a, b)를 만족하는 (gcd, x, y) 반환
    재귀를 사용하지 않는 반복적 구현
    """
    old_r, r = a, b
    old_s, s = 1, 0
    old_t, t = 0, 1
    
    while r != 0:
        q = old_r // r
        old_r, r = r, old_r - q * r
        old_s, s = s, old_s - q * s
        old_t, t = t, old_t - q * t
    
    return old_r, old_s, old_t  # gcd, x, y

def mod_inverse(a, m):
    """a의 mod m에서의 역원 계산 (gcd(a, m) = 1 이어야 함)"""
    g, x, _ = extended_gcd(a % m, m)
    if g != 1:
        raise ValueError(f"역원이 존재하지 않음: gcd({a}, {m}) = {g}")
    return x % m

# 검증
print(mod_inverse(3, 7))   # 5  (3*5 = 15 ≡ 1 mod 7)
print(mod_inverse(10, 17)) # 12 (10*12 = 120 ≡ 1 mod 17)
```

---

## CRT 직접 구현: 두 합동식 병합

두 합동식을 하나로 합치는 연산을 반복하면 임의 개수의 합동식을 처리할 수 있다. 이 병합 연산은 다음과 같이 작동한다.

`x ≡ r₁ (mod m₁)` 과 `x ≡ r₂ (mod m₂)` 를 합쳐 `x ≡ r (mod lcm(m₁, m₂))` 형태로 만든다.

단, m₁과 m₂가 서로소가 아닌 경우(gcd ≠ 1)에도 `r₂ - r₁`이 gcd의 배수이면 해가 존재한다. 이를 **일반화 CRT**라 부른다.

```python
def gcd(a, b):
    while b:
        a, b = b, a % b
    return a

def lcm(a, b):
    return a // gcd(a, b) * b

def crt_merge(r1, m1, r2, m2):
    """
    x ≡ r1 (mod m1)
    x ≡ r2 (mod m2)
    를 합쳐 x ≡ r (mod m) 반환 (m = lcm(m1, m2))
    해가 없으면 None 반환
    """
    g = gcd(m1, m2)
    
    if (r2 - r1) % g != 0:
        return None  # 해 없음
    
    # m1/g 의 mod m2/g 에서의 역원
    _, inv, _ = extended_gcd(m1 // g, m2 // g)
    
    lcm_val = lcm(m1, m2)
    r = (r1 + m1 * ((r2 - r1) // g * inv % (m2 // g))) % lcm_val
    
    return r, lcm_val

def crt(remainders, moduli):
    """
    x ≡ remainders[i] (mod moduli[i]) 의 연립 합동식 풀기
    모듈들이 쌍마다 서로소가 아닌 경우도 처리
    """
    r, m = remainders[0], moduli[0]
    
    for i in range(1, len(remainders)):
        result = crt_merge(r, m, remainders[i], moduli[i])
        if result is None:
            return None  # 해 없음
        r, m = result
    
    return r, m  # x ≡ r (mod m)

# 예시 1: 『손자산경』의 고전 문제
# x ≡ 2 (mod 3), x ≡ 3 (mod 5), x ≡ 2 (mod 7)
r, m = crt([2, 3, 2], [3, 5, 7])
print(f"x ≡ {r} (mod {m})")  # x ≡ 23 (mod 105)

# 예시 2: 모듈이 서로소가 아닌 경우
# x ≡ 1 (mod 4), x ≡ 3 (mod 6)  → gcd(4,6)=2, (3-1)%2=0 이므로 해 있음
result = crt([1, 3], [4, 6])
if result:
    r, m = result
    print(f"x ≡ {r} (mod {m})")  # x ≡ 9 (mod 12)
    print(f"검증: 9%4={9%4}, 9%6={9%6}")  # 1, 3

# 예시 3: 해가 없는 경우
# x ≡ 1 (mod 4), x ≡ 2 (mod 6)  → gcd(4,6)=2, (2-1)%2=1 ≠ 0
result = crt([1, 2], [4, 6])
print(f"해: {result}")  # None
```

---

## RSA에서의 CRT 최적화

RSA 복호화는 `m = c^d mod n` (n = p * q) 계산이다. d가 수백 비트이면 직접 계산은 느리다. CRT를 사용하면 다음과 같이 절반 크기의 지수 연산 두 번으로 쪼갤 수 있다.

```python
def mod_pow(base, exp, mod):
    """빠른 모듈러 지수승 O(log exp)"""
    result = 1
    base %= mod
    while exp > 0:
        if exp & 1:
            result = result * base % mod
        base = base * base % mod
        exp >>= 1
    return result

def rsa_decrypt_crt(c, d, p, q):
    """
    CRT를 이용한 RSA 복호화
    일반 방법보다 약 4배 빠름 (절반 크기 지수, 절반 크기 모듈 두 번)
    """
    n = p * q
    
    # CRT 사전 계산 (키 생성 시 한 번만)
    dp = d % (p - 1)  # d mod (p-1)  [페르마 소정리: a^(p-1) ≡ 1 mod p]
    dq = d % (q - 1)  # d mod (q-1)
    q_inv = mod_inverse(q, p)  # q의 mod p에서의 역원
    
    # 각 소수 모듈에서 복호화
    m_p = mod_pow(c, dp, p)   # c^dp mod p
    m_q = mod_pow(c, dq, q)   # c^dq mod q
    
    # CRT로 합치기
    h = (q_inv * (m_p - m_q)) % p
    m = m_q + h * q
    
    return m % n

# 간단한 RSA 예시 (실제로는 훨씬 큰 소수 사용)
p, q = 61, 53
n = p * q        # 3233
e = 17           # 공개 지수
# d = 2753 (개인 지수, e*d ≡ 1 mod lcm(p-1, q-1))
d = mod_inverse(e, lcm(p-1, q-1))

plaintext = 65
ciphertext = mod_pow(plaintext, e, n)
print(f"암호화: {plaintext} → {ciphertext}")

# 일반 복호화
decrypted_normal = mod_pow(ciphertext, d, n)

# CRT 복호화
decrypted_crt = rsa_decrypt_crt(ciphertext, d, p, q)

print(f"일반 복호화: {decrypted_normal}")
print(f"CRT 복호화:  {decrypted_crt}")
print(f"일치: {decrypted_normal == decrypted_crt == plaintext}")
```

---

## 경쟁 프로그래밍: 큰 수 나머지 복원

다항식 곱셈, 조합 수 계산 등에서 답이 너무 커서 두 소수 모듈로 나눈 나머지만 계산하고, CRT로 실제 값을 복원하는 기법이다.

```python
MOD1 = 998244353   # FFT-friendly prime
MOD2 = 1000000007  # 흔히 사용되는 소수

def solve_with_crt_doubling(n):
    """
    두 소수 모듈에서 계산하고 CRT로 실제 값 복원
    (답이 MOD1 * MOD2 미만인 경우만 유효)
    """
    # 예: n! 계산 (실제로는 복잡한 DP 계산을 각 모듈에서 수행)
    ans1 = 1
    for i in range(1, n + 1):
        ans1 = ans1 * i % MOD1
    
    ans2 = 1
    for i in range(1, n + 1):
        ans2 = ans2 * i % MOD2
    
    # CRT로 실제 답 복원
    result = crt([ans1, ans2], [MOD1, MOD2])
    if result:
        actual, modulus = result
        return actual
    return None

# 작은 예시 검증
import math
for n in range(1, 8):
    recovered = solve_with_crt_doubling(n)
    print(f"{n}! = {math.factorial(n)}, CRT 복원 = {recovered}, 일치: {math.factorial(n) == recovered}")
```

---

## 수론적 변환(NTT)과의 연계

FFT를 정수 계산에 적용할 때 부동소수점 오차를 피하기 위해 NTT(Number Theoretic Transform)를 사용한다. NTT는 특정 소수 모듈(예: 998244353 = 119 × 2²³ + 1)에서만 동작하므로, 원래 계수가 큰 다항식 곱셈에는 여러 NTT-friendly 소수로 계산하고 CRT로 합치는 **Garner 알고리즘**을 사용한다.

```python
def garner(remainders, moduli, final_mod=None):
    """
    Garner 알고리즘: CRT로 큰 수를 혼합 기수 표현으로 변환
    N개의 모듈 모두 쌍마다 서로소여야 함
    """
    k = len(moduli)
    # mixed_radix[i]: 혼합 기수 표현의 계수
    coef = list(remainders)
    
    for i in range(k):
        for j in range(i):
            # coef[i] = (coef[i] - coef[j]) * (moduli[j])^(-1) mod moduli[i]
            coef[i] = (coef[i] - coef[j]) * mod_inverse(moduli[j], moduli[i]) % moduli[i]
    
    # 실제 값 재구성
    result = 0
    product = 1
    for i in range(k):
        result += coef[i] * product
        product *= moduli[i]
        if final_mod:
            result %= final_mod
            product %= final_mod
    
    return result

# 테스트: x ≡ 2 (mod 3), x ≡ 3 (mod 5), x ≡ 2 (mod 7) → 23
answer = garner([2, 3, 2], [3, 5, 7])
print(f"Garner 결과: {answer}")  # 23
```

---

## 주의사항과 팁

**1. 서로소 조건 검증**  
CRT 적용 전 모든 모듈 쌍이 서로소인지 확인하라. 경쟁 문제에서는 소수 모듈을 주는 경우가 대부분이지만, 그렇지 않으면 일반화 CRT를 사용해야 한다.

**2. 오버플로 주의 (C/C++ 환경)**  
`(r1 + m1 * k) % lcm` 계산에서 m1 * k가 long long을 초과할 수 있다. Python은 자동으로 임의 정밀도를 사용하지만, C++에서는 `__int128` 또는 `(long long)m1 * k % lcm` 처럼 중간 값 관리가 필요하다.

**3. 음수 모듈러 처리**  
Python의 `%` 연산자는 항상 비음수를 반환하지만, C++의 `%` 연산자는 음수 피연산자에서 음수를 반환할 수 있다. `((x % m) + m) % m` 패턴으로 항상 양수로 정규화하라.

**4. 역원이 존재하지 않는 경우**  
`mod_inverse(a, m)` 계산 시 `gcd(a, m) ≠ 1`이면 역원이 존재하지 않는다. 이는 해당 합동식이 해를 가지지 않거나, 알고리즘 설계에 오류가 있음을 의미한다.

**5. Garner 알고리즘의 장점**  
일반 CRT는 결과가 M = m₁·m₂·…·mₖ이라는 거대한 수가 되어 후속 연산에서 오버플로가 발생하기 쉽다. Garner 알고리즘은 최종 모듈을 지정해 결과를 그 모듈 이하로 유지하면서 동시에 정확도를 보장한다.

---

## 참고 자료
- [CP-Algorithms: Chinese Remainder Theorem](https://cp-algorithms.com/algebra/chinese-remainder-theorem.html)
- [Wikipedia: Chinese Remainder Theorem](https://en.wikipedia.org/wiki/Chinese_remainder_theorem)
- [CP-Algorithms: Garner Algorithm](https://cp-algorithms.com/algebra/garner.html)
- [Codeforces: Number Theory Notes](https://codeforces.com/blog/entry/61290)
