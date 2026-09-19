---
layout: post
title: "Flutter Flame 게임 엔진 심화: FlameGame·Component 시스템·Forge2D 물리 엔진으로 2D 모바일 게임 완전 정복"
date: 2026-09-19
categories: [flutter, android]
tags: [flutter, flame, game-engine, forge2d, box2d, component, dart]
---

## 개요

Flutter는 UI 프레임워크로 잘 알려져 있지만, **Flame** 이라는 게임 엔진을 사용하면 복잡한 2D 게임도 Flutter 위에서 구현할 수 있습니다. Flame은 FlutterFavorite으로 선정된 패키지로, 게임 루프·컴포넌트 시스템·충돌 감지·파티클 이펙트·물리 엔진 등 게임 개발에 필요한 핵심 모듈을 제공합니다. 이 글에서는 Flame의 핵심 아키텍처인 Flame Component System(FCS)을 심화 분석하고, Forge2D(Box2D) 물리 엔진과 연동하여 실제 작동하는 2D 게임을 구현하는 방법을 단계적으로 설명합니다.

---

## Flame이란 무엇인가

Flame(v1.38.2)은 Flutter 위에서 동작하는 경량 게임 엔진입니다. Flutter의 `Canvas` API와 `GameWidget`을 기반으로 별도의 게임 루프를 구성하며, 일반 Flutter 위젯과 완전히 공존할 수 있습니다. 이는 Unity나 Godot과 같은 독립 엔진과 달리 **Dart 코드만으로** 게임 로직과 앱 UI를 함께 관리할 수 있다는 의미입니다.

### Flame의 핵심 구성 요소

| 모듈 | 역할 |
|---|---|
| `FlameGame` | 게임 루프와 컴포넌트 트리를 관리하는 최상위 클래스 |
| `Component` | 게임 오브젝트의 기본 단위. `update()`·`render()` 생명주기를 가짐 |
| `PositionComponent` | 위치·크기·회전을 내장한 컴포넌트. 스프라이트·텍스트·도형의 기반 |
| `SpriteComponent` | 이미지 스프라이트를 렌더링하는 컴포넌트 |
| `CollisionDetection` | AABB·원형·다각형 히트박스 기반 충돌 감지 시스템 |
| `flame_forge2d` | Box2D 기반 강체 물리 엔진 연동 |
| `Effects` | MoveEffect·RotateEffect 등 선언형 애니메이션 이펙트 |

---

## 왜 Flutter로 게임을 만드는가

### 장점

1. **단일 코드베이스**: Android·iOS·Web·Desktop 모두 한 번에 빌드됩니다.
2. **Flutter 생태계 통합**: Riverpod·BLoC 등 상태 관리, Firebase, Hive 등 기존 Flutter 패키지를 그대로 사용할 수 있습니다.
3. **핫 리로드**: 게임 파라미터(속도·크기·색상)를 코드로 바꾸면 즉시 반영되어 이터레이션 속도가 빠릅니다.
4. **Dart의 타입 안전성**: 게임 상태를 sealed class·records로 모델링하면 런타임 오류를 컴파일 타임에 잡을 수 있습니다.

### 한계

- 3D 렌더링이나 AAA급 셰이더 파이프라인이 필요한 게임에는 Unity·Unreal이 적합합니다.
- Flame의 물리 엔진(Forge2D)은 Box2D 기반으로 충분하지만, 수천 개의 강체를 동시에 시뮬레이션하는 대규모 물리 씬에는 성능 튜닝이 필요합니다.

---

## 실제 구현 예제 1: FlameGame과 Component 시스템

첫 번째 예제에서는 FlameGame을 설정하고, 터치로 이동하는 플레이어 컴포넌트를 만들며, 자동 생성되는 장애물과의 충돌 감지를 구현합니다.

```dart
// pubspec.yaml에 추가
// dependencies:
//   flame: ^1.38.2
//   flame_forge2d: ^0.20.0

import 'dart:math';
import 'package:flame/components.dart';
import 'package:flame/events.dart';
import 'package:flame/game.dart';
import 'package:flame/collisions.dart';
import 'package:flutter/material.dart';

// ① 게임 진입점 — GameWidget으로 Flutter 앱에 임베드
void main() {
  runApp(
    GameWidget(
      game: DuckGame(),
      overlayBuilderMap: {
        'GameOver': (context, game) => _GameOverOverlay(game: game as DuckGame),
      },
    ),
  );
}

// ② FlameGame 서브클래스 — 게임 루프 + 컴포넌트 트리 관리
class DuckGame extends FlameGame with HasCollisionDetection, TapCallbacks {
  late final PlayerComponent player;
  double _spawnTimer = 0;
  static const _spawnInterval = 1.5; // 초

  @override
  Future<void> onLoad() async {
    // 카메라 뷰포트 고정
    camera.viewfinder.anchor = Anchor.topLeft;

    // 배경 컴포넌트 추가
    add(RectangleComponent(
      size: size,
      paint: Paint()..color = const Color(0xFF1a1a2e),
    ));

    // 플레이어 추가
    player = PlayerComponent()
      ..position = Vector2(size.x / 2, size.y - 100)
      ..anchor = Anchor.center;
    add(player);

    // 점수 HUD
    add(ScoreText());
  }

  @override
  void update(double dt) {
    super.update(dt);
    _spawnTimer += dt;
    if (_spawnTimer >= _spawnInterval) {
      _spawnTimer = 0;
      _spawnObstacle();
    }
  }

  void _spawnObstacle() {
    final x = Random().nextDouble() * (size.x - 40) + 20;
    add(ObstacleComponent()
      ..position = Vector2(x, -30)
      ..anchor = Anchor.center);
  }

  @override
  void onTapDown(TapDownEvent event) {
    // 탭한 x 위치로 플레이어 이동
    player.moveTo(event.localPosition.x);
  }

  void gameOver() {
    pauseEngine();
    overlays.add('GameOver');
  }
}

// ③ 플레이어 컴포넌트 — PositionComponent + CollisionCallbacks
class PlayerComponent extends PositionComponent
    with CollisionCallbacks, HasGameRef<DuckGame> {
  static const _speed = 300.0;
  double? _targetX;

  PlayerComponent() : super(size: Vector2(50, 50));

  @override
  Future<void> onLoad() async {
    // 원형 히트박스 등록
    add(CircleHitbox(radius: 22, anchor: Anchor.center)
      ..position = size / 2);
  }

  @override
  void render(Canvas canvas) {
    // 간단한 원형 플레이어 렌더링 (실제로는 SpriteComponent 사용 권장)
    final paint = Paint()..color = const Color(0xFFe94560);
    canvas.drawCircle(Offset(size.x / 2, size.y / 2), 22, paint);
  }

  @override
  void update(double dt) {
    if (_targetX != null) {
      final dx = _targetX! - position.x;
      if (dx.abs() < 2) {
        _targetX = null;
      } else {
        position.x += dx.sign * _speed * dt;
      }
    }
  }

  void moveTo(double x) => _targetX = x;

  @override
  void onCollisionStart(
    Set<Vector2> intersectionPoints,
    PositionComponent other,
  ) {
    if (other is ObstacleComponent) {
      gameRef.gameOver();
    }
  }
}

// ④ 장애물 컴포넌트 — 위에서 아래로 낙하
class ObstacleComponent extends PositionComponent
    with HasGameRef<DuckGame> {
  static const _fallSpeed = 220.0;

  ObstacleComponent() : super(size: Vector2(40, 40));

  @override
  Future<void> onLoad() async {
    add(RectangleHitbox());
  }

  @override
  void render(Canvas canvas) {
    final paint = Paint()..color = const Color(0xFF16213e);
    canvas.drawRRect(
      RRect.fromRectAndRadius(
        Rect.fromLTWH(0, 0, size.x, size.y),
        const Radius.circular(8),
      ),
      paint,
    );
  }

  @override
  void update(double dt) {
    position.y += _fallSpeed * dt;
    // 화면 아래를 벗어나면 컴포넌트 제거
    if (position.y > gameRef.size.y + 50) removeFromParent();
  }
}

// ⑤ HUD 텍스트 컴포넌트
class ScoreText extends TextComponent with HasGameRef<DuckGame> {
  double _elapsed = 0;

  ScoreText()
      : super(
          text: '0 s',
          textRenderer: TextPaint(
            style: const TextStyle(color: Colors.white, fontSize: 20),
          ),
          anchor: Anchor.topRight,
        );

  @override
  Future<void> onLoad() async {
    position = Vector2(gameRef.size.x - 16, 16);
  }

  @override
  void update(double dt) {
    _elapsed += dt;
    text = '${_elapsed.toStringAsFixed(1)} s';
  }
}

// 게임 오버 오버레이 (일반 Flutter 위젯)
class _GameOverOverlay extends StatelessWidget {
  final DuckGame game;
  const _GameOverOverlay({required this.game});

  @override
  Widget build(BuildContext context) {
    return Center(
      child: Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          const Text('GAME OVER',
              style: TextStyle(color: Colors.white, fontSize: 36)),
          const SizedBox(height: 16),
          ElevatedButton(
            onPressed: () {
              game.overlays.remove('GameOver');
              game.resumeEngine();
            },
            child: const Text('다시 시작'),
          ),
        ],
      ),
    );
  }
}
```

### 핵심 포인트 정리

- **`HasCollisionDetection` mixin**: FlameGame에 추가하면 `CollisionDetection` 시스템이 활성화됩니다. 컴포넌트에 `Hitbox`를 추가하면 자동으로 충돌을 감지합니다.
- **`TapCallbacks` mixin**: Flutter의 `GestureDetector` 대신 Flame 게임 루프와 동기화된 터치 이벤트를 처리합니다.
- **`overlays`**: 게임 위에 Flutter 위젯을 올릴 수 있어, HUD나 메뉴 UI를 일반 Flutter로 구현할 수 있습니다.
- **`removeFromParent()`**: 컴포넌트를 게임 트리에서 안전하게 제거합니다. 직접 리스트를 조작하지 말고 반드시 이 메서드를 사용해야 합니다.

---

## 실제 구현 예제 2: Flame Forge2D로 물리 기반 플랫포머 구현

두 번째 예제에서는 `flame_forge2d`를 사용하여 중력·반발·마찰이 적용된 물리 기반 씬을 구현합니다. `BodyComponent`는 Box2D의 강체(RigidBody)를 Flame Component와 연결하는 다리 역할을 합니다.

```dart
import 'package:flame_forge2d/flame_forge2d.dart';
import 'package:flutter/material.dart';

// ① Forge2DGame 서브클래스
// Forge2DGame은 FlameGame을 확장하며 Box2D World를 내장합니다.
class PhysicsGame extends Forge2DGame {
  // zoom: 물리 좌표계(m)를 화면 픽셀로 변환하는 배율
  PhysicsGame() : super(zoom: 10, gravity: Vector2(0, 30));

  @override
  Future<void> onLoad() async {
    // 바닥 + 벽 생성
    await addAll([
      Ground(position: Vector2(size.x / 2 / 10, size.y / 10)),
      WallLeft(),
      WallRight(rightX: size.x / 10),
    ]);

    // 공 여러 개 생성
    for (int i = 0; i < 5; i++) {
      final x = 5.0 + i * 3.0;
      await add(Ball(position: Vector2(x, 2.0), radius: 0.8));
    }

    // 플랫폼
    await add(Platform(
      position: Vector2(size.x / 20, size.y / 14),
      size: Vector2(8, 0.5),
    ));
  }
}

// ② 구형 강체 — BodyComponent<PhysicsGame>을 상속
class Ball extends BodyComponent<PhysicsGame> {
  final Vector2 _initPos;
  final double radius;

  Ball({required Vector2 position, required this.radius})
      : _initPos = position;

  @override
  Body createBody() {
    // BodyDef: 강체의 타입·위치·속성 정의
    final bodyDef = BodyDef(
      position: _initPos,
      type: BodyType.dynamic, // 중력·충격에 반응
      linearDamping: 0.1,     // 공기 저항
    );

    // FixtureDef: 모양·물리 속성(밀도, 반발계수, 마찰) 정의
    final fixtureDef = FixtureDef(
      CircleShape()..radius = radius,
      density: 1.0,
      restitution: 0.6, // 탄성: 0(완전 비탄성) ~ 1(완전 탄성)
      friction: 0.3,
    );

    return world.createBody(bodyDef)..createFixture(fixtureDef);
  }

  @override
  void render(Canvas canvas) {
    // Forge2D는 body.position이 물리 좌표계이므로
    // render()에서는 로컬 좌표(원점 기준)로 그립니다.
    final paint = Paint()
      ..color = HSVColor.fromAHSV(
        1.0,
        (body.position.x * 30) % 360, // 위치에 따라 색상 변화
        0.8,
        0.9,
      ).toColor();
    canvas.drawCircle(Offset.zero, radius * gameRef.zoom, paint);
  }
}

// ③ 정적 강체 — 바닥
class Ground extends BodyComponent<PhysicsGame> {
  final Vector2 position;
  Ground({required this.position});

  @override
  Body createBody() {
    final bodyDef = BodyDef(position: position, type: BodyType.static);
    final shape = EdgeShape()
      ..set(Vector2(-50, 0), Vector2(50, 0)); // 수평 선분
    return world.createBody(bodyDef)
      ..createFixture(FixtureDef(shape, friction: 0.5));
  }
}

// ④ 벽 (정적)
class WallLeft extends BodyComponent<PhysicsGame> {
  @override
  Body createBody() {
    final shape = EdgeShape()..set(Vector2(0, -50), Vector2(0, 50));
    return world.createBody(BodyDef(type: BodyType.static))
      ..createFixture(FixtureDef(shape));
  }
}

class WallRight extends BodyComponent<PhysicsGame> {
  final double rightX;
  WallRight({required this.rightX});

  @override
  Body createBody() {
    final shape = EdgeShape()..set(Vector2(rightX, -50), Vector2(rightX, 50));
    return world.createBody(BodyDef(type: BodyType.static))
      ..createFixture(FixtureDef(shape));
  }
}

// ⑤ 키네마틱 강체 — 물리에 반응하지 않지만 직접 속도 제어 가능
// 움직이는 플랫폼에 활용
class Platform extends BodyComponent<PhysicsGame> {
  final Vector2 position;
  final Vector2 size;
  double _dir = 1;

  Platform({required this.position, required this.size});

  @override
  Body createBody() {
    final bodyDef = BodyDef(
      position: position,
      type: BodyType.kinematic, // 물리 반응 없이 직접 조작
    );
    final shape = PolygonShape()
      ..setAsBoxXY(size.x / 2, size.y / 2);
    return world.createBody(bodyDef)
      ..createFixture(FixtureDef(shape, friction: 0.8));
  }

  @override
  void update(double dt) {
    super.update(dt);
    // 좌우 왕복 운동
    body.linearVelocity = Vector2(3.0 * _dir, 0);
    if (body.position.x > 12 || body.position.x < 3) {
      _dir *= -1;
    }
  }

  @override
  void render(Canvas canvas) {
    final paint = Paint()..color = const Color(0xFF0f3460);
    canvas.drawRect(
      Rect.fromCenter(
        center: Offset.zero,
        width: size.x * gameRef.zoom,
        height: size.y * gameRef.zoom,
      ),
      paint,
    );
  }
}

// 앱 진입점
void main() {
  runApp(GameWidget(game: PhysicsGame()));
}
```

### Forge2D 핵심 개념

**BodyType 비교**

| BodyType | 중력·충격 반응 | 직접 이동 | 주요 용도 |
|---|---|---|---|
| `dynamic` | O | 힘·충격으로 | 플레이어, 적, 발사체 |
| `static` | X | 불가 | 바닥, 벽, 고정 발판 |
| `kinematic` | X | 속도 직접 설정 | 움직이는 발판, 엘리베이터 |

**좌표계 주의**: Forge2D의 물리 좌표계는 미터(m) 단위입니다. `zoom` 파라미터(기본값 10)가 1m를 10픽셀로 변환합니다. `render()` 내부의 `Canvas`는 이미 `zoom` 배율로 스케일된 로컬 좌표계이므로, 직접 `gameRef.zoom`을 다시 곱하지 않도록 주의해야 합니다.

---

## 고급 패턴: Effects와 ParticleSystem

Flame의 Effect 시스템을 사용하면 코드 한 줄로 컴포넌트에 애니메이션을 추가할 수 있습니다.

```dart
// 컴포넌트가 충돌했을 때 흔들림(Shake) + 제거 이펙트
void onHit() {
  add(
    SequenceEffect([
      MoveEffect.by(
        Vector2(5, 0),
        EffectController(duration: 0.05, reverseDuration: 0.05, repeatCount: 3),
      ),
      RemoveEffect(), // 이펙트 완료 후 자동 제거
    ]),
  );
}

// 파티클 폭발 이펙트
void spawnExplosion(Vector2 position) {
  gameRef.add(
    ParticleSystemComponent(
      position: position,
      particle: Particle.generate(
        count: 40,
        lifespan: 0.8,
        generator: (i) => AcceleratedParticle(
          speed: Vector2.random() * 150 - Vector2.all(75),
          acceleration: Vector2(0, 100), // 중력
          child: CircleParticle(
            radius: 3,
            paint: Paint()..color = Colors.orange.withOpacity(0.8),
          ),
        ),
      ),
    ),
  );
}
```

---

## 주의사항과 실전 팁

### 1. 컴포넌트 생명주기를 철저히 준수

`onLoad()`는 비동기로 실행됩니다. 스프라이트나 오디오 같은 에셋 로딩을 `onLoad()` 안에서 `await`해야 합니다. 생성자에서 `gameRef`를 사용하면 `null` 참조 오류가 발생합니다.

```dart
// 잘못된 예
class MyComponent extends PositionComponent with HasGameRef {
  MyComponent() {
    size = gameRef.size; // gameRef가 아직 null!
  }
}

// 올바른 예
class MyComponent extends PositionComponent with HasGameRef {
  @override
  Future<void> onLoad() async {
    size = gameRef.size; // onLoad에서 안전하게 접근
    await add(SpriteComponent(sprite: await gameRef.loadSprite('duck.png')));
  }
}
```

### 2. Forge2D에서 body.position 직접 수정 금지

물리 시뮬레이션 중에 `body.position`을 직접 변경하면 충돌 감지가 꼬입니다. 위치를 강제로 변경해야 할 경우 `body.setTransform()`을, 힘을 가하려면 `body.applyLinearImpulse()` 또는 `body.applyForce()`를 사용합니다.

### 3. 에셋 캐싱과 메모리 관리

Flame은 `Images.load()`, `Audio.load()` 결과를 내부적으로 캐싱합니다. 씬 전환 시 필요 없는 에셋은 `game.images.clearCache()`로 해제하여 메모리 누수를 방지하세요.

### 4. 개발 도구 활용

`FlameGame`에 `HasPerformanceTracker` mixin을 추가하면 FPS·업데이트 시간·렌더 시간을 실시간으로 확인할 수 있습니다. 디버그 빌드에서만 활성화하는 패턴을 권장합니다.

```dart
class MyGame extends FlameGame with HasPerformanceTracker {
  @override
  void render(Canvas canvas) {
    super.render(canvas);
    if (kDebugMode) {
      // FPS 오버레이 표시
      final fps = performanceTracker.fps.toStringAsFixed(1);
      debugPrint('FPS: $fps');
    }
  }
}
```

### 5. 화면 크기 변경 대응

태블릿·폴더블 기기에서 화면 비율이 바뀔 때 `onGameResize()`를 오버라이드하여 UI 컴포넌트 위치를 재계산합니다.

```dart
@override
void onGameResize(Vector2 newSize) {
  super.onGameResize(newSize);
  // HUD 컴포넌트 위치 재배치
  scoreText.position = Vector2(newSize.x - 16, 16);
}
```

---

## 마치며

Flame은 Flutter의 생산성과 Dart의 타입 안전성을 게임 개발 영역에 그대로 가져오는 강력한 선택입니다. 간단한 캐주얼 게임부터 Forge2D 물리 엔진을 활용한 플랫포머까지, 싱글 코드베이스로 Android·iOS·Web 전부를 지원할 수 있습니다. 특히 Flutter 앱에 게임 요소(미니게임, 인터랙티브 튜토리얼, 가챠 애니메이션)를 추가하는 시나리오에서 Flame은 매우 현실적인 솔루션입니다.

## 참고 자료
- [Flame 공식 패키지 (pub.dev)](https://pub.dev/packages/flame)
- [flame_forge2d 패키지 (pub.dev)](https://pub.dev/packages/flame_forge2d)
- [Flame GitHub 저장소](https://github.com/flame-engine/flame)
