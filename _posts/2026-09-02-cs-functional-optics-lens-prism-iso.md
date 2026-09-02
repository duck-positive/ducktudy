---
layout: post
title: "함수형 Optics 완전 정복: Lens·Prism·Iso로 불변 데이터를 우아하게 다루는 법"
date: 2026-09-02
categories: [cs, computer-science]
tags: [optics, lens, prism, iso, functional-programming, haskell, scala, monocle, immutability, profunctor]
---

## 개념 설명

Optics(광학)는 **중첩된 불변 자료구조를 함수형으로 조회·수정·변환**하기 위한 추상화 라이브러리입니다. "광학"이라는 이름은 데이터 구조의 특정 부분에 "초점(focus)"을 맞춰 들여다보거나 변경한다는 은유에서 비롯됩니다.

Optics는 1985년경 Haskell 커뮤니티에서 등장하여 Simon Peyton Jones, Edward Kmett 등에 의해 발전했습니다. 현재는 Haskell의 `lens` 라이브러리, Scala의 `Monocle`, Kotlin의 `Arrow-Optics`, PureScript의 `purescript-profunctor-lenses`가 대표적인 구현체입니다.

### Optics의 종류

```
Iso ──→ Lens ──→ Optional ──→ Traversal
  └──→ Prism ──→ Optional
```

| 종류 | 초점 대상 | 읽기 | 쓰기 | 예시 |
|------|-----------|------|------|------|
| **Iso** | 1개 (항상 성공) | 항상 | 항상 | `String ↔ List[Char]` |
| **Lens** | 1개 (항상 성공) | 항상 | 항상 | `User.name` |
| **Prism** | 0 또는 1개 | 조건부 | 조건부 | `Either.Left`, `Option.Some` |
| **Optional** | 0 또는 1개 | 조건부 | 항상 | `Map.at(key)` |
| **Traversal** | 0개 이상 | 항상 | 항상 | `List.each` |
| **Fold** | 0개 이상 | 읽기 전용 | — | `List.folded` |
| **Setter** | 0개 이상 | — | 쓰기 전용 | `mapped` |

## 왜 필요한가

### 불변 데이터의 수정 문제

함수형 프로그래밍에서는 불변(immutable) 자료구조를 선호합니다. 그런데 중첩된 구조의 깊은 필드를 수정할 때 코드가 매우 장황해집니다.

```kotlin
// 중첩된 불변 데이터 클래스
data class Address(val street: String, val city: String, val zipCode: String)
data class User(val name: String, val age: Int, val address: Address)

val user = User(
    name = "Alice",
    age = 30,
    address = Address(street = "강남대로 1", city = "서울", zipCode = "06234")
)

// 전통적 copy 방식: 중첩이 깊어질수록 복잡
val updatedUser = user.copy(
    address = user.address.copy(
        city = "부산"
    )
)

// Optics 방식: 간결하고 합성 가능
// val userCityLens = User.address compose Address.city
// val updatedUser2 = userCityLens.set(user, "부산")
```

세 단계만 중첩돼도 `copy`가 중첩됩니다. 다섯 단계를 넘어가면 코드가 읽기 불가능해집니다.

### Optics의 핵심 가치

1. **합성(Composition)** — `userLens compose addressLens compose cityLens`처럼 체인
2. **재사용** — 동일한 렌즈를 `get`, `set`, `modify`, `over` 등에 재사용
3. **타입 안전성** — 컴파일 타임에 필드 존재 여부를 검증
4. **DRY 원칙** — 접근 경로를 한 번만 정의

## 실제 구현 예제

### 예제 1: Haskell `lens` 라이브러리로 Lens 이해하기

```haskell
{-# LANGUAGE TemplateHaskell #-}
import Control.Lens

-- 데이터 구조 정의
data Address = Address
    { _street  :: String
    , _city    :: String
    , _zipCode :: String
    } deriving (Show)

data User = User
    { _name    :: String
    , _age     :: Int
    , _address :: Address
    } deriving (Show)

-- TemplateHaskell로 렌즈 자동 생성 (언더스코어 prefix 컨벤션)
makeLenses ''Address
makeLenses ''User

-- Lens 합성: User → Address → city 경로
userCity :: Lens' User String
userCity = address . city  -- (.) 연산자로 Lens 합성

example :: IO ()
example = do
    let user = User
            { _name = "Alice"
            , _age  = 30
            , _address = Address
                { _street  = "강남대로 1"
                , _city    = "서울"
                , _zipCode = "06234"
                }
            }

    -- get: 값 읽기
    putStrLn $ "도시: " ++ (user ^. userCity)  -- "서울"

    -- set: 값 교체 (불변)
    let user2 = user & userCity .~ "부산"
    putStrLn $ "변경 후: " ++ (user2 ^. userCity)  -- "부산"

    -- over/modify: 값 변환
    let user3 = user & age %~ (+1)
    putStrLn $ "나이: " ++ show (user3 ^. age)  -- 31

    -- 원본 불변 보장
    putStrLn $ "원본 도시: " ++ (user ^. userCity)  -- "서울" (변화 없음)
```

### 예제 2: Lens의 수학적 구조

Lens는 다음 두 함수의 쌍으로 정의됩니다.

```haskell
-- Lens s a = (s → a, s → a → s)
-- s: 큰 구조 타입, a: 초점 타입

data Lens' s a = Lens'
    { view :: s -> a        -- getter: 구조에서 값 추출
    , set  :: s -> a -> s   -- setter: 구조에서 값 교체
    }

-- 수동으로 Lens 생성
nameLens :: Lens' User String
nameLens = Lens'
    { view = _name
    , set  = \user newName -> user { _name = newName }
    }

-- Lens가 만족해야 할 3가지 법칙 (Lens Laws)
-- 1. get-set: view (set s a) = a
-- 2. set-get: set s (view s) = s
-- 3. set-set: set (set s a1) a2 = set s a2

verifyLensLaws :: Bool
verifyLensLaws =
    let u = User "Alice" 30 (Address "강남대로" "서울" "06234")
        l = nameLens
        a = "Bob"
    in
    -- 법칙 1: 설정한 값을 읽으면 그 값이 나온다
    (view l (set l u a) == a) &&
    -- 법칙 2: 읽은 값으로 다시 설정하면 원본과 같다
    (set l u (view l u) == u) &&
    -- 법칙 3: 두 번 설정하면 마지막 설정만 남는다
    (set l (set l u "Charlie") a == set l u a)
```

### 예제 3: Prism으로 합 타입(Sum Type) 다루기

Prism은 **합 타입(Sum Type, 즉 `Either`, `Maybe`, `sealed class`)의 특정 케이스에 초점**을 맞추는 Optics입니다.

```scala
// Scala + Monocle 라이브러리 예시
import monocle.Prism

// sealed trait: 합 타입
sealed trait Shape
case class Circle(radius: Double) extends Shape
case class Rectangle(width: Double, height: Double) extends Shape
case class Triangle(base: Double, height: Double) extends Shape

// Circle 케이스에 대한 Prism
val circlePrism: Prism[Shape, Circle] = Prism[Shape, Circle] {
    case c: Circle => Some(c)  // Circle이면 Some으로 추출
    case _         => None     // 다른 케이스이면 None
}(identity)  // Circle → Shape 복원

// Rectangle에 대한 Prism
val rectanglePrism: Prism[Shape, Rectangle] = Prism[Shape, Rectangle] {
    case r: Rectangle => Some(r)
    case _            => None
}(identity)

// 사용 예시
val circle: Shape = Circle(5.0)
val rect: Shape = Rectangle(4.0, 6.0)

// getOption: 해당 케이스이면 Some, 아니면 None
println(circlePrism.getOption(circle))  // Some(Circle(5.0))
println(circlePrism.getOption(rect))    // None

// modify: 해당 케이스일 때만 수정
val biggerCircle = circlePrism.modify(c => c.copy(radius = c.radius * 2))(circle)
println(biggerCircle)  // Circle(10.0)

val unchangedRect = circlePrism.modify(c => c.copy(radius = c.radius * 2))(rect)
println(unchangedRect)  // Rectangle(4.0, 6.0) — 변화 없음

// Prism 법칙:
// 1. getOption-reverseGet: getOption(reverseGet(a)) == Some(a)
// 2. reverseGet-getOption: getOption(s).fold(s)(reverseGet) == s
```

### 예제 4: Iso — 무손실 변환

Iso는 두 타입 사이의 **완전한 가역 변환(isomorphism)**을 표현합니다. `Lens`와 `Prism`을 동시에 만족하는 가장 강력한 Optics입니다.

```haskell
import Control.Lens

-- String과 [Char]는 동치: Iso 정의
stringToList :: Iso' String [Char]
stringToList = iso id id  -- String IS [Char] in Haskell

-- 섭씨-화씨 변환 Iso
celsiusToFahrenheit :: Iso' Double Double
celsiusToFahrenheit = iso c2f f2c
  where
    c2f c = c * 9/5 + 32
    f2c f = (f - 32) * 5/9

example :: IO ()
example = do
    let temp = 100.0  -- 섭씨 100도

    -- view: 섭씨 → 화씨
    print $ temp ^. celsiusToFahrenheit  -- 212.0

    -- re (역방향 적용): 화씨 → 섭씨
    print $ 212.0 ^. re celsiusToFahrenheit  -- 100.0

    -- Iso는 Lens로도 사용 가능
    let temps = [0.0, 20.0, 37.0, 100.0]
    -- 모든 섭씨 온도를 화씨로 변환
    print $ temps ^.. traversed . celsiusToFahrenheit
    -- [32.0, 68.0, 98.6, 212.0]

-- 뉴타입 Iso (안전한 타입 변환)
newtype Celsius    = Celsius Double deriving (Show)
newtype Fahrenheit = Fahrenheit Double deriving (Show)

celsFahr :: Iso' Celsius Fahrenheit
celsFahr = iso (\(Celsius c) -> Fahrenheit (c * 9/5 + 32))
               (\(Fahrenheit f) -> Celsius ((f - 32) * 5/9))
```

### 예제 5: Kotlin + Arrow-Optics 실전 예제

```kotlin
// build.gradle.kts
// dependencies {
//     implementation("io.arrow-kt:arrow-optics:1.2.0")
//     kapt("io.arrow-kt:arrow-optics-ksp-plugin:1.2.0")
// }

import arrow.optics.*
import arrow.optics.dsl.*

@optics
data class Config(
    val database: DatabaseConfig,
    val server: ServerConfig
) {
    companion object
}

@optics
data class DatabaseConfig(
    val host: String,
    val port: Int,
    val poolSize: Int
) {
    companion object
}

@optics
data class ServerConfig(
    val host: String,
    val port: Int,
    val maxConnections: Int
) {
    companion object
}

fun main() {
    val config = Config(
        database = DatabaseConfig(host = "localhost", port = 5432, poolSize = 10),
        server = ServerConfig(host = "0.0.0.0", port = 8080, maxConnections = 100)
    )

    // Arrow-Optics가 @optics 어노테이션으로 렌즈를 자동 생성
    // Config.database.host: Lens<Config, String>

    // 단순 읽기
    val dbHost = Config.database.host.get(config)
    println("DB Host: $dbHost")  // localhost

    // 깊은 중첩 수정 — copy 중첩 없이 간결하게
    val updatedConfig = Config.database.poolSize.set(config, 20)
    println("Pool Size: ${Config.database.poolSize.get(updatedConfig)}")  // 20

    // modify: 기존 값을 기반으로 변환
    val scaledConfig = Config.server.maxConnections.modify(config) { it * 2 }
    println("Max Connections: ${Config.server.maxConnections.get(scaledConfig)}")  // 200

    // 원본 불변 보장
    println("원본 Pool Size: ${Config.database.poolSize.get(config)}")  // 10
}
```

## Profunctor Optics: 이론적 기반

현대 Optics 라이브러리는 **Profunctor**를 기반으로 구현됩니다. Profunctor는 두 타입 매개변수를 가지는 타입 클래스로, 첫 번째 매개변수는 반공변(contravariant), 두 번째는 공변(covariant)입니다.

```haskell
class Profunctor p where
    dimap :: (a -> b) -> (c -> d) -> p b c -> p a d
    -- 입력에는 함수를 적용(반공변), 출력에는 함수를 적용(공변)

-- Lens = p a b -> p s t 형태로 표현 가능
-- 이 표현의 핵심 장점: 일반 함수 합성(.)으로 모든 Optics를 합성할 수 있음!
type Optic p s t a b = p a b -> p s t

type Lens s t a b      = forall p. Strong p  => Optic p s t a b
type Prism s t a b     = forall p. Choice p  => Optic p s t a b
type Iso s t a b       = forall p. Profunctor p => Optic p s t a b
type Traversal s t a b = forall p. Wander p  => Optic p s t a b
```

이 표현의 핵심 장점은 모든 Optics가 **타입 클래스 제약만 다른 동일한 형태**를 가진다는 것입니다. Iso는 가장 약한 제약(Profunctor), Lens는 Strong, Prism은 Choice를 추가로 요구합니다. 따라서 Iso는 자동으로 Lens와 Prism 양쪽으로 사용 가능합니다.

## Optics vs 일반 접근자 비교

| 특성 | Getter/Setter 메서드 | Optics |
|------|---------------------|--------|
| 합성 | 수동 위임 필요 | `compose`/`(.)` 자동 |
| 재사용 | 로직 중복 | 한 번 정의, 어디서든 사용 |
| 컴파일 검증 | 런타임 오류 가능 | 컴파일 타임 타입 검사 |
| 일급 값 | 불가 | 변수에 저장, 함수 인자 전달 가능 |
| 법칙 | 암묵적 | 수학적으로 명시 (get-set, set-get, set-set) |

## 주의사항과 팁

1. **학습 곡선** — Profunctor 기반 Optics의 타입 시그니처는 처음엔 복잡합니다. `Lens' s a`(단순 Lens)부터 시작해 `Lens s t a b`(다형 Lens)로 확장하세요.

2. **성능** — Haskell의 `lens` 라이브러리는 GHC 최적화를 통해 일반 레코드 접근과 동등한 성능을 냅니다. Scala의 Monocle은 약간의 오버헤드가 있습니다.

3. **코드 생성** — Haskell은 `makeLenses` (TemplateHaskell), Scala는 `@Lenses` 어노테이션, Kotlin/Arrow는 `@optics` + KSP를 활용해 보일러플레이트를 제거하세요.

4. **법칙 테스트** — Lens/Prism/Iso 법칙을 property-based testing으로 자동 검증할 수 있습니다. Haskell `hedgehog`, Scala `ScalaCheck`를 활용하세요.

5. **언제 Optics가 과할까** — 중첩이 2단계 이하이면 단순 `copy`가 더 읽기 쉽습니다. 3단계 이상의 중첩, 또는 동일 접근 경로를 여러 곳에서 재사용할 때 Optics 도입을 검토하세요.

6. **Traversal로 컬렉션 다루기** — `each` Traversal은 컬렉션의 모든 원소를 일괄 수정합니다. `filtered` Traversal은 조건을 만족하는 원소만 선택합니다.

```haskell
-- 모든 사용자의 나이를 1씩 증가 (Traversal)
let users = [User "Alice" 30 ..., User "Bob" 25 ...]
let older = users & each . age %~ (+1)

-- 30세 이상 사용자의 이름만 대문자로 (filtered + Traversal)
let seniors = users & each . filtered (\u -> u ^. age >= 30) . name %~ map toUpper
```

## 참고 자료

- [lens — Haskell Optics 라이브러리 (Hackage)](https://hackage.haskell.org/package/lens)
- [profunctor-optics — Profunctor 기반 경량 Optics (Hackage)](https://hackage.haskell.org/package/profunctor-optics)
- [profunctors — Edward Kmett의 Haskell Profunctor 라이브러리 (GitHub)](https://github.com/ekmett/profunctors)
- [Z3 Theorem Prover — Optics 법칙 SMT 검증에 활용 가능 (GitHub)](https://github.com/Z3Prover/z3)
