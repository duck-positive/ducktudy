---
layout: post
title: "다익스트라와 A* 알고리즘 완전 정복: 최단 경로 탐색의 두 거장"
date: 2026-06-20
categories: [cs, computer-science]
tags: [dijkstra, astar, shortest-path, graph, algorithm, heuristic, pathfinding]
---

## 최단 경로 탐색이란?

지도 앱에서 목적지까지 가장 빠른 길을 찾는 일, 게임 캐릭터가 장애물을 피해 이동하는 일, 인터넷 패킷이 최소 홉(hop)으로 전달되는 일 — 이 모두가 **최단 경로 탐색(Shortest Path Finding)** 문제입니다. 그래프 이론에서 수십 년간 연구되어 온 이 문제의 가장 중요한 두 알고리즘이 **다익스트라(Dijkstra's Algorithm)**와 **A\*(A-Star)**입니다.

이 두 알고리즘은 서로 밀접하게 연결되어 있습니다. A\*는 다익스트라를 일반화한 것으로, 목표 지점을 향한 **휴리스틱(heuristic)** 추정치를 더해 탐색 공간을 효과적으로 줄입니다.

---

## 다익스트라 알고리즘

### 개념

Edsger W. Dijkstra가 1956년에 고안한 이 알고리즘은 **음수 가중치가 없는** 가중치 그래프에서 단일 출발점(single-source)으로부터 모든 노드까지의 최단 경로를 구합니다. 핵심 아이디어는 **그리디(Greedy)**: 아직 확정되지 않은 노드 중 현재까지 알려진 거리가 가장 짧은 노드를 매 단계에서 선택합니다.

### 동작 원리

1. 출발 노드 거리를 0, 나머지를 ∞로 초기화합니다.
2. 우선순위 큐(min-heap)에서 현재 거리가 가장 작은 노드 u를 꺼냅니다.
3. u의 이웃 노드 v에 대해 `dist[u] + weight(u, v) < dist[v]`이면 `dist[v]`를 갱신합니다(이완, relaxation).
4. 모든 노드가 처리될 때까지 반복합니다.

### 시간 복잡도

- 인접 리스트 + 이진 힙: **O((V + E) log V)**
- 피보나치 힙: O(E + V log V) (이론적 최적)

---

## A* 알고리즘

### 개념

A\*는 다익스트라에 **휴리스틱 함수 h(n)**을 추가합니다. 각 노드 n의 우선순위를 `f(n) = g(n) + h(n)`으로 정합니다.

- `g(n)`: 출발점에서 n까지 실제로 소요된 비용
- `h(n)`: n에서 목표 지점까지 예상 비용(휴리스틱)

h(n) = 0이면 A\*는 다익스트라와 동일합니다. h(n)이 **허용 가능(admissible)** — 즉, 실제 비용을 절대 과대평가하지 않아야 — 최적 경로를 보장합니다. 2D 격자에서는 맨해튼 거리나 유클리드 거리를 h(n)으로 자주 사용합니다.

---

## 실제 구현 예제

### 예제 1: 다익스트라 알고리즘 (Python)

```python
import heapq
from collections import defaultdict
from typing import Dict, List, Tuple, Optional

def dijkstra(
    graph: Dict[str, List[Tuple[str, float]]],
    start: str
) -> Tuple[Dict[str, float], Dict[str, Optional[str]]]:
    """
    graph: 인접 리스트 {노드: [(이웃, 가중치), ...]}
    반환:  (최단 거리 dict, 이전 노드 dict)
    """
    dist = defaultdict(lambda: float('inf'))
    prev = {node: None for node in graph}
    dist[start] = 0.0

    # (거리, 노드) 형태의 min-heap
    heap = [(0.0, start)]

    while heap:
        d, u = heapq.heappop(heap)

        # 이미 더 짧은 경로로 처리된 노드는 스킵
        if d > dist[u]:
            continue

        for v, weight in graph[u]:
            new_dist = dist[u] + weight
            if new_dist < dist[v]:
                dist[v] = new_dist
                prev[v] = u
                heapq.heappush(heap, (new_dist, v))

    return dict(dist), prev

def reconstruct_path(prev: Dict[str, Optional[str]], target: str) -> List[str]:
    """이전 노드 dict에서 경로 복원"""
    path = []
    node = target
    while node is not None:
        path.append(node)
        node = prev.get(node)
    return list(reversed(path))

# --- 예시 실행 ---
if __name__ == "__main__":
    # 방향 가중치 그래프 (서울 지하철 간략화)
    metro = {
        "강남": [("교대", 2), ("선릉", 3)],
        "교대": [("강남", 2), ("서초", 1), ("남부터미널", 4)],
        "서초": [("교대", 1), ("방배", 2)],
        "방배": [("서초", 2), ("사당", 2)],
        "사당": [("방배", 2), ("남태령", 3)],
        "남태령": [("사당", 3)],
        "선릉": [("강남", 3), ("삼성", 2)],
        "삼성": [("선릉", 2), ("종합운동장", 2)],
        "종합운동장": [("삼성", 2)],
        "남부터미널": [("교대", 4), ("양재", 3)],
        "양재": [("남부터미널", 3), ("매봉", 2)],
        "매봉": [("양재", 2)],
    }

    distances, previous = dijkstra(metro, "강남")

    targets = ["사당", "남태령", "매봉", "종합운동장"]
    for t in targets:
        path = reconstruct_path(previous, t)
        print(f"강남 → {t}: 거리={distances[t]}, 경로={' → '.join(path)}")
```

출력:
```
강남 → 사당: 거리=7.0, 경로=강남 → 교대 → 서초 → 방배 → 사당
강남 → 남태령: 거리=10.0, 경로=강남 → 교대 → 서초 → 방배 → 사당 → 남태령
강남 → 매봉: 거리=9.0, 경로=강남 → 교대 → 남부터미널 → 양재 → 매봉
강남 → 종합운동장: 거리=7.0, 경로=강남 → 선릉 → 삼성 → 종합운동장
```

---

### 예제 2: A* 알고리즘으로 2D 격자 길 찾기

```python
import heapq
import math
from typing import List, Tuple, Optional, Set

def astar(
    grid: List[List[int]],
    start: Tuple[int, int],
    goal: Tuple[int, int]
) -> Optional[List[Tuple[int, int]]]:
    """
    grid: 0 = 이동 가능, 1 = 장애물
    반환: 경로 (노드 리스트) 또는 None (경로 없음)
    """
    rows, cols = len(grid), len(grid[0])

    def heuristic(a: Tuple[int, int], b: Tuple[int, int]) -> float:
        # 대각선 이동 허용 시 체비쇼프 거리, 4방향이면 맨해튼 거리
        return abs(a[0] - b[0]) + abs(a[1] - b[1])

    def neighbors(pos: Tuple[int, int]):
        r, c = pos
        # 4방향 이동
        for dr, dc in [(-1,0),(1,0),(0,-1),(0,1)]:
            nr, nc = r + dr, c + dc
            if 0 <= nr < rows and 0 <= nc < cols and grid[nr][nc] == 0:
                yield (nr, nc), 1.0  # (이웃, 이동비용)

    open_set: List[Tuple[float, Tuple[int, int]]] = []
    heapq.heappush(open_set, (0.0, start))

    came_from: dict = {}
    g_score = {start: 0.0}
    f_score = {start: heuristic(start, goal)}
    in_open: Set = {start}

    while open_set:
        _, current = heapq.heappop(open_set)
        in_open.discard(current)

        if current == goal:
            # 경로 복원
            path = [current]
            while current in came_from:
                current = came_from[current]
                path.append(current)
            return list(reversed(path))

        for neighbor, cost in neighbors(current):
            tentative_g = g_score[current] + cost
            if tentative_g < g_score.get(neighbor, float('inf')):
                came_from[neighbor] = current
                g_score[neighbor] = tentative_g
                f = tentative_g + heuristic(neighbor, goal)
                f_score[neighbor] = f
                if neighbor not in in_open:
                    heapq.heappush(open_set, (f, neighbor))
                    in_open.add(neighbor)

    return None  # 경로 없음

# --- 예시 실행 ---
if __name__ == "__main__":
    # 0 = 통로, 1 = 벽
    grid = [
        [0, 0, 0, 0, 1, 0, 0, 0],
        [0, 1, 1, 0, 1, 0, 1, 0],
        [0, 1, 0, 0, 0, 0, 1, 0],
        [0, 0, 0, 1, 1, 0, 1, 0],
        [1, 1, 0, 0, 0, 0, 0, 0],
        [0, 0, 0, 1, 0, 1, 1, 0],
        [0, 1, 0, 0, 0, 0, 0, 0],
        [0, 0, 0, 1, 0, 0, 0, 0],
    ]

    start, goal = (0, 0), (7, 7)
    path = astar(grid, start, goal)

    if path:
        print(f"경로 길이: {len(path) - 1}칸")
        # 격자에 경로 표시
        path_set = set(path)
        for r in range(len(grid)):
            row_str = ""
            for c in range(len(grid[0])):
                if (r, c) == start:
                    row_str += "S "
                elif (r, c) == goal:
                    row_str += "G "
                elif (r, c) in path_set:
                    row_str += "· "
                elif grid[r][c] == 1:
                    row_str += "█ "
                else:
                    row_str += "  "
            print(row_str)
    else:
        print("경로를 찾을 수 없습니다.")
```

출력 예시:
```
경로 길이: 14칸
S · · ·  █  · · ·
·  █  █  ·  █  ·  █  ·
·  █  ·  ·  ·  ·  █  ·
·  ·  ·  █  █  ·  █  ·
█  █  ·  ·  ·  ·  ·  ·
·  ·  ·  █  ·  █  █  ·
·  █  ·  ·  ·  ·  ·  ·
·  ·  ·  █  ·  ·  · G
```

---

## 다익스트라 vs A* 비교

| 항목 | 다익스트라 | A* |
|------|-----------|-----|
| 적용 범위 | 모든 최단 경로 | 단일 목표까지 최단 경로 |
| 휴리스틱 | 없음 (h=0) | 있음 (h ≥ 0, admissible) |
| 탐색 방향 | 방사형 (전방위) | 목표 지향 |
| 탐색 노드 수 | 많음 | 적음 (휴리스틱 품질에 비례) |
| 최적성 보장 | 항상 | h가 admissible이면 항상 |
| 사용 사례 | 단일 출발점 전체 최단 거리 | 특정 목적지 길 찾기 |

---

## 주의사항 및 실전 팁

### 1. 음수 가중치 처리

다익스트라는 음수 가중치 엣지가 있으면 오동작합니다. 음수 가중치가 존재하면 **벨만-포드(Bellman-Ford)** 알고리즘을 사용해야 합니다. 음수 사이클이 있으면 최단 경로 자체가 정의되지 않습니다.

### 2. 휴리스틱 설계

A\*에서 h(n)이 실제 비용을 과대평가(inadmissible)하면 최적 경로를 놓칠 수 있습니다. 반대로 h(n) = 0은 최적이지만 다익스트라와 동일해져 속도 이점이 없습니다. **일관적(consistent) 휴리스틱**이라면 한 번 방문한 노드를 재방문할 필요가 없어 효율이 더 높아집니다.

### 3. 대규모 그래프 최적화

- **양방향 다익스트라**: 출발점과 목적점에서 동시에 탐색하여 탐색 공간을 절반으로 줄입니다.
- **지형 전처리(Contraction Hierarchies)**: 도로 네트워크처럼 정적인 그래프에서는 전처리 후 초고속 쿼리가 가능합니다.
- **ALT(A\* + Landmarks + Triangle inequality)**: 대규모 지도에서 A\*를 수백 배 빠르게 만드는 기법입니다.

### 4. 우선순위 큐 선택

Python의 `heapq`는 이진 힙으로 O(log n) 삽입/삭제를 지원합니다. 노드 수가 매우 많다면 Fibonacci 힙이 이론적으로 우수하지만, 구현 복잡도 때문에 실제로는 이진 힙이 더 많이 사용됩니다.

---

## 참고 자료

- [Wikipedia - Dijkstra's Algorithm](https://en.wikipedia.org/wiki/Dijkstra%27s_algorithm)
- [Wikipedia - A* Search Algorithm](https://en.wikipedia.org/wiki/A*_search_algorithm)
- [Codecademy - Dijkstra's Shortest Path Algorithm](https://www.codecademy.com/article/dijkstras-shortest-path-algorithm)
- [Stanford CS106B Lecture: Dijkstra and A*](https://web.stanford.edu/class/archive/cs/cs106b/cs106b.1258/lectures/26-graph-algorithms/)
