---
layout: post
title: "심볼릭 실행(Symbolic Execution) 완전 정복: 프로그램의 모든 경로를 수학적으로 탐색하는 분석 기법"
date: 2026-09-02
categories: [cs, computer-science]
tags: [symbolic-execution, program-analysis, smt-solver, z3, klee, angr, concolic-testing, software-testing]
---

## 개념 설명

심볼릭 실행(Symbolic Execution)은 프로그램을 구체적인 입력값 대신 **기호(symbol)** 값으로 실행하는 정적/동적 분석 기법입니다. 1976년 James C. King이 제안한 이 기법은, 입력을 `x`, `y` 같은 미지수로 대체하여 프로그램이 거쳐가는 모든 실행 경로(path)를 수학적 제약식으로 표현합니다.

예를 들어 `if (x > 0)` 분기를 만나면, 참 경로에서는 `x > 0`이라는 **경로 조건(path condition)** 을 누적하고, 거짓 경로에서는 `x <= 0`을 누적합니다. 각 경로의 끝에서 SMT(Satisfiability Modulo Theories) 솔버에 이 조건을 넘겨 해당 경로를 실행시키는 구체적 입력을 자동으로 구합니다.

### 핵심 구성 요소

| 구성 요소 | 역할 |
|-----------|------|
| 심볼릭 스토어 | 변수 → 심볼릭 수식 매핑 |
| 경로 조건 | 현재까지 만족해야 할 제약식의 논리곱 |
| SMT 솔버 | 경로 조건의 만족 가능성(satisfiability) 판정 |
| 실행 트리 | 분기마다 뻗어나가는 상태 공간 |

## 왜 필요한가

### 일반 테스팅의 한계

단위 테스트는 개발자가 상상하는 입력만 검증합니다. 100개의 테스트가 있어도 실제 버그를 유발하는 엣지 케이스를 놓치기 쉽습니다.

```python
def process(x: int, y: int) -> int:
    result = x * 2
    if result == y:
        if y > 1000000:
            raise RuntimeError("특정 조건에서만 발생하는 버그!")
    return result
```

위 함수에서 버그를 유발하려면 `y = x * 2` 이고 `y > 1000000`인 입력이 필요합니다. 무작위 테스트(fuzzing)로 이를 찾을 확률은 극히 낮습니다. 심볼릭 실행은 이를 자동으로 찾아냅니다.

### 심볼릭 실행의 장점

1. **경로 완전성** — 도달 가능한 모든 실행 경로를 탐색합니다.
2. **자동 테스트 생성** — 각 경로를 커버하는 테스트 입력을 자동으로 만듭니다.
3. **버그·취약점 자동 발견** — 메모리 오류, 나눗셈 오류, 어설션 위반 등을 탐지합니다.
4. **증명 능력** — 특정 속성이 모든 경로에서 유지됨을 증명할 수 있습니다.

### 주요 활용 분야

- **취약점 탐색** — 버퍼 오버플로, Use-After-Free 등 메모리 버그 자동 탐지
- **바이너리 분석** — 역공학, 악성코드 분석 (angr 프레임워크)
- **테스트 자동화** — KLEE는 GNU Coreutils에서 수백 개의 버그를 자동 발견
- **프로토콜 검증** — 네트워크 프로토콜 상태 기계의 모든 상태 탐색

## 실제 구현 예제

### 예제 1: Python으로 간단한 심볼릭 실행기 구현

```python
from z3 import Int, Solver, sat, And

def symbolic_execution_demo():
    """
    def target(x, y):
        if x + y > 10:
            if x - y > 0:
                return "Path A: x+y>10 and x>y"
            else:
                return "Path B: x+y>10 and x<=y"
        else:
            return "Path C: x+y<=10"
    
    위 함수의 각 경로를 커버하는 입력을 자동 생성
    """
    x = Int('x')
    y = Int('y')

    paths = [
        # (경로 이름, 경로 조건)
        ("Path A", And(x + y > 10, x - y > 0)),
        ("Path B", And(x + y > 10, x - y <= 0)),
        ("Path C", x + y <= 10),
    ]

    for name, condition in paths:
        solver = Solver()
        solver.add(condition)
        if solver.check() == sat:
            model = solver.model()
            x_val = model[x].as_long()
            y_val = model[y].as_long()
            print(f"{name}: x={x_val}, y={y_val} → 조건 충족")
        else:
            print(f"{name}: 도달 불가능한 경로 (UNSAT)")

symbolic_execution_demo()
# Path A: x=11, y=0   → 조건 충족
# Path B: x=0, y=11   → 조건 충족
# Path C: x=0, y=0    → 조건 충족
```

### 예제 2: Z3로 취약점 자동 탐지

```python
from z3 import BitVec, Solver, sat, And, URem, ULT

def find_divide_by_zero():
    """
    C 코드:
      uint32_t dangerous(uint32_t a, uint32_t b) {
          uint32_t c = a % b;   // b == 0이면 Division-by-zero
          return c;
      }
    
    b == 0이 되는 입력을 자동으로 찾는다.
    """
    a = BitVec('a', 32)
    b = BitVec('b', 32)

    # 취약점 조건: b가 0인 경우
    vuln_condition = b == 0

    solver = Solver()
    solver.add(vuln_condition)

    if solver.check() == sat:
        model = solver.model()
        print(f"[취약점 발견] a={model[a]}, b={model[b]}")
        print(f"  → b=0일 때 a % b는 Division-by-zero 발생!")
    else:
        print("취약점 없음")


def find_integer_overflow():
    """
    uint8_t add(uint8_t x, uint8_t y) {
        return x + y;  // 결과가 255를 넘으면 오버플로
    }
    
    오버플로를 일으키는 입력을 찾는다.
    """
    x = BitVec('x', 8)   # 8비트 부호없는 정수
    y = BitVec('y', 8)

    # 두 값의 합이 8비트로 표현 불가능한 경우 (오버플로)
    # ZExtend로 16비트로 확장 후 비교
    from z3 import ZeroExt
    overflow_cond = ZeroExt(8, x) + ZeroExt(8, y) > 255

    solver = Solver()
    solver.add(overflow_cond)

    if solver.check() == sat:
        model = solver.model()
        xv = model[x].as_long()
        yv = model[y].as_long()
        print(f"[정수 오버플로 발견] x={xv}, y={yv}, 합={xv+yv} > 255")

find_divide_by_zero()
find_integer_overflow()
```

### 예제 3: KLEE를 이용한 C 프로그램 자동 테스트

KLEE는 LLVM IR 수준에서 심볼릭 실행을 수행합니다.

```c
/* test_target.c */
#include <klee/klee.h>
#include <assert.h>

int double_check(int x) {
    if (x > 100) {
        if (x * 2 < 0) {
            /* 정수 오버플로 버그! 이 경로를 탐지해야 함 */
            return -1;
        }
    }
    return x * 2;
}

int main() {
    int x;
    /* x를 심볼릭 변수로 선언 */
    klee_make_symbolic(&x, sizeof(x), "x");

    int result = double_check(x);
    /* KLEE는 result == -1인 경로의 입력을 자동 생성 */
    klee_assert(result >= 0);

    return 0;
}

/* 컴파일 및 실행:
 * clang -emit-llvm -c -g test_target.c -o test_target.bc
 * klee test_target.bc
 *
 * KLEE 출력:
 * KLEE: ERROR: test_target.c:10: assertion failed
 * 생성된 테스트 케이스: x = 1073741825 (0x40000001)
 */
```

### 예제 4: angr로 바이너리 분석

```python
import angr
import claripy

def find_secret_path(binary_path: str):
    """
    바이너리 내의 특정 "성공" 주소에 도달하는 입력을 찾는다.
    CTF 문제나 크랙미(crackme) 분석에 활용.
    """
    project = angr.Project(binary_path, auto_load_libs=False)

    # 16바이트 심볼릭 입력 생성
    flag = claripy.BVS('flag', 8 * 16)

    # 진입점에서 시작하는 초기 상태
    initial_state = project.factory.entry_state(
        args=[binary_path],
        stdin=angr.SimFileStream(name='stdin', content=flag, size=16)
    )

    # 도달해야 할 주소 (성공 분기)와 피해야 할 주소 (실패 분기)
    find_addr = 0x4006b0    # "Correct!" 출력 주소
    avoid_addr = 0x4006c0   # "Wrong!" 출력 주소

    simgr = project.factory.simulation_manager(initial_state)
    simgr.explore(find=find_addr, avoid=avoid_addr)

    if simgr.found:
        solution_state = simgr.found[0]
        # 조건을 만족하는 구체 입력값 추출
        solution = solution_state.solver.eval(flag, cast_to=bytes)
        print(f"[성공] 입력값: {solution}")
        return solution
    else:
        print("해당 경로에 도달할 수 없습니다.")
        return None
```

## 심볼릭 실행의 핵심 도전 과제

### 1. 경로 폭발 문제 (Path Explosion)

프로그램의 분기가 많아질수록 탐색해야 할 경로의 수가 지수적으로 증가합니다. `n`개의 분기가 있으면 최대 `2^n`개의 경로가 존재합니다.

**해결 전략:**
- **탐색 전략 최적화** — DFS, BFS, 커버리지 기반(Coverage-guided) 탐색
- **경로 병합(State Merging)** — 유사한 경로를 하나로 합쳐 상태 공간 압축
- **요약(Summary)** — 함수의 입출력 관계만 요약해 함수 내부 경로 생략

### 2. 솔버 병목 문제

복잡한 경로 조건은 SMT 솔버가 풀기 매우 어렵습니다. 특히 부동소수점, 배열, 포인터 연산이 포함된 경우 솔버 실행 시간이 수 초에서 수 분까지 걸립니다.

**해결 전략:**
- **제약식 독립성(Constraint Independence)** — 연관 없는 변수를 분리해 독립적으로 풀기
- **캐싱** — 동일 제약식 재계산 방지
- **부분 구체화(Partial Concretization)** — 일부 심볼릭 값을 구체값으로 대체

### 3. 환경 모델링 문제

시스템 콜, 라이브러리 함수, 네트워크 I/O 같은 외부 환경을 심볼릭으로 모델링하기 어렵습니다.

**해결 전략:**
- **환경 스텁(Stub)** — 외부 함수의 심볼릭 반환값 정의 (KLEE의 POSIX emulation)
- **콘콜릭 실행(Concolic Execution)** — 구체 실행과 심볼릭 실행을 병행

## 콘콜릭 실행 (Concolic Testing)

Concrete + Symbolic의 합성어로, **구체 실행을 먼저 수행**하고 그 결과를 기반으로 심볼릭 분석을 병행합니다. DART, SAGE, Driller가 대표적 구현체입니다.

```
1. 임의 입력으로 구체 실행 → 실행 경로 기록
2. 기록된 경로의 제약식에서 마지막 분기 조건을 반전
3. SMT 솔버로 새 입력 생성 → 새 경로 탐색
4. 1-3 반복
```

이 방식은 경로 폭발 문제를 줄이면서도 높은 코드 커버리지를 달성할 수 있습니다.

## 실무 적용 팁

1. **엔트리 포인트 선택** — 전체 바이너리보다 특정 함수/모듈만 분석하면 효율이 높습니다.
2. **제약 단순화** — 문자열보다 정수 타입의 입력이 솔버가 훨씬 빠르게 처리합니다.
3. **타임아웃 설정** — 현실적 시간 제한(예: 경로당 30초)을 두고 Best-effort로 탐색합니다.
4. **퍼징과 결합** — AFL 등 커버리지 기반 퍼저로 얕은 버그를 먼저 찾고, 심볼릭 실행으로 깊은 경로를 탐색합니다(하이브리드 퍼징).
5. **KLEE vs angr** — KLEE는 소스 코드가 있는 C/C++에 적합, angr는 바이너리 역공학에 강력합니다.

## 참고 자료

- [KLEE Symbolic Execution Engine (GitHub)](https://github.com/klee/klee)
- [angr Binary Analysis Framework (GitHub)](https://github.com/angr/angr)
- [Z3 Theorem Prover — Microsoft Research (GitHub)](https://github.com/Z3Prover/z3)
- [Z3 Haskell Bindings (Hackage)](https://hackage.haskell.org/package/z3)
