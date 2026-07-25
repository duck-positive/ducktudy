---
layout: post
title: "Android NDK & JNI 심화: Kotlin에서 C/C++ 네이티브 코드를 호출하는 완전 가이드"
date: 2026-07-25
categories: [android, flutter]
tags: [android, ndk, jni, kotlin, cpp, native, cmake, performance]
---

Android 앱 개발의 99%는 Kotlin으로 충분하지만, 극한의 성능이 필요하거나 기존 C/C++ 라이브러리를 재사용해야 할 때는 NDK와 JNI를 피할 수 없습니다. 이 글에서는 단순한 "Hello, JNI"를 넘어, 실제 프로덕션에서 마주치는 패턴 — 배열 처리, Kotlin 콜백, 예외 처리, 스레드 안전성 — 을 모두 다룹니다.

---

## 1. 개념 설명: NDK와 JNI란?

**Android NDK(Native Development Kit)**는 C/C++ 코드를 Android 앱에 통합할 수 있게 해주는 도구 모음입니다. NDK를 통해 컴파일된 코드는 JVM 위에서 실행되는 것이 아니라, 장치의 CPU에서 직접 실행되는 네이티브 코드(`.so` 파일, ELF shared library)가 됩니다.

**JNI(Java Native Interface)**는 이 두 세계를 연결하는 다리입니다. Kotlin/Java 코드가 네이티브 C/C++ 함수를 호출하거나, 반대로 C/C++ 코드가 Kotlin/Java 메서드를 호출할 수 있게 해주는 프레임워크입니다.

```
┌─────────────────────────────────────┐
│   Kotlin/Java 코드 (ART/JVM)        │  ← 앱 로직, UI
├─────────────────────────────────────┤
│   JNI 레이어                        │  ← external fun, extern "C"
├─────────────────────────────────────┤
│   C/C++ 코드 (.so 네이티브 라이브러리)│  ← 고성능 연산, 기존 라이브러리
└─────────────────────────────────────┘
```

NDK를 사용하려면 세 가지 핵심 도구가 필요합니다:

- **NDK**: C/C++ 코드를 컴파일하는 크로스 컴파일러 툴체인
- **CMake**: 네이티브 빌드 스크립트 시스템 (현재 Android의 권장 빌드 시스템)
- **LLDB**: 네이티브 코드 디버거

---

## 2. 왜 NDK가 필요한가?

NDK 사용을 고려해야 하는 상황은 크게 세 가지입니다.

### 2.1 극한의 성능이 필요한 경우

JVM의 GC 일시 정지, JIT 컴파일 워밍업 오버헤드, 박싱/언박싱 비용 없이 CPU에 직접 접근합니다. 특히 다음 영역에서 효과적입니다:

- **실시간 오디오/비디오 처리**: 샘플 단위 오디오 필터, 코덱 처리
- **물리 시뮬레이션**: 게임 엔진의 충돌 감지, 강체 역학
- **신호 처리**: FFT, DSP 필터, 센서 데이터 처리
- **이미지/컴퓨터 비전**: 픽셀 단위 필터링, OpenCV 연동

### 2.2 기존 C/C++ 라이브러리 재사용

수십 년간 최적화된 C/C++ 라이브러리(OpenCV, FFTW, SQLite, OpenSSL, libpng 등)를 별도 포팅 없이 Android에서 직접 사용할 수 있습니다.

### 2.3 저수준 하드웨어 접근

Android NDK API를 통해 Sensor, OpenSL ES, Vulkan, OpenGL ES에 네이티브 레벨로 접근할 수 있습니다. Java 계층을 거치지 않으므로 레이턴시가 크게 줄어듭니다.

> **주의**: NDK는 만능 성능 해결사가 아닙니다. JNI 경계를 넘는 데이터 마샬링(타입 변환) 비용이 있으므로, 빈번한 소규모 호출보다는 대용량 데이터 배치 처리에 유리합니다.

---

## 3. 실제 구현 예제 1: 기본 JNI 함수 호출

### 3.1 프로젝트 구조

```
app/
├── src/
│   └── main/
│       ├── cpp/
│       │   ├── CMakeLists.txt
│       │   └── native_calculator.cpp
│       └── java/com/example/ndk/
│           └── NativeCalculator.kt
└── build.gradle.kts
```

### 3.2 CMakeLists.txt 설정

```cmake
cmake_minimum_required(VERSION 3.22.1)
project("ndkdemo")

add_library(
    native_calculator
    SHARED
    native_calculator.cpp
)

find_library(log-lib log)

target_link_libraries(
    native_calculator
    ${log-lib}
)
```

### 3.3 C++ 네이티브 구현

JNI 함수명은 `Java_패키지명_클래스명_함수명` 규칙을 따릅니다. 패키지명에서 `.`은 `_`로, `_`은 `_1`로 치환합니다.

```cpp
// native_calculator.cpp
#include <jni.h>
#include <string>
#include <android/log.h>
#include <cmath>

#define LOG_TAG "NativeCalc"
#define LOGI(...) __android_log_print(ANDROID_LOG_INFO, LOG_TAG, __VA_ARGS__)

extern "C" {

JNIEXPORT jstring JNICALL
Java_com_example_ndk_NativeCalculator_getLibraryVersion(
        JNIEnv* env, jobject /* thiz */) {
    return env->NewStringUTF("NDKDemo v1.0.0 (C++17)");
}

JNIEXPORT jlong JNICALL
Java_com_example_ndk_NativeCalculator_fibonacci(
        JNIEnv* env, jobject /* thiz */, jint n) {
    if (n <= 0) return 0;
    if (n == 1) return 1;

    jlong a = 0, b = 1;
    for (int i = 2; i <= n; ++i) {
        jlong c = a + b;
        a = b;
        b = c;
    }
    LOGI("fibonacci(%d) = %lld", n, b);
    return b;
}

// float 배열을 받아 RMS(Root Mean Square) 계산
JNIEXPORT jfloat JNICALL
Java_com_example_ndk_NativeCalculator_computeRms(
        JNIEnv* env, jobject /* thiz */,
        jfloatArray samples, jint length) {

    jboolean isCopy;
    jfloat* data = env->GetFloatArrayElements(samples, &isCopy);
    if (data == nullptr) return -1.0f;

    double sumOfSquares = 0.0;
    for (int i = 0; i < length; ++i) {
        sumOfSquares += static_cast<double>(data[i]) * data[i];
    }

    // 반드시 Release 호출: 0 = 변경사항 Java 배열에 반영 후 메모리 해제
    env->ReleaseFloatArrayElements(samples, data, 0);

    return static_cast<jfloat>(std::sqrt(sumOfSquares / length));
}

} // extern "C"
```

### 3.4 Kotlin JNI 선언 및 사용

```kotlin
class NativeCalculator {

    // external 키워드로 JNI 구현 선언
    external fun getLibraryVersion(): String
    external fun fibonacci(n: Int): Long
    external fun computeRms(samples: FloatArray, length: Int): Float

    companion object {
        init {
            System.loadLibrary("native_calculator")
        }
    }
}

// 사용 예
class MainActivity : AppCompatActivity() {
    private val calc = NativeCalculator()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        Log.i("NDK", calc.getLibraryVersion())       // NDKDemo v1.0.0 (C++17)
        Log.i("NDK", "fib(40) = ${calc.fibonacci(40)}")  // 102334155

        val samples = FloatArray(44100) { (Math.random() - 0.5).toFloat() }
        val rms = calc.computeRms(samples, samples.size)
        Log.i("NDK", "RMS = $rms")   // ~0.289 (균일 분포의 이론값)
    }
}
```

### 3.5 build.gradle.kts 설정

```kotlin
android {
    defaultConfig {
        ndk {
            abiFilters += listOf("arm64-v8a", "x86_64")
        }
        externalNativeBuild {
            cmake {
                cppFlags += listOf("-std=c++17", "-O2")
            }
        }
    }
    externalNativeBuild {
        cmake {
            path = file("src/main/cpp/CMakeLists.txt")
            version = "3.22.1"
        }
    }
}
```

---

## 4. 실제 구현 예제 2: 고급 JNI 패턴 — Kotlin 객체 조작과 콜백

네이티브 코드에서 Kotlin 객체를 생성하거나, 진행률을 콜백으로 보고하는 더 복잡한 패턴입니다. `JNI_OnLoad`를 활용해 `jclass`와 `jmethodID`를 캐싱하면 반복 조회 비용을 제거할 수 있습니다.

### 4.1 Kotlin 정의

```kotlin
data class ProcessResult(
    val brightness: Float,
    val contrast: Float,
    val dominantColor: Int
)

class ImageProcessor {

    fun interface ProgressCallback {
        fun onProgress(percent: Int, message: String)
    }

    external fun processImage(
        pixels: IntArray,
        width: Int,
        height: Int,
        callback: ProgressCallback
    ): ProcessResult

    companion object {
        init { System.loadLibrary("image_processor") }
    }
}
```

### 4.2 C++ 구현: JNI_OnLoad 활용과 콜백

```cpp
// image_processor.cpp
#include <jni.h>
#include <android/log.h>
#include <cmath>

static JavaVM* g_vm = nullptr;
static jclass  g_resultClass = nullptr;
static jmethodID g_resultCtor = nullptr;
static jmethodID g_onProgress = nullptr;

extern "C" {

JNIEXPORT jint JNI_OnLoad(JavaVM* vm, void* /* reserved */) {
    g_vm = vm;
    JNIEnv* env;
    if (vm->GetEnv(reinterpret_cast<void**>(&env), JNI_VERSION_1_6) != JNI_OK)
        return JNI_ERR;

    // ProcessResult 클래스 GlobalRef 캐싱
    jclass local = env->FindClass("com/example/ndk/ProcessResult");
    if (!local) return JNI_ERR;
    g_resultClass = reinterpret_cast<jclass>(env->NewGlobalRef(local));
    env->DeleteLocalRef(local);

    // 생성자: (float brightness, float contrast, int dominantColor)
    g_resultCtor = env->GetMethodID(g_resultClass, "<init>", "(FFI)V");

    // ProgressCallback.onProgress(int, String) 메서드 캐싱
    jclass cbClass = env->FindClass(
        "com/example/ndk/ImageProcessor$ProgressCallback");
    g_onProgress = env->GetMethodID(
        cbClass, "onProgress", "(ILjava/lang/String;)V");
    env->DeleteLocalRef(cbClass);

    return JNI_VERSION_1_6;
}

JNIEXPORT void JNI_OnUnload(JavaVM* vm, void* /* reserved */) {
    JNIEnv* env;
    vm->GetEnv(reinterpret_cast<void**>(&env), JNI_VERSION_1_6);
    if (g_resultClass) {
        env->DeleteGlobalRef(g_resultClass);
        g_resultClass = nullptr;
    }
}

JNIEXPORT jobject JNICALL
Java_com_example_ndk_ImageProcessor_processImage(
        JNIEnv* env, jobject /* thiz */,
        jintArray pixels, jint width, jint height,
        jobject callback) {

    jint* data = env->GetIntArrayElements(pixels, nullptr);
    if (!data) return nullptr;

    jlong total = (jlong)width * height;
    double sumBright = 0.0;
    long long buckets[8] = {};

    for (jlong i = 0; i < total; ++i) {
        jint px = data[i];
        int r = (px >> 16) & 0xFF;
        int g = (px >>  8) & 0xFF;
        int b =  px        & 0xFF;

        // BT.709 가중 밝기
        sumBright += 0.2126 * r + 0.7152 * g + 0.0722 * b;
        buckets[((r >> 5) & 4) | ((g >> 6) & 2) | (b >> 7)]++;

        // 10% 단위 진행률 콜백
        if (total >= 10 && i % (total / 10) == 0) {
            int pct = (int)(i * 100 / total);
            jstring msg = env->NewStringUTF("이미지 분석 중...");
            env->CallVoidMethod(callback, g_onProgress, pct, msg);
            env->DeleteLocalRef(msg);

            if (env->ExceptionCheck()) {
                env->ExceptionClear();
                env->ReleaseIntArrayElements(pixels, data, JNI_ABORT);
                return nullptr;
            }
        }
    }

    env->ReleaseIntArrayElements(pixels, data, JNI_ABORT);

    float brightness = (float)(sumBright / total / 255.0);

    int dominant = 0;
    for (int i = 1; i < 8; ++i)
        if (buckets[i] > buckets[dominant]) dominant = i;

    // Kotlin 객체 생성 후 반환
    return env->NewObject(g_resultClass, g_resultCtor,
                          brightness, 0.5f, (jint)(dominant * 32));
}

} // extern "C"
```

### 4.3 Kotlin에서 사용

```kotlin
val processor = ImageProcessor()
val bmp = BitmapFactory.decodeResource(resources, R.drawable.sample)
val pixels = IntArray(bmp.width * bmp.height)
bmp.getPixels(pixels, 0, bmp.width, 0, 0, bmp.width, bmp.height)

val result = processor.processImage(
    pixels  = pixels,
    width   = bmp.width,
    height  = bmp.height,
    callback = { pct, msg -> Log.d("NDK", "[$pct%] $msg") }
)

Log.i("NDK", "밝기: ${"%.3f".format(result.brightness)}")
Log.i("NDK", "주요색: #${result.dominantColor.toString(16).padStart(6, '0')}")
```

---

## 5. 주의사항 및 실전 팁

### 5.1 JNI 경계 호출 횟수 최소화

JNI 호출 자체에는 약 수 μs의 오버헤드가 있습니다. 루프 내부에서 반복 호출하는 대신 배열 단위로 일괄 처리하세요.

```kotlin
// 나쁜 예: JNI 호출 N회
for (sample in audioSamples) calc.processSample(sample)

// 좋은 예: JNI 호출 1회
calc.processSampleBatch(audioSamples)
```

### 5.2 LocalRef 슬롯 누수 방지

JNI LocalRef 슬롯은 기본 512개입니다. 루프 안에서 `NewStringUTF`, `NewObject` 등을 호출하면 슬롯이 고갈되어 `OutOfMemoryError`가 발생합니다.

```cpp
for (int i = 0; i < 10000; ++i) {
    jstring s = env->NewStringUTF("hello");
    // ... 사용 ...
    env->DeleteLocalRef(s);  // 반드시 수동 삭제
}
// 또는 PushLocalFrame / PopLocalFrame으로 일괄 관리
env->PushLocalFrame(16);
// ... LocalRef 생성 ...
env->PopLocalFrame(nullptr);  // 프레임 내 모든 LocalRef 해제
```

### 5.3 JNIEnv는 스레드 로컬

`JNIEnv*`는 **스레드별 독립 객체**입니다. 전역 변수로 저장하면 크래시가 발생합니다. 다른 스레드에서 JNI가 필요하면 `JavaVM`을 통해 획득하세요.

```cpp
// ❌ 절대 금지
static JNIEnv* g_env;

// ✅ 올바른 방법: JavaVM 저장 후 필요 시 Attach
void workerThread() {
    JNIEnv* env;
    g_vm->AttachCurrentThread(&env, nullptr);
    // ... JNI 작업 ...
    g_vm->DetachCurrentThread();
}
```

### 5.4 예외 확인 습관화

`CallVoidMethod`, `CallObjectMethod` 등 Kotlin 메서드를 호출한 후에는 반드시 `ExceptionCheck()`로 예외 여부를 확인해야 합니다. 예외가 있는 상태에서 대부분의 JNI 함수를 호출하면 정의되지 않은 동작(UB)이 발생합니다.

```cpp
env->CallVoidMethod(obj, method, arg);
if (env->ExceptionCheck()) {
    env->ExceptionDescribe();  // 로그캣에 스택 트레이스 출력
    env->ExceptionClear();
    // 정리 후 빠른 종료
    return nullptr;
}
```

### 5.5 CheckJNI로 개발 중 버그 조기 발견

개발 단계에서 `CheckJNI`를 활성화하면 잘못된 레퍼런스 사용, 타입 불일치, 슬롯 누수를 즉시 감지합니다.

```bash
# 일반 디바이스에서 활성화
adb shell setprop debug.checkjni 1
```

### 5.6 @FastNative / @CriticalNative (Android 8+)

단순 연산 함수에는 JNI 디스패치 오버헤드를 줄이는 어노테이션을 사용할 수 있습니다:

```kotlin
// FastNative: 오버헤드 ~30% 감소, managed 힙 접근 가능
@FastNative
external fun fastAdd(a: Int, b: Int): Int

// CriticalNative: 오버헤드 ~90% 감소, JNIEnv/jclass 파라미터 없음
// managed 객체 파라미터 불가, GC를 일시 정지시키므로 빠른 함수에만 사용
@CriticalNative
external fun criticalDotProduct(a: FloatArray, b: FloatArray, n: Int): Float
```

C++ 쪽에서는 `RegisterNatives`로 명시적으로 등록해야 합니다(Android 8~13). Android 14+에서는 공개 API로 지원됩니다.

---

## 마무리

Android NDK와 JNI는 진입 장벽이 있지만, 한 번 익히면 Kotlin만으로는 달성할 수 없는 성능 영역을 개척할 수 있습니다. 핵심 원칙을 정리하면:

| 원칙 | 내용 |
|------|------|
| 경계 최소화 | JNI 호출을 배치 단위로 묶어 오버헤드 분산 |
| LocalRef 관리 | 루프 내 생성 시 즉시 `DeleteLocalRef` 또는 `PushLocalFrame` |
| JNIEnv 격리 | 스레드 간 공유 금지, `AttachCurrentThread` 사용 |
| 예외 처리 | 모든 Kotlin 메서드 호출 후 `ExceptionCheck()` |
| GlobalRef 정리 | `JNI_OnUnload`에서 `DeleteGlobalRef` 호출 |
| CheckJNI | 개발 중 항상 활성화하여 문제 조기 발견 |

NDK는 모든 곳에 적용할 도구가 아닙니다. 프로파일링을 통해 실제 병목을 확인하고, 그 지점에 집중적으로 적용하는 것이 올바른 접근법입니다.

---

## 참고 자료

- [Android NDK 시작하기 — 공식 가이드](https://developer.android.com/ndk/guides)
- [JNI Tips — Android 공식 문서](https://developer.android.com/ndk/guides/jni-tips)
- [프로젝트에 C/C++ 코드 추가하기 — Android Studio](https://developer.android.com/studio/projects/add-native-code)
- [Android NDK API 레퍼런스](https://developer.android.com/ndk/reference)
