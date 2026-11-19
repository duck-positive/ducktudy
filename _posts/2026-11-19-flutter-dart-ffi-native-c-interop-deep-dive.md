---
layout: post
title: "Flutter dart:ffi 심화: 네이티브 C/C++ 라이브러리를 직접 호출하는 완전 가이드"
date: 2026-11-19
categories: [android, flutter]
tags: [flutter, dart, ffi, native, c, cpp, interop, 네이티브, 성능최적화]
---

Flutter는 대부분의 UI와 비즈니스 로직을 Dart로 처리하지만, 때로는 C나 C++로 작성된 고성능 라이브러리를 직접 호출해야 하는 상황이 생깁니다. 이미지 처리, 암호화, 음성 코덱, 물리 엔진처럼 수십 년에 걸쳐 최적화된 C 라이브러리를 Dart에서 재구현하는 것은 비효율적입니다. `dart:ffi`(Foreign Function Interface)는 Dart 코드에서 네이티브 라이브러리를 직접 호출할 수 있게 해 주는 공식 메커니즘입니다.

## dart:ffi란 무엇인가?

FFI(Foreign Function Interface)는 한 언어로 작성된 코드가 다른 언어로 작성된 함수를 호출할 수 있도록 하는 인터페이스입니다. `dart:ffi`는 Dart 2.6에서 실험적으로 도입되었고, Dart 2.12(Null Safety)와 함께 안정화되었습니다. Dart 3.x 에서는 Native Assets 기능이 추가되어 빌드 통합까지 지원하고 있습니다.

`dart:ffi`의 핵심 개념은 다음과 같습니다.

- **NativeType**: C의 기본 타입에 대응하는 Dart 타입 계층. `Int32`, `Double`, `Pointer`, `Struct` 등이 있습니다.
- **Pointer\<T\>**: 네이티브 메모리 주소를 나타내는 타입. C의 포인터와 직접 대응됩니다.
- **DynamicLibrary**: 공유 라이브러리(.so, .dylib, .dll)를 열어 함수 심볼을 조회합니다.
- **NativeFunction**: 네이티브 C 함수의 타입 시그니처를 Dart 타입 시스템 안에서 표현합니다.

플랫폼 채널(MethodChannel)과 비교하면 FFI는 직접 메모리를 공유하므로 직렬화 오버헤드가 없습니다. 대신 메모리 관리를 직접 해야 하고, 타입 안전성이 낮아 버그 발생 가능성이 높습니다.

## 왜 dart:ffi가 필요한가?

### 1. 성능이 극단적으로 중요한 연산

AES 암호화, FFT(고속 푸리에 변환), H.264 디코딩처럼 수백만 번의 반복 연산이 필요한 경우, Dart 구현은 C 구현에 비해 2~10배 느릴 수 있습니다. 특히 모바일 환경에서 배터리 소모와 발열을 줄이려면 C 레벨 최적화가 결정적입니다.

### 2. 기존 C/C++ 라이브러리 재사용

OpenCV, SQLite, libsodium, zstd 같은 라이브러리는 이미 수십 년간 검증된 C 구현이 존재합니다. 이를 Dart로 재구현하는 것은 시간 낭비이며, `dart:ffi`로 그대로 호출하면 됩니다.

### 3. 플랫폼 채널의 한계 극복

MethodChannel은 메시지를 직렬화하여 메인 스레드에서 처리합니다. 초당 수천 번 호출이 필요한 실시간 센서 데이터 처리나 오디오 콜백 같은 경우에는 채널 오버헤드가 치명적입니다. `dart:ffi`는 직렬화 없이 직접 호출하므로 이런 시나리오에 적합합니다.

### 4. Dart VM이 없는 환경

Flutter 앱의 Dart 코드가 아니라 Dart CLI나 임베디드 환경에서도 `dart:ffi`를 사용하여 시스템 라이브러리와 상호작용할 수 있습니다.

## 실제 구현 예제

### 예제 1: C 함수를 dart:ffi로 직접 호출하기

먼저 간단한 C 연산 함수를 정의하고, Dart에서 직접 호출하는 예제입니다.

**native/math_utils.c**
```c
#include <stdint.h>
#include <math.h>

// 두 벡터의 내적(dot product) 계산 — SIMD 최적화 가능한 핵심 연산
double dot_product(const double* a, const double* b, int32_t length) {
    double result = 0.0;
    for (int32_t i = 0; i < length; i++) {
        result += a[i] * b[i];
    }
    return result;
}

// 문자열 역순 — 메모리 직접 조작 예시
void reverse_string(char* str, int32_t length) {
    for (int32_t i = 0, j = length - 1; i < j; i++, j--) {
        char tmp = str[i];
        str[i] = str[j];
        str[j] = tmp;
    }
}
```

**lib/native_math.dart**
```dart
import 'dart:ffi';
import 'dart:io';
import 'package:ffi/ffi.dart'; // pub.dev의 helper 패키지

// C 함수 타입 정의
typedef DotProductC = Double Function(
    Pointer<Double>, Pointer<Double>, Int32);
typedef DotProductDart = double Function(
    Pointer<Double>, Pointer<Double>, int);

typedef ReverseStringC = Void Function(Pointer<Utf8>, Int32);
typedef ReverseStringDart = void Function(Pointer<Utf8>, int);

class NativeMath {
  late final DynamicLibrary _lib;
  late final DotProductDart _dotProduct;
  late final ReverseStringDart _reverseString;

  NativeMath() {
    // 플랫폼별 라이브러리 경로 처리
    _lib = _loadLibrary();

    // 함수 심볼 조회 및 Dart 함수로 변환
    _dotProduct = _lib
        .lookupFunction<DotProductC, DotProductDart>('dot_product');
    _reverseString = _lib
        .lookupFunction<ReverseStringC, ReverseStringDart>('reverse_string');
  }

  DynamicLibrary _loadLibrary() {
    if (Platform.isAndroid) {
      return DynamicLibrary.open('libmath_utils.so');
    } else if (Platform.isIOS || Platform.isMacOS) {
      // iOS/macOS는 정적 링크 또는 프레임워크로 번들
      return DynamicLibrary.process();
    } else if (Platform.isWindows) {
      return DynamicLibrary.open('math_utils.dll');
    }
    throw UnsupportedError('지원하지 않는 플랫폼: ${Platform.operatingSystem}');
  }

  /// 두 벡터의 내적을 계산합니다. O(n) 시간복잡도의 C 구현을 직접 호출.
  double dotProduct(List<double> a, List<double> b) {
    assert(a.length == b.length, '벡터 길이가 일치해야 합니다');
    final int n = a.length;

    // calloc으로 C 힙에 메모리 할당 (Dart GC 바깥)
    final Pointer<Double> pA = calloc<Double>(n);
    final Pointer<Double> pB = calloc<Double>(n);

    try {
      // Dart 리스트 → 네이티브 메모리 복사
      for (int i = 0; i < n; i++) {
        pA[i] = a[i];
        pB[i] = b[i];
      }
      return _dotProduct(pA, pB, n);
    } finally {
      // 반드시 해제! 누수 시 OOM 위험
      calloc.free(pA);
      calloc.free(pB);
    }
  }

  /// C에서 문자열을 직접 역순 처리 (Utf8 변환 예시)
  String reverseString(String input) {
    // Dart String → C UTF-8 문자열 (null terminator 포함)
    final Pointer<Utf8> pStr = input.toNativeUtf8();
    try {
      _reverseString(pStr, input.length);
      return pStr.toDartString();
    } finally {
      calloc.free(pStr);
    }
  }
}
```

**사용 예시**
```dart
void main() {
  final math = NativeMath();

  // 내적 계산
  final a = [1.0, 2.0, 3.0];
  final b = [4.0, 5.0, 6.0];
  print('내적: ${math.dotProduct(a, b)}'); // 32.0

  // 문자열 역순
  print(math.reverseString('flutter')); // rettulf
}
```

---

### 예제 2: Struct와 Isolate를 활용한 블로킹 FFI 처리

FFI 호출은 동기적으로 호출 스레드에서 실행됩니다. 오래 걸리는 C 함수를 메인 아이솔릿에서 호출하면 UI가 멈춥니다. `Isolate.run`을 활용하여 비동기적으로 처리하는 패턴을 살펴봅니다.

```dart
import 'dart:ffi';
import 'dart:isolate';
import 'package:ffi/ffi.dart';

// C 구조체와 1:1 대응하는 Dart Struct
final class ImageBuffer extends Struct {
  @Int32()
  external int width;

  @Int32()
  external int height;

  @Int32()
  external int channels;

  external Pointer<Uint8> data;
}

// C 함수 시그니처 정의
typedef ApplyGaussianBlurC = Void Function(
    Pointer<ImageBuffer>, Float sigma);
typedef ApplyGaussianBlurDart = void Function(
    Pointer<ImageBuffer>, double sigma);

class ImageProcessor {
  static late final DynamicLibrary _lib;
  static late final ApplyGaussianBlurDart _applyGaussianBlur;

  static void _initLib() {
    _lib = DynamicLibrary.open('libimage_processor.so');
    _applyGaussianBlur = _lib.lookupFunction<
        ApplyGaussianBlurC, ApplyGaussianBlurDart>('apply_gaussian_blur');
  }

  /// 가우시안 블러를 Isolate 안에서 실행하여 UI 스레드를 보호합니다.
  static Future<List<int>> blurAsync({
    required List<int> pixels,
    required int width,
    required int height,
    required double sigma,
  }) async {
    // Isolate.run: Dart 2.19+, 별도 스레드에서 실행 후 결과 반환
    return await Isolate.run(() {
      _initLib();

      final int totalBytes = width * height * 3; // RGB
      final Pointer<Uint8> pixelPtr = calloc<Uint8>(totalBytes);
      final Pointer<ImageBuffer> bufPtr = calloc<ImageBuffer>();

      try {
        // 픽셀 데이터를 네이티브 메모리에 복사
        for (int i = 0; i < totalBytes; i++) {
          pixelPtr[i] = pixels[i];
        }

        // 구조체 필드 초기화
        bufPtr.ref.width = width;
        bufPtr.ref.height = height;
        bufPtr.ref.channels = 3;
        bufPtr.ref.data = pixelPtr;

        // 블로킹 C 함수 호출 (UI 스레드 아님!)
        _applyGaussianBlur(bufPtr, sigma);

        // 결과를 Dart List로 복사하여 반환 (Isolate 간 전달 가능)
        return List<int>.generate(
            totalBytes, (i) => bufPtr.ref.data[i]);
      } finally {
        calloc.free(pixelPtr);
        calloc.free(bufPtr);
      }
    });
  }
}
```

**Flutter Widget에서 호출**
```dart
class ImageFilterWidget extends StatefulWidget {
  final Uint8List imageData;
  final int width, height;

  const ImageFilterWidget(
      {super.key, required this.imageData,
       required this.width, required this.height});

  @override
  State<ImageFilterWidget> createState() => _ImageFilterWidgetState();
}

class _ImageFilterWidgetState extends State<ImageFilterWidget> {
  bool _isProcessing = false;
  List<int>? _processedPixels;

  Future<void> _applyBlur() async {
    setState(() => _isProcessing = true);
    try {
      final result = await ImageProcessor.blurAsync(
        pixels: widget.imageData.toList(),
        width: widget.width,
        height: widget.height,
        sigma: 2.0,
      );
      setState(() => _processedPixels = result);
    } finally {
      setState(() => _isProcessing = false);
    }
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        if (_isProcessing)
          const CircularProgressIndicator()
        else
          ElevatedButton(
              onPressed: _applyBlur, child: const Text('블러 적용')),
        if (_processedPixels != null)
          Text('처리 완료: ${_processedPixels!.length} bytes'),
      ],
    );
  }
}
```

---

## NativeCallable: C에서 Dart 콜백 받기

Dart 3.1부터 `NativeCallable`을 사용하면 C에서 Dart 함수를 콜백으로 호출할 수 있습니다. 오디오 콜백, 센서 인터럽트, 비동기 완료 핸들러 패턴에 유용합니다.

```dart
import 'dart:ffi';

typedef ProgressCallbackC = Void Function(Int32 progress);

// C에서 호출할 Dart 함수를 네이티브 콜백으로 등록
void registerProgressCallback() {
  final callback = NativeCallable<ProgressCallbackC>.listener(
    (int progress) {
      // 항상 메인 아이솔릿에서 실행됨
      print('진행률: $progress%');
    },
  );

  // C 함수에 콜백 포인터 전달
  final lib = DynamicLibrary.open('libworker.so');
  final register = lib.lookupFunction<
      Void Function(Pointer<NativeFunction<ProgressCallbackC>>),
      void Function(Pointer<NativeFunction<ProgressCallbackC>>)>(
    'register_progress_callback',
  );
  register(callback.nativeFunction);

  // 콜백이 더 이상 필요 없을 때 반드시 close() 호출
  // callback.close();
}
```

## package:ffigen으로 바인딩 자동 생성

C 헤더 파일이 수백 줄이면 `typedef`와 `lookupFunction`을 직접 작성하는 것은 비현실적입니다. `package:ffigen`을 사용하면 `.h` 파일에서 Dart 바인딩 코드를 자동으로 생성할 수 있습니다.

**pubspec.yaml**
```yaml
dev_dependencies:
  ffigen: ^12.0.0

ffigen:
  name: LibSodiumBindings
  description: 'libsodium C 바인딩'
  output: 'lib/src/libsodium_bindings.dart'
  headers:
    entry-points:
      - 'third_party/libsodium/include/sodium.h'
  functions:
    include:
      - 'crypto_.*'
      - 'sodium_.*'
```

```bash
dart run ffigen
```

ffigen은 헤더 파싱에 libclang을 사용하므로 LLVM을 설치해야 합니다. 생성된 코드는 `NativeLibrary` 클래스 형태로 모든 함수를 포함합니다.

## 주의사항과 실전 팁

### 1. 메모리 누수 방지를 위한 try/finally 필수

`calloc.free()` 없이 `calloc<T>()` 로 할당한 메모리는 Dart GC가 관리하지 않습니다. 반드시 `try/finally` 블록으로 감싸야 합니다. Arena 패키지의 `using()` 헬퍼를 활용하면 더 안전합니다.

```dart
import 'package:ffi/ffi.dart';

double safeCompute(List<double> values) {
  return using((Arena arena) {
    final ptr = arena<Double>(values.length);
    for (int i = 0; i < values.length; i++) ptr[i] = values[i];
    return _cFunction(ptr, values.length);
    // using 블록을 벗어나면 arena가 모든 할당을 자동 해제
  });
}
```

### 2. FFI 호출은 동기적 — UI 스레드 블로킹 금지

FFI 호출은 `async/await`로 비동기화되지 않습니다. 1ms 이상 걸릴 수 있는 모든 C 함수는 `Isolate.run` 또는 `compute`로 별도 스레드에서 실행해야 합니다.

### 3. 타입 불일치는 런타임 크래시

C의 `int`(보통 32비트)와 Dart의 `Int64` 불일치, 또는 `Float`과 `Double` 혼동은 조용히 잘못된 값을 반환하거나 세그멘테이션 폴트를 유발합니다. C 헤더의 정확한 타입을 반드시 확인하세요. `stdint.h`의 `int32_t`, `uint8_t` 같은 고정폭 타입을 사용하는 C 코드가 더 안전합니다.

### 4. Android에서 .so 파일 패키징

CMake를 사용하여 빌드하고 `android/app/build.gradle`에 ABI 필터를 설정하세요. 모든 ABI(`armeabi-v7a`, `arm64-v8a`, `x86_64`)를 포함하면 APK 크기가 커지므로, 타겟 기기에 맞는 ABI만 포함하는 것을 권장합니다.

### 5. iOS에서 정적 링크

iOS는 동적 라이브러리 로딩(`dlopen`) 제한이 있어, 보통 정적 라이브러리(.a)를 XCode 프로젝트에 링크하고 `DynamicLibrary.process()`로 접근합니다.

### 6. Native Assets (실험적 기능)

Dart 3.3+의 Native Assets는 `pubspec.yaml`에 C 소스를 선언하면 `flutter build` 시 자동으로 컴파일하고 번들합니다. `package_ffi` 템플릿을 사용하면 플랫폼별 빌드 설정 없이 멀티플랫폼 FFI 패키지를 만들 수 있습니다.

```bash
flutter create --template=package_ffi my_native_lib
```

## 정리

`dart:ffi`는 Flutter에서 성능이 중요한 네이티브 코드를 호출하는 가장 직접적이고 효율적인 방법입니다. 플랫폼 채널 대비 직렬화 오버헤드가 없어 초당 수천 번 이상의 호출도 가능합니다. 다만, 메모리를 직접 관리해야 하므로 `try/finally` 패턴과 `Arena`를 습관화하고, 블로킹 C 함수는 반드시 Isolate 안에서 실행해야 합니다. ffigen으로 바인딩을 자동 생성하고 Native Assets로 빌드 통합까지 완성하면, 검증된 C 라이브러리의 성능을 Flutter 앱 전체에서 그대로 활용할 수 있습니다.

## 참고 자료
- [Flutter dart:ffi 공식 가이드 (flutter.dev)](https://docs.flutter.dev/interop/dart-ffi)
- [C interop using dart:ffi (dart.dev)](https://dart.dev/server/c-interop)
- [dart:ffi API 레퍼런스 (api.dart.dev)](https://api.dart.dev/stable/dart-ffi/dart-ffi-library.html)
- [package:ffigen — C 바인딩 코드 자동 생성 (pub.dev)](https://pub.dev/packages/ffigen)
