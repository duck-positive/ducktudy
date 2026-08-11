---
layout: post
title: "오일러 경로·회로 완전 정복: Hierholzer 알고리즘으로 모든 간선을 한 번에 순회하기"
date: 2026-08-11
categories: [cs, computer-science]
tags: [graph, euler, hierholzer, algorithm, dfs, competitive-programming]
---

## 개념 설명

1736년, 수학자 레온하르트 오일러(Leonhard Euler)는 프로이센의 쾨니히스베르크(Königsberg) 도시를 흐르는 프레겔(Pregel) 강과 일곱 개의 다리에 관한 유명한 문제를 풀었습니다. "모든 다리를 정확히 한 번씩 건너 출발점으로 돌아올 수 있는가?" 라는 이 질문에 오일러는 불가능하다는 것을 증명하면서 **그래프 이론**이라는 학문의 문을 열었습니다.

이 문제에서 탄생한 두 가지 핵심 개념이 바로 **오일러 경로(Eulerian Path)**와 **오일러 회로(Eulerian Circuit)**입니다.

- **오일러 경로**: 그래프의 모든 간선을 정확히 한 번씩 방문하는 경로
- **오일러 회로**: 오일러 경로 중에서 시작 정점과 끝 정점이 같은 경로

오일러 경로와 오일러 회로는 **해밀턴 경로(Hamiltonian Path, 모든 정점을 한 번씩 방문)**와 혼동하기 쉬우나, 핵심 차이는 "정점" 방문이 아니라 "간선" 방문이라는 점입니다. 해밀턴 경로는 NP-완전 문제이지만, 오일러 경로는 선형 시간(O(V + E))에 풀 수 있다는 점에서 알고리즘적으로 매우 흥미롭습니다.

### 존재 조건

**무방향 그래프(Undirected Graph)**:
- **오일러 회로** 존재 조건: 그래프가 연결되어 있고, 모든 정점의 차수(degree)가 짝수
- **오일러 경로** 존재 조건: 그래프가 연결되어 있고, 홀수 차수를 가진 정점이 정확히 2개

홀수 차수 정점 2개가 있다면, 그 중 하나가 시작점, 다른 하나가 끝점이 됩니다.

**방향 그래프(Directed Graph)**:
- **오일러 회로** 존재 조건: 그래프가 강연결(strongly connected)되어 있고, 모든 정점에서 진입 차수(in-degree) = 진출 차수(out-degree)
- **오일러 경로** 존재 조건: 그래프가 약연결(weakly connected)되어 있고, 정확히 하나의 정점에서 out-degree = in-degree + 1 (시작점), 정확히 하나의 정점에서 in-degree = out-degree + 1 (끝점), 나머지는 in-degree = out-degree

---

## 왜 필요한가

오일러 경로·회로 알고리즘은 실생활과 공학에서 다양하게 응용됩니다.

1. **DNA 서열 조립(DNA Sequencing)**: 차세대 염기서열 분석(Next Generation Sequencing)에서 단편 서열을 이어 붙이는 de Bruijn 그래프 기반 조립이 오일러 경로 문제로 환원됩니다. Velvet, SPAdes 같은 도구가 이 원리를 사용합니다.

2. **중국인 우체부 문제(Chinese Postman Problem)**: 모든 도로를 최소 비용으로 순회하는 배달 경로 최적화 문제는 오일러 회로의 변형입니다.

3. **인쇄 회로 기판(PCB) 최적화**: 드릴 헤드가 모든 구멍 위치를 최소 이동으로 방문하는 경로를 계산할 때 사용됩니다.

4. **경쟁 프로그래밍**: LeetCode 332번(행정 구역 재건), 753번(잠금 해제 순열), ICPC 문제 등 다수의 고난이도 문제가 오일러 경로로 환원됩니다.

5. **필기 인식 및 경로 계획**: 한 획으로 그릴 수 있는 도형 판별이나, 네트워크 링크를 모두 점검하는 순회 경로 설계에도 쓰입니다.

---

## 실제 구현 예제

### 예제 1: Hierholzer 알고리즘 (방향 그래프 오일러 경로)

Hierholzer 알고리즘(1873년)은 현재까지 가장 효율적인 오일러 경로 탐색 알고리즘입니다. 핵심 아이디어는 다음과 같습니다:

1. 시작 정점에서 DFS를 수행하며 사용한 간선을 제거한다.
2. 더 이상 갈 곳이 없는 정점을 스택(결과)에 push한다.
3. 스택을 역순으로 출력하면 오일러 경로가 완성된다.

이 방법은 역방향으로 경로를 쌓기 때문에 "역순 DFS 후 뒤집기" 패턴이라고도 불립니다.

```python
from collections import defaultdict

def find_euler_path(n: int, edges: list[tuple[int, int]]) -> list[int]:
    """
    Hierholzer 알고리즘으로 방향 그래프의 오일러 경로를 반환합니다.
    존재하지 않으면 빈 리스트를 반환합니다.

    시간 복잡도: O(V + E)
    공간 복잡도: O(V + E)
    """
    graph = defaultdict(list)
    in_deg = defaultdict(int)
    out_deg = defaultdict(int)

    for u, v in edges:
        graph[u].append(v)
        out_deg[u] += 1
        in_deg[v] += 1

    # 시작점 결정: 오일러 경로가 존재하는지 확인
    start = -1
    nodes = set()
    for u, v in edges:
        nodes.add(u)
        nodes.add(v)

    diff_pos = []  # out > in (시작 후보)
    diff_neg = []  # in > out (끝 후보)

    for node in nodes:
        diff = out_deg[node] - in_deg[node]
        if diff == 1:
            diff_pos.append(node)
        elif diff == -1:
            diff_neg.append(node)
        elif diff != 0:
            return []  # 오일러 경로 불가

    if len(diff_pos) == 0 and len(diff_neg) == 0:
        # 오일러 회로: 임의의 시작점
        start = next(iter(nodes))
    elif len(diff_pos) == 1 and len(diff_neg) == 1:
        # 오일러 경로
        start = diff_pos[0]
    else:
        return []  # 오일러 경로 불가

    # Hierholzer DFS (반복적 구현 - 재귀 스택 오버플로 방지)
    stack = [start]
    path = []

    while stack:
        v = stack[-1]
        if graph[v]:
            # 아직 미사용 간선이 있으면 다음 정점으로
            u = graph[v].pop()
            stack.append(u)
        else:
            # 더 이상 갈 곳 없으면 경로에 추가
            path.append(stack.pop())

    # 오일러 경로가 맞는지 검증 (모든 간선 사용 여부)
    if len(path) != len(edges) + 1:
        return []

    return path[::-1]


# ===================== 테스트 =====================
# LeetCode 332: 항공권 재건 문제와 동일한 구조
# [JFK -> SFO, JFK -> ATL, SFO -> ATL, ATL -> JFK, ATL -> SFO]
airports = {'JFK': 0, 'SFO': 1, 'ATL': 2}
idx = {v: k for k, v in airports.items()}

edges = [(0, 1), (0, 2), (1, 2), (2, 0), (2, 1)]  # JFK->SFO, JFK->ATL, ...
result = find_euler_path(3, edges)
print("오일러 경로:", [idx[i] for i in result])
# 출력: ['JFK', 'ATL', 'JFK', 'SFO', 'ATL', 'SFO']

# 오일러 회로 테스트 (정육각형)
hex_edges = [(0,1),(1,2),(2,3),(3,4),(4,5),(5,0)]
result2 = find_euler_path(6, hex_edges)
print("오일러 회로:", result2)
# 출력: [0, 1, 2, 3, 4, 5, 0]
```

### 예제 2: 무방향 그래프 오일러 회로 (연결성 검사 포함)

무방향 그래프에서는 간선을 양방향으로 처리해야 하며, Fleury 알고리즘보다 Hierholzer가 훨씬 효율적입니다.

```python
from collections import defaultdict

class UndirectedEulerGraph:
    """
    무방향 그래프에서 오일러 회로 또는 오일러 경로를 찾습니다.
    """
    def __init__(self, n: int):
        self.n = n
        # 멀티셋 대신 딕셔너리를 사용해 간선 삭제를 O(1)로 처리
        self.adj = defaultdict(dict)  # adj[u][v] = 간선 개수 (중복 허용)
        self.edge_count = 0

    def add_edge(self, u: int, v: int):
        self.adj[u][v] = self.adj[u].get(v, 0) + 1
        self.adj[v][u] = self.adj[v].get(u, 0) + 1
        self.edge_count += 1

    def _dfs_euler(self, start: int) -> list[int]:
        stack = [start]
        path = []

        while stack:
            v = stack[-1]
            # 아직 사용하지 않은 인접 정점 탐색
            found = False
            for u in list(self.adj[v].keys()):
                if self.adj[v][u] > 0:
                    # 간선 사용: 양방향 모두 감소
                    self.adj[v][u] -= 1
                    self.adj[u][v] -= 1
                    if self.adj[v][u] == 0:
                        del self.adj[v][u]
                        del self.adj[u][v]
                    stack.append(u)
                    found = True
                    break
            if not found:
                path.append(stack.pop())

        return path[::-1]

    def euler_circuit(self) -> list[int]:
        """오일러 회로: 모든 정점의 차수가 짝수여야 함"""
        nodes = set(self.adj.keys())
        for node in nodes:
            degree = sum(self.adj[node].values())
            if degree % 2 != 0:
                raise ValueError(f"정점 {node}의 차수({degree})가 홀수 — 오일러 회로 불가")

        # 임의 시작점에서 시작
        start = next(iter(nodes))
        path = self._dfs_euler(start)

        if len(path) != self.edge_count + 1:
            raise ValueError("그래프가 연결되어 있지 않습니다")
        return path

    def euler_path(self) -> list[int]:
        """오일러 경로: 홀수 차수 정점이 정확히 2개여야 함"""
        nodes = set(self.adj.keys())
        odd_nodes = [n for n in nodes if sum(self.adj[n].values()) % 2 != 0]

        if len(odd_nodes) == 0:
            return self.euler_circuit()
        elif len(odd_nodes) != 2:
            raise ValueError(f"홀수 차수 정점이 {len(odd_nodes)}개 — 오일러 경로 불가")

        start = odd_nodes[0]  # 홀수 차수 정점 중 하나에서 시작
        path = self._dfs_euler(start)

        if len(path) != self.edge_count + 1:
            raise ValueError("그래프가 연결되어 있지 않습니다")
        return path


# ===================== 테스트 =====================
g = UndirectedEulerGraph(5)
# 아래 그래프는 홀수 차수 정점이 0, 2이므로 오일러 회로 존재
#   0-1-2-3-4-0 (오각형) + 0-2, 1-3 대각선 추가
g.add_edge(0, 1); g.add_edge(1, 2); g.add_edge(2, 3)
g.add_edge(3, 4); g.add_edge(4, 0); g.add_edge(0, 2); g.add_edge(1, 3)

try:
    circuit = g.euler_circuit()
    print(f"오일러 회로 ({len(circuit)-1}개 간선):", circuit)
except ValueError as e:
    print("오류:", e)

# 오일러 경로 테스트 (홀수 차수 정점 2개 생성)
g2 = UndirectedEulerGraph(4)
g2.add_edge(0, 1); g2.add_edge(1, 2); g2.add_edge(2, 3); g2.add_edge(0, 2)
path = g2.euler_path()
print(f"오일러 경로 ({len(path)-1}개 간선):", path)
```

---

## de Bruijn 수열과 오일러 경로

오일러 경로의 가장 아름다운 응용 중 하나는 **de Bruijn 수열(de Bruijn Sequence)**입니다.

길이 k인 모든 이진 문자열을 정확히 한 번씩 부분 문자열로 포함하는 최단 순환 문자열을 구하는 문제입니다. 예를 들어 k=3이면 8개의 이진 문자열(000, 001, 010, ..., 111)을 모두 포함하는 최단 순환 문자열은 `00010111`입니다.

이를 de Bruijn 그래프로 모델링하면:
- 정점: 길이 k-1인 모든 이진 문자열
- 간선 `u → v`: u의 마지막 k-2 비트와 v의 첫 k-2 비트가 같을 때 (shift 연결)

이 그래프에서 오일러 회로를 찾으면 de Bruijn 수열이 완성됩니다! 방향 그래프의 모든 정점이 in-degree = out-degree = 2이므로 항상 오일러 회로가 존재합니다.

---

## 주의사항 및 팁

### 1. Fleury 알고리즘 대신 Hierholzer를 사용하라
Fleury 알고리즘은 다리(bridge) 간선을 피하며 경로를 구성해 이해하기 쉽지만, 매 단계마다 다리 탐색에 O(E)가 필요해 전체 복잡도가 O(E²)입니다. Hierholzer는 O(V + E)로 훨씬 빠릅니다.

### 2. 재귀 DFS의 스택 오버플로 주의
간선이 수십만 개인 경우 재귀 깊이가 Python의 기본 재귀 제한(1000)을 초과합니다. 위 코드처럼 명시적 스택을 사용한 반복 구현을 권장합니다.

### 3. 멀티 그래프(Multi-graph) 처리
같은 두 정점 사이에 여러 간선이 있는 경우(멀티 그래프)를 반드시 고려해야 합니다. 인접 리스트에서 간선 수를 카운트하거나, 각 간선에 고유 ID를 부여해 사용 여부를 별도 배열로 관리하세요.

### 4. 연결성 전처리
오일러 경로 존재 조건을 만족해도, 실제로 해당 간선들이 연결된 컴포넌트에 있지 않으면 오일러 경로가 존재하지 않습니다. 차수 조건 확인 전에 BFS/DFS로 연결성을 먼저 검사하는 것을 권장합니다.

### 5. 시작 정점 선택
오일러 회로를 구할 때는 임의의 정점에서 시작해도 되지만, 오일러 경로에서는 반드시 out-degree = in-degree + 1인 정점(방향 그래프) 또는 홀수 차수 정점(무방향 그래프) 중 하나에서 시작해야 합니다.

---

## 참고 자료

- [Eulerian Path — Wikipedia](https://en.wikipedia.org/wiki/Eulerian_path)
- [Hierholzer's Algorithm — cp-algorithms.com](https://cp-algorithms.com/graph/euler_path.html)
- [De Bruijn Sequence — Wikipedia](https://en.wikipedia.org/wiki/De_Bruijn_sequence)
- [Euler Archive — Königsberg Bridge Problem](https://scholarlycommons.pacific.edu/euler-works/53/)
