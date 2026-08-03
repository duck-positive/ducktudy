---
layout: post
title: "Android Jetpack Lifecycle 심화: repeatOnLifecycle·flowWithLifecycle·collectAsStateWithLifecycle 완전 정복"
date: 2026-08-03
categories: [android, kotlin]
tags: [android, lifecycle, coroutines, flow, repeatOnLifecycle, flowWithLifecycle, collectAsStateWithLifecycle, jetpack]
---

Android 앱을 개발하다 보면 반드시 마주치는 문제가 있습니다. 바로 **생명주기(Lifecycle)와 코루틴·Flow의 안전한 결합**입니다. 화면이 백그라운드로 전환되었을 때도 네트워크 요청이 계속 실행되거나, 이미 파괴된 View에 데이터를 업데이트하려다 크래시가 발생하는 상황은 Android 개발자라면 한 번쯤 경험해봤을 것입니다.

Jetpack의 `lifecycle-runtime-ktx` 라이브러리는 이 문제를 해결하기 위한 강력한 API를 제공합니다. 이 글에서는 `repeatOnLifecycle`, `flowWithLifecycle`, `collectAsStateWithLifecycle` 세 가지 API의 내부 동작 원리를 깊이 파헤치고, 언제 어떤 API를 선택해야 하는지 실제 코드와 함께 완전히 정복합니다.

---

## 1. 문제의 출발점: 왜 생명주기를 고려해야 하는가

### 1.1 기존 방식의 위험성

과거에는 `launchWhenStarted` / `launchWhenResumed` 같은 API를 많이 사용했습니다.

```kotlin
// ❌ 위험한 방식 - 지금은 Deprecated
lifecycleScope.launchWhenStarted {
    viewModel.uiState.collect { state ->
        updateUI(state)
    }
}
```

이 API는 생명주기가 `STARTED` 미만으로 내려가면 코루틴을 **일시 중단(suspend)**합니다. 겉보기에는 안전해 보이지만, 코루틴 자체는 살아 있으므로 **업스트림 Flow가 계속 실행**됩니다. 핫 Flow(StateFlow, SharedFlow)의 경우 백그라운드에서도 데이터 생산이 멈추지 않아 메모리 낭비, 배터리 소모, 불필요한 네트워크 요청이 발생합니다.

### 1.2 올바른 해법의 핵심 원칙

생명주기가 활성 상태(`STARTED` 이상)일 때만 코루틴을 **완전히 시작**하고, 비활성 상태(`STOPPED`)로 내려가면 코루틴을 **완전히 취소**해야 합니다. 이것이 `repeatOnLifecycle`의 탄생 배경입니다.

---

## 2. repeatOnLifecycle: 생명주기 연동 코루틴의 핵심

### 2.1 동작 원리

`repeatOnLifecycle`은 `LifecycleOwner`(Activity, Fragment)의 생명주기를 감지하여 다음을 수행합니다:

- 생명주기가 **지정한 상태 이상**이 되면 → 블록 내 코루틴을 **새로 시작**
- 생명주기가 **지정한 상태 미만**으로 내려가면 → 블록 내 코루틴을 **취소**
- 이 과정을 생명주기가 완전히 파괴될 때까지 **반복(repeat)**

이름 그대로 "생명주기에 따라 반복"합니다.

```kotlin
// ✅ 올바른 방식
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        // 이 블록은 STARTED 이상일 때만 실행되고,
        // STOPPED로 내려가면 취소되었다가 다시 STARTED가 되면 재시작됩니다.
        viewModel.uiState.collect { state ->
            updateUI(state)
        }
    }
}
```

### 2.2 생명주기 상태 선택 가이드

| 상태 | Activity | Fragment | 적합한 상황 |
|---|---|---|---|
| `CREATED` | `onCreate` ~ `onDestroy` | `onAttach` ~ `onDetach` | 거의 사용 안 함 |
| `STARTED` | `onStart` ~ `onStop` | `onStart` ~ `onStop` | **가장 권장** - 화면이 보일 때만 수집 |
| `RESUMED` | `onResume` ~ `onPause` | `onResume` ~ `onPause` | 오디오 포커스처럼 포그라운드 완전 활성 시만 필요한 경우 |

일반적인 UI 업데이트에는 `Lifecycle.State.STARTED`를 사용하는 것이 표준입니다. `RESUMED`는 너무 좁고, `CREATED`는 너무 넓습니다.

### 2.3 실전 예제: MVVM + UiState 패턴

다음은 실제 앱에서 `repeatOnLifecycle`을 활용한 완전한 예제입니다.

```kotlin
// ViewModel
data class UiState(
    val isLoading: Boolean = false,
    val items: List<Item> = emptyList(),
    val error: String? = null
)

class ItemListViewModel(
    private val repository: ItemRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow(UiState(isLoading = true))
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()

    init {
        loadItems()
    }

    private fun loadItems() {
        viewModelScope.launch {
            repository.getItemsFlow()
                .catch { e ->
                    _uiState.update { it.copy(isLoading = false, error = e.message) }
                }
                .collect { items ->
                    _uiState.update {
                        it.copy(isLoading = false, items = items)
                    }
                }
        }
    }
}

// Activity (View 레이어)
class ItemListActivity : AppCompatActivity() {

    private val viewModel: ItemListViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_item_list)

        lifecycleScope.launch {
            // Activity가 살아있는 동안 반복적으로 Flow를 수집
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect { state ->
                    when {
                        state.isLoading -> showLoading()
                        state.error != null -> showError(state.error)
                        else -> showItems(state.items)
                    }
                }
            }
        }
        // repeatOnLifecycle 이후 코드는 Lifecycle이 DESTROYED 된 후에 실행됩니다.
    }
}
```

### 2.4 여러 Flow를 동시에 수집하는 패턴

```kotlin
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        // launch { } 블록을 통해 여러 Flow를 병렬로 수집
        launch {
            viewModel.uiState.collect { render(it) }
        }
        launch {
            viewModel.sideEffect.collect { handleEffect(it) }
        }
        launch {
            viewModel.networkStatus.collect { updateNetworkBanner(it) }
        }
    }
}
```

이 패턴에서 `repeatOnLifecycle` 블록 안의 모든 자식 코루틴은 생명주기가 내려가면 **함께 취소**되고, 올라오면 **함께 재시작**됩니다.

---

## 3. flowWithLifecycle: 단일 Flow를 위한 간결한 연산자

### 3.1 개념과 내부 구조

`flowWithLifecycle`은 `repeatOnLifecycle`을 기반으로 만들어진 **Flow 연산자**입니다. 단 하나의 Flow에만 생명주기 인식 수집을 적용할 때 코드를 간결하게 작성할 수 있습니다.

```kotlin
// repeatOnLifecycle 방식
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.uiState.collect { render(it) }
    }
}

// flowWithLifecycle 방식 (동일한 결과)
viewModel.uiState
    .flowWithLifecycle(lifecycle, Lifecycle.State.STARTED)
    .onEach { render(it) }
    .launchIn(lifecycleScope)
```

두 방식은 기능적으로 동등합니다. 다만 `flowWithLifecycle`은 **단일 Flow**에 적합하고, 여러 Flow를 동시에 다뤄야 할 때는 `repeatOnLifecycle`이 더 명확합니다.

### 3.2 연산자 순서의 중요성

`flowWithLifecycle`을 사용할 때 **연산자 체인의 순서**가 매우 중요합니다.

```kotlin
// ✅ 올바른 순서: flowWithLifecycle 이전에 map 적용
viewModel.rawDataFlow
    .map { transformData(it) }          // 업스트림 - 생명주기가 비활성이면 취소됨
    .flowWithLifecycle(lifecycle)        // ← 이 지점이 경계
    .onEach { updateUI(it) }            // 다운스트림 - 생명주기 활성 시에만 실행
    .launchIn(lifecycleScope)

// ⚠️ 주의: flowWithLifecycle 이후의 연산자는 생명주기와 무관하게 실행될 수 있음
viewModel.rawDataFlow
    .flowWithLifecycle(lifecycle)
    .map { heavyTransform(it) }         // 데이터가 오지 않으면 실행 안 되지만 의도 불명확
    .onEach { updateUI(it) }
    .launchIn(lifecycleScope)
```

`flowWithLifecycle` **이전** 연산자들은 생명주기가 비활성이 되면 함께 취소됩니다. **이후** 연산자들은 데이터가 흘러오지 않으면 실행되지 않지만, 구독 자체는 유지됩니다. 일반적으로 변환 로직은 `flowWithLifecycle` **이전**에 두는 것이 의도가 명확합니다.

### 3.3 Fragment에서의 올바른 사용법

Fragment에서 특별히 주의해야 할 점이 있습니다. `viewLifecycleOwner`를 사용해야 합니다.

```kotlin
class ItemFragment : Fragment() {

    private val viewModel: ItemViewModel by viewModels()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        // ❌ 위험: Fragment의 lifecycle은 View 파괴 후에도 살아있음
        // viewModel.uiState
        //     .flowWithLifecycle(lifecycle)  // Fragment lifecycle 사용하면 안 됨!
        //     .launchIn(lifecycleScope)

        // ✅ 올바른 방법: viewLifecycleOwner 사용
        viewModel.uiState
            .flowWithLifecycle(viewLifecycleOwner.lifecycle, Lifecycle.State.STARTED)
            .onEach { state -> updateUI(state) }
            .launchIn(viewLifecycleOwner.lifecycleScope)
    }
}
```

Fragment는 `lifecycle`(Fragment 자체의 생명주기)과 `viewLifecycleOwner.lifecycle`(View의 생명주기) 두 가지가 있습니다. UI 업데이트는 반드시 `viewLifecycleOwner`를 사용해야 Back Stack에 있을 때 View가 파괴되어도 안전합니다.

---

## 4. collectAsStateWithLifecycle: Jetpack Compose에서의 생명주기 인식 수집

### 4.1 Compose에서 생명주기 인식이 필요한 이유

Jetpack Compose에서 Flow를 State로 변환할 때 기본적으로 `collectAsState()`를 사용합니다. 그런데 이 API는 **Composition의 생명주기**에만 묶여 있어서, 앱이 백그라운드로 가도 Flow 수집이 계속됩니다.

```kotlin
// ❌ 백그라운드에서도 수집 계속됨
val uiState by viewModel.uiState.collectAsState()
```

`collectAsStateWithLifecycle()`은 `repeatOnLifecycle(Lifecycle.State.STARTED)`를 내부적으로 사용하여 앱이 백그라운드에 있을 때 수집을 자동으로 중지합니다.

### 4.2 의존성 추가

```kotlin
// build.gradle.kts (app 모듈)
dependencies {
    implementation("androidx.lifecycle:lifecycle-runtime-compose:2.8.0")
}
```

### 4.3 실전 예제: Compose + ViewModel + collectAsStateWithLifecycle

```kotlin
// ViewModel
class WeatherViewModel(
    private val weatherRepository: WeatherRepository
) : ViewModel() {

    private val _city = MutableStateFlow("Seoul")

    val weatherState: StateFlow<WeatherUiState> = _city
        .flatMapLatest { city ->
            weatherRepository.getWeatherStream(city)
                .map { weather -> WeatherUiState.Success(weather) }
                .catch { e -> emit(WeatherUiState.Error(e.message ?: "Unknown error")) }
        }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            initialValue = WeatherUiState.Loading
        )

    fun changeCity(city: String) {
        _city.value = city
    }
}

sealed interface WeatherUiState {
    data object Loading : WeatherUiState
    data class Success(val weather: Weather) : WeatherUiState
    data class Error(val message: String) : WeatherUiState
}

// Composable
@Composable
fun WeatherScreen(
    viewModel: WeatherViewModel = viewModel(),
    modifier: Modifier = Modifier
) {
    // ✅ Lifecycle.State.STARTED 상태일 때만 수집
    val uiState by viewModel.weatherState.collectAsStateWithLifecycle()

    WeatherContent(
        uiState = uiState,
        onCityChange = viewModel::changeCity,
        modifier = modifier
    )
}

@Composable
private fun WeatherContent(
    uiState: WeatherUiState,
    onCityChange: (String) -> Unit,
    modifier: Modifier = Modifier
) {
    Column(modifier = modifier.fillMaxSize()) {
        when (uiState) {
            is WeatherUiState.Loading -> {
                CircularProgressIndicator(modifier = Modifier.align(Alignment.CenterHorizontally))
            }
            is WeatherUiState.Success -> {
                Text(text = "온도: ${uiState.weather.temperature}°C")
                Text(text = "날씨: ${uiState.weather.condition}")
            }
            is WeatherUiState.Error -> {
                Text(
                    text = "오류: ${uiState.message}",
                    color = MaterialTheme.colorScheme.error
                )
            }
        }
    }
}
```

### 4.4 minActiveState 커스터마이징

기본값은 `Lifecycle.State.STARTED`이지만 필요에 따라 변경할 수 있습니다.

```kotlin
// 포그라운드 완전 활성 시에만 수집 (더 엄격한 제어)
val uiState by viewModel.uiState.collectAsStateWithLifecycle(
    minActiveState = Lifecycle.State.RESUMED
)

// 거의 사용하지 않지만 CREATED부터 수집
val uiState by viewModel.uiState.collectAsStateWithLifecycle(
    minActiveState = Lifecycle.State.CREATED
)
```

---

## 5. SharingStarted.WhileSubscribed와의 시너지

`collectAsStateWithLifecycle`은 `StateFlow`를 `stateIn`으로 변환할 때 `SharingStarted.WhileSubscribed(stopTimeoutMillis)`와 함께 사용하면 강력한 시너지를 냅니다.

```kotlin
val uiState: StateFlow<UiState> = repository.dataFlow
    .stateIn(
        scope = viewModelScope,
        // 구독자가 없어진 후 5초 뒤에 업스트림 Flow 취소
        started = SharingStarted.WhileSubscribed(5_000),
        initialValue = UiState.Loading
    )
```

이 조합의 동작 흐름:

1. 앱이 포그라운드 → `collectAsStateWithLifecycle`이 구독 시작
2. ViewModel의 `StateFlow` 구독자 수 > 0 → `SharingStarted.WhileSubscribed`에 의해 업스트림 Flow 활성화
3. 앱이 백그라운드로 전환 → `collectAsStateWithLifecycle`이 구독 해제
4. 구독자 수 = 0, 5초 타이머 시작
5. 5초 이내 앱이 복귀하면 → 타이머 취소, 기존 Flow 유지 (화면 회전 대응)
6. 5초 초과 시 → 업스트림 Flow 취소 (배터리·네트워크 절약)

이 전략은 화면 회전 시 불필요한 재시작을 방지하면서도 장시간 백그라운드에서는 리소스를 절약합니다.

---

## 6. 주의사항 및 실전 팁

### 6.1 콜드 Flow vs 핫 Flow에 따른 전략

| Flow 종류 | 특징 | 권장 수집 방식 |
|---|---|---|
| 콜드 Flow (일반 `flow { }`) | 구독 시 새로 시작 | `repeatOnLifecycle` (매번 재시작 OK) |
| `StateFlow` | 항상 최신값 보유 | `collectAsStateWithLifecycle` |
| `SharedFlow` | 히스토리 없음, 이벤트성 | `repeatOnLifecycle` (이벤트 손실 주의) |

### 6.2 일회성 이벤트(Side Effect) 처리

`SharedFlow`로 일회성 이벤트를 처리할 때 `repeatOnLifecycle`을 사용하면 백그라운드 전환 중 이벤트가 손실될 수 있습니다. 이 경우에는 `Channel`을 사용하거나, 이벤트를 UiState에 포함시키는 방식을 고려하세요.

```kotlin
// ✅ Channel 방식 (백버퍼로 손실 방지)
private val _sideEffect = Channel<SideEffect>(Channel.BUFFERED)
val sideEffect = _sideEffect.receiveAsFlow()

// ViewModel
fun performAction() {
    viewModelScope.launch {
        // 비즈니스 로직 수행
        _sideEffect.send(SideEffect.ShowToast("완료!"))
    }
}

// Activity/Fragment에서 수집
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.sideEffect.collect { effect ->
            when (effect) {
                is SideEffect.ShowToast -> Toast.makeText(this@Activity, effect.message, Toast.LENGTH_SHORT).show()
                is SideEffect.Navigate -> navigateTo(effect.destination)
            }
        }
    }
}
```

### 6.3 테스트에서의 활용

```kotlin
@Test
fun `uiState is collected only when STARTED`() = runTest {
    val viewModel = ItemListViewModel(fakeRepository)

    // TestLifecycleOwner로 생명주기 시뮬레이션
    val lifecycleOwner = TestLifecycleOwner(initialState = Lifecycle.State.RESUMED)
    val collectedStates = mutableListOf<UiState>()

    val job = launch {
        viewModel.uiState
            .flowWithLifecycle(lifecycleOwner.lifecycle, Lifecycle.State.STARTED)
            .toList(collectedStates)
    }

    // STOPPED로 전환 - 수집 중단
    lifecycleOwner.currentState = Lifecycle.State.CREATED
    advanceTimeBy(1000)

    // 다시 STARTED로 복귀 - 수집 재개
    lifecycleOwner.currentState = Lifecycle.State.STARTED

    job.cancel()
    // collectedStates 검증
}
```

### 6.4 체크리스트

| 확인 항목 | 설명 |
|---|---|
| `launchWhenStarted` 사용 제거 | Deprecated, `repeatOnLifecycle`로 전환 |
| Fragment에서 `viewLifecycleOwner` 사용 | `lifecycle` 대신 `viewLifecycleOwner.lifecycle` |
| Compose에서 `collectAsState` → `collectAsStateWithLifecycle` | 백그라운드 리소스 낭비 방지 |
| `SharingStarted.WhileSubscribed(5_000)` 설정 | 화면 회전 대응 + 백그라운드 절약 |
| 여러 Flow 동시 수집은 `repeatOnLifecycle` 선호 | 코드 명확성 유지 |

---

## 7. API 선택 가이드 요약

```
수집 대상이 하나인가?
├─ 예 → flowWithLifecycle() 또는 collectAsStateWithLifecycle() (Compose)
└─ 아니오 (여러 Flow) → repeatOnLifecycle { launch { } ... }

Compose인가?
├─ 예 → collectAsStateWithLifecycle()
└─ 아니오 (View 시스템) → repeatOnLifecycle 또는 flowWithLifecycle
```

`repeatOnLifecycle`은 가장 강력하고 범용적인 API입니다. `flowWithLifecycle`과 `collectAsStateWithLifecycle`은 이를 기반으로 만들어진 편의 API입니다. 세 API 모두 **백그라운드에서 완전히 코루틴을 취소**한다는 핵심 원칙을 공유합니다.

---

생명주기와 코루틴의 올바른 결합은 단순한 코드 스타일의 문제가 아닙니다. 메모리 누수, 불필요한 배터리 소모, 예기치 않은 크래시를 방지하는 **안정적인 앱의 기반**입니다. `repeatOnLifecycle`을 중심으로 한 이 패턴을 프로젝트 전반에 적용하면 더 견고하고 효율적인 Android 앱을 만들 수 있습니다.

## 참고 자료
- [Use Kotlin coroutines with lifecycle-aware components | Android Developers](https://developer.android.com/topic/libraries/architecture/coroutines)
- [Kotlin Coroutines Guide | Kotlin Documentation](https://kotlinlang.org/docs/coroutines-guide.html)
