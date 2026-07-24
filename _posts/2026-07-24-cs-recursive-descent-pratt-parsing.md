---
layout: post
title: "재귀 하강 파서와 Pratt 파싱 완전 정복: 나만의 프로그래밍 언어 파서를 직접 구현하는 법"
date: 2026-07-24
categories: [cs, computer-science]
tags: [compiler, parser, recursive-descent, pratt-parsing, lexer, ast, interpreter, language-design]
---

## 개념 설명

컴파일러나 인터프리터를 만들 때 가장 먼저 마주치는 난관이 **파싱(parsing)**이다. 파싱은 소스 코드의 토큰 스트림을 받아 프로그램의 구조를 나타내는 **추상 구문 트리(AST, Abstract Syntax Tree)**를 만들어내는 과정이다. 파서를 구현하는 방법은 크게 두 가지 계열로 나뉜다: 파서 생성기(YACC, ANTLR)를 사용하는 방법과, 손으로 직접 작성하는 방법이다. 후자 중 가장 널리 쓰이는 기법이 **재귀 하강 파싱(Recursive Descent Parsing)**과 그 확장인 **Pratt 파싱**이다.

GCC, Clang, TypeScript, Rust 컴파일러, Go 컴파일러, Python 인터프리터(3.9+) 등 주요 언어의 프로덕션 컴파일러 대다수가 손으로 작성한 재귀 하강 파서를 사용한다. 이유는 간단하다: 유지보수가 쉽고, 오류 메시지를 정밀하게 제어할 수 있으며, 성능 최적화가 용이하기 때문이다.

### 재귀 하강 파싱의 원리

재귀 하강 파서는 문법의 각 규칙(production rule)을 하나의 함수로 변환한다. 문법 규칙들이 서로를 참조하면 함수들이 서로를 재귀적으로 호출한다는 뜻이다. 예를 들어 다음 문법을 생각해보자:

```
expression  → term (('+' | '-') term)*
term        → factor (('*' | '/') factor)*
factor      → NUMBER | '(' expression ')'
```

이 문법에서 `expression`은 `term`을 포함하고, `term`은 `factor`를 포함하며, `factor`는 다시 `expression`을 포함할 수 있다. 재귀 하강 파서에서는 이를 그대로 함수로 표현한다.

그런데 이 문법에는 **연산자 우선순위(precedence)**가 자연스럽게 인코딩되어 있다. `*`와 `/`가 `+`와 `-`보다 더 안쪽 규칙(`term`)에 있기 때문에 더 강하게 결합한다. 즉, `2 + 3 * 4`는 `2 + (3 * 4)`로 파싱된다.

### Pratt 파싱의 탄생 배경

위의 일반 재귀 하강 방식은 단점이 있다. 연산자가 늘어날수록 문법 레이어가 하나씩 추가되어야 한다. C 언어는 연산자 우선순위가 15단계인데, 이를 재귀 하강으로 처리하면 15개의 함수가 필요하다. 또한 후위 연산자(postfix), 중위 연산자(infix), 삼항 연산자(ternary) 등 다양한 형태의 연산자를 균일하게 처리하기 어렵다.

1973년 Vaughan Pratt는 논문 "Top Down Operator Precedence"에서 이 문제를 우아하게 해결하는 방법을 제안했다. **Pratt 파싱(또는 Top-Down Operator Precedence, TDOP)**은 각 토큰에 **결합력(binding power)**을 부여하고, 파싱 함수가 "최소 결합력"을 인자로 받아 그보다 강한 연산자들을 처리하는 방식이다.

---

## 왜 Pratt 파싱인가

Pratt 파싱의 핵심 강점은 **연산자 추가가 O(1)**이라는 점이다. 새 연산자를 추가할 때 파싱 규칙 전체를 수정할 필요 없이, 해당 토큰의 결합력과 파싱 함수만 등록하면 된다. 이는 언어 확장이 잦은 동적 DSL이나 매크로 시스템에서 특히 유용하다.

또한 Pratt 파싱은 **표현식 파싱에 집중**한다. 문장(statement) 파싱은 일반 재귀 하강으로, 표현식(expression) 파싱은 Pratt 방식으로 처리하는 **하이브리드 방식**이 실제 컴파일러에서 가장 많이 쓰인다. Crafting Interpreters의 Lox 언어, Rust 컴파일러, TypeScript 파서가 이 방식을 사용한다.

---

## 실제 구현 예제

### 렉서 구현

```python
from enum import Enum, auto
from dataclasses import dataclass
from typing import Optional

class TokenType(Enum):
    NUMBER = auto()
    PLUS = auto()       # +
    MINUS = auto()      # -
    STAR = auto()       # *
    SLASH = auto()      # /
    CARET = auto()      # ^ (지수)
    BANG = auto()       # ! (팩토리얼, 후위 연산자)
    LPAREN = auto()     # (
    RPAREN = auto()     # )
    EOF = auto()

@dataclass
class Token:
    type: TokenType
    value: str

class Lexer:
    def __init__(self, source: str):
        self.source = source
        self.pos = 0

    def next_token(self) -> Token:
        # 공백 건너뛰기
        while self.pos < len(self.source) and self.source[self.pos].isspace():
            self.pos += 1

        if self.pos >= len(self.source):
            return Token(TokenType.EOF, "")

        ch = self.source[self.pos]

        if ch.isdigit():
            start = self.pos
            while self.pos < len(self.source) and self.source[self.pos].isdigit():
                self.pos += 1
            return Token(TokenType.NUMBER, self.source[start:self.pos])

        self.pos += 1
        mapping = {
            '+': TokenType.PLUS,
            '-': TokenType.MINUS,
            '*': TokenType.STAR,
            '/': TokenType.SLASH,
            '^': TokenType.CARET,
            '!': TokenType.BANG,
            '(': TokenType.LPAREN,
            ')': TokenType.RPAREN,
        }
        if ch in mapping:
            return Token(mapping[ch], ch)
        raise SyntaxError(f"알 수 없는 문자: {ch!r}")

    def tokenize(self) -> list:
        tokens = []
        while True:
            tok = self.next_token()
            tokens.append(tok)
            if tok.type == TokenType.EOF:
                break
        return tokens
```

### Pratt 파서 구현

```python
@dataclass
class NumberNode:
    value: int
    def __str__(self): return str(self.value)

@dataclass
class BinaryNode:
    op: str
    left: object
    right: object
    def __str__(self): return f"({self.left} {self.op} {self.right})"

@dataclass
class UnaryNode:
    op: str
    operand: object
    prefix: bool  # True: 전위(-x), False: 후위(x!)
    def __str__(self):
        return f"({self.op}{self.operand})" if self.prefix else f"({self.operand}{self.op})"

class PrattParser:
    """
    결합력(binding power) 테이블:
      +, -  : 10 (낮음)
      *, /  : 20 (중간)
      ^ (지수): 30 (높음, 우결합)
      -(단항): 40 (전위 연산자)
      !    : 50 (후위 연산자, 매우 강함)
    """

    # (좌결합력, 우결합력) 쌍
    INFIX_BP = {
        TokenType.PLUS:  (10, 11),   # 좌결합: 좌 = 우 - 1이면 됨
        TokenType.MINUS: (10, 11),
        TokenType.STAR:  (20, 21),
        TokenType.SLASH: (20, 21),
        TokenType.CARET: (30, 29),   # 우결합: 우 < 좌 (30 > 29)
    }
    POSTFIX_BP = {
        TokenType.BANG: 50,
    }

    def __init__(self, tokens: list):
        self.tokens = tokens
        self.pos = 0

    def peek(self) -> Token:
        return self.tokens[self.pos]

    def consume(self) -> Token:
        tok = self.tokens[self.pos]
        self.pos += 1
        return tok

    def parse_expr(self, min_bp: int = 0) -> object:
        """
        Pratt 파싱의 핵심 함수.
        min_bp: 현재 컨텍스트에서 허용하는 최소 결합력
        """
        tok = self.consume()
        # --- 전위(prefix) 파싱 ---
        if tok.type == TokenType.NUMBER:
            left = NumberNode(int(tok.value))
        elif tok.type == TokenType.MINUS:
            # 단항 마이너스: 오른쪽 피연산자를 결합력 40으로 파싱
            operand = self.parse_expr(min_bp=40)
            left = UnaryNode('-', operand, prefix=True)
        elif tok.type == TokenType.LPAREN:
            left = self.parse_expr(min_bp=0)  # 괄호 안은 처음부터
            assert self.consume().type == TokenType.RPAREN, "')' 기대"
        else:
            raise SyntaxError(f"예상치 못한 토큰: {tok}")

        # --- 중위(infix) / 후위(postfix) 파싱 루프 ---
        while True:
            op_tok = self.peek()

            # 후위 연산자 처리
            if op_tok.type in self.POSTFIX_BP:
                bp = self.POSTFIX_BP[op_tok.type]
                if bp <= min_bp:
                    break
                self.consume()
                left = UnaryNode(op_tok.value, left, prefix=False)
                continue

            # 중위 연산자 처리
            if op_tok.type not in self.INFIX_BP:
                break
            l_bp, r_bp = self.INFIX_BP[op_tok.type]
            if l_bp <= min_bp:
                break
            self.consume()
            right = self.parse_expr(min_bp=r_bp)
            left = BinaryNode(op_tok.value, left, right)

        return left

    def parse(self):
        result = self.parse_expr()
        assert self.peek().type == TokenType.EOF, "예상치 못한 토큰"
        return result

def parse_and_eval(expr: str) -> tuple:
    """파싱 후 AST를 문자열로, 그리고 실제 평가 결과를 반환"""
    lexer = Lexer(expr)
    tokens = lexer.tokenize()
    parser = PrattParser(tokens)
    ast = parser.parse()

    def eval_node(node):
        if isinstance(node, NumberNode):
            return node.value
        if isinstance(node, BinaryNode):
            l, r = eval_node(node.left), eval_node(node.right)
            ops = {'+': l+r, '-': l-r, '*': l*r, '/': l/r, '^': l**r}
            return ops[node.op]
        if isinstance(node, UnaryNode):
            v = eval_node(node.operand)
            if node.op == '-': return -v
            if node.op == '!':
                import math; return math.factorial(v)

    return str(ast), eval_node(ast)

# 테스트
tests = [
    "1 + 2 * 3",          # 우선순위: 덧셈 < 곱셈
    "2 ^ 3 ^ 2",          # 우결합: 2^(3^2) = 2^9 = 512
    "-5 + 3",             # 단항 마이너스
    "5! + 1",             # 후위 팩토리얼
    "(1 + 2) * 3",        # 괄호
    "2 * 3 + 4 * 5",      # 혼합
]

for expr in tests:
    ast_str, result = parse_and_eval(expr)
    print(f"{expr:25s} → AST: {ast_str:35s} = {result}")
```

출력 결과:
```
1 + 2 * 3                 → AST: (1 + (2 * 3))                    = 7
2 ^ 3 ^ 2                 → AST: (2 ^ (3 ^ 2))                    = 512
-5 + 3                    → AST: ((-5) + 3)                       = -2
5! + 1                    → AST: ((5!) + 1)                       = 121
(1 + 2) * 3               → AST: ((1 + 2) * 3)                    = 9
2 * 3 + 4 * 5             → AST: ((2 * 3) + (4 * 5))              = 26
```

---

## 문장 파싱: 재귀 하강과의 하이브리드

실제 언어에서는 표현식만 있는 게 아니라 `if`, `while`, `return` 같은 문장도 있다. 이때는 문장 파싱에는 일반 재귀 하강을, 표현식 파싱에는 Pratt 방식을 결합한다.

```python
class StatementParser:
    """문장(statement) 파서 — 재귀 하강 방식"""

    def __init__(self, source: str):
        lexer = Lexer(source)
        self.tokens = lexer.tokenize()
        self.pos = 0

    def parse_statement(self):
        tok = self.peek()
        if tok.type == TokenType.EOF:
            return None
        # 표현식 문(expression statement)
        expr = self._parse_expression()
        return expr

    def _parse_expression(self):
        """Pratt 파서에 위임"""
        # 현재 위치에서 남은 토큰들로 Pratt 파서 생성
        sub_parser = PrattParser(self.tokens[self.pos:])
        result = sub_parser.parse_expr()
        self.pos += sub_parser.pos
        return result

    def peek(self):
        return self.tokens[self.pos] if self.pos < len(self.tokens) else Token(TokenType.EOF, "")
```

---

## 주의사항과 실전 팁

**좌재귀(left recursion) 문제**: 순수 재귀 하강은 좌재귀 문법을 처리하지 못한다. `expr → expr '+' term`처럼 자기 자신을 왼쪽에서 참조하면 무한 재귀에 빠진다. Pratt 파싱은 루프 기반이라 이 문제를 자연스럽게 회피한다.

**에러 복구(error recovery)**: 좋은 파서는 에러를 만나도 가능한 한 계속 파싱을 진행해 여러 오류를 한 번에 보고한다. 이를 위해 동기화 지점(synchronization point, 예: `;` 또는 `}`)을 찾아 파싱을 재개하는 "패닉 모드(panic mode)" 전략을 구현할 수 있다.

**결합력 선택**: 결합력 값 자체보다 그 상대적 순서가 중요하다. 좌결합 연산자는 `(l_bp, l_bp+1)`, 우결합 연산자는 `(r_bp, r_bp-1)`로 설정한다. 이렇게 하면 `a + b + c`는 `(a+b)+c`(좌결합), `a^b^c`는 `a^(b^c)`(우결합)로 올바르게 파싱된다.

**삼항 연산자 처리**: `a ? b : c` 같은 삼항 연산자도 Pratt 파싱으로 처리할 수 있다. `?`를 중위 연산자로 등록하고, 파싱 함수에서 `:` 토큰까지 읽어 false 분기를 파싱하면 된다.

**성능 최적화**: 토큰 룩업에 해시맵 대신 배열(enum 값을 인덱스로)을 사용하면 캐시 효율이 좋아진다. 또한 자주 호출되는 `parse_expr`를 인라인하거나 반복문으로 변환해 함수 호출 오버헤드를 줄일 수 있다.

## 참고 자료

- [Parsing Expressions · Crafting Interpreters](https://craftinginterpreters.com/parsing-expressions.html)
- [Simple but Powerful Pratt Parsing - matklad](https://matklad.github.io/2020/04/13/simple-but-powerful-pratt-parsing.html)
- [Recursive descent parser - Wikipedia](https://en.wikipedia.org/wiki/Recursive_descent_parser)
- [Pratt Parsers: Expression Parsing Made Easy - Bob Nystrom](http://journal.stuffwithstuff.com/2011/03/19/pratt-parsers-expression-parsing-made-easy/)
