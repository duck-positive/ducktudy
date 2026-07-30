---
layout: post
title: "함수형 프로그래밍 핵심 패턴 완전 정복: Functor, Applicative, Monad"
date: 2026-07-30
categories: [cs, computer-science]
tags: [functional-programming, monad, functor, applicative, haskell, scala, kotlin, referential-transparency]
---

함수형 프로그래밍(Functional Programming, FP)은 수십 년간 학계에서만 사용되던 패러다임이었지만, 지금은 Kotlin, Scala, Rust, TypeScript 등 실무에서 많이 쓰이는 언어들에 깊숙이 스며들었다. 특히 **Functor**, **Applicative**, **Monad**라는 세 개의 타입 클래스는 FP의 핵심 추상화 도구로, 이를 이해하면 `Optional`, `Result`, `Future`, `Flow`, `Observable` 같은 타입들이 왜 그렇게 설계되었는지 명확하게 보인다.

## 왜 함수형 프로그래밍이 필요한가?

명령형(Imperative) 코드의 가장 큰 문제는 **부작용(Side Effect)**이다. 변수를 변경하고, 전역 상태를 공유하고, I/O를 아무 곳에서나 수행하면 코드가 거대해질수록 버그 추적이 어렵다.

```kotlin
// 명령형: 부작용이 곳곳에 숨어 있다
var count = 0
fun increment() {
    count++ // 전역 상태 변경
}
```

함수형 프로그래밍은 이 문제를 **순수 함수(Pure Function)**와 **불변 데이터(Immutable Data)**로 해결한다.

```kotlin
// 함수형: count 값을 바꾸지 않고, 새 값을 반환한다
fun increment(count: Int): Int = count + 1
```

순수 함수는 두 가지 조건을 만족한다.
1. 같은 입력에는 항상 같은 출력이 나온다 (**참조 투명성, Referential Transparency**)
2. 함수 실행이 외부 상태에 영향을 미치지 않는다 (**부작용 없음**)

참조 투명성이 보장되면, 표현식을 그 결과값으로 자유롭게 치환할 수 있어 코드 추론이 쉬워진다.

```
val x = increment(5)  // x = 6
// 이 표현식을 언제, 어디서 평가해도 항상 6이다.
```

---

## Functor: 컨텍스트 속 값을 변환하기

Functor는 가장 단순한 추상화다. **컨텍스트(박스) 안에 있는 값에 함수를 적용**할 수 있다면 Functor다.

Haskell에서의 정의:

```haskell
class Functor f where
    fmap :: (a -> b) -> f a -> f b
```

`fmap`은 `a -> b` 함수를 받아서, `f a` 라는 컨텍스트 안의 값에 적용하여 `f b`를 반환한다. 컨텍스트(`f`)는 건드리지 않고, 안에 있는 값만 변환한다.

### Functor 법칙

모든 Functor는 두 가지 법칙을 반드시 만족해야 한다.

1. **항등 법칙(Identity Law)**: `fmap id == id`
   - `id`(항등 함수)로 fmap해도 컨텍스트는 바뀌지 않는다.
2. **합성 법칙(Composition Law)**: `fmap (f . g) == fmap f . fmap g`
   - 두 함수를 합성 후 fmap하는 것과, 각각 fmap하는 것이 동일하다.

### Kotlin에서의 Functor: `Optional`

Kotlin의 nullable 타입(`T?`)이나 Arrow 라이브러리의 `Option`은 사실상 Functor다.

```kotlin
// Option<T>는 값이 있거나(Some) 없는(None) 컨텍스트
// map이 바로 fmap에 해당한다

val price: Int? = 100
val discounted: Int? = price?.let { it * 90 / 100 }  // Some(90)

val noPrice: Int? = null
val noDiscount: Int? = noPrice?.let { it * 90 / 100 } // None

// 리스트도 Functor다: 리스트의 모든 원소에 함수를 적용한다
val numbers = listOf(1, 2, 3, 4, 5)
val doubled = numbers.map { it * 2 }  // [2, 4, 6, 8, 10]
```

리스트의 `map`, Optional의 `map`, `Future`의 `map`이 모두 같은 원리(`fmap`)로 동작한다는 걸 알면 코드가 훨씬 일관되게 보인다.

---

## Applicative: 컨텍스트 속 함수 적용하기

Functor가 "컨텍스트 안의 **값**에 일반 함수를 적용"한다면, Applicative는 "컨텍스트 안의 **함수**를 컨텍스트 안의 **값**에 적용"한다.

Haskell 정의:

```haskell
class Functor f => Applicative f where
    pure  :: a -> f a
    (<*>) :: f (a -> b) -> f a -> f b
```

- `pure`는 일반 값을 최소한의 컨텍스트로 감싼다.
- `<*>` (apply)는 컨텍스트 안의 함수를 컨텍스트 안의 값에 적용한다.

### Applicative의 실용성: 여러 컨텍스트 조합

여러 Optional 값을 동시에 다룰 때 Applicative가 빛난다.

```kotlin
// Arrow 라이브러리 활용 예시
import arrow.core.*

data class User(val name: String, val age: Int, val email: String)

fun validateName(name: String): Option<String> =
    if (name.isNotBlank()) Some(name) else None

fun validateAge(age: Int): Option<Int> =
    if (age in 1..150) Some(age) else None

fun validateEmail(email: String): Option<String> =
    if (email.contains("@")) Some(email) else None

// Applicative 스타일로 여러 검증 결과를 조합
fun createUser(name: String, age: Int, email: String): Option<User> =
    Option.applicative().map(
        validateName(name),
        validateAge(age),
        validateEmail(email)
    ) { (n, a, e) -> User(n, a, e) }.fix()
```

이 패턴은 모든 검증이 독립적으로 실행된다는 점이 Monad와의 차이다. Applicative는 각 컨텍스트가 서로 의존하지 않을 때 적합하다.

---

## Monad: 컨텍스트를 연결하는 파이프라인

Monad는 Applicative를 확장한다. 핵심은 `bind`(`>>=`, Kotlin에서는 `flatMap`)다.

```haskell
class Applicative m => Monad m where
    (>>=) :: m a -> (a -> m b) -> m b
    return :: a -> m a
```

`>>=`(bind)는 `m a` 안의 값을 꺼내어, `a -> m b` 함수에 넘겨주고, 결과인 `m b`를 반환한다. 이때 컨텍스트가 "중첩"되지 않고 하나로 합쳐진다.

`fmap`이 `f a -> f b`를 반환한다면, bind는 컨텍스트 안에서 새 컨텍스트를 생성하는 함수를 처리할 수 있다.

### Monad 법칙

1. **왼쪽 항등 법칙**: `return a >>= f == f a`
2. **오른쪽 항등 법칙**: `m >>= return == m`
3. **결합 법칙**: `(m >>= f) >>= g == m >>= (\x -> f x >>= g)`

### Kotlin에서 Monad: `Result`와 `flatMap`

```kotlin
// Result<T>는 성공(Success) 또는 실패(Failure) 컨텍스트
// flatMap이 바로 bind(>>=)에 해당한다

fun findUser(id: Int): Result<String> =
    if (id > 0) Result.success("Alice") else Result.failure(Exception("Invalid ID"))

fun fetchProfile(name: String): Result<String> =
    if (name.isNotEmpty()) Result.success("Profile of $name") else Result.failure(Exception("Empty name"))

fun getOrder(profile: String): Result<String> =
    Result.success("Order for: $profile")

// Monad 체이닝: 앞 단계가 실패하면 이후 단계는 실행되지 않는다
val result: Result<String> = findUser(1)
    .flatMap { name -> fetchProfile(name) }
    .flatMap { profile -> getOrder(profile) }

println(result)  // Success(Order for: Profile of Alice)

val failed: Result<String> = findUser(-1)
    .flatMap { name -> fetchProfile(name) }
    .flatMap { profile -> getOrder(profile) }

println(failed)  // Failure(java.lang.Exception: Invalid ID)
```

`flatMap`이 없으면 중첩 if/when 지옥이 된다.

```kotlin
// flatMap 없이 같은 로직을 작성하면:
val r1 = findUser(1)
if (r1.isSuccess) {
    val r2 = fetchProfile(r1.getOrThrow())
    if (r2.isSuccess) {
        val r3 = getOrder(r2.getOrThrow())
        // ...
    }
}
```

Monad의 `flatMap`이 이 수직 들여쓰기를 수평 파이프라인으로 바꿔준다.

### Haskell의 do-notation: Monad를 명령형처럼 쓰기

Haskell은 Monad 체이닝을 `do` 블록으로 표현한다. 마치 명령형 코드처럼 보이지만, 내부는 순수 함수다.

```haskell
getOrder :: Int -> Maybe String
getOrder userId = do
    name    <- findUser userId      -- findUser가 Nothing이면 여기서 중단
    profile <- fetchProfile name    -- fetchProfile이 Nothing이면 여기서 중단
    order   <- getOrder profile
    return order
```

이것은 아래와 완전히 동일하다:

```haskell
getOrder userId =
    findUser userId >>= \name ->
    fetchProfile name >>= \profile ->
    getOrder profile >>= \order ->
    return order
```

---

## IO Monad: 부작용을 타입으로 격리하기

Haskell의 IO Monad는 "부작용을 타입 시스템으로 격리"하는 패턴이다. IO를 반환하는 함수는 "이 함수는 부작용이 있다"는 것을 타입으로 명시한다.

```haskell
-- IO String은 "String을 반환하는, 부작용이 있을 수 있는 계산"
readName :: IO String
readName = do
    putStrLn "이름을 입력하세요:"
    name <- getLine
    return name

main :: IO ()
main = do
    name <- readName
    putStrLn ("안녕하세요, " ++ name ++ "!")
```

Kotlin의 `suspend` 함수, Rust의 `Future`, Java의 `CompletableFuture`도 모두 이 아이디어에 영향을 받았다.

---

## 실전 적용: Kotlin Arrow로 함수형 에러 처리

```kotlin
import arrow.core.*

sealed class AppError {
    object UserNotFound : AppError()
    object ProfileInvalid : AppError()
    data class NetworkError(val message: String) : AppError()
}

fun findUser(id: Int): Either<AppError, String> =
    if (id > 0) "Alice".right() else AppError.UserNotFound.left()

fun fetchProfile(name: String): Either<AppError, String> =
    if (name.isNotEmpty()) "Profile($name)".right() else AppError.ProfileInvalid.left()

fun processOrder(id: Int): Either<AppError, String> =
    findUser(id)
        .flatMap { name -> fetchProfile(name) }
        .map { profile -> "Order[$profile]" }

fun main() {
    when (val result = processOrder(1)) {
        is Either.Right -> println("성공: ${result.value}")
        is Either.Left  -> println("실패: ${result.value}")
    }
    // 출력: 성공: Order[Profile(Alice)]
}
```

`Either<Error, Value>`는 `Option`보다 강력하다. 실패 이유를 왼쪽(Left)에 담을 수 있어, 에러 컨텍스트를 잃지 않고 체이닝이 가능하다.

---

## Functor → Applicative → Monad 관계 요약

```
Functor   : fmap   (a -> b)    -> f a -> f b      # 값 변환
Applicative: apply  f (a -> b)  -> f a -> f b      # 함수도 컨텍스트 안에 있음
Monad     : bind   (a -> f b)  -> f a -> f b      # 값으로 새 컨텍스트 생성
```

| 추상화 | 핵심 연산 | 사용 시점 |
|---|---|---|
| Functor | `map` | 컨텍스트 안 값을 단순 변환 |
| Applicative | `apply`, `pure` | 독립적인 여러 컨텍스트 조합 |
| Monad | `flatMap`, `bind` | 이전 결과에 따라 다음 컨텍스트가 달라질 때 |

---

## 주의사항과 팁

1. **Monad는 만능이 아니다**: 독립적인 연산에는 Applicative가 더 적합하다. Monad는 순차적 의존성이 있을 때만 쓰자.
2. **Stack Overflow 주의**: 깊이 중첩된 flatMap은 스택을 소비한다. Haskell에서는 Trampoline, Kotlin에서는 `tailrec`이나 Channel 등으로 대응한다.
3. **법칙을 지켜라**: 직접 Functor/Monad 인스턴스를 만들 때는 항등·합성·결합 법칙을 반드시 검증하자. 법칙을 어기면 라이브러리 내부에서 예측 불가능한 동작이 생긴다.
4. **Kotlin Flow는 Cold Observable Monad**: `Flow<T>`는 Monad로 볼 수 있다. `flatMapConcat`, `flatMapMerge`가 bind 역할을 한다.
5. **디버깅 어려움**: 파이프라인이 길면 어디서 실패했는지 추적하기 어렵다. Arrow의 `Either` 체이닝에서 `tapLeft`를 활용해 실패 로깅을 추가하자.

---

## 참고 자료
- [Monad - HaskellWiki](https://wiki.haskell.org/Monad)
- [A Gentle Introduction to Haskell: Monads](https://www.haskell.org/tutorial/monads.html)
- [Real World Haskell - Chapter 14: Monads](https://book.realworldhaskell.org/read/monads.html)
- [Monad (functional programming) - Wikipedia](https://en.wikipedia.org/wiki/Monad_(functional_programming))
