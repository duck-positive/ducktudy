---
layout: post
title: "Android Picture-in-Picture(PiP) 심화: 자동 진입·커스텀 액션·Media3 연동 완전 정복"
date: 2026-08-18
categories: [android, flutter]
tags: [android, pip, picture-in-picture, media3, kotlin, jetpack]
---

PiP(Picture-in-Picture)는 동영상·화상통화·내비게이션 앱에서 사용자가 다른 앱을 사용하는 중에도 핵심 콘텐츠를 작은 플로팅 창으로 유지할 수 있게 해 주는 다중 창 기능입니다. Android 8.0(API 26)에서 도입된 이래 Android 12(API 31)에서 자동 진입·스무스 리사이징·원탭 컨트롤이 추가되고, Android 15에서는 이음새 없는(seamless) 전환이 더욱 정교해졌습니다. 이 글에서는 PiP의 내부 동작 원리부터 프로덕션 수준의 구현 패턴까지 단계적으로 파헤칩니다.

---

## 1. PiP의 핵심 개념과 버전별 진화

PiP 창은 시스템이 관리하는 플로팅 레이어 위에 Activity 전체가 렌더링되는 구조입니다. 일반 `Activity` 가 화면을 꽉 채우다가 PiP 모드로 전환되면, 시스템은 해당 Activity의 뷰 계층을 축소해 별도 SurfaceControl 레이어에 배치합니다. Activity의 생명주기는 변하지 않으며 `onPause()` 없이 `onPictureInPictureModeChanged()` 콜백만 호출됩니다.

| Android 버전 | 추가된 기능 |
|---|---|
| 8.0 (API 26) | PiP 기본 지원, `enterPictureInPictureMode()` |
| 10 (API 29) | `RemoteAction` 으로 커스텀 버튼 최대 3개 |
| 12 (API 31) | `setAutoEnterEnabled(true)` 자동 진입, `setSeamlessResizeEnabled`, 원탭 컨트롤, 핀치-투-줌 리사이즈 |
| 13 (API 33) | `setExpandedAspectRatio` 로 전체 화면 복귀 시 비율 지정 |
| 15 (API 35) | `setSourceRectHint` 애니메이션 정밀도 향상, Lottie 기반 전환 지원 |

---

## 2. 왜 PiP가 필요한가

**멀티태스킹 UX 유지**: 유튜브 프리미엄·넷플릭스·Duo 같은 앱에서 영상을 틀어 놓은 채 카카오톡을 읽을 수 있습니다. 이 경험이 빠지면 사용자는 앱 전환 순간 콘텐츠를 잃어 이탈률이 높아집니다.

**백그라운드 재생 대비 낮은 복잡도**: 완전한 백그라운드 재생은 `MediaSessionService` + Foreground Service + 알림을 모두 구현해야 합니다. 반면 PiP는 Activity가 화면에 유지되므로 재생 엔진에 별도 변경 없이 동일 코드 경로를 사용할 수 있습니다.

**내비게이션·화상통화**: 실시간으로 지도를 보면서 문자를 보낼 수 있고, 화상 통화 창을 띄운 채 다른 앱을 탐색하는 시나리오도 PiP로 간단히 해결됩니다.

---

## 3. 실제 구현 예제

### 3-1. Manifest 선언과 PictureInPictureParams 설정

가장 먼저 `AndroidManifest.xml` 에서 Activity에 PiP 지원을 선언하고, 시스템이 구성 변경을 Activity 재시작 없이 처리하도록 `configChanges` 를 지정합니다.

```xml
<activity
    android:name=".PlayerActivity"
    android:supportsPictureInPicture="true"
    android:configChanges="screenSize|smallestScreenSize|screenLayout|orientation|keyboardHidden"
    android:exported="false" />
```

다음은 `PlayerActivity` 의 전체 PiP 흐름입니다.

```kotlin
class PlayerActivity : AppCompatActivity() {

    private val playerView: PlayerView by lazy { findViewById(R.id.player_view) }

    // PiP 파라미터를 한 곳에서 빌드하는 헬퍼
    private fun buildPipParams(): PictureInPictureParams {
        // 16:9 영상에 맞춘 비율 (Rational 활용)
        val aspectRatio = Rational(16, 9)

        // 플레이어 뷰의 화면 좌표 → 매끄러운 진입 애니메이션
        val sourceRectHint = Rect().also { playerView.getGlobalVisibleRect(it) }

        // 커스텀 RemoteAction: 재생/일시정지 버튼
        val playPauseAction = RemoteAction(
            Icon.createWithResource(this, R.drawable.ic_pause),
            "일시정지",
            "영상 일시정지",
            PendingIntent.getBroadcast(
                this,
                REQUEST_CODE_PAUSE,
                Intent(ACTION_MEDIA_CONTROL).putExtra(EXTRA_CONTROL_TYPE, CONTROL_TYPE_PAUSE),
                PendingIntent.FLAG_IMMUTABLE or PendingIntent.FLAG_UPDATE_CURRENT
            )
        )

        return PictureInPictureParams.Builder()
            .setAspectRatio(aspectRatio)
            .setSourceRectHint(sourceRectHint)
            .setActions(listOf(playPauseAction))   // Android 10+
            .setAutoEnterEnabled(true)             // Android 12+: 홈 버튼 없이도 자동 진입
            .setSeamlessResizeEnabled(true)        // Android 12+: 리사이즈 시 블랙아웃 없음
            .build()
    }

    override fun onStart() {
        super.onStart()
        // 자동 진입을 위해 파라미터를 미리 등록
        setPictureInPictureParams(buildPipParams())
    }

    // Android 12 미만 기기에서 사용자가 뒤로 가기 또는 홈 누를 때 수동 진입
    @Deprecated("Use autoEnterEnabled on API 31+")
    override fun onUserLeaveHint() {
        if (packageManager.hasSystemFeature(PackageManager.FEATURE_PICTURE_IN_PICTURE)) {
            enterPictureInPictureMode(buildPipParams())
        }
    }

    override fun onPictureInPictureModeChanged(
        isInPictureInPictureMode: Boolean,
        newConfig: Configuration
    ) {
        super.onPictureInPictureModeChanged(isInPictureInPictureMode, newConfig)
        // PiP 모드: 컨트롤 UI 숨김 / 일반 모드: 복원
        playerView.useController = !isInPictureInPictureMode
    }

    override fun onStop() {
        super.onStop()
        // PiP 창이 닫힐 때 재생 중지
        if (!isInPictureInPictureMode) {
            // player.pause()
        }
    }

    companion object {
        const val ACTION_MEDIA_CONTROL = "media_control"
        const val EXTRA_CONTROL_TYPE = "control_type"
        const val CONTROL_TYPE_PAUSE = 1
        const val REQUEST_CODE_PAUSE = 101
    }
}
```

**핵심 포인트**: `setAutoEnterEnabled(true)` 를 사용하면 Android 12 이상에서 `onUserLeaveHint()` 를 구현할 필요가 없습니다. 시스템이 홈/최근 앱 제스처 감지 시 자동으로 PiP 전환을 수행합니다.

---

### 3-2. BroadcastReceiver로 커스텀 PiP 액션 처리

PiP 창 위에 나타나는 버튼(RemoteAction)은 PendingIntent를 통해 작동합니다. 가장 깔끔한 패턴은 Activity 내부에 `BroadcastReceiver` 를 등록하는 것입니다.

```kotlin
class PlayerActivity : AppCompatActivity() {

    private var isPlaying = true

    private val pipActionReceiver = object : BroadcastReceiver() {
        override fun onReceive(context: Context, intent: Intent) {
            if (intent.action != ACTION_MEDIA_CONTROL) return

            when (intent.getIntExtra(EXTRA_CONTROL_TYPE, 0)) {
                CONTROL_TYPE_PAUSE -> {
                    // player.pause()
                    isPlaying = false
                    updatePipActions()
                }
                CONTROL_TYPE_PLAY -> {
                    // player.play()
                    isPlaying = true
                    updatePipActions()
                }
            }
        }
    }

    override fun onStart() {
        super.onStart()
        registerReceiver(
            pipActionReceiver,
            IntentFilter(ACTION_MEDIA_CONTROL),
            RECEIVER_NOT_EXPORTED  // Android 13+ 보안 요구사항
        )
        setPictureInPictureParams(buildPipParams())
    }

    override fun onStop() {
        unregisterReceiver(pipActionReceiver)
        super.onStop()
    }

    // 재생 상태가 바뀔 때마다 PiP 버튼 아이콘을 업데이트
    private fun updatePipActions() {
        val (icon, label, controlType, requestCode) = if (isPlaying) {
            arrayOf(R.drawable.ic_pause, "일시정지", CONTROL_TYPE_PAUSE, REQUEST_CODE_PAUSE)
        } else {
            arrayOf(R.drawable.ic_play, "재생", CONTROL_TYPE_PLAY, REQUEST_CODE_PLAY)
        }

        val action = RemoteAction(
            Icon.createWithResource(this, icon as Int),
            label as String,
            label,
            PendingIntent.getBroadcast(
                this,
                requestCode as Int,
                Intent(ACTION_MEDIA_CONTROL).putExtra(EXTRA_CONTROL_TYPE, controlType),
                PendingIntent.FLAG_IMMUTABLE or PendingIntent.FLAG_UPDATE_CURRENT
            )
        )

        val params = PictureInPictureParams.Builder()
            .setActions(listOf(action))
            .build()

        // 전체 파라미터 재설정 없이 액션만 부분 업데이트
        setPictureInPictureParams(params)
    }

    companion object {
        const val ACTION_MEDIA_CONTROL = "media_control"
        const val EXTRA_CONTROL_TYPE = "control_type"
        const val CONTROL_TYPE_PAUSE = 1
        const val CONTROL_TYPE_PLAY  = 2
        const val REQUEST_CODE_PAUSE = 101
        const val REQUEST_CODE_PLAY  = 102
    }
}
```

`setPictureInPictureParams()` 는 PiP 모드 진입 전뿐만 아니라 **PiP 창이 표시되는 도중에도 호출**할 수 있습니다. 재생 상태가 바뀔 때마다 호출해 버튼 아이콘을 동적으로 교체하는 것이 권장 패턴입니다.

---

## 4. 주의사항 및 실전 팁

### 4-1. `onPause()` 에서 재생을 멈추지 마세요

일반적인 생명주기에서는 `onPause()` 에서 재생을 멈추는 것이 맞지만, PiP 모드 진입 시에도 `onPause()` 가 호출됩니다. 이때 재생을 멈추면 PiP 창이 검게 됩니다.

```kotlin
// 잘못된 방법
override fun onPause() {
    super.onPause()
    player.pause()  // PiP 진입 시에도 실행되어 화면이 멈춤
}

// 올바른 방법
override fun onPause() {
    super.onPause()
    if (!isInPictureInPictureMode) {
        player.pause()
    }
}
```

### 4-2. 태블릿·폴더블 기기에서 aspectRatio 조심

`setAspectRatio()` 에 전달하는 비율은 `android:minAspectRatio` (0.418 ≈ 1:2.39)와 `android:maxAspectRatio` (2.5 ≈ 21:9) 사이여야 합니다. 범위를 벗어나면 `IllegalArgumentException` 이 발생합니다.

```kotlin
// 안전한 클램핑 예시
fun safeAspectRatio(width: Int, height: Int): Rational {
    val ratio = width.toFloat() / height.toFloat()
    val clamped = ratio.coerceIn(0.419f, 2.499f)
    // 분수로 근사
    return if (clamped >= 1f)
        Rational((clamped * 100).toInt(), 100)
    else
        Rational(100, (100 / clamped).toInt())
}
```

### 4-3. `sourceRectHint` 로 매끄러운 전환 애니메이션 구현

`setSourceRectHint()` 는 현재 화면에서 PiP 창이 시작될 위치를 시스템에 힌트로 제공합니다. 이를 생략하면 Activity 전체가 축소되는 애니메이션이 실행되어 어색합니다. 플레이어 뷰의 글로벌 좌표를 실시간으로 계산해 전달하세요.

```kotlin
playerView.addOnLayoutChangeListener { _, left, top, right, bottom,
    oldLeft, oldTop, oldRight, oldBottom ->
    if (left != oldLeft || top != oldTop || right != oldRight || bottom != oldBottom) {
        val hint = Rect(left, top, right, bottom)
        setPictureInPictureParams(
            PictureInPictureParams.Builder()
                .setSourceRectHint(hint)
                .build()
        )
    }
}
```

### 4-4. 기기 지원 여부 런타임 체크

에뮬레이터나 일부 저가형 기기는 PiP를 지원하지 않습니다. 항상 런타임에 확인하세요.

```kotlin
val supportsPip = packageManager.hasSystemFeature(PackageManager.FEATURE_PICTURE_IN_PICTURE)
if (!supportsPip) {
    // PiP 진입 버튼 숨김 또는 대체 UX 제공
}
```

### 4-5. 스택 정리: `noHistory` 와 `clearTaskOnLaunch`

PiP 창에서 알림이나 딥링크를 클릭했을 때 의도치 않게 백 스택이 쌓이는 문제가 생길 수 있습니다. `FLAG_ACTIVITY_CLEAR_TOP` 또는 `FLAG_ACTIVITY_SINGLE_TOP` 플래그를 PendingIntent에 추가해 중복 Activity 생성을 방지하세요.

---

## 5. 정리

Android PiP는 단순히 창을 줄이는 것이 아니라, Activity 생명주기와 SurfaceControl 레이어를 정밀하게 제어하는 기능입니다. 핵심을 요약하면:

- **Manifest**: `supportsPictureInPicture=true` + `configChanges` 필수
- **자동 진입(API 31+)**: `setAutoEnterEnabled(true)` 로 `onUserLeaveHint()` 대체
- **재생 제어**: `onPause()` 에 `isInPictureInPictureMode` 가드 필수
- **동적 액션 업데이트**: 재생 상태 변화 시 `setPictureInPictureParams()` 재호출
- **애니메이션 품질**: `setSourceRectHint()` 로 전환 좌표 제공

올바르게 구현된 PiP는 사용자가 앱을 떠나지 않고도 핵심 콘텐츠와 연결된 상태를 유지하게 해 주며, 이는 체류 시간과 만족도를 크게 높이는 UX 전략입니다.

---

## 참고 자료
- [Android 공식 문서: Picture-in-Picture 사용하기](https://developer.android.com/develop/ui/views/picture-in-picture)
- [PictureInPictureParams.Builder API Reference](https://developer.android.com/reference/android/app/PictureInPictureParams.Builder)
- [RemoteAction API Reference](https://developer.android.com/reference/android/app/RemoteAction)
