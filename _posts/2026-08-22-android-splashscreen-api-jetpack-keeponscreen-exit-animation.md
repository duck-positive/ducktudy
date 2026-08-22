---
layout: post
title: "Android SplashScreen API 심화: Android 12+ 스플래시 화면 표준화와 Jetpack SplashScreen 완전 정복"
date: 2026-08-22
categories: [android]
tags: [android, splashscreen, jetpack, kotlin, animation, ux]
---

Android 12(API 31)가 출시되면서 스플래시 화면 구현 방식이 완전히 바뀌었습니다. 기존에 개발자들이 즐겨 쓰던 `windowBackground` 트릭이나 별도 런처 Activity 방식은 Android 12 이상에서 이중 스플래시 화면 문제를 일으키거나 시스템 기본값에 덮어씌워집니다. 이 글에서는 새로운 SplashScreen API의 내부 동작을 이해하고, Jetpack SplashScreen 라이브러리로 API 23까지 하위 호환성을 유지하면서 아이콘 애니메이션, 화면 유지 조건, 종료 애니메이션까지 완전히 제어하는 방법을 코드와 함께 깊이 설명합니다.

---

## 1. 개념 설명: Android 12 이전과 이후

### Android 11 이하의 스플래시 화면

과거 방식은 두 가지가 주로 쓰였습니다.

**방식 1 — windowBackground 트릭**
Activity 테마에 `windowBackground`로 이미지나 색상을 지정해 첫 프레임이 그려지기 전까지 배경만 보여주는 방법입니다. 별도 Activity 없이 빠르게 동작하지만, 복잡한 애니메이션 처리가 어렵습니다.

```xml
<!-- res/values/themes.xml (구 방식) -->
<style name="Theme.App.SplashScreen" parent="Theme.AppCompat.NoActionBar">
    <item name="android:windowBackground">@drawable/splash_background</item>
</style>
```

**방식 2 — 전용 SplashActivity**
별도 Activity를 만들어 브랜딩 화면을 구성한 뒤 MainActivity로 전환합니다. 자유도가 높지만 Android 12+ 환경에서는 시스템이 SplashActivity 위에 또 한 번 시스템 스플래시 화면을 덮어 씌우는 문제가 발생합니다.

### Android 12+의 새로운 구조

Android 12부터 시스템은 **모든 앱 콜드·웜 스타트**에 강제로 스플래시 화면을 삽입합니다. 이 스플래시 화면은 세 가지 요소로 구성됩니다.

| 요소 | 설명 |
|---|---|
| 윈도우 배경 (`windowSplashScreenBackground`) | 단색 또는 색상 드로어블 |
| 아이콘 (`windowSplashScreenAnimatedIcon`) | 적응형 아이콘 또는 AnimatedVectorDrawable |
| 아이콘 배경 (`windowSplashScreenIconBackgroundColor`) | 아이콘 뒤쪽 원형 영역 색상 |

핵심은 앱 아이콘이 반드시 **240dp × 240dp 영역** 안에 표시되어야 하며, 안전 영역은 내부 **160dp** 입니다. 아이콘 크기나 여백이 이 규격을 벗어나면 잘려 보입니다.

---

## 2. 왜 필요한가: 마이그레이션하지 않으면 생기는 문제

Android 12 이상 기기에서 구 방식 앱을 실행하면:

1. **이중 스플래시 화면**: 시스템 스플래시 화면이 먼저 보인 뒤, 앱 자체 SplashActivity가 또 한 번 나타나 UX가 지저분해집니다.
2. **앱 아이콘 자동 적용**: 개발자가 의도한 커스텀 이미지 대신 런처 아이콘이 흰 배경에 표시됩니다.
3. **진입 애니메이션 누락**: `into-app motion`(앱으로 전환되는 확대 애니메이션)이 적용되지 않습니다.

**Jetpack SplashScreen 라이브러리**(`androidx.core:core-splashscreen`)는 이 API를 API 23(Android 6.0)까지 백포팅해 주므로, 이 라이브러리 하나로 모든 타겟 범위를 커버할 수 있습니다.

---

## 3. 실제 구현 예제

### 3-1. 기본 설정: 의존성·테마·진입점

**build.gradle.kts**:
```kotlin
dependencies {
    implementation("androidx.core:core-splashscreen:1.0.1")
}
```

**res/values/themes.xml** — 스플래시 전용 테마를 선언합니다:
```xml
<resources>
    <!-- 앱 기본 테마에서 상속 -->
    <style name="Theme.App.Starting" parent="Theme.SplashScreen">
        <!-- 배경 색상 -->
        <item name="windowSplashScreenBackground">@color/splash_background</item>
        <!-- AnimatedVectorDrawable 또는 일반 드로어블 -->
        <item name="windowSplashScreenAnimatedIcon">@drawable/ic_splash_animated</item>
        <!-- 애니메이션 재생 시간 (최대 1000ms, 기본 0ms) -->
        <item name="windowSplashScreenAnimationDuration">800</item>
        <!-- 스플래시 이후 전환할 실제 앱 테마 -->
        <item name="postSplashScreenTheme">@style/Theme.MyApp</item>
    </style>
</resources>
```

**AndroidManifest.xml** — MainActivity에 스플래시 테마 지정:
```xml
<activity
    android:name=".MainActivity"
    android:theme="@style/Theme.App.Starting"
    android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>
```

**MainActivity.kt** — `installSplashScreen()`은 반드시 `super.onCreate()` 이전에 호출해야 합니다:
```kotlin
class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        // 반드시 super.onCreate() 이전에 호출
        val splashScreen = installSplashScreen()

        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
    }
}
```

`installSplashScreen()`은 내부적으로 `window.requestFeature(Window.FEATURE_ACTIVITY_TRANSITIONS)`를 호출하고, 현재 테마를 `postSplashScreenTheme`으로 교체하며, 스플래시 Window를 Activity Window 위에 올려 놓습니다.

---

### 3-2. 핵심 구현: 데이터 로딩 중 스플래시 화면 유지

앱이 초기 데이터를 네트워크·DB에서 가져오는 동안 스플래시 화면을 유지해야 할 때 `setKeepOnScreenCondition`을 사용합니다.

```kotlin
class MainActivity : AppCompatActivity() {

    private val viewModel: MainViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        val splashScreen = installSplashScreen()

        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        // 초기화가 완료될 때까지 스플래시 화면 유지
        splashScreen.setKeepOnScreenCondition {
            // true → 스플래시 유지, false → 스플래시 종료
            viewModel.isLoading.value
        }

        // 상태 관찰
        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect { state ->
                    when (state) {
                        is UiState.Ready -> renderContent(state.data)
                        is UiState.Error -> showError(state.message)
                        else -> Unit
                    }
                }
            }
        }
    }
}
```

`setKeepOnScreenCondition`은 매 프레임마다 람다를 호출합니다. 따라서 람다 내부에서 **비용이 높은 연산(IO, 복잡한 계산)을 수행하면 안 됩니다**. StateFlow나 LiveData의 현재 값을 읽는 수준으로 가볍게 유지하세요.

**ViewModel 예시**:
```kotlin
class MainViewModel : ViewModel() {

    private val _isLoading = MutableStateFlow(true)
    val isLoading: StateFlow<Boolean> = _isLoading.asStateFlow()

    sealed interface UiState {
        data object Loading : UiState
        data class Ready(val data: AppData) : UiState
        data class Error(val message: String) : UiState
    }

    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()

    init {
        viewModelScope.launch {
            try {
                val data = repository.loadInitialData()
                _uiState.value = UiState.Ready(data)
            } catch (e: Exception) {
                _uiState.value = UiState.Error(e.message ?: "Unknown error")
            } finally {
                _isLoading.value = false  // 성공·실패 모두 스플래시 해제
            }
        }
    }
}
```

---

### 3-3. 고급 구현: 커스텀 종료 애니메이션

기본 종료 애니메이션(아이콘이 축소되며 사라지는 효과) 대신 직접 애니메이션을 구성하려면 `setOnExitAnimationListener`를 사용합니다.

```kotlin
splashScreen.setOnExitAnimationListener { splashScreenView ->
    // splashScreenView: SplashScreenViewProvider
    // .view       → 스플래시 최상위 View
    // .iconView   → 아이콘 View (ImageView)
    // .iconAnimationStartMillis → 아이콘 애니메이션 시작 시각
    // .iconAnimationDurationMillis → 아이콘 애니메이션 지속 시간

    // 아이콘 애니메이션이 끝난 뒤 커스텀 전환
    val remainingDuration = (
        splashScreenView.iconAnimationStartMillis +
        splashScreenView.iconAnimationDurationMillis -
        SystemClock.uptimeMillis()
    ).coerceAtLeast(0L)

    // 슬라이드 업 & 페이드 아웃
    val slideUp = ObjectAnimator.ofFloat(
        splashScreenView.view,
        View.TRANSLATION_Y,
        0f,
        -splashScreenView.view.height.toFloat()
    )
    val fadeOut = ObjectAnimator.ofFloat(
        splashScreenView.view,
        View.ALPHA,
        1f, 0f
    )

    AnimatorSet().apply {
        playTogether(slideUp, fadeOut)
        interpolator = AnticipateInterpolator()
        duration = 400L
        startDelay = remainingDuration
        doOnEnd { splashScreenView.remove() }  // 반드시 remove() 호출
        start()
    }
}
```

> **주의**: `setOnExitAnimationListener`를 등록하면 기본 종료 애니메이션이 완전히 비활성화됩니다. 리스너 내부에서 반드시 `splashScreenView.remove()`를 호출해야 스플래시 화면이 사라집니다. 호출하지 않으면 스플래시가 영구 잔류합니다.

---

## 4. 주의사항 및 팁

### 아이콘 크기 규격 준수

| 항목 | 규격 |
|---|---|
| 전체 영역 | 240dp × 240dp |
| 안전 영역 (visible area) | 160dp × 160dp (중앙) |
| 배경 원 직경 | 240dp |

적응형 아이콘(`<adaptive-icon>`)을 사용하면 시스템이 자동으로 마스킹과 크기 조정을 처리합니다. 레거시 아이콘을 쓰면 시스템이 흰 배경 원 안에 삽입하므로 의도치 않은 여백이 생깁니다.

### AnimatedVectorDrawable 제한

`windowSplashScreenAnimatedIcon`에 AnimatedVectorDrawable을 사용할 때:
- **최대 재생 시간은 1,000ms** 입니다. 더 길게 설정해도 잘립니다.
- `windowSplashScreenAnimationDuration`은 시스템에 "이 시간 동안 재생된다"고 알려주는 힌트이지, 실제 재생을 강제하지는 않습니다.
- API 31 미만에서는 AnimatedVectorDrawable 애니메이션이 재생되지 않고 첫 프레임만 표시됩니다(Jetpack 라이브러리 사용 시).

### 다크 모드 대응

`res/values-night/themes.xml`에 별도 테마를 정의해 다크 모드에서 배경색과 아이콘 배경색을 별도 지정할 수 있습니다:

```xml
<!-- res/values-night/themes.xml -->
<style name="Theme.App.Starting" parent="Theme.SplashScreen">
    <item name="windowSplashScreenBackground">@color/splash_background_dark</item>
    <item name="windowSplashScreenAnimatedIcon">@drawable/ic_splash_animated_dark</item>
    <item name="postSplashScreenTheme">@style/Theme.MyApp.Dark</item>
</style>
```

### Compose 앱에서의 사용

Jetpack Compose를 사용하는 경우에도 동일한 패턴이 적용됩니다. `ComponentActivity`는 `AppCompatActivity`의 하위 클래스이므로 `installSplashScreen()`을 그대로 호출할 수 있습니다. 다만 `setContentView` 대신 `setContent`를 사용하면 됩니다.

### 구성 변경(Configuration Change) 처리

화면 회전 같은 구성 변경이 발생하면 Activity가 재시작됩니다. `installSplashScreen()`은 Activity 재시작 시에도 호출되지만, `savedInstanceState`가 null이 아닌 경우(재시작)에는 스플래시 화면을 표시하지 않도록 시스템이 내부적으로 처리합니다. 별도 분기 처리를 할 필요가 없습니다.

### 테스트 방법

```bash
# 앱 프로세스를 강제 종료 후 콜드 스타트 시뮬레이션
adb shell am force-stop com.example.myapp
adb shell monkey -p com.example.myapp -c android.intent.category.LAUNCHER 1
```

에뮬레이터보다 실기기에서 테스트하는 것이 권장됩니다. 에뮬레이터는 스플래시 화면 타이밍이 다를 수 있습니다.

---

## 정리

Android 12+ SplashScreen API는 단순히 "화면 하나 추가하는 것"이 아니라, 앱 진입 UX의 표준을 시스템 레벨에서 강제하는 구조적 변화입니다. 핵심 포인트를 정리하면:

- **`installSplashScreen()`** — `super.onCreate()` 이전 호출 필수
- **`setKeepOnScreenCondition`** — 데이터 로딩 중 스플래시 유지, 람다는 매 프레임 실행되므로 가볍게
- **`setOnExitAnimationListener`** — 완전 커스텀 종료 애니메이션, `remove()` 호출 필수
- **아이콘 240dp / 안전 영역 160dp** — 규격 위반 시 잘림 발생
- **Jetpack SplashScreen 라이브러리** — API 23까지 하위 호환, 프로덕션 표준

기존 SplashActivity나 windowBackground 방식을 사용 중이라면 이 구조로 마이그레이션해 Android 12+ 환경에서의 이중 스플래시 문제를 해소하고, 일관된 진입 경험을 제공할 수 있습니다.

## 참고 자료
- [Splash screens | Android Developers](https://developer.android.com/develop/ui/views/launch/splash-screen)
- [Migrate your splash screen implementation to Android 12 and later](https://developer.android.com/develop/ui/views/launch/splash-screen/migrate)
- [androidx.core.splashscreen API Reference](https://developer.android.com/reference/androidx/core/splashscreen/package-summary)
