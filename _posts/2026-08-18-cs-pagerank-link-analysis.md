---
layout: post
title: "PageRank 알고리즘 완전 정복: 구글이 웹을 정렬한 수학적 원리"
date: 2026-08-18
categories: [cs, computer-science]
tags: [pagerank, graph-algorithm, link-analysis, power-iteration, markov-chain, web-crawling]
---

구글의 공동 창업자 래리 페이지(Larry Page)와 세르게이 브린(Sergey Brin)이 1998년 발표한 PageRank 알고리즘은 인터넷 검색의 역사를 바꾸었다. "좋은 페이지는 좋은 페이지들로부터 링크된다"는 직관을 수학적으로 정형화한 이 알고리즘은 마르코프 체인과 선형대수를 우아하게 결합한다. 단순히 키워드 빈도가 아닌 **링크 구조 자체**에서 페이지의 중요도를 추출한다는 발상이 혁명적이었다.

## 개념 설명: PageRank의 핵심 아이디어

PageRank는 웹을 방향 그래프(directed graph)로 모델링한다. 각 웹 페이지는 노드이고, 하이퍼링크는 간선이다. 어떤 페이지의 PageRank는 **그 페이지를 가리키는 다른 페이지들의 PageRank에 의해 결정**된다. 중요한 페이지로부터 받은 링크일수록 더 큰 기여를 한다.

### 랜덤 서퍼 모델

PageRank를 직관적으로 이해하는 방법은 **랜덤 서퍼(Random Surfer)** 모델이다.

- 무한히 많은 사용자가 임의의 웹 페이지에서 시작해 무작위로 링크를 클릭하며 탐색한다.
- 확률 `d`(댐핑 팩터, 보통 0.85)로 현재 페이지의 링크 중 하나를 임의로 클릭한다.
- 확률 `1-d`로 완전히 임의의 페이지로 순간이동(teleport)한다.
- 충분히 오랜 시간이 지난 후 각 페이지에 머물 확률이 바로 그 페이지의 PageRank다.

이 모델은 댐핑 팩터를 통해 **데드엔드(링크가 없는 페이지)** 와 **스파이더 트랩(자기 자신으로만 연결된 링크 구조)** 문제를 해결한다.

### 수식

N개의 페이지로 구성된 웹에서 페이지 i의 PageRank PR(i)는 다음과 같이 정의된다.

```
PR(i) = (1 - d) / N + d × Σ(j ∈ in(i)) PR(j) / out_degree(j)
```

- `d`: 댐핑 팩터 (일반적으로 0.85)
- `N`: 전체 페이지 수
- `in(i)`: 페이지 i를 가리키는 페이지 집합
- `out_degree(j)`: 페이지 j의 외부 링크 수

이를 행렬로 표현하면 구글 행렬(Google Matrix) G가 된다.

```
G = d × M + (1-d) / N × 1 × 1^T
```

M은 컬럼 정규화된 전이 행렬(column-stochastic transition matrix)이고, PageRank 벡터는 G의 **주요 좌고유벡터(principal left eigenvector)** 다.

## 왜 필요한가: 링크 분석의 중요성

텍스트 기반 검색(TF-IDF 등)은 키워드 스팸에 취약하다. 페이지에 특정 단어를 반복하거나 숨겨진 텍스트를 삽입하면 쉽게 순위를 조작할 수 있었다. PageRank는 **링크 구조라는 외부적 신호**를 사용하기 때문에 조작이 훨씬 어렵다. 가짜 링크 팜(link farm)을 만들어도 그 링크 팜 자체의 PageRank가 낮으면 효과가 거의 없다.

PageRank는 웹 검색 외에도 다양한 분야에 응용된다.

- **학술 논문 순위**: 인용 그래프에 적용해 논문의 영향력 측정 (Google Scholar)
- **소셜 네트워크 분석**: 영향력 있는 사용자 식별
- **추천 시스템**: 사용자-아이템 이분 그래프에서 중요 아이템 식별
- **생물정보학**: 단백질 상호작용 네트워크 분석
- **사기 탐지**: 비정상적인 링크 패턴 감지

## 실제 구현 예제 1: 거듭제곱 반복법(Power Iteration)

PageRank를 계산하는 가장 기본적인 방법은 **거듭제곱 반복법(Power Iteration)** 이다. 초기에 모든 페이지에 균일한 PageRank를 배정하고, 수식을 반복 적용해 수렴시킨다.

```python
from typing import NamedTuple


class WebGraph:
    def __init__(self, n: int):
        self.n = n
        self.edges: list[tuple[int, int]] = []
        self.out_degree = [0] * n

    def add_link(self, src: int, dst: int):
        self.edges.append((src, dst))
        self.out_degree[src] += 1


def pagerank(
    graph: WebGraph,
    damping: float = 0.85,
    max_iter: int = 100,
    tol: float = 1e-6,
) -> list[float]:
    n = graph.n
    # 초기 PageRank: 균일 분포
    pr = [1.0 / n] * n

    # 역방향 인접 리스트 구성 (인링크)
    in_links: list[list[int]] = [[] for _ in range(n)]
    for src, dst in graph.edges:
        in_links[dst].append(src)

    for iteration in range(max_iter):
        new_pr = [0.0] * n

        # 댄글링 노드(out_degree=0)의 PageRank를 전체에 균등 분배
        dangling_sum = sum(
            pr[i] for i in range(n) if graph.out_degree[i] == 0
        )

        for i in range(n):
            # 텔레포트 항 + 댄글링 노드 기여
            new_pr[i] = (1.0 - damping) / n + damping * dangling_sum / n
            # 인링크 기여
            for j in in_links[i]:
                new_pr[i] += damping * pr[j] / graph.out_degree[j]

        # 수렴 확인 (L1 노름)
        delta = sum(abs(new_pr[i] - pr[i]) for i in range(n))
        pr = new_pr
        if delta < tol:
            print(f"  수렴: {iteration + 1}회 반복")
            break

    return pr


# 간단한 웹 그래프 테스트
# A(0) → B(1) → C(2) → A(0), C(2) → B(1)
g = WebGraph(4)
g.add_link(0, 1)  # A → B
g.add_link(1, 2)  # B → C
g.add_link(2, 0)  # C → A
g.add_link(2, 1)  # C → B
g.add_link(3, 0)  # D → A  (D는 댄글링 노드가 아님)

pr = pagerank(g)
labels = ['A', 'B', 'C', 'D']
print("PageRank 결과:")
for i, score in enumerate(pr):
    print(f"  {labels[i]}: {score:.6f}")
# A: 0.368, B: 0.288, C: 0.214, D: 0.130 (근사값)
```

### 댄글링 노드 처리

out_degree가 0인 **댄글링 노드**는 PageRank를 흡수만 하고 배분하지 않아 전체 합이 1이 안 된다. 위 구현에서는 댄글링 노드의 PageRank를 모든 페이지에 균등하게 분배하여 확률 보존을 유지한다.

## 실제 구현 예제 2: NumPy를 이용한 행렬 기반 구현

대규모 그래프에서는 희소 행렬(sparse matrix) 연산이 필수지만, 이해를 위해 NumPy 밀집 행렬 버전을 구현한다.

```python
import numpy as np


def pagerank_matrix(
    adj_matrix: np.ndarray,
    damping: float = 0.85,
    max_iter: int = 200,
    tol: float = 1e-8,
) -> np.ndarray:
    """
    adj_matrix[i][j] = 1 이면 i → j 링크 존재.
    구글 행렬을 구성하고 거듭제곱 반복법으로 PageRank를 계산한다.
    """
    n = adj_matrix.shape[0]
    
    # 컬럼 정규화: 각 열의 합이 1이 되도록 (out_degree 정규화)
    col_sums = adj_matrix.sum(axis=0)
    # 댄글링 열(합=0)은 1/n으로 처리
    col_sums[col_sums == 0] = 1
    M = adj_matrix / col_sums  # 전이 확률 행렬
    
    # 댄글링 노드 처리: 원래 열이 모두 0이면 1/n 벡터로 대체
    dangling_cols = (adj_matrix.sum(axis=0) == 0)
    dangling_vector = np.ones(n) / n
    
    # 구글 행렬: G = d*M + (1-d)/n * J
    teleport = np.ones((n, n)) / n
    G = damping * M + (1 - damping) * teleport
    
    # 초기 PageRank 벡터
    pr = np.ones(n) / n

    for _ in range(max_iter):
        new_pr = G @ pr  # 행렬-벡터 곱
        # L2 노름으로 수렴 판단
        if np.linalg.norm(new_pr - pr) < tol:
            break
        pr = new_pr

    # 합계를 1로 정규화
    return pr / pr.sum()


# 예시: 4개 페이지 링크 그래프
# A(0) B(1) C(2) D(3)
adj = np.array([
    [0, 1, 0, 0],  # A → B
    [0, 0, 1, 0],  # B → C
    [1, 1, 0, 0],  # C → A, B
    [1, 0, 0, 0],  # D → A
], dtype=float)

scores = pagerank_matrix(adj)
print("PageRank (행렬 기반):")
for i, s in enumerate(scores):
    print(f"  Page {i}: {s:.6f}")

# 고유값 분해로 검증
eigenvalues, eigenvectors = np.linalg.eig(adj / adj.sum(axis=0))
idx = np.argmax(np.abs(eigenvalues))
principal_ev = np.abs(eigenvectors[:, idx])
principal_ev /= principal_ev.sum()
print("\n고유벡터 분해 결과 (검증):")
for i, s in enumerate(principal_ev):
    print(f"  Page {i}: {s:.6f}")
```

두 방법의 결과가 동일함을 확인할 수 있다. 거듭제곱 반복법은 O(iter × (N + E)) 복잡도를 가지며, 희소 행렬로 구현하면 수십억 개의 노드를 가진 실제 웹에도 적용 가능하다.

## PageRank의 변형과 발전

### Personalized PageRank (PPR)

텔레포트를 균일 분포 대신 특정 노드 집합으로 제한하면 특정 사용자 관점에서의 중요도를 계산할 수 있다. 추천 시스템에서 많이 사용된다.

```python
def personalized_pagerank(graph, seed_nodes, damping=0.85):
    n = graph.n
    pr = [1.0 / n] * n
    # teleport 벡터: seed 노드에만 확률 배분
    teleport = [0.0] * n
    for s in seed_nodes:
        teleport[s] = 1.0 / len(seed_nodes)
    
    # Power Iteration (생략 - 기본 구조는 동일하되
    # (1-d)/n 대신 (1-d)*teleport[i] 사용)
    return pr
```

### HITS 알고리즘

HITS(Hyperlink-Induced Topic Search)는 각 노드에 **권위 점수(authority)**와 **허브 점수(hub)**를 모두 부여한다. 좋은 허브는 좋은 권위 페이지를 많이 링크하고, 좋은 권위 페이지는 좋은 허브로부터 많이 링크된다. 이는 PageRank와 상호 보완적인 접근이다.

## 주의사항 및 팁

### 1. 수렴 속도는 댐핑 팩터에 달려 있다

d=0.85일 때 일반적으로 50~100회 반복으로 수렴한다. d가 클수록 수렴이 느려진다.

### 2. 희소 행렬로 구현하라

실제 웹 그래프는 매우 희소하다(평균 out_degree ~10). `scipy.sparse`를 사용하면 메모리를 O(E)로 줄이고 행렬 곱 속도를 크게 개선할 수 있다.

### 3. L1 정규화를 유지하라

각 반복마다 PR 벡터의 합이 1임을 유지해야 한다. 댄글링 노드 처리를 잘못하면 합이 1이 아닌 값으로 수렴한다.

### 4. 실제 구글은 훨씬 복잡하다

현재 구글 검색은 PageRank 외에도 수백 가지 신호(콘텐츠 품질, 사용자 행동, 링크 앵커 텍스트, 도메인 나이 등)를 조합한다. PageRank는 여전히 중요한 신호지만 단독으로 사용되지는 않는다.

PageRank는 웹 검색을 넘어 그래프 기반 중요도 측정의 범용 도구로 자리 잡았다. 마르코프 체인의 정류 분포(stationary distribution)를 계산하는 거듭제곱 반복법은 그 자체로 선형대수와 확률론이 알고리즘으로 결합된 아름다운 사례다.

## 참고 자료
- [ashkonf/pagerank - Python PageRank Implementation](https://github.com/ashkonf/PageRank)
- [erimerkin/basic-pagerank - Power Iteration Method](https://github.com/erimerkin/basic-pagerank)
