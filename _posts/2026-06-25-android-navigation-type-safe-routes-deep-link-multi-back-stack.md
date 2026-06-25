---
layout: post
title: "Android Navigation Component 심화: Type-Safe Routes, Deep Link, Multi-Back Stack 완전 정복"
date: 2026-06-25
categories: [android, flutter]
tags: [android, navigation, jetpack, type-safe, deep-link, multi-back-stack, compose, kotlin]
---

Jetpack Navigation Component는 Android 앱의 화면 전환 로직을 표준화한 공식 프레임워크다. 2022년 2.4 버전에서 Multi-Back Stack 지원이 추가되고, 2024년 2.8 버전에서 Type-Safe Routes가 안정화되면서 Navigation은 완전히 새로운 수준으로 도약했다. 2026년 현재 최신 안정 버전은 **2.9.8**이다.

이 글에서는 기존 문자열 기반 라우팅의 한계를 극복한 Type-Safe Navigation, 딥링크 구현 전략, 그리고 하단 탭 UI의 핵심인 Multi-Back Stack을 실전 코드와 함께 완전히 정복한다.

---

## 왜 Type-Safe Routes가 필요한가

기존 Navigation은 라우트를 문자열로 정의했다.

```kotlin
// 기존 방식 — 오타가 있어도 컴파일 타임에 발견되지 않음
navController.navigate("profile/user123?showEdit=true")

// NavHost 정의
composable("profile/{userId}?showEdit={showEdit}") { backStackEntry ->
    val userId = backStackEntry.arguments?.getString("userId") ?: ""
    val showEdit = backStackEntry.arguments?.getBoolean("showEdit") ?: false
    ProfileScreen(userId = userId, showEdit = showEdit)
}
```

이 방식의 문제점은 다음과 같다.

- **런타임 크래시**: 오타나 인수 타입 불일치가 컴파일 타임에 잡히지 않는다.
- **장황한 인수 추출**: `arguments?.getString(...)` 같은 반복 코드가 필요하다.
- **Null 안전성 부재**: 모든 인수가 nullable로 추출된다.
- **리팩토링 취약**: 라우트 이름을 변경해도 IDE가 모든 참조를 자동 갱신하지 못한다.

Navigation 2.8의 Type-Safe Routes는 **Kotlin Serialization**을 활용해 이 모든 문제를 해결한다.

---

## Type-Safe Navigation 구현

### 의존성 추가

```kotlin
// build.gradle.kts (app)
plugins {
    id("org.jetbrains.kotlin.plugin.serialization")
}

dependencies {
    implementation("androidx.navigation:navigation-compose:2.8.9")
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.6.3")
}
```

### 라우트 클래스 정의

인수가 없는 라우트는 `object`, 인수가 있는 라우트는 `data class`로 정의한다.

```kotlin
import kotlinx.serialization.Serializable

// 인수 없는 라우트
@Serializable
object HomeRoute

@Serializable
object SearchRoute

// 인수 있는 라우트
@Serializable
data class ProfileRoute(
    val userId: String,
    val showEdit: Boolean = false
)

@Serializable
data class ArticleRoute(val articleId: Int)
```

`@Serializable` 어노테이션 하나로 Kotlin Serialization이 라우트 직렬화/역직렬화를 자동 처리한다.

### NavHost 설정 — 완전한 예제

```kotlin
@Composable
fun AppNavHost(
    navController: NavHostController,
    modifier: Modifier = Modifier
) {
    NavHost(
        navController = navController,
        startDestination = HomeRoute,
        modifier = modifier
    ) {
        composable<HomeRoute> {
            HomeScreen(
                onOpenProfile = { userId ->
                    navController.navigate(ProfileRoute(userId = userId))
                },
                onOpenArticle = { articleId ->
                    navController.navigate(ArticleRoute(articleId = articleId))
                }
            )
        }

        composable<ProfileRoute> { backStackEntry ->
            // toRoute()가 역직렬화를 자동 처리
            val profile: ProfileRoute = backStackEntry.toRoute()
            ProfileScreen(
                userId = profile.userId,
                showEdit = profile.showEdit,
                onBack = { navController.popBackStack() }
            )
        }

        composable<ArticleRoute> { backStackEntry ->
            val article: ArticleRoute = backStackEntry.toRoute()
            ArticleDetailScreen(
                articleId = article.articleId,
                onBack = { navController.popBackStack() }
            )
        }
    }
}
```

`backStackEntry.toRoute<T>()`는 백 스택 항목의 인수를 지정된 타입으로 안전하게 역직렬화한다. 타입이 맞지 않으면 **빌드 타임**에 오류가 발생한다.

---

## Deep Link 구현

딥링크는 외부(웹 브라우저, 다른 앱, 알림)에서 앱의 특정 화면으로 직접 진입할 수 있게 해주는 메커니즘이다. Type-Safe Navigation과 결합하면 URI 파라미터가 라우트 클래스의 프로퍼티에 자동으로 매핑된다.

### AndroidManifest.xml 설정

```xml
<activity
    android:name=".MainActivity"
    android:exported="true">

    <!-- 커스텀 스킴 딥링크 -->
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="ducktudy" android:host="profile" />
    </intent-filter>

    <!-- Android App Links (HTTPS) — autoVerify로 인증 -->
    <intent-filter android:autoVerify="true">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="https" android:host="ducktudy.com" />
    </intent-filter>
</activity>
```

### NavHost에 딥링크 연결

```kotlin
composable<ProfileRoute>(
    deepLinks = listOf(
        // ducktudy://profile?userId=abc&showEdit=true
        navDeepLink<ProfileRoute>(basePath = "ducktudy://profile"),
        // https://ducktudy.com/profile?userId=abc&showEdit=true
        navDeepLink<ProfileRoute>(basePath = "https://ducktudy.com/profile")
    )
) { backStackEntry ->
    val profile: ProfileRoute = backStackEntry.toRoute()
    ProfileScreen(userId = profile.userId, showEdit = profile.showEdit)
}

composable<ArticleRoute>(
    deepLinks = listOf(
        navDeepLink<ArticleRoute>(basePath = "https://ducktudy.com/article")
    )
) { backStackEntry ->
    val article: ArticleRoute = backStackEntry.toRoute()
    ArticleDetailScreen(articleId = article.articleId)
}
```

`navDeepLink<T>(basePath = "...")` 형식에서 `T`의 프로퍼티 이름이 쿼리 파라미터 이름으로 자동 매핑된다. `ArticleRoute(articleId = 42)`는 `https://ducktudy.com/article?articleId=42`로 변환된다.

### ADB로 딥링크 테스트

```bash
# 커스텀 스킴
adb shell am start -W -a android.intent.action.VIEW \
    -d "ducktudy://profile?userId=user42&showEdit=true" \
    com.example.myapp

# HTTPS 딥링크
adb shell am start -W -a android.intent.action.VIEW \
    -d "https://ducktudy.com/article?articleId=99" \
    com.example.myapp
```

### 딥링크와 인증 흐름

인증이 필요한 화면에 딥링크로 진입했을 때, 로그인 후 원래 목적지로 리다이렉트해야 하는 경우가 많다. `savedStateHandle`을 활용한 패턴이다.

```kotlin
@Composable
fun ProfileScreen(userId: String) {
    val isLoggedIn = /* 인증 상태 확인 */
    if (!isLoggedIn) {
        // 로그인 화면으로 이동하면서 현재 라우트 정보 전달
        LaunchedEffect(Unit) {
            navController.navigate(LoginRoute) {
                // 로그인 성공 후 돌아올 목적지 기록
                navController.currentBackStackEntry
                    ?.savedStateHandle
                    ?.set("pendingUserId", userId)
            }
        }
        return
    }
    // 실제 화면 렌더링
}
```

---

## Multi-Back Stack

Multi-Back Stack은 하단 내비게이션 바처럼 여러 탭을 가진 UI에서 **각 탭이 독립적인 백 스택을 유지**하게 해준다. 탭 A에서 여러 화면을 탐색하다 탭 B로 전환해도, 다시 탭 A로 돌아오면 이전 탐색 상태가 그대로 복원된다.

### 핵심 옵션 이해

| 옵션 | 역할 |
|------|------|
| `saveState = true` | 팝할 때 현재 백 스택 상태를 NavController에 저장 |
| `restoreState = true` | 동일 라우트의 저장된 상태가 있으면 복원 |
| `launchSingleTop = true` | 동일 화면이 스택 최상위에 있으면 재생성하지 않음 |

### 완전한 Multi-Back Stack 구현

```kotlin
// 탭 라우트 정의
@Serializable object HomeTabRoute
@Serializable object SearchTabRoute
@Serializable object ProfileTabRoute

// 각 탭의 시작 화면 라우트
@Serializable object HomeRoute
@Serializable object SearchRoute
@Serializable data class MyProfileRoute(val userId: String = "me")

sealed class TopLevelDestination(
    val route: Any,
    val label: String,
    val icon: ImageVector
) {
    data object Home : TopLevelDestination(HomeTabRoute, "홈", Icons.Default.Home)
    data object Search : TopLevelDestination(SearchTabRoute, "검색", Icons.Default.Search)
    data object Profile : TopLevelDestination(ProfileTabRoute, "프로필", Icons.Default.Person)
}

@Composable
fun MainScreen() {
    val navController = rememberNavController()
    val navBackStackEntry by navController.currentBackStackEntryAsState()
    val currentDestination = navBackStackEntry?.destination

    val topLevelDestinations = listOf(
        TopLevelDestination.Home,
        TopLevelDestination.Search,
        TopLevelDestination.Profile
    )

    Scaffold(
        bottomBar = {
            NavigationBar {
                topLevelDestinations.forEach { destination ->
                    val isSelected = currentDestination?.hierarchy?.any { navDest ->
                        navDest.hasRoute(destination.route::class)
                    } == true

                    NavigationBarItem(
                        selected = isSelected,
                        onClick = {
                            navController.navigate(destination.route) {
                                // 시작 목적지까지 팝하면서 상태 저장
                                popUpTo(navController.graph.findStartDestination().id) {
                                    saveState = true
                                }
                                launchSingleTop = true
                                restoreState = true
                            }
                        },
                        icon = { Icon(destination.icon, contentDescription = destination.label) },
                        label = { Text(destination.label) }
                    )
                }
            }
        }
    ) { innerPadding ->
        NavHost(
            navController = navController,
            startDestination = HomeTabRoute,
            modifier = Modifier.padding(innerPadding)
        ) {
            // 홈 탭 중첩 그래프
            navigation<HomeTabRoute>(startDestination = HomeRoute) {
                composable<HomeRoute> {
                    HomeScreen(
                        onOpenArticle = { id ->
                            navController.navigate(ArticleRoute(articleId = id))
                        }
                    )
                }
                composable<ArticleRoute> { backStackEntry ->
                    val route: ArticleRoute = backStackEntry.toRoute()
                    ArticleDetailScreen(articleId = route.articleId)
                }
            }

            // 검색 탭 중첩 그래프
            navigation<SearchTabRoute>(startDestination = SearchRoute) {
                composable<SearchRoute> { SearchScreen() }
            }

            // 프로필 탭 중첩 그래프
            navigation<ProfileTabRoute>(startDestination = MyProfileRoute()) {
                composable<MyProfileRoute> { backStackEntry ->
                    val route: MyProfileRoute = backStackEntry.toRoute()
                    MyProfileScreen(userId = route.userId)
                }
            }
        }
    }
}
```

이 구현에서 홈 탭에서 기사 상세 화면까지 탐색한 뒤 검색 탭으로 전환하고 다시 홈 탭으로 돌아오면, 기사 상세 화면이 그대로 유지된다.

---

## 그래프 범위 ViewModel 공유

Navigation은 각 백 스택 항목에 독립적인 `ViewModelStoreOwner`를 제공한다. 같은 탭 내 여러 화면이 ViewModel을 공유하려면 **중첩 그래프의 백 스택 항목**을 ViewModelStoreOwner로 지정한다.

```kotlin
@Composable
fun ArticleDetailScreen() {
    val navBackStackEntry = rememberUpdatedState(
        navController.currentBackStackEntryAsState().value
    )

    // HomeTabRoute 그래프 범위의 ViewModel
    val homeGraphEntry = remember(navBackStackEntry.value) {
        navController.getBackStackEntry<HomeTabRoute>()
    }
    val sharedViewModel: HomeSharedViewModel = hiltViewModel(homeGraphEntry)

    // sharedViewModel은 홈 탭 내 모든 화면이 동일 인스턴스를 공유
}
```

---

## 주의사항 및 팁

**1. `@Serializable` 어노테이션 빠뜨리지 않기**  
라우트 클래스에 `@Serializable`이 없으면 런타임에 `SerializationException`이 발생한다. 빌드 타임 경고가 표시되지 않으므로 습관적으로 확인해야 한다.

**2. `popUpTo` 대상을 `findStartDestination().id`로 지정하라**  
`navController.graph.startDestinationId` 대신 `findStartDestination().id`를 사용해야 중첩 그래프에서도 올바르게 동작한다.

**3. Android App Links는 도메인 소유권 인증이 필요하다**  
`android:autoVerify="true"`를 사용하는 HTTPS 딥링크는 서버에 `.well-known/assetlinks.json` 파일을 배포해야 OS가 앱을 기본 핸들러로 등록한다. 이 파일이 없으면 브라우저가 열린다.

**4. Navigation 단위 테스트**  
Compose 환경에서 탐색 로직을 테스트할 때는 `TestNavHostController`를 주입하거나 `createComposeRule()`과 함께 `NavHostController`를 모킹한다.

```kotlin
@Test
fun navigateToProfile_updatesCurrentDestination() {
    composeTestRule.setContent {
        val navController = rememberNavController()
        AppNavHost(navController = navController)
        LaunchedEffect(Unit) {
            navController.navigate(ProfileRoute(userId = "test"))
        }
    }
    // UI 검증
    composeTestRule.onNodeWithText("test").assertIsDisplayed()
}
```

**5. 기존 코드 마이그레이션**  
문자열 기반 라우트에서 Type-Safe 라우트로 한 번에 전환하기 어렵다면, `NavDeepLinkRequest`를 활용해 단계적으로 마이그레이션할 수 있다. 공식 [마이그레이션 가이드](https://developer.android.com/guide/navigation/type-safe-destinations)에 상세한 절차가 나와 있다.

---

## 마무리

Navigation 2.8의 Type-Safe Routes는 단순한 편의 기능이 아니라 Android 앱의 탐색 로직 안정성을 근본적으로 개선하는 패러다임 전환이다. `@Serializable`로 라우트를 정의하고, `toRoute()`로 안전하게 역직렬화하며, `navDeepLink<T>(basePath = ...)`로 딥링크를 연결하는 패턴을 익히면 런타임 크래시의 주요 원인 하나를 완전히 제거할 수 있다. Multi-Back Stack과 결합하면 현대적인 탭 UI의 사용자 경험도 자연스럽게 구현된다.

## 참고 자료
- [Type safety in Kotlin DSL and Navigation Compose](https://developer.android.com/guide/navigation/design/type-safety)
- [Support multiple back stacks](https://developer.android.com/guide/navigation/backstack/multi-back-stacks)
- [Create a deep link for a destination](https://developer.android.com/guide/navigation/design/deep-link)
- [Navigation 릴리스 노트](https://developer.android.com/jetpack/androidx/releases/navigation)
