---
layout: post
title: "Android 위치 서비스 심화: FusedLocationProviderClient와 Geofencing API로 정밀한 위치 기반 앱 만들기"
date: 2026-10-14
categories: [android]
tags: [android, location, fusedlocation, geofencing, kotlin, flow, gms]
---

## 개요

스마트폰이 보편화된 이후, 위치 기반 서비스(Location-Based Services, LBS)는 현대 앱에서 빠질 수 없는 핵심 기능이 되었다. 음식 배달 앱이 주문자의 위치를 추적하고, 피트니스 앱이 이동 경로를 기록하며, 알림 앱이 특정 장소에 도착했을 때 자동으로 팝업을 띄우는 것 — 이 모두가 위치 서비스의 응용이다.

Android는 초기부터 `LocationManager`를 통해 위치 서비스를 제공했지만, GPS·네트워크·센서 각각을 직접 다루어야 해서 구현이 복잡하고 배터리 소모도 컸다. Google Play Services가 등장하면서 이를 하나의 통합 인터페이스인 **FusedLocationProviderClient(FLP)** 로 추상화했다. FLP는 GPS, Wi-Fi, 셀룰러 등 여러 신호를 지능적으로 합성(fuse)해 높은 정확도와 낮은 배터리 소모를 동시에 달성한다.

이 글에서는 FLP를 Kotlin Flow로 래핑하는 방법과, 특정 지리적 영역 진입/이탈을 감지하는 **Geofencing API** 심화 구현을 다룬다.

---

## 왜 FusedLocationProviderClient인가?

### 기존 LocationManager의 한계

`android.location.LocationManager`를 직접 사용하면 개발자가 다음 세 가지를 모두 직접 관리해야 한다.

- **GPS_PROVIDER**: 높은 정확도, 그러나 실외에서만 동작하며 배터리를 많이 소모
- **NETWORK_PROVIDER**: 빠른 응답, 낮은 정확도 (수십~수백 미터 오차)
- **PASSIVE_PROVIDER**: 다른 앱의 위치 요청에 편승 — 배터리 절약, 그러나 빈도 불규칙

각 프로바이더마다 `requestLocationUpdates()`를 따로 호출하고, 어느 값을 신뢰할지 직접 판단해야 한다. 여기에 Activity 생명주기에 맞춰 시작/중지를 관리하면 코드가 순식간에 비대해진다.

### FLP가 해결하는 것

`FusedLocationProviderClient`는 내부적으로 여러 신호를 결합해 **가장 적절한 위치**를 단일 스트림으로 제공한다. 개발자는 원하는 정확도와 업데이트 간격만 `LocationRequest`로 선언하면 된다.

```
GPS + Wi-Fi + 셀룰러 신호
        ↓
  [Fused Location Provider]  ← Google Play Services 내부 처리
        ↓
  FusedLocationProviderClient API
        ↓
     앱 코드
```

장점을 요약하면 다음과 같다.

| 항목 | LocationManager | FusedLocationProviderClient |
|------|----------------|-----------------------------|
| 정확도 | 프로바이더별로 다름 | 자동으로 최적 합성 |
| 배터리 | 개발자가 최적화 | 플랫폼 수준 최적화 |
| 코드 복잡도 | 높음 | 낮음 |
| Google Play 의존 | 없음 | 있음 |

---

## 실제 구현 예제 1: FLP + Kotlin callbackFlow

### 의존성 추가

```kotlin
// build.gradle.kts (app)
dependencies {
    implementation("com.google.android.gms:play-services-location:21.3.0")
}
```

### 권한 선언

```xml
<!-- AndroidManifest.xml -->
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
<!-- Android 10 이상에서 백그라운드 위치 수신 시 -->
<uses-permission android:name="android.permission.ACCESS_BACKGROUND_LOCATION" />
```

### LocationRepository — callbackFlow로 위치 스트림 만들기

```kotlin
import android.annotation.SuppressLint
import android.content.Context
import android.location.Location
import android.os.Looper
import com.google.android.gms.location.*
import kotlinx.coroutines.channels.awaitClose
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.callbackFlow
import kotlinx.coroutines.flow.catch
import kotlinx.coroutines.launch

class LocationRepository(context: Context) {

    private val fusedClient: FusedLocationProviderClient =
        LocationServices.getFusedLocationProviderClient(context)

    /**
     * 위치 업데이트를 Flow<Location>으로 노출한다.
     * @param intervalMs    업데이트 목표 간격 (밀리초)
     * @param minUpdateMs   최소 업데이트 간격 (밀리초)
     * @param priority      PRIORITY_HIGH_ACCURACY | PRIORITY_BALANCED_POWER_ACCURACY
     *                       | PRIORITY_LOW_POWER | PRIORITY_PASSIVE
     */
    @SuppressLint("MissingPermission")
    fun locationFlow(
        intervalMs: Long = 5_000L,
        minUpdateMs: Long = 2_000L,
        priority: Int = Priority.PRIORITY_HIGH_ACCURACY,
    ): Flow<Location> = callbackFlow {

        val request = LocationRequest.Builder(priority, intervalMs)
            .setMinUpdateIntervalMillis(minUpdateMs)
            .setWaitForAccurateLocation(false)
            .build()

        val callback = object : LocationCallback() {
            override fun onLocationResult(result: LocationResult) {
                result.lastLocation?.let { location ->
                    // trySend는 버퍼가 가득 찬 경우 이벤트를 드롭한다.
                    // 연속 위치 스트림에서는 최신 값이 중요하므로 적절한 선택이다.
                    trySend(location)
                }
            }

            override fun onLocationAvailability(availability: LocationAvailability) {
                if (!availability.isLocationAvailable) {
                    // GPS가 꺼졌거나 실내 환경 등 — 필요 시 상위에서 처리
                }
            }
        }

        fusedClient.requestLocationUpdates(
            request,
            callback,
            Looper.getMainLooper()
        )

        // 채널이 닫히면 (Flow 수집 취소 시) 반드시 콜백을 제거한다.
        awaitClose {
            fusedClient.removeLocationUpdates(callback)
        }
    }.catch { e ->
        // SecurityException: 권한 없을 때 처리
        e.printStackTrace()
    }

    /** 마지막으로 알려진 위치를 한 번만 가져온다. */
    @SuppressLint("MissingPermission")
    suspend fun getLastKnownLocation(): Location? {
        return try {
            fusedClient.lastLocation.await()   // kotlinx-coroutines-play-services
        } catch (e: Exception) {
            null
        }
    }
}
```

### ViewModel에서 수집

```kotlin
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.google.android.gms.location.Priority
import kotlinx.coroutines.flow.*

class LocationViewModel(
    private val repo: LocationRepository
) : ViewModel() {

    // 화면 회전 등 재구독 시에도 마지막 값을 즉시 제공 (stateIn)
    val locationState: StateFlow<LocationUiState> = repo
        .locationFlow(
            intervalMs = 3_000L,
            priority = Priority.PRIORITY_BALANCED_POWER_ACCURACY
        )
        .map { loc ->
            LocationUiState.Success(
                lat = loc.latitude,
                lng = loc.longitude,
                accuracy = loc.accuracy,
                speed = loc.speed,
            )
        }
        .catch { emit(LocationUiState.Error(it.message ?: "Unknown error")) }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            initialValue = LocationUiState.Loading
        )
}

sealed interface LocationUiState {
    data object Loading : LocationUiState
    data class Success(
        val lat: Double,
        val lng: Double,
        val accuracy: Float,
        val speed: Float,
    ) : LocationUiState
    data class Error(val message: String) : LocationUiState
}
```

### Compose UI에서 수집

```kotlin
@Composable
fun LocationScreen(viewModel: LocationViewModel = viewModel()) {
    val state by viewModel.locationState.collectAsStateWithLifecycle()

    when (state) {
        is LocationUiState.Loading -> CircularProgressIndicator()
        is LocationUiState.Error   -> Text("오류: ${(state as LocationUiState.Error).message}")
        is LocationUiState.Success -> {
            val s = state as LocationUiState.Success
            Column {
                Text("위도: ${s.lat}")
                Text("경도: ${s.lng}")
                Text("정확도: ${s.accuracy}m")
                Text("속도: ${"%.1f".format(s.speed * 3.6f)} km/h")
            }
        }
    }
}
```

`collectAsStateWithLifecycle()`을 사용하면 앱이 포그라운드일 때만 Flow를 구독하여 불필요한 배터리 소모를 막을 수 있다.

---

## 실제 구현 예제 2: Geofencing API 심화 구현

Geofencing은 가상의 지리적 경계(원형 영역)를 설정하고, 기기가 그 영역에 **진입(ENTER)**, **이탈(EXIT)**, 또는 일정 시간 **머물렀을 때(DWELL)** 이벤트를 발생시키는 기능이다. 편의점 쿠폰, 근처 알림, 배달 도착 감지 등에 폭넓게 쓰인다.

### 아키텍처 다이어그램

```
앱
 ├─ GeofenceManager       ← 지오펜스 등록/해제
 ├─ GeofenceBroadcastReceiver  ← 전환 이벤트 수신
 └─ GeofenceRepository    ← 이벤트를 Flow로 브로드캐스트
```

### GeofenceManager

```kotlin
import android.annotation.SuppressLint
import android.app.PendingIntent
import android.content.Context
import android.content.Intent
import com.google.android.gms.location.*
import kotlinx.coroutines.tasks.await

data class GeofenceData(
    val id: String,
    val lat: Double,
    val lng: Double,
    val radiusMeters: Float = 150f,
    val expirationMs: Long = Geofence.NEVER_EXPIRE,
    val transitionTypes: Int = Geofence.GEOFENCE_TRANSITION_ENTER or
            Geofence.GEOFENCE_TRANSITION_EXIT or
            Geofence.GEOFENCE_TRANSITION_DWELL,
    val dwellDelayMs: Int = 30_000,
)

class GeofenceManager(private val context: Context) {

    private val client: GeofencingClient = LocationServices.getGeofencingClient(context)

    private val pendingIntent: PendingIntent by lazy {
        val intent = Intent(context, GeofenceBroadcastReceiver::class.java)
        PendingIntent.getBroadcast(
            context,
            0,
            intent,
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_MUTABLE
        )
    }

    @SuppressLint("MissingPermission")
    suspend fun addGeofences(geofences: List<GeofenceData>): Result<Unit> = runCatching {
        val gmsGeofences = geofences.map { data ->
            Geofence.Builder()
                .setRequestId(data.id)
                .setCircularRegion(data.lat, data.lng, data.radiusMeters)
                .setExpirationDuration(data.expirationMs)
                .setTransitionTypes(data.transitionTypes)
                // DWELL 이벤트: 영역 안에서 dwellDelayMs 이상 머물러야 트리거
                .setLoiteringDelay(data.dwellDelayMs)
                .build()
        }

        val request = GeofencingRequest.Builder()
            // 앱이 이미 지오펜스 안에 있을 때도 즉시 ENTER 이벤트 발생
            .setInitialTrigger(GeofencingRequest.INITIAL_TRIGGER_ENTER)
            .addGeofences(gmsGeofences)
            .build()

        client.addGeofences(request, pendingIntent).await()
    }

    suspend fun removeGeofences(ids: List<String>): Result<Unit> = runCatching {
        client.removeGeofences(ids).await()
    }

    suspend fun removeAllGeofences(): Result<Unit> = runCatching {
        client.removeGeofences(pendingIntent).await()
    }
}
```

### GeofenceBroadcastReceiver

```kotlin
import android.content.BroadcastReceiver
import android.content.Context
import android.content.Intent
import com.google.android.gms.location.Geofence
import com.google.android.gms.location.GeofenceStatusCodes
import com.google.android.gms.location.GeofencingEvent

class GeofenceBroadcastReceiver : BroadcastReceiver() {

    override fun onReceive(context: Context, intent: Intent) {
        val event = GeofencingEvent.fromIntent(intent) ?: return

        if (event.hasError()) {
            val errorMessage = GeofenceStatusCodes.getStatusCodeString(event.errorCode)
            GeofenceEventBus.publish(GeofenceEvent.Error(errorMessage))
            return
        }

        val transition = when (event.geofenceTransition) {
            Geofence.GEOFENCE_TRANSITION_ENTER -> GeofenceTransition.ENTER
            Geofence.GEOFENCE_TRANSITION_EXIT  -> GeofenceTransition.EXIT
            Geofence.GEOFENCE_TRANSITION_DWELL -> GeofenceTransition.DWELL
            else -> return
        }

        val triggeringIds = event.triggeringGeofences
            ?.map { it.requestId }
            ?: return

        GeofenceEventBus.publish(
            GeofenceEvent.Transition(
                geofenceIds = triggeringIds,
                transition = transition,
                triggerLocation = event.triggeringLocation,
            )
        )
    }
}

enum class GeofenceTransition { ENTER, EXIT, DWELL }

sealed interface GeofenceEvent {
    data class Transition(
        val geofenceIds: List<String>,
        val transition: GeofenceTransition,
        val triggerLocation: android.location.Location?,
    ) : GeofenceEvent

    data class Error(val message: String) : GeofenceEvent
}
```

### GeofenceEventBus — SharedFlow 기반 이벤트 버스

```kotlin
import kotlinx.coroutines.flow.MutableSharedFlow
import kotlinx.coroutines.flow.SharedFlow
import kotlinx.coroutines.flow.asSharedFlow

object GeofenceEventBus {
    private val _events = MutableSharedFlow<GeofenceEvent>(
        extraBufferCapacity = 16
    )
    val events: SharedFlow<GeofenceEvent> = _events.asSharedFlow()

    fun publish(event: GeofenceEvent) {
        _events.tryEmit(event)
    }
}
```

### ViewModel에서 지오펜스 이벤트 수집

```kotlin
class GeofenceViewModel(
    private val manager: GeofenceManager
) : ViewModel() {

    val geofenceEvents: SharedFlow<GeofenceEvent> = GeofenceEventBus.events

    init {
        viewModelScope.launch {
            // 앱 실행 시 예시 지오펜스 3개 등록
            manager.addGeofences(
                listOf(
                    GeofenceData(id = "office",   lat = 37.5665, lng = 126.9780, radiusMeters = 200f),
                    GeofenceData(id = "home",     lat = 37.4979, lng = 127.0276, radiusMeters = 150f),
                    GeofenceData(id = "gym",      lat = 37.5172, lng = 127.0473, radiusMeters = 100f),
                )
            )
        }
    }
}
```

---

## 주의사항과 팁

### 1. 지오펜스 정확도와 반경 설정

Wi-Fi가 꺼진 상태에서 GPS만 사용하면 위치 정확도가 크게 떨어진다. 공식 문서는 최소 반경을 **100~150m**로 권장한다. 반경이 너무 작으면 GPS 오차로 인해 진입/이탈 이벤트가 반복해서 발생하는 **핑퐁(ping-pong) 현상**이 생긴다. 이를 완화하려면 `GEOFENCE_TRANSITION_DWELL`을 함께 사용해 일정 시간 머문 경우에만 알림을 발송하라.

### 2. 앱당 지오펜스 개수 제한

Google Play Services는 **앱당 사용자 1명 기준 최대 100개**의 지오펜스를 허용한다. 이 한도를 초과하면 `GEOFENCE_TOO_MANY_GEOFENCES` 오류가 반환된다. 동적으로 지오펜스를 교체하는 전략(예: 현재 위치와 가장 가까운 N개만 등록)이 필요할 수 있다.

### 3. 기기 재시작 후 재등록

지오펜스는 기기를 재시작하면 자동으로 제거된다. `BOOT_COMPLETED` 브로드캐스트를 수신해 재등록 로직을 실행해야 한다.

```kotlin
// BootReceiver.kt
class BootReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        if (intent.action == Intent.ACTION_BOOT_COMPLETED) {
            // WorkManager를 통해 지오펜스 재등록 작업 실행
            GeofenceRestoreWorker.enqueue(context)
        }
    }
}
```

### 4. Android 10 이상 백그라운드 위치 권한

Android 10(API 29)부터 백그라운드에서 위치를 수신하려면 `ACCESS_BACKGROUND_LOCATION` 권한이 별도로 필요하다. 이 권한을 요청할 때는 먼저 포그라운드 위치 권한을 받은 뒤 **별도의 단계로** 요청해야 한다. Google Play 정책상 백그라운드 위치를 수집하는 경우 심사 과정에서 사용 목적을 명확히 밝혀야 한다.

```kotlin
// 포그라운드 권한 먼저 요청
val fgLauncher = registerForActivityResult(
    ActivityResultContracts.RequestMultiplePermissions()
) { grants ->
    if (grants.all { it.value }) {
        // 다음 단계로 백그라운드 권한 요청
        bgLauncher.launch(Manifest.permission.ACCESS_BACKGROUND_LOCATION)
    }
}

val bgLauncher = registerForActivityResult(
    ActivityResultContracts.RequestPermission()
) { granted ->
    if (granted) startGeofencing()
}
```

### 5. 배터리 최적화

FLP의 `Priority.PRIORITY_BALANCED_POWER_ACCURACY`는 약 100m 정확도를 제공하면서 GPS를 최소화한다. 정밀한 실시간 추적이 필요 없다면 이 우선순위를 사용하라. 또한 `LocationRequest.Builder.setMaxUpdateDelayMillis()`를 설정하면 업데이트를 일괄 처리해 Doze 모드에서의 웨이크업 횟수를 줄일 수 있다.

### 6. 지오펜스 이벤트 지연

Android 8.0(Oreo) 이상에서 백그라운드 앱은 지오펜스 이벤트를 **수 분 단위**로 지연해 수신할 수 있다. 이는 배터리 절약을 위한 플랫폼 수준의 결정으로, 앱에서 제어할 수 없다. 실시간 성이 중요한 경우 포그라운드 서비스를 함께 운용하는 것을 검토하라.

---

## 마무리

FusedLocationProviderClient와 Geofencing API는 Android 위치 서비스의 두 핵심 축이다. FLP는 callbackFlow와 stateIn을 결합해 선언형으로 위치 스트림을 노출할 수 있고, Geofencing은 SharedFlow 기반 이벤트 버스 패턴으로 앱의 여러 레이어에 깔끔하게 전달할 수 있다.

위치 서비스를 구현할 때 가장 중요한 원칙은 **최소 권한 요청**, **생명주기 인식 수집**, **배터리 소모 최소화** 세 가지다. 이 원칙을 지키며 사용자에게 꼭 필요한 위치 데이터만 수집하는 앱을 만들자.

---

## 참고 자료

- [Create and monitor geofences — Android Developers](https://developer.android.com/training/location/geofencing)
- [Request location updates — Android Developers](https://developer.android.com/training/location/request-updates)
- [FusedLocationProviderClient API Reference](https://developer.android.com/android/reference/com/google/android/gms/location/FusedLocationProviderClient)
- [Build location-aware apps — Android Developers](https://developer.android.com/training/location)
