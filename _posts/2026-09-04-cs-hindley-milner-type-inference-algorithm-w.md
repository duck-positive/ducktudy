---
layout: post
title: "Hindley-Milner 타입 추론 완전 정복: Algorithm W로 이해하는 정적 타입 언어의 수학적 토대"
date: 2026-09-04
categories: [cs, computer-science]
tags: [type-inference, hindley-milner, algorithm-w, type-theory, programming-languages, compiler]
---

## 타입 추론이란 무엇인가

정적 타입 언어(Haskell, OCaml, Rust, F#)는 프로그래머가 모든 변수에 타입 어노테이션을 달지 않아도 컴파일러가 올바른 타입을 자동으로 결정합니다. 이 마법 같은 능력의 수학적 토대가 바로 **Hindley-Milner(HM) 타입 시스템**과 그 추론 알고리즘인 **Algorithm W**입니다.

```haskell
-- Haskell: 타입 어노테이션 없이도 컴파일러가 타입을 추론
identity x = x          -- 컴파일러: identity :: a -> a
add x y = x + y         -- 컴파일러: add :: Num a => a -> a -> a
map f []     = []        -- 컴파일러: map :: (a -> b) -> [a] -> [b]
map f (x:xs) = f x : map f xs
```

1969년 Roger Hindley와 1978년 Robin Milner가 독립적으로 발견한 이 타입 시스템은 **완전성(completeness)** 을 보장합니다. 타입이 존재한다면 반드시 추론하고, 타입이 없다면 반드시 오류를 보고합니다. 즉, 타입 어노테이션이 전혀 없어도 프로그램의 올바른 가장 일반적인 타입(principal type)을 항상 찾아냅니다.

---

## 왜 HM 타입 추론이 중요한가

### 개발자 경험과 안전성의 양립

타입 어노테이션 없는 동적 언어(Python, JavaScript)는 유연하지만 런타임 오류가 발생하기 쉽습니다. 반면 Java처럼 모든 곳에 타입을 명시해야 하는 언어는 안전하지만 번거롭습니다. HM 타입 추론은 이 두 세계의 장점을 결합합니다.

- **안전성**: 정적 타입의 강력한 오류 사전 감지
- **간결성**: 어노테이션 없이도 자동 추론
- **다형성**: 하나의 함수가 여러 타입에서 동작하는 파라메트릭 다형성

### 현대 언어에서의 영향

HM 타입 시스템의 아이디어는 수많은 현대 언어에 직접적으로 영향을 주었습니다.

- **Rust**: `let x = 5;`에서 `x: i32` 자동 추론
- **TypeScript**: `const arr = [1, 2, 3]`에서 `number[]` 추론
- **Kotlin**: `val x = "hello"`에서 `String` 추론
- **Swift**: `let y = 3.14`에서 `Double` 추론
- **Haskell, OCaml, F#**: HM의 완전한 구현

---

## 핵심 개념: 유니피케이션과 타입 변수

HM 추론의 핵심은 두 가지입니다.

**타입 변수 (Type Variables)**: 아직 결정되지 않은 타입을 나타내는 변수 `α`, `β`, `γ` 등입니다. `identity x = x`에서 `x`의 타입은 처음에 `α`라는 타입 변수로 설정됩니다.

**유니피케이션 (Unification)**: 두 타입을 같게 만드는 치환(substitution)을 찾는 과정입니다. `α → Int`라는 제약이 있으면 `α`를 `Int`로 치환합니다. 서로 다른 구체 타입(`Int`와 `Bool`)을 같게 만들 수는 없어서 타입 오류가 발생합니다.

**일반화와 인스턴스화**: `let f = identity`처럼 다형 함수를 다른 이름으로 바인딩하면, 타입 변수를 **양화(quantify)**하여 스킴(scheme) `∀α. α → α`를 만듭니다(일반화). 이 함수를 `Int`에 적용하면 `α`를 `Int`로 치환하는 인스턴스화가 발생합니다.

---

## 실제 구현 예제

### 예제 1: 유니피케이션 알고리즘 구현

```python
from dataclasses import dataclass, field
from typing import Optional, Dict

# 타입 표현
@dataclass
class TypeVar:
    name: str
    def __repr__(self): return self.name

@dataclass
class TypeCon:  # 기본 타입 (Int, Bool 등)
    name: str
    def __repr__(self): return self.name

@dataclass
class TypeApp:  # 타입 적용 (예: List Int, a -> b)
    func: object
    arg: object
    def __repr__(self): return f"({self.func} {self.arg})"

def make_arrow(t1, t2):
    return TypeApp(TypeApp(TypeCon("->"), t1), t2)

# 치환(Substitution): 타입 변수 → 타입
Subst = Dict[str, object]

def apply_subst(subst: Subst, ty: object) -> object:
    """타입에 치환을 적용"""
    if isinstance(ty, TypeVar):
        return apply_subst(subst, subst[ty.name]) if ty.name in subst else ty
    elif isinstance(ty, TypeCon):
        return ty
    elif isinstance(ty, TypeApp):
        return TypeApp(apply_subst(subst, ty.func), apply_subst(subst, ty.arg))
    return ty

def compose_subst(s1: Subst, s2: Subst) -> Subst:
    """두 치환의 합성: s2를 s1에 적용한 뒤 s1과 합침"""
    result = {k: apply_subst(s1, v) for k, v in s2.items()}
    result.update(s1)
    return result

def free_vars(ty: object) -> set:
    """타입 내의 자유 타입 변수 집합"""
    if isinstance(ty, TypeVar):
        return {ty.name}
    elif isinstance(ty, TypeCon):
        return set()
    elif isinstance(ty, TypeApp):
        return free_vars(ty.func) | free_vars(ty.arg)
    return set()

def var_bind(name: str, ty: object) -> Subst:
    """타입 변수 name을 ty에 바인딩 (occurs check 포함)"""
    if isinstance(ty, TypeVar) and ty.name == name:
        return {}  # 같은 변수면 빈 치환
    if name in free_vars(ty):
        raise TypeError(f"Occurs check 실패: {name} occurs in {ty}")
    return {name: ty}

def unify(t1: object, t2: object) -> Subst:
    """두 타입을 통합하는 치환 계산 (Most General Unifier)"""
    if isinstance(t1, TypeCon) and isinstance(t2, TypeCon):
        if t1.name == t2.name:
            return {}
        raise TypeError(f"타입 불일치: {t1} vs {t2}")
    elif isinstance(t1, TypeVar):
        return var_bind(t1.name, t2)
    elif isinstance(t2, TypeVar):
        return var_bind(t2.name, t1)
    elif isinstance(t1, TypeApp) and isinstance(t2, TypeApp):
        s1 = unify(t1.func, t2.func)
        s2 = unify(apply_subst(s1, t1.arg), apply_subst(s1, t2.arg))
        return compose_subst(s2, s1)
    else:
        raise TypeError(f"타입 통합 불가: {t1} vs {t2}")


# 테스트
INT = TypeCon("Int")
BOOL = TypeCon("Bool")
a, b, c = TypeVar("a"), TypeVar("b"), TypeVar("c")

# 예시 1: a → Int 와 Bool → b 통합
# → a = Bool, b = Int
subst = unify(make_arrow(a, INT), make_arrow(BOOL, b))
print("통합 결과:", subst)  # {'a': Bool, 'b': Int}

# 예시 2: (a → a) 와 (Int → Int) 통합
subst2 = unify(make_arrow(a, a), make_arrow(INT, INT))
print("통합 결과:", subst2)  # {'a': Int}

# 예시 3: occurs check — a 와 (a → Int) 통합 불가
try:
    unify(a, make_arrow(a, INT))
except TypeError as e:
    print("오류:", e)  # Occurs check 실패
```

### 예제 2: 간소화된 Algorithm W 구현

```python
_fresh_counter = 0

def fresh_var() -> TypeVar:
    global _fresh_counter
    _fresh_counter += 1
    return TypeVar(f"t{_fresh_counter}")

@dataclass
class Scheme:
    """다형 타입 스킴: ∀[vars]. type"""
    vars: list
    ty: object
    def __repr__(self): return f"∀{self.vars}.{self.ty}" if self.vars else str(self.ty)

# 환경: 변수명 → 타입 스킴
Env = Dict[str, Scheme]

def generalize(env: Env, ty: object) -> Scheme:
    """환경에 없는 자유 변수를 전칭 양화"""
    env_fvs = set()
    for scheme in env.values():
        env_fvs |= free_vars(scheme.ty) - set(scheme.vars)
    ty_fvs = free_vars(ty)
    quantified = list(ty_fvs - env_fvs)
    return Scheme(quantified, ty)

def instantiate(scheme: Scheme) -> object:
    """스킴의 전칭 변수를 신선한 타입 변수로 치환"""
    subst = {v: fresh_var() for v in scheme.vars}
    return apply_subst(subst, scheme.ty)

def free_vars_env(env: Env) -> set:
    result = set()
    for s in env.values():
        result |= free_vars(s.ty) - set(s.vars)
    return result

def infer(env: Env, expr) -> tuple:
    """Algorithm W: (치환, 타입) 반환"""
    if isinstance(expr, str):
        # 변수
        if expr not in env:
            raise TypeError(f"정의되지 않은 변수: {expr}")
        ty = instantiate(env[expr])
        return {}, ty

    elif isinstance(expr, int):
        return {}, TypeCon("Int")

    elif isinstance(expr, bool):
        return {}, TypeCon("Bool")

    elif isinstance(expr, tuple) and expr[0] == "lam":
        # λx.body
        _, x, body = expr
        tv = fresh_var()
        new_env = {**env, x: Scheme([], tv)}
        s, t = infer(new_env, body)
        return s, make_arrow(apply_subst(s, tv), t)

    elif isinstance(expr, tuple) and expr[0] == "app":
        # f arg
        _, f_expr, arg_expr = expr
        s1, t1 = infer(env, f_expr)
        s2, t2 = infer({k: Scheme(v.vars, apply_subst(s1, v.ty)) for k, v in env.items()}, arg_expr)
        tv = fresh_var()
        s3 = unify(apply_subst(s2, t1), make_arrow(t2, tv))
        return compose_subst(s3, compose_subst(s2, s1)), apply_subst(s3, tv)

    elif isinstance(expr, tuple) and expr[0] == "let":
        # let x = e1 in e2
        _, x, e1, e2 = expr
        s1, t1 = infer(env, e1)
        env1 = {k: Scheme(v.vars, apply_subst(s1, v.ty)) for k, v in env.items()}
        scheme = generalize(env1, t1)
        new_env = {**env1, x: scheme}
        s2, t2 = infer(new_env, e2)
        return compose_subst(s2, s1), t2

    raise TypeError(f"알 수 없는 표현식: {expr}")


# 테스트
base_env: Env = {
    "add": Scheme([], make_arrow(TypeCon("Int"), make_arrow(TypeCon("Int"), TypeCon("Int")))),
    "iszero": Scheme([], make_arrow(TypeCon("Int"), TypeCon("Bool"))),
}

# identity 함수: λx. x → ∀a. a → a
s, ty = infer(base_env, ("lam", "x", "x"))
scheme = generalize(base_env, ty)
print(f"identity: {scheme}")  # ∀[t1]. (-> t1 t1)

# let id = λx.x in id 5  → Int
expr = ("let", "id", ("lam", "x", "x"), ("app", "id", 5))
s, ty = infer(base_env, expr)
print(f"let id = λx.x in id 5: {apply_subst(s, ty)}")  # Int

# add 함수 적용: add 3 → Int → Int
s, ty = infer(base_env, ("app", "add", 3))
print(f"add 3: {apply_subst(s, ty)}")  # (-> Int Int)

# 타입 오류: iszero를 (λx.x)에 적용 — Bool에 Int 연산 불가 확인
try:
    s, ty = infer(base_env, ("app", ("lam", "x", ("app", "iszero", "x")), True))
    print(f"iszero True: {apply_subst(s, ty)}")
except TypeError as e:
    print(f"타입 오류 감지: {e}")
```

---

## Let 다형성의 중요성

HM의 핵심 중 하나는 **let 바인딩에서의 다형성**입니다. 람다 파라미터는 단형(monomorphic)이지만, let 바인딩은 다형(polymorphic)입니다.

```haskell
-- 가능: let을 통한 다형성
let id = \x -> x
in  (id 5, id True)   -- (Int, Bool)

-- 불가: 람다 파라미터는 단형
(\f -> (f 5, f True)) (\x -> x)   -- 타입 오류!
```

위 코드에서 `\f -> (f 5, f True)`는 f에 `Int → ?`와 `Bool → ?`를 동시에 요구하므로 타입 오류가 발생합니다. 반면 let 바인딩은 id를 `∀a. a → a`로 일반화하므로 여러 타입에 인스턴스화할 수 있습니다.

---

## 주의사항 및 한계

### 결정 불가능한 확장들

순수 HM은 결정 가능(decidable)하지만, 일부 확장은 결정 불가능합니다.

- **Rank-N 다형성**: `∀α. (∀β. β → β) → α → α` 형태. Haskell의 `RankNTypes` 확장에서 지원하지만 추론 불가, 어노테이션 필요
- **GADTs (Generalized Algebraic Data Types)**: 더 표현력 있는 타입을 위해 어노테이션 필요
- **타입 클래스/트레이트**: Haskell의 타입 클래스, Rust의 트레이트는 HM을 확장한 Damas-Hindley-Milner-Wadler-Blott 시스템에 기반

### Occurs Check의 성능

나이브한 유니피케이션은 Occurs Check(타입 변수가 자기 자신을 포함하는지 확인)로 인해 O(n²) 복잡도를 가집니다. 실제 컴파일러는 Union-Find 자료구조를 사용하여 거의 O(n)에 가깝게 최적화합니다.

### let 표현식 폭발 문제

`let f = e in f f f ... f` 패턴에서 f가 n번 사용되면, 각 사용처에서 인스턴스화가 일어나 타입 검사 시간이 지수적으로 증가할 수 있습니다. 실제 컴파일러는 이 문제를 여러 방식으로 완화합니다.

---

## 참고 자료

- [Hindley-Milner Type System - Wikipedia](https://en.wikipedia.org/wiki/Hindley%E2%80%93Milner_type_system)
- [Damas-Hindley-Milner Inference - Max Bernstein](https://bernsteinbear.com/blog/type-inference/)
- [Type Inference Lecture Notes - CS4410 Northeastern](https://course.ccs.neu.edu/cs4410sp19/lec_type-inference_notes.html)
- [Principal Type-Schemes for Functional Programs - Damas & Milner (1982)](https://dl.acm.org/doi/10.1145/582153.582176)
