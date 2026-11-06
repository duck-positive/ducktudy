---
layout: post
title: "Android AudioFocus 완전 정복: AudioManager, AudioFocusRequest, AudioAttributes 심화"
date: 2026-11-06
categories: [android]
tags: [android, audiofocus, audiomanager, audioattributes, media3, exoplayer, kotlin]
---

## 개요

스마트폰에서 음악을 듣다가 전화가 오면 음악이 멈추고, 통화가 끝나면 다시 재생됩니다. 네비게이션 안내 음성이 나올 때는 음악 볼륨이 자동으로 줄어들었다가 복원됩니다. 이 모든 것이 **AudioFocus** 메커니즘 덕분입니다.

Android는 단일 오디오 스트림에 여러 앱이 동시에 접근할 수 있도록 허용하지만, 사용자 경험 측면에서 한 번에 하나의 오디오 소스에만 집중(focus)하도록 협력을 권장합니다. AudioFocus는 이 협력 프로토콜의 핵심입니다.

이 글에서는 AudioFocus의 동작 원리부터 `AudioFocusRequest` Builder API, `AudioAttributes`, Media3/ExoPlayer 연동까지 실무에서 반드시 알아야 할 내용을 깊이 있게 다룹니다.

---

## AudioFocus가 왜 필요한가

### 문제 상황

AudioFocus가 없거나 제대로 구현되지 않으면 다음과 같은 문제가 발생합니다:

- **다중 오디오 충돌**: 음악 앱과 팟캐스트 앱이 동시에 소리를 출력해 엉망이 됩니다.
- **전화 중 음악 재생**: 통화 중에도 음악이 계속 재생되어 사용자를 당황스럽게 합니다.
- **안내 음성 묻힘**: 네비게이션 안내 음성이 큰 볼륨의 음악에 묻혀 들리지 않습니다.

### Android 버전별 정책 변화

| Android 버전 | 정책 |
|---|---|
| API 25 이하 | 앱이 직접 duck/pause 처리. 시스템 미강제. |
| API 26 ~ 30 | `AudioFocusRequest` Builder API 도입. 자동 ducking 지원. |
| API 31 이상 | 시스템이 fade-out 강제 적용. |
| API 35 이상 | 포그라운드 서비스 또는 최상위 앱만 focus 요청 가능. |

API 35 (Android 15) 이상을 타겟팅하는 앱은 반드시 Foreground Service 또는 최상위 앱 상태에서만 `requestAudioFocus()`를 호출해야 합니다. 그렇지 않으면 `AUDIOFOCUS_REQUEST_FAILED`가 반환됩니다.

---

## 핵심 개념: Focus Gain 타입

AudioFocus를 요청할 때 네 가지 gain 타입 중 하나를 지정합니다:

| Gain 타입 | 의미 | 사용 사례 |
|---|---|---|
| `AUDIOFOCUS_GAIN` | 장기적인 독점 focus. 이전 소유자는 중단 예정. | 음악 플레이어, 비디오 앱 |
| `AUDIOFOCUS_GAIN_TRANSIENT` | 짧은 시간 동안만 독점. 이전 소유자는 일시 정지. | 알람, VoIP 통화 |
| `AUDIOFOCUS_GAIN_TRANSIENT_MAY_DUCK` | 짧은 시간 + 이전 소유자 볼륨 감소 허용. | 네비게이션 안내, 알림 소리 |
| `AUDIOFOCUS_GAIN_TRANSIENT_EXCLUSIVE` | 짧은 시간 독점 + 다른 모든 오디오 차단. | 음성 인식, 오디오 녹음 |

---

## 실제 구현 예제 1: AudioFocusRequest 완전 구현

아래 예제는 음악 플레이어 앱에서 AudioFocus를 올바르게 요청하고 반환하는 전체 구현입니다.

```kotlin
import android.content.Context
import android.media.AudioAttributes
import android.media.AudioFocusRequest
import android.media.AudioManager
import android.os.Build
import android.os.Handler
import android.os.Looper

class AudioFocusManager(private val context: Context) {

    private val audioManager = context.getSystemService(Context.AUDIO_SERVICE) as AudioManager
    private val handler = Handler(Looper.getMainLooper())

    private val focusLock = Any()
    private var playbackDelayed = false
    private var resumeOnFocusGain = false

    // AudioAttributes: 오디오 용도와 콘텐츠 타입 명시
    private val audioAttributes = AudioAttributes.Builder()
        .setUsage(AudioAttributes.USAGE_MEDIA)
        .setContentType(AudioAttributes.CONTENT_TYPE_MUSIC)
        .build()

    // AudioFocusRequest: API 26+ 방식
    private val focusRequest: AudioFocusRequest = AudioFocusRequest.Builder(AudioManager.AUDIOFOCUS_GAIN)
        .setAudioAttributes(audioAttributes)
        .setAcceptsDelayedFocusGain(true)   // 전화 중 지연 granted 허용
        .setWillPauseWhenDucked(false)       // duck 시 시스템에게 맡김 (음악 앱의 경우)
        .setOnAudioFocusChangeListener(::onAudioFocusChange, handler)
        .build()

    var onPlay: () -> Unit = {}
    var onPause: () -> Unit = {}
    var onStop: () -> Unit = {}

    fun requestFocus(): Boolean {
        val result = audioManager.requestAudioFocus(focusRequest)
        return synchronized(focusLock) {
            when (result) {
                AudioManager.AUDIOFOCUS_REQUEST_GRANTED -> {
                    playbackDelayed = false
                    onPlay()
                    true
                }
                AudioManager.AUDIOFOCUS_REQUEST_DELAYED -> {
                    // 전화 중이거나 시스템이 잠근 경우 — 나중에 granted 될 예정
                    playbackDelayed = true
                    false
                }
                else -> {
                    // AUDIOFOCUS_REQUEST_FAILED
                    false
                }
            }
        }
    }

    fun abandonFocus() {
        audioManager.abandonAudioFocusRequest(focusRequest)
        synchronized(focusLock) {
            resumeOnFocusGain = false
            playbackDelayed = false
        }
    }

    private fun onAudioFocusChange(focusChange: Int) {
        when (focusChange) {
            AudioManager.AUDIOFOCUS_GAIN -> {
                // Focus 복구: 지연 재생 또는 중단 후 복귀
                val shouldResume = synchronized(focusLock) {
                    val resume = playbackDelayed || resumeOnFocusGain
                    playbackDelayed = false
                    resumeOnFocusGain = false
                    resume
                }
                if (shouldResume) onPlay()
            }
            AudioManager.AUDIOFOCUS_LOSS -> {
                // 영구 손실: 사용자가 명시적으로 재생을 눌러야 복구
                synchronized(focusLock) {
                    resumeOnFocusGain = false
                    playbackDelayed = false
                }
                onPause()
                // 30초 후 완전 중지 (사용자가 재개하지 않으면)
                handler.postDelayed({ onStop() }, 30_000L)
            }
            AudioManager.AUDIOFOCUS_LOSS_TRANSIENT -> {
                // 일시적 손실 (예: VoIP 통화)
                synchronized(focusLock) {
                    resumeOnFocusGain = true // focus 복구 시 자동 재개
                }
                onPause()
            }
            AudioManager.AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK -> {
                // 볼륨을 낮춰도 되는 일시적 손실 (예: 알림음)
                // setWillPauseWhenDucked(false)이면 시스템이 자동 처리하므로 콜백 없음
                // setWillPauseWhenDucked(true)면 여기서 직접 pause 처리
            }
        }
    }
}
```

### 사용 방법

```kotlin
// Activity 또는 Service에서
val focusManager = AudioFocusManager(this).apply {
    onPlay = { mediaPlayer.start() }
    onPause = { mediaPlayer.pause() }
    onStop = { mediaPlayer.stop() }
}

// 재생 시작 전
if (focusManager.requestFocus()) {
    // 즉시 재생 가능
} else {
    // 지연됨 — onPlay 콜백이 나중에 자동으로 호출됨
}

// 재생 완료 또는 앱 종료 시
focusManager.abandonFocus()
```

---

## 실제 구현 예제 2: Media3/ExoPlayer 자동 AudioFocus 관리

Media3를 사용하는 경우, ExoPlayer에 AudioFocus 관리를 위임하면 직접 구현 없이도 올바르게 동작합니다.

```kotlin
import androidx.media3.common.AudioAttributes
import androidx.media3.common.C
import androidx.media3.exoplayer.ExoPlayer

class PlayerActivity : AppCompatActivity() {

    private lateinit var player: ExoPlayer

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // Media3 AudioAttributes (android.media.AudioAttributes와 다름)
        val audioAttributes = AudioAttributes.Builder()
            .setUsage(C.USAGE_MEDIA)
            .setContentType(C.AUDIO_CONTENT_TYPE_MUSIC)
            .build()

        player = ExoPlayer.Builder(this)
            .build()
            .apply {
                // handleAudioFocus = true: ExoPlayer가 AudioFocus를 자동 관리
                setAudioAttributes(audioAttributes, /* handleAudioFocus= */ true)
            }

        // 이제 requestAudioFocus / abandonAudioFocusRequest를 직접 호출하지 않아도 됨
        // ExoPlayer가 play/pause 상태에 맞춰 자동으로 처리
    }

    override fun onDestroy() {
        super.onDestroy()
        player.release()
    }
}
```

### 팟캐스트/오디오북 앱에서의 설정

음성 콘텐츠의 경우 `CONTENT_TYPE_SPEECH`를 사용해야 시스템이 자동 ducking 대신 pause를 호출합니다.

```kotlin
// 팟캐스트 / 오디오북 전용 설정
val speechAudioAttributes = AudioAttributes.Builder()
    .setUsage(C.USAGE_MEDIA)
    .setContentType(C.AUDIO_CONTENT_TYPE_SPEECH) // 핵심: 음성 콘텐츠임을 명시
    .build()

player.setAudioAttributes(speechAudioAttributes, /* handleAudioFocus= */ true)

// 이렇게 하면:
// - 다른 앱이 AUDIOFOCUS_GAIN_TRANSIENT_MAY_DUCK으로 focus 요청 시
//   시스템이 duck하지 않고 onAudioFocusChange(AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK) 호출
// - 앱에서 직접 pause 처리 가능
```

---

## AudioAttributes 심화: Usage와 ContentType

`AudioAttributes`는 오디오의 "목적"과 "형태"를 시스템에 알려주는 메타데이터입니다.

### Usage 주요 상수

| Usage | 설명 |
|---|---|
| `USAGE_MEDIA` | 음악, 비디오 등 미디어 재생 |
| `USAGE_GAME` | 게임 오디오 |
| `USAGE_ALARM` | 알람 소리 |
| `USAGE_NOTIFICATION` | 알림 소리 |
| `USAGE_VOICE_COMMUNICATION` | 통화 음성 |
| `USAGE_ASSISTANCE_NAVIGATION_GUIDANCE` | 네비게이션 안내 |

### ContentType 주요 상수

| ContentType | 설명 |
|---|---|
| `CONTENT_TYPE_MUSIC` | 음악 |
| `CONTENT_TYPE_SPEECH` | 음성/대화 (팟캐스트, 오디오북) |
| `CONTENT_TYPE_MOVIE` | 영화 사운드트랙 |
| `CONTENT_TYPE_SONIFICATION` | 효과음 |

---

## BECOMING_NOISY: 이어폰 연결 해제 처리

AudioFocus와 함께 반드시 구현해야 하는 또 다른 항목이 `ACTION_AUDIO_BECOMING_NOISY`입니다. 이어폰이 갑자기 빠지면 오디오가 스피커로 전환되는데, 이때 자동으로 재생을 멈춰야 합니다.

```kotlin
import android.content.BroadcastReceiver
import android.content.Context
import android.content.Intent
import android.content.IntentFilter
import android.media.AudioManager

class NoisyReceiver(
    private val context: Context,
    private val onBecomingNoisy: () -> Unit
) {
    private val receiver = object : BroadcastReceiver() {
        override fun onReceive(context: Context, intent: Intent) {
            if (intent.action == AudioManager.ACTION_AUDIO_BECOMING_NOISY) {
                onBecomingNoisy()
            }
        }
    }

    private var isRegistered = false

    fun register() {
        if (!isRegistered) {
            val filter = IntentFilter(AudioManager.ACTION_AUDIO_BECOMING_NOISY)
            context.registerReceiver(receiver, filter)
            isRegistered = true
        }
    }

    fun unregister() {
        if (isRegistered) {
            context.unregisterReceiver(receiver)
            isRegistered = false
        }
    }
}

// 사용 예
val noisyReceiver = NoisyReceiver(this) {
    // 이어폰이 빠지면 즉시 일시정지
    player.pause()
    focusManager.abandonFocus()
}

// 재생 시작 시 등록
noisyReceiver.register()

// 재생 중지 또는 앱 종료 시 해제
noisyReceiver.unregister()
```

---

## 주의사항 및 실무 팁

### 1. AudioFocus는 협력적 시스템이다

API 30 이하에서는 시스템이 AudioFocus 준수를 강제하지 않습니다. 앱이 `AUDIOFOCUS_LOSS`를 받고도 계속 재생하는 경우 시스템이 막을 수 없습니다. 그러나 이는 사용자 경험을 심각하게 해치므로 반드시 올바르게 구현해야 합니다.

### 2. `resumeOnFocusGain` 플래그 관리

사용자가 직접 pause 버튼을 눌렀을 때는 `resumeOnFocusGain`을 `false`로 설정해야 합니다. 그렇지 않으면 사용자가 멈춰놓은 음악이 focus 복구 시 자동으로 재개되는 오동작이 발생합니다.

```kotlin
// 사용자가 직접 pause 버튼을 눌렀을 때
fun onUserPause() {
    synchronized(focusLock) {
        resumeOnFocusGain = false // 자동 재개 방지
    }
    player.pause()
    // focus는 abandon하지 않음 — 사용자가 재개할 수 있도록 유지
}
```

### 3. Delayed Focus 반드시 처리하기

전화 중에 앱을 시작하면 `AUDIOFOCUS_REQUEST_DELAYED`가 반환됩니다. 많은 앱이 이 케이스를 무시하여 통화 종료 후 재생이 시작되지 않는 버그를 만들어냅니다. `setAcceptsDelayedFocusGain(true)`와 `OnAudioFocusChangeListener` 조합으로 반드시 처리하세요.

### 4. Service에서 AudioFocus 관리하기

백그라운드 재생 앱은 Service(특히 `MediaSessionService` 또는 `MediaBrowserServiceCompat`) 내에서 AudioFocus를 관리해야 합니다. Activity에서 관리하면 Activity가 destroy될 때 focus가 함께 사라질 수 있습니다.

### 5. Android 15 (API 35) 대응

API 35 이상 타겟 앱은 Foreground Service 선언이 필수입니다. Media3의 `MediaSessionService`를 사용하면 이 요건을 자동으로 충족할 수 있습니다.

```xml
<!-- AndroidManifest.xml -->
<service
    android:name=".PlaybackService"
    android:foregroundServiceType="mediaPlayback"
    android:exported="true">
    <intent-filter>
        <action android:name="androidx.media3.session.MediaSessionService" />
    </intent-filter>
</service>
```

---

## 정리

AudioFocus는 Android 오디오 앱 개발에서 가장 기본적이면서도 쉽게 놓치는 요소입니다. 아래 체크리스트로 구현을 확인하세요:

- [ ] `AudioFocusRequest.Builder`로 focus 요청 (API 26+)
- [ ] `AudioAttributes`에 적절한 `Usage`와 `ContentType` 설정
- [ ] `AUDIOFOCUS_REQUEST_DELAYED` 케이스 처리
- [ ] `AUDIOFOCUS_LOSS` / `AUDIOFOCUS_LOSS_TRANSIENT` 별 동작 분기
- [ ] 사용자가 pause 시 `resumeOnFocusGain = false` 처리
- [ ] `ACTION_AUDIO_BECOMING_NOISY` BroadcastReceiver 등록/해제
- [ ] Service에서 AudioFocus 관리 (백그라운드 재생)
- [ ] API 35+ 타겟 시 Foreground Service 선언

Media3/ExoPlayer를 사용한다면 `setAudioAttributes(attrs, true)`로 대부분을 위임할 수 있지만, 커스텀 플레이어나 복잡한 오디오 세션을 다루는 경우 직접 구현이 필요합니다.

## 참고 자료

- [Manage audio focus — Android Developers](https://developer.android.com/media/optimize/audio-focus)
- [AudioFocusRequest API Reference — Android Developers](https://developer.android.com/reference/android/media/AudioFocusRequest)
- [Understanding Audio Focus (Part 3/3) — Android Developers Medium](https://medium.com/androiddevelopers/audio-focus-3-cdc09da9c122)
- [Media3 ExoPlayer setAudioAttributes — Android Developers](https://developer.android.com/reference/androidx/media3/exoplayer/ExoPlayer#setAudioAttributes(androidx.media3.common.AudioAttributes,boolean))
