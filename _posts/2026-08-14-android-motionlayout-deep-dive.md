---
layout: post
title: "Android MotionLayout 심화: MotionScene·KeyFrame·OnSwipe로 선언형 복잡 애니메이션 완전 정복"
date: 2026-08-14
categories: [android, flutter]
tags: [android, motionlayout, constraintlayout, animation, keyframe, onswipe, motionscene, kotlin]
---

## MotionLayout이란?

`MotionLayout`은 `ConstraintLayout`의 서브클래스로, 안드로이드에서 복잡한 뷰 애니메이션을 **선언적(declarative)** 방식으로 기술할 수 있게 해주는 레이아웃 컴포넌트입니다. 2018년 Google I/O에서 처음 발표된 이후 `androidx.constraintlayout:constraintlayout` 2.0 이상에서 안정 버전으로 제공되고 있으며, 2.2.x 계열에서 Compose 연동까지 지원 범위가 확장되었습니다.

핵심 아이디어는 단순합니다. 뷰가 **시작 상태(start ConstraintSet)**에서 **끝 상태(end ConstraintSet)**로 어떻게 변해야 하는지를 별도의 XML 파일(`MotionScene`)에 기술하면, MotionLayout이 두 상태 사이를 자동으로 부드럽게 보간(interpolation)합니다. 단순한 위치 이동부터 크기 변환, 회전, 알파값 변경, 색상 전환, 커스텀 속성 변환까지 모두 지원하며 제스처 인식과 진행도(progress) 기반 제어도 내장합니다.

---

## 왜 MotionLayout인가?

기존 안드로이드 애니메이션 방식은 목적에 따라 세 가지로 나뉩니다.

| 방식 | 장점 | 단점 |
|---|---|---|
| `ObjectAnimator` | 단일 속성 애니메이션에 적합 | 여러 뷰·속성 동시 처리 시 코드 폭발 |
| `TransitionManager` | 씬 전환 지원 | 제스처 연동·progress 제어 어려움 |
| `AnimatorSet` | 복수 애니메이션 조합 가능 | 동기화·타이밍 제어 번거로움 |

MotionLayout은 이 세 방식의 한계를 동시에 극복합니다.

- **제스처 연동**: `OnSwipe` 핸들러로 드래그 제스처에 반응하는 애니메이션을 Kotlin 코드 없이 구현
- **진행도 제어**: `0.0f ~ 1.0f` 사이의 progress 값으로 스크롤·슬라이더와 직접 연동
- **KeyFrame**: 시작과 끝 사이 임의 지점에서 뷰 속성을 재정의해 포물선 경로, 오버슈팅, 바운스 효과 구현
- **선언적**: 로직과 UI 상태를 분리하여 유지보수성과 가독성 향상
- **Motion Editor**: Android Studio의 시각적 편집 도구로 XML 없이 애니메이션 프로토타입 제작

---

## 핵심 구조: MotionScene과 ConstraintSet

MotionLayout 애니메이션의 모든 명세는 `res/xml/` 디렉토리의 **MotionScene** XML 파일에 담깁니다. 레이아웃 XML과 애니메이션 정의를 물리적으로 분리함으로써 관심사 분리(SoC)를 실현합니다.

| 구성요소 | 역할 |
|---|---|
| `ConstraintSet` | 특정 상태에서 각 뷰의 제약 조건 및 속성 집합 |
| `Transition` | 두 ConstraintSet 간 전환 방식(duration, interpolator, 핸들러) |
| `KeyFrameSet` | 전환 도중 중간 지점의 임시 상태 정의 |
| `OnSwipe` | 드래그 제스처 트리거 |
| `OnClick` | 클릭 트리거 |

---

## 코드 예제 1: 기본 MotionLayout — 스와이프로 카드 슬라이드

화면 왼쪽 상단의 프로필 카드를 드래그하면 오른쪽 하단으로 이동하면서 크기가 축소되고 알파값이 낮아지는 UI를 구현합니다.

**레이아웃 파일 (res/layout/activity_main.xml)**

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.motion.widget.MotionLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/motionLayout"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    app:layoutDescription="@xml/scene_profile_card">

    <androidx.cardview.widget.CardView
        android:id="@+id/profileCard"
        android:layout_width="300dp"
        android:layout_height="200dp"
        app:cardCornerRadius="16dp"
        app:cardElevation="8dp" />

    <Button
        android:id="@+id/toggleButton"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Toggle" />

</androidx.constraintlayout.motion.widget.MotionLayout>
```

**MotionScene 파일 (res/xml/scene_profile_card.xml)**

```xml
<?xml version="1.0" encoding="utf-8"?>
<MotionScene xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto">

    <ConstraintSet android:id="@+id/start">
        <Constraint
            android:id="@id/profileCard"
            android:layout_width="300dp"
            android:layout_height="200dp"
            android:alpha="1.0"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintTop_toTopOf="parent"
            android:layout_marginStart="16dp"
            android:layout_marginTop="16dp" />
    </ConstraintSet>

    <ConstraintSet android:id="@+id/end">
        <Constraint
            android:id="@id/profileCard"
            android:layout_width="100dp"
            android:layout_height="70dp"
            android:alpha="0.4"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintBottom_toBottomOf="parent"
            android:layout_marginEnd="16dp"
            android:layout_marginBottom="16dp" />
    </ConstraintSet>

    <Transition
        app:constraintSetStart="@id/start"
        app:constraintSetEnd="@id/end"
        app:duration="400"
        app:motionInterpolator="easeInOut">

        <!-- 드래그로 직접 조작 가능하게 설정 -->
        <OnSwipe
            app:touchAnchorId="@id/profileCard"
            app:touchAnchorSide="bottom"
            app:dragDirection="dragDown"
            app:onTouchUp="autoComplete" />
    </Transition>

</MotionScene>
```

**Kotlin 코드 — 프로그래밍 방식 전환 및 상태 감지**

```kotlin
class MainActivity : AppCompatActivity() {

    private lateinit var motionLayout: MotionLayout
    private var isExpanded = true

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        motionLayout = findViewById(R.id.motionLayout)

        // TransitionListener로 애니메이션 이벤트 감지
        motionLayout.addTransitionListener(object : MotionLayout.TransitionListener {
            override fun onTransitionStarted(ml: MotionLayout, startId: Int, endId: Int) {
                // 전환 시작 시 호출
            }

            override fun onTransitionChange(
                ml: MotionLayout, startId: Int, endId: Int, progress: Float
            ) {
                // progress 0.0f → 1.0f: 진행도에 따른 외부 UI 업데이트 가능
                updateExternalUI(progress)
            }

            override fun onTransitionCompleted(ml: MotionLayout, currentId: Int) {
                isExpanded = (currentId == R.id.start)
            }

            override fun onTransitionTrigger(
                ml: MotionLayout, triggerId: Int, positive: Boolean, progress: Float
            ) {}
        })

        // 버튼으로 프로그래밍 방식 전환
        findViewById<Button>(R.id.toggleButton).setOnClickListener {
            if (isExpanded) {
                motionLayout.transitionToEnd()
            } else {
                motionLayout.transitionToStart()
            }
        }
    }

    private fun updateExternalUI(progress: Float) {
        // 예: 외부 ProgressBar와 동기화
        // progressBar.progress = (progress * 100).toInt()
    }
}
```

---

## 코드 예제 2: KeyFrameSet으로 포물선 경로·바운스 구현 + SeekBar 연동

`KeyFrameSet`을 사용하면 애니메이션의 시작(0%)과 끝(100%) 사이 임의의 지점에서 뷰 속성을 재정의할 수 있습니다. 이를 통해 직선이 아닌 곡선 궤적, 회전, 스케일 바운스 등을 선언만으로 구현할 수 있습니다.

**MotionScene with KeyFrameSet (res/xml/scene_bounce_ball.xml)**

```xml
<?xml version="1.0" encoding="utf-8"?>
<MotionScene xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto">

    <ConstraintSet android:id="@+id/start">
        <Constraint android:id="@id/ball"
            android:layout_width="60dp"
            android:layout_height="60dp"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintTop_toTopOf="parent"
            android:layout_marginStart="32dp"
            android:layout_marginTop="200dp" />
    </ConstraintSet>

    <ConstraintSet android:id="@+id/end">
        <Constraint android:id="@id/ball"
            android:layout_width="60dp"
            android:layout_height="60dp"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintTop_toTopOf="parent"
            android:layout_marginEnd="32dp"
            android:layout_marginTop="200dp" />
    </ConstraintSet>

    <Transition
        app:constraintSetStart="@id/start"
        app:constraintSetEnd="@id/end"
        app:duration="1200">

        <KeyFrameSet>
            <!-- 50% 지점에서 Y 오프셋으로 포물선 궤적 생성 -->
            <KeyPosition
                app:motionTarget="@id/ball"
                app:framePosition="50"
                app:keyPositionType="parentRelative"
                app:percentY="0.25" />

            <!-- 50% 지점에서 스케일 업 (공기 저항 효과) -->
            <KeyAttribute
                app:motionTarget="@id/ball"
                app:framePosition="50"
                android:scaleX="1.4"
                android:scaleY="1.4" />

            <!-- 75% 지점에서 절반 회전 -->
            <KeyAttribute
                app:motionTarget="@id/ball"
                app:framePosition="75"
                android:rotation="180" />

            <!-- 100%: 원래 크기로 복귀 + 360도 회전 완료 -->
            <KeyAttribute
                app:motionTarget="@id/ball"
                app:framePosition="100"
                android:rotation="360"
                android:scaleX="1.0"
                android:scaleY="1.0" />
        </KeyFrameSet>

        <OnClick
            app:targetId="@id/ball"
            app:clickAction="toggle" />
    </Transition>

</MotionScene>
```

**SeekBar로 애니메이션 progress 수동 제어 (Kotlin)**

```kotlin
class BallAnimationActivity : AppCompatActivity() {

    private lateinit var motionLayout: MotionLayout
    private lateinit var seekBar: SeekBar

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_ball_animation)

        motionLayout = findViewById(R.id.motionLayout)
        seekBar = findViewById(R.id.seekBar)

        // SeekBar로 애니메이션 progress를 수동 스크럽
        seekBar.setOnSeekBarChangeListener(object : SeekBar.OnSeekBarChangeListener {
            override fun onProgressChanged(seekBar: SeekBar, progress: Int, fromUser: Boolean) {
                if (fromUser) {
                    // 0~100 → 0.0f~1.0f 매핑
                    motionLayout.progress = progress / 100f
                }
            }

            override fun onStartTrackingTouch(seekBar: SeekBar) {
                // 드래그 시작 시 자동 애니메이션 중지
                motionLayout.stopAnimation()
            }

            override fun onStopTrackingTouch(seekBar: SeekBar) {}
        })

        // MotionLayout ↔ SeekBar 양방향 동기화
        motionLayout.addTransitionListener(object : MotionLayout.TransitionListener {
            override fun onTransitionChange(
                ml: MotionLayout, startId: Int, endId: Int, progress: Float
            ) {
                // 터치 중이 아닐 때만 SeekBar 업데이트 (무한 루프 방지)
                if (!seekBar.isPressed) {
                    seekBar.progress = (progress * 100).toInt()
                }
            }

            override fun onTransitionStarted(ml: MotionLayout, startId: Int, endId: Int) {}
            override fun onTransitionCompleted(ml: MotionLayout, currentId: Int) {}
            override fun onTransitionTrigger(
                ml: MotionLayout, triggerId: Int, positive: Boolean, progress: Float
            ) {}
        })

        // 자동 재생 버튼
        findViewById<Button>(R.id.playButton).setOnClickListener {
            motionLayout.transitionToEnd()
        }

        // 처음으로 되감기
        findViewById<Button>(R.id.resetButton).setOnClickListener {
            motionLayout.transitionToStart()
        }
    }
}
```

---

## 주의사항 및 실전 팁

### 1. 직접 자식 뷰(Direct Children)만 제어된다

MotionLayout은 직속 자식 뷰만 `ConstraintSet` 타겟으로 지정할 수 있습니다. 중첩 뷰 그룹 내부의 뷰를 제어하려면 해당 뷰를 MotionLayout 직속 자식으로 끌어올리거나, 내부에 별도의 MotionLayout을 중첩해야 합니다.

### 2. `CustomAttribute`로 기본 제공되지 않는 속성 애니메이션

`alpha`, `rotation`, `scaleX/Y` 등 기본 속성 외에 CardView의 `cardElevation`, 텍스트 색상 등 커스텀 속성은 `<CustomAttribute>`를 사용합니다.

```xml
<Constraint android:id="@id/profileCard">
    <CustomAttribute
        app:attributeName="cardElevation"
        app:customFloatValue="16" />
    <CustomAttribute
        app:attributeName="cardBackgroundColor"
        app:customColorValue="#FF6200EE" />
</Constraint>
```

단, 커스텀 속성 대상 클래스에 반드시 `public` getter/setter(`getXxx()` / `setXxx()`)가 존재해야 합니다.

### 3. `transitionToEnd()` vs `setProgress()` 선택 기준

| 상황 | 권장 방식 |
|---|---|
| 버튼 클릭으로 단순 전환 | `transitionToEnd()` / `transitionToStart()` |
| 스크롤·슬라이더와 1:1 연동 | `setProgress(progress)` 직접 주입 |
| 특정 지점에서 멈추기 | `setProgress(0.5f)` 후 `stopAnimation()` |

### 4. `onTouchUp` 속성으로 손가락을 떼는 동작 제어

`OnSwipe`의 `app:onTouchUp` 속성은 사용자가 손가락을 뗀 후 동작을 결정합니다.

- `autoComplete`: 현재 진행 방향에 따라 자동으로 start 또는 end로 완주
- `stop`: 현재 위치에서 정지
- `decelerate`: 감속하며 정지
- `decelerateAndComplete`: 감속 후 가장 가까운 끝점으로 완주

### 5. Jetpack Compose 프로젝트에서는 `MotionLayout` composable 사용

`constraintlayout-compose` 라이브러리를 추가하면 Compose에서도 MotionLayout을 사용할 수 있습니다.

```kotlin
// build.gradle.kts
implementation("androidx.constraintlayout:constraintlayout-compose:1.1.1")
```

```kotlin
@Composable
fun AnimatedCard() {
    var animateToEnd by remember { mutableStateOf(false) }
    val progress by animateFloatAsState(
        targetValue = if (animateToEnd) 1f else 0f,
        animationSpec = tween(600)
    )

    MotionLayout(
        motionScene = MotionScene(content = motionSceneContent),
        progress = progress,
        modifier = Modifier.fillMaxSize()
    ) {
        Box(
            modifier = Modifier
                .layoutId("box")
                .background(Color.Blue)
                .clickable { animateToEnd = !animateToEnd }
        )
    }
}
```

Compose 버전은 `layoutId` Modifier로 MotionScene의 `ConstraintSet`과 뷰를 연결하며, `animateFloatAsState`로 progress를 자연스럽게 구동합니다.

### 6. 디버깅: `app:motionDebug` 속성

개발 중 경로와 진행도를 시각화하려면 레이아웃의 MotionLayout 태그에 다음을 추가하세요.

```xml
app:motionDebug="SHOW_ALL"
```

`SHOW_PATH`는 이동 경로선만, `SHOW_PROGRESS`는 진행도 수치만 표시합니다. 릴리스 빌드 전 반드시 제거하세요.

---

## 마치며

MotionLayout은 단순 View 애니메이션의 한계를 뛰어넘어, 여러 뷰의 복잡한 상태 전환을 완전히 선언적으로 표현할 수 있게 해줍니다. `KeyFrameSet`으로 자유로운 중간 상태를 정의하고, `OnSwipe`로 제스처 인식을 연결하며, Kotlin 코드에서 `progress`를 직접 주입함으로써 스크롤·슬라이더 연동까지 막힘없이 구현할 수 있습니다. 특히 Compose `MotionLayout`과 연계하면 선언형 UI 철학을 유지하면서도 복잡한 애니메이션을 손쉽게 다룰 수 있습니다.

## 참고 자료
- [Manage motion and widget animation with MotionLayout — Android Developers](https://developer.android.com/develop/ui/views/animations/motionlayout)
- [MotionLayout API Reference — AndroidX](https://developer.android.com/reference/androidx/constraintlayout/motion/widget/MotionLayout)
- [ConstraintLayout & MotionLayout Samples — android/platform-samples](https://github.com/android/platform-samples/tree/main/samples/user-interface/constraintlayout)
