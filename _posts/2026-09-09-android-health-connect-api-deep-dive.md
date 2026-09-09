---
layout: post
title: "Android Health Connect API 심화: 건강·피트니스 데이터 읽기·쓰기와 권한 관리 완전 정복"
date: 2026-09-09
categories: [android, flutter]
tags: [android, health-connect, healthconnectclient, jetpack, kotlin, fitness, permissions, wearable]
---

## 개요

스마트워치, 피트니스 밴드, 러닝 앱, 식단 관리 앱 — 오늘날 헬스케어 앱 생태계는 수많은 플레이어로 이루어져 있습니다. 이들이 각자의 저장소에 데이터를 쌓는다면 사용자는 심박수를 러닝 앱에서 확인하고, 수면 데이터는 스마트워치 앱에서, 체중은 또 다른 앱에서 봐야 합니다. **Android Health Connect**는 바로 이 파편화 문제를 해결하기 위해 구글이 설계한 통합 헬스케어 데이터 허브입니다.

Android 14(API 34)부터는 Android Framework에 기본 내장되어 있으며, Android 9(API 28) 이상에서는 Play 스토어를 통해 설치할 수 있습니다. 단 하나의 SDK로 심박수·걸음 수·수면·체중·운동 세션 등 50가지 이상의 데이터 타입을 읽고 쓸 수 있습니다.

이 아티클에서는 Health Connect SDK를 처음부터 끝까지 직접 구현하면서, 권한 요청·데이터 쓰기·읽기·집계·변경 감지까지 프로덕션 수준의 코드로 완전 정복합니다.

---

## 왜 Health Connect가 필요한가?

### 1. 데이터 파편화 문제

Google Fit, Samsung Health, Garmin Connect, Strava — 각 앱이 자체 저장소를 사용하면 같은 사용자의 데이터가 여러 곳에 분산됩니다. 서드파티 앱이 여러 소스의 데이터를 읽으려면 각 앱의 API를 개별적으로 연동해야 했습니다.

### 2. 프라이버시 중앙화

Health Connect는 Android의 표준 권한 시스템을 통해 **어떤 앱이 어떤 데이터에 접근할 수 있는지**를 한 곳에서 관리합니다. 사용자는 설정 화면 하나에서 모든 헬스케어 앱의 권한을 제어할 수 있습니다.

### 3. Google Fit 대체

구글은 Google Fit API를 2026년 말에 종료할 예정입니다. 현재 Google Fit을 사용 중인 앱은 반드시 Health Connect로 마이그레이션해야 합니다.

---

## 프로젝트 설정

### 의존성 추가

```kotlin
// build.gradle.kts (app)
dependencies {
    implementation("androidx.health.connect:connect-client:1.1.0-alpha11")
}
```

### AndroidManifest.xml 설정

Health Connect를 사용하는 앱은 반드시 패키지 가시성을 선언하고, 필요한 권한과 권한 이유 화면 액티비티를 등록해야 합니다.

```xml
<!-- AndroidManifest.xml -->
<manifest>

    <!-- Health Connect 패키지 가시성 선언 -->
    <queries>
        <package android:name="com.google.android.apps.healthdata" />
    </queries>

    <!-- 필요한 권한 선언 -->
    <uses-permission android:name="android.permission.health.READ_STEPS" />
    <uses-permission android:name="android.permission.health.WRITE_STEPS" />
    <uses-permission android:name="android.permission.health.READ_HEART_RATE" />
    <uses-permission android:name="android.permission.health.WRITE_HEART_RATE" />
    <uses-permission android:name="android.permission.health.READ_WEIGHT" />
    <uses-permission android:name="android.permission.health.WRITE_WEIGHT" />
    <uses-permission android:name="android.permission.health.READ_EXERCISE" />
    <uses-permission android:name="android.permission.health.WRITE_EXERCISE" />

    <application>
        <!-- 권한 이유(Rationale) 화면 액티비티 -->
        <activity android:name=".HealthConnectPrivacyPolicyActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="androidx.health.ACTION_SHOW_PERMISSIONS_RATIONALE" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```

---

## 코드 예제 1: Health Connect 초기화 및 권한 요청

권한 요청은 Android의 표준 `ActivityResultContract`를 사용합니다. `ViewModel`에서 관리하는 것이 권장 패턴입니다.

```kotlin
import androidx.health.connect.client.HealthConnectClient
import androidx.health.connect.client.permission.HealthPermission
import androidx.health.connect.client.records.StepsRecord
import androidx.health.connect.client.records.HeartRateRecord
import androidx.health.connect.client.records.WeightRecord
import androidx.health.connect.client.records.ExerciseSessionRecord
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.launch

class HealthViewModel(application: Application) : AndroidViewModel(application) {

    // 앱에서 필요한 권한 집합
    val requiredPermissions = setOf(
        HealthPermission.getReadPermission(StepsRecord::class),
        HealthPermission.getWritePermission(StepsRecord::class),
        HealthPermission.getReadPermission(HeartRateRecord::class),
        HealthPermission.getWritePermission(HeartRateRecord::class),
        HealthPermission.getReadPermission(WeightRecord::class),
        HealthPermission.getWritePermission(WeightRecord::class),
        HealthPermission.getReadPermission(ExerciseSessionRecord::class),
        HealthPermission.getWritePermission(ExerciseSessionRecord::class),
    )

    private val _uiState = MutableStateFlow<HealthUiState>(HealthUiState.Uninitialized)
    val uiState: StateFlow<HealthUiState> = _uiState

    // Health Connect 클라이언트 (지연 초기화)
    private val healthConnectClient: HealthConnectClient by lazy {
        HealthConnectClient.getOrCreate(application)
    }

    /**
     * SDK 가용성 확인 후 초기화
     * Android 14+: SDK_AVAILABLE (항상 사용 가능)
     * Android 9-13: Play 스토어 설치 여부에 따라 다름
     */
    fun initialize() {
        val sdkStatus = HealthConnectClient.getSdkStatus(getApplication())
        when (sdkStatus) {
            HealthConnectClient.SDK_UNAVAILABLE -> {
                _uiState.value = HealthUiState.Error("Health Connect를 지원하지 않는 기기입니다.")
            }
            HealthConnectClient.SDK_UNAVAILABLE_PROVIDER_UPDATE_REQUIRED -> {
                // Play 스토어에서 Health Connect 앱 설치 필요
                _uiState.value = HealthUiState.InstallRequired
            }
            HealthConnectClient.SDK_AVAILABLE -> {
                viewModelScope.launch {
                    checkAndRequestPermissions()
                }
            }
        }
    }

    /**
     * 현재 부여된 권한을 확인하고 필요한 경우 요청 트리거
     */
    suspend fun checkAndRequestPermissions() {
        val grantedPermissions = healthConnectClient.permissionController
            .getGrantedPermissions()

        if (grantedPermissions.containsAll(requiredPermissions)) {
            _uiState.value = HealthUiState.Ready
        } else {
            // 부족한 권한을 Activity에서 요청하도록 상태 변경
            val missingPermissions = requiredPermissions - grantedPermissions
            _uiState.value = HealthUiState.PermissionsRequired(missingPermissions)
        }
    }
}

// UI 상태 봉인 클래스
sealed class HealthUiState {
    object Uninitialized : HealthUiState()
    object InstallRequired : HealthUiState()
    data class PermissionsRequired(val permissions: Set<String>) : HealthUiState()
    object Ready : HealthUiState()
    data class Error(val message: String) : HealthUiState()
}
```

```kotlin
// Activity에서 권한 요청 처리
class HealthActivity : ComponentActivity() {

    private val viewModel: HealthViewModel by viewModels()

    // PermissionController가 제공하는 전용 ActivityResultContract 사용
    private val permissionLauncher = registerForActivityResult(
        PermissionController.createRequestPermissionResultContract()
    ) { grantedPermissions ->
        // 결과 처리
        viewModelScope.launch {
            viewModel.checkAndRequestPermissions()
        }
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        lifecycleScope.launch {
            viewModel.uiState.collect { state ->
                when (state) {
                    is HealthUiState.PermissionsRequired -> {
                        // 권한 요청 다이얼로그 표시
                        permissionLauncher.launch(state.permissions)
                    }
                    is HealthUiState.InstallRequired -> {
                        // Health Connect 설치 유도
                        val intent = Intent(Intent.ACTION_VIEW).apply {
                            data = Uri.parse(
                                "https://play.google.com/store/apps/details?id=com.google.android.apps.healthdata"
                            )
                        }
                        startActivity(intent)
                    }
                    else -> { /* UI 업데이트 */ }
                }
            }
        }

        viewModel.initialize()
    }
}
```

---

## 코드 예제 2: 데이터 쓰기, 읽기, 집계 완전 구현

권한이 확보된 후 실제 데이터를 읽고 쓰는 로직입니다. `HealthConnectManager`로 분리하는 것이 클린 아키텍처에 맞습니다.

```kotlin
import androidx.health.connect.client.HealthConnectClient
import androidx.health.connect.client.records.*
import androidx.health.connect.client.request.AggregateRequest
import androidx.health.connect.client.request.ReadRecordsRequest
import androidx.health.connect.client.time.TimeRangeFilter
import java.time.Instant
import java.time.ZonedDateTime
import java.time.temporal.ChronoUnit

class HealthConnectManager(private val client: HealthConnectClient) {

    // ─── 쓰기 ────────────────────────────────────────────

    /**
     * 걸음 수 기록 삽입
     * StepsRecord는 특정 시간 구간 동안의 걸음 수를 나타냄
     */
    suspend fun writeSteps(
        startTime: Instant,
        endTime: Instant,
        count: Long
    ): Boolean {
        return try {
            val record = StepsRecord(
                startTime = startTime,
                endTime = endTime,
                startZoneOffset = ZonedDateTime.now().offset,
                endZoneOffset = ZonedDateTime.now().offset,
                count = count
            )
            client.insertRecords(listOf(record))
            true
        } catch (e: Exception) {
            false
        }
    }

    /**
     * 운동 세션 기록: 운동 유형, 심박수 샘플, 거리 등을 함께 삽입
     */
    suspend fun writeExerciseSession(
        startTime: ZonedDateTime,
        endTime: ZonedDateTime,
        exerciseType: Int,       // ExerciseSessionRecord.EXERCISE_TYPE_RUNNING 등
        title: String
    ) {
        val sessionRecord = ExerciseSessionRecord(
            startTime = startTime.toInstant(),
            startZoneOffset = startTime.offset,
            endTime = endTime.toInstant(),
            endZoneOffset = endTime.offset,
            exerciseType = exerciseType,
            title = title
        )

        // 심박수 샘플 (세션 중 여러 시점)
        val heartRateRecord = HeartRateRecord(
            startTime = startTime.toInstant(),
            startZoneOffset = startTime.offset,
            endTime = endTime.toInstant(),
            endZoneOffset = endTime.offset,
            samples = listOf(
                HeartRateRecord.Sample(
                    time = startTime.plusMinutes(5).toInstant(),
                    beatsPerMinute = 140
                ),
                HeartRateRecord.Sample(
                    time = startTime.plusMinutes(10).toInstant(),
                    beatsPerMinute = 155
                ),
                HeartRateRecord.Sample(
                    time = startTime.plusMinutes(15).toInstant(),
                    beatsPerMinute = 148
                )
            )
        )

        // 체중 기록 (세션과 별개로 삽입)
        val weightRecord = WeightRecord(
            time = startTime.toInstant(),
            zoneOffset = startTime.offset,
            weight = Mass.kilograms(70.5)
        )

        // 한 번의 트랜잭션으로 여러 레코드 삽입
        client.insertRecords(listOf(sessionRecord, heartRateRecord, weightRecord))
    }

    // ─── 읽기 ────────────────────────────────────────────

    /**
     * 지난 7일간의 걸음 수 레코드를 읽어 반환
     * Health Connect는 기본적으로 최근 30일 데이터만 접근 가능
     */
    suspend fun readStepsLastWeek(): List<StepsRecord> {
        val now = Instant.now()
        val weekAgo = now.minus(7, ChronoUnit.DAYS)

        val request = ReadRecordsRequest(
            recordType = StepsRecord::class,
            timeRangeFilter = TimeRangeFilter.between(weekAgo, now)
        )

        return client.readRecords(request).records
    }

    /**
     * 최근 30일 운동 세션 목록 조회
     */
    suspend fun readExerciseSessions(): List<ExerciseSessionRecord> {
        val now = Instant.now()
        val monthAgo = now.minus(30, ChronoUnit.DAYS)

        val request = ReadRecordsRequest(
            recordType = ExerciseSessionRecord::class,
            timeRangeFilter = TimeRangeFilter.between(monthAgo, now),
            ascendingOrder = false   // 최신순 정렬
        )

        return client.readRecords(request).records
    }

    // ─── 집계 ────────────────────────────────────────────

    /**
     * 오늘의 총 걸음 수를 집계 (여러 앱/기기의 데이터 합산)
     */
    suspend fun aggregateStepsToday(): Long {
        val today = Instant.now().truncatedTo(ChronoUnit.DAYS)
        val now = Instant.now()

        val response = client.aggregate(
            AggregateRequest(
                metrics = setOf(StepsRecord.COUNT_TOTAL),
                timeRangeFilter = TimeRangeFilter.between(today, now)
            )
        )

        return response[StepsRecord.COUNT_TOTAL] ?: 0L
    }

    /**
     * 지난 7일간 심박수 통계: 평균/최대/최소
     */
    suspend fun aggregateHeartRateLastWeek(): HeartRateSummary {
        val now = Instant.now()
        val weekAgo = now.minus(7, ChronoUnit.DAYS)
        val filter = TimeRangeFilter.between(weekAgo, now)

        val response = client.aggregate(
            AggregateRequest(
                metrics = setOf(
                    HeartRateRecord.BPM_AVG,
                    HeartRateRecord.BPM_MAX,
                    HeartRateRecord.BPM_MIN
                ),
                timeRangeFilter = filter
            )
        )

        return HeartRateSummary(
            avg = response[HeartRateRecord.BPM_AVG] ?: 0,
            max = response[HeartRateRecord.BPM_MAX] ?: 0,
            min = response[HeartRateRecord.BPM_MIN] ?: 0
        )
    }

    // ─── 변경 감지(Changes API) ──────────────────────────

    /**
     * 변경 토큰 발급: 이후 데이터 변경 사항을 추적하기 위한 시작점
     * 토큰은 30일간 유효하며, DataStore에 저장해두고 재사용
     */
    suspend fun getChangesToken(): String {
        return client.getChangesToken(
            ChangesTokenRequest(
                recordTypes = setOf(
                    StepsRecord::class,
                    HeartRateRecord::class,
                    WeightRecord::class
                )
            )
        )
    }

    /**
     * 저장된 토큰 이후의 데이터 변경 사항을 조회
     * 반환된 nextChangesToken을 다음 호출에 사용
     */
    suspend fun getChangesSinceToken(token: String): ChangesResult {
        val response = client.getChanges(token)
        val upserted = mutableListOf<Record>()
        val deleted = mutableListOf<DeletedRecord>()

        for (change in response.changes) {
            when (change) {
                is UpsertionChange -> upserted.add(change.record)
                is DeletionChange  -> deleted.add(change.deletedRecord)
            }
        }

        return ChangesResult(
            upserted = upserted,
            deleted = deleted,
            nextToken = response.nextChangesToken,
            hasMoreChanges = response.hasMore
        )
    }
}

// 데이터 모델
data class HeartRateSummary(val avg: Long, val max: Long, val min: Long)
data class ChangesResult(
    val upserted: List<Record>,
    val deleted: List<DeletedRecord>,
    val nextToken: String,
    val hasMoreChanges: Boolean
)
```

---

## 백그라운드에서 데이터 읽기

기본적으로 Health Connect는 포그라운드에서만 데이터를 읽을 수 있습니다. 백그라운드 읽기(WorkManager, 알림 등)가 필요하면 추가 권한과 기능 가용성 확인이 필요합니다.

```kotlin
// AndroidManifest.xml
<uses-permission android:name="android.permission.health.READ_HEALTH_DATA_IN_BACKGROUND" />

// 기능 가용성 확인
suspend fun checkBackgroundReadAvailability(): Boolean {
    val status = client.features.getFeatureStatus(
        HealthConnectFeatures.FEATURE_READ_HEALTH_DATA_IN_BACKGROUND
    )
    return status == HealthConnectFeatures.FEATURE_STATUS_AVAILABLE
}
```

백그라운드 읽기는 WorkManager와 조합해서 야간 수면 분석, 일일 활동 요약 등에 활용합니다.

---

## 주의사항 및 실전 팁

### 1. 권한은 언제든 철회될 수 있다
사용자가 설정에서 권한을 철회할 수 있으므로, 매번 API 호출 전 `getGrantedPermissions()`로 권한을 재확인하거나 `SecurityException`을 적절히 처리해야 합니다.

```kotlin
suspend fun safeReadSteps(): List<StepsRecord> {
    val granted = client.permissionController.getGrantedPermissions()
    if (!granted.contains(HealthPermission.getReadPermission(StepsRecord::class))) {
        return emptyList()  // 권한 없음 처리
    }
    return readStepsLastWeek()
}
```

### 2. 30일 데이터 접근 제한
기본적으로 다른 앱이 쓴 데이터는 최근 30일만 읽을 수 있습니다. 더 오래된 기록이 필요하다면 `READ_HEALTH_DATA_HISTORY` 권한을 별도로 요청해야 합니다(구글 플레이 정책 검토 필요).

### 3. 데이터 타입별 집계 가능 여부 확인
모든 레코드 타입이 집계를 지원하지는 않습니다. `StepsRecord.COUNT_TOTAL`처럼 각 레코드 타입의 companion object에 선언된 집계 메트릭을 사용하세요.

### 4. 개인정보처리방침 필수
Play 스토어에 앱을 출시하려면 Health Connect 사용에 대한 개인정보처리방침 URL을 Google Play 콘솔에 등록하고, 앱 내 권한 이유 화면(`ACTION_SHOW_PERMISSIONS_RATIONALE`)도 반드시 구현해야 합니다.

### 5. 테스트: 에뮬레이터보다 실기기 권장
Health Connect는 실기기에서 훨씬 안정적으로 동작합니다. 에뮬레이터에서는 Health Connect 앱 설치가 안 되거나 API 호출이 실패할 수 있습니다.

---

## 정리

| 기능 | 메서드 | 비고 |
|------|--------|------|
| SDK 상태 확인 | `HealthConnectClient.getSdkStatus()` | Android 14+는 항상 AVAILABLE |
| 권한 요청 | `PermissionController.createRequestPermissionResultContract()` | 표준 ActivityResult |
| 데이터 삽입 | `client.insertRecords()` | 여러 레코드 한 번에 삽입 가능 |
| 데이터 읽기 | `client.readRecords()` | TimeRangeFilter 필수 |
| 집계 | `client.aggregate()` | 평균/합계/최대/최소 지원 |
| 변경 감지 | `client.getChanges()` | 토큰 기반, 30일 유효 |
| 백그라운드 읽기 | 별도 권한 + 기능 확인 | WorkManager와 조합 |

Android Health Connect API는 헬스케어 앱의 데이터 파편화 문제를 해결하는 가장 현대적인 방법입니다. Google Fit 종료(2026년 말) 전에 마이그레이션을 시작하고, 단일 통합 API로 사용자에게 더 일관된 헬스케어 경험을 제공하세요.

---

## 참고 자료

- [Health Connect 시작 가이드 — Android Developers](https://developer.android.com/health-and-fitness/health-connect/get-started)
- [Health Connect 완전 통합 Codelab](https://developer.android.com/codelabs/health-connect)
- [Health Connect Jetpack 릴리즈 노트](https://developer.android.com/jetpack/androidx/releases/health-connect)
- [HealthConnectClient API 레퍼런스](https://developer.android.com/reference/androidx/health/connect/client/HealthConnectClient)
