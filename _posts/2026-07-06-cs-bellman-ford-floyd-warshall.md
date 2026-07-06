---
layout: post
title: "Bellman-Ford와 Floyd-Warshall: 음수 가중치와 전체 쌍 최단 경로 완전 정복"
date: 2026-07-06
categories: [cs, computer-science]
tags: [bellman-ford, floyd-warshall, shortest-path, graph-algorithm, negative-cycle, dynamic-programming, competitive-programming]
---

다익스트라(Dijkstra) 알고리즘은 최단 경로 분야의 명성 높은 알고리즘이지만, 하나의 결정적 한계를 가집니다: **음수 가중치 간선이 있으면 동작하지 않습니다**. 그리고 다익스트라는 한 출발점에서의 최단 경로만 구합니다. 이 두 한계를 해결하는 알고리즘이 **Bellman-Ford**와 **Floyd-Warshall**입니다. 이 두 알고리즘은 그래프 알고리즘의 핵심이며, 라우팅 프로토콜(RIP, BGP), 통화 차익 거래 감지, 게임 패스파인딩 등 실전에서 폭넓게 사용됩니다.

---

## 1. 왜 다익스트라는 음수 간선에서 실패하는가

다익스트라는 **그리디 접근**을 사용합니다: 현재 가장 거리가 짧은 정점을 선택하고, 그 정점을 통해 이웃을 업데이트합니다. 이미 확정된 정점은 다시 방문하지 않습니다.

```
그래프:  A → B (가중치: 1)
         A → C (가중치: 4)
         B → C (가중치: -3)

다익스트라 실행:
1. A 선택, dist[B]=1, dist[C]=4
2. B 선택 (최소 1), dist[C] = min(4, 1 + (-3)) = -2 ← 올바름
3. C 선택 ← 이미 확정? 아님! B를 통한 경로로 dist[C]가 바뀌었어도
   다익스트라는 이미 B를 처리했으므로 더 이상 업데이트 없음

(단, 이 예시에서는 우연히 동작함. 더 복잡한 경우엔 오답)
```

음수 간선이 있으면 "짧은 거리 = 확정"이라는 그리디의 불변식이 깨집니다. **Bellman-Ford**는 이 불변식 없이도 동작합니다.

---

## 2. Bellman-Ford 알고리즘

### 2.1 핵심 아이디어: 반복 완화(Iterative Relaxation)

Bellman-Ford의 핵심 통찰은 단순합니다: **V-1개의 간선을 포함하는 최단 경로가 존재한다면, V-1번의 완화로 반드시 발견된다.**

**완화(Relaxation):** 간선 (u, v, w)에 대해 `dist[v] > dist[u] + w`이면 `dist[v] = dist[u] + w`로 갱신합니다.

**V-1번 반복하는 이유:** V개의 정점이 있는 그래프에서 단순 경로(사이클 없는)의 최대 길이는 V-1개의 간선입니다. 1번 반복은 최대 1개의 간선을 사용하는 최단 경로를 구합니다. k번 반복은 최대 k개의 간선을 사용하는 최단 경로를 보장합니다.

### 2.2 Python 구현

```python
from typing import List, Tuple, Dict, Optional
import math

def bellman_ford(
    vertices: int,
    edges: List[Tuple[int, int, float]],  # (출발, 도착, 가중치)
    source: int
) -> Tuple[Optional[Dict[int, float]], Optional[List[int]]]:
    """
    Bellman-Ford 알고리즘
    Returns:
        (dist, negative_cycle): 음수 사이클이 있으면 cycle 경로 반환, 없으면 None
    """
    INF = math.inf
    dist = {v: INF for v in range(vertices)}
    dist[source] = 0
    predecessor = {v: None for v in range(vertices)}

    # V-1번 완화
    for i in range(vertices - 1):
        updated = False
        for u, v, w in edges:
            if dist[u] != INF and dist[u] + w < dist[v]:
                dist[v] = dist[u] + w
                predecessor[v] = u
                updated = True
        if not updated:  # 조기 종료 최적화
            break

    # 음수 사이클 감지: V번째 반복에서도 업데이트가 일어나면 음수 사이클 존재
    negative_cycle_vertex = None
    for u, v, w in edges:
        if dist[u] != INF and dist[u] + w < dist[v]:
            negative_cycle_vertex = v
            break

    if negative_cycle_vertex is None:
        return dist, None  # 음수 사이클 없음, 최단 거리 반환

    # 음수 사이클 경로 추출
    # V번 이동해서 반드시 사이클 안에 들어간 정점 찾기
    x = negative_cycle_vertex
    for _ in range(vertices):
        x = predecessor[x]

    # 사이클 추출
    cycle = []
    start = x
    cycle.append(start)
    x = predecessor[x]
    while x != start:
        cycle.append(x)
        x = predecessor[x]
    cycle.append(start)
    cycle.reverse()

    return None, cycle  # 음수 사이클 존재


# 사용 예시 1: 음수 간선 있는 그래프
vertices = 5
edges = [
    (0, 1, 6),
    (0, 2, 7),
    (1, 2, 8),
    (1, 3, 5),
    (1, 4, -4),
    (2, 3, -3),
    (2, 4, 9),
    (3, 1, -2),
    (4, 0, 2),
    (4, 3, 7),
]

dist, cycle = bellman_ford(vertices, edges, source=0)
if cycle:
    print(f"음수 사이클 발견: {' → '.join(map(str, cycle))}")
else:
    print("최단 거리:", dist)
    # 출력: {0: 0, 1: 2, 2: 7, 3: 4, 4: -2}


# 사용 예시 2: 음수 사이클 있는 그래프
edges_with_neg_cycle = [
    (0, 1, 1),
    (1, 2, -1),
    (2, 3, -1),
    (3, 1, -1),  # 1→2→3→1 음수 사이클!
]

dist2, cycle2 = bellman_ford(4, edges_with_neg_cycle, source=0)
if cycle2:
    print(f"음수 사이클: {' → '.join(map(str, cycle2))}")
    # 출력: 음수 사이클: 1 → 2 → 3 → 1
```

### 2.3 실전 응용: 통화 차익거래(Arbitrage) 감지

금융 시장에서 통화 환율이 순환적으로 유리하면 아무 위험 없이 이익을 얻는 차익거래가 가능합니다. 이를 음수 사이클 문제로 변환할 수 있습니다.

```python
import math

def detect_arbitrage(currencies: List[str], rates: List[List[float]]) -> Optional[List[str]]:
    """
    통화 차익거래 기회 감지
    rates[i][j] = i통화 1단위로 j통화를 살 수 있는 양
    아이디어: log 변환 후 음수 사이클 = 차익거래 기회
    """
    n = len(currencies)
    # log 변환: 곱셈 → 덧셈, 역수: 이득 → 음수 비용
    # 차익거래: rate[i1][i2] * rate[i2][i3] * rate[i3][i1] > 1
    # log 변환: log(r12) + log(r23) + log(r31) > 0
    # 음수 사이클: -log(r12) + (-log(r23)) + (-log(r31)) < 0
    edges = []
    for i in range(n):
        for j in range(n):
            if i != j and rates[i][j] > 0:
                weight = -math.log(rates[i][j])
                edges.append((i, j, weight))

    # 가상의 소스 정점 추가 (모든 통화에서 0 비용으로 접근 가능)
    for i in range(n):
        edges.append((n, i, 0))

    _, cycle = bellman_ford(n + 1, edges, source=n)

    if cycle is None:
        return None  # 차익거래 없음

    # 사이클을 통화명으로 변환
    return [currencies[v] for v in cycle if v < n]


# 예시: USD, EUR, GBP 환율
currencies = ["USD", "EUR", "GBP"]
rates = [
    [1.0,  0.85, 0.73],  # USD → (USD, EUR, GBP)
    [1.18, 1.0,  0.86],  # EUR
    [1.37, 1.17, 1.0 ],  # GBP
]

# 차익거래 기회: USD→EUR→GBP→USD = 1 * 0.85 * 0.86 * 1.37 = 1.001...
arbitrage = detect_arbitrage(currencies, rates)
if arbitrage:
    print(f"차익거래 기회: {' → '.join(arbitrage)}")
```

---

## 3. Floyd-Warshall 알고리즘

### 3.1 모든 쌍 최단 경로 문제

Bellman-Ford는 단일 출발점(Single-Source) 최단 경로를 구합니다. V개의 정점에 대해 Bellman-Ford를 V번 실행하면 모든 쌍 최단 경로를 구할 수 있지만, 시간 복잡도는 O(V² × E)입니다.

**Floyd-Warshall**은 동적 프로그래밍으로 O(V³)에 모든 쌍 최단 경로를 구합니다.

### 3.2 핵심 점화식

```
dp[i][j][k] = 정점 {0, 1, ..., k}만을 경유지로 사용할 때 i→j 최단 거리

점화식:
dp[i][j][k] = min(
    dp[i][j][k-1],          # k를 경유하지 않음
    dp[i][k][k-1] + dp[k][j][k-1]  # k를 경유
)

초기값:
dp[i][j][−1] = 직접 간선의 가중치 (없으면 ∞, i==j이면 0)
```

k 차원은 생략 가능(현재 k만 의존): dp[i][j]로 2D 배열로 구현.

### 3.3 Python 구현

```python
import math
from typing import List, Tuple, Optional

def floyd_warshall(
    n: int,
    edges: List[Tuple[int, int, float]]
) -> Tuple[List[List[float]], List[List[Optional[int]]]]:
    """
    Floyd-Warshall 알고리즘
    Returns:
        dist[i][j]: i에서 j까지의 최단 거리
        next[i][j]: 최단 경로에서 i 다음 정점
    """
    INF = math.inf

    # 초기화
    dist = [[INF] * n for _ in range(n)]
    nxt  = [[None] * n for _ in range(n)]

    for i in range(n):
        dist[i][i] = 0

    for u, v, w in edges:
        if w < dist[u][v]:
            dist[u][v] = w
            nxt[u][v] = v

    # Floyd-Warshall 핵심 3중 루프
    for k in range(n):          # 경유 정점
        for i in range(n):      # 출발 정점
            for j in range(n):  # 도착 정점
                if dist[i][k] != INF and dist[k][j] != INF:
                    new_dist = dist[i][k] + dist[k][j]
                    if new_dist < dist[i][j]:
                        dist[i][j] = new_dist
                        nxt[i][j] = nxt[i][k]

    # 음수 사이클 감지: dist[i][i] < 0인 정점 존재
    has_negative_cycle = any(dist[i][i] < 0 for i in range(n))

    return dist, nxt, has_negative_cycle


def reconstruct_path(nxt: List[List[Optional[int]]], src: int, dst: int) -> Optional[List[int]]:
    """최단 경로 재구성"""
    if nxt[src][dst] is None:
        return None  # 경로 없음

    path = [src]
    cur = src
    while cur != dst:
        cur = nxt[cur][dst]
        if cur is None:
            return None
        path.append(cur)
        if len(path) > len(nxt):  # 음수 사이클 보호
            return None

    return path


# 사용 예시: 도시 간 최단 경로
# 0:서울, 1:대전, 2:대구, 3:부산
city_names = ["서울", "대전", "대구", "부산"]
city_edges = [
    (0, 1, 140),   # 서울 → 대전 140km
    (1, 0, 140),   # 대전 → 서울
    (1, 2, 120),   # 대전 → 대구 120km
    (2, 1, 120),
    (2, 3, 87),    # 대구 → 부산 87km
    (3, 2, 87),
    (0, 2, 300),   # 서울 → 대구 직접
    (2, 0, 300),
]

dist, nxt, has_neg_cycle = floyd_warshall(4, city_edges)

print("=== 모든 쌍 최단 거리 (km) ===")
print(f"{'':6}", end="")
for name in city_names:
    print(f"{name:8}", end="")
print()

for i, row_name in enumerate(city_names):
    print(f"{row_name:6}", end="")
    for j in range(4):
        d = dist[i][j]
        print(f"{'∞':8}" if d == math.inf else f"{d:8.0f}", end="")
    print()

print()
# 서울 → 부산 최단 경로
path = reconstruct_path(nxt, 0, 3)
path_names = [city_names[v] for v in path]
print(f"서울 → 부산 최단 경로: {' → '.join(path_names)}")
print(f"총 거리: {dist[0][3]:.0f}km")

# 출력:
# 서울 → 부산 최단 경로: 서울 → 대전 → 대구 → 부산
# 총 거리: 347km
```

---

## 4. 두 알고리즘 비교와 선택 기준

| 항목 | Bellman-Ford | Floyd-Warshall |
|------|-------------|----------------|
| **문제 유형** | 단일 출발점 최단 경로 | 모든 쌍 최단 경로 |
| **시간 복잡도** | O(V × E) | O(V³) |
| **공간 복잡도** | O(V) | O(V²) |
| **음수 간선** | ✅ 처리 가능 | ✅ 처리 가능 |
| **음수 사이클 감지** | ✅ 가능 | ✅ 가능 |
| **경로 재구성** | predecessor 배열 | next 배열 |
| **밀집 그래프** | 느림 (E 큼) | 적합 |
| **희소 그래프** | 적합 | 느림 |

### 언제 무엇을 쓸까?

- **Bellman-Ford:** 단일 출발점, 음수 간선, 음수 사이클 감지, 희소 그래프
- **Floyd-Warshall:** 모든 쌍, 밀집 그래프 (V ≤ 1000), 구현 간결함

---

## 5. 실전 응용과 최적화 기법

### 5.1 Johnson's Algorithm: 희소 그래프에서의 모든 쌍 최단 경로

모든 쌍 최단 경로를 구하면서도 희소 그래프에서 Floyd-Warshall보다 빠른 알고리즘입니다:

1. Bellman-Ford로 재가중치(reweighting) 함수 h[v] 계산
2. 간선 가중치를 `w'(u,v) = w(u,v) + h[u] - h[v]`로 변환 (모두 비음수)
3. V번의 다익스트라 실행

시간 복잡도: O(V × E × log V) — 희소 그래프(E ≈ V)에서 Floyd-Warshall보다 빠름

### 5.2 네트워크 라우팅: RIP와 Bellman-Ford

**RIP(Routing Information Protocol)**는 Bellman-Ford의 분산 버전입니다:

```
각 라우터가 이웃에게 자신의 거리 벡터를 주기적으로 광고
이웃의 정보를 받아 자신의 라우팅 테이블 갱신
→ "거리 벡터 라우팅(Distance Vector Routing)"
```

**카운트 투 인피니티 문제:** 링크 장애 시 잘못된 경로가 무한 루프로 증가합니다. RIP는 hop 수 16을 무한대로 정의하여 이를 제한합니다.

```python
# 분산 Bellman-Ford (RIP 개념 시뮬레이션)
class Router:
    def __init__(self, router_id: int, neighbors: Dict[int, float]):
        self.id = router_id
        self.dist = {router_id: 0.0}  # 자신까지의 거리
        self.next_hop = {router_id: router_id}
        self.neighbors = neighbors  # {neighbor_id: link_cost}

    def receive_advertisement(self, from_router: int, their_dist: Dict[int, float]):
        """이웃 라우터의 거리 벡터 수신 및 테이블 갱신"""
        updated = False
        link_cost = self.neighbors.get(from_router, math.inf)

        for dest, their_dist_to_dest in their_dist.items():
            new_dist = link_cost + their_dist_to_dest
            if new_dist < self.dist.get(dest, math.inf):
                self.dist[dest] = new_dist
                self.next_hop[dest] = from_router
                updated = True

        return updated

    def advertise(self) -> Dict[int, float]:
        """자신의 거리 벡터 광고"""
        return dict(self.dist)
```

---

## 6. 알고리즘 선택 완전 가이드

```
음수 가중치 있나?
 ├─ 아니오 → 다익스트라 사용 (O(E log V))
 └─ 예
     ├─ 음수 사이클 감지 필요? → Bellman-Ford
     ├─ 단일 출발점? → Bellman-Ford
     └─ 모든 쌍 필요?
          ├─ 밀집 그래프 (E ≈ V²)? → Floyd-Warshall
          └─ 희소 그래프 (E ≈ V)?  → Johnson's Algorithm
```

---

## 참고 자료

- [CP-Algorithms — Bellman-Ford Algorithm](https://cp-algorithms.com/graph/bellman_ford.html)
- [CP-Algorithms — Floyd-Warshall Algorithm](https://cp-algorithms.com/graph/all-pair-shortest-path-floyd-warshall.html)
- [USACO Guide — Shortest Paths with Negative Weights](https://usaco.guide/adv/sp-neg)
- [Stanford CS97SI — Shortest Path Algorithms (PDF)](https://web.stanford.edu/class/cs97si/07-shortest-path-algorithms.pdf)
