---
layout: post
title: "Android 플로팅 UI 완전 정복: WindowManager TYPE_APPLICATION_OVERLAY와 Bubble API 심화"
date: 2026-11-16
categories: [android]
tags: [android, windowmanager, overlay, bubble, floating-ui, system-alert-window, kotlin]
---

채팅 헤드, 플로팅 버튼, 화면 위를 떠다니는 번역 위젯 — 이런 플로팅 UI는 어떻게 만들어질까요? Android에서 다른 앱 위에 뷰를 그리는 방법은 크게 두 가지입니다. 하나는 `WindowManager`에 직접 뷰를 추가하는 클래식 오버레이 방식이고, 다른 하나는 Android 10부터 도입된 **Bubble API**입니다. 이 글에서는 두 방식의 내부 동작 원리부터 프로덕션 수준의 구현까지 완전히 정복합니다.

## 1. 개념 설명: 플로팅 UI의 두 가지 축

### WindowManager 오버레이

Android는 모든 UI를 **Window** 단위로 관리합니다. Activity가 가진 Window, Dialog의 Window, 상태 바의 Window 등이 Z-축으로 쌓여 있습니다. `WindowManager` 서비스를 통해 임의의 View를 이 Z-스택에 삽입할 수 있는데, 이것이 오버레이의 핵심입니다.

API 26(Android 8.0) 이전에는 `TYPE_PHONE`, `TYPE_SYSTEM_ALERT` 등 여러 타입이 있었지만, Android 8.0부터 비시스템 앱이 사용할 수 있는 오버레이 타입은 **`TYPE_APPLICATION_OVERLAY`** 하나로 통합되었습니다. 이 타입의 창은:

- 모든 앱 Activity Window 위에 그려집니다.
- 상태 바, IME, 시스템 알림 등 **중요 시스템 창 아래에** 위치합니다.
- `SYSTEM_ALERT_WINDOW` 권한이 필요하며, Android 6.0부터 런타임에 사용자에게 명시적으로 허용받아야 합니다.

```
z-order (높을수록 위)
┌─────────────────────────┐
│   상태 바 / IME (시스템)  │  ← 항상 최상위
├─────────────────────────┤
│  TYPE_APPLICATION_OVERLAY│  ← 우리가 그리는 플로팅 창
├─────────────────────────┤
│  일반 앱 Activity Window  │  ← 일반 앱 UI
└─────────────────────────┘
```

### Bubble API

**Bubble**은 Android 10에서 개발자 프리뷰로 등장해 Android 11에서 정식 출시된 고수준 API입니다. 알림 시스템을 기반으로 동작하며, 특별한 권한 없이도 앱 콘텐츠를 플로팅 원형 아이콘으로 표시할 수 있습니다. Android 17에서는 아예 분리된 창 모드(Windowing Mode)로 격상되어, 사용자가 어떤 앱이든 버블 UI로 고정할 수 있게 되었습니다.

버블은 두 가지 상태를 가집니다:
- **Collapsed**: 화면 가장자리에 작은 원형 아이콘으로 표시
- **Expanded**: 아이콘을 탭하면 지정한 Activity가 플로팅 창으로 펼쳐짐

---

## 2. 왜 필요한가?

### WindowManager 오버레이의 사용 사례

| 사례 | 설명 |
|------|------|
| 보조 도구 | 화면 위에 항상 떠 있는 번역기, 색상 피커, 자 |
| 접근성 서비스 | 시각 장애인을 위한 오버레이 버튼 |
| 개발자 도구 | 디버그 정보를 실시간으로 오버레이 표시 |
| 미디어 컨트롤러 | 다른 앱 사용 중에도 보이는 플로팅 플레이어 |
| 게임 내 이식 UI | 다른 앱 위에 렌더링되는 보조 HUD |

### Bubble API의 사용 사례

Bubble은 주로 **대화(Conversation)** 맥락에서 쓰입니다. 카카오톡 채팅 헤드처럼 메시지 앱에서 특정 대화를 화면에 고정해 두고, 다른 앱을 사용하면서도 빠르게 답장할 수 있는 UX를 제공합니다. Android 11부터는 버블이 `ShortcutInfo`와 연동되어야 하므로 대화형 앱에 특히 적합합니다.

---

## 3. 실제 구현 예제

### 예제 1: WindowManager 플로팅 위젯

드래그 가능한 플로팅 버튼을 Service 안에서 구현하는 예시입니다.

```xml
<!-- AndroidManifest.xml -->
<uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW" />

<service android:name=".FloatingWidgetService" />
```

```kotlin
// FloatingWidgetService.kt
class FloatingWidgetService : Service() {

    private lateinit var windowManager: WindowManager
    private lateinit var floatingView: View
    private lateinit var params: WindowManager.LayoutParams

    override fun onBind(intent: Intent?): IBinder? = null

    override fun onCreate() {
        super.onCreate()
        windowManager = getSystemService(WINDOW_SERVICE) as WindowManager

        floatingView = LayoutInflater.from(this)
            .inflate(R.layout.layout_floating_widget, null)

        params = WindowManager.LayoutParams(
            WindowManager.LayoutParams.WRAP_CONTENT,
            WindowManager.LayoutParams.WRAP_CONTENT,
            // API 26+ 에서는 반드시 TYPE_APPLICATION_OVERLAY 사용
            WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY,
            WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE,
            PixelFormat.TRANSLUCENT
        ).apply {
            gravity = Gravity.TOP or Gravity.START
            x = 100
            y = 300
        }

        windowManager.addView(floatingView, params)
        setupDragListener()
    }

    private fun setupDragListener() {
        var initialX = 0
        var initialY = 0
        var initialTouchX = 0f
        var initialTouchY = 0f

        floatingView.setOnTouchListener { _, event ->
            when (event.action) {
                MotionEvent.ACTION_DOWN -> {
                    initialX = params.x
                    initialY = params.y
                    initialTouchX = event.rawX
                    initialTouchY = event.rawY
                    true
                }
                MotionEvent.ACTION_MOVE -> {
                    params.x = initialX + (event.rawX - initialTouchX).toInt()
                    params.y = initialY + (event.rawY - initialTouchY).toInt()
                    windowManager.updateViewLayout(floatingView, params)
                    true
                }
                else -> false
            }
        }

        // 닫기 버튼
        floatingView.findViewById<ImageView>(R.id.iv_close).setOnClickListener {
            stopSelf()
        }
    }

    override fun onDestroy() {
        super.onDestroy()
        if (::floatingView.isInitialized) {
            windowManager.removeView(floatingView)
        }
    }
}
```

서비스를 시작하기 전에 반드시 권한을 확인해야 합니다:

```kotlin
// Activity에서 서비스 시작 전 권한 체크
fun startFloatingWidget() {
    if (!Settings.canDrawOverlays(this)) {
        val intent = Intent(
            Settings.ACTION_MANAGE_OVERLAY_PERMISSION,
            Uri.parse("package:$packageName")
        )
        // API 30+ 에서는 결과를 받는 방식 권장
        overlayPermissionLauncher.launch(intent)
    } else {
        startService(Intent(this, FloatingWidgetService::class.java))
    }
}

private val overlayPermissionLauncher =
    registerForActivityResult(ActivityResultContracts.StartActivityForResult()) {
        if (Settings.canDrawOverlays(this)) {
            startService(Intent(this, FloatingWidgetService::class.java))
        }
    }
```

---

### 예제 2: Bubble API로 채팅 버블 구현

Android 11+ 대상의 채팅 앱에서 대화를 버블로 표시하는 완전한 예시입니다.

```xml
<!-- AndroidManifest.xml -->
<activity
    android:name=".bubble.BubbleChatActivity"
    android:label="채팅"
    android:allowEmbedded="true"
    android:resizeableActivity="true"
    android:documentLaunchMode="always" />
```

```kotlin
// BubbleNotificationHelper.kt
object BubbleNotificationHelper {

    private const val CHANNEL_ID = "chat_bubble_channel"
    private const val NOTIFICATION_ID = 1001
    private const val SHORTCUT_CATEGORY = "com.example.category.CHAT_SHARE_TARGET"

    fun showChatBubble(
        context: Context,
        contactName: String,
        contactId: String,
        message: String,
        icon: IconCompat
    ) {
        createNotificationChannel(context)
        publishShortcut(context, contactId, contactName, icon)

        // 버블 확장 시 보여줄 Activity의 PendingIntent
        val bubbleIntent = PendingIntent.getActivity(
            context,
            contactId.hashCode(),
            Intent(context, BubbleChatActivity::class.java).apply {
                putExtra("contact_id", contactId)
            },
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
        )

        // 버블 메타데이터 구성
        val bubbleMetadata = NotificationCompat.BubbleMetadata.Builder(bubbleIntent, icon)
            .setDesiredHeight(600)
            .setAutoExpandBubble(false)       // 사용자 액션에 의해서만 확장
            .setSuppressNotification(false)   // 알림도 함께 표시
            .build()

        // Person 객체: Android 11+ 버블에 필수
        val sender = Person.Builder()
            .setName(contactName)
            .setIcon(icon)
            .setImportant(true)
            .build()

        // MessagingStyle 알림 구성
        val messagingStyle = NotificationCompat.MessagingStyle(sender)
            .addMessage(message, System.currentTimeMillis(), sender)

        val notification = NotificationCompat.Builder(context, CHANNEL_ID)
            .setSmallIcon(R.drawable.ic_chat)
            .setStyle(messagingStyle)
            .setBubbleMetadata(bubbleMetadata)
            .setShortcutId(contactId)           // shortcut과 연동 (API 30+ 필수)
            .addPerson(sender)
            .setCategory(NotificationCompat.CATEGORY_MESSAGE)
            .build()

        NotificationManagerCompat.from(context)
            .notify(NOTIFICATION_ID + contactId.hashCode(), notification)
    }

    private fun createNotificationChannel(context: Context) {
        val channel = NotificationChannel(
            CHANNEL_ID,
            "채팅 버블",
            NotificationManager.IMPORTANCE_HIGH
        ).apply {
            description = "대화 버블 알림"
        }
        context.getSystemService(NotificationManager::class.java)
            .createNotificationChannel(channel)
    }

    private fun publishShortcut(
        context: Context,
        id: String,
        name: String,
        icon: IconCompat
    ) {
        val shortcut = ShortcutInfoCompat.Builder(context, id)
            .setCategories(setOf(SHORTCUT_CATEGORY))
            .setIntent(Intent(Intent.ACTION_DEFAULT))
            .setLongLived(true)
            .setShortLabel(name)
            .setIcon(icon)
            .build()
        // pushDynamicShortcut: 이미 존재하면 갱신, 없으면 추가
        ShortcutManagerCompat.pushDynamicShortcut(context, shortcut)
    }
}
```

버블로 열리는 Activity에서는 현재 버블 상태를 감지할 수 있습니다:

```kotlin
// BubbleChatActivity.kt
class BubbleChatActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_bubble_chat)

        // 현재 버블 모드인지 확인
        if (isInBubbleMode()) {
            // 버블 창 크기에 맞는 간소화 UI 표시
            showCompactUI()
        } else {
            showFullUI()
        }
    }

    private fun isInBubbleMode(): Boolean {
        // API 31+
        return if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
            isLaunchedFromBubble
        } else {
            // API 29~30: taskId와 함께 버블 판별
            intent.flags and Intent.FLAG_ACTIVITY_MULTIPLE_TASK != 0
        }
    }

    // 버블이 collapse될 때 뒤로 가기를 처리
    // BackHandler composable 대신 Activity에서 처리하는 방법
    override fun onBackPressed() {
        // 버블을 닫지 않고 collapse 상태로 전환
        moveTaskToBack(false)
    }
}
```

---

## 4. 주의사항 및 팁

### 오버레이 권한 처리

`SYSTEM_ALERT_WINDOW`는 일반적인 dangerous permission과 달리 **특수 앱 권한(Special App Access)**으로 분류됩니다. `ActivityResultContracts.RequestPermission()`이 아닌 `Settings.ACTION_MANAGE_OVERLAY_PERMISSION`으로 설정 화면을 열어야 하고, 결과는 `onActivityResult` 또는 `ActivityResultLauncher`로 받습니다. Play Store는 이 권한을 선언한 앱에 대해 추가 심사를 적용하므로, 반드시 필요한 경우에만 사용하세요.

```kotlin
// 권한 상태를 Flow로 관찰하는 패턴
fun observeOverlayPermission(context: Context): Flow<Boolean> = flow {
    while (true) {
        emit(Settings.canDrawOverlays(context))
        delay(1000)
    }
}.distinctUntilChanged()
```

### Android 12+의 터치 차단 규칙

Android 12부터 `FLAG_NOT_TOUCHABLE`로 설정된 오버레이가 아래 창으로 터치를 전달하려면, **창의 `alpha`가 0.8 이하**여야 합니다. `alpha > 0.8`인 오버레이가 터치를 가리면 시스템이 차단합니다. 여러 오버레이가 쌓인 경우 `1 - ∏(1 - alphaᵢ) ≤ 0.8` 조건을 만족해야 합니다.

```kotlin
params.alpha = 0.8f  // 0.8 이하여야 하위 창이 터치 이벤트를 받음
params.flags = params.flags or WindowManager.LayoutParams.FLAG_NOT_TOUCHABLE
```

### 메모리 누수 방지

`WindowManager.addView()`로 추가된 뷰는 **Service나 Activity가 파괴되어도 자동으로 제거되지 않습니다**. 반드시 `onDestroy()`에서 `removeView()`를 호출해야 합니다. `isAttachedToWindow` 체크로 중복 제거를 방지하세요.

```kotlin
override fun onDestroy() {
    super.onDestroy()
    if (::floatingView.isInitialized && floatingView.isAttachedToWindow) {
        windowManager.removeView(floatingView)
    }
}
```

### Bubble 버스 사례 — 알림 취소 금지

버블로 표시 중인 알림을 `cancel()`하면 버블도 함께 사라집니다. 사용자가 버블을 열었을 때는 알림만 숨기고 버블은 유지하려면, `setSuppressNotification(true)`로 메타데이터를 갱신하고 알림을 **업데이트**해야 합니다.

```kotlin
// 사용자가 버블 채팅을 읽었을 때 알림 뱃지만 제거
fun markAsRead(context: Context, notificationId: Int, existingBuilder: NotificationCompat.Builder) {
    val updatedBubble = NotificationCompat.BubbleMetadata.Builder()
        .setSuppressNotification(true)   // 알림은 숨기되 버블은 유지
        .build()

    val updatedNotification = existingBuilder
        .setBubbleMetadata(updatedBubble)
        .setOnlyAlertOnce(true)
        .build()

    NotificationManagerCompat.from(context).notify(notificationId, updatedNotification)
}
```

### 버전별 대응 전략 요약

| Android 버전 | 오버레이 타입 | Bubble 사용 가능 | Shortcut 필수 |
|---|---|---|---|
| < 8.0 (API 26) | `TYPE_SYSTEM_ALERT` | ✗ | — |
| 8.0 ~ 9 (API 26~28) | `TYPE_APPLICATION_OVERLAY` | ✗ | — |
| 10 (API 29) | `TYPE_APPLICATION_OVERLAY` | △ (개발자 옵션) | ✗ |
| 11 (API 30) | `TYPE_APPLICATION_OVERLAY` | ✓ | ✓ |
| 12+ (API 31+) | `TYPE_APPLICATION_OVERLAY` | ✓ | ✓ |
| 17+ | `TYPE_APPLICATION_OVERLAY` | ✓ (창 모드 격상) | ✓ |

### 성능 팁

- 오버레이 뷰는 항상 화면에 그려지므로 **GPU 오버드로를 최소화**해야 합니다. 불필요한 배경색은 제거하고 `PixelFormat.TRANSLUCENT`를 활용하세요.
- 오버레이가 사용자 화면 가림(UI Obscuration)을 유발하지 않도록 `alpha`와 크기를 최소화하세요. Google Play 정책은 사용자를 속이기 위한 오버레이를 엄격히 금지합니다.
- 드래그 연산에서 `WindowManager.updateViewLayout()`은 메인 스레드에서 호출해야 합니다. `MotionEvent` 처리 중 비동기 처리를 하면 뷰 위치와 터치 위치 불일치가 발생합니다.

---

## 마치며

`WindowManager TYPE_APPLICATION_OVERLAY`는 강력하지만 그만큼 사용자 UX와 보안에 직접적인 영향을 줍니다. 반드시 필요한 경우에만 쓰고, 권한을 명확히 안내해야 합니다. 반면 Bubble API는 권한 없이도 대화형 콘텐츠를 플로팅으로 제공할 수 있는 현대적인 대안입니다. Android 17 이후로는 버블이 더욱 강화된 창 모드로 발전하고 있으므로, 새로 만드는 채팅/통화 앱이라면 Bubble API를 최우선으로 검토하세요.

## 참고 자료
- [Android Bubbles 공식 가이드](https://developer.android.com/guide/topics/ui/bubbles)
- [WindowManager.LayoutParams 레퍼런스](https://developer.android.com/reference/android/view/WindowManager.LayoutParams)
- [다른 앱 위에 그리기 권한 가이드](https://developer.android.com/reference/android/Manifest.permission#SYSTEM_ALERT_WINDOW)
- [SociaLite 샘플 앱 (Bubble 구현 레퍼런스)](https://github.com/android/socialite)
