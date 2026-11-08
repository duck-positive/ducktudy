---
layout: post
title: "베이즈 네트워크(Bayesian Network) 완전 정복: 조건부 독립성과 믿음 전파로 불확실성을 추론하는 법"
date: 2026-11-08
categories: [cs, computer-science]
tags: [bayesian-network, probabilistic-inference, belief-propagation, pgm, machine-learning, statistics, ai]
---

## 1. 베이즈 네트워크란 무엇인가?

베이즈 네트워크(Bayesian Network, BN)는 확률 변수들 사이의 조건부 독립성 관계를 방향 비순환 그래프(DAG, Directed Acyclic Graph)로 표현하는 확률적 그래픽 모델(Probabilistic Graphical Model)이다. 1988년 Judea Pearl이 체계화했으며, Pearl은 이 연구로 2011년 튜링상을 수상했다.

베이즈 네트워크가 강력한 이유는 **결합 확률 분포(joint probability distribution)를 조건부 독립성을 이용해 압축하여 표현**하기 때문이다.

예를 들어 N개의 이진 확률 변수가 있다면 완전 결합 분포는 `2^N - 1`개의 파라미터를 필요로 한다. 그러나 조건부 독립성이 존재하면 각 변수는 부모 노드의 값만 알면 조건부 확률표(CPT, Conditional Probability Table)를 구성할 수 있고, 파라미터 수를 극적으로 줄일 수 있다.

**수학적 정의:**

베이즈 네트워크 `B = (G, P)`에서 `G`는 DAG이고, 각 노드 `Xi`에 대해 조건부 확률 `P(Xi | parents(Xi))`가 정의된다. 결합 분포는 아래와 같이 분해된다:

```
P(X1, X2, ..., Xn) = Π P(Xi | parents(Xi))
```

이 분해 덕분에 추론과 학습이 효율적으로 수행된다.

## 2. 왜 베이즈 네트워크가 필요한가?

### 불확실성의 명시적 모델링

의료 진단을 예로 들자. 환자가 기침을 한다고 해서 반드시 폐암인 것은 아니다. 감기, 기관지염, 흡연 등 다양한 원인이 있다. 베이즈 네트워크는 이런 불확실한 인과 관계를 확률로 명시적으로 표현한다.

### 인과 추론 (Causal Reasoning)

관측된 증거(evidence)가 주어졌을 때 원인의 사후 확률을 계산하는 **진단 추론(diagnostic reasoning)**, 원인이 주어졌을 때 결과의 확률을 계산하는 **예측 추론(predictive reasoning)**, 그리고 간접적 연결을 통한 **설명적 추론(abductive reasoning)**을 모두 하나의 프레임워크 안에서 처리할 수 있다.

### 소규모 데이터에서의 강점

딥러닝은 대규모 데이터에서 뛰어나지만 의료, 법률, 과학 도메인처럼 데이터가 적고 도메인 지식이 풍부한 경우에는 베이즈 네트워크가 더 효과적이다. 도메인 전문가의 지식을 그래프 구조와 CPT로 직접 인코딩할 수 있기 때문이다.

## 3. 조건부 독립성과 D-분리

베이즈 네트워크 추론의 핵심은 **D-분리(D-separation)** 개념이다. 세 가지 연결 패턴을 이해해야 한다.

**패턴 1: Chain (인과 체인)**
```
X → Z → Y
```
Z가 관측되면 X와 Y는 조건부 독립이다. Z가 모든 정보를 매개한다.

**패턴 2: Fork (공통 원인)**
```
X ← Z → Y
```
Z(공통 원인)가 관측되면 X와 Y는 조건부 독립이다. 예: 흡연이 폐암과 심장병을 모두 유발하는 경우, 흡연을 알면 폐암과 심장병 사이의 연관이 사라진다.

**패턴 3: Collider (공통 결과)**
```
X → Z ← Y
```
Z가 관측되지 않으면 X와 Y는 독립이다. 그러나 Z(또는 그 자손)가 관측되면 X와 Y는 종속이 된다! 이것이 **Berkson's Paradox**이다.

## 4. 믿음 전파(Belief Propagation) 알고리즘

트리 구조 베이즈 네트워크에서 정확한 추론을 O(N)에 수행하는 알고리즘이다.

각 노드는 이웃에게 **메시지(message)**를 전달한다. 메시지는 두 종류이다:
- `π` 메시지: 부모에서 자식 방향으로 전달되는 "예측" 메시지
- `λ` 메시지: 자식에서 부모 방향으로 전달되는 "우도" 메시지

**Python으로 구현한 간단한 Belief Propagation:**

```python
import numpy as np
from collections import defaultdict

class BayesianNode:
    def __init__(self, name, states, parents=None):
        self.name = name
        self.states = states          # ['T', 'F'] 등
        self.parents = parents or []
        self.cpt = {}                 # Conditional Probability Table
        self.pi_messages = {}         # 부모로부터 받은 메시지
        self.lambda_messages = {}     # 자식으로부터 받은 메시지

    def set_cpt(self, table):
        """CPT 설정: {(부모_값들): [P(X=s0), P(X=s1), ...]}"""
        self.cpt = table

    def compute_belief(self, evidence=None):
        """
        관측 증거가 없는 경우 결합 메시지로 신념(belief) 계산.
        단순화를 위해 단일 노드, 부모 없는 케이스.
        """
        if evidence is not None and self.name in evidence:
            # 직접 관측된 경우
            belief = np.zeros(len(self.states))
            belief[evidence[self.name]] = 1.0
            return belief
        # 부모 메시지와 자식 메시지 결합
        pi = self.cpt.get((), np.array([0.5, 0.5]))
        lambda_total = np.ones(len(self.states))
        for lam in self.lambda_messages.values():
            lambda_total *= lam
        raw = pi * lambda_total
        return raw / raw.sum()  # 정규화


def variable_elimination(bn, query_var, evidence):
    """
    Variable Elimination으로 사후 확률 계산 (간략 버전).
    P(query_var | evidence)를 반환.
    """
    # 실제 구현에서는 elimination ordering, factor operations 필요
    # 여기서는 개념 설명을 위한 뼈대 구조
    factors = {node.name: node.cpt for node in bn}
    for ev_var, ev_val in evidence.items():
        # 증거 변수를 관측값으로 고정
        factors = reduce_factor(factors, ev_var, ev_val)
    # query_var 외 모든 변수를 marginalize out
    result = marginalize_all(factors, query_var)
    total = sum(result.values())
    return {k: v / total for k, v in result.items()}


# 사용 예시: 단순 의료 진단 베이즈 네트워크
# 구조: Smoking → LungCancer → Dyspnea
#                 ↓
#              Bronchitis → Dyspnea

smoking_priors = np.array([0.3, 0.7])  # [P(S=T), P(S=F)]

# P(LungCancer | Smoking)
lung_cancer_cpt = {
    (True,):  np.array([0.1, 0.9]),  # [P(L=T|S=T), P(L=F|S=T)]
    (False,): np.array([0.01, 0.99])
}

# P(Dyspnea | LungCancer, Bronchitis)
dyspnea_cpt = {
    (True,  True):  np.array([0.9, 0.1]),
    (True,  False): np.array([0.7, 0.3]),
    (False, True):  np.array([0.8, 0.2]),
    (False, False): np.array([0.1, 0.9]),
}

print("P(LungCancer=T | Smoking=T) =", lung_cancer_cpt[(True,)][0])
print("P(Dyspnea=T | LungCancer=T, Bronchitis=T) =", dyspnea_cpt[(True, True)][0])
```

## 5. 완전한 추론 예제: Asia 네트워크

Asia 네트워크는 Belief Propagation 연구의 표준 벤치마크이다.

```python
"""
Asia 네트워크 구조:
  Visit_Asia → Tuberculosis ─┐
                              ├→ TbOrCancer → Dyspnea
  Smoking → LungCancer ──────┘
  Smoking → Bronchitis → Dyspnea
  TbOrCancer → XRay
"""

class Factor:
    def __init__(self, variables, table):
        self.variables = variables
        self.table = table  # numpy array, shape = (|v1|, |v2|, ...)

    def reduce(self, var, value):
        """var를 value로 고정한 reduced factor 반환"""
        idx = self.variables.index(var)
        new_vars = [v for v in self.variables if v != var]
        new_table = np.take(self.table, value, axis=idx)
        return Factor(new_vars, new_table)

    def marginalize(self, var):
        """var를 합산하여 제거"""
        idx = self.variables.index(var)
        new_vars = [v for v in self.variables if v != var]
        new_table = self.table.sum(axis=idx)
        return Factor(new_vars, new_table)

    def multiply(self, other):
        """두 Factor를 곱 (공통 변수 기준 브로드캐스트)"""
        # 간략화된 버전 — 실제로는 변수 정렬이 필요
        common = set(self.variables) & set(other.variables)
        # ... (실제 구현 생략)
        pass


def sum_product_ve(factors, query, evidence):
    """
    Variable Elimination with Sum-Product.
    factors: List[Factor]
    query: 쿼리 변수 이름
    evidence: {var_name: observed_value}
    """
    # 1단계: 증거 적용 (reduce)
    reduced = []
    for f in factors:
        for var, val in evidence.items():
            if var in f.variables:
                f = f.reduce(var, val)
        reduced.append(f)

    # 2단계: 쿼리 변수 외 모든 변수 제거 (elimination ordering 적용)
    all_vars = {v for f in reduced for v in f.variables}
    to_eliminate = all_vars - {query}

    for var in to_eliminate:
        relevant = [f for f in reduced if var in f.variables]
        irrelevant = [f for f in reduced if var not in f.variables]
        # 관련 Factor들을 곱한 후 var를 marginalize
        combined = relevant[0]
        for f in relevant[1:]:
            combined = combined.multiply(f)
        marginalized = combined.marginalize(var)
        reduced = irrelevant + [marginalized]

    # 3단계: 나머지 Factor 곱 → 정규화
    result = reduced[0]
    for f in reduced[1:]:
        result = result.multiply(f)

    # 정규화
    total = result.table.sum()
    result.table /= total
    return result


# 로그 도메인 계산 (언더플로 방지)
def log_belief_propagation(cpt, evidence_log_likelihoods):
    """
    수치 안정성을 위해 log 도메인에서 메시지 계산.
    log(P(X)) = log(CPT) + sum(log(messages))
    """
    log_prior = np.log(cpt + 1e-300)  # 0 방지
    log_lambda = sum(evidence_log_likelihoods)
    log_unnormalized = log_prior + log_lambda
    # log-sum-exp 정규화
    max_val = log_unnormalized.max()
    log_z = max_val + np.log(np.exp(log_unnormalized - max_val).sum())
    return np.exp(log_unnormalized - log_z)

# 예시: Dyspnea가 관측되었을 때 LungCancer 사후 확률 계산
prior_lc = np.array([0.1, 0.9])  # P(LungCancer)
# 관측(Dyspnea=True)에 의한 likelihood message
lambda_from_dyspnea = np.array([0.7, 0.1])  # P(Dyspnea=T | LungCancer)

posterior = log_belief_propagation(prior_lc, [np.log(lambda_from_dyspnea)])
print(f"P(LungCancer=T | Dyspnea=T) ≈ {posterior[0]:.4f}")
# 출력 예시: P(LungCancer=T | Dyspnea=T) ≈ 0.4375
```

## 6. 베이즈 네트워크의 실제 응용 분야

- **의료 진단**: 증상과 검사 결과로부터 질병 사후 확률 추정
- **이메일 스팸 필터**: 나이브 베이즈 (Naive Bayes)는 각 변수가 클래스에 대해 조건부 독립이라 가정하는 단순화된 BN
- **자율주행**: 센서 융합, 장애물 의도 추론
- **금융 리스크 모델링**: 채무 불이행 연쇄 반응 시뮬레이션
- **자연어 처리**: 숨겨진 마르코프 모델(HMM)은 시계열 BN의 특수한 형태

## 7. 주의사항 및 한계

**루프(cycle) 그래프에서의 근사 추론:** Belief Propagation은 트리 구조에서만 정확하다. 루프가 있는 그래프(대부분의 실용적 모델)에서는 Loopy Belief Propagation(메시지를 반복적으로 전달하는 근사)이나 MCMC(Markov Chain Monte Carlo) 같은 샘플링 기법을 사용해야 한다.

**CPT의 조합 폭발:** 부모가 많은 노드의 CPT 크기는 부모 수에 지수적으로 증가한다. Noisy-OR, Noisy-AND 같은 파라미터 공유 모델을 사용해 완화할 수 있다.

**구조 학습의 어려움:** 그래프 구조 자체를 데이터에서 학습하는 Structure Learning 문제는 NP-hard이다. PC 알고리즘, GES(Greedy Equivalence Search) 등이 사용된다.

## 참고 자료
- [Judea Pearl - Causality (공식 사이트)](https://bayes.cs.ucla.edu/jp_home.html)
- [Tutorial on Exact Belief Propagation in Bayesian Networks (arxiv)](https://arxiv.org/abs/1201.4724)
- [pgmpy - Python 라이브러리 (GitHub)](https://github.com/pgmpy/pgmpy)
- [Probabilistic Graphical Models — Stanford Coursera](https://www.coursera.org/specializations/probabilistic-graphical-models)
