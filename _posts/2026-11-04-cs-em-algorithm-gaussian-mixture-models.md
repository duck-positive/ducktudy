---
layout: post
title: "EM 알고리즘과 가우시안 혼합 모델 완전 정복: 숨겨진 변수를 추론하는 반복 최적화의 수학적 원리"
date: 2026-11-04
categories: [cs, computer-science]
tags: [em-algorithm, gaussian-mixture-model, machine-learning, statistics, clustering, latent-variables, expectation-maximization]
---

## 개요

EM(Expectation-Maximization) 알고리즘은 **잠재 변수(latent variable)**가 존재하는 확률 모델에서 최대 가능도 추정(MLE)을 수행하기 위한 반복적 최적화 알고리즘이다. 1977년 Dempster, Laird, Rubin이 발표한 논문 "Maximum Likelihood from Incomplete Data via the EM Algorithm"에서 체계화된 이 알고리즘은, 직접 최대화가 불가능한 로그 가능도 함수를 E-step과 M-step의 반복을 통해 단조 증가시키는 방식으로 국소 최적해에 도달한다.

가우시안 혼합 모델(Gaussian Mixture Model, GMM)은 EM 알고리즘의 가장 대표적인 적용 사례다. GMM은 여러 가우시안 분포의 볼록 조합으로 복잡한 데이터 분포를 모델링하며, 클러스터링·밀도 추정·이상치 탐지 등 다양한 분야에 활용된다.

---

## 왜 EM 알고리즘이 필요한가

### 불완전 데이터 문제

데이터가 완전하다면 최대 가능도 추정은 단순하다. 예를 들어 단일 가우시안 분포의 평균과 분산은 표본 평균과 표본 분산으로 직접 계산된다.

그러나 데이터가 **여러 가우시안에서 섞여 나왔는데 어느 가우시안에서 왔는지 모른다면** 이야기가 달라진다. 각 데이터 포인트가 어느 클러스터에 속하는지를 나타내는 **잠재 변수 Z**가 존재하지만 관측되지 않는다. 이때 로그 가능도는 다음과 같이 합(sum) 안에 로그가 들어가는 형태가 되어 닫힌 형태(closed-form)의 해가 존재하지 않는다:

```
log p(X | θ) = Σᵢ log [ Σₖ πₖ · N(xᵢ | μₖ, Σₖ) ]
```

여기서 πₖ는 혼합 계수, μₖ와 Σₖ는 k번째 가우시안의 평균과 공분산이다. 합(Σₖ) 안에 로그가 있어 미분이 복잡해진다.

### ELBO와 Jensen 부등식

EM 알고리즘은 이 문제를 **증거 하한(ELBO, Evidence Lower BOund)**을 도입해 해결한다. Jensen 부등식에 의해 로그 가능도의 하한을 구성할 수 있으며, E-step에서는 현재 파라미터로 ELBO를 최대화하는 분포 q(Z)를 계산하고, M-step에서는 그 q(Z)를 고정한 채 파라미터 θ를 최대화한다.

**핵심 보장**: 각 EM 반복에서 로그 가능도 log p(X|θ)는 절대 감소하지 않는다. 단조 증가 성질이 수렴을 보장한다.

---

## EM 알고리즘의 단계

### E-step (Expectation)

현재 파라미터 θ^(t)를 사용해 각 데이터 포인트가 각 클러스터에 속할 **사후 확률(responsibility)**을 계산한다:

```
r(i,k) = P(Z=k | xᵢ, θ^(t))
       = πₖ · N(xᵢ | μₖ, Σₖ) / Σⱼ πⱼ · N(xᵢ | μⱼ, Σⱼ)
```

이 r(i,k)는 "데이터 포인트 xᵢ가 k번째 가우시안에서 왔을 확률"로 해석된다.

### M-step (Maximization)

E-step에서 계산한 responsibility를 사용해 파라미터를 갱신한다:

```
Nₖ = Σᵢ r(i,k)                          # 가중 샘플 수
πₖ^(t+1) = Nₖ / N                        # 혼합 계수 갱신
μₖ^(t+1) = (1/Nₖ) Σᵢ r(i,k) · xᵢ      # 평균 갱신
Σₖ^(t+1) = (1/Nₖ) Σᵢ r(i,k)(xᵢ-μₖ)(xᵢ-μₖ)ᵀ  # 공분산 갱신
```

---

## 가우시안 혼합 모델 구현

### 코드 예제 1: NumPy만으로 GMM EM 구현

```python
import numpy as np
from scipy.stats import multivariate_normal

def gmm_em(X, K, max_iter=100, tol=1e-6, random_state=42):
    """
    GMM을 EM 알고리즘으로 학습.
    X: (N, D) 데이터, K: 클러스터 수
    반환: (pi, mu, sigma, log_likelihoods)
    """
    rng = np.random.default_rng(random_state)
    N, D = X.shape

    # 초기화: K-means++ 스타일 (간단 버전: 랜덤 샘플 선택)
    idx = rng.choice(N, K, replace=False)
    mu = X[idx].copy()                        # (K, D)
    sigma = [np.eye(D) for _ in range(K)]    # (K, D, D)
    pi = np.ones(K) / K                       # (K,)

    log_likelihoods = []

    for iteration in range(max_iter):
        # === E-step: responsibility 계산 ===
        R = np.zeros((N, K))
        for k in range(K):
            R[:, k] = pi[k] * multivariate_normal.pdf(X, mean=mu[k], cov=sigma[k])
        
        # 수치 안정성: 각 행의 합으로 정규화
        row_sums = R.sum(axis=1, keepdims=True)
        row_sums = np.where(row_sums == 0, 1e-300, row_sums)
        R /= row_sums

        # 로그 가능도 계산
        log_likelihood = np.sum(np.log(row_sums + 1e-300))
        log_likelihoods.append(log_likelihood)

        # === M-step: 파라미터 갱신 ===
        Nk = R.sum(axis=0)  # (K,)
        pi = Nk / N

        for k in range(K):
            if Nk[k] < 1e-6:
                continue
            mu[k] = (R[:, k:k+1] * X).sum(axis=0) / Nk[k]
            diff = X - mu[k]
            sigma[k] = (R[:, k:k+1] * diff).T @ diff / Nk[k]
            sigma[k] += 1e-6 * np.eye(D)   # 수치 안정성

        # 수렴 판단
        if iteration > 0 and abs(log_likelihoods[-1] - log_likelihoods[-2]) < tol:
            print(f"수렴 완료: {iteration+1}번째 반복")
            break

    return pi, mu, sigma, log_likelihoods


# --- 실험 ---
np.random.seed(42)
# 세 클러스터 데이터 생성
X1 = np.random.multivariate_normal([0, 0], [[1, 0.5], [0.5, 1]], 200)
X2 = np.random.multivariate_normal([4, 4], [[1, -0.3], [-0.3, 1]], 150)
X3 = np.random.multivariate_normal([0, 5], [[0.5, 0], [0, 2]], 100)
X = np.vstack([X1, X2, X3])

pi, mu, sigma, lls = gmm_em(X, K=3, max_iter=200)

print("혼합 계수 π:", np.round(pi, 3))
print("클러스터 평균:")
for k, m in enumerate(mu):
    print(f"  클러스터 {k}: {np.round(m, 2)}")
print(f"최종 로그 가능도: {lls[-1]:.2f}")
```

**실행 결과 (예시)**:
```
수렴 완료: 47번째 반복
혼합 계수 π: [0.448 0.332 0.220]
클러스터 평균:
  클러스터 0: [-0.02  0.03]
  클러스터 1: [4.02 3.98]
  클러스터 2: [0.01 5.03]
최종 로그 가능도: -1823.45
```

### 코드 예제 2: AIC/BIC로 최적 클러스터 수 선택

```python
import numpy as np
from scipy.stats import multivariate_normal

def compute_bic_aic(X, K, pi, mu, sigma):
    """GMM의 BIC와 AIC를 계산해 모델 복잡도를 평가."""
    N, D = X.shape
    
    # 로그 가능도 계산
    log_lk = 0.0
    for i in range(N):
        total = sum(
            pi[k] * multivariate_normal.pdf(X[i], mean=mu[k], cov=sigma[k])
            for k in range(K)
        )
        log_lk += np.log(total + 1e-300)
    
    # 자유 파라미터 수: K-1 (π) + K*D (μ) + K*D*(D+1)/2 (Σ)
    num_params = (K - 1) + K * D + K * D * (D + 1) // 2
    
    bic = -2 * log_lk + num_params * np.log(N)
    aic = -2 * log_lk + 2 * num_params
    return bic, aic, log_lk

# --- K=1~6에 대해 BIC/AIC 비교 ---
results = []
for K in range(1, 7):
    pi, mu, sigma, lls = gmm_em(X, K=K, max_iter=200, random_state=42)
    bic, aic, ll = compute_bic_aic(X, K, pi, mu, sigma)
    results.append((K, bic, aic, ll))
    print(f"K={K}: BIC={bic:.1f}, AIC={aic:.1f}, logL={ll:.1f}")

best_k_bic = min(results, key=lambda r: r[1])[0]
best_k_aic = min(results, key=lambda r: r[2])[0]
print(f"\nBIC 최적 K={best_k_bic}, AIC 최적 K={best_k_aic}")
```

BIC는 과적합을 더 강하게 패널티하므로 일반적으로 AIC보다 보수적인 K를 선택한다. 위 예제에서는 실제 K=3인 데이터에서 BIC와 AIC 모두 K=3을 선택하는 것을 확인할 수 있다.

---

## GMM vs K-Means: 핵심 차이

| 특성 | K-Means | GMM(EM) |
|------|---------|---------|
| 클러스터 형태 | 구형(원형) | 타원형(임의 공분산) |
| 멤버십 | Hard (0 또는 1) | Soft (확률값) |
| 목적 함수 | 군집 내 거리 합 | 로그 가능도 최대화 |
| 수렴 보장 | 국소 최적 | 로그 가능도 단조 증가 |
| 초기화 민감도 | 높음 | 높음 |
| 계산 복잡도 | O(N·K·iter) | O(N·K·D²·iter) |

K-Means는 사실 GMM에서 공분산을 단위행렬로 고정하고 responsibility를 0/1로 근사한 특수 케이스로 볼 수 있다.

---

## 주의사항과 실전 팁

### 1. 초기화의 중요성

EM은 국소 최적해로 수렴한다. 초기화가 나쁘면 나쁜 해를 반환할 수 있다.

**권장 전략**: K-Means++로 초기 평균을 설정한 뒤 EM을 실행. scikit-learn의 `GaussianMixture`는 `n_init` 파라미터로 여러 번 초기화 후 최선의 결과를 선택한다.

### 2. 수치 불안정성

공분산 행렬이 특이행렬(singular)에 가까워지면 가우시안 PDF가 발산할 수 있다. 공분산 행렬에 소량의 단위행렬 (`reg_covar * I`)을 더해 양정치(positive definite)를 보장한다.

### 3. 빈 클러스터 처리

특정 클러스터에 responsibility가 모두 0에 가까워지면 Nk → 0이 되어 파라미터 갱신이 불가능하다. 이 경우 해당 클러스터를 재초기화하거나 K를 줄이는 전략이 필요하다.

### 4. 로그-합-지수(log-sum-exp) 트릭

E-step에서 `Σₖ πₖ · N(xᵢ|μₖ, Σₖ)` 계산 시 개별 항이 언더플로우(underflow)로 0이 될 수 있다. 로그 공간에서 계산하고 `scipy.special.logsumexp`를 활용해 수치 안정성을 확보한다.

### 5. 수렴 판단 기준

로그 가능도 변화량이 `tol` (기본값 1e-6) 미만이면 수렴으로 판단한다. 그러나 진동이 발생할 수 있으므로 최소 반복 횟수를 설정하는 것이 좋다.

---

## 활용 사례

- **이미지 분할**: 픽셀의 색상 분포를 GMM으로 모델링해 배경/전경 분리
- **음성 인식**: 음소(phoneme) 모델링에 GMM 사용 (HMM과 결합)
- **이상치 탐지**: 낮은 사후 확률을 갖는 포인트를 이상치로 판단
- **밀도 추정**: 커널 밀도 추정(KDE) 대비 파라메트릭 접근의 장점

---

## 참고 자료
- [scikit-learn: Gaussian Mixture Models](https://scikit-learn.org/stable/modules/mixture.html)
- [Wikipedia: Expectation-maximization algorithm](https://en.wikipedia.org/wiki/Expectation%E2%80%93maximization_algorithm)
- [Dempster, Laird, Rubin (1977): Maximum Likelihood from Incomplete Data via the EM Algorithm — JSTOR](https://www.jstor.org/stable/2984875)
- [Bishop (2006): Pattern Recognition and Machine Learning — Springer](https://www.springer.com/gp/book/9780387310732)
