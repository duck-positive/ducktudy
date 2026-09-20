---
layout: post
title: "Android MediaCodec 심화: 하드웨어 가속 비디오 인코딩/디코딩 완전 정복"
date: 2026-09-20
categories: [android, kotlin]
tags: [android, mediacodec, mediaextractor, mediamuxer, video, encoding, decoding, hardware-acceleration, kotlin]
---

## 개요

Android의 `MediaCodec`은 하드웨어 또는 소프트웨어 코덱을 통해 미디어 데이터를 직접 인코딩하거나 디코딩할 수 있는 저수준(low-level) API입니다. ExoPlayer(Media3)와 같은 고수준 라이브러리가 내부적으로 MediaCodec을 사용하지만, 특수한 요구사항이 있을 때는 직접 다뤄야 할 상황이 발생합니다.

이 글에서는 MediaCodec의 동작 원리부터 비동기 콜백 모드를 활용한 비디오 디코딩, H.264 하드웨어 인코딩까지 단계별로 깊이 있게 다룹니다.

---

## MediaCodec이란 무엇인가?

### 아키텍처 개요

```
[데이터 소스]  →  MediaExtractor  →  MediaCodec(디코더)  →  Surface/Buffer
[카메라/마이크] →  raw frame/PCM  →  MediaCodec(인코더)  →  MediaMuxer  →  파일
```

`MediaCodec`은 두 가지 방향으로 동작합니다.

- **디코더**: H.264, H.265, VP9 등 압축된 미디어 데이터를 받아 raw 프레임(YUV, RGB)으로 변환
- **인코더**: raw 데이터를 받아 압축된 스트림으로 변환

코덱은 **하드웨어 코덱**과 **소프트웨어 코덱** 두 종류가 있으며, `MediaCodecList`를 통해 기기에서 지원하는 코덱 목록을 조회할 수 있습니다.

### 동작 모드

MediaCodec은 두 가지 모드로 동작합니다.

1. **동기(Synchronous) 모드**: `dequeueInputBuffer()`, `dequeueOutputBuffer()` 를 직접 호출해 버퍼를 관리. 스레드 블로킹이 발생할 수 있음
2. **비동기(Asynchronous) 모드**: `setCallback()`으로 콜백을 등록하면 시스템이 버퍼 가용 여부를 알려줌. Android 5.0(API 21) 이상에서 권장

### 버퍼 큐 구조

MediaCodec은 내부적으로 두 가지 버퍼 큐를 관리합니다.

- **Input Buffer**: 압축 데이터(디코더) 또는 raw 데이터(인코더)를 채워 넣는 버퍼
- **Output Buffer**: 처리 완료된 결과가 담기는 버퍼

앱은 이 버퍼들을 acquire → fill/consume → release 하는 사이클로 코덱과 통신합니다.

---

## 왜 직접 MediaCodec을 사용해야 하는가?

고수준 라이브러리(Media3, ExoPlayer)가 MediaCodec을 추상화해 제공하지만, 다음 상황에서는 직접 접근이 필요합니다.

- **실시간 스트리밍 파이프라인**: RTMP/SRT 스트리밍을 위한 커스텀 인코딩 파이프라인 구성
- **비디오 트랜스코딩**: 해상도/비트레이트 변환, 코덱 변환을 정밀하게 제어
- **프레임 단위 처리**: 특정 프레임에 필터, 워터마크, 오버레이 적용 후 재인코딩
- **하드웨어 렌더링 최적화**: `Surface`를 직접 제공해 GPU 메모리 복사 없이 디코딩
- **저지연(Low-latency) 처리**: `KEY_LATENCY` 파라미터로 초저지연 인코딩 설정

---

## 실제 구현 예제 1: 비동기 모드 비디오 디코딩

`MediaExtractor`로 MP4 파일에서 비디오 트랙을 추출하고, `MediaCodec`의 비동기 콜백 모드로 `Surface`에 렌더링합니다.

```kotlin
import android.media.MediaCodec
import android.media.MediaExtractor
import android.media.MediaFormat
import android.view.Surface
import java.io.File
import java.nio.ByteBuffer
import java.util.concurrent.atomic.AtomicBoolean

class AsyncVideoDecoder(private val surface: Surface) {

    private var codec: MediaCodec? = null
    private var extractor: MediaExtractor? = null
    private val isRunning = AtomicBoolean(false)

    fun decode(file: File) {
        val ext = MediaExtractor().also { extractor = it }
        ext.setDataSource(file.absolutePath)

        // 비디오 트랙 찾기
        val videoTrackIndex = (0 until ext.trackCount).firstOrNull { i ->
            ext.getTrackFormat(i).getString(MediaFormat.KEY_MIME)
                ?.startsWith("video/") == true
        } ?: error("비디오 트랙 없음")

        ext.selectTrack(videoTrackIndex)
        val format = ext.getTrackFormat(videoTrackIndex)
        val mime = format.getString(MediaFormat.KEY_MIME)!!

        val decoder = MediaCodec.createDecoderByType(mime).also { codec = it }

        isRunning.set(true)

        // 비동기 콜백 등록 (configure 전에 반드시 설정)
        decoder.setCallback(object : MediaCodec.Callback() {

            override fun onInputBufferAvailable(codec: MediaCodec, index: Int) {
                if (!isRunning.get()) return

                val inputBuffer: ByteBuffer = codec.getInputBuffer(index) ?: return
                val sampleSize = ext.readSampleData(inputBuffer, 0)

                if (sampleSize < 0) {
                    // 스트림 종료 신호
                    codec.queueInputBuffer(
                        index, 0, 0,
                        0L,
                        MediaCodec.BUFFER_FLAG_END_OF_STREAM
                    )
                    isRunning.set(false)
                } else {
                    val presentationTimeUs = ext.sampleTime
                    codec.queueInputBuffer(index, 0, sampleSize, presentationTimeUs, 0)
                    ext.advance()
                }
            }

            override fun onOutputBufferAvailable(
                codec: MediaCodec,
                index: Int,
                info: MediaCodec.BufferInfo
            ) {
                if (info.flags and MediaCodec.BUFFER_FLAG_END_OF_STREAM != 0) {
                    codec.releaseOutputBuffer(index, false)
                    cleanup()
                    return
                }
                // Surface 렌더링: true 전달 시 Surface에 자동 렌더링
                codec.releaseOutputBuffer(index, true)
            }

            override fun onError(codec: MediaCodec, e: MediaCodec.CodecException) {
                cleanup()
            }

            override fun onOutputFormatChanged(codec: MediaCodec, format: MediaFormat) {
                // 해상도 변경 등 포맷 변경 시 처리
            }
        })

        // Surface에 직접 렌더링하도록 설정
        decoder.configure(format, surface, null, 0 /* decoder flag */)
        decoder.start()
    }

    fun stop() {
        isRunning.set(false)
        cleanup()
    }

    private fun cleanup() {
        runCatching { codec?.stop(); codec?.release() }
        runCatching { extractor?.release() }
        codec = null
        extractor = null
    }
}
```

핵심 포인트:
- `setCallback()`은 `configure()` **이전**에 반드시 호출해야 합니다
- `releaseOutputBuffer(index, true)` 에서 두 번째 인자를 `true`로 설정하면 Surface에 자동 렌더링됩니다
- EOS(End Of Stream) 처리를 빠뜨리면 코덱이 정상 종료되지 않습니다

---

## 실제 구현 예제 2: H.264 하드웨어 인코더

카메라 또는 `ImageReader`에서 얻은 YUV 프레임을 H.264로 인코딩하고, `MediaMuxer`로 MP4 파일에 저장합니다.

```kotlin
import android.media.MediaCodec
import android.media.MediaCodecInfo
import android.media.MediaFormat
import android.media.MediaMuxer
import java.io.File

class H264HardwareEncoder(
    private val outputFile: File,
    private val width: Int,
    private val height: Int,
    private val bitrateBps: Int = 4_000_000, // 4Mbps
    private val frameRate: Int = 30
) {
    private val MIME = MediaFormat.MIMETYPE_VIDEO_AVC  // "video/avc" = H.264
    private val TIMEOUT_US = 10_000L

    private lateinit var encoder: MediaCodec
    private lateinit var muxer: MediaMuxer
    private var videoTrackIndex = -1
    private var muxerStarted = false
    private var presentationTimeUs = 0L

    fun prepare() {
        val format = MediaFormat.createVideoFormat(MIME, width, height).apply {
            setInteger(MediaFormat.KEY_COLOR_FORMAT,
                MediaCodecInfo.CodecCapabilities.COLOR_FormatYUV420Flexible)
            setInteger(MediaFormat.KEY_BIT_RATE, bitrateBps)
            setInteger(MediaFormat.KEY_FRAME_RATE, frameRate)
            setInteger(MediaFormat.KEY_I_FRAME_INTERVAL, 1) // 1초마다 키프레임
            // 저지연 인코딩 설정 (Android 10+)
            setInteger(MediaFormat.KEY_LATENCY, 0)
        }

        encoder = MediaCodec.createEncoderByType(MIME)
        encoder.configure(format, null, null, MediaCodec.CONFIGURE_FLAG_ENCODE)
        encoder.start()

        muxer = MediaMuxer(outputFile.absolutePath, MediaMuxer.OutputFormat.MUXER_OUTPUT_MPEG_4)
    }

    fun encodeFrame(yuvData: ByteArray, isEndOfStream: Boolean = false) {
        val inputIndex = encoder.dequeueInputBuffer(TIMEOUT_US)
        if (inputIndex >= 0) {
            val inputBuffer = encoder.getInputBuffer(inputIndex)!!
            inputBuffer.clear()
            inputBuffer.put(yuvData)

            val flags = if (isEndOfStream) MediaCodec.BUFFER_FLAG_END_OF_STREAM else 0
            encoder.queueInputBuffer(inputIndex, 0, yuvData.size, presentationTimeUs, flags)
            presentationTimeUs += 1_000_000L / frameRate
        }

        drainEncoder(isEndOfStream)
    }

    private fun drainEncoder(endOfStream: Boolean) {
        val bufferInfo = MediaCodec.BufferInfo()

        while (true) {
            val outputIndex = encoder.dequeueOutputBuffer(bufferInfo, TIMEOUT_US)

            when {
                outputIndex == MediaCodec.INFO_OUTPUT_FORMAT_CHANGED -> {
                    check(!muxerStarted) { "포맷이 두 번 변경됨" }
                    videoTrackIndex = muxer.addTrack(encoder.outputFormat)
                    muxer.start()
                    muxerStarted = true
                }
                outputIndex == MediaCodec.INFO_TRY_AGAIN_LATER -> {
                    if (!endOfStream) break
                }
                outputIndex >= 0 -> {
                    val encodedData = encoder.getOutputBuffer(outputIndex)!!

                    if (bufferInfo.flags and MediaCodec.BUFFER_FLAG_CODEC_CONFIG != 0) {
                        bufferInfo.size = 0
                    }

                    if (bufferInfo.size > 0 && muxerStarted) {
                        encodedData.position(bufferInfo.offset)
                        encodedData.limit(bufferInfo.offset + bufferInfo.size)
                        muxer.writeSampleData(videoTrackIndex, encodedData, bufferInfo)
                    }

                    encoder.releaseOutputBuffer(outputIndex, false)

                    if (bufferInfo.flags and MediaCodec.BUFFER_FLAG_END_OF_STREAM != 0) break
                }
            }
        }
    }

    fun release() {
        runCatching {
            encoder.stop()
            encoder.release()
        }
        runCatching {
            if (muxerStarted) muxer.stop()
            muxer.release()
        }
    }
}
```

사용 예시:

```kotlin
val encoder = H264HardwareEncoder(
    outputFile = File(cacheDir, "output.mp4"),
    width = 1280,
    height = 720,
    bitrateBps = 5_000_000
)
encoder.prepare()

cameraFrames.forEach { frame ->
    encoder.encodeFrame(frame.toYuv420ByteArray())
}

encoder.encodeFrame(lastFrame.toYuv420ByteArray(), isEndOfStream = true)
encoder.release()
```

---

## 주의사항 및 팁

### 1. 코덱 선택: 하드웨어 vs 소프트웨어

`createDecoderByType()`은 기기가 우선순위에 따라 코덱을 선택합니다. 하드웨어 코덱을 명시적으로 선택하려면 `MediaCodecList`를 사용하세요.

```kotlin
fun findHardwareDecoder(mime: String): String? {
    val list = MediaCodecList(MediaCodecList.REGULAR_CODECS)
    return list.codecInfos
        .filter { !it.isEncoder && it.isHardwareAccelerated }
        .firstOrNull { info -> info.supportedTypes.any { it.equals(mime, ignoreCase = true) } }
        ?.name
}
```

### 2. Surface vs ByteBuffer 출력

| 방식 | 장점 | 단점 |
|------|------|------|
| Surface (렌더링) | GPU 메모리 직접 전달, 복사 없음 | ByteBuffer로 프레임 접근 불가 |
| ByteBuffer (처리) | 픽셀 단위 조작 가능 | CPU 메모리 복사 발생, 성능 저하 |

프레임 필터링이나 영상 분석이 필요 없다면 반드시 `Surface` 방식을 사용하세요.

### 3. KEY_COLOR_FORMAT 호환성 문제

인코더의 색상 포맷은 기기마다 다를 수 있습니다. `COLOR_FormatYUV420Flexible` (API 21+)을 사용하면 대부분의 기기에서 호환됩니다.

```kotlin
val capabilities = codecInfo.getCapabilitiesForType(MIME)
val isFlexibleSupported = MediaCodecInfo.CodecCapabilities.COLOR_FormatYUV420Flexible in capabilities.colorFormats
```

### 4. ANR 방지: 비동기 모드 필수

동기 모드에서 `dequeueOutputBuffer(TIMEOUT_US)`를 메인 스레드에서 호출하면 ANR이 발생합니다. 반드시 백그라운드 스레드 또는 비동기 콜백 모드를 사용하세요.

### 5. 메모리 누수 방지

`MediaCodec.release()`와 `MediaExtractor.release()`를 항상 `finally` 블록 또는 `use { }` 패턴으로 호출하세요. 릴리즈하지 않으면 코덱 세션이 시스템에 유지되어 다음 코덱 생성 시 `MediaCodec.CodecException`이 발생할 수 있습니다.

### 6. Android 12+ 호환성 주의

Android 12(API 31)부터 `Compatible Media Transcoding` 기능이 추가되어, HEVC로 촬영된 영상이 앱에서 읽힐 때 시스템이 AVC로 자동 변환할 수 있습니다. 직접 인코딩 파이프라인을 구성하는 경우 `ApplicationMediaCapabilities`를 명시적으로 선언해 시스템 트랜스코딩을 제어하세요.

---

## 정리

`MediaCodec`은 Android 미디어 처리의 핵심 저수준 API로, 하드웨어 가속 코덱에 직접 접근할 수 있습니다. 고수준 라이브러리로 해결할 수 없는 커스텀 인코딩 파이프라인, 실시간 스트리밍, 프레임 단위 처리가 필요한 경우 강력한 도구가 됩니다.

| 사용 상황 | 권장 API |
|-----------|----------|
| 일반 동영상 재생 | Media3 ExoPlayer |
| 커스텀 코덱 파이프라인 | MediaCodec + MediaExtractor |
| 파일 저장 필요 | MediaCodec + MediaMuxer |
| 저지연 스트리밍 | MediaCodec (KEY_LATENCY=0) |

다음 단계로 `ImageReader` + `MediaCodec` 인코더를 연결해 카메라 실시간 스트리밍 파이프라인을 구성해 보시기 바랍니다.

---

## 참고 자료

- [MediaCodec API Reference (Kotlin)](https://developer.android.com/reference/kotlin/android/media/MediaCodec)
- [MediaExtractor API Reference](https://developer.android.com/reference/android/media/MediaExtractor)
- [MediaMuxer API Reference](https://developer.android.com/reference/android/media/MediaMuxer)
- [Compatible Media Transcoding (Android 12+)](https://developer.android.com/media/platform/transcoding)
