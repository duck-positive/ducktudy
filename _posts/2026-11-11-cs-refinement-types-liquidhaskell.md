---
layout: post
title: "Refinement Types와 LiquidHaskell: 타입에 술어를 내장해 버그를 컴파일 타임에 잡기"
date: 2026-11-11
categories: [cs, computer-science]
tags: [refinement-types, liquidhaskell, type-system, formal-verification, haskell, static-analysis]
---

소프트웨어 개발에서 버그의 상당수는 타입 시스템이 충분히 강하지 않기 때문에 발생합니다. `Int` 타입은 양의 정수와 음의 정수, 0을 모두 허용하지만, 함수가 실제로 양수만을 기대한다면? `List` 타입은 빈 리스트도 허용하지만, 함수가 비어 있지 않은 리스트만을 필요로 한다면? 이런 제약 조건들은 주석에 적히거나 런타임 예외로 처리되는 경우가 많습니다. **Refinement Types**는 이 문제를 타입 수준에서 해결합니다.

## Refinement Types란?

Refinement Types(정제 타입)는 기본 타입(base type)에 논리적 술어(predicate)를 추가하는 타입 시스템입니다. 예를 들어 `{v : Int | v > 0}`은 "양수인 정수"를 나타내고, `{v : [a] | len v > 0}`은 "비어 있지 않은 리스트"를 의미합니다. 이러한 술어는 컴파일 시점에 SMT(Satisfiability Modulo Theories) 솔버를 통해 자동으로 검증됩니다.

Refinement Types의 핵심 아이디어는 **타입이 명세(specification)가 된다**는 것입니다. 함수의 사전 조건(precondition)과 사후 조건(postcondition)을 타입 시그니처에 직접 인코딩하여, 별도의 런타임 검사나 formal proof 없이도 프로그램의 올바름을 컴파일 타임에 보장할 수 있습니다.

Refinement Types는 1991년 Tim Freeman과 Frank Pfenning의 논문에서 처음 제안되었으며, 이후 Dependent Types와 결합되어 Coq, Agda 같은 증명 보조기와 LiquidHaskell, F*, Dafny 같은 실용적인 도구들로 발전했습니다.

## 왜 필요한가?

### 런타임 오류를 컴파일 타임에 방지

가장 흔한 프로그래밍 버그 중 하나인 배열 범위 초과(array out-of-bounds), 0으로 나누기(division by zero), null 포인터 역참조는 모두 "값이 특정 조건을 만족하지 않을 때" 발생합니다. Refinement Types를 사용하면 이러한 오류가 발생 가능한지를 컴파일 타임에 자동으로 확인합니다.

### 명세와 구현의 일치

함수의 계약(contract)을 주석이나 테스트가 아닌 타입 시스템에 직접 표현하므로, 명세가 항상 구현과 함께 존재하고, 구현이 명세를 위반하면 컴파일이 실패합니다.

### 자동 검증

별도의 증명 작성이 불필요합니다. SMT 솔버(Z3 등)가 자동으로 타입 체크를 수행하므로, 개발자는 논리적 술어만 적어 주면 됩니다.

## LiquidHaskell: Haskell의 Refinement Type 체커

LiquidHaskell은 Haskell에 Refinement Types를 추가한 도구입니다. GHC 플러그인 형태로 동작하며, Haskell 코드에 `{-@ ... @-}` 형태의 특수 주석을 통해 타입 술어를 작성합니다.

### 설치 및 기본 설정

```bash
# Stack을 사용한 LiquidHaskell 설치
stack install liquidhaskell

# 또는 Cabal 사용
cabal install liquidhaskell

# GHC 플러그인으로 활성화 (package.yaml 또는 .cabal 파일에 추가)
# ghc-options: -fplugin=LiquidHaskell
```

## 코드 예제 1: 기본 Refinement Types

```haskell
-- LiquidHaskell 기본 예제
-- 파일: SafeArithmetic.hs

module SafeArithmetic where

-- 양수 정수 타입 별칭
{-@ type Pos = {v:Int | v > 0} @-}
{-@ type NonNeg = {v:Int | v >= 0} @-}
{-@ type NonZero = {v:Int | v /= 0} @-}

-- 0으로 나누기를 컴파일 타임에 방지
{-@ safeDiv :: Int -> NonZero -> Int @-}
safeDiv :: Int -> Int -> Int
safeDiv x y = x `div` y

-- NonNeg 타입: 비음수 정수의 절댓값 계산
{-@ absoluteValue :: Int -> NonNeg @-}
absoluteValue :: Int -> Int
absoluteValue x
  | x < 0    = -x
  | otherwise = x

-- 리스트 길이 타입 활용: 비어 있지 않은 리스트
{-@ type NEList a = {v:[a] | len v > 0} @-}

-- safeHead는 비어 있지 않은 리스트만 받음
{-@ safeHead :: NEList a -> a @-}
safeHead :: [a] -> a
safeHead (x:_) = x
safeHead []    = error "impossible: LiquidHaskell이 이 케이스를 허용하지 않음"

-- safeTail도 마찬가지
{-@ safeTail :: NEList a -> [a] @-}
safeTail :: [a] -> [a]
safeTail (_:xs) = xs
safeTail []     = error "impossible"

-- 사용 예시
example :: Int
example =
  let xs = [1, 2, 3] :: [Int]  -- LiquidHaskell이 len xs = 3 > 0을 추론
  in safeHead xs                -- 컴파일 성공

-- 아래 코드는 컴파일 실패: empty list
-- badExample :: Int
-- badExample = safeHead ([] :: [Int])  -- 오류: len [] = 0 위반
```

이 예제에서 `safeDiv`는 두 번째 인수가 `NonZero`여야 하므로, `safeDiv 10 0`을 호출하면 컴파일 오류가 발생합니다. `safeHead`는 `NEList a` 타입의 인수만 받으므로, 빈 리스트를 넘기면 역시 컴파일 타임에 오류가 발생합니다.

## 코드 예제 2: 배열 안전 접근과 정렬 알고리즘 검증

```haskell
-- 배열 범위 안전 접근 및 삽입 정렬 검증
-- 파일: SafeArray.hs

module SafeArray where

import Data.Vector (Vector, (!))
import qualified Data.Vector as V

-- 벡터와 유효한 인덱스 범위 타입
{-@ type ValidIdx v = {i:Int | 0 <= i && i < vlen v} @-}

-- 안전한 벡터 접근: 인덱스가 범위 내임을 컴파일 타임에 검증
{-@ safeGet :: v:(Vector a) -> ValidIdx v -> a @-}
safeGet :: Vector a -> Int -> a
safeGet vec i = vec ! i

-- 정렬된 리스트를 나타내는 Measure
{-@ measure isSorted :: [Int] -> Bool
    isSorted []       = True
    isSorted [x]      = True
    isSorted (x:y:zs) = x <= y && isSorted (y:zs)
  @-}

{-@ type SortedList = {v:[Int] | isSorted v} @-}

-- 정렬된 리스트에 원소 삽입 (삽입 후에도 정렬 유지)
{-@ insertSorted :: Int -> SortedList -> SortedList @-}
insertSorted :: Int -> [Int] -> [Int]
insertSorted x [] = [x]
insertSorted x (y:ys)
  | x <= y    = x : y : ys
  | otherwise = y : insertSorted x ys

-- 삽입 정렬: 결과가 항상 정렬된 리스트임을 보장
{-@ insertionSort :: [Int] -> SortedList @-}
insertionSort :: [Int] -> [Int]
insertionSort []     = []
insertionSort (x:xs) = insertSorted x (insertionSort xs)

-- 두 정렬 리스트 병합 (병합 정렬의 핵심)
{-@ merge :: SortedList -> SortedList -> SortedList @-}
merge :: [Int] -> [Int] -> [Int]
merge [] ys = ys
merge xs [] = xs
merge (x:xs) (y:ys)
  | x <= y    = x : merge xs (y:ys)
  | otherwise = y : merge (x:xs) ys

-- 사용 예시: 모든 타입 조건을 LiquidHaskell이 자동으로 검증
exampleSort :: [Int]
exampleSort =
  let unsorted = [5, 3, 1, 4, 2]
      sorted   = insertionSort unsorted  -- SortedList 타입 반환 보장
  in sorted   -- [1,2,3,4,5]
```

`insertionSort`의 반환 타입이 `SortedList`이므로, LiquidHaskell은 `insertSorted`와 재귀 호출이 정렬 불변식을 유지하는지 자동으로 검증합니다. 이 검증이 실패하면 컴파일 오류가 발생합니다.

## Liquid Types의 작동 원리: SMT 솔버와의 연동

LiquidHaskell은 타입 체크 과정에서 다음 단계를 수행합니다:

1. **타입 추론(Type Inference)**: Hindley-Milner 타입 추론으로 기본 타입을 결정
2. **제약 생성(Constraint Generation)**: 함수 호출 지점마다 술어 조건을 논리 제약으로 변환
3. **SMT 검증(SMT Verification)**: Z3 솔버로 모든 제약이 충족 가능한지 검증
4. **오류 보고**: 충족 불가능한 제약이 있으면 해당 위치의 타입 오류를 보고

예를 들어 `safeDiv x (y - 1)`을 호출할 때, LiquidHaskell은 `y - 1 /= 0`이라는 제약을 Z3에게 질의합니다. 이를 증명할 수 없으면 컴파일 오류로 처리합니다.

## 다른 언어의 Refinement Types

### F* (F-Star)

Microsoft Research의 F*는 Refinement Types와 Dependent Types를 모두 지원하는 언어로, TLS 1.3 구현(Everest 프로젝트)에 사용되었습니다:

```fsharp
// F* 예제: 길이가 같은 두 리스트를 받아 쌍으로 묶기
val zip: #a:Type -> #b:Type -> l1:list a -> l2:list b{length l2 = length l1}
       -> Tot (list (a * b)) (decreases l1)
```

### Dafny

Amazon Web Services에서 사용하는 Dafny는 `requires`와 `ensures` 절로 사전/사후 조건을 직접 표현합니다:

```csharp
// Dafny 예제: 최댓값 함수
method Max(a: int, b: int) returns (m: int)
  ensures m >= a && m >= b
  ensures m == a || m == b
{
  if a > b { m := a; } else { m := b; }
}
```

## 주의사항과 팁

### 1. 증명 부담(Proof Burden)

모든 술어를 SMT 솔버가 자동으로 검증할 수 있는 것은 아닙니다. 복잡한 수학적 성질(소수 여부, 행렬 역행렬 등)은 증명을 직접 작성해야 하거나 Liquid Measure로 표현해야 합니다.

### 2. 컴파일 시간 증가

SMT 솔버 호출로 인해 컴파일 시간이 늘어날 수 있습니다. 대형 프로젝트에서는 이를 고려해야 하며, 중요한 모듈에만 선택적으로 적용하는 전략이 효과적입니다.

### 3. 술어 표현력의 한계

LiquidHaskell의 술어는 선형 산술(linear arithmetic)과 비선형 산술의 일부만 자동으로 처리합니다. 복잡한 술어는 `assume` 또는 Measure를 통해 힌트를 제공해야 합니다.

### 4. 점진적 도입(Gradual Adoption)

전체 프로젝트에 한 번에 도입하는 것보다, 버그가 많이 발생하는 모듈이나 안전이 중요한 함수부터 점진적으로 도입하는 것이 현실적입니다.

### 5. 외부 라이브러리 연동

Refinement Types를 지원하지 않는 외부 라이브러리 함수는 `assume` 애노테이션으로 LiquidHaskell에게 타입 정보를 알려 주어야 합니다.

## 실무 적용 사례

- **암호화 라이브러리**: 키 길이, 패딩 크기 등 도메인 제약을 타입으로 표현
- **파서 구현**: 파싱 성공 시 반환 길이 보장, 범위 검사 자동화
- **데이터 처리 파이프라인**: 각 단계의 입출력 형식 불변식 검증
- **금융 계산**: 양수 금액, 유효 날짜 범위 등을 타입으로 강제

Refinement Types는 타입 시스템과 formal verification의 중간 지점에 위치하며, 실용적인 수준의 프로그램 정확성 보장을 제공합니다. 완전한 증명 보조기보다 훨씬 사용하기 쉽고, 단순한 런타임 검사보다 훨씬 강력한 보장을 제공한다는 점에서 점점 더 많은 주목을 받고 있습니다.

## 참고 자료

- [LiquidHaskell 공식 GitHub 저장소](https://github.com/ucsd-progsys/liquidhaskell)
- [Programming with Refinement Types - LiquidHaskell 튜토리얼](https://github.com/ucsd-progsys/liquidhaskell-tutorial)
