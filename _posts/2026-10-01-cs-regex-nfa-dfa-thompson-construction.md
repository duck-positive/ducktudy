---
layout: post
title: "정규 표현식 엔진 내부 구조: Thompson NFA와 Powerset 구성법으로 DFA 변환하기"
date: 2026-10-01
categories: [cs, computer-science]
tags: [regex, nfa, dfa, automata, thompson-construction, powerset-construction, finite-automata, compiler]
---

## 정규 표현식 엔진이란

정규 표현식(Regular Expression)은 문자열 패턴을 기술하는 언어이다. `a(b|c)*d` 같은 패턴을 입력받아 임의의 문자열이 그 패턴에 매칭되는지 판별하는 것이 정규 표현식 엔진의 역할이다. 언뜻 간단해 보이지만, 이 엔진 내부에는 정교한 오토마타 이론이 숨어 있다.

정규 표현식 엔진의 핵심 파이프라인은 다음과 같다:

```
정규 표현식 문자열
      ↓ (파싱)
   구문 트리 (AST)
      ↓ (Thompson 구성법)
   NFA (비결정적 유한 오토마톤)
      ↓ (Powerset 구성법, 선택적)
   DFA (결정적 유한 오토마톤)
      ↓ (Hopcroft 최소화, 선택적)
  최소화된 DFA
      ↓
   매칭 실행
```

---

## 왜 NFA와 DFA가 필요한가

### 유한 오토마톤(Finite Automaton)

유한 오토마톤은 상태(state)와 전이(transition)로 구성된 계산 모델이다:
- **상태 집합 Q**: 오토마톤이 가질 수 있는 모든 상태
- **알파벳 Σ**: 입력 문자들의 집합
- **전이 함수 δ**: 현재 상태 + 입력 문자 → 다음 상태
- **시작 상태 q₀**
- **수락 상태 집합 F**

### NFA vs DFA

| 특성 | NFA | DFA |
|------|-----|-----|
| 한 상태에서 같은 입력에 대해 | 여러 전이 가능 | 정확히 하나의 전이 |
| ε-전이 | 가능 | 불가 |
| 구성 | 직관적, 컴파일 쉬움 | 복잡하지만 실행이 빠름 |
| 실행 복잡도 | O(n·m) (n=상태수, m=입력 길이) | O(m) |
| 상태 수 | O(n) | 최악 O(2^n) |

NFA는 정규 표현식에서 직접 변환하기 쉽지만, 실행 시 "어느 경로를 택해야 하는가"를 결정할 수 없어 여러 경로를 동시에 추적해야 한다. DFA는 항상 결정론적이라 빠르게 실행되지만 상태 폭발 문제가 있다.

---

## Thompson 구성법: 정규 표현식 → NFA

Kenneth Thompson이 1968년 제안한 알고리즘으로, 정규 표현식의 구조를 재귀적으로 NFA 조각으로 변환한다.

### 기본 NFA 조각

**단일 문자 `a`**:
```
→ (q0) --a--> ((q1))
```
시작 상태 q0에서 문자 `a`를 읽으면 수락 상태 q1로 전이.

**ε-전이 (빈 문자열)**:
```
→ (q0) --ε--> ((q1))
```

### 연결(Concatenation): `ab`

두 NFA `M1`, `M2`를 연결. M1의 수락 상태와 M2의 시작 상태를 ε-전이로 연결:
```
→ (s1) --a--> (a1) --ε--> (s2) --b--> ((a2))
```

### 선택(Alternation): `a|b`

새 시작/수락 상태를 만들고 두 NFA를 병렬로 연결:
```
         (s1) --a--> (a1)
        /ε              \ε
→ (s0)                   ((f0))
        \ε              /ε
         (s2) --b--> (a2)
```

### 클린 스타(Kleene Star): `a*`

새 시작/수락 상태를 추가하고 반복 가능한 루프 생성:
```
        ┌──────ε──────┐
        ↓             │
→ (s0) --ε--> (s1) --a--> (a1) --ε--> ((f0))
   │                                      ↑
   └──────────────────ε───────────────────┘
```

---

## Python으로 Thompson NFA 직접 구현

아래는 `a(b|c)*d` 같은 정규 표현식을 NFA로 변환하고 문자열 매칭을 수행하는 완전한 구현이다:

```python
from dataclasses import dataclass, field
from typing import Optional, FrozenSet

# NFA 상태
_state_counter = 0
def new_state() -> int:
    global _state_counter
    _state_counter += 1
    return _state_counter

@dataclass
class NFA:
    start: int
    accept: int
    # 전이 테이블: {상태: {문자 또는 'ε': {다음 상태들}}}
    transitions: dict = field(default_factory=dict)

    def add_transition(self, from_: int, char: str, to: int):
        self.transitions.setdefault(from_, {}).setdefault(char, set()).add(to)

    def epsilon_closure(self, states: FrozenSet[int]) -> FrozenSet[int]:
        """ε-전이로 도달 가능한 모든 상태의 집합 반환"""
        closure = set(states)
        stack = list(states)
        while stack:
            state = stack.pop()
            for next_state in self.transitions.get(state, {}).get('ε', set()):
                if next_state not in closure:
                    closure.add(next_state)
                    stack.append(next_state)
        return frozenset(closure)

    def move(self, states: FrozenSet[int], char: str) -> FrozenSet[int]:
        """주어진 상태 집합에서 char로 전이 가능한 모든 상태 반환"""
        result = set()
        for state in states:
            result |= self.transitions.get(state, {}).get(char, set())
        return frozenset(result)

    def accepts(self, string: str) -> bool:
        """문자열을 NFA로 매칭 (서브셋 시뮬레이션)"""
        current = self.epsilon_closure(frozenset([self.start]))
        for char in string:
            current = self.epsilon_closure(self.move(current, char))
        return self.accept in current


def build_char(c: str) -> NFA:
    """단일 문자 NFA 생성"""
    s, a = new_state(), new_state()
    nfa = NFA(start=s, accept=a)
    nfa.add_transition(s, c, a)
    return nfa

def build_concat(n1: NFA, n2: NFA) -> NFA:
    """연결 NFA: n1 다음에 n2"""
    merged = NFA(start=n1.start, accept=n2.accept)
    # n1의 전이 복사
    for s, trans in n1.transitions.items():
        for c, targets in trans.items():
            for t in targets:
                merged.add_transition(s, c, t)
    # n2의 전이 복사
    for s, trans in n2.transitions.items():
        for c, targets in trans.items():
            for t in targets:
                merged.add_transition(s, c, t)
    # n1 수락 → n2 시작 ε-전이
    merged.add_transition(n1.accept, 'ε', n2.start)
    return merged

def build_union(n1: NFA, n2: NFA) -> NFA:
    """선택 NFA: n1 또는 n2"""
    s, a = new_state(), new_state()
    merged = NFA(start=s, accept=a)
    for nfa in [n1, n2]:
        merged.add_transition(s, 'ε', nfa.start)
        merged.add_transition(nfa.accept, 'ε', a)
        for st, trans in nfa.transitions.items():
            for c, targets in trans.items():
                for t in targets:
                    merged.add_transition(st, c, t)
    return merged

def build_star(n: NFA) -> NFA:
    """Kleene Star NFA: n*"""
    s, a = new_state(), new_state()
    merged = NFA(start=s, accept=a)
    merged.add_transition(s, 'ε', n.start)   # s → n 시작
    merged.add_transition(s, 'ε', a)          # s → 수락 (0번 매칭)
    merged.add_transition(n.accept, 'ε', n.start)  # 루프
    merged.add_transition(n.accept, 'ε', a)        # n → 최종 수락
    for st, trans in n.transitions.items():
        for c, targets in trans.items():
            for t in targets:
                merged.add_transition(st, c, t)
    return merged


# 테스트: a(b|c)*d
a_nfa  = build_char('a')
b_nfa  = build_char('b')
c_nfa  = build_char('c')
d_nfa  = build_char('d')

b_or_c = build_union(b_nfa, c_nfa)
bc_star = build_star(b_or_c)
pattern = build_concat(a_nfa, build_concat(bc_star, d_nfa))

test_cases = ["ad", "abd", "acd", "abcd", "abcbcd", "abbd", "a", "bd", "adb"]
for s in test_cases:
    print(f"  '{s}' → {'매칭 O' if pattern.accepts(s) else '매칭 X'}")
```

실행 결과:
```
  'ad'     → 매칭 O
  'abd'    → 매칭 O
  'acd'    → 매칭 O
  'abcd'   → 매칭 O
  'abcbcd' → 매칭 O
  'abbd'   → 매칭 O
  'a'      → 매칭 X
  'bd'     → 매칭 X
  'adb'    → 매칭 X
```

---

## Powerset 구성법: NFA → DFA 변환

NFA의 "여러 상태를 동시에 있을 수 있다"는 특성을, **상태들의 집합 자체를 DFA의 단일 상태로** 만들어 DFA로 변환하는 방법이다.

```python
from collections import deque

@dataclass
class DFA:
    start: FrozenSet[int]
    accepting: set  # 수락 상태 집합들의 집합
    transitions: dict  # {상태집합: {char: 다음 상태집합}}
    alphabet: set

    def accepts(self, string: str) -> bool:
        current = self.start
        for char in string:
            current = self.transitions.get(current, {}).get(char)
            if current is None:
                return False
        return current in self.accepting


def nfa_to_dfa(nfa: NFA, alphabet: set) -> DFA:
    """Powerset 구성법으로 NFA → DFA 변환"""
    start = nfa.epsilon_closure(frozenset([nfa.start]))
    
    dfa_transitions = {}
    accepting = set()
    queue = deque([start])
    visited = {start}
    
    if nfa.accept in start:
        accepting.add(start)
    
    while queue:
        current_set = queue.popleft()
        dfa_transitions[current_set] = {}
        
        for char in alphabet:
            if char == 'ε':
                continue
            next_set = nfa.epsilon_closure(nfa.move(current_set, char))
            if next_set:  # 전이 가능한 상태가 있을 때만
                dfa_transitions[current_set][char] = next_set
                if next_set not in visited:
                    visited.add(next_set)
                    queue.append(next_set)
                    if nfa.accept in next_set:
                        accepting.add(next_set)
    
    return DFA(start=start, accepting=accepting,
               transitions=dfa_transitions, alphabet=alphabet)


# NFA를 DFA로 변환
# 먼저 NFA에서 사용된 알파벳 추출
alphabet = set()
for trans in pattern.transitions.values():
    alphabet.update(k for k in trans.keys() if k != 'ε')

dfa = nfa_to_dfa(pattern, alphabet)

print(f"DFA 상태 수: {len(dfa.transitions)}")
print()

# 동일한 테스트 수행
for s in ["ad", "abd", "acd", "abcbcd", "a", "bd"]:
    nfa_result = pattern.accepts(s)
    dfa_result = dfa.accepts(s)
    match = "✓" if nfa_result == dfa_result else "✗"
    print(f"  {match} '{s}': NFA={nfa_result}, DFA={dfa_result}")
```

---

## 실전 정규 표현식 파서 (후위 표기법 변환)

실제 엔진은 `a(b|c)*d` 같은 중위 표기법 정규식을 파싱해야 한다. 연결 연산자를 명시적(`·`)으로 삽입한 뒤 Shunting-Yard 알고리즘으로 후위 표기법으로 변환한다:

```python
def insert_concat(regex: str) -> str:
    """연결 연산자 '·' 명시적 삽입"""
    result = []
    operators = {'|', '*', '+', '?', '(', ')'}
    
    for i, c in enumerate(regex):
        result.append(c)
        if i + 1 < len(regex):
            curr, next_ = c, regex[i + 1]
            # 다음 문자에 연결 연산자가 필요한 경우
            if (curr not in {'(', '|'} and
                next_ not in {')', '|', '*', '+', '?'}):
                result.append('·')
    return ''.join(result)

def to_postfix(regex: str) -> str:
    """중위 표기법 → 후위 표기법 (Shunting-Yard)"""
    precedence = {'|': 1, '·': 2, '*': 3, '+': 3, '?': 3}
    output, stack = [], []
    
    for c in regex:
        if c == '(':
            stack.append(c)
        elif c == ')':
            while stack and stack[-1] != '(':
                output.append(stack.pop())
            stack.pop()
        elif c in precedence:
            while (stack and stack[-1] != '(' and
                   stack[-1] in precedence and
                   precedence.get(stack[-1], 0) >= precedence[c]):
                output.append(stack.pop())
            stack.append(c)
        else:
            output.append(c)
    
    while stack:
        output.append(stack.pop())
    return ''.join(output)

def regex_to_nfa(regex: str) -> NFA:
    """정규 표현식 문자열 → NFA"""
    postfix = to_postfix(insert_concat(regex))
    stack = []
    
    for c in postfix:
        if c == '·':
            n2, n1 = stack.pop(), stack.pop()
            stack.append(build_concat(n1, n2))
        elif c == '|':
            n2, n1 = stack.pop(), stack.pop()
            stack.append(build_union(n1, n2))
        elif c == '*':
            stack.append(build_star(stack.pop()))
        else:
            stack.append(build_char(c))
    
    return stack[0]

# 테스트
patterns = [
    ("a(b|c)*d", ["ad", "abd", "abcbcd", "bd", "adb"]),
    ("(ab)+",    ["ab", "abab", "ababab", "a", "b", "abc"]),
    ("a*b",      ["b", "ab", "aaab", "a", "c"]),
]

for regex_str, tests in patterns:
    nfa = regex_to_nfa(regex_str)
    print(f"패턴: {regex_str}")
    for s in tests:
        print(f"  '{s}' → {'매칭 O' if nfa.accepts(s) else '매칭 X'}")
    print()
```

---

## 주의사항 및 팁

### 1. 백트래킹 기반 엔진의 함정

Perl, Python(`re` 모듈), Java의 `java.util.regex`는 Thompson NFA 방식이 아닌 **백트래킹(backtracking)** 방식을 사용한다. 이 방식은 `(a+)+b`처럼 중첩 수량자를 가진 패턴에서 **지수 시간 복잡도**가 발생한다(ReDoS, Regular Expression Denial of Service 취약점).

```python
import re, time

# ReDoS 취약 패턴
pattern = re.compile(r'(a+)+b')
test_input = 'a' * 25  # 'b' 없는 25개의 'a'

start = time.time()
try:
    result = pattern.match(test_input)
except:
    pass
elapsed = time.time() - start
print(f"백트래킹 소요 시간: {elapsed:.3f}초")  # 수초 이상 걸릴 수 있음

# 해결: Python의 re2 바인딩이나 Thompson NFA 기반 라이브러리 사용
# pip install google-re2
```

### 2. ε-클로저 캐싱

NFA 시뮬레이션에서 ε-클로저 계산은 반복 호출된다. 결과를 캐싱하면 성능이 크게 향상된다.

### 3. DFA 최소화 (Hopcroft 알고리즘)

Powerset 구성법으로 생성된 DFA는 동치인 상태를 포함할 수 있다. Hopcroft 알고리즘으로 DFA를 최소화하면 O(n log n) 시간에 최소 상태 DFA를 얻을 수 있다.

### 4. 실제 엔진의 선택

| 엔진 | 방식 | 특징 |
|------|------|------|
| Rust `regex` crate | Thompson NFA + lazy DFA | 선형 시간 보장, ReDoS 없음 |
| Python `re` | 백트래킹 NFA | PCRE 호환, 복잡한 캡처 그룹 지원 |
| Python `re2` | DFA | 선형 시간, 일부 PCRE 문법 미지원 |
| grep, awk | DFA | 빠름, 캡처 그룹 제한 |
| PCRE | 백트래킹 + 최적화 | 풍부한 기능, ReDoS 주의 |

---

## 정리

정규 표현식 엔진의 핵심은 **Thompson 구성법(정규식 → NFA)**과 **Powerset 구성법(NFA → DFA)**이다. NFA는 구성이 쉽고 상태 수가 작지만 실행 시 비결정성을 처리해야 하고, DFA는 실행이 O(m)으로 빠르지만 상태 폭발이 발생할 수 있다. 현대의 우수한 정규 표현식 엔진은 두 방식을 혼합하여 lazy DFA(실행 중 필요한 DFA 상태만 동적으로 계산)를 사용해 두 장점을 결합한다.

## 참고 자료
- [Thompson's construction - Wikipedia](https://en.wikipedia.org/wiki/Thompson%27s_construction)
- [Powerset construction - Wikipedia](https://en.wikipedia.org/wiki/Powerset_construction)
- [Regular expression - Wikipedia](https://en.wikipedia.org/wiki/Regular_expression)
- [Rust regex crate documentation](https://docs.rs/regex/latest/regex/)
