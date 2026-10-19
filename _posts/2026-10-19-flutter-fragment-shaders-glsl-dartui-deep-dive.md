---
layout: post
title: "Flutter Fragment Shaders 심화: dart:ui FragmentShader API로 GLSL 커스텀 셰이더 구현하기"
date: 2026-10-19
categories: [android, flutter]
tags: [flutter, fragment-shader, glsl, dart-ui, custom-painter, animation, gpu, graphics]
---

Flutter 앱에서 놀라운 시각 효과를 구현하고 싶다면 Fragment Shader가 정답입니다. Canvas API로는 표현하기 어려운 파형 왜곡, 색상 그라디언트 혼합, 글리치 이펙트 등을 GPU에서 직접 처리하면 CPU 부담 없이 60fps 이상의 부드러운 애니메이션을 달성할 수 있습니다. 이 글에서는 Flutter의 `dart:ui`가 제공하는 `FragmentProgram` / `FragmentShader` API와 GLSL 셰이더 파일 작성 방법을 단계별로 알아봅니다.

---

## 1. Fragment Shader란 무엇인가

현대 GPU 렌더링 파이프라인은 크게 **Vertex Shader**(정점 처리)와 **Fragment Shader**(픽셀 처리) 두 단계로 구성됩니다. Fragment Shader는 화면에 출력되는 모든 픽셀의 최종 색상을 결정하는 프로그램으로, **GLSL(OpenGL Shading Language)** 이라는 C 계열 언어로 작성합니다.

```
CPU(Dart/Flutter) → GPU 렌더링 파이프라인
                       ↓
               Vertex Shader (도형 정점 처리)
                       ↓
               Rasterization (픽셀화)
                       ↓
            ★ Fragment Shader (픽셀별 색상 계산)
                       ↓
              최종 화면 출력
```

Fragment Shader의 핵심은 **픽셀 단위의 병렬 계산**입니다. GPU는 수천 개의 코어로 모든 픽셀을 동시에 계산하기 때문에, CPU에서 픽셀 하나씩 처리하는 방식보다 수백 배 빠릅니다.

Flutter는 Flutter 3.0부터 `dart:ui`의 `FragmentProgram` API를 통해 커스텀 GLSL 셰이더를 공식 지원합니다. Skia와 Impeller 백엔드 모두 지원하며, Android / iOS / macOS / Linux / Windows / Web 플랫폼에서 동작합니다.

---

## 2. 왜 Fragment Shader가 필요한가

### Canvas API의 한계

Flutter의 `CustomPainter`와 `Canvas` API는 강력하지만 CPU 기반 드로잉이기 때문에 아래와 같은 작업에서 성능 병목이 발생합니다.

| 효과 | Canvas API | Fragment Shader |
|------|-----------|----------------|
| 복잡한 그라디언트 | 가능하지만 느림 | GPU 병렬 처리 |
| 픽셀별 왜곡 효과 | 구현 어려움 | 자연스럽게 구현 |
| 실시간 파형 애니메이션 | CPU 과부하 | 60fps 유지 |
| 이미지 필터 / 색보정 | Dart 루프 필요 | 셰이더 1줄로 해결 |

Fragment Shader를 사용하면 복잡한 수학 계산(삼각함수, 노이즈 함수, 좌표 변환)을 GPU에 위임하여 메인 스레드 부담 없이 화려한 효과를 구현할 수 있습니다.

### 적합한 사용 사례

- **홈 화면 위젯·배너**의 물결(wave) 배경
- 게임의 **화염·물·안개** 파티클 효과
- 사진 편집 앱의 **색상 필터·글리치** 이펙트
- **전환 애니메이션**의 픽셀 왜곡
- 실시간 **데이터 시각화** (파형, 히트맵)

---

## 3. 실제 구현 예제

### 준비: pubspec.yaml 설정

GLSL 셰이더 파일(`.frag`)을 `shaders/` 디렉터리에 넣고, `pubspec.yaml`에 등록합니다.

```yaml
flutter:
  uses-material-design: true
  shaders:
    - shaders/gradient_wave.frag
    - shaders/color_shift.frag
```

Flutter 빌드 도구가 `.frag` 파일을 각 백엔드에 맞는 바이너리 형식으로 자동 컴파일합니다. 핫 리로드 시에도 셰이더가 자동 재컴파일됩니다.

---

### 예제 1: 커스텀 그라디언트 색상 셰이더

**shaders/color_shift.frag** — UV 좌표를 기반으로 배경 그라디언트를 생성하는 기본 셰이더입니다.

```glsl
#include <flutter/runtime_effect.glsl>

// Dart에서 넘겨주는 uniform 값
uniform float uTime;         // 경과 시간 (초)
uniform vec2  uResolution;   // 캔버스 크기 (width, height)

out vec4 fragColor;

void main() {
  // 현재 픽셀의 정규화된 UV 좌표 (0.0 ~ 1.0)
  vec2 uv = FlutterFragCoord().xy / uResolution;

  // 시간에 따라 색상이 천천히 이동하는 그라디언트
  float r = 0.5 + 0.5 * sin(uTime + uv.x * 3.14);
  float g = 0.5 + 0.5 * sin(uTime + uv.y * 3.14 + 2.094);
  float b = 0.5 + 0.5 * sin(uTime + (uv.x + uv.y) * 3.14 + 4.189);

  fragColor = vec4(r, g, b, 1.0);
}
```

**lib/color_shift_painter.dart** — `FragmentProgram`을 로드하고 `CustomPainter`에 적용합니다.

```dart
import 'dart:ui' as ui;
import 'package:flutter/material.dart';

class ColorShiftWidget extends StatefulWidget {
  const ColorShiftWidget({super.key});

  @override
  State<ColorShiftWidget> createState() => _ColorShiftWidgetState();
}

class _ColorShiftWidgetState extends State<ColorShiftWidget>
    with SingleTickerProviderStateMixin {
  ui.FragmentProgram? _program;
  late final AnimationController _controller;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(seconds: 6),
    )..repeat();

    // 비동기로 셰이더 프로그램 로드
    _loadShader();
  }

  Future<void> _loadShader() async {
    final program = await ui.FragmentProgram.fromAsset(
      'shaders/color_shift.frag',
    );
    if (mounted) setState(() => _program = program);
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    if (_program == null) {
      return const Center(child: CircularProgressIndicator());
    }
    return AnimatedBuilder(
      animation: _controller,
      builder: (context, _) {
        return CustomPaint(
          painter: _ColorShiftPainter(
            program: _program!,
            time: _controller.value * 6.0, // 0.0 ~ 6.0초
          ),
          size: const Size.fromHeight(300),
        );
      },
    );
  }
}

class _ColorShiftPainter extends CustomPainter {
  _ColorShiftPainter({required this.program, required this.time});

  final ui.FragmentProgram program;
  final double time;

  @override
  void paint(Canvas canvas, Size size) {
    // FragmentShader 인스턴스 생성 후 uniform 값 설정
    final shader = program.fragmentShader()
      ..setFloat(0, time)               // uTime
      ..setFloat(1, size.width)         // uResolution.x
      ..setFloat(2, size.height);       // uResolution.y

    final paint = Paint()..shader = shader;
    canvas.drawRect(Offset.zero & size, paint);
  }

  @override
  bool shouldRepaint(_ColorShiftPainter old) =>
      old.time != time || old.program != program;
}
```

핵심 포인트:
- `FragmentProgram.fromAsset()`은 **비동기**이므로 앱 시작 전 미리 로드하거나 `FutureBuilder`로 처리합니다.
- `fragmentShader()`를 호출할 때마다 새 인스턴스가 생성됩니다. 매 프레임 `paint()` 안에서 생성해도 무방합니다.
- uniform 인덱스는 GLSL 파일에 선언된 순서와 일치해야 합니다.

---

### 예제 2: 물결(Wave) 효과 + 이미지 샘플러 애니메이션

이번 예제는 자식 위젯을 텍스처로 캡처하여 물결 왜곡 효과를 적용합니다. `flutter_shaders` 패키지의 `AnimatedSampler`를 활용합니다.

**shaders/gradient_wave.frag** — 입력 이미지를 sin 함수로 왜곡합니다.

```glsl
#include <flutter/runtime_effect.glsl>

uniform float uTime;
uniform vec2  uResolution;
uniform sampler2D uTexture;   // 자식 위젯 텍스처

out vec4 fragColor;

void main() {
  vec2 uv = FlutterFragCoord().xy / uResolution;

  // X축 방향으로 sin 파형 왜곡
  float waveStrength = 0.02;
  float waveSpeed    = 3.0;
  float waveFreq     = 10.0;

  float offsetX = sin(uv.y * waveFreq + uTime * waveSpeed) * waveStrength;
  float offsetY = cos(uv.x * waveFreq + uTime * waveSpeed) * waveStrength * 0.5;

  vec2 warpedUV = uv + vec2(offsetX, offsetY);

  // 왜곡된 좌표에서 텍스처 샘플링
  fragColor = texture(uTexture, warpedUV);
}
```

**pubspec.yaml** 의존성 추가:

```yaml
dependencies:
  flutter_shaders: ^0.1.5

flutter:
  shaders:
    - shaders/gradient_wave.frag
```

**lib/wave_effect_widget.dart** — `AnimatedSampler`로 자식을 캡처하고 셰이더 적용:

```dart
import 'dart:ui' as ui;
import 'package:flutter/material.dart';
import 'package:flutter_shaders/flutter_shaders.dart';

class WaveEffectWidget extends StatefulWidget {
  const WaveEffectWidget({super.key, required this.child});
  final Widget child;

  @override
  State<WaveEffectWidget> createState() => _WaveEffectWidgetState();
}

class _WaveEffectWidgetState extends State<WaveEffectWidget>
    with SingleTickerProviderStateMixin {
  late final AnimationController _controller;
  ui.FragmentProgram? _program;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(seconds: 10),
    )..repeat();

    ui.FragmentProgram.fromAsset('shaders/gradient_wave.frag')
        .then((p) => mounted ? setState(() => _program = p) : null);
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final program = _program;
    if (program == null) return widget.child;

    return AnimatedBuilder(
      animation: _controller,
      builder: (context, _) {
        return AnimatedSampler(
          // builder: 캡처된 이미지(image)와 캔버스 크기(size)를 받아 셰이더를 적용
          (ui.Image image, Size size, Canvas canvas) {
            final shader = program.fragmentShader()
              ..setFloat(0, _controller.value * 10.0)  // uTime
              ..setFloat(1, size.width)                 // uResolution.x
              ..setFloat(2, size.height)                // uResolution.y
              ..setImageSampler(0, image);              // uTexture

            canvas.drawRect(
              Offset.zero & size,
              Paint()..shader = shader,
            );
          },
          child: widget.child,
        );
      },
    );
  }
}

// 사용 예시
class MyHomePage extends StatelessWidget {
  const MyHomePage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(
        child: WaveEffectWidget(
          child: Container(
            width: 300,
            height: 200,
            color: Colors.blue,
            child: const Center(
              child: Text(
                '물결 효과',
                style: TextStyle(
                  color: Colors.white,
                  fontSize: 32,
                  fontWeight: FontWeight.bold,
                ),
              ),
            ),
          ),
        ),
      ),
    );
  }
}
```

`AnimatedSampler`의 `builder` 콜백은 매 프레임마다 호출되며, `ui.Image` 형태의 자식 스냅샷을 셰이더에 넘겨줍니다. `setImageSampler(index, image)`는 GLSL의 `sampler2D` uniform에 해당합니다.

---

## 4. 주의사항 및 고급 팁

### GLSL 버전과 Flutter 전용 확장

Flutter는 GLSL 460부터 100까지 다양한 버전을 지원하지만, 반드시 `#include <flutter/runtime_effect.glsl>`을 선언하고 `FlutterFragCoord()`를 사용해야 합니다. `gl_FragCoord`를 직접 사용하면 Impeller 백엔드에서 Y축이 반전되는 문제가 발생합니다.

```glsl
// ❌ 잘못된 방법 (Impeller에서 Y축 반전 발생)
vec2 uv = gl_FragCoord.xy / uResolution;

// ✅ 올바른 방법
#include <flutter/runtime_effect.glsl>
vec2 uv = FlutterFragCoord().xy / uResolution;
```

### 지원되지 않는 GLSL 기능

Flutter Fragment Shader는 다음 기능을 **지원하지 않습니다**:

| 제한 사항 | 대안 |
|-----------|------|
| UBO(Uniform Buffer Object) | 개별 `uniform float/vec` 사용 |
| SSBO(Shader Storage Buffer) | 미지원 |
| `sampler2D` 이외 샘플러 타입 | `sampler2D`만 사용 |
| `unsigned int`, `bool` 타입 | `float`으로 대체 |
| 다중 텍스처 함수 오버로드 | `texture(sampler, uv)` 형식만 허용 |
| `#version` 지시자 없이 geometry/compute shader | Fragment Shader만 지원 |

### 성능 최적화 팁

**1. FragmentProgram은 한 번만 로드하세요**
`FragmentProgram.fromAsset()`을 매 빌드마다 호출하면 메모리와 시간이 낭비됩니다. `initState()`에서 한 번 로드하거나, 전역 캐시에 저장해두는 것이 좋습니다.

```dart
// 앱 시작 시 미리 로드하는 패턴
class ShaderCache {
  static final Map<String, ui.FragmentProgram> _cache = {};

  static Future<ui.FragmentProgram> load(String assetPath) async {
    return _cache.putIfAbsent(
      assetPath,
      () => ui.FragmentProgram.fromAsset(assetPath) as ui.FragmentProgram,
    );
    // 실제로는 putIfAbsent가 Future를 반환하지 않으므로 아래처럼 구현
  }
}
```

**2. uniform 값은 최소화하세요**
uniform 설정은 CPU→GPU 데이터 전송이므로, 변경이 없는 값은 셰이더 상수(`const`)로 처리합니다.

**3. 불필요한 분기 제거**
GLSL의 `if-else`는 GPU에서 분기 발산(branch divergence)을 유발합니다. 가능하면 `mix()`, `step()`, `smoothstep()` 같은 수학 함수로 대체하세요.

```glsl
// ❌ GPU 분기 발산
if (uv.x > 0.5) {
  fragColor = vec4(1.0, 0.0, 0.0, 1.0);
} else {
  fragColor = vec4(0.0, 0.0, 1.0, 1.0);
}

// ✅ 수학 함수로 분기 없이 처리
float t = step(0.5, uv.x);
fragColor = mix(vec4(0.0, 0.0, 1.0, 1.0), vec4(1.0, 0.0, 0.0, 1.0), t);
```

### Impeller 백엔드 호환성

Flutter 3.10부터 iOS에서 Impeller가 기본값이며, Android도 점진적으로 전환 중입니다. Flutter Fragment Shader는 Skia와 Impeller 모두 지원하지만, 셰이더 파일이 각 백엔드에서 사전 컴파일(AOT)되어야 하므로 `pubspec.yaml` 등록이 필수입니다.

런타임에 동적으로 생성한 GLSL 문자열은 지원하지 않습니다. 셰이더 파일은 반드시 앱 빌드 시점에 `shaders:` 섹션에 선언되어 있어야 합니다.

### 디버깅 팁

셰이더 오류는 런타임 예외로 나타납니다. 개발 중에는 다음 방법을 활용하세요:

- **핫 리로드**: 디버그 빌드에서 `.frag` 파일 수정 후 저장하면 자동 재컴파일됩니다.
- **오류 메시지**: GLSL 컴파일 오류는 Flutter 콘솔에 출력됩니다.
- **단순화 테스트**: 복잡한 셰이더에서 오류 시, uniform을 단순 상수로 교체하며 원인을 좁혀가세요.
- **`fragColor`를 빨간색으로**: 특정 조건에서 `fragColor = vec4(1.0, 0.0, 0.0, 1.0);`으로 설정해 디버그 포인트로 활용합니다.

---

## 정리

Flutter Fragment Shader는 **GLSL로 GPU에서 직접 픽셀을 제어**하는 강력한 도구입니다.

| 항목 | 내용 |
|------|------|
| 파일 형식 | `.frag` (GLSL) |
| 등록 위치 | `pubspec.yaml`의 `flutter: shaders:` |
| 로드 API | `ui.FragmentProgram.fromAsset()` |
| 셰이더 적용 | `Paint()..shader = fragmentShader` |
| 이미지 적용 | `AnimatedSampler` (flutter_shaders 패키지) |
| 백엔드 지원 | Skia, Impeller 모두 지원 |

Fragment Shader를 활용하면 Canvas API 만으로 구현하기 어려운 파형 왜곡, 실시간 색상 변환, 이미지 필터 효과를 GPU 성능으로 부드럽게 구현할 수 있습니다. `AnimatedSampler`와 결합하면 기존 Flutter 위젯 트리에 셰이더 효과를 간편하게 오버레이할 수 있어 활용 범위가 매우 넓습니다.

---

## 참고 자료

- [flutter_shaders 패키지 (pub.dev)](https://pub.dev/packages/flutter_shaders)
- [AnimatedSampler API 문서](https://pub.dev/documentation/flutter_shaders/latest/flutter_shaders/AnimatedSampler-class.html)
- [flutter_shaders GitHub 저장소](https://github.com/goderbauer/flutter_shaders)
