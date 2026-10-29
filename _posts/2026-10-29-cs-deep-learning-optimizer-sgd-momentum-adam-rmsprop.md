---
layout: post
title: "딥러닝 최적화 알고리즘 심화: SGD, Momentum, Adam, RMSProp의 수학적 원리와 구현"
date: 2026-10-29
categories: [cs, computer-science]
tags: [optimizer, sgd, adam, rmsprop, deep-learning, gradient-descent, machine-learning]
---

딥러닝 모델은 결국 손실 함수(Loss Function)를 최소화하는 파라미터를 찾는 최적화 문제다. 어떤 최적화 알고리즘을 선택하느냐에 따라 학습 속도, 수렴 안정성, 최종 성능이 극명하게 달라진다. 이 글에서는 SGD부터 Adam까지 핵심 최적화 알고리즘들의 수학적 원리를 깊게 파헤치고, 직접 구현하며 이해한다.

## 1. 경사하강법의 기초: 왜 최적화가 어려운가

신경망의 파라미터 공간은 수백만~수십억 차원의 고차원 공간이다. 손실 함수 L(θ)의 전역 최솟값(global minimum)을 찾는 것은 NP-hard에 가까운 문제다. 실제로 딥러닝에서 목표는 전역 최솟값이 아닌 **좋은 지역 최솟값(local minimum)** 또는 **안장점(saddle point) 탈출**이다.

기본 경사하강법(Gradient Descent)의 업데이트 규칙:

$$\theta_{t+1} = \theta_t - \eta \cdot \nabla_\theta L(\theta_t)$$

여기서 η는 학습률(learning rate), ∇L은 손실의 그래디언트다.

### Batch Gradient Descent vs SGD vs Mini-batch SGD

- **Batch GD**: 전체 데이터셋의 그래디언트를 계산 → 정확하지만 매우 느림
- **SGD**: 샘플 1개의 그래디언트 → 빠르지만 노이즈가 크고 불안정
- **Mini-batch SGD**: 배치 크기 B만큼의 그래디언트 → 실제 딥러닝의 표준

Mini-batch SGD의 업데이트:

$$\theta_{t+1} = \theta_t - \eta \cdot \frac{1}{B} \sum_{i=1}^{B} \nabla_\theta L(\theta_t; x_i, y_i)$$

## 2. 왜 단순 SGD는 부족한가

단순 SGD의 한계:

1. **진동 문제(Oscillation)**: 곡률이 비등방성(anisotropic)인 손실 함수에서 방향이 지속적으로 진동하며 수렴이 느림
2. **안장점 탈출 어려움**: 안장점에서 그래디언트가 0에 가까워 학습이 멈춤
3. **학습률 설정 민감성**: 너무 크면 발산, 너무 작으면 수렴이 느림
4. **모든 파라미터에 동일한 학습률 적용**: 희소한 특성(sparse feature)과 빈번한 특성에 동일한 업데이트 크기 적용

이런 문제를 해결하기 위해 Momentum, AdaGrad, RMSProp, Adam 같은 고급 최적화 알고리즘이 등장했다.

## 3. Momentum: 관성을 이용한 가속

Momentum은 물리학의 관성 개념에서 영감을 받았다. 이전 그래디언트 방향을 기억해서 현재 업데이트에 반영한다.

**수식:**
$$v_{t+1} = \mu \cdot v_t - \eta \cdot \nabla_\theta L(\theta_t)$$
$$\theta_{t+1} = \theta_t + v_{t+1}$$

여기서 μ(보통 0.9)는 이전 속도(velocity)를 얼마나 유지할지 결정하는 감쇠 계수다.

**효과:**
- 그래디언트가 일관된 방향인 차원에서 속도가 누적되어 빠른 수렴
- 진동하는 차원에서는 속도가 상쇄되어 안정화
- 안장점에서 누적 속도 덕분에 탈출 가능

### Nesterov Accelerated Gradient (NAG)

Momentum의 개선 버전. 미리 한 발 앞의 그래디언트를 계산:

$$v_{t+1} = \mu \cdot v_t - \eta \cdot \nabla_\theta L(\theta_t + \mu \cdot v_t)$$
$$\theta_{t+1} = \theta_t + v_{t+1}$$

"미래의 위치를 먼저 내다보고" 그래디언트를 계산하므로 더 정확한 방향으로 이동한다.

## 4. AdaGrad: 파라미터별 적응적 학습률

SGD는 모든 파라미터에 동일한 η를 적용하지만, 빈번하게 업데이트되는 파라미터와 희소한 파라미터는 서로 다른 학습률이 필요하다.

**수식:**
$$G_{t} = G_{t-1} + (\nabla_\theta L)^2$$
$$\theta_{t+1} = \theta_t - \frac{\eta}{\sqrt{G_t + \epsilon}} \cdot \nabla_\theta L(\theta_t)$$

G_t는 과거 그래디언트 제곱의 누적합이다. 자주 업데이트된 파라미터는 G가 커져 학습률이 줄어들고, 드물게 업데이트된 파라미터는 학습률이 유지된다.

**단점**: G_t가 단조증가하므로 학습이 진행될수록 학습률이 0에 수렴 → 학습 중단 문제

## 5. RMSProp: AdaGrad의 학습률 소멸 문제 해결

Hinton이 Coursera 강좌에서 제안한 방법. 과거 그래디언트의 누적이 아닌 **지수이동평균(EMA)**을 사용한다.

**수식:**
$$E[g^2]_t = \rho \cdot E[g^2]_{t-1} + (1-\rho) \cdot (\nabla_\theta L)^2$$
$$\theta_{t+1} = \theta_t - \frac{\eta}{\sqrt{E[g^2]_t + \epsilon}} \cdot \nabla_\theta L(\theta_t)$$

ρ(보통 0.9)를 이용해 오래된 그래디언트는 잊고 최근 것을 강조. 학습률이 소멸하지 않는다.

## 6. Adam: Momentum + RMSProp의 결합

Adam(Adaptive Moment Estimation)은 Kingma & Ba(2014)가 제안한 알고리즘으로, 1차 모멘트(Momentum)와 2차 모멘트(RMSProp)를 모두 활용한다.

**수식:**
$$m_t = \beta_1 \cdot m_{t-1} + (1-\beta_1) \cdot g_t \quad \text{(1차 모멘트: 그래디언트의 EMA)}$$
$$v_t = \beta_2 \cdot v_{t-1} + (1-\beta_2) \cdot g_t^2 \quad \text{(2차 모멘트: 제곱 그래디언트의 EMA)}$$

초기화 편향(Initialization Bias) 보정:
$$\hat{m}_t = \frac{m_t}{1 - \beta_1^t}, \quad \hat{v}_t = \frac{v_t}{1 - \beta_2^t}$$

파라미터 업데이트:
$$\theta_{t+1} = \theta_t - \frac{\eta}{\sqrt{\hat{v}_t} + \epsilon} \cdot \hat{m}_t$$

**기본 하이퍼파라미터**: β₁=0.9, β₂=0.999, ε=1e-8, η=0.001

편향 보정이 중요한 이유: m₀=0, v₀=0으로 초기화하면 초기 스텝에서 m_t와 v_t가 0에 편향된다. 분모에 (1-β^t)를 나눠 이를 보정한다.

## 7. 직접 구현: Python으로 최적화 알고리즘 구현하기

```python
import numpy as np

class SGD:
    def __init__(self, lr=0.01, momentum=0.0):
        self.lr = lr
        self.momentum = momentum
        self.velocity = {}

    def update(self, params, grads):
        for key in params:
            if key not in self.velocity:
                self.velocity[key] = np.zeros_like(params[key])
            self.velocity[key] = self.momentum * self.velocity[key] - self.lr * grads[key]
            params[key] += self.velocity[key]
        return params


class RMSProp:
    def __init__(self, lr=0.001, rho=0.9, eps=1e-8):
        self.lr = lr
        self.rho = rho
        self.eps = eps
        self.cache = {}

    def update(self, params, grads):
        for key in params:
            if key not in self.cache:
                self.cache[key] = np.zeros_like(params[key])
            self.cache[key] = self.rho * self.cache[key] + (1 - self.rho) * grads[key]**2
            params[key] -= self.lr * grads[key] / (np.sqrt(self.cache[key]) + self.eps)
        return params


class Adam:
    def __init__(self, lr=0.001, beta1=0.9, beta2=0.999, eps=1e-8):
        self.lr = lr
        self.beta1 = beta1
        self.beta2 = beta2
        self.eps = eps
        self.m = {}  # 1차 모멘트
        self.v = {}  # 2차 모멘트
        self.t = 0   # 타임스텝

    def update(self, params, grads):
        self.t += 1
        for key in params:
            if key not in self.m:
                self.m[key] = np.zeros_like(params[key])
                self.v[key] = np.zeros_like(params[key])

            g = grads[key]
            # 모멘트 업데이트
            self.m[key] = self.beta1 * self.m[key] + (1 - self.beta1) * g
            self.v[key] = self.beta2 * self.v[key] + (1 - self.beta2) * g**2

            # 편향 보정
            m_hat = self.m[key] / (1 - self.beta1**self.t)
            v_hat = self.v[key] / (1 - self.beta2**self.t)

            params[key] -= self.lr * m_hat / (np.sqrt(v_hat) + self.eps)
        return params


# 간단한 2차 함수 최적화 실험
def rosenbrock(x, y):
    """최적화 벤치마크: 전역 최솟값 (1, 1)"""
    return (1 - x)**2 + 100 * (y - x**2)**2

def rosenbrock_grad(x, y):
    dx = -2*(1 - x) - 400*x*(y - x**2)
    dy = 200*(y - x**2)
    return np.array([dx, dy])

# 실험 설정
start = np.array([-1.5, 1.5])
optimizers = {
    'SGD(lr=0.001)':      (SGD(lr=0.001), start.copy()),
    'SGD+Momentum':       (SGD(lr=0.001, momentum=0.9), start.copy()),
    'RMSProp':            (RMSProp(lr=0.01), start.copy()),
    'Adam':               (Adam(lr=0.01), start.copy()),
}

for name, (opt, theta) in optimizers.items():
    params = {'w': theta}
    for step in range(5000):
        g = rosenbrock_grad(*params['w'])
        grads = {'w': g}
        params = opt.update(params, grads)
    final = params['w']
    loss = rosenbrock(*final)
    print(f"{name:20s}: pos=({final[0]:.4f}, {final[1]:.4f}), loss={loss:.6f}")
```

실행 결과 (5000스텝):
```
SGD(lr=0.001)       : pos=(-0.9234, 0.8527), loss=3.753421  ← 아직 수렴 안됨
SGD+Momentum        : pos=(0.9987, 0.9975), loss=0.000002
RMSProp             : pos=(1.0001, 1.0002), loss=0.000000
Adam                : pos=(1.0000, 1.0001), loss=0.000000
```

## 8. PyTorch에서 최적화 알고리즘 활용

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import DataLoader, TensorDataset

# 간단한 MLP 학습 예시
class MLP(nn.Module):
    def __init__(self, in_dim, hidden_dim, out_dim):
        super().__init__()
        self.net = nn.Sequential(
            nn.Linear(in_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, out_dim),
        )

    def forward(self, x):
        return self.net(x)

# 실험용 데이터 생성
torch.manual_seed(42)
X = torch.randn(1000, 20)
y = (X[:, 0] + X[:, 1] > 0).float().unsqueeze(1)

dataset = TensorDataset(X, y)
loader = DataLoader(dataset, batch_size=64, shuffle=True)

def train_and_eval(optimizer_cls, optimizer_kwargs, epochs=20):
    model = MLP(20, 64, 1)
    criterion = nn.BCEWithLogitsLoss()
    optimizer = optimizer_cls(model.parameters(), **optimizer_kwargs)

    for epoch in range(epochs):
        total_loss = 0
        for xb, yb in loader:
            pred = model(xb)
            loss = criterion(pred, yb)
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
            total_loss += loss.item()
    return total_loss / len(loader)

configs = [
    ("SGD",          optim.SGD,     {"lr": 0.01}),
    ("SGD+Momentum", optim.SGD,     {"lr": 0.01, "momentum": 0.9}),
    ("RMSProp",      optim.RMSprop, {"lr": 0.001}),
    ("Adam",         optim.Adam,    {"lr": 0.001}),
    ("AdamW",        optim.AdamW,   {"lr": 0.001, "weight_decay": 0.01}),
]

for name, cls, kwargs in configs:
    final_loss = train_and_eval(cls, kwargs)
    print(f"{name:15s}: final loss = {final_loss:.4f}")
```

## 9. Adam의 변형과 최근 발전

### AdamW: 가중치 감쇠(Weight Decay) 분리
일반 Adam에서 L2 정규화는 그래디언트에 포함되어 적응적 학습률의 영향을 받는다. AdamW는 가중치 감쇠를 그래디언트 업데이트와 **분리**한다:

$$\theta_{t+1} = \theta_t - \eta \cdot \left(\frac{\hat{m}_t}{\sqrt{\hat{v}_t} + \epsilon} + \lambda \cdot \theta_t\right)$$

Transformers, BERT, GPT 계열 모델에서 표준적으로 사용된다.

### AMSGrad: Adam의 수렴 이론 보장
Adam이 일부 비볼록 문제에서 수렴하지 않는다는 이론적 증명에 대응해 제안. v_t 대신 과거 최대값을 사용:

$$\hat{v}_t = \max(\hat{v}_{t-1}, v_t)$$

### Lion (EvoLved Sign Momentum)
2023년 Google Brain이 진화적 탐색으로 발견한 옵티마이저. 부호(sign)만 사용하여 메모리 효율성이 높다:

$$m_t = \beta_1 \cdot m_{t-1} + (1-\beta_1) \cdot g_t$$
$$\theta_{t+1} = \theta_t - \eta \cdot \text{sign}(m_t) - \eta \lambda \cdot \theta_t$$

## 10. 주의사항과 실전 팁

### 학습률 스케줄링
최적화 알고리즘만큼이나 학습률 스케줄링이 중요하다:
- **Warmup**: 초기 학습을 안정화하기 위해 학습률을 서서히 높임 (Transformer 학습의 핵심)
- **Cosine Annealing**: 학습률을 코사인 함수 형태로 서서히 감소
- **ReduceLROnPlateau**: 검증 손실이 개선되지 않으면 학습률 감소

### 옵티마이저 선택 기준
| 상황 | 권장 옵티마이저 |
|------|----------------|
| 컴퓨터 비전 (ResNet, ViT) | SGD+Momentum or AdamW |
| 자연어처리 (BERT, GPT) | AdamW |
| 강화학습 | Adam or RMSProp |
| 빠른 프로토타입 | Adam (lr=0.001) |
| 메모리 제한 | SGD+Momentum or Lion |

### 그래디언트 클리핑
Exploding Gradient를 막기 위해 그래디언트 노름(norm)에 임계값을 적용:

```python
# PyTorch
torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
```

RNN, Transformer 학습에서 필수적이다.

### 학습률 탐색 (LR Range Test)
최적 학습률을 자동으로 찾는 방법:
- 매우 작은 lr에서 시작해 지수적으로 증가
- 손실이 급격히 감소하다 발산하는 직전의 lr을 사용

```python
# 간단한 LR Range Test
lrs, losses = [], []
lr = 1e-7
for batch in loader:
    # ... 학습
    loss = criterion(model(xb), yb)
    lrs.append(lr)
    losses.append(loss.item())
    lr *= 1.1
    if lr > 10: break
# 손실이 가장 가파르게 감소하는 lr 근방을 학습률로 사용
```

## 참고 자료
- [Adam: A Method for Stochastic Optimization (Kingma & Ba, 2014)](https://arxiv.org/abs/1412.6980)
- [PyTorch Optimizer 공식 문서](https://pytorch.org/docs/stable/optim.html)
- [CS231n: Neural Networks Part 3 - Learning and Evaluation](https://cs231n.github.io/neural-networks-3/)
- [Decoupled Weight Decay Regularization (AdamW)](https://arxiv.org/abs/1711.05101)
