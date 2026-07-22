---
layout: post
title: "선형 프로그래밍과 심플렉스 알고리즘: 이론과 구현"
date: 2026-07-22
categories: [cs, computer-science]
tags: [linear-programming, simplex-algorithm, optimization, operations-research, algorithm, scipy]
---

**선형 프로그래밍(Linear Programming, LP)**은 선형 목적 함수를 선형 제약 조건 하에서 최적화하는 수학적 기법입니다. George Dantzig가 1947년 개발한 **심플렉스 알고리즘(Simplex Algorithm)**은 LP 문제를 해결하는 가장 대표적인 방법으로, 60년이 지난 지금도 실무에서 광범위하게 사용됩니다. 자원 배분, 네트워크 최적화, 물류, 금융 등 수많은 분야에서 핵심 역할을 합니다.

## 개념 설명: 선형 프로그래밍이란?

LP 문제는 다음과 같이 표준형으로 정의됩니다.

```
최대화: c^T x
조건:   Ax ≤ b
        x ≥ 0
```

- **x**: 결정 변수 벡터 (n차원)
- **c**: 목적 함수 계수 벡터
- **A**: 제약 조건 행렬 (m×n)
- **b**: 제약 조건 우변 벡터

### 간단한 예시

과자 공장에서 제품 A, B를 생산합니다.

- 제품 A: 단위당 이익 5만 원, 원료 2kg, 노동 1시간
- 제품 B: 단위당 이익 4만 원, 원료 1kg, 노동 2시간
- 원료 제한: 100kg, 노동 제한: 80시간

**목적함수**: max 5x₁ + 4x₂  
**제약조건**: 2x₁ + x₂ ≤ 100, x₁ + 2x₂ ≤ 80, x₁, x₂ ≥ 0

---

## 왜 심플렉스 알고리즘인가?

LP의 실현 가능 영역(feasible region)은 **볼록 다면체(convex polytope)**입니다. 최적해는 반드시 이 다면체의 **꼭짓점(vertex, corner point)**에서 달성됩니다. 이를 **기본 가능 해(Basic Feasible Solution, BFS)** 라고 합니다.

심플렉스 알고리즘은 BFS에서 시작해 **목적 함수 값을 개선하는 인접 꼭짓점으로 이동**을 반복합니다. 꼭짓점 수는 최악 지수적이지만, 실제로는 거의 항상 다항 시간 내에 수렴합니다.

---

## 심플렉스 알고리즘 구현

### 단계 1: 표준형으로 변환 (슬랙 변수 도입)

부등식 제약 `Ax ≤ b`를 등식으로 변환합니다.

```
2x₁ + x₂ + s₁        = 100
x₁ + 2x₂      + s₂   = 80
목적함수: 5x₁ + 4x₂ → max
```

s₁, s₂는 **슬랙 변수(slack variables)**입니다.

### 단계 2: 심플렉스 타블로 구성

```python
import numpy as np

def simplex_maximize(c, A, b):
    """
    표준형 LP 최대화 심플렉스 알고리즘
    maximize:  c^T x
    subject to: Ax <= b, x >= 0

    Returns: (optimal_value, solution_x)
    """
    m, n = A.shape
    # 슬랙 변수 추가해 등식 형태로 변환
    # 타블로: [A | I | b]
    #          [-c^T | 0 | 0]  (목적 함수 행)
    tableau = np.zeros((m + 1, n + m + 1))

    # 제약 조건 행
    tableau[:m, :n] = A
    tableau[:m, n:n+m] = np.eye(m)
    tableau[:m, -1] = b

    # 목적 함수 행 (최대화를 위해 음수화)
    tableau[m, :n] = -c

    # 기저 변수 인덱스 (초기: 슬랙 변수들)
    basis = list(range(n, n + m))

    def pivot(row, col):
        """피벗 연산: row 행의 col 열 원소를 1로 만들고 나머지 행을 소거"""
        tableau[row] /= tableau[row, col]
        for r in range(m + 1):
            if r != row:
                tableau[r] -= tableau[r, col] * tableau[row]
        basis[row] = col

    iteration = 0
    max_iter = 1000

    while iteration < max_iter:
        iteration += 1
        # 진입 변수 선택: 목적 함수 행에서 가장 음수인 계수 (Bland's rule 대안)
        obj_row = tableau[m, :-1]
        if np.all(obj_row >= -1e-10):
            break  # 최적해 도달

        entering = np.argmin(obj_row)

        # 이탈 변수 선택: 최소 비율 검사 (minimum ratio test)
        ratios = []
        for r in range(m):
            if tableau[r, entering] > 1e-10:
                ratios.append((tableau[r, -1] / tableau[r, entering], r))

        if not ratios:
            raise ValueError("문제가 비유계(unbounded)입니다")

        _, leaving_row = min(ratios)
        pivot(leaving_row, entering)

    # 결과 추출
    solution = np.zeros(n)
    for r, b_idx in enumerate(basis):
        if b_idx < n:
            solution[b_idx] = tableau[r, -1]

    optimal_value = tableau[m, -1]
    return optimal_value, solution


# 과자 공장 예시
c = np.array([5.0, 4.0])          # 이익 계수
A = np.array([[2.0, 1.0],         # 원료 제약
              [1.0, 2.0]])         # 노동 제약
b = np.array([100.0, 80.0])       # 자원 한도

optimal, solution = simplex_maximize(c, A, b)
print(f"최적 이익: {optimal:.2f}만 원")
print(f"제품 A 생산량: {solution[0]:.2f} 단위")
print(f"제품 B 생산량: {solution[1]:.2f} 단위")

# 출력:
# 최적 이익: 280.00만 원
# 제품 A 생산량: 40.00 단위
# 제품 B 생산량: 20.00 단위
```

### 단계 3: SciPy를 활용한 실무 적용

```python
from scipy.optimize import linprog
import numpy as np

def solve_production_planning():
    """
    생산 계획 최적화 (SciPy linprog 활용)
    
    공장에서 제품 4종을 생산. 각 제품의 이익, 기계 시간, 원자재 소비를 고려.
    목표: 총 이익 최대화
    """
    # 이익 계수 (linprog는 최소화이므로 음수화)
    profits = np.array([10.0, 6.0, 4.0, 8.0])   # 제품 A, B, C, D
    c = -profits  # 최대화 → 최소화로 변환

    # 제약 조건 행렬 (부등식: Ax <= b)
    A_ub = np.array([
        [1.0, 1.0, 1.0, 1.0],   # 총 생산량 제한
        [3.0, 1.0, 2.0, 4.0],   # 기계 1 시간 (시간/단위)
        [2.0, 3.0, 1.0, 2.0],   # 기계 2 시간
        [1.0, 0.0, 2.0, 1.0],   # 원자재 M1 (kg/단위)
    ])
    b_ub = np.array([100.0, 300.0, 250.0, 80.0])

    # 등식 제약 (없음)
    A_eq = None
    b_eq = None

    # 변수 범위 (비음수 조건)
    bounds = [(0, None)] * 4

    result = linprog(
        c,
        A_ub=A_ub,
        b_ub=b_ub,
        A_eq=A_eq,
        b_eq=b_eq,
        bounds=bounds,
        method='highs'  # HiGHS 솔버 (scipy 1.6+ 기본값, 내부 심플렉스/내점법)
    )

    if result.success:
        print("=== 생산 계획 최적화 결과 ===")
        products = ['A', 'B', 'C', 'D']
        for i, prod in enumerate(products):
            if result.x[i] > 0.001:
                print(f"  제품 {prod}: {result.x[i]:.2f} 단위")
        print(f"최대 이익: {-result.fun:.2f}만 원")
        print(f"솔버 반복 횟수: {result.nit}")
    else:
        print(f"최적화 실패: {result.message}")


solve_production_planning()


def network_flow_as_lp():
    """
    네트워크 최대 유량 문제를 LP로 표현
    
    소스(0) → 노드1 → 노드2 → 싱크(3) 네트워크
    간선: (0→1, cap=10), (0→2, cap=8), (1→3, cap=6), (2→3, cap=7), (1→2, cap=5)
    """
    # 변수: x01, x02, x13, x23, x12 (각 간선의 유량)
    # 최대화: x13 + x23 (싱크로의 총 유량)
    c = np.array([0.0, 0.0, -1.0, -1.0, 0.0])  # 최소화: -(x13+x23)

    # 용량 제약: x_ij <= cap_ij
    A_cap = np.eye(5)
    b_cap = np.array([10.0, 8.0, 6.0, 7.0, 5.0])

    # 유량 보존 제약 (등식): 각 노드에서 유입 = 유출
    # 노드1: x01 + x12_rev - x12 - x13 = 0  (단순화)
    # 실제 구현에서는 방향 그래프의 보존 법칙 적용
    A_eq = np.array([
        [1.0, 0.0, -1.0, 0.0, -1.0],  # 노드 1 보존
        [0.0, 1.0,  0.0, -1.0, 1.0],  # 노드 2 보존
    ])
    b_eq = np.array([0.0, 0.0])

    bounds = [(0, cap) for cap in [10, 8, 6, 7, 5]]

    result = linprog(c, A_ub=A_cap, b_ub=b_cap,
                     A_eq=A_eq, b_eq=b_eq,
                     bounds=bounds, method='highs')

    if result.success:
        edges = ['0→1', '0→2', '1→3', '2→3', '1→2']
        print("\n=== 네트워크 최대 유량 (LP 방식) ===")
        for e, f in zip(edges, result.x):
            print(f"  {e}: {f:.2f}")
        print(f"  최대 유량: {-result.fun:.2f}")


network_flow_as_lp()
```

---

## 알고리즘 분석

### 시간 복잡도

| 방법 | 최악 복잡도 | 실제 성능 |
|------|------------|-----------|
| 심플렉스 | 지수 시간 (Klee-Minty) | 다항 시간 수렴 |
| 내점법 (Interior Point) | O(n^3.5 L) | 이론적 다항 |
| 타원체법 (Ellipsoid) | O(n^6 L) | 실제로는 느림 |

심플렉스 알고리즘은 이론적 최악 복잡도는 지수이지만, Smale의 9번 문제가 가리키듯 실제 임의 문제에서는 다항 시간으로 동작합니다.

### Bland's Rule (순환 방지)

진입/이탈 변수 선택 시 순환(cycling) 문제가 발생할 수 있습니다. **Bland's Rule**: 진입 변수로 가장 작은 인덱스의 음수 계수 변수를 선택하면 순환이 방지됩니다.

---

## 주의사항 및 팁

### 1. 실무에서는 전문 솔버 사용

직접 구현한 심플렉스보다 `scipy.optimize.linprog` (HiGHS), `PuLP` (GLPK/CBC), `Gurobi`, `CPLEX` 등의 검증된 솔버를 사용하세요. 수치 안정성과 성능이 비교할 수 없이 뛰어납니다.

### 2. 이중성 (Duality)

모든 LP 문제(원문제, primal)에는 대응하는 **쌍대 문제(dual)**가 존재합니다. 원문제 최대값 = 쌍대 문제 최소값 (강한 쌍대성). 이를 활용해 민감도 분석(sensitivity analysis)을 수행할 수 있습니다.

```python
# SciPy linprog 결과에서 쌍대 변수(shadow price) 확인
result = linprog(c, A_ub=A_ub, b_ub=b_ub, bounds=bounds, method='highs')
print("자원별 그림자 가격 (쌍대 변수):")
print(result.ineqlin.marginals)  # 자원 한 단위 증가 시 이익 변화량
```

### 3. 정수 프로그래밍 (Integer Programming)

변수가 정수여야 하면 **혼합 정수 프로그래밍(MIP)**으로 확장합니다. Branch & Bound, Cutting Plane 등의 기법을 사용합니다. LP 완화(relaxation)는 MIP의 하한을 제공합니다.

### 4. 대규모 문제에서의 수치 안정성

- 제약 행렬을 미리 스케일링(scaling)하면 수치 안정성이 향상됩니다.
- Revised Simplex Method는 전체 타블로 대신 기저 역행렬만 관리해 희소 행렬 구조를 활용합니다.
- LU 분해를 이용한 수치 안정적인 기저 역행렬 갱신이 실무 솔버의 핵심입니다.

선형 프로그래밍은 이론과 실용성을 겸비한 강력한 최적화 도구입니다. 심플렉스 알고리즘의 우아한 기하학적 직관과 현대 솔버의 엔지니어링을 이해하면, 현실 세계의 복잡한 의사결정 문제를 체계적으로 해결할 수 있습니다.

## 참고 자료
- [Introduction to the Simplex Algorithm — Baeldung on CS](https://www.baeldung.com/cs/simplex-algorithm-linear-programming)
- [Simplex Algorithm Tabular Method — GeeksforGeeks](https://www.geeksforgeeks.org/python/simplex-algorithm-tabular-method/)
- [Linear Programming with the Simplex Method — Gurobi Resources](https://www.gurobi.com/resources/ch5-linear-programming-simplex-method/)
- [Simplex Method — Medium](https://medium.com/@muditbits/simplex-method-for-linear-programming-1f88fc981f50)
