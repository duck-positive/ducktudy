---
layout: post
title: "Kotlin Multiplatform(KMP) 심화: expect/actual 메커니즘과 공유 비즈니스 로직으로 Android·iOS를 동시에 정복하는 전략"
date: 2026-11-09
categories: [android, kotlin]
tags: [kotlin, kmp, kotlin-multiplatform, expect-actual, shared-code, ios, android, coroutines, ktor]
---

현대 모바일 개발에서 Android와 iOS를 동시에 지원해야 하는 상황은 매우 흔합니다. 기존에는 두 플랫폼의 비즈니스 로직을 각각 Kotlin과 Swift로 중복 작성하는 비효율이 발생했습니다. **Kotlin Multiplatform(KMP)**는 이 문제를 Kotlin 한 언어로 비즈니스 로직을 공유함으로써 해결합니다. Google이 2024년 공식 지지를 선언한 이후 Jetpack 라이브러리(ViewModel, Room, DataStore 등)도 KMP를 지원하며 생태계가 급성장하고 있습니다.

## KMP란 무엇인가?

Kotlin Multiplatform은 JetBrains가 개발한 기술로, **단일 Kotlin 코드베이스에서 여러 플랫폼(Android, iOS, Desktop, Web)을 타겟으로 컴파일**할 수 있게 해줍니다. Flutter처럼 UI까지 공유하는 방식(Compose Multiplatform)도 있지만, KMP의 핵심 가치는 **UI는 각 플랫폼의 네이티브로, 비즈니스 로직만 공유**하는 접근법에 있습니다.

### 프로젝트 구조

KMP 프로젝트는 세 가지 핵심 소스셋으로 구성됩니다.

```
shared/
├── commonMain/        # 모든 플랫폼에서 공유되는 코드
│   └── kotlin/
│       └── com/example/shared/
│           ├── domain/      # UseCase, Repository 인터페이스
│           ├── data/        # 네트워크, 로컬 DB
│           └── util/        # 공통 유틸리티
├── androidMain/       # Android 전용 actual 구현체
│   └── kotlin/
└── iosMain/           # iOS 전용 actual 구현체
    └── kotlin/
```

`commonMain`에 작성된 코드는 Kotlin/JVM(Android), Kotlin/Native(iOS), Kotlin/JS(Web)로 각각 컴파일됩니다. 이때 플랫폼별로 다르게 동작해야 하는 부분을 처리하는 핵심 메커니즘이 바로 `expect/actual`입니다.

## 왜 KMP가 필요한가?

### 코드 중복의 현실

예를 들어 사용자 인증 로직을 생각해봅시다. 토큰 갱신, 세션 만료 처리, 비밀번호 검증 규칙 같은 비즈니스 로직은 Android와 iOS에서 동일해야 합니다. 그러나 기존 방식에서는:

- Kotlin 개발자가 Android용 로직을 작성
- Swift 개발자가 동일한 로직을 iOS용으로 재작성
- 버그 수정 시 양쪽 모두 패치 필요
- 로직 불일치로 인한 플랫폼별 버그 발생

KMP를 사용하면 이 로직을 `commonMain`에 한 번만 작성하고, 양 플랫폼에서 동일하게 사용할 수 있습니다.

### Google의 공식 지지

2024년 Google I/O에서 Google은 KMP를 공식 지지한다고 선언했습니다. Google Docs 앱이 KMP로 비즈니스 로직을 Android, iOS, Web에 공유하는 첫 번째 프로덕션 사례가 되었습니다. 현재 다음 Jetpack 라이브러리들이 KMP를 지원합니다.

- `androidx.lifecycle:lifecycle-viewmodel` — ViewModel
- `androidx.room:room-runtime` — 로컬 DB
- `androidx.datastore:datastore-preferences` — 설정 저장
- `androidx.paging:paging-common` — 페이징
- `androidx.collection:collection` — 컬렉션 유틸

## expect/actual 메커니즘

`expect/actual`은 KMP의 핵심입니다. `commonMain`에서 플랫폼별 구현이 필요한 선언을 `expect`로 정의하고, 각 플랫폼의 소스셋에서 `actual`로 구현합니다.

### 코드 예제 1: 플랫폼별 현재 시간 처리

타임스탬프 처리는 플랫폼마다 API가 다릅니다. KMP에서는 이를 `expect/actual`로 추상화합니다.

```kotlin
// commonMain: 선언만 하고 구현은 각 플랫폼에 위임
expect fun currentTimeMillis(): Long

expect class PlatformContext

// 플랫폼 독립적인 비즈니스 로직에서 사용
class SessionManager(private val context: PlatformContext) {
    private var lastActiveTime: Long = currentTimeMillis()

    fun isSessionExpired(timeoutMs: Long = 30 * 60 * 1000L): Boolean {
        return currentTimeMillis() - lastActiveTime > timeoutMs
    }

    fun refreshSession() {
        lastActiveTime = currentTimeMillis()
    }
}
```

```kotlin
// androidMain: Android 전용 actual 구현
import android.content.Context

actual fun currentTimeMillis(): Long = System.currentTimeMillis()

actual class PlatformContext(val context: Context)
```

```kotlin
// iosMain: iOS 전용 actual 구현
import platform.Foundation.NSDate

actual fun currentTimeMillis(): Long =
    (NSDate().timeIntervalSince1970 * 1000).toLong()

actual class PlatformContext
```

이렇게 하면 `SessionManager`는 `commonMain`에 위치하면서도 플랫폼별 시간 API를 올바르게 사용합니다.

### 코드 예제 2: Ktor + 코루틴으로 공유 Repository 구현

가장 실용적인 KMP 활용 패턴은 네트워크 레이어를 공유하는 것입니다. Ktor HTTP 클라이언트는 KMP를 완전 지원합니다.

```kotlin
// commonMain: 공통 데이터 모델과 Repository
import io.ktor.client.*
import io.ktor.client.call.*
import io.ktor.client.plugins.contentnegotiation.*
import io.ktor.client.request.*
import io.ktor.serialization.kotlinx.json.*
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.flow
import kotlinx.serialization.Serializable

@Serializable
data class User(
    val id: Int,
    val name: String,
    val email: String
)

@Serializable
data class ApiResponse<T>(
    val data: T,
    val message: String
)

interface UserRepository {
    fun getUsers(): Flow<List<User>>
    suspend fun getUserById(id: Int): User
}

class UserRepositoryImpl(
    private val httpClient: HttpClient
) : UserRepository {

    override fun getUsers(): Flow<List<User>> = flow {
        val response = httpClient.get("https://api.example.com/users")
            .body<ApiResponse<List<User>>>()
        emit(response.data)
    }

    override suspend fun getUserById(id: Int): User {
        return httpClient.get("https://api.example.com/users/$id")
            .body<ApiResponse<User>>()
            .data
    }
}

// expect로 플랫폼별 HttpClient 팩토리 선언
expect fun createHttpClient(): HttpClient
```

```kotlin
// androidMain: OkHttp 엔진 사용
import io.ktor.client.*
import io.ktor.client.engine.okhttp.*
import io.ktor.client.plugins.contentnegotiation.*
import io.ktor.serialization.kotlinx.json.*
import kotlinx.serialization.json.Json

actual fun createHttpClient(): HttpClient = HttpClient(OkHttp) {
    install(ContentNegotiation) {
        json(Json {
            ignoreUnknownKeys = true
            isLenient = true
        })
    }
    engine {
        config {
            connectTimeout(30, java.util.concurrent.TimeUnit.SECONDS)
            readTimeout(30, java.util.concurrent.TimeUnit.SECONDS)
        }
    }
}
```

```kotlin
// iosMain: Darwin 엔진 사용
import io.ktor.client.*
import io.ktor.client.engine.darwin.*
import io.ktor.client.plugins.contentnegotiation.*
import io.ktor.serialization.kotlinx.json.*
import kotlinx.serialization.json.Json

actual fun createHttpClient(): HttpClient = HttpClient(Darwin) {
    install(ContentNegotiation) {
        json(Json {
            ignoreUnknownKeys = true
            isLenient = true
        })
    }
    engine {
        configureRequest {
            setTimeoutInterval(30.0)
        }
    }
}
```

### ViewModel 공유 (Jetpack ViewModel KMP 지원)

`androidx.lifecycle:lifecycle-viewmodel` 2.8.0부터 ViewModel이 KMP를 지원합니다. 이제 `commonMain`에 직접 ViewModel을 작성할 수 있습니다.

```kotlin
// commonMain: 공유 ViewModel (더 이상 expect/actual 불필요)
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.launch

data class UserListUiState(
    val users: List<User> = emptyList(),
    val isLoading: Boolean = false,
    val error: String? = null
)

class UserListViewModel(
    private val userRepository: UserRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow(UserListUiState())
    val uiState: StateFlow<UserListUiState> = _uiState.asStateFlow()

    init {
        loadUsers()
    }

    private fun loadUsers() {
        viewModelScope.launch {
            _uiState.value = _uiState.value.copy(isLoading = true)
            try {
                userRepository.getUsers().collect { users ->
                    _uiState.value = UserListUiState(users = users)
                }
            } catch (e: Exception) {
                _uiState.value = UserListUiState(error = e.message)
            }
        }
    }
}
```

Android에서는 기존의 `viewModel()` Compose 확장 함수로, iOS에서는 직접 인스턴스화하여 사용합니다.

## Gradle 설정

`build.gradle.kts` (shared 모듈)에서 KMP 멀티플랫폼 소스셋을 설정합니다.

```kotlin
// shared/build.gradle.kts
plugins {
    alias(libs.plugins.kotlinMultiplatform)
    alias(libs.plugins.androidLibrary)
    alias(libs.plugins.kotlinSerialization)
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
            implementation(libs.kotlinx.coroutines.core)
            implementation(libs.kotlinx.serialization.json)
            implementation(libs.ktor.client.core)
            implementation(libs.ktor.client.content.negotiation)
            implementation(libs.ktor.serialization.kotlinx.json)
            implementation(libs.androidx.lifecycle.viewmodel)
        }
        androidMain.dependencies {
            implementation(libs.ktor.client.okhttp)
        }
        iosMain.dependencies {
            implementation(libs.ktor.client.darwin)
        }
    }
}
```

## 주의사항 및 실전 팁

### 1. iOS 코루틴 주의: `@Throws` 어노테이션

Swift는 Kotlin의 `suspend` 함수를 비동기 메서드로 사용하지만, 예외 처리 방식이 다릅니다. iOS에서 호출될 `suspend` 함수에는 `@Throws`를 명시해야 합니다.

```kotlin
// commonMain
@Throws(Exception::class)
suspend fun fetchUserSafely(id: Int): User {
    return userRepository.getUserById(id)
}
```

### 2. Flow를 iOS에서 사용하려면 래퍼 필요

Kotlin `Flow`는 Swift에서 직접 사용하기 어렵습니다. SKIE 라이브러리를 사용하면 `Flow`를 Swift의 `AsyncSequence`로 자동 변환해주어 간편하게 사용할 수 있습니다.

### 3. expect class vs expect fun 선택 기준

- 단순 계산이나 값 반환: `expect fun` 사용
- 플랫폼 Context나 복잡한 객체: `expect class` 사용
- Jetpack 라이브러리로 대체 가능하면: KMP 지원 버전으로 업그레이드

### 4. 모듈 분리 전략 (클린 아키텍처 적용)

```
:shared:domain     — 순수 비즈니스 로직, 인터페이스 (공통)
:shared:data       — Repository 구현, 네트워크/DB (공통)
:shared:platform   — expect/actual 구현체 (플랫폼 분리)
:androidApp        — Android UI (Jetpack Compose)
:iosApp            — iOS UI (SwiftUI)
```

### 5. 점진적 마이그레이션

기존 Android 프로젝트에 KMP를 도입할 때는:
1. 가장 로직이 단순하고 UI 의존성이 낮은 `util` 패키지부터 이동
2. 다음으로 `domain` 계층 (UseCase, Entity) 이동
3. 마지막으로 `data` 계층 (Repository, Datasource) 이동

이 순서로 작업하면 리스크를 최소화하면서 KMP의 이점을 점진적으로 얻을 수 있습니다.

## 정리

Kotlin Multiplatform은 "한 번 작성하고 여러 곳에서 실행"이라는 오랜 꿈을 현실적인 방식으로 실현합니다. UI는 각 플랫폼의 네이티브 경험을 유지하면서, 비즈니스 로직의 중복을 제거하여 개발 생산성과 코드 일관성을 동시에 높일 수 있습니다. `expect/actual` 메커니즘을 통한 플랫폼별 추상화, Ktor를 활용한 공유 네트워크 레이어, 그리고 Jetpack ViewModel의 KMP 지원으로 이제 Android 개발자도 최소한의 학습 비용으로 iOS 로직을 함께 관리할 수 있는 시대가 열렸습니다.

## 참고 자료
- [Kotlin Multiplatform | Android Developers](https://developer.android.com/kotlin/multiplatform)
- [Kotlin Multiplatform Setup | Android Developers](https://developer.android.com/kotlin/multiplatform/setup)
