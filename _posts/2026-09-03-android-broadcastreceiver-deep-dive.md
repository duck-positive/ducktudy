---
layout: post
title: "Android BroadcastReceiver 심화: 정적/동적 등록부터 Android 14+ 제한, Flow 기반 대안 패턴까지 완전 정복"
date: 2026-09-03
categories: [android]
tags: [android, broadcastreceiver, kotlin, flow, intent, broadcast, coroutines]
---

Android 시스템의 메시지 버스라 할 수 있는 **BroadcastReceiver**는 컴포넌트 간 이벤트를 전달하는 핵심 메커니즘입니다. 배터리 변화, 네트워크 상태, 화면 켜짐/꺼짐 등 시스템 이벤트를 수신하거나, 앱 내부 컴포넌트 간에 데이터를 주고받을 때 광범위하게 사용됩니다. 그러나 Android 8.0(API 26)부터 시작된 백그라운드 실행 제한, Android 14의 암묵적 브로드캐스트 제약, `LocalBroadcastManager` 의 공식 Deprecation까지 — BroadcastReceiver를 올바르게 사용하려면 버전별 동작 차이와 현대적인 대안 패턴을 깊이 이해해야 합니다.

---

## 1. BroadcastReceiver란 무엇인가

BroadcastReceiver는 Android의 4대 컴포넌트 중 하나로, `Intent` 를 래핑한 브로드캐스트 메시지를 수신하는 역할을 합니다. 발신자와 수신자가 서로를 직접 알지 못해도 되는 **Publish-Subscribe** 구조를 따르며, 시스템 이벤트부터 커스텀 앱 간 통신까지 다양하게 활용됩니다.

브로드캐스트의 종류는 크게 세 가지입니다.

- **일반 브로드캐스트(Normal Broadcast)**: `sendBroadcast()`로 전송. 등록된 모든 수신자에게 비결정적 순서로 동시에 전달됩니다.
- **순서형 브로드캐스트(Ordered Broadcast)**: `sendOrderedBroadcast()`로 전송. 우선순위(`android:priority`) 순서에 따라 하나씩 전달되며, 각 수신자가 결과를 수정하거나 중단(abort)할 수 있습니다.
- **로컬 브로드캐스트(Local Broadcast)**: 동일 프로세스 내에서만 전달. 이전에는 `LocalBroadcastManager`로 구현했으나 현재 Deprecated 상태입니다.

---

## 2. 왜 알아야 하는가

### Android 8.0 이후 정적 등록 제한

Android 8.0(Oreo, API 26)부터 대부분의 **암묵적 브로드캐스트(Implicit Broadcast)**에 대해 Manifest 정적 등록이 금지되었습니다. 이 변경은 앱이 실행 중이지 않아도 수십 개의 앱이 동일 브로드캐스트에 반응해 일제히 깨어나는 "Thundering Herd" 문제를 방지하기 위한 것입니다.

예를 들어 `ACTION_CONNECTIVITY_CHANGE`는 정적 등록 불가 목록에 포함되어 있습니다. 반면 `ACTION_BOOT_COMPLETED`, `ACTION_LOCKED_BOOT_COMPLETED` 등 일부 브로드캐스트는 [예외 목록](https://developer.android.com/develop/background-work/background-tasks/broadcasts/broadcast-exceptions)에 포함되어 여전히 정적 등록이 허용됩니다.

### Android 14의 추가 제약

Android 14(API 34)에서는 `registerReceiver()` 호출 시 `RECEIVER_EXPORTED` 또는 `RECEIVER_NOT_EXPORTED` 플래그를 **명시적으로 지정**해야 합니다. 생략하면 `SecurityException`이 발생합니다. 또한 캐시된(Cached) 상태의 앱에 대해 덜 중요한 브로드캐스트 전달을 지연시키는 정책이 강화되었습니다.

### LocalBroadcastManager Deprecation

`LocalBroadcastManager`는 AndroidX Core 1.1.0(2020년)부터 공식 Deprecated 처리되었습니다. Google은 동일 프로세스 내 통신에는 Kotlin Flow, SharedFlow, LiveData 등 현대적인 옵저버 패턴을 사용하도록 권장합니다.

---

## 3. 등록 방식 완전 정복

### 3-1. Manifest 정적 등록 (Static Registration)

정적 등록은 앱이 실행 중이지 않아도 시스템이 이벤트 발생 시 앱을 깨워 수신자를 실행합니다. 단, Android 8.0 이후 암묵적 브로드캐스트에는 사용할 수 없습니다.

```xml
<!-- AndroidManifest.xml -->
<receiver
    android:name=".BootCompletedReceiver"
    android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED" />
    </intent-filter>
</receiver>
```

```kotlin
// BootCompletedReceiver.kt
class BootCompletedReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        if (intent.action == Intent.ACTION_BOOT_COMPLETED) {
            // 부팅 완료 후 WorkManager 작업 재스케줄링
            WorkManager.getInstance(context).enqueueUniquePeriodicWork(
                "sync_work",
                ExistingPeriodicWorkPolicy.KEEP,
                PeriodicWorkRequestBuilder<SyncWorker>(1, TimeUnit.HOURS).build()
            )
        }
    }
}
```

`onReceive()`는 **메인 스레드**에서 호출되며 약 10초 이내에 완료해야 합니다. 장시간 작업이 필요하다면 반드시 `WorkManager`나 `JobScheduler`에 위임해야 합니다. `goAsync()`를 사용해 비동기 처리를 하는 방법도 있지만, 이 경우에도 60초 이내에 `PendingResult.finish()`를 호출해야 합니다.

### 3-2. 동적 등록 (Dynamic Registration) — Android 14+ 대응

동적 등록은 Context가 유효한 동안만 수신하므로, 라이프사이클과 함께 관리해야 합니다.

```kotlin
// Android 14+ RECEIVER_NOT_EXPORTED 명시 필수
class NetworkStateActivity : AppCompatActivity() {

    private val networkReceiver = object : BroadcastReceiver() {
        override fun onReceive(context: Context, intent: Intent) {
            val action = intent.action ?: return
            when (action) {
                ConnectivityManager.CONNECTIVITY_ACTION -> {
                    val noConnectivity = intent.getBooleanExtra(
                        ConnectivityManager.EXTRA_NO_CONNECTIVITY, false
                    )
                    updateUi(isConnected = !noConnectivity)
                }
            }
        }
    }

    override fun onStart() {
        super.onStart()
        val filter = IntentFilter(ConnectivityManager.CONNECTIVITY_ACTION)
        // Android 14(API 34) 이상: 플래그 명시 필수
        ContextCompat.registerReceiver(
            this,
            networkReceiver,
            filter,
            ContextCompat.RECEIVER_NOT_EXPORTED  // 외부 앱에서 접근 불가
        )
    }

    override fun onStop() {
        super.onStop()
        unregisterReceiver(networkReceiver)
    }

    private fun updateUi(isConnected: Boolean) {
        // UI 업데이트
    }
}
```

`ContextCompat.registerReceiver()`는 AndroidX Core 1.9.0 이상에서 사용 가능하며, API 레벨에 따라 자동으로 플래그를 처리해 하위 호환성을 보장합니다.

---

## 4. 현대적 대안: Kotlin Flow + SharedFlow 기반 이벤트 버스

동일 프로세스 내 컴포넌트 간 이벤트 전달은 `LocalBroadcastManager` 대신 **SharedFlow**를 활용하는 것이 권장됩니다. 이 방식은 라이프사이클 인식, 타입 안전성, 백프레셔 처리가 모두 가능합니다.

```kotlin
// AppEventBus.kt — 싱글턴 이벤트 버스
sealed class AppEvent {
    data class UserLoggedIn(val userId: String) : AppEvent()
    data class DataSynced(val count: Int) : AppEvent()
    object SessionExpired : AppEvent()
}

object AppEventBus {
    private val _events = MutableSharedFlow<AppEvent>(
        replay = 0,
        extraBufferCapacity = 64,
        onBufferOverflow = BufferOverflow.DROP_OLDEST
    )
    val events: SharedFlow<AppEvent> = _events.asSharedFlow()

    suspend fun emit(event: AppEvent) {
        _events.emit(event)
    }

    fun tryEmit(event: AppEvent): Boolean {
        return _events.tryEmit(event)
    }
}
```

```kotlin
// HomeFragment.kt — 수신 측 (라이프사이클 인식)
class HomeFragment : Fragment() {

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        // repeatOnLifecycle: STARTED 상태일 때만 수집
        viewLifecycleOwner.lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                AppEventBus.events
                    .filterIsInstance<AppEvent.DataSynced>()
                    .collect { event ->
                        showSyncResult(event.count)
                    }
            }
        }
    }

    private fun showSyncResult(count: Int) {
        Toast.makeText(context, "동기화 완료: ${count}건", Toast.LENGTH_SHORT).show()
    }
}

// SyncService.kt — 발신 측
class SyncService : Service() {
    private val serviceScope = CoroutineScope(Dispatchers.IO + SupervisorJob())

    fun performSync() {
        serviceScope.launch {
            val syncedCount = runSync()
            AppEventBus.emit(AppEvent.DataSynced(syncedCount))
        }
    }

    private suspend fun runSync(): Int {
        // 실제 동기화 로직
        delay(1000)
        return 42
    }

    override fun onDestroy() {
        serviceScope.cancel()
        super.onDestroy()
    }

    override fun onBind(intent: Intent?): IBinder? = null
}
```

`SharedFlow`는 `replay = 0`으로 설정하면 이전 이벤트를 재수신하지 않아 브로드캐스트 의미론에 더 가깝습니다. 반면 `StateFlow`는 최신 상태를 보존하는 특성이 있으므로 이벤트보다는 **상태 공유**에 적합합니다.

---

## 5. 시스템 브로드캐스트 실전: 배터리 상태 모니터링

시스템 브로드캐스트 중 `ACTION_BATTERY_CHANGED`는 `registerReceiver(null, filter)` 패턴으로 **현재 상태를 즉시 조회**할 수 있는 Sticky Broadcast입니다.

```kotlin
// BatteryMonitor.kt
class BatteryMonitor(private val context: Context) {

    fun getCurrentBatteryStatus(): BatteryInfo {
        val filter = IntentFilter(Intent.ACTION_BATTERY_CHANGED)
        // Sticky Broadcast: null 수신자로 현재 상태 즉시 반환
        val intent = context.registerReceiver(null, filter)
        return intent?.toBatteryInfo() ?: BatteryInfo.Unknown
    }

    fun observeBatteryStatus(): Flow<BatteryInfo> = callbackFlow {
        val receiver = object : BroadcastReceiver() {
            override fun onReceive(context: Context, intent: Intent) {
                trySend(intent.toBatteryInfo())
            }
        }
        val filter = IntentFilter().apply {
            addAction(Intent.ACTION_BATTERY_CHANGED)
            addAction(Intent.ACTION_BATTERY_LOW)
            addAction(Intent.ACTION_BATTERY_OKAY)
        }
        ContextCompat.registerReceiver(
            context, receiver, filter, ContextCompat.RECEIVER_NOT_EXPORTED
        )
        awaitClose { context.unregisterReceiver(receiver) }
    }

    private fun Intent.toBatteryInfo(): BatteryInfo {
        val level = getIntExtra(BatteryManager.EXTRA_LEVEL, -1)
        val scale = getIntExtra(BatteryManager.EXTRA_SCALE, -1)
        val status = getIntExtra(BatteryManager.EXTRA_STATUS, -1)
        val percentage = if (level >= 0 && scale > 0) level * 100 / scale else -1
        val isCharging = status == BatteryManager.BATTERY_STATUS_CHARGING ||
                status == BatteryManager.BATTERY_STATUS_FULL
        return BatteryInfo(percentage, isCharging)
    }
}

data class BatteryInfo(val percentage: Int, val isCharging: Boolean) {
    companion object {
        val Unknown = BatteryInfo(-1, false)
    }
}
```

`callbackFlow`를 사용하면 BroadcastReceiver의 콜백 기반 API를 Kotlin Flow로 자연스럽게 래핑할 수 있습니다. `awaitClose` 블록에서 `unregisterReceiver`를 호출하므로 메모리 누수 걱정이 없습니다.

---

## 6. 주의사항 및 팁

### onReceive()에서 절대 하지 말아야 할 것들

- **장시간 작업**: 10초 초과 시 ANR 발생. `WorkManager`에 위임하세요.
- **UI 업데이트**: `onReceive()`는 백그라운드에서도 호출될 수 있습니다.
- **코루틴 `launch`**: BroadcastReceiver가 반환되면 프로세스가 종료될 수 있으므로, `goAsync()`와 함께 사용해야 합니다.

### 보안 고려사항

- 외부 앱이 수신자를 악용하지 못하도록 `android:exported="false"` 설정을 기본으로 하세요.
- 민감한 데이터를 브로드캐스트에 담지 마세요. 다른 앱이 스니핑할 수 있습니다.
- `sendBroadcast(intent, permission)` 형태로 퍼미션을 지정하면 해당 퍼미션을 가진 앱만 수신할 수 있습니다.
- `intent.setPackage("com.example.app")`으로 수신 앱을 한정하면 암묵적 브로드캐스트의 보안 위협을 줄일 수 있습니다.

### 버전별 체크리스트

| Android 버전 | 주요 변경 사항 |
|---|---|
| 7.0 (API 24) | `ACTION_NEW_PICTURE`, `ACTION_NEW_VIDEO` 정적 등록 불가 |
| 8.0 (API 26) | 대부분의 암묵적 브로드캐스트 정적 등록 금지 |
| 9.0 (API 28) | `NETWORK_STATE_CHANGED_ACTION` 등 추가 제한 |
| 14 (API 34) | `registerReceiver()` 플래그 명시 의무화 |
| 16 (API 36) | 프로세스 간 브로드캐스트 우선순위 순서 보장 제거 |

### 언제 BroadcastReceiver를 쓰고 언제 Flow를 쓸까

| 시나리오 | 권장 방식 |
|---|---|
| 시스템 이벤트 수신 (부팅, 충전 등) | BroadcastReceiver (동적/정적) |
| 앱 내부 컴포넌트 간 이벤트 | SharedFlow / StateFlow |
| 다른 앱과의 명시적 통신 | 명시적 Intent (setPackage 설정) |
| 실시간 상태 모니터링 | callbackFlow로 래핑 |

---

## 마무리

BroadcastReceiver는 여전히 Android 생태계에서 없어서는 안 될 메커니즘입니다. 그러나 버전별 제약이 점점 강해지면서, 실무에서는 **시스템 이벤트 수신**에만 BroadcastReceiver를 한정하고, 앱 내부 통신은 Kotlin Flow 기반으로 전환하는 것이 최선의 패턴입니다. `callbackFlow`를 활용하면 기존 콜백 기반 BroadcastReceiver를 Flow로 깔끔하게 래핑할 수 있으며, 라이프사이클 인식과 구조적 동시성의 이점을 모두 누릴 수 있습니다.

## 참고 자료
- [Broadcasts overview - Android Developers](https://developer.android.com/develop/background-work/background-tasks/broadcasts)
- [BroadcastReceiver API Reference](https://developer.android.com/reference/android/content/BroadcastReceiver)
- [Behavior changes: Apps targeting Android 14 or higher](https://developer.android.com/about/versions/14/behavior-changes-14)
- [LocalBroadcastManager - AndroidX Releases](https://developer.android.com/jetpack/androidx/releases/localbroadcastmanager)
