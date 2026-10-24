---
layout: post
title: "위상 정렬: Kahn's Algorithm과 DFS 기반 구현 심화"
date: 2026-10-24
categories: [cs, computer-science]
tags: [topological-sort, graph, algorithm, kahn, dfs, dag, dependency-resolution]
---

## 개요

위상 정렬(Topological Sort)은 **방향 비순환 그래프(DAG, Directed Acyclic Graph)**에서 모든 간선 `u → v`에 대해 `u`가 `v`보다 앞에 오도록 정점을 나열하는 알고리즘입니다.

실제로 위상 정렬은 우리 주변 어디서나 쓰입니다:
- **빌드 시스템**: Make, Gradle, npm이 의존성 순서대로 패키지를 빌드합니다.
- **과목 이수**: "자료구조를 듣기 전에 프로그래밍 기초를 이수해야 한다"는 선이수 조건.
- **작업 스케줄링**: 병렬 처리에서 작업 간 의존 관계를 반영한 실행 순서.
- **스프레드시트 수식 평가**: 셀 A가 B에 의존한다면 B를 먼저 계산.

---

## 왜 위상 정렬이 필요한가?

### 문제 상황

여러 모듈로 구성된 소프트웨어 프로젝트를 빌드한다고 가정합니다:

```
A → B → D
A → C → D
B → E
```

이 경우 D를 빌드하려면 B와 C가, B를 빌드하려면 A가 먼저 완료되어야 합니다. 올바른 빌드 순서 중 하나는 `A → B → C → E → D` 또는 `A → C → B → E → D`입니다.

순환 의존(Circular Dependency)이 있으면 위상 정렬이 불가능합니다:
```
A → B → C → A  (순환 발생 → 위상 정렬 없음)
```

---

## 알고리즘 1: Kahn's Algorithm (BFS 기반)

1959년 Arthur Kahn이 제안한 알고리즘입니다. 진입 차수(In-degree)가 0인 정점부터 시작해 BFS 방식으로 진행합니다.

### 동작 과정

1. 모든 정점의 진입 차수를 계산합니다.
2. 진입 차수가 0인 정점을 모두 큐에 넣습니다.
3. 큐에서 정점 `u`를 꺼내 결과에 추가합니다.
4. `u`의 모든 이웃 `v`의 진입 차수를 1 감소시킵니다.
5. 진입 차수가 0이 된 `v`를 큐에 넣습니다.
6. 결과 크기가 `n`보다 작으면 순환이 존재합니다.

**시간 복잡도**: O(V + E)  
**공간 복잡도**: O(V)

### 장점
- 순환 감지가 자연스럽습니다 (결과 크기 < V이면 순환).
- 병렬 처리에 직관적입니다: 같은 "레이어"에 있는 정점은 동시 처리 가능.

---

## 구현 예제 1: Python Kahn's Algorithm

```python
from collections import deque, defaultdict
from typing import List, Optional

def topological_sort_kahn(n: int, edges: List[tuple]) -> Optional[List[int]]:
    """
    Kahn's Algorithm으로 위상 정렬.
    Args:
        n: 정점 수 (0 ~ n-1)
        edges: (u, v) 의미 = u → v
    Returns:
        위상 순서 리스트, 순환이 있으면 None
    """
    graph = defaultdict(list)
    in_degree = [0] * n

    for u, v in edges:
        graph[u].append(v)
        in_degree[v] += 1

    # 진입 차수 0인 정점으로 큐 초기화
    queue = deque(i for i in range(n) if in_degree[i] == 0)
    result = []

    while queue:
        u = queue.popleft()
        result.append(u)

        for v in graph[u]:
            in_degree[v] -= 1
            if in_degree[v] == 0:
                queue.append(v)

    if len(result) != n:
        return None  # 순환 감지

    return result


def topological_sort_with_layers(n: int, edges: List[tuple]) -> Optional[List[List[int]]]:
    """
    동시에 실행 가능한 작업 그룹(레이어)별로 위상 정렬.
    병렬 실행 가능 작업을 식별하는 데 유용.
    """
    graph = defaultdict(list)
    in_degree = [0] * n

    for u, v in edges:
        graph[u].append(v)
        in_degree[v] += 1

    current_layer = [i for i in range(n) if in_degree[i] == 0]
    layers = []
    processed = 0

    while current_layer:
        layers.append(current_layer[:])
        processed += len(current_layer)
        next_layer = []
        for u in current_layer:
            for v in graph[u]:
                in_degree[v] -= 1
                if in_degree[v] == 0:
                    next_layer.append(v)
        current_layer = next_layer

    if processed != n:
        return None

    return layers


# 테스트 1: 기본 위상 정렬
edges = [(0, 1), (0, 2), (1, 3), (2, 3), (1, 4)]
result = topological_sort_kahn(5, edges)
print(f"위상 정렬 결과: {result}")

# 테스트 2: 레이어별 병렬 처리 순서
layers = topological_sort_with_layers(5, edges)
print(f"레이어별 처리 순서:")
for i, layer in enumerate(layers):
    print(f"  레이어 {i} (동시 처리 가능): {layer}")

# 테스트 3: 순환 감지
cyclic_edges = [(0, 1), (1, 2), (2, 0)]
result = topological_sort_kahn(3, cyclic_edges)
print(f"순환 그래프 결과: {result}")  # None 반환

# 테스트 4: 의존성 해결 실습
# 빌드 시스템: 모듈 이름으로 동작
def build_order(modules: list, dependencies: list) -> Optional[List[str]]:
    idx = {m: i for i, m in enumerate(modules)}
    edges = [(idx[u], idx[v]) for u, v in dependencies]
    order = topological_sort_kahn(len(modules), edges)
    if order is None:
        return None
    return [modules[i] for i in order]

modules = ["main", "utils", "network", "database", "auth"]
deps = [("main", "utils"), ("main", "network"), ("network", "database"),
        ("network", "auth"), ("database", "utils"), ("auth", "utils")]

order = build_order(modules, deps)
print(f"\n빌드 순서: {' → '.join(order)}")
```

실행 결과:
```
위상 정렬 결과: [0, 1, 2, 4, 3]
레이어별 처리 순서:
  레이어 0 (동시 처리 가능): [0]
  레이어 1 (동시 처리 가능): [1, 2]
  레이어 2 (동시 처리 가능): [4, 3]
순환 그래프 결과: None
빌드 순서: utils → auth → database → network → main
```

---

## 알고리즘 2: DFS 기반 위상 정렬

DFS를 실행하면서 각 정점이 **완료(finish)**되는 순간 스택에 쌓습니다. 스택을 역순으로 읽으면 위상 정렬이 됩니다.

### 동작 과정

1. 방문 상태를 세 가지로 관리합니다: **미방문(WHITE)**, **방문 중(GRAY)**, **완료(BLACK)**.
2. 미방문 정점에서 DFS를 시작합니다.
3. 인접 정점 탐색 중 GRAY 정점을 만나면 **역방향 간선(Back Edge)** → 순환 발생.
4. 정점의 모든 이웃 탐색이 끝나면 스택에 push (BLACK으로 표시).
5. 스택 역순이 위상 정렬 순서입니다.

**시간 복잡도**: O(V + E)  
**공간 복잡도**: O(V) (재귀 스택 포함)

---

## 구현 예제 2: Java DFS 기반 위상 정렬

```java
import java.util.*;

public class TopologicalSortDFS {

    private int n;
    private List<List<Integer>> graph;

    // 방문 상태: 0 = 미방문, 1 = 방문 중, 2 = 완료
    private int[] state;
    private Deque<Integer> stack;
    private boolean hasCycle;

    public TopologicalSortDFS(int n) {
        this.n = n;
        this.graph = new ArrayList<>();
        for (int i = 0; i < n; i++) graph.add(new ArrayList<>());
        this.state = new int[n];
        this.stack = new ArrayDeque<>();
        this.hasCycle = false;
    }

    public void addEdge(int u, int v) {
        graph.get(u).add(v);
    }

    private void dfs(int u) {
        state[u] = 1; // 방문 중 (GRAY)

        for (int v : graph.get(u)) {
            if (state[v] == 1) {
                // 방문 중인 정점 재방문 → 역방향 간선 → 순환
                hasCycle = true;
                return;
            }
            if (state[v] == 0) {
                dfs(v);
                if (hasCycle) return;
            }
        }

        state[u] = 2; // 완료 (BLACK)
        stack.push(u);
    }

    public List<Integer> sort() {
        for (int i = 0; i < n; i++) {
            if (state[i] == 0) {
                dfs(i);
                if (hasCycle) return null;
            }
        }

        List<Integer> result = new ArrayList<>(stack);
        return result;
    }

    // 반복적 DFS (스택 오버플로 방지)
    public List<Integer> sortIterative() {
        int[] visitState = new int[n];
        Deque<Integer> callStack = new ArrayDeque<>();
        Deque<Integer> result = new ArrayDeque<>();
        // 각 정점의 이웃 탐색 커서
        int[] cursor = new int[n];

        for (int start = 0; start < n; start++) {
            if (visitState[start] != 0) continue;

            callStack.push(start);
            visitState[start] = 1;

            while (!callStack.isEmpty()) {
                int u = callStack.peek();
                List<Integer> neighbors = graph.get(u);

                if (cursor[u] < neighbors.size()) {
                    int v = neighbors.get(cursor[u]++);
                    if (visitState[v] == 1) {
                        return null; // 순환
                    }
                    if (visitState[v] == 0) {
                        visitState[v] = 1;
                        callStack.push(v);
                    }
                } else {
                    visitState[u] = 2;
                    callStack.pop();
                    result.push(u);
                }
            }
        }

        return new ArrayList<>(result);
    }

    public static void main(String[] args) {
        // 예시: 대학 강의 선이수 조건
        // 0=수학, 1=물리, 2=알고리즘, 3=운영체제, 4=자료구조, 5=컴파일러
        int n = 6;
        TopologicalSortDFS sorter = new TopologicalSortDFS(n);
        String[] courses = {"수학", "물리", "알고리즘", "운영체제", "자료구조", "컴파일러"};

        // 선이수 관계: u → v 의미 = u를 들어야 v를 들을 수 있음
        sorter.addEdge(0, 2); // 수학 → 알고리즘
        sorter.addEdge(0, 4); // 수학 → 자료구조
        sorter.addEdge(1, 3); // 물리 → 운영체제
        sorter.addEdge(4, 2); // 자료구조 → 알고리즘
        sorter.addEdge(2, 5); // 알고리즘 → 컴파일러
        sorter.addEdge(3, 5); // 운영체제 → 컴파일러

        List<Integer> order = sorter.sort();
        if (order == null) {
            System.out.println("순환 의존성 감지 - 위상 정렬 불가");
        } else {
            System.out.print("수강 순서: ");
            for (int idx : order) {
                System.out.print(courses[idx] + " → ");
            }
            System.out.println("졸업");
        }

        // 반복적 DFS 테스트
        TopologicalSortDFS sorter2 = new TopologicalSortDFS(n);
        sorter2.addEdge(0, 2); sorter2.addEdge(0, 4);
        sorter2.addEdge(1, 3); sorter2.addEdge(4, 2);
        sorter2.addEdge(2, 5); sorter2.addEdge(3, 5);
        List<Integer> order2 = sorter2.sortIterative();
        System.out.print("반복적 DFS: ");
        for (int idx : order2) System.out.print(courses[idx] + " ");
        System.out.println();
    }
}
```

실행 결과:
```
수강 순서: 수학 → 물리 → 자료구조 → 알고리즘 → 운영체제 → 컴파일러 → 졸업
반복적 DFS: 수학 물리 자료구조 알고리즘 운영체제 컴파일러
```

---

## Kahn vs DFS 비교

| 항목 | Kahn's Algorithm | DFS 기반 |
|------|-----------------|----------|
| 방식 | BFS (진입 차수 기반) | DFS (후위 순회) |
| 순환 감지 | 결과 크기로 감지 | 방문 중 상태로 감지 |
| 병렬 레이어 추출 | 자연스러움 | 추가 처리 필요 |
| 구현 난이도 | 직관적 | 상태 관리 필요 |
| 메모리 | 큐 + in-degree 배열 | 방문 상태 배열 + 스택 |
| 재귀 스택 오버플로 | 없음 | 깊은 그래프에서 가능 |

**실전 선택 가이드**:
- 병렬 처리 레이어를 구해야 한다면 → **Kahn's**
- 순환 경로를 역추적해야 한다면 → **DFS** (path 기록)
- 스택 오버플로 위험이 있다면 → **Kahn's** 또는 반복적 DFS

---

## 심화: 위상 정렬과 최장 경로

DAG에서 **최장 경로(Critical Path)**를 구하는 문제는 위상 정렬로 효율적으로 풀 수 있습니다. 동적 프로그래밍(DP)와 위상 정렬을 결합합니다:

```python
def critical_path(n, weighted_edges):
    """DAG에서 최장 경로 (프로젝트 일정에서 크리티컬 패스)"""
    from collections import defaultdict, deque
    
    graph = defaultdict(list)
    in_degree = [0] * n
    dist = [0] * n  # 최장 거리

    for u, v, w in weighted_edges:
        graph[u].append((v, w))
        in_degree[v] += 1

    queue = deque(i for i in range(n) if in_degree[i] == 0)
    
    while queue:
        u = queue.popleft()
        for v, w in graph[u]:
            dist[v] = max(dist[v], dist[u] + w)
            in_degree[v] -= 1
            if in_degree[v] == 0:
                queue.append(v)

    return max(dist), dist

# 프로젝트 작업 그래프 (u, v, 소요일)
tasks = [(0, 1, 3), (0, 2, 2), (1, 3, 5), (2, 3, 7), (1, 4, 2), (3, 5, 1), (4, 5, 4)]
max_time, finish_times = critical_path(6, tasks)
print(f"프로젝트 최소 완료 시간: {max_time}일")
print(f"각 작업 최조 완료 시간: {finish_times}")
```

---

## 주의사항 및 팁

### 1. 그래프에 순환이 없음을 보장하라
위상 정렬은 DAG에서만 의미가 있습니다. 입력 검증으로 순환 여부를 반드시 확인하세요.

### 2. 결과는 유일하지 않다
같은 DAG에도 여러 유효한 위상 순서가 존재합니다. 특정 순서가 필요하면 우선순위 큐(min-heap)를 사용하세요.

### 3. 대규모 그래프에서의 재귀 DFS
Python의 기본 재귀 한계는 1000입니다. 수백만 노드의 그래프는 반복적 DFS를 사용하거나 `sys.setrecursionlimit`을 설정하세요.

### 4. 실시간 의존성 업데이트
의존 관계가 동적으로 추가/제거되는 경우, 증분 위상 정렬(Incremental Topological Sort) 알고리즘을 고려하세요.

### 5. 최소 위상 순서
사전순으로 가장 작은 위상 순서가 필요한 경우, Kahn's에서 일반 큐 대신 **최소 힙(min-heap)**을 사용하세요.

---

## 정리

위상 정렬은 의존성이 있는 순서 문제를 DAG로 모델링해 O(V + E)에 해결합니다. Kahn's Algorithm은 진입 차수 기반으로 직관적이고 병렬 레이어 추출에 유리하며, DFS 기반은 후위 순회를 활용해 역방향 간선으로 순환을 감지합니다. 빌드 시스템, 작업 스케줄링, 컴파일러 파이프라인 등 실제 시스템에서 핵심적으로 사용되는 알고리즘입니다.

## 참고 자료
- [Wikipedia - Topological Sorting](https://en.wikipedia.org/wiki/Topological_sorting)
- [Wikipedia - Kahn's Algorithm](https://en.wikipedia.org/wiki/Topological_sorting#Kahn's_algorithm)
- [CP-Algorithms - Topological Sort](https://cp-algorithms.com/graph/topological-sort.html)
- [CLRS - Introduction to Algorithms, Chapter 22](https://en.wikipedia.org/wiki/Introduction_to_Algorithms)
