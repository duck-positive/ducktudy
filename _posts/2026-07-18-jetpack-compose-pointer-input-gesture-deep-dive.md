---
layout: post
title: "Jetpack Compose 제스처 처리 심화: pointerInput API로 드래그·멀티터치·커스텀 제스처 완전 정복"
date: 2026-07-18
categories: [android, flutter]
tags: [android, jetpack-compose, gesture, pointer-input, touch, drag, transformable, kotlin]
---

## 개요

Jetpack Compose의 제스처 시스템은 전통적인 View 기반 `OnTouchListener`와는 완전히 다른 패러다임을 채택하고 있습니다. 선언형 UI 원칙에 따라 제스처도 Modifier 체인에 함수 형태로 정의하며, 레이어드(layered) 추상화 구조 덕분에 단순 탭 감지부터 핀치·줌·회전 같은 복합 멀티터치까지 일관된 API로 다룰 수 있습니다.

이 글에서는 Compose 제스처 시스템의 내부 동작 원리를 파악하고, `detectTapGestures`, `detectDragGestures`, `transformable`, 그리고 `awaitPointerEventScope`를 활용한 커스텀 제스처 구현까지 단계별로 살펴봅니다.

---

## 왜 Compose 제스처 API가 필요한가?

### View 시스템의 한계

전통적인 `View.setOnTouchListener`는 `ACTION_DOWN`, `ACTION_MOVE`, `ACTION_UP` 이벤트를 직접 파싱해야 했습니다. 멀티터치를 처리하려면 `getPointerId`, `findPointerIndex` 같은 저수준 API를 직접 다뤄야 했고, 제스처 인식기(`GestureDetector`, `ScaleGestureDetector`)를 별도로 생성하여 각각 이벤트를 포워딩해야 했습니다. 컴포넌트 간 터치 이벤트 소비 여부를 조율하는 과정에서 `onInterceptTouchEvent` 오버라이드가 필연적으로 등장하고, 이는 코드 복잡도를 크게 높였습니다.

### Compose가 선택한 세 가지 추상화 수준

Compose는 다음 세 계층을 제공합니다.

1. **컴포넌트 내장 지원** — `Button`, `TextField`, `LazyColumn` 등 대부분의 컴포넌트는 이미 적절한 제스처를 내부적으로 처리합니다.
2. **고수준 Modifier** — `clickable`, `draggable`, `scrollable`, `transformable` 등 특정 제스처를 전담하는 Modifier를 사용합니다.
3. **저수준 `pointerInput` Modifier** — 완전히 커스텀한 제스처 로직이 필요할 때 `PointerInputScope`를 직접 다룹니다.

이 계층 구조 덕분에 개발자는 필요한 만큼만 복잡도를 선택할 수 있습니다.

---

## 구현 예제 1: detectTapGestures와 detectDragGestures

아래 예제는 카드 컴포넌트 하나로 탭, 롱프레스, 드래그를 모두 처리합니다. 드래그 시 카드가 손가락을 따라 이동하고, 롱프레스 시 하이라이트 상태가 토글됩니다.

```kotlin
@Composable
fun DraggableCard() {
    var offset by remember { mutableStateOf(Offset.Zero) }
    var isHighlighted by remember { mutableStateOf(false) }
    var dragStarted by remember { mutableStateOf(false) }

    Box(modifier = Modifier.fillMaxSize()) {
        Box(
            modifier = Modifier
                .offset { IntOffset(offset.x.roundToInt(), offset.y.roundToInt()) }
                .size(120.dp)
                .background(
                    color = if (isHighlighted) Color(0xFFE94560) else Color(0xFF1A1A2E),
                    shape = RoundedCornerShape(16.dp)
                )
                // 탭·롱프레스 감지
                .pointerInput(Unit) {
                    detectTapGestures(
                        onTap = {
                            // 탭: 하이라이트 초기화
                            isHighlighted = false
                        },
                        onLongPress = {
                            // 롱프레스: 하이라이트 토글
                            isHighlighted = !isHighlighted
                        }
                    )
                }
                // 드래그 감지 — 별도 pointerInput 블록으로 체인
                .pointerInput(Unit) {
                    detectDragGestures(
                        onDragStart = { dragStarted = true },
                        onDragEnd = { dragStarted = false },
                        onDragCancel = { dragStarted = false },
                        onDrag = { change, dragAmount ->
                            // change.consume()으로 이벤트 소비 → 부모 스크롤 차단
                            change.consume()
                            offset += dragAmount
                        }
                    )
                },
            contentAlignment = Alignment.Center
        ) {
            Text(
                text = if (dragStarted) "이동 중" else "드래그 하세요",
                color = Color.White,
                fontSize = 13.sp,
                textAlign = TextAlign.Center
            )
        }
    }
}
```

### 핵심 포인트

- `pointerInput` Modifier는 **여러 개 체인**으로 쌓을 수 있습니다. 탭과 드래그를 같은 블록 안에 `launch { detectTapGestures(...) }` / `launch { detectDragGestures(...) }` 식으로 코루틴으로 병렬 실행하는 방법도 있습니다.
- `change.consume()`을 호출하면 해당 이벤트가 이미 소비됐다고 표시되어 부모 컴포저블(예: `LazyColumn`)의 스크롤 처리를 막을 수 있습니다.
- `onDrag`의 `dragAmount`는 **이전 이벤트 대비 델타**이므로, 절대 위치가 필요하면 `change.position`을 사용해야 합니다.

---

## 구현 예제 2: transformable + detectTransformGestures로 포토 뷰어 구현

이미지 뷰어처럼 핀치 줌, 회전, 패닝을 동시에 지원하는 컴포넌트입니다. `transformable`은 선언적으로 간단히 처리하고, 줌 범위 제한 같은 추가 로직은 상태 업데이트 시 직접 클램핑합니다.

```kotlin
@Composable
fun PhotoViewer(painter: Painter) {
    var scale by remember { mutableFloatStateOf(1f) }
    var rotation by remember { mutableFloatStateOf(0f) }
    var offset by remember { mutableStateOf(Offset.Zero) }

    val transformState = rememberTransformableState { zoomChange, panChange, rotationChange ->
        // 줌 범위 제한: 0.5x ~ 5x
        scale = (scale * zoomChange).coerceIn(0.5f, 5f)
        rotation += rotationChange
        offset += panChange
    }

    Box(
        modifier = Modifier
            .fillMaxSize()
            .clipToBounds()
            // graphicsLayer로 변환 적용 (레이아웃에 영향 없음)
            .graphicsLayer(
                scaleX = scale,
                scaleY = scale,
                rotationZ = rotation,
                translationX = offset.x,
                translationY = offset.y
            )
            .transformable(state = transformState),
        contentAlignment = Alignment.Center
    ) {
        Image(
            painter = painter,
            contentDescription = "확대 가능한 이미지",
            modifier = Modifier.fillMaxSize(),
            contentScale = ContentScale.Fit
        )
    }
}
```

`transformable`과 다른 제스처 Modifier(예: `scrollable`)를 동시에 쓰면 이벤트 충돌이 발생할 수 있습니다. 이때는 `transformable(state, lockRotationOnZoomPan = true)` 옵션으로 줌 중 회전을 잠그거나, `pointerInput`의 `detectTransformGestures`를 직접 사용해 충돌을 수동으로 조율합니다.

---

## 구현 예제 3: awaitPointerEventScope를 활용한 커스텀 제스처

고수준 API로 구현할 수 없는 복합 제스처(예: 두 손가락 탭)는 `awaitPointerEventScope`로 직접 구현합니다.

```kotlin
suspend fun PointerInputScope.detectTwoFingerTap(onTwoFingerTap: () -> Unit) {
    awaitPointerEventScope {
        while (true) {
            // Initial 패스에서 포인터 이벤트 대기
            val event = awaitPointerEvent(pass = PointerEventPass.Initial)
            val downPointers = event.changes.filter { it.pressed }

            if (downPointers.size == 2) {
                // 두 포인터가 모두 눌린 상태에서 짧은 시간 내 UP 감지
                val release = withTimeoutOrNull(300L) {
                    awaitPointerEvent(pass = PointerEventPass.Initial)
                }
                val upPointers = release?.changes?.filter { !it.pressed }
                if (upPointers?.size == 2) {
                    onTwoFingerTap()
                }
            }
        }
    }
}

// 사용 예
@Composable
fun TwoFingerTapBox() {
    var tapCount by remember { mutableIntStateOf(0) }

    Box(
        modifier = Modifier
            .fillMaxSize()
            .background(Color(0xFF0F3460))
            .pointerInput(Unit) {
                detectTwoFingerTap { tapCount++ }
            },
        contentAlignment = Alignment.Center
    ) {
        Text(
            text = "두 손가락 탭 횟수: $tapCount",
            color = Color.White,
            fontSize = 18.sp
        )
    }
}
```

`PointerEventPass`에는 세 가지 패스가 있습니다.

| 패스 | 방향 | 용도 |
|------|------|------|
| `Initial` | 부모 → 자식 | 자식보다 먼저 이벤트를 가로채야 할 때 |
| `Main` | 자식 → 부모 | 일반 이벤트 처리 (기본값) |
| `Final` | 부모 → 자식 | 다른 모든 처리 이후 마무리 소비 확인 |

---

## 이벤트 소비(Consumption)와 전파 규칙

Compose의 이벤트 소비는 View 시스템의 `return true/false`보다 훨씬 세밀합니다.

- `change.consume()` — 해당 change의 모든 컴포넌트를 소비로 표시
- `change.consumePositionChange()` — 위치 변화만 소비
- `change.isConsumed` — 이미 소비됐는지 확인

중요한 점은, **이벤트 소비가 전파를 즉시 막지는 않는다**는 것입니다. 이벤트는 모든 패스를 완전히 순회하지만, 다른 핸들러에서 `change.isConsumed`를 확인하고 자발적으로 처리를 건너뛰는 방식으로 동작합니다.

---

## 성능 주의사항 및 팁

### 1. `key` 파라미터로 recomposition 제어

`pointerInput(key1, key2) { ... }` 의 key가 바뀌면 기존 코루틴이 취소되고 새 블록이 시작됩니다. 상태를 캡처해야 한다면 람다 참조보다 `rememberUpdatedState`를 사용하세요.

```kotlin
// 잘못된 예: lambda가 stale closure 문제 일으킬 수 있음
.pointerInput(Unit) {
    detectTapGestures { onTap(someState) }
}

// 올바른 예: rememberUpdatedState로 항상 최신 값 참조
val currentOnTap by rememberUpdatedState(onTap)
.pointerInput(Unit) {
    detectTapGestures { currentOnTap(someState) }
}
```

### 2. `graphicsLayer` vs `offset` Modifier

드래그로 컴포넌트를 이동할 때, `Modifier.offset { IntOffset(...) }`는 레이아웃 패스를 재측정 없이 오프셋만 적용하지만, `graphicsLayer`는 RenderNode 레이어에서 GPU 변환을 적용하므로 더 빠릅니다. 빠른 드래그 애니메이션에는 `graphicsLayer`를 선호하세요.

### 3. 스크롤 컨테이너와의 충돌 해결

`LazyColumn` 안에 드래그 가능한 아이템이 있을 때, 수직 드래그는 두 제스처가 경합합니다. `Modifier.nestedScroll`을 활용하면 내부 컴포넌트가 먼저 소비하고 남은 스크롤을 부모에게 위임하는 구조를 만들 수 있습니다.

### 4. 접근성(Accessibility) 고려

`pointerInput`만 사용하면 TalkBack 사용자는 제스처를 사용할 수 없습니다. `Modifier.semantics`로 대안 액션을 명시하세요.

```kotlin
.semantics {
    onClick(label = "카드를 선택합니다") { performClick(); true }
}
```

---

## 정리

Jetpack Compose의 제스처 시스템은 세 계층의 추상화를 제공합니다. 단순한 경우에는 `clickable`, `draggable`, `transformable`만으로 충분하며, 고급 시나리오에서는 `pointerInput` + `detectXxx` 함수군을, 완전 커스텀 제스처에는 `awaitPointerEventScope`를 직접 사용합니다. 이벤트 소비 규칙과 `PointerEventPass`를 이해하면 중첩된 스크롤·드래그 충돌도 명확히 해결할 수 있습니다.

## 참고 자료
- [Understand gestures — Android Developers](https://developer.android.com/develop/ui/compose/touch-input/pointer-input/understand-gestures)
- [Drag, swipe, and fling — Android Developers](https://developer.android.com/develop/ui/compose/touch-input/pointer-input/drag-swipe-fling)
- [Multitouch: Panning, zooming, rotating — Android Developers](https://developer.android.com/develop/ui/compose/touch-input/pointer-input/multi-touch)
