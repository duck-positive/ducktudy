---
layout: post
title: "Android AccessibilityService 심화: 화면 감지·자동화·커스텀 접근성 기능 구현 완전 정복"
date: 2026-11-22
categories: [android]
tags: [android, accessibility, accessibilityservice, a11y, kotlin, ui-automation]
---

Android `AccessibilityService`는 시스템이 발생시키는 접근성 이벤트를 수신하고, 화면 콘텐츠를 탐색하며, 사용자 대신 제스처와 액션을 수행할 수 있는 강력한 백그라운드 서비스 API다. TalkBack, Switch Access, 자동화 테스트 프레임워크 등이 이 API 위에 구축되어 있다. 이 글에서는 `AccessibilityService`의 작동 원리부터 이벤트 처리, 노드 탐색, 제스처 디스패치, 고급 기능까지 실제 코드와 함께 완전히 정복한다.

---

## 1. 왜 AccessibilityService가 필요한가

표준 Android 앱은 자신의 뷰 계층구조만 제어할 수 있다. 하지만 접근성 보조 도구(스크린 리더, 스위치 제어, 음성 제어 앱)나 엔터프라이즈 자동화 솔루션은 **기기 전체의 UI를 관찰하고 조작**해야 한다. `AccessibilityService`가 이를 가능하게 하는 공식 진입점이다.

주요 활용 사례는 다음과 같다.

- **스크린 리더**: 화면의 텍스트·버튼을 TTS로 읽어준다.
- **스위치 액세스**: 단일 버튼 또는 외부 스위치로 UI를 탐색한다.
- **워크플로 자동화**: 반복적인 UI 인터랙션을 프로그래밍적으로 처리한다.
- **접근성 모니터링**: 특정 앱의 UI 변화를 감지해 사용자에게 알린다.

> **주의**: 접근성 서비스는 장애가 있는 사용자를 위한 보조 도구 목적으로만 사용해야 한다. Google Play는 비접근성 목적(광고 클릭 자동화 등)으로의 사용을 금지한다.

---

## 2. 접근성 서비스 생명주기

```
사용자가 설정에서 서비스 활성화
        ↓
  시스템이 서비스에 바인딩
        ↓
  onServiceConnected() 호출
        ↓
  이벤트 발생 → onAccessibilityEvent() 반복 호출
        ↓
  사용자가 비활성화 or disableSelf() 호출
        ↓
  서비스 종료
```

서비스가 시스템과 연결된 후에야 `AccessibilityServiceInfo` 설정이 적용된다. `onServiceConnected()`는 초기화의 올바른 위치다.

---

## 3. 서비스 선언과 설정

### 3.1 Manifest 등록

```xml
<service
    android:name=".MyAccessibilityService"
    android:permission="android.permission.BIND_ACCESSIBILITY_SERVICE"
    android:exported="true"
    android:label="@string/accessibility_service_label">
    <intent-filter>
        <action android:name="android.accessibilityservice.AccessibilityService" />
    </intent-filter>
    <meta-data
        android:name="android.accessibilityservice"
        android:resource="@xml/accessibility_service_config" />
</service>
```

`BIND_ACCESSIBILITY_SERVICE` 권한은 오직 시스템만 서비스에 바인딩할 수 있도록 제한한다. `exported="true"` 없이는 시스템이 바인딩 자체를 거부한다.

### 3.2 XML 설정 파일 (res/xml/accessibility_service_config.xml)

```xml
<accessibility-service
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:description="@string/accessibility_service_description"
    android:accessibilityEventTypes="typeWindowStateChanged|typeViewClicked|typeViewTextChanged"
    android:accessibilityFlags="flagDefault|flagReportViewIds|flagRequestEnhancedWebAccessibility"
    android:accessibilityFeedbackType="feedbackGeneric"
    android:notificationTimeout="100"
    android:canRetrieveWindowContent="true"
    android:canPerformGestures="true"
    android:settingsActivity="com.example.ServiceSettingsActivity" />
```

핵심 속성 설명:

| 속성 | 설명 |
|------|------|
| `accessibilityEventTypes` | 수신할 이벤트 유형. `typeAllMask`는 모든 이벤트 수신 (CPU 부하 주의) |
| `canRetrieveWindowContent` | UI 계층 탐색 허용 여부 |
| `canPerformGestures` | `dispatchGesture()` API 사용 허용 여부 |
| `notificationTimeout` | 동일한 유형의 이벤트 사이 최소 간격 (ms) |

---

## 4. 핵심 구현: 이벤트 처리와 노드 탐색

아래는 화면에서 특정 텍스트가 포함된 버튼을 자동으로 탐지하고 클릭하는 실제 구현 예제다.

```kotlin
class SmartAccessibilityService : AccessibilityService() {

    override fun onServiceConnected() {
        // 런타임에 서비스 설정 동적 변경
        serviceInfo = serviceInfo.apply {
            eventTypes = AccessibilityEvent.TYPE_WINDOW_STATE_CHANGED or
                    AccessibilityEvent.TYPE_VIEW_CLICKED or
                    AccessibilityEvent.TYPE_WINDOW_CONTENT_CHANGED
            feedbackType = AccessibilityServiceInfo.FEEDBACK_GENERIC
            notificationTimeout = 100
            flags = flags or AccessibilityServiceInfo.FLAG_REPORT_VIEW_IDS
        }
    }

    override fun onAccessibilityEvent(event: AccessibilityEvent) {
        when (event.eventType) {
            AccessibilityEvent.TYPE_WINDOW_STATE_CHANGED -> {
                // 새 창이 열릴 때마다 UI 계층 탐색
                val packageName = event.packageName?.toString() ?: return
                if (packageName == "com.target.app") {
                    scanAndInteract()
                }
            }
            AccessibilityEvent.TYPE_VIEW_TEXT_CHANGED -> {
                // 텍스트 변경 감지
                val source = event.source ?: return
                val newText = source.text?.toString()
                handleTextChange(source, newText)
                source.recycle() // 반드시 recycle() 호출
            }
        }
    }

    private fun scanAndInteract() {
        val root = rootInActiveWindow ?: return
        try {
            // 텍스트가 "확인"인 버튼 탐색
            val nodes = root.findAccessibilityNodeInfosByText("확인")
            for (node in nodes) {
                if (node.isClickable && node.isEnabled) {
                    node.performAction(AccessibilityNodeInfo.ACTION_CLICK)
                    break
                }
                node.recycle()
            }
        } finally {
            root.recycle()
        }
    }

    private fun handleTextChange(node: AccessibilityNodeInfo, text: String?) {
        if (text?.length ?: 0 > 100) {
            // 긴 텍스트는 TTS로 요약 안내
            speakText("텍스트 입력 완료: ${text?.take(20)}...")
        }
    }

    private fun speakText(message: String) {
        // TextToSpeech 구현 (생략)
    }

    override fun onInterrupt() {
        // 피드백 중단 처리
    }
}
```

**노드 탐색 패턴**:

```
rootInActiveWindow
    ├── findAccessibilityNodeInfosByText("버튼명")   // 텍스트로 검색
    ├── findAccessibilityNodeInfosByViewId("com.pkg:id/submit")  // ID로 검색
    └── findFocus(FOCUS_INPUT / FOCUS_ACCESSIBILITY)  // 현재 포커스 노드
```

> `AccessibilityNodeInfo` 객체는 사용 후 반드시 `recycle()`을 호출해야 한다. Android API 33 이상에서는 자동 관리되지만, 하위 호환을 위해 명시적 호출을 권장한다.

---

## 5. 제스처 디스패치: 화면을 프로그래밍으로 조작

`dispatchGesture()` API를 통해 탭, 스와이프, 멀티터치를 프로그래밍적으로 수행할 수 있다. `canPerformGestures="true"` 설정이 필수다.

```kotlin
/**
 * 화면의 특정 좌표를 탭하는 제스처
 * @param x 화면 X 좌표 (픽셀)
 * @param y 화면 Y 좌표 (픽셀)
 */
fun performTap(x: Float, y: Float) {
    val tapPath = Path().apply {
        moveTo(x, y)
        // 탭은 동일 지점에서 시작과 끝
        lineTo(x, y)
    }

    val stroke = GestureDescription.StrokeDescription(
        tapPath,
        0L,   // 시작 지연 (ms)
        50L   // 지속 시간 (ms) — 50ms는 일반 탭
    )

    val gesture = GestureDescription.Builder()
        .addStroke(stroke)
        .build()

    dispatchGesture(gesture, object : GestureResultCallback() {
        override fun onCompleted(gestureDescription: GestureDescription?) {
            Log.d("A11y", "탭 제스처 완료: ($x, $y)")
        }
        override fun onCancelled(gestureDescription: GestureDescription?) {
            Log.w("A11y", "탭 제스처 취소됨")
        }
    }, null)
}

/**
 * 위에서 아래로 스와이프하는 제스처 (알림 패널 열기 등)
 */
fun swipeDown(startX: Float, startY: Float, endY: Float) {
    val swipePath = Path().apply {
        moveTo(startX, startY)
        lineTo(startX, endY)
    }

    val stroke = GestureDescription.StrokeDescription(swipePath, 0L, 300L)
    val gesture = GestureDescription.Builder().addStroke(stroke).build()

    dispatchGesture(gesture, null, null)
}

/**
 * 연속 제스처: L자 모양 드래그
 * StrokeDescription.continueStroke()를 활용해 여러 경로를 체이닝
 */
fun performLShapedDrag() {
    val path1 = Path().apply {
        moveTo(200f, 300f)
        lineTo(500f, 300f)  // 오른쪽으로 이동
    }
    val path2 = Path().apply {
        moveTo(500f, 300f)
        lineTo(500f, 600f)  // 아래로 이동
    }

    // willContinue = true: 다음 stroke와 연결됨을 명시
    val stroke1 = GestureDescription.StrokeDescription(path1, 0L, 300L, true)
    val stroke2 = stroke1.continueStroke(path2, 0L, 300L, false)

    val gesture = GestureDescription.Builder()
        .addStroke(stroke1)
        .addStroke(stroke2)
        .build()

    dispatchGesture(gesture, null, null)
}
```

**전역 액션** (`performGlobalAction`)으로 시스템 수준 작업도 수행 가능하다:

```kotlin
// 뒤로 가기
performGlobalAction(GLOBAL_ACTION_BACK)

// 홈 화면
performGlobalAction(GLOBAL_ACTION_HOME)

// 최근 앱
performGlobalAction(GLOBAL_ACTION_RECENTS)

// 알림 창 열기
performGlobalAction(GLOBAL_ACTION_NOTIFICATIONS)

// 빠른 설정 열기
performGlobalAction(GLOBAL_ACTION_QUICK_SETTINGS)
```

---

## 6. 고급 패턴: 화면 콘텐츠 모니터링과 상태 머신

실제 프로덕션 접근성 서비스는 단순한 이벤트 처리를 넘어 **상태 머신 패턴**으로 구현된다.

```kotlin
class WorkflowAccessibilityService : AccessibilityService() {

    private enum class State { IDLE, WAITING_FOR_DIALOG, FILLING_FORM, DONE }
    private var currentState = State.IDLE

    override fun onAccessibilityEvent(event: AccessibilityEvent) {
        val pkg = event.packageName?.toString() ?: return
        if (pkg != TARGET_PACKAGE) return

        when (currentState) {
            State.IDLE -> {
                if (event.eventType == AccessibilityEvent.TYPE_WINDOW_STATE_CHANGED) {
                    val windowTitle = event.text.joinToString()
                    if (windowTitle.contains("결제 확인")) {
                        currentState = State.WAITING_FOR_DIALOG
                        handlePaymentDialog()
                    }
                }
            }
            State.WAITING_FOR_DIALOG -> {
                if (event.eventType == AccessibilityEvent.TYPE_VIEW_CLICKED) {
                    currentState = State.FILLING_FORM
                    fillInForm()
                }
            }
            State.FILLING_FORM -> {
                if (event.eventType == AccessibilityEvent.TYPE_WINDOW_STATE_CHANGED &&
                    event.text.any { it.contains("완료") }) {
                    currentState = State.DONE
                    notifyUser("작업 완료")
                }
            }
            State.DONE -> currentState = State.IDLE
        }
    }

    private fun handlePaymentDialog() {
        rootInActiveWindow?.let { root ->
            val confirmBtn = root.findAccessibilityNodeInfosByText("확인").firstOrNull()
            confirmBtn?.performAction(AccessibilityNodeInfo.ACTION_CLICK)
            confirmBtn?.recycle()
            root.recycle()
        }
    }

    private fun fillInForm() { /* 폼 작성 로직 */ }
    private fun notifyUser(msg: String) { /* 알림 로직 */ }

    override fun onInterrupt() {}

    companion object {
        private const val TARGET_PACKAGE = "com.example.targetapp"
    }
}
```

---

## 7. 주의사항과 실전 팁

### 7.1 성능 최적화

- `typeAllMask` 대신 필요한 이벤트만 구독하라. 모든 이벤트 수신은 CPU 사용률을 크게 높인다.
- `notificationTimeout`을 적절히 설정해 이벤트 폭풍(event storm)을 완화하라.
- `onAccessibilityEvent()`는 메인 스레드에서 호출되므로 무거운 작업은 별도 코루틴이나 HandlerThread에서 수행하라.

```kotlin
private val serviceScope = CoroutineScope(Dispatchers.Default + SupervisorJob())

override fun onAccessibilityEvent(event: AccessibilityEvent) {
    // 무거운 노드 탐색은 백그라운드에서
    serviceScope.launch {
        processEventAsync(event.packageName?.toString())
    }
}

override fun onDestroy() {
    serviceScope.cancel()
    super.onDestroy()
}
```

### 7.2 노드 메모리 관리

API 33 미만에서는 `AccessibilityNodeInfo`를 직접 `recycle()`해야 메모리 누수를 방지할 수 있다. 확장 함수로 안전하게 처리하는 패턴:

```kotlin
inline fun <T> AccessibilityNodeInfo.use(block: (AccessibilityNodeInfo) -> T): T {
    return try {
        block(this)
    } finally {
        recycle()
    }
}

// 사용 예
event.source?.use { node ->
    Log.d("A11y", "Node text: ${node.text}")
}
```

### 7.3 보안 고려사항

- 접근성 서비스는 화면의 모든 텍스트(비밀번호 포함)를 읽을 수 있다. 패스워드 필드(`isPassword = true`)의 텍스트는 반드시 마스킹하라.
- `canRetrieveWindowContent`는 꼭 필요할 때만 활성화하라.
- Google Play 정책상 비접근성 목적(광고 클릭, 게임 자동화 등)으로 사용하면 앱이 삭제된다.

### 7.4 기기 호환성 체크

```kotlin
// GestureDescription은 API 24+
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {
    performTap(x, y)
}

// FingerprintGestureController는 API 26+
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
    val controller = fingerprintGestureController
    // ...
}
```

### 7.5 서비스 활성화 유도

사용자는 직접 설정 > 접근성에서 서비스를 켜야 한다. 앱에서 직접 이동:

```kotlin
fun openAccessibilitySettings() {
    val intent = Intent(Settings.ACTION_ACCESSIBILITY_SETTINGS)
    startActivity(intent)
}

fun isServiceEnabled(context: Context, serviceClass: Class<*>): Boolean {
    val expectedComponentName = ComponentName(context, serviceClass)
    val enabledServices = Settings.Secure.getString(
        context.contentResolver,
        Settings.Secure.ENABLED_ACCESSIBILITY_SERVICES
    ) ?: return false
    return enabledServices.split(":").any {
        ComponentName.unflattenFromString(it) == expectedComponentName
    }
}
```

---

## 8. 정리

`AccessibilityService`는 Android 접근성 생태계의 핵심이다. 단순한 이벤트 수신부터 UI 계층 탐색, 프로그래밍 제스처까지 폭넓은 능력을 갖추고 있다. 올바른 목적(장애 사용자 지원, 엔터프라이즈 자동화)으로 사용할 때 강력한 도구가 된다.

핵심 체크리스트:
- `BIND_ACCESSIBILITY_SERVICE` 권한 선언 필수
- `canRetrieveWindowContent`와 `canPerformGestures`는 필요한 경우만 활성화
- 이벤트 처리는 최소한으로, 노드는 반드시 `recycle()`
- 무거운 작업은 백그라운드 스레드/코루틴으로 분리
- 비밀번호 필드 텍스트 접근 금지

## 참고 자료
- [Android 공식 접근성 서비스 가이드](https://developer.android.com/guide/topics/ui/accessibility/service)
- [AccessibilityService API 레퍼런스](https://developer.android.com/reference/android/accessibilityservice/AccessibilityService)
- [GestureDescription API 레퍼런스](https://developer.android.com/reference/kotlin/android/accessibilityservice/GestureDescription)
- [Create your own accessibility service (Views)](https://developer.android.com/guide/topics/ui/accessibility/views/service)
