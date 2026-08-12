---
layout: post
title: "Jetpack Compose UI 테스팅 심화: ComposeTestRule·Semantics·스크린샷 테스트 완전 정복"
date: 2026-08-12
categories: [android]
tags: [jetpack-compose, testing, composetestrule, semantics, roborazzi, paparazzi, ui-test, kotlin]
---

Android 개발에서 UI 테스트는 항상 골칫거리였습니다. Espresso로 View 계층을 탐색하고, `idleResource`를 등록하고, 타이밍 문제로 간헐적으로 실패하는 Flaky Test와 싸우는 일이 일상이었습니다. Jetpack Compose는 이 패러다임을 완전히 바꿨습니다. **Semantics 트리** 기반의 테스트 API는 UI 계층을 추상화하고, `ComposeTestRule`은 동기화 문제를 프레임워크 수준에서 해결합니다. 이 글에서는 Compose UI 테스트의 핵심 개념부터 스크린샷 테스트까지, 프로덕션에서 바로 쓸 수 있는 수준의 심화 내용을 다룹니다.

---

## 1. Compose 테스트가 필요한 이유: Espresso의 한계

기존 View 기반 Espresso 테스트의 대표적인 문제는 **구현 세부사항에 의존**한다는 점입니다. 예를 들어 `R.id.button_submit`이라는 리소스 ID로 버튼을 찾는 코드는 레이아웃 XML 구조가 바뀌거나 ID가 변경되는 순간 테스트가 깨집니다. Compose 테스트는 `testTag`나 시맨틱 의미(텍스트, 역할, 상태)로 노드를 탐색하기 때문에 구현 변경에 훨씬 강합니다.

두 번째 문제는 **비동기 동기화**입니다. Espresso는 `IdlingResource`를 직접 등록해야 Coroutine, Animation 등 비동기 작업을 기다릴 수 있습니다. `ComposeTestRule`은 Compose의 Recomposition, Animation, LaunchedEffect 완료를 자동으로 기다립니다. 테스트 코드에 `Thread.sleep()`이 사라집니다.

---

## 2. 테스트 환경 설정

### 의존성 추가

```kotlin
// build.gradle.kts (app 모듈)
dependencies {
    androidTestImplementation("androidx.compose.ui:ui-test-junit4")
    debugImplementation("androidx.compose.ui:ui-test-manifest")

    // Hilt 테스트 지원
    androidTestImplementation("com.google.dagger:hilt-android-testing:2.51.1")
    kaptAndroidTest("com.google.dagger:hilt-android-compiler:2.51.1")

    // MockK
    androidTestImplementation("io.mockk:mockk-android:1.13.12")
}
```

`ui-test-junit4`는 `createComposeRule()` / `createAndroidComposeRule<>()` 팩토리를 제공하고, `ui-test-manifest`는 테스트 Activity(`ComponentActivity`)의 Manifest 항목을 자동으로 추가합니다. `debugImplementation`이어야 하는 이유는 릴리스 빌드에 테스트용 Activity가 포함되지 않도록 하기 위함입니다.

### ComposeTestRule 종류

| 규칙 | 설명 | 언제 사용하나 |
|---|---|---|
| `createComposeRule()` | 빈 `ComponentActivity` 위에 Composable 설정 | 독립 컴포넌트 단위 테스트 |
| `createAndroidComposeRule<MyActivity>()` | 실제 Activity 위에서 실행 | 딥링크, Navigation, 시스템 UI 포함 테스트 |
| `createComposeRule(effectContext)` | 커스텀 CoroutineContext 사용 | 코루틴 테스트 제어 |

---

## 3. Semantics 트리: Compose 테스트의 핵심

Compose UI는 Widget → Element → RenderObject 트리 외에 **Semantics 트리**를 병렬로 유지합니다. 이 트리는 접근성 서비스(TalkBack)와 테스트 프레임워크가 UI의 의미를 파악하는 데 사용됩니다.

### Semantics 프로퍼티 확인하기

Android Studio의 **Layout Inspector** → "Show Semantics" 옵션, 또는 테스트에서 `printToLog()`로 확인할 수 있습니다:

```kotlin
@Test
fun debugSemanticsTree() {
    composeTestRule.setContent {
        LoginScreen()
    }
    // 전체 Semantics 트리를 로그에 출력
    composeTestRule.onRoot().printToLog("SemanticTree")
}
```

출력 예시:
```
Node #1 at (0.0, 0.0, 1080.0, 2400.0)px
 |-Node #5 at (96.0, 240.0, 984.0, 360.0)px
 | Text = '[이메일]'
 | Role = 'EditText'
 | EditableText = ''
 | Actions = [SetText, SetSelection, ...]
 |-Node #9 at (96.0, 400.0, 984.0, 520.0)px
   Text = '[로그인]'
   Role = 'Button'
   Actions = [OnClick]
```

### 커스텀 Semantics 프로퍼티 추가

기본 Composable이 제공하지 않는 의미론적 정보를 직접 추가할 수 있습니다. 예를 들어 로딩 상태를 외부에서 감지하고 싶다면:

```kotlin
// 커스텀 시맨틱 키 정의
val LoadingStateKey = SemanticsPropertyKey<Boolean>("IsLoading")
var SemanticsPropertyReceiver.isLoading by LoadingStateKey

// 컴포저블에서 사용
@Composable
fun LoadingButton(
    isLoading: Boolean,
    onClick: () -> Unit,
    modifier: Modifier = Modifier
) {
    Box(
        modifier = modifier
            .semantics { this.isLoading = isLoading }
            .clickable(onClick = onClick),
        contentAlignment = Alignment.Center
    ) {
        if (isLoading) {
            CircularProgressIndicator(Modifier.size(24.dp))
        } else {
            Text("제출")
        }
    }
}

// 테스트에서 커스텀 프로퍼티로 노드 탐색
@Test
fun loadingButton_showsProgressIndicator_whenLoading() {
    var isLoading by mutableStateOf(false)

    composeTestRule.setContent {
        LoadingButton(isLoading = isLoading, onClick = {})
    }

    // 초기 상태: 로딩 아님
    composeTestRule
        .onNode(SemanticsMatcher.expectValue(LoadingStateKey, false))
        .assertIsDisplayed()

    isLoading = true
    composeTestRule.waitForIdle()

    // 로딩 중 상태 확인
    composeTestRule
        .onNode(SemanticsMatcher.expectValue(LoadingStateKey, true))
        .assertIsDisplayed()
    composeTestRule.onNodeWithText("제출").assertDoesNotExist()
}
```

---

## 4. ComposeTestRule API 완전 분석

### 노드 탐색 (Finder)

```kotlin
// 텍스트로 찾기 (대소문자 구분 없음 옵션)
composeTestRule.onNodeWithText("로그인", ignoreCase = true)

// testTag로 찾기
composeTestRule.onNodeWithTag("login_button")

// 콘텐츠 설명으로 찾기 (접근성 레이블)
composeTestRule.onNodeWithContentDescription("닫기 버튼")

// 복합 조건 매처
composeTestRule.onNode(
    hasText("로그인") and hasClickAction() and isEnabled()
)

// 형제/자식 노드 탐색
composeTestRule.onNodeWithTag("user_list")
    .onChildren()
    .filter(hasText("김철수"))
    .assertCountEquals(1)
```

### 액션 수행 (Action)

```kotlin
composeTestRule.onNodeWithText("로그인").performClick()

// 텍스트 입력
composeTestRule.onNodeWithTag("email_field").performTextInput("test@example.com")

// IME 액션 (키보드 확인 버튼)
composeTestRule.onNodeWithTag("password_field").performImeAction()

// 스크롤
composeTestRule.onNodeWithTag("content_list").performScrollToIndex(10)

// 드래그 제스처
composeTestRule.onNodeWithTag("slider").performTouchInput {
    swipeRight(startX = 0f, endX = 500f)
}
```

### 어설션 (Assertion)

```kotlin
composeTestRule.onNodeWithText("홈").assertIsDisplayed()
composeTestRule.onNodeWithTag("submit_button").assertIsEnabled()
composeTestRule.onNodeWithTag("error_message").assertIsNotDisplayed()
composeTestRule.onNodeWithText("오류 발생").assertExists()

// 여러 어설션 체이닝
composeTestRule.onNodeWithTag("profile_card")
    .assertIsDisplayed()
    .assertHasClickAction()
    .assert(hasText("홍길동", substring = true))
```

---

## 5. 실전 예제 1: ViewModel 연동 UI 테스트

실제 앱에서는 Composable이 ViewModel과 연동됩니다. Hilt와 MockK를 활용한 완전한 테스트 예시입니다.

```kotlin
// ViewModel
@HiltViewModel
class LoginViewModel @Inject constructor(
    private val authRepository: AuthRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow(LoginUiState())
    val uiState: StateFlow<LoginUiState> = _uiState.asStateFlow()

    fun onEmailChange(email: String) {
        _uiState.update { it.copy(email = email) }
    }

    fun onPasswordChange(password: String) {
        _uiState.update { it.copy(password = password) }
    }

    fun login() {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true, error = null) }
            authRepository.login(_uiState.value.email, _uiState.value.password)
                .onSuccess { _uiState.update { it.copy(isLoading = false, isLoggedIn = true) } }
                .onFailure { e -> _uiState.update { it.copy(isLoading = false, error = e.message) } }
        }
    }
}

data class LoginUiState(
    val email: String = "",
    val password: String = "",
    val isLoading: Boolean = false,
    val isLoggedIn: Boolean = false,
    val error: String? = null
)

// 테스트 클래스
@HiltAndroidTest
class LoginScreenTest {

    @get:Rule(order = 0)
    val hiltRule = HiltAndroidRule(this)

    @get:Rule(order = 1)
    val composeTestRule = createAndroidComposeRule<MainActivity>()

    @BindValue
    val authRepository: AuthRepository = mockk()

    @Before
    fun setup() {
        hiltRule.inject()
    }

    @Test
    fun loginScreen_displaysError_whenLoginFails() {
        // given: 로그인 실패 시나리오 설정
        coEvery {
            authRepository.login(any(), any())
        } returns Result.failure(Exception("이메일 또는 비밀번호가 올바르지 않습니다."))

        composeTestRule.setContent {
            val viewModel: LoginViewModel = hiltViewModel()
            LoginScreen(viewModel = viewModel)
        }

        // when: 이메일, 비밀번호 입력 후 로그인 시도
        composeTestRule.onNodeWithTag("email_field")
            .performTextInput("wrong@example.com")
        composeTestRule.onNodeWithTag("password_field")
            .performTextInput("wrongpassword")
        composeTestRule.onNodeWithTag("login_button")
            .performClick()

        // then: 에러 메시지 표시 확인
        composeTestRule.waitUntil(timeoutMillis = 3_000) {
            composeTestRule
                .onAllNodesWithText("이메일 또는 비밀번호가 올바르지 않습니다.")
                .fetchSemanticsNodes().isNotEmpty()
        }
        composeTestRule.onNodeWithTag("login_button").assertIsEnabled()
    }

    @Test
    fun loginScreen_disablesButton_whileLoading() {
        // given: 로딩이 오래 걸리는 시나리오
        coEvery {
            authRepository.login(any(), any())
        } coAnswers {
            delay(5_000) // 5초 지연
            Result.success(Unit)
        }

        composeTestRule.setContent {
            val viewModel: LoginViewModel = hiltViewModel()
            LoginScreen(viewModel = viewModel)
        }

        composeTestRule.onNodeWithTag("email_field").performTextInput("test@example.com")
        composeTestRule.onNodeWithTag("password_field").performTextInput("password123")
        composeTestRule.onNodeWithTag("login_button").performClick()

        // 로딩 중에는 버튼 비활성화
        composeTestRule.onNodeWithTag("login_button").assertIsNotEnabled()
        composeTestRule.onNodeWithTag("loading_indicator").assertIsDisplayed()
    }
}
```

---

## 6. 실전 예제 2: LazyColumn 리스트 테스트

페이징과 스크롤이 있는 리스트는 테스트에서 특별한 주의가 필요합니다.

```kotlin
@Composable
fun UserListScreen(
    users: List<User>,
    onUserClick: (User) -> Unit
) {
    LazyColumn(modifier = Modifier.testTag("user_list")) {
        items(users, key = { it.id }) { user ->
            UserCard(
                user = user,
                modifier = Modifier
                    .testTag("user_card_${user.id}")
                    .clickable { onUserClick(user) }
            )
        }
    }
}

// 테스트
class UserListScreenTest {

    @get:Rule
    val composeTestRule = createComposeRule()

    private val testUsers = (1..50).map { id ->
        User(id = id, name = "사용자 $id", email = "user$id@example.com")
    }

    @Test
    fun userList_scrollsToItem_andClickable() {
        var clickedUser: User? = null

        composeTestRule.setContent {
            UserListScreen(
                users = testUsers,
                onUserClick = { clickedUser = it }
            )
        }

        // 화면에 보이지 않는 50번째 항목으로 스크롤
        composeTestRule
            .onNodeWithTag("user_list")
            .performScrollToIndex(49)

        // 스크롤 후 항목이 보이는지 확인
        composeTestRule
            .onNodeWithTag("user_card_50")
            .assertIsDisplayed()
            .performClick()

        // 클릭 이벤트 확인
        assertThat(clickedUser?.id).isEqualTo(50)
    }

    @Test
    fun userList_displaysCorrectCount() {
        composeTestRule.setContent {
            UserListScreen(users = testUsers.take(5), onUserClick = {})
        }

        // 현재 화면에 보이는 항목들만 검사
        composeTestRule
            .onAllNodesWithContentDescription("사용자 카드")
            .assertCountEquals(5)
    }
}
```

---

## 7. 스크린샷 테스트: Roborazzi 심화

스크린샷 테스트(골든 테스트)는 UI의 시각적 회귀를 자동으로 감지합니다. **Roborazzi**는 Robolectric 위에서 JVM 테스트로 실행되므로 에뮬레이터 없이도 스크린샷을 생성할 수 있습니다.

### Roborazzi 설정

```kotlin
// build.gradle.kts
plugins {
    id("io.github.takahirom.roborazzi") version "1.26.0"
}

dependencies {
    testImplementation("io.github.takahirom.roborazzi:roborazzi:1.26.0")
    testImplementation("io.github.takahirom.roborazzi:roborazzi-compose:1.26.0")
    testImplementation("org.robolectric:robolectric:4.13")
    testImplementation("androidx.compose.ui:ui-test-junit4")
}

roborazzi {
    outputDir.set(project.file("screenshots"))
}
```

### 스크린샷 캡처 테스트

```kotlin
@RunWith(RobolectricTestRunner::class)
@Config(sdk = [34], qualifiers = "w360dp-h800dp-xxhdpi")
class UserCardScreenshotTest {

    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun userCard_defaultState_matchesGolden() {
        composeTestRule.setContent {
            MaterialTheme {
                UserCard(
                    user = User(id = 1, name = "홍길동", email = "hong@example.com"),
                    modifier = Modifier.fillMaxWidth()
                )
            }
        }

        composeTestRule
            .onNodeWithTag("user_card_1")
            .captureRoboImage("screenshots/user_card_default.png")
    }

    @Test
    fun userCard_darkTheme_matchesGolden() {
        composeTestRule.setContent {
            MaterialTheme(colorScheme = darkColorScheme()) {
                UserCard(
                    user = User(id = 1, name = "홍길동", email = "hong@example.com"),
                    modifier = Modifier.fillMaxWidth()
                )
            }
        }

        composeTestRule
            .onRoot()
            .captureRoboImage(
                filePath = "screenshots/user_card_dark.png",
                roborazziOptions = RoborazziOptions(
                    compareOptions = RoborazziOptions.CompareOptions(
                        changeThreshold = 0.01f // 1% 이상 변경 시 실패
                    )
                )
            )
    }
}
```

### 골든 이미지 관리 워크플로우

```bash
# 최초 골든 이미지 생성
./gradlew recordRoborazziDebug

# CI에서 비교 실행
./gradlew verifyRoborazziDebug

# 변경 후 골든 업데이트
./gradlew recordRoborazziDebug
git add screenshots/
git commit -m "chore: update golden screenshots"
```

Roborazzi는 테스트 실패 시 **비교 이미지**를 자동으로 생성합니다:
- `{name}_actual.png`: 현재 렌더링 결과
- `{name}_compare.png`: 골든 vs 실제 diff 시각화

---

## 8. 동기화와 waitUntil

`ComposeTestRule`은 기본적으로 `waitForIdle()`을 자동 호출합니다. 하지만 **무한 애니메이션**이나 **외부 타이머**가 있는 경우 idle 상태에 도달하지 못해 테스트가 무한 대기하는 문제가 발생합니다.

```kotlin
// 문제 상황: 깜빡이는 커서가 있는 텍스트 필드
// → 애니메이션이 계속 실행되므로 idle 상태가 되지 않음

// 해결책 1: mainClock 수동 제어
composeTestRule.mainClock.autoAdvance = false
composeTestRule.mainClock.advanceTimeBy(1_000) // 1초 진행

// 해결책 2: 조건 기반 대기
composeTestRule.waitUntil(timeoutMillis = 5_000) {
    composeTestRule
        .onAllNodesWithText("로딩 완료")
        .fetchSemanticsNodes()
        .isNotEmpty()
}

// 해결책 3: 애니메이션 비활성화 (테스트 환경에서 권장)
// AndroidManifest.xml (debug)
// <uses-permission android:name="android.permission.WRITE_SETTINGS" />
// 또는 WindowAnimationScale을 0으로 설정
```

---

## 9. 주의사항과 팁

### testTag는 의미있게 지정하라

```kotlin
// 나쁜 예: 컴포넌트 구조에 의존
Modifier.testTag("column_0_row_2_button")

// 좋은 예: 비즈니스 의미 반영
Modifier.testTag("checkout_confirm_button")
```

### 프로덕션 코드에 testTag 포함 문제

`testTag`는 릴리스 빌드에도 포함됩니다. 빌드 오버헤드는 미미하지만, 신경 쓰인다면 `BuildConfig`로 분기하거나 `Modifier.testTag(if (BuildConfig.DEBUG) tag else "")` 패턴을 사용합니다.

### Semantics Merging 주의

Compose는 기본적으로 부모 Semantics 노드에 자식 노드를 병합합니다. `Button` 내부의 `Text`는 별도 노드로 접근되지 않을 수 있습니다. `useUnmergedTree = true` 옵션으로 병합 전 트리를 탐색할 수 있습니다:

```kotlin
composeTestRule.onNodeWithText("제출", useUnmergedTree = true)
```

### 테스트 격리

각 `@Test`마다 새로운 컴포지션이 생성됩니다. 상태를 공유하지 말고, `@Before`에서 상태를 초기화하세요. Hilt 테스트라면 `hiltRule.inject()`를 `@Before`에서 반드시 호출해야 합니다.

### CI 에서 스크린샷 테스트 안정화

Roborazzi 스크린샷은 폰트 렌더링, 안티앨리어싱 등 환경 차이로 로컬과 CI에서 미세하게 다를 수 있습니다. `changeThreshold`를 적절히 설정하고(0.01~0.05), CI 환경에서 골든 이미지를 생성하는 것을 권장합니다.

---

## 10. 정리: 테스팅 전략 선택 가이드

| 테스트 유형 | 도구 | 적용 범위 | 실행 속도 |
|---|---|---|---|
| 단위 테스트 | JUnit + MockK | ViewModel, UseCase, Repository | 빠름 |
| Composable 단위 테스트 | ComposeTestRule | 단일 컴포넌트 | 중간 |
| 스크린샷 테스트 | Roborazzi | 시각적 회귀 감지 | 중간 |
| 통합 테스트 | createAndroidComposeRule | 화면 전체, Navigation | 느림 |
| E2E 테스트 | Espresso / UIAutomator | 전체 앱 플로우 | 매우 느림 |

Compose 테스트의 핵심 철학은 **구현이 아닌 의미**를 테스트하는 것입니다. Semantics 트리를 잘 설계하면 접근성과 테스트 가능성을 동시에 얻을 수 있습니다. `ComposeTestRule`의 자동 동기화와 Roborazzi의 시각적 회귀 탐지를 조합하면, 리팩토링 후에도 자신 있게 변경을 배포할 수 있는 테스트 스위트를 구축할 수 있습니다.

---

## 참고 자료
- [Testing your Compose layout — Android Developers](https://developer.android.com/develop/ui/compose/testing)
- [Semantics in Compose — Android Developers](https://developer.android.com/develop/ui/compose/testing/semantics)
- [Roborazzi — GitHub (takahirom)](https://github.com/takahirom/roborazzi/)
