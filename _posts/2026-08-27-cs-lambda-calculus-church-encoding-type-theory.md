---
layout: post
title: "람다 대수 심화 — 함수형 프로그래밍의 수학적 토대와 Church 인코딩"
date: 2026-08-27
categories: [cs, computer-science]
tags: [lambda-calculus, functional-programming, church-encoding, type-theory, beta-reduction, y-combinator, turing-completeness]
---

Haskell, Scala, Clojure, 그리고 최근의 Rust와 Kotlin에 이르기까지 현대 언어들은 점점 더 함수형 프로그래밍 개념을 채택하고 있다. 이 모든 개념의 수학적 토대는 1930년대 Alonzo Church가 고안한 **람다 대수(Lambda Calculus)** 에 있다. 람다 대수를 깊이 이해하면 클로저, 고차 함수, 타입 시스템, 심지어 `async/await`의 내부 동작까지 한층 명확하게 이해할 수 있다.

## 개념 설명: 람다 대수란?

람다 대수는 세 가지 요소만으로 이루어진 극단적으로 단순한 계산 모델이다.

### 문법 (Syntax)

람다 항(lambda term) `M`은 다음 세 가지 형태 중 하나다:

```
M ::= x           (변수, Variable)
    | λx.M        (추상화, Abstraction — "x를 받아 M을 반환하는 함수")
    | M N          (적용, Application — "함수 M에 인자 N 적용")
```

예시:
- `λx.x` — 항등 함수 (identity function)
- `λx.λy.x` — 두 인자 중 첫 번째를 반환하는 함수 (K combinator)
- `(λx.x x)(λx.x x)` — 무한 루프 (Omega combinator)

### 환원 규칙 (Reduction Rules)

**α-변환(Alpha Conversion)**: 묶인 변수의 이름 변경. `λx.x ≡ λy.y` — 두 표현은 같은 함수.

**β-환원(Beta Reduction)**: 함수 적용의 계산 규칙:

```
(λx.M) N → M[x := N]
```

`x`에 `N`을 대입하여 `M`을 계산한다.

예시:
```
(λx.x + 1) 5
→ 5 + 1       (β-환원: x에 5 대입)
→ 6
```

여러 단계:
```
(λx.λy.x + y) 3 4
→ (λy.3 + y) 4    (3을 x에 대입)
→ 3 + 4            (4를 y에 대입)
→ 7
```

**η-변환(Eta Conversion)**: `λx.(f x) ≡ f` — `f`와 `λx.(f x)`는 같은 함수.

### 커링(Currying)

람다 대수에서 모든 함수는 단일 인자만 받는다. 다중 인자 함수는 **커링**으로 표현한다.

```
f(x, y) = x + y
→ λx.λy.x + y
→ f(3)(4) = 7
```

이것이 Haskell에서 모든 함수가 기본적으로 커링되어 있는 이유다.

## 왜 필요한가?

### 1. 계산 이론의 기반

람다 대수는 튜링 기계(Turing Machine)와 동등한 계산 능력을 가진다 (Church-Turing Thesis). 즉, 컴퓨터로 계산 가능한 모든 함수는 람다 대수로 표현 가능하다. 이는 프로그래밍 언어 설계의 이론적 상한선을 정의한다.

### 2. 프로그래밍 언어 설계

현대 언어의 많은 기능이 람다 대수에서 유래한다:
- **클로저(Closure)**: 자유 변수를 캡처하는 람다 항
- **고차 함수(HOF)**: 함수를 인자로 받거나 반환하는 함수
- **부분 적용(Partial Application)**: 커링된 함수에 일부 인자만 적용
- **지연 평가(Lazy Evaluation)**: 정규형이 존재하면 어떤 환원 순서를 선택해도 동일한 결과 (Church-Rosser Theorem)

### 3. 타입 이론의 기반

타입이 있는 람다 대수(Simply Typed Lambda Calculus)는 현대 타입 시스템의 기반이다. System F(다형 람다 대수)는 Haskell의 타입 시스템, Java/Kotlin의 제네릭, Rust의 트레이트 시스템의 이론적 토대다.

## 실제 구현 예제

### 예제 1: Python으로 람다 대수 인터프리터 구현

```python
from __future__ import annotations
from dataclasses import dataclass
from typing import Union

# 람다 대수의 세 가지 항 표현
@dataclass
class Var:
    name: str
    def __repr__(self): return self.name

@dataclass
class Lam:  # λx.body
    param: str
    body: Term
    def __repr__(self): return f"(λ{self.param}.{self.body})"

@dataclass
class App:  # func arg
    func: Term
    arg: Term
    def __repr__(self): return f"({self.func} {self.arg})"

Term = Union[Var, Lam, App]

# 자유 변수 집합
def free_vars(term: Term) -> set[str]:
    if isinstance(term, Var):
        return {term.name}
    elif isinstance(term, Lam):
        return free_vars(term.body) - {term.param}
    else:  # App
        return free_vars(term.func) | free_vars(term.arg)

# 변수 이름 생성 (α-변환용)
_counter = 0
def fresh_var(base="x") -> str:
    global _counter
    _counter += 1
    return f"{base}_{_counter}"

# 대입: M[x := N]
def substitute(M: Term, x: str, N: Term) -> Term:
    if isinstance(M, Var):
        return N if M.name == x else M

    elif isinstance(M, App):
        return App(substitute(M.func, x, N), substitute(M.arg, x, N))

    elif isinstance(M, Lam):
        if M.param == x:
            return M  # x가 λx에 의해 묶임 — 대입 불가
        elif M.param not in free_vars(N):
            return Lam(M.param, substitute(M.body, x, N))
        else:
            # 변수 충돌 — α-변환으로 회피
            fresh = fresh_var(M.param)
            renamed = substitute(M.body, M.param, Var(fresh))
            return Lam(fresh, substitute(renamed, x, N))

# β-환원 (한 단계)
def beta_step(term: Term) -> tuple[Term, bool]:
    """환원 가능하면 (결과, True), 아니면 (원본, False)"""
    if isinstance(term, App):
        if isinstance(term.func, Lam):
            # (λx.M) N → M[x := N]
            result = substitute(term.func.body, term.func.param, term.arg)
            return result, True
        # 서브항에서 환원 시도
        func2, reduced = beta_step(term.func)
        if reduced:
            return App(func2, term.arg), True
        arg2, reduced = beta_step(term.arg)
        if reduced:
            return App(term.func, arg2), True
    elif isinstance(term, Lam):
        body2, reduced = beta_step(term.body)
        if reduced:
            return Lam(term.param, body2), True
    return term, False

# 정규형이 될 때까지 β-환원 반복
def normalize(term: Term, max_steps=100) -> Term:
    for step in range(max_steps):
        result, reduced = beta_step(term)
        if not reduced:
            print(f"  ({step} steps) Normal form: {term}")
            return term
        term = result
    print(f"  ({max_steps} steps) Did not converge (possibly infinite loop)")
    return term

# Church 인코딩 빌더
def church_num(n: int) -> Term:
    """n을 Church 숫자로 인코딩: λf.λx. f(f(f...f(x)...))"""
    f, x = Var("f"), Var("x")
    body = x
    for _ in range(n): body = App(f, body)
    return Lam("f", Lam("x", body))

# 테스트
print("=== β-환원 데모 ===")

# (λx.x) y → y (항등 함수)
I = Lam("x", Var("x"))
term1 = App(I, Var("y"))
print(f"\n항등 함수: {term1}")
normalize(term1)

# (λx.λy.x) a b → a (K combinator, 상수 함수)
K = Lam("x", Lam("y", Var("x")))
term2 = App(App(K, Var("a")), Var("b"))
print(f"\nK combinator: {term2}")
normalize(term2)

# Church 숫자 확인
print(f"\nChurch 0: {church_num(0)}")
print(f"Church 1: {church_num(1)}")
print(f"Church 2: {church_num(2)}")
print(f"Church 3: {church_num(3)}")
```

### 예제 2: Church 인코딩 — 데이터를 함수로 표현하기

Church 인코딩의 핵심 아이디어: **데이터 자체가 자신에 대한 연산을 선택하는 함수다**.

```python
# Python의 람다를 이용한 Church 인코딩 시뮬레이션

# === 불리언 ===
# TRUE = λt.λf.t  (첫 번째를 선택)
# FALSE = λt.λf.f (두 번째를 선택)
TRUE  = lambda t: lambda f: t
FALSE = lambda t: lambda f: f

# IF-THEN-ELSE = λb.λt.λf.b t f
IF = lambda b: lambda t: lambda f: b(t)(f)

# AND = λp.λq.p q p
AND = lambda p: lambda q: p(q)(p)

# OR = λp.λq.p p q
OR  = lambda p: lambda q: p(p)(q)

# NOT = λp.p FALSE TRUE
NOT = lambda p: p(FALSE)(TRUE)

# 불리언 → Python bool 변환
to_bool = lambda b: b(True)(False)

print("=== Church Boolean ===")
print(f"TRUE  = {to_bool(TRUE)}")
print(f"FALSE = {to_bool(FALSE)}")
print(f"AND TRUE FALSE = {to_bool(AND(TRUE)(FALSE))}")
print(f"OR  FALSE TRUE = {to_bool(OR(FALSE)(TRUE))}")
print(f"NOT TRUE = {to_bool(NOT(TRUE))}")

# === Church 숫자 ===
# 0 = λf.λx.x           (f를 0번 적용)
# 1 = λf.λx.f x          (f를 1번 적용)
# n = λf.λx.f(f(...f(x)...))  (f를 n번 적용)
ZERO  = lambda f: lambda x: x
ONE   = lambda f: lambda x: f(x)
TWO   = lambda f: lambda x: f(f(x))
THREE = lambda f: lambda x: f(f(f(x)))

# 숫자 → Python int 변환
to_int = lambda n: n(lambda x: x + 1)(0)

# SUCC = λn.λf.λx.f (n f x)  (1 증가)
SUCC = lambda n: lambda f: lambda x: f(n(f)(x))

# ADD = λm.λn.λf.λx.m f (n f x)  (m + n)
ADD  = lambda m: lambda n: lambda f: lambda x: m(f)(n(f)(x))

# MUL = λm.λn.λf.m (n f)  (m * n)
MUL  = lambda m: lambda n: lambda f: m(n(f))

# POW = λm.λn.n m  (m^n)
POW  = lambda m: lambda n: n(m)

FOUR = SUCC(THREE)
FIVE = ADD(TWO)(THREE)
SIX  = MUL(TWO)(THREE)

print("\n=== Church Numerals ===")
print(f"ZERO  = {to_int(ZERO)}")
print(f"ONE   = {to_int(ONE)}")
print(f"SUCC(THREE) = {to_int(FOUR)}")
print(f"ADD(2)(3)   = {to_int(FIVE)}")
print(f"MUL(2)(3)   = {to_int(SIX)}")
print(f"POW(2)(3)   = {to_int(POW(TWO)(THREE))}")

# === ISZERO: 숫자가 0인지 판별 ===
# ISZERO = λn.n (λx.FALSE) TRUE
ISZERO = lambda n: n(lambda _: FALSE)(TRUE)

print(f"\nISZERO(0) = {to_bool(ISZERO(ZERO))}")
print(f"ISZERO(3) = {to_bool(ISZERO(THREE))}")

# === Y Combinator: 재귀를 람다 대수로 표현 ===
# 람다 대수에는 재귀가 없다 — Y 컴비네이터로 구현
# Y = λf.(λx.f(x x))(λx.f(x x))
# Python의 엄격한 평가(eager evaluation)로 인해 Z combinator 사용
# Z = λf.(λx.f(λv.x x v))(λx.f(λv.x x v))

import sys
sys.setrecursionlimit(1000)

def Z(f):
    """Z Combinator (call-by-value 환경에서 동작하는 Y combinator 변형)"""
    return (lambda x: f(lambda v: x(x)(v)))(lambda x: f(lambda v: x(x)(v)))

# Z combinator로 팩토리얼 구현 (자기 자신을 참조하는 변수 없이!)
factorial = Z(lambda self: lambda n: 1 if n == 0 else n * self(n - 1))
fibonacci  = Z(lambda self: lambda n: n if n <= 1 else self(n - 1) + self(n - 2))

print("\n=== Y Combinator 재귀 ===")
print(f"factorial(5) = {factorial(5)}")
print(f"factorial(7) = {factorial(7)}")
print(f"fibonacci(10) = {fibonacci(10)}")
```

실행하면 이름 없는 익명 함수들만으로 `factorial`과 `fibonacci`를 구현하는 것을 확인할 수 있다.

## 주의사항과 팁

### 1. 평가 전략: Call-by-Value vs Call-by-Name

**Call-by-Value (엄격한 평가)**: 인자를 먼저 평가한 후 함수에 전달. Python, Java, Kotlin이 이 방식.

```
(λx.x + x)(2 + 3)
→ (λx.x + x)(5)    # 먼저 2+3=5 계산
→ 5 + 5 = 10
```

**Call-by-Name (지연 평가)**: 인자를 평가하지 않고 그대로 전달. Haskell이 이 방식(실제로는 Call-by-Need).

```
(λx.x + x)(2 + 3)
→ (2+3) + (2+3)     # 2+3을 두 번 계산할 수 있음
→ 5 + 5 = 10
```

Call-by-Name이 Call-by-Value보다 더 많은 항을 정규화할 수 있다 (발산하는 항을 포함할 때도 동작 가능).

### 2. 변수 캡처(Variable Capture) 주의

대입 시 자유 변수가 의도치 않게 묶이는 문제가 발생할 수 있다.

```
(λx.λy.x)[x := y]  # x에 y를 대입하려 할 때
→ λy.y              # 잘못된 결과! y가 λy에 캡처됨
```

올바른 처리는 α-변환으로 충돌하는 묶인 변수를 먼저 이름 변경한다:

```
(λx.λy.x)[x := y]
→ α: (λx.λz.x)[x := y]  # y → z로 α-변환
→ λz.y                    # 올바른 결과
```

### 3. Church-Rosser 정리의 실용적 의미

Church-Rosser 정리는 "정규형이 존재하면 어떤 환원 순서를 선택해도 동일한 결과에 도달한다"고 보장한다. 이것이 순수 함수형 프로그램에서 **참조 투명성(Referential Transparency)** 이 성립하는 이론적 근거이며, 병렬 평가가 가능한 이유이기도 하다.

### 4. 단순 타입 람다 대수 (Simply Typed Lambda Calculus)

타입 없는 람다 대수에서 `(λx.x x)(λx.x x)` 같은 무한 루프가 가능하다. **단순 타입 람다 대수**에서는 모든 항에 타입을 부여함으로써 무한 루프를 원천 차단한다 (강 정규화 정리, Strong Normalization).

```
타입 문법:
τ ::= α         (기본 타입: Int, Bool, ...)
    | τ → τ    (함수 타입)

타이핑 규칙:
  Γ, x:τ ⊢ M : σ
  ─────────────────  (추상화)
  Γ ⊢ λx.M : τ → σ

  Γ ⊢ M : τ → σ    Γ ⊢ N : τ
  ────────────────────────────  (적용)
  Γ ⊢ M N : σ
```

Haskell의 타입 추론(Hindley-Milner)은 이 규칙을 자동으로 적용하여 타입을 추론한다.

### 5. 현대 언어와의 연결

| 람다 대수 개념 | 현대 언어 구현체 |
|------------|--------------|
| λx.M | Python `lambda x: M`, Kotlin `{ x -> M }` |
| 커링 | Haskell의 모든 함수, Kotlin `fun f(x: Int) = { y: Int -> x + y }` |
| Y combinator | Rust의 trait 재귀, `fn` 포인터 트릭 |
| 단순 타입 λ대수 | Hindley-Milner 타입 추론 (Haskell, OCaml) |
| System F | Haskell `forall`, Rust `for<'a>` |
| Church 인코딩 | Rust `enum Option<T>`, Haskell `data Maybe a` |

Church 인코딩의 관점에서 Rust의 `Option::None`은 `λsome.λnone.none`이고, `Option::Some(v)`는 `λsome.λnone.some v`다. 이것이 바로 `match` 표현식이 동작하는 수학적 기반이다.

## 참고 자료
- [Lambda Calculi — Internet Encyclopedia of Philosophy](https://iep.utm.edu/lambda-calculi/)
- [Lambda Calculus — Stanford Encyclopedia of Philosophy](https://plato.stanford.edu/entries/lambda-calculus/)
- [Beta Reduction (Part 1) — PRL Blog, Northeastern University](https://prl.khoury.northeastern.edu/blog/2016/11/02/beta-reduction-part-1/)
- [Types and Programming Languages — Benjamin Pierce (MIT Press)](https://mitpress.mit.edu/9780262162098/types-and-programming-languages/)
