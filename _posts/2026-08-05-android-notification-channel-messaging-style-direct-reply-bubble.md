---
layout: post
title: "Android 알림(Notification) 심화: NotificationChannel·MessagingStyle·Direct Reply·버블 알림 완전 정복"
date: 2026-08-05
categories: [android]
tags: [android, notification, notificationchannel, messagingstyle, directreply, bubble, kotlin]
---

Android 알림 시스템은 앱이 포어그라운드에 없을 때도 사용자와 소통할 수 있는 핵심 채널이다. Android 8.0(API 26)의 NotificationChannel 도입, Android 7.0(API 24)의 Direct Reply, Android 11(API 30)의 버블(Bubble) 알림까지 — 버전마다 중요한 변화가 쌓였지만, 이 변화들을 체계적으로 정리한 자료는 드물다. 이 글에서는 채널 설계부터 MessagingStyle, Direct Reply, 버블 알림까지 실제로 동작하는 코드와 함께 단계적으로 정복한다.

---

## 1. 개념 설명: Android 알림 아키텍처

Android의 알림 스택은 크게 세 층으로 이루어진다.

```
앱 코드
  └─ NotificationCompat.Builder (알림 콘텐츠 구성)
       └─ NotificationChannel (중요도·소리·진동 정책 보유)
            └─ NotificationManager (시스템에 게시)
```

**NotificationChannel**은 Android 8.0부터 필수다. 채널 없이 알림을 게시하면 API 26 이상 기기에서 알림이 표시되지 않는다. 채널은 한 번 생성한 뒤에는 중요도 등 핵심 속성을 앱에서 바꿀 수 없으며, 사용자만이 시스템 설정에서 변경할 수 있다.

**중요도(Importance) 레벨**은 다음 5단계다.

| 상수 | 값 | 동작 |
|---|---|---|
| IMPORTANCE_HIGH | 4 | 헤즈업 알림 + 소리 + 진동 |
| IMPORTANCE_DEFAULT | 3 | 소리 + 진동 |
| IMPORTANCE_LOW | 2 | 소리·진동 없음 |
| IMPORTANCE_MIN | 1 | 상태 바에만 표시 |
| IMPORTANCE_NONE | 0 | 완전 비활성 |

중요도가 높을수록 사용자의 집중을 방해하므로, 앱의 성격에 맞는 레벨을 신중하게 선택해야 한다.

---

## 2. 왜 제대로 알아야 하는가

### 2-1. POST_NOTIFICATIONS 런타임 퍼미션 (API 33+)

Android 13(API 33)부터 `POST_NOTIFICATIONS` 퍼미션이 런타임 퍼미션으로 격상됐다. 이전 버전에서는 앱 설치 시 자동으로 알림 권한이 부여됐지만, API 33 이상에서는 반드시 사용자 동의를 받아야 한다.

```kotlin
// AndroidManifest.xml에 선언
// <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />

// Activity에서 런타임 요청
private val requestPermissionLauncher = registerForActivityResult(
    ActivityResultContracts.RequestPermission()
) { isGranted ->
    if (isGranted) {
        // 알림 발송 가능
    } else {
        // 사용자에게 알림의 필요성 설명 후 재요청
    }
}

fun requestNotificationPermission() {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
        when {
            ContextCompat.checkSelfPermission(
                this, Manifest.permission.POST_NOTIFICATIONS
            ) == PackageManager.PERMISSION_GRANTED -> {
                // 이미 허용됨
            }
            shouldShowRequestPermissionRationale(Manifest.permission.POST_NOTIFICATIONS) -> {
                // 사용자에게 왜 필요한지 설명하는 UI 표시
                showNotificationRationaleDialog()
            }
            else -> {
                requestPermissionLauncher.launch(Manifest.permission.POST_NOTIFICATIONS)
            }
        }
    }
}
```

### 2-2. 채널 설계의 중요성

채널은 **사용자가 앱 알림을 세밀하게 제어하는 단위**다. 메시지 앱이라면 "새 메시지", "그룹 메시지", "시스템 공지" 등을 별도 채널로 분리해야 한다. 모든 알림을 단일 채널에 몰아넣으면 사용자가 특정 종류만 끄고 싶어도 전체를 꺼야 한다.

---

## 3. 실제 구현 예제

### 예제 1: NotificationChannel 생성 및 기본 알림 발송

```kotlin
import android.app.NotificationChannel
import android.app.NotificationManager
import android.app.PendingIntent
import android.content.Context
import android.content.Intent
import android.os.Build
import androidx.core.app.NotificationCompat
import androidx.core.app.NotificationManagerCompat

object NotificationHelper {

    private const val CHANNEL_ID_MESSAGE = "channel_message"
    private const val CHANNEL_ID_PROMO = "channel_promo"

    fun createChannels(context: Context) {
        if (Build.VERSION.SDK_INT < Build.VERSION_CODES.O) return

        val notificationManager =
            context.getSystemService(Context.NOTIFICATION_SERVICE) as NotificationManager

        // 메시지 채널: 높은 중요도, 헤즈업 알림
        val messageChannel = NotificationChannel(
            CHANNEL_ID_MESSAGE,
            "새 메시지",
            NotificationManager.IMPORTANCE_HIGH
        ).apply {
            description = "새 메시지가 도착하면 알려드립니다."
            enableVibration(true)
            vibrationPattern = longArrayOf(0, 250, 250, 250)
            enableLights(true)
            lightColor = android.graphics.Color.BLUE
        }

        // 프로모션 채널: 낮은 중요도, 조용한 알림
        val promoChannel = NotificationChannel(
            CHANNEL_ID_PROMO,
            "이벤트·프로모션",
            NotificationManager.IMPORTANCE_LOW
        ).apply {
            description = "특가 행사 및 이벤트 안내입니다."
        }

        notificationManager.createNotificationChannels(
            listOf(messageChannel, promoChannel)
        )
    }

    fun sendBasicNotification(
        context: Context,
        notificationId: Int,
        title: String,
        body: String
    ) {
        val tapIntent = Intent(context, MainActivity::class.java).apply {
            flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TASK
        }
        val pendingIntent = PendingIntent.getActivity(
            context, 0, tapIntent,
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
        )

        val notification = NotificationCompat.Builder(context, CHANNEL_ID_MESSAGE)
            .setSmallIcon(R.drawable.ic_notification)
            .setContentTitle(title)
            .setContentText(body)
            .setPriority(NotificationCompat.PRIORITY_HIGH)
            .setContentIntent(pendingIntent)
            .setAutoCancel(true)   // 탭하면 자동으로 알림 제거
            .build()

        with(NotificationManagerCompat.from(context)) {
            notify(notificationId, notification)
        }
    }
}
```

**핵심 포인트:**
- `createNotificationChannels()`로 여러 채널을 한 번에 등록하면 이미 존재하는 채널은 무시되므로 앱 시작 시 매번 호출해도 안전하다.
- `PendingIntent.FLAG_IMMUTABLE`은 API 31 이상에서 필수다.

---

### 예제 2: MessagingStyle + Direct Reply + 그룹 알림

메시지 앱의 핵심인 `MessagingStyle`과 알림 내에서 직접 답장할 수 있는 `Direct Reply`를 결합한 예제다.

```kotlin
import android.app.PendingIntent
import android.content.BroadcastReceiver
import android.content.Context
import android.content.Intent
import androidx.core.app.NotificationCompat
import androidx.core.app.NotificationManagerCompat
import androidx.core.app.Person
import androidx.core.app.RemoteInput
import androidx.core.graphics.drawable.IconCompat

const val KEY_TEXT_REPLY = "key_text_reply"
const val ACTION_REPLY = "com.example.ACTION_REPLY"
const val EXTRA_NOTIFICATION_ID = "extra_notification_id"
const val EXTRA_CONVERSATION_ID = "extra_conversation_id"

fun buildMessagingNotification(
    context: Context,
    notificationId: Int,
    conversationId: String,
    senderName: String,
    messages: List<Pair<String, Long>> // (text, timestamp)
) {
    // 보낸 사람 Person 객체
    val sender = Person.Builder()
        .setName(senderName)
        .setIcon(IconCompat.createWithResource(context, R.drawable.ic_avatar_default))
        .build()

    // MessagingStyle 구성
    val messagingStyle = NotificationCompat.MessagingStyle("나")
        .setConversationTitle(senderName)

    messages.forEach { (text, timestamp) ->
        messagingStyle.addMessage(
            NotificationCompat.MessagingStyle.Message(
                text, timestamp, sender
            )
        )
    }

    // Direct Reply 액션 구성
    val remoteInput = RemoteInput.Builder(KEY_TEXT_REPLY)
        .setLabel("답장 입력...")
        .build()

    val replyIntent = Intent(context, ReplyReceiver::class.java).apply {
        action = ACTION_REPLY
        putExtra(EXTRA_NOTIFICATION_ID, notificationId)
        putExtra(EXTRA_CONVERSATION_ID, conversationId)
    }
    val replyPendingIntent = PendingIntent.getBroadcast(
        context,
        notificationId,
        replyIntent,
        PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_MUTABLE // RemoteInput은 MUTABLE 필수
    )

    val replyAction = NotificationCompat.Action.Builder(
        R.drawable.ic_reply, "답장", replyPendingIntent
    )
        .addRemoteInput(remoteInput)
        .setAllowGeneratedReplies(true) // 스마트 답장 활성화
        .build()

    val notification = NotificationCompat.Builder(context, CHANNEL_ID_MESSAGE)
        .setSmallIcon(R.drawable.ic_notification)
        .setStyle(messagingStyle)
        .addAction(replyAction)
        .setPriority(NotificationCompat.PRIORITY_HIGH)
        .setCategory(NotificationCompat.CATEGORY_MESSAGE)
        .setShortcutId(conversationId) // 대화 단축키 연결 (버블 알림에도 필요)
        .setAutoCancel(false) // 답장 후에도 알림 유지
        .build()

    NotificationManagerCompat.from(context).notify(notificationId, notification)
}

// BroadcastReceiver: Direct Reply 처리
class ReplyReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        val bundle = RemoteInput.getResultsFromIntent(intent) ?: return
        val replyText = bundle.getCharSequence(KEY_TEXT_REPLY)?.toString() ?: return
        val notificationId = intent.getIntExtra(EXTRA_NOTIFICATION_ID, -1)
        val conversationId = intent.getStringExtra(EXTRA_CONVERSATION_ID) ?: return

        // 실제 메시지 전송 처리 (Repository 호출 등)
        sendReplyToServer(conversationId, replyText)

        // 답장 후 알림을 "답장 완료" 상태로 업데이트
        val updatedNotification = NotificationCompat.Builder(context, CHANNEL_ID_MESSAGE)
            .setSmallIcon(R.drawable.ic_notification)
            .setContentText("답장을 보냈습니다.")
            .setAutoCancel(true)
            .build()

        NotificationManagerCompat.from(context).notify(notificationId, updatedNotification)
    }

    private fun sendReplyToServer(conversationId: String, text: String) {
        // 실제 구현에서는 WorkManager나 코루틴으로 처리
    }
}
```

**핵심 포인트:**
- `RemoteInput`을 사용하는 `PendingIntent`는 반드시 `FLAG_MUTABLE`이어야 한다. 시스템이 사용자가 입력한 텍스트를 인텐트에 주입해야 하기 때문이다.
- `setCategory(CATEGORY_MESSAGE)`를 설정하면 도어드림(DND) 우선 허용 목록에서 메시지로 인식한다.
- Direct Reply 후 알림을 즉시 업데이트하지 않으면 스피너가 무한 회전하는 것처럼 보이므로, 반드시 업데이트 알림을 다시 게시해야 한다.

---

### 예제 3: 버블(Bubble) 알림 구현

버블 알림은 Android 11(API 30)에서 정식화됐다. 채팅 헤드처럼 화면 위에 떠 있는 UI를 제공한다.

버블 알림의 전제 조건:
1. 채널 중요도가 `IMPORTANCE_HIGH` 이상
2. `NotificationCompat.BubbleMetadata` 설정
3. 버블로 열릴 `Activity`에 `android:allowEmbedded="true"`, `android:resizeable="true"` 설정
4. `ShortcutInfo`와 `Person` 연동

```kotlin
import androidx.core.app.NotificationCompat
import androidx.core.content.pm.ShortcutInfoCompat
import androidx.core.content.pm.ShortcutManagerCompat
import androidx.core.graphics.drawable.IconCompat

fun buildBubbleNotification(
    context: Context,
    notificationId: Int,
    conversationId: String,
    senderName: String,
    latestMessage: String
) {
    val person = Person.Builder()
        .setName(senderName)
        .setIcon(IconCompat.createWithResource(context, R.drawable.ic_avatar_default))
        .setImportant(true)
        .build()

    // 1. ShortcutInfo 등록 (버블에 필수)
    val shortcutIntent = Intent(context, ConversationActivity::class.java).apply {
        action = Intent.ACTION_VIEW
        putExtra(EXTRA_CONVERSATION_ID, conversationId)
    }
    val shortcut = ShortcutInfoCompat.Builder(context, conversationId)
        .setLongLived(true)
        .setIntent(shortcutIntent)
        .setShortLabel(senderName)
        .setIcon(IconCompat.createWithResource(context, R.drawable.ic_avatar_default))
        .setPerson(person)
        .build()
    ShortcutManagerCompat.pushDynamicShortcut(context, shortcut)

    // 2. BubbleMetadata 구성
    val bubbleIntent = PendingIntent.getActivity(
        context,
        notificationId,
        Intent(context, BubbleActivity::class.java).apply {
            putExtra(EXTRA_CONVERSATION_ID, conversationId)
        },
        PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
    )
    val bubbleMetadata = NotificationCompat.BubbleMetadata.Builder(
        bubbleIntent,
        IconCompat.createWithResource(context, R.drawable.ic_avatar_default)
    )
        .setDesiredHeight(600)
        .setAutoExpandBubble(false)      // 처음에 자동 펼치지 않음
        .setSuppressNotification(false)  // 버블이 펼쳐졌을 때도 알림 표시
        .build()

    // 3. MessagingStyle + BubbleMetadata 결합
    val messagingStyle = NotificationCompat.MessagingStyle("나")
        .addMessage(latestMessage, System.currentTimeMillis(), person)

    val notification = NotificationCompat.Builder(context, CHANNEL_ID_MESSAGE)
        .setSmallIcon(R.drawable.ic_notification)
        .setStyle(messagingStyle)
        .setPriority(NotificationCompat.PRIORITY_HIGH)
        .setCategory(NotificationCompat.CATEGORY_MESSAGE)
        .setShortcutId(conversationId)
        .setBubbleMetadata(bubbleMetadata)
        .addPerson(person)
        .build()

    NotificationManagerCompat.from(context).notify(notificationId, notification)
}
```

`AndroidManifest.xml`에서 `BubbleActivity` 설정:

```xml
<activity
    android:name=".BubbleActivity"
    android:allowEmbedded="true"
    android:documentLaunchMode="always"
    android:resizeable="true"
    android:exported="false" />
```

---

## 4. 알림 그룹화 (GroupSummary)

동일한 앱에서 여러 알림을 발송할 때 그룹으로 묶으면 사용자 경험이 향상된다.

```kotlin
const val GROUP_KEY_MESSAGES = "group_messages"
const val SUMMARY_ID = 0

fun notifyGroupSummary(context: Context, count: Int) {
    val summaryNotification = NotificationCompat.Builder(context, CHANNEL_ID_MESSAGE)
        .setSmallIcon(R.drawable.ic_notification)
        .setContentTitle("새 메시지 ${count}개")
        .setStyle(
            NotificationCompat.InboxStyle()
                .setSummaryText("${count}개의 새 메시지")
        )
        .setGroup(GROUP_KEY_MESSAGES)
        .setGroupSummary(true)   // 이 알림이 그룹 요약임을 표시
        .setAutoCancel(true)
        .build()

    NotificationManagerCompat.from(context).notify(SUMMARY_ID, summaryNotification)
}

fun notifyGroupItem(context: Context, id: Int, sender: String, message: String) {
    val notification = NotificationCompat.Builder(context, CHANNEL_ID_MESSAGE)
        .setSmallIcon(R.drawable.ic_notification)
        .setContentTitle(sender)
        .setContentText(message)
        .setGroup(GROUP_KEY_MESSAGES) // 동일한 그룹 키
        .setAutoCancel(true)
        .build()

    NotificationManagerCompat.from(context).notify(id, notification)
}
```

**주의:** 그룹 요약 알림은 실제 알림보다 먼저 게시되면 안 된다. 개별 알림들을 먼저 게시하고 마지막에 요약을 게시해야 올바르게 그룹화된다.

---

## 5. 주의사항 및 실전 팁

### 5-1. 채널 ID는 버전 관리 전략을 세울 것

한 번 생성한 채널의 중요도·소리 설정은 앱에서 변경할 수 없다. 중요도를 바꿔야 한다면 새 채널 ID(`channel_message_v2` 등)로 새 채널을 생성하고, 구 채널은 `deleteNotificationChannel()`로 정리해야 한다. 그러나 채널 삭제 후 동일 ID로 재생성하면 사용자가 설정한 값이 복원되므로, ID 자체를 바꾸는 것이 유일한 방법이다.

### 5-2. setOnlyAlertOnce()로 업데이트 알림 소음 제거

진행 상태(다운로드, 업로드 등)를 표시하는 알림은 자주 업데이트된다. `setOnlyAlertOnce(true)`를 설정하면 알림이 처음 게시될 때만 소리·진동이 발생하고 업데이트 시에는 조용히 갱신된다.

```kotlin
NotificationCompat.Builder(context, CHANNEL_ID_MESSAGE)
    .setOnlyAlertOnce(true)
    // ...
```

### 5-3. Foreground Service 알림은 별도 채널로

`startForeground()`에 사용하는 알림은 사용자가 완전히 끌 수 없다. 이 알림을 메시지 채널 등 사용자가 자주 보는 채널에 배치하면 혼란을 준다. 반드시 `IMPORTANCE_LOW` 혹은 `IMPORTANCE_MIN`의 전용 채널(`channel_foreground_service`)에 게시하라.

### 5-4. 알림 탭 시 특정 화면으로 이동 (Back Stack 설정)

알림을 탭해서 앱을 열었을 때 뒤로 가기를 누르면 앱 홈 화면으로 가야 한다. `TaskStackBuilder`로 인위적인 백 스택을 만들면 된다.

```kotlin
val resultIntent = Intent(context, ConversationActivity::class.java).apply {
    putExtra(EXTRA_CONVERSATION_ID, conversationId)
}
val stackBuilder = TaskStackBuilder.create(context).apply {
    addNextIntentWithParentStack(resultIntent)
}
val pendingIntent = stackBuilder.getPendingIntent(
    0,
    PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
)
```

### 5-5. 테스트 시 알림 채널 초기화

기기에서 테스트할 때 채널 설정을 리셋하려면 앱 데이터를 초기화하거나 아래 ADB 명령을 사용한다.

```bash
adb shell pm clear <package-name>
```

채널은 앱 데이터에 저장되므로 앱 삭제 후 재설치해도 초기화된다.

---

## 6. 정리

| 기능 | 필요 API | 핵심 클래스/메서드 |
|---|---|---|
| 알림 채널 | API 26+ | `NotificationChannel`, `createNotificationChannel()` |
| 런타임 권한 | API 33+ | `POST_NOTIFICATIONS`, `requestPermissionLauncher` |
| MessagingStyle | API 16+ | `NotificationCompat.MessagingStyle`, `Person` |
| Direct Reply | API 24+ | `RemoteInput`, `NotificationCompat.Action` |
| 버블 알림 | API 30+ | `BubbleMetadata`, `ShortcutInfoCompat` |
| 그룹화 | API 20+ | `setGroup()`, `setGroupSummary(true)` |

Android 알림 시스템의 각 기능은 독립적으로 사용할 수도 있지만, 메시지 앱이라면 `MessagingStyle + Direct Reply + 버블`을 모두 결합할 때 최고의 사용자 경험을 제공한다. 채널 설계를 앱 초기에 충분히 고민하고, API 33 권한 요청 타이밍을 신중하게 정하는 것이 프로덕션 품질 알림 시스템의 출발점이다.

---

## 참고 자료
- [Create a notification (Android Developers)](https://developer.android.com/develop/ui/views/notifications/build-notification)
- [Create and manage notification channels (Android Developers)](https://developer.android.com/develop/ui/compose/notifications/channels)
- [Use notification bubbles for conversations (Android Developers)](https://developer.android.com/guide/topics/ui/bubbles)
- [About notifications and conversations (Android Developers)](https://developer.android.com/social-and-messaging/guides/communication/notifications-conversations)
