---
layout: post
title: "Flutter Web WASM 심화: Skwasm 렌더러·CanvasKit·dart:js_interop으로 프로덕션 웹앱 완전 정복"
date: 2026-08-11
categories: [android, flutter]
tags: [flutter, flutter-web, wasm, webassembly, skwasm, canvaskit, dart-js-interop, performance]
---

Flutter가 웹 플랫폼을 지원하기 시작한 이후, 렌더러 전략은 꾸준히 진화해왔습니다. HTML 렌더러에서 CanvasKit(Skia + JS)으로, 그리고 이제는 **Skwasm(Skia + WebAssembly)** 으로의 전환이 완성 단계에 이르렀습니다. Flutter 3.24부터 `--wasm` 플래그는 실험적 기능을 넘어 프로덕션 배포에 충분히 활용할 수 있는 수준이 되었습니다. 이 글에서는 Flutter Web WASM 빌드의 내부 동작 원리, Skwasm 렌더러의 장점, dart:js_interop 마이그레이션, 그리고 프로덕션 서버 설정까지 완전히 분석합니다.

## Flutter Web 렌더러의 역사와 현재

Flutter Web은 세 가지 렌더러를 거쳐 발전해왔습니다.

### 1. HTML 렌더러 (레거시)

초기 Flutter Web은 DOM 요소(div, canvas, img)를 직접 조작하는 HTML 렌더러를 사용했습니다. 번들 크기는 작았지만 Skia 기반의 픽셀 단위 렌더링 품질을 포기해야 했습니다. Flutter 3.22에서 공식적으로 **deprecated** 되었습니다.

### 2. CanvasKit 렌더러 (JavaScript 기반)

Skia를 Emscripten으로 WebAssembly에 컴파일하고, 이를 JavaScript에서 호출하는 방식입니다. 렌더링 품질은 네이티브와 동일하지만, Dart 코드 자체는 여전히 JavaScript로 트랜스파일됩니다. 초기 번들에 Skia WASM 바이너리(약 1.5~2MB)가 포함되어 초기 로딩이 느립니다.

### 3. Skwasm 렌더러 (WASM 기반, 현재 기본값)

Flutter 3.22부터 기본 렌더러로 채택되었습니다. **Dart 코드 자체도 WebAssembly(WasmGC)로 컴파일**되어 실행됩니다. 렌더링 워크로드를 별도의 Web Worker 스레드로 오프로드하여 메인 스레드 부담을 줄이는 것이 핵심입니다.

```
┌──────────────────────────────────────────────────────────┐
│  브라우저 메인 스레드                                        │
│  ┌────────────────┐   postMessage   ┌─────────────────┐  │
│  │  Dart/WASM App │ ──────────────▶ │  Skwasm Renderer │  │
│  │  (WasmGC)      │                 │  Web Worker      │  │
│  └────────────────┘                 └─────────────────┘  │
│                                              │             │
│                                              ▼             │
│                                       GPU Canvas 2D       │
└──────────────────────────────────────────────────────────┘
```

## 왜 Skwasm + WASM인가?

### 성능 수치로 보는 차이

실제 측정값 기준으로, 애니메이션이 많은 화면에서 Skwasm WASM 빌드는 CanvasKit(JS) 대비 약 **25~40% 빠른 첫 번째 페인트 시간**과 **15~30% 향상된 프레임 레이트**를 보입니다. 이는 두 가지 요인에서 기인합니다.

1. **WasmGC의 타입 안전성**: Dart의 타입 시스템이 WASM 레벨에서 보존되어 JIT 없이도 컴파일러 최적화가 극대화됩니다.
2. **멀티스레드 렌더링**: Skwasm은 Web Worker에서 독립적으로 실행되므로, 상태 관리나 이벤트 처리가 렌더링을 차단하지 않습니다.

### 브라우저 지원 현황 (2026년 기준)

WasmGC(Garbage Collected WebAssembly)는 2026년 초 기준 전 세계 브라우저 트래픽의 약 92%를 커버합니다.

| 브라우저 | WasmGC 지원 버전 |
|---|---|
| Chrome / Edge | 119+ |
| Firefox | 120+ |
| Safari | 18.2+ |
| iOS Safari | **미지원** (WebKit 제한) |

나머지 약 8%(구형 브라우저, iOS)에 대해서는 Flutter가 빌드 시 JavaScript 폴백을 자동 생성합니다. `flutter build web --wasm`은 WASM 빌드와 JS 폴백 빌드를 함께 생성하고, 브라우저 감지 로직을 포함한 `flutter.js`가 적절한 빌드를 선택합니다.

## 실제 구현: WASM 빌드 및 서버 설정

### 빌드 명령어

```bash
# 개발 중 Chrome에서 직접 테스트
flutter run -d chrome --wasm

# 프로덕션 빌드 (WASM + JS 폴백)
flutter build web --wasm

# 소스맵 포함 (디버깅용)
flutter build web --wasm --source-maps
```

빌드 결과물은 `build/web/` 디렉토리에 생성됩니다. 여기에 `.wasm` 파일과 기존 `.js` 파일이 함께 포함됩니다.

### 코드 예제 1: Ktor 서버에서 COEP/COOP 헤더 설정 (Kotlin)

Skwasm의 멀티스레드 렌더링(Web Worker + SharedArrayBuffer)을 활성화하려면 반드시 두 개의 HTTP 헤더를 서버에서 설정해야 합니다. 이 헤더가 없으면 Web Worker 간 SharedArrayBuffer 공유가 차단됩니다.

```kotlin
import io.ktor.server.application.*
import io.ktor.server.engine.*
import io.ktor.server.netty.*
import io.ktor.server.plugins.defaultheaders.*
import io.ktor.server.routing.*
import io.ktor.server.http.content.*
import io.ktor.http.*
import java.io.File

fun main() {
    embeddedServer(Netty, port = 8080) {
        // Flutter Web WASM 멀티스레드 렌더링을 위한 보안 헤더
        install(DefaultHeaders) {
            // SharedArrayBuffer 사용을 위한 Cross-Origin Isolation 헤더
            header("Cross-Origin-Embedder-Policy", "credentialless")
            header("Cross-Origin-Opener-Policy", "same-origin")
        }

        routing {
            // Flutter 빌드 산출물 정적 서빙
            staticFiles("/", File("build/web")) {
                default("index.html")

                // .wasm 파일에 대해 올바른 MIME 타입 설정
                contentType { file ->
                    when {
                        file.name.endsWith(".wasm") ->
                            ContentType("application", "wasm")
                        file.name.endsWith(".js") ->
                            ContentType.Application.JavaScript
                        else -> null
                    }
                }
            }

            // SPA 라우팅을 위한 폴백 (GoRouter 딥링크 지원)
            get("{...}") {
                call.respondFile(File("build/web/index.html"))
            }
        }
    }.start(wait = true)
}
```

**주의**: `Cross-Origin-Embedder-Policy: credentialless`는 `require-corp`보다 완화된 정책으로, 서드파티 리소스(Google Fonts, CDN 이미지)를 CORS 헤더 없이도 로드할 수 있습니다. 단, `credentials: 'include'`를 사용하는 외부 API 호출은 여전히 영향을 받을 수 있습니다.

### 코드 예제 2: dart:js_interop 마이그레이션 (Dart)

WASM 컴파일의 가장 큰 준비 작업은 `dart:html`과 `package:js`를 제거하는 것입니다. 이 레거시 라이브러리들은 WASM 컴파일 대상에서 사용할 수 없습니다.

```dart
import 'dart:js_interop';
import 'package:web/web.dart' as web;

// ─── JavaScript 외부 함수 선언 ───────────────────────────────────────────────

// @JS 어노테이션으로 window.gtag 같은 전역 JS 함수 바인딩
@JS('gtag')
external void _gtagNative(JSString command, JSString eventName, JSObject params);

// JS 객체를 직접 리터럴로 생성하기 위한 확장
extension type GtagEventParams._(JSObject _) implements JSObject {
  external factory GtagEventParams({
    JSString event_category,
    JSString event_label,
    JSNumber value,
  });
}

// ─── Dart에서 JS 함수 호출 ────────────────────────────────────────────────────

class AnalyticsService {
  void trackEvent({
    required String category,
    required String label,
    int value = 0,
  }) {
    // Dart 타입을 JS 타입으로 변환: .toJS 확장 메서드 사용
    _gtagNative(
      'event'.toJS,
      'button_click'.toJS,
      GtagEventParams(
        event_category: category.toJS,
        event_label: label.toJS,
        value: value.toJS,
      ),
    );
  }

  // package:web을 통한 DOM 접근 (dart:html 대체)
  void updateTitle(String newTitle) {
    // web.document는 package:web이 제공하는 타입 안전 DOM API
    web.document.title = newTitle;
  }

  // JS Promise를 Dart Future로 변환
  Future<String> fetchUserToken() async {
    // window.crypto.subtle 같은 Web API 접근
    final subtle = web.window.crypto.subtle;
    // toDart로 JSPromise → Future 변환
    final key = await subtle
        .generateKey(
          CryptoKeyAlgorithm(name: 'HMAC'.toJS),
          true.toJS,
          ['sign'.toJS, 'verify'.toJS].toJS,
        )
        .toDart;
    return key.toString();
  }
}

// ─── Dart 객체를 JS로 내보내기 ────────────────────────────────────────────────

// @JSExport로 표시된 클래스는 JS 코드에서 접근 가능
@JSExport()
class FlutterBridge {
  // JS에서 flutter.callDart('navigate', '/home') 형태로 호출 가능
  void navigate(String route) {
    // Navigator나 GoRouter에 라우팅 요청
    print('Navigate to: $route');
  }
}
```

**핵심 변환 규칙**:

| 레거시 | 신규 |
|---|---|
| `dart:html` | `package:web` |
| `package:js`, `dart:js` | `dart:js_interop` |
| `js.context['key']` | `@JS('key') external ...` |
| `promiseObj.then(...)` | `promiseObj.toDart` |
| `dartValue.jsify()` | `dartValue.toJS` |

## 주의사항 및 실전 팁

### 1. WASM 호환성 체크를 자동화하라

`dart pub global run dart_tools:compat_check`로 프로젝트 전체의 WASM 비호환 패키지를 사전에 감지할 수 있습니다. CI 파이프라인에 추가하는 것을 권장합니다.

### 2. iOS는 여전히 폴백에 의존한다

2026년 현재 모든 iOS 브라우저는 WebKit 엔진을 강제 사용하며, WebKit은 아직 WasmGC를 완전 지원하지 않습니다. iOS 사용자 비율이 높은 서비스는 JS 폴백 성능도 함께 최적화해야 합니다.

### 3. 번들 크기 전략

`flutter build web --wasm`은 WASM 바이너리를 별도로 지연 로드합니다. 초기 HTML + JS 셸은 가볍게 유지되고, `.wasm` 파일은 `<link rel="preload">` 힌트로 병렬 다운로드됩니다. `web/index.html`에 직접 preload 힌트를 추가하면 FCP(First Contentful Paint)를 더욱 단축할 수 있습니다.

```html
<!-- web/index.html에 추가 -->
<link rel="preload" href="flutter_service_worker.js" as="script">
<link rel="preload" href="main.dart.wasm" as="fetch" type="application/wasm" crossorigin>
```

### 4. COEP 헤더로 인한 서드파티 리소스 문제 해결

`credentialless` 정책을 사용해도 일부 서드파티 SDK(광고, 분석)가 동작하지 않을 수 있습니다. 이 경우 Flutter Web을 iframe으로 임베드하고, 상위 프레임에서 헤더를 설정하는 구조가 대안이 될 수 있습니다.

### 5. 빌드 시간 최적화

WASM 빌드는 JS 빌드보다 컴파일 시간이 더 깁니다. CI에서 `--no-tree-shake-icons`와 함께 캐싱 전략을 적용하고, 개발 중에는 `flutter run -d chrome`(JS 모드)을 사용하다가 스테이징 배포 시에만 WASM 빌드를 실행하는 것이 효율적입니다.

## 마치며

Flutter Web의 WASM 전환은 단순한 성능 업그레이드가 아닙니다. Dart가 JavaScript 런타임 의존성에서 완전히 벗어나 WebAssembly 생태계의 일원이 되는 패러다임 전환입니다. `dart:js_interop`과 `package:web`으로의 마이그레이션은 초기 비용이 있지만, WasmGC 기반의 타입 안전한 상호운용성과 멀티스레드 렌더링의 이점은 그 비용을 충분히 정당화합니다. iOS WasmGC 지원이 확대되는 시점이 Flutter Web의 완전한 WASM 시대의 개막이 될 것입니다.

## 참고 자료
- [Flutter 공식 WASM 지원 문서 (GitHub)](https://github.com/flutter/website/blob/main/sites/docs/src/content/platform-integration/web/wasm.md)
- [Dart SDK dart:js_interop 소스 (GitHub)](https://github.com/dart-lang/sdk/blob/main/sdk/lib/js_interop/js_interop.dart)
