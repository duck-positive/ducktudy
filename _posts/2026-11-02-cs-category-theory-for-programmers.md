---
layout: post
title: "범주론(Category Theory) 프로그래머를 위한 완전 정복: Functor·Natural Transformation·Yoneda Lemma를 코드로 이해하기"
date: 2026-11-02
categories: [cs, computer-science]
tags: [category-theory, functor, monad, natural-transformation, yoneda, haskell, functional-programming, type-theory]
---

범주론(Category Theory)은 20세기 중반 수학자들이 위상수학과 대수학 사이의 공통 구조를 포착하기 위해 개발한 추상 수학의 한 분야다. 그런데 1980년대 이후 함수형 프로그래밍 커뮤니티가 이를 발견하면서, 오늘날 Haskell의 `Functor`, `Monad`, Rust의 `Iterator`, Scala의 `cats` 라이브러리가 범주론의 직접적인 구현이 되었다.

"추상 넌센스(abstract nonsense)"라는 별명을 가진 이 분야가 왜 소프트웨어 엔지니어에게 중요한가? 범주론은 **합성(composition)**과 **구조 보존(structure preservation)**에 대한 보편 언어를 제공한다. 개별 데이터 구조나 언어의 특수성을 초월해, "이 패턴들은 왜 모두 비슷하게 느껴지는가"에 대한 깊은 답을 준다.

## 범주(Category)란 무엇인가

범주 `C`는 세 가지 요소로 구성된다.

1. **대상(Objects)**: `Ob(C)` — 집합, 타입, 벡터 공간, 위상 공간 등 무엇이든 될 수 있다.
2. **사상(Morphisms, 또는 Arrows)**: 두 대상 사이의 "관계" 또는 "변환". 대상 `A`에서 `B`로 가는 사상의 집합을 `Hom(A, B)`라 표기한다.
3. **합성(Composition)**: `f: A → B`와 `g: B → C`가 있으면 `g ∘ f: A → C`가 존재한다.

그리고 두 가지 법칙을 만족해야 한다.

- **결합법칙**: `h ∘ (g ∘ f) = (h ∘ g) ∘ f`
- **항등원**: 모든 대상 `A`에 대해 `id_A: A → A`가 존재해서 `f ∘ id_A = f = id_B ∘ f`

가장 직관적인 범주는 **Hask** — Haskell 타입의 범주다. 대상은 Haskell 타입(`Int`, `String`, `[a]` 등)이고, 사상은 함수(`f: A -> B`)이며, 합성은 `(.)` 연산자다.

프로그래밍 관점에서 범주론의 핵심은 "**데이터가 아닌 변환**에 집중한다"는 것이다.

---

## 함자(Functor): 범주 사이의 구조 보존 변환

### 개념

두 범주 `C`와 `D` 사이의 함자(Functor) `F: C → D`는:

1. 각 대상 `A ∈ Ob(C)`를 대상 `F(A) ∈ Ob(D)`로 매핑
2. 각 사상 `f: A → B`를 사상 `F(f): F(A) → F(B)`로 매핑

하면서 다음 법칙을 지켜야 한다.

- **항등원 보존**: `F(id_A) = id_{F(A)}`
- **합성 보존**: `F(g ∘ f) = F(g) ∘ F(f)`

Haskell에서 함자는 정확히 `Functor` 타입클래스다.

```haskell
-- Functor 타입클래스 정의
class Functor f where
    fmap :: (a -> b) -> f a -> f b
    -- 법칙:
    -- fmap id = id
    -- fmap (g . f) = fmap g . fmap f

-- Maybe 함자: 실패 가능성을 감싼 컨텍스트
instance Functor Maybe where
    fmap _ Nothing  = Nothing
    fmap f (Just x) = Just (f x)

-- List 함자
instance Functor [] where
    fmap = map

-- Tree 함자 (직접 정의)
data Tree a = Leaf | Node (Tree a) a (Tree a)

instance Functor Tree where
    fmap _ Leaf         = Leaf
    fmap f (Node l v r) = Node (fmap f l) (f v) (fmap f r)
```

`fmap`은 함수 `f: a -> b`를 "컨텍스트 안에 들어간 함수" `f a -> f b`로 **승격(lift)**시킨다. `Maybe`의 경우 "실패 가능성이라는 컨텍스트"를, 리스트의 경우 "비결정성(nondeterminism)이라는 컨텍스트"를 투명하게 다룬다.

### 함자 법칙 검증

```haskell
-- 항등원 보존 확인
prop_identity :: (Eq (f a), Functor f) => f a -> Bool
prop_identity fa = fmap id fa == id fa

-- 합성 보존 확인
prop_composition :: (Eq (f c), Functor f)
                 => (b -> c) -> (a -> b) -> f a -> Bool
prop_composition g f fa =
    fmap (g . f) fa == (fmap g . fmap f) fa

-- QuickCheck로 검증
-- quickCheck (prop_identity :: Maybe Int -> Bool)         -- True
-- quickCheck (prop_composition (+1) (*2) :: [Int] -> Bool) -- True
```

---

## 자연 변환(Natural Transformation): 함자 사이의 변환

### 개념

두 함자 `F, G: C → D`가 있을 때, **자연 변환(Natural Transformation)** `η: F ⟹ G`는:

- 각 대상 `A ∈ Ob(C)`에 대해 성분 사상(component) `η_A: F(A) → G(A)`를 대응

하면서 **자연성 조건(naturality condition)**을 만족해야 한다.

모든 사상 `f: A → B`에 대해:

```
G(f) ∘ η_A = η_B ∘ F(f)
```

다이어그램으로 표현하면 다음 정사각형이 **가환(commute)**해야 한다.

```
F(A) ---η_A---> G(A)
  |               |
F(f)            G(f)
  |               |
  v               v
F(B) ---η_B---> G(B)
```

이는 "변환을 먼저 한 뒤 사상을 적용하는 것"과 "사상을 먼저 적용한 뒤 변환하는 것"이 같아야 함을 의미한다 — 변환이 맥락(context)에 무관하게 균일하게 동작해야 한다는 뜻이다.

Haskell에서 자연 변환은 다형 함수(polymorphic function)다.

```haskell
-- 자연 변환의 Haskell 표현
type Nat f g = forall a. f a -> g a

-- safeHead: List → Maybe 자연 변환
safeHead :: Nat [] Maybe
safeHead []    = Nothing
safeHead (x:_) = Just x

-- 자연성 조건 검증:
-- fmap f (safeHead xs) == safeHead (fmap f xs)
prop_naturality :: (Eq b) => (Int -> b) -> [Int] -> Bool
prop_naturality f xs =
    fmap f (safeHead xs) == safeHead (fmap f xs)

-- flatten: Maybe (Maybe a) → Maybe a  (자연 변환)
flatten :: Maybe (Maybe a) -> Maybe a
flatten Nothing         = Nothing
flatten (Just Nothing)  = Nothing
flatten (Just (Just x)) = Just x

-- maybeToList: Maybe → []  자연 변환
maybeToList :: Nat Maybe []
maybeToList Nothing  = []
maybeToList (Just x) = [x]

-- listToMaybe: [] → Maybe  자연 변환 (역방향 - 손실이 있음)
listToMaybe :: Nat [] Maybe
listToMaybe []    = Nothing
listToMaybe (x:_) = Just x
```

자연 변환은 "컨테이너의 모양(shape)을 바꾸되 내용물(elements)은 그대로 두는" 변환이다. 이것이 바로 Haskell에서 다형 함수가 임의로 값을 생성하거나 검사하지 않는 이유다 — parametricity가 자연성을 보장한다.

---

## Yoneda Lemma: 범주론의 핵심 정리

### 개념

Yoneda Lemma는 범주론에서 가장 중요한 정리 중 하나다. 직관적으로 "**대상은 그것에서 나오는 화살표(사상)들의 전체 집합으로 완전히 결정된다**"는 것을 말한다.

형식적으로, 함자 `F: C → Set`과 대상 `A ∈ C`에 대해:

```
Nat(Hom(A, -), F) ≅ F(A)
```

즉, "대상 `A`에서 시작하는 모든 사상을 `F`로 변환하는 자연 변환의 집합"과 "`F(A)`의 원소들"이 일대일 대응된다.

Haskell에서는 다음과 같이 표현된다.

```haskell
-- Hom(A, -)는 Haskell에서 (a ->)  즉  Reader a
newtype Yoneda f a = Yoneda { runYoneda :: forall b. (a -> b) -> f b }

-- Yoneda ≅ f  (동형)
toYoneda :: Functor f => f a -> Yoneda f a
toYoneda fa = Yoneda (\f -> fmap f fa)

fromYoneda :: Yoneda f a -> f a
fromYoneda (Yoneda y) = y id

-- Yoneda 내장(Embedding): (a -> b) -> (r -> a) -> (r -> b)
-- 이는 단순히 함수 합성이다! 
-- Yoneda Lemma는 함수 합성이 자연 변환임을 보인다

-- CPS(Continuation Passing Style)와의 연결
-- Cont r a = (a -> r) -> r  는 Yoneda f a의 특수 경우 (f = Const r)

-- 실용적 응용: Yoneda fusion
-- fmap f . fmap g = fmap (f . g) -- 이 최적화는 Yoneda에서 직접 도출된다

-- 함수 합성의 연쇄를 하나의 fmap으로 병합
efficientMap :: Functor f => [a -> a] -> f a -> f a
efficientMap fs = fmap (foldr (.) id fs)
-- 개별 fmap 적용: O(n) 번의 구조 순회
-- Yoneda 활용: O(1) 번의 구조 순회 (내부에서 함수 합성만)
```

### Yoneda의 실용적 의미

1. **CPS 변환**: Continuation Passing Style은 Yoneda Embedding의 특수 경우다. CPS로 변환된 코드가 "항상 꼬리 호출 최적화 가능"한 이유가 여기에 있다.

2. **Free Monad 최적화**: `Data.Sequence`나 `Data.Map`의 `fmap` 연쇄가 Yoneda fusion으로 최적화된다.

3. **메모이제이션**: 함수 `a -> b`는 `Hom(a, b)`. Yoneda 원리는 "함수는 그 입력에 대한 반응 전체로 완전히 정의된다"고 말한다 — 메모이제이션의 정당성 근거다.

---

## 모나드(Monad): 모노이드이자 함자

### 개념

모나드는 흔히 "burritos" 비유로 오해받지만, 범주론적으로는 명확히 정의된다. 모나드는 **자기 함자(endofunctor) `T: C → C`와 두 자연 변환**으로 구성된다.

- `η: Id ⟹ T` (unit, return)
- `μ: T² ⟹ T` (multiplication, join)

이 둘이 모노이드 법칙을 만족할 때 모나드라 한다.

```haskell
class Functor m => Monad m where
    return :: a -> m a           -- η: 순수값을 컨텍스트로 래핑
    (>>=)  :: m a -> (a -> m b) -> m b  -- Kleisli composition

-- 모나드 법칙
-- return a >>= f   = f a             (좌 항등원)
-- m >>= return     = m               (우 항등원)
-- (m >>= f) >>= g  = m >>= (\x -> f x >>= g)  (결합법칙)

-- IO 모나드: 실세계 부작용을 타입 시스템으로 격리
-- State 모나드: 상태 변환을 합성 가능하게
-- Either 모나드: 실패 가능 계산의 연쇄

-- Kleisli 범주: 모나드로부터 새로운 범주를 파생
-- 대상: C의 대상 그대로
-- 사상: A → T(B) 형태의 함수 (Kleisli arrow)
-- 합성: (>=>)

(>=>) :: Monad m => (a -> m b) -> (b -> m c) -> a -> m c
f >=> g = \x -> f x >>= g

-- 이 Kleisli 범주에서 (>=>) 은 합성이고,
-- return은 항등 사상이다
-- 모나드 법칙 = Kleisli 범주의 범주 법칙
```

---

## 주의사항 및 학습 팁

### 1. 추상화의 비용

범주론적 추상화는 강력하지만 비용이 있다. 초심자에게 `Monad`를 범주론으로 설명하면 오히려 혼란을 준다. **직관 먼저, 추상화 나중**이 올바른 순서다. `Maybe`와 `IO`를 숙련되게 사용한 뒤 Yoneda Lemma를 공부하면 "아하!" 순간이 온다.

### 2. 법칙을 항상 확인하라

함자 법칙, 모나드 법칙을 위반하는 구현은 범주론적으로 올바르지 않으며 예상치 못한 버그의 원인이 된다. Haskell에서는 QuickCheck나 Hedgehog으로 속성 기반 테스트를 통해 법칙을 검증할 수 있다.

### 3. 범주론은 언어를 초월한다

Haskell에서 가장 직접적으로 드러나지만, 범주론적 패턴은 Rust(`Iterator`, `Option`), Scala(`cats`, `ZIO`), Kotlin(`Arrow`), TypeScript(`fp-ts`)에서도 구현된다. 한 언어에서 개념을 이해하면 다른 언어로 쉽게 전이된다.

### 4. 추천 학습 경로

- Bartosz Milewski의 *Category Theory for Programmers* (블로그/책, 무료)
- `Learn You a Haskell for Great Good!` (함수형 프로그래밍 기초)
- `Haskell Programming from First Principles` (심화)
- `Seven Sketches in Compositionality` (응용 범주론, 덜 추상적)

범주론은 소프트웨어의 근본 구조를 다른 각도에서 보게 해주는 렌즈다. 모든 코드를 범주론으로 작성할 필요는 없지만, 이 개념들이 설계 결정에 영향을 미치는 방식은 프로그래머의 사고를 한 단계 끌어올린다.

## 참고 자료
- [Category Theory for Programmers — Bartosz Milewski (GitHub PDF)](https://github.com/hmemcpy/milewski-ctfp-pdf)
- [Natural Transformations — Category Theory for Programmers](https://ctfpis.gitbook.io/project/category-theory-for-programmers/part_one/natural-transformations)
- [Practical Introduction to Category Theory](https://monadics.vercel.app/practical-introduction-category-theory-structure-computation)
- [Haskell Functor (Haskell Wiki)](https://wiki.haskell.org/Functor)
