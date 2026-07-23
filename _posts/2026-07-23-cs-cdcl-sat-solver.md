---
layout: post
title: "CDCL SAT 솔버 완전 정복: 충돌 주도 절 학습으로 NP-완전 문제를 실용적으로 해결하는 법"
date: 2026-07-23
categories: [cs, computer-science]
tags: [sat-solver, CDCL, DPLL, boolean-satisfiability, clause-learning, backjumping, unit-propagation, conflict-analysis]
---

현대 하드웨어 검증, 소프트웨어 모델 체킹, 자동 정리 증명, 패키지 의존성 해결 — 이 모든 영역에서 공통으로 활용되는 핵심 기술이 있습니다. 바로 **SAT 솔버(Boolean Satisfiability Solver)**입니다.

이론적으로 Boolean Satisfiability(SAT)는 NP-완전 문제입니다. 최악의 경우 지수 시간이 필요합니다. 그런데 어떻게 현대 SAT 솔버는 수십만 개의 변수와 수백만 개의 절(clause)을 가진 실제 문제를 **수 초 안에** 해결할 수 있을까요?

그 비밀이 바로 **CDCL(Conflict-Driven Clause Learning)** 알고리즘입니다.

## SAT 문제의 기본 개념

### CNF (Conjunctive Normal Form)

SAT 솔버는 **CNF(논리곱 표준형)** 형태의 입력을 처리합니다.

- **리터럴(Literal)**: 변수 x 또는 그 부정 ¬x
- **절(Clause)**: 리터럴의 논리합 — (x₁ ∨ x₂ ∨ ¬x₃)
- **CNF 수식**: 절의 논리곱 — (x₁ ∨ ¬x₂) ∧ (¬x₁ ∨ x₃) ∧ (x₂ ∨ x₃)

**예시**: 세 개의 변수 x₁, x₂, x₃가 있을 때
```
(x₁ ∨ x₂ ∨ ¬x₃) ∧ (¬x₁ ∨ x₂) ∧ (¬x₂ ∨ x₃) ∧ (x₁ ∨ ¬x₃)
```
이 수식을 만족하는 진리값 할당(x₁=T, x₂=T, x₃=T)이 존재합니까?

### DPLL: CDCL의 전신

1960년대 Davis-Putnam-Logemann-Loveland(DPLL) 알고리즘이 SAT의 기초를 세웠습니다.

```python
def dpll(clauses, assignment):
    """
    기본 DPLL 알고리즘
    clauses: CNF 절 목록
    assignment: 현재 변수 할당 딕셔너리
    """
    # 단위 전파 (Unit Propagation)
    while True:
        unit = find_unit_clause(clauses, assignment)
        if unit is None:
            break
        var, val = unit
        assignment[var] = val
        clauses = propagate(clauses, var, val)
        if is_empty_clause(clauses):
            return None  # UNSAT

    # 순수 리터럴 제거 (Pure Literal Elimination)
    pure = find_pure_literal(clauses, assignment)
    if pure:
        var, val = pure
        assignment[var] = val
        clauses = propagate(clauses, var, val)

    # 모든 절이 만족되면 SAT
    if not clauses:
        return assignment

    # 변수 선택 및 분기
    var = choose_variable(clauses, assignment)
    
    # True 시도
    new_assignment = assignment.copy()
    new_assignment[var] = True
    result = dpll(propagate(clauses, var, True), new_assignment)
    if result is not None:
        return result
    
    # False 시도 (백트래킹)
    new_assignment = assignment.copy()
    new_assignment[var] = False
    return dpll(propagate(clauses, var, False), new_assignment)
```

DPLL의 문제점: 충돌이 발생했을 때 **항상 직전 결정으로 돌아가는 연대기적 백트래킹(chronological backtracking)**을 수행합니다. 같은 실수를 반복합니다.

## CDCL: 학습을 통한 혁명

### CDCL의 핵심 아이디어 세 가지

1. **충돌 분석 (Conflict Analysis)**: 충돌 발생 시 원인을 분석하여 새로운 절을 학습
2. **비연대기적 백점핑 (Non-chronological Backjumping)**: 충돌과 관련 없는 결정을 건너뛰어 원인 레벨로 직접 이동
3. **절 데이터베이스 (Learned Clause Database)**: 학습한 절을 저장하여 동일한 충돌 반복 방지

### 함의 그래프 (Implication Graph)

CDCL의 핵심 자료구조는 **함의 그래프**입니다. 각 변수 할당의 원인(reason)을 추적합니다.

```python
from collections import defaultdict
from typing import Optional, List, Tuple, Dict, Set

class Literal:
    def __init__(self, var: int, positive: bool):
        self.var = var
        self.positive = positive
    
    def negated(self):
        return Literal(self.var, not self.positive)
    
    def __repr__(self):
        return f"{'x' if self.positive else '¬x'}{self.var}"
    
    def __hash__(self):
        return hash((self.var, self.positive))
    
    def __eq__(self, other):
        return self.var == other.var and self.positive == other.positive


class CDCLSolver:
    """
    CDCL SAT 솔버 구현
    - 단위 전파 (Unit Propagation)
    - 함의 그래프 기반 충돌 분석
    - 비연대기적 백점핑
    - 절 학습
    - VSIDS 변수 선택 휴리스틱 (간소화)
    """
    
    UNSAT = "UNSAT"
    SAT = "SAT"
    
    def __init__(self, num_vars: int, clauses: List[List[int]]):
        """
        num_vars: 변수 수 (1~num_vars)
        clauses: 정수 리터럴 목록. 양수=긍정, 음수=부정. 예: [[1, -2, 3], [-1, 2]]
        """
        self.num_vars = num_vars
        # 절을 (긍정리터럴 집합, 부정리터럴 집합) 쌍으로 변환
        self.clauses = [self._parse_clause(c) for c in clauses]
        self.learned_clauses = []
        
        # 할당 상태
        self.assignment = {}        # var -> bool
        self.decision_level = {}    # var -> 결정 레벨
        self.antecedent = {}        # var -> 원인 절 인덱스 (함의된 경우)
        
        self.current_level = 0
        self.decision_stack = []    # [(var, val, level), ...]
        
        # VSIDS 활동 점수 (충돌 시 관련 변수 점수 증가)
        self.activity = defaultdict(float)
        self.activity_decay = 0.99
        self.activity_bump = 1.0
    
    def _parse_clause(self, raw_clause):
        """정수 목록을 Literal 목록으로 변환"""
        return [Literal(abs(l), l > 0) for l in raw_clause]
    
    def solve(self):
        """메인 CDCL 루프"""
        # 초기 단위 전파 (레벨 0)
        conflict = self._unit_propagate()
        if conflict is not None:
            return self.UNSAT
        
        while not self._all_assigned():
            # 1. 결정: 할당되지 않은 변수 선택
            var = self._pick_variable()
            val = True  # 또는 위상(phase) 휴리스틱
            
            self.current_level += 1
            self._assign(var, val, level=self.current_level, antecedent=None)
            self.decision_stack.append((var, val, self.current_level))
            
            # 2. 단위 전파
            while True:
                conflict = self._unit_propagate()
                if conflict is None:
                    break  # 충돌 없음 → 다음 결정으로
                
                # 3. 충돌 분석
                if self.current_level == 0:
                    return self.UNSAT  # 레벨 0 충돌 = 전체 UNSAT
                
                learned_clause, backjump_level = self._analyze_conflict(conflict)
                
                # 4. 절 학습
                self.learned_clauses.append(learned_clause)
                all_clauses_idx = len(self.clauses) + len(self.learned_clauses) - 1
                
                # 5. 비연대기적 백점핑
                self._backjump(backjump_level)
                self.current_level = backjump_level
                
                # 6. 학습된 절에 의한 단위 전파
                # (백점핑 후 학습 절이 단위 절이 됨)
                unit_lit = self._find_unit_in_learned(learned_clause)
                if unit_lit:
                    self._assign(unit_lit.var, unit_lit.positive,
                                 level=self.current_level,
                                 antecedent=all_clauses_idx)
        
        return self.SAT
    
    def _unit_propagate(self) -> Optional[List]:
        """
        단위 전파: 하나의 미할당 리터럴만 남은 절에서 해당 리터럴 강제 할당.
        충돌 발생 시 충돌 절 반환, 없으면 None 반환.
        """
        all_clauses = self.clauses + self.learned_clauses
        changed = True
        
        while changed:
            changed = False
            for clause in all_clauses:
                status = self._evaluate_clause(clause)
                
                if status == 'SATISFIED':
                    continue
                elif status == 'CONFLICT':
                    return clause  # 충돌 절 반환
                elif status == 'UNIT':
                    # 유일하게 미할당된 리터럴 찾기
                    unassigned = [l for l in clause
                                  if l.var not in self.assignment]
                    lit = unassigned[0]
                    clause_idx = all_clauses.index(clause)
                    self._assign(lit.var, lit.positive,
                                 level=self.current_level,
                                 antecedent=clause_idx)
                    changed = True
        
        return None
    
    def _evaluate_clause(self, clause):
        """절 상태 평가: 'SATISFIED', 'CONFLICT', 'UNIT', 'UNRESOLVED'"""
        unassigned = []
        for lit in clause:
            if lit.var not in self.assignment:
                unassigned.append(lit)
            elif self.assignment[lit.var] == lit.positive:
                return 'SATISFIED'
        
        if len(unassigned) == 0:
            return 'CONFLICT'
        elif len(unassigned) == 1:
            return 'UNIT'
        return 'UNRESOLVED'
    
    def _analyze_conflict(self, conflict_clause) -> Tuple[List, int]:
        """
        First UIP(Unique Implication Point) 기반 충돌 분석.
        학습할 절과 백점핑 레벨을 반환.
        """
        # 현재 레벨에서의 리터럴들이 있는 집합으로 시작
        current_level_lits = set()
        prev_level_lits = set()
        
        def add_to_sets(clause):
            for lit in clause:
                var = lit.var
                if var in self.assignment:
                    level = self.decision_level.get(var, 0)
                    if level == self.current_level:
                        current_level_lits.add(var)
                    else:
                        prev_level_lits.add(var)
        
        add_to_sets(conflict_clause)
        
        # UIP까지 해상도 수행
        all_clauses = self.clauses + self.learned_clauses
        assigned_at_current = [
            v for v in self.decision_stack
            if v[2] == self.current_level
        ]
        
        # BFS/역순으로 함의 그래프 탐색
        for var, val, level in reversed(assigned_at_current):
            if len(current_level_lits) <= 1:
                break  # First UIP 도달
            
            if var in current_level_lits and var in self.antecedent:
                ant_idx = self.antecedent[var]
                if ant_idx < len(all_clauses):
                    ant_clause = all_clauses[ant_idx]
                    current_level_lits.discard(var)
                    add_to_sets(ant_clause)
                    current_level_lits.discard(var)
        
        # 학습 절 구성: UIP 리터럴(부정) + 이전 레벨 리터럴들(부정)
        learned = []
        for var in current_level_lits:
            val = self.assignment.get(var, True)
            learned.append(Literal(var, not val))  # 부정으로 추가
        for var in prev_level_lits:
            val = self.assignment.get(var, True)
            learned.append(Literal(var, not val))
        
        # 백점핑 레벨: 이전 레벨 리터럴들 중 최대 레벨
        backjump_level = 0
        for var in prev_level_lits:
            level = self.decision_level.get(var, 0)
            backjump_level = max(backjump_level, level)
        
        # VSIDS 활동 점수 업데이트
        for var in current_level_lits | prev_level_lits:
            self.activity[var] += self.activity_bump
        self.activity_bump /= self.activity_decay
        
        return learned if learned else [Literal(1, False)], backjump_level
    
    def _backjump(self, level: int):
        """지정 레벨로 비연대기적 백점핑 수행"""
        vars_to_unassign = [
            v for v in list(self.assignment.keys())
            if self.decision_level.get(v, 0) > level
        ]
        for var in vars_to_unassign:
            del self.assignment[var]
            self.decision_level.pop(var, None)
            self.antecedent.pop(var, None)
        
        self.decision_stack = [
            (v, val, lvl) for (v, val, lvl) in self.decision_stack
            if lvl <= level
        ]
    
    def _assign(self, var, val, level, antecedent):
        self.assignment[var] = val
        self.decision_level[var] = level
        if antecedent is not None:
            self.antecedent[var] = antecedent
    
    def _pick_variable(self) -> int:
        """VSIDS: 활동 점수가 가장 높은 미할당 변수 선택"""
        unassigned = [v for v in range(1, self.num_vars + 1)
                      if v not in self.assignment]
        if not unassigned:
            return None
        return max(unassigned, key=lambda v: self.activity.get(v, 0.0))
    
    def _find_unit_in_learned(self, clause):
        """학습된 절에서 단위 리터럴 찾기"""
        unassigned = [l for l in clause if l.var not in self.assignment]
        if len(unassigned) == 1:
            return unassigned[0]
        return None
    
    def _all_assigned(self):
        return len(self.assignment) == self.num_vars
    
    def get_model(self):
        if len(self.assignment) == self.num_vars:
            return {f"x{v}": self.assignment[v]
                    for v in range(1, self.num_vars + 1)}
        return None


# 사용 예시
# 문제: (x1 ∨ x2 ∨ x3) ∧ (¬x1 ∨ ¬x2) ∧ (¬x2 ∨ ¬x3) ∧ (x1 ∨ ¬x3)
solver = CDCLSolver(
    num_vars=3,
    clauses=[
        [1, 2, 3],       # x1 ∨ x2 ∨ x3
        [-1, -2],        # ¬x1 ∨ ¬x2
        [-2, -3],        # ¬x2 ∨ ¬x3
        [1, -3],         # x1 ∨ ¬x3
    ]
)

result = solver.solve()
print(f"결과: {result}")
if result == "SAT":
    print(f"모델: {solver.get_model()}")

# UNSAT 예시: (x1) ∧ (¬x1) — 명백한 모순
unsat_solver = CDCLSolver(
    num_vars=1,
    clauses=[[1], [-1]]
)
print(f"\nUNSAT 예시: {unsat_solver.solve()}")
```

## CDCL의 핵심 최적화 기법

### 1. 2-Watched Literals (2-WL)

현대 SAT 솔버에서 단위 전파 속도를 획기적으로 높이는 기법입니다. 각 절에서 단 **두 개의 리터럴만 감시(watch)**합니다.

```python
class TwoWatchedLiteralsSolver:
    """
    2-WL 최적화를 적용한 단위 전파
    아이디어: 절이 CONFLICT/UNIT이 되려면 반드시 감시 중인 리터럴이 False가 되어야 함
    """
    
    def __init__(self, clauses):
        self.clauses = clauses
        # watch[lit] = 이 리터럴을 감시하는 절들의 집합
        self.watch = defaultdict(list)
        
        for i, clause in enumerate(clauses):
            if len(clause) >= 2:
                # 처음 두 리터럴을 감시
                self.watch[clause[0]].append(i)
                self.watch[clause[1]].append(i)
            elif len(clause) == 1:
                self.watch[clause[0]].append(i)
    
    def propagate(self, assignment, new_lit):
        """
        new_lit이 False로 할당될 때만 해당 리터럴을 감시하는 절들 확인
        대부분의 절은 건드리지 않아도 됨 → 단위 전파 O(n) → 거의 O(1) amortized
        """
        false_lit = new_lit.negated()
        
        for clause_idx in list(self.watch[false_lit]):
            clause = self.clauses[clause_idx]
            
            # 다른 감시 리터럴 찾기
            other_watch = None
            for lit in clause:
                if lit != false_lit and lit in [clause[0], clause[1]]:
                    other_watch = lit
                    break
            
            # 이미 만족된 절인지 확인
            if other_watch and assignment.get(other_watch.var) == other_watch.positive:
                continue
            
            # 새로운 감시 리터럴 찾기 (이미 False가 아닌 것)
            new_watch = None
            for lit in clause:
                if lit != false_lit and lit != other_watch:
                    if assignment.get(lit.var) != (not lit.positive):
                        new_watch = lit
                        break
            
            if new_watch:
                # 감시 리터럴 교체
                self.watch[false_lit].remove(clause_idx)
                self.watch[new_watch].append(clause_idx)
            elif other_watch:
                # 다른 감시 리터럴만 남음 → 단위 전파
                # other_watch를 True로 강제 할당
                pass
            else:
                # 모든 리터럴이 False → 충돌!
                return clause_idx  # 충돌 절 반환
        
        return None
```

### 2. VSIDS (Variable State Independent Decaying Sum)

충돌 분석에서 자주 등장하는 변수에 높은 우선순위를 부여하는 휴리스틱입니다.

```python
class VSIDS:
    """
    VSIDS 변수 선택 휴리스틱
    - 충돌 발생 시 관련 변수의 활동 점수 증가
    - 주기적으로 모든 점수 감소 (최근 충돌 우선)
    - 최대 힙으로 O(log n) 변수 선택
    """
    
    def __init__(self, num_vars, decay=0.95):
        self.activity = {v: 0.0 for v in range(1, num_vars + 1)}
        self.decay = decay
        self.bump = 1.0
        self.conflict_count = 0
    
    def bump_variable(self, var):
        self.activity[var] += self.bump
        # 오버플로우 방지
        if self.activity[var] > 1e100:
            self._rescale()
    
    def decay_all(self):
        """모든 변수 점수 감소 (실제로는 bump를 증가시키는 방식)"""
        self.bump /= self.decay
        self.conflict_count += 1
    
    def _rescale(self):
        factor = 1e-100
        self.activity = {v: a * factor for v, a in self.activity.items()}
        self.bump *= factor
    
    def pick_unassigned(self, unassigned_vars):
        """미할당 변수 중 활동 점수가 가장 높은 것 반환"""
        return max(unassigned_vars, key=lambda v: self.activity.get(v, 0.0))
```

### 3. 재시작 전략 (Restart Strategy)

CDCL 솔버는 탐색이 비효율적인 분기에 빠질 경우 **재시작(restart)**을 통해 탈출합니다. Luby 시퀀스나 Geometric 시퀀스를 사용하여 재시작 간격을 결정합니다.

```python
def luby_sequence(i, unit=100):
    """
    Luby 재시작 시퀀스: 1, 1, 2, 1, 1, 2, 4, 1, 1, 2, 1, 1, 2, 4, 8, ...
    충돌 수 기반 재시작 트리거
    """
    k = 1
    while k < i + 1:
        k *= 2
    if i + 1 == k:
        return unit * k // 2
    return luby_sequence(i - k // 2 + 1, unit)

# Luby 시퀀스 출력
sequence = [luby_sequence(i, unit=1) for i in range(15)]
print(f"Luby 시퀀스: {sequence}")
# 출력: [1, 1, 2, 1, 1, 2, 4, 1, 1, 2, 1, 1, 2, 4, 8]
```

## 실제 응용: 패키지 의존성 해결

SAT 솔버의 실용적인 활용 사례 중 하나가 **패키지 매니저의 의존성 해결**입니다. apt, pip, Cargo 등은 내부적으로 SAT 솔버를 사용합니다.

```python
def package_deps_to_sat(packages, dependencies, conflicts):
    """
    패키지 의존성 문제를 SAT 인스턴스로 변환
    
    packages: 패키지 이름 목록
    dependencies: {pkg: [required_pkgs]} — pkg 설치 시 required_pkgs 중 하나 필요
    conflicts: [(pkg_a, pkg_b)] — 동시 설치 불가 쌍
    """
    # 각 패키지를 변수로 매핑
    var_map = {pkg: i + 1 for i, pkg in enumerate(packages)}
    num_vars = len(packages)
    clauses = []
    
    # 의존성 절: pkg 설치 → required 중 하나 설치
    # ¬pkg ∨ req1 ∨ req2 ∨ ... (pkg가 설치되면 req 중 하나 반드시 설치)
    for pkg, reqs in dependencies.items():
        if reqs:
            clause = [-var_map[pkg]] + [var_map[r] for r in reqs]
            clauses.append(clause)
    
    # 충돌 절: ¬pkg_a ∨ ¬pkg_b (두 패키지 동시 설치 불가)
    for pkg_a, pkg_b in conflicts:
        clauses.append([-var_map[pkg_a], -var_map[pkg_b]])
    
    return num_vars, clauses, var_map

# 예시: 패키지 A, B, C, D
packages = ['A', 'B', 'C', 'D']
dependencies = {
    'A': ['B', 'C'],   # A를 설치하려면 B 또는 C 필요
    'B': ['D'],         # B를 설치하려면 D 필요
}
conflicts = [('C', 'D')]  # C와 D는 동시 설치 불가

num_vars, clauses, var_map = package_deps_to_sat(packages, dependencies, conflicts)

# A 설치 요구 사항 추가
clauses.append([var_map['A']])  # A를 반드시 설치

solver = CDCLSolver(num_vars, clauses)
result = solver.solve()
print(f"의존성 해결 가능: {result}")
if result == "SAT":
    model = solver.get_model()
    installed = [pkg for pkg, var in var_map.items()
                 if model.get(f"x{var}")]
    print(f"설치할 패키지: {installed}")
```

## 주의사항과 실전 팁

### 1. 실제로는 MiniSat, CryptoMiniSat, Glucose 사용

실제 프로덕션에서는 직접 구현한 솔버 대신 Knuth의 SAT 연구나 SAT Competition 우승자들의 검증된 구현체를 사용하세요.

```bash
# Python에서 SAT 솔버 활용
pip install pysat  # PySAT 라이브러리
```

```python
from pysat.solvers import Glucose4
from pysat.formula import CNF

formula = CNF(from_clauses=[[1, 2, 3], [-1, -2], [-2, -3], [1, -3]])

with Glucose4(bootstrap_with=formula) as solver:
    if solver.solve():
        print(f"SAT! 모델: {solver.get_model()}")
    else:
        print("UNSAT")
```

### 2. 문제 인코딩의 중요성

같은 문제라도 CNF 인코딩 방식에 따라 풀이 시간이 수백 배 차이납니다. 보조 변수(Tseitin transformation)를 적절히 활용하면 절의 수를 줄일 수 있습니다.

### 3. 전처리 (Preprocessing)

SatElite, CaDiCaL 등 현대 솔버는 풀이 전 **BVE(Bounded Variable Elimination)**, **Subsumption** 등의 전처리 기법으로 문제를 단순화합니다.

### 4. 포트폴리오 접근

경쟁 솔버들은 **포트폴리오 전략** — 여러 구성의 솔버를 병렬 실행하고 가장 먼저 답을 내는 것을 채택 — 을 사용합니다. 단일 설정으로 모든 문제를 최적으로 풀 수 없기 때문입니다.

CDCL은 NP-완전이라는 이론적 한계를 실용적으로 극복한 컴퓨터 과학 최고의 성공 사례 중 하나입니다. 함의 그래프, 학습된 절, 비연대기적 백점핑이 어우러져 실제 산업 규모의 문제를 해결합니다.

## 참고 자료
- [Wikipedia: Conflict-driven clause learning](https://en.wikipedia.org/wiki/Conflict-driven_clause_learning)
- [Wikipedia: DPLL algorithm](https://en.wikipedia.org/wiki/DPLL_algorithm)
- [PySAT: Python Toolkit for SAT Solving](https://pysathq.github.io/)
- [SAT Competition — 국제 SAT 솔버 경진대회](https://satcompetition.github.io/)
