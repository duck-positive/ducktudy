---
layout: post
title: "Android Compose for TV 심화: D-Pad 포커스 관리부터 TV Material 컴포넌트까지 완전 정복"
date: 2026-10-22
categories: [android, flutter]
tags: [android, tv, compose, jetpack-compose, dpad, focus, leanback, androidx-tv]
---

Android TV 앱 개발은 스마트폰 앱 개발과 본질적으로 다릅니다. 사용자는 소파에 2~3미터 떨어진 채 리모컨으로 앱을 조작하는 "10-Foot UI" 환경에서 살아가고 있습니다. 터치스크린 대신 D-Pad(방향키)와 SELECT, BACK 버튼이 전부인 이 환경에서 **포커스(Focus) 관리**는 UX의 심장이라 할 수 있습니다. 이번 글에서는 Jetpack Compose for TV를 활용해 TV 앱을 현대적으로 개발하는 방법을 심층적으로 다룹니다.

---

## 1. 개념 설명: Compose for TV란 무엇인가

### 1.1 Leanback의 시대에서 Compose의 시대로

과거 Android TV 앱 개발의 표준은 **Leanback 라이브러리**(`androidx.leanback`)였습니다. `BrowseSupportFragment`, `DetailsFragment`, `PlaybackSupportFragment` 등 Fragment 기반 컴포넌트들이 TV UI의 전부였죠. 하지만 Leanback은 여러 한계를 가지고 있었습니다.

첫째, **Fragment 상속 강제**. TV 앱 구조가 Leanback이 제공하는 특정 Fragment 구조에 종속되어 자유로운 커스터마이징이 어려웠습니다. 둘째, **선언형 UI 불가**. Jetpack Compose가 Android 개발의 주류가 된 이후에도 Leanback은 여전히 명령형 View 시스템에 머물러 있었습니다. 셋째, **보일러플레이트 코드 과다**. 단순한 카탈로그 화면 하나를 만드는 데도 수백 줄의 코드가 필요했습니다.

이를 해결하기 위해 Google은 **Jetpack Compose for TV**(`androidx.tv:tv-material`)를 도입했습니다. 선언형 Compose UI를 TV에 그대로 가져오되, TV 환경에 최적화된 컴포넌트와 포커스 동작을 제공합니다.

### 1.2 Compose for TV 핵심 구성 요소

Compose for TV는 두 레이어로 구성됩니다:

| 라이브러리 | 역할 |
|---|---|
| `androidx.tv:tv-material` | TV 전용 Material 컴포넌트 (Card, Button, NavigationDrawer 등) |
| `androidx.compose.foundation` (1.7.0+) | 표준 LazyColumn/LazyRow에 TV 포커스 스크롤 통합 |

> **중요 변경사항**: `androidx.tv.foundation`의 `TvLazyColumn`, `TvLazyRow` 등 TV 전용 레이지 레이아웃은 **deprecated** 되었으며, Compose Foundation 1.7.0부터 표준 `LazyColumn`, `LazyRow`로 통합되었습니다.

---

## 2. 왜 필요한가: D-Pad 포커스 관리의 복잡성

스마트폰에서는 사용자가 원하는 위치를 직접 터치합니다. TV에서는 D-Pad로 포커스를 이동한 뒤 SELECT 버튼으로만 선택할 수 있습니다. TV 앱에서 포커스 관리가 복잡한 이유는 다음과 같습니다:

- **2D 네비게이션**: 상하좌우 방향키로 격자 형태의 UI를 탐색해야 함
- **포커스 복원**: 상세 화면에서 돌아왔을 때 마지막으로 본 아이템에 포커스 복원 필요
- **포커스 가시성**: 포커스된 아이템이 항상 화면 내에 보여야 하므로 자동 스크롤 필요
- **초기 포커스**: 화면 진입 시 어느 요소에 포커스를 줄지 명시적 설정 필요

기존 Leanback에서 이 모든 것을 직접 구현하면 코드가 크게 복잡해졌습니다. Compose for TV는 이를 선언적으로 훨씬 간결하게 해결합니다.

---

## 3. 실제 구현 예제

### 3.1 예제 1: Compose for TV 기본 카탈로그 앱

먼저 의존성을 설정하고, TV Material 테마를 적용한 기본 카탈로그 화면을 구성합니다.

**`build.gradle.kts` (앱 모듈)**:

```kotlin
dependencies {
    val composeBom = platform("androidx.compose:compose-bom:2026.09.00")
    implementation(composeBom)

    implementation("androidx.activity:activity-compose:1.13.0")
    implementation("androidx.compose.ui:ui-tooling-preview")
    debugImplementation("androidx.compose.ui:ui-tooling")

    // Compose for TV — Mobile의 material3 대신 반드시 이것을 사용
    implementation("androidx.tv:tv-material:1.0.0")

    // 이미지 로딩
    implementation("io.coil-kt:coil-compose:2.7.0")
}
```

**`MainActivity.kt`**:

```kotlin
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.LazyRow
import androidx.compose.foundation.lazy.items
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.layout.ContentScale
import androidx.compose.ui.unit.dp
import androidx.tv.material3.*
import coil.compose.AsyncImage

data class Movie(val id: String, val title: String, val genre: String, val thumbnailUrl: String)

@OptIn(ExperimentalTvMaterial3Api::class)
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            // ⚠️ androidx.tv.material3.MaterialTheme 사용
            // androidx.compose.material3.MaterialTheme와 혼용 금지
            MaterialTheme(colorScheme = darkColorScheme()) {
                TvCatalogScreen()
            }
        }
    }
}

@OptIn(ExperimentalTvMaterial3Api::class)
@Composable
fun TvCatalogScreen() {
    val moviesByGenre = remember {
        sampleMovies().groupBy { it.genre }
    }

    // 표준 LazyColumn 사용 (TvLazyColumn 아님 — deprecated)
    LazyColumn(
        modifier = Modifier.fillMaxSize(),
        contentPadding = PaddingValues(vertical = 48.dp, horizontal = 80.dp),
        verticalArrangement = Arrangement.spacedBy(32.dp)
    ) {
        moviesByGenre.forEach { (genre, films) ->
            item(key = genre) {
                Text(
                    text = genre,
                    style = MaterialTheme.typography.headlineMedium,
                    color = MaterialTheme.colorScheme.onBackground
                )
                Spacer(modifier = Modifier.height(12.dp))
                MovieCatalogRow(movies = films)
            }
        }
    }
}

@Composable
fun MovieCatalogRow(movies: List<Movie>) {
    // 표준 LazyRow 사용 (TvLazyRow 아님 — deprecated)
    LazyRow(
        horizontalArrangement = Arrangement.spacedBy(16.dp)
    ) {
        items(movies, key = { it.id }) { movie ->
            MovieCard(movie = movie)
        }
    }
}

@OptIn(ExperimentalTvMaterial3Api::class)
@Composable
fun MovieCard(movie: Movie, modifier: Modifier = Modifier) {
    var isFocused by remember { mutableStateOf(false) }

    // androidx.tv.material3.Card — TV 전용 포커스 인디케이터 자동 포함
    Card(
        onClick = { /* 상세 화면 이동 */ },
        modifier = modifier
            .size(width = 200.dp, height = 150.dp)
            .onFocusChanged { isFocused = it.isFocused },
        scale = CardDefaults.scale(focusedScale = 1.08f),
        border = CardDefaults.border(
            focusedBorder = Border(
                border = BorderStroke(3.dp, Color(0xFFE5A00D)),
                shape = RoundedCornerShape(8.dp)
            )
        )
    ) {
        Box(modifier = Modifier.fillMaxSize()) {
            AsyncImage(
                model = movie.thumbnailUrl,
                contentDescription = movie.title,
                contentScale = ContentScale.Crop,
                modifier = Modifier.fillMaxSize()
            )
            // 포커스 시 오버레이 표시
            if (isFocused) {
                Box(
                    modifier = Modifier
                        .fillMaxSize()
                        .background(Color.Black.copy(alpha = 0.35f))
                ) {
                    Text(
                        text = "▶ 재생",
                        modifier = Modifier.align(Alignment.Center),
                        color = Color.White,
                        style = MaterialTheme.typography.bodyLarge
                    )
                }
            }
            // 제목 항상 표시
            Text(
                text = movie.title,
                modifier = Modifier
                    .align(Alignment.BottomStart)
                    .background(Color.Black.copy(alpha = 0.5f))
                    .padding(8.dp),
                color = Color.White,
                style = MaterialTheme.typography.labelLarge
            )
        }
    }
}
```

이 코드에서 `androidx.tv.material3.Card`는 D-Pad 포커스 이동 시 황금색 테두리와 1.08배 확대 애니메이션이 자동으로 적용됩니다. Mobile `Card`에는 없는 TV 특화 포커스 동작이 내장되어 있습니다.

---

### 3.2 예제 2: 포커스 복원과 BringIntoViewSpec

TV 앱에서 가장 흔한 UX 문제는 **포커스 초기화**입니다. 카탈로그에서 스크롤을 내린 뒤 상세 화면으로 갔다가 돌아오면, 포커스가 첫 번째 아이템으로 돌아가버립니다. 이를 해결하는 완전한 포커스 복원 구현입니다.

```kotlin
import androidx.compose.foundation.LocalBringIntoViewSpec
import androidx.compose.foundation.BringIntoViewSpec
import androidx.compose.ui.focus.focusRestorer

/**
 * BringIntoViewSpec 커스텀 구현:
 * 포커스된 아이템을 LazyRow/LazyColumn의 지정 위치에 스크롤
 *
 * @param parentFraction 컨테이너 기준 아이템을 위치시킬 비율 (0.0 = 시작, 1.0 = 끝)
 * @param childFraction  아이템 내 정렬 기준점 비율
 */
@Composable
fun PositionFocusedItemInLazyLayout(
    parentFraction: Float = 0.3f,
    childFraction: Float = 0.5f,
    content: @Composable () -> Unit
) {
    val bringIntoViewSpec = remember(parentFraction, childFraction) {
        object : BringIntoViewSpec {
            override fun calculateScrollDistance(
                offset: Float,
                size: Float,
                containerSize: Float
            ): Float {
                val childSmallerThanParent = size <= containerSize
                val idealTarget = containerSize * parentFraction - size * childFraction
                val targetForItem = if (childSmallerThanParent) {
                    idealTarget.coerceIn(0f, containerSize - size)
                } else {
                    idealTarget
                }
                return offset - targetForItem
            }
        }
    }
    CompositionLocalProvider(LocalBringIntoViewSpec provides bringIntoViewSpec) {
        content()
    }
}

@Composable
fun TvCatalogWithFocusRestoration() {
    val movies = remember { sampleMovies() }
    val moviesByGenre = movies.groupBy { it.genre }

    PositionFocusedItemInLazyLayout(
        parentFraction = 0.1f, // 컨테이너 10% 지점에 아이템 위치
        childFraction = 0.0f   // 아이템의 왼쪽 끝이 그 지점에 정렬
    ) {
        LazyColumn(
            modifier = Modifier
                .fillMaxSize()
                // focusRestorer: 이 Composable 그룹에서 포커스가 나갔다가
                // 돌아올 때 마지막 포커스 위치를 자동으로 복원
                .focusRestorer(),
            contentPadding = PaddingValues(vertical = 48.dp, horizontal = 80.dp),
            verticalArrangement = Arrangement.spacedBy(32.dp)
        ) {
            moviesByGenre.forEach { (genre, films) ->
                item(key = genre) {
                    GenreSectionWithRestoration(genre = genre, movies = films)
                }
            }
        }
    }
}

@Composable
fun GenreSectionWithRestoration(genre: String, movies: List<Movie>) {
    Column {
        Text(text = genre, style = MaterialTheme.typography.titleLarge)
        Spacer(modifier = Modifier.height(8.dp))

        LazyRow(
            modifier = Modifier.focusRestorer(), // 가로 행 내 포커스도 복원
            horizontalArrangement = Arrangement.spacedBy(12.dp)
        ) {
            items(movies, key = { it.id }) { movie ->
                MovieCard(movie = movie)
            }
        }
    }
}

// 특정 아이템에 초기 포커스 설정
@Composable
fun TvScreenWithInitialFocus() {
    val firstItemRequester = remember { FocusRequester() }

    LaunchedEffect(Unit) {
        // 화면 진입 시 첫 번째 아이템에 포커스 자동 이동
        firstItemRequester.requestFocus()
    }

    LazyRow(horizontalArrangement = Arrangement.spacedBy(12.dp)) {
        itemsIndexed(sampleMovies()) { index, movie ->
            MovieCard(
                movie = movie,
                modifier = if (index == 0) Modifier.focusRequester(firstItemRequester)
                           else Modifier
            )
        }
    }
}

// 비디오 플레이어: D-Pad 키 이벤트 직접 처리
@Composable
fun VideoPlayerOverlay(
    isPlaying: Boolean,
    onPlayPause: () -> Unit,
    onSeekBack: () -> Unit,
    onSeekForward: () -> Unit,
    modifier: Modifier = Modifier
) {
    Box(
        modifier = modifier
            .fillMaxSize()
            .focusable() // 키 이벤트를 받으려면 반드시 필요
            .onKeyEvent { keyEvent ->
                if (keyEvent.type == KeyEventType.KeyDown) {
                    when (keyEvent.key) {
                        Key.DirectionLeft  -> { onSeekBack();    true }
                        Key.DirectionRight -> { onSeekForward(); true }
                        Key.Enter, Key.MediaPlayPause -> { onPlayPause(); true }
                        else -> false // 이벤트를 소비하지 않아 상위로 전파
                    }
                } else false
            }
    ) {
        // 플레이어 컨트롤 UI
        PlayerControlsBar(isPlaying = isPlaying)
    }
}
```

`Modifier.focusRestorer()`는 Compose 1.7+ 부터 공식 지원됩니다. 이 Modifier가 적용된 Composable 그룹에서 포커스가 빠져나갔다가 돌아오면, 내부적으로 `saveFocusedChild()` / `restoreFocusedChild()` 메커니즘을 사용해 마지막 포커스 위치를 자동으로 복원합니다.

---

## 4. 주의사항 및 팁

### 팁 1: TV Material vs Mobile Material — 절대 혼용 금지

```kotlin
// ❌ 혼용 금지: 동일한 이름이지만 완전히 다른 패키지
import androidx.compose.material3.MaterialTheme     // 스마트폰용
import androidx.tv.material3.MaterialTheme           // TV용

// ✅ TV 앱에서는 반드시 TV 버전만 사용
import androidx.tv.material3.MaterialTheme
import androidx.tv.material3.darkColorScheme
import androidx.tv.material3.ExperimentalTvMaterial3Api
```

두 `MaterialTheme`을 혼용하면 색상, 타이포그래피, Shape 스타일이 충돌하며 예측 불가능한 UI 버그가 발생합니다.

### 팁 2: TV 전용 레이아웃 Deprecated 마이그레이션

```kotlin
// ❌ Deprecated (androidx.tv.foundation)
TvLazyColumn { /* ... */ }
TvLazyRow(pivotOffsets = PivotOffsets(parentFraction = 0.3f)) { /* ... */ }

// ✅ 권장 (androidx.compose.foundation 1.7.0+)
LazyColumn { /* ... */ }
PositionFocusedItemInLazyLayout(parentFraction = 0.3f) {
    LazyRow { /* ... */ }
}
```

### 팁 3: NavigationDrawer로 사이드 메뉴 구현

```kotlin
@OptIn(ExperimentalTvMaterial3Api::class)
@Composable
fun TvAppWithSideNav() {
    NavigationDrawer(
        drawerContent = { drawerState ->
            val isOpen = drawerState == DrawerValue.Open
            Column(
                modifier = Modifier
                    .background(MaterialTheme.colorScheme.surface)
                    .fillMaxHeight()
                    .width(if (isOpen) 240.dp else 56.dp)
                    .padding(8.dp)
            ) {
                NavigationDrawerItem(
                    selected = true,
                    onClick = { /* 홈 */ },
                    leadingContent = { Icon(Icons.Default.Home, null) }
                ) {
                    if (isOpen) Text("홈")
                }
                NavigationDrawerItem(
                    selected = false,
                    onClick = { /* 검색 */ },
                    leadingContent = { Icon(Icons.Default.Search, null) }
                ) {
                    if (isOpen) Text("검색")
                }
            }
        }
    ) {
        // 메인 컨텐츠
        TvCatalogWithFocusRestoration()
    }
}
```

D-Pad 왼쪽 키를 누르면 드로어가 열리고, 오른쪽 키를 누르면 자동으로 닫히는 동작이 기본 제공됩니다.

### 팁 4: TV 앱 포커스 디버깅

포커스 이동이 예상과 다를 때 다음 도구를 활용하세요:

- **`Modifier.onFocusChanged { state -> Log.d("Focus", "hasFocus=${state.hasFocus}") }`**: 각 Composable의 포커스 변화 로그 출력
- **Android Studio Layout Inspector**: TV 앱에서도 Compose UI 계층 구조와 포커스 상태 시각화 가능
- **에뮬레이터 D-Pad**: Android TV 에뮬레이터에서 키보드 방향키로 D-Pad 시뮬레이션

### 팁 5: 10-Foot UI 디자인 원칙

TV 화면은 일반적으로 1080p(1920×1080) 이상이지만 거리가 멀기 때문에 텍스트와 터치 타겟을 크게 잡아야 합니다:

- **최소 폰트 크기**: 24sp 이상 권장
- **터치 타겟 (포커스 영역)**: 최소 48×48dp (모바일과 동일 기준이지만, TV에서는 더 크게)
- **외곽 여백**: 화면 좌우에 최소 48~96dp의 안전 마진
- **포커스 강조**: 포커스된 아이템은 최소 1.05배 이상 확대하여 명확히 구분

---

## 결론

Jetpack Compose for TV는 Leanback의 복잡한 Fragment 구조를 선언형 Compose로 완전히 대체합니다. 핵심 세 가지를 기억하세요:

1. **`androidx.tv:tv-material` 사용**: TV 특화 포커스 인디케이터, 스케일 애니메이션 자동 내장
2. **표준 LazyColumn/LazyRow 사용** (Foundation 1.7+): TV 전용 레이아웃은 deprecated
3. **`Modifier.focusRestorer()` + `PositionFocusedItemInLazyLayout`**: 자연스러운 포커스 복원과 스크롤 위치 제어

TV 앱은 스마트폰 앱보다 적은 사용자를 대상으로 하지만, 거실의 큰 화면에서 최고의 시청 경험을 제공하는 것은 여전히 중요한 도전입니다. Compose for TV는 그 도전을 훨씬 우아하게 만들어줍니다.

## 참고 자료
- [Compose for TV 공식 가이드](https://developer.android.com/training/tv/playback/compose)
- [TV 스크롤 레이아웃 및 BringIntoViewSpec](https://developer.android.com/training/tv/playback/compose/lists)
- [Compose 포커스 관리 공식 문서](https://developer.android.com/jetpack/compose/touch-input/focus)
