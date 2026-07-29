---
layout: post
title: "Android Predictive Back Gesture 심화 가이드: OnBackPressedCallback부터 PredictiveBackHandler까지"
date: 2026-07-29
categories: [android, flutter]
tags: [android, predictive-back, gesture, navigation, onbackpressedcallback, jetpack-compose, backhandler, kotlin]
---

Android 13부터 도입된 **Predictive Back Gesture(예측형 뒤로 가기 제스처)**는 사용자가 뒤로 스와이프하기 *전에* 어디로 이동할지 미리 시각적으로 확인할 수 있게 해주는 새로운 내비게이션 패러다임입니다. Android 15에서 기본 활성화로 전환된 이 기능을 앱에 올바르게 통합하는 방법을 심층적으로 살펴봅니다.

## 개념 설명: Predictive Back Gesture란?

기존 안드로이드의 뒤로 가기 동작은 **즉각적(imperative)**이었습니다. 사용자가 뒤로 제스처를 완료하면 화면이 전환되고, 결과를 미리 알 수 없었습니다. Predictive Back Gesture는 이 흐름을 바꿉니다.

사용자가 화면 가장자리에서 안으로 스와이프를 시작하면:
1. 현재 화면이 약간 축소되며 뒤에 있는 화면(또는 홈 화면)이 살짝 보입니다.
2. 사용자는 제스처를 **완료**하거나 **취소**할 수 있습니다.
3. 제스처가 완료되면 전환 애니메이션이 실행됩니다.

이 기능은 세 가지 레벨로 나뉩니다:

- **Back-to-Home**: 앱의 첫 화면에서 뒤로 가면 홈 화면이 미리 보입니다 (Android 13+, 옵트인 시 자동 제공).
- **Cross-Task**: 다른 앱으로의 이동을 미리 보여줍니다 (Android 14+).
- **In-App Back**: 앱 내 이전 화면을 미리 보여줍니다 (Android 14+, 직접 구현 필요).

## 왜 필요한가?

### 1. UX 개선 — 결정의 확신

사용자가 실수로 뒤로 가서 중요한 폼이나 진행 중인 작업을 잃어버리는 경험은 흔합니다. Predictive Back은 이를 방지합니다. 제스처를 시작했다가 취소하면 현재 상태가 유지됩니다.

### 2. 플랫폼 방향성 — 의무화 수순

Android 15부터 `android:enableOnBackInvokedCallback="true"` 없이 `onBackPressed()`를 오버라이드하면 시스템 애니메이션이 깨집니다. Google은 향후 이를 강제화할 예정이므로, 지금 마이그레이션하는 것이 기술 부채를 줄이는 길입니다.

### 3. 기존 API 문제

`Activity.onBackPressed()`와 `KeyEvent.KEYCODE_BACK` 처리는 deprecated 상태이며, 이들은 제스처 진행 상황(progress)을 추적할 수 없습니다. 애니메이션과 연동하려면 새 API가 필수적입니다.

## 실제 구현 예제

### 예제 1: View 시스템 — OnBackPressedCallback으로 뒤로 가기 가로채기

아래 예제는 사용자가 편집 중인 데이터가 있을 때 확인 다이얼로그를 띄우는 패턴입니다. `OnBackPressedCallback`을 활성화/비활성화하여 상태에 따라 동작을 제어합니다.

```kotlin
class EditProfileFragment : Fragment() {

    private var hasUnsavedChanges = false

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        // 1. 콜백 생성 — 초기에는 비활성화
        val backCallback = requireActivity().onBackPressedDispatcher.addCallback(
            viewLifecycleOwner,
            enabled = false
        ) {
            // 뒤로 가기 가로채기: 미저장 변경 사항 경고
            MaterialAlertDialogBuilder(requireContext())
                .setTitle("변경 사항 취소")
                .setMessage("저장하지 않은 변경 사항이 있습니다. 정말 나가시겠습니까?")
                .setPositiveButton("나가기") { _, _ ->
                    // 콜백을 비활성화하고 실제 뒤로 가기 실행
                    isEnabled = false
                    requireActivity().onBackPressedDispatcher.onBackPressed()
                }
                .setNegativeButton("취소", null)
                .show()
        }

        // 2. 텍스트 변경 시 콜백 활성화
        binding.nameEditText.doAfterTextChanged { text ->
            hasUnsavedChanges = text.toString() != originalName
            backCallback.isEnabled = hasUnsavedChanges
        }

        // 3. 저장 버튼 클릭 시 콜백 비활성화
        binding.saveButton.setOnClickListener {
            saveProfile()
            backCallback.isEnabled = false
        }
    }
}
```

**핵심 포인트:**
- `addCallback(viewLifecycleOwner, ...)`: 라이프사이클에 바인딩되어 Fragment가 파괴되면 자동 해제됩니다.
- `isEnabled = false` 후 `onBackPressed()`: 자기 자신을 비활성화한 뒤 디스패처를 다시 호출하면 스택의 다음 콜백(또는 기본 동작)이 실행됩니다.

---

### 예제 2: Jetpack Compose — PredictiveBackHandler로 애니메이션 연동

`PredictiveBackHandler`는 제스처의 **진행 상황(progress 0.0~1.0)**을 `Flow`로 제공합니다. 이를 활용하면 제스처에 맞춰 실시간으로 UI를 변형할 수 있습니다.

```kotlin
import androidx.activity.BackEventCompat
import androidx.activity.compose.PredictiveBackHandler
import androidx.compose.animation.core.animateFloatAsState
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.scale
import androidx.compose.ui.graphics.graphicsLayer
import kotlinx.coroutines.flow.Flow

@Composable
fun DetailScreen(onNavigateBack: () -> Unit) {
    // 제스처 진행도 상태 (0f = 시작, 1f = 완료 직전)
    var backProgress by remember { mutableFloatStateOf(0f) }
    var isGestureActive by remember { mutableStateOf(false) }

    // progress를 부드럽게 애니메이션화 (취소 시 0으로 복귀)
    val animatedProgress by animateFloatAsState(
        targetValue = backProgress,
        label = "back_progress"
    )

    // 스케일: 0f일 때 1.0, 1f일 때 0.9
    val scale = 1f - (animatedProgress * 0.1f)
    // 좌우 오프셋: 스와이프 방향에 따라 화면이 살짝 이동
    val translationX = animatedProgress * 80f

    PredictiveBackHandler(enabled = true) { progress: Flow<BackEventCompat> ->
        isGestureActive = true
        try {
            progress.collect { backEvent ->
                // backEvent.progress: 0.0 ~ 1.0 (제스처 완료 비율)
                // backEvent.swipeEdge: LEFT or RIGHT
                // backEvent.touchX, backEvent.touchY: 터치 좌표
                backProgress = backEvent.progress
            }
            // Flow가 완료(completed)되면 → 제스처 완료
            onNavigateBack()
        } catch (e: CancellationException) {
            // Flow가 취소(cancelled)되면 → 제스처 취소, UI 원복
            backProgress = 0f
        } finally {
            isGestureActive = false
        }
    }

    Box(
        modifier = Modifier
            .fillMaxSize()
            .graphicsLayer {
                scaleX = scale
                scaleY = scale
                this.translationX = if (isGestureActive) translationX else 0f
            }
    ) {
        // 화면 콘텐츠
        DetailContent()
    }
}
```

**핵심 포인트:**
- `progress: Flow<BackEventCompat>`: `collect` 블록에서 제스처 진행도를 받습니다. Flow가 *정상 완료*되면 "뒤로 가기 확정", `CancellationException`이 던져지면 "취소"를 의미합니다.
- `graphicsLayer`: 리컴포지션 없이 GPU 레이어 변환만 수행하므로 성능에 안전합니다.
- `animateFloatAsState`: 취소 시 progress가 갑자기 0으로 뛰지 않고 부드럽게 복귀합니다.

---

### 예제 3: Manifest 옵트인 설정

```xml
<!-- AndroidManifest.xml -->
<application
    android:enableOnBackInvokedCallback="true"
    ... >

    <!-- 특정 액티비티만 비활성화할 때 -->
    <activity
        android:name=".LegacyActivity"
        android:enableOnBackInvokedCallback="false" />
</application>
```

이 설정 없이는 Android 13/14에서 Back-to-Home 애니메이션이 표시되지 않습니다.

## 주의사항 및 팁

### 1. `onBackPressed()` 완전 탈피

`Activity.onBackPressed()`는 API 33(Android 13)에서 deprecated되었습니다. 기존 코드베이스에서 `super.onBackPressed()`를 호출하는 패턴은 전부 `OnBackPressedDispatcher`로 마이그레이션해야 합니다. Fragment 내 `onBackPressed` 오버라이드도 동일합니다.

```kotlin
// ❌ 사용하지 말 것
override fun onBackPressed() {
    super.onBackPressed()
}

// ✅ 권장 방식
onBackPressedDispatcher.addCallback(this) {
    // 처리 로직
}
```

### 2. 루트 액티비티에서의 가로채기 주의

앱의 루트 액티비티에서 `BackHandler`나 `OnBackPressedCallback`이 활성화되어 있으면, Back-to-Home 시스템 애니메이션이 **표시되지 않습니다**. 홈으로 가는 경로를 차단하지 않도록 UI 상태에 따라 콜백을 정확히 활성화/비활성화해야 합니다.

### 3. 여러 콜백이 있을 때 실행 순서

`OnBackPressedDispatcher`는 **스택** 구조입니다. 나중에 추가된 콜백이 먼저 실행됩니다. 활성화된 콜백만 실행되며, 실행된 콜백은 뒤로 가기를 "소비"하여 아래 콜백은 실행되지 않습니다.

```kotlin
// callback1 추가
dispatcher.addCallback(owner, callback1) // 먼저 추가

// callback2 추가
dispatcher.addCallback(owner, callback2) // 나중에 추가 → 먼저 실행됨

// 뒤로 가기 발생 시:
// callback2가 enabled이면 callback2만 실행
// callback2가 disabled이면 callback1 실행
```

### 4. Compose에서 `BackHandler` vs `PredictiveBackHandler`

| 구분 | BackHandler | PredictiveBackHandler |
|------|------------|----------------------|
| 용도 | 단순 가로채기 | 제스처 진행도 추적 |
| 애니메이션 연동 | 불가 | 가능 |
| API 레벨 최소 요건 | 모든 버전(AndroidX) | AndroidX Activity 1.8.0+ |
| 코드 복잡도 | 낮음 | 중간 |

진행 애니메이션이 필요 없다면 `BackHandler`로 충분합니다. 앱 브랜드에 맞는 커스텀 전환 효과를 원한다면 `PredictiveBackHandler`를 사용하세요.

### 5. 테스트 환경 설정

- **Android 13-14**: Settings → System → Developer options → Predictive back animations 활성화
- **Android 15 이상**: `android:enableOnBackInvokedCallback="true"` 설정만으로 자동 활성화
- **에뮬레이터**: Android 13+ 에뮬레이터에서 동일하게 테스트 가능

### 6. Navigation Component와의 통합

Jetpack Navigation을 사용한다면 NavController가 내부적으로 `OnBackPressedCallback`을 등록합니다. Navigation Compose 2.7.0+는 Predictive Back을 기본 지원하며, `NavHost`가 자동으로 백 스택 전환 미리보기를 처리합니다.

```kotlin
// Navigation Compose — 별도 설정 불필요
NavHost(
    navController = navController,
    startDestination = "home"
) {
    composable("home") { HomeScreen(navController) }
    composable("detail") { DetailScreen(navController) }
}
```

## 마무리

Predictive Back Gesture는 단순한 UX 개선을 넘어 Android 플랫폼의 내비게이션 철학이 변화하는 신호입니다. `onBackPressed()` 중심의 명령형 모델에서, 제스처 상태를 실시간으로 관찰하고 반응하는 **리액티브 모델**로의 전환입니다.

지금 당장 모든 앱에 완전한 커스텀 애니메이션이 필요하지는 않습니다. 먼저 Manifest 옵트인과 `OnBackPressedCallback` 마이그레이션만으로도 시스템이 제공하는 기본 애니메이션(Back-to-Home, Cross-Activity)을 무료로 얻을 수 있습니다. 이후 앱의 핵심 화면부터 `PredictiveBackHandler`로 브랜드 경험을 점진적으로 강화해 나가는 전략을 권장합니다.

## 참고 자료
- [Add support for the predictive back gesture - Android Developers](https://developer.android.com/guide/navigation/custom-back/predictive-back-gesture)
- [About Predictive Back in Jetpack Compose - Android Developers](https://developer.android.com/develop/ui/compose/system/predictive-back)
- [Provide custom back navigation - Android Developers](https://developer.android.com/guide/navigation/custom-back)
