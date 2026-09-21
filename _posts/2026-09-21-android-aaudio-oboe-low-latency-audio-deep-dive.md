---
layout: post
title: "Android 저지연 오디오: AAudio와 Oboe 라이브러리 심화 가이드"
date: 2026-09-21
categories: [android]
tags: [android, aaudio, oboe, ndk, audio, low-latency, jni, cpp]
---

오디오 레이턴시(latency)는 사용자가 버튼을 누르거나 악기를 연주하는 순간부터 실제 소리가 출력되기까지의 시간 지연입니다. 게임, 음악 제작 앱, 실시간 음성 통화 앱에서 이 지연이 20ms를 넘어서면 사용자는 즉각적으로 어색함을 느끼기 시작합니다. Android 8.0(Oreo)부터 도입된 **AAudio**와, 이를 감싸는 오픈소스 C++ 라이브러리 **Oboe**는 바로 이 문제를 근본적으로 해결하기 위해 설계되었습니다.

이 글에서는 AAudio와 Oboe의 내부 동작 원리를 이해하고, JNI를 통해 Kotlin 앱에 통합하는 실전 구현을 단계별로 살펴봅니다.

---

## 왜 저지연 오디오가 어려운가?

기존 Android 오디오 스택을 돌아보면 문제가 명확해집니다.

- **MediaPlayer / SoundPool**: Java 레이어와 AudioFlinger 서비스 사이의 여러 레이어를 경유. 레이턴시가 평균 100~200ms에 달합니다.
- **AudioTrack (Java)**: MediaPlayer보다 낮지만, 여전히 GC 멈춤(pause)과 JNI 오버헤드가 발생합니다.
- **OpenSL ES**: C API이지만 API 설계가 복잡하고, 특정 기기별 버그가 많아 직접 사용이 어렵습니다.

**AAudio**는 Android 8.0에서 등장한 C Native API로, AudioFlinger를 통하지 않는 **MMAP(Memory-Mapped) 경로**를 활용해 오디오 데이터를 HAL(Hardware Abstraction Layer)에 직접 전달합니다. 이론적으로 5~10ms 수준의 레이턴시 달성이 가능합니다.

**Oboe**는 AAudio의 C++ 래퍼로:
- API 27(Android 8.1) 이상에서는 AAudio를 사용
- 그 이하에서는 OpenSL ES로 자동 폴백
- 기기별 버그 우회 로직(QuirksManager) 내장
- API 16(Android 4.1)부터 99%의 기기를 지원

실무에서는 Oboe를 직접 사용하고, AAudio를 이해하는 것이 최적의 전략입니다.

---

## AAudio 핵심 개념

### 오디오 스트림(AudioStream)

모든 오디오 I/O는 스트림 단위로 이루어집니다. 스트림은 다음 속성으로 구성됩니다:

| 속성 | 설명 |
|------|------|
| Direction | INPUT(마이크) 또는 OUTPUT(스피커) |
| SharingMode | EXCLUSIVE(전용, 최저 레이턴시) / SHARED(공유) |
| PerformanceMode | LOW_LATENCY / POWER_SAVING / NONE |
| SampleRate | 48000Hz 권장 (디바이스 네이티브 레이트) |
| Format | FLOAT / I16 / I24 / I32 |

### 콜백 vs 블로킹 I/O

저지연을 위해서는 **콜백 방식**이 필수입니다. AAudio는 오디오 데이터가 필요할 때마다 고우선순위 스레드에서 콜백 함수를 호출합니다. 블로킹 `write()` 방식은 스케줄러 지연에 취약합니다.

콜백 내부에서 절대로 하지 말아야 할 것들:
- `malloc()` / `free()` (메모리 할당)
- `mutex` 잠금 (데드락 위험)
- 파일 I/O, 네트워크 I/O
- `sleep()` 또는 블로킹 대기
- JNI 호출 (GC 트리거 가능성)

---

## 실전 구현 예제

### 예제 1: Oboe를 사용한 사인파 재생 (C++)

먼저 `app/build.gradle`에 Oboe를 추가합니다:

```kotlin
android {
    defaultConfig {
        externalNativeBuild {
            cmake {
                arguments "-DANDROID_STL=c++_shared"
            }
        }
    }
    externalNativeBuild {
        cmake {
            path "src/main/cpp/CMakeLists.txt"
        }
    }
}

dependencies {
    implementation("androidx.games:games-activity:3.0.4")
    // Oboe를 CMakeLists.txt에서 find_package로 가져옴
}
```

`CMakeLists.txt`:

```cmake
cmake_minimum_required(VERSION 3.22.1)
project(AudioEngine)

# Oboe 라이브러리 연결
find_package(oboe REQUIRED CONFIG)

add_library(audio_engine SHARED
    AudioEngine.cpp
)

target_link_libraries(audio_engine
    oboe::oboe
    android
    log
)
```

`AudioEngine.h`:

```cpp
#pragma once
#include <oboe/Oboe.h>
#include <cmath>
#include <atomic>

class AudioEngine : public oboe::AudioStreamDataCallback {
public:
    oboe::Result start();
    void stop();

    oboe::DataCallbackResult onAudioReady(
        oboe::AudioStream* audioStream,
        void* audioData,
        int32_t numFrames) override;

private:
    std::shared_ptr<oboe::AudioStream> mStream;
    std::atomic<float> mPhase{0.0f};
    static constexpr float kFrequency = 440.0f; // A4 음
    static constexpr float kAmplitude = 0.5f;
    float mPhaseIncrement = 0.0f;
};
```

`AudioEngine.cpp`:

```cpp
#include "AudioEngine.h"
#include <android/log.h>

#define LOG_TAG "AudioEngine"
#define LOGI(...) __android_log_print(ANDROID_LOG_INFO, LOG_TAG, __VA_ARGS__)

oboe::Result AudioEngine::start() {
    oboe::AudioStreamBuilder builder;
    builder
        .setDirection(oboe::Direction::Output)
        .setPerformanceMode(oboe::PerformanceMode::LowLatency)   // 저지연 모드
        .setSharingMode(oboe::SharingMode::Exclusive)            // 전용 모드
        .setFormat(oboe::AudioFormat::Float)
        .setChannelCount(oboe::ChannelCount::Stereo)
        .setUsage(oboe::Usage::Game)                             // 게임 사용 선언
        .setDataCallback(this);

    oboe::Result result = builder.openManagedStream(mStream);
    if (result != oboe::Result::OK) {
        LOGI("스트림 열기 실패: %s", oboe::convertToText(result));
        return result;
    }

    // 달성된 레이턴시 확인
    LOGI("샘플레이트: %d", mStream->getSampleRate());
    LOGI("버퍼 크기(프레임): %d", mStream->getBufferSizeInFrames());
    LOGI("성능 모드: %s",
        mStream->getPerformanceMode() == oboe::PerformanceMode::LowLatency
            ? "LowLatency" : "Other");

    mPhaseIncrement = 2.0f * M_PI * kFrequency / mStream->getSampleRate();

    result = mStream->requestStart();
    return result;
}

void AudioEngine::stop() {
    if (mStream) {
        mStream->requestStop();
        mStream->close();
    }
}

// 이 함수는 고우선순위 오디오 스레드에서 호출됨 - JNI/malloc 금지
oboe::DataCallbackResult AudioEngine::onAudioReady(
        oboe::AudioStream* audioStream,
        void* audioData,
        int32_t numFrames) {

    float* output = static_cast<float*>(audioData);
    int32_t channelCount = audioStream->getChannelCount();

    for (int i = 0; i < numFrames; ++i) {
        float sample = sinf(mPhase.load()) * kAmplitude;
        for (int ch = 0; ch < channelCount; ++ch) {
            *output++ = sample;
        }
        float nextPhase = mPhase.load() + mPhaseIncrement;
        if (nextPhase >= 2.0f * M_PI) nextPhase -= 2.0f * M_PI;
        mPhase.store(nextPhase);
    }

    return oboe::DataCallbackResult::Continue;
}
```

---

### 예제 2: Kotlin에서 JNI로 제어하기

C++ 엔진을 Kotlin에서 제어하기 위한 JNI 브릿지와 Kotlin 클래스입니다.

`AudioEngineJNI.cpp` (JNI 브릿지):

```cpp
#include <jni.h>
#include "AudioEngine.h"

static AudioEngine* gEngine = nullptr;

extern "C" {

JNIEXPORT jlong JNICALL
Java_com_example_audioapp_AudioEngineController_nativeCreate(
        JNIEnv* env, jobject /* thiz */) {
    auto* engine = new AudioEngine();
    return reinterpret_cast<jlong>(engine);
}

JNIEXPORT void JNICALL
Java_com_example_audioapp_AudioEngineController_nativeStart(
        JNIEnv* env, jobject /* thiz */, jlong handle) {
    auto* engine = reinterpret_cast<AudioEngine*>(handle);
    engine->start();
}

JNIEXPORT void JNICALL
Java_com_example_audioapp_AudioEngineController_nativeStop(
        JNIEnv* env, jobject /* thiz */, jlong handle) {
    auto* engine = reinterpret_cast<AudioEngine*>(handle);
    engine->stop();
}

JNIEXPORT void JNICALL
Java_com_example_audioapp_AudioEngineController_nativeDestroy(
        JNIEnv* env, jobject /* thiz */, jlong handle) {
    delete reinterpret_cast<AudioEngine*>(handle);
}

} // extern "C"
```

`AudioEngineController.kt`:

```kotlin
class AudioEngineController : Closeable {

    private val nativeHandle: Long = nativeCreate()

    init {
        require(nativeHandle != 0L) { "네이티브 엔진 생성 실패" }
    }

    fun start() {
        nativeStart(nativeHandle)
    }

    fun stop() {
        nativeStop(nativeHandle)
    }

    override fun close() {
        nativeStop(nativeHandle)
        nativeDestroy(nativeHandle)
    }

    private external fun nativeCreate(): Long
    private external fun nativeStart(handle: Long)
    private external fun nativeStop(handle: Long)
    private external fun nativeDestroy(handle: Long)

    companion object {
        init {
            System.loadLibrary("audio_engine")
        }
    }
}
```

`MainActivity.kt`:

```kotlin
class MainActivity : AppCompatActivity() {

    private val audioController by lazy { AudioEngineController() }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        // RECORD_AUDIO 권한은 입력 스트림에만 필요; 출력은 권한 불필요
        binding.btnStart.setOnClickListener { audioController.start() }
        binding.btnStop.setOnClickListener { audioController.stop() }
    }

    override fun onDestroy() {
        super.onDestroy()
        audioController.close()
    }
}
```

---

## 버퍼 크기 튜닝과 더블 버퍼링

Oboe/AAudio에서 버퍼 크기는 레이턴시에 직접적인 영향을 미칩니다.

- **버퍼가 너무 작으면** → 언더런(underrun) 발생, 오디오 끊김
- **버퍼가 너무 크면** → 레이턴시 증가

`framesPerBurst`(하드웨어가 한 번에 처리하는 프레임 수)의 배수로 설정하는 것이 최적입니다. 더블 버퍼링은 가장 안정적인 선택입니다:

```cpp
// 스트림 열기 후 버퍼 크기 조정
int32_t framesPerBurst = mStream->getFramesPerBurst();
// 더블 버퍼: 최저 레이턴시와 안정성의 균형
mStream->setBufferSizeInFrames(framesPerBurst * 2);

LOGI("버스트 크기: %d 프레임", framesPerBurst);
LOGI("설정된 버퍼: %d 프레임", mStream->getBufferSizeInFrames());
```

Pixel 7 기준 `framesPerBurst`는 96~192프레임(48kHz에서 2~4ms)이므로, 더블 버퍼링 시 약 4~8ms의 레이턴시를 달성할 수 있습니다.

---

## 오류 처리와 스트림 재시작

오디오 스트림은 헤드셋 연결/해제, 통화 시작 등의 이벤트로 끊길 수 있습니다. `AudioStreamErrorCallback`을 구현해 자동 재시작 로직을 추가해야 합니다:

```cpp
class AudioEngine : public oboe::AudioStreamDataCallback,
                    public oboe::AudioStreamErrorCallback {
public:
    // 스트림 오류 시 자동 호출됨
    void onErrorAfterClose(
            oboe::AudioStream* oboeStream,
            oboe::Result error) override {

        if (error == oboe::Result::ErrorDisconnected) {
            // 오디오 기기 변경 → 스트림 재시작
            start();
        }
    }
};
```

빌더에 오류 콜백도 등록:

```cpp
builder
    .setDataCallback(this)
    .setErrorCallback(this);  // 오류 콜백 추가
```

---

## 주의사항과 실전 팁

### 1. 네이티브 샘플레이트 사용
48000Hz를 직접 지정하기보다 `AudioManager`에서 기기의 네이티브 샘플레이트를 쿼리해 설정하면 내부 샘플레이트 변환(SRC)을 피할 수 있습니다:

```kotlin
val audioManager = getSystemService(AUDIO_SERVICE) as AudioManager
val nativeSampleRate = audioManager
    .getProperty(AudioManager.PROPERTY_OUTPUT_SAMPLE_RATE)
    ?.toIntOrNull() ?: 48000
// 이 값을 C++ 쪽으로 전달해 빌더에 설정
```

### 2. EXCLUSIVE 모드 폴백 처리
EXCLUSIVE 모드가 거부될 수 있습니다. Oboe는 자동으로 SHARED로 폴백하지만, 수동으로 처리할 경우:

```cpp
if (result == oboe::Result::ErrorUnavailable) {
    builder.setSharingMode(oboe::SharingMode::Shared);
    builder.openManagedStream(mStream);
}
```

### 3. Oboe 버전 관리
`find_package(oboe)` 사용 시 AGP와 Oboe 버전 호환성을 확인하세요. 2026년 현재 Oboe 1.9.x와 AGP 8.x 조합이 안정적입니다.

### 4. 레이턴시 측정
실제 레이턴시를 측정할 때는 Android 공식 [Audio Latency Tester](https://play.google.com/store/apps/details?id=com.mantz_it.rfanalyzer) 또는 Oboe의 내장 측정 API를 사용하세요:

```cpp
auto latencyResult = mStream->calculateLatencyMillis();
if (latencyResult) {
    LOGI("측정된 레이턴시: %.1f ms", latencyResult.value());
}
```

### 5. 프로세스 우선순위
`SCHED_FIFO` 또는 `THREAD_PRIORITY_URGENT_AUDIO` 설정은 Oboe가 내부적으로 처리합니다. 직접 변경하지 마세요.

---

## 정리

| 항목 | 일반 AudioTrack | AAudio (Oboe) |
|------|----------------|---------------|
| 최소 레이턴시 | ~100ms | ~5ms |
| 언어 | Java/Kotlin | C++ (JNI 래핑) |
| API 레벨 | 전 버전 | API 16+ (Oboe) |
| GC 영향 | 있음 | 없음 (네이티브) |
| 기기 호환성 | 높음 | Oboe QuirksManager로 보완 |

저지연 오디오는 NDK와 C++에 대한 기초 지식을 요구하지만, Oboe가 복잡한 부분을 대부분 처리해 줍니다. 게임 효과음, 음악 메트로놈, 실시간 악기 앱을 개발한다면 Oboe는 사실상 필수 선택입니다.

---

## 참고 자료

- [AAudio 공식 가이드 - Android NDK Developers](https://developer.android.com/ndk/guides/audio/aaudio/aaudio)
- [Oboe 라이브러리 공식 문서 - Android Game Development](https://developer.android.com/games/sdk/oboe)
- [저지연 오디오 체크리스트 - Android Developers](https://developer.android.com/games/sdk/oboe/low-latency-audio)
