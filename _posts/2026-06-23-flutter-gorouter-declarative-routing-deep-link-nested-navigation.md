---
layout: post
title: "Flutter GoRouter 심화: 선언형 라우팅·Deep Link·중첩 네비게이션 완전 정복"
date: 2026-06-23
categories: [flutter]
tags: [flutter, go_router, navigation, deep-link, shell-route, declarative-routing]
---

Flutter 앱이 커질수록 `Navigator.push()` 기반의 명령형(Imperative) 네비게이션은 빠르게 한계를 드러냅니다. 딥 링크 처리, 인증 상태에 따른 자동 리다이렉트, 바텀 네비게이션 바와 함께하는 중첩 네비게이션 — 이 모든 시나리오를 명령형 방식으로 구현하면 코드는 곧 스파게티가 됩니다. Flutter 팀이 공식 권장 라우팅 라이브러리로 지정한 **GoRouter**는 이 문제를 URL 기반 선언형 방식으로 깔끔하게 해결합니다. 이 글에서는 GoRouter의 핵심 개념부터 `ShellRoute`를 활용한 중첩 네비게이션, 딥 링크 처리, 그리고 실전에서 마주치는 함정까지 깊이 있게 파헤칩니다.

---

## GoRouter란 무엇인가

GoRouter는 Flutter의 `Router` API(Navigator 2.0)를 추상화하여 URL 기반의 선언적 라우팅을 제공하는 패키지입니다. 2023년부터 `flutter.dev` 가 직접 publish하며 Flutter 공식 패키지 저장소(`flutter/packages`)에서 관리됩니다. 2026년 현재 최신 버전은 **17.3.0**입니다.

```
go_router: ^17.3.0
```

### Navigator 1.0 vs GoRouter

| 비교 항목 | Navigator 1.0 | GoRouter |
|---|---|---|
| 라우팅 방식 | 명령형 (push/pop) | 선언형 (URL 기반) |
| 딥 링크 | 별도 구현 필요 | 자동 지원 |
| 웹/데스크탑 URL | 불완전 | 완전 지원 |
| 중첩 네비게이션 | 복잡 | ShellRoute로 간결 |
| 인증 리다이렉트 | 수동 처리 | redirect 콜백으로 선언적 처리 |

---

## 왜 GoRouter가 필요한가

### 문제 1: 딥 링크 처리의 복잡성

사용자가 `myapp://products/42` 링크를 탭했을 때, 앱이 올바른 화면(상품 상세 #42)을 바로 열어야 합니다. Navigator 1.0으로 이를 구현하려면 플랫폼별 인텐트 필터 설정뿐 아니라 앱 시작 시점의 라우팅 로직을 별도로 작성해야 합니다. GoRouter는 URL 패턴만 선언하면 플랫폼 딥 링크가 자동으로 해당 화면을 빌드합니다.

### 문제 2: 인증 흐름 관리

로그인하지 않은 사용자가 `/home`에 접근하려 할 때 `/login`으로 보내야 합니다. Navigator 1.0에서는 각 화면 진입 시마다 이 체크를 수동으로 넣어야 합니다. GoRouter의 `redirect` 콜백은 상태 변화에 반응하여 전역적으로 처리해 줍니다.

### 문제 3: 바텀 네비게이션 바와의 상태 유지

탭을 전환할 때 각 탭의 스크롤 위치나 서브 라우트 스택을 유지해야 합니다. `ShellRoute` / `StatefulShellRoute`가 이 문제를 우아하게 해결합니다.

---

## 실제 구현 예제

### 예제 1: 기본 라우팅 + 리다이렉트

아래는 인증 상태에 따른 리다이렉트와 경로 파라미터를 포함한 GoRouter 기본 설정입니다.

```dart
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';

// 인증 상태를 관리하는 간단한 ChangeNotifier
class AuthState extends ChangeNotifier {
  bool _isLoggedIn = false;
  bool get isLoggedIn => _isLoggedIn;

  void login() {
    _isLoggedIn = true;
    notifyListeners();
  }

  void logout() {
    _isLoggedIn = false;
    notifyListeners();
  }
}

final authState = AuthState();

final GoRouter router = GoRouter(
  initialLocation: '/home',
  // authState 변경 시 redirect가 재평가되도록 refreshListenable 등록
  refreshListenable: authState,
  redirect: (BuildContext context, GoRouterState state) {
    final isLoggedIn = authState.isLoggedIn;
    final isGoingToLogin = state.matchedLocation == '/login';

    if (!isLoggedIn && !isGoingToLogin) {
      // 로그인 후 원래 목적지로 돌아오기 위해 from 파라미터 전달
      return '/login?from=${state.matchedLocation}';
    }
    if (isLoggedIn && isGoingToLogin) {
      return '/home';
    }
    return null; // 리다이렉트 없음
  },
  routes: [
    GoRoute(
      path: '/login',
      builder: (context, state) {
        final from = state.uri.queryParameters['from'] ?? '/home';
        return LoginScreen(redirectTo: from);
      },
    ),
    GoRoute(
      path: '/home',
      builder: (context, state) => const HomeScreen(),
    ),
    GoRoute(
      // :id는 경로 파라미터 — /products/42 형식
      path: '/products/:id',
      builder: (context, state) {
        final productId = state.pathParameters['id']!;
        return ProductDetailScreen(productId: productId);
      },
    ),
    GoRoute(
      path: '/search',
      builder: (context, state) {
        // 쿼리 파라미터: /search?q=flutter
        final query = state.uri.queryParameters['q'] ?? '';
        return SearchScreen(initialQuery: query);
      },
    ),
  ],
);

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp.router(
      routerConfig: router,
      title: 'GoRouter Demo',
    );
  }
}

// 화면에서 네비게이션하는 방법
// context.go('/products/42')       — 스택을 교체 (뒤로 가기 불가)
// context.push('/products/42')     — 스택에 추가 (뒤로 가기 가능)
// context.replace('/products/42')  — 현재 화면을 교체
// context.pop()                    — 뒤로 가기
```

`refreshListenable`은 GoRouter의 핵심 기능 중 하나입니다. `ChangeNotifier`를 등록하면 상태 변화(로그인/로그아웃) 시 GoRouter가 자동으로 `redirect`를 재평가합니다. `Riverpod`을 사용하는 경우엔 `GoRouterRefreshStream`으로 `Stream`을 연결할 수 있습니다.

---

### 예제 2: StatefulShellRoute를 이용한 바텀 네비게이션 + 중첩 네비게이션

`StatefulShellRoute`는 각 탭의 네비게이터 스택을 독립적으로 유지합니다. 탭 A에서 `/home/detail`로 들어간 상태에서 탭 B로 전환했다가 다시 탭 A로 돌아오면 `/home/detail`이 그대로 남아 있습니다.

```dart
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';

final _rootNavigatorKey = GlobalKey<NavigatorState>();
final _homeTabKey = GlobalKey<NavigatorState>(debugLabel: 'homeTab');
final _searchTabKey = GlobalKey<NavigatorState>(debugLabel: 'searchTab');
final _profileTabKey = GlobalKey<NavigatorState>(debugLabel: 'profileTab');

final GoRouter appRouter = GoRouter(
  navigatorKey: _rootNavigatorKey,
  initialLocation: '/home',
  routes: [
    StatefulShellRoute.indexedStack(
      builder: (context, state, navigationShell) {
        // navigationShell이 각 탭의 상태를 관리
        return ScaffoldWithNavBar(navigationShell: navigationShell);
      },
      branches: [
        // 탭 1: 홈
        StatefulShellBranch(
          navigatorKey: _homeTabKey,
          routes: [
            GoRoute(
              path: '/home',
              builder: (context, state) => const HomeTab(),
              routes: [
                // 홈 탭 내부의 서브 라우트
                GoRoute(
                  path: 'detail/:id',
                  // 루트 네비게이터에 표시 — 탭 바 없이 전체 화면
                  parentNavigatorKey: _rootNavigatorKey,
                  builder: (context, state) {
                    final id = state.pathParameters['id']!;
                    return DetailScreen(id: id);
                  },
                ),
              ],
            ),
          ],
        ),
        // 탭 2: 검색
        StatefulShellBranch(
          navigatorKey: _searchTabKey,
          routes: [
            GoRoute(
              path: '/search',
              builder: (context, state) => const SearchTab(),
            ),
          ],
        ),
        // 탭 3: 프로필
        StatefulShellBranch(
          navigatorKey: _profileTabKey,
          routes: [
            GoRoute(
              path: '/profile',
              builder: (context, state) => const ProfileTab(),
              routes: [
                GoRoute(
                  path: 'edit',
                  builder: (context, state) => const EditProfileScreen(),
                ),
              ],
            ),
          ],
        ),
      ],
    ),
  ],
);

// 바텀 네비게이션 바 Scaffold
class ScaffoldWithNavBar extends StatelessWidget {
  final StatefulNavigationShell navigationShell;

  const ScaffoldWithNavBar({super.key, required this.navigationShell});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: navigationShell,
      bottomNavigationBar: BottomNavigationBar(
        currentIndex: navigationShell.currentIndex,
        onTap: (index) {
          // goBranch로 탭 전환; initialLocation: true를 주면
          // 이미 해당 탭에 있을 때 루트 경로로 리셋
          navigationShell.goBranch(
            index,
            initialLocation: index == navigationShell.currentIndex,
          );
        },
        items: const [
          BottomNavigationBarItem(icon: Icon(Icons.home), label: '홈'),
          BottomNavigationBarItem(icon: Icon(Icons.search), label: '검색'),
          BottomNavigationBarItem(icon: Icon(Icons.person), label: '프로필'),
        ],
      ),
    );
  }
}
```

`parentNavigatorKey: _rootNavigatorKey`를 서브 라우트에 지정하면 해당 화면은 탭 쉘 바깥의 루트 네비게이터에 표시됩니다. 이를 통해 상품 상세, 결제 화면처럼 전체 화면을 덮어야 하는 라우트를 자연스럽게 처리할 수 있습니다.

---

## 딥 링크 플랫폼 설정

GoRouter가 URL을 처리하는 것 자체는 Dart 코드만으로 완결되지만, 외부에서 들어오는 딥 링크를 받으려면 플랫폼별 설정이 필요합니다.

### Android — AndroidManifest.xml

```xml
<activity ...>
  <intent-filter android:autoVerify="true">
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <!-- 커스텀 스킴 -->
    <data android:scheme="myapp" />
    <!-- 앱 링크 (HTTPS) -->
    <data android:scheme="https" android:host="example.com" />
  </intent-filter>
</activity>
```

앱 링크(HTTPS)를 사용하려면 `/.well-known/assetlinks.json`을 해당 도메인에 호스팅해야 합니다.

### iOS — Info.plist

```xml
<key>CFBundleURLTypes</key>
<array>
  <dict>
    <key>CFBundleURLSchemes</key>
    <array>
      <string>myapp</string>
    </array>
  </dict>
</array>
```

유니버설 링크를 쓰려면 `apple-app-site-association` 파일도 호스팅이 필요합니다.

---

## 주의사항 및 실전 팁

### 1. `go` vs `push` 의미 차이를 명확히 이해하라

`context.go('/home')`은 현재 네비게이션 스택 전체를 `/home` 하나로 교체합니다. 따라서 바텀 시트나 다이얼로그에서 `go`를 호출하면 뒤로 가기 버튼이 작동하지 않을 수 있습니다. 스택에 화면을 쌓아야 하는 경우에는 반드시 `context.push()`를 사용하세요.

### 2. `redirect` 루프에 주의하라

`redirect`에서 항상 특정 경로로 리다이렉트하도록 짜면 무한 루프가 발생합니다. 반드시 `null`을 반환하는 탈출 조건을 명시하세요. GoRouter는 루프 감지 기능이 있지만, 로직 오류 시 `FlutterError`를 던지므로 디버그 빌드에서 조기에 확인할 수 있습니다.

### 3. `GoRouterState`는 `of(context)`가 아니라 주입받아 써라

`GoRouterState.of(context)`는 매 빌드마다 위젯 트리를 타고 올라가므로 비용이 있습니다. `GoRoute`의 `builder` 콜백 파라미터로 전달되는 `state`를 직접 사용하거나, 화면 생성자에 필요한 데이터만 뽑아서 전달하는 방식을 선호하세요.

### 4. `StatefulShellRoute`와 Riverpod을 함께 쓸 때

각 탭이 독립적인 ProviderScope를 가져야 한다면, `StatefulShellBranch`의 `builder` 내부에 `ProviderScope`를 중첩하면 됩니다. 그러나 탭 간 공유 상태는 상위 ProviderScope에서 관리해야 한다는 점을 잊지 마세요.

### 5. 타입 안전 라우팅 — `TypedGoRoute`

GoRouter 14+ 부터 `go_router_builder` 패키지와 함께 타입 안전 라우트를 선언할 수 있습니다. 문자열 경로 대신 Dart 클래스로 라우트를 정의하여 컴파일 타임에 오류를 잡을 수 있습니다.

```dart
@TypedGoRoute<ProductRoute>(path: '/products/:id')
class ProductRoute extends GoRouteData {
  final int id;
  const ProductRoute({required this.id});

  @override
  Widget build(BuildContext context, GoRouterState state) {
    return ProductDetailScreen(productId: id.toString());
  }
}

// 사용
ProductRoute(id: 42).go(context);
```

이 방식은 경로 파라미터 타입 변환까지 자동으로 처리해 주므로, 중규모 이상의 프로젝트에서 특히 유용합니다.

---

## 정리

GoRouter는 단순히 `Navigator.push()`의 래퍼가 아닙니다. URL 기반 선언형 라우팅, 딥 링크 자동 처리, `StatefulShellRoute`를 통한 탭별 스택 분리, 그리고 `redirect` 콜백을 활용한 전역 인증 흐름까지 — Flutter 앱의 네비게이션 복잡도가 높아질수록 GoRouter의 가치는 극명하게 드러납니다. `go`와 `push`의 의미 차이를 정확히 이해하고, `refreshListenable`로 상태 변화를 연결하며, 필요에 따라 `TypedGoRoute`로 타입 안전성까지 확보한다면 어떤 규모의 앱도 깔끔하게 관리할 수 있습니다.

## 참고 자료
- [go_router | pub.dev](https://pub.dev/packages/go_router)
- [GoRouter Deep Linking | Dart API Docs](https://pub.dev/documentation/go_router/latest/topics/Deep%20linking-topic.html)
- [GoRouter Class Reference | Dart API Docs](https://pub.dev/documentation/go_router/latest/go_router/GoRouter-class.html)
- [ShellRoute Class Reference | Dart API Docs](https://pub.dev/documentation/go_router/latest/go_router/ShellRoute-class.html)
