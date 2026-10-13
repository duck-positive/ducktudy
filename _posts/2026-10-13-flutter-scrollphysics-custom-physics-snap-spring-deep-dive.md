---
layout: post
title: "Flutter ScrollPhysics 심화: 커스텀 스크롤 물리 엔진으로 네이티브급 UX 구현하기"
date: 2026-10-13
categories: [android, flutter]
tags: [flutter, scrollphysics, scroll, animation, ux, dart, custom-physics, snap, spring]
---

Flutter의 스크롤은 단순한 위/아래 이동이 아닙니다. 손가락을 뗐을 때 얼마나 미끄러지는지, 경계에서 어떻게 튕기는지, 특정 위치에 스냅되는지 — 이 모든 동작이 `ScrollPhysics`에 의해 결정됩니다. 이 글에서는 Flutter의 ScrollPhysics 시스템을 깊이 파헤치고, 실무에서 자주 필요한 커스텀 물리 엔진을 직접 구현해 봅니다.

## ScrollPhysics란 무엇인가?

`ScrollPhysics`는 Flutter에서 스크롤 동작의 '물리적 규칙'을 정의하는 추상 클래스입니다. 이 클래스를 구현함으로써 다음을 세밀하게 제어할 수 있습니다:

- **관성(Momentum)**: 손을 뗀 후 얼마나 멀리 미끄러지는가
- **경계 동작**: 끝에 도달했을 때 어떻게 반응하는가 (탄성 튕김 vs 하드 스톱)
- **스냅(Snap)**: 특정 위치에 자동으로 정렬되는가
- **드래그 마찰**: 드래그 중 얼마나 저항감을 주는가

ScrollPhysics의 핵심 메서드는 다음과 같습니다:

- `createBallisticSimulation()` — 손을 뗀 후 관성 시뮬레이션을 반환
- `applyPhysicsToUserOffset()` — 사용자 드래그 오프셋을 물리 규칙에 맞게 변환
- `applyBoundaryConditions()` — 스크롤이 경계를 넘지 못하도록 처리
- `applyTo()` — 다른 ScrollPhysics와 결합할 때 반드시 구현

## 기본 제공 ScrollPhysics 종류

Flutter는 플랫폼별로 적합한 내장 ScrollPhysics를 제공합니다:

| 클래스 | 설명 | 기본 플랫폼 |
|--------|------|------------|
| `BouncingScrollPhysics` | 경계에서 탄성 있게 튕김 | iOS |
| `ClampingScrollPhysics` | 경계에서 딱 멈춤 | Android |
| `PageScrollPhysics` | 페이지 단위 스냅 | PageView |
| `FixedExtentScrollPhysics` | 고정 높이 아이템 스냅 | ListWheelScrollView |
| `NeverScrollableScrollPhysics` | 스크롤 비활성화 | — |
| `AlwaysScrollableScrollPhysics` | 내용이 없어도 스크롤 가능 | — |

하지만 카드 캐러셀, 뮤직 플레이어 리스트, 독특한 고무줄 효과처럼 기본 물리 엔진으로는 표현하기 어려운 경우가 실무에서 빈번하게 등장합니다.

## 왜 커스텀 ScrollPhysics가 필요한가?

다음과 같은 상황에서 커스텀 물리 엔진이 필요합니다:

1. **카드 캐러셀**: ListView에서 카드 단위로 스냅하되 화면 중앙에 위치시킬 때
2. **음악 플레이어 플레이리스트**: 특정 트랙으로 부드럽게 슬라이드 스냅할 때
3. **커스텀 pull-to-refresh**: 당기는 저항감을 단계적으로 증가시킬 때
4. **게임 UI 스크롤**: 비선형적인 가속과 스프링 효과를 조합할 때
5. **페이지네이션 트리거**: 경계에 도달하는 방식 자체를 바꿀 때

## 실제 구현 예제 1: 아이템 스냅 ScrollPhysics

가장 빈번하게 요구되는 패턴은 특정 크기의 아이템 단위로 스냅하는 물리 엔진입니다. `PageView`는 전체 화면 단위지만, 카드처럼 커스텀 크기의 위젯에 스냅하려면 직접 구현해야 합니다.

```dart
import 'dart:math' as math;
import 'package:flutter/material.dart';

/// 지정된 픽셀 단위(itemDimension)로 스냅하는 ScrollPhysics
class SnapScrollPhysics extends ScrollPhysics {
  const SnapScrollPhysics({
    super.parent,
    required this.itemDimension,
  });

  final double itemDimension;

  @override
  SnapScrollPhysics applyTo(ScrollPhysics? ancestor) {
    return SnapScrollPhysics(
      parent: buildParent(ancestor),
      itemDimension: itemDimension,
    );
  }

  double _getTargetPixels(
    ScrollMetrics position,
    Tolerance tolerance,
    double velocity,
  ) {
    double page = position.pixels / itemDimension;

    if (velocity < -tolerance.velocity) {
      // 역방향 플링 → 이전 스냅 포인트
      page = page.floorToDouble();
    } else if (velocity > tolerance.velocity) {
      // 정방향 플링 → 다음 스냅 포인트
      page = page.ceilToDouble();
    } else {
      // 정지 상태 → 가장 가까운 스냅 포인트
      page = page.roundToDouble();
    }

    return math.max(
      0,
      math.min(page * itemDimension, position.maxScrollExtent),
    );
  }

  @override
  Simulation? createBallisticSimulation(
    ScrollMetrics position,
    double velocity,
  ) {
    // 경계에 있을 때 부모 물리 엔진에게 위임
    if ((velocity <= 0.0 && position.pixels <= position.minScrollExtent) ||
        (velocity >= 0.0 && position.pixels >= position.maxScrollExtent)) {
      return super.createBallisticSimulation(position, velocity);
    }

    final Tolerance tolerance = toleranceFor(position);
    final double target = _getTargetPixels(position, tolerance, velocity);

    if (target != position.pixels) {
      return ScrollSpringSimulation(
        spring,
        position.pixels,
        target,
        velocity,
        tolerance: tolerance,
      );
    }
    return null;
  }

  @override
  bool get allowImplicitScrolling => false;
}
```

이 물리 엔진을 적용한 수평 카드 캐러셀 예시입니다:

```dart
class CardCarouselPage extends StatelessWidget {
  const CardCarouselPage({super.key});

  static const double _cardWidth = 280.0;
  static const double _cardMargin = 8.0;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('카드 캐러셀')),
      body: ListView.builder(
        scrollDirection: Axis.horizontal,
        // 카드 폭 + 양쪽 마진을 스냅 단위로 설정
        physics: const SnapScrollPhysics(
          itemDimension: _cardWidth + _cardMargin * 2,
        ),
        itemCount: 10,
        itemBuilder: (context, index) {
          return Container(
            width: _cardWidth,
            margin: const EdgeInsets.all(_cardMargin),
            decoration: BoxDecoration(
              borderRadius: BorderRadius.circular(16),
              color: Colors.primaries[index % Colors.primaries.length],
              boxShadow: [
                BoxShadow(
                  color: Colors.black.withOpacity(0.15),
                  blurRadius: 8,
                  offset: const Offset(0, 4),
                ),
              ],
            ),
            child: Center(
              child: Text(
                'Card ${index + 1}',
                style: const TextStyle(
                  color: Colors.white,
                  fontSize: 24,
                  fontWeight: FontWeight.bold,
                ),
              ),
            ),
          );
        },
      ),
    );
  }
}
```

`allowImplicitScrolling`을 `false`로 오버라이드한 것이 핵심입니다. `true`이면 PageView가 내부적으로 이웃 페이지를 미리 빌드(preload)하는데, 커스텀 스냅에서는 불필요한 빌드가 발생합니다.

## 실제 구현 예제 2: 커스텀 스프링 물리 엔진

기본 `BouncingScrollPhysics`의 스프링 강도를 세밀하게 제어하거나, 경계 밖 드래그에 고무줄(Rubber Band) 저항감을 구현하고 싶을 때 커스텀 스프링 물리 엔진을 직접 만들 수 있습니다.

```dart
import 'dart:math' as math;
import 'package:flutter/material.dart';
import 'package:flutter/physics.dart';

/// 스프링 강도·댐핑·질량을 세밀하게 조정 가능한 ScrollPhysics
class CustomSpringScrollPhysics extends ScrollPhysics {
  const CustomSpringScrollPhysics({
    super.parent,
    this.stiffness = 100.0, // 스프링 강도 (높을수록 빨리 복원)
    this.dampingRatio = 0.8, // 댐핑 비율 (1.0 = 임계댐핑, 튕김 없음)
    this.mass = 1.0,         // 가상 질량 (높을수록 무거운 느낌)
  });

  final double stiffness;
  final double dampingRatio;
  final double mass;

  @override
  CustomSpringScrollPhysics applyTo(ScrollPhysics? ancestor) {
    return CustomSpringScrollPhysics(
      parent: buildParent(ancestor),
      stiffness: stiffness,
      dampingRatio: dampingRatio,
      mass: mass,
    );
  }

  /// withDampingRatio 헬퍼로 복잡한 댐핑 계산을 위임
  SpringDescription get _spring => SpringDescription.withDampingRatio(
        mass: mass,
        stiffness: stiffness,
        ratio: dampingRatio,
      );

  @override
  Simulation? createBallisticSimulation(
    ScrollMetrics position,
    double velocity,
  ) {
    final Tolerance tolerance = toleranceFor(position);

    // 경계를 벗어난 경우 → 커스텀 스프링으로 복원
    if (position.outOfRange) {
      final double target = position.pixels < position.minScrollExtent
          ? position.minScrollExtent
          : position.maxScrollExtent;

      return SpringSimulation(
        _spring,
        position.pixels,
        target,
        velocity,
        tolerance: tolerance,
      );
    }

    // 속도가 허용치 이하이면 정지
    if (velocity.abs() < tolerance.velocity) return null;

    return super.createBallisticSimulation(position, velocity);
  }

  @override
  double applyPhysicsToUserOffset(ScrollMetrics position, double offset) {
    // 경계 밖 드래그 시 Rubber Band 저항감 부여
    if (position.outOfRange) {
      final double overscrollPastStart =
          math.max(position.minScrollExtent - position.pixels, 0.0);
      final double overscrollPastEnd =
          math.max(position.pixels - position.maxScrollExtent, 0.0);
      final double overscroll =
          math.max(overscrollPastStart, overscrollPastEnd);

      // 경계에서 멀수록 저항이 강해지는 비선형 함수 (최대 85% 감쇠)
      final double multiplier = 1.0 - math.min(overscroll / 200.0, 0.85);
      return offset * multiplier;
    }
    return offset;
  }
}
```

세 가지 스프링 강도를 직접 비교해볼 수 있는 예시입니다:

```dart
class SpringPhysicsDemo extends StatelessWidget {
  const SpringPhysicsDemo({super.key});

  // 딱딱한 스프링: 빠른 복원, 약간의 튕김
  static const _stiffSpring = CustomSpringScrollPhysics(
    stiffness: 300.0,
    dampingRatio: 0.9,
  );

  // 부드러운 스프링: 느린 복원, 살짝 튕김
  static const _softSpring = CustomSpringScrollPhysics(
    stiffness: 50.0,
    dampingRatio: 0.6,
  );

  // 젤리 스프링: 과감한 언더댐핑, 여러 번 튕김
  static const _jellySpring = CustomSpringScrollPhysics(
    stiffness: 80.0,
    dampingRatio: 0.4,
    mass: 0.5,
  );

  @override
  Widget build(BuildContext context) {
    return DefaultTabController(
      length: 3,
      child: Scaffold(
        appBar: AppBar(
          title: const Text('스프링 물리 비교'),
          bottom: const TabBar(
            tabs: [
              Tab(text: '딱딱'),
              Tab(text: '부드럽'),
              Tab(text: '젤리'),
            ],
          ),
        ),
        body: TabBarView(
          children: [_buildList(_stiffSpring), _buildList(_softSpring), _buildList(_jellySpring)],
        ),
      ),
    );
  }

  Widget _buildList(CustomSpringScrollPhysics physics) {
    return ListView.builder(
      physics: physics,
      itemCount: 20,
      itemBuilder: (context, index) => ListTile(
        leading: CircleAvatar(child: Text('${index + 1}')),
        title: Text('Item ${index + 1}'),
        subtitle: Text('스프링 강도: ${physics.stiffness}'),
      ),
    );
  }
}
```

## ScrollPhysics 체이닝 패턴

Flutter의 ScrollPhysics는 `parent`를 통해 체인을 형성할 수 있습니다. 여러 물리 효과를 조합할 때 유용합니다:

```dart
// 스냅 + 탄성 경계를 동시에 적용
const combinedPhysics = SnapScrollPhysics(
  parent: BouncingScrollPhysics(),
  itemDimension: 300,
);
```

`parent`가 체인의 끝에서 경계 동작을 담당하고, 상위 물리 엔진이 스냅이나 스프링 로직을 먼저 처리합니다. **`applyTo()` 구현 시 `buildParent(ancestor)` 호출을 절대 빠뜨리면 안 됩니다.** 이를 생략하면 `ScrollView`가 주입하는 플랫폼 기본 물리 엔진이 무시됩니다.

## 주의사항과 실전 팁

### 1. physics 비교는 runtimeType 기반

Flutter는 physics 변경 여부를 `runtimeType`으로 비교합니다. 같은 클래스에 파라미터만 다른 경우 변경을 인식하지 못할 수 있습니다. 이 때문에 physics 객체를 `const` 상수로 선언하는 것이 좋습니다.

### 2. applyTo() 구현 필수 패턴

```dart
// ❌ parent 누락 — 플랫폼 기본 물리 엔진이 무시됨
@override
SnapScrollPhysics applyTo(ScrollPhysics? ancestor) {
  return SnapScrollPhysics(itemDimension: itemDimension);
}

// ✅ buildParent(ancestor) 반드시 포함
@override
SnapScrollPhysics applyTo(ScrollPhysics? ancestor) {
  return SnapScrollPhysics(
    parent: buildParent(ancestor),
    itemDimension: itemDimension,
  );
}
```

### 3. toleranceFor() 활용

하드코딩된 속도/위치 허용 범위 대신 `toleranceFor(position)`을 사용하면 DPI와 기기에 따라 자동으로 보정됩니다.

### 4. null 반환 = 즉시 정지

`createBallisticSimulation()`에서 `null`을 반환하면 관성 없이 즉시 멈춥니다. 이를 이용해 특정 조건에서만 관성 스크롤을 제거할 수 있습니다.

### 5. SpringDescription.withDampingRatio 권장

직접 댐핑 계수를 계산하는 것은 수식이 복잡합니다. Flutter가 제공하는 헬퍼를 사용하면 직관적인 `ratio` 값(0~1)으로 제어가 훨씬 쉽습니다:

```dart
// ratio = 1.0: 임계댐핑 (튕김 없이 부드럽게 복원)
// ratio < 1.0: 언더댐핑 (복원 후 튕김 발생)
// ratio > 1.0: 오버댐핑 (느리게 복원, 튕김 없음)
final spring = SpringDescription.withDampingRatio(
  mass: 1.0,
  stiffness: 100.0,
  ratio: 0.75,
);
```

### 6. Simulation은 불변 객체로

`createBallisticSimulation()`은 스크롤이 시작할 때마다 호출됩니다. 이 메서드가 무거운 연산을 수행하면 첫 프레임에 jank가 발생할 수 있으니, 시뮬레이션 객체는 가볍게 생성하도록 유지하세요.

## 마치며

`ScrollPhysics`는 Flutter에서 '손맛'을 직접 설계하는 강력한 도구입니다. 기본 제공 물리 엔진이 충족하지 못하는 UX 요구사항을 만날 때, 커스텀 ScrollPhysics로 네이티브 앱에 버금가는 인터랙션을 구현할 수 있습니다. 스냅·스프링·고무줄 효과를 체이닝 패턴으로 조합하면, 사용자가 '앱처럼 느껴진다'고 감탄하는 섬세한 스크롤 경험을 만들 수 있습니다.

## 참고 자료
- [Flutter scroll_physics.dart 소스 코드 (GitHub)](https://github.com/flutter/flutter/blob/main/packages/flutter/lib/src/widgets/scroll_physics.dart)
- [snap_panel 패키지 - 스프링 애니메이션과 스냅 포인트](https://pub.dev/packages/snap_panel)
- [custom_refresh_indicator - 커스텀 ScrollPhysics 활용 예시](https://pub.dev/packages/custom_refresh_indicator)
