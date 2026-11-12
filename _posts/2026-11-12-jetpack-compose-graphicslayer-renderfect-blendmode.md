---
layout: post
title: "Jetpack Compose graphicsLayer 심화: 하드웨어 가속 레이어·3D 변환·BlendMode·RenderEffect로 고성능 비주얼 이펙트 완전 정복"
date: 2026-11-12
categories: [android, flutter]
tags: [android, jetpack-compose, graphicsLayer, RenderEffect, BlendMode, CompositingStrategy, 성능최적화]
---

Jetpack Compose에서 UI를 단순히 배치하고 그리는 것을 넘어서, **GPU 레이어를 직접 제어**하고 싶다면 `Modifier.graphicsLayer`를 깊이 이해해야 합니다. 이 글에서는 graphicsLayer가 내부적으로 어떻게 동작하는지, 3D 변환·BlendMode·CompositingStrategy·RenderEffect를 어떻게 조합해야 하는지를 실제 코드 예제와 함께 완전히 정복합니다.

---

## 1. graphicsLayer란 무엇인가?

`Modifier.graphicsLayer`는 컴포저블의 드로잉 명령을 **RenderNode**(하드웨어 가속 레이어)에 기록하는 Modifier입니다. 이 레이어는 Android의 하드웨어 렌더링 파이프라인에서 독립적으로 관리되며, 다음과 같은 핵심 역할을 합니다.

- **드로잉 명령 격리**: 레이어 내부의 드로잉 명령은 외부 변화와 무관하게 재실행 없이 재합성될 수 있습니다.
- **변환 적용**: scaleX/Y, translationX/Y, rotationX/Y/Z 등의 변환을 GPU에서 직접 처리합니다.
- **오프스크린 버퍼**: BlendMode 또는 RenderEffect 사용 시 컴포저블 전체를 비트맵으로 래스터화해 처리합니다.

중요한 점은 `graphicsLayer`는 **레이아웃 단계에 영향을 주지 않는다**는 것입니다. 즉, `scaleX = 2f`로 화면에는 두 배로 보여도 주변 컴포저블의 측정·배치에는 전혀 영향이 없습니다.

---

## 2. 왜 graphicsLayer를 써야 하는가?

### 성능: 애니메이션과 recomposition 분리

Compose에서 애니메이션을 구현할 때 가장 중요한 최적화 중 하나는 **recomposition 없이 변환을 적용하는 것**입니다. `graphicsLayer`의 람다 버전을 사용하면 애니메이션 값이 바뀔 때 recomposition이 일어나지 않고 레이어 프로퍼티만 갱신됩니다.

```kotlin
// ❌ 나쁜 예: animatedAlpha 변경마다 recomposition 발생
val alpha by animateFloatAsState(if (visible) 1f else 0f)
Box(modifier = Modifier.alpha(alpha)) { ... }

// ✅ 좋은 예: 람다 graphicsLayer로 recomposition 없이 GPU 레이어만 갱신
val alpha by animateFloatAsState(if (visible) 1f else 0f)
Box(modifier = Modifier.graphicsLayer { this.alpha = alpha }) { ... }
```

### BlendMode: 픽셀 합성 제어

BlendMode는 현재 레이어의 픽셀을 화면에 이미 그려진 픽셀과 어떻게 합성할지를 결정합니다. 기본값은 `BlendMode.SrcOver`(위에 그리기)이며, `BlendMode.Clear`를 사용하면 해당 영역의 픽셀을 완전히 지울 수 있습니다.

단, BlendMode가 `SrcOver`가 아닌 경우, `graphicsLayer`는 **반드시 오프스크린 버퍼**를 생성합니다. 오프스크린 버퍼 없이 BlendMode를 사용하면 의도치 않게 화면 전체(배경·다른 앱까지)에 BlendMode가 적용됩니다.

---

## 3. 실제 구현 예제

### 예제 1: CompositingStrategy.Offscreen + BlendMode로 원형 프로필 이미지 구현

소셜 앱에서 흔히 볼 수 있는 **온라인 상태 뱃지가 있는 원형 프로필 이미지**를 구현합니다. 핵심은 `BlendMode.Clear`로 뱃지가 위치할 영역의 픽셀을 완전히 제거한 뒤 그 위에 초록 원을 그리는 것입니다.

```kotlin
@Composable
fun ProfileImageWithBadge(
    imageUrl: String,
    isOnline: Boolean,
    modifier: Modifier = Modifier,
) {
    val badgeSize = 20.dp

    Box(
        modifier = modifier.size(64.dp),
        contentAlignment = Alignment.BottomEnd
    ) {
        AsyncImage(
            model = imageUrl,
            contentDescription = "프로필 이미지",
            contentScale = ContentScale.Crop,
            modifier = Modifier
                .fillMaxSize()
                // 반드시 Offscreen으로 BlendMode가 이 레이어 내에서만 작동하도록 격리
                .graphicsLayer {
                    compositingStrategy = CompositingStrategy.Offscreen
                }
                .drawWithCache {
                    // 원형 클리핑 경로
                    val circlePath = Path().apply {
                        addOval(Rect(Offset.Zero, Size(size.width, size.height)))
                    }
                    // 뱃지 영역 제거 경로 (오른쪽 하단)
                    val badgeClearPath = Path().apply {
                        val radius = badgeSize.toPx() / 2f + 2.dp.toPx()
                        addOval(
                            Rect(
                                center = Offset(size.width, size.height),
                                radius = radius
                            )
                        )
                    }
                    onDrawWithContent {
                        // 1. 원형으로 클리핑해 이미지 그리기
                        clipPath(circlePath) {
                            this@onDrawWithContent.drawContent()
                        }
                        // 2. 뱃지 자리 픽셀 제거 (BlendMode.Clear)
                        if (isOnline) {
                            drawPath(
                                path = badgeClearPath,
                                color = Color.Black,
                                blendMode = BlendMode.Clear
                            )
                        }
                    }
                }
        )

        // 3. 온라인 상태 뱃지 (초록 원)
        if (isOnline) {
            Box(
                modifier = Modifier
                    .size(badgeSize)
                    .clip(CircleShape)
                    .background(Color(0xFF4CAF50))
                    .border(2.dp, Color.White, CircleShape)
            )
        }
    }
}
```

**핵심 포인트**: `CompositingStrategy.Offscreen`이 없으면 `BlendMode.Clear`가 이 컴포저블의 경계 밖, 즉 화면 배경까지 투명하게 만들어버립니다. Offscreen을 사용해 먼저 이 컴포저블의 내용을 별도 버퍼에 그린 뒤 화면에 합성해야 합니다.

---

### 예제 2: RenderEffect + BlurEffect로 Frosted Glass 효과 구현

iOS의 Frosted Glass(반투명 블러 배경) 효과를 Android 12 이상에서 `BlurEffect`로 구현합니다. `RenderEffect.isSupported()`로 플랫폼 지원 여부를 반드시 확인해야 합니다.

```kotlin
@Composable
fun FrostedGlassCard(
    content: @Composable () -> Unit,
    modifier: Modifier = Modifier,
) {
    val blurEffect = remember {
        BlurEffect(radiusX = 20f, radiusY = 20f, edgeTreatment = TileMode.Clamp)
    }
    val isBlurSupported = remember { blurEffect.isSupported() }

    Box(modifier = modifier) {
        // 1. 배경 콘텐츠 (블러 처리)
        Box(
            modifier = Modifier
                .matchParentSize()
                .then(
                    if (isBlurSupported) {
                        // Android 12+(API 31) 이상: GPU 하드웨어 블러
                        Modifier.graphicsLayer {
                            renderEffect = blurEffect
                        }
                    } else {
                        // 하위 버전 폴백: 반투명 오버레이로 대체
                        Modifier.alpha(0f)
                    }
                )
        ) {
            content()
        }

        // 2. 반투명 오버레이 (흰색 or 색조)
        Box(
            modifier = Modifier
                .matchParentSize()
                .background(Color.White.copy(alpha = 0.3f))
        )

        // 3. 실제 카드 콘텐츠
        Box(modifier = Modifier.padding(16.dp)) {
            content()
        }
    }
}

// 사용 예시
@Composable
fun WeatherCard() {
    Box(
        modifier = Modifier
            .fillMaxWidth()
            .height(200.dp)
            .background(
                Brush.linearGradient(
                    listOf(Color(0xFF1A237E), Color(0xFF7B1FA2))
                )
            )
    ) {
        // 배경 이미지나 그라디언트 위에 Frosted Glass 카드
        FrostedGlassCard(
            modifier = Modifier
                .align(Alignment.Center)
                .width(280.dp)
                .height(120.dp)
                .clip(RoundedCornerShape(16.dp)),
            content = {
                Box(
                    modifier = Modifier
                        .fillMaxSize()
                        .background(
                            Brush.linearGradient(
                                listOf(Color(0xFF7B1FA2), Color(0xFF1A237E))
                            )
                        )
                )
            }
        ) {
            Column(
                modifier = Modifier.fillMaxSize().padding(16.dp),
                verticalArrangement = Arrangement.Center
            ) {
                Text(
                    text = "서울",
                    color = Color.White,
                    style = MaterialTheme.typography.headlineMedium,
                    fontWeight = FontWeight.Bold
                )
                Text(
                    text = "맑음 · 23°C",
                    color = Color.White.copy(alpha = 0.8f),
                    style = MaterialTheme.typography.bodyLarge
                )
            }
        }
    }
}
```

**RenderEffect 체이닝**: `BlurEffect`의 첫 번째 파라미터로 다른 `RenderEffect`를 넘기면 효과를 연쇄 적용할 수 있습니다. 예를 들어 `BlurEffect(renderEffect = OffsetEffect(null, 10f, 10f), radiusX = 15f, radiusY = 15f)`는 오프셋 적용 후 블러를 처리합니다.

---

## 4. 주요 graphicsLayer 프로퍼티 정리

| 프로퍼티 | 타입 | 설명 |
|---|---|---|
| `scaleX`, `scaleY` | Float | 수평/수직 스케일 (1f = 원본) |
| `translationX`, `translationY` | Float | 픽셀 단위 이동 |
| `rotationX`, `rotationY`, `rotationZ` | Float | 도(degree) 단위 3D 회전 |
| `transformOrigin` | TransformOrigin | 변환 기준점 (기본: 중앙) |
| `alpha` | Float | 투명도 (0f~1f) |
| `clip` | Boolean | Shape에 따른 클리핑 여부 |
| `shape` | Shape | 클리핑 형태 |
| `shadowElevation` | Float | 그림자 깊이 |
| `renderEffect` | RenderEffect? | 시각 효과 (API 31+) |
| `compositingStrategy` | CompositingStrategy | 합성 전략 |

---

## 5. 주의사항 및 성능 팁

### 오프스크린 버퍼는 비용이 있다

`alpha < 1.0f`, `RenderEffect` 설정, `CompositingStrategy.Offscreen` 사용 시 GPU가 컴포저블 크기의 오프스크린 텍스처를 생성합니다. 이 텍스처는 VRAM을 사용하므로 대형 컴포저블에 무분별하게 적용하면 메모리 압박이 생길 수 있습니다.

### BlurEffect는 API 31 이상에서만 동작

`BlurEffect`는 Android 12(API 31) 이상에서만 지원됩니다. `isSupported()`를 반드시 확인하고 하위 버전 폴백을 제공하세요.

```kotlin
val blurEffect = BlurEffect(radiusX = 25f, radiusY = 25f)
if (blurEffect.isSupported()) {
    Modifier.graphicsLayer { renderEffect = blurEffect }
} else {
    Modifier  // 폴백: 아무 효과 없이 진행
}
```

### 람다 버전 graphicsLayer를 사용할 것

`Modifier.graphicsLayer { ... }` 람다 버전을 사용하면 State 변경 시 recomposition 없이 그래픽 레이어 단에서만 처리되어 성능이 크게 향상됩니다. `animateFloatAsState` 등과 함께 사용할 때 특히 효과적입니다.

### LayerOutsets로 클리핑 문제 해결

오프스크린 버퍼를 사용할 때 그림자나 글로우 효과가 컴포저블 경계에서 잘릴 수 있습니다. `outsets = LayerOutsets(24.dp)` 파라미터로 버퍼를 확장해 경계 너머의 효과도 렌더링할 수 있습니다 (Compose 1.12.0-alpha01+).

```kotlin
Modifier.graphicsLayer(
    alpha = 0.9f,
    outsets = LayerOutsets(all = 24.dp),
    clip = false
)
```

### rememberGraphicsLayer로 비트맵 캡처

`rememberGraphicsLayer()`와 `drawWithContent`를 조합하면 컴포저블의 현재 상태를 비트맵으로 캡처할 수 있습니다. 공유하기 기능이나 썸네일 생성에 활용할 수 있습니다 (Compose 1.7.0-alpha07+).

---

## 6. 정리

`Modifier.graphicsLayer`는 Jetpack Compose에서 하드웨어 가속 GPU 레이어를 직접 제어하는 강력한 도구입니다. 단순한 alpha나 scale 조정부터 BlendMode를 활용한 복잡한 픽셀 합성, RenderEffect를 이용한 블러·색상 필터까지 폭넓게 활용할 수 있습니다. 핵심은 세 가지입니다.

1. **recomposition 없이 애니메이션하려면 람다 버전 graphicsLayer를 쓴다**
2. **BlendMode 사용 시 반드시 CompositingStrategy.Offscreen으로 격리한다**
3. **RenderEffect는 API 31+ 전용이므로 isSupported()로 가드한다**

이 세 가지 원칙을 지키면 성능을 희생하지 않으면서도 네이티브 앱 수준의 비주얼 이펙트를 구현할 수 있습니다.

---

## 참고 자료
- [Compose 그래픽 Modifier 공식 가이드 (Android Developers)](https://developer.android.com/develop/ui/compose/graphics/draw/modifiers)
- [RenderEffect API Reference (Android Developers)](https://developer.android.com/reference/kotlin/androidx/compose/ui/graphics/RenderEffect)
- [BlurEffect API Reference (Android Developers)](https://developer.android.com/reference/kotlin/androidx/compose/ui/graphics/BlurEffect)
