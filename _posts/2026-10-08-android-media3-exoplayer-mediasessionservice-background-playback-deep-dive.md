---
layout: post
title: "Android Media3 심화: ExoPlayer·MediaSession·MediaSessionService로 백그라운드 오디오 재생 앱 완전 구현"
date: 2026-10-08
categories: [android]
tags: [android, media3, exoplayer, mediasession, mediasessionservice, background-playback, kotlin]
---

오디오·비디오 재생 기능은 현대 모바일 앱에서 빠질 수 없는 요소입니다. 그러나 단순한 재생을 넘어 화면이 꺼진 상태에서도 음악을 틀고, 시스템 미디어 컨트롤에서 일시정지하고, Bluetooth 헤드셋 버튼으로 다음 곡을 넘기는 완전한 미디어 앱을 만들려면 Android의 미디어 세션 아키텍처를 깊이 이해해야 합니다. Jetpack Media3는 이 복잡한 문제를 통합적으로 해결하는 라이브러리입니다.

## 1. Jetpack Media3란 무엇인가

Jetpack Media3는 Android 미디어 재생 생태계를 하나로 통합한 라이브러리 집합입니다. 기존에는 ExoPlayer, MediaCompat, MediaSession API가 각기 따로 존재했고, 이들을 연결하는 "커넥터" 코드가 별도로 필요했습니다. Media3는 이 파편화를 해소하고 단일 `Player` 인터페이스를 중심으로 모든 컴포넌트를 통합합니다.

### 핵심 컴포넌트 관계도

```
[UI Layer]             [Session Layer]          [Player Layer]
PlayerView ──────▶ MediaController ──────▶ MediaSession ──────▶ ExoPlayer
(or Compose)          (원격 제어)              (외부 공개)          (실제 재생)
                            │
                     MediaSessionService
                       (백그라운드 Service)
```

| 클래스 | 역할 |
|---|---|
| `ExoPlayer` | `Player` 인터페이스의 기본 구현체. 실제 디코딩·렌더링 담당 |
| `MediaSession` | 플레이어 상태를 시스템/외부 앱에 공개하고 명령을 수신 |
| `MediaSessionService` | 백그라운드에서 세션과 플레이어를 호스팅하는 Service |
| `MediaController` | UI나 외부 앱이 세션에 명령을 보내는 클라이언트 |
| `MediaLibraryService` | MediaSessionService 확장형; 콘텐츠 라이브러리 탐색 지원 |

## 2. 왜 Media3가 필요한가

### 2-1. 백그라운드 재생의 복잡성

Activity가 파괴되거나 화면이 꺼져도 음악이 계속 재생되려면 플레이어가 Activity 생명주기와 분리된 곳, 즉 **Service** 안에 있어야 합니다. 그냥 Service가 아니라 포그라운드 서비스(Foreground Service)로 실행해 OS가 자원 부족 시에도 프로세스를 종료하지 않도록 해야 합니다.

### 2-2. 외부 컨트롤러와의 통신

- Android 잠금 화면 미디어 컨트롤
- 시스템 상태바 미디어 알림
- Bluetooth 헤드셋·스피커 미디어 버튼
- Google Assistant, Android Auto, Wear OS
- 다른 앱 (팟캐스트 앱에서 다른 미디어 앱 제어)

이 모든 클라이언트는 `MediaSession`이라는 표준 채널을 통해 플레이어와 통신합니다. `MediaSessionService`는 이 채널을 시스템에 등록하고 항상 접근 가능하게 유지하는 역할을 합니다.

### 2-3. 구 ExoPlayer 대비 달라진 점

구 ExoPlayer(독립 라이브러리)에서 Media3로 마이그레이션 시 패키지명이 바뀝니다:
- `com.google.android.exoplayer2.*` → `androidx.media3.*`

그 외 핵심 API는 거의 동일하지만, `MediaSession`, `MediaController`의 구현이 크게 단순화됐습니다.

## 3. 의존성 설정

```kotlin
// build.gradle.kts (app)
dependencies {
    val media3Version = "1.5.0"
    implementation("androidx.media3:media3-exoplayer:$media3Version")
    implementation("androidx.media3:media3-session:$media3Version")
    implementation("androidx.media3:media3-ui:$media3Version")
    // HLS, DASH 등 적응형 스트리밍이 필요한 경우
    implementation("androidx.media3:media3-exoplayer-hls:$media3Version")
    implementation("androidx.media3:media3-exoplayer-dash:$media3Version")
}
```

`compileOptions`에서 Java 8 이상을 활성화해야 합니다:

```kotlin
android {
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_1_8
        targetCompatibility = JavaVersion.VERSION_1_8
    }
}
```

## 4. 실제 구현 예제 1: PlaybackService (MediaSessionService)

백그라운드 재생의 핵심인 `MediaSessionService`를 구현합니다.

```kotlin
import androidx.annotation.OptIn
import androidx.media3.common.AudioAttributes
import androidx.media3.common.C
import androidx.media3.common.MediaItem
import androidx.media3.common.util.UnstableApi
import androidx.media3.exoplayer.ExoPlayer
import androidx.media3.session.MediaSession
import androidx.media3.session.MediaSessionService

class PlaybackService : MediaSessionService() {

    private var mediaSession: MediaSession? = null

    @OptIn(UnstableApi::class)
    override fun onCreate() {
        super.onCreate()

        // AudioFocus와 오디오 속성 설정
        val audioAttributes = AudioAttributes.Builder()
            .setUsage(C.USAGE_MEDIA)
            .setContentType(C.AUDIO_CONTENT_TYPE_MUSIC)
            .build()

        val player = ExoPlayer.Builder(this)
            .setAudioAttributes(audioAttributes, /* handleAudioFocus= */ true)
            .setHandleAudioBecomingNoisy(true) // 이어폰 제거 시 자동 일시정지
            .build()

        mediaSession = MediaSession.Builder(this, player)
            .setCallback(MediaSessionCallback())
            .build()
    }

    override fun onGetSession(
        controllerInfo: MediaSession.ControllerInfo
    ): MediaSession? = mediaSession

    override fun onTaskRemoved(rootIntent: android.content.Intent?) {
        val player = mediaSession?.player ?: return
        // 재생 중이 아닐 때는 서비스를 종료
        if (!player.playWhenReady
            || player.mediaItemCount == 0
            || player.playbackState == androidx.media3.common.Player.STATE_ENDED
        ) {
            stopSelf()
        }
    }

    override fun onDestroy() {
        mediaSession?.run {
            player.release()
            release()
            mediaSession = null
        }
        super.onDestroy()
    }

    // 외부 컨트롤러의 명령을 처리하는 콜백
    private inner class MediaSessionCallback : MediaSession.Callback {
        override fun onAddMediaItems(
            mediaSession: MediaSession,
            controller: MediaSession.ControllerInfo,
            mediaItems: List<MediaItem>
        ): com.google.common.util.concurrent.ListenableFuture<List<MediaItem>> {
            // MediaItem의 URI를 실제 재생 가능한 형태로 채워준다
            val resolvedItems = mediaItems.map { item ->
                item.buildUpon()
                    .setUri(item.mediaMetadata.mediaUri ?: item.requestMetadata.mediaUri)
                    .build()
            }
            return com.google.common.util.concurrent.Futures.immediateFuture(resolvedItems)
        }
    }
}
```

### AndroidManifest.xml 설정

```xml
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK" />
<!-- Android 13+ 미디어 알림 권한 -->
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />

<application ...>
    <service
        android:name=".PlaybackService"
        android:foregroundServiceType="mediaPlayback"
        android:exported="true">
        <intent-filter>
            <!-- Media3 클라이언트가 서비스를 발견하는 액션 -->
            <action android:name="androidx.media3.session.MediaSessionService" />
            <!-- 구형 MediaBrowser 호환 클라이언트 지원 -->
            <action android:name="android.media.browse.MediaBrowserService" />
        </intent-filter>
    </service>

    <!-- 재생 중단 후 재개 기능을 위한 BroadcastReceiver -->
    <receiver
        android:name="androidx.media3.session.MediaButtonReceiver"
        android:exported="true">
        <intent-filter>
            <action android:name="android.intent.action.MEDIA_BUTTON" />
        </intent-filter>
    </receiver>
</application>
```

## 5. 실제 구현 예제 2: UI에서 MediaController 연결

Activity나 Fragment에서 `MediaController`를 통해 Service의 플레이어를 제어합니다.

```kotlin
import androidx.annotation.OptIn
import androidx.appcompat.app.AppCompatActivity
import androidx.lifecycle.lifecycleScope
import androidx.media3.common.MediaItem
import androidx.media3.common.MediaMetadata
import androidx.media3.common.util.UnstableApi
import androidx.media3.session.MediaController
import androidx.media3.session.SessionToken
import androidx.media3.ui.PlayerView
import com.google.common.util.concurrent.ListenableFuture
import com.google.common.util.concurrent.MoreExecutors
import kotlinx.coroutines.launch

class PlayerActivity : AppCompatActivity() {

    private lateinit var playerView: PlayerView
    private var controllerFuture: ListenableFuture<MediaController>? = null
    private var controller: MediaController? = null

    @OptIn(UnstableApi::class)
    override fun onStart() {
        super.onStart()
        val sessionToken = SessionToken(
            this,
            android.content.ComponentName(this, PlaybackService::class.java)
        )
        controllerFuture = MediaController.Builder(this, sessionToken).buildAsync()
        controllerFuture?.addListener(
            {
                controller = controllerFuture?.get()
                playerView.player = controller
                // 재생목록 구성
                setupPlaylist()
            },
            MoreExecutors.directExecutor()
        )
    }

    private fun setupPlaylist() {
        val controller = controller ?: return

        val items = listOf(
            buildMediaItem(
                id = "track_001",
                title = "아름다운 노래",
                artist = "홍길동",
                uri = "https://example.com/song1.mp3",
                artworkUri = "https://example.com/art1.jpg"
            ),
            buildMediaItem(
                id = "track_002",
                title = "또 다른 노래",
                artist = "김철수",
                uri = "https://example.com/song2.mp3",
                artworkUri = "https://example.com/art2.jpg"
            )
        )

        controller.setMediaItems(items)
        controller.prepare()
        controller.play()
    }

    private fun buildMediaItem(
        id: String,
        title: String,
        artist: String,
        uri: String,
        artworkUri: String
    ): MediaItem {
        val metadata = MediaMetadata.Builder()
            .setTitle(title)
            .setArtist(artist)
            .setArtworkUri(android.net.Uri.parse(artworkUri))
            .build()

        return MediaItem.Builder()
            .setMediaId(id)
            .setRequestMetadata(
                MediaItem.RequestMetadata.Builder()
                    .setMediaUri(android.net.Uri.parse(uri))
                    .build()
            )
            .setMediaMetadata(metadata)
            .build()
    }

    override fun onStop() {
        super.onStop()
        playerView.player = null
        controller?.release()
        controllerFuture?.let {
            MediaController.releaseFuture(it)
        }
        controllerFuture = null
        controller = null
    }
}
```

### Jetpack Compose에서 사용하기

```kotlin
@Composable
fun PlayerScreen(controller: MediaController?) {
    val isPlaying by remember(controller) {
        derivedStateOf { controller?.isPlaying ?: false }
    }

    // Player 상태를 관찰하는 Listener 등록
    DisposableEffect(controller) {
        val listener = object : androidx.media3.common.Player.Listener {
            override fun onIsPlayingChanged(playing: Boolean) {
                // 상태 변화 처리
            }
            override fun onMediaItemTransition(
                mediaItem: androidx.media3.common.MediaItem?,
                reason: Int
            ) {
                // 트랙 변경 시 처리
            }
        }
        controller?.addListener(listener)
        onDispose { controller?.removeListener(listener) }
    }

    Column(
        modifier = Modifier.fillMaxWidth(),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        IconButton(
            onClick = {
                if (controller?.isPlaying == true) controller.pause()
                else controller?.play()
            }
        ) {
            Icon(
                imageVector = if (isPlaying) Icons.Default.Pause else Icons.Default.PlayArrow,
                contentDescription = if (isPlaying) "일시정지" else "재생"
            )
        }
        Row {
            IconButton(onClick = { controller?.seekToPreviousMediaItem() }) {
                Icon(Icons.Default.SkipPrevious, contentDescription = "이전 곡")
            }
            IconButton(onClick = { controller?.seekToNextMediaItem() }) {
                Icon(Icons.Default.SkipNext, contentDescription = "다음 곡")
            }
        }
    }
}
```

## 6. 재생 재개 기능 (Playback Resumption)

사용자가 앱을 강제 종료한 뒤 시스템 미디어 알림에서 다시 재생할 수 있도록 재개 기능을 구현합니다.

```kotlin
class PlaybackService : MediaSessionService() {

    override fun onCreate() {
        super.onCreate()
        // ... player, mediaSession 초기화 ...

        mediaSession = MediaSession.Builder(this, player)
            .setCallback(ResumptionCallback())
            .build()
    }

    private inner class ResumptionCallback : MediaSession.Callback {
        override fun onPlaybackResumption(
            mediaSession: MediaSession,
            controller: MediaSession.ControllerInfo
        ): com.google.common.util.concurrent.ListenableFuture<MediaSession.MediaItemsWithStartPosition> {
            // SharedPreferences나 Room DB에서 마지막 재생 목록을 불러온다
            val prefs = getSharedPreferences("playback_prefs", MODE_PRIVATE)
            val lastUri = prefs.getString("last_uri", null)
            val lastPosition = prefs.getLong("last_position", 0L)

            if (lastUri == null) {
                return com.google.common.util.concurrent.Futures.immediateFailedFuture(
                    UnsupportedOperationException()
                )
            }

            val item = MediaItem.fromUri(lastUri)
            val result = MediaSession.MediaItemsWithStartPosition(
                listOf(item),
                /* startIndex = */ 0,
                /* startPositionMs = */ lastPosition
            )
            return com.google.common.util.concurrent.Futures.immediateFuture(result)
        }
    }

    // 재생 위치를 주기적으로 저장하는 리스너
    private fun savePlaybackPosition(player: ExoPlayer) {
        val prefs = getSharedPreferences("playback_prefs", MODE_PRIVATE)
        player.addListener(object : androidx.media3.common.Player.Listener {
            override fun onPositionDiscontinuity(
                oldPosition: androidx.media3.common.Player.PositionInfo,
                newPosition: androidx.media3.common.Player.PositionInfo,
                reason: Int
            ) {
                prefs.edit()
                    .putString("last_uri", player.currentMediaItem?.localConfiguration?.uri?.toString())
                    .putLong("last_position", player.currentPosition)
                    .apply()
            }
        })
    }
}
```

## 7. 주의사항과 팁

### 7-1. 스레드 안전성

`ExoPlayer`는 **단일 스레드(기본: 메인 스레드)**에서만 접근해야 합니다. 백그라운드 스레드에서 player 메서드를 호출하면 `IllegalStateException`이 발생합니다. `MediaController`는 내부적으로 IPC를 처리하므로 원격 Service의 플레이어도 메인 스레드에서 안전하게 제어할 수 있습니다.

### 7-2. AudioFocus 관리

`setAudioAttributes(..., handleAudioFocus = true)`로 설정하면 Media3가 AudioFocus를 자동 관리합니다. 전화가 오면 자동으로 볼륨을 낮추거나(duck) 일시정지하고, 통화가 끝나면 재개합니다. `setHandleAudioBecomingNoisy(true)`를 함께 설정하면 이어폰 제거 시 자동으로 일시정지합니다.

### 7-3. 알림 커스터마이징

Android 13(API 33) 이상에서는 시스템 UI가 `MediaMetadata`와 `CommandButton` 목록을 기반으로 알림을 직접 구성합니다. `MediaNotification.Provider`로 알림을 완전히 커스터마이징하는 방식은 권장되지 않으며, 대신 다음 방법을 사용하세요:

```kotlin
// 미디어 세션에서 표시할 버튼 지정
mediaSession = MediaSession.Builder(this, player)
    .setMediaButtonPreferences(
        listOf(
            CommandButton.Builder(CommandButton.ICON_SKIP_BACK_15)
                .setPlayerCommand(androidx.media3.common.Player.COMMAND_SEEK_BACK)
                .build(),
            CommandButton.Builder(CommandButton.ICON_SKIP_FORWARD_30)
                .setPlayerCommand(androidx.media3.common.Player.COMMAND_SEEK_FORWARD)
                .build()
        )
    )
    .build()
```

### 7-4. Android Auto 지원

Android Auto 지원을 위해서는 `MediaSessionService` 대신 `MediaLibraryService`를 사용하고, `MediaLibrarySession.Callback.onGetLibraryRoot()`를 구현하여 탐색 가능한 콘텐츠 트리를 제공해야 합니다. 또한 `AndroidManifest.xml`의 `<meta-data>`에 Auto 지원을 명시해야 합니다.

### 7-5. 메모리 누수 방지

`MediaController`를 `onStop()`에서 반드시 `release()`해야 합니다. `ListenableFuture`로 비동기 연결 시 `MediaController.releaseFuture(future)`를 사용하면 컨트롤러가 아직 연결되지 않은 경우도 안전하게 정리됩니다.

### 7-6. ViewModel에서 MediaController 관리

Activity 재생성(화면 회전)마다 MediaController를 재연결하는 오버헤드를 줄이려면 ViewModel에서 관리합니다:

```kotlin
class PlayerViewModel(application: Application) : AndroidViewModel(application) {
    private var controllerFuture: ListenableFuture<MediaController>? = null
    var controller: MediaController? = null
        private set

    fun connect() {
        val token = SessionToken(
            getApplication(),
            android.content.ComponentName(getApplication(), PlaybackService::class.java)
        )
        controllerFuture = MediaController.Builder(getApplication(), token).buildAsync()
        controllerFuture?.addListener(
            { controller = controllerFuture?.get() },
            MoreExecutors.directExecutor()
        )
    }

    override fun onCleared() {
        controller?.release()
        controllerFuture?.let { MediaController.releaseFuture(it) }
        super.onCleared()
    }
}
```

## 정리

Jetpack Media3는 ExoPlayer, MediaSession, UI 컴포넌트를 단일 `Player` 인터페이스로 통합하여 완전한 미디어 재생 앱 구현을 크게 단순화합니다. 핵심은 다음 세 가지 관계입니다:

1. **`ExoPlayer`** — 실제 디코딩과 재생을 담당하는 엔진
2. **`MediaSession` + `MediaSessionService`** — 백그라운드에서 플레이어를 호스팅하고 외부 클라이언트에 공개
3. **`MediaController`** — UI나 외부 앱이 세션에 명령을 전달하는 클라이언트

AudioFocus, 알림, 재생 재개, Bluetooth 버튼 처리는 Media3가 대부분 자동으로 처리하므로, 개발자는 비즈니스 로직(재생목록 구성, 메타데이터 제공)에 집중할 수 있습니다.

## 참고 자료
- [Jetpack Media3 소개 — Android Developers](https://developer.android.com/media/media3)
- [MediaSessionService로 백그라운드 재생 — Android Developers](https://developer.android.com/media/media3/session/background-playback)
- [ExoPlayer 시작하기 — Android Developers](https://developer.android.com/media/media3/exoplayer/hello-world)
- [Media3 샘플 앱 — GitHub AndroidX](https://github.com/androidx/media/tree/release/demos/session)
