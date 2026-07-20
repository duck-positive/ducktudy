---
layout: post
title: "Android Edge-to-Edge 심화: WindowInsets API·WindowCompat·IME 애니메이션으로 완전한 몰입형 UI 구현하기"
date: 2026-07-20
categories: [android]
tags: [android, edge-to-edge, windowinsets, ime, compose, kotlin, android15]
---

## 개요: Edge-to-Edge란 무엇인가

Edge-to-Edge는 앱의 콘텐츠가 화면 전체를 덮도록 시스템 바(상태 바, 내비게이션 바) 뒤까지 그려지게 하는 UI 패턴입니다. Android 15(API 35)부터는 `targetSdk 35`로 빌드하면 이 모드가 **강제 적용**됩니다. 기존처럼 시스템 바 영역이 자동으로 보호되지 않기 때문에, WindowInsets API를 올바르게 이해하고 처리하지 않으면 중요한 UI 요소가 시스템 바에 가려지는 문제가 발생합니다.

Android 15 이전에도 `WindowCompat.enableEdgeToEdge(window)`를 호출하면 opt-in 방식으로 Edge-to-Edge를 활성화할 수 있었지만, 이제는 선택이 아닌 필수가 된 것입니다.

---

## 왜 필요한가

### Android 15의 강제 적용

Android 15(API 35) 이상에서 `targetSdk 35`로 빌드된 앱은 자동으로 Edge-to-Edge 모드가 활성화됩니다. 기존 앱이 이를 대응하지 않으면 다음과 같은 문제가 발생합니다.

- 하단 내비게이션 바가 버튼이나 리스트 아이템을 가림
- 상태 바 뒤에 툴바 텍스트가 숨겨짐
- IME(소프트 키보드)가 올라올 때 입력 필드가 키보드 뒤에 가려짐

### 사용자 경험 개선

올바르게 구현하면 다음과 같은 이점을 얻을 수 있습니다.

- 콘텐츠가 화면 끝까지 자연스럽게 펼쳐지는 몰입감 제공
- IME 등장·사라짐에 애니메이션을 동기화하여 자연스러운 전환 효과
- 다이나믹 아일랜드, 노치, 워터폴 디스플레이 등 다양한 기기 형태 대응

---

## WindowInsets의 종류

Android에서 `WindowInsets`는 시스템 UI가 차지하는 영역을 앱에 알려주는 메커니즘입니다. 주요 종류는 다음과 같습니다.

| Inset 유형 | 설명 |
|---|---|
| `systemBars` | 상태 바 + 내비게이션 바 합산 |
| `statusBars` | 상태 바 영역 |
| `navigationBars` | 하단 또는 측면 내비게이션 바 |
| `ime` | 소프트 키보드 영역 |
| `displayCutout` | 노치 / 펀치홀 카메라 영역 |
| `systemGestures` | 시스템 제스처 영역 |
| `safeDrawing` | 안전하게 그릴 수 있는 전체 영역 |
| `safeGestures` | 제스처 충돌 없는 안전 영역 |
| `safeContent` | safeDrawing + safeGestures 합산 |

`safeDrawing`은 모든 시스템 UI 뒤에 그려지지 않도록 보호해 주는 편의 타입으로, `systemBars`와 `displayCutout`을 별도로 합산하는 대신 사용할 수 있습니다.

---

## 기본 구현: View 기반

### Edge-to-Edge 활성화 및 Insets 처리

```kotlin
class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // Android 15 미만에서도 Edge-to-Edge 적용
        WindowCompat.enableEdgeToEdge(window)

        setContentView(R.layout.activity_main)

        val rootView = findViewById<View>(R.id.root_layout)
        ViewCompat.setOnApplyWindowInsetsListener(rootView) { view, windowInsets ->
            val insets = windowInsets.getInsets(
                WindowInsetsCompat.Type.systemBars() or
                WindowInsetsCompat.Type.displayCutout()
            )
            view.updatePadding(
                left   = insets.left,
                top    = insets.top,
                right  = insets.right,
                bottom = insets.bottom
            )
            WindowInsetsCompat.CONSUMED
        }
    }
}
```

`WindowCompat.enableEdgeToEdge(window)`는 시스템 바를 투명하게 만들고 앱 콘텐츠가 뒤로 확장되도록 설정합니다. `ViewCompat.setOnApplyWindowInsetsListener`는 시스템이 insets를 전달할 때 콜백을 받는 진입점이며, 루트 뷰에 시스템 바 + DisplayCutout 크기만큼 패딩을 추가해 콘텐츠가 가려지지 않도록 합니다. 마지막으로 `WindowInsetsCompat.CONSUMED`를 반환하면 자식 뷰에게 insets가 중복으로 전파되는 것을 막을 수 있습니다.

### RecyclerView에 Insets 적용

모든 뷰에 동일하게 padding을 적용하면 스크롤 영역에서 문제가 생깁니다. `RecyclerView`처럼 스크롤 가능한 뷰는 `padding + clipToPadding=false` 패턴을 사용합니다.

```kotlin
ViewCompat.setOnApplyWindowInsetsListener(recyclerView) { view, windowInsets ->
    val insets = windowInsets.getInsets(
        WindowInsetsCompat.Type.systemBars() or
        WindowInsetsCompat.Type.displayCutout()
    )
    // 좌·우·하단만 패딩 추가 (상단은 앱바가 이미 처리)
    view.updatePadding(
        left   = insets.left,
        right  = insets.right,
        bottom = insets.bottom
    )
    // clipToPadding = false: 스크롤 시 패딩 영역까지 콘텐츠가 보이고,
    // 스크롤 끝에서는 패딩만큼 여백 확보
    windowInsets
}
```

XML에서는 `android:clipToPadding="false"`로 설정합니다.

---

## Jetpack Compose에서 WindowInsets

Compose에서는 훨씬 선언적이고 간결한 방식으로 insets를 처리할 수 있습니다.

### Scaffold를 활용한 자동 처리

Material 3의 `Scaffold`는 기본적으로 `WindowInsets`를 내부적으로 처리합니다.

```kotlin
@Composable
fun MainScreen(messages: List<Message>) {
    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("Edge-to-Edge 예제") },
                // 기본값으로 상태 바 inset 자동 처리
                windowInsets = TopAppBarDefaults.windowInsets
            )
        },
        bottomBar = {
            NavigationBar(
                // 내비게이션 바 inset 자동 처리
                windowInsets = NavigationBarDefaults.windowInsets
            ) {
                NavigationBarItem(
                    selected = true,
                    onClick = {},
                    icon = { Icon(Icons.Default.Home, null) },
                    label = { Text("홈") }
                )
            }
        }
    ) { innerPadding ->
        // innerPadding은 상·하단 바 영역을 제외한 콘텐츠 영역
        LazyColumn(
            contentPadding = innerPadding,
            modifier = Modifier.fillMaxSize()
        ) {
            items(messages) { msg ->
                MessageItem(msg)
            }
        }
    }
}
```

`Scaffold`의 `innerPadding`을 `LazyColumn`의 `contentPadding`으로 전달하면, 스크롤 목록의 끝에 자동으로 여백이 생겨 마지막 아이템이 내비게이션 바에 가려지지 않습니다.

### 개별 Modifier로 Insets 처리

`Scaffold` 없이 직접 insets를 처리할 때는 `windowInsetsPadding` Modifier 또는 개별 편의 Modifier를 사용합니다.

```kotlin
@Composable
fun CustomScreen() {
    Column(
        modifier = Modifier
            .fillMaxSize()
            .statusBarsPadding()         // 상태 바 패딩
            .navigationBarsPadding()     // 내비게이션 바 패딩
    ) {
        Text("안전 영역 안에 표시되는 콘텐츠")
    }
}

// 또는 safeDrawing으로 한 번에 처리
@Composable
fun SafeDrawingScreen() {
    Box(
        modifier = Modifier
            .fillMaxSize()
            .windowInsetsPadding(WindowInsets.safeDrawing)
    ) {
        Text("노치/바 모두 고려된 안전 영역")
    }
}
```

---

## IME(소프트 키보드) 애니메이션 동기화

Android 11(API 30)부터 `WindowInsetsAnimationCompat`을 사용해 IME 애니메이션과 앱 레이아웃 변경을 동기화할 수 있습니다. 이 기능이 없으면 키보드가 올라오면서 뷰가 갑자기 튀어오르는 어색한 UX가 발생합니다.

### View 기반 IME 애니메이션 콜백

```kotlin
class ChatFragment : Fragment(R.layout.fragment_chat) {

    private lateinit var binding: FragmentChatBinding

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        // adjustResize/adjustPan 대신 adjustNothing 사용
        requireActivity().window.setSoftInputMode(
            WindowManager.LayoutParams.SOFT_INPUT_ADJUST_NOTHING
        )

        var startBottom = 0
        var endBottom = 0

        ViewCompat.setWindowInsetsAnimationCallback(
            binding.root,
            object : WindowInsetsAnimationCompat.Callback(DISPATCH_MODE_STOP) {

                override fun onPrepare(animation: WindowInsetsAnimationCompat) {
                    startBottom = ViewCompat.getRootWindowInsets(binding.root)
                        ?.getInsets(WindowInsetsCompat.Type.ime())?.bottom ?: 0
                }

                override fun onStart(
                    animation: WindowInsetsAnimationCompat,
                    bounds: WindowInsetsAnimationCompat.BoundsCompat
                ): WindowInsetsAnimationCompat.BoundsCompat {
                    endBottom = ViewCompat.getRootWindowInsets(binding.root)
                        ?.getInsets(WindowInsetsCompat.Type.ime())?.bottom ?: 0
                    return bounds
                }

                override fun onProgress(
                    insets: WindowInsetsCompat,
                    runningAnimations: MutableList<WindowInsetsAnimationCompat>
                ): WindowInsetsCompat {
                    val imeAnim = runningAnimations.firstOrNull {
                        it.typeMask and WindowInsetsCompat.Type.ime() != 0
                    } ?: return insets

                    // 보간된 프레임 값으로 입력창을 자연스럽게 이동
                    val fraction = imeAnim.interpolatedFraction
                    val translationY = lerp(
                        startBottom.toFloat(),
                        endBottom.toFloat(),
                        fraction
                    )
                    binding.inputLayout.translationY = -translationY
                    return insets
                }
            }
        )

        // IME 등장 시 메시지 목록 최하단으로 스크롤
        ViewCompat.setOnApplyWindowInsetsListener(binding.root) { _, windowInsets ->
            val imeHeight = windowInsets.getInsets(WindowInsetsCompat.Type.ime()).bottom
            if (imeHeight > 0) {
                binding.messageList.scrollToPosition(adapter.itemCount - 1)
            }
            windowInsets
        }
    }

    private fun lerp(start: Float, end: Float, fraction: Float) =
        start + (end - start) * fraction
}
```

`onPrepare`에서 시작 상태를 저장하고, `onStart`에서 종료 상태를 저장한 뒤, `onProgress`에서 `interpolatedFraction`을 이용해 매 프레임마다 뷰의 `translationY`를 갱신합니다. 이렇게 하면 시스템 IME 애니메이션 곡선과 앱 레이아웃 변경이 완벽하게 동기화됩니다.

### Compose에서 IME 처리

Compose에서는 `imePadding()`과 `imeNestedScroll()` Modifier로 훨씬 간단하게 처리할 수 있습니다.

```kotlin
@Composable
fun MessageScreen(
    messages: List<Message>,
    onSend: (String) -> Unit
) {
    var inputText by remember { mutableStateOf("") }
    val listState = rememberLazyListState()

    Column(
        modifier = Modifier
            .fillMaxSize()
            .navigationBarsPadding()
            .imePadding()  // IME가 올라오면 자동으로 하단 패딩 추가
    ) {
        LazyColumn(
            state = listState,
            modifier = Modifier
                .weight(1f)
                .imeNestedScroll(),  // IME와 스크롤 제스처 연동
            reverseLayout = true,
            contentPadding = PaddingValues(horizontal = 16.dp, vertical = 8.dp)
        ) {
            items(messages, key = { it.id }) { message ->
                MessageBubble(message = message)
                Spacer(Modifier.height(4.dp))
            }
        }

        Row(
            modifier = Modifier
                .fillMaxWidth()
                .padding(horizontal = 16.dp, vertical = 8.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            OutlinedTextField(
                value = inputText,
                onValueChange = { inputText = it },
                modifier = Modifier.weight(1f),
                placeholder = { Text("메시지 입력") },
                maxLines = 4
            )
            Spacer(Modifier.width(8.dp))
            FilledIconButton(onClick = {
                if (inputText.isNotBlank()) {
                    onSend(inputText)
                    inputText = ""
                }
            }) {
                Icon(Icons.AutoMirrored.Filled.Send, contentDescription = "전송")
            }
        }
    }
}
```

`imeNestedScroll()`은 사용자가 스크롤을 내릴 때 IME를 자동으로 닫는 효과도 제공합니다. 채팅 앱의 메시지 목록에서 자주 사용하는 패턴입니다.

---

## 시스템 바 아이콘 색상 제어

배경색에 따라 상태 바·내비게이션 바의 아이콘 색상을 조정해야 할 때는 `WindowInsetsController`를 사용합니다.

```kotlin
// View 기반
val controller = WindowCompat.getInsetsController(window, window.decorView)

// 밝은 배경: 어두운 아이콘으로 전환
controller.isAppearanceLightStatusBars = true
controller.isAppearanceLightNavigationBars = true

// Compose: SideEffect로 Activity Window에 접근
@Composable
fun SystemBarAppearance(useDarkIcons: Boolean) {
    val view = LocalView.current
    if (!view.isInEditMode) {
        SideEffect {
            val window = (view.context as Activity).window
            WindowCompat.getInsetsController(window, view).apply {
                isAppearanceLightStatusBars = useDarkIcons
                isAppearanceLightNavigationBars = useDarkIcons
            }
        }
    }
}
```

---

## BottomSheet / Dialog 별도 처리

`BottomSheetDialogFragment`와 `Dialog`는 독립적인 `Window`를 생성합니다. 따라서 액티비티의 Edge-to-Edge 설정과 별개로 각 Window에서도 설정이 필요합니다.

```kotlin
class MessageInputBottomSheet : BottomSheetDialogFragment() {

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        // BottomSheet의 Window에도 별도로 적용
        dialog?.window?.let { dialogWindow ->
            WindowCompat.setDecorFitsSystemWindows(dialogWindow, false)
        }

        ViewCompat.setOnApplyWindowInsetsListener(view) { v, insets ->
            val imeBottom = insets.getInsets(WindowInsetsCompat.Type.ime()).bottom
            val navBottom = insets.getInsets(WindowInsetsCompat.Type.navigationBars()).bottom
            // IME가 없으면 내비게이션 바만큼, IME가 있으면 IME 높이만큼 패딩
            v.updatePadding(bottom = maxOf(imeBottom, navBottom))
            insets
        }
    }
}
```

---

## 주의사항 및 팁

**insets를 한 번만 소비하라.** `WindowInsetsCompat.CONSUMED`를 반환하면 자식 뷰에게 insets가 전파되지 않습니다. 루트 뷰에서 소비했다면 자식 뷰에서 중복으로 처리하지 않도록 주의하세요. 반면 `windowInsets`를 그대로 반환하면 자식 뷰까지 insets가 전파됩니다.

**windowSoftInputMode 충돌.** `adjustResize`와 `adjustPan`은 Edge-to-Edge와 충돌할 수 있습니다. IME 애니메이션을 직접 제어하려면 `SOFT_INPUT_ADJUST_NOTHING`으로 설정하고 `WindowInsetsAnimationCallback`을 사용하는 것이 권장됩니다.

**Compose에서 Scaffold innerPadding 중복 주의.** `Scaffold`의 `innerPadding`을 콘텐츠에 전달하면서 동시에 `systemBarsPadding()`을 추가하면 패딩이 두 배가 됩니다. `Scaffold`의 `contentWindowInsets = WindowInsets(0)`으로 Scaffold의 자동 처리를 비활성화하고 직접 제어할 수 있습니다.

**에뮬레이터에서는 제스처/버튼 두 가지 모두 테스트하라.** 3버튼 내비게이션 바와 제스처 내비게이션은 inset 크기가 다릅니다. 제스처 모드에서는 하단 내비게이션 바 inset이 작아지고 `systemGestures` inset이 커집니다.

**DisplayCutout은 systemBars와 함께 처리하라.** 노치나 펀치홀이 있는 기기에서는 `systemBars` inset만으로는 부족합니다. `systemBars() or displayCutout()` 또는 `safeDrawing`을 사용해야 모든 기기에서 콘텐츠가 안전하게 보입니다.

---

## 결론

Android 15에서 Edge-to-Edge가 강제화됨에 따라, WindowInsets API를 제대로 이해하고 사용하는 것은 이제 선택이 아닌 필수입니다. 핵심 포인트를 정리하면 다음과 같습니다.

1. `WindowCompat.enableEdgeToEdge(window)`로 활성화한다.
2. `ViewCompat.setOnApplyWindowInsetsListener` 또는 Compose의 `windowInsetsPadding` Modifier로 insets를 처리한다.
3. IME 애니메이션은 `WindowInsetsAnimationCallback` 또는 Compose의 `imePadding()`·`imeNestedScroll()`로 동기화한다.
4. `displayCutout`을 `systemBars`와 함께 처리하거나 `safeDrawing`을 사용한다.
5. Dialog·BottomSheet 등 별도 Window는 개별 설정이 필요하다.

Compose를 사용한다면 Material 3의 `Scaffold`가 대부분의 보일러플레이트를 처리해주지만, 커스텀 레이아웃이나 복잡한 IME 인터랙션이 필요한 경우에는 저수준 API를 직접 다룰 수 있어야 합니다.

---

## 참고 자료

- [About window insets — Jetpack Compose (Android Developers)](https://developer.android.com/develop/ui/compose/system/insets)
- [Display content edge-to-edge in views (Android Developers)](https://developer.android.com/develop/ui/views/layout/edge-to-edge)
- [Control and animate the software keyboard (Android Developers)](https://developer.android.com/develop/ui/views/layout/sw-keyboard)
