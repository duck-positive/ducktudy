---
layout: post
title: "Android Hilt 심화: Module·Component·Scope 완전 정복"
date: 2026-06-20
categories: [android]
tags: [android, hilt, dagger, dependency-injection, di, kotlin, jetpack]
---

의존성 주입(Dependency Injection, DI)은 현대 Android 앱 아키텍처의 근간입니다. Google이 Dagger 위에 올려 만든 Hilt는 보일러플레이트를 대폭 줄이면서도 컴파일 타임 안정성을 유지하는 DI 라이브러리입니다. 기본 사용법은 금방 익힐 수 있지만, **Module·Component·Scope** 세 개념의 상호작용을 깊이 이해하지 못하면 메모리 누수나 예상치 못한 객체 공유 문제가 발생합니다. 이 글에서는 Hilt의 내부 동작 원리부터 실전 패턴까지 완전히 정복합니다.

---

## 왜 Hilt가 필요한가?

수동 DI(ServiceLocator, 생성자 직접 연결)는 소규모 프로젝트에서는 무리가 없지만, 앱이 커질수록 다음 문제가 드러납니다.

- **생성자 연쇄 폭발**: `Repository`가 `DataSource`를, `DataSource`가 `OkHttpClient`를, `OkHttpClient`가 `Interceptor`를 요구하는 의존성 체인이 길어질수록 `Application`이나 `Activity`가 모든 생성 책임을 떠안게 됩니다.
- **생명주기 불일치**: ViewModel이 Application 레벨 singleton을 잘못 참조하거나, 반대로 짧은 생명주기를 가진 객체가 Activity 이후에도 살아남아 메모리 누수를 일으킵니다.
- **테스트 어려움**: 구현체가 직접 생성되면 Mock 교체가 불가능합니다.

Hilt는 이 모든 문제를 **컴파일 타임에** 감지하고, Android 생명주기에 맞는 Component 계층을 자동으로 관리합니다.

---

## Hilt Component 계층 구조

Hilt가 생성하는 Component들은 Android 생명주기와 1:1로 매핑됩니다. 아래 표가 계층 구조를 요약합니다.

| Component | 생성 시점 | 소멸 시점 | 대응 스코프 |
|---|---|---|---|
| `SingletonComponent` | `Application#onCreate()` | `Application` 소멸 | `@Singleton` |
| `ActivityRetainedComponent` | `Activity#onCreate()` | `Activity#onDestroy()` (회전 생존) | `@ActivityRetainedScoped` |
| `ActivityComponent` | `Activity#onCreate()` | `Activity#onDestroy()` | `@ActivityScoped` |
| `ViewModelComponent` | `ViewModel` 생성 | `ViewModel` 소멸 | `@ViewModelScoped` |
| `FragmentComponent` | `Fragment#onAttach()` | `Fragment#onDestroy()` | `@FragmentScoped` |
| `ViewComponent` | `View#init` | `View` 소멸 | `@ViewScoped` |
| `ServiceComponent` | `Service#onCreate()` | `Service#onDestroy()` | `@ServiceScoped` |

**핵심 규칙**: 하위 Component는 상위 Component의 바인딩을 그대로 사용할 수 있지만, 반대는 불가능합니다. `FragmentComponent`에서 `@Singleton` 객체를 주입받는 것은 가능하지만, `SingletonComponent`에서 `@FragmentScoped` 객체를 참조하면 컴파일 에러가 납니다.

---

## Module과 @InstallIn

Module은 Hilt에게 "이 타입을 어떻게 만들어줘"라고 알려주는 설명서입니다. `@Module`과 `@InstallIn`을 함께 선언하며, `@InstallIn`에 어느 Component에 설치할지 지정합니다.

### @Provides vs @Binds

```kotlin
// NetworkModule.kt
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {

    @Provides
    @Singleton
    fun provideOkHttpClient(
        loggingInterceptor: HttpLoggingInterceptor
    ): OkHttpClient {
        return OkHttpClient.Builder()
            .addInterceptor(loggingInterceptor)
            .connectTimeout(30, TimeUnit.SECONDS)
            .build()
    }

    @Provides
    @Singleton
    fun provideRetrofit(okHttpClient: OkHttpClient): Retrofit {
        return Retrofit.Builder()
            .baseUrl("https://api.example.com/")
            .client(okHttpClient)
            .addConverterFactory(GsonConverterFactory.create())
            .build()
    }

    @Provides
    fun provideLoggingInterceptor(): HttpLoggingInterceptor {
        return HttpLoggingInterceptor().apply {
            level = if (BuildConfig.DEBUG)
                HttpLoggingInterceptor.Level.BODY
            else
                HttpLoggingInterceptor.Level.NONE
        }
    }
}
```

`@Provides`는 직접 인스턴스를 반환하므로 외부 라이브러리 클래스처럼 **생성자를 수정할 수 없는 타입**에 사용합니다. 반면 자체 구현 클래스라면 `@Binds`가 더 효율적입니다. `@Binds`는 코드를 생성하지 않고 컴파일 타임에 직접 연결되기 때문에 APK 크기와 빌드 시간 모두 유리합니다.

```kotlin
// RepositoryModule.kt
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {

    // 인터페이스 → 구현체 바인딩: @Binds 사용
    @Binds
    @Singleton
    abstract fun bindUserRepository(
        impl: UserRepositoryImpl
    ): UserRepository

    @Binds
    @Singleton
    abstract fun bindProductRepository(
        impl: ProductRepositoryImpl
    ): ProductRepository
}
```

`@Binds`가 있는 Module은 반드시 **abstract class**여야 합니다. `@Provides`와 `@Binds`를 같은 class에 혼용해야 할 때는 `@Provides` 함수를 `companion object` 안에 넣으면 됩니다.

---

## Scope 심층 분석

### @Singleton이 정말 싱글턴인가?

`@Singleton`으로 스코프된 객체는 `SingletonComponent`가 살아있는 동안, 즉 앱 프로세스가 종료될 때까지 단 하나의 인스턴스만 유지됩니다. 하지만 **스코프를 붙이지 않은 의존성**은 주입될 때마다 새 인스턴스가 생성된다는 점을 반드시 기억해야 합니다.

```kotlin
@Module
@InstallIn(ActivityComponent::class)
object AnalyticsModule {

    // 스코프 없음 → Activity마다 새 인스턴스
    @Provides
    fun provideEventTracker(context: Context): EventTracker {
        return EventTracker(context)
    }

    // @ActivityScoped → 같은 Activity 내에서 공유
    @Provides
    @ActivityScoped
    fun provideSessionManager(context: Context): SessionManager {
        return SessionManager(context)
    }
}
```

### @ViewModelScoped의 올바른 사용

`@ViewModelScoped`는 ViewModel이 살아있는 동안 동일 인스턴스를 공유하므로, ViewModel 내 여러 UseCase가 같은 상태를 공유해야 할 때 유용합니다.

```kotlin
// UseCase들이 같은 PagingState 인스턴스를 공유하는 예
@ViewModelScoped
class PagingState @Inject constructor() {
    var currentPage: Int = 0
    var isLoading: Boolean = false
}

class FetchProductsUseCase @Inject constructor(
    private val repository: ProductRepository,
    private val state: PagingState   // ViewModel 내 공유
)

class RefreshUseCase @Inject constructor(
    private val repository: ProductRepository,
    private val state: PagingState   // 동일 인스턴스
)

@HiltViewModel
class ProductViewModel @Inject constructor(
    private val fetchUseCase: FetchProductsUseCase,
    private val refreshUseCase: RefreshUseCase
) : ViewModel()
```

---

## 실전 예제: 멀티 바인딩과 @Qualifier

같은 타입의 의존성이 여러 개 필요할 때는 `@Qualifier`를 사용합니다. 예를 들어 인증이 필요한 API와 Public API를 분리하는 경우가 대표적입니다.

```kotlin
// Qualifier 어노테이션 정의
@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class AuthenticatedClient

@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class PublicClient

// Module에서 두 종류의 OkHttpClient 제공
@Module
@InstallIn(SingletonComponent::class)
object HttpClientModule {

    @Provides
    @Singleton
    @AuthenticatedClient
    fun provideAuthenticatedClient(
        tokenInterceptor: TokenInterceptor,
        loggingInterceptor: HttpLoggingInterceptor
    ): OkHttpClient {
        return OkHttpClient.Builder()
            .addInterceptor(tokenInterceptor)
            .addInterceptor(loggingInterceptor)
            .build()
    }

    @Provides
    @Singleton
    @PublicClient
    fun providePublicClient(
        loggingInterceptor: HttpLoggingInterceptor
    ): OkHttpClient {
        return OkHttpClient.Builder()
            .addInterceptor(loggingInterceptor)
            .build()
    }
}

// Repository에서 Qualifier로 구분하여 주입받기
class AuthApiRepository @Inject constructor(
    @AuthenticatedClient private val client: OkHttpClient,
    private val baseUrl: String
)

class PublicApiRepository @Inject constructor(
    @PublicClient private val client: OkHttpClient,
    private val baseUrl: String
)
```

---

## EntryPoint: Hilt 미지원 클래스에서 의존성 꺼내기

`ContentProvider`, `BroadcastReceiver` 일부, 혹은 서드파티 라이브러리처럼 Hilt가 생명주기를 관리하지 않는 클래스에서 의존성이 필요할 때는 `@EntryPoint`를 사용합니다.

```kotlin
@EntryPoint
@InstallIn(SingletonComponent::class)
interface DatabaseEntryPoint {
    fun appDatabase(): AppDatabase
}

// Hilt 미지원 클래스에서 사용
class LegacyContentProvider : ContentProvider() {

    private lateinit var db: AppDatabase

    override fun onCreate(): Boolean {
        val appContext = context?.applicationContext ?: return false
        val entryPoint = EntryPointAccessors.fromApplication(
            appContext,
            DatabaseEntryPoint::class.java
        )
        db = entryPoint.appDatabase()
        return true
    }
}
```

---

## 주의사항 및 실전 팁

**1. @HiltViewModel 없이 ViewModel을 주입하면 런타임 크래시가 발생합니다.**
`viewModels()` 델리게이트는 Hilt의 `HiltViewModelFactory`를 통해 동작합니다. `@HiltViewModel`이 빠지면 팩토리가 ViewModel을 찾지 못해 `IllegalStateException`이 발생합니다.

**2. @Singleton 남용 주의**
모든 의존성을 `@Singleton`으로 선언하면 메모리가 해제되지 않고, 테스트 격리도 어려워집니다. 실제로 앱 전역에서 공유해야 할 이유가 있는 경우에만 사용하세요. 데이터베이스, Retrofit 인스턴스, Repository 정도가 적합합니다.

**3. 멀티모듈 프로젝트에서 feature module 주의**
Feature module은 app module에 역방향으로 의존하기 때문에 `@InstallIn`을 통한 일반적인 방식이 동작하지 않습니다. feature module에서는 `@EntryPoint`로 app module의 바인딩을 가져오거나, 별도의 Dagger Component로 연결해야 합니다.

**4. 테스트에서 Module 교체**
`@TestInstallIn`으로 운영 Module을 테스트용으로 교체할 수 있습니다. `@UninstallModules`와 함께 사용하면 특정 Module만 골라 대체할 수 있어 테스트 유연성이 크게 높아집니다.

```kotlin
@TestInstallIn(
    components = [SingletonComponent::class],
    replaces = [NetworkModule::class]
)
@Module
object FakeNetworkModule {

    @Provides
    @Singleton
    fun provideOkHttpClient(): OkHttpClient {
        return OkHttpClient.Builder().build() // 실제 네트워크 없이 테스트
    }
}
```

**5. Hilt 빌드 속도 최적화**
Hilt는 APT(어노테이션 프로세서)로 동작하므로 모듈이 많아질수록 빌드가 느려질 수 있습니다. `kapt`보다 `ksp(Kotlin Symbol Processing)`를 사용하면 빌드 시간을 20~40% 단축할 수 있습니다. Hilt 2.48 이상에서 ksp 공식 지원이 제공됩니다.

---

## 정리

| 개념 | 역할 | 핵심 어노테이션 |
|---|---|---|
| Module | 의존성 생성 방법 정의 | `@Module`, `@Provides`, `@Binds` |
| Component | 의존성 생명주기 컨테이너 | `@InstallIn(XxxComponent::class)` |
| Scope | 동일 인스턴스 공유 범위 | `@Singleton`, `@ActivityScoped`, `@ViewModelScoped` |
| Qualifier | 같은 타입의 다중 바인딩 구분 | `@Qualifier` + 커스텀 어노테이션 |
| EntryPoint | Hilt 외부에서 의존성 접근 | `@EntryPoint`, `EntryPointAccessors` |

Module·Component·Scope 세 개념의 상호작용을 이해하면 Hilt 코드에서 "왜 이 객체가 공유되는가", "왜 새 인스턴스가 생기는가"를 즉각 파악할 수 있습니다. 아키텍처 설계 단계에서 각 의존성의 올바른 스코프를 결정하는 습관이 곧 안정적인 앱의 시작입니다.

## 참고 자료
- [Dependency injection with Hilt - Android Developers](https://developer.android.com/training/dependency-injection/hilt-android)
- [Hilt in multi-module apps - Android Developers](https://developer.android.com/training/dependency-injection/hilt-multi-module)
- [Using Hilt in your Android app (Codelab) - Android Developers](https://developer.android.com/codelabs/android-hilt)
