---
layout: post
title: "Android Jetpack Media3 심화: ExoPlayer·MediaSession·MediaSessionService로 완전한 백그라운드 미디어 재생 앱 구현하기"
date: 2026-07-09
categories: [android, flutter]
tags: [android, media3, exoplayer, mediasession, mediasessionservice, kotlin, jetpack, background-playback]
---

Jetpack Media3는 Android의 오디오/비디오 재생을 위한 통합 미디어 라이브러리 세트입니다. 기존에 별도로 관리되던 ExoPlayer, MediaSession, MediaRouter가 하나의 일관된 API 아래 통합되었습니다. 2022년 안정 버전 출시 이후 2026년 현재 1.10 버전까지 지속적으로 발전해 왔으며, 복잡했던 미디어 재생 아키텍처를 대폭 단순화했습니다.

이 아티클에서는 Media3의 핵심 컴포넌트인 ExoPlayer, MediaSession, MediaSessionService를 조합하여 백그라운드에서도 동작하는 실전 미디어 재생 앱을 단계적으로 구현합니다.

## 왜 Media3가 필요한가?

### 기존 미디어 API의 문제점

Android에서 미디어 재생 앱을 개발해 본 경험이 있다면, 다음과 같은 복잡성에 부딪혔을 것입니다.

**1. 분산된 라이브러리 생태계**

과거에는 `ExoPlayer`(별도 라이브러리), `MediaSessionCompat`(Support Library), `MediaBrowserCompat` 등을 각각 추가하고 서로 연결하는 "커넥터" 코드를 직접 작성해야 했습니다. 예를 들어 `ExoPlayer`와 `MediaSessionConnector`를 연결하고, 다시 `MediaBrowserServiceCompat`을 구현하는 방식이었는데, 이 커넥터들이 버전마다 달라 유지보수가 매우 번거로웠습니다.

**2. 스레딩 모델의 불일치**

`MediaPlayer`는 내부적으로 복잡한 상태 머신을 가지고 있어, 잘못된 상태에서 메서드를 호출하면 `IllegalStateException`이 발생합니다. 반면 `ExoPlayer`는 단일 스레드 모델을 사용하지만, 이를 `Service`와 연결하면 IPC 경계에서 스레딩 문제가 발생하기 쉬웠습니다.

**3. Android Auto / Wear OS 지원의 어려움**

`MediaBrowserServiceCompat`을 올바르게 구현하지 않으면 Android Auto, Wear OS, Google Assistant 등 외부 클라이언트가 미디어 앱을 제어할 수 없습니다. 기존에는 이 표준 인터페이스 구현이 매우 번거로웠습니다.

### Media3가 해결하는 것

Media3는 이 모든 문제를 `Player` 인터페이스 하나로 통합합니다. `ExoPlayer`가 `Player`를 구현하고, `MediaSession`도 `Player`를 받으며, UI 컴포넌트(`PlayerView`, `PlayerSurface`)도 `Player`를 직접 바인딩합니다. 복잡한 커넥터가 사라진 것입니다.

```
ExoPlayer (Player 구현체)
    └── MediaSession (Player를 감싸서 외부 제어 인터페이스 제공)
            └── MediaSessionService (Service 생명주기 관리)
                        └── MediaController (클라이언트에서 Player처럼 사용)
```

## 의존성 설정

`build.gradle.kts`에 필요한 Media3 모듈을 추가합니다. Media3는 모듈형으로 설계되어 필요한 것만 선택할 수 있습니다.

```kotlin
dependencies {
    val media3Version = "1.10.1"

    // 핵심: ExoPlayer
    implementation("androidx.media3:media3-exoplayer:$media3Version")

    // 적응형 스트리밍 (HLS, DASH)
    implementation("androidx.media3:media3-exoplayer-hls:$media3Version")
    implementation("androidx.media3:media3-exoplayer-dash:$media3Version")

    // MediaSession 및 MediaSessionService
    implementation("androidx.media3:media3-session:$media3Version")

    // PlayerView (전통 View 기반)
    implementation("androidx.media3:media3-ui:$media3Version")

    // Jetpack Compose UI (PlayerSurface 등)
    implementation("androidx.media3:media3-ui-compose-material3:$media3Version")
}
```

또한 Java 8+ 기능을 활성화해야 합니다:

```kotlin
android {
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_1_8
        targetCompatibility = JavaVersion.VERSION_1_8
    }
}
```

## 핵심 구현: MediaSessionService

미디어 재생 앱의 핵심은 `MediaSessionService`입니다. 앱이 백그라운드로 이동하거나 화면이 잠겨도 음악이 계속 재생되려면, 플레이어를 Activity가 아닌 `Service`에서 관리해야 합니다. `MediaSessionService`는 이를 위한 전용 Service 기반 클래스입니다.

```kotlin
class PlaybackService : MediaSessionService() {

    private var mediaSession: MediaSession? = null

    override fun onCreate() {
        super.onCreate()

        // ExoPlayer 인스턴스 생성
        val player = ExoPlayer.Builder(this)
            .setAudioAttributes(
                AudioAttributes.Builder()
                    .setContentType(C.AUDIO_CONTENT_TYPE_MUSIC)
                    .setUsage(C.USAGE_MEDIA)
                    .build(),
                /* handleAudioFocus= */ true  // 오디오 포커스 자동 관리
            )
            .setHandleAudioBecomingNoisy(true)  // 이어폰 뽑으면 자동 일시정지
            .build()

        // MediaSession에 Player 연결
        mediaSession = MediaSession.Builder(this, player)
            .setCallback(MediaSessionCallback())
            .build()
    }

    // 클라이언트(Activity)에서 bindService()를 호출하면 이 세션을 반환
    override fun onGetSession(
        controllerInfo: MediaSession.ControllerInfo
    ): MediaSession? = mediaSession

    override fun onDestroy() {
        mediaSession?.run {
            player.release()
            release()
            mediaSession = null
        }
        super.onDestroy()
    }

    override fun onTaskRemoved(rootIntent: Intent?) {
        val player = mediaSession?.player ?: return
        if (!player.playWhenReady || player.mediaItemCount == 0) {
            // 재생 중이 아니면 서비스 종료
            stopSelf()
        }
    }

    // 외부 클라이언트(Android Auto 등)의 커맨드 인가 제어
    private inner class MediaSessionCallback : MediaSession.Callback {
        override fun onAddMediaItems(
            mediaSession: MediaSession,
            controller: MediaSession.ControllerInfo,
            mediaItems: List<MediaItem>
        ): ListenableFuture<List<MediaItem>> {
            // URI 해석 등 실제 MediaItem으로 변환
            val resolvedItems = mediaItems.map { item ->
                item.buildUpon()
                    .setUri(item.requestMetadata.mediaUri)
                    .build()
            }
            return Futures.immediateFuture(resolvedItems)
        }
    }
}
```

`AndroidManifest.xml`에 서비스를 등록합니다:

```xml
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK" />

<application>
    <service
        android:name=".PlaybackService"
        android:foregroundServiceType="mediaPlayback"
        android:exported="true">
        <intent-filter>
            <action android:name="androidx.media3.session.MediaSessionService" />
        </intent-filter>
    </service>
</application>
```

## UI 연결: Activity에서 MediaController 사용

Activity에서는 `ExoPlayer`를 직접 생성하지 않고, `MediaController`를 통해 서비스의 플레이어를 제어합니다. `MediaController`는 `Player` 인터페이스를 구현하므로, `PlayerView`에 그대로 바인딩됩니다.

```kotlin
class PlayerActivity : AppCompatActivity() {

    private var mediaController: MediaController? = null
    private lateinit var playerView: PlayerView

    override fun onStart() {
        super.onStart()

        val sessionToken = SessionToken(
            this,
            ComponentName(this, PlaybackService::class.java)
        )

        val controllerFuture = MediaController.Builder(this, sessionToken)
            .buildAsync()

        controllerFuture.addListener({
            val controller = controllerFuture.get()
            mediaController = controller

            // MediaController를 PlayerView에 직접 바인딩
            playerView.player = controller

            // 재생할 미디어 설정
            val playlist = listOf(
                MediaItem.Builder()
                    .setUri("https://example.com/audio1.mp3")
                    .setMediaMetadata(
                        MediaMetadata.Builder()
                            .setTitle("첫 번째 트랙")
                            .setArtist("아티스트 A")
                            .setArtworkUri(Uri.parse("https://example.com/cover1.jpg"))
                            .build()
                    )
                    .build(),
                MediaItem.Builder()
                    .setUri("https://example.com/audio2.mp3")
                    .setMediaMetadata(
                        MediaMetadata.Builder()
                            .setTitle("두 번째 트랙")
                            .setArtist("아티스트 B")
                            .build()
                    )
                    .build()
            )

            controller.setMediaItems(playlist)
            controller.prepare()
            controller.play()

        }, MoreExecutors.directExecutor())
    }

    override fun onStop() {
        // PlayerView에서 플레이어 분리 (Service는 계속 실행)
        playerView.player = null
        mediaController?.release()
        mediaController = null
        super.onStop()
    }
}
```

## Jetpack Compose에서의 통합

Compose 환경에서는 `Player.Listener`를 활용하여 플레이어 상태를 Compose 상태로 관찰합니다. `DisposableEffect`로 리스너의 생명주기를 관리하고, `LaunchedEffect`로 재생 위치를 주기적으로 갱신합니다.

```kotlin
@Composable
fun MediaPlayerScreen(player: Player) {
    var isPlaying by remember { mutableStateOf(player.isPlaying) }
    var currentPosition by remember { mutableLongStateOf(player.currentPosition) }
    var duration by remember { mutableLongStateOf(player.duration) }
    var currentMediaMetadata by remember { mutableStateOf(player.mediaMetadata) }

    // Player 상태 변화를 Compose 상태로 동기화
    DisposableEffect(player) {
        val listener = object : Player.Listener {
            override fun onIsPlayingChanged(playing: Boolean) {
                isPlaying = playing
            }
            override fun onMediaMetadataChanged(metadata: MediaMetadata) {
                currentMediaMetadata = metadata
            }
            override fun onPlaybackStateChanged(playbackState: Int) {
                if (playbackState == Player.STATE_READY) {
                    duration = player.duration
                }
            }
        }
        player.addListener(listener)
        onDispose { player.removeListener(listener) }
    }

    // 재생 중일 때만 포지션을 500ms 간격으로 폴링
    LaunchedEffect(isPlaying) {
        while (isPlaying) {
            currentPosition = player.currentPosition
            delay(500)
        }
    }

    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        // 트랙 정보
        Text(
            text = currentMediaMetadata.title?.toString() ?: "알 수 없는 트랙",
            style = MaterialTheme.typography.headlineSmall,
            fontWeight = FontWeight.Bold
        )
        Text(
            text = currentMediaMetadata.artist?.toString() ?: "",
            style = MaterialTheme.typography.bodyMedium,
            color = MaterialTheme.colorScheme.onSurface.copy(alpha = 0.6f)
        )

        Spacer(modifier = Modifier.height(16.dp))

        // 시크 바
        Slider(
            value = if (duration > 0) currentPosition / duration.toFloat() else 0f,
            onValueChange = { fraction ->
                player.seekTo((duration * fraction).toLong())
            },
            modifier = Modifier.fillMaxWidth()
        )

        Row(
            modifier = Modifier.fillMaxWidth(),
            horizontalArrangement = Arrangement.SpaceBetween
        ) {
            Text(formatDuration(currentPosition), style = MaterialTheme.typography.bodySmall)
            Text(formatDuration(duration), style = MaterialTheme.typography.bodySmall)
        }

        Spacer(modifier = Modifier.height(16.dp))

        // 재생 컨트롤
        Row(
            horizontalArrangement = Arrangement.spacedBy(16.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            IconButton(onClick = { player.seekToPreviousMediaItem() }) {
                Icon(Icons.Default.SkipPrevious, contentDescription = "이전")
            }
            FilledIconButton(
                onClick = { if (isPlaying) player.pause() else player.play() },
                modifier = Modifier.size(64.dp)
            ) {
                Icon(
                    if (isPlaying) Icons.Default.Pause else Icons.Default.PlayArrow,
                    contentDescription = if (isPlaying) "일시정지" else "재생",
                    modifier = Modifier.size(32.dp)
                )
            }
            IconButton(onClick = { player.seekToNextMediaItem() }) {
                Icon(Icons.Default.SkipNext, contentDescription = "다음")
            }
        }
    }
}

private fun formatDuration(ms: Long): String {
    if (ms <= 0) return "0:00"
    val seconds = (ms / 1000) % 60
    val minutes = (ms / (1000 * 60)) % 60
    return "$minutes:${seconds.toString().padStart(2, '0')}"
}
```

## 주의사항 및 고급 팁

### 1. 스레딩 규칙 엄수

`ExoPlayer` 인스턴스는 반드시 단일 스레드(기본값: 메인 스레드)에서만 접근해야 합니다. `MediaSessionService` 내부에서도 `player` 메서드를 `CoroutineScope(Dispatchers.Main)`이나 `Handler(Looper.getMainLooper())`를 통해 호출하세요. Media3 1.5.0부터는 위반 시 `IllegalStateException`을 더 엄격하게 발생시킵니다.

### 2. AudioFocus 자동 관리

`setAudioAttributes()`에 `handleAudioFocus = true`를 설정하면 Media3가 오디오 포커스를 자동으로 관리합니다. 전화가 오거나 다른 앱이 오디오를 시작하면 자동으로 일시정지 또는 볼륨 덕킹(ducking)이 적용됩니다. 수동으로 `AudioManager.requestAudioFocus()`를 호출할 필요가 없습니다.

### 3. MediaController 릴리즈 시점

`MediaController.release()`를 호출해도 `PlaybackService`는 계속 실행됩니다. Activity/Fragment가 파괴될 때 컨트롤러만 해제하고, 서비스는 알림(Notification)을 통해 사용자가 명시적으로 중지할 때까지 유지됩니다. `onStop()`에서 `release()`하고 `onStart()`에서 재연결하는 패턴을 사용하세요.

### 4. 미디어 알림 자동 생성

`MediaSessionService`는 재생 중에 미디어 스타일 알림을 자동으로 표시합니다. 커스터마이징이 필요하다면 `DefaultMediaNotificationProvider`를 상속받아 `setMediaNotificationProvider()`로 설정하세요. `MediaMetadata`에 `artworkUri`를 올바르게 설정하면 알림에도 앨범 아트가 표시됩니다.

### 5. HLS/DASH 스트리밍

HLS를 재생하려면 `media3-exoplayer-hls` 의존성을 추가하고 `.m3u8` URI를 `MediaItem.fromUri()`에 그대로 넘기면 됩니다. Media3는 URI의 Content-Type 또는 파일 확장자를 기반으로 자동으로 적절한 소스를 선택합니다. DASH(`.mpd`), SmoothStreaming 모두 동일한 방식으로 동작합니다.

### 6. CacheDataSource로 오프라인 재생 지원

`SimpleCache`와 `CacheDataSource`를 조합하면 스트리밍 콘텐츠를 로컬에 캐시하여 오프라인 재생을 지원할 수 있습니다. `ExoPlayer.Builder`에 `setMediaSourceFactory()`를 통해 캐싱 레이어를 주입하면, 별도의 다운로드 로직 없이 재생하면서 자동으로 캐시됩니다.

## 마치며

Jetpack Media3는 기존 Android 미디어 재생의 복잡성을 `Player` 인터페이스 하나로 통합하여, 백그라운드 재생·Android Auto 지원·Compose UI 연동까지 일관된 방식으로 구현할 수 있게 해줍니다. 특히 `MediaSessionService`를 중심으로 한 아키텍처는 서비스와 UI를 깔끔하게 분리하면서도 `MediaController`를 통해 투명하게 연결되어, 기존 `ExoPlayer + MediaBrowserServiceCompat` 조합에 비해 코드량을 크게 줄여줍니다. 새로운 오디오/비디오 재생 기능을 개발한다면 Media3를 기본 선택으로 고려하세요.

## 참고 자료
- [Introduction to Jetpack Media3](https://developer.android.com/media/media3)
- [Media3 ExoPlayer 개요](https://developer.android.com/media/media3/exoplayer)
- [Getting Started with ExoPlayer](https://developer.android.com/media/media3/exoplayer/hello-world)
- [Media Player App 구현 가이드](https://developer.android.com/media/implement/playback-app)
