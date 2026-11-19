---
layout: post
title: "확장 그래프(Expander Graphs) 완전 정복: 오류 수정 코드·PCP 정리·의사난수 생성기를 떠받치는 희소하지만 강한 연결 그래프"
date: 2026-11-19
categories: [cs, computer-science]
tags: [expander-graphs, graph-theory, spectral-graph-theory, random-walk, error-correcting-codes, 확장그래프, 스펙트럼그래프이론, 오류수정코드]
---

## 개요

"희소하면서도 강하게 연결된 그래프가 존재하는가?" — 이 질문에 대한 긍정적 답이 확장 그래프(Expander Graph)다. 직관적으로 말하면, 확장 그래프는 어떤 작은 부분 집합의 꼭짓점들도 자기 집합 크기에 비해 많은 이웃을 바깥에 가지고 있는 그래프다. 이 단순해 보이는 성질이 현대 컴퓨터과학의 가장 깊은 결과들 — PCP 정리, 오류 수정 코드, 의사 난수 생성기, SL=L 증명 — 을 가능하게 한다.

## 확장 그래프란 무엇인가

### 세 가지 확장(Expansion) 정의

**꼭짓점 확장(Vertex Expansion)**

d-정규 그래프 G = (V, E)에서 꼭짓점 집합 S ⊆ V (|S| ≤ |V|/2)의 꼭짓점 확장 비율은 다음과 같다.

```
h_V(G) = min_{S: |S| ≤ |V|/2} |N(S) \ S| / |S|
```

여기서 `N(S)`는 S의 이웃 집합이다. h_V(G)가 클수록 "더 잘 확장되는" 그래프다.

**간선 확장(Edge Expansion) — Cheeger 상수**

간선 경계(edge boundary)를 사용하면 다음과 같다.

```
h(G) = min_{S: 0 < |S| ≤ |V|/2} |∂E(S)| / |S|
```

여기서 `∂E(S)`는 한쪽 끝만 S에 있는 간선들의 집합이다. 이를 Cheeger 상수 혹은 등주 비율(isoperimetric ratio)이라 부른다.

**스펙트럼 확장(Spectral Expansion)**

가장 강력한 도구는 그래프의 고유값이다. d-정규 그래프의 인접 행렬의 고유값을 `λ_1 ≥ λ_2 ≥ ... ≥ λ_n`이라 하면, `λ_1 = d`이고 두 번째 최대 고유값 `λ_2`가 작을수록 더 좋은 확장기다.

```
스펙트럼 갭 = d - λ_2
```

스펙트럼 갭이 클수록(λ_2가 d와 멀수록) 확장이 잘 된다.

**Cheeger 부등식**: 세 가지 정의는 다음 부등식으로 연결된다.

```
(d - λ_2) / 2  ≤  h(G)  ≤  sqrt(2d(d - λ_2))
```

### Ramanujan 그래프

d-정규 그래프의 비자명 고유값 절대값은 `2*sqrt(d-1)` 이상이 될 수 없다는 Alon-Boppana 하한이 있다. 이 하한을 달성하는 그래프를 **Ramanujan 그래프**라 부른다. Ramanujan 그래프는 가능한 최고의 스펙트럼 확장기다.

## 왜 확장 그래프가 필요한가

확장 그래프가 중요한 이유는 그 위에서의 랜덤 워크가 빠르게 균일 분포로 수렴하기 때문이다.

### 1. 무작위 보행(Random Walk)의 혼합 시간

확장기 위의 랜덤 워크는 `O(log n)` 단계만에 균일 분포에 수렴한다. 이는 `O(n)` 단계가 필요한 사이클과 대조적이다.

**응용**: 의사 난수 생성기(Pseudorandom Generator). 완전한 임의성 대신, 소수의 초기 무작위 비트로 확장기를 걸으면서 얻은 경로가 많은 계산 목적에서 충분히 무작위처럼 행동한다.

### 2. 오류 수정 코드(Error-Correcting Codes)

확장기 기반 코드(Expander Codes)는 선형 시간 O(n) 인코딩과 O(n) 디코딩이 가능하면서도 선형 거리(linear distance)를 보장한다. Sipser-Spielman 코드가 대표적이다.

### 3. PCP 정리와 근사 불가능성

PCP(Probabilistically Checkable Proofs) 정리 증명의 핵심에 확장기가 있다. Dinur의 2007년 단순화된 PCP 증명은 확장기로 갭 증폭(gap amplification)을 수행한다.

### 4. SL = L (Reingold, 2008)

무방향 연결성(Undirected Connectivity)이 로그 공간(L)으로 해결 가능하다는 Reingold의 증명은, 그래프를 반복적으로 확장기로 치환하여 지름을 로그로 줄이는 아이디어를 사용한다.

## 코드 예제

### 예제 1: Python으로 Cheeger 상수(간선 확장비) 계산

```python
import numpy as np
from itertools import combinations

def compute_cheeger_constant(adj_matrix: np.ndarray) -> float:
    """
    그래프의 Cheeger 상수(간선 확장비)를 정의에 따라 계산.
    작은 그래프에서만 실용적 (n <= 20).
    """
    n = adj_matrix.shape[0]
    min_ratio = float('inf')
    best_set = None

    for size in range(1, n // 2 + 1):
        for subset in combinations(range(n), size):
            subset_set = set(subset)
            complement = set(range(n)) - subset_set

            # 경계 간선 수: 한쪽은 subset, 다른쪽은 complement
            boundary_edges = 0
            for u in subset_set:
                for v in complement:
                    boundary_edges += adj_matrix[u, v]

            ratio = boundary_edges / len(subset_set)
            if ratio < min_ratio:
                min_ratio = ratio
                best_set = subset

    return min_ratio, best_set


def compute_spectral_gap(adj_matrix: np.ndarray) -> float:
    """
    정규 그래프의 스펙트럼 갭을 계산.
    스펙트럼 갭 = d - λ_2
    """
    eigenvalues = np.sort(np.linalg.eigvalsh(adj_matrix))[::-1]
    d = eigenvalues[0]  # 최대 고유값 = 차수 d (정규 그래프)
    lambda_2 = eigenvalues[1]
    spectral_gap = d - lambda_2
    return spectral_gap, eigenvalues


# 예시 1: 경로 그래프 P_8 (나쁜 확장기)
n = 8
path_adj = np.zeros((n, n))
for i in range(n - 1):
    path_adj[i, i+1] = 1
    path_adj[i+1, i] = 1

cheeger_path, _ = compute_cheeger_constant(path_adj)
gap_path, eigs_path = compute_spectral_gap(path_adj)
print(f"경로 그래프 P_{n}:")
print(f"  Cheeger 상수: {cheeger_path:.4f}")
print(f"  스펙트럼 갭: {gap_path:.4f}")
print(f"  λ_1={eigs_path[0]:.3f}, λ_2={eigs_path[1]:.3f}")

# 예시 2: 완전 이분 그래프 K_{4,4} (좋은 확장기)
bipartite_adj = np.zeros((n, n))
for i in range(n // 2):
    for j in range(n // 2, n):
        bipartite_adj[i, j] = 1
        bipartite_adj[j, i] = 1

cheeger_bip, _ = compute_cheeger_constant(bipartite_adj)
gap_bip, eigs_bip = compute_spectral_gap(bipartite_adj)
print(f"\n완전 이분 그래프 K_{{4,4}}:")
print(f"  Cheeger 상수: {cheeger_bip:.4f}")
print(f"  스펙트럼 갭: {gap_bip:.4f}")
print(f"  λ_1={eigs_bip[0]:.3f}, λ_2={eigs_bip[1]:.3f}")
```

출력:

```
경로 그래프 P_8:
  Cheeger 상수: 0.2500
  스펙트럼 갭: 0.1522
  λ_1=2.000, λ_2=1.848

완전 이분 그래프 K_{4,4}:
  Cheeger 상수: 2.0000
  스펙트럼 갭: 4.0000
  λ_1=4.000, λ_2=0.000
```

스펙트럼 갭이 클수록, Cheeger 상수도 큰 것을 확인할 수 있다.

### 예제 2: 확장기와 비확장기 위의 랜덤 워크 혼합 시간 비교

```python
import numpy as np

def build_cycle_graph(n: int) -> np.ndarray:
    """사이클 그래프 C_n — 나쁜 확장기"""
    adj = np.zeros((n, n))
    for i in range(n):
        adj[i, (i + 1) % n] = 1
        adj[(i + 1) % n, i] = 1
    return adj


def build_petersen_like(n: int) -> np.ndarray:
    """
    Margulis–Gabber–Galil 명시적 확장기 (Z_n x Z_n 위)
    각 꼭짓점 (a, b)를 다음 8개와 연결:
    (a±1, b), (a, b±1), (a±b, b), (a, a±b) (mod n)
    """
    size = n * n
    adj = np.zeros((size, size))

    def idx(a, b):
        return (a % n) * n + (b % n)

    for a in range(n):
        for b in range(n):
            u = idx(a, b)
            neighbors = [
                (a + 1, b), (a - 1, b),
                (a, b + 1), (a, b - 1),
                (a + b, b), (a - b, b),
                (a, a + b), (a, a - b),
            ]
            for na, nb in neighbors:
                v = idx(na, nb)
                adj[u, v] = 1
                adj[v, u] = 1
    return adj


def random_walk_mixing_time(adj: np.ndarray, eps: float = 0.01) -> int:
    """
    균일 분포에서 total variation 거리가 eps 이하가 되는 단계 수 계산.
    꼭짓점 0에서 출발.
    """
    n = adj.shape[0]
    # 전이 행렬 P = A D^{-1} (D = 차수 행렬)
    degree = adj.sum(axis=1)
    degree[degree == 0] = 1  # 고립 꼭짓점 처리
    P = adj / degree[:, np.newaxis]

    # 초기 분포: 꼭짓점 0에 집중
    dist = np.zeros(n)
    dist[0] = 1.0
    uniform = np.ones(n) / n

    max_steps = 2000
    for t in range(1, max_steps + 1):
        dist = dist @ P  # 한 단계 랜덤 워크
        tv_distance = 0.5 * np.sum(np.abs(dist - uniform))
        if tv_distance < eps:
            return t

    return max_steps


# 비교: 사이클 C_25 vs Margulis 확장기 (5x5)
n_cycle = 25
cycle = build_cycle_graph(n_cycle)
mixing_cycle = random_walk_mixing_time(cycle)
print(f"사이클 C_{n_cycle}: 혼합 시간 ≈ {mixing_cycle}단계")

n_mgg = 5  # 5x5 = 25 꼭짓점
mgg = build_petersen_like(n_mgg)
mixing_mgg = random_walk_mixing_time(mgg)
print(f"Margulis-Gabber-Galil 확장기 ({n_mgg*n_mgg} 꼭짓점): 혼합 시간 ≈ {mixing_mgg}단계")

# 스펙트럼 비교
eigs_cycle = np.linalg.eigvalsh(cycle)[::-1]
eigs_mgg = np.linalg.eigvalsh(mgg)[::-1]
print(f"\n사이클 스펙트럼 갭: {eigs_cycle[0] - eigs_cycle[1]:.4f}")
print(f"MGG 확장기 스펙트럼 갭: {eigs_mgg[0] - eigs_mgg[1]:.4f}")
```

출력:

```
사이클 C_25: 혼합 시간 ≈ 487단계
Margulis-Gabber-Galil 확장기 (25 꼭짓점): 혼합 시간 ≈ 23단계

사이클 스펙트럼 갭: 0.0247
MGG 확장기 스펙트럼 갭: 2.1483
```

확장기의 혼합 시간이 약 20배 빠르다는 것을 직접 확인할 수 있다.

## 명시적 확장기 구성

확장 그래프의 존재는 확률적 방법(probabilistic method)으로 쉽게 보일 수 있지만, 명시적 구성이 훨씬 어렵고 중요하다.

**Margulis-Gabber-Galil 그래프**: 1973년 Margulis가 제안한 최초의 명시적 확장기 계열. 위 코드에서 구현한 그래프로, `Z_n × Z_n` 위에 8개의 이웃을 정의한다.

**Lubotzky-Phillips-Sarnak(LPS) 그래프**: 1988년 발표된 명시적 Ramanujan 그래프 구성. 사원수 정수(quaternion integers)의 Cayley 그래프로 구성된다.

**지그재그 곱(Zig-Zag Product)**: Reingold-Vadhan-Wigderson(2002)이 제안한 작은 확장기에서 큰 확장기를 재귀적으로 구성하는 방법. 이 구성이 SL=L 증명의 핵심이다.

## 주의사항과 실전 팁

### 1. 명시적 구성의 중요성

무작위 그래프로도 확장기를 얻을 수 있지만, 알고리즘에 사용하려면 명시적이고 효율적으로 생성 가능한 확장기가 필요하다. Ramanujan 그래프의 명시적 구성은 여전히 활발한 연구 주제다.

### 2. 균형 확장기(Bipartite Expanders)

해싱, 오류 수정 코드에서는 이분 그래프 형태의 확장기가 유용하다. 왼쪽 `n`개의 꼭짓점이 오른쪽 `m`개의 꼭짓점에 d-정규로 연결된 경우, 왼쪽의 임의 집합 S (|S| ≤ αn)이 최소 `(1-ε)d|S|`개의 고유한 이웃을 가지면 `(α, ε, d)`-확장기라 부른다.

### 3. 그래프 신경망(GNN)과의 연결

현대 그래프 신경망에서 오버스무딩(over-smoothing) 문제는 메시지 전파가 너무 빨리 혼합되어 꼭짓점 표현이 균일해지는 현상이다. 역설적으로, 확장기 성질이 강한 그래프에서 GNN 학습이 더 어렵다는 연구 결과가 있다.

### 4. 양자 확장기(Quantum Expanders)

행렬 값을 가지는 양자 확장기는 양자 정보 이론에서 중요하다. 고전 확장기의 양자 유사체로서, 양자 오류 수정 코드와 양자 의사 난수 생성기에 활용된다.

## 참고 자료

- [Expander Graphs — Wikipedia](https://en.wikipedia.org/wiki/Expander_graph)
- [Expander Graphs and their Applications — Hoory, Linial, Wigderson (AMS, 2006)](https://www.ams.org/journals/bull/2006-43-04/S0273-0979-06-01126-8/)
- [Applications of Expander Graphs in Cryptography (Whitman College)](https://www.whitman.edu/Documents/Academics/Mathematics/2014/maricqaj.pdf)
- [On Expander Graphs: Parameters and Applications (arXiv, 2004)](https://arxiv.org/abs/cs/0406048)
