---
layout: post
title: "Android 멀티 모듈 아키텍처 심화: Feature 모듈·Core 모듈·Gradle Convention Plugin으로 빌드 성능을 극대화하는 전략"
date: 2026-07-07
categories: [android]
tags: [android, modularization, gradle, convention-plugin, multi-module, kotlin]
---

앱이 커질수록 단일 모듈(monolithic) 구조는 한계를 드러냅니다. 코드 한 줄을 바꿔도 전체 모듈을 재컴파일해야 하고, 팀원끼리 같은 파일을 두고 충돌하며, 어느새 모든 클래스가 서로를 직접 참조하는 스파게티 코드가 됩니다. **멀티 모듈 아키텍처**는 이 문제를 해결하는 구조적 처방입니다. 이 글에서는 모듈 유형별 역할 정의부터 Gradle Convention Plugin 작성, 모듈 간 Navigation까지 실전 수준으로 다룹니다.

---

## 왜 멀티 모듈인가?

### 빌드 성능의 劇的인 개선

Gradle은 변경된 모듈과 그 의존 모듈만 재컴파일합니다. `feature:profile` 모듈만 수정했다면 `feature:feed`나 `core:network` 모듈은 캐시된 결과를 그대로 씁니다. 대형 프로젝트에서 클린 빌드 시간이 10분이라도 증분 빌드는 30초로 줄어드는 경험을 할 수 있습니다. 특히 **Gradle Build Cache**와 결합하면 CI에서도 이전 빌드 결과물을 재활용해 파이프라인이 빨라집니다.

### 강제된 아키텍처 경계

모듈이 나뉘면 의존 방향이 명시됩니다. `feature:checkout`이 `feature:profile`을 직접 `import`하려 하면 컴파일 오류가 납니다. 이 물리적 장벽이 아키텍처 원칙을 규약이 아닌 컴파일 오류로 강제합니다.

### 테스트 고립성

각 모듈은 독립 단위로 테스트할 수 있습니다. `core:domain` 모듈은 Android 프레임워크 없이 순수 JVM 테스트로 빠르게 검증하고, `feature:*` 모듈은 Mock 데이터 소스를 주입해 UI 테스트만 집중할 수 있습니다.

### 동적 배포 (Play Feature Delivery)

`feature` 모듈을 **Dynamic Feature Module**로 선언하면 앱 설치 후 필요한 시점에만 해당 기능을 다운로드할 수 있습니다. 앱 초기 설치 크기를 줄이면서도 전체 기능을 제공하는 전략입니다.

---

## 모듈 유형 분류

Now in Android 등 Google 공식 레퍼런스 앱이 사용하는 계층 구조를 기반으로 설명합니다.

```
:app                         ← 앱 진입점, 루트 네비게이션
:feature:feed                ← 피드 화면 (UI + ViewModel)
:feature:profile             ← 프로필 화면
:feature:settings            ← 설정 화면
:data:feed                   ← 피드 Repository + DataSource
:data:user                   ← 사용자 Repository + DataSource
:domain                      ← UseCase, 비즈니스 로직 (순수 Kotlin)
:core:ui                     ← 공통 Composable, 테마, 타이포그래피
:core:network                ← Ktor/Retrofit 클라이언트 설정
:core:database               ← Room 스키마 및 DAO
:core:common                 ← 확장 함수, 유틸리티
:core:testing                ← FakeRepository, 테스트용 DI 모듈
```

| 모듈 유형 | 의존 방향 | Android 의존 여부 |
|-----------|-----------|------------------|
| `:app` | feature·core 의존 가능 | 있음 |
| `:feature:*` | data·core·domain 의존 가능 | 있음 (Compose) |
| `:data:*` | core·domain 의존 가능 | 있음 (Room·DataStore) |
| `:domain` | 외부 의존 없음 | 없음 (순수 Kotlin) |
| `:core:*` | 다른 core 최소 의존 | 유형마다 다름 |

---

## 실제 구현: Gradle Convention Plugin

멀티 모듈 프로젝트에서 가장 고통스러운 부분은 **반복되는 build.gradle.kts**입니다. 10개 모듈이 생기면 `compileSdk`, `minSdk`, Kotlin 버전, Compose 설정, ktlint 설정이 10번 중복됩니다. **Convention Plugin**은 이 공통 설정을 플러그인으로 추출해 한 줄로 적용하게 합니다.

### 1단계: `build-logic` 모듈 생성

프로젝트 루트에 `build-logic/` 디렉토리를 만들고 독립 Gradle 프로젝트로 선언합니다.

```kotlin
// settings.gradle.kts (루트)
pluginManagement {
    includeBuild("build-logic")  // Convention Plugin을 빌드 로직으로 포함
}

include(":app")
include(":feature:feed")
include(":feature:profile")
include(":data:feed")
include(":core:ui")
include(":core:network")
include(":domain")
```

```kotlin
// build-logic/settings.gradle.kts
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
    }
    versionCatalogs {
        create("libs") {
            from(files("../gradle/libs.versions.toml"))
        }
    }
}

rootProject.name = "build-logic"
include(":convention")
```

### 2단계: 공통 Convention Plugin 작성

```kotlin
// build-logic/convention/src/main/kotlin/AndroidLibraryConventionPlugin.kt

import com.android.build.gradle.LibraryExtension
import org.gradle.api.Plugin
import org.gradle.api.Project
import org.gradle.kotlin.dsl.configure
import org.gradle.kotlin.dsl.dependencies
import org.gradle.kotlin.dsl.getByType

class AndroidLibraryConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            with(pluginManager) {
                apply("com.android.library")
                apply("org.jetbrains.kotlin.android")
                apply("org.jlleitschuh.gradle.ktlint")
            }

            extensions.configure<LibraryExtension> {
                compileSdk = 35

                defaultConfig {
                    minSdk = 26
                    testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
                    consumerProguardFiles("consumer-rules.pro")
                }

                compileOptions {
                    sourceCompatibility = JavaVersion.VERSION_17
                    targetCompatibility = JavaVersion.VERSION_17
                }

                // 모든 라이브러리 모듈에서 BuildConfig 비활성화 (빌드 속도 향상)
                buildFeatures {
                    buildConfig = false
                }
            }

            dependencies {
                // 모든 Android 모듈에 공통으로 필요한 의존성
                add("implementation", libs.findLibrary("kotlin.stdlib").get())
                add("testImplementation", libs.findLibrary("junit").get())
            }
        }
    }
}
```

```kotlin
// build-logic/convention/src/main/kotlin/AndroidLibraryComposeConventionPlugin.kt

import com.android.build.gradle.LibraryExtension
import org.gradle.api.Plugin
import org.gradle.api.Project
import org.gradle.kotlin.dsl.configure
import org.gradle.kotlin.dsl.dependencies

class AndroidLibraryComposeConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            // AndroidLibraryConventionPlugin을 먼저 적용
            pluginManager.apply("com.ducktudy.android.library")
            pluginManager.apply("org.jetbrains.kotlin.plugin.compose")

            extensions.configure<LibraryExtension> {
                buildFeatures {
                    compose = true
                }
            }

            dependencies {
                val bom = libs.findLibrary("androidx.compose.bom").get()
                add("implementation", platform(bom))
                add("implementation", libs.findLibrary("androidx.compose.ui").get())
                add("implementation", libs.findLibrary("androidx.compose.material3").get())
                add("implementation", libs.findLibrary("androidx.compose.ui.tooling.preview").get())
                add("debugImplementation", libs.findLibrary("androidx.compose.ui.tooling").get())
                add("androidTestImplementation", platform(bom))
            }
        }
    }
}
```

```kotlin
// build-logic/convention/build.gradle.kts

plugins {
    `kotlin-dsl`
}

dependencies {
    compileOnly(libs.android.gradlePlugin)
    compileOnly(libs.kotlin.gradlePlugin)
    compileOnly(libs.compose.compiler.gradlePlugin)
}

gradlePlugin {
    plugins {
        register("androidLibrary") {
            id = "com.ducktudy.android.library"
            implementationClass = "AndroidLibraryConventionPlugin"
        }
        register("androidLibraryCompose") {
            id = "com.ducktudy.android.library.compose"
            implementationClass = "AndroidLibraryComposeConventionPlugin"
        }
    }
}
```

이제 각 모듈의 `build.gradle.kts`가 극도로 단순해집니다.

```kotlin
// feature/feed/build.gradle.kts
plugins {
    alias(libs.plugins.ducktudy.android.library.compose)
    alias(libs.plugins.hilt)
    alias(libs.plugins.kotlin.serialization)
}

android {
    namespace = "com.ducktudy.feature.feed"
}

dependencies {
    implementation(projects.core.ui)
    implementation(projects.data.feed)
    implementation(libs.hilt.android)
    ksp(libs.hilt.compiler)
    implementation(libs.androidx.lifecycle.viewmodel.compose)
}
```

---

## 실제 구현: 모듈 간 Navigation

Feature 모듈들은 서로를 모르기 때문에, 네비게이션을 어떻게 연결할지가 핵심 과제입니다. 가장 권장되는 패턴은 **네비게이션 그래프를 `:app` 모듈에서 조립**하는 것입니다.

```kotlin
// core/navigation/src/main/kotlin/NavigationDestination.kt

interface NavigationDestination {
    val route: String
    val titleTextId: Int
}
```

```kotlin
// feature/feed/src/main/kotlin/FeedNavigation.kt

import androidx.navigation.NavController
import androidx.navigation.NavGraphBuilder
import androidx.navigation.NavOptions
import androidx.navigation.compose.composable

const val FEED_ROUTE = "feed"

fun NavController.navigateToFeed(navOptions: NavOptions? = null) {
    navigate(FEED_ROUTE, navOptions)
}

// NavGraphBuilder 확장 함수로 그래프 조각을 제공
fun NavGraphBuilder.feedScreen(
    onArticleClick: (String) -> Unit,
) {
    composable(route = FEED_ROUTE) {
        FeedRoute(onArticleClick = onArticleClick)
    }
}
```

```kotlin
// feature/profile/src/main/kotlin/ProfileNavigation.kt

const val PROFILE_ROUTE = "profile/{userId}"

fun NavController.navigateToProfile(userId: String, navOptions: NavOptions? = null) {
    navigate("profile/$userId", navOptions)
}

fun NavGraphBuilder.profileScreen(onBackClick: () -> Unit) {
    composable(
        route = PROFILE_ROUTE,
        arguments = listOf(navArgument("userId") { type = NavType.StringType })
    ) { backStackEntry ->
        val userId = backStackEntry.arguments?.getString("userId") ?: return@composable
        ProfileRoute(userId = userId, onBackClick = onBackClick)
    }
}
```

```kotlin
// app/src/main/kotlin/DucktudyNavHost.kt

@Composable
fun DucktudyNavHost(
    navController: NavHostController,
    modifier: Modifier = Modifier,
    startDestination: String = FEED_ROUTE,
) {
    NavHost(
        navController = navController,
        startDestination = startDestination,
        modifier = modifier,
    ) {
        // 각 feature 모듈이 제공한 NavGraphBuilder 확장 함수를 조립
        feedScreen(
            onArticleClick = { articleId ->
                // 추후 article 상세 화면으로 이동 (feature:detail 모듈)
            }
        )
        profileScreen(
            onBackClick = { navController.popBackStack() }
        )
        settingsScreen(
            onBackClick = { navController.popBackStack() }
        )
    }
}
```

이 패턴의 핵심은 `feature` 모듈이 자신의 **라우트 상수**와 **NavGraphBuilder 확장 함수**만 공개한다는 점입니다. `:app`이 이를 모아 전체 네비게이션 그래프를 완성합니다. Feature 모듈들은 서로를 전혀 알지 못합니다.

---

## 주의사항 및 팁

### 1. `api` 대신 `implementation`을 기본으로

`api` 키워드는 의존성을 전이(transitive)시킵니다. `feature:feed`가 `data:feed`를 `api`로 선언하면 `feature:feed`를 의존하는 `:app`도 `data:feed`의 내부 구현을 볼 수 있게 됩니다. 이는 캡슐화를 무너뜨리고 불필요한 재컴파일을 유발합니다. 내부 구현은 항상 `implementation`으로 숨기고, 공개 API를 위한 모듈(`:data:feed:api`)을 별도로 두는 패턴을 고려하세요.

### 2. 모듈 과분화 주의

모듈을 너무 잘게 나누면 Gradle 설정 단계(configuration phase)가 오래 걸립니다. 화면 하나당 모듈 하나보다는 **비슷한 도메인의 화면을 묶는** 것이 현실적입니다. 예를 들어 로그인·회원가입·비밀번호 재설정을 `feature:auth` 하나로 묶는 식입니다.

### 3. `:domain` 모듈은 순수 Kotlin으로

`:domain` 모듈에 `com.android.library` 플러그인을 적용하면 Android SDK 의존성이 생겨 JVM 단위 테스트가 Robolectric이나 에뮬레이터를 필요로 하게 됩니다. `kotlin("jvm")` 플러그인만 사용해 순수 Kotlin 모듈로 유지하면 테스트 속도가 10배 이상 빨라집니다.

### 4. Version Catalog로 의존성 일원화

`gradle/libs.versions.toml`에 모든 라이브러리 버전을 선언하고, Convention Plugin과 각 모듈 모두 이 카탈로그를 참조하면 버전 불일치 문제를 원천 차단할 수 있습니다.

### 5. `includeBuild` vs `composite build`

`build-logic`를 `includeBuild`로 포함하면 IDE에서 Convention Plugin 소스 코드를 직접 탐색하고 디버깅할 수 있습니다. `classpath` 방식으로 별도 배포하는 것보다 개발 경험이 훨씬 좋습니다.

### 6. Gradle 병렬 실행 활성화

```properties
# gradle.properties
org.gradle.parallel=true
org.gradle.caching=true
org.gradle.configuration-cache=true
```

Configuration Cache는 설정 단계 결과를 캐시해 두 번째 빌드부터 설정 단계를 건너뜁니다. 대형 멀티 모듈 프로젝트에서 체감 효과가 큽니다.

---

## 마치며

멀티 모듈 전환은 한 번에 이뤄지지 않습니다. 실용적인 접근은 기존 단일 모듈에서 **`:core:network`처럼 변경 빈도가 낮고 의존성이 명확한 모듈부터 분리**하는 것입니다. Convention Plugin을 먼저 구축해 두면 이후 모듈 추가 비용이 크게 낮아집니다. Now in Android 프로젝트는 이 구조의 가장 완성도 높은 실제 예시이므로, 그 구조를 꼼꼼히 분석하는 것을 강력히 권장합니다.

## 참고 자료
- [Guide to Android app modularization - Android Developers](https://developer.android.com/topic/modularization)
- [Common modularization patterns - Android Developers](https://developer.android.com/topic/modularization/patterns)
- [Now in Android - Google 공식 레퍼런스 앱](https://github.com/android/nowinandroid)
