---
layout: post
title: "Android WorkManager 심화: WorkChain·Constraint·PeriodicWork 완전 정복"
date: 2026-06-16
categories: [android]
tags: [android, workmanager, kotlin, background, jetpack, coroutines]
---

## WorkManager란 무엇인가?

Android에서 백그라운드 작업을 신뢰성 있게 실행하는 것은 오랫동안 개발자들의 골칫거리였습니다. `AsyncTask`는 메모리 누수를 일으키기 쉬웠고, `Service`는 Android 8.0(Oreo) 이후 제한이 강화되었으며, `AlarmManager`는 정확한 시간 제어에는 유용하지만 보장된 실행을 제공하지 않습니다. **WorkManager**는 이러한 문제들을 해결하기 위해 Jetpack이 제공하는 공식 백그라운드 작업 스케줄러입니다.

WorkManager는 앱이 종료되거나 기기가 재부팅되더라도 예약된 작업이 **반드시 실행됨을 보장**합니다. 내부적으로 Android 버전에 따라 `JobScheduler`(API 23+) 또는 `BroadcastReceiver` + `AlarmManager`를 자동으로 선택해 사용합니다. 개발자는 WorkManager의 단일 API만 사용하면 됩니다.

### WorkManager가 적합한 시나리오

| 시나리오 | 적합 여부 |
|---|---|
| 서버에 로그/분석 데이터 업로드 | ✅ 적합 |
| 주기적인 데이터 동기화 | ✅ 적합 |
| 이미지 압축 및 업로드 파이프라인 | ✅ 적합 |
| 정확히 특정 시각에 실행해야 하는 알림 | ❌ AlarmManager 사용 권장 |
| 즉시 실행이 필요한 포그라운드 작업 | ❌ Coroutine/Thread 사용 권장 |

---

## 왜 WorkManager인가?

Android 배터리 최적화 정책(Doze Mode, App Standby)이 강화되면서 백그라운드 작업은 점점 더 제약을 받습니다. WorkManager는 이러한 OS 제약 안에서 동작하되, 작업이 지연될 수는 있어도 **결국에는 실행됨을 보장**합니다. 이것이 다른 스케줄러와의 핵심 차이입니다.

또한 WorkManager는 **코루틴과의 완벽한 통합**을 제공합니다. `CoroutineWorker`를 사용하면 suspend 함수 내에서 네트워크 요청이나 Room 데이터베이스 작업을 자연스럽게 처리할 수 있습니다.

---

## Constraints: 작업 실행 조건 정의

WorkManager의 강력한 기능 중 하나는 **Constraints(제약 조건)**입니다. 네트워크 연결 상태, 배터리 수준, 기기 충전 여부, 저장 공간 상태 등을 기반으로 작업 실행 시점을 세밀하게 제어할 수 있습니다.

### 사용 가능한 Constraint 종류

- `setRequiredNetworkType(NetworkType)` — 네트워크 타입 (CONNECTED, UNMETERED, METERED 등)
- `setRequiresCharging(Boolean)` — 기기 충전 중 여부
- `setRequiresBatteryNotLow(Boolean)` — 배터리 부족 상태 아닌지 여부
- `setRequiresDeviceIdle(Boolean)` — 기기 유휴 상태 여부 (API 23+)
- `setRequiresStorageNotLow(Boolean)` — 저장 공간 부족 상태 아닌지 여부

---

## 실제 구현 예제

### 예제 1: Constraint와 CoroutineWorker를 사용한 데이터 동기화

실제 앱에서 자주 사용되는 패턴입니다. Wi-Fi 연결 상태이고 배터리가 부족하지 않을 때만 서버와 데이터를 동기화하는 Worker입니다.

```kotlin
import androidx.work.*
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext
import java.util.concurrent.TimeUnit

// 1. CoroutineWorker 구현
class DataSyncWorker(
    context: Context,
    params: WorkerParameters,
    private val syncRepository: SyncRepository
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        val userId = inputData.getString(KEY_USER_ID)
            ?: return Result.failure(
                workDataOf("error" to "userId is missing")
            )

        return try {
            withContext(Dispatchers.IO) {
                val lastSyncTime = inputData.getLong(KEY_LAST_SYNC_TIME, 0L)
                val syncResult = syncRepository.syncUserData(userId, lastSyncTime)

                if (syncResult.isSuccess) {
                    Result.success(
                        workDataOf("synced_count" to syncResult.getOrNull()?.count)
                    )
                } else {
                    // 재시도 가능한 오류이면 retry, 그렇지 않으면 failure
                    if (syncResult.exceptionOrNull() is IOException) {
                        Result.retry()
                    } else {
                        Result.failure(workDataOf("error" to syncResult.exceptionOrNull()?.message))
                    }
                }
            }
        } catch (e: Exception) {
            Result.retry()
        }
    }

    companion object {
        const val KEY_USER_ID = "user_id"
        const val KEY_LAST_SYNC_TIME = "last_sync_time"
        const val WORK_NAME = "data_sync_work"
    }
}

// 2. Constraints + PeriodicWorkRequest 구성
class WorkScheduler(private val workManager: WorkManager) {

    fun schedulePeriodicSync(userId: String) {
        val constraints = Constraints.Builder()
            .setRequiredNetworkType(NetworkType.UNMETERED)  // Wi-Fi 전용
            .setRequiresBatteryNotLow(true)                 // 배터리 부족 시 건너뜀
            .build()

        val inputData = workDataOf(
            DataSyncWorker.KEY_USER_ID to userId,
            DataSyncWorker.KEY_LAST_SYNC_TIME to System.currentTimeMillis()
        )

        val syncRequest = PeriodicWorkRequestBuilder<DataSyncWorker>(
            repeatInterval = 6,
            repeatIntervalTimeUnit = TimeUnit.HOURS,
            flexTimeInterval = 30,
            flexTimeIntervalUnit = TimeUnit.MINUTES  // 6시간±30분 유연 창
        )
            .setConstraints(constraints)
            .setInputData(inputData)
            .setBackoffCriteria(
                BackoffPolicy.EXPONENTIAL,  // 지수 백오프 재시도
                WorkRequest.MIN_BACKOFF_MILLIS,
                TimeUnit.MILLISECONDS
            )
            .build()

        // ExistingPeriodicWorkPolicy.KEEP: 이미 예약된 작업이 있으면 유지
        // ExistingPeriodicWorkPolicy.REPLACE: 기존 작업을 새 작업으로 교체
        workManager.enqueueUniquePeriodicWork(
            DataSyncWorker.WORK_NAME,
            ExistingPeriodicWorkPolicy.KEEP,
            syncRequest
        )
    }

    fun observeSyncStatus(lifecycleOwner: LifecycleOwner) {
        workManager
            .getWorkInfosForUniqueWorkLiveData(DataSyncWorker.WORK_NAME)
            .observe(lifecycleOwner) { workInfos ->
                val latestInfo = workInfos?.firstOrNull() ?: return@observe
                when (latestInfo.state) {
                    WorkInfo.State.RUNNING   -> Log.d("Sync", "동기화 실행 중...")
                    WorkInfo.State.SUCCEEDED -> {
                        val count = latestInfo.outputData.getInt("synced_count", 0)
                        Log.d("Sync", "동기화 완료: $count 건")
                    }
                    WorkInfo.State.FAILED    -> Log.e("Sync", "동기화 실패")
                    WorkInfo.State.ENQUEUED  -> Log.d("Sync", "제약 조건 대기 중...")
                    else -> Unit
                }
            }
    }
}
```

---

### 예제 2: WorkChain으로 이미지 처리 파이프라인 구성

WorkManager의 **WorkChain(작업 체인)**은 여러 작업을 순서대로 또는 병렬로 실행하고, 이전 작업의 출력을 다음 작업의 입력으로 전달하는 기능입니다. 이미지 다운로드 → 리사이즈 → 워터마크 삽입 → 업로드로 이어지는 파이프라인을 구성해보겠습니다.

```kotlin
import androidx.work.*
import java.util.concurrent.TimeUnit

// --- Worker 1: 이미지 다운로드 ---
class ImageDownloadWorker(context: Context, params: WorkerParameters)
    : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        val imageUrl = inputData.getString("image_url") ?: return Result.failure()

        return withContext(Dispatchers.IO) {
            val localPath = downloadImage(imageUrl)  // 실제 다운로드 로직
            Result.success(workDataOf("local_path" to localPath))
        }
    }
}

// --- Worker 2: 이미지 리사이즈 ---
class ImageResizeWorker(context: Context, params: WorkerParameters)
    : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        // 이전 Worker의 출력을 inputData로 자동 수신
        val localPath = inputData.getString("local_path") ?: return Result.failure()
        val targetWidth = inputData.getInt("target_width", 1080)

        return withContext(Dispatchers.Default) {
            val resizedPath = resizeImage(localPath, targetWidth)
            Result.success(workDataOf("resized_path" to resizedPath))
        }
    }
}

// --- Worker 3-A: 워터마크 삽입 (병렬 분기 A) ---
class WatermarkWorker(context: Context, params: WorkerParameters)
    : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        val resizedPath = inputData.getString("resized_path") ?: return Result.failure()
        return withContext(Dispatchers.Default) {
            val markedPath = applyWatermark(resizedPath, "© DuckTudy")
            Result.success(workDataOf("final_path" to markedPath))
        }
    }
}

// --- Worker 3-B: 썸네일 생성 (병렬 분기 B) ---
class ThumbnailWorker(context: Context, params: WorkerParameters)
    : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        val resizedPath = inputData.getString("resized_path") ?: return Result.failure()
        return withContext(Dispatchers.Default) {
            val thumbnailPath = generateThumbnail(resizedPath, 200)
            Result.success(workDataOf("thumbnail_path" to thumbnailPath))
        }
    }
}

// --- Worker 4: 최종 업로드 (병렬 분기 합류) ---
class ImageUploadWorker(context: Context, params: WorkerParameters)
    : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        // ArrayCreatingInputMerger 사용 시 병렬 분기 결과가 배열로 합쳐짐
        val finalPath    = inputData.getString("final_path")
        val thumbnailPath = inputData.getString("thumbnail_path")

        return withContext(Dispatchers.IO) {
            uploadToServer(finalPath, thumbnailPath)
            Result.success()
        }
    }
}

// --- WorkChain 조합 ---
class ImagePipelineScheduler(private val workManager: WorkManager) {

    fun startImagePipeline(imageUrl: String) {
        val constraints = Constraints.Builder()
            .setRequiredNetworkType(NetworkType.CONNECTED)
            .build()

        // Step 1: 다운로드
        val downloadRequest = OneTimeWorkRequestBuilder<ImageDownloadWorker>()
            .setInputData(workDataOf("image_url" to imageUrl))
            .setConstraints(constraints)
            .build()

        // Step 2: 리사이즈
        val resizeRequest = OneTimeWorkRequestBuilder<ImageResizeWorker>()
            .setInputData(workDataOf("target_width" to 1080))
            .build()

        // Step 3-A: 워터마크 (병렬)
        val watermarkRequest = OneTimeWorkRequestBuilder<WatermarkWorker>().build()

        // Step 3-B: 썸네일 (병렬)
        val thumbnailRequest = OneTimeWorkRequestBuilder<ThumbnailWorker>().build()

        // Step 4: 업로드 — ArrayCreatingInputMerger로 3-A, 3-B 결과 통합
        val uploadRequest = OneTimeWorkRequestBuilder<ImageUploadWorker>()
            .setInputMerger(ArrayCreatingInputMerger::class)
            .setConstraints(constraints)
            .build()

        // 체인 구성:
        // download → resize → [watermark, thumbnail] → upload
        workManager
            .beginWith(downloadRequest)
            .then(resizeRequest)
            .then(listOf(watermarkRequest, thumbnailRequest))  // 병렬 실행
            .then(uploadRequest)
            .enqueue()
    }
}
```

위 예제에서 핵심은 `beginWith().then().then().enqueue()` 패턴입니다. `then(listOf(...))`를 사용하면 여러 Worker가 병렬로 실행되고, 그 다음 `then()`에서 합류합니다. 병렬 분기가 합류할 때 여러 Worker의 출력 데이터를 어떻게 병합할지는 **InputMerger**로 결정합니다.

---

## InputMerger 이해하기

WorkChain에서 병렬로 실행된 작업들의 결과가 하나의 Worker로 합쳐질 때, 동일한 키를 가진 데이터를 어떻게 처리할지 결정하는 것이 InputMerger입니다.

| InputMerger | 동작 방식 |
|---|---|
| `OverwritingInputMerger` (기본값) | 동일한 키가 충돌하면 마지막 값으로 덮어씀 |
| `ArrayCreatingInputMerger` | 동일한 키의 값들을 배열로 묶어 전달 |

병렬 분기가 같은 키(`"result"`)로 값을 반환한다면 `ArrayCreatingInputMerger`를 사용하세요. 그렇지 않으면 한 쪽 값이 유실됩니다.

---

## 고유 작업(UniqueWork)으로 중복 실행 방지

`enqueueUniqueWork` 또는 `enqueueUniquePeriodicWork`를 사용하면 동일 이름의 작업이 이미 실행 중일 때의 동작을 `ExistingWorkPolicy`로 제어할 수 있습니다.

```kotlin
// OneTimeWork 기준
workManager.enqueueUniqueWork(
    "image_sync",
    ExistingWorkPolicy.APPEND_OR_REPLACE,  // 실패/취소된 경우 새 작업으로 교체
    syncRequest
)
```

| Policy | 동작 |
|---|---|
| `KEEP` | 기존 작업이 RUNNING/ENQUEUED면 새 요청 무시 |
| `REPLACE` | 기존 작업을 취소하고 새 작업으로 대체 |
| `APPEND` | 기존 작업이 완료된 후 새 작업을 체인에 추가 |
| `APPEND_OR_REPLACE` | APPEND와 유사하나 기존 작업이 FAILED/CANCELLED면 REPLACE |

---

## 주의사항 및 실전 팁

### 1. PeriodicWork의 최소 간격은 15분

`PeriodicWorkRequest`의 반복 간격은 **최소 15분**입니다. 15분 미만으로 설정하면 자동으로 15분으로 조정됩니다. 더 짧은 주기가 필요한 경우 WorkManager가 아닌 Coroutine + `delay`나 `Handler`를 고려하세요.

### 2. CoroutineWorker는 자동으로 취소 처리됨

`CoroutineWorker`의 `doWork()`는 기본적으로 `Dispatchers.Default`에서 실행되며, WorkManager가 작업을 취소할 때 자동으로 코루틴도 취소됩니다. `withContext(Dispatchers.IO)`를 적극 활용해 I/O 바운드 작업을 분리하세요.

### 3. Hilt를 이용한 Worker DI (의존성 주입)

WorkManager는 기본적으로 Worker의 생성자에 `Context`와 `WorkerParameters`만 허용합니다. Repository 등 의존성을 주입하려면 **HiltWorker**를 사용하세요.

```kotlin
@HiltWorker
class DataSyncWorker @AssistedInject constructor(
    @Assisted context: Context,
    @Assisted params: WorkerParameters,
    private val syncRepository: SyncRepository  // Hilt로 주입
) : CoroutineWorker(context, params) {
    override suspend fun doWork(): Result { /* ... */ }
}

// Application에서 HiltWorkerFactory 등록
@HiltAndroidApp
class App : Application(), Configuration.Provider {
    @Inject lateinit var workerFactory: HiltWorkerFactory

    override val workManagerConfiguration: Configuration
        get() = Configuration.Builder()
            .setWorkerFactory(workerFactory)
            .build()
}
```

### 4. 작업 상태 모니터링은 Flow로도 가능

LiveData 대신 Flow 기반 API를 사용하면 더 현대적인 방식으로 상태를 관찰할 수 있습니다.

```kotlin
viewModelScope.launch {
    workManager.getWorkInfosForUniqueWorkFlow("data_sync_work")
        .collect { workInfos ->
            val state = workInfos.firstOrNull()?.state
            _syncState.value = state
        }
}
```

### 5. 테스트 시 WorkManagerTestInitHelper 활용

단위 테스트에서는 실제 WorkManager 대신 테스트용 구현체를 사용하세요. Constraint 조건 없이 즉시 실행시킬 수 있습니다.

```kotlin
@Before
fun setup() {
    val config = Configuration.Builder()
        .setMinimumLoggingLevel(Log.DEBUG)
        .setExecutor(SynchronousExecutor())
        .build()
    WorkManagerTestInitHelper.initializeTestWorkManager(context, config)
}

@Test
fun testSyncWorker() {
    val testDriver = WorkManagerTestInitHelper.getTestDriver(context)!!
    val request = PeriodicWorkRequestBuilder<DataSyncWorker>(15, TimeUnit.MINUTES).build()

    workManager.enqueueUniquePeriodicWork("test", ExistingPeriodicWorkPolicy.KEEP, request)
    testDriver.setPeriodDelayMet(request.id)  // 주기 지연 강제 충족
    testDriver.setAllConstraintsMet(request.id)  // 제약 조건 강제 충족

    val workInfo = workManager.getWorkInfoById(request.id).get()
    assertEquals(WorkInfo.State.ENQUEUED, workInfo?.state)
}
```

---

## 정리

WorkManager는 Android 백그라운드 작업의 표준이며, 다음 세 가지 개념을 마스터하면 대부분의 시나리오를 처리할 수 있습니다.

1. **Constraint** — 작업이 실행될 조건을 선언적으로 정의
2. **WorkChain** — `beginWith().then().enqueue()`로 복잡한 작업 파이프라인 구성
3. **UniqueWork** — `ExistingWorkPolicy`로 중복 실행과 작업 교체 전략 제어

`CoroutineWorker`와 Hilt DI를 결합하면 테스트 가능하고 유지보수하기 좋은 백그라운드 작업 시스템을 만들 수 있습니다.

## 참고 자료
- [Getting started with WorkManager — Android Developers](https://developer.android.com/develop/background-work/background-tasks/persistent/getting-started)
- [Chaining work — Android Developers](https://developer.android.com/topic/libraries/architecture/workmanager/how-to/chain-work)
- [Managing work — Android Developers](https://developer.android.com/develop/background-work/background-tasks/persistent/how-to/manage-work)
- [Advanced WorkManager Codelab — Android Developers](https://developer.android.com/codelabs/android-adv-workmanager)
