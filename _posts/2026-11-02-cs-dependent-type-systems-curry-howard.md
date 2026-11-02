---
layout: post
title: "의존형 타입 시스템(Dependent Type System) 완전 정복: Curry-Howard 대응으로 증명이 곧 프로그램이 되는 세계"
date: 2026-11-02
categories: [cs, computer-science]
tags: [dependent-types, curry-howard, type-theory, agda, idris, proof-assistant, type-system, formal-verification]
---

"타입은 명세(specification)이고 프로그램은 증명(proof)이다."

이 놀라운 주장이 **Curry-Howard 대응(Curry-Howard Correspondence)**의 핵심이다. 1934년 Haskell Curry와 1969년 William Howard가 독립적으로 발견한 이 대응 관계는, 논리학의 명제(proposition)와 프로그래밍 언어의 타입(type)이 정확히 같은 구조를 가진다는 것을 보인다. 그리고 이를 극한까지 밀고 나간 것이 **의존형 타입 시스템(Dependent Type System)**이다.

## 왜 의존형 타입이 필요한가

일반적인 정적 타입 시스템 — Java, Haskell, Rust 등 — 은 많은 것을 보장하지만 한계가 있다.

```haskell
-- Haskell에서 이 함수의 타입 시그니처
head :: [a] -> a
```

이 시그니처만 보면 `head`가 안전하게 보인다. 그러나 빈 리스트를 넘기면 런타임 에러가 발생한다. 타입 시스템이 "리스트가 비어있지 않아야 한다"는 조건을 표현할 수 없기 때문이다.

의존형 타입은 이 문제를 해결한다. **타입이 값에 의존**할 수 있어서, 예를 들어 "길이가 n인 벡터"를 타입으로 표현하고, `head`가 "길이가 n+1인 벡터"를 받아야 한다고 타입 수준에서 강제할 수 있다.

```
-- Agda 스타일 (의사 코드)
head : Vect (n + 1) a → a
```

이제 빈 벡터를 넘기면 **컴파일 타임에** 타입 오류가 발생한다. 런타임 에러 없이.

---

## Curry-Howard 대응: 명제 = 타입, 증명 = 프로그램

### 대응 관계

| 논리학 | 타입 이론 |
|--------|-----------|
| 명제 P | 타입 A |
| P의 증명 | 타입 A의 항(term) |
| P → Q (함의) | 함수 타입 A → B |
| P ∧ Q (연언) | 곱 타입 A × B |
| P ∨ Q (선언) | 합 타입 A + B |
| ⊥ (모순) | 빈 타입 (Void) |
| ¬P (부정) | P → Void |
| ∀x. P(x) | 의존 함수 타입 Π(x: A). B(x) |
| ∃x. P(x) | 의존 쌍 타입 Σ(x: A). B(x) |

마지막 두 행이 핵심이다. "모든 x에 대해 P(x)" — 전칭 양화사(universal quantification) — 는 **의존 함수 타입(Π-type, Pi-type)**에 대응하고, "P(x)인 x가 존재한다" — 존재 양화사(existential quantification) — 는 **의존 쌍 타입(Σ-type, Sigma-type)**에 대응한다.

### 의존 함수 타입 (Π-type)

```
Π(x: A). B(x)
```

반환 타입 `B(x)`가 입력값 `x`에 **의존**한다. Haskell의 `forall a. f a`는 이것의 제한된 형태 (값이 아닌 타입에만 의존).

```agda
-- Agda로 보는 Π-type 예제
-- "자연수 n을 받아 길이 n인 Bool 벡터를 반환하는 함수"
allFalse : (n : ℕ) → Vec Bool n
allFalse zero    = []
allFalse (suc n) = false ∷ allFalse n
```

`allFalse 3`은 `Vec Bool 3` 타입이고, `allFalse 5`는 `Vec Bool 5` 타입이다 — 반환 타입이 값에 따라 달라진다.

### 의존 쌍 타입 (Σ-type)

```
Σ(x: A). B(x)
```

"B(x)라는 성질을 만족하는 x: A와 그 증명의 쌍"이다. 존재 명제의 증인(witness)과 증명을 함께 담는다.

---

## Agda로 배우는 의존형 타입

### 설치 및 기본 개념

Agda는 의존형 타입 언어이자 증명 보조 도구(proof assistant)다. 함수형 프로그래밍(Haskell과 유사한 문법)과 수학적 증명이 하나의 언어로 통합된다.

```agda
-- 자연수 정의 (페아노 공리계)
data ℕ : Set where
  zero : ℕ
  suc  : ℕ → ℕ

-- {-# BUILTIN NATURAL ℕ #-} 를 사용하면 리터럴 숫자 사용 가능

-- 덧셈 정의
_+_ : ℕ → ℕ → ℕ
zero  + m = m
suc n + m = suc (n + m)

-- 벡터 정의: 길이가 타입에 인코딩됨
data Vec (A : Set) : ℕ → Set where
  []  : Vec A zero
  _∷_ : {n : ℕ} → A → Vec A n → Vec A (suc n)

-- 안전한 head: 길이 (suc n) 이상임을 타입이 보장
head : {A : Set} {n : ℕ} → Vec A (suc n) → A
head (x ∷ _) = x

-- head []  -- 컴파일 에러! Vec A 0 ≠ Vec A (suc n)

-- append: 길이 덧셈이 타입에 반영됨
_++_ : {A : Set} {m n : ℕ} → Vec A m → Vec A n → Vec A (m + n)
[]       ++ ys = ys
(x ∷ xs) ++ ys = x ∷ (xs ++ ys)
```

`append`의 타입 `Vec A m → Vec A n → Vec A (m + n)`은 Haskell이나 Rust로는 표현할 수 없다. 길이가 값이고, 그 값이 타입에 등장한다.

### 정리 증명: 덧셈의 교환법칙

의존형 타입 시스템에서 수학적 정리를 증명할 수 있다. Agda에서 타입이 명제이고, 그 타입의 항(term)이 증명이다.

```agda
-- 동일성 타입 (Martin-Löf Identity Type)
-- a ≡ b 는 "a와 b가 같음"을 주장하는 타입
-- refl은 a ≡ a 의 증명
data _≡_ {A : Set} : A → A → Set where
  refl : {x : A} → x ≡ x

-- cong: 함수에 같음을 적용
cong : {A B : Set} {a b : A} → (f : A → B) → a ≡ b → f a ≡ f b
cong f refl = refl

-- 덧셈의 우 항등원: n + 0 ≡ n
-- (정의에 의해 0 + n = n이지만, n + 0은 귀납법이 필요)
+-identityʳ : (n : ℕ) → n + zero ≡ n
+-identityʳ zero    = refl          -- 기저: 0 + 0 = 0
+-identityʳ (suc n) =               -- 귀납: suc n + 0 = suc n
  cong suc (+-identityʳ n)           --   cong suc (n + 0 ≡ n)

-- 덧셈의 교환법칙: n + m ≡ m + n
+-comm : (m n : ℕ) → m + n ≡ n + m
+-comm zero    n = sym (+-identityʳ n)
+-comm (suc m) n = {! 연습문제 !}    -- 구멍(hole)으로 남겨두고 타입 체커가 안내
```

Agda의 강력한 점은 **인터랙티브 개발**이다. 구멍(hole) `{! ... !}`에 커서를 두면 현재 맥락(context)과 목표 타입을 보여주며, 프로그래머가 단계별로 증명을 완성할 수 있다.

### Idris로 보는 실용적 의존형 프로그래밍

Idris는 Agda와 달리 **실용 프로그래밍**을 목표로 설계된 의존형 언어다. 실행 가능한 코드를 생성하고, Haskell과 유사한 문법으로 접근하기 쉽다.

```idris
-- Idris 2 예제

-- 의존형 벡터 (Vect는 표준 라이브러리 제공)
-- data Vect : Nat -> Type -> Type

-- 행렬 타입: 행×열이 타입에 인코딩
Matrix : Nat -> Nat -> Type -> Type
Matrix rows cols a = Vect rows (Vect cols a)

-- 행렬 전치 (transpose)
-- 입력: rows × cols 행렬
-- 출력: cols × rows 행렬  (타입이 이 사실을 보장!)
transpose : Matrix rows cols a -> Matrix cols rows a
transpose []        = replicate _ []
transpose (r :: rs) = zipWith (::) r (transpose rs)

-- 안전한 인덱싱: Fin n은 0부터 n-1까지의 유한 타입
-- Fin n의 값으로는 n 이상을 표현할 수 없으므로 범위 초과 불가
index : Fin n -> Vect n a -> a
index FZ     (x :: _)  = x
index (FS k) (_ :: xs) = index k xs

-- 정렬 알고리즘의 타입 시그니처
-- 출력 길이 = 입력 길이, 출력이 정렬됨을 증명
-- (실제 증명 포함 버전은 더 복잡하지만 개념 illustration)
sort : Ord a => Vect n a -> Vect n a
sort xs = fromList (sortList (toList xs))
  where
    sortList : List a -> List a
    sortList [] = []
    sortList (x :: xs) = insert x (sortList xs)
    insert : a -> List a -> List a
    insert x []        = [x]
    insert x (y :: ys) = if x <= y then x :: y :: ys
                          else y :: insert x ys
```

---

## 의존형 타입의 실제 응용

### 1. 안전한 프로토콜 구현

의존형 타입으로 통신 프로토콜의 상태 머신을 타입으로 인코딩하면, 잘못된 순서로 함수를 호출하는 것이 컴파일 타임에 방지된다.

```idris
-- 세션 타입(Session Types) 개념의 의사 코드
data SocketState = Closed | Listening | Connected

data Socket : SocketState -> Type where
  mkSocket : Socket Closed

-- bind는 Closed 소켓에만 가능
bind : Socket Closed -> Port -> Socket Listening
-- accept는 Listening 소켓에만 가능
accept : Socket Listening -> Socket Connected
-- send는 Connected 소켓에만 가능
send : Socket Connected -> String -> ()
-- Closed 소켓에 send 시도 → 컴파일 에러
```

### 2. 의존형 타입의 한계

- **타입 검사가 결정 불가능**: 임의의 의존형 타입 체계에서 타입 검사는 정지 문제(halting problem)로 귀결될 수 있다. Agda와 Coq는 **전체성(totality)** — 모든 함수가 종료해야 함 — 을 요구해 이를 해결한다.
- **개발 비용**: 정확한 타입을 설계하는 것 자체가 어렵고 시간이 많이 든다. 종종 증명을 작성하는 것이 프로그램 작성보다 더 어렵다.
- **생태계 미성숙**: Agda, Idris는 아직 Haskell이나 Rust처럼 풍부한 생태계가 없다.

### 3. 주류 언어에서의 영향

의존형 타입의 아이디어는 주류 언어에 점진적으로 스며들고 있다.

- **Rust**: `const generics`로 배열 크기를 타입 파라미터로 표현 (`[T; N]`)
- **TypeScript**: 템플릿 리터럴 타입, 조건부 타입으로 값에 따른 타입 분기
- **Liquid Haskell**: Refinement Types — Haskell 타입에 SMT 솔버 검증 조건 추가

```rust
// Rust const generics: 크기가 타입에 인코딩된 배열
fn dot_product<const N: usize>(a: [f64; N], b: [f64; N]) -> f64 {
    a.iter().zip(b.iter()).map(|(x, y)| x * y).sum()
}

// dot_product([1.0, 2.0], [3.0, 4.0])     // OK: N=2
// dot_product([1.0, 2.0], [3.0, 4.0, 5.0]) // 컴파일 에러: [f64; 2] ≠ [f64; 3]
```

---

## 학습 경로와 주의사항

### 추천 학습 순서

1. **Haskell 기초** — 강한 정적 타입, 타입 클래스, 대수적 데이터 타입 이해
2. **Hindley-Milner 타입 추론** — 타입 추론의 수학적 기반
3. **Curry-Howard 대응 개념** — 명제=타입, 증명=프로그램 직관 확립
4. **Agda 또는 Idris 입문** — The Little Typer (Idris 기반 입문서 추천)
5. **Martin-Löf 타입 이론** — 의존형 타입의 수학적 토대

### 핵심 교훈

의존형 타입 시스템의 진정한 가치는 "런타임 버그를 컴파일 타임으로 앞당긴다"는 데 있다. 항공기 제어 소프트웨어, 의료 기기 펌웨어, 암호화 라이브러리처럼 오류 비용이 막대한 영역에서 의존형 타입은 정식 증명 방법론 없이는 불가능한 수준의 보증을 제공한다.

Curry-Howard 대응은 궁극적으로 "프로그래밍과 수학 증명은 같은 것"이라는 심오한 통찰이다. 프로그래머가 올바른 함수를 작성할 때, 그는 사실 논리적 명제를 증명하고 있다 — 의존형 타입은 이 사실을 명시적이고 검증 가능하게 만든다.

## 참고 자료
- [Dependent type — Wikipedia](https://en.wikipedia.org/wiki/Dependent_type)
- [Agda 공식 문서](https://agda.readthedocs.io/en/latest/)
- [The Little Typer (Idris 기반 입문서)](https://thelittletyper.com/)
- [PLFA: Programming Language Foundations in Agda](https://plfa.github.io/)
