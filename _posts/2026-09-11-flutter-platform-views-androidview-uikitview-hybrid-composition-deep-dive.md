---
layout: post
title: "Flutter Platform Views 심화: AndroidView·UiKitView와 세 가지 렌더링 전략"
date: 2026-09-11
categories: [flutter, android]
tags: [flutter, platform-views, androidview, uikitview, hybrid-composition, native-view, kotlin, dart]
---

## 개요

Flutter는 자체 렌더링 엔진(Impeller/Skia)으로 모든 위젯을 직접 그립니다. 덕분에 플랫폼에 관계없이 픽셀 퍼펙트한 UI를 제공하지만, 역설적으로 **플랫폼 고유 뷰**(Google Maps SDK 지도, 네이티브 WebView, 카메라 미리보기, ARCore 뷰 등)를 Flutter 위젯 트리 안에 삽입하는 일은 결코 단순하지 않습니다.

이 글에서는 Flutter의 **Platform Views** 메커니즘을 깊이 파고듭니다. Android 측 Kotlin 코드와 Dart 측 코드를 실전 수준으로 작성하고, 세 가지 렌더링 전략(Virtual Display, Hybrid Composition, HCPP)의 트레이드오프를 분석합니다.

---

## Platform Views란?

Platform Views는 Flutter 위젯 트리 안에 **OS 네이티브 뷰(Android: `View`, iOS: `UIView`)**를 삽입할 수 있게 해주는 Flutter 프레임워크의 확장 지점입니다. Dart 코드에서는 `AndroidView` 또는 `UiKitView` 위젯을 선언하고, 플랫폼 코드에서는 `FlutterPlatformView`와 `PlatformViewFactory`를 구현하여 서로 연결합니다.

내부적으로 Flutter 엔진은 다음 세 가지 렌더링 전략 중 하나를 선택합니다.

| 전략 | 설명 | Android 요구 버전 |
|------|------|----------|
| **Virtual Display** | 네이티브 뷰를 가상 디스플레이에 렌더링 후 Flutter 텍스처로 합성 | API 20+ |
| **Hybrid Composition** | 네이티브 뷰를 실제 뷰 계층에 삽입, Flutter 레이어와 합성 | API 20+ |
| **HCPP (Hybrid Composition++)** | Vulkan 동기화 기반 최적화 버전 | API 34+, Vulkan |

---

## 왜 Platform Views가 필요한가?

다음 상황에서 Platform Views가 필수입니다.

1. **SDK 전용 뷰**: Google Maps SDK, Mapbox SDK, Naver Maps SDK 등 네이티브 렌더링에 의존하는 지도 위젯
2. **시스템 수준 렌더링**: 카메라 프리뷰(`TextureView`/`SurfaceView`), 미디어 플레이어 뷰
3. **접근성(Accessibility)**: TalkBack·VoiceOver가 올바르게 동작하려면 시스템 접근성 트리에 실제 뷰가 있어야 합니다
4. **WebView**: `WebView`는 자체 JS 엔진, 렌더러, 쿠키 저장소를 갖고 있어 Flutter로 재구현이 불가합니다
5. **레거시 네이티브 컴포넌트 재사용**: 기존 Android/iOS 커스텀 뷰 자산을 Flutter 앱에 점진적으로 통합할 때

---

## 실전 구현 예제 1: Android 측 (Kotlin)

간단한 예제로 **빨간 사각형 네이티브 뷰**를 Flutter에 노출해 보겠습니다. 실제 업무에서는 이 자리에 지도 SDK 초기화 코드나 미디어 뷰어가 들어갑니다.

### 1-1. FlutterPlatformView 구현

```kotlin
// NativeRedView.kt
import android.content.Context
import android.graphics.Color
import android.view.View
import android.widget.FrameLayout
import io.flutter.plugin.platform.PlatformView

class NativeRedView(
    private val context: Context,
    private val creationParams: Map<String, Any>?
) : PlatformView {

    private val container: FrameLayout = FrameLayout(context).apply {
        setBackgroundColor(Color.RED)
        // creationParams에서 초기 설정값 읽기
        val label = creationParams?.get("label") as? String ?: "Native View"
        addView(
            android.widget.TextView(context).apply {
                text = label
                setTextColor(Color.WHITE)
                textSize = 18f
                gravity = android.view.Gravity.CENTER
            },
            FrameLayout.LayoutParams(
                FrameLayout.LayoutParams.MATCH_PARENT,
                FrameLayout.LayoutParams.MATCH_PARENT
            )
        )
    }

    // Flutter 엔진이 표시할 실제 Android View
    override fun getView(): View = container

    // 뷰가 위젯 트리에서 제거될 때 리소스 해제
    override fun dispose() {
        // 필요한 경우 리소스 정리 (리스너 제거, SDK 해제 등)
    }
}
```

### 1-2. PlatformViewFactory 구현

```kotlin
// NativeRedViewFactory.kt
import android.content.Context
import io.flutter.plugin.common.StandardMessageCodec
import io.flutter.plugin.platform.PlatformView
import io.flutter.plugin.platform.PlatformViewFactory

class NativeRedViewFactory : PlatformViewFactory(StandardMessageCodec.INSTANCE) {

    override fun create(
        context: Context,
        viewId: Int,
        args: Any?
    ): PlatformView {
        @Suppress("UNCHECKED_CAST")
        val creationParams = args as? Map<String, Any>
        return NativeRedView(context, creationParams)
    }
}
```

### 1-3. Plugin에 Factory 등록

```kotlin
// NativeViewPlugin.kt
import io.flutter.embedding.engine.plugins.FlutterPlugin

class NativeViewPlugin : FlutterPlugin {

    override fun onAttachedToEngine(binding: FlutterPlugin.FlutterPluginBinding) {
        binding.platformViewRegistry.registerViewFactory(
            "com.example.native_red_view",  // Dart에서 사용할 viewType 키
            NativeRedViewFactory()
        )
    }

    override fun onDetachedFromEngine(binding: FlutterPlugin.FlutterPluginBinding) {
        // 필요 시 정리 로직
    }
}
```

`AndroidManifest.xml`이나 `FlutterActivity`의 `configureFlutterEngine()`에서 플러그인을 등록합니다.

```kotlin
// MainActivity.kt
import io.flutter.embedding.android.FlutterActivity
import io.flutter.embedding.engine.FlutterEngine

class MainActivity : FlutterActivity() {
    override fun configureFlutterEngine(flutterEngine: FlutterEngine) {
        super.configureFlutterEngine(flutterEngine)
        flutterEngine.plugins.add(NativeViewPlugin())
    }
}
```

---

## 실전 구현 예제 2: Dart 측

### 2-1. AndroidView 단순 사용

```dart
// native_red_view_widget.dart
import 'package:flutter/foundation.dart';
import 'package:flutter/gestures.dart';
import 'package:flutter/material.dart';
import 'package:flutter/rendering.dart';
import 'package:flutter/services.dart';

class NativeRedViewWidget extends StatelessWidget {
  const NativeRedViewWidget({super.key, this.label = 'Hello from Native!'});

  final String label;

  @override
  Widget build(BuildContext context) {
    const viewType = 'com.example.native_red_view';
    final creationParams = <String, dynamic>{'label': label};

    if (defaultTargetPlatform == TargetPlatform.android) {
      // Hybrid Composition 방식 (권장)
      return PlatformViewLink(
        viewType: viewType,
        surfaceFactory: (context, controller) {
          return AndroidViewSurface(
            controller: controller as AndroidViewController,
            gestureRecognizers: const <Factory<OneSequenceGestureRecognizer>>{},
            hitTestBehavior: PlatformViewHitTestBehavior.opaque,
          );
        },
        onCreatePlatformView: (params) {
          return PlatformViewsService.initSurfaceAndroidView(
            id: params.id,
            viewType: viewType,
            layoutDirection: TextDirection.ltr,
            creationParams: creationParams,
            creationParamsCodec: const StandardMessageCodec(),
            onFocus: () => params.onFocusChanged(true),
          )
            ..addOnPlatformViewCreatedListener(params.onPlatformViewCreated)
            ..create();
        },
      );
    } else if (defaultTargetPlatform == TargetPlatform.iOS) {
      return UiKitView(
        viewType: viewType,
        layoutDirection: TextDirection.ltr,
        creationParams: creationParams,
        creationParamsCodec: const StandardMessageCodec(),
      );
    }
    return const Placeholder();
  }
}
```

### 2-2. 제스처 충돌 처리 (EagerGestureRecognizer)

플랫폼 뷰가 스크롤 뷰 내부에 있으면 Flutter의 제스처 경쟁이 발생합니다. 아래처럼 전용 인식기를 등록해 네이티브 뷰가 제스처를 우선 소비하도록 합니다.

```dart
// 스크롤 내부 플랫폼 뷰용 제스처 설정
PlatformViewLink(
  viewType: 'com.example.native_red_view',
  surfaceFactory: (context, controller) {
    return AndroidViewSurface(
      controller: controller as AndroidViewController,
      gestureRecognizers: <Factory<OneSequenceGestureRecognizer>>{
        // 수직 드래그를 네이티브 뷰가 먼저 처리
        Factory<VerticalDragGestureRecognizer>(
          () => VerticalDragGestureRecognizer(),
        ),
        // 탭도 네이티브에게 위임
        Factory<TapGestureRecognizer>(
          () => TapGestureRecognizer(),
        ),
      },
      hitTestBehavior: PlatformViewHitTestBehavior.opaque,
    );
  },
  onCreatePlatformView: (params) {
    return PlatformViewsService.initSurfaceAndroidView(
      id: params.id,
      viewType: 'com.example.native_red_view',
      layoutDirection: TextDirection.ltr,
      creationParamsCodec: const StandardMessageCodec(),
    )
      ..addOnPlatformViewCreatedListener(params.onPlatformViewCreated)
      ..create();
  },
)
```

---

## 세 가지 렌더링 전략 심층 분석

### Virtual Display (레거시)

네이티브 뷰를 `VirtualDisplay`(가상 화면 버퍼)에 렌더링하고, 그 결과를 `SurfaceTexture`를 통해 Flutter 텍스처로 업로드합니다.

- **장점**: Flutter 레이어와 직접 합성되므로 Elevation/Shadow 등 Flutter 효과 적용 가능
- **단점**: 텍스트 입력(`TextField`)이 제대로 동작하지 않음, IME 키보드 처리 불안정, 접근성 트리와 분리됨

### Hybrid Composition (HC)

네이티브 뷰를 실제 Android ViewGroup 계층에 삽입하고, Flutter 콘텐츠를 그 위아래로 나눠서 렌더링합니다.

- **장점**: 텍스트 입력 완벽 지원, TalkBack 접근성 정상 동작, 정확한 터치 처리
- **단점**: Android 10 미만에서 각 Flutter 프레임을 메인 메모리로 복사 → GPU 메모리 ↑, FPS ↓. Flutter 래스터 스레드와 플랫폼 스레드가 병합되어 경쟁 발생

### HCPP (Hybrid Composition++)

Android API 34 + Vulkan 환경에서 `SurfaceControl` 트랜잭션 동기화를 사용해 HC의 성능 병목을 해소한 버전입니다. `flutter build apk`로 빌드할 때 자동으로 활성화됩니다.

- **장점**: HC 장점 유지 + GPU 메모리 복사 제거 → 성능이 Virtual Display 수준으로 개선
- **단점**: API 34 미만 기기에서는 HC로 폴백

---

## 주의사항 및 팁

### 1. initExpensiveAndroidView vs initSurfaceAndroidView

```dart
// Virtual Display 방식 (레거시, 텍스트 입력 깨짐)
PlatformViewsService.initExpensiveAndroidView(...)

// Hybrid Composition 방식 (권장)
PlatformViewsService.initSurfaceAndroidView(...)
```

신규 개발에서는 반드시 `initSurfaceAndroidView`를 사용하세요.

### 2. 메모리 누수 방지

`PlatformView.dispose()`에서 SDK 리소스를 반드시 해제하세요. Google Maps 같은 무거운 SDK는 `dispose()`를 놓치면 GL 컨텍스트가 해제되지 않아 수백 MB 메모리 누수가 발생합니다.

```kotlin
override fun dispose() {
    mapView.onDestroy()  // 예: Google Map
    // 각 SDK 문서에서 종료 메서드 확인 필수
}
```

### 3. iOS에서 `PlatformViewFactory` 구현

iOS는 `FlutterPlatformViewFactory`와 `FlutterPlatformView` 프로토콜을 Swift로 구현합니다. `UiKitView`는 내부적으로 Scissor-based 합성을 사용해 HC와 유사한 방식으로 동작합니다.

### 4. 성능 측정

`flutter run --profile` 후 DevTools의 **Performance** 탭에서 `PlatformView#draw` 항목을 확인하세요. 이 항목이 프레임 예산의 30% 이상을 차지하면 리팩터링을 고려해야 합니다.

### 5. 크기 제약 주의

Platform View는 Flutter 레이아웃 엔진으로부터 크기를 전달받습니다. `Expanded` 또는 `SizedBox`로 명시적 크기를 주지 않으면 뷰가 렌더링되지 않거나 크기가 0이 되는 문제가 발생합니다.

```dart
SizedBox(
  width: 300,
  height: 200,
  child: NativeRedViewWidget(label: '크기 명시 필수'),
)
```

---

## 마무리

Platform Views는 Flutter와 네이티브 SDK 사이의 다리입니다. 렌더링 전략을 잘못 선택하면 텍스트 입력 불능, FPS 저하, 접근성 오동작 같은 심각한 문제로 이어집니다. 핵심 원칙은 다음과 같습니다.

- **신규 코드**: `PlatformViewLink + initSurfaceAndroidView` (Hybrid Composition)
- **Android 34+**: HCPP가 자동 적용되므로 추가 설정 불필요
- **제스처 충돌**: `gestureRecognizers`로 명시적으로 제어
- **dispose**: SDK 리소스 해제를 절대 누락하지 않는다

## 참고 자료
- [Hosting native Android views in your Flutter app with Platform Views](https://docs.flutter.dev/platform-integration/android/platform-views)
- [Flutter Hybrid Composition 공식 Wiki](https://github.com/flutter/flutter/wiki/Hybrid-Composition)
- [The Evolution of Flutter PlatformView — Medium/GSYTech](https://medium.com/@GSYTech/the-evolution-of-flutter-platformview-8486e9cd62d3)
- [Integration of Flutter Apps with Native Features — KINTO Tech Blog](https://blog.kinto-technologies.com/posts/2024-12-17-flutter-platform-view-android-en/)
