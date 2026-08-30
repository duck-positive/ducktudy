---
layout: post
title: "마스터 정리(Master Theorem) 완전 정복: 분할 정복 알고리즘의 시간 복잡도를 한 줄로 분석하기"
date: 2026-08-30
categories: [cs, computer-science]
tags: [master-theorem, recurrence, algorithm-analysis, divide-and-conquer, complexity]
---

"병합 정렬의 시간 복잡도는 왜 O(n log n)인가?" 이 질문에 답하려면 점화식(recurrence relation)을 풀어야 합니다. 병합 정렬은 `T(n) = 2T(n/2) + O(n)`으로 표현되는데, 이 점화식을 일일이 전개하는 대신 **마스터 정리(Master Theorem)**를 적용하면 한 줄에 답을 구할 수 있습니다. 이 글에서는 마스터 정리의 세 가지 케이스, 일반화된 Akra-Bazzi 방법, 그리고 실제 알고리즘에 적용하는 법까지 완전히 정복합니다.

## 1. 점화식(Recurrence Relation)이란?

알고리즘의 실행 시간 T(n)을 더 작은 입력에 대한 실행 시간으로 표현한 것이 **점화식**입니다. 분할 정복 알고리즘은 일반적으로 다음 형태의 점화식을 가집니다:

```
T(n) = a·T(n/b) + f(n)
```

각 항의 의미:
- **a**: 재귀 호출 횟수 (a ≥ 1인 정수)
- **n/b**: 각 하위 문제의 크기 (b > 1인 실수)
- **f(n)**: 분할과 병합에 드는 비용 (비재귀 부분)

예시:
| 알고리즘 | 점화식 | a | b | f(n) |
|---------|-------|---|---|------|
| 병합 정렬 | T(n) = 2T(n/2) + n | 2 | 2 | n |
| 이진 탐색 | T(n) = T(n/2) + 1 | 1 | 2 | 1 |
| 카라츠바 곱셈 | T(n) = 3T(n/2) + n | 3 | 2 | n |
| 스트라센 행렬 곱 | T(n) = 7T(n/2) + n² | 7 | 2 | n² |

## 2. 마스터 정리: 세 가지 케이스

**마스터 정리(Master Theorem)**는 Jon Bentley, Dorothea Blostein, James B. Saxe가 1980년 제안했고, CLRS(Introduction to Algorithms) 교재로 널리 알려졌습니다.

### 핵심 비교: n^(log_b a) vs f(n)

마스터 정리의 핵심은 재귀 비용 `n^(log_b(a))`와 비재귀 비용 `f(n)`을 비교하는 것입니다.

- `n^(log_b(a))`는 재귀 호출 트리의 **리프 수**에 해당합니다.
- `f(n)`은 각 레벨에서의 **분할·병합 비용**입니다.

### Case 1: 재귀 비용이 지배적

**조건**: `f(n) = O(n^(log_b(a) - ε))` for some ε > 0

(f(n)이 n^(log_b a)보다 **다항식으로 작을** 때)

**결론**: `T(n) = Θ(n^(log_b(a)))`

**직관**: 재귀 트리의 리프에서 대부분의 일이 일어납니다.

### Case 2: 비용이 균등

**조건**: `f(n) = Θ(n^(log_b(a)) · log^k(n))` for some k ≥ 0

(f(n)이 n^(log_b a)와 **다항식으로 동일할** 때)

**결론**: `T(n) = Θ(n^(log_b(a)) · log^(k+1)(n))`

**가장 흔한 케이스** (k=0): `f(n) = Θ(n^(log_b a))` → `T(n) = Θ(n^(log_b a) · log n)`

### Case 3: 비재귀 비용이 지배적

**조건**: 
1. `f(n) = Ω(n^(log_b(a) + ε))` for some ε > 0
2. **정규 조건(Regularity Condition)**: `a·f(n/b) ≤ c·f(n)` for some c < 1 and large n

**결론**: `T(n) = Θ(f(n))`

**직관**: 루트(첫 번째 분할)에서 대부분의 일이 일어납니다.

## 3. 마스터 정리 직접 적용해보기

### 예제 1: 병합 정렬

```
T(n) = 2T(n/2) + n
```

- a = 2, b = 2, f(n) = n
- log_b(a) = log₂(2) = 1
- n^(log_b(a)) = n^1 = n
- f(n) = n = Θ(n^1) = Θ(n^(log_b(a)))

→ **Case 2** (k=0): `T(n) = Θ(n log n)` ✓

### 예제 2: 이진 탐색

```
T(n) = T(n/2) + 1
```

- a = 1, b = 2, f(n) = 1
- log_b(a) = log₂(1) = 0
- n^0 = 1
- f(n) = 1 = Θ(1) = Θ(n^0)

→ **Case 2** (k=0): `T(n) = Θ(log n)` ✓

### 예제 3: 카라츠바 알고리즘 (빠른 곱셈)

```
T(n) = 3T(n/2) + n
```

- a = 3, b = 2, f(n) = n
- log_b(a) = log₂(3) ≈ 1.585
- n^(log₂3) ≈ n^1.585 > n
- f(n) = n = O(n^(log₂3 - ε)) with ε = log₂3 - 1 ≈ 0.585

→ **Case 1**: `T(n) = Θ(n^(log₂3)) = Θ(n^1.585)` ✓

(단순 곱셈 O(n²)보다 훨씬 빠름)

### 예제 4: 스트라센 행렬 곱셈

```
T(n) = 7T(n/2) + n²
```

- a = 7, b = 2, f(n) = n²
- log_b(a) = log₂(7) ≈ 2.807
- n^2.807 > n²
- f(n) = n² = O(n^(log₂7 - ε)) with ε ≈ 0.807

→ **Case 1**: `T(n) = Θ(n^(log₂7)) ≈ Θ(n^2.807)` ✓

(단순 행렬 곱 O(n³)보다 빠름)

## 4. 파이썬으로 마스터 정리 자동 분류기 구현

```python
import math
from fractions import Fraction
from enum import Enum

class MasterCase(Enum):
    CASE1 = "Case 1: T(n) = Θ(n^log_b(a))"
    CASE2 = "Case 2: T(n) = Θ(n^log_b(a) * log^(k+1)(n))"
    CASE3 = "Case 3: T(n) = Θ(f(n))"
    NOT_APPLICABLE = "마스터 정리 적용 불가"

def analyze_master_theorem(a: int, b: float, fn_exponent: float, 
                            fn_log_exponent: float = 0) -> dict:
    """
    T(n) = a·T(n/b) + n^fn_exponent · log^fn_log_exponent(n) 형태의 점화식 분석
    
    Args:
        a: 재귀 호출 횟수
        b: 분할 비율 (> 1)
        fn_exponent: f(n) = n^fn_exponent 의 지수
        fn_log_exponent: f(n)에 log 계수가 있는 경우 (k)
    """
    assert a >= 1 and b > 1, "a ≥ 1, b > 1 이어야 합니다"
    
    log_b_a = math.log(a, b)
    diff = fn_exponent - log_b_a
    eps = 1e-9
    
    result = {
        "a": a, "b": b,
        "log_b(a)": round(log_b_a, 4),
        "f(n) exponent": fn_exponent,
        "diff (f_exp - log_b_a)": round(diff, 4)
    }
    
    if diff < -eps:
        # f(n) = O(n^(log_b_a - ε)): Case 1
        result["case"] = MasterCase.CASE1
        result["solution"] = f"T(n) = Θ(n^{round(log_b_a, 4)})"
        
    elif abs(diff) <= eps:
        # f(n) = Θ(n^log_b_a · log^k n): Case 2
        result["case"] = MasterCase.CASE2
        k = fn_log_exponent
        result["solution"] = f"T(n) = Θ(n^{round(log_b_a, 4)} · log^{k+1}(n))"
        
    elif diff > eps:
        # f(n) = Ω(n^(log_b_a + ε)): Case 3 (정규 조건 가정)
        result["case"] = MasterCase.CASE3
        result["solution"] = f"T(n) = Θ(n^{fn_exponent})"
        result["note"] = "정규 조건(regularity condition) 확인 필요"
    
    return result

def print_analysis(label: str, a: int, b: float, fn_exp: float, fn_log_exp: float = 0):
    r = analyze_master_theorem(a, b, fn_exp, fn_log_exp)
    print(f"\n{'='*50}")
    print(f"점화식: T(n) = {a}T(n/{b}) + n^{fn_exp}" +
          (f" * log^{fn_log_exp}(n)" if fn_log_exp else ""))
    print(f"예시: {label}")
    print(f"log_{b}({a}) = {r['log_b(a)']}")
    print(f"Case: {r['case'].value}")
    print(f"결론: {r['solution']}")
    if 'note' in r:
        print(f"주의: {r['note']}")

# 주요 알고리즘 분석
print_analysis("병합 정렬", a=2, b=2, fn_exp=1)
print_analysis("이진 탐색", a=1, b=2, fn_exp=0)
print_analysis("카라츠바 곱셈", a=3, b=2, fn_exp=1)
print_analysis("스트라센 행렬 곱", a=7, b=2, fn_exp=2)
print_analysis("빠른 선택(Quickselect 평균)", a=1, b=4/3, fn_exp=1)
```

**출력:**
```
==================================================
점화식: T(n) = 2T(n/2) + n^1
예시: 병합 정렬
log_2(2) = 1.0
Case: Case 2: T(n) = Θ(n^log_b(a) * log^(k+1)(n))
결론: T(n) = Θ(n^1.0 · log^1(n))

==================================================
점화식: T(n) = 1T(n/2) + n^0
예시: 이진 탐색
log_2(1) = 0.0
Case: Case 2: T(n) = Θ(n^log_b(a) * log^(k+1)(n))
결론: T(n) = Θ(n^0.0 · log^1(n))

==================================================
점화식: T(n) = 3T(n/2) + n^1
예시: 카라츠바 곱셈
log_2(3) = 1.585
Case: Case 1: T(n) = Θ(n^log_b(a))
결론: T(n) = Θ(n^1.585)
```

## 5. 마스터 정리가 적용되지 않는 경우

### 적용 불가 사례 1: f(n)이 다항식이 아닌 경우

```
T(n) = 2T(n/2) + n·log(n)
```

- f(n) = n·log(n), log_b(a) = 1
- n·log(n) vs n → 차이가 다항식이 아닌 로그 함수
- 마스터 정리 Case 2의 확장으로 처리: k=1이면 `T(n) = Θ(n log² n)` ✓
- 이는 확장된 마스터 정리(Extended Master Theorem)에서 다룹니다.

### 적용 불가 사례 2: 하위 문제 크기가 다른 경우

```
T(n) = T(n/3) + T(2n/3) + n
```

이 점화식은 병합하는 두 부분의 크기가 달라 표준 마스터 정리로 풀 수 없습니다. 이 경우 **Akra-Bazzi 방법**을 사용합니다.

## 6. Akra-Bazzi 방법: 마스터 정리의 일반화

Akra-Bazzi 방법은 다음 형태의 더 일반적인 점화식을 다룹니다:

```
T(n) = Σᵢ aᵢ·T(bᵢ·n) + g(n)
```

여기서 각 하위 문제의 분할 비율 bᵢ가 서로 다를 수 있습니다.

**풀이 과정**:
1. 지수 p를 구합니다: `Σᵢ aᵢ · bᵢᵖ = 1` (이 방정식의 해)
2. 결론: `T(n) = Θ(nᵖ · (1 + ∫₁ⁿ g(u)/u^(p+1) du))`

**예시** `T(n) = T(n/3) + T(2n/3) + n`:

지수 p 구하기: `(1/3)ᵖ + (2/3)ᵖ = 1`

- p=1이면: 1/3 + 2/3 = 1 ✓

따라서 p=1이고:

`∫₁ⁿ u/(u²) du = ∫₁ⁿ 1/u du = log(n)`

결론: `T(n) = Θ(n log n)` (이 점화식은 퀵정렬의 최악 케이스와 관련)

## 7. 점화식 전개법과 비교

마스터 정리를 적용할 수 없을 때는 직접 전개(substitution method)나 재귀 트리(recursion tree) 방법을 씁니다.

**재귀 트리로 병합 정렬 분석**:

```
레벨 0: n          (비용: n)
레벨 1: n/2, n/2   (비용: n)
레벨 2: 4 × n/4    (비용: n)
...
레벨 k: 2^k × n/2^k (비용: n)
...
레벨 log₂n: n × 1  (리프, 비용: n)

총 레벨 수: log₂n + 1
총 비용: n × (log₂n + 1) = Θ(n log n) ✓
```

## 8. 실전 알고리즘 복잡도 정리표

| 알고리즘 | 점화식 | 마스터 정리 케이스 | 시간 복잡도 |
|---------|-------|---------------|-----------|
| 병합 정렬 | 2T(n/2) + n | Case 2 | O(n log n) |
| 퀵 정렬(평균) | 2T(n/2) + n | Case 2 | O(n log n) |
| 이진 탐색 | T(n/2) + 1 | Case 2 | O(log n) |
| 카라츠바 | 3T(n/2) + n | Case 1 | O(n^1.585) |
| 스트라센 | 7T(n/2) + n² | Case 1 | O(n^2.807) |
| 하노이 탑 | 2T(n-1) + 1 | (적용 불가) | O(2^n) |
| FFT | 2T(n/2) + n | Case 2 | O(n log n) |
| 최근접 점 쌍 | 2T(n/2) + n | Case 2 | O(n log n) |

## 9. 마스터 정리의 한계와 주의사항

### 비점근적 상수를 무시한다

마스터 정리는 점근적 복잡도만 알려줍니다. `T(n) = Θ(n log n)`이어도 상수 인자에 따라 실제 성능은 크게 다를 수 있습니다.

### 비교 조건의 경계 케이스

Case 1과 Case 3 사이의 **다항식 차이(polynomial gap)** 조건을 확인해야 합니다. f(n) = n^(log_b a) · log(n)처럼 로그 인자만 차이 나는 경우는 확장 마스터 정리나 직접 전개가 필요합니다.

### 정규 조건(Case 3)

Case 3에서는 정규 조건 `a·f(n/b) ≤ c·f(n)`을 반드시 확인해야 합니다. 대부분의 다항식 f(n)에서는 성립하지만, 이상한 형태의 f(n)에서는 실패할 수 있습니다.

마스터 정리는 알고리즘 분석의 핵심 도구입니다. 새 알고리즘을 설계할 때 분할 정복 구조가 보이면 점화식을 세우고 마스터 정리를 적용하는 습관을 들이면 복잡도 분석 속도가 비약적으로 향상됩니다.

## 참고 자료
- [Master Theorem - Wikipedia](https://en.wikipedia.org/wiki/Master_theorem_%28analysis_of_algorithms%29)
- [Master Theorem - Brilliant.org](https://brilliant.org/wiki/master-theorem/)
- [Recurrence Relation - Wikipedia](https://en.wikipedia.org/wiki/Recurrence_relation)
- [Introduction to Algorithms (CLRS) 4th Edition - Chapter 4](https://mitpress.mit.edu/9780262046305/introduction-to-algorithms/)
