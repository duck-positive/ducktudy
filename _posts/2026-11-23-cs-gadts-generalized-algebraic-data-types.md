---
layout: post
title: "GADTs 완전 정복: 타입 안전한 AST와 인터프리터를 만드는 일반화된 대수적 데이터 타입"
date: 2026-11-23
categories: [cs, computer-science]
tags: [gadts, type-system, haskell, rust, type-safety, algebraic-data-types, phantom-types, dependent-types]
---

## 개념 설명

GADT(Generalized Algebraic Data Type, 일반화된 대수적 데이터 타입)는 대수적 데이터 타입(ADT)의 강력한 확장으로, **각 생성자(constructor)가 반환 타입을 더 구체적으로 명시**할 수 있게 합니다. 이를 통해 타입 시스템이 프로그램의 불변식(invariant)을 더 정확하게 표현하고, 런타임 오류를 컴파일 타임에 잡아낼 수 있습니다.

### 일반 ADT와 GADT의 차이

**일반 ADT (Haskell)**:
```haskell
-- 모든 생성자가 Expr a를 반환 (a는 미확정)
data Expr a
  = Lit Int         -- Expr a (a는 뭐든 될 수 있음 → 타입 불안전)
  | BoolLit Bool    -- Expr a
  | Add (Expr a) (Expr a)  -- Expr a
  | If (Expr a) (Expr a) (Expr a)  -- Expr a
```

이 ADT로는 `Add (BoolLit True) (Lit 42)` 같은 타입 오류를 컴파일 타임에 잡을 수 없습니다.

**GADT (Haskell)**:
```haskell
{-# LANGUAGE GADTs #-}

-- 각 생성자가 반환 타입을 구체적으로 지정
data Expr a where
  Lit     :: Int  -> Expr Int           -- 반드시 Expr Int
  BoolLit :: Bool -> Expr Bool          -- 반드시 Expr Bool
  Add     :: Expr Int  -> Expr Int  -> Expr Int   -- Int + Int = Int
  Mul     :: Expr Int  -> Expr Int  -> Expr Int
  Eq      :: Expr Int  -> Expr Int  -> Expr Bool  -- Int == Int = Bool
  If      :: Expr Bool -> Expr a -> Expr a -> Expr a  -- 조건은 반드시 Bool
```

이제 `Add (BoolLit True) (Lit 42)`는 **컴파일 에러**가 됩니다. `Add`의 첫 번째 인자는 `Expr Int`여야 하는데 `BoolLit True`는 `Expr Bool`이기 때문입니다.

### 타입 정제(Type Refinement)

GADT의 핵심 메커니즘은 패턴 매칭 시 **타입 변수가 더 구체적인 타입으로 정제(refined)**된다는 것입니다.

```haskell
eval :: Expr a -> a
eval (Lit n)       = n          -- 여기서 a는 Int로 정제
eval (BoolLit b)   = b          -- 여기서 a는 Bool로 정제
eval (Add e1 e2)   = eval e1 + eval e2  -- 여기서 a는 Int
eval (Mul e1 e2)   = eval e1 * eval e2
eval (Eq e1 e2)    = eval e1 == eval e2  -- a는 Bool
eval (If c t e)
  | eval c    = eval t
  | otherwise = eval e
```

이 `eval` 함수는 **타입 안전하며 완전(total)** 합니다. `a`가 `Int`인 경우와 `Bool`인 경우를 컴파일러가 추적하므로, 각 케이스의 반환 타입이 자동으로 올바르게 처리됩니다.

---

## 왜 필요한가

### 1. 불가능한 상태를 타입으로 표현

가장 고전적인 예시는 길이가 타입에 인코딩된 벡터(Vec)입니다.

```haskell
-- 자연수를 타입 레벨에서 표현
data Nat = Zero | Succ Nat

-- 길이가 타입 레벨에서 추적되는 벡터
data Vec (n :: Nat) a where
  Nil  :: Vec 'Zero a
  Cons :: a -> Vec n a -> Vec ('Succ n) a

-- 타입 안전한 head: 빈 벡터에 head를 호출할 수 없음
head :: Vec ('Succ n) a -> a
head (Cons x _) = x
-- head Nil은 컴파일 에러!
```

빈 벡터에 `head`를 호출하면 `'Zero`와 `'Succ n`이 일치하지 않아 컴파일 타임에 오류가 발생합니다.

### 2. 프로토콜 상태 기계 모델링

```haskell
data ConnectionState = Closed | Open | Authenticated

data Connection (s :: ConnectionState) where
  NewConn    :: Connection 'Closed
  Connect    :: Connection 'Closed -> Connection 'Open
  Auth       :: String -> Connection 'Open -> Connection 'Authenticated
  SendData   :: String -> Connection 'Authenticated -> IO ()
  Disconnect :: Connection s -> Connection 'Closed
```

이렇게 하면 `Auth` 없이 `SendData`를 호출하거나, 이미 열린 연결에 다시 `Connect`하는 오류를 컴파일 타임에 잡을 수 있습니다.

---

## 실제 구현 예제

### 예제 1: Haskell - 완전한 GADT 기반 표현식 평가기

```haskell
{-# LANGUAGE GADTs, DataKinds, KindSignatures, StandaloneDeriving #-}

module TypedExprEval where

import Data.Kind (Type)

-- 타입 안전 표현식 AST
data Expr :: Type -> Type where
  -- 리터럴
  LitI   :: Int  -> Expr Int
  LitB   :: Bool -> Expr Bool
  LitD   :: Double -> Expr Double
  
  -- 산술 연산
  AddI   :: Expr Int    -> Expr Int    -> Expr Int
  MulI   :: Expr Int    -> Expr Int    -> Expr Int
  AddD   :: Expr Double -> Expr Double -> Expr Double
  
  -- 비교 연산
  EqI    :: Expr Int  -> Expr Int  -> Expr Bool
  LtI    :: Expr Int  -> Expr Int  -> Expr Bool
  
  -- 논리 연산
  And    :: Expr Bool -> Expr Bool -> Expr Bool
  Or     :: Expr Bool -> Expr Bool -> Expr Bool
  Not    :: Expr Bool -> Expr Bool
  
  -- 조건식
  If     :: Expr Bool -> Expr a -> Expr a -> Expr a
  
  -- 형변환
  ItoD   :: Expr Int -> Expr Double  -- Int to Double

-- 완전하고 타입 안전한 평가기
eval :: Expr a -> a
eval (LitI n)      = n
eval (LitB b)      = b
eval (LitD d)      = d
eval (AddI e1 e2)  = eval e1 + eval e2
eval (MulI e1 e2)  = eval e1 * eval e2
eval (AddD e1 e2)  = eval e1 + eval e2
eval (EqI e1 e2)   = eval e1 == eval e2
eval (LtI e1 e2)   = eval e1 < eval e2
eval (And e1 e2)   = eval e1 && eval e2
eval (Or  e1 e2)   = eval e1 || eval e2
eval (Not e)       = not (eval e)
eval (If c t e)    = if eval c then eval t else eval e
eval (ItoD e)      = fromIntegral (eval e)

-- 타입 안전한 Pretty Printer
prettyPrint :: Expr a -> String
prettyPrint (LitI n)      = show n
prettyPrint (LitB b)      = show b
prettyPrint (LitD d)      = show d
prettyPrint (AddI e1 e2)  = "(" ++ prettyPrint e1 ++ " + " ++ prettyPrint e2 ++ ")"
prettyPrint (MulI e1 e2)  = "(" ++ prettyPrint e1 ++ " * " ++ prettyPrint e2 ++ ")"
prettyPrint (EqI e1 e2)   = "(" ++ prettyPrint e1 ++ " == " ++ prettyPrint e2 ++ ")"
prettyPrint (LtI e1 e2)   = "(" ++ prettyPrint e1 ++ " < " ++ prettyPrint e2 ++ ")"
prettyPrint (And e1 e2)   = "(" ++ prettyPrint e1 ++ " && " ++ prettyPrint e2 ++ ")"
prettyPrint (Or e1 e2)    = "(" ++ prettyPrint e1 ++ " || " ++ prettyPrint e2 ++ ")"
prettyPrint (Not e)       = "(not " ++ prettyPrint e ++ ")"
prettyPrint (If c t e)    = "(if " ++ prettyPrint c ++ " then " ++ prettyPrint t ++ " else " ++ prettyPrint e ++ ")"
prettyPrint (ItoD e)      = "(toDouble " ++ prettyPrint e ++ ")"

-- 사용 예시
example :: IO ()
example = do
  -- (3 + 4) * 2 = 14
  let expr1 = MulI (AddI (LitI 3) (LitI 4)) (LitI 2)
  putStrLn $ prettyPrint expr1 ++ " = " ++ show (eval expr1)
  
  -- if (3 < 5) then true && false else false = false
  let expr2 = If (LtI (LitI 3) (LitI 5))
                 (And (LitB True) (LitB False))
                 (LitB False)
  putStrLn $ prettyPrint expr2 ++ " = " ++ show (eval expr2)
  
  -- 아래는 컴파일 에러: Bool + Int는 불가
  -- let bad = AddI (LitB True) (LitI 42)
```

### 예제 2: Rust에서 PhantomData로 GADT 개념 구현

Rust는 GADTs를 직접 지원하지 않지만, **PhantomData**와 **트레이트 경계**를 활용하여 유사한 타입 안전성을 달성할 수 있습니다.

```rust
use std::marker::PhantomData;

// 타입 마커: 컴파일 타임에만 존재, 런타임 비용 0
struct Int;
struct Bool;
struct Double;

// GADT-스타일 표현식 (PhantomData로 타입 추적)
enum Expr<T> {
    // 리터럴
    LitI(i64, PhantomData<Int>),
    LitB(bool, PhantomData<Bool>),
    LitD(f64, PhantomData<Double>),
    
    // 산술 (Int만)
    AddI(Box<Expr<Int>>, Box<Expr<Int>>, PhantomData<Int>),
    MulI(Box<Expr<Int>>, Box<Expr<Int>>, PhantomData<Int>),
    
    // 비교 → Bool 반환
    EqI(Box<Expr<Int>>, Box<Expr<Int>>, PhantomData<Bool>),
    LtI(Box<Expr<Int>>, Box<Expr<Int>>, PhantomData<Bool>),
    
    // 논리
    And(Box<Expr<Bool>>, Box<Expr<Bool>>, PhantomData<Bool>),
    Not(Box<Expr<Bool>>, PhantomData<Bool>),
    
    // 조건
    IfExpr {
        cond: Box<Expr<Bool>>,
        then_e: Box<Expr<T>>,
        else_e: Box<Expr<T>>,
        phantom: PhantomData<T>,
    },
}

// 스마트 생성자
fn lit_i(n: i64) -> Expr<Int> {
    Expr::LitI(n, PhantomData)
}

fn lit_b(b: bool) -> Expr<Bool> {
    Expr::LitB(b, PhantomData)
}

fn add_i(e1: Expr<Int>, e2: Expr<Int>) -> Expr<Int> {
    Expr::AddI(Box::new(e1), Box::new(e2), PhantomData)
}

fn mul_i(e1: Expr<Int>, e2: Expr<Int>) -> Expr<Int> {
    Expr::MulI(Box::new(e1), Box::new(e2), PhantomData)
}

fn eq_i(e1: Expr<Int>, e2: Expr<Int>) -> Expr<Bool> {
    Expr::EqI(Box::new(e1), Box::new(e2), PhantomData)
}

fn lt_i(e1: Expr<Int>, e2: Expr<Int>) -> Expr<Bool> {
    Expr::LtI(Box::new(e1), Box::new(e2), PhantomData)
}

fn and_b(e1: Expr<Bool>, e2: Expr<Bool>) -> Expr<Bool> {
    Expr::And(Box::new(e1), Box::new(e2), PhantomData)
}

fn not_b(e: Expr<Bool>) -> Expr<Bool> {
    Expr::Not(Box::new(e), PhantomData)
}

fn if_expr<T>(cond: Expr<Bool>, then_e: Expr<T>, else_e: Expr<T>) -> Expr<T> {
    Expr::IfExpr {
        cond: Box::new(cond),
        then_e: Box::new(then_e),
        else_e: Box::new(else_e),
        phantom: PhantomData,
    }
}

// 타입별 평가 트레이트
trait Eval {
    type Output;
    fn eval(&self) -> Self::Output;
}

impl Eval for Expr<Int> {
    type Output = i64;
    fn eval(&self) -> i64 {
        match self {
            Expr::LitI(n, _) => *n,
            Expr::AddI(e1, e2, _) => e1.eval() + e2.eval(),
            Expr::MulI(e1, e2, _) => e1.eval() * e2.eval(),
            Expr::IfExpr { cond, then_e, else_e, .. } => {
                if cond.eval() { then_e.eval() } else { else_e.eval() }
            }
            _ => unreachable!("type safety guarantees this never happens"),
        }
    }
}

impl Eval for Expr<Bool> {
    type Output = bool;
    fn eval(&self) -> bool {
        match self {
            Expr::LitB(b, _) => *b,
            Expr::EqI(e1, e2, _) => e1.eval() == e2.eval(),
            Expr::LtI(e1, e2, _) => e1.eval() < e2.eval(),
            Expr::And(e1, e2, _) => e1.eval() && e2.eval(),
            Expr::Not(e, _) => !e.eval(),
            Expr::IfExpr { cond, then_e, else_e, .. } => {
                if cond.eval() { then_e.eval() } else { else_e.eval() }
            }
            _ => unreachable!(),
        }
    }
}

fn main() {
    // (3 + 4) * 2 = 14
    let expr1 = mul_i(add_i(lit_i(3), lit_i(4)), lit_i(2));
    println!("(3 + 4) * 2 = {}", expr1.eval());

    // if (10 < 20) then 100 else 200 = 100
    let expr2 = if_expr(lt_i(lit_i(10), lit_i(20)), lit_i(100), lit_i(200));
    println!("if (10 < 20) then 100 else 200 = {}", expr2.eval());

    // !(3 == 4) && (1 < 2) = true
    let expr3 = and_b(not_b(eq_i(lit_i(3), lit_i(4))), lt_i(lit_i(1), lit_i(2)));
    println!("!(3==4) && (1<2) = {}", expr3.eval());
    
    // 아래는 컴파일 에러: add_i는 Expr<Int> 인자만 받음
    // let bad = add_i(lit_b(true), lit_i(42));
}
```

출력 결과:
```
(3 + 4) * 2 = 14
if (10 < 20) then 100 else 200 = 100
!(3==4) && (1<2) = true
```

---

## 주의사항과 팁

### 1. GHC 확장 활성화

Haskell에서 GADT를 사용하려면 `{-# LANGUAGE GADTs #-}` 확장이 필요합니다. 이 확장은 자동으로 `GADTSyntax`와 `MonoLocalBinds`도 활성화합니다. `MonoLocalBinds`는 `let`/`where` 바인딩의 타입 일반화를 제한하므로 의도치 않은 동작이 발생할 수 있습니다.

### 2. 존재 타입(Existential Types)과의 관계

GADT는 존재 타입을 자연스럽게 표현합니다.

```haskell
-- 타입 정보를 런타임에 보존하는 Dynamic 값
data Dynamic where
  Dyn :: Typeable a => a -> Dynamic

-- 꺼낼 때 타입 증거를 활용
fromDynamic :: Typeable a => Dynamic -> Maybe a
fromDynamic (Dyn x) = cast x
```

### 3. 패턴 매칭 순서 주의

GHC는 여러 패턴을 왼쪽에서 오른쪽으로 타입 검사합니다. 함수 인자 순서를 바꾸면 타입 추론이 실패할 수 있습니다. 이런 경우 명시적 타입 어노테이션을 추가하세요.

### 4. 타입 안전성의 한계: 외부 데이터 파싱

GADT 기반 AST는 소스 코드에서 직접 구성할 때만 타입 안전을 보장합니다. JSON이나 문자열에서 파싱할 때는 `Either`나 존재 타입으로 감싼 뒤 런타임 타입 검사가 필요합니다.

```haskell
-- 파싱된 (타입 미확정) 표현식
data UntypedExpr
  = ULit Int
  | UBoolLit Bool
  | UAdd UntypedExpr UntypedExpr
  | UIf UntypedExpr UntypedExpr UntypedExpr

-- 타입 검사 후 GADT로 변환
data TypedExpr where
  TypedI :: Expr Int  -> TypedExpr
  TypedB :: Expr Bool -> TypedExpr

typeCheck :: UntypedExpr -> Either String TypedExpr
typeCheck (ULit n)         = Right (TypedI (LitI n))
typeCheck (UBoolLit b)     = Right (TypedB (LitB b))
typeCheck (UAdd e1 e2) = do
  t1 <- typeCheck e1
  t2 <- typeCheck e2
  case (t1, t2) of
    (TypedI i1, TypedI i2) -> Right (TypedI (AddI i1 i2))
    _ -> Left "Type error: Add requires Int operands"
typeCheck (UIf c t e) = do
  tc <- typeCheck c
  tt <- typeCheck t
  te <- typeCheck e
  case (tc, tt, te) of
    (TypedB bc, TypedI ti, TypedI ei) -> Right (TypedI (If bc ti ei))
    (TypedB bc, TypedB tb, TypedB eb) -> Right (TypedB (If bc tb eb))
    _ -> Left "Type error: If branches must have same type, condition must be Bool"
```

### 5. 성능 고려사항

GADT는 타입 에러 증거(type evidence, coercion)를 런타임에 가지고 다닐 수 있어 미묘한 오버헤드가 발생할 수 있습니다. GHC는 대부분의 경우 이를 최적화하여 제거하지만, `-O2` 플래그 없이는 남아있을 수 있습니다.

### 6. TypeScript의 Discriminated Unions

TypeScript는 GADTs를 직접 지원하지 않지만, **판별 유니온(Discriminated Unions)** 으로 비슷한 효과를 낼 수 있습니다.

```typescript
type Expr<T> =
  | { kind: 'lit_i'; value: number; _type: T extends number ? T : never }
  | { kind: 'lit_b'; value: boolean; _type: T extends boolean ? T : never }
  // ...

function eval<T>(expr: Expr<T>): T { /* ... */ }
```

다만 TypeScript의 타입 시스템은 Haskell만큼 정교하지 않아 일부 불변식은 표현할 수 없습니다.

---

## 참고 자료

- [GHC Mirror - Glasgow Haskell Compiler](https://github.com/ghc/ghc)
- [typelevel/cats: Lightweight, modular Scala functional programming library](https://github.com/typelevel/cats)
