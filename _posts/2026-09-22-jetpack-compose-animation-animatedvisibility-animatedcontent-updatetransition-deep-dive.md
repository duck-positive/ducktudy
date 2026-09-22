---
layout: post
title: "Jetpack Compose 애니메이션 심화: AnimatedVisibility·AnimatedContent·updateTransition 완전 정복"
date: 2026-09-22
categories: [android, flutter]
tags: [android, jetpack-compose, animation, kotlin, ui, animatedvisibility, animatedcontent, updatetransition]
---

Jetpack Compose는 선언형 UI 패러다임에 맞는 강력한 애니메이션 API를 제공한다. 기존 View 시스템의 명령형 방식과 달리, Compose 애니메이션은 **상태 변화에 반응하여 자동으로 보간(interpolation)된 값을 생성**한다. 이 글에서는 실무에서 가장 많이 쓰이는 `AnimatedVisibility`, `AnimatedContent`, `updateTransition`을 깊이 있게 다루고, `animate*AsState`부터 `Animatable` 저수준 API까지 계층 구조를 완전히 정복한다.

## Compose 애니메이션의 계층 구조

Compose 애니메이션 API는 추상화 수준에 따라 세 계층으로 나뉜다.

```
고수준  │ AnimatedVisibility / AnimatedContent / animateContentSize
        │ animate*AsState / updateTransition / rememberInfiniteTransition
저수준  │ Animatable / Animation
```

**고수준 API**는 선언적으로 사용하기 쉽고 대부분의 시나리오를 커버한다. **저수준 API**는 복잡한 시퀀셜 애니메이션이나 제스처 기반 드래그 처리 등 세밀한 제어가 필요할 때 사용한다.

## 왜 Compose 애니메이션인가?

View 시스템에서 애니메이션을 구현하려면 `ObjectAnimator`, `ValueAnimator`, `AnimatorSet`, `MotionLayout` 등 다양한 API를 조합해야 했다. 상태 전환마다 코드가 폭발적으로 증가하고, 취소(cancel)와 재시작(restart) 처리를 수동으로 관리해야 하는 부담이 컸다.

Compose는 이 문제를 근본적으로 해결한다.

- **자동 취소·재시작**: 상태가 바뀌면 진행 중인 애니메이션을 현재 값에서 부드럽게 새 목표 값으로 전환한다.
- **선언적 명세**: "이 상태일 때 이 값이어야 한다"고 선언하면 Compose가 보간을 처리한다.
- **Physics-based 기본값**: 기본 `animationSpec`이 `spring`(스프링 물리 기반)이라 자연스러운 감속·탄성이 내장되어 있다.
- **Skippable 최적화**: 애니메이션 값이 변해도 그 값을 사용하는 컴포저블만 재구성(recompose)된다.

## 실전 예제 1: AnimatedVisibility와 AnimatedContent

### AnimatedVisibility — 표시/숨김 전환 애니메이션

`AnimatedVisibility`는 자식 컴포저블의 등장·퇴장에 자동으로 애니메이션을 적용한다. `enter`와 `exit` 파라미터에 `EnterTransition`과 `ExitTransition`을 조합하여 원하는 효과를 만든다.

```kotlin
@Composable
fun NotificationBanner(message: String, visible: Boolean) {
    AnimatedVisibility(
        visible = visible,
        enter = slideInVertically(
            initialOffsetY = { -it }, // 위에서 아래로 슬라이드
            animationSpec = spring(
                dampingRatio = Spring.DampingRatioMediumBouncy,
                stiffness = Spring.StiffnessMediumLow
            )
        ) + fadeIn(animationSpec = tween(200)),
        exit = slideOutVertically(
            targetOffsetY = { -it }
        ) + fadeOut(animationSpec = tween(150))
    ) {
        Surface(
            modifier = Modifier
                .fillMaxWidth()
                .padding(horizontal = 16.dp, vertical = 8.dp),
            color = MaterialTheme.colorScheme.primaryContainer,
            shape = RoundedCornerShape(12.dp),
            shadowElevation = 4.dp
        ) {
            Text(
                text = message,
                modifier = Modifier.padding(16.dp),
                style = MaterialTheme.typography.bodyMedium,
                color = MaterialTheme.colorScheme.onPrimaryContainer
            )
        }
    }
}
```

`enter`와 `exit`에 `+` 연산자로 여러 효과를 합성할 수 있다. `slideInVertically + fadeIn` 조합은 배너, 스낵바, 툴팁 등 다양한 UI 패턴에 즉시 적용할 수 있다.

`AnimatedVisibility` 내부에서는 `transition.animateFloat` 등을 직접 사용할 수도 있다. 자식 컴포저블이 `AnimatedVisibilityScope`를 수신자(receiver)로 가지므로, `this.transition`에 접근해 맞춤 애니메이션을 추가할 수 있다.

### AnimatedContent — 콘텐츠 전환 애니메이션

`AnimatedContent`는 **상태 값에 따라 다른 컴포저블을 보여줄 때** 전환 효과를 자동으로 처리한다. `targetState`가 바뀌면 이전 콘텐츠는 `exit` 효과로 사라지고 새 콘텐츠는 `enter` 효과로 등장한다.

```kotlin
sealed class LoadState {
    object Loading : LoadState()
    data class Success(val data: List<String>) : LoadState()
    data class Error(val message: String) : LoadState()
}

@Composable
fun LoadingContent(state: LoadState) {
    AnimatedContent(
        targetState = state,
        transitionSpec = {
            // 상태 전환 방향에 따라 슬라이드 방향 결정
            val enter = when (targetState) {
                is LoadState.Success -> slideInHorizontally { it } + fadeIn()
                is LoadState.Error   -> slideInHorizontally { it } + fadeIn()
                LoadState.Loading    -> fadeIn(tween(200))
            }
            val exit = fadeOut(animationSpec = tween(150))
            enter togetherWith exit using SizeTransform(clip = false)
        },
        label = "LoadStateTransition"
    ) { currentState ->
        when (currentState) {
            LoadState.Loading -> {
                Box(Modifier.fillMaxWidth(), contentAlignment = Alignment.Center) {
                    CircularProgressIndicator()
                }
            }
            is LoadState.Success -> {
                LazyColumn {
                    items(currentState.data) { item ->
                        Text(
                            text = item,
                            modifier = Modifier.padding(16.dp)
                        )
                    }
                }
            }
            is LoadState.Error -> {
                Column(
                    modifier = Modifier.fillMaxWidth().padding(24.dp),
                    horizontalAlignment = Alignment.CenterHorizontally
                ) {
                    Icon(Icons.Default.Error, contentDescription = null, tint = MaterialTheme.colorScheme.error)
                    Spacer(Modifier.height(8.dp))
                    Text(currentState.message, color = MaterialTheme.colorScheme.error)
                }
            }
        }
    }
}
```

`transitionSpec` 람다 안의 `this`는 `AnimatedContentTransitionScope`이며, `initialState`와 `targetState`를 통해 "어디서 어디로" 전환하는지 알 수 있다. 이를 이용해 방향에 따른 슬라이드 애니메이션을 구현하는 것이 핵심 패턴이다.

`SizeTransform(clip = false)`를 사용하면 컨테이너 크기가 애니메이션되는 동안 자식 콘텐츠가 경계 밖으로 노출될 수 있어 더 자연스러운 팽창/수축 효과를 만들 수 있다.

## 실전 예제 2: updateTransition으로 다중 속성 동시 애니메이션

`animate*AsState`는 단일 속성 애니메이션에 편리하지만, 여러 속성이 **같은 상태 전환에 동기화**되어야 할 때는 `updateTransition`을 사용해야 한다. 하나의 `Transition` 객체가 여러 파생 애니메이션을 동기화하므로 타이밍 불일치 없이 일관성 있는 전환이 보장된다.

```kotlin
enum class CardState { Collapsed, Expanded }

@Composable
fun AnimatedCard(
    content: @Composable () -> Unit,
    detailContent: @Composable () -> Unit
) {
    var cardState by remember { mutableStateOf(CardState.Collapsed) }

    val transition = updateTransition(
        targetState = cardState,
        label = "CardStateTransition"
    )

    // 여러 속성을 하나의 Transition에서 동시에 애니메이션
    val cornerRadius by transition.animateDp(
        transitionSpec = { spring(stiffness = Spring.StiffnessMedium) },
        label = "cornerRadius"
    ) { state ->
        if (state == CardState.Collapsed) 16.dp else 0.dp
    }

    val elevation by transition.animateDp(
        transitionSpec = { spring(stiffness = Spring.StiffnessMedium) },
        label = "elevation"
    ) { state ->
        if (state == CardState.Collapsed) 4.dp else 8.dp
    }

    val contentAlpha by transition.animateFloat(
        transitionSpec = { tween(durationMillis = 200) },
        label = "contentAlpha"
    ) { state ->
        if (state == CardState.Expanded) 1f else 0f
    }

    val detailHeight by transition.animateDp(
        transitionSpec = { spring(dampingRatio = Spring.DampingRatioLowBouncy) },
        label = "detailHeight"
    ) { state ->
        if (state == CardState.Expanded) 200.dp else 0.dp
    }

    Surface(
        modifier = Modifier
            .fillMaxWidth()
            .padding(16.dp)
            .clickable {
                cardState = if (cardState == CardState.Collapsed)
                    CardState.Expanded else CardState.Collapsed
            },
        shape = RoundedCornerShape(cornerRadius),
        shadowElevation = elevation,
        color = MaterialTheme.colorScheme.surface
    ) {
        Column {
            // 기본 콘텐츠는 항상 표시
            Box(modifier = Modifier.padding(16.dp)) {
                content()
            }

            // 상세 콘텐츠는 애니메이션으로 펼쳐짐
            Box(
                modifier = Modifier
                    .fillMaxWidth()
                    .height(detailHeight)
                    .graphicsLayer { alpha = contentAlpha }
                    .padding(horizontal = 16.dp)
            ) {
                if (transition.currentState == CardState.Expanded ||
                    transition.targetState == CardState.Expanded) {
                    detailContent()
                }
            }
        }
    }
}
```

핵심은 `transition.currentState`와 `transition.targetState`를 함께 확인하는 패턴이다. 애니메이션이 진행 중일 때도(`currentState != targetState`) 상세 콘텐츠를 렌더링해야 `detailHeight`가 0이 아닌 중간값일 때 내용이 보인다.

### InfiniteTransition — 무한 반복 로딩 인디케이터

```kotlin
@Composable
fun PulsingDot(color: Color = MaterialTheme.colorScheme.primary) {
    val infiniteTransition = rememberInfiniteTransition(label = "pulse")

    val scale by infiniteTransition.animateFloat(
        initialValue = 0.85f,
        targetValue = 1.15f,
        animationSpec = infiniteRepeatable(
            animation = tween(700, easing = FastOutSlowInEasing),
            repeatMode = RepeatMode.Reverse
        ),
        label = "scale"
    )

    val alpha by infiniteTransition.animateFloat(
        initialValue = 0.5f,
        targetValue = 1f,
        animationSpec = infiniteRepeatable(
            animation = tween(700, easing = LinearEasing),
            repeatMode = RepeatMode.Reverse
        ),
        label = "alpha"
    )

    Box(
        modifier = Modifier
            .size(24.dp)
            .graphicsLayer {
                scaleX = scale
                scaleY = scale
                this.alpha = alpha
            }
            .background(color, CircleShape)
    )
}
```

`graphicsLayer`를 사용하면 `scaleX`, `scaleY`, `alpha`, `translationX` 등을 **재구성(recomposition) 없이** 드로우 단계에서만 변경할 수 있어 성능이 훨씬 좋다.

## 애니메이션 스펙(AnimationSpec) 선택 가이드

| 상황 | 권장 스펙 |
|------|-----------|
| 대부분의 UI 전환 (기본값) | `spring(dampingRatio = Spring.DampingRatioNoBouncy)` |
| 카드 확장, 패널 슬라이드 | `spring(dampingRatio = Spring.DampingRatioMediumBouncy)` |
| 정확한 지속 시간이 필요할 때 | `tween(durationMillis = 300, easing = FastOutSlowInEasing)` |
| 특정 시점에 특정 값을 지정 | `keyframes { durationMillis = 500; 0.8f at 100 }` |
| 무한 반복 | `infiniteRepeatable(animation = tween(...))` |
| 즉시 전환 (애니메이션 없음) | `snap()` |

Material Design 가이드라인은 입장(enter) 애니메이션에 `FastOutSlowInEasing`, 퇴장(exit)에 `FastOutLinearInEasing`을 권장한다. `spring`은 물리 기반이라 자연스럽지만 지속 시간을 직접 제어할 수 없다는 점에 주의하자.

## 성능 최적화 팁

### 1. graphicsLayer 람다 활용으로 재구성 방지

```kotlin
// 비효율: alpha 변경마다 재구성 발생
Box(modifier = Modifier.alpha(animatedAlpha))

// 효율적: 드로우 단계에서만 처리
Box(modifier = Modifier.graphicsLayer { alpha = animatedAlpha })
```

`graphicsLayer { }` 블록은 컴포저블의 재구성 없이 GPU 레이어 속성을 변경한다. 지속적으로 변하는 애니메이션 값에는 항상 이 방식을 우선한다.

### 2. 조건부 콘텐츠는 AnimatedVisibility로 감싸기

`if (visible) { HeavyComposable() }` 패턴은 `visible`이 `false`로 바뀌는 순간 즉시 컴포저블을 제거한다. 퇴장 애니메이션이 필요하다면 반드시 `AnimatedVisibility`로 감싸야 한다. `AnimatedVisibility`는 퇴장 애니메이션이 완료된 후에야 자식을 컴포지션에서 제거한다.

### 3. label 파라미터 반드시 지정

`animate*AsState`, `updateTransition` 등 모든 애니메이션 API의 `label` 파라미터를 지정하면 Android Studio의 **Animation Preview** 도구에서 각 애니메이션을 이름으로 구분하여 디버깅할 수 있다.

### 4. 레이아웃 애니메이션 vs 드로우 애니메이션 구분

크기·위치 변화(`width`, `height`, `offset`)는 레이아웃 단계에 영향을 주어 상대적으로 비용이 크다. 가능하면 `graphicsLayer`의 `scaleX`, `scaleY`, `translationX`로 드로우 단계에서 처리하고, 실제 레이아웃 변경이 필요할 때만 `animateDpAsState`나 `animateContentSize()`를 사용한다.

## 주의사항

- **`AnimatedContent`에서 key 다루기**: `targetState`가 같은 타입이지만 서로 다른 콘텐츠를 보여줄 경우, Compose는 두 상태를 같다고 판단해 애니메이션이 실행되지 않을 수 있다. 이럴 때는 `key(targetState)` 를 사용하거나 `Transition.segment`를 활용해 강제로 구분한다.

- **`rememberInfiniteTransition`은 컴포저블이 숨겨져도 계속 실행**된다. 배터리 절약을 위해 화면 밖에 나갔을 때 중단해야 한다면, `LocalLifecycleOwner`와 `DisposableEffect`를 조합해 생명주기에 맞춰 일시 정지하는 로직을 추가하자.

- **`spring` 스펙은 duration 보장이 없다**. 애니메이션 테스팅에서 `spring`은 속도가 임계값 이하가 될 때까지 실행되므로, 테스트 시 `mainClock.advanceTimeByFrame()` 대신 `mainClock.advanceTimeUntilIdle()`을 사용해야 한다.

- **Android Studio의 Animation Preview 활용**: `@Preview`가 적용된 컴포저블에서 Compose 애니메이션 API를 사용하면 IDE의 Animation Preview 창에서 타임라인 기반으로 인터랙티브하게 미리볼 수 있다. `label` 파라미터가 여기서 이름으로 표시된다.

## 참고 자료
- [Compose 애니메이션 소개](https://developer.android.com/develop/ui/compose/animation/introduction)
- [값 기반 애니메이션 (animate*AsState, updateTransition, Animatable)](https://developer.android.com/develop/ui/compose/animation/value-based)
- [Compose 애니메이션 빠른 가이드](https://developer.android.com/develop/ui/compose/animation/quick-guide)
