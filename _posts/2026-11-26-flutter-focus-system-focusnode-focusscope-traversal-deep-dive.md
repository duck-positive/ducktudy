---
layout: post
title: "Flutter Focus 시스템 심화: FocusNode·FocusScope·FocusTraversalGroup으로 키보드 탐색과 포커스 관리 완전 정복"
date: 2026-11-26
categories: [android, flutter]
tags: [flutter, focusnode, focusscope, focustraversalgroup, keyboard, accessibility, dart]
---

Flutter 앱을 데스크톱이나 태블릿, 또는 외부 키보드가 연결된 환경에서 동작시켜 본 적 있다면 "Tab 키를 눌렀을 때 포커스가 의도치 않은 순서로 이동한다"거나 "텍스트 입력 후 다음 필드로 자동으로 넘어가지 않는다"는 불편함을 겪어봤을 것입니다. Flutter의 **포커스 시스템**은 이 모든 동작을 제어하는 핵심 메커니즘입니다. 이 글에서는 `FocusNode`, `FocusScope`, `FocusTraversalGroup`, `FocusTraversalPolicy`의 내부 구조를 깊이 파헤치고, 실전 코드와 함께 완벽하게 제어하는 방법을 알아봅니다.

---

## 1. 개념 설명: Flutter 포커스 시스템의 구조

### 포커스 트리(Focus Tree)

Flutter의 포커스 시스템은 위젯 트리와 별개로 존재하는 **포커스 트리**를 유지합니다. 이 트리는 `FocusManager`가 최상위 루트를 갖고, 그 아래로 `FocusScopeNode`와 `FocusNode`들이 계층 구조를 이룹니다.

```
FocusManager (root)
  └─ FocusScopeNode (앱 최상위)
       ├─ FocusScopeNode (Route 1)
       │    ├─ FocusNode (TextField A)
       │    └─ FocusNode (TextField B)
       └─ FocusScopeNode (Route 2)
            └─ FocusNode (Button C)
```

이 구조 덕분에 새로운 Route가 push될 때 이전 Route의 FocusScopeNode는 자동으로 포커스를 잃고, pop될 때 다시 복원됩니다.

### 핵심 클래스 정리

| 클래스 | 역할 |
|---|---|
| `FocusNode` | 단일 포커스 가능한 노드. 포커스 상태와 이벤트를 관리 |
| `FocusScopeNode` | FocusNode의 그룹. 범위 내 포커스 이동을 제어 |
| `Focus` | FocusNode를 위젯 트리에 연결하는 위젯 |
| `FocusScope` | FocusScopeNode를 위젯 트리에 연결하는 위젯 |
| `FocusManager` | 앱 전체의 포커스 트리를 관리하는 싱글턴 |
| `FocusTraversalGroup` | 그룹 내 순회 정책을 지정하는 위젯 |
| `FocusTraversalPolicy` | Tab 순서 등 포커스 이동 알고리즘을 정의하는 추상 클래스 |

### 포커스의 두 가지 개념

Flutter에서 포커스는 두 가지 의미를 갖습니다.

- **hasFocus**: 해당 노드 또는 그 하위 노드 중 하나가 포커스를 가짐
- **hasPrimaryFocus**: 해당 노드 자신이 직접 포커스를 가짐

`FocusNode.hasFocus`는 자식 노드가 포커스를 가져도 `true`가 되지만, `FocusNode.hasPrimaryFocus`는 오직 자신이 최종 포커스 대상일 때만 `true`입니다. 이 차이를 이해하는 것이 복잡한 포커스 UI를 구현할 때 매우 중요합니다.

---

## 2. 왜 필요한가

### 접근성(Accessibility) 요구사항

ADA, WCAG 2.1 같은 접근성 표준은 키보드만으로 앱의 모든 기능을 사용할 수 있어야 함을 요구합니다. Android에서는 TalkBack, iOS에서는 VoiceOver가 포커스 순서에 따라 화면을 읽어 줍니다. 포커스 순서가 잘못되면 시각장애인 사용자가 앱을 전혀 사용할 수 없는 상황이 됩니다.

### 데스크톱·웹·태블릿 지원

Flutter는 하나의 코드베이스로 iOS, Android, 웹, macOS, Windows, Linux를 모두 지원합니다. 데스크톱 환경에서는 Tab/Shift+Tab 키보드 탐색이 기본 사용 패턴이고, 외부 키보드가 연결된 태블릿에서도 마찬가지입니다. 포커스를 제대로 관리하지 않으면 데스크톱 앱의 품질이 급격히 떨어집니다.

### 폼(Form) UX 개선

로그인 화면, 회원가입 화면처럼 여러 `TextField`가 연속되는 경우, 사용자가 엔터나 "다음" 버튼을 누를 때 다음 필드로 포커스가 자동 이동하는 UX는 필수적입니다. 이를 구현하려면 `FocusNode`를 직접 다뤄야 합니다.

---

## 3. 실제 구현 예제

### 예제 1: 로그인 폼의 포커스 이동 완벽 구현

가장 흔한 사용 사례인 이메일 → 비밀번호 → 로그인 버튼으로의 포커스 흐름을 구현합니다. `TextInputAction.next`와 `FocusScope.of(context).nextFocus()`를 조합하면 소프트 키보드의 액션 버튼도 연동됩니다.

```dart
import 'package:flutter/material.dart';

class LoginForm extends StatefulWidget {
  const LoginForm({super.key});

  @override
  State<LoginForm> createState() => _LoginFormState();
}

class _LoginFormState extends State<LoginForm> {
  // 각 필드에 대한 FocusNode를 명시적으로 생성
  final _emailFocus = FocusNode();
  final _passwordFocus = FocusNode();
  final _buttonFocus = FocusNode();

  final _emailController = TextEditingController();
  final _passwordController = TextEditingController();

  @override
  void dispose() {
    // FocusNode는 반드시 dispose 해야 메모리 누수 방지
    _emailFocus.dispose();
    _passwordFocus.dispose();
    _buttonFocus.dispose();
    _emailController.dispose();
    _passwordController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.all(24.0),
      child: Column(
        mainAxisAlignment: MainAxisAlignment.center,
        children: [
          TextField(
            controller: _emailController,
            focusNode: _emailFocus,
            decoration: const InputDecoration(
              labelText: '이메일',
              border: OutlineInputBorder(),
            ),
            keyboardType: TextInputType.emailAddress,
            // 소프트 키보드 액션 버튼을 "다음"으로 설정
            textInputAction: TextInputAction.next,
            onSubmitted: (_) {
              // 엔터 키 또는 "다음" 버튼 → 비밀번호 필드로 이동
              _passwordFocus.requestFocus();
            },
          ),
          const SizedBox(height: 16),
          TextField(
            controller: _passwordController,
            focusNode: _passwordFocus,
            decoration: const InputDecoration(
              labelText: '비밀번호',
              border: OutlineInputBorder(),
            ),
            obscureText: true,
            textInputAction: TextInputAction.done,
            onSubmitted: (_) {
              // "완료" 버튼 → 로그인 버튼에 포커스 이동 후 로그인 실행
              _buttonFocus.requestFocus();
              _login();
            },
          ),
          const SizedBox(height: 24),
          Focus(
            focusNode: _buttonFocus,
            // 포커스 상태에 따라 버튼 스타일을 변경
            child: Builder(builder: (context) {
              final isFocused = Focus.of(context).hasPrimaryFocus;
              return AnimatedContainer(
                duration: const Duration(milliseconds: 150),
                decoration: BoxDecoration(
                  borderRadius: BorderRadius.circular(8),
                  boxShadow: isFocused
                      ? [
                          BoxShadow(
                            color: Colors.blue.withOpacity(0.5),
                            blurRadius: 8,
                            spreadRadius: 2,
                          )
                        ]
                      : [],
                ),
                child: ElevatedButton(
                  onPressed: _login,
                  style: ElevatedButton.styleFrom(
                    minimumSize: const Size.fromHeight(48),
                  ),
                  child: const Text('로그인'),
                ),
              );
            }),
          ),
        ],
      ),
    );
  }

  void _login() {
    // 로그인 로직
    debugPrint('로그인 시도: ${_emailController.text}');
    // 완료 후 포커스 해제
    FocusScope.of(context).unfocus();
  }
}
```

**핵심 포인트:**
- `FocusNode`는 `StatefulWidget`의 `State` 안에서 생성하고, `dispose()`에서 반드시 해제합니다.
- `_passwordFocus.requestFocus()`는 `FocusScope.of(context).requestFocus(_passwordFocus)`와 동일하지만, 직접 호출이 더 간결하고 성능적으로도 미세하게 유리합니다.
- `Focus.of(context).hasPrimaryFocus`로 현재 포커스 상태를 읽어 시각적 피드백을 줄 수 있습니다.

---

### 예제 2: FocusTraversalGroup과 커스텀 순회 정책

대시보드처럼 복잡한 레이아웃에서는 기본 Tab 순서가 의도와 다를 수 있습니다. `FocusTraversalGroup`과 `OrderedTraversalPolicy`를 사용해 명시적으로 순서를 지정합니다.

```dart
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';

/// 복잡한 대시보드 레이아웃에서 커스텀 포커스 순서 구현
class DashboardScreen extends StatelessWidget {
  const DashboardScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('대시보드')),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Row(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            // 왼쪽 사이드바 - 별도의 FocusTraversalGroup
            Expanded(
              flex: 1,
              child: FocusTraversalGroup(
                // 위젯 생성 순서대로 순회
                policy: const WidgetOrderTraversalPolicy(),
                child: Column(
                  children: [
                    _SidebarItem(label: '홈', order: 1),
                    _SidebarItem(label: '통계', order: 2),
                    _SidebarItem(label: '설정', order: 3),
                  ],
                ),
              ),
            ),
            const SizedBox(width: 16),
            // 오른쪽 콘텐츠 - OrderedTraversalPolicy로 명시적 순서 지정
            Expanded(
              flex: 3,
              child: FocusTraversalGroup(
                policy: const OrderedTraversalPolicy(),
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    // 검색창을 첫 번째 포커스 대상으로 지정
                    FocusTraversalOrder(
                      order: const NumericFocusOrder(1),
                      child: _SearchField(),
                    ),
                    const SizedBox(height: 16),
                    // 카드들은 2, 3, 4 순서
                    Row(
                      children: [
                        Expanded(
                          child: FocusTraversalOrder(
                            order: const NumericFocusOrder(2),
                            child: _DashboardCard(title: '총 사용자', value: '1,234'),
                          ),
                        ),
                        const SizedBox(width: 8),
                        Expanded(
                          child: FocusTraversalOrder(
                            order: const NumericFocusOrder(3),
                            child: _DashboardCard(title: '오늘 방문자', value: '89'),
                          ),
                        ),
                        const SizedBox(width: 8),
                        Expanded(
                          child: FocusTraversalOrder(
                            order: const NumericFocusOrder(4),
                            child: _DashboardCard(title: '매출', value: '₩2.3M'),
                          ),
                        ),
                      ],
                    ),
                    const SizedBox(height: 16),
                    // 액션 버튼은 마지막
                    FocusTraversalOrder(
                      order: const NumericFocusOrder(5),
                      child: ElevatedButton(
                        onPressed: () {},
                        child: const Text('보고서 내보내기'),
                      ),
                    ),
                  ],
                ),
              ),
            ),
          ],
        ),
      ),
    );
  }
}

class _SearchField extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return KeyboardListener(
      focusNode: FocusNode(),
      onKeyEvent: (event) {
        // Escape 키로 포커스 해제
        if (event is KeyDownEvent &&
            event.logicalKey == LogicalKeyboardKey.escape) {
          FocusScope.of(context).unfocus();
        }
      },
      child: const TextField(
        decoration: InputDecoration(
          hintText: '검색...',
          prefixIcon: Icon(Icons.search),
          border: OutlineInputBorder(),
        ),
      ),
    );
  }
}

class _SidebarItem extends StatelessWidget {
  final String label;
  final int order;

  const _SidebarItem({required this.label, required this.order});

  @override
  Widget build(BuildContext context) {
    return Focus(
      child: Builder(
        builder: (context) {
          final isFocused = Focus.of(context).hasPrimaryFocus;
          return ListTile(
            tileColor: isFocused ? Colors.blue.shade50 : null,
            title: Text(label),
            onTap: () {},
          );
        },
      ),
    );
  }
}

class _DashboardCard extends StatelessWidget {
  final String title;
  final String value;

  const _DashboardCard({required this.title, required this.value});

  @override
  Widget build(BuildContext context) {
    return Focus(
      child: Builder(
        builder: (context) {
          final isFocused = Focus.of(context).hasPrimaryFocus;
          return Card(
            elevation: isFocused ? 8 : 1,
            shape: RoundedRectangleBorder(
              borderRadius: BorderRadius.circular(8),
              side: isFocused
                  ? const BorderSide(color: Colors.blue, width: 2)
                  : BorderSide.none,
            ),
            child: Padding(
              padding: const EdgeInsets.all(16),
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Text(title,
                      style: Theme.of(context).textTheme.bodySmall),
                  const SizedBox(height: 8),
                  Text(value,
                      style: Theme.of(context).textTheme.headlineSmall),
                ],
              ),
            ),
          );
        },
      ),
    );
  }
}
```

**이 예제의 핵심:**
- `FocusTraversalGroup`은 Tab 키 순회 범위를 격리합니다. 왼쪽 사이드바 그룹과 오른쪽 콘텐츠 그룹이 각자 독립적으로 순회됩니다.
- `OrderedTraversalPolicy` + `FocusTraversalOrder(order: NumericFocusOrder(n))`로 UI에 보이는 시각적 순서와 무관하게 임의의 탐색 순서를 강제할 수 있습니다.
- `Focus` 위젯 안에서 `Builder`를 사용해 `Focus.of(context).hasPrimaryFocus`를 읽으면, 포커스 변경 시 해당 위젯만 리빌드됩니다. 상위 위젯 전체를 리빌드하는 것보다 훨씬 효율적입니다.

---

## 4. 주의사항 및 고급 팁

### Tip 1: FocusNode에 직접 onKeyEvent 등록하지 말 것

`Focus` 또는 `FocusScope` 위젯이 이미 관리하는 `FocusNode`에 직접 `onKeyEvent` 콜백을 추가하면 위젯 트리와 충돌이 발생할 수 있습니다. 키보드 이벤트는 반드시 `Focus` 위젯의 `onKeyEvent` 프로퍼티나 `KeyboardListener`, `RawKeyboardListener`를 사용하세요.

```dart
// ❌ 잘못된 방식 - FocusNode에 직접 콜백 등록
final node = FocusNode()
  ..onKeyEvent = (_, event) {
    return KeyEventResult.ignored;
  };

// ✅ 올바른 방식 - Focus 위젯의 onKeyEvent 사용
Focus(
  onKeyEvent: (node, event) {
    if (event is KeyDownEvent &&
        event.logicalKey == LogicalKeyboardKey.enter) {
      // 처리 완료, 이벤트 소비
      return KeyEventResult.handled;
    }
    return KeyEventResult.ignored;
  },
  child: const TextField(...),
)
```

### Tip 2: dispose() 누락은 메모리 누수의 주범

`FocusNode`는 리스너를 등록하는 `ChangeNotifier`의 서브클래스입니다. `dispose()` 없이 버려지면 리스너가 해제되지 않아 메모리 누수가 발생합니다. `AutomaticKeepAliveClientMixin`을 사용하는 화면에서는 특히 주의하세요.

```dart
@override
void dispose() {
  _myFocusNode.removeListener(_onFocusChange); // 리스너 먼저 제거
  _myFocusNode.dispose();                      // 그 다음 dispose
  super.dispose();
}
```

### Tip 3: skipTraversal과 canRequestFocus 활용

포커스 순회에서 특정 위젯을 제외하고 싶거나, 포커스를 받을 수 없게 만들고 싶을 때 사용합니다.

```dart
// 순회는 되지 않지만 직접 클릭으로는 포커스 가능
Focus(
  skipTraversal: true,
  child: IconButton(icon: const Icon(Icons.info), onPressed: () {}),
)

// 완전히 포커스 불가 (장식용 UI)
Focus(
  canRequestFocus: false,
  child: const Text('이 텍스트는 포커스를 받지 않음'),
)
```

### Tip 4: FocusManager.instance로 전역 포커스 감지

현재 앱 전체에서 어떤 노드가 포커스를 가지고 있는지 감지하고 싶을 때는 `FocusManager`의 리스너를 등록합니다. 분석 이벤트 추적이나 디버깅에 유용합니다.

```dart
@override
void initState() {
  super.initState();
  FocusManager.instance.addListener(_onFocusChange);
}

void _onFocusChange() {
  final focused = FocusManager.instance.primaryFocus;
  debugPrint('현재 포커스: ${focused?.debugLabel}');
}

@override
void dispose() {
  FocusManager.instance.removeListener(_onFocusChange);
  super.dispose();
}
```

### Tip 5: 첫 렌더링 후 자동 포커스

`autofocus: true` 속성을 `TextField`나 `Focus` 위젯에 사용하면 화면이 처음 나타날 때 자동으로 포커스가 이동합니다. 단, 로그인 화면처럼 항상 같은 필드에 포커스가 있어야 하는 경우에만 사용하세요. 메인 화면처럼 다양한 콘텐츠가 있는 화면에서 `autofocus`를 남발하면 사용자가 의도치 않게 소프트 키보드를 보게 되는 UX 문제가 생깁니다.

```dart
TextField(
  autofocus: true, // 화면 진입 즉시 포커스
  decoration: const InputDecoration(labelText: '이메일'),
)
```

---

## 마치며

Flutter의 포커스 시스템은 단순한 텍스트 입력 처리를 넘어, 접근성·키보드 탐색·폼 UX의 핵심을 이루는 중요한 인프라입니다. `FocusNode`와 `FocusScope`의 관계를 이해하고, `FocusTraversalGroup`으로 복잡한 레이아웃의 탐색 순서를 명시적으로 제어하면, 모바일부터 데스크톱까지 모든 플랫폼에서 일관된 고품질 UX를 제공할 수 있습니다.

특히 멀티플랫폼 Flutter 앱을 개발할 때는 초기 설계 단계부터 포커스 흐름을 고려하는 것을 강력히 권장합니다. 나중에 포커스 순서를 고치는 것은 처음부터 올바르게 설계하는 것보다 훨씬 많은 비용이 듭니다.

## 참고 자료
- [Understanding Flutter's keyboard focus system](https://docs.flutter.dev/ui/interactivity/focus)
- [FocusNode class - Flutter API](https://api.flutter.dev/flutter/widgets/FocusNode-class.html)
- [FocusTraversalGroup class - Flutter API](https://api.flutter.dev/flutter/widgets/FocusTraversalGroup-class.html)
- [FocusTraversalPolicy class - Flutter API](https://api.flutter.dev/flutter/widgets/FocusTraversalPolicy-class.html)
