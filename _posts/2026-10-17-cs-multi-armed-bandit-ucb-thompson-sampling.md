---
layout: post
title: "멀티 암드 밴딧(Multi-Armed Bandit): UCB와 Thompson Sampling으로 탐색-활용 딜레마 정복하기"
date: 2026-10-17
categories: [cs, computer-science]
tags: [멀티암드밴딧, 강화학습, UCB, Thompson샘플링, 탐색과활용, 확률적알고리즘, A/B테스트]
---

## 탐색과 활용 딜레마

새로운 도시에서 저녁을 먹으러 나갔다고 상상해봅시다. 한 식당은 이미 맛있다는 것을 알지만, 아직 가보지 못한 식당들이 더 맛있을 수도 있습니다. 이미 아는 선택지를 골라 확실한 보상을 취할 것인가(**활용, exploitation**), 아니면 새로운 선택지를 탐색해 더 나은 것을 찾을 것인가(**탐색, exploration**)?

이것이 바로 **탐색-활용 딜레마(Exploration-Exploitation Tradeoff)** 입니다. 이 문제를 형식화한 것이 **멀티 암드 밴딧(Multi-Armed Bandit, MAB)** 문제입니다. 이름은 여러 팔(arm)을 가진 슬롯머신에서 유래했습니다. 각 팔마다 당첨 확률이 다르며, 어떤 팔이 최선인지 모르는 상태에서 총 보상을 극대화해야 합니다.

## 왜 중요한가: 실제 응용 분야

MAB 문제는 추상적인 이론이 아닙니다. 현실 세계 곳곳에서 핵심 알고리즘으로 활용됩니다.

| 응용 분야 | 팔(Arm) | 보상(Reward) |
|---------|---------|------------|
| 온라인 광고 | 광고 문안 변형 | 클릭률 (CTR) |
| 임상시험 | 치료 방법 | 환자 회복률 |
| 웹 UI A/B 테스트 | 버튼 색상/위치 | 전환율 (CVR) |
| 추천 시스템 | 추천 콘텐츠 | 시청/구매율 |
| 하이퍼파라미터 튜닝 | 모델 설정 | 검증 정확도 |
| 네트워크 라우팅 | 경로 선택 | 레이턴시/처리량 |

전통적인 A/B 테스트는 미리 정해진 기간 동안 트래픽을 균등하게 나눠 최종 결과를 비교합니다. 반면 MAB는 **실험 도중에도 더 좋아 보이는 팔에 더 많은 트래픽을 점진적으로 배분**합니다. 이를 통해 실험 중 손실(regret)을 최소화하면서도 최적 팔을 빠르게 식별할 수 있습니다.

## 핵심 개념: 누적 후회(Cumulative Regret)

알고리즘의 품질은 **후회(Regret)** 로 측정합니다. 후회는 최적 팔만 선택했을 때의 보상과 실제 보상의 차이입니다.

```
R(T) = T * μ* - Σ_{t=1}^{T} r_t

여기서:
  T    = 총 라운드 수
  μ*   = 최적 팔의 기대 보상
  r_t  = 라운드 t에서 실제로 얻은 보상
```

좋은 알고리즘은 T가 커질수록 R(T)/T → 0이 되어야 합니다. 이론적으로 최적 알고리즘의 후회는 **O(log T)** 임이 증명되어 있습니다(Lai-Robbins 하한).

```python
import numpy as np
import matplotlib.pyplot as plt
from typing import List

# 시뮬레이션을 위한 기본 설정
np.random.seed(42)

class BernoulliBandit:
    """베르누이 분포를 따르는 K개 팔의 슬롯머신"""
    def __init__(self, true_probs: List[float]):
        self.true_probs = true_probs
        self.k = len(true_probs)
        self.optimal_arm = np.argmax(true_probs)
        self.optimal_reward = max(true_probs)

    def pull(self, arm: int) -> int:
        """arm을 당겼을 때 0 또는 1을 반환"""
        return int(np.random.random() < self.true_probs[arm])


# 5개 팔, 각 팔의 실제 성공 확률 (알고리즘은 이를 모름)
TRUE_PROBS = [0.2, 0.5, 0.4, 0.8, 0.3]
bandit = BernoulliBandit(TRUE_PROBS)
```

## 알고리즘 1: Epsilon-Greedy (기준점)

가장 단순한 접근법입니다. 확률 ε로 랜덤 탐색, 확률 1-ε로 현재까지 최고 성능의 팔을 선택합니다.

```python
class EpsilonGreedy:
    def __init__(self, k: int, epsilon: float = 0.1):
        self.k = k
        self.epsilon = epsilon
        self.counts = np.zeros(k)    # 각 팔 선택 횟수
        self.values = np.zeros(k)    # 각 팔 평균 보상

    def select_arm(self) -> int:
        if np.random.random() < self.epsilon:
            return np.random.randint(self.k)  # 탐색
        return np.argmax(self.values)          # 활용

    def update(self, arm: int, reward: float):
        self.counts[arm] += 1
        n = self.counts[arm]
        # 온라인 평균 업데이트: μ_n = μ_{n-1} + (r - μ_{n-1}) / n
        self.values[arm] += (reward - self.values[arm]) / n
```

**문제점:** ε이 고정되어 있어 탐색이 과도하게 많거나 적을 수 있습니다. 시간이 지나도 최적이 아닌 팔을 계속 탐색합니다.

## 알고리즘 2: UCB (Upper Confidence Bound)

UCB는 "낙관적 불확실성 처리(Optimism in the Face of Uncertainty)" 원칙을 따릅니다. 각 팔에 대해 **신뢰 상한선(Upper Confidence Bound)** 을 계산하고, 가장 높은 상한선을 가진 팔을 선택합니다.

```
UCB1 선택 기준:
  arm* = argmax_i [ μ̂_i + √(2 * ln(t) / n_i) ]

여기서:
  μ̂_i = 팔 i의 현재 평균 보상
  t    = 현재 총 라운드 수
  n_i  = 팔 i의 선택 횟수
```

신뢰 상한 항 `√(2 * ln(t) / n_i)` 는 팔 i가 적게 선택될수록(n_i 작을수록) 커집니다. 즉, **덜 탐색된 팔에 보너스를 부여**하여 자연스럽게 탐색을 유도합니다.

```python
class UCB1:
    def __init__(self, k: int):
        self.k = k
        self.counts = np.zeros(k)
        self.values = np.zeros(k)
        self.total = 0

    def select_arm(self) -> int:
        # 아직 한 번도 선택하지 않은 팔이 있으면 먼저 선택
        for i in range(self.k):
            if self.counts[i] == 0:
                return i

        # UCB1 스코어 계산
        ucb_scores = self.values + np.sqrt(
            2 * np.log(self.total) / self.counts
        )
        return int(np.argmax(ucb_scores))

    def update(self, arm: int, reward: float):
        self.total += 1
        self.counts[arm] += 1
        n = self.counts[arm]
        self.values[arm] += (reward - self.values[arm]) / n


# UCB1의 후회 상한 (이론적 보장):
# R(T) ≤ Σ_{i≠i*} [ (8 * ln T) / Δ_i ] + (1 + π²/3) * Σ Δ_i
# 여기서 Δ_i = μ* - μ_i (최적과의 간극)
```

UCB1은 **O(log T)** 후회를 보장합니다. Lai & Robbins(1985)의 정보 이론적 하한에 점근적으로 최적입니다.

## 알고리즘 3: Thompson Sampling (베이지안 접근)

Thompson Sampling은 베이지안 확률론에 기반합니다. 각 팔의 보상 확률을 **사전 분포(prior)**로 모델링하고, 관측 데이터를 통해 **사후 분포(posterior)**를 업데이트합니다. 매 라운드마다 각 팔의 사후 분포에서 샘플을 추출하고, 가장 높은 샘플을 가진 팔을 선택합니다.

베르누이 밴딧의 경우 켤레 사전(conjugate prior)으로 **베타 분포(Beta distribution)** 를 사용합니다.

```
사전 분포:   θ_i ~ Beta(α_i, β_i)   (초기: α=1, β=1 → Uniform[0,1])
보상 관측:   r_t ∈ {0, 1}
사후 업데이트:
  성공(r=1): α_i += 1
  실패(r=0): β_i += 1
```

```python
class ThompsonSampling:
    def __init__(self, k: int):
        self.k = k
        # Beta 분포 파라미터: 초기 = Beta(1, 1) = Uniform
        self.alpha = np.ones(k)  # 성공 횟수 + 1
        self.beta  = np.ones(k)  # 실패 횟수 + 1

    def select_arm(self) -> int:
        # 각 팔의 사후 분포에서 샘플 추출
        samples = np.random.beta(self.alpha, self.beta)
        return int(np.argmax(samples))

    def update(self, arm: int, reward: int):
        # 베이즈 업데이트: 관측된 보상으로 분포 갱신
        if reward == 1:
            self.alpha[arm] += 1
        else:
            self.beta[arm] += 1


# 직관적 이해:
# - 아직 탐색이 적은 팔: α, β가 모두 작아 분포 불확실성이 크다
#   → 높은 샘플이 나올 확률이 높아 자연스럽게 탐색됨
# - 많이 탐색한 팔: α, β가 커지며 분포가 실제 확률에 수렴
#   → 분산이 줄어 안정적으로 활용됨
```

**Thompson Sampling의 장점:**
- 구현이 UCB보다 단순합니다.
- 파라미터 튜닝이 불필요합니다 (UCB는 탐색 계수 C를 튜닝해야 함).
- 실제로 UCB1보다 낮은 후회를 보이는 경우가 많습니다.
- 연속적인 보상(가우시안 분포 등)으로 확장하기 쉽습니다.

## 전체 시뮬레이션 및 비교

```python
def run_simulation(algorithm, bandit: BernoulliBandit,
                   n_rounds: int = 1000) -> dict:
    """알고리즘을 n_rounds 동안 실행하고 결과 반환"""
    rewards = []
    chosen_arms = []
    cumulative_regret = []
    total_regret = 0

    for t in range(n_rounds):
        arm = algorithm.select_arm()
        reward = bandit.pull(arm)
        algorithm.update(arm, reward)

        regret = bandit.optimal_reward - bandit.true_probs[arm]
        total_regret += regret

        rewards.append(reward)
        chosen_arms.append(arm)
        cumulative_regret.append(total_regret)

    return {
        'total_reward': sum(rewards),
        'cumulative_regret': cumulative_regret,
        'arm_counts': (
            np.bincount(chosen_arms, minlength=bandit.k).tolist()
        ),
    }

# 각 알고리즘 실행
N_ROUNDS = 2000
algorithms = {
    'Epsilon-Greedy(ε=0.1)': EpsilonGreedy(bandit.k, epsilon=0.1),
    'UCB1': UCB1(bandit.k),
    'Thompson Sampling': ThompsonSampling(bandit.k),
}

print(f"최적 팔: {bandit.optimal_arm} (확률: {TRUE_PROBS[bandit.optimal_arm]})")
print(f"{'알고리즘':<25} {'총 보상':>8} {'누적 후회':>10} {'최적팔 선택 비율':>16}")
print("-" * 65)

for name, algo in algorithms.items():
    result = run_simulation(algo, bandit, N_ROUNDS)
    total_reward = result['total_reward']
    final_regret = result['cumulative_regret'][-1]
    arm_counts = result['arm_counts']
    optimal_ratio = arm_counts[bandit.optimal_arm] / N_ROUNDS * 100
    print(f"{name:<25} {total_reward:>8} {final_regret:>10.2f} {optimal_ratio:>15.1f}%")
```

실행 결과 (예시):
```
최적 팔: 3 (확률: 0.8)
알고리즘                   총 보상   누적 후회  최적팔 선택 비율
-----------------------------------------------------------------
Epsilon-Greedy(ε=0.1)      1476     138.42          85.5%
UCB1                        1559      62.18          94.3%
Thompson Sampling           1571      48.31          95.8%
```

## 컨텍스트 밴딧(Contextual Bandit)으로의 확장

기본 MAB는 팔의 보상이 고정되어 있다고 가정합니다. **컨텍스트 밴딧**은 각 라운드마다 "문맥(context)" 벡터(사용자 특성, 시간대, 기기 등)가 주어지고, 이를 활용해 팔의 보상을 예측합니다.

```python
class LinUCB:
    """선형 컨텍스트 밴딧: LinUCB 알고리즘"""
    def __init__(self, k: int, d: int, alpha: float = 1.0):
        self.k = k
        self.d = d      # 컨텍스트 벡터 차원
        self.alpha = alpha
        # 각 팔마다 독립적인 릿지 회귀 모델
        self.A = [np.eye(d) for _ in range(k)]       # (d x d) 행렬
        self.b = [np.zeros((d, 1)) for _ in range(k)] # (d x 1) 벡터

    def select_arm(self, context: np.ndarray) -> int:
        x = context.reshape(-1, 1)  # (d x 1)
        p = []
        for i in range(self.k):
            A_inv = np.linalg.inv(self.A[i])
            theta = A_inv @ self.b[i]               # 추정 파라미터
            ucb = (theta.T @ x + self.alpha *
                   np.sqrt(x.T @ A_inv @ x)).item()
            p.append(ucb)
        return int(np.argmax(p))

    def update(self, arm: int, context: np.ndarray, reward: float):
        x = context.reshape(-1, 1)
        self.A[arm] += x @ x.T
        self.b[arm] += reward * x

# 사용 예시
d = 5  # 컨텍스트 벡터 차원
linucb = LinUCB(k=3, d=d, alpha=1.5)
for _ in range(100):
    ctx = np.random.randn(d)
    arm = linucb.select_arm(ctx)
    reward = np.random.random()  # 실제 보상
    linucb.update(arm, ctx, reward)
```

## 주의사항과 실용적 팁

**1. 정상성(Stationarity) 가정**
기본 MAB는 팔의 보상 분포가 시간이 지나도 변하지 않는다고 가정합니다. 실제 광고 CTR이나 사용자 선호도는 시간에 따라 변합니다. 이 경우 **Sliding Window UCB** 또는 **Discounted UCB**를 사용하세요.

```python
class DiscountedUCB:
    """시간이 지남에 따라 오래된 정보를 할인하는 UCB 변형"""
    def __init__(self, k: int, gamma: float = 0.99, xi: float = 0.5):
        self.k = k
        self.gamma = gamma  # 할인 인자 (0 < γ < 1)
        self.xi = xi        # 탐색 계수
        self.n = np.zeros(k)     # 할인된 선택 횟수
        self.mu = np.zeros(k)    # 할인된 평균 보상
        self.t = 0

    def update(self, arm: int, reward: float):
        self.t += 1
        for i in range(self.k):
            self.n[i] *= self.gamma
            self.mu[i] *= self.gamma
        self.n[arm] += 1
        self.mu[arm] += reward
```

**2. 배치 업데이트**
실시간 시스템에서는 매 요청마다 모델을 업데이트하기 어렵습니다. 일정 수의 관측을 모아 배치로 업데이트하는 방식이 현실적입니다.

**3. 다수 사용자 동시 처리**
여러 사용자가 동시에 요청을 보낼 때, 같은 팔을 여러 번 동시에 선택하는 문제가 생깁니다. **배치 Thompson Sampling**이나 **Bootstrapped UCB** 등의 변형을 검토하세요.

**4. 탐색 계수 튜닝 (UCB)**
UCB1의 탐색 계수(√2 * ln t)는 이론적으로 최적이지만, 실제 데이터에서는 더 작은 계수가 잘 동작할 수 있습니다. 도메인 지식을 반영해 튜닝하세요.

**5. Prior 선택 (Thompson Sampling)**
베르누이 보상이 아닌 연속 보상에는 가우시안 Thompson Sampling(사전: 정규분포, 우도: 정규분포)을 사용합니다. 강한 도메인 지식이 있다면 균일 사전 대신 정보적 사전(informative prior)을 설정하세요.

## 정리

멀티 암드 밴딧 문제는 온라인 의사결정의 근본적인 딜레마를 다룹니다. UCB는 수학적으로 탄탄한 보장을 제공하고, Thompson Sampling은 베이지안 관점에서 우아하고 실용적인 해법을 제시합니다. 두 알고리즘 모두 O(log T) 후회를 달성하며, 실제 시스템에서의 성능은 종종 Thompson Sampling이 더 우수합니다. 추천 시스템, 광고 최적화, 임상시험 등 현실 문제에 MAB를 적용할 때는 정상성 가정, 동시성, 배치 업데이트 등의 현실적 제약을 반드시 고려하세요.

## 참고 자료
- [Multi-armed Bandit - Wikipedia](https://en.wikipedia.org/wiki/Multi-armed_bandit)
- [Multi-Armed Bandit Problem in Reinforcement Learning - GeeksforGeeks](https://www.geeksforgeeks.org/multi-armed-bandit-problem-in-reinforcement-learning/)
- [Sutton & Barto: Reinforcement Learning: An Introduction (Chapter 2)](http://incompleteideas.net/book/RLbook2020.pdf)
- [A Tutorial on Thompson Sampling (Russo et al., 2020)](https://arxiv.org/abs/1707.02038)
