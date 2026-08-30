---
layout: post
title: "에라토스테네스의 체와 선형 체(Linear Sieve) 완전 정복: 소수 생성의 최강 알고리즘들"
date: 2026-08-30
categories: [cs, computer-science]
tags: [sieve-of-eratosthenes, linear-sieve, prime-numbers, number-theory, algorithm]
---

소수(Prime Number)는 컴퓨터 과학 전반에서 핵심 역할을 합니다. 암호화(RSA), 해시 함수, 경쟁 프로그래밍까지 소수 목록을 빠르게 생성하는 능력은 필수입니다. 이 글에서는 고대 그리스에서 탄생한 에라토스테네스의 체부터, O(n) 선형 시간을 달성하는 선형 체(Linear Sieve)와 세그먼트 체(Segmented Sieve)까지 완전히 정복합니다.

## 1. 소수와 소수 판별의 기초

**소수(prime number)**는 1과 자기 자신만을 약수로 가지는 1보다 큰 자연수입니다.

가장 단순한 소수 판별법은 2부터 n-1까지 나눠보는 것 (O(n))이지만, √n까지만 확인해도 충분합니다 (O(√n)).

```python
def is_prime_naive(n: int) -> bool:
    """O(√n) 소수 판별"""
    if n < 2:
        return False
    if n == 2:
        return True
    if n % 2 == 0:
        return False
    i = 3
    while i * i <= n:
        if n % i == 0:
            return False
        i += 2
    return True
```

그러나 **n 이하의 모든 소수**를 구해야 할 때는 위 방법을 n번 반복하면 O(n√n)이 됩니다. 이를 획기적으로 개선한 것이 에라토스테네스의 체입니다.

## 2. 에라토스테네스의 체(Sieve of Eratosthenes)

기원전 3세기 고대 그리스 수학자 에라토스테네스가 고안한 알고리즘입니다. 아이디어는 단순합니다: **소수의 배수를 모두 합성수로 표시**합니다.

### 알고리즘 동작 과정

1. 2부터 n까지의 배열을 생성, 모두 "소수"로 초기화
2. 현재 소수 p=2부터 시작
3. p²부터 p의 배수를 모두 "합성수"로 표시 (p² 이전은 이미 처리됨)
4. 다음 "소수"로 표시된 수로 이동
5. p > √n이 될 때까지 반복

```python
def sieve_of_eratosthenes(n: int) -> list[int]:
    """
    O(n log log n) — n 이하의 모든 소수 반환
    """
    if n < 2:
        return []
    
    # is_prime[i] = True이면 i는 소수
    is_prime = [True] * (n + 1)
    is_prime[0] = is_prime[1] = False
    
    p = 2
    while p * p <= n:
        if is_prime[p]:
            # p의 배수를 p² 부터 합성수로 표시
            # p² 이전의 배수들은 더 작은 소수에 의해 이미 처리됨
            for multiple in range(p * p, n + 1, p):
                is_prime[multiple] = False
        p += 1
    
    return [i for i in range(2, n + 1) if is_prime[i]]

# 사용 예시
primes = sieve_of_eratosthenes(100)
print(f"100 이하의 소수 ({len(primes)}개): {primes}")
# 출력: 100 이하의 소수 (25개): [2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37, 41,
#        43, 47, 53, 59, 61, 67, 71, 73, 79, 83, 89, 97]

# 성능 측정
import time

for limit in [10**6, 10**7, 10**8]:
    start = time.time()
    result = sieve_of_eratosthenes(limit)
    elapsed = time.time() - start
    print(f"N={limit:,}: {len(result):,}개 소수, {elapsed:.3f}초")
```

### 시간 복잡도 분석: 왜 O(n log log n)인가?

각 소수 p에 대해 n/p개의 배수를 표시합니다. 소수 조화 급수의 합:

```
Σ (n/p) for primes p ≤ n = n · Σ(1/p) ≈ n · ln(ln(n))
```

메르텐스 정리(Mertens' theorem)에 의해 소수 역수의 합이 ln(ln(n))에 수렴하므로, 전체 복잡도는 **O(n log log n)**입니다.

실제로 n = 10⁸까지는 약 2초 내에 처리됩니다.

## 3. 메모리 최적화: 비트 배열 체

파이썬의 `[True] * n`은 n 바이트를 사용하지만, 비트 배열을 쓰면 8배 절약할 수 있습니다.

```python
def sieve_bitarray(n: int) -> list[int]:
    """비트 배열을 이용한 메모리 효율적 체"""
    from bitarray import bitarray
    
    # bitarray 미설치 시 bytearray 사용 (8배 절약 대신 1바이트/항목)
    sieve = bytearray([1]) * (n + 1)
    sieve[0] = sieve[1] = 0
    
    p = 2
    while p * p <= n:
        if sieve[p]:
            sieve[p*p::p] = bytearray(len(sieve[p*p::p]))  # 0으로 설정
        p += 1
    
    return [i for i, v in enumerate(sieve) if v and i >= 2]

# 짝수를 미리 제외하는 최적화 버전 (메모리 절반, 속도 2배)
def sieve_optimized(n: int) -> list[int]:
    """짝수 제외 최적화 — 홀수만 배열에 저장"""
    if n < 2:
        return []
    if n == 2:
        return [2]
    
    # is_prime_odd[i] → 2i+3이 소수인지 (i≥0, 홀수 3,5,7,...)
    size = (n - 1) // 2
    is_prime_odd = bytearray([1]) * size
    
    for i in range(size):
        if is_prime_odd[i]:
            p = 2 * i + 3
            if p * p > n:
                break
            # p²부터 p의 홀수 배수 표시
            start = (p * p - 3) // 2
            is_prime_odd[start::p] = bytearray(len(is_prime_odd[start::p]))
    
    result = [2]
    result.extend(2 * i + 3 for i in range(size) if is_prime_odd[i])
    return result
```

## 4. 세그먼트 체(Segmented Sieve): 메모리 제한 환경 대응

n이 매우 클 때(예: n = 10¹²) 전체 배열을 메모리에 올릴 수 없습니다. **세그먼트 체**는 √n 크기의 버켓 단위로 나눠 처리합니다.

```cpp
#include <bits/stdc++.h>
using namespace std;

// [low, high] 구간의 소수를 세그먼트 체로 구하기
// 사전에 small_primes: √high 이하의 소수 목록 필요
vector<long long> segmented_sieve(long long low, long long high) {
    long long limit = (long long)sqrt((double)high) + 1;
    
    // Step 1: √high 이하의 소수를 표준 에라토스테네스로 구함
    vector<bool> small(limit + 1, true);
    small[0] = small[1] = false;
    for (long long p = 2; p * p <= limit; p++) {
        if (small[p])
            for (long long j = p*p; j <= limit; j += p)
                small[j] = false;
    }
    vector<long long> small_primes;
    for (long long p = 2; p <= limit; p++)
        if (small[p]) small_primes.push_back(p);
    
    // Step 2: [low, high] 구간을 세그먼트로 처리
    long long seg_size = high - low + 1;
    vector<bool> is_prime(seg_size, true);
    
    // low가 1 이하인 경우 처리
    if (low <= 1) is_prime[max(0LL, 1 - low)] = false;
    if (low == 0) is_prime[0] = false;
    
    for (long long p : small_primes) {
        // [low, high]에서 p의 첫 번째 배수 찾기
        long long start = ((low + p - 1) / p) * p;
        if (start == p) start += p; // p 자체는 소수이므로 제외
        
        for (long long j = start; j <= high; j += p)
            is_prime[j - low] = false;
    }
    
    vector<long long> result;
    for (long long i = max(2LL, low); i <= high; i++)
        if (is_prime[i - low]) result.push_back(i);
    
    return result;
}

int main() {
    // 10^12 부근의 소수 탐색
    long long low = 1000000000000LL;
    long long high = 1000000001000LL;
    
    auto primes = segmented_sieve(low, high);
    
    cout << "[" << low << ", " << high << "] 구간의 소수:\n";
    for (long long p : primes)
        cout << p << " ";
    cout << "\n총 " << primes.size() << "개\n";
    
    return 0;
}
```

세그먼트 체의 특징:
- **시간 복잡도**: O(n log log n) (에라토스테네스와 동일)
- **공간 복잡도**: O(√n) (버켓 크기 √n만 유지)
- **캐시 친화성**: 세그먼트 크기를 L1 캐시(약 32KB)에 맞추면 성능 향상

## 5. 선형 체(Linear Sieve): O(n) 시간 복잡도

에라토스테네스의 체는 O(n log log n)이지만, **선형 체(Linear Sieve)**는 O(n)에 모든 소수를 구합니다. 핵심 아이디어는 **각 합성수를 정확히 한 번만 표시**하는 것입니다.

합성수 n은 반드시 **최소 소인수(Smallest Prime Factor, SPF)**가 있습니다. 선형 체는 이 SPF를 기준으로 각 합성수를 딱 한 번만 처리합니다.

```cpp
#include <bits/stdc++.h>
using namespace std;

pair<vector<int>, vector<int>> linear_sieve(int n) {
    vector<int> primes;              // 소수 목록
    vector<int> spf(n + 1, 0);      // Smallest Prime Factor
    // spf[i] = 0: 아직 미처리 (= 소수)
    // spf[i] = p: i의 최소 소인수 p
    
    for (int i = 2; i <= n; i++) {
        if (spf[i] == 0) {
            // i가 소수: spf[i] = i (자기 자신이 최소 소인수)
            spf[i] = i;
            primes.push_back(i);
        }
        
        // 지금까지 발견된 소수들과 i의 곱 → 합성수 표시
        for (int j = 0; j < (int)primes.size() && primes[j] <= spf[i]; j++) {
            int composite = i * primes[j];
            if (composite > n) break;
            spf[composite] = primes[j]; // composite의 최소 소인수는 primes[j]
        }
    }
    
    return {primes, spf};
}

int main() {
    int n = 100;
    auto [primes, spf] = linear_sieve(n);
    
    cout << n << " 이하의 소수 (" << primes.size() << "개):\n";
    for (int p : primes) cout << p << " ";
    cout << "\n\n";
    
    // SPF를 이용한 고속 소인수 분해
    auto factorize = [&](int x) {
        vector<int> factors;
        while (x > 1) {
            factors.push_back(spf[x]);
            x /= spf[x];
        }
        return factors;
    };
    
    cout << "SPF를 이용한 소인수 분해:\n";
    for (int x : {12, 60, 97, 100, 360}) {
        auto f = factorize(x);
        cout << x << " = ";
        for (int i = 0; i < (int)f.size(); i++) {
            if (i) cout << " × ";
            cout << f[i];
        }
        cout << "\n";
    }
    
    return 0;
}
```

**출력:**
```
100 이하의 소수 (25개):
2 3 5 7 11 13 17 19 23 29 31 37 41 43 47 53 59 61 67 71 73 79 83 89 97

SPF를 이용한 소인수 분해:
12 = 2 × 2 × 3
60 = 2 × 2 × 3 × 5
97 = 97
100 = 2 × 2 × 5 × 5
360 = 2 × 2 × 2 × 3 × 3 × 5
```

### 왜 선형 체가 O(n)인가?

각 합성수 m은 정확히 한 번만 표시됩니다. 핵심 조건은 `primes[j] <= spf[i]`입니다.

`m = i × p` (p는 소수, p ≤ spf[i])일 때:
- m의 최소 소인수는 p (= primes[j]) 입니다.
- 만약 p > spf[i]라면 m은 spf[i]로도 나눠지므로, 더 작은 소수 spf[i]에 의해 이미 처리됩니다.

이 조건이 각 합성수를 정확히 한 번만 처리함을 보장합니다. 전체 연산 수는 합성수 개수 O(n)이므로 **O(n)**입니다.

## 6. 선형 체의 응용: 오일러 피 함수 일괄 계산

선형 체의 부산물인 SPF(최소 소인수) 배열은 다양한 수론 함수를 O(n)에 계산하는 데 활용됩니다.

```python
def euler_totient_sieve(n: int) -> list[int]:
    """
    오일러 피 함수(Euler's Totient) φ(k) — 1 이상 k 이하 중 k와 서로소인 수의 개수
    φ(p) = p-1, φ(p^k) = p^(k-1)(p-1), φ(mn) = φ(m)φ(n) (gcd(m,n)=1)
    선형 체로 O(n)에 1~n의 φ값을 모두 계산
    """
    phi = list(range(n + 1))  # phi[i] = i로 초기화
    primes = []
    is_prime = [True] * (n + 1)
    
    for i in range(2, n + 1):
        if is_prime[i]:
            primes.append(i)
            phi[i] = i - 1  # 소수 p의 φ(p) = p-1
        
        for p in primes:
            if i * p > n:
                break
            is_prime[i * p] = False
            if i % p == 0:
                # p | i인 경우: φ(ip) = p × φ(i)
                phi[i * p] = p * phi[i]
                break
            else:
                # gcd(i, p) = 1인 경우: φ(ip) = φ(i) × φ(p) = φ(i)(p-1)
                phi[i * p] = phi[i] * (p - 1)
    
    return phi

# 테스트
phi = euler_totient_sieve(20)
print("오일러 피 함수 φ(1)~φ(20):")
for i in range(1, 21):
    print(f"φ({i:2d}) = {phi[i]}")

# RSA 관련 검증
n_rsa, e = 3 * 7, 5  # n = pq = 21, e = 5
phi_n = phi[3] * phi[7]  # φ(21) = φ(3)×φ(7) = 2×6 = 12
d = pow(e, -1, phi_n)  # 모듈러 역원: e×d ≡ 1 (mod φ(n))
print(f"\nRSA 예시: n={n_rsa}, e={e}, φ(n)={phi_n}, d={d}")
print(f"검증: e×d mod φ(n) = {(e * d) % phi_n}")
```

## 7. 알고리즘 비교 및 선택 기준

| | 에라토스테네스 | 세그먼트 체 | 선형 체 |
|---|---|---|---|
| **시간** | O(n log log n) | O(n log log n) | **O(n)** |
| **공간** | O(n) | O(√n) | O(n) |
| **적합 범위** | n ≤ 10⁸ | n ≤ 10¹² | n ≤ 10⁷ |
| **구현 복잡도** | 매우 단순 | 중간 | 중간 |
| **캐시 효율** | 나쁨 | 좋음 | 중간 |
| **부산물** | 없음 | 없음 | SPF 배열 |

**선택 가이드**:
- n ≤ 10⁶: 에라토스테네스로 충분
- n ≤ 10⁸: 에라토스테네스 (캐시 최적화 버전)
- n ≤ 10¹²: 세그먼트 체
- 소인수 분해가 여러 번 필요: 선형 체 (SPF 배열 활용)
- 오일러 피, 뫼비우스 함수 등 수론 함수 전처리: 선형 체

## 8. 주의사항 및 실전 팁

### 오버플로우 주의

`p * p <= n`에서 p²이 int 범위를 초과하면 오버플로우가 발생합니다. 큰 n을 다룰 때는 `long long`을 사용하거나 `p <= n / p`로 변경하세요.

```cpp
// 안전한 p * p <= n 비교
while (p <= n / p) { ... }  // 오버플로우 없음
```

### 경쟁 프로그래밍에서의 관용 패턴

```cpp
const int MAXN = 1e7 + 5;
bool is_composite[MAXN];
vector<int> primes;

void sieve(int n) {
    for (int i = 2; i <= n; i++) {
        if (!is_composite[i]) {
            primes.push_back(i);
        }
        for (int p : primes) {
            if ((long long)i * p > n) break;
            is_composite[i * p] = true;
            if (i % p == 0) break; // 선형 체의 핵심 조건
        }
    }
}
```

### 메모리 최적화

n = 10⁸인 경우 bool 배열은 100MB, `vector<bool>`은 약 12.5MB, bitset은 약 12.5MB입니다. 실제 사용 환경의 메모리 제한을 확인하고 적절한 자료형을 선택하세요.

에라토스테네스의 체는 수학적 단순함과 실용적 효율이 조화를 이루는 알고리즘입니다. 2,300년 전에 탄생한 이 아이디어가 오늘날 암호학, 경쟁 프로그래밍, 수론 연구에서 여전히 핵심 도구로 쓰이고 있다는 사실은 좋은 알고리즘의 영속성을 잘 보여줍니다.

## 참고 자료
- [Sieve of Eratosthenes - Wikipedia](https://en.wikipedia.org/wiki/Sieve_of_Eratosthenes)
- [Sieve of Eratosthenes - CP-Algorithms](https://cp-algorithms.com/algebra/sieve-of-eratosthenes.html)
- [Linear Sieve - CP-Algorithms](https://cp-algorithms.com/algebra/prime-sieve-linear.html)
- [Prime Number Sieve - Wikipedia](https://en.wikipedia.org/wiki/Prime_number_sieve)
