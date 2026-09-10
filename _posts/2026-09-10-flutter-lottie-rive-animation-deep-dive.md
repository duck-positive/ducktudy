---
layout: post
title: "Flutter Lottie & Rive 애니메이션 심화: After Effects 익스포트부터 인터랙티브 상태 머신까지 완전 정복"
date: 2026-09-10
categories: [android, flutter]
tags: [flutter, lottie, rive, animation, dart, ui, interactive, state-machine]
---

모바일 앱에서 애니메이션은 단순한 시각적 장식이 아닙니다. 사용자가 버튼을 눌렀을 때 즉각적인 피드백을 제공하고, 화면 전환에서 문맥을 유지시켜 주며, 로딩 상태를 덜 지루하게 만들어 주는 핵심 UX 요소입니다. Flutter에는 `AnimationController`와 `Tween`으로 직접 구현하는 방법도 있지만, 디자이너가 After Effects나 Rive Editor에서 만든 복잡한 벡터 애니메이션을 코드로 재구현하는 것은 비현실적입니다.

이 아티클에서는 두 가지 업계 표준 솔루션인 **Lottie**와 **Rive**를 심층적으로 다룹니다. 단순한 "패키지 추가하고 파일 로드" 수준을 넘어, `AnimationController`와의 연동, `ValueDelegate`를 이용한 런타임 속성 변경, Rive의 StateMachine과 SMIInput을 통한 인터랙티브 애니메이션까지 실무에서 바로 쓸 수 있는 패턴을 다룹니다.

## Lottie vs Rive: 언제 무엇을 쓸까?

두 도구는 목적 자체가 다릅니다.

| 구분 | Lottie | Rive |
|------|--------|------|
| 원본 도구 | Adobe After Effects + Bodymovin | Rive Editor |
| 파일 형식 | `.json`, `.lottie` (DotLottie) | `.riv` |
| 상호작용 | 제한적 (재생/정지/속도 조절) | 풍부함 (StateMachine, Input) |
| 주요 용도 | 일회성 재생 애니메이션, 로딩, 아이콘 전환 | 버튼 상태, 게임 캐릭터, 복잡한 UI 인터랙션 |
| 런타임 크기 | 작음 | 중간 (자체 렌더러 포함 가능) |

Lottie는 **"디자이너가 만든 걸 그대로 재생"** 하는 데 최적화되어 있고, Rive는 **"앱 코드와 애니메이션이 실시간으로 대화"** 하는 데 강점이 있습니다.

## 왜 두 가지 모두 알아야 하는가?

실무에서는 두 가지를 동시에 사용하는 경우가 많습니다. 스플래시 화면 인트로나 빈 상태(empty state) 일러스트에는 Lottie를 쓰고, 결제 완료 버튼이나 좋아요 토글처럼 사용자 입력에 반응하는 애니메이션에는 Rive를 씁니다.

또한 Flutter의 `AnimationController`와의 연동 없이 패키지를 사용하는 개발자가 많은데, 이 방식은 유연성이 크게 떨어집니다. 제대로 이해하면 스크롤에 따라 애니메이션을 제어하거나, 특정 프레임 구간만 반복하는 등 다양한 응용이 가능합니다.

## 1. Lottie 심화

### 설치 및 기본 설정

`pubspec.yaml`에 패키지를 추가합니다. 이 글 작성 시점의 최신 안정 버전은 3.5.1입니다.

```yaml
dependencies:
  lottie: ^3.5.1
```

After Effects에서 Bodymovin 플러그인으로 익스포트한 JSON 파일을 `assets/animations/` 폴더에 넣고 `pubspec.yaml`에 등록합니다.

```yaml
flutter:
  assets:
    - assets/animations/
```

### 기본 사용: 자동 재생

가장 단순한 사용법입니다. `Lottie.asset()`은 위젯을 빌드하는 즉시 애니메이션을 시작하고 기본적으로 반복 재생합니다.

```dart
import 'package:flutter/material.dart';
import 'package:lottie/lottie.dart';

class SplashAnimation extends StatelessWidget {
  const SplashAnimation({super.key});

  @override
  Widget build(BuildContext context) {
    return Center(
      child: Lottie.asset(
        'assets/animations/splash.json',
        width: 200,
        height: 200,
        fit: BoxFit.contain,
        repeat: true,
        animate: true,
      ),
    );
  }
}
```

`repeat: false`로 설정하면 한 번만 재생하고 멈춥니다. `animate: false`는 첫 번째 프레임에서 정지 상태로 둡니다.

### AnimationController와 연동: 세밀한 제어

`Lottie.asset()`의 `controller` 파라미터에 `AnimationController`를 넘기면 재생을 완전히 제어할 수 있습니다. 단, `onLoaded` 콜백에서 `composition.duration`을 컨트롤러의 `duration`에 할당해야 정상 속도로 재생됩니다.

```dart
import 'package:flutter/material.dart';
import 'package:lottie/lottie.dart';

class ControlledLottie extends StatefulWidget {
  const ControlledLottie({super.key});

  @override
  State<ControlledLottie> createState() => _ControlledLottieState();
}

class _ControlledLottieState extends State<ControlledLottie>
    with SingleTickerProviderStateMixin {
  late final AnimationController _controller;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(vsync: this);
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  void _playOnce() {
    _controller.reset();
    _controller.forward();
  }

  void _playLoop() {
    _controller.repeat();
  }

  void _scrubTo(double progress) {
    // progress: 0.0 ~ 1.0
    _controller.value = progress.clamp(0.0, 1.0);
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      mainAxisAlignment: MainAxisAlignment.center,
      children: [
        Lottie.asset(
          'assets/animations/check.json',
          controller: _controller,
          onLoaded: (composition) {
            // 이 콜백 없이는 duration이 0이 되어 애니메이션이 재생되지 않습니다.
            _controller.duration = composition.duration;
          },
          width: 200,
          height: 200,
        ),
        const SizedBox(height: 24),
        Row(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            ElevatedButton(
              onPressed: _playOnce,
              child: const Text('한 번 재생'),
            ),
            const SizedBox(width: 12),
            ElevatedButton(
              onPressed: _playLoop,
              child: const Text('반복 재생'),
            ),
          ],
        ),
        Slider(
          value: _controller.value,
          onChanged: _scrubTo,
        ),
      ],
    );
  }
}
```

`_controller.value`는 0.0부터 1.0 사이의 진행률을 나타냅니다. 이를 이용하면 스크롤 위치에 따라 애니메이션을 스크러빙하는 것도 가능합니다.

### ValueDelegate로 런타임 색상/텍스트 변경

Lottie의 강력한 기능 중 하나는 `ValueDelegate`를 통해 애니메이션의 특정 레이어 속성을 Flutter 코드에서 동적으로 변경할 수 있다는 것입니다. After Effects 레이어 이름으로 접근합니다.

```dart
Lottie.asset(
  'assets/animations/heart.json',
  delegates: LottieDelegates(
    values: [
      // 레이어 이름 "Heart Fill" > "Fill 1" > "Color" 속성을 런타임에 변경
      ValueDelegate.color(
        const ['Heart Fill', 'Fill 1', '**'],
        value: Colors.red,
      ),
      // "Heart Outline" 레이어의 투명도
      ValueDelegate.opacity(
        const ['Heart Outline', '**'],
        value: 80, // 0 ~ 100
      ),
    ],
  ),
)
```

경로는 After Effects의 레이어 계층 구조를 따르며, `'**'`는 와일드카드입니다. 이 기능을 활용하면 하나의 Lottie 파일로 다크/라이트 테마 대응이나 사용자 지정 색상 브랜딩을 구현할 수 있습니다.

### 성능 최적화: renderCache

복잡한 Lottie 애니메이션은 CPU와 GPU를 많이 사용합니다. 특히 여러 개의 Lottie 위젯이 동시에 재생되는 경우 프레임 드롭이 발생할 수 있습니다. `renderCache: RenderCache.raster`를 사용하면 각 프레임을 오프스크린 비트맵으로 캐시하여 반복 재생 시 렌더링 비용을 줄입니다.

```dart
Lottie.asset(
  'assets/animations/loader.json',
  renderCache: RenderCache.raster, // 메모리 ↑, CPU ↓
  repeat: true,
)
```

단, 메모리 사용량이 증가하므로 짧은 루프 애니메이션에 적합하고, 길거나 대형 애니메이션에는 주의가 필요합니다.

## 2. Rive 심화

Rive는 단순 재생을 넘어 **StateMachine**과 **Input**을 통해 앱 상태에 따라 애니메이션이 스스로 전환되는 구조를 제공합니다.

### 설치

```yaml
dependencies:
  rive: ^0.14.11
```

`.riv` 파일을 `assets/animations/`에 추가하고 `pubspec.yaml`에 등록합니다.

### 기본 재생: SimpleAnimation

단순 재생만 필요하다면 `SimpleAnimation`으로 충분합니다.

```dart
import 'package:flutter/material.dart';
import 'package:rive/rive.dart';

class SimpleLogo extends StatelessWidget {
  const SimpleLogo({super.key});

  @override
  Widget build(BuildContext context) {
    return const SizedBox(
      width: 200,
      height: 200,
      child: RiveAnimation.asset(
        'assets/animations/logo.riv',
        animations: ['idle'], // Rive Editor에서 정의한 타임라인 이름
        fit: BoxFit.contain,
      ),
    );
  }
}
```

### StateMachine과 SMIInput: 인터랙티브 애니메이션

Rive의 핵심은 **StateMachine**입니다. Rive Editor에서 디자이너가 상태(Idle, Hover, Pressed, Success 등)와 전환 조건을 정의해두면, Flutter 코드에서는 `SMIInput` 값만 바꿔주면 됩니다. 애니메이션 로직은 모두 `.riv` 파일 안에 캡슐화되어 있습니다.

다음은 좋아요 버튼 애니메이션 예제입니다. Rive Editor에서 `like_button`이라는 StateMachine에 `isLiked`라는 `SMIBool` Input이 정의되어 있다고 가정합니다.

```dart
import 'package:flutter/material.dart';
import 'package:rive/rive.dart';

class LikeButton extends StatefulWidget {
  const LikeButton({super.key});

  @override
  State<LikeButton> createState() => _LikeButtonState();
}

class _LikeButtonState extends State<LikeButton> {
  SMIBool? _isLiked;
  bool _liked = false;

  void _onRiveInit(Artboard artboard) {
    // StateMachineController를 아트보드에 연결합니다.
    final controller = StateMachineController.fromArtboard(
      artboard,
      'like_button', // Rive Editor의 StateMachine 이름
      onStateChange: _onStateChange,
    );

    if (controller != null) {
      artboard.addController(controller);
      // Input 이름으로 SMIBool을 가져옵니다.
      _isLiked = controller.findInput<bool>('isLiked') as SMIBool?;
    }
  }

  void _onStateChange(String stateMachineName, String stateName) {
    // 현재 재생 중인 상태 이름을 받을 수 있습니다.
    debugPrint('State changed: $stateMachineName -> $stateName');
  }

  void _toggleLike() {
    setState(() {
      _liked = !_liked;
      // SMIBool 값을 변경하면 StateMachine이 자동으로 전환 애니메이션을 재생합니다.
      _isLiked?.change(_liked);
    });
  }

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: _toggleLike,
      child: SizedBox(
        width: 80,
        height: 80,
        child: RiveAnimation.asset(
          'assets/animations/like_button.riv',
          fit: BoxFit.contain,
          onInit: _onRiveInit,
        ),
      ),
    );
  }
}
```

`SMIBool` 외에도 `SMINumber`(숫자 조건), `SMITrigger`(일회성 이벤트) 타입이 있습니다. 예를 들어, 캐릭터의 걷는 속도를 `SMINumber`로 실시간 제어하거나, 버튼 클릭 이벤트를 `SMITrigger`로 전달할 수 있습니다.

### SMITrigger 예제: 완료 애니메이션

```dart
// Trigger는 한 번 발사하면 자동으로 리셋됩니다.
SMITrigger? _completeTrigger;

void _onRiveInit(Artboard artboard) {
  final controller = StateMachineController.fromArtboard(
    artboard,
    'checkout_flow',
  );
  if (controller != null) {
    artboard.addController(controller);
    _completeTrigger =
        controller.findInput<bool>('complete') as SMITrigger?;
  }
}

void _onPaymentSuccess() {
  // Trigger를 발사하면 StateMachine이 Success 상태로 전환됩니다.
  _completeTrigger?.fire();
}
```

### Rive 렌더러 선택

Rive는 Flutter의 기본 렌더러(Skia/Impeller) 대신 자체 고성능 렌더러를 사용할 수 있습니다. 특히 복잡한 벡터 경로나 메시 애니메이션이 많을 때 유용합니다.

```dart
void main() {
  // 앱 시작 시 Rive 렌더러를 선택합니다.
  // RiveNativeAssets와 함께 사용할 때 권장됩니다.
  WidgetsFlutterBinding.ensureInitialized();
  RiveAnimation.useRiveRenderer(); // Rive 자체 렌더러
  runApp(const MyApp());
}
```

Flutter 렌더러를 사용할 경우 `useFlutterRenderer()`를 호출합니다. Impeller 환경에서는 Rive 자체 렌더러가 더 안정적인 경우가 많습니다.

## 3. 두 패키지를 함께 쓰는 실전 패턴

로딩 → 완료 → 에러 세 가지 상태를 가진 결제 처리 화면을 구현하는 예제입니다. 로딩 인디케이터는 Lottie로, 완료/에러 피드백은 Rive StateMachine으로 처리합니다.

```dart
enum PaymentState { loading, success, error }

class PaymentScreen extends StatefulWidget {
  const PaymentScreen({super.key});

  @override
  State<PaymentScreen> createState() => _PaymentScreenState();
}

class _PaymentScreenState extends State<PaymentScreen>
    with SingleTickerProviderStateMixin {
  PaymentState _state = PaymentState.loading;
  late final AnimationController _lottieController;
  SMITrigger? _successTrigger;
  SMITrigger? _errorTrigger;

  @override
  void initState() {
    super.initState();
    _lottieController = AnimationController(vsync: this);
    _startPayment();
  }

  Future<void> _startPayment() async {
    // 실제로는 API 호출
    await Future.delayed(const Duration(seconds: 2));
    final isSuccess = DateTime.now().second.isEven;

    if (!mounted) return;
    setState(() {
      _state = isSuccess ? PaymentState.success : PaymentState.error;
    });

    if (isSuccess) {
      _successTrigger?.fire();
    } else {
      _errorTrigger?.fire();
    }
  }

  void _onRiveInit(Artboard artboard) {
    final controller = StateMachineController.fromArtboard(
      artboard,
      'payment_result',
    );
    if (controller != null) {
      artboard.addController(controller);
      _successTrigger =
          controller.findInput<bool>('success') as SMITrigger?;
      _errorTrigger =
          controller.findInput<bool>('error') as SMITrigger?;
    }
  }

  @override
  void dispose() {
    _lottieController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(
        child: AnimatedSwitcher(
          duration: const Duration(milliseconds: 300),
          child: _state == PaymentState.loading
              ? Lottie.asset(
                  'assets/animations/payment_loading.json',
                  key: const ValueKey('loading'),
                  controller: _lottieController,
                  onLoaded: (comp) {
                    _lottieController.duration = comp.duration;
                    _lottieController.repeat();
                  },
                  width: 160,
                )
              : SizedBox(
                  key: const ValueKey('result'),
                  width: 160,
                  height: 160,
                  child: RiveAnimation.asset(
                    'assets/animations/payment_result.riv',
                    onInit: _onRiveInit,
                  ),
                ),
        ),
      ),
    );
  }
}
```

## 주의사항 및 팁

### 1. Lottie JSON 파일 크기 관리

After Effects 프로젝트가 복잡할수록 JSON 파일이 수 MB에 달할 수 있습니다. LottieFiles의 온라인 최적화 도구나 `lottie-minify` CLI를 사용하여 파일 크기를 줄이는 것이 좋습니다. DotLottie(`.lottie`) 형식은 JSON을 zip으로 압축하여 평균 30~40% 크기를 절약합니다.

### 2. Rive 파일 버전 호환성

Rive Editor와 `rive` Flutter 패키지의 파일 포맷 버전이 일치해야 합니다. 패키지를 업그레이드할 때 `.riv` 파일도 Rive Editor에서 최신 포맷으로 재익스포트해야 할 수 있습니다. 버전 불일치는 런타임 예외가 아닌 빈 화면으로 나타나는 경우가 많아 디버깅이 어려울 수 있습니다.

### 3. 웹 플랫폼 고려사항

Lottie 패키지는 웹에서 완벽 지원됩니다. 반면 Rive는 웹에서 Rive 자체 렌더러를 사용할 때 WASM 모듈이 추가로 로드되므로 초기 로딩 시간이 늘어날 수 있습니다. 웹을 주 타겟으로 한다면 Flutter 렌더러 모드를 사용하는 것이 안정적입니다.

### 4. 메모리 해제

`StateMachineController`는 별도로 `dispose()`를 호출할 필요 없지만, `Artboard`에 컨트롤러를 추가(addController)한 경우 위젯이 파괴될 때 자동으로 정리됩니다. 단, `RiveFile`을 직접 로드하는 고급 사용 패턴에서는 명시적으로 해제해야 합니다.

### 5. 디자이너와의 협업 규칙 정립

Lottie는 레이어 이름이, Rive는 StateMachine 이름·Input 이름이 Flutter 코드와 직결됩니다. 디자이너가 임의로 이름을 바꾸면 런타임 오류가 발생합니다. 팀 내에서 명명 컨벤션 문서를 만들고, 변경 시 개발팀에 반드시 알리는 프로세스를 구축해야 합니다.

### 6. 성능 프로파일링

복잡한 Rive 애니메이션이 여러 개 동시에 실행되거나, Lottie 파일에 이미지 레이어가 많으면 프레임 드롭이 발생합니다. Flutter DevTools의 Performance 탭에서 프레임 타임라인을 확인하고, 불필요한 리빌드가 발생하지 않도록 `RepaintBoundary`로 감싸는 것을 고려하세요.

## 정리

Lottie와 Rive는 서로 경쟁하는 도구가 아니라 상호 보완적입니다.

- **Lottie**: After Effects 워크플로우가 있는 팀에서, 디자이너가 만든 복잡한 연출을 그대로 앱에 가져와야 할 때. `AnimationController`와 연동해 스크롤 기반 애니메이션이나 순차 재생을 구현할 때.
- **Rive**: 앱의 상태 변화에 반응하는 인터랙티브 애니메이션이 필요할 때. 버튼 상태, 캐릭터 애니메이션, 게임 UI처럼 조건에 따라 다른 모션이 재생되어야 할 때.

두 도구를 올바르게 활용하면 Flutter 개발자가 직접 복잡한 `AnimationController` 체인을 작성하는 대신, 디자이너가 정의한 의도를 코드에서 깔끔하게 트리거하는 방식으로 역할을 분리할 수 있습니다. 그것이 바로 두 도구의 가장 큰 가치입니다.

## 참고 자료
- [lottie | Flutter package (pub.dev)](https://pub.dev/packages/lottie)
- [rive | Flutter package (pub.dev)](https://pub.dev/packages/rive)
- [Lottie for Flutter API Documentation](https://pub.dev/documentation/lottie/latest/)
