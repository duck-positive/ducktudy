---
layout: post
title: "IEEE 754 부동소수점 표준 완전 정복: 0.1 + 0.2 ≠ 0.3의 비밀"
date: 2026-07-27
categories: [cs, computer-science]
tags: [ieee754, floating-point, 부동소수점, 수치해석, 컴퓨터구조]
---

모든 프로그래머가 한 번쯤 당혹감을 느끼는 코드가 있다.

```python
>>> 0.1 + 0.2
0.30000000000000004
>>> 0.1 + 0.2 == 0.3
False
```

버그처럼 보이지만 이것은 완벽하게 올바른 동작이다. 이 현상을 이해하려면 컴퓨터가 실수(real number)를 어떻게 저장하는지, 즉 IEEE 754 부동소수점 표준을 이해해야 한다.

## 개념: 부동소수점이란 무엇인가

### 왜 고정소수점이 아닌가

가장 단순한 방법은 **고정소수점(fixed-point)**이다. 예를 들어 32비트 중 16비트는 정수부, 16비트는 소수부에 고정 할당하는 방식이다. 이 방법은 직관적이지만 치명적인 단점이 있다. 표현 범위와 정밀도를 동시에 확보하기 어렵다. 1.0000001 같은 수는 정밀하게, 그러면서 동시에 1,000,000,000,000 같은 큰 수도 표현하려면 비트가 턱없이 부족하다.

과학적 표기법(scientific notation)에서 해법을 얻었다. `1.23456 × 10^8` 처럼 유효숫자와 지수를 분리하면, 작은 수와 큰 수를 모두 효율적으로 표현할 수 있다. 이것이 부동소수점의 핵심 아이디어다.

### IEEE 754 구조

1985년 IEEE가 발표한 IEEE 754 표준은 부동소수점 수를 세 필드로 인코딩한다:

```
단정밀도(single precision, 32비트):
┌───┬────────────┬──────────────────────────┐
│ S │  Exponent  │        Fraction           │
│ 1 │    8 bits  │         23 bits           │
└───┴────────────┴──────────────────────────┘

배정밀도(double precision, 64비트):
┌───┬────────────────┬──────────────────────────────────────────────────────┐
│ S │   Exponent     │                     Fraction                         │
│ 1 │    11 bits     │                      52 bits                         │
└───┴────────────────┴──────────────────────────────────────────────────────┘
```

- **부호(Sign, S)**: 0이면 양수, 1이면 음수
- **지수(Exponent)**: 편향(bias) 방식으로 저장. 단정밀도는 bias=127, 배정밀도는 bias=1023
- **가수(Fraction/Mantissa)**: 소수점 이하 부분 (정규화된 수는 항상 `1.xxx` 형태이므로 1은 암묵적으로 생략)

수의 실제 값은 다음 공식으로 계산한다:

```
값 = (-1)^S × 1.Fraction × 2^(Exponent - Bias)
```

### 0.1은 이진수로 표현할 수 없다

10진수 0.1을 이진수로 변환해보자.

```
0.1 × 2 = 0.2  → 0
0.2 × 2 = 0.4  → 0
0.4 × 2 = 0.8  → 0
0.8 × 2 = 1.6  → 1
0.6 × 2 = 1.2  → 1
0.2 × 2 = 0.4  → 0  ← 반복 시작
...
```

결과: `0.000110011001100110011...` (무한 반복). 이진수로는 0.1을 정확히 표현할 수 없다. 52비트 가수로 어느 지점에서 잘라야 하므로 반올림 오류가 발생한다.

## 왜 필요한가: 특수값과 예외 처리

IEEE 754가 단순한 인코딩 규칙이 아닌 이유는 특수값(special values)과 예외 처리 규칙을 정의하기 때문이다.

### 특수값의 종류

| 지수 필드 | 가수 필드 | 의미 |
|-----------|-----------|------|
| 모두 0 | 모두 0 | ±0 |
| 모두 0 | 0이 아님 | 비정규화수(Subnormal) |
| 모두 1 | 모두 0 | ±무한대(Infinity) |
| 모두 1 | 0이 아님 | NaN(Not a Number) |
| 그 외 | 임의 | 정규화수(Normal) |

```python
import math

# 양의 무한대
pos_inf = float('inf')
neg_inf = float('-inf')
print(pos_inf > 1e308)   # True
print(1.0 / pos_inf)     # 0.0

# NaN - 자기 자신과도 같지 않다
nan = float('nan')
print(nan == nan)        # False (!)
print(math.isnan(nan))   # True

# 0으로 나누기: 예외 대신 Infinity 반환
print(1.0 / 0.0)  # ZeroDivisionError (Python은 예외)
# C에서는: INFINITY 반환 (IEEE 754 준수)

# 비정규화수(subnormal): 0에 가장 가까운 표현 가능한 수
import sys
min_normal = sys.float_info.min       # 약 2.2e-308
min_subnormal = 5e-324                # 비정규화수의 최솟값
print(min_subnormal > 0)             # True
print(min_subnormal / 2 == 0)        # True (언더플로우)
```

### 비정규화수(Subnormal Numbers)

정규화수는 항상 `1.xxx × 2^e` 형태라 0과 최소 정규화수 사이에 큰 간격이 생긴다. IEEE 754는 이 간격을 비정규화수로 채운다. 지수가 모두 0일 때 암묵적 선두 비트가 1이 아닌 0이 된다:

```
비정규화수 값 = (-1)^S × 0.Fraction × 2^(1 - Bias)
```

비정규화수 덕분에 점진적 언더플로우(gradual underflow)가 가능하다. 수가 0에 가까워질수록 정밀도가 점차 줄어드는 방식으로, 갑작스러운 0으로의 전환을 막는다.

## 실제 구현 예제

### 예제 1: 부동소수점 비트 표현 직접 보기

```c
#include <stdio.h>
#include <stdint.h>
#include <string.h>

// float의 비트 표현 분석
void analyze_float(float f) {
    uint32_t bits;
    memcpy(&bits, &f, sizeof(bits));  // type-punning 없이 안전하게 복사

    uint32_t sign     = (bits >> 31) & 0x1;
    uint32_t exponent = (bits >> 23) & 0xFF;
    uint32_t fraction = bits & 0x7FFFFF;

    printf("Value: %f\n", f);
    printf("Bits:  %08X\n", bits);
    printf("Sign:  %u\n", sign);
    printf("Exp:   %u (실제 지수 = %d)\n", exponent, (int)exponent - 127);
    printf("Frac:  %06X (%u)\n", fraction, fraction);

    // 실제 값 재계산
    if (exponent == 0xFF) {
        if (fraction == 0) printf("특수값: %sInfinity\n", sign ? "-" : "+");
        else               printf("특수값: NaN\n");
    } else if (exponent == 0) {
        double val = (double)fraction / (1 << 23) * (1.0 / (1 << 126));
        printf("비정규화수: %.20e\n", val * (sign ? -1 : 1));
    } else {
        double mantissa = 1.0 + (double)fraction / (1 << 23);
        int exp = (int)exponent - 127;
        printf("값 재계산: %.20f\n", (sign ? -1.0 : 1.0) * mantissa * pow(2.0, exp));
    }
    printf("\n");
}

int main() {
    analyze_float(0.1f);
    analyze_float(1.0f);
    analyze_float(0.0f / 0.0f);   // NaN (컴파일러에 따라 경고)
    return 0;
}
```

실행 결과 (0.1f):
```
Value: 0.100000
Bits:  3DCCCCCD
Sign:  0
Exp:   121 (실제 지수 = -4)
Frac:  4CCCCD (5033165)
값 재계산: 0.10000000149011611938
```

0.1을 float으로 저장하면 실제로는 `0.10000000149...`가 저장된다. 이것이 오류의 근원이다.

### 예제 2: 안전한 부동소수점 비교

```python
import math
import struct

def float_to_bits(f: float) -> int:
    """double(64비트) 부동소수점을 정수 비트로 변환"""
    return struct.unpack('Q', struct.pack('d', f))[0]

def ulp(f: float) -> float:
    """ULP(Unit in the Last Place): 해당 수의 최소 표현 단위"""
    if math.isnan(f) or math.isinf(f):
        return f
    bits = float_to_bits(abs(f))
    bits_plus_1 = bits + 1
    next_f = struct.unpack('d', struct.pack('Q', bits_plus_1))[0]
    return next_f - abs(f)

def nearly_equal(a: float, b: float, max_ulps: int = 4) -> bool:
    """ULP 기반 부동소수점 비교 (올바른 방법)"""
    if math.isnan(a) or math.isnan(b):
        return False
    if math.isinf(a) or math.isinf(b):
        return a == b
    bits_a = float_to_bits(a)
    bits_b = float_to_bits(b)
    # 부호가 다르면 (±0 제외) 다른 수
    if (bits_a >> 63) != (bits_b >> 63):
        return a == b  # ±0 처리
    return abs(bits_a - bits_b) <= max_ulps

# 잘못된 비교
print(0.1 + 0.2 == 0.3)                      # False (틀림!)

# 올바른 비교 방법들
print(math.isclose(0.1 + 0.2, 0.3))          # True
print(nearly_equal(0.1 + 0.2, 0.3))          # True
print(abs(0.1 + 0.2 - 0.3) < 1e-9)          # True (단순한 epsilon 방식)

# ULP 확인
x = 1.0
print(f"1.0의 ULP: {ulp(x)}")               # 2.220446049250313e-16
print(f"1000.0의 ULP: {ulp(1000.0)}")       # 1.1368683772161603e-13
# 큰 수일수록 ULP(정밀도)가 커진다!

# Decimal 모듈: 정확한 십진수 연산
from decimal import Decimal, getcontext
getcontext().prec = 50
print(Decimal('0.1') + Decimal('0.2'))       # 0.3 (정확!)
print(Decimal('0.1') + Decimal('0.2') == Decimal('0.3'))  # True
```

## 주의사항과 실전 팁

### 1. 누산 오류 (Catastrophic Cancellation)

비슷한 두 수를 빼면 유효 자릿수가 급감한다:

```python
a = 1.0000000000000002
b = 1.0000000000000001
print(a - b)  # 0.0 (!)  실제로는 1e-16이어야 하지만 정보 손실
```

**해결책**: Kahan 보상 합산(Kahan Summation Algorithm) 사용

```python
def kahan_sum(numbers):
    total = 0.0
    compensation = 0.0
    for num in numbers:
        y = num - compensation
        t = total + y
        compensation = (t - total) - y
        total = t
    return total

# 일반 sum vs Kahan sum
nums = [0.1] * 10
print(sum(nums))         # 0.9999999999999999
print(kahan_sum(nums))   # 1.0
```

### 2. 반올림 모드

IEEE 754는 5가지 반올림 모드를 정의한다:

- **Round to nearest, ties to even** (기본값): 0.5는 짝수 쪽으로 (2.5 → 2, 3.5 → 4)
- **Round toward +∞**: 양의 무한대 방향
- **Round toward -∞**: 음의 무한대 방향
- **Round toward zero**: 절삭
- **Round to nearest, ties away from zero**: 수학적 반올림

### 3. NaN의 전파

NaN은 감염된다. NaN이 포함된 모든 산술 연산은 NaN을 반환한다:

```python
nan = float('nan')
print(nan + 1)    # nan
print(nan * 0)    # nan
print(nan < 0)    # False
print(nan > 0)    # False
print(nan == nan) # False ← 이를 이용한 NaN 검출 트릭
```

### 4. 금융 계산은 절대로 float을 쓰지 마라

금융 시스템에서 0.1원 단위의 오차가 누적되면 수백만 원 규모의 불일치가 발생한다. 금융 계산에는 반드시 `Decimal` (Python), `BigDecimal` (Java), 또는 정수 기반 연산 (센트 단위로 저장)을 사용해야 한다.

### 5. 비교 연산의 교훈

| 상황 | 권장 방법 |
|------|-----------|
| 일반적인 비교 | `math.isclose(a, b, rel_tol=1e-9)` |
| 0에 가까운 수 비교 | `abs(a - b) < epsilon` (절대 허용오차) |
| 정렬/분류에 쓸 때 | ULP 기반 비교 |
| 금융/회계 | `Decimal` 타입 사용 |

## 결론

IEEE 754는 단순한 비트 패킹 규칙이 아니다. 부동소수점 수의 표현 범위, 정밀도, 예외 처리, 반올림 동작을 포괄하는 완전한 산술 체계다. `0.1 + 0.2 ≠ 0.3`은 버그가 아니라 이진 부동소수점의 수학적 한계에서 비롯된 필연적 결과다.

실무에서 핵심은 세 가지다:
1. 부동소수점으로 `==` 비교하지 마라
2. 금융 계산에는 `Decimal`을 써라
3. 누산 오류가 우려될 때는 Kahan 합산을 고려하라

부동소수점을 이해하면 수십 시간의 디버깅을 수 분으로 단축할 수 있다.

## 참고 자료
- [IEEE 754-2008 Standard - Wikipedia](https://en.wikipedia.org/wiki/IEEE_754)
- [IEEE Standard for Floating-Point Arithmetic (IEEE 754-2019)](https://ieeexplore.ieee.org/document/8766229)
- [What Every Computer Scientist Should Know About Floating-Point Arithmetic](https://docs.oracle.com/cd/E19957-01/806-3568/ncg_goldberg.html)
- [Python Floating Point Arithmetic - Python Docs](https://docs.python.org/3/tutorial/floatingpoint.html)
