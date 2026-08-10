---
layout: post
title: "Jetpack Compose Pager 심화: HorizontalPager·VerticalPager·TabRow 완전 정복"
date: 2026-08-10
categories: [android, flutter]
tags: [android, jetpack-compose, pager, horizontalpager, verticalpager, tabrow, kotlin]
---

## 개념 설명

Jetpack Compose 1.4부터 공식적으로 안정화된 `HorizontalPager`와 `VerticalPager`는 사용자가 콘텐츠를 좌우 또는 상하로 스와이프하며 탐색할 수 있는 컴포저블입니다. 기존 View 시스템의 `ViewPager2`를 Compose 세계에서 대체하는 컴포넌트로, `androidx.compose.foundation` 라이브러리에 포함되어 있습니다. 현재 최신 안정 버전은 `compose-foundation 1.11.4`입니다.

### 핵심 구성 요소

- **`PagerState`**: 페이저의 상태를 관리하는 객체로, 현재 페이지·스크롤 오프셋·페이지 수 등의 정보를 제공합니다.
- **`rememberPagerState`**: `PagerState`를 Composition에서 기억하는 함수입니다.
- **`HorizontalPager`**: 수평 방향으로 페이지를 스와이프하는 컴포저블
- **`VerticalPager`**: 수직 방향으로 페이지를 스와이프하는 컴포저블

```kotlin
dependencies {
    implementation("androidx.compose.foundation:foundation:1.11.4")
}
```

`PagerState`의 주요 프로퍼티는 다음과 같습니다.

| 프로퍼티 | 설명 |
|---|---|
| `currentPage` | 스냅 위치에 가장 가까운 페이지 인덱스. 스크롤 중에도 즉시 업데이트 |
| `settledPage` | 스크롤/애니메이션이 완전히 멈춘 후의 페이지 인덱스 |
| `targetPage` | 현재 스크롤이 멈출 예정인 페이지 인덱스 |
| `currentPageOffsetFraction` | 현재 페이지로부터의 스크롤 비율 (-1.0 ~ 1.0) |

---

## 왜 필요한가

### ViewPager2와의 비교

기존 `ViewPager2`는 `FragmentStateAdapter`나 `RecyclerView.Adapter`를 구현해야 하는 복잡한 구조를 가졌습니다. Compose Pager는 이를 선언형 방식으로 대폭 단순화합니다.

**ViewPager2 방식 (구 방식):**

```kotlin
// Fragment 기반 어댑터 필수 구현
class MyPagerAdapter(fa: FragmentActivity) : FragmentStateAdapter(fa) {
    override fun getItemCount() = 5
    override fun createFragment(position: Int) = MyFragment.newInstance(position)
}
viewPager2.adapter = MyPagerAdapter(this)

// TabLayout 연동도 별도 작업 필요
TabLayoutMediator(tabLayout, viewPager2) { tab, position ->
    tab.text = "Tab ${position + 1}"
}.attach()
```

**Compose Pager 방식 (현재):**

```kotlin
val pagerState = rememberPagerState(pageCount = { 5 })
HorizontalPager(state = pagerState) { page ->
    MyPageContent(page)
}
```

Compose Pager의 장점은 다음과 같습니다.

- 보일러플레이트 코드 대폭 감소
- Fragment 생명주기와의 충돌이 없음
- `State`를 다른 Compose 컴포넌트와 자연스럽게 공유 가능
- 콘텐츠를 Lazy하게 렌더링하여 메모리 효율 확보
- `graphicsLayer` 등 Compose API와 직접 연동하여 풍부한 시각 효과 구현 가능

---

## 실제 구현 예제

### 예제 1: HorizontalPager + TabRow 완전 연동

가장 일반적인 패턴인 탭과 페이저의 양방향 동기화입니다. 탭을 클릭하면 페이저가 이동하고, 페이저를 스와이프하면 탭이 따라옵니다.

```kotlin
@Composable
fun TabbedPagerScreen() {
    val tabs = listOf("홈", "탐색", "즐겨찾기", "프로필")
    val pagerState = rememberPagerState(pageCount = { tabs.size })
    val coroutineScope = rememberCoroutineScope()

    Column(modifier = Modifier.fillMaxSize()) {
        TabRow(
            selectedTabIndex = pagerState.currentPage,
            indicator = { tabPositions ->
                // 인디케이터가 스와이프 제스처에 따라 부드럽게 이동
                if (pagerState.currentPage < tabPositions.size) {
                    TabRowDefaults.PrimaryIndicator(
                        modifier = Modifier.tabIndicatorOffset(
                            tabPositions[pagerState.currentPage]
                        )
                    )
                }
            }
        ) {
            tabs.forEachIndexed { index, title ->
                Tab(
                    selected = pagerState.currentPage == index,
                    onClick = {
                        // 탭 클릭 시 페이저가 애니메이션으로 이동
                        coroutineScope.launch {
                            pagerState.animateScrollToPage(index)
                        }
                    },
                    text = { Text(title) }
                )
            }
        }

        HorizontalPager(
            state = pagerState,
            modifier = Modifier.weight(1f)
        ) { page ->
            Box(
                modifier = Modifier
                    .fillMaxSize()
                    .background(
                        when (page) {
                            0 -> MaterialTheme.colorScheme.primaryContainer
                            1 -> MaterialTheme.colorScheme.secondaryContainer
                            2 -> MaterialTheme.colorScheme.tertiaryContainer
                            else -> MaterialTheme.colorScheme.surfaceVariant
                        }
                    ),
                contentAlignment = Alignment.Center
            ) {
                Text(
                    text = "${tabs[page]} 콘텐츠",
                    style = MaterialTheme.typography.headlineMedium
                )
            }
        }
    }
}
```

핵심 포인트:

- `pagerState.currentPage`를 `selectedTabIndex`로 전달하면 탭이 페이저와 자동 동기화됩니다.
- `animateScrollToPage()`는 suspend 함수이므로 반드시 `coroutineScope.launch` 블록 안에서 호출합니다.
- 더 정교한 탭 인디케이터 애니메이션을 원한다면 `pagerState.currentPageOffsetFraction`을 `Modifier.tabIndicatorOffset`의 커스텀 구현과 결합할 수 있습니다.

---

### 예제 2: 무한 스크롤 캐러셀 + 시각 효과 + 커스텀 인디케이터

`Int.MAX_VALUE` 트릭, `graphicsLayer` 스케일/알파 효과, 확장 애니메이션 인디케이터를 모두 조합한 고급 이미지 캐러셀입니다.

```kotlin
data class BannerItem(val id: String, val imageUrl: String, val title: String)

@Composable
fun InfiniteCarouselScreen(banners: List<BannerItem>) {
    if (banners.isEmpty()) return

    val actualCount = banners.size
    // 충분히 큰 가상 페이지 수로 양방향 무한 스크롤 구현
    val virtualCount = Int.MAX_VALUE
    // 중앙에서 시작하되, 실제 페이지 경계와 정렬
    val initialPage = virtualCount / 2 - (virtualCount / 2 % actualCount)

    val pagerState = rememberPagerState(
        initialPage = initialPage,
        pageCount = { virtualCount }
    )

    // 자동 재생
    LaunchedEffect(pagerState) {
        while (true) {
            delay(3_000L)
            pagerState.animateScrollToPage(pagerState.currentPage + 1)
        }
    }

    Column(
        modifier = Modifier.fillMaxWidth(),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        HorizontalPager(
            state = pagerState,
            // contentPadding으로 인접 페이지를 살짝 노출 (Carousel 효과)
            contentPadding = PaddingValues(horizontal = 40.dp),
            pageSpacing = 12.dp,
            modifier = Modifier
                .fillMaxWidth()
                .height(200.dp)
        ) { virtualPage ->
            val actualPage = virtualPage % actualCount
            val banner = banners[actualPage]

            Card(
                modifier = Modifier
                    .fillMaxSize()
                    .graphicsLayer {
                        // 스크롤 비율 계산 (현재 페이지: 0.0, 인접 페이지: 1.0에 수렴)
                        val pageOffset = (
                            (pagerState.currentPage - virtualPage) +
                            pagerState.currentPageOffsetFraction
                        ).absoluteValue

                        // 크기: 현재 페이지 100%, 인접 85%
                        val scale = lerp(
                            start = 0.85f,
                            stop = 1f,
                            fraction = 1f - pageOffset.coerceIn(0f, 1f)
                        )
                        scaleX = scale
                        scaleY = scale

                        // 투명도: 현재 페이지 완전 불투명, 인접 60%
                        alpha = lerp(
                            start = 0.6f,
                            stop = 1f,
                            fraction = 1f - pageOffset.coerceIn(0f, 1f)
                        )
                    },
                shape = RoundedCornerShape(16.dp),
                elevation = CardDefaults.cardElevation(defaultElevation = 6.dp)
            ) {
                Box(modifier = Modifier.fillMaxSize()) {
                    AsyncImage(
                        model = banner.imageUrl,
                        contentDescription = banner.title,
                        contentScale = ContentScale.Crop,
                        modifier = Modifier.fillMaxSize()
                    )
                    // 그라데이션 오버레이와 제목 텍스트
                    Box(
                        modifier = Modifier
                            .fillMaxWidth()
                            .align(Alignment.BottomCenter)
                            .background(
                                Brush.verticalGradient(
                                    colors = listOf(Color.Transparent, Color.Black.copy(alpha = 0.7f))
                                )
                            )
                            .padding(12.dp)
                    ) {
                        Text(
                            text = banner.title,
                            color = Color.White,
                            style = MaterialTheme.typography.titleMedium
                        )
                    }
                }
            }
        }

        Spacer(modifier = Modifier.height(12.dp))

        // 확장 애니메이션 도트 인디케이터
        val currentActualPage = pagerState.currentPage % actualCount
        Row(
            horizontalArrangement = Arrangement.spacedBy(6.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            repeat(actualCount) { index ->
                val isSelected = currentActualPage == index
                Box(
                    modifier = Modifier
                        .animateContentSize(animationSpec = spring(stiffness = Spring.StiffnessMediumLow))
                        .height(8.dp)
                        // 선택된 점은 가로로 확장
                        .width(if (isSelected) 24.dp else 8.dp)
                        .clip(CircleShape)
                        .background(
                            if (isSelected) MaterialTheme.colorScheme.primary
                            else MaterialTheme.colorScheme.outline.copy(alpha = 0.4f)
                        )
                )
            }
        }
    }
}
```

핵심 포인트:

1. **무한 스크롤**: `pageCount = { Int.MAX_VALUE }`로 설정하고 `virtualPage % actualCount`로 실제 페이지를 계산합니다. `initialPage`를 중간값으로 설정해 처음부터 양방향 스크롤이 가능합니다.
2. **`graphicsLayer`**: Composition 단계가 아닌 Drawing 단계에서 실행되므로 `currentPageOffsetFraction`을 여기서 읽는 것이 안전하고 성능상 권장됩니다.
3. **`lerp()`**: `androidx.compose.ui.util.lerp`를 임포트해 두 값 사이를 선형 보간합니다.
4. **`contentPadding`**: 좌우 padding을 주면 인접 페이지가 살짝 보이는 Carousel 느낌을 연출할 수 있습니다.
5. **자동 재생**: `LaunchedEffect`와 `delay()`를 조합해 타이머 기반 자동 슬라이드를 구현합니다.

---

## 주의사항 및 팁

### 1. `currentPageOffsetFraction`은 Composition에서 직접 읽지 말 것

Compose 1.11부터 이 프로퍼티는 `@FrequentlyChangingValue`로 마킹되었습니다. Composition 중에 직접 읽으면 스크롤 중 지속적인 리컴포지션이 발생해 성능이 저하됩니다.

```kotlin
// ❌ Composition 중 읽기 — 매 프레임 리컴포지션 유발
val offset = pagerState.currentPageOffsetFraction
Text(text = "offset: $offset")

// ✅ graphicsLayer 람다 내부에서 읽기 — Drawing 단계에서 실행, 리컴포지션 없음
Modifier.graphicsLayer {
    val offset = pagerState.currentPageOffsetFraction
    alpha = 1f - offset.absoluteValue
}
```

### 2. `beyondViewportPageCount`로 프리페치 범위 조절

기본적으로 Pager는 뷰포트 기준 인접 페이지를 미리 렌더링합니다. 콘텐츠의 무게에 따라 적절히 조절하세요.

```kotlin
HorizontalPager(
    state = pagerState,
    // 무거운 콘텐츠: 0으로 줄여 필요할 때만 렌더링
    // 빠른 스크롤 UX: 1~2로 늘려 미리 준비
    beyondViewportPageCount = 1
) { page ->
    HeavyContent(page)
}
```

### 3. 데이터가 동적으로 바뀐다면 `key` 파라미터 사용

`key`를 지정하면 Pager가 데이터 변경 시 올바른 페이지를 유지합니다. 지정하지 않으면 아이템이 추가/삭제될 때 잘못된 페이지가 표시될 수 있습니다.

```kotlin
HorizontalPager(
    state = pagerState,
    key = { items[it].id } // 데이터 고유 ID를 키로 사용
) { page ->
    ItemCard(items[page])
}
```

### 4. `settledPage`와 `currentPage` 올바른 구분

```kotlin
// 스크롤 중에도 즉시 반응해야 하는 탭 UI 동기화에는 currentPage
TabRow(selectedTabIndex = pagerState.currentPage) { ... }

// 스크롤이 완전히 완료된 후에만 실행해야 하는 API 호출, 분석 이벤트에는 settledPage
LaunchedEffect(pagerState.settledPage) {
    analytics.logPageView(pagerState.settledPage)
    viewModel.loadPageData(pagerState.settledPage)
}
```

### 5. 초기 페이지 복원과 상태 저장

딥링크나 프로세스 재시작 후 특정 페이지로 바로 이동해야 할 때는 `initialPage` 파라미터와 `scrollToPage()`(애니메이션 없음)를 조합합니다.

```kotlin
val savedPage = getSavedPage() // ViewModel이나 SavedStateHandle에서 복원

val pagerState = rememberPagerState(
    initialPage = savedPage, // Composition 첫 렌더링 시 페이지 설정
    pageCount = { items.size }
)

// 런타임 중 외부에서 페이지를 변경해야 하는 경우
LaunchedEffect(deepLinkPage) {
    if (deepLinkPage != null) {
        pagerState.scrollToPage(deepLinkPage) // 즉시 이동 (애니메이션 없음)
    }
}
```

### 6. VerticalPager 사용 시 주의사항

`VerticalPager`는 `HorizontalPager`와 API가 동일하지만, 내부적으로 수직 스크롤을 처리합니다. 스크롤 방향이 겹치는 `LazyColumn`이나 `NestedScrollConnection`과 함께 사용할 때는 스크롤 충돌에 주의하세요.

```kotlin
val pagerState = rememberPagerState(pageCount = { 10 })

VerticalPager(
    state = pagerState,
    modifier = Modifier.fillMaxSize()
) { page ->
    // 내부에 수직 스크롤 컴포넌트가 있으면 스크롤 충돌 발생 가능
    // nestedScroll API로 우선순위를 명시적으로 지정해야 함
    Box(modifier = Modifier.fillMaxSize()) {
        Text("Page $page", modifier = Modifier.align(Alignment.Center))
    }
}
```

---

## 참고 자료

- [Pager in Compose — Android Developers](https://developer.android.com/develop/ui/compose/layouts/pager)
- [androidx.compose.foundation.pager API Reference](https://developer.android.com/reference/kotlin/androidx/compose/foundation/pager/package-summary)
- [Compose Foundation Release Notes](https://developer.android.com/jetpack/androidx/releases/compose-foundation)
