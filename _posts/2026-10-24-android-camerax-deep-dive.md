---
layout: post
title: "Android CameraX 심화: Preview·ImageCapture·VideoCapture·ImageAnalysis 완전 정복"
date: 2026-10-24
categories: [android, flutter]
tags: [android, camerax, kotlin, camera2, imageanalysis, videocapture, mlkit, jetpack]
---

Android에서 카메라 기능을 구현하려면 예전에는 Camera1 API 또는 Camera2 API를 직접 다루어야 했습니다. 두 API 모두 방대하고 복잡한 설정 코드가 필요했으며, 기기마다 다른 동작을 처리하는 데 많은 시간이 소요됐습니다. Jetpack의 **CameraX** 라이브러리는 이런 복잡성을 유스케이스(Use Case) 단위로 추상화하여, 개발자가 "무엇을 할 것인가"에만 집중할 수 있게 해줍니다.

이 글에서는 CameraX의 핵심 유스케이스를 모두 다루고, 실전에서 바로 쓸 수 있는 Kotlin 코드와 함께 주의사항까지 정리합니다.

---

## 1. CameraX란 무엇인가

CameraX는 Jetpack 라이브러리 중 하나로, 내부적으로는 Camera2 API 위에서 동작합니다. API 레벨 21(Android 5.0 Lollipop)부터 지원하며 현재 출시된 안드로이드 기기의 98% 이상에서 사용 가능합니다.

CameraX가 제공하는 핵심 유스케이스는 다음 네 가지입니다.

| 유스케이스 | 설명 |
|---|---|
| **Preview** | 카메라 화면을 PreviewView에 출력 |
| **ImageCapture** | JPEG/PNG 등 정지 이미지 저장 |
| **VideoCapture** | 동영상과 오디오를 파일로 저장 |
| **ImageAnalysis** | 프레임을 버퍼로 받아 ML 알고리즘 등에 공급 |

---

## 2. Camera2 대비 CameraX를 써야 하는 이유

Camera2 API는 하드웨어를 세밀하게 제어할 수 있지만, 기본적인 사진 한 장을 찍으려 해도 `CameraDevice`, `CameraCaptureSession`, `CaptureRequest` 등 수십 줄의 보일러플레이트가 필요합니다. 기기마다 노출 모드, 오토포커스 특성이 달라 분기 처리가 많아지고, 회전 처리도 손수 계산해야 합니다.

CameraX는 이 모든 차이를 내부에서 해결합니다.

- **회전(Rotation) 자동 처리**: `ImageCapture`나 `ImageAnalysis` 결과물에 올바른 EXIF 방향이 자동 포함됩니다.
- **기기 호환성 쿼크(Quirk) 해소**: 삼성, 화웨이, 구글 픽셀 등 다양한 제조사의 비표준 동작을 내부 데이터베이스(`DeviceQuirksDatabase`)로 처리합니다.
- **LifecycleOwner 연동**: `bindToLifecycle`로 Activity/Fragment 생명주기에 맞춰 카메라가 자동으로 열리고 닫힙니다.
- **Extensions API**: HDR, 야간 모드, 보케(아웃포커스) 등 OEM 카메라 기능을 코드 몇 줄로 활성화할 수 있습니다.

---

## 3. 의존성 설정

```kotlin
// build.gradle.kts (app)
dependencies {
    val cameraxVersion = "1.4.0"

    implementation("androidx.camera:camera-camera2:$cameraxVersion")
    implementation("androidx.camera:camera-lifecycle:$cameraxVersion")
    implementation("androidx.camera:camera-view:$cameraxVersion")        // PreviewView
    implementation("androidx.camera:camera-video:$cameraxVersion")       // VideoCapture
    implementation("androidx.camera:camera-mlkit-vision:$cameraxVersion") // MLKit 연동
}
```

`AndroidManifest.xml`에 권한도 선언해야 합니다.

```xml
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.RECORD_AUDIO" />
```

---

## 4. Preview + ImageCapture 구현

가장 기본적인 시나리오입니다. `PreviewView`를 레이아웃에 배치하고, `CameraX`를 생명주기에 바인딩합니다.

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

        if (allPermissionsGranted()) {
            startCamera()
        } else {
            requestPermissions()
        }

        binding.btnCapture.setOnClickListener { takePhoto() }
    }

    private fun startCamera() {
        val cameraProviderFuture = ProcessCameraProvider.getInstance(this)

        cameraProviderFuture.addListener({
            val cameraProvider = cameraProviderFuture.get()

            // 1. Preview 유스케이스
            val preview = Preview.Builder().build().also {
                it.surfaceProvider = binding.previewView.surfaceProvider
            }

            // 2. ImageCapture 유스케이스
            imageCapture = ImageCapture.Builder()
                .setCaptureMode(ImageCapture.CAPTURE_MODE_MINIMIZE_LATENCY)
                .build()

            val cameraSelector = CameraSelector.DEFAULT_BACK_CAMERA

            try {
                cameraProvider.unbindAll()
                // 생명주기에 바인딩 — Activity가 destroyed되면 자동 해제
                cameraProvider.bindToLifecycle(
                    this, cameraSelector, preview, imageCapture
                )
            } catch (e: Exception) {
                Log.e("CameraX", "Use case binding failed", e)
            }

        }, ContextCompat.getMainExecutor(this))
    }

    private fun takePhoto() {
        val imageCapture = imageCapture ?: return

        // 파일 저장 옵션
        val contentValues = ContentValues().apply {
            put(MediaStore.MediaColumns.DISPLAY_NAME, "IMG_${System.currentTimeMillis()}")
            put(MediaStore.MediaColumns.MIME_TYPE, "image/jpeg")
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
                put(MediaStore.Images.Media.RELATIVE_PATH, "Pictures/CameraXSample")
            }
        }

        val outputOptions = ImageCapture.OutputFileOptions.Builder(
            contentResolver,
            MediaStore.Images.Media.EXTERNAL_CONTENT_URI,
            contentValues
        ).build()

        imageCapture.takePicture(
            outputOptions,
            ContextCompat.getMainExecutor(this),
            object : ImageCapture.OnImageSavedCallback {
                override fun onImageSaved(output: ImageCapture.OutputFileResults) {
                    Toast.makeText(this@CameraActivity, "사진 저장됨: ${output.savedUri}", Toast.LENGTH_SHORT).show()
                }

                override fun onError(exc: ImageCaptureException) {
                    Log.e("CameraX", "Photo capture failed: ${exc.message}", exc)
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

`bindToLifecycle`에 `preview`와 `imageCapture`를 함께 전달하면 두 유스케이스가 동시에 동작합니다. 동일한 `ProcessCameraProvider`에서 최대 세 개의 유스케이스를 동시에 바인딩할 수 있으며, 기기 하드웨어 제한에 따라 일부 조합은 지원되지 않을 수 있습니다.

---

## 5. VideoCapture 동영상 녹화

`VideoCapture`는 CameraX 1.1.0부터 안정화된 API로, `Recorder`와 함께 사용합니다.

```kotlin
class VideoCameraActivity : AppCompatActivity() {

    private var videoCapture: VideoCapture<Recorder>? = null
    private var recording: Recording? = null
    private lateinit var cameraExecutor: ExecutorService

    private fun startCamera() {
        val cameraProviderFuture = ProcessCameraProvider.getInstance(this)
        cameraProviderFuture.addListener({
            val cameraProvider = cameraProviderFuture.get()

            val preview = Preview.Builder().build().also {
                it.surfaceProvider = binding.previewView.surfaceProvider
            }

            // Recorder 품질 설정
            val recorder = Recorder.Builder()
                .setQualitySelector(
                    QualitySelector.from(
                        Quality.HIGHEST,
                        FallbackStrategy.higherQualityOrLowerThan(Quality.SD)
                    )
                )
                .build()

            videoCapture = VideoCapture.withOutput(recorder)

            try {
                cameraProvider.unbindAll()
                cameraProvider.bindToLifecycle(
                    this, CameraSelector.DEFAULT_BACK_CAMERA,
                    preview, videoCapture
                )
            } catch (e: Exception) {
                Log.e("CameraX", "VideoCapture binding failed", e)
            }
        }, ContextCompat.getMainExecutor(this))
    }

    @SuppressLint("MissingPermission")
    private fun captureVideo() {
        val videoCapture = videoCapture ?: return

        // 이미 녹화 중이면 중지
        val curRecording = recording
        if (curRecording != null) {
            curRecording.stop()
            recording = null
            return
        }

        val contentValues = ContentValues().apply {
            put(MediaStore.MediaColumns.DISPLAY_NAME, "VID_${System.currentTimeMillis()}")
            put(MediaStore.MediaColumns.MIME_TYPE, "video/mp4")
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
                put(MediaStore.Video.Media.RELATIVE_PATH, "Movies/CameraXSample")
            }
        }

        val mediaStoreOutput = MediaStoreOutputOptions.Builder(
            contentResolver,
            MediaStore.Video.Media.EXTERNAL_CONTENT_URI
        ).setContentValues(contentValues).build()

        recording = videoCapture.output
            .prepareRecording(this, mediaStoreOutput)
            .withAudioEnabled()  // 오디오 녹음 활성화
            .start(ContextCompat.getMainExecutor(this)) { recordEvent ->
                when (recordEvent) {
                    is VideoRecordEvent.Start -> {
                        binding.btnRecord.text = "중지"
                    }
                    is VideoRecordEvent.Finalize -> {
                        if (!recordEvent.hasError()) {
                            Toast.makeText(this, "동영상 저장됨", Toast.LENGTH_SHORT).show()
                        } else {
                            recording?.close()
                            recording = null
                            Log.e("CameraX", "Video capture error: ${recordEvent.error}")
                        }
                        binding.btnRecord.text = "녹화"
                    }
                }
            }
    }
}
```

`QualitySelector`는 `Quality.HIGHEST`부터 `Quality.SD`까지 폴백 체인을 정의할 수 있어, 기기가 4K를 지원하지 않아도 자동으로 낮은 해상도로 떨어집니다.

---

## 6. ImageAnalysis + MLKit 실시간 QR 코드 인식

`ImageAnalysis`는 카메라 프레임을 실시간으로 버퍼로 제공합니다. `MlKitAnalyzer`를 사용하면 좌표 변환까지 자동으로 처리되어 `PreviewView` 위에 오버레이를 정확하게 그릴 수 있습니다.

```kotlin
// 의존성 추가
// implementation("com.google.mlkit:barcode-scanning:17.3.0")

class QrScannerActivity : AppCompatActivity() {

    private lateinit var cameraController: LifecycleCameraController

    private fun startScanner() {
        val barcodeScanner = BarcodeScanning.getClient(
            BarcodeScannerOptions.Builder()
                .setBarcodeFormats(Barcode.FORMAT_QR_CODE)
                .build()
        )

        cameraController = LifecycleCameraController(this).apply {
            setEnabledUseCases(CameraController.IMAGE_ANALYSIS)
        }

        // MlKitAnalyzer: 좌표계를 PreviewView 기준으로 자동 변환
        cameraController.setImageAnalysisAnalyzer(
            ContextCompat.getMainExecutor(this),
            MlKitAnalyzer(
                listOf(barcodeScanner),
                COORDINATE_SYSTEM_VIEW_REFERENCED,
                ContextCompat.getMainExecutor(this)
            ) { result ->
                val barcodes = result?.getValue(barcodeScanner) ?: return@MlKitAnalyzer
                for (barcode in barcodes) {
                    val rawValue = barcode.rawValue ?: continue
                    val boundingBox = barcode.boundingBox
                    // QR 코드 값과 위치를 UI에 반영
                    runOnUiThread {
                        binding.overlay.drawQrOverlay(rawValue, boundingBox)
                    }
                }
            }
        )

        binding.previewView.controller = cameraController
        cameraController.bindToLifecycle(this)
    }
}
```

`LifecycleCameraController`는 `CameraController`의 생명주기 인식 버전으로, `CameraProvider`를 명시적으로 관리하지 않아도 됩니다. `COORDINATE_SYSTEM_VIEW_REFERENCED`를 지정하면 ML Kit 결과의 `boundingBox`가 `PreviewView` 픽셀 좌표로 반환되어 오버레이를 그리기 매우 편리합니다.

---

## 7. 주의사항 및 실전 팁

### 7-1. 동시 유스케이스 제한
CameraX는 기기의 카메라 하드웨어 수준(LIMITED, FULL, LEVEL_3)에 따라 동시에 바인딩 가능한 유스케이스 조합이 달라집니다. `Preview + ImageCapture + ImageAnalysis` 세 가지를 동시에 쓰면 LEGACY 기기에서는 실패할 수 있습니다. `Camera2CameraInfo`로 하드웨어 레벨을 사전에 확인하세요.

```kotlin
val camera2Info = Camera2CameraInfo.from(cameraInfo)
val hwLevel = camera2Info.getCameraCharacteristic(
    CameraCharacteristics.INFO_SUPPORTED_HARDWARE_LEVEL
)
```

### 7-2. ImageAnalysis 프레임 드롭 전략
`ImageAnalysis.Builder().setBackpressureStrategy()`로 `STRATEGY_KEEP_ONLY_LATEST`(기본값)나 `STRATEGY_BLOCK_PRODUCER`를 선택할 수 있습니다. ML 처리가 느릴 경우 `KEEP_ONLY_LATEST`를 유지하면 항상 최신 프레임만 처리하며, `BLOCK_PRODUCER`는 분석이 끝날 때까지 프레임 생산을 멈추어 버퍼를 채우지 않습니다.

### 7-3. 화면 회전 처리
`ImageCapture.setTargetRotation(display.rotation)` 또는 `OrientationEventListener`를 통해 회전을 실시간으로 업데이트해야 합니다. 특히 가로 모드를 고정한 앱이라도 EXIF 회전 정보가 잘못 기록될 수 있으므로, `OrientationEventListener`로 지속적으로 갱신하는 패턴이 권장됩니다.

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
    }
}
orientationEventListener.enable()
```

### 7-4. Extensions API로 OEM 카메라 기능 활성화
일부 기기에서는 야간 모드, HDR, 보케 효과 등을 Extensions API로 한 줄에 활성화할 수 있습니다.

```kotlin
val extensionsManager = ExtensionsManager.getInstanceAsync(context, cameraProvider).await()
if (extensionsManager.isExtensionAvailable(CameraSelector.DEFAULT_BACK_CAMERA, ExtensionMode.NIGHT)) {
    val nightCameraSelector = extensionsManager.getExtensionEnabledCameraSelector(
        CameraSelector.DEFAULT_BACK_CAMERA, ExtensionMode.NIGHT
    )
    cameraProvider.bindToLifecycle(this, nightCameraSelector, preview, imageCapture)
}
```

### 7-5. CameraController vs CameraProvider
- **`ProcessCameraProvider`**: 유연한 제어가 필요할 때. 유스케이스를 직접 조합하고, 여러 카메라를 동시에 열거나 커스텀 SurfaceProvider를 사용할 때 적합합니다.
- **`LifecycleCameraController`**: 단순한 시나리오에 빠르게 쓸 때. `PreviewView`와 결합하면 한두 줄로 카메라가 동작합니다. Zoom, Tap-to-focus, Pinch-to-zoom이 `PreviewView`와 자동으로 연동됩니다.

---

## 마치며

CameraX는 복잡한 Camera2 API를 유스케이스 단위로 추상화하여 개발 생산성을 크게 높여줍니다. Preview, ImageCapture, VideoCapture, ImageAnalysis라는 네 가지 빌딩 블록을 이해하면 다양한 카메라 앱을 빠르게 구현할 수 있습니다. MLKit와의 통합, Extensions API를 활용하면 QR 스캐너부터 야간 사진 앱까지 단일 코드베이스로 모두 처리할 수 있습니다.

---

## 참고 자료
- [Android CameraX 개요 — developer.android.com](https://developer.android.com/media/camera/camerax)
- [CameraX 유스케이스 가이드 — developer.android.com](https://developer.android.com/training/camerax)
- [CameraX ML Kit Analyzer 가이드 — developer.android.com](https://developer.android.com/training/camerax/mlkitanalyzer)
