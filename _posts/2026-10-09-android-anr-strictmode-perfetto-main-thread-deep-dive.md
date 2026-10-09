---
layout: post
title: "Android ANR 완전 분석 및 예방: StrictMode·Perfetto 트레이싱으로 메인 스레드 병목을 잡는 법"
date: 2026-10-09
categories: [android]
tags: [android, anr, strictmode, perfetto, coroutines, performance, kotlin]
---

앱이 5초 이상 응답하지 않으면 Android 시스템은 "ANR(Application Not Responding)" 다이얼로그를 사용자에게 보여줍니다. 사용자는 그 순간 앱을 강제 종료하거나, 최악의 경우 앱을 삭제합니다. Android Vitals 기준으로 ANR이 전체 세션의 0.47%를 초과하면 Play Store 노출에 불이익이 생깁니다. 이 글에서는 ANR의 발생 원인을 구조적으로 이해하고, `StrictMode`와 `Perfetto` 트레이싱을 활용해 사전에 탐지·예방하는 심화 전략을 다룹니다.

---

## ANR이란 무엇인가

Android의 모든 UI 작업은 **메인 스레드(Main Thread, UI Thread)**에서 실행됩니다. 이 스레드가 처리해야 할 이벤트를 특정 시간 안에 처리하지 못하면 ANR이 트리거됩니다.

**ANR 트리거 조건 (공식 기준)**

| 유형 | 타임아웃 |
|---|---|
| Input Dispatching (터치, 키 입력) | 5초 |
| Broadcast Receiver (포그라운드) | 10초 |
| Broadcast Receiver (백그라운드) | 60초 |
| Service (포그라운드) | 20초 |
| Service (백그라운드) | 200초 |

가장 흔한 ANR 유형은 Input Dispatching입니다. 사용자가 화면을 터치했는데 메인 스레드가 다른 작업으로 점유되어 5초 안에 이벤트를 처리하지 못하면 발생합니다.

---

## 왜 ANR 예방이 어려운가

ANR의 진짜 문제는 **재현이 어렵다는 점**입니다. 개발 환경에서는 잘 동작하다가, 출시 후 저사양 기기나 네트워크가 느린 환경에서만 발생합니다. 원인도 다양합니다:

- **메인 스레드에서의 I/O**: `SharedPreferences.commit()`, 파일 읽기, 데이터베이스 쿼리
- **메인 스레드에서의 네트워크 호출**: 오래된 코드의 `HttpURLConnection` 동기 호출
- **Lock 경합**: 여러 스레드가 동일한 뮤텍스를 두고 다툴 때 메인 스레드가 대기
- **Binder 호출 지연**: 다른 앱/시스템 서비스의 느린 Binder 응답
- **과도한 초기화**: `Application.onCreate()`에서 수십 개의 SDK를 동기 초기화

개발자는 이를 사전에 잡기 위해 `StrictMode`를 활용하고, 운영 중 발생한 ANR을 분석하기 위해 `Perfetto`를 사용합니다.

---

## 실제 구현 예제 1: StrictMode로 개발 중 위반 탐지하기

`StrictMode`는 Android가 제공하는 개발 시 진단 도구로, 메인 스레드에서의 디스크/네트워크 I/O나 기타 정책 위반을 실시간으로 감지합니다. **반드시 배포 빌드에서는 비활성화해야 합니다.**

```kotlin
class MyApplication : Application() {

    override fun onCreate() {
        super.onCreate()

        if (BuildConfig.DEBUG) {
            enableStrictMode()
        }

        // SDK 초기화는 반드시 비동기로
        initializeSdksAsync()
    }

    private fun enableStrictMode() {
        StrictMode.setThreadPolicy(
            StrictMode.ThreadPolicy.Builder()
                .detectAll()                    // 모든 스레드 정책 위반 감지
                .penaltyLog()                   // Logcat에 스택 트레이스 출력
                .penaltyDialog()                // 위반 시 다이얼로그 표시 (선택)
                // .penaltyDeath()              // 위반 시 앱 강제 종료 (엄격 모드)
                .build()
        )

        StrictMode.setVmPolicy(
            StrictMode.VmPolicy.Builder()
                .detectLeakedSqlLiteObjects()   // SQLite 커서 누수 탐지
                .detectLeakedClosableObjects()  // Closeable 미닫기 탐지
                .detectActivityLeaks()          // Activity 메모리 누수 탐지
                .detectUnsafeIntentLaunch()     // 안전하지 않은 Intent 탐지 (API 31+)
                .penaltyLog()
                .build()
        )
    }

    private fun initializeSdksAsync() {
        // Coroutine으로 백그라운드에서 SDK 초기화
        applicationScope.launch(Dispatchers.IO) {
            AnalyticsSdk.initialize(this@MyApplication)
            CrashReporterSdk.initialize(this@MyApplication)
        }
    }
}
```

`StrictMode`가 위반을 감지하면 Logcat에 다음과 같은 스택 트레이스가 출력됩니다:

```
D/StrictMode: StrictMode policy violation; ~duration=45 ms: 
  android.os.StrictMode$StrictModeDiskReadViolation: policy=0x3
    at android.os.StrictMode$AndroidBlockGuardPolicy.onReadFromDisk(StrictMode.java:1504)
    at java.io.UnixFileSystem.checkAccess(UnixFileSystem.java:251)
    at com.example.app.UserRepository.getUserSync(UserRepository.kt:42)
    at com.example.app.MainActivity.onCreate(MainActivity.kt:28)
```

이 로그를 보고 `UserRepository.getUserSync()`가 메인 스레드에서 디스크 I/O를 수행함을 즉시 알 수 있습니다.

---

## 실제 구현 예제 2: Kotlin Coroutines로 메인 스레드 안전하게 보호하기

ANR 예방의 핵심은 **시간이 걸릴 수 있는 모든 작업을 메인 스레드 밖으로 이동**하는 것입니다. Kotlin Coroutines의 `Dispatchers`를 활용하면 이를 구조적으로 구현할 수 있습니다.

```kotlin
// ViewModel에서 ANR-safe 데이터 로딩 패턴
@HiltViewModel
class UserProfileViewModel @Inject constructor(
    private val userRepository: UserRepository,
    private val preferencesRepository: PreferencesRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow<UserProfileUiState>(UserProfileUiState.Loading)
    val uiState: StateFlow<UserProfileUiState> = _uiState.asStateFlow()

    init {
        loadUserProfile()
    }

    private fun loadUserProfile() {
        viewModelScope.launch {
            _uiState.value = UserProfileUiState.Loading
            try {
                // withContext(Dispatchers.IO): 디스크/네트워크 I/O를 IO 스레드 풀로 위임
                val user = withContext(Dispatchers.IO) {
                    userRepository.getCurrentUser()     // Room 쿼리
                }
                val preferences = withContext(Dispatchers.IO) {
                    preferencesRepository.getAll()      // DataStore 읽기
                }
                // 결과는 자동으로 메인 스레드에서 처리됨
                _uiState.value = UserProfileUiState.Success(user, preferences)
            } catch (e: Exception) {
                _uiState.value = UserProfileUiState.Error(e.message ?: "Unknown error")
            }
        }
    }

    // SharedPreferences 대신 DataStore를 사용해 ANR 원인 제거
    // SharedPreferences.commit()은 메인 스레드를 블로킹함
    // DataStore는 항상 비동기(suspend 함수)로 동작
    suspend fun saveUserSetting(key: String, value: String) {
        withContext(Dispatchers.IO) {
            preferencesRepository.save(key, value)  // DataStore suspend 함수
        }
    }
}

// Activity에서 ANR-safe BroadcastReceiver 등록
class NetworkStatusReceiver : BroadcastReceiver() {

    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.Main.immediate)

    override fun onReceive(context: Context, intent: Intent) {
        // goAsync()로 비동기 처리 (기본 10초 타임아웃 연장)
        val pendingResult = goAsync()

        scope.launch {
            try {
                withContext(Dispatchers.IO) {
                    // 네트워크 상태 처리 (무거운 작업)
                    processNetworkStatusChange(intent)
                }
            } finally {
                // 반드시 finish() 호출로 시스템에 완료 신호
                pendingResult.finish()
            }
        }
    }

    override fun onDestroy() {
        scope.cancel()
    }
}

// 메인 스레드 블로킹 없이 Lock을 사용하는 패턴
// 일반 synchronized 대신 Mutex 사용
class SafeCache<K, V> {
    private val mutex = Mutex()
    private val cache = mutableMapOf<K, V>()

    // suspend 함수: Lock 획득 중 메인 스레드를 블로킹하지 않고 suspend
    suspend fun get(key: K): V? = mutex.withLock {
        cache[key]
    }

    suspend fun put(key: K, value: V) = mutex.withLock {
        cache[key] = value
    }
}
```

---

## Perfetto로 운영 중 ANR 분석하기

배포 후 ANR이 발생하면 `Perfetto` 시스템 트레이싱으로 원인을 분석합니다.

**1단계: 기기에서 트레이스 수집**

```bash
# adb로 Perfetto 트레이스 수집 (Android 10+)
adb shell perfetto \
  -c - --txt \
  -o /data/misc/perfetto-traces/trace \
<<EOF
buffers: {
  size_kb: 102400
  fill_policy: DISCARD
}
data_sources: {
  config {
    name: "linux.ftrace"
    ftrace_config {
      ftrace_events: "sched/sched_switch"
      ftrace_events: "sched/sched_waking"
      ftrace_events: "binder/binder_transaction"
      atrace_categories: "am"
      atrace_categories: "wm"
      atrace_categories: "view"
    }
  }
}
data_sources: {
  config {
    name: "android.log"
    android_log_config {
      min_prio: PRIO_WARN
    }
  }
}
duration_ms: 10000
EOF

# 트레이스 파일 PC로 가져오기
adb pull /data/misc/perfetto-traces/trace ./anr_trace.perfetto
```

**2단계: Perfetto UI에서 분석**

`https://ui.perfetto.dev`에 파일을 드래그하면 타임라인이 열립니다. 분석 포인트:

- `main` 스레드 행에서 긴 회색/빨간 구간(BLOCKED 상태) 찾기
- Binder 트랜잭션이 길게 이어지는 구간 확인
- `sched_switch`에서 메인 스레드가 Runnable임에도 CPU를 못 받는 구간 확인

**3단계: ApplicationExitInfo로 필드 ANR 원인 파악**

Android 11 이상에서는 앱 종료 원인을 코드로 확인할 수 있습니다.

```kotlin
// Application.onCreate()에서 이전 세션의 ANR 원인 수집
fun collectExitReasons(context: Context) {
    val activityManager = context.getSystemService(ActivityManager::class.java)
    val exitReasons = activityManager.getHistoricalProcessExitReasons(
        context.packageName,
        0,  // 0 = 가장 최근
        5   // 최대 5개
    )

    exitReasons
        .filter { it.reason == ApplicationExitInfo.REASON_ANR }
        .forEach { exitInfo ->
            // ANR 트레이스를 InputStream으로 읽기
            exitInfo.traceInputStream?.use { stream ->
                val trace = stream.bufferedReader().readText()
                // Firebase Crashlytics나 서버로 전송
                CrashReporter.recordAnrTrace(
                    timestamp = exitInfo.timestamp,
                    description = exitInfo.description,
                    trace = trace
                )
            }
        }
}
```

---

## ANR을 만드는 흔한 패턴과 수정법

### 1. `SharedPreferences.commit()` → DataStore

```kotlin
// 나쁜 예: commit()은 메인 스레드를 블로킹
sharedPreferences.edit().putString("key", "value").commit()

// 좋은 예: apply()는 비동기 (단, 즉시 반영 보장 없음)
sharedPreferences.edit().putString("key", "value").apply()

// 최선: DataStore를 사용 (완전 비동기, 타입 안전)
suspend fun saveKey(value: String) {
    dataStore.edit { prefs -> prefs[KEY] = value }
}
```

### 2. `Bitmap` 처리

```kotlin
// 나쁜 예: 고해상도 Bitmap 디코딩을 메인 스레드에서 수행
val bitmap = BitmapFactory.decodeFile(path)  // ANR 원인!

// 좋은 예: IO 스레드에서 처리 후 메인 스레드에 전달
val bitmap = withContext(Dispatchers.IO) {
    val options = BitmapFactory.Options().apply {
        inSampleSize = 2  // 메모리도 절약
    }
    BitmapFactory.decodeFile(path, options)
}
imageView.setImageBitmap(bitmap)
```

---

## 주의사항 및 팁

**1. `StrictMode`는 반드시 `DEBUG` 빌드에서만**
`penaltyDeath()`를 프로덕션에 남기면 사용자 기기에서 앱이 강제 종료됩니다.

**2. `goAsync()`의 타임아웃을 지키세요**
포그라운드 BroadcastReceiver에서 `goAsync()`를 사용해도 기본 타임아웃(10초)은 그대로입니다. 10초 안에 `pendingResult.finish()`를 호출해야 합니다.

**3. Binder 트랜잭션 크기를 최소화하세요**
Activity와 Fragment 사이에 `Bundle`로 전달하는 데이터가 너무 크면(1MB 초과) `TransactionTooLargeException`이 발생하고 ANR로 이어질 수 있습니다. 큰 데이터는 `ViewModel`이나 `Repository`에 두고 ID만 전달하세요.

**4. `Application.onCreate()`는 비어있어야 합니다**
앱 시작 시간과 ANR은 직결됩니다. Jetpack App Startup 라이브러리로 SDK 초기화를 지연하고, 비동기로 처리하세요.

**5. Android Vitals 지속 모니터링**
Google Play Console의 Android Vitals에서 ANR 발생률을 주기적으로 확인하세요. 디바이스 모델, Android 버전, 앱 버전별로 필터링해 특정 기기에서만 발생하는 ANR을 잡을 수 있습니다.

**6. `Mutex` vs `synchronized`**
Coroutines 환경에서 `synchronized` 블록은 스레드를 블로킹합니다. 메인 스레드에서 `synchronized`를 사용하면 ANR이 발생할 수 있습니다. 항상 Coroutines의 `Mutex`를 사용하세요.

---

## 정리

| 단계 | 도구 | 목적 |
|---|---|---|
| 개발 중 | StrictMode | 메인 스레드 I/O 위반 실시간 탐지 |
| 테스트 중 | Android Studio Profiler | CPU/메모리 점유 시각화 |
| 운영 중 | Android Vitals | 필드 ANR 발생률 모니터링 |
| 분석 시 | Perfetto, ApplicationExitInfo | ANR 원인 심층 분석 |

ANR은 단순히 "무거운 작업을 백그라운드로"의 문제가 아닙니다. Lock 경합, Binder 지연, 과도한 초기화 등 여러 원인이 복합적으로 작용합니다. `StrictMode`로 개발 단계에서 위반을 잡고, Kotlin Coroutines로 구조적인 안전망을 구축하며, `Perfetto`와 `ApplicationExitInfo`로 운영 중 발생한 ANR을 정밀 분석하는 것이 완성된 ANR 예방 전략입니다.

## 참고 자료
- [Keep your app responsive - Android Developers](https://developer.android.com/training/articles/perf-anr)
- [ANRs (Android Vitals) - Android Developers](https://developer.android.com/topic/performance/vitals/anr)
- [Overview of system tracing - Android Developers](https://developer.android.com/topic/performance/tracing/)
- [StrictMode API Reference - Android Developers](https://developer.android.com/reference/android/os/StrictMode)
