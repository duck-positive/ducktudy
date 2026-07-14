---
layout: post
title: "Android App Startup Library 심화: 의존성 그래프 기반 초기화 순서 제어와 스타트업 최적화 완전 정복"
date: 2026-07-14
categories: [android]
tags: [android, jetpack, app-startup, performance, cold-start, initializer, contentprovider]
---

앱이 처음 실행되는 순간, 사용자는 이미 판단을 시작한다. Google의 Android Vitals 기준으로 콜드 스타트가 5초를 넘기면 "과도한 지연"으로 분류되고, 이는 Play Store 노출 지표에도 영향을 미친다. 그런데 많은 앱이 정작 개발자 자신도 모르는 사이에 ContentProvider를 통한 숨겨진 초기화 때문에 소중한 수백 밀리초를 낭비하고 있다. Jetpack App Startup Library는 이 문제를 정면으로 해결하는 도구다.

## ContentProvider 초기화의 숨겨진 비용

Android 앱이 콜드 스타트를 할 때, 시스템은 `Application.onCreate()`보다 **먼저** AndroidManifest에 등록된 모든 ContentProvider를 초기화한다. Firebase, WorkManager, Timber 등 많은 라이브러리가 이 타이밍을 활용해 자동 초기화를 수행해 왔다. 라이브러리 개발자 입장에서는 편리하지만, 앱 개발자 입장에서는 "왜 내 앱이 느리지?"라는 의문만 남는다.

문제의 핵심은 두 가지다.

첫째, **ContentProvider 인스턴스 생성 비용**이다. 각 ContentProvider는 독립적으로 클래스 로딩, 인스턴스 생성, `onCreate()` 실행을 수행한다. Google의 실측에 따르면 WorkManager의 ContentProvider 기반 자동 초기화는 평균 **67ms**를 추가한다. Firebase Analytics, Crashlytics 등 여러 라이브러리가 동시에 ContentProvider를 등록하면 누적 비용은 수백 ms에 달한다.

둘째, **초기화 순서를 제어할 수 없다**. A 라이브러리의 초기화가 B 라이브러리에 의존할 때, 별도의 ContentProvider로는 실행 순서를 보장하기 어렵다.

App Startup Library는 이 두 문제를 하나의 공유 ContentProvider(`InitializationProvider`)와 명시적 의존성 그래프로 해결한다.

## App Startup Library의 핵심 구조

```kotlin
// androidx.startup:startup-runtime:1.2.0
dependencies {
    implementation("androidx.startup:startup-runtime:1.2.0")
}
```

라이브러리의 핵심은 세 가지 컴포넌트로 구성된다.

| 컴포넌트 | 역할 |
|---|---|
| `Initializer<T>` | 초기화 단위 인터페이스 |
| `InitializationProvider` | 공유 ContentProvider (단 하나) |
| `AppInitializer` | 초기화 실행 엔진 |

`InitializationProvider`는 앱 전체에 단 하나만 존재하며, 매니페스트에 등록된 `Initializer` 목록을 읽어 의존성 그래프를 빌드하고 위상 정렬(topological sort)된 순서로 초기화를 실행한다.

## 실전 예제 1: 의존성 그래프를 갖는 멀티 라이브러리 초기화

실제 프로젝트에서 자주 마주치는 시나리오를 구현해 보자. Analytics 시스템이 WorkManager에 의존하고, Crashlytics가 Analytics에 의존하는 경우다.

```kotlin
// 1단계: WorkManager 초기화 (의존성 없음)
class WorkManagerInitializer : Initializer<WorkManager> {

    override fun create(context: Context): WorkManager {
        val config = Configuration.Builder()
            .setMinimumLoggingLevel(android.util.Log.INFO)
            .build()
        WorkManager.initialize(context, config)
        return WorkManager.getInstance(context)
    }

    override fun dependencies(): List<Class<out Initializer<*>>> {
        return emptyList()
    }
}

// 2단계: Analytics 초기화 (WorkManager에 의존)
class AnalyticsInitializer : Initializer<AnalyticsClient> {

    override fun create(context: Context): AnalyticsClient {
        // WorkManager가 이미 초기화되어 있음이 보장된다
        val workManager = WorkManager.getInstance(context)
        return AnalyticsClient(context, workManager)
    }

    override fun dependencies(): List<Class<out Initializer<*>>> {
        return listOf(WorkManagerInitializer::class.java)
    }
}

// 3단계: Crashlytics 초기화 (Analytics에 의존)
class CrashlyticsInitializer : Initializer<FirebaseCrashlytics> {

    override fun create(context: Context): FirebaseCrashlytics {
        val analytics = AppInitializer.getInstance(context)
            .initializeComponent(AnalyticsInitializer::class.java)
        
        FirebaseCrashlytics.getInstance().apply {
            setCrashlyticsCollectionEnabled(true)
            setCustomKey("analytics_enabled", analytics.isEnabled)
        }
        return FirebaseCrashlytics.getInstance()
    }

    override fun dependencies(): List<Class<out Initializer<*>>> {
        return listOf(AnalyticsInitializer::class.java)
    }
}
```

AndroidManifest에는 최상위 Initializer만 등록하면 된다. 의존성 체인은 자동으로 탐색된다.

```xml
<provider
    android:name="androidx.startup.InitializationProvider"
    android:authorities="${applicationId}.androidx-startup"
    android:exported="false"
    tools:node="merge">

    <!-- CrashlyticsInitializer만 등록해도
         AnalyticsInitializer → WorkManagerInitializer 순서로 자동 초기화 -->
    <meta-data
        android:name="com.example.app.CrashlyticsInitializer"
        android:value="androidx.startup" />
</provider>
```

`AppInitializer`는 내부적으로 의존성 그래프를 DFS로 탐색해 이미 초기화된 컴포넌트는 재실행하지 않는다. 동일한 `Initializer`를 여러 경로에서 참조해도 정확히 한 번만 실행된다.

## 실전 예제 2: 지연 초기화(Lazy Initialization)로 TTID 단축하기

앱 시작 시 모든 컴포넌트를 즉시 초기화할 필요는 없다. 예를 들어 Push 알림 SDK는 사용자가 로그인한 이후에만 필요하다. App Startup은 이런 경우를 위한 지연 초기화 패턴을 지원한다.

```kotlin
// 자동 초기화에서 제외할 Initializer
class PushNotificationInitializer : Initializer<PushManager> {

    override fun create(context: Context): PushManager {
        return PushManager.initialize(context).also { manager ->
            manager.registerDeviceToken()
        }
    }

    override fun dependencies(): List<Class<out Initializer<*>>> {
        return listOf(AnalyticsInitializer::class.java)
    }
}
```

```xml
<!-- 자동 초기화 비활성화: tools:node="remove"로 meta-data 제거 -->
<provider
    android:name="androidx.startup.InitializationProvider"
    android:authorities="${applicationId}.androidx-startup"
    android:exported="false"
    tools:node="merge">
    <meta-data
        android:name="com.example.app.PushNotificationInitializer"
        tools:node="remove" />
</provider>
```

로그인 성공 시점에 수동으로 초기화를 트리거한다.

```kotlin
class AuthViewModel(
    private val context: Context,
    private val authRepository: AuthRepository
) : ViewModel() {

    fun onLoginSuccess(userId: String) {
        viewModelScope.launch {
            authRepository.saveUser(userId)
            
            // 로그인 후 Push SDK 지연 초기화
            // dependencies()에 선언된 AnalyticsInitializer도 함께 처리됨
            val pushManager = AppInitializer.getInstance(context)
                .initializeComponent(PushNotificationInitializer::class.java)
            
            pushManager.setUserId(userId)
        }
    }
}
```

이 패턴의 강점은 `initializeComponent`를 호출할 때 선언된 모든 의존성(`AnalyticsInitializer`)도 자동으로 처리된다는 점이다. 이미 초기화되었다면 캐시된 결과를 반환하고, 아직 초기화되지 않았다면 순서대로 실행한다.

## 라이브러리 개발자를 위한 통합 패턴

SDK/라이브러리를 개발할 때 App Startup과 통합하면 사용자(앱 개발자)에게 훨씬 나은 경험을 제공할 수 있다.

```kotlin
// 라이브러리 내부에 Initializer 번들
class MyLibraryInitializer : Initializer<MyLibrary> {

    override fun create(context: Context): MyLibrary {
        return MyLibrary.Builder(context)
            .setApplicationId(getApplicationId(context))
            .build()
            .also { MyLibrary.instance = it }
    }

    override fun dependencies(): List<Class<out Initializer<*>>> {
        return emptyList()
    }

    private fun getApplicationId(context: Context): String {
        return context.packageManager
            .getApplicationInfo(context.packageName, PackageManager.GET_META_DATA)
            .metaData
            ?.getString("com.example.mylibrary.APP_ID")
            ?: throw IllegalStateException("APP_ID not found in AndroidManifest")
    }
}
```

라이브러리의 `AndroidManifest.xml`에 기본 등록을 포함시키고, 앱 개발자가 원한다면 `tools:node="remove"`로 비활성화할 수 있도록 문서화하는 것이 모범 사례다.

## Macrobenchmark로 초기화 성능 측정하기

App Startup의 효과를 수치로 확인하려면 Jetpack Macrobenchmark를 활용한다.

```kotlin
@RunWith(AndroidJUnit4::class)
class StartupBenchmark {

    @get:Rule
    val benchmarkRule = MacrobenchmarkRule()

    @Test
    fun coldStartup() = benchmarkRule.measureRepeated(
        packageName = "com.example.app",
        metrics = listOf(StartupTimingMetric()),
        iterations = 5,
        startupMode = StartupMode.COLD
    ) {
        pressHome()
        startActivityAndWait()
    }

    @Test
    fun coldStartupWithFullyDrawn() = benchmarkRule.measureRepeated(
        packageName = "com.example.app",
        metrics = listOf(StartupTimingMetric()),
        iterations = 5,
        startupMode = StartupMode.COLD,
        setupBlock = {
            pressHome()
        }
    ) {
        startActivityAndWait()
        // reportFullyDrawn() 호출까지 대기
        device.wait(Until.hasObject(By.res("com.example.app:id/content_loaded")), 5000)
    }
}
```

Macrobenchmark는 `timeToInitialDisplay`와 `timeToFullyDrawn` 두 지표를 제공한다. App Startup Library 도입 전후를 비교하면 ContentProvider 수에 비례하는 개선 효과를 확인할 수 있다.

## 주의사항과 실전 팁

**메인 스레드 블로킹 금지**: `Initializer.create()`는 메인 스레드에서 실행된다. 네트워크 요청, 무거운 디스크 I/O, 긴 연산을 이 메서드 안에 넣으면 ANR 위험이 생긴다. 무거운 작업은 코루틴이나 백그라운드 스레드로 위임하고, `create()`에서는 클라이언트 객체 초기화와 가벼운 설정만 수행해야 한다.

**순환 의존성 금지**: `AppInitializer`는 런타임에 순환 의존성을 감지하면 `IllegalStateException`을 던진다. A → B → A 같은 순환이 생기지 않도록 의존성 방향을 단방향으로 설계해야 한다.

**다중 프로세스 앱 주의**: App Startup 1.2.0부터 다중 `InitializationProvider`를 각 프로세스에 별도로 등록하는 것이 가능해졌다. `:push_process` 같은 별도 프로세스에서 실행되는 컴포넌트가 있다면 프로세스별로 필요한 초기화를 분리해야 한다.

**라이브러리의 기존 ContentProvider 제거**: Firebase 같은 라이브러리는 이미 App Startup으로 마이그레이션됐거나 자체 ContentProvider를 가지고 있다. `tools:node="remove"`로 충돌을 방지하고, 라이브러리 릴리즈 노트를 통해 지원 여부를 확인하는 것이 중요하다.

**초기화 실패 처리**: `create()` 내부에서 예외가 발생하면 앱이 크래시된다. 선택적 초기화(실패해도 앱이 동작 가능한 경우)는 `try-catch`로 감싸고, 실패 상태를 로깅하는 방어 코드를 추가해야 한다.

```kotlin
class OptionalAnalyticsInitializer : Initializer<AnalyticsClient?> {

    override fun create(context: Context): AnalyticsClient? {
        return try {
            AnalyticsClient.initialize(context)
        } catch (e: Exception) {
            // 초기화 실패 시 null 반환, 앱은 계속 동작
            Log.w("Startup", "Analytics initialization failed", e)
            null
        }
    }

    override fun dependencies(): List<Class<out Initializer<*>>> = emptyList()
}
```

## 마치며

App Startup Library는 단순히 "ContentProvider를 하나로 합쳐주는 도구"가 아니다. 앱 초기화 로직 전체를 그래프 형태로 선언하고, 지연 초기화를 통해 TTID(Time to Initial Display)를 줄이며, 팀 전체가 "어떤 순서로 무엇이 초기화되는가"를 명확히 추론할 수 있게 만드는 아키텍처 패턴이다. 특히 멀티 모듈 프로젝트에서 각 모듈의 초기화 책임을 명시적으로 분리할 수 있어, 의존성 관리 측면에서도 큰 이점을 제공한다.

## 참고 자료
- [App Startup - Android Developers](https://developer.android.com/topic/libraries/app-startup)
- [App startup time - Android Developers](https://developer.android.com/topic/performance/vitals/launch-time)
- [Startup Jetpack Release Notes](https://developer.android.com/jetpack/androidx/releases/startup)
- [App Startup, Part 2: Lazy Initialization - Android Developers Blog](https://medium.com/androiddevelopers/app-startup-part-2-c431e80d0df)
