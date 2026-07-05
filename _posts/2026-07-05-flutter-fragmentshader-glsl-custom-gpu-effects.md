---
layout: post
title: "Flutter FragmentShader 심화: GLSL 커스텀 셰이더로 픽셀 단위 GPU 이펙트 구현하기"
date: 2026-07-05
categories: [android, flutter]
tags: [flutter, shader, glsl, fragmentshader, gpu, animation, custompainter, dart]
---

Flutter는 화려한 UI를 쉽게 만들 수 있는 강력한 렌더링 엔진을 내장하고 있습니다. 그러나 블러, 그라디언트, 물결 왜곡처럼 픽셀 단위의 그래픽 이펙트가 필요할 때 Flutter의 위젯 레이어만으로는 GPU 병렬 처리를 제대로 활용하기 어렵습니다. Flutter 3.7에서 공식 안정화된 **FragmentProgram API**는 GLSL(OpenGL Shading Language)로 작성한 셰이더를 직접 GPU에서 실행할 수 있는 길을 열어 주었습니다. 이 글에서는 GLSL 셰이더의 기본 원리부터 실제 프로덕션에 적용할 수 있는 두 가지 이펙트 구현까지 단계별로 깊이 있게 다룹니다.

## FragmentShader란 무엇인가

**셰이더(Shader)** 는 GPU 위에서 병렬로 실행되는 작은 프로그램입니다. 화면을 구성하는 수백만 개의 픽셀 각각에 대해, GPU는 셰이더 코드를 동시에 실행하여 최종 색상을 결정합니다. 프래그먼트 셰이더(Fragment Shader)는 래스터화(rasterization) 단계에서 각 픽셀의 색상을 계산하는 셰이더를 의미합니다.

Flutter의 렌더링 파이프라인은 내부적으로 Skia(혹은 Impeller) 위에서 동작하며, Flutter 3.7 이전에도 셰이더를 사용했지만 개발자가 직접 작성할 수 있는 공개 API는 실험적 수준이었습니다. **Flutter 3.7**부터는 `FragmentProgram` 클래스와 `dart:ui`의 `FragmentShader`가 공식 안정 API로 승격되어 프로덕션 코드에 사용할 수 있게 되었습니다.

### 핵심 클래스 구조

| 클래스 | 역할 |
|---|---|
| `FragmentProgram` | GLSL 소스를 컴파일한 GPU 프로그램. `fromAsset()`으로 로드 |
| `FragmentShader` | `FragmentProgram`의 인스턴스. uniform 값을 설정하고 `Paint.shader`에 할당 |
| `Paint.shader` | `Canvas.drawRect()` 등에 셰이더를 적용하는 속성 |

## 왜 FragmentShader가 필요한가

Flutter가 이미 `ImageFilter.blur()`, `ColorFilter`, `BackdropFilter` 등 다양한 효과를 제공하는데 굳이 셰이더를 직접 작성해야 할 이유가 있을까요? 다음과 같은 상황에서 FragmentShader는 탁월한 선택입니다.

**1. 기본 위젯 API로 표현 불가능한 이펙트**: 물결 왜곡, 노이즈 기반 디졸브 트랜지션, 픽셀화(pixelation), 크로마틱 어베레이션(chromatic aberration) 등은 기존 Flutter API로 구현이 불가능하거나 극히 어렵습니다.

**2. 고성능 애니메이션**: CPU 기반 `CustomPainter`로 매 프레임마다 복잡한 수학 연산을 수행하면 UI 스레드가 블로킹됩니다. 셰이더는 GPU의 수천 개 코어에서 병렬 실행되므로 복잡한 픽셀 연산도 60fps 이상을 유지합니다.

**3. 위젯 트리를 텍스처로 캡처해 이펙트 적용**: `flutter_shaders` 패키지의 `AnimatedSampler`를 사용하면 임의의 Flutter 위젯 서브트리를 실시간 텍스처로 캡처하여 셰이더에 공급할 수 있습니다. 이를 통해 "특정 위젯에 실시간 유리 왜곡 효과 적용" 같은 작업이 가능해집니다.

**4. ShaderToy 자산 재활용**: ShaderToy.com의 수만 개 커뮤니티 GLSL 셰이더를 Flutter로 이식하는 작업이 비교적 간단해집니다.

## 프로젝트 설정

Flutter의 셰이더는 `.frag` 확장자를 가진 GLSL 파일로 작성하며, `pubspec.yaml`의 `flutter.shaders` 섹션에 등록해야 합니다. Flutter 빌드 도구가 이를 자동으로 컴파일하여 앱 번들에 포함시킵니다.

```yaml
# pubspec.yaml
flutter:
  shaders:
    - shaders/wave_distortion.frag
    - shaders/grayscale.frag
```

셰이더 파일은 반드시 `#include <flutter/runtime_effect.glsl>` 헤더를 포함해야 합니다. 이 헤더는 Flutter가 제공하는 `FlutterFragCoord()` 함수를 사용할 수 있게 해 줍니다. OpenGL의 `gl_FragCoord`를 직접 쓰면 플랫폼마다 좌표계가 달라져 버그가 발생하므로, 반드시 `FlutterFragCoord()`를 사용해야 합니다.

## 예제 1: 물결 왜곡(Wave Distortion) 이펙트

가장 먼저 시간 기반 물결 왜곡 효과를 구현해 봅니다. 이미지나 위젯 위에 마치 물결치는 것처럼 UV 좌표를 사인 함수로 변형합니다.

### GLSL 셰이더 파일 (`shaders/wave_distortion.frag`)

```glsl
#include <flutter/runtime_effect.glsl>

// uniform: Dart 코드에서 매 프레임 전달하는 값
uniform float uTime;        // 경과 시간 (초)
uniform vec2  uResolution;  // 화면 해상도 (width, height)
uniform sampler2D uTexture; // 원본 이미지 텍스처

out vec4 fragColor;

void main() {
  // FlutterFragCoord()로 현재 픽셀 좌표를 얻음 (0,0) ~ (width, height)
  vec2 fragCoord = FlutterFragCoord().xy;

  // UV 좌표 정규화: 0.0 ~ 1.0
  vec2 uv = fragCoord / uResolution;

  // 물결 왜곡: x축과 y축에 각각 사인파 적용
  float amplitude = 0.015;
  float frequency = 10.0;
  float speed     = 2.0;

  uv.x += sin(uv.y * frequency + uTime * speed) * amplitude;
  uv.y += sin(uv.x * frequency + uTime * speed) * amplitude;

  // 왜곡된 UV로 텍스처 샘플링
  fragColor = texture(uTexture, uv);
}
```

### Dart 코드: `WaveWidget`

```dart
import 'dart:ui' as ui;
import 'package:flutter/material.dart';
import 'package:flutter/scheduler.dart';

class WaveWidget extends StatefulWidget {
  final ImageProvider imageProvider;
  const WaveWidget({super.key, required this.imageProvider});

  @override
  State<WaveWidget> createState() => _WaveWidgetState();
}

class _WaveWidgetState extends State<WaveWidget>
    with SingleTickerProviderStateMixin {
  late final Ticker _ticker;
  FragmentShader? _shader;
  ui.Image?       _image;
  double          _elapsedSeconds = 0.0;

  @override
  void initState() {
    super.initState();
    _loadResources();
    _ticker = createTicker(_onTick)..start();
  }

  Future<void> _loadResources() async {
    // FragmentProgram을 pubspec에 등록한 asset 경로로 로드
    final program = await FragmentProgram.fromAsset(
      'shaders/wave_distortion.frag',
    );
    final imageStream = widget.imageProvider.resolve(ImageConfiguration.empty);
    imageStream.addListener(ImageStreamListener((info, _) {
      if (mounted) setState(() {
        _shader = program.fragmentShader();
        _image  = info.image;
      });
    }));
  }

  void _onTick(Duration elapsed) {
    if (_shader != null && mounted) {
      setState(() {
        _elapsedSeconds = elapsed.inMicroseconds / 1e6;
      });
    }
  }

  @override
  void dispose() {
    _ticker.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    if (_shader == null || _image == null) {
      return const Center(child: CircularProgressIndicator());
    }
    return LayoutBuilder(builder: (context, constraints) {
      final size = Size(constraints.maxWidth, constraints.maxHeight);

      // 매 프레임마다 uniform 값 업데이트
      _shader!
        ..setFloat(0, _elapsedSeconds)          // uTime
        ..setFloat(1, size.width)               // uResolution.x
        ..setFloat(2, size.height)              // uResolution.y
        ..setImageSampler(0, _image!);          // uTexture

      return CustomPaint(
        size: size,
        painter: _ShaderPainter(_shader!),
      );
    });
  }
}

class _ShaderPainter extends CustomPainter {
  final FragmentShader shader;
  _ShaderPainter(this.shader);

  @override
  void paint(Canvas canvas, Size size) {
    final paint = Paint()..shader = shader;
    canvas.drawRect(Offset.zero & size, paint);
  }

  @override
  bool shouldRepaint(_ShaderPainter old) => true;
}
```

`setFloat`의 인덱스 순서는 GLSL 파일에서 `uniform`이 선언된 순서와 정확히 일치해야 합니다. `uTime`은 `float`(인덱스 0), `uResolution`은 `vec2`(인덱스 1, 2로 두 개의 float을 차지), `uTexture`는 `sampler2D`로 `setImageSampler(0, ...)`에 해당합니다.

## 예제 2: 위젯 그레이스케일 이펙트 — AnimatedSampler 활용

두 번째 예제는 임의의 Flutter 위젯 서브트리 전체를 실시간으로 그레이스케일로 변환하는 이펙트입니다. `flutter_shaders` 패키지의 `AnimatedSampler`를 활용하면 자식 위젯을 텍스처로 캡처하여 셰이더에 공급할 수 있습니다.

```yaml
# pubspec.yaml dependencies
dependencies:
  flutter_shaders: ^0.1.3
```

### GLSL 셰이더 (`shaders/grayscale.frag`)

```glsl
#include <flutter/runtime_effect.glsl>

uniform vec2      uSize;       // 위젯 크기
uniform float     uStrength;   // 0.0 (원본) ~ 1.0 (완전 그레이스케일)
uniform sampler2D uTexture;    // 캡처된 자식 위젯 텍스처

out vec4 fragColor;

void main() {
  vec2 uv = FlutterFragCoord().xy / uSize;
  vec4 color = texture(uTexture, uv);

  // NTSC 표준 가중치를 이용한 밝기 계산
  float luminance = dot(color.rgb, vec3(0.299, 0.587, 0.114));

  // uStrength에 따라 원본과 그레이스케일을 선형 보간
  vec3 gray   = vec3(luminance);
  vec3 result = mix(color.rgb, gray, uStrength);

  fragColor = vec4(result, color.a);
}
```

### Dart 코드: `GrayscaleWidget`

```dart
import 'dart:ui' as ui;
import 'package:flutter/material.dart';
import 'package:flutter_shaders/flutter_shaders.dart';

class GrayscaleWidget extends StatefulWidget {
  final Widget child;
  /// 0.0 (원본) ~ 1.0 (완전 그레이스케일)
  final double strength;

  const GrayscaleWidget({
    super.key,
    required this.child,
    this.strength = 1.0,
  });

  @override
  State<GrayscaleWidget> createState() => _GrayscaleWidgetState();
}

class _GrayscaleWidgetState extends State<GrayscaleWidget> {
  late final Future<FragmentProgram> _programFuture;

  @override
  void initState() {
    super.initState();
    _programFuture = FragmentProgram.fromAsset('shaders/grayscale.frag');
  }

  @override
  Widget build(BuildContext context) {
    return FutureBuilder<FragmentProgram>(
      future: _programFuture,
      builder: (context, snapshot) {
        if (!snapshot.hasData) return widget.child;

        final program = snapshot.data!;

        // AnimatedSampler: 자식 위젯을 ui.Image로 캡처하여 builder에 전달
        return AnimatedSampler(
          (ui.Image texture, Size size, Canvas canvas) {
            final shader = program.fragmentShader()
              ..setFloat(0, size.width)       // uSize.x
              ..setFloat(1, size.height)      // uSize.y
              ..setFloat(2, widget.strength)  // uStrength
              ..setImageSampler(0, texture);  // uTexture

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
```

사용 예시는 매우 단순합니다. 기존 위젯 트리를 `GrayscaleWidget`으로 감싸기만 하면 됩니다.

```dart
// 예: 특정 카드 전체를 조건부로 그레이스케일 처리
GrayscaleWidget(
  strength: isDisabled ? 1.0 : 0.0,
  child: ProductCard(product: product),
)
```

`strength` 값을 `AnimationController`와 연결하면 활성/비활성 전환 시 부드러운 그레이스케일 페이드 애니메이션도 구현할 수 있습니다.

## 주의사항 및 팁

### 1. uniform 인덱스 순서를 엄격히 지킬 것

GLSL에서 `vec2`는 두 개의 float을 차지하고, `vec4`는 네 개를 차지합니다. Dart 쪽에서 `setFloat`를 호출할 때 인덱스를 계산하는 실수가 흔히 발생합니다. GLSL 선언 순서대로 차지하는 float 개수를 직접 세어 인덱스를 맞춰야 합니다.

### 2. Impeller 백엔드 대응

Flutter의 차세대 렌더링 엔진 **Impeller**는 iOS에서 기본 활성화되어 있고 Android에서도 점진적으로 활성화되고 있습니다. Impeller는 GLSL ES 3.2를 사용하므로, 기존 Skia 백엔드와 미묘한 GLSL 문법 차이가 있을 수 있습니다. 특히 `gl_FragCoord` 대신 반드시 `FlutterFragCoord()`를 사용해야 Impeller에서 정상 동작합니다.

### 3. 플랫폼별 빌드 테스트 필수

셰이더는 컴파일 방식이 플랫폼마다 다릅니다. Android는 GLSL → SPIR-V, iOS/macOS는 GLSL → MSL(Metal Shading Language)로 변환됩니다. `flutter run`으로 각 플랫폼에서 직접 확인하는 것이 필수입니다. 특히 복잡한 수학 함수(삼각함수, 지수함수) 사용 시 플랫폼별로 정밀도 차이가 날 수 있습니다.

### 4. FragmentProgram은 전역으로 캐시

`FragmentProgram.fromAsset()`은 비동기 로딩 비용이 있습니다. 위젯 생성마다 호출하지 말고, 앱 시작 시 한 번 로드하여 싱글톤 또는 `InheritedWidget`으로 공급하는 패턴이 권장됩니다.

### 5. 성능 프로파일링

`flutter_devtools`의 **GPU 프레임 타임** 뷰를 활용하세요. 셰이더 자체는 빠르지만, 매 프레임마다 `setFloat`와 `setImageSampler`를 반복 호출하는 오버헤드가 누적될 수 있습니다. `AnimatedSampler`를 사용할 때는 `enabled: false`로 셰이더를 비활성화하면 자식 위젯이 일반적인 방식으로 렌더링되어 성능을 절약할 수 있습니다.

### 6. 웹 플랫폼 주의

Flutter Web에서 셰이더 지원은 CanvasKit 렌더러에서만 동작하며, HTML 렌더러에서는 무시됩니다. 웹 타깃이 있다면 반드시 CanvasKit으로 빌드해야 합니다(`--web-renderer canvaskit`).

## 마무리

Flutter의 FragmentShader API는 위젯 레이어에서는 불가능했던 픽셀 단위의 GPU 병렬 처리를 Dart 코드와 자연스럽게 연결해 줍니다. 물결, 그레이스케일에서 시작해 ShaderToy의 광대한 셰이더 라이브러리를 Flutter 앱에 이식하는 데까지 응용 범위가 넓습니다. `flutter_shaders` 패키지의 `AnimatedSampler`를 활용하면 임의의 위젯 서브트리에 실시간 이펙트를 적용하는 것도 어렵지 않습니다. 셰이더 코드와 Dart 코드 사이의 uniform 인덱스 동기화, 플랫폼별 테스트, `FragmentProgram` 캐싱만 지키면 프로덕션 환경에서도 안정적으로 활용할 수 있습니다.

## 참고 자료

- [flutter_shaders 패키지 (pub.dev)](https://pub.dev/packages/flutter_shaders)
- [AnimatedSampler 클래스 API 문서](https://pub.dev/documentation/flutter_shaders/latest/flutter_shaders/AnimatedSampler-class.html)
- [shady 패키지 — ShaderToy 호환 셰이더 관리 유틸리티](https://pub.dev/packages/shady)
