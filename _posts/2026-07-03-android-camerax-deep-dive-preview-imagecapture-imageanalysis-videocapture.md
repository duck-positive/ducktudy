---
layout: post
title: "Android CameraX 심화: PreviewView·ImageCapture·ImageAnalysis·VideoCapture 완전 정복"
date: 2026-07-03
categories: [android]
tags: [android, camerax, imageanalysis, videocapture, mlkit, jetpack, kotlin]
---

카메라 기능은 현대 Android 앱에서 빠질 수 없는 핵심 요소다. 그러나 기존의 Camera1 API는 너무 저수준이었고, Camera2는 강력하지만 보일러플레이트가 방대해 진입 장벽이 높았다. Jetpack CameraX는 이 문제를 해결하기 위해 등장한 고수준 추상화 라이브러리로, 수십 줄의 Camera2 코드를 단 몇 줄로 압축하면서도 기기 호환성과 생명주기 관리를 자동으로 처리한다.

이 글에서는 CameraX의 핵심 아키텍처부터 실전 코드까지, 심화 수준으로 완전히 파고든다.

---

## 1. CameraX란 무엇인가: 아키텍처 개요

CameraX는 세 가지 핵심 계층으로 구성된다.

**Camera Provider (ProcessCameraProvider)**  
앱의 카메라 접근 진입점이다. 싱글턴으로 동작하며, 내부적으로 Camera2 세션을 관리한다. `ProcessCameraProvider.getInstance(context)`는 `ListenableFuture`를 반환하며, 완료 시점에 카메라가 사용 가능해진다.

**Use Case**  
CameraX의 핵심 추상화 단위다. 각 Use Case는 카메라 출력의 특정 목적을 표현한다.

| Use Case | 용도 |
|---|---|
| `Preview` | 화면에 카메라 미리보기 표시 |
| `ImageCapture` | 고해상도 정지 이미지 캡처 |
| `ImageAnalysis` | CPU가 접근 가능한 프레임 분석 (ML Kit 등) |
| `VideoCapture` | 비디오 + 오디오 녹화 |

**Lifecycle Integration**  
CameraX는 `LifecycleOwner`와 바인딩되어 Activity/Fragment의 생명주기에 따라 카메라 세션을 자동으로 열고 닫는다. 개발자가 `onResume()`/`onPause()`에서 직접 카메라를 관리할 필요가 없다.

```
ProcessCameraProvider
    └── bindToLifecycle(lifecycleOwner, cameraSelector, useCase1, useCase2, ...)
            ├── Preview
            ├── ImageCapture
            ├── ImageAnalysis
            └── VideoCapture<Recorder>
```

---

## 2. 왜 CameraX인가: Camera2와의 비교

Camera2로 단순한 사진 촬영 기능 하나를 구현하려면, CameraManager로 카메라를 열고, CameraDevice.StateCallback을 처리하고, CaptureSession을 구성하고, CaptureRequest를 빌드하고, Surface를 연결하는 수십 줄의 코드가 필요하다. 심지어 기기마다 다른 quirk(기기별 버그)를 직접 처리해야 한다.

CameraX가 해결하는 주요 문제들:

- **기기 호환성**: 2,000개 이상의 Android 기기에서 테스트된 quirk 데이터베이스를 내장해 기기별 버그를 자동으로 우회한다.
- **생명주기 자동 관리**: `bindToLifecycle()` 한 줄로 카메라 세션의 열기/닫기를 자동화한다.
- **Use Case 조합 최적화**: 여러 Use Case를 동시에 바인딩하면 CameraX가 내부적으로 카메라 하드웨어의 스트림 조합 제약을 고려해 최적의 구성을 선택한다.
- **Jetpack 생태계 통합**: ML Kit Analyzer, ExifInterface, Compose와의 통합이 공식 지원된다.

---

## 3. 실제 구현 예제 1: PreviewView + ImageCapture (사진 촬영)

먼저 의존성을 추가한다.

```kotlin
// build.gradle.kts (app)
val cameraxVersion = "1.5.0"
implementation("androidx.camera:camera-core:$cameraxVersion")
implementation("androidx.camera:camera-camera2:$cameraxVersion")
implementation("androidx.camera:camera-lifecycle:$cameraxVersion")
implementation("androidx.camera:camera-view:$cameraxVersion")
```

그리고 레이아웃에 `PreviewView`를 배치한다.

```xml
<androidx.camera.view.PreviewView
    android:id="@+id/previewView"
    android:layout_width="match_parent"
    android:layout_height="match_parent" />
```

다음은 PreviewView와 ImageCapture를 바인딩하고 사진을 촬영하는 전체 구현이다.

```kotlin
class CameraActivity : AppCompatActivity() {

    private lateinit var binding: ActivityCameraBinding
    private lateinit var cameraExecutor: ExecutorService
    private var imageCapture: ImageCapture? = null

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityCameraBinding.inflate(layoutInflater)
        setContentView(binding.root)

        cameraExecutor = Executors.newSingleThreadExecutor()

        // 카메라 권한 확인 후 시작 (권한 요청 로직은 생략)
        startCamera()

        binding.btnCapture.setOnClickListener { takePhoto() }
    }

    private fun startCamera() {
        val cameraProviderFuture = ProcessCameraProvider.getInstance(this)

        cameraProviderFuture.addListener({
            val cameraProvider = cameraProviderFuture.get()

            // 1. Preview Use Case 구성
            val preview = Preview.Builder()
                .build()
                .also { it.setSurfaceProvider(binding.previewView.surfaceProvider) }

            // 2. ImageCapture Use Case 구성
            imageCapture = ImageCapture.Builder()
                .setCaptureMode(ImageCapture.CAPTURE_MODE_MINIMIZE_LATENCY)
                // JPEG 품질 95%, 회전 정보 자동 처리
                .setJpegQuality(95)
                .setTargetRotation(binding.previewView.display.rotation)
                .build()

            // 3. 후면 카메라 선택
            val cameraSelector = CameraSelector.DEFAULT_BACK_CAMERA

            try {
                // 기존 바인딩 해제 후 재바인딩
                cameraProvider.unbindAll()
                val camera = cameraProvider.bindToLifecycle(
                    this,
                    cameraSelector,
                    preview,
                    imageCapture
                )

                // 탭-투-포커스 설정
                val factory = binding.previewView.meteringPointFactory
                binding.previewView.setOnTouchListener { _, event ->
                    val point = factory.createPoint(event.x, event.y)
                    val action = FocusMeteringAction.Builder(point)
                        .setAutoCancelDuration(3, TimeUnit.SECONDS)
                        .build()
                    camera.cameraControl.startFocusAndMetering(action)
                    true
                }

            } catch (e: Exception) {
                Log.e("CameraX", "Use case binding failed", e)
            }

        }, ContextCompat.getMainExecutor(this))
    }

    private fun takePhoto() {
        val imageCapture = imageCapture ?: return

        // MediaStore를 통한 저장 (Android 10+ 권장 방식)
        val contentValues = ContentValues().apply {
            put(MediaStore.MediaColumns.DISPLAY_NAME, "IMG_${System.currentTimeMillis()}")
            put(MediaStore.MediaColumns.MIME_TYPE, "image/jpeg")
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
                put(MediaStore.Images.Media.RELATIVE_PATH, "Pictures/CameraXApp")
            }
        }

        val outputOptions = ImageCapture.OutputFileOptions.Builder(
            contentResolver,
            MediaStore.Images.Media.EXTERNAL_CONTENT_URI,
            contentValues
        ).build()

        imageCapture.takePicture(
            outputOptions,
            cameraExecutor,
            object : ImageCapture.OnImageSavedCallback {
                override fun onImageSaved(output: ImageCapture.OutputFileResults) {
                    val savedUri = output.savedUri
                    runOnUiThread {
                        Toast.makeText(this@CameraActivity, "사진 저장: $savedUri", Toast.LENGTH_SHORT).show()
                    }
                }

                override fun onError(exception: ImageCaptureException) {
                    Log.e("CameraX", "Photo capture failed: ${exception.message}", exception)
                }
            }
        )
    }

    override fun onDestroy() {
        super.onDestroy()
        cameraExecutor.shutdown()
    }
}
```

**핵심 포인트:**

- `CAPTURE_MODE_MINIMIZE_LATENCY`: 속도 우선 모드다. 고화질이 필요하다면 `CAPTURE_MODE_MAXIMIZE_QUALITY`를 사용한다.
- `setTargetRotation()`: 기기 회전에 따라 이미지 방향이 올바르게 저장되도록 한다.
- `FocusMeteringAction`: 탭-투-포커스를 구현하는 공식 API다. `setAutoCancelDuration()`으로 AF가 자동으로 해제되는 시간을 설정한다.

---

## 4. 실제 구현 예제 2: ImageAnalysis + ML Kit 바코드 스캐너

`ImageAnalysis` Use Case는 각 카메라 프레임에 대해 `analyze()` 메서드를 호출한다. ML Kit의 `BarcodeScanning`과 결합하면 실시간 바코드 스캐너를 만들 수 있다.

의존성 추가:

```kotlin
// ML Kit 바코드 스캐닝
implementation("com.google.mlkit:barcode-scanning:17.3.0")
// CameraX + ML Kit 통합 (MlKitAnalyzer)
implementation("androidx.camera:camera-mlkit-vision:1.5.0")
```

`MlKitAnalyzer`를 활용한 실시간 바코드 스캐너 구현:

```kotlin
class BarcodeScannerActivity : AppCompatActivity() {

    private lateinit var binding: ActivityBarcodeScannerBinding
    private lateinit var cameraExecutor: ExecutorService

    // ML Kit 바코드 스캐너 (FORMAT_ALL_FORMATS로 모든 형식 지원)
    private val barcodeScanner = BarcodeScanning.getClient(
        BarcodeScannerOptions.Builder()
            .setBarcodeFormats(Barcode.FORMAT_QR_CODE, Barcode.FORMAT_EAN_13)
            .build()
    )

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityBarcodeScannerBinding.inflate(layoutInflater)
        setContentView(binding.root)
        cameraExecutor = Executors.newSingleThreadExecutor()
        startBarcodeScanner()
    }

    private fun startBarcodeScanner() {
        val cameraProviderFuture = ProcessCameraProvider.getInstance(this)

        cameraProviderFuture.addListener({
            val cameraProvider = cameraProviderFuture.get()

            // MlKitAnalyzer: CameraX와 ML Kit 통합의 공식 방법
            // CameraController와 함께 사용하면 좌표 변환을 자동 처리한다
            val analyzer = MlKitAnalyzer(
                listOf(barcodeScanner),          // 분석기 목록
                COORDINATE_SYSTEM_VIEW_REFERENCED, // PreviewView 좌표계 사용
                cameraExecutor
            ) { result ->
                val barcodeResults = result.getValue(barcodeScanner)
                if (barcodeResults.isNullOrEmpty()) return@MlKitAnalyzer

                val firstBarcode = barcodeResults[0]
                val rawValue = firstBarcode.rawValue ?: return@MlKitAnalyzer

                runOnUiThread {
                    // 바코드 경계 박스를 오버레이에 그리기
                    val boundingBox = firstBarcode.boundingBox
                    binding.overlayView.updateBoundingBox(boundingBox)
                    binding.tvResult.text = "스캔 결과: $rawValue\n형식: ${getBarcodeFormatName(firstBarcode.format)}"
                }
            }

            // CameraController로 간단하게 설정 (PreviewView와 통합됨)
            val cameraController = LifecycleCameraController(this).apply {
                setEnabledUseCases(CameraController.IMAGE_ANALYSIS)
                setImageAnalysisAnalyzer(cameraExecutor, analyzer)
                bindToLifecycle(this@BarcodeScannerActivity)
            }

            binding.previewView.controller = cameraController

        }, ContextCompat.getMainExecutor(this))
    }

    private fun getBarcodeFormatName(format: Int): String = when (format) {
        Barcode.FORMAT_QR_CODE -> "QR Code"
        Barcode.FORMAT_EAN_13 -> "EAN-13"
        Barcode.FORMAT_CODE_128 -> "Code 128"
        else -> "기타 (형식 코드: $format)"
    }

    override fun onDestroy() {
        super.onDestroy()
        cameraExecutor.shutdown()
        barcodeScanner.close()
    }
}
```

`ImageAnalysis`를 `CameraProvider`와 직접 바인딩하는 방식도 있다. 이 경우 분석 전략을 세밀하게 제어할 수 있다.

```kotlin
val imageAnalysis = ImageAnalysis.Builder()
    // 최신 프레임만 분석 (처리가 느릴 때 이전 프레임 드롭)
    .setBackpressureStrategy(ImageAnalysis.STRATEGY_KEEP_ONLY_LATEST)
    .setOutputImageFormat(ImageAnalysis.OUTPUT_IMAGE_FORMAT_YUV_420_888)
    .setTargetResolution(Size(1280, 720))
    .build()

imageAnalysis.setAnalyzer(cameraExecutor) { imageProxy ->
    val mediaImage = imageProxy.image ?: run {
        imageProxy.close()
        return@setAnalyzer
    }

    val inputImage = InputImage.fromMediaImage(
        mediaImage,
        imageProxy.imageInfo.rotationDegrees // 회전 정보 반드시 전달
    )

    barcodeScanner.process(inputImage)
        .addOnSuccessListener { barcodes ->
            barcodes.forEach { barcode ->
                Log.d("BarcodeScanner", "값: ${barcode.rawValue}")
            }
        }
        .addOnFailureListener { e ->
            Log.e("BarcodeScanner", "스캔 실패", e)
        }
        .addOnCompleteListener {
            // 반드시 close() 호출해야 다음 프레임이 전달됨
            imageProxy.close()
        }
}
```

**중요**: `imageProxy.close()`를 반드시 호출해야 한다. 이를 생략하면 분석기가 멈추고 새 프레임이 들어오지 않는다. `addOnCompleteListener`에서 항상 닫는 것이 안전하다.

---

## 5. VideoCapture 심화: Recorder와 Recording

CameraX 1.1부터 도입된 `VideoCapture<Recorder>` API는 `MediaRecorder`의 복잡함을 완전히 감춘다.

```kotlin
// 의존성 추가
implementation("androidx.camera:camera-video:1.5.0")
```

```kotlin
class VideoCaptureActivity : AppCompatActivity() {

    private var recording: Recording? = null
    private lateinit var videoCapture: VideoCapture<Recorder>
    private lateinit var cameraExecutor: ExecutorService

    private fun startCamera() {
        val cameraProviderFuture = ProcessCameraProvider.getInstance(this)
        cameraProviderFuture.addListener({
            val cameraProvider = cameraProviderFuture.get()

            val preview = Preview.Builder().build()
                .also { it.setSurfaceProvider(binding.previewView.surfaceProvider) }

            // Recorder 구성: 화질과 인코더 설정
            val recorder = Recorder.Builder()
                .setQualitySelector(
                    QualitySelector.from(
                        Quality.HIGHEST,   // 최고 화질 우선
                        FallbackStrategy.higherQualityOrLowerThan(Quality.SD)
                    )
                )
                .setExecutor(cameraExecutor)
                .build()

            videoCapture = VideoCapture.withOutput(recorder)

            try {
                cameraProvider.unbindAll()
                cameraProvider.bindToLifecycle(
                    this, CameraSelector.DEFAULT_BACK_CAMERA,
                    preview, videoCapture
                )
            } catch (e: Exception) {
                Log.e("CameraX", "Video binding failed", e)
            }
        }, ContextCompat.getMainExecutor(this))
    }

    private fun startRecording() {
        val videoCapture = videoCapture

        val contentValues = ContentValues().apply {
            put(MediaStore.Video.Media.DISPLAY_NAME, "VID_${System.currentTimeMillis()}")
            put(MediaStore.Video.Media.MIME_TYPE, "video/mp4")
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
                put(MediaStore.Video.Media.RELATIVE_PATH, "Movies/CameraXApp")
            }
        }

        val mediaStoreOutput = MediaStoreOutputOptions.Builder(
            contentResolver,
            MediaStore.Video.Media.EXTERNAL_CONTENT_URI
        ).setContentValues(contentValues).build()

        // 오디오 권한이 있는 경우 withAudioEnabled() 호출
        recording = videoCapture.output
            .prepareRecording(this, mediaStoreOutput)
            .withAudioEnabled()
            .start(ContextCompat.getMainExecutor(this)) { recordEvent ->
                when (recordEvent) {
                    is VideoRecordEvent.Start -> {
                        binding.btnRecord.text = "중지"
                        Log.d("CameraX", "녹화 시작")
                    }
                    is VideoRecordEvent.Finalize -> {
                        if (recordEvent.hasError()) {
                            Log.e("CameraX", "녹화 오류: ${recordEvent.error}")
                        } else {
                            val uri = recordEvent.outputResults.outputUri
                            Log.d("CameraX", "저장 완료: $uri")
                            Toast.makeText(this, "영상 저장 완료", Toast.LENGTH_SHORT).show()
                        }
                        binding.btnRecord.text = "녹화"
                        recording = null
                    }
                }
            }
    }

    private fun stopRecording() {
        recording?.stop()
    }
}
```

`QualitySelector.from()`의 `FallbackStrategy`는 기기가 요청한 화질을 지원하지 않을 때 대안을 자동 선택하도록 한다. `higherQualityOrLowerThan(Quality.SD)`는 SD보다 높은 화질 중 가장 가까운 것, 없으면 SD 이하에서 선택한다.

---

## 6. Use Case 조합 제약과 주의사항

CameraX는 동시에 여러 Use Case를 바인딩할 수 있지만, 카메라 하드웨어의 제약으로 인해 모든 조합이 보장되지는 않는다.

**보장되는 조합 (Camera2 Limited 레벨 이상):**
- `Preview` + `ImageCapture`
- `Preview` + `ImageAnalysis`
- `Preview` + `VideoCapture<Recorder>`

**보장되지 않는 조합:**
- `VideoCapture` + `ImageCapture` + `ImageAnalysis` 동시 사용: 일부 기기에서 `bindToLifecycle()`이 `IllegalArgumentException`을 던진다. `Camera2CameraInfo.getSupportedSessionTypes()`로 사전 확인이 필요하다.

**ImageAnalysis 백프레셔 전략 선택 기준:**

| 전략 | 동작 | 적합한 상황 |
|---|---|---|
| `STRATEGY_KEEP_ONLY_LATEST` | 처리 중 새 프레임 드롭 | ML 추론, 실시간 분석 |
| `STRATEGY_BLOCK_PRODUCER` | 분석 완료까지 카메라 대기 | 모든 프레임 필요한 경우 |

ML Kit처럼 처리 시간이 가변적인 경우 `STRATEGY_KEEP_ONLY_LATEST`가 적합하다. `STRATEGY_BLOCK_PRODUCER`는 프레임 드롭 없이 순서를 보장하지만, 분석이 느리면 프리뷰도 끊길 수 있다.

**회전(Rotation) 처리의 함정:**

`ImageCapture`와 `ImageAnalysis` 모두 `setTargetRotation()`을 통해 기기 회전 정보를 받아야 한다. `OrientationEventListener`를 연결해 기기 회전 시 자동 업데이트하는 것이 권장된다.

```kotlin
val orientationEventListener = object : OrientationEventListener(this) {
    override fun onOrientationChanged(orientation: Int) {
        val rotation = when (orientation) {
            in 45..134 -> Surface.ROTATION_270
            in 135..224 -> Surface.ROTATION_180
            in 225..314 -> Surface.ROTATION_90
            else -> Surface.ROTATION_0
        }
        imageCapture?.targetRotation = rotation
        imageAnalysis?.targetRotation = rotation
    }
}
orientationEventListener.enable()
```

**Executor 선택:**

`ImageAnalysis.setAnalyzer()`에 전달하는 Executor는 분석 처리 스레드를 결정한다. `ContextCompat.getMainExecutor()`는 메인 스레드에서 실행하므로 ML 추론처럼 무거운 작업에는 부적합하다. `Executors.newSingleThreadExecutor()`를 사용하고, 앱 종료 시 반드시 `shutdown()`을 호출한다.

---

## 7. 실전 팁

**CameraController vs CameraProvider**

`LifecycleCameraController`는 `ProcessCameraProvider`를 사용하는 방법의 고수준 래퍼다. `PreviewView`와의 연동이 간단하고, ML Kit Analyzer의 좌표 변환을 자동 처리한다. 대신 Use Case 설정의 세밀한 제어가 필요하다면 `ProcessCameraProvider`를 직접 사용한다.

**메모리 누수 예방**

`ImageProxy`는 내부적으로 `ImageReader`의 슬롯을 점유한다. `close()`를 호출하지 않으면 슬롯이 고갈되어 카메라가 더 이상 프레임을 전달하지 않는다. try-finally 패턴 또는 `.use {}` 확장 함수로 안전하게 닫는다.

```kotlin
imageAnalysis.setAnalyzer(cameraExecutor) { imageProxy ->
    imageProxy.use { proxy ->
        // 분석 로직 — use 블록이 끝나면 자동으로 close() 호출
        val bitmap = proxy.toBitmap()
        // ...
    }
}
```

**카메라 초기화 실패 처리**

`ProcessCameraProvider.getInstance()`의 `ListenableFuture`에서 예외가 발생할 수 있다. `addListener` 내부에서 `try-catch`로 `CameraUnavailableException`, `CameraIdExhaustedException` 등을 처리해야 한다.

---

## 결론

CameraX는 단순한 래퍼가 아닌, Android 카메라 생태계의 복잡성을 체계적으로 추상화한 아키텍처다. Use Case 기반 설계 덕분에 프리뷰, 캡처, 분석, 녹화를 독립적으로 조합할 수 있고, ML Kit과의 공식 통합(`MlKitAnalyzer`)으로 AI 기반 카메라 앱 개발의 진입 장벽도 크게 낮아졌다. `imageProxy.close()` 호출과 Executor 선택, 그리고 Use Case 조합 제약만 숙지하면, 프로덕션 품질의 카메라 기능을 자신 있게 구현할 수 있다.

## 참고 자료
- [CameraX 공식 개요 — Android Developers](https://developer.android.com/media/camera/camerax)
- [ImageAnalysis 공식 문서 — Android Developers](https://developer.android.com/media/camera/camerax/analyze)
- [VideoCapture 아키텍처 — Android Developers](https://developer.android.com/media/camera/camerax/video-capture)
- [ML Kit Analyzer 공식 문서 — Android Developers](https://developer.android.com/media/camera/camerax/mlkitanalyzer)
