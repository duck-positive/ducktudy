---
layout: post
title: "Android LiteRT(구 TensorFlow Lite) 심화: 온디바이스 AI 추론, GPU 가속 델리게이트, 모델 양자화 완전 정복"
date: 2026-10-30
categories: [android, flutter]
tags: [android, litert, tensorflow-lite, on-device-ai, gpu-delegate, quantization, kotlin, machine-learning]
---

## 개요

스마트폰이 단순한 통신 기기를 넘어 AI 엣지 컴퓨팅 플랫폼으로 진화하면서, 클라우드 서버에 의존하지 않고 **기기 내에서 직접 딥러닝 추론(On-Device Inference)** 을 수행하는 수요가 급격히 증가하고 있습니다. Google은 2024년 말 TensorFlow Lite를 **LiteRT(Light Runtime)** 로 리브랜딩하고 V2 아키텍처를 발표했습니다. 기존 TFLite의 인터프리터 기반 API는 유지 보수 모드에 진입하였으며, LiteRT의 **Compiled Model API** 와 **자동 가속기 선택(Automatic Accelerator Selection)** 이 공식 권장 방향입니다.

이 글에서는 Android 앱에서 LiteRT를 사용해 온디바이스 AI 추론을 구현하는 방법을 처음부터 끝까지 완전하게 다룹니다.

---

## 왜 온디바이스 추론이 필요한가?

서버 기반 AI는 강력하지만 치명적인 단점이 있습니다.

| 구분 | 서버 기반 추론 | 온디바이스 추론 |
|------|--------------|----------------|
| 응답 속도 | 네트워크 RTT 포함 (수백ms~수초) | 수 ms~수십 ms |
| 인터넷 연결 | 필수 | 불필요 |
| 개인정보 보호 | 데이터가 서버로 전송됨 | 기기 내에서만 처리 |
| 비용 | API 호출 비용 발생 | 초기 모델 크기만 부담 |
| 보안 | 데이터 전송 위험 | 로컬 처리로 노출 없음 |

사진 필터, 텍스트 분류, 포즈 추정, 실시간 객체 감지 등 레이턴시와 프라이버시가 중요한 기능은 온디바이스 추론이 압도적으로 유리합니다.

---

## LiteRT 아키텍처 이해

### TFLite → LiteRT 변천사

```
TensorFlow Lite (2017~2024)
   └── 인터프리터 기반 API
   └── 수동 델리게이트 설정 (GPU, NNAPI, Hexagon)
   
LiteRT V2 (2025~)
   └── Compiled Model API (자동 가속기 선택)
   └── ML Drift (통합 GPU 레이어)
   └── NPU 통합 지원
   └── LiteRT-LM (LLM 특화)
```

### 핵심 컴포넌트

- **LiteRT 런타임**: CPU/GPU/NPU에서 `.tflite` 또는 LiteRT 컴파일된 모델을 실행
- **GPU 델리게이트**: OpenGL ES / Vulkan 기반 GPU 가속
- **NNAPI 델리게이트**: Android Neural Networks API를 통한 하드웨어 가속
- **Hexagon 델리게이트**: Qualcomm DSP 가속 (특정 Snapdragon 한정)
- **모델 양자화**: FP32 → INT8 변환으로 모델 크기·속도 최적화

---

## 개발 환경 설정

### Gradle 의존성

```kotlin
// build.gradle.kts (app)
dependencies {
    // LiteRT 코어 (Google Play Services 연동)
    implementation("com.google.android.gms:play-services-tflite-java:16.4.0")
    // GPU 델리게이트 (Play Services 경로)
    implementation("com.google.android.gms:play-services-tflite-gpu:16.3.0")
    
    // 또는: 독립 번들(Play Services 없이 실행 가능)
    // implementation("org.tensorflow:tensorflow-lite:2.16.1")
    // implementation("org.tensorflow:tensorflow-lite-gpu:2.16.1")
    
    // GPU 델리게이트 호환성 체크 유틸
    implementation("org.tensorflow:tensorflow-lite-gpu-delegate-plugin:0.4.4")
    
    // 메타데이터 지원 (입출력 텐서 정보 자동 파싱)
    implementation("org.tensorflow:tensorflow-lite-support:0.4.4")
}
```

> **중요**: LiteRT V2의 Compiled Model API는 현재 `com.google.android.gms:play-services-litert`로 별도 아티팩트로 제공되고 있으며, 기존 `tensorflow-lite` 아티팩트와 혼용하지 않도록 주의하세요.

### AndroidManifest.xml

```xml
<!-- 모델 파일이 assets에 있을 때 압축 비활성화 필수 -->
<application
    android:extractNativeLibs="false"
    ...>
```

---

## 구현 예제 1: 이미지 분류 (MobileNetV3 + GPU 델리게이트)

다음 예제는 사전 훈련된 MobileNetV3 모델로 카메라 프레임을 실시간 분류하는 완전한 구현입니다.

### 모델 로드 및 인터프리터 초기화

```kotlin
import org.tensorflow.lite.Interpreter
import org.tensorflow.lite.gpu.CompatibilityList
import org.tensorflow.lite.gpu.GpuDelegate
import android.content.Context
import java.nio.ByteBuffer
import java.nio.MappedByteBuffer
import java.nio.channels.FileChannel

class ImageClassifier(private val context: Context) : AutoCloseable {

    private val interpreter: Interpreter
    private val gpuDelegate: GpuDelegate?
    private val inputBuffer: ByteBuffer

    // MobileNetV3 입력: 224×224 RGB, FP32 (각 픽셀 4바이트)
    private val INPUT_SIZE = 224
    private val NUM_CHANNELS = 3
    private val FLOAT_BYTES = 4

    init {
        val compatList = CompatibilityList()
        val options = Interpreter.Options()

        // GPU 지원 여부 런타임 확인
        gpuDelegate = if (compatList.isDelegateSupportedOnThisDevice) {
            val delegateOptions = compatList.bestOptionsForThisDevice
            GpuDelegate(delegateOptions).also { options.addDelegate(it) }
        } else {
            // CPU fallback: 스레드 수를 물리 코어 수에 맞게 설정
            options.numThreads = Runtime.getRuntime().availableProcessors()
            null
        }

        val modelBuffer = loadModelFromAssets("mobilenet_v3_small_100_224_classification.tflite")
        interpreter = Interpreter(modelBuffer, options)

        // 입력 버퍼 미리 할당 (매 추론마다 재할당 방지)
        val inputSize = INPUT_SIZE * INPUT_SIZE * NUM_CHANNELS * FLOAT_BYTES
        inputBuffer = ByteBuffer.allocateDirect(inputSize).apply {
            order(java.nio.ByteOrder.nativeOrder())
        }
    }

    private fun loadModelFromAssets(filename: String): MappedByteBuffer {
        val assetFd = context.assets.openFd(filename)
        return assetFd.createInputStream().channel.map(
            FileChannel.MapMode.READ_ONLY,
            assetFd.startOffset,
            assetFd.declaredLength
        )
    }

    /**
     * Bitmap을 받아 상위 K개 분류 결과를 반환합니다.
     * @param bitmap 224×224로 리사이즈된 입력 이미지
     * @param topK 반환할 결과 개수
     */
    fun classify(bitmap: android.graphics.Bitmap, topK: Int = 5): List<ClassificationResult> {
        preprocessBitmap(bitmap)

        // 출력 버퍼: MobileNetV3은 1000개 클래스
        val outputBuffer = Array(1) { FloatArray(1000) }
        interpreter.run(inputBuffer, outputBuffer)

        return outputBuffer[0]
            .mapIndexed { index, score -> ClassificationResult(index, score) }
            .sortedByDescending { it.score }
            .take(topK)
    }

    private fun preprocessBitmap(bitmap: android.graphics.Bitmap) {
        inputBuffer.rewind()
        // ImageNet 정규화: (픽셀/255.0 - mean) / std
        val mean = floatArrayOf(0.485f, 0.456f, 0.406f)
        val std  = floatArrayOf(0.229f, 0.224f, 0.225f)

        for (y in 0 until INPUT_SIZE) {
            for (x in 0 until INPUT_SIZE) {
                val pixel = bitmap.getPixel(x, y)
                inputBuffer.putFloat(((pixel shr 16 and 0xFF) / 255f - mean[0]) / std[0]) // R
                inputBuffer.putFloat(((pixel shr 8  and 0xFF) / 255f - mean[1]) / std[1]) // G
                inputBuffer.putFloat(((pixel        and 0xFF) / 255f - mean[2]) / std[2]) // B
            }
        }
    }

    override fun close() {
        interpreter.close()
        gpuDelegate?.close()
    }

    data class ClassificationResult(val classIndex: Int, val score: Float)
}
```

### 실제 사용 예 (ViewModel)

```kotlin
class ClassifierViewModel(application: Application) : AndroidViewModel(application) {

    private val classifier by lazy { ImageClassifier(application) }

    private val _results = MutableStateFlow<List<ImageClassifier.ClassificationResult>>(emptyList())
    val results: StateFlow<List<ImageClassifier.ClassificationResult>> = _results.asStateFlow()

    fun runClassification(bitmap: Bitmap) {
        viewModelScope.launch(Dispatchers.Default) {
            // 추론은 Default 디스패처(CPU 집중 스레드)에서 실행
            val scaled = Bitmap.createScaledBitmap(bitmap, 224, 224, true)
            val top5 = classifier.classify(scaled)
            _results.value = top5
        }
    }

    override fun onCleared() {
        super.onCleared()
        classifier.close()
    }
}
```

---

## 구현 예제 2: 커스텀 텍스트 분류 (TFLite Support + 양자화 모델)

실제 프로덕션에서는 `tensorflow-lite-support` 라이브러리가 제공하는 `TextClassifier`나 `ImageClassifier` 헬퍼를 활용하면 전처리 파이프라인을 대폭 간소화할 수 있습니다.

```kotlin
import org.tensorflow.lite.support.text.TextClassifier
import org.tensorflow.lite.support.label.Category
import org.tensorflow.lite.task.core.BaseOptions

class SentimentAnalyzer(context: Context) : AutoCloseable {

    private val textClassifier: TextClassifier

    init {
        val baseOptions = BaseOptions.builder()
            .useGpu()           // GPU 가속 요청 (지원 시 자동 활성화)
            .setNumThreads(4)   // CPU fallback 스레드 수
            .build()

        val options = TextClassifier.TextClassifierOptions.builder()
            .setBaseOptions(baseOptions)
            .setMaxResults(2)
            .build()

        // assets/sentiment_classifier.tflite 로드
        textClassifier = TextClassifier.createFromFileAndOptions(
            context, "sentiment_classifier.tflite", options
        )
    }

    /**
     * 긍정/부정 감성 분류. Category.score가 높을수록 확률 높음.
     */
    fun analyze(text: String): List<Category> {
        val result = textClassifier.classify(text)
        // result.classificationResults[0].categories는 내부적으로 정렬된 상태
        return result.classificationResults.firstOrNull()?.categories ?: emptyList()
    }

    override fun close() = textClassifier.close()
}

// 사용 예
val analyzer = SentimentAnalyzer(context)
val categories = analyzer.analyze("이 앱 정말 최고예요!")
categories.forEach { category ->
    Log.d("Sentiment", "${category.label}: ${"%.2f".format(category.score * 100)}%")
}
// 출력: positive: 97.34%
//       negative: 2.66%
```

---

## 모델 양자화 (Quantization) 심화

양자화는 모델 가중치·활성값을 FP32에서 더 낮은 비트폭(INT8, FP16)으로 변환하여 모델 크기와 추론 속도를 줄이는 기법입니다.

### 양자화 방식 비교

| 방식 | 크기 감소 | 정확도 손실 | 추가 학습 필요 |
|------|-----------|------------|--------------|
| Dynamic Range Quantization | ~4× | 최소 | 불필요 |
| Full Integer Quantization | ~4× | 중간 | 불필요 (캘리브레이션 필요) |
| FP16 Quantization | ~2× | 거의 없음 | 불필요 |
| Quantization-Aware Training (QAT) | ~4× | 최소 | 필요 |

### Python 변환 스크립트 (참고용)

```python
import tensorflow as tf

# 1. SavedModel 또는 Keras 모델 불러오기
model = tf.saved_model.load("my_model")

# 2. TFLite 변환기 초기화
converter = tf.lite.TFLiteConverter.from_saved_model("my_model")

# 3. 동적 범위 양자화 (가장 빠르고 간단)
converter.optimizations = [tf.lite.Optimize.DEFAULT]

# 4. Full Integer Quantization (캘리브레이션 데이터 필요)
def representative_dataset():
    for sample in calibration_samples:
        yield [sample.astype("float32")]

converter.representative_dataset = representative_dataset
converter.target_spec.supported_ops = [tf.lite.OpsSet.TFLITE_BUILTINS_INT8]
converter.inference_input_type  = tf.int8
converter.inference_output_type = tf.int8

# 5. 변환 및 저장
tflite_model = converter.convert()
with open("model_quantized.tflite", "wb") as f:
    f.write(tflite_model)
```

### 양자화 전후 비교 (MobileNetV3 기준)

| 지표 | FP32 | INT8 |
|------|------|------|
| 모델 크기 | 16 MB | 4 MB |
| Pixel 6 추론 시간 | 45 ms | 12 ms |
| Top-1 정확도 (ImageNet) | 75.2% | 74.8% |

---

## GPU 델리게이트 심화

### CompatibilityList 활용 패턴

```kotlin
val compatList = CompatibilityList()

val options = if (compatList.isDelegateSupportedOnThisDevice) {
    // 기기에 최적화된 델리게이트 옵션 자동 선택
    val delegateOptions = compatList.bestOptionsForThisDevice
    Interpreter.Options().apply {
        addDelegate(GpuDelegate(delegateOptions))
    }
} else {
    Interpreter.Options().apply {
        numThreads = 4
        // NNAPI 델리게이트 시도 (API 27+)
        if (android.os.Build.VERSION.SDK_INT >= 27) {
            setUseNNAPI(true)
        }
    }
}
```

### GPU 델리게이트 제약사항

- 양자화 모델(INT8)을 GPU에서 실행하려면 `GpuDelegateOptions.quantizedModelsAllowed = true` 설정 필요
- 동적 형태(Dynamic Shape) 텐서는 GPU 델리게이트에서 지원하지 않음
- 모델 내 일부 Op이 GPU를 지원하지 않으면 해당 Op만 CPU로 fallback됨

---

## 주의사항 및 실전 팁

### 1. 모델 파일 압축 방지

`assets/` 폴더의 `.tflite` 파일은 이미 압축되어 있으므로, `aaptOptions`에서 재압축을 막아야 MappedByteBuffer로 직접 메모리 매핑할 수 있습니다.

```kotlin
// build.gradle.kts
android {
    aaptOptions {
        noCompress += "tflite"
    }
}
```

### 2. 추론 스레드 분리

메인 스레드에서 추론을 실행하면 UI 프리징이 발생합니다. 반드시 `Dispatchers.Default` 또는 전용 스레드에서 실행하세요.

```kotlin
// ❌ 잘못된 예
lifecycleScope.launch {
    val result = classifier.classify(bitmap) // UI 스레드에서 실행 → ANR 위험
}

// ✅ 올바른 예
lifecycleScope.launch(Dispatchers.Default) {
    val result = classifier.classify(bitmap)
    withContext(Dispatchers.Main) {
        updateUI(result)
    }
}
```

### 3. 인터프리터 재사용

`Interpreter` 생성은 비용이 크므로, 앱 생명주기 동안 단일 인스턴스를 재사용하고 `AutoCloseable`로 적절히 해제하세요.

### 4. 모델 크기와 앱 크기

- `.tflite` 파일은 `assets/`에 포함하거나, Android App Bundle의 **on-demand delivery** 를 활용해 필요 시 다운로드하는 방식으로 앱 초기 크기를 줄일 수 있습니다.
- [Play Feature Delivery](https://developer.android.com/guide/playcore/feature-delivery)와 결합하면 모델 파일을 별도 동적 기능 모듈로 분리 가능합니다.

### 5. 모델 서명 및 보안

`.tflite` 파일 자체는 암호화되지 않으므로, 민감한 독자적 모델이라면 저장 시 AES 암호화 후 런타임에 복호화하여 `ByteBuffer`로 전달하는 방식을 고려하세요.

### 6. LiteRT 마이그레이션 체크리스트

- [ ] `import org.tensorflow.lite.*` → `import com.google.android.gms.tflite.*` (Play Services 경로)
- [ ] 수동 `GpuDelegate` → `BaseOptions.useGpu()` (Task Library 사용 시)
- [ ] `Interpreter` 직접 사용 → `ImageClassifier` / `TextClassifier` 헬퍼 클래스
- [ ] `setUseNNAPI(true)` → `BaseOptions.useNnapi()` 또는 자동 가속기 선택

---

## 성능 측정: Android GPU Inspector / Perfetto

실제 추론 성능을 측정하려면 Android GPU Inspector(AGI) 또는 `android.os.Trace`를 활용하세요.

```kotlin
android.os.Trace.beginSection("LiteRT Inference")
try {
    val result = classifier.classify(bitmap)
} finally {
    android.os.Trace.endSection()
}
```

Perfetto 트레이스로 섹션을 열면 GPU 실행 타임라인과 CPU 버블을 함께 분석할 수 있습니다.

---

## 마치며

LiteRT(구 TFLite)는 **온디바이스 AI**의 사실상 표준으로 자리잡은 프레임워크입니다. 오늘 다룬 핵심 요약은 다음과 같습니다.

1. **GPU 델리게이트**: `CompatibilityList`로 런타임에 GPU 지원 여부를 확인하고, 지원 시 자동으로 GPU 가속을 활성화하세요.
2. **모델 양자화**: INT8 양자화로 모델을 4× 압축하고 추론 속도를 3~4배 향상시킬 수 있습니다. 정확도 손실을 측정 후 QAT를 적용하세요.
3. **Task Library 활용**: `ImageClassifier`, `TextClassifier` 등 헬퍼 클래스를 사용하면 전처리·후처리를 표준화하고 코드량을 크게 줄일 수 있습니다.
4. **LiteRT V2 준비**: TFLite 인터프리터 API는 유지 보수 모드이므로, 신규 프로젝트는 LiteRT Compiled Model API와 Play Services 경로를 채택하는 것이 장기적으로 유리합니다.

온디바이스 AI는 단순한 추세가 아니라, 응답속도·프라이버시·오프라인 지원의 세 가지 이점을 동시에 제공하는 구조적 해결책입니다. Android 앱에 AI 기능을 도입할 때 서버 비용과 레이턴시 문제를 겪고 있다면, LiteRT 온디바이스 추론을 가장 먼저 검토해 보세요.

---

## 참고 자료

- [LiteRT GitHub Repository (google-ai-edge/LiteRT)](https://github.com/google-ai-edge/LiteRT)
- [TensorFlow Lite Examples Repository (tensorflow/examples)](https://github.com/tensorflow/examples/tree/master/lite/examples)
- [TensorFlow Lite GPU Delegate Guide (tensorflow.org)](https://www.tensorflow.org/lite/android/delegates/gpu)
- [Android ML & AI — developer.android.com](https://developer.android.com/ml)
