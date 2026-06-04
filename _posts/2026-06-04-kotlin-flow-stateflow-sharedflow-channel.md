---
layout: post
title: "Android Kotlin Flow 심화: StateFlow, SharedFlow, Channel 완벽 가이드"
date: 2026-06-04
categories: [android, kotlin]
tags: [kotlin, coroutines, flow, stateflow, sharedflow, channel, android, mvvm, viewmodel]
---

## 개요

Android 앱 개발에서 비동기 데이터 스트림 처리는 피할 수 없는 주제입니다. Kotlin Coroutines와 함께 등장한 Flow API는 RxJava의 복잡성을 줄이면서도 강력한 반응형 프로그래밍을 가능하게 합니다. 그 중에서도 **StateFlow**, **SharedFlow**, **Channel**은 실무에서 가장 자주 사용되는 Hot Flow 타입으로, 각각의 특성을 정확히 이해하고 올바른 상황에 적용하는 것이 고품질 Android 앱 개발의 핵심입니다.

이 글에서는 세 가지 타입의 개념적 차이부터 실제 MVVM 아키텍처에서의 활용 패턴, 그리고 흔히 발생하는 실수와 해결책까지 심층적으로 다룹니다.

---

## 1. 개념 이해: Cold Flow vs Hot Flow

### Cold Flow (기본 Flow)

일반적인 `Flow`는 Cold Flow입니다. 각 구독자(collector)가 `collect`를 호출할 때마다 새로운 데이터 스트림이 시작됩니다. 즉, 구독자가 없으면 데이터 생산 자체가 일어나지 않습니다.

```kotlin
val coldFlow = flow {
    println("생산 시작")
    emit(1)
    emit(2)
    emit(3)
}

// 각 collect마다 "생산 시작"이 출력됨
coldFlow.collect { println(it) }
coldFlow.collect { println(it) }
```

### Hot Flow

Hot Flow는 구독자의 존재 여부와 무관하게 독립적으로 실행됩니다. `StateFlow`와 `SharedFlow`가 대표적인 Hot Flow입니다. 구독자가 없어도 데이터를 방출하며, 새 구독자는 중간에 참여할 수 있습니다.

| 구분 | Cold Flow | Hot Flow |
|------|-----------|----------|
| 데이터 생산 시점 | collect 시작 시 | 독립적으로 진행 |
| 구독자 수 | 1:1 | 1:N (멀티캐스트) |
| 대표 타입 | `flow { }` | StateFlow, SharedFlow |

---

## 2. StateFlow: 상태를 위한 Flow

### 개념

`StateFlow`는 **항상 하나의 값을 보유**하는 상태 홀더(State Holder)입니다. LiveData의 대체재로 자주 사용되며 다음 특성을 가집니다:

- **초기값 필수**: 생성 시 반드시 초기값을 지정해야 합니다.
- **replay = 1**: 새 구독자는 즉시 현재 상태값을 받습니다.
- **중복 값 무시**: 동일한 값이 연속 emit되어도 한 번만 전달됩니다 (`distinctUntilChanged` 내장).
- **스레드 안전**: `value` 프로퍼티 읽기/쓰기가 스레드 안전하게 동작합니다.

### LiveData 대비 StateFlow의 장점

- Kotlin Coroutines와 자연스러운 통합
- 안드로이드 플랫폼에 종속되지 않아 순수 Kotlin 모듈에서도 사용 가능
- 강력한 Flow 연산자(`map`, `filter`, `combine` 등) 체이닝 가능
- 테스트 용이성 (Turbine 라이브러리 활용)

---

## 3. 실제 구현 예제 1: StateFlow를 활용한 UI 상태 관리

```kotlin
// 상태를 표현하는 데이터 클래스
data class UserListUiState(
    val isLoading: Boolean = false,
    val users: List<User> = emptyList(),
    val error: String? = null
)

class UserListViewModel(
    private val userRepository: UserRepository
) : ViewModel() {

    // 외부에는 읽기 전용 StateFlow만 노출
    private val _uiState = MutableStateFlow(UserListUiState())
    val uiState: StateFlow<UserListUiState> = _uiState.asStateFlow()

    init {
        loadUsers()
    }

    private fun loadUsers() {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true, error = null) }
            userRepository.getUsersFlow()
                .catch { e ->
                    _uiState.update {
                        it.copy(isLoading = false, error = e.message ?: "알 수 없는 오류")
                    }
                }
                .collect { users ->
                    _uiState.update {
                        it.copy(isLoading = false, users = users)
                    }
                }
        }
    }

    fun retry() = loadUsers()

    fun clearError() {
        _uiState.update { it.copy(error = null) }
    }
}

// Fragment에서 올바른 수집 방식
class UserListFragment : Fragment(R.layout.fragment_user_list) {
    private val viewModel: UserListViewModel by viewModels()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        // repeatOnLifecycle: 생명주기에 맞게 자동으로 collect/cancel
        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect { state ->
                    with(binding) {
                        progressBar.isVisible = state.isLoading
                        recyclerView.isVisible = !state.isLoading && state.error == null
                        errorView.isVisible = state.error != null
                        errorText.text = state.error
                        adapter.submitList(state.users)
                    }
                }
            }
        }

        binding.retryButton.setOnClickListener { viewModel.retry() }
    }
}
```

**핵심 포인트**: `repeatOnLifecycle(Lifecycle.State.STARTED)`를 사용하면 앱이 백그라운드로 전환될 때 자동으로 collect가 취소되고, 포그라운드로 복귀 시 재시작됩니다. `lifecycleScope.launch { uiState.collect { } }` 패턴은 백그라운드에서도 계속 동작하므로 **절대 사용하지 마세요**.

---

## 4. SharedFlow: 이벤트 브로드캐스팅

### 개념

`SharedFlow`는 여러 구독자에게 값을 브로드캐스팅하는 Hot Flow입니다. StateFlow와 달리 **초기값이 없으며**, `replay` 파라미터로 새 구독자가 받을 이전 이벤트 수를 세밀하게 제어할 수 있습니다.

| 파라미터 | 설명 | 기본값 |
|---------|------|--------|
| `replay` | 새 구독자에게 재전송할 이전 이벤트 수 | 0 |
| `extraBufferCapacity` | 구독자가 느릴 때 추가로 버퍼링할 이벤트 수 | 0 |
| `onBufferOverflow` | 버퍼가 가득 찼을 때의 처리 방식 | `SUSPEND` |

### 일회성 이벤트 처리 문제

UI 개발에서 "토스트 메시지 표시", "화면 이동", "다이얼로그 표시" 같은 **일회성 이벤트(One-shot Events)**를 StateFlow로 처리하면 화면 회전 후 이미 처리된 이벤트가 다시 실행되는 문제가 생깁니다. `SharedFlow(replay = 0)`는 새 구독자가 과거 이벤트를 받지 않으므로 이 문제를 해결합니다.

---

## 5. 실제 구현 예제 2: SharedFlow를 활용한 일회성 UI 이벤트 처리

```kotlin
// 이벤트 타입 정의
sealed class LoginUiEvent {
    data class ShowSnackbar(val message: String) : LoginUiEvent()
    data class NavigateTo(val destination: String) : LoginUiEvent()
    object ShowLoadingDialog : LoginUiEvent()
    object DismissLoadingDialog : LoginUiEvent()
}

data class LoginUiState(
    val email: String = "",
    val password: String = "",
    val isEmailValid: Boolean = true,
    val isPasswordValid: Boolean = true
)

class LoginViewModel(
    private val authRepository: AuthRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow(LoginUiState())
    val uiState: StateFlow<LoginUiState> = _uiState.asStateFlow()

    // replay = 0: 새 구독자는 과거 이벤트를 받지 않음
    // extraBufferCapacity = 1: 구독자가 일시적으로 없을 때도 이벤트 유실 방지
    private val _events = MutableSharedFlow<LoginUiEvent>(
        replay = 0,
        extraBufferCapacity = 1,
        onBufferOverflow = BufferOverflow.DROP_OLDEST
    )
    val events: SharedFlow<LoginUiEvent> = _events.asSharedFlow()

    fun onEmailChanged(email: String) {
        _uiState.update { it.copy(email = email, isEmailValid = true) }
    }

    fun onPasswordChanged(password: String) {
        _uiState.update { it.copy(password = password, isPasswordValid = true) }
    }

    fun login() {
        val state = _uiState.value
        val isEmailValid = state.email.isNotBlank() &&
            android.util.Patterns.EMAIL_ADDRESS.matcher(state.email).matches()
        val isPasswordValid = state.password.length >= 6

        if (!isEmailValid || !isPasswordValid) {
            _uiState.update { it.copy(isEmailValid = isEmailValid, isPasswordValid = isPasswordValid) }
            return
        }

        viewModelScope.launch {
            _events.emit(LoginUiEvent.ShowLoadingDialog)
            authRepository.login(state.email, state.password)
                .onSuccess {
                    _events.emit(LoginUiEvent.DismissLoadingDialog)
                    _events.emit(LoginUiEvent.NavigateTo("main"))
                }
                .onFailure { error ->
                    _events.emit(LoginUiEvent.DismissLoadingDialog)
                    _events.emit(LoginUiEvent.ShowSnackbar(error.message ?: "로그인에 실패했습니다."))
                }
        }
    }
}

// Fragment에서 StateFlow와 SharedFlow를 동시에 수집
class LoginFragment : Fragment(R.layout.fragment_login) {
    private val viewModel: LoginViewModel by viewModels()
    private var loadingDialog: AlertDialog? = null

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        setupViews()
        collectFlows()
    }

    private fun setupViews() {
        binding.loginButton.setOnClickListener { viewModel.login() }
        binding.emailInput.doAfterTextChanged { viewModel.onEmailChanged(it.toString()) }
        binding.passwordInput.doAfterTextChanged { viewModel.onPasswordChanged(it.toString()) }
    }

    private fun collectFlows() {
        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                launch {
                    viewModel.uiState.collect { state ->
                        binding.emailInputLayout.error =
                            if (!state.isEmailValid) "올바른 이메일을 입력하세요" else null
                        binding.passwordInputLayout.error =
                            if (!state.isPasswordValid) "비밀번호는 6자 이상이어야 합니다" else null
                    }
                }
                launch {
                    viewModel.events.collect { event ->
                        handleEvent(event)
                    }
                }
            }
        }
    }

    private fun handleEvent(event: LoginUiEvent) {
        when (event) {
            is LoginUiEvent.ShowSnackbar ->
                Snackbar.make(requireView(), event.message, Snackbar.LENGTH_LONG).show()
            is LoginUiEvent.NavigateTo ->
                findNavController().navigate(R.id.action_login_to_main)
            LoginUiEvent.ShowLoadingDialog -> {
                loadingDialog = AlertDialog.Builder(requireContext())
                    .setView(R.layout.dialog_loading)
                    .setCancelable(false)
                    .show()
            }
            LoginUiEvent.DismissLoadingDialog -> {
                loadingDialog?.dismiss()
                loadingDialog = null
            }
        }
    }
}
```

---

## 6. stateIn / shareIn 연산자: Cold Flow를 Hot Flow로 변환

Repository의 Cold Flow를 ViewModel에서 Hot Flow로 변환할 때 사용하는 핵심 연산자입니다.

```kotlin
class ProductViewModel(
    private val repository: ProductRepository
) : ViewModel() {

    // Cold Flow를 StateFlow로 변환
    val products: StateFlow<List<Product>> = repository.getProductsFlow()
        .stateIn(
            scope = viewModelScope,
            // 구독자가 없어진 후 5초 후 업스트림 취소 (화면 회전 대응)
            started = SharingStarted.WhileSubscribed(5_000),
            initialValue = emptyList()
        )

    // 여러 Flow를 결합하는 패턴
    val dashboardState: StateFlow<DashboardUiState> = combine(
        repository.getUserFlow(),
        repository.getSettingsFlow(),
        repository.getNotificationsFlow()
    ) { user, settings, notifications ->
        DashboardUiState(user = user, settings = settings, notifications = notifications)
    }.stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5_000),
        initialValue = DashboardUiState()
    )
}
```

`SharingStarted.WhileSubscribed(5_000)`은 Google이 공식 권장하는 설정입니다. 화면 회전 시 5초 이내 재구독되면 업스트림 Flow를 재시작하지 않아 불필요한 네트워크 요청을 방지합니다.

---

## 7. Channel vs SharedFlow 선택 기준

`Channel`은 전통적인 생산자-소비자 패턴을 Coroutines로 구현한 것으로, 각 이벤트가 **하나의 소비자에게만** 전달됩니다.

| 상황 | 권장 타입 |
|------|----------|
| UI 상태 (현재값 항상 필요) | `StateFlow` |
| UI 이벤트 (일회성, 여러 구독자 가능) | `SharedFlow(replay=0)` |
| 엄격한 1:1 작업 큐 (순서 보장 필요) | `Channel` |
| 콜백 기반 API를 Flow로 변환 | `callbackFlow` (내부적으로 Channel 사용) |

---

## 8. 주의사항 및 실무 팁

### 절대 피해야 할 패턴

```kotlin
// 잘못된 예: 백그라운드에서도 계속 실행됨 (메모리 낭비, 크래시 위험)
lifecycleScope.launch {
    viewModel.uiState.collect { ... }
}

// 올바른 예: 포그라운드에서만 실행
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.uiState.collect { ... }
    }
}
```

### update()를 항상 사용하라

```kotlin
// 비권장: 동시성 문제 발생 가능
_uiState.value = _uiState.value.copy(isLoading = true)

// 권장: 원자적 업데이트 보장
_uiState.update { it.copy(isLoading = true) }
```

### Turbine으로 Flow 테스트

```kotlin
@Test
fun `로그인 성공 시 NavigateTo 이벤트 발생`() = runTest {
    val viewModel = LoginViewModel(fakeSuccessRepository)

    viewModel.events.test {
        viewModel.login()
        assertEquals(LoginUiEvent.ShowLoadingDialog, awaitItem())
        assertEquals(LoginUiEvent.DismissLoadingDialog, awaitItem())
        val navEvent = awaitItem() as LoginUiEvent.NavigateTo
        assertEquals("main", navEvent.destination)
        cancelAndIgnoreRemainingEvents()
    }
}
```

---

## 결론

| 타입 | 특성 | 주요 용도 |
|------|------|----------|
| **StateFlow** | 항상 최신값 보유, replay=1 | UI 상태 관리, LiveData 대체 |
| **SharedFlow** | 멀티캐스트, replay 조절 가능 | 일회성 UI 이벤트, 이벤트 버스 |
| **Channel** | 1:1 파이프, 순서 보장 | 작업 큐, callbackFlow 내부 |

세 가지 타입의 특성을 명확히 이해하고 `repeatOnLifecycle`과 함께 올바르게 사용하면, 생명주기에 안전하고 메모리 누수 없는 반응형 Android 앱을 구축할 수 있습니다. `stateIn`/`shareIn` 연산자를 활용해 Repository의 Cold Flow를 ViewModel 경계에서 Hot Flow로 변환하는 패턴을 익혀두면 실무에서 바로 활용할 수 있습니다.

## 참고 자료
- [StateFlow and SharedFlow \| Android Developers](https://developer.android.com/kotlin/flow/stateflow-and-sharedflow)
- [Kotlin Flows on Android \| Android Developers](https://developer.android.com/kotlin/flow)
- [SharedFlow API Reference \| Kotlin](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-shared-flow/)
- [Kotlin Flow Patterns Every Senior Android Dev Must Know](https://dev.to/software_mvp-factory/kotlin-flow-patterns-every-senior-android-dev-must-know-228b)
