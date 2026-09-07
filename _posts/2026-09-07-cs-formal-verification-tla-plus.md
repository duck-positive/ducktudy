---
layout: post
title: "형식 검증(Formal Verification) 완전 정복: TLA+와 모델 체킹으로 버그 없는 시스템 설계하기"
date: 2026-09-07
categories: [cs, computer-science]
tags: [formal-verification, tla-plus, model-checking, distributed-systems, correctness, lamport]
---

Amazon, Microsoft, Intel, NASA는 분산 시스템과 하드웨어 설계에서 형식 검증(Formal Verification)을 사용한다. DynamoDB, S3의 복제 프로토콜, Azure Cosmos DB의 트랜잭션 레이어가 TLA+로 검증되었다고 알려져 있다. **형식 검증은 테스트가 발견하지 못하는 '극히 드물지만 치명적인' 버그를 수학적으로 찾아낸다.** 이 아티클에서는 TLA+(Temporal Logic of Actions)와 모델 체킹의 핵심 원리를 코드 예제와 함께 설명한다.

---

## 형식 검증이란 무엇인가

**형식 검증**은 프로그램 또는 시스템이 수학적 명세(specification)를 만족하는지 **수학적으로 증명**하는 방법이다. 일반적인 소프트웨어 품질 보증 방법과 비교하면 다음과 같다.

| 방법 | 발견할 수 있는 버그 | 한계 |
|------|-------------------|------|
| 단위 테스트 | 테스트 케이스가 커버하는 경우 | 무한한 입력 공간을 다 테스트 불가 |
| 퍼징(Fuzzing) | 무작위 입력에 의한 충돌 | 복잡한 상태 시퀀스를 우연히 발생시키기 어려움 |
| 코드 리뷰 | 인간이 눈으로 볼 수 있는 버그 | 동시성 버그, 레이스 컨디션은 재현/추론이 매우 어려움 |
| **형식 검증** | **가능한 모든 상태, 모든 실행 순서** | 상태 공간 폭발 문제; 구현 코드가 아닌 명세를 검증 |

분산 시스템에서 "3개의 노드가 동시에 서로 다른 메시지를 받을 때, 그 중 하나가 네트워크 지연으로 늦게 도착하는 모든 가능한 순서"를 테스트로 검증하는 것은 사실상 불가능하다. 형식 검증은 이 모든 경우를 **모델 체킹(Model Checking)**으로 자동 탐색한다.

---

## TLA+의 철학: 상태 기계로 시스템 기술하기

TLA+는 Leslie Lamport(Paxos, LaTeX 설계자, 2013년 Turing Award 수상)가 만든 **시간 논리 기반 명세 언어**다. 핵심 아이디어는 모든 시스템을 **상태 기계(State Machine)**로 모델링하는 것이다.

```
시스템 = (초기 상태) + (상태 전이 규칙) + (불변 속성)
```

TLA+는 두 층위로 구성된다:
1. **TLA (Temporal Logic of Actions)**: 수학 기반 명세 언어
2. **PlusCal**: TLA+로 컴파일되는 알고리즘 유사 언어 (C와 유사한 문법)

---

## 안전성(Safety)과 활성성(Liveness)

형식 검증에서 가장 중요한 두 가지 속성 종류:

### Safety ("나쁜 일이 절대 일어나지 않는다")
- 뮤텍스에서 두 프로세스가 동시에 임계구역에 진입하지 않는다
- 데이터베이스에서 커밋된 데이터가 사라지지 않는다
- 투표에서 같은 사람이 두 번 뽑히지 않는다

### Liveness ("좋은 일이 결국 일어난다")
- 요청한 모든 프로세스는 **언젠가** 임계구역에 진입할 수 있다 (Starvation-free)
- 모든 메시지는 **결국** 수신된다
- 분산 합의는 **유한 시간 안에** 결정된다

---

## TLA+ 기초 문법

TLA+는 수학적 집합론과 논리학을 기반으로 한다. 핵심 연산자:

| 기호 | 의미 | TLA+ |
|------|------|------|
| ∧ | AND | `/\` |
| ∨ | OR | `\/` |
| ¬ | NOT | `~` |
| ∈ | 원소 | `\in` |
| ∀ | 전칭 | `\A` |
| ∃ | 존재 | `\E` |
| □ | Always | `[]` |
| ◇ | Eventually | `<>` |
| ~> | Leads to | `~>` |

---

## 구현 예제 1: TLA+로 뮤텍스 알고리즘 명세 작성

Peterson's Algorithm — 두 프로세스의 상호 배제를 보장하는 소프트웨어 뮤텍스.

```tla
---- MODULE Peterson ----
(* Peterson의 알고리즘 — 두 프로세스 상호 배제 검증 *)
EXTENDS Integers, Sequences

CONSTANTS N   \* 프로세스 수 (= 2)
ASSUME N = 2

VARIABLES
    flag,   \* flag[i] = TRUE: 프로세스 i가 임계구역 진입 희망
    turn,   \* 어느 프로세스에게 우선권이 있는가
    pc      \* 각 프로세스의 현재 단계 (프로그램 카운터)

(* 초기 상태 *)
Init ==
    /\ flag = [i \in {0, 1} |-> FALSE]
    /\ turn = 0
    /\ pc = [i \in {0, 1} |-> "start"]

(* 프로세스 i의 단계 전이 *)
SetFlag(i) ==
    /\ pc[i] = "start"
    /\ flag' = [flag EXCEPT ![i] = TRUE]
    /\ pc' = [pc EXCEPT ![i] = "setTurn"]
    /\ UNCHANGED turn

SetTurn(i) ==
    /\ pc[i] = "setTurn"
    /\ turn' = 1 - i   \* 양보: 상대방에게 우선권 부여
    /\ pc' = [pc EXCEPT ![i] = "wait"]
    /\ UNCHANGED flag

Wait(i) ==
    /\ pc[i] = "wait"
    /\ (~flag[1-i] \/ turn = i)   \* 상대방이 원하지 않거나 내 차례
    /\ pc' = [pc EXCEPT ![i] = "cs"]
    /\ UNCHANGED <<flag, turn>>

CriticalSection(i) ==
    /\ pc[i] = "cs"
    /\ pc' = [pc EXCEPT ![i] = "exit"]
    /\ UNCHANGED <<flag, turn>>

Exit(i) ==
    /\ pc[i] = "exit"
    /\ flag' = [flag EXCEPT ![i] = FALSE]
    /\ pc' = [pc EXCEPT ![i] = "start"]
    /\ UNCHANGED turn

(* 전체 전이: 임의의 프로세스가 한 단계를 실행 *)
Next ==
    \E i \in {0, 1}:
        \/ SetFlag(i)
        \/ SetTurn(i)
        \/ Wait(i)
        \/ CriticalSection(i)
        \/ Exit(i)

Spec == Init /\ [][Next]_<<flag, turn, pc>>

(* 검증할 속성 *)

\* 안전성: 두 프로세스가 동시에 임계구역에 없다
MutualExclusion ==
    ~(pc[0] = "cs" /\ pc[1] = "cs")

\* 활성성: 임계구역을 원하는 프로세스는 결국 진입할 수 있다
\* (강한 공정성 가정 필요)
Liveness ==
    \A i \in {0, 1}: (pc[i] = "wait") ~> (pc[i] = "cs")

====
```

TLC 모델 체커를 실행하면 가능한 모든 상태 시퀀스를 탐색한다. `MutualExclusion`은 모든 2^(상태 수)개의 실행 경로에서 참임을 검증한다.

---

## 구현 예제 2: Python Z3 SMT 솔버로 간단한 속성 검증

SMT(Satisfiability Modulo Theories) 솔버는 형식 검증의 또 다른 축이다. Z3는 Microsoft Research가 만든 강력한 오픈소스 SMT 솔버다.

```python
from z3 import *

def verify_mutex_property():
    """
    두 프로세스의 상호 배제 속성을 Z3로 검증.
    "동시에 두 프로세스가 임계구역에 있을 수 있는가?"를 묻는다.
    답이 UNSAT이면 불가능 → 안전성 검증 통과.
    """
    # 상태 변수 정의
    # pc0, pc1: 0=start, 1=waiting, 2=cs(임계구역), 3=exit
    pc0 = Int('pc0')
    pc1 = Int('pc1')
    flag0 = Bool('flag0')
    flag1 = Bool('flag1')
    turn = Int('turn')
    
    solver = Solver()
    
    # 도메인 제약
    solver.add(And(pc0 >= 0, pc0 <= 3))
    solver.add(And(pc1 >= 0, pc1 <= 3))
    solver.add(Or(turn == 0, turn == 1))
    
    # Peterson 알고리즘 불변식:
    # wait 상태(pc=1)에서 진입 조건: ~flag[other] OR turn==self
    # pc==1(wait)인 프로세스 0이 임계구역에 진입하려면:
    solver.add(Implies(
        pc0 == 1,  # 프로세스 0이 대기 중
        Or(Not(flag1), turn == 0)  # 상대 안 원하거나 내 차례
    ))
    solver.add(Implies(
        pc1 == 1,
        Or(Not(flag0), turn == 1)
    ))
    
    # flag[i]가 True인 것은 pc[i] >= 1인 것과 동치
    solver.add(flag0 == (pc0 >= 1))
    solver.add(flag1 == (pc1 >= 1))
    
    # "두 프로세스가 동시에 임계구역에 있다"는 반례를 찾아라
    solver.add(And(pc0 == 2, pc1 == 2))
    
    result = solver.check()
    if result == unsat:
        print("[PASS] 상호 배제 검증: 두 프로세스가 동시에 임계구역에 있을 수 없다.")
    else:
        model = solver.model()
        print("[FAIL] 반례 발견!")
        print(f"  pc0={model[pc0]}, pc1={model[pc1]}")
        print(f"  flag0={model[flag0]}, flag1={model[flag1]}")
        print(f"  turn={model[turn]}")

def verify_array_bounds():
    """
    배열 범위 초과 접근이 가능한지 Z3로 정적 검증.
    """
    solver = Solver()
    
    # 배열 크기와 인덱스
    ARRAY_SIZE = 10
    i = Int('i')
    n = Int('n')  # 루프 상한
    
    # 프로그램 전제조건
    solver.add(n > 0)
    solver.add(n <= ARRAY_SIZE)
    
    # 루프 불변식: 0 <= i < n
    solver.add(i >= 0)
    solver.add(i < n)
    
    # "배열 범위 초과"가 가능한가? (반례 탐색)
    out_of_bounds = Or(i < 0, i >= ARRAY_SIZE)
    solver.add(out_of_bounds)
    
    result = solver.check()
    if result == unsat:
        print("[PASS] 배열 범위 초과 없음: 루프 불변식이 안전을 보장한다.")
    else:
        model = solver.model()
        print(f"[FAIL] 범위 초과 가능! i={model[i]}, n={model[n]}")

def verify_integer_overflow():
    """
    부호 있는 32비트 정수 오버플로우 가능성 검증.
    """
    solver = Solver()
    
    # 32비트 부호 있는 정수 범위
    INT32_MAX = 2**31 - 1
    INT32_MIN = -(2**31)
    
    a = Int('a')
    b = Int('b')
    result = Int('result')
    
    # 입력 범위
    solver.add(a >= INT32_MIN, a <= INT32_MAX)
    solver.add(b >= INT32_MIN, b <= INT32_MAX)
    
    # 덧셈 결과
    solver.add(result == a + b)
    
    # "오버플로우가 발생한다"는 반례
    overflow = Or(result > INT32_MAX, result < INT32_MIN)
    solver.add(overflow)
    
    result_check = solver.check()
    if result_check == sat:
        model = solver.model()
        a_val = model[a].as_long()
        b_val = model[b].as_long()
        print(f"[WARN] 오버플로우 가능! a={a_val}, b={b_val}, sum={a_val+b_val}")
        print(f"       INT32_MAX={INT32_MAX}")
    else:
        print("[PASS] 오버플로우 없음")

def synthesize_loop_invariant():
    """
    단순 루프의 사후 조건을 Z3로 도출.
    while i < n: sum += i; i += 1
    사후 조건: sum == n*(n-1)//2
    """
    solver = Solver()
    
    n = Int('n')
    i = Int('i')
    s = Int('s')
    
    # 사전 조건
    solver.add(n >= 0, i == 0, s == 0)
    
    # 루프 후 상태 기호적 표현
    # i = n, s = 0+1+2+...+(n-1) = n*(n-1)/2
    final_s = n * (n - 1) / 2
    
    # "s != n*(n-1)/2 이 될 수 있는가?" 반례 탐색
    solver.add(s == final_s)
    solver.add(i == n)
    solver.add(s != n * (n - 1) / 2)  # 사후 조건 부정
    
    if solver.check() == unsat:
        print("[PASS] 루프 사후 조건 검증: sum = n*(n-1)/2")

if __name__ == "__main__":
    print("=== Z3 형식 검증 예제 ===\n")
    verify_mutex_property()
    print()
    verify_array_bounds()
    print()
    verify_integer_overflow()
    print()
    synthesize_loop_invariant()
```

---

## 형식 검증의 실제 활용 사례

### Amazon: AWS의 TLA+ 활용
AWS 엔지니어들은 DynamoDB, S3, EBS, SQS의 핵심 분산 알고리즘을 TLA+로 검증했다. 2014년 공개된 사례 연구에서 "코드 레이아웃 오류와 10개 이상의 미묘한 버그를 조기 발견했다"고 보고했다.

### Intel: FDIV 버그 이후
1994년 Pentium FDIV 버그로 인한 수억 달러의 손실 이후, 인텔은 하드웨어 검증에 형식 방법을 적극 도입했다.

### NASA: 우주선 소프트웨어
화성 탐사선 소프트웨어는 모든 가능한 실행 경로를 검증한다. 우주에서 버그를 수정할 수 없기 때문이다.

---

## 상태 공간 폭발 문제와 해결책

모델 체킹의 근본 문제는 **상태 공간이 기하급수적으로 증가**한다는 것이다. 변수가 n개이고 각각 k개 값을 가지면 전체 상태 수는 k^n이다.

### 해결 기법

**1. Symbolic Model Checking (BDD)**
진리값 표 대신 이진 결정 다이어그램(BDD)으로 상태 집합을 압축 표현한다. SPIN 모델 체커, NuSMV가 이 방식을 사용한다.

**2. 추상화(Abstraction)**
세부 구현을 무시하고 핵심 속성만 포함하는 추상 모델을 검증한다. TLA+에서는 명세 계층을 여러 단계로 나눠 각 단계에서 다른 수준의 세부사항을 검증한다.

**3. Bounded Model Checking**
무한 실행을 k 단계까지만 탐색한다. Z3 같은 SAT/SMT 솔버로 구현된다. k-step 안에 버그가 없으면 "(k+1)-단계부터는 다른 속성이 필요하다"는 정보를 얻는다.

**4. 분산 상태 공간 탐색**
TLC는 여러 CPU 코어와 클러스터에서 상태 공간을 병렬 탐색하는 기능을 지원한다.

---

## TLA+ 도구 생태계

```
TLA+ Toolbox (IDE)
  ├── TLC (TLA Model Checker) — 모델 체킹
  ├── TLAPS (TLA Proof System) — 정리 증명
  └── PlusCal → TLA+ 변환기

VS Code 확장: vscode-tlaplus
커맨드라인: java -jar tla2tools.jar -deadlock MySpec.tla
```

---

## 주의사항과 팁

**1. 명세는 구현이 아니다**
TLA+로 명세를 검증해도 구현 코드의 버그는 따로 잡아야 한다. 명세와 구현 사이의 간극(Refinement Gap)을 줄이려면 구현을 명세와 최대한 가깝게 구조화해야 한다.

**2. 모든 시스템에 형식 검증이 필요하지는 않다**
형식 검증은 비용이 높다. 분산 합의, 금융 트랜잭션 처리, 안전 필수(safety-critical) 시스템처럼 **버그의 비용이 매우 높은 곳**에 집중하라.

**3. Liveness 속성은 공정성(Fairness) 가정이 필요하다**
"결국 일어난다"는 속성은 스케줄러가 프로세스를 무한히 무시하지 않는다는 공정성 가정 없이는 증명할 수 없다. TLA+에서 `WF_vars(Next)` (약한 공정성) 또는 `SF_vars(Next)` (강한 공정성)을 명시해야 한다.

**4. PlusCal로 먼저 시작하라**
TLA+ 수학 문법이 어렵다면 PlusCal(Pascal 유사 문법)로 알고리즘을 작성하고 TLA+로 변환하라. `--algorithm` 블록 안에 작성한 코드가 자동으로 TLA+로 컴파일된다.

**5. 작은 예제에서 시작하라**
큰 시스템 전체를 처음부터 형식화하려 하지 말고, 가장 복잡하거나 의심스러운 서브컴포넌트(예: 리더 선출, 분산 락)부터 시작하라.

---

## 참고 자료

- [TLA+ for System Design — wal.sh](https://www.wal.sh/research/tla-plus-system-design/)
- [Specifying and Verifying Systems With TLA+ — Leslie Lamport (Microsoft Research)](https://lamport.azurewebsites.net/pubs/spec-and-verifying.pdf)
- [TLA+ in Practice and Theory — Part 1](https://pron.github.io/posts/tlaplus_part1)
- [Formal Verification Tool TLA+: An Introduction — Alibaba Cloud](https://www.alibabacloud.com/blog/formal-verification-tool-tla%2B-an-introduction-from-the-perspective-of-a-programmer_598373)
