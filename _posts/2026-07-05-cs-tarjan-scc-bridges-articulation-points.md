---
layout: post
title: "Tarjan's Algorithm 완전 정복: SCC, 브릿지, 단절점을 단 한 번의 DFS로 찾는 법"
date: 2026-07-05
categories: [cs, computer-science]
tags: [tarjan, SCC, strongly-connected-components, bridges, articulation-points, graph, dfs, low-link]
---

## 개요

1972년 Robert Tarjan이 발표한 알고리즘은 그래프 이론에서 손꼽히는 우아한 발명 중 하나다. **단 한 번의 DFS 순회**만으로 다음 세 가지를 모두 찾아낼 수 있다:

1. **강연결 요소(Strongly Connected Component, SCC)**: 방향 그래프에서 모든 노드 쌍 사이에 양방향 경로가 존재하는 최대 부분 그래프
2. **브릿지(Bridge / Cut Edge)**: 제거 시 그래프의 연결 요소 수가 증가하는 간선
3. **단절점(Articulation Point / Cut Vertex)**: 제거 시 그래프가 분리되는 정점

세 문제 모두 `disc[]`(발견 시각)와 `low[]`(도달 가능한 최소 발견 시각)라는 두 배열로 해결된다. 시간 복잡도는 모두 **O(V + E)** 선형이다.

---

## 왜 필요한가

### SCC: 순환 의존 탐지

패키지 관리자(npm, Maven, pip)의 의존성 그래프에서 순환 의존(A→B→C→A)을 탐지할 때 SCC가 사용된다. 하나의 SCC 내부에 있는 패키지들은 서로 강하게 결합되어 있음을 의미한다. Kosaraju 알고리즘도 같은 문제를 풀지만 DFS를 두 번 수행하는 반면, Tarjan은 한 번으로 끝낸다.

### 브릿지: 네트워크 취약점 분석

인터넷 라우팅 토폴로지에서 브릿지 간선은 단일 실패 지점(Single Point of Failure)이다. 이 간선이 끊기면 네트워크가 두 개의 섬으로 분리된다. ISP 망 설계, 사회 연결망 분석, 도로 네트워크 취약점 분석에서 핵심적으로 사용된다.

### 단절점: 인프라 가용성 분석

단절점인 노드가 장애를 일으키면 전체 서비스의 일부가 격리된다. 데이터센터 간 연결, 마이크로서비스 의존 그래프에서 단절점을 미리 식별해 이중화 설계를 할 수 있다.

---

## 핵심 개념: disc[]와 low[]

```
disc[v]: DFS 탐색 중 정점 v를 처음 방문한 시각 (타임스탬프)
low[v]:  v의 서브트리에서 back edge를 통해 도달할 수 있는
         정점들 중 최소 disc 값
```

`low[v]`의 갱신 규칙:
```
low[v] = min(
    disc[v],                  # 자기 자신
    low[w]  for tree edge (v→w),     # 자식 서브트리
    disc[w] for back edge (v→w)      # 역방향 간선
)
```

이 두 값만으로 SCC, 브릿지, 단절점을 모두 판별할 수 있다.

---

## 알고리즘 1: SCC (방향 그래프)

SCC 탐지에서는 스택(Stack)을 추가로 사용한다. 정점을 처음 방문할 때 스택에 넣고, DFS가 끝났을 때 `low[v] == disc[v]`이면 스택 상단에서 v까지 팝하여 하나의 SCC를 구성한다.

### Python 구현

```python
from collections import defaultdict

class TarjanSCC:
    def __init__(self, n: int):
        self.n = n
        self.graph = defaultdict(list)

    def add_edge(self, u: int, v: int):
        self.graph[u].append(v)

    def find_sccs(self) -> list[list[int]]:
        disc = [-1] * self.n
        low = [0] * self.n
        on_stack = [False] * self.n
        stack = []
        sccs = []
        timer = [0]

        def dfs(v: int):
            disc[v] = low[v] = timer[0]
            timer[0] += 1
            stack.append(v)
            on_stack[v] = True

            for w in self.graph[v]:
                if disc[w] == -1:          # tree edge
                    dfs(w)
                    low[v] = min(low[v], low[w])
                elif on_stack[w]:          # back edge (스택 위에 있는 정점만)
                    low[v] = min(low[v], disc[w])

            # SCC의 루트 판별
            if low[v] == disc[v]:
                scc = []
                while True:
                    w = stack.pop()
                    on_stack[w] = False
                    scc.append(w)
                    if w == v:
                        break
                sccs.append(scc)

        for i in range(self.n):
            if disc[i] == -1:
                dfs(i)

        return sccs


# 예시: 0→1→2→0 (하나의 SCC), 3→4 (두 개의 SCC)
tarjan = TarjanSCC(5)
for u, v in [(0,1),(1,2),(2,0),(1,3),(3,4)]:
    tarjan.add_edge(u, v)

sccs = tarjan.find_sccs()
print("SCCs:", sccs)
# SCCs: [[4], [3], [0, 2, 1]]  (역방향 출력, 위상 정렬 역순)
```

### SCC의 위상 정렬 활용

SCC를 하나의 슈퍼노드(super-node)로 압축하면 방향 그래프가 DAG(Directed Acyclic Graph)가 된다. 이 DAG의 위상 정렬은 프로그램 분석, 컴파일러 의존성 해석, 작업 스케줄링에 직접 사용된다.

---

## 알고리즘 2: 브릿지와 단절점 (무방향 그래프)

무방향 그래프에서는 back edge 처리 시 **부모 방향 간선을 제외**해야 한다.

판별 조건:
- **브릿지**: `low[w] > disc[v]`이면 간선 `(v, w)`는 브릿지
- **단절점**: 루트라면 자식 수 ≥ 2, 비루트라면 `low[w] >= disc[v]`

### Python 구현

```python
from collections import defaultdict

class TarjanBridgeArticulation:
    def __init__(self, n: int):
        self.n = n
        self.graph = defaultdict(list)

    def add_edge(self, u: int, v: int):
        self.graph[u].append(v)
        self.graph[v].append(u)

    def find_bridges_and_articulations(self):
        disc = [-1] * self.n
        low = [0] * self.n
        parent = [-1] * self.n
        bridges = []
        articulations = set()
        timer = [0]

        def dfs(v: int):
            disc[v] = low[v] = timer[0]
            timer[0] += 1
            child_count = 0

            for w in self.graph[v]:
                if disc[w] == -1:
                    child_count += 1
                    parent[w] = v
                    dfs(w)
                    low[v] = min(low[v], low[w])

                    # 브릿지 판별
                    if low[w] > disc[v]:
                        bridges.append((v, w))

                    # 단절점 판별 (비루트)
                    if parent[v] != -1 and low[w] >= disc[v]:
                        articulations.add(v)

                elif w != parent[v]:  # back edge (부모 방향 제외)
                    low[v] = min(low[v], disc[w])

            # 단절점 판별 (루트: 자식이 2개 이상)
            if parent[v] == -1 and child_count >= 2:
                articulations.add(v)

        for i in range(self.n):
            if disc[i] == -1:
                dfs(i)

        return bridges, articulations


# 예시 그래프: 0-1-2-3, 1-3 (1은 단절점이 아님), 2-4 (2가 단절점, 1-2는 브릿지?)
g = TarjanBridgeArticulation(5)
for u, v in [(0,1),(1,2),(2,3),(3,4)]:
    g.add_edge(u, v)

bridges, articulations = g.find_bridges_and_articulations()
print("브릿지:", bridges)
print("단절점:", articulations)
# 브릿지: [(0,1),(1,2),(2,3),(3,4)]  — 선형 그래프는 모든 간선이 브릿지
# 단절점: {1, 2, 3}
```

---

## 알고리즘 3: 반복(Iterative) Tarjan — 재귀 깊이 제한 우회

Python 기본 재귀 깊이 제한(`sys.setrecursionlimit` 기본값 1000)이 문제가 될 수 있다. 노드가 수십만 개인 그래프에서는 반복적(iterative) 구현이 필요하다.

```python
def find_sccs_iterative(graph: dict, n: int) -> list[list[int]]:
    disc = [-1] * n
    low = [0] * n
    on_stack = [False] * n
    stack = []
    sccs = []
    timer = [0]

    for start in range(n):
        if disc[start] != -1:
            continue

        # (정점, 이웃 반복자 인덱스)
        call_stack = [(start, 0)]
        disc[start] = low[start] = timer[0]
        timer[0] += 1
        stack.append(start)
        on_stack[start] = True

        while call_stack:
            v, idx = call_stack[-1]
            neighbors = list(graph.get(v, []))

            if idx < len(neighbors):
                call_stack[-1] = (v, idx + 1)
                w = neighbors[idx]

                if disc[w] == -1:
                    disc[w] = low[w] = timer[0]
                    timer[0] += 1
                    stack.append(w)
                    on_stack[w] = True
                    call_stack.append((w, 0))
                elif on_stack[w]:
                    low[v] = min(low[v], disc[w])
            else:
                call_stack.pop()
                if call_stack:
                    parent_v = call_stack[-1][0]
                    low[parent_v] = min(low[parent_v], low[v])

                if low[v] == disc[v]:
                    scc = []
                    while True:
                        w = stack.pop()
                        on_stack[w] = False
                        scc.append(w)
                        if w == v:
                            break
                    sccs.append(scc)

    return sccs
```

---

## 시각화: low[] 값이 변하는 과정

다음 그래프를 예로 들자:
```
0 → 1 → 2 → 0  (사이클)
1 → 3 → 4
```

DFS 순서: 0, 1, 2, (back edge 2→0 처리), 3, 4

| 정점 | disc[] | low[] (최종) |
|---|---|---|
| 0 | 0 | 0 |
| 1 | 1 | 0 |
| 2 | 2 | 0 |
| 3 | 3 | 3 |
| 4 | 4 | 4 |

`low[1] = 0`이지만 `disc[1] = 1`이므로 1은 SCC 루트가 아니다.
`low[0] = 0 = disc[0]`이므로 0이 SCC 루트 → {0, 2, 1} 팝.
`low[3] = 3 = disc[3]` → {3}, `low[4] = 4 = disc[4]` → {4}.

---

## 주의사항 및 팁

### 1. 간선 중복 (멀티그래프)
무방향 그래프에서 같은 두 노드 사이에 간선이 2개 이상 있으면(멀티엣지) 부모 방향 간선을 `parent[v] != w` 조건으로 걸러낼 수 없다. 이 경우 **간선 ID**를 기준으로 부모 간선을 추적해야 한다.

### 2. 재귀 깊이
Python에서는 `sys.setrecursionlimit`를 올리거나 위에서 보인 iterative 버전을 사용한다. C++이라면 기본 스택이 충분히 크므로 재귀 구현으로도 대부분 문제없다.

### 3. Kosaraju vs Tarjan
두 알고리즘 모두 O(V+E)지만:
- Kosaraju: DFS 2회, 구현이 직관적, 역방향 그래프 필요
- Tarjan: DFS 1회, 구현이 조금 복잡, 추가 공간(스택) 필요

### 4. Condensation Graph 구성
SCC를 찾은 뒤 각 SCC를 노드로 압축한 DAG를 구성하면 2-SAT, 최장 경로, 위상 정렬 등 수많은 문제의 전처리 단계로 활용된다.

---

## 실전 활용 사례

- **컴파일러**: 프로시저 간 호출 그래프에서 순환 호출 탐지(인라인 최적화 불가 영역 식별)
- **패키지 의존성**: npm `--detect-cycles`, Gradle의 순환 의존 경고
- **게임 맵**: 이동 가능한 구역 분리(단절점 = 좁은 길목 = 전략적 요충지)
- **소셜 네트워크 분석**: 커뮤니티 탐지, 브릿지 유저 식별
- **2-SAT 풀이**: SCC 압축 후 변수와 그 부정이 같은 SCC에 있으면 UNSAT 판별

---

## 정리

Tarjan's Algorithm은 `disc[]`와 `low[]` 두 배열과 DFS 단 한 번으로 SCC, 브릿지, 단절점을 모두 찾는 선형 시간 알고리즘이다. 핵심은 low[v] 갱신 규칙과 SCC 루트 판별 조건(`low[v] == disc[v]`)이다. 방향 그래프에서는 스택을 활용한 SCC 분리, 무방향 그래프에서는 부모 방향 간선 처리 방식이 다르다는 점을 구분해서 이해하면 모든 변형 문제에 대응할 수 있다.

## 참고 자료
- [GeeksforGeeks: Tarjan's Algorithm to find Strongly Connected Components](https://www.geeksforgeeks.org/dsa/tarjan-algorithm-find-strongly-connected-components/)
- [Codeforces Blog: Articulation points and bridges (Tarjan's Algorithm)](https://codeforces.com/blog/entry/71146)
- [Wikipedia: Tarjan's strongly connected components algorithm](https://en.wikipedia.org/wiki/Tarjan%27s_strongly_connected_components_algorithm)
- [LeetCode The Hard Way: Tarjan's Algorithm](https://leetcodethehardway.com/tutorials/graph-theory/tarjans-algorithm)
