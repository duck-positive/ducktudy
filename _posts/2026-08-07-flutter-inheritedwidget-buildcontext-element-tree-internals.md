---
layout: post
title: "Flutter InheritedWidget 완전 정복: BuildContext 의존성 전파와 Element 트리 내부 동작"
date: 2026-08-07
categories: [flutter]
tags: [flutter, inheritedwidget, buildcontext, inheritedmodel, state-management, element-tree]
---

Flutter에서 `Theme.of(context)`, `MediaQuery.of(context)`, `Navigator.of(context)` 같은 API를 매일 쓰지만, 내부에서 어떻게 동작하는지 정확히 아는 개발자는 많지 않습니다. 이 글에서는 `InheritedWidget`의 내부 구조, `BuildContext` 의존성 추적 메커니즘, `InheritedModel`을 활용한 부분 리빌드 최적화까지 완전히 파헤칩니다.

---

## 1. InheritedWidget이란 무엇인가

Flutter의 위젯 트리는 기본적으로 데이터를 **위에서 아래**로 전달합니다. 부모가 자식에게 생성자를 통해 값을 넘기는 방식이죠. 그런데 트리가 깊어지면 문제가 생깁니다. 중간 단계의 위젯들이 데이터를 전혀 쓰지 않으면서도 단순히 "아래로 넘기기 위해" 파라미터를 받아야 하는 **prop drilling** 현상이 발생합니다.

```dart
// prop drilling: 중간 위젯이 쓰지도 않는 user를 계속 전달
class ParentWidget extends StatelessWidget {
  final User user;
  const ParentWidget({required this.user, super.key});

  @override
  Widget build(BuildContext context) {
    return MiddleWidget(user: user);
  }
}

class MiddleWidget extends StatelessWidget {
  final User user; // 이 위젯은 user를 전혀 사용하지 않는다
  const MiddleWidget({required this.user, super.key});

  @override
  Widget build(BuildContext context) {
    return DeepChildWidget(user: user);
  }
}
```

`InheritedWidget`은 이 문제를 해결합니다. 트리의 어느 깊이에서든 `BuildContext`를 통해 가장 가까운 조상 `InheritedWidget`을 **O(1) 시간**에 찾을 수 있습니다. 또한, `InheritedWidget`이 변경되면 그것에 **의존(depend)** 하고 있는 하위 위젯들만 선택적으로 리빌드됩니다.

---

## 2. InheritedWidget이 필요한 이유: 세 가지 문제 해결

### 2-1. Prop Drilling 제거

`Theme`, `MediaQuery`, `Localizations`처럼 앱 전체에서 필요한 정보를 트리 최상단에 한 번만 제공하면, 어디서든 `context`를 통해 꺼내 쓸 수 있습니다.

### 2-2. 세밀한 리빌드 제어

일반 `StatefulWidget`이 `setState`를 호출하면 해당 위젯의 `build` 전체가 재실행됩니다. `InheritedWidget`은 `updateShouldNotify`를 통해 **실제로 변경된 경우에만** 의존 위젯들에게 알립니다.

### 2-3. Provider 패턴의 기반

`package:provider`, `Riverpod`, `flutter_bloc`처럼 Flutter 생태계의 주요 상태 관리 라이브러리들은 모두 내부적으로 `InheritedWidget`을 래핑해서 만들어져 있습니다. 이 메커니즘을 이해하면 상태 관리 라이브러리의 동작 원리도 명확해집니다.

---

## 3. 내부 동작: Element 트리와 `_inheritedWidgets` HashMap

`InheritedWidget`의 O(1) 조회가 가능한 비밀은 `Element` 클래스 내부의 `_inheritedWidgets` 필드에 있습니다.

### 3-1. `_inheritedWidgets`의 구조

Flutter의 모든 `Element`는 다음과 같은 필드를 가지고 있습니다 (Flutter 프레임워크 소스 기준):

```dart
// framework.dart (개념적 표현)
abstract class Element {
  Map<Type, InheritedElement>? _inheritedWidgets;
  Set<InheritedElement>? _dependencies;
  // ...
}
```

`_inheritedWidgets`는 `Type → InheritedElement` 매핑을 가진 HashMap입니다. 키는 `InheritedWidget` 서브클래스의 타입이고, 값은 해당 타입의 `InheritedElement`입니다.

### 3-2. 마운트 시 맵 전파

위젯이 트리에 마운트될 때 `_updateInheritance()` 메서드가 호출됩니다:

```dart
// Element의 기본 구현 (개념적 표현)
void _updateInheritance() {
  // 부모의 _inheritedWidgets 맵을 그대로 복사
  _inheritedWidgets = _parent?._inheritedWidgets;
}

// InheritedElement의 오버라이드 (개념적 표현)
@override
void _updateInheritance() {
  // 부모의 맵을 복사한 뒤, 자신을 추가
  final Map<Type, InheritedElement> incomingWidgets =
      _parent?._inheritedWidgets ?? const <Type, InheritedElement>{};
  _inheritedWidgets = HashMap<Type, InheritedElement>.from(incomingWidgets);
  _inheritedWidgets![widget.runtimeType] = this;
}
```

이 덕분에 모든 `Element`의 `_inheritedWidgets`에는 **자신의 모든 조상 InheritedElement**가 담겨 있습니다. 따라서 어떤 깊이에서도 타입으로 O(1) 조회가 가능합니다.

### 3-3. `dependOnInheritedWidgetOfExactType`의 두 가지 동작

`BuildContext.of()` 패턴의 핵심인 이 메서드는 두 가지 일을 합니다:

1. `_inheritedWidgets`에서 타입으로 `InheritedElement`를 찾는다 (O(1))
2. 현재 Element를 해당 `InheritedElement`의 **의존자(dependent)** 로 등록한다

```dart
// BuildContext 구현 (Element) - 개념적 표현
@override
T? dependOnInheritedWidgetOfExactType<T extends InheritedWidget>({Object? aspect}) {
  final InheritedElement? ancestor = _inheritedWidgets?[T];
  if (ancestor != null) {
    // 이 Element를 ancestor의 _dependents에 등록
    return dependOnInheritedElement(ancestor, aspect: aspect) as T;
  }
  _hadUnsatisfiedDependencies = true;
  return null;
}
```

반면, `getInheritedWidgetOfExactType`은 등록 없이 단순 조회만 합니다. 의존 관계가 생기지 않아 해당 `InheritedWidget`이 변경되어도 리빌드가 트리거되지 않습니다.

### 3-4. 변경 시 알림 전파

`InheritedWidget`이 교체될 때(부모가 리빌드될 때), `InheritedElement`는 `notifyClients()`를 호출합니다:

```dart
// InheritedElement - 개념적 표현
@override
void updated(InheritedWidget oldWidget) {
  if (widget.updateShouldNotify(oldWidget)) {
    notifyClients(oldWidget);
  }
}

@override
void notifyClients(InheritedWidget oldWidget) {
  for (final Element dependent in _dependents.keys) {
    notifyDependent(oldWidget, dependent);
  }
}
```

`updateShouldNotify`가 `true`를 반환할 때만 등록된 의존자들이 `markNeedsBuild()`로 표시되고, 다음 프레임에서 리빌드됩니다.

---

## 4. 실제 구현 예제 1: 커스텀 InheritedWidget

앱 전체에서 사용자 세션 정보를 제공하는 `UserSession InheritedWidget`을 구현합니다.

```dart
// user_session.dart

class UserSession {
  final String userId;
  final String displayName;
  final bool isPremium;

  const UserSession({
    required this.userId,
    required this.displayName,
    required this.isPremium,
  });

  UserSession copyWith({
    String? userId,
    String? displayName,
    bool? isPremium,
  }) {
    return UserSession(
      userId: userId ?? this.userId,
      displayName: displayName ?? this.displayName,
      isPremium: isPremium ?? this.isPremium,
    );
  }

  @override
  bool operator ==(Object other) =>
      identical(this, other) ||
      other is UserSession &&
          userId == other.userId &&
          displayName == other.displayName &&
          isPremium == other.isPremium;

  @override
  int get hashCode => Object.hash(userId, displayName, isPremium);
}

// InheritedWidget 정의
class UserSessionScope extends InheritedWidget {
  final UserSession session;

  const UserSessionScope({
    required this.session,
    required super.child,
    super.key,
  });

  // 의존 위젯들이 리빌드될지 결정하는 핵심 메서드
  @override
  bool updateShouldNotify(UserSessionScope oldWidget) {
    return session != oldWidget.session;
  }

  // 편의 정적 메서드 - 의존성 등록 O(1)
  static UserSession of(BuildContext context) {
    final scope = context
        .dependOnInheritedWidgetOfExactType<UserSessionScope>();
    assert(scope != null, 'UserSessionScope not found in widget tree');
    return scope!.session;
  }

  // 의존성 등록 없이 단순 읽기 (리빌드 트리거 X)
  static UserSession? maybeOf(BuildContext context) {
    return context
        .getInheritedWidgetOfExactType<UserSessionScope>()
        ?.session;
  }
}

// StatefulWidget으로 세션 상태 관리
class UserSessionProvider extends StatefulWidget {
  final Widget child;
  const UserSessionProvider({required this.child, super.key});

  @override
  State<UserSessionProvider> createState() => _UserSessionProviderState();
}

class _UserSessionProviderState extends State<UserSessionProvider> {
  UserSession _session = const UserSession(
    userId: '',
    displayName: 'Guest',
    isPremium: false,
  );

  void updateSession(UserSession newSession) {
    setState(() => _session = newSession);
  }

  @override
  Widget build(BuildContext context) {
    return UserSessionScope(
      session: _session,
      child: widget.child,
    );
  }
}

// 사용 예시
class ProfileHeader extends StatelessWidget {
  const ProfileHeader({super.key});

  @override
  Widget build(BuildContext context) {
    // 이 위젯은 UserSession이 변경될 때만 리빌드됨
    final session = UserSessionScope.of(context);

    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        Text(
          '안녕하세요, ${session.displayName}님',
          style: const TextStyle(fontSize: 20, fontWeight: FontWeight.bold),
        ),
        if (session.isPremium)
          const Chip(
            label: Text('Premium'),
            backgroundColor: Colors.amber,
          ),
      ],
    );
  }
}
```

---

## 5. 실제 구현 예제 2: InheritedModel로 부분 리빌드 최적화

`InheritedWidget`은 변경 시 등록된 모든 의존자를 리빌드합니다. 하지만 데이터의 일부만 변경됐는데 모든 의존자가 리빌드되면 낭비입니다. `InheritedModel`은 **aspect** 개념을 도입해 이 문제를 해결합니다.

```dart
// app_theme_model.dart

// aspect를 enum으로 정의
enum ThemeAspect { colorScheme, typography, spacing }

class AppThemeData {
  final ColorScheme colorScheme;
  final TextTheme typography;
  final double baseSpacing;

  const AppThemeData({
    required this.colorScheme,
    required this.typography,
    required this.baseSpacing,
  });
}

class AppThemeModel extends InheritedModel<ThemeAspect> {
  final AppThemeData theme;

  const AppThemeModel({
    required this.theme,
    required super.child,
    super.key,
  });

  // 1) 전체 업데이트 여부: 모든 aspect 알림 트리거 전 확인
  @override
  bool updateShouldNotify(AppThemeModel oldWidget) {
    return theme != oldWidget.theme;
  }

  // 2) 특정 aspect만 변경됐는지 확인
  @override
  bool updateShouldNotifyDependent(
    AppThemeModel oldWidget,
    Set<ThemeAspect> dependencies,
  ) {
    if (dependencies.contains(ThemeAspect.colorScheme)) {
      if (theme.colorScheme != oldWidget.theme.colorScheme) return true;
    }
    if (dependencies.contains(ThemeAspect.typography)) {
      if (theme.typography != oldWidget.theme.typography) return true;
    }
    if (dependencies.contains(ThemeAspect.spacing)) {
      if (theme.baseSpacing != oldWidget.theme.baseSpacing) return true;
    }
    return false;
  }

  // aspect를 지정해서 해당 부분이 변경될 때만 리빌드
  static ColorScheme colorSchemeOf(BuildContext context) {
    return InheritedModel.inheritFrom<AppThemeModel>(
      context,
      aspect: ThemeAspect.colorScheme,
    )!.theme.colorScheme;
  }

  static TextTheme typographyOf(BuildContext context) {
    return InheritedModel.inheritFrom<AppThemeModel>(
      context,
      aspect: ThemeAspect.typography,
    )!.theme.typography;
  }

  static double spacingOf(BuildContext context) {
    return InheritedModel.inheritFrom<AppThemeModel>(
      context,
      aspect: ThemeAspect.spacing,
    )!.theme.baseSpacing;
  }
}

// 색상만 사용하는 위젯: colorScheme이 바뀔 때만 리빌드
class ThemedButton extends StatelessWidget {
  final String label;
  final VoidCallback onPressed;

  const ThemedButton({
    required this.label,
    required this.onPressed,
    super.key,
  });

  @override
  Widget build(BuildContext context) {
    // colorScheme aspect에만 의존 → typography나 spacing이 바뀌어도 리빌드 안 됨
    final colors = AppThemeModel.colorSchemeOf(context);

    return ElevatedButton(
      style: ElevatedButton.styleFrom(
        backgroundColor: colors.primary,
        foregroundColor: colors.onPrimary,
      ),
      onPressed: onPressed,
      child: Text(label),
    );
  }
}

// 타이포그래피만 사용하는 위젯: typography aspect에만 의존
class ThemedHeadline extends StatelessWidget {
  final String text;
  const ThemedHeadline({required this.text, super.key});

  @override
  Widget build(BuildContext context) {
    final typography = AppThemeModel.typographyOf(context);

    return Text(
      text,
      style: typography.headlineMedium,
    );
  }
}
```

`InheritedModel`을 사용하면 색상 테마만 변경됐을 때 `ThemedButton`만 리빌드되고 `ThemedHeadline`은 리빌드되지 않습니다. 복잡한 앱에서 성능 차이가 상당히 클 수 있습니다.

---

## 6. dependOnInheritedWidgetOfExactType vs getInheritedWidgetOfExactType

| 메서드 | 의존성 등록 | 변경 시 리빌드 | 주 용도 |
|---|---|---|---|
| `dependOnInheritedWidgetOfExactType<T>()` | O | O | `build()` 메서드 내부에서 데이터를 읽을 때 |
| `getInheritedWidgetOfExactType<T>()` | X | X | 의존 없이 현재 값만 읽을 때 (이벤트 핸들러 등) |

```dart
// 올바른 패턴: build()에서는 의존성 등록 필요
@override
Widget build(BuildContext context) {
  // 변경 시 이 위젯이 리빌드됨
  final theme = context.dependOnInheritedWidgetOfExactType<AppThemeModel>();
  return Container(color: theme?.theme.colorScheme.primary);
}

// 올바른 패턴: 이벤트 핸들러에서는 단순 읽기
void _onButtonPressed(BuildContext context) {
  // 이 시점의 값만 읽으면 됨, 리빌드 등록 불필요
  final session = context.getInheritedWidgetOfExactType<UserSessionScope>();
  print(session?.session.userId);
}
```

---

## 7. 주의사항 및 성능 팁

### 7-1. `context`의 위치에 주의하라

`InheritedWidget`은 자신의 **하위** 위젯들에서만 조회됩니다. 같은 위젯이나 상위에서는 조회되지 않습니다.

```dart
// 잘못된 패턴
class WrongWidget extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return UserSessionScope(
      session: const UserSession(userId: '1', displayName: 'Test', isPremium: false),
      child: Builder(
        // Builder를 쓰지 않으면 context가 UserSessionScope보다 위에 있어서 null 반환
        builder: (innerContext) {
          // 이 innerContext는 UserSessionScope 아래에 있으므로 정상 조회
          final session = UserSessionScope.of(innerContext);
          return Text(session.displayName);
        },
      ),
    );
  }
}
```

### 7-2. `updateShouldNotify`에서 불필요한 비교를 피하라

```dart
// 비효율적: 매번 깊은 비교
@override
bool updateShouldNotify(MyInherited oldWidget) {
  return !const DeepCollectionEquality().equals(data, oldWidget.data);
}

// 효율적: 불변 객체 + identity 비교 or == 오버라이드
@override
bool updateShouldNotify(MyInherited oldWidget) {
  return data != oldWidget.data; // Data 클래스에서 == 오버라이드
}
```

### 7-3. `InheritedWidget`을 직접 쓰지 말고 `StatefulWidget`과 함께 사용하라

`InheritedWidget` 자체는 불변입니다. 데이터가 바뀌어야 하면 `StatefulWidget`이 `InheritedWidget`을 새로 생성해 트리에 제공해야 합니다. 대부분의 패턴은 `StatefulWidget` + `InheritedWidget` 쌍으로 구현됩니다.

### 7-4. 과도한 범위(scope) 분리

하나의 큰 `InheritedWidget`보다 관심사별로 작게 나누면, `updateShouldNotify`가 더 정밀하게 동작하고 불필요한 리빌드가 줄어듭니다.

```dart
// 나쁜 예: 모든 앱 상태를 하나의 InheritedWidget에
class AppState extends InheritedWidget {
  final UserSession session;
  final CartItems cartItems;
  final NotificationList notifications;
  // ...변경 하나가 모든 의존 위젯을 리빌드
}

// 좋은 예: 관심사 분리
class UserSessionScope extends InheritedWidget { ... }
class CartScope extends InheritedWidget { ... }
class NotificationScope extends InheritedWidget { ... }
```

### 7-5. Provider 패키지는 InheritedWidget 래퍼다

`Provider.of<T>(context)`는 내부적으로 `context.dependOnInheritedWidgetOfExactType<InheritedProvider<T>>()`를 호출합니다. `read()`는 `getInheritedWidgetOfExactType`을 사용해 의존 등록 없이 값을 읽습니다. 이 차이가 `watch()` vs `read()`의 동작 차이입니다.

---

## 8. 정리

| 개념 | 요약 |
|---|---|
| `_inheritedWidgets` | 모든 Element가 가진 조상 InheritedElement HashMap. 마운트 시 부모에서 복사됨 |
| `dependOnInheritedWidgetOfExactType` | O(1) 조회 + 의존성 등록. `build()`에서 사용 |
| `getInheritedWidgetOfExactType` | O(1) 조회만. 이벤트 핸들러 등 리빌드 불필요한 곳에서 사용 |
| `updateShouldNotify` | 의존 위젯들에게 알릴지 여부 결정 |
| `InheritedModel` | aspect 기반 부분 리빌드. 대형 데이터 모델에 효율적 |

`InheritedWidget`은 Flutter UI 프레임워크의 핵심 메커니즘입니다. `Provider`, `Riverpod`, `BLoC` 같은 라이브러리를 쓸 때도 내부에서 이 메커니즘이 돌아가고 있습니다. 이 원리를 이해하면 상태 관리 라이브러리의 동작을 예측하고, 성능 문제를 디버깅하는 데 큰 도움이 됩니다.

---

## 참고 자료

- [InheritedWidget class - Flutter API](https://api.flutter.dev/flutter/widgets/InheritedWidget-class.html)
- [InheritedModel class - Flutter API](https://api.flutter.dev/flutter/widgets/InheritedModel-class.html)
- [dependOnInheritedWidgetOfExactType - BuildContext API](https://api.flutter.dev/flutter/widgets/BuildContext/dependOnInheritedWidgetOfExactType.html)
- [Inside Flutter - Flutter 공식 아키텍처 문서](https://docs.flutter.dev/resources/inside-flutter)
