---
layout: post
title: "Kotlin Coroutines 심화: CoroutineContext 내부 구조와 커스텀 Dispatcher·Element 구현 완전 정복"
date: 2026-11-08
categories: [android, kotlin]
tags: [kotlin, coroutines, coroutinecontext, dispatcher, android, concurrency]
---

## 개요

Kotlin Coroutines를 사용하다 보면 `Dispatchers.IO`, `Dispatchers.Main`, `launch(Dispatchers.Default)` 같은 코드를 자연스럽게 쓰게 된다. 하지만 이 `Dispatchers`가 실제로 어떤 타입인지, 왜 `+` 연산으로 컨텍스트를 조합할 수 있는지, 그리고 직접 커스텀 Dispatcher나 CoroutineContext.Element를 만들 수 있는지를 깊이 아는 개발자는 많지 않다.

이 글에서는 `CoroutineContext`의 내부 자료구조와 타입 계층을 분석하고, 실용적인 커스텀 `CoroutineContext.Element`와 `CoroutineDispatcher`를 직접 구현해본다. 이 지식은 코루틴 기반의 라이브러리 설계, 테스트 더블 작성, 추적(tracing) 시스템 연동 등 고급 시나리오에서 핵심적으로 쓰인다.

---

## CoroutineContext란 무엇인가

`CoroutineContext`는 코루틴의 실행 환경을 정의하는 불변(immutable) 집합이다. `kotlin.coroutines` 패키지의 인터페이스로, 각 코루틴은 자신만의 `CoroutineContext`를 가진다.

```kotlin
public interface CoroutineContext {
    public operator fun <E : Element> get(key: Key<E>): E?
    public fun <R> fold(initial: R, operation: (R, Element) -> R): R
    public operator fun plus(context: CoroutineContext): CoroutineContext
    public fun minusKey(key: Key<*>): CoroutineContext

    public interface Key<E : Element>
    public interface Element : CoroutineContext {
        public val key: Key<*>
    }
}
```

`CoroutineContext`는 사실 **연결 리스트(linked list)** 구조로 구현된다. 내부 구현 클래스 `CombinedContext`가 `Element`들을 체인 형태로 연결하고, `get(key)` 호출 시 체인을 선형 탐색한다.

```
CoroutineContext = [CoroutineName] + [Job] + [CoroutineDispatcher] + ...
                        ↓
CombinedContext(CombinedContext(CombinedContext(EmptyCoroutineContext, CoroutineName), Job), CoroutineDispatcher)
```

`+` 연산자는 왼쪽 컨텍스트에서 오른쪽 키와 동일한 Element를 제거한 뒤 오른쪽을 추가한다. 따라서 같은 Key를 가진 Element는 항상 하나만 존재한다.

---

## 왜 커스텀 Element와 Dispatcher가 필요한가

### 1. 요청 ID 전파 (Request Tracing)

분산 시스템에서 요청 추적(request tracing)은 필수다. HTTP 요청이 들어올 때 발급된 `requestId`를 코루틴이 중단·재개를 반복하더라도 잃어버리지 않아야 한다. `ThreadLocal`은 코루틴이 다른 스레드에서 재개될 때 값을 잃는다. 이를 해결하기 위해 `CoroutineContext.Element`를 활용한다.

### 2. 커스텀 스레드 풀 Dispatcher

기본 `Dispatchers.IO`는 공유 풀이므로, 특정 작업(예: 외부 API 호출)에 전용 스레드 풀을 제한하고 싶을 때 커스텀 `CoroutineDispatcher`를 만들어 풀 크기와 큐 용량을 제어한다.

### 3. 테스트 Dispatcher

`TestCoroutineDispatcher` / `UnconfinedTestDispatcher` 같은 테스트 전용 Dispatcher가 `CoroutineDispatcher`의 서브클래스임을 이해하면 직접 테스트 더블을 작성할 수 있다.

---

## 실제 구현 예제 1: 커스텀 CoroutineContext.Element

**목표**: HTTP 요청 ID를 코루틴 컨텍스트에 태그하여, 중단·재개가 반복되어도 항상 접근 가능하게 한다.

```kotlin
import kotlin.coroutines.AbstractCoroutineContextElement
import kotlin.coroutines.CoroutineContext
import kotlinx.coroutines.*

// 1. CoroutineContext.Element 구현
class RequestIdContext(val requestId: String) : AbstractCoroutineContextElement(RequestIdContext) {
    companion object Key : CoroutineContext.Key<RequestIdContext>

    override fun toString(): String = "RequestId($requestId)"
}

// 2. 편의 함수: 현재 컨텍스트에서 requestId를 꺼내는 확장
val CoroutineContext.requestId: String?
    get() = this[RequestIdContext]?.requestId

// 3. 사용 예시
suspend fun fetchUserData(userId: String): String {
    // 현재 코루틴의 컨텍스트에서 requestId를 꺼낸다
    val reqId = currentCoroutineContext().requestId ?: "unknown"
    delay(100) // 네트워크 호출 시뮬레이션 — 이 중단 이후에도 requestId는 유지됨
    return "User[$userId] for request=$reqId"
}

suspend fun processOrder(orderId: String): String {
    val reqId = currentCoroutineContext().requestId ?: "unknown"
    delay(50)
    return "Order[$orderId] for request=$reqId"
}

fun main() = runBlocking {
    val requestId = "req-20261108-abc"

    // RequestIdContext를 컨텍스트에 주입
    withContext(RequestIdContext(requestId)) {
        // 여러 코루틴을 동시에 실행하더라도 각자의 컨텍스트에 requestId가 존재
        val userDeferred = async { fetchUserData("42") }
        val orderDeferred = async { processOrder("99") }

        println(userDeferred.await())   // User[42] for request=req-20261108-abc
        println(orderDeferred.await())  // Order[99] for request=req-20261108-abc
    }

    // 다른 requestId로 별도 처리
    withContext(RequestIdContext("req-20261108-xyz")) {
        println(fetchUserData("7"))    // User[7] for request=req-20261108-xyz
    }
}
```

**핵심 포인트**:
- `AbstractCoroutineContextElement`를 상속하고 `companion object Key`를 선언하면 타입 안전한 Key 조회가 가능하다.
- `+` 연산으로 기존 컨텍스트에 추가하거나 교체할 수 있다.
- `withContext`로 자식 코루틴에 자동 전파된다.

---

## 실제 구현 예제 2: 커스텀 CoroutineDispatcher

**목표**: 외부 API 호출 전용으로 최대 4개의 스레드만 사용하는 `Dispatcher`를 만들고, 작업 완료 후 명시적으로 종료한다.

```kotlin
import kotlinx.coroutines.*
import java.util.concurrent.Executors
import java.util.concurrent.atomic.AtomicInteger
import kotlin.coroutines.CoroutineContext

// 1. 커스텀 CoroutineDispatcher 구현
class BoundedDispatcher(
    private val name: String,
    private val maxThreads: Int
) : CoroutineDispatcher(), AutoCloseable {

    private val threadCount = AtomicInteger(0)
    private val executor = Executors.newFixedThreadPool(maxThreads) { runnable ->
        Thread(runnable, "$name-thread-${threadCount.incrementAndGet()}").also {
            it.isDaemon = true
        }
    }

    // isDispatchNeeded: false를 반환하면 dispatch를 건너뛰고 현재 스레드에서 직접 실행
    // 여기서는 항상 디스패치하여 전용 풀에서 실행한다
    override fun isDispatchNeeded(context: CoroutineContext): Boolean = true

    override fun dispatch(context: CoroutineContext, block: Runnable) {
        executor.execute(block)
    }

    override fun close() {
        executor.shutdown()
    }

    override fun toString(): String = "BoundedDispatcher($name, maxThreads=$maxThreads)"
}

// 2. 사용 예시: 외부 API 전용 Dispatcher
suspend fun callExternalApi(endpoint: String): String {
    delay(50) // 실제 HTTP 호출 대신 시뮬레이션
    return "Response from $endpoint on ${Thread.currentThread().name}"
}

fun main() = runBlocking {
    // 최대 2개 스레드로 제한된 외부 API Dispatcher 생성
    val apiDispatcher = BoundedDispatcher("ExternalApi", maxThreads = 2)

    try {
        val jobs = (1..6).map { id ->
            async(apiDispatcher) {
                callExternalApi("/api/resource/$id")
            }
        }
        jobs.forEach { println(it.await()) }
    } finally {
        // 반드시 close()로 스레드 풀을 해제한다
        apiDispatcher.close()
    }
}

// 출력 예시 (최대 2개 스레드 이름만 나타남):
// Response from /api/resource/1 on ExternalApi-thread-1
// Response from /api/resource/2 on ExternalApi-thread-2
// Response from /api/resource/3 on ExternalApi-thread-1
// ...
```

**핵심 포인트**:
- `CoroutineDispatcher`는 `ContinuationInterceptor`의 서브클래스로, `dispatch()` 메서드만 구현하면 된다.
- `isDispatchNeeded()`를 `false`로 반환하면 불필요한 스레드 전환을 피할 수 있다(예: 이미 올바른 스레드에 있을 때).
- `AutoCloseable`을 구현하여 `use {}` 블록이나 `try/finally`로 스레드 풀 누수를 방지한다.

---

## 커스텀 Element + Dispatcher 조합

실전에서는 Element와 Dispatcher를 함께 조합하여 사용한다.

```kotlin
// ThreadContextElement: ThreadLocal 값을 코루틴 전환 시 함께 복원
class MdcContext(private val map: Map<String, String>) :
    ThreadContextElement<Map<String, String>>,
    AbstractCoroutineContextElement(MdcContext) {

    companion object Key : CoroutineContext.Key<MdcContext>

    // 코루틴이 스레드에 올라탈 때 호출 — ThreadLocal에 값 설정
    override fun updateThreadContext(context: CoroutineContext): Map<String, String> {
        val oldState = MDC.getCopyOfContextMap() ?: emptyMap()
        MDC.setContextMap(map)
        return oldState
    }

    // 코루틴이 스레드에서 내려올 때 호출 — ThreadLocal 복원
    override fun restoreThreadContext(context: CoroutineContext, oldState: Map<String, String>) {
        MDC.setContextMap(oldState)
    }
}
// (실제 사용 시 SLF4J MDC import 필요)
```

`ThreadContextElement`는 `CoroutineContext.Element`의 서브타입으로, 코루틴이 스레드에 올라탈 때(resume) `updateThreadContext`를 호출하고 내려올 때(suspend) `restoreThreadContext`를 호출한다. 이를 통해 `ThreadLocal` 기반 로깅 컨텍스트(MDC)를 코루틴에서도 완벽히 유지할 수 있다.

---

## CoroutineContext 조합 연산 내부 동작

```kotlin
val ctx = Dispatchers.Default + CoroutineName("worker") + RequestIdContext("abc")
```

내부적으로 `+` 연산은 다음과 같이 처리된다:

1. `Dispatchers.Default + CoroutineName("worker")` → `CombinedContext(Dispatchers.Default, CoroutineName("worker"))`
2. `CombinedContext(...) + RequestIdContext("abc")` → `CombinedContext(CombinedContext(Dispatchers.Default, CoroutineName("worker")), RequestIdContext("abc"))`

`ctx[CoroutineName]`을 호출하면 체인을 순회하며 `Key`가 일치하는 `Element`를 반환한다. 최악의 경우 O(n) 탐색이지만, 실제 컨텍스트의 Element 수는 보통 5개 이하이므로 성능 문제는 없다.

---

## 주의사항과 팁

### 1. 커스텀 Dispatcher는 반드시 종료하라
`newSingleThreadContext()`나 직접 구현한 `Executor` 기반 Dispatcher는 사용 후 반드시 `close()`를 호출해야 한다. 그렇지 않으면 데몬 스레드가 아닌 경우 JVM이 종료되지 않거나 스레드 풀이 누수된다.

```kotlin
val dispatcher = newSingleThreadContext("MyThread")
dispatcher.use {
    withContext(it) { /* 작업 */ }
} // use 블록을 벗어나면 자동으로 close() 호출
```

### 2. Element Key는 companion object로만 선언하라
`CoroutineContext.Key<E>`의 인스턴스가 여럿이면 같은 클래스의 Element를 꺼낼 수 없다. `companion object Key`를 싱글턴으로 선언하는 것이 표준 패턴이다.

### 3. Dispatchers.Unconfined 는 운영 코드에 쓰지 마라
`Dispatchers.Unconfined`는 첫 번째 중단 지점까지는 현재 스레드에서 실행되고, 재개 시에는 중단된 함수가 선택한 스레드에서 실행된다. 테스트나 특수한 경우를 제외하면 예측 불가한 스레드 전환을 일으킨다.

### 4. `coroutineContext`와 `currentCoroutineContext()`의 차이
- `coroutineContext`: `CoroutineScope` 블록 내에서 직접 접근하는 프로퍼티 (컴파일러 인트린직)
- `currentCoroutineContext()`: `suspend` 함수 내 어디서든 현재 컨텍스트를 꺼내는 최상위 suspend 함수

```kotlin
// CoroutineScope 블록 안에서
launch {
    val ctx = coroutineContext  // OK
}

// suspend 함수 안에서
suspend fun foo() {
    val ctx = currentCoroutineContext()  // OK
}
```

### 5. CoroutineContext는 불변이다
`+` 연산은 항상 새 컨텍스트 인스턴스를 반환한다. 기존 컨텍스트를 직접 변경하는 방법은 없다. 상태를 변경하려면 `withContext(newContext)`를 사용하거나 새 컨텍스트를 만들어 `launch` / `async`에 전달해야 한다.

---

## 마무리

`CoroutineContext`는 단순한 설정 값의 묶음이 아니라, Kotlin 코루틴 전체 메커니즘의 핵심 뼈대다. `Job`, `CoroutineDispatcher`, `CoroutineExceptionHandler`, `CoroutineName`이 모두 `CoroutineContext.Element`이고, 이들을 `+` 연산으로 조합하는 것이 코루틴 실행 환경의 전부다.

직접 `AbstractCoroutineContextElement`를 상속한 커스텀 Element를 만들면 요청 ID, 인증 토큰, MDC 로깅 컨텍스트 등을 코루틴 전체 생명주기에 걸쳐 안전하게 전파할 수 있다. 그리고 `CoroutineDispatcher`를 직접 구현하면 전용 스레드 풀, 모니터링 계층 추가, 테스트 제어 등 다양한 고급 시나리오를 지원할 수 있다.

## 참고 자료
- [Kotlin 공식 문서 — Coroutine context and dispatchers](https://kotlinlang.org/docs/coroutine-context-and-dispatchers.html)
- [Kotlin stdlib — AbstractCoroutineContextElement](https://kotlinlang.org/api/core/kotlin-stdlib/kotlin.coroutines/-abstract-coroutine-context-element/)
- [ProAndroidDev — Everything about Customizing coroutine contexts in kotlin](https://proandroiddev.com/everything-about-customizing-coroutine-contexts-in-kotlin-0a8a129bbc41)
