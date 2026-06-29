---
layout: post
title: "Flutter Dart FFI 심화: C/C++ 네이티브 라이브러리를 Dart에서 직접 호출하는 기술"
date: 2026-06-29
categories: [android, flutter]
tags: [flutter, dart, ffi, native, c, cpp, performance, interop]
---

플러터(Flutter) 앱을 개발하다 보면 Dart만으로는 처리하기 어려운 영역을 만날 때가 있습니다. 고성능 수치 연산, 이미지/오디오 처리, 암호화 알고리즘, 기존에 C/C++로 만들어진 레거시 라이브러리 통합 등이 대표적인 사례입니다. 이때 우리에게는 두 가지 선택지가 있습니다: Platform Channel을 통해 네이티브 코드를 간접 호출하거나, `dart:ffi`를 사용해 Dart에서 C 함수를 **직접** 호출하는 것입니다.

이번 포스트에서는 `dart:ffi`(Foreign Function Interface)의 동작 원리부터 실제 프로젝트에 적용하는 방법까지 심층적으로 다룹니다.

---

## 1. FFI란 무엇인가?

**FFI(Foreign Function Interface)**는 한 프로그래밍 언어가 다른 언어로 작성된 함수나 라이브러리를 호출할 수 있도록 해주는 인터페이스입니다. Dart의 `dart:ffi`는 Dart Native 플랫폼(모바일, 데스크톱, CLI)에서 C 언어로 작성된 공유 라이브러리(`.so`, `.dylib`, `.dll`)를 직접 로드하고 그 안의 함수를 호출할 수 있게 해줍니다.

### Platform Channel과의 차이점

| 항목 | Platform Channel | Dart FFI |
|---|---|---|
| 통신 방식 | 메시지 직렬화/역직렬화 | 메모리 직접 접근 |
| 성능 오버헤드 | 비교적 높음 (IPC 비용) | 매우 낮음 (함수 포인터 호출) |
| 타입 시스템 | 동적 (Map 기반) | 정적 (C 타입 1:1 매핑) |
| 플랫폼 의존성 | OS별 코드 필요 | C ABI 준수 라이브러리면 공통 |
| 사용 난이도 | 쉬움 | 높음 (메모리 관리 필요) |

Platform Channel은 플랫폼 고유 API(카메라, GPS 등)에 접근할 때 적합하고, FFI는 순수 계산 로직이나 크로스플랫폼 C 라이브러리를 통합할 때 유리합니다.

---

## 2. 왜 Dart FFI가 필요한가?

### 성능이 중요한 연산

Dart는 JIT/AOT 컴파일을 지원하지만, 저수준 비트 연산, SIMD 최적화, 메모리 레이아웃 제어가 필요한 작업에서는 C/C++에 비해 여전히 느릴 수 있습니다. 암호화 라이브러리(OpenSSL), 이미지 코덱(libjpeg-turbo), 머신러닝 추론(NNAPI, TFLite C API) 같은 경우가 대표적입니다.

### 기존 C 라이브러리 재사용

수십 년간 검증된 C 라이브러리를 굳이 Dart로 재구현할 필요가 없습니다. `libsqlite3`, `libz`, `libssl` 같은 라이브러리는 이미 대부분의 플랫폼에 내장되어 있거나 쉽게 번들링할 수 있습니다.

### 단일 바이너리 공유

FFI를 사용하면 하나의 C 공유 라이브러리를 Flutter(모바일/데스크톱)와 Dart CLI 앱이 공통으로 사용할 수 있습니다. Platform Channel은 iOS/Android 각각의 네이티브 코드가 필요하지만, FFI는 C ABI만 맞으면 됩니다.

---

## 3. dart:ffi 핵심 타입 시스템

FFI를 제대로 사용하려면 C 타입과 Dart 타입 사이의 매핑을 이해해야 합니다.

### 기본 숫자 타입 매핑

| C 타입 | dart:ffi 타입 | Dart 값 타입 |
|---|---|---|
| `int32_t` / `int` | `Int32` | `int` |
| `int64_t` / `long long` | `Int64` | `int` |
| `uint8_t` | `Uint8` | `int` |
| `float` | `Float` | `double` |
| `double` | `Double` | `double` |
| `void` | `Void` | `void` |
| `char*` | `Pointer<Utf8>` | `String` (변환 필요) |
| `struct*` | `Pointer<MyStruct>` | - |

### Pointer와 NativeFunction

```dart
import 'dart:ffi';
import 'package:ffi/ffi.dart';

// C 함수 시그니처 정의
// int add(int a, int b)
typedef AddFuncNative = Int32 Function(Int32 a, Int32 b);
typedef AddFunc = int Function(int a, int b);

void main() {
  // 공유 라이브러리 로드
  final lib = DynamicLibrary.open('libmymath.so');

  // 함수 심볼 룩업 후 Dart 함수로 변환
  final add = lib
      .lookup<NativeFunction<AddFuncNative>>('add')
      .asFunction<AddFunc>();

  print(add(3, 7)); // 10
}
```

`NativeFunction<AddFuncNative>`는 C 함수 포인터 타입을 나타내고, `.asFunction<AddFunc>()`는 이를 Dart callable로 변환합니다. 두 개의 typedef를 항상 쌍으로 정의하는 패턴이 관례입니다.

---

## 4. 실제 구현 예제 1: 고성능 수치 연산 통합

### C 라이브러리 작성 (native/math_ops.c)

```c
#include <stdint.h>
#include <math.h>

// 배열의 합계 계산 (SIMD 최적화 가능 구조)
int64_t sum_array(const int32_t* data, int32_t length) {
    int64_t result = 0;
    for (int32_t i = 0; i < length; i++) {
        result += data[i];
    }
    return result;
}

// 표준편차 계산
double std_deviation(const double* data, int32_t length) {
    if (length == 0) return 0.0;
    double mean = 0.0;
    for (int32_t i = 0; i < length; i++) {
        mean += data[i];
    }
    mean /= length;

    double variance = 0.0;
    for (int32_t i = 0; i < length; i++) {
        double diff = data[i] - mean;
        variance += diff * diff;
    }
    return sqrt(variance / length);
}
```

### Dart 래퍼 클래스 작성

```dart
import 'dart:ffi';
import 'dart:io';
import 'package:ffi/ffi.dart';

// Native 함수 시그니처
typedef SumArrayNative = Int64 Function(Pointer<Int32> data, Int32 length);
typedef SumArray = int Function(Pointer<Int32> data, int length);

typedef StdDeviationNative = Double Function(Pointer<Double> data, Int32 length);
typedef StdDeviation = double Function(Pointer<Double> data, int length);

class NativeMath {
  late final DynamicLibrary _lib;
  late final SumArray _sumArray;
  late final StdDeviation _stdDeviation;

  NativeMath() {
    // 플랫폼별 라이브러리 경로 처리
    final libPath = _getLibPath();
    _lib = DynamicLibrary.open(libPath);

    _sumArray = _lib
        .lookup<NativeFunction<SumArrayNative>>('sum_array')
        .asFunction<SumArray>();

    _stdDeviation = _lib
        .lookup<NativeFunction<StdDeviationNative>>('std_deviation')
        .asFunction<StdDeviation>();
  }

  String _getLibPath() {
    if (Platform.isAndroid) return 'libmathops.so';
    if (Platform.isIOS) return 'math_ops.framework/math_ops';
    if (Platform.isMacOS) return 'libmathops.dylib';
    if (Platform.isLinux) return 'libmathops.so';
    if (Platform.isWindows) return 'mathops.dll';
    throw UnsupportedError('Unsupported platform');
  }

  /// Dart List를 네이티브 메모리에 복사 후 sum 계산
  int sumArray(List<int> data) {
    // calloc으로 네이티브 힙에 메모리 할당
    final ptr = calloc<Int32>(data.length);
    try {
      // Dart List → Native Array 복사
      for (int i = 0; i < data.length; i++) {
        ptr[i] = data[i];
      }
      return _sumArray(ptr, data.length);
    } finally {
      // 반드시 해제 (GC가 관리하지 않음!)
      calloc.free(ptr);
    }
  }

  double standardDeviation(List<double> data) {
    final ptr = calloc<Double>(data.length);
    try {
      for (int i = 0; i < data.length; i++) {
        ptr[i] = data[i];
      }
      return _stdDeviation(ptr, data.length);
    } finally {
      calloc.free(ptr);
    }
  }
}

// 사용 예시
void main() {
  final math = NativeMath();

  final numbers = List.generate(1000000, (i) => i);
  final sum = math.sumArray(numbers);
  print('Sum: $sum'); // Sum: 499999500000

  final doubles = [2.0, 4.0, 4.0, 4.0, 5.0, 5.0, 7.0, 9.0];
  final sd = math.standardDeviation(doubles);
  print('Std Dev: ${sd.toStringAsFixed(4)}'); // 2.0000
}
```

**핵심 포인트:** `calloc.free(ptr)`를 `finally` 블록에서 반드시 호출해야 합니다. Dart GC는 네이티브 메모리를 관리하지 않습니다.

---

## 5. 실제 구현 예제 2: C Struct와 콜백 함수

복잡한 데이터 구조를 주고받으려면 `Struct` 클래스를 사용합니다.

### C 헤더 (native/sensor.h)

```c
#include <stdint.h>

typedef struct {
    double x;
    double y;
    double z;
    int64_t timestamp_ms;
} SensorData;

typedef void (*SensorCallback)(const SensorData* data);

// 센서 시뮬레이터 (실제로는 하드웨어 드라이버)
void start_sensor(SensorCallback callback, int32_t interval_ms);
void stop_sensor(void);
```

### Dart에서 Struct 정의 및 콜백 등록

```dart
import 'dart:ffi';
import 'dart:isolate';

// C struct에 1:1 대응하는 Dart Struct
final class SensorData extends Struct {
  @Double()
  external double x;

  @Double()
  external double y;

  @Double()
  external double z;

  @Int64()
  external int timestampMs;
}

// 콜백 시그니처
typedef SensorCallbackNative = Void Function(Pointer<SensorData> data);
typedef StartSensorNative = Void Function(
    Pointer<NativeFunction<SensorCallbackNative>> callback,
    Int32 intervalMs);
typedef StartSensor = void Function(
    Pointer<NativeFunction<SensorCallbackNative>> callback,
    int intervalMs);
typedef StopSensorNative = Void Function();
typedef StopSensor = void Function();

class SensorManager {
  late final StartSensor _startSensor;
  late final StopSensor _stopSensor;
  NativeCallable<SensorCallbackNative>? _nativeCallable;

  SensorManager(DynamicLibrary lib) {
    _startSensor = lib
        .lookup<NativeFunction<StartSensorNative>>('start_sensor')
        .asFunction<StartSensor>();
    _stopSensor = lib
        .lookup<NativeFunction<StopSensorNative>>('stop_sensor')
        .asFunction<StopSensor>();
  }

  void startListening(void Function(double x, double y, double z) onData) {
    // NativeCallable: C → Dart 방향 콜백을 위한 안전한 래퍼
    // listener 타입: 임의 스레드에서 호출 가능
    _nativeCallable = NativeCallable<SensorCallbackNative>.listener(
      (Pointer<SensorData> dataPtr) {
        final data = dataPtr.ref;
        onData(data.x, data.y, data.z);
      },
    );

    _startSensor(_nativeCallable!.nativeFunction, 100); // 100ms 간격
  }

  void stopListening() {
    _stopSensor();
    _nativeCallable?.close(); // 콜백 리소스 해제 필수!
    _nativeCallable = null;
  }
}

// 사용 예시 (Flutter Widget에서)
void _initSensor() {
  final lib = DynamicLibrary.open('libsensor.so');
  final manager = SensorManager(lib);

  manager.startListening((x, y, z) {
    setState(() {
      _sensorX = x;
      _sensorY = y;
      _sensorZ = z;
    });
  });
}
```

`NativeCallable.listener`는 멀티스레드 환경에서 C 코드가 Dart 콜백을 안전하게 호출할 수 있게 해주는 핵심 클래스입니다. Flutter 3.7+에서 안정화되었습니다.

---

## 6. ffigen으로 바인딩 자동 생성

수백 개의 함수가 있는 대형 C 라이브러리를 수동으로 바인딩하는 것은 비현실적입니다. `package:ffigen`은 C 헤더 파일을 파싱해 Dart 바인딩을 자동 생성합니다.

### pubspec.yaml 설정

```yaml
dev_dependencies:
  ffigen: ^20.0.0

ffigen:
  name: MathOpsBindings
  description: Auto-generated bindings for libmathops
  output: lib/src/generated/math_ops_bindings.dart
  headers:
    entry-points:
      - native/math_ops.h
    include-directives:
      - '**math_ops.h'
  functions:
    include:
      names:
        - sum_array
        - std_deviation
  structs:
    include:
      names:
        - SensorData
```

생성 명령: `dart run ffigen --config ffigen.yaml`

---

## 7. 주의사항 및 실전 팁

### 메모리 관리 원칙

1. **`calloc` vs `malloc`**: `calloc`은 0으로 초기화, `malloc`은 초기화 안 함. struct를 다룰 때는 `calloc` 권장.
2. **`using` 패턴**: `package:ffi`의 `using()` 함수를 사용하면 스코프 기반으로 자동 해제됩니다.

```dart
// using()으로 메모리 수명 자동 관리
final result = using<int>((Arena arena) {
  final ptr = arena<Int32>(100); // arena가 스코프 종료 시 자동 해제
  // ... 작업 수행 ...
  return computeResult(ptr);
});
```

3. **`Pointer.fromAddress(0)`은 null 포인터**: C에서 `NULL`을 반환하는 함수는 `Pointer.isNull` 또는 주소 비교로 검사해야 합니다.

### 스레드 안전성

- Dart Isolate는 별도 스레드에서 실행됩니다. FFI 함수를 Isolate에서 호출하면 UI 스레드를 블로킹하지 않을 수 있습니다.
- 단, `NativeCallable.listener`로 등록한 콜백은 Dart 스레드 풀에서 처리되므로 `setState()`를 직접 호출하면 안 됩니다. `ReceivePort`나 `StreamController`로 메인 Isolate에 이벤트를 전달해야 합니다.

### AOT 컴파일 시 트리 쉐이킹

Flutter의 AOT 컴파일러는 사용되지 않는 심볼을 제거합니다. C 라이브러리의 심볼이 사라지지 않도록 Android의 경우 `CMakeLists.txt`에서 `set_target_properties(... PROPERTIES LINK_FLAGS "-Wl,--export-dynamic")`를 추가하거나, 심볼을 `extern "C" __attribute__((visibility("default")))`로 명시적으로 export해야 합니다.

### iOS 정적 링킹

iOS 앱은 동적 라이브러리(`.dylib`)를 앱 번들에 포함할 수 없습니다(App Store 정책). C 코드를 정적 라이브러리(`.a`)로 빌드하고 Xcode 프로젝트의 `OTHER_LDFLAGS`에 추가해야 합니다. `flutter create --template=package_ffi`로 생성한 패키지는 이 설정을 자동으로 처리합니다.

---

## 8. Flutter FFI 패키지 템플릿 활용

Flutter 3.10+부터는 `flutter create --template=package_ffi my_package` 명령으로 FFI 패키지 기본 구조를 자동 생성할 수 있습니다. 이 템플릿은:

- `build.dart` — 크로스플랫폼 네이티브 빌드 훅
- `src/` — C 소스 코드
- `lib/src/my_package_bindings_generated.dart` — 자동 생성 바인딩

의 구조로 iOS, Android, macOS, Linux, Windows 빌드를 단일 스크립트로 처리합니다.

---

## 마치며

`dart:ffi`는 Flutter/Dart 생태계에서 네이티브 성능을 끌어올릴 수 있는 강력한 도구입니다. 진입 장벽이 높지만, 포인터 타입 시스템과 메모리 관리 규칙을 한 번 익히고 나면 Platform Channel로는 불가능했던 성능 최적화와 레거시 C 라이브러리 통합이 가능해집니다.

특히 `using()` 패턴과 `NativeCallable.listener`를 활용한 콜백 등록, `ffigen`을 통한 자동 바인딩 생성 3가지를 실전에서 잘 조합한다면, 대형 C 라이브러리도 안전하게 Flutter 앱에 통합할 수 있습니다.

---

## 참고 자료

- [dart:ffi - ffi 패키지 (pub.dev)](https://pub.dev/packages/ffi)
- [ffigen - FFI 바인딩 자동 생성 도구 (pub.dev)](https://pub.dev/packages/ffigen)
- [Dart FFI C Interop 공식 가이드 (dart.dev)](https://dart.dev/interop/c-interop)
- [Flutter Bind Native Code 공식 문서 (docs.flutter.dev)](https://docs.flutter.dev/platform-integration/bind-native-code)
