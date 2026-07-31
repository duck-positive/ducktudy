---
layout: post
title: "Hindley-Milner 타입 추론 알고리즘: 타입 어노테이션 없이도 타입을 아는 컴파일러의 비밀"
date: 2026-07-31
categories: [cs, computer-science]
tags: [type-inference, hindley-milner, algorithm-w, unification, type-system, functional-programming, haskell, ocaml, rust]
---

Haskell, OCaml, Standard ML과 같은 함수형 언어에서는 변수 타입을 명시적으로 선언하지 않아도 컴파일러가 타입을 자동으로 추론한다. 심지어 다음 OCaml 코드는 타입 어노테이션이 전혀 없지만 컴파일러는 `add`의 타입이 `int -> int -> int`임을 정확히 안다:

```ocaml
let add x y = x + y
```

이 마법 같은 기능의 이론적 토대가 바로 **Hindley-Milner(HM) 타입 시스템**이다. 1978년 Roger Hindley와 Robin Milner가 독립적으로 발표한 이 알고리즘은 현대 함수형 프로그래밍 언어 타입 시스템의 근간이며, Rust의 타입 추론에도 깊은 영향을 미쳤다.

## 왜 타입 추론이 필요한가

강타입 언어에서 모든 식에 타입을 명시하면 코드가 장황해진다. 반면 타입을 전혀 확인하지 않으면 런타임 오류가 발생한다. **타입 추론**은 두 세계의 장점을 결합한다: 프로그래머는 어노테이션을 생략할 수 있고, 컴파일러는 여전히 정적 타입 안전성을 보장한다.

HM 타입 시스템의 핵심 목표는 모든 표현식에 대해 **주요 타입(principal type)**, 즉 가장 일반적인 타입을 추론하는 것이다. 이는 항상 유일하게 존재하며 완전 자동으로 결정될 수 있다.

## HM 타입 시스템의 구성 요소

### 타입의 종류

HM 타입 시스템에서 타입은 두 층으로 구분된다:

- **단형 타입(Monotype, τ)**: 타입 변수 `α`, 기본 타입 `Int`, `Bool`, 혹은 함수 타입 `τ₁ → τ₂`
- **다형 타입(Polytype, σ)**: 전칭 한정자가 붙은 타입 `∀α. τ` (예: `∀α. α → α`)

`id x = x`의 주요 타입은 `∀α. α → α`이다. 어떤 타입 `T`에 대해서도 `T → T`로 인스턴스화할 수 있다.

### 단일화(Unification)

HM 추론의 심장은 **단일화 알고리즘(Unification Algorithm)**이다. 단일화는 두 타입 식을 동일하게 만드는 **타입 치환(substitution)**을 찾는다. 타입 치환 `S`는 타입 변수를 구체적인 타입으로 매핑하는 함수다.

예를 들어 `α → Int`와 `Bool → β`를 단일화하려면 `{α ↦ Bool, β ↦ Int}`라는 치환을 찾아야 한다.

## Algorithm W — 핵심 추론 알고리즘

Algorithm W는 Robin Milner가 1978년 논문에서 제시한 HM 추론의 표준 구현이다. 표현식을 재귀적으로 분석하며 타입 변수를 생성하고, 제약 조건을 단일화하여 타입을 결정한다.

### Python으로 구현하는 간단한 HM 타입 추론기

다음은 HM 타입 추론의 핵심인 단일화 알고리즘과 타입 변수 생성 메커니즘을 Python으로 구현한 예제다:

```python
from dataclasses import dataclass, field
from typing import Optional

# 타입 표현
@dataclass
class TVar:
    """타입 변수: α, β, γ ..."""
    name: str
    def __repr__(self): return self.name

@dataclass
class TCon:
    """타입 상수: Int, Bool, String ..."""
    name: str
    def __repr__(self): return self.name

@dataclass
class TFun:
    """함수 타입: τ₁ → τ₂"""
    param: object
    ret: object
    def __repr__(self): return f"({self.param} -> {self.ret})"

# 전역 타입 변수 카운터
_counter = [0]
def fresh_tvar():
    _counter[0] += 1
    return TVar(f"t{_counter[0]}")

# 치환(Substitution) 적용
def apply_subst(subst, ty):
    if isinstance(ty, TVar):
        return subst.get(ty.name, ty)
    elif isinstance(ty, TCon):
        return ty
    elif isinstance(ty, TFun):
        return TFun(apply_subst(subst, ty.param),
                    apply_subst(subst, ty.ret))

# 치환 합성
def compose_subst(s1, s2):
    result = {k: apply_subst(s1, v) for k, v in s2.items()}
    result.update(s1)
    return result

# 발생 검사(Occurs Check): α가 τ 안에 있는지 확인
def occurs(tvar_name, ty):
    if isinstance(ty, TVar):
        return ty.name == tvar_name
    elif isinstance(ty, TFun):
        return occurs(tvar_name, ty.param) or occurs(tvar_name, ty.ret)
    return False

# 단일화 알고리즘
def unify(t1, t2):
    t1 = apply_subst({}, t1)
    t2 = apply_subst({}, t2)

    if isinstance(t1, TCon) and isinstance(t2, TCon):
        if t1.name == t2.name:
            return {}
        raise TypeError(f"Cannot unify {t1} with {t2}")

    elif isinstance(t1, TVar):
        if isinstance(t2, TVar) and t1.name == t2.name:
            return {}
        if occurs(t1.name, t2):
            raise TypeError(f"Infinite type: {t1} occurs in {t2}")
        return {t1.name: t2}

    elif isinstance(t2, TVar):
        return unify(t2, t1)

    elif isinstance(t1, TFun) and isinstance(t2, TFun):
        s1 = unify(t1.param, t2.param)
        s2 = unify(apply_subst(s1, t1.ret),
                   apply_subst(s1, t2.ret))
        return compose_subst(s2, s1)

    raise TypeError(f"Cannot unify {t1} and {t2}")

# 테스트: (α → β)와 (Int → Bool) 단일화
s = unify(TFun(TVar("a"), TVar("b")),
          TFun(TCon("Int"), TCon("Bool")))
print(s)  # {'a': Int, 'b': Bool}

# 테스트: 발생 검사 (무한 타입 방지)
try:
    unify(TVar("a"), TFun(TVar("a"), TCon("Int")))
except TypeError as e:
    print(e)  # Infinite type: a occurs in (a -> Int)
```

### 표현식 타입 추론기

이제 람다 계산법 기반의 간단한 표현식에 대한 타입 추론기를 구현해보자:

```python
# AST 노드
@dataclass
class Var:
    name: str  # 변수 참조

@dataclass
class Lam:
    param: str  # 람다 파라미터
    body: object  # 람다 본문

@dataclass
class App:
    func: object  # 함수
    arg: object   # 인수

@dataclass
class Lit:
    value: object  # 리터럴 값 (int, bool 등)

# 타입 환경(Type Environment): 변수명 → 타입
def infer(env, expr, subst=None):
    if subst is None:
        subst = {}

    if isinstance(expr, Lit):
        if isinstance(expr.value, int):
            return TCon("Int"), subst
        elif isinstance(expr.value, bool):
            return TCon("Bool"), subst

    elif isinstance(expr, Var):
        if expr.name not in env:
            raise TypeError(f"Unbound variable: {expr.name}")
        return apply_subst(subst, env[expr.name]), subst

    elif isinstance(expr, Lam):
        # 파라미터에 새 타입 변수 할당
        param_ty = fresh_tvar()
        new_env = {**env, expr.param: param_ty}
        body_ty, s1 = infer(new_env, expr.body, subst)
        return TFun(apply_subst(s1, param_ty), body_ty), s1

    elif isinstance(expr, App):
        func_ty, s1 = infer(env, expr.func, subst)
        arg_ty, s2 = infer(env, expr.arg, compose_subst(s1, subst))
        ret_ty = fresh_tvar()
        s3 = unify(apply_subst(s2, func_ty), TFun(arg_ty, ret_ty))
        final_subst = compose_subst(s3, compose_subst(s2, s1))
        return apply_subst(final_subst, ret_ty), final_subst

# 테스트 1: \x -> x (항등 함수)
identity = Lam("x", Var("x"))
ty, _ = infer({}, identity)
print(f"\\x -> x : {ty}")  # (t1 -> t1)

# 테스트 2: \x -> x + 1 (정수 함수, + 는 Int -> Int -> Int)
env = {"add": TFun(TCon("Int"), TFun(TCon("Int"), TCon("Int")))}
add_one = Lam("x", App(App(Var("add"), Var("x")), Lit(1)))
ty, _ = infer(env, add_one)
print(f"\\x -> add x 1 : {ty}")  # (Int -> Int)
```

## Hindley-Milner의 다형성: let-다형성

HM의 강력함은 **let 표현식을 통한 다형성**에 있다. `let f = \x -> x in ...`에서 `f`는 `∀α. α → α` 타입으로 일반화(generalize)된다. 이후 `f 1`은 `Int → Int`로, `f True`는 `Bool → Bool`로 개별 인스턴스화된다.

이것이 바로 Haskell에서 `map :: (a -> b) -> [a] -> [b]`처럼 모든 타입에 대해 동작하는 함수를 타입 어노테이션 없이 작성할 수 있는 이유다.

## 실제 언어에서의 HM 타입 추론

**OCaml**: HM 타입 추론을 가장 순수하게 구현한 언어 중 하나. REPL에서 `# let id x = x;;`를 입력하면 `val id : 'a -> 'a = <fun>`을 반환한다.

**Haskell**: HM을 기반으로 **타입 클래스(type class)**를 추가하여 ad-hoc 다형성도 지원한다.

**Rust**: HM에서 영감받았지만 소유권·생명주기 시스템 때문에 순수 HM보다 훨씬 복잡하다. 지역 변수 타입은 추론하지만 함수 시그니처는 명시해야 한다.

**Standard ML**: HM 타입 추론을 최초로 실용화한 언어.

## 주의사항 및 한계

1. **2차 다형성(Rank-2 Polymorphism)은 추론 불가**: `∀α. (∀β. β → β) → α → α`와 같이 전칭 한정자가 중첩되면 주요 타입 추론이 결정 불가능해진다. Haskell의 `RankNTypes` 확장은 이를 허용하지만 명시적 어노테이션이 필요하다.

2. **발생 검사 비용**: `occurs` 검사를 매번 수행하면 `O(n²)`이 될 수 있다. 실제 구현은 Union-Find를 사용해 준선형 시간으로 최적화한다.

3. **오류 메시지 품질**: 순수 Algorithm W는 타입 불일치 오류의 위치를 정확히 가리키기 어렵다. 현대 컴파일러는 제약 기반(constraint-based) 접근법을 사용해 오류 보고를 개선한다.

4. **가변 참조(Mutable Reference) 문제**: HM에 가변 참조가 추가되면 **값 제한(value restriction)**을 도입해야 안전한 타입 추론이 가능하다. 이는 OCaml의 `ref` 처리에서 확인할 수 있다.

5. **시간 복잡도**: Algorithm W는 이론적으로 지수 시간이 가능하지만, 실제 코드에서는 거의 선형에 가까운 성능을 보인다.

## 참고 자료
- [Hindley-Milner type system - Wikipedia](https://en.wikipedia.org/wiki/Hindley%E2%80%93Milner_type_system)
- [A Theory of Type Polymorphism in Programming - Milner, 1978](https://www.sciencedirect.com/science/article/pii/0022000078900144)
- [OCaml Manual - Type Inference](https://v2.ocaml.org/manual/comp.html)
- [Write You a Haskell - Type Inference](http://dev.stephendiehl.com/fun/006_hindley_milner.html)
