---
layout: post
title: "위상 정렬(Topological Sort) 완전 정복: Kahn's 알고리즘과 DFS로 작업 순서 결정하기"
date: 2026-07-11
categories: [cs, computer-science]
tags: [graph, topological-sort, DAG, BFS, DFS, algorithms, scheduling]
---

## 위상 정렬이란?

위상 정렬(Topological Sort)은 **방향 비순환 그래프(DAG, Directed Acyclic Graph)**의 모든 정점을 간선의 방향을 거스르지 않도록 나열하는 알고리즘이다. 즉, 간선 u → v가 존재한다면 결과 순서에서 u는 반드시 v보다 앞에 위치해야 한다.

위상 정렬은 실생활 문제를 그래프로 모델링하는 가장 강력한 도구 중 하나다. 대학 수강 신청의 선수과목 체계, Make/Gradle 같은 빌드 시스템의 의존성 해결, CPU 명령어 스케줄링, 데이터 파이프라인 처리 순서 등 수없이 많은 문제가 위상 정렬로 해결된다.

### 전제 조건: DAG

위상 정렬은 **사이클이 없는 방향 그래프**에서만 가능하다. 사이클이 존재하면 "A가 B보다 먼저, B가 C보다 먼저, C가 A보다 먼저"와 같은 모순이 생겨 선형 순서를 정할 수 없다. 따라서 위상 정렬 알고리즘은 보통 사이클 검출을 내장하고 있다.

---

## 왜 위상 정렬이 필요한가?

### 빌드 시스템의 의존성 해결

```
A → B → D
A → C → D
```

위 의존성 그래프에서 D를 빌드하려면 B와 C가 먼저 빌드되어야 하고, B와 C를 빌드하려면 A가 먼저 빌드되어야 한다. 위상 정렬을 실행하면 `A → B → C → D` 또는 `A → C → B → D` 같은 유효한 빌드 순서가 나온다.

### 강의 수강 계획

대학 커리큘럼에서 알고리즘 수업을 들으려면 자료구조를 먼저 이수해야 하고, 자료구조를 들으려면 프로그래밍 기초를 먼저 이수해야 한다. 이런 선수과목 관계를 DAG로 표현하면 위상 정렬로 올바른 수강 순서를 결정할 수 있다.

### 스프레드시트 계산 순서

Excel 같은 스프레드시트에서 셀 A1이 B1에 의존하고 B1이 C1에 의존할 때, 올바른 계산 순서를 결정하는 데 위상 정렬이 사용된다.

---

## 알고리즘 1: Kahn's Algorithm (BFS 기반)

Kahn's 알고리즘은 **진입 차수(in-degree)**를 활용한 BFS 기반 접근법이다.

### 핵심 아이디어

1. 모든 정점의 진입 차수를 계산한다.
2. 진입 차수가 0인 정점(선행 조건이 없는 것)을 큐에 넣는다.
3. 큐에서 정점을 꺼내 결과에 추가하고, 해당 정점에서 나가는 간선을 제거(인접 정점의 진입 차수 감소)한다.
4. 진입 차수가 0이 된 정점을 큐에 추가한다.
5. 큐가 빌 때까지 반복. 결과 길이 < 전체 정점 수면 **사이클이 있다는 의미**다.

### Python 구현

```python
from collections import deque
from typing import List, Dict, Optional

def topological_sort_kahn(graph: Dict[int, List[int]], n: int) -> Optional[List[int]]:
    """
    Kahn's Algorithm: BFS 기반 위상 정렬
    graph: 인접 리스트 {정점: [인접 정점, ...]}
    n: 전체 정점 수 (0 ~ n-1)
    반환: 위상 정렬 결과 리스트, 사이클 존재 시 None
    """
    # 1단계: 진입 차수 계산
    in_degree = [0] * n
    for u in range(n):
        for v in graph.get(u, []):
            in_degree[v] += 1

    # 2단계: 진입 차수가 0인 정점을 큐에 삽입
    queue = deque()
    for i in range(n):
        if in_degree[i] == 0:
            queue.append(i)

    result = []

    # 3단계: BFS 탐색
    while queue:
        u = queue.popleft()
        result.append(u)

        for v in graph.get(u, []):
            in_degree[v] -= 1
            if in_degree[v] == 0:
                queue.append(v)

    # 4단계: 사이클 검출
    if len(result) != n:
        return None  # 사이클 존재
    return result


# 예제: 수강 신청 순서 결정
# 0: 프로그래밍 기초, 1: 자료구조, 2: 알고리즘, 3: 운영체제, 4: 데이터베이스
courses = {
    0: [1, 3],  # 프로그래밍 기초 → 자료구조, 운영체제
    1: [2],     # 자료구조 → 알고리즘
    2: [4],     # 알고리즘 → 데이터베이스
    3: [],
    4: [],
}

order = topological_sort_kahn(courses, 5)
names = ["프로그래밍 기초", "자료구조", "알고리즘", "운영체제", "데이터베이스"]
if order:
    print("수강 순서:", " → ".join(names[i] for i in order))
    # 출력: 수강 순서: 프로그래밍 기초 → 자료구조 → 운영체제 → 알고리즘 → 데이터베이스

# 사이클 예제
cycle_graph = {0: [1], 1: [2], 2: [0]}  # 0→1→2→0 사이클
result = topological_sort_kahn(cycle_graph, 3)
print("사이클 감지:", result is None)  # 출력: 사이클 감지: True
```

### Kahn's 알고리즘의 특징

- **시간 복잡도**: O(V + E) — 모든 정점과 간선을 한 번씩 처리
- **공간 복잡도**: O(V) — 진입 차수 배열과 큐
- **사이클 감지**: 결과 길이가 V보다 작으면 사이클 존재
- **결과의 유일성**: 큐에 여러 항목이 있을 경우 여러 유효한 순서가 가능

---

## 알고리즘 2: DFS 기반 위상 정렬

DFS(깊이 우선 탐색) 기반 위상 정렬은 재귀적으로 그래프를 탐색하면서, 더 이상 나아갈 곳이 없는 정점부터 스택에 쌓는 방식이다. 스택을 역순으로 읽으면 위상 정렬 결과가 된다.

### 핵심 아이디어

각 정점에는 세 가지 상태가 있다:
- **WHITE (0)**: 아직 방문하지 않음
- **GRAY (1)**: 현재 DFS 경로에 있음 (방문 중)
- **BLACK (2)**: 모든 후손을 처리 완료

GRAY 상태의 정점을 다시 방문하면 **후방 간선(back edge)**이 있다는 뜻, 즉 사이클이다.

### Python 구현

```python
from typing import List, Dict, Optional

def topological_sort_dfs(graph: Dict[int, List[int]], n: int) -> Optional[List[int]]:
    """
    DFS 기반 위상 정렬
    graph: 인접 리스트
    n: 전체 정점 수
    반환: 위상 정렬 결과 리스트, 사이클 존재 시 None
    """
    WHITE, GRAY, BLACK = 0, 1, 2
    color = [WHITE] * n
    result = []
    has_cycle = [False]

    def dfs(u: int):
        if has_cycle[0]:
            return
        color[u] = GRAY  # 방문 시작 (경로에 포함)

        for v in graph.get(u, []):
            if color[v] == GRAY:
                # GRAY 정점을 다시 방문 = 사이클
                has_cycle[0] = True
                return
            if color[v] == WHITE:
                dfs(v)

        color[u] = BLACK  # 모든 후손 처리 완료
        result.append(u)  # 완료된 정점을 결과에 추가

    for i in range(n):
        if color[i] == WHITE:
            dfs(i)

    if has_cycle[0]:
        return None

    result.reverse()  # 역순이 위상 정렬 순서
    return result


# 빌드 시스템 의존성 예제
# A=0, B=1, C=2, D=3, E=4
# D는 B와 C에 의존, B와 C는 A에 의존, E는 D에 의존
build_deps = {
    0: [1, 2],  # A → B, C
    1: [3],     # B → D
    2: [3],     # C → D
    3: [4],     # D → E
    4: [],
}

order = topological_sort_dfs(build_deps, 5)
labels = ['A', 'B', 'C', 'D', 'E']
if order:
    print("빌드 순서:", " → ".join(labels[i] for i in order))
    # 출력: 빌드 순서: A → C → B → D → E (또는 A → B → C → D → E)
```

---

## 두 알고리즘 비교

| 특성 | Kahn's Algorithm | DFS 기반 |
|------|-----------------|---------|
| 기반 탐색 | BFS | DFS |
| 사이클 감지 | 결과 길이 비교 | GRAY 상태 재방문 |
| 구현 복잡도 | 직관적 | 다소 복잡 |
| 메모리 | O(V) 큐 | O(V) 스택 (재귀) |
| 특징 | 병렬 처리에 적합 | 스택 오버플로 주의 |

---

## 심화: 모든 위상 정렬 순서 열거

때로는 모든 유효한 위상 정렬 순서를 찾아야 한다. 백트래킹으로 구현할 수 있다.

```python
from collections import deque
from typing import List, Dict

def all_topological_sorts(graph: Dict[int, List[int]], n: int) -> List[List[int]]:
    """
    모든 유효한 위상 정렬 순서를 열거
    경고: 지수 시간 복잡도, 소규모 그래프에만 사용
    """
    in_degree = [0] * n
    for u in range(n):
        for v in graph.get(u, []):
            in_degree[v] += 1

    results = []
    current = []
    visited = [False] * n

    def backtrack():
        # 진입 차수가 0이고 아직 방문하지 않은 정점 수집
        available = [i for i in range(n) if in_degree[i] == 0 and not visited[i]]

        if not available:
            if len(current) == n:
                results.append(current[:])
            return

        for u in available:
            # 선택
            visited[u] = True
            current.append(u)
            for v in graph.get(u, []):
                in_degree[v] -= 1

            backtrack()

            # 취소
            visited[u] = False
            current.pop()
            for v in graph.get(u, []):
                in_degree[v] += 1

    backtrack()
    return results


# 간단한 예제: 3개 정점, 1개 간선
# 0→2, 1은 독립
simple = {0: [2], 1: [], 2: []}
all_orders = all_topological_sorts(simple, 3)
print(f"유효한 순서 수: {len(all_orders)}")
for order in all_orders:
    print(order)
# [0, 1, 2], [0, 2, 1]은 유효하지 않고
# [0, 1, 2], [1, 0, 2] 등이 출력됨
```

---

## 실전 응용: 작업 스케줄링 시뮬레이터

```python
from collections import deque, defaultdict
from typing import List, Tuple, Dict

def schedule_tasks(tasks: List[str], deps: List[Tuple[str, str]]) -> List[List[str]]:
    """
    작업과 의존성을 받아 병렬 실행 가능한 단계(레벨)로 그룹화
    deps: [(선행 작업, 후행 작업), ...]
    반환: 각 단계에서 병렬 실행 가능한 작업 그룹
    """
    # 인덱스 매핑
    idx = {task: i for i, task in enumerate(tasks)}
    n = len(tasks)
    graph = defaultdict(list)
    in_degree = [0] * n

    for pre, post in deps:
        u, v = idx[pre], idx[post]
        graph[u].append(v)
        in_degree[v] += 1

    queue = deque([i for i in range(n) if in_degree[i] == 0])
    levels = []

    while queue:
        level_size = len(queue)
        level = []

        for _ in range(level_size):
            u = queue.popleft()
            level.append(tasks[u])
            for v in graph[u]:
                in_degree[v] -= 1
                if in_degree[v] == 0:
                    queue.append(v)

        levels.append(level)

    return levels


# CI/CD 파이프라인 예제
pipeline_tasks = ["코드 체크아웃", "단위 테스트", "통합 테스트", "도커 빌드", "스테이징 배포", "E2E 테스트", "프로덕션 배포"]
pipeline_deps = [
    ("코드 체크아웃", "단위 테스트"),
    ("코드 체크아웃", "통합 테스트"),
    ("단위 테스트", "도커 빌드"),
    ("통합 테스트", "도커 빌드"),
    ("도커 빌드", "스테이징 배포"),
    ("스테이징 배포", "E2E 테스트"),
    ("E2E 테스트", "프로덕션 배포"),
]

schedule = schedule_tasks(pipeline_tasks, pipeline_deps)
for i, level in enumerate(schedule, 1):
    print(f"단계 {i} (병렬 실행 가능): {level}")

# 출력:
# 단계 1 (병렬 실행 가능): ['코드 체크아웃']
# 단계 2 (병렬 실행 가능): ['단위 테스트', '통합 테스트']
# 단계 3 (병렬 실행 가능): ['도커 빌드']
# 단계 4 (병렬 실행 가능): ['스테이징 배포']
# 단계 5 (병렬 실행 가능): ['E2E 테스트']
# 단계 6 (병렬 실행 가능): ['프로덕션 배포']
```

이 레벨별 그룹화는 **BFS 기반 위상 정렬의 자연스러운 확장**이다. 같은 레벨의 작업들은 의존 관계가 없으므로 병렬로 실행할 수 있다.

---

## 주의사항과 팁

### 1. 사이클 처리

실제 시스템에서는 예상치 못한 사이클이 생길 수 있다. 사이클 감지 시 어떤 정점이 사이클을 형성하는지 찾아서 오류 메시지에 포함하면 디버깅이 쉬워진다.

```python
def find_cycle(graph: Dict[int, List[int]], n: int) -> List[int]:
    """사이클을 구성하는 정점 경로 반환"""
    WHITE, GRAY, BLACK = 0, 1, 2
    color = [WHITE] * n
    parent = [-1] * n

    def dfs(u: int) -> int:
        color[u] = GRAY
        for v in graph.get(u, []):
            if color[v] == GRAY:
                return v  # 사이클 시작 정점
            if color[v] == WHITE:
                parent[v] = u
                result = dfs(v)
                if result != -1:
                    return result
        color[u] = BLACK
        return -1

    for i in range(n):
        if color[i] == WHITE:
            cycle_start = dfs(i)
            if cycle_start != -1:
                # 사이클 경로 재구성
                cycle = [cycle_start]
                cur = parent[cycle_start]
                while cur != cycle_start:
                    cycle.append(cur)
                    cur = parent[cur]
                cycle.reverse()
                return cycle
    return []
```

### 2. 재귀 깊이 제한

Python의 기본 재귀 깊이는 1,000이다. 매우 큰 그래프에서 DFS 기반 알고리즘을 사용할 경우 `sys.setrecursionlimit()`으로 늘리거나, 명시적 스택을 사용하는 반복적 구현으로 바꿔야 한다.

### 3. 위상 정렬 순서의 비유일성

하나의 DAG에 여러 개의 유효한 위상 정렬 순서가 존재할 수 있다. 특정 순서를 보장하려면 우선순위 큐(최소/최대 힙)를 사용하면 된다.

```python
import heapq

def topological_sort_lexicographic(graph, n):
    """사전 순으로 가장 작은 위상 정렬 반환"""
    in_degree = [0] * n
    for u in range(n):
        for v in graph.get(u, []):
            in_degree[v] += 1

    heap = [i for i in range(n) if in_degree[i] == 0]
    heapq.heapify(heap)
    result = []

    while heap:
        u = heapq.heappop(heap)
        result.append(u)
        for v in sorted(graph.get(u, [])):
            in_degree[v] -= 1
            if in_degree[v] == 0:
                heapq.heappush(heap, v)

    return result if len(result) == n else None
```

### 4. 위상 정렬을 활용한 최단/최장 경로

DAG에서는 위상 정렬 순서로 정점을 처리하면서 DP를 적용하면 O(V+E)에 최단 경로(SSSP)와 최장 경로를 구할 수 있다. 일반 그래프의 O(E log V) 다익스트라보다 빠르다.

```python
def dag_longest_path(graph, weights, n):
    """
    DAG에서 위상 정렬 기반 최장 경로
    weights: {(u, v): 가중치}
    반환: 최장 경로 길이
    """
    order = topological_sort_kahn(graph, n)
    if order is None:
        return -1  # 사이클 존재

    dp = [0] * n
    for u in order:
        for v in graph.get(u, []):
            w = weights.get((u, v), 1)
            dp[v] = max(dp[v], dp[u] + w)

    return max(dp)
```

---

## 마치며

위상 정렬은 단순해 보이지만 실무에서 광범위하게 활용되는 필수 알고리즘이다. 빌드 시스템, 패키지 의존성 관리, 스프레드시트 계산, CPU 명령어 스케줄링, 데이터 파이프라인 등 "순서"가 중요한 문제라면 위상 정렬을 먼저 떠올려야 한다.

핵심은 두 가지다: 진입 차수 기반의 Kahn's 알고리즘은 직관적이고 병렬 처리에 자연스럽게 확장되며, DFS 기반은 기존 DFS 코드와 통합하기 쉽다. 두 알고리즘 모두 O(V+E)의 최적 시간 복잡도를 가지므로 상황에 맞게 선택하면 된다.

## 참고 자료
- [Topological Sorting - Wikipedia](https://en.wikipedia.org/wiki/Topological_sorting)
- [Topological Sorting - GeeksforGeeks](https://www.geeksforgeeks.org/dsa/topological-sorting/)
- [Kahn's Algorithm vs DFS - GeeksforGeeks](https://www.geeksforgeeks.org/dsa/kahns-algorithm-vs-dfs-approach-a-comparative-analysis/)
- [Topological Sort - cp-algorithms.com](https://cp-algorithms.com/graph/topological-sort.html)
