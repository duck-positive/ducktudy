---
layout: post
title: "Android Jetpack Compose UI 테스팅 심화: composeTestRule·Semantics Matcher·Screenshot Test 완전 정복"
date: 2026-11-17
categories: [android, flutter]
tags: [android, jetpack-compose, testing, ui-test, semantics, screenshot-test, roborazzi, paparazzi, composeTestRule, kotlin]
---

Jetpack Compose로 UI를 개발하면서 가장 많이 간과되는 부분 중 하나가 바로 **테스팅**입니다. View 기반의 Espresso와 달리, Compose는 완전히 새로운 테스팅 패러다임인 **Semantics Tree** 기반의 API를 사용합니다. 이 글에서는 `composeTestRule`의 작동 원리부터 커스텀 Semantics Matcher 작성, 그리고 Roborazzi를 활용한 Screenshot Test까지 실무에서 바로 적용 가능한 Compose 테스팅 전략을 완전히 정복합니다.

---

## 왜 Compose UI 테스팅이 중요한가

기존 View 시스템에서는 Espresso가 `ViewId`를 기반으로 UI 요소를 찾았습니다. 하지만 Compose는 XML 레이아웃도, ViewId도 없습니다. 모든 UI는 코틀린 함수로 선언되며, 이를 테스트하려면 **Semantics Tree**라는 새로운 개념을 이해해야 합니다.

Semantics Tree는 Compose UI가 렌더링될 때 병렬로 생성되는 트리 구조로, 접근성 서비스(TalkBack, 스위치 제어)와 테스팅 프레임워크 모두 이 트리를 통해 UI와 상호작용합니다. 즉, **접근성을 잘 구현한 앱은 테스트하기도 쉽다**는 강력한 상관관계가 있습니다.

Compose 테스팅이 반드시 필요한 이유는 다음과 같습니다:

- **리그레션 방지**: 복잡한 상태 머신(StateFlow, ViewModel)을 가진 Compose UI는 미묘한 상태 변화로 인한 렌더링 버그가 발생하기 쉽습니다.
- **리팩터링 안전망**: Composable 함수를 분리·통합할 때 동작이 보존되는지 보장합니다.
- **스냅샷 테스트**: UI 변경사항을 픽셀 단위로 비교하여 의도치 않은 시각적 변화를 CI에서 즉시 감지합니다.
- **개발 속도 향상**: 에뮬레이터 없이 JVM 위에서 빠르게 UI 테스트를 실행할 수 있습니다.

---

## 개념 이해: Semantics Tree와 테스트 계층

Compose UI 테스트가 작동하는 방식을 그림으로 표현하면 다음과 같습니다:

```
Compose UI 코드
     ↓ (렌더링)
UI 계층 (Widget Tree)
     ↓ (병렬 생성)
Semantics Tree  ←── 테스트 프레임워크가 여기에 접근
     ↓
composeTestRule.onNode(matcher) → SemanticsNodeInteraction
```

`SemanticsNodeInteraction`은 Compose 테스팅의 핵심 클래스로, 특정 노드를 찾아 액션을 수행하거나 단언(assertion)을 적용할 수 있게 해줍니다.

Semantics Tree에서 노드를 찾는 방법은 크게 세 가지입니다:

1. **`onNodeWithText()`**: 텍스트 내용으로 찾기
2. **`onNodeWithContentDescription()`**: 접근성 설명으로 찾기
3. **`onNodeWithTag()`**: `Modifier.testTag()`로 지정한 태그로 찾기

---

## 프로젝트 설정

`build.gradle.kts`에 다음 의존성을 추가합니다:

```kotlin
// app/build.gradle.kts
dependencies {
    // Compose UI 테스팅의 핵심 — JUnit4 Rule 포함
    androidTestImplementation("androidx.compose.ui:ui-test-junit4:1.7.5")

    // createComposeRule() 사용 시 필요한 테스트 매니페스트
    debugImplementation("androidx.compose.ui:ui-test-manifest:1.7.5")

    // Screenshot 테스팅 (Roborazzi)
    testImplementation("io.github.takahirom.roborazzi:roborazzi:1.26.0")
    testImplementation("io.github.takahirom.roborazzi:roborazzi-compose:1.26.0")
    testImplementation("org.robolectric:robolectric:4.13")
}
```

Roborazzi 플러그인도 추가합니다:

```kotlin
// build.gradle.kts (project level)
plugins {
    id("io.github.takahirom.roborazzi") version "1.26.0" apply false
}

// app/build.gradle.kts
plugins {
    id("io.github.takahirom.roborazzi")
}
```

---

## 실제 구현 예제 1: composeTestRule과 Semantics Matcher 완전 활용

먼저 테스트할 간단한 로그인 화면을 만들겠습니다:

```kotlin
// LoginScreen.kt
@Composable
fun LoginScreen(
    uiState: LoginUiState,
    onEmailChange: (String) -> Unit,
    onPasswordChange: (String) -> Unit,
    onLoginClick: () -> Unit
) {
    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(24.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Text(
            text = "환영합니다",
            style = MaterialTheme.typography.headlineMedium
        )

        Spacer(modifier = Modifier.height(32.dp))

        OutlinedTextField(
            value = uiState.email,
            onValueChange = onEmailChange,
            label = { Text("이메일") },
            modifier = Modifier
                .fillMaxWidth()
                .testTag("email_field"),  // 테스트 태그 필수
            isError = uiState.emailError != null,
            supportingText = uiState.emailError?.let { { Text(it) } }
        )

        Spacer(modifier = Modifier.height(16.dp))

        OutlinedTextField(
            value = uiState.password,
            onValueChange = onPasswordChange,
            label = { Text("비밀번호") },
            visualTransformation = PasswordVisualTransformation(),
            modifier = Modifier
                .fillMaxWidth()
                .testTag("password_field"),
            keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Password)
        )

        Spacer(modifier = Modifier.height(32.dp))

        Button(
            onClick = onLoginClick,
            enabled = uiState.isLoginEnabled,
            modifier = Modifier
                .fillMaxWidth()
                .testTag("login_button")
        ) {
            if (uiState.isLoading) {
                CircularProgressIndicator(
                    modifier = Modifier.size(20.dp),
                    color = MaterialTheme.colorScheme.onPrimary
                )
            } else {
                Text("로그인")
            }
        }

        if (uiState.errorMessage != null) {
            Spacer(modifier = Modifier.height(16.dp))
            Text(
                text = uiState.errorMessage,
                color = MaterialTheme.colorScheme.error,
                modifier = Modifier.testTag("error_message")
            )
        }
    }
}

data class LoginUiState(
    val email: String = "",
    val password: String = "",
    val emailError: String? = null,
    val isLoading: Boolean = false,
    val errorMessage: String? = null,
    val isLoginEnabled: Boolean = false
)
```

이제 이 화면에 대한 포괄적인 테스트를 작성합니다:

```kotlin
// LoginScreenTest.kt
@RunWith(AndroidJUnit4::class)
class LoginScreenTest {

    @get:Rule
    val composeTestRule = createComposeRule()

    // --- 초기 상태 테스트 ---

    @Test
    fun `초기 상태에서 로그인 버튼이 비활성화되어 있어야 한다`() {
        composeTestRule.setContent {
            LoginScreen(
                uiState = LoginUiState(),
                onEmailChange = {},
                onPasswordChange = {},
                onLoginClick = {}
            )
        }

        composeTestRule
            .onNodeWithTag("login_button")
            .assertIsNotEnabled()
    }

    @Test
    fun `환영합니다 텍스트가 화면에 표시되어야 한다`() {
        composeTestRule.setContent {
            LoginScreen(
                uiState = LoginUiState(),
                onEmailChange = {},
                onPasswordChange = {},
                onLoginClick = {}
            )
        }

        composeTestRule
            .onNodeWithText("환영합니다")
            .assertIsDisplayed()
    }

    // --- 사용자 인터랙션 테스트 ---

    @Test
    fun `이메일 입력 시 콜백이 올바르게 호출된다`() {
        var capturedEmail = ""

        composeTestRule.setContent {
            LoginScreen(
                uiState = LoginUiState(),
                onEmailChange = { capturedEmail = it },
                onPasswordChange = {},
                onLoginClick = {}
            )
        }

        composeTestRule
            .onNodeWithTag("email_field")
            .performTextInput("test@example.com")

        assertThat(capturedEmail).isEqualTo("test@example.com")
    }

    @Test
    fun `로그인 버튼 클릭 시 콜백이 호출된다`() {
        var loginClicked = false

        composeTestRule.setContent {
            LoginScreen(
                uiState = LoginUiState(isLoginEnabled = true),
                onEmailChange = {},
                onPasswordChange = {},
                onLoginClick = { loginClicked = true }
            )
        }

        composeTestRule
            .onNodeWithTag("login_button")
            .performClick()

        assertThat(loginClicked).isTrue()
    }

    // --- 상태별 UI 렌더링 테스트 ---

    @Test
    fun `로딩 중에는 버튼 내 ProgressIndicator가 표시된다`() {
        composeTestRule.setContent {
            LoginScreen(
                uiState = LoginUiState(isLoading = true, isLoginEnabled = true),
                onEmailChange = {},
                onPasswordChange = {},
                onLoginClick = {}
            )
        }

        // ProgressIndicator는 contentDescription이 없으므로
        // 버튼이 enabled 상태인지로 간접 확인하거나 semantics를 직접 활용
        composeTestRule
            .onNodeWithTag("login_button")
            .assertIsEnabled()

        // 텍스트 "로그인"이 보이지 않아야 함 (로딩 중)
        composeTestRule
            .onNodeWithText("로그인")
            .assertDoesNotExist()
    }

    @Test
    fun `에러 메시지가 있을 때 화면에 표시된다`() {
        val errorMessage = "이메일 또는 비밀번호가 올바르지 않습니다"

        composeTestRule.setContent {
            LoginScreen(
                uiState = LoginUiState(errorMessage = errorMessage),
                onEmailChange = {},
                onPasswordChange = {},
                onLoginClick = {}
            )
        }

        composeTestRule
            .onNodeWithTag("error_message")
            .assertIsDisplayed()
            .assertTextEquals(errorMessage)
    }

    @Test
    fun `이메일 에러가 있을 때 에러 텍스트가 필드 아래 표시된다`() {
        composeTestRule.setContent {
            LoginScreen(
                uiState = LoginUiState(emailError = "올바른 이메일 형식이 아닙니다"),
                onEmailChange = {},
                onPasswordChange = {},
                onLoginClick = {}
            )
        }

        // useUnmergedTree=true 로 merged node 안의 텍스트도 탐색
        composeTestRule
            .onNodeWithText("올바른 이메일 형식이 아닙니다", useUnmergedTree = true)
            .assertIsDisplayed()
    }

    // --- 커스텀 Semantics Matcher ---

    @Test
    fun `커스텀 matcher로 버튼 활성화 상태를 검증한다`() {
        composeTestRule.setContent {
            LoginScreen(
                uiState = LoginUiState(
                    email = "valid@email.com",
                    password = "password123",
                    isLoginEnabled = true
                ),
                onEmailChange = {},
                onPasswordChange = {},
                onLoginClick = {}
            )
        }

        // hasTestTag + isEnabled를 AND로 결합한 커스텀 matcher
        composeTestRule
            .onNode(
                hasTestTag("login_button") and isEnabled()
            )
            .assertExists()
    }
}
```

---

## 실제 구현 예제 2: Roborazzi를 활용한 Screenshot Test

Screenshot Test는 UI의 시각적 외형을 "황금 이미지(Golden Image)"로 저장해두고, 코드 변경 후 렌더링 결과가 달라졌는지 픽셀 단위로 비교합니다. CI 파이프라인에 통합하면 디자인 시스템 위반이나 의도치 않은 UI 변경을 자동으로 감지할 수 있습니다.

```kotlin
// LoginScreenScreenshotTest.kt
@RunWith(RobolectricTestRunner::class)
@GraphicsMode(GraphicsMode.Mode.NATIVE)
@Config(sdk = [33], qualifiers = "w411dp-h891dp-480dpi")
class LoginScreenScreenshotTest {

    @get:Rule
    val composeTestRule = createComposeRule()

    @get:Rule
    val roborazziRule = RoborazziRule(
        options = RoborazziRule.Options(
            // 스크린샷 저장 경로 설정
            outputDirectoryPath = "src/test/snapshots/images",
            // 비교 실패 시 diff 이미지 생성
            recordingMode = RoborazziRule.RecordingMode.AssertMasterImage
        )
    )

    @Test
    @Config(qualifiers = "+night")  // 다크 모드
    fun `로그인화면_다크모드_초기상태`() {
        composeTestRule.setContent {
            MaterialTheme(
                colorScheme = darkColorScheme()
            ) {
                LoginScreen(
                    uiState = LoginUiState(),
                    onEmailChange = {},
                    onPasswordChange = {},
                    onLoginClick = {}
                )
            }
        }

        // 전체 화면 스크린샷 캡처
        composeTestRule.onRoot().captureRoboImage()
    }

    @Test
    fun `로그인화면_라이트모드_에러상태`() {
        composeTestRule.setContent {
            MaterialTheme(
                colorScheme = lightColorScheme()
            ) {
                LoginScreen(
                    uiState = LoginUiState(
                        email = "wrong@email",
                        emailError = "올바른 이메일 형식이 아닙니다",
                        errorMessage = "로그인에 실패했습니다"
                    ),
                    onEmailChange = {},
                    onPasswordChange = {},
                    onLoginClick = {}
                )
            }
        }

        composeTestRule.onRoot().captureRoboImage()
    }

    @Test
    fun `로그인화면_로딩상태`() {
        composeTestRule.setContent {
            MaterialTheme {
                LoginScreen(
                    uiState = LoginUiState(
                        isLoading = true,
                        isLoginEnabled = true
                    ),
                    onEmailChange = {},
                    onPasswordChange = {},
                    onLoginClick = {}
                )
            }
        }

        composeTestRule.onRoot().captureRoboImage()
    }

    @Test
    fun `특정_컴포넌트만_스크린샷_캡처`() {
        composeTestRule.setContent {
            MaterialTheme {
                LoginScreen(
                    uiState = LoginUiState(isLoginEnabled = true),
                    onEmailChange = {},
                    onPasswordChange = {},
                    onLoginClick = {}
                )
            }
        }

        // 버튼 노드만 선택해서 캡처
        composeTestRule
            .onNodeWithTag("login_button")
            .captureRoboImage()
    }
}
```

스크린샷 비교 워크플로우는 다음과 같습니다:

```
# 1. 최초 황금 이미지 생성
./gradlew recordRoborazziDebug

# 2. 코드 변경 후 비교 실행
./gradlew verifyRoborazziDebug

# 3. 변경 내용을 확인하고 의도적 변경이면 황금 이미지 갱신
./gradlew recordRoborazziDebug
```

---

## 고급 패턴: 커스텀 Semantics와 Matcher 작성

실무에서는 기본 제공 Matcher만으로 부족할 때가 있습니다. `Modifier.semantics`로 커스텀 프로퍼티를 추가하고, 이에 맞는 Matcher를 작성하는 패턴을 익혀두면 테스트 코드의 표현력이 크게 높아집니다.

```kotlin
// 커스텀 Semantics 프로퍼티 정의
val LoadingStateKey = SemanticsPropertyKey<Boolean>("isLoading")
var SemanticsPropertyReceiver.isLoading by LoadingStateKey

// Composable에 커스텀 Semantics 적용
@Composable
fun LoadingButton(
    isLoading: Boolean,
    onClick: () -> Unit,
    modifier: Modifier = Modifier,
    content: @Composable () -> Unit
) {
    Button(
        onClick = onClick,
        enabled = !isLoading,
        modifier = modifier.semantics {
            isLoading = isLoading  // 커스텀 프로퍼티 설정
            if (isLoading) {
                contentDescription = "로딩 중입니다. 잠시 기다려주세요."
            }
        }
    ) {
        if (isLoading) {
            CircularProgressIndicator(modifier = Modifier.size(20.dp))
        } else {
            content()
        }
    }
}

// 커스텀 Matcher 함수 작성
fun isInLoadingState(): SemanticsMatcher =
    SemanticsMatcher("is in loading state") { node ->
        node.config.getOrElseNullable(LoadingStateKey) { null } == true
    }

// 테스트에서 커스텀 Matcher 사용
@Test
fun `로딩 상태에서 커스텀 Matcher가 올바르게 동작한다`() {
    composeTestRule.setContent {
        LoadingButton(isLoading = true, onClick = {}) {
            Text("제출")
        }
    }

    composeTestRule
        .onNode(isInLoadingState())
        .assertExists()
        .assertContentDescriptionEquals("로딩 중입니다. 잠시 기다려주세요.")
}
```

---

## 시간 제어와 애니메이션 테스팅

Compose 테스트에서 가장 까다로운 부분 중 하나가 **애니메이션과 타이머**입니다. `composeTestRule`의 Clock API를 활용하면 시간을 직접 제어할 수 있습니다:

```kotlin
@Test
fun `딜레이 후 에러 메시지가 사라진다`() {
    var showError by mutableStateOf(true)

    composeTestRule.setContent {
        LaunchedEffect(Unit) {
            delay(2000L)
            showError = false
        }

        if (showError) {
            Text(
                text = "에러가 발생했습니다",
                modifier = Modifier.testTag("error_text")
            )
        }
    }

    // 초기 상태 확인
    composeTestRule
        .onNodeWithTag("error_text")
        .assertIsDisplayed()

    // 1초 진행 — 에러 메시지 아직 표시
    composeTestRule.mainClock.advanceTimeBy(1000L)
    composeTestRule
        .onNodeWithTag("error_text")
        .assertIsDisplayed()

    // 2초 추가 진행 — 에러 메시지 사라져야 함
    composeTestRule.mainClock.advanceTimeBy(1500L)
    composeTestRule
        .onNodeWithTag("error_text")
        .assertDoesNotExist()
}
```

---

## Semantics Tree 디버깅

테스트가 실패했을 때 현재 Semantics Tree 구조를 출력하면 문제를 빠르게 파악할 수 있습니다:

```kotlin
@Test
fun `Semantics Tree 구조 디버깅`() {
    composeTestRule.setContent {
        LoginScreen(
            uiState = LoginUiState(),
            onEmailChange = {},
            onPasswordChange = {},
            onLoginClick = {}
        )
    }

    // Merged Tree 출력 (접근성이 합치는 형태)
    composeTestRule.onRoot().printToLog("SEMANTICS_MERGED")

    // Unmerged Tree 출력 (실제 모든 노드)
    composeTestRule.onRoot(useUnmergedTree = true).printToLog("SEMANTICS_UNMERGED")
}
```

출력 결과는 Logcat에서 `SEMANTICS_MERGED` 태그로 확인할 수 있으며, 각 노드의 `testTag`, `text`, `contentDescription`, `isEnabled` 등 모든 Semantics 프로퍼티가 표시됩니다.

---

## 주의사항 및 실무 팁

**1. `testTag`는 다음 원칙으로 관리하세요**

중복 태그는 `onNodeWithTag()`가 예외를 던지거나 엉뚱한 노드를 선택하게 만듭니다. 화면별 네임스페이스를 붙이는 컨벤션을 팀 차원에서 정의하는 것이 중요합니다:

```kotlin
// 나쁜 예: 앱 전체에서 중복 가능
Modifier.testTag("button")

// 좋은 예: 화면과 역할을 명확히
Modifier.testTag("login_screen:submit_button")
Modifier.testTag("profile_screen:edit_button")
```

**2. `useUnmergedTree` 플래그를 이해하세요**

`Row`나 `Button` 같은 컴포저블은 기본적으로 자식 텍스트를 하나의 노드로 병합합니다. 내부 특정 텍스트에 접근하려면 `useUnmergedTree = true`를 사용합니다:

```kotlin
// 병합된 트리: "확인" 텍스트가 버튼 노드에 흡수됨
composeTestRule.onNodeWithText("확인")  // 작동 안 할 수 있음

// 미병합 트리: Text 노드를 직접 찾음
composeTestRule.onNodeWithText("확인", useUnmergedTree = true)  // 확실
```

**3. 스크린샷 테스트의 황금 이미지는 반드시 Git에 커밋하세요**

Roborazzi가 생성한 `.png` 황금 이미지는 `.gitignore`에서 제외하고 Git에 추가해야 CI에서 비교가 가능합니다. 이미지가 많으면 Git LFS 사용을 검토하세요.

**4. 에뮬레이터 종속성을 최소화하세요**

Instrumented Test(`androidTest`)는 에뮬레이터가 필요해 CI가 느려집니다. Roborazzi처럼 Robolectric 기반으로 JVM에서 돌리는 로컬 테스트(`test`)를 최대화하고, 실제 디바이스 동작이 중요한 테스트만 Instrumented로 남기세요.

**5. `SemanticsNodeInteraction` 연속 체이닝 활용**

여러 단언을 연쇄 호출하면 보일러플레이트를 줄일 수 있습니다:

```kotlin
composeTestRule
    .onNodeWithTag("login_button")
    .assertIsDisplayed()
    .assertIsEnabled()
    .assertTextEquals("로그인")
    .performClick()
```

---

## 마치며

Jetpack Compose UI 테스팅은 처음에는 낯설지만, Semantics Tree 개념을 이해하고 나면 View 기반 테스팅보다 훨씬 명확하고 표현력이 높다는 것을 느낄 수 있습니다. 특히 접근성 구현과 테스팅이 같은 Semantics 메커니즘을 공유하기 때문에, 테스트를 충실히 작성할수록 접근성도 자연스럽게 향상됩니다.

오늘 소개한 `composeTestRule`의 기본 사용법, 커스텀 Semantics Matcher, 시간 제어, 그리고 Roborazzi를 활용한 Screenshot Test를 프로젝트에 단계적으로 적용해 보세요. CI 파이프라인에 연동하는 순간, 팀 전체의 개발 생산성과 UI 품질이 한 단계 도약하는 것을 경험할 수 있을 것입니다.

---

## 참고 자료
- [Test your Compose layout — Android Developers](https://developer.android.com/jetpack/compose/testing)
- [Semantics in Compose Testing — Android Developers](https://developer.android.com/develop/ui/compose/testing/semantics)
- [Roborazzi — GitHub](https://github.com/takahirom/roborazzi)
- [Screenshot Testing in Compose — ProAndroidDev](https://proandroiddev.com/screenshot-testing-in-compose-f8a7389a7e6)
