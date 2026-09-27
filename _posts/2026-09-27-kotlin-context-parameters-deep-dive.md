---
layout: post
title: "Kotlin Context Parameters 심화: 암묵적 컨텍스트로 의존성 전달을 혁신하는 법"
date: 2026-09-27
categories: [android, kotlin]
tags: [kotlin, context-parameters, dependency-injection, dsl, coroutines, android]
---

Kotlin 2.4.0에서 Stable로 승격된 **Context Parameters**는 함수와 프로퍼티가 호출 스코프에서 암묵적으로 제공되는 의존성을 선언할 수 있게 해주는 언어 기능이다. 기존 Context Receivers 실험의 후계자로, 이름 있는 파라미터 기반의 더 명확하고 안전한 설계로 재탄생했다.

## 1. Context Parameters란 무엇인가?

안드로이드 앱을 개발하다 보면 로거(Logger), 데이터베이스 커넥션, 현재 사용자 세션 같은 **횡단 관심사(Cross-Cutting Concerns)**를 여러 함수 계층을 통해 전달해야 하는 상황이 자주 생긴다. 지금까지는 두 가지 방법이 있었다.

- **명시적 매개변수 전달**: 모든 함수 시그니처에 파라미터를 추가하는 보일러플레이트 지옥
- **전역 상태/싱글톤**: 테스트가 어렵고, 암묵적 의존성으로 코드 이해를 어렵게 만듦

Context Parameters는 **세 번째 길**을 제시한다. 함수 선언부에 `context(name: Type)` 형태로 컨텍스트 파라미터를 명시하면, 컴파일러가 현재 스코프에서 해당 타입의 값을 찾아 자동으로 전달해준다.

```kotlin
// 기본 문법: context 키워드로 컨텍스트 파라미터 선언
context(logger: Logger)
fun processOrder(orderId: String) {
    logger.info("Processing order: $orderId")
    val order = fetchOrder(orderId)   // fetchOrder도 같은 컨텍스트를 전파받을 수 있다
    logger.info("Order fetched: $order")
}

// 호출 시점: Logger 인스턴스를 스코프에 제공
fun main() {
    val logger = ConsoleLogger()
    with(logger) {
        processOrder("ORD-001")  // logger가 암묵적으로 전달됨
    }
}
```

**Context Receivers와의 핵심 차이**: Context Parameters는 암묵적 수신자(`this`)가 아니라 **이름 있는 파라미터**다. `logger.info(...)`처럼 선언된 이름으로 명시적으로 접근하며, `info(...)`만으로는 호출할 수 없다. 이 덕분에 여러 컨텍스트가 중첩될 때 어떤 컨텍스트의 함수를 호출하는지 코드에서 바로 파악할 수 있다.

---

## 2. 왜 Context Parameters가 필요한가?

### 2.1 매개변수 드릴링(Parameter Drilling) 문제

전형적인 안드로이드 Use Case 구현에서 `AnalyticsTracker`를 여러 계층에 전달하는 상황을 살펴보자.

```kotlin
// 기존 방식: 매개변수 드릴링 — tracker가 모든 시그니처를 오염시킨다
fun handleUserLogin(user: User, tracker: AnalyticsTracker) {
    tracker.log("Login started")
    validateUser(user, tracker)
}

fun validateUser(user: User, tracker: AnalyticsTracker) {
    tracker.log("Validating user: ${user.id}")
    fetchUserProfile(user.id, tracker)
}

fun fetchUserProfile(userId: String, tracker: AnalyticsTracker) {
    tracker.log("Fetching profile: $userId")
    // 실제 로직...
}
```

`AnalyticsTracker`는 비즈니스 로직의 핵심이 아닌데도 모든 함수 시그니처를 오염시킨다. 나중에 `RequestId`도 전달해야 한다면 모든 시그니처를 또 수정해야 한다.

### 2.2 Context Parameters로의 전환

```kotlin
// Context Parameters 방식: 시그니처는 비즈니스 의미만 담는다
context(tracker: AnalyticsTracker)
fun handleUserLogin(user: User) {
    tracker.log("Login started")
    validateUser(user)
}

context(tracker: AnalyticsTracker)
fun validateUser(user: User) {
    tracker.log("Validating user: ${user.id}")
    fetchUserProfile(user.id)
}

context(tracker: AnalyticsTracker)
fun fetchUserProfile(userId: String) {
    tracker.log("Fetching profile: $userId")
}

// 호출 지점: 딱 한 번만 컨텍스트를 제공
fun onLoginSuccess(user: User) {
    val tracker = AnalyticsTracker.getInstance()
    with(tracker) {
        handleUserLogin(user)  // tracker가 전체 호출 체인에 전파된다
    }
}
```

함수 시그니처는 `User`만 다루고, `AnalyticsTracker`는 컨텍스트로 깔끔하게 분리된다.

---

## 3. 실제 구현 예제

### 3.1 다중 컨텍스트 — 트랜잭션 + 로거 조합

실무에서 가장 강력한 패턴은 여러 컨텍스트를 조합하는 것이다. 데이터베이스 트랜잭션과 로거를 함께 사용하는 Repository를 구현해보자.

```kotlin
interface Logger {
    fun info(msg: String)
    fun error(msg: String, e: Throwable? = null)
}

interface Transaction {
    fun commit()
    fun rollback()
}

data class User(val id: String, val name: String, val email: String)

// 두 컨텍스트를 함께 선언: Transaction과 Logger 모두 필요
context(tx: Transaction, logger: Logger)
fun insertUser(user: User): Result<User> {
    return try {
        logger.info("Inserting user: ${user.id}")
        // 실제 DB 삽입 로직 (예시)
        DatabaseDriver.execute(
            "INSERT INTO users (id, name, email) VALUES (?, ?, ?)",
            user.id, user.name, user.email
        )
        logger.info("User inserted successfully: ${user.id}")
        Result.success(user)
    } catch (e: Exception) {
        logger.error("Failed to insert user: ${user.id}", e)
        tx.rollback()
        Result.failure(e)
    }
}

context(tx: Transaction, logger: Logger)
fun updateUserEmail(userId: String, newEmail: String): Result<Unit> {
    logger.info("Updating email for user: $userId → $newEmail")
    DatabaseDriver.execute(
        "UPDATE users SET email = ? WHERE id = ?",
        newEmail, userId
    )
    return Result.success(Unit)
}

// 사용 예시: with() 중첩으로 두 컨텍스트를 순차적으로 제공
fun migrateUserEmails(users: List<User>) {
    val logger = Slf4jLogger()
    val tx = DatabaseTransaction.begin()

    with(tx) {
        with(logger) {
            try {
                users.forEach { user ->
                    updateUserEmail(user.id, "${user.id}@newdomain.com")
                        .getOrThrow()
                }
                tx.commit()
                logger.info("Migration completed: ${users.size} users")
            } catch (e: Exception) {
                tx.rollback()
                logger.error("Migration failed", e)
            }
        }
    }
}
```

컴파일러는 `insertUser`와 `updateUserEmail` 호출 시 스코프 체인에서 `Transaction`과 `Logger` 인스턴스를 찾아 자동 전달한다. 두 의존성 중 하나라도 스코프에 없으면 컴파일 에러가 발생한다.

### 3.2 Coroutine과의 통합 — 스코프 기반 비동기 작업

코루틴의 `CoroutineScope`와 `ApiClient`를 Context Parameters로 전달하면, 비동기 의존성도 함수 시그니처를 오염시키지 않고 주입할 수 있다.

```kotlin
import kotlinx.coroutines.*

interface ApiClient {
    suspend fun fetchUser(id: String): User
    suspend fun fetchOrders(userId: String): List<Order>
}

data class Order(val id: String, val total: Double)
data class DashboardData(val user: User, val recentOrders: List<Order>)

// CoroutineScope와 ApiClient를 컨텍스트로 선언
context(scope: CoroutineScope, client: ApiClient)
suspend fun loadUserDashboard(userId: String): DashboardData {
    // 두 API를 병렬로 호출
    val userDeferred = scope.async { client.fetchUser(userId) }
    val ordersDeferred = scope.async { client.fetchOrders(userId) }

    return DashboardData(
        user = userDeferred.await(),
        recentOrders = ordersDeferred.await().take(5)
    )
}

context(scope: CoroutineScope, client: ApiClient)
suspend fun refreshDashboard(userId: String, onComplete: (DashboardData) -> Unit) {
    val data = loadUserDashboard(userId)  // 컨텍스트가 자동 전파된다
    onComplete(data)
}

// ViewModel에서의 사용
class DashboardViewModel(private val apiClient: ApiClient) : ViewModel() {

    val dashboardData = MutableStateFlow<DashboardData?>(null)

    fun load(userId: String) {
        viewModelScope.launch {
            // viewModelScope를 CoroutineScope로, apiClient를 ApiClient로 제공
            with(viewModelScope) {
                with(apiClient) {
                    refreshDashboard(userId) { data ->
                        dashboardData.value = data
                    }
                }
            }
        }
    }
}
```

`loadUserDashboard`와 `refreshDashboard` 모두 `CoroutineScope`와 `ApiClient`를 파라미터로 받지 않지만, 컴파일러가 컨텍스트에서 찾아 전달해준다.

### 3.3 타입 안전 DSL 설계

DSL 설계에서도 Context Parameters는 강력하다. 특정 스코프 내에서만 유효한 함수를 컨텍스트로 표현할 수 있다.

```kotlin
@DslMarker
annotation class FormDsl

@FormDsl
class FormBuilder {
    private val fields = mutableListOf<String>()

    fun field(name: String, value: String) {
        fields.add("""<input name="$name" value="$value">""")
    }

    fun build() = """<form>${fields.joinToString("\n")}</form>"""
}

@FormDsl
class ValidationBuilder {
    private val rules = mutableListOf<String>()

    fun required(fieldName: String) {
        rules.add("$fieldName is required")
    }

    fun minLength(fieldName: String, min: Int) {
        rules.add("$fieldName must be at least $min characters")
    }

    fun buildRules() = rules.toList()
}

// Context Parameters로 FormBuilder 스코프 내에서만 유효한 함수 정의
context(form: FormBuilder)
fun textField(name: String, placeholder: String = "") {
    form.field(name, placeholder)
}

context(form: FormBuilder, validation: ValidationBuilder)
fun requiredTextField(name: String, minLen: Int = 1) {
    form.field(name, "")
    validation.required(name)
    if (minLen > 1) validation.minLength(name, minLen)
}

// 사용
fun buildLoginForm(): Pair<String, List<String>> {
    val form = FormBuilder()
    val validation = ValidationBuilder()

    with(form) {
        with(validation) {
            requiredTextField("username", minLen = 3)
            requiredTextField("password", minLen = 8)
            textField("rememberMe")
        }
    }

    return form.build() to validation.buildRules()
}
```

---

## 4. Context Receivers에서의 마이그레이션

Kotlin 2.2.0부터 기존 Context Receivers(`@ExperimentalContextReceivers`)는 Context Parameters로 대체되었다. 주요 차이점은 다음과 같다.

| 구분 | Context Receivers (구) | Context Parameters (신) |
|------|----------------------|------------------------|
| 접근 방식 | `this@Type` 또는 확장 함수처럼 암묵적 | 이름 지정 필수: `logger.info(...)` |
| 암묵적 `this` | ✅ (여러 수신자 충돌 가능) | ❌ (명시적 접근 강제) |
| 익명 선언 | 불가 | `context(_: Logger)` 후 `contextOf<Logger>()` |
| 생성자 지원 | 실험적 | ❌ 불가 (팩토리 함수로 대체) |
| 안정성 | 제거됨 | Kotlin 2.4.0 Stable |

IntelliJ IDEA 2025.2+에서는 **Change Signature** 리팩터링을 통해 기존 Context Receivers 코드를 Context Parameters로 자동 마이그레이션할 수 있다.

---

## 5. 주의사항 및 팁

### 5.1 생성자에는 사용 불가 — 팩토리 함수로 대체

현재 버전에서 생성자는 Context Parameters를 선언할 수 없다. 대신 최상위 팩토리 함수를 활용하라.

```kotlin
// ❌ 컴파일 에러: 생성자는 context 선언 불가
class UserService context(db: Database)(val userId: String)

// ✅ 팩토리 함수 패턴으로 대체
context(db: Database)
fun createUserService(userId: String): UserService =
    UserServiceImpl(userId, db)
```

### 5.2 `contextOf<T>()` — 익명 컨텍스트 접근

이름 없이 `_`로 선언한 익명 컨텍스트는 `contextOf<T>()`로 접근한다.

```kotlin
context(_: Logger)
fun log(message: String) {
    contextOf<Logger>().info(message)
}
```

### 5.3 Explicit Context Arguments — 오버로딩 모호성 해결

Kotlin 2.4.0 실험적 기능으로, 동일 이름 함수가 다른 컨텍스트로 오버로딩될 때 명시적으로 어느 오버로드를 호출할지 지정할 수 있다. `-Xexplicit-context-arguments` 컴파일러 옵션이 필요하다.

```kotlin
// build.gradle.kts
kotlin {
    compilerOptions {
        freeCompilerArgs.add("-Xexplicit-context-arguments")
    }
}

context(emailSender: EmailSender)
fun send(message: String) { emailSender.sendEmail(message) }

context(smsSender: SmsSender)
fun send(message: String) { smsSender.sendSms(message) }

context(email: EmailSender, sms: SmsSender)
fun notifyAll(message: String) {
    send(emailSender = email, message)   // 명시적 컨텍스트 인수로 오버로드 선택
    send(smsSender = sms, message)
}
```

### 5.4 테스트에서의 활용

Context Parameters는 전역 상태나 DI 컨테이너 없이 테스트 더블을 주입할 수 있어 테스트 코드가 간결해진다.

```kotlin
class FakeLogger : Logger {
    val logs = mutableListOf<String>()
    override fun info(msg: String) { logs.add(msg) }
    override fun error(msg: String, e: Throwable?) { logs.add("ERROR: $msg") }
}

class FakeTransaction : Transaction {
    var committed = false
    var rolledBack = false
    override fun commit() { committed = true }
    override fun rollback() { rolledBack = true }
}

class InsertUserTest {

    @Test
    fun `insertUser logs correctly and returns success`() {
        val fakeLogger = FakeLogger()
        val fakeTx = FakeTransaction()
        val user = User("u1", "김철수", "kim@test.com")

        val result = with(fakeTx) {
            with(fakeLogger) {
                insertUser(user)
            }
        }

        assertTrue(result.isSuccess)
        assertTrue(fakeLogger.logs.any { it.contains("u1") })
        assertFalse(fakeTx.rolledBack)
    }

    @Test
    fun `insertUser rolls back on failure`() {
        val fakeLogger = FakeLogger()
        val fakeTx = FakeTransaction()
        val user = User("bad-id", "에러유저", "err@test.com")

        // DatabaseDriver.execute가 예외를 던지는 시나리오
        with(fakeTx) {
            with(fakeLogger) {
                insertUser(user)  // 내부에서 예외 발생 가정
            }
        }

        assertTrue(fakeTx.rolledBack)
        assertTrue(fakeLogger.logs.any { it.startsWith("ERROR") })
    }
}
```

### 5.5 스코프 과부하 주의

Context Parameters는 강력하지만, 너무 많은 컨텍스트를 하나의 함수에 선언하면 오히려 가독성이 떨어진다. 일반적으로 2~3개 이상의 컨텍스트가 필요하다면 별도의 집합 타입(`ApplicationContext`, `RequestContext` 등)으로 묶는 것을 고려하라.

```kotlin
// 너무 많은 컨텍스트 — 리팩터링 신호
context(db: Database, logger: Logger, cache: Cache, metrics: Metrics, tx: Transaction)
fun complexOperation() { ... }

// 집합 타입으로 정리
data class AppContext(
    val db: Database,
    val logger: Logger,
    val cache: Cache,
    val metrics: Metrics
)

context(app: AppContext, tx: Transaction)
fun complexOperation() {
    app.logger.info("Starting...")
    app.db.query(...)
}
```

---

## 결론

Kotlin Context Parameters는 매개변수 드릴링 문제를 우아하게 해결하면서도, 타입 안전성과 코드 명시성을 동시에 유지한다. Kotlin 2.4.0에서 Stable로 승격됨에 따라 Android 프로젝트와 KMP 프로젝트 모두에서 실무 적용이 가능한 단계에 도달했다.

Repository 패턴, 트랜잭션 스코핑, 코루틴 컨텍스트 전달, DSL 설계 등 다양한 시나리오에서 함수 시그니처를 깔끔하게 유지하면서 의존성을 명확히 표현할 수 있다. 기존 Context Receivers를 사용하고 있다면 IntelliJ의 자동 마이그레이션 기능을 활용해 점진적으로 전환하자.

## 참고 자료
- [Context parameters \| Kotlin Documentation](https://kotlinlang.org/docs/context-parameters.html)
- [What's new in Kotlin 2.4.0](https://kotlinlang.org/docs/whatsnew24.html)
- [KEEP-0367: Context Parameters Proposal](https://github.com/Kotlin/KEEP/blob/master/proposals/context-parameters.md)
- [What's new in Kotlin 2.2.0](https://kotlinlang.org/docs/whatsnew22.html)
