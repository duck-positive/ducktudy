---
layout: post
title: "선형 타입과 부구조적 타입 시스템 완전 정복: 자원을 정확히 한 번 사용하는 타입 이론"
date: 2026-11-12
categories: [cs, computer-science]
tags: [type-system, linear-types, substructural, affine-types, rust, haskell, programming-languages]
---

## 구조적 타입 규칙이란 무엇인가

전통적인 타입 시스템에는 세 가지 **구조 규칙(structural rules)**이 암묵적으로 존재한다.

| 규칙 | 의미 | 예시 |
|------|------|------|
| **교환(Exchange)** | 타입 환경에서 변수 순서를 바꿀 수 있다 | `x:A, y:B ⊢ e` ↔ `y:B, x:A ⊢ e` |
| **약화(Weakening)** | 사용하지 않는 변수를 환경에 추가할 수 있다 | `Γ ⊢ e` → `Γ, x:A ⊢ e` |
| **수축(Contraction)** | 같은 변수를 여러 번 사용할 수 있다 | `x:A, x:A ⊢ e` → `x:A ⊢ e` |

**부구조적 타입 시스템(Substructural Type Systems)**은 이 세 규칙 중 일부를 제거하거나 제한함으로써 자원(resource) 사용 방식에 강력한 정적 보증을 제공한다.

| 시스템 | 교환 | 약화 | 수축 | 사용 횟수 |
|--------|------|------|------|----------|
| 순서 타입(Ordered) | ✗ | ✗ | ✗ | 정확히 1번, 순서대로 |
| **선형 타입(Linear)** | ✓ | ✗ | ✗ | **정확히 1번** |
| 아핀 타입(Affine) | ✓ | ✓ | ✗ | **최대 1번** |
| 관련 타입(Relevant) | ✓ | ✗ | ✓ | 최소 1번 이상 |
| 일반 타입(Unrestricted) | ✓ | ✓ | ✓ | 제한 없음 |

이 중 가장 중요하고 실용적인 것이 **선형 타입(Linear Types)**과 **아핀 타입(Affine Types)**이다.

## 선형 타입이 필요한 이유

### 자원 관리 문제

프로그래밍에서 "자원(resource)"은 파일 핸들, 소켓, 뮤텍스, 메모리 버퍼처럼 **정확히 한 번만 올바르게 처리해야 하는 값**이다. 일반 타입 시스템에서는 이를 강제할 방법이 없다.

```python
# 일반 타입 시스템에서의 자원 관리 오류 예시
def bad_usage():
    f = open("data.txt")
    data = f.read()
    f.close()
    
    # 버그: 이미 닫힌 파일을 다시 사용!
    more = f.read()   # 런타임 오류 → ValueError: I/O operation on closed file
    
    # 버그: 파일을 닫지 않고 함수 종료!
    g = open("other.txt")
    return data  # g의 close() 누락 → 리소스 누수
```

선형 타입 시스템이 있다면 이런 오류는 **컴파일 타임에 차단**된다. 선형 값은:
- 사용(소비)하지 않으면 타입 오류 (약화 규칙 없음)
- 두 번 이상 사용하면 타입 오류 (수축 규칙 없음)

### 동시성과 프로토콜 안전성

선형 타입은 채널 통신 프로토콜의 순서를 타입 수준에서 강제할 수 있다. Session Types(세션 타입)가 선형 타입을 기반으로 구축되는 이유다.

### 메모리 안전성 (Rust의 소유권 모델)

Rust는 아핀 타입(Affine Types)을 실용화한 언어다. 값은 최대 한 번만 사용(이동)될 수 있으며, `Copy` 트레이트로 명시적으로 허용하지 않는 한 복제(수축)가 금지된다.

## 선형 논리의 연결: Per Martin-Löf의 유산

선형 타입은 Jean-Yves Girard가 1987년 제안한 **선형 논리(Linear Logic)**에 직접 대응한다. Curry-Howard 대응에 의해 타입은 명제이고 프로그램은 증명이다.

| 선형 논리 | 타입 이론 | 의미 |
|-----------|----------|------|
| ⊗ (times) | A ⊗ B | 곱 타입 (두 자원을 동시에 가짐) |
| ⅋ (par) | A ⅋ B | 공곱 (동시에 두 자원 중 하나 선택 대기) |
| ! (bang) | !A | 비선형 타입 (무제한 복제 허용) |
| & (with) | A & B | 외부 선택 |
| ⊕ (plus) | A ⊕ B | 내부 선택 |

`!A` 타입(bang 타입)은 "이 자원은 선형 세계에서 무제한으로 복제할 수 있다"는 명시적 허가다.

## 실제 구현 예제

### 예제 1: Rust의 소유권으로 보는 아핀 타입

Rust는 아핀 타입 시스템을 갖고 있다. 각 값은 최대 한 번만 "이동(move)"될 수 있다.

```rust
fn main() {
    // ─── 소유권 이동(move): 아핀 사용 ───────────────────────────────
    let s1 = String::from("hello");  // s1이 String의 소유자
    let s2 = s1;                     // 소유권이 s2로 이동; s1은 무효화

    // println!("{}", s1);  // 컴파일 오류!
    // → error[E0382]: borrow of moved value: `s1`
    println!("{}", s2);  // 정상

    // ─── 복제(clone): 명시적 수축(contraction) ─────────────────────
    let s3 = String::from("world");
    let s4 = s3.clone();  // 명시적으로 복제 허용
    println!("{} {}", s3, s4);  // 두 값 모두 유효

    // ─── Copy 타입: 암묵적 수축 허용 ───────────────────────────────
    let x: i32 = 42;    // i32는 Copy 트레이트 구현
    let y = x;           // 복사(수축), x는 여전히 유효
    println!("{} {}", x, y);

    // ─── 약화 금지: Rust는 미사용 변수를 경고한다 ──────────────────
    let _unused = String::from("dropped");
    // 약화는 허용하되 경고를 발생시킴 (완전한 선형 타입보다 느슨)
    // Drop 트레이트로 소멸자가 자동 호출됨

    // ─── 파일 핸들 자원 관리 ────────────────────────────────────────
    use std::fs::File;
    use std::io::Write;
    {
        let mut f = File::create("test.txt").unwrap();
        f.write_all(b"hello").unwrap();
        // 블록을 벗어나면 f의 소유권이 소멸 → Drop 호출 → close()
    }
    // f를 사용하려면 컴파일 오류 (스코프 밖)
}

// ─── 함수에서의 소유권 이동 ──────────────────────────────────────────
fn consume(s: String) {   // s의 소유권을 가져감
    println!("Consuming: {}", s);
}  // s는 여기서 drop됨

fn borrow(s: &str) {      // 불변 대여 (약화와 수축 모두 허용)
    println!("Borrowing: {}", s);
}

fn modify(s: &mut String) {  // 가변 대여 (유일한 접근 보장)
    s.push_str(" world");
}
```

### 예제 2: Haskell 선형 타입 확장 (Linear Haskell)

GHC 9.0부터 Haskell은 선형 타입을 지원한다. `%1 ->` 화살표는 "이 함수는 인자를 정확히 한 번 사용한다"는 의미다.

```haskell
{-# LANGUAGE LinearTypes #-}
{-# LANGUAGE QualifiedDo #-}
import Prelude.Linear
import qualified System.IO.Linear as LIO

-- 선형 함수: a를 정확히 한 번 사용
linearId :: a %1 -> a
linearId x = x   -- x를 정확히 한 번 사용 → 선형 타입 충족

-- 선형 함수는 인자를 복제하거나 버릴 수 없다
-- 아래 코드는 컴파일 오류
-- linearDrop :: a %1 -> ()
-- linearDrop _ = ()   -- ERROR: 선형 인자를 버릴 수 없음 (약화 금지)

-- linearDup :: a %1 -> (a, a)
-- linearDup x = (x, x)  -- ERROR: 선형 인자를 복제할 수 없음 (수축 금지)

-- 선형 쌍 (multiplicative conjunction, A ⊗ B)
swap :: (a, b) %1 -> (b, a)
swap (x, y) = (y, x)   -- 두 선형 값을 각각 정확히 한 번 사용

-- 선형 파일 핸들: 반드시 닫아야 하는 타입 보증
-- (실제 Linear Haskell에서의 패턴)
withLinearFile :: FilePath
               -> (LIO.Handle %1 -> LIO.IO ())
               -> IO ()
withLinearFile path action = LIO.run $ do
    h <- LIO.openFile path LIO.ReadMode
    action h   -- h를 정확히 한 번 사용해야 함
    -- action이 h를 닫지 않으면 컴파일 오류!

-- 비선형 타입으로 승격: Ur (unrestricted)
-- !A에 해당하는 타입
data Ur a where
    Ur :: a -> Ur a   -- a를 비선형 세계에서 선형 세계로 가져옴

-- Ur을 사용하면 선형 컨텍스트에서 복제 가능
copyUr :: Ur a %1 -> (Ur a, Ur a)
copyUr (Ur x) = (Ur x, Ur x)   -- Ur 안의 값은 비선형이므로 복제 허용
```

## 선형 타입과 아핀 타입의 차이

가장 많이 헷갈리는 부분이 선형(linear)과 아핀(affine)의 차이다.

- **선형 타입**: 약화(weakening)와 수축(contraction) 모두 금지. 값을 **정확히 한 번** 사용해야 한다. 사용하지 않으면 타입 오류.
- **아핀 타입**: 수축만 금지. 값을 **최대 한 번** 사용할 수 있다. 사용하지 않아도 된다(암묵적으로 버려짐).

Rust는 아핀 시스템이다. 사용하지 않은 값은 `Drop`을 통해 자동 소멸된다. 완전한 선형 시스템이라면 미사용 선형 값도 타입 오류가 되어야 하지만, 실용성을 위해 약화를 허용하되 소멸자를 호출하는 방식을 택했다.

## 연관 개념: 고유성 타입과 소유권 타입

| 언어 | 메커니즘 | 계열 |
|------|---------|------|
| Rust | 소유권(Ownership) + 대여(Borrowing) | 아핀 타입 |
| Clean | 고유성 타입(Uniqueness Types) | 아핀 타입 |
| Linear Haskell | 선형 화살표(`%1 ->`) | 선형 타입 |
| Idris 2 | 수량 타입(Quantitative Type Theory) | 선형/아핀/비선형 통합 |
| ATS | 선형 의존 타입 | 선형 타입 + 의존 타입 |

## 주의사항과 실전 팁

### 1. 선형 타입은 "정확히 한 번"을 타입 수준에서 강제한다

분기(if/else)에서 주의가 필요하다. 양쪽 분기에서 모두 선형 값을 소비하거나, 양쪽 분기에서 모두 소비하지 않아야 한다.

```haskell
-- 올바른 패턴: 양쪽 분기에서 선형 값 소비
useLinear :: Bool -> a %1 -> b %1 -> (a, b)
useLinear cond x y
    | cond    = (x, y)   -- x, y 각각 한 번 사용
    | otherwise = (x, y) -- 동일
-- 만약 한쪽에서 x를 버린다면 타입 오류
```

### 2. Rust의 대여(borrow)는 선형성을 "일시 정지"한다

Rust의 불변 대여(`&T`)는 여러 곳에서 동시에 가능하고, 가변 대여(`&mut T`)는 유일하게 허용된다. 이는 선형 타입의 소유권을 임시로 나눠주는 메커니즘이다. 대여가 끝나면 원래 소유자에게 소유권이 돌아온다.

### 3. 성능 이점

선형 타입 시스템에서 컴파일러는 "이 값이 더 이상 사용되지 않는다"는 사실을 타입 수준에서 알 수 있다. 이를 활용하면 가비지 컬렉션 없이 안전한 메모리 관리가 가능하고, 불변 자료구조를 선형으로 갱신(in-place mutation)하는 최적화도 가능하다.

### 4. 실용적 채택의 어려움

순수 선형 타입은 프로그래밍하기 불편하다. 로그를 찍기 위해 값을 "사용"하면 그 값을 반환해야 하고, 조건문마다 모든 선형 값의 사용을 명시해야 한다. 이런 불편함 때문에 Rust처럼 아핀 타입 + 대여 검사기를 결합하는 절충안이 실용적으로 선택된다.

## 참고 자료

- [Substructural Type Systems — David Walker, in ATTAPL (MIT Press, 2002)](https://www.cs.cmu.edu/~./rwh/papers/lts/lts.pdf)
- [Linear Haskell: Practical Linearity in a Higher-Order Polymorphic Language (Bernardy et al., POPL 2018)](https://arxiv.org/abs/1710.09756)
- [GHC User's Guide: Linear Types](https://ghc.gitlab.haskell.org/ghc/doc/users_guide/exts/linear_types.html)
- [Wikipedia: Substructural Type System](https://en.wikipedia.org/wiki/Substructural_type_system)
