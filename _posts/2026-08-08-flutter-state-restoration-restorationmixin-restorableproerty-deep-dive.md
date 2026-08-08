---
layout: post
title: "Flutter State Restoration 심화: RestorationMixin과 RestorableProperty로 앱 재시작에도 살아남는 UI 상태 완전 정복"
date: 2026-08-08
categories: [android, flutter]
tags: [flutter, state-restoration, RestorationMixin, RestorableProperty, dart, ui-state]
---

앱이 백그라운드에서 OS에 의해 종료된 뒤 사용자가 다시 앱으로 돌아왔을 때, 이전에 보던 화면과 입력하던 내용이 그대로 남아있다면 어떨까요? Flutter의 State Restoration 시스템은 이것을 `RestorationMixin`과 `RestorableProperty`라는 표준 API로 가능하게 합니다. 이 글에서는 이 시스템의 내부 구조부터 실전 구현, 고급 패턴까지 완전히 정복합니다.

---

## 1. 개념: State Restoration이란?

**State Restoration(상태 복원)**은 앱이 시스템에 의해 종료되거나, 사용자가 앱을 백그라운드로 전환했다가 다시 돌아왔을 때 UI 상태를 이전과 동일하게 복구하는 메커니즘입니다.

Flutter에서 State Restoration은 다음 핵심 클래스들로 구성됩니다.

| 클래스 | 역할 |
|---|---|
| `RestorationManager` | OS와 직접 통신하며 복원 데이터를 관리하는 최상위 컨트롤러 |
| `RestorationBucket` | 계층적 키-값 저장소. 위젯 트리 구조를 반영하여 중첩 가능 |
| `RestorationScope` | 하위 위젯에 `RestorationBucket`을 제공하는 InheritedWidget |
| `RestorationMixin` | `State`에 믹스인하여 복원 등록 API를 제공 |
| `RestorableProperty<T>` | 타입 안전하게 직렬화/역직렬화되는 복원 가능 값 래퍼 |

Flutter의 복원 시스템은 Android의 `onSaveInstanceState` / `onRestoreInstanceState`, iOS의 State Preservation API와 유사한 개념이지만, 훨씬 더 선언적이고 타입 안전한 방식으로 추상화되어 있습니다. `RestorationManager`가 OS와 통신하며, 복원 데이터를 위젯 트리 계층에 맞게 자동으로 분배합니다.

### 내부 데이터 흐름

```
앱 종료 시:
  State → RestorableProperty.toPrimitives()
        → RestorationBucket (in-memory Map)
        → RestorationManager
        → OS (Android Bundle / iOS NSUserActivity)

앱 복원 시:
  OS → RestorationManager
     → RestorationBucket 재구성
     → RestorationScope를 통해 하위 위젯에 배포
     → RestorationMixin.restoreState() 호출
     → RestorableProperty.fromPrimitives()
     → State 값 복원
```

---

## 2. 왜 필요한가?

일반적인 `setState`나 상태 관리 패키지(Riverpod, BLoC 등)는 **메모리 내 상태**만 관리합니다. 프로세스가 종료되면 모든 상태가 사라집니다. 다음 상황에서 이 문제가 발생합니다.

- **메모리 부족(OOM)**: 사용자가 앱을 백그라운드에 두고 다른 앱을 여러 개 실행하면 OS가 프로세스를 종료합니다. 복귀 시 앱은 처음부터 재시작됩니다.
- **Activity 재생성(Android)**: 화면 회전, 다크 모드 전환, 언어 변경 시 Activity가 재생성됩니다.
- **멀티태스킹**: iOS에서 앱을 오랫동안 백그라운드에 두면 `applicationDidEnterBackground` 이후 프로세스가 정리될 수 있습니다.
- **폼 데이터 손실**: 긴 회원가입 폼을 입력하다가 전화가 왔다 돌아오면 모든 입력이 사라집니다.
- **스크롤 위치**: 수천 개 항목의 리스트를 보다가 나갔다 돌아오면 맨 위로 돌아갑니다.

`RestorationMixin`을 사용하면 이 문제를 Flutter 표준 API만으로 해결할 수 있으며, 테스트도 공식 지원합니다.

---

## 3. 기본 설정: MaterialApp에 restorationScopeId 추가

State Restoration을 활성화하려면 앱 최상단에 `restorationScopeId`를 반드시 설정해야 합니다. 이 값이 없으면 `RestorationMixin`이 동작하지 않습니다.

```dart
void main() {
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      // 이 ID가 없으면 RestorationMixin이 작동하지 않음
      restorationScopeId: 'app',
      title: 'State Restoration Demo',
      home: const CounterPage(),
    );
  }
}
```

---

## 4. 실제 구현 예제

### 예제 1: 기본 RestorationMixin 사용 — 카운터와 텍스트 필드 복원

```dart
class CounterPage extends StatefulWidget {
  const CounterPage({super.key});

  @override
  State<CounterPage> createState() => _CounterPageState();
}

// RestorationMixin을 State에 믹스인
class _CounterPageState extends State<CounterPage> with RestorationMixin {
  // RestorableProperty 계열: 프로세스 종료 후에도 값이 복원됨
  final RestorableInt _counter = RestorableInt(0);
  final RestorableString _lastAction = RestorableString('없음');
  final RestorableBool _isDarkMode = RestorableBool(false);
  // TextEditingController도 복원 가능
  final RestorableTextEditingController _nameController =
      RestorableTextEditingController();

  // restorationId: 이 State를 식별하는 고유 키
  // 같은 화면에 동일 위젯이 여러 개라면 각각 다른 ID를 사용해야 함
  @override
  String get restorationId => 'counter_page';

  // 사용할 RestorableProperty를 여기서 등록
  // initialRestore: true면 앱 첫 실행, false면 복원 중
  @override
  void restoreState(RestorationBucket? oldBucket, bool initialRestore) {
    registerForRestoration(_counter, 'counter');
    registerForRestoration(_lastAction, 'last_action');
    registerForRestoration(_isDarkMode, 'is_dark_mode');
    registerForRestoration(_nameController, 'name_controller');
  }

  @override
  void dispose() {
    _counter.dispose();
    _lastAction.dispose();
    _isDarkMode.dispose();
    _nameController.dispose();
    super.dispose();
  }

  void _increment() {
    setState(() {
      _counter.value++;
      _lastAction.value = '증가 (${_counter.value}회)';
    });
  }

  void _decrement() {
    setState(() {
      if (_counter.value > 0) _counter.value--;
      _lastAction.value = '감소 (${_counter.value}회)';
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('State Restoration 기본 예제'),
        actions: [
          Switch(
            value: _isDarkMode.value,
            onChanged: (v) => setState(() => _isDarkMode.value = v),
          ),
        ],
      ),
      body: Padding(
        padding: const EdgeInsets.all(24),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            // 이름 입력 — 앱 종료 후에도 입력값 유지
            TextField(
              controller: _nameController.value,
              decoration: const InputDecoration(
                labelText: '이름 입력 (종료 후에도 유지됨)',
                border: OutlineInputBorder(),
              ),
            ),
            const SizedBox(height: 32),
            Center(
              child: Text(
                '${_counter.value}',
                style: Theme.of(context).textTheme.displayLarge,
              ),
            ),
            Center(
              child: Text(
                '마지막 동작: ${_lastAction.value}',
                style: Theme.of(context).textTheme.bodyMedium,
              ),
            ),
            const SizedBox(height: 16),
            Row(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                FilledButton.icon(
                  onPressed: _decrement,
                  icon: const Icon(Icons.remove),
                  label: const Text('감소'),
                ),
                const SizedBox(width: 16),
                FilledButton.icon(
                  onPressed: _increment,
                  icon: const Icon(Icons.add),
                  label: const Text('증가'),
                ),
              ],
            ),
          ],
        ),
      ),
    );
  }
}
```

이 코드에서 핵심은 세 가지입니다.

1. `with RestorationMixin` — State에 믹스인
2. `restorationId` getter — 위젯 트리에서의 고유 식별자
3. `restoreState()` — `registerForRestoration()`으로 각 프로퍼티 등록

`registerForRestoration(_counter, 'counter')`를 호출하는 순간, `_counter`의 값이 변경될 때마다 `RestorationBucket`에 자동으로 저장됩니다.

---

### 예제 2: 커스텀 RestorableProperty — 복잡한 객체 복원

`RestorableInt`, `RestorableString` 등 기본 제공 타입 외에 커스텀 클래스를 복원하려면 `RestorableProperty<T>`를 직접 상속합니다.

```dart
// 복원할 커스텀 모델
class CartItem {
  final String id;
  final String name;
  final int quantity;
  final double price;

  const CartItem({
    required this.id,
    required this.name,
    required this.quantity,
    required this.price,
  });

  Map<String, dynamic> toMap() => {
    'id': id,
    'name': name,
    'quantity': quantity,
    'price': price,
  };

  factory CartItem.fromMap(Map<String, dynamic> map) => CartItem(
    id: map['id'] as String,
    name: map['name'] as String,
    quantity: map['quantity'] as int,
    price: (map['price'] as num).toDouble(),
  );
}

// 커스텀 RestorableProperty 구현
class RestorableCartItemList extends RestorableProperty<List<CartItem>> {
  RestorableCartItemList([List<CartItem>? initial])
      : _initial = initial ?? [];

  final List<CartItem> _initial;

  @override
  List<CartItem> createDefaultValue() => List.from(_initial);

  @override
  Object? toPrimitives() {
    return value.map((item) => item.toMap()).toList();
  }

  @override
  List<CartItem> fromPrimitives(Object? data) {
    if (data == null) return createDefaultValue();
    final rawList = data as List<dynamic>;
    return rawList
        .map((e) => CartItem.fromMap(Map<String, dynamic>.from(e as Map)))
        .toList();
  }

  void addItem(CartItem item) {
    value = [...value, item];
    notifyListeners();
  }

  void removeItem(String id) {
    value = value.where((e) => e.id != id).toList();
    notifyListeners();
  }

  void updateQuantity(String id, int quantity) {
    value = value.map((e) {
      return e.id == id
          ? CartItem(id: e.id, name: e.name, quantity: quantity, price: e.price)
          : e;
    }).toList();
    notifyListeners();
  }
}
```

---

## 5. Navigator와 페이지 스택 복원

State Restoration의 진짜 강점은 **페이지 스택**도 복원한다는 점입니다.

### GoRouter와 통합

```dart
MaterialApp.router(
  restorationScopeId: 'app',
  routerConfig: GoRouter(
    restorationScopeId: 'router',
    routes: [
      GoRoute(path: '/', builder: (_, __) => const HomePage()),
      GoRoute(path: '/cart', builder: (_, __) => const CartPage()),
    ],
  ),
)
```

### `restorablePush` 사용

```dart
// 복원 불가 (일반 push)
Navigator.push(context, MaterialPageRoute(builder: (_) => DetailPage(id: id)));

// 복원 가능 — route factory는 반드시 static 또는 top-level 함수여야 함
Navigator.restorablePush(context, _detailRoute, arguments: id);

static Route<void> _detailRoute(BuildContext context, Object? arguments) {
  return MaterialPageRoute(
    builder: (_) => DetailPage(id: arguments as String),
  );
}
```

---

## 6. 테스트에서 State Restoration 검증

```dart
testWidgets('장바구니 상태가 앱 재시작 후에도 유지된다', (tester) async {
  await tester.pumpWidget(
    RootRestorationScope(
      restorationId: 'root',
      child: const MaterialApp(
        restorationScopeId: 'app',
        home: CartPage(),
      ),
    ),
  );

  await tester.tap(find.byIcon(Icons.add_shopping_cart));
  await tester.pump();
  await tester.tap(find.byIcon(Icons.add_shopping_cart));
  await tester.pump();

  final restorationData = await tester.getRestorationData();

  // 앱 "재시작" 시뮬레이션
  await tester.pumpWidget(
    RootRestorationScope(
      restorationId: 'root',
      child: const MaterialApp(
        restorationScopeId: 'app',
        home: CartPage(),
      ),
    ),
  );
  await tester.restoreFrom(restorationData);
  await tester.pump();

  expect(find.text('상품 1'), findsOneWidget);
  expect(find.text('상품 2'), findsOneWidget);
});
```

---

## 7. 주의사항 및 팁

### restorationId 고유성 보장

```dart
// 잘못된 예: 리스트의 모든 아이템이 같은 ID
@override
String get restorationId => 'list_item';

// 올바른 예: 아이템 고유 ID 포함
@override
String get restorationId => 'list_item_${widget.itemId}';
```

### 저장 용량 제한

복원 데이터는 OS의 Bundle(Android) / NSUserActivity(iOS)에 저장되므로 용량 제한이 있습니다. 대용량 데이터는 절대 RestorableProperty에 저장하지 말고, ID/키만 저장하는 패턴을 사용하세요.

```dart
// 나쁜 예: 대용량 이미지 바이트를 직접 저장
final RestorableString _imageData = RestorableString(base64Image);

// 좋은 예: 이미지 경로/URL만 저장
final RestorableString _imagePath = RestorableString('');
```

### Riverpod/BLoC와의 통합

```dart
class _PageState extends ConsumerState<MyPage> with RestorationMixin {
  final RestorableInt _counter = RestorableInt(0);

  @override
  String get restorationId => 'my_page';

  @override
  void restoreState(RestorationBucket? oldBucket, bool initialRestore) {
    registerForRestoration(_counter, 'counter');
    // 복원된 경우 Riverpod Provider 동기화
    if (!initialRestore) {
      Future.microtask(() {
        ref.read(counterProvider.notifier).state = _counter.value;
      });
    }
  }
}
```

### Android에서 직접 테스트하기

Android 개발자 옵션에서 **"앱이 백그라운드 상태가 되는 즉시 활동 유지 안 함(Don't keep activities)"**을 활성화하면 백그라운드 전환 시마다 복원 로직을 테스트할 수 있습니다.

---

## 마무리

Flutter의 State Restoration 시스템은 처음에는 낯설어 보이지만, 핵심 구조를 이해하면 매우 체계적입니다.

- `MaterialApp(restorationScopeId: 'app')` — 시스템 활성화
- `with RestorationMixin` + `restorationId` — State에 연결
- `restoreState()`에서 `registerForRestoration()` — 프로퍼티 등록
- `RestorableProperty<T>` 상속 — 커스텀 타입 지원

이 네 단계만 기억하면 어떤 복잡한 UI 상태도 프로세스 종료에도 안전하게 복원할 수 있습니다. 특히 폼이 많은 앱, 전자상거래 앱, 미디어 앱에서 UX를 크게 개선할 수 있는 기술입니다.

## 참고 자료

- [Flutter 공식 문서: Android에서 상태 복원하기](https://docs.flutter.dev/platform-integration/android/restore-state-android)
- [Flutter 공식 문서: iOS에서 상태 복원하기](https://docs.flutter.dev/platform-integration/ios/restore-state-ios)
- [Dart API: RestorationMixin mixin](https://api.flutter.dev/flutter/widgets/RestorationMixin-mixin.html)
- [Dart API: RestorableProperty class](https://api.flutter.dev/flutter/widgets/RestorableProperty-class.html)
