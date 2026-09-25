---
layout: post
title: "Flutter Build Flavors와 --dart-define 완전 정복: dev·staging·prod 멀티 환경 설정 관리"
date: 2026-09-25
categories: [android, flutter]
tags: [flutter, dart, build-flavors, dart-define, environment, configuration, firebase, ci-cd]
---

모든 프로덕션 앱은 최소한 두 가지 환경을 가진다. 개발 서버를 바라보는 **dev** 빌드와 실제 사용자에게 배포되는 **production** 빌드다. 여기에 QA팀용 **staging**, 성능 측정용 **profiling** 환경까지 더해지면 플레이버 분리는 선택이 아닌 필수다. 이 글에서는 Flutter의 **Build Flavors**와 **`--dart-define` / `--dart-define-from-file`** 을 조합해 런타임 코드 분기 없이 컴파일 타임에 환경별 설정을 완전히 분리하는 방법을 깊이 있게 다룬다.

---

## 1. 개념 설명

### Build Flavors

Android의 `productFlavors`와 iOS의 Xcode Scheme을 Flutter 레이어에서 통합한 개념이다. 플레이버를 지정하면 하나의 코드베이스에서 **서로 다른 앱 ID(패키지명), 아이콘, 서명 설정**을 가진 APK/IPA를 독립적으로 빌드할 수 있다.

```
flutter run  --flavor dev  -t lib/main_dev.dart
flutter build apk --flavor prod -t lib/main_prod.dart
```

### --dart-define

`--dart-define=KEY=VALUE` 플래그는 Dart 컴파일러에게 **컴파일 타임 상수**를 주입한다. Dart 코드에서는 `const String.fromEnvironment('KEY')` 또는 `const bool.fromEnvironment('KEY')` 로 읽는다. **런타임 조건 분기 없이** 빌드 시점에 값이 확정되므로 트리 셰이킹(tree shaking)으로 불필요한 코드가 제거된다.

### --dart-define-from-file (Flutter 3.7+)

여러 개의 `--dart-define` 플래그를 입력하는 불편을 해소하기 위해 Flutter 3.7에서 도입됐다. JSON 파일 하나를 지정하면 파일 내 모든 키-값 쌍이 일괄 주입된다.

```
flutter run --dart-define-from-file=envs/dev.json
```

---

## 2. 왜 필요한가

### 2-1. 환경별 엔드포인트 분리

hardcode된 URL은 즉각적인 기술 부채다. `kDebugMode` 체크로 분기하면 프로덕션 빌드에 dev URL이 심어질 위험이 항상 존재한다.

### 2-2. Firebase 프로젝트 분리

Google Services 파일(`google-services.json`, `GoogleService-Info.plist`)을 플레이버별로 따로 두어 dev Firebase 프로젝트와 prod Firebase 프로젝트를 완전히 격리할 수 있다.

### 2-3. CI/CD 파이프라인 단순화

GitHub Actions나 Fastlane에서 플레이버와 `--dart-define-from-file` 파일 경로만 파라미터로 바꾸면 동일한 파이프라인으로 모든 환경을 빌드할 수 있다.

### 2-4. 비밀 정보 보호

API 키나 엔드포인트를 소스코드에 박아두면 Git 이력에 영구 기록된다. `.env` 파일을 `.gitignore`에 추가하고 `--dart-define-from-file`로 주입하면 레포지토리를 깔끔하게 유지할 수 있다.

---

## 3. 실제 구현 예제

### 3-1. 환경 파일 구조 설계

프로젝트 루트에 `envs/` 디렉토리를 만들고 각 환경별 JSON 파일을 생성한다. `.gitignore`에 `envs/`를 추가해 커밋되지 않게 한다. CI 서버에는 Secrets로 관리한다.

```
my_app/
├── envs/
│   ├── dev.json        # .gitignore에 포함
│   ├── staging.json
│   └── prod.json
├── lib/
│   ├── main_dev.dart
│   ├── main_staging.dart
│   ├── main_prod.dart
│   └── config/
│       └── app_config.dart
└── android/
    └── app/
        └── build.gradle.kts
```

`envs/dev.json`:
```json
{
  "APP_ENV": "dev",
  "BASE_URL": "https://api-dev.example.com",
  "API_KEY": "dev_api_key_xxxx",
  "ENABLE_LOGGING": "true",
  "SENTRY_DSN": ""
}
```

`envs/prod.json`:
```json
{
  "APP_ENV": "prod",
  "BASE_URL": "https://api.example.com",
  "API_KEY": "prod_api_key_yyyy",
  "ENABLE_LOGGING": "false",
  "SENTRY_DSN": "https://abc@o123.ingest.sentry.io/456"
}
```

### 3-2. AppConfig — 타입 안전한 컴파일 타임 설정 클래스

```dart
// lib/config/app_config.dart

enum AppEnv { dev, staging, prod }

/// 모든 값은 컴파일 타임 상수다.
/// --dart-define-from-file로 주입되지 않으면 defaultValue가 사용된다.
class AppConfig {
  // 생성 불가 유틸리티 클래스
  const AppConfig._();

  static const _envName = String.fromEnvironment('APP_ENV', defaultValue: 'dev');

  static AppEnv get env => switch (_envName) {
        'prod' => AppEnv.prod,
        'staging' => AppEnv.staging,
        _ => AppEnv.dev,
      };

  static const baseUrl = String.fromEnvironment(
    'BASE_URL',
    defaultValue: 'https://api-dev.example.com',
  );

  static const apiKey = String.fromEnvironment(
    'API_KEY',
    defaultValue: '',
  );

  static const enableLogging = bool.fromEnvironment(
    'ENABLE_LOGGING',
    defaultValue: true,
  );

  static const sentryDsn = String.fromEnvironment('SENTRY_DSN', defaultValue: '');

  static bool get isProd => env == AppEnv.prod;
  static bool get isDev => env == AppEnv.dev;

  @override
  String toString() =>
      'AppConfig(env: $env, baseUrl: $baseUrl, logging: $enableLogging)';
}
```

`const` 키워드 덕분에 `AppConfig.enableLogging`이 `false`로 확정된 prod 빌드에서는 로깅 코드 전체가 트리 셰이킹으로 제거된다.

### 3-3. 멀티 엔트리포인트

```dart
// lib/main_dev.dart
import 'package:flutter/material.dart';
import 'app.dart';
import 'config/app_config.dart';

void main() {
  // 빌드 타임에 AppConfig.isDev == true 가 보장된다.
  assert(AppConfig.isDev, 'main_dev.dart는 dev 플레이버 전용입니다.');
  runApp(const MyApp());
}
```

```dart
// lib/main_prod.dart
import 'package:flutter/material.dart';
import 'app.dart';
import 'config/app_config.dart';
import 'package:sentry_flutter/sentry_flutter.dart';

Future<void> main() async {
  await SentryFlutter.init(
    (options) {
      options.dsn = AppConfig.sentryDsn;
      options.environment = AppConfig._envName;
    },
    appRunner: () => runApp(const MyApp()),
  );
}
```

### 3-4. Android productFlavors 설정

```kotlin
// android/app/build.gradle.kts

android {
    flavorDimensions += "environment"

    productFlavors {
        create("dev") {
            dimension = "environment"
            applicationIdSuffix = ".dev"
            versionNameSuffix = "-dev"
            resValue("string", "app_name", "MyApp Dev")
        }
        create("staging") {
            dimension = "environment"
            applicationIdSuffix = ".staging"
            versionNameSuffix = "-staging"
            resValue("string", "app_name", "MyApp Staging")
        }
        create("prod") {
            dimension = "environment"
            // suffix 없음 — 프로덕션 앱 ID 그대로 유지
            resValue("string", "app_name", "MyApp")
        }
    }
}
```

### 3-5. Firebase 퍼-플레이버 google-services.json

Android는 `src/{flavor}/google-services.json` 경로를 자동으로 인식한다.

```
android/app/src/
├── dev/
│   └── google-services.json    ← dev Firebase 프로젝트
├── staging/
│   └── google-services.json    ← staging Firebase 프로젝트
└── prod/
    └── google-services.json    ← prod Firebase 프로젝트
```

플레이버 빌드 시 Gradle 플러그인이 올바른 파일을 자동으로 선택한다. iOS는 `ios/Runner/` 아래 Xcode Scheme별 Run Script로 `GoogleService-Info.plist`를 복사하는 방식을 사용한다.

### 3-6. VS Code launch.json 으로 원클릭 실행

```json
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Flutter (dev)",
      "request": "launch",
      "type": "dart",
      "program": "lib/main_dev.dart",
      "args": [
        "--flavor", "dev",
        "--dart-define-from-file", "envs/dev.json"
      ]
    },
    {
      "name": "Flutter (staging)",
      "request": "launch",
      "type": "dart",
      "program": "lib/main_staging.dart",
      "args": [
        "--flavor", "staging",
        "--dart-define-from-file", "envs/staging.json"
      ]
    },
    {
      "name": "Flutter (prod)",
      "request": "launch",
      "type": "dart",
      "program": "lib/main_prod.dart",
      "args": [
        "--flavor", "prod",
        "--dart-define-from-file", "envs/prod.json"
      ]
    }
  ]
}
```

---

## 4. 주의사항 및 팁

### 4-1. const 를 빠뜨리지 마라

`String.fromEnvironment`는 반드시 `const` 컨텍스트에서 호출해야 컴파일 타임 상수로 동작한다. `const`가 없으면 런타임에 빈 문자열을 반환하고 트리 셰이킹도 적용되지 않는다.

```dart
// ❌ 런타임에 항상 defaultValue 반환
final url = String.fromEnvironment('BASE_URL');

// ✅ 컴파일 타임 상수로 확정
const url = String.fromEnvironment('BASE_URL', defaultValue: 'https://fallback.dev');
```

### 4-2. envs/ 파일 절대 커밋 금지

API 키가 담긴 JSON 파일이 Git에 올라가면 히스토리에 영구 기록된다. `.gitignore`에 `envs/` 를 추가하고, CI에서는 GitHub Actions `${{ secrets.PROD_ENV_JSON }}`를 파일로 저장해서 사용한다.

```yaml
# .github/workflows/build.yml (예시)
- name: Write env file
  run: echo '${{ secrets.PROD_ENV_JSON }}' > envs/prod.json

- name: Build prod
  run: flutter build apk --flavor prod -t lib/main_prod.dart --dart-define-from-file=envs/prod.json
```

### 4-3. `--dart-define` 값은 모두 String

JSON의 `true`, `false`, 숫자 값도 `--dart-define` 으로 주입되면 문자열로 처리된다. `bool.fromEnvironment`는 정확히 `"true"` 문자열에만 `true`를 반환하므로 JSON에 `"ENABLE_LOGGING": "true"` 처럼 따옴표로 감싼다.

### 4-4. `envied` 패키지로 타입 안전성 강화

`envied` 패키지는 `build_runner`와 함께 `.env` 파일을 Dart 클래스로 코드 생성해준다. 값 오타를 컴파일 에러로 잡고, 선택적 난독화 기능으로 APK 바이너리 내 평문 키 노출을 줄일 수 있다.

```dart
// lib/config/env.dart
import 'package:envied/envied.dart';

part 'env.g.dart';

@Envied(path: 'envs/.env.prod', obfuscate: true)
abstract class Env {
  @EnviedField()
  static const String apiKey = _Env.apiKey;

  @EnviedField()
  static const String baseUrl = _Env.baseUrl;
}
```

### 4-5. Flavor 이름과 dart-define을 같이 사용하는 패턴

Flavor 자체는 네이티브 수준(앱 ID, 서명, Google Services)을 분리하고, `--dart-define-from-file`은 Dart 레이어 설정을 담당하는 명확한 역할 분담 패턴이 권장된다. 동일 플레이버라도 `.env` 파일만 교체해 다른 API 서버를 바라보게 할 수 있어 QA·로드테스트 시나리오에 유용하다.

---

## 요약

| 접근 방식 | 적합한 경우 | 한계 |
|---|---|---|
| `--dart-define` | 소수의 단순 키-값 | 키가 많아지면 CLI 명령이 길어짐 |
| `--dart-define-from-file` | JSON 기반 다수의 설정 (Flutter 3.7+) | .env 형식 미지원 (JSON만) |
| Build Flavors | 앱 ID·아이콘·Firebase 프로젝트 분리 | 네이티브 설정 파일 관리 필요 |
| `envied` 패키지 | 타입 안전성 + 난독화가 필요한 경우 | 코드 생성 단계(build_runner) 추가 |

Build Flavors로 네이티브 레이어를 나누고, `--dart-define-from-file`로 Dart 레이어 설정을 주입하며, `AppConfig` 클래스로 타입 안전하게 접근하는 세 겹 구조가 현재 Flutter 커뮤니티의 베스트 프랙티스다. CI/CD 파이프라인에서 환경 JSON 파일을 Secret으로 관리하면 보안과 자동화 두 마리 토끼를 모두 잡을 수 있다.

## 참고 자료
- [Flutter 공식 문서 — Set up Flutter flavors for Android](https://docs.flutter.dev/deployment/flavors)
- [Dart 공식 문서 — Configuring apps with compilation environment declarations](https://dart.dev/libraries/core/environment-declarations)
- [pub.dev — envied 패키지](https://pub.dev/packages/envied)
- [pub.dev — flutter_flavor 패키지](https://pub.dev/packages/flutter_flavor)
