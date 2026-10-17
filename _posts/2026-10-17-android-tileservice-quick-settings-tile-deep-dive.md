---
layout: post
title: "Android TileService 심화: Quick Settings 타일로 앱 기능을 알림창에서 바로 실행하기"
date: 2026-10-17
categories: [android]
tags: [android, tileservice, quicksettings, kotlin, coroutines, service]
---

Android의 **Quick Settings**는 화면 상단에서 아래로 스와이프할 때 나타나는 빠른 설정 패널입니다. Wi-Fi 토글, 블루투스 활성화, 화면 밝기 조절처럼 자주 쓰는 기능을 한 번의 탭으로 실행할 수 있죠. Android 7.0(API 24)부터는 서드파티 앱도 여기에 **커스텀 타일**을 추가할 수 있게 됐습니다. 이 글에서는 `TileService` API를 처음부터 끝까지 파헤쳐서, 완성도 높은 Quick Settings 타일을 직접 구현하는 방법을 알아봅니다.

---

## Quick Settings 타일이란?

Quick Settings 타일은 `Tile` 객체와 `TileService`로 이루어집니다.

- **`Tile`**: 타일의 아이콘, 라벨, 상태(State)를 담는 데이터 객체입니다.
- **`TileService`**: `Service`의 서브클래스로, 타일의 라이프사이클 콜백을 제공합니다.

타일에는 세 가지 상태가 있습니다:

| 상태 | 상수 | 의미 |
|---|---|---|
| 활성 | `Tile.STATE_ACTIVE` | 기능이 켜진 상태 (파란색으로 강조) |
| 비활성 | `Tile.STATE_INACTIVE` | 기능이 꺼진 상태 (회색) |
| 사용 불가 | `Tile.STATE_UNAVAILABLE` | 조건 미충족으로 탭 불가 (반투명 처리) |

---

## 왜 Quick Settings 타일이 필요한가?

앱을 열지 않고도 핵심 기능을 제어할 수 있다는 점이 가장 큰 장점입니다.

**적합한 사용 사례:**
- VPN 앱의 연결/해제 토글
- 자동화 앱의 특정 루틴 즉시 실행
- 음악 앱의 재생/정지 제어
- 집중 모드, DND 커스텀 프리셋 활성화

**부적합한 사용 사례:**
- 단순히 앱을 여는 런처 역할 (앱 아이콘과 다를 바 없음)
- 복잡한 UI가 필요한 일회성 설정
- 앱당 3개 이상의 타일 (사용자 패널이 지저분해짐)

Quick Settings 타일은 **자주, 빠르게** 쓰이는 기능에만 쓰는 게 원칙입니다.

---

## 구현 예제 1: 기본 토글 타일

가장 흔한 패턴인 **ON/OFF 토글 타일**을 구현합니다. 앱 내의 특정 기능(예: 야간 모드)을 빠르게 켜고 끄는 예제입니다.

### 1단계: 아이콘 준비

Quick Settings 타일 아이콘은 **24×24dp, 흰색, 투명 배경의 VectorDrawable**이어야 합니다. `res/drawable/ic_tile_night.xml`을 만듭니다.

### 2단계: TileService 구현

```kotlin
import android.graphics.drawable.Icon
import android.service.quicksettings.Tile
import android.service.quicksettings.TileService
import androidx.core.service.quicksettings.TileServiceCompat

class NightModeTileService : TileService() {

    // 타일이 패널에 추가될 때 1회 호출 (초기화에 사용)
    override fun onTileAdded() {
        super.onTileAdded()
        updateTileState()
    }

    // 타일이 화면에 보일 때마다 호출 — 여기서 상태를 최신화
    override fun onStartListening() {
        super.onStartListening()
        updateTileState()
    }

    // 타일을 탭했을 때 호출
    override fun onClick() {
        super.onClick()

        // 잠금 화면에서의 보안 처리
        if (isLocked) {
            // 잠금을 해제하고 실행 (사용자에게 잠금 해제 요청)
            unlockAndRun { toggleNightMode() }
        } else {
            toggleNightMode()
        }
    }

    // 타일이 패널에서 제거될 때 호출
    override fun onTileRemoved() {
        super.onTileRemoved()
        // 리소스 정리 등 필요 시 처리
    }

    private fun toggleNightMode() {
        val prefs = getSharedPreferences("app_prefs", MODE_PRIVATE)
        val isNightMode = prefs.getBoolean("night_mode", false)
        prefs.edit().putBoolean("night_mode", !isNightMode).apply()
        updateTileState()
    }

    private fun updateTileState() {
        val prefs = getSharedPreferences("app_prefs", MODE_PRIVATE)
        val isNightMode = prefs.getBoolean("night_mode", false)

        qsTile?.apply {
            state = if (isNightMode) Tile.STATE_ACTIVE else Tile.STATE_INACTIVE
            label = if (isNightMode) "야간 모드 ON" else "야간 모드 OFF"
            contentDescription = label // 접근성
            icon = Icon.createWithResource(
                this@NightModeTileService,
                if (isNightMode) R.drawable.ic_tile_night_on
                else R.drawable.ic_tile_night_off
            )
            updateTile() // ← 반드시 호출해야 UI에 반영됨
        }
    }
}
```

### 3단계: AndroidManifest.xml 등록

```xml
<service
    android:name=".NightModeTileService"
    android:icon="@drawable/ic_tile_night"
    android:label="@string/tile_night_mode"
    android:permission="android.permission.BIND_QUICK_SETTINGS_TILE"
    android:exported="true">

    <intent-filter>
        <action android:name="android.service.quicksettings.action.QS_TILE" />
    </intent-filter>

    <!-- 활성 모드: 시스템이 필요할 때만 서비스를 바인딩 (권장) -->
    <meta-data
        android:name="android.service.quicksettings.ACTIVE_TILE"
        android:value="true" />

    <!-- 접근성: 두 가지 상태를 전환하는 타일임을 선언 -->
    <meta-data
        android:name="android.service.quicksettings.TOGGLEABLE_TILE"
        android:value="true" />

    <!-- Android 15+: 타일 카테고리 분류 -->
    <meta-data
        android:name="android.service.quicksettings.TILE_CATEGORY"
        android:value="display" />
</service>
```

> **활성 모드(Active Mode)란?**  
> `ACTIVE_TILE` 메타데이터를 선언하면 `onStartListening()`/`onStopListening()`이 타일이 보일 때만 호출되는 대신, 앱이 직접 `TileService.requestListeningState()`를 호출해 업데이트를 요청하는 방식으로 바뀝니다. 불필요한 백그라운드 실행을 막아 배터리 효율이 높습니다.

---

## 구현 예제 2: Kotlin Coroutine 연동 심화

실제 앱에서는 타일 상태를 **Room 데이터베이스나 DataStore**에서 비동기로 읽어와야 합니다. `TileService`는 일반 `Service`이므로 코루틴을 직접 쓸 수 있습니다.

```kotlin
import android.service.quicksettings.Tile
import android.service.quicksettings.TileService
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.first

class VpnTileService : TileService() {

    // TileService의 수명과 연동된 코루틴 스코프
    private val scope = CoroutineScope(Dispatchers.Main + SupervisorJob())

    // DataStore 등 리포지토리 — 실제로는 DI로 주입하거나 lazy init
    private val vpnRepository by lazy { VpnRepository(applicationContext) }

    override fun onStartListening() {
        super.onStartListening()
        // Flow를 구독해 실시간 상태 반영
        scope.launch {
            vpnRepository.isConnectedFlow
                .collect { isConnected ->
                    updateTile(isConnected)
                }
        }
    }

    override fun onStopListening() {
        super.onStopListening()
        // 타일이 보이지 않으면 구독 중단 — 리소스 낭비 방지
        scope.coroutineContext.cancelChildren()
    }

    override fun onClick() {
        super.onClick()
        // 현재 상태를 읽은 후 반전
        scope.launch {
            val isConnected = vpnRepository.isConnectedFlow.first()
            if (isConnected) {
                vpnRepository.disconnect()
            } else {
                // VPN 연결은 민감한 작업 — 잠금 해제 후 실행
                if (isLocked) {
                    unlockAndRun {
                        scope.launch { vpnRepository.connect() }
                    }
                } else {
                    vpnRepository.connect()
                }
            }
        }
    }

    override fun onDestroy() {
        super.onDestroy()
        scope.cancel() // 서비스 종료 시 모든 코루틴 취소
    }

    private fun updateTile(isConnected: Boolean) {
        qsTile?.apply {
            state = when {
                isConnected -> Tile.STATE_ACTIVE
                else -> Tile.STATE_INACTIVE
            }
            label = if (isConnected) "VPN 연결됨" else "VPN 끊김"
            contentDescription = label
            updateTile()
        }
    }
}
```

### Android 13+: 타일 추가 요청 다이얼로그

Android 13(API 33)부터는 사용자가 Quick Settings 편집 모드에 직접 들어가지 않아도, 앱에서 타일 추가 다이얼로그를 띄울 수 있습니다.

```kotlin
// Activity나 Fragment에서 호출
fun requestAddTile(context: Context) {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
        val statusBarManager = context.getSystemService(StatusBarManager::class.java)
        statusBarManager.requestAddTileService(
            ComponentName(context, VpnTileService::class.java),
            "VPN 타일",  // 다이얼로그에 표시될 타일 이름
            Icon.createWithResource(context, R.drawable.ic_tile_vpn),
            Executors.newSingleThreadExecutor()
        ) { resultCode ->
            when (resultCode) {
                StatusBarManager.TILE_ADD_REQUEST_RESULT_TILE_ADDED ->
                    Log.d("Tile", "타일이 추가됐습니다")
                StatusBarManager.TILE_ADD_REQUEST_RESULT_TILE_ALREADY_ADDED ->
                    Log.d("Tile", "이미 추가된 타일입니다")
                StatusBarManager.TILE_ADD_REQUEST_RESULT_TILE_NOT_ADDED ->
                    Log.d("Tile", "사용자가 추가를 거부했습니다")
            }
        }
    }
}
```

---

## 활성 모드에서 외부 업데이트 반영하기

활성 모드 타일은 `onStartListening()`이 자동 호출되지 않기 때문에, **백그라운드에서 상태가 바뀌었을 때** 다음과 같이 타일 업데이트를 직접 요청해야 합니다.

```kotlin
// 앱 내 어디서든 (예: VPN 상태가 바뀌었을 때 호출)
fun notifyTileUpdate(context: Context) {
    TileService.requestListeningState(
        context,
        ComponentName(context, VpnTileService::class.java)
    )
    // → VpnTileService.onStartListening() 가 일시적으로 호출됨
}
```

---

## 주의사항 및 팁

### 1. 타일 라벨은 18자 이하
Quick Settings 패널의 공간이 제한적이기 때문에 Android는 18자 이내의 라벨을 권장합니다. 초과하면 말줄임표로 잘립니다.

### 2. UI 스레드에서 `updateTile()` 호출
`updateTile()`은 반드시 메인(UI) 스레드에서 호출해야 합니다. 코루틴에서 사용할 때는 `Dispatchers.Main`으로 전환하세요.

### 3. `showDialog()` vs `startActivityAndCollapse()`
- **`showDialog()`**: 잠금 화면에는 표시되지 않습니다. 간단한 확인 다이얼로그에 적합합니다.
- **`startActivityAndCollapse()`**: Quick Settings 패널을 닫고 앱을 전면에 띄웁니다. API 34부터 `Intent` 대신 `PendingIntent`를 받습니다.

### 4. `isLocked` vs `isSecure`
- **`isLocked`**: 화면이 잠긴 상태인지 여부입니다.
- **`isSecure`**: 잠금 화면에 PIN/패턴 등 보안이 설정되어 있는지 여부입니다.
- 민감한 기능(결제, 개인정보 조회)은 `isSecure && isLocked`를 모두 확인한 뒤 `unlockAndRun()`을 사용하세요.

### 5. DI와 TileService
Hilt는 `Service`를 지원하지만 TileService는 시스템이 직접 바인딩하므로 `@AndroidEntryPoint` 어노테이션만 추가하면 의존성 주입이 그대로 동작합니다.

```kotlin
@AndroidEntryPoint
class VpnTileService : TileService() {
    @Inject lateinit var vpnRepository: VpnRepository
    // ...
}
```

### 6. OEM별 차이
삼성 One UI, Pixel UI, Xiaomi MIUI 등 제조사마다 Quick Settings 패널의 디자인과 타일 표시 개수가 다릅니다. 자동화 테스트보다 실기기 확인을 권장합니다.

---

## 정리

| 항목 | 핵심 포인트 |
|---|---|
| 최소 API | 24 (Android 7.0) |
| 필수 권한 | `BIND_QUICK_SETTINGS_TILE` |
| 권장 모드 | 활성 모드(`ACTIVE_TILE = true`) |
| 라벨 최대 길이 | 18자 |
| 상태 갱신 방법 | `qsTile.updateTile()` |
| 외부 상태 반영 | `TileService.requestListeningState()` |
| Android 13+ 타일 추가 | `StatusBarManager.requestAddTileService()` |
| Hilt 지원 | `@AndroidEntryPoint` 추가 |

Quick Settings 타일은 구현 자체는 간단하지만, **라이프사이클 관리**와 **배터리 효율**을 제대로 신경 쓰지 않으면 사용자 경험을 오히려 해칩니다. 활성 모드를 기본으로 사용하고, `onStopListening()`에서 반드시 코루틴을 취소하며, 상태 업데이트는 `Tile.updateTile()` 호출 전까지 UI에 반영되지 않는다는 점만 기억해도 대부분의 문제를 피할 수 있습니다.

## 참고 자료
- [Create custom Quick Settings tiles for your app — Android Developers](https://developer.android.com/develop/ui/views/quicksettings-tiles)
- [TileService API Reference — Android Developers](https://developer.android.com/reference/android/service/quicksettings/TileService)
- [Tile API Reference — Android Developers](https://developer.android.com/reference/android/service/quicksettings/Tile)
- [TileServiceCompat (AndroidX) — Android Developers](https://developer.android.com/reference/androidx/core/service/quicksettings/TileServiceCompat)
