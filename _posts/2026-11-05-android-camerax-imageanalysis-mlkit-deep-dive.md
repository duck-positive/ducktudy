---
layout: post
title: "Android CameraX 심화: Preview·ImageCapture·ImageAnalysis와 ML Kit 실시간 QR 스캔 완전 정복"
date: 2026-11-05
categories: [android]
tags: [android, camerax, mlkit, imageanalysis, kotlin, jetpack, camera]
---

스마트폰의 카메라 기능은 현대 앱에서 빠질 수 없는 핵심 요소다. 하지만 안드로이드에서 카메라를 다루는 일은 오랫동안 개발자들의 골칫거리였다. 수백 가지 기기 파편화, 복잡한 Camera2 API, 생명주기 관리의 어려움 등이 겹쳐 단순한 사진 촬영 하나도 수백 줄의 보일러플레이트를 요구했다. 이런 고통을 해결하기 위해 Google이 내놓은 답이 바로 **CameraX**다.

## CameraX란 무엇인가

CameraX는 Jetpack 라이브러리의 일부로, 카메라 앱 개발을 획기적으로 단순화하기 위해 설계된 라이브러리다. Android 5.0(API 레벨 21) 이상을 지원하며, 현재 존재하는 안드로이드 기기의 98% 이상에서 동작한다. 내부적으로는 Camera2 API 위에서 동작하지만, 개발자가 직접 Camera2의 복잡한 세션 관리, 스레드 처리, 기기별 예외 케이스를 다루지 않아도 된다.

CameraX의 핵심 철학은 **UseCase(유즈케이스)** 기반 설계다. 개발자는 "무엇을 하고 싶은가"만 선언하면 되고, 나머지는 CameraX가 처리한다. 주요 UseCase는 다음 네 가지다:

- **Preview**: 카메라 화면을 `PreviewView`에 실시간으로 표시한다.
- **ImageCapture**: 고품질 정지 이미지를 파일이나 메모리 버퍼로 저장한다.
- **VideoCapture**: 동영상과 오디오를 녹화한다.
- **ImageAnalysis**: 카메라 프레임을 실시간으로 분석한다. ML Kit, OpenCV 등과 연동할 때 사용한다.

## 왜 CameraX가 필요한가

### Camera1 → Camera2 → CameraX의 역사

안드로이드 카메라 API의 역사는 개발자의 고통의 역사이기도 하다.

**Camera1 API** (Android 1.0 ~ 5.0 시대): 사용하기는 쉬웠지만 기능이 제한적이었고, Android 5.0부터 공식 deprecated 처리됐다.

**Camera2 API** (Android 5.0+): 카메라 하드웨어를 세밀하게 제어할 수 있지만, `CameraDevice`, `CaptureSession`, `CaptureRequest`, `ImageReader` 등 수십 개의 클래스를 직접 연결해야 한다. 간단한 사진 촬영도 400줄 이상의 코드를 작성해야 하는 경우가 흔했다.

**CameraX**: 위의 복잡성을 추상화하면서도 Camera2의 모든 기능을 활용할 수 있다. 필요하다면 `Camera2Interop`을 통해 Camera2 API 직접 접근도 가능하다.

### CameraX의 핵심 장점

**생명주기 자동 관리**: `bindToLifecycle()`에 `LifecycleOwner`를 전달하면 Activity/Fragment의 생명주기에 맞게 카메라가 자동으로 열리고 닫힌다. `onResume`/`onPause`에서 직접 카메라를 열고 닫는 코드를 작성할 필요가 없다.

**기기 일관성 보장**: Google은 수십 개 제조사의 수백 개 기기에서 CameraX의 동작을 자동화된 테스트 랩으로 지속적으로 검증한다. 기기별로 발생하는 rotation, aspect ratio, preview size 불일치 같은 문제들을 CameraX가 내부적으로 처리한다.

**Extensions API**: 단 두 줄의 코드로 보케(인물 모드), HDR, 야간 모드, 뷰티 모드 등 기기 네이티브 카메라의 고급 기능을 활용할 수 있다.

## 프로젝트 세팅

`build.gradle.kts`에 의존성을 추가한다:

```kotlin
val cameraxVersion = "1.4.0"
dependencies {
    implementation("androidx.camera:camera-core:$cameraxVersion")
    implementation("androidx.camera:camera-camera2:$cameraxVersion")
    implementation("androidx.camera:camera-lifecycle:$cameraxVersion")
    implementation("androidx.camera:camera-view:$cameraxVersion")
    implementation("androidx.camera:camera-video:$cameraxVersion")
    // ML Kit 연동 시 추가
    implementation("androidx.camera:camera-mlkit-vision:$cameraxVersion")
    implementation("com.google.mlkit:barcode-scanning:17.3.0")
}
```

`AndroidManifest.xml`에 권한과 카메라 feature를 선언한다:

```xml
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.RECORD_AUDIO" />
<uses-feature android:name="android.hardware.camera.any" />
```

## 실제 구현 예제

### 예제 1: Preview + ImageCapture — 사진 촬영 앱

가장 기본적인 시나리오인 카메라 미리보기와 사진 촬영 기능을 구현한다.

**레이아웃 (activity_camera.xml)**:
```xml
<androidx.constraintlayout.widget.ConstraintLayout ...>
    <androidx.camera.view.PreviewView
        android:id="@+id/previewView"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintBottom_toTopOf="@id/btnCapture" />

    <Button
        android:id="@+id/btnCapture"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="촬영"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent" />
</androidx.constraintlayout.widget.ConstraintLayout>
```

**CameraActivity.kt**:
```kotlin
class CameraActivity : AppCompatActivity() {

    private lateinit var binding: ActivityCameraBinding
    private var imageCapture: ImageCapture? = null
    private lateinit var cameraExecutor: ExecutorService

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityCameraBinding.inflate(layoutInflater)
        setContentView(binding.root)

        // 카메라 권한 확인 후 카메라 시작
        if (allPermissionsGranted()) {
            startCamera()
        } else {
            ActivityCompat.requestPermissions(this, REQUIRED_PERMISSIONS, REQUEST_CODE)
        }

        binding.btnCapture.setOnClickListener { takePhoto() }
        cameraExecutor = Executors.newSingleThreadExecutor()
    }

    private fun startCamera() {
        val cameraProviderFuture = ProcessCameraProvider.getInstance(this)

        cameraProviderFuture.addListener({
            val cameraProvider = cameraProviderFuture.get()

            // 1. Preview UseCase 설정
            val preview = Preview.Builder().build().also {
                it.surfaceProvider = binding.previewView.surfaceProvider
            }

            // 2. ImageCapture UseCase 설정
            imageCapture = ImageCapture.Builder()
                .setCaptureMode(ImageCapture.CAPTURE_MODE_MINIMIZE_LATENCY)
                .build()

            // 3. 후면 카메라 선택
            val cameraSelector = CameraSelector.DEFAULT_BACK_CAMERA

            try {
                // 기존 바인딩 해제 후 재바인딩
                cameraProvider.unbindAll()
                cameraProvider.bindToLifecycle(
                    this,           // LifecycleOwner — 생명주기 자동 관리
                    cameraSelector,
                    preview,
                    imageCapture
                )
            } catch (e: Exception) {
                Log.e(TAG, "UseCase 바인딩 실패", e)
            }

        }, ContextCompat.getMainExecutor(this))
    }

    private fun takePhoto() {
        val imageCapture = imageCapture ?: return

        // MediaStore를 통해 사진 저장 (Android 10+ 대응)
        val name = SimpleDateFormat("yyyy-MM-dd-HH-mm-ss-SSS", Locale.US)
            .format(System.currentTimeMillis())
        val contentValues = ContentValues().apply {
            put(MediaStore.MediaColumns.DISPLAY_NAME, name)
            put(MediaStore.MediaColumns.MIME_TYPE, "image/jpeg")
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
                put(MediaStore.Images.Media.RELATIVE_PATH, "Pictures/CameraX-Demo")
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
                    Toast.makeText(baseContext, "사진 저장 완료: ${output.savedUri}", Toast.LENGTH_SHORT).show()
                }

                override fun onError(exc: ImageCaptureException) {
                    Log.e(TAG, "사진 촬영 실패: ${exc.message}", exc)
                }
            }
        )
    }

    private fun allPermissionsGranted() = REQUIRED_PERMISSIONS.all {
        ContextCompat.checkSelfPermission(baseContext, it) == PackageManager.PERMISSION_GRANTED
    }

    override fun onDestroy() {
        super.onDestroy()
        cameraExecutor.shutdown()
    }

    companion object {
        private const val TAG = "CameraX-Demo"
        private const val REQUEST_CODE = 10
        private val REQUIRED_PERMISSIONS = arrayOf(Manifest.permission.CAMERA)
    }
}
```

`CAPTURE_MODE_MINIMIZE_LATENCY`는 촬영 지연을 최소화하고, `CAPTURE_MODE_MAXIMIZE_QUALITY`는 화질을 우선한다. 용도에 따라 선택하면 된다.

### 예제 2: ImageAnalysis + ML Kit — 실시간 QR코드 스캐너

`ImageAnalysis` UseCase와 `MlKitAnalyzer`를 결합해 실시간으로 QR코드를 스캔하는 기능을 구현한다. `CameraController`를 사용하면 PreviewView와의 좌표 변환을 자동으로 처리해준다.

```kotlin
class QrScannerActivity : AppCompatActivity() {

    private lateinit var binding: ActivityQrScannerBinding
    private lateinit var cameraController: LifecycleCameraController

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityQrScannerBinding.inflate(layoutInflater)
        setContentView(binding.root)

        setupQrScanner()
    }

    private fun setupQrScanner() {
        // 1. ML Kit BarcodeScanner 설정 — QR코드만 감지
        val options = BarcodeScannerOptions.Builder()
            .setBarcodeFormats(Barcode.FORMAT_QR_CODE)
            .build()
        val barcodeScanner = BarcodeScanning.getClient(options)

        // 2. CameraController 초기화 (생명주기 자동 연결)
        cameraController = LifecycleCameraController(this)
        cameraController.bindToLifecycle(this)

        // 3. MlKitAnalyzer를 ImageAnalysis 분석기로 등록
        //    COORDINATE_SYSTEM_VIEW_REFERENCED: 결과 좌표가 PreviewView 기준으로 자동 변환됨
        cameraController.setImageAnalysisAnalyzer(
            ContextCompat.getMainExecutor(this),
            MlKitAnalyzer(
                listOf(barcodeScanner),
                CameraController.COORDINATE_SYSTEM_VIEW_REFERENCED,
                ContextCompat.getMainExecutor(this)
            ) { result ->
                val barcodes = result.getValue(barcodeScanner)
                if (!barcodes.isNullOrEmpty()) {
                    val qrValue = barcodes[0].rawValue ?: return@MlKitAnalyzer
                    handleQrResult(qrValue, barcodes[0].boundingBox)
                }
            }
        )

        // 4. PreviewView에 컨트롤러 연결
        binding.previewView.controller = cameraController
    }

    private fun handleQrResult(value: String, boundingBox: Rect?) {
        // 중복 감지 방지: 이미 처리 중이면 무시
        if (binding.tvResult.text == value) return

        runOnUiThread {
            binding.tvResult.text = value
            // boundingBox 좌표는 이미 PreviewView 기준으로 변환돼 있어
            // 오버레이 UI를 그리는 데 바로 사용할 수 있다
            drawOverlay(boundingBox)
        }
    }

    private fun drawOverlay(rect: Rect?) {
        // CustomView에 bounding box를 그려 사용자에게 스캔 위치를 시각적으로 표시
        binding.overlayView.updateBoundingBox(rect)
    }

    override fun onDestroy() {
        super.onDestroy()
        BarcodeScanning.getClient().close()
    }
}
```

`MlKitAnalyzer`의 핵심 장점은 `COORDINATE_SYSTEM_VIEW_REFERENCED`옵션이다. 카메라 센서 좌표와 화면 표시 좌표 사이의 복잡한 변환 행렬 계산을 CameraX가 대신 처리해주기 때문에, 반환된 `boundingBox`를 `PreviewView` 위에 오버레이로 바로 그릴 수 있다.

## 주의사항과 실전 팁

### 1. 권한 처리는 반드시 런타임에

Android 6.0(API 23)부터 카메라 권한은 반드시 런타임에 요청해야 한다. `ActivityResultContracts.RequestPermission()`을 사용하는 것이 현대적인 방식이다:

```kotlin
private val requestPermission =
    registerForActivityResult(ActivityResultContracts.RequestPermission()) { granted ->
        if (granted) startCamera() else showPermissionRationale()
    }
```

### 2. ImageAnalysis 백프레셔 전략

`ImageAnalysis`는 기본적으로 최신 프레임을 유지(`STRATEGY_KEEP_ONLY_LATEST`)한다. ML Kit 처리가 오래 걸리는 경우, 이전 프레임은 자동으로 버려지므로 큐가 쌓이지 않는다. 모든 프레임을 처리해야 하는 경우에만 `STRATEGY_BLOCK_PRODUCER`를 사용하되, 이 경우 처리 속도가 프레임 생성 속도를 따라가지 못하면 카메라 프리뷰 자체가 느려질 수 있으므로 주의해야 한다.

### 3. UseCase 조합 제약

모든 UseCase 조합이 모든 기기에서 지원되는 것은 아니다. `Preview + ImageCapture + ImageAnalysis` 세 가지 동시 사용은 대부분의 기기에서 지원되지만, `VideoCapture`를 추가하면 일부 구형 기기에서 `bindToLifecycle()`이 예외를 던질 수 있다. `camera-core` 1.3.0부터 지원되는 `ConcurrentCameraSession`을 통해 듀얼 카메라 동시 사용도 가능하다.

### 4. ImageCapture 모드 선택

- `CAPTURE_MODE_MINIMIZE_LATENCY`: 촬영 버튼 반응이 중요한 경우 (SNS, 일반 카메라)
- `CAPTURE_MODE_MAXIMIZE_QUALITY`: 고품질 사진이 필요한 경우 (문서 스캔, 상품 촬영)

### 5. Camera Extensions로 야간 모드 적용

단 몇 줄로 기기 네이티브 야간 모드를 적용할 수 있다:

```kotlin
val extensionsManager = ExtensionsManager.getInstanceAsync(context, cameraProvider).await()
if (extensionsManager.isExtensionAvailable(cameraSelector, ExtensionMode.NIGHT)) {
    val nightCameraSelector = extensionsManager.getExtensionEnabledCameraSelector(
        cameraSelector, ExtensionMode.NIGHT
    )
    cameraProvider.bindToLifecycle(this, nightCameraSelector, preview, imageCapture)
}
```

### 6. Jetpack Compose와의 통합

Compose 환경에서는 `AndroidView`로 `PreviewView`를 감싸거나, `camera-compose` 아티팩트의 `CameraPreview` Composable을 사용할 수 있다:

```kotlin
@Composable
fun CameraPreviewScreen() {
    val lifecycleOwner = LocalLifecycleOwner.current
    AndroidView(
        factory = { context ->
            PreviewView(context).also { previewView ->
                val cameraController = LifecycleCameraController(context)
                cameraController.bindToLifecycle(lifecycleOwner)
                previewView.controller = cameraController
            }
        },
        modifier = Modifier.fillMaxSize()
    )
}
```

## 마치며

CameraX는 단순한 래퍼 라이브러리가 아니다. 기기 파편화라는 안드로이드의 고질적 문제를 체계적으로 해결하고, 생명주기 관리와 ML Kit 연동을 자연스럽게 추상화한 완성도 높은 아키텍처다. Camera2 API를 직접 다루던 시절의 수백 줄 보일러플레이트가 이제 수십 줄의 선언적 코드로 줄어들었다.

특히 `ImageAnalysis + MlKitAnalyzer` 조합은 QR코드 스캔, 얼굴 인식, 텍스트 인식(OCR), 객체 감지 등 다양한 비전 AI 기능을 앱에 쉽게 통합할 수 있는 강력한 도구다. 온디바이스 AI가 점점 중요해지는 시대에 CameraX의 역할은 더욱 커질 것이다.

## 참고 자료
- [CameraX 공식 개요 문서](https://developer.android.com/media/camera/camerax)
- [ImageCapture — 사진 촬영 가이드](https://developer.android.com/training/camerax/take-photo)
- [VideoCapture 아키텍처 가이드](https://developer.android.com/media/camera/camerax/video-capture)
- [ML Kit Analyzer 공식 문서](https://developer.android.com/media/camera/camerax/mlkitanalyzer)
