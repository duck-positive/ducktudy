---
layout: post
title: "Android Wear OS 심화: Tiles·Complications·Health Services API로 스마트워치 앱 완전 정복"
date: 2026-09-12
categories: [android, wearos]
tags: [wear-os, tiles, complications, health-services, compose-for-wear-os, kotlin, jetpack]
---

웨어러블 기기 시장이 빠르게 성장하면서, Android 개발자에게 Wear OS 앱 개발은 이제 선택이 아닌 필수가 되어가고 있다. 그러나 Wear OS는 일반 Android 앱 개발과는 전혀 다른 제약과 패러다임을 요구한다. 배터리 용량이 극히 제한되고, 화면이 작으며, 상호작용 시간이 초 단위로 짧아야 한다. 이 아티클에서는 Wear OS의 핵심 세 가지 축인 **Tiles**, **Complications**, **Health Services API**를 깊이 있게 살펴보고, 실전에서 바로 적용할 수 있는 Kotlin 코드 예제와 함께 완전히 정복한다.

---

## 개념 설명: Wear OS의 세 가지 핵심 축

### Tiles: 빠른 정보 접근의 시작점

Tile은 워치 페이스 옆에 위치한 카드 형태의 UI로, 사용자가 손목을 들어 한 번 스와이프하면 바로 핵심 정보를 확인할 수 있다. 앱을 실행하지 않고도 운동 현황, 날씨, 다음 약속, 현재 심박수 등을 즉시 볼 수 있는 것이 Tile의 핵심 가치다.

Tile은 `TileService`를 통해 구현되며, 레이아웃은 `protolayout` 라이브러리를 사용해 **선언적으로** 정의된다. 중요한 점은 Tile의 렌더링이 앱 프로세스 밖에서 이루어질 수 있다는 것이다. 따라서 레이아웃 정의는 순수 데이터 구조여야 하며, View나 Compose를 직접 사용할 수 없다(단, Compose Glance처럼 `TileService` 내부에서 Compose로 레이아웃을 생성하는 `ProtoLayout` 빌더는 사용 가능하다).

### Complications: 워치 페이스에 생명을 불어넣는 데이터

Complication은 워치 페이스(시계 화면)에 표시되는 부가 정보다. 예를 들어, 아날로그 시계 화면의 6시 방향에 걸음 수를 보여주는 작은 숫자나 아이콘이 전형적인 Complication이다.

앱이 Complication 데이터 제공자가 되려면 `ComplicationDataSourceService`를 구현해야 한다. 데이터 제공자는 데이터를 주기적으로 또는 변경 시에 워치 페이스에 전달하고, 워치 페이스가 렌더링 방식을 결정한다. 제공자와 렌더러가 분리된 이 구조는 책임이 명확하게 나뉘는 장점이 있다.

### Health Services API: 배터리를 아끼면서 정확한 건강 데이터 수집

Health Services는 Wear OS 3부터 내장된 플랫폼 서비스로, 디바이스의 각종 센서(심박수 센서, 가속도계, GPS 등)를 앱이 직접 다루는 대신 Health Services가 중간에서 관리해준다.

이를 통해:
- **배터리 효율**: 여러 앱이 같은 센서를 중복 사용하지 않고 Health Services가 한 번만 센서를 구동해 공유
- **정확도**: 플랫폼이 직접 보정하고 계산한 데이터 제공
- **일관성**: 디바이스 제조사가 달라도 동일한 API로 데이터 수집 가능

Health Services의 세 가지 클라이언트가 있다:
- `PassiveMonitoringClient`: 백그라운드에서 주기적으로 데이터를 수집 (걸음 수, 칼로리 등)
- `MeasureClient`: 포그라운드에서 빠른 실시간 측정 (심박수 실시간 표시 등)
- `ExerciseClient`: 운동 세션 관리 (러닝·사이클링 운동 추적)

---

## 왜 필요한가: Wear OS 개발의 독특한 제약

일반 Android 앱 개발 경험이 있더라도 Wear OS에서는 새로운 사고방식이 필요하다.

**1. 배터리 극한 관리**: 스마트워치 배터리는 수백 mAh 수준으로 스마트폰의 1/10에 불과하다. 앱이 조금만 방전을 유발해도 사용자 경험이 크게 저하된다. Health Services 없이 앱이 직접 심박수 센서를 상시 구동하면 하루도 버티기 어렵다.

**2. 짧은 인터랙션 시간**: 사용자가 손목을 들어 정보를 확인하는 시간은 평균 2~3초다. 앱을 실행하고 로딩을 기다릴 여유가 없다. Tile은 이 문제를 해결하기 위해 존재한다.

**3. 작은 화면과 둥근 디스플레이**: 대부분의 Wear OS 기기는 원형 화면이다. 일반 Android의 LinearLayout이나 RecyclerView 패턴을 그대로 쓰면 가장자리가 잘리고 콘텐츠가 비좁아진다. Wear Compose의 `ScalingLazyColumn`이나 `Scaffold` 같은 Wear 전용 컴포넌트를 사용해야 한다.

**4. 항상 켜진 화면(AOD) 지원**: Wear OS는 Ambient Mode(저전력 항상 켜짐 모드)를 지원한다. 이 모드에서는 화면 밝기와 색상이 극도로 제한된다. 앱이 Ambient Mode를 올바르게 처리하지 않으면 배터리가 빠르게 닳는다.

이러한 이유들로 Wear OS만을 위한 전용 API와 아키텍처 패턴이 탄생했다.

---

## 실제 구현 예제

### 예제 1: Wear OS Tile 구현 — 심박수 타일

다음은 현재 심박수를 보여주는 간단한 Tile을 구현한 예제다. Jetpack의 `wear-tiles` 라이브러리와 `wear-protolayout`을 사용한다.

**build.gradle.kts 의존성 추가:**

```kotlin
dependencies {
    implementation("androidx.wear.tiles:tiles:1.4.1")
    implementation("androidx.wear.tiles:tiles-material:1.4.1")
    implementation("androidx.wear.protolayout:protolayout:1.2.1")
    implementation("androidx.wear.protolayout:protolayout-material:1.2.1")
    implementation("androidx.health:health-services-client:1.1.0-rc01")
}
```

**HeartRateTileService.kt:**

```kotlin
import androidx.wear.protolayout.ColorBuilders.argb
import androidx.wear.protolayout.DimensionBuilders.dp
import androidx.wear.protolayout.DimensionBuilders.sp
import androidx.wear.protolayout.LayoutElementBuilders.*
import androidx.wear.protolayout.ResourceBuilders
import androidx.wear.protolayout.TimelineBuilders
import androidx.wear.tiles.RequestBuilders
import androidx.wear.tiles.TileBuilders
import androidx.wear.tiles.TileService
import com.google.common.util.concurrent.Futures
import com.google.common.util.concurrent.ListenableFuture

class HeartRateTileService : TileService() {

    // 캐시된 최신 심박수 값 (실제 앱에서는 DataStore 등을 통해 가져옴)
    private var latestHeartRate: Int = 0

    override fun onTileRequest(
        requestParams: RequestBuilders.TileRequest
    ): ListenableFuture<TileBuilders.Tile> {
        val tile = TileBuilders.Tile.Builder()
            .setResourcesVersion("1")
            .setTileTimeline(buildTimeline())
            // 타일을 30분마다 새로 고침
            .setFreshnessIntervalMillis(30 * 60 * 1000L)
            .build()
        return Futures.immediateFuture(tile)
    }

    override fun onResourcesRequest(
        requestParams: RequestBuilders.ResourcesRequest
    ): ListenableFuture<ResourceBuilders.Resources> {
        return Futures.immediateFuture(
            ResourceBuilders.Resources.Builder()
                .setVersion("1")
                .build()
        )
    }

    private fun buildTimeline(): TimelineBuilders.Timeline {
        val layout = buildLayout()
        val timelineEntry = TimelineBuilders.TimelineEntry.Builder()
            .setLayout(layout)
            .build()
        return TimelineBuilders.Timeline.Builder()
            .addTimelineEntry(timelineEntry)
            .build()
    }

    private fun buildLayout(): Layout {
        val heartRateText = if (latestHeartRate > 0) "$latestHeartRate BPM" else "-- BPM"

        val content = Column.Builder()
            .setWidth(expand())
            .setHeight(expand())
            .setHorizontalAlignment(HORIZONTAL_ALIGN_CENTER)
            .addContent(
                Text.Builder()
                    .setText("❤️ 심박수")
                    .setFontStyle(
                        FontStyle.Builder()
                            .setSize(sp(14f))
                            .setColor(argb(0xFFAAAAAA.toInt()))
                            .build()
                    )
                    .build()
            )
            .addContent(
                Text.Builder()
                    .setText(heartRateText)
                    .setFontStyle(
                        FontStyle.Builder()
                            .setSize(sp(36f))
                            .setColor(argb(0xFFFF4444.toInt()))
                            .setBold(true)
                            .build()
                    )
                    .build()
            )
            .addContent(
                Text.Builder()
                    .setText("탭하여 운동 시작")
                    .setFontStyle(
                        FontStyle.Builder()
                            .setSize(sp(12f))
                            .setColor(argb(0xFF888888.toInt()))
                            .build()
                    )
                    .build()
            )
            .build()

        return Layout.Builder()
            .setRoot(
                Box.Builder()
                    .setWidth(expand())
                    .setHeight(expand())
                    .addContent(content)
                    .build()
            )
            .build()
    }
}
```

**AndroidManifest.xml 등록:**

```xml
<service
    android:name=".HeartRateTileService"
    android:exported="true"
    android:label="심박수 타일"
    android:permission="com.google.android.wearable.permission.BIND_TILE_PROVIDER">
    <intent-filter>
        <action android:name="androidx.wear.tiles.action.BIND_TILE_PROVIDER" />
    </intent-filter>
    <meta-data
        android:name="androidx.wear.tiles.PREVIEW_ACTIVITY"
        android:value=".TilePreviewActivity" />
</service>
```

---

### 예제 2: Health Services API로 실시간 심박수 수집

다음은 Wear OS에서 `MeasureClient`를 사용해 포그라운드에서 실시간 심박수를 수집하는 예제다. Compose for Wear OS와 함께 사용한다.

**HeartRateViewModel.kt:**

```kotlin
import android.content.Context
import androidx.health.services.client.HealthServices
import androidx.health.services.client.MeasureCallback
import androidx.health.services.client.data.Availability
import androidx.health.services.client.data.DataPointContainer
import androidx.health.services.client.data.DataType
import androidx.health.services.client.data.DeltaDataType
import androidx.health.services.client.data.SampleDataPoint
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.guava.await
import kotlinx.coroutines.launch

data class HeartRateUiState(
    val heartRate: Double = 0.0,
    val availability: String = "초기화 중...",
    val isLoading: Boolean = true
)

class HeartRateViewModel(private val context: Context) : ViewModel() {

    private val healthServicesClient = HealthServices.getClient(context)
    private val measureClient = healthServicesClient.measureClient

    private val _uiState = MutableStateFlow(HeartRateUiState())
    val uiState: StateFlow<HeartRateUiState> = _uiState.asStateFlow()

    // MeasureCallback: Health Services가 심박수를 전달할 때 호출되는 콜백
    private val heartRateCallback = object : MeasureCallback {
        override fun onAvailabilityChanged(
            dataType: DeltaDataType<*, *>,
            availability: Availability
        ) {
            val statusText = when (availability) {
                Availability.AVAILABLE -> "측정 중"
                Availability.ACQUIRING -> "센서 준비 중"
                else -> "사용 불가"
            }
            _uiState.value = _uiState.value.copy(
                availability = statusText,
                isLoading = availability != Availability.AVAILABLE
            )
        }

        override fun onDataReceived(data: DataPointContainer) {
            // HEART_RATE_BPM 타입의 샘플 데이터 포인트 추출
            val heartRatePoints = data.getData(DataType.HEART_RATE_BPM)
            val latestBpm = heartRatePoints
                .filterIsInstance<SampleDataPoint<Double>>()
                .maxByOrNull { it.timeDurationFromBoot }
                ?.value ?: return

            _uiState.value = _uiState.value.copy(
                heartRate = latestBpm,
                isLoading = false
            )
        }
    }

    fun startHeartRateMonitoring() {
        viewModelScope.launch {
            // 먼저 심박수 측정이 지원되는지 확인
            val capabilities = measureClient.getCapabilitiesAsync().await()
            if (DataType.HEART_RATE_BPM !in capabilities.supportedDataTypesMeasure) {
                _uiState.value = _uiState.value.copy(
                    availability = "이 기기는 심박수를 지원하지 않습니다",
                    isLoading = false
                )
                return@launch
            }
            // 콜백 등록 — 등록 즉시 심박수 데이터 수신 시작
            measureClient.registerMeasureCallback(
                DataType.HEART_RATE_BPM,
                heartRateCallback
            )
        }
    }

    fun stopHeartRateMonitoring() {
        viewModelScope.launch {
            measureClient.unregisterMeasureCallbackAsync(
                DataType.HEART_RATE_BPM,
                heartRateCallback
            ).await()
        }
    }

    override fun onCleared() {
        super.onCleared()
        stopHeartRateMonitoring()
    }
}
```

**HeartRateScreen.kt (Compose for Wear OS):**

```kotlin
import androidx.compose.foundation.background
import androidx.compose.foundation.layout.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import androidx.wear.compose.material3.*
import androidx.lifecycle.compose.collectAsStateWithLifecycle

@Composable
fun HeartRateScreen(viewModel: HeartRateViewModel) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    LaunchedEffect(Unit) {
        viewModel.startHeartRateMonitoring()
    }

    DisposableEffect(Unit) {
        onDispose {
            viewModel.stopHeartRateMonitoring()
        }
    }

    Box(
        modifier = Modifier
            .fillMaxSize()
            .background(Color.Black),
        contentAlignment = Alignment.Center
    ) {
        Column(
            horizontalAlignment = Alignment.CenterHorizontally,
            verticalArrangement = Arrangement.spacedBy(8.dp)
        ) {
            Text(
                text = "❤️ 심박수",
                color = Color.Gray,
                fontSize = 14.sp
            )

            if (uiState.isLoading) {
                CircularProgressIndicator(
                    modifier = Modifier.size(48.dp),
                    indicatorColor = Color.Red
                )
            } else {
                Text(
                    text = "${uiState.heartRate.toInt()} BPM",
                    color = Color.Red,
                    fontSize = 40.sp,
                    fontWeight = FontWeight.Bold
                )
            }

            Text(
                text = uiState.availability,
                color = Color.LightGray,
                fontSize = 12.sp
            )
        }
    }
}
```

---

### 예제 3: Complication 데이터 제공자 구현

워치 페이스에 걸음 수를 제공하는 Complication 데이터 소스를 만든다.

```kotlin
import androidx.wear.watchface.complications.data.*
import androidx.wear.watchface.complications.datasource.ComplicationDataSourceService
import androidx.wear.watchface.complications.datasource.ComplicationRequest

class StepCountComplicationService : ComplicationDataSourceService() {

    override fun getPreviewData(type: ComplicationType): ComplicationData? {
        // Android Studio에서 미리보기용 더미 데이터 반환
        return when (type) {
            ComplicationType.SHORT_TEXT -> ShortTextComplicationData.Builder(
                text = PlainComplicationText.Builder("8,432").build(),
                contentDescription = PlainComplicationText.Builder("걸음 수").build()
            )
                .setTitle(PlainComplicationText.Builder("걸음").build())
                .build()
            ComplicationType.RANGED_VALUE -> RangedValueComplicationData.Builder(
                value = 8432f,
                min = 0f,
                max = 10000f,
                contentDescription = PlainComplicationText.Builder("오늘의 걸음 수").build()
            )
                .setText(PlainComplicationText.Builder("8,432").build())
                .build()
            else -> null
        }
    }

    override fun onComplicationRequest(
        request: ComplicationRequest,
        listener: ComplicationRequestListener
    ) {
        // 실제 앱에서는 여기서 Health Services PassiveMonitoringClient로
        // 저장된 걸음 수 데이터를 DataStore에서 읽어온다
        val currentSteps = getCurrentStepsFromDataStore()
        val goalSteps = 10000

        val data = when (request.complicationType) {
            ComplicationType.SHORT_TEXT -> ShortTextComplicationData.Builder(
                text = PlainComplicationText.Builder(
                    formatStepCount(currentSteps)
                ).build(),
                contentDescription = PlainComplicationText.Builder("오늘 걸음 수").build()
            )
                .setTitle(PlainComplicationText.Builder("걸음").build())
                .build()

            ComplicationType.RANGED_VALUE -> RangedValueComplicationData.Builder(
                value = currentSteps.toFloat(),
                min = 0f,
                max = goalSteps.toFloat(),
                contentDescription = PlainComplicationText.Builder("걸음 수 진행률").build()
            )
                .setText(PlainComplicationText.Builder(
                    formatStepCount(currentSteps)
                ).build())
                .setTitle(PlainComplicationText.Builder("걸음").build())
                .build()

            else -> return
        }

        listener.onComplicationData(data)
    }

    private fun getCurrentStepsFromDataStore(): Int {
        // 실제 구현에서는 DataStore 또는 Room에서 값을 읽어옴
        return 8432
    }

    private fun formatStepCount(steps: Int): String {
        return if (steps >= 1000) "${steps / 1000},${(steps % 1000).toString().padStart(3, '0')}"
        else steps.toString()
    }
}
```

**AndroidManifest.xml 등록:**

```xml
<service
    android:name=".StepCountComplicationService"
    android:exported="true"
    android:icon="@drawable/ic_steps"
    android:label="걸음 수"
    android:permission="com.google.android.wearable.permission.BIND_COMPLICATION_PROVIDER">
    <intent-filter>
        <action android:name="androidx.wear.watchface.complications.datasource.BIND_COMPLICATION_DATASOURCE" />
    </intent-filter>
    <meta-data
        android:name="android.support.wearable.complications.SUPPORTED_TYPES"
        android:value="SHORT_TEXT,RANGED_VALUE" />
    <meta-data
        android:name="android.support.wearable.complications.UPDATE_PERIOD_SECONDS"
        android:value="300" />
</service>
```

---

## 주의사항 및 팁

### 1. 배터리 최적화가 최우선

Tile에서 네트워크 요청이나 긴 비동기 작업을 절대 직접 실행하지 말아야 한다. `onTileRequest()`는 가능한 빠르게 캐시된 데이터를 반환해야 한다. 실제 데이터 갱신은 `WorkManager`를 통해 주기적으로 백그라운드에서 처리하고, 결과를 `DataStore`에 저장한 뒤 Tile이 이 캐시에서 읽도록 설계해야 한다.

```kotlin
// 올바른 패턴: Tile은 캐시된 데이터를 읽기만 함
override fun onTileRequest(...): ListenableFuture<Tile> {
    val cachedData = runBlocking { dataStore.data.first() }
    // → 즉시 반환
    return Futures.immediateFuture(buildTile(cachedData))
}
```

### 2. Health Services의 권한 선언 필수

`BODY_SENSORS` 권한을 선언하지 않으면 심박수 데이터를 받을 수 없고, 이는 런타임 오류가 아닌 데이터 미수신으로 나타나 디버깅이 어렵다.

```xml
<uses-permission android:name="android.permission.BODY_SENSORS" />
<!-- Wear OS 4 이상에서는 백그라운드 센서 권한도 필요 -->
<uses-permission android:name="android.permission.BODY_SENSORS_BACKGROUND" />
```

### 3. Complication 업데이트 주기 설정에 주의

`UPDATE_PERIOD_SECONDS`를 너무 낮게 설정하면(예: 0초) Wear OS가 앱을 지속적으로 깨워 배터리를 빠르게 소모한다. 걸음 수처럼 자주 변하는 데이터는 300초(5분), 날씨처럼 자주 변하지 않는 데이터는 3600초(1시간) 이상을 권장한다. 더 즉각적인 업데이트가 필요하다면 `requestUpdate()` 또는 `requestUpdateAll()`을 사용해 변경 시점에 능동적으로 업데이트한다.

### 4. Wear OS 에뮬레이터와 실기기 테스트

에뮬레이터는 Health Services API의 센서 시뮬레이션을 지원하지만 실기기와 동작이 다를 수 있다. 특히 심박수 센서의 Availability 상태 전환은 실기기에서만 정확히 테스트된다. 가능하다면 실제 Wear OS 기기로 테스트하라.

### 5. Tile 프리뷰 활용

Android Studio Hedgehog 이상에서는 `@Preview` 애노테이션 대신 `TilePreviewHelper`를 사용해 Tile 레이아웃을 IDE에서 미리 볼 수 있다. 빌드 없이 레이아웃 변경을 즉시 확인할 수 있어 개발 속도가 크게 향상된다.

### 6. `androidx.wear.tiles:tiles-tooling` 활용

`tiles-tooling` 라이브러리는 Tile 렌더링 테스트를 위한 `TileRenderer`를 제공한다. 단위 테스트에서 Tile 레이아웃이 올바르게 생성되는지 검증할 수 있다.

### 7. Wear OS 버전별 호환성 확인

Health Services API는 Wear OS 3(API 30)부터 안정화되었다. Wear OS 2 기기를 지원해야 한다면 `CapabilitiesResponse`로 기능 지원 여부를 확인한 후 폴백 처리를 해야 한다.

---

## 정리

Wear OS 개발은 일반 Android 개발과 겉으로는 비슷해 보이지만, 배터리와 화면 크기라는 극단적 제약 속에서 완전히 다른 설계 원칙을 따라야 한다. **Tiles**는 앱을 열지 않고도 핵심 정보를 제공하는 창구이며, **Complications**는 워치 페이스와 생태계를 연결하는 다리, **Health Services**는 배터리를 아끼면서 정확한 건강 데이터를 안정적으로 공급하는 기반이다.

이 세 가지를 조합하면 손목 위의 작은 화면에서도 유용하고, 배터리 효율적이며, 사용자가 2초 이내에 원하는 정보를 얻을 수 있는 완성도 높은 Wear OS 앱을 만들 수 있다.

## 참고 자료
- [Wear OS Tiles 공식 문서 (Android Developers)](https://developer.android.com/training/wearables/tiles)
- [Health Services on Wear OS 공식 문서 (Android Developers)](https://developer.android.com/health-and-fitness/health-services)
- [Compose for Wear OS 공식 가이드 (Android Developers)](https://developer.android.com/training/wearables/compose)
- [Complications 공식 문서 (Android Developers)](https://developer.android.com/training/wearables/complications)
