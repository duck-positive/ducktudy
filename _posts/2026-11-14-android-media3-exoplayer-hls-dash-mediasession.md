---
layout: post
title: "Android Media3 ExoPlayer 심화: HLS/DASH 어댑티브 스트리밍·MediaSession·커스텀 UI 완전 정복"
date: 2026-11-14
categories: [android]
tags: [android, media3, exoplayer, hls, dash, mediasession, streaming, kotlin]
---

현대 Android 앱에서 미디어 재생은 단순히 `MediaPlayer`를 호출하는 수준을 훨씬 넘어섰다. 넷플릭스·유튜브·스포티파이가 구현하는 어댑티브 스트리밍, 잠금 화면 컨트롤, 차량 디스플레이 연동 같은 기능은 모두 견고한 미디어 아키텍처 위에서 동작한다. Jetpack **Media3**와 그 핵심 엔진인 **ExoPlayer**는 이런 복잡한 요구사항을 체계적으로 해결하기 위해 설계된 라이브러리다. 이 글에서는 단순 재생을 넘어 HLS/DASH 어댑티브 스트리밍, MediaSession을 통한 시스템 연동, 그리고 Compose 기반 커스텀 플레이어 UI까지 심층적으로 다룬다.

## 왜 Media3 ExoPlayer인가

안드로이드 플랫폼 기본 API인 `MediaPlayer`는 오랜 역사를 가지지만 여러 한계가 있다. 커스텀 스트리밍 프로토콜 지원 불가, 복잡한 상태 머신, DRM 처리의 어려움, 그리고 반드시 메인 스레드에서 다뤄야 하는 제약이 대표적이다.

**ExoPlayer**는 이런 한계를 모두 극복한다:

- **어댑티브 스트리밍 내장**: HLS(m3u8), DASH(mpd), SmoothStreaming을 별도 라이브러리로 쉽게 추가 가능
- **완전한 커스터마이징**: `Renderer`, `DataSource`, `LoadControl` 등 거의 모든 컴포넌트를 교체할 수 있다
- **Play Store 업데이트**: 앱 업데이트를 통해 플레이어 버전을 갱신할 수 있어, OS 업데이트를 기다릴 필요가 없다
- **Media3 통합**: `MediaSession`, `MediaController`, `Transformer`(편집)와 단일 `Player` 인터페이스로 연결된다

Media3 이전에는 ExoPlayer, MediaSession, 미디어 편집 라이브러리가 각각 독립적이었다. Media3는 이들을 하나의 우산 아래 통합했으며, 더 이상 컴포넌트 사이에 별도 커넥터가 필요하지 않다.

## 의존성 설정

```kotlin
// build.gradle.kts (Module: app)
dependencies {
    val media3Version = "1.11.1"

    // 핵심 ExoPlayer
    implementation("androidx.media3:media3-exoplayer:$media3Version")
    // HLS(m3u8) 스트리밍
    implementation("androidx.media3:media3-exoplayer-hls:$media3Version")
    // DASH(mpd) 스트리밍
    implementation("androidx.media3:media3-exoplayer-dash:$media3Version")
    // MediaSession 연동
    implementation("androidx.media3:media3-session:$media3Version")
    // 기본 UI (PlayerView)
    implementation("androidx.media3:media3-ui:$media3Version")
    // Compose UI 유틸리티 (PlayerSurface 등)
    implementation("androidx.media3:media3-ui-compose:$media3Version")
}
```

## 핵심 개념: Player 인터페이스와 컴포넌트 아키텍처

Media3의 가장 중요한 설계 원칙은 `Player` 인터페이스다. `ExoPlayer`, `MediaController` 모두 이 인터페이스를 구현하므로, `MediaSession`도 `PlayerView`도 모두 같은 인터페이스로 상호작용한다.

```
Activity / Fragment
      │
      ▼
  PlayerView ──── Player (ExoPlayer)
                      │
                  MediaSession ──── MediaSessionService (백그라운드)
                      │
              외부(잠금 화면, Android Auto, 이어폰 버튼)
```

이 구조 덕분에 로컬 재생과 Cast(원격) 재생을 전환할 때도 UI는 `Player` 인터페이스만 바라보면 된다.

## 구현 예제 1: HLS 어댑티브 스트리밍 플레이어

가장 흔한 스트리밍 시나리오인 HLS를 구현한다. HLS는 `.m3u8` 플레이리스트 파일로 여러 화질(bitrate)을 정의하며, ExoPlayer는 현재 네트워크 대역폭에 따라 자동으로 최적 화질을 선택한다.

```kotlin
class StreamingPlayerActivity : AppCompatActivity() {

    private lateinit var player: ExoPlayer
    private lateinit var binding: ActivityStreamingPlayerBinding

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityStreamingPlayerBinding.inflate(layoutInflater)
        setContentView(binding.root)
    }

    override fun onStart() {
        super.onStart()
        initializePlayer()
    }

    private fun initializePlayer() {
        // 커스텀 로드 컨트롤: 버퍼링 전략을 조정한다
        val loadControl = DefaultLoadControl.Builder()
            .setBufferDurationsMs(
                /* minBufferMs = */ 15_000,
                /* maxBufferMs = */ 50_000,
                /* bufferForPlaybackMs = */ 2_500,
                /* bufferForPlaybackAfterRebufferMs = */ 5_000,
            )
            .build()

        player = ExoPlayer.Builder(this)
            .setLoadControl(loadControl)
            .build()
            .also { exoPlayer ->
                binding.playerView.player = exoPlayer

                // HLS 스트림 설정
                val hlsUri = Uri.parse("https://devstreaming-cdn.apple.com/videos/streaming/examples/bipbop_adv_example_hevc/master.m3u8")
                val mediaItem = MediaItem.Builder()
                    .setUri(hlsUri)
                    // 명시적으로 MIME 타입을 지정하면 URI 확장자와 무관하게 HLS로 처리된다
                    .setMimeType(MimeTypes.APPLICATION_M3U8)
                    .build()

                exoPlayer.setMediaItem(mediaItem)
                exoPlayer.prepare()
                exoPlayer.playWhenReady = true

                // 플레이어 이벤트 리스너
                exoPlayer.addListener(object : Player.Listener {
                    override fun onPlaybackStateChanged(playbackState: Int) {
                        when (playbackState) {
                            Player.STATE_BUFFERING -> showBufferingIndicator(true)
                            Player.STATE_READY -> showBufferingIndicator(false)
                            Player.STATE_ENDED -> onPlaybackEnded()
                            Player.STATE_IDLE -> Unit
                        }
                    }

                    override fun onPlayerError(error: PlaybackException) {
                        // 에러 코드에 따른 복구 전략
                        when (error.errorCode) {
                            PlaybackException.ERROR_CODE_IO_NETWORK_CONNECTION_FAILED -> {
                                // 네트워크 오류: 잠시 후 재시도
                                retryPlayback()
                            }
                            PlaybackException.ERROR_CODE_BEHIND_LIVE_WINDOW -> {
                                // 라이브 스트림에서 너무 뒤처진 경우: 현재 라이브 위치로 이동
                                exoPlayer.seekToDefaultPosition()
                                exoPlayer.prepare()
                            }
                            else -> showError(error.message)
                        }
                    }

                    override fun onVideoSizeChanged(videoSize: VideoSize) {
                        // 동영상 해상도 변경 시 종횡비 조정
                        binding.playerView.setAspectRatioListener { _, _, _ ->
                            // 필요 시 레이아웃 갱신
                        }
                    }
                })
            }
    }

    override fun onStop() {
        super.onStop()
        // API 24+: 멀티윈도우 환경에서는 onStop에서 해제
        releasePlayer()
    }

    private fun releasePlayer() {
        player.release()
    }

    private fun showBufferingIndicator(show: Boolean) {
        binding.progressBar.isVisible = show
    }

    private fun onPlaybackEnded() {
        // 재생 완료 처리
    }

    private fun retryPlayback() {
        player.prepare()
    }

    private fun showError(message: String?) {
        Toast.makeText(this, "재생 오류: $message", Toast.LENGTH_LONG).show()
    }
}
```

## 구현 예제 2: MediaSessionService로 백그라운드 재생 구현

앱이 백그라운드로 가거나 잠금 화면이 활성화되어도 음악이 계속 재생되려면 `MediaSessionService`가 필요하다. 이 서비스는 `ExoPlayer`와 `MediaSession`을 관리하고, 외부(잠금 화면 컨트롤, Android Auto, Wear OS, 이어폰 버튼)와의 통신을 담당한다.

```kotlin
// AndroidManifest.xml에 등록 필요:
// <service android:name=".PlaybackService"
//          android:foregroundServiceType="mediaPlayback"
//          android:exported="true">
//   <intent-filter>
//     <action android:name="androidx.media3.session.MediaSessionService"/>
//   </intent-filter>
// </service>

class PlaybackService : MediaSessionService() {

    private var mediaSession: MediaSession? = null
    private lateinit var player: ExoPlayer

    override fun onCreate() {
        super.onCreate()
        player = ExoPlayer.Builder(this).build()

        mediaSession = MediaSession.Builder(this, player)
            // 외부 컨트롤러가 미디어 명령을 보낼 때 허용 여부를 결정
            .setCallback(object : MediaSession.Callback {
                override fun onAddMediaItems(
                    mediaSession: MediaSession,
                    controller: MediaSession.ControllerInfo,
                    mediaItems: List<MediaItem>,
                ): ListenableFuture<List<MediaItem>> {
                    // URI 등 필요한 정보를 채워 반환한다
                    val updatedItems = mediaItems.map { item ->
                        item.buildUpon()
                            .setUri(item.requestMetadata.mediaUri)
                            .build()
                    }
                    return Futures.immediateFuture(updatedItems)
                }
            })
            .build()
    }

    // 클라이언트(Activity)가 연결을 요청할 때 세션을 반환한다
    override fun onGetSession(controllerInfo: MediaSession.ControllerInfo): MediaSession? {
        return mediaSession
    }

    override fun onDestroy() {
        mediaSession?.run {
            player.release()
            release()
            mediaSession = null
        }
        super.onDestroy()
    }
}

// Activity에서 MediaController로 서비스와 통신한다
class PlayerActivity : AppCompatActivity() {

    private var mediaController: MediaController? = null

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        val sessionToken = SessionToken(
            this,
            ComponentName(this, PlaybackService::class.java),
        )

        // MediaController는 MediaSession과 같은 Player 인터페이스를 구현한다
        // Activity에서는 로컬 ExoPlayer가 아닌 MediaController를 Player로 사용한다
        val controllerFuture = MediaController.Builder(this, sessionToken).buildAsync()
        controllerFuture.addListener(
            {
                mediaController = controllerFuture.get()
                // PlayerView에 MediaController를 바인딩한다
                binding.playerView.player = mediaController

                // 재생 목록에 아이템 추가
                val mediaItem = MediaItem.Builder()
                    .setMediaId("track_001")
                    .setUri("https://example.com/audio.mp3")
                    .setMediaMetadata(
                        MediaMetadata.Builder()
                            .setTitle("샘플 트랙")
                            .setArtist("아티스트 이름")
                            .setArtworkUri(Uri.parse("https://example.com/cover.jpg"))
                            .build()
                    )
                    .build()

                mediaController?.setMediaItem(mediaItem)
                mediaController?.prepare()
                mediaController?.play()
            },
            MoreExecutors.directExecutor(),
        )
    }

    override fun onDestroy() {
        mediaController?.release()
        super.onDestroy()
    }
}
```

`MediaSessionService`를 사용하면 다음이 자동으로 처리된다:

- **포그라운드 서비스 알림**: 재생 중 상태바에 미디어 컨트롤 알림이 표시된다
- **Audio Focus**: 다른 앱이 오디오를 재생하면 적절하게 볼륨을 낮추거나 일시정지한다
- **잠금 화면 컨트롤**: 미디어 컨트롤이 잠금 화면에 나타난다
- **헤드셋 버튼 이벤트**: 이어폰의 재생/정지 버튼이 동작한다
- **Android Auto / Wear OS**: 별도 코드 없이 차량 디스플레이와 스마트워치에서 제어 가능하다

## DASH 어댑티브 스트리밍 추가

HLS 외에 DASH(Dynamic Adaptive Streaming over HTTP)도 지원하려면 의존성을 추가한 뒤 `.mpd` URI를 사용하면 된다. ExoPlayer는 URI의 `.mpd` 확장자나 MIME 타입(`MimeTypes.APPLICATION_MPD`)으로 자동 감지한다.

```kotlin
val dashMediaItem = MediaItem.Builder()
    .setUri("https://www.bok.or.kr/sample/video.mpd")
    .setMimeType(MimeTypes.APPLICATION_MPD)
    // DRM 보호 콘텐츠의 경우 Widevine 라이선스 서버를 지정한다
    .setDrmConfiguration(
        MediaItem.DrmConfiguration.Builder(C.WIDEVINE_UUID)
            .setLicenseUri("https://proxy.uat.widevine.com/proxy")
            .build()
    )
    .build()
```

## 주의사항과 실전 팁

### 1. 생명주기 관리: API 레벨에 따라 다르다

```kotlin
// API 23 이하: onStart/onStop에서 초기화/해제
// API 24+: 멀티윈도우 지원을 위해 onStart/onStop 사용 (onResume/onPause 아님)
// 화면이 꺼져도 PiP 등 멀티윈도우에서 재생이 계속될 수 있기 때문이다
```

### 2. 플레이어 해제를 반드시 호출하라

`player.release()`를 호출하지 않으면 코덱 리소스, 오디오 세션, 네트워크 연결이 해제되지 않는다. 메모리 누수보다 심각한 시스템 리소스 고갈이 발생할 수 있다.

### 3. `ERROR_CODE_BEHIND_LIVE_WINDOW` 처리

라이브 스트림에서 버퍼링이 너무 오래 걸리거나 앱이 백그라운드에서 오래 있으면 플레이어가 라이브 윈도우를 벗어난다. 이 오류는 `prepare()` 재호출로 복구할 수 있다.

### 4. `setWakeMode`로 화면 꺼짐 대응

오디오만 재생하는 경우 화면이 꺼져도 CPU가 유지되도록 wake lock을 설정해야 한다:

```kotlin
player = ExoPlayer.Builder(this)
    .setWakeMode(C.WAKE_MODE_LOCAL) // 로컬 wake lock
    // 혹은 C.WAKE_MODE_NETWORK: Wi-Fi도 유지
    .build()
```

### 5. 트랙 선택 커스터마이징

특정 화질이나 자막 트랙을 강제로 선택하려면 `TrackSelectionParameters`를 사용한다:

```kotlin
player.trackSelectionParameters = player.trackSelectionParameters
    .buildUpon()
    .setMaxVideoSizeSd() // SD 화질 이하로 제한 (저사양 기기 대응)
    .setPreferredAudioLanguage("ko") // 한국어 오디오 우선
    .setPreferredTextLanguage("ko") // 한국어 자막 우선
    .build()
```

### 6. `@OptIn(markerClass = UnstableApi.class)` 어노테이션

Media3의 일부 고급 API(`HlsMediaSource.Factory`, `DefaultLoadControl.Builder` 등)는 `@UnstableApi`로 표시되어 있다. 이는 API가 변경될 수 있다는 의미이며, 사용 시 `@OptIn(markerClass = UnstableApi::class)` 어노테이션을 클래스나 함수에 달아야 컴파일 경고를 억제할 수 있다.

## 마치며

Media3 ExoPlayer는 단순한 미디어 플레이어를 훨씬 넘어선 플랫폼이다. `Player` 인터페이스를 중심으로 한 단일화된 아키텍처 덕분에 로컬 재생, HLS/DASH 스트리밍, 백그라운드 오디오, 시스템 UI 연동, Android Auto까지 동일한 코드 구조로 처리할 수 있다. 처음에는 `MediaSessionService`와 `MediaController`의 조합이 복잡해 보이지만, 이 구조가 갖춰지면 잠금 화면 컨트롤부터 차량 디스플레이 연동까지 "공짜"로 얻을 수 있다는 점이 Media3의 가장 큰 가치다.

## 참고 자료
- [Jetpack Media3 공식 문서](https://developer.android.com/media/media3)
- [Media3 ExoPlayer 가이드](https://developer.android.com/media/media3/exoplayer)
- [HLS 스트리밍 지원 상세](https://developer.android.com/media/media3/exoplayer/hls)
- [AndroidX Media3 GitHub 저장소](https://github.com/androidx/media)
