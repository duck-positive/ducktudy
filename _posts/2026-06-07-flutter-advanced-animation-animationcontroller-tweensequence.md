---
layout: post
title: "Flutter 고급 애니메이션 — AnimationController와 TweenSequence 완전 정복"
date: 2026-06-07
categories: [flutter]
tags: [flutter, animation, animationcontroller, tweensequence, staggered, dart]
---

## Flutter 애니메이션의 핵심 구조

Flutter의 애니메이션 시스템은 크게 세 가지 레이어로 구성됩니다. **Animation 객체**, **AnimationController**, 그리고 **Tween**이 바로 그것입니다. 이 세 가지가 유기적으로 동작하면서 60fps 혹은 120fps의 부드러운 화면 전환을 만들어냅니다.

`Animation<T>` 추상 클래스는 현재 값과 상태(진행·완료·역방향 등)를 가집니다. `AnimationController`는 이 Animation의 구체적인 구현체로, 하드웨어가 새 프레임을 렌더링할 준비가 될 때마다 0.0 에서 1.0 사이의 값을 생성합니다. `Tween`은 이 0.0~1.0 범위를 원하는 타입과 범위로 매핑해주는 역할을 합니다.

### AnimationController 핵심 파라미터

```dart
AnimationController({
  double? value,              // 초기값 (기본 0.0)
  Duration? duration,         // 정방향 재생 시간
  Duration? reverseDuration,  // 역방향 재생 시간
  required TickerProvider vsync, // 화면 갱신 틱 공급자
  double lowerBound = 0.0,
  double upperBound = 1.0,
})
```

`vsync`는 `TickerProviderStateMixin` 또는 `SingleTickerProviderStateMixin`을 통해 제공됩니다. vsync가 없으면 위젯이 화면에 없을 때도 CPU 자원을 낭비하게 됩니다.

---

## 왜 TweenSequence가 필요한가?

단일 `Tween`은 A에서 B로의 직선적(혹은 커브를 적용한) 변환만 표현합니다. 하지만 실제 UI에서는 이런 복합 시나리오가 자주 필요합니다.

- **0~40%**: 0에서 100으로 빠르게 이동 (`easeIn`)
- **40~60%**: 100에서 잠깐 유지 (정지)
- **60~100%**: 100에서 80으로 되돌아오며 탄성 효과 (`elasticOut`)

이런 **단계별 복합 애니메이션**을 하나의 AnimationController로 처리하려면 `TweenSequence`가 필수입니다. 직접 `Interval`을 계산해서 여러 `CurvedAnimation`을 합성하는 방법도 있지만, 코드 복잡도가 크게 올라갑니다.

`TweenSequence`는 각 단계에 **weight**(가중치)를 부여해서 전체 duration에서 차지하는 비율을 지정합니다. weight의 합이 전체 100%입니다.

---

## 실제 구현 예제

### 예제 1: TweenSequence로 탄성 바운스 낙하 효과 구현

카드가 위에서 내려오면서 바닥에 닿을 때 살짝 튀어오르는 효과입니다.

```dart
import 'package:flutter/material.dart';

class BounceDropCard extends StatefulWidget {
  const BounceDropCard({super.key});

  @override
  State<BounceDropCard> createState() => _BounceDropCardState();
}

class _BounceDropCardState extends State<BounceDropCard>
    with SingleTickerProviderStateMixin {
  late final AnimationController _controller;
  late final Animation<double> _offsetAnimation;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 900),
    );

    // TweenSequence: 낙하(60%) → 반등(20%) → 안정화(20%)
    _offsetAnimation = TweenSequence<double>([
      TweenSequenceItem(
        tween: Tween(begin: -300.0, end: 0.0)
            .chain(CurveTween(curve: Curves.easeIn)),
        weight: 60,
      ),
      TweenSequenceItem(
        tween: Tween(begin: 0.0, end: -40.0)
            .chain(CurveTween(curve: Curves.easeOut)),
        weight: 20,
      ),
      TweenSequenceItem(
        tween: Tween(begin: -40.0, end: 0.0)
            .chain(CurveTween(curve: Curves.elasticOut)),
        weight: 20,
      ),
    ]).animate(_controller);

    _controller.forward();
  }

  @override
  void dispose() {
    _controller.dispose(); // 반드시 dispose!
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: _offsetAnimation,
      builder: (context, child) {
        return Transform.translate(
          offset: Offset(0, _offsetAnimation.value),
          child: child,
        );
      },
      // child를 분리하면 매 프레임 Card 트리를 재생성하지 않음
      child: Card(
        elevation: 8,
        shape: RoundedRectangleBorder(
          borderRadius: BorderRadius.circular(16),
        ),
        child: const Padding(
          padding: EdgeInsets.all(24),
          child: Text(
            '안녕하세요!',
            style: TextStyle(fontSize: 24, fontWeight: FontWeight.bold),
          ),
        ),
      ),
    );
  }
}
```

**핵심 포인트:** `child`를 `AnimatedBuilder`의 `child` 파라미터로 분리하면, builder가 호출될 때마다 `Card` 위젯 트리를 재생성하지 않아 렌더링 성능이 크게 향상됩니다.

---

### 예제 2: Staggered 애니메이션 — 리스트 아이템 순차 등장

여러 위젯이 순서대로 페이드인 + 슬라이드되는 스태거드(엇박자) 애니메이션입니다. 하나의 `AnimationController`를 공유하고, 각 아이템마다 `Interval`로 타이밍을 분리합니다.

```dart
import 'package:flutter/material.dart';

class StaggeredListView extends StatefulWidget {
  const StaggeredListView({super.key});

  @override
  State<StaggeredListView> createState() => _StaggeredListViewState();
}

class _StaggeredListViewState extends State<StaggeredListView>
    with SingleTickerProviderStateMixin {
  late final AnimationController _controller;
  final List<String> _items = ['Flutter', 'Dart', 'Riverpod', 'GoRouter'];

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 1200),
    )..forward();
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  Animation<double> _fadeAnim(int index) {
    final start = index / _items.length * 0.6;
    return Tween<double>(begin: 0.0, end: 1.0).animate(
      CurvedAnimation(
        parent: _controller,
        curve: Interval(start, start + 0.4, curve: Curves.easeOut),
      ),
    );
  }

  Animation<Offset> _slideAnim(int index) {
    final start = index / _items.length * 0.6;
    return Tween<Offset>(
      begin: const Offset(0.3, 0),
      end: Offset.zero,
    ).animate(
      CurvedAnimation(
        parent: _controller,
        curve: Interval(start, start + 0.4, curve: Curves.easeOut),
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      itemCount: _items.length,
      padding: const EdgeInsets.all(16),
      itemBuilder: (context, index) {
        return AnimatedBuilder(
          animation: _controller,
          builder: (context, child) {
            return FadeTransition(
              opacity: _fadeAnim(index),
              child: SlideTransition(
                position: _slideAnim(index),
                child: child,
              ),
            );
          },
          child: Card(
            margin: const EdgeInsets.symmetric(vertical: 8),
            child: ListTile(
              leading: const Icon(Icons.code, color: Colors.blue),
              title: Text(
                _items[index],
                style: const TextStyle(fontWeight: FontWeight.w600),
              ),
            ),
          ),
        );
      },
    );
  }
}
```

`Interval(start, end, curve: ...)` 의 `start`/`end`는 0.0~1.0 사이 값으로, 전체 컨트롤러 duration 중 이 애니메이션이 활성화될 구간을 지정합니다. index가 증가할수록 `start`가 커지므로 자연스럽게 엇박자 효과가 생깁니다.

---

## 주의사항과 실전 팁

### 1. dispose()는 선택이 아닌 필수

`AnimationController`는 내부적으로 `Ticker`를 보유합니다. `dispose()`를 호출하지 않으면 위젯이 트리에서 제거된 후에도 Ticker가 계속 살아 있어 **메모리 누수**가 발생합니다. Flutter DevTools의 Memory 탭에서 확인할 수 있습니다.

### 2. 컨트롤러 여러 개 — TickerProviderStateMixin 사용

`AnimationController`가 둘 이상이라면 반드시 `TickerProviderStateMixin`을 사용하세요. `SingleTickerProviderStateMixin`으로 두 개 이상의 컨트롤러에 `vsync: this`를 전달하면 런타임 에러가 발생합니다.

### 3. AnimatedBuilder child 파라미터 최적화

```dart
AnimatedBuilder(
  animation: animation,
  builder: (context, child) => Transform.scale(
    scale: animation.value,
    child: child, // 재생성되지 않음
  ),
  child: const ExpensiveWidget(), // 한 번만 빌드됨
);
```

`child` 파라미터를 활용하면 애니메이션 프레임마다 불필요하게 서브트리가 재빌드되는 것을 막을 수 있습니다.

### 4. RepaintBoundary로 GPU 레이어 격리

복잡한 애니메이션이 주변 UI에 영향을 주지 않도록 `RepaintBoundary`로 감싸면, 해당 영역만 별도 레이어로 GPU에서 처리됩니다.

```dart
RepaintBoundary(
  child: AnimatedBuilder(
    animation: _controller,
    builder: (_, __) => const MyComplexAnimation(),
  ),
);
```

Flutter DevTools의 **Repaint Rainbow** 기능을 활성화하면 어느 영역이 매 프레임 다시 그려지는지 색상으로 확인할 수 있어 최적화 포인트를 빠르게 찾을 수 있습니다.

### 5. CurveTween 체이닝 순서

`Tween.chain(CurveTween(curve: Curves.elasticOut))`은 Tween의 `transform()` 결과에 커브를 적용합니다. `TweenSequence` 내 각 단계에 독립적인 커브를 적용할 때 유용합니다. 체이닝은 **내부 → 외부** 순으로 적용되므로 순서에 주의하세요.

---

## 마치며

`AnimationController` + `TweenSequence` 조합은 Flutter에서 복잡한 다단계 애니메이션을 단일 컨트롤러로 깔끔하게 관리할 수 있게 해줍니다. 핵심은 **weight로 비율 지정**, **chain으로 커브 합성**, **child 파라미터로 불필요한 리빌드 방지**입니다. 여기에 `RepaintBoundary`까지 더하면 60fps를 안정적으로 유지하는 고품질 애니메이션을 구현할 수 있습니다.

---

## 참고 자료

- [Flutter 공식 애니메이션 튜토리얼](https://docs.flutter.dev/ui/animations/tutorial)
- [TweenSequence 클래스 — Dart API](https://api.flutter.dev/flutter/animation/TweenSequence-class.html)
- [AnimationController 클래스 — Dart API](https://api.flutter.dev/flutter/animation/AnimationController-class.html)
- [Staggered Animations — Flutter Docs](https://docs.flutter.dev/ui/animations/staggered-animations)
