---
layout: post
title: "재귀 스킴(Recursion Schemes) 완전 정복: Catamorphism·Anamorphism·Hylomorphism으로 재귀를 추상화하기"
date: 2026-11-15
categories: [cs, computer-science]
tags: [functional-programming, recursion-schemes, catamorphism, anamorphism, hylomorphism, haskell, type-theory]
---

함수형 프로그래밍에서 재귀는 불가피하지만, 무분별한 직접 재귀는 코드의 의도를 흐립니다. **재귀 스킴(Recursion Schemes)**은 재귀 패턴을 고차 함수로 추상화하여 "어떻게(how) 순회하는가"와 "무엇을(what) 계산하는가"를 분리합니다. 1991년 Erik Meijer, Maarten Fokkinga, Ross Paterson이 발표한 "Functional Programming with Bananas, Lenses, Envelopes and Barbed Wire"에서 체계화된 이 아이디어는 범주론(Category Theory)의 F-대수(F-Algebra) 이론에 기반합니다.

## 개념 설명

### Fix 타입: 재귀의 최소 고정점

재귀 스킴의 핵심은 **Fix** 타입입니다. 임의의 함수자(Functor) `f`에 대해, `Fix f`는 `f`의 최소 고정점(least fixed point)을 나타냅니다.

```haskell
-- Fix: 재귀적 타입의 고정점
newtype Fix f = Fix { unFix :: f (Fix f) }

-- 예: 자연수 패턴 함수자
data NatF a = ZeroF | SuccF a deriving (Functor, Show)
type Nat = Fix NatF

-- 예: 리스트 패턴 함수자
data ListF a r = NilF | ConsF a r deriving (Functor, Show)
type List a = Fix (ListF a)

-- 예: 이진 트리 패턴 함수자
data TreeF a r = LeafF | NodeF r a r deriving (Functor, Show)
type Tree a = Fix (TreeF a)
```

일반적인 재귀 ADT(예: `data Tree a = Leaf | Node (Tree a) a (Tree a)`)와의 차이:
- **일반 ADT**: 자기 자신을 직접 참조
- **Fix 기반**: 패턴 함수자가 재귀 위치에 타입 변수 `r`을 사용, `Fix`가 재귀를 담당

이 분리 덕분에 재귀 위치를 `fmap`으로 조작할 수 있고, 그것이 catamorphism의 핵심입니다.

### Catamorphism: 접기(Fold)의 일반화

```haskell
-- F-대수: 함수자를 하나의 값으로 접는 함수
type Algebra f a = f a -> a

-- Catamorphism: Fix f를 a로 아래에서 위로 접기
cata :: Functor f => Algebra f a -> Fix f -> a
cata alg = alg . fmap (cata alg) . unFix
--              ^^^^^^^^^^^^^^^^
--              재귀: 각 자식에 cata를 적용한 후 alg로 합치기
```

리스트의 `foldr`은 catamorphism의 특수 경우입니다:
- `F = ListF a`, `Algebra = \case NilF -> z; ConsF x acc -> f x acc`

### Anamorphism: 펼치기(Unfold)의 일반화

```haskell
-- F-쌍대수: 하나의 씨앗 값으로 함수자를 생성
type Coalgebra f a = a -> f a

-- Anamorphism: 씨앗 값 a로부터 Fix f 구조를 생성
ana :: Functor f => Coalgebra f a -> a -> Fix f
ana coalg = Fix . fmap (ana coalg) . coalg
```

리스트의 `unfoldr`은 anamorphism의 특수 경우입니다.

### Hylomorphism: Unfold 후 Fold

```haskell
-- Hylomorphism = Catamorphism ∘ Anamorphism
-- 중요: 중간 Fix f 구조를 메모리에 쌓지 않아 더 효율적
hylo :: Functor f => Algebra f b -> Coalgebra f a -> a -> b
hylo alg coalg = alg . fmap (hylo alg coalg) . coalg
```

이론적으로 `cata alg . ana coalg`와 동일하지만, hylomorphism은 중간 자료구조를 생성하지 않으므로 메모리 효율적입니다.

### 추가 스킴들

```haskell
-- Paramorphism: 현재 Fix f 노드와 하위 결과를 동시에 사용
type RAlgebra f a = f (Fix f, a) -> a
para :: Functor f => RAlgebra f a -> Fix f -> a
para alg = alg . fmap (\t -> (t, para alg t)) . unFix

-- Apomorphism: 조기 종료가 가능한 unfold (Either로 종료 여부 표현)
type RCoalgebra f a = a -> f (Either (Fix f) a)
apo :: Functor f => RCoalgebra f a -> a -> Fix f
apo coalg = Fix . fmap (either id (apo coalg)) . coalg
```

## 왜 필요한가?

### 1. 관심사 분리 (Separation of Concerns)

직접 재귀 코드에서는 순회 로직과 계산 로직이 뒤섞입니다:

```haskell
-- 직접 재귀: 순회와 계산이 혼재
sumTree :: Tree Int -> Int
sumTree (Leaf)       = 0
sumTree (Node l x r) = sumTree l + x + sumTree r
--       ^^^^^^^^^                 ^^^^^^^^^^^
--       순회 로직                 계산 로직

-- Catamorphism: 완전한 분리
sumTree :: Tree Int -> Int
sumTree = cata $ \case
    LeafF       -> 0
    NodeF l x r -> l + x + r  -- 오직 계산만
```

### 2. 재사용성과 조합성

동일한 catamorphism을 여러 계산에 재사용할 수 있습니다. 순회 패턴 자체가 재사용되는 것입니다.

### 3. 최적화 (Deforestation / Fusion)

GHC 컴파일러는 `cata . ana` 패턴을 자동으로 중간 구조를 제거한 hylo로 최적화(deforestation)할 수 있습니다. `{-# RULES #-}` pragma를 통해 "fold/unfold fusion"이 적용됩니다.

### 4. 종료 보장

Catamorphism은 항상 종료합니다. F-대수의 수학적 성질 덕분에 무한 루프가 불가능합니다(코귀납적 구조에는 아나모피즘을 사용).

## 실제 구현 예제

### 예제 1: recursion-schemes 라이브러리 활용 (Haskell)

```haskell
{-# LANGUAGE DeriveFunctor, TemplateHaskell, TypeFamilies #-}

import Data.Functor.Foldable
import Data.Functor.Foldable.TH (makeBaseFunctor)

-- 1. 일반 ADT 정의
data Expr
    = Lit Int
    | Add Expr Expr
    | Mul Expr Expr
    | Var String
    deriving (Show)

-- 2. 자동으로 패턴 함수자 생성 (ExprF)
makeBaseFunctor ''Expr
-- 위 선언이 아래를 자동 생성:
-- data ExprF r = LitF Int | AddF r r | MulF r r | VarF String
--   deriving Functor
-- type instance Base Expr = ExprF

-- 3. Catamorphism으로 수식 평가
eval :: [(String, Int)] -> Expr -> Int
eval env = cata $ \case
    LitF n     -> n
    AddF l r   -> l + r
    MulF l r   -> l * r
    VarF name  -> maybe 0 id (lookup name env)

-- 4. Catamorphism으로 수식을 문자열로 변환
prettyPrint :: Expr -> String
prettyPrint = cata $ \case
    LitF n     -> show n
    AddF l r   -> "(" ++ l ++ " + " ++ r ++ ")"
    MulF l r   -> "(" ++ l ++ " * " ++ r ++ ")"
    VarF name  -> name

-- 5. Catamorphism으로 수식 최적화 (상수 접기)
optimize :: Expr -> Expr
optimize = cata $ \case
    AddF (Lit 0) r   -> r         -- 0 + x = x
    AddF l (Lit 0)   -> l         -- x + 0 = x
    MulF (Lit 0) _   -> Lit 0     -- 0 * x = 0
    MulF _ (Lit 0)   -> Lit 0     -- x * 0 = 0
    MulF (Lit 1) r   -> r         -- 1 * x = x
    MulF l (Lit 1)   -> l         -- x * 1 = x
    AddF (Lit a) (Lit b) -> Lit (a + b)  -- 상수 접기
    MulF (Lit a) (Lit b) -> Lit (a * b)  -- 상수 접기
    other -> embed other  -- 변환 불가: 원래 형태 유지

-- 6. Anamorphism으로 범위 리스트 생성
rangeList :: Int -> Fix (ListF Int)
rangeList = ana $ \n ->
    if n <= 0
        then NilF
        else ConsF n (n - 1)

-- 7. Hylomorphism으로 퀵소트 구현
data QSortF r = QSNil | QSNode r Int r deriving Functor

qsort :: [Int] -> [Int]
qsort = hylo alg coalg
  where
    -- coalg: 분해 (anamorphism 역할)
    coalg []     = QSNil
    coalg (x:xs) = QSNode smaller x larger
      where
        smaller = filter (<= x) xs
        larger  = filter (>  x) xs
    -- alg: 합치기 (catamorphism 역할)
    alg QSNil            = []
    alg (QSNode l x r)   = l ++ [x] ++ r

-- 8. Paramorphism으로 피보나치 수열 (이전 두 단계 모두 필요)
-- 자연수를 직접 재귀로 순회하면서 현재 Fix NatF 보존
fib :: Fix NatF -> Int
fib = para $ \case
    ZeroF             -> 0
    SuccF (Fix ZeroF, _)  -> 1
    SuccF (prev, fibPrev) ->
        fibPrev + para fibAlg prev  -- 이전 두 항 합산
  where
    fibAlg = \case
        ZeroF        -> 0
        SuccF (_, n) -> n

-- 사용 예시
main :: IO ()
main = do
    let expr = Mul (Add (Lit 2) (Var "x")) (Add (Lit 0) (Lit 3))
    let env  = [("x", 5)]

    putStrLn $ "수식:   " ++ prettyPrint expr
    -- "((2 + x) * (0 + 3))"
    putStrLn $ "평가:   " ++ show (eval env expr)
    -- 21  (= (2+5)*3)
    putStrLn $ "최적화: " ++ prettyPrint (optimize expr)
    -- "((2 + x) * 3)"

    putStrLn $ "퀵소트: " ++ show (qsort [3,1,4,1,5,9,2,6,5,3])
    -- [1,1,2,3,3,4,5,5,6,9]
```

### 예제 2: Python으로 구현한 재귀 스킴

```python
from dataclasses import dataclass
from typing import TypeVar, Callable, Any, Generic, List, Optional

# ------------------------------------------------------------------
# Fix 타입 및 핵심 스킴 구현
# ------------------------------------------------------------------

class Fix:
    """재귀 타입의 최소 고정점"""
    def __init__(self, layer):
        self.layer = layer

    def __repr__(self):
        return f"Fix({self.layer})"

def cata(alg: Callable, fmap: Callable, tree: Fix) -> Any:
    """Catamorphism: 아래에서 위로 접기"""
    # 현재 레이어를 열고 각 자식에 재귀 적용 후 alg 호출
    mapped = fmap(lambda child: cata(alg, fmap, child), tree.layer)
    return alg(mapped)

def ana(coalg: Callable, fmap: Callable, seed: Any) -> Fix:
    """Anamorphism: 씨앗으로부터 구조 생성"""
    layer = coalg(seed)
    return Fix(fmap(lambda s: ana(coalg, fmap, s), layer))

def hylo(alg: Callable, coalg: Callable, fmap: Callable, seed: Any) -> Any:
    """Hylomorphism: unfold 후 fold (중간 구조 없음)"""
    layer = coalg(seed)
    mapped = fmap(lambda s: hylo(alg, coalg, fmap, s), layer)
    return alg(mapped)

# ------------------------------------------------------------------
# 이진 트리 함수자
# ------------------------------------------------------------------

@dataclass
class LeafF:
    pass

@dataclass
class NodeF:
    left: Any   # r 타입 (자식 위치)
    value: int
    right: Any  # r 타입

def tree_fmap(f: Callable, layer) -> Any:
    """TreeF에 대한 fmap: 자식 위치에만 f 적용"""
    if isinstance(layer, LeafF):
        return LeafF()
    return NodeF(f(layer.left), layer.value, f(layer.right))

# 이진 탐색 트리 삽입 (anamorphism으로 구성)
def bst_insert_coalg(pair):
    """(값 목록, 삽입할 값) -> TreeF"""
    values, insert_val = pair
    if not values:
        return LeafF()
    mid = len(values) // 2
    pivot = sorted(values)[mid]
    left_vals  = [v for v in values if v < pivot]
    right_vals = [v for v in values if v > pivot]
    return NodeF(
        (left_vals, None),    # 왼쪽 서브트리 씨앗
        pivot,
        (right_vals, None)    # 오른쪽 서브트리 씨앗
    )

def build_bst(values: List[int]) -> Fix:
    """정렬된 BST를 anamorphism으로 구성"""
    def coalg(vals):
        if not vals:
            return LeafF()
        s = sorted(vals)
        mid = len(s) // 2
        return NodeF(s[:mid], s[mid], s[mid+1:])

    return ana(coalg, tree_fmap, sorted(values))

# 트리 catamorphism들
def tree_depth(tree: Fix) -> int:
    """트리 깊이 (catamorphism)"""
    return cata(
        lambda layer: 0 if isinstance(layer, LeafF)
                      else 1 + max(layer.left, layer.right),
        tree_fmap,
        tree
    )

def tree_sum(tree: Fix) -> int:
    """트리 합산 (catamorphism)"""
    return cata(
        lambda layer: 0 if isinstance(layer, LeafF)
                      else layer.left + layer.value + layer.right,
        tree_fmap,
        tree
    )

def tree_inorder(tree: Fix) -> List[int]:
    """중위 순회 (catamorphism)"""
    return cata(
        lambda layer: [] if isinstance(layer, LeafF)
                      else layer.left + [layer.value] + layer.right,
        tree_fmap,
        tree
    )

# ------------------------------------------------------------------
# Hylomorphism으로 머지소트 구현
# ------------------------------------------------------------------

@dataclass
class MergeNilF:
    pass

@dataclass
class MergeSplitF:
    left: Any   # r 타입
    right: Any  # r 타입
    pivot: list # 분할되지 않은 단일 원소 (리프 역할)

def mergesort_fmap(f, layer):
    if isinstance(layer, MergeNilF):
        return MergeNilF()
    if isinstance(layer, list):  # 단일 원소 리프
        return layer
    return MergeSplitF(f(layer.left), f(layer.right), layer.pivot)

def mergesort_coalg(lst: list):
    """리스트를 반으로 분할 (unfold)"""
    if len(lst) <= 1:
        return lst  # 베이스 케이스: 리스트 그대로 반환
    mid = len(lst) // 2
    return MergeSplitF(lst[:mid], lst[mid:], [])

def mergesort_alg(layer) -> list:
    """정렬된 두 리스트를 합치기 (fold)"""
    if isinstance(layer, list):
        return layer  # 베이스 케이스
    l, r = layer.left, layer.right
    result = []
    i = j = 0
    while i < len(l) and j < len(r):
        if l[i] <= r[j]:
            result.append(l[i]); i += 1
        else:
            result.append(r[j]); j += 1
    return result + l[i:] + r[j:]

def mergesort(lst: list) -> list:
    """Hylomorphism으로 구현한 머지소트"""
    return hylo(mergesort_alg, mergesort_coalg, mergesort_fmap, lst)

# ------------------------------------------------------------------
# 테스트
# ------------------------------------------------------------------
if __name__ == '__main__':
    values = [5, 3, 8, 1, 9, 2, 7, 4, 6]
    bst = build_bst(values)

    print(f"입력: {values}")
    print(f"BST 중위 순회: {tree_inorder(bst)}")   # [1, 2, 3, 4, 5, 6, 7, 8, 9]
    print(f"BST 깊이:      {tree_depth(bst)}")      # 3 또는 4
    print(f"BST 합산:      {tree_sum(bst)}")         # 45

    data = [3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5]
    print(f"\n머지소트 입력:  {data}")
    print(f"머지소트 결과:  {mergesort(data)}")      # [1, 1, 2, 3, 3, 4, 5, 5, 5, 6, 9]
```

## 주의사항과 팁

### 1. 스택 오버플로우 문제

깊은 재귀 구조에 catamorphism을 적용하면 스택 오버플로우가 발생합니다. 해결책:
- **트램폴린(Trampolining)**: 꼬리 재귀로 변환
- **CPS (Continuation-Passing Style)**: 상태를 힙에 저장
- **Haskell의 lazy evaluation**: 무한 구조에 anamorphism을 적용할 때 문제 없음(코귀납적)

### 2. TemplateHaskell로 보일러플레이트 제거

```haskell
{-# LANGUAGE TemplateHaskell #-}
import Data.Functor.Foldable.TH

data MyTree a = MyLeaf | MyNode (MyTree a) a (MyTree a)
makeBaseFunctor ''MyTree
-- 자동 생성: data MyTreeF a r = MyLeafF | MyNodeF r a r
-- instance Functor (MyTreeF a)
-- instance Recursive (MyTree a)
-- instance Corecursive (MyTree a)
```

### 3. 범주론적 관점

재귀 스킴은 범주론의 개념에 대응합니다:
- **Catamorphism** = initial F-algebra로의 유일한 사상(unique morphism)
- **Anamorphism** = terminal F-coalgebra로의 유일한 사상
- **Hylomorphism** = 임의 F-algebra와 F-coalgebra 사이의 사상

이 관계가 보장하는 것: 재귀 스킴으로 표현된 함수는 항상 범주론적으로 "올바른" 계산을 수행합니다.

### 4. 성능: Fusion 법칙

GHC는 다음 fusion 법칙을 `{-# RULES #-}` pragma로 자동 적용합니다:

```haskell
-- fold/unfold fusion
cata alg . ana coalg = hylo alg coalg
-- 중간 Fix 구조가 완전히 제거됨
```

이를 활용하면 파이프라인 형태의 함수 조합이 성능 손실 없이 동작합니다.

## 참고 자료

- [recursion-schemes: Haskell 패키지 (Hackage)](https://hackage.haskell.org/package/recursion-schemes)
- [Fantastic Morphisms and Where to Find Them (arXiv:2202.13633)](https://arxiv.org/abs/2202.13633)
- [Bananas, Lenses, Envelopes and Barbed Wire - 원본 논문 (1991)](https://maartenfokkinga.github.io/utwente/mmf91m.pdf)
- [Effectful Recursion Schemes - Effekt 언어 블로그](https://effekt-lang.org/blog/recursion-schemes)
