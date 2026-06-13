---
layout: post
title: "Kotlin Flow 심화: StateFlow · SharedFlow · callbackFlow 완전 정복"
date: 2026-06-13
categories: [android, flutter]
tags: [kotlin, coroutines, flow, stateflow, sharedflow, callbackflow, android, mvvm]
---

## 들어가며

Kotlin Coroutines의 `Flow`는 이제 Android 개발의 표준 비동기 스트림 처리 도구로 자리 잡았습니다. 그러나 단순히 `flow { emit(...) }` 블록을 사용하는 것을 넘어, **StateFlow**, **SharedFlow**, **callbackFlow** 각각의 특성과 차이를 정확히 이해해야 프로덕션 수준의 앱을 만들 수 있습니다. 이 글에서는 세 가지 Flow 유형의 내부 동작 원리부터 실전 패턴, 그리고 흔히 발생하는 실수까지 심층적으로 다룹니다.

---

## 1. Cold Flow vs Hot Flow — 핵심 개념 차이

Flow를 이해하는 첫 단계는 **Cold Flow**와 **Hot Flow**의 차이를 명확히 아는 것입니다.

- **Cold Flow**: `collect()`가 호출될 때마다 블록이 새로 실행됩니다. 구독자 수만큼 독립적인 스트림이 만들어집니다. 일반 `flow { }` 빌더가 이에 해당합니다.
- **Hot Flow**: 구독자와 무관하게 이미 실행 중인 스트림입니다. `StateFlow`와 `SharedFlow`가 여기에 속합니다. 구독자가 없어도 값이 업데이트될 수 있으며, 새 구독자는 중간부터 참여합니다.

이 차이를 모르고 `StateFlow`를 Cold Flow처럼 다루면 **메모리 누수**나 **불필요한 중복 연산** 문제가 생깁니다.

---

## 2. StateFlow — 단일 상태를 위한 Hot Flow

### 개념

`StateFlow`는 항상 **하나의 최신 값(state)**을 보유하는 Hot Flow입니다. 내부적으로 `SharedFlow(replay=1, onBufferOverflow=DROP_OLDEST)`와 동일하게 동작하지만, 몇 가지 중요한 차이가 있습니다.

- 초기값이 반드시 필요합니다 (`MutableStateFlow(initialValue)`)
- `.value` 프로퍼티로 현재 값을 동기적으로 읽을 수 있습니다
- **동등성 기반 중복 제거(Distinct Until Changed)**가 내장되어 있습니다 — 같은 값을 emit해도 새 이벤트를 발생시키지 않습니다

### 왜 필요한가?

ViewModel에서 UI 상태를 관리할 때 `LiveData` 대신 `StateFlow`를 사용하면 Coroutines 생태계와 자연스럽게 통합되고, 테스트 코드 작성이 훨씬 쉬워집니다. 특히 `Lifecycle.repeatOnLifecycle`과 함께 사용하면 생명주기 안전한 UI 업데이트를 간결하게 구현할 수 있습니다.

### 실전 구현 예제 1 — ViewModel과 StateFlow

```kotlin
// UiState 정의
data class NewsUiState(
    val articles: List<Article> = emptyList(),
    val isLoading: Boolean = false,
    val errorMessage: String? = null
)

class NewsViewModel(
    private val repository: NewsRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow(NewsUiState(isLoading = true))
    val uiState: StateFlow<NewsUiState> = _uiState.asStateFlow()

    init {
        loadNews()
    }

    private fun loadNews() {
        viewModelScope.launch {
            repository.getLatestNews()
                .catch { e ->
                    _uiState.update { it.copy(isLoading = false, errorMessage = e.message) }
                }
                .collect { articles ->
                    _uiState.update { it.copy(articles = articles, isLoading = false) }
                }
        }
    }
}

// Activity에서 관찰
class NewsActivity : AppCompatActivity() {
    private val viewModel: NewsViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        lifecycleScope.launch {
            // repeatOnLifecycle: STARTED 이하에서는 collect를 자동 취소/재개
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect { state ->
                    updateUi(state)
                }
            }
        }
    }

    private fun updateUi(state: NewsUiState) {
        binding.progressBar.isVisible = state.isLoading
        binding.errorText.text = state.errorMessage ?: ""
        articlesAdapter.submitList(state.articles)
    }
}
```

### stateIn 연산자로 Cold Flow를 StateFlow로 변환

Repository에서 Room이나 네트워크 스트림을 반환할 때, ViewModel에서 `stateIn`으로 변환하면 업스트림 Flow를 효율적으로 관리할 수 있습니다.

```kotlin
val uiState: StateFlow<NewsUiState> = repository.getLatestNews()
    .map { articles -> NewsUiState(articles = articles) }
    .catch { e -> emit(NewsUiState(errorMessage = e.message)) }
    .stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(stopTimeoutMillis = 5_000L),
        initialValue = NewsUiState(isLoading = true)
    )
```

`WhileSubscribed(5000)`은 화면 회전(구성 변경) 시 5초 동안 업스트림을 유지했다가 이후 취소합니다. 이 덕분에 불필요한 네트워크 재요청을 막으면서도 메모리 누수를 방지할 수 있습니다.

---

## 3. SharedFlow — 다수 구독자를 위한 이벤트 스트림

### 개념

`SharedFlow`는 **여러 구독자에게 동시에 값을 브로드캐스팅**할 수 있는 Hot Flow입니다. `StateFlow`와 달리 초기값이 없으며, 동등성 기반 중복 제거도 없습니다. 구성 가능한 핵심 파라미터는 다음과 같습니다.

- `replay`: 새 구독자에게 최근 N개의 값을 다시 전달할 개수 (기본값 0)
- `extraBufferCapacity`: 추가 버퍼 용량
- `onBufferOverflow`: 버퍼가 가득 찼을 때 동작 (`SUSPEND`, `DROP_LATEST`, `DROP_OLDEST`)

### 왜 필요한가?

`StateFlow`는 "현재 상태"를 표현하기에 적합하지만, **일회성 이벤트**(토스트 메시지, 네비게이션 이동, 에러 다이얼로그)를 표현하기에는 적합하지 않습니다. 화면 회전 시 상태가 재구독되면 같은 이벤트가 다시 실행될 수 있기 때문입니다. 이럴 때 `SharedFlow(replay=0)`이 최적의 선택입니다.

### 실전 구현 예제 2 — 일회성 이벤트 처리

```kotlin
// UiEvent 정의
sealed class UiEvent {
    data class ShowToast(val message: String) : UiEvent()
    data class NavigateTo(val route: String) : UiEvent()
    object ShowErrorDialog : UiEvent()
}

class LoginViewModel(
    private val authRepository: AuthRepository
) : ViewModel() {

    // replay=0: 새 구독자에게 과거 이벤트 재전달 없음
    private val _events = MutableSharedFlow<UiEvent>(
        replay = 0,
        extraBufferCapacity = 16,
        onBufferOverflow = BufferOverflow.DROP_OLDEST
    )
    val events: SharedFlow<UiEvent> = _events.asSharedFlow()

    fun onLoginClicked(email: String, password: String) {
        viewModelScope.launch {
            when (val result = authRepository.login(email, password)) {
                is AuthResult.Success -> {
                    _events.emit(UiEvent.NavigateTo("home"))
                }
                is AuthResult.Failure -> {
                    _events.emit(UiEvent.ShowToast(result.message))
                }
                AuthResult.NetworkError -> {
                    _events.emit(UiEvent.ShowErrorDialog)
                }
            }
        }
    }
}

// Fragment에서 관찰
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.events.collect { event ->
            when (event) {
                is UiEvent.ShowToast -> Toast.makeText(context, event.message, Toast.LENGTH_SHORT).show()
                is UiEvent.NavigateTo -> findNavController().navigate(event.route)
                UiEvent.ShowErrorDialog -> showErrorDialog()
            }
        }
    }
}
```

---

## 4. callbackFlow — 콜백 API를 Flow로 변환하기

### 개념

레거시 콜백 API(예: Firebase Realtime Database, 위치 API, 센서)를 Flow로 감싸야 할 때 `callbackFlow`를 사용합니다. 일반 `flow { }` 빌더는 단일 코루틴 내에서만 `emit()`을 호출할 수 있는 반면, `callbackFlow`는 **외부 스레드에서도 안전하게 값을 전달**할 수 있습니다.

### 핵심 요소

- `trySend(value)`: 채널에 값을 전달 (스레드 안전)
- `awaitClose { }`: Flow가 취소될 때 리소스를 정리하는 필수 블록 — 누락 시 `IllegalStateException` 발생

### 실전 구현 예제 3 — Firebase 리스너를 Flow로 래핑

```kotlin
// Firebase Firestore 실시간 리스너를 Flow로 변환
fun FirebaseFirestore.observeDocument(
    collection: String,
    documentId: String
): Flow<DocumentSnapshot?> = callbackFlow {

    val docRef = collection(collection).document(documentId)

    val listener = docRef.addSnapshotListener { snapshot, error ->
        if (error != null) {
            // 에러 발생 시 채널을 닫아 Flow를 종료
            close(error)
            return@addSnapshotListener
        }
        // 외부 Firestore 스레드에서 안전하게 전달
        trySend(snapshot).isSuccess
    }

    // Flow가 취소(구독 해지)되면 Firestore 리스너 제거
    awaitClose {
        listener.remove()
    }
}

// 사용 예시
class UserProfileViewModel(
    private val firestore: FirebaseFirestore
) : ViewModel() {

    val userProfile: StateFlow<UserProfile?> = firestore
        .observeDocument("users", currentUserId)
        .map { snapshot -> snapshot?.toObject(UserProfile::class.java) }
        .catch { e -> emit(null) }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            initialValue = null
        )
}
```

---

## 5. 주의사항 및 실전 팁

### 팁 1: `collect` 위치에 따른 생명주기 이슈

```kotlin
// ❌ 잘못된 방법 — Activity가 백그라운드로 가도 계속 collect
lifecycleScope.launch {
    viewModel.uiState.collect { ... }
}

// ✅ 올바른 방법 — STARTED 미만에서 자동 취소/재개
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.uiState.collect { ... }
    }
}
```

### 팁 2: StateFlow의 동등성 함정

```kotlin
// ❌ data class가 아닌 List를 직접 update하면 같은 참조로 인식되어 emit 안 됨
_uiState.update { it.copy(articles = it.articles.also { list -> list.add(newArticle) }) }

// ✅ 항상 새 컬렉션을 만들어야 StateFlow가 변경을 감지함
_uiState.update { it.copy(articles = it.articles + newArticle) }
```

### 팁 3: SharedFlow 구독자 수 확인

```kotlin
// 구독자가 없을 때 이벤트를 emit하면 DROP_OLDEST 정책에 따라 소실될 수 있음
// subscriptionCount로 확인 가능
val hasSubscribers = _events.subscriptionCount.value > 0
```

### 팁 4: `shareIn` vs `stateIn` 선택 기준

| 특성 | `stateIn` | `shareIn` |
|------|-----------|-----------|
| 초기값 | 필요 | 불필요 |
| `.value` 접근 | 가능 | 불가능 |
| replay 기본값 | 1 | 설정 가능 |
| 용도 | UI 상태 | 이벤트/공유 스트림 |

### 팁 5: 테스트 시 `turbine` 라이브러리 활용

```kotlin
// build.gradle.kts
testImplementation("app.cash.turbine:turbine:1.1.0")

// 테스트 코드
@Test
fun `news loading success test`() = runTest {
    val viewModel = NewsViewModel(fakeRepository)
    viewModel.uiState.test {
        // 초기 로딩 상태
        assertThat(awaitItem().isLoading).isTrue()
        // 데이터 로드 완료 상태
        assertThat(awaitItem().articles).isNotEmpty()
        cancelAndIgnoreRemainingEvents()
    }
}
```

---

## 마치며

`StateFlow`, `SharedFlow`, `callbackFlow`는 각각 명확한 용도가 있습니다.

- **StateFlow**: 단일 UI 상태를 표현할 때. 항상 최신 값이 필요한 경우.
- **SharedFlow**: 일회성 이벤트, 다수 구독자에게 브로드캐스팅할 때.
- **callbackFlow**: 레거시 콜백 API를 Flow 세계로 끌어들일 때.

세 가지를 올바르게 조합하면 ViewModel → Repository → DataSource 계층 전체를 Coroutines/Flow로 일관성 있게 구성할 수 있으며, 생명주기 관리, 오류 처리, 테스트 용이성 모든 측면에서 LiveData 기반 아키텍처보다 훨씬 강력한 코드를 작성할 수 있습니다.

---

## 참고 자료
- [Kotlin flows on Android — Android Developers](https://developer.android.com/kotlin/flow)
- [StateFlow and SharedFlow — Android Developers](https://developer.android.com/kotlin/flow/stateflow-and-sharedflow)
- [StateFlow API — kotlinx.coroutines](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-state-flow/)
- [callbackFlow API — kotlinx.coroutines](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/callback-flow.html)
