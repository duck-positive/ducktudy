---
layout: post
title: "Link-Cut Tree 완전 정복: 동적 트리의 O(log N) 연산을 가능하게 하는 고급 자료구조"
date: 2026-11-16
categories: [cs, computer-science]
tags: [link-cut-tree, dynamic-trees, splay-tree, graph-algorithms, competitive-programming]
---

## 개념 설명

컴퓨터 과학에서 트리는 가장 중요한 자료구조 중 하나입니다. 그러나 현실 문제에서는 트리가 고정되어 있지 않고 동적으로 변화하는 경우가 많습니다. 예를 들어, 그래프에서 간선이 추가되거나 제거되는 상황, 혹은 경로 위의 가중치를 효율적으로 쿼리해야 하는 상황이 있습니다.

**링크-컷 트리(Link-Cut Tree, LCT)**는 1983년 Daniel Sleator와 Robert Tarjan이 발표한 "A Data Structure for Dynamic Trees" 논문에서 소개된 자료구조로, 동적으로 변화하는 포레스트(트리들의 집합)에서 다음 연산들을 O(log n) 분할 상환(amortized) 시간에 지원합니다.

| 연산 | 설명 |
|------|------|
| `link(u, v)` | 두 트리를 u와 v를 잇는 간선으로 연결 |
| `cut(u, v)` | u와 v 사이의 간선을 제거하여 트리를 분리 |
| `find_root(u)` | u가 속한 트리의 루트를 반환 |
| `path_aggregate(u, v)` | u에서 v까지 경로 위 값들에 대한 집계 계산 |
| `make_root(u)` | u를 자신이 속한 트리의 루트로 만들기 |

LCT는 Heavy-Light Decomposition과 개념적으로 유사하지만, 트리가 동적으로 변화해도 연산이 가능하다는 점에서 훨씬 더 강력합니다. HLD는 정적 트리에서 O(log² n) 경로 쿼리를 지원하지만, 간선의 추가·제거를 지원하지 않습니다.

---

## 왜 필요한가

정적인 트리 문제는 세그먼트 트리, HLD, LCA 등으로 효율적으로 해결할 수 있습니다. 그러나 다음과 같은 **동적 트리 문제**들에서는 LCT가 필수적입니다.

**동적 연결성(Dynamic Connectivity)**: 그래프에서 간선이 추가·제거될 때, 두 노드가 연결되어 있는지를 O(log n)에 확인합니다. 나이브한 BFS/DFS 방식은 간선 변경마다 O(n + m)이 소요됩니다.

**동적 최소 신장 트리(Dynamic MST)**: 간선의 추가·제거에 따라 MST를 갱신합니다. LCT를 사용하면 각 갱신을 O(log n)에 처리할 수 있습니다.

**최대 유량 최적화**: Sleator-Tarjan의 원래 논문에서 LCT를 사용하여 최대 유량을 O(mn log n)에 계산하는 알고리즘을 제시했습니다. 이는 단순 구현의 O(mn) 대비 큰 개선입니다.

**경쟁 프로그래밍**: 트리의 구조가 쿼리 중간에 바뀌는 문제들에서 LCT는 거의 유일한 해결책입니다.

---

## 핵심 아이디어: 선호 경로 분해

LCT의 핵심은 **선호 경로 분해(Preferred Path Decomposition)**입니다.

**선호 자식(Preferred Child)**: 각 노드는 최대 하나의 "선호 자식"을 가집니다. 어떤 자식을 선호 자식으로 지정할지는 access 연산이 결정합니다.

**선호 경로(Preferred Path)**: 선호 자식을 따라 내려가는 경로의 체인입니다. 하나의 트리는 여러 개의 선호 경로로 분해됩니다.

**보조 트리(Auxiliary Tree)**: 각 선호 경로는 Splay Tree로 관리됩니다. Splay Tree의 키는 각 노드의 깊이(depth)이므로, 경로 상에서 얕은 노드가 왼쪽 서브트리, 깊은 노드가 오른쪽 서브트리에 위치합니다.

```
표현되는 트리:          LCT 내부 (Splay Trees):
       1                 [1-2-3] (하나의 Splay Tree)
      / \                        |
     2   6              [4-5]   [6]   [7]
     |
     3
    / \
   4   7
   |
   5
```

노드의 부모 포인터는 두 종류가 있습니다.
- **보조 자식 포인터**: Splay Tree 내의 왼쪽·오른쪽 자식
- **경로 부모 포인터(Path Parent Pointer)**: 각 Splay Tree의 루트 노드가 가리키는 연결점. 이 포인터가 존재하면 해당 노드가 Splay Tree의 루트임을 의미합니다.

가장 중요한 연산인 **Access(u)**는 루트부터 u까지의 경로 전체를 하나의 선호 경로로 만드는 연산입니다. Access 과정에서 u를 포함하는 Splay Tree를 루트까지 반복적으로 올라가며 경로를 연결합니다.

---

## 실제 구현 예제

### 예제 1: Link-Cut Tree 핵심 구현 (Python)

```python
import sys
sys.setrecursionlimit(200000)

class Node:
    def __init__(self, val=0):
        self.ch = [None, None]  # Splay tree 내 left/right 자식
        self.p = None           # 부모 (보조 또는 경로 부모)
        self.val = val          # 노드 고유 값
        self.agg = val          # 집계값 (경로 합)
        self.rev = False        # 경로 역전 플래그 (make_root용)

def is_root(x):
    """x가 자신의 Splay Tree의 루트인지 확인"""
    p = x.p
    # 경로 부모 포인터를 가지면 Splay Tree의 루트
    return p is None or (p.ch[0] is not x and p.ch[1] is not x)

def push_up(x):
    """집계값 갱신 (부모 → 자식 방향 반영)"""
    x.agg = x.val
    if x.ch[0]: x.agg += x.ch[0].agg
    if x.ch[1]: x.agg += x.ch[1].agg

def push_down(x):
    """역전 플래그를 자식에게 전파"""
    if x.rev:
        x.ch[0], x.ch[1] = x.ch[1], x.ch[0]
        if x.ch[0]: x.ch[0].rev ^= True
        if x.ch[1]: x.ch[1].rev ^= True
        x.rev = False

def _collect_path(x, stack):
    """루트까지의 경로를 스택에 수집 (반복적 구현)"""
    while not is_root(x):
        stack.append(x)
        x = x.p
    stack.append(x)
    return stack

def rotate(x):
    """Splay tree 단일 회전"""
    p = x.p
    g = p.p
    d = 1 if p.ch[1] is x else 0

    if not is_root(p):
        if g.ch[0] is p: g.ch[0] = x
        elif g.ch[1] is p: g.ch[1] = x

    p.ch[d] = x.ch[d ^ 1]
    if x.ch[d ^ 1]: x.ch[d ^ 1].p = p
    x.ch[d ^ 1] = p
    p.p = x
    x.p = g
    push_up(p)
    push_up(x)

def splay(x):
    """x를 자신의 Splay Tree 루트로 올리기"""
    # 먼저 루트까지 push_down 적용
    stack = []
    _collect_path(x, stack)
    while stack:
        push_down(stack.pop())

    # Splay 회전
    while not is_root(x):
        p = x.p
        if not is_root(p):
            g = p.p
            if (g.ch[0] is p) == (p.ch[0] is x):
                rotate(p)  # zig-zig
            else:
                rotate(x)  # zig-zag
        rotate(x)

def access(x):
    """루트에서 x까지를 하나의 선호 경로로 만들기"""
    last = None
    y = x
    while y:
        splay(y)
        y.ch[1] = last  # 기존 선호 자식 제거, last로 교체
        push_up(y)
        last = y
        y = y.p
    splay(x)
    return last

def make_root(x):
    """x를 자신이 속한 트리의 루트로 만들기"""
    access(x)
    x.rev ^= True
    push_down(x)

def find_root(x):
    """x가 속한 트리의 루트 반환"""
    access(x)
    while x.ch[0]:
        push_down(x)
        x = x.ch[0]
    splay(x)  # 접근 복잡도 최적화를 위해 필수
    return x

def link(x, y):
    """x와 y를 연결 (x를 y의 자식으로 만들기)"""
    make_root(x)
    if find_root(y) is not x:  # 사이클 방지
        x.p = y

def cut(x, y):
    """x와 y 사이의 간선 제거"""
    make_root(x)
    access(y)
    # access 후 y의 Splay Tree에서 x는 y의 왼쪽 자식이어야 함
    if y.ch[0] is x and x.ch[1] is None:
        y.ch[0] = None
        x.p = None
        push_up(y)

def path_sum(x, y):
    """x에서 y까지 경로의 값 합산"""
    make_root(x)
    access(y)
    return y.agg
```

### 예제 2: LCT 활용 — 동적 연결성과 경로 쿼리

```python
def demo_lct():
    n = 7
    # 노드 생성 (값 = 인덱스 + 1)
    nodes = [Node(i + 1) for i in range(n)]
    
    # 초기 트리 구성
    # 1-2-3-4-5 (체인), 1-6, 1-7
    pairs = [(0,1),(1,2),(2,3),(3,4),(0,5),(0,6)]
    for u, v in pairs:
        link(nodes[u], nodes[v])
    
    # 쿼리 1: 2에서 5까지 경로 합 (노드 2,3,4,5 = 값 2+3+4+5 = 14)
    result = path_sum(nodes[1], nodes[4])
    print(f"[쿼리] 2→5 경로 합: {result}")  # 14
    
    # 간선 3-4 제거
    cut(nodes[2], nodes[3])
    print("[연산] 간선 3-4 제거 완료")
    
    # 쿼리 2: 2와 5가 연결되어 있는가?
    make_root(nodes[1])
    root5 = find_root(nodes[4])
    connected = (root5 is nodes[1])
    print(f"[쿼리] 2와 5 연결 여부: {connected}")  # False
    
    # 간선 2-5 추가
    link(nodes[1], nodes[4])
    print("[연산] 간선 2-5 추가 완료")
    
    # 쿼리 3: 재연결 후 2→5 경로 합 (노드 2,5 = 2+5 = 7)
    result2 = path_sum(nodes[1], nodes[4])
    print(f"[쿼리] 재연결 후 2→5 경로 합: {result2}")  # 7

demo_lct()
```

---

## 시간 복잡도 분석

LCT의 각 연산은 **O(log n) 분할 상환(amortized)** 시간에 동작합니다.

이 분석은 Splay Tree의 분할 상환 분석에 기반합니다. Sleator-Tarjan은 **포텐셜 함수** Φ = Σ ⌊log(size(x))⌋를 사용하여, access 연산이 O(log n) 분할 상환임을 증명했습니다.

Access 연산에서 선호 경로의 전환이 O(log n)번 발생합니다. 이를 **선호 경로 전환 수의 경계**라고 합니다. 각 전환은 Splay 연산을 유발하지만, 분할 상환 분석에 의해 전체 비용은 O(log n)으로 수렴합니다.

| 연산 | 시간 복잡도 |
|------|------------|
| access | O(log n) amortized |
| make_root | O(log n) amortized |
| find_root | O(log n) amortized |
| link | O(log n) amortized |
| cut | O(log n) amortized |
| path_aggregate | O(log n) amortized |

---

## 주의사항과 팁

**1. is_root 판단의 중요성**: 노드가 자신의 Splay Tree의 루트인지 판단할 때, 반드시 부모의 `ch` 배열을 확인해야 합니다. 경로 부모 포인터를 보조 자식 포인터와 혼동하면 잘못된 Splay가 발생합니다.

**2. Push-down 순서**: Splay 연산 전에 반드시 경로 상의 모든 노드에 위에서 아래로(루트 → 대상) push_down을 적용해야 합니다. 역전 플래그(rev)를 올바르게 전파하지 않으면 경로 순서가 뒤섞입니다. 반복적 구현으로 스택에 경로를 먼저 수집한 뒤 역순으로 push_down하는 것이 안전합니다.

**3. Find_root 후 Splay 필수**: `find_root` 후 찾은 루트 노드를 반드시 `splay`해야 접근 복잡도를 O(log n) amortized로 유지합니다. 누락 시 최악 케이스 O(n) 연산이 발생할 수 있습니다.

**4. Cut 연산의 사전 조건 검증**: `cut(u, v)` 전에 u와 v가 실제로 직접 연결된 간선으로 이어져 있는지 확인하는 로직을 추가하세요. 잘못된 cut은 자료구조를 망가뜨립니다. 구현에서 `y.ch[0] is x and x.ch[1] is None`을 확인하는 이유입니다.

**5. 배열 기반 구현**: 경쟁 프로그래밍에서는 클래스/포인터 기반보다 전역 배열 기반 구현이 캐시 효율이 좋아 2~3배 빠릅니다. 최대 노드 수를 미리 정의하고 `ch[MAXN][2]`, `par[MAXN]`, `val[MAXN]` 형태로 구현하세요.

**6. 집계 연산 확장**: 경로 합 외에도 최솟값, 최댓값, XOR 등 다양한 집계를 `push_up`을 수정하는 것만으로 쉽게 구현할 수 있습니다. 경로 가중치 갱신(업데이트)도 lazy propagation을 추가하면 됩니다.

---

## 참고 자료

- [KTH Algorithm Competition Template Library (KACTL)](https://github.com/kth-competitive-programming/kactl) — 경쟁 프로그래밍용 LCT 참조 구현 포함
- [CP-Algorithms — Link-Cut Tree](https://github.com/cp-algorithms/cp-algorithms) — 한국어/영어 LCT 개념 설명 및 예제
