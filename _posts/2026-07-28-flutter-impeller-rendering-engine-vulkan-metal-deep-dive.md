---
layout: post
title: "Flutter Impeller 렌더링 엔진 심화: Vulkan·Metal·오프라인 셰이더 컴파일로 Shader Jank를 완전히 제거하는 법"
date: 2026-07-28
categories: [android, flutter]
tags: [flutter, impeller, rendering, vulkan, metal, shader, performance, skia]
---

Flutter가 Skia 기반 렌더링 파이프라인에서 벗어나 Impeller라는 완전히 새로운 렌더링 엔진으로 전환한 것은 단순한 업그레이드가 아닙니다. 이 글에서는 Impeller의 내부 아키텍처, Skia와의 근본적인 차이, 오프라인 셰이더 컴파일이 Jank를 제거하는 원리, 그리고 개발자가 실제로 활용할 수 있는 기법까지 깊이 있게 살펴봅니다.

## 왜 Skia로는 충분하지 않았는가

Flutter는 2018년 출시 이후 오랫동안 Google의 Skia 2D 그래픽 라이브러리를 렌더링 백엔드로 사용했습니다. Skia는 Chrome, Android, ChromeOS 등 수많은 플랫폼에서 검증된 안정적인 엔진이지만, Flutter 앱에서는 반복적으로 한 가지 심각한 문제가 보고되었습니다. 바로 **셰이더 컴파일 Jank(Shader Compilation Jank)**입니다.

### 셰이더 컴파일 Jank란?

GPU가 도형이나 이미지를 화면에 그리려면 **GLSL**(OpenGL Shading Language)이나 **HLSL** 같은 셰이더 프로그램이 필요합니다. 이 셰이더는 CPU가 아닌 GPU에서 실행되는 작은 프로그램으로, 각 픽셀의 색상과 변환을 계산합니다.

Skia의 경우 셰이더를 **런타임에 JIT(Just-In-Time) 컴파일**합니다. 즉, 앱 실행 중에 처음 특정 UI 요소가 등장할 때 비로소 GPU 드라이버가 해당 셰이더를 컴파일합니다. 이 컴파일 과정은 수 밀리초에서 수십 밀리초가 걸리며, 결과적으로 애니메이션 첫 실행 시 눈에 띄는 프레임 드롭이 발생합니다. Flutter 팀이 오랫동안 "처음 실행 시 버벅거림"으로 불렀던 그 현상입니다.

이 문제는 특히 다음 상황에서 두드러집니다:
- 복잡한 Hero 애니메이션이 첫 실행될 때
- 커스텀 Paint로 그려진 복잡한 경로가 처음 렌더링될 때
- Lottie 같은 벡터 기반 애니메이션이 시작될 때

Skia는 `SkSL`(Skia Shading Language)이라는 중간 셰이더 언어를 도입하고 셰이더를 미리 웜업하는 방식으로 이를 완화하려 했지만, Flutter 팀은 더 근본적인 해결책이 필요하다고 판단했습니다.

## Impeller의 핵심 설계 철학

Impeller는 처음부터 Flutter 전용으로 설계된 렌더링 런타임입니다. Flutter 엔진 소스코드(`flutter/engine` 저장소)에 독립적인 디렉토리로 존재하며, 5가지 핵심 목표를 중심으로 설계되었습니다:

1. **예측 가능한 성능(Predictable Performance)**: 모든 셰이더를 엔진 빌드 시점에 오프라인으로 컴파일
2. **계측 가능성(Instrumentable)**: 모든 GPU 리소스를 Dart DevTools에서 추적 가능
3. **이식성(Portable)**: 단일 코드베이스로 Metal, Vulkan, OpenGL ES 지원
4. **현대 GPU API 활용**: Metal(iOS/macOS), Vulkan(Android) 기반 직접 구현
5. **병렬 워크로드 분산**: CPU와 GPU 작업을 효율적으로 나눠 처리

### 오프라인 셰이더 컴파일 파이프라인

Impeller의 핵심 혁신은 **오프라인 셰이더 컴파일**입니다. 개발자가 `flutter build` 명령을 실행할 때 모든 셰이더가 미리 컴파일됩니다.

```
GLSL 4.60 셰이더 소스
        ↓ (spirv-cross 변환)
      SPIRV (중간 표현)
        ↓ (플랫폼별 트랜스파일)
  Metal Shading Language (iOS/macOS)
  Vulkan SPIR-V (Android)
  OpenGL ES GLSL (fallback)
        ↓ (컴파일 및 최적화)
   바이너리 블롭 + C++ 반사 바인딩
        ↓
    앱 번들에 포함
```

이 파이프라인 덕분에 앱이 실행될 때는 이미 GPU가 바로 사용할 수 있는 컴파일된 셰이더 바이너리가 번들에 포함되어 있습니다. 런타임 컴파일이 전혀 없으므로 Jank 자체가 구조적으로 불가능합니다.

## Impeller의 계층 구조

Impeller는 명확한 책임 분리를 가진 계층형 아키텍처로 구성됩니다:

```
Flutter Framework (Dart)
        ↓
  Display List Layer   ← Flutter의 flow 서브시스템과 통합
        ↓
  Entity Framework     ← 2D 렌더러 (파이프라인, 필터, 블렌드)
        ↓
  Renderer Layer       ← 백엔드 독립적 알로케이터, 테셀레이터
        ↓
  Geometry Layer       ← 수학 라이브러리, 경로, 변환
        ↓
  Backend (Metal / Vulkan / OpenGL ES)
        ↓
       GPU
```

각 계층의 역할:
- **Display List**: Flutter의 `PictureRecorder`가 기록한 그리기 명령을 Impeller Entity로 변환
- **Entity Framework**: 각 그리기 명령을 최적화된 GPU 파이프라인 상태와 연결
- **Renderer**: 버퍼, 텍스처, 파이프라인 상태를 추상화
- **Geometry**: `Path`, `Matrix`, `Rect` 등의 수학적 기본 요소

### 타일 기반 렌더링

Impeller는 화면을 약 256×256 픽셀 타일로 나눠 렌더링합니다. 각 프레임에서 내용이 변경된 타일만 재래스터화하므로, 정적인 배경 위에서 일부 요소만 움직이는 경우 GPU 부하를 크게 줄일 수 있습니다.

## 실제 구현 예제 1: Impeller 활성화 및 디버깅

Flutter 3.27 이후로 Android(API 29+)와 iOS 모두에서 Impeller가 기본값입니다. 하지만 명시적으로 설정하거나 비교 테스트를 하고 싶을 때가 있습니다.

### pubspec.yaml로 Impeller 강제 활성화

```yaml
flutter:
  # Android에서 Impeller 활성화 (Flutter 3.22+ 기본값)
  enable-impeller: true
```

### 런타임에서 렌더링 엔진 확인

```dart
import 'dart:ui' as ui;
import 'package:flutter/foundation.dart';

class RenderingInfo extends StatefulWidget {
  const RenderingInfo({super.key});

  @override
  State<RenderingInfo> createState() => _RenderingInfoState();
}

class _RenderingInfoState extends State<RenderingInfo> {
  String _rendererInfo = '';

  @override
  void initState() {
    super.initState();
    _loadRendererInfo();
  }

  Future<void> _loadRendererInfo() async {
    // Flutter 3.27+ 에서 렌더러 정보를 가져오는 방법
    // Impeller는 'Impeller', Skia는 'Skia'를 포함하는 문자열을 반환
    final info = await ui.ImpellerApiAvailable.query();
    setState(() {
      _rendererInfo = info?.rendererName ?? 'Unknown';
    });
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        Text(
          'Renderer: $_rendererInfo',
          style: const TextStyle(fontFamily: 'monospace'),
        ),
        // flutter --profile 빌드로 실행 후 DevTools > Performance > 
        // "Frame Chart"에서 렌더링 병목 지점을 타임라인으로 확인 가능
        Text(
          'Debug: kDebugMode=$kDebugMode, kProfileMode=$kProfileMode',
          style: const TextStyle(fontFamily: 'monospace', fontSize: 12),
        ),
      ],
    );
  }
}
```

명령줄에서 확인하는 방법:

```bash
# 디버그 빌드에서 Impeller 활성화 여부 확인
flutter run --verbose 2>&1 | grep -i impeller

# Impeller를 명시적으로 비활성화하고 Skia 사용 (비교 테스트용)
flutter run --no-enable-impeller

# 프로파일 모드에서 실행 (더 정확한 성능 측정)
flutter run --profile

# Impeller 강제 활성화 플래그 (이미 기본값인 플랫폼에서는 불필요)
flutter run --enable-impeller
```

## 실제 구현 예제 2: 성능 민감한 커스텀 페인터와 Impeller 최적화

Impeller는 `CustomPainter`의 `Canvas` API와 완벽하게 호환됩니다. 하지만 Impeller에서 최대 성능을 끌어내려면 몇 가지 패턴을 따라야 합니다.

```dart
import 'dart:math' as math;
import 'package:flutter/material.dart';

/// Impeller에 최적화된 파동 애니메이션 CustomPainter
/// 
/// 최적화 포인트:
/// 1. Path 객체를 매 프레임 재생성하지 않고 캐싱
/// 2. shouldRepaint에서 불필요한 재드로우 방지
/// 3. drawPath 대신 drawPoints로 간단한 도형은 단순화
class WavePainter extends CustomPainter {
  WavePainter({
    required this.animationValue,
    required this.waveColor,
    required this.amplitude,
    required this.frequency,
  }) : _paint = Paint()
         ..color = waveColor
         ..style = PaintingStyle.stroke
         ..strokeWidth = 2.5
         // Impeller는 AntiAlias를 GPU 파이프라인 상태에서 처리하므로
         // isAntiAlias를 false로 설정해도 품질 저하 없이 약간의 CPU 절약 가능
         ..isAntiAlias = true,
       _fillPaint = Paint()
         ..color = waveColor.withOpacity(0.15)
         ..style = PaintingStyle.fill;

  final double animationValue; // 0.0 ~ 1.0
  final Color waveColor;
  final double amplitude;
  final double frequency;

  final Paint _paint;
  final Paint _fillPaint;

  @override
  void paint(Canvas canvas, Size size) {
    final path = Path();
    final fillPath = Path();

    final centerY = size.height / 2;
    final phaseShift = animationValue * 2 * math.pi;

    // 파도 경로 생성: 수평 방향으로 점들을 잇는 부드러운 곡선
    path.moveTo(0, centerY);
    fillPath.moveTo(0, size.height);
    fillPath.lineTo(0, centerY);

    // Impeller의 테셀레이터는 복잡한 Path를 효율적으로 GPU 삼각형으로 변환
    // cubicTo를 활용한 부드러운 사인파
    const steps = 60;
    final stepWidth = size.width / steps;

    for (int i = 0; i <= steps; i++) {
      final x = i * stepWidth;
      final y = centerY + amplitude * math.sin(frequency * (x / size.width) * 2 * math.pi + phaseShift);

      if (i == 0) {
        path.moveTo(x, y);
        fillPath.lineTo(x, y);
      } else {
        // 제어점을 이용한 부드러운 연결 (conicTo보다 cubicTo가 Impeller에서 더 최적화됨)
        final prevX = (i - 1) * stepWidth;
        final prevY = centerY + amplitude * math.sin(frequency * (prevX / size.width) * 2 * math.pi + phaseShift);
        final cpX = (prevX + x) / 2;
        path.cubicTo(cpX, prevY, cpX, y, x, y);
        fillPath.lineTo(x, y);
      }
    }

    // 채우기 경로 닫기
    fillPath.lineTo(size.width, size.height);
    fillPath.close();

    // Impeller에서 saveLayer는 오프스크린 버퍼를 생성하므로 꼭 필요할 때만 사용
    // 단순 투명도라면 paint.color.withOpacity()를 직접 사용하는 것이 더 효율적
    canvas.drawPath(fillPath, _fillPaint);
    canvas.drawPath(path, _paint);

    // 두 번째 파도 (위상 차 적용) - 레이어 저장 없이 동일 Canvas에 그리기
    final wave2Paint = Paint()
      ..color = waveColor.withOpacity(0.5)
      ..style = PaintingStyle.stroke
      ..strokeWidth = 1.5;

    final wave2Path = _buildWavePath(size, centerY + amplitude * 0.3, phaseShift + math.pi);
    canvas.drawPath(wave2Path, wave2Paint);
  }

  Path _buildWavePath(Size size, double centerY, double phaseShift) {
    final path = Path();
    const steps = 60;
    final stepWidth = size.width / steps;

    for (int i = 0; i <= steps; i++) {
      final x = i * stepWidth;
      final y = centerY + (amplitude * 0.6) * math.sin(frequency * (x / size.width) * 2 * math.pi + phaseShift);

      if (i == 0) {
        path.moveTo(x, y);
      } else {
        final prevX = (i - 1) * stepWidth;
        final prevY = centerY + (amplitude * 0.6) * math.sin(frequency * (prevX / size.width) * 2 * math.pi + phaseShift);
        final cpX = (prevX + x) / 2;
        path.cubicTo(cpX, prevY, cpX, y, x, y);
      }
    }
    return path;
  }

  @override
  bool shouldRepaint(WavePainter oldDelegate) {
    // animationValue가 변경될 때만 재드로우
    return oldDelegate.animationValue != animationValue ||
           oldDelegate.waveColor != waveColor ||
           oldDelegate.amplitude != amplitude;
  }
}

/// WavePainter를 사용하는 위젯
class AnimatedWave extends StatefulWidget {
  const AnimatedWave({
    super.key,
    this.color = Colors.blue,
    this.amplitude = 20.0,
    this.frequency = 2.0,
  });

  final Color color;
  final double amplitude;
  final double frequency;

  @override
  State<AnimatedWave> createState() => _AnimatedWaveState();
}

class _AnimatedWaveState extends State<AnimatedWave>
    with SingleTickerProviderStateMixin {
  late final AnimationController _controller;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(seconds: 3),
    )..repeat(); // 무한 반복
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: _controller,
      builder: (context, child) {
        return CustomPaint(
          painter: WavePainter(
            animationValue: _controller.value,
            waveColor: widget.color,
            amplitude: widget.amplitude,
            frequency: widget.frequency,
          ),
          // RepaintBoundary는 Impeller에서도 레이어 분리를 명시적으로 지정할 때 유용
          // 단, 불필요한 RepaintBoundary는 오히려 오프스크린 버퍼를 낭비할 수 있음
          size: const Size(double.infinity, 200),
        );
      },
    );
  }
}
```

### saveLayer 주의사항

Impeller에서 `canvas.saveLayer()`는 오프스크린 렌더 패스를 생성합니다. 이는 GPU 메모리 대역폭을 추가로 소비하므로 다음 경우에만 사용하세요:

- `BlendMode`가 `dstIn`, `xor` 등 복합 혼합 모드가 필요할 때
- 여러 드로잉 명령을 하나의 투명도 레이어로 묶을 때

단순히 투명도를 적용하려면 `Paint.color.withOpacity()`를 직접 사용하는 것이 약 3배 이상 효율적입니다.

## Impeller의 플랫폼별 동작

### iOS (Metal 백엔드)

iOS에서 Impeller는 Apple의 Metal API를 직접 사용합니다. Flutter 3.10(2023년 5월)부터 iOS에서 Impeller가 기본값이 되었으며, Flutter 3.27에서는 Skia 폴백이 완전히 제거되었습니다.

Metal은 OpenGL ES보다 CPU 드라이버 오버헤드가 크게 낮고, 명시적인 파이프라인 상태 관리를 통해 GPU를 더 효율적으로 활용할 수 있습니다.

### Android (Vulkan 백엔드)

Android에서 Impeller는 Vulkan API를 사용합니다. Android API 29(Android 10) 이상에서 기본값으로 활성화되며, 그 이하에서는 OpenGL ES 백엔드로 폴백합니다.

Vulkan은 멀티스레드 커맨드 버퍼 기록을 지원하므로, Impeller는 Raster 스레드와 별도의 백그라운드 스레드에서 GPU 명령을 준비할 수 있습니다.

```
Flutter UI 스레드     ←→  Dart 코드 실행, 위젯 빌드
Flutter Raster 스레드  ←→  Display List 처리, 커맨드 버퍼 구성
Impeller Worker 스레드  ←→  텍스처 업로드, 리소스 관리
GPU                    ←→  실제 렌더링
```

### macOS/Linux/Windows (Preview)

macOS에서는 Metal, Linux/Windows에서는 Vulkan 또는 OpenGL을 통해 Impeller를 실험적으로 사용할 수 있습니다. `AndroidManifest.xml`이나 `Info.plist`에 플래그를 추가하지 않아도, `flutter run --enable-impeller` 플래그로 테스트할 수 있습니다.

## Impeller 전환 시 주의사항과 팁

### 1. 텍스트 렌더링 차이

Impeller는 자체 서체 시스템(Typographer)을 갖고 있습니다. 일부 복잡한 언어(아랍어, 힌디어 등 복잡한 서자법)나 특수 폰트에서 미묘한 텍스트 렌더링 차이가 발생할 수 있습니다. 다국어 앱이라면 반드시 실제 디바이스에서 텍스트를 검증하세요.

### 2. 특정 BlendMode 지원

Impeller 초기 버전에서는 일부 희귀한 `BlendMode` 조합이 지원되지 않았습니다. Flutter 3.27 기준으로 대부분 해결되었지만, 커스텀 블렌드를 사용하는 앱이라면 UI 비교 테스트를 수행하세요.

```dart
// Impeller에서 잘 동작하는 일반적인 BlendMode
Paint()
  ..blendMode = BlendMode.srcOver   // 기본값, 완벽 지원
  ..blendMode = BlendMode.multiply  // 지원
  ..blendMode = BlendMode.screen    // 지원
  ..blendMode = BlendMode.overlay   // 지원

// 복잡한 BlendMode - 최신 Impeller에서 지원하지만 성능 영향 확인 필요
Paint()
  ..blendMode = BlendMode.luminosity
  ..blendMode = BlendMode.hue
```

### 3. 셰이더 기반 이펙트와 Impeller

`dart:ui`의 `FragmentProgram`을 이용한 커스텀 GLSL 셰이더도 Impeller에서 동작합니다. 단, Impeller는 셰이더를 빌드 타임에 검증하므로 셰이더 문법 오류가 있으면 앱 빌드 자체가 실패합니다. Skia에서는 런타임에야 오류가 드러났던 것과 대조적입니다.

```dart
// pubspec.yaml에 셰이더 등록
// flutter:
//   shaders:
//     - shaders/my_effect.frag

import 'dart:ui' as ui;
import 'package:flutter/services.dart';

Future<ui.FragmentShader> loadShader() async {
  // Impeller 빌드 시 이미 컴파일된 셰이더 바이너리를 로드
  final program = await ui.FragmentProgram.fromAsset('shaders/my_effect.frag');
  return program.fragmentShader();
}
```

### 4. Golden 테스트 업데이트

Impeller와 Skia는 안티앨리어싱 알고리즘이 달라 픽셀 단위 렌더링 결과가 다를 수 있습니다. Impeller로 전환할 때 기존 Golden 테스트 이미지를 반드시 갱신해야 합니다.

```bash
# Golden 테스트 이미지 갱신
flutter test --update-goldens
```

### 5. Impeller 버그 리포팅

만약 Impeller에서 특정 UI 렌더링 이슈를 발견했다면 다음 절차를 따르세요:

1. `--no-enable-impeller` 플래그로 동일 이슈가 Skia에서도 재현되는지 확인
2. Impeller에서만 재현된다면 [flutter/flutter GitHub Issues](https://github.com/flutter/flutter/issues)에 `impeller` 레이블과 함께 보고
3. 재현 코드와 함께 `flutter doctor -v` 출력, 기기 모델, Flutter 버전 첨부

## Skia vs Impeller 성능 비교

Flutter 팀의 공식 벤치마크 및 커뮤니티 테스트 결과(2025 기준):

| 항목                      | Skia            | Impeller          |
|---------------------------|-----------------|-------------------|
| 첫 애니메이션 프레임 드롭  | 빈번 (~12% 드롭) | 거의 없음 (~1.5%) |
| 정적 UI 프레임 시간        | 비슷함          | 비슷하거나 약간 개선 |
| 복잡한 Path 렌더링         | 보통            | 타일 기반으로 개선 |
| 텍스트 렌더링 품질         | 높음            | 동등 (일부 폰트 주의) |
| 메모리 사용량              | 기준            | 약간 증가 (셰이더 캐시) |
| 120Hz 고주사율 유지        | 불안정          | 안정적             |

결론적으로 Impeller는 Jank 제거에서 압도적인 우위를 보이며, 이것이 Flutter 팀이 전환을 결정한 핵심 이유입니다.

## 정리

Flutter Impeller는 Skia의 런타임 셰이더 컴파일이라는 구조적 한계를 오프라인 컴파일로 해결한 근본적인 아키텍처 혁신입니다. Metal과 Vulkan이라는 현대 GPU API를 직접 활용하고, 타일 기반 렌더링과 명확한 계층 분리로 예측 가능하고 안정적인 60~120FPS를 달성합니다.

Flutter 3.27 기준으로 Android/iOS 모두에서 기본값이 된 Impeller를 이제 "사용할 것인가 말 것인가"가 아닌, **"어떻게 최대한 활용할 것인가"**의 관점에서 이해해야 합니다. CustomPainter 최적화, saveLayer 절약, 셰이더 등록 방식을 숙지하면 Impeller의 잠재력을 완전히 끌어낼 수 있습니다.

## 참고 자료
- [Flutter Impeller 공식 README (flutter/engine)](https://github.com/flutter/flutter/blob/master/engine/src/flutter/impeller/README.md)
- [Impeller Wiki (flutter/flutter)](https://github.com/flutter/flutter/wiki/Impeller)
- [Flutter Impeller Deep Dive: Eliminating Shader Compilation Jank (XsOne Consultants)](https://xsoneconsultants.com/blog/flutter-impeller-engine/)
- [How Impeller Is Transforming Flutter UI Rendering in 2026 (DEV Community)](https://dev.to/eira-wexford/how-impeller-is-transforming-flutter-ui-rendering-in-2026-3dpd)
