---
layout: post
title: "Kotlin Coroutines 심화: Exception Handling, SupervisorScope, CoroutineExceptionHandler 완전 정복"
date: 2026-06-11
categories: [android, flutter]
tags: [android, kotlin, coroutines, exception-handling, supervisorscope, viewmodel]
---

코루틴에서의 예외 처리는 일반 코드와 근본적으로 다르게 동작합니다. 구조화된 동시성(Structured Concurrency)의 특성 때문에 예외가 부모-자식 코루틴 트리를 타고 전파되는 방식이 독특하며, 이를 제대로 이해하지 못하면 앱이 예상치 못한 방식으로 크래시 나거나 예외가 조용히 삼켜져 디버깅이 어려워집니다. 오늘은 `CoroutineExceptionHandler`, `SupervisorJob`, `supervisorScope`를 실전 코드와 함께 완전히 정복합니다.

---

## 1. 구조화된 동시성과 예외 전파의 기본 원칙

코루틴은 부모-자식 계층 구조를 이룹니다. 부모 코루틴이 `launch`로 자식을 생성하면 자식의 `Job`은 부모 `Job`에 등록됩니다. 기본 `Job`에서 자식 코루틴이 예외로 실패하면 다음 순서로 동작합니다.

1. 예외가 발생한 자식 코루틴이 취소됨
2. 예외가 부모 코루틴에 전파됨
3. 부모가 나머지 형제(sibling) 코루틴을 모두 취소함
4. 부모 코루틴도 실패 (예외가 상위로 계속 전파)

이를 **양방향 취소(bidirectional cancellation)**라고 합니다. 하나가 무너지면 전체가 취소되는 구조입니다.

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val job = launch {
        launch {
            delay(100)
            throw RuntimeException("자식 코루틴 오류")
        }
        launch {
            delay(1000)
            println("이 코드는 실행되지 않습니다") // 자식1 실패로 취소됨
        }
    }
    job.join()
    println("부모 코루틴도 취소됨")
}
// 출력: Exception in thread "main" java.lang.RuntimeException: 자식 코루틴 오류
```

첫 번째 자식에서 예외가 발생하면 두 번째 자식도 즉시 취소되고 부모 `Job`도 실패합니다. 이것이 코루틴의 기본 동작입니다.

---

## 2. CoroutineExceptionHandler: 최후의 방어선

`CoroutineExceptionHandler`는 처리되지 않은 예외를 잡는 전역 핸들러입니다. 단, **루트 코루틴**에서만 동작한다는 점이 핵심입니다. 자식 코루틴에 설치해도 무시됩니다.

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val handler = CoroutineExceptionHandler { coroutineContext, throwable ->
        println("예외 처리됨: ${throwable.message}")
        println("컨텍스트: ${coroutineContext[CoroutineName]}")
    }

    val scope = CoroutineScope(Dispatchers.Default + handler)

    scope.launch(CoroutineName("RootCoroutine")) {
        throw IllegalStateException("루트 코루틴 예외")
    }.join()

    // 자식 코루틴에 설치 — 효과 없음
    scope.launch {
        launch(handler) { // 이 핸들러는 무시됨
            throw RuntimeException("자식 예외 — 핸들러 무시")
        }
    }.join()

    println("프로그램 계속 실행")
}
// 출력:
// 예외 처리됨: 루트 코루틴 예외
// 컨텍스트: CoroutineName(RootCoroutine)
// Exception in thread "..." RuntimeException: 자식 예외 — 핸들러 무시
// 프로그램 계속 실행
```

> **핵심**: `CoroutineExceptionHandler`가 예외를 처리해도 해당 스코프의 `Job`은 취소됩니다. 핸들러는 예외를 *로깅/처리*할 뿐, 스코프를 복구하지는 않습니다. 예외 발생 후 동일 스코프에서 새 코루틴을 시작하려면 새 스코프를 생성해야 합니다.

---

## 3. SupervisorJob과 supervisorScope: 독립적 실패 허용

일반 `Job`에서는 자식 하나가 실패하면 모든 형제가 취소됩니다. 각 자식이 독립적으로 실패하도록 하려면 `SupervisorJob`이나 `supervisorScope`를 사용합니다.

### 3.1 SupervisorJob

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val supervisor = SupervisorJob()
    val scope = CoroutineScope(Dispatchers.Default + supervisor)

    val handler = CoroutineExceptionHandler { _, throwable ->
        println("독립 처리: ${throwable.message}")
    }

    val child1 = scope.launch(handler) {
        delay(50)
        throw RuntimeException("자식1 실패")
    }

    val child2 = scope.launch {
        delay(200)
        println("자식2 성공 — 자식1 실패의 영향을 받지 않음")
    }

    joinAll(child1, child2)
    println("supervisor 스코프 계속 동작")
    supervisor.cancel()
}
// 출력:
// 독립 처리: 자식1 실패
// 자식2 성공 — 자식1 실패의 영향을 받지 않음
// supervisor 스코프 계속 동작
```

`SupervisorJob`에서는 자식의 실패가 부모나 다른 형제에게 전파되지 않습니다. 각 자식이 자체적으로 `CoroutineExceptionHandler`를 가지거나 `try-catch`로 예외를 처리해야 합니다.

> **참고**: Android의 `viewModelScope`는 내부적으로 `SupervisorJob() + Dispatchers.Main.immediate`로 구성됩니다. 덕분에 한 `launch` 블록의 실패가 다른 `launch` 블록에 영향을 주지 않습니다.

### 3.2 supervisorScope 빌더

`supervisorScope`는 현재 suspend 함수 안에서 일시적으로 SupervisorJob 스코프를 만듭니다. 블록이 완료되거나 실패하면 원래 스코프로 돌아옵니다.

```kotlin
suspend fun loadDashboardData(userId: String): DashboardData = supervisorScope {
    // 세 요청을 병렬로 실행. 일부가 실패해도 나머지는 계속 진행
    val profileDeferred = async { userRepository.getProfile(userId) }
    val notificationsDeferred = async { notificationRepository.getNotifications(userId) }
    val feedDeferred = async { feedRepository.getFeed(userId) }

    val profile = profileDeferred.await() // 필수 — 실패하면 전체 실패
    val notifications = runCatching { notificationsDeferred.await() }
        .getOrDefault(emptyList())           // 선택적 — 실패해도 빈 리스트 반환
    val feed = runCatching { feedDeferred.await() }
        .getOrDefault(emptyList())           // 선택적 — 실패해도 빈 리스트 반환

    DashboardData(profile, notifications, feed)
}
```

`supervisorScope` 안에서 `async`를 사용할 때는 반드시 `await()`를 `try-catch` 또는 `runCatching`으로 감싸야 합니다. 감싸지 않으면 `await()` 호출 시 예외가 부모로 전파됩니다.

---

## 4. launch vs async의 예외 처리 차이

`launch`와 `async`는 예외를 처리하는 방식이 다릅니다.

| 빌더 | 예외 발생 시점 | 처리 방법 |
|------|--------------|---------|
| `launch` | 예외 발생 즉시 전파 | `CoroutineExceptionHandler` |
| `async` | `await()` 호출 시점에 노출 | `try-catch` 또는 `runCatching` |

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val scope = CoroutineScope(SupervisorJob())

    // launch: 예외가 즉시 전파 (CoroutineExceptionHandler 필요)
    scope.launch {
        throw RuntimeException("launch 예외 — 즉시 전파")
    }

    // async: Deferred에 예외 저장. await() 호출 시 노출
    val deferred = scope.async {
        delay(100)
        throw RuntimeException("async 예외 — await()에서 노출")
    }

    delay(50) // launch 예외 먼저 발생

    try {
        deferred.await()
    } catch (e: RuntimeException) {
        println("async 예외 잡힘: ${e.message}")
    }

    scope.cancel()
}
```

---

## 5. Android ViewModel 실전 패턴

```kotlin
class ArticleViewModel @Inject constructor(
    private val articleRepository: ArticleRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow<ArticleUiState>(ArticleUiState.Loading)
    val uiState: StateFlow<ArticleUiState> = _uiState.asStateFlow()

    // 예외 핸들러: UI 상태를 Error로 업데이트
    private val exceptionHandler = CoroutineExceptionHandler { _, throwable ->
        val message = when (throwable) {
            is IOException -> "네트워크 오류가 발생했습니다"
            is HttpException -> "서버 오류: ${throwable.code()}"
            else -> throwable.message ?: "알 수 없는 오류"
        }
        _uiState.update { ArticleUiState.Error(message) }
    }

    // 단일 로드 작업
    fun loadArticles() {
        viewModelScope.launch(exceptionHandler) {
            _uiState.update { ArticleUiState.Loading }
            val articles = articleRepository.getArticles()
            _uiState.update { ArticleUiState.Success(articles) }
        }
    }

    // 병렬 로드 — 선택적 데이터는 실패 허용
    fun loadArticleDetail(id: String) {
        viewModelScope.launch(exceptionHandler) {
            _uiState.update { ArticleUiState.Loading }

            supervisorScope {
                val articleDeferred = async { articleRepository.getArticle(id) }
                val relatedDeferred = async { articleRepository.getRelated(id) }
                val commentsDeferred = async { articleRepository.getComments(id) }

                val article = articleDeferred.await() // 필수: 실패하면 exceptionHandler로 전파
                val related = runCatching { relatedDeferred.await() }.getOrDefault(emptyList())
                val comments = runCatching { commentsDeferred.await() }.getOrDefault(emptyList())

                _uiState.update {
                    ArticleUiState.Detail(article, related, comments)
                }
            }
        }
    }

    // Flow 기반 실시간 데이터 수집
    fun observeArticles() {
        viewModelScope.launch {
            articleRepository.observeArticles()
                .catch { e ->
                    // Flow 내부 예외는 catch 연산자로 처리
                    _uiState.update { ArticleUiState.Error(e.message ?: "오류") }
                }
                .collect { articles ->
                    _uiState.update { ArticleUiState.Success(articles) }
                }
        }
    }
}
```

---

## 6. CancellationException은 절대 삼키면 안 된다

`CancellationException`은 코루틴 취소 신호입니다. 이를 `catch`로 잡아서 삼키면 코루틴이 취소되어도 계속 실행되어 리소스 누수가 발생합니다.

```kotlin
// 잘못된 패턴: CancellationException을 삼켜버림
suspend fun riskyOperation(): Data {
    return try {
        api.fetchData()
    } catch (e: Exception) { // CancellationException도 잡힘!
        fallbackData          // 코루틴 취소가 무시됨
    }
}

// 올바른 패턴 1: CancellationException 재던지기
suspend fun safeOperation(): Data {
    return try {
        api.fetchData()
    } catch (e: CancellationException) {
        throw e // 반드시 재던지기
    } catch (e: Exception) {
        fallbackData
    }
}

// 올바른 패턴 2: runCatching 활용
// Kotlin 1.8+에서 runCatching은 CancellationException을 자동으로 재던짐
suspend fun safeOperation2(): Result<Data> = runCatching {
    api.fetchData()
}
```

---

## 7. Flow의 예외 처리

`Flow`에서의 예외는 코루틴 예외와 별개로 처리됩니다.

```kotlin
// Flow 내부 예외 처리 패턴
fun getArticlesFlow(): Flow<List<Article>> = flow {
    emit(articleRepository.getArticles())
}
.catch { e ->
    // 상류(upstream) 예외를 잡아 복구
    emit(emptyList()) // 빈 리스트 emit으로 스트림 유지
}
.onEach { articles ->
    // 정상 데이터 처리
}
.flowOn(Dispatchers.IO) // 상류의 실행 컨텍스트 전환
```

> `catch` 연산자는 자신보다 **상류(upstream)**에서 발생한 예외만 처리합니다. `catch` 이후에 발생하는 예외는 별도로 처리해야 합니다.

---

## 8. 주의사항 및 실전 팁

1. **`GlobalScope` 사용 금지**: 구조화된 동시성을 깨뜨리고 예외 추적을 어렵게 만듭니다. `viewModelScope`, `lifecycleScope`, 직접 생성한 `CoroutineScope`를 사용하세요.

2. **`CoroutineExceptionHandler`는 루트에만**: 자식 코루틴에 설치된 핸들러는 무시됩니다. 반드시 `CoroutineScope` 생성 시 또는 최상위 `launch` 호출 시 전달하세요.

3. **`supervisorScope` 안에서 `async`는 `try-catch` 필수**: `supervisorScope`는 자식의 실패를 부모로 전파하지 않지만, `await()`는 예외를 다시 던집니다. `runCatching { deferred.await() }` 패턴을 습관화하세요.

4. **스코프가 취소된 후 재사용 금지**: `Job`이 취소된 스코프에서 새 코루틴을 시작하면 즉시 취소됩니다. `CoroutineExceptionHandler` 처리 후에도 동일 스코프는 재사용 불가입니다.

5. **`ensureActive()` 활용**: 긴 계산 로직 안에서 주기적으로 `ensureActive()`를 호출하면 취소 신호를 즉시 반영할 수 있습니다.

6. **테스트 시 `runTest` + `TestCoroutineScheduler` 활용**: `runTest`는 가상 시간을 제어하며, `advanceTimeBy()`로 `delay`를 즉시 진행시킬 수 있습니다.

---

## 결론

Kotlin Coroutines의 예외 처리 모델은 강력하지만 잘못 이해하면 위험합니다. 핵심은 세 가지입니다. 첫째, 기본 `Job`은 자식 실패를 상위로 전파하고 형제를 취소합니다. 둘째, `SupervisorJob`/`supervisorScope`를 사용하면 각 자식이 독립적으로 실패를 처리합니다. 셋째, `CancellationException`은 절대 삼키지 말고 재던져야 합니다. 이 세 가지를 습관으로 만들면 안정적인 비동기 코드를 작성할 수 있습니다.

---

## 참고 자료

- [Coroutine exceptions handling \| Kotlin Documentation](https://kotlinlang.org/docs/exception-handling.html)
- [Best practices for coroutines in Android \| Android Developers](https://developer.android.com/kotlin/coroutines/coroutines-best-practices)
- [Asynchronous Flow \| Kotlin Documentation](https://kotlinlang.org/docs/flow.html)
- [StateFlow and SharedFlow \| Android Developers](https://developer.android.com/kotlin/flow/stateflow-and-sharedflow)
