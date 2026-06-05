---
layout: post
title: "Flutter 성능 최적화 완전 정복: DevTools와 Repaint Rainbow로 60FPS 달성하기"
date: 2026-06-05
categories: [android, flutter]
tags: [flutter, performance, devtools, repaint-rainbow, repaintboundary, optimization, 60fps, jank]
---

Flutter 앱이 느려지거나 화면이 뚝뚝 끊기는 경험(Jank)은 사용자 경험에 치명적이다. 60FPS를 유지하려면 각 프레임을 16.67ms 이내에 렌더링해야 하는데, 불필요한 위젯 재빌드나 과도한 Paint 영역이 이 시간을 초과시킨다. 이 글에서는 Flutter DevTools의 Performance 탭과 Repaint Rainbow를 활용하여 성능 병목을 찾아내고, RepaintBoundary와 const 생성자를 통해 최적화하는 실전 방법을 다룬다.

## 1. 왜 Flutter 성능 최적화가 필요한가?

Flutter의 렌더링 파이프라인은 크게 세 단계로 나뉜다: **Build → Layout → Paint**. 각 단계에서 불필요한 작업이 발생하면 프레임 시간이 늘어난다.

- **Build 단계**: `setState()`가 호출되면 해당 위젯 서브트리 전체를 다시 빌드한다. 규모가 큰 위젯 트리에서 빈번한 상태 변경은 CPU 부하를 급격히 높인다.
- **Paint 단계**: GPU가 화면에 픽셀을 그릴 때, 변경된 영역만 그려야 하지만 최적화가 없으면 전체 화면을 다시 그리게 된다.
- **16ms 규칙**: 60FPS에서 한 프레임은 16.67ms다. UI 스레드와 Raster 스레드가 각각 이 시간을 초과하면 Jank가 발생한다.

이를 해결하기 위한 두 가지 강력한 도구가 바로 **Flutter DevTools**와 **Repaint Rainbow**다.

## 2. Flutter DevTools: 성능 병목의 나침반

### DevTools 실행 방법

Flutter 앱을 `flutter run --profile` 모드로 실행하면 DevTools와 연결하여 실제 성능 데이터를 수집할 수 있다. Debug 모드는 JIT 컴파일로 인해 성능이 왜곡되므로 Profile 모드를 반드시 사용해야 한다.

```bash
flutter run --profile
# VS Code에서: "Flutter: Run in Profile Mode" 선택
# Android Studio에서: Run > Profile 선택
```

그 후 브라우저에서 DevTools를 열면 Performance 탭에 접근할 수 있다.

### Frame Chart 해석하기

Performance 탭의 Frame Chart에서 각 프레임은 막대로 표시된다. 빨간색 막대는 Jank 프레임(16ms 초과)을 나타내며, UI 스레드(파란색)와 Raster 스레드(녹색) 두 줄로 나뉜다.

| 스레드 | 담당 작업 | 병목 시 해결책 |
|--------|-----------|----------------|
| **UI 스레드** | Dart 코드 실행 (Build, Layout, Paint 명령 생성) | `const` 생성자, 위젯 분리 |
| **Raster 스레드** | GPU에 실제 픽셀을 그리는 작업 | `RepaintBoundary`, 이미지 캐싱 |

### Track Widget Builds 기능

DevTools Performance 탭에서 **Track Widget Builds** 옵션을 활성화하면 타임라인에서 어떤 위젯이 몇 번 빌드됐는지 확인할 수 있다. 이 기능을 통해 불필요한 재빌드가 발생하는 위젯을 정확히 찾아낼 수 있다.

## 3. Repaint Rainbow: 화면 재페인팅 시각화

### Repaint Rainbow란?

Repaint Rainbow는 Flutter의 디버그 플래그로, 화면에서 다시 페인팅(repaint)되는 영역을 무지개 색상으로 시각화한다. 페인팅이 발생할 때마다 해당 영역의 색상이 변하므로, 색상이 자주 바뀌는 영역이 과도하게 페인팅되고 있다는 신호다.

### Repaint Rainbow 활성화

```dart
import 'package:flutter/rendering.dart';

void main() {
  // Repaint Rainbow 활성화 — 렌더 레이어별로 무지개 테두리 표시
  debugRepaintRainbowEnabled = true;

  // 텍스트 재페인팅도 시각화하고 싶다면
  debugRepaintTextRainbowEnabled = true;

  runApp(const MyApp());
}
```

앱을 실행하면 각 렌더 레이어가 무지개 색상 테두리로 표시된다. 프레임마다 색상이 바뀌는 영역을 주목하라. 애니메이션이 없는 정적인 부분까지 색상이 변한다면 불필요한 재페인팅이 발생하고 있는 것이다.

> **핵심 판단 기준**: 정적 콘텐츠(텍스트, 아이콘, 리스트 아이템)의 색상이 계속 바뀌면 문제다. 애니메이션 위젯만 바뀌는 것은 정상이다.

## 4. RepaintBoundary로 페인팅 최적화

### 문제 상황: 전파되는 재페인팅

Repaint Rainbow를 켰을 때 화면 전체가 계속 색상이 바뀐다면, 일부 위젯의 변화가 전체 트리로 전파되고 있다는 뜻이다. 예를 들어 하단에 애니메이션 위젯이 있으면, 상단의 정적인 리스트까지 함께 재페인팅될 수 있다.

### RepaintBoundary 적용 전 — 전체 재페인팅 문제

```dart
// 문제: AnimatedWidget이 변할 때 ListView도 함께 재페인팅됨
class ProblematicScreen extends StatefulWidget {
  const ProblematicScreen({super.key});

  @override
  State<ProblematicScreen> createState() => _ProblematicScreenState();
}

class _ProblematicScreenState extends State<ProblematicScreen>
    with SingleTickerProviderStateMixin {
  late final AnimationController _controller;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 800),
    )..repeat(reverse: true);
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        // 정적 리스트 — 매 프레임마다 재페인팅됨 (문제!)
        Expanded(
          child: ListView.builder(
            itemCount: 100,
            itemBuilder: (context, index) => ListTile(
              title: Text('아이템 $index'),
            ),
          ),
        ),
        // 하단 애니메이션이 전체 Column을 오염시킴
        AnimatedBuilder(
          animation: _controller,
          builder: (context, child) {
            return Container(
              height: 60,
              color: Color.lerp(
                Colors.blue,
                Colors.red,
                _controller.value,
              ),
              child: const Center(child: Text('애니메이션 영역')),
            );
          },
        ),
      ],
    );
  }
}
```

### RepaintBoundary 적용 후 — 독립 레이어로 격리

```dart
// 해결: RepaintBoundary로 각 영역을 독립된 레이어로 분리
class OptimizedScreen extends StatefulWidget {
  const OptimizedScreen({super.key});

  @override
  State<OptimizedScreen> createState() => _OptimizedScreenState();
}

class _OptimizedScreenState extends State<OptimizedScreen>
    with SingleTickerProviderStateMixin {
  late final AnimationController _controller;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 800),
    )..repeat(reverse: true);
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Expanded(
          // RepaintBoundary: ListView를 독립 레이어로 분리
          // 하단 애니메이션이 변해도 이 영역은 재페인팅되지 않음
          child: RepaintBoundary(
            child: ListView.builder(
              itemCount: 100,
              itemBuilder: (context, index) => ListTile(
                title: Text('아이템 $index'),
              ),
            ),
          ),
        ),
        // 애니메이션 영역도 RepaintBoundary로 격리
        // 오직 이 레이어만 매 프레임 재페인팅됨
        RepaintBoundary(
          child: AnimatedBuilder(
            animation: _controller,
            builder: (context, child) {
              return Container(
                height: 60,
                color: Color.lerp(
                  Colors.blue,
                  Colors.red,
                  _controller.value,
                ),
                child: const Center(child: Text('애니메이션 영역')),
              );
            },
          ),
        ),
      ],
    );
  }
}
```

`RepaintBoundary`를 적용하면 Flutter 렌더러가 해당 위젯을 별도의 레이어(Layer)로 처리하므로, 그 안에서의 변화가 외부로 전파되지 않는다. Repaint Rainbow에서 확인하면 ListView 영역의 색상이 더 이상 변하지 않는 것을 볼 수 있다.

## 5. const 생성자와 위젯 분리로 Build 최적화

### const 생성자의 위력

`const` 생성자로 생성된 위젯은 컴파일 타임에 상수로 처리된다. 부모 위젯이 재빌드될 때 `const` 위젯은 재빌드되지 않고 캐시된 인스턴스를 재사용한다.

```dart
// Bad: 매번 새 인스턴스 생성
@override
Widget build(BuildContext context) {
  return Column(
    children: [
      Icon(Icons.star),      // 매 빌드마다 새 객체 생성
      Text('제목'),          // 매 빌드마다 새 객체 생성
      _buildDynamicPart(),
    ],
  );
}

// Good: const로 불변 위젯을 컴파일 타임 상수로 처리
@override
Widget build(BuildContext context) {
  return Column(
    children: [
      const Icon(Icons.star), // 상수 — 재빌드 없음
      const Text('제목'),      // 상수 — 재빌드 없음
      _buildDynamicPart(),    // 동적 부분만 재빌드
    ],
  );
}
```

### 위젯 분리로 재빌드 범위 최소화

`setState()`의 영향 범위를 줄이려면 상태를 가진 위젯을 최대한 작게 분리해야 한다.

```dart
// Bad: 전체 페이지가 카운터 변경으로 재빌드됨
class BadCounterPage extends StatefulWidget {
  const BadCounterPage({super.key});
  @override
  State<BadCounterPage> createState() => _BadCounterPageState();
}

class _BadCounterPageState extends State<BadCounterPage> {
  int _count = 0;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Column(
        children: [
          const ExpensiveStaticWidget(), // _count와 무관한데도 재빌드됨
          const AnotherHeavyWidget(),    // 동일 문제
          Text('카운트: $_count'),
          ElevatedButton(
            onPressed: () => setState(() => _count++),
            child: const Text('증가'),
          ),
        ],
      ),
    );
  }
}

// Good: 카운터 부분만 StatefulWidget으로 분리
class GoodCounterPage extends StatelessWidget {
  const GoodCounterPage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Column(
        children: [
          const ExpensiveStaticWidget(), // 절대 재빌드되지 않음
          const AnotherHeavyWidget(),    // 절대 재빌드되지 않음
          const _CounterWidget(),        // 이 위젯만 재빌드됨
        ],
      ),
    );
  }
}

class _CounterWidget extends StatefulWidget {
  const _CounterWidget();
  @override
  State<_CounterWidget> createState() => __CounterWidgetState();
}

class __CounterWidgetState extends State<_CounterWidget> {
  int _count = 0;

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Text('카운트: $_count'),
        ElevatedButton(
          onPressed: () => setState(() => _count++),
          child: const Text('증가'),
        ),
      ],
    );
  }
}
```

## 6. PerformanceOverlay로 실시간 FPS 모니터링

앱 자체에 FPS 오버레이를 표시해 실시간으로 성능을 확인할 수 있다.

```dart
MaterialApp(
  showPerformanceOverlay: true, // 상단에 FPS 그래프 표시
  home: const MyHomePage(),
)
```

오버레이는 두 줄의 그래프를 표시한다. 상단은 Raster 스레드, 하단은 UI 스레드를 나타낸다. 녹색 막대는 16ms 이하(정상), 빨간색은 초과(Jank)를 의미한다. 실기기에서 스크롤이나 애니메이션을 실행하며 빨간색 막대가 나타나는 시점을 포착하면 된다.

## 7. 추가 최적화 팁

### 이미지 디코딩 크기 제한

고해상도 이미지를 로드할 때 `cacheWidth`, `cacheHeight` 파라미터로 디코딩 크기를 제한하면 메모리와 페인팅 비용을 동시에 절감할 수 있다.

```dart
Image.asset(
  'assets/large_image.png',
  cacheWidth: 200,   // 화면 표시 크기에 맞게 디코딩 크기 제한
  cacheHeight: 200,
)
```

### ListView에서 itemExtent 활용

`ListView.builder`에서 `itemExtent`를 지정하면 Flutter가 레이아웃 계산을 건너뛰어 스크롤 성능이 크게 향상된다.

```dart
ListView.builder(
  itemExtent: 72.0,  // 고정 높이 지정 — 레이아웃 계산 생략
  itemCount: items.length,
  itemBuilder: (context, index) => ItemWidget(item: items[index]),
)
```

## 8. 주의사항

- **RepaintBoundary 남용 금지**: 별도 레이어 생성으로 메모리가 추가로 사용된다. 변화가 드문 정적 위젯에 과도하게 적용하면 GPU 메모리가 낭비된다. 반드시 Repaint Rainbow로 확인 후 필요한 곳에만 적용하라.
- **Profile 모드에서만 측정**: Debug 모드는 JIT 오버헤드로 실제 성능과 차이가 크다. 항상 `flutter run --profile`로 측정하라.
- **측정 → 최적화 → 재측정**: 느낌이나 가정이 아닌 DevTools 데이터에 기반하여 최적화하라. 특정 위젯이 느리다고 가정하기 전에 반드시 프로파일링부터 하라.

## 정리

Flutter 성능 최적화의 핵심은 **측정 → 분석 → 최적화** 사이클이다. DevTools의 Frame Chart와 Track Widget Builds로 병목을 찾고, Repaint Rainbow로 과도한 재페인팅 영역을 식별한 뒤, `RepaintBoundary`와 `const` 생성자로 최적화한다. 도구를 무작정 적용하기보다 데이터에 기반한 최적화가 가장 효과적이다.

## 참고 자료

- [Flutter DevTools — 공식 GitHub 저장소](https://github.com/flutter/devtools)
- [PerformanceOverlay 소스 코드 — flutter/flutter](https://github.com/flutter/flutter/blob/master/packages/flutter/lib/src/widgets/performance_overlay.dart)
- [flutter_performance_optimizer — pub.dev](https://pub.dev/packages/flutter_performance_optimizer)
