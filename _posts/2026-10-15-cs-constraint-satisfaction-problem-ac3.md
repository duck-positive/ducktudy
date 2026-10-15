---
layout: post
title: "제약 만족 문제(CSP)와 AC-3 알고리즘: 아크 일관성과 백트래킹 탐색"
date: 2026-10-15
categories: [cs, computer-science]
tags: [CSP, constraint-satisfaction, AC-3, arc-consistency, backtracking, AI, search-algorithm]
---

## 제약 만족 문제(CSP)란?

**제약 만족 문제(Constraint Satisfaction Problem, CSP)**는 변수들의 집합과 각 변수의 가능한 값 범위(도메인), 그리고 변수들 사이의 제약 조건이 주어졌을 때, 모든 제약을 동시에 만족하는 변수 값 할당을 찾는 문제다.

CSP는 다음 세 요소로 정의된다.

- **X** = {X₁, X₂, …, Xₙ}: 변수 집합
- **D** = {D₁, D₂, …, Dₙ}: 각 변수의 도메인 (가능한 값의 집합)
- **C**: 제약 조건의 집합 (변수들의 관계)

### 친숙한 CSP 예시

| 문제 | 변수 | 도메인 | 제약 조건 |
|---|---|---|---|
| 스도쿠 | 81개 칸 | {1…9} | 행/열/박스 내 값 유일 |
| 지도 색칠 | 각 지역 | {빨,파,초,노} | 인접 지역은 다른 색 |
| N-Queens | 퀸 위치 | {1…N} | 공격 불가 |
| 시간표 작성 | 강의 배정 | {시간, 강의실} | 충돌 없음 |

일반적인 상태 공간 탐색과 달리, CSP는 **구조화된 문제 표현** 덕분에 훨씬 강력한 최적화가 가능하다. 특히 **제약 전파(Constraint Propagation)**를 통해 탐색 공간을 극적으로 줄일 수 있다.

---

## 아크 일관성(Arc Consistency)과 AC-3

### 아크 일관성이란?

**아크(Arc)**는 두 변수 사이의 유향 관계 (Xᵢ → Xⱼ)다. 아크 (Xᵢ, Xⱼ)가 **일관성**이 있으려면, Xᵢ의 도메인의 모든 값에 대해 제약 Cᵢⱼ를 만족하는 Xⱼ의 값이 존재해야 한다.

만약 Xᵢ = 빨강일 때 Xⱼ가 될 수 있는 값이 도메인에 없다면, Xᵢ에서 '빨강'을 제거해도 해(solution)를 잃지 않는다.

**AC-3(Arc Consistency Algorithm 3)**는 모든 아크를 큐에 넣고 순차 처리하며 도메인을 줄여나가는 알고리즘이다. 1977년 Alan Mackworth가 제안했다.

### AC-3 의사 코드

```
function AC-3(csp):
    queue ← 모든 아크 {(Xi, Xj)} 초기화
    while queue가 비어있지 않음:
        (Xi, Xj) ← queue에서 꺼냄
        if REVISE(csp, Xi, Xj):          # Xi 도메인 축소 발생?
            if Di가 비었으면:
                return FAILURE           # 해 없음
            for Xk in Xi의 이웃 (Xj 제외):
                queue에 (Xk, Xi) 추가   # Xi가 줄었으므로 재확인 필요
    return SUCCESS

function REVISE(csp, Xi, Xj):
    revised ← false
    for x in Di:
        if Di에서 Cij를 만족하는 y ∈ Dj가 없음:
            Di에서 x 제거
            revised ← true
    return revised
```

시간 복잡도: **O(ed³)** — e는 아크 수, d는 최대 도메인 크기.

---

## 코드 예제 1: AC-3 알고리즘 구현 (Python)

```python
from collections import deque

class CSP:
    def __init__(self, variables, domains, constraints):
        """
        variables:   변수 목록 ["A", "B", "C", ...]
        domains:     {변수: {가능한 값들}}
        constraints: {(Xi, Xj): 제약 함수 (xi, xj) -> bool}
        """
        self.variables   = variables
        self.domains     = {v: set(d) for v, d in domains.items()}
        self.constraints = constraints
        # 이웃 관계 빠른 조회를 위해 역방향 맵도 구성
        self.neighbors   = {v: set() for v in variables}
        for (xi, xj) in constraints:
            self.neighbors[xi].add(xj)
            self.neighbors[xj].add(xi)

    def revise(self, xi, xj):
        """Xi 도메인에서 Xj 도메인과 호환되지 않는 값 제거"""
        revised = False
        constraint = self.constraints.get((xi, xj)) or self.constraints.get((xj, xi))
        if constraint is None:
            return False

        to_remove = set()
        for x in self.domains[xi]:
            # xi = x 일 때 제약을 만족하는 xj 값이 존재하는지 확인
            satisfiable = any(
                constraint(x, y) if (xi, xj) in self.constraints else constraint(y, x)
                for y in self.domains[xj]
            )
            if not satisfiable:
                to_remove.add(x)
                revised = True

        self.domains[xi] -= to_remove
        return revised

    def ac3(self):
        """AC-3 알고리즘으로 아크 일관성 강제"""
        # 모든 방향의 아크로 큐 초기화
        queue = deque()
        for (xi, xj) in self.constraints:
            queue.append((xi, xj))
            queue.append((xj, xi))

        while queue:
            xi, xj = queue.popleft()
            if self.revise(xi, xj):
                if not self.domains[xi]:
                    print(f"  ✗ {xi}의 도메인이 비었습니다 → 해 없음")
                    return False
                print(f"  축소: {xi}의 도메인 → {sorted(self.domains[xi])}")
                # Xi 도메인이 줄었으므로, Xi의 이웃들에서 Xi를 향한 아크 재확인
                for xk in self.neighbors[xi] - {xj}:
                    queue.append((xk, xi))

        return True


# ─── 지도 색칠 문제 예시: 호주 지도 ───
# WA, NT, SA, Q, NSW, V, T 지역
# 인접: WA-NT, WA-SA, NT-SA, NT-Q, SA-Q, SA-NSW, SA-V, Q-NSW, NSW-V

variables = ["WA", "NT", "SA", "Q", "NSW", "V", "T"]
colors    = {"RED", "GREEN", "BLUE"}
domains   = {v: colors.copy() for v in variables}

# 인접 지역끼리 다른 색이어야 한다
adjacent = [
    ("WA","NT"), ("WA","SA"), ("NT","SA"), ("NT","Q"),
    ("SA","Q"),  ("SA","NSW"), ("SA","V"), ("Q","NSW"), ("NSW","V")
]
ne = lambda x, y: x != y
constraints = {pair: ne for pair in adjacent}

csp = CSP(variables, domains, constraints)
print("=== AC-3 실행 전 도메인 ===")
for v, d in csp.domains.items():
    print(f"  {v}: {sorted(d)}")

print("\n=== AC-3 실행 중 ===")
result = csp.ac3()

print(f"\n=== AC-3 결과: {'일관성 유지' if result else '실패'} ===")
for v, d in csp.domains.items():
    print(f"  {v}: {sorted(d)}")
```

지도 색칠 문제처럼 3색이면 AC-3만으로 도메인이 완전히 줄어들지 않을 수 있다. 이때는 **백트래킹** 탐색과 결합해야 한다.

---

## 코드 예제 2: AC-3 + 백트래킹으로 N-Queens 풀기

```python
def solve_nqueens(n):
    """N-Queens 문제를 CSP + AC-3 추론 + 백트래킹으로 해결"""
    # 변수: 행 번호 (0..n-1), 도메인: 열 위치 (0..n-1)
    variables = list(range(n))
    domains   = {i: set(range(n)) for i in variables}

    # 제약: 같은 열, 같은 대각선에 퀸 없어야 함
    def queens_ok(row1, col1, row2, col2):
        return (col1 != col2 and
                abs(col1 - col2) != abs(row1 - row2))

    def forward_check(assignment, var, value):
        """다음 변수들의 도메인에서 충돌 값 제거 (Forward Checking)"""
        pruned = {}
        for unassigned in variables:
            if unassigned in assignment:
                continue
            to_prune = set()
            for v in domains[unassigned]:
                if not queens_ok(var, value, unassigned, v):
                    to_prune.add(v)
            if to_prune:
                pruned[unassigned] = to_prune
                domains[unassigned] -= to_prune
            if not domains[unassigned]:
                # 도메인이 빔 → 실패, 되돌리기
                for u, pruned_vals in pruned.items():
                    domains[u] |= pruned_vals
                return None
        return pruned

    def mrv_variable(assignment):
        """최소 잔여 값(Minimum Remaining Values) 휴리스틱으로 변수 선택"""
        unassigned = [v for v in variables if v not in assignment]
        return min(unassigned, key=lambda v: len(domains[v]))

    def backtrack(assignment):
        if len(assignment) == n:
            return assignment

        var = mrv_variable(assignment)
        for value in sorted(domains[var]):
            assignment[var] = value

            pruned = forward_check(assignment, var, value)
            if pruned is not None:
                result = backtrack(assignment)
                if result:
                    return result
                # 되돌리기
                for u, pruned_vals in pruned.items():
                    domains[u] |= pruned_vals

            del assignment[var]

        return None

    solution = backtrack({})
    return solution


# 실행
for n in [4, 8, 12]:
    sol = solve_nqueens(n)
    if sol:
        print(f"\n{n}-Queens 해:")
        for row in range(n):
            col = sol[row]
            print("  " + ". " * col + "Q " + ". " * (n - col - 1))
    else:
        print(f"\n{n}-Queens: 해 없음")
```

이 구현은 세 가지 핵심 기법을 결합한다.
1. **AC-3** (아크 일관성): 탐색 전 또는 도중 도메인 사전 정리
2. **Forward Checking**: 값 배정 후 미래 변수 도메인 즉시 정리
3. **MRV (Minimum Remaining Values)**: 도메인이 가장 작은 변수부터 선택해 실패를 빨리 감지

---

## 제약 전파의 수준

아크 일관성 외에도 더 강한 일관성 개념들이 있다.

| 일관성 수준 | 설명 | 비용 |
|---|---|---|
| 노드 일관성 (NC) | 단항 제약 적용 | O(nd) |
| **아크 일관성 (AC)** | 이항 제약: AC-3 | O(ed³) |
| 경로 일관성 (PC) | 삼항 제약 | 매우 높음 |
| k-일관성 | k개 변수 간 모든 부분 할당 | 지수적 |

실무에서는 AC-3가 비용 대비 효과가 가장 좋아 가장 널리 사용된다.

---

## 다양한 휴리스틱

### 변수 선택 휴리스틱
- **MRV (Minimum Remaining Values)**: 도메인이 가장 작은 변수 먼저 → "fail-first"
- **Degree 휴리스틱**: 미배정 변수와 가장 많이 연결된 변수 먼저 → MRV 타이브레이커

### 값 선택 휴리스틱
- **LCV (Least Constraining Value)**: 이웃의 도메인을 가장 덜 줄이는 값 먼저 → "succeed-first"

---

## 주의사항과 팁

**도메인 복원 비용**: 백트래킹 시 도메인에서 제거한 값들을 복원해야 한다. 이를 효율적으로 관리하기 위해 각 단계에서 제거한 값을 스택에 기록한다.

**이진 CSP로 변환**: 3항 이상의 제약 조건은 보조 변수를 도입해 이진 CSP로 변환 가능하다. AC-3는 이진 제약에 특화됐기 때문이다.

**실수 도메인**: AC-3는 이산 도메인에 최적화돼 있다. 연속 도메인 CSP는 구간 산술(Interval Arithmetic)과 구간 전파(Interval Propagation) 기법을 사용한다.

**전역 제약 (Global Constraints)**: `alldifferent`, `sum`, `cardinality` 같은 전역 제약은 특수 알고리즘으로 훨씬 효율적으로 처리 가능하다. 현대 CSP 솔버(예: Choco, OR-Tools)는 이를 내장한다.

---

## 참고 자료
- [Constraint Satisfaction Problem — Wikipedia](https://en.wikipedia.org/wiki/Constraint_satisfaction_problem)
- [AC-3 Algorithm — Wikipedia](https://en.wikipedia.org/wiki/AC-3_algorithm)
- [CS 188: Artificial Intelligence — CSP Lecture (UC Berkeley)](https://inst.eecs.berkeley.edu/~cs188/sp19/assets/slides/lecture9.pdf)
- [Constraint Satisfaction Problems — University of Toronto CS 486](https://www.cs.toronto.edu/~axgao/cs486686_f21/lecture_notes/Lecture_04_on_Constraint_Satisfaction_Problems.pdf)
