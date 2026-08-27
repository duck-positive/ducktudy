---
layout: post
title: "역전파 알고리즘 심화 — 딥러닝 경사 하강의 수학적 기반과 NumPy 구현"
date: 2026-08-27
categories: [cs, computer-science]
tags: [backpropagation, neural-network, deep-learning, gradient-descent, chain-rule, autograd, vanishing-gradient]
---

딥러닝의 눈부신 성과 뒤에는 1986년 Rumelhart, Hinton, Williams가 대중화한 **역전파(Backpropagation)** 알고리즘이 있다. 오늘날 PyTorch의 `autograd`, TensorFlow의 `GradientTape`도 모두 이 원리 위에서 동작한다. 역전파를 코드로 직접 구현해 보면, 딥러닝 프레임워크의 내부 동작 방식을 훨씬 깊이 이해할 수 있다.

## 개념 설명: 역전파의 수학적 기반

### 문제 정의: 신경망 학습이란?

신경망은 입력 `x`를 받아 예측값 `ŷ`를 출력하는 파라미터화된 함수 `f(x; W)`이다. 학습이란 실제값 `y`와 예측값 `ŷ` 사이의 오차를 정의하는 **손실 함수(Loss Function)** `L(y, ŷ)`를 최소화하는 가중치 `W`를 찾는 과정이다.

가중치를 업데이트하려면 각 가중치에 대한 손실 함수의 편미분 `∂L/∂W`가 필요하다. 이 그래디언트를 효율적으로 계산하는 알고리즘이 역전파다.

### 연쇄 법칙(Chain Rule)

핵심은 미적분의 **연쇄 법칙**이다. 합성 함수 `L = f(g(h(x)))`의 미분은:

```
dL/dx = dL/df · df/dg · dg/dh · dh/dx
```

신경망은 여러 레이어로 구성된 합성 함수이므로, 출력에서 시작하여 입력 방향으로 그래디언트를 역방향으로 전파할 수 있다.

### 순전파(Forward Pass)와 역전파(Backward Pass)

2-레이어 신경망을 예로 들면:

```
# 순전파
z1 = W1 @ x + b1       # 선형 변환
a1 = ReLU(z1)          # 활성화 함수
z2 = W2 @ a1 + b2
ŷ = Softmax(z2)         # 출력
L = CrossEntropy(y, ŷ) # 손실

# 역전파 (연쇄 법칙 적용)
dL/dz2 = ŷ - y              # Softmax + CrossEntropy 합산 미분
dL/dW2 = dL/dz2 @ a1.T      # 가중치 그래디언트
dL/db2 = sum(dL/dz2)
dL/da1 = W2.T @ dL/dz2      # 이전 레이어로 전파
dL/dz1 = dL/da1 * ReLU'(z1) # ReLU 미분: z1>0이면 1, 아니면 0
dL/dW1 = dL/dz1 @ x.T
dL/db1 = sum(dL/dz1)
```

## 왜 필요한가?

그래디언트를 계산하는 다른 방법들과 비교해보자:

- **수치 미분**: `(f(x+ε) - f(x)) / ε` 으로 근사하지만, 파라미터가 수백만 개인 모델에서는 계산 불가능할 만큼 느리다 (각 파라미터마다 별도의 순전파 필요).
- **기호 미분**: 수식을 기호적으로 미분하면 표현식이 지수적으로 팽창한다.
- **역전파(자동 미분의 역방향 모드)**: 순전파와 동일한 계산 복잡도로 모든 파라미터의 그래디언트를 한 번에 계산한다. `O(forward_pass)` 복잡도.

이것이 역전파가 딥러닝 학습의 핵심 알고리즘으로 자리잡은 이유다.

## 실제 구현 예제

### 예제 1: NumPy로 2-레이어 신경망 역전파 구현

```python
import numpy as np

def relu(z):
    return np.maximum(0, z)

def relu_deriv(z):
    return (z > 0).astype(float)

def softmax(z):
    e = np.exp(z - np.max(z, axis=0, keepdims=True))
    return e / e.sum(axis=0, keepdims=True)

def cross_entropy_loss(y_hat, y):
    """y: one-hot, y_hat: softmax 출력"""
    m = y.shape[1]
    return -np.sum(y * np.log(y_hat + 1e-8)) / m

class TwoLayerNet:
    def __init__(self, input_dim, hidden_dim, output_dim, lr=0.01):
        # He 초기화 (ReLU에 적합)
        self.W1 = np.random.randn(hidden_dim, input_dim) * np.sqrt(2 / input_dim)
        self.b1 = np.zeros((hidden_dim, 1))
        self.W2 = np.random.randn(output_dim, hidden_dim) * np.sqrt(2 / hidden_dim)
        self.b2 = np.zeros((output_dim, 1))
        self.lr = lr
        self._cache = {}  # 역전파에 필요한 중간값 저장

    def forward(self, X):
        """순전파: 중간값을 캐시에 저장"""
        z1 = self.W1 @ X + self.b1    # (hidden, m)
        a1 = relu(z1)                  # (hidden, m)
        z2 = self.W2 @ a1 + self.b2   # (output, m)
        a2 = softmax(z2)               # (output, m)

        self._cache = {"X": X, "z1": z1, "a1": a1, "z2": z2, "a2": a2}
        return a2

    def backward(self, Y):
        """역전파: 연쇄 법칙으로 각 파라미터의 그래디언트 계산"""
        X, z1, a1, z2, a2 = (self._cache[k] for k in ["X", "z1", "a1", "z2", "a2"])
        m = X.shape[1]

        # 출력층 그래디언트 (Softmax + CrossEntropy 합산 미분)
        dz2 = a2 - Y                         # (output, m)
        dW2 = dz2 @ a1.T / m                 # (output, hidden)
        db2 = dz2.sum(axis=1, keepdims=True) / m

        # 은닉층으로 그래디언트 전파
        da1 = self.W2.T @ dz2                # (hidden, m)
        dz1 = da1 * relu_deriv(z1)           # ReLU 역방향
        dW1 = dz1 @ X.T / m                  # (hidden, input)
        db1 = dz1.sum(axis=1, keepdims=True) / m

        # SGD 업데이트
        self.W2 -= self.lr * dW2
        self.b2 -= self.lr * db2
        self.W1 -= self.lr * dW1
        self.b1 -= self.lr * db1

        return {"dW1": dW1, "db1": db1, "dW2": dW2, "db2": db2}

    def train(self, X, Y, epochs=1000):
        for epoch in range(epochs):
            y_hat = self.forward(X)
            loss = cross_entropy_loss(y_hat, Y)
            self.backward(Y)
            if epoch % 100 == 0:
                preds = np.argmax(y_hat, axis=0)
                labels = np.argmax(Y, axis=0)
                acc = (preds == labels).mean()
                print(f"Epoch {epoch:4d} | Loss: {loss:.4f} | Acc: {acc:.3f}")

# XOR 문제 학습
if __name__ == "__main__":
    np.random.seed(42)
    X = np.array([[0, 0, 1, 1],
                  [0, 1, 0, 1]])  # (2, 4)
    Y_raw = np.array([0, 1, 1, 0])  # XOR 레이블
    Y = np.eye(2)[:, Y_raw]        # One-hot (2, 4)

    net = TwoLayerNet(input_dim=2, hidden_dim=8, output_dim=2, lr=0.5)
    net.train(X, Y, epochs=1000)
    print("\n최종 예측:", np.argmax(net.forward(X), axis=0))
    print("정답:     ", Y_raw)
```

실행하면 약 300 에폭 근방에서 XOR 문제가 완벽히 해결되는 것을 확인할 수 있다.

### 예제 2: 계산 그래프와 자동 미분(Autograd) 미니 구현

PyTorch의 `autograd`가 내부적으로 어떻게 동작하는지 이해하기 위한 간단한 계산 그래프 구현이다.

```python
import numpy as np

class Tensor:
    """자동 미분을 지원하는 미니 텐서 클래스"""
    def __init__(self, data, _children=(), _op=""):
        self.data = np.array(data, dtype=float)
        self.grad = 0.0
        self._backward = lambda: None
        self._prev = set(_children)
        self._op = _op

    def __add__(self, other):
        other = other if isinstance(other, Tensor) else Tensor(other)
        out = Tensor(self.data + other.data, (self, other), '+')

        def _backward():
            # dL/dself = dL/dout · dout/dself = dL/dout · 1
            self.grad += out.grad
            other.grad += out.grad
        out._backward = _backward
        return out

    def __mul__(self, other):
        other = other if isinstance(other, Tensor) else Tensor(other)
        out = Tensor(self.data * other.data, (self, other), '*')

        def _backward():
            self.grad += other.data * out.grad
            other.grad += self.data * out.grad
        out._backward = _backward
        return out

    def relu(self):
        out = Tensor(np.maximum(0, self.data), (self,), 'ReLU')

        def _backward():
            self.grad += (out.data > 0) * out.grad
        out._backward = _backward
        return out

    def pow(self, exponent):
        out = Tensor(self.data ** exponent, (self,), f'**{exponent}')

        def _backward():
            self.grad += exponent * (self.data ** (exponent - 1)) * out.grad
        out._backward = _backward
        return out

    def backward(self):
        """위상 정렬(Topological Sort) 후 역방향으로 그래디언트 전파"""
        topo, visited = [], set()

        def build_topo(v):
            if v not in visited:
                visited.add(v)
                for child in v._prev:
                    build_topo(child)
                topo.append(v)

        build_topo(self)
        self.grad = 1.0  # dL/dL = 1
        for node in reversed(topo):
            node._backward()

    def __repr__(self):
        return f"Tensor(data={self.data:.4f}, grad={self.grad:.4f})"

# 간단한 2차 함수 최소화: L = (x - 3)^2
x = Tensor(0.0)
for step in range(20):
    # 순전파: L = (x - 3)^2 = x^2 - 6x + 9
    diff = x + Tensor(-3.0)   # x - 3
    L = diff.pow(2)            # (x-3)^2

    # 역전파
    x.grad = 0.0
    L.backward()

    # SGD 업데이트
    lr = 0.1
    x.data -= lr * x.grad

    if step % 4 == 0:
        print(f"Step {step:2d}: x={x.data:.4f}, L={L.data:.4f}, grad={x.grad:.4f}")

print(f"\n최적 x = {x.data:.4f} (기대값: 3.0)")

# 신경망 계층 예시
print("\n--- 신경망 순전파/역전파 예시 ---")
w = Tensor(0.5)   # 가중치
b = Tensor(0.0)   # 편향
x_in = Tensor(2.0)  # 입력

# 순전파: out = relu(w * x + b)
z = w * x_in + b
out = z.relu()
loss = (out + Tensor(-1.0)).pow(2)  # MSE (목표: 1.0)

loss.backward()
print(f"w.grad = {w.grad:.4f}")
print(f"b.grad = {b.grad:.4f}")
print(f"∂L/∂w = 2*(relu(wx+b)-1) * relu'(wx+b) * x = {w.grad:.4f}")
```

이 미니 `autograd` 구현은 PyTorch가 내부적으로 하는 일과 동일하다. 각 연산이 `_backward` 클로저를 통해 역전파 함수를 등록하고, `backward()` 호출 시 위상 정렬된 순서로 그래디언트를 전파한다.

## 주의사항과 팁

### 1. 기울기 소실(Vanishing Gradient) 문제

시그모이드 함수의 도함수는 최대 0.25다. 레이어가 깊어질수록 `0.25^n`으로 그래디언트가 0에 수렴하여 초기 레이어가 학습되지 않는다. 이를 해결하기 위해:
- **ReLU 계열 활성화 함수** 사용: `max(0, x)`의 도함수는 양의 영역에서 1
- **배치 정규화(Batch Normalization)**: 각 레이어의 입력 분포를 안정화
- **잔차 연결(Residual Connection)**: 그래디언트가 숏컷 경로로 직접 흐름

### 2. 기울기 폭발(Exploding Gradient) 문제

RNN 등에서 그래디언트가 지수적으로 커질 수 있다. **그래디언트 클리핑(Gradient Clipping)** 으로 그래디언트의 노름이 임계값을 넘으면 스케일을 줄인다.

```python
def clip_gradients(params, max_norm=1.0):
    total_norm = np.sqrt(sum(np.sum(p.grad**2) for p in params))
    if total_norm > max_norm:
        clip_coef = max_norm / (total_norm + 1e-8)
        for p in params:
            p.grad *= clip_coef
```

### 3. 수치 안정성: log-sum-exp 트릭

Softmax 계산 시 `exp(z)`가 오버플로우할 수 있다. 안전한 계산을 위해 항상 최댓값을 빼준다.

```python
# 불안정한 버전
softmax_unsafe = np.exp(z) / np.exp(z).sum()

# 안정적인 버전 (log-sum-exp 트릭)
z_stable = z - z.max()
softmax_stable = np.exp(z_stable) / np.exp(z_stable).sum()
```

### 4. 그래디언트 검증(Gradient Check)

역전파 구현의 정확성을 검증할 때 수치 미분과 비교한다.

```python
def gradient_check(net, X, Y, eps=1e-7):
    """수치 미분으로 역전파 구현 검증"""
    for param_name in ["W1", "b1", "W2", "b2"]:
        param = getattr(net, param_name)
        numerical_grad = np.zeros_like(param)
        it = np.nditer(param, flags=["multi_index"])
        while not it.finished:
            idx = it.multi_index
            orig = param[idx]

            param[idx] = orig + eps
            L_plus = cross_entropy_loss(net.forward(X), Y)

            param[idx] = orig - eps
            L_minus = cross_entropy_loss(net.forward(X), Y)

            numerical_grad[idx] = (L_plus - L_minus) / (2 * eps)
            param[idx] = orig
            it.iternext()

        net.forward(X)
        analytic_grad = net.backward(Y)[f"d{param_name}"]
        diff = np.abs(numerical_grad - analytic_grad).max()
        print(f"{param_name}: max diff = {diff:.2e} {'✓' if diff < 1e-5 else '✗'}")
```

일반적으로 수치 미분과 해석적 미분의 차이가 `1e-5` 미만이면 구현이 올바른 것으로 판단한다.

### 5. Adam 옵티마이저 이해

단순 SGD보다 Adam(Adaptive Moment Estimation)이 실제로 더 잘 동작하는 이유는, 각 파라미터마다 학습률을 적응적으로 조절하기 때문이다.

```python
class AdamOptimizer:
    def __init__(self, params, lr=1e-3, beta1=0.9, beta2=0.999, eps=1e-8):
        self.params = params
        self.lr, self.beta1, self.beta2, self.eps = lr, beta1, beta2, eps
        self.m = {id(p): np.zeros_like(p) for p in params}  # 1차 모멘트
        self.v = {id(p): np.zeros_like(p) for p in params}  # 2차 모멘트
        self.t = 0

    def step(self, grads):
        self.t += 1
        for p, g in zip(self.params, grads):
            key = id(p)
            self.m[key] = self.beta1 * self.m[key] + (1 - self.beta1) * g
            self.v[key] = self.beta2 * self.v[key] + (1 - self.beta2) * g**2

            # 편향 보정
            m_hat = self.m[key] / (1 - self.beta1**self.t)
            v_hat = self.v[key] / (1 - self.beta2**self.t)

            p -= self.lr * m_hat / (np.sqrt(v_hat) + self.eps)
```

## 참고 자료
- [Neural Networks and Deep Learning — Chapter 2 (Michael Nielsen)](http://neuralnetworksanddeeplearning.com/chap2.html)
- [CS231n: Backpropagation, Intuitions (Stanford)](https://cs231n.github.io/optimization-2/)
- [PyTorch Autograd 공식 문서](https://pytorch.org/docs/stable/notes/autograd.html)
- [Backpropagation Step-by-Step Derivation (Swarthmore)](https://www.cs.swarthmore.edu/~meeden/cs81/s10/BackPropDeriv.pdf)
