---
layout: post
title: "Android Baseline Profiles 심화: 콜드 스타트를 30% 단축하는 AOT 최적화 전략"
date: 2026-06-22
categories: [android]
tags: [android, baseline-profiles, macrobenchmark, app-startup, performance, aot, art]
---

## 왜 첫 실행은 느린가?

Android 앱을 처음 설치하거나 업데이트한 직후, 사용자는 유독 느린 실행 속도를 경험합니다. 이는 Android Runtime(ART)이 앱 코드를 어떻게 처리하는지와 직결된 문제입니다. 앱이 처음 실행될 때 ART는 Dalvik Bytecode를 해석(interpret)하거나, Just-In-Time(JIT) 컴파일로 실시간 최적화합니다. 이 과정 자체가 CPU와 시간을 소모하기 때문에 첫 실행이 느려집니다.

기존에는 몇 번 앱을 사용하고 나면 ART가 실행 패턴을 학습해 JIT 컴파일 결과를 프로파일 파일(`*.prof`)에 저장하고, 이를 기반으로 Ahead-of-Time(AOT) 컴파일을 수행해 이후 실행 속도를 높였습니다. 그러나 이 학습 과정에는 수십~수백 번의 실행이 필요하므로, 신규 사용자나 앱 업데이트 직후 사용자는 항상 느린 첫 경험을 겪게 됩니다.

**Baseline Profiles**은 이 문제를 근본적으로 해결합니다. 개발자가 직접 "자주 사용되는 코드 경로"를 정의한 프로파일 파일을 앱에 동봉하면, ART는 설치 즉시 해당 코드를 AOT로 사전 컴파일합니다. Google에 따르면 이를 통해 **첫 실행 시 코드 실행 속도가 최대 30% 향상**됩니다.

---

## ART 컴파일 전략 이해하기

Android Runtime은 앱 실행 시 세 가지 방식으로 코드를 처리합니다.

| 방식 | 설명 | 초기 실행 속도 |
|------|------|--------------|
| Interpretation | 바이트코드를 직접 해석 실행 | 느림 |
| JIT Compilation | 실행 중 핫 코드를 기계어로 컴파일 | 중간 |
| AOT Compilation | 설치/업데이트 시 미리 기계어로 컴파일 | 빠름 |

Baseline Profile은 AOT Compilation의 혜택을 **첫 실행부터** 받을 수 있게 해주는 Profile Guided Optimization(PGO)의 한 형태입니다.

### Baseline Profile vs Startup Profile

두 프로파일을 명확히 구분해야 합니다.

- **Startup Profile**: 앱 콜드 스타트에 필요한 코드 경로만 포함. `baseline-prof.txt`에 별도로 정의해 DEX 레이아웃 최적화에 사용됩니다.
- **Baseline Profile**: Startup Profile의 상위 집합. 콜드 스타트 외에도 주요 사용자 여정(스크롤, 화면 전환 등)의 코드 경로를 포함합니다.

실제 배포 APK/AAB에는 두 파일 모두 포함시키는 것이 권장됩니다.

---

## 왜 Baseline Profile이 비즈니스에 중요한가?

Google I/O 자료와 여러 실측 연구에 따르면 앱 콜드 스타트 시간은 사용자 이탈률에 직결됩니다.

- 첫 실행 속도가 1초 느려지면 사용자 이탈률 약 20% 증가
- Now in Android 샘플 앱에서 Baseline Profile 적용 후 콜드 스타트 **324ms → 229ms** (약 30% 단축)
- Google Play 핵심 지표(Android Vitals)에 포함되어 앱 검색 노출 순위에도 영향

특히 Kotlin과 Jetpack Compose를 사용하는 현대 앱에서 효과가 큽니다. Compose 자체 라이브러리도 Baseline Profile을 내장하고 있으며, 앱 빌드 시 이를 병합해 자동으로 혜택을 받습니다.

---

## 프로젝트 설정

Android Studio Iguana 이상과 AGP 8.2 이상에서는 **Baseline Profile 모듈 템플릿**을 제공합니다. 수동으로 설정할 경우 두 개의 모듈이 필요합니다.

**`app/build.gradle.kts`**:

```kotlin
plugins {
    id("com.android.application")
    id("androidx.baselineprofile")
}

dependencies {
    implementation("androidx.profileinstaller:profileinstaller:1.3.1")
    "baselineProfile"(project(":baselineprofile"))
}

baselineProfile {
    automaticGenerationDuringBuild = false // CI에서만 생성하려면 false
    saveInSrc = true
}
```

**`baselineprofile/build.gradle.kts`**:

```kotlin
plugins {
    id("com.android.test")
    id("androidx.baselineprofile")
}

android {
    targetProjectPath = ":app"

    defaultConfig {
        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
    }
}

dependencies {
    implementation("androidx.test.ext:junit:1.1.5")
    implementation("androidx.test.uiautomator:uiautomator:2.3.0")
    implementation("androidx.benchmark:benchmark-macro-junit4:1.2.3")
}

baselineProfile {
    useConnectedDevices = true
}
```

---

## 실제 구현 예제

### 코드 예제 1 — BaselineProfileGenerator

Baseline Profile 생성 모듈에서 주요 사용자 여정을 정의합니다. `BaselineProfileRule`은 지정된 시나리오를 실행하면서 ART 프로파일 데이터를 수집하고 `baseline-prof.txt`를 자동 생성합니다.

```kotlin
// baselineprofile/src/main/kotlin/com/example/baselineprofile/BaselineProfileGenerator.kt

@OptIn(ExperimentalBaselineProfilesApi::class)
class BaselineProfileGenerator {

    @get:Rule
    val baselineProfileRule = BaselineProfileRule()

    @Test
    fun generateBaselineProfile() = baselineProfileRule.collect(
        packageName = "com.example.myapp",
        profileBlock = {
            // 1. 앱 콜드 스타트 시뮬레이션
            startActivityAndWait()

            // 2. 콘텐츠가 완전히 로드될 때까지 대기
            device.wait(
                Until.hasObject(By.res("com.example.myapp:id/content_list")),
                5_000L
            )

            // 3. 피드 스크롤 시나리오 — 주요 사용자 여정
            device.findObject(By.res("com.example.myapp:id/content_list"))
                ?.let { list ->
                    repeat(5) {
                        list.fling(Direction.DOWN)
                        Thread.sleep(500)
                    }
                }

            // 4. 상세 화면 진입 시나리오
            device.findObject(By.res("com.example.myapp:id/item_card"))?.click()
            device.wait(
                Until.hasObject(By.res("com.example.myapp:id/detail_content")),
                3_000L
            )
        }
    )
}
```

테스트 실행 후 생성된 `app/src/main/baseline-prof.txt` 예시입니다.

```
HSPLcom/example/myapp/MainActivity;->onCreate(Landroid/os/Bundle;)V
HSPLcom/example/myapp/ui/home/HomeScreen;->Content(Landroidx/compose/runtime/Composer;I)V
HSPLcom/example/myapp/data/repository/FeedRepository;->getFeeds()Lkotlinx/coroutines/flow/Flow;
Lcom/example/myapp/ui/theme/AppTheme;
```

각 prefix의 의미는 다음과 같습니다.

- `H`: 핫 메서드 (자주 실행됨)
- `S`: 스타트업 경로에 포함
- `P`: Post-startup (스타트업 직후 실행됨)
- `L`: 클래스 단위 초기화 최적화

---

### 코드 예제 2 — Macrobenchmark로 성능 측정

Baseline Profile 적용 전/후를 정량적으로 비교하려면 `Macrobenchmark` 라이브러리를 사용합니다. `CompilationMode`를 달리해 세 가지 시나리오를 측정합니다.

```kotlin
// macrobenchmark/src/main/kotlin/com/example/benchmark/StartupBenchmark.kt

@RunWith(AndroidJUnit4::class)
class StartupBenchmark {

    @get:Rule
    val benchmarkRule = MacrobenchmarkRule()

    // 케이스 1: AOT 컴파일 없음 — 첫 설치 직후와 동일한 최악의 조건
    @Test
    fun startupWithNoCompilation() = benchmarkRule.measureRepeated(
        packageName = "com.example.myapp",
        metrics = listOf(StartupTimingMetric()),
        compilationMode = CompilationMode.None(),
        iterations = 10,
        startupMode = StartupMode.COLD
    ) {
        pressHome()
        startActivityAndWait()
    }

    // 케이스 2: Baseline Profile 적용 — 권장 운영 조건
    @Test
    fun startupWithBaselineProfile() = benchmarkRule.measureRepeated(
        packageName = "com.example.myapp",
        metrics = listOf(StartupTimingMetric()),
        compilationMode = CompilationMode.Partial(
            baselineProfileMode = BaselineProfileMode.Require
        ),
        iterations = 10,
        startupMode = StartupMode.COLD
    ) {
        pressHome()
        startActivityAndWait()
    }

    // 케이스 3: 완전한 AOT 컴파일 — 이론적 최대치
    @Test
    fun startupWithFullCompilation() = benchmarkRule.measureRepeated(
        packageName = "com.example.myapp",
        metrics = listOf(
            StartupTimingMetric(),
            FrameTimingMetric() // 프레임 드롭도 함께 측정
        ),
        compilationMode = CompilationMode.Full(),
        iterations = 10,
        startupMode = StartupMode.COLD
    ) {
        pressHome()
        startActivityAndWait()
        device.findObject(By.res("com.example.myapp:id/content_list"))
            ?.fling(Direction.DOWN)
    }
}
```

실제 측정 결과 예시 (Now in Android 앱 기준):

```
startupWithNoCompilation
  timeToInitialDisplayMs   min=312.8   median=324.8   max=341.2

startupWithBaselineProfile
  timeToInitialDisplayMs   min=221.3   median=229.0   max=238.6   ← 약 30% 단축

startupWithFullCompilation
  timeToInitialDisplayMs   min=198.1   median=205.4   max=213.7
```

`CompilationMode.Partial(BaselineProfileMode.Require)`는 Baseline Profile이 없으면 테스트가 실패하도록 강제해 CI에서 프로파일 누락을 방지할 수 있습니다.

---

## App Startup 라이브러리와의 연계

Baseline Profile과 함께 **Jetpack App Startup** 라이브러리를 사용하면 초기화 비용도 줄일 수 있습니다. 여러 컴포넌트의 초기화를 단일 `ContentProvider`로 통합해 ContentProvider 오버헤드를 제거합니다.

`Initializer<T>` 인터페이스를 구현해 초기화 컴포넌트를 정의합니다.

```kotlin
class AnalyticsInitializer : Initializer<FirebaseApp> {
    override fun create(context: Context): FirebaseApp {
        return FirebaseApp.initializeApp(context)!!
    }
    // 이 초기화기가 의존하는 다른 초기화기 목록
    override fun dependencies(): List<Class<out Initializer<*>>> = emptyList()
}

class DatabaseInitializer : Initializer<Unit> {
    override fun create(context: Context) {
        // Room DB 사전 워밍업
        AppDatabase.getInstance(context)
    }
    // Analytics 초기화 후 실행
    override fun dependencies() = listOf(AnalyticsInitializer::class.java)
}
```

`AndroidManifest.xml`에서 `InitializationProvider`에 등록합니다.

```xml
<provider
    android:name="androidx.startup.InitializationProvider"
    android:authorities="${applicationId}.androidx-startup"
    android:exported="false"
    tools:node="merge">
    <meta-data
        android:name="com.example.AnalyticsInitializer"
        android:value="androidx.startup" />
</provider>
```

스타트업 시점에 꼭 필요하지 않은 초기화는 **Lazy Initialization**으로 미루면 TTID(Time to Initial Display)를 더 줄일 수 있습니다.

```kotlin
// 필요한 시점에 수동으로 초기화
AppInitializer.getInstance(context)
    .initializeComponent(DatabaseInitializer::class.java)
```

---

## 주의사항 및 팁

### 실제 기기에서만 측정하라

Macrobenchmark는 반드시 **에뮬레이터가 아닌 실제 물리 기기**에서 실행해야 합니다. 에뮬레이터는 AOT 컴파일 동작이 다르고 결과가 왜곡됩니다. 아래 설정으로 에뮬레이터 실행 시 오류를 발생시켜 CI 실수를 방지합니다.

```kotlin
// macrobenchmark/build.gradle.kts
android {
    defaultConfig {
        testInstrumentationRunnerArguments["androidx.benchmark.suppressErrors"] = "EMULATOR"
    }
}
```

### `profileinstaller` 의존성은 필수

기기에 Profile Installer가 없으면 Baseline Profile이 적용되지 않습니다. `implementation("androidx.profileinstaller:profileinstaller:1.3.1")`을 앱 모듈에 반드시 추가하세요. 이 라이브러리가 앱 설치 후 백그라운드에서 프로파일을 ART에 전달합니다.

### 릴리즈 빌드에서만 효과가 있다

Baseline Profile은 **릴리즈 빌드**에서만 활성화됩니다. 디버그 빌드에서는 ProGuard/R8가 비활성화되어 있고 AOT 최적화도 적용되지 않습니다. 성능 측정은 항상 `release` 빌드 타입으로 진행하세요.

### CI/CD에서 자동화하기

AGP 8.2 이상에서는 매 릴리즈 빌드 시 Baseline Profile을 자동 생성하도록 설정할 수 있습니다.

```kotlin
// app/build.gradle.kts
baselineProfile {
    automaticGenerationDuringBuild = true
    saveInSrc = true
    mergeIntoMain = true // 모든 빌드 타입에 동일 프로파일 적용
}
```

### 라이브러리 Baseline Profile 활용

Jetpack Compose, Room, Retrofit 등 주요 라이브러리들은 이미 자체 Baseline Profile을 내장하고 있습니다. AGP는 이들을 자동으로 앱 프로파일에 병합합니다. 별도 설정 없이도 라이브러리 레벨의 최적화 혜택을 받을 수 있다는 점을 기억하세요.

### 주기적으로 프로파일 갱신하라

앱의 코드베이스가 크게 바뀌면 기존 Baseline Profile이 낡아집니다. 새로운 주요 기능을 추가하거나 릴리즈 마일스톤마다 `generateBaselineProfile` 태스크를 재실행해 프로파일을 최신 상태로 유지하세요.

---

## 참고 자료

- [Baseline Profiles overview — Android Developers](https://developer.android.com/topic/performance/baselineprofiles/overview)
- [Benchmark Baseline Profiles with Macrobenchmark — Android Developers](https://developer.android.com/topic/performance/baselineprofiles/measure-baselineprofile)
- [App Startup library — Android Developers](https://developer.android.com/topic/libraries/app-startup)
