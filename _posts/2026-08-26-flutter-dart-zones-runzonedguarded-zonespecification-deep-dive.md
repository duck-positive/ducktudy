---
layout: post
title: "Flutter Dart Zones 심화: runZonedGuarded와 ZoneSpecification으로 비동기 에러를 완벽히 제어하는 법"
date: 2026-08-26
categories: [android, flutter]
tags: [flutter, dart, zones, runZonedGuarded, ZoneSpecification, error-handling, async, crashlytics]
---

Flutter 앱이 프로덕션에서 조용히 죽는 이유 중 상당수는 "잡히지 않은 비동기 에러" 때문입니다. `try-catch`는 동기 코드와 `await` 직전까지만 에러를 잡을 수 있고, Future 체인 중간이나 Timer 콜백 안에서 발생한 에러는 그냥 사라집니다. Dart의 **Zone** 은 이 문제를 근본적으로 해결하는 실행 컨텍스트(execution context) 메커니즘입니다. Firebase Crashlytics, 글로벌 로깅, 테스트 환경 격리 등 고급 Flutter 개발의 핵심을 Zone이 받치고 있습니다. 이 글에서는 Zone의 내부 원리부터 실전 패턴까지 완전히 파헤칩니다.

---

## Zone이란 무엇인가?

Zone은 Dart 비동기 실행의 "스레드 로컬 스토리지" 이자 "실행 컨텍스트"입니다. 모든 Dart 코드는 반드시 어떤 Zone 안에서 실행됩니다. 프로그램이 시작되면 `Zone.root`가 생성되고, `main()` 함수는 이 루트 Zone에서 실행됩니다.

Zone의 핵심 특성은 세 가지입니다:

1. **상속**: 자식 Zone은 부모 Zone의 설정을 상속합니다.
2. **비동기 컨텍스트 전파**: Future, Stream, Timer, microtask 등 모든 비동기 연산은 자신이 **등록된** Zone을 기억하고, 콜백이 실행될 때 그 Zone 안에서 실행됩니다.
3. **오버라이드 가능**: `ZoneSpecification`을 통해 에러 처리, 프린트, 타이머 등 동작을 재정의할 수 있습니다.

```
Zone.root
  └── main() Zone (기본적으로 Zone.root와 동일)
        └── runZonedGuarded로 생성한 자식 Zone
              └── runApp() → Flutter 위젯 트리 전체
```

이 계층에서 중요한 사실이 있습니다. **비동기 에러는 동일한 errorZone 내에서만 전파됩니다.** `runZonedGuarded`가 만든 Zone은 독립적인 errorZone을 갖기 때문에, 그 안의 모든 미처리 비동기 에러를 `onError` 콜백 하나로 잡을 수 있습니다.

---

## 왜 Zone이 필요한가?

### try-catch의 한계

```dart
// 이 코드는 에러를 잡지 못합니다!
void main() {
  try {
    Future.delayed(Duration(seconds: 1), () {
      throw Exception('1초 후 발생한 에러');
    });
  } catch (e) {
    print('잡혔다: $e'); // 절대 실행되지 않음
  }
}
```

`try-catch`는 동기적으로 실행되는 코드 블록만 보호합니다. `Future.delayed`의 콜백은 나중에, 다른 이벤트 루프 사이클에서 실행되므로 `catch`가 이를 감지하지 못합니다.

### FlutterError.onError의 한계

Flutter는 `FlutterError.onError`를 통해 위젯 트리 내부 에러를 잡을 수 있지만, **위젯 빌드 외부**에서 발생하는 에러 — 예를 들어 버튼의 `onPressed` 안에서 await 없이 발생한 예외, HTTP 콜백 에러 등은 여기서도 잡히지 않습니다.

Zone은 이 두 가지 한계를 동시에 해결합니다.

---

## 실제 구현 예제 1: runZonedGuarded로 전역 에러 수집기 구현

```dart
import 'dart:async';
import 'package:flutter/material.dart';
import 'package:flutter/foundation.dart';

Future<void> main() async {
  // Zone 밖에서 호출하면 안 됩니다.
  // ensureInitialized는 반드시 runZonedGuarded 안에서 호출해야
  // Zone 바인딩이 올바르게 연결됩니다.
  runZonedGuarded<Future<void>>(
    () async {
      WidgetsFlutterBinding.ensureInitialized();

      // Flutter 자체 에러 핸들러: 위젯 빌드 에러를 잡습니다.
      FlutterError.onError = (FlutterErrorDetails details) {
        // release 모드에서는 크래시리포터로 전송
        if (kReleaseMode) {
          _reportError(details.exception, details.stack ?? StackTrace.empty);
        } else {
          // debug 모드에서는 기본 동작(콘솔 출력) 유지
          FlutterError.presentError(details);
        }
      };

      // PlatformDispatcher: Flutter 2.8 이후 비동기 에러를 더 세밀하게 잡는 방법
      PlatformDispatcher.instance.onError = (error, stack) {
        _reportError(error, stack);
        return true; // true를 반환하면 에러가 처리된 것으로 표시
      };

      runApp(const MyApp());
    },
    // Zone의 onError: Future/Stream/Timer 등 모든 미처리 비동기 에러
    (Object error, StackTrace stack) {
      _reportError(error, stack);
    },
  );
}

void _reportError(Object error, StackTrace stack) {
  // 실제 프로덕션에서는 FirebaseCrashlytics.instance.recordError(error, stack)
  debugPrint('=== 글로벌 에러 수집 ===');
  debugPrint('에러: $error');
  debugPrint('스택: $stack');
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Zone Demo',
      home: Scaffold(
        appBar: AppBar(title: const Text('Zone 에러 처리 데모')),
        body: Center(
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              // 이 에러는 FlutterError.onError가 잡습니다
              ElevatedButton(
                onPressed: () => throw Exception('동기 에러 (FlutterError)'),
                child: const Text('동기 에러 발생'),
              ),
              const SizedBox(height: 16),
              // 이 에러는 runZonedGuarded의 onError가 잡습니다
              ElevatedButton(
                onPressed: () {
                  Future.delayed(const Duration(milliseconds: 100), () {
                    throw Exception('비동기 에러 (Zone)');
                  });
                },
                child: const Text('비동기 에러 발생'),
              ),
            ],
          ),
        ),
      ),
    );
  }
}
```

**핵심 포인트**: `WidgetsFlutterBinding.ensureInitialized()`를 `runZonedGuarded` **안에서** 호출해야 합니다. 바깥에서 호출하면 Flutter 바인딩이 `Zone.root`에 등록되어 Zone 기반 에러 처리가 제대로 동작하지 않습니다.

---

## 실제 구현 예제 2: ZoneSpecification으로 로깅 인터셉터 구현

`ZoneSpecification`은 Zone의 동작 자체를 재정의하는 더 고급 API입니다. `print` 출력을 가로채거나, 타이머를 모킹하거나, `Zone.current[key]` 로 컨텍스트 값을 전달하는 데 사용합니다.

```dart
import 'dart:async';

/// Zone을 이용한 구조적 로깅 인터셉터
/// print() 호출을 가로채 타임스탬프와 컨텍스트를 자동으로 추가합니다.
class ZoneLogger {
  static R run<R>(
    String context,
    R Function() body, {
    void Function(String log)? sink,
  }) {
    final logs = <String>[];
    final startTime = DateTime.now();

    final spec = ZoneSpecification(
      // print() 호출 인터셉트
      print: (Zone self, ZoneDelegate parent, Zone zone, String line) {
        final elapsed = DateTime.now().difference(startTime).inMilliseconds;
        final formatted = '[$context +${elapsed}ms] $line';
        logs.add(formatted);
        // 부모 Zone의 print를 호출해야 실제로 출력됩니다
        parent.print(zone, formatted);
      },

      // 처리되지 않은 에러 핸들링
      handleUncaughtError: (
        Zone self,
        ZoneDelegate parent,
        Zone zone,
        Object error,
        StackTrace stackTrace,
      ) {
        final elapsed = DateTime.now().difference(startTime).inMilliseconds;
        final msg = '[$context +${elapsed}ms] ❌ UNCAUGHT: $error';
        logs.add(msg);
        sink?.call(logs.join('\n'));
        // 부모로 에러를 전파
        parent.handleUncaughtError(zone, error, stackTrace);
      },
    );

    return Zone.current.fork(specification: spec).run<R>(body);
  }
}

// Zone 로컬 값 전달: ZoneValues
final _requestIdKey = Object();

String? get currentRequestId =>
    Zone.current[_requestIdKey] as String?;

R runWithRequestId<R>(String requestId, R Function() body) {
  return runZoned(body, zoneValues: {_requestIdKey: requestId});
}

// 사용 예시
void main() {
  // ZoneLogger 사용 예시
  ZoneLogger.run('HTTP-REQUEST', () {
    print('요청 시작');
    Future.delayed(const Duration(milliseconds: 50), () {
      print('응답 수신');
    });
    Future.delayed(const Duration(milliseconds: 100), () {
      print('파싱 완료');
    });
  });

  // ZoneValues를 통한 요청 ID 전파
  // HTTP 요청마다 고유 ID를 Zone에 주입하면,
  // 해당 Zone 안의 모든 Future 콜백에서 requestId를 조회할 수 있습니다.
  runWithRequestId('req-abc-123', () async {
    print('현재 요청 ID: ${currentRequestId}'); // req-abc-123

    await Future.delayed(const Duration(milliseconds: 10));

    // await 이후에도 Zone이 유지되므로 requestId 접근 가능
    print('비동기 후 요청 ID: ${currentRequestId}'); // req-abc-123
  });
}
```

출력 예시:
```
[HTTP-REQUEST +0ms] 요청 시작
현재 요청 ID: req-abc-123
[HTTP-REQUEST +50ms] 응답 수신
비동기 후 요청 ID: req-abc-123
[HTTP-REQUEST +100ms] 파싱 완료
```

`ZoneValues`의 강력함은 **비동기 경계를 넘어도 값이 유지된다**는 점입니다. `await` 이후에도, Timer 콜백 안에서도, Stream listen 핸들러 안에서도 같은 Zone 안이면 `Zone.current[key]`로 값을 꺼낼 수 있습니다. Go의 `context.Context`나 Java의 `ThreadLocal`과 유사하지만 비동기 환경에 자연스럽게 통합됩니다.

---

## Zone과 Flutter 테스트 환경

Zone은 테스트에서도 핵심적으로 활용됩니다. `flutter_test` 패키지의 `testWidgets`는 내부적으로 자체 Zone에서 테스트를 실행합니다. `FakeAsync`도 Zone의 타이머 인터셉트(`createTimer`, `createPeriodicTimer`)를 활용해 시간을 제어합니다.

```dart
import 'dart:async';
import 'package:fake_async/fake_async.dart';
import 'package:test/test.dart';

void main() {
  test('FakeAsync는 Zone 타이머 인터셉트로 시간을 제어합니다', () {
    fakeAsync((async) {
      var result = '';

      // 실제로 1시간을 기다리지 않습니다
      Future.delayed(const Duration(hours: 1), () {
        result = '1시간 후 실행됨';
      });

      // Zone 안의 타이머를 가상으로 1시간 전진
      async.elapse(const Duration(hours: 1));

      expect(result, '1시간 후 실행됨');
    });
  });
}
```

---

## Zone 동작의 세부 원리: 콜백 등록과 실행

Zone이 비동기 컨텍스트를 유지하는 방법을 이해하려면 "등록(register)"과 "실행(run)"의 분리를 알아야 합니다.

Dart의 이벤트 루프가 Future 콜백을 실행할 때의 흐름:

```
1. myZone.run(() { Future.then(callback) })
   → myZone.registerUnaryCallback(callback) 호출
   → 등록된 wrappedCallback을 큐에 저장

2. 이벤트 루프가 큐에서 wrappedCallback을 꺼냄
   → myZone.runUnary(wrappedCallback, value) 호출
   → callback이 myZone 컨텍스트에서 실행됨
```

`ZoneSpecification`의 `registerUnaryCallback`과 `runUnary`를 재정의하면 모든 콜백의 등록과 실행 시점을 가로챌 수 있습니다. 이것이 `FakeAsync`가 실제 타이머를 실행하지 않고 시간을 제어하는 방법입니다.

---

## 주의사항 및 실전 팁

### 1. Zone 경계를 넘는 에러 전파 불가

```dart
final zoneA = Zone.current.fork(
  specification: ZoneSpecification(
    handleUncaughtError: (s, p, z, e, st) => print('Zone A 에러: $e'),
  ),
);
final zoneB = Zone.current.fork(); // 별도 errorZone

// 이 에러는 Zone A의 핸들러로 가지 않습니다
// zoneA와 zoneB는 서로 다른 errorZone을 가집니다
zoneA.run(() {
  zoneB.run(() {
    throw Exception('Zone B 에러');
  });
});
```

`runZonedGuarded`가 생성하는 Zone은 독립적인 errorZone을 갖습니다. 에러는 **자신의 errorZone의 핸들러**로만 전달됩니다. 중첩된 `runZonedGuarded`가 있다면 내부 핸들러가 먼저 받습니다.

### 2. runZonedGuarded는 동기 에러도 잡습니다

```dart
runZonedGuarded(() {
  throw Exception('동기 에러도 잡힙니다');
}, (e, st) {
  print('잡힘: $e'); // 실행됩니다
});
```

`onError`는 `body` 함수가 동기적으로 던지는 에러도 처리합니다.

### 3. Zone은 Isolate와 다릅니다

Zone은 메인 Isolate 안에서만 동작합니다. `Isolate.spawn`으로 생성한 새 Isolate는 독립적인 이벤트 루프와 Zone.root를 가지므로, 부모 Zone의 설정이 전혀 전파되지 않습니다. 따라서 Isolate 안의 에러는 별도로 `Isolate.current.addErrorListener`나 `compute`의 에러 핸들링으로 처리해야 합니다.

### 4. ZoneSpecification의 delegate 패턴

`ZoneSpecification`의 각 콜백에는 `parent: ZoneDelegate` 파라미터가 있습니다. 반드시 `parent.print(zone, ...)` 처럼 부모 델리게이트를 호출해야 기본 동작이 유지됩니다. 잊으면 출력이 사라지거나 타이머가 동작하지 않는 버그가 생깁니다.

### 5. Flutter 3.10+ PlatformDispatcher.onError 우선 순위

Flutter 3.10부터 `PlatformDispatcher.instance.onError`가 `runZonedGuarded`의 Zone 에러 핸들러보다 **더 넓은 범위**의 에러를 잡을 수 있습니다. 두 가지를 함께 설정하는 것이 프로덕션 앱의 완전한 에러 수집을 보장합니다:

```dart
// 완전한 에러 수집 조합
PlatformDispatcher.instance.onError = (error, stack) {
  _reportError(error, stack);
  return true;
};
FlutterError.onError = (details) {
  _reportError(details.exception, details.stack ?? StackTrace.empty);
};
// + runZonedGuarded onError
```

---

## 정리

| 메커니즘 | 잡는 에러 | 사용 시점 |
|---|---|---|
| `try-catch` | 동기 에러, `await` 직전 | 지역적 에러 처리 |
| `FlutterError.onError` | 위젯 빌드/렌더링 에러 | 위젯 트리 에러 리포팅 |
| `PlatformDispatcher.onError` | Flutter 엔진 레벨 에러 | Flutter 3.10+ 권장 |
| `runZonedGuarded` onError | 모든 미처리 비동기 에러 | 글로벌 에러 수집 |
| `ZoneSpecification` | 콜백 등록/실행/출력 인터셉트 | 로깅, 테스트, 시간 제어 |

Dart Zone은 단순한 에러 처리 도구를 넘어, 비동기 실행 컨텍스트 전체를 제어하는 강력한 원시 메커니즘입니다. `runZonedGuarded`로 프로덕션 앱의 에러를 빠짐없이 수집하고, `ZoneSpecification`으로 로깅·테스트·타이머 인터셉트를 구현해 보세요. Flutter 앱의 신뢰성이 눈에 띄게 향상될 것입니다.

## 참고 자료
- [Dart Zones 공식 가이드 — dart.dev](https://dart.dev/libraries/async/zones)
- [Flutter 에러 처리 공식 문서 — flutter.dev](https://docs.flutter.dev/testing/errors)
- [runZonedGuarded API — api.flutter.dev](https://api.flutter.dev/flutter/dart-async/runZonedGuarded.html)
- [Firebase Crashlytics Flutter 연동 가이드 — firebase.flutter.dev](https://firebase.flutter.dev/docs/crashlytics/usage/)
