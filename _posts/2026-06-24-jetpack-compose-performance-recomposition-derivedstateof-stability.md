---
layout: post
title: "Jetpack Compose 성능 최적화 심화: Recomposition 최소화, derivedStateOf, Stability 완전 정복"
date: 2026-06-24
categories: [android, flutter]
tags: [android, jetpack-compose, recomposition, derivedStateOf, stability, performance, kotlin]
---

Jetpack Compose는 선언형 UI 패러다임으로 Android 개발의 방식을 근본적으로 바꿨습니다. 그런데 "선언형이라 쉽다"는 인식과 달리, **성능 최적화**는 오히려 더 섬세한 이해를 요구합니다. 특히 Recomposition(재구성)이 불필요하게 과다 발생하면 앱 프레임이 뚝뚝 끊기고 배터리를 과도하게 소모하게 됩니다. 이 글에서는 Recomposition의 내부 동작 원리부터 `derivedStateOf`, `Stability`, `@Stable` 어노테이션까지 심층적으로 살펴봅니다.

---

## 1. Recomposition이란 무엇인가?

Compose의 핵심 철학은 **UI를 상태(State)의 함수**로 표현하는 것입니다. 상태가 바뀌면 Compose 런타임은 해당 상태를 읽는 Composable 함수를 다시 호출하여 UI를 갱신합니다. 이 과정을 **Recomposition**이라고 합니다.

```
State 변경 → 영향받는 Composable 재호출 → 새로운 Composition → UI 업데이트
```

Recomposition 자체는 Compose의 핵심 메커니즘이므로 나쁜 것이 아닙니다. 문제는 **필요 이상으로 많이 발생하는 불필요한 Recomposition**입니다.

### Recomposition의 스마트한 점

Compose 런타임은 기본적으로 영리하게 동작합니다. 어떤 Composable의 입력 파라미터가 바뀌지 않았고, 그 Composable이 **스킵 가능(Skippable)** 하다면 재호출을 건너뜁니다. 이것이 Compose 성능 최적화의 핵심입니다.

### 실제 문제가 발생하는 시나리오

```kotlin
// ❌ 문제 있는 코드: 매번 새 람다 인스턴스 생성
@Composable
fun BadExample(items: List<String>) {
    items.forEach { item ->
        // 매 Recomposition마다 새 람다 객체 생성 → 불필요한 하위 Recomposition 유발
        ItemRow(text = item, onClick = { handleClick(item) })
    }
}
```

---

## 2. 왜 불필요한 Recomposition이 발생하는가?

### 2-1. 불안정한(Unstable) 파라미터

Compose 컴파일러는 각 Composable에 대해 **안정성(Stability)** 을 추론합니다. 파라미터가 안정적이면 컴파일러는 "이전 값과 같으면 스킵해도 된다"는 최적화 코드를 생성합니다. 반대로 파라미터가 불안정하면 매번 재호출합니다.

**안정적(Stable)** 으로 인정받는 타입:
- 원시 타입: `Int`, `String`, `Boolean`, `Float` 등
- 불변(immutable) 데이터 클래스: 모든 프로퍼티가 `val`이고 Stable한 타입인 경우
- Kotlin의 `@Immutable` / `@Stable` 어노테이션이 붙은 클래스

**불안정(Unstable)** 으로 취급받는 타입:
- `List<T>`, `Set<T>`, `Map<K, V>` — Kotlin 컬렉션은 인터페이스이므로 가변일 수 있음
- `var` 프로퍼티를 가진 클래스
- 외부 라이브러리의 클래스 (컴파일러가 내부를 볼 수 없음)

### 2-2. 잘못 설정된 State 읽기 범위

State를 읽는 위치가 넓을수록 Recomposition 범위가 커집니다.

```kotlin
// ❌ 나쁜 예: 최상위에서 offset을 읽어 Column 전체가 Recompose
@Composable
fun BadScrollExample() {
    val listState = rememberLazyListState()
    val offset = listState.firstVisibleItemScrollOffset // 여기서 읽음

    Column {
        // offset이 바뀔 때마다 Column 전체 Recompose
        Header(offset = offset)
        LazyList(state = listState)
    }
}

// ✅ 좋은 예: Modifier.offset의 람다 내부에서 읽어 레이아웃 단계로 State 읽기 지연
@Composable
fun GoodScrollExample() {
    val listState = rememberLazyListState()

    Box {
        LazyList(state = listState)
        // Modifier 람다는 Layout Phase에서 실행 → Recomposition 없이 위치만 갱신
        Header(
            modifier = Modifier.offset {
                IntOffset(x = 0, y = -listState.firstVisibleItemScrollOffset)
            }
        )
    }
}
```

---

## 3. `derivedStateOf`: 파생 상태의 최적화

### 언제 사용해야 하는가?

`derivedStateOf`는 **"하나 이상의 State 객체를 입력으로 받아 새로운 State를 계산할 때"** 사용합니다. 결정적인 차이는 **원본 State가 자주 바뀌더라도 계산 결과가 바뀌지 않으면 Recomposition을 트리거하지 않는다**는 점입니다.

### 실전 예제: 스크롤 위치에 따른 버튼 표시

```kotlin
@Composable
fun ScrollToTopButton() {
    val listState = rememberLazyListState()

    // ❌ 잘못된 방법: firstVisibleItemIndex가 바뀔 때마다 Recompose
    // val showButton = listState.firstVisibleItemIndex > 0

    // ✅ 올바른 방법: 결과(true/false)가 바뀔 때만 Recompose
    val showButton by remember {
        derivedStateOf { listState.firstVisibleItemIndex > 0 }
    }

    Box {
        LazyColumn(state = listState) {
            items(100) { index ->
                Text(
                    text = "아이템 $index",
                    modifier = Modifier
                        .fillMaxWidth()
                        .padding(16.dp)
                )
            }
        }

        AnimatedVisibility(
            visible = showButton,
            modifier = Modifier.align(Alignment.BottomEnd)
        ) {
            FloatingActionButton(
                onClick = { /* 최상단으로 스크롤 */ }
            ) {
                Icon(Icons.Default.KeyboardArrowUp, contentDescription = "맨 위로")
            }
        }
    }
}
```

사용자가 스크롤하는 동안 `firstVisibleItemIndex`는 끊임없이 바뀝니다. 하지만 `derivedStateOf`를 사용하면, **0에서 1로 바뀌는 순간**(false→true)과 **1에서 0으로 바뀌는 순간**(true→false)에만 버튼 Composable이 Recompose됩니다.

### `derivedStateOf` 남용 주의

```kotlin
// ❌ 불필요한 derivedStateOf — 단순 State 전달에 사용할 필요 없음
val name by remember { derivedStateOf { viewModel.name } }

// ✅ 그냥 이렇게 쓰세요
val name by viewModel.nameState
```

`derivedStateOf`는 "계산 비용이 있고, 입력이 자주 바뀌지만 결과는 가끔만 바뀌는 경우"에 의미가 있습니다.

---

## 4. Stability: 스킵 가능한 Composable 만들기

### Compose 컴파일러 리포트 활용

Compose 컴파일러는 `--reportsDestination` 옵션으로 각 Composable의 안정성 분석 결과를 파일로 출력할 수 있습니다.

`build.gradle.kts`에 다음을 추가합니다:

```kotlin
composeCompiler {
    reportsDestination = layout.buildDirectory.dir("compose_compiler")
    stabilityConfigurationFiles = listOf(
        rootProject.layout.projectDirectory.file("stability_config.conf")
    )
}
```

생성된 리포트 파일에서 각 Composable의 안정성을 확인할 수 있습니다:

```
restartable skippable scheme("[androidx.compose.ui.UiComposable]") fun UserCard(
  stable name: String
  stable age: Int
  unstable profile: UserProfile  // ← 이게 문제!
)
```

`unstable`로 표시된 파라미터가 있으면 해당 Composable은 스킵되지 않습니다.

### `@Stable` 과 `@Immutable` 어노테이션

```kotlin
// ❌ 불안정한 데이터 클래스 (List 포함)
data class UserProfile(
    val name: String,
    val tags: List<String>  // Kotlin List는 불안정
)

// ✅ 방법 1: @Immutable 어노테이션 사용
// 계약: 이 클래스의 공개 프로퍼티는 절대 바뀌지 않는다고 컴파일러에 약속
@Immutable
data class UserProfile(
    val name: String,
    val tags: List<String>
)

// ✅ 방법 2: ImmutableList 사용 (kotlinx.collections.immutable)
// build.gradle.kts: implementation("org.jetbrains.kotlinx:kotlinx-collections-immutable:0.3.7")
data class UserProfile(
    val name: String,
    val tags: ImmutableList<String>  // 안정적으로 인식됨
)
```

`@Immutable`은 더 강한 계약입니다. "이 클래스의 모든 공개 프로퍼티는 생성 이후 절대 바뀌지 않는다"고 컴파일러에 알립니다. 이를 어기면 런타임 버그가 발생할 수 있으므로 신중하게 사용해야 합니다.

`@Stable`은 더 약한 계약으로, "프로퍼티가 바뀌면 Compose에 알려준다"는 의미입니다. `mutableStateOf`로 관리되는 프로퍼티를 가진 클래스에 적합합니다.

---

## 5. 람다와 remember: 놓치기 쉬운 성능 트랩

### 람다 참조의 안정성

Compose Composable에 람다를 파라미터로 전달할 때, 람다 자체가 불안정한 타입으로 취급될 수 있습니다.

```kotlin
// ❌ 매 Recomposition마다 새 람다 객체 생성 → 하위 Composable 스킵 불가
@Composable
fun ParentScreen(viewModel: MyViewModel) {
    ItemList(
        items = viewModel.items,
        onItemClick = { id -> viewModel.onItemSelected(id) }  // 매번 새 람다
    )
}

// ✅ remember로 람다 캐싱
@Composable
fun ParentScreen(viewModel: MyViewModel) {
    val onItemClick = remember(viewModel) {
        { id: String -> viewModel.onItemSelected(id) }
    }

    ItemList(
        items = viewModel.items,
        onItemClick = onItemClick
    )
}
```

### State 읽기를 최대한 늦춰라 (Defer State Reads)

Compose의 렌더링 파이프라인은 세 단계로 구성됩니다:

1. **Composition** (구성): Composable 함수 실행, UI 트리 생성
2. **Layout** (레이아웃): 각 UI 요소의 크기와 위치 결정
3. **Drawing** (드로잉): 실제 화면에 픽셀 그리기

State 읽기를 **늦은 단계**로 미루면 앞 단계의 Recomposition을 건너뛸 수 있습니다.

```kotlin
// ❌ Composition 단계에서 State 읽기 → 매번 전체 Recompose
@Composable
fun AnimatedHeader(scrollState: ScrollState) {
    val alpha = 1f - (scrollState.value / 500f).coerceIn(0f, 1f)
    Box(modifier = Modifier.alpha(alpha)) {
        Text("헤더")
    }
}

// ✅ Drawing 단계로 State 읽기 지연 → Recomposition 없음
@Composable
fun AnimatedHeader(scrollState: ScrollState) {
    Box(
        modifier = Modifier.graphicsLayer {
            // 이 람다는 Drawing Phase에서 실행됨
            alpha = 1f - (scrollState.value / 500f).coerceIn(0f, 1f)
        }
    ) {
        Text("헤더")
    }
}
```

---

## 6. 실전 최적화 체크리스트

### 6-1. LazyList 아이템의 key 설정

```kotlin
LazyColumn {
    // ❌ key 없음: 아이템 추가/삭제 시 전체 재구성
    items(userList) { user ->
        UserCard(user = user)
    }

    // ✅ stable한 key 지정: 변경된 아이템만 Recompose
    items(
        items = userList,
        key = { user -> user.id }  // 고유하고 안정적인 키
    ) { user ->
        UserCard(user = user)
    }
}
```

### 6-2. `remember`에 올바른 key 전달

```kotlin
// ❌ key 없음: 항상 같은 값을 캐시 (userId 변경 반영 안 됨)
val userDetails = remember { fetchUserDetails(userId) }

// ✅ userId를 key로 전달: userId 변경 시 재계산
val userDetails = remember(userId) { fetchUserDetails(userId) }
```

### 6-3. ViewModel State를 세분화하여 공개

```kotlin
// ❌ 하나의 거대한 UiState: 어느 한 부분만 바뀌어도 전체 Recompose
data class BadUiState(
    val userName: String,
    val userAvatar: Bitmap?,
    val feedItems: List<FeedItem>,
    val isLoading: Boolean,
    val errorMessage: String?
)

// ✅ 관심사별로 분리된 State 흐름
class GoodViewModel : ViewModel() {
    val userName: StateFlow<String> = ...
    val feedItems: StateFlow<ImmutableList<FeedItem>> = ...
    val isLoading: StateFlow<Boolean> = ...
    // 각 State를 구독하는 Composable만 Recompose됨
}
```

---

## 7. 성능 측정: Compose Metrics와 Layout Inspector

### Composition Tracing

Android Studio의 **Layout Inspector**와 **Profiler**를 활용하여 불필요한 Recomposition을 시각적으로 확인할 수 있습니다. 특히 `Recomposition Count` 컬럼에서 특정 Composable이 지나치게 많이 재구성되는지 확인하세요.

```kotlin
// Compose Tracing 라이브러리로 커스텀 트레이스 추가
// build.gradle.kts: implementation("androidx.compose.runtime:runtime-tracing:1.0.0")
// 자동으로 Composable 함수 이름이 Perfetto 트레이스에 기록됨
```

### 릴리즈 빌드에서 테스트하라

Compose의 디버그 빌드는 추가적인 검증 코드가 포함되어 있어 실제보다 느립니다. **성능 프로파일링은 반드시 릴리즈 빌드**에서 진행해야 정확한 데이터를 얻을 수 있습니다.

```bash
./gradlew assembleRelease
adb install -r app/build/outputs/apk/release/app-release.apk
```

---

## 8. 주의사항 및 핵심 팁 정리

| 상황 | 잘못된 접근 | 올바른 접근 |
|------|------------|------------|
| 리스트 파라미터 | `List<T>` 직접 전달 | `ImmutableList<T>` 또는 `@Immutable` 데이터 클래스 |
| 파생 State | 매번 재계산 | `derivedStateOf`로 결과가 바뀔 때만 Recompose |
| 람다 파라미터 | 인라인 람다 생성 | `remember`로 캐싱 |
| 애니메이션 값 | Composition 단계에서 읽기 | `graphicsLayer`, `Modifier.offset { }` 등 지연 읽기 |
| LazyList 아이템 | key 미설정 | 안정적인 `key` 함수 전달 |
| 성능 측정 | 디버그 빌드 | 릴리즈 빌드에서 프로파일링 |

불필요한 Recomposition을 없애는 것은 단순히 "최적화 기법"이 아닙니다. Compose의 동작 원리를 이해하고, **데이터의 안정성**과 **State 읽기 범위**를 설계 단계에서부터 고려하는 것이 핵심입니다. Compose 컴파일러 리포트를 주기적으로 확인하고, 측정 → 분석 → 개선의 사이클을 꾸준히 반복하세요.

## 참고 자료
- [Jetpack Compose Performance | Android Developers](https://developer.android.com/develop/ui/compose/performance)
- [Follow best practices | Jetpack Compose | Android Developers](https://developer.android.com/develop/ui/compose/performance/bestpractices)
- [State and Jetpack Compose | Android Developers](https://developer.android.com/develop/ui/compose/state)
