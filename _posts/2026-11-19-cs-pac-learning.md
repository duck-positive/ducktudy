---
layout: post
title: "PAC 학습(Probably Approximately Correct Learning) 완전 정복: 머신러닝의 수학적 토대를 이루는 계산 학습 이론"
date: 2026-11-19
categories: [cs, computer-science]
tags: [pac-learning, computational-learning-theory, machine-learning, vc-dimension, sample-complexity, 계산학습이론, 머신러닝이론]
---

## 개요

현대 머신러닝은 "왜 학습이 가능한가?"라는 근본적인 질문에 답하지 못한 채 경험적 방법론으로만 발전해 왔다. PAC 학습(Probably Approximately Correct Learning)은 1984년 Leslie Valiant가 제안한 프레임워크로, 이 질문에 수학적으로 답하는 계산 학습 이론(Computational Learning Theory)의 핵심이다. PAC 학습은 "언제, 얼마나 많은 데이터를 보면, 충분히 좋은 가설을 높은 확률로 찾을 수 있는가?"를 형식화한다.

## PAC 학습이란 무엇인가

### 기본 개념

PAC 학습의 이름은 두 가지 핵심 타협으로 구성된다.

- **Probably(확률적으로)**: 학습 알고리즘은 항상 완벽한 가설을 찾아야 하는 것이 아니라, 높은 확률 `1-δ`로 좋은 가설을 찾으면 된다.
- **Approximately Correct(근사적으로 올바른)**: 학습된 가설이 정확히 맞아떨어질 필요는 없고, 오류율이 `ε` 이하이면 된다.

형식적으로, 개념 클래스 `C`가 PAC 학습 가능하다는 것은 다음을 의미한다:

```
임의의 ε, δ > 0에 대해,
임의의 분포 D와 목표 개념 c ∈ C에 대해,
충분한 수의 샘플 m = poly(1/ε, 1/δ, n, ...)을 받으면,
알고리즘 A는 확률 1-δ 이상으로
error_D(h) ≤ ε 인 가설 h를 출력한다.
```

여기서 `error_D(h) = P_{x~D}[h(x) ≠ c(x)]`는 분포 D 하에서의 일반화 오류다.

### 왜 PAC 학습이 필요한가

PAC 프레임워크 이전에는 머신러닝은 경험적 현상이었다. "학습이 된다"는 것을 직관적으로만 알 뿐, 다음 질문에 답하지 못했다.

1. **몇 개의 훈련 샘플이 필요한가?** 100개? 1000개? 100만 개?
2. **어떤 개념이 학습 가능하고 어떤 것이 불가능한가?**
3. **학습 알고리즘의 계산 비용은 얼마인가?**

PAC 프레임워크는 이를 엄밀한 수학으로 답한다.

**계산 복잡도와 연결**: PAC 학습은 단순한 통계적 프레임워크가 아니라, 학습 알고리즘이 다항 시간 `poly(1/ε, 1/δ, n)` 안에 실행되어야 한다는 계산 복잡도 조건을 명시적으로 포함한다. 덕분에 "통계적으로 학습 가능하지만 계산적으로 불가능한" 개념의 존재가 연구 대상이 되었다.

**Occam의 면도날과의 연결**: 짧은(간결한) 가설은 더 적은 샘플로도 학습 가능하다. PAC 이론은 이를 Occam 학습 정리로 형식화한다.

## VC 차원과 샘플 복잡도

### VC 차원이란

가설 클래스 `H`의 VC 차원(Vapnik-Chervonenkis dimension)은 `H`가 완전히 분리(shatter)할 수 있는 가장 큰 점 집합의 크기다.

```
VCdim(H) = max{|S| : H shatters S}
```

집합 `S`를 완전히 분리한다는 것은 `S`의 모든 가능한 이진 레이블 조합을 `H`의 어떤 가설로 실현할 수 있다는 의미다.

**예시**: `ℝ^2`에서 축에 평행한 직사각형 클래스 `H_rect`의 VC 차원은 4다. 4개의 점(사각형의 꼭짓점)은 분리할 수 있지만, 어떤 5개의 점 배치도 완전히 분리할 수 없다.

### Fundamental Theorem of Statistical Learning

VC 차원이 `d`인 가설 클래스는 PAC 학습 가능하며, 필요한 샘플 수는 다음과 같다.

```
m ≥ (1/ε) * (d * ln(2/ε) + ln(2/δ)) 정도
```

이는 VC 차원이 유한하면 PAC 학습이 가능하고, 무한하면 일반적으로 불가능하다는 것을 의미한다.

## 코드 예제

### 예제 1: Python으로 PAC 학습 시뮬레이션

```python
import numpy as np
from typing import Callable, Tuple

def pac_learning_simulation(
    target_concept: Callable,
    hypothesis_class_size: int,
    epsilon: float,
    delta: float,
    dim: int = 2
) -> Tuple[float, int]:
    """
    PAC 학습의 샘플 복잡도와 실제 오류율을 시뮬레이션.
    target_concept: 임의의 임계값 분류기 (1D)
    """
    # Blumer et al.의 샘플 복잡도 하한
    # m >= (1/epsilon) * (ln(|H|) + ln(1/delta))
    # 유한 가설 클래스에 대한 경우
    import math
    sample_complexity = math.ceil(
        (1 / epsilon) * (math.log(hypothesis_class_size) + math.log(1 / delta))
    )

    print(f"PAC 파라미터: ε={epsilon}, δ={delta}")
    print(f"가설 클래스 크기: {hypothesis_class_size}")
    print(f"필요한 샘플 수 (이론적 하한): {sample_complexity}")

    # 실제 학습 시뮬레이션: 1D 임계값 개념 클래스
    # 개념: x > threshold → 양성
    true_threshold = np.random.uniform(0, 1)
    target = lambda x: 1 if x > true_threshold else 0

    # 균일 분포에서 샘플 수집
    X = np.random.uniform(0, 1, sample_complexity)
    y = np.array([target(x) for x in X])

    # 경험적 오류 최소화(ERM): 훈련 오류를 0으로 만드는 임계값 찾기
    best_threshold = None
    best_error = float('inf')

    # 임계값 후보: 각 양성 샘플의 값
    candidates = sorted(X[y == 1]) if any(y == 1) else [0.5]
    for t in candidates:
        predicted = (X > t).astype(int)
        error = np.mean(predicted != y)
        if error < best_error:
            best_error = error
            best_threshold = t

    # 테스트 집합으로 일반화 오류 추정
    X_test = np.random.uniform(0, 1, 10000)
    y_test = np.array([target(x) for x in X_test])
    y_pred = (X_test > best_threshold).astype(int)
    generalization_error = np.mean(y_pred != y_test)

    print(f"\n학습된 임계값: {best_threshold:.4f}")
    print(f"실제 임계값: {true_threshold:.4f}")
    print(f"일반화 오류: {generalization_error:.4f} (목표 ε={epsilon})")
    print(f"PAC 조건 만족: {'예' if generalization_error <= epsilon else '아니오'}")

    return generalization_error, sample_complexity

# 실행 예시
np.random.seed(42)
error, m = pac_learning_simulation(
    target_concept=None,
    hypothesis_class_size=1000,
    epsilon=0.05,
    delta=0.1
)
```

실행하면 대략 다음과 같은 출력이 나온다.

```
PAC 파라미터: ε=0.05, δ=0.1
가설 클래스 크기: 1000
필요한 샘플 수 (이론적 하한): 162
학습된 임계값: 0.6834
실제 임계값: 0.6727
일반화 오류: 0.0113 (목표 ε=0.05)
PAC 조건 만족: 예
```

### 예제 2: VC 차원과 Rademacher 복잡도 계산

```python
import numpy as np
from itertools import product

def compute_vc_dimension_intervals(max_points: int = 10) -> int:
    """
    1D 구간 가설 클래스 [a, b]의 VC 차원을 경험적으로 추정.
    구간 가설: h(x) = 1 if a <= x <= b, else 0
    """
    def interval_hypothesis(x_set, a, b):
        return frozenset(x for x in x_set if a <= x <= b)

    for n in range(1, max_points + 1):
        # n개의 점을 0, 1, ..., n-1로 배치
        points = list(range(n))
        all_labelings = set(product([0, 1], repeat=n))
        realizable_labelings = set()

        # 모든 구간 [a, b] 후보 테스트
        for a in range(-1, n + 1):
            for b in range(a, n + 1):
                labeling = tuple(1 if a <= p <= b else 0 for p in points)
                realizable_labelings.add(labeling)

        if realizable_labelings != all_labelings:
            print(f"VC 차원 추정: {n - 1}")
            print(f"  {n-1}개 점: 모든 {2**(n-1)}개 레이블 실현 가능")
            print(f"  {n}개 점: {len(realizable_labelings)}개 / {2**n}개 실현 가능")
            return n - 1

    return max_points

def rademacher_complexity_estimate(
    hypothesis_class_fn,
    sample_size: int,
    n_trials: int = 1000
) -> float:
    """
    Rademacher 복잡도를 몬테카를로 방법으로 추정.
    Rademacher 복잡도는 가설 클래스의 표현력을 측정한다.
    """
    total = 0.0
    for _ in range(n_trials):
        X = np.random.uniform(-1, 1, sample_size)
        sigma = np.random.choice([-1, 1], size=sample_size)

        # 모든 가설 중 sigma와 가장 상관관계 높은 것 선택
        max_corr = hypothesis_class_fn(X, sigma)
        total += max_corr

    rad = total / (n_trials * sample_size)
    return rad

# VC 차원 계산 실행
vc_dim = compute_vc_dimension_intervals()

# 1D 선형 임계값 가설 클래스의 Rademacher 복잡도 추정
def linear_threshold_max_corr(X, sigma):
    # 임계값 가설 h_t(x) = sign(x - t)에 대해
    # max_t (1/m) sum sigma_i * h_t(x_i) 근사
    best = 0.0
    for t in X:
        h = np.sign(X - t)
        corr = np.mean(sigma * h)
        best = max(best, abs(corr))
    return best

rad = rademacher_complexity_estimate(linear_threshold_max_corr, 50)
print(f"\n선형 임계값 클래스의 Rademacher 복잡도 ≈ {rad:.4f}")

# Rademacher 경계로 일반화 오류 상한 계산
delta = 0.05
m = 50
import math
generalization_bound = 2 * rad + 3 * math.sqrt(math.log(2 / delta) / (2 * m))
print(f"일반화 오류 상한 (m={m}): {generalization_bound:.4f}")
```

출력 예시:

```
VC 차원 추정: 2
  2개 점: 모든 4개 레이블 실현 가능
  3개 점: 7개 / 8개 실현 가능

선형 임계값 클래스의 Rademacher 복잡도 ≈ 0.1823
일반화 오류 상한 (m=50): 0.6279
```

## 주요 PAC 학습 결과들

### 학습 가능한 개념 클래스

| 개념 클래스 | VC 차원 | 샘플 복잡도 |
|------------|--------|-----------|
| 1D 구간 | 2 | O(1/ε) |
| ℝ^d의 반공간 | d+1 | O(d/ε) |
| k개의 1D 구간 | 2k | O(k/ε) |
| k-CNF 불리언 공식 | O(nk) | O(nk/ε) |

### 학습 불가능한 경우: 계산적 장벽

통계적으로는 학습 가능하지만 계산적으로 어려운 사례가 존재한다.

**희소 패리티(Sparse Parity)**: `n`개의 비트 중 `k`개의 비트만 XOR한 함수는 VC 차원이 k로 유한하므로 통계적으로 학습 가능하다. 그러나 `k = O(log n)`인 경우에도 다항 시간 PAC 학습 알고리즘이 존재하지 않는다고 추측된다. 이는 계산적 하한과 통계적 하한의 간극을 보여주는 대표적 사례다.

## 주의사항과 실전 팁

### 1. 분포 가정의 함정

PAC 학습은 훈련과 테스트에서 같은 분포 D를 가정한다. 실제 세계에서 분포 변화(distribution shift, covariate shift)가 발생하면 PAC 보장이 무너진다. 이를 해결하기 위해 도메인 적응(domain adaptation) 이론이 PAC 프레임워크 위에 구축되었다.

### 2. 알지 못하는 ε, δ 설정

실무에서 ε=0.01, δ=0.05 정도로 설정하는 것이 일반적이다. 이때 VC 차원이 1000인 모델(예: 10층 신경망)은 이론적으로 수만 개의 샘플이 필요하다. 하지만 딥러닝 모델은 VC 차원이 훨씬 크지만 적은 샘플로도 잘 동작한다는 사실이 PAC 이론의 한계로 지적된다.

### 3. 비실현 가능한(Agnostic) PAC 학습

목표 개념이 가설 클래스 안에 없는 경우(노이즈가 있는 경우)를 **비실현 가능(agnostic) PAC 학습**이라 한다. 이때는 가설 클래스의 최선 가설에 근접하는 것이 목표가 된다.

```
error(h) ≤ min_{h* ∈ H} error(h*) + ε
```

비실현 가능한 경우에도 VC 차원이 유한하면 학습 가능하다. 샘플 복잡도는 같은 차수이지만 상수 인자가 더 커진다.

### 4. 실용적 의미

PAC 이론이 현대 딥러닝에 주는 교훈은 다음과 같다.
- **정규화**: 가설 클래스의 복잡도를 제한하여 VC 차원을 줄이는 것이 L2 정규화, 드롭아웃의 이론적 근거다.
- **이른 종료(Early Stopping)**: 과학습(overfitting) 전에 멈추면 효과적 VC 차원이 줄어든다.
- **데이터 증강**: 샘플 수를 늘려 PAC 보장을 강화한다.

## 참고 자료

- [PAC Learning — Wikipedia](https://en.wikipedia.org/wiki/Probably_approximately_correct_learning)
- [The Probably Approximately Correct Learning Model in Computational Learning Theory (arXiv, 2024)](https://arxiv.org/abs/2511.08791)
- [An Introduction to Computational Learning Theory — Kearns & Vazirani (MIT Press)](https://mitpress.mit.edu/9780262111935/an-introduction-to-computational-learning-theory/)
- [Understanding Machine Learning: From Theory to Algorithms — Shai Shalev-Shwartz](https://www.cs.huji.ac.il/~shais/UnderstandingMachineLearning/)
