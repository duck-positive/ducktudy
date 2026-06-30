---
layout: post
title: "Jetpack Compose Custom Layout 심화: Layout API, SubcomposeLayout, Intrinsic Measurement 완전 정복"
date: 2026-06-30
categories: [android, flutter]
tags: [jetpack-compose, custom-layout, layout-api, subcomposelayout, intrinsic-measurement, kotlin, android]
---

## 개요

Jetpack Compose는 `Row`, `Column`, `Box` 등 다양한 표준 레이아웃을 제공하지만, 실무에서는 이 컴포저블만으로는 표현할 수 없는 복잡한 UI를 구현해야 하는 경우가 생깁니다. 예를 들어, 스태거드 그리드(Staggered Grid), 방사형(Radial) 메뉴, 혹은 자식 컴포저블의 크기를 먼저 측정한 뒤 그 값을 기반으로 다른 자식을 구성해야 하는 경우가 있습니다.

이때 필요한 것이 바로 **커스텀 레이아웃(Custom Layout)** 입니다. Compose는 `Layout` 컴포저블, `Modifier.layout`, `SubcomposeLayout` 등 저수준 레이아웃 API를 제공하며, 이를 통해 측정(Measurement)과 배치(Placement) 단계를 완전히 제어할 수 있습니다.

이 아티클에서는 Compose의 레이아웃 프로토콜부터 시작해, 커스텀 레이아웃 구현 방법, 그리고 고급 기능인 `SubcomposeLayout`과 Intrinsic Measurement까지 단계별로 살펴보겠습니다.

---

## 왜 커스텀 레이아웃이 필요한가?

### 1. 표준 레이아웃으로 불가능한 UI

`Row`, `Column`, `Box` 조합으로 대부분의 UI를 표현할 수 있지만, 다음 시나리오에서는 커스텀 레이아웃이 필요합니다.

- **Staggered Grid**: 각 아이템의 높이가 다른 Pinterest 스타일 그리드
- **Radial Layout**: 자식들을 원형으로 배치하는 레이아웃
- **Aspect-ratio-aware Layout**: 특정 비율을 유지하면서 자식을 채우는 레이아웃
- **SubcomposeLayout이 필요한 경우**: 한 자식의 측정 결과를 다른 자식의 컴포지션에 사용해야 하는 경우 (`LazyColumn`, `BoxWithConstraints` 내부가 이 방식으로 구현되어 있습니다)

### 2. 성능 최적화

Compose는 **단일 패스 측정(Single-pass Measurement)** 을 원칙으로 합니다. 구 View 시스템에서 `RelativeLayout` 같은 레이아웃은 자식을 두 번 이상 측정해 성능 문제가 발생하곤 했습니다. Compose는 이를 구조적으로 금지함으로써 O(n) 시간 복잡도의 레이아웃 패스를 보장합니다. 커스텀 레이아웃을 구현할 때도 이 원칙을 반드시 지켜야 합니다.

---

## Compose 측정·배치 프로토콜

Compose의 레이아웃 시스템은 세 단계로 동작합니다.

```
컴포지션(Composition) → 레이아웃(Layout) → 드로잉(Drawing)
```

레이아웃 단계는 두 하위 단계로 나뉩니다.

1. **측정(Measurement)**: 부모가 자식에게 `Constraints`(최소/최대 너비·높이 범위)를 전달합니다. 자식은 이 범위 안에서 자신의 크기를 결정하고 부모에게 보고합니다.
2. **배치(Placement)**: 부모가 자식의 위치(x, y)를 결정하고 `placeRelative` 또는 `place`로 배치합니다.

`Constraints`는 다음 4개의 값을 가집니다.

```kotlin
// Constraints는 부모가 자식에게 허용하는 크기 범위를 정의
data class Constraints(
    val minWidth: Int = 0,
    val maxWidth: Int = Constraints.Infinity,
    val minHeight: Int = 0,
    val maxHeight: Int = Constraints.Infinity
)
```

`Constraints.Infinity`는 제한 없음을 의미하며, 스크롤 컨테이너 내부에서 자주 사용됩니다. 예를 들어 `LazyColumn` 안의 자식들은 `maxHeight = Infinity` 조건을 받아 원하는 높이로 자유롭게 늘어날 수 있습니다.

---

## 실제 구현 예제 1: Layout 컴포저블로 스태거드 그리드 만들기

아래 예제는 자식을 두 열로 나누어 배치하되, 각 아이템의 높이가 달라도 올바르게 쌓이는 **스태거드 컬럼 레이아웃**을 구현합니다. Pinterest 카드 피드처럼 각 열에서 다음 아이템이 들어갈 위치(가장 짧은 열)를 추적하여 균형 잡힌 배치를 만들어냅니다.

```kotlin
@Composable
fun StaggeredVerticalLayout(
    modifier: Modifier = Modifier,
    columnCount: Int = 2,
    horizontalSpacing: Dp = 8.dp,
    verticalSpacing: Dp = 8.dp,
    content: @Composable () -> Unit
) {
    val hSpacingPx = with(LocalDensity.current) { horizontalSpacing.roundToPx() }
    val vSpacingPx = with(LocalDensity.current) { verticalSpacing.roundToPx() }

    Layout(
        modifier = modifier,
        content = content
    ) { measurables, constraints ->
        // 각 열의 너비 계산 (수평 간격 제외)
        val columnWidth = (constraints.maxWidth - hSpacingPx * (columnCount - 1)) / columnCount
        val itemConstraints = constraints.copy(
            minWidth = columnWidth,
            maxWidth = columnWidth
        )

        // 각 자식을 한 번씩만 측정 (Single-pass 원칙)
        val placeables = measurables.map { it.measure(itemConstraints) }

        // 각 열의 현재 Y 오프셋 추적
        val columnHeights = IntArray(columnCount) { 0 }

        // 가장 짧은 열에 아이템을 할당해 배치 좌표 계산
        val positions = placeables.map { placeable ->
            val column = columnHeights.indexOfMin()
            val x = column * (columnWidth + hSpacingPx)
            val y = columnHeights[column]
            columnHeights[column] += placeable.height + vSpacingPx
            x to y
        }

        val totalHeight = (columnHeights.max() - vSpacingPx)
            .coerceAtLeast(constraints.minHeight)

        layout(constraints.maxWidth, totalHeight) {
            placeables.forEachIndexed { index, placeable ->
                val (x, y) = positions[index]
                placeable.placeRelative(x, y)
            }
        }
    }
}

// 가장 낮은 값의 인덱스를 반환하는 확장 함수
private fun IntArray.indexOfMin(): Int {
    var minIdx = 0
    for (i in 1 until size) {
        if (this[i] < this[minIdx]) minIdx = i
    }
    return minIdx
}
```

### 사용 예시

```kotlin
@Composable
fun StaggeredDemo() {
    val colors = listOf(
        Color(0xFFEF9A9A), Color(0xFF80CBC4), Color(0xFFFFCC80),
        Color(0xFFCE93D8), Color(0xFF80DEEA), Color(0xFFA5D6A7)
    )
    val heights = listOf(120.dp, 80.dp, 160.dp, 100.dp, 90.dp, 140.dp)

    StaggeredVerticalLayout(
        modifier = Modifier
            .fillMaxWidth()
            .padding(16.dp),
        columnCount = 2,
        horizontalSpacing = 12.dp,
        verticalSpacing = 12.dp
    ) {
        heights.forEachIndexed { index, height ->
            Box(
                modifier = Modifier
                    .fillMaxWidth()
                    .height(height)
                    .clip(RoundedCornerShape(12.dp))
                    .background(colors[index % colors.size]),
                contentAlignment = Alignment.Center
            ) {
                Text(
                    text = "Item $index",
                    style = MaterialTheme.typography.bodyMedium,
                    fontWeight = FontWeight.Bold
                )
            }
        }
    }
}
```

---

## 실제 구현 예제 2: SubcomposeLayout으로 헤더 높이 기반 바디 구성

`SubcomposeLayout`은 **한 자식의 측정 결과**를 **다른 자식의 컴포지션**에 사용해야 할 때 필요합니다. 표준 `Layout`은 컴포지션이 완료된 후 측정이 이루어지므로 "측정 결과 → 컴포지션"의 순환 참조가 불가능합니다.

`SubcomposeLayout`은 측정 단계에서 새로운 서브컴포지션(subcompose)을 트리거할 수 있도록 허용합니다. `LazyColumn`, `BoxWithConstraints`, `BottomSheetScaffold` 등 복잡한 표준 컴포저블이 내부적으로 이 API를 사용합니다.

아래 예제는 고정 헤더의 높이를 측정한 후, 그 높이 값을 바디 컴포저블에 전달해 올바른 상단 패딩을 적용하는 레이아웃입니다.

```kotlin
@Composable
fun HeaderAwareLayout(
    modifier: Modifier = Modifier,
    header: @Composable () -> Unit,
    body: @Composable (topPadding: Dp) -> Unit
) {
    SubcomposeLayout(modifier = modifier) { constraints ->
        // Step 1: 헤더를 서브컴포즈하고 높이 측정
        val headerPlaceables = subcompose(slotId = "header", content = header)
            .map { it.measure(constraints.copy(minHeight = 0)) }

        val headerHeight = headerPlaceables.maxByOrNull { it.height }?.height ?: 0
        // 픽셀 → Dp 변환 (density는 SubcomposeLayout 스코프에서 제공)
        val headerHeightDp = (headerHeight / density).dp

        // Step 2: 헤더 높이를 바디에 전달하여 서브컴포즈
        val bodyConstraints = constraints.copy(
            minHeight = 0,
            maxHeight = (constraints.maxHeight - headerHeight).coerceAtLeast(0)
        )
        val bodyPlaceables = subcompose(slotId = "body") {
            body(topPadding = headerHeightDp)
        }.map { it.measure(bodyConstraints) }

        layout(constraints.maxWidth, constraints.maxHeight) {
            headerPlaceables.forEach { it.placeRelative(0, 0) }
            bodyPlaceables.forEach { it.placeRelative(0, headerHeight) }
        }
    }
}
```

### 사용 예시

```kotlin
@Composable
fun HeaderAwareDemo() {
    HeaderAwareLayout(
        modifier = Modifier.fillMaxSize(),
        header = {
            Surface(
                color = MaterialTheme.colorScheme.primaryContainer,
                shadowElevation = 4.dp
            ) {
                Column(
                    modifier = Modifier
                        .fillMaxWidth()
                        .padding(horizontal = 16.dp, vertical = 12.dp)
                ) {
                    Text(
                        text = "동적 헤더",
                        style = MaterialTheme.typography.headlineSmall,
                        fontWeight = FontWeight.Bold
                    )
                    Spacer(modifier = Modifier.height(4.dp))
                    Text(
                        text = "이 헤더의 높이에 따라 바디가 자동으로 조정됩니다.",
                        style = MaterialTheme.typography.bodyMedium
                    )
                }
            }
        },
        body = { topPadding ->
            LazyColumn(
                contentPadding = PaddingValues(
                    top = topPadding + 8.dp,
                    start = 16.dp,
                    end = 16.dp,
                    bottom = 16.dp
                ),
                verticalArrangement = Arrangement.spacedBy(8.dp)
            ) {
                items(30) { index ->
                    Card(modifier = Modifier.fillMaxWidth()) {
                        Text(
                            text = "리스트 아이템 #$index",
                            modifier = Modifier.padding(16.dp),
                            style = MaterialTheme.typography.bodyLarge
                        )
                    }
                }
            }
        }
    )
}
```

---

## Modifier.layout: 단일 컴포저블 변환

전체 레이아웃이 아닌 **단일 컴포저블의 측정·배치만 수정**하고 싶다면 `Modifier.layout`을 사용합니다. 이는 `Layout` 컴포저블보다 가볍고, 기존 Modifier 체인에 끼워 넣기 쉽습니다.

아래 예제는 `Text`의 상단 패딩을 단순 픽셀 오프셋이 아닌 **폰트 첫 번째 베이스라인 기준**으로 조정하는 커스텀 Modifier입니다. 디자이너가 텍스트 상단의 여백을 "베이스라인 기준"으로 지정할 때 활용합니다.

```kotlin
fun Modifier.firstBaselineToTop(firstBaselineToTop: Dp): Modifier = layout { measurable, constraints ->
    val placeable = measurable.measure(constraints)

    // 첫 번째 베이스라인 위치 조회 (폰트에 따라 다름)
    check(placeable[FirstBaseline] != AlignmentLine.Unspecified) {
        "FirstBaseline이 정의되지 않은 컴포저블에는 적용할 수 없습니다."
    }
    val firstBaseline = placeable[FirstBaseline]

    // 원하는 베이스라인 위치까지 y 오프셋 계산
    val placeableY = firstBaselineToTop.roundToPx() - firstBaseline
    val height = placeable.height + placeableY

    layout(placeable.width, height) {
        placeable.placeRelative(0, placeableY)
    }
}

// 사용
Text(
    text = "베이스라인 기준 패딩",
    modifier = Modifier.firstBaselineToTop(32.dp)
)
```

---

## Intrinsic Measurement: 다중 패스 없이 고유 크기 활용하기

Compose는 다중 패스 측정을 금지하지만, **Intrinsic Measurement(고유 크기 측정)** 를 통해 자식의 이상적인 크기 정보를 측정 전에 질의할 수 있습니다.

가장 흔한 사용 예는 `Row` 안에서 `Divider`를 자식들 중 가장 큰 높이에 맞추는 경우입니다. `IntrinsicSize.Max`를 `height` 모디파이어로 지정하면 `Row`는 자식들의 최대 고유 높이를 구해 자신의 높이로 사용합니다.

```kotlin
@Composable
fun TwoTextsWithDivider() {
    // IntrinsicSize.Max: 자식 중 가장 큰 고유 높이를 Row의 높이로 사용
    Row(modifier = Modifier.height(IntrinsicSize.Max)) {
        Text(
            modifier = Modifier
                .weight(1f)
                .wrapContentWidth(Alignment.Start),
            text = "왼쪽\n텍스트\n멀티라인"
        )
        // fillMaxHeight()로 Row의 전체 높이를 채움
        Divider(
            color = MaterialTheme.colorScheme.outline,
            modifier = Modifier
                .fillMaxHeight()
                .width(1.dp)
        )
        Text(
            modifier = Modifier
                .weight(1f)
                .wrapContentWidth(Alignment.End),
            text = "오른쪽"
        )
    }
}
```

커스텀 레이아웃에서 Intrinsic을 직접 구현할 때는 `MeasurePolicy`의 `minIntrinsicWidth` / `maxIntrinsicHeight` 등을 오버라이드합니다.

```kotlin
Layout(
    content = content,
    modifier = modifier,
    measurePolicy = object : MeasurePolicy {
        override fun MeasureScope.measure(
            measurables: List<Measurable>,
            constraints: Constraints
        ): MeasureResult {
            val placeables = measurables.map { it.measure(constraints) }
            val width = placeables.maxOfOrNull { it.width } ?: constraints.minWidth
            val height = placeables.sumOf { it.height }.coerceIn(
                constraints.minHeight,
                constraints.maxHeight
            )
            return layout(width, height) {
                var yOffset = 0
                placeables.forEach { placeable ->
                    placeable.placeRelative(0, yOffset)
                    yOffset += placeable.height
                }
            }
        }

        // 자식들의 최대 고유 높이를 합산하여 반환
        override fun IntrinsicMeasureScope.maxIntrinsicHeight(
            measurables: List<IntrinsicMeasurable>,
            width: Int
        ): Int = measurables.sumOf { it.maxIntrinsicHeight(width) }

        override fun IntrinsicMeasureScope.minIntrinsicWidth(
            measurables: List<IntrinsicMeasurable>,
            height: Int
        ): Int = measurables.maxOfOrNull { it.minIntrinsicWidth(height) } ?: 0
    }
)
```

---

## 주의사항 및 실전 팁

### 1. 단일 패스 측정 원칙 엄수

`Layout` 내부에서 같은 `measurable`을 두 번 `measure()`하면 런타임 예외(`IllegalStateException`)가 발생합니다. 여러 크기 후보를 시험하고 싶다면 `SubcomposeLayout` 또는 Intrinsic Measurement를 사용하세요.

### 2. Constraints 범위 위반 주의

`measure()`에 전달하는 `Constraints`가 `minWidth > maxWidth` 상태가 되면 크래시가 발생합니다. 항상 `coerceAtLeast(0)` 처리를 습관화하세요.

```kotlin
val childConstraints = constraints.copy(
    maxHeight = (constraints.maxHeight - reservedHeight).coerceAtLeast(0)
)
```

### 3. SubcomposeLayout의 슬롯 ID 안정성

슬롯 ID가 자주 변경되면 불필요한 서브컴포지션이 발생합니다. 슬롯 ID에는 안정적인 문자열이나 `enum` 값을 사용하세요.

```kotlin
// 권장: 안정적인 슬롯 ID
subcompose(slotId = "header") { ... }

// 피해야 할 패턴: 매 재구성마다 변하는 값을 슬롯 ID로 사용
subcompose(slotId = System.currentTimeMillis()) { ... } // BAD
```

### 4. placeRelative vs place

- **`placeRelative(x, y)`**: RTL(오른쪽에서 왼쪽) 레이아웃을 자동으로 미러링합니다. **일반적으로 이 메서드를 사용하세요.**
- **`place(x, y)`**: RTL 미러링 없이 절대 좌표로 배치합니다. 특별히 절대 좌표가 필요한 경우(예: 캔버스 위 오버레이)에만 사용합니다.

### 5. LookaheadLayout과 Shared Element Transition

Compose 1.5+에서 안정화된 `LookaheadScope` (이전의 `LookaheadLayout`)는 레이아웃 변경 애니메이션을 위해 "미래의 레이아웃 상태"를 미리 계산합니다. `AnimatedContent`, Shared Element Transition 내부에서 활용되며, 복잡한 위치 이동 애니메이션을 만들 때 강력합니다.

---

## 마무리

Jetpack Compose의 커스텀 레이아웃 API는 처음에는 낯설게 느껴지지만, 측정(`Constraints`)과 배치(`placeRelative`)라는 두 가지 핵심 개념을 이해하면 어떤 UI 요구사항도 구현할 수 있습니다.

| API | 사용 시점 |
|---|---|
| `Layout` | 복수 자식을 완전히 제어하고 싶을 때 |
| `Modifier.layout` | 단일 컴포저블의 측정·배치만 수정할 때 |
| `SubcomposeLayout` | 한 자식의 크기를 다른 자식의 컴포지션에 사용할 때 |
| Intrinsic Measurement | 부모에게 최소/최대 이상적 크기를 알려야 할 때 |

이 네 가지 도구를 상황에 맞게 선택하면, 표준 레이아웃으로는 불가능했던 정교하고 효율적인 UI를 Compose 위에서 구현할 수 있습니다.

---

## 참고 자료

- [Custom layouts \| Jetpack Compose \| Android Developers](https://developer.android.com/develop/ui/compose/layouts/custom)
- [Compose layout basics \| Jetpack Compose \| Android Developers](https://developer.android.com/develop/ui/compose/layouts/basics)
- [Basic layouts in Compose Codelab \| Android Developers](https://developer.android.com/codelabs/jetpack-compose-layouts)
