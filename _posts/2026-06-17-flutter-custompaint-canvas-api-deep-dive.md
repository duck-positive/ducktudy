---
layout: post
title: "Flutter CustomPainter와 Canvas API 심화: 픽셀 단위로 UI를 직접 그리는 기술"
date: 2026-06-17
categories: [android, flutter]
tags: [flutter, dart, custompaint, canvas, animation, performance]
---

Flutter의 위젯 시스템은 대부분의 UI 요구 사항을 충족하지만, 복잡한 그래픽이나 커스텀 시각화는 기본 위젯만으로 구현하기 어렵습니다. `CustomPainter`와 `Canvas API`는 이 한계를 돌파하는 Flutter의 저수준 드로잉 시스템입니다.

## 1. CustomPainter란 무엇인가?

`CustomPainter`는 Flutter에서 저수준(low-level) 그래픽을 직접 제어할 수 있게 해주는 추상 클래스입니다. 개발자는 이 클래스를 상속받아 `paint()`와 `shouldRepaint()` 두 메서드를 구현하고, `CustomPaint` 위젯과 함께 사용합니다.

`Canvas`는 실제 드로잉이 이루어지는 표면으로, HTML5의 `<canvas>` 요소나 Android의 `Canvas`와 유사한 개념입니다. 좌표계는 좌상단이 `(0, 0)`이며, X축은 오른쪽, Y축은 아래쪽 방향으로 증가합니다.

### CustomPaint 위젯 구조

```dart
CustomPaint(
  painter: MyCustomPainter(),         // 자식 위젯 아래에 그리기 (배경)
  foregroundPainter: MyFgPainter(),   // 자식 위젯 위에 그리기 (전경)
  size: const Size(300, 200),         // child가 없을 때 크기 지정
  child: SomeWidget(),                // 선택적 자식 위젯
)
```

## 2. 왜 CustomPainter가 필요한가?

Flutter의 기본 위젯(`Container`, `DecoratedBox`, `ClipPath`)으로 구현하기 어려운 경우가 있습니다.

### 2.1 기본 위젯으로 구현 불가능한 UI

- **차트 및 그래프**: 라인 차트, 바 차트, 파이 차트처럼 동적 데이터를 시각화하는 경우
- **게임 UI**: 체력 바, 미니맵, 파티클 효과
- **커스텀 프로그레스 인디케이터**: 원형이나 곡선 경로를 따라가는 진행 표시기
- **파형(Waveform) 시각화**: 오디오 재생 시 실시간 파형 표시
- **복잡한 클리핑 경로**: 단순 `RoundedRectangle`을 넘어서는 커스텀 형태

### 2.2 성능 최적화

`shouldRepaint()` 메서드를 통해 불필요한 재렌더링을 방지할 수 있습니다. `repaint` 리스너를 사용하면 Flutter 위젯 트리 빌드를 생략하고 Canvas 레이어만 독립적으로 갱신할 수 있어 애니메이션 성능이 대폭 향상됩니다.

### 2.3 픽셀 단위의 정밀 제어

Canvas API를 통해 복잡한 베지어 곡선, 그라디언트, 마스크, 이미지 합성 등 고급 그래픽 효과를 픽셀 수준에서 제어할 수 있습니다.

## 3. Canvas API 핵심 메서드

| 메서드 | 설명 |
|--------|------|
| `drawLine()` | 두 점을 잇는 직선 |
| `drawRect()` | 사각형 |
| `drawRRect()` | 모서리가 둥근 사각형 |
| `drawCircle()` | 원 |
| `drawArc()` | 호(arc) |
| `drawPath()` | `Path` 객체를 이용한 복합 도형 |
| `drawImage()` | 이미지 |
| `drawParagraph()` | 텍스트 단락 |
| `save()` / `restore()` | 캔버스 상태 저장/복원 |
| `translate()` / `rotate()` / `scale()` | 변환(transform) |
| `clipPath()` / `clipRect()` | 클리핑 영역 설정 |

### Paint 클래스

모든 드로잉 메서드에 `Paint` 객체를 전달하여 스타일을 지정합니다.

```dart
final paint = Paint()
  ..color = Colors.blue
  ..strokeWidth = 2.0
  ..style = PaintingStyle.stroke  // PaintingStyle.fill 로 변경하면 채우기
  ..strokeCap = StrokeCap.round
  ..isAntiAlias = true;
```

## 4. 실제 구현 예제 1: 애니메이션 원형 프로그레스 인디케이터

그라디언트와 텍스트가 포함된 원형 프로그레스를 `CustomPainter`로 구현합니다.

```dart
import 'dart:math' as math;
import 'package:flutter/material.dart';

class CircularProgressPainter extends CustomPainter {
  final double progress; // 0.0 ~ 1.0
  final Color backgroundColor;
  final Color progressColor;
  final double strokeWidth;

  CircularProgressPainter({
    required this.progress,
    this.backgroundColor = const Color(0xFFE0E0E0),
    this.progressColor = Colors.blue,
    this.strokeWidth = 8.0,
  });

  @override
  void paint(Canvas canvas, Size size) {
    final center = Offset(size.width / 2, size.height / 2);
    final radius = (size.shortestSide / 2) - strokeWidth / 2;

    // 배경 원
    final bgPaint = Paint()
      ..color = backgroundColor
      ..style = PaintingStyle.stroke
      ..strokeWidth = strokeWidth
      ..strokeCap = StrokeCap.round;
    canvas.drawCircle(center, radius, bgPaint);

    // 진행 호(arc) — SweepGradient 적용
    final progressPaint = Paint()
      ..style = PaintingStyle.stroke
      ..strokeWidth = strokeWidth
      ..strokeCap = StrokeCap.round
      ..shader = SweepGradient(
        startAngle: -math.pi / 2,
        endAngle: -math.pi / 2 + 2 * math.pi * progress,
        colors: [progressColor.withOpacity(0.4), progressColor],
      ).createShader(Rect.fromCircle(center: center, radius: radius));

    canvas.drawArc(
      Rect.fromCircle(center: center, radius: radius),
      -math.pi / 2,              // 12시 방향에서 시작
      2 * math.pi * progress,    // 진행률만큼 호를 그림
      false,
      progressPaint,
    );

    // 퍼센트 텍스트
    final textSpan = TextSpan(
      text: '${(progress * 100).toInt()}%',
      style: TextStyle(
        color: progressColor,
        fontSize: size.shortestSide * 0.2,
        fontWeight: FontWeight.bold,
      ),
    );
    final tp = TextPainter(text: textSpan, textDirection: TextDirection.ltr)
      ..layout();
    tp.paint(
      canvas,
      Offset(center.dx - tp.width / 2, center.dy - tp.height / 2),
    );
  }

  @override
  bool shouldRepaint(CircularProgressPainter old) =>
      old.progress != progress ||
      old.progressColor != progressColor ||
      old.strokeWidth != strokeWidth;
}

// AnimationController와 결합한 위젯
class AnimatedCircularProgress extends StatefulWidget {
  const AnimatedCircularProgress({super.key});

  @override
  State<AnimatedCircularProgress> createState() =>
      _AnimatedCircularProgressState();
}

class _AnimatedCircularProgressState extends State<AnimatedCircularProgress>
    with SingleTickerProviderStateMixin {
  late final AnimationController _controller;
  late final Animation<double> _animation;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(seconds: 2),
    );
    _animation = Tween<double>(begin: 0.0, end: 0.75).animate(
      CurvedAnimation(parent: _controller, curve: Curves.easeInOut),
    );
    _controller.forward();
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: _animation,
      builder: (context, _) => CustomPaint(
        painter: CircularProgressPainter(
          progress: _animation.value,
          progressColor: Colors.deepPurple,
          strokeWidth: 12.0,
        ),
        size: const Size(200, 200),
      ),
    );
  }
}
```

**핵심 포인트:**
- `shouldRepaint()`에서 변경된 속성만 비교하여 불필요한 재렌더링 방지
- `SweepGradient`를 `Paint.shader`에 연결하여 그라디언트 호 구현
- `TextPainter.layout()` 호출 후 `paint()`로 캔버스에 텍스트 렌더링

## 5. 실제 구현 예제 2: 인터랙티브 라인 차트

`GestureDetector`와 `CustomPainter`를 결합하여 터치 지점에 툴팁이 표시되는 라인 차트를 구현합니다.

```dart
import 'dart:math' as math;
import 'package:flutter/material.dart';

class LineChartPainter extends CustomPainter {
  final List<double> dataPoints;
  final double? selectedX;
  final Color lineColor;
  final Color fillColor;

  LineChartPainter({
    required this.dataPoints,
    this.selectedX,
    this.lineColor = Colors.blue,
    Color? fillColor,
  }) : fillColor = fillColor ?? Colors.blue.withOpacity(0.1);

  // 데이터 값을 캔버스 좌표로 변환
  Offset _toCanvas(Size size, int index, double value, double minY, double rangeY) {
    final x = size.width * index / (dataPoints.length - 1);
    final y = size.height * 0.9 * (1 - (value - minY) / rangeY) + size.height * 0.05;
    return Offset(x, y);
  }

  @override
  void paint(Canvas canvas, Size size) {
    if (dataPoints.length < 2) return;

    final maxY = dataPoints.reduce(math.max);
    final minY = dataPoints.reduce(math.min);
    final rangeY = (maxY - minY).abs() < 1e-9 ? 1.0 : maxY - minY;

    // --- 1. 채우기 영역 (cubic bezier 스무딩) ---
    final fillPath = Path()..moveTo(0, size.height);
    for (int i = 0; i < dataPoints.length; i++) {
      final pt = _toCanvas(size, i, dataPoints[i], minY, rangeY);
      if (i == 0) {
        fillPath.lineTo(pt.dx, pt.dy);
      } else {
        final prev = _toCanvas(size, i - 1, dataPoints[i - 1], minY, rangeY);
        final cpX = (prev.dx + pt.dx) / 2;
        fillPath.cubicTo(cpX, prev.dy, cpX, pt.dy, pt.dx, pt.dy);
      }
    }
    fillPath.lineTo(size.width, size.height);
    fillPath.close();
    canvas.drawPath(fillPath, Paint()..color = fillColor);

    // --- 2. 라인 ---
    final linePath = Path();
    for (int i = 0; i < dataPoints.length; i++) {
      final pt = _toCanvas(size, i, dataPoints[i], minY, rangeY);
      if (i == 0) {
        linePath.moveTo(pt.dx, pt.dy);
      } else {
        final prev = _toCanvas(size, i - 1, dataPoints[i - 1], minY, rangeY);
        final cpX = (prev.dx + pt.dx) / 2;
        linePath.cubicTo(cpX, prev.dy, cpX, pt.dy, pt.dx, pt.dy);
      }
    }
    canvas.drawPath(
      linePath,
      Paint()
        ..color = lineColor
        ..style = PaintingStyle.stroke
        ..strokeWidth = 2.5
        ..strokeJoin = StrokeJoin.round,
    );

    // --- 3. 선택 포인트 및 툴팁 ---
    if (selectedX != null) {
      final idx = (selectedX! / size.width * (dataPoints.length - 1))
          .round()
          .clamp(0, dataPoints.length - 1);
      final pt = _toCanvas(size, idx, dataPoints[idx], minY, rangeY);

      // 세로 안내선
      canvas.drawLine(
        Offset(pt.dx, 0),
        Offset(pt.dx, size.height),
        Paint()
          ..color = lineColor.withOpacity(0.3)
          ..strokeWidth = 1.0,
      );

      // 강조 점 (흰 테두리 + 색상 원)
      canvas.drawCircle(pt, 6, Paint()..color = Colors.white);
      canvas.drawCircle(
        pt,
        6,
        Paint()
          ..color = lineColor
          ..style = PaintingStyle.stroke
          ..strokeWidth = 2,
      );

      _drawTooltip(canvas, size, pt, dataPoints[idx]);
    }
  }

  void _drawTooltip(Canvas canvas, Size size, Offset point, double value) {
    const tw = 70.0, th = 32.0;
    double tx = (point.dx - tw / 2).clamp(0.0, size.width - tw);
    double ty = point.dy - th - 12;
    if (ty < 0) ty = point.dy + 12;

    canvas.drawRRect(
      RRect.fromRectAndRadius(
          Rect.fromLTWH(tx, ty, tw, th), const Radius.circular(6)),
      Paint()..color = Colors.black87,
    );

    final tp = TextPainter(
      text: TextSpan(
        text: value.toStringAsFixed(1),
        style: const TextStyle(
            color: Colors.white, fontSize: 13, fontWeight: FontWeight.bold),
      ),
      textDirection: TextDirection.ltr,
    )..layout();
    tp.paint(canvas, Offset(tx + (tw - tp.width) / 2, ty + (th - tp.height) / 2));
  }

  @override
  bool shouldRepaint(LineChartPainter old) =>
      old.dataPoints != dataPoints || old.selectedX != selectedX;
}

// 인터랙티브 래퍼
class InteractiveLineChart extends StatefulWidget {
  final List<double> dataPoints;
  const InteractiveLineChart({super.key, required this.dataPoints});

  @override
  State<InteractiveLineChart> createState() => _InteractiveLineChartState();
}

class _InteractiveLineChartState extends State<InteractiveLineChart> {
  double? _selectedX;

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onPanUpdate: (d) => setState(() => _selectedX = d.localPosition.dx),
      onPanEnd: (_) => setState(() => _selectedX = null),
      child: CustomPaint(
        painter: LineChartPainter(
          dataPoints: widget.dataPoints,
          selectedX: _selectedX,
        ),
        size: const Size(double.infinity, 200),
      ),
    );
  }
}
```

## 6. 성능 최적화: shouldRepaint와 repaint 리스너

### 6.1 shouldRepaint() 올바르게 구현하기

```dart
// 나쁜 예: 항상 true → 매 빌드마다 재렌더링
@override
bool shouldRepaint(MyPainter old) => true;

// 나쁜 예: 항상 false → 데이터 변경이 화면에 반영 안 됨
@override
bool shouldRepaint(MyPainter old) => false;

// 좋은 예: 실제 변경된 속성만 비교
@override
bool shouldRepaint(MyPainter old) =>
    old.progress != progress || old.color != color;
```

### 6.2 repaint 리스너로 위젯 빌드 우회

`CustomPainter` 생성자의 `repaint` 파라미터에 `Listenable`(주로 `AnimationController`)을 전달하면, 해당 리스너가 변경될 때만 `paint()`가 호출됩니다. 위젯 트리 빌드(`setState`)를 전혀 거치지 않으므로 고주사율 애니메이션에 매우 효율적입니다.

```dart
class AnimatedWavePainter extends CustomPainter {
  final Animation<double> animation;

  // repaint에 animation을 넘기면 animation.value 변경 시 자동으로 paint() 호출
  AnimatedWavePainter({required this.animation})
      : super(repaint: animation);

  @override
  void paint(Canvas canvas, Size size) {
    final path = Path();
    for (double x = 0; x <= size.width; x++) {
      final y = size.height / 2 +
          20 *
              math.sin(
                (x / size.width * 2 * math.pi) +
                    animation.value * 2 * math.pi,
              );
      x == 0 ? path.moveTo(x, y) : path.lineTo(x, y);
    }
    canvas.drawPath(
      path,
      Paint()
        ..color = Colors.teal
        ..strokeWidth = 2
        ..style = PaintingStyle.stroke,
    );
  }

  // repaint 리스너가 담당하므로 항상 false
  @override
  bool shouldRepaint(AnimatedWavePainter old) => false;
}
```

## 7. 고급 기법: Path, Clip, save/restore

### 7.1 Path로 복잡한 도형 만들기

```dart
// 별(Star) 모양 Path 생성
Path createStarPath(Offset center, double outerR, double innerR, int points) {
  final path = Path();
  final step = math.pi / points;
  for (int i = 0; i < points * 2; i++) {
    final r = i.isEven ? outerR : innerR;
    final a = i * step - math.pi / 2;
    final pt = Offset(center.dx + r * math.cos(a), center.dy + r * math.sin(a));
    i == 0 ? path.moveTo(pt.dx, pt.dy) : path.lineTo(pt.dx, pt.dy);
  }
  return path..close();
}
```

### 7.2 save()/restore()로 변환 격리

```dart
@override
void paint(Canvas canvas, Size size) {
  canvas.save();                              // 현재 변환 상태 저장
  canvas.translate(size.width / 2, size.height / 2);
  canvas.rotate(math.pi / 4);               // 45도 회전
  canvas.drawRect(
    const Rect.fromLTWH(-50, -50, 100, 100),
    Paint()..color = Colors.red,
  );
  canvas.restore();                           // 변환 이전 상태로 복원

  // 이후 드로잉은 원래 좌표계 기준
  canvas.drawCircle(
    Offset(size.width / 2, size.height / 2),
    20,
    Paint()..color = Colors.blue,
  );
}
```

### 7.3 clipPath로 이미지 클리핑

```dart
canvas.save();
canvas.clipPath(createStarPath(center, 100, 50, 5));
// 이후 모든 드로잉은 별 모양 안에만 표시
canvas.drawImage(image, Offset.zero, Paint());
canvas.restore();
```

## 8. 주의사항 및 팁

### 8.1 paint() 내부에서 객체 생성 최소화

`paint()`는 매 프레임(최대 120fps)마다 호출될 수 있습니다. `Paint()`, `Path()` 등을 매번 새로 생성하면 GC 압력이 높아져 프레임 드롭이 발생합니다.

```dart
// 나쁜 예: 매 프레임 객체 생성
@override
void paint(Canvas canvas, Size size) {
  canvas.drawCircle(center, 50, Paint()..color = Colors.red);
}

// 좋은 예: 필드로 선언하여 재사용
final _paint = Paint()..color = Colors.red;

@override
void paint(Canvas canvas, Size size) {
  canvas.drawCircle(center, 50, _paint);
}
```

### 8.2 isComplex / willChange 힌트

```dart
CustomPaint(
  isComplex: true,   // 복잡한 그래픽: 레스터 캐시 활성화 힌트
  willChange: false, // 정적 그래픽은 false, 매 프레임 변경되면 true
  painter: MyStaticPainter(),
)
```

`isComplex: true` + `willChange: false` 조합은 정적 복잡 UI에 레스터 캐시를 적용하여 재렌더링 비용을 줄입니다. 반대로 애니메이션처럼 매 프레임 변하는 경우에는 `willChange: true`로 설정하여 캐시 비용을 피하세요.

### 8.3 hitTest() 오버라이드

기본적으로 `CustomPaint`는 자식 영역에만 히트 테스트를 적용합니다. 그린 도형에 터치 이벤트를 받으려면 `hitTest()`를 오버라이드하세요.

```dart
@override
bool? hitTest(Offset position) {
  // 반경 50 원 안만 히트로 판정
  return (position - const Offset(100, 100)).distance <= 50;
}
```

### 8.4 TextPainter 사용 시 layout() 선행 필수

```dart
final tp = TextPainter(
  text: const TextSpan(text: 'Hello', style: TextStyle(fontSize: 16)),
  textDirection: TextDirection.ltr,
)..layout(maxWidth: 200); // layout() 없이 paint() 호출 시 예외 발생

tp.paint(canvas, Offset(x, y));
```

### 8.5 RepaintBoundary로 레이어 분리

`CustomPaint` 위젯을 `RepaintBoundary`로 감싸면 해당 영역이 독립적인 레이어로 분리되어, 다른 위젯이 리빌드되더라도 이 영역은 재렌더링되지 않습니다.

```dart
RepaintBoundary(
  child: CustomPaint(
    painter: HeavyPainter(),
    size: const Size(300, 300),
  ),
)
```

## 9. 마치며

`CustomPainter`와 `Canvas API`는 Flutter에서 표현력의 한계를 허무는 도구입니다. 단순 도형부터 실시간 애니메이션 차트, 인터랙티브 게임 UI까지, 기본 위젯으로는 불가능한 모든 그래픽을 구현할 수 있습니다.

성능을 유지하는 핵심 원칙은 세 가지입니다.

1. **`shouldRepaint()`를 정교하게 구현**하여 불필요한 재렌더링 방지
2. **`repaint` 리스너**를 활용해 위젯 빌드를 우회한 Canvas 단독 업데이트
3. **`paint()` 내 객체 생성 최소화**로 GC 압력 절감

이 세 가지를 지키면서 구현한 `CustomPainter`는 60fps는 물론 120fps 환경에서도 안정적으로 동작합니다.

## 참고 자료
- [Flutter 공식 API 문서 — CustomPainter class](https://api.flutter.dev/flutter/rendering/CustomPainter-class.html)
- [Flutter 공식 API 문서 — Canvas class](https://api.flutter.dev/flutter/dart-ui/Canvas-class.html)
- [A Deep Dive Into CustomPaint in Flutter — Flutter Community](https://medium.com/flutter-community/a-deep-dive-into-custompaint-in-flutter-47ab44e3f216)
- [Flutter Custom Painters and Advanced Graphics: Deep Dive](https://dasroot.net/posts/2026/01/flutter-custom-painters-advanced-graphics-deep-dive/)
