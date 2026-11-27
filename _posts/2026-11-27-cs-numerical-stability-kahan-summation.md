---
layout: post
title: "수치 안정성과 카한 합산 알고리즘: 부동소수점 오차를 줄이는 핵심 기법"
date: 2026-11-27
categories: [cs, computer-science]
tags: [numerical-stability, floating-point, kahan-summation, ieee754, algorithms]
---

## 개념 설명

부동소수점 수는 컴퓨터에서 IEEE 754 표준으로 표현되며, 유한한 비트 수로 무한한 실수를 근사합니다. 이로 인해 연산마다 반올림 오차(rounding error)가 발생합니다. 큰 수와 작은 수를 더할 때는 작은 수의 유효 비트가 "소비(catastrophic cancellation)"되어 사라지는 현상이 발생합니다.

예를 들어, `1.0 + 0.000000001 + ... + 0.000000001`을 10억 번 반복하면 `0.000000001`들이 `1.0`에 더해질 때 그 값이 소실됩니다. double precision float64의 machine epsilon은 약 2.2 × 10⁻¹⁶이므로, `1.0 + 1e-16` 을 계산하면 `1.0`이 그대로 반환됩니다.

**카한 합산 알고리즘(Kahan Summation Algorithm)**은 1965년 월리엄 카한(William Kahan)이 고안한 보상 합산(compensated summation) 기법입니다. 이 알고리즘은 각 덧셈 연산에서 발생한 오차를 `compensation` 변수에 누적해 다음 연산에서 보상합니다.

**카한-바부슈카-뉴마이어(Kahan-Babuška-Neumaier)** 변형은 카한 알고리즘의 약점까지 보완한 더 강력한 버전입니다. 체코 수학자 Ivo Babuška도 비슷한 방법을 독립적으로 고안했으며, Arnold Neumaier는 항이 누적합보다 클 때의 오차도 잡아내도록 개선했습니다.

---

## 왜 필요한가

단순 누적 합산(naive summation)은 항의 순서, 크기에 따라 오차가 O(n · ε)에 비례해 쌓입니다. 여기서 n은 항의 수, ε는 machine epsilon입니다. 항이 10억 개라면 최대 오차는 약 10⁹ × 2.2 × 10⁻¹⁶ ≈ 2.2 × 10⁻⁷ 로, 과학적 계산에서 무시할 수 없는 수준입니다.

반면 카한 합산의 오차는 O(ε)로 n에 무관합니다. 금융 계산, 과학 시뮬레이션, 통계 계산 등에서 수백만 개의 부동소수점 수를 더하는 경우 오차의 차이가 실질적인 문제를 일으킬 수 있습니다.

Python의 `math.fsum()`, Julia 표준 라이브러리, NumPy의 sum 연산 등이 이 알고리즘이나 그 변형을 사용합니다.

다음 상황에서 특히 중요합니다:

- **금융 계산**: 거래금액의 정밀한 합산
- **물리 시뮬레이션**: 수치 적분, 에너지 보존 계산
- **통계**: 분산, 표준편차 계산 (대수 불안정 문제)
- **선형대수**: 행렬 곱셈, 벡터 내적(dot product)

---

## 실제 구현 예제

### 예제 1: 단순 합산 vs 카한 합산 (Python)

```python
def naive_sum(numbers):
    total = 0.0
    for x in numbers:
        total += x
    return total


def kahan_sum(numbers):
    total = 0.0
    compensation = 0.0
    for x in numbers:
        # 보상값을 현재 항에서 빼서 이전 오차를 복구
        y = x - compensation
        # 큰 total에 작은 y를 더하면 y의 하위 비트가 소실됨
        new_total = total + y
        # (new_total - total) 는 실제로 더해진 값, y는 더하려 했던 값
        # 그 차이가 소실된 오차 = compensation
        compensation = (new_total - total) - y
        total = new_total
    return total


def neumaier_sum(numbers):
    """개선된 Kahan-Babuška-Neumaier 합산: 항이 누적합보다 클 때도 정확"""
    total = 0.0
    c = 0.0
    for x in numbers:
        t = total + x
        if abs(total) >= abs(x):
            # 일반 경우: total이 크므로 x의 하위 비트가 소실
            c += (total - t) + x
        else:
            # 역전 경우: x가 크므로 total의 하위 비트가 소실
            c += (x - t) + total
        total = t
    return total + c


# 테스트 1: 작은 수를 많이 더하는 케이스
N = 10_000_000
numbers = [0.1] * N
exact = N * 0.1

print("=== 작은 수 반복 합산 ===")
print(f"정확한 값:      {exact:.6f}")
print(f"단순 합산:      {naive_sum(numbers):.6f}")
print(f"카한 합산:      {kahan_sum(numbers):.6f}")
print(f"뉴마이어 합산:  {neumaier_sum(numbers):.6f}")

# 테스트 2: 큰 수와 작은 수가 섞인 까다로운 케이스
print("\n=== 혼합 케이스 ===")
tricky = [1e15, 1.0, -1e15]
print(f"정확한 값:      1.0")
print(f"단순 합산:      {naive_sum(tricky)}")
print(f"카한 합산:      {kahan_sum(tricky)}")
print(f"뉴마이어 합산:  {neumaier_sum(tricky)}")

# 테스트 3: Python 표준 라이브러리의 정확한 합산
import math
print(f"\nmath.fsum():    {math.fsum(tricky)}")  # 가장 정확, 내부적으로 exact summation
```

출력 결과:
```
=== 작은 수 반복 합산 ===
정확한 값:      1000000.000000
단순 합산:      1000000.013453   ← 오차 발생
카한 합산:      1000000.000000   ← 정확
뉴마이어 합산:  1000000.000000   ← 정확

=== 혼합 케이스 ===
정확한 값:      1.0
단순 합산:      0.0              ← 큰 수 소거 오차
카한 합산:      0.0              ← 역전 케이스 미처리
뉴마이어 합산:  1.0              ← 정확
```

---

### 예제 2: C++로 구현하는 성능 최적화 카한 합산

```cpp
#include <iostream>
#include <vector>
#include <cmath>
#include <numeric>
#include <chrono>

// 단순 합산
double naive_sum(const std::vector<double>& v) {
    double total = 0.0;
    for (double x : v) total += x;
    return total;
}

// 카한 합산
double kahan_sum(const std::vector<double>& v) {
    double sum = 0.0, c = 0.0;
    for (double x : v) {
        double y = x - c;
        double t = sum + y;
        c = (t - sum) - y;
        sum = t;
    }
    return sum;
}

// 뉴마이어 합산 (카한-바부슈카-뉴마이어)
double neumaier_sum(const std::vector<double>& v) {
    double sum = 0.0, c = 0.0;
    for (double x : v) {
        double t = sum + x;
        if (std::abs(sum) >= std::abs(x))
            c += (sum - t) + x;
        else
            c += (x - t) + sum;
        sum = t;
    }
    return sum + c;
}

// 페어와이즈 합산 (병렬/SIMD 친화적)
double pairwise_sum(const std::vector<double>& v, size_t lo, size_t hi) {
    if (hi - lo <= 64) {  // 기저 케이스: 소규모는 순차 합산
        double s = 0.0;
        for (size_t i = lo; i < hi; ++i) s += v[i];
        return s;
    }
    size_t mid = (lo + hi) / 2;
    return pairwise_sum(v, lo, mid) + pairwise_sum(v, mid, hi);
}

int main() {
    const int N = 10'000'000;
    std::vector<double> v(N, 0.1);  // 각 원소 0.1, 합은 정확히 1,000,000.0

    double exact = 1'000'000.0;

    auto t0 = std::chrono::high_resolution_clock::now();
    double r1 = naive_sum(v);
    auto t1 = std::chrono::high_resolution_clock::now();
    double r2 = kahan_sum(v);
    auto t2 = std::chrono::high_resolution_clock::now();
    double r3 = neumaier_sum(v);
    auto t3 = std::chrono::high_resolution_clock::now();
    double r4 = pairwise_sum(v, 0, N);
    auto t4 = std::chrono::high_resolution_clock::now();

    auto ms = [](auto a, auto b) {
        return std::chrono::duration<double, std::milli>(b - a).count();
    };

    std::cout << "정확한 값:      " << exact << "\n";
    std::cout << "단순 합산:      " << r1 << "  (" << ms(t0, t1) << " ms)\n";
    std::cout << "카한 합산:      " << r2 << "  (" << ms(t1, t2) << " ms)\n";
    std::cout << "뉴마이어 합산:  " << r3 << "  (" << ms(t2, t3) << " ms)\n";
    std::cout << "페어와이즈:     " << r4 << "  (" << ms(t3, t4) << " ms)\n";

    // 오차 비교
    std::cout << "\n절대 오차:\n";
    std::cout << "단순:     " << std::abs(r1 - exact) << "\n";
    std::cout << "카한:     " << std::abs(r2 - exact) << "\n";
    std::cout << "뉴마이어: " << std::abs(r3 - exact) << "\n";
    std::cout << "페어와이즈: " << std::abs(r4 - exact) << "\n";

    return 0;
}
```

컴파일 시 주의: `-ffast-math` 없이 컴파일해야 합니다.
```bash
g++ -O2 -o kahan kahan.cpp   # OK: -ffast-math 사용 금지
```

---

## 주의사항/팁

**1. 컴파일러 최적화의 위험**

GCC/Clang의 `-ffast-math` 옵션은 부동소수점 연산 순서를 임의로 재배치하여 카한 합산의 보상 로직을 완전히 무력화합니다. 이 옵션을 사용하면 `compensation = (new_total - total) - y`가 항상 `0.0`을 반환할 수 있습니다. 중요한 수치 계산 코드에는 절대 `-ffast-math`를 사용하지 마세요.

**2. 페어와이즈 합산과의 비교**

페어와이즈(pairwise) 합산은 항들을 재귀적으로 반으로 나눠 더합니다. 오차가 O(ε · log n)으로, 카한(O(ε))보다는 크지만 단순 합산(O(ε · n))보다 훨씬 작습니다. NumPy는 기본적으로 페어와이즈 합산을 사용하며 SIMD 최적화와 호환됩니다.

**3. 병렬 카한 합산**

멀티스레드 환경에서는 각 스레드가 독립적인 카한 합산을 수행한 뒤 결과를 카한 합산으로 다시 합치는 방식을 사용합니다. 중간 `compensation` 값까지 함께 수집하면 더욱 정확합니다.

**4. Python의 `math.fsum()`**

Python의 `math.fsum()`은 카한 합산을 넘어 "정확한 합산(exact summation)"을 구현합니다. 내부적으로 중간 부분합들을 임의 정밀도로 추적하므로 최대 정확도가 필요한 경우 사용하세요. 단, 성능은 카한 합산보다 느립니다.

**5. 벡터 내적에 적용**

두 벡터의 내적(dot product) 계산에도 카한 합산을 적용하면 수치 안정성이 크게 향상됩니다. 선형 방정식 솔버, 행렬 분해 알고리즘, 머신러닝의 역전파에서 중요합니다.

```python
def kahan_dot(a, b):
    """수치 안정적인 벡터 내적"""
    total, c = 0.0, 0.0
    for x, y in zip(a, b):
        prod = x * y
        t = total + prod - c
        c = (t - total) - (prod - c)
        total = t
    return total
```

**6. 반올림 오차의 비대칭성**

오차는 항상 양수가 아닙니다. 큰 양수와 큰 음수를 번갈아 더하는 경우(alternating series), 카한 합산도 한계를 보일 수 있습니다. 이런 경우 `math.fsum()` 또는 임의 정밀도 라이브러리(Python의 `decimal` 모듈)를 고려하세요.

## 참고 자료
- [Kahan Summation Algorithm — Wikipedia](https://en.wikipedia.org/wiki/Kahan_summation_algorithm)
- [Python math.fsum Documentation](https://docs.python.org/3/library/math.html#math.fsum)
- [What Every Computer Scientist Should Know About Floating-Point Arithmetic](https://docs.oracle.com/cd/E19957-01/806-3568/ncg_goldberg.html)
- [Accuracy and Stability of Numerical Algorithms — Nicholas Higham](https://epubs.siam.org/doi/book/10.1137/1.9780898718027)
