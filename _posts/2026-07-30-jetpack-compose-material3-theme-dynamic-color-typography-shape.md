---
layout: post
title: "Jetpack Compose Material 3 심화: MaterialTheme·동적 색상·Typography·Shape으로 일관된 디자인 시스템 구축하기"
date: 2026-07-30
categories: [android, flutter]
tags: [android, jetpack-compose, material3, material-you, dynamic-color, typography, theming, design-system, kotlin]
---

Android 생태계에서 UI 개발의 중심이 Jetpack Compose로 이동하면서, 디자인 시스템의 기반이 되는 **Material Design 3(M3, Material You)** 역시 완전히 새롭게 재편되었습니다. Material 3는 단순한 컴포넌트 라이브러리가 아니라, 사용자의 기기 배경화면에서 색상 팔레트를 추출해 앱 전체에 동적으로 적용하는 개인화 시스템입니다. 이 글에서는 `MaterialTheme`의 내부 구조부터 동적 색상(Dynamic Color), Typography, Shape 시스템까지 심도 있게 분석하고, 실제 프로덕션에서 활용할 수 있는 완전한 테마 구현 전략을 다룹니다.

## Material 3가 필요한 이유: M2와 무엇이 다른가

Material Design 2는 수년간 Android 앱 개발의 표준이었지만, 단조로운 색상 팔레트(Primary/Secondary/Surface)와 제한된 개인화 기능에는 명백한 한계가 있었습니다. Material 3는 이러한 문제를 근본적으로 해결하기 위해 다음 세 가지 핵심 변화를 도입했습니다.

**첫째, 색상 역할(Color Role) 체계의 확장입니다.** M2는 Primary, Secondary, Background 등 12개 내외의 색상 슬롯을 사용했지만, M3는 `primary`, `onPrimary`, `primaryContainer`, `onPrimaryContainer`와 같이 각 색상에 대응하는 역할 쌍(pair)을 정의해 총 29개의 색상 역할을 제공합니다. 이 체계를 따르면 텍스트와 배경 간의 명도 대비를 항상 WCAG 접근성 기준에 맞게 유지할 수 있습니다.

**둘째, 동적 색상(Dynamic Color)입니다.** Android 12(API 31) 이상에서는 사용자의 홈 화면 배경화면을 분석해 5가지 기본 색상을 추출하고, Material Color Utilities 알고리즘이 이로부터 완전한 색상 스킴을 자동 생성합니다. 앱은 단 두 줄의 코드(`dynamicLightColorScheme`, `dynamicDarkColorScheme`)로 이 색상을 그대로 사용할 수 있습니다.

**셋째, 컴포넌트 라이브러리의 전면 재설계입니다.** `TopAppBar`가 3단계(CenterAligned, Small, Medium, Large)로 분화되고, `NavigationBar`·`NavigationRail`·`NavigationDrawer`가 화면 크기에 따라 적응형으로 전환되며, `SearchBar`·`DatePicker`·`TimePicker` 같은 완성도 높은 컴포넌트들이 새로 추가되었습니다.

## MaterialTheme의 구조: 세 가지 축

Compose Material 3의 `MaterialTheme`은 세 가지 서브시스템으로 구성됩니다.

```kotlin
MaterialTheme(
    colorScheme: ColorScheme = MaterialTheme.colorScheme,
    typography: Typography = MaterialTheme.typography,
    shapes: Shapes = MaterialTheme.shapes,
    content: @Composable () -> Unit
)
```

이 세 파라미터는 각각 `CompositionLocal`을 통해 컴포지션 트리 전체에 전파됩니다. 내부 구현을 보면 `LocalColorScheme`, `LocalTypography`, `LocalShapes`라는 세 개의 `ProvidableCompositionLocal`이 `MaterialTheme` 블록 안에서 `CompositionLocalProvider`로 주입됩니다. 덕분에 임의의 컴포저블에서 `MaterialTheme.colorScheme.primary`처럼 직접 접근할 수 있습니다.

### ColorScheme: 29개 색상 역할 이해하기

M3의 `ColorScheme`은 `primary`, `secondary`, `tertiary`의 세 가지 강조 색상(accent color)을 중심으로 구성됩니다. 각 강조 색상은 다음 네 가지 역할로 확장됩니다:

- `primary` — 기본 강조 배경색 (버튼, FAB 등)
- `onPrimary` — `primary` 위에 렌더링되는 콘텐츠 색 (텍스트, 아이콘)
- `primaryContainer` — `primary`의 연한 변형 (칩, 카드 등)
- `onPrimaryContainer` — `primaryContainer` 위 콘텐츠 색

이 패턴이 `secondary`, `tertiary`에도 동일하게 적용되며, 추가로 `surface`, `surfaceVariant`, `surfaceTint`, `error`, `background` 등의 역할이 존재합니다. **중요한 점은 `on*` 접두사 역할을 사용하면 명도 대비 4.5:1(AA 기준)이 자동으로 보장된다는 것입니다.**

### Typography: 15개 텍스트 스타일 체계

M3의 Typography는 `Display`, `Headline`, `Title`, `Body`, `Label`의 5가지 범주에 각각 `Large`, `Medium`, `Small`이 붙어 총 15개의 텍스트 스타일을 제공합니다.

| 범주 | 용도 |
|------|------|
| `displayLarge` (57sp) | 히어로 텍스트, 대형 숫자 |
| `headlineLarge` (32sp) | 섹션 제목 |
| `titleLarge` (22sp) | 앱 바 제목, 카드 제목 |
| `bodyLarge` (16sp) | 본문 텍스트 |
| `labelLarge` (14sp) | 버튼 텍스트, 탭 레이블 |

### Shapes: 5단계 곡률 시스템

`Shapes`는 `extraSmall`(4dp)부터 `extraLarge`(28dp)까지 5단계로 코너 반경을 정의합니다. `full` 은 완전한 원형(50%)을 의미합니다. 컴포넌트마다 기본 Shape가 정해져 있어(`Button` → `CircleShape`, `Card` → `RoundedCornerShape(12.dp)` 등), Shape 시스템을 일관되게 커스터마이징하면 앱 전체 룩앤필을 한 번에 바꿀 수 있습니다.

## 실제 구현 예제 1: 완전한 커스텀 Material 3 테마

다음은 프로덕션 앱에서 사용할 수 있는 완전한 Material 3 테마 구현 예제입니다.

```kotlin
import android.os.Build
import androidx.compose.foundation.isSystemInDarkTheme
import androidx.compose.material3.*
import androidx.compose.runtime.Composable
import androidx.compose.runtime.SideEffect
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.platform.LocalView
import androidx.core.view.WindowCompat

// ─── 라이트 테마 색상 스킴 ───────────────────────────────────────────────────
private val LightColorScheme = lightColorScheme(
    primary         = Color(0xFF6750A4),
    onPrimary       = Color(0xFFFFFFFF),
    primaryContainer    = Color(0xFFEADDFF),
    onPrimaryContainer  = Color(0xFF21005D),
    secondary       = Color(0xFF625B71),
    onSecondary     = Color(0xFFFFFFFF),
    secondaryContainer  = Color(0xFFE8DEF8),
    onSecondaryContainer = Color(0xFF1D192B),
    tertiary        = Color(0xFF7D5260),
    onTertiary      = Color(0xFFFFFFFF),
    tertiaryContainer   = Color(0xFFFFD8E4),
    onTertiaryContainer = Color(0xFF31111D),
    error           = Color(0xFFB3261E),
    errorContainer  = Color(0xFFF9DEDC),
    background      = Color(0xFFFFFBFE),
    surface         = Color(0xFFFFFBFE),
    surfaceVariant  = Color(0xFFE7E0EC),
    outline         = Color(0xFF79747E),
)

// ─── 다크 테마 색상 스킴 ────────────────────────────────────────────────────
private val DarkColorScheme = darkColorScheme(
    primary         = Color(0xFFD0BCFF),
    onPrimary       = Color(0xFF381E72),
    primaryContainer    = Color(0xFF4F378B),
    onPrimaryContainer  = Color(0xFFEADDFF),
    secondary       = Color(0xFFCCC2DC),
    onSecondary     = Color(0xFF332D41),
    secondaryContainer  = Color(0xFF4A4458),
    onSecondaryContainer = Color(0xFFE8DEF8),
    tertiary        = Color(0xFFEFB8C8),
    onTertiary      = Color(0xFF492532),
    tertiaryContainer   = Color(0xFF633B48),
    onTertiaryContainer = Color(0xFFFFD8E4),
    error           = Color(0xFFF2B8B5),
    errorContainer  = Color(0xFF8C1D18),
    background      = Color(0xFF1C1B1F),
    surface         = Color(0xFF1C1B1F),
    surfaceVariant  = Color(0xFF49454F),
    outline         = Color(0xFF938F99),
)

// ─── 커스텀 Typography ────────────────────────────────────────────────────
private val AppTypography = Typography(
    titleLarge = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.SemiBold,
        fontSize = 22.sp,
        lineHeight = 28.sp,
        letterSpacing = 0.sp
    ),
    bodyLarge = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Normal,
        fontSize = 16.sp,
        lineHeight = 24.sp,
        letterSpacing = 0.5.sp
    ),
    labelSmall = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Medium,
        fontSize = 11.sp,
        lineHeight = 16.sp,
        letterSpacing = 0.5.sp
    ),
)

// ─── 메인 테마 컴포저블 ───────────────────────────────────────────────────
@Composable
fun MyAppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    dynamicColor: Boolean = true,
    content: @Composable () -> Unit
) {
    val colorScheme = when {
        // Android 12+에서만 동적 색상 사용
        dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S -> {
            val context = LocalContext.current
            if (darkTheme) dynamicDarkColorScheme(context)
            else dynamicLightColorScheme(context)
        }
        darkTheme -> DarkColorScheme
        else      -> LightColorScheme
    }

    // 상태 바 색상을 테마에 맞게 동적으로 설정
    val view = LocalView.current
    if (!view.isInEditMode) {
        SideEffect {
            val window = (view.context as Activity).window
            WindowCompat.getInsetsController(window, view)
                .isAppearanceLightStatusBars = !darkTheme
        }
    }

    MaterialTheme(
        colorScheme = colorScheme,
        typography  = AppTypography,
        content     = content
    )
}
```

이 구현의 핵심 포인트는 다음과 같습니다.

1. **`dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S`** 조건으로 API 31 미만 기기에서는 fallback 색상을 사용합니다.
2. **`SideEffect` + `WindowCompat`** 조합으로 상태 바 아이콘 색상을 다크/라이트 테마에 맞게 실시간으로 전환합니다.
3. 별도의 `AppShapes`를 지정하지 않으면 M3 기본 Shape 시스템이 사용됩니다.

## 실제 구현 예제 2: M3 컴포넌트를 활용한 스크린 구현

다음은 `Scaffold`, `TopAppBar`, `NavigationBar`, `FloatingActionButton`, `Card` 등 M3 컴포넌트를 활용한 완전한 화면 구현 예제입니다.

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun ArticleListScreen(
    articles: List<Article>,
    selectedTab: Int,
    onTabSelected: (Int) -> Unit,
    onFabClick: () -> Unit,
) {
    Scaffold(
        // ─ TopAppBar: CenterAligned 스타일 ──────────────────────────────
        topBar = {
            CenterAlignedTopAppBar(
                title = {
                    Text(
                        text = "Ducktudy",
                        style = MaterialTheme.typography.titleLarge,
                    )
                },
                actions = {
                    IconButton(onClick = { /* 검색 */ }) {
                        Icon(
                            imageVector = Icons.Default.Search,
                            contentDescription = "검색",
                            tint = MaterialTheme.colorScheme.onSurface,
                        )
                    }
                },
                colors = TopAppBarDefaults.centerAlignedTopAppBarColors(
                    containerColor = MaterialTheme.colorScheme.surfaceColorAtElevation(3.dp),
                    titleContentColor = MaterialTheme.colorScheme.onSurface,
                ),
            )
        },
        // ─ FAB: 확장형 ──────────────────────────────────────────────────
        floatingActionButton = {
            ExtendedFloatingActionButton(
                text = { Text("새 글 작성") },
                icon = { Icon(Icons.Default.Edit, contentDescription = null) },
                onClick = onFabClick,
                containerColor = MaterialTheme.colorScheme.tertiaryContainer,
                contentColor  = MaterialTheme.colorScheme.onTertiaryContainer,
            )
        },
        // ─ NavigationBar ────────────────────────────────────────────────
        bottomBar = {
            NavigationBar(
                containerColor = MaterialTheme.colorScheme.surfaceColorAtElevation(3.dp),
            ) {
                listOf("홈", "즐겨찾기", "설정").forEachIndexed { index, label ->
                    val icon = when (index) {
                        0    -> Icons.Default.Home
                        1    -> Icons.Default.Favorite
                        else -> Icons.Default.Settings
                    }
                    NavigationBarItem(
                        selected = selectedTab == index,
                        onClick  = { onTabSelected(index) },
                        icon     = {
                            Icon(icon, contentDescription = label)
                        },
                        label    = { Text(label) },
                        colors   = NavigationBarItemDefaults.colors(
                            selectedIconColor       = MaterialTheme.colorScheme.onSecondaryContainer,
                            selectedTextColor       = MaterialTheme.colorScheme.onSurface,
                            indicatorColor          = MaterialTheme.colorScheme.secondaryContainer,
                            unselectedIconColor     = MaterialTheme.colorScheme.onSurfaceVariant,
                            unselectedTextColor     = MaterialTheme.colorScheme.onSurfaceVariant,
                        ),
                    )
                }
            }
        },
        containerColor = MaterialTheme.colorScheme.background,
    ) { paddingValues ->
        LazyColumn(
            contentPadding = paddingValues,
            verticalArrangement = Arrangement.spacedBy(8.dp),
        ) {
            items(articles) { article ->
                ArticleCard(article = article)
            }
        }
    }
}

// ─── 아티클 카드 컴포저블 ─────────────────────────────────────────────────
@Composable
fun ArticleCard(article: Article) {
    ElevatedCard(
        modifier = Modifier
            .fillMaxWidth()
            .padding(horizontal = 16.dp),
        // M3 Card는 기본적으로 MaterialTheme.shapes.medium 사용
        colors = CardDefaults.elevatedCardColors(
            containerColor = MaterialTheme.colorScheme.surfaceVariant,
        ),
    ) {
        Column(modifier = Modifier.padding(16.dp)) {
            // 카테고리 칩
            SuggestionChip(
                onClick = {},
                label   = {
                    Text(
                        text  = article.category,
                        style = MaterialTheme.typography.labelSmall,
                    )
                },
                colors = SuggestionChipDefaults.suggestionChipColors(
                    containerColor = MaterialTheme.colorScheme.primaryContainer,
                    labelColor     = MaterialTheme.colorScheme.onPrimaryContainer,
                ),
            )
            Spacer(modifier = Modifier.height(8.dp))
            // 제목
            Text(
                text  = article.title,
                style = MaterialTheme.typography.titleMedium,
                color = MaterialTheme.colorScheme.onSurface,
                maxLines = 2,
                overflow = TextOverflow.Ellipsis,
            )
            Spacer(modifier = Modifier.height(4.dp))
            // 날짜
            Text(
                text  = article.date,
                style = MaterialTheme.typography.bodySmall,
                color = MaterialTheme.colorScheme.onSurfaceVariant,
            )
        }
    }
}
```

이 코드에서 주목할 점들입니다:

- **`surfaceColorAtElevation(dp)`**: M3에서는 고도(elevation)를 그림자가 아닌 **토날 오버레이(tonal overlay)**로 표현합니다. 이 함수를 사용하면 고도에 따른 `primary` 색상의 오버레이가 자동으로 계산됩니다.
- **`NavigationBarItemDefaults.colors(indicatorColor = ...)`**: 선택된 아이템에 나타나는 M3 고유의 pill 형태 indicator 색상을 커스터마이징합니다.
- **`ElevatedCard`**: M3는 `FilledCard`, `ElevatedCard`, `OutlinedCard` 세 가지 변형을 제공합니다.

## Material 3 테마 확장: 커스텀 색상 추가하기

프로젝트에 M3의 기본 색상 역할에 없는 색상(예: 구독 배너 강조색, 프리미엄 배지 색)이 필요할 때는 `ColorScheme`을 직접 확장합니다.

```kotlin
// ColorScheme 확장 프로퍼티로 커스텀 색상 추가
val ColorScheme.premium: Color
    @Composable
    get() = if (isSystemInDarkTheme()) Color(0xFFFFD700) else Color(0xFFB8860B)

val ColorScheme.subscriptionBanner: Color
    @Composable
    get() = if (isSystemInDarkTheme()) Color(0xFF1A3A5C) else Color(0xFFDEEAF7)

// 사용 예시
@Composable
fun PremiumBadge() {
    Box(
        modifier = Modifier
            .background(
                color = MaterialTheme.colorScheme.premium,
                shape = MaterialTheme.shapes.extraSmall,
            )
            .padding(horizontal = 6.dp, vertical = 2.dp)
    ) {
        Text(
            text  = "PRO",
            style = MaterialTheme.typography.labelSmall,
            color = Color.Black,
        )
    }
}
```

## 주의사항 및 실전 팁

### 1. Material 2와 3를 혼용하지 말 것

`androidx.compose.material`(M2)과 `androidx.compose.material3`(M3)를 같은 프로젝트에 함께 임포트하면 `Text`, `Button`, `Icon` 등 동일한 이름의 컴포저블이 충돌합니다. 반드시 하나만 사용하거나, 임포트 별칭으로 명확히 구분해야 합니다.

### 2. dynamicColor는 API 30 이하에서 반드시 false 처리

`dynamicLightColorScheme` / `dynamicDarkColorScheme`은 API 31 미만에서 `UnsupportedOperationException`을 던집니다. `Build.VERSION.SDK_INT >= Build.VERSION_CODES.S` 조건을 반드시 확인하세요.

### 3. Material Theme Builder 활용

Google의 [Material Theme Builder](https://material-foundation.github.io/material-theme-builder/) 웹 도구를 사용하면 브랜드 색상 하나를 입력해 전체 `lightColorScheme`/`darkColorScheme` 코드를 자동 생성할 수 있습니다. 이 코드를 그대로 프로젝트에 붙여넣으면 일관성 있는 색상 시스템을 빠르게 구축할 수 있습니다.

### 4. 상태 바/네비게이션 바 색상과의 연동

Edge-to-edge 레이아웃(`WindowCompat.setDecorFitsSystemWindows(window, false)`)을 적용할 때, 상태 바와 네비게이션 바의 아이콘 색상을 `SideEffect` 안에서 `WindowCompat.getInsetsController`로 동기화해야 합니다.

### 5. `isSystemInDarkTheme()`만으로는 부족하다

앱 내 다크 모드 토글을 지원하려면 `DataStore`에 사용자 설정을 저장하고, `ViewModel`에서 Flow로 노출한 뒤, `MyAppTheme(darkTheme = uiState.isDarkMode)`처럼 외부에서 주입하는 패턴을 사용해야 합니다. `isSystemInDarkTheme()`는 시스템 설정만 반영하므로 앱 내 설정이 있으면 이를 우선해야 합니다.

### 6. Compose BOM 사용

`androidx.compose:compose-bom`을 사용하면 Material 3를 포함한 모든 Compose 라이브러리의 버전 호환성을 한 번에 관리할 수 있습니다.

```kotlin
// build.gradle.kts
implementation(platform("androidx.compose:compose-bom:2026.07.00"))
implementation("androidx.compose.material3:material3")
```

## 마치며

Material 3는 단순히 컴포넌트의 모양을 바꾸는 것이 아닌, 사용자 경험의 개인화와 접근성을 설계의 중심에 놓는 패러다임 전환입니다. `colorScheme`의 색상 역할 체계를 이해하고, 동적 색상과 폴백 테마를 올바르게 처리하며, Typography와 Shape 시스템을 일관되게 활용할 때 비로소 M3가 약속하는 진정한 Material You 경험을 앱에 구현할 수 있습니다.

## 참고 자료
- [Material Design 3 in Jetpack Compose - Android Developers](https://developer.android.com/develop/ui/compose/designsystems/material3)
- [Compose Material 3 Release Notes - AndroidX](https://developer.android.com/jetpack/androidx/releases/compose-material3)
- [Custom Design Systems in Jetpack Compose - Android Developers](https://developer.android.com/develop/ui/compose/designsystems/custom)
