---
layout: post
title: "Jetpack Compose 고급 패턴 — CompositionLocal, SideEffect API, Snapshot 완전 정복"
date: 2026-06-12
categories: [android, flutter]
tags: [android, jetpack-compose, compositionlocal, sideeffect, snapshot, kotlin, advanced]
---

Jetpack Compose는 선언형 UI 패러다임을 Android에 도입하면서 상태 관리와 부수 효과(Side Effect) 처리 방식도 완전히 새롭게 정의했습니다. 이 글에서는 실무에서 자주 마주치지만 제대로 이해하기 어려운 세 가지 고급 주제인 **CompositionLocal**, **SideEffect API 패밀리**, **Snapshot 시스템**을 심층적으로 다룹니다. 단순히 API 사용법에 그치지 않고, 각각의 동작 원리와 설계 의도까지 이해함으로써 더 견고하고 유지보수하기 쉬운 Compose 코드를 작성할 수 있습니다.

## 1. CompositionLocal — 암묵적 의존성 주입

### 개념 설명

일반적으로 Compose에서 데이터를 하위 컴포저블에 전달하려면 파라미터를 통한 명시적 전달(prop drilling)을 사용합니다. 그러나 테마, 로케일, 분석 트래커처럼 **트리 전체에서 공통으로 필요하지만 매 컴포저블마다 명시적으로 전달하기 번거로운 데이터**에는 `CompositionLocal`이 효과적인 해결책입니다.

`CompositionLocal`은 Compose 트리에서 상위 노드가 값을 제공(provide)하면, 하위 노드 어디서나 해당 값을 읽을 수 있는 암묵적 의존성 주입 메커니즘입니다. Android의 `MaterialTheme.colorScheme`이나 `LocalContext.current` 모두 내부적으로 이 메커니즘을 사용합니다.

### 왜 필요한가

- **prop drilling 제거**: 5단계 깊이의 컴포저블에 테마 색상을 전달하기 위해 중간 컴포저블 모두가 해당 파라미터를 받아야 하는 불필요한 결합도를 없앱니다.
- **스코프 지정 가능**: `CompositionLocalProvider`로 감싼 범위 내에서만 값을 오버라이드할 수 있어, 특정 서브트리에서 다른 테마나 설정을 적용하기 쉽습니다.
- **테스트 용이성**: 테스트 환경에서 `CompositionLocalProvider`로 의존성을 쉽게 교체할 수 있습니다.

### compositionLocalOf vs staticCompositionLocalOf

두 팩토리 함수의 차이는 **리컴포지션 범위**에 있습니다.

- `compositionLocalOf`: 값 변경 시 해당 값을 **실제로 읽는** 컴포저블만 리컴포지션됩니다. 값이 자주 변경될 때 적합합니다.
- `staticCompositionLocalOf`: 읽기가 추적되지 않아 값 변경 시 `CompositionLocalProvider`로 감싼 **전체 콘텐츠 람다**가 리컴포지션됩니다. 값이 거의 변경되지 않는 경우(예: 의존성 주입 컨테이너) 성능상 유리합니다.

### 실제 구현 예제

```kotlin
// 1) CompositionLocal 정의
data class AppAnalytics(
    val trackScreen: (String) -> Unit,
    val trackEvent: (String, Map<String, Any>) -> Unit,
)

// 값이 자주 바뀌지 않으므로 staticCompositionLocalOf 사용
val LocalAnalytics = staticCompositionLocalOf<AppAnalytics> {
    error("AppAnalytics가 CompositionLocalProvider로 제공되지 않았습니다")
}

// 2) 앱 최상단에서 제공
@Composable
fun MyApp() {
    val analytics = remember {
        AppAnalytics(
            trackScreen = { name -> FirebaseAnalytics.logScreen(name) },
            trackEvent = { name, params -> FirebaseAnalytics.logEvent(name, params) },
        )
    }

    CompositionLocalProvider(LocalAnalytics provides analytics) {
        AppNavHost()
    }
}

// 3) 하위 컴포저블에서 소비
@Composable
fun ProductDetailScreen(productId: String) {
    val analytics = LocalAnalytics.current

    // 화면 진입 시 트래킹 (LaunchedEffect로 한 번만 실행)
    LaunchedEffect(productId) {
        analytics.trackScreen("ProductDetail")
        analytics.trackEvent("view_item", mapOf("item_id" to productId))
    }

    // UI 구성 ...
}

// 4) 서브트리에서 오버라이드 (테스트용 no-op Analytics 주입)
@Composable
fun PreviewWrapper(content: @Composable () -> Unit) {
    val noOpAnalytics = AppAnalytics(
        trackScreen = {},
        trackEvent = { _, _ -> },
    )
    CompositionLocalProvider(LocalAnalytics provides noOpAnalytics) {
        content()
    }
}
```

---

## 2. SideEffect API 패밀리 — 올바른 부수 효과 처리

### 개념 설명

Compose의 컴포저블 함수는 언제든 재실행될 수 있으며, 실행 순서도 보장되지 않습니다. 따라서 **컴포저블 본문에서 직접 부수 효과를 발생시키는 것은 안전하지 않습니다.** Compose는 이를 위해 생명주기와 결합된 SideEffect API 패밀리를 제공합니다.

각 API의 목적을 정확히 이해하는 것이 핵심입니다:

| API | 실행 시점 | 정리(cleanup) | 주요 용도 |
|---|---|---|---|
| `SideEffect` | 모든 성공적인 리컴포지션 후 | 없음 | Compose 외부 객체에 최신 상태 동기화 |
| `LaunchedEffect` | 컴포지션 진입 + 키 변경 시 | 코루틴 자동 취소 | 비동기 작업 (API 호출, 딜레이 등) |
| `DisposableEffect` | 컴포지션 진입 + 키 변경 시 | `onDispose` 블록 | 이벤트 리스너 등록/해제, 리소스 관리 |
| `rememberUpdatedState` | — | — | 오래 실행되는 effect 내에서 최신 람다 참조 |
| `produceState` | 컴포지션 진입 시 | 코루틴 자동 취소 | 외부 데이터 소스를 State로 변환 |
| `snapshotFlow` | — | — | Compose State를 Flow로 변환 |

### 왜 필요한가

부수 효과를 컴포저블 본문에서 직접 실행하면 리컴포지션마다 반복 실행되어 **무한 루프, 메모리 누수, 불필요한 네트워크 호출** 등의 문제가 발생합니다. SideEffect API는 이를 컴포지션 생명주기와 정확히 동기화하여 안전하게 실행합니다.

### 실제 구현 예제

다음은 실제 앱에서 흔히 필요한 세 가지 시나리오를 하나의 화면으로 구성한 예제입니다.

```kotlin
@Composable
fun ChatScreen(
    roomId: String,
    viewModel: ChatViewModel = hiltViewModel(),
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    val listState = rememberLazyListState()
    val scope = rememberCoroutineScope()

    // ── 시나리오 1: LaunchedEffect ──────────────────────────────────────
    // roomId가 변경될 때마다 채팅방 구독을 재시작합니다.
    // 이전 코루틴은 자동으로 취소됩니다.
    LaunchedEffect(roomId) {
        viewModel.subscribeToRoom(roomId)
    }

    // ── 시나리오 2: rememberUpdatedState + LaunchedEffect ───────────────
    // 타임아웃 핸들러처럼 오래 실행되는 effect 안에서 최신 람다를 안전하게 참조합니다.
    // onTimeout 람다는 바뀔 수 있지만, effect를 재시작하고 싶지 않을 때 사용합니다.
    val currentOnTimeout by rememberUpdatedState(viewModel::onSessionTimeout)
    LaunchedEffect(true) {
        delay(30 * 60 * 1000L) // 30분
        currentOnTimeout()
    }

    // ── 시나리오 3: DisposableEffect ────────────────────────────────────
    // 시스템 백 버튼 인터셉트 — 화면이 사라지면 반드시 해제해야 합니다.
    val backDispatcher = LocalOnBackPressedDispatcherOwner.current?.onBackPressedDispatcher
    DisposableEffect(backDispatcher) {
        val callback = object : OnBackPressedCallback(true) {
            override fun handleOnBackPressed() {
                viewModel.handleBackPress()
            }
        }
        backDispatcher?.addCallback(callback)
        onDispose {
            callback.remove() // 컴포지션 종료 시 반드시 정리
        }
    }

    // ── 시나리오 4: SideEffect ──────────────────────────────────────────
    // 매 리컴포지션 후 Compose 외부의 분석 SDK에 최신 상태를 동기화합니다.
    SideEffect {
        FirebasePerformance.getInstance()
            .getTrace("chat_screen")
            .putAttribute("message_count", uiState.messages.size.toString())
    }

    // ── 시나리오 5: snapshotFlow ────────────────────────────────────────
    // listState의 스크롤 위치 변화를 Flow로 관찰하여 "맨 아래로" 버튼을 제어합니다.
    val showScrollToBottom = remember { mutableStateOf(false) }
    LaunchedEffect(listState) {
        snapshotFlow { listState.firstVisibleItemIndex }
            .distinctUntilChanged()
            .collect { index ->
                showScrollToBottom.value = index > 5
            }
    }

    // UI
    Box(modifier = Modifier.fillMaxSize()) {
        LazyColumn(state = listState) {
            items(uiState.messages, key = { it.id }) { message ->
                ChatMessageItem(message)
            }
        }
        if (showScrollToBottom.value) {
            FloatingActionButton(
                onClick = {
                    scope.launch {
                        listState.animateScrollToItem(uiState.messages.lastIndex)
                    }
                },
                modifier = Modifier.align(Alignment.BottomEnd).padding(16.dp)
            ) {
                Icon(Icons.Default.ArrowDownward, contentDescription = "맨 아래로")
            }
        }
    }
}
```

---

## 3. Snapshot 시스템 — Compose 반응성의 심장

### 개념 설명

Compose의 상태 관리가 마법처럼 동작하는 이유는 **Snapshot 시스템** 덕분입니다. `mutableStateOf`, `mutableStateListOf` 등 Compose 상태 객체는 모두 Snapshot 시스템 위에 구축되어 있습니다.

Snapshot 시스템의 핵심 개념:

- **Snapshot**: 특정 시점의 모든 상태(State) 값의 일관된 읽기 뷰입니다. 마치 데이터베이스의 트랜잭션 격리 수준과 유사합니다.
- **읽기 추적(Read Tracking)**: 컴포저블이 실행되는 동안 어떤 State 객체가 읽혔는지 자동으로 기록합니다.
- **변경 알림**: State 값이 변경되면 해당 State를 읽었던 컴포저블만 정확히 리컴포지션 대상으로 표시합니다.

이 시스템 덕분에 Compose는 전체 UI 트리를 재렌더링하지 않고 **변경된 State를 의존하는 컴포저블만** 정밀하게 리컴포지션할 수 있습니다.

### snapshotFlow 심화 활용

`snapshotFlow`는 Snapshot 시스템을 Flow와 연결하는 브릿지입니다. 블록 내에서 읽힌 State 객체가 변경될 때마다 새 값을 emit합니다. `distinctUntilChanged`가 기본 적용되어 동일한 값은 중복 emit하지 않습니다.

```kotlin
// ViewModel에서 Compose State를 Flow로 변환하여 비즈니스 로직 처리
class SearchViewModel : ViewModel() {

    var searchQuery by mutableStateOf("")
        private set

    // snapshotFlow로 쿼리 변화를 감지하여 디바운스 검색 수행
    val searchResults: StateFlow<List<SearchResult>> = snapshotFlow { searchQuery }
        .debounce(300)
        .filter { it.length >= 2 }
        .mapLatest { query ->
            searchRepository.search(query)
        }
        .catch { emit(emptyList()) }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            initialValue = emptyList(),
        )

    fun onQueryChange(query: String) {
        searchQuery = query
    }
}

// Composable에서 사용
@Composable
fun SearchScreen(viewModel: SearchViewModel = hiltViewModel()) {
    val results by viewModel.searchResults.collectAsStateWithLifecycle()

    Column {
        TextField(
            value = viewModel.searchQuery,
            onValueChange = viewModel::onQueryChange,
            placeholder = { Text("검색어를 입력하세요") },
        )
        LazyColumn {
            items(results) { result ->
                SearchResultItem(result)
            }
        }
    }
}
```

---

## 주의사항 및 실무 팁

### CompositionLocal 사용 시 주의

1. **남용 금지**: 단순히 prop drilling이 귀찮다는 이유만으로 CompositionLocal을 사용하면 코드의 추론이 어려워집니다. 테마, DI 컨테이너, 시스템 서비스처럼 **진짜 전역적인 관심사**에만 사용하세요.
2. **기본값 제공**: `compositionLocalOf`의 기본값으로 `error()`를 사용하면 Provider 없이 접근 시 명확한 오류 메시지를 얻을 수 있습니다.
3. **staticCompositionLocalOf 선택 기준**: 값이 앱 생명주기 동안 한 번만 설정되는 경우(예: 의존성 그래프, 시스템 서비스)에는 `staticCompositionLocalOf`를 사용하여 불필요한 리컴포지션 추적 오버헤드를 제거하세요.

### SideEffect API 선택 가이드

- **"매 리컴포지션 후 외부에 알려야 한다"** → `SideEffect`
- **"비동기 작업을 해야 한다"** → `LaunchedEffect`
- **"등록한 것을 반드시 해제해야 한다"** → `DisposableEffect`
- **"오래 실행되는 effect 안에서 최신 람다를 참조해야 한다"** → `rememberUpdatedState`
- **"State 변화를 Flow처럼 처리해야 한다"** → `snapshotFlow`
- **"`LaunchedEffect(true)` 사용은 재고하라**: 키로 `true`나 `Unit`을 사용하면 컴포지션 진입 시 딱 한 번만 실행됩니다. 이것이 의도라면 괜찮지만, 특정 데이터의 변화를 추적하려는 것이었다면 올바른 키를 사용해야 합니다.

### Snapshot 최적화

- **`derivedStateOf` 활용**: 여러 State를 조합한 계산 결과가 있고, 중간 State가 자주 변경되더라도 최종 결과가 같다면 `derivedStateOf`로 감싸 리컴포지션을 줄이세요.
- **`remember { snapshotFlow { ... } }` 패턴 주의**: `snapshotFlow`로 생성된 Flow는 컴포지션 외부에서 수집하면 Snapshot 컨텍스트가 없어 동작하지 않을 수 있습니다. `LaunchedEffect` 내부에서 수집하는 것이 안전합니다.

---

Jetpack Compose의 `CompositionLocal`, SideEffect API, Snapshot 시스템은 서로 긴밀하게 연결되어 있습니다. `CompositionLocal`은 의존성을 트리에 암묵적으로 전달하고, SideEffect API는 컴포지션 생명주기에 맞춰 부수 효과를 안전하게 실행하며, Snapshot 시스템은 이 모든 것의 반응성 기반을 제공합니다. 세 가지 개념을 함께 이해하면 Compose 앱의 동작 원리를 한층 깊이 파악할 수 있고, 성능 문제와 버그를 훨씬 효과적으로 진단할 수 있습니다.

## 참고 자료
- [Side-effects in Compose — Android Developers](https://developer.android.com/develop/ui/compose/side-effects)
- [Locally scoped data with CompositionLocal — Android Developers](https://developer.android.com/develop/ui/compose/compositionlocal)
- [Advanced State and Side Effects in Jetpack Compose (Codelab)](https://developer.android.com/codelabs/jetpack-compose-advanced-state-side-effects)
- [androidx.compose.runtime.snapshots API Reference](https://developer.android.com/reference/kotlin/androidx/compose/runtime/snapshots/package-summary)
