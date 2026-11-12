---
layout: post
title: "양방향 타입 검사 완전 정복: Checking 모드와 Synthesis 모드로 구현하는 효율적인 타입 추론"
date: 2026-11-12
categories: [cs, computer-science]
tags: [type-system, type-checking, type-inference, bidirectional, programming-languages, compiler]
---

## 양방향 타입 검사란 무엇인가

타입 추론(type inference)은 현대 프로그래밍 언어에서 없어서는 안 될 기능이다. Haskell, Rust, Kotlin, Swift 등 대부분의 현대 언어는 프로그래머가 모든 곳에 타입을 명시하지 않아도 컴파일러가 스스로 타입을 알아낸다. 이 마법 같은 기능의 핵심 알고리즘 중 하나가 **양방향 타입 검사(Bidirectional Type Checking)**다.

양방향 타입 검사는 Luca Cardelli가 1990년대에 제안하고, 이후 Joshua Dunfield와 Neelakantan Krishnaswami가 2013년 ICFP 논문("Complete and Easy Bidirectional Typechecking for Higher-Rank Polymorphism")에서 체계화한 타입 추론 기법이다. 이 알고리즘은 두 가지 모드로 타입 판단을 나눈다.

- **Checking 모드(⇐)**: "이 표현식이 주어진 타입 `T`를 가지는지 확인하라." — 타입이 이미 알려진 상태에서 위에서 아래로(top-down) 흐른다.
- **Synthesis 모드(⇒)**: "이 표현식의 타입을 직접 추론해서 알려달라." — 타입이 아래에서 위로(bottom-up) 합성된다.

이 두 모드를 번갈아 사용하면서 복잡한 단일화(unification) 과정 없이도 고차 다형성(higher-rank polymorphism)까지 처리할 수 있다는 것이 양방향 타입 검사의 핵심 통찰이다.

## 왜 양방향 타입 검사가 필요한가

### 단순 힌들리-밀너의 한계

Hindley-Milner(HM) 타입 추론은 `let`-다형성을 지원하고 완전한 타입 추론을 제공한다. 그러나 HM에는 다음과 같은 한계가 있다.

1. **고차 다형성 미지원**: `∀a. a → a`를 인자로 받는 함수처럼 타입 변수가 함수 인자 자체에 등장하는 Rank-2 이상의 다형성을 처리하지 못한다.
2. **복잡한 단일화**: HM은 제약(constraint) 생성 후 단일화(unification)를 수행하는데, 이 과정에서 오류 메시지가 불명확해지고 구현이 복잡해진다.
3. **타입 어노테이션 처리 어색**: 사용자가 명시적으로 타입을 적어준 경우 이를 활용하는 방법이 체계적이지 않다.

양방향 타입 검사는 이러한 문제를 **"사용자가 적은 타입 어노테이션을 최대한 활용하여 위·아래 방향으로 정보를 전파한다"**는 발상으로 해결한다.

### 실용적 이점

- **단순한 구현**: 단일화 알고리즘 없이 재귀적인 패턴 매칭만으로 구현 가능하다.
- **좋은 오류 메시지**: 타입이 어느 방향에서 왔는지 명확하므로 불일치가 발생할 때 정확한 위치를 지목할 수 있다.
- **고차 다형성**: Rank-2, Rank-N 다형성을 사용자 어노테이션 도움을 받아 처리할 수 있다.
- **확장성**: 의존 타입(dependent types), 정제 타입(refinement types) 등으로 쉽게 확장된다.

## 핵심 개념: 타입 판단 규칙

양방향 타입 검사의 타입 판단(type judgment)은 다음 두 가지 형식을 사용한다.

```
Γ ⊢ e ⇒ A    (synthesis: 표현식 e의 타입을 환경 Γ에서 추론하면 A이다)
Γ ⊢ e ⇐ A    (checking: 표현식 e가 환경 Γ에서 타입 A를 가지는지 확인한다)
```

주요 규칙 몇 가지를 살펴보자.

**변수 (Var)**: 환경에서 직접 타입을 읽으므로 synthesis 모드에서 처리한다.
```
x : A ∈ Γ
─────────────
Γ ⊢ x ⇒ A
```

**람다 (Lam-Check)**: 함수의 반환 타입이 알려진 경우(checking) 인자를 환경에 추가하고 본문을 checking한다.
```
Γ, x:A ⊢ e ⇐ B
────────────────────
Γ ⊢ λx.e ⇐ A → B
```

**어노테이션 (Anno)**: 명시적 타입 어노테이션은 synthesis와 checking을 연결하는 다리다.
```
Γ ⊢ e ⇐ A
──────────────────
Γ ⊢ (e : A) ⇒ A
```

**Sub (모드 전환)**: synthesis로 얻은 타입을 checking에 사용할 때 서브타입 관계를 확인한다.
```
Γ ⊢ e ⇒ A    A <: B
──────────────────────
Γ ⊢ e ⇐ B
```

## 실제 구현 예제

### 예제 1: Python으로 구현하는 간단한 양방향 타입 검사기

다음은 단순한 람다 계산법에 양방향 타입 검사를 구현한 Python 예제다.

```python
from __future__ import annotations
from dataclasses import dataclass
from typing import Union, Optional

# ─── AST ───────────────────────────────────────────────────────────────
@dataclass
class Var:
    name: str

@dataclass
class Lam:
    param: str
    body: "Expr"

@dataclass
class App:
    func: "Expr"
    arg: "Expr"

@dataclass
class Anno:   # (e : T)
    expr: "Expr"
    ty: "Type"

Expr = Union[Var, Lam, App, Anno]

# ─── 타입 ──────────────────────────────────────────────────────────────
@dataclass(frozen=True)
class TBase:   # Int, Bool 등 기저 타입
    name: str

@dataclass(frozen=True)
class TArrow:  # A → B
    param: "Type"
    ret: "Type"

Type = Union[TBase, TArrow]

# ─── 양방향 타입 검사 ───────────────────────────────────────────────────
Env = dict[str, Type]

def synth(env: Env, expr: Expr) -> Type:
    """Synthesis 모드: 표현식의 타입을 추론"""
    match expr:
        case Var(name):
            if name not in env:
                raise TypeError(f"미정의 변수: {name}")
            return env[name]

        case Anno(e, ty):
            check(env, e, ty)
            return ty

        case App(func, arg):
            func_ty = synth(env, func)
            if not isinstance(func_ty, TArrow):
                raise TypeError(f"함수가 아닌 값에 적용: {func_ty}")
            check(env, arg, func_ty.param)
            return func_ty.ret

        case Lam():
            raise TypeError(
                "람다의 타입은 알 수 없습니다. 어노테이션을 추가하세요: (λx.e : A→B)"
            )

        case _:
            raise TypeError(f"알 수 없는 표현식: {expr}")


def check(env: Env, expr: Expr, ty: Type) -> None:
    """Checking 모드: 표현식이 주어진 타입을 가지는지 확인"""
    match expr, ty:
        case Lam(param, body), TArrow(param_ty, ret_ty):
            # 람다는 checking 모드에서만 처리 가능!
            new_env = {**env, param: param_ty}
            check(new_env, body, ret_ty)

        case _:
            # 그 외는 synthesis 후 타입 비교(서브타이핑 생략)
            inferred = synth(env, expr)
            if inferred != ty:
                raise TypeError(
                    f"타입 불일치: 기대 {ty}, 실제 {inferred}"
                )


# ─── 테스트 ─────────────────────────────────────────────────────────────
Int = TBase("Int")
Bool = TBase("Bool")

# (λx.x : Int → Int)  — 어노테이션이 있으므로 성공
id_int = Anno(Lam("x", Var("x")), TArrow(Int, Int))
result = synth({}, id_int)
print(f"id_int 타입: {result}")   # TArrow(TBase('Int'), TBase('Int'))

# (not : Bool → Bool) (true : Bool)  — 함수 적용
env = {
    "not": TArrow(Bool, Bool),
    "true": Bool,
}
app_expr = App(Var("not"), Var("true"))
result2 = synth(env, app_expr)
print(f"not true 타입: {result2}")  # TBase('Bool')
```

### 예제 2: Haskell로 보는 고차 다형성(Rank-2)

표준 HM으로는 처리할 수 없는 Rank-2 다형성 예제를 Haskell 타입 시스템과 함께 살펴보자.

```haskell
{-# LANGUAGE RankNTypes #-}

-- Rank-2 함수: 인자 자체가 다형적인 함수를 받는다
applyTwice :: (forall a. a -> a) -> (Int, Bool)
applyTwice f = (f 42, f True)

-- 올바른 사용: id는 'forall a. a -> a'를 만족한다
result :: (Int, Bool)
result = applyTwice id    -- (42, True)

-- 양방향 타입 검사가 없다면?
-- GHC는 applyTwice에 명시된 (forall a. a -> a) 타입을
-- checking 모드로 인자에 전파하여 검사한다.
-- 이것이 없으면 HM 단일화만으로는 처리 불가능!

-- 잘못된 사용: 단형 함수는 전달 불가
-- applyTwice (+1)  -- ERROR: 타입 불일치

-- 양방향 검사를 명시적으로 활용하는 패턴
-- 반환 타입을 어노테이션으로 고정하면
-- 그 타입이 본문 checking에 흘러 들어간다
safeDivide :: Int -> Int -> Maybe Int
safeDivide _ 0 = Nothing      -- checking: Maybe Int ⇐ Nothing
safeDivide x y = Just (x `div` y)  -- checking: Maybe Int ⇐ Just ...

-- 레코드 타입에서 양방향 검사의 위력
data Config = Config
    { port    :: Int
    , host    :: String
    , debug   :: Bool
    }

-- 타입이 위에서 checking 모드로 흘러내려
-- 각 필드의 리터럴 타입을 별도 추론 없이 처리한다
defaultConfig :: Config
defaultConfig = Config
    { port  = 8080         -- checking: Int ⇐ 8080
    , host  = "localhost"  -- checking: String ⇐ "localhost"
    , debug = False        -- checking: Bool ⇐ False
    }
```

## 주의사항과 실전 팁

### 1. Synthesis 불가 표현식에 어노테이션 필수

람다 표현식(`λx.e`)은 단독으로는 synthesis가 불가능하다. 고차 함수 인자로 전달할 때 타입이 맥락에서 propagate되지 않는다면 반드시 명시적 타입 어노테이션을 붙여야 한다.

```python
# 잘못된 예: 람다 단독으로는 타입 추론 불가
# result = synth({}, Lam("x", Var("x")))  # TypeError!

# 올바른 예: 어노테이션으로 타입 명시
result = synth({}, Anno(Lam("x", Var("x")), TArrow(Int, Int)))
```

### 2. 모드 전환(Sub)과 서브타입

Checking과 synthesis를 연결하는 Sub 규칙에서 단순 등가 비교(A = B) 대신 서브타입 관계(A <: B)를 사용하면 공변/반변 처리, 다형성, 구조적 서브타이핑을 함께 다룰 수 있다. 실용적인 구현에서는 이 부분을 별도 모듈로 분리하는 것이 좋다.

### 3. 완전성(completeness) 보장의 범위

Dunfield & Krishnaswami(2013)의 알고리즘은 DK 시스템으로도 불리며, Rank-N 다형성까지 완전하고 건전(sound)하다. 그러나 전체 의존 타입이나 무한 타입에 대해서는 별도 확장이 필요하다.

### 4. 오류 메시지를 위한 위치 추적

양방향 타입 검사기를 구현할 때는 각 표현식에 소스 위치(span) 정보를 함께 저장하고, 오류 발생 시 "어떤 모드(checking/synthesis)에서, 어떤 표현식의, 어떤 위치에서" 불일치가 발생했는지 풍부한 오류 메시지를 출력하도록 설계하면 사용자 경험이 크게 향상된다.

### 5. 타입스크립트와 현대 언어에서의 적용

TypeScript 컴파일러도 맥락적 타입(contextual typing)이라는 이름으로 양방향 타입 검사의 아이디어를 활용한다. 콜백 함수의 매개변수 타입이 자동으로 추론되는 것이 그 대표적인 예다.

```typescript
// checking 모드: arr.map의 콜백 타입 (n: number) => number 로 흘러내림
const doubled = [1, 2, 3].map(n => n * 2);
//                          ^─ n의 타입이 number로 자동 추론됨
```

## 참고 자료

- [Complete and Easy Bidirectional Typechecking for Higher-Rank Polymorphism (Dunfield & Krishnaswami, ICFP 2013)](https://arxiv.org/abs/1306.6032)
- [Bidirectional Typing (Dunfield & Krishnaswami, ACM Computing Surveys, 2021)](https://arxiv.org/abs/1908.05839)
- [Lecture Notes on Bidirectional Type Checking — CMU 15-312](https://www.cs.cmu.edu/~fp/courses/15312-f04/handouts/15-bidirectional.pdf)
- [GHC User's Guide: RankNTypes Extension](https://downloads.haskell.org/ghc/latest/docs/users_guide/exts/rank_polymorphism.html)
