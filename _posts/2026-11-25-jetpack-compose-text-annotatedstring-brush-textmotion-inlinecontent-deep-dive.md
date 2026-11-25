---
layout: post
title: "Jetpack Compose Text 심화: AnnotatedString·Brush·TextMotion·InlineContent로 고급 텍스트 렌더링 완전 정복"
date: 2026-11-25
categories: [android, flutter]
tags: [android, jetpack-compose, text, annotatedstring, brush, textmotion, inlinecontent, kotlin]
---

Jetpack Compose에서 `Text`는 가장 자주 쓰이는 컴포저블이지만, 그 내부에는 `AnnotatedString`, `Brush`, `TextMotion`, `InlineContent` 등 대부분의 개발자가 완전히 활용하지 못하는 강력한 고급 API가 있습니다. 이 아티클에서는 단순 텍스트 출력을 넘어 복잡한 멀티 스타일, 그라디언트 효과, 애니메이션 최적화, 텍스트 내 위젯 삽입까지 완벽하게 다룹니다.

## 왜 고급 텍스트 API가 필요한가?

UI 개발에서 텍스트는 단순한 정보 전달 수단이 아닙니다. 쇼핑앱의 상품 이름에는 "NEW" 배지가 함께 붙고, 소셜 앱의 멘션(`@user`)은 색상이 다르며 클릭이 가능해야 합니다. 대시보드 헤드라인에는 그라디언트 효과가 들어가고, 튜토리얼 텍스트는 아이콘과 텍스트가 한 줄에 혼합됩니다.

Android View 시스템에서는 이런 요구사항을 `SpannableString`, `Html.fromHtml()`, `ImageSpan` 등을 조합해 구현했습니다. Jetpack Compose는 이를 타입 안전하고 선언적인 API로 대체합니다.

또한 텍스트를 애니메이션할 때 발생하는 "계단 현상(jerky animation)"은 기본 렌더링 힌팅 때문인데, `TextMotion` API로 이를 제어할 수 있습니다. 이런 세부 사항을 모르면 불필요한 커스텀 Canvas 구현을 만들게 됩니다.

---

## 1. AnnotatedString 완전 정복

### 구조 이해

`AnnotatedString`은 세 가지로 구성됩니다:

- **text**: 원본 문자열
- **spanStyles**: `SpanStyle` 범위 목록 (인라인 스타일 — 색상, 폰트, 데코레이션)
- **paragraphStyles**: `ParagraphStyle` 범위 목록 (단락 정렬, 줄 높이, 들여쓰기)

이 구조는 Android View의 `SpannableString`과 유사하지만, 타입 안전하고 `buildAnnotatedString` DSL로 선언적으로 구성할 수 있습니다.

### buildAnnotatedString DSL 실전 예제

```kotlin
@Composable
fun AnnotatedStringDemo() {
    val annotatedText = buildAnnotatedString {
        // 1. 일반 텍스트
        append("안녕하세요! ")

        // 2. 굵고 붉은 텍스트 (SpanStyle)
        withStyle(
            style = SpanStyle(
                fontWeight = FontWeight.Bold,
                color = Color(0xFFE53935),
                fontSize = 20.sp
            )
        ) {
            append("Jetpack Compose")
        }

        append("로 만드는 ")

        // 3. 파란색 밑줄 텍스트 (링크 스타일)
        withStyle(
            style = SpanStyle(
                color = Color(0xFF1E88E5),
                textDecoration = TextDecoration.Underline
            )
        ) {
            append("고급 텍스트")
        }

        // 4. 단락 구분 후 중앙 정렬 (ParagraphStyle)
        append("\n")
        withStyle(style = ParagraphStyle(textAlign = TextAlign.Center)) {
            withStyle(style = SpanStyle(fontSize = 12.sp, color = Color.Gray)) {
                append("위 텍스트는 여러 SpanStyle이 조합된 AnnotatedString입니다.")
            }
        }
    }

    // SelectionContainer: 사용자가 텍스트를 선택하고 복사할 수 있게 함
    SelectionContainer {
        Text(
            text = annotatedText,
            modifier = Modifier.padding(16.dp)
        )
    }
}
```

`SelectionContainer`로 감싸지 않으면 기본적으로 텍스트 선택이 비활성화됩니다. 복사를 허용해야 하는 콘텐츠(이메일, 전화번호, 코드 스니펫)에는 반드시 추가하세요.

### 클릭 가능한 링크 처리 (Compose 1.7+)

Compose 1.7 이후 `LinkAnnotation.Url`을 사용하면 `ClickableText`(deprecated) 없이 클릭 가능한 링크를 처리할 수 있습니다:

```kotlin
@Composable
fun ClickableLinkText() {
    val annotatedText = buildAnnotatedString {
        append("공식 문서는 ")

        // pushLink로 URL 링크 범위 추가
        pushLink(
            LinkAnnotation.Url(
                url = "https://developer.android.com/compose",
                styles = TextLinkStyles(
                    // 기본 상태
                    style = SpanStyle(
                        color = Color(0xFF1565C0),
                        textDecoration = TextDecoration.Underline,
                        fontWeight = FontWeight.Medium
                    ),
                    // 호버 상태 (포인터 디바이스에서)
                    hoveredStyle = SpanStyle(
                        color = Color(0xFF0D47A1),
                        background = Color(0x1A1565C0)
                    ),
                    // 눌린 상태
                    pressedStyle = SpanStyle(
                        color = Color(0xFF0D47A1),
                        background = Color(0x331565C0)
                    )
                )
            )
        )
        append("여기")
        pop()

        append("를 참고하세요.")
    }

    Text(
        text = annotatedText,
        modifier = Modifier.padding(16.dp)
    )
}
```

Compose 1.8에서는 `LinkAnnotation.Clickable`도 추가되어, URL이 아닌 임의의 클릭 처리(예: 멘션, 해시태그)도 같은 방식으로 처리할 수 있습니다.

---

## 2. Brush API로 그라디언트 텍스트 구현

### 전체 텍스트 그라디언트

`TextStyle`의 `brush` 파라미터에 `Brush` 인스턴스를 전달하면 텍스트 전체에 그라디언트가 적용됩니다:

```kotlin
@Composable
fun GradientTextDemo() {
    // 선형 그라디언트
    val linearBrush = Brush.linearGradient(
        colors = listOf(
            Color(0xFFE040FB), // 보라
            Color(0xFF40C4FF), // 하늘
            Color(0xFF69F0AE)  // 초록
        )
    )

    Column(modifier = Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        // 전체 텍스트 그라디언트
        Text(
            text = "그라디언트 헤드라인",
            style = TextStyle(
                fontSize = 32.sp,
                fontWeight = FontWeight.ExtraBold,
                brush = linearBrush
            )
        )

        // 방사형 그라디언트
        Text(
            text = "방사형 효과",
            style = TextStyle(
                fontSize = 24.sp,
                fontWeight = FontWeight.Bold,
                brush = Brush.radialGradient(
                    colors = listOf(Color(0xFFFFEB3B), Color(0xFFF44336)),
                    radius = 200f
                )
            )
        )
    }
}
```

### 부분 그라디언트 + 투명도 조합

`SpanStyle`의 `brush`와 `alpha`를 함께 사용하면 텍스트의 특정 범위에만 그라디언트를 적용하고 투명도도 조절할 수 있습니다:

```kotlin
@Composable
fun PartialGradientText() {
    val rainbowColors = listOf(
        Color(0xFFE53935), Color(0xFFF4511E), Color(0xFFFFB300),
        Color(0xFF43A047), Color(0xFF1E88E5), Color(0xFF8E24AA)
    )
    val brush = Brush.linearGradient(colors = rainbowColors)

    Text(
        text = buildAnnotatedString {
            withStyle(SpanStyle(color = Color.DarkGray, fontSize = 16.sp)) {
                append("일반 텍스트 + ")
            }
            // 완전 불투명 그라디언트
            withStyle(SpanStyle(brush = brush, alpha = 1f, fontSize = 20.sp, fontWeight = FontWeight.Bold)) {
                append("무지개 그라디언트")
            }
            // 반투명 그라디언트
            withStyle(SpanStyle(brush = brush, alpha = 0.35f, fontSize = 16.sp)) {
                append(" + 반투명 효과")
            }
        },
        modifier = Modifier.padding(16.dp)
    )
}
```

> **팁**: `Brush.radialGradient()`, `Brush.sweepGradient()`도 동일하게 사용할 수 있습니다. `tileMode` 파라미터로 그라디언트 반복 패턴을 지정할 수 있으며, `TextField`에도 `TextStyle(brush = ...)` 형태로 적용됩니다.

### 애니메이션 그라디언트

`rememberInfiniteTransition`으로 흐르는 그라디언트를 만들 수 있습니다:

```kotlin
@Composable
fun AnimatedGradientText(text: String) {
    val infiniteTransition = rememberInfiniteTransition(label = "gradient")
    val offset by infiniteTransition.animateFloat(
        initialValue = 0f,
        targetValue = 1000f,
        animationSpec = infiniteRepeatable(
            animation = tween(durationMillis = 2500, easing = LinearEasing),
            repeatMode = RepeatMode.Reverse
        ),
        label = "gradient_offset"
    )

    val animatedBrush = Brush.linearGradient(
        colors = listOf(
            Color(0xFF7C4DFF), Color(0xFF00BCD4),
            Color(0xFF7C4DFF), Color(0xFF00BCD4)
        ),
        start = Offset(offset, 0f),
        end = Offset(offset + 600f, 0f)
    )

    Text(
        text = text,
        style = TextStyle(
            fontSize = 28.sp,
            fontWeight = FontWeight.Bold,
            brush = animatedBrush
        )
    )
}
```

---

## 3. TextMotion으로 텍스트 애니메이션 최적화

### 왜 계단 현상이 생기는가?

Compose 기본 텍스트 렌더링은 **픽셀 그리드 힌팅(pixel grid hinting)**을 적용합니다. 이는 가독성을 위해 글자를 가장 가까운 픽셀에 스냅시키는 최적화입니다. 정적 텍스트에서는 유용하지만, scale/translate/rotate 애니메이션 중에는 텍스트가 뚝뚝 끊어지는 현상이 발생합니다.

`TextMotion.Animated`는 이 힌팅을 비활성화하여 연속적이고 부드러운 애니메이션을 만들어 줍니다:

```kotlin
@Composable
fun AnimatedTextScaleDemo() {
    var enlarged by remember { mutableStateOf(false) }
    val scale by animateFloatAsState(
        targetValue = if (enlarged) 2.5f else 1f,
        animationSpec = spring(
            dampingRatio = Spring.DampingRatioMediumBouncy,
            stiffness = Spring.StiffnessLow
        ),
        label = "text_scale"
    )
    val alpha by animateFloatAsState(
        targetValue = if (enlarged) 1f else 0.7f,
        label = "text_alpha"
    )

    Box(
        modifier = Modifier
            .fillMaxWidth()
            .height(200.dp),
        contentAlignment = Alignment.Center
    ) {
        Text(
            text = "탭해서 확대!",
            style = TextStyle(
                fontSize = 20.sp,
                fontWeight = FontWeight.Bold,
                color = Color(0xFF1A237E),
                // 핵심: 애니메이션 모드로 설정
                textMotion = TextMotion.Animated
            ),
            modifier = Modifier
                .graphicsLayer {
                    scaleX = scale
                    scaleY = scale
                    this.alpha = alpha
                }
                .clickable { enlarged = !enlarged }
        )
    }
}
```

### Static vs Animated 비교

| 속성 | `TextMotion.Static` (기본) | `TextMotion.Animated` |
|------|--------------------------|----------------------|
| 픽셀 힌팅 | 활성화 | 비활성화 |
| 가독성 | 높음 (특히 저 DPI) | 약간 낮음 |
| 애니메이션 품질 | 계단 현상 | 부드러움 |
| 권장 상황 | 정적 텍스트 | scale/translate/rotate 애니메이션 |

> **실전 팁**: DPI가 낮은 기기에서 `TextMotion.Animated`는 가독성 저하가 눈에 띌 수 있습니다. `LocalDensity.current.density >= 2.0f` 조건으로 고 DPI 기기에서만 Animated를 적용하는 방어 코드를 추가하세요.

---

## 4. InlineContent로 텍스트 안에 컴포저블 삽입

### Placeholder와 InlineTextContent

`InlineTextContent`는 텍스트 흐름 안에 임의의 Composable을 삽입합니다. 이를 위해 두 단계가 필요합니다:

1. `AnnotatedString`에 `appendInlineContent(id, alternateText)`로 자리 표시자 추가
2. `inlineContent` 맵에 해당 `id`를 키로 `InlineTextContent` 정의

```kotlin
@Composable
fun InlineContentDemo() {
    val starId = "inline_star"
    val badgeId = "inline_badge_new"

    val annotatedText = buildAnnotatedString {
        append("이 기능은 ")
        appendInlineContent(starId, "[★]")  // alternateText: 접근성 대체 텍스트
        append(" 최고 평점이며 ")
        appendInlineContent(badgeId, "[NEW]")
        append(" 새롭게 추가되었습니다.")
    }

    val inlineContent = mapOf(
        starId to InlineTextContent(
            placeholder = Placeholder(
                width = 20.sp,   // sp 단위 필수! dp 사용 시 접근성 폰트 크기 변경에 비율 어긋남
                height = 20.sp,
                placeholderVerticalAlign = PlaceholderVerticalAlign.Center
            )
        ) {
            Icon(
                imageVector = Icons.Filled.Star,
                contentDescription = "별점",
                tint = Color(0xFFFFC107),
                modifier = Modifier.fillMaxSize()
            )
        },

        badgeId to InlineTextContent(
            placeholder = Placeholder(
                width = 42.sp,
                height = 18.sp,
                placeholderVerticalAlign = PlaceholderVerticalAlign.TextCenter
            )
        ) {
            Box(
                modifier = Modifier
                    .fillMaxSize()
                    .background(
                        color = Color(0xFFE53935),
                        shape = RoundedCornerShape(4.dp)
                    ),
                contentAlignment = Alignment.Center
            ) {
                Text(
                    text = "NEW",
                    color = Color.White,
                    fontSize = 10.sp,
                    fontWeight = FontWeight.ExtraBold,
                    letterSpacing = 0.5.sp
                )
            }
        }
    )

    Text(
        text = annotatedText,
        inlineContent = inlineContent,
        style = MaterialTheme.typography.bodyLarge,
        modifier = Modifier.padding(16.dp)
    )
}
```

### PlaceholderVerticalAlign 옵션

`Placeholder`의 `placeholderVerticalAlign`은 인라인 콘텐츠의 수직 정렬을 결정합니다:

- `Center`: Placeholder의 중앙을 행(line)의 중앙에 맞춤
- `TextCenter`: 텍스트 중앙에 맞춤 (어센더/디센더 제외)
- `TextTop` / `TextBottom`: 텍스트 상단/하단에 맞춤
- `AboveBaseline`: 베이스라인 위로 정렬 (아이콘에 권장)

---

## 5. 주의사항 및 실전 팁

### AnnotatedString 성능 최적화

`buildAnnotatedString` 블록은 호출될 때마다 새 객체를 생성합니다. 리컴포지션이 자주 발생하는 컴포저블에서는 `remember`로 캐싱해야 합니다:

```kotlin
// ❌ 매 리컴포지션마다 새 AnnotatedString 생성
@Composable
fun BadExample(username: String) {
    Text(text = buildAnnotatedString {
        withStyle(SpanStyle(color = Color.Blue)) { append("@$username") }
        append(" 님이 메시지를 보냈습니다.")
    })
}

// ✅ username이 변경될 때만 재생성
@Composable
fun GoodExample(username: String) {
    val annotated = remember(username) {
        buildAnnotatedString {
            withStyle(SpanStyle(color = Color(0xFF1565C0), fontWeight = FontWeight.Medium)) {
                append("@$username")
            }
            append(" 님이 메시지를 보냈습니다.")
        }
    }
    Text(text = annotated)
}
```

### InlineContent의 접근성

`appendInlineContent`의 두 번째 파라미터 `alternateText`는 TalkBack 같은 스크린 리더가 읽어주는 대체 텍스트입니다. 아이콘의 의미를 설명하는 텍스트를 반드시 제공하세요:

```kotlin
appendInlineContent("star_icon", "[별점 5점]")  // ✅ 의미 있는 대체 텍스트
appendInlineContent("star_icon", "[★]")          // △ 스크린 리더가 "별" 이라고 읽음
appendInlineContent("star_icon", "")             // ❌ 접근성 무시
```

### Compose 1.8의 autoSize 파라미터

Compose 1.8부터 `Text`에 `autoSize` 파라미터가 추가되어, 주어진 공간에 맞게 폰트 크기를 자동으로 조절합니다:

```kotlin
Text(
    text = "이 텍스트는 컨테이너 크기에 맞게 자동으로 작아집니다.",
    maxLines = 1,
    // Compose 1.8+
    autoSize = TextAutoSize.StepBased(
        minFontSize = 12.sp,
        maxFontSize = 32.sp,
        stepSize = 1.sp
    ),
    modifier = Modifier.fillMaxWidth()
)
```

이를 활용하면 배너, 카드 헤더 등 고정 크기 영역에서 텍스트 오버플로를 없앨 수 있습니다.

---

## 정리: API별 사용 가이드

| API | 주요 용도 | 핵심 메서드 |
|-----|----------|------------|
| `AnnotatedString` | 복수 스타일, 클릭 처리 | `buildAnnotatedString`, `withStyle` |
| `SpanStyle` | 인라인 텍스트 스타일 | `color`, `fontSize`, `fontWeight`, `brush` |
| `ParagraphStyle` | 단락 정렬·들여쓰기 | `textAlign`, `lineHeight`, `textIndent` |
| `Brush` in TextStyle | 전체 그라디언트 | `Brush.linearGradient()`, `radialGradient()` |
| `Brush` in SpanStyle | 부분 그라디언트 | `brush`, `alpha` 조합 |
| `TextMotion.Animated` | 텍스트 애니메이션 품질 | `TextStyle(textMotion = ...)` |
| `InlineTextContent` | 텍스트 내 위젯 삽입 | `appendInlineContent`, `Placeholder` |

Jetpack Compose의 텍스트 API는 단순함 뒤에 풍부한 표현력을 숨기고 있습니다. AnnotatedString의 타입 안전한 DSL, Brush의 선언적 그라디언트, TextMotion의 렌더링 힌트 제어, InlineContent의 컴포저블 임베딩까지—이 네 가지를 조합하면 디자이너의 요구사항을 순수 Compose 코드로 구현할 수 있습니다. 커스텀 Canvas 렌더링이나 `SpannableString` 없이도, 훨씬 유지보수하기 쉬운 선언적 텍스트 렌더링 코드를 작성해 보세요.

## 참고 자료

- [Style text — Jetpack Compose](https://developer.android.com/develop/ui/compose/text/style-text)
- [Display and format text — Jetpack Compose](https://developer.android.com/develop/ui/compose/text/display-text)
- [TextMotion API Reference](https://developer.android.com/reference/kotlin/androidx/compose/ui/text/style/TextMotion)
- [Androidify: Building a delightful UI with Compose — Android Developers Blog](https://android-developers.googleblog.com/2025/05/androidify-building-delightful-ui-with-compose.html)
