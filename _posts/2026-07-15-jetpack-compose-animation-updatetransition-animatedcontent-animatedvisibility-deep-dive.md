---
layout: post
title: "Jetpack Compose 애니메이션 심화: updateTransition·AnimatedContent·AnimatedVisibility·Animatable 완전 정복"
date: 2026-07-15
categories: [android, flutter]
tags: [android, jetpack-compose, animation, updateTransition, AnimatedContent, AnimatedVisibility, Animatable, kotlin]
---

현대 Android 앱에서 애니메이션은 더 이상 선택 사항이 아닙니다. 사용자는 상태 전환이 자연스럽고 물리적으로 납득 가능한 앱을 기대합니다. Jetpack Compose는 기존 View 시스템의 `ObjectAnimator`, `ValueAnimator`, `TransitionManager` 대신 선언적이고 코루틴 친화적인 애니메이션 API 체계를 제공합니다. 이 글에서는 Compose 애니메이션의 4대 핵심 API인 `AnimatedVisibility`, `AnimatedContent`, `updateTransition`, `Animatable`을 내부 동작 원리와 함께 완전히 정복합니다.

---

## 왜 Compose 애니메이션 API를 깊게 알아야 하는가

Compose의 애니메이션 API는 레이어드(layered) 구조로 설계되어 있습니다. 최상위 레이어인 `animate*AsState`부터 중간 레이어인 `Transition`, 그리고 가장 하위 레이어인 `Animatable`과 `Animation`까지 각 레이어는 편의성과 제어 수준을 교환합니다.

```
animate*AsState (최고 편의성, 낮은 제어)
    ↓
AnimatedVisibility / AnimatedContent (중간 레이어)
    ↓
updateTransition / Transition (상태 묶음 관리)
    ↓
Animatable (최저 편의성, 최고 제어)
    ↓
Animation (내부 구현, 직접 사용 드묾)
```

이 계층을 이해하면 "어느 API를 언제 써야 하는가"라는 질문에 명확히 답할 수 있습니다. 잘못된 레이어를 선택하면 코드가 불필요하게 복잡해지거나, 반대로 애니메이션이 끊기거나 물리적으로 어색해집니다.

---

## 1. AnimatedVisibility: 나타남과 사라짐의 제어

`AnimatedVisibility`는 컴포저블의 등장(enter)과 퇴장(exit)을 애니메이션으로 처리하는 가장 간단한 방법입니다. 내부적으로는 `Transition<Boolean>`으로 구현되어 있어 상태가 `true → false`로 바뀔 때 퇴장 애니메이션이 완전히 끝나야만 컴포저블이 컴포지션 트리에서 제거됩니다. 이 점이 단순 `if (visible)` 분기와의 핵심 차이입니다.

```kotlin
@Composable
fun NotificationBanner(message: String) {
    var visible by remember { mutableStateOf(false) }

    LaunchedEffect(message) {
        visible = true
        delay(3000)
        visible = false
    }

    AnimatedVisibility(
        visible = visible,
        enter = slideInVertically(
            initialOffsetY = { -it },
            animationSpec = spring(
                dampingRatio = Spring.DampingRatioMediumBouncy,
                stiffness = Spring.StiffnessMedium
            )
        ) + fadeIn(animationSpec = tween(300)),
        exit = slideOutVertically(
            targetOffsetY = { -it },
            animationSpec = tween(250, easing = FastOutLinearInEasing)
        ) + fadeOut(animationSpec = tween(200))
    ) {
        Surface(
            modifier = Modifier
                .fillMaxWidth()
                .padding(horizontal = 16.dp, vertical = 8.dp),
            color = MaterialTheme.colorScheme.primaryContainer,
            shape = RoundedCornerShape(12.dp),
            shadowElevation = 4.dp
        ) {
            Text(
                text = message,
                modifier = Modifier.padding(16.dp),
                style = MaterialTheme.typography.bodyMedium
            )
        }
    }
}
```

### enter/exit 조합 연산자

`enter`와 `exit` 파라미터는 `EnterTransition`과 `ExitTransition`을 `+` 연산자로 조합할 수 있습니다. 내부적으로 `CombinedEnterTransition`으로 합산되어 병렬 실행됩니다.

| 종류 | 진입 | 퇴장 |
|------|------|------|
| 페이드 | `fadeIn` | `fadeOut` |
| 슬라이드 | `slideInHorizontally`, `slideInVertically` | `slideOutHorizontally`, `slideOutVertically` |
| 스케일 | `scaleIn` | `scaleOut` |
| 확장/축소 | `expandIn`, `expandHorizontally` | `shrinkOut`, `shrinkHorizontally` |

### 자식 요소에 개별 애니메이션 적용

`AnimatedVisibility` 블록 내부에서 `Modifier.animateEnterExit()`를 사용하면 자식마다 다른 애니메이션을 지정할 수 있습니다. 부모의 `enter/exit`와 독립적으로 동작하며, 부모의 상태 전환에 동기화됩니다.

```kotlin
AnimatedVisibility(visible = show) {
    Column {
        Text(
            "제목",
            modifier = Modifier.animateEnterExit(
                enter = fadeIn() + slideInHorizontally(),
                exit = fadeOut() + slideOutHorizontally()
            )
        )
        Text(
            "본문",
            modifier = Modifier.animateEnterExit(
                enter = fadeIn(tween(delayMillis = 150)),
                exit = fadeOut()
            )
        )
    }
}
```

---

## 2. AnimatedContent: 컨텐츠 교체 애니메이션

`AnimatedContent`는 `targetState` 값이 바뀔 때 이전 컨텐츠에서 새 컨텐츠로 전환하는 애니메이션을 처리합니다. 전환 중에는 이전 컨텐츠와 새 컨텐츠가 동시에 렌더링되므로 `zIndex`와 `ContentTransform`의 설계가 중요합니다.

```kotlin
@Composable
fun ScoreCounter(score: Int) {
    AnimatedContent(
        targetState = score,
        label = "score animation",
        transitionSpec = {
            // 점수가 증가할 때는 위에서 아래로, 감소할 때는 아래에서 위로
            val direction = if (targetState > initialState) 1 else -1
            (slideInVertically { height -> -height * direction } + fadeIn())
                .togetherWith(
                    slideOutVertically { height -> height * direction } + fadeOut()
                )
                .using(SizeTransform(clip = false))
        }
    ) { targetCount ->
        Text(
            text = "$targetCount",
            style = MaterialTheme.typography.displayLarge,
            fontWeight = FontWeight.Bold
        )
    }
}
```

### ContentTransform 구성 요소

`ContentTransform`은 세 가지로 구성됩니다:
- `enterTransition: EnterTransition` — 새 컨텐츠 진입 방식
- `exitTransition: ExitTransition` — 이전 컨텐츠 퇴장 방식
- `sizeTransform: SizeTransform?` — 두 컨텐츠의 크기 차이를 처리하는 방식

`SizeTransform(clip = false)`를 지정하면 전환 중에 컨텐츠가 컨테이너 경계 밖으로 넘칠 수 있습니다. 넘침이 시각적으로 어색하다면 `clip = true`(기본값)를 사용하세요.

---

## 3. updateTransition: 여러 속성을 하나의 상태로 묶기

`updateTransition`은 단일 상태 변수(`enum`, `sealed class`, `Boolean` 등)에 따라 여러 애니메이션 값을 동시에 구동할 때 사용합니다. 내부적으로 `Transition` 객체를 생성하고, 상태가 바뀔 때 모든 자식 애니메이션이 동기화되어 전환됩니다.

```kotlin
enum class CardState { Collapsed, Expanded }

@Composable
fun AnimatedCard(
    content: @Composable () -> Unit,
    expandedContent: @Composable () -> Unit
) {
    var cardState by remember { mutableStateOf(CardState.Collapsed) }
    val transition = updateTransition(targetState = cardState, label = "card transition")

    val cardElevation by transition.animateDp(
        transitionSpec = { spring(stiffness = Spring.StiffnessMediumLow) },
        label = "card elevation"
    ) { state ->
        when (state) {
            CardState.Collapsed -> 2.dp
            CardState.Expanded -> 8.dp
        }
    }

    val cardCornerRadius by transition.animateDp(
        label = "corner radius"
    ) { state ->
        when (state) {
            CardState.Collapsed -> 12.dp
            CardState.Expanded -> 24.dp
        }
    }

    val contentAlpha by transition.animateFloat(
        transitionSpec = { tween(durationMillis = 200) },
        label = "content alpha"
    ) { state ->
        when (state) {
            CardState.Collapsed -> 0f
            CardState.Expanded -> 1f
        }
    }

    val arrowRotation by transition.animateFloat(
        transitionSpec = { spring(stiffness = Spring.StiffnessMedium) },
        label = "arrow rotation"
    ) { state ->
        when (state) {
            CardState.Collapsed -> 0f
            CardState.Expanded -> 180f
        }
    }

    Card(
        modifier = Modifier
            .fillMaxWidth()
            .clickable {
                cardState = if (cardState == CardState.Collapsed)
                    CardState.Expanded else CardState.Collapsed
            },
        shape = RoundedCornerShape(cardCornerRadius),
        elevation = CardDefaults.cardElevation(defaultElevation = cardElevation)
    ) {
        Column(modifier = Modifier.padding(16.dp)) {
            Row(
                horizontalArrangement = Arrangement.SpaceBetween,
                modifier = Modifier.fillMaxWidth()
            ) {
                content()
                Icon(
                    imageVector = Icons.Default.ExpandMore,
                    contentDescription = null,
                    modifier = Modifier.rotate(arrowRotation)
                )
            }

            // 확장 내용은 AnimatedVisibility로 show/hide
            transition.AnimatedVisibility(
                visible = { it == CardState.Expanded },
                enter = expandVertically() + fadeIn(),
                exit = shrinkVertically() + fadeOut()
            ) {
                Column(modifier = Modifier.graphicsLayer { alpha = contentAlpha }) {
                    Spacer(modifier = Modifier.height(12.dp))
                    expandedContent()
                }
            }
        }
    }
}
```

### Transition.AnimatedVisibility와 Transition.AnimatedContent

`updateTransition`이 반환한 `Transition` 객체에는 멤버 함수로 `AnimatedVisibility`와 `AnimatedContent`가 있습니다. 이를 사용하면 내부 컴포저블도 부모 전환과 동기화됩니다. 독립적인 상위 레벨 `AnimatedVisibility`를 쓰면 타이밍이 어긋날 수 있습니다.

---

## 4. Animatable: 코루틴 기반 정밀 제어

`Animatable`은 코루틴 기반으로 동작하는 가장 저수준의 애니메이션 API입니다. `animate*AsState`가 내부적으로 사용하는 기반이기도 합니다. 다음과 같은 경우에 직접 사용합니다:
- 제스처 인터럽트(손가락을 떼는 순간 애니메이션 방향 전환)
- 물리 기반 스냅(snap after fling)
- 복수 코루틴이 동일한 값을 두고 경쟁하는 구조

```kotlin
@Composable
fun ShakeableTextField(
    value: String,
    onValueChange: (String) -> Unit,
    isError: Boolean
) {
    val offsetX = remember { Animatable(0f) }
    val scope = rememberCoroutineScope()

    LaunchedEffect(isError) {
        if (isError) {
            // 떨림(shake) 애니메이션: 스냅 + 좌우 진동 + 스프링 복귀
            offsetX.snapTo(0f)
            val shakeOffsets = listOf(20f, -20f, 14f, -14f, 8f, -8f, 4f, -4f, 0f)
            for (offset in shakeOffsets) {
                offsetX.animateTo(
                    targetValue = offset,
                    animationSpec = tween(
                        durationMillis = 50,
                        easing = LinearEasing
                    )
                )
            }
        }
    }

    OutlinedTextField(
        value = value,
        onValueChange = onValueChange,
        isError = isError,
        modifier = Modifier.offset { IntOffset(offsetX.value.roundToInt(), 0) },
        label = { Text("입력") }
    )
}

@Composable
fun DraggableChip(label: String) {
    val offsetX = remember { Animatable(0f) }
    val offsetY = remember { Animatable(0f) }
    val scope = rememberCoroutineScope()

    Box(
        modifier = Modifier
            .offset {
                IntOffset(
                    offsetX.value.roundToInt(),
                    offsetY.value.roundToInt()
                )
            }
            .pointerInput(Unit) {
                detectDragGestures(
                    onDragEnd = {
                        scope.launch {
                            // 드래그 종료 시 원점으로 스프링 복귀
                            launch {
                                offsetX.animateTo(
                                    0f,
                                    spring(dampingRatio = Spring.DampingRatioMediumBouncy)
                                )
                            }
                            launch {
                                offsetY.animateTo(
                                    0f,
                                    spring(dampingRatio = Spring.DampingRatioMediumBouncy)
                                )
                            }
                        }
                    },
                    onDrag = { _, dragAmount ->
                        scope.launch {
                            // 드래그 중에는 즉시 반응 (snapTo)
                            offsetX.snapTo(offsetX.value + dragAmount.x)
                            offsetY.snapTo(offsetY.value + dragAmount.y)
                        }
                    }
                )
            }
    ) {
        SuggestionChip(
            onClick = {},
            label = { Text(label) }
        )
    }
}
```

### snapTo vs animateTo

| 메서드 | 설명 | 사용 시점 |
|--------|------|-----------|
| `snapTo(value)` | 즉시 값 설정, 애니메이션 없음 | 제스처 추적, 초기화 |
| `animateTo(value, spec)` | 지정 스펙으로 애니메이션 | 자동 전환, 복귀 |
| `animateDecay(velocity, decay)` | 감속 애니메이션 (fling) | 스와이프 릴리즈 |
| `stop()` | 진행 중인 애니메이션 즉시 중단 | 새로운 제스처 시작 시 |

`Animatable`은 코루틴으로 구동되므로 새로운 `animateTo` 호출이 진행 중인 애니메이션을 자동으로 취소합니다. 이 덕분에 인터럽트 처리가 자연스럽게 이루어집니다.

---

## 5. AnimationSpec 심화: spring vs tween vs keyframes

### spring

물리 기반으로 지속 시간 없이 `dampingRatio`와 `stiffness`로 제어합니다. 인터럽트 시 현재 속도를 보존하여 자연스러운 연속성을 보장합니다.

```kotlin
// 과감한 바운스
spring(
    dampingRatio = Spring.DampingRatioHighBouncy,  // 0.2f
    stiffness = Spring.StiffnessMedium             // 400f
)

// 빠르고 부드러운 전환 (바운스 없음)
spring(
    dampingRatio = Spring.DampingRatioNoBouncy,    // 1f
    stiffness = Spring.StiffnessHigh               // 10_000f
)
```

### tween

지속 시간 기반. `durationMillis`, `delayMillis`, `easing`으로 제어합니다.

```kotlin
tween(
    durationMillis = 300,
    delayMillis = 50,
    easing = FastOutSlowInEasing  // Material Design 표준 커브
)
```

### keyframes

특정 타임스탬프에서의 값을 직접 지정합니다. 복잡한 다단계 이징 구현에 적합합니다.

```kotlin
keyframes {
    durationMillis = 400
    0f at 0 with FastOutLinearInEasing
    0.8f at 200 with LinearOutSlowInEasing
    1f at 400
}
```

---

## 주의사항 및 성능 팁

### 1. label 파라미터를 항상 지정하라

`animate*AsState`, `updateTransition`, `Animatable` 등 모든 animation API에는 `label` 파라미터가 있습니다. Android Studio의 **Animation Preview** 도구에서 각 애니메이션을 식별하는 데 사용됩니다. 생략해도 동작하지만, 디버깅이 극도로 어려워집니다.

### 2. 불필요한 recomposition 방지

애니메이션 값을 읽는 컴포저블이 매 프레임마다 recompose되면 성능이 저하됩니다. `graphicsLayer { alpha = animatedAlpha.value }` 처럼 `Modifier.graphicsLayer` 내에서 값을 소비하면 recomposition 없이 렌더 단계에서만 반영됩니다.

```kotlin
// 나쁜 예: Text가 매 프레임 recompose됨
val alpha by animateFloatAsState(if (show) 1f else 0f, label = "alpha")
Box(modifier = Modifier.alpha(alpha)) { ... }

// 좋은 예: graphicsLayer는 렌더 단계에서만 적용
Box(modifier = Modifier.graphicsLayer { this.alpha = alpha }) { ... }
```

### 3. AnimatedContent에서 key 지정

`AnimatedContent`의 `targetState`가 데이터 클래스나 복잡한 객체일 경우, `contentKey` 파라미터로 실제 전환 트리거 기준을 명시적으로 지정하세요. 불필요한 애니메이션 전환을 방지합니다.

```kotlin
AnimatedContent(
    targetState = uiState,
    contentKey = { it.screenType }  // screenType이 바뀔 때만 전환
) { state ->
    when (state.screenType) { ... }
}
```

### 4. InfiniteTransition의 리소스 관리

`rememberInfiniteTransition`은 컴포저블이 화면에 있는 동안 지속 실행됩니다. 백그라운드 상태에서도 CPU를 소모하므로, `LaunchedEffect`와 `isActive` 상태를 결합하거나 `Lifecycle.Event.ON_PAUSE`에서 애니메이션을 일시 중단하는 처리를 추가하는 것이 좋습니다.

### 5. updateTransition의 상태 타입 선택

`updateTransition`의 상태 타입으로 `Boolean`보다는 `enum class`나 `sealed class`를 쓰는 것이 유지보수 면에서 낫습니다. 상태가 3가지 이상으로 확장될 때 `Boolean`은 한계를 드러냅니다.

---

## 요약

| API | 사용 시점 | 제어 수준 |
|-----|-----------|-----------|
| `animate*AsState` | 단일 값의 단순 전환 | 최저 |
| `AnimatedVisibility` | 표시/숨김 전환 | 낮음 |
| `AnimatedContent` | 컨텐츠 교체 전환 | 낮음 |
| `updateTransition` | 다중 속성을 하나의 상태로 | 중간 |
| `Animatable` | 제스처 인터럽트·정밀 제어 | 최고 |

Compose 애니메이션의 핵심은 "상태가 변하면 애니메이션이 따라온다"는 선언적 사고입니다. 어디서 상태를 변경할지만 결정하면, 실제 보간(interpolation)과 타이밍은 API가 처리합니다. 올바른 레이어를 선택하고 `label`, `graphicsLayer`, `contentKey`를 습관적으로 적용하면, 성능과 가독성 모두를 잡을 수 있습니다.

## 참고 자료

- [Animations in Compose — 공식 소개 문서](https://developer.android.com/develop/ui/compose/animation/introduction)
- [Animation modifiers and composables — AnimatedVisibility·AnimatedContent 레퍼런스](https://developer.android.com/develop/ui/compose/animation/composables-modifiers)
- [Value-based animations — Transition·Animatable 레퍼런스](https://developer.android.com/develop/ui/compose/animation/value-based)
- [Animating elements in Jetpack Compose — 공식 코드랩](https://developer.android.com/codelabs/jetpack-compose-animation)
