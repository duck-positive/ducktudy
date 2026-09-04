---
layout: post
title: "Jetpack Compose LazyLayout 저수준 API 심화: 커스텀 레이지 컴포넌트 직접 만들기"
date: 2026-09-04
categories: [android, flutter]
tags: [android, jetpack-compose, lazylayout, custom-layout, kotlin, performance, virtualization]
---

Jetpack Compose에서 우리가 매일 사용하는 `LazyColumn`, `LazyRow`, `LazyVerticalGrid`, `LazyVerticalStaggeredGrid`는 모두 `LazyLayout`이라는 저수준(low-level) 컴포저블 위에 구축되어 있습니다. `LazyLayout`은 `androidx.compose.foundation.lazy.layout` 패키지에 포함된 핵심 프리미티브로, 뷰포트에 보이는 아이템만 컴포즈(compose)하고 레이아웃하는 가상화(virtualization) 메커니즘을 제공합니다.

일반적으로 개발자가 `LazyLayout`을 직접 사용할 일은 많지 않습니다. 그러나 표준 레이지 컴포넌트가 제공하지 않는 독특한 레이아웃이 필요할 때—예컨대 방사형 리스트, 무한 캐러셀, 또는 특수한 스냅 동작을 가진 원통형 피커—`LazyLayout` API를 직접 다루면 성능을 희생하지 않고도 완전히 커스텀된 레이지 컴포넌트를 만들 수 있습니다.

## LazyLayout의 구성 요소

`LazyLayout`은 세 가지 핵심 개념을 중심으로 설계되어 있습니다.

**LazyLayoutItemProvider**는 전체 아이템 데이터를 관리하는 인터페이스입니다. 총 아이템 수(`itemCount`), 각 아이템의 컴포저블(`@Composable fun Item(index: Int, key: Any)`), 그리고 아이템의 키(`getKey`)와 콘텐츠 타입(`getContentType`)을 제공합니다. 중요한 점은 `itemProvider`가 전체 데이터에 대한 메타정보를 갖지만, 실제 컴포즈는 레이아웃 단계에서 `measure(index, constraints)`를 호출할 때만 발생한다는 것입니다.

**LazyLayout 컴포저블** 자체는 `MeasurePolicy`를 통해 어떤 아이템을 언제 측정하고 배치할지 결정합니다. `MeasureScope.measure(index, constraints)`를 호출하면 해당 인덱스의 아이템이 필요 시점에 컴포즈됩니다. 이것이 가상화의 핵심입니다.

**LazyLayoutPrefetchState**는 사용자가 스크롤하기 전에 미리 아이템을 컴포즈해두는 프리패칭(prefetching) 전략을 관리합니다. 표준 `LazyColumn`이 부드러운 스크롤을 제공하는 비결이 바로 이 프리패치 메커니즘입니다.

## 왜 필요한가

`LazyColumn`과 `LazyRow`는 각각 수직/수평 단일 축 스크롤만 지원합니다. `LazyVerticalGrid`는 균일한 열 너비를 가정하며, `LazyVerticalStaggeredGrid`도 열 기반 배치에 한정됩니다. 다음과 같은 시나리오에서는 `LazyLayout`을 직접 사용해야 합니다:

- **원통형 드럼 피커(Drum Picker)**: 아이템이 원통 표면에 배치된 것처럼 회전하는 iOS 스타일 피커
- **방사형 메뉴**: 중앙에서 원형으로 퍼져나가는 레이지 아이템 목록
- **2D 무한 스크롤 캔버스**: 가로/세로 동시 스크롤이 가능한 가상화된 무한 그리드
- **커스텀 스냅 동작**: `LazyListState`의 정해진 스냅 동작 대신 완전히 커스텀한 물리 기반 스냅 로직
- **비균일 축 레이아웃**: 아이템 위치가 인덱스 외에 다른 데이터(예: 타임라인의 시각)에 의해 결정되는 경우

성능 관점에서도 중요합니다. 10,000개 아이템 목록을 `Column` + `verticalScroll`로 렌더링하면 모든 아이템이 즉시 컴포즈되어 메모리를 과도하게 소비하지만, `LazyLayout` 기반 구현은 화면에 보이는 10~20개 아이템만 실제로 컴포즈합니다.

## 실제 구현 예제

### 예제 1: 기본 커스텀 LazyLayout — 수직 레이지 리스트

가장 단순한 형태의 `LazyLayout` 구현부터 시작하겠습니다. `LazyColumn`과 유사하게 동작하는 커스텀 수직 레이지 리스트를 만들면서 내부 구조를 이해합니다.

```kotlin
import androidx.compose.foundation.gestures.Orientation
import androidx.compose.foundation.gestures.scrollable
import androidx.compose.foundation.gestures.rememberScrollableState
import androidx.compose.foundation.lazy.layout.LazyLayout
import androidx.compose.foundation.lazy.layout.LazyLayoutItemProvider
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.Constraints
import kotlin.math.roundToInt

// 아이템 제공자: 전체 데이터셋을 보유하지만 실제 컴포즈는 요청 시에만 발생
@Stable
class SimpleListItemProvider(
    private val items: List<@Composable () -> Unit>
) : LazyLayoutItemProvider {
    override val itemCount: Int get() = items.size

    @Composable
    override fun Item(index: Int, key: Any) {
        items[index]()
    }

    // 안정적인 키 없이 인덱스를 키로 사용 (실제 코드에서는 고유 ID 권장)
    override fun getKey(index: Int): Any = index
}

@Composable
fun CustomLazyList(
    modifier: Modifier = Modifier,
    items: List<@Composable () -> Unit>
) {
    var scrollOffset by remember { mutableFloatStateOf(0f) }

    val itemProvider = remember(items) {
        SimpleListItemProvider(items)
    }

    val scrollableState = rememberScrollableState { delta ->
        scrollOffset = (scrollOffset - delta).coerceAtLeast(0f)
        delta
    }

    LazyLayout(
        itemProvider = { itemProvider },
        modifier = modifier.scrollable(
            state = scrollableState,
            orientation = Orientation.Vertical
        )
    ) { constraints ->
        val itemConstraints = Constraints(
            maxWidth = constraints.maxWidth,
            maxHeight = Constraints.Infinity
        )

        // 뷰포트 내 보이는 아이템만 측정
        var currentY = -scrollOffset.roundToInt()
        val placeables = mutableListOf<Pair<androidx.compose.ui.layout.Placeable, Int>>()

        for (index in 0 until itemProvider.itemCount) {
            if (currentY > constraints.maxHeight) break

            // 이미 스크롤로 가려진 아이템은 대략적인 높이로 건너뜀
            if (currentY + 56 > 0) {
                val measurables = measure(index, itemConstraints)
                measurables.forEach { placeable ->
                    placeables.add(placeable to currentY)
                    currentY += placeable.height
                }
            } else {
                currentY += 56 // 평균 아이템 높이 추정값(dp → px은 LocalDensity로 변환 권장)
            }
        }

        layout(constraints.maxWidth, constraints.maxHeight) {
            placeables.forEach { (placeable, y) ->
                placeable.place(0, y)
            }
        }
    }
}

// 사용 예시
@Composable
fun CustomLazyListDemo() {
    val data = remember { (1..1000).map { "아이템 #$it" } }

    CustomLazyList(
        modifier = Modifier.fillMaxSize()
    ) {
        data.map { text ->
            {
                Text(
                    text = text,
                    modifier = Modifier
                        .fillMaxWidth()
                        .padding(horizontal = 16.dp, vertical = 12.dp)
                )
            }
        }
    }
}
```

위 예제에서 핵심은 `measure(index, constraints)` 호출입니다. 이 함수는 `LazyLayout`의 `MeasureScope` 확장 함수로, 특정 인덱스의 아이템을 요청 시에만 컴포즈하고 측정합니다. `for` 루프가 1,000개 아이템을 순회하더라도 실제로 `measure`를 호출한 아이템만 컴포즈됩니다.

### 예제 2: 원통형 드럼 피커 (Drum Picker)

이제 `LazyLayout`이 진정한 강점을 발휘하는 사례를 구현합니다. iOS의 UIPickerView처럼 아이템이 원통 표면에 배치되어 회전하는 드럼 피커입니다. 이 레이아웃은 표준 `LazyColumn`으로는 구현하기 어렵습니다—아이템의 Y 위치가 단순한 순차 배치가 아니라 사인 함수에 의해 비선형 변환되기 때문입니다.

```kotlin
import androidx.compose.animation.core.animateFloatAsState
import androidx.compose.animation.core.spring
import androidx.compose.foundation.gestures.Orientation
import androidx.compose.foundation.gestures.draggable
import androidx.compose.foundation.gestures.rememberDraggableState
import androidx.compose.foundation.lazy.layout.LazyLayout
import androidx.compose.foundation.lazy.layout.LazyLayoutItemProvider
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.Constraints
import kotlin.math.*

private const val ITEM_HEIGHT_PX = 160 // 각 아이템 높이 (픽셀)
private const val VISIBLE_RADIUS = 3   // 중앙에서 몇 개 아이템까지 보여줄지

@Stable
class DrumPickerItemProvider(
    override val itemCount: Int,
    private val content: @Composable (index: Int) -> Unit
) : LazyLayoutItemProvider {
    @Composable
    override fun Item(index: Int, key: Any) = content(index)

    override fun getKey(index: Int): Any = index
    override fun getContentType(index: Int): Any? = "drum-item"
}

@Composable
fun DrumPicker(
    items: List<String>,
    selectedIndex: Int,
    onIndexSelected: (Int) -> Unit,
    modifier: Modifier = Modifier
) {
    var dragOffset by remember { mutableFloatStateOf(0f) }

    // 드래그 종료 시 스냅될 목표 오프셋
    val targetOffset by animateFloatAsState(
        targetValue = -selectedIndex * ITEM_HEIGHT_PX.toFloat() + dragOffset,
        animationSpec = spring(dampingRatio = 0.7f, stiffness = 300f),
        label = "drumOffset"
    )

    val provider = remember(items) {
        DrumPickerItemProvider(items.size) { index ->
            Box(
                modifier = Modifier.fillMaxSize(),
                contentAlignment = Alignment.Center
            ) {
                Text(
                    text = items[index],
                    style = MaterialTheme.typography.headlineMedium.copy(
                        color = if (index == selectedIndex)
                            MaterialTheme.colorScheme.primary
                        else
                            MaterialTheme.colorScheme.onSurface.copy(alpha = 0.4f)
                    )
                )
            }
        }
    }

    val draggableState = rememberDraggableState { delta ->
        dragOffset += delta
    }

    LazyLayout(
        itemProvider = { provider },
        modifier = modifier.draggable(
            state = draggableState,
            orientation = Orientation.Vertical,
            onDragStopped = {
                // 가장 가까운 아이템으로 스냅
                val rawIndex = (-(targetOffset) / ITEM_HEIGHT_PX).roundToInt()
                onIndexSelected(rawIndex.coerceIn(0, items.lastIndex))
                dragOffset = 0f
            }
        )
    ) { constraints ->
        val centerY = constraints.maxHeight / 2

        // 화면 중앙 기준으로 현재 인덱스 계산
        val centerIndex = (-(targetOffset) / ITEM_HEIGHT_PX).roundToInt()
            .coerceIn(0, items.lastIndex)

        val firstVisible = (centerIndex - VISIBLE_RADIUS).coerceAtLeast(0)
        val lastVisible  = (centerIndex + VISIBLE_RADIUS).coerceAtMost(items.lastIndex)

        val itemConstraints = Constraints.fixed(
            width = constraints.maxWidth,
            height = ITEM_HEIGHT_PX
        )

        data class PlacedItem(
            val placeable: androidx.compose.ui.layout.Placeable,
            val index: Int
        )

        val placedItems = (firstVisible..lastVisible).flatMap { index ->
            measure(index, itemConstraints).map { PlacedItem(it, index) }
        }

        layout(constraints.maxWidth, constraints.maxHeight) {
            placedItems.forEach { (placeable, index) ->
                // 원통 표면 시뮬레이션: 선형 Y에서 사인 기반 Y로 변환
                val linearY = centerY + index * ITEM_HEIGHT_PX + targetOffset.roundToInt()
                val distanceFromCenter = (linearY - centerY).toFloat()
                val normalizedDist = (distanceFromCenter / (VISIBLE_RADIUS * ITEM_HEIGHT_PX))
                    .coerceIn(-1f, 1f)

                // 원통 곡률: 멀수록 세로 방향으로 압축되는 효과
                val cylinderY = centerY + (
                    sin(normalizedDist * PI / 2).toFloat() *
                    VISIBLE_RADIUS * ITEM_HEIGHT_PX
                ).roundToInt()

                placeable.place(0, cylinderY - ITEM_HEIGHT_PX / 2)
            }
        }
    }
}

// 사용 예시: 시/분 시간 선택기
@Composable
fun TimePicker() {
    val hours   = (0..23).map { it.toString().padStart(2, '0') }
    val minutes = (0..59).map { it.toString().padStart(2, '0') }

    var selectedHour   by remember { mutableIntStateOf(0) }
    var selectedMinute by remember { mutableIntStateOf(0) }

    Row(
        modifier = Modifier.fillMaxWidth(),
        horizontalArrangement = Arrangement.Center,
        verticalAlignment = Alignment.CenterVertically
    ) {
        DrumPicker(
            items = hours,
            selectedIndex = selectedHour,
            onIndexSelected = { selectedHour = it },
            modifier = Modifier.width(80.dp).height(280.dp)
        )
        Text(":", style = MaterialTheme.typography.headlineLarge)
        DrumPicker(
            items = minutes,
            selectedIndex = selectedMinute,
            onIndexSelected = { selectedMinute = it },
            modifier = Modifier.width(80.dp).height(280.dp)
        )
    }
}
```

이 예제에서 원통 효과의 핵심은 `sin(normalizedDist * PI / 2)` 변환입니다. 아이템의 선형 Y 위치(`linearY`)를 사인 함수로 비선형 변환하면 중앙에 가까운 아이템은 본래 위치에 가깝게, 멀리 있는 아이템은 중심 쪽으로 당겨지는 원통 표면 효과가 만들어집니다. `LazyLayout`이 없었다면 이 비선형 배치를 가상화하는 것이 매우 복잡했을 것입니다.

## 주의사항 및 팁

### 1. measure는 반드시 MeasureScope 내부에서만 호출

`measure(index, constraints)`는 `LazyLayout`의 `measurePolicy` 람다 내부에서만 유효합니다. 이 함수를 이벤트 핸들러나 `LaunchedEffect` 같은 컴포즈 외부 컨텍스트에서 호출하려 해선 안 됩니다. 아이템 컴포즈는 항상 레이아웃 단계(layout pass)에 동기적으로 이루어집니다.

### 2. 전체 아이템에 measure를 호출하지 말 것

`LazyLayoutItemProvider.itemCount`는 전체 데이터셋 크기를 나타내지만, 실제로 `measure`를 호출하는 인덱스 범위는 반드시 뷰포트에 보이는 것만으로 한정해야 합니다. 전체 아이템에 `measure`를 호출하면 `LazyLayout`의 가상화 이점이 완전히 사라져 `Column`과 성능 차이가 없어집니다.

### 3. 아이템 키(key)로 상태 안정성 확보

`LazyLayoutItemProvider.getKey(index)`를 반드시 구현하세요. 키가 없으면 아이템 재정렬 시 컴포즈 상태(텍스트 필드 내용, 애니메이션 진행 상황 등)가 잘못된 아이템에 붙을 수 있습니다.

```kotlin
// 나쁜 예: 인덱스를 키로 사용 (재정렬 시 상태 꼬임 발생)
override fun getKey(index: Int): Any = index

// 좋은 예: 데이터의 안정적인 고유 ID 사용
override fun getKey(index: Int): Any = dataList[index].id
```

### 4. contentType으로 재사용 성능 최적화

아이템 종류가 여럿일 때는 `getContentType(index)`를 구현해 동일 타입 아이템 간 컴포즈 재사용(reuse)이 일어나도록 하세요. 예를 들어 헤더와 일반 아이템이 혼재할 경우:

```kotlin
override fun getContentType(index: Int): Any? = when {
    dataList[index].isHeader -> "header"
    dataList[index].isAd     -> "ad"
    else                     -> "content"
}
```

타입이 일치하는 아이템끼리는 내부 컴포즈 트리를 재활용하므로 스크롤 성능이 향상됩니다.

### 5. remember로 itemProvider 안정화

`LazyLayout`은 매 리컴포즈마다 `itemProvider` 람다를 호출합니다. 이 람다가 매번 새로운 객체를 반환하면 아이템 상태가 초기화될 수 있으므로 반드시 `remember`로 감싸야 합니다.

```kotlin
// 올바른 방법: 데이터가 변경될 때만 새 provider 생성
val provider = remember(dataList) {
    MyItemProvider(dataList)
}

LazyLayout(itemProvider = { provider }) { ... }
```

### 6. LazyLayoutPrefetchState 연동

고속 스크롤 중 프레임 드롭을 방지하려면 `LazyLayoutPrefetchState`와 `NestedScrollConnection`을 함께 사용해 스크롤 방향 및 속도에 따라 다음 아이템을 미리 컴포즈해두는 전략을 구현하는 것이 좋습니다. 표준 `LazyColumn`은 이 전략을 내부적으로 사용합니다.

## 정리

`LazyLayout`은 Jetpack Compose의 레이지 컴포넌트 생태계 최하단에 위치한 강력한 빌딩 블록입니다. 표준 `LazyColumn`/`LazyRow`가 제공하는 직선형 레이아웃이나 `LazyVerticalGrid`의 격자형 배치를 넘어서는 독창적인 UI가 필요할 때, `LazyLayout` API를 직접 활용하면 가상화 성능을 유지하면서도 완전히 자유로운 레이아웃을 구현할 수 있습니다.

핵심 원칙을 정리하면:

1. `LazyLayoutItemProvider`로 데이터 선언과 컴포즈를 분리한다
2. `measurePolicy` 내에서 뷰포트 가시성을 기준으로 `measure` 호출 범위를 최소화한다
3. `getKey`와 `getContentType`을 올바르게 구현해 Compose 런타임의 재사용 최적화를 활용한다
4. `remember`로 `itemProvider`를 안정화해 불필요한 상태 초기화를 방지한다

`LazyLayout` API는 현재 `@ExperimentalFoundationApi`로 표시되어 향후 API 변경 가능성이 있습니다. 프로덕션 코드에서 사용할 때는 이 점을 염두에 두고, 표준 컴포넌트로 대체할 수 없는 케이스에만 선별적으로 적용하는 것을 권장합니다.

## 참고 자료
- [Jetpack Compose Lazy Lists and Grids 공식 문서](https://developer.android.com/jetpack/compose/lists)
- [androidx.compose.foundation.lazy.layout API Reference](https://developer.android.com/reference/kotlin/androidx/compose/foundation/lazy/layout/package-summary)
