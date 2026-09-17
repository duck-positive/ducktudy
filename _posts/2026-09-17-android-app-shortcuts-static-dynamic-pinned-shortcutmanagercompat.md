---
layout: post
title: "Android App Shortcuts 심화: Static·Dynamic·Pinned Shortcuts와 ShortcutManagerCompat 완전 정복"
date: 2026-09-17
categories: [android]
tags: [android, shortcuts, shortcutmanager, launcher, jetpack, kotlin]
---

Android 7.1(API 25)에서 도입된 **App Shortcuts**는 사용자가 앱 아이콘을 길게 누르면 나타나는 빠른 실행 메뉴입니다. Gmail의 "새 메시지 작성", 지도 앱의 "집으로 이동" 같이 핵심 기능으로 즉시 진입할 수 있는 진입점을 제공합니다. 이 글에서는 Static·Dynamic·Pinned Shortcuts의 내부 동작 원리부터 `ShortcutManagerCompat` API 활용, Google Assistant 연동, 그리고 실전에서 반드시 알아야 할 주의사항까지 깊이 있게 살펴봅니다.

## 1. App Shortcuts의 세 가지 유형

### 1-1. Static Shortcuts (정적 단축키)

XML 리소스 파일에 정의되며 APK 또는 App Bundle에 패키징됩니다. 앱이 설치된 순간부터 사용 가능하며, 변경하려면 앱 업데이트가 필요합니다. 앱의 핵심 워크플로우처럼 사용자 컨텍스트와 무관하게 항상 동일한 기능을 제공하는 진입점에 적합합니다.

- **저장 위치**: `res/xml/shortcuts.xml`
- **적합한 용도**: "새 작성", "검색", "홈으로 이동" 같은 앱의 핵심 액션
- **제한**: 최대 개수는 Static + Dynamic 합산으로 `getMaxShortcutCountPerActivity()` 반환값(보통 15)을 초과할 수 없음

### 1-2. Dynamic Shortcuts (동적 단축키)

런타임에 앱 코드에서 생성·수정·삭제합니다. 사용자의 최근 대화 상대, 즐겨찾기 항목 등 사용자 데이터에 기반한 개인화된 단축키를 제공할 수 있습니다.

- **저장 위치**: 앱 프로세스 메모리 → 시스템 ShortcutManager 서비스
- **적합한 용도**: "최근 연락처에게 전화", "마지막으로 편집한 문서 열기" 등 컨텍스트 기반 액션
- **주의**: 앱 데이터를 지우거나 재설치하면 소멸 (백업·복원 시 유지되지 않을 수 있음)

### 1-3. Pinned Shortcuts (고정 단축키)

사용자가 홈 화면에 직접 배치하는 단축키입니다. 시스템이 사용자에게 확인 다이얼로그를 표시하며, 사용자가 승인해야만 생성됩니다. 앱에서 직접 삭제할 수 없고 **비활성화(disable)만 가능**합니다.

- **적합한 용도**: 특정 연락처 채팅, 특정 플레이리스트 실행 등 개별 항목의 바로가기
- **특징**: 앱을 삭제해도 런처 UI에 남아있을 수 있음 (비활성화 상태로 표시)

---

## 2. 왜 App Shortcuts가 중요한가

### 사용자 참여(Engagement) 향상

단축키는 앱 진입 마찰을 줄입니다. 사용자가 매일 반복하는 작업을 2~3단계 탐색 없이 홈 화면에서 바로 시작할 수 있으면 DAU와 리텐션에 긍정적인 영향을 줍니다. Google Play 내부 데이터에 따르면 단축키를 적극 활용하는 앱은 재방문율이 눈에 띄게 높습니다.

### Google Assistant 및 App Prediction 연동

`pushDynamicShortcut()`으로 등록한 단축키는 Android의 **App Prediction 엔진**에 신호를 보냅니다. 시스템은 이 데이터를 학습해 Pixel 런처의 예측 앱 목록, Google Assistant의 앱 액션 추천, Pixel 5+의 스마트 스택 위젯 등에 해당 기능을 자동으로 노출합니다.

### 앱 차별화

경쟁 앱이 단순히 아이콘만 제공할 때, 잘 설계된 단축키는 사용자에게 앱의 핵심 가치를 한눈에 보여주는 "미리보기" 역할을 합니다.

---

## 3. 실제 구현 예제

### 3-1. Static Shortcuts 구현

**Step 1: `res/xml/shortcuts.xml` 생성**

```xml
<?xml version="1.0" encoding="utf-8"?>
<shortcuts xmlns:android="http://schemas.android.com/apk/res/android">

    <!-- 새 메시지 작성 -->
    <shortcut
        android:shortcutId="compose_message"
        android:enabled="true"
        android:icon="@drawable/ic_shortcut_compose"
        android:shortcutShortLabel="@string/shortcut_compose_short"
        android:shortcutLongLabel="@string/shortcut_compose_long">
        <intent
            android:action="android.intent.action.VIEW"
            android:targetPackage="com.example.myapp"
            android:targetClass="com.example.myapp.ui.compose.ComposeActivity" />
        <!-- 백 스택: ComposeActivity → MainActivity 순으로 쌓임 -->
        <intent
            android:action="android.intent.action.MAIN"
            android:targetPackage="com.example.myapp"
            android:targetClass="com.example.myapp.ui.main.MainActivity" />
        <categories android:name="android.shortcut.conversation" />
    </shortcut>

    <!-- 즐겨찾기 목록 -->
    <shortcut
        android:shortcutId="open_favorites"
        android:enabled="true"
        android:icon="@drawable/ic_shortcut_star"
        android:shortcutShortLabel="@string/shortcut_favorites_short"
        android:shortcutLongLabel="@string/shortcut_favorites_long">
        <intent
            android:action="android.intent.action.VIEW"
            android:targetPackage="com.example.myapp"
            android:targetClass="com.example.myapp.ui.favorites.FavoritesActivity" />
    </shortcut>

</shortcuts>
```

**Step 2: `AndroidManifest.xml`의 LAUNCHER Activity에 메타데이터 등록**

```xml
<activity
    android:name=".ui.main.MainActivity"
    android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
    <!-- App Shortcuts 등록 -->
    <meta-data
        android:name="android.app.shortcuts"
        android:resource="@xml/shortcuts" />
</activity>
```

Static Shortcut의 `<intent>` 태그를 여러 개 나열하면 자동으로 백 스택이 구성됩니다. 나중에 나열된 인텐트가 최상위에 올라오므로 순서에 주의하세요.

---

### 3-2. Dynamic Shortcuts 구현 (ShortcutManagerCompat 활용)

의존성 추가:

```kotlin
// build.gradle.kts (app)
implementation("androidx.core:core-ktx:1.13.0")
// Google Assistant 연동 시 추가
implementation("androidx.core:core-google-shortcuts:1.1.0")
```

```kotlin
import android.app.PendingIntent
import android.content.Context
import android.content.Intent
import androidx.core.app.Person
import androidx.core.content.pm.ShortcutInfoCompat
import androidx.core.content.pm.ShortcutManagerCompat
import androidx.core.graphics.drawable.IconCompat

class AppShortcutManager(private val context: Context) {

    /**
     * 최근 대화 상대 기반으로 Dynamic Shortcuts를 갱신합니다.
     * Application.onCreate() 또는 로그인 직후에 호출하세요.
     */
    fun refreshConversationShortcuts(recentContacts: List<Contact>) {
        // 기기에서 허용하는 최대 단축키 수 확인
        val maxCount = ShortcutManagerCompat.getMaxShortcutCountPerActivity(context)

        val shortcuts = recentContacts
            .take(minOf(3, maxCount))   // Static Shortcuts 자리를 남겨두기 위해 3개로 제한
            .mapIndexed { index, contact ->
                ShortcutInfoCompat.Builder(context, "contact_${contact.id}")
                    .setShortLabel(contact.displayName)         // 최대 10자
                    .setLongLabel("${contact.displayName}에게 메시지 보내기")  // 최대 25자
                    .setIcon(
                        IconCompat.createWithResource(context, R.drawable.ic_person)
                    )
                    .setIntent(
                        Intent(context, ChatActivity::class.java).apply {
                            action = Intent.ACTION_VIEW
                            putExtra(ChatActivity.EXTRA_CONTACT_ID, contact.id)
                            // 단축키를 통한 진입임을 식별
                            putExtra(ChatActivity.EXTRA_FROM_SHORTCUT, true)
                        }
                    )
                    // 대화형 단축키임을 명시 → 시스템의 App Prediction에 활용
                    .setCategories(
                        setOf(ShortcutInfoCompat.SHORTCUT_CATEGORY_CONVERSATION)
                    )
                    // Person 정보: Google Assistant가 연락처 정보를 인식하는 데 사용
                    .setPerson(
                        Person.Builder()
                            .setName(contact.displayName)
                            .setKey(contact.id)     // 고유 키 (전화번호, UID 등)
                            .build()
                    )
                    // rank: 낮을수록 상위에 표시 (0이 최고 우선순위)
                    .setRank(index)
                    .setLongLived(true) // 단축키가 숨겨져도 직접 공유 등에서 참조 가능
                    .build()
            }

        // 기존 Dynamic Shortcuts를 교체
        ShortcutManagerCompat.setDynamicShortcuts(context, shortcuts)
    }

    /**
     * 새 대화가 시작될 때 해당 단축키를 추가/갱신하고 사용 기록을 남깁니다.
     * pushDynamicShortcut은 setDynamicShortcuts와 달리 기존 목록을 유지하면서
     * 개별 단축키를 추가하거나 업데이트합니다.
     */
    fun onConversationStarted(contact: Contact) {
        val shortcut = ShortcutInfoCompat.Builder(context, "contact_${contact.id}")
            .setShortLabel(contact.displayName)
            .setLongLabel("${contact.displayName}에게 메시지 보내기")
            .setIcon(IconCompat.createWithResource(context, R.drawable.ic_chat))
            .setIntent(
                Intent(context, ChatActivity::class.java).apply {
                    action = Intent.ACTION_VIEW
                    putExtra(ChatActivity.EXTRA_CONTACT_ID, contact.id)
                    putExtra(ChatActivity.EXTRA_FROM_SHORTCUT, true)
                }
            )
            .setCategories(setOf(ShortcutInfoCompat.SHORTCUT_CATEGORY_CONVERSATION))
            .setPerson(Person.Builder().setName(contact.displayName).setKey(contact.id).build())
            .setLongLived(true)
            .build()

        // 개별 단축키 추가 (최대 수 초과 시 rank 낮은 것부터 자동 제거)
        ShortcutManagerCompat.pushDynamicShortcut(context, shortcut)

        // 시스템에 사용 기록 보고 → App Prediction 정확도 향상
        ShortcutManagerCompat.reportShortcutUsed(context, "contact_${contact.id}")
    }

    /**
     * 연락처 삭제 시 해당 Dynamic Shortcut을 제거합니다.
     */
    fun removeContactShortcut(contactId: String) {
        ShortcutManagerCompat.removeDynamicShortcuts(context, listOf("contact_$contactId"))
    }

    /**
     * 로그아웃 시 모든 Dynamic Shortcuts 제거
     */
    fun clearAllDynamicShortcuts() {
        ShortcutManagerCompat.removeAllDynamicShortcuts(context)
    }

    /**
     * 현재 등록된 Dynamic Shortcuts 목록 조회
     * 앱 실행 시 단축키 복원이 필요한지 확인하는 데 사용
     */
    fun getExistingShortcuts(): List<ShortcutInfoCompat> {
        return ShortcutManagerCompat.getDynamicShortcuts(context)
    }
}
```

---

### 3-3. Pinned Shortcuts 구현

사용자가 특정 항목을 "홈에 추가"하는 기능에 활용합니다.

```kotlin
import android.app.PendingIntent
import android.content.Context
import android.content.Intent
import android.widget.Toast
import androidx.core.content.pm.ShortcutInfoCompat
import androidx.core.content.pm.ShortcutManagerCompat
import androidx.core.graphics.drawable.IconCompat

object PinnedShortcutHelper {

    /**
     * Pinned Shortcut 생성을 시스템에 요청합니다.
     * 런처가 지원하는 경우 사용자에게 확인 다이얼로그가 표시됩니다.
     */
    fun requestPinShortcut(context: Context, item: FavoriteItem) {
        // 현재 런처가 Pinned Shortcuts를 지원하는지 먼저 확인
        if (!ShortcutManagerCompat.isRequestPinShortcutSupported(context)) {
            Toast.makeText(
                context,
                "현재 런처는 홈 화면 바로가기를 지원하지 않습니다",
                Toast.LENGTH_SHORT
            ).show()
            return
        }

        val shortcut = ShortcutInfoCompat.Builder(context, "favorite_${item.id}")
            .setShortLabel(item.name)
            .setLongLabel("${item.name} 열기")
            .setIcon(IconCompat.createWithResource(context, R.drawable.ic_star))
            .setIntent(
                Intent(context, DetailActivity::class.java).apply {
                    action = Intent.ACTION_VIEW
                    putExtra(DetailActivity.EXTRA_ITEM_ID, item.id)
                }
            )
            .build()

        // 사용자가 단축키를 추가했을 때 수신할 콜백 인텐트
        // (선택 사항 — null을 넘겨도 동작하지만 결과를 알 수 없음)
        val resultIntent = ShortcutManagerCompat.createShortcutResultIntent(context, shortcut)
        val successCallback = PendingIntent.getBroadcast(
            context,
            item.id.hashCode(),
            resultIntent,
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
        )

        ShortcutManagerCompat.requestPinShortcut(
            context,
            shortcut,
            successCallback.intentSender
        )
    }

    /**
     * Pinned Shortcut은 삭제 불가 → 비활성화만 가능
     * 연결된 데이터가 삭제된 경우 호출해 단축키를 회색으로 표시합니다.
     */
    fun disablePinnedShortcut(context: Context, itemId: String) {
        ShortcutManagerCompat.disableShortcuts(
            context,
            listOf("favorite_$itemId"),
            "이 항목이 삭제되었습니다"   // 사용자에게 표시될 메시지
        )
    }

    /**
     * 데이터가 복구됐거나 다시 유효해진 경우 단축키를 재활성화합니다.
     */
    fun enablePinnedShortcut(context: Context, itemId: String) {
        ShortcutManagerCompat.enableShortcuts(
            context,
            listOf(
                ShortcutInfoCompat.Builder(context, "favorite_$itemId")
                    .setShortLabel("") // 최소 필드만 채우면 됨
                    .setIntent(Intent(Intent.ACTION_VIEW))
                    .build()
            )
        )
    }
}
```

---

## 4. 심화: 백 스택(Back Stack) 올바르게 구성하기

단축키로 앱에 진입하면 사용자가 백 버튼을 눌렀을 때 자연스럽게 앱의 메인 화면으로 돌아오도록 백 스택을 구성해야 합니다. Dynamic Shortcut에서는 인텐트를 배열로 넘겨 처리합니다.

```kotlin
// 여러 인텐트를 설정해 백 스택 구성
val backStackIntents = arrayOf(
    // 루트: MainActivity
    Intent(context, MainActivity::class.java).apply {
        action = Intent.ACTION_MAIN
    },
    // 중간: InboxActivity
    Intent(context, InboxActivity::class.java).apply {
        action = Intent.ACTION_VIEW
    },
    // 최상위 (단축키 목적지): ThreadActivity
    Intent(context, ThreadActivity::class.java).apply {
        action = Intent.ACTION_VIEW
        putExtra("THREAD_ID", threadId)
    }
)

val shortcut = ShortcutInfoCompat.Builder(context, "thread_$threadId")
    .setShortLabel(subject)
    .setLongLabel("$subject 열기")
    .setIcon(IconCompat.createWithResource(context, R.drawable.ic_mail))
    .setIntents(backStackIntents)   // setIntent() 대신 setIntents() 사용
    .build()
```

또는 `AndroidManifest.xml`에서 `android:parentActivityName`을 설정하는 방식으로도 백 스택을 자동 구성할 수 있습니다.

---

## 5. 단축키 진입 감지 및 reportShortcutUsed

단축키로 진입한 경우를 Activity에서 감지하고 시스템에 보고해야 합니다.

```kotlin
class ChatActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_chat)

        val contactId = intent.getStringExtra(EXTRA_CONTACT_ID) ?: return
        val fromShortcut = intent.getBooleanExtra(EXTRA_FROM_SHORTCUT, false)

        if (fromShortcut) {
            // 단축키로 진입했음을 시스템에 보고
            // → Android App Prediction이 이 단축키의 우선순위를 높임
            ShortcutManagerCompat.reportShortcutUsed(this, "contact_$contactId")
        }

        loadConversation(contactId)
    }

    companion object {
        const val EXTRA_CONTACT_ID = "extra_contact_id"
        const val EXTRA_FROM_SHORTCUT = "extra_from_shortcut"
    }
}
```

---

## 6. 주의사항 및 실전 팁

### ⚠️ 민감한 정보 포함 금지

단축키의 레이블, 아이콘, 인텐트 Extra에 **개인 식별 정보(전화번호, 이메일, 금융 정보 등)를 직접 넣지 마세요**. 단축키 데이터는 기기 백업에 포함될 수 있으며, 다른 기기로 복원 시 노출될 위험이 있습니다. ID값만 Extra로 넘기고, 실제 데이터는 앱 내에서 ID로 조회하는 방식을 권장합니다.

### ⚠️ 단축키 수 관리

Static + Dynamic 합계가 `getMaxShortcutCountPerActivity()` (일반적으로 15)를 초과하면 `pushDynamicShortcut()`이 rank가 낮은 기존 단축키를 자동으로 제거합니다. 홈 화면에서 보이는 단축키는 런처마다 다르지만 보통 **최대 4개**입니다. 실제 사용자가 보는 개수와 시스템이 관리하는 개수를 혼동하지 마세요.

### ⚠️ Dynamic Shortcuts 복원

백업·복원 후 Dynamic Shortcuts는 사라질 수 있습니다. `Application.onCreate()`에서 `getDynamicShortcuts()`로 기존 단축키를 확인하고, 필요하면 재생성하는 로직을 추가하세요.

```kotlin
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        restoreDynamicShortcutsIfNeeded()
    }

    private fun restoreDynamicShortcutsIfNeeded() {
        val existing = ShortcutManagerCompat.getDynamicShortcuts(this)
        if (existing.isEmpty() && userIsLoggedIn()) {
            // 단축키가 사라진 경우 재생성
            AppShortcutManager(this).refreshConversationShortcuts(
                getRecentContacts()
            )
        }
    }
}
```

### 💡 레이블 길이 제한

| 레이블 종류 | 권장 최대 길이 |
|---|---|
| `setShortLabel()` | 10자 |
| `setLongLabel()` | 25자 |

이 제한을 초과해도 런타임 예외는 발생하지 않지만 런처에서 말줄임 처리되거나 잘립니다.

### 💡 Static과 Dynamic의 조합 전략

- **Static**: 앱의 핵심 기능 1~2개 (항상 존재해야 하는 진입점)
- **Dynamic**: 사용자 컨텍스트 기반 2~3개 (최근 이용 항목, 개인화 콘텐츠)
- **Pinned**: 사용자가 직접 요청하는 특정 즐겨찾기

이 조합으로 일관성(Static)과 개인화(Dynamic), 사용자 주도성(Pinned)을 동시에 제공할 수 있습니다.

### 💡 `setLongLived(true)` 설정

Dynamic Shortcut이 한도 초과로 숨겨지더라도 `setLongLived(true)`를 설정하면 직접 공유(Direct Share), Notification 연동, Assistant 추천 등에서 계속 참조될 수 있습니다. Conversation 카테고리 단축키에는 반드시 설정하는 것을 권장합니다.

---

## 참고 자료

- [App shortcuts overview — Android Developers](https://developer.android.com/develop/ui/views/launch/shortcuts)
- [ShortcutManagerCompat API reference — AndroidX](https://developer.android.com/reference/androidx/core/content/pm/ShortcutManagerCompat)
- [Best practices for shortcuts — Android Developers](https://developer.android.com/develop/ui/views/launch/shortcuts/best-practices)
- [Create shortcuts — Android Developers](https://developer.android.com/guide/topics/ui/shortcuts/creating-shortcuts)
