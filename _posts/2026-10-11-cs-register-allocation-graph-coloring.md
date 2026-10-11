---
layout: post
title: "컴파일러 레지스터 할당과 그래프 채색 완전 정복: Chaitin 알고리즘부터 스필·합체까지"
date: 2026-10-11
categories: [cs, computer-science]
tags: [compiler, register-allocation, graph-coloring, chaitin, liveness-analysis, spilling, coalescing]
---

컴파일러가 소스 코드를 기계어로 변환할 때, 가장 성능에 민감한 단계 중 하나가 **레지스터 할당(Register Allocation)**입니다. CPU 레지스터는 메모리보다 수십~수백 배 빠르지만, 그 수는 극히 제한적입니다(x86-64 기준 16개의 범용 레지스터). 컴파일러는 가능한 한 많은 변수를 레지스터에 유지하려 하며, 이 문제는 수학적으로 **그래프 채색(Graph Coloring)** 문제와 동치임이 증명되어 있습니다.

## 왜 레지스터 할당이 중요한가

현대 CPU에서 레지스터 접근은 1 사이클(~0.3ns)인 반면, L1 캐시는 4~5 사이클, 메인 메모리는 200~300 사이클이 소요됩니다. 변수를 레지스터에 유지하지 못하고 메모리에 쓰고 읽는 연산(스필, spill)이 발생하면 루프 한 번의 비용이 수백 배로 늘어날 수 있습니다.

특히 다음 상황에서 레지스터 할당의 중요성이 두드러집니다:

- **루프 내 반복 연산**: 루프 카운터, 누적 합산 변수 등은 반드시 레지스터에 상주해야 합니다.
- **인라인 함수**: 인라인 후 지역 변수가 급증할 때 효율적인 할당이 필요합니다.
- **벡터화(SIMD)**: 여러 SIMD 레지스터를 최적 배분해야 병렬 연산 효과를 누릴 수 있습니다.

## 핵심 개념: 라이브 레인지와 간섭

### 라이브 레인지 (Live Range)

변수가 정의(def)된 시점부터 마지막으로 사용(use)된 시점까지의 구간을 **라이브 레인지**라고 합니다. 두 변수의 라이브 레인지가 겹치면(live range overlap), 두 변수는 같은 레지스터를 사용할 수 없습니다. 이 관계를 **간섭(interference)**이라 합니다.

### 간섭 그래프 (Interference Graph)

- **노드**: 각 변수(또는 가상 레지스터)
- **엣지**: 두 변수가 간섭할 때(동시에 살아있을 때) 엣지로 연결
- **채색 문제**: 인접한 노드가 서로 다른 색을 갖도록 채색할 때, 필요한 최소 색의 수가 레지스터 수입니다.

만약 k개의 레지스터로 그래프를 k-채색할 수 없다면, 일부 변수를 메모리에 넘기는 **스필(Spill)** 처리가 필요합니다.

## 라이브니스 분석 구현

다음 코드는 기본 블록(Basic Block) 단위로 역방향 데이터 플로우 분석(Backward Dataflow Analysis)을 수행하여 각 지점에서 살아있는 변수를 계산합니다.

```python
from dataclasses import dataclass, field
from typing import Set, List

@dataclass
class BasicBlock:
    instructions: list
    successors: list = field(default_factory=list)
    defs: Set[str] = field(default_factory=set)
    uses: Set[str] = field(default_factory=set)
    live_in: Set[str] = field(default_factory=set)
    live_out: Set[str] = field(default_factory=set)

def compute_def_use(block: BasicBlock):
    """각 블록에서 def/use 집합 계산 (순서 중요: use before def)"""
    defs, uses = set(), set()
    for instr in block.instructions:
        # 사용되는 변수 중 아직 이 블록에서 정의되지 않은 것은 use
        for operand in instr.get('uses', []):
            if operand not in defs:
                uses.add(operand)
        # 정의되는 변수
        if dest := instr.get('dest'):
            defs.add(dest)
    block.defs = defs
    block.uses = uses

def compute_liveness(blocks: List[BasicBlock]):
    """역방향 반복 데이터 플로우로 전역 라이브니스 계산"""
    for block in blocks:
        compute_def_use(block)
    
    # 수렴할 때까지 반복
    changed = True
    while changed:
        changed = False
        for block in reversed(blocks):
            # live_out = 후계 블록들의 live_in의 합집합
            new_live_out = set()
            for succ in block.successors:
                new_live_out |= succ.live_in
            # live_in = uses ∪ (live_out − defs)
            new_live_in = block.uses | (new_live_out - block.defs)
            if new_live_in != block.live_in or new_live_out != block.live_out:
                block.live_in = new_live_in
                block.live_out = new_live_out
                changed = True
    return blocks

# 사용 예
b1 = BasicBlock(instructions=[
    {'dest': 'x', 'uses': ['a', 'b']},  # x = a + b
    {'dest': 'y', 'uses': ['x', 'c']},  # y = x * c
])
b2 = BasicBlock(instructions=[
    {'dest': 'z', 'uses': ['y']},        # z = y + 1
])
b1.successors = [b2]
compute_liveness([b1, b2])
print(f"b1 live_out: {b1.live_out}")  # {'y'}
print(f"b2 live_in:  {b2.live_in}")   # {'y'}
```

## Chaitin의 그래프 채색 알고리즘 구현

1982년 G.J. Chaitin이 IBM의 PL.8 컴파일러를 위해 개발한 이 알고리즘은 현대 모든 최적화 컴파일러의 기반이 되었습니다.

```python
from collections import defaultdict
from typing import Dict, Set, List, Tuple, Optional

class InterferenceGraph:
    def __init__(self):
        self.adj: Dict[str, Set[str]] = defaultdict(set)
        self.spill_costs: Dict[str, float] = {}
    
    def add_node(self, var: str, spill_cost: float = 1.0):
        self.adj.setdefault(var, set())
        self.spill_costs[var] = spill_cost
    
    def add_edge(self, u: str, v: str):
        if u != v and u in self.adj and v in self.adj:
            self.adj[u].add(v)
            self.adj[v].add(u)
    
    def degree(self, v: str) -> int:
        return len(self.adj[v])
    
    def chaitin_color(self, k: int) -> Tuple[Dict[str, int], List[str]]:
        """
        Chaitin's simplify/spill/select algorithm
        k: 사용 가능한 레지스터 수
        반환: (채색 딕셔너리, 스필된 변수 목록)
        """
        # 현재 그래프 복사 (원본 보존)
        graph = {v: set(neighbors) for v, neighbors in self.adj.items()}
        stack: List[str] = []
        spilled: List[str] = []
        
        while graph:
            # Phase 1: Simplify — 차수 < k인 노드를 스택에 push
            low_degree = [v for v in graph if len(graph[v]) < k]
            
            if low_degree:
                v = low_degree[0]
                stack.append(v)
                # 이웃 노드에서 v를 제거
                for neighbor in list(graph[v]):
                    graph[neighbor].discard(v)
                del graph[v]
            else:
                # Phase 2: Spill — 스필 비용 대비 차수 비율이 가장 낮은 노드 선택
                # (스필 비용이 낮고 간섭이 많은 노드를 먼저 스필)
                v = min(
                    graph,
                    key=lambda x: self.spill_costs.get(x, 1.0) / max(len(graph[x]), 1)
                )
                spilled.append(v)
                for neighbor in list(graph[v]):
                    graph[neighbor].discard(v)
                del graph[v]
        
        # Phase 3: Select — 스택에서 pop하며 레지스터 번호 배정
        coloring: Dict[str, int] = {}
        while stack:
            v = stack.pop()
            # 인접 노드에서 이미 사용된 색 집합
            used_colors = {coloring[n] for n in self.adj[v] if n in coloring}
            # 사용 가능한 가장 낮은 번호의 레지스터 배정
            for color in range(k):
                if color not in used_colors:
                    coloring[v] = color
                    break
            else:
                # 채색 실패 → 이 노드도 스필 (낙관적 채색 실패)
                spilled.append(v)
        
        return coloring, spilled

# 실제 사용 예
ig = InterferenceGraph()
for var in ['a', 'b', 'c', 'd', 'e']:
    ig.add_node(var, spill_cost=1.0)
ig.add_node('loop_counter', spill_cost=100.0)  # 루프 변수는 높은 스필 비용

# 간섭 관계 추가
ig.add_edge('a', 'b')
ig.add_edge('a', 'c')
ig.add_edge('b', 'c')
ig.add_edge('b', 'd')
ig.add_edge('c', 'e')

coloring, spilled = ig.chaitin_color(k=3)  # 3개의 레지스터
print(f"레지스터 배정: {coloring}")
print(f"스필된 변수: {spilled}")
```

## 합체(Coalescing)와 Linear Scan

### 합체 (Register Coalescing)

`MOVE x, y` 명령어처럼 한 레지스터의 값을 다른 레지스터로 복사하는 경우, x와 y가 간섭하지 않는다면 같은 레지스터에 배정하여 MOVE 명령어 자체를 제거할 수 있습니다. 이를 **합체(Coalescing)**라 합니다.

- **George 조건**: v를 w와 합체할 때, v의 모든 이웃 t가 `t와 w가 이미 간섭`하거나 `t의 차수가 k 미만`이어야 합니다.
- **Briggs 조건**: 합체 후 새 노드의 차수 ≥ k인 이웃의 수가 k 미만이어야 합니다.

### Linear Scan (선형 스캔 할당)

JIT 컴파일러(V8, JVM HotSpot, LLVM ORC)에서는 그래프 채색보다 빠른 **선형 스캔 알고리즘**을 사용합니다:

```python
def linear_scan_allocate(live_ranges: list, k: int) -> dict:
    """
    live_ranges: [(start, end, var_name), ...] (start로 정렬됨)
    k: 레지스터 수
    """
    import heapq
    
    # 활성 라이브 레인지들 (end 기준 최소 힙)
    active = []  # (end, var_name, register)
    available_regs = list(range(k))
    allocation = {}
    spilled = []
    
    for start, end, var in sorted(live_ranges):
        # 종료된 라이브 레인지의 레지스터 반환
        while active and active[0][0] < start:
            expired_end, expired_var, reg = heapq.heappop(active)
            available_regs.append(reg)
        
        if available_regs:
            reg = available_regs.pop()
            allocation[var] = reg
            heapq.heappush(active, (end, var, reg))
        else:
            # 레지스터 부족 → 가장 늦게 끝나는 것과 비교하여 스필 결정
            if active and active[-1][0] > end:
                # 현재 변수보다 늦게 끝나는 것을 스필
                spill_end, spill_var, spill_reg = active[-1]
                heapq.heappop(active)
                spilled.append(spill_var)
                allocation[var] = spill_reg
                heapq.heappush(active, (end, var, spill_reg))
            else:
                spilled.append(var)
    
    return allocation, spilled
```

## 주의사항 및 실전 팁

### 스필 코드 삽입

스필된 변수는 메모리(스택 프레임)에 저장됩니다. 컴파일러는 변수를 사용하기 전에 `LOAD` 명령을, 정의 후에는 `STORE` 명령을 삽입합니다. 스필이 많아지면 스택 프레임이 커지고 캐시 효율이 떨어집니다.

### Pre-coloring (사전 채색)

함수 호출 규약(Calling Convention)에서는 특정 레지스터가 특정 용도로 예약됩니다(x86-64: `rdi`, `rsi`, `rdx`는 함수 인자; `rax`는 반환값). 이런 물리 레지스터는 그래프에 추가되어 사전에 채색됩니다.

### SSA 기반 할당

현대 컴파일러(GCC, LLVM, JVM)는 SSA(Static Single Assignment) 형태에서 레지스터 할당을 수행합니다. SSA에서는 각 변수가 정확히 한 번만 정의되므로, 라이브 레인지 계산이 단순해지고 스필·합체 최적화가 더 정확해집니다.

## 참고 자료

- [Wikipedia: Register allocation](https://en.wikipedia.org/wiki/Register_allocation)
- [Wikipedia: Graph coloring](https://en.wikipedia.org/wiki/Graph_coloring)
- [LLVM Documentation: Register Allocator](https://llvm.org/docs/CodeGenerator.html#register-allocation)
- [Engineering a Compiler (Cooper & Torczon) - Chapter 13](https://www.cs.rice.edu/~keith/EAC2.e/)
