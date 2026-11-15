---
layout: post
title: "Flutter RepaintBoundary 완전 정복: 위젯 재렌더링을 최소화하는 렌더링 최적화 전략"
date: 2026-11-15
categories: [android, flutter]
tags: [flutter, dart, repaintboundary, rendering, performance, optimization, custompaint, animation]
---

Flutter 앱 개발을 하다 보면 애니메이션이 버벅거리거나 스크롤이 부드럽지 않은 경험을 하게 됩니다. 대부분의 경우 불필요한 위젯 재렌더링이 원인입니다. `RepaintBoundary`는 Flutter가 제공하는 강력한 렌더링 최적화 도구로, 올바르게 사용하면 GPU 연산을 획기적으로 줄일 수 있습니다. 이 글에서는 `RepaintBoundary`의 내부 동작 원리부터 실전 적용 패턴, 디버깅 방법, 그리고 주의해야 할 함정까지 완전히 정복합니다.

## 개념 설명: Flutter 렌더링 파이프라인과 RepaintBoundary

Flutter의 렌더링은 세 단계로 구성됩니다.

1. **Build**: `Widget` 트리를 `Element` 트리로 변환
2. **Layout**: `RenderObject`의 크기와 위치 계산 (`onLayout`)
3. **Paint**: `RenderObject`를 실제 픽셀로 그리기 (`onPaint`)

이 세 단계 중 Paint 단계가 GPU를 가장 많이 사용합니다. 위젯 하나가 변경되면 Flutter는 해당 `RenderObject`와 연결된 레이어를 다시 그려야 합니다. 문제는 **레이어 경계**가 없을 때 발생합니다. 즉, 작은 변화 하나가 의도치 않게 더 넓은 영역을 더티(dirty) 상태로 만들어 불필요한 재렌더링을 유발합니다.

`RepaintBoundary` 위젯은 내부적으로 `RenderRepaintBoundary`를 생성하고, 이는 Flutter 엔진에게 "이 경계 안의 콘텐츠는 별도의 GPU 레이어로 처리하라"고 지시합니다. GPU는 이미 렌더링된 레이어를 **래스터 캐시(Raster Cache)**에 저장하고, 해당 레이어가 변경되지 않는 한 캐시에서 직접 가져다 씁니다.

```
위젯 트리 (RepaintBoundary 없이)
┌────────────────────────────────┐
│ Scaffold                       │
│  ├─ AppBar  ← 애니메이션 변경  │  ← 전체 레이어 재렌더링!
│  ├─ ComplexChart               │
│  └─ BottomNav                  │
└────────────────────────────────┘

위젯 트리 (RepaintBoundary 적용 후)
┌────────────────────────────────┐
│ Scaffold                       │
│  ├─ [Layer A] AppBar ← 변경    │  ← Layer A만 재렌더링
│  ├─ [Layer B] ComplexChart     │  ← 캐시에서 재사용 ✓
│  └─ [Layer C] BottomNav        │  ← 캐시에서 재사용 ✓
└────────────────────────────────┘
```

## 왜 필요한가: 성능 저하의 실제 원인

### 케이스 1: 애니메이션과 복잡한 정적 콘텐츠의 혼재

초당 60프레임(또는 120프레임)으로 동작하는 애니메이션이 있을 때, 화면에 복잡한 정적 위젯도 함께 있다면, `RepaintBoundary` 없이는 매 프레임마다 정적 위젯도 함께 렌더링됩니다. 60fps 기준으로 초당 60회씩 불필요한 GPU 연산이 발생합니다.

### 케이스 2: 스크롤 뷰 내부의 복잡한 항목

`ListView.builder`나 `GridView.builder`는 내부적으로 각 항목을 `RepaintBoundary`로 자동 감쌉니다. 그러나 직접 `SingleChildScrollView`와 `Column`을 조합해 목록을 만들면 이 최적화가 적용되지 않아, 한 항목이 변경될 때 전체 목록이 재렌더링될 수 있습니다.

### 케이스 3: CustomPainter를 사용하는 복잡한 차트나 그래픽

`CustomPainter`로 그린 복잡한 차트나 그래픽은 렌더링 비용이 큽니다. 차트 데이터는 변경되지 않는데 주변 위젯의 상태 변화로 인해 차트가 반복 렌더링되면 큰 성능 낭비가 발생합니다.

## 실제 구현 예제

### 예제 1: 애니메이션과 정적 콘텐츠 분리

다음 예제는 회전 애니메이션과 복잡한 정적 카드 목록이 함께 있는 화면을 최적화합니다.

```dart
import 'package:flutter/material.dart';

class OptimizedAnimationScreen extends StatefulWidget {
  const OptimizedAnimationScreen({super.key});

  @override
  State<OptimizedAnimationScreen> createState() =>
      _OptimizedAnimationScreenState();
}

class _OptimizedAnimationScreenState extends State<OptimizedAnimationScreen>
    with SingleTickerProviderStateMixin {
  late final AnimationController _controller;
  late final Animation<double> _rotationAnim;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(seconds: 3),
    )..repeat();
    _rotationAnim = Tween<double>(begin: 0.0, end: 1.0).animate(_controller);
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('RepaintBoundary 최적화 예제')),
      body: Column(
        children: [
          // ① 애니메이션 위젯을 별도 레이어로 격리
          RepaintBoundary(
            child: AnimatedBuilder(
              animation: _rotationAnim,
              builder: (context, child) {
                return Transform.rotate(
                  angle: _rotationAnim.value * 2 * 3.14159,
                  child: child,
                );
              },
              // child는 AnimatedBuilder 내부에서 재빌드되지 않음
              child: Container(
                width: 80,
                height: 80,
                decoration: const BoxDecoration(
                  gradient: LinearGradient(
                    colors: [Colors.blue, Colors.purple],
                  ),
                  shape: BoxShape.circle,
                ),
                child: const Icon(Icons.refresh, color: Colors.white, size: 40),
              ),
            ),
          ),
          const SizedBox(height: 16),
          // ② 복잡한 정적 콘텐츠를 별도 레이어로 격리
          Expanded(
            child: RepaintBoundary(
              child: _StaticComplexList(),
            ),
          ),
        ],
      ),
    );
  }
}

class _StaticComplexList extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      itemCount: 50,
      itemBuilder: (context, index) {
        return Card(
          margin: const EdgeInsets.symmetric(horizontal: 16, vertical: 4),
          child: ListTile(
            leading: CircleAvatar(
              backgroundColor: Colors.primaries[index % Colors.primaries.length],
              child: Text('${index + 1}',
                  style: const TextStyle(color: Colors.white)),
            ),
            title: Text('정적 항목 #${index + 1}'),
            subtitle: Text('이 콘텐츠는 애니메이션이 돌아도 재렌더링되지 않습니다'),
            trailing: const Icon(Icons.chevron_right),
          ),
        );
      },
    );
  }
}
```

이 예제에서 `_rotationAnim`은 초당 60회 값이 변경됩니다. `RepaintBoundary` 없이는 `_StaticComplexList`도 매 프레임 재렌더링됩니다. `RepaintBoundary`를 추가하면 애니메이션 레이어와 목록 레이어가 분리되어, 목록은 GPU 래스터 캐시에서 재사용됩니다.

---

### 예제 2: CustomPainter 차트를 두 레이어로 분리

복잡한 차트 배경은 한 번만 그리고, 사용자 인터랙션으로 변경되는 선택 인디케이터만 별도 레이어에서 갱신하는 패턴입니다.

```dart
import 'dart:math';
import 'package:flutter/material.dart';

class SplitLayerChart extends StatefulWidget {
  const SplitLayerChart({super.key});

  @override
  State<SplitLayerChart> createState() => _SplitLayerChartState();
}

class _SplitLayerChartState extends State<SplitLayerChart> {
  final List<double> _data = List.generate(
    20,
    (i) => 20 + (sin(i * 0.5) * 15) + Random().nextDouble() * 10,
  );
  final ValueNotifier<int> _selectedIndex = ValueNotifier<int>(-1);

  @override
  void dispose() {
    _selectedIndex.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('분리 레이어 차트')),
      body: Column(
        children: [
          Padding(
            padding: const EdgeInsets.all(16),
            child: GestureDetector(
              onPanUpdate: (details) {
                final box = context.findRenderObject() as RenderBox;
                final localPos = box.globalToLocal(details.globalPosition);
                final stepX = box.size.width / (_data.length - 1);
                final index = (localPos.dx / stepX).round().clamp(0, _data.length - 1);
                _selectedIndex.value = index;
              },
              child: SizedBox(
                height: 200,
                child: Stack(
                  children: [
                    // ① 차트 배경: 데이터가 변경되지 않는 한 재렌더링 없음
                    RepaintBoundary(
                      child: CustomPaint(
                        size: const Size(double.infinity, 200),
                        painter: _ChartBackgroundPainter(dataPoints: _data),
                      ),
                    ),
                    // ② 선택 인디케이터: 드래그할 때만 갱신
                    ValueListenableBuilder<int>(
                      valueListenable: _selectedIndex,
                      builder: (context, index, _) {
                        return RepaintBoundary(
                          child: CustomPaint(
                            size: const Size(double.infinity, 200),
                            painter: _SelectionPainter(
                              dataPoints: _data,
                              selectedIndex: index,
                            ),
                          ),
                        );
                      },
                    ),
                  ],
                ),
              ),
            ),
          ),
          ValueListenableBuilder<int>(
            valueListenable: _selectedIndex,
            builder: (context, index, _) {
              if (index < 0) return const Text('차트를 드래그해 보세요');
              return Text(
                '인덱스 $index: ${_data[index].toStringAsFixed(1)}',
                style: const TextStyle(fontSize: 18, fontWeight: FontWeight.bold),
              );
            },
          ),
        ],
      ),
    );
  }
}

class _ChartBackgroundPainter extends CustomPainter {
  final List<double> dataPoints;

  const _ChartBackgroundPainter({required this.dataPoints});

  @override
  void paint(Canvas canvas, Size size) {
    if (dataPoints.length < 2) return;

    final maxVal = dataPoints.reduce(max);
    final minVal = dataPoints.reduce(min);
    final range = maxVal - minVal;

    // 그리드 라인
    final gridPaint = Paint()
      ..color = Colors.grey.shade200
      ..strokeWidth = 1.0;
    for (int i = 0; i <= 4; i++) {
      final y = size.height * i / 4;
      canvas.drawLine(Offset(0, y), Offset(size.width, y), gridPaint);
    }

    // 차트 선
    final linePaint = Paint()
      ..color = Colors.blue.shade400
      ..strokeWidth = 2.5
      ..style = PaintingStyle.stroke
      ..strokeCap = StrokeCap.round
      ..strokeJoin = StrokeJoin.round;

    final fillPaint = Paint()
      ..color = Colors.blue.shade400.withOpacity(0.1)
      ..style = PaintingStyle.fill;

    final stepX = size.width / (dataPoints.length - 1);
    final path = Path();
    final fillPath = Path();

    for (int i = 0; i < dataPoints.length; i++) {
      final x = i * stepX;
      final normalised = (dataPoints[i] - minVal) / range;
      final y = size.height - normalised * size.height * 0.85 - size.height * 0.075;

      if (i == 0) {
        path.moveTo(x, y);
        fillPath.moveTo(x, size.height);
        fillPath.lineTo(x, y);
      } else {
        path.lineTo(x, y);
        fillPath.lineTo(x, y);
      }
    }

    fillPath.lineTo(size.width, size.height);
    fillPath.close();

    canvas.drawPath(fillPath, fillPaint);
    canvas.drawPath(path, linePaint);

    // 데이터 포인트 점
    final dotPaint = Paint()
      ..color = Colors.blue.shade600
      ..style = PaintingStyle.fill;
    for (int i = 0; i < dataPoints.length; i++) {
      final x = i * stepX;
      final normalised = (dataPoints[i] - minVal) / range;
      final y = size.height - normalised * size.height * 0.85 - size.height * 0.075;
      canvas.drawCircle(Offset(x, y), 3, dotPaint);
    }
  }

  @override
  bool shouldRepaint(_ChartBackgroundPainter oldDelegate) =>
      oldDelegate.dataPoints != dataPoints;
}

class _SelectionPainter extends CustomPainter {
  final List<double> dataPoints;
  final int selectedIndex;

  const _SelectionPainter({
    required this.dataPoints,
    required this.selectedIndex,
  });

  @override
  void paint(Canvas canvas, Size size) {
    if (selectedIndex < 0 || dataPoints.isEmpty) return;

    final maxVal = dataPoints.reduce(max);
    final minVal = dataPoints.reduce(min);
    final range = maxVal - minVal;

    final stepX = size.width / (dataPoints.length - 1);
    final x = selectedIndex * stepX;
    final normalised = (dataPoints[selectedIndex] - minVal) / range;
    final y = size.height - normalised * size.height * 0.85 - size.height * 0.075;

    final linePaint = Paint()
      ..color = Colors.red.shade400
      ..strokeWidth = 1.5
      ..style = PaintingStyle.stroke;

    canvas.drawLine(Offset(x, 0), Offset(x, size.height), linePaint);

    // 선택된 포인트 강조
    final outerDot = Paint()
      ..color = Colors.red.shade100
      ..style = PaintingStyle.fill;
    final innerDot = Paint()
      ..color = Colors.red.shade600
      ..style = PaintingStyle.fill;

    canvas.drawCircle(Offset(x, y), 10, outerDot);
    canvas.drawCircle(Offset(x, y), 5, innerDot);
  }

  @override
  bool shouldRepaint(_SelectionPainter oldDelegate) =>
      oldDelegate.selectedIndex != selectedIndex ||
      oldDelegate.dataPoints != dataPoints;
}
```

이 예제의 핵심은 `_ChartBackgroundPainter`와 `_SelectionPainter`를 각각 별도의 `RepaintBoundary`로 감싸는 것입니다. 사용자가 차트를 드래그해도 복잡한 차트 배경(`_ChartBackgroundPainter`)은 GPU 캐시에서 재사용되고, 선택 인디케이터(`_SelectionPainter`)만 다시 그려집니다.

## 디버깅: repaint rainbow와 raster cache checkerboard

### debugRepaintRainbowEnabled로 재렌더링 영역 시각화

```dart
import 'package:flutter/rendering.dart';

void main() {
  // 개발 중에만 활성화 — 프로파일/릴리스 빌드에서는 무시됨
  debugRepaintRainbowEnabled = true;
  runApp(const MyApp());
}
```

이 플래그를 활성화하면 렌더링될 때마다 위젯 경계가 무지개색으로 번갈아 표시됩니다. `RepaintBoundary` 추가 전후를 비교해 최적화 효과를 직관적으로 확인할 수 있습니다. 60fps 화면에서 애니메이션 외 영역이 계속 색이 바뀐다면 불필요한 재렌더링이 발생하고 있다는 신호입니다.

### checkerboardRasterCacheImages와 checkerboardOffscreenLayers

```dart
MaterialApp(
  // 래스터 캐시에 올라간 항목에 체커보드 패턴 표시
  checkerboardRasterCacheImages: true,
  // 오프스크린 레이어에 체커보드 패턴 표시
  checkerboardOffscreenLayers: true,
  home: const MyHomePage(),
)
```

`checkerboardRasterCacheImages`는 GPU 래스터 캐시에 올라간 레이어를 체커보드로 표시합니다. `RepaintBoundary`로 올바르게 격리된 위젯은 여기에 표시되어야 합니다. 반면 `checkerboardOffscreenLayers`는 오프스크린으로 그려지는 레이어(예: `ShaderMask`, `BackdropFilter`, `SaveLayer`)를 보여줍니다. 이런 오프스크린 레이어가 너무 많으면 오히려 성능이 저하됩니다.

### Flutter DevTools로 프레임 타임 측정

```bash
flutter run --profile
```

프로파일 모드에서 DevTools의 **Performance** 탭을 열어 Timeline을 확인합니다. `RepaintBoundary` 추가 전후의 프레임 시간(특히 Raster 스레드 시간)을 비교하면 최적화 효과를 수치로 검증할 수 있습니다. 체감이 아닌 측정으로 증명하는 것이 중요합니다.

## 주의사항과 팁

### 1. 측정 먼저(Measure First) 원칙

`RepaintBoundary`는 GPU 메모리를 추가 소비합니다. 각 경계마다 별도의 오프스크린 텍스처가 생성됩니다. 모든 위젯에 무분별하게 적용하면 메모리 압박으로 오히려 성능이 나빠질 수 있습니다. **항상 DevTools로 측정하고 필요한 곳에만 적용하세요.**

### 2. shouldRepaint를 정확하게 구현

```dart
// 잘못된 예: 항상 true 반환 → RepaintBoundary 효과 없음
@override
bool shouldRepaint(MyPainter old) => true;

// 올바른 예: 실제 데이터 변경 여부 비교
@override
bool shouldRepaint(MyPainter old) =>
    old.color != color || old.dataPoints != dataPoints;
```

`CustomPainter.shouldRepaint`가 항상 `true`를 반환하면 `RepaintBoundary`가 있어도 매 프레임 다시 그려집니다. 실제 의미 있는 변경만 감지하여 반환하세요.

### 3. AnimatedBuilder의 child 파라미터 활용

```dart
AnimatedBuilder(
  animation: animation,
  builder: (context, child) {
    return Transform.rotate(
      angle: animation.value,
      child: child, // ← child는 재빌드되지 않음
    );
  },
  child: const ExpensiveWidget(), // ← 한 번만 빌드됨
)
```

`AnimatedBuilder`의 `child` 파라미터에 넣은 위젯은 애니메이션이 반복되어도 재빌드되지 않습니다. 복잡한 정적 자식이 있다면 이 파라미터를 활용하고, 추가로 `RepaintBoundary`로 감싸면 Paint 단계도 최적화됩니다.

### 4. ListView는 이미 RepaintBoundary를 삽입

`ListView.builder`는 각 항목 사이에 자동으로 `RepaintBoundary`를 삽입합니다. 따라서 `ListView.builder` 항목 위젯을 추가로 `RepaintBoundary`로 감싸면 불필요한 중첩 레이어가 생성됩니다. 직접 `Column` + `SingleChildScrollView`로 목록을 구현할 때는 직접 적용하세요.

### 5. 너무 작은 위젯에는 적용하지 말 것

`RepaintBoundary`는 "그리기 비용"이 "레이어 생성 비용"보다 클 때 효과적입니다. 단순한 텍스트나 아이콘 하나에 적용하면 오히려 오버헤드만 늘어납니다. **복잡한 CustomPainter, 많은 자식이 있는 Column/Row, 무거운 이미지 처리 등에 적용하세요.**

## 정리

`RepaintBoundary`는 Flutter 렌더링 최적화의 핵심 도구입니다. 핵심 원리는 간단합니다. **변경되지 않는 위젯은 GPU 레이어 캐시에서 재사용하고, 자주 변경되는 위젯만 별도 레이어에서 렌더링한다.** 이를 통해:

- 애니메이션이 화면 전체를 재렌더링하는 것을 방지합니다
- `CustomPainter` 차트나 그래픽의 렌더링 비용을 최소화합니다
- 복잡한 UI에서 일관된 60fps/120fps를 유지할 수 있습니다

단, `debugRepaintRainbowEnabled`와 `checkerboardRasterCacheImages`로 실제 렌더링 상황을 시각화하고, DevTools의 Performance 탭으로 수치를 확인한 후 필요한 곳에만 적용하는 것이 올바른 접근법입니다. 측정 없는 최적화는 최적화가 아닙니다.

## 참고 자료
- [RepaintBoundary class - Flutter API 공식 문서](https://api.flutter.dev/flutter/widgets/RepaintBoundary-class.html)
- [PerformanceOverlay.checkerboardRasterCacheImages - Flutter API](https://api.flutter.dev/flutter/widgets/PerformanceOverlay/checkerboardRasterCacheImages.html)
- [Flutter Rendering Optimization Tips - gskinner blog](https://blog.gskinner.com/?p=5994)
- [Understanding Composited Layers - Flutter 성능 심층 분석](https://medium.com/@pomis172/understanding-composited-layers-to-improve-the-performance-of-flutter-apps-7b91079b4dd1)
