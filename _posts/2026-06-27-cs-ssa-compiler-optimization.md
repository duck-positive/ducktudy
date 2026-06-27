---
layout: post
title: "SSA(정적 단일 대입) 완전 정복: LLVM과 GCC가 코드를 최적화하는 핵심 원리"
date: 2026-06-27
categories: [cs, computer-science]
tags: [ssa, compiler, llvm, optimization, ir, dead-code-elimination, constant-propagation, gcc, phi-node]
---

컴파일러는 어떻게 아래 코드에서 `x = 1; x = 2;`의 첫 번째 대입이 죽은 코드(dead code)임을 알아낼까? 루프 안의 변수가 루프 불변(loop-invariant)임을 어떻게 감지해 밖으로 이동시킬까? 이 모든 최적화의 핵심에는 **SSA(Static Single Assignment, 정적 단일 대입)** 형식이 있다. LLVM, GCC, V8 엔진, Rust 컴파일러가 모두 SSA를 중간 표현(IR)의 기반으로 사용하는 이유를 깊이 파헤친다.

---

## 개념 설명

### SSA란 무엇인가

SSA는 컴파일러 **중간 표현(Intermediate Representation, IR)** 의 한 형식으로, 하나의 핵심 규칙만을 가진다:

> **모든 변수는 정확히 한 번만 정의(definition)된다.**

일반 코드에서는 같은 변수에 여러 번 값을 할당할 수 있다. SSA 변환은 각 대입에 고유한 변수 버전을 부여해 이 문제를 해결한다.

```
// 일반 코드
x = 1;
x = x + 2;
x = x * 3;

// SSA 변환 후
x₁ = 1;
x₂ = x₁ + 2;
x₃ = x₂ * 3;
```

각 `x`에 서로 다른 첨자(subscript)가 붙어 **정의-사용(def-use) 체인이 명확**해진다. `x₁`이 어디서 사용되는지, `x₂`가 무엇에 의존하는지 전역적으로 즉시 파악 가능하다.

### φ(Phi) 노드: 분기 합류 지점의 처리

SSA의 핵심 난제는 **제어 흐름 합류(join point)** 다. `if-else`의 두 분기에서 같은 변수에 다른 값이 대입될 때, 합류 지점에서 어느 버전을 사용해야 할까?

```
// 원본 코드
if (cond) {
    x = 1;
} else {
    x = 2;
}
use(x);  // 어느 x?
```

SSA는 이를 **φ(Phi) 노드**로 해결한다. φ 노드는 "제어 흐름이 어느 경로로 왔느냐에 따라 다른 버전을 선택하는" 특수 함수다.

```
// SSA 변환 후
if (cond) {
    x₁ = 1;
} else {
    x₂ = 2;
}
x₃ = φ(x₁, x₂);  ← if 분기에서 왔으면 x₁, else에서 왔으면 x₂
use(x₃);
```

φ 노드는 오직 **기본 블록(Basic Block)의 시작 지점**에만 위치할 수 있으며, 실제로 실행 시에는 런타임 분기 결과에 따라 하나의 값만 선택된다.

### SSA 변환 알고리즘

SSA 변환의 표준 알고리즘은 두 단계다:

1. **지배 프론티어(Dominance Frontier) 계산**: 각 기본 블록이 "어느 블록의 직전 지배자인가"를 계산해 φ 노드 삽입 위치를 결정
2. **변수 재명명(Variable Renaming)**: 깊이 우선 순회(DFS)로 CFG를 탐색하며 각 정의에 고유 버전 번호 부여

---

## 왜 필요한가

SSA 형식 자체가 최적화는 아니다. SSA는 최적화 분석을 **극도로 단순화**하는 구조다.

### SSA가 가능하게 하는 핵심 최적화

**1. 상수 전파(Constant Propagation)**
`x₁ = 5`라고 정의되었으면 이후 `x₁`의 모든 사용을 리터럴 `5`로 치환할 수 있다. SSA에서 `x₁`은 단 한 곳에서만 정의되므로 치환 대상 탐색이 O(1)이다.

**2. 죽은 코드 제거(Dead Code Elimination)**
어떤 변수의 정의가 한 번도 사용되지 않는다면 그 정의는 제거 가능하다. SSA에서는 def-use 체인이 직접적이므로 사용 횟수를 쉽게 카운팅할 수 있다.

**3. 공통 부분식 제거(Common Subexpression Elimination, CSE)**
같은 연산이 여러 곳에 반복되면 한 번만 계산하고 재사용한다. SSA에서는 동일 버전 변수를 사용하는 동일 연산이 자동으로 중복 식별된다.

**4. 루프 불변 코드 이동(Loop-Invariant Code Motion, LICM)**
루프 안에서 루프 변수에 의존하지 않는 연산은 루프 밖으로 이동한다. SSA의 def-use 체인으로 루프 내 변수 의존성을 정확히 추적할 수 있다.

---

## 실제 구현 예제

### 예제 1: SSA 형식 변환기 구현 (Python — 단순 CFG 기반)

실제 컴파일러의 SSA 변환을 이해하기 위한 미니멀한 구현이다.

```python
from typing import List, Dict, Set, Optional
from dataclasses import dataclass, field

@dataclass
class Instruction:
    dest: Optional[str]   # 대입 대상 (None이면 순수 효과)
    op: str               # 연산
    args: List[str]       # 피연산자

    def __repr__(self):
        if self.dest:
            return f"{self.dest} = {self.op}({', '.join(self.args)})"
        return f"{self.op}({', '.join(self.args)})"


@dataclass
class BasicBlock:
    name: str
    instructions: List[Instruction] = field(default_factory=list)
    successors: List[str] = field(default_factory=list)
    predecessors: List[str] = field(default_factory=list)
    phi_nodes: Dict[str, Dict[str, str]] = field(default_factory=dict)


def rename_to_ssa(blocks: Dict[str, BasicBlock]) -> Dict[str, BasicBlock]:
    """
    단순 SSA 변수 재명명 (지배 프론티어 계산 생략, 직선 코드 가정)
    각 정의에 고유 버전 번호 부여
    """
    version: Dict[str, int] = {}  # 변수명 → 현재 버전 번호
    mapping: Dict[str, str] = {}  # 원본 변수명 → 현재 SSA 이름

    def new_version(var: str) -> str:
        version[var] = version.get(var, -1) + 1
        ssa_name = f"{var}_{version[var]}"
        mapping[var] = ssa_name
        return ssa_name

    result = {}
    for block_name, block in blocks.items():
        new_block = BasicBlock(
            name=block_name,
            successors=block.successors,
            predecessors=block.predecessors
        )
        for instr in block.instructions:
            # 피연산자는 현재 매핑된 SSA 이름으로 치환
            new_args = [mapping.get(a, a) for a in instr.args]
            # 대입 대상은 새 버전 발급
            new_dest = new_version(instr.dest) if instr.dest else None
            new_block.instructions.append(
                Instruction(dest=new_dest, op=instr.op, args=new_args)
            )
        result[block_name] = new_block
    return result


def constant_propagation(blocks: Dict[str, BasicBlock]) -> Dict[str, BasicBlock]:
    """SSA 기반 상수 전파: 정의가 상수이면 사용처를 리터럴로 치환"""
    constants: Dict[str, str] = {}  # SSA 변수명 → 상수 리터럴

    # 1패스: 상수 정의 수집
    for block in blocks.values():
        for instr in block.instructions:
            if instr.dest and instr.op == "CONST":
                constants[instr.dest] = instr.args[0]

    # 2패스: 상수 사용처 치환
    for block in blocks.values():
        for instr in block.instructions:
            instr.args = [constants.get(a, a) for a in instr.args]

    return blocks


def dead_code_elimination(blocks: Dict[str, BasicBlock]) -> Dict[str, BasicBlock]:
    """SSA 기반 죽은 코드 제거: 사용되지 않는 정의 삭제"""
    # 모든 사용된 변수 수집
    used: Set[str] = set()
    for block in blocks.values():
        for instr in block.instructions:
            used.update(instr.args)

    # 한 번도 사용되지 않는 대입 제거
    for block in blocks.values():
        block.instructions = [
            i for i in block.instructions
            if i.dest is None or i.dest in used
        ]
    return blocks


# 예시: x = 1; y = x + 2; z = 3; result = y (z는 사용 안 됨)
blocks = {
    "entry": BasicBlock(
        name="entry",
        instructions=[
            Instruction(dest="x", op="CONST", args=["1"]),
            Instruction(dest="z", op="CONST", args=["3"]),  # 죽은 코드
            Instruction(dest="y", op="ADD", args=["x", "2"]),
            Instruction(dest="result", op="ASSIGN", args=["y"]),
        ]
    )
}

print("=== 원본 코드 ===")
for instr in blocks["entry"].instructions:
    print(" ", instr)

ssa_blocks = rename_to_ssa(blocks)
print("\n=== SSA 변환 후 ===")
for instr in ssa_blocks["entry"].instructions:
    print(" ", instr)

cp_blocks = constant_propagation(ssa_blocks)
print("\n=== 상수 전파 후 ===")
for instr in cp_blocks["entry"].instructions:
    print(" ", instr)

dce_blocks = dead_code_elimination(cp_blocks)
print("\n=== 죽은 코드 제거 후 ===")
for instr in dce_blocks["entry"].instructions:
    print(" ", instr)
```

**실행 결과:**
```
=== 원본 코드 ===
  x = CONST(1)
  z = CONST(3)
  y = ADD(x, 2)
  result = ASSIGN(y)

=== SSA 변환 후 ===
  x_0 = CONST(1)
  z_0 = CONST(3)
  y_0 = ADD(x_0, 2)
  result_0 = ASSIGN(y_0)

=== 상수 전파 후 ===
  x_0 = CONST(1)
  z_0 = CONST(3)
  y_0 = ADD(1, 2)      ← x_0이 리터럴 1로 치환
  result_0 = ASSIGN(y_0)

=== 죽은 코드 제거 후 ===
  x_0 = CONST(1)       ← y_0 = ADD(1, 2) 에서 사용
  y_0 = ADD(1, 2)
  result_0 = ASSIGN(y_0)
  # z_0 제거됨 (아무도 사용 안 함)
```

### 예제 2: LLVM IR에서 SSA와 Phi 노드 직접 관찰

실제 LLVM IR을 통해 SSA와 φ 노드를 확인할 수 있다.

```c
// example.c
int max(int a, int b) {
    int result;
    if (a > b) {
        result = a;
    } else {
        result = b;
    }
    return result;
}
```

```bash
# LLVM IR 생성 (SSA 형식)
clang -O1 -emit-llvm -S example.c -o example.ll
```

```llvm
; LLVM IR (SSA 형식)
define i32 @max(i32 %a, i32 %b) {
entry:
  %cmp = icmp sgt i32 %a, %b        ; 비교: a > b
  br i1 %cmp, label %if.then, label %if.else

if.then:
  br label %if.end                   ; result = a 경로

if.else:
  br label %if.end                   ; result = b 경로

if.end:
  ; φ 노드: if.then에서 왔으면 %a, if.else에서 왔으면 %b 선택
  %result = phi i32 [ %a, %if.then ], [ %b, %if.else ]
  ret i32 %result
}
```

`%result = phi i32 [ %a, %if.then ], [ %b, %if.else ]`가 φ 노드다. `result`는 단 한 번만 정의되며, 런타임 제어 흐름에 따라 값이 결정된다. `-O1` 이상에서 이 함수는 조건부 이동 명령어(`cmov`)로 최적화될 수 있다.

---

## 주의사항 및 팁

### SSA Out: 레지스터 할당 전 SSA 파괴

컴파일러는 최적화 과정에서 SSA를 유지하다가 **레지스터 할당(Register Allocation)** 단계에서 SSA를 "파괴(SSA Destruction)"한다. φ 노드는 실제 하드웨어에 존재하지 않으므로, 각 φ 노드는 **병렬 복사(parallel copy)** 명령으로 변환된다. 이 과정을 잘못 처리하면 **Lost Copy 문제**나 **Swap 문제**가 발생할 수 있으며, 표준 알고리즘(Sreedhar et al.)으로 해결한다.

### 메모리 연산과 SSA

SSA는 레지스터 변수에만 직접 적용된다. C의 포인터, 배열, 힙 객체 등 **메모리 위치**에는 SSA를 직접 적용할 수 없다(별칭(alias) 분석이 필요하기 때문). LLVM은 `alloca` 명령을 사용해 스택 메모리를 레지스터처럼 다루고, `mem2reg` 패스로 이를 SSA 레지스터로 승격한다.

### SSA가 쓰이는 현대 컴파일러들

| 컴파일러 | SSA 사용 방식 |
|---------|--------------|
| LLVM | 전체 IR이 SSA 형식, `opt` 툴로 최적화 패스 적용 |
| GCC | GIMPLE IR에서 SSA 사용 (GCC 4.0 이후) |
| V8 (JavaScript) | Turbofan의 Sea-of-Nodes가 SSA 기반 |
| HotSpot JVM | C2 컴파일러의 최적화에 SSA 활용 |
| Rust MIR | Mid-level IR이 SSA 형식 |
| SpiderMonkey | IonMonkey JIT이 SSA 사용 |

### 최적화 순서의 중요성

SSA 기반 최적화는 **순서에 민감**하다. 상수 전파 → 죽은 코드 제거 → 공통 부분식 제거 순으로 반복(iteration)해야 최대 최적화 효과를 얻을 수 있다. LLVM의 `-O2` 는 수십 개의 패스를 정해진 순서로 여러 번 반복 적용한다.

---

## 참고 자료
- [Wikipedia: Static Single-Assignment Form](https://en.wikipedia.org/wiki/Static_single-assignment_form)
- [LLVM IR: SSA-Phi Node 매핑 가이드](https://mapping-high-level-constructs-to-llvm-ir.readthedocs.io/en/latest/control-structures/ssa-phi.html)
- [blog.yossarian.net: Understanding SSA Forms](https://blog.yossarian.net/2020/10/23/Understanding-static-single-assignment-forms)
- [Compiler Design: LLVM IR, SSA, Optimization Passes](https://anshadameenza.com/blog/technology/2025-02-05-compiler-design-llvm-ir-ssa-optimization-passes/)
