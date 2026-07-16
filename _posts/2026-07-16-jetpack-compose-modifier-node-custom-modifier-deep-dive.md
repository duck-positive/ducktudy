---
layout: post
title: "Jetpack Compose Modifier 심화: 내부 동작 원리·Modifier.Node API·커스텀 Modifier 완전 정복"
date: 2026-07-16
categories: [android, flutter]
tags: [android, jetpack-compose, modifier, modifier-node, custom-modifier, kotlin, ui]
---

Jetpack Compose를 처음 배울 때 `Modifier.padding().background().clickable()` 처럼 메서드를 체인 방식으로 연결하면 원하는 UI가 뚝딱 만들어지는 경험을 한다. 그런데 이 체인의 순서가 왜 결과에 영향을 미치는지, 내부에서 실제로 어떤 구조로 처리되는지, 그리고 나만의 커스텀 Modifier를 만드는 올바른 방법은 무엇인지 깊이 파고들면 Compose에 대한 이해가 완전히 달라진다. 이 글에서는 Modifier의 내부 동작 원리부터 최신 Modifier.Node API를 활용한 고성능 커스텀 Modifier 구현까지 완전히 정복한다.

## Modifier란 무엇인가 — 개념과 설계 철학

Jetpack Compose에서 Modifier는 Composable 함수에 **행동(behavior), 외관(appearance), 레이아웃(layout)**을 부여하는 불변 객체다. Android의 전통적인 XML 속성 시스템과 다르게, Compose는 모든 속성을 Modifier라는 단일 추상화로 통일했다. 패딩도 Modifier, 클릭 이벤트도 Modifier, 그림자 효과도 Modifier다.

```kotlin
// XML 시절: 속성이 View 내부에 산재
// android:padding="16dp"
// android:background="@color/blue"
// android:clickable="true"

// Compose: 모든 것이 Modifier 체인
Text(
    text = "Hello Compose",
    modifier = Modifier
        .padding(16.dp)
        .background(Color.Blue)
        .clickable { /* ... */ }
)
```

Modifier가 이런 설계를 택한 이유는 **컴포저블의 단일 책임 원칙(SRP)**을 지키기 위해서다. `Text`는 텍스트 렌더링에만 집중하고, 외부 스타일·동작은 Modifier가 담당한다. 덕분에 같은 Composable 함수를 다양한 맥락에서 재사용할 수 있다.

## Modifier의 내부 구조 — 불변 연결 리스트

Modifier는 사실 단순한 인터페이스다. Kotlin 소스를 살펴보면 `Modifier`는 `companion object`를 가진 인터페이스이며, 체이닝은 **불변 연결 리스트(immutable linked list)**로 구현된다.

```kotlin
// Modifier 내부 구조 (단순화)
interface Modifier {
    // 리스트를 왼쪽(처음)부터 순서대로 접어 하나의 값으로 만든다
    fun <R> foldIn(initial: R, operation: (R, Element) -> R): R
    // 리스트를 오른쪽(끝)부터 순서대로 접는다
    fun <R> foldOut(initial: R, operation: (Element, R) -> R): R

    interface Element : Modifier { /* ... */ }
    companion object : Modifier { /* 빈 Modifier */ }
}
```

`Modifier.padding(16.dp).background(Color.Red)` 를 호출하면 내부적으로 `CombinedModifier` 노드가 생성되어 다음과 같은 연결 구조를 형성한다.

```
CombinedModifier(
    outer = PaddingModifier(16.dp),    // 첫 번째 요소 (바깥)
    inner = BackgroundModifier(Red)    // 두 번째 요소 (안쪽)
)
```

### 순서가 왜 중요한가

Modifier 체인은 **바깥(outer)에서 안쪽(inner)으로 레이아웃을 감싸는** 방식으로 적용된다. 즉, 먼저 선언한 Modifier가 더 바깥 레이어를 담당한다.

```kotlin
// 케이스 A: 패딩 → 배경
// 배경이 패딩 안쪽(콘텐츠 영역)에만 칠해진다
Box(
    modifier = Modifier
        .padding(16.dp)
        .background(Color.Red)
)

// 케이스 B: 배경 → 패딩
// 배경이 패딩 영역까지 포함해서 칠해진다
Box(
    modifier = Modifier
        .background(Color.Red)
        .padding(16.dp)
)
```

clickable의 경우에도 순서가 ripple 영역에 영향을 준다. `clickable` 이후에 `padding`을 넣으면 패딩 영역은 클릭 영역에서 제외된다.

## Modifier.composed의 문제점과 Modifier.Node의 등장

초창기 Compose에서 상태를 갖는 커스텀 Modifier를 만들려면 `composed {}` 블록을 사용했다.

```kotlin
// 구식 방식: composed {} — 성능 문제 있음
fun Modifier.myShimmer(): Modifier = composed {
    val shimmerAlpha by animateFloatAsState(
        targetValue = 1f, animationSpec = infiniteRepeatable(...)
    )
    this.then(Modifier.graphicsLayer { alpha = shimmerAlpha })
}
```

`composed`는 내부적으로 **서브컴포지션(subcomposition)**을 사용한다. 즉, Modifier마다 독립적인 Composition 트리를 만들어 Composable 함수를 호출할 수 있게 한다. 편리하지만 다음 문제가 있다.

- **성능 오버헤드**: 매 리컴포지션마다 서브컴포지션이 새로 평가된다.
- **재사용 불가**: 같은 설정의 Modifier라도 노드를 재사용하지 못하고 매번 새로 만든다.
- **테스트 어려움**: 서브컴포지션 내부 상태에 접근하기 어렵다.

이 문제를 해결하기 위해 Compose 1.4부터 **Modifier.Node API**가 도입되었다. Compose 팀에 따르면 기존 `clickable` 구현을 Modifier.Node로 마이그레이션한 결과 약 **80% 성능 향상**이 측정되었다.

## Modifier.Node API 완전 정복

Modifier.Node 기반 커스텀 Modifier는 세 가지 구성 요소로 이뤄진다.

| 구성 요소 | 역할 |
|---|---|
| `Modifier.Node` 구현체 | 실제 로직과 상태를 보유하는 노드 |
| `ModifierNodeElement` | 노드를 생성·업데이트하는 불변 팩토리 |
| 확장 함수 (Factory) | 개발자에게 공개되는 API 진입점 |

### 예제 1 — 디버그용 DrawBorder 커스텀 Modifier

개발 중 컴포저블의 경계를 시각적으로 확인하고 싶을 때 유용한 `Modifier.debugBorder()` 를 구현해 보자.

```kotlin
import androidx.compose.ui.Modifier
import androidx.compose.ui.geometry.Rect
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.graphics.Paint
import androidx.compose.ui.graphics.drawscope.ContentDrawScope
import androidx.compose.ui.node.DrawModifierNode
import androidx.compose.ui.node.ModifierNodeElement
import androidx.compose.ui.platform.InspectorInfo
import androidx.compose.ui.unit.Dp
import androidx.compose.ui.unit.dp

// 1. Modifier.Node 구현체 — DrawModifierNode로 그리기 로직 담당
private class DebugBorderNode(
    var color: Color,
    var width: Dp
) : Modifier.Node(), DrawModifierNode {

    override fun ContentDrawScope.draw() {
        // 먼저 자식 콘텐츠를 그린다
        drawContent()
        // 그 위에 경계선을 오버레이
        val strokeWidthPx = width.toPx()
        drawRect(
            color = color,
            style = androidx.compose.ui.graphics.drawscope.Stroke(
                width = strokeWidthPx
            )
        )
    }
}

// 2. ModifierNodeElement — 노드 생성/업데이트/동등성 처리
private data class DebugBorderElement(
    val color: Color,
    val width: Dp
) : ModifierNodeElement<DebugBorderNode>() {

    override fun create() = DebugBorderNode(color, width)

    override fun update(node: DebugBorderNode) {
        // 파라미터가 변경될 때만 호출됨 — 노드를 재사용한다!
        node.color = color
        node.width = width
    }

    override fun InspectorInfo.inspectableProperties() {
        name = "debugBorder"
        properties["color"] = color
        properties["width"] = width
    }
}

// 3. 확장 함수 — 공개 API 진입점
fun Modifier.debugBorder(
    color: Color = Color.Red,
    width: Dp = 2.dp
): Modifier = this then DebugBorderElement(color, width)
```

사용 예시:

```kotlin
@Composable
fun DebugScreen() {
    Column(
        modifier = Modifier
            .padding(16.dp)
            .debugBorder(color = Color.Cyan, width = 1.dp)
    ) {
        Text("항목 1", modifier = Modifier.debugBorder(Color.Green))
        Text("항목 2", modifier = Modifier.debugBorder(Color.Blue))
    }
}
```

### 예제 2 — 포커스 여부에 따라 스케일이 바뀌는 FocusScale Modifier

TV 앱이나 키보드 접근성이 필요한 앱에서 포커스 상태를 시각적으로 표현하는 Modifier를 만들어 보자. 이 예제는 `FocusEventModifierNode`를 활용해 포커스 상태를 수신하고, `AnimationFrameClock`으로 애니메이션을 구동한다.

```kotlin
import androidx.compose.animation.core.Animatable
import androidx.compose.animation.core.spring
import androidx.compose.ui.Modifier
import androidx.compose.ui.focus.FocusState
import androidx.compose.ui.graphics.graphicsLayer
import androidx.compose.ui.node.FocusEventModifierNode
import androidx.compose.ui.node.ModifierNodeElement
import androidx.compose.ui.platform.InspectorInfo
import kotlinx.coroutines.launch

// Modifier.Node + FocusEventModifierNode 구현
private class FocusScaleNode(
    var focusedScale: Float,
    var unfocusedScale: Float
) : Modifier.Node(), FocusEventModifierNode,
    androidx.compose.ui.node.LayoutAwareModifierNode,
    androidx.compose.ui.node.DrawModifierNode {

    private val scaleAnimatable = Animatable(unfocusedScale)

    override fun onFocusEvent(focusState: FocusState) {
        val targetScale = if (focusState.isFocused) focusedScale else unfocusedScale
        // coroutineScope는 Modifier.Node의 생명주기에 바인딩됨
        coroutineScope.launch {
            scaleAnimatable.animateTo(
                targetValue = targetScale,
                animationSpec = spring(dampingRatio = 0.6f, stiffness = 300f)
            )
        }
    }

    override fun ContentDrawScope.draw() {
        // 현재 애니메이션 스케일 값으로 그리기
        val scale = scaleAnimatable.value
        drawContext.canvas.save()
        drawContext.canvas.scale(
            sx = scale,
            sy = scale,
            pivotX = size.width / 2f,
            pivotY = size.height / 2f
        )
        drawContent()
        drawContext.canvas.restore()
    }

    // 노드가 Composition에서 분리될 때 자동 취소됨
    override fun onDetach() {
        // coroutineScope가 자동으로 취소되므로 별도 처리 불필요
    }
}

private data class FocusScaleElement(
    val focusedScale: Float,
    val unfocusedScale: Float
) : ModifierNodeElement<FocusScaleNode>() {

    override fun create() = FocusScaleNode(focusedScale, unfocusedScale)

    override fun update(node: FocusScaleNode) {
        node.focusedScale = focusedScale
        node.unfocusedScale = unfocusedScale
    }

    override fun InspectorInfo.inspectableProperties() {
        name = "focusScale"
        properties["focusedScale"] = focusedScale
        properties["unfocusedScale"] = unfocusedScale
    }
}

fun Modifier.focusScale(
    focusedScale: Float = 1.1f,
    unfocusedScale: Float = 1.0f
): Modifier = this then FocusScaleElement(focusedScale, unfocusedScale)
```

TV 메뉴에서 이렇게 사용한다:

```kotlin
@Composable
fun TvMenuButton(label: String) {
    var isFocused by remember { mutableStateOf(false) }

    Button(
        onClick = { /* ... */ },
        modifier = Modifier
            .onFocusChanged { isFocused = it.isFocused }
            .focusScale(focusedScale = 1.15f)
    ) {
        Text(label)
    }
}
```

## Modifier.Node가 제공하는 인터페이스 목록

Modifier.Node는 단독으로는 아무것도 하지 않는다. 아래 인터페이스 중 필요한 것을 조합해서 구현한다.

| 인터페이스 | 기능 |
|---|---|
| `DrawModifierNode` | Canvas 위에 직접 그리기 (`draw()`) |
| `LayoutModifierNode` | 측정·배치 커스터마이징 (`measure()`) |
| `PointerInputModifierNode` | 포인터(터치/마우스) 이벤트 수신 |
| `FocusEventModifierNode` | 포커스 상태 이벤트 수신 |
| `SemanticsModifierNode` | 접근성 시맨틱 트리에 정보 추가 |
| `GlobalPositionAwareModifierNode` | 전역 좌표계에서의 위치 수신 |
| `CompositionLocalConsumerModifierNode` | CompositionLocal 값 읽기 |
| `ObserverModifierNode` | 스냅샷 상태 변화 관찰 |

여러 인터페이스를 동시에 구현할 수 있어 복잡한 동작을 단일 노드로 표현할 수 있다.

## 주의사항 및 실전 팁

### 1. `composed {}` 대신 `Modifier.Node`를 써야 하는 경우

애니메이션, 포커스, 포인터 이벤트 등 **상태(state)나 코루틴이 필요한 커스텀 Modifier**는 반드시 Modifier.Node로 구현한다. 단순히 기존 Modifier를 조합하는 경우는 확장 함수로 충분하다.

```kotlin
// 이 정도는 확장 함수로 충분
fun Modifier.cardStyle(): Modifier = this
    .shadow(elevation = 4.dp, shape = RoundedCornerShape(12.dp))
    .background(Color.White, shape = RoundedCornerShape(12.dp))
    .padding(16.dp)
```

### 2. `ModifierNodeElement`에서 `equals()` 구현 필수

`data class`를 사용하면 자동으로 처리되지만, 일반 클래스로 구현할 경우 `equals()`와 `hashCode()`를 반드시 오버라이드해야 한다. Compose는 두 Element가 같은지 비교해 노드 재사용 여부를 결정하기 때문이다. `equals()` 가 항상 `false`를 반환하면 불필요한 노드 재생성이 반복된다.

### 3. `update()` 에서 노드 속성만 변경할 것

`update()` 는 리컴포지션 시 파라미터가 바뀌었을 때 호출된다. 이 함수 안에서 새 상태 객체를 생성하거나 무거운 작업을 하지 말 것. 오직 노드의 프로퍼티 값만 갱신해야 한다.

### 4. `coroutineScope` 생명주기 주의

`Modifier.Node`의 `coroutineScope`는 노드가 Composition에 부착(`onAttach`)될 때 활성화되고, 분리(`onDetach`)될 때 자동으로 취소된다. 코루틴을 별도 `Job`에 저장해두고 수동으로 취소할 필요가 없다.

### 5. `invalidateDraw()` / `invalidateLayout()` 명시적 호출

`ObserverModifierNode`나 외부 상태 변화에 반응해 UI를 갱신해야 할 때, 자동 무효화가 비활성화(`autoInvalidate = false`)된 노드에서는 `invalidateDraw()` 또는 `invalidateLayout()`을 명시적으로 호출해야 한다. 기본값(`autoInvalidate = true`)에서는 Compose가 자동으로 처리한다.

### 6. Modifier 순서 — 레이아웃 단계와 그리기 단계를 구분

Modifier 체인 중 `LayoutModifierNode`는 레이아웃 단계에서, `DrawModifierNode`는 그리기 단계에서 동작한다. 하나의 Modifier.Node에서 두 인터페이스를 모두 구현하면, 두 단계 모두에서 커스텀 동작을 수행할 수 있다. 다만 이 경우 해당 노드가 레이아웃 단계에서 크기를 변경하면 그리기 단계에서도 변경된 크기 기준으로 동작한다는 점을 유의한다.

## 마무리

Modifier는 Jetpack Compose의 UI 시스템이 얼마나 정교하게 설계되었는지를 보여주는 핵심 요소다. 불변 연결 리스트 구조로 안전하고 예측 가능한 합성을 보장하고, Modifier.Node API는 서브컴포지션 오버헤드를 제거해 고성능 커스텀 컴포넌트를 만들 수 있게 한다. 다음에 커스텀 UI 동작이 필요할 때, `composed {}`를 습관적으로 사용하기보다 `Modifier.Node`를 먼저 고려해보자. 성능과 유연성 두 가지를 모두 잡을 수 있다.

## 참고 자료

- [Compose modifiers — Android Developers](https://developer.android.com/develop/ui/compose/modifiers)
- [Create custom modifiers — Android Developers](https://developer.android.com/develop/ui/compose/custom-modifiers)
- [List of Compose modifiers — Android Developers](https://developer.android.com/develop/ui/compose/modifiers-list)
