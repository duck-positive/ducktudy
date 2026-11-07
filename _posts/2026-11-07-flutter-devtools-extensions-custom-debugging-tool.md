---
layout: post
title: "Flutter DevTools Extensions 심화: 커스텀 디버깅 패널을 패키지에 내장하는 완전 가이드"
date: 2026-11-07
categories: [android, flutter]
tags: [flutter, devtools, debugging, dart, extension, widget-inspector, pub-dev]
---

Flutter 앱을 개발하면서 "우리 패키지 사용자가 DevTools 안에서 바로 상태를 볼 수 있으면 얼마나 좋을까?" 하는 생각을 해본 적이 있다면, **DevTools Extensions**이 그 답입니다. Flutter 3.17 / Dart 3.2부터 정식 지원되는 이 기능을 사용하면, 여러분의 pub.dev 패키지 안에 커스텀 DevTools 탭을 내장할 수 있습니다. 이 글에서는 DevTools Extensions의 개념부터 실제 패키지에 적용하는 전 과정을 코드와 함께 설명합니다.

---

## DevTools Extensions란?

Flutter DevTools는 여러분이 `flutter run`을 실행할 때 연결되는 웹 기반 디버깅 플랫폼입니다. Widget Inspector, Performance, Memory, Network 탭이 기본으로 제공되지만, 서드파티 패키지는 자체 탭을 추가할 수 없었습니다.

DevTools Extensions는 이 제약을 해소합니다. 패키지에 `extension/devtools/` 폴더를 추가하고, 그 안에 Flutter 웹 앱을 빌드해 넣으면 DevTools가 자동으로 감지해 새 탭으로 등록합니다. 사용자는 별도로 플러그인을 설치할 필요 없이, 해당 패키지를 `pubspec.yaml`에 추가하는 것만으로 커스텀 패널을 쓸 수 있게 됩니다.

```
내 패키지
├── lib/
│   └── my_package.dart
├── extension/
│   └── devtools/
│       ├── config.yaml      ← DevTools 메타데이터
│       └── build/           ← 빌드된 Flutter Web 앱
└── pubspec.yaml
```

---

## 왜 DevTools Extension이 필요한가?

### 1. 패키지 사용자 경험 향상
`provider`, `riverpod`, `bloc` 같은 상태 관리 패키지를 쓸 때, 현재 상태가 어떻게 변화하는지 DevTools 안에서 직접 보고 싶어집니다. 로그를 `print`로 뿌리는 것보다 시각화된 패널이 훨씬 직관적입니다.

### 2. 커스텀 로직 가시화
네트워크 캐시 상태, 로컬 DB 쿼리 결과, A/B 테스트 플래그 같은 도메인 특화 정보를 DevTools 탭에서 실시간으로 조회할 수 있으면 디버깅 효율이 크게 오릅니다.

### 3. pub.dev 패키지 경쟁력 강화
동일한 기능을 제공하는 패키지라도 DevTools Extension이 있는 쪽이 훨씬 개발자 친화적으로 평가받습니다. 2024년 이후 여러 인기 패키지(drift, provider 등)가 Extensions를 도입한 이유이기도 합니다.

---

## 프로젝트 구조 설정

### 요구 사항

- Flutter SDK 3.17.0 이상
- Dart SDK 3.2.0 이상
- `devtools_extensions` 패키지 (pub.dev)
- `devtools_app_shared` 패키지 (pub.dev)

### 1단계: Extension 앱 생성

기존 패키지(`my_cache_package`)에 Extension을 추가한다고 가정합니다.

```bash
# 패키지 루트에서 실행
mkdir -p extension
cd extension
flutter create --template app --platforms web devtools_app
```

`extension/devtools_app/pubspec.yaml`에 의존성을 추가합니다:

```yaml
dependencies:
  flutter:
    sdk: flutter
  devtools_extensions: ^0.0.19
  devtools_app_shared: ^0.0.9
```

### 2단계: config.yaml 작성

`extension/devtools/config.yaml`을 만들어 DevTools에 패널 정보를 알립니다:

```yaml
name: MyCachePanel
issueTracker: https://github.com/yourname/my_cache_package/issues
version: 0.1.0
materialIconCodePoint: '0xe1d8'  # cache 아이콘
```

---

## 실제 구현 예제 1: 기본 Extension 앱 뼈대

`extension/devtools_app/lib/main.dart`에 아래처럼 작성합니다. `DevToolsExtension` 위젯으로 루트를 감싸는 것이 핵심입니다.

```dart
import 'package:devtools_app_shared/ui.dart';
import 'package:devtools_extensions/devtools_extensions.dart';
import 'package:flutter/material.dart';

void main() {
  runApp(const MyCacheExtensionApp());
}

class MyCacheExtensionApp extends StatelessWidget {
  const MyCacheExtensionApp({super.key});

  @override
  Widget build(BuildContext context) {
    // DevToolsExtension이 DevTools와의 통신 채널을 초기화합니다.
    return const DevToolsExtension(
      child: MyCachePanel(),
    );
  }
}

class MyCachePanel extends StatefulWidget {
  const MyCachePanel({super.key});

  @override
  State<MyCachePanel> createState() => _MyCachePanelState();
}

class _MyCachePanelState extends State<MyCachePanel> {
  List<String> _cacheKeys = [];
  bool _isLoading = false;

  Future<void> _fetchCacheState() async {
    setState(() => _isLoading = true);

    // serviceManager를 통해 디버그 대상 앱의 VM Service에 접근합니다.
    final response = await serviceManager.service!.callServiceExtension(
      'ext.myCachePackage.getCacheKeys',
      isolateId: serviceManager.isolateManager.selectedIsolate.value!.id!,
    );

    final keys = List<String>.from(
      (response.json!['keys'] as List<dynamic>).cast<String>(),
    );

    if (mounted) {
      setState(() {
        _cacheKeys = keys;
        _isLoading = false;
      });
    }
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        // DevTools 공통 헤더 스타일
        AreaPaneHeader(
          title: const Text('Cache Inspector'),
          actions: [
            IconButton(
              icon: const Icon(Icons.refresh),
              tooltip: '새로고침',
              onPressed: _fetchCacheState,
            ),
          ],
        ),
        Expanded(
          child: _isLoading
              ? const CenteredCircularProgressIndicator()
              : _cacheKeys.isEmpty
                  ? const Center(child: Text('캐시가 비어있습니다'))
                  : ListView.builder(
                      itemCount: _cacheKeys.length,
                      itemBuilder: (_, index) => ListTile(
                        leading: const Icon(Icons.storage),
                        title: Text(_cacheKeys[index]),
                      ),
                    ),
        ),
      ],
    );
  }
}
```

**핵심 포인트:**
- `DevToolsExtension` 위젯이 DevTools 프레임과의 iFrame 통신을 추상화합니다.
- `serviceManager`는 `devtools_app_shared`가 제공하는 글로벌 싱글톤으로, 연결된 VM Service에 접근합니다.
- `AreaPaneHeader`, `CenteredCircularProgressIndicator` 같은 DevTools 공통 위젯을 쓰면 네이티브 DevTools 탭과 시각적으로 일관됩니다.

---

## 실제 구현 예제 2: 패키지 측 Service Extension 등록

DevTools Extension이 데이터를 가져가려면, 대상 앱(패키지를 사용하는 앱)이 VM Service Extension을 등록해야 합니다. `my_cache_package`의 `lib/my_cache_package.dart`에 아래를 추가합니다:

```dart
import 'dart:developer' as developer;
import 'package:flutter/foundation.dart';

class MyCacheManager {
  static final MyCacheManager instance = MyCacheManager._();
  MyCacheManager._() {
    // debug 모드에서만 Service Extension 등록
    if (kDebugMode) {
      _registerServiceExtensions();
    }
  }

  final Map<String, dynamic> _cache = {};

  void put(String key, dynamic value) {
    _cache[key] = value;
  }

  dynamic get(String key) => _cache[key];

  void _registerServiceExtensions() {
    // 'ext.myCachePackage.getCacheKeys' 를 DevTools에서 호출 가능하게 등록
    developer.registerExtension(
      'ext.myCachePackage.getCacheKeys',
      (method, parameters) async {
        final keys = _cache.keys.toList();
        return developer.ServiceExtensionResponse.result(
          '{"keys": ${keys.map((k) => '"$k"').toList()}}',
        );
      },
    );

    // 특정 키의 값을 반환하는 Extension
    developer.registerExtension(
      'ext.myCachePackage.getValue',
      (method, parameters) async {
        final key = parameters['key'] ?? '';
        final value = _cache[key];
        final jsonValue = value == null ? 'null' : '"$value"';
        return developer.ServiceExtensionResponse.result(
          '{"key": "$key", "value": $jsonValue}',
        );
      },
    );

    // 캐시 초기화 Extension
    developer.registerExtension(
      'ext.myCachePackage.clearCache',
      (method, parameters) async {
        _cache.clear();
        return developer.ServiceExtensionResponse.result('{"success": true}');
      },
    );
  }
}
```

**등록 흐름:**

```
앱 시작
  → MyCacheManager 초기화
    → developer.registerExtension() 호출
      → VM Service에 'ext.myCachePackage.*' 등록
        → DevTools Extension에서 callServiceExtension()으로 호출 가능
```

---

## 개발 중 시뮬레이션 환경으로 테스트

실제 DevTools 없이 Extension UI를 개발할 때는 `--dart-define` 플래그를 사용합니다:

```bash
cd extension/devtools_app
flutter run -d chrome \
  --dart-define=use_simulated_environment=true
```

`use_simulated_environment=true`를 설정하면 `DevToolsExtension` 위젯이 모의(Mock) `serviceManager`를 사용해, 별도의 Flutter 앱 없이도 UI를 개발하고 핫 리스타트를 활용할 수 있습니다.

---

## 빌드 및 패키지 배포

Extension 개발이 완료되면, 빌드 결과물을 패키지의 `extension/devtools/build/` 폴더에 복사합니다. `devtools_extensions` 패키지가 제공하는 스크립트를 활용하면 간편합니다:

```bash
cd extension/devtools_app
dart run devtools_extensions build_and_copy \
  --source=. \
  --dest=../../extension/devtools
```

검증:

```bash
dart run devtools_extensions validate \
  --package=../../   # 패키지 루트 경로
```

배포 전 최종 구조:

```
my_cache_package/
├── extension/
│   └── devtools/
│       ├── config.yaml
│       └── build/
│           ├── main.dart.js
│           ├── index.html
│           └── ...
└── pubspec.yaml
```

`pub publish`를 실행하면 `extension/devtools/` 디렉터리가 패키지와 함께 배포되고, 사용자는 DevTools를 열었을 때 자동으로 커스텀 탭을 볼 수 있습니다.

---

## 주의사항 및 팁

### 1. kDebugMode 조건 필수
`developer.registerExtension`은 반드시 `kDebugMode` 블록 안에서만 호출하세요. Release 빌드에서 VM Service Extension이 활성화되면 성능에 영향을 주고 보안 문제가 발생할 수 있습니다.

### 2. 버전 호환성 관리
`devtools_extensions`와 `devtools_app_shared`는 DevTools 자체와 버전이 맞아야 합니다. pub.dev에서 `devtools_extensions`의 최신 버전을 확인하고, 의존성을 `^0.0.19` 처럼 범위로 지정하세요. 너무 오래된 버전은 최신 DevTools에서 로드되지 않을 수 있습니다.

### 3. JSON 직렬화는 단순하게
VM Service Extension의 응답은 순수 JSON 문자열이어야 합니다. `jsonEncode`를 활용하고, 중첩 객체나 DateTime 타입은 직렬화 가능한 형태로 변환하세요:

```dart
import 'dart:convert';

developer.registerExtension('ext.pkg.getStats', (_, __) async {
  final data = {
    'hitCount': _hitCount,
    'missCount': _missCount,
    'lastUpdated': DateTime.now().toIso8601String(),
  };
  return developer.ServiceExtensionResponse.result(jsonEncode(data));
});
```

### 4. `devtools_options.yaml`로 사용자 설정 허용
패키지를 사용하는 앱의 루트에 `devtools_options.yaml`을 두면, 어떤 Extension을 활성화/비활성화할지 팀 단위로 관리할 수 있습니다:

```yaml
extensions:
  - my_cache_package: true
  - another_package: false
```

### 5. 프로덕션 앱에서의 영향 없음
Extension 자체(빌드된 웹 앱)는 `extension/devtools/build/` 안에만 있고, Dart 컴파일 대상이 아닙니다. 앱 번들 크기나 런타임에 영향을 주지 않습니다.

---

## 마치며

DevTools Extensions는 단순한 디버깅 도구를 넘어, 패키지 생태계의 개발자 경험(DX)을 혁신하는 기능입니다. 여러분의 패키지가 복잡한 내부 상태를 가질수록, DevTools Extension으로 제공하는 가시성은 사용자에게 더 큰 가치를 줍니다. 이 글에서 살펴본 뼈대 코드를 토대로, 여러분만의 커스텀 디버깅 패널을 만들어보세요.

## 참고 자료
- [Flutter DevTools Extensions 공식 문서](https://docs.flutter.dev/tools/devtools/extensions)
- [Flutter 공식 블로그: Dart & Flutter DevTools Extensions](https://flutter.dev/blog/dart-flutter-devtools-extensions)
- [devtools_extensions 패키지 (pub.dev)](https://pub.dev/packages/devtools_extensions)
- [devtools_app_shared 패키지 (pub.dev)](https://pub.dev/packages/devtools_app_shared)
