---
layout: post
title: "Flutter Adaptive Layout 심화: WindowSizeClass·AdaptiveScaffold·Foldable 완전 정복"
date: 2026-07-08
categories: [android, flutter]
tags: [flutter, adaptive-layout, responsive, windowsizeclass, material3, foldable, adaptivescaffold]
---

스마트폰 한 화면만 고려하던 시대는 지났습니다. 태블릿, 폴더블, 크롬북, 데스크톱 웹까지—Flutter 앱은 이제 한 코드베이스로 수십 가지 화면 크기와 폼팩터를 우아하게 지원해야 합니다. 이 글에서는 **Responsive(반응형)** 과 **Adaptive(적응형)** 의 차이부터 시작해, Material 3의 WindowSizeClass 기반 레이아웃 분기, `adaptive_scaffold_plus`를 활용한 내비게이션 자동 전환, 폴더블 기기의 힌지(hinge) 대응까지 실전 코드와 함께 완전히 파헤칩니다.

---

## 1. Responsive vs Adaptive: 무엇이 다른가

두 개념은 자주 혼용되지만 의미가 다릅니다.

- **Responsive(반응형)**: 화면 크기가 바뀌면 동일한 레이아웃을 비율에 맞게 **늘리거나 줄이는** 것. `Flexible`, `Expanded`, `FractionallySizedBox`처럼 크기를 백분율로 지정하는 방식이 대표적입니다.
- **Adaptive(적응형)**: 화면 크기나 플랫폼에 따라 **완전히 다른 레이아웃·컴포넌트를 렌더링**하는 것. 폰에서는 하단 `NavigationBar`, 태블릿에서는 측면 `NavigationRail`, 데스크톱에서는 `NavigationDrawer`를 보여주는 것이 전형적인 예입니다.

실제 고품질 앱은 두 전략을 모두 씁니다. 각 화면 크기 범위 안에서는 Responsive로 세부 조정하고, 범위 경계를 넘을 때는 Adaptive로 레이아웃 자체를 교체합니다.

---

## 2. Material 3 WindowSizeClass: 공식 화면 분류 기준

Material Design 3은 화면 너비를 세 가지 **WindowSizeClass**로 분류합니다.

| 클래스 | 너비 범위 | 대표 기기 |
|--------|----------|----------|
| **Compact** | 0 – 599dp | 세로 방향 스마트폰 |
| **Medium** | 600 – 839dp | 폴더블(펼침), 소형 태블릿, 가로 방향 폰 |
| **Expanded** | 840dp 이상 | 대형 태블릿, 데스크톱, 크롬북 |

Flutter SDK 3.13부터 `MediaQuery.sizeOf(context).width`로 직접 분기하거나, 공식 패키지(`adaptive_scaffold_plus` 등)의 `Breakpoints`를 사용할 수 있습니다.

---

## 3. 직접 구현: WindowSizeClass 유틸리티와 레이아웃 분기

가장 기본적인 방법으로, 직접 `WindowSizeClass`를 정의하고 레이아웃을 분기하는 패턴입니다. 외부 패키지 없이도 충분히 구현할 수 있습니다.

```dart
// lib/core/window_size_class.dart

enum WindowSizeClass { compact, medium, expanded }

extension WindowSizeClassX on BuildContext {
  WindowSizeClass get windowSizeClass {
    final width = MediaQuery.sizeOf(this).width;
    if (width < 600) return WindowSizeClass.compact;
    if (width < 840) return WindowSizeClass.medium;
    return WindowSizeClass.expanded;
  }
}
```

```dart
// lib/ui/home/home_screen.dart

class HomeScreen extends StatefulWidget {
  const HomeScreen({super.key});
  @override
  State<HomeScreen> createState() => _HomeScreenState();
}

class _HomeScreenState extends State<HomeScreen> {
  int _selectedIndex = 0;

  static const _destinations = [
    NavigationDestination(icon: Icon(Icons.home_outlined),  selectedIcon: Icon(Icons.home),  label: '홈'),
    NavigationDestination(icon: Icon(Icons.search_outlined), selectedIcon: Icon(Icons.search), label: '탐색'),
    NavigationDestination(icon: Icon(Icons.person_outlined), selectedIcon: Icon(Icons.person), label: '프로필'),
  ];

  @override
  Widget build(BuildContext context) {
    final sizeClass = context.windowSizeClass;

    return switch (sizeClass) {
      WindowSizeClass.compact  => _CompactLayout(
          destinations: _destinations,
          selectedIndex: _selectedIndex,
          onDestinationSelected: (i) => setState(() => _selectedIndex = i),
          body: _buildBody(),
        ),
      WindowSizeClass.medium   => _MediumLayout(
          destinations: _destinations,
          selectedIndex: _selectedIndex,
          onDestinationSelected: (i) => setState(() => _selectedIndex = i),
          body: _buildBody(),
        ),
      WindowSizeClass.expanded => _ExpandedLayout(
          destinations: _destinations,
          selectedIndex: _selectedIndex,
          onDestinationSelected: (i) => setState(() => _selectedIndex = i),
          body: _buildBody(),
        ),
    };
  }

  Widget _buildBody() {
    return const [
      _HomeTab(),
      _SearchTab(),
      _ProfileTab(),
    ][_selectedIndex];
  }
}

// ── Compact: 하단 NavigationBar ──────────────────────────────────
class _CompactLayout extends StatelessWidget {
  final List<NavigationDestination> destinations;
  final int selectedIndex;
  final ValueChanged<int> onDestinationSelected;
  final Widget body;

  const _CompactLayout({
    required this.destinations,
    required this.selectedIndex,
    required this.onDestinationSelected,
    required this.body,
  });

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: body,
      bottomNavigationBar: NavigationBar(
        destinations: destinations,
        selectedIndex: selectedIndex,
        onDestinationSelected: onDestinationSelected,
      ),
    );
  }
}

// ── Medium: 측면 NavigationRail ──────────────────────────────────
class _MediumLayout extends StatelessWidget {
  final List<NavigationDestination> destinations;
  final int selectedIndex;
  final ValueChanged<int> onDestinationSelected;
  final Widget body;

  const _MediumLayout({
    required this.destinations,
    required this.selectedIndex,
    required this.onDestinationSelected,
    required this.body,
  });

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Row(
        children: [
          NavigationRail(
            destinations: destinations
                .map((d) => NavigationRailDestination(
                      icon: d.icon,
                      selectedIcon: d.selectedIcon,
                      label: Text(d.label),
                    ))
                .toList(),
            selectedIndex: selectedIndex,
            onDestinationSelected: onDestinationSelected,
            labelType: NavigationRailLabelType.selected,
          ),
          const VerticalDivider(thickness: 1, width: 1),
          Expanded(child: body),
        ],
      ),
    );
  }
}

// ── Expanded: 영구 NavigationDrawer ─────────────────────────────
class _ExpandedLayout extends StatelessWidget {
  final List<NavigationDestination> destinations;
  final int selectedIndex;
  final ValueChanged<int> onDestinationSelected;
  final Widget body;

  const _ExpandedLayout({
    required this.destinations,
    required this.selectedIndex,
    required this.onDestinationSelected,
    required this.body,
  });

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Row(
        children: [
          NavigationDrawer(
            selectedIndex: selectedIndex,
            onDestinationSelected: onDestinationSelected,
            children: [
              const DrawerHeader(child: FlutterLogo(size: 48)),
              ...destinations.map((d) => NavigationDrawerDestination(
                    icon: d.icon,
                    selectedIcon: d.selectedIcon,
                    label: Text(d.label),
                  )),
            ],
          ),
          Expanded(child: body),
        ],
      ),
    );
  }
}
```

위 코드에서 `switch` 표현식(Dart 3.0+)으로 `WindowSizeClass`에 따라 세 가지 레이아웃 위젯을 교체합니다. `MediaQuery.sizeOf()`는 `MediaQuery.of(context).size`와 달리 **사이즈 변경 시에만 리빌드**를 트리거하므로 성능이 더 좋습니다.

---

## 4. AdaptiveScaffold를 활용한 자동 전환

`adaptive_scaffold_plus` 패키지를 사용하면 위의 레이아웃 분기 로직을 대부분 패키지가 처리해 줍니다. 내비게이션 전환 애니메이션도 자동으로 포함됩니다.

```yaml
# pubspec.yaml
dependencies:
  adaptive_scaffold_plus: ^1.0.9
```

```dart
// lib/ui/shell/app_shell.dart

import 'package:adaptive_scaffold_plus/adaptive_scaffold_plus.dart';
import 'package:flutter/material.dart';

class AppShell extends StatelessWidget {
  const AppShell({super.key});

  @override
  Widget build(BuildContext context) {
    return AdaptiveScaffold(
      // 내비게이션 항목: Material 3 NavigationDestination 그대로 사용
      destinations: const [
        NavigationDestination(
          icon: Icon(Icons.inbox_outlined),
          selectedIcon: Icon(Icons.inbox),
          label: '받은 메일',
        ),
        NavigationDestination(
          icon: Icon(Icons.article_outlined),
          selectedIcon: Icon(Icons.article),
          label: '피드',
        ),
        NavigationDestination(
          icon: Icon(Icons.chat_bubble_outlined),
          selectedIcon: Icon(Icons.chat_bubble),
          label: '메시지',
        ),
        NavigationDestination(
          icon: Icon(Icons.video_call_outlined),
          selectedIcon: Icon(Icons.video_call),
          label: '영상통화',
        ),
      ],
      // 화면 크기에 따라 다른 body 제공 (선택사항)
      body: (_) => const _InboxView(),
      secondaryBody: (_) => const _DetailView(),   // Medium 이상에서 우측 패널
      // 커스텀 브레이크포인트 오버라이드 (선택사항)
      smallBreakpoint: const Breakpoint(endWidth: 700),
      mediumBreakpoint: const Breakpoint(beginWidth: 700, endWidth: 1000),
      largeBreakpoint: const Breakpoint(beginWidth: 1000),
    );
  }
}

class _InboxView extends StatelessWidget {
  const _InboxView();

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      padding: const EdgeInsets.all(8),
      itemCount: 20,
      itemBuilder: (_, i) => Card(
        child: ListTile(
          leading: const CircleAvatar(child: Icon(Icons.person)),
          title: Text('메일 $i'),
          subtitle: const Text('내용 미리보기...'),
          trailing: Text('${i}분 전'),
        ),
      ),
    );
  }
}

class _DetailView extends StatelessWidget {
  const _DetailView();

  @override
  Widget build(BuildContext context) {
    return const Center(
      child: Text('메일을 선택하세요', style: TextStyle(fontSize: 18)),
    );
  }
}
```

`AdaptiveScaffold`의 `secondaryBody`를 지정하면 Medium/Large 크기에서 자동으로 2-패널 레이아웃(list-detail 패턴)을 구성합니다. 기본 내비게이션 전환 애니메이션도 내장되어 있어 별도 구현이 불필요합니다.

---

## 5. 폴더블 기기 대응: DisplayFeature와 힌지 회피

폴더블 기기는 화면 중앙에 **힌지(hinge)** 또는 **폴드(fold)** 영역이 존재합니다. 이 영역에 중요한 UI 요소(버튼, 텍스트 입력)가 겹치면 사용자 경험이 크게 저하됩니다.

Flutter는 `MediaQueryData.displayFeatures`를 통해 힌지 위치와 종류를 제공합니다. 또한 `TwoPane` 위젯 계열을 사용하면 힌지를 기준으로 레이아웃을 자동으로 분리할 수 있습니다.

```dart
// lib/ui/fold_aware/fold_aware_layout.dart

import 'dart:ui' show DisplayFeatureType;
import 'package:flutter/material.dart';

/// 폴더블 힌지가 있으면 힌지를 기준으로 두 패널을 분리하고,
/// 없으면 일반 좌우 분할 레이아웃을 사용합니다.
class FoldAwareLayout extends StatelessWidget {
  final Widget primaryPanel;
  final Widget secondaryPanel;

  const FoldAwareLayout({
    super.key,
    required this.primaryPanel,
    required this.secondaryPanel,
  });

  @override
  Widget build(BuildContext context) {
    final mediaQuery = MediaQuery.of(context);
    final hingeFeature = mediaQuery.displayFeatures.where((f) =>
      f.type == DisplayFeatureType.hinge ||
      f.type == DisplayFeatureType.fold,
    ).firstOrNull;

    if (hingeFeature != null) {
      // 힌지가 세로축(수직 분할) 위치에 있는 경우
      final hingeLeft  = hingeFeature.bounds.left;
      final hingeRight = hingeFeature.bounds.right;
      final totalWidth = mediaQuery.size.width;

      return Row(
        children: [
          SizedBox(width: hingeLeft,                   child: primaryPanel),
          SizedBox(width: hingeRight - hingeLeft),     // 힌지 영역 빈 공간
          SizedBox(width: totalWidth - hingeRight,     child: secondaryPanel),
        ],
      );
    }

    // 힌지가 없으면 일반 50:50 분할
    return Row(
      children: [
        Expanded(child: primaryPanel),
        const VerticalDivider(thickness: 1, width: 1),
        Expanded(child: secondaryPanel),
      ],
    );
  }
}
```

```dart
// 사용 예시
class ArticleScreen extends StatelessWidget {
  const ArticleScreen({super.key});

  @override
  Widget build(BuildContext context) {
    final sizeClass = context.windowSizeClass;

    if (sizeClass == WindowSizeClass.compact) {
      return const Scaffold(body: _ArticleListView());
    }

    return Scaffold(
      body: FoldAwareLayout(
        primaryPanel: const _ArticleListView(),
        secondaryPanel: const _ArticleDetailView(),
      ),
    );
  }
}
```

`displayFeatures`는 실제 폴더블 기기나 안드로이드 에뮬레이터의 폴더블 모드에서만 값이 채워집니다. 일반 기기에서는 빈 리스트가 반환되므로 폴백(fallback) 로직이 반드시 필요합니다.

---

## 6. 핵심 패턴: List-Detail (master-detail) 구현

이메일 앱, 설정 앱처럼 목록을 클릭하면 우측에 상세가 열리는 패턴이 대표적인 적응형 UI입니다. Compact에서는 목록 → 상세 순으로 화면 전환을 하고, Medium/Expanded에서는 두 화면을 나란히 보여줍니다.

```dart
// lib/ui/settings/settings_screen.dart

class SettingsScreen extends StatefulWidget {
  const SettingsScreen({super.key});
  @override
  State<SettingsScreen> createState() => _SettingsScreenState();
}

class _SettingsScreenState extends State<SettingsScreen> {
  String? _selectedKey;

  final _items = const [
    ('계정',    '프로필, 보안, 개인정보'),
    ('알림',    '푸시, 이메일, SMS'),
    ('디스플레이', '테마, 글꼴 크기'),
    ('데이터',   '저장 공간, 캐시'),
    ('정보',    '버전, 라이선스'),
  ];

  @override
  Widget build(BuildContext context) {
    final sizeClass = context.windowSizeClass;
    final isExpanded = sizeClass != WindowSizeClass.compact;

    if (isExpanded) {
      return Scaffold(
        body: Row(
          children: [
            SizedBox(
              width: 300,
              child: _buildList(),
            ),
            const VerticalDivider(thickness: 1, width: 1),
            Expanded(
              child: _selectedKey != null
                  ? _SettingDetailPage(settingKey: _selectedKey!)
                  : const Center(child: Text('항목을 선택하세요')),
            ),
          ],
        ),
      );
    }

    // Compact: 목록만 표시, 탭 시 Navigator로 이동
    return Scaffold(
      appBar: AppBar(title: const Text('설정')),
      body: _buildList(),
    );
  }

  Widget _buildList() {
    return ListView(
      children: _items.map((item) {
        final (title, subtitle) = item;
        return ListTile(
          title: Text(title),
          subtitle: Text(subtitle),
          selected: _selectedKey == title,
          onTap: () {
            final sizeClass = context.windowSizeClass;
            if (sizeClass == WindowSizeClass.compact) {
              Navigator.push(
                context,
                MaterialPageRoute(
                  builder: (_) => _SettingDetailPage(settingKey: title),
                ),
              );
            } else {
              setState(() => _selectedKey = title);
            }
          },
        );
      }).toList(),
    );
  }
}

class _SettingDetailPage extends StatelessWidget {
  final String settingKey;
  const _SettingDetailPage({required this.settingKey});

  @override
  Widget build(BuildContext context) {
    return Center(
      child: Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          const Icon(Icons.settings, size: 64, color: Colors.grey),
          const SizedBox(height: 16),
          Text(settingKey, style: Theme.of(context).textTheme.headlineMedium),
          const SizedBox(height: 8),
          const Text('상세 설정 항목이 여기에 표시됩니다.'),
        ],
      ),
    );
  }
}
```

---

## 7. 테스트: 다양한 화면 크기 시뮬레이션

적응형 레이아웃은 반드시 여러 화면 크기에서 테스트해야 합니다. `flutter test`에서 `MediaQuery`를 오버라이드하는 방법으로 각 크기 클래스를 시뮬레이션할 수 있습니다.

```dart
// test/home_screen_test.dart

import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';

Widget buildWithSize(Widget child, Size size) {
  return MaterialApp(
    home: MediaQuery(
      data: MediaQueryData(size: size),
      child: child,
    ),
  );
}

void main() {
  testWidgets('Compact: NavigationBar 표시', (tester) async {
    await tester.pumpWidget(
      buildWithSize(const HomeScreen(), const Size(390, 844)),
    );
    expect(find.byType(NavigationBar), findsOneWidget);
    expect(find.byType(NavigationRail), findsNothing);
  });

  testWidgets('Medium: NavigationRail 표시', (tester) async {
    await tester.pumpWidget(
      buildWithSize(const HomeScreen(), const Size(768, 1024)),
    );
    expect(find.byType(NavigationRail), findsOneWidget);
    expect(find.byType(NavigationBar), findsNothing);
  });

  testWidgets('Expanded: NavigationDrawer 표시', (tester) async {
    await tester.pumpWidget(
      buildWithSize(const HomeScreen(), const Size(1280, 800)),
    );
    expect(find.byType(NavigationDrawer), findsOneWidget);
    expect(find.byType(NavigationRail), findsNothing);
  });
}
```

---

## 8. 주의사항과 실전 팁

**1. `MediaQuery.sizeOf()` vs `MediaQuery.of().size`**
`MediaQuery.sizeOf(context)`는 Flutter 3.10에서 추가된 API로, 사이즈 변경 시에만 의존 위젯을 리빌드합니다. `MediaQuery.of(context).size`는 padding, textScale 등 다른 MediaQuery 속성이 바뀌어도 리빌드를 유발합니다. 성능 최적화를 위해 항상 `sizeOf`를 사용하세요.

**2. 브레이크포인트를 너무 많이 만들지 않기**
Custom breakpoint를 세밀하게 정의할수록 유지보수 비용이 급증합니다. Material 3의 세 가지 클래스(Compact / Medium / Expanded)가 대부분의 앱에서 충분합니다. 예외가 있다면 특정 컴포넌트 단위로만 커스텀 브레이크포인트를 적용하세요.

**3. 레이아웃 전환 시 상태 보존**
`NavigationBar`에서 `NavigationRail`로 전환될 때 위젯 트리가 교체되므로, 상태(스크롤 위치, 선택된 항목 등)가 초기화될 수 있습니다. `PageStorageKey`나 상위 상태 관리(Riverpod, BLoC)로 상태를 끌어올려 보존하세요.

**4. 폴더블 기기 에뮬레이터 활용**
Android Studio 에뮬레이터에서 **Resizable Emulator**(Pixel Fold, Galaxy Fold 등)를 사용하면 힌지 펼침/접힘 상태를 시뮬레이션할 수 있습니다. `DisplayFeatureType.hinge`와 `fold`가 실제로 반환되는지 확인하면서 개발하세요.

**5. 가로/세로 방향 전환도 WindowSizeClass를 바꿈**
폰을 가로로 돌리면 너비가 800dp를 넘어 Expanded로 분류될 수 있습니다. 이때 갑자기 `NavigationDrawer`가 나타나는 것은 원하지 않을 수 있으므로, 필요하다면 `MediaQuery.orientationOf(context)`와 조합해 별도 처리하세요.

**6. 접근성(Accessibility) 고려**
`NavigationRail`과 `NavigationDrawer`는 `NavigationBar`보다 터치 타깃이 작아질 수 있습니다. `NavigationRailDestination`의 `label`을 항상 포함하고, 충분한 여백(`padding`)을 확보하세요.

---

## 정리

Flutter에서 진정한 적응형 UI를 구현하는 핵심은 다음 세 가지입니다.

1. **WindowSizeClass** (Compact / Medium / Expanded) 기반으로 레이아웃 트리를 분기한다.
2. **AdaptiveScaffold** 패키지를 사용하면 내비게이션 전환·애니메이션·2패널 레이아웃을 최소한의 코드로 얻는다.
3. **폴더블 기기**는 `MediaQuery.displayFeatures`로 힌지 위치를 감지해 UI 요소가 힌지 위에 겹치지 않도록 보호한다.

단일 코드베이스로 폰부터 데스크톱까지 커버하는 앱을 만드는 일은 더 이상 특별한 마법이 아닙니다. Material 3의 가이드라인을 따르고, 위 패턴을 조합하면 어떤 폼팩터에서도 쾌적한 사용자 경험을 제공할 수 있습니다.

---

## 참고 자료
- [adaptive_scaffold_plus 패키지 (pub.dev)](https://pub.dev/packages/adaptive_scaffold_plus)
- [flutter_adaptive_scaffold 패키지 — 공식 원본 (현재 deprecated)](https://pub.dev/packages/flutter_adaptive_scaffold)
- [flutter_layout_grid — CSS Grid 수준의 Flutter 레이아웃](https://pub.dev/packages/flutter_layout_grid)
