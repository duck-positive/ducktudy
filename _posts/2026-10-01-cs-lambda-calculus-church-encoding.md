---
layout: post
title: "람다 칼큘러스와 처치 인코딩: 함수형 프로그래밍의 수학적 기반 완전 정복"
date: 2026-10-01
categories: [cs, computer-science]
tags: [lambda-calculus, church-encoding, functional-programming, type-theory, haskell, python]
---

## 람다 칼큘러스란 무엇인가

람다 칼큘러스(Lambda Calculus, λ-calculus)는 1930년대 Alonzo Church가 고안한 수학적 계산 모델이다. 함수를 정의하고 적용하는 규칙만으로 **모든 계산 가능한 함수**를 표현할 수 있다는 것을 증명한 이론으로, 튜링 머신과 동등한 계산 능력을 가진다.

람다 칼큘러스는 세 가지 기본 구성 요소만으로 이루어진다:

1. **변수(Variable)**: `x`, `y`, `z` — 값을 나타내는 이름
2. **추상화(Abstraction)**: `λx.e` — 매개변수 `x`를 받아 식 `e`를 반환하는 함수 정의
3. **적용(Application)**: `(f e)` — 함수 `f`에 인수 `e`를 적용

이 세 가지만으로 자연수, 불리언, 조건문, 재귀까지 전부 표현 가능하다. 현대 함수형 언어(Haskell, ML, Clojure, Erlang)는 모두 람다 칼큘러스를 이론적 기반으로 삼는다.

---

## 왜 람다 칼큘러스를 알아야 하는가

### 함수형 프로그래밍의 뿌리

Haskell의 `\x -> x + 1`, Python의 `lambda x: x + 1`, JavaScript의 `x => x + 1`은 모두 람다 표기법에서 직접 유래했다. 이 표기법의 수학적 의미를 이해하면:

- **고차 함수**가 왜 자연스럽게 합성 가능한지 알 수 있다
- **커링(Currying)**이 왜 가능하며 어떤 의미인지 이해된다
- **순수 함수**와 **참조 투명성** 개념의 수학적 근거를 파악할 수 있다
- **타입 이론**과 Curry-Howard 대응이 왜 중요한지 보인다

### 컴파일러와 인터프리터 설계

람다 칼큘러스는 컴파일러의 **중간 표현(Intermediate Representation)**으로 널리 사용된다. GHC(Glasgow Haskell Compiler)는 Haskell 소스를 `Core`라는 람다 칼큘러스 기반 IR로 변환한 뒤 최적화한다. 즉, 람다 칼큘러스를 이해하면 컴파일러가 코드를 어떻게 분석·최적화하는지 직관을 얻을 수 있다.

---

## 핵심 규칙: α-변환, β-환원, η-환원

### α-변환 (Alpha Conversion)

변수 이름 충돌을 피하기 위해 **매개변수 이름을 바꾸는 것**이다. 의미는 동일하다.

```
λx.x  ≡  λy.y  ≡  λz.z
```

### β-환원 (Beta Reduction)

함수를 **실제로 적용하는 규칙**이다. 함수 본체에서 매개변수를 인수로 치환한다.

```
(λx.x + 1) 5  →  5 + 1  →  6
(λx.λy.x + y) 3 4  →  (λy.3 + y) 4  →  3 + 4  →  7
```

### η-환원 (Eta Reduction)

`λx.(f x)`는 `f`와 동일하다. 즉 **포인트-프리(point-free) 스타일**의 수학적 근거다.

```
λx.(f x)  ≡  f  (단, x가 f 안에 자유변수로 없는 경우)
```

---

## 처치 인코딩 (Church Encoding)

처치 인코딩은 자료형과 연산을 **오직 함수만으로 표현**하는 방법이다. 핵심 아이디어는 "값이 무엇인지 생각하지 말고, 그 값이 어떻게 사용되는지 생각하라"는 것이다.

### 불리언 인코딩

불리언 `true`는 "두 값 중 첫 번째를 선택하는 함수", `false`는 "두 번째를 선택하는 함수"다.

```
true  = λt.λf.t   -- 두 인수 중 첫 번째 반환
false = λt.λf.f   -- 두 인수 중 두 번째 반환

if_then_else = λcond.λthen.λelse. cond then else
```

이를 Python으로 직접 구현하면:

```python
# 처치 인코딩 불리언 구현
TRUE  = lambda t: lambda f: t
FALSE = lambda t: lambda f: f

AND  = lambda p: lambda q: p(q)(p)
OR   = lambda p: lambda q: p(p)(q)
NOT  = lambda p: lambda a: lambda b: p(b)(a)
IF   = lambda cond: lambda then_: lambda else_: cond(then_)(else_)

# 테스트
print(IF(TRUE)(lambda: "참")(lambda: "거짓")())   # "참"
print(IF(FALSE)(lambda: "참")(lambda: "거짓")())  # "거짓"
print(IF(AND(TRUE)(FALSE))(lambda: "참")(lambda: "거짓")())  # "거짓"
print(IF(OR(TRUE)(FALSE))(lambda: "참")(lambda: "거짓")())   # "참"
```

### 처치 숫자 (Church Numerals)

자연수 `n`은 "함수 `f`를 `n`번 적용하는 함수"로 인코딩한다.

```
0 = λf.λx.x          -- f를 0번 적용 → x 그대로
1 = λf.λx.f x        -- f를 1번 적용
2 = λf.λx.f (f x)    -- f를 2번 적용
3 = λf.λx.f (f (f x))-- f를 3번 적용

SUCC = λn.λf.λx.f (n f x)         -- 후계자 함수
ADD  = λm.λn.λf.λx.m f (n f x)    -- 덧셈
MULT = λm.λn.λf. m (n f)           -- 곱셈
```

Python으로 처치 숫자를 구현하고 실제 숫자로 변환해보자:

```python
# 처치 숫자 구현
ZERO  = lambda f: lambda x: x
ONE   = lambda f: lambda x: f(x)
TWO   = lambda f: lambda x: f(f(x))
THREE = lambda f: lambda x: f(f(f(x)))

# 처치 숫자를 파이썬 int로 변환
def church_to_int(n):
    return n(lambda x: x + 1)(0)

# 후계자: SUCC(n) = n+1
SUCC = lambda n: lambda f: lambda x: f(n(f)(x))

# 덧셈: ADD(m)(n) = m+n
ADD = lambda m: lambda n: lambda f: lambda x: m(f)(n(f)(x))

# 곱셈: MULT(m)(n) = m*n
MULT = lambda m: lambda n: lambda f: m(n(f))

# 거듭제곱: POW(m)(n) = m^n
POW = lambda m: lambda n: n(m)

# 검증
print(church_to_int(ZERO))               # 0
print(church_to_int(SUCC(TWO)))          # 3
print(church_to_int(ADD(TWO)(THREE)))    # 5
print(church_to_int(MULT(TWO)(THREE)))   # 6
print(church_to_int(POW(TWO)(THREE)))    # 8

# 동적으로 처치 숫자 생성
def int_to_church(n):
    if n == 0:
        return ZERO
    return SUCC(int_to_church(n - 1))

TEN = int_to_church(10)
print(church_to_int(MULT(TEN)(TEN)))     # 100
```

### 처치 페어 (Church Pair)

튜플도 함수로 인코딩 가능하다:

```
PAIR  = λx.λy.λf.f x y     -- 쌍 생성
FST   = λp.p (λx.λy.x)     -- 첫 번째 원소 추출
SND   = λp.p (λx.λy.y)     -- 두 번째 원소 추출
```

```python
PAIR = lambda x: lambda y: lambda f: f(x)(y)
FST  = lambda p: p(lambda x: lambda y: x)
SND  = lambda p: p(lambda x: lambda y: y)

point = PAIR(3)(4)
print(FST(point))  # 3
print(SND(point))  # 4
```

---

## Y 결합자: 재귀의 마법

람다 칼큘러스에는 이름이 없으므로 일반적인 재귀(`fib(n-1)` 처럼 자기 자신을 이름으로 부르는 것)가 불가능하다. 이를 해결하는 것이 **Y 결합자(Y Combinator)**다.

```
Y = λf.(λx.f (x x)) (λx.f (x x))
```

Y 결합자를 `g`에 적용하면:
```
Y g  →  (λx.g (x x)) (λx.g (x x))
     →  g ((λx.g (x x)) (λx.g (x x)))
     →  g (Y g)
```

즉 `Y g = g (Y g)`: 재귀의 불동점이 된다. Python에서는 재귀 호출을 방지하기 위해 **Z 결합자(엄격 언어용 Y 결합자)**를 사용한다:

```python
# Z 결합자 (엄격 평가 언어용)
Z = lambda f: (lambda x: f(lambda v: x(x)(v)))(lambda x: f(lambda v: x(x)(v)))

# 팩토리얼을 이름 없이 재귀로 구현
factorial = Z(lambda self: lambda n: 1 if n == 0 else n * self(n - 1))

print(factorial(5))   # 120
print(factorial(10))  # 3628800

# 피보나치를 이름 없이 재귀로 구현
fibonacci = Z(lambda self: lambda n: n if n <= 1 else self(n - 1) + self(n - 2))

print([fibonacci(i) for i in range(10)])  # [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```

---

## Haskell로 보는 람다 칼큘러스

Haskell은 람다 칼큘러스와 가장 직접적으로 대응되는 언어다:

```haskell
-- 처치 숫자 타입 정의
type Church = forall a. (a -> a) -> a -> a

zero :: Church
zero = \f -> \x -> x

one :: Church
one = \f -> \x -> f x

-- 후계자
succ' :: Church -> Church
succ' n = \f -> \x -> f (n f x)

-- 덧셈
add :: Church -> Church -> Church
add m n = \f -> \x -> m f (n f x)

-- 처치 숫자를 Int로 변환
toInt :: Church -> Int
toInt n = n (+1) 0

-- Y 결합자 (Haskell에서는 타입 검사 통과를 위해 newtype 래핑 필요)
newtype Fix f = Fix { unFix :: f (Fix f) }

y :: (a -> a) -> a
y f = let x = f x in x  -- Haskell의 lazy evaluation 덕분에 직접 가능

-- 재귀 없이 팩토리얼
factorial :: Integer -> Integer
factorial = y (\self n -> if n == 0 then 1 else n * self (n - 1))
```

---

## 주의사항 및 실무 팁

### 1. 정규형 전략 (Normal Form Strategy)

β-환원을 어떤 순서로 적용하느냐에 따라 결과가 달라지거나 무한 루프에 빠질 수 있다.

- **최좌단-최외측(Normal Order)**: 가장 바깥쪽 redex부터 환원. 결과가 존재하면 반드시 찾는다 → Haskell의 **지연 평가(lazy evaluation)**의 이론적 기반
- **최좌단-최내측(Applicative Order)**: 인수를 먼저 완전히 환원 후 적용 → 대부분의 언어(Python, Java)의 **엄격 평가(eager evaluation)**

```python
# 무한 루프를 일으키는 표현식
# 최외측 환원: (λx.5) (발산하는 표현식) → 5 (종료)
# 최내측 환원: 인수를 먼저 평가 → 무한 루프

# Python은 엄격 평가이므로 아래는 무한 루프 발생
import sys
sys.setrecursionlimit(100)

def const_five(x):
    return 5

# 엄격 언어에서는 인수를 먼저 계산하려 시도 → 오류
def diverge():
    return diverge()  # 무한 재귀

# const_five(diverge())  # RecursionError!

# 지연 평가로 해결
def lazy_const_five(x_thunk):
    return 5  # x_thunk를 호출하지 않음

print(lazy_const_five(lambda: diverge()))  # 5 (정상 종료)
```

### 2. 자유 변수와 바인딩

β-환원 시 변수 포획(Variable Capture) 문제를 주의해야 한다:

```
(λx.λy.x) y  →  λy.y  (잘못된 환원! y가 포획됨)
```

올바른 처리: α-변환으로 충돌 방지
```
(λx.λy.x) y  →  (α) (λx.λz.x) y  →  λz.y  (올바른 결과)
```

### 3. De Bruijn 인덱스

실제 인터프리터 구현 시 변수명 대신 **숫자 인덱스**를 사용해 포획 문제를 원천 차단한다:

```
λx.λy.x   →  λ.λ.1   (1은 "한 단계 밖의 바인더")
λx.λy.y   →  λ.λ.0   (0은 "현재 가장 가까운 바인더")
λx.x x    →  λ.0 0
```

이 표현 방식은 GHC Core, LLVM IR 등 컴파일러 내부 표현에서 실제로 사용된다.

---

## 정리

| 개념 | 람다 칼큘러스 표현 | 현대 언어 대응 |
|------|------------------|--------------|
| `true` | `λt.λf.t` | `True`, `true` |
| `0` | `λf.λx.x` | `0` |
| `SUCC` | `λn.λf.λx.f(n f x)` | `n + 1` |
| `Y` | `λf.(λx.f(x x))(λx.f(x x))` | 재귀 함수 |
| α-변환 | `λx.x ≡ λy.y` | 변수 이름 변경 |
| β-환원 | `(λx.e) v → e[x:=v]` | 함수 호출 |
| η-환원 | `λx.(f x) ≡ f` | 포인트-프리 스타일 |

람다 칼큘러스는 프로그래밍 언어 이론의 가장 단순하고도 강력한 기반이다. 처치 인코딩을 직접 구현해보는 과정을 통해, 데이터와 제어 흐름이 모두 함수의 합성으로 환원될 수 있다는 함수형 프로그래밍의 본질을 체득할 수 있다.

## 참고 자료
- [Lambda calculus - Wikipedia](https://en.wikipedia.org/wiki/Lambda_calculus)
- [Church encoding - Wikipedia](https://en.wikipedia.org/wiki/Church_encoding)
- [Lecture Notes on the Lambda Calculus - Peter Selinger](https://arxiv.org/abs/0804.3434)
- [Y combinator - Wikipedia](https://en.wikipedia.org/wiki/Fixed-point_combinator)
