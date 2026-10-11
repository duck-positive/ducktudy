---
layout: post
title: "Android OkHttp & Retrofit 심화: 커스텀 인터셉터·Authenticator·ConnectionPool 완전 정복"
date: 2026-10-07
categories: [android, kotlin]
tags: [android, okhttp, retrofit, interceptor, authenticator, connectionpool, kotlin, network]
---

현대 Android 앱의 네트워크 레이어는 단순한 HTTP 요청 이상의 역할을 합니다. 인증 토큰 자동 갱신, 요청/응답 로깅, 재시도 전략, 연결 풀 관리 등 수많은 횡단 관심사(cross-cutting concern)를 효율적으로 처리해야 합니다. OkHttp와 Retrofit은 이러한 문제를 해결하는 업계 표준 라이브러리이지만, 그 고급 기능을 온전히 활용하는 개발자는 많지 않습니다. 이 글에서는 인터셉터 내부 동작, Authenticator를 이용한 토큰 자동 갱신, 그리고 ConnectionPool 튜닝까지 실전 중심으로 깊이 있게 살펴봅니다.

## OkHttp와 Retrofit의 역할과 관계

**OkHttp**는 JVM/Android를 위한 저수준 HTTP 클라이언트입니다. HTTP/2, 연결 풀링, GZIP 압축, 응답 캐싱, 재시도 등을 처리하며, 인터셉터를 통해 요청·응답 파이프라인에 개입할 수 있습니다. 현재 최신 버전은 5.x이며 Android API 21+를 지원합니다.

**Retrofit**은 OkHttp 위에 올라가는 타입 안전 HTTP 클라이언트입니다. 코틀린 인터페이스와 어노테이션으로 REST API를 선언하면, Retrofit이 OkHttp 요청으로 변환하고 응답을 다시 도메인 객체로 역직렬화해 줍니다. 두 라이브러리는 강하게 결합되어 있으며, Retrofit은 내부적으로 `Call.Factory`로서 OkHttpClient를 사용합니다.

```kotlin
// build.gradle.kts
dependencies {
    implementation("com.squareup.okhttp3:okhttp:5.0.0-alpha.14")
    implementation("com.squareup.okhttp3:logging-interceptor:5.0.0-alpha.14")
    implementation("com.squareup.retrofit2:retrofit:2.11.0")
    implementation("com.squareup.retrofit2:converter-moshi:2.11.0")
}
```

### Application Interceptor vs Network Interceptor

OkHttp 인터셉터에는 두 가지 종류가 있으며, 이 차이를 이해하는 것이 핵심입니다.

| 구분 | Application Interceptor | Network Interceptor |
|---|---|---|
| 등록 위치 | `addInterceptor()` | `addNetworkInterceptor()` |
| 호출 시점 | 캐시 확인 이전 | 실제 네트워크 직전 |
| 리다이렉트/재시도 | 한 번만 호출됨 | 매번 호출됨 |
| 캐시 응답 | 가로챌 수 있음 | 캐시 응답에는 호출 안 됨 |
| 주요 용도 | 인증 헤더, 공통 파라미터 | 트래픽 분석, 네트워크 레이어 로깅 |

**Application Interceptor**는 실제 네트워크 요청 여부와 관계없이 `call.proceed()` 전후에 개입합니다. 인증 토큰 추가나 공통 헤더 삽입에 적합합니다.

**Network Interceptor**는 실제 네트워크 전송 직전에 호출되며, 캐시 히트 시에는 실행되지 않습니다. 헤더 기록이나 트래픽 모니터링에 적합합니다.

## 왜 이것이 중요한가

실제 프로덕션 앱을 개발하다 보면 반드시 마주치는 상황이 있습니다.

- **401 Unauthorized**: Access Token이 만료되었을 때, 모든 API 호출에서 401을 처리해야 한다면 코드 중복이 폭발적으로 늘어납니다.
- **공통 헤더 누락**: `Authorization`, `Accept-Language`, `X-Device-Id` 같은 헤더를 매 요청마다 수동으로 추가하는 것은 실수의 원인이 됩니다.
- **로깅 부재**: 프로덕션 이슈 디버깅 시 요청/응답 본문이 없으면 원인 파악이 어렵습니다.
- **연결 풀 미설정**: 기본 ConnectionPool(5개 커넥션, 5분 유지)이 고트래픽 앱에는 부족하거나 과도할 수 있습니다.

이 모든 문제를 인터셉터 하나 혹은 소수의 컴포넌트로 중앙 집중적으로 해결할 수 있습니다.

## 실전 구현: 토큰 자동 갱신 인터셉터와 Authenticator

가장 복잡하고 흔한 사례는 **JWT Access Token 만료 시 자동 갱신**입니다. 올바른 구현은 인터셉터와 Authenticator를 조합해야 합니다.

```kotlin
import kotlinx.coroutines.runBlocking
import kotlinx.coroutines.sync.Mutex
import kotlinx.coroutines.sync.withLock
import okhttp3.Authenticator
import okhttp3.Interceptor
import okhttp3.OkHttpClient
import okhttp3.Request
import okhttp3.Response
import okhttp3.Route

// 토큰 저장소 (실제로는 DataStore나 EncryptedSharedPreferences 사용 권장)
class TokenRepository {
    @Volatile private var accessToken: String = ""
    @Volatile private var refreshToken: String = ""
    private val mutex = Mutex()

    fun getAccessToken(): String = accessToken

    suspend fun refreshTokens(): Boolean = mutex.withLock {
        return try {
            // 실제 토큰 갱신 API 호출 (여기서는 의사코드)
            val newTokens = authApi.refresh(refreshToken)
            accessToken = newTokens.accessToken
            refreshToken = newTokens.refreshToken
            true
        } catch (e: Exception) {
            false
        }
    }
}

// Application Interceptor: 모든 요청에 Authorization 헤더 삽입
class AuthInterceptor(private val tokenRepository: TokenRepository) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val token = tokenRepository.getAccessToken()
        val request = if (token.isNotBlank()) {
            chain.request().newBuilder()
                .header("Authorization", "Bearer $token")
                .header("Accept-Language", "ko-KR")
                .build()
        } else {
            chain.request()
        }
        return chain.proceed(request)
    }
}

// Authenticator: 401 응답 수신 시 토큰 갱신 후 재시도
class TokenAuthenticator(
    private val tokenRepository: TokenRepository
) : Authenticator {

    override fun authenticate(route: Route?, response: Response): Request? {
        // 이미 갱신된 토큰으로 재시도했는데도 401이면 null 반환 (무한 루프 방지)
        val currentToken = tokenRepository.getAccessToken()
        val requestToken = response.request.header("Authorization")
            ?.removePrefix("Bearer ")

        if (currentToken != requestToken) {
            // 다른 코루틴이 이미 토큰을 갱신했다면 최신 토큰으로 바로 재시도
            return response.request.newBuilder()
                .header("Authorization", "Bearer $currentToken")
                .build()
        }

        // 토큰 갱신 시도 (Authenticator는 동기 컨텍스트이므로 runBlocking 사용)
        val refreshed = runBlocking { tokenRepository.refreshTokens() }

        return if (refreshed) {
            response.request.newBuilder()
                .header("Authorization", "Bearer ${tokenRepository.getAccessToken()}")
                .build()
        } else {
            null // 갱신 실패 → 로그아웃 처리 필요
        }
    }
}
```

**핵심 포인트**:
- `AuthInterceptor`는 Application Interceptor로 모든 요청에 토큰을 선제적으로 추가합니다.
- `TokenAuthenticator`는 서버가 401을 반환할 때만 호출됩니다. OkHttp는 기본적으로 최대 20번까지 재시도하므로, 무한 루프 방지 로직이 필수입니다.
- `Mutex`를 사용해 토큰 갱신 중 다른 코루틴이 동시에 갱신을 시도하지 못하도록 막습니다.
- `Authenticator.authenticate()`는 동기 함수이므로 코루틴을 사용하려면 `runBlocking`이 필요합니다.

## 실전 구현: OkHttpClient·Retrofit·ConnectionPool 조립

```kotlin
import okhttp3.ConnectionPool
import okhttp3.OkHttpClient
import okhttp3.logging.HttpLoggingInterceptor
import retrofit2.Retrofit
import retrofit2.converter.moshi.MoshiConverterFactory
import java.util.concurrent.TimeUnit

object NetworkModule {

    private val tokenRepository = TokenRepository()

    // ConnectionPool: 최대 10개 커넥션, 3분간 유지
    private val connectionPool = ConnectionPool(
        maxIdleConnections = 10,
        keepAliveDuration = 3,
        timeUnit = TimeUnit.MINUTES
    )

    // 로깅 인터셉터: BODY 레벨은 프로덕션에서 NONE 또는 BASIC으로 설정
    private val loggingInterceptor = HttpLoggingInterceptor().apply {
        level = if (BuildConfig.DEBUG) {
            HttpLoggingInterceptor.Level.BODY
        } else {
            HttpLoggingInterceptor.Level.NONE
        }
    }

    // OkHttpClient는 싱글톤으로 관리 (스레드 풀·커넥션 풀 공유)
    val okHttpClient: OkHttpClient by lazy {
        OkHttpClient.Builder()
            // Application Interceptor: 헤더 삽입 (캐시 응답에도 적용)
            .addInterceptor(AuthInterceptor(tokenRepository))
            // Network Interceptor: 실제 전송 레벨 로깅
            .addNetworkInterceptor(loggingInterceptor)
            // 401 처리
            .authenticator(TokenAuthenticator(tokenRepository))
            // 타임아웃 설정
            .connectTimeout(15, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .writeTimeout(30, TimeUnit.SECONDS)
            // 연결 풀 커스터마이징
            .connectionPool(connectionPool)
            // 실패한 요청 자동 재시도 (GET 요청만 해당)
            .retryOnConnectionFailure(true)
            .build()
    }

    // Retrofit 인스턴스: OkHttpClient 공유
    val retrofit: Retrofit by lazy {
        Retrofit.Builder()
            .baseUrl("https://api.example.com/")
            .client(okHttpClient)
            .addConverterFactory(MoshiConverterFactory.create())
            .build()
    }

    // API 서비스 생성
    inline fun <reified T> createService(): T = retrofit.create(T::class.java)
}

// Retrofit API 인터페이스 예시
interface UserApi {
    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: Long): UserResponse

    @POST("auth/refresh")
    suspend fun refreshToken(@Body body: RefreshRequest): TokenResponse
}

// 사용 예시
val userApi = NetworkModule.createService<UserApi>()

// Repository에서 호출
class UserRepository(private val userApi: UserApi) {
    suspend fun getUser(id: Long): Result<UserResponse> = runCatching {
        userApi.getUser(id)
    }
}
```

### ConnectionPool 설정 기준

| 설정 | 기본값 | 권장 상황 |
|---|---|---|
| maxIdleConnections | 5 | 트래픽 폭발 시 10~20으로 증가 |
| keepAliveDuration | 5분 | 서버 Keep-Alive timeout보다 짧게 설정 |
| 타임아웃 | 10초 | API 특성에 맞게 조정, 최대 60초 초과 지양 |

서버의 Keep-Alive timeout이 예를 들어 60초라면, `keepAliveDuration`을 55초 이하로 설정해야 서버가 먼저 커넥션을 끊는 상황을 방지할 수 있습니다.

## 주의사항 및 실전 팁

### 1. OkHttpClient 공유

OkHttpClient는 내부적으로 스레드 풀, 연결 풀, 캐시를 포함하는 **무거운 객체**입니다. Hilt나 Koin 같은 DI 프레임워크에서 싱글톤으로 주입해 전체 앱이 공유해야 합니다. API 호출마다 새 인스턴스를 생성하면 메모리 누수와 성능 저하의 원인이 됩니다.

### 2. Authenticator의 무한 루프 방지

`TokenAuthenticator.authenticate()`가 `null`을 반환하면 OkHttp는 재시도를 멈추고 원래의 401 응답을 호출자에게 전달합니다. 반드시 토큰 갱신 실패 케이스에서 `null`을 반환하고, 이때 로그아웃 이벤트를 발행해야 합니다. `EventBus`, `SharedFlow`, 또는 싱글톤 상태 홀더를 통해 이 이벤트를 처리하세요.

```kotlin
// 로그아웃 이벤트 발행 예시 (StateFlow 활용)
object AuthEventBus {
    private val _events = MutableSharedFlow<AuthEvent>(extraBufferCapacity = 1)
    val events: SharedFlow<AuthEvent> = _events.asSharedFlow()

    fun emit(event: AuthEvent) { _events.tryEmit(event) }
}

// Authenticator 내에서
if (!refreshed) {
    AuthEventBus.emit(AuthEvent.SessionExpired)
    return null
}
```

### 3. 동시 토큰 갱신 경합 조건 (Race Condition)

여러 API 요청이 동시에 401을 받으면 각각이 토큰 갱신을 시도합니다. `Mutex`(코루틴) 또는 `synchronized`(동기 코드)로 임계 구역을 보호하고, 갱신 완료 후 최신 토큰을 확인해 이미 갱신된 경우 재시도만 수행해야 합니다. 위 코드 예제의 `currentToken != requestToken` 분기가 바로 이 역할을 합니다.

### 4. 로깅 인터셉터와 민감 정보

`HttpLoggingInterceptor.Level.BODY`는 요청/응답 본문 전체를 로그에 남깁니다. 비밀번호, 신용카드 정보, 개인 식별 정보가 포함된 API라면 프로덕션에서 반드시 `Level.NONE`으로 설정하거나, 커스텀 로깅 인터셉터에서 민감 헤더를 마스킹해야 합니다.

```kotlin
// 민감 헤더 마스킹 예시
class SecureLoggingInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()
        val maskedAuth = request.header("Authorization")
            ?.let { "Bearer [REDACTED]" } ?: "(none)"
        // 마스킹된 정보만 로깅
        Log.d("HTTP", "${request.method} ${request.url} | Auth: $maskedAuth")
        return chain.proceed(request)
    }
}
```

### 5. 테스트 시 MockWebServer 활용

OkHttp 팀이 제공하는 `MockWebServer`를 사용하면 실제 서버 없이 인터셉터, Authenticator, 재시도 로직을 단위 테스트할 수 있습니다.

```kotlin
// 테스트 의존성
testImplementation("com.squareup.okhttp3:mockwebserver:5.0.0-alpha.14")

// 사용 예시
val server = MockWebServer()
server.enqueue(MockResponse().setResponseCode(401))
server.enqueue(MockResponse().setBody("""{"data":"ok"}"""))

// 토큰 갱신 후 재시도가 이루어지는지 검증
val request = Request.Builder().url(server.url("/api/data")).build()
val response = okHttpClient.newCall(request).execute()
assertThat(server.requestCount).isEqualTo(2) // 원본 요청 + 재시도
```

## 마무리

OkHttp와 Retrofit의 고급 기능을 올바르게 조합하면, 네트워크 레이어의 인증, 로깅, 재시도, 연결 관리를 중앙에서 일관성 있게 처리할 수 있습니다. 특히 인터셉터와 Authenticator의 역할 분리, `Mutex`를 활용한 스레드 안전한 토큰 갱신, OkHttpClient 싱글톤 관리는 프로덕션 앱에서 반드시 숙지해야 할 패턴입니다. 이 글의 코드 예제를 기반으로 자신의 앱에 맞게 커스터마이징해 보세요.

## 참고 자료
- [OkHttp GitHub Repository](https://github.com/square/okhttp)
- [Retrofit GitHub Repository](https://github.com/square/retrofit)
- [Android 네트워크 연결 가이드 - Android Developers](https://developer.android.com/develop/connectivity/network-ops/connecting)
