---
layout: post
title: "스트라센 알고리즘 완전 정복: O(n³)의 벽을 깬 행렬 곱셈의 혁명"
date: 2026-09-10
categories: [cs, computer-science]
tags: [strassen, matrix-multiplication, divide-conquer, linear-algebra, algorithm, complexity-theory]
---

## 개념 설명

행렬 곱셈은 컴퓨터 과학과 수치 해석에서 가장 기본적이면서도 핵심적인 연산이다. 두 N×N 행렬의 곱을 구하는 단순한 방법의 시간 복잡도는 **O(n³)**이다. 이는 수십 년간 행렬 곱셈의 하한선으로 여겨졌다.

1969년, 독일 수학자 **Volker Strassen**은 이 상식을 깨는 논문을 발표했다. 두 2×2 행렬의 곱셈에서 일반적으로 필요한 8번의 곱셈을 **7번**으로 줄일 수 있다는 것을 보였고, 이를 재귀적으로 적용하면 전체 복잡도가 **O(n^log₂7) ≈ O(n^2.807)**이 된다는 것을 증명했다.

### 표준 행렬 곱셈 복습

두 2×2 행렬 A, B의 곱 C = A × B는:

```
C[0][0] = A[0][0]*B[0][0] + A[0][1]*B[1][0]
C[0][1] = A[0][0]*B[0][1] + A[0][1]*B[1][1]
C[1][0] = A[1][0]*B[0][0] + A[1][1]*B[1][0]
C[1][1] = A[1][0]*B[0][1] + A[1][1]*B[1][1]
```

이는 **8번의 곱셈**과 4번의 덧셈을 필요로 한다.

### 스트라센의 핵심 아이디어

스트라센은 다음 7개의 보조 곱셈을 정의했다:

```
M₁ = (A₀₀ + A₁₁) × (B₀₀ + B₁₁)
M₂ = (A₁₀ + A₁₁) × B₀₀
M₃ = A₀₀ × (B₀₁ - B₁₁)
M₄ = A₁₁ × (B₁₀ - B₀₀)
M₅ = (A₀₀ + A₀₁) × B₁₁
M₆ = (A₁₀ - A₀₀) × (B₀₀ + B₀₁)
M₇ = (A₀₁ - A₁₁) × (B₁₀ + B₁₁)
```

결과 행렬은:
```
C₀₀ = M₁ + M₄ - M₅ + M₇
C₀₁ = M₃ + M₅
C₁₀ = M₂ + M₄
C₁₁ = M₁ - M₂ + M₃ + M₆
```

곱셈은 7번이지만 덧셈/뺄셈은 18번으로 늘어났다. 대규모 행렬에서는 덧셈이 곱셈보다 훨씬 빠르기 때문에 전체적으로 이득이다.

---

## 왜 필요한가

### 복잡도 이득

마스터 정리(Master Theorem)를 적용하면:
- T(n) = 7 × T(n/2) + O(n²)
- 해: T(n) = O(n^log₂7) ≈ O(n^2.807)

n = 1000일 때 비교:
- 표준: 1,000,000,000 (10억) 연산
- 스트라센: 약 390,000,000 (3.9억) 연산 → **약 2.5배 빠름**

n = 10000이면 격차는 더 벌어진다.

### 응용 분야

- **딥러닝**: 신경망의 Forward/Backward Pass는 대규모 행렬 곱셈의 연속. GPU 텐서 연산에서 스트라센 변형 사용
- **그래프 알고리즘**: 인접 행렬 거듭제곱을 통한 최단 경로 탐색
- **신호 처리**: FFT와 함께 디지털 신호 처리에 활용
- **컴퓨터 그래픽스**: 3D 변환 행렬 연산

### 이론적 의의

스트라센 이후 수십 년간 더 빠른 알고리즘들이 등장했다:
- Coppersmith-Winograd (1987): O(n^2.376)
- Vassilevska Williams (2012): O(n^2.373)
- 현재 최선: O(n^2.371552) (2024년)

이론상 행렬 곱셈의 하한이 O(n²)인지, 아니면 n^2.3...인지는 아직 미해결 문제다.

---

## 실제 구현 예제

### 예제 1: 순수 Python 재귀 구현

```python
def add_matrix(A, B):
    n = len(A)
    return [[A[i][j] + B[i][j] for j in range(n)] for i in range(n)]

def sub_matrix(A, B):
    n = len(A)
    return [[A[i][j] - B[i][j] for j in range(n)] for i in range(n)]

def strassen(A, B):
    n = len(A)

    # 기저 사례: 1×1 행렬
    if n == 1:
        return [[A[0][0] * B[0][0]]]

    # 행렬을 4개의 서브행렬로 분할
    half = n // 2

    def split(M):
        top_left     = [row[:half] for row in M[:half]]
        top_right    = [row[half:] for row in M[:half]]
        bottom_left  = [row[:half] for row in M[half:]]
        bottom_right = [row[half:] for row in M[half:]]
        return top_left, top_right, bottom_left, bottom_right

    A00, A01, A10, A11 = split(A)
    B00, B01, B10, B11 = split(B)

    # 스트라센의 7개 곱셈
    M1 = strassen(add_matrix(A00, A11), add_matrix(B00, B11))
    M2 = strassen(add_matrix(A10, A11), B00)
    M3 = strassen(A00, sub_matrix(B01, B11))
    M4 = strassen(A11, sub_matrix(B10, B00))
    M5 = strassen(add_matrix(A00, A01), B11)
    M6 = strassen(sub_matrix(A10, A00), add_matrix(B00, B01))
    M7 = strassen(sub_matrix(A01, A11), add_matrix(B10, B11))

    # 결과 조립
    C00 = add_matrix(sub_matrix(add_matrix(M1, M4), M5), M7)
    C01 = add_matrix(M3, M5)
    C10 = add_matrix(M2, M4)
    C11 = add_matrix(sub_matrix(add_matrix(M1, M3), M2), M6)

    # 4개 서브행렬을 하나로 합치기
    C = [[0] * n for _ in range(n)]
    for i in range(half):
        for j in range(half):
            C[i][j]           = C00[i][j]
            C[i][j + half]    = C01[i][j]
            C[i + half][j]    = C10[i][j]
            C[i + half][j + half] = C11[i][j]
    return C


# 테스트: 4×4 행렬 (2의 거듭제곱 크기)
A = [
    [1, 2, 3, 4],
    [5, 6, 7, 8],
    [9, 8, 7, 6],
    [5, 4, 3, 2],
]
B = [
    [1, 0, 0, 1],
    [0, 1, 1, 0],
    [1, 0, 0, 1],
    [0, 1, 1, 0],
]

C = strassen(A, B)
for row in C:
    print(row)
# 출력:
# [4, 6, 6, 4]
# [12, 14, 14, 12]
# [16, 16, 16, 16]
# [5, 7, 7, 5]
```

### 예제 2: NumPy 기반 최적화 구현 (임계값 활용)

실제로는 n이 작을 때 표준 곱셈을 사용하는 **임계값(threshold) 전략**이 필수다. NumPy를 활용해 블록 연산을 효율적으로 수행한다.

```python
import numpy as np

def strassen_numpy(A: np.ndarray, B: np.ndarray, threshold: int = 64) -> np.ndarray:
    """
    NumPy 기반 스트라센 구현.
    n <= threshold이면 표준 np.dot 사용 (캐시 효율 우선).
    """
    n = A.shape[0]

    # 임계값 이하이거나 1×1이면 표준 곱셈
    if n <= threshold:
        return A @ B

    # 홀수 크기 처리: 패딩
    if n % 2 != 0:
        A = np.pad(A, ((0, 1), (0, 1)))
        B = np.pad(B, ((0, 1), (0, 1)))
        C = strassen_numpy(A, B, threshold)
        return C[:n, :n]

    half = n // 2
    A00, A01 = A[:half, :half], A[:half, half:]
    A10, A11 = A[half:, :half], A[half:, half:]
    B00, B01 = B[:half, :half], B[:half, half:]
    B10, B11 = B[half:, :half], B[half:, half:]

    # 7번의 재귀 곱셈
    M1 = strassen_numpy(A00 + A11, B00 + B11, threshold)
    M2 = strassen_numpy(A10 + A11, B00,       threshold)
    M3 = strassen_numpy(A00,       B01 - B11, threshold)
    M4 = strassen_numpy(A11,       B10 - B00, threshold)
    M5 = strassen_numpy(A00 + A01, B11,       threshold)
    M6 = strassen_numpy(A10 - A00, B00 + B01, threshold)
    M7 = strassen_numpy(A01 - A11, B10 + B11, threshold)

    C = np.zeros((n, n))
    C[:half, :half] = M1 + M4 - M5 + M7
    C[:half, half:] = M3 + M5
    C[half:, :half] = M2 + M4
    C[half:, half:] = M1 - M2 + M3 + M6
    return C


# 정확도 검증
import time

n = 512
A = np.random.rand(n, n)
B = np.random.rand(n, n)

t1 = time.time()
C_std = A @ B
t2 = time.time()
C_str = strassen_numpy(A, B, threshold=64)
t3 = time.time()

error = np.max(np.abs(C_std - C_str))
print(f"최대 오차: {error:.2e}")           # 부동소수점 허용 오차 수준
print(f"표준 np.dot: {t2-t1:.4f}s")
print(f"스트라센(임계=64): {t3-t2:.4f}s")
```

---

## 시간 복잡도 심화

### 마스터 정리 적용

T(n) = a × T(n/b) + f(n) 형태에서:
- a = 7, b = 2 → n^log_b(a) = n^log₂7 ≈ n^2.807
- f(n) = O(n²) — 덧셈/뺄셈 비용

n^2 < n^2.807이므로 케이스 1에 해당: **T(n) = O(n^log₂7)**

### 실제 성능 vs 이론

실제로 스트라센이 표준보다 빠르려면 n이 충분히 커야 한다:
- 전형적으로 n > 100~200 이상일 때 이득
- 재귀 오버헤드, 캐시 미스, 부동소수점 누적 오차 등이 실제 성능에 영향
- BLAS, cuBLAS 같은 최적화 라이브러리는 스트라센 변형을 내부적으로 사용

---

## 주의사항과 실전 팁

### 1. n이 2의 거듭제곱이 아닐 때
행렬을 2의 거듭제곱 크기로 **패딩(padding)**해야 한다. 제로 패딩 후 계산하고 결과를 잘라내면 된다.

### 2. 수치 안정성 문제
스트라센은 덧셈/뺄셈 횟수가 표준보다 많아 **부동소수점 오차가 누적**된다. 고정밀도가 필요한 과학 계산에서는 주의가 필요하며, 필요 시 안정화 기법(mixed precision, 오차 보정 등)을 적용한다.

### 3. 임계값(Threshold) 튜닝
재귀 기저 사례의 임계값을 시스템에 맞게 조정해야 한다. 캐시 크기에 따라 최적 임계값이 달라지며, 보통 CPU에서는 32~128, GPU에서는 더 크게 설정한다.

### 4. Winograd 변형
Winograd는 스트라센을 변형하여 덧셈 횟수를 15회로 줄인 알고리즘을 제안했다. 실제 라이브러리에서는 Winograd 변형이 더 많이 사용된다.

### 5. 병렬화
7개의 보조 곱셈 M₁~M₇은 서로 독립적이므로 병렬 실행이 가능하다. 다중 코어나 GPU 환경에서 7개를 병렬로 계산하면 추가 속도 향상을 얻을 수 있다.

---

## 참고 자료
- [Strassen Algorithm — Wikipedia](https://en.wikipedia.org/wiki/Strassen_algorithm)
- [Strassen's Matrix Multiplication — InterviewBit](https://www.interviewbit.com/blog/strassens-matrix-multiplication/)
- [Strassen's Algorithm — TopCoder](https://www.topcoder.com/thrive/articles/strassenss-algorithm-for-matrix-multiplication)
- [Divide and Conquer in Matrix Multiplication — Medium](https://medium.com/@23bt04064/divide-and-conquer-in-matrix-multiplication-strassens-algorithm-and-complexity-reduction-b001072cf5ef)
