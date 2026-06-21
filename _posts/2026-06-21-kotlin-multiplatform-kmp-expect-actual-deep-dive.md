---
layout: post
title: "Kotlin Multiplatform(KMP) 심화: expect/actual, 공유 ViewModel, Ktor+SQLDelight로 완전한 멀티플랫폼 앱 구축하기"
date: 2026-06-21
categories: [android, flutter]
tags: [kotlin, kmp, kotlin-multiplatform, expect-actual, ktor, sqldelight, compose-multiplatform, ios, android]
---

Kotlin Multiplatform(KMP)은 JetBrains와 Google이 공동으로 지원하는 코드 공유 기술입니다. "한 번 작성하고, 어디서나 실행하라(Write Once, Run Anywhere)"는 오래된 꿈을 현실적인 방식으로 구현하되, 각 플랫폼의 네이티브 강점은 그대로 유지합니다. 이 글에서는 KMP의 핵심 메커니즘인 `expect/actual`부터 시작해, Ktor와 SQLDelight를 활용한 실전 아키텍처까지 깊이 있게 살펴봅니다.

## KMP란 무엇이며, 왜 필요한가?

Android 개발자라면 한 번쯤 이런 상황을 겪어봤을 겁니다. iOS 팀과 동일한 비즈니스 로직을 각자 Swift와 Kotlin으로 따로 구현하다가, 한쪽에서 버그를 수정해도 다른 쪽에 반영을 잊어 불일치가 생기는 경우입니다. KMP는 이 문제를 근본적으로 해결합니다.

### Flutter와의 차이점

Flutter가 모든 UI를 자체 렌더링 엔진(Skia/Impeller)으로 그리는 방식이라면, KMP는 **비즈니스 로직만 공유**하고 UI는 각 플랫폼의 네이티브 방식(Jetpack Compose, SwiftUI)을 사용하는 접근입니다. Compose Multiplatform(CMP)을 함께 사용하면 UI도 공유할 수 있지만, 그것도 "강제"가 아닌 "선택"입니다.

```
┌───────────────────────────────────────┐
│           Shared (commonMain)          │
│  - 비즈니스 로직                        │
│  - Repository, UseCase                 │
│  - ViewModel (AAC ViewModel 공유)       │
│  - Ktor HTTP 클라이언트                 │
│  - SQLDelight DB                       │
└──────────────┬────────────────────────┘
               │
    ┌──────────┴──────────┐
    ▼                     ▼
┌────────────┐      ┌────────────┐
│ androidMain│      │  iosMain   │
│ (Compose)  │      │ (SwiftUI)  │
└────────────┘      └────────────┘
```

2026년 현재, Duolingo, Google Docs, Forbes, McDonald's 등 수십 개의 대형 앱이 KMP를 프로덕션에서 사용하고 있으며, Google은 ViewModel, Room, DataStore, Navigation, Paging 등 주요 Jetpack 라이브러리를 KMP에서 공식 지원합니다.

---

## expect/actual: KMP의 핵심 메커니즘

KMP에서 플랫폼별 구현을 다르게 제공해야 할 때 사용하는 언어 키워드가 바로 `expect`와 `actual`입니다.

- **`expect`**: `commonMain`에서 "이 기능이 존재한다"는 계약(contract)을 선언
- **`actual`**: `androidMain`, `iosMain` 등 각 플랫폼 소스셋에서 실제 구현 제공

### 예제 1: 플랫폼별 날짜 포맷터 구현

날짜를 포맷하는 로직은 플랫폼마다 기본 API가 다릅니다. `expect/actual`로 공통 인터페이스를 제공해봅시다.

```kotlin
// commonMain/kotlin/util/DateFormatter.kt
expect class DateFormatter() {
    fun format(timestamp: Long, pattern: String = "yyyy-MM-dd"): String
}
```

```kotlin
// androidMain/kotlin/util/DateFormatter.android.kt
import java.text.SimpleDateFormat
import java.util.Date
import java.util.Locale

actual class DateFormatter actual constructor() {
    actual fun format(timestamp: Long, pattern: String): String {
        val sdf = SimpleDateFormat(pattern, Locale.getDefault())
        return sdf.format(Date(timestamp))
    }
}
```

```kotlin
// iosMain/kotlin/util/DateFormatter.ios.kt
import platform.Foundation.NSDate
import platform.Foundation.NSDateFormatter
import platform.Foundation.dateWithTimeIntervalSince1970

actual class DateFormatter actual constructor() {
    actual fun format(timestamp: Long, pattern: String): String {
        val formatter = NSDateFormatter()
        formatter.dateFormat = pattern
        val date = NSDate.dateWithTimeIntervalSince1970(timestamp / 1000.0)
        return formatter.stringFromDate(date)
    }
}
```

`commonMain`의 코드는 `DateFormatter`가 어떻게 구현되는지 알 필요 없이, 그냥 `DateFormatter().format(timestamp)`를 호출하면 됩니다. 컴파일 시점에 각 플랫폼의 `actual` 구현체로 자동으로 연결됩니다.

### expect/actual의 다양한 활용 패턴

`expect/actual`은 클래스뿐만 아니라 함수, 오브젝트, 타입 별칭에도 사용할 수 있습니다.

```kotlin
// commonMain: 플랫폼별 로거
expect fun log(tag: String, message: String)

// androidMain
import android.util.Log
actual fun log(tag: String, message: String) {
    Log.d(tag, message)
}

// iosMain
actual fun log(tag: String, message: String) {
    println("[$tag] $message")
}
```

---

## 프로젝트 구조: Gradle 멀티모듈 설정

KMP 프로젝트의 `build.gradle.kts`를 제대로 이해하는 것이 중요합니다.

```kotlin
// shared/build.gradle.kts
plugins {
    alias(libs.plugins.kotlinMultiplatform)
    alias(libs.plugins.androidLibrary)
    alias(libs.plugins.kotlinxSerialization)
    alias(libs.plugins.sqldelight)
}

kotlin {
    androidTarget {
        compilations.all {
            kotlinOptions { jvmTarget = "17" }
        }
    }

    listOf(
        iosX64(),
        iosArm64(),
        iosSimulatorArm64()
    ).forEach { iosTarget ->
        iosTarget.binaries.framework {
            baseName = "Shared"
            isStatic = true
        }
    }

    sourceSets {
        commonMain.dependencies {
            // Ktor
            implementation(libs.ktor.client.core)
            implementation(libs.ktor.client.content.negotiation)
            implementation(libs.ktor.serialization.kotlinx.json)
            // Coroutines
            implementation(libs.kotlinx.coroutines.core)
            // SQLDelight
            implementation(libs.sqldelight.runtime)
            implementation(libs.sqldelight.coroutines.extensions)
        }

        androidMain.dependencies {
            implementation(libs.ktor.client.okhttp)
            implementation(libs.sqldelight.android.driver)
        }

        iosMain.dependencies {
            implementation(libs.ktor.client.darwin)
            implementation(libs.sqldelight.native.driver)
        }
    }
}
```

`sourceSets` 블록에서 `commonMain`, `androidMain`, `iosMain`으로 소스셋을 분리하는 것이 핵심입니다. Ktor 클라이언트 엔진이나 SQLDelight 드라이버처럼 플랫폼에 따라 다른 의존성은 각 플랫폼 소스셋에만 추가합니다.

---

## 실전 예제: Ktor + SQLDelight로 네트워크 캐싱 구현

이제 실제 앱에서 자주 쓰이는 패턴을 구현해봅시다. SpaceX 발사 목록을 가져와 로컬 DB에 캐싱하는 Repository입니다.

### Step 1: SQLDelight 스키마 정의

```sql
-- commonMain/sqldelight/com/example/db/Launch.sq
CREATE TABLE LaunchEntity (
    id TEXT NOT NULL PRIMARY KEY,
    missionName TEXT NOT NULL,
    launchDateUtc TEXT NOT NULL,
    rocketName TEXT NOT NULL,
    isSuccess INTEGER AS Boolean,
    details TEXT,
    missionPatchUrl TEXT
);

getLaunches:
SELECT * FROM LaunchEntity ORDER BY launchDateUtc DESC;

insertLaunch:
INSERT OR REPLACE INTO LaunchEntity(
    id, missionName, launchDateUtc, rocketName, isSuccess, details, missionPatchUrl
) VALUES (?, ?, ?, ?, ?, ?, ?);

clearAll:
DELETE FROM LaunchEntity;
```

### Step 2: 플랫폼별 DB 드라이버 팩토리 (expect/actual)

```kotlin
// commonMain
expect class DatabaseDriverFactory {
    fun createDriver(): SqlDriver
}

// androidMain
import android.content.Context
import app.cash.sqldelight.db.SqlDriver
import app.cash.sqldelight.driver.android.AndroidSqliteDriver
import com.example.db.AppDatabase

actual class DatabaseDriverFactory(private val context: Context) {
    actual fun createDriver(): SqlDriver {
        return AndroidSqliteDriver(AppDatabase.Schema, context, "spacex.db")
    }
}

// iosMain
import app.cash.sqldelight.db.SqlDriver
import app.cash.sqldelight.driver.native.NativeSqliteDriver
import com.example.db.AppDatabase

actual class DatabaseDriverFactory {
    actual fun createDriver(): SqlDriver {
        return NativeSqliteDriver(AppDatabase.Schema, "spacex.db")
    }
}
```

### Step 3: Ktor HTTP 클라이언트 설정

```kotlin
// commonMain/kotlin/network/SpaceXApi.kt
import io.ktor.client.*
import io.ktor.client.call.*
import io.ktor.client.plugins.contentnegotiation.*
import io.ktor.client.request.*
import io.ktor.serialization.kotlinx.json.*
import kotlinx.serialization.Serializable
import kotlinx.serialization.json.Json

@Serializable
data class RocketLaunch(
    val id: String,
    val name: String,
    val date_utc: String,
    val success: Boolean? = null,
    val details: String? = null,
    val links: Links? = null
)

@Serializable
data class Links(val patch: Patch? = null)

@Serializable
data class Patch(val small: String? = null)

class SpaceXApi {
    private val client = HttpClient {
        install(ContentNegotiation) {
            json(Json {
                ignoreUnknownKeys = true
                isLenient = true
            })
        }
    }

    suspend fun getLaunches(): List<RocketLaunch> =
        client.get("https://api.spacexdata.com/v5/launches").body()
}
```

### Step 4: 예제 2 — Repository (캐시-우선 전략)

```kotlin
// commonMain/kotlin/repository/LaunchRepository.kt
import app.cash.sqldelight.coroutines.asFlow
import app.cash.sqldelight.coroutines.mapToList
import com.example.db.AppDatabase
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.catch
import kotlinx.coroutines.flow.flow

class LaunchRepository(
    driverFactory: DatabaseDriverFactory,
    private val api: SpaceXApi = SpaceXApi()
) {
    private val db = AppDatabase(driverFactory.createDriver())
    private val queries = db.launchEntityQueries

    // DB의 데이터를 Flow로 실시간 관찰
    fun observeLaunches(): Flow<List<LaunchEntity>> =
        queries.getLaunches()
            .asFlow()
            .mapToList(Dispatchers.Default)

    // 네트워크에서 새 데이터를 가져와 DB에 저장 (cache-first 전략)
    suspend fun refreshLaunches(): Result<Unit> = runCatching {
        val launches = api.getLaunches()
        db.transaction {
            queries.clearAll()
            launches.forEach { launch ->
                queries.insertLaunch(
                    id = launch.id,
                    missionName = launch.name,
                    launchDateUtc = launch.date_utc,
                    rocketName = "Falcon 9",
                    isSuccess = launch.success,
                    details = launch.details,
                    missionPatchUrl = launch.links?.patch?.small
                )
            }
        }
    }
}
```

이 Repository는 완전히 `commonMain`에 위치하므로 Android와 iOS 모두에서 동일한 코드를 공유합니다. Android ViewModel에서든, iOS의 SwiftUI ViewModel에서든 `LaunchRepository`를 그대로 사용할 수 있습니다.

---

## Jetpack ViewModel을 KMP에서 공유하기

2024년 이후 Google이 공식 지원을 확장하면서 `androidx.lifecycle:lifecycle-viewmodel`이 KMP 호환 버전으로 제공됩니다.

```kotlin
// commonMain/kotlin/viewmodel/LaunchViewModel.kt
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.launchIn
import kotlinx.coroutines.flow.onEach
import kotlinx.coroutines.launch

data class LaunchUiState(
    val launches: List<LaunchEntity> = emptyList(),
    val isLoading: Boolean = false,
    val error: String? = null
)

class LaunchViewModel(
    private val repository: LaunchRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow(LaunchUiState())
    val uiState: StateFlow<LaunchUiState> = _uiState.asStateFlow()

    init {
        repository.observeLaunches()
            .onEach { launches ->
                _uiState.value = _uiState.value.copy(launches = launches)
            }
            .launchIn(viewModelScope)

        refresh()
    }

    fun refresh() {
        viewModelScope.launch {
            _uiState.value = _uiState.value.copy(isLoading = true, error = null)
            repository.refreshLaunches()
                .onFailure { e ->
                    _uiState.value = _uiState.value.copy(error = e.message)
                }
            _uiState.value = _uiState.value.copy(isLoading = false)
        }
    }
}
```

이 ViewModel 코드 역시 100% `commonMain`에 위치합니다. Android에서는 `viewModel()` 컴포저블로 인스턴스를 가져오고, iOS에서는 Kotlin/Native 브릿지를 통해 Swift 코드에서 접근합니다.

---

## 주의사항과 실전 팁

### 1. Kotlin/Native의 동시성 제약

iOS(Kotlin/Native)는 멀티스레드 접근에 엄격한 규칙이 있습니다. Kotlin 1.9.20 이후 New Memory Manager가 기본값이 되어 대부분의 제약이 해소됐지만, `@ThreadLocal`이나 `@SharedImmutable`과 같은 어노테이션이 필요한 경우가 여전히 있습니다.

```kotlin
// iOS에서 싱글톤 객체를 스레드 안전하게 사용할 때
@ThreadLocal
object Logger {
    var level: Int = 0
}
```

### 2. Coroutines Dispatchers 주의

iOS에서는 `Dispatchers.Main`을 사용하려면 `kotlinx-coroutines-core`에 `native-mt` 버전이 필요했지만, Kotlin 1.9 이후 통합됐습니다. 단, iOS에서 백그라운드 작업 시 `Dispatchers.Default`는 제한적으로 사용하고, `Dispatchers.IO`는 기본 지원되지 않으므로 `withContext(Dispatchers.Default)`로 대체합니다.

### 3. `actual typealias`로 기존 라이브러리 재사용

플랫폼 전용 클래스를 그대로 노출하고 싶을 때는 `actual typealias`를 활용합니다.

```kotlin
// commonMain
expect class AtomicInt(initialValue: Int) {
    fun get(): Int
    fun set(value: Int)
    fun incrementAndGet(): Int
}

// androidMain (java.util.concurrent.atomic 활용)
actual typealias AtomicInt = java.util.concurrent.atomic.AtomicInteger
```

### 4. iOS 프레임워크 크기 최적화

KMP 프레임워크는 기본 설정에서 상당히 커질 수 있습니다. `isStatic = true`로 설정하고, 불필요한 의존성을 정리하면 프레임워크 바이너리를 크게 줄일 수 있습니다. Gradle에서 `freeCompilerArgs += "-Xbinary=bundleId=..."`를 통해 세부 최적화 옵션을 적용할 수도 있습니다.

### 5. 점진적 도입 전략

KMP의 가장 큰 장점 중 하나는 **기존 앱에 점진적으로 도입**할 수 있다는 점입니다. 전체 앱을 KMP로 전환하는 대신, 공통 유틸리티 함수나 데이터 파싱 로직부터 시작하는 것이 현실적입니다. `shared` 모듈을 Android 프로젝트에 `implementation(project(":shared"))`로 추가하기만 하면 됩니다.

---

## 정리: KMP를 선택해야 하는 순간

| 상황 | 추천 |
|------|------|
| Android + iOS 동시 개발, 비즈니스 로직 공유 필요 | **KMP 적극 추천** |
| UI 구현까지 최대로 공유하고 싶음 | **Compose Multiplatform 검토** |
| Android Only | KMP 불필요 |
| 빠른 MVP, 작은 팀 | Flutter 또는 React Native 검토 |

KMP는 "플랫폼을 추상화하는" 기술이 아니라 **"플랫폼을 존중하면서 공유를 극대화하는"** 기술입니다. `expect/actual`로 플랫폼 경계를 명확히 정의하고, Ktor와 SQLDelight 같은 KMP 네이티브 라이브러리로 공유 레이어를 채우면, Android와 iOS 개발자가 같은 코드베이스에서 각자의 장점을 살릴 수 있는 강력한 아키텍처를 만들 수 있습니다.

## 참고 자료
- [What is Kotlin Multiplatform — 공식 문서](https://kotlinlang.org/docs/multiplatform/kmp-overview.html)
- [Kotlin Multiplatform — Android Developers](https://developer.android.com/kotlin/multiplatform)
- [Ktor + SQLDelight로 멀티플랫폼 앱 만들기 — 공식 튜토리얼](https://kotlinlang.org/docs/multiplatform/multiplatform-ktor-sqldelight.html)
