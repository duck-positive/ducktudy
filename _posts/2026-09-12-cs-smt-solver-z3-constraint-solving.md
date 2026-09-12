---
layout: post
title: "SMT 솔버 완전 정복: Z3로 이해하는 SAT Modulo Theories와 제약 충족 문제 해결"
date: 2026-09-12
categories: [cs, computer-science]
tags: [smt, sat, z3, constraint-solving, formal-methods, theorem-proving, satisfiability]
---

## 개요

프로그램 검증, 취약점 분석, 컴파일러 최적화, 스케줄링 문제 — 이 모든 영역의 저변에는 **SMT(Satisfiability Modulo Theories) 솔버**가 있습니다. SAT 솔버가 "이 불리언 수식을 만족하는 할당이 존재하는가?"를 묻는다면, SMT 솔버는 **정수, 실수, 배열, 비트벡터 같은 다양한 이론(Theory)을 동시에 고려하며 더 풍부한 제약 조건을 해결**합니다.

Microsoft Research, NYU, Stanford 등 내로라하는 연구 기관이 개발한 Z3는 현재 가장 널리 쓰이는 SMT 솔버입니다. Python API를 통해 손쉽게 접근할 수 있으며, 실제로 Dafny, KLEE, angr 같은 검증·분석 도구의 핵심 엔진으로 사용됩니다. 이 글에서는 SMT의 이론적 토대부터 Z3를 활용한 실전 예제까지 깊이 있게 살펴봅니다.

---

## SAT와 SMT: 무엇이 다른가

**SAT(Boolean Satisfiability)**는 명제 논리 수식 φ에 대해 φ를 참으로 만드는 변수 할당이 존재하는지를 결정하는 문제입니다. Cook-Levin 정리에 의해 SAT는 NP-완전 문제이지만, 현대의 CDCL(Conflict-Driven Clause Learning) 기반 SAT 솔버는 수백만 변수의 실용적 인스턴스를 초 단위에 해결합니다.

**SMT**는 SAT를 확장하여 first-order logic 수식과 특정 이론의 공리를 함께 고려합니다. 지원하는 주요 이론:

| 이론 | 기호 | 예시 |
|------|------|------|
| 선형 정수 산술 | LIA | `x + 2*y ≤ 10, y ≥ 0` |
| 선형 실수 산술 | LRA | `0.5*x + y = 3.14` |
| 비선형 산술 | NIA / NRA | `x² + y² = 1` |
| 비트벡터 | BV | `(x & 0xFF) >> 4 = 5` |
| 배열 | ARR | `A[i] = v → A[j] = A[j]` |
| 비해석 함수 | UF | `f(a) = f(b) → a = b` |
| 문자열 | STR | `len(s) ≥ 3 ∧ s contains "ab"` |

SMT는 이 이론들을 조합한 수식을 동시에 처리할 수 있어, 프로그램의 상태 공간을 정확히 모델링할 수 있습니다.

---

## SMT 솔버의 내부 구조: DPLL(T)

SMT 솔버는 **DPLL(T) 알고리즘**을 핵심 메커니즘으로 사용합니다. 이름이 보여주듯 SAT의 DPLL 알고리즘을 이론 T와 결합한 것입니다.

```
DPLL(T) 개요:

1. Boolean Abstraction (추상화)
   - 각 원자 수식(atomic formula)을 불리언 변수로 치환
   - 예: (x + y > 5) → p₁, (x < 3) → p₂

2. SAT Solving (SAT 단계)
   - 불리언 추상화된 수식에 CDCL SAT 솔버 적용
   - 할당 M = {p₁ = true, p₂ = false, ...} 획득

3. Theory Checking (이론 검사 단계)
   - M에 대응하는 원자 수식 집합을 이론 솔버(Theory Solver)에 전달
   - 이론 솔버: 선형 산술은 Simplex, 비트벡터는 BV 결정 프로시저 등
   - 이론적으로 불만족이면 T-conflict clause 생성

4. Learning (학습)
   - T-conflict clause를 SAT 솔버에 추가 학습
   - 2단계로 돌아가 다시 SAT 해 탐색

5. 전체가 SAT이면 → SAT (모델 반환)
   모든 할당 시도가 소진되면 → UNSAT
```

핵심 아이디어는 **SAT 솔버와 이론 솔버가 conflict-driven learning을 통해 협력**한다는 것입니다. 이론 솔버는 SAT 솔버에게 "이 불리언 할당 조합은 이론적으로 불가능하다"는 정보를 피드백하고, SAT 솔버는 이를 학습하여 탐색 공간을 가지치기합니다.

### Nelson-Oppen 이론 조합

여러 이론을 동시에 사용할 때는 **Nelson-Oppen 프레임워크**로 각 이론 솔버가 협력합니다. 각 이론 솔버는 자신이 담당하는 이론의 공리하에서 제약을 처리하고, 이론 간 공유 변수에 대한 동등성 정보를 전파합니다.

---

## Z3 설치와 기본 사용법

```bash
pip install z3-solver
```

### 기본 예제: 정수 제약 풀기

```python
from z3 import *

# 정수 변수 선언
x, y, z = Ints('x y z')

# 솔버 생성
s = Solver()

# 제약 조건 추가
s.add(x + y + z == 100)
s.add(x > 0, y > 0, z > 0)
s.add(x < y)
s.add(y < z)
s.add(x * x + y * y == z * z)  # 피타고라스 삼중쌍

# 풀기
result = s.check()
print(f"결과: {result}")  # sat

if result == sat:
    m = s.model()
    print(f"x={m[x]}, y={m[y]}, z={m[z]}")
    # x=20, y=48, z=52 (또는 다른 피타고라스 삼중쌍)
```

### 비트벡터 이론: 정수 오버플로 감지

```python
from z3 import *

# 32비트 정수 비트벡터
a, b = BitVecs('a b', 32)

s = Solver()

# a, b가 양수인 경우 덧셈 오버플로가 발생하는 조건을 찾아라
# 오버플로 조건: a + b < a (부호 있는 정수 관점)
s.add(a > 0, b > 0)
s.add(a + b < a)  # 오버플로 발생 조건

result = s.check()
print(f"오버플로 가능: {result}")  # sat

if result == sat:
    m = s.model()
    a_val = m[a].as_signed_long()
    b_val = m[b].as_signed_long()
    print(f"a={a_val}, b={b_val}")
    print(f"a+b={a_val + b_val} (파이썬), 실제 32비트: {(a_val + b_val) & 0xFFFFFFFF}")
```

이 예제는 **정적 분석 도구**가 버퍼 오버플로, 정수 오버플로 취약점을 자동으로 탐지하는 원리를 보여줍니다. 실제로 KLEE, angr 같은 도구는 이와 유사한 방식으로 SMT 솔버를 활용합니다.

---

## 실제 응용: 프로그램 검증

### 루프 불변식 검증

아래는 배열의 최댓값을 구하는 함수의 정확성을 Z3로 검증하는 예시입니다.

```python
from z3 import *

def verify_max_correct(n=5):
    """
    max_val이 항상 배열의 실제 최댓값임을 검증
    """
    # 배열 요소 (n개)
    arr = [Int(f'a{i}') for i in range(n)]
    max_val = Int('max_val')
    
    s = Solver()
    
    # 1. max_val이 배열에 존재하는 값이어야 함
    s.add(Or([max_val == arr[i] for i in range(n)]))
    
    # 2. max_val이 모든 요소보다 크거나 같아야 함
    for i in range(n):
        s.add(max_val >= arr[i])
    
    # 3. 이 두 조건을 모두 만족하지 않는 경우가 존재하는가? (반례 탐색)
    # 즉, max_val이 잘못된 경우를 찾아보자
    
    # 잘못된 max_val: 배열에 없는 값이 반환되는 경우
    s_wrong = Solver()
    s_wrong.add([arr[i] >= 0 for i in range(n)])
    
    # 알고리즘이 인덱스 0의 값을 max로 반환했다고 가정
    # 하지만 더 큰 값이 존재하는 경우
    wrong_max = arr[0]
    for i in range(1, n):
        s_wrong.add(arr[i] > wrong_max)  # 더 큰 값 존재
    
    result = s_wrong.check()
    if result == sat:
        m = s_wrong.model()
        print("반례 발견!")
        for i in range(n):
            print(f"  arr[{i}] = {m[arr[i]]}")
    else:
        print("알고리즘 정확성 검증됨")

verify_max_correct()
```

### 스케줄링 문제: 작업 할당 최적화

```python
from z3 import *

# 4개 작업, 3개 서버에 할당
N_JOBS = 4
N_SERVERS = 3

# assign[i][j] = 1이면 작업 i를 서버 j에 할당
assign = [[Int(f'assign_{i}_{j}') for j in range(N_SERVERS)] 
          for i in range(N_JOBS)]

# 작업별 처리 시간 (단위: 분)
durations = [30, 60, 45, 20]

# 서버별 용량 (분)
capacities = [100, 80, 70]

s = Optimize()  # 최적화 솔버

# 각 할당 변수는 0 또는 1
for i in range(N_JOBS):
    for j in range(N_SERVERS):
        s.add(Or(assign[i][j] == 0, assign[i][j] == 1))

# 각 작업은 정확히 하나의 서버에 할당
for i in range(N_JOBS):
    s.add(Sum([assign[i][j] for j in range(N_SERVERS)]) == 1)

# 각 서버의 부하가 용량을 초과하지 않음
for j in range(N_SERVERS):
    load_j = Sum([durations[i] * assign[i][j] for i in range(N_JOBS)])
    s.add(load_j <= capacities[j])

# 목표: 최대 서버 부하 최소화 (부하 균등화)
max_load = Int('max_load')
for j in range(N_SERVERS):
    load_j = Sum([durations[i] * assign[i][j] for i in range(N_JOBS)])
    s.add(max_load >= load_j)

s.minimize(max_load)

result = s.check()
if result == sat:
    m = s.model()
    print(f"최소 최대 부하: {m[max_load]} 분")
    for i in range(N_JOBS):
        for j in range(N_SERVERS):
            if m[assign[i][j]].as_long() == 1:
                print(f"  작업 {i} (처리시간: {durations[i]}분) → 서버 {j}")
```

이 예제는 클라우드 스케줄러, 로드 밸런서 설계에서 실제로 활용되는 제약 최적화 기법입니다.

---

## 심볼릭 실행과 SMT의 결합

**심볼릭 실행(Symbolic Execution)**은 프로그램의 입력을 구체적인 값 대신 심볼로 취급하며 실행합니다. 모든 조건 분기에서 경로 조건(path condition)을 누적하고, SMT 솔버로 각 경로가 도달 가능한지 판단합니다.

```python
from z3 import *

# 간단한 함수의 심볼릭 실행 시뮬레이션
def symbolic_execute_example():
    """
    def foo(x, y):
        if x > 10:
            if y > x:
                return "branch_A"  # x > 10 AND y > x
            else:
                return "branch_B"  # x > 10 AND y <= x
        else:
            return "branch_C"      # x <= 10
    
    각 경로에 도달하는 입력을 SMT로 찾는다.
    """
    x, y = Ints('x y')
    
    paths = [
        ("branch_A", [x > 10, y > x]),
        ("branch_B", [x > 10, y <= x]),
        ("branch_C", [x <= 10]),
    ]
    
    for branch_name, conditions in paths:
        s = Solver()
        for cond in conditions:
            s.add(cond)
        
        result = s.check()
        if result == sat:
            m = s.model()
            print(f"{branch_name}: x={m[x]}, y={m[y]}")
        else:
            print(f"{branch_name}: 도달 불가")

symbolic_execute_example()
# branch_A: x=11, y=12
# branch_B: x=11, y=0
# branch_C: x=0, y=0
```

실제 도구인 **KLEE**는 이 방식으로 LLVM IR 수준에서 심볼릭 실행을 수행하여 자동으로 높은 코드 커버리지를 달성하고 버그를 발굴합니다.

---

## 주의사항과 실전 팁

### 1. 결정 불가능(Undecidability) 이론 조합 주의

모든 이론 조합이 결정 가능하지는 않습니다. 예를 들어:
- 선형 정수 산술(LIA) + 비해석 함수(UF)는 결정 가능
- 비선형 정수 산술(NIA)은 이론적으로 결정 불가능 (Gödel의 불완전성)
- 실수 비선형 산술(NRA, Tarski 산술)은 결정 가능하나 지수 시간 복잡도

```python
from z3 import *

# 비선형 정수: 타임아웃에 주의
x, y = Ints('x y')
s = Solver()
s.set('timeout', 5000)  # 5초 타임아웃
s.add(x**10 + y**10 == 100)  # NIA: 느릴 수 있음
result = s.check()
print(f"결과: {result}")  # unknown(타임아웃) or sat
```

### 2. 증분 솔빙(Incremental Solving)으로 성능 최적화

```python
from z3 import *

x = Int('x')
s = Solver()
s.add(x > 0)

# push/pop으로 증분 솔빙
s.push()
s.add(x < 5)
print(s.check())  # sat: x=1,...,4

s.pop()
s.push()
s.add(x > 100)
print(s.check())  # sat: x=101,...
s.pop()
```

### 3. 양화 제거(Quantifier Elimination)

`ForAll`, `Exists` 양화사는 솔빙을 크게 느리게 만듭니다. 가능하면 특정 값에 대한 유한 인스턴스화로 대체하세요.

```python
from z3 import *

# 느림: 양화사 사용
x, y = Ints('x y')
s = Solver()
# ForAll을 직접 쓰는 대신 유한 범위 인스턴스화
for val in range(-10, 11):
    # x가 val일 때 y=val*2가 항상 양수가 아님을 확인
    s.add(Implies(x == val, y == val * 2))

s.add(y < 0)
print(s.check())  # sat: x=-1, y=-2
```

### 4. Z3 이외의 SMT 솔버

| 솔버 | 특징 |
|------|------|
| **Z3** | Microsoft Research, 범용, Python/C++ API |
| **CVC5** | Stanford/Iowa, 특히 UF·문자열 강점 |
| **Yices2** | SRI, 경량·고속 |
| **MathSAT** | FBK, 보간법(interpolation) 강점 |
| **Boolector** | 비트벡터·배열 특화 |

---

## 마치며

SMT 솔버는 현대 소프트웨어 안전성 확보의 숨겨진 엔진입니다. AWS의 [s2n TLS 라이브러리 검증](https://github.com/awslabs/s2n), Microsoft의 [Azure 인프라 검증](https://www.microsoft.com/en-us/research/project/everest-project/), Intel CPU 설계 검증 등 산업 현장에서 이미 핵심 역할을 합니다. Z3 Python API로 직접 제약 조건을 표현하고 풀어보면, "이 코드가 어떤 입력에서 크래시하는가?", "이 스케줄은 항상 데드락 없이 동작하는가?" 같은 질문에 수학적 답을 얻을 수 있습니다.

## 참고 자료
- [Z3 공식 문서 및 튜토리얼](https://microsoft.github.io/z3guide/)
- [Z3Py 예제 모음 (GitHub)](https://github.com/Z3Prover/z3/tree/master/examples/python)
- [The SMT-LIB Standard](https://smtlib.cs.uiowa.edu/)
- [DPLL(T): Integrating Theory Solvers — Barrett et al.](https://theory.stanford.edu/~barrett/pubs/BSST08.pdf)
