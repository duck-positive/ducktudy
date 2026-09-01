---
layout: post
title: "Android MediaProjection API 심화: 화면 캡처·녹화 앱을 VirtualDisplay와 MediaCodec으로 완전 구현하기"
date: 2026-09-01
categories: [android]
tags: [android, mediaprojection, screencapture, virtualDisplay, mediacodec, foregroundservice, kotlin]
---

## 개요

Android에서 화면을 캡처하거나 녹화하는 기능은 화상 회의 앱, 게임 스트리밍 앱, 교육용 튜토리얼 앱에서 핵심 기능입니다. Android 5.0(API 21)부터 제공된 **MediaProjection API**는 공식적이고 안전한 방법으로 디바이스 화면을 캡처할 수 있도록 해줍니다. Android 14에서는 포그라운드 서비스 타입 선언이 강제되었고, Android 14의 앱 창 공유(App Screen Sharing) 기능으로 UX가 대폭 개선되었습니다. 이 글에서는 `MediaProjectionManager`, `VirtualDisplay`, `ImageReader`, `MediaCodec`을 연계하여 실제 작동하는 화면 녹화 앱을 처음부터 완성하는 방법을 설명합니다.

---

## 왜 MediaProjection API가 필요한가

과거 Android에서 화면을 캡처하려면 루트 권한이나 비공개 API를 사용해야 했습니다. `MediaProjection` API는 이 문제를 해결합니다.

- **사용자 동의 기반**: `createScreenCaptureIntent()`로 반드시 사용자에게 허가를 받아야 합니다. 앱이 임의로 화면을 녹화할 수 없습니다.
- **표준화된 인터페이스**: `VirtualDisplay`를 통해 화면 내용을 `Surface`로 끌어오고, `ImageReader`(정지 캡처)나 `MediaCodec`(동영상 인코딩)에 연결할 수 있습니다.
- **Android 14+ 보안 강화**: 포그라운드 서비스 타입을 `mediaProjection`으로 명시적으로 선언해야 하며, 세션마다 새로운 사용자 동의가 필요합니다.
- **단일 앱 공유(Android 14+)**: 전체 화면 대신 특정 앱 창만 공유할 수 있어 개인 정보 보호가 강화되었습니다.

---

## 핵심 구성 요소

```
사용자 동의
    ↓
MediaProjectionManager.createScreenCaptureIntent()
    ↓
ActivityResultLauncher → MediaProjection 토큰 획득
    ↓
MediaProjection.createVirtualDisplay()
    ↓
Surface (ImageReader or MediaCodec)
    ↓
PNG 저장 / H.264 인코딩 → MP4 저장
```

| 클래스 | 역할 |
|---|---|
| `MediaProjectionManager` | 화면 캡처 인텐트 생성 및 `MediaProjection` 인스턴스 반환 |
| `MediaProjection` | VirtualDisplay 생성의 진입점 |
| `VirtualDisplay` | 실제 화면을 미러링하는 가상 디스플레이 |
| `ImageReader` | 정지 화면 캡처 (PNG/JPEG 저장) |
| `MediaCodec` | 실시간 H.264 인코딩 |
| `MediaMuxer` | 인코딩된 비디오를 MP4 컨테이너로 묶음 |

---

## AndroidManifest.xml 설정

```xml
<!-- AndroidManifest.xml -->
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<!-- Android 14+: mediaProjection 타입 필수 -->
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PROJECTION" />

<service
    android:name=".ScreenCaptureService"
    android:foregroundServiceType="mediaProjection"
    android:exported="false" />
```

Android 14(API 34)부터 포그라운드 서비스에 `foregroundServiceType="mediaProjection"`이 없으면 `SecurityException`이 발생합니다.

---

## 구현 예제 1 — 화면 정지 캡처 (ImageReader)

사용자에게 동의를 받고 현재 화면을 PNG로 저장하는 예제입니다.

```kotlin
// ScreenCaptureActivity.kt
class ScreenCaptureActivity : AppCompatActivity() {

    private lateinit var mediaProjectionManager: MediaProjectionManager
    private var mediaProjection: MediaProjection? = null

    // ActivityResultLauncher로 사용자 동의 처리
    private val capturePermissionLauncher =
        registerForActivityResult(ActivityResultContracts.StartActivityForResult()) { result ->
            if (result.resultCode == Activity.RESULT_OK && result.data != null) {
                // 동의 완료 → MediaProjection 객체 생성
                mediaProjection = mediaProjectionManager.getMediaProjection(
                    result.resultCode,
                    result.data!!
                )
                // Android 14+: 콜백 등록 (메인 스레드 Handler 지정)
                mediaProjection?.registerCallback(projectionCallback, Handler(Looper.getMainLooper()))
                startScreenCapture()
            }
        }

    private val projectionCallback = object : MediaProjection.Callback() {
        override fun onStop() {
            // 사용자가 상태바 칩에서 공유 중단 시 호출
            releaseResources()
        }
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_screen_capture)

        mediaProjectionManager =
            getSystemService(Context.MEDIA_PROJECTION_SERVICE) as MediaProjectionManager

        binding.btnCapture.setOnClickListener {
            // 세션마다 새로운 동의 요청 (Android 10+부터 재사용 불가)
            capturePermissionLauncher.launch(
                mediaProjectionManager.createScreenCaptureIntent()
            )
        }
    }

    private fun startScreenCapture() {
        val projection = mediaProjection ?: return
        val windowMetrics = windowManager.currentWindowMetrics
        val width = windowMetrics.bounds.width()
        val height = windowMetrics.bounds.height()
        val density = resources.displayMetrics.densityDpi

        // ImageReader: RGBA_8888 포맷으로 픽셀 데이터 수신
        val imageReader = ImageReader.newInstance(width, height, PixelFormat.RGBA_8888, 2)

        val virtualDisplay = projection.createVirtualDisplay(
            "ScreenCapture",
            width, height, density,
            DisplayManager.VIRTUAL_DISPLAY_FLAG_AUTO_MIRROR,
            imageReader.surface,  // ImageReader의 Surface를 대상으로 지정
            null, null
        )

        // 1초 후 현재 프레임 캡처 (렌더링 안정화 대기)
        Handler(Looper.getMainLooper()).postDelayed({
            val image: Image? = imageReader.acquireLatestImage()
            if (image != null) {
                val planes = image.planes
                val buffer = planes[0].buffer
                val pixelStride = planes[0].pixelStride
                val rowStride = planes[0].rowStride
                val rowPadding = rowStride - pixelStride * width

                val bitmap = Bitmap.createBitmap(
                    width + rowPadding / pixelStride,
                    height,
                    Bitmap.Config.ARGB_8888
                )
                bitmap.copyPixelsFromBuffer(buffer)
                image.close()

                // MediaStore에 PNG 저장 (Android 10+ Scoped Storage)
                saveBitmapToMediaStore(bitmap)
            }
            // 리소스 해제
            virtualDisplay.release()
            imageReader.close()
            projection.stop()
        }, 1000L)
    }

    private fun saveBitmapToMediaStore(bitmap: Bitmap) {
        val contentValues = ContentValues().apply {
            put(MediaStore.Images.Media.DISPLAY_NAME, "screenshot_${System.currentTimeMillis()}.png")
            put(MediaStore.Images.Media.MIME_TYPE, "image/png")
            put(MediaStore.Images.Media.RELATIVE_PATH, Environment.DIRECTORY_PICTURES + "/Screenshots")
            put(MediaStore.Images.Media.IS_PENDING, 1)
        }
        val uri = contentResolver.insert(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, contentValues)
        uri?.let {
            contentResolver.openOutputStream(it)?.use { stream ->
                bitmap.compress(Bitmap.CompressFormat.PNG, 100, stream)
            }
            contentValues.put(MediaStore.Images.Media.IS_PENDING, 0)
            contentResolver.update(it, contentValues, null, null)
        }
    }

    private fun releaseResources() {
        mediaProjection?.unregisterCallback(projectionCallback)
        mediaProjection?.stop()
        mediaProjection = null
    }

    override fun onDestroy() {
        super.onDestroy()
        releaseResources()
    }
}
```

**핵심 포인트**: `ImageReader`의 `acquireLatestImage()`는 반드시 `image.close()`를 호출해야 합니다. 닫지 않으면 `ImageReader`의 버퍼가 가득 차서 새 프레임을 받지 못합니다.

---

## 구현 예제 2 — 화면 녹화 (MediaCodec + MediaMuxer + ForegroundService)

실시간으로 화면을 H.264로 인코딩하여 MP4로 저장하는 전체 구현입니다. Android 14+에서 포그라운드 서비스가 필수이므로 서비스로 분리합니다.

```kotlin
// ScreenRecordService.kt
class ScreenRecordService : Service() {

    companion object {
        const val ACTION_START = "action.START_RECORDING"
        const val ACTION_STOP = "action.STOP_RECORDING"
        const val EXTRA_RESULT_CODE = "extra.RESULT_CODE"
        const val EXTRA_RESULT_DATA = "extra.RESULT_DATA"
        private const val CHANNEL_ID = "screen_record_channel"
        private const val MIME_TYPE = "video/avc"   // H.264
        private const val FRAME_RATE = 30
        private const val I_FRAME_INTERVAL = 1
        private const val BIT_RATE = 8_000_000     // 8 Mbps
    }

    private var mediaProjection: MediaProjection? = null
    private var virtualDisplay: VirtualDisplay? = null
    private var mediaCodec: MediaCodec? = null
    private var mediaMuxer: MediaMuxer? = null
    private var videoTrackIndex = -1
    private var muxerStarted = false
    private var outputFile: File? = null
    private val encoderThread = HandlerThread("EncoderThread").also { it.start() }
    private val encoderHandler = Handler(encoderThread.looper)

    override fun onBind(intent: Intent?): IBinder? = null

    override fun onCreate() {
        super.onCreate()
        createNotificationChannel()
    }

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        when (intent?.action) {
            ACTION_START -> {
                // Android 14+: startForeground 호출 전에 서비스 타입 지정
                startForeground(
                    1,
                    buildNotification(),
                    ServiceInfo.FOREGROUND_SERVICE_TYPE_MEDIA_PROJECTION
                )
                val resultCode = intent.getIntExtra(EXTRA_RESULT_CODE, Activity.RESULT_CANCELED)
                val data = intent.getParcelableExtra<Intent>(EXTRA_RESULT_DATA) ?: return START_NOT_STICKY

                val mpManager = getSystemService(MEDIA_PROJECTION_SERVICE) as MediaProjectionManager
                mediaProjection = mpManager.getMediaProjection(resultCode, data)
                mediaProjection?.registerCallback(
                    object : MediaProjection.Callback() {
                        override fun onStop() { stopRecording() }
                    },
                    Handler(Looper.getMainLooper())
                )
                startRecording()
            }
            ACTION_STOP -> stopRecording()
        }
        return START_NOT_STICKY
    }

    private fun startRecording() {
        val dm = resources.displayMetrics
        val width = dm.widthPixels
        val height = dm.heightPixels
        val density = dm.densityDpi

        // 출력 파일 (MediaStore 사용 권장, 여기서는 app-specific storage로 단순화)
        outputFile = File(getExternalFilesDir(Environment.DIRECTORY_MOVIES),
            "record_${System.currentTimeMillis()}.mp4")

        // MediaCodec 설정
        val format = MediaFormat.createVideoFormat(MIME_TYPE, width, height).apply {
            setInteger(MediaFormat.KEY_COLOR_FORMAT,
                MediaCodecInfo.CodecCapabilities.COLOR_FormatSurface)
            setInteger(MediaFormat.KEY_BIT_RATE, BIT_RATE)
            setInteger(MediaFormat.KEY_FRAME_RATE, FRAME_RATE)
            setInteger(MediaFormat.KEY_I_FRAME_INTERVAL, I_FRAME_INTERVAL)
        }

        mediaCodec = MediaCodec.createEncoderByType(MIME_TYPE).apply {
            configure(format, null, null, MediaCodec.CONFIGURE_FLAG_ENCODE)
        }

        mediaMuxer = MediaMuxer(outputFile!!.absolutePath, MediaMuxer.OutputFormat.MUXER_OUTPUT_MPEG_4)

        // MediaCodec의 입력 Surface를 VirtualDisplay에 연결
        val inputSurface = mediaCodec!!.createInputSurface()
        mediaCodec!!.start()

        virtualDisplay = mediaProjection?.createVirtualDisplay(
            "ScreenRecord",
            width, height, density,
            DisplayManager.VIRTUAL_DISPLAY_FLAG_AUTO_MIRROR,
            inputSurface,
            null, null
        )

        // 비동기 인코딩 루프
        drainEncoder()
    }

    private fun drainEncoder() {
        encoderHandler.post {
            val codec = mediaCodec ?: return@post
            val muxer = mediaMuxer ?: return@post
            val bufferInfo = MediaCodec.BufferInfo()

            while (true) {
                val outputBufferIndex = codec.dequeueOutputBuffer(bufferInfo, 10_000L)
                when {
                    outputBufferIndex == MediaCodec.INFO_OUTPUT_FORMAT_CHANGED -> {
                        // 이 시점에서 MediaMuxer 시작
                        videoTrackIndex = muxer.addTrack(codec.outputFormat)
                        muxer.start()
                        muxerStarted = true
                    }
                    outputBufferIndex >= 0 -> {
                        val encodedData = codec.getOutputBuffer(outputBufferIndex) ?: continue
                        if (bufferInfo.flags and MediaCodec.BUFFER_FLAG_CODEC_CONFIG != 0) {
                            bufferInfo.size = 0  // SPS/PPS는 MediaMuxer가 자동 처리
                        }
                        if (bufferInfo.size > 0 && muxerStarted) {
                            encodedData.position(bufferInfo.offset)
                            encodedData.limit(bufferInfo.offset + bufferInfo.size)
                            muxer.writeSampleData(videoTrackIndex, encodedData, bufferInfo)
                        }
                        codec.releaseOutputBuffer(outputBufferIndex, false)
                        if (bufferInfo.flags and MediaCodec.BUFFER_FLAG_END_OF_STREAM != 0) {
                            break  // EOS 수신 → 루프 종료
                        }
                    }
                }
            }
        }
    }

    private fun stopRecording() {
        try {
            mediaCodec?.signalEndOfInputStream()  // 인코더에 EOS 신호
            Thread.sleep(200)  // 마지막 프레임 플러시 대기
            mediaCodec?.stop()
            mediaCodec?.release()
            if (muxerStarted) mediaMuxer?.stop()
            mediaMuxer?.release()
        } catch (e: Exception) {
            Log.e("ScreenRecord", "Error stopping recorder", e)
        } finally {
            virtualDisplay?.release()
            mediaProjection?.stop()
            mediaCodec = null
            mediaMuxer = null
            virtualDisplay = null
            mediaProjection = null
            muxerStarted = false
        }
        // MediaStore에 파일 등록 (갤러리에서 보이게)
        outputFile?.let { file ->
            MediaScannerConnection.scanFile(this, arrayOf(file.absolutePath), null, null)
        }
        stopForeground(STOP_FOREGROUND_REMOVE)
        stopSelf()
    }

    private fun createNotificationChannel() {
        val channel = NotificationChannel(
            CHANNEL_ID, "화면 녹화", NotificationManager.IMPORTANCE_LOW
        )
        getSystemService(NotificationManager::class.java).createNotificationChannel(channel)
    }

    private fun buildNotification() = NotificationCompat.Builder(this, CHANNEL_ID)
        .setContentTitle("화면 녹화 중")
        .setContentText("상태바 칩을 탭하거나 중지 버튼을 누르세요")
        .setSmallIcon(R.drawable.ic_record)
        .addAction(R.drawable.ic_stop, "중지",
            PendingIntent.getService(this, 0,
                Intent(this, ScreenRecordService::class.java).apply { action = ACTION_STOP },
                PendingIntent.FLAG_IMMUTABLE))
        .build()
}
```

---

## 화면 크기 계산 — WindowMetrics API (API 30+)

Android 11부터 `WindowMetrics`를 사용해 화면 크기를 구해야 합니다. `DisplayMetrics`의 `widthPixels`는 멀티 윈도우 환경에서 잘못된 값을 반환할 수 있습니다.

```kotlin
// Activity 컨텍스트에서 정확한 화면 크기 계산
fun getWindowSize(activity: Activity): Pair<Int, Int> {
    return if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
        val metrics = activity.windowManager.currentWindowMetrics
        val insets = metrics.windowInsets.getInsetsIgnoringVisibility(
            WindowInsetsCompat.Type.systemBars()
        )
        val width = metrics.bounds.width() - insets.left - insets.right
        val height = metrics.bounds.height() - insets.top - insets.bottom
        Pair(width, height)
    } else {
        @Suppress("DEPRECATION")
        val dm = activity.resources.displayMetrics
        Pair(dm.widthPixels, dm.heightPixels)
    }
}
```

VirtualDisplay 생성 시 반드시 실제 렌더링 영역과 일치하는 해상도를 전달해야 `createVirtualDisplay()`의 `densityDpi`와 함께 올바른 스케일링이 적용됩니다.

---

## 주의사항과 실전 팁

### 1. 세션 토큰 재사용 금지
Android 10(API 29)부터 `MediaProjection` 토큰은 한 번만 사용할 수 있습니다. 새로운 캡처 세션을 시작할 때마다 `createScreenCaptureIntent()`로 사용자 동의를 다시 받아야 합니다. 이전 토큰을 재사용하면 `SecurityException: MediaProjection has been used before`가 발생합니다.

### 2. 포그라운드 서비스 타입 (Android 14 필수)
Android 14(API 34) 이상에서는 반드시:
- `AndroidManifest.xml`에 `FOREGROUND_SERVICE_MEDIA_PROJECTION` 권한 선언
- `Service`의 `foregroundServiceType="mediaProjection"` 지정
- `startForeground(id, notification, FOREGROUND_SERVICE_TYPE_MEDIA_PROJECTION)` 호출
이 세 가지를 모두 만족해야 합니다. 하나라도 빠지면 `MediaProjection.createVirtualDisplay()` 호출 시 `SecurityException`이 발생합니다.

### 3. 콜백 등록으로 중단 이벤트 처리
Android 15 QPR1부터 상태바에 화면 공유 칩이 표시되고 사용자가 직접 공유를 중단할 수 있습니다. `MediaProjection.Callback.onStop()`을 반드시 구현하여 리소스를 즉시 해제해야 합니다. 그렇지 않으면 앱이 중단된 프로젝션에 계속 데이터를 쓰려 해 ANR이나 크래시가 발생할 수 있습니다.

### 4. MediaCodec EOS 처리
`signalEndOfInputStream()` 호출 후 `drainEncoder()`가 `BUFFER_FLAG_END_OF_STREAM` 플래그를 받을 때까지 기다려야 `MediaMuxer.stop()`을 안전하게 호출할 수 있습니다. 이 순서를 지키지 않으면 MP4 파일이 손상될 수 있습니다.

### 5. 단일 앱 공유 UX (Android 14+)
`createScreenCaptureIntent()`가 표시하는 다이얼로그에서 사용자는 '전체 화면'과 '특정 앱' 중 하나를 선택할 수 있습니다. 앱이 단일 앱 공유 모드에서 제외되길 원한다면(예: 보안 앱), `WindowManager.LayoutParams.layoutInDisplayCutoutMode`를 통해 옵트아웃을 구현할 수 있습니다.

### 6. 오디오 녹음 병행
화면과 함께 오디오를 녹음하려면 `AudioRecord`나 `MediaProjection.createAudioCapture()`(Android 10+)를 별도로 설정하고, `MediaMuxer`에 오디오 트랙을 추가해야 합니다. 시스템 오디오 캡처에는 `RECORD_AUDIO` 권한이 필요합니다.

### 7. 화면 회전 대응
화면이 회전하면 `VirtualDisplay`에 연결된 Surface의 크기가 자동으로 업데이트되지 않습니다. `OrientationEventListener`나 `ComponentActivity.onConfigurationChanged()`를 감지하여 VirtualDisplay를 재생성해야 합니다.

---

## 전체 아키텍처 요약

```
Activity (사용자 동의 요청)
    │
    ├─ capturePermissionLauncher → MediaProjection 획득
    │
    └─ ScreenCaptureService 시작 (ForegroundService)
           │
           ├─ MediaCodec.createInputSurface()
           │
           ├─ MediaProjection.createVirtualDisplay(surface)
           │       화면 → Surface → MediaCodec 입력
           │
           ├─ drainEncoder() (백그라운드 스레드)
           │       MediaCodec → 인코딩 → MediaMuxer → MP4
           │
           └─ stopRecording()
                   signalEndOfInputStream → EOS → Muxer.stop() → 파일 완성
```

MediaProjection API는 Android의 강력한 화면 공유 생태계의 핵심입니다. `VirtualDisplay`를 `ImageReader`에 연결하면 정지 화면을 캡처하고, `MediaCodec`의 입력 Surface에 연결하면 실시간 H.264 비디오 스트림을 만들 수 있습니다. 각 Android 버전별 요구사항(포그라운드 서비스 타입, 토큰 재사용 금지, 콜백 처리)을 꼼꼼히 지키면 안정적인 화면 녹화 앱을 구현할 수 있습니다.

## 참고 자료
- [Android Media Projection 공식 가이드](https://developer.android.com/media/grow/media-projection)
- [MediaProjectionManager API Reference](https://developer.android.com/reference/android/media/projection/MediaProjectionManager)
- [VirtualDisplay API Reference](https://developer.android.com/reference/android/hardware/display/VirtualDisplay)
