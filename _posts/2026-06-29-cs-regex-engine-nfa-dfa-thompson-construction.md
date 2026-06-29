---
layout: post
title: "정규표현식 엔진 내부 구조 완전 정복: NFA, DFA, Thompson's Construction이 패턴을 매칭하는 방법"
date: 2026-06-29
categories: [cs, computer-science]
tags: [regex, nfa, dfa, thompson-construction, automata, finite-state-machine, backtracking, redos]
---

## 개념 설명: 정규표현식 엔진이란 무엇인가

정규표현식(Regular Expression)은 거의 모든 프로그래밍 언어에서 기본으로 제공하는 강력한 패턴 매칭 도구다. 하지만 `re.match()` 한 줄 뒤에서 실제로 무슨 일이 일어나는지 아는 개발자는 많지 않다. 정규표현식 엔진은 크게 두 가지 방식으로 동작한다: **NFA(비결정적 유한 오토마톤)** 기반과 **DFA(결정적 유한 오토마톤)** 기반이다.

### 유한 오토마톤의 두 가지 종류

**DFA(Deterministic Finite Automaton)**는 현재 상태와 입력 문자에 의해 다음 상태가 유일하게 결정되는 오토마톤이다. 어떤 상태에서 어떤 문자를 읽든 전이할 수 있는 상태가 정확히 하나다. 구현이 빠르지만, 정규표현식으로부터 직접 DFA를 만들면 상태 수가 지수적으로 폭발할 수 있다.

**NFA(Nondeterministic Finite Automaton)**는 하나의 상태에서 동일한 입력에 대해 여러 전이가 가능하거나, 입력 없이 전이하는 **ε-전이(epsilon transition)**가 존재할 수 있다. NFA는 "올바른 경로를 추측"한다는 개념으로 이해할 수 있다. 어느 경로든 하나라도 수락 상태에 도달하면 매칭 성공이다.

이론적으로 NFA와 DFA는 동일한 언어(정규 언어)를 인식한다. 모든 NFA는 동치인 DFA로 변환할 수 있다(부분 집합 구성법). 하지만 DFA로 변환하면 상태 수가 NFA 상태 수 n에 대해 최대 2^n까지 증가할 수 있다. n=30인 정규표현식이 10억 개 이상의 DFA 상태를 요구할 수 있는 것이다.

### Thompson's Construction: 정규표현식을 NFA로

Ken Thompson이 1968년 CACM 논문에서 발표하고 Russ Cox가 현대적으로 재조명한 **Thompson's Construction**은 정규표현식을 O(n) 크기의 NFA로 변환하는 체계적인 알고리즘이다. 핵심 아이디어는 **연산자별로 부분 NFA를 만들고 합성**하는 것이다.

**기본 문자 `a`**: 시작 상태 s에서 'a'를 읽으면 수락 상태 t로 전이하는 2-상태 NFA.

**연접(concatenation) `ab`**: NFA_a의 수락 상태와 NFA_b의 시작 상태를 ε-전이로 연결. 총 상태 수는 두 NFA의 합.

**선택(alternation) `a|b`**: 새 시작 상태에서 NFA_a와 NFA_b 각각으로 ε-전이. 두 NFA의 수락 상태에서 새 수락 상태로 ε-전이.

**반복(Kleene star) `a*`**: 새 시작/수락 상태를 추가하고 ε-전이 4개를 연결. NFA_a 수락→NFA_a 시작의 역방향 ε-전이가 반복을 구현한다.

이렇게 만든 NFA는 O(n) 상태와 O(n) 전이를 가지며(n = 정규표현식 길이), NFA를 **동시 상태 집합 시뮬레이션**으로 실행하면 O(n × m) 시간 복잡도를 보장한다(m = 입력 문자열 길이).

---

## 왜 NFA 기반 엔진이 필요한가

### 백트래킹 엔진의 지뢰: 재앙적 역추적 (Catastrophic Backtracking)

Python의 `re`, Perl, Java의 `java.util.regex` 등 대부분의 범용 정규표현식 라이브러리는 **백트래킹(backtracking) NFA** 기반이다. 이 방식은 역참조(backreference), 룩어헤드(lookahead) 같은 강력한 확장 기능을 지원하지만, 특정 패턴에서 치명적인 약점이 드러난다.

`(a+)+` 패턴에 `"aaaaab"` 같이 매칭이 실패하는 문자열을 적용하면 어떻게 될까? 엔진은 가능한 모든 그룹화를 시도한다: `(aaaa)b`, `(aaa)(a)b`, `(aa)(aa)b`, `(aa)(a)(a)b`, `(a)(aaa)b`, ... 이 경우 시도 횟수가 입력 길이 n에 대해 **지수적(2^n)**으로 증가한다. 이를 **재앙적 역추적(Catastrophic Backtracking)** 또는 **ReDoS(Regular Expression Denial of Service)**라고 한다.

실제로 2019년 Cloudflare의 글로벌 장애도 단 하나의 잘못된 정규표현식이 트리거가 되었다. ReDoS는 단순한 이론적 위협이 아니다.

**Thompson's NFA 기반 엔진**은 이 문제가 근본적으로 없다. 단일 상태가 아니라 **가능한 모든 상태 집합**을 병렬로 추적하기 때문에 백트래킹 자체가 발생하지 않는다. 어떤 패턴이든 입력 문자열 길이에 대해 선형 시간(O(n×m))을 보장한다.

---

## 실제 구현 예제

### 예제 1: Python으로 구현하는 Thompson NFA 시뮬레이터

Thompson NFA의 핵심인 동시 상태 추적을 직접 구현한다. `a*b` 패턴을 수동으로 NFA로 모델링하고 시뮬레이션한다.

```python
from collections import defaultdict
from typing import Set

class NFA:
    """Thompson NFA 시뮬레이터 (ε-전이 포함, 병렬 상태 추적)"""

    def __init__(self, num_states: int, start: int, accept: Set[int]):
        self.num_states = num_states
        self.start = start
        self.accept = accept
        self.transitions: dict = defaultdict(lambda: defaultdict(set))
        self.epsilon: dict = defaultdict(set)

    def add_transition(self, src: int, char: str, dst: int):
        self.transitions[src][char].add(dst)

    def add_epsilon(self, src: int, dst: int):
        self.epsilon[src].add(dst)

    def epsilon_closure(self, states: Set[int]) -> Set[int]:
        """ε-전이로 도달 가능한 모든 상태 집합 반환 (BFS)"""
        closure = set(states)
        stack = list(states)
        while stack:
            state = stack.pop()
            for next_state in self.epsilon[state]:
                if next_state not in closure:
                    closure.add(next_state)
                    stack.append(next_state)
        return closure

    def match(self, text: str) -> bool:
        """NFA 시뮬레이션: 여러 상태를 병렬 추적 (선형 시간 보장)"""
        current_states = self.epsilon_closure({self.start})
        for char in text:
            next_states = set()
            for state in current_states:
                for dst in self.transitions[state].get(char, set()):
                    next_states.add(dst)
            current_states = self.epsilon_closure(next_states)
        return bool(current_states & self.accept)


def build_a_star_b_nfa() -> NFA:
    """
    패턴 a*b를 Thompson's Construction으로 수동 빌드.

    States:
      0 -ε-> 1  (a* 진입: 적어도 1개 a 시도)
      0 -ε-> 3  (a* 스킵: 0개 a)
      1 -a-> 2
      2 -ε-> 1  (a 반복)
      2 -ε-> 3  (a 종료 후 b로)
      3 -b-> 4  (수락)
    """
    nfa = NFA(num_states=5, start=0, accept={4})
    nfa.add_epsilon(0, 1)
    nfa.add_epsilon(0, 3)
    nfa.add_transition(1, 'a', 2)
    nfa.add_epsilon(2, 1)
    nfa.add_epsilon(2, 3)
    nfa.add_transition(3, 'b', 4)
    return nfa


nfa = build_a_star_b_nfa()
test_cases = [
    ("b",    True),
    ("ab",   True),
    ("aaab", True),
    ("a",    False),
    ("ba",   False),
    ("",     False),
]
for text, expected in test_cases:
    result = nfa.match(text)
    status = "✓" if result == expected else "✗"
    print(f"{status} match('{text}') = {result} (expected {expected})")
```

### 예제 2: 재앙적 역추적 시연 및 ReDoS 취약성 검사기

```python
import re
import time

def backtracking_demo():
    """
    (a+)+ 패턴의 백트래킹 지수 폭발과 안전한 대안 비교.
    실제 환경에서 n=25 이상이면 위험 패턴은 수십 초,
    안전 패턴은 마이크로초 수준이다.
    """
    dangerous = re.compile(r'^(a+)+$')
    safe      = re.compile(r'^a+$')

    print(f"{'n':>4}  {'위험한 패턴 (ms)':>18}  {'안전한 패턴 (ms)':>18}")
    print("-" * 50)

    for n in [5, 10, 15, 20]:
        text = 'a' * n + 'b'   # 마지막 'b'로 매칭 실패 강제

        start = time.perf_counter()
        dangerous.match(text)
        t_danger = (time.perf_counter() - start) * 1000

        start = time.perf_counter()
        safe.match(text)
        t_safe = (time.perf_counter() - start) * 1000

        print(f"{n:>4}  {t_danger:>18.4f}  {t_safe:>18.6f}")


def check_redos_risk(pattern: str) -> str:
    """중첩 수량자 기반 간단 ReDoS 휴리스틱 검사."""
    dangerous_shapes = [
        r'\([^)]*[+*][^)]*\)[+*]',           # (X+)+ 형태
        r'\([^)]*[+*][^)]*\)\{[0-9]+,',      # (X+){n, 형태
        r'\([^)]*\|[^)]*\)[+*]',             # (a|ab)+ 처럼 중의적 NFA
    ]
    for shape in dangerous_shapes:
        if re.search(shape, pattern):
            return "⚠  취약 가능성"
    return "✓  안전"


backtracking_demo()

print("\n=== ReDoS 취약성 검사 ===")
patterns = [
    r'^(a+)+$',
    r'^(a|aa)+$',
    r'^a+$',
    r'^[a-z]{1,100}$',
    r'(\w+\s?){20}',
]
for p in patterns:
    print(f"{check_redos_risk(p)}: {p}")
```

---

## 주의사항과 실전 팁

### 1. ReDoS 방어 전략

사용자 입력을 정규표현식에 포함시킬 때는 반드시 `re.escape()`로 이스케이핑한다. 서버 측에서 동적 패턴을 평가한다면 `google/re2` (Python 바인딩: `pip install google-re2`)나 Rust의 `regex` 크레이트처럼 선형 시간을 보장하는 라이브러리로 교체하는 것이 근본 해결책이다.

### 2. 엔진별 특성 비교

| 엔진 | 방식 | 역참조 | 시간 복잡도 |
|------|------|--------|------------|
| Python `re`, PCRE | 백트래킹 NFA | 지원 | 최악 지수 |
| RE2 (Google) | Thompson NFA + DFA | 미지원 | O(n×m) 보장 |
| Hyperscan (Intel) | SIMD DFA | 미지원 | 고성능 멀티패턴 |
| Rust `regex` | Thompson NFA + lazy DFA | 미지원 | O(n×m) 보장 |

역참조(`\1`, `\2`)는 정규 언어 이론의 범위를 벗어나므로 NFA/DFA 기반 순수 엔진으로는 지원 불가능하다. 역참조가 필요 없다면 RE2를 우선 검토하라.

### 3. 패턴 컴파일 캐시 활용

`re.compile()`은 패턴을 NFA/DFA로 컴파일하는 비용이 있다. CPython은 최근 512개의 컴파일 결과를 캐싱하지만, 동적으로 생성되는 패턴이 수천 가지라면 캐시를 오염시킨다. 루프 밖에서 미리 컴파일하여 재사용하는 습관을 들이자.

### 4. `re.fullmatch` vs `re.match` vs `re.search`

`re.match`는 문자열 **시작**부터 패턴을 시도하고, `re.search`는 **어디서든** 시도하며, `re.fullmatch`는 **전체 문자열**이 패턴에 맞아야 한다. 입력 검증 시 `re.fullmatch`가 가장 안전하다. `re.match(r'\d+', '123abc')`는 성공하지만 `re.fullmatch(r'\d+', '123abc')`는 실패한다.

---

## 참고 자료
- [Regular Expression Matching Can Be Simple And Fast — Russ Cox](https://swtch.com/~rsc/regexp/regexp1.html)
- [Thompson's construction — Wikipedia](https://en.wikipedia.org/wiki/Thompson%27s_construction)
- [Nondeterministic finite automaton — Wikipedia](https://en.wikipedia.org/wiki/Nondeterministic_finite_automaton)
- [Regular expression — Wikipedia](https://en.wikipedia.org/wiki/Regex)
