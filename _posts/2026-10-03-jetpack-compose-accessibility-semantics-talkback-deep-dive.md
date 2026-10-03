---
layout: post
title: "Jetpack Compose Accessibility 심화: Semantics API·TalkBack·접근성 서비스 완전 정복"
date: 2026-10-03
categories: [android, flutter]
tags: [jetpack-compose, accessibility, semantics, talkback, android, kotlin]
---

모바일 앱 개발에서 접근성(Accessibility)은 종종 후순위로 밀리지만, 실제 서비스 품질과 법적 요구 사항 모두에서 빼놓을 수 없는 요소입니다. Jetpack Compose는 기존 View 시스템보다 훨씬 강력하고 일관된 접근성 모델을 제공합니다. 이 글에서는 Semantics API의 내부 동작 원리부터, TalkBack과의 실제 통합, 그리고 커스텀 컴포저블에서 완전한 접근성을 구현하는 방법까지 깊이 있게 다룹니다.

---

## 1. 왜 Compose 접근성인가?

전 세계 약 10억 명의 사람들이 어떤 형태의 장애를 가지고 있으며, 그 중 상당수가 스마트폰을 사용합니다. Android에서 가장 널리 쓰이는 접근성 서비스인 **TalkBack**은 시각 장애 사용자가 화면을 음성으로 탐색할 수 있게 합니다. **Switch Access**는 운동 장애 사용자를 위해 하나 혹은 몇 개의 스위치만으로 UI를 탐색하는 방법을 제공합니다.

기존 Android View 시스템에서는 `AccessibilityNodeInfo`를 직접 오버라이드하거나, `contentDescription`, `importantForAccessibility` 속성을 설정해야 했습니다. Compose는 이를 **Semantics Tree**라는 추상화 계층으로 통합했습니다.

### Semantics Tree란?

Compose의 UI는 **Widget Tree**, **Element Tree**, **RenderObject Tree** 이외에도 **Semantics Tree**라는 별도의 트리를 유지합니다. 이 트리는 UI의 시각적 표현이 아닌 **의미(Meaning)**를 담습니다. 접근성 서비스, 자동 완성, 테스트 프레임워크 모두 이 트리를 사용합니다.

```
// 실제 UI 트리
Column
  ├── Image(painter = heartIcon)
  └── Text("좋아요 42개")

// Semantics 트리 (merged 상태)
Node(
  contentDescription = "좋아요 42개",
  role = Role.Button,
  onClick = { /* 좋아요 토글 */ }
)
```

Semantics 트리는 두 가지 버전으로 존재합니다:
- **Unmerged Tree**: 각 컴포저블의 시맨틱 정보를 독립적으로 유지
- **Merged Tree**: `mergeDescendants = true` 설정에 의해 부모 노드로 자식의 시맨틱이 병합된 버전

Layout Inspector에서 두 버전 모두 확인할 수 있으며, 테스트 코드에서는 `useUnmergedTree` 파라미터로 선택할 수 있습니다.

---

## 2. Modifier.semantics 기초

모든 Compose 접근성 설정의 시작점은 `Modifier.semantics { }` 블록입니다.

```kotlin
Text(
    text = "제목",
    modifier = Modifier.semantics {
        // 이 노드를 헤딩으로 표시 (TalkBack이 "제목, 헤딩"으로 읽음)
        heading()
    }
)
```

### 주요 SemanticsProperties

| 프로퍼티 | 타입 | 설명 |
|---|---|---|
| `contentDescription` | String | 시각적 내용을 설명하는 텍스트 |
| `stateDescription` | String | 현재 상태 설명 (예: "선택됨") |
| `role` | Role | 컴포넌트의 역할 (Button, Image, Tab 등) |
| `disabled` | Unit | 비활성화 상태 표시 |
| `heading` | Unit | 섹션 헤딩 표시 |
| `liveRegion` | LiveRegionMode | 동적으로 변하는 영역 |
| `paneTitle` | String | 모달/다이얼로그 같은 창 제목 |
| `error` | String | 에러 메시지 |
| `progressBarRangeInfo` | ProgressBarRangeInfo | 진행 상태 범위 정보 |

---

## 3. 실전 구현 예제 1: 커스텀 좋아요 버튼

아이콘과 카운트 텍스트로 구성된 좋아요 버튼을 접근성 친화적으로 구현합니다.

```kotlin
@Composable
fun LikeButton(
    isLiked: Boolean,
    likeCount: Int,
    onToggle: () -> Unit,
    modifier: Modifier = Modifier
) {
    val likeLabel = if (isLiked) "좋아요 취소" else "좋아요"
    val countText = "${likeCount}개"
    
    // 두 요소를 하나의 접근성 노드로 병합
    Row(
        modifier = modifier
            .semantics(mergeDescendants = true) {
                // 전체 행에 대한 contentDescription 설정
                contentDescription = "$likeLabel, $countText"
                // 상태 설명
                stateDescription = if (isLiked) "활성화됨" else "비활성화됨"
                // 역할 지정
                role = Role.Button
                // 클릭 레이블: TalkBack이 "두 번 탭하여 좋아요" 대신 표시할 내용
                onClick(
                    label = likeLabel,
                    action = null // null이면 Modifier.clickable의 실제 액션을 사용
                )
            }
            .clickable(onClick = onToggle)
            .padding(8.dp),
        verticalAlignment = Alignment.CenterVertically
    ) {
        Icon(
            imageVector = if (isLiked) Icons.Filled.Favorite else Icons.Outlined.FavoriteBorder,
            contentDescription = null, // 부모에서 처리하므로 null
            tint = if (isLiked) Color.Red else MaterialTheme.colorScheme.onSurface
        )
        Spacer(Modifier.width(4.dp))
        Text(
            text = countText,
            style = MaterialTheme.typography.bodyMedium
        )
    }
}
```

### 핵심 포인트

1. `mergeDescendants = true`: Row 안의 Icon과 Text 시맨틱을 부모로 병합해 TalkBack이 두 번 포커스하지 않습니다.
2. `contentDescription` 명시: 병합된 노드에 명확한 설명을 제공합니다.
3. `stateDescription`: 단순 "선택됨/해제됨" 같은 상태를 사람이 이해하기 쉬운 문장으로 표현합니다.
4. 자식 Icon의 `contentDescription = null`: 부모가 전체 의미를 처리하므로 중복 설명을 방지합니다.

---

## 4. 실전 구현 예제 2: 스와이프 투 딜리트 리스트 아이템

제스처가 있는 복잡한 리스트 아이템에서 커스텀 접근성 액션을 추가하는 방법입니다. 스와이프 제스처는 접근성 서비스 사용 시 작동하지 않으므로, 동일한 기능을 접근성 액션으로 노출해야 합니다.

```kotlin
@Composable
fun SwipeToDeleteItem(
    item: TaskItem,
    onDelete: () -> Unit,
    onComplete: () -> Unit,
    modifier: Modifier = Modifier
) {
    val dismissState = rememberSwipeToDismissBoxState()
    
    // 스와이프 상태에 따라 onDelete 트리거
    LaunchedEffect(dismissState.currentValue) {
        if (dismissState.currentValue == SwipeToDismissBoxValue.EndToStart) {
            onDelete()
        }
    }

    SwipeToDismissBox(
        state = dismissState,
        backgroundContent = {
            Box(
                modifier = Modifier
                    .fillMaxSize()
                    .background(Color.Red)
                    .padding(horizontal = 20.dp),
                contentAlignment = Alignment.CenterEnd
            ) {
                Icon(Icons.Default.Delete, contentDescription = null, tint = Color.White)
            }
        },
        modifier = modifier.semantics {
            // 스와이프 제스처를 대체하는 커스텀 접근성 액션 등록
            customActions = listOf(
                CustomAccessibilityAction(
                    label = "${item.title} 삭제",
                    action = {
                        onDelete()
                        true // 액션 처리됨
                    }
                ),
                CustomAccessibilityAction(
                    label = "${item.title} 완료 표시",
                    action = {
                        onComplete()
                        true
                    }
                )
            )
        }
    ) {
        Card(
            modifier = Modifier
                .fillMaxWidth()
                .semantics(mergeDescendants = true) {
                    contentDescription = buildString {
                        append(item.title)
                        if (item.isCompleted) append(", 완료됨")
                        item.dueDate?.let { append(", 마감일: $it") }
                    }
                    stateDescription = if (item.isCompleted) "완료" else "미완료"
                    role = Role.Button
                }
        ) {
            Row(
                modifier = Modifier.padding(16.dp),
                verticalAlignment = Alignment.CenterVertically
            ) {
                Checkbox(
                    checked = item.isCompleted,
                    onCheckedChange = { onComplete() },
                    modifier = Modifier.semantics {
                        // 부모가 mergeDescendants이므로 개별 설명은 제거
                        contentDescription = null
                    }
                )
                Spacer(Modifier.width(12.dp))
                Column {
                    Text(
                        text = item.title,
                        style = MaterialTheme.typography.bodyLarge,
                        textDecoration = if (item.isCompleted) TextDecoration.LineThrough else null
                    )
                    item.dueDate?.let {
                        Text(
                            text = "마감: $it",
                            style = MaterialTheme.typography.bodySmall,
                            color = MaterialTheme.colorScheme.onSurfaceVariant
                        )
                    }
                }
            }
        }
    }
}

data class TaskItem(
    val id: String,
    val title: String,
    val isCompleted: Boolean,
    val dueDate: String? = null
)
```

### 커스텀 액션의 동작 방식

TalkBack에서 아이템에 포커스된 상태에서 사용자가 **세 손가락 탭** 또는 **로컬 컨텍스트 메뉴**를 열면 `customActions`에 등록된 액션 목록이 나타납니다. 스와이프할 수 없는 사용자도 동일한 기능을 사용할 수 있습니다.

---

## 5. LiveRegion: 동적 콘텐츠 알림

화면 내용이 자동으로 변경될 때(새 메시지, 카운트다운, 업로드 진행률 등) TalkBack에 변경을 자동으로 알리려면 `liveRegion`을 사용합니다.

```kotlin
@Composable
fun UploadProgressIndicator(
    progress: Float,
    statusMessage: String
) {
    Column {
        LinearProgressIndicator(
            progress = { progress },
            modifier = Modifier
                .fillMaxWidth()
                .semantics {
                    // 진행률을 구체적인 범위 정보로 표현
                    progressBarRangeInfo = ProgressBarRangeInfo(
                        current = progress,
                        range = 0f..1f,
                        steps = 0
                    )
                    contentDescription = "업로드 진행률 ${(progress * 100).toInt()}퍼센트"
                }
        )
        
        Text(
            text = statusMessage,
            modifier = Modifier.semantics {
                // Polite: 현재 읽는 내용이 끝난 후 알림
                // Assertive: 즉시 중단하고 알림 (긴급 상황에만 사용)
                liveRegion = LiveRegionMode.Polite
            }
        )
    }
}
```

`LiveRegionMode.Polite`는 TalkBack이 현재 읽고 있는 내용을 마친 후에 변경 사항을 알립니다. `Assertive`는 즉시 끊고 알리므로 오류 메시지 같은 긴급 상황에만 사용하세요.

---

## 6. 접근성 트리 디버깅

### printToLog 활용

```kotlin
// 테스트 또는 디버그 빌드에서 시맨틱 트리 출력
class AccessibilityDebugActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            MyScreen()
        }
    }
    
    override fun onResume() {
        super.onResume()
        if (BuildConfig.DEBUG) {
            // ViewTree에서 SemanticOwner 접근
            window.decorView.post {
                // 실제로는 ComposeView에서 접근
                // 테스트 코드에서는 composeTestRule.onRoot().printToLog("A11y")
            }
        }
    }
}
```

### 테스트에서 시맨틱 검증

```kotlin
@Test
fun likeButton_accessibility_semantics() {
    composeTestRule.setContent {
        LikeButton(
            isLiked = false,
            likeCount = 42,
            onToggle = {}
        )
    }

    // Merged 트리에서 검증
    composeTestRule
        .onNodeWithContentDescription("좋아요, 42개")
        .assertHasClickAction()
        .assertIsEnabled()

    // 역할(Role) 검증
    composeTestRule
        .onNodeWithContentDescription("좋아요, 42개")
        .assert(hasRole(Role.Button))
    
    // 언머지드 트리에서 개별 노드 확인
    composeTestRule
        .onAllNodes(hasAnyAncestor(hasContentDescription("좋아요, 42개")), useUnmergedTree = true)
        .assertCountEquals(2) // Icon + Text
}
```

---

## 7. 최소 터치 타겟 크기 보장

WCAG 2.1과 Material Design 가이드라인은 최소 48dp×48dp 터치 타겟을 권장합니다. 컴포저블이 시각적으로 작더라도 터치 영역을 늘리는 방법:

```kotlin
@Composable
fun SmallIconButton(
    icon: ImageVector,
    description: String,
    onClick: () -> Unit
) {
    Box(
        modifier = Modifier
            // 시각적 크기는 24dp이지만 터치 영역은 48dp
            .size(48.dp)
            .clickable(
                onClick = onClick,
                // ripple을 작게 유지하면서 터치 영역은 48dp
                indication = ripple(bounded = false, radius = 12.dp),
                interactionSource = remember { MutableInteractionSource() }
            )
            .semantics {
                contentDescription = description
                role = Role.Button
            },
        contentAlignment = Alignment.Center
    ) {
        Icon(
            imageVector = icon,
            contentDescription = null, // 부모에서 처리
            modifier = Modifier.size(24.dp)
        )
    }
}
```

`Modifier.minimumInteractiveComponentSize()`를 사용하면 더 간단하게 최소 크기를 보장할 수 있습니다.

```kotlin
Icon(
    imageVector = Icons.Default.Close,
    contentDescription = "닫기",
    modifier = Modifier
        .minimumInteractiveComponentSize() // 자동으로 48dp 보장
        .clickable { onClose() }
)
```

---

## 8. 주의사항 및 실전 팁

### 1. contentDescription 남용 금지

모든 컴포저블에 `contentDescription`을 설정하는 것은 오히려 해롭습니다. `Text` 컴포저블은 이미 텍스트 내용을 시맨틱으로 노출하므로 별도 설정이 불필요합니다. 아이콘, 이미지, 장식적 요소에만 명시적으로 설정하세요.

```kotlin
// 나쁜 예 - 불필요한 중복
Text(
    text = "제목",
    modifier = Modifier.semantics { contentDescription = "제목" } // 중복!
)

// 좋은 예
Icon(
    imageVector = Icons.Default.Search,
    contentDescription = "검색" // 아이콘에는 필요
)
```

### 2. clearAndSetSemantics 활용

기존 컴포저블의 시맨틱을 완전히 초기화하고 새로 설정할 때 사용합니다.

```kotlin
// Rating 컴포저블 내부에서 별 5개를 개별 노드 대신 하나로 표현
Row(
    modifier = Modifier.clearAndSetSemantics {
        contentDescription = "평점 4.5점 (5점 만점)"
        role = Role.Image
    }
) {
    repeat(5) { index ->
        Icon(/* 별 아이콘 */)
    }
}
```

### 3. 접근성 서비스 활성화 여부 감지

접근성 서비스가 활성화됐을 때 더 풍부한 피드백을 제공하거나, 특정 UI를 변경하고 싶을 때:

```kotlin
@Composable
fun rememberIsAccessibilityEnabled(): Boolean {
    val context = LocalContext.current
    return remember {
        val am = context.getSystemService(Context.ACCESSIBILITY_SERVICE) as AccessibilityManager
        am.isEnabled && am.isTouchExplorationEnabled
    }
}

@Composable
fun AdaptiveCard(content: String) {
    val isAccessibilityEnabled = rememberIsAccessibilityEnabled()
    
    Card(
        modifier = Modifier
            .fillMaxWidth()
            // 접근성 모드에서는 더 큰 패딩 적용
            .padding(if (isAccessibilityEnabled) 16.dp else 8.dp)
    ) {
        Text(
            text = content,
            // 접근성 모드에서는 더 큰 폰트
            style = if (isAccessibilityEnabled) 
                MaterialTheme.typography.bodyLarge
            else 
                MaterialTheme.typography.bodyMedium
        )
    }
}
```

### 4. 포커스 순서 제어

시각적 레이아웃과 논리적 탐색 순서가 다를 때 `focusProperties`와 시맨틱을 조합합니다.

```kotlin
val firstItem = remember { FocusRequester() }
val secondItem = remember { FocusRequester() }
val thirdItem = remember { FocusRequester() }

// 두 번째 아이템이 첫 번째 다음에 오도록 강제
Column {
    TextField(
        value = firstName,
        onValueChange = { firstName = it },
        modifier = Modifier
            .focusRequester(firstItem)
            .focusProperties { next = secondItem }
    )
    // 시각적으로는 세 번째지만 논리적으로는 두 번째
    TextField(
        value = lastName,
        onValueChange = { lastName = it },
        modifier = Modifier
            .focusRequester(thirdItem)
            .focusProperties { next = firstItem }
    )
    TextField(
        value = middleName,
        onValueChange = { middleName = it },
        modifier = Modifier
            .focusRequester(secondItem)
            .focusProperties { next = thirdItem }
    )
}
```

---

## 9. 체크리스트: 릴리스 전 접근성 검수

프로덕션 배포 전 다음 항목을 반드시 확인하세요:

- [ ] 모든 이미지와 아이콘에 의미 있는 `contentDescription` 또는 `contentDescription = null` (장식용)
- [ ] 클릭 가능한 모든 요소가 최소 48dp×48dp 터치 타겟을 가짐
- [ ] 색상만으로 상태를 전달하지 않음 (색상 + 텍스트/아이콘 병행)
- [ ] 동적으로 변하는 콘텐츠에 `liveRegion` 설정
- [ ] 스와이프, 드래그 같은 커스텀 제스처에 대응하는 `customActions` 제공
- [ ] TalkBack 활성화 후 전체 화면 탐색 테스트 완료
- [ ] Switch Access로 키보드 탐색 테스트 완료
- [ ] `컴포즈테스트룰`로 시맨틱 속성 자동 검증 추가

---

## 참고 자료

- [Semantics in Jetpack Compose](https://developer.android.com/jetpack/compose/semantics)
- [Accessibility in Jetpack Compose Codelab](https://developer.android.com/codelabs/jetpack-compose-accessibility)
