---
layout: post
title: "Android 터치 이벤트 디스패치 완전 정복: dispatchTouchEvent, onInterceptTouchEvent, onTouchEvent의 내부 동작 원리"
date: 2026-09-30
categories: [android]
tags: [android, touch, event, dispatchTouchEvent, onInterceptTouchEvent, onTouchEvent, GestureDetector, NestedScrolling, ViewGroup, kotlin]
---

Android 개발을 하다 보면 스크롤 충돌, 터치가 먹히지 않는 버튼, 부모 뷰가 자식 뷰의 제스처를 빼앗아 가는 문제를 한 번쯤 겪게 된다. 이 문제들은 모두 **터치 이벤트 디스패치 메커니즘**을 이해하지 못할 때 발생한다. 이 글에서는 Android View 시스템이 손가락 하나의 터치를 어떻게 앱 계층 전체로 전달하는지, 그 내부 구조를 Kotlin 코드 예제와 함께 완전히 분석한다.

---

## 1. 개념 설명: 터치 이벤트는 어디서 시작되는가?

### 하드웨어 → 커널 → Android 프레임워크

손가락이 화면에 닿는 순간, 터치 컨트롤러 IC가 인터럽트를 발생시켜 리눅스 커널의 `evdev` 드라이버로 원시 데이터를 전달한다. Android는 `EventHub`와 `InputReader` 스레드가 이 데이터를 읽어 `MotionEvent` 객체로 변환한 뒤, `InputDispatcher`가 올바른 창(Window)으로 이벤트를 라우팅한다.

앱 프로세스에서는 `ViewRootImpl`이 `InputEventReceiver`를 통해 이벤트를 수신하고, `DecorView.dispatchTouchEvent()`를 호출하면서 View 계층 트리를 타고 내려가는 **디스패치 여정**이 시작된다.

### MotionEvent의 핵심 액션

| 액션 상수 | 의미 |
|---|---|
| `ACTION_DOWN` | 첫 번째 포인터가 화면에 닿음. 제스처의 시작 |
| `ACTION_MOVE` | 포인터가 이동 중 |
| `ACTION_UP` | 마지막 포인터가 화면에서 떨어짐. 제스처 종료 |
| `ACTION_CANCEL` | 부모가 이벤트를 가로채는 등 제스처가 강제 종료됨 |
| `ACTION_POINTER_DOWN` | 멀티터치: 추가 포인터 접촉 |
| `ACTION_POINTER_UP` | 멀티터치: 추가 포인터 분리 |

`ACTION_DOWN`은 새로운 제스처 시퀀스의 시작을 알리는 특별한 이벤트다. View 시스템은 `DOWN` 이벤트를 기준으로 "누가 이 제스처를 처리할 것인가"를 결정한다.

---

## 2. 왜 이 메커니즘이 필요한가?

현대 앱의 UI는 단순히 버튼 하나를 누르는 것이 아니다. `RecyclerView` 안에 `ViewPager2`가 있고, 그 안에 또 `SwipeRefreshLayout`이 중첩된 구조가 흔하다. 이런 복잡한 계층에서:

- **수평 스와이프**는 `ViewPager2`가 처리해야 하고
- **수직 스크롤**은 `RecyclerView`가 처리해야 하며
- **당겨서 새로고침**은 `SwipeRefreshLayout`이 처리해야 한다

이 세 가지가 동시에 존재할 때, 각 ViewGroup이 "내 제스처인가, 자식 뷰에게 넘겨야 하는가"를 결정하는 체계가 바로 `dispatchTouchEvent` / `onInterceptTouchEvent` / `onTouchEvent` 삼총사다.

---

## 3. 세 메서드의 역할과 호출 흐름

### 3.1 dispatchTouchEvent(event: MotionEvent): Boolean

모든 View와 ViewGroup이 가지는 메서드. 이벤트를 받아서 **어디로 보낼지** 결정하는 라우터 역할이다.

- ViewGroup의 경우: 자식 뷰들을 순회하며 적절한 자식에게 이벤트를 위임하거나, `onInterceptTouchEvent()`를 통해 직접 처리할지 결정한다.
- View의 경우(leaf node): `onTouchEvent()`를 직접 호출한다.
- 반환값이 `true`이면 이벤트가 소비된 것이고, `false`이면 부모로 이벤트 처리 기회가 돌아간다.

### 3.2 onInterceptTouchEvent(event: MotionEvent): Boolean

**ViewGroup에만 존재**하는 메서드. `dispatchTouchEvent()` 내부에서 호출된다.

- `false` 반환(기본값): 자식 View에게 이벤트를 넘긴다.
- `true` 반환: 이벤트를 가로채고, 자식에게는 `ACTION_CANCEL`을 발송한 뒤 자신의 `onTouchEvent()`로 이벤트를 처리한다.

### 3.3 onTouchEvent(event: MotionEvent): Boolean

실제 제스처 처리 로직이 들어가는 메서드.

- `true` 반환: 이벤트 소비. 이 View가 이후 이벤트(`MOVE`, `UP`)도 계속 받는다.
- `false` 반환: 이벤트 미소비. 부모 View의 `onTouchEvent()`로 이벤트가 올라간다.

### 3.4 전체 흐름 다이어그램

```
Activity.dispatchTouchEvent()
    └─ DecorView.dispatchTouchEvent()
        └─ ViewGroup A (e.g. RecyclerView)
            ├─ onInterceptTouchEvent() → false: 자식에게 위임
            └─ ViewGroup B (e.g. CardView)
                ├─ onInterceptTouchEvent() → false: 자식에게 위임
                └─ View C (e.g. Button)
                    └─ onTouchEvent() → true: 이벤트 소비
```

만약 View C의 `onTouchEvent()`가 `false`를 반환하면:

```
View C.onTouchEvent() → false
    ↑ ViewGroup B.onTouchEvent() → false
        ↑ ViewGroup A.onTouchEvent() → false
            ↑ Activity.onTouchEvent() → (최후의 처리)
```

---

## 4. 코드 예제 1: 수평/수직 제스처 충돌 해결

다음은 `RecyclerView` 내부에 `ViewPager2`가 있을 때 수평 스와이프만 `ViewPager2`로 전달하고, 수직 스크롤은 `RecyclerView`에서 처리하도록 `onInterceptTouchEvent()`를 재정의한 예제다.

```kotlin
class HorizontalGestureInterceptor(context: Context) : ViewPager2(context) {

    private val touchSlop = ViewConfiguration.get(context).scaledTouchSlop
    private var initialX = 0f
    private var initialY = 0f

    override fun onInterceptTouchEvent(ev: MotionEvent): Boolean {
        when (ev.actionMasked) {
            MotionEvent.ACTION_DOWN -> {
                initialX = ev.x
                initialY = ev.y
                // DOWN은 무조건 자식에게 전달하여 자식이 클릭을 인식할 수 있게 한다
                parent.requestDisallowInterceptTouchEvent(true)
            }
            MotionEvent.ACTION_MOVE -> {
                val dx = Math.abs(ev.x - initialX)
                val dy = Math.abs(ev.y - initialY)
                if (dx > touchSlop && dx > dy) {
                    // 수평 움직임이 명확하면 부모(RecyclerView)가 가로채지 못하게 막는다
                    parent.requestDisallowInterceptTouchEvent(true)
                } else {
                    // 수직 움직임이면 부모가 가로챌 수 있도록 허용
                    parent.requestDisallowInterceptTouchEvent(false)
                }
            }
        }
        return super.onInterceptTouchEvent(ev)
    }
}
```

**핵심 포인트:**

- `requestDisallowInterceptTouchEvent(true)`: 부모 ViewGroup의 `onInterceptTouchEvent()`가 호출되지 않도록 플래그를 설정한다. 이 플래그는 `FLAG_DISALLOW_INTERCEPT`로 구현되어 있다.
- `ACTION_DOWN` 시점에서 판단하지 않고 `ACTION_MOVE`에서 방향을 확인한다. `DOWN` 시점에는 아직 의도를 알 수 없기 때문이다.
- `ViewConfiguration.scaledTouchSlop`을 사용해 의도하지 않은 미세한 손떨림을 필터링한다.

---

## 5. 코드 예제 2: 커스텀 ViewGroup의 onInterceptTouchEvent 완전 구현

실제로 내부 스크롤 가능한 자식 View(예: `WebView`)가 있고, 외부 ViewGroup은 풀다운 새로고침을 담당하는 구조를 구현한다.

```kotlin
class PullToRefreshLayout @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null
) : FrameLayout(context, attrs) {

    private val touchSlop = ViewConfiguration.get(context).scaledTouchSlop
    private var lastY = 0f
    private var isBeingDragged = false
    private var activePointerId = MotionEvent.INVALID_POINTER_ID

    override fun onInterceptTouchEvent(ev: MotionEvent): Boolean {
        if (!isEnabled) return false

        when (ev.actionMasked) {
            MotionEvent.ACTION_DOWN -> {
                activePointerId = ev.getPointerId(0)
                lastY = ev.getY(ev.findPointerIndex(activePointerId))
                isBeingDragged = false
            }

            MotionEvent.ACTION_MOVE -> {
                if (activePointerId == MotionEvent.INVALID_POINTER_ID) return false
                val pointerIndex = ev.findPointerIndex(activePointerId)
                if (pointerIndex < 0) return false

                val y = ev.getY(pointerIndex)
                val dy = y - lastY

                // 아래로 드래그 + 자식이 더 이상 위로 스크롤할 수 없을 때만 가로챈다
                if (dy > touchSlop && !canScrollUp()) {
                    isBeingDragged = true
                    lastY = y
                }
            }

            MotionEvent.ACTION_CANCEL, MotionEvent.ACTION_UP -> {
                isBeingDragged = false
                activePointerId = MotionEvent.INVALID_POINTER_ID
            }

            MotionEvent.ACTION_POINTER_UP -> {
                // 멀티터치: 활성 포인터가 들어올려졌다면 다른 포인터로 교체
                val pointerIndex = ev.actionIndex
                val pointerId = ev.getPointerId(pointerIndex)
                if (pointerId == activePointerId) {
                    val newPointerIndex = if (pointerIndex == 0) 1 else 0
                    activePointerId = ev.getPointerId(newPointerIndex)
                    lastY = ev.getY(newPointerIndex)
                }
            }
        }

        return isBeingDragged
    }

    override fun onTouchEvent(ev: MotionEvent): Boolean {
        if (!isBeingDragged) return false

        when (ev.actionMasked) {
            MotionEvent.ACTION_MOVE -> {
                val pointerIndex = ev.findPointerIndex(activePointerId)
                if (pointerIndex < 0) return false
                val dy = ev.getY(pointerIndex) - lastY
                lastY = ev.getY(pointerIndex)
                // 실제 새로고침 UI 업데이트
                moveSpinner(dy)
            }
            MotionEvent.ACTION_UP, MotionEvent.ACTION_CANCEL -> {
                // 새로고침 판단 후 애니메이션 처리
                finishSpinner()
                isBeingDragged = false
                activePointerId = MotionEvent.INVALID_POINTER_ID
            }
        }
        return true
    }

    /**
     * 자식 View가 위 방향으로 더 스크롤 가능한지 확인한다.
     * ViewCompat.canScrollVertically(child, -1): 위로 스크롤 가능 여부
     */
    private fun canScrollUp(): Boolean {
        val target = getChildAt(0) ?: return false
        return ViewCompat.canScrollVertically(target, -1)
    }

    private fun moveSpinner(dy: Float) {
        // 새로고침 인디케이터 위치 업데이트 로직 (생략)
    }

    private fun finishSpinner() {
        // 새로고침 완료/취소 처리 로직 (생략)
    }
}
```

이 코드가 핵심적으로 보여주는 것:

1. **멀티터치 포인터 추적**: `activePointerId`로 특정 손가락을 추적하고, `ACTION_POINTER_UP` 시에도 추적을 유지한다.
2. **canScrollVertically()**: 자식 View가 스크롤 가능한 상태인지 확인해 불필요하게 제스처를 가로채지 않는다.
3. **상태 기계(State Machine)**: `isBeingDragged` 플래그로 드래그 중/아닌 상태를 명확히 관리한다.

---

## 6. GestureDetector: 고수준 제스처 감지

`onTouchEvent()`를 직접 다루는 것은 번거롭다. `GestureDetector`를 사용하면 더블탭, 롱 프레스, 플링 등의 고수준 제스처를 손쉽게 감지할 수 있다.

```kotlin
class InteractiveView(context: Context) : View(context) {

    private val gestureDetector = GestureDetectorCompat(context,
        object : GestureDetector.SimpleOnGestureListener() {

            override fun onDown(e: MotionEvent): Boolean {
                // GestureDetector가 제대로 동작하려면 onDown에서 반드시 true를 반환해야 한다
                return true
            }

            override fun onSingleTapConfirmed(e: MotionEvent): Boolean {
                // 더블탭과 구분된 확실한 단일 탭
                performClick()
                return true
            }

            override fun onDoubleTap(e: MotionEvent): Boolean {
                // 더블탭 처리
                zoomToggle()
                return true
            }

            override fun onLongPress(e: MotionEvent) {
                // 롱 프레스 처리
                showContextMenu()
            }

            override fun onFling(
                e1: MotionEvent?,
                e2: MotionEvent,
                velocityX: Float,
                velocityY: Float
            ): Boolean {
                // 플링 제스처: velocityX, velocityY는 px/sec 단위
                val minFlingVelocity = ViewConfiguration.get(context).scaledMinimumFlingVelocity
                return if (Math.abs(velocityX) > minFlingVelocity) {
                    handleFling(velocityX)
                    true
                } else false
            }

            override fun onScroll(
                e1: MotionEvent?,
                e2: MotionEvent,
                distanceX: Float,
                distanceY: Float
            ): Boolean {
                // 스크롤 처리: distanceX/Y는 이전 이벤트와의 차이 (양수 = 오른쪽/아래)
                scrollBy(distanceX.toInt(), distanceY.toInt())
                return true
            }
        }
    )

    override fun onTouchEvent(event: MotionEvent): Boolean {
        return gestureDetector.onTouchEvent(event) || super.onTouchEvent(event)
    }

    private fun zoomToggle() { /* 줌 토글 */ }
    private fun showContextMenu() { /* 컨텍스트 메뉴 */ }
    private fun handleFling(velocity: Float) { /* 플링 처리 */ }
}
```

**주의**: `onDown()`에서 `true`를 반환하지 않으면 `GestureDetector`가 이후 이벤트를 무시한다. `SimpleOnGestureListener`는 모든 메서드의 기본 반환값이 `false`이므로, 반드시 `onDown()`을 오버라이드해야 한다.

---

## 7. NestedScrolling 프로토콜

Android 5.0(API 21)부터 도입된 `NestedScrollingChild` / `NestedScrollingParent` 인터페이스는 전통적인 `onInterceptTouchEvent()` 방식의 한계를 넘어선다. 기존 방식은 "부모 아니면 자식" 중 하나만 이벤트를 처리할 수 있었지만, `NestedScrolling`은 **부모와 자식이 협력**하여 스크롤량을 분배할 수 있다.

대표적인 예: `CoordinatorLayout` + `AppBarLayout` + `RecyclerView`

1. 사용자가 `RecyclerView`를 위로 스크롤한다.
2. `RecyclerView`(NestedScrollingChild)가 `dispatchNestedPreScroll()`을 호출해 부모에게 먼저 물어본다: "이 스크롤 중 얼마를 네가 소비할래?"
3. `CoordinatorLayout`(NestedScrollingParent)이 `AppBarLayout`을 접으면서 일부 스크롤을 소비한다.
4. 남은 스크롤량을 `RecyclerView`가 내부 리스트 스크롤에 사용한다.

이 덕분에 `AppBarLayout`이 자연스럽게 접히고, 동시에 `RecyclerView`도 스크롤되는 매끄러운 UX가 구현된다.

---

## 8. 주의사항 및 실전 팁

### 8.1 ACTION_DOWN에서 true를 반환하지 않으면 이후 이벤트를 받지 못한다

`onTouchEvent()`에서 `ACTION_DOWN`에 대해 `false`를 반환하면, Android 시스템은 "이 View는 이 제스처에 관심 없다"고 판단하고 이후 `MOVE`, `UP` 이벤트를 전혀 보내지 않는다.

### 8.2 onInterceptTouchEvent()는 ACTION_MOVE까지 반복 호출된다

`onInterceptTouchEvent()`가 `false`를 반환하는 동안에는 `ACTION_DOWN`, `ACTION_MOVE`가 계속 전달된다. 하지만 한 번이라도 `true`를 반환하면 이후로는 `onInterceptTouchEvent()` 자체가 호출되지 않고 직접 `onTouchEvent()`로 이벤트가 간다.

### 8.3 requestDisallowInterceptTouchEvent()의 수명

이 플래그는 새로운 `ACTION_DOWN` 이벤트가 오면 자동으로 초기화된다. 즉, 한 제스처 시퀀스(`DOWN` ~ `UP/CANCEL`) 내에서만 유효하다.

### 8.4 클릭 접근성을 위한 performClick() 호출

`onTouchEvent()`를 오버라이드할 때 `ACTION_UP`에서 클릭 처리를 하려면 반드시 `performClick()`을 호출해야 한다. 그래야 접근성 서비스(`TalkBack`)와 `OnClickListener`가 정상 동작한다.

```kotlin
override fun onTouchEvent(event: MotionEvent): Boolean {
    if (event.actionMasked == MotionEvent.ACTION_UP) {
        performClick() // 접근성 + 클릭 리스너 처리
    }
    return true
}

override fun performClick(): Boolean {
    super.performClick()
    // 커스텀 클릭 처리
    return true
}
```

### 8.5 TouchDelegate로 작은 뷰의 터치 영역 확대

버튼이 너무 작아 터치하기 어려울 때, `TouchDelegate`를 사용하면 부모 View의 영역을 사용해 터치 판정 영역을 넓힐 수 있다.

```kotlin
fun View.expandTouchArea(extraPadding: Int) {
    val parent = parent as? View ?: return
    parent.post {
        val rect = Rect()
        getHitRect(rect)
        rect.inset(-extraPadding, -extraPadding)
        parent.touchDelegate = TouchDelegate(rect, this)
    }
}

// 사용
smallButton.expandTouchArea(dpToPx(8)) // 모든 방향으로 8dp 확장
```

### 8.6 Compose와 View 시스템의 터치 이벤트 혼용

`AndroidView` 컴포저블 안에 기존 View를 넣거나, `ComposeView`를 기존 ViewGroup 안에 넣을 때 터치 이벤트가 올바르게 흐르지 않는 문제가 생길 수 있다. 이 경우 Compose의 `pointerInput` 모디파이어와 View의 `onTouchEvent()` 중 어느 쪽이 우선순위를 갖는지 명확히 설계해야 한다. 일반적으로 Compose는 내부적으로 `AndroidComposeView`라는 단일 View를 사용해 이벤트를 처리하므로, 해당 View의 `dispatchTouchEvent()`가 Compose의 포인터 이벤트로 변환된다.

---

## 9. 정리: 이벤트 전달 3원칙

1. **다운 스트림(자식 방향)**: `dispatchTouchEvent()` → `onInterceptTouchEvent()` → 자식의 `dispatchTouchEvent()`
2. **업 스트림(부모 방향)**: 자식이 `false` 반환 시 부모의 `onTouchEvent()`로 역류
3. **가로채기**: `onInterceptTouchEvent()`가 `true`를 반환하는 순간 자식은 `ACTION_CANCEL`을 받고 게임 아웃, 이후 이벤트는 모두 해당 ViewGroup의 `onTouchEvent()`로 직행

이 세 원칙을 머릿속에 명확히 새기고, 멀티터치·NestedScrolling·GestureDetector를 조합하면 어떤 복잡한 터치 UX라도 충돌 없이 구현할 수 있다.

---

## 참고 자료

- [Android 공식 문서: ViewGroup에서 터치 이벤트 관리](https://developer.android.com/training/gestures/viewgroup)
- [Android 공식 문서: 커스텀 뷰를 인터랙티브하게 만들기](https://developer.android.com/training/custom-views/making-interactive.html)
- [Android 공식 참조: NestedScrollingChild2](https://developer.android.com/reference/kotlin/androidx/core/view/NestedScrollingChild2)
- [Kotlin과 고급 터치 인터랙션 구현 가이드](https://reintech.io/blog/kotlin-and-gestures-building-apps-with-advanced-touch-interactions)
