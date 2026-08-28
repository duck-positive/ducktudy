---
layout: post
title: "Jetpack Compose Canvas 심화: DrawScope·Path·Shader로 완전한 커스텀 그래픽 구현하기"
date: 2026-08-28
categories: [android, flutter]
tags: [android, jetpack-compose, canvas, drawscope, path, shader, graphics, kotlin]
---

Android 개발을 하다 보면 기본 UI 컴포넌트만으로는 표현하기 어려운 복잡한 그래픽을 구현해야 할 때가 있습니다. 파이 차트, 물결 애니메이션, 커스텀 진행률 표시기, 혹은 게임 UI처럼 픽셀 단위의 세밀한 제어가 필요한 경우입니다. Jetpack Compose는 `Canvas` 컴포저블과 `DrawScope`를 통해 이러한 요구를 우아하게 처리할 수 있는 강력한 그래픽 시스템을 제공합니다. 이 글에서는 Compose의 그래픽 API를 깊이 파고들어, 실제 프로덕션에서 활용할 수 있는 수준의 커스텀 그래픽을 구현하는 법을 배웁니다.

---

## 왜 Compose Canvas가 필요한가?

기존 View 시스템에서는 커스텀 그래픽을 위해 `View`를 상속하고 `onDraw(Canvas)`를 오버라이드해야 했습니다. `Paint` 객체를 손으로 관리하고, `invalidate()`를 직접 호출해야 했으며, 상태 변경과 렌더링 사이의 동기화를 개발자가 책임져야 했습니다.

Compose Canvas는 이 문제를 세 가지 방식으로 해결합니다.

**1. 선언형 그래픽**: 상태가 바뀌면 Compose가 알아서 리컴포지션하고 재드로잉합니다. `invalidate()` 호출이 필요 없습니다.

**2. 코루틴 친화적 애니메이션**: `animate*AsState`, `Animatable`과 자연스럽게 통합되어 복잡한 애니메이션도 간결하게 표현됩니다.

**3. 타입 안전한 API**: `DrawScope`는 좌표계, 변환, 클리핑 등을 타입 안전하게 제공하며, `Dp`와 `Px` 단위 변환도 명시적으로 처리합니다.

---

## DrawScope의 기본 구조

`DrawScope`는 모든 Compose 드로잉 작업의 기반이 되는 인터페이스입니다. `Canvas` 컴포저블 내부, `Modifier.drawBehind`, `Modifier.drawWithContent`, `Modifier.drawWithCache`에서 이 스코프가 제공됩니다.

```kotlin
// 좌표 원점은 왼쪽 상단 (0, 0)
// X는 오른쪽, Y는 아래쪽으로 증가
Canvas(modifier = Modifier.fillMaxSize()) {
    // size: DrawScope가 그릴 수 있는 영역의 크기 (FloatSize)
    val w = size.width
    val h = size.height
    val center = Offset(w / 2f, h / 2f)

    // drawCircle: 중심, 반지름, 색상
    drawCircle(
        color = Color(0xFF6200EE),
        radius = 100f,
        center = center
    )

    // drawRect: 좌상단 오프셋, 크기, 색상
    drawRect(
        color = Color(0xFF03DAC6),
        topLeft = Offset(50f, 50f),
        size = Size(200f, 100f)
    )

    // drawLine: 시작점, 끝점, 색상, 두께
    drawLine(
        color = Color.Black,
        start = Offset(0f, 0f),
        end = Offset(w, h),
        strokeWidth = 4f
    )
}
```

### Dp에서 Px로 변환

`DrawScope`는 `Density` 인터페이스를 구현하므로, `dp`를 픽셀로 변환할 수 있습니다:

```kotlin
Canvas(modifier = Modifier.fillMaxSize()) {
    val radiusPx = 50.dp.toPx()   // Dp → Float(px)
    val strokePx = 2.dp.toPx()

    drawCircle(
        color = Color.Blue,
        radius = radiusPx,
        style = Stroke(width = strokePx)
    )
}
```

---

## Path로 복잡한 도형 그리기

`Path`는 직선, 곡선, 호(Arc) 등 임의의 복잡한 형태를 정의하는 핵심 도구입니다. Flutter의 `Path`와 유사하지만, Compose에서는 `androidx.compose.ui.graphics.Path`를 사용합니다.

### Path 기본 명령어

| 메서드 | 설명 |
|--------|------|
| `moveTo(x, y)` | 현재 위치 이동 (선 없이) |
| `lineTo(x, y)` | 직선 그리기 |
| `quadraticBezierTo(x1, y1, x2, y2)` | 2차 베지어 곡선 |
| `cubicTo(x1, y1, x2, y2, x3, y3)` | 3차 베지어 곡선 |
| `arcTo(rect, startAngle, sweepAngle, forceMoveTo)` | 타원 호 |
| `close()` | 경로 닫기 |

### 실전 예제 1 — 방사형 차트(Spider Chart)

아래 코드는 6개의 축을 가진 방사형 차트를 Compose Canvas로 구현합니다. 실제 데이터 시각화에 바로 사용할 수 있습니다.

```kotlin
@Composable
fun SpiderChart(
    values: List<Float>,      // 0f ~ 1f 사이 정규화된 값
    labels: List<String>,
    modifier: Modifier = Modifier
) {
    require(values.size == labels.size)
    val count = values.size
    val angleStep = (2 * PI / count).toFloat()

    Canvas(modifier = modifier.aspectRatio(1f)) {
        val radius = size.minDimension / 2f * 0.8f
        val center = Offset(size.width / 2f, size.height / 2f)

        // 1. 축 그리기
        val gridLevels = 5
        for (level in 1..gridLevels) {
            val levelRadius = radius * level / gridLevels
            val gridPath = Path().apply {
                for (i in 0 until count) {
                    val angle = angleStep * i - PI.toFloat() / 2f
                    val x = center.x + levelRadius * cos(angle)
                    val y = center.y + levelRadius * sin(angle)
                    if (i == 0) moveTo(x, y) else lineTo(x, y)
                }
                close()
            }
            drawPath(
                path = gridPath,
                color = Color.LightGray,
                style = Stroke(width = 1.dp.toPx())
            )
        }

        // 2. 축 라인 그리기
        for (i in 0 until count) {
            val angle = angleStep * i - PI.toFloat() / 2f
            val endX = center.x + radius * cos(angle)
            val endY = center.y + radius * sin(angle)
            drawLine(
                color = Color.Gray,
                start = center,
                end = Offset(endX, endY),
                strokeWidth = 1.dp.toPx()
            )
        }

        // 3. 데이터 영역 채우기
        val dataPath = Path().apply {
            for (i in 0 until count) {
                val angle = angleStep * i - PI.toFloat() / 2f
                val r = radius * values[i]
                val x = center.x + r * cos(angle)
                val y = center.y + r * sin(angle)
                if (i == 0) moveTo(x, y) else lineTo(x, y)
            }
            close()
        }
        drawPath(
            path = dataPath,
            color = Color(0x806200EE),
            style = Fill
        )
        drawPath(
            path = dataPath,
            color = Color(0xFF6200EE),
            style = Stroke(width = 2.dp.toPx())
        )

        // 4. 데이터 포인트
        for (i in 0 until count) {
            val angle = angleStep * i - PI.toFloat() / 2f
            val r = radius * values[i]
            drawCircle(
                color = Color(0xFF6200EE),
                radius = 4.dp.toPx(),
                center = Offset(
                    center.x + r * cos(angle),
                    center.y + r * sin(angle)
                )
            )
        }
    }
}

// 사용 예
SpiderChart(
    values = listOf(0.8f, 0.6f, 0.9f, 0.4f, 0.7f, 0.5f),
    labels = listOf("속도", "정확도", "안정성", "효율", "보안", "확장성"),
    modifier = Modifier
        .fillMaxWidth()
        .padding(16.dp)
)
```

---

## drawWithCache로 성능 최적화

`Canvas` 내부에서 `Path`, `Brush`, `Shader` 등을 매번 생성하면 GC 압박이 생깁니다. `Modifier.drawWithCache`는 크기가 바뀌지 않는 한 객체를 캐시해 성능을 높입니다.

```kotlin
val waveModifier = Modifier
    .fillMaxWidth()
    .height(100.dp)
    .drawWithCache {
        // 이 블록은 크기가 바뀔 때만 재실행됨
        val wavePath = Path()
        val w = size.width
        val h = size.height

        onDrawBehind {
            // 이 블록은 상태 변경 시 재실행됨
            wavePath.reset()
            wavePath.moveTo(0f, h * 0.5f)

            val waveLength = w / 3f
            var x = 0f
            while (x <= w) {
                wavePath.quadraticBezierTo(
                    x + waveLength / 4f, h * 0.2f,
                    x + waveLength / 2f, h * 0.5f
                )
                wavePath.quadraticBezierTo(
                    x + waveLength * 3 / 4f, h * 0.8f,
                    x + waveLength, h * 0.5f
                )
                x += waveLength
            }
            wavePath.lineTo(w, h)
            wavePath.lineTo(0f, h)
            wavePath.close()

            drawPath(
                path = wavePath,
                color = Color(0xFF03DAC6)
            )
        }
    }
```

---

## Brush와 Shader로 그라디언트 구현

`Brush`는 단색이 아닌 그라디언트나 이미지로 도형을 채울 수 있게 해줍니다.

```kotlin
Canvas(modifier = Modifier.fillMaxSize()) {
    // LinearGradient
    val linearBrush = Brush.linearGradient(
        colors = listOf(Color(0xFF6200EE), Color(0xFF03DAC6)),
        start = Offset(0f, 0f),
        end = Offset(size.width, size.height)
    )
    drawRect(brush = linearBrush, size = size)

    // RadialGradient
    val radialBrush = Brush.radialGradient(
        colors = listOf(Color.White, Color.Transparent),
        center = Offset(size.width / 2f, size.height / 2f),
        radius = size.minDimension / 2f
    )
    drawCircle(
        brush = radialBrush,
        radius = size.minDimension / 2f
    )

    // SweepGradient (원형 회전 그라디언트)
    val sweepBrush = Brush.sweepGradient(
        colors = listOf(
            Color.Red, Color.Yellow, Color.Green,
            Color.Cyan, Color.Blue, Color.Magenta, Color.Red
        ),
        center = Offset(size.width / 2f, size.height / 2f)
    )
    drawCircle(
        brush = sweepBrush,
        radius = 80.dp.toPx(),
        center = Offset(size.width / 2f, size.height / 2f),
        style = Stroke(width = 20.dp.toPx())
    )
}
```

---

## 실전 예제 2 — 애니메이션 파형 진행률 표시기

상태와 애니메이션을 결합해 실제 앱에서 쓸 수 있는 동적 컴포넌트를 만들어 봅니다.

```kotlin
@Composable
fun WaveProgressIndicator(
    progress: Float,           // 0f ~ 1f
    modifier: Modifier = Modifier,
    waveColor: Color = Color(0xFF6200EE),
    backgroundColor: Color = Color(0xFFEEEEEE)
) {
    // 파도 위상 애니메이션 (무한 반복)
    val infiniteTransition = rememberInfiniteTransition(label = "wave")
    val phase by infiniteTransition.animateFloat(
        initialValue = 0f,
        targetValue = 2 * PI.toFloat(),
        animationSpec = infiniteRepeatable(
            animation = tween(durationMillis = 2000, easing = LinearEasing)
        ),
        label = "phase"
    )

    // progress 변화 애니메이션
    val animatedProgress by animateFloatAsState(
        targetValue = progress,
        animationSpec = tween(durationMillis = 800),
        label = "progress"
    )

    Canvas(modifier = modifier) {
        val w = size.width
        val h = size.height
        val waterLevel = h * (1f - animatedProgress)

        // 배경 원
        drawCircle(
            color = backgroundColor,
            radius = size.minDimension / 2f
        )

        // 물 채우기 클리핑 (원 안에만 그리도록)
        clipPath(
            path = Path().apply {
                addOval(Rect(Offset.Zero, size))
            }
        ) {
            // 파동 경로
            val wavePath = Path().apply {
                moveTo(0f, waterLevel)
                val waveLength = w / 2f
                val amplitude = h * 0.04f
                var x = -waveLength
                while (x <= w) {
                    quadraticBezierTo(
                        x + waveLength / 4f,
                        waterLevel - amplitude * sin(phase + x / waveLength * 2 * PI.toFloat()),
                        x + waveLength / 2f,
                        waterLevel
                    )
                    quadraticBezierTo(
                        x + waveLength * 3 / 4f,
                        waterLevel + amplitude * sin(phase + x / waveLength * 2 * PI.toFloat()),
                        x + waveLength,
                        waterLevel
                    )
                    x += waveLength
                }
                lineTo(w, h)
                lineTo(0f, h)
                close()
            }
            drawPath(path = wavePath, color = waveColor.copy(alpha = 0.6f))

            // 두 번째 파동 (위상 차이를 줘서 깊이감 표현)
            val wavePath2 = Path().apply {
                moveTo(0f, waterLevel + 5f)
                val waveLength = w / 2.5f
                val amplitude = h * 0.03f
                var x = -waveLength
                while (x <= w) {
                    quadraticBezierTo(
                        x + waveLength / 4f,
                        waterLevel + 5f + amplitude * sin(phase * 1.2f + x / waveLength * 2 * PI.toFloat()),
                        x + waveLength / 2f,
                        waterLevel + 5f
                    )
                    quadraticBezierTo(
                        x + waveLength * 3 / 4f,
                        waterLevel + 5f - amplitude * sin(phase * 1.2f + x / waveLength * 2 * PI.toFloat()),
                        x + waveLength,
                        waterLevel + 5f
                    )
                    x += waveLength
                }
                lineTo(w, h)
                lineTo(0f, h)
                close()
            }
            drawPath(path = wavePath2, color = waveColor.copy(alpha = 0.8f))
        }

        // 원 테두리
        drawCircle(
            color = waveColor,
            radius = size.minDimension / 2f - 2.dp.toPx(),
            style = Stroke(width = 2.dp.toPx())
        )
    }
}

// 사용 예
@Composable
fun ProgressDemo() {
    var progress by remember { mutableStateOf(0.3f) }
    Column(
        modifier = Modifier.fillMaxSize().padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        WaveProgressIndicator(
            progress = progress,
            modifier = Modifier.size(200.dp)
        )
        Spacer(Modifier.height(24.dp))
        Slider(
            value = progress,
            onValueChange = { progress = it },
            modifier = Modifier.fillMaxWidth()
        )
    }
}
```

---

## 변환(Transformation) 활용

`DrawScope`는 스코프 기반 변환을 지원합니다. 변환은 해당 블록에만 적용되며, 블록이 끝나면 자동으로 원래 상태로 복원됩니다.

```kotlin
Canvas(modifier = Modifier.fillMaxSize()) {
    val center = Offset(size.width / 2f, size.height / 2f)

    // rotate: 중심점을 기준으로 회전
    rotate(degrees = 45f, pivot = center) {
        drawRect(
            color = Color.Red,
            topLeft = center - Offset(50f, 50f),
            size = Size(100f, 100f)
        )
    }

    // scale: 특정 지점 기준으로 확대/축소
    scale(scaleX = 1.5f, scaleY = 0.8f, pivot = center) {
        drawOval(
            color = Color.Blue.copy(alpha = 0.5f),
            topLeft = center - Offset(60f, 40f),
            size = Size(120f, 80f)
        )
    }

    // translate: 이동 후 그리기
    translate(left = -100f, top = -50f) {
        drawCircle(color = Color.Green, radius = 30f, center = center)
    }

    // withTransform: 여러 변환을 한 번에 적용 (더 효율적)
    withTransform({
        rotate(30f, center)
        translate(50f, 0f)
    }) {
        drawRect(
            color = Color.Yellow,
            topLeft = center,
            size = Size(80f, 40f)
        )
    }
}
```

---

## BlendMode로 합성 효과 구현

`BlendMode`는 이미 그려진 내용(destination)과 새로 그릴 내용(source)을 합성하는 방식을 결정합니다. Photoshop의 레이어 혼합 모드와 유사합니다.

```kotlin
Canvas(modifier = Modifier.fillMaxSize()) {
    // 배경 빨간 원 (destination)
    drawCircle(
        color = Color.Red,
        radius = 100f,
        center = Offset(size.width / 2f - 40f, size.height / 2f)
    )

    // Multiply 블렌드: 두 색상의 어두운 부분이 강조됨
    drawCircle(
        color = Color.Blue,
        radius = 100f,
        center = Offset(size.width / 2f + 40f, size.height / 2f),
        blendMode = BlendMode.Multiply
    )
}

// 마스크 효과: DstIn을 활용한 알파 마스킹
Canvas(modifier = Modifier.size(200.dp)) {
    // offscreen 버퍼에 그린 후 합성 (BlendMode를 올바르게 적용하기 위해 필요)
    drawIntoCanvas { canvas ->
        canvas.saveLayer(Rect(Offset.Zero, size), Paint())

        // 1단계: 이미지나 그라디언트 그리기 (destination)
        drawRect(
            brush = Brush.horizontalGradient(listOf(Color.Red, Color.Blue)),
            size = size
        )

        // 2단계: 마스크 도형 그리기 (DstIn: destination의 마스크 안에만 남김)
        drawCircle(
            color = Color.Black,
            radius = size.minDimension / 2f,
            blendMode = BlendMode.DstIn
        )

        canvas.restore()
    }
}
```

---

## Modifier.drawBehind vs Modifier.drawWithContent

| | `drawBehind` | `drawWithContent` |
|---|---|---|
| 콘텐츠 위치 | 콘텐츠 뒤에 그림 | 콘텐츠 앞/뒤 모두 가능 |
| 콘텐츠 호출 | 자동 | `drawContent()` 직접 호출 |
| 용도 | 배경 장식 | 오버레이, 하이라이트 |

```kotlin
// 텍스트 뒤에 강조 배경 그리기
Text(
    text = "중요한 텍스트",
    modifier = Modifier.drawBehind {
        drawRoundRect(
            color = Color.Yellow,
            cornerRadius = CornerRadius(8.dp.toPx()),
            topLeft = Offset(-4.dp.toPx(), -2.dp.toPx()),
            size = Size(size.width + 8.dp.toPx(), size.height + 4.dp.toPx())
        )
    }
)

// 콘텐츠 위에 스크래치 오버레이 그리기
Box(
    modifier = Modifier
        .size(200.dp)
        .drawWithContent {
            drawContent()   // 먼저 콘텐츠를 그리고
            drawRect(       // 그 위에 반투명 오버레이
                color = Color.Black.copy(alpha = 0.3f),
                size = size
            )
        }
)
```

---

## 주의사항 및 성능 팁

### 1. Path 객체 재사용
`DrawScope` 람다는 리컴포지션마다 호출됩니다. `Path`, `Paint` 같은 객체를 람다 내부에서 생성하면 GC 부담이 증가합니다.

```kotlin
// 나쁜 예: 매 드로우마다 Path 생성
Canvas(modifier = Modifier.fillMaxSize()) {
    val path = Path()  // 매번 생성됨!
    path.moveTo(0f, 0f)
    // ...
}

// 좋은 예: drawWithCache로 캐시 활용
Modifier.drawWithCache {
    val path = Path()  // 크기 변경 시에만 재생성
    onDrawBehind {
        path.reset()
        path.moveTo(0f, 0f)
        // ...
    }
}
```

### 2. clipPath 오버헤드 이해
`clipPath`는 소프트웨어 렌더링 경로를 사용할 수 있어 하드웨어 가속 `clipRect`보다 느릴 수 있습니다. 직사각형 클리핑은 `clipRect`를 우선적으로 사용하세요.

### 3. graphicsLayer로 합성 최적화
복잡한 `BlendMode` 효과는 `Modifier.graphicsLayer`와 `compositingStrategy = CompositingStrategy.Offscreen`을 함께 사용해 올바른 알파 합성을 보장하세요.

```kotlin
Box(
    modifier = Modifier
        .size(200.dp)
        .graphicsLayer {
            compositingStrategy = CompositingStrategy.Offscreen
        }
        .drawWithContent {
            drawContent()
            drawRect(
                color = Color.Transparent,
                blendMode = BlendMode.DstIn,
                brush = Brush.verticalGradient(
                    listOf(Color.Black, Color.Transparent)
                )
            )
        }
)
```

### 4. TextMeasurer 사용 시 remember 필수

Canvas 내에서 텍스트를 그릴 때 `TextMeasurer`는 반드시 `rememberTextMeasurer()`로 생성해 재사용해야 합니다.

```kotlin
val textMeasurer = rememberTextMeasurer()
Canvas(modifier = Modifier.fillMaxSize()) {
    val textLayout = textMeasurer.measure(
        text = "85%",
        style = TextStyle(fontSize = 24.sp, fontWeight = FontWeight.Bold)
    )
    drawText(
        textLayoutResult = textLayout,
        topLeft = Offset(
            (size.width - textLayout.size.width) / 2f,
            (size.height - textLayout.size.height) / 2f
        )
    )
}
```

### 5. 애니메이션에서 State 읽기 최적화
`Canvas` 블록에서 상태를 직접 읽으면 해당 Canvas만 재드로우됩니다. 반면 `Canvas` 밖에서 상태를 읽으면 부모 컴포저블까지 리컴포지션이 발생합니다. 가능한 한 상태를 `Canvas` 내부에서 읽어 범위를 최소화하세요.

---

## 마치며

Jetpack Compose Canvas와 `DrawScope`는 View 시스템의 `onDraw`보다 훨씬 현대적이고 강력한 그래픽 API를 제공합니다. 상태 기반 선언형 패러다임과 자연스럽게 통합되며, `drawWithCache`와 `graphicsLayer` 같은 최적화 도구를 함께 활용하면 성능도 놓치지 않을 수 있습니다.

복잡한 차트, 게임 UI, 물결 애니메이션, 커스텀 로딩 인디케이터 등 기존 컴포넌트로는 표현하기 어려운 그래픽을 직접 픽셀 단위로 제어하고 싶다면, 오늘 살펴본 API들을 토대로 시작해 보시기 바랍니다.

---

## 참고 자료

- [Graphics in Compose — Android Developers](https://developer.android.com/develop/ui/compose/graphics/draw/overview)
- [Graphics Modifiers — Android Developers](https://developer.android.com/develop/ui/compose/graphics/draw/modifiers)
