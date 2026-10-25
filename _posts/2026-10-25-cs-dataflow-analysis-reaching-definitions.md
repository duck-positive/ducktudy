---
layout: post
title: "데이터플로우 분석 심화: Reaching Definitions, Live Variables, 컴파일러 최적화의 기초"
date: 2026-10-25
categories: [cs, computer-science]
tags: [compiler, dataflow-analysis, reaching-definitions, live-variables, optimization, cfg, ssa]
---

## 개요

현대 컴파일러가 코드를 최적화하는 핵심 기반 기술 중 하나가 **데이터플로우 분석(Dataflow Analysis)**이다. LLVM, GCC, JVM의 JIT 컴파일러가 모두 이 기법에 의존한다. 데이터플로우 분석은 프로그램을 실제로 실행하지 않고도 변수의 정의와 사용 관계, 생존 범위, 상수 전파 가능성 등을 정적으로 계산한다.

이 글에서는 데이터플로우 분석의 핵심 개념인 **제어 흐름 그래프(CFG)**, **GEN/KILL 집합**, **Reaching Definitions**, **Live Variables**, **Available Expressions**를 깊이 있게 다루고, 이를 실제 최적화에 연결하는 방법을 Python 코드로 구현한다.

---

## 제어 흐름 그래프 (Control Flow Graph, CFG)

데이터플로우 분석의 시작점은 CFG다. 프로그램을 **기본 블록(Basic Block)**으로 분할하고, 기본 블록 간의 제어 흐름을 방향 그래프로 표현한다.

**기본 블록**이란 다음 조건을 만족하는 최대 연속 명령어 시퀀스다:
- 첫 번째 명령어만 분기 목적지(레이블)일 수 있음
- 마지막 명령어만 분기/점프 명령어일 수 있음

```python
# CFG 기본 구조 구현
from dataclasses import dataclass, field
from typing import Set, Dict, List, Optional

@dataclass
class Instruction:
    op: str           # 연산 종류: 'assign', 'add', 'branch', 'return'
    dst: Optional[str] = None   # 목적지 변수
    src1: Optional[str] = None
    src2: Optional[str] = None
    value: Optional[int] = None  # 상수값
    
    def __repr__(self):
        if self.op == 'assign':
            return f"{self.dst} = {self.value if self.value is not None else self.src1}"
        elif self.op in ('add', 'sub', 'mul'):
            return f"{self.dst} = {self.src1} {self.op} {self.src2}"
        return f"{self.op} {self.dst or ''}"


@dataclass
class BasicBlock:
    id: str
    instructions: List[Instruction] = field(default_factory=list)
    successors: List[str] = field(default_factory=list)
    predecessors: List[str] = field(default_factory=list)
    
    # 데이터플로우 분석 결과
    gen: Set[str] = field(default_factory=set)
    kill: Set[str] = field(default_factory=set)
    in_set: Set[str] = field(default_factory=set)
    out_set: Set[str] = field(default_factory=set)


@dataclass
class CFG:
    blocks: Dict[str, BasicBlock] = field(default_factory=dict)
    entry: str = 'entry'
    exit: str = 'exit'
    
    def add_edge(self, src: str, dst: str):
        self.blocks[src].successors.append(dst)
        self.blocks[dst].predecessors.append(src)
```

---

## 데이터플로우 방정식의 일반 구조

모든 데이터플로우 분석은 다음 일반 방정식을 따른다:

**Forward Analysis (전방 분석):**
```
IN[B]  = ⊔ OUT[pred(B)]   (모든 전임자의 OUT 합)
OUT[B] = GEN[B] ∪ (IN[B] - KILL[B])
```

**Backward Analysis (후방 분석):**
```
OUT[B] = ⊔ IN[succ(B)]    (모든 후임자의 IN 합)
IN[B]  = GEN[B] ∪ (OUT[B] - KILL[B])
```

여기서 ⊔는 meet operator로, 분석 목적에 따라 ∪(합집합) 또는 ∩(교집합)이 된다.

---

## Reaching Definitions (도달 정의)

### 개념

정의(definition) `d: x = ...`가 프로그램 지점 p에 **도달(reach)**한다는 것은, d에서 p로 가는 경로상에서 x를 재정의하는 명령어 없이 도달할 수 있다는 의미다.

**사용처:** Dead Code Elimination, Constant Propagation, Use-Def Chain 구축

**분석 방향:** 전방(Forward), Meet: ∪(Union)

```
GEN[B]  = 블록 B에서 생성되는 정의 집합 (가장 마지막 정의)
KILL[B] = 블록 B에서 죽이는 정의 집합 (같은 변수의 다른 정의들)
IN[B]   = B 진입 시 도달하는 정의 집합
OUT[B]  = B 탈출 시 도달하는 정의 집합
```

### 구현

```python
def compute_gen_kill_reaching(cfg: CFG, all_definitions: Dict[str, Set[str]]):
    """
    Reaching Definitions 분석을 위한 GEN/KILL 계산
    
    all_definitions: {변수명: {해당 변수의 모든 정의 ID 집합}}
    각 정의는 고유 ID (예: "d1", "d2" ...)
    """
    for block_id, block in cfg.blocks.items():
        gen = set()
        kill = set()
        
        # 블록 내 명령어를 순서대로 처리
        for instr in block.instructions:
            if instr.dst:  # 정의 명령어
                def_id = f"{block_id}:{instr.dst}"
                
                # 이 정의가 같은 변수의 이전 정의를 KILL
                if instr.dst in all_definitions:
                    kill |= (all_definitions[instr.dst] - {def_id})
                
                # GEN에 이 정의 추가 (이전에 같은 변수를 GEN했으면 제거)
                gen = {d for d in gen if not d.endswith(f":{instr.dst}")}
                gen.add(def_id)
        
        block.gen = gen
        block.kill = kill


def reaching_definitions(cfg: CFG) -> Dict[str, Set[str]]:
    """
    반복 알고리즘으로 Reaching Definitions 계산
    수렴(Fixpoint)까지 반복
    """
    # 초기화: IN[entry] = {}, OUT[b] = GEN[b]
    for block in cfg.blocks.values():
        block.in_set = set()
        block.out_set = set(block.gen)
    
    changed = True
    iteration = 0
    
    while changed:
        changed = False
        iteration += 1
        
        # 전방 분석: entry에서 exit 방향으로 순회
        for block_id in topological_order(cfg):
            block = cfg.blocks[block_id]
            
            # IN[B] = ∪ OUT[pred(B)]
            new_in = set()
            for pred_id in block.predecessors:
                new_in |= cfg.blocks[pred_id].out_set
            
            # OUT[B] = GEN[B] ∪ (IN[B] - KILL[B])
            new_out = block.gen | (new_in - block.kill)
            
            if new_in != block.in_set or new_out != block.out_set:
                block.in_set = new_in
                block.out_set = new_out
                changed = True
    
    print(f"수렴까지 {iteration}번 반복")
    return {bid: cfg.blocks[bid].in_set for bid in cfg.blocks}


def topological_order(cfg: CFG) -> List[str]:
    """CFG의 위상 정렬 (BFS 기반, 루프 있으면 근사)"""
    visited = []
    queue = [cfg.entry]
    seen = set()
    
    while queue:
        node = queue.pop(0)
        if node in seen:
            continue
        seen.add(node)
        visited.append(node)
        for succ in cfg.blocks[node].successors:
            queue.append(succ)
    
    return visited
```

---

## Live Variables (생존 변수)

### 개념

변수 x가 프로그램 지점 p에서 **생존(live)**한다는 것은, p에서 시작하는 어떤 실행 경로를 따라 x를 재정의하지 않고 x를 읽는 명령어에 도달할 수 있다는 의미다.

**사용처:** Dead Code Elimination (죽은 대입 제거), 레지스터 할당

**분석 방향:** 후방(Backward), Meet: ∪(Union)

```
GEN[B] (USE[B]) = 블록 B에서 정의 전에 사용되는 변수 집합
KILL[B] (DEF[B]) = 블록 B에서 정의되는 변수 집합
IN[B]  = B 진입 시 생존하는 변수 집합
OUT[B] = B 탈출 시 생존하는 변수 집합
```

### 구현

```python
def compute_gen_kill_liveness(cfg: CFG):
    """Live Variables 분석을 위한 USE/DEF(GEN/KILL) 계산"""
    for block in cfg.blocks.values():
        use = set()  # 정의 전에 사용된 변수
        defs = set()  # 이 블록에서 정의된 변수
        
        for instr in block.instructions:
            # 사용되는 변수 (아직 정의 전이면 USE에 추가)
            for src in [instr.src1, instr.src2]:
                if src and src not in defs:
                    use.add(src)
            
            # 정의되는 변수
            if instr.dst:
                defs.add(instr.dst)
        
        block.gen = use    # GEN = USE
        block.kill = defs  # KILL = DEF


def live_variables(cfg: CFG) -> Dict[str, Set[str]]:
    """반복 알고리즘으로 Live Variables 계산 (후방 분석)"""
    # 초기화
    for block in cfg.blocks.values():
        block.in_set = set(block.gen)
        block.out_set = set()
    
    changed = True
    iteration = 0
    
    while changed:
        changed = False
        iteration += 1
        
        # 후방 분석: exit에서 entry 방향으로 순회
        for block_id in reversed(topological_order(cfg)):
            block = cfg.blocks[block_id]
            
            # OUT[B] = ∪ IN[succ(B)]
            new_out = set()
            for succ_id in block.successors:
                new_out |= cfg.blocks[succ_id].in_set
            
            # IN[B] = USE[B] ∪ (OUT[B] - DEF[B])
            new_in = block.gen | (new_out - block.kill)
            
            if new_in != block.in_set or new_out != block.out_set:
                block.in_set = new_in
                block.out_set = new_out
                changed = True
    
    print(f"수렴까지 {iteration}번 반복")
    return {bid: cfg.blocks[bid].in_set for bid in cfg.blocks}


def dead_code_elimination(cfg: CFG, live_in: Dict[str, Set[str]]):
    """
    Live Variables 분석 결과로 죽은 대입 제거
    OUT에 생존하지 않는 변수에 대한 대입 = 죽은 코드
    """
    eliminated = []
    
    for block_id, block in cfg.blocks.items():
        live_after = set(cfg.blocks[block_id].out_set)
        
        # 역순으로 처리 (뒤에서부터 live 집합 추적)
        new_instrs = []
        for instr in reversed(block.instructions):
            if instr.dst and instr.dst not in live_after:
                # 이 정의의 결과가 사용되지 않음 = 죽은 코드
                eliminated.append(f"제거: {instr} in {block_id}")
            else:
                new_instrs.insert(0, instr)
                # live 집합 역방향 갱신
                live_after.discard(instr.dst)
                for src in [instr.src1, instr.src2]:
                    if src:
                        live_after.add(src)
        
        block.instructions = new_instrs
    
    for e in eliminated:
        print(e)
    return eliminated
```

---

## Available Expressions (가용 표현식)

**개념:** 표현식 `x op y`가 프로그램 지점 p에서 **가용(available)**하다는 것은, p까지의 모든 경로에서 x op y가 계산되었고, 이후 x나 y가 재정의되지 않았다는 의미다.

**사용처:** Common Subexpression Elimination (공통 부분식 제거)

**분석 방향:** 전방(Forward), Meet: ∩(Intersection) — 모든 경로에서 가용해야 하므로

```
GEN[B]  = 블록 B에서 계산되며 그 안에서 피연산자가 재정의되지 않는 표현식
KILL[B] = 변수 x가 재정의될 때, x를 포함하는 모든 표현식
```

```python
def available_expressions(cfg: CFG, all_exprs: Set[str]) -> Dict[str, Set[str]]:
    """
    Available Expressions 분석 (전방, 교집합 meet)
    """
    # 초기화: IN[entry] = {}, OUT[b] = all_exprs (TOP)
    for bid, block in cfg.blocks.items():
        if bid == cfg.entry:
            block.out_set = set(block.gen)
        else:
            block.out_set = set(all_exprs)  # Universal set로 초기화
    
    changed = True
    while changed:
        changed = False
        
        for block_id in topological_order(cfg):
            if block_id == cfg.entry:
                continue
            block = cfg.blocks[block_id]
            
            # IN[B] = ∩ OUT[pred(B)]  (교집합!)
            preds = block.predecessors
            if preds:
                new_in = set(cfg.blocks[preds[0]].out_set)
                for pred_id in preds[1:]:
                    new_in &= cfg.blocks[pred_id].out_set
            else:
                new_in = set()
            
            # OUT[B] = GEN[B] ∪ (IN[B] - KILL[B])
            new_out = block.gen | (new_in - block.kill)
            
            if new_in != block.in_set or new_out != block.out_set:
                block.in_set = new_in
                block.out_set = new_out
                changed = True
    
    return {bid: cfg.blocks[bid].in_set for bid in cfg.blocks}
```

---

## 세 가지 분석 비교

| 분석 | 방향 | Meet | GEN | KILL | 용도 |
|-----|------|-----|-----|------|-----|
| Reaching Definitions | 전방 | ∪ | 생성된 정의 | 같은 변수의 다른 정의 | 상수 전파, Dead Code |
| Live Variables | 후방 | ∪ | 사용 전 변수 | 정의된 변수 | 레지스터 할당, Dead Code |
| Available Expressions | 전방 | ∩ | 계산된 표현식 | 피연산자가 바뀐 표현식 | CSE (공통 부분식 제거) |

---

## 실전 최적화: 상수 전파 (Constant Propagation)

Reaching Definitions를 사용해 사용 지점에 도달하는 정의가 단 하나이고 그 정의가 상수 대입이면, 사용을 상수로 교체할 수 있다.

```python
def constant_propagation(cfg: CFG, reaching: Dict[str, Set[str]],
                          def_map: Dict[str, Instruction]):
    """
    Reaching Definitions 기반 상수 전파
    def_map: {정의ID → 정의 Instruction}
    """
    changes = 0
    
    for block_id, block in cfg.blocks.items():
        current_reaching = set(cfg.blocks[block_id].in_set)
        
        for instr in block.instructions:
            # 사용되는 변수마다 상수 전파 시도
            for attr in ['src1', 'src2']:
                var = getattr(instr, attr)
                if var is None:
                    continue
                
                # 이 변수에 도달하는 정의 중 var를 정의하는 것들
                var_defs = [d for d in current_reaching
                           if d.endswith(f":{var}")]
                
                if len(var_defs) == 1:
                    def_instr = def_map.get(var_defs[0])
                    if def_instr and def_instr.op == 'assign' and def_instr.value is not None:
                        # 상수로 교체
                        print(f"상수 전파: {instr}에서 {var} → {def_instr.value}")
                        setattr(instr, attr, None)
                        if attr == 'src1':
                            instr.value = def_instr.value
                        changes += 1
            
            # 이 명령어의 정의로 reaching 업데이트
            if instr.dst:
                def_id = f"{block_id}:{instr.dst}"
                # 같은 변수의 이전 정의 제거
                current_reaching = {d for d in current_reaching
                                   if not d.endswith(f":{instr.dst}")}
                current_reaching.add(def_id)
    
    print(f"총 {changes}개 상수 전파 적용")
    return changes
```

---

## SSA Form과의 연결

**Static Single Assignment (SSA)**는 각 변수가 정확히 한 번만 정의되도록 프로그램을 변환한다. SSA에서는 Reaching Definitions가 자명하다—각 사용은 정확히 하나의 정의에서 온다.

분기 합류 지점에서는 **Φ 함수(Phi function)**를 삽입해 여러 정의를 합친다:

```
SSA 변환 전:          SSA 변환 후:
if (cond):            if (cond):
    x = 1                 x1 = 1
else:                 else:
    x = 2                 x2 = 2
y = x + 1            x3 = Φ(x1, x2)   ← Phi function
                     y = x3 + 1
```

SSA의 Dominance Frontier를 계산해 Φ 함수 삽입 위치를 결정하는 알고리즘은 Cytron et al.(1991)의 고전적 결과다.

---

## 주의사항 및 팁

### 수렴 보장

반복 알고리즘이 항상 수렴하려면 다음이 필요하다:
1. Lattice의 높이가 유한할 것 (변수/정의의 수는 유한)
2. 전달 함수(Transfer Function)가 단조(Monotone)일 것: A ⊆ B → f(A) ⊆ f(B)
3. Meet operator가 교환적, 결합적, 멱등적일 것

이 세 조건이 만족되면 최대 `O(n² × d)` 반복으로 수렴한다 (n: 기본 블록 수, d: 정의/변수 수).

### 정밀도와 안전성 (Precision vs Safety)

- **May 분석**: "가능한 모든" 경우를 포함 (Reaching Definitions, Live Variables → ∪ meet)
- **Must 분석**: "반드시" 참인 경우만 포함 (Available Expressions → ∩ meet)

최적화는 안전해야 한다: May 분석의 초집합이면 안전하고, Must 분석의 부분집합이면 안전하다.

### 루프와 Widening

루프가 있는 CFG에서 수렴이 매우 느릴 수 있다. **Widening(확대)** 기법은 부정확하지만 빠른 수렴을 위해 상계를 넓게 잡는다. 이후 Narrowing으로 정밀도를 회복한다 (Abstract Interpretation의 핵심 개념).

---

## 참고 자료
- [Engineering a Compiler (Cooper & Torczon) - Chapter 9: Data-Flow Analysis](https://www.elsevier.com/books/engineering-a-compiler/cooper/978-0-12-088478-0)
- [Data Flow Analysis - Wikipedia](https://en.wikipedia.org/wiki/Data-flow_analysis)
- [LLVM Language Reference Manual - SSA Form](https://llvm.org/docs/LangRef.html)
- [Cytron et al., 1991: Efficiently Computing SSA Form](https://dl.acm.org/doi/10.1145/115372.115320)
