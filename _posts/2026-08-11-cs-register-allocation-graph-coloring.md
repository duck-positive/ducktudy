---
layout: post
title: "레지스터 할당과 그래프 채색: 컴파일러가 변수를 CPU 레지스터에 배치하는 방법"
date: 2026-08-11
categories: [cs, computer-science]
tags: [compiler, register-allocation, graph-coloring, optimization, llvm, chaitin]
---

## 개념 설명

현대 CPU는 연산을 빠르게 처리하기 위해 극소수의 초고속 저장소인 **레지스터(Register)**를 제공합니다. x86-64 아키텍처는 범용 레지스터 16개(RAX, RBX, RCX, ..., R15), ARM64는 31개를 제공합니다. 반면 프로그램에서 사용하는 변수(지역 변수, 임시값, 루프 카운터 등)의 수는 수백, 수천 개에 달할 수 있습니다.

**레지스터 할당(Register Allocation)**은 컴파일러 백엔드에서 수행하는 핵심 최적화 과정으로, 무한한 가상 레지스터(또는 소스 코드 변수)를 유한한 물리 레지스터에 배치하는 문제입니다. 레지스터에 들어가지 못한 변수는 **스필(Spill)**되어 스택 메모리에 저장되며, 이때 load/store 명령이 추가되어 성능이 저하됩니다.

### 생존 구간 분석 (Live Range Analysis)

두 변수가 **동시에 살아있다(live at the same time)**면, 같은 레지스터를 사용할 수 없습니다. 어떤 변수가 특정 시점에 "살아있다"는 의미는 그 변수의 현재 값이 이후에 사용될 것이라는 뜻입니다.

예를 들어 아래 코드를 보겠습니다:

```
a = 1          ; a의 생존 구간 시작
b = 2          ; b의 생존 구간 시작
c = a + b      ; a, b 모두 살아있음
d = c * 2      ; c만 살아있음 (a, b는 사망)
               ; d의 생존 구간은 여기서 시작
return d
```

여기서:
- 명령 2와 3 사이에는 a, b가 동시에 살아있으므로 서로 다른 레지스터를 써야 합니다.
- c와 d는 동시에 살아있지 않으므로 같은 레지스터를 재사용할 수 있습니다.

### 간섭 그래프 (Interference Graph)

생존 구간이 겹치는 변수 쌍은 **간섭(interfere)**한다고 말합니다. 이를 그래프로 표현하면:
- **정점(Node)**: 각 변수
- **간선(Edge)**: 생존 구간이 겹치는 두 변수 사이의 연결

이제 레지스터 할당 문제는 정확히 **k-채색 문제(k-Graph Coloring)**가 됩니다:
- 색의 수 k = 물리 레지스터의 수
- "인접한 정점은 다른 색" = "동시에 살아있는 변수는 다른 레지스터"

k-채색은 일반적으로 NP-완전 문제이지만, 인터프리터 그래프는 특수한 성질(chordal에 가까운 구조)을 가져 실용적인 근사 알고리즘이 잘 동작합니다.

---

## 왜 필요한가

레지스터 할당은 컴파일러 최적화에서 가장 중요한 단계 중 하나입니다.

1. **레지스터 vs 메모리 접근 속도 차이**: 레지스터 접근은 약 1 사이클, L1 캐시는 4 사이클, L2는 12 사이클, DRAM은 200+ 사이클입니다. 스필이 많아지면 메모리 접근이 급증하여 프로그램이 수배~수십 배 느려집니다.

2. **루프 집약적 코드에서의 영향**: 루프 변수를 레지스터에 유지하면 반복마다 load/store를 생략할 수 있습니다. 반복 횟수가 1억 번이라면 스필 여부가 수십억 명령의 차이를 만듭니다.

3. **벡터화/SIMD와의 시너지**: SIMD 레지스터(AVX-512 기준 512비트 × 32개)를 효율적으로 활용하려면 정교한 레지스터 할당이 필수입니다.

4. **디버그 vs 릴리즈 빌드의 성능 차이**: gcc/clang의 `-O0`은 모든 변수를 스택에 저장하고, `-O2`/`-O3`는 공격적인 레지스터 할당을 수행합니다. 이것이 디버그 빌드가 릴리즈 빌드보다 3~10배 느린 주요 원인입니다.

---

## 실제 구현 예제

### 예제 1: 간섭 그래프 구성 및 그래프 채색 (Python)

아래 코드는 단순화된 레지스터 할당기를 구현합니다. Chaitin-Briggs 알고리즘의 핵심 로직을 담고 있습니다.

```python
from collections import defaultdict

class RegisterAllocator:
    """
    Chaitin-Briggs 스타일의 그래프 채색 레지스터 할당기.
    
    알고리즘 단계:
    1. Build   — 간섭 그래프 구성
    2. Simplify — 차수 < k인 정점을 스택에 push하며 제거
    3. Spill    — 제거할 정점이 없으면 스필 후보 선택
    4. Select   — 스택에서 꺼내며 색(레지스터) 할당
    5. Start Over — 스필된 코드를 다시 컴파일 (여기서는 생략)
    """

    def __init__(self, num_registers: int):
        self.k = num_registers          # 물리 레지스터 수
        self.interference = defaultdict(set)
        self.variables = set()

    def add_variable(self, var: str):
        self.variables.add(var)
        if var not in self.interference:
            self.interference[var] = set()

    def add_interference(self, u: str, v: str):
        """두 변수 사이에 간섭 간선 추가 (동시 생존)"""
        if u == v:
            return
        self.interference[u].add(v)
        self.interference[v].add(u)
        self.variables.add(u)
        self.variables.add(v)

    def allocate(self) -> tuple[dict[str, int | None], list[str]]:
        """
        색 할당을 수행합니다.
        Returns:
            (allocation, spilled):
              allocation: var -> register_id (None이면 스필됨)
              spilled: 스필된 변수 목록
        """
        # 작업용 복사본
        degree = {v: len(self.interference[v]) for v in self.variables}
        adj_copy = {v: set(self.interference[v]) for v in self.variables}

        stack = []       # Simplify 단계에서 쌓인 정점들
        spilled = []     # 스필 결정된 변수들
        removed = set()

        remaining = set(self.variables)

        # Simplify + Spill 단계
        changed = True
        while remaining and changed:
            changed = False
            # 차수 < k인 정점 우선 제거
            low_degree = [v for v in remaining if degree[v] < self.k]
            for v in low_degree:
                stack.append(v)
                removed.add(v)
                remaining.discard(v)
                # 이웃의 degree 감소
                for u in adj_copy[v]:
                    if u not in removed:
                        degree[u] -= 1
                changed = True

            # 모두 차수 >= k이면 스필 후보 선택 (가장 차수 높은 것을 스필)
            if remaining and not changed:
                # 실제 컴파일러는 여기서 비용 모델을 사용 (사용 빈도, 루프 깊이 등)
                victim = max(remaining, key=lambda v: degree[v])
                spilled.append(victim)
                removed.add(victim)
                remaining.discard(victim)
                for u in adj_copy[victim]:
                    if u not in removed:
                        degree[u] -= 1
                changed = True

        # Select 단계: 스택에서 꺼내며 레지스터 배정
        allocation: dict[str, int | None] = {v: None for v in spilled}

        while stack:
            v = stack.pop()
            # 이웃에게 사용된 레지스터 집합 파악
            used = {allocation[u] for u in self.interference[v]
                    if u in allocation and allocation[u] is not None}
            # 가장 작은 번호의 미사용 레지스터 할당
            reg = None
            for r in range(self.k):
                if r not in used:
                    reg = r
                    break
            if reg is None:
                # 할당 불가 — 추가 스필 (이 단순 구현에서는 여기까지 오지 않음)
                spilled.append(v)
            else:
                allocation[v] = reg

        return allocation, spilled


# ===================== 테스트 =====================
# 간단한 코드 블록의 생존 구간:
# a: [1, 3], b: [2, 4], c: [3, 5], d: [4, 6]
# 간섭: (a,b), (a,c), (b,c), (b,d), (c,d)

allocator = RegisterAllocator(num_registers=2)  # 레지스터 2개만 있다고 가정

# 변수 등록
for var in ['a', 'b', 'c', 'd']:
    allocator.add_variable(var)

# 간섭 간선 추가 (동시에 생존하는 변수 쌍)
allocator.add_interference('a', 'b')  # a, b 동시 생존
allocator.add_interference('a', 'c')  # a, c 동시 생존
allocator.add_interference('b', 'c')  # b, c 동시 생존
allocator.add_interference('b', 'd')  # b, d 동시 생존
allocator.add_interference('c', 'd')  # c, d 동시 생존

allocation, spilled = allocator.allocate()

reg_names = ['R0', 'R1', 'R2', 'R3']
print("레지스터 할당 결과:")
for var, reg in sorted(allocation.items()):
    if reg is not None:
        print(f"  {var} -> {reg_names[reg]}")
    else:
        print(f"  {var} -> [스필: 스택 메모리]")
print(f"스필된 변수: {spilled}")
# 예시 출력:
#   a -> R0
#   b -> R1
#   c -> R0  (a, b 다음에 c가 배치되어 a의 레지스터 재사용)
#   d -> [스필: 스택 메모리] 또는 R0/R1 중 하나
```

### 예제 2: 생존 구간 분석 (간단한 기본 블록)

실제 컴파일러에서는 SSA(Static Single Assignment) 폼을 기반으로 생존 구간을 계산합니다.

```python
from dataclasses import dataclass

@dataclass
class Instruction:
    idx: int
    dest: str | None      # 결과 저장 변수
    srcs: list[str]       # 소스 변수들
    op: str               # 연산 이름

def compute_liveness(instructions: list[Instruction]) -> dict[str, tuple[int, int]]:
    """
    기본 블록(single basic block)에서 각 변수의 생존 구간 [first_def, last_use]을 계산합니다.
    
    Returns:
        var -> (definition_point, last_use_point)
    """
    first_def: dict[str, int] = {}
    last_use: dict[str, int] = {}

    for instr in instructions:
        # 소스 변수의 마지막 사용 시점 갱신
        for src in instr.srcs:
            last_use[src] = instr.idx

        # 목적지 변수의 최초 정의 시점 기록 (최초만)
        if instr.dest and instr.dest not in first_def:
            first_def[instr.dest] = instr.idx

    live_ranges: dict[str, tuple[int, int]] = {}
    all_vars = set(first_def.keys()) | set(last_use.keys())

    for var in all_vars:
        start = first_def.get(var, 0)
        end = last_use.get(var, start)
        live_ranges[var] = (start, end)

    return live_ranges


def build_interference_from_ranges(
    live_ranges: dict[str, tuple[int, int]]
) -> list[tuple[str, str]]:
    """
    생존 구간이 겹치는 변수 쌍 → 간섭 간선 목록 반환
    """
    vars_list = list(live_ranges.keys())
    interference_edges = []

    for i in range(len(vars_list)):
        for j in range(i + 1, len(vars_list)):
            u, v = vars_list[i], vars_list[j]
            us, ue = live_ranges[u]
            vs, ve = live_ranges[v]
            # 구간이 겹치면 간섭
            if us <= ve and vs <= ue:
                interference_edges.append((u, v))

    return interference_edges


# ===================== 테스트 =====================
# 아래 의사 코드를 명령어로 표현:
# 0: a = 5
# 1: b = 3
# 2: c = a + b    (a, b를 사용 → 이후 a, b는 필요 없음)
# 3: d = c * 2    (c를 사용)
# 4: e = d + 1    (d를 사용)
# 5: return e

instrs = [
    Instruction(0, 'a', [], 'const'),
    Instruction(1, 'b', [], 'const'),
    Instruction(2, 'c', ['a', 'b'], 'add'),
    Instruction(3, 'd', ['c'], 'mul'),
    Instruction(4, 'e', ['d'], 'add'),
    Instruction(5, None, ['e'], 'return'),
]

ranges = compute_liveness(instrs)
print("\n생존 구간:")
for var, (s, e) in sorted(ranges.items()):
    timeline = ['.' if s <= i <= e else ' ' for i in range(6)]
    print(f"  {var}: [{s}, {e}]  |{''.join(timeline)}|")

edges = build_interference_from_ranges(ranges)
print(f"\n간섭 간선: {edges}")

# 이 결과를 RegisterAllocator에 입력하면 최적 레지스터 배치를 얻을 수 있음
```

---

## LLVM의 레지스터 할당 파이프라인

실제 LLVM/Clang은 세 가지 레지스터 할당기를 지원합니다:

| 할당기 | 사용 시점 | 특징 |
|---|---|---|
| **Greedy** (기본값, `-O2` 이상) | 릴리즈 빌드 | 비용 모델 기반, 루프 감도 고려, 최고 품질 |
| **Linear Scan** | JIT 컴파일 | 빠른 컴파일 속도, 품질은 Greedy보다 낮음 |
| **Basic** | 디버깅/테스트 | 가장 단순, 그래프 채색 기반 |

LLVM Greedy 할당기의 핵심 혁신은 전역 간섭 그래프를 **구간 트리(Interval Tree)**로 표현하고, 스필 비용을 루프 깊이와 사용 빈도를 반영한 **휴리스틱 함수**로 계산한다는 점입니다.

---

## 주의사항 및 팁

### 1. 물리 레지스터 제약 (Pre-Coloring)
함수 호출 규약(Calling Convention)에 따라 특정 레지스터는 미리 예약됩니다. 예를 들어 x86-64 SystemV ABI에서 함수의 1~6번째 인자는 RDI, RSI, RDX, RCX, R8, R9에 들어갑니다. 이 "사전 채색(pre-colored)" 노드는 할당기가 반드시 존중해야 합니다.

### 2. Coalescing으로 복사 제거
`a = b`처럼 단순 복사 명령은 a와 b가 같은 레지스터에 배치되면 명령을 제거할 수 있습니다. Chaitin-Briggs의 **Conservative Coalescing**은 채색 가능성을 손상시키지 않는 범위에서 복사를 제거합니다.

### 3. Callee-saved vs Caller-saved 레지스터
x86-64에서 RBX, R12~R15는 피호출자 저장 레지스터(callee-saved)로, 함수 내에서 사용하면 반드시 복원해야 합니다. 할당기는 이를 스필 비용에 반영해 가급적 caller-saved 레지스터를 먼저 사용합니다.

### 4. SSA 폼과의 관계
LLVM은 중간 표현(IR)을 SSA(Static Single Assignment) 폼으로 유지하다가, 레지스터 할당 직전 `SSADestructionPass`로 phi 노드를 병렬 복사로 변환합니다. SSA 폼에서는 각 값이 정확히 한 번만 정의되므로 생존 구간 분석이 훨씬 단순해집니다.

### 5. -fno-omit-frame-pointer 옵션의 비용
프레임 포인터(RBP/EBP)를 유지하면 스택 추적(stack unwinding)이 쉬워지지만, 범용 레지스터 하나를 영구히 사용하는 셈입니다. 레지스터가 빡빡한 x86-32(8개)에서는 성능에 크게 영향을 줍니다.

---

## 참고 자료

- [Register allocation — Wikipedia](https://en.wikipedia.org/wiki/Register_allocation)
- [Chaitin, G.J. (1982). "Register Allocation via Coloring" — Bell Labs Original Paper](https://dl.acm.org/doi/10.1145/872726.806984)
- [LLVM Greedy Register Allocator — LLVM Documentation](https://llvm.org/docs/CodeGenerator.html#register-allocator)
- [Linear Scan Register Allocation — Wimmer & Franz, 2010](https://dl.acm.org/doi/10.1145/1772954.1772979)
