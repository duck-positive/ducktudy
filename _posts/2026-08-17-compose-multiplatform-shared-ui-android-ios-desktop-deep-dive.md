---
layout: post
title: "Compose Multiplatform 심화: 단일 Compose UI로 Android·iOS·Desktop을 동시에 지원하는 멀티플랫폼 전략 완전 정복"
date: 2026-08-17
categories: [android, flutter]
tags: [compose-multiplatform, kotlin, android, ios, desktop, kmp, jetpack-compose, multiplatform]
---

Kotlin Multiplatform(KMP)이 비즈니스 로직 공유의 표준으로 자리잡은 지금, 그 위에 얹히는 **Compose Multiplatform(CMP)** 은 UI 레이어까지 공유하는 새로운 패러다임을 열었습니다. 2025년 5월 CMP 1.8.0에서 iOS 지원이 Stable로 전환되면서, 이제 단일 Kotlin/Compose 코드베이스로 Android·iOS·Desktop·Web 앱을 동시에 구축하는 것이 프로덕션에서 현실이 되었습니다. 이 글에서는 CMP의 구조·동작 원리·실전 구현·주의사항을 깊이 파헤칩니다.

---

## 1. Compose Multiplatform이란 무엇인가

**Jetpack Compose**는 Android 전용 선언형 UI 프레임워크입니다. CMP는 JetBrains가 이 API를 기반으로 iOS, Desktop(Windows/macOS/Linux), Web(Wasm·Beta)까지 확장한 프레임워크입니다.

KMP와의 관계를 명확히 정리하면:

| | KMP | CMP |
|---|---|---|
| 공유 대상 | 비즈니스 로직 (Repository, UseCase, ViewModel) | UI 레이어 (`@Composable` 함수) |
| UI 구현 | 플랫폼별 네이티브 (SwiftUI, Jetpack Compose, …) | 단일 Compose 코드 |
| 렌더링 | OS 네이티브 위젯 | Skia(Android·Desktop) / Metal(iOS) |

CMP는 KMP를 기반으로 동작합니다. 즉 **CMP = KMP(로직 공유) + Compose UI(UI 공유)**로 이해하면 됩니다.

---

## 2. 왜 CMP가 필요한가

### 기존 크로스플랫폼 접근의 한계

- **Flutter**: Dart 생태계에 종속되며 기존 Kotlin/Java 자산을 재사용하기 어렵습니다. JVM 기반 라이브러리를 FFI 없이는 쓸 수 없습니다.
- **React Native**: JavaScript 브릿지 오버헤드, JavaScript 런타임 관리 부담이 있습니다.
- **KMP + 네이티브 UI**: 비즈니스 로직은 공유하지만 UI는 SwiftUI와 Jetpack Compose를 따로 작성해야 하므로 UI 중복이 남습니다.

CMP는 **Kotlin 생태계(Ktor, Koin, SQLDelight, Coroutines, Arrow, …) 전체를 활용**하면서 UI까지 공유합니다. Android 팀에게 가장 낮은 러닝 커브를 제공한다는 점도 큰 장점입니다.

### iOS Stable 선언의 의미

CMP 1.8.0(2025년 5월)에서 iOS 타겟이 Stable로 격상되었습니다. Compose UI 코드가 iOS에서 Metal 렌더링 파이프라인 위에서 실행되며, 성능도 많이 개선되었습니다. 실제로 여러 회사가 프로덕션 앱을 App Store에 출시하기 시작했습니다.

---

## 3. 프로젝트 구조와 모듈 레이아웃

CMP 프로젝트의 표준 디렉터리 구조는 다음과 같습니다.

```
MyApp/
├── composeApp/               # 공유 UI 모듈
│   └── src/
│       ├── commonMain/       # 모든 플랫폼 공통 코드 (Composable, ViewModel, …)
│       ├── androidMain/      # Android 전용 코드 (actual 구현, Activity, …)
│       ├── iosMain/          # iOS 전용 코드 (actual 구현, UIViewController, …)
│       └── desktopMain/      # Desktop 전용 코드
├── iosApp/                   # Xcode 프로젝트 (컨테이너)
├── androidApp/               # Android 앱 모듈 (Activity만 포함)
└── shared/                   # 순수 KMP 비즈니스 로직 (선택)
```

`composeApp/build.gradle.kts`에서 멀티플랫폼 타겟을 선언합니다.

```kotlin
kotlin {
    androidTarget {
        compilations.all {
            kotlinOptions.jvmTarget = "17"
        }
    }
    listOf(
        iosX64(), iosArm64(), iosSimulatorArm64()
    ).forEach { iosTarget ->
        iosTarget.binaries.framework {
            baseName = "ComposeApp"
            isStatic = true
        }
    }
    jvm("desktop")

    sourceSets {
        val commonMain by getting {
            dependencies {
                implementation(compose.runtime)
                implementation(compose.foundation)
                implementation(compose.material3)
                implementation(compose.components.resources)
                implementation(libs.lifecycle.viewmodel.compose)
                implementation(libs.navigation.compose)
            }
        }
        val androidMain by getting {
            dependencies {
                implementation(libs.activity.compose)
            }
        }
        val iosMain by getting
        val desktopMain by getting {
            dependencies {
                implementation(compose.desktop.currentOs)
            }
        }
    }
}
```

---

## 4. 구현 예제 1 — 공유 UI와 expect/actual 플랫폼 분기

플랫폼마다 다르게 동작해야 하는 코드는 `expect`/`actual` 키워드로 분리합니다. 공유 UI에서는 `@Composable` 함수를 그대로 사용합니다.

```kotlin
// commonMain/kotlin/Platform.kt
expect fun getPlatformInfo(): String

// androidMain/kotlin/Platform.android.kt
import android.os.Build

actual fun getPlatformInfo(): String =
    "Android ${Build.VERSION.RELEASE} (API ${Build.VERSION.SDK_INT})"

// iosMain/kotlin/Platform.ios.kt
import platform.UIKit.UIDevice

actual fun getPlatformInfo(): String =
    UIDevice.currentDevice.run { "$systemName $systemVersion" }

// desktopMain/kotlin/Platform.desktop.kt
actual fun getPlatformInfo(): String =
    "Desktop — ${System.getProperty("os.name")} ${System.getProperty("os.version")}"
```

이제 `commonMain`에서 공유 UI를 작성합니다.

```kotlin
// commonMain/kotlin/App.kt
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp

@Composable
fun App() {
    MaterialTheme {
        Surface(modifier = Modifier.fillMaxSize()) {
            var counter by remember { mutableStateOf(0) }

            Column(
                modifier = Modifier.fillMaxSize().padding(24.dp),
                horizontalAlignment = Alignment.CenterHorizontally,
                verticalArrangement = Arrangement.Center
            ) {
                Text(
                    text = "Compose Multiplatform 데모",
                    style = MaterialTheme.typography.headlineMedium
                )
                Spacer(Modifier.height(8.dp))
                Text(
                    text = "실행 플랫폼: ${getPlatformInfo()}",
                    style = MaterialTheme.typography.bodySmall,
                    color = MaterialTheme.colorScheme.outline
                )
                Spacer(Modifier.height(32.dp))
                Text(
                    text = "$counter",
                    style = MaterialTheme.typography.displayLarge
                )
                Spacer(Modifier.height(16.dp))
                Row(horizontalArrangement = Arrangement.spacedBy(12.dp)) {
                    OutlinedButton(onClick = { if (counter > 0) counter-- }) {
                        Text("−")
                    }
                    Button(onClick = { counter++ }) {
                        Text("+")
                    }
                }
            }
        }
    }
}
```

Android에서는 `MainActivity`에서, iOS에서는 `MainViewController.kt`에서 `App()`을 각각 진입점으로 호출합니다.

```kotlin
// androidMain/kotlin/MainActivity.kt
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent { App() }
    }
}

// iosMain/kotlin/MainViewController.kt
fun MainViewController(): UIViewController =
    ComposeUIViewController { App() }
```

---

## 5. 구현 예제 2 — 공유 ViewModel과 Koin DI로 상태 관리

CMP에서는 `androidx.lifecycle:lifecycle-viewmodel-compose`의 `ViewModel`을 `commonMain`에서 그대로 사용할 수 있습니다(CMP 1.7+부터 공식 지원). Koin은 KMP를 공식 지원하므로 DI도 공유 가능합니다.

```kotlin
// commonMain/kotlin/di/AppModule.kt
import org.koin.dsl.module
import org.koin.core.module.dsl.viewModelOf

val appModule = module {
    single<TaskRepository> { TaskRepositoryImpl() }
    viewModelOf(::TaskViewModel)
}

// commonMain/kotlin/data/TaskRepository.kt
interface TaskRepository {
    fun getTasksFlow(): Flow<List<Task>>
    suspend fun addTask(title: String)
    suspend fun toggleTask(id: Long)
    suspend fun deleteTask(id: Long)
}

// commonMain/kotlin/viewmodel/TaskViewModel.kt
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.launch

data class Task(val id: Long, val title: String, val isDone: Boolean)

class TaskViewModel(private val repository: TaskRepository) : ViewModel() {

    val tasks: StateFlow<List<Task>> = repository.getTasksFlow()
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), emptyList())

    fun addTask(title: String) {
        viewModelScope.launch { repository.addTask(title) }
    }

    fun toggleTask(id: Long) {
        viewModelScope.launch { repository.toggleTask(id) }
    }

    fun deleteTask(id: Long) {
        viewModelScope.launch { repository.deleteTask(id) }
    }
}

// commonMain/kotlin/ui/TaskScreen.kt
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.Add
import androidx.compose.material.icons.filled.Delete
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import org.koin.compose.viewmodel.koinViewModel

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun TaskScreen(viewModel: TaskViewModel = koinViewModel()) {
    val tasks by viewModel.tasks.collectAsState()
    var showDialog by remember { mutableStateOf(false) }
    var inputText by remember { mutableStateOf("") }

    Scaffold(
        topBar = { TopAppBar(title = { Text("할 일 목록") }) },
        floatingActionButton = {
            FloatingActionButton(onClick = { showDialog = true }) {
                Icon(Icons.Default.Add, contentDescription = "추가")
            }
        }
    ) { padding ->
        LazyColumn(
            modifier = Modifier.fillMaxSize().padding(padding),
            contentPadding = PaddingValues(16.dp),
            verticalArrangement = Arrangement.spacedBy(8.dp)
        ) {
            items(tasks, key = { it.id }) { task ->
                TaskItem(
                    task = task,
                    onToggle = { viewModel.toggleTask(task.id) },
                    onDelete = { viewModel.deleteTask(task.id) }
                )
            }
        }
    }

    if (showDialog) {
        AlertDialog(
            onDismissRequest = { showDialog = false; inputText = "" },
            title = { Text("새 할 일 추가") },
            text = {
                OutlinedTextField(
                    value = inputText,
                    onValueChange = { inputText = it },
                    label = { Text("제목") },
                    singleLine = true
                )
            },
            confirmButton = {
                TextButton(onClick = {
                    if (inputText.isNotBlank()) {
                        viewModel.addTask(inputText.trim())
                        inputText = ""
                        showDialog = false
                    }
                }) { Text("추가") }
            },
            dismissButton = {
                TextButton(onClick = { showDialog = false; inputText = "" }) { Text("취소") }
            }
        )
    }
}

@Composable
private fun TaskItem(task: Task, onToggle: () -> Unit, onDelete: () -> Unit) {
    Card(modifier = Modifier.fillMaxWidth()) {
        Row(
            modifier = Modifier.padding(horizontal = 16.dp, vertical = 8.dp),
            verticalAlignment = androidx.compose.ui.Alignment.CenterVertically
        ) {
            Checkbox(checked = task.isDone, onCheckedChange = { onToggle() })
            Spacer(Modifier.width(8.dp))
            Text(
                text = task.title,
                modifier = Modifier.weight(1f),
                style = if (task.isDone)
                    MaterialTheme.typography.bodyLarge.copy(
                        textDecoration = androidx.compose.ui.text.style.TextDecoration.LineThrough
                    )
                else MaterialTheme.typography.bodyLarge
            )
            IconButton(onClick = onDelete) {
                Icon(Icons.Default.Delete, contentDescription = "삭제")
            }
        }
    }
}
```

Koin 초기화는 플랫폼 진입점에서 각각 호출합니다.

```kotlin
// androidMain — Application 클래스
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        startKoin {
            androidContext(this@MyApp)
            modules(appModule)
        }
    }
}

// iosMain — MainViewController.kt
fun MainViewController(): UIViewController {
    startKoin { modules(appModule) }
    return ComposeUIViewController { App() }
}
```

---

## 6. 리소스 공유: 이미지·폰트·문자열

CMP 1.6 이후 `composeResources` 디렉터리를 통해 이미지, 폰트, 문자열 리소스를 플랫폼 없이 공유할 수 있습니다.

```
commonMain/
└── composeResources/
    ├── drawable/
    │   └── app_icon.png
    ├── font/
    │   └── Pretendard-Regular.ttf
    └── values/
        └── strings.xml
```

코드에서는 생성된 `Res` 객체로 접근합니다.

```kotlin
@Composable
fun LogoImage() {
    Image(
        painter = painterResource(Res.drawable.app_icon),
        contentDescription = "앱 로고"
    )
}

@Composable
fun WelcomeText() {
    Text(text = stringResource(Res.string.welcome_message))
}

@Composable
fun CustomFontText() {
    val fontFamily = FontFamily(Font(Res.font.Pretendard_Regular))
    Text(text = "Pretendard 폰트", fontFamily = fontFamily)
}
```

---

## 7. 주의사항과 실전 팁

### 네비게이션은 서드파티 라이브러리 활용

Jetpack Navigation Compose는 Android 전용입니다. CMP 환경에서는 **Decompose** 또는 **Voyager**를 사용하거나, JetBrains가 공식 지원하는 `navigation-compose` (CMP 포팅 버전)을 사용합니다.

```kotlin
// build.gradle.kts (commonMain)
implementation("org.jetbrains.androidx.navigation:navigation-compose:2.8.0")
```

### iOS에서 접근성은 아직 성숙 중

iOS 접근성(VoiceOver)은 SwiftUI 수준에 도달하지 못했습니다. 장애인 지원이 중요한 앱이라면 iOS UI를 SwiftUI로 유지하고 KMP로 로직만 공유하는 하이브리드 전략을 고려하세요.

### 플랫폼별 UI 분기는 `@Composable` expect/actual로

```kotlin
// commonMain
@Composable
expect fun PlatformDatePicker(
    selectedDate: LocalDate,
    onDateSelected: (LocalDate) -> Unit
)

// androidMain: Material 3 DatePickerDialog 활용
// iosMain: UIDatePicker를 UIKitView로 래핑
```

### iOS 빌드에는 Xcode가 필수

iOS 시뮬레이터/디바이스 빌드는 macOS + Xcode 환경에서만 가능합니다. CI/CD 파이프라인에 macOS 러너(GitHub Actions의 `macos-latest`)를 추가하세요.

### 성능 프로파일링은 플랫폼별로

- **Android**: Android Studio Profiler, Systrace
- **iOS**: Xcode Instruments (Composable 호출은 Swift 프레임 스택으로 표시)
- **Desktop**: JVM Flight Recorder

과도한 리컴포지션은 공유 `@Composable`에서도 동일하게 발생합니다. `derivedStateOf`, `remember`, `key()` 활용 원칙은 Jetpack Compose와 동일합니다.

### Gradle Version Catalog로 의존성 통합 관리

CMP는 KMP + Compose 의존성이 플랫폼별로 많아 `libs.versions.toml`로 관리하는 것이 필수입니다.

```toml
# gradle/libs.versions.toml
[versions]
compose-multiplatform = "1.8.0"
lifecycle = "2.9.0"
koin = "4.0.0"
navigation-compose = "2.8.0"

[libraries]
lifecycle-viewmodel-compose = { module = "org.jetbrains.androidx.lifecycle:lifecycle-viewmodel-compose", version.ref = "lifecycle" }
navigation-compose = { module = "org.jetbrains.androidx.navigation:navigation-compose", version.ref = "navigation-compose" }
koin-compose = { module = "io.insert-koin:koin-compose", version.ref = "koin" }

[plugins]
compose-multiplatform = { id = "org.jetbrains.compose", version.ref = "compose-multiplatform" }
```

---

## 8. KMP vs CMP vs Flutter: 언제 무엇을 선택하나

| 시나리오 | 추천 |
|---|---|
| 팀이 Kotlin/Android 전문 | **CMP** — 최단 러닝 커브, 자산 재활용 |
| 기존 iOS 앱이 SwiftUI로 잘 구현됨 | **KMP** — 로직만 공유, UI는 네이티브 유지 |
| 팀이 Dart에 익숙하고 Kotlin 자산 없음 | **Flutter** |
| 접근성·플랫폼 고유 UX가 최우선 | 네이티브(Jetpack Compose + SwiftUI) |

---

## 결론

Compose Multiplatform은 Android 개발자에게 가장 자연스러운 크로스플랫폼 경로입니다. 기존 Jetpack Compose 지식이 그대로 공유 UI에 적용되며, Kotlin 생태계의 모든 라이브러리를 활용할 수 있습니다. iOS Stable 선언 이후 프로덕션 사례가 빠르게 증가하고 있는 만큼, Android 팀에서 iOS 지원을 고려한다면 CMP는 가장 먼저 검토해야 할 선택지입니다.

## 참고 자료
- [Compose Multiplatform 공식 문서](https://kotlinlang.org/compose-multiplatform/)
- [첫 Compose Multiplatform 앱 만들기 (JetBrains 튜토리얼)](https://kotlinlang.org/docs/multiplatform/compose-multiplatform-create-first-app.html)
- [JetBrains/compose-multiplatform GitHub 저장소](https://github.com/jetbrains/compose-multiplatform)
- [Kotlin Multiplatform | Android Developers 공식 문서](https://developer.android.com/kotlin/multiplatform)
