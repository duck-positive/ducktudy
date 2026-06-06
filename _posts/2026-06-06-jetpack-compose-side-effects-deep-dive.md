---
layout: post
title: "Jetpack Compose SideEffect 완전 정복: LaunchedEffect부터 snapshotFlow까지"
date: 2026-06-06
categories: [android, compose]
tags: [jetpack-compose, android, kotlin, side-effects, launchedeffect, disposableeffect, producestate, snapshotflow, coroutines]
---

## 개요

Jetpack Compose는 UI를 선언적으로 기술하는 패러다임으로, 컴포저블 함수는 **순수 함수(pure function)** 에 가깝게 작성되어야 합니다. 같은 입력에 같은 출력을 내고, 외부 상태를 직접 건드리지 않는 것이 원칙입니다. 그런데 현실의 앱은 네트워크 요청, 애니메이션 타이머, 생명주기 리스너 등록처럼 "외부 세계와 상호작용하는 작업" 이 반드시 필요합니다. 이를 **부수 효과(Side Effect)** 라고 부릅니다.

Compose 팀은 이런 부수 효과를 안전하게 처리하기 위해 여러 Effect API를 제공합니다. 이 글에서는 `LaunchedEffect`, `rememberUpdatedState`, `DisposableEffect`, `SideEffect`, `produceState`, `derivedStateOf`, `snapshotFlow` 각각의 개념과 올바른 사용 시나리오, 그리고 실무에서 자주 저지르는 실수를 심층적으로 다룹니다.

---

## 1. 왜 Side Effect API가 필요한가?

Compose의 리컴포지션(recomposition)은 언제든, 얼마든지 반복될 수 있습니다. 만약 컴포저블 함수 본문에 직접 코루틴 실행이나 리스너 등록을 넣는다면 리컴포지션이 발생할 때마다 중복 실행되거나 메모리 누수가 발생합니다.

```kotlin
// ❌ 잘못된 패턴 — 리컴포지션 때마다 네트워크 요청 발생
@Composable
fun BadExample(userId: String) {
    // 컴포저블 본문에서 직접 실행하면 안 됨
    viewModel.loadUser(userId)  // 리컴포지션마다 호출!
    Text("Loading...")
}
```

Effect API는 컴포지션 생명주기와 동기화된 안전한 실행 환경을 제공합니다. 핵심 원칙은 다음 세 가지입니다.

1. **Effect는 컴포지션에 진입할 때 시작된다**
2. **Key가 변경되면 이전 Effect를 취소하고 새 Effect를 시작한다**
3. **컴포지션에서 빠져나올 때 자동으로 정리된다**

---

## 2. LaunchedEffect — 코루틴 기반 부수 효과

`LaunchedEffect`는 suspend 함수를 컴포저블 내에서 안전하게 실행할 때 사용합니다. 지정한 **key**가 변경될 때마다 이전 코루틴을 취소하고 새 코루틴을 시작합니다.

```kotlin
@Composable
fun UserProfileScreen(userId: String, viewModel: UserViewModel) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    // userId가 바뀔 때마다 기존 요청을 취소하고 새로 시작
    LaunchedEffect(userId) {
        viewModel.loadUser(userId)
    }

    when (val state = uiState) {
        is UiState.Loading -> CircularProgressIndicator()
        is UiState.Success -> UserContent(user = state.user)
        is UiState.Error   -> ErrorMessage(message = state.message)
    }
}
```

### rememberUpdatedState와 함께 쓰는 패턴

`LaunchedEffect`에서 콜백 람다를 사용할 때 흔한 문제가 있습니다. key를 `Unit`으로 고정하면 람다가 바뀌어도 Effect가 재시작되지 않아 항상 초기 람다를 캡처합니다.

```kotlin
@Composable
fun SplashScreen(onTimeout: () -> Unit) {
    // onTimeout이 바뀌어도 최신 버전을 참조하도록 보장
    val currentOnTimeout by rememberUpdatedState(onTimeout)

    LaunchedEffect(Unit) {          // key = Unit → 최초 1회만 실행
        delay(3_000L)
        currentOnTimeout()          // 항상 최신 람다를 호출
    }

    SplashContent()
}
```

`rememberUpdatedState`는 항상 최신 값에 대한 참조를 반환하면서도, Effect를 재시작하지 않습니다. 긴 지연 작업 중 콜백이 바뀔 수 있는 시나리오에서 필수적인 패턴입니다.

---

## 3. DisposableEffect — 정리(cleanup)가 필요한 부수 효과

외부 리스너 등록처럼 반드시 해제 코드가 필요한 경우 `DisposableEffect`를 사용합니다. `onDispose` 블록이 필수이며, 이를 빠뜨리면 IDE가 빌드 에러를 표시합니다.

```kotlin
@Composable
fun LifecycleAwareAnalytics(
    lifecycleOwner: LifecycleOwner = LocalLifecycleOwner.current
) {
    DisposableEffect(lifecycleOwner) {
        val observer = LifecycleEventObserver { _, event ->
            when (event) {
                Lifecycle.Event.ON_RESUME  -> Analytics.trackScreenResume()
                Lifecycle.Event.ON_PAUSE   -> Analytics.trackScreenPause()
                else -> Unit
            }
        }
        lifecycleOwner.lifecycle.addObserver(observer)

        // 컴포지션 이탈 or key 변경 시 반드시 해제
        onDispose {
            lifecycleOwner.lifecycle.removeObserver(observer)
        }
    }
}
```

이 패턴은 `MapView`, `MediaPlayer`, Bluetooth 연결 등 네이티브 리소스를 다룰 때 메모리 누수를 방지하는 핵심 도구입니다.

---

## 4. SideEffect — Compose 상태를 비-Compose 코드에 전달

`SideEffect`는 매 리컴포지션 성공 후 실행됩니다. Compose 외부 객체(예: Firebase Analytics, 분석 SDK)에 Compose 상태를 동기화할 때 사용합니다.

```kotlin
@Composable
fun AppContent(userType: String) {
    val analytics = remember { FirebaseAnalytics.getInstance(LocalContext.current) }

    // 리컴포지션이 성공할 때마다 userType을 Analytics에 동기화
    SideEffect {
        analytics.setUserProperty("user_type", userType)
    }

    MainScreen()
}
```

`SideEffect`는 suspend 함수를 호출할 수 없으며, 취소 개념도 없습니다. 단순 동기 작업에만 사용하세요.

---

## 5. produceState — 외부 데이터를 Compose State로 변환

`produceState`는 Flow, LiveData, 콜백 기반 API처럼 Compose가 아닌 소스의 데이터를 `State`로 변환합니다. 내부적으로 `LaunchedEffect`와 `remember { mutableStateOf() }`를 조합한 편의 API입니다.

```kotlin
@Composable
fun NetworkImage(url: String): State<ImageBitmap?> {
    return produceState<ImageBitmap?>(initialValue = null, url) {
        // suspend 블록: value 에 할당하면 State가 갱신됨
        value = withContext(Dispatchers.IO) {
            loadImageFromNetwork(url)
        }
        // awaitDispose로 정리 작업도 가능
        awaitDispose { /* 리소스 해제 */ }
    }
}

@Composable
fun AvatarImage(userId: String) {
    val avatarUrl = "https://example.com/avatars/$userId.png"
    val bitmap by NetworkImage(avatarUrl)

    if (bitmap != null) {
        Image(bitmap = bitmap!!, contentDescription = "Avatar")
    } else {
        CircularProgressIndicator()
    }
}
```

---

## 6. derivedStateOf — 과도한 리컴포지션 방지

`derivedStateOf`는 다른 State로부터 파생된 State를 만들 때 사용합니다. 입력 State가 자주 변하지만 파생값이 덜 변하는 경우 리컴포지션을 줄여줍니다.

```kotlin
@Composable
fun ChatList(messages: List<Message>) {
    val listState = rememberLazyListState()

    // messages 수가 바뀔 때마다가 아닌, 실제 버튼 표시 여부가 바뀔 때만 리컴포지션
    val showScrollToTop by remember {
        derivedStateOf { listState.firstVisibleItemIndex > 0 }
    }

    Box {
        LazyColumn(state = listState) {
            items(messages) { msg -> MessageItem(msg) }
        }
        if (showScrollToTop) {
            ScrollToTopButton(listState)
        }
    }
}
```

---

## 7. snapshotFlow — Compose State를 Flow로 변환

`snapshotFlow`는 `derivedStateOf`의 반대 방향입니다. Compose의 `State`를 Flow로 변환해 `collect`, `filter`, `debounce` 등 Flow 연산자를 활용할 수 있게 합니다.

```kotlin
@Composable
fun SearchScreen(viewModel: SearchViewModel) {
    val listState = rememberLazyListState()

    LaunchedEffect(listState) {
        snapshotFlow { listState.firstVisibleItemIndex }
            .distinctUntilChanged()
            .filter { it > 5 }
            .collect { index ->
                // 특정 위치 이상 스크롤 시 분석 이벤트 전송
                viewModel.logScrollDepth(index)
            }
    }

    LazyColumn(state = listState) {
        // ...
    }
}
```

`snapshotFlow`는 Compose Snapshot 시스템과 통합되어 State 변경을 감지합니다. `LaunchedEffect` 내에서 사용해 생명주기를 컴포지션에 위임하세요.

---

## 8. 실전 통합 예제: 실시간 검색 화면

아래 예제는 여러 Effect API를 조합해 실제 검색 화면을 구현합니다.

```kotlin
@Composable
fun SearchScreen(
    viewModel: SearchViewModel,
    lifecycleOwner: LifecycleOwner = LocalLifecycleOwner.current
) {
    var query by remember { mutableStateOf("") }
    val results by viewModel.searchResults.collectAsStateWithLifecycle()
    val listState = rememberLazyListState()

    // 1. 검색어가 바뀔 때 디바운스 후 검색 실행
    LaunchedEffect(query) {
        delay(300L)                 // 300ms 디바운스
        if (query.length >= 2) {
            viewModel.search(query)
        }
    }

    // 2. 생명주기 이벤트 감지 — 화면 재진입 시 캐시 갱신
    DisposableEffect(lifecycleOwner) {
        val observer = LifecycleEventObserver { _, event ->
            if (event == Lifecycle.Event.ON_RESUME) {
                viewModel.refreshIfStale()
            }
        }
        lifecycleOwner.lifecycle.addObserver(observer)
        onDispose { lifecycleOwner.lifecycle.removeObserver(observer) }
    }

    // 3. 스크롤 깊이 트래킹
    LaunchedEffect(listState) {
        snapshotFlow { listState.firstVisibleItemIndex }
            .distinctUntilChanged()
            .collect { viewModel.trackScrollDepth(it) }
    }

    // 4. 화면 노출 시마다 분석 이벤트
    SideEffect {
        viewModel.trackScreenView(query)
    }

    Column {
        OutlinedTextField(
            value = query,
            onValueChange = { query = it },
            label = { Text("검색") },
            modifier = Modifier.fillMaxWidth()
        )
        LazyColumn(state = listState) {
            items(results) { item -> SearchResultItem(item) }
        }
    }
}
```

---

## 9. 주의사항 및 실전 팁

### Key 선택 기준
- Effect 블록 내에서 읽는 **가변 값**은 반드시 key로 등록하세요.
- key를 `Unit`으로 고정하면 "컴포지션 생존 기간 동안 1회만" 실행됩니다. 의도적일 때만 사용하세요.
- 람다는 key로 쓰지 마세요. 매 리컴포지션마다 새 인스턴스가 만들어져 Effect가 끝없이 재시작됩니다. 대신 `rememberUpdatedState`를 활용하세요.

### Effect 선택 가이드
| 상황 | 사용할 API |
|---|---|
| suspend 함수 실행 (1회 or key 기반) | `LaunchedEffect` |
| 리스너/리소스 등록·해제 쌍 필요 | `DisposableEffect` |
| Compose 외부 객체에 State 동기화 (동기) | `SideEffect` |
| 외부 데이터 → `State` 변환 | `produceState` |
| 고빈도 State → 저빈도 파생값 | `derivedStateOf` |
| `State` → Flow 변환 | `snapshotFlow` |
| 컴포저블 외부에서 코루틴 필요 | `rememberCoroutineScope` |

### 흔한 실수
1. **`DisposableEffect` 없이 리스너 등록** — 컴포저블이 사라져도 해제되지 않아 메모리 누수 발생
2. **`LaunchedEffect`에 람다를 key로 사용** — 무한 재시작 루프
3. **`SideEffect` 내에서 suspend 호출** — 컴파일 에러 또는 IllegalStateException
4. **`derivedStateOf` 없이 고빈도 State를 직접 읽기** — 불필요한 리컴포지션 폭발

---

## 10. 마치며

Compose의 Effect API는 처음에는 종류가 많아 혼란스러울 수 있습니다. 그러나 핵심은 단순합니다. **"언제 시작하고, 언제 끝나고, 정리가 필요한가?"** 라는 세 가지 질문에 답하면 올바른 API가 자연스럽게 선택됩니다. 이 API들을 정확히 이해하면 Compose 앱의 안정성과 성능이 크게 향상되며, 메모리 누수나 레이스 컨디션 같은 난해한 버그를 원천 차단할 수 있습니다.

## 참고 자료
- [Side-effects in Compose - Android Developers 공식 문서](https://developer.android.com/develop/ui/compose/side-effects)
- [Advanced State and Side Effects in Jetpack Compose - Codelab](https://developer.android.com/codelabs/jetpack-compose-advanced-state-side-effects)
