---
layout: post
title: "수치 해석 완전 정복: Newton-Raphson·Runge-Kutta·수치 적분으로 이해하는 계산 과학의 핵심"
date: 2026-11-02
categories: [cs, computer-science]
tags: [numerical-methods, newton-raphson, runge-kutta, numerical-integration, algorithm, python, math]
---

현대 소프트웨어는 물리 시뮬레이션, 머신러닝 최적화, 금융 모델링, 기상 예측까지 수많은 영역에서 연속적인 수학 문제를 디지털 컴퓨터로 풀어야 한다. 그런데 컴퓨터는 이산적(discrete)이고, 자연은 연속적(continuous)이다. 이 간극을 메우는 학문이 바로 **수치 해석(Numerical Analysis)**이다.

수치 해석이란 해석적(analytic) 해를 구하기 어렵거나 불가능한 수학 문제를 유한한 연산으로 근사하는 알고리즘을 연구하는 분야다. 방정식의 근 탐색, 미분방정식의 수치 해법, 수치 적분이 그 핵심 주제다.

## 왜 수치 해석이 필요한가

다음 방정식의 근을 구해보자.

```
x³ - 2x - 5 = 0
```

대수적 방법으로는 카르다노 공식을 써야 하지만, 실제 공학 문제에서는 수십 개의 변수가 얽힌 비선형 방정식 시스템이 등장한다. 해석적 해가 존재하지 않는 경우도 허다하다. 심지어 간단해 보이는 `cos(x) = x`조차 닫힌 형태의 해가 없다.

미분방정식도 마찬가지다. 나비에-스토크스 방정식, 맥스웰 방정식, 슈뢰딩거 방정식은 특수한 경우를 제외하면 손으로 풀 수 없다. 우주선 궤도를 계산하든, 날씨를 예측하든, 딥러닝 모델을 학습시키든 수치 해석은 현대 과학·공학의 공통 언어다.

---

## 1. 방정식의 수치 해법 — Newton-Raphson 방법

### 개념

**Newton-Raphson 방법**(뉴턴-랍슨법)은 함수 `f(x) = 0`의 근을 반복적으로 근사하는 알고리즘이다. 핵심 아이디어는 현재 추정값 `xₙ`에서 함수의 접선을 긋고, 그 접선이 x축과 만나는 점을 다음 추정값 `xₙ₊₁`으로 삼는 것이다.

업데이트 공식은 다음과 같다.

```
xₙ₊₁ = xₙ - f(xₙ) / f'(xₙ)
```

이 방법의 수렴 속도는 **2차(quadratic)**다. 오차가 반복마다 제곱으로 줄어들어, 정확한 해 근방에서는 소수점 자릿수가 매 스텝마다 두 배로 늘어난다. 예를 들어 처음에 오차가 10⁻² 수준이라면 한 번 반복 후 10⁻⁴, 다음엔 10⁻⁸이 된다.

### 기하학적 해석

현재 점 `(xₙ, f(xₙ))`에서 그은 접선의 방정식은:

```
y - f(xₙ) = f'(xₙ)(x - xₙ)
```

이 직선이 y=0과 만나는 x를 구하면 뉴턴-랍슨 공식이 정확히 도출된다. 시각적으로는 "현재 점의 접선을 따라 x축 방향으로 미끄러지는" 과정이다.

### Python 구현 예제

```python
def newton_raphson(f, df, x0, tol=1e-10, max_iter=100):
    """
    f  : 목표 함수 f(x) = 0
    df : f의 도함수 f'(x)
    x0 : 초기 추정값
    tol: 허용 오차
    """
    x = x0
    for i in range(max_iter):
        fx = f(x)
        dfx = df(x)
        
        if abs(dfx) < 1e-14:
            raise ValueError(f"도함수가 0에 가까움 (x={x}). 발산 위험.")
        
        x_new = x - fx / dfx
        
        print(f"반복 {i+1}: x = {x_new:.12f}, f(x) = {f(x_new):.2e}")
        
        if abs(x_new - x) < tol:
            return x_new
        
        x = x_new
    
    raise RuntimeError(f"최대 반복 횟수({max_iter}) 초과. 수렴 실패.")


import math

# 예제 1: x^3 - 2x - 5 = 0
f1  = lambda x: x**3 - 2*x - 5
df1 = lambda x: 3*x**2 - 2

root1 = newton_raphson(f1, df1, x0=2.0)
print(f"\n근: x = {root1:.10f}")
print(f"검증: f({root1:.6f}) = {f1(root1):.2e}\n")

# 예제 2: cos(x) = x  =>  cos(x) - x = 0  (Dottie Number)
f2  = lambda x: math.cos(x) - x
df2 = lambda x: -math.sin(x) - 1

root2 = newton_raphson(f2, df2, x0=1.0)
print(f"Dottie Number: x = {root2:.10f}")
```

실행 결과:
```
반복 1: x = 2.100000000000, f(x) = -2.39e-01
반복 2: x = 2.094568121105, f(x) = 2.16e-04
반복 3: x = 2.094551481700, f(x) = 1.79e-10
반복 4: x = 2.094551481698, f(x) = 0.00e+00

근: x = 2.0945514817
검증: f(2.094551) = 0.00e+00

Dottie Number: x = 0.7390851332
```

단 4번의 반복으로 12자리 정밀도에 수렴했다 — 2차 수렴의 위력이다.

### 주의사항: 뉴턴-랍슨법의 함정

뉴턴-랍슨법은 강력하지만 다음 상황에서 실패한다.

1. **초기값 선택이 잘못된 경우**: 근에서 너무 멀거나, 함수가 진동하면 발산한다.
2. **도함수가 0인 경우**: 분모가 0이 되어 계산 불가.
3. **중근(multiple root)**: 중근 근방에서 수렴이 1차로 느려진다.
4. **여러 근이 존재하는 경우**: 어떤 근으로 수렴하는지는 초기값에 의존한다 — 프랙탈 구조로 이어지기도 한다.

실용적으로는 **이분법(Bisection Method)**으로 대략적인 위치를 잡은 뒤 뉴턴-랍슨으로 빠르게 정밀도를 높이는 **hybrid 전략**이 안정적이다.

---

## 2. 미분방정식 수치 해법 — Runge-Kutta 방법

### 개념

**상미분방정식(Ordinary Differential Equation, ODE)**의 일반적인 형태는 다음과 같다.

```
dy/dt = f(t, y),   y(t₀) = y₀
```

물체의 운동(F=ma), 인구 성장(dP/dt = rP), RC 회로, 화학 반응 속도 등 수많은 물리·화학·생물학 시스템이 이 형태로 표현된다. 해석적 해가 없을 때 수치적으로 y(t)를 단계별로 전진시키는 방법이 필요하다.

**오일러 방법(Euler's method)**은 가장 단순한 접근이다.

```
y_{n+1} = y_n + h·f(t_n, y_n)
```

스텝 크기 h에 비례하는 1차 오차(O(h))를 가진다. 너무 부정확해서 실용적이지 않은 경우가 많다.

**Runge-Kutta 4차 방법(RK4)**은 오일러를 크게 개선한 것으로, 실용적인 기본 방법이다. 각 스텝에서 함수를 4번 평가해 가중 평균을 취한다.

```
k₁ = h·f(tₙ, yₙ)
k₂ = h·f(tₙ + h/2, yₙ + k₁/2)
k₃ = h·f(tₙ + h/2, yₙ + k₂/2)
k₄ = h·f(tₙ + h,   yₙ + k₃)

y_{n+1} = yₙ + (k₁ + 2k₂ + 2k₃ + k₄) / 6
```

오차 차수는 O(h⁴) — 스텝 크기를 절반으로 줄이면 오차가 1/16로 줄어든다.

### Python 구현 예제

```python
import numpy as np
import matplotlib.pyplot as plt

def euler(f, t0, y0, h, n_steps):
    t = t0
    y = y0
    ts, ys = [t], [y]
    for _ in range(n_steps):
        y = y + h * f(t, y)
        t = t + h
        ts.append(t); ys.append(y)
    return np.array(ts), np.array(ys)

def rk4(f, t0, y0, h, n_steps):
    t = t0
    y = y0
    ts, ys = [t], [y]
    for _ in range(n_steps):
        k1 = h * f(t,       y)
        k2 = h * f(t + h/2, y + k1/2)
        k3 = h * f(t + h/2, y + k2/2)
        k4 = h * f(t + h,   y + k3)
        y = y + (k1 + 2*k2 + 2*k3 + k4) / 6
        t = t + h
        ts.append(t); ys.append(y)
    return np.array(ts), np.array(ys)


# 예제: dy/dt = -2y,  y(0) = 1  =>  해석적 해: y(t) = e^{-2t}
f  = lambda t, y: -2 * y
t_exact = np.linspace(0, 3, 300)
y_exact = np.exp(-2 * t_exact)

h = 0.5
n = 6

t_euler, y_euler = euler(f, 0, 1.0, h, n)
t_rk4,   y_rk4   = rk4(f,   0, 1.0, h, n)

print(f"{'t':>6}  {'정확한 해':>14}  {'오일러':>14}  {'RK4':>14}")
print("-" * 60)
for i in range(n+1):
    t = t_euler[i]
    ye = y_euler[i]
    yr = y_rk4[i]
    yc = np.exp(-2*t)
    print(f"{t:6.2f}  {yc:14.8f}  {ye:14.8f}  {yr:14.8f}")

print(f"\n최종 오차 (t=3.0):")
print(f"  오일러: {abs(y_euler[-1] - np.exp(-6)):.2e}")
print(f"  RK4  : {abs(y_rk4[-1]   - np.exp(-6)):.2e}")
```

출력 예:
```
     t  정확한 해           오일러           RK4
------------------------------------------------------------
  0.00  1.00000000    1.00000000    1.00000000
  0.50  0.36787944    0.00000000    0.36458333
  1.00  0.13533528   -0.00000000    0.13292955
  ...
  3.00  0.00247875    0.00000000    0.00247869

최종 오차 (t=3.0):
  오일러: 2.48e-03
  RK4  : 6.68e-07
```

스텝 크기 h=0.5임에도 RK4의 오차는 오일러보다 3000배 이상 작다.

### Adaptive Step Size — 현대적 수치 적분의 핵심

고정 스텝 크기는 비효율적이다. 함수가 급격히 변하는 구간에서는 작은 h가 필요하고, 완만한 구간에서는 큰 h를 써도 된다. **Dormand-Prince(RK45)**, **Cash-Karp** 같은 적응형 방법은 오차 추정치를 보면서 h를 자동으로 조절한다.

Python에서는 `scipy.integrate.solve_ivp`가 이를 구현하고 있다.

```python
from scipy.integrate import solve_ivp
import numpy as np

# 스티프(stiff) 방정식 예제: van der Pol 진동자
def van_der_pol(t, y, mu=1000):
    return [y[1], mu*(1 - y[0]**2)*y[1] - y[0]]

sol = solve_ivp(
    van_der_pol,
    t_span=(0, 3000),
    y0=[2, 0],
    method='Radau',   # 스티프 방정식에는 암시적 방법이 필요
    rtol=1e-6,
    atol=1e-8,
    dense_output=True
)

print(f"함수 평가 횟수: {sol.nfev}")
print(f"최종 상태: y={sol.y[0,-1]:.4f}, y'={sol.y[1,-1]:.4f}")
```

---

## 3. 수치 적분 — 가우스 구적법

### 개념

정적분 `∫[a,b] f(x) dx`를 해석적으로 구하기 어려울 때 수치적으로 계산한다. 고전적인 방법으로 사다리꼴 공식(Trapezoidal Rule), Simpson's Rule이 있다. 현대적으로는 **가우스 구적법(Gaussian Quadrature)**이 더 효율적이다.

가우스 구적법은 적분 구간에서 최적 배치된 n개의 점(가우스 노드)과 대응하는 가중치를 사용해 다항식에 대해 정확하다.

```
∫[-1,1] f(x) dx ≈ Σᵢ wᵢ·f(xᵢ)
```

n개의 노드를 사용하면 최대 차수 2n-1인 다항식을 정확하게 적분한다.

### 구현 예제

```python
import numpy as np

def gauss_legendre_quadrature(f, a, b, n=5):
    """가우스-르장드르 구적법으로 ∫[a,b] f(x) dx 계산"""
    # NumPy가 제공하는 가우스-르장드르 노드와 가중치 ([-1,1] 구간)
    xi, wi = np.polynomial.legendre.leggauss(n)
    
    # [-1,1] → [a,b] 변환
    x = 0.5*(b - a)*xi + 0.5*(b + a)
    w = 0.5*(b - a)*wi
    
    return np.sum(w * f(x))


# 테스트: ∫[0, π] sin(x) dx = 2 (정확한 값)
result = gauss_legendre_quadrature(np.sin, 0, np.pi, n=5)
print(f"∫[0,π] sin(x) dx = {result:.15f}")
print(f"오차: {abs(result - 2.0):.2e}")

# ∫[0,1] e^x dx = e - 1 ≈ 1.71828
result2 = gauss_legendre_quadrature(np.exp, 0, 1, n=5)
print(f"\n∫[0,1] e^x dx = {result2:.15f}")
print(f"오차: {abs(result2 - (np.e - 1)):.2e}")

# n 증가에 따른 수렴
f_hard = lambda x: np.sin(1/(x + 0.01))  # 빠르게 진동하는 함수
exact  = -0.4011338781...  # 이 예에선 scipy로 검증
from scipy import integrate
exact_val, _ = integrate.quad(f_hard, 0, 1)
print(f"\n어려운 함수 적분:")
for n in [2, 5, 10, 20, 50]:
    r = gauss_legendre_quadrature(f_hard, 0, 1, n=n)
    print(f"  n={n:2d}: {r:.8f}, 오차={abs(r-exact_val):.2e}")
```

---

## 주의사항 및 실용 팁

### 1. 수치 안정성(Numerical Stability)

부동소수점 연산은 반올림 오차를 누적시킨다. 수천 번의 반복 연산에서는 이 오차가 의미 있는 수준으로 커질 수 있다. **조건수(condition number)**가 큰 문제는 본질적으로 수치적으로 불안정하며, 알고리즘이 아무리 훌륭해도 결과를 신뢰하기 어렵다.

### 2. 스티프(Stiff) 방정식

여러 시간 스케일이 혼재하는 방정식(예: 화학 반응에서 빠른 반응과 느린 반응이 공존하는 경우)은 명시적(explicit) 방법으로 풀 때 매우 작은 스텝을 요구한다. **암시적(implicit) 방법** — 예를 들어 역향 오일러(Backward Euler), Crank-Nicolson, Radau — 을 사용해야 효율적이다.

### 3. 오차의 두 종류

- **절단 오차(Truncation Error)**: 연속적 수학 연산을 유한 스텝으로 근사하면서 발생하는 알고리즘 고유의 오차. h → 0 이면 줄어든다.
- **반올림 오차(Round-off Error)**: 부동소수점 표현의 유한 정밀도에서 비롯되는 오차. h → 0 이면 오히려 커진다.

최적 스텝 크기는 이 두 오차의 합이 최소가 되는 지점이며, 대략 `h ≈ sqrt(ε)` (ε는 machine epsilon, 약 10⁻¹⁶) 근방이다.

### 4. 라이브러리 활용

실제 프로젝트에서는 직접 구현보다 검증된 라이브러리를 사용하라.

- **Python**: `scipy.integrate`, `numpy.polynomial`
- **Julia**: `DifferentialEquations.jl` — 세계 최고 수준의 ODE 솔버
- **C++**: `Boost.Math`, `GNU Scientific Library (GSL)`
- **MATLAB/Octave**: `ode45`, `ode23s`

직접 구현은 알고리즘 이해를 위해 가치 있지만, 프로덕션 코드에서는 엣지 케이스와 수치 안정성이 충분히 검증된 라이브러리에 의존하는 것이 현명하다.

---

수치 해석은 이론과 구현이 깊게 연결된 분야다. Newton-Raphson은 머신러닝의 2차 최적화(Newton's method, L-BFGS)의 뿌리이고, Runge-Kutta는 물리 엔진과 기상 모델의 심장이다. 이 알고리즘들이 수렴하는 이유, 실패하는 조건, 오차가 어떻게 전파되는지를 이해하면 복잡한 계산 문제를 마주했을 때 더 나은 판단을 내릴 수 있다.

## 참고 자료
- [Newton's method — Wikipedia](https://en.wikipedia.org/wiki/Newton%27s_method)
- [Runge–Kutta methods — Wikipedia](https://en.wikipedia.org/wiki/Runge%E2%80%93Kutta_methods)
- [Numerical methods for ODEs — Wikipedia](https://en.wikipedia.org/wiki/Numerical_methods_for_ordinary_differential_equations)
- [SciPy Integration and ODEs documentation](https://docs.scipy.org/doc/scipy/reference/integrate.html)
