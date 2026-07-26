---
layout: post
title: "Kotlin Contracts 심화: callsInPlace·implies로 컴파일러 스마트 캐스트와 초기화 검사를 직접 제어하는 법"
date: 2026-07-26
categories: [android, kotlin]
tags: [kotlin, contracts, smartcast, callsInPlace, implies, compiler, android]
---

Kotlin의 타입 시스템은 스마트 캐스트(Smart Cast) 덕분에 명시적 캐스팅 없이도 안전하게 타입을 좁힐 수 있습니다. 그런데 컴파일러가 스마트 캐스트를 거부하거나, `val`로 선언한 변수가 초기화됐다고 인식하지 못해서 당황한 경험이 있으신가요? 이 한계를 정면으로 돌파하는 도구가 바로 **Kotlin Contracts**입니다. 이 글에서는 Kotlin 1.3에서 도입된 Contracts API의 내부 동작 원리부터, `callsInPlace`와 `implies`를 사용한 실전 활용까지 깊이 있게 다룹니다.

---

## 1. 왜 컴파일러는 모르는 게 있는가?

Kotlin 컴파일러는 흐름 분석(Control Flow Analysis)을 통해 스마트 캐스트를 수행합니다. 아래 코드는 정상적으로 동작합니다.

```kotlin
fun process(value: Any?) {
    if (value is String) {
        println(value.length) // String으로 스마트 캐스트 성공
    }
}
```

그런데 동일한 조건 검사를 함수로 추출하면 어떨까요?

```kotlin
fun isString(value: Any?): Boolean = value is String

fun process(value: Any?) {
    if (isString(value)) {
        println(value.length) // 컴파일 에러! Smart cast to 'String' is impossible
    }
}
```

컴파일러는 `isString()` 함수가 내부적으로 무엇을 검사하는지 알 방법이 없습니다. 그 함수가 외부 라이브러리에 있을 수도 있고, 복잡한 로직을 가질 수도 있습니다. 컴파일러의 관점에서 `isString(value)`가 `true`를 반환해도 `value`가 `String`이라는 보증이 없습니다.

람다 초기화 문제도 마찬가지입니다.

```kotlin
fun runOnce(block: () -> Unit) {
    block()
}

fun main() {
    val name: String
    runOnce {
        name = "Kotlin" // 초기화
    }
    println(name) // 컴파일 에러! Variable 'name' must be initialized
}
```

`runOnce` 내부에서 `block()`이 반드시 호출된다는 사실을 컴파일러는 모릅니다. 나중에 호출될 수도 있고, 조건에 따라 아예 호출되지 않을 수도 있습니다. 따라서 `name`이 초기화됐는지 알 수 없습니다.

**Kotlin Contracts**는 이런 상황에서 개발자가 컴파일러에게 "이 함수는 이렇게 동작한다"고 **보증**을 제공하는 메커니즘입니다.

---

## 2. Contracts의 구조와 두 가지 효과

Contracts API는 `kotlin.contracts` 패키지에 있으며, 현재 실험적(Experimental) 상태입니다. 함수 본문의 **가장 첫 번째 문장**으로 `contract { }` 블록을 배치해야 합니다.

```kotlin
@OptIn(ExperimentalContracts::class)
fun myFunction() {
    contract {
        // Effect 선언
    }
    // 함수 본문
}
```

### 2.1 Effect 종류

현재 Kotlin Contracts는 두 가지 주요 효과(Effect)를 제공합니다.

| Effect | 설명 |
|--------|------|
| `callsInPlace(block, kind)` | 람다가 얼마나 자주 호출되는지 명시 |
| `returns(value) implies condition` | 반환 값에 따른 타입/조건 추론 |

`InvocationKind`에는 네 가지 옵션이 있습니다.

- `EXACTLY_ONCE`: 정확히 한 번 호출됨
- `AT_LEAST_ONCE`: 최소 한 번 이상 호출됨
- `AT_MOST_ONCE`: 최대 한 번 호출되거나 아예 호출되지 않음
- `UNKNOWN`: 횟수 불명확 (기본값)

---

## 3. 실전 예제 1: callsInPlace로 val 초기화 보증

가장 일반적인 사용 사례입니다. `val`로 선언된 변수가 람다 내부에서 단 한 번 초기화됨을 보증합니다.

```kotlin
import kotlin.contracts.ExperimentalContracts
import kotlin.contracts.InvocationKind
import kotlin.contracts.contract

@OptIn(ExperimentalContracts::class)
inline fun <T> runSafely(block: () -> T): T {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return block()
}

@OptIn(ExperimentalContracts::class)
inline fun measureBlock(label: String, block: () -> Unit) {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    val startTime = System.nanoTime()
    block()
    val elapsed = System.nanoTime() - startTime
    println("[$label] elapsed: ${elapsed / 1_000_000}ms")
}

fun main() {
    // val 초기화 보증 예제
    val userId: Long
    val userName: String
    
    runSafely {
        userId = 42L       // 컴파일러가 초기화를 인식
        userName = "Alice"
    }
    
    println("User: $userName (id=$userId)") // 컴파일 성공!
    
    // 성능 측정 예제
    val result: List<Int>
    measureBlock("filter-map") {
        result = (1..1_000_000)
            .filter { it % 2 == 0 }
            .map { it * it }
            .take(100)
    }
    println("First result: ${result.first()}")
    
    // 중첩 사용 — 외부 val도 초기화됨
    val processed: String
    measureBlock("string-process") {
        runSafely {
            processed = "Hello, Contracts!"
                .uppercase()
                .reversed()
        }
    }
    println(processed) // 컴파일 성공!
}
```

> **핵심 포인트**: `callsInPlace(block, EXACTLY_ONCE)`가 있으면 컴파일러는 `block` 안에서 이루어진 모든 초기화를 함수 호출 지점에서 완료된 것으로 간주합니다. `EXACTLY_ONCE`를 사용하면 `val`에도 단 한 번만 대입할 수 있습니다.

---

## 4. 실전 예제 2: returns + implies로 스마트 캐스트 제어

두 번째 효과인 `returns(value) implies condition`은 함수가 특정 값을 반환할 때 조건이 참임을 컴파일러에게 알립니다. 이를 통해 별도 함수로 분리된 타입 검사에서도 스마트 캐스트가 동작하게 만들 수 있습니다.

```kotlin
import kotlin.contracts.ExperimentalContracts
import kotlin.contracts.contract

// 예제: 커스텀 타입 가드 함수들

@OptIn(ExperimentalContracts::class)
fun Any?.isString(): Boolean {
    contract {
        returns(true) implies (this@isString is String)
    }
    return this is String
}

@OptIn(ExperimentalContracts::class)
fun String?.isNotNullOrBlank(): Boolean {
    contract {
        returns(true) implies (this@isNotNullOrBlank != null)
    }
    return !isNullOrBlank()
}

// Sealed class 계층 예제
sealed class ApiResult<out T> {
    data class Success<T>(val data: T) : ApiResult<T>()
    data class Error(val code: Int, val message: String) : ApiResult<Nothing>()
    object Loading : ApiResult<Nothing>()
}

@OptIn(ExperimentalContracts::class)
fun <T> ApiResult<T>.isSuccess(): Boolean {
    contract {
        returns(true) implies (this@isSuccess is ApiResult.Success)
    }
    return this is ApiResult.Success
}

@OptIn(ExperimentalContracts::class)
fun <T> ApiResult<T>.isError(): Boolean {
    contract {
        returns(true) implies (this@isError is ApiResult.Error)
    }
    return this is ApiResult.Error
}

// 실제 사용 예시
fun handleResult(result: ApiResult<String>) {
    if (result.isSuccess()) {
        // Contract 덕분에 ApiResult.Success로 스마트 캐스트!
        println("데이터: ${result.data}")
    }
    if (result.isError()) {
        // ApiResult.Error로 스마트 캐스트!
        println("에러 ${result.code}: ${result.message}")
    }
}

fun processInput(value: Any?) {
    if (value.isString()) {
        // String으로 스마트 캐스트!
        println("문자열 길이: ${value.length}")
        println("대문자: ${value.uppercase()}")
    }
}

fun processUserName(name: String?) {
    if (name.isNotNullOrBlank()) {
        // name이 null이 아님을 보증 — String으로 스마트 캐스트!
        println("환영합니다, ${name.trim()}님!")
    }
}

fun main() {
    handleResult(ApiResult.Success("Hello, Contracts!"))
    handleResult(ApiResult.Error(404, "Not Found"))
    
    processInput("Kotlin")
    processInput(42)
    
    processUserName("   Alice   ")
    processUserName(null)
    processUserName("")
}
```

> **핵심 포인트**: `returns(true) implies (this@isSuccess is ApiResult.Success)`는 "이 함수가 `true`를 반환하면, `this`는 `ApiResult.Success`이다"라는 관계를 컴파일러에게 알립니다. 이 덕분에 `if (result.isSuccess())` 블록 안에서 `result.data`에 직접 접근할 수 있습니다.

### returns()와 returnsNotNull()

| 함수 | 의미 |
|------|------|
| `returns()` | 함수가 정상 반환(예외 없이) |
| `returns(true)` | 함수가 `true`를 반환 |
| `returns(false)` | 함수가 `false`를 반환 |
| `returns(null)` | 함수가 `null`을 반환 |
| `returnsNotNull()` | 함수가 null이 아닌 값을 반환 |

`returns()` (값 없음)는 함수가 예외를 던지지 않는다는 보증으로, `require`·`check`·`error` 표준 함수들이 이 방식을 사용합니다.

```kotlin
// 표준 라이브러리 require 함수가 내부적으로 사용하는 패턴
@OptIn(ExperimentalContracts::class)
fun require(value: Boolean): Unit {
    contract {
        returns() implies value  // 반환됐다면 value가 true임을 보증
    }
    if (!value) throw IllegalArgumentException()
}

fun parse(input: String?): Int {
    require(input != null)   // 이후 input은 String으로 스마트 캐스트
    return input.toInt()     // 컴파일 성공!
}
```

---

## 5. 실제 Android 프로젝트 적용 패턴

### 5.1 ViewModel 상태 업데이트 헬퍼

```kotlin
import kotlin.contracts.ExperimentalContracts
import kotlin.contracts.InvocationKind
import kotlin.contracts.contract
import androidx.lifecycle.MutableStateFlow

@OptIn(ExperimentalContracts::class)
inline fun <T> MutableStateFlow<T>.update(transform: (T) -> T) {
    contract {
        callsInPlace(transform, InvocationKind.EXACTLY_ONCE)
    }
    value = transform(value)
}

// 사용 예
data class UiState(
    val isLoading: Boolean = false,
    val userName: String = "",
    val error: String? = null
)

class UserViewModel : androidx.lifecycle.ViewModel() {
    private val _uiState = MutableStateFlow(UiState())
    
    fun loadUser(userId: Long) {
        val previousName: String
        _uiState.update { state ->
            previousName = state.userName  // val 초기화 가능!
            state.copy(isLoading = true)
        }
        // previousName 사용 가능
        println("Loading user, previous: $previousName")
    }
}
```

### 5.2 타입 안전 sealed class 처리 DSL

```kotlin
import kotlin.contracts.ExperimentalContracts
import kotlin.contracts.contract

sealed class ViewEvent {
    data class Navigate(val route: String) : ViewEvent()
    data class ShowSnackbar(val message: String) : ViewEvent()
    data class ShowDialog(val title: String, val body: String) : ViewEvent()
}

@OptIn(ExperimentalContracts::class)
fun ViewEvent.isNavigate(): Boolean {
    contract { returns(true) implies (this@isNavigate is ViewEvent.Navigate) }
    return this is ViewEvent.Navigate
}

@OptIn(ExperimentalContracts::class)
fun ViewEvent.isSnackbar(): Boolean {
    contract { returns(true) implies (this@isSnackbar is ViewEvent.ShowSnackbar) }
    return this is ViewEvent.ShowSnackbar
}

// Fragment에서 사용 예시
fun handleEvent(event: ViewEvent) {
    when {
        event.isNavigate() -> {
            // event.route 직접 접근 가능 (스마트 캐스트)
            navigateTo(event.route)
        }
        event.isSnackbar() -> {
            // event.message 직접 접근 가능
            showSnackbar(event.message)
        }
        else -> { /* ... */ }
    }
}

fun navigateTo(route: String) { println("Navigating to: $route") }
fun showSnackbar(message: String) { println("Snackbar: $message") }
```

---

## 6. inline 함수와의 관계

`callsInPlace` contract는 `inline` 함수와 특히 잘 어울립니다. 표준 라이브러리의 `run`, `let`, `apply`, `also`, `with` 함수들은 이미 `callsInPlace` contract를 내장하고 있어 스마트 캐스트와 `val` 초기화가 정상 동작합니다.

```kotlin
fun main() {
    val result: String
    
    // run은 이미 callsInPlace EXACTLY_ONCE contract 보유
    run {
        result = "Hello!"
    }
    println(result) // 컴파일 성공 — 표준 라이브러리가 contract 제공
    
    var counter = 0
    // repeat도 callsInPlace AT_LEAST_ONCE contract 보유
    repeat(5) {
        counter++
    }
    println(counter) // 5
}
```

단, `callsInPlace`는 `inline` 함수에만 유효하게 동작합니다. 인라이닝이 없는 일반 함수에 선언해도 컴파일러가 경고를 발생시키며, 런타임 검증이 이뤄지지 않습니다.

---

## 7. 주의사항과 실전 팁

### 7.1 Contracts는 컴파일러 힌트 — 런타임 보증이 아님

**가장 중요한 주의사항**입니다. Contracts는 컴파일러에게 정보를 제공할 뿐, 런타임에 어떤 검증도 수행하지 않습니다. 잘못된 Contract를 선언해도 컴파일 에러가 발생하지 않고, 잘못된 동작이 조용히 숨겨집니다.

```kotlin
// 위험한 예: 실제로는 여러 번 호출되지만 EXACTLY_ONCE 선언
@OptIn(ExperimentalContracts::class)
fun dangerousFunction(block: () -> Unit) {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE) // 거짓말!
    }
    block()
    block() // 실제로는 두 번 호출 — 런타임 에러 아니지만 val 재대입 시 논리 오류
}
```

### 7.2 @ExperimentalContracts 어노테이션 전파

사용자 정의 Contract 함수를 사용하는 모든 위치에서 `@OptIn(ExperimentalContracts::class)`가 필요합니다. 모듈 전역으로 활성화하려면 `build.gradle.kts`에 추가할 수 있습니다.

```kotlin
// build.gradle.kts
kotlin {
    compilerOptions {
        freeCompilerArgs.addAll(
            "-opt-in=kotlin.contracts.ExperimentalContracts"
        )
    }
}
```

### 7.3 contract 블록은 반드시 첫 번째 문장

`contract { }` 블록은 함수 본문의 **첫 번째 표현식**이어야 합니다. 다른 코드가 먼저 오면 컴파일 에러가 발생합니다.

```kotlin
// 컴파일 에러!
@OptIn(ExperimentalContracts::class)
fun wrong(block: () -> Unit) {
    println("before") // contract 이전에 다른 코드가 있으면 안 됨
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    block()
}
```

### 7.4 확장 함수에서 this 참조

확장 함수에서 Contract의 `this`는 함수 수신자(receiver)를 가리킵니다. `implies` 조건에서 수신자를 참조하려면 **레이블이 붙은 this**를 사용해야 합니다.

```kotlin
@OptIn(ExperimentalContracts::class)
fun Any?.isNotNull(): Boolean {
    contract {
        returns(true) implies (this@isNotNull != null)
        //                     ^^^^^^^^^^^^^^^^ 레이블 필수!
    }
    return this != null
}
```

### 7.5 현재 지원되지 않는 것들

- `implies` 조건에는 타입 체크(`is`)와 null 체크(`!= null`)만 사용 가능
- 복잡한 논리 조합은 지원하지 않음 (`&&`, `||` 제한적)
- `callsInPlace`에 `UNKNOWN`을 사용하면 컴파일러 최적화 없음

---

## 8. 표준 라이브러리의 Contracts 활용 사례

Kotlin 표준 라이브러리는 이미 많은 함수에 Contracts를 적용하고 있습니다.

| 함수 | Contract 효과 |
|------|--------------|
| `run { }` | `callsInPlace(block, EXACTLY_ONCE)` |
| `let { }` | `callsInPlace(block, EXACTLY_ONCE)` |
| `apply { }` | `callsInPlace(block, EXACTLY_ONCE)` |
| `also { }` | `callsInPlace(block, EXACTLY_ONCE)` |
| `with(obj) { }` | `callsInPlace(block, EXACTLY_ONCE)` |
| `repeat(n) { }` | `callsInPlace(action, AT_LEAST_ONCE)` (n > 0 시) |
| `require(condition)` | `returns() implies condition` |
| `check(condition)` | `returns() implies condition` |
| `requireNotNull(value)` | `returns() implies (value != null)` |

이 함수들이 contract 덕분에 아래와 같이 동작합니다.

```kotlin
fun main() {
    val x: Int
    val y: Int
    
    x = run { 10 }         // val 초기화 가능
    let { y = 20 }         // val 초기화 가능
    
    val name: String? = "Kotlin"
    requireNotNull(name)   // 이후 name은 String으로 스마트 캐스트
    println(name.length)   // 컴파일 성공!
}
```

---

## 9. 마무리

Kotlin Contracts는 강력하지만 아직 실험적인 API입니다. 무분별하게 사용하기보다는, 컴파일러가 추론하지 못해 `val` 초기화나 스마트 캐스트가 막히는 **특정 상황**에서 해결책으로 활용하는 것이 좋습니다. 다음 상황에서 Contracts 도입을 검토해 보세요.

1. 람다를 받는 인라인 유틸리티 함수에서 내부 `val` 초기화가 필요할 때
2. Sealed class 계층의 타입 검사를 별도 함수로 분리하고 스마트 캐스트도 유지하고 싶을 때
3. `require`, `check`처럼 전제조건 검사 후 null 체크나 타입 보증이 필요할 때

Contracts의 올바른 사용은 코드의 안전성과 가독성을 동시에 높여주며, Kotlin의 타입 시스템이 제공하는 최대한의 정적 분석 혜택을 누릴 수 있게 해줍니다.

## 참고 자료
- [Kotlin 1.3 What's New — Contracts](https://kotlinlang.org/docs/whatsnew13.html#contracts)
- [Kotlin 고차 함수와 람다](https://kotlinlang.org/docs/lambdas.html)
- [Kotlin 인라인 함수](https://kotlinlang.org/docs/inline-functions.html)
