---
layout: post
title: "Karger's Algorithm과 무작위화 알고리즘 분석: 최소 컷을 확률로 정복하는 법"
date: 2026-10-21
categories: [cs, computer-science]
tags: [karger, randomized-algorithm, minimum-cut, graph-theory, probability, monte-carlo, las-vegas]
---

# Karger's Algorithm과 무작위화 알고리즘 분석: 최소 컷을 확률로 정복하는 법

**최소 컷(Minimum Cut)** 문제는 그래프에서 두 집합으로 정점을 나눌 때 두 집합을 연결하는 간선의 최소 수(또는 최소 가중치 합)를 구하는 문제다. 통신망 설계, 이미지 분할, 클러스터링, 네트워크 신뢰도 분석 등 수많은 응용을 가진다.

1993년 David Karger(당시 Stanford 박사 과정생)는 놀라울 만큼 간단한 **무작위화 알고리즘**으로 이 문제를 해결했다. 알고리즘의 정확성을 보장하지 않지만, 충분히 많이 반복하면 높은 확률로 올바른 답을 얻는다. 이것이 **Monte Carlo 알고리즘**의 전형적인 예다.

## 개념 설명

### 최소 컷 문제

무방향 그래프 $G = (V, E)$에서 **컷(cut)**은 정점 집합 $V$를 두 비공 부분집합 $S$, $\bar{S} = V \setminus S$로 나누는 것이다. 컷의 **크기**는 $S$와 $\bar{S}$ 사이의 간선 수다.

**글로벌 최소 컷(Global Minimum Cut)**: 특정 소스·싱크 없이, 모든 가능한 컷 중 크기가 최소인 것을 찾는다.

기존 접근법:
- **최대 유량 기반**: 최소 컷–최대 유량 정리를 이용해 소스·싱크를 고정하고 최대 유량을 구하지만, 모든 정점 쌍에 대해 $O(n)$번 실행해야 하므로 느리다.
- **분기 한정법**: NP-하드인 변형 문제에는 유용하지만 일반 최소 컷엔 과도하다.

Karger의 접근법은 극히 단순하다: **무작위로 간선을 하나 선택해 두 끝점을 합친다(수축, contraction)**. 두 정점이 남을 때까지 반복하면, 그 두 정점 사이의 간선 수가 컷의 크기를 나타낸다.

### 수축(Contraction) 연산

간선 $(u, v)$를 수축하면:
1. $u$와 $v$를 하나의 슈퍼 정점 $uv$로 합친다.
2. $u$ 또는 $v$에 연결된 모든 간선이 $uv$에 연결된다.
3. $u$와 $v$ 사이의 간선(자기 루프)은 제거한다.
4. 그 외 평행 간선은 유지한다(멀티그래프).

수축을 반복할수록 정점 수는 줄어들고, 최종 두 슈퍼 정점 사이의 간선이 한 가능한 컷을 나타낸다.

### 확률 분석

그래프에 $n$개의 정점이 있고, 최소 컷의 크기가 $k$라 하자.

**핵심 관찰**: 최소 컷에 속한 간선이 수축되지 않으면, 최종 컷이 최소 컷이다.

**단계 $i$에서 최소 컷 간선이 살아남을 확률**:

1. $n - i + 1$개의 정점이 남아 있을 때, 총 간선 수 $\geq (n - i + 1) \cdot k / 2$ (각 정점의 차수 $\geq k$이므로)
2. 최소 컷 간선 수 = $k$
3. 단계 $i$에서 최소 컷 간선이 선택될 확률 $\leq \frac{k}{(n-i+1)k/2} = \frac{2}{n-i+1}$
4. 따라서 선택되지 않을 확률 $\geq 1 - \frac{2}{n-i+1} = \frac{n-i-1}{n-i+1}$

$n-2$번의 수축 전체에서 최소 컷이 살아남을 확률:

$$P(\text{성공}) \geq \prod_{i=0}^{n-3} \frac{n-i-1}{n-i+1} = \frac{n-2}{n} \cdot \frac{n-3}{n-1} \cdot \frac{n-4}{n-2} \cdots \frac{2}{4} \cdot \frac{1}{3} = \frac{2}{n(n-1)}$$

$n = 100$이면 성공 확률 $\approx 0.02\%$. 낮아 보이지만, $T = n^2 \ln n$번 반복하면 실패 확률이 $1/n$로 줄어든다.

## 왜 이게 의미 있는가?

### 무작위화 알고리즘의 힘

**Monte Carlo 알고리즘**: 실행 시간이 확정적이지만, 결과가 일정 확률로 틀릴 수 있다. 반복 횟수를 늘려 오류 확률을 지수적으로 줄인다.

**Las Vegas 알고리즘**: 결과는 항상 정확하지만, 실행 시간이 확률적이다(예: 랜덤 QuickSort).

Karger's Algorithm은 Monte Carlo이지만, 여러 번 실행하고 최솟값을 취하면 사실상 Las Vegas처럼 동작한다.

### Karger-Stein 개선 (1996)

Karger와 Stein은 단순 반복 대신 **재귀적 수축**을 도입해 성능을 크게 개선했다:

- 두 개 남을 때까지 수축하는 대신, $\lceil n/\sqrt{2} \rceil$개까지 수축한 뒤 두 복사본을 재귀적으로 탐색
- 성공 확률: 단일 실행 $\Omega(1/\log n)$ (크게 향상)
- 전체 복잡도: $O(n^2 \log^3 n)$

## 실제 구현 예제

### 예제 1: Karger's Random Contraction Algorithm

```python
import random
from collections import defaultdict
import copy

class Graph:
    def __init__(self):
        self.adj = defaultdict(list)  # 인접 리스트 (멀티그래프)
        self.vertices = set()

    def add_edge(self, u, v):
        self.adj[u].append(v)
        self.adj[v].append(u)
        self.vertices.add(u)
        self.vertices.add(v)

    def num_vertices(self):
        return len(self.vertices)

    def num_edges(self):
        return sum(len(neighbors) for neighbors in self.adj.values()) // 2


def karger_contract(graph: Graph) -> int:
    """
    Karger's Random Contraction Algorithm.
    두 정점이 남을 때까지 무작위 간선을 수축.
    최종 두 슈퍼 정점 사이의 간선 수 반환.
    """
    # 깊은 복사 (원본 보존)
    adj = defaultdict(list)
    for v, neighbors in graph.adj.items():
        adj[v] = list(neighbors)
    vertices = list(graph.vertices)

    # Union-Find 기반 수축
    parent = {v: v for v in vertices}

    def find(x):
        while parent[x] != x:
            parent[x] = parent[parent[x]]
            x = parent[x]
        return x

    def union(x, y):
        px, py = find(x), find(y)
        if px != py:
            parent[px] = py

    num_vertices = len(vertices)

    # 모든 간선 목록 생성
    edges = []
    for v in vertices:
        for u in adj[v]:
            if v < u:
                edges.append((v, u))

    while num_vertices > 2:
        # 무작위 간선 선택
        e = random.choice(edges)
        u, v = e
        pu, pv = find(u), find(v)

        if pu == pv:
            # 자기 루프 - 건너뜀
            edges.remove(e)
            continue

        # 수축: pu를 pv로 합침
        union(pu, pv)
        num_vertices -= 1

        # 자기 루프 제거
        edges = [(a, b) for a, b in edges if find(a) != find(b)]

    return len(edges)


def karger_min_cut(graph: Graph, trials: int = None) -> int:
    """
    충분히 많은 횟수 반복해 최솟값 반환.
    성공 확률 >= 1 - 1/n.
    """
    n = graph.num_vertices()
    if trials is None:
        import math
        trials = int(n * n * math.log(n))

    min_cut = float('inf')
    for _ in range(trials):
        cut = karger_contract(graph)
        min_cut = min(min_cut, cut)

    return min_cut


# 예시: 피터슨 그래프 (최소 컷 = 3)
g = Graph()
edges = [
    (0,1), (1,2), (2,3), (3,4), (4,0),
    (0,5), (1,6), (2,7), (3,8), (4,9),
    (5,7), (7,9), (9,6), (6,8), (8,5)
]
for u, v in edges:
    g.add_edge(u, v)

result = karger_min_cut(g, trials=100)
print(f"추정 최소 컷: {result}")  # 3 기대
```

### 예제 2: Karger-Stein 개선 알고리즘

```python
import random
import math
from collections import defaultdict

class KargerStein:
    def __init__(self):
        pass

    def contract(self, adj: dict, target: int) -> dict:
        """adj를 target 정점까지 수축"""
        vertices = list(adj.keys())
        parent = {v: v for v in vertices}

        def find(x):
            while parent[x] != x:
                parent[x] = parent[parent[x]]
                x = parent[x]
            return x

        edges = []
        for v in vertices:
            for u in adj[v]:
                if id(v) < id(u) or v < u:
                    edges.append((v, u))

        current = len(vertices)
        while current > target:
            if not edges:
                break
            i = random.randrange(len(edges))
            u, v = edges[i]
            pu, pv = find(u), find(v)
            if pu == pv:
                edges.pop(i)
                continue
            parent[pu] = pv
            current -= 1
            edges = [(a, b) for a, b in edges if find(a) != find(b)]

        # 새 인접 리스트 구성
        new_adj = defaultdict(list)
        for u, v in edges:
            pu, pv = find(u), find(v)
            if pu != pv:
                new_adj[pu].append(pv)
                new_adj[pv].append(pu)
        return dict(new_adj)

    def min_cut_recursive(self, adj: dict) -> int:
        n = len(adj)
        if n <= 6:
            # 소규모는 브루트포스
            min_c = float('inf')
            for _ in range(n * n):
                c = self._single_contract(dict(adj))
                min_c = min(min_c, c)
            return min_c

        # t = ceil(n / sqrt(2)) + 1
        t = math.ceil(n / math.sqrt(2)) + 1

        results = []
        for _ in range(2):
            # t까지 수축
            contracted = self.contract(dict(adj), t)
            # 재귀
            results.append(self.min_cut_recursive(contracted))

        return min(results)

    def _single_contract(self, adj: dict) -> int:
        """단순 수축 (소규모용)"""
        parent = {v: v for v in adj}

        def find(x):
            while parent[x] != x:
                x = parent[x]
            return x

        edges = []
        for v in adj:
            for u in adj[v]:
                if v < u:
                    edges.append((v, u))

        n = len(adj)
        while n > 2 and edges:
            i = random.randrange(len(edges))
            u, v = edges[i]
            pu, pv = find(u), find(v)
            if pu == pv:
                edges.pop(i)
                continue
            parent[pu] = pv
            n -= 1
            edges = [(a, b) for a, b in edges if find(a) != find(b)]

        return len(edges)

    def solve(self, adj: dict) -> int:
        """Karger-Stein 최소 컷"""
        n = len(adj)
        repetitions = math.ceil(math.log(n) ** 2)
        min_cut = float('inf')
        for _ in range(repetitions):
            cut = self.min_cut_recursive(dict(adj))
            min_cut = min(min_cut, cut)
        return min_cut

# 사용 예시
adj = defaultdict(list)
test_edges = [(0,1),(0,2),(1,2),(2,3),(3,4),(3,5),(4,5)]
for u, v in test_edges:
    adj[u].append(v)
    adj[v].append(u)

ks = KargerStein()
result = ks.solve(dict(adj))
print(f"Karger-Stein 최소 컷: {result}")  # 2 기대 (0-1-2 vs 3-4-5)
```

## 확률 증폭(Probability Amplification)

단일 실행 성공 확률이 $p = 2/(n(n-1))$이라면, $T$번 독립 실행 후 모두 실패할 확률은:

$$P(\text{모두 실패}) = (1-p)^T \leq e^{-pT}$$

$T = n^2 \ln n / 2$로 설정하면 실패 확률 $\leq e^{-\ln n} = 1/n$.

이 **확률 증폭(amplification)** 기법은 무작위화 알고리즘 설계의 핵심 도구다:
- 성공 확률 $p$가 작아도 $O(1/p)$번 반복으로 높은 확률 보장
- 실패 확률을 지수적으로 감소시킬 수 있다

## 실행 복잡도 비교

| 알고리즘 | 복잡도 | 성공 확률 |
|----------|--------|-----------|
| 기본 Karger (1회) | $O(n^2)$ | $\geq 2/n^2$ |
| 기본 Karger ($n^2 \ln n$회 반복) | $O(n^4 \log n)$ | $\geq 1 - 1/n$ |
| Karger-Stein (단일) | $O(n^2 \log n)$ | $\geq 1/\log n$ |
| Karger-Stein ($\log^2 n$회 반복) | $O(n^2 \log^3 n)$ | $\geq 1 - 1/n$ |
| 결정론적 (Gabow et al.) | $O(m n \log(n^2/m))$ | 100% |

## 주의사항과 팁

### 멀티그래프와 평행 간선

Karger's Algorithm은 평행 간선을 허용하는 멀티그래프에서 작동한다. 단순 그래프를 입력으로 받더라도 수축 과정에서 평행 간선이 생기므로, 구현 시 간선 제거 없이 유지해야 한다. 평행 간선이 많을수록 그 간선이 선택될 확률이 높아지는 것이 알고리즘의 의도적인 설계다.

### 랜덤 시드 관리

재현 가능한 실험을 위해 랜덤 시드를 명시하라. 성능 테스트 시 여러 시드로 실험하여 평균적 성능을 측정해야 한다.

### 최솟값 추적

$T$번 실행 중 얻은 컷의 최솟값이 최종 답이다. 단순히 첫 번째 해를 반환하면 안 된다.

### 무작위화 알고리즘의 일반 원칙

1. **확률 분석**: 정확한 성공 확률 하한을 증명한다.
2. **증폭**: 반복 횟수를 정해 원하는 성공 확률을 달성한다.
3. **드 무아브르-라플라스 정리**: 대수의 법칙을 활용해 분산을 줄인다.
4. **최악 입력**: 무작위화는 최악 입력이 없다 — 적대자는 무작위 선택을 예측할 수 없다.

### 실용적 응용

- **이미지 분할**: 픽셀 그래프의 최소 컷으로 배경·전경 분리
- **VLSI 설계**: 회로 분할로 열 발산·배선 길이 최소화
- **소셜 네트워크 분석**: 커뮤니티 간 연결의 병목 탐지
- **네트워크 신뢰도**: 통신망에서 단절에 필요한 최소 링크 수 계산

## 참고 자료
- [Wikipedia: Karger's Algorithm](https://en.wikipedia.org/wiki/Karger%27s_algorithm)
- [Stanford CS265 Lecture 2: Karger's Algorithm](https://web.stanford.edu/class/cs265/Lectures/Lecture2/l2.pdf)
- [Dartmouth Randomized Algorithms Lecture Notes](https://cs.dartmouth.edu/~deepc/LecNotes/Rand/lec3.pdf)
- [Wikipedia: Randomized Algorithm](https://en.wikipedia.org/wiki/Randomized_algorithm)
