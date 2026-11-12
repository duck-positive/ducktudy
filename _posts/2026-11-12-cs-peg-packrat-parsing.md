---
layout: post
title: "PEG와 패크랫 파서 완전 정복: 문맥 자유 문법을 대체하는 결정적 선형 시간 파싱"
date: 2026-11-12
categories: [cs, computer-science]
tags: [parsing, PEG, packrat, grammar, compiler, parser-combinator, memoization]
---

## 문맥 자유 문법(CFG)의 한계

대부분의 프로그래밍 언어 파서는 **문맥 자유 문법(Context-Free Grammar, CFG)**을 기반으로 한다. BNF로 표현된 언어 명세를 Yacc/Bison 같은 도구로 변환하면 LALR(1) 파서가 생성된다. 그러나 CFG와 그 파서 생성기에는 오랫동안 개발자를 괴롭혀 온 문제들이 있다.

### CFG의 고질적 문제

1. **모호성(Ambiguity)**: CFG는 같은 입력에 대해 두 개 이상의 파스 트리가 존재할 수 있다. `1 + 2 * 3`을 파싱할 때 덧셈 먼저냐 곱셈 먼저냐는 문법 규칙으로 해결해야 하지만, 모호한 문법은 파서 생성기가 거부한다.

2. **교체(alternation)의 순서 무관성**: CFG의 `A | B`는 집합적(nondeterministic)이다. 순서가 의미를 갖지 않으므로, 선택지 간 우선순위를 표현할 수 없다.

3. **쉬프트-리듀스 충돌**: LALR(1) 파서는 1 토큰 미리보기만 사용하므로, 복잡한 언어는 "충돌"이 발생하여 개발자가 직접 문법을 조정해야 한다.

4. **공백·주석 처리**: 렉서(lexer)와 파서를 분리해야 해서 "렉서 해킹(lexer hack)"이 필요한 상황이 생긴다. 대표적 사례가 C의 typedef 구문이다.

5. **재귀 하강의 비효율**: 직접 재귀 하강(recursive descent) 파서는 직관적이지만 역추적(backtracking)이 발생하면 지수 시간이 걸릴 수 있다.

## PEG: 인식 기반 구문 형식

Bryan Ford는 2004년 POPL 논문 "Parsing Expression Grammars: A Recognition-Based Syntactic Foundation"에서 **PEG(Parsing Expression Grammar)**를 제안했다. PEG는 CFG와 비슷한 외양을 갖지만, 근본적으로 다른 의미론을 가진다.

### PEG의 핵심 연산자

PEG 표현식(parsing expression) `e`는 다음과 같이 정의된다.

| 표현식 | 의미 |
|--------|------|
| `""` | 빈 문자열 (항상 성공) |
| `"c"` | 리터럴 문자 c |
| `[a-z]` | 문자 클래스 |
| `e1 e2` | 순서 접합(sequence): e1 성공 후 e2 |
| `e1 / e2` | **순서 선택(ordered choice)**: e1 먼저 시도, 실패하면 e2 |
| `e*` | 0회 이상 반복 (greedy) |
| `e+` | 1회 이상 반복 (greedy) |
| `e?` | 0 또는 1회 |
| `&e` | 긍정 예측(lookahead): e가 성공하면 입력 소비 없이 성공 |
| `!e` | 부정 예측(negative lookahead): e가 실패해야 성공 |

### CFG의 `|` vs PEG의 `/`

가장 중요한 차이는 선택 연산자다.

- **CFG `A | B`**: "A 또는 B" (비결정적, 모호성 허용)
- **PEG `A / B`**: "A를 먼저 시도. A가 성공하면 B는 무시. A가 실패하면 입력을 되돌리고 B를 시도."

이 **순서 선택(ordered choice)**이 PEG를 항상 결정적으로 만드는 핵심이다. PEG로 작성된 문법은 **절대 모호하지 않다.**

### PEG 문법 예시: 사칙연산

```
# PEG 문법: 사칙연산 표현식 (우선순위가 문법 구조에 내장됨)
Expr   ← Sum
Sum    ← Product (("+" / "-") Product)*
Product ← Unary (("*" / "/") Unary)*
Unary  ← "-" Unary / Primary
Primary ← "(" Expr ")" / Number
Number ← [0-9]+

# CFG에서는 우선순위를 별도로 명시해야 하지만,
# PEG에서는 계층적 구조가 자연스럽게 우선순위를 정의함
```

## 패크랫 파싱: 선형 시간 보장

PEG는 역추적(backtracking)을 사용하므로 최악의 경우 지수 시간이 걸릴 수 있다. Ford의 천재적인 해결책이 **패크랫 파싱(Packrat Parsing)**이다.

### 핵심 아이디어: 메모이제이션

"파서에 메모이제이션을 적용하면 어떨까?" 각 파싱 함수를 `(rule, position)` 쌍을 키로 하는 메모 테이블에 결과를 저장한다. 같은 위치에서 같은 규칙을 두 번 이상 시도하면 저장된 결과를 즉시 반환한다.

**정리**: 입력 길이 n, 규칙 수 k인 PEG 문법의 패크랫 파서는 **O(nk) = O(n) 시간**에 파싱을 완료한다. 역추적이 발생해도 이전 결과가 메모에 있으므로 재계산하지 않는다.

```
메모 테이블: Map<(rule_id, position), Result<(AST, next_pos), Failure>>
```

## 실제 구현 예제

### 예제 1: Python으로 구현하는 패크랫 파서

사칙연산을 처리하는 완전한 패크랫 파서를 Python으로 구현해보자.

```python
from __future__ import annotations
from dataclasses import dataclass, field
from typing import Optional, Callable, Any

# ─── 결과 타입 ───────────────────────────────────────────────────────
@dataclass
class Success:
    value: Any
    pos: int   # 소비된 후의 위치

FAIL = None  # None = 파싱 실패

Result = Optional[Success]

# ─── 패크랫 파서 ─────────────────────────────────────────────────────
class PackratParser:
    def __init__(self, text: str):
        self.text = text
        self.memo: dict[tuple[str, int], Result] = {}

    def _memoize(self, rule_name: str, pos: int,
                 fn: Callable[[int], Result]) -> Result:
        """메모이제이션 래퍼"""
        key = (rule_name, pos)
        if key in self.memo:
            return self.memo[key]
        result = fn(pos)
        self.memo[key] = result
        return result

    # ─── 기본 파서 조합자 ────────────────────────────────────────────
    def literal(self, s: str, pos: int) -> Result:
        if self.text.startswith(s, pos):
            return Success(s, pos + len(s))
        return FAIL

    def char_range(self, lo: str, hi: str, pos: int) -> Result:
        if pos < len(self.text) and lo <= self.text[pos] <= hi:
            return Success(self.text[pos], pos + 1)
        return FAIL

    def seq(self, *parsers, pos: int) -> Result:
        """순서 접합: e1 e2 e3 ..."""
        values = []
        cur = pos
        for p in parsers:
            r = p(cur)
            if r is FAIL:
                return FAIL
            values.append(r.value)
            cur = r.pos
        return Success(values, cur)

    def choice(self, *parsers, pos: int) -> Result:
        """순서 선택 (ordered choice): e1 / e2 / e3"""
        for p in parsers:
            r = p(pos)  # 실패하면 같은 pos에서 다음 시도
            if r is not FAIL:
                return r
        return FAIL

    def star(self, parser: Callable, pos: int) -> Result:
        """0회 이상 반복: e*"""
        values = []
        cur = pos
        while True:
            r = parser(cur)
            if r is FAIL:
                break
            values.append(r.value)
            cur = r.pos
        return Success(values, cur)

    # ─── 사칙연산 PEG 문법 구현 ──────────────────────────────────────
    # Expr ← Sum
    # Sum ← Product (("+" / "-") Product)*
    # Product ← Unary (("*" / "/") Unary)*
    # Unary ← "-" Unary / Primary
    # Primary ← "(" Expr ")" / Number
    # Number ← [0-9]+

    def skip_ws(self, pos: int) -> int:
        while pos < len(self.text) and self.text[pos] in ' \t':
            pos += 1
        return pos

    def parse_number(self, pos: int) -> Result:
        def _parse(p: int) -> Result:
            p = self.skip_ws(p)
            start = p
            while p < len(self.text) and self.text[p].isdigit():
                p += 1
            if p == start:
                return FAIL
            return Success(int(self.text[start:p]), p)
        return self._memoize("number", pos, _parse)

    def parse_primary(self, pos: int) -> Result:
        def _parse(p: int) -> Result:
            p = self.skip_ws(p)
            # "(" Expr ")"
            if p < len(self.text) and self.text[p] == '(':
                r = self.parse_expr(p + 1)
                if r is not FAIL:
                    q = self.skip_ws(r.pos)
                    if q < len(self.text) and self.text[q] == ')':
                        return Success(r.value, q + 1)
            # Number
            return self.parse_number(p)
        return self._memoize("primary", pos, _parse)

    def parse_unary(self, pos: int) -> Result:
        def _parse(p: int) -> Result:
            p = self.skip_ws(p)
            if p < len(self.text) and self.text[p] == '-':
                r = self.parse_unary(p + 1)
                if r is not FAIL:
                    return Success(-r.value, r.pos)
            return self.parse_primary(p)
        return self._memoize("unary", pos, _parse)

    def parse_product(self, pos: int) -> Result:
        def _parse(p: int) -> Result:
            left = self.parse_unary(p)
            if left is FAIL:
                return FAIL
            val, cur = left.value, left.pos
            while True:
                cur = self.skip_ws(cur)
                if cur < len(self.text) and self.text[cur] in '*/':
                    op = self.text[cur]
                    r = self.parse_unary(cur + 1)
                    if r is FAIL:
                        break
                    val = val * r.value if op == '*' else val // r.value
                    cur = r.pos
                else:
                    break
            return Success(val, cur)
        return self._memoize("product", pos, _parse)

    def parse_sum(self, pos: int) -> Result:
        def _parse(p: int) -> Result:
            left = self.parse_product(p)
            if left is FAIL:
                return FAIL
            val, cur = left.value, left.pos
            while True:
                cur = self.skip_ws(cur)
                if cur < len(self.text) and self.text[cur] in '+-':
                    op = self.text[cur]
                    r = self.parse_product(cur + 1)
                    if r is FAIL:
                        break
                    val = val + r.value if op == '+' else val - r.value
                    cur = r.pos
                else:
                    break
            return Success(val, cur)
        return self._memoize("sum", pos, _parse)

    def parse_expr(self, pos: int) -> Result:
        return self._memoize("expr", pos, self.parse_sum)


def eval_expr(text: str) -> int:
    parser = PackratParser(text)
    result = parser.parse_expr(0)
    if result is FAIL or result.pos != len(text.strip()):
        raise ValueError(f"파싱 실패: {text!r}")
    return result.value


# ─── 테스트 ──────────────────────────────────────────────────────────
print(eval_expr("1 + 2 * 3"))        # 7  (우선순위 올바름)
print(eval_expr("(1 + 2) * 3"))      # 9
print(eval_expr("10 - 3 - 2"))       # 5  (좌결합)
print(eval_expr("-5 * (3 + 2)"))     # -25
print(eval_expr("100 / 10 / 2"))     # 5
```

### 예제 2: 파서 조합자 라이브러리로 PEG 문법 선언적으로 작성

실전에서는 `pyparsing`(Python), `nom`(Rust), `parsec`(Haskell) 같은 PEG 기반 파서 조합자 라이브러리를 사용한다.

```python
# pip install pyparsing
from pyparsing import (
    Word, alphas, alphanums, nums, Literal, Optional, ZeroOrMore,
    Forward, Group, Suppress, pyparsing_common, infixNotation, opAssoc,
    QuotedString, Keyword, ParseException
)

# ─── JSON 서브셋 파서 (PEG 스타일) ──────────────────────────────────
# value ← object / array / string / number / "true" / "false" / "null"
# object ← "{" (member ("," member)*)? "}"
# member ← string ":" value
# array ← "[" (value ("," value)*)? "]"

json_value = Forward()   # 순환 참조를 위한 전방 선언

# 기본 타입
json_null    = Keyword("null").setParseAction(lambda: None)
json_true    = Keyword("true").setParseAction(lambda: True)
json_false   = Keyword("false").setParseAction(lambda: False)
json_number  = pyparsing_common.number()
json_string  = QuotedString('"', escChar='\\')

# 배열: "[" value* "]"
json_array = Group(
    Suppress("[") +
    Optional(json_value + ZeroOrMore(Suppress(",") + json_value)) +
    Suppress("]")
)

# 객체: "{" (string ":" value)* "}"
json_pair = Group(json_string + Suppress(":") + json_value)
json_object = Group(
    Suppress("{") +
    Optional(json_pair + ZeroOrMore(Suppress(",") + json_pair)) +
    Suppress("}")
)

# 순서 선택(ordered choice): / 연산자
json_value <<= (
    json_null    |   # 먼저 null 시도
    json_true    |   # 다음 true
    json_false   |   # 다음 false
    json_number  |   # 다음 숫자
    json_string  |   # 다음 문자열
    json_array   |   # 다음 배열
    json_object      # 마지막으로 객체
)

# 파싱 테스트
samples = [
    '{"name": "Alice", "age": 30, "scores": [95, 87, 92]}',
    '[1, true, null, "hello", {"key": "value"}]',
    '42',
    '"simple string"',
]

for s in samples:
    try:
        result = json_value.parseString(s, parseAll=True)
        print(f"입력: {s!r}")
        print(f"결과: {result.asList()}\n")
    except ParseException as e:
        print(f"오류: {e}\n")

# ─── 긍정/부정 예측(lookahead) 활용 ────────────────────────────────
from pyparsing import FollowedBy, NotAny

# !keyword identifier: 예약어가 아닌 식별자
keywords = {"if", "else", "while", "for", "return"}
reserved = Keyword("if") | Keyword("else") | Keyword("while") \
         | Keyword("for") | Keyword("return")

identifier = ~reserved + Word(alphas + "_", alphanums + "_")
# ~reserved는 NotAny(reserved)와 동일: 부정 예측
```

## PEG의 한계와 주의사항

### 1. 좌재귀(Left Recursion) 불가

PEG에서 직접 좌재귀는 무한 루프를 일으킨다. `A ← A b / c` 같은 규칙은 `A`가 `A`를 호출하고 `A`가 다시 `A`를 호출하는 무한 재귀에 빠진다.

```
# 잘못된 PEG: 좌재귀 (무한 루프!)
# Expr ← Expr "+" Term / Term

# 올바른 변환: 반복으로 대체
# Expr ← Term ("+" Term)*
```

Warth et al.(2008)이 제안한 메모이제이션 기반 좌재귀 처리 알고리즘이 있으며, 일부 PEG 라이브러리(`pest`, PEG.js)는 이를 지원한다.

### 2. 메모리 사용량 O(n × k)

패크랫 파싱의 O(n) 시간 보증은 O(n × k) 공간을 대가로 한다. 규칙 수 k가 크고 입력 n이 크면 메모 테이블이 매우 커질 수 있다. 실용적으로는 메모이제이션을 선택적으로 적용(자주 호출되는 규칙만)하여 균형을 맞춘다.

### 3. 의미 있는 오류 메시지 생성

PEG 파서는 "가장 멀리 도달한 위치(farthest failure position)"를 추적하면 의미 있는 오류 메시지를 생성할 수 있다. 순서 선택 중 가장 깊은 위치에서 실패한 지점이 사용자 오류와 가장 가깝기 때문이다.

```python
class PackratParserWithError(PackratParser):
    def __init__(self, text: str):
        super().__init__(text)
        self.farthest_pos = 0
        self.farthest_msg = ""

    def fail(self, pos: int, msg: str):
        if pos > self.farthest_pos:
            self.farthest_pos = pos
            self.farthest_msg = msg
        return FAIL
```

### 4. CFG vs PEG 선택 기준

| 상황 | 추천 |
|------|------|
| 기존 언어 명세(BNF/EBNF)가 있고 모호성이 없다 | LALR/GLR (Yacc/Bison) |
| 새 언어를 직접 설계, 렉서 없이 한번에 처리 | PEG + Packrat |
| 성능보다 가독성, 프로토타이핑 | PEG 파서 조합자 |
| 자연어, 고도로 모호한 문법 | Earley, GLR |

### 5. 실전 PEG 파서 라이브러리

| 언어 | 라이브러리 | 특징 |
|------|----------|------|
| Rust | `pest`, `nom` | 매크로 기반 PEG, 바이트 수준 nom |
| Python | `pyparsing`, `parsimonious` | 파이써닉한 조합자 |
| JavaScript | `PEG.js`, `nearley` | 웹 파싱, 좌재귀 지원 |
| Haskell | `parsec`, `megaparsec` | 모나딕 파서 조합자 |
| Java | `ANTLR 4` | PEG 유사 LL(*) 알고리즘 |
| Go | `participle` | 구조체 태그 기반 |

## 참고 자료

- [Parsing Expression Grammars: A Recognition-Based Syntactic Foundation (Bryan Ford, POPL 2004)](https://pdos.csail.mit.edu/~baford/packrat/popl04/peg-popl04.pdf)
- [Packrat Parsing: Simple, Powerful, Lazy, Linear Time (Bryan Ford, ICFP 2002)](https://pdos.csail.mit.edu/~baford/packrat/icfp02/)
- [The Packrat Parsing and PEG Bibliography — bford.info](https://bford.info/packrat/)
- [pest — Rust PEG 파서 공식 문서](https://pest.rs/book/)
