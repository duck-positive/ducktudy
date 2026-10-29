---
layout: post
title: "Compose for Wear OS 완전 정복: TransformingLazyColumn과 SwipeDismissable Navigation 심화"
date: 2026-10-29
categories: [android, flutter]
tags: [wear-os, compose, android, TransformingLazyColumn, SwipeDismissableNavHost, material3, rotary-input]
---

스마트워치 앱 개발은 스마트폰 앱 개발과 근본적으로 다른 접근 방식을 요구합니다. Jetpack Compose for Wear OS는 이 특수한 환경을 위한 전용 UI 컴포넌트를 제공하며, 최신 버전(1.7.x)에서는 Material 3 기반의 `TransformingLazyColumn`, `ScreenScaffold`, `SwipeDismissableNavHost` 등 강력한 API가 안정화되었습니다. 이 글에서는 실제 심박수 모니터링 앱을 예시로 삼아 Compose for Wear OS의 핵심 개념과 구현 패턴을 깊이 있게 다룹니다.

## 1. Compose for Wear OS란?

Compose for Wear OS는 Google이 스마트워치 앱 UI 개발을 위해 Jetpack Compose를 Wear OS 환경에 맞게 최적화한 라이브러리 모음입니다. 일반 Compose와 동일한 선언형 프로그래밍 패러다임을 따르면서도, 스마트워치의 특수한 물리적 제약과 인터랙션 패턴을 처리하는 전용 컴포넌트들을 제공합니다.

스마트워치는 스마트폰과 근본적으로 다른 사용 환경을 가집니다:

- **원형(Round) 또는 소형 사각 디스플레이**: 화면 크기가 40~50mm에 불과하며, 원형 화면의 경우 가장자리 콘텐츠가 잘려 보일 수 있습니다
- **베젤(Bezel) 기반 회전 입력**: Crown 또는 베젤을 돌려 스크롤하는 Rotary Input이 주요 인터랙션 방법입니다
- **Swipe-to-Dismiss 패턴**: 화면을 왼쪽에서 오른쪽으로 스와이프해서 이전 화면으로 돌아가는 고유한 네비게이션 방식입니다
- **극도로 제한된 배터리**: 상시 표시(Always-On Display), 환경 모드 등을 고려해야 합니다
- **짧은 인터랙션 시간**: 평균 사용 시간이 2~3초 수준으로, UI가 극도로 단순하고 빠르게 읽혀야 합니다

이러한 환경에 최적화된 UI를 위해 Compose for Wear OS는 `TransformingLazyColumn`, `SwipeDismissableNavHost`, `ScreenScaffold` 등의 전용 컴포넌트를 제공합니다.

### 의존성 설정

```kotlin
// app/build.gradle.kts
dependencies {
    val wearComposeVersion = "1.7.1"

    // Wear OS Material 3 (일반 mobile material3 대신 사용)
    implementation("androidx.wear.compose:compose-material3:$wearComposeVersion")

    // Wear OS Foundation (스크롤, 포커스 등)
    implementation("androidx.wear.compose:compose-foundation:$wearComposeVersion")

    // Wear OS Navigation (일반 navigation-compose 대신 사용)
    implementation("androidx.wear.compose:compose-navigation:$wearComposeVersion")

    // Tooling (Preview 지원)
    debugImplementation("androidx.wear.compose:compose-ui-tooling:$wearComposeVersion")
}
```

주의할 점은 **Wear OS용 라이브러리는 반드시 mobile용과 분리하여 사용**해야 한다는 것입니다. `androidx.compose.material3:material3`와 `androidx.wear.compose:compose-material3`를 혼용하면 `MaterialTheme`가 두 개 존재하게 되어 색상, 타이포그래피, 도형이 불일치하는 문제가 발생합니다.

---

## 2. 왜 Wear OS 전용 컴포넌트가 필요한가?

일반 Compose의 `LazyColumn`을 Wear OS에서 그대로 사용하면 세 가지 핵심 문제가 발생합니다.

**첫째, 원형 화면 가장자리 클리핑 문제**: 원형 화면에서 리스트의 상단과 하단 항목은 코너에 걸려 일부가 잘립니다. 사용자는 내용을 제대로 읽을 수 없고, 터치 조작이 어렵습니다.

**둘째, Rotary Input 미지원**: 일반 `LazyColumn`은 베젤 회전 입력이 자동으로 연결되지 않습니다. 별도 처리 없이는 손가락 터치로만 스크롤할 수 있어 Wear OS 기기의 주요 입력 방식을 무시하게 됩니다.

**셋째, 시각적 피드백 부재**: Wear OS 사용자는 짧은 시간 안에 현재 스크롤 위치를 파악해야 합니다. 전용 `PositionIndicator`와 스케일/페이드 애니메이션(TransformingEffect)이 있어야 직관적인 UX를 제공할 수 있습니다.

`TransformingLazyColumn`은 이 모든 문제를 해결합니다. 리스트 항목이 화면 상하단으로 이동할수록 자동으로 크기가 줄어들고 투명도가 낮아지는 변형 효과(Transforming Effect)를 적용하여, 원형 화면에서도 자연스럽고 읽기 쉬운 UI를 만들어줍니다.

> **ScalingLazyColumn vs TransformingLazyColumn**: 이전 Wear Compose Material 2.x에서는 `ScalingLazyColumn`을 사용했지만, Material 3 기반 Wear Compose(1.7.x 이상)에서는 `TransformingLazyColumn`이 권장됩니다. `transformedHeight`와 `SurfaceTransformation` API가 더 정밀한 레이아웃 제어를 제공하며 성능도 향상되었습니다.

---

## 3. 실제 구현 예제

### 예제 1: TransformingLazyColumn과 ScreenScaffold로 심박수 목록 구현

다음 예제는 심박수 데이터를 `TransformingLazyColumn`으로 표시하는 기본 Wear OS 화면입니다. `ScreenScaffold`가 시간 표시, 포지션 인디케이터, EdgeButton 슬롯을 자동으로 관리합니다.

```kotlin
data class HeartRateEntry(val time: String, val bpm: Int)

@Composable
fun HeartRateListScreen(
    entries: List<HeartRateEntry>,
    onEntryClick: (HeartRateEntry) -> Unit
) {
    val columnState = rememberTransformingLazyColumnState()
    val transformationSpec = rememberTransformationSpec()

    ScreenScaffold(
        scrollState = columnState,
        timeText = {
            // 스크롤 시 TimeText가 화면 위로 사라지는 효과
            TimeText(modifier = Modifier.scrollAway { columnState })
        },
        edgeButton = {
            EdgeButton(onClick = { /* 새 측정 시작 */ }) {
                Icon(Icons.Default.Add, contentDescription = "측정 시작")
            }
        }
    ) { contentPadding ->
        TransformingLazyColumn(
            state = columnState,
            contentPadding = contentPadding
        ) {
            item {
                // ListHeader: 원형 화면 상단에 자연스럽게 배치되는 제목 컴포넌트
                ListHeader {
                    Text(
                        text = "심박수 기록",
                        style = MaterialTheme.typography.titleMedium
                    )
                }
            }

            items(entries, key = { it.time }) { entry ->
                TitleCard(
                    modifier = Modifier
                        .fillMaxWidth()
                        // 스크롤 중 높이 변화를 레이아웃 시스템에 알림 (필수)
                        .transformedHeight(this, transformationSpec),
                    // 스케일 + 페이드 변형 효과 적용
                    transformation = SurfaceTransformation(transformationSpec),
                    onClick = { onEntryClick(entry) },
                    title = { Text(entry.time) }
                ) {
                    Row(
                        verticalAlignment = Alignment.CenterVertically,
                        horizontalArrangement = Arrangement.SpaceBetween,
                        modifier = Modifier.fillMaxWidth()
                    ) {
                        Text(
                            text = "${entry.bpm} BPM",
                            style = MaterialTheme.typography.bodyLarge,
                            color = when {
                                entry.bpm > 100 -> MaterialTheme.colorScheme.error
                                entry.bpm < 60  -> MaterialTheme.colorScheme.secondary
                                else            -> MaterialTheme.colorScheme.onSurface
                            }
                        )
                        Icon(
                            imageVector = Icons.Default.Favorite,
                            contentDescription = null,
                            tint = MaterialTheme.colorScheme.primary
                        )
                    }
                }
            }
        }
    }
}
```

여기서 핵심 포인트는 두 가지입니다:
- `Modifier.transformedHeight(this, transformationSpec)`: 스크롤 중 레이아웃 높이 변화를 컴포지션 시스템에 알려줍니다. 빠진 경우 항목 간 겹침 또는 빈 공간 문제가 발생합니다.
- `transformation = SurfaceTransformation(transformationSpec)`: 실제 스케일 및 페이드 효과를 Card/Button에 적용합니다. `TitleCard`, `Button` 등 Wear Material 3 컴포넌트는 이 파라미터를 지원합니다.

---

### 예제 2: SwipeDismissableNavHost와 화면 간 이동

Wear OS의 네비게이션은 일반 Compose Navigation과 API가 유사하지만, Swipe-to-Dismiss 동작을 기본으로 내장한 `SwipeDismissableNavHost`를 사용합니다. 화면을 오른쪽으로 스와이프하면 자동으로 이전 화면으로 돌아갑니다.

```kotlin
sealed class Screen(val route: String) {
    data object HeartRateList : Screen("heart_rate_list")
    data object Settings : Screen("settings")

    data object HeartRateDetail : Screen("heart_rate_detail/{time}") {
        fun createRoute(time: String) = "heart_rate_detail/$time"
    }
}

@Composable
fun WearApp() {
    // Wear OS 전용 NavController (WearNavigator 기반)
    val navController = rememberSwipeDismissableNavController()

    // AppScaffold: TimeText 같은 앱 수준 UI를 최상위에서 관리
    // SwipeDismissableNavHost를 반드시 AppScaffold 안에 배치
    AppScaffold {
        SwipeDismissableNavHost(
            navController = navController,
            startDestination = Screen.HeartRateList.route
        ) {
            composable(Screen.HeartRateList.route) {
                HeartRateListScreen(
                    entries = sampleHeartRateData,
                    onEntryClick = { entry ->
                        navController.navigate(
                            Screen.HeartRateDetail.createRoute(entry.time)
                        )
                    }
                )
            }

            composable(
                route = Screen.HeartRateDetail.route,
                arguments = listOf(
                    navArgument("time") { type = NavType.StringType }
                )
            ) { backStackEntry ->
                val time = backStackEntry.arguments?.getString("time")
                    ?: return@composable

                HeartRateDetailScreen(time = time)
            }

            composable(Screen.Settings.route) {
                SettingsScreen()
            }
        }
    }
}

@Composable
fun HeartRateDetailScreen(time: String) {
    val columnState = rememberTransformingLazyColumnState()
    val transformationSpec = rememberTransformationSpec()

    // ScreenScaffold: 화면별 PositionIndicator와 EdgeButton 슬롯 제공
    ScreenScaffold(
        scrollState = columnState
    ) { contentPadding ->
        TransformingLazyColumn(
            state = columnState,
            contentPadding = contentPadding
        ) {
            item {
                Text(
                    text = "측정 시간: $time",
                    style = MaterialTheme.typography.titleSmall,
                    modifier = Modifier
                        .fillMaxWidth()
                        .padding(horizontal = 10.dp),
                    textAlign = TextAlign.Center
                )
            }
            item {
                Card(
                    modifier = Modifier
                        .fillMaxWidth()
                        .transformedHeight(this, transformationSpec),
                    transformation = SurfaceTransformation(transformationSpec),
                    onClick = {}
                ) {
                    Column(
                        modifier = Modifier
                            .fillMaxWidth()
                            .padding(16.dp),
                        horizontalAlignment = Alignment.CenterHorizontally
                    ) {
                        Text(
                            text = "72 BPM",
                            style = MaterialTheme.typography.displaySmall,
                            color = MaterialTheme.colorScheme.primary
                        )
                        Text(
                            text = "정상 범위",
                            style = MaterialTheme.typography.labelMedium,
                            color = MaterialTheme.colorScheme.onSurfaceVariant
                        )
                    }
                }
            }
            // 추가 상세 데이터...
        }
    }
}
```

`AppScaffold` + `SwipeDismissableNavHost` + `ScreenScaffold`의 세 레이어 구조는 Compose for Wear OS에서 권장하는 기본 아키텍처입니다:

| 레이어 | 역할 |
|---|---|
| `AppScaffold` | 앱 전체의 `TimeText` 등 글로벌 UI 관리 |
| `SwipeDismissableNavHost` | 화면 간 이동 및 Swipe-to-Dismiss 처리 |
| `ScreenScaffold` | 화면별 `PositionIndicator`, `EdgeButton` 관리 |

---

## 4. 주의사항 및 실전 팁

### Rotary Input 명시적 연결

`ScreenScaffold`가 내부적으로 Rotary Input을 `TransformingLazyColumn`에 연결하지만, 커스텀 스크롤 가능 컴포넌트를 구현하는 경우에는 직접 연결해야 합니다. `rememberActiveFocusRequester()`와 함께 사용하여 포커스도 잊지 않고 처리하세요:

```kotlin
val focusRequester = rememberActiveFocusRequester()
val coroutineScope = rememberCoroutineScope()

MyCustomScrollable(
    modifier = Modifier
        .rotaryScrollable(
            behavior = RotaryScrollableDefaults.behavior(lazyListState),
            focusRequester = focusRequester
        )
        .focusRequester(focusRequester)
        .focusable()
)
```

### 배터리와 Always-On Display 최적화

Wear OS 앱은 배터리 소비를 최소화해야 합니다. 환경 모드(Ambient Mode)가 활성화되면 색상을 흑백으로 제한하고 애니메이션을 완전히 제거하세요. `AmbientLifecycleObserver`를 `ViewModel`에 연결하여 앱 상태에 따라 UI를 자동 전환하는 패턴이 권장됩니다.

또한, 불필요한 recomposition을 줄이기 위해 `@Stable`, `@Immutable` 어노테이션을 데이터 클래스에 적극 활용하고, `derivedStateOf`로 파생 상태를 효율적으로 관리하세요.

### Preview와 테스트

Android Studio에서 Wear OS Emulator를 사용해 원형/사각형 화면 모두 테스트하세요. Compose Preview에서는 `@WearPreviewDevices` 어노테이션으로 여러 Wear OS 기기 크기를 한 번에 확인할 수 있습니다. Rotary Input은 에뮬레이터에서 마우스 스크롤로 시뮬레이션 가능합니다.

```kotlin
@WearPreviewDevices
@WearPreviewFontScales
@Composable
fun HeartRateListScreenPreview() {
    WearAppTheme {
        HeartRateListScreen(
            entries = listOf(
                HeartRateEntry("10:30", 72),
                HeartRateEntry("11:00", 85),
                HeartRateEntry("11:30", 68)
            ),
            onEntryClick = {}
        )
    }
}
```

### Mobile Compose와 Wear Compose 혼용 금지

`androidx.compose.material3:material3`와 `androidx.wear.compose:compose-material3`를 동시에 임포트하면 `MaterialTheme`, `Button`, `Text` 등 동일한 이름의 컴포넌트가 충돌합니다. Wear OS 전용 모듈만 의존성에 포함하고, 만약 두 라이브러리를 함께 사용해야 한다면 import alias를 적극 활용하여 혼동을 방지하세요:

```kotlin
import androidx.compose.material3.Button as MobileButton
import androidx.wear.compose.material3.Button as WearButton
```

---

## 정리

Compose for Wear OS는 일반 Compose의 강력함을 스마트워치의 제약된 환경에 맞게 재설계한 라이브러리입니다. `TransformingLazyColumn`의 자동 변형 효과, `SwipeDismissableNavHost`의 네이티브 네비게이션, `ScreenScaffold`의 구조적 레이아웃 관리를 함께 사용하면 자연스럽고 배터리 효율적인 Wear OS 앱을 빠르게 개발할 수 있습니다. 특히 Mobile Compose 라이브러리와 절대 혼용하지 않는 것, Rotary Input을 명시적으로 연결하는 것, 환경 모드 대응을 설계 초기부터 고려하는 것이 핵심 원칙입니다.

## 참고 자료
- [Compose for Wear OS 공식 시작 가이드](https://developer.android.com/training/wearables/compose)
- [TransformingLazyColumn으로 Wear OS 목록 구현하기](https://developer.android.com/training/wearables/compose/lists)
- [Compose for Wear OS Navigation 가이드](https://developer.android.com/training/wearables/compose/navigation)
