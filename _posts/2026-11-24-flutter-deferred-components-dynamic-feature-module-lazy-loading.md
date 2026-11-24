---
layout: post
title: "Flutter Deferred Components 심화: Dynamic Feature Module로 앱 첫 설치 용량을 절감하는 지연 로딩 전략 완전 정복"
date: 2026-11-24
categories: [android, flutter]
tags: [flutter, dart, deferred-components, dynamic-feature-module, lazy-loading, app-bundle, performance, app-size]
---

앱 스토어에서 사용자가 앱을 설치할 때 첫 번째 장벽은 **설치 파일 크기**다. Play Store 통계에 따르면 APK가 100MB를 넘으면 다운로드 완료율이 급격히 떨어진다. Flutter는 이 문제를 해결하기 위해 **Deferred Components**를 지원한다. Android Dynamic Feature Module과 Dart의 지연 임포트(`deferred as`)를 결합해, 앱 실행 중 필요한 시점에 Dart 코드와 애셋을 추가로 내려받는 메커니즘이다.

이 글에서는 Deferred Components의 내부 동작 원리부터 pubspec.yaml 설정, 실제 위젯 구현, 주의사항까지 단계별로 살펴본다.

---

## 1. 왜 Deferred Components인가?

Flutter 앱의 릴리스 빌드는 `gen_snapshot`이 모든 Dart 코드를 하나의 거대한 AOT(Ahead-of-Time) 스냅샷으로 컴파일한다. 기능이 늘어날수록 이 스냅샷 크기도 함께 커진다. Deferred Components는 이 스냅샷을 **여러 개의 로딩 유닛(Loading Unit)**으로 쪼개는 방식이다.

| 방식 | 최초 설치 크기 | 추가 기능 로드 |
|---|---|---|
| 기본 App Bundle | 전체 Dart AOT 포함 | — |
| Deferred Components | 핵심 AOT만 포함 | 런타임에 Play Store에서 다운로드 |

**로딩 유닛**이란 `deferred` 키워드로만 임포트되는 라이브러리 집합 중, 베이스 라이브러리에서 분리 가능한 최소 단위다. `gen_snapshot`이 `--loading_unit_manifest` 플래그를 받으면 분리된 AOT 아티팩트를 생성하고, 각 로딩 유닛은 별도의 Dynamic Feature Module에 패키징된다.

---

## 2. 아키텍처 개요

```
[Flutter App]
  ├── base module          ← 항상 설치됨 (필수 Dart 코드 + 핵심 애셋)
  ├── feature_ar module    ← AR 기능 (사용자가 AR 화면 진입 시 다운로드)
  └── feature_map module   ← 지도 기능 (사용자가 지도 화면 진입 시 다운로드)
```

동작 순서:
1. 사용자가 AR 화면으로 이동
2. `loadLibrary()` 호출 → `DeferredComponent.installDeferredComponent()` 트리거
3. Play Store의 Play Core 라이브러리가 `feature_ar` 모듈 다운로드
4. 다운로드 완료 후 `loadLibrary()` Future가 완료
5. AR 위젯 렌더링

---

## 3. 프로젝트 설정

### pubspec.yaml

```yaml
flutter:
  deferred-components:
    - name: feature_ar
      libraries:
        - package:my_app/src/ar/ar_view.dart
      assets:
        - assets/ar_models/robot.glb
        - assets/ar_models/car.glb

    - name: feature_map
      libraries:
        - package:my_app/src/map/map_screen.dart
        - package:my_app/src/map/map_controller.dart
      assets:
        - assets/map_tiles/

  assets:
    - assets/images/
    - assets/fonts/
```

> `libraries`에 명시된 Dart 파일은 반드시 최종적으로 `deferred as`로만 임포트되어야 로딩 유닛에서 분리된다. 일반 임포트가 하나라도 있으면 베이스 모듈에 포함된다.

---

## 4. Dart 지연 임포트 코드 예제

### 예제 1: 기본 deferred import와 loadLibrary() 패턴

```dart
// lib/src/main_navigation.dart

import 'package:flutter/material.dart';

// deferred as 키워드로 지연 임포트
import 'package:my_app/src/ar/ar_view.dart' deferred as arFeature;
import 'package:my_app/src/map/map_screen.dart' deferred as mapFeature;

class FeatureLauncher extends StatefulWidget {
  final String destination; // 'ar' | 'map'
  const FeatureLauncher({super.key, required this.destination});

  @override
  State<FeatureLauncher> createState() => _FeatureLauncherState();
}

class _FeatureLauncherState extends State<FeatureLauncher> {
  bool _isLoading = false;
  String? _errorMessage;

  Future<void> _loadAndNavigate() async {
    setState(() {
      _isLoading = true;
      _errorMessage = null;
    });

    try {
      if (widget.destination == 'ar') {
        // loadLibrary()가 완료되기 전까지 arFeature의 심볼에 접근하면 안 된다.
        await arFeature.loadLibrary();
        if (!mounted) return;
        await Navigator.of(context).push(
          MaterialPageRoute(builder: (_) => arFeature.ArView()),
        );
      } else if (widget.destination == 'map') {
        await mapFeature.loadLibrary();
        if (!mounted) return;
        await Navigator.of(context).push(
          MaterialPageRoute(builder: (_) => mapFeature.MapScreen()),
        );
      }
    } catch (e) {
      setState(() {
        _errorMessage = '기능 로드 실패: $e';
      });
    } finally {
      if (mounted) setState(() => _isLoading = false);
    }
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      mainAxisSize: MainAxisSize.min,
      children: [
        if (_isLoading)
          const CircularProgressIndicator()
        else
          ElevatedButton(
            onPressed: _loadAndNavigate,
            child: Text(
              widget.destination == 'ar' ? 'AR 기능 열기' : '지도 열기',
            ),
          ),
        if (_errorMessage != null)
          Text(_errorMessage!, style: const TextStyle(color: Colors.red)),
      ],
    );
  }
}
```

**핵심 포인트:**
- `loadLibrary()`는 **Future**를 반환한다. `await` 없이 임포트한 심볼에 접근하면 `Null check operator used on a null value` 런타임 에러가 발생한다.
- 디버그 모드에서는 `loadLibrary()`가 즉시 완료되므로 로딩 UI가 사실상 보이지 않는다. 릴리스/프로파일 모드에서만 실제 네트워크 다운로드가 발생한다.
- `loadLibrary()`는 멱등적이다. 이미 로드된 라이브러리에 대해 다시 호출해도 안전하다.

---

### 예제 2: DeferredWidget 헬퍼 클래스로 선언적 처리

복수의 지연 로딩 화면이 있을 때, 각 화면마다 로딩/에러 상태를 반복 작성하는 것은 비효율적이다. Flutter Gallery의 `deferred_widget.dart`에서 착안한 범용 헬퍼를 만들어보자.

```dart
// lib/src/common/deferred_widget.dart

import 'package:flutter/material.dart';

typedef LibraryLoader = Future<void> Function();
typedef DeferredWidgetBuilder = Widget Function();

/// 지연 로딩 완료 전 플레이스홀더를 보여주고,
/// 완료 후 실제 위젯을 렌더링하는 범용 래퍼.
class DeferredWidget extends StatefulWidget {
  const DeferredWidget({
    super.key,
    required this.libraryLoader,
    required this.createWidget,
    this.placeholder,
    this.errorBuilder,
  });

  /// loadLibrary()를 호출하는 함수. 예: () => arFeature.loadLibrary()
  final LibraryLoader libraryLoader;

  /// 로드 완료 후 렌더링할 위젯 생성 함수.
  final DeferredWidgetBuilder createWidget;

  /// 로딩 중 보여줄 위젯 (기본값: CircularProgressIndicator).
  final Widget? placeholder;

  /// 에러 발생 시 보여줄 위젯 빌더.
  final Widget Function(Object error)? errorBuilder;

  @override
  State<DeferredWidget> createState() => _DeferredWidgetState();
}

class _DeferredWidgetState extends State<DeferredWidget> {
  // null: 로딩 중 / false: 에러 / true: 완료
  bool? _loaded;
  Object? _error;

  // 같은 loader에 대해 중복 로드를 막기 위한 static 캐시
  static final Map<LibraryLoader, Future<void>> _cache = {};

  @override
  void initState() {
    super.initState();
    _loadLibrary();
  }

  Future<void> _loadLibrary() async {
    try {
      // 이미 로드 중이거나 완료된 loader는 캐시에서 꺼내 재사용
      await (_cache[widget.libraryLoader] ??=
          widget.libraryLoader());
      if (mounted) setState(() => _loaded = true);
    } catch (e) {
      if (mounted) setState(() {
        _loaded = false;
        _error = e;
      });
    }
  }

  @override
  Widget build(BuildContext context) {
    if (_loaded == true) {
      return widget.createWidget();
    }
    if (_loaded == false) {
      return widget.errorBuilder?.call(_error!) ??
          Center(
            child: Column(
              mainAxisSize: MainAxisSize.min,
              children: [
                const Icon(Icons.error_outline, color: Colors.red, size: 48),
                const SizedBox(height: 8),
                Text('로드 실패: $_error'),
                TextButton(
                  onPressed: () {
                    _cache.remove(widget.libraryLoader);
                    setState(() { _loaded = null; _error = null; });
                    _loadLibrary();
                  },
                  child: const Text('다시 시도'),
                ),
              ],
            ),
          );
    }
    // 로딩 중
    return widget.placeholder ??
        const Center(child: CircularProgressIndicator());
  }
}
```

**사용 예시:**

```dart
// AR 화면 라우트
import 'package:my_app/src/ar/ar_view.dart' deferred as arFeature;

Widget buildArRoute() => DeferredWidget(
  libraryLoader: arFeature.loadLibrary,
  createWidget: () => arFeature.ArView(),
  placeholder: const Scaffold(
    body: Center(child: Text('AR 모듈 다운로드 중...')),
  ),
);
```

이 패턴의 핵심은 `static Map<LibraryLoader, Future<void>> _cache`다. 같은 함수 레퍼런스를 키로 사용하므로, 사용자가 화면을 나갔다가 다시 들어와도 이미 완료된 `Future`를 재사용한다.

---

## 5. Android 프로젝트 설정

Flutter 툴이 `flutter build appbundle --deferred-components`를 처음 실행하면 `android/` 폴더에 각 컴포넌트별 Dynamic Feature Module 디렉터리를 자동 생성한다.

```
android/
  app/
    build.gradle          ← 기본 모듈
  feature_ar/
    build.gradle          ← DFM: apply plugin: 'com.android.dynamic-feature'
    src/main/
      AndroidManifest.xml ← <dist:module dist:instant="false">
  feature_map/
    build.gradle
    src/main/
      AndroidManifest.xml
```

**`android/app/build.gradle`** 에 DFM 모듈을 선언해야 한다:

```gradle
android {
    // ...
    dynamicFeatures = [":feature_ar", ":feature_map"]
}
```

**빌드 명령:**

```bash
# 첫 실행 시 DFM 디렉터리 자동 생성
flutter build appbundle

# 생성된 로딩 유닛 확인
cat deferred_components_loading_units.yaml
```

---

## 6. DeferredComponent 서비스 API

Flutter의 `services` 패키지는 `DeferredComponent` 클래스를 제공한다. 이를 직접 사용하면 로딩 진행률을 추적할 수 있다.

```dart
import 'package:flutter/services.dart';

Future<void> installWithProgress(String componentName) async {
  // 1. 설치 요청
  await DeferredComponent.installDeferredComponent(
    componentName: componentName,
  );
  // 2. 설치 완료 확인 (실제로는 위 Future가 완료될 때 설치됨)
}

Future<void> uninstallIfNeeded(String componentName) async {
  // 디스크 공간이 부족하거나 사용자가 기능을 제거할 때
  await DeferredComponent.uninstallDeferredComponent(
    componentName: componentName,
  );
}
```

> 중요: `DeferredComponent.installDeferredComponent()`는 Dart의 `loadLibrary()`와 별개다. `loadLibrary()`는 Dart VM 수준의 코드 로딩이고, `DeferredComponent`는 Android Dynamic Feature Module 설치를 직접 제어한다. 일반적으로는 `loadLibrary()`만으로 충분하며, Flutter 엔진이 내부적으로 DFM 설치를 처리한다.

---

## 7. 웹에서의 Deferred Loading

Android와 달리 웹에서는 Dynamic Feature Module이 없다. 대신 `dart compile js`가 각 지연 임포트를 별도의 `.part.js` 파일로 분리한다.

```
build/web/
  main.dart.js           ← 베이스 번들
  main.dart.js_1.part.js ← arFeature 로딩 유닛
  main.dart.js_2.part.js ← mapFeature 로딩 유닛
```

사용자가 `loadLibrary()`를 호출하면 브라우저가 해당 `.part.js`를 CDN에서 내려받아 실행한다. CDN 캐시를 활용하면 두 번째 방문부터는 추가 네트워크 비용이 없다.

웹에서 주의할 점: `deferred` 임포트된 라이브러리의 타입을 주 파일에서 타입 힌트로 사용하면 컴파일 오류가 발생한다. 공유 타입은 별도의 중간 라이브러리로 분리해야 한다.

---

## 8. 주의사항 및 실전 팁

### 1) 타입 접근 제한
지연 임포트된 라이브러리의 클래스/함수를 `loadLibrary()` 완료 전에 타입으로만 참조하면 런타임 에러가 발생한다.

```dart
// ❌ 잘못된 사용: loadLibrary() 전에 타입 참조
import 'package:my_app/src/ar/ar_view.dart' deferred as arFeature;

// 이 타입 힌트는 컴파일은 되지만 런타임에 오류 가능성
ArView? _cachedView; // ArView는 arFeature 패키지에 있으므로 위험

// ✅ 올바른 사용: 공유 인터페이스 추출
import 'package:my_app/src/common/base_view.dart'; // 공유 추상 클래스

BaseView? _cachedView; // 공유 패키지의 타입 사용
```

### 2) 디버그 모드에서의 동작 차이
디버그 빌드에서는 모든 Deferred Components가 베이스 모듈에 포함된 채로 실행된다. `loadLibrary()`는 즉시 완료되어 실제 네트워크 다운로드를 시뮬레이션하지 않는다. **반드시 `--profile` 또는 `--release` 모드**로 빌드해 실제 동작을 검증해야 한다.

### 3) 첫 번째 프레임 렌더링 지연
Deferred Component가 로드되는 동안 메인 스레드는 블로킹되지 않는다. 그러나 다운로드 중에는 사용자가 로딩 UI를 봐야 한다. Shimmer 애니메이션이나 Skeleton UI로 기다림을 자연스럽게 처리하자.

### 4) 오프라인 환경 대응
Play Store 연결이 없을 때 `loadLibrary()`는 에러를 던진다. 반드시 try-catch로 감싸고 "오프라인 상태에서는 이 기능을 사용할 수 없습니다" 같은 안내 메시지를 보여줘야 한다.

### 5) 분할 기준 설계
모든 화면을 Deferred Components로 만들면 오히려 사용자 경험이 나빠진다. 아래 기준으로 분할 대상을 선별하라:

- **첫 실행 경로(Critical Path)** 에 없는 기능 (AR, 지도, 고급 편집 등)
- **코드 크기가 큰 단위** (500KB 이상 압축 크기)
- **사용 빈도가 낮은 기능** (DAU 대비 사용률 20% 미만)
- **독립적으로 테스트 가능한 단위** (다른 기능에 대한 의존성이 적은 것)

---

## 9. 빌드 및 검증

```bash
# 1. App Bundle 빌드 (Deferred Components 포함)
flutter build appbundle --release

# 2. 로딩 유닛 매니페스트 확인
cat deferred_components_loading_units.yaml

# 3. bundletool로 로컬 테스트
bundletool build-apks \
  --bundle=build/app/outputs/bundle/release/app-release.aab \
  --output=test.apks \
  --local-testing

bundletool install-apks --apks=test.apks

# 4. 각 모듈 크기 확인
bundletool get-size total --apks=test.apks
```

`--local-testing` 플래그를 사용하면 실제 Play Store 없이 로컬에서 Dynamic Feature Module 다운로드를 시뮬레이션할 수 있다.

---

## 마치며

Flutter Deferred Components는 **큰 앱을 필요한 순간에 필요한 만큼만 내려받는** 현대적인 앱 배포 전략이다. 올바르게 적용하면 최초 설치 크기를 수십 MB 단위로 줄일 수 있고, 그만큼 설치 완료율이 높아진다.

구현 시 가장 중요한 원칙은 세 가지다:
1. `loadLibrary()` 완료 전 심볼 접근 금지
2. 릴리스 빌드에서만 실제 동작 검증
3. 오프라인/에러 상태 반드시 처리

앱의 첫인상은 설치 순간부터 시작된다. Deferred Components로 그 첫인상을 가볍게 만들어 보자.

## 참고 자료
- [Flutter Deferred Components 공식 문서](https://docs.flutter.dev/perf/deferred-components)
- [Dart 언어 라이브러리 - 지연 로딩](https://dart.dev/language/libraries)
- [Flutter 블로그 - 웹 앱 성능 최적화: Tree Shaking과 Deferred Loading](https://flutter.dev/blog/optimizing-performance-in-flutter-web-apps-with-tree-shaking-and-deferred-loading)
- [DeferredComponent API 레퍼런스](https://api.flutter.dev/flutter/services/DeferredComponent-class.html)
