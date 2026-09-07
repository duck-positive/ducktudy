---
layout: post
title: "Android ML Kit 심화: CameraX와 함께하는 온디바이스 텍스트 인식 & 바코드 스캔"
date: 2026-09-07
categories: [android, flutter]
tags: [android, mlkit, camerax, text-recognition, barcode, ocr, on-device-ml]
---

스마트폰 카메라를 활용한 실시간 텍스트 인식(OCR)과 바코드 스캔은 현대 앱에서 점점 필수 기능이 되고 있습니다. Google의 **ML Kit**는 이러한 온디바이스 머신러닝 기능을 별도의 모델 학습 없이 손쉽게 앱에 통합할 수 있는 강력한 SDK입니다. 이번 포스트에서는 CameraX와 ML Kit를 결합해 실시간 텍스트 인식과 바코드 스캔을 구현하는 심화 방법을 다룹니다.

## ML Kit란 무엇인가?

ML Kit는 Google이 제공하는 모바일용 머신러닝 SDK로, Android와 iOS 모두를 지원합니다. 이미 학습된 고품질 ML 모델을 앱에 직접 번들링하거나, Google Play Services를 통해 경량 방식으로 사용할 수 있습니다.

ML Kit가 제공하는 주요 Vision API는 다음과 같습니다.

- **Text Recognition v2**: 라틴 문자는 물론 한글, 일본어, 중국어, 데바나가리 문자까지 지원
- **Barcode Scanning**: QR코드, EAN-13, UPC-A 등 수십 가지 바코드 형식 지원
- **Face Detection**: 얼굴 위치, 랜드마크, 표정 분류
- **Object Detection & Tracking**: 실시간 객체 인식 및 추적
- **Image Labeling**: 이미지 내 사물 분류

ML Kit는 **번들형(Bundled)**과 **언번들형(Unbundled)** 두 가지 배포 방식을 지원합니다. 번들형은 모델이 APK 안에 포함되어 오프라인에서도 즉시 동작하지만 앱 크기가 증가합니다. 언번들형은 Google Play Services에서 모델을 다운로드해 앱 크기가 최소화되지만 최초 실행 시 다운로드가 필요합니다.

## 왜 ML Kit + CameraX 조합인가?

기존 Camera API는 복잡한 생명주기 관리와 보일러플레이트 코드가 많았습니다. CameraX는 이를 해결한 Jetpack 라이브러리로, `ImageAnalysis` 유스케이스를 통해 카메라 프레임을 ML Kit에 직접 전달하는 파이프라인을 매우 간결하게 구성할 수 있습니다.

두 라이브러리를 조합하면 다음과 같은 장점이 생깁니다.

1. **생명주기 자동 관리**: `ProcessCameraProvider`가 Activity/Fragment 생명주기와 연동
2. **BackpressureStrategy**: 분석 처리 속도보다 카메라 프레임이 빠를 때 자동으로 프레임을 드롭해 메모리 안정성 확보
3. **ImageProxy 직접 전달**: ML Kit의 `InputImage.fromMediaImage()`로 변환 오버헤드 최소화

## 의존성 설정

`build.gradle.kts` (앱 모듈)에 다음을 추가합니다.

```kotlin
dependencies {
    // CameraX
    val cameraxVersion = "1.4.0"
    implementation("androidx.camera:camera-core:$cameraxVersion")
    implementation("androidx.camera:camera-camera2:$cameraxVersion")
    implementation("androidx.camera:camera-lifecycle:$cameraxVersion")
    implementation("androidx.camera:camera-view:$cameraxVersion")

    // ML Kit - 언번들형 (Google Play Services 기반, 앱 크기 최소화)
    implementation("com.google.android.gms:play-services-mlkit-text-recognition:19.0.1")
    implementation("com.google.android.gms:play-services-mlkit-text-recognition-korean:16.0.1")
    implementation("com.google.android.gms:play-services-mlkit-barcode-scanning:18.3.1")
}
```

카메라 권한을 `AndroidManifest.xml`에 추가하는 것도 잊지 마세요.

```xml
<uses-permission android:name="android.permission.CAMERA" />
<uses-feature android:name="android.hardware.camera.autofocus" />
```

---

## 실제 구현 예제 1: 실시간 텍스트 인식 (OCR)

아래는 CameraX의 `ImageAnalysis`와 ML Kit `TextRecognizer`를 결합한 완전한 구현입니다.

```kotlin
class TextRecognitionAnalyzer(
    private val onTextDetected: (String) -> Unit
) : ImageAnalysis.Analyzer {

    private val recognizer = TextRecognition.getClient(
        TextRecognizerOptions.DEFAULT_OPTIONS
    )
    // 처리 중 플래그로 프레임 중복 처리 방지
    private var isProcessing = AtomicBoolean(false)

    @androidx.camera.core.ExperimentalGetImage
    override fun analyze(imageProxy: ImageProxy) {
        if (isProcessing.get()) {
            imageProxy.close()
            return
        }
        val mediaImage = imageProxy.image ?: run {
            imageProxy.close()
            return
        }

        isProcessing.set(true)
        val inputImage = InputImage.fromMediaImage(
            mediaImage,
            imageProxy.imageInfo.rotationDegrees
        )

        recognizer.process(inputImage)
            .addOnSuccessListener { visionText ->
                val detectedText = visionText.textBlocks
                    .joinToString("\n") { block ->
                        block.lines.joinToString(" ") { it.text }
                    }
                if (detectedText.isNotBlank()) {
                    onTextDetected(detectedText)
                }
            }
            .addOnFailureListener { e ->
                Log.e("TextRecognition", "인식 실패", e)
            }
            .addOnCompleteListener {
                isProcessing.set(false)
                imageProxy.close()
            }
    }
}

// Activity 또는 Fragment에서 카메라 바인딩
class MainActivity : AppCompatActivity() {

    private lateinit var cameraProviderFuture: ListenableFuture<ProcessCameraProvider>

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(binding.root)
        startCamera()
    }

    private fun startCamera() {
        cameraProviderFuture = ProcessCameraProvider.getInstance(this)
        cameraProviderFuture.addListener({
            val cameraProvider = cameraProviderFuture.get()

            val preview = Preview.Builder().build().also {
                it.setSurfaceProvider(binding.previewView.surfaceProvider)
            }

            val imageAnalysis = ImageAnalysis.Builder()
                .setTargetResolution(Size(1280, 720))
                .setBackpressureStrategy(ImageAnalysis.STRATEGY_KEEP_ONLY_LATEST)
                .build()
                .also {
                    it.setAnalyzer(
                        ContextCompat.getMainExecutor(this),
                        TextRecognitionAnalyzer { text ->
                            binding.tvResult.text = text
                        }
                    )
                }

            val cameraSelector = CameraSelector.DEFAULT_BACK_CAMERA

            try {
                cameraProvider.unbindAll()
                cameraProvider.bindToLifecycle(
                    this, cameraSelector, preview, imageAnalysis
                )
            } catch (e: Exception) {
                Log.e("CameraX", "바인딩 실패", e)
            }
        }, ContextCompat.getMainExecutor(this))
    }
}
```

`TextBlock → Line → Element` 계층 구조로 인식 결과가 반환되므로, 필요에 따라 각 블록의 `boundingBox` 좌표를 활용해 UI에 오버레이를 그릴 수도 있습니다.

---

## 실제 구현 예제 2: 실시간 바코드 스캔 (QR 포함)

바코드 스캔은 스캔 성공 후 처리를 중단하는 "원샷(One-shot)" 패턴이 일반적입니다.

```kotlin
class BarcodeScanningAnalyzer(
    private val onBarcodeDetected: (List<Barcode>) -> Unit
) : ImageAnalysis.Analyzer {

    private val options = BarcodeScannerOptions.Builder()
        .setBarcodeFormats(
            Barcode.FORMAT_QR_CODE,
            Barcode.FORMAT_EAN_13,
            Barcode.FORMAT_EAN_8,
            Barcode.FORMAT_CODE_128,
            Barcode.FORMAT_UPC_A,
            Barcode.FORMAT_DATA_MATRIX
        )
        .build()

    private val scanner = BarcodeScanning.getClient(options)
    private var isScanComplete = AtomicBoolean(false)

    @androidx.camera.core.ExperimentalGetImage
    override fun analyze(imageProxy: ImageProxy) {
        if (isScanComplete.get()) {
            imageProxy.close()
            return
        }
        val mediaImage = imageProxy.image ?: run {
            imageProxy.close()
            return
        }

        val inputImage = InputImage.fromMediaImage(
            mediaImage,
            imageProxy.imageInfo.rotationDegrees
        )

        scanner.process(inputImage)
            .addOnSuccessListener { barcodes ->
                if (barcodes.isNotEmpty()) {
                    isScanComplete.set(true)
                    onBarcodeDetected(barcodes)
                }
            }
            .addOnFailureListener { e ->
                Log.e("BarcodeScanning", "스캔 실패", e)
            }
            .addOnCompleteListener {
                imageProxy.close()
            }
    }

    fun reset() {
        isScanComplete.set(false)
    }
}

// 바코드 결과 처리 예시
private fun handleBarcodes(barcodes: List<Barcode>) {
    barcodes.forEach { barcode ->
        when (barcode.valueType) {
            Barcode.TYPE_URL -> {
                val url = barcode.url?.url
                Log.d("Barcode", "URL: $url")
            }
            Barcode.TYPE_CONTACT_INFO -> {
                val contact = barcode.contactInfo
                Log.d("Barcode", "연락처: ${contact?.name?.formattedName}")
            }
            Barcode.TYPE_WIFI -> {
                val wifi = barcode.wifi
                Log.d("Barcode", "SSID: ${wifi?.ssid}, PW: ${wifi?.password}")
            }
            Barcode.TYPE_TEXT -> {
                Log.d("Barcode", "텍스트: ${barcode.rawValue}")
            }
            else -> {
                Log.d("Barcode", "원시값: ${barcode.rawValue}")
                Log.d("Barcode", "포맷: ${barcode.format}")
            }
        }
    }
}
```

---

## 주의사항 & 고급 팁

### 1. 언번들형 모델 다운로드 대기 처리

언번들형 ML Kit를 사용하면 앱 최초 실행 시 모델 다운로드가 발생합니다. `ModuleInstallClient`를 활용해 미리 다운로드하거나 상태를 확인하세요.

```kotlin
val moduleInstallClient = ModuleInstall.getClient(context)
val optionalModuleApi = OptionalModuleApi()
moduleInstallClient
    .areModulesAvailable(optionalModuleApi)
    .addOnSuccessListener { response ->
        if (!response.areModulesAvailable()) {
            // 미리 설치 요청
            moduleInstallClient.installModules(
                ModuleInstallRequest.newBuilder()
                    .addApis(optionalModuleApi)
                    .build()
            )
        }
    }
```

### 2. 카메라 해상도와 인식 정확도 트레이드오프

`setTargetResolution`을 너무 높게 설정하면 ML Kit 분석 속도가 느려져 실시간 처리에 지연이 발생합니다. 텍스트 인식은 `1280×720`, 바코드 스캔은 `640×480`이 일반적으로 최적의 균형점입니다.

### 3. ImageProxy 클로즈 보장

`addOnCompleteListener`에서 반드시 `imageProxy.close()`를 호출해야 합니다. 이를 빠뜨리면 카메라 세션이 새 프레임을 제공하지 않아 앱이 멈춰 보이는 현상이 발생합니다.

### 4. 텍스트 인식 결과 필터링

`TextBlock`은 언어, 신뢰도(`confidence`) 등의 메타데이터를 제공합니다. 낮은 신뢰도의 블록을 걸러내거나 특정 패턴(이메일, 전화번호 등)만 추출하려면 결과에 정규식을 적용하세요.

### 5. `@ExperimentalGetImage` 어노테이션

`imageProxy.image`에 접근하려면 `@androidx.camera.core.ExperimentalGetImage` 어노테이션이 필요합니다. 안정 API가 아니므로 향후 변경될 수 있으나, 현재로선 ML Kit와 CameraX를 연동하는 가장 효율적인 방법입니다.

---

## 마무리

ML Kit와 CameraX는 복잡한 ML 파이프라인을 놀랍도록 간결하게 만들어 줍니다. 텍스트 인식의 경우 `TextBlock` 계층을 깊이 파고들면 단어 단위 바운딩 박스를 활용한 AR 오버레이, 번역 기능 연동 등 다양한 확장이 가능합니다. 바코드 스캔은 `Barcode.valueType`을 통해 URL, Wi-Fi 정보, 연락처 등 다양한 타입을 구조적으로 처리할 수 있어 실무 앱에 즉시 적용 가능한 완성도를 제공합니다.

온디바이스 처리라는 특성 덕분에 네트워크 없이도 동작하고, 사용자 데이터가 서버로 전송되지 않아 개인정보 보호 측면에서도 큰 장점이 있습니다.

## 참고 자료
- [Google ML Kit 공식 샘플 (googlesamples/mlkit)](https://github.com/googlesamples/mlkit)
- [ML Kit Vision Quickstart (Android)](https://github.com/googlesamples/mlkit/tree/master/android/vision-quickstart)
