---
layout: post
title: "Flutter webview_flutter 심화: WebViewController·JavaScriptChannel·NavigationDelegate·쿠키 관리 완전 정복"
date: 2026-10-21
categories: [android, flutter]
tags: [flutter, webview, webview_flutter, javascript-channel, navigation-delegate, cookie, platform-view]
---

웹 콘텐츠와 네이티브 Flutter 앱을 결합해야 하는 상황은 생각보다 자주 발생합니다. 기존 웹 서비스를 앱에 내장하거나, 사용자 약관·공지사항을 웹 페이지로 표시하거나, 결제 게이트웨이를 연동할 때 WebView가 필수적입니다. Flutter 생태계에서는 `webview_flutter` 패키지가 이 역할을 담당하며, Android는 시스템 `WebView`를, iOS는 `WKWebView`를 네이티브 내부 구현으로 활용합니다.

이 아티클에서는 단순한 URL 로딩을 넘어, `WebViewController`, `NavigationDelegate`, `JavaScriptChannel`, 쿠키 관리, 보안 설정까지 `webview_flutter 4.x` 기준으로 심도 있게 다룹니다.

## 왜 webview_flutter인가?

Flutter는 자체 렌더링 엔진(Skia/Impeller)으로 모든 UI를 그립니다. 그러나 웹 콘텐츠는 브라우저 렌더링 엔진이 필요합니다. Flutter가 자체적으로 HTML·CSS·JS를 파싱할 수 없기 때문에, **Platform View**를 통해 네이티브 WebView 컴포넌트를 Flutter 위젯 트리 안에 끼워 넣는 방식을 사용합니다.

`webview_flutter 4.0+`부터는 API 구조가 크게 바뀌었습니다:

- `WebViewController`가 독립된 컨트롤러로 분리되어 `WebViewWidget`과 느슨하게 결합됩니다.
- `JavascriptChannel` 오타 수정 → `JavaScriptChannel` (대소문자 변경)
- Platform 구현체를 플러그인으로 분리합니다 (`webview_flutter_android`, `webview_flutter_wkwebview`).
- `initialUrl` 방식 폐기, `loadRequest(Uri)` 방식으로 통일됩니다.

이 변화는 테스트 가능성과 플랫폼 확장성을 크게 향상시켰습니다.

## 환경 설정

`pubspec.yaml`에 다음을 추가합니다:

```yaml
dependencies:
  webview_flutter: ^4.10.0
  webview_flutter_android: ^4.3.0
  webview_flutter_wkwebview: ^3.14.0
```

**Android 설정** (`android/app/src/main/AndroidManifest.xml`):

```xml
<uses-permission android:name="android.permission.INTERNET" />
```

**iOS 설정** (`ios/Runner/Info.plist`):

```xml
<key>NSAppTransportSecurity</key>
<dict>
  <key>NSAllowsArbitraryLoads</key>
  <true/>
</dict>
```

Android는 최소 SDK 24 이상, iOS는 13.0 이상이 필요합니다. `minSdkVersion`이 24 미만이라면 `android/app/build.gradle`에서 수정하세요.

## 핵심 아키텍처: WebViewController와 WebViewWidget 분리

`webview_flutter 4.x`에서 가장 중요한 변화는 **컨트롤러와 위젯의 분리**입니다. 컨트롤러에서 모든 설정과 제어 로직을 담당하고, 위젯은 순수하게 렌더링만 담당합니다.

```dart
import 'package:flutter/material.dart';
import 'package:webview_flutter/webview_flutter.dart';

class MyWebViewPage extends StatefulWidget {
  final String url;
  const MyWebViewPage({super.key, required this.url});

  @override
  State<MyWebViewPage> createState() => _MyWebViewPageState();
}

class _MyWebViewPageState extends State<MyWebViewPage> {
  late final WebViewController _controller;
  double _loadingProgress = 0;
  bool _isLoading = true;

  @override
  void initState() {
    super.initState();

    _controller = WebViewController()
      ..setJavaScriptMode(JavaScriptMode.unrestricted)
      ..setBackgroundColor(Colors.white)
      ..setNavigationDelegate(
        NavigationDelegate(
          onProgress: (progress) {
            setState(() {
              _loadingProgress = progress / 100.0;
              _isLoading = progress < 100;
            });
          },
          onPageStarted: (url) {
            debugPrint('Page started: $url');
          },
          onPageFinished: (url) {
            debugPrint('Page finished: $url');
          },
          onNavigationRequest: (request) {
            // 커스텀 스킴(딥링크) 차단 — 네이티브에서 별도 처리
            if (request.url.startsWith('intent://') ||
                request.url.startsWith('market://') ||
                request.url.startsWith('kakaolink://')) {
              debugPrint('Blocking: ${request.url}');
              return NavigationDecision.prevent;
            }
            return NavigationDecision.navigate;
          },
          onWebResourceError: (error) {
            debugPrint('Resource error: ${error.description} (${error.errorCode})');
          },
          onHttpError: (error) {
            debugPrint('HTTP ${error.response?.statusCode}: ${error.request?.uri}');
          },
        ),
      )
      ..loadRequest(Uri.parse(widget.url));
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('WebView'),
        actions: [
          IconButton(
            icon: const Icon(Icons.refresh),
            onPressed: () => _controller.reload(),
          ),
        ],
        bottom: _isLoading
            ? PreferredSize(
                preferredSize: const Size.fromHeight(4),
                child: LinearProgressIndicator(value: _loadingProgress),
              )
            : null,
      ),
      body: PopScope(
        canPop: false,
        onPopInvokedWithResult: (didPop, _) async {
          if (didPop) return;
          if (await _controller.canGoBack()) {
            await _controller.goBack();
          } else {
            if (context.mounted) Navigator.of(context).pop();
          }
        },
        child: WebViewWidget(controller: _controller),
      ),
    );
  }
}
```

`WebViewController`를 `initState`에서 초기화하고, 빌드 시 `WebViewWidget(controller: _controller)`에 주입합니다. `PopScope`와 `canGoBack()`을 조합해 WebView의 히스토리 스택을 먼저 소진하고, 이후 Flutter 네비게이션 스택을 pop합니다.

### NavigationDelegate 콜백 상세

| 콜백 | 설명 |
|------|------|
| `onProgress` | 0~100 로딩 진행률. `LinearProgressIndicator`와 연동 |
| `onPageStarted` | 페이지 로딩 시작 URL |
| `onPageFinished` | 페이지 로딩 완료 URL |
| `onNavigationRequest` | 네비게이션 허용/차단 결정. `NavigationDecision.navigate` 또는 `prevent` 반환 |
| `onWebResourceError` | CSS, JS, 이미지 등 리소스 로딩 오류 |
| `onHttpError` | HTTP 4xx/5xx 응답 오류 |

## JavaScript Channel: 양방향 통신 구현

웹 페이지와 Flutter 앱 간의 양방향 통신은 많은 앱에서 필수입니다. `addJavaScriptChannel`로 등록한 채널 이름이 `window.{채널명}`으로 웹 페이지에 주입됩니다.

**시나리오**: 결제 완료 시 웹 페이지에서 Flutter로 결과를 전달하고, Flutter는 화면을 갱신하거나 다이얼로그를 표시합니다.

```dart
import 'dart:convert';
import 'package:flutter/material.dart';
import 'package:webview_flutter/webview_flutter.dart';

class PaymentWebViewPage extends StatefulWidget {
  const PaymentWebViewPage({super.key});

  @override
  State<PaymentWebViewPage> createState() => _PaymentWebViewPageState();
}

class _PaymentWebViewPageState extends State<PaymentWebViewPage> {
  late final WebViewController _controller;
  String _statusMessage = '결제 진행 중...';

  @override
  void initState() {
    super.initState();

    _controller = WebViewController()
      ..setJavaScriptMode(JavaScriptMode.unrestricted)
      // Web → Flutter: 채널 등록
      ..addJavaScriptChannel(
        'FlutterBridge',
        onMessageReceived: (JavaScriptMessage message) {
          _handleWebMessage(message.message);
        },
      )
      ..setNavigationDelegate(NavigationDelegate(
        onPageFinished: (_) => _injectFlutterContext(),
      ))
      ..loadRequest(Uri.parse('https://example.com/payment'));
  }

  // Web → Flutter 메시지 처리
  void _handleWebMessage(String rawMessage) {
    try {
      final data = jsonDecode(rawMessage) as Map<String, dynamic>;
      final type = data['type'] as String? ?? '';

      switch (type) {
        case 'payment_success':
          setState(() => _statusMessage = '결제 완료: ${data['orderId']}');
          _showResult('결제가 완료되었습니다!', isSuccess: true);
        case 'payment_fail':
          setState(() => _statusMessage = '결제 실패: ${data['reason']}');
          _showResult('결제에 실패했습니다. 다시 시도해 주세요.', isSuccess: false);
        case 'close':
          if (context.mounted) Navigator.of(context).pop(data['orderId']);
      }
    } catch (e) {
      debugPrint('Message parse error: $e / raw: $rawMessage');
    }
  }

  // Flutter → Web: JS 실행으로 컨텍스트 주입
  Future<void> _injectFlutterContext() async {
    final isDark = Theme.of(context).brightness == Brightness.dark;
    await _controller.runJavaScript('''
      window.__FLUTTER_CONTEXT__ = {
        platform: "flutter",
        isDarkMode: $isDark,
        version: "1.0.0"
      };
      if (typeof onFlutterContextReady === "function") {
        onFlutterContextReady(window.__FLUTTER_CONTEXT__);
      }
    ''');
  }

  // Flutter → Web: JS 반환값 수신
  Future<String?> _getCartTotal() async {
    final result = await _controller.runJavaScriptReturningResult(
      'JSON.stringify(window.cartTotal ?? null)',
    );
    return result.toString();
  }

  void _showResult(String message, {required bool isSuccess}) {
    if (!context.mounted) return;
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(
        content: Text(message),
        backgroundColor: isSuccess ? Colors.green : Colors.red,
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(_statusMessage, style: const TextStyle(fontSize: 14)),
        actions: [
          IconButton(
            icon: const Icon(Icons.shopping_cart_outlined),
            onPressed: () async {
              final total = await _getCartTotal();
              if (context.mounted) {
                ScaffoldMessenger.of(context)
                    .showSnackBar(SnackBar(content: Text('장바구니 합계: $total')));
              }
            },
          ),
        ],
      ),
      body: WebViewWidget(controller: _controller),
    );
  }
}
```

웹 페이지 측에서는 다음과 같이 Flutter로 메시지를 보냅니다:

```javascript
// Web 측 JavaScript 코드
function notifyFlutter(type, payload = {}) {
  if (window.FlutterBridge && window.FlutterBridge.postMessage) {
    window.FlutterBridge.postMessage(JSON.stringify({ type, ...payload }));
  }
}

// 결제 성공 콜백
function onPaymentComplete(orderId) {
  notifyFlutter('payment_success', { orderId });
}

// Flutter 컨텍스트 수신
function onFlutterContextReady(ctx) {
  if (ctx.isDarkMode) {
    document.body.classList.add('dark-mode');
  }
}
```

### 통신 방향 정리

| 방향 | 방법 | API |
|------|------|-----|
| Web → Flutter | `window.{채널명}.postMessage(string)` | `addJavaScriptChannel` |
| Flutter → Web (반환 없음) | JS 코드 직접 실행 | `runJavaScript(String)` |
| Flutter → Web (반환 있음) | JS 표현식 실행 후 결과 수신 | `runJavaScriptReturningResult(String)` |

`runJavaScriptReturningResult`는 `Future<Object>` (실제로는 `String` 또는 `num`)를 반환합니다. JSON 객체를 받으려면 웹에서 `JSON.stringify()`를 감싸야 합니다.

## 쿠키 관리: WebViewCookieManager

로그인 상태 유지나 세션 공유를 위해 쿠키를 프로그래밍적으로 설정해야 하는 경우가 있습니다. `WebViewCookieManager`를 사용하세요.

```dart
import 'package:webview_flutter/webview_flutter.dart';

class AuthWebViewHelper {
  static final _cookieManager = WebViewCookieManager();

  /// 인증 토큰을 쿠키로 설정한 후 페이지 로드
  static Future<void> loadWithAuthToken({
    required WebViewController controller,
    required String url,
    required String accessToken,
    required String refreshToken,
  }) async {
    final domain = Uri.parse(url).host;

    await Future.wait([
      _cookieManager.setCookie(
        WebViewCookie(
          name: 'access_token',
          value: accessToken,
          domain: domain,
          path: '/',
        ),
      ),
      _cookieManager.setCookie(
        WebViewCookie(
          name: 'refresh_token',
          value: refreshToken,
          domain: domain,
          path: '/',
        ),
      ),
    ]);

    await controller.loadRequest(Uri.parse(url));
  }

  /// 쿠키 존재 여부 확인
  static Future<bool> hasCookie(String name, String domain) async {
    final cookies = await _cookieManager.getCookies(Uri.parse('https://$domain'));
    return cookies.any((c) => c.name == name);
  }

  /// 로그아웃 시 모든 쿠키 삭제
  static Future<void> clearAll() async {
    await _cookieManager.clearCookies();
  }
}
```

사용 예:

```dart
await AuthWebViewHelper.loadWithAuthToken(
  controller: _controller,
  url: 'https://example.com/my-page',
  accessToken: userSession.accessToken,
  refreshToken: userSession.refreshToken,
);
```

iOS (`WKWebView`)는 쿠키 동기화 시점이 비동기적이므로, `loadRequest` 직전에 `setCookie`를 반드시 완료해야 합니다. `Future.wait`로 모든 쿠키를 설정한 뒤 `loadRequest`를 호출하는 순서를 지키세요.

## 플랫폼별 고급 설정

### Android: 디버깅 활성화

```dart
import 'package:webview_flutter_android/webview_flutter_android.dart';

void enableAndroidWebViewDebugging() {
  if (WebViewPlatform.instance is AndroidWebViewPlatform) {
    // Chrome DevTools에서 chrome://inspect 로 WebView 디버깅 가능
    AndroidWebViewController.enableDebugging(true);
  }
}
```

개발 빌드에서만 활성화하고 릴리스 빌드에서는 반드시 비활성화하세요.

### Android: Hybrid Composition vs Virtual Display

기본값은 **Hybrid Composition**입니다. 스크롤이나 텍스트 선택 인터랙션이 중요하다면 그대로 사용하세요. 렌더링 성능을 우선한다면 Virtual Display로 전환할 수 있지만, 키보드 입력 이슈가 발생할 수 있습니다.

```dart
final platform = const AndroidWebViewWidgetCreationParams(
  displayWithHybridComposition: false, // Virtual Display 모드
);
```

### iOS: WKWebView allowsInlineMediaPlayback

iOS에서 비디오 인라인 재생을 허용하려면:

```dart
import 'package:webview_flutter_wkwebview/webview_flutter_wkwebview.dart';

final params = WebKitWebViewControllerCreationParams(
  allowsInlineMediaPlayback: true,
  mediaTypesRequiringUserAction: const <PlaybackMediaTypes>{},
);
final controller = WebViewController.fromPlatformCreationParams(params);
```

## 주의사항 및 실전 팁

### 1. JavaScript 보안

`JavaScriptMode.unrestricted`는 신뢰할 수 있는 자사 웹 페이지에만 사용하세요. 외부 URL(사용자 생성 콘텐츠, 광고 등)을 로드할 때는 `JavaScriptMode.disabled`를 고려하거나 URL 화이트리스트를 엄격히 적용하세요.

```dart
onNavigationRequest: (request) {
  final allowedHosts = {'example.com', 'api.example.com'};
  final host = Uri.tryParse(request.url)?.host ?? '';
  return allowedHosts.contains(host)
      ? NavigationDecision.navigate
      : NavigationDecision.prevent;
},
```

### 2. 딥링크와 커스텀 스킴 처리

카카오 로그인, 네이버 로그인 등에서 `intent://` 또는 소셜 로그인 콜백 스킴이 발생합니다. `onNavigationRequest`에서 이를 차단하고 `url_launcher`로 별도 처리하세요. 처리하지 않으면 Android에서 앱 크래시가 발생합니다.

### 3. 메모리 관리

WebView는 많은 메모리를 소비합니다. 사용이 끝난 WebView 페이지는 신속하게 dispose하고, 필요하다면 `_controller.clearCache()`와 `_controller.clearLocalStorage()`를 호출해 캐시를 정리하세요.

### 4. File Upload 처리 (Android)

Android에서 `<input type="file">`을 지원하려면 `onShowFileSelector` 콜백을 구현해야 합니다:

```dart
import 'package:webview_flutter_android/webview_flutter_android.dart';

final androidController = _controller.platform as AndroidWebViewController;
await androidController.setOnShowFileSelector((params) async {
  // file_picker 패키지 등을 사용해 파일 선택
  return []; // 선택된 파일 URI 목록 반환
});
```

### 5. 에러 페이지 UX

`onWebResourceError`와 `onHttpError`를 활용해 사용자에게 친화적인 에러 화면을 보여주세요. 기본 브라우저 에러 페이지는 앱 UX와 어울리지 않습니다.

```dart
onWebResourceError: (error) {
  if (error.isForMainFrame == true) {
    _controller.loadHtmlString('''
      <html><body style="font-family:sans-serif; padding:40px; text-align:center;">
        <h2>페이지를 불러올 수 없습니다</h2>
        <p>네트워크 연결을 확인해 주세요.</p>
      </body></html>
    ''');
  }
},
```

## 결론

`webview_flutter 4.x`는 컨트롤러와 위젯을 분리하고, 플랫폼 구현체를 플러그인으로 분리함으로써 더 유연하고 테스트하기 쉬운 아키텍처를 제공합니다.

핵심 요약:
- **`WebViewController`**: JavaScript 모드, 쿠키, 네비게이션 델리게이트, JS 채널 등 모든 설정의 진입점
- **`NavigationDelegate`**: 네비게이션 허용/차단, 로딩 진행률, 에러 처리
- **`addJavaScriptChannel`**: Web → Flutter 단방향 메시지 채널 (JS에서 `postMessage` 호출)
- **`runJavaScript` / `runJavaScriptReturningResult`**: Flutter → Web 방향 제어
- **`WebViewCookieManager`**: 쿠키 설정/삭제/조회

보안, 성능, UX 세 축을 균형 있게 고려하여 WebView를 단순한 URL 렌더러가 아닌, Flutter 앱과 웹이 협력하는 통합 시스템으로 설계하세요.

## 참고 자료
- [webview_flutter 패키지 (pub.dev)](https://pub.dev/packages/webview_flutter)
- [webview_flutter API 문서](https://pub.dev/documentation/webview_flutter/latest/)
- [webview_flutter_android 패키지 (pub.dev)](https://pub.dev/packages/webview_flutter_android)
