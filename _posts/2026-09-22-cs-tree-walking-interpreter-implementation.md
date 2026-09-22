---
layout: post
title: "AST 워킹 인터프리터 완전 정복: 렉서부터 클로저까지 나만의 언어 구현하기"
date: 2026-09-22
categories: [cs, computer-science]
tags: [interpreter, compiler, ast, lexer, parser, closures, programming-languages, environment-model]
---

## 개념 설명

프로그래밍 언어를 실행하는 방법은 크게 두 가지입니다. 하나는 소스 코드를 기계어나 바이트코드로 **컴파일**하는 방식이고, 다른 하나는 소스 코드를 그 자리에서 해석해 실행하는 **인터프리팅(Interpreting)** 방식입니다.

**AST 워킹 인터프리터(Tree-Walking Interpreter)**는 가장 직관적인 인터프리터 구현 방식입니다. 소스 코드를 추상 구문 트리(Abstract Syntax Tree, AST)로 변환한 뒤, 트리의 각 노드를 재귀적으로 방문하며 의미를 즉시 실행합니다. Python, Ruby의 초기 구현, PHP 4 등이 이 방식을 사용했습니다.

전체 파이프라인:
```
소스 코드 → [렉서] → 토큰 스트림 → [파서] → AST → [평가기] → 결과값
```

## 왜 직접 구현을 이해해야 하는가

### 디버깅 능력 향상

언어의 내부 동작을 이해하면 이상한 버그의 원인을 빠르게 파악할 수 있습니다. "왜 JavaScript에서 `this`가 바뀌는가?", "왜 클로저에서 루프 변수를 캡처하면 예상과 다른 값이 나오는가?" 같은 질문에 명확히 답할 수 있게 됩니다.

### 도메인 특화 언어(DSL) 설계

설정 파일, 쿼리 언어, 템플릿 엔진 등 DSL을 만들 때 인터프리터 지식이 필수입니다. Nginx 설정, Terraform HCL, Jest 매처 등 현대 소프트웨어는 도처에 미니 언어를 사용합니다.

### 컴퓨터 과학의 정수

인터프리터 구현은 자료구조, 재귀, 스코프 규칙, 메모리 관리를 모두 아우르는 종합 과제입니다. 이를 완성했을 때의 이해 깊이는 다른 방식으로 얻기 어렵습니다.

## 단계별 구현

우리는 간단한 계산 언어를 Python으로 구현합니다. 지원할 기능: 숫자·문자열·불리언, 사칙연산, 변수, 조건문, 함수, 클로저.

### 1단계: 렉서 (Lexer / Tokenizer)

렉서는 소스 코드 문자열을 의미 있는 토큰 시퀀스로 변환합니다.

```python
from enum import Enum, auto
from dataclasses import dataclass
from typing import Any, Optional

class TokenType(Enum):
    # 리터럴
    NUMBER = auto()
    STRING = auto()
    TRUE = auto()
    FALSE = auto()
    NIL = auto()
    # 식별자
    IDENT = auto()
    # 연산자
    PLUS = auto(); MINUS = auto(); STAR = auto(); SLASH = auto()
    EQ = auto(); NEQ = auto(); LT = auto(); GT = auto()
    ASSIGN = auto()
    # 구분자
    LPAREN = auto(); RPAREN = auto()
    LBRACE = auto(); RBRACE = auto()
    COMMA = auto(); SEMI = auto()
    # 키워드
    LET = auto(); FN = auto(); IF = auto(); ELSE = auto()
    RETURN = auto()
    EOF = auto()

@dataclass
class Token:
    type: TokenType
    value: Any
    line: int

KEYWORDS = {
    'let': TokenType.LET, 'fn': TokenType.FN,
    'if': TokenType.IF, 'else': TokenType.ELSE,
    'return': TokenType.RETURN, 'true': TokenType.TRUE,
    'false': TokenType.FALSE, 'nil': TokenType.NIL,
}

class Lexer:
    def __init__(self, source: str):
        self.source = source
        self.pos = 0
        self.line = 1

    def peek(self) -> Optional[str]:
        return self.source[self.pos] if self.pos < len(self.source) else None

    def advance(self) -> str:
        ch = self.source[self.pos]
        self.pos += 1
        if ch == '\n': self.line += 1
        return ch

    def tokenize(self) -> list[Token]:
        tokens = []
        while self.pos < len(self.source):
            ch = self.peek()
            if ch in ' \t\r\n':
                self.advance(); continue
            if ch == '#':
                while self.peek() and self.peek() != '\n': self.advance()
                continue
            
            if ch.isdigit():
                start = self.pos
                while self.peek() and (self.peek().isdigit() or self.peek() == '.'):
                    self.advance()
                tokens.append(Token(TokenType.NUMBER, float(self.source[start:self.pos]), self.line))
            elif ch == '"':
                self.advance()
                start = self.pos
                while self.peek() and self.peek() != '"': self.advance()
                tokens.append(Token(TokenType.STRING, self.source[start:self.pos], self.line))
                self.advance()
            elif ch.isalpha() or ch == '_':
                start = self.pos
                while self.peek() and (self.peek().isalnum() or self.peek() == '_'):
                    self.advance()
                word = self.source[start:self.pos]
                ttype = KEYWORDS.get(word, TokenType.IDENT)
                tokens.append(Token(ttype, word, self.line))
            else:
                self.advance()
                mapping = {
                    '+': TokenType.PLUS, '-': TokenType.MINUS,
                    '*': TokenType.STAR, '/': TokenType.SLASH,
                    '(': TokenType.LPAREN, ')': TokenType.RPAREN,
                    '{': TokenType.LBRACE, '}': TokenType.RBRACE,
                    ',': TokenType.COMMA, ';': TokenType.SEMI,
                    '<': TokenType.LT, '>': TokenType.GT,
                }
                if ch == '=' and self.peek() == '=':
                    self.advance(); tokens.append(Token(TokenType.EQ, '==', self.line))
                elif ch == '!=' :
                    pass  # 간략화
                elif ch == '=':
                    tokens.append(Token(TokenType.ASSIGN, '=', self.line))
                elif ch in mapping:
                    tokens.append(Token(mapping[ch], ch, self.line))
        tokens.append(Token(TokenType.EOF, None, self.line))
        return tokens
```

### 2단계: AST 노드 정의

```python
from dataclasses import dataclass, field
from typing import Union

# AST 노드 타입들
@dataclass
class NumberLit:   value: float
@dataclass
class StringLit:   value: str
@dataclass
class BoolLit:     value: bool
@dataclass
class NilLit:      pass
@dataclass
class Identifier:  name: str
@dataclass
class BinOp:
    op: str
    left: Any
    right: Any
@dataclass
class LetStmt:
    name: str
    value: Any
@dataclass
class IfExpr:
    condition: Any
    then_body: list
    else_body: list
@dataclass
class FnLit:
    params: list[str]
    body: list
@dataclass
class CallExpr:
    func: Any
    args: list
@dataclass
class ReturnStmt:
    value: Any
```

### 3단계: 환경 모델 (Environment)

환경(Environment)은 변수 이름을 값에 매핑하는 스코프입니다. 중첩 스코프는 외부 환경에 대한 참조로 구현합니다.

```python
class Environment:
    def __init__(self, outer=None):
        self.store = {}
        self.outer = outer  # 외부 스코프 참조

    def get(self, name: str):
        if name in self.store:
            return self.store[name]
        if self.outer:
            return self.outer.get(name)  # 외부 스코프 탐색
        raise NameError(f"Undefined variable: '{name}'")

    def set(self, name: str, value):
        self.store[name] = value
        return value

    def set_existing(self, name: str, value):
        """이미 선언된 변수 재할당 (외부 스코프까지 탐색)"""
        if name in self.store:
            self.store[name] = value
            return value
        if self.outer:
            return self.outer.set_existing(name, value)
        raise NameError(f"Undefined variable: '{name}'")
```

### 4단계: 평가기 (Evaluator)

평가기가 핵심입니다. AST를 재귀적으로 순회하며 각 노드의 의미를 실행합니다.

```python
class ReturnValue(Exception):
    def __init__(self, value): self.value = value

class Function:
    """함수 값 — 파라미터, 바디, 클로저 환경을 함께 보관"""
    def __init__(self, params, body, closure_env):
        self.params = params
        self.body = body
        self.closure_env = closure_env  # 선언 시점의 환경 캡처

class Evaluator:
    def eval(self, node, env: Environment):
        match node:
            case NumberLit(value=v): return v
            case StringLit(value=v): return v
            case BoolLit(value=v):   return v
            case NilLit():           return None
            
            case Identifier(name=name):
                return env.get(name)
            
            case BinOp(op=op, left=l, right=r):
                lv = self.eval(l, env)
                rv = self.eval(r, env)
                match op:
                    case '+': return lv + rv
                    case '-': return lv - rv
                    case '*': return lv * rv
                    case '/': return lv / rv
                    case '<': return lv < rv
                    case '>': return lv > rv
                    case '==': return lv == rv
            
            case LetStmt(name=name, value=val_node):
                val = self.eval(val_node, env)
                env.set(name, val)
                return val
            
            case IfExpr(condition=cond, then_body=then, else_body=else_):
                if self.eval(cond, env):
                    return self.eval_block(then, env)
                else:
                    return self.eval_block(else_, env) if else_ else None
            
            case FnLit(params=params, body=body):
                # 현재 환경을 클로저로 캡처
                return Function(params, body, closure_env=env)
            
            case CallExpr(func=fn_node, args=arg_nodes):
                fn = self.eval(fn_node, env)
                args = [self.eval(a, env) for a in arg_nodes]
                return self.apply_function(fn, args)
            
            case ReturnStmt(value=v):
                raise ReturnValue(self.eval(v, env))
    
    def eval_block(self, stmts, env):
        result = None
        for stmt in stmts:
            result = self.eval(stmt, env)
        return result
    
    def apply_function(self, fn: Function, args: list):
        # 클로저 환경을 외부로 하는 새 환경 생성
        local_env = Environment(outer=fn.closure_env)
        for param, arg in zip(fn.params, args):
            local_env.set(param, arg)
        try:
            return self.eval_block(fn.body, local_env)
        except ReturnValue as rv:
            return rv.value
```

### 클로저 테스트

```python
# 테스트: 클로저가 외부 환경을 올바르게 캡처하는지 확인
ev = Evaluator()
global_env = Environment()

# let counter = fn() { let count = 0; fn() { count = count + 1; count } }
# 간략화된 표현으로 직접 노드 생성
make_counter = FnLit(
    params=[],
    body=[
        LetStmt("count", NumberLit(0)),
        FnLit(params=[], body=[
            LetStmt("count", BinOp('+', Identifier("count"), NumberLit(1))),
            Identifier("count")
        ])
    ]
)

global_env.set("make_counter", ev.eval(make_counter, global_env))

# 평가기 출력 예시 (실제 파서 없이 AST 직접 구성)
# counter1, counter2를 각각 만들면 독립적인 클로저 환경을 가짐
```

## 바이트코드 VM과의 비교

AST 워킹 인터프리터는 구현이 단순하지만 각 노드 방문 시 dispatch 비용이 있습니다. CPython은 2008년부터 바이트코드 VM 방식을 사용하며, 소스를 먼저 컴팩트한 바이트코드로 컴파일한 뒤 스택 기반 가상 머신에서 실행합니다. Ruby 1.9+, Lua, LuaJIT, V8(초기)도 비슷한 구조입니다.

| 방식 | 장점 | 단점 |
|---|---|---|
| AST 워킹 | 구현 단순, 빠른 프로토타이핑 | 실행 느림, 노드 순회 오버헤드 |
| 바이트코드 VM | 빠른 실행, 직렬화 가능 | 구현 복잡, 컴파일 단계 추가 |
| JIT 컴파일 | 최고 성능 | 매우 복잡한 구현 |

## 주의사항과 팁

### 1. 스코프 버그 — 클로저 환경은 복사가 아닌 참조

가장 흔한 실수입니다. 함수를 생성할 때 현재 환경을 **복사**하면 나중에 외부 변수가 변경되어도 클로저는 이전 값을 봅니다. 올바른 구현은 환경 객체 자체에 대한 **참조**를 저장해야 합니다.

### 2. 무한 재귀 감지

인터프리터는 재귀 깊이 제한이 없으면 스택 오버플로우가 발생합니다. 호출 스택 깊이를 추적하고 한계를 초과하면 명시적 오류를 발생시켜야 합니다.

### 3. 꼬리 호출 최적화(TCO)

재귀 프로그래밍 언어에서 꼬리 위치의 함수 호출은 새 스택 프레임을 생성하지 않고 현재 프레임을 재사용해야 합니다. 트리 워킹 인터프리터에서 TCO는 `eval_block` 내에서 루프로 변환하는 방식으로 구현합니다.

### 4. 에러 처리 — 위치 정보 유지

토큰에 line/column 정보를 저장하고 평가 오류 발생 시 이 정보를 함께 리포트해야 사용자가 오류를 빠르게 찾을 수 있습니다.

## 참고 자료

- [Crafting Interpreters — Robert Nystrom](https://craftinginterpreters.com/)
- [Structure and Interpretation of Computer Programs (SICP)](https://mitpress.mit.edu/sites/default/files/sicp/full-text/book/book.html)
- [Writing an Interpreter in Go — Thorsten Ball](https://interpreterbook.com/)
- [Bytecode Compilers and Interpreters — Max Bernstein](https://bernsteinbear.com/blog/bytecode-interpreters/)
