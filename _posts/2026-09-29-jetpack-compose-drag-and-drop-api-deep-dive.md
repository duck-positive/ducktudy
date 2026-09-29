---
layout: post
title: "Jetpack Compose Drag & Drop API 심화: dragAndDropSource와 dragAndDropTarget 완전 정복"
date: 2026-09-29
categories: [android, flutter]
tags: [android, jetpack-compose, drag-and-drop, dragAndDropSource, dragAndDropTarget, ClipData, kotlin]
---

## 개요

드래그 앤 드롭(Drag & Drop)은 사용자가 UI 요소를 집어서 다른 위치에 내려놓는 인터랙션으로, 파일 관리, 칸반 보드, 이미지 에디터 등 다양한 앱에서 필수적인 UX 패턴입니다. Jetpack Compose 1.5(BOM 2023.08.00)부터 공식적으로 `dragAndDropSource`와 `dragAndDropTarget` Modifier가 안정화되면서, 기존 View 시스템의 복잡한 구현 대비 훨씬 선언적이고 간결한 방식으로 드래그 앤 드롭을 구현할 수 있게 되었습니다.

이 글에서는 Compose 드래그 앤 드롭 API의 내부 동작 원리부터 실전 패턴까지 깊이 있게 살펴봅니다.

---

## 왜 Compose 전용 Drag & Drop API가 필요한가?

기존 View 시스템에서는 드래그 앤 드롭을 구현하려면 다음과 같은 단계를 거쳐야 했습니다.

1. `View.startDragAndDrop()` 호출
2. `View.OnDragListener` 등록
3. `DragEvent.ACTION_DROP`, `DragEvent.ACTION_DRAG_ENTERED` 등 이벤트를 수동으로 처리

이 방식은 코드량이 많고, 상태 관리가 복잡하며, Compose의 선언적 UI 패러다임과 어울리지 않습니다.

Compose 전용 API는 다음 문제를 해결합니다.

- **선언적 상태 관리**: 드래그 상태를 `remember`와 결합해 Composable 내부에서 자연스럽게 관리
- **ClipData 기반 데이터 교환**: Android 시스템 표준 데이터 교환 인터페이스와 완전히 호환
- **크로스 앱 드래그 앤 드롭**: `DRAG_FLAG_GLOBAL` 플래그를 통해 다른 앱과도 데이터 교환 가능
- **Interop 보장**: View 기반 구현과 혼용해도 동일한 `DragEvent`를 공유

---

## 핵심 API 구조

### DragAndDropTransferData

드래그 시 전송할 데이터를 캡슐화하는 클래스입니다.

```kotlin
data class DragAndDropTransferData(
    val clipData: ClipData,          // 실제 전송 데이터
    val flags: Int = 0,             // DRAG_FLAG_GLOBAL 등
    val localState: Any? = null     // 같은 Activity 내 임시 상태
)
```

- `clipData`: 텍스트, URI, HTML 등 다양한 MIME 타입 지원
- `flags = View.DRAG_FLAG_GLOBAL`: 다른 앱으로 데이터 전송 허용
- `localState`: 같은 Activity 내에서만 유효한 객체 참조 (성능 최적화용)

### DragAndDropTarget 인터페이스

드롭 대상에서 발생하는 이벤트를 처리하는 인터페이스입니다.

```kotlin
interface DragAndDropTarget {
    fun onStarted(event: DragAndDropEvent) { }    // 드래그 세션 시작
    fun onEntered(event: DragAndDropEvent) { }    // 드롭 영역 진입
    fun onMoved(event: DragAndDropEvent) { }      // 드롭 영역 내 이동
    fun onExited(event: DragAndDropEvent) { }     // 드롭 영역 이탈
    fun onChanged(event: DragAndDropEvent) { }    // 드래그 상태 변경
    fun onEnded(event: DragAndDropEvent) { }      // 드래그 세션 종료
    fun onDrop(event: DragAndDropEvent): Boolean  // 드롭 발생 (필수 구현)
}
```

`onDrop`은 반환값이 `Boolean`이며, `true`를 반환해야 드롭이 성공적으로 처리됩니다.

---

## 실제 구현 예제

### 예제 1: 텍스트 드래그 앤 드롭 (같은 앱 내)

아이템 목록에서 텍스트를 드래그해 다른 영역에 드롭하는 가장 기본적인 패턴입니다.

```kotlin
import androidx.compose.foundation.draganddrop.dragAndDropSource
import androidx.compose.foundation.draganddrop.dragAndDropTarget
import androidx.compose.foundation.gestures.detectTapGestures
import androidx.compose.ui.draganddrop.*

@Composable
fun DragAndDropTextDemo() {
    var droppedText by remember { mutableStateOf("여기에 드롭하세요") }
    var isTargetHovered by remember { mutableStateOf(false) }

    Row(
        modifier = Modifier
            .fillMaxWidth()
            .padding(16.dp),
        horizontalArrangement = Arrangement.SpaceEvenly
    ) {
        // 드래그 소스
        Box(
            modifier = Modifier
                .size(120.dp)
                .background(Color(0xFF1A73E8), RoundedCornerShape(12.dp))
                .dragAndDropSource {
                    detectTapGestures(
                        onLongPress = { offset ->
                            startTransfer(
                                DragAndDropTransferData(
                                    clipData = ClipData.newPlainText(
                                        "drag_label",
                                        "Jetpack Compose DnD!"
                                    ),
                                    localState = "source_box"
                                )
                            )
                        }
                    )
                },
            contentAlignment = Alignment.Center
        ) {
            Text(
                text = "길게 눌러\n드래그",
                color = Color.White,
                textAlign = TextAlign.Center,
                fontSize = 14.sp
            )
        }

        // 드롭 타겟
        val dropTarget = remember {
            object : DragAndDropTarget {
                override fun onEntered(event: DragAndDropEvent) {
                    isTargetHovered = true
                }

                override fun onExited(event: DragAndDropEvent) {
                    isTargetHovered = false
                }

                override fun onEnded(event: DragAndDropEvent) {
                    isTargetHovered = false
                }

                override fun onDrop(event: DragAndDropEvent): Boolean {
                    val clipData = event.toAndroidDragEvent().clipData
                    droppedText = clipData?.getItemAt(0)?.text?.toString()
                        ?: return false
                    isTargetHovered = false
                    return true
                }
            }
        }

        Box(
            modifier = Modifier
                .size(120.dp)
                .background(
                    color = if (isTargetHovered) Color(0xFF34A853) else Color(0xFFEA4335),
                    shape = RoundedCornerShape(12.dp)
                )
                .dragAndDropTarget(
                    shouldStartDragAndDrop = { event ->
                        event.mimeTypes().contains(ClipDescription.MIMETYPE_TEXT_PLAIN)
                    },
                    target = dropTarget
                ),
            contentAlignment = Alignment.Center
        ) {
            Text(
                text = droppedText,
                color = Color.White,
                textAlign = TextAlign.Center,
                fontSize = 12.sp,
                modifier = Modifier.padding(8.dp)
            )
        }
    }
}
```

`isTargetHovered` 상태를 활용해 드롭 타겟의 색상을 변경함으로써 사용자에게 드롭 가능 여부를 시각적으로 피드백합니다.

### 예제 2: 칸반 보드 - 카드 드래그 이동

실전에서 자주 쓰이는 칸반 보드 패턴입니다. `localState`를 활용해 객체 참조를 직접 전달합니다.

```kotlin
data class KanbanCard(val id: Int, val title: String, val column: String)

@Composable
fun KanbanBoard() {
    val columns = listOf("할 일", "진행 중", "완료")
    var cards by remember {
        mutableStateOf(
            listOf(
                KanbanCard(1, "UI 디자인", "할 일"),
                KanbanCard(2, "API 연동", "할 일"),
                KanbanCard(3, "코드 리뷰", "진행 중"),
                KanbanCard(4, "테스트 작성", "진행 중"),
                KanbanCard(5, "배포", "완료"),
            )
        )
    }

    Row(
        modifier = Modifier
            .fillMaxWidth()
            .horizontalScroll(rememberScrollState()),
        horizontalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        columns.forEach { columnName ->
            KanbanColumn(
                title = columnName,
                cards = cards.filter { it.column == columnName },
                onCardDropped = { card ->
                    cards = cards.map {
                        if (it.id == card.id) it.copy(column = columnName) else it
                    }
                }
            )
        }
    }
}

@Composable
fun KanbanColumn(
    title: String,
    cards: List<KanbanCard>,
    onCardDropped: (KanbanCard) -> Unit
) {
    var isHovered by remember { mutableStateOf(false) }

    val columnDropTarget = remember(title) {
        object : DragAndDropTarget {
            override fun onEntered(event: DragAndDropEvent) { isHovered = true }
            override fun onExited(event: DragAndDropEvent) { isHovered = false }
            override fun onEnded(event: DragAndDropEvent) { isHovered = false }

            override fun onDrop(event: DragAndDropEvent): Boolean {
                // localState로 카드 객체를 직접 참조 (직렬화 불필요)
                val card = event.toAndroidDragEvent().localState as? KanbanCard
                    ?: return false
                onCardDropped(card)
                isHovered = false
                return true
            }
        }
    }

    Column(
        modifier = Modifier
            .width(200.dp)
            .fillMaxHeight()
            .background(
                color = if (isHovered) Color(0xFFE8F5E9) else Color(0xFFF5F5F5),
                shape = RoundedCornerShape(8.dp)
            )
            .border(
                width = if (isHovered) 2.dp else 0.dp,
                color = if (isHovered) Color(0xFF4CAF50) else Color.Transparent,
                shape = RoundedCornerShape(8.dp)
            )
            .dragAndDropTarget(
                shouldStartDragAndDrop = { event ->
                    // ClipData의 MIME 타입으로 드롭 수락 여부 결정
                    event.mimeTypes().contains("application/kanban-card")
                },
                target = columnDropTarget
            )
            .padding(8.dp)
    ) {
        Text(
            text = "$title (${cards.size})",
            fontWeight = FontWeight.Bold,
            modifier = Modifier.padding(8.dp)
        )

        cards.forEach { card ->
            KanbanCardItem(card = card)
            Spacer(modifier = Modifier.height(4.dp))
        }
    }
}

@Composable
fun KanbanCardItem(card: KanbanCard) {
    Card(
        modifier = Modifier
            .fillMaxWidth()
            .dragAndDropSource {
                detectTapGestures(
                    onLongPress = {
                        startTransfer(
                            DragAndDropTransferData(
                                // MIME 타입을 커스텀으로 지정해 타겟 필터링 정확도 향상
                                clipData = ClipData(
                                    ClipDescription(
                                        "kanban_card",
                                        arrayOf("application/kanban-card")
                                    ),
                                    ClipData.Item(card.title)
                                ),
                                localState = card  // 같은 Activity 내에서 객체 직접 전달
                            )
                        )
                    }
                )
            },
        elevation = CardDefaults.cardElevation(defaultElevation = 2.dp)
    ) {
        Text(
            text = card.title,
            modifier = Modifier.padding(12.dp),
            fontSize = 14.sp
        )
    }
}
```

커스텀 MIME 타입 `"application/kanban-card"`를 사용하면 `shouldStartDragAndDrop` 필터에서 다른 드래그 세션과 명확히 구분할 수 있습니다.

---

## 크로스 앱 드래그 앤 드롭

Android 7.0(API 24)부터 지원되는 크로스 앱 드래그 앤 드롭은 `DRAG_FLAG_GLOBAL` 플래그와 URI 권한 처리가 핵심입니다.

```kotlin
// 드래그 소스 - 다른 앱으로 URI 전달
Modifier.dragAndDropSource {
    detectTapGestures(
        onLongPress = {
            startTransfer(
                DragAndDropTransferData(
                    clipData = ClipData.newUri(
                        contentResolver,
                        "image",
                        imageUri
                    ),
                    flags = View.DRAG_FLAG_GLOBAL or
                            View.DRAG_FLAG_GLOBAL_URI_READ
                )
            )
        }
    )
}

// 드롭 타겟 - 외부 앱으로부터 URI 수신 시 권한 요청
object : DragAndDropTarget {
    override fun onDrop(event: DragAndDropEvent): Boolean {
        // 외부 앱의 content URI에 접근하기 위한 임시 권한 요청
        val permission = activity.requestDragAndDropPermissions(
            event.toAndroidDragEvent()
        )
        val uri = event.toAndroidDragEvent()
            .clipData
            ?.getItemAt(0)
            ?.uri
            ?: return false

        // URI 처리 로직
        processUri(uri)

        // 작업 완료 후 권한 반환
        permission?.release()
        return true
    }
}
```

`requestDragAndDropPermissions()`는 드롭된 외부 URI에 대한 임시 읽기 권한을 부여하며, 사용 후 반드시 `release()`를 호출해 권한을 반환해야 합니다.

---

## 주의사항 및 실전 팁

### 1. `shouldStartDragAndDrop` 람다는 기억 최소화

`shouldStartDragAndDrop` 람다는 드래그 세션이 시작될 때마다 호출됩니다. 무거운 연산을 넣으면 성능 저하가 발생합니다.

```kotlin
// 나쁜 예 - 매번 새 컬렉션 생성
shouldStartDragAndDrop = { event ->
    val allowedTypes = listOf("image/png", "image/jpeg", "image/webp") // 매번 생성
    event.mimeTypes().any { it in allowedTypes }
}

// 좋은 예 - 상수로 미리 정의
private val ALLOWED_IMAGE_TYPES = setOf("image/png", "image/jpeg", "image/webp")

shouldStartDragAndDrop = { event ->
    event.mimeTypes().any { it in ALLOWED_IMAGE_TYPES }
}
```

### 2. `localState`는 같은 Activity 내에서만 유효

`DragAndDropTransferData.localState`에 넣은 객체는 직렬화되지 않습니다. 따라서 서로 다른 앱, 또는 서로 다른 Activity 간 드래그 앤 드롭에서는 `null`이 반환됩니다. 크로스 앱 시나리오에서는 반드시 `ClipData`를 통해 데이터를 전달해야 합니다.

```kotlin
override fun onDrop(event: DragAndDropEvent): Boolean {
    // localState는 크로스 앱에서 null일 수 있으므로 방어적으로 처리
    val localCard = event.toAndroidDragEvent().localState as? KanbanCard
    if (localCard != null) {
        // 같은 앱 내 드롭: 객체 직접 사용
        handleLocalDrop(localCard)
    } else {
        // 크로스 앱 드롭: ClipData에서 파싱
        val text = event.toAndroidDragEvent().clipData?.getItemAt(0)?.text
        handleExternalDrop(text)
    }
    return true
}
```

### 3. `DragAndDropTarget`을 `remember`로 안정화

`dragAndDropTarget`의 `target` 파라미터는 람다 내부에서 매 Recomposition마다 새로 생성되면 안 됩니다. 반드시 `remember`로 감싸야 합니다.

```kotlin
// 나쁜 예 - Recomposition마다 새 객체 생성 → 예상치 못한 동작
Modifier.dragAndDropTarget(
    shouldStartDragAndDrop = { true },
    target = object : DragAndDropTarget {  // 매번 새 인스턴스
        override fun onDrop(event: DragAndDropEvent) = true
    }
)

// 좋은 예
val target = remember {
    object : DragAndDropTarget {
        override fun onDrop(event: DragAndDropEvent) = true
    }
}
Modifier.dragAndDropTarget(
    shouldStartDragAndDrop = { true },
    target = target
)
```

### 4. 드래그 중 Shadow 커스터마이징

`dragAndDropSource`의 두 번째 파라미터인 `dragDecorationPainter`를 활용하면 드래그 중인 아이템의 그림자(shadow)를 커스터마이징할 수 있습니다.

```kotlin
Modifier.dragAndDropSource(
    drawDragDecoration = {
        drawRoundRect(
            color = Color(0x80000000),
            cornerRadius = CornerRadius(8.dp.toPx()),
        )
    }
) {
    detectTapGestures(onLongPress = { startTransfer(...) })
}
```

### 5. API 레벨 분기

`dragAndDropSource` / `dragAndDropTarget`은 내부적으로 `View.startDragAndDrop()`을 사용하므로 최소 API 24(Android 7.0)가 필요합니다. 더 낮은 버전을 지원해야 한다면 `Build.VERSION.SDK_INT` 분기가 필요합니다.

---

## 마무리

Jetpack Compose의 `dragAndDropSource`와 `dragAndDropTarget` API는 기존 View 시스템의 복잡한 드래그 앤 드롭 구현을 선언적으로 단순화합니다. `ClipData` 기반의 표준 인터페이스 덕분에 앱 내 드래그는 물론, 다른 앱과의 데이터 교환까지 일관된 방식으로 처리할 수 있습니다.

칸반 보드, 이미지 갤러리, 파일 관리, 멀티 윈도우 지원 앱 등 다양한 시나리오에서 이 API를 활용해 직관적인 UX를 구현해 보세요.

## 참고 자료
- [Jetpack Compose Drag and Drop 공식 문서](https://developer.android.com/develop/ui/compose/touch-input/user-interactions/drag-and-drop)
- [Drag and Drop in Compose Codelab](https://developer.android.com/codelabs/codelab-dnd-compose)
- [androidx.compose.foundation.draganddrop 패키지 레퍼런스](https://developer.android.com/reference/kotlin/androidx/compose/foundation/draganddrop/package-summary)
