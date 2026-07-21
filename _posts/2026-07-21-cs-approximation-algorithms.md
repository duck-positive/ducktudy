---
layout: post
title: "근사 알고리즘(Approximation Algorithms) 완전 정복: NP-난해 문제를 다항 시간에 근사 풀기"
date: 2026-07-21
categories: [cs, computer-science]
tags: [approximation-algorithms, NP-hard, TSP, vertex-cover, set-cover, greedy, complexity-theory]
---

## 근사 알고리즘이란 무엇인가?

P ≠ NP가 사실이라면(아직 증명되지 않았지만 대다수 컴퓨터과학자들이 믿는 명제), NP-난해(NP-hard) 문제들은 다항 시간 내에 정확한 최적 해를 구할 수 없다. 하지만 현실에서는 정확한 최적 해가 아니더라도 충분히 좋은 해가 필요하다.

**근사 알고리즘(Approximation Algorithm)**은 NP-난해 문제에 대해 다항 시간 내에 실행되면서, 최적 해(OPT)와의 비율이 보장된 해를 구하는 알고리즘이다.

### 근사 비율 (Approximation Ratio)

알고리즘 A의 근사 비율 α는 다음과 같이 정의된다:

- **최소화 문제**: A가 반환한 해의 비용 C_A와 최적 비용 C_OPT에 대해, **모든 입력**에서 C_A ≤ α · C_OPT
- **최대화 문제**: C_A ≥ (1/α) · C_OPT

α ≥ 1이며, α = 1이면 정확한 최적 해, α = 1.5이면 최적 해의 1.5배 이내임을 보장한다.

---

## 왜 근사 알고리즘이 필요한가?

NP-난해 문제는 우리 주변 어디에나 있다:

- **외판원 문제 (TSP)**: 물류 경로 최적화, 반도체 제조 공정에서의 드릴 경로
- **정점 커버 (Vertex Cover)**: 네트워크 보안 모니터링 지점 선택, 바이러스 확산 통제
- **집합 커버 (Set Cover)**: 통신망 기지국 배치, 공공 서비스 입지 선정, 테스트 케이스 최소화
- **배낭 문제 (Knapsack)**: 포트폴리오 최적화, 리소스 할당
- **그래프 채색 (Graph Coloring)**: 컴파일러 레지스터 할당, 시험 시간표 배정

이런 문제들을 정확히 풀려면 지수 시간이 걸리지만, 근사 알고리즘은 현실적인 시간 내에 "충분히 좋은" 해를 보장한다.

---

## 문제 1: 정점 커버 (Vertex Cover) — 2-근사

**정점 커버(Vertex Cover)** 문제는 그래프 G = (V, E)에서 모든 에지가 적어도 하나의 끝점을 포함하는 최소 크기의 정점 집합을 찾는 것이다. NP-완전 문제지만 2-근사 알고리즘이 존재한다.

**알고리즘**: 최대 매칭(maximal matching)을 구하고, 그 매칭에 포함된 에지의 양쪽 끝점 모두를 커버 집합에 추가한다.

```python
def vertex_cover_2approx(adj_list, n):
    """
    2-근사 정점 커버 알고리즘.
    adj_list: {u: [v1, v2, ...]} 형태의 인접 리스트
    n: 정점 수
    반환: 정점 커버 집합
    """
    cover = set()
    matched = set()  # 이미 처리된 에지의 끝점

    # 그리디 방식으로 maximal matching 구성
    for u in range(n):
        for v in adj_list.get(u, []):
            if u < v:  # 중복 방지
                if u not in matched and v not in matched:
                    # 이 에지를 매칭에 추가
                    cover.add(u)
                    cover.add(v)
                    matched.add(u)
                    matched.add(v)

    return cover


def verify_vertex_cover(adj_list, n, cover):
    """정점 커버가 유효한지 검증"""
    for u in range(n):
        for v in adj_list.get(u, []):
            if u < v:
                if u not in cover and v not in cover:
                    return False, (u, v)  # 이 에지가 커버되지 않음
    return True, None


# 예시 그래프
# 0-1, 0-2, 1-3, 2-3, 3-4
graph = {
    0: [1, 2],
    1: [0, 3],
    2: [0, 3],
    3: [1, 2, 4],
    4: [3]
}

cover = vertex_cover_2approx(graph, 5)
valid, uncovered = verify_vertex_cover(graph, 5, cover)
print(f"커버 집합: {cover}")         # 예: {0, 1, 3} 또는 {0, 3, 2} 등
print(f"유효한 커버: {valid}")        # True
print(f"커버 크기: {len(cover)}")     # OPT ≤ 커버 크기 ≤ 2 × OPT

# 2-근사 증명:
# OPT는 최대 매칭의 모든 에지를 커버해야 하므로 OPT ≥ |매칭|
# 우리의 커버 크기 = 2 × |매칭| ≤ 2 × OPT
```

### 왜 2-근사인가?

설정한 매칭 M의 크기를 |M|이라 하면:
1. 최적 해는 M의 모든 에지를 커버해야 하므로 OPT ≥ |M|
2. 우리 알고리즘의 결과 크기 = 2|M|

따라서 결과 크기 / OPT ≤ 2|M| / |M| = 2. 이것이 2-근사의 엄밀한 증명이다.

---

## 문제 2: 집합 커버 (Set Cover) — O(log n)-근사

**집합 커버(Set Cover)** 문제: 전체 집합 U와 부분집합 모음 S₁, S₂, ..., Sₘ이 주어질 때, 모든 원소를 커버하는 최소 개수의 부분집합 선택.

그리디 알고리즘은 O(log n) 근사 비율을 달성한다: 매 단계에서 가장 많은 새 원소를 커버하는 집합을 선택한다.

```python
def set_cover_greedy(universe, subsets):
    """
    그리디 집합 커버. O(log n) 근사.
    universe: 전체 원소의 집합
    subsets: List of frozensets
    반환: 선택된 부분집합들의 인덱스 리스트
    """
    uncovered = set(universe)
    selected = []
    available = list(enumerate(subsets))  # (index, subset) 쌍

    while uncovered:
        # 가장 많은 미커버 원소를 포함하는 집합 선택
        best_idx, best_set = max(
            ((i, s) for i, s in available if s & uncovered),
            key=lambda x: len(x[1] & uncovered),
            default=(None, None)
        )
        if best_idx is None:
            break  # 더 이상 커버 불가 (입력 오류)

        selected.append(best_idx)
        uncovered -= best_set
        available = [(i, s) for i, s in available if i != best_idx]

    return selected


def set_cover_with_cost(universe, subsets, costs):
    """
    비용 가중 집합 커버: 커버당 비용을 고려한 그리디
    각 단계에서 (커버 원소 수 / 비용) 비율이 가장 높은 집합 선택
    """
    uncovered = set(universe)
    selected = []
    total_cost = 0
    available = list(enumerate(zip(subsets, costs)))

    while uncovered:
        best_idx, best_data = max(
            ((i, d) for i, d in available if d[0] & uncovered),
            key=lambda x: len(x[1][0] & uncovered) / x[1][1],
            default=(None, None)
        )
        if best_idx is None:
            break

        best_set, best_cost = best_data
        selected.append(best_idx)
        total_cost += best_cost
        uncovered -= best_set
        available = [(i, d) for i, d in available if i != best_idx]

    return selected, total_cost


# 예시: 시험 테스트 케이스 최소화
# universe = 테스트해야 할 기능 번호
universe = {1, 2, 3, 4, 5, 6, 7, 8}
subsets = [
    frozenset({1, 2, 3, 4}),   # 테스트 스위트 0
    frozenset({3, 4, 5, 6}),   # 테스트 스위트 1
    frozenset({5, 6, 7, 8}),   # 테스트 스위트 2
    frozenset({1, 5, 7}),      # 테스트 스위트 3
    frozenset({2, 4, 6, 8}),   # 테스트 스위트 4
]

selected = set_cover_greedy(universe, subsets)
covered = set().union(*(subsets[i] for i in selected))
print(f"선택된 집합: {selected}")
print(f"커버된 원소: {covered}")
print(f"최적 해는 2개이지만 그리디는 최대 {len(selected)}개 선택")
# 집합 커버 OPT=2(예: {0,2})이지만 그리디가 최대 O(log 8)=3배 내

# 비용 가중 예시: 기지국 배치
universe2 = set(range(1, 9))
subsets2 = [frozenset({1,2,3}), frozenset({2,3,4,5}), frozenset({4,5,6,7,8})]
costs2 = [10, 15, 20]
sel, cost = set_cover_with_cost(universe2, subsets2, costs2)
print(f"\n기지국 선택: {sel}, 총 비용: {cost}")
```

### O(log n) 근사 증명 스케치

OPT개의 최적 집합이 n개의 원소를 커버한다고 하자. 그리디의 각 단계에서 최소한 (남은 원소 수 / OPT)개의 새 원소를 커버할 수 있다. 이 재귀적 관계를 풀면 그리디가 n개를 모두 커버하기까지 최대 OPT × ln(n) 단계가 필요함을 보일 수 있다.

---

## 문제 3: 외판원 문제 (TSP) — 2-근사와 Christofides 알고리즘

**외판원 문제(Traveling Salesman Problem, TSP)**는 n개 도시를 정확히 한 번씩 방문하고 출발 도시로 돌아오는 최소 비용 경로를 찾는 문제다. 일반적인 TSP는 근사하기 매우 어렵지만, **삼각 부등식(triangle inequality)**을 만족하는 Metric TSP는 2-근사 알고리즘이 존재한다.

```python
import heapq
from collections import defaultdict

def metric_tsp_2approx(dist_matrix):
    """
    Metric TSP의 2-근사 알고리즘 (MST 기반).
    1. 최소 신장 트리(MST) 구성 (Prim's algorithm)
    2. MST를 DFS pre-order로 순회
    3. 이미 방문한 도시는 건너뜀 (shortcut)
    
    이 알고리즘은 2-OPT 근사를 보장함.
    """
    n = len(dist_matrix)
    
    # Step 1: Prim's MST
    in_mst = [False] * n
    min_edge = [float('inf')] * n
    parent = [-1] * n
    min_edge[0] = 0
    
    mst_adj = defaultdict(list)
    heap = [(0, 0)]  # (비용, 정점)
    
    while heap:
        cost, u = heapq.heappop(heap)
        if in_mst[u]:
            continue
        in_mst[u] = True
        if parent[u] != -1:
            mst_adj[parent[u]].append(u)
            mst_adj[u].append(parent[u])
        
        for v in range(n):
            if not in_mst[v] and dist_matrix[u][v] < min_edge[v]:
                min_edge[v] = dist_matrix[u][v]
                parent[v] = u
                heapq.heappush(heap, (dist_matrix[u][v], v))
    
    mst_cost = sum(min_edge)
    
    # Step 2: DFS pre-order traversal
    visited = [False] * n
    tour = []
    
    def dfs(node):
        visited[node] = True
        tour.append(node)
        for neighbor in sorted(mst_adj[node]):
            if not visited[neighbor]:
                dfs(neighbor)
    
    import sys
    sys.setrecursionlimit(10000)
    dfs(0)
    tour.append(tour[0])  # 출발 도시로 복귀
    
    # Step 3: 투어 비용 계산
    total_cost = sum(
        dist_matrix[tour[i]][tour[i+1]]
        for i in range(len(tour) - 1)
    )
    
    return tour, total_cost, mst_cost


# 예시: 5개 도시 (0~4), 유클리드 거리 기반
import math

cities = [(0,0), (1,3), (4,3), (4,0), (2,1)]

def euclidean_dist(p1, p2):
    return math.sqrt((p1[0]-p2[0])**2 + (p1[1]-p2[1])**2)

n = len(cities)
dist = [[euclidean_dist(cities[i], cities[j]) for j in range(n)] for i in range(n)]

tour, cost, mst_cost = metric_tsp_2approx(dist)
print(f"투어 경로: {tour}")
print(f"투어 비용: {cost:.3f}")
print(f"MST 비용: {mst_cost:.3f}")
print(f"근사 비율 상한: 2 × MST 비용 = {2 * mst_cost:.3f}")
# 투어 비용 ≤ 2 × MST 비용 ≤ 2 × OPT
```

### 2-근사 증명 스케치

1. **MST 비용 ≤ OPT**: 최적 투어에서 에지 하나를 제거하면 신장 트리가 된다. MST는 최소 신장 트리이므로 MST ≤ OPT.
2. **DFS 순회의 비용 ≤ 2 × MST 비용**: MST의 각 에지를 정확히 두 번 traversal하면 오일러 순회(Eulerian circuit)가 된다. 그 비용 = 2 × MST.
3. **Shortcut이 비용을 늘리지 않음**: 삼각 부등식에 의해, 방문한 도시를 건너뛰는 shortcut은 비용을 증가시키지 않는다.
4. 결론: 알고리즘 비용 ≤ 2 × MST ≤ 2 × OPT.

### Christofides-Serdyukov 알고리즘 (1.5-근사)

1976년 Christofides(그리고 독립적으로 Serdyukov)가 발표한 알고리즘은 1.5-근사를 달성한다. MST 구성 후 홀수 차수 정점들끼리 **최소 완전 매칭(minimum perfect matching)**을 수행해 오일러 그래프를 만들고, shortcut으로 해밀턴 경로를 얻는다. 오랫동안 metric TSP 최선 근사였으나 2020년 Karlin, Klein, Oveis Gharan이 (3/2 - ε)-근사로 개선했다.

---

## 근사 불가능성 (Inapproximability)

모든 NP-난해 문제가 좋은 근사 비율을 가지는 것은 아니다.

- **일반 TSP (삼각 부등식 없음)**: P ≠ NP이면 임의의 상수 근사 비율조차 달성 불가. 만약 α-근사 알고리즘이 존재한다면 그것을 이용해 해밀턴 경로(Hamiltonian Path) 문제를 다항 시간에 풀 수 있기 때문이다.
- **집합 커버**: (1 - ε) × ln(n) 보다 좋은 근사는 P ≠ NP이면 불가 (Feige, 1998). 즉 그리디 알고리즘이 사실상 최선이다.
- **Clique 문제**: n^(1-ε) 근사도 P ≠ NP이면 불가.

---

## 주의사항과 실전 팁

### 1. 근사 비율은 최악의 경우 보장
근사 비율은 모든 입력에 대한 보장이다. 실제 입력에서는 훨씬 좋은 결과가 나올 수 있다.

### 2. 랜덤화 근사 알고리즘
일부 문제는 랜덤화를 통해 더 나은 기대 근사 비율을 달성한다. MAX-3SAT의 경우 단순 랜덤 할당이 7/8-근사를 보장한다.

### 3. FPTAS (완전 다항 시간 근사 체계)
배낭 문제(Knapsack)처럼 일부 NP-난해 문제는 임의의 ε > 0에 대해 (1+ε)-근사를 O(n/ε) 시간에 달성하는 **FPTAS**가 존재한다. 이는 실용적으로 거의 최적에 가까운 해를 얻을 수 있음을 의미한다.

### 4. LP 기반 근사 알고리즘
많은 근사 알고리즘은 선형 계획법(LP) 완화를 이용한다. LP를 풀어 최적의 분수 해를 구한 뒤 반올림(rounding)해 정수 해를 얻는다. LP 완화의 최적값은 OPT의 하한이 되어 근사 비율 증명에 활용된다.

### 5. 실무에서의 선택
실제 문제에서는 이론적 근사 비율 외에도 실행 시간, 구현 난이도, 평균적 성능이 중요하다. 예를 들어 TSP를 실무에서 풀 때는 근사 비율보다 빠른 휴리스틱(Lin-Kernighan, OR-Tools 등)을 사용하는 경우가 많다.

---

## 참고 자료

- [Approximation algorithm - Wikipedia](https://en.wikipedia.org/wiki/Approximation_algorithm)
- [Vertex cover - Wikipedia](https://en.wikipedia.org/wiki/Vertex_cover)
- [Set cover problem - Wikipedia](https://en.wikipedia.org/wiki/Set_cover_problem)
- [Travelling salesman problem - Wikipedia](https://en.wikipedia.org/wiki/Travelling_salesman_problem)
