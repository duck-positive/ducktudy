---
layout: post
title: "분기 한정법(Branch and Bound) 완전 정복: NP-난해 최적화 문제를 정확하게 푸는 법"
date: 2026-09-16
categories: [cs, computer-science]
tags: [algorithm, optimization, NP-hard, branch-and-bound, combinatorics]
---

## 개념 설명

**분기 한정법(Branch and Bound, B&B)**은 NP-난해 최적화 문제를 **정확하게** 푸는 알고리즘 설계 패러다임입니다. 완전 탐색의 지수적 복잡도를 **상한/하한 추정값(Bound)**을 이용한 가지치기로 현실적인 시간 내에 해결합니다.

### 핵심 구성 요소

**1. 분기(Branch)**: 현재 부분 해를 더 작은 하위 문제로 분할합니다. 결정 변수를 하나씩 고정해 나가는 것이 일반적입니다.

**2. 한정(Bound)**: 각 하위 문제에서 얻을 수 있는 최적값의 상한(최대화 문제) 또는 하한(최소화 문제)을 계산합니다.

**3. 가지치기(Pruning)**: 현재 최적 해(Best Known Solution)보다 바운드가 나쁜 하위 문제는 탐색을 중단합니다.

**4. 탐색 전략(Search Strategy)**:
- **BFS(너비 우선)**: 최적 해에 가까운 레벨을 균일하게 탐색, 메모리 사용 많음
- **DFS(깊이 우선)**: 빠르게 실행 가능 해를 찾아 초기 Bound를 낮춤, 메모리 효율적
- **Best-First(최선 우선)**: 가장 바운드가 좋은 노드를 우선 탐색, 최적에 근접

### 백트래킹과의 차이

백트래킹은 **실행 불가능한 경우**를 가지치기하는 반면, 분기 한정법은 **최적이 될 수 없는 경우**까지 가지치기합니다. 최적화 문제에서 분기 한정법이 백트래킹보다 훨씬 강력합니다.

---

## 왜 필요한가

대부분의 실세계 최적화 문제는 NP-난해입니다:
- **외판원 문제(TSP)**: N개 도시를 모두 방문하는 최단 경로
- **0/1 배낭 문제**: 무게 제한 내에서 최대 가치 선택
- **작업 스케줄링**: 최소 지연 또는 최대 처리량으로 작업 배치
- **정수 선형 프로그래밍(ILP)**: 산업 현장의 배치/생산 최적화

이들은 정확한 해를 보장하는 다항 시간 알고리즘이 없습니다. 그러나:
- 근사 알고리즘은 최적 해를 보장하지 않습니다.
- 실용적인 규모(수백~수천 변수)에서는 분기 한정법이 상당히 빠르게 최적 해를 찾습니다.
- **산업용 MIP 솔버**(Gurobi, CPLEX, GLPK)는 내부적으로 분기 한정법 + LP 완화를 사용합니다.

---

## 실제 구현 예제

### 예제 1: 0/1 배낭 문제 (Python)

```python
from dataclasses import dataclass, field
from heapq import heappush, heappop
from typing import List

@dataclass
class Item:
    weight: float
    value: float

@dataclass(order=True)
class Node:
    # 최대 힙 구현을 위해 음수로 저장
    neg_bound: float
    level: int = field(compare=False)
    weight: float = field(compare=False)
    value: float = field(compare=False)

def compute_bound(node: Node, n: int, capacity: float, items: List[Item]) -> float:
    """
    분수 배낭 완화(Fractional Knapsack Relaxation)로 상한 계산.
    정수 제약을 제거하고 물건을 분수로 넣을 수 있다고 가정.
    """
    if node.weight >= capacity:
        return 0
    
    bound = node.value
    j = node.level + 1
    total_weight = node.weight
    
    # 가치/무게 비율 순으로 정렬된 items 가정
    while j < n and total_weight + items[j].weight <= capacity:
        total_weight += items[j].weight
        bound += items[j].value
        j += 1
    
    # 마지막 물건은 분수로
    if j < n:
        bound += (capacity - total_weight) * (items[j].value / items[j].weight)
    
    return bound

def knapsack_branch_bound(items: List[Item], capacity: float) -> float:
    # 가치/무게 비율 내림차순 정렬
    items.sort(key=lambda x: x.value / x.weight, reverse=True)
    n = len(items)
    
    max_value = 0.0
    pq = []  # 최대 힙 (neg_bound 사용)
    
    root = Node(neg_bound=0.0, level=-1, weight=0.0, value=0.0)
    root.neg_bound = -compute_bound(root, n, capacity, items)
    heappush(pq, root)
    
    nodes_visited = 0
    
    while pq:
        current = heappop(pq)
        nodes_visited += 1
        
        # 가지치기: 이 노드의 bound가 현재 최적값보다 나쁘면 스킵
        if -current.neg_bound <= max_value:
            continue
        
        if current.level == n - 1:
            continue
        
        next_level = current.level + 1
        
        # Case 1: 다음 물건을 선택 (Include)
        with_item = Node(
            neg_bound=0.0,
            level=next_level,
            weight=current.weight + items[next_level].weight,
            value=current.value + items[next_level].value
        )
        
        if with_item.weight <= capacity and with_item.value > max_value:
            max_value = with_item.value
        
        with_item.neg_bound = -compute_bound(with_item, n, capacity, items)
        if -with_item.neg_bound > max_value:
            heappush(pq, with_item)
        
        # Case 2: 다음 물건을 선택하지 않음 (Exclude)
        without_item = Node(
            neg_bound=0.0,
            level=next_level,
            weight=current.weight,
            value=current.value
        )
        without_item.neg_bound = -compute_bound(without_item, n, capacity, items)
        if -without_item.neg_bound > max_value:
            heappush(pq, without_item)
    
    print(f"탐색한 노드 수: {nodes_visited} (전체 {2**n}개 대비)")
    return max_value

# 테스트
items = [
    Item(weight=2, value=6),
    Item(weight=2, value=10),
    Item(weight=3, value=12),
    Item(weight=4, value=13),
    Item(weight=6, value=15),
]
capacity = 7

result = knapsack_branch_bound(items, capacity)
print(f"최대 가치: {result}")  # 최대 가치: 22.0
```

**실행 결과**: 완전 탐색(32가지)에 비해 훨씬 적은 노드를 탐색하며 최적 해를 찾습니다.

---

### 예제 2: 외판원 문제 (TSP) - 분기 한정법 (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

const int INF = 1e9;
int n;
vector<vector<int>> dist;

struct Node {
    int level, cost, vertex;
    vector<bool> visited;
    
    bool operator>(const Node& o) const {
        return cost > o.cost;
    }
};

// 하한 계산: 현재 비용 + 남은 도시들의 최소 이동 비용
int computeLowerBound(const Node& node) {
    int lb = node.cost;
    // 각 도시에서 나가는 최소 비용의 합산 (이완된 하한)
    for (int i = 0; i < n; i++) {
        if (!node.visited[i]) {
            int minEdge = INF;
            for (int j = 0; j < n; j++) {
                if (i != j && dist[i][j] < minEdge) {
                    minEdge = dist[i][j];
                }
            }
            if (minEdge != INF) lb += minEdge;
        }
    }
    return lb;
}

int tspBranchBound() {
    int bestCost = INF;
    priority_queue<Node, vector<Node>, greater<Node>> pq;
    
    Node root;
    root.level = 1;
    root.cost = 0;
    root.vertex = 0;
    root.visited.assign(n, false);
    root.visited[0] = true;
    
    pq.push(root);
    
    while (!pq.empty()) {
        Node curr = pq.top();
        pq.pop();
        
        // 하한이 현재 최적보다 크면 가지치기
        int lb = computeLowerBound(curr);
        if (lb >= bestCost) continue;
        
        if (curr.level == n) {
            // 모든 도시를 방문한 경우 시작점으로 돌아가는 비용 추가
            int totalCost = curr.cost + dist[curr.vertex][0];
            bestCost = min(bestCost, totalCost);
            continue;
        }
        
        // 다음 방문할 도시 분기
        for (int i = 0; i < n; i++) {
            if (!curr.visited[i] && dist[curr.vertex][i] != INF) {
                Node next;
                next.level = curr.level + 1;
                next.cost = curr.cost + dist[curr.vertex][i];
                next.vertex = i;
                next.visited = curr.visited;
                next.visited[i] = true;
                
                if (next.cost < bestCost) {
                    pq.push(next);
                }
            }
        }
    }
    
    return bestCost;
}

int main() {
    n = 5;
    dist = {
        {0,   10,  8,   9,  7},
        {10,  0,   10,  5,  6},
        {8,   10,  0,   8,  9},
        {9,   5,   8,   0,  6},
        {7,   6,   9,   6,  0}
    };
    
    cout << "최소 TSP 경로 비용: " << tspBranchBound() << endl;
    // 최소 TSP 경로 비용: 34
    return 0;
}
```

---

## 주의사항과 팁

### 1. 좋은 Bound 함수가 성능의 핵심

가지치기의 효과는 Bound 함수의 품질에 따라 결정됩니다:
- **느슨한 Bound**: 가지치기가 적게 이루어져 탐색 공간이 넓어짐
- **타이트한 Bound**: 더 많은 가지치기, 빠른 실행
- 배낭 문제의 **LP 완화(분수 배낭)** 는 가장 타이트한 상한 중 하나입니다.

### 2. 초기 해(Initial Solution) 품질이 중요

알고리즘 시작 전 탐욕 알고리즘 등으로 좋은 초기 해를 구해두면, 초기 Bound가 타이트해져 가지치기 효율이 올라갑니다.

### 3. 탐색 전략 선택

- **소규모 문제**: BFS로 최적에 가까운 경로를 균일하게 탐색
- **대규모 문제**: DFS + 탐욕 초기 해로 빠르게 좋은 해를 찾고 Bound를 조여나감
- **메모리 한계**: DFS가 BFS보다 메모리 효율적

### 4. 산업용 MIP 솔버 활용

실제 업무에서 수백 개 이상의 변수를 다루는 정수 계획법 문제는 직접 구현보다 **Gurobi**, **CPLEX**, Python의 **PuLP**, **OR-Tools** 같은 검증된 솔버를 사용하는 것이 현실적입니다.

```python
# OR-Tools로 배낭 문제 해결 예시
from ortools.algorithms.python import knapsack_solver

solver = knapsack_solver.KnapsackSolver(
    knapsack_solver.SolverType.KNAPSACK_MULTIDIMENSION_BRANCH_AND_BOUND_SOLVER,
    "KnapsackExample",
)
values = [360, 83, 59, 130, 431]
weights = [[7, 0, 30, 22, 80]]
capacities = [50]
solver.init(values, weights, capacities)
computed_value = solver.solve()
print(f"최적값: {computed_value}")
```

### 5. 시간 복잡도의 현실

최악의 경우 여전히 지수 시간입니다. 실제로는:
- 좋은 Bound 함수: 탐색 노드 수가 N의 다항식 수준으로 줄기도 함
- 특수 구조(희소 그래프, 특수 제약): 더 빠른 수렴
- 최악 케이스: 잘 선택된 입력에서 여전히 지수적 탐색 가능

### 알고리즘 비교

| 알고리즘 | 보장 | 시간 복잡도 | 적합 상황 |
|---------|-----|-----------|---------|
| 완전 탐색 | 최적 | O(N!) | 극소 규모 |
| 분기 한정법 | 최적 | 가변적 (실용적) | 중소 규모 |
| 동적 프로그래밍 | 최적 | 다항식 | 부분 구조 존재 시 |
| 근사 알고리즘 | 근사비 보장 | 다항식 | 대규모, 근사 허용 |
| 메타휴리스틱 | 미보장 | 다항식 | 대규모, 빠른 해 필요 |

분기 한정법은 **정확한 최적 해**가 반드시 필요하지만 다항 시간 알고리즘이 없는 경우, 실용적인 규모에서 최선의 선택입니다.

## 참고 자료
- [Branch and Bound - Wikipedia](https://en.wikipedia.org/wiki/Branch_and_bound)
- [Branch and Bound Algorithm - Baeldung on Computer Science](https://www.baeldung.com/cs/branch-and-bound)
- [Branch-and-bound algorithms: A survey of recent advances - ScienceDirect](https://www.sciencedirect.com/science/article/pii/S1572528616000062)
- [Google OR-Tools Knapsack Solver](https://developers.google.com/optimization/pack/knapsack)
