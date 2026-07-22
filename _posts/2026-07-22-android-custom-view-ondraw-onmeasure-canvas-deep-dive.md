---
layout: post
title: "Android Custom View 심화: onDraw·onMeasure·Canvas API·커스텀 속성으로 픽셀 단위 뷰 완전 정복"
date: 2026-07-22
categories: [android, flutter]
tags: [android, custom-view, canvas, onDraw, onMeasure, kotlin, ui]
---

Android에서 제공하는 기본 위젯(TextView, Button, ImageView 등)은 대부분의 UI 요구를 충족하지만, 디자인 시스템의 요구나 비즈니스 로직이 복잡해질수록 기본 위젯만으로는 한계가 있습니다. 이때 등장하는 것이 **Custom View**입니다. 이 글에서는 `onMeasure()`, `onDraw()`, Canvas API, 커스텀 속성(attrs.xml)을 중심으로 실전 Custom View 구현 방법을 심화 탐구합니다.

---

## 1. Custom View란 무엇인가?

Custom View는 `View` 클래스를 직접 상속하여 원하는 모양과 동작을 픽셀 단위로 제어할 수 있는 강력한 도구입니다. Android 렌더링 파이프라인은 크게 세 단계를 거칩니다:

1. **Measure**: 뷰의 크기를 측정 (`onMeasure()`)
2. **Layout**: 뷰를 화면에 배치 (`onLayout()`)
3. **Draw**: 뷰를 실제로 그림 (`onDraw()`)

이 세 단계를 이해하고 올바르게 구현하면, 어떤 복잡한 UI도 직접 만들어낼 수 있습니다.

---

## 2. 왜 Custom View가 필요한가?

### 2.1 기존 위젯의 한계

기본 View를 조합해서 만들 수 없는 UI가 존재합니다. 예를 들어:

- **원형 진행 표시줄** (아크 기반): ProgressBar의 기본 스타일로는 세밀한 제어가 불가능
- **커스텀 차트**: 막대, 꺾은선, 파이 차트를 앱 디자인에 맞게 그려야 할 때
- **파형(Waveform) UI**: 오디오 플레이어의 파형 시각화
- **시계 아날로그 UI**: 시침/분침/초침을 정밀하게 그려야 할 때

### 2.2 성능 이점

기본 위젯을 여러 개 조합해 레이아웃 계층을 깊게 만드는 것보다, 단일 Custom View로 그리면 **View 계층이 줄어들어** Measure/Layout/Draw 패스가 단순해집니다. 특히 RecyclerView의 각 아이템처럼 수십~수백 개 반복되는 경우 이 차이는 체감됩니다.

### 2.3 재사용성

한 번 잘 만든 Custom View는 XML 속성(attrs.xml)을 통해 코드 수정 없이 다양한 화면에서 재활용할 수 있습니다. 디자이너와의 협업에서도 XML 레이아웃 속성만 바꾸면 되므로 유연성이 높습니다.

---

## 3. View 생명주기와 핵심 메서드

Custom View를 구현할 때 반드시 이해해야 할 메서드들이 있습니다:

| 메서드 | 역할 | 언제 호출되는가 |
|--------|------|----------------|
| `constructor` | 초기화 | 코드/XML로 생성할 때 |
| `onAttachedToWindow()` | 윈도우에 추가됨 | 화면에 붙을 때 |
| `onMeasure(widthSpec, heightSpec)` | 크기 결정 | 부모가 크기를 물어볼 때 |
| `onSizeChanged(w, h, old, old)` | 크기 변경 확정 | 실제 크기가 결정될 때 |
| `onLayout(changed, l, t, r, b)` | 자식 배치 | ViewGroup에서 필요 |
| `onDraw(canvas)` | 화면에 그리기 | 매 프레임마다 |
| `invalidate()` | 재드로우 요청 | 상태가 변경됐을 때 직접 호출 |
| `requestLayout()` | 재측정/배치 요청 | 크기가 바뀌었을 때 직접 호출 |

### MeasureSpec 이해하기

`onMeasure()`에서 부모가 전달하는 `MeasureSpec`은 크기와 모드를 인코딩한 32비트 정수입니다:

| 모드 | 설명 | 예시 |
|------|------|------|
| `EXACTLY` | 부모가 정확한 크기를 지정 | `layout_width="100dp"`, `match_parent` |
| `AT_MOST` | 이 크기를 넘으면 안 됨 | `wrap_content` |
| `UNSPECIFIED` | 제한 없음 | ScrollView 내부 |

`resolveSize(desiredSize, measureSpec)` 헬퍼를 사용하면 이 세 가지 경우를 자동으로 처리할 수 있습니다.

---

## 4. 실제 구현 예제 1: CircleProgressView

원형 진행 표시줄을 직접 구현해 봅니다. 퍼센트 값을 받아 아크를 그리고, 중앙에 텍스트로 값을 표시합니다.

### 4.1 커스텀 속성 정의 (res/values/attrs.xml)

```xml
<resources>
    <declare-styleable name="CircleProgressView">
        <attr name="cpv_progress" format="float" />
        <attr name="cpv_trackColor" format="color" />
        <attr name="cpv_progressColor" format="color" />
        <attr name="cpv_strokeWidth" format="dimension" />
        <attr name="cpv_showText" format="boolean" />
    </declare-styleable>
</resources>
```

### 4.2 CircleProgressView.kt

```kotlin
class CircleProgressView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : View(context, attrs, defStyleAttr) {

    var progress: Float = 0f
        set(value) {
            field = value.coerceIn(0f, 100f)
            invalidate() // 내용만 바뀌므로 invalidate()만 호출
        }

    private val trackPaint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        style = Paint.Style.STROKE
    }
    private val progressPaint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        style = Paint.Style.STROKE
        strokeCap = Paint.Cap.ROUND
    }
    private val textPaint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        textAlign = Paint.Align.CENTER
    }

    // onDraw()에서 객체 생성 금지 — 멤버 변수로 미리 생성
    private val ovalRect = RectF()
    private var showText: Boolean = true

    init {
        val ta = context.obtainStyledAttributes(attrs, R.styleable.CircleProgressView)
        try {
            progress = ta.getFloat(R.styleable.CircleProgressView_cpv_progress, 0f)
            val strokeWidth = ta.getDimension(
                R.styleable.CircleProgressView_cpv_strokeWidth,
                resources.displayMetrics.density * 12f
            )
            trackPaint.strokeWidth = strokeWidth
            progressPaint.strokeWidth = strokeWidth
            trackPaint.color = ta.getColor(
                R.styleable.CircleProgressView_cpv_trackColor,
                Color.parseColor("#E0E0E0")
            )
            progressPaint.color = ta.getColor(
                R.styleable.CircleProgressView_cpv_progressColor,
                Color.parseColor("#6200EE")
            )
            showText = ta.getBoolean(
                R.styleable.CircleProgressView_cpv_showText, true
            )
        } finally {
            ta.recycle() // TypedArray는 반드시 recycle() 호출!
        }
    }

    override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
        val desiredSize = (resources.displayMetrics.density * 100).toInt()
        val w = resolveSize(desiredSize, widthMeasureSpec)
        val h = resolveSize(desiredSize, heightMeasureSpec)
        // 정사각형 유지: 너비/높이 중 작은 값으로 설정
        val size = minOf(w, h)
        setMeasuredDimension(size, size)
    }

    override fun onSizeChanged(w: Int, h: Int, oldw: Int, oldh: Int) {
        val halfStroke = progressPaint.strokeWidth / 2f
        ovalRect.set(halfStroke, halfStroke, w - halfStroke, h - halfStroke)
        textPaint.textSize = w * 0.22f
    }

    override fun onDraw(canvas: Canvas) {
        // 트랙(배경 원) 그리기
        canvas.drawArc(ovalRect, -90f, 360f, false, trackPaint)

        // 진행 아크 그리기 (-90도에서 시작 = 12시 방향)
        val sweepAngle = progress / 100f * 360f
        canvas.drawArc(ovalRect, -90f, sweepAngle, false, progressPaint)

        // 퍼센트 텍스트
        if (showText) {
            textPaint.color = progressPaint.color
            val x = width / 2f
            val y = height / 2f - (textPaint.descent() + textPaint.ascent()) / 2f
            canvas.drawText("${progress.toInt()}%", x, y, textPaint)
        }
    }
}
```

### 4.3 XML에서 사용

```xml
<com.example.app.CircleProgressView
    android:id="@+id/progressView"
    android:layout_width="120dp"
    android:layout_height="120dp"
    app:cpv_progress="72"
    app:cpv_progressColor="#6200EE"
    app:cpv_trackColor="#E0E0E0"
    app:cpv_strokeWidth="10dp"
    app:cpv_showText="true" />
```

코드에서 progress를 변경하면 `invalidate()`가 자동으로 호출되어 부드럽게 갱신됩니다:

```kotlin
progressView.progress = 85f
```

---

## 5. 실제 구현 예제 2: BarChartView (애니메이션 막대 차트)

데이터 배열을 받아 막대 차트를 그리고, `ValueAnimator`로 상승 애니메이션을 구현합니다.

```kotlin
class BarChartView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : View(context, attrs, defStyleAttr) {

    data class Bar(val label: String, val value: Float, val color: Int)

    private var bars: List<Bar> = emptyList()
    private var animationProgress = 0f

    // onDraw()에서 생성 금지 — 모두 멤버 변수로 선언
    private val barPaint = Paint(Paint.ANTI_ALIAS_FLAG)
    private val labelPaint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        textAlign = Paint.Align.CENTER
        color = Color.DKGRAY
    }

    private val animator = ValueAnimator.ofFloat(0f, 1f).apply {
        duration = 800L
        interpolator = DecelerateInterpolator()
        addUpdateListener { anim ->
            animationProgress = anim.animatedValue as Float
            invalidate()
        }
    }

    fun setData(data: List<Bar>) {
        bars = data
        animationProgress = 0f
        animator.start()
    }

    override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
        val minHeight = (resources.displayMetrics.density * 200).toInt()
        val h = resolveSize(minHeight, heightMeasureSpec)
        setMeasuredDimension(MeasureSpec.getSize(widthMeasureSpec), h)
    }

    override fun onDraw(canvas: Canvas) {
        if (bars.isEmpty()) return

        val density = resources.displayMetrics.density
        val padding = density * 16f
        val labelHeight = density * 24f
        val chartHeight = height - padding * 2 - labelHeight
        val barWidth = (width - padding * 2) / bars.size.toFloat()
        val gap = barWidth * 0.2f
        val maxValue = bars.maxOf { it.value }

        labelPaint.textSize = density * 11f

        bars.forEachIndexed { i, bar ->
            val left = padding + i * barWidth + gap / 2f
            val right = left + barWidth - gap
            val barHeight = (bar.value / maxValue) * chartHeight * animationProgress
            val top = padding + chartHeight - barHeight
            val bottom = padding + chartHeight

            barPaint.color = bar.color
            canvas.drawRoundRect(left, top, right, bottom, density * 6f, density * 6f, barPaint)

            canvas.drawText(
                bar.label,
                left + (right - left) / 2f,
                bottom + labelHeight * 0.85f,
                labelPaint
            )
        }
    }

    override fun onDetachedFromWindow() {
        super.onDetachedFromWindow()
        animator.cancel() // 뷰가 분리되면 Animator 반드시 취소
    }
}
```

사용 예:

```kotlin
barChartView.setData(
    listOf(
        BarChartView.Bar("Mon", 40f, Color.parseColor("#6200EE")),
        BarChartView.Bar("Tue", 75f, Color.parseColor("#03DAC5")),
        BarChartView.Bar("Wed", 55f, Color.parseColor("#FF6D00")),
        BarChartView.Bar("Thu", 90f, Color.parseColor("#6200EE")),
        BarChartView.Bar("Fri", 30f, Color.parseColor("#03DAC5")),
    )
)
```

---

## 6. 주의사항과 팁

### 6.1 onDraw()에서 객체 생성 절대 금지

`onDraw()`는 초당 60~120회 호출될 수 있습니다. 이 메서드 내부에서 `Paint()`, `RectF()`, `Path()` 등을 생성하면 **GC 압력**이 높아져 프레임 드롭이 발생합니다.

```kotlin
// 잘못된 예 — 매 프레임마다 객체 생성
override fun onDraw(canvas: Canvas) {
    val paint = Paint() // 절대 금지!
    canvas.drawCircle(cx, cy, radius, paint)
}

// 올바른 예 — 멤버 변수로 미리 초기화
private val paint = Paint(Paint.ANTI_ALIAS_FLAG)

override fun onDraw(canvas: Canvas) {
    canvas.drawCircle(cx, cy, radius, paint) // 재사용
}
```

### 6.2 TypedArray는 반드시 recycle()

`obtainStyledAttributes()`로 얻은 `TypedArray`는 내부적으로 오브젝트 풀을 사용합니다. `recycle()`을 호출하지 않으면 **메모리 누수**가 발생합니다. Kotlin의 `use {}` 확장을 활용하면 편리합니다:

```kotlin
context.obtainStyledAttributes(attrs, R.styleable.MyView).use { ta ->
    // TypedArray 사용
} // use {} 블록이 끝나면 자동으로 recycle() 호출
```

### 6.3 invalidate() vs requestLayout()

- **`invalidate()`**: 뷰의 **내용**만 바뀌었을 때 (크기는 그대로). 재드로우(Draw)만 트리거.
- **`requestLayout()`**: 뷰의 **크기나 위치**가 바뀌었을 때. Measure→Layout→Draw 전체를 트리거.

불필요하게 `requestLayout()`을 호출하면 전체 View 트리를 재측정하므로 성능에 악영향을 미칩니다.

### 6.4 Hardware Acceleration 주의

Android 3.0부터 기본으로 하드웨어 가속이 활성화되어 있습니다. 일부 Canvas 연산(`drawBitmapMesh`, 특정 `PorterDuff` 모드 등)은 하드웨어 가속에서 지원되지 않을 수 있습니다. 이 경우 `setLayerType(LAYER_TYPE_SOFTWARE, null)`로 소프트웨어 렌더링으로 전환하세요. 단, 소프트웨어 렌더링은 성능 비용이 있으므로 꼭 필요한 경우에만 사용합니다.

### 6.5 접근성(Accessibility) 고려

Custom View는 기본 접근성 지원이 없습니다. `android:contentDescription`을 설정하고, 상태가 변경되면 `sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_SELECTED)`를 호출해 스크린 리더가 변경을 인식할 수 있도록 해야 합니다.

### 6.6 Jetpack Compose와의 공존

기존 View 시스템의 Custom View를 Jetpack Compose에서 사용하려면 `AndroidView` 컴포저블을 활용하세요:

```kotlin
@Composable
fun CircleProgressComposable(progress: Float) {
    AndroidView(
        factory = { context -> CircleProgressView(context) },
        update = { view -> view.progress = progress }
    )
}
```

---

## 마치며

Android Custom View는 단순히 "그림을 그리는 것"이 아니라, Measure→Layout→Draw의 원리를 깊이 이해하고 올바르게 구현하는 것이 핵심입니다. `onDraw()`에서의 객체 생성 금지, `TypedArray` 누수 방지, 적절한 `invalidate()` vs `requestLayout()` 선택, Animator의 생명주기 관리 등 주의사항을 지키면 성능에도 안전한 고품질 Custom View를 만들 수 있습니다. 앞서 소개한 `CircleProgressView`와 `BarChartView`를 기반으로 확장해 나가며, 프로젝트만의 독창적인 UI 컴포넌트를 직접 구현해 보세요.

---

## 참고 자료

- [Create a custom drawing \| Android Developers](https://developer.android.com/develop/ui/views/layout/custom-views/custom-drawing)
- [Create a view class \| Android Developers](https://developer.android.com/develop/ui/views/layout/custom-views/create-view)
- [Canvas API Reference \| Android Developers](https://developer.android.com/reference/android/graphics/Canvas)
- [Create custom view components \| Android Developers](https://developer.android.com/develop/ui/views/layout/custom-views/custom-components)
