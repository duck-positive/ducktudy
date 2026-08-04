---
layout: post
title: "Android MVI 아키텍처 심화: StateFlow·Channel·Contract 패턴으로 완성하는 단방향 데이터 흐름"
date: 2026-08-04
categories: [android, flutter]
tags: [android, mvi, stateflow, channel, kotlin, coroutines, jetpack-compose, architecture]
---

Android 앱의 복잡성이 증가할수록 UI 상태를 안전하게 관리하는 아키텍처 패턴의 중요성도 커집니다. Google이 공식적으로 MVVM과 단방향 데이터 흐름(UDF, Unidirectional Data Flow)을 권장하는 가운데, 더욱 엄격한 상태 일관성을 요구하는 프로젝트에서 **MVI(Model-View-Intent)** 패턴이 주목받고 있습니다. 이 글에서는 Kotlin Coroutines의 `StateFlow`와 `Channel`을 기반으로 실용적인 MVI 패턴을 구현하는 방법을 심층적으로 살펴봅니다.

---

## MVI 패턴이란?

MVI는 **Model-View-Intent**의 약자로, 세 가지 핵심 구성요소로 이루어집니다.

- **Model**: 화면 전체의 UI 상태를 나타내는 **불변(immutable) 데이터 클래스**입니다. 화면에 렌더링해야 할 모든 정보를 단 하나의 객체로 표현합니다.
- **View**: UI 컴포넌트(Activity, Fragment, Composable)로, Model을 화면에 렌더링하고 사용자 이벤트를 Intent로 변환하여 위로 전달합니다.
- **Intent**: 사용자의 액션이나 이벤트를 추상화한 것입니다. 버튼 클릭, 텍스트 입력, 화면 진입 등이 Intent(또는 Event)가 됩니다.

MVI의 핵심은 **단방향 데이터 흐름**입니다. 데이터는 항상 한 방향으로만 흐릅니다.

```
View → Intent(Event) → ViewModel(처리) → State → View
```

이 순환 구조는 어떤 시점이든 UI의 상태가 단 하나의 `State` 객체에 의해 완전히 결정됨을 보장합니다.

---

## 왜 MVI가 필요한가?

### MVVM의 상태 불일치 문제

기존 MVVM에서는 ViewModel이 여러 개의 `LiveData` 또는 `StateFlow`를 독립적으로 노출하는 경우가 많습니다.

```kotlin
// 전형적인 MVVM — 상태가 여러 프로퍼티로 분산됨
class UserListViewModel : ViewModel() {
    val isLoading = MutableStateFlow(false)
    val users = MutableStateFlow<List<User>>(emptyList())
    val errorMessage = MutableStateFlow<String?>(null)
    val selectedFilter = MutableStateFlow(FilterType.ALL)
}
```

이 방식의 문제점은 상태들 사이에 논리적 연관관계가 있음에도 불구하고 각각 독립적으로 업데이트될 수 있다는 점입니다. `isLoading = true` 상태에서 `users`에 데이터가 존재하거나, `errorMessage`가 null이 아닌데 `users`도 비어 있지 않은 **모순된 상태**가 생길 수 있습니다. UI가 이러한 불일치 상황을 매번 처리해야 하므로 View 코드가 복잡해집니다.

### MVI가 제공하는 보장

MVI는 다음 네 가지를 제공합니다.

1. **단일 진실 공급원(Single Source of Truth)**: 모든 UI 상태가 하나의 `State` 객체로 통합됩니다. 불일치 자체가 구조적으로 불가능해집니다.
2. **예측 가능성**: 현재 `State`와 발생한 `Event`를 알면 다음 `State`를 언제나 예측할 수 있습니다.
3. **디버깅 용이성**: 상태 변화 이력을 로그로 남기거나 타임트래블 디버깅을 적용하기 쉽습니다.
4. **높은 테스트 가능성**: ViewModel의 상태 변환 로직이 순수 함수에 가까워 단위 테스트 작성이 쉬워집니다.

---

## 실제 구현 예제 1 — MVI Contract 정의와 ViewModel

MVI를 구현할 때 가장 널리 사용되는 패턴은 **Contract 인터페이스**로 화면별 `State`, `Event`, `Effect`를 한곳에 모으는 것입니다. `Effect`(또는 Side Effect)는 Toast, Navigation, Snackbar처럼 **일회성으로만 처리**해야 하는 이벤트를 의미합니다.

```kotlin
// UserListContract.kt — 화면의 MVI 계약
interface UserListContract {

    // 불변 UI 상태. 화면에 필요한 모든 정보를 하나로 통합
    data class State(
        val isLoading: Boolean = false,
        val users: List<User> = emptyList(),
        val errorMessage: String? = null,
        val filterType: FilterType = FilterType.ALL,
        val searchQuery: String = ""
    ) {
        // 파생 상태: 필터·검색을 적용한 목록 (별도 StateFlow 불필요)
        val filteredUsers: List<User>
            get() = users.filter { user ->
                (filterType == FilterType.ALL || user.type == filterType) &&
                (searchQuery.isEmpty() || user.name.contains(searchQuery, ignoreCase = true))
            }
    }

    // 사용자 액션 (Intent). View → ViewModel 방향
    sealed class Event {
        object LoadUsers : Event()
        data class SearchQueryChanged(val query: String) : Event()
        data class FilterChanged(val filterType: FilterType) : Event()
        data class UserClicked(val userId: String) : Event()
        data class UserDeleted(val userId: String) : Event()
        object RetryClicked : Event()
    }

    // 일회성 사이드 이펙트. Navigation, Toast 등 StateFlow에 넣으면 안 되는 것들
    sealed class Effect {
        data class NavigateToDetail(val userId: String) : Effect()
        data class ShowSnackbar(val message: String) : Effect()
        object ScrollToTop : Effect()
    }
}

enum class FilterType { ALL, ACTIVE, INACTIVE }
data class User(val id: String, val name: String, val type: FilterType)
```

Contract를 정의했으면 ViewModel을 구현합니다. 상태는 `MutableStateFlow`로, 일회성 이펙트는 `Channel`로 분리하는 것이 핵심입니다. `StateFlow`는 최신 값을 보존하고 재구독 시 즉시 방출하기 때문에 Navigation처럼 한 번만 처리돼야 하는 이벤트에 적합하지 않습니다. `Channel`은 이를 해결합니다.

```kotlin
// UserListViewModel.kt
@HiltViewModel
class UserListViewModel @Inject constructor(
    private val userRepository: UserRepository
) : ViewModel() {

    // 외부에는 읽기 전용 StateFlow로 노출
    private val _state = MutableStateFlow(UserListContract.State())
    val state: StateFlow<UserListContract.State> = _state.asStateFlow()

    // 일회성 이펙트: Channel → Flow 변환으로 노출
    // Channel.BUFFERED로 구독 전 발행된 이펙트도 유실되지 않게 함
    private val _effect = Channel<UserListContract.Effect>(Channel.BUFFERED)
    val effect: Flow<UserListContract.Effect> = _effect.receiveAsFlow()

    init {
        handleEvent(UserListContract.Event.LoadUsers)
    }

    // 외부의 단일 진입점. View는 이 함수만 호출
    fun handleEvent(event: UserListContract.Event) {
        when (event) {
            is UserListContract.Event.LoadUsers -> loadUsers()
            is UserListContract.Event.RetryClicked -> loadUsers()
            is UserListContract.Event.SearchQueryChanged -> updateState {
                copy(searchQuery = event.query)
            }
            is UserListContract.Event.FilterChanged -> {
                updateState { copy(filterType = event.filterType) }
                sendEffect(UserListContract.Effect.ScrollToTop)
            }
            is UserListContract.Event.UserClicked ->
                sendEffect(UserListContract.Effect.NavigateToDetail(event.userId))
            is UserListContract.Event.UserDeleted -> deleteUser(event.userId)
        }
    }

    private fun loadUsers() {
        viewModelScope.launch {
            updateState { copy(isLoading = true, errorMessage = null) }
            userRepository.getUsers()
                .onSuccess { users ->
                    updateState { copy(isLoading = false, users = users) }
                }
                .onFailure { e ->
                    updateState {
                        copy(isLoading = false, errorMessage = e.message ?: "오류가 발생했습니다.")
                    }
                }
        }
    }

    private fun deleteUser(userId: String) {
        viewModelScope.launch {
            userRepository.deleteUser(userId)
                .onSuccess {
                    updateState { copy(users = users.filter { it.id != userId }) }
                    sendEffect(UserListContract.Effect.ShowSnackbar("사용자가 삭제되었습니다."))
                }
                .onFailure {
                    sendEffect(UserListContract.Effect.ShowSnackbar("삭제에 실패했습니다."))
                }
        }
    }

    // 상태 업데이트 헬퍼: reducer 함수를 받아 atomic하게 갱신
    private fun updateState(reducer: UserListContract.State.() -> UserListContract.State) {
        _state.update { it.reducer() }
    }

    private fun sendEffect(effect: UserListContract.Effect) {
        viewModelScope.launch { _effect.send(effect) }
    }
}
```

---

## 실제 구현 예제 2 — Jetpack Compose UI 연동

ViewModel을 Compose에 연결할 때는 `collectAsStateWithLifecycle()`로 상태를 구독하고, `LaunchedEffect`로 일회성 이펙트를 처리합니다.

```kotlin
// UserListScreen.kt
@Composable
fun UserListScreen(
    viewModel: UserListViewModel = hiltViewModel(),
    onNavigateToDetail: (String) -> Unit
) {
    val state by viewModel.state.collectAsStateWithLifecycle()
    val snackbarHostState = remember { SnackbarHostState() }
    val listState = rememberLazyListState()
    val scope = rememberCoroutineScope()

    // Effect 수신: LaunchedEffect로 한 번만 구독
    LaunchedEffect(Unit) {
        viewModel.effect.collect { effect ->
            when (effect) {
                is UserListContract.Effect.NavigateToDetail ->
                    onNavigateToDetail(effect.userId)
                is UserListContract.Effect.ShowSnackbar ->
                    snackbarHostState.showSnackbar(effect.message)
                is UserListContract.Effect.ScrollToTop ->
                    scope.launch { listState.animateScrollToItem(0) }
            }
        }
    }

    Scaffold(snackbarHost = { SnackbarHost(snackbarHostState) }) { padding ->
        Column(
            modifier = Modifier
                .fillMaxSize()
                .padding(padding)
        ) {
            // 검색창: 이벤트만 올리고 상태는 State에서 읽음
            OutlinedTextField(
                value = state.searchQuery,
                onValueChange = {
                    viewModel.handleEvent(UserListContract.Event.SearchQueryChanged(it))
                },
                placeholder = { Text("검색") },
                modifier = Modifier
                    .fillMaxWidth()
                    .padding(horizontal = 16.dp, vertical = 8.dp)
            )

            // 필터 탭
            FilterType.entries.forEach { filter ->
                FilterChip(
                    selected = state.filterType == filter,
                    onClick = {
                        viewModel.handleEvent(UserListContract.Event.FilterChanged(filter))
                    },
                    label = { Text(filter.name) }
                )
            }

            // 본문: State의 단일 필드만 보고 렌더링 분기
            when {
                state.isLoading -> CircularProgressIndicator(modifier = Modifier.align(Alignment.CenterHorizontally))
                state.errorMessage != null -> {
                    Text(text = state.errorMessage!!, color = MaterialTheme.colorScheme.error)
                    Button(onClick = { viewModel.handleEvent(UserListContract.Event.RetryClicked) }) {
                        Text("재시도")
                    }
                }
                state.filteredUsers.isEmpty() -> Text("표시할 사용자가 없습니다.")
                else -> LazyColumn(state = listState) {
                    items(state.filteredUsers, key = { it.id }) { user ->
                        UserListItem(
                            user = user,
                            onClick = {
                                viewModel.handleEvent(UserListContract.Event.UserClicked(user.id))
                            },
                            onDelete = {
                                viewModel.handleEvent(UserListContract.Event.UserDeleted(user.id))
                            }
                        )
                    }
                }
            }
        }
    }
}
```

---

## 주의사항 및 실전 팁

### 1. Effect에 StateFlow를 사용하지 말 것

Navigation 이벤트나 Toast를 `StateFlow`로 노출하면 화면 회전 시 같은 이벤트가 재발행될 수 있습니다. **일회성 이벤트는 반드시 `Channel`** 을 사용하세요. `Channel.BUFFERED` 또는 `Channel.UNLIMITED`를 설정해 구독 전 발행된 이펙트가 유실되지 않도록 합니다.

### 2. State는 항상 불변으로

`State` data class의 모든 프로퍼티는 `val`로 선언해야 합니다. `copy()`를 통해서만 새로운 State를 생성하고, ViewModel 내부에서만 `_state`를 갱신합니다. `var` 프로퍼티나 가변 컬렉션(`MutableList`)이 State에 포함되면 MVI의 예측 가능성이 무너집니다.

### 3. 파생 상태는 State 내부에 계산 프로퍼티로

필터링된 목록처럼 다른 State 필드로부터 계산되는 값은 별도의 StateFlow를 만들지 말고 `State` data class 내부의 `val` 계산 프로퍼티로 선언하세요. `StateFlow`를 추가하면 다시 MVVM의 분산 상태 문제로 돌아가게 됩니다.

### 4. handleEvent는 View의 유일한 진입점으로

View는 ViewModel의 어떤 함수도 직접 호출하지 말고 오직 `handleEvent(event)`만 호출해야 합니다. 이 규칙이 지켜져야 모든 상태 변화를 단일 경로에서 추적할 수 있습니다.

### 5. 테스트 작성이 쉬워집니다

MVI 패턴에서 ViewModel 테스트는 "주어진 초기 State에서 특정 Event를 처리하면 예상한 State가 되는가"를 검증하는 구조가 됩니다. `Turbine` 라이브러리를 사용하면 `StateFlow`와 `Channel`의 방출 순서를 순서대로 단언(assert)할 수 있어 테스트 코드가 선언적이고 명확해집니다.

---

## 정리

MVI 패턴은 UI 상태 관리의 복잡성이 증가하는 현대 Android 앱에서 MVVM의 한계를 보완하는 강력한 대안입니다. `StateFlow`로 단일 불변 상태를 관리하고, `Channel`로 일회성 이펙트를 분리하며, Contract 패턴으로 화면별 명세를 한곳에 모으면 유지보수성과 테스트 가능성을 동시에 얻을 수 있습니다. 기존 MVVM 코드베이스에 점진적으로 도입하는 것도 어렵지 않으므로, 상태 불일치로 인한 버그가 자주 발생하는 화면부터 적용해 보길 권장합니다.

## 참고 자료
- [UI layer — App architecture \| Android Developers](https://developer.android.com/topic/architecture/ui-layer)
- [State holders and UI state — App architecture \| Android Developers](https://developer.android.com/topic/architecture/ui-layer/stateholders)
- [UI events — App architecture \| Android Developers](https://developer.android.com/topic/architecture/ui-layer/events)
- [Channels \| Kotlin Documentation](https://kotlinlang.org/docs/channels.html)
