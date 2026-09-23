---
layout: post
title: "Jetpack Compose Nested Scrolling 심화: NestedScrollConnection과 NestedScrollDispatcher로 커스텀 스크롤 동작 구현하기"
date: 2026-09-23
categories: [android, flutter]
tags: [jetpack-compose, nested-scroll, NestedScrollConnection, NestedScrollDispatcher, android, kotlin, collapsing-toolbar, material3]
---

현대적인 모바일 앱에서 스크롤 인터랙션은 단순한 목록 스크롤을 넘어 매우 복잡한 형태로 진화했습니다. 콜랩싱 툴바, 스크롤에 반응하는 헤더 이미지, 탭 전환과 연계된 스크롤 등 다양한 UI 패턴이 요구됩니다. Jetpack Compose는 이를 위해 체계적인 **중첩 스크롤(Nested Scrolling)** 시스템을 제공합니다. 이번 포스트에서는 `NestedScrollConnection`과 `NestedScrollDispatcher`의 내부 원리를 깊이 파고들어, 실전에서 바로 활용할 수 있는 커스텀 스크롤 구현 방법을 다룹니다.

## 개념 설명: 중첩 스크롤 시스템의 구성 요소

Compose의 중첩 스크롤 시스템은 세 가지 핵심 요소로 구성됩니다.

### NestedScrollConnection

`NestedScrollConnection`은 스크롤 이벤트가 컴포저블 계층을 통해 전파될 때, 이 이벤트를 가로채거나 수정할 수 있게 해주는 인터페이스입니다. 네 가지 콜백 메서드를 통해 스크롤 사이클의 각 단계에 참여합니다:

- **`onPreScroll(available, source)`**: 자식 노드가 스크롤을 처리하기 **전에** 부모가 먼저 델타를 소비할 기회입니다. 반환값은 실제로 소비한 `Offset`입니다.
- **`onPostScroll(consumed, available, source)`**: 자식 노드가 스크롤을 처리한 **후에** 남은 델타를 처리할 기회입니다.
- **`onPreFling(available)`**: 플링(fling) 제스처가 시작되기 전 처리합니다. `Velocity`를 받아 소비량을 반환합니다.
- **`onPostFling(consumed, available)`**: 플링 제스처 처리 후 남은 속도를 처리합니다.

### NestedScrollDispatcher

`NestedScrollDispatcher`는 커스텀 스크롤 가능 컴포저블을 만들 때 사용합니다. 직접 제스처를 감지해 처리하는 컴포넌트에서, 스크롤 이벤트를 상위 중첩 스크롤 체인으로 올바르게 디스패치하는 역할을 합니다.

### nestedScroll Modifier

`Modifier.nestedScroll(connection, dispatcher?)`를 통해 컴포저블이 중첩 스크롤 시스템에 참여합니다. `connection`은 상위에서 이벤트를 수신하고, `dispatcher`는 하위로 이벤트를 전달할 때 사용합니다.

### 스크롤 사이클의 흐름

중첩 스크롤 사이클은 다음 순서로 진행됩니다:

1. **Pre-scroll 단계**: 스크롤 이벤트 발생 → 계층 최상위 부모의 `onPreScroll` 호출 → 부모가 원하는 만큼 델타 소비
2. **노드 소비**: 자식 스크롤 노드(`LazyColumn` 등)가 나머지 델타를 처리
3. **Post-scroll 단계**: 자식이 처리하고 남은 델타를 `onPostScroll`로 다시 부모에게 전달

이 메커니즘 덕분에 콜랩싱 툴바처럼 복잡한 UI도 자연스러운 스크롤 동작을 구현할 수 있습니다.

---

## 왜 필요한가?

### 기본 내장 지원과 그 한계

`LazyColumn`, `verticalScroll` 등의 기본 스크롤 컴포저블들은 중첩 스크롤 시스템에 자동으로 참여합니다. 이를 **"nested-scroll-by-default"** 규칙이라고 합니다. 단순한 목록 중첩은 별도 설정 없이도 동작합니다.

그러나 다음과 같은 고급 시나리오에서는 직접 `NestedScrollConnection`을 구현해야 합니다:

1. **콜랩싱 툴바**: 스크롤에 따라 앱바 높이가 동적으로 줄어드는 패턴
2. **스크롤 연동 애니메이션**: 스크롤 위치에 따라 UI 요소의 알파, 크기, 위치가 변하는 패턴
3. **당겨서 새로고침(Pull-to-Refresh)**: 최상단에서 추가 스크롤 시 새로고침 트리거
4. **드래그로 확장/축소**: 하단 시트(Bottom Sheet)를 드래그로 조절하는 패턴
5. **커스텀 스크롤 컴포넌트**: 직접 만든 스크롤 컴포넌트를 기존 시스템에 통합

특히 Material 3의 `TopAppBar`는 내부적으로 `TopAppBarScrollBehavior` + `NestedScrollConnection`을 사용해 콜랩싱 동작을 구현합니다. 이 원리를 이해하면 더욱 정교한 커스텀 UI를 만들 수 있습니다.

### 뷰 시스템과의 차이

기존 View 시스템에서는 `NestedScrollingParent3`/`NestedScrollingChild3` 인터페이스를 구현해야 했습니다. Compose는 이를 훨씬 간결한 인터페이스로 통합했습니다. 더불어 Compose와 View 간 상호운용(`rememberNestedScrollInteropConnection`)도 지원해, 마이그레이션 중인 앱에서도 중첩 스크롤을 유지할 수 있습니다.

---

## 실제 구현 예제

### 예제 1: 커스텀 콜랩싱 헤더 구현

가장 흔한 중첩 스크롤 사용 사례인 콜랩싱 헤더를 직접 구현합니다. 스크롤에 따라 헤더가 점진적으로 축소되고, 아래로 스크롤 시 다시 확장됩니다.

```kotlin
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.geometry.Offset
import androidx.compose.ui.input.nestedscroll.NestedScrollConnection
import androidx.compose.ui.input.nestedscroll.NestedScrollSource
import androidx.compose.ui.input.nestedscroll.nestedScroll
import androidx.compose.ui.unit.Dp
import androidx.compose.ui.unit.dp

val MaxHeaderHeight = 200.dp
val MinHeaderHeight = 56.dp

@Composable
fun CollapsingHeaderScreen() {
    var headerHeight by remember { mutableStateOf(MaxHeaderHeight) }

    // NestedScrollConnection: 헤더 높이를 Pre-scroll 단계에서 조절
    val nestedScrollConnection = remember {
        object : NestedScrollConnection {
            override fun onPreScroll(
                available: Offset,
                source: NestedScrollSource
            ): Offset {
                // available.y < 0: 위로 스크롤 (헤더 축소)
                // available.y > 0: 아래로 스크롤 (헤더 확장 — onPostScroll에서 처리)
                val delta = available.y
                val previousHeight = headerHeight
                val newHeight = (headerHeight.value + delta).dp
                headerHeight = newHeight.coerceIn(MinHeaderHeight, MaxHeaderHeight)

                // 실제로 소비한 델타만 반환 (나머지는 LazyColumn으로 전달됨)
                val consumed = headerHeight.value - previousHeight.value
                return Offset(0f, consumed)
            }

            override fun onPostScroll(
                consumed: Offset,
                available: Offset,
                source: NestedScrollSource
            ): Offset {
                // LazyColumn이 더 이상 스크롤할 수 없을 때(최상단),
                // 남은 아래 방향 델타로 헤더를 확장
                if (available.y > 0f) {
                    val previousHeight = headerHeight
                    val newHeight = (headerHeight.value + available.y).dp
                    headerHeight = newHeight.coerceIn(MinHeaderHeight, MaxHeaderHeight)
                    val consumed2 = headerHeight.value - previousHeight.value
                    return Offset(0f, consumed2)
                }
                return Offset.Zero
            }
        }
    }

    Column(
        modifier = Modifier
            .fillMaxSize()
            .nestedScroll(nestedScrollConnection)
    ) {
        // 동적 높이 헤더
        CollapsingHeader(height = headerHeight)

        // 스크롤 가능한 콘텐츠
        LazyColumn(modifier = Modifier.fillMaxSize()) {
            items((1..50).toList()) { index ->
                ListItem(
                    headlineContent = { Text("아이템 #$index") },
                    supportingContent = { Text("위로 스크롤하면 헤더가 축소됩니다") }
                )
                HorizontalDivider()
            }
        }
    }
}

@Composable
private fun CollapsingHeader(height: Dp) {
    // 헤더 진행률: 1f = 완전 확장, 0f = 완전 축소
    val progress = ((height - MinHeaderHeight) / (MaxHeaderHeight - MinHeaderHeight))
        .coerceIn(0f, 1f)

    Surface(
        modifier = Modifier
            .fillMaxWidth()
            .height(height),
        color = MaterialTheme.colorScheme.primaryContainer
    ) {
        Box(modifier = Modifier.fillMaxSize()) {
            // 확장 시에만 보이는 큰 제목
            Text(
                text = "오늘의 아티클",
                modifier = Modifier.padding(start = 16.dp, top = 16.dp),
                style = MaterialTheme.typography.headlineMedium,
                color = MaterialTheme.colorScheme.onPrimaryContainer
                    .copy(alpha = progress)
            )
            // 항상 표시되는 작은 제목 (하단 고정)
            Text(
                text = "Nested Scrolling 심화",
                modifier = Modifier
                    .align(androidx.compose.ui.Alignment.BottomStart)
                    .padding(16.dp),
                style = MaterialTheme.typography.titleMedium,
                color = MaterialTheme.colorScheme.onPrimaryContainer
            )
        }
    }
}
```

이 예제의 핵심은 `onPreScroll`에서 스크롤 델타를 **선점 소비**한다는 점입니다. 위로 스크롤할 때 헤더를 먼저 축소한 후, 남은 델타만 `LazyColumn`으로 전달됩니다. `onPostScroll`에서는 `LazyColumn`이 최상단에 도달해 남긴 델타로 헤더를 다시 확장합니다.

---

### 예제 2: NestedScrollDispatcher로 커스텀 드래그 컴포넌트 구현

직접 만든 수평 드래그 컴포넌트가 중첩 스크롤 시스템과 협력하도록 `NestedScrollDispatcher`를 활용합니다. 카드가 이동 한계에 도달하면 남은 델타를 상위 컨테이너로 전달합니다.

```kotlin
import androidx.compose.foundation.gestures.detectHorizontalDragGestures
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.geometry.Offset
import androidx.compose.ui.input.nestedscroll.NestedScrollConnection
import androidx.compose.ui.input.nestedscroll.NestedScrollDispatcher
import androidx.compose.ui.input.nestedscroll.NestedScrollSource
import androidx.compose.ui.input.nestedscroll.nestedScroll
import androidx.compose.ui.input.pointer.pointerInput
import androidx.compose.ui.input.pointer.util.VelocityTracker
import androidx.compose.ui.unit.Velocity
import androidx.compose.ui.unit.dp
import kotlinx.coroutines.launch

@Composable
fun DraggableCardWithNestedScroll() {
    val dispatcher = remember { NestedScrollDispatcher() }
    val scope = rememberCoroutineScope()
    var offsetX by remember { mutableStateOf(0f) }
    val maxOffset = 180f
    val velocityTracker = remember { VelocityTracker() }

    // NoOp connection: 이 컴포넌트 자체는 상위 이벤트를 가로채지 않음
    val noOpConnection = remember { object : NestedScrollConnection {} }

    Box(
        modifier = Modifier
            .fillMaxWidth()
            .height(140.dp)
            .nestedScroll(connection = noOpConnection, dispatcher = dispatcher)
            .pointerInput(Unit) {
                detectHorizontalDragGestures(
                    onDragStart = {
                        velocityTracker.resetTracking()
                    },
                    onHorizontalDrag = { change, dragAmount ->
                        change.consume()
                        velocityTracker.addPosition(change.uptimeMillis, change.position)

                        val available = Offset(dragAmount, 0f)

                        // 1단계: 상위 부모에게 Pre-scroll 기회 제공
                        val preConsumed = dispatcher.dispatchPreScroll(
                            available = available,
                            source = NestedScrollSource.UserInput
                        )
                        val remaining = available - preConsumed

                        // 2단계: 이 컴포넌트가 남은 델타 처리
                        val previousOffset = offsetX
                        offsetX = (offsetX + remaining.x).coerceIn(-maxOffset, maxOffset)
                        val selfConsumed = Offset(offsetX - previousOffset, 0f)

                        // 3단계: 소비 후 남은 델타를 Post-scroll로 상위에 전달
                        // (한계에 도달하면 상위 HorizontalPager 등이 이어받음)
                        val leftover = remaining - selfConsumed
                        dispatcher.dispatchPostScroll(
                            consumed = selfConsumed,
                            available = leftover,
                            source = NestedScrollSource.UserInput
                        )
                    },
                    onDragEnd = {
                        scope.launch {
                            val velocity = velocityTracker.calculateVelocity()
                            val preVelocity = dispatcher.dispatchPreFling(
                                available = Velocity(velocity.x, 0f)
                            )
                            val leftoverVelocity = Velocity(velocity.x - preVelocity.x, 0f)
                            dispatcher.dispatchPostFling(
                                consumed = preVelocity,
                                available = leftoverVelocity
                            )
                        }
                    }
                )
            },
        contentAlignment = Alignment.Center
    ) {
        // 배경: 드래그 범위 표시
        Surface(
            modifier = Modifier
                .fillMaxWidth()
                .height(100.dp),
            color = MaterialTheme.colorScheme.surfaceVariant,
            shape = MaterialTheme.shapes.medium
        ) {}

        // 드래그 가능한 카드
        Card(
            modifier = Modifier
                .offset(x = offsetX.dp)
                .size(width = 160.dp, height = 80.dp),
            colors = CardDefaults.cardColors(
                containerColor = MaterialTheme.colorScheme.secondaryContainer
            ),
            elevation = CardDefaults.cardElevation(defaultElevation = 4.dp)
        ) {
            Box(
                modifier = Modifier.fillMaxSize(),
                contentAlignment = Alignment.Center
            ) {
                Column(horizontalAlignment = Alignment.CenterHorizontally) {
                    Text(
                        text = "← 드래그 →",
                        style = MaterialTheme.typography.bodyMedium,
                        color = MaterialTheme.colorScheme.onSecondaryContainer
                    )
                    Text(
                        text = "${offsetX.toInt()}px",
                        style = MaterialTheme.typography.labelSmall,
                        color = MaterialTheme.colorScheme.onSecondaryContainer
                            .copy(alpha = 0.7f)
                    )
                }
            }
        }
    }
}
```

`NestedScrollDispatcher`를 사용할 때는 반드시 3단계(pre-scroll → self → post-scroll)를 모두 거쳐야 합니다. 이를 생략하면 상위 컨테이너가 예상치 못한 동작을 할 수 있습니다.

---

### Material 3 TopAppBar와의 연계

Material 3의 `TopAppBar`는 `scrollBehavior` 파라미터를 통해 내부적으로 `NestedScrollConnection`을 사용합니다.

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun Material3TopAppBarExample() {
    // 세 가지 스크롤 동작 중 선택
    // pinnedScrollBehavior     - 앱바 고정 (스크롤에 반응하지 않음)
    // enterAlwaysScrollBehavior - 아래로 스크롤하면 항상 앱바 표시
    // exitUntilCollapsedScrollBehavior - 최상단 도달 시에만 앱바 확장
    val scrollBehavior = TopAppBarDefaults.exitUntilCollapsedScrollBehavior()

    Scaffold(
        // Scaffold의 최상위에 nestedScroll 연결
        modifier = Modifier.nestedScroll(scrollBehavior.nestedScrollConnection),
        topBar = {
            LargeTopAppBar(
                title = { Text("Nested Scroll 예제") },
                scrollBehavior = scrollBehavior
            )
        }
    ) { innerPadding ->
        LazyColumn(
            modifier = Modifier
                .fillMaxSize()
                .padding(innerPadding)
        ) {
            items(100) { index ->
                ListItem(
                    headlineContent = { Text("아이템 $index") },
                    supportingContent = { Text("위로 스크롤하면 LargeTopAppBar가 축소됩니다") }
                )
                HorizontalDivider()
            }
        }
    }
}
```

`scrollBehavior.nestedScrollConnection`은 `Scaffold` 레벨에 연결되어야 합니다. `LazyColumn`에만 연결하면 내부 padding 영역에서 발생하는 스크롤이 앱바에 전달되지 않을 수 있습니다.

---

## 주의사항 및 팁

### 1. 소비량을 정확히 반환하기

`onPreScroll`의 반환값은 **실제로 소비한 양**이어야 합니다. 더 많이 반환하면 자식 스크롤이 비정상적으로 동작합니다.

```kotlin
// ❌ 잘못된 예: 전체 델타 반환 → 자식이 절대 스크롤 불가
override fun onPreScroll(available: Offset, source: NestedScrollSource): Offset {
    headerHeight = (headerHeight + available.y.dp).coerceIn(MinHeaderHeight, MaxHeaderHeight)
    return available  // 실제 변화와 무관하게 전체 소비 선언
}

// ✅ 올바른 예: 실제 변화량만 반환
override fun onPreScroll(available: Offset, source: NestedScrollSource): Offset {
    val previous = headerHeight
    headerHeight = (headerHeight + available.y.dp).coerceIn(MinHeaderHeight, MaxHeaderHeight)
    val actualConsumed = headerHeight.value - previous.value
    return Offset(0f, actualConsumed)
}
```

### 2. NestedScrollSource 구분 활용

`NestedScrollSource`를 구분하면 사용자 터치와 프로그래매틱 스크롤을 다르게 처리할 수 있습니다:

```kotlin
override fun onPreScroll(available: Offset, source: NestedScrollSource): Offset {
    return when (source) {
        NestedScrollSource.UserInput -> {
            // 사용자 드래그 중에만 헤더 축소
            consumeForHeader(available.y)
        }
        else -> Offset.Zero  // 프로그래매틱 스크롤은 무시
    }
}
```

### 3. remember로 Connection 객체 안정화

`NestedScrollConnection` 인스턴스가 리컴포지션마다 새로 생성되면 시스템이 매번 재연결을 수행해 성능이 저하됩니다.

```kotlin
// ❌ 리컴포지션마다 새 객체 생성
val connection = object : NestedScrollConnection { ... }

// ✅ 안정적으로 캐시
val connection = remember { object : NestedScrollConnection { ... } }

// State를 캡처해야 한다면 rememberUpdatedState 활용
val currentHeight by rememberUpdatedState(headerHeight)
val connection = remember {
    object : NestedScrollConnection {
        override fun onPreScroll(...): Offset {
            // currentHeight를 안전하게 참조
        }
    }
}
```

### 4. Pager + LazyColumn 조합 시 방향 필터링

`HorizontalPager` 안에 `LazyColumn`이 있는 경우, 수직 스크롤 이벤트가 Pager의 수평 스와이프를 방해하지 않도록 방향을 명확히 분리해야 합니다:

```kotlin
override fun onPreScroll(available: Offset, source: NestedScrollSource): Offset {
    // X축 이벤트는 절대 소비하지 않음 (Pager에게 넘김)
    return Offset(0f, processVerticalOnly(available.y))
}
```

### 5. 뷰-Compose 상호운용 시 interopConnection 사용

기존 View 기반의 `NestedScrollingParent`와 Compose 중첩 스크롤을 연결할 때는 별도 interop 연결이 필요합니다:

```kotlin
// Compose 내부 View가 외부 Compose 스크롤과 협력하게 함
val interopConnection = rememberNestedScrollInteropConnection()
AndroidView(
    factory = { ctx -> RecyclerView(ctx) },
    modifier = Modifier.nestedScroll(interopConnection)
)
```

### 6. 성능 모니터링

`onPreScroll`/`onPostScroll`은 매 프레임 스크롤 이벤트마다 호출됩니다. 이 콜백 내부에서 무거운 계산을 수행하면 프레임 드롭이 발생합니다. State 업데이트만 수행하고, 실제 렌더링과 애니메이션은 Compose의 Snapshot 시스템에 위임하세요.

---

## 정리

Jetpack Compose의 중첩 스크롤 시스템은 `NestedScrollConnection`과 `NestedScrollDispatcher`를 중심으로 설계된 강력하고 유연한 메커니즘입니다.

| 구성 요소 | 역할 | 주요 사용 시나리오 |
|-----------|------|-------------------|
| `NestedScrollConnection` | 스크롤 이벤트 수신·수정 | 콜랩싱 헤더, 스크롤 연동 UI |
| `NestedScrollDispatcher` | 스크롤 이벤트 상위로 발신 | 커스텀 드래그 컴포넌트 |
| `nestedScroll` Modifier | 시스템 참여 선언 | 모든 중첩 스크롤 참여 컴포저블 |

핵심은 스크롤 델타의 **정확한 소비량 반환**과 **3단계 사이클(pre → self → post)** 준수입니다. 이를 지키면 Material 3 컴포넌트들과도 자연스럽게 협력하는 정교한 스크롤 UI를 구현할 수 있습니다.

---

## 참고 자료
- [Nested scrolling modifiers | Jetpack Compose | Android Developers](https://developer.android.com/develop/ui/compose/touch-input/scroll/nested-scroll-modifiers)
- [Nested scrolling | Jetpack Compose | Android Developers](https://developer.android.com/develop/ui/compose/touch-input/pointer-input/nested-scroll)
- [NestedScrollConnection API Reference | Android Developers](https://developer.android.com/reference/kotlin/androidx/compose/ui/input/nestedscroll/NestedScrollConnection)
- [nestedScrollConnection in Compose Multiplatform Material3 | Kotlin](https://kotlinlang.org/api/compose-multiplatform/material3/androidx.compose.material3/-top-app-bar-scroll-behavior/nested-scroll-connection.html)
