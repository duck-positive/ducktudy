---
layout: post
title: "Flutter Plugin 개발 심화: Federated Plugin 구조와 Pigeon으로 타입 안전한 네이티브 연동 완전 정복"
date: 2026-07-27
categories: [android, flutter]
tags: [flutter, plugin, federated-plugin, pigeon, kotlin, dart, platform-channel, code-generation]
---

Flutter 플러그인을 처음 만들어 봤다면 한 번쯤 이런 경험을 했을 것이다. Dart에서 `invokeMethod("getBatteryLevel")`을 호출했는데 Android 쪽에서 `getBatterylevel`(소문자 l)이라고 오타를 냈고, 그 버그를 찾는 데 한 시간을 날린 경험. Flutter 팀도 이 문제를 알았다. 그래서 나온 것이 **Pigeon**이고, 팀 규모의 플러그인 관리를 위해 고안된 것이 **Federated Plugin** 구조다.

이번 글에서는 두 기술을 결합해 **타입 안전하고 플랫폼별로 독립 배포 가능한** Flutter 네이티브 연동 레이어를 처음부터 끝까지 직접 구현해 본다.

---

## 1. Flutter 플러그인 아키텍처 변천사

Flutter가 처음 나왔을 때 플러그인은 단일 패키지였다. Android, iOS, Web 코드가 한 저장소에 뒤섞였고, iOS 담당자가 버그를 고쳐도 Android 코드와 함께 패키지를 새로 릴리즈해야 했다. 커뮤니티가 Windows 구현을 추가하려면 원본 저장소에 PR을 보내야만 했다.

Flutter 팀은 이 문제를 **세 가지 역할로 패키지를 분리**해 해결했다. 이것이 Federated Plugin이다.

| 패키지 역할 | 내용 | 예시 (공식 camera 플러그인) |
|---|---|---|
| **app-facing package** | 개발자가 직접 `pubspec.yaml`에 추가하는 public API | `camera` |
| **platform interface package** | Dart abstract class로 플랫폼 계약을 정의 | `camera_platform_interface` |
| **platform implementation package** | 플랫폼별 실제 구현 | `camera_android`, `camera_ios` |

이 구조의 핵심은 `platform interface`가 중간 계약 역할을 한다는 점이다. `camera_android`는 `camera_platform_interface`의 abstract class를 `extends`하고, `camera`(app-facing)는 `CameraPlatform.instance`를 통해 런타임에 주입된 구현에 위임한다. Android와 iOS 팀은 서로 독립적으로 패키지 버전을 관리할 수 있다.

---

## 2. raw MethodChannel의 문제점

Pigeon을 모르는 개발자는 이런 코드를 작성한다.

```dart
// Dart 측: 타입 정보가 없다
final result = await _channel.invokeMethod<int>('getBatteryLevel');
```

```kotlin
// Android 측: 문자열 비교 + 강제 캐스팅
override fun onMethodCall(call: MethodCall, result: MethodChannel.Result) {
    if (call.method == "getBatteryLevel") {  // 오타 → 런타임 에러
        val args = call.arguments as Map<String, Any>  // 강제 캐스팅 → 크래시 위험
        val level = args["level"] as Int
        result.success(level)
    }
}
```

문제는 세 가지다.

1. **오타가 컴파일 타임에 잡히지 않는다.** `"getBatteryLevel"` vs `"getBatterylevel"` 차이를 IDE가 경고해 주지 않는다.
2. **양쪽 코드가 동기화됐는지 보장할 수 없다.** Dart에서 `int`를 기대하는데 Kotlin이 `String`을 보내도 빌드가 통과된다.
3. **리팩터링 비용이 크다.** 메서드 이름이나 인자 구조를 바꾸면 양쪽을 수동으로 찾아서 고쳐야 한다.

---

## 3. Pigeon: 인터페이스 정의 → 코드 자동 생성

Pigeon은 Flutter 팀이 공식 제공하는 코드 생성 도구다. **Dart로 작성한 인터페이스 정의 파일 하나**로부터 Android(Kotlin/Java), iOS(Swift/Objective-C), Windows(C++) 코드를 모두 자동으로 생성한다.

### 3.1 설정

```yaml
# pubspec.yaml (battery_android 패키지)
dev_dependencies:
  pigeon: ^27.0.0
```

### 3.2 인터페이스 정의 파일 작성

프로젝트 루트의 `pigeons/` 디렉터리에 인터페이스 파일을 작성한다. 이 파일은 `lib/` 밖에 위치하며, 절대 `import`해서 쓰지 않는다. **오직 코드 생성을 위한 명세 파일**이다.

```dart
// pigeons/battery_api.dart

import 'package:pigeon/pigeon.dart';

@ConfigurePigeon(PigeonOptions(
  dartOut: 'lib/src/battery_api.g.dart',
  kotlinOut:
      'android/src/main/kotlin/com/example/battery/BatteryApi.g.kt',
  kotlinOptions: KotlinOptions(package: 'com.example.battery'),
  swiftOut: 'ios/Classes/BatteryApi.g.swift',
))

// --- 공유 데이터 모델 ---
class BatteryInfo {
  const BatteryInfo({
    required this.level,
    required this.isCharging,
    required this.temperature,
  });

  final int level;           // 0~100
  final bool isCharging;
  final double temperature;  // 섭씨
}

// --- Flutter → 네이티브 방향 (HostApi) ---
// Dart에서 호출하면 Kotlin/Swift 구현이 실행된다
@HostApi()
abstract class BatteryHostApi {
  BatteryInfo getBatteryInfo();

  void startMonitoring();

  // @async: 비동기 메서드임을 Pigeon에 알린다
  @async
  List<BatteryInfo> getBatteryHistory(int lastNHours);
}

// --- 네이티브 → Flutter 방향 (FlutterApi) ---
// Kotlin/Swift에서 호출하면 Dart 콜백이 실행된다
@FlutterApi()
abstract class BatteryFlutterApi {
  void onBatteryLevelChanged(BatteryInfo info);
  void onChargingStateChanged(bool isCharging);
}
```

**코드 생성 실행:**

```bash
dart run pigeon --input pigeons/battery_api.dart
```

이 명령 하나로 Dart, Kotlin, Swift 보일러플레이트가 모두 생성된다. 이제 Kotlin에는 `BatteryHostApi` 인터페이스가, Dart에는 `BatteryHostApi` 클래스가 자동 생성돼 있다.

---

## 4. Android 측 구현 (Kotlin)

생성된 `BatteryApi.g.kt`의 `BatteryHostApi` 인터페이스를 구현한다. 문자열이 단 하나도 없다는 점을 주목하라.

```kotlin
// android/src/main/kotlin/com/example/battery/BatteryPlugin.kt

package com.example.battery

import android.content.Context
import android.content.Intent
import android.content.IntentFilter
import android.os.BatteryManager
import io.flutter.embedding.engine.plugins.FlutterPlugin

class BatteryPlugin : FlutterPlugin, BatteryHostApi {

    private lateinit var context: Context
    private var flutterApi: BatteryFlutterApi? = null

    override fun onAttachedToEngine(binding: FlutterPlugin.FlutterPluginBinding) {
        context = binding.applicationContext
        // 생성된 setUp 함수로 등록. 채널 이름 문자열이 없다!
        BatteryHostApi.setUp(binding.binaryMessenger, this)
        flutterApi = BatteryFlutterApi(binding.binaryMessenger)
    }

    override fun onDetachedFromEngine(binding: FlutterPlugin.FlutterPluginBinding) {
        BatteryHostApi.setUp(binding.binaryMessenger, null)
        flutterApi = null
    }

    // HostApi 구현: Dart에서 getBatteryInfo()를 호출하면 여기가 실행된다
    override fun getBatteryInfo(): BatteryInfo {
        val batteryIntent = context.registerReceiver(
            null,
            IntentFilter(Intent.ACTION_BATTERY_CHANGED)
        ) ?: throw FlutterError(
            "unavailable",
            "배터리 정보를 가져올 수 없습니다",
            null
        )

        val level = batteryIntent.getIntExtra(BatteryManager.EXTRA_LEVEL, -1)
        val scale = batteryIntent.getIntExtra(BatteryManager.EXTRA_SCALE, -1)
        val status = batteryIntent.getIntExtra(BatteryManager.EXTRA_STATUS, -1)
        val temp = batteryIntent.getIntExtra(BatteryManager.EXTRA_TEMPERATURE, 0)

        return BatteryInfo(
            level = if (scale > 0) (level * 100L / scale) else -1L,
            isCharging = status == BatteryManager.BATTERY_STATUS_CHARGING
                      || status == BatteryManager.BATTERY_STATUS_FULL,
            temperature = temp / 10.0,
        )
    }

    // 비동기 HostApi: callback으로 결과를 전달한다
    override fun getBatteryHistory(
        lastNHours: Long,
        callback: (Result<List<BatteryInfo>>) -> Unit
    ) {
        Thread {
            try {
                val history = fetchHistoryFromDb(lastNHours.toInt())
                callback(Result.success(history))
            } catch (e: Exception) {
                callback(Result.failure(e))
            }
        }.start()
    }

    override fun startMonitoring() {
        // BroadcastReceiver로 배터리 변화 감지
        // 변화 발생 시 flutterApi?.onBatteryLevelChanged(info) 호출
    }

    // FlutterApi 역방향 호출: Kotlin에서 Dart 콜백을 트리거한다
    internal fun notifyLevelChanged(info: BatteryInfo) {
        flutterApi?.onBatteryLevelChanged(info) { error ->
            error?.let { e -> println("Flutter callback error: ${e.message}") }
        }
    }

    private fun fetchHistoryFromDb(hours: Int): List<BatteryInfo> {
        // Room 또는 SQLite 쿼리로 이력 반환
        return emptyList()
    }
}
```

---

## 5. Dart 측 사용

생성된 `battery_api.g.dart`를 임포트하면 완전한 타입 안전성이 보장된다.

```dart
// lib/src/battery_repository.dart

import 'battery_api.g.dart';

class BatteryRepository {
  // DI를 위해 생성자로 주입 받는다 (테스트 시 Mock 교체 가능)
  BatteryRepository({BatteryHostApi? api}) : _api = api ?? BatteryHostApi();

  final BatteryHostApi _api;

  Future<BatteryInfo> fetchInfo() async {
    // 반환 타입이 BatteryInfo로 명확히 보장된다
    return _api.getBatteryInfo();
  }

  Future<List<BatteryInfo>> fetchHistory(int hours) async {
    return _api.getBatteryHistory(hours);
  }

  Future<void> startMonitoring({
    required void Function(BatteryInfo) onLevelChanged,
    required void Function(bool) onChargingChanged,
  }) async {
    // 네이티브 → Flutter 콜백 등록
    BatteryFlutterApi.setUp(_CallbackHandler(
      onLevelChanged: onLevelChanged,
      onChargingChanged: onChargingChanged,
    ));
    await _api.startMonitoring();
  }
}

// FlutterApi 구현체 (네이티브로부터 호출됨)
class _CallbackHandler implements BatteryFlutterApi {
  const _CallbackHandler({
    required this.onLevelChanged,
    required this.onChargingChanged,
  });

  final void Function(BatteryInfo) onLevelChanged;
  final void Function(bool) onChargingChanged;

  @override
  void onBatteryLevelChanged(BatteryInfo info) => onLevelChanged(info);

  @override
  void onChargingStateChanged(bool isCharging) => onChargingChanged(isCharging);
}
```

---

## 6. Federated Plugin 디렉터리 구조

위 코드를 Federated Plugin 규약에 맞게 배치하면 다음 구조가 된다.

```
battery/                            # app-facing: 사용자가 추가하는 패키지
├── lib/
│   └── battery.dart               # BatteryRepository를 감싸는 public API
└── pubspec.yaml                   # battery_platform_interface 의존

battery_platform_interface/        # 플랫폼 계약
├── lib/
│   ├── battery_platform_interface.dart  # abstract BatteryPlatform
│   └── src/
│       └── battery_info.dart           # 공유 데이터 모델
└── pubspec.yaml

battery_android/                   # Android 구현
├── android/
│   └── src/main/kotlin/
│       └── com/example/battery/
│           ├── BatteryPlugin.kt        # HostApi 구현
│           └── BatteryApi.g.kt        # Pigeon 생성 코드
├── lib/
│   ├── battery_android.dart           # registerWith + BatteryPlatform 구현
│   └── src/
│       └── battery_api.g.dart         # Pigeon 생성 Dart 코드
└── pigeons/
    └── battery_api.dart               # Pigeon 정의 파일 (유일한 진실의 원천)
```

`battery_android/lib/battery_android.dart`는 `BatteryPlatform`(platform interface의 abstract class)을 `extends`하고, `registerWith`에서 자신을 `BatteryPlatform.instance`로 등록한다. 최종 사용자는 `battery` 패키지만 추가하면 되고, `battery_android`는 `endorsed` 플러그인으로 자동 포함된다.

---

## 7. 주의사항 및 팁

### 7.1 Pigeon이 지원하지 않는 타입

`DateTime`, `Duration`, `Uri` 같은 Dart 내장 타입은 Pigeon이 직접 지원하지 않는다. 반드시 primitive로 변환해서 전달해야 한다.

```dart
// 잘못된 방법 — Pigeon이 생성 에러를 낸다
class Event {
  final DateTime timestamp;  // ❌ 지원 안 됨
}

// 올바른 방법
class Event {
  final int timestampMs;     // ✅ Unix epoch milliseconds
}
```

### 7.2 구조화된 에러 처리

Kotlin에서 `FlutterError`를 사용하면 Dart에서 `PlatformException`으로 구조화된 에러를 받을 수 있다.

```kotlin
// Kotlin: 코드, 메시지, 세부 정보를 담아 던진다
throw FlutterError("permission_denied", "배터리 권한이 없습니다", null)
```

```dart
// Dart: code 필드로 에러를 분기한다
try {
  final info = await _api.getBatteryInfo();
} on PlatformException catch (e) {
  if (e.code == 'permission_denied') {
    // 권한 요청 다이얼로그 표시
  }
}
```

### 7.3 코드 생성 파일을 CI로 검증

인터페이스를 변경하고 `dart run pigeon`을 실행하지 않은 채 PR을 올리는 실수를 방지하려면 CI에서 이를 체크한다.

```yaml
# .github/workflows/check_generated_code.yml
- name: Regenerate Pigeon files
  run: dart run pigeon --input pigeons/battery_api.dart
- name: Fail if generated files are out of date
  run: git diff --exit-code lib/src/battery_api.g.dart
```

### 7.4 Mock을 활용한 단위 테스트

Pigeon이 생성한 클래스는 mockito로 쉽게 모킹할 수 있다.

```dart
// test/battery_repository_test.dart
@GenerateMocks([BatteryHostApi])
void main() {
  late MockBatteryHostApi mockApi;

  setUp(() {
    mockApi = MockBatteryHostApi();
  });

  test('정상 배터리 정보를 파싱한다', () async {
    when(mockApi.getBatteryInfo()).thenAnswer(
      (_) async => BatteryInfo(level: 85, isCharging: true, temperature: 28.5),
    );

    final repo = BatteryRepository(api: mockApi);
    final info = await repo.fetchInfo();

    expect(info.level, 85);
    expect(info.isCharging, isTrue);
    expect(info.temperature, closeTo(28.5, 0.01));
  });
}
```

### 7.5 양방향 통신의 생명 주기

`FlutterApi`를 등록하는 `BatteryFlutterApi.setUp`은 `onAttachedToEngine`에서 주입된 `binaryMessenger`와 짝을 이뤄야 한다. `onDetachedFromEngine` 시점에 `BatteryHostApi.setUp(messenger, null)`로 반드시 해제해야 메모리 누수를 막을 수 있다.

---

## 마치며

raw `MethodChannel`은 빠른 프로토타이핑에는 충분하지만, 팀 단위 플러그인 개발에는 적합하지 않다. **Pigeon**은 인터페이스를 단일 Dart 파일로 정의하고 나머지 보일러플레이트를 전부 생성해 줌으로써, 플랫폼 불일치로 인한 런타임 크래시 가능성을 컴파일 타임에 차단한다. **Federated Plugin** 구조는 여기에 플랫폼별 독립 버전 관리와 커뮤니티 기여 모델을 더한다.

새 Flutter 플러그인을 시작한다면, 처음부터 Federated 구조와 Pigeon을 채택하라. 초기 설정 비용은 있지만, 유지보수 단계에서 되돌아오는 이자는 훨씬 크다.

## 참고 자료
- [pigeon \| Dart package (pub.dev)](https://pub.dev/packages/pigeon)
- [pigeon API 문서 (pub.dev)](https://pub.dev/documentation/pigeon/latest/)
- [flutter/packages - pigeon 소스 (GitHub)](https://github.com/flutter/packages/tree/main/packages/pigeon)
