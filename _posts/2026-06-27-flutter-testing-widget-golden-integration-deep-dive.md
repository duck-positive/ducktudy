---
layout: post
title: "Flutter 테스팅 심화: Widget Test·Golden Test·Integration Test 완전 정복"
date: 2026-06-27
categories: [flutter]
tags: [flutter, testing, widget-test, golden-test, integration-test, dart, tdd]
---

앱의 품질을 보장하는 가장 확실한 방법은 테스트 자동화입니다. Flutter는 **Unit Test**, **Widget Test**, **Golden Test**, **Integration Test**라는 네 가지 테스트 레이어를 공식적으로 지원하며, 각각의 역할과 작성 방법이 뚜렷이 다릅니다. 이 글에서는 현업에서 특히 중요하면서도 자주 놓치는 **Widget Test, Golden Test, Integration Test**를 깊이 파고들어 실전에서 바로 쓸 수 있는 패턴을 정리합니다.

---

## 왜 Flutter 테스팅이 중요한가?

Flutter 앱은 단일 Dart 코드베이스로 Android, iOS, Web, Desktop을 모두 커버합니다. 이 말은 하나의 버그가 모든 플랫폼에 동시에 영향을 준다는 뜻이기도 합니다. 특히 UI 로직이 비즈니스 로직과 뒤섞이기 쉬운 Flutter 특성상, 자동화된 테스트 없이는 리팩터링 자체가 위험 행위가 됩니다.

Flutter 테스팅의 레이어별 역할을 정리하면 다음과 같습니다:

| 레이어 | 속도 | 비용 | 커버 범위 |
|---|---|---|---|
| Unit Test | 매우 빠름 | 낮음 | 순수 함수, ViewModel, Repository |
| Widget Test | 빠름 | 중간 | 개별 위젯 렌더링·인터랙션 |
| Golden Test | 중간 | 중간 | 픽셀 단위 UI 회귀 감지 |
| Integration Test | 느림 | 높음 | 전체 앱 플로우 (실기기/에뮬레이터) |

---

## 1. Widget Test 심화

### 개념

Widget Test는 `flutter_test` 패키지의 `testWidgets()` 함수를 사용하여 **실제 디바이스 없이** 위젯을 렌더링하고 인터랙션을 검증합니다. Flutter 엔진이 오프스크린(off-screen)으로 위젯 트리를 구성하므로 수 밀리초 안에 실행됩니다.

핵심 클래스는 세 가지입니다:

- **`WidgetTester`**: 위젯 빌드, 탭/스크롤/입력 등 인터랙션 수행
- **`Finder`**: `find.byType()`, `find.byKey()`, `find.text()` 등으로 위젯 탐색
- **`Matcher`**: `findsOneWidget`, `findsNothing`, `findsNWidgets(n)` 등으로 검증

### 실전 예제 1: 카운터 위젯 테스트

```dart
// test/counter_widget_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:my_app/widgets/counter_widget.dart';

void main() {
  group('CounterWidget', () {
    testWidgets('초기값이 0으로 렌더링된다', (WidgetTester tester) async {
      await tester.pumpWidget(
        const MaterialApp(home: CounterWidget()),
      );

      // Text 위젯에서 '0' 텍스트를 찾음
      expect(find.text('0'), findsOneWidget);
      expect(find.text('증가'), findsOneWidget);
    });

    testWidgets('증가 버튼 탭 시 카운터가 1 증가한다', (WidgetTester tester) async {
      await tester.pumpWidget(
        const MaterialApp(home: CounterWidget()),
      );

      // 버튼 탭 후 rebuild 대기
      await tester.tap(find.text('증가'));
      await tester.pump();

      expect(find.text('1'), findsOneWidget);
    });

    testWidgets('비동기 데이터 로딩 중 로딩 인디케이터가 표시된다',
        (WidgetTester tester) async {
      await tester.pumpWidget(
        const MaterialApp(home: AsyncDataWidget()),
      );

      // 첫 pump: 위젯 빌드
      expect(find.byType(CircularProgressIndicator), findsOneWidget);

      // 비동기 완료까지 대기
      await tester.pumpAndSettle();

      expect(find.byType(CircularProgressIndicator), findsNothing);
      expect(find.byType(ListView), findsOneWidget);
    });

    testWidgets('에러 상태에서 재시도 버튼이 나타난다', (WidgetTester tester) async {
      await tester.pumpWidget(
        MaterialApp(
          home: AsyncDataWidget(
            // 에러를 강제로 주입
            repository: FailingRepository(),
          ),
        ),
      );

      await tester.pumpAndSettle();

      expect(find.text('다시 시도'), findsOneWidget);

      // 재시도 버튼 탭
      await tester.tap(find.text('다시 시도'));
      await tester.pump();

      // 로딩 인디케이터가 다시 나타나야 함
      expect(find.byType(CircularProgressIndicator), findsOneWidget);
    });
  });
}
```

### pump vs pumpAndSettle

| 메서드 | 동작 |
|---|---|
| `pump()` | 단 한 프레임을 진행. 애니메이션 중간 상태 테스트에 유용 |
| `pump(duration)` | 지정 시간만큼 프레임 진행 |
| `pumpAndSettle()` | 애니메이션과 Future가 모두 완료될 때까지 반복 pump |

`pumpAndSettle()`은 무한 애니메이션(`AnimationController`가 반복)이 있을 경우 타임아웃으로 실패하니 주의해야 합니다. 이 경우 `pump(const Duration(seconds: 1))`처럼 직접 시간을 제어하세요.

---

## 2. Golden Test 심화

### 개념

Golden Test(스냅샷 테스트)는 위젯의 **픽셀 단위 렌더링 결과를 이미지 파일로 저장**하고, 이후 테스트 실행 시 저장된 이미지와 현재 렌더링을 비교합니다. UI 회귀를 자동으로 감지하는 강력한 도구입니다.

핵심 API는 `matchesGoldenFile(path)` Matcher입니다:

```dart
await expectLater(
  find.byType(MyWidget),
  matchesGoldenFile('goldens/my_widget.png'),
);
```

골든 파일 최초 생성 또는 갱신은 다음 명령어로 수행합니다:

```bash
flutter test --update-goldens
```

### 실전 예제 2: golden_toolkit을 이용한 다중 시나리오 Golden Test

`golden_toolkit` 패키지는 여러 상태를 한 이미지에 나란히 렌더링하는 `GoldenBuilder`를 제공합니다.

```dart
// test/product_card_golden_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:golden_toolkit/golden_toolkit.dart';
import 'package:my_app/widgets/product_card.dart';

void main() {
  setUpAll(() async {
    // 폰트 로드 (기본 폰트가 없으면 박스로 렌더링됨)
    await loadAppFonts();
  });

  group('ProductCard Golden Tests', () {
    testGoldens('다양한 상태의 ProductCard 렌더링', (WidgetTester tester) async {
      await tester.pumpWidgetBuilder(
        GoldenBuilder.column()
          ..addScenario(
            '정상 상태',
            ProductCard(
              product: Product(
                name: '에어팟 프로',
                price: 329000,
                imageUrl: 'https://example.com/img.png',
                isDiscounted: false,
              ),
            ),
          )
          ..addScenario(
            '할인 배지 표시',
            ProductCard(
              product: Product(
                name: '에어팟 프로',
                price: 249000,
                originalPrice: 329000,
                isDiscounted: true,
              ),
            ),
          )
          ..addScenario(
            '품절 상태',
            ProductCard(
              product: Product(
                name: '에어팟 프로',
                price: 329000,
                isSoldOut: true,
              ),
            ),
          )
          ..addScenario(
            '로딩 스켈레톤',
            const ProductCardSkeleton(),
          )
          .build(),
        surfaceSize: const Size(400, 800),
      );

      await screenMatchesGolden(tester, 'product_card_scenarios');
    });

    testGoldens('다크 모드 ProductCard', (WidgetTester tester) async {
      await tester.pumpWidgetBuilder(
        ProductCard(
          product: Product(name: '맥북 프로', price: 2390000),
        ),
        wrapper: materialAppWrapper(
          theme: ThemeData.dark(),
        ),
        surfaceSize: const Size(400, 200),
      );

      await screenMatchesGolden(tester, 'product_card_dark_mode');
    });
  });
}
```

### 골든 테스트 운용 팁

**1. CI에서의 픽셀 불일치 문제**

로컬 macOS와 Linux CI 환경 간에 폰트 렌더링 차이가 발생합니다. 해결 방법은 두 가지입니다:

- **Skia Gold** 사용 (Google의 공식 golden 비교 서비스)
- CI 전용 골든 파일 분리: `goldens/ci/` vs `goldens/local/` 디렉토리로 관리

**2. 테스트 격리**

골든 테스트는 외부 이미지나 네트워크 요청이 있으면 재현성이 깨집니다. `Image.network()`를 테스트에서는 반드시 mock으로 교체하거나 `FakeHttpOverrides`를 사용하세요.

```dart
setUpAll(() {
  HttpOverrides.global = FakeHttpOverrides();
});
```

---

## 3. Integration Test 심화

### 개념

Integration Test는 **실제 앱을 실기기 또는 에뮬레이터에서 구동**하며 전체 사용자 플로우를 테스트합니다. `integration_test` 패키지와 `flutter_test`의 `WidgetTester`를 함께 사용하므로 Widget Test와 문법이 유사하지만, 앱 프로세스 내부에서 직접 실행된다는 점이 다릅니다.

**프로젝트 구조:**

```
my_app/
├── integration_test/
│   ├── app_test.dart         # 테스트 진입점
│   └── flows/
│       ├── auth_flow_test.dart
│       └── checkout_flow_test.dart
├── test_driver/
│   └── integration_test.dart # 드라이버 (레거시 방식)
```

**실행 명령어:**

```bash
# Android
flutter test integration_test/ -d emulator-5554

# iOS
flutter test integration_test/ -d "iPhone 15"

# 특정 파일만
flutter test integration_test/flows/auth_flow_test.dart
```

### 실전 예제 3: 로그인 → 홈 플로우 Integration Test

```dart
// integration_test/flows/auth_flow_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';
import 'package:my_app/main.dart' as app;

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  group('인증 플로우', () {
    setUp(() async {
      // 테스트마다 앱 상태 초기화
      await clearSharedPreferences();
    });

    testWidgets('로그인 성공 후 홈 화면으로 이동', (WidgetTester tester) async {
      app.main();
      await tester.pumpAndSettle();

      // 로그인 화면 확인
      expect(find.text('로그인'), findsOneWidget);

      // 이메일 입력
      await tester.enterText(
        find.byKey(const Key('email_field')),
        'test@example.com',
      );

      // 비밀번호 입력
      await tester.enterText(
        find.byKey(const Key('password_field')),
        'password123',
      );

      // 키보드 닫기
      await tester.testTextInput.receiveAction(TextInputAction.done);
      await tester.pumpAndSettle();

      // 로그인 버튼 탭
      await tester.tap(find.byKey(const Key('login_button')));

      // 네트워크 응답 대기 (최대 10초)
      await tester.pumpAndSettle(const Duration(seconds: 10));

      // 홈 화면으로 이동됐는지 확인
      expect(find.byType(HomeScreen), findsOneWidget);
      expect(find.text('환영합니다'), findsOneWidget);
    });

    testWidgets('잘못된 비밀번호 입력 시 에러 메시지 표시', (WidgetTester tester) async {
      app.main();
      await tester.pumpAndSettle();

      await tester.enterText(
        find.byKey(const Key('email_field')),
        'test@example.com',
      );
      await tester.enterText(
        find.byKey(const Key('password_field')),
        'wrong_password',
      );

      await tester.tap(find.byKey(const Key('login_button')));
      await tester.pumpAndSettle(const Duration(seconds: 5));

      expect(find.text('이메일 또는 비밀번호가 올바르지 않습니다.'), findsOneWidget);
      // 여전히 로그인 화면에 있어야 함
      expect(find.byType(HomeScreen), findsNothing);
    });

    testWidgets('자동 로그인: 토큰 존재 시 홈으로 바로 이동', (WidgetTester tester) async {
      // 사전에 토큰 저장 (로그인된 상태 시뮬레이션)
      await saveAuthToken('valid_token_here');

      app.main();
      await tester.pumpAndSettle(const Duration(seconds: 3));

      // 로그인 화면을 거치지 않고 홈으로 이동
      expect(find.byType(LoginScreen), findsNothing);
      expect(find.byType(HomeScreen), findsOneWidget);
    });
  });
}
```

### Patrol: Integration Test의 한계를 극복하는 패키지

표준 `integration_test`는 Flutter 위젯 트리 내부만 제어할 수 있습니다. **시스템 다이얼로그**(권한 요청, 알림 팝업)나 **다른 앱**과의 상호작용이 필요한 경우에는 `patrol` 패키지를 사용합니다.

```dart
// Patrol 예제: 카메라 권한 허용 후 촬영 테스트
import 'package:patrol/patrol.dart';

void main() {
  patrolTest('카메라 권한 허용 후 사진 촬영', ($) async {
    await $.pumpWidgetAndSettle(const MyApp());

    await $('카메라 열기').tap();

    // 시스템 권한 다이얼로그를 네이티브 레벨에서 탭
    if (await $.native.isPermissionDialogVisible()) {
      await $.native.grantPermissionWhenInUse();
    }

    await $.pumpAndSettle();
    expect($('촬영'), findsOneWidget);

    await $('촬영').tap();
    await $.pumpAndSettle();

    expect($('사진이 저장되었습니다'), findsOneWidget);
  });
}
```

---

## 4. 테스트 아키텍처 설계 팁

### Mock과 Fake의 구분

- **Mock**: `mockito` 또는 `mocktail`로 생성. 호출 여부와 인자를 검증할 때 사용.
- **Fake**: 테스트용 구현체를 직접 작성. 비즈니스 로직을 실제처럼 동작시킬 때 사용.

Widget Test에서는 Repository 레이어를 Fake로 교체하는 패턴을 권장합니다:

```dart
class FakeUserRepository implements UserRepository {
  @override
  Future<User> getUser(String id) async {
    return User(id: id, name: '테스트 유저', email: 'test@example.com');
  }
}
```

### 테스트 피라미드 황금비율

일반적으로 권장되는 비율은 **Unit 70% : Widget 20% : Integration 10%** 입니다. Integration Test는 실행 비용이 높으므로 크리티컬한 사용자 경로(로그인, 결제, 핵심 기능)에만 집중합니다.

### CI 파이프라인 구성 (GitHub Actions 예시)

```yaml
# .github/workflows/test.yml
name: Flutter Tests
on: [push, pull_request]

jobs:
  unit_widget_test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.32.0'
      - run: flutter pub get
      - run: flutter test --coverage
      - run: flutter test --update-goldens  # 골든 파일 갱신 여부 체크

  integration_test:
    runs-on: macos-latest  # iOS 테스트를 위해
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
      - uses: reactivecircus/android-emulator-runner@v2
        with:
          api-level: 33
          script: flutter test integration_test/
```

---

## 주의사항 및 팁 요약

1. **`pumpAndSettle` 타임아웃**: 기본 100ms × 1000회 = 100초. 무한 애니메이션에서는 사용하지 마세요.
2. **Key 사용**: `ValueKey`, `ObjectKey`보다는 `const Key('id')` 형태로 일관되게 사용하면 Finder 유지보수가 편합니다.
3. **골든 테스트 용량**: 골든 PNG 파일들은 Git LFS로 관리하거나 `.gitignore`에 추가하고 CI에서만 생성하는 전략이 효과적입니다.
4. **테스트 격리**: 각 테스트는 독립적이어야 합니다. `setUp`/`tearDown`으로 SharedPreferences, 인메모리 DB 등을 초기화하세요.
5. **`find.descendant`**: 복잡한 위젯 트리에서는 `find.descendant(of: X, matching: Y)` 조합으로 탐색 범위를 좁혀 flakiness를 줄이세요.

---

Flutter 테스팅은 단순히 버그를 잡는 도구가 아닙니다. 리팩터링을 두려움 없이 할 수 있게 해주는 **안전망**이며, 팀 전체의 개발 속도를 높이는 투자입니다. Widget Test로 UI 로직을 검증하고, Golden Test로 시각적 회귀를 막고, Integration Test로 사용자 플로우를 보증하는 세 겹의 방어선을 갖추세요.

## 참고 자료
- [Flutter 공식 테스팅 개요 - flutter.dev](https://docs.flutter.dev/testing/overview)
- [integration_test 패키지 - pub.dev](https://pub.dev/packages/integration_test)
- [golden_toolkit 패키지 - pub.dev](https://pub.dev/packages/golden_toolkit)
- [patrol - 네이티브 UI 자동화 테스팅 - pub.dev](https://pub.dev/packages/patrol)
