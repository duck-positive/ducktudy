---
layout: post
title: "Parser Combinators 완전 정복: 함수형 파싱의 우아한 세계 — Rust·Haskell 구현"
date: 2026-09-12
categories: [cs, computer-science]
tags: [parser-combinators, parsing, functional-programming, rust, haskell, nom, monad, DSL]
---

## 개요

파서를 어떻게 작성하시나요? 정규표현식이 너무 약하고, lex/yacc는 러닝 커브가 가파르며, 손으로 짠 재귀 하강 파서는 유지보수가 지옥 같을 때 — **Parser Combinators**가 답입니다.

Parser Combinators는 **함수형 프로그래밍의 함성(Composition) 철학을 파싱에 적용**한 기법입니다. 단순한 파서들을 블록처럼 조합하여 복잡한 파서를 만들고, 각 조합 규칙 자체가 하나의 함수이므로 테스트·재사용·합성이 자연스럽습니다. Haskell의 Parsec, Rust의 nom, Scala의 fastparse 등이 대표적 구현체이며, 실제로 SQL 파서, Protobuf 파서, Markdown 파서 등 수많은 프로덕션 파서에 사용됩니다.

이 글에서는 Parser Combinators의 이론적 토대를 이해하고, Rust nom과 순수 Python 구현으로 직접 파서를 만들어봅니다.

---

## Parser Combinator의 핵심 아이디어

### 파서를 함수로 모델링

Parser Combinator의 출발점은 **파서를 단순한 함수로 정의**하는 것입니다.

```
Parser<T> = Input → Result<(T, Input), Error>

- Input: 파싱할 입력 문자열 (또는 그 나머지 부분)
- T: 파싱 결과 타입
- (T, Input): 성공 시 (파싱된 값, 남은 입력)
- Error: 실패 시 오류 정보
```

예를 들어 숫자 파서는 `String → Result<(i64, String), Error>` 형태입니다. 이 단순한 정의에서 파서 합성의 힘이 나옵니다.

### 기본 파서들

```python
from dataclasses import dataclass
from typing import TypeVar, Generic, Callable, Optional, Tuple, List

T = TypeVar('T')
U = TypeVar('U')

@dataclass
class ParseResult(Generic[T]):
    value: T
    remaining: str

# 파서 = 문자열을 받아 (값, 나머지) 또는 None 반환
Parser = Callable[[str], Optional[ParseResult]]

def char(c: str) -> Parser:
    """단일 문자를 파싱하는 기본 파서"""
    def parse(input: str) -> Optional[ParseResult]:
        if input and input[0] == c:
            return ParseResult(c, input[1:])
        return None
    return parse

def satisfy(pred: Callable[[str], bool]) -> Parser:
    """조건을 만족하는 문자를 파싱"""
    def parse(input: str) -> Optional[ParseResult]:
        if input and pred(input[0]):
            return ParseResult(input[0], input[1:])
        return None
    return parse

def string(s: str) -> Parser:
    """정확한 문자열을 파싱"""
    def parse(input: str) -> Optional[ParseResult]:
        if input.startswith(s):
            return ParseResult(s, input[len(s):])
        return None
    return parse

# 기본 파서들
digit = satisfy(str.isdigit)
letter = satisfy(str.isalpha)
space = satisfy(str.isspace)

# 테스트
print(char('a')('abc'))    # ParseResult(value='a', remaining='bc')
print(char('a')('xyz'))    # None
print(string('hello')('hello world'))  # ParseResult(value='hello', remaining=' world')
```

---

## 핵심 Combinator 구현

### many: 0번 이상 반복

```python
def many(parser: Parser) -> Parser:
    """parser를 0번 이상 반복 적용. 항상 성공 (결과는 리스트)"""
    def parse(input: str) -> Optional[ParseResult]:
        results = []
        while True:
            result = parser(input)
            if result is None:
                break
            results.append(result.value)
            input = result.remaining
        return ParseResult(results, input)
    return parse

def many1(parser: Parser) -> Parser:
    """parser를 1번 이상 반복 (0번은 실패)"""
    def parse(input: str) -> Optional[ParseResult]:
        result = many(parser)(input)
        if result and result.value:
            return result
        return None
    return parse
```

### seq: 순서대로 파싱 (AND 합성)

```python
def seq(*parsers: Parser) -> Parser:
    """여러 파서를 순서대로 적용, 모든 결과를 리스트로 반환"""
    def parse(input: str) -> Optional[ParseResult]:
        results = []
        remaining = input
        for p in parsers:
            result = p(remaining)
            if result is None:
                return None
            results.append(result.value)
            remaining = result.remaining
        return ParseResult(results, remaining)
    return parse

def seq_right(left: Parser, right: Parser) -> Parser:
    """left를 파싱하고 버린 뒤 right 결과만 반환 (>>)"""
    def parse(input: str) -> Optional[ParseResult]:
        l = left(input)
        if l is None:
            return None
        return right(l.remaining)
    return parse

def seq_left(left: Parser, right: Parser) -> Parser:
    """left 결과를 유지하고 right를 버림 (<<)"""
    def parse(input: str) -> Optional[ParseResult]:
        l = left(input)
        if l is None:
            return None
        r = right(l.remaining)
        if r is None:
            return None
        return ParseResult(l.value, r.remaining)
    return parse
```

### choice: 대안 선택 (OR 합성)

```python
def choice(*parsers: Parser) -> Parser:
    """첫 번째로 성공하는 파서의 결과 반환 (|)"""
    def parse(input: str) -> Optional[ParseResult]:
        for p in parsers:
            result = p(input)
            if result is not None:
                return result
        return None
    return parse
```

### map: 파싱 결과 변환

```python
def fmap(parser: Parser, func: Callable) -> Parser:
    """파서 결과에 함수 적용"""
    def parse(input: str) -> Optional[ParseResult]:
        result = parser(input)
        if result is None:
            return None
        return ParseResult(func(result.value), result.remaining)
    return parse
```

---

## 실전 예제: JSON 파서 만들기

이제 기본 Combinator들로 간단한 JSON 서브셋 파서를 만들어보겠습니다.

```python
# 공백 제거 헬퍼
def skip_spaces(parser: Parser) -> Parser:
    return seq_right(many(space), parser)

def token(parser: Parser) -> Parser:
    """앞뒤 공백을 무시하는 파서"""
    return seq_left(skip_spaces(parser), many(space))

# --- JSON 파서 구성 ---

def json_number() -> Parser:
    """정수 파싱"""
    def parse(input: str):
        neg = char('-')(input)
        start = neg.remaining if neg else input
        sign = '-' if neg else ''
        
        digits = many1(digit)(start)
        if digits is None or not digits.value:
            return None
        
        num = int(sign + ''.join(digits.value))
        return ParseResult(num, digits.remaining)
    return parse

def json_string() -> Parser:
    """간단한 문자열 파싱 (이스케이프 미지원)"""
    def parse(input: str):
        open_q = char('"')(input)
        if open_q is None:
            return None
        
        content = []
        remaining = open_q.remaining
        while remaining and remaining[0] != '"':
            content.append(remaining[0])
            remaining = remaining[1:]
        
        if not remaining:  # 닫는 따옴표 없음
            return None
        
        return ParseResult(''.join(content), remaining[1:])
    return parse

def json_bool() -> Parser:
    true_p = fmap(string('true'), lambda _: True)
    false_p = fmap(string('false'), lambda _: False)
    return choice(true_p, false_p)

def json_null() -> Parser:
    return fmap(string('null'), lambda _: None)

def json_value() -> Parser:
    """재귀적 JSON 값 파서"""
    def parse(input: str):
        return choice(
            token(json_number()),
            token(json_string()),
            token(json_bool()),
            token(json_null()),
            token(json_array()),
            token(json_object()),
        )(input)
    return parse

def json_array() -> Parser:
    """[v1, v2, ...] 파싱"""
    def parse(input: str):
        open_b = token(char('['))(input)
        if open_b is None:
            return None
        
        # 빈 배열
        close = token(char(']'))(open_b.remaining)
        if close:
            return ParseResult([], close.remaining)
        
        # 첫 번째 원소
        elements = []
        remaining = open_b.remaining
        
        first = json_value()(remaining)
        if first is None:
            return None
        elements.append(first.value)
        remaining = first.remaining
        
        # 나머지 원소들 (콤마 구분)
        while True:
            comma = token(char(','))(remaining)
            if comma is None:
                break
            elem = json_value()(comma.remaining)
            if elem is None:
                break
            elements.append(elem.value)
            remaining = elem.remaining
        
        close = token(char(']'))(remaining)
        if close is None:
            return None
        
        return ParseResult(elements, close.remaining)
    return parse

def json_object() -> Parser:
    """{"key": value, ...} 파싱"""
    def parse(input: str):
        open_b = token(char('{'))(input)
        if open_b is None:
            return None
        
        close = token(char('}'))(open_b.remaining)
        if close:
            return ParseResult({}, close.remaining)
        
        obj = {}
        remaining = open_b.remaining
        
        def parse_pair(inp):
            key = token(json_string())(inp)
            if key is None:
                return None
            colon = token(char(':'))(key.remaining)
            if colon is None:
                return None
            val = json_value()(colon.remaining)
            if val is None:
                return None
            return ParseResult((key.value, val.value), val.remaining)
        
        first = parse_pair(remaining)
        if first is None:
            return None
        obj[first.value[0]] = first.value[1]
        remaining = first.remaining
        
        while True:
            comma = token(char(','))(remaining)
            if comma is None:
                break
            pair = parse_pair(comma.remaining)
            if pair is None:
                break
            obj[pair.value[0]] = pair.value[1]
            remaining = pair.remaining
        
        close = token(char('}'))(remaining)
        if close is None:
            return None
        
        return ParseResult(obj, close.remaining)
    return parse

# 테스트
test_cases = [
    '42',
    '"hello world"',
    'true',
    '[1, 2, 3]',
    '{"name": "Alice", "age": 30, "active": true}',
    '{"scores": [100, 95, 87], "grade": "A"}',
]

parser = json_value()
for tc in test_cases:
    result = parser(tc)
    if result:
        print(f"입력: {tc!r:50s} → {result.value!r}")
    else:
        print(f"파싱 실패: {tc!r}")
```

출력:
```
입력: '42'                                              → 42
입력: '"hello world"'                                  → 'hello world'
입력: 'true'                                           → True
입력: '[1, 2, 3]'                                      → [1, 2, 3]
입력: '{"name": "Alice", "age": 30, "active": true}'  → {'name': 'Alice', 'age': 30, 'active': True}
입력: '{"scores": [100, 95, 87], "grade": "A"}'        → {'scores': [100, 95, 87], 'grade': 'A'}
```

---

## Rust nom: 프로덕션급 Parser Combinators

Rust의 **nom** 라이브러리는 바이너리 프로토콜부터 텍스트 DSL까지 처리하는 산업용 Parser Combinator 프레임워크입니다. 제로 카피(zero-copy) 파싱과 컴파일 타임 최적화를 지원합니다.

```rust
use nom::{
    IResult,
    branch::alt,
    bytes::complete::{tag, take_while1},
    character::complete::{char, digit1, multispace0},
    combinator::{map, map_res, opt},
    multi::separated_list0,
    sequence::{delimited, preceded, separated_pair, tuple},
};

#[derive(Debug, PartialEq)]
enum Expr {
    Num(i64),
    Var(String),
    Add(Box<Expr>, Box<Expr>),
    Mul(Box<Expr>, Box<Expr>),
}

// 공백 제거 헬퍼
fn ws<'a, F, O>(f: F) -> impl FnMut(&'a str) -> IResult<&'a str, O>
where
    F: FnMut(&'a str) -> IResult<&'a str, O>,
{
    delimited(multispace0, f, multispace0)
}

// 숫자 파서
fn parse_num(input: &str) -> IResult<&str, Expr> {
    map(
        map_res(
            preceded(opt(char('-')), digit1),
            |s: &str| s.parse::<i64>()
        ),
        Expr::Num
    )(input)
}

// 변수 이름 파서
fn parse_var(input: &str) -> IResult<&str, Expr> {
    map(
        take_while1(|c: char| c.is_alphabetic() || c == '_'),
        |s: &str| Expr::Var(s.to_string())
    )(input)
}

// 원자 표현식 (숫자 또는 변수 또는 괄호)
fn parse_atom(input: &str) -> IResult<&str, Expr> {
    ws(alt((
        parse_num,
        parse_var,
        delimited(char('('), parse_add, char(')')),
    )))(input)
}

// 곱셈 (높은 우선순위)
fn parse_mul(input: &str) -> IResult<&str, Expr> {
    let (input, first) = parse_atom(input)?;
    
    let mut result = first;
    let mut remaining = input;
    
    loop {
        match preceded(ws(char('*')), parse_atom)(remaining) {
            Ok((rest, right)) => {
                result = Expr::Mul(Box::new(result), Box::new(right));
                remaining = rest;
            }
            Err(_) => break,
        }
    }
    
    Ok((remaining, result))
}

// 덧셈 (낮은 우선순위)
fn parse_add(input: &str) -> IResult<&str, Expr> {
    let (input, first) = parse_mul(input)?;
    
    let mut result = first;
    let mut remaining = input;
    
    loop {
        match preceded(ws(char('+')), parse_mul)(remaining) {
            Ok((rest, right)) => {
                result = Expr::Add(Box::new(result), Box::new(right));
                remaining = rest;
            }
            Err(_) => break,
        }
    }
    
    Ok((remaining, result))
}

fn main() {
    let exprs = [
        "42",
        "x + y",
        "2 * 3 + 4",      // 우선순위: (2*3) + 4 = 10
        "(2 + 3) * 4",    // 우선순위: 5 * 4 = 20
        "a + b * c + d",  // a + (b*c) + d
    ];
    
    for expr in &exprs {
        match parse_add(expr) {
            Ok((remaining, ast)) => {
                println!("입력: {:20} → AST: {:?}", expr, ast);
                if !remaining.trim().is_empty() {
                    println!("  (나머지: {:?})", remaining);
                }
            }
            Err(e) => println!("파싱 실패: {:?}", e),
        }
    }
}
```

---

## Parser Combinators의 이론적 토대: Monad

Parser Combinator는 함수형 프로그래밍의 **Monad** 구조를 사용합니다.

```
Parser<T>는 Monad:

1. return (pure): T → Parser<T>
   - 값을 "항상 성공하는 파서"로 감쌈
   - pure(x) = λinput → (x, input)

2. bind (>>=): Parser<T> → (T → Parser<U>) → Parser<U>
   - 파서의 결과를 다음 파서에 전달
   - p >>= f = λinput → 
       match p(input) with
       | None → None
       | Some(v, rest) → f(v)(rest)
```

이 Monad 구조는 `seq`, `fmap` 등의 Combinator가 항상 **합성 가능(composable)**하도록 보장합니다. 실패 전파도 자동입니다 — 중간에 하나라도 실패하면 전체가 실패합니다.

---

## Parsec류 vs nom: 언제 무엇을 쓸까

| 특성 | Parsec/Megaparsec (Haskell) | nom (Rust) | 손수 파서 |
|------|---------------------------|-----------|---------|
| 에러 메시지 | 훌륭함 (줄/열 번호) | 보통 | 커스터마이즈 가능 |
| 바이너리 파싱 | 어색 | 최적 | 가능 |
| 제로 카피 | 제한적 | 지원 | 지원 가능 |
| 학습 곡선 | 중간 | 중간 | 낮음 |
| 성능 | 좋음 | 매우 빠름 | 최고 (최적화 시) |
| 왼쪽 재귀 | 직접 지원 안 함 | 직접 지원 안 함 | 주의 필요 |

**왼쪽 재귀**는 Parser Combinators의 주요 제약 사항입니다. `expr = expr '+' term` 같은 문법은 무한 루프를 발생시킵니다. Pratt 파싱이나 반복(`many`) 패턴으로 대체하여 해결합니다.

---

## 주의사항과 팁

### 1. 역추적(Backtracking) 비용

`choice`는 기본적으로 첫 번째 파서를 시도하고 실패하면 다음으로 넘어갑니다. 긴 입력을 소비하고 실패하면 역추적이 비쌀 수 있습니다. nom에서는 `cut`으로 역추적을 방지합니다.

```rust
// nom에서 cut: 여기서 실패하면 역추적하지 말고 즉시 오류 반환
use nom::combinator::cut;

fn parse_json_string(input: &str) -> IResult<&str, String> {
    let (input, _) = char('"')(input)?;
    // 큰따옴표를 파싱했으므로 이후 실패는 역추적 불가
    cut(parse_string_contents)(input)
}
```

### 2. 에러 메시지 품질

실전 파서에서는 에러 위치와 기대 값을 명확히 보고해야 합니다. Haskell Megaparsec은 `<?>` 연산자로 커스텀 에러 메시지를 설정합니다.

```haskell
-- Haskell Megaparsec 예시
jsonValue :: Parser Value
jsonValue = 
      (JNum <$> scientific <?> "number")
  <|> (JStr <$> stringLiteral <?> "string")
  <|> (JBool <$> boolean <?> "boolean")
  <|> (JNull <$ symbol "null" <?> "null")
```

### 3. 테스트 전략

Parser Combinators의 장점 중 하나는 **각 파서를 독립적으로 단위 테스트**할 수 있다는 것입니다.

```python
# 단위 테스트
assert digit('3abc') == ParseResult('3', 'abc')
assert digit('abc') is None
assert many(digit)('123abc') == ParseResult(['1','2','3'], 'abc')
assert json_value()('{"key": 42}').value == {'key': 42}
```

---

## 마치며

Parser Combinators는 "파싱은 어렵다"는 고정관념을 깨는 우아한 접근입니다. 복잡해 보이는 파서를 작은 함수들의 합성으로 표현할 수 있고, 각 조각은 독립적으로 테스트 가능하며, 전체는 읽기 쉬운 선언적 코드가 됩니다. DSL 파서, 설정 파일 파서, 프로토콜 분석기를 만들어야 할 때 Parser Combinators를 먼저 고려해보세요. 특히 Rust nom은 임베디드 시스템, 네트워크 프로토콜, 바이너리 포맷 파싱에 이미 광범위하게 사용되고 있는 검증된 라이브러리입니다.

## 참고 자료
- [nom 공식 문서 (Rust)](https://docs.rs/nom/latest/nom/)
- [Megaparsec 튜토리얼 (Haskell)](https://markkarpov.com/tutorial/megaparsec.html)
- [Monadic Parsing in Haskell — Hutton & Meijer (1998)](https://www.cs.nott.ac.uk/~pszgmh/monparsing.pdf)
- [fastparse 라이브러리 (Scala)](https://com-lihaoyi.github.io/fastparse/)
