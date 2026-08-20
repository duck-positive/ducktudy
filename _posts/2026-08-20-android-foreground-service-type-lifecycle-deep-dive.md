---
layout: post
title: "Android Foreground Service 심화: ForegroundServiceType, 생명주기, Android 14+ 완전 정복"
date: 2026-08-20
categories: [android]
tags: [android, foreground-service, foreground-service-type, notification, kotlin, background-work]
---

Android 앱 개발에서 백그라운드 작업은 피할 수 없는 주제입니다. 음악 재생, 파일 다운로드, 위치 추적 등 사용자가 앱을 떠나도 계속되어야 하는 작업들이 있죠. 이런 작업을 안정적으로 처리하는 핵심 컴포넌트가 바로 **Foreground Service**입니다. 특히 Android 14(API 34)부터 `ForegroundServiceType` 선언이 의무화되면서 올바른 구현이 더욱 중요해졌습니다.

---

## Foreground Service란?

Android의 서비스는 크게 세 가지로 나뉩니다.

- **Started Service**: `startService()`로 시작, 백그라운드에서 실행
- **Bound Service**: 컴포넌트와 바인딩, 바인딩이 끊기면 종료
- **Foreground Service**: 사용자에게 알림(Notification)을 표시하며 실행되는 서비스

Foreground Service는 시스템이 메모리 부족 상황에서도 쉽게 종료하지 않습니다. 사용자는 상태 표시줄의 알림을 통해 해당 서비스가 실행 중임을 항상 인지할 수 있어야 하기 때문입니다.

## 왜 Foreground Service가 필요한가?

Android 8.0(Oreo)부터 시작된 **백그라운드 실행 제한**이 핵심 이유입니다.

- **Doze 모드**: 화면이 꺼지고 일정 시간이 지나면 네트워크/CPU를 제한
- **앱 대기 버킷(App Standby Buckets)**: 사용 빈도에 따라 리소스 제한
- **백그라운드 서비스 제한**: 앱이 백그라운드에 있을 때 백그라운드 서비스 시작 불가

이 제한들을 피하면서 장기 실행 작업을 처리하는 올바른 방법이 Foreground Service입니다. 단, 배터리 소모에 직접적인 영향을 주므로 반드시 필요한 경우에만 사용해야 합니다.

### 언제 Foreground Service를 써야 하는가?

| 작업 유형 | 권장 솔루션 |
|-----------|------------|
| 음악/동영상 재생 | Foreground Service (mediaPlayback) |
| 파일 업로드/다운로드 | Foreground Service (dataSync) |
| 위치 추적 내비게이션 | Foreground Service (location) |
| 단발성 백그라운드 작업 | WorkManager |
| 짧은 지연 작업 | Coroutine + LifecycleScope |
| 주기적 작업 | WorkManager PeriodicWorkRequest |

---

## Android 14+의 ForegroundServiceType 의무화

Android 14(API 34)부터 Foreground Service를 시작할 때 반드시 `foregroundServiceType`을 Manifest에 선언해야 합니다. 선언하지 않으면 `MissingForegroundServiceTypeException`이 발생합니다.

### 주요 ForegroundServiceType 목록

| 타입 | Manifest 값 | 권한 | 용도 |
|------|------------|------|------|
| 미디어 재생 | `mediaPlayback` | `FOREGROUND_SERVICE_MEDIA_PLAYBACK` | 오디오/비디오 재생 |
| 위치 | `location` | `FOREGROUND_SERVICE_LOCATION` | 내비게이션, 위치 공유 |
| 데이터 동기화 | `dataSync` | `FOREGROUND_SERVICE_DATA_SYNC` | 파일 업다운로드, 백업 |
| 카메라 | `camera` | `FOREGROUND_SERVICE_CAMERA` | 백그라운드 카메라 |
| 마이크 | `microphone` | `FOREGROUND_SERVICE_MICROPHONE` | 음성 녹음 |
| 연결된 기기 | `connectedDevice` | `FOREGROUND_SERVICE_CONNECTED_DEVICE` | Bluetooth, NFC |
| 건강 | `health` | `FOREGROUND_SERVICE_HEALTH` | 운동 추적, 센서 |
| 단기 서비스 | `shortService` | `FOREGROUND_SERVICE` | ~3분 제한 임시 작업 |
| 특수 사용 | `specialUse` | `FOREGROUND_SERVICE_SPECIAL_USE` | Play 심사 필요 |

---

## 실제 구현 예제 1: 파일 다운로드 Foreground Service

### 1단계: Manifest 선언

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <!-- 기본 Foreground Service 권한 -->
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
    <!-- Android 14+ dataSync 타입 권한 -->
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE_DATA_SYNC" />
    <!-- 알림 권한 (Android 13+) -->
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />

    <application ...>
        <service
            android:name=".DownloadForegroundService"
            android:foregroundServiceType="dataSync"
            android:exported="false" />
    </application>
</manifest>
```

### 2단계: 서비스 구현

```kotlin
import android.app.Notification
import android.app.NotificationChannel
import android.app.NotificationManager
import android.app.PendingIntent
import android.app.Service
import android.content.Intent
import android.content.pm.ServiceInfo
import android.os.Build
import android.os.IBinder
import androidx.core.app.NotificationCompat
import androidx.core.app.ServiceCompat
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.Job
import kotlinx.coroutines.launch
import kotlinx.coroutines.withContext

class DownloadForegroundService : Service() {

    private val serviceScope = CoroutineScope(Dispatchers.IO + Job())
    private lateinit var notificationManager: NotificationManager

    companion object {
        const val CHANNEL_ID = "download_channel"
        const val NOTIFICATION_ID = 1001
        const val EXTRA_URL = "extra_url"
        const val EXTRA_FILE_NAME = "extra_file_name"

        fun startDownload(context: android.content.Context, url: String, fileName: String) {
            val intent = Intent(context, DownloadForegroundService::class.java).apply {
                putExtra(EXTRA_URL, url)
                putExtra(EXTRA_FILE_NAME, fileName)
            }
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
                context.startForegroundService(intent)
            } else {
                context.startService(intent)
            }
        }
    }

    override fun onCreate() {
        super.onCreate()
        notificationManager = getSystemService(NOTIFICATION_SERVICE) as NotificationManager
        createNotificationChannel()
    }

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        val url = intent?.getStringExtra(EXTRA_URL) ?: run {
            stopSelf()
            return START_NOT_STICKY
        }
        val fileName = intent.getStringExtra(EXTRA_FILE_NAME) ?: "unknown_file"

        // Android 14+: ServiceCompat.startForeground()로 타입 명시
        ServiceCompat.startForeground(
            this,
            NOTIFICATION_ID,
            buildNotification("준비 중...", 0),
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.UPSIDE_DOWN_CAKE) {
                ServiceInfo.FOREGROUND_SERVICE_TYPE_DATA_SYNC
            } else {
                0
            }
        )

        serviceScope.launch {
            performDownload(url, fileName)
        }

        return START_NOT_STICKY
    }

    private suspend fun performDownload(url: String, fileName: String) {
        try {
            // 실제 다운로드 시뮬레이션
            for (progress in 0..100 step 10) {
                withContext(Dispatchers.Main) {
                    updateNotification("$fileName 다운로드 중...", progress)
                }
                kotlinx.coroutines.delay(500)
            }
            withContext(Dispatchers.Main) {
                updateNotification("$fileName 다운로드 완료!", 100)
                kotlinx.coroutines.delay(2000)
                stopSelf()
            }
        } catch (e: Exception) {
            withContext(Dispatchers.Main) {
                updateNotification("다운로드 실패: ${e.message}", 0)
                stopSelf()
            }
        }
    }

    private fun createNotificationChannel() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            val channel = NotificationChannel(
                CHANNEL_ID,
                "파일 다운로드",
                NotificationManager.IMPORTANCE_LOW  // 소리 없이, 상태 표시줄 표시
            ).apply {
                description = "파일 다운로드 진행 상황"
                setShowBadge(false)
            }
            notificationManager.createNotificationChannel(channel)
        }
    }

    private fun buildNotification(message: String, progress: Int): Notification {
        val cancelIntent = Intent(this, DownloadForegroundService::class.java).apply {
            action = "ACTION_CANCEL"
        }
        val cancelPendingIntent = PendingIntent.getService(
            this, 0, cancelIntent,
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
        )

        return NotificationCompat.Builder(this, CHANNEL_ID)
            .setContentTitle("파일 다운로드")
            .setContentText(message)
            .setSmallIcon(android.R.drawable.stat_sys_download)
            .setProgress(100, progress, progress == 0)
            .setOngoing(true)
            .setSilent(true)
            .addAction(android.R.drawable.ic_delete, "취소", cancelPendingIntent)
            .build()
    }

    private fun updateNotification(message: String, progress: Int) {
        notificationManager.notify(NOTIFICATION_ID, buildNotification(message, progress))
    }

    override fun onBind(intent: Intent?): IBinder? = null

    override fun onDestroy() {
        super.onDestroy()
        serviceScope.coroutineContext[Job]?.cancel()
    }
}
```

---

## 실제 구현 예제 2: 위치 추적 Foreground Service

위치 기반 서비스(내비게이션, 운동 추적 등)는 `location` 타입을 사용합니다.

```kotlin
import android.annotation.SuppressLint
import android.app.Notification
import android.app.NotificationChannel
import android.app.NotificationManager
import android.app.Service
import android.content.Intent
import android.content.pm.ServiceInfo
import android.location.Location
import android.os.Build
import android.os.IBinder
import android.os.Looper
import androidx.core.app.NotificationCompat
import androidx.core.app.ServiceCompat
import com.google.android.gms.location.FusedLocationProviderClient
import com.google.android.gms.location.LocationCallback
import com.google.android.gms.location.LocationRequest
import com.google.android.gms.location.LocationResult
import com.google.android.gms.location.LocationServices
import com.google.android.gms.location.Priority

class LocationTrackingService : Service() {

    private lateinit var fusedLocationClient: FusedLocationProviderClient
    private lateinit var locationCallback: LocationCallback
    private lateinit var notificationManager: NotificationManager

    companion object {
        const val CHANNEL_ID = "location_tracking_channel"
        const val NOTIFICATION_ID = 2001
        private const val LOCATION_INTERVAL_MS = 5000L
    }

    override fun onCreate() {
        super.onCreate()
        fusedLocationClient = LocationServices.getFusedLocationProviderClient(this)
        notificationManager = getSystemService(NOTIFICATION_SERVICE) as NotificationManager
        createNotificationChannel()
        setupLocationCallback()
    }

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        // Android 14+: location 타입 명시
        ServiceCompat.startForeground(
            this,
            NOTIFICATION_ID,
            buildNotification("위치 추적 시작 중..."),
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.UPSIDE_DOWN_CAKE) {
                ServiceInfo.FOREGROUND_SERVICE_TYPE_LOCATION
            } else {
                0
            }
        )
        startLocationUpdates()
        return START_STICKY  // 시스템이 종료해도 재시작 필요
    }

    private fun setupLocationCallback() {
        locationCallback = object : LocationCallback() {
            override fun onLocationResult(result: LocationResult) {
                result.lastLocation?.let { location ->
                    onLocationReceived(location)
                }
            }
        }
    }

    @SuppressLint("MissingPermission")
    private fun startLocationUpdates() {
        val locationRequest = LocationRequest.Builder(
            Priority.PRIORITY_HIGH_ACCURACY,
            LOCATION_INTERVAL_MS
        ).apply {
            setMinUpdateDistanceMeters(10f)  // 10m 이상 이동 시에만 업데이트
            setWaitForAccurateLocation(false)
        }.build()

        fusedLocationClient.requestLocationUpdates(
            locationRequest,
            locationCallback,
            Looper.getMainLooper()
        )
    }

    private fun onLocationReceived(location: Location) {
        val message = "위도: %.4f, 경도: %.4f".format(location.latitude, location.longitude)
        notificationManager.notify(NOTIFICATION_ID, buildNotification(message))
        // 여기서 DB 저장, 서버 전송 등의 작업 수행
    }

    private fun createNotificationChannel() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            val channel = NotificationChannel(
                CHANNEL_ID,
                "위치 추적",
                NotificationManager.IMPORTANCE_LOW
            ).apply {
                description = "실시간 위치 추적"
                setShowBadge(false)
            }
            notificationManager.createNotificationChannel(channel)
        }
    }

    private fun buildNotification(contentText: String): Notification {
        return NotificationCompat.Builder(this, CHANNEL_ID)
            .setContentTitle("위치 추적 중")
            .setContentText(contentText)
            .setSmallIcon(android.R.drawable.ic_menu_mylocation)
            .setOngoing(true)
            .setSilent(true)
            .build()
    }

    override fun onBind(intent: Intent?): IBinder? = null

    override fun onDestroy() {
        super.onDestroy()
        fusedLocationClient.removeLocationUpdates(locationCallback)
    }
}
```

### Manifest에 location 타입 선언

```xml
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />

<service
    android:name=".LocationTrackingService"
    android:foregroundServiceType="location"
    android:exported="false" />
```

---

## Foreground Service 생명주기 완전 이해

```
startForegroundService() 호출
        ↓
  onCreate() 실행
        ↓
  onStartCommand() 실행  ←── 여기서 startForeground() 반드시 호출 (5초 이내)
        ↓
  [서비스 실행 중]
        ↓
  stopSelf() 또는 stopService() 호출
        ↓
  onDestroy() 실행
```

### 핵심 규칙: 5초 이내 startForeground() 호출

`startForegroundService()`를 호출한 후 5초 이내에 `startForeground()`를 호출하지 않으면 `ANR`이 발생합니다. `onCreate()`가 아닌 `onStartCommand()`에서 즉시 호출하는 것이 안전합니다.

```kotlin
// 잘못된 방법: 비동기 작업 후 호출 시 ANR 위험
override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
    // ❌ 위험: 초기화에 시간이 걸리면 5초 초과 가능
    initHeavyResource {
        startForeground(ID, buildNotification())
    }
    return START_NOT_STICKY
}

// 올바른 방법: 즉시 startForeground() 호출
override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
    // ✅ 즉시 Foreground로 전환
    ServiceCompat.startForeground(this, ID, buildNotification(), serviceType)

    // 무거운 작업은 코루틴으로
    serviceScope.launch {
        initHeavyResource()
    }
    return START_NOT_STICKY
}
```

### START_STICKY vs START_NOT_STICKY

| 반환값 | 시스템 종료 후 재시작 | intent 전달 |
|--------|-------------------|-----------  |
| `START_STICKY` | 재시작 O, intent=null | 적합: 위치 추적, 음악 재생 |
| `START_NOT_STICKY` | 재시작 X | 적합: 일회성 다운로드 |
| `START_REDELIVER_INTENT` | 재시작 O, 마지막 intent 재전달 | 적합: 중요 데이터 처리 |

---

## 주의사항 및 팁

### 1. Android 13+ 알림 권한 런타임 요청 필수

```kotlin
// Activity에서 알림 권한 요청
private val requestPermissionLauncher = registerForActivityResult(
    ActivityResultContracts.RequestPermission()
) { isGranted ->
    if (isGranted) {
        DownloadForegroundService.startDownload(this, url, fileName)
    } else {
        showPermissionRationale()
    }
}

private fun startServiceSafely() {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
        when {
            checkSelfPermission(android.Manifest.permission.POST_NOTIFICATIONS)
                == android.content.pm.PackageManager.PERMISSION_GRANTED -> {
                DownloadForegroundService.startDownload(this, url, fileName)
            }
            shouldShowRequestPermissionRationale(android.Manifest.permission.POST_NOTIFICATIONS) -> {
                showPermissionRationale()
            }
            else -> {
                requestPermissionLauncher.launch(android.Manifest.permission.POST_NOTIFICATIONS)
            }
        }
    } else {
        DownloadForegroundService.startDownload(this, url, fileName)
    }
}
```

### 2. 알림 채널 중요도(Importance) 선택 기준

| 중요도 | 상수 | 동작 | 사용 예 |
|--------|------|------|---------|
| 긴급 | `IMPORTANCE_HIGH` | 소리+팝업 | 전화, 중요 알림 |
| 높음 | `IMPORTANCE_DEFAULT` | 소리 있음 | 일반 알림 |
| 중간 | `IMPORTANCE_LOW` | 소리 없음 | 진행 중 작업 ✅ 권장 |
| 낮음 | `IMPORTANCE_MIN` | 상태 표시줄에만 | 백그라운드 동기화 |

Foreground Service의 알림은 보통 `IMPORTANCE_LOW`가 적합합니다. 사용자를 방해하지 않으면서도 진행 상황을 표시할 수 있습니다.

### 3. 메모리 누수 방지: 코루틴 Job 관리

```kotlin
class MyForegroundService : Service() {
    // SupervisorJob 사용: 자식 코루틴 하나가 실패해도 나머지 계속 실행
    private val serviceJob = SupervisorJob()
    private val serviceScope = CoroutineScope(Dispatchers.IO + serviceJob)

    override fun onDestroy() {
        super.onDestroy()
        serviceJob.cancel()  // 반드시 취소하여 누수 방지
    }
}
```

### 4. 서비스 타입 조합 (여러 타입 동시 선언)

카메라로 녹화하며 위치도 추적하는 경우처럼 여러 타입이 필요할 수 있습니다.

```xml
<!-- Manifest: 비트OR로 여러 타입 선언 -->
<service
    android:name=".RecordingService"
    android:foregroundServiceType="camera|microphone|location"
    android:exported="false" />
```

```kotlin
// startForeground() 시에도 OR 연산으로 타입 명시
ServiceCompat.startForeground(
    this,
    NOTIFICATION_ID,
    buildNotification(),
    ServiceInfo.FOREGROUND_SERVICE_TYPE_CAMERA or
    ServiceInfo.FOREGROUND_SERVICE_TYPE_MICROPHONE or
    ServiceInfo.FOREGROUND_SERVICE_TYPE_LOCATION
)
```

### 5. shortService 타입: 권한 없이 단기 작업

`shortService`는 약 3분의 타임아웃이 있지만, 별도 FGS 권한 없이 사용할 수 있습니다. 단기 중요 작업에 유용합니다.

```xml
<!-- POST_NOTIFICATIONS 외에 추가 FGS 권한 불필요 -->
<service
    android:name=".QuickTaskService"
    android:foregroundServiceType="shortService"
    android:exported="false" />
```

### 6. WorkManager와 Foreground Service 조합

장기 실행 WorkManager 작업은 내부적으로 Foreground Service를 사용합니다.

```kotlin
class LongRunningWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {

    override suspend fun getForegroundInfo(): ForegroundInfo {
        return ForegroundInfo(
            NOTIFICATION_ID,
            buildNotification(),
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.UPSIDE_DOWN_CAKE) {
                ServiceInfo.FOREGROUND_SERVICE_TYPE_DATA_SYNC
            } else {
                0
            }
        )
    }

    override suspend fun doWork(): Result {
        setForeground(getForegroundInfo())
        // 장기 작업 수행
        performLongRunningTask()
        return Result.success()
    }
}
```

---

## 정리

Android Foreground Service는 강력하지만 올바르게 사용해야 하는 컴포넌트입니다.

1. **Android 14+**: `foregroundServiceType`을 Manifest와 코드 양쪽에 반드시 선언
2. **5초 규칙**: `onStartCommand()`에서 즉시 `ServiceCompat.startForeground()` 호출
3. **알림 채널**: `IMPORTANCE_LOW`로 사용자 방해 최소화
4. **코루틴 Job**: `onDestroy()`에서 반드시 취소
5. **적절한 타입 선택**: 필요한 최소한의 타입만 사용 (Privacy 정책 준수)
6. **대안 검토**: 단기 작업은 WorkManager `shortService`, 주기적 작업은 WorkManager PeriodicWork 사용

올바른 Foreground Service 구현은 사용자 경험과 배터리 효율 모두를 잡는 핵심입니다.

---

## 참고 자료

- [Foreground service types - Android Developers](https://developer.android.com/develop/background-work/services/fgs/service-types)
- [Foreground service types are required (Android 14) - Android Developers](https://developer.android.com/about/versions/14/changes/fgs-types-required)
- [Launch a foreground service - Android Developers](https://developer.android.com/develop/background-work/services/fgs/launch)
