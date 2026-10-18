---
layout: post
title: "Android AGSL & RuntimeShader 심화: GPU 셰이더로 고급 비주얼 이펙트 구현하기"
date: 2026-10-18
categories: [android, flutter]
tags: [android, agsl, runtimeshader, jetpack-compose, gpu, shader, visual-effects, graphics]
---

Android 13(API 33)부터 공식 지원된 **AGSL(Android Graphics Shading Language)** 과 `RuntimeShader`는 개발자가 GPU 레벨에서 커스텀 픽셀 셰이더를 작성할 수 있게 해줍니다. 기존 Canvas API로는 표현하기 어려웠던 물결 효과, 디스토션, 색상 왜곡 같은 고품질 비주얼 이펙트를 이제 Jetpack Compose와 함께 손쉽게 구현할 수 있습니다. 이 글에서는 AGSL의 개념부터 실전 Compose 통합까지 단계별로 살펴봅니다.

## 1. AGSL이란 무엇인가

AGSL은 구글이 Android 13에서 도입한 셰이딩 언어로, OpenGL ES Shading Language(GLSL)의 문법을 기반으로 하되 Android Canvas/RenderNode 파이프라인에 맞게 조정된 언어입니다. 핵심은 **픽셀 단위 함수(fragment function)** 를 작성하는 것으로, 화면 좌표 `fragCoord`를 입력받아 `half4` 형태의 색상을 반환합니다.

```
half4 main(float2 fragCoord) {
    return half4(1.0, 0.0, 0.0, 1.0); // 빨간색
}
```

이 함수는 GPU에서 병렬로 실행되므로, 복잡한 색상 계산도 CPU 부담 없이 처리됩니다. AGSL과 GLSL의 주요 차이점은 다음과 같습니다:

| 항목 | GLSL | AGSL |
|------|------|------|
| 좌표 원점 | 좌하단 | 좌상단 |
| 색상 공간 | 선형 | 플랫폼 색상 공간 |
| 텍스처 샘플링 | `texture()` | `shader.eval()` |
| 적용 대상 | OpenGL 컨텍스트 | Android Canvas/RenderNode |

`RuntimeShader`는 AGSL 소스 문자열을 받아 컴파일하고, Android의 `Paint`, `Brush`, `RenderEffect` 등 다양한 드로잉 API와 연결할 수 있는 클래스입니다.

## 2. 왜 필요한가

### 기존 Canvas API의 한계

기존 Android Canvas API는 `drawRect`, `drawCircle`, `drawPath` 같은 도형 드로잉과 `LinearGradient`, `RadialGradient` 등의 단순 그라디언트만 지원합니다. 다음과 같은 효과는 CPU에서 비트맵을 직접 조작해야 했고, 성능 문제가 심각했습니다:

- 픽셀 단위의 노이즈/펄린 노이즈 패턴
- 물결(wave) 또는 파동(ripple) 왜곡 애니메이션
- 크로마틱 어버레이션(색수차) 효과
- 복잡한 프로시저럴 패턴

### GPU 가속의 이점

`RuntimeShader`를 사용하면 같은 효과를 GPU에서 실행합니다. 예를 들어 1080×2400 해상도 화면에서 물결 효과를 CPU로 처리하면 약 259만 픽셀을 매 프레임 순차 계산해야 하지만, GPU에서는 수천 개의 코어가 병렬로 처리하므로 16ms(60FPS) 내에 안정적으로 완료됩니다.

또한 **RenderEffect**와 결합하면 특정 Composable 계층 전체에 셰이더 효과를 오버레이할 수 있어, 기존 UI 위에 포스트 프로세싱 효과를 입히는 것도 가능합니다.

## 3. 실제 구현 예제

### 예제 1: ShaderBrush로 Compose에서 그라디언트 애니메이션 구현

아래 예제는 시간에 따라 색상이 부드럽게 변하는 동적 그라디언트를 Jetpack Compose에서 구현합니다.

```kotlin
import android.graphics.RuntimeShader
import androidx.compose.animation.core.*
import androidx.compose.foundation.Canvas
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.ShaderBrush

private const val ANIMATED_GRADIENT_SHADER = """
    uniform float2 iResolution;
    uniform float iTime;

    half4 main(float2 fragCoord) {
        // 정규화된 UV 좌표 (0.0 ~ 1.0)
        float2 uv = fragCoord / iResolution;

        // 시간 기반 색상 사이클
        float r = 0.5 + 0.5 * sin(iTime + uv.x * 3.14159);
        float g = 0.5 + 0.5 * sin(iTime + uv.y * 3.14159 + 2.094);
        float b = 0.5 + 0.5 * sin(iTime + (uv.x + uv.y) * 3.14159 + 4.189);

        return half4(r, g, b, 1.0);
    }
"""

@Composable
fun AnimatedGradientCanvas() {
    // API 33+ 체크
    if (android.os.Build.VERSION.SDK_INT < android.os.Build.VERSION_CODES.TIRAMISU) {
        return
    }

    val shader = remember { RuntimeShader(ANIMATED_GRADIENT_SHADER) }
    val shaderBrush = remember { ShaderBrush(shader) }

    // 무한 반복 애니메이션 (0 → 6.28 = 2π, 즉 한 바퀴)
    val infiniteTransition = rememberInfiniteTransition(label = "gradient")
    val time by infiniteTransition.animateFloat(
        initialValue = 0f,
        targetValue = 6.2832f,
        animationSpec = infiniteRepeatable(
            animation = tween(durationMillis = 4000, easing = LinearEasing),
            repeatMode = RepeatMode.Restart
        ),
        label = "time"
    )

    Canvas(modifier = Modifier.fillMaxSize()) {
        shader.setFloatUniform("iResolution", size.width, size.height)
        shader.setFloatUniform("iTime", time)
        drawRect(brush = shaderBrush)
    }
}
```

**핵심 포인트:**
- `RuntimeShader`는 `remember`로 인스턴스를 유지합니다. 매 recomposition마다 생성하면 컴파일 비용이 반복 발생합니다.
- `ShaderBrush`는 `drawRect`, `drawCircle`, `drawPath` 등 Brush를 받는 모든 DrawScope 함수에서 사용 가능합니다.
- `setFloatUniform`으로 Kotlin 코드에서 셰이더 내부 `uniform` 변수에 값을 전달합니다.

### 예제 2: RenderEffect로 기존 Composable에 물결 왜곡 적용

다음 예제는 `RenderEffect.createRuntimeShaderEffect`를 사용하여 버튼이나 카드 같은 기존 Composable 위에 물결 왜곡을 실시간으로 입힙니다. 이 API는 자식 Composable의 렌더링 결과를 셰이더의 입력으로 사용할 수 있습니다.

```kotlin
import android.graphics.RenderEffect
import android.graphics.RuntimeShader
import androidx.compose.animation.core.*
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.asComposeRenderEffect
import androidx.compose.ui.graphics.graphicsLayer
import androidx.compose.ui.unit.dp

private const val WAVE_DISTORTION_SHADER = """
    uniform shader content;    // 자식 뷰의 렌더링 결과
    uniform float2 iResolution;
    uniform float iTime;
    uniform float iStrength;   // 왜곡 강도 (0.0 ~ 1.0)

    half4 main(float2 fragCoord) {
        float2 uv = fragCoord / iResolution;

        // 사인파 기반 물결 왜곡
        float waveX = sin(uv.y * 20.0 + iTime * 3.0) * iStrength * 0.02;
        float waveY = cos(uv.x * 15.0 + iTime * 2.5) * iStrength * 0.015;

        float2 distortedCoord = fragCoord + float2(
            waveX * iResolution.x,
            waveY * iResolution.y
        );

        // 경계 클램프 (화면 밖 샘플링 방지)
        distortedCoord = clamp(distortedCoord, float2(0.0), iResolution - 1.0);

        return content.eval(distortedCoord);
    }
"""

@Composable
fun WaveDistortionCard(isAnimating: Boolean, content: @Composable () -> Unit) {
    if (android.os.Build.VERSION.SDK_INT < android.os.Build.VERSION_CODES.TIRAMISU) {
        content()
        return
    }

    val shader = remember { RuntimeShader(WAVE_DISTORTION_SHADER) }

    val infiniteTransition = rememberInfiniteTransition(label = "wave")
    val time by infiniteTransition.animateFloat(
        initialValue = 0f,
        targetValue = 100f,
        animationSpec = infiniteRepeatable(
            animation = tween(100000, easing = LinearEasing),
            repeatMode = RepeatMode.Restart
        ),
        label = "time"
    )

    // 애니메이션 강도: isAnimating이면 1.0, 아니면 0.0으로 부드럽게 전환
    val strength by animateFloatAsState(
        targetValue = if (isAnimating) 1.0f else 0.0f,
        animationSpec = tween(durationMillis = 600),
        label = "strength"
    )

    Box(
        modifier = Modifier
            .fillMaxWidth()
            .wrapContentHeight()
            .graphicsLayer {
                val renderEffect = RenderEffect.createRuntimeShaderEffect(
                    shader, "content"
                )
                shader.setFloatUniform("iResolution", size.width, size.height)
                shader.setFloatUniform("iTime", time)
                shader.setFloatUniform("iStrength", strength)

                // strength가 0이면 RenderEffect 제거 (성능 최적화)
                this.renderEffect = if (strength > 0.01f) {
                    renderEffect.asComposeRenderEffect()
                } else null
            }
    ) {
        content()
    }
}

// 사용 예시
@Composable
fun DemoScreen() {
    var waveEnabled by remember { mutableStateOf(false) }

    Column(
        modifier = Modifier.fillMaxSize().padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        WaveDistortionCard(isAnimating = waveEnabled) {
            Card(
                modifier = Modifier.fillMaxWidth().height(200.dp)
            ) {
                Box(
                    modifier = Modifier.fillMaxSize(),
                    contentAlignment = Alignment.Center
                ) {
                    Text("Wave Effect Demo", style = MaterialTheme.typography.headlineMedium)
                }
            }
        }

        Spacer(modifier = Modifier.height(24.dp))

        Button(onClick = { waveEnabled = !waveEnabled }) {
            Text(if (waveEnabled) "효과 끄기" else "물결 효과 켜기")
        }
    }
}
```

**핵심 포인트:**
- `RenderEffect.createRuntimeShaderEffect(shader, "content")`의 두 번째 인자는 AGSL 코드 내 `uniform shader content` 변수 이름과 일치해야 합니다.
- `graphicsLayer` 내부에서 `size`는 해당 Composable의 픽셀 크기입니다. `setFloatUniform`은 매 프레임 호출되어 `iResolution`을 최신 상태로 유지합니다.
- `strength`가 0에 가까울 때 `renderEffect = null`을 할당하면 GPU 연산을 완전히 생략하여 배터리와 성능을 절약합니다.

## 4. 주의사항과 실전 팁

### API 레벨 요구 사항

`RuntimeShader`는 **API 33(Android 13)** 이상에서만 사용 가능합니다. `RenderEffect.createRuntimeShaderEffect`도 마찬가지입니다. 반드시 `Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU` 조건을 확인하고, 하위 버전에는 Fallback UI를 제공해야 합니다.

```kotlin
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
    // AGSL 효과 적용
} else {
    // 기본 UI 표시
}
```

### RuntimeShader 컴파일 비용

`RuntimeShader(shaderSrc)` 생성자는 내부적으로 셰이더를 컴파일합니다. 이 비용은 초기 1회 발생하지만, 메인 스레드에서 호출되므로 복잡한 셰이더의 경우 짧은 지연이 발생할 수 있습니다. `remember`로 Composable 생명주기 동안 단 한 번만 생성하고, 필요하다면 백그라운드 스레드에서 미리 워밍업하는 전략을 사용하세요.

### uniform 갱신과 스레드 안전성

`setFloatUniform`, `setColorUniform` 등은 **스레드 세이프하지 않습니다**. 반드시 UI 스레드(메인 스레드)에서 호출해야 합니다. `graphicsLayer` 블록과 `Canvas` 드로잉 블록은 Compose의 드로잉 단계에서 UI 스레드에서 실행되므로 안전합니다.

### 성능 프로파일링

복잡한 셰이더는 GPU 오버드로우(overdraw)를 유발할 수 있습니다. Android Studio의 **GPU Profiler** 또는 `adb shell dumpsys gfxinfo` 명령으로 GPU 프레임 시간을 모니터링하세요. 특히 `RenderEffect`를 중첩 적용하면 렌더링 패스가 늘어나 성능에 영향을 줍니다.

### 셰이더 내 분기문 최소화

GPU는 병렬로 동작하는 특성상 `if-else` 분기가 많으면 워프(warp) 단위 분기 발산(divergence)이 발생하여 성능이 저하됩니다. 가능하면 `mix()`, `step()`, `smoothstep()` 같은 수학 함수로 분기를 대체하세요.

```
// 나쁜 예: 분기
half4 color;
if (uv.x > 0.5) { color = half4(1,0,0,1); } else { color = half4(0,0,1,1); }

// 좋은 예: mix로 대체
half4 color = mix(half4(0,0,1,1), half4(1,0,0,1), step(0.5, uv.x));
```

### 색상 공간 주의

AGSL은 **sRGB 색상 공간**에서 작동합니다. `setColorUniform`으로 전달하는 색상도 sRGB로 해석됩니다. 선형 블렌딩이 필요하다면 셰이더 내에서 `pow(color, half4(2.2))`로 감마 인코딩을 수동으로 처리하거나, Android의 색상 관리 API를 활용하세요.

## 마무리

AGSL과 `RuntimeShader`는 Android 그래픽의 새로운 지평을 열었습니다. 기존에는 OpenGL ES나 Vulkan 같은 저수준 API를 직접 사용해야 했던 GPU 셰이더 효과를, 이제는 Kotlin 코드와 함께 Jetpack Compose에 자연스럽게 통합할 수 있습니다. API 33+ 제약이 있지만, 고급 UI를 추구하는 앱에서는 점점 필수적인 기술이 되어가고 있습니다. 공식 AGSL 레퍼런스와 Compose 그래픽 가이드를 함께 참고하면 더 다양한 효과를 구현할 수 있습니다.

## 참고 자료
- [AGSL 사용 가이드 (Android Developers)](https://developer.android.com/develop/ui/views/graphics/agsl/using-agsl)
- [RuntimeShader API 레퍼런스](https://developer.android.com/reference/android/graphics/RuntimeShader)
- [Graphics in Compose (Jetpack Compose)](https://developer.android.com/jetpack/compose/graphics/draw/overview)
