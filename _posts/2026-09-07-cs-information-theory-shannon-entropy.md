---
layout: post
title: "정보 이론 완전 정복: Shannon 엔트로피부터 KL 발산, 채널 용량까지"
date: 2026-09-07
categories: [cs, computer-science]
tags: [information-theory, shannon-entropy, kl-divergence, cross-entropy, mutual-information, machine-learning]
---

1948년 Claude Shannon이 "A Mathematical Theory of Communication"을 발표했을 때, 그것은 단순한 통신 이론을 넘어 **정보의 본질을 수학적으로 정의한 혁명적 업적**이었다. Shannon의 정보 이론은 오늘날 데이터 압축, 암호화, 머신러닝의 손실 함수, 의사결정 트리, 강화학습에 이르기까지 컴퓨터 과학 전반에 깊숙이 침투해 있다. 이 아티클에서는 Shannon 엔트로피부터 KL 발산, 상호정보량, 채널 용량까지 정보 이론의 핵심 개념을 코드와 함께 완전히 정복한다.

---

## 정보란 무엇인가

Shannon의 핵심 통찰은 **"정보량은 놀라움의 양"**이라는 것이다. 확률 p로 발생하는 사건이 발생했을 때 얻는 정보량은 다음과 같이 정의된다.

```
I(x) = -log₂(p(x))  [단위: bits]
```

- 확실한 사건(p = 1): I = 0 bits — 이미 알고 있으므로 정보가 없다
- 드문 사건(p = 0.01): I = log₂(100) ≈ 6.64 bits — 매우 놀랍고 정보가 많다

자연로그(ln)를 사용하면 단위가 **nats(내츠)**가 되고, log₁₀이면 **hartleys(하틀리)**가 된다. 머신러닝에서는 자연로그를 주로 쓴다.

---

## Shannon 엔트로피

**엔트로피(Entropy)**는 확률 분포의 **평균 정보량**, 즉 **불확실성의 측도**다.

$$H(X) = -\sum_{x \in \mathcal{X}} p(x) \log_2 p(x)$$

이것을 기댓값(expectation)으로 표현하면:

$$H(X) = \mathbb{E}[-\log_2 p(X)]$$

### 핵심 성질

1. **비음수성**: H(X) ≥ 0. 불확실성은 항상 0 이상이다.
2. **균일분포 최대**: 모든 결과가 같은 확률을 가질 때 엔트로피가 최대이다.
   - 동전 던지기(2면): H_max = 1 bit
   - 주사위(6면): H_max = log₂(6) ≈ 2.585 bits
3. **결정적 분포 최소**: p(x₀) = 1인 결과가 하나뿐이면 H = 0.

### 왜 -log p인가?
엔트로피가 만족해야 할 세 가지 공리:
1. 균일분포에서 결과 수가 많을수록 불확실성이 커야 한다
2. 독립 사건의 결합 엔트로피는 각 엔트로피의 합이어야 한다
3. 연속적이어야 한다

-log p는 이 세 공리를 모두 만족하는 **유일한** 함수임이 증명된다.

---

## 교차 엔트로피(Cross-Entropy)

실제 분포 p를 추정 분포 q로 인코딩할 때의 **평균 코드 길이**다.

$$H(p, q) = -\sum_{x} p(x) \log q(x)$$

머신러닝에서 분류 문제의 손실 함수로 널리 사용된다. 진짜 레이블 p와 모델 예측 q의 교차 엔트로피를 최소화하는 것이 목표다.

```
H(p, q) = H(p) + D_KL(p || q)
```

즉 교차 엔트로피 = 실제 엔트로피 + KL 발산. 실제 엔트로피는 고정이므로 교차 엔트로피 최소화 ≡ KL 발산 최소화다.

---

## KL 발산(Kullback-Leibler Divergence)

두 확률분포 p, q의 **상대 엔트로피**, 즉 p를 q로 근사할 때의 정보 손실이다.

$$D_{KL}(p \| q) = \sum_{x} p(x) \log \frac{p(x)}{q(x)}$$

### 핵심 특성
- **비음수**: D_KL ≥ 0 (Gibbs 부등식)
- **비대칭**: D_KL(p||q) ≠ D_KL(q||p) → 진짜 거리(metric)가 아니다
- p와 q가 같으면 D_KL = 0

KL 발산의 비대칭성은 중요한 실용적 차이를 낳는다:
- **Forward KL (D_KL(p||q))**: p가 0이 아닌 모든 곳에서 q도 0이 아니어야 한다. 분포 전체를 커버하려 한다 → **mean-seeking**
- **Reverse KL (D_KL(q||p))**: q가 분포의 일부 모드(mode)에 집중한다 → **mode-seeking**

VAE(변분 오토인코더)는 Forward KL을, GAN은 암묵적으로 Reverse KL과 관련된 목적함수를 최적화한다.

---

## 상호정보량(Mutual Information)

두 확률변수 X, Y가 **서로에 대해 얼마나 많은 정보를 공유하는지** 측정한다.

$$I(X;Y) = \sum_{x,y} p(x,y) \log \frac{p(x,y)}{p(x)p(y)}$$

KL 발산으로 표현하면:
$$I(X;Y) = D_{KL}(p(X,Y) \| p(X)p(Y))$$

즉, 결합분포가 독립 곱분포에서 얼마나 벗어났는지를 측정한다.

엔트로피로 표현하면:
$$I(X;Y) = H(X) - H(X|Y) = H(Y) - H(Y|X) = H(X) + H(Y) - H(X,Y)$$

### 응용
- **특성 선택(Feature Selection)**: 레이블과 상호정보량이 높은 특성을 선택
- **ICA(독립 성분 분석)**: 상호정보량을 최소화해 독립 성분을 찾음
- **정보 병목(Information Bottleneck)**: 최소 정보로 최대 예측 성능 달성

---

## 구현 예제 1: 엔트로피, KL 발산, 상호정보량 직접 구현 (Python)

```python
import numpy as np
from scipy.stats import entropy as scipy_entropy
from collections import Counter

def shannon_entropy(probs, base=2):
    """이산 확률분포의 Shannon 엔트로피 계산."""
    probs = np.array(probs, dtype=float)
    # 0인 확률은 제외 (0 * log(0) = 0으로 정의)
    probs = probs[probs > 0]
    if base == 2:
        return -np.sum(probs * np.log2(probs))
    return -np.sum(probs * np.log(probs))  # nats

def entropy_from_text(text):
    """텍스트의 문자 빈도 기반 엔트로피 계산."""
    counts = Counter(text)
    total = sum(counts.values())
    probs = [c / total for c in counts.values()]
    return shannon_entropy(probs)

def kl_divergence(p, q, eps=1e-10):
    """KL 발산 D_KL(p || q) 계산."""
    p = np.array(p, dtype=float)
    q = np.array(q, dtype=float)
    # 수치 안정성을 위한 epsilon 추가
    q = np.maximum(q, eps)
    mask = p > 0
    return np.sum(p[mask] * np.log(p[mask] / q[mask]))

def cross_entropy(p, q, eps=1e-10):
    """교차 엔트로피 H(p, q) 계산."""
    p = np.array(p, dtype=float)
    q = np.array(q, dtype=float)
    q = np.maximum(q, eps)
    mask = p > 0
    return -np.sum(p[mask] * np.log(q[mask]))

def mutual_information_discrete(joint_prob):
    """이산 결합분포로부터 상호정보량 계산.
    joint_prob: 2D array, joint_prob[i][j] = P(X=i, Y=j)
    """
    joint = np.array(joint_prob, dtype=float)
    px = joint.sum(axis=1)  # P(X)
    py = joint.sum(axis=0)  # P(Y)
    
    mi = 0.0
    for i in range(joint.shape[0]):
        for j in range(joint.shape[1]):
            if joint[i, j] > 0 and px[i] > 0 and py[j] > 0:
                mi += joint[i, j] * np.log2(joint[i, j] / (px[i] * py[j]))
    return mi

# ── 실험 1: 여러 분포의 엔트로피 비교 ──
print("=== 엔트로피 비교 ===")
uniform_4  = [0.25, 0.25, 0.25, 0.25]  # 균일분포
skewed     = [0.7, 0.1, 0.1, 0.1]      # 편향 분포
degenerate = [1.0, 0.0, 0.0, 0.0]      # 결정적 분포

print(f"균일분포:   H = {shannon_entropy(uniform_4):.4f} bits")
print(f"편향분포:   H = {shannon_entropy(skewed):.4f} bits")
print(f"결정적분포: H = {shannon_entropy(degenerate):.4f} bits")

# ── 실험 2: KL 발산의 비대칭성 ──
print("\n=== KL 발산 비대칭성 ===")
p = [0.9, 0.1]
q = [0.5, 0.5]
print(f"D_KL(p||q) = {kl_divergence(p, q):.4f} nats")
print(f"D_KL(q||p) = {kl_divergence(q, p):.4f} nats")
print(f"H(p,q) = D_KL(p||q) + H(p): "
      f"{kl_divergence(p,q):.4f} + {shannon_entropy(p, base=None):.4f}"
      f" = {cross_entropy(p, q):.4f}")

# ── 실험 3: 텍스트 엔트로피 ──
print("\n=== 텍스트 엔트로피 ===")
texts = {
    "영어 문장": "the quick brown fox jumps over the lazy dog",
    "반복 문자": "aaaaaaaaaa",
    "무작위":    "xkzqjvmwnprtybhfdoueaicslg",
}
for name, text in texts.items():
    print(f"{name}: H = {entropy_from_text(text):.4f} bits/char")

# ── 실험 4: 상호정보량 ──
print("\n=== 상호정보량 ===")
# X: 날씨(맑음, 비), Y: 우산 여부(있음, 없음)
# 높은 상관관계
joint_high = [[0.40, 0.05],   # P(맑음, 우산) and P(맑음, 없음)
              [0.05, 0.50]]   # P(비, 우산) and P(비, 없음)
# 독립 (상관관계 없음)
joint_indep = [[0.225, 0.225],
               [0.275, 0.275]]

print(f"높은 상관 MI: {mutual_information_discrete(joint_high):.4f} bits")
print(f"독립     MI: {mutual_information_discrete(joint_indep):.6f} bits")
```

---

## 구현 예제 2: 정보 이론으로 의사결정 트리 분할 기준 구현

```python
import numpy as np
from typing import List, Tuple

def information_gain(y: np.ndarray, y_left: np.ndarray, y_right: np.ndarray) -> float:
    """Information Gain = H(parent) - weighted H(children)"""
    def entropy(labels):
        if len(labels) == 0:
            return 0.0
        _, counts = np.unique(labels, return_counts=True)
        probs = counts / len(labels)
        return -np.sum(probs * np.log2(probs + 1e-12))
    
    n = len(y)
    n_l, n_r = len(y_left), len(y_right)
    
    h_parent = entropy(y)
    h_children = (n_l / n) * entropy(y_left) + (n_r / n) * entropy(y_right)
    return h_parent - h_children

def best_split(X: np.ndarray, y: np.ndarray) -> Tuple[int, float, float]:
    """모든 피처와 임계값 조합 중 최대 Information Gain을 찾는다."""
    best_ig, best_feature, best_threshold = -np.inf, None, None
    n_features = X.shape[1]
    
    for feature_idx in range(n_features):
        values = np.sort(np.unique(X[:, feature_idx]))
        thresholds = (values[:-1] + values[1:]) / 2  # 인접 값의 중간점
        
        for threshold in thresholds:
            left_mask = X[:, feature_idx] <= threshold
            right_mask = ~left_mask
            
            y_left, y_right = y[left_mask], y[right_mask]
            if len(y_left) == 0 or len(y_right) == 0:
                continue
            
            ig = information_gain(y, y_left, y_right)
            if ig > best_ig:
                best_ig = ig
                best_feature = feature_idx
                best_threshold = threshold
    
    return best_feature, best_threshold, best_ig

# ── 실험: 붓꽃 데이터셋에서 최적 분할 찾기 ──
from sklearn.datasets import load_iris
iris = load_iris()
X, y = iris.data, iris.target

feat, thresh, ig = best_split(X, y)
print(f"최적 분할: 피처={iris.feature_names[feat]}")
print(f"임계값={thresh:.3f}, Information Gain={ig:.4f} bits")

# 각 피처별 최적 IG 비교
print("\n피처별 최대 Information Gain:")
for i, fname in enumerate(iris.feature_names):
    _, t, ig_i = best_split(X[:, i:i+1], y)
    print(f"  {fname:<30} IG={ig_i:.4f}")

# Gini vs Entropy 비교
def gini_impurity(y):
    _, counts = np.unique(y, return_counts=True)
    probs = counts / len(y)
    return 1 - np.sum(probs ** 2)

print("\n부모 노드의 엔트로피:", round(-sum(
    (np.sum(y==c)/len(y)) * np.log2(np.sum(y==c)/len(y))
    for c in np.unique(y)), 4), "bits")
print("부모 노드의 Gini:   ", round(gini_impurity(y), 4))
```

---

## 채널 용량(Channel Capacity)

Shannon의 두 번째 위대한 기여는 **채널 용량 정리**다. 잡음이 있는 채널에서도 오류 없이 전송 가능한 최대 정보율이 존재한다.

$$C = \max_{p(x)} I(X; Y) \text{ [bits/channel use]}$$

AWGN(Additive White Gaussian Noise) 채널에서의 Shannon 용량 공식:

$$C = B \log_2\left(1 + \frac{S}{N}\right) \text{ [bits/s]}$$

- B: 대역폭(Hz)
- S/N: 신호 대 잡음비(SNR)

이것이 **Shannon-Hartley 정리**이며, Wi-Fi·LTE·5G 등 모든 현대 통신 기술의 이론적 상한이다.

---

## 주의사항과 팁

**1. 0 확률 처리**
log(0)은 정의되지 않는다. 코드에서는 항상 `eps = 1e-10` 같은 작은 값을 더하거나, 0인 항목을 아예 제외하라. numpy에서는 `np.where(p > 0, -p * np.log2(p), 0).sum()` 패턴을 활용하자.

**2. KL 발산은 거리가 아니다**
비대칭이므로 두 분포 사이의 "거리"로 사용할 수 없다. **Jensen-Shannon 발산(JSD)**는 대칭화된 KL 발산으로, 진짜 거리의 일부 성질을 만족한다: JSD(p||q) = (KL(p||M) + KL(q||M)) / 2 (M = (p+q)/2).

**3. 교차 엔트로피와 Log Loss**
머신러닝에서 이진 분류의 "Binary Cross-Entropy Loss"는 단순히 두 클래스(0, 1)에 대한 교차 엔트로피다. 손실 함수로 쓸 때는 배치 평균을 구한다.

**4. 엔트로피 단위 일관성**
log₂ → bits, ln → nats, log₁₀ → hartleys. 특히 PyTorch의 `F.cross_entropy`는 자연로그를 쓴다. 외부 공식과 비교할 때 단위 불일치로 오해하지 않도록 주의하라.

**5. 조건부 엔트로피 H(Y|X) ≠ H(Y - X)**
H(Y|X) = H(X, Y) - H(X)이며, 이것은 데이터 압축에서 X를 알고 있을 때 Y를 인코딩하는 데 필요한 추가 비트 수를 의미한다.

---

## 참고 자료

- [Information Theory: A Tutorial Introduction (arxiv)](https://arxiv.org/pdf/1802.05968)
- [Entropy and Information Theory — Stanford University (R. Gray)](https://ee.stanford.edu/~gray/it.pdf)
- [From Shannon to Modern AI — Machine Learning Mastery](https://machinelearningmastery.com/from-shannon-to-modern-ai-a-complete-information-theory-guide-for-machine-learning/)
- [A Brief Introduction to Shannon's Information Theory (arxiv)](https://arxiv.org/pdf/1612.09316)
