---
layout: post
title: "Kotlin Coroutines 테스팅 심화: runTest·TestScope·Turbine으로 비동기 코드를 완벽히 검증하는 법"
date: 2026-09-08
categories: [android, kotlin]
tags: [kotlin, coroutines, testing, runTest, TestScope, Turbine, Flow, Android, JUnit]
---

비동기 코드는 테스트하기 어렵습니다. 실제 시간을 기다려야 하고, 스레드 타이밍에 따라 결과가 달라지며, `Flow`처럼 시간에 따라 값을 방출하는 스트림은 단순한 `assertEquals`만으로는 검증할 수 없습니다. `kotlinx-coroutines-test` 라이브러리와 Turbine은 바로 이 문제를 해결하기 위해 설계되었습니다. 이 글에서는 `runTest`, `TestScope`, `TestDispatcher`, 그리고 Turbine을 활용해 코루틴과 Flow 기반 코드를 완벽히 검증하는 방법을 심층적으로 다룹니다.

---

## 왜 코루틴 테스팅이 어려운가?

기존 JUnit 환경에서 코루틴을 테스트하면 다음과 같은 문제에 부딪힙니다.

**1. `delay`가 실제로 실행됨**: 5초짜리 재시도 로직을 테스트하면 테스트 스위트 전체가 5초씩 느려집니다.

**2. 스레드 문제**: `Dispatchers.Main`은 Android 플랫폼에 묶여 있어 JVM 단위 테스트(로컬 테스트)에서는 존재하지 않습니다. ViewModel이 `Dispatchers.Main`에서 코루틴을 시작하면 테스트에서 `Module with the Main dispatcher had failed to initialize`라는 에러가 발생합니다.

**3. 실행 완료 보장 어려움**: 코루틴은 비동기로 실행되므로 테스트 함수가 끝나도 코루틴이 완료되지 않았을 수 있습니다.

**4. Hot Flow 수집**: `StateFlow`나 `SharedFlow` 같은 hot stream은 영원히 collect되므로, 테스트가 응답 없이 멈춰버립니다.

`kotlinx-coroutines-test`는 **가상 시간(virtual time)** 을 도입해 이 모든 문제를 해결합니다.

---

## 의존성 설정

```kotlin
// build.gradle.kts (module level)
dependencies {
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.8.1")
    
    // Turbine (Flow 테스팅 전용)
    testImplementation("app.cash.turbine:turbine:1.2.0")
    
    // JUnit 4 또는 JUnit 5
    testImplementation("junit:junit:4.13.2")
}
```

---

## `runTest`: 코루틴 테스팅의 핵심

`runTest`는 코루틴을 **동기적으로** 실행하는 테스트 코루틴 빌더입니다. 내부적으로 `delay`를 가상 시간으로 대체해 실제 대기 없이 즉시 실행됩니다.

```kotlin
import kotlinx.coroutines.delay
import kotlinx.coroutines.test.runTest
import org.junit.Test
import kotlin.test.assertEquals

class UserRepositoryTest {

    @Test
    fun `5초 재시도 로직이 올바르게 동작한다`() = runTest {
        var attempt = 0
        
        suspend fun fetchWithRetry(): String {
            repeat(3) {
                attempt++
                if (attempt < 3) {
                    delay(5_000L) // 실제 환경에서는 5초 대기
                    return@repeat
                }
            }
            return "success"
        }
        
        val result = fetchWithRetry()
        
        // delay(5_000)이 두 번 있어도 테스트는 즉시 완료됨
        assertEquals("success", result)
        assertEquals(3, attempt)
        
        // 가상 시간이 얼마나 흘렀는지 확인
        println("Virtual time elapsed: ${testScheduler.currentTime}ms") // 10000ms
    }
}
```

**핵심 포인트**: `delay(5_000L)`이 두 번 실행됐지만 테스트는 실제로 10초가 걸리지 않습니다. `TestCoroutineScheduler`가 가상 시간으로 처리하기 때문입니다.

---

## TestDispatcher: StandardTestDispatcher vs UnconfinedTestDispatcher

`runTest` 안에서 코루틴이 실행되는 방식을 제어하는 두 가지 Dispatcher가 있습니다.

### StandardTestDispatcher (기본값)

코루틴을 큐에 넣고 **명시적으로 진행을 유도**해야 실행됩니다. 정확한 실행 순서 제어가 필요할 때 사용합니다.

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.test.*
import org.junit.Test
import kotlin.test.assertEquals

class StandardDispatcherTest {

    @Test
    fun `StandardTestDispatcher로 코루틴 실행 순서를 제어한다`() = runTest {
        val results = mutableListOf<String>()
        
        // launch는 큐에 등록되지만 즉시 실행되지 않음
        launch {
            results.add("coroutine-1")
        }
        launch {
            results.add("coroutine-2")
        }
        
        // 이 시점에서 results는 비어 있음
        assertEquals(emptyList(), results)
        
        // 큐에 있는 모든 코루틴 실행
        runCurrent()
        assertEquals(listOf("coroutine-1", "coroutine-2"), results)
    }

    @Test
    fun `advanceTimeBy로 delay를 수동으로 조정한다`() = runTest {
        val log = mutableListOf<String>()
        
        launch {
            log.add("start")
            delay(1_000L)
            log.add("after 1s")
            delay(2_000L)
            log.add("after 3s")
        }
        
        runCurrent() // "start" 실행
        assertEquals(listOf("start"), log)
        
        advanceTimeBy(1_001L) // 1초 + 1ms 이상 진행
        runCurrent()
        assertEquals(listOf("start", "after 1s"), log)
        
        advanceUntilIdle() // 나머지 모두 실행
        assertEquals(listOf("start", "after 1s", "after 3s"), log)
    }
}
```

### UnconfinedTestDispatcher

코루틴이 `launch`되는 즉시 **현재 스레드에서 적극적으로(eagerly) 실행**됩니다. `StateFlow`를 수집할 때 첫 번째 값을 즉시 받고 싶을 때 유용합니다.

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.test.*
import org.junit.Test
import kotlin.test.assertEquals

class UnconfinedDispatcherTest {

    @Test
    fun `UnconfinedTestDispatcher로 StateFlow 즉시 수집`() = runTest {
        val stateFlow = MutableStateFlow(0)
        val collectedValues = mutableListOf<Int>()
        
        // UnconfinedTestDispatcher로 즉시 수집 시작
        backgroundScope.launch(UnconfinedTestDispatcher(testScheduler)) {
            stateFlow.collect { collectedValues.add(it) }
        }
        
        // 이미 초기값 0이 수집됨
        assertEquals(listOf(0), collectedValues)
        
        stateFlow.value = 1
        assertEquals(listOf(0, 1), collectedValues)
        
        stateFlow.value = 2
        assertEquals(listOf(0, 1, 2), collectedValues)
    }
}
```

**`backgroundScope` 사용 이유**: `backgroundScope`에서 시작한 코루틴은 테스트가 끝날 때 자동으로 취소됩니다. 일반 `launch`로 infinite flow를 수집하면 `runTest`가 해당 코루틴 완료를 기다리다 테스트가 영원히 끝나지 않습니다.

---

## Dispatchers.Main 교체: ViewModel 테스팅의 필수 패턴

ViewModel은 보통 `viewModelScope`(= `Dispatchers.Main.immediate`)에서 코루틴을 실행합니다. 로컬 단위 테스트에서는 Main dispatcher가 없으므로 교체가 필요합니다.

```kotlin
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.launch

// 테스트할 ViewModel
class CounterViewModel(
    private val repository: CounterRepository
) : ViewModel() {
    
    private val _uiState = MutableStateFlow<CounterUiState>(CounterUiState.Loading)
    val uiState: StateFlow<CounterUiState> = _uiState
    
    fun loadCount() {
        viewModelScope.launch {
            _uiState.value = CounterUiState.Loading
            val count = repository.getCount() // suspend 함수
            _uiState.value = CounterUiState.Success(count)
        }
    }
}

sealed class CounterUiState {
    object Loading : CounterUiState()
    data class Success(val count: Int) : CounterUiState()
    data class Error(val message: String) : CounterUiState()
}

interface CounterRepository {
    suspend fun getCount(): Int
}
```

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.test.*
import org.junit.After
import org.junit.Before
import org.junit.Test
import kotlin.test.assertEquals
import kotlin.test.assertIs

class CounterViewModelTest {

    private val testDispatcher = StandardTestDispatcher()
    
    @Before
    fun setUp() {
        Dispatchers.setMain(testDispatcher)
    }
    
    @After
    fun tearDown() {
        Dispatchers.resetMain()
    }
    
    @Test
    fun `loadCount 호출 시 Loading 후 Success 상태로 전환된다`() = runTest {
        val fakeRepository = object : CounterRepository {
            override suspend fun getCount(): Int {
                delay(100L) // 네트워크 지연 시뮬레이션
                return 42
            }
        }
        
        val viewModel = CounterViewModel(fakeRepository)
        val states = mutableListOf<CounterUiState>()
        
        backgroundScope.launch(UnconfinedTestDispatcher(testScheduler)) {
            viewModel.uiState.collect { states.add(it) }
        }
        
        viewModel.loadCount()
        advanceUntilIdle() // 모든 pending 코루틴 완료
        
        assertEquals(2, states.size)
        assertIs<CounterUiState.Loading>(states[0])
        assertIs<CounterUiState.Success>(states[1])
        assertEquals(42, (states[1] as CounterUiState.Success).count)
    }
}
```

### JUnit Rule로 반복 코드 제거

매 테스트 클래스마다 `setMain`/`resetMain`을 작성하는 것은 번거롭습니다. JUnit Rule로 추출하세요:

```kotlin
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.test.StandardTestDispatcher
import kotlinx.coroutines.test.TestDispatcher
import kotlinx.coroutines.test.resetMain
import kotlinx.coroutines.test.setMain
import org.junit.rules.TestWatcher
import org.junit.runner.Description

class MainDispatcherRule(
    val testDispatcher: TestDispatcher = StandardTestDispatcher()
) : TestWatcher() {
    
    override fun starting(description: Description) {
        Dispatchers.setMain(testDispatcher)
    }
    
    override fun finished(description: Description) {
        Dispatchers.resetMain()
    }
}
```

이제 테스트 클래스에서 단 한 줄로 적용할 수 있습니다:

```kotlin
class MyViewModelTest {
    
    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()
    
    @Test
    fun `MainDispatcherRule로 간결하게 테스트`() = runTest {
        // Dispatchers.Main이 자동으로 testDispatcher로 교체됨
    }
}
```

---

## Turbine: Flow 테스팅을 우아하게

`backgroundScope.launch { flow.collect { ... } }` 패턴은 작동하지만 장황합니다. Turbine은 Flow 테스팅 전용 DSL을 제공해 코드를 훨씬 간결하게 만들어 줍니다.

### 기본 사용법

```kotlin
import app.cash.turbine.test
import kotlinx.coroutines.flow.flow
import kotlinx.coroutines.test.runTest
import org.junit.Test
import kotlin.test.assertEquals

class TurbineBasicTest {

    @Test
    fun `flow 방출값을 순서대로 검증한다`() = runTest {
        val countFlow = flow {
            emit(1)
            delay(100L)
            emit(2)
            delay(100L)
            emit(3)
        }
        
        countFlow.test {
            assertEquals(1, awaitItem())
            assertEquals(2, awaitItem())
            assertEquals(3, awaitItem())
            awaitComplete() // flow 완료 확인
        }
    }
    
    @Test
    fun `flow가 에러를 방출하는지 검증한다`() = runTest {
        val errorFlow = flow<Int> {
            emit(1)
            throw RuntimeException("네트워크 오류")
        }
        
        errorFlow.test {
            assertEquals(1, awaitItem())
            val error = awaitError()
            assertEquals("네트워크 오류", error.message)
        }
    }
}
```

### ViewModel StateFlow 테스팅 with Turbine

```kotlin
import app.cash.turbine.test
import kotlinx.coroutines.*
import kotlinx.coroutines.test.*
import org.junit.Rule
import org.junit.Test
import kotlin.test.assertEquals
import kotlin.test.assertIs

class CounterViewModelTurbineTest {
    
    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()
    
    @Test
    fun `Turbine으로 uiState 전환 흐름을 검증한다`() = runTest {
        val fakeRepository = object : CounterRepository {
            override suspend fun getCount(): Int {
                delay(100L)
                return 99
            }
        }
        
        val viewModel = CounterViewModel(fakeRepository)
        
        viewModel.uiState.test {
            // 초기 상태
            assertIs<CounterUiState.Loading>(awaitItem())
            
            viewModel.loadCount()
            
            // Loading → Success 전환
            assertIs<CounterUiState.Loading>(awaitItem())
            
            val successState = awaitItem()
            assertIs<CounterUiState.Success>(successState)
            assertEquals(99, successState.count)
        }
    }
    
    @Test
    fun `여러 업데이트를 skipItems로 건너뛴다`() = runTest {
        val fakeRepository = object : CounterRepository {
            override suspend fun getCount() = 10
        }
        
        val viewModel = CounterViewModel(fakeRepository)
        
        viewModel.uiState.test {
            skipItems(1) // Loading 스킵
            viewModel.loadCount()
            skipItems(1) // 두 번째 Loading 스킵
            
            val final = awaitItem()
            assertIs<CounterUiState.Success>(final)
            assertEquals(10, final.count)
        }
    }
}
```

### turbineScope: 여러 Flow를 동시에 테스팅

```kotlin
import app.cash.turbine.turbineScope
import kotlinx.coroutines.flow.MutableSharedFlow
import kotlinx.coroutines.test.runTest
import org.junit.Test
import kotlin.test.assertEquals

class MultiplFlowTurbineTest {
    
    @Test
    fun `두 Flow를 동시에 독립적으로 검증한다`() = runTest {
        val flowA = MutableSharedFlow<Int>()
        val flowB = MutableSharedFlow<String>()
        
        turbineScope {
            val turbineA = flowA.testIn(backgroundScope)
            val turbineB = flowB.testIn(backgroundScope)
            
            flowA.emit(42)
            flowB.emit("hello")
            flowA.emit(100)
            
            assertEquals(42, turbineA.awaitItem())
            assertEquals("hello", turbineB.awaitItem())
            assertEquals(100, turbineA.awaitItem())
            
            turbineA.cancel()
            turbineB.cancel()
        }
    }
}
```

---

## 실전 패턴: Repository 레이어 테스팅

네트워크 + 캐시 로직을 가진 Repository를 테스트하는 완전한 예제입니다:

```kotlin
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.flow

// 도메인 레이어
data class Article(val id: Int, val title: String)

interface ArticleApi {
    suspend fun fetchArticles(): List<Article>
}

interface ArticleCache {
    suspend fun getArticles(): List<Article>?
    suspend fun saveArticles(articles: List<Article>)
}

class ArticleRepository(
    private val api: ArticleApi,
    private val cache: ArticleCache
) {
    fun getArticles(): Flow<List<Article>> = flow {
        // 1. 캐시 먼저 방출
        val cached = cache.getArticles()
        if (cached != null) emit(cached)
        
        // 2. 네트워크 요청 후 최신 데이터 방출
        val fresh = api.fetchArticles()
        cache.saveArticles(fresh)
        emit(fresh)
    }
}
```

```kotlin
import app.cash.turbine.test
import kotlinx.coroutines.test.runTest
import org.junit.Test
import kotlin.test.assertEquals

class ArticleRepositoryTest {
    
    private val cachedArticles = listOf(
        Article(1, "캐시된 아티클")
    )
    private val freshArticles = listOf(
        Article(1, "최신 아티클"),
        Article(2, "새로운 아티클")
    )
    
    @Test
    fun `캐시가 있을 때 캐시 먼저, 네트워크 결과 나중에 방출한다`() = runTest {
        val fakeApi = object : ArticleApi {
            override suspend fun fetchArticles() = freshArticles
        }
        val fakeCache = object : ArticleCache {
            private var stored: List<Article>? = cachedArticles
            override suspend fun getArticles() = stored
            override suspend fun saveArticles(articles: List<Article>) { stored = articles }
        }
        
        val repository = ArticleRepository(fakeApi, fakeCache)
        
        repository.getArticles().test {
            // 캐시 데이터 먼저
            val first = awaitItem()
            assertEquals(1, first.size)
            assertEquals("캐시된 아티클", first[0].title)
            
            // 네트워크 최신 데이터
            val second = awaitItem()
            assertEquals(2, second.size)
            assertEquals("최신 아티클", second[0].title)
            
            awaitComplete()
        }
    }
    
    @Test
    fun `캐시가 없을 때 네트워크 데이터만 방출한다`() = runTest {
        val fakeApi = object : ArticleApi {
            override suspend fun fetchArticles() = freshArticles
        }
        val fakeCache = object : ArticleCache {
            private var stored: List<Article>? = null
            override suspend fun getArticles() = stored
            override suspend fun saveArticles(articles: List<Article>) { stored = articles }
        }
        
        val repository = ArticleRepository(fakeApi, fakeCache)
        
        repository.getArticles().test {
            val only = awaitItem()
            assertEquals(2, only.size)
            awaitComplete()
        }
    }
}
```

---

## 주의사항 및 팁

### 1. `advanceUntilIdle` vs `runCurrent` 선택

| 함수 | 동작 | 사용 시점 |
|------|------|-----------|
| `runCurrent()` | 현재 시각에 예약된 코루틴만 실행 | 실행 순서를 단계별로 제어할 때 |
| `advanceTimeBy(ms)` | 지정한 시간만큼 가상 시간 진행 | 특정 delay 이후 상태 검증 시 |
| `advanceUntilIdle()` | 더 이상 실행할 코루틴이 없을 때까지 진행 | 모든 코루틴이 완료되길 기다릴 때 |

### 2. TestDispatcher를 공유해야 할 때

여러 컴포넌트가 각자 다른 TestDispatcher를 쓰면 가상 시간이 동기화되지 않아 예측 불가능한 결과가 나올 수 있습니다. 항상 같은 `TestCoroutineScheduler` 인스턴스를 공유하세요:

```kotlin
val scheduler = TestCoroutineScheduler()
val dispatcher1 = StandardTestDispatcher(scheduler)
val dispatcher2 = StandardTestDispatcher(scheduler)
```

### 3. Turbine 타임아웃 설정

기본 타임아웃은 1초입니다. 느린 환경에서는 조정이 필요할 수 있습니다:

```kotlin
flowUnderTest.test(timeout = 5.seconds) {
    // 느린 CI 환경을 위한 넉넉한 타임아웃
    assertEquals(expected, awaitItem())
}
```

### 4. `ensureAllEventsConsumed` 활용

Turbine의 `test { }` 블록은 끝날 때 자동으로 미수집 이벤트가 없는지 확인합니다. 예상치 못한 추가 방출이 있으면 테스트가 실패하므로, 암묵적인 방출 검증 도구로도 활용할 수 있습니다.

### 5. `runTest`는 중첩 불가

`runTest` 안에서 또 다른 `runTest`를 호출하지 마세요. 중첩이 필요한 경우 `coroutineScope { }` 또는 `withContext`를 사용하세요.

---

## 마무리

Kotlin 코루틴 테스팅은 처음에는 복잡해 보이지만, 핵심 개념을 이해하면 강력한 도구가 됩니다.

- **`runTest`**: 가상 시간으로 `delay`를 즉시 처리하는 테스트 코루틴 빌더
- **`StandardTestDispatcher`**: 명시적 진행 제어가 필요할 때
- **`UnconfinedTestDispatcher`**: Flow 수집 등 즉각적 실행이 필요할 때
- **`backgroundScope`**: 무한 flow를 테스트가 끝날 때 자동 취소
- **`MainDispatcherRule`**: ViewModel 테스팅을 위한 재사용 가능한 JUnit Rule
- **Turbine**: Flow 방출을 선언적이고 간결하게 검증

비동기 로직에 대한 탄탄한 테스트는 리팩토링 안정성과 코드 신뢰도를 극적으로 높여줍니다. 오늘 당장 프로젝트의 Repository와 ViewModel 테스트에 적용해 보세요.

## 참고 자료
- [Testing Kotlin coroutines on Android (공식 문서)](https://developer.android.com/kotlin/coroutines/test)
- [Testing Kotlin flows on Android (공식 문서)](https://developer.android.com/kotlin/flow/test)
- [Turbine GitHub Repository](https://github.com/cashapp/turbine)
