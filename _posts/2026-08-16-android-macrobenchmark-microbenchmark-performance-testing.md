---
layout: post
title: "Android Benchmark 라이브러리 심화: Macrobenchmark와 Microbenchmark로 콜드 스타트부터 UI 렌더링까지 수치로 증명하는 법"
date: 2026-08-16
categories: [android, performance]
tags: [android, macrobenchmark, microbenchmark, benchmark, performance, jetpack, kotlin]
---

성능 최적화를 논할 때 가장 먼저 필요한 것은 측정이다. "느린 것 같다"는 감상보다 "콜드 스타트가 1,400ms이고, 최적화 후 980ms로 줄었다"는 수치가 훨씬 강력하다. Jetpack Benchmark 라이브러리는 바로 이 **수치 기반 성능 증명**을 위한 공식 도구다.

## 왜 Benchmark 라이브러리가 필요한가

기존에 개발자들이 성능을 측정하던 방법들은 다음과 같은 문제를 갖는다.

- **System.currentTimeMillis() / System.nanoTime()**: JVM 워밍업, GC 발생, JIT 컴파일 타이밍에 따라 결과가 천차만별이다. 첫 번째 측정값과 열 번째 측정값이 3배 이상 차이 나는 것은 흔한 일이다.
- **Android Profiler**: 디버그 빌드에서만 동작하는 경우가 많고, 프로파일링 오버헤드 자체가 앱 성능에 영향을 미친다.
- **수동 스톱워치**: 재현성이 없고, 빌드 간 비교가 불가능하다.

Jetpack Benchmark 라이브러리는 이 문제들을 해결하기 위해 다음을 제공한다.

1. **자동 워밍업(warmup)**: 안정적인 JIT 컴파일 상태를 확보한 뒤 측정을 시작한다.
2. **통계적 집계**: 여러 반복 실행의 중앙값과 편차를 자동으로 계산한다.
3. **릴리즈 빌드 강제**: 프로파일링 오버헤드 없는 실제 사용 환경에서 측정한다.
4. **CI 통합**: JSON 결과 파일을 출력해 지속적 성능 회귀 탐지에 활용할 수 있다.

---

## Macrobenchmark vs Microbenchmark

라이브러리는 두 가지 도구를 제공하며, 목적에 따라 선택해야 한다.

| 항목 | Macrobenchmark | Microbenchmark |
|------|---------------|----------------|
| 측정 대상 | 앱 전체 사용자 경험 (콜드 스타트, 스크롤, 애니메이션) | 특정 함수·코드 경로 (JSON 파싱, 정렬, DB 쿼리) |
| 실행 프로세스 | 별도 프로세스에서 앱을 제어 (out-of-process) | 동일 프로세스 내 실행 (in-process) |
| 빌드 타입 | profileable + release 빌드 | release 빌드 |
| 속도 | 반복당 수십 초~분 단위 | 반복당 수 밀리초~초 단위 |
| 주요 메트릭 | StartupTimingMetric, FrameTimingMetric, TraceSectionMetric | 실행 시간(ns), 메모리 할당 횟수 |

---

## Macrobenchmark 설정과 구현

### 1. 모듈 구조 설정

Macrobenchmark는 **별도의 테스트 모듈**로 분리해야 한다. `app/build.gradle.kts`에 profileable 설정을 추가하고, 별도 `macrobenchmark` 모듈을 생성한다.

```kotlin
// app/build.gradle.kts
android {
    buildTypes {
        create("benchmark") {
            initWith(getByName("release"))
            signingConfig = signingConfigs.getByName("debug")
            // profilingMode를 위해 profileable 설정
            proguardFiles("benchmark-rules.pro")
            isDebuggable = false
        }
    }
}
```

```kotlin
// macrobenchmark/build.gradle.kts
plugins {
    alias(libs.plugins.android.test)
    alias(libs.plugins.kotlin.android)
}

android {
    namespace = "com.example.macrobenchmark"
    targetProjectPath = ":app"
    experimentalProperties["android.experimental.self-instrumenting"] = true

    defaultConfig {
        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
    }

    buildTypes {
        create("benchmark") {
            isDebuggable = false
            signingConfig = signingConfigs.getByName("debug")
            matchingFallbacks += listOf("release")
        }
    }
}

dependencies {
    implementation(libs.junit)
    implementation(libs.androidx.test.ext.junit)
    implementation(libs.androidx.test.espresso.core)
    implementation(libs.androidx.test.uiautomator)
    implementation(libs.androidx.benchmark.macro.junit4)
    implementation(libs.androidx.profileinstaller)
}
```

### 2. 콜드 스타트 벤치마크 작성

가장 핵심적인 사용 사례인 앱 시작 시간 측정이다. `StartupTimingMetric`은 `timeToInitialDisplay`(TTID)와 `timeToFullDisplay`(TTFD)를 모두 측정한다.

```kotlin
@RunWith(AndroidJUnit4::class)
class StartupBenchmark {

    @get:Rule
    val benchmarkRule = MacrobenchmarkRule()

    /**
     * 콜드 스타트: 프로세스가 완전히 종료된 상태에서 앱을 실행.
     * 실제 사용자가 앱을 처음 켤 때와 동일한 조건.
     */
    @Test
    fun startupCold() = benchmarkRule.measureRepeated(
        packageName = "com.example.myapp",
        metrics = listOf(StartupTimingMetric()),
        compilationMode = CompilationMode.None(), // AOT 없이 인터프리터 모드
        startupMode = StartupMode.COLD,
        iterations = 10,
        setupBlock = {
            // 각 반복 전 앱을 강제 종료
            pressHome()
        }
    ) {
        startActivityAndWait()
        // reportFullyDrawn()이 앱 내에서 호출될 때까지 대기
        // (앱이 reportFullyDrawn을 호출하면 TTFD가 기록됨)
    }

    /**
     * 웜 스타트: 프로세스는 살아 있으나 Activity가 종료된 상태.
     * 사용자가 최근 앱 목록에서 앱을 다시 진입할 때와 유사한 조건.
     */
    @Test
    fun startupWarm() = benchmarkRule.measureRepeated(
        packageName = "com.example.myapp",
        metrics = listOf(StartupTimingMetric()),
        compilationMode = CompilationMode.Partial(
            baselineProfileMode = BaselineProfileMode.Require
        ),
        startupMode = StartupMode.WARM,
        iterations = 10,
        setupBlock = { pressHome() }
    ) {
        startActivityAndWait()
    }

    /**
     * 스크롤 성능 측정: FrameTimingMetric으로 프레임 드롭과 지연을 탐지.
     */
    @Test
    fun scrollListPerformance() = benchmarkRule.measureRepeated(
        packageName = "com.example.myapp",
        metrics = listOf(FrameTimingMetric()),
        compilationMode = CompilationMode.Full(),
        startupMode = StartupMode.WARM,
        iterations = 5,
        setupBlock = {
            pressHome()
            startActivityAndWait()
        }
    ) {
        // UI Automator로 리스트를 찾아 스크롤
        val device = device
        val recyclerView = device.findObject(
            By.res("com.example.myapp:id/recycler_view")
        )
        recyclerView?.let {
            it.setGestureMargin(device.displayWidth / 5)
            it.fling(Direction.DOWN)
            device.waitForIdle()
            it.fling(Direction.UP)
            device.waitForIdle()
        }
    }
}
```

### 3. 앱에 reportFullyDrawn() 추가

TTFD(Time to Full Display)를 정확히 측정하려면 앱이 완전히 준비됐을 때 신호를 보내야 한다.

```kotlin
class MainActivity : AppCompatActivity() {

    private lateinit var viewModel: MainViewModel

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        viewModel = ViewModels.of(this).get(MainViewModel::class.java)

        // 데이터 로딩 완료 후 reportFullyDrawn 호출
        viewModel.uiState.observe(this) { state ->
            if (state is UiState.Success) {
                // 모든 콘텐츠가 화면에 렌더링된 시점
                reportFullyDrawn()
            }
        }
    }
}
```

---

## Microbenchmark 설정과 구현

### 1. 의존성 추가

Microbenchmark는 기존 `androidTest` 폴더에 추가하는 방식으로 더 간단하게 설정할 수 있다.

```kotlin
// app/build.gradle.kts
android {
    defaultConfig {
        testInstrumentationRunner = "androidx.benchmark.junit4.AndroidBenchmarkRunner"
        // 또는 기존 runner를 유지하고 아래 설정 추가
        testInstrumentationRunnerArguments["androidx.benchmark.suppressErrors"] = "EMULATOR"
    }

    buildTypes {
        getByName("debug") {
            isDefault = true
        }
        create("benchmark") {
            initWith(getByName("release"))
            signingConfig = signingConfigs.getByName("debug")
        }
    }
}

dependencies {
    androidTestImplementation(libs.androidx.benchmark.junit4)
}
```

### 2. JSON 파싱 성능 측정 예제

실제 앱에서 자주 호출되는 JSON 파싱 코드의 성능을 측정하는 예제다. `BenchmarkRule`을 사용하며, `measureRepeated` 블록 안의 코드만 측정 대상이 된다.

```kotlin
@RunWith(AndroidJUnit4::class)
class JsonParsingBenchmark {

    @get:Rule
    val benchmarkRule = BenchmarkRule()

    private val context = ApplicationProvider.getApplicationContext<Context>()
    private val gson = Gson()
    private val moshi = Moshi.Builder().add(KotlinJsonAdapterFactory()).build()

    private val sampleJson = """
        {
          "users": [
            {"id":1,"name":"User1","email":"user1@example.com","score":7,"active":false},
            {"id":2,"name":"User2","email":"user2@example.com","score":14,"active":true},
            {"id":3,"name":"User3","email":"user3@example.com","score":21,"active":false}
          ]
        }
    """.trimIndent()

    /**
     * Gson을 사용한 JSON 파싱 성능.
     * BenchmarkRule.measureRepeated가 워밍업 후 안정적인 측정값을 반환.
     */
    @Test
    fun gsonParsing() {
        benchmarkRule.measureRepeated {
            gson.fromJson(sampleJson, UserListResponse::class.java)
        }
    }

    /**
     * Moshi를 사용한 JSON 파싱 성능.
     * 결과를 Gson과 비교해 어느 라이브러리가 이 데이터 구조에 더 빠른지 확인.
     */
    @Test
    fun moshiParsing() {
        val adapter = moshi.adapter(UserListResponse::class.java)
        benchmarkRule.measureRepeated {
            adapter.fromJson(sampleJson)
        }
    }

    /**
     * kotlinx.serialization을 사용한 JSON 파싱 성능.
     * 코드 생성 기반이라 리플렉션을 사용하지 않아 일반적으로 가장 빠름.
     */
    @Test
    fun kotlinSerializationParsing() {
        benchmarkRule.measureRepeated {
            Json.decodeFromString<UserListResponse>(sampleJson)
        }
    }
}

@Serializable
data class UserListResponse(
    val users: List<User>
)

@Serializable
data class User(
    val id: Int,
    val name: String,
    val email: String,
    val score: Int,
    val active: Boolean
)
```

### 3. 결과 해석

벤치마크 실행 후 Android Studio 콘솔에 다음과 같은 결과가 출력된다.

```
JsonParsingBenchmark.gsonParsing
  timeNs   median=1,821,342, min=1,743,221, max=2,104,553
  allocationCount   median=2,847

JsonParsingBenchmark.moshiParsing
  timeNs   median=1,204,881, min=1,178,443, max=1,307,229
  allocationCount   median=1,923

JsonParsingBenchmark.kotlinSerializationParsing
  timeNs   median=687,442, min=654,331, max=721,108
  allocationCount   median=1,204
```

이 결과에서 kotlinx.serialization이 Gson보다 약 2.6배 빠르고 메모리 할당도 절반 이하임을 수치로 확인할 수 있다.

---

## CI 통합: 성능 회귀 자동 탐지

벤치마크를 한 번 실행하는 것만으로는 부족하다. **PR마다 성능 회귀 여부를 자동으로 탐지**하는 것이 핵심이다.

```yaml
# .github/workflows/benchmark.yml
name: Performance Benchmarks

on:
  pull_request:
    branches: [main]

jobs:
  benchmark:
    runs-on: self-hosted  # 실제 Android 기기가 연결된 러너 필요
    steps:
      - uses: actions/checkout@v4

      - name: Run Microbenchmarks
        run: |
          ./gradlew :app:connectedBenchmarkAndroidTest \
            -Pandroid.testInstrumentationRunnerArguments.class=com.example.JsonParsingBenchmark

      - name: Run Macrobenchmarks
        run: |
          ./gradlew :macrobenchmark:connectedBenchmarkAndroidTest

      - name: Upload benchmark results
        uses: actions/upload-artifact@v4
        with:
          name: benchmark-results
          path: |
            **/build/outputs/connected_android_test_additional_output/**/*.json

      - name: Compare with baseline
        uses: benchmark-action/github-action-benchmark@v1
        with:
          tool: 'customBiggerIsBetter'
          output-file-path: benchmark-results.json
          alert-threshold: '150%'  # 기준값 대비 50% 이상 악화 시 알림
          fail-on-alert: true
          github-token: ${{ secrets.GITHUB_TOKEN }}
          comment-on-alert: true
```

---

## TraceSectionMetric으로 앱 내부 측정

Macrobenchmark에서 앱의 특정 코드 구간만 집중적으로 측정하고 싶을 때 `TraceSectionMetric`과 `Trace.beginSection()` / `Trace.endSection()`을 조합한다.

```kotlin
// 앱 코드에 트레이스 마커 추가
class DataRepository {
    suspend fun loadDashboardData(): DashboardData {
        return withContext(Dispatchers.IO) {
            Trace.beginSection("LoadDashboardData") // 측정 시작 마커
            try {
                val users = userDao.getAllUsers()
                val posts = postApi.getRecentPosts()
                DashboardData(users, posts)
            } finally {
                Trace.endSection() // 측정 종료 마커
            }
        }
    }
}

// Macrobenchmark 테스트에서 TraceSectionMetric 사용
@Test
fun dashboardLoadingTime() = benchmarkRule.measureRepeated(
    packageName = "com.example.myapp",
    metrics = listOf(
        TraceSectionMetric("LoadDashboardData"), // 마커 구간만 측정
        FrameTimingMetric()
    ),
    compilationMode = CompilationMode.Full(),
    startupMode = StartupMode.WARM,
    iterations = 5,
    setupBlock = { pressHome() }
) {
    startActivityAndWait()
    // 대시보드 화면으로 이동하는 UI Automator 코드
    device.findObject(By.text("Dashboard"))?.click()
    device.waitForIdle(3_000)
}
```

---

## 주의사항과 실전 팁

### 에뮬레이터에서 실행하지 않기

Macrobenchmark는 에뮬레이터에서 실행 자체가 제한되며, Microbenchmark는 실행은 되지만 결과를 신뢰할 수 없다. 반드시 **실제 Android 기기**에서 실행해야 한다. 빌드 설정에 다음을 추가해 경고를 오류로 전환할 수 있다.

```kotlin
// androidTest용 build.gradle.kts
android {
    defaultConfig {
        // EMULATOR 경고를 오류로 승격 (CI에서 실수 방지)
        testInstrumentationRunnerArguments[
            "androidx.benchmark.suppressErrors"
        ] = ""
    }
}
```

### 클럭 잠금(Lock Clocks)으로 일관성 확보

기기의 CPU 동적 주파수 조정(DVFS)은 측정 결과의 분산을 키운다. 루팅된 기기에서는 클럭을 고정할 수 있다.

```bash
./gradlew lockClocks    # 클럭 고정
./gradlew connectedCheck # 벤치마크 실행
./gradlew unlockClocks  # 클럭 해제
```

### CompilationMode 선택 전략

- `CompilationMode.None()`: Baseline Profile 적용 이전의 최악 시나리오를 측정. 최적화 효과 비교 기준선으로 사용.
- `CompilationMode.Partial(BaselineProfileMode.Require)`: Baseline Profile이 실제로 적용된 상태를 측정. 사용자가 앱을 설치 후 처음 실행하는 것과 동일한 조건.
- `CompilationMode.Full()`: 완전한 AOT 컴파일 상태. 이론적 최대 성능의 상한선을 확인할 때 사용.

### allocationCount를 간과하지 않기

Microbenchmark는 실행 시간 외에도 **메모리 할당 횟수(allocationCount)**를 측정한다. GC 압박이 크면 결국 프레임 드롭으로 이어지므로, 핫 경로의 할당을 최소화하는 것이 중요하다. 특히 RecyclerView의 `onBindViewHolder`, Compose의 Composable 재구성(recomposition) 과정에서의 할당을 집중적으로 확인하자.

### 반복 횟수(iterations) 설정

- Microbenchmark: 라이브러리가 자동으로 워밍업 후 최적 반복 횟수를 결정하므로 별도 설정 불필요.
- Macrobenchmark: 앱 스타트업은 최소 10회, 스크롤 성능은 5회 이상 권장. 기기 발열에 의한 스로틀링 방지를 위해 30회를 초과하지 않는 것이 좋다.

---

## 정리

| 시나리오 | 추천 도구 | 핵심 메트릭 |
|---------|----------|------------|
| 앱 시작 시간 최적화 | Macrobenchmark | StartupTimingMetric (TTID / TTFD) |
| 스크롤 60fps 달성 | Macrobenchmark | FrameTimingMetric |
| JSON 파서 비교 | Microbenchmark | timeNs median |
| DB 쿼리 최적화 | Microbenchmark | timeNs median + allocationCount |
| Baseline Profile 효과 검증 | Macrobenchmark | StartupTimingMetric (None vs Partial 비교) |
| CI 성능 회귀 방지 | 양쪽 모두 | JSON 결과 파일 + 임계값 알림 |

Benchmark 라이브러리를 CI 파이프라인에 통합하면 성능 최적화가 코드 품질과 동일한 수준의 릴리즈 게이팅 조건이 된다. "빠르다"는 직관에서 "1,400ms에서 980ms로 30% 단축했다"는 근거로 넘어가는 순간, 팀의 성능 문화가 달라진다.

## 참고 자료
- [Android Benchmark Overview - Android Developers](https://developer.android.com/topic/performance/benchmarking/benchmarking-overview)
- [Write a Macrobenchmark - Android Developers](https://developer.android.com/topic/performance/benchmarking/macrobenchmark-overview)
- [Microbenchmark Overview - Android Developers](https://developer.android.com/topic/performance/benchmarking/microbenchmark-overview)
