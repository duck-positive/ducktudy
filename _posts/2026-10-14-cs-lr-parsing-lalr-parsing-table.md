---
layout: post
title: "LR 파싱 완전 정복: LALR(1) 파싱 테이블 자동 구성과 충돌 해결"
date: 2026-10-14
categories: [cs, computer-science]
tags: [compiler, parsing, LALR, LR-parser, automata, syntax-analysis]
---

## 1. 개념 설명 — LR 파싱이란 무엇인가

LR 파싱은 **L**eft-to-right scan, **R**ightmost derivation의 약자로, 입력 토큰 스트림을 왼쪽에서 오른쪽으로 읽으면서 **오른쪽 유도의 역순**으로 파스 트리를 구성하는 **상향식(bottom-up)** 파싱 기법이다.

컴파일러 교재의 고전 *Dragon Book(Aho et al.)*에서 체계화된 이 방식은 오늘날 GCC, Clang, Python CPython, Ruby MRI 등 거의 모든 산업용 컴파일러/인터프리터의 프런트엔드에 적용된다.

### LR 파서의 종류

| 종류 | 설명 | 파싱 능력 | 상태 수 |
|------|------|-----------|---------|
| SLR(1) | Simple LR | 약함 | 적음 |
| LALR(1) | Lookahead LR | 보통 (실용적) | 적음 |
| LR(1) | Canonical LR | 가장 강함 | 많음 |

LALR(1)은 LR(1)의 상태 수를 SLR(1) 수준으로 줄이면서 대부분의 프로그래밍 언어 문법을 처리할 수 있어 **산업 표준**으로 자리 잡았다. Yacc/Bison이 생성하는 테이블이 바로 LALR(1)이다.

### 핵심 자료구조: 파싱 스택 + 상태

LR 파서는 두 가지 테이블로 동작한다:

- **ACTION 테이블**: `(상태, 터미널)` → `shift s / reduce A→α / accept / error`
- **GOTO 테이블**: `(상태, 비터미널)` → `상태`

파서는 스택에 `(심볼, 상태)` 쌍을 쌓고, lookahead 토큰을 보고 다음 동작을 결정한다.

---

## 2. 왜 필요한가 — LL 파싱의 한계와 LR의 장점

LL(1) 파서는 구현이 단순하지만 **좌재귀(left recursion)**를 직접 처리하지 못하고, 문법을 변형해야 한다는 한계가 있다. 예를 들어 산술 표현식을 자연스럽게 표현하는 문법:

```
E → E + T | T
T → T * F | F
F → ( E ) | id
```

이 문법은 `E → E + T`처럼 좌재귀가 있어 LL(1)로 파싱할 수 없다. LR 파서는 이 문법을 **변환 없이** 처리한다.

또한 LL(k)가 처리하는 문법은 LR(k)가 처리하는 문법의 **진부분집합**이다. 즉, LR 파싱은 표현력이 더 크다.

---

## 3. LALR(1) 파싱 테이블 구성 — 핵심 알고리즘

### 3.1 LR(1) 아이템과 클로저

LR(1) **아이템**은 `[A → α · β, a]` 형태로, `·`(dot) 위치가 현재 파싱 진행 상황을 나타내고, `a`는 lookahead 터미널이다.

**클로저(Closure)** 연산:

```
Closure(I):
  repeat
    for each [A → α · B β, a] ∈ I:
      for each production B → γ:
        for each b ∈ FIRST(β a):
          I ← I ∪ {[B → · γ, b]}
  until I unchanged
  return I
```

**Goto** 연산:

```
Goto(I, X) = Closure({[A → αX · β, a] | [A → α · X β, a] ∈ I})
```

### 3.2 파이썬으로 LR(1) 아이템 집합 구성하기

아래는 단순 산술 문법에 대해 LR(1) 아이템 집합(Canonical Collection)을 구성하는 핵심 코드다:

```python
from collections import defaultdict
from itertools import chain

# 문법 정의
# S' -> S
# S  -> E
# E  -> E + T | T
# T  -> T * F | F
# F  -> ( E ) | id

GRAMMAR = {
    "S'": [["S"]],
    "S":  [["E"]],
    "E":  [["E", "+", "T"], ["T"]],
    "T":  [["T", "*", "F"], ["F"]],
    "F":  [["(", "E", ")"], ["id"]],
}
TERMINALS   = {"+", "*", "(", ")", "id", "$"}
START       = "S'"

def first_of(seq, terminals, grammar, memo=None):
    """FIRST(α a) 를 계산한다."""
    if memo is None:
        memo = {}
    result = set()
    for sym in seq:
        if sym in terminals:
            result.add(sym)
            break
        # 비터미널 → FIRST 재귀
        key = sym
        if key not in memo:
            memo[key] = set()
            for prod in grammar.get(sym, []):
                memo[key] |= first_of(prod, terminals, grammar, memo)
        result |= (memo[key] - {"ε"})
        if "ε" not in memo.get(key, set()):
            break
    else:
        result.add("ε")
    return result

def closure(items, grammar, terminals):
    """LR(1) 클로저 계산."""
    result = set(items)
    changed = True
    while changed:
        changed = False
        new = set()
        for (lhs, prod, dot, la) in result:
            if dot < len(prod):
                B = prod[dot]
                if B not in terminals:
                    beta_a = list(prod[dot+1:]) + [la]
                    for b in first_of(beta_a, terminals, grammar):
                        if b == "ε":
                            continue
                        for rhs in grammar.get(B, []):
                            item = (B, tuple(rhs), 0, b)
                            if item not in result:
                                new.add(item)
        if new:
            result |= new
            changed = True
    return frozenset(result)

def goto(items, symbol, grammar, terminals):
    """LR(1) Goto 연산."""
    moved = set()
    for (lhs, prod, dot, la) in items:
        if dot < len(prod) and prod[dot] == symbol:
            moved.add((lhs, prod, dot+1, la))
    return closure(moved, grammar, terminals) if moved else frozenset()

def build_canonical_collection(grammar, terminals, start):
    """전체 LR(1) 아이템 집합 컬렉션 구성."""
    init_prod = grammar[start][0]
    init_item = (start, tuple(init_prod), 0, "$")
    I0 = closure({init_item}, grammar, terminals)
    
    collection = [I0]
    state_map  = {I0: 0}
    transitions = {}
    
    worklist = [I0]
    while worklist:
        I = worklist.pop()
        # 모든 심볼 수집
        symbols = set()
        for (_, prod, dot, _) in I:
            if dot < len(prod):
                symbols.add(prod[dot])
        for X in symbols:
            J = goto(I, X, grammar, terminals)
            if not J:
                continue
            if J not in state_map:
                state_map[J] = len(collection)
                collection.append(J)
                worklist.append(J)
            transitions[(state_map[I], X)] = state_map[J]
    
    return collection, state_map, transitions

collection, state_map, transitions = build_canonical_collection(
    GRAMMAR, TERMINALS, START
)
print(f"LR(1) 상태 수: {len(collection)}")  # 출력: 36 내외

# LALR 병합: 동일 core(lhs, prod, dot)를 가진 상태를 합침
def merge_lalr(collection):
    core_map = defaultdict(list)
    for i, state in enumerate(collection):
        core = frozenset((lhs, prod, dot) for (lhs, prod, dot, _) in state)
        core_map[core].append(i)
    return core_map

merged = merge_lalr(collection)
lalr_states = sum(1 for states in merged.values() if len(states) > 1)
print(f"병합 후 LALR(1) 상태 수: {len(collection) - lalr_states}")
```

### 3.3 ACTION/GOTO 테이블 충돌 처리

LALR 병합 과정에서 **Reduce-Reduce 충돌**이 발생할 수 있다. 아래 파이썬 코드는 충돌을 감지하는 기초 로직이다:

```python
def build_action_table(collection, state_map, transitions, grammar, terminals):
    action = {}
    conflicts = []

    for state in collection:
        s = state_map[state]
        for (lhs, prod, dot, la) in state:
            if dot < len(prod):
                # Shift
                sym = prod[dot]
                if sym in terminals:
                    next_s = transitions.get((s, sym))
                    if next_s is not None:
                        entry = ("shift", next_s)
                        key = (s, sym)
                        if key in action and action[key] != entry:
                            conflicts.append(("shift-reduce", s, sym, action[key], entry))
                        else:
                            action[key] = entry
            else:
                # Reduce 또는 Accept
                if lhs == "S'" and prod == ("S",) and la == "$":
                    action[(s, "$")] = ("accept",)
                else:
                    entry = ("reduce", lhs, prod)
                    key = (s, la)
                    if key in action and action[key] != entry:
                        conflicts.append(("reduce-reduce", s, la, action[key], entry))
                    else:
                        action[key] = entry

    return action, conflicts

action_table, conflicts = build_action_table(
    collection, state_map, transitions, GRAMMAR, TERMINALS
)
if conflicts:
    print("충돌 발생!")
    for c in conflicts:
        print(c)
else:
    print("충돌 없음 — 유효한 LALR(1) 문법입니다.")
```

---

## 4. LR 파서 런타임 — 스택 기반 시뮬레이션

아래는 미리 구성된 ACTION/GOTO 테이블을 사용해 실제 파싱을 수행하는 드라이버다:

```python
def lr_parse(tokens, action, goto_table, state_map):
    """
    tokens: [("id",None), ("+",None), ("id",None), ("$",None)]
    """
    stack  = [0]        # 상태 스택
    sym_stack = []      # 심볼 스택 (AST 노드)
    pos    = 0

    while True:
        s   = stack[-1]
        tok = tokens[pos][0]  # 현재 lookahead

        act = action.get((s, tok))
        if act is None:
            raise SyntaxError(f"파싱 오류: 상태={s}, 토큰={tok}")

        if act[0] == "shift":
            _, next_s = act
            sym_stack.append(tok)
            stack.append(next_s)
            pos += 1

        elif act[0] == "reduce":
            _, lhs, prod = act
            n = len(prod)
            children = sym_stack[-n:] if n else []
            sym_stack = sym_stack[:-n] if n else sym_stack
            stack     = stack[:-n]     if n else stack
            # 팝 후 GOTO
            top_s = stack[-1]
            next_s = goto_table.get((top_s, lhs))
            if next_s is None:
                raise SyntaxError(f"GOTO 오류: 상태={top_s}, 비터미널={lhs}")
            sym_stack.append((lhs, children))
            stack.append(next_s)

        elif act[0] == "accept":
            print("파싱 성공!")
            return sym_stack[-1]

# 사용 예시
tokens = [("id",None), ("+",None), ("id",None), ("*",None), ("id",None), ("$",None)]
# 위의 action_table, transitions를 goto_table로 변환 후 사용
```

---

## 5. 주의사항과 팁

### Shift-Reduce 충돌 해결 — 연산자 우선순위

`if-then-else` dangling-else 문제나 `+`, `*` 혼재 시 shift-reduce 충돌이 발생한다. Bison은 `%left`, `%right`, `%prec` 지시자로 이를 선언적으로 해결한다:

```yacc
%left  '+' '-'
%left  '*' '/'
%right UMINUS

expr: expr '+' expr  { $$ = $1 + $3; }
    | expr '*' expr  { $$ = $1 * $3; }
    | '-' expr %prec UMINUS { $$ = -$2; }
    ;
```

### LALR vs. GLR

LALR이 충돌을 해결할 수 없는 경우(예: C++의 `T * p` 문제, ISO-C 방언) **GLR(Generalized LR)** 파서가 대안이다. GCC 4.x 이후 C++ 프런트엔드는 GLR 파서를 사용한다.

### 도구 선택 가이드

| 도구 | 언어 | 방식 |
|------|------|------|
| Bison/Yacc | C/C++ | LALR(1) |
| ANTLR 4 | 다중 | LL(*) Adaptive |
| Tree-sitter | 다중 | GLR(incremental) |
| pest (Rust) | Rust | PEG |

실용 팁: 새 언어 설계 시 문법을 초기에 Bison으로 검증하면 LALR 충돌을 조기에 발견할 수 있다. 충돌이 너무 많으면 ANTLR의 LL(*)이나 PEG를 고려하라.

---

## 참고 자료

- [LR 파서 — Wikipedia](https://en.wikipedia.org/wiki/LR_parser)
- [GNU Bison 공식 문서](https://www.gnu.org/software/bison/manual/)
- [Dragon Book: Compilers — Principles, Techniques, and Tools (Aho et al.)](https://suif.stanford.edu/dragonbook/)
- [LALR Parsing — Stanford CS143 강의 노트](https://web.stanford.edu/class/archive/cs/cs143/cs143.1128/handouts/140%20LALR%20Parsing.pdf)
