---
layout: post
title: "자동 미분(Automatic Differentiation) 완전 정복: 계산 그래프와 역방향 모드 AD로 딥러닝 프레임워크 이해하기"
date: 2026-09-24
categories: [cs, computer-science]
tags: [automatic-differentiation, deep-learning, backpropagation, computation-graph, pytorch, jax, dual-numbers]
---

딥러닝 프레임워크인 PyTorch, TensorFlow, JAX는 어떻게 복잡한 신경망의 기울기(gradient)를 자동으로 계산할까? 그 핵심에는 **자동 미분(Automatic Differentiation, AD)**이 있다. 단순히 수치 미분이나 기호 미분을 쓰는 것이 아니라, 연산 그래프와 연쇄 법칙(chain rule)을 조합해 정확하고 효율적인 미분을 실현하는 기법이다. 이 아티클에서는 자동 미분의 두 가지 모드(전방향/역방향), 이중수(dual number), 계산 그래프, 그리고 실제 구현까지 완전히 파헤친다.

---

## 1. 왜 자동 미분이 필요한가?

미분을 계산하는 방법은 크게 세 가지가 있다.

| 방법 | 설명 | 한계 |
|---|---|---|
| **수치 미분** | `(f(x+h) - f(x)) / h` | 부동소수점 오차, O(n) 비용 |
| **기호 미분** | 수식을 대수적으로 전개 | 표현식 폭발(expression swell), 조건문·루프 처리 불가 |
| **자동 미분** | 연쇄 법칙을 코드 실행과 동시에 적용 | 거의 없음; 사실상 표준 |

신경망은 수백만 개의 매개변수를 가진 합성 함수다. 수치 미분은 매개변수 수만큼 반복 연산이 필요하고, 기호 미분은 거대한 수식을 다룰 수 없다. 자동 미분은 코드를 그대로 실행하면서 동시에 미분값을 정확하게 추적하므로, 두 방법의 한계를 모두 극복한다.

---

## 2. 이중수(Dual Numbers)와 전방향 모드

### 이중수란?

이중수는 `a + bε` 형태의 수로, `ε² = 0`이면서 `ε ≠ 0`이라는 성질을 갖는다. 이 성질을 Taylor 전개에 적용하면:

```
f(a + bε) = f(a) + f'(a) · b · ε + 0
```

즉, 이중수를 함수에 넣으면 함수값 `f(a)`와 도함수 `f'(a) · b`가 동시에 얻어진다.

### 전방향 모드 AD (Forward Mode)

전방향 모드는 입력에서 출력으로 계산하면서 각 중간 변수의 도함수(접선(tangent))를 함께 전파한다.

```python
# 이중수 클래스 직접 구현
class Dual:
    def __init__(self, val, tangent=0.0):
        self.val = val       # 함수값
        self.tangent = tangent  # 도함수값

    def __add__(self, other):
        if isinstance(other, Dual):
            return Dual(self.val + other.val, self.tangent + other.tangent)
        return Dual(self.val + other, self.tangent)

    def __mul__(self, other):
        if isinstance(other, Dual):
            # (fg)' = f'g + fg' (곱의 법칙)
            return Dual(
                self.val * other.val,
                self.tangent * other.val + self.val * other.tangent
            )
        return Dual(self.val * other, self.tangent * other)

    def __radd__(self, other): return self.__add__(other)
    def __rmul__(self, other): return self.__mul__(other)

import math

def sin(d: Dual) -> Dual:
    return Dual(math.sin(d.val), math.cos(d.val) * d.tangent)

def exp(d: Dual) -> Dual:
    e = math.exp(d.val)
    return Dual(e, e * d.tangent)

# f(x) = x² · sin(x) + exp(x) 의 x=1.0에서 미분
# x를 이중수로: val=1.0, tangent=1.0 (dx/dx = 1)
x = Dual(1.0, 1.0)
result = x * x * sin(x) + exp(x)

print(f"f(1.0)  = {result.val:.6f}")     # 함수값
print(f"f'(1.0) = {result.tangent:.6f}") # 도함수값

# 검증: f'(x) = 2x·sin(x) + x²·cos(x) + exp(x)
analytic = 2*1.0*math.sin(1.0) + 1.0**2*math.cos(1.0) + math.exp(1.0)
print(f"해석적 도함수 = {analytic:.6f}")
```

전방향 모드는 입력이 적고 출력이 많은 경우(예: Jacobian 벡터 곱)에 효율적이다. 입력 변수가 `n`개라면 `n`번의 순전파가 필요하다.

---

## 3. 계산 그래프와 역방향 모드 AD

### 역방향 모드 (Backward Mode / Reverse Mode)

역방향 모드는 출력에서 입력 방향으로 기울기를 역전파한다. 신경망 학습에서의 **역전파(backpropagation)**가 바로 역방향 AD다. 입력이 많고 출력이 스칼라 하나인 상황(손실 함수 등)에서 압도적으로 효율적이다.

단 **한 번의 역방향 패스**로 모든 입력에 대한 편미분을 구할 수 있다.

### 계산 그래프 (Computation Graph)

계산 그래프는 함수 연산을 방향성 비순환 그래프(DAG)로 표현한다. 노드는 중간 변수이고, 엣지는 연산 관계를 나타낸다. 역방향 패스에서는 이 그래프를 역방향으로 순회하며 **보(adjoint)** 값 `ȳ = ∂output/∂node`를 누적한다.

```python
# 계산 그래프 기반 역방향 AD 미니 구현
from typing import Callable, List, Tuple
import math

class Node:
    def __init__(self, val: float):
        self.val = val
        self.grad = 0.0
        # (부모 노드, 로컬 기울기) 리스트
        self._parents: List[Tuple['Node', float]] = []

    def backward(self):
        """위상 정렬 후 역방향 전파"""
        topo = []
        visited = set()

        def build(node):
            if id(node) not in visited:
                visited.add(id(node))
                for parent, _ in node._parents:
                    build(parent)
                topo.append(node)

        build(self)
        self.grad = 1.0
        for node in reversed(topo):
            for parent, local_grad in node._parents:
                parent.grad += node.grad * local_grad

def add(a: Node, b: Node) -> Node:
    out = Node(a.val + b.val)
    out._parents = [(a, 1.0), (b, 1.0)]  # d(a+b)/da = 1, d(a+b)/db = 1
    return out

def mul(a: Node, b: Node) -> Node:
    out = Node(a.val * b.val)
    out._parents = [(a, b.val), (b, a.val)]  # d(ab)/da = b, d(ab)/db = a
    return out

def sin_node(a: Node) -> Node:
    out = Node(math.sin(a.val))
    out._parents = [(a, math.cos(a.val))]   # d(sin x)/dx = cos x
    return out

def exp_node(a: Node) -> Node:
    e = math.exp(a.val)
    out = Node(e)
    out._parents = [(a, e)]   # d(exp x)/dx = exp x
    return out

# f(x, y) = x·sin(y) + exp(x·y) 에서 편미분 계산
x = Node(2.0)
y = Node(3.0)

xy  = mul(x, y)          # x*y
out = add(mul(x, sin_node(y)), exp_node(xy))

out.backward()

print(f"f(2, 3)          = {out.val:.6f}")
print(f"∂f/∂x at (2,3)   = {x.grad:.6f}")
print(f"∂f/∂y at (2,3)   = {y.grad:.6f}")

# 검증
# ∂f/∂x = sin(y) + y·exp(x·y)
df_dx = math.sin(3.0) + 3.0 * math.exp(6.0)
# ∂f/∂y = x·cos(y) + x·exp(x·y)
df_dy = 2.0 * math.cos(3.0) + 2.0 * math.exp(6.0)
print(f"해석적 ∂f/∂x     = {df_dx:.6f}")
print(f"해석적 ∂f/∂y     = {df_dy:.6f}")
```

---

## 4. 전방향 vs 역방향 모드 복잡도 비교

함수 `f: ℝⁿ → ℝᵐ`에 대해:

| 모드 | 비용 | 적합한 상황 |
|---|---|---|
| **전방향** | O(n) 순전파 | n ≪ m (입력 적음, 출력 많음) |
| **역방향** | O(m) 역전파 | n ≫ m (입력 많음, 출력 적음) |

딥러닝 손실 함수는 `f: ℝ^(수백만) → ℝ¹`이므로 역방향 모드가 압도적으로 유리하다. 단 한 번의 역방향 패스로 수백만 개의 매개변수에 대한 기울기를 모두 얻는다.

---

## 5. JAX의 함수형 AD와 JIT

Google JAX는 자동 미분을 함수 변환(function transformation)으로 제공한다.

```python
# JAX 스타일 AD 개념 (실제 JAX API 참고)
# pip install jax jaxlib

import jax
import jax.numpy as jnp

def f(x):
    return jnp.sin(x) ** 2 + jnp.exp(-x)

# 1차 도함수
df = jax.grad(f)
# 2차 도함수 (grad of grad)
ddf = jax.grad(jax.grad(f))
# Jacobian 벡터 곱 (전방향 모드)
jvp_fn = jax.jvp   # (f, primals, tangents) → (primals_out, tangents_out)
# 벡터 Jacobian 곱 (역방향 모드)
vjp_fn = jax.vjp   # (f, *primals) → (out, vjp_func)

x0 = jnp.array(1.0)
print(f"f(1.0)   = {f(x0):.6f}")
print(f"f'(1.0)  = {df(x0):.6f}")
print(f"f''(1.0) = {ddf(x0):.6f}")

# vmap: 배치 자동화 (벡터화된 자동 미분)
batch_grad = jax.vmap(jax.grad(f))
xs = jnp.linspace(0, 2, 5)
print("배치 기울기:", batch_grad(xs))
```

JAX의 핵심 특징:
- `jax.grad`: 역방향 모드 AD
- `jax.jvp`: 전방향 모드 AD (Jacobian-vector product)
- `jax.vjp`: 역방향 모드 AD (vector-Jacobian product)
- `jax.jit`: XLA 컴파일로 GPU/TPU 가속
- `jax.vmap`: 배치 자동화

---

## 6. 고차 도함수와 고급 활용

역방향 AD를 중첩하면 고차 도함수를 계산할 수 있다.

```python
# 고차 도함수: 미니 구현으로 2차 도함수 계산
# Hessian H[i,j] = ∂²f/∂xᵢ∂xⱼ

def compute_hessian_diag(f, x_vals):
    """대각 헤시안 ∂²f/∂xᵢ² 계산 (근사적 구현)"""
    n = len(x_vals)
    hess_diag = []

    for i in range(n):
        # i번째 입력에 대한 2차 도함수: (f(x+h) - 2f(x) + f(x-h)) / h²
        h = 1e-5
        x_fwd = x_vals.copy(); x_fwd[i] += h
        x_bwd = x_vals.copy(); x_bwd[i] -= h

        hii = (f(x_fwd) - 2*f(x_vals) + f(x_bwd)) / (h * h)
        hess_diag.append(hii)

    return hess_diag

def rosenbrock(x):
    """로젠브록 함수: 최적화 벤치마크"""
    return sum(100 * (x[i+1] - x[i]**2)**2 + (1 - x[i])**2
               for i in range(len(x)-1))

x = [1.5, 1.5]
hd = compute_hessian_diag(rosenbrock, x)
print(f"Rosenbrock at {x}: f = {rosenbrock(x):.4f}")
print(f"대각 헤시안: {[f'{v:.2f}' for v in hd]}")
```

---

## 7. 주의사항과 팁

### 메모리 트레이드오프
역방향 모드 AD는 역전파 시 중간 변수가 필요하므로 **모든 중간 계산을 메모리에 저장**한다. 계산 그래프가 깊어질수록 메모리 사용량이 증가한다. 이를 해결하기 위해:
- **그래디언트 체크포인팅(gradient checkpointing)**: 중간값 일부를 버리고 필요할 때 재계산
- **반복적 역방향(iterative reverse-mode)**: 메모리 O(√n) 트레이드오프

### 제어 흐름과 동적 그래프
PyTorch의 **define-by-run** 방식은 조건문(`if`)과 반복문(`for`)을 포함한 일반 Python 코드에서 계산 그래프를 동적으로 생성한다. 즉, 실행 경로에 따라 그래프가 달라질 수 있어 재귀 신경망, 가변 길이 입력 처리가 자연스럽다.

### 수치 안정성
- Sigmoid와 Softmax는 수치적으로 불안정할 수 있으므로 `log-sum-exp` 트릭을 활용한다.
- 기울기 소실/폭발 문제는 Batch Normalization, LayerNorm, 잔차 연결(residual connection)로 완화한다.

### 분리(detach)
`tensor.detach()`로 그래프에서 텐서를 분리하면 해당 연산이 역전파 그래프에 포함되지 않는다. Fine-tuning 시 특정 레이어를 고정(freeze)할 때 활용한다.

---

## 참고 자료

- [Automatic Differentiation in Machine Learning: a Survey (arXiv)](https://arxiv.org/pdf/1502.05767)
- [Forward Mode Automatic Differentiation & Dual Numbers (Medium/TDS)](https://medium.com/data-science/forward-mode-automatic-differentiation-dual-numbers-8f47351064bf)
- [Deep Learning: Computational Aspects (arXiv)](https://arxiv.org/pdf/1808.08618)
- [MATLAB Deep Learning with Automatic Differentiation (MathWorks)](https://www.mathworks.com/help/deeplearning/ug/deep-learning-with-automatic-differentiation-in-matlab.html)
