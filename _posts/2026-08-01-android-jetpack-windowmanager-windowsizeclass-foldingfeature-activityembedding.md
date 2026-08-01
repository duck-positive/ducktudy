---
layout: post
title: "Android Jetpack WindowManager 1.5 심화: WindowSizeClass·FoldingFeature·ActivityEmbedding으로 폴더블과 태블릿을 완전 지원하는 적응형 UI 구현하기"
date: 2026-08-01
categories: [android]
tags: [android, jetpack, windowmanager, windowsizeclass, foldingfeature, activityembedding, foldable, tablet, adaptive-layout, compose]
---

## 개요

2025년 현재, Android 생태계는 폰(phone), 태블릿(tablet), 폴더블(foldable), 크롬북(ChromeOS), 외장 디스플레이(connected display)까지 폭발적으로 다양해졌습니다. 더 이상 "스마트폰 세로 화면"만 고려한 레이아웃으로는 충분하지 않습니다. Google은 이 다양한 폼 팩터를 단일 API 세트로 대응할 수 있도록 Jetpack WindowManager 라이브러리를 지속적으로 발전시켜 왔고, 2025년 하반기에 릴리스된 1.5.0 버전에서 주요 기능이 안정화되었습니다.

이 아티클에서는 Jetpack WindowManager의 세 가지 핵심 축인 **WindowSizeClass**, **FoldingFeature**, **ActivityEmbedding**을 깊이 있게 다루고, 실제 Compose 코드와 함께 적응형 앱을 구현하는 방법을 단계별로 설명합니다.

---

## 왜 Jetpack WindowManager가 필요한가

과거에는 `Display.getSize()`, `WindowManager.getDefaultDisplay()` 같은 API로 화면 크기를 구했습니다. 그러나 이 방식에는 치명적인 문제가 있습니다.

- **멀티 윈도우 환경 미지원**: 분할 화면(Split Screen)이나 자유 형식 창(Free-form Window) 모드에서는 앱이 사용할 수 있는 영역이 기기 전체 화면 크기와 다릅니다.
- **폴더블 기기 미지원**: 폴더블 기기는 접힌 상태와 펼쳐진 상태에 따라 사용 가능한 영역이 완전히 달라집니다.
- **하드웨어 중심 사고**: 물리적 해상도보다 앱에게 실제로 할당된 창(window) 크기가 더 중요합니다.

Jetpack WindowManager는 이 모든 시나리오를 일관된 API로 다룰 수 있게 해주며, 앱이 실행 중인 환경이 바뀌면 Flow/Callback으로 변경 사항을 알려줍니다.

---

## 1. WindowSizeClass: 반응형 레이아웃의 공통 언어

### 개념

`WindowSizeClass`는 앱이 사용할 수 있는 창 크기를 **Compact · Medium · Expanded · Large · Extra-large** 다섯 단계로 분류하는 표준 브레이크포인트 시스템입니다. 픽셀 수나 인치가 아닌 dp 단위를 사용하므로 화면 밀도와 무관하게 일관성 있는 판단이 가능합니다.

| 클래스 | 너비 기준(dp) | 대표 기기 |
|---|---|---|
| Compact | < 600 | 세로 모드 스마트폰 |
| Medium | 600 ~ 839 | 세로 모드 태블릿 |
| Expanded | 840 ~ 1199 | 가로 모드 태블릿 |
| Large | 1200 ~ 1599 | 대형 태블릿, 외장 디스플레이 |
| Extra-large | ≥ 1600 | 데스크톱급 디스플레이 |

WindowManager 1.5에서 `Large`와 `Extra-large`가 새로 추가되었으며, Android 16의 외장 디스플레이 연결 기능(connected displays)을 직접 지원합니다.

### Compose에서 WindowSizeClass 활용하기

```kotlin
// build.gradle.kts
dependencies {
    implementation("androidx.window:window:1.5.0")
    implementation("androidx.compose.material3.adaptive:adaptive:1.2.0")
}
```

```kotlin
@Composable
fun AdaptiveApp() {
    // currentWindowAdaptiveInfo는 Compose Material3 Adaptive 라이브러리 제공
    // Large/Extra-large 클래스 활성화를 위해 반드시 true 전달
    val adaptiveInfo = currentWindowAdaptiveInfo(supportLargeAndXLargeWidth = true)
    val sizeClass = adaptiveInfo.windowSizeClass

    when {
        sizeClass.isWidthAtLeastBreakpoint(WindowSizeClass.WIDTH_DP_EXPANDED_LOWER_BOUND) -> {
            // 840dp 이상: 태블릿 가로, 폴더블 펼침
            TwoPaneLayout()
        }
        sizeClass.isWidthAtLeastBreakpoint(WindowSizeClass.WIDTH_DP_MEDIUM_LOWER_BOUND) -> {
            // 600dp 이상: 태블릿 세로
            NavigationRailLayout()
        }
        else -> {
            // Compact: 일반 스마트폰
            SinglePaneLayout()
        }
    }
}

@Composable
fun TwoPaneLayout() {
    Row(modifier = Modifier.fillMaxSize()) {
        NavigationRail(modifier = Modifier.fillMaxHeight()) {
            NavigationRailItem(
                selected = true,
                onClick = {},
                icon = { Icon(Icons.Default.Home, contentDescription = "홈") },
                label = { Text("홈") }
            )
            NavigationRailItem(
                selected = false,
                onClick = {},
                icon = { Icon(Icons.Default.Person, contentDescription = "프로필") },
                label = { Text("프로필") }
            )
        }
        Row(modifier = Modifier.weight(1f)) {
            LazyColumn(modifier = Modifier.weight(1f)) {
                items(20) { index ->
                    ListItem(
                        headlineContent = { Text("항목 $index") },
                        modifier = Modifier.clickable {}
                    )
                }
            }
            Box(
                modifier = Modifier
                    .weight(1f)
                    .fillMaxHeight()
                    .background(MaterialTheme.colorScheme.surfaceVariant),
                contentAlignment = Alignment.Center
            ) {
                Text("상세 내용", style = MaterialTheme.typography.bodyLarge)
            }
        }
    }
}
```

### View 기반 코드에서 WindowMetricsCalculator 직접 사용

Compose 환경이 아닌 일반 Activity/Fragment에서도 WindowSizeClass를 구할 수 있습니다.

```kotlin
val windowMetrics = WindowMetricsCalculator.getOrCreate()
    .computeCurrentWindowMetrics(this) // Activity 또는 Application context (1.5부터 지원)

val sizeClass = WindowSizeClass.BREAKPOINTS_V2
    .computeWindowSizeClass(windowMetrics)

val isExpanded = sizeClass.isWidthAtLeastBreakpoint(
    WindowSizeClass.WIDTH_DP_EXPANDED_LOWER_BOUND
)

// Large/Extra-large 여부 확인
val isLargeOrBigger = sizeClass.isWidthAtLeastBreakpoint(
    WindowSizeClass.WIDTH_DP_LARGE_LOWER_BOUND
)

Log.d("WindowSize", "너비 클래스: ${sizeClass.windowWidthSizeClass}, " +
    "isExpanded=$isExpanded, isLargeOrBigger=$isLargeOrBigger")
```

---

## 2. FoldingFeature: 폴더블 기기의 접힘 상태 감지

### 개념

`FoldingFeature`는 폴더블 기기의 힌지(hinge) 정보를 추상화한 클래스입니다. 힌지의 위치, 접힘 각도(state), 방향(orientation), 힌지가 콘텐츠를 가리는지 여부(occlusionType)를 알 수 있습니다.

| 속성 | 가능한 값 | 설명 |
|---|---|---|
| `state` | FLAT, HALF_OPENED | 완전히 펴짐 또는 반만 펴짐 |
| `orientation` | HORIZONTAL, VERTICAL | 힌지 방향 |
| `occlusionType` | NONE, FULL | 힌지가 콘텐츠를 가리는지 여부 |
| `isSeparating` | Boolean | 두 개의 논리적 디스플레이 영역으로 분리되는지 여부 |
| `bounds` | Rect | 힌지 영역의 경계 사각형 |

폴더블 기기의 대표적인 자세는 두 가지입니다.

- **테이블탑(TableTop)**: 가로 힌지로 반 접힌 상태 → 영상 통화, 동영상 시청에 유용
- **북(Book)**: 세로 힌지로 반 접힌 상태 → 전자책 읽기, 이중 패널 UI에 유용

### WindowInfoTracker Flow로 실시간 감지

```kotlin
@Composable
fun FoldAwareScreen() {
    val context = LocalContext.current
    val activity = context as ComponentActivity

    var foldingFeature by remember { mutableStateOf<FoldingFeature?>(null) }

    LaunchedEffect(Unit) {
        WindowInfoTracker.getOrCreate(activity)
            .windowLayoutInfo(activity)
            .collect { layoutInfo ->
                foldingFeature = layoutInfo.displayFeatures
                    .filterIsInstance<FoldingFeature>()
                    .firstOrNull()
            }
    }

    val postureLabel = when {
        isTableTopPosture(foldingFeature) -> "테이블탑 자세 (가로 힌지, 반 접힘)"
        isBookPosture(foldingFeature) -> "북 자세 (세로 힌지, 반 접힘)"
        foldingFeature?.state == FoldingFeature.State.FLAT -> "완전히 펼쳐진 상태"
        else -> "일반 화면"
    }

    if (isTableTopPosture(foldingFeature)) {
        TableTopLayout(foldingFeature!!)
    } else {
        Box(modifier = Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
            Text(text = postureLabel, style = MaterialTheme.typography.headlineSmall)
        }
    }
}

fun isTableTopPosture(feature: FoldingFeature?): Boolean =
    feature?.state == FoldingFeature.State.HALF_OPENED &&
    feature.orientation == FoldingFeature.Orientation.HORIZONTAL

fun isBookPosture(feature: FoldingFeature?): Boolean =
    feature?.state == FoldingFeature.State.HALF_OPENED &&
    feature.orientation == FoldingFeature.Orientation.VERTICAL

@Composable
fun TableTopLayout(feature: FoldingFeature) {
    val density = LocalDensity.current
    Column(modifier = Modifier.fillMaxSize()) {
        // 상단: 주요 콘텐츠 (예: 동영상 플레이어)
        Box(
            modifier = Modifier
                .fillMaxWidth()
                .weight(1f)
                .background(Color.Black),
            contentAlignment = Alignment.Center
        ) {
            Icon(
                imageVector = Icons.Default.PlayArrow,
                contentDescription = "동영상 재생",
                tint = Color.White,
                modifier = Modifier.size(64.dp)
            )
        }

        // occlusionType이 FULL이면 힌지 영역에 콘텐츠 배치 금지
        if (feature.occlusionType == FoldingFeature.OcclusionType.FULL) {
            Spacer(
                modifier = Modifier
                    .fillMaxWidth()
                    .height(with(density) { feature.bounds.height().toDp() })
                    .background(Color.DarkGray)
            )
        }

        // 하단: 컨트롤 패널
        Row(
            modifier = Modifier
                .fillMaxWidth()
                .weight(1f)
                .padding(16.dp),
            horizontalArrangement = Arrangement.SpaceEvenly,
            verticalAlignment = Alignment.CenterVertically
        ) {
            IconButton(onClick = {}) { Icon(Icons.Default.SkipPrevious, "이전") }
            IconButton(onClick = {}) { Icon(Icons.Default.PlayArrow, "재생") }
            IconButton(onClick = {}) { Icon(Icons.Default.SkipNext, "다음") }
        }
    }
}
```

---

## 3. ActivityEmbedding: 두 Activity를 한 화면에 나란히

`ActivityEmbedding`은 태블릿처럼 넓은 화면에서 두 Activity를 동시에 표시하는 기능입니다. 기존의 Fragment 기반 마스터-디테일 패턴을 Activity 수준에서 구현할 수 있게 해주며, 레거시 앱 구조를 유지하면서도 대화면 최적화가 가능합니다.

### XML 규칙 파일로 분할 선언

```xml
<!-- res/xml/main_split_config.xml -->
<SplitPairRule
    xmlns:window="http://schemas.android.com/apk/res-auto"
    window:splitRatio="0.4"
    window:splitMinWidthDp="840"
    window:finishPrimaryWithSecondary="never"
    window:finishSecondaryWithPrimary="always">
    <SplitPairFilter
        window:primaryActivityName=".MainActivity"
        window:secondaryActivityName=".DetailActivity"/>
</SplitPairRule>

<ActivityRule
    window:alwaysExpand="true">
    <ActivityFilter
        window:activityName=".FullScreenActivity"/>
</ActivityRule>
```

```kotlin
// Application에서 규칙 등록
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        RuleController.getInstance(this)
            .setRules(RuleController.parseRules(this, R.xml.main_split_config))
    }
}

// MainActivity에서 분할 상태 감지 및 Detail 열기
class MainActivity : AppCompatActivity() {

    private val splitController by lazy {
        SplitController.getInstance(this)
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // 기기가 ActivityEmbedding을 지원하는지 확인
        if (!SplitController.getInstance(this).isSplitSupported()) {
            Log.d("ActivityEmbedding", "이 기기는 ActivityEmbedding을 지원하지 않습니다.")
        }

        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                splitController.splitInfoList(this@MainActivity)
                    .collect { splitInfoList ->
                        val isSplit = splitInfoList.isNotEmpty()
                        Log.d("ActivityEmbedding", "분할 모드 활성화: $isSplit")
                    }
            }
        }
    }

    fun openDetail(itemId: Long) {
        Intent(this, DetailActivity::class.java).apply {
            putExtra("item_id", itemId)
            startActivity(this)
            // SplitPairRule에 의해 840dp 이상 화면에서는 자동으로 나란히 배치됨
        }
    }
}
```

WindowManager 1.5에서는 `EmbeddingConfiguration#isAutoSaveEmbeddingState()`를 활성화하면 프로세스 재생성 후에도 분할 상태가 자동으로 복원됩니다.

---

## 주의사항 및 팁

### 1. WindowSizeClass는 하드웨어 크기가 아닌 앱 창 크기

멀티 윈도우 모드나 분할 화면에서는 실제 화면 크기가 아닌 앱에 할당된 창 크기를 기준으로 WindowSizeClass가 결정됩니다. `Display.getSize()` 등 하드웨어 기반 API와 절대로 혼용하지 마세요. 항상 `WindowMetricsCalculator`를 사용하세요.

### 2. Configuration Change 처리 전략

폴더블 기기를 펼치거나 접으면 `onConfigurationChanged`가 발생합니다. `AndroidManifest.xml`에 `android:configChanges="screenSize|smallestScreenSize|screenLayout|orientation"` 을 선언하면 Activity 재생성 없이 처리할 수 있지만, `WindowInfoTracker`의 Flow를 `repeatOnLifecycle`과 함께 수집하는 방식이 더 안전하고 ViewModel과의 연동에도 자연스럽습니다.

### 3. 실제 기기 없이 FoldingFeature 테스트

`window-testing` 아티팩트의 `WindowLayoutInfoPublisherRule`을 사용하면 에뮬레이터나 실제 폴더블 기기 없이도 힌지 상태를 시뮬레이션할 수 있습니다.

```kotlin
@get:Rule
val windowLayoutInfoPublisherRule = WindowLayoutInfoPublisherRule()

@Test
fun testTableTopPosture() {
    val scenario = ActivityScenario.launch(MainActivity::class.java)
    scenario.onActivity { activity ->
        val hinge = FoldingFeature(
            activity = activity,
            state = FoldingFeature.State.HALF_OPENED,
            orientation = FoldingFeature.Orientation.HORIZONTAL,
            size = 0
        )
        windowLayoutInfoPublisherRule.overrideWindowLayoutInfo(
            WindowLayoutInfo(listOf(hinge))
        )
    }
    onView(withText("테이블탑 자세 (가로 힌지, 반 접힘)"))
        .check(matches(isDisplayed()))
}
```

### 4. ActivityEmbedding은 API 32+ 및 OEM 지원 필요

ActivityEmbedding은 Android 12L (API 32) 이상에서, 그리고 OEM이 해당 기능을 지원하는 기기에서만 활성화됩니다. 앱 초기화 시 반드시 `SplitController.getInstance(this).isSplitSupported()`로 지원 여부를 확인하고, 미지원 환경에서는 Fragment 기반 UI를 폴백으로 제공하세요.

### 5. Large/Extra-large 클래스 명시적 활성화 필요

WindowManager 1.5의 새 브레이크포인트(Large, Extra-large)를 사용하려면 `currentWindowAdaptiveInfo(supportLargeAndXLargeWidth = true)` 옵션을 반드시 명시해야 합니다. 기본값은 `false`이므로 누락 시 1200dp 이상 창에서도 Expanded 클래스가 반환됩니다.

---

## 참고 자료
- [Window Size Classes 공식 가이드](https://developer.android.com/develop/adaptive-apps/guides/use-window-size-classes)
- [폴더블 앱 구현 가이드 — FoldingFeature](https://developer.android.com/develop/ui/compose/layouts/adaptive/foldables/make-your-app-fold-aware)
- [다양한 디스플레이 크기 지원 가이드](https://developer.android.com/develop/adaptive-apps/guides/support-different-display-sizes)
- [Jetpack WindowManager 1.5 안정 버전 릴리스 노트](https://developer.android.com/blog/posts/jetpack-window-manager-1-5-is-stable)
