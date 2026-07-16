---
layout: post
title: "차분 프라이버시(Differential Privacy) 완전 정복: 수학적으로 개인정보를 보호하는 방법"
date: 2026-07-16
categories: [cs, computer-science]
tags: [differential-privacy, privacy, machine-learning, statistics, laplace-mechanism, gaussian-mechanism]
---

## 차분 프라이버시란 무엇인가

개인정보 보호를 위한 전통적 접근법은 **익명화(anonymization)**였다. 이름, 주민등록번호 등 식별자를 제거하면 개인을 특정할 수 없다고 생각했다. 그러나 2006년 Netflix는 10만 명의 영화 평점 데이터를 익명화해 공개했지만, Arvind Narayanan과 Vitaly Shmatikoff는 공개 IMDB 평점 데이터와 교차 분석하여 개인을 재식별하는 데 성공했다. 익명화만으로는 불충분함이 증명된 것이다.

**차분 프라이버시(Differential Privacy, DP)**는 2006년 Cynthia Dwork이 제안한 수학적 프라이버시 보장 프레임워크다. 특정 개인의 데이터가 데이터셋에 포함되어 있든 없든, 쿼리 결과가 **거의 동일**해야 한다는 원칙을 수학적으로 정의한다. 공격자가 무한한 배경 지식을 가지고 있더라도, 특정 개인에 대한 추가 정보를 얻을 수 없도록 보장한다.

Apple, Google, Microsoft, US Census Bureau 등이 실제로 사용자 데이터 수집에 차분 프라이버시를 적용하고 있다.

---

## 왜 차분 프라이버시가 필요한가

### 합계 공격 (Sum Attack)

10명의 급여 데이터가 있다고 가정하자.

```
전체 급여 합계: 5억 원
Alice를 제외한 9명의 합계: 4억 2천만 원
→ Alice의 급여 = 8천만 원 (공개된 집계 정보만으로 추론 가능!)
```

집계 쿼리라도 특정 개인의 정보를 **직접적으로 노출**할 수 있다. 이것이 바로 차분 프라이버시가 해결하려는 문제다.

### 재식별 공격 (Re-identification)

위 Netflix 사례처럼, 여러 데이터셋을 조합하면 익명화된 데이터에서도 개인을 특정할 수 있다. 차분 프라이버시는 출력 자체에 노이즈를 추가해 재식별 자체를 수학적으로 어렵게 만든다.

---

## 형식적 정의

### ε-차분 프라이버시 (순수 차분 프라이버시)

이웃 데이터셋(neighboring datasets) D와 D'을 **단 한 개의 레코드만 다른 데이터셋**으로 정의한다. 랜덤화된 알고리즘 M이 모든 이웃 데이터셋 D, D'과 모든 가능한 출력 집합 S에 대해 다음을 만족하면 **ε-차분 프라이버시**를 만족한다:

```
Pr[M(D) ∈ S] ≤ e^ε × Pr[M(D') ∈ S]
```

- **ε (엡실론)**: 프라이버시 예산(privacy budget). 0에 가까울수록 강한 프라이버시 보호. 작은 ε는 두 분포가 거의 구별 불가능함을 의미.
- 보통 실무에서 ε ∈ {0.1, 0.5, 1.0, 3.0} 범위를 사용.

### (ε, δ)-차분 프라이버시 (완화된 정의)

순수 DP는 너무 제약이 강해 유용성이 떨어질 수 있다. δ를 추가해 소확률로 ε 범위를 위반하는 것을 허용한다:

```
Pr[M(D) ∈ S] ≤ e^ε × Pr[M(D') ∈ S] + δ
```

- **δ**: 프라이버시 보장이 완전히 실패할 확률. δ = 0이면 순수 DP와 동일.
- 실무에서 δ ≤ 1/n² (n = 데이터셋 크기)으로 설정하는 것이 일반적.

---

## 핵심 메커니즘 구현

### 민감도(Sensitivity) 개념

메커니즘을 설계하기 전에 **민감도(sensitivity)**를 계산해야 한다. 민감도는 이웃 데이터셋 간 쿼리 결과가 최대 얼마나 달라질 수 있는지를 측정한다.

```
Δf = max_{D, D'이웃} ||f(D) - f(D')||₁
```

예를 들어 "COUNT 쿼리"는 민감도가 1이다 (한 명 추가/삭제 시 카운트가 최대 1 변화). "SUM 쿼리 (최댓값 M으로 클리핑)"는 민감도가 M이다.

### 라플라스 메커니즘 (Laplace Mechanism)

수치형 쿼리에 대한 가장 기본적인 DP 메커니즘이다. 결과에 라플라스 분포에서 샘플링한 노이즈를 추가한다.

```python
import numpy as np
from scipy.stats import laplace

def laplace_mechanism(query_result, sensitivity, epsilon):
    """
    ε-차분 프라이버시를 만족하는 라플라스 메커니즘
    
    query_result: 실제 쿼리 결과 (스칼라 또는 배열)
    sensitivity: 쿼리의 L1 민감도
    epsilon: 프라이버시 예산
    """
    scale = sensitivity / epsilon  # 라플라스 분포의 스케일 파라미터 b = Δf/ε
    noise = np.random.laplace(loc=0.0, scale=scale, size=np.shape(query_result))
    return query_result + noise


# 예제: 차분 프라이버시 보장 급여 합계
salaries = np.array([5000, 6000, 7500, 4800, 9000,
                     5500, 6200, 8000, 7000, 5800])  # 만원 단위
true_sum = np.sum(salaries)
print(f"실제 급여 합계: {true_sum}만원")

# 민감도: 한 명 추가/삭제 시 합계 변화량 최대값 (클리핑 기준 10,000만원)
CLIP_MAX = 10_000  # 최대 급여 클리핑 값
sensitivity = CLIP_MAX

# 다양한 엡실론 값으로 실험
for epsilon in [0.1, 0.5, 1.0, 3.0]:
    dp_result = laplace_mechanism(true_sum, sensitivity, epsilon)
    error = abs(dp_result - true_sum)
    print(f"  ε={epsilon:.1f}: DP 합계={dp_result:.0f}만원, 오차={error:.0f}만원")

# 출력 예시:
# 실제 급여 합계: 64800만원
#   ε=0.1: DP 합계=54321만원, 오차=10479만원  (높은 프라이버시, 낮은 정확도)
#   ε=0.5: DP 합계=62984만원, 오차=1816만원
#   ε=1.0: DP 합계=65103만원, 오차=303만원
#   ε=3.0: DP 합계=64897만원, 오차=97만원   (낮은 프라이버시, 높은 정확도)
```

### 가우시안 메커니즘 (Gaussian Mechanism)

(ε, δ)-DP를 만족시키는 메커니즘. L2 민감도를 사용하며, 고차원 벡터 쿼리에 적합하다.

```python
def gaussian_mechanism(query_result, l2_sensitivity, epsilon, delta):
    """
    (ε, δ)-차분 프라이버시를 만족하는 가우시안 메커니즘
    
    σ ≥ √(2·ln(1.25/δ)) · Δ₂f / ε
    """
    sigma = np.sqrt(2 * np.log(1.25 / delta)) * l2_sensitivity / epsilon
    noise = np.random.normal(loc=0.0, scale=sigma, size=np.shape(query_result))
    return query_result + noise


def private_histogram(data, bins, data_range, epsilon, delta=1e-5):
    """
    차분 프라이버시를 적용한 히스토그램 생성
    
    히스토그램의 L2 민감도 = 1 (한 데이터포인트는 정확히 하나의 빈에 속함)
    """
    counts, edges = np.histogram(data, bins=bins, range=data_range)
    
    # 민감도: 한 데이터포인트 추가/삭제 시 히스토그램 L2 변화량 = 1
    l2_sensitivity = 1.0
    
    dp_counts = gaussian_mechanism(counts.astype(float), l2_sensitivity, epsilon, delta)
    
    # 음수 카운트는 0으로 클리핑 (히스토그램은 비음수)
    dp_counts = np.maximum(dp_counts, 0)
    
    return dp_counts, edges


# 예제: 나이 분포 히스토그램
np.random.seed(42)
ages = np.random.normal(loc=35, scale=10, size=10_000).clip(18, 80).astype(int)

dp_hist, edges = private_histogram(ages, bins=10, data_range=(18, 80),
                                   epsilon=1.0, delta=1e-5)
true_hist, _ = np.histogram(ages, bins=10, range=(18, 80))

print("나이 분포 히스토그램 비교:")
for i, (true, dp) in enumerate(zip(true_hist, dp_hist)):
    age_range = f"{edges[i]:.0f}-{edges[i+1]:.0f}"
    print(f"  {age_range}세: 실제={true:5d}, DP={dp:7.1f}, 오차={abs(dp-true):.1f}")
```

---

## 프라이버시 합성 정리

### 직렬 합성 (Sequential Composition)

같은 데이터셋에 여러 DP 메커니즘을 순차 적용하면 프라이버시 예산이 **합산**된다:

```
M₁이 ε₁-DP, M₂가 ε₂-DP를 만족하면
M₁과 M₂를 순차 적용한 결과는 (ε₁ + ε₂)-DP를 만족
```

```python
# 프라이버시 예산 추적 (Privacy Accounting)
class PrivacyBudget:
    def __init__(self, total_epsilon, total_delta=1e-5):
        self.total_epsilon = total_epsilon
        self.total_delta = total_delta
        self.spent_epsilon = 0.0
        self.spent_delta = 0.0

    def consume(self, epsilon, delta=0.0):
        if self.spent_epsilon + epsilon > self.total_epsilon:
            raise ValueError(f"프라이버시 예산 초과! "
                           f"잔여: {self.total_epsilon - self.spent_epsilon:.3f}")
        self.spent_epsilon += epsilon
        self.spent_delta += delta

    def remaining(self):
        return self.total_epsilon - self.spent_epsilon

    def __repr__(self):
        return (f"PrivacyBudget(소비={self.spent_epsilon:.2f}/"
                f"{self.total_epsilon}, 잔여={self.remaining():.2f})")


# 사용 예
budget = PrivacyBudget(total_epsilon=3.0)

# 쿼리 1: 평균 나이
mean_age = laplace_mechanism(np.mean(ages), sensitivity=62/10_000, epsilon=1.0)
budget.consume(1.0)
print(f"평균 나이 (DP): {mean_age:.1f}, {budget}")

# 쿼리 2: 중앙값 (근사)
budget.consume(1.0)
print(f"예산 소비 후: {budget}")

# 쿼리 3: 예산 초과 시도
try:
    budget.consume(2.0)  # 잔여 1.0이지만 2.0 요청 → 오류
except ValueError as e:
    print(f"오류: {e}")
```

### 병렬 합성 (Parallel Composition)

**서로 겹치지 않는** 데이터 부분집합에 DP 메커니즘을 적용하면 예산이 합산되지 않고 **최대값**만 사용된다:

```
각 파티션에 ε-DP를 적용하면 전체 결과는 ε-DP (합산이 아님!)
```

이를 이용해 히스토그램처럼 데이터를 파티션으로 나누는 쿼리는 하나의 ε만 소비한다.

---

## 실전 응용: 연합 학습에서의 DP

머신러닝에서 **DP-SGD(Differentially Private Stochastic Gradient Descent)**는 훈련 과정에서 차분 프라이버시를 보장한다.

```python
import torch
import torch.nn as nn

def dp_sgd_step(model, data, labels, optimizer,
                epsilon, delta, max_grad_norm, batch_size):
    """
    DP-SGD 한 스텝:
    1. 미니배치 내 각 샘플의 그래디언트를 개별 계산
    2. 그래디언트를 max_grad_norm으로 클리핑 (민감도 제한)
    3. 가우시안 노이즈 추가 (DP 보장)
    4. 파라미터 업데이트
    """
    optimizer.zero_grad()

    # 각 샘플별 그래디언트 계산 (per-sample gradients)
    per_sample_grads = []
    for x, y in zip(data, labels):
        loss = model(x.unsqueeze(0))
        loss = nn.CrossEntropyLoss()(loss, y.unsqueeze(0))
        loss.backward()

        # 파라미터별 그래디언트 수집
        grads = [p.grad.clone() for p in model.parameters()]
        optimizer.zero_grad()
        per_sample_grads.append(grads)

    # 그래디언트 클리핑 (L2 노름을 max_grad_norm으로 제한)
    clipped_grads = []
    for grads in per_sample_grads:
        total_norm = torch.sqrt(sum(g.norm()**2 for g in grads))
        scale = min(1.0, max_grad_norm / (total_norm + 1e-6))
        clipped_grads.append([g * scale for g in grads])

    # 평균 그래디언트 계산 + 가우시안 노이즈 추가
    sigma = (max_grad_norm / epsilon) * torch.sqrt(2 * torch.log(torch.tensor(1.25 / delta)))
    for i, p in enumerate(model.parameters()):
        avg_grad = torch.stack([cg[i] for cg in clipped_grads]).mean(0)
        noise = torch.normal(0, sigma / batch_size, avg_grad.shape)
        p.grad = avg_grad + noise

    optimizer.step()
```

Google의 TensorFlow Privacy, PyTorch Opacus 라이브러리가 이 DP-SGD를 프로덕션 수준으로 구현하고 있다.

---

## 프라이버시-유용성 트레이드오프

차분 프라이버시의 근본적 제약은 **프라이버시와 유용성(utility)이 상충**한다는 점이다:

- ε이 작을수록 → 더 강한 프라이버시 + 더 많은 노이즈 + 낮은 정확도
- ε이 클수록 → 약한 프라이버시 + 적은 노이즈 + 높은 정확도

### 실전에서의 ε 선택 가이드

| ε 범위 | 의미 | 사용 사례 |
|-------|------|---------|
| ε < 1 | 강한 프라이버시 | 의료 기록, 금융 데이터 |
| 1 ≤ ε ≤ 3 | 중간 | 일반 통계, 집계 분석 |
| ε > 5 | 약한 프라이버시 | 공개 데이터, 낮은 민감도 |

Apple은 iOS에서 이모지·입력 제안 수집에 ε = 1~3을, US Census Bureau는 2020년 인구조사에 ε = 19.61을 사용했다.

---

## 주의사항과 실전 팁

### 1. 전역 민감도를 과소평가하지 말라

쿼리의 민감도를 잘못 계산하면 DP 보장이 무너진다. 무한한 민감도를 가진 쿼리(예: 클리핑 없는 평균)는 DP 메커니즘을 적용하기 전에 반드시 클리핑해야 한다.

### 2. 프라이버시 예산 소진에 주의

같은 데이터셋에 반복적으로 쿼리하면 예산이 누적 소진된다. Rényi 차분 프라이버시(RDP)나 zero-concentrated DP(zCDP) 같은 고급 합성 정리를 사용하면 더 타이트하게 예산을 추적할 수 있다.

### 3. 지역 DP vs 중앙 DP

- **중앙 DP**: 신뢰할 수 있는 데이터 수집자가 원시 데이터를 수집하고 메커니즘을 적용. 더 나은 유용성.
- **지역 DP**: 각 사용자가 자신의 기기에서 노이즈를 추가 후 전송. 신뢰할 수 없는 수집자에도 안전. Apple과 Google이 사용.

### 4. 차분 프라이버시는 만능이 아니다

DP는 통계적 프라이버시를 보장하지만, 소수 집단(minority group)에서는 여전히 정보가 누출될 수 있다. 또한 모델의 훈련 데이터 유출(membership inference attack)을 완전히 막지는 못한다.

---

## 참고 자료

- [The ABCs of Differential Privacy — Esmaeil Alizadeh](https://ealizadeh.com/blog/abc-of-differential-privacy/)
- [TAILOR Handbook: (ε, δ)-Differential Privacy](http://tailor.isti.cnr.it/handbookTAI/Privacy_and_Data_Governance/L3.epsilon_delta_DP.html)
- [Differential Privacy in Machine Learning Survey (2025)](https://arxiv.org/pdf/2506.11687)
- [Google Differential Privacy Library](https://github.com/google/differential-privacy)
