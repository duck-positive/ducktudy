---
layout: post
title: "Flutter dart:async 심화: Stream, StreamController, StreamTransformer, Broadcast Stream 내부 구조 완전 정복"
date: 2026-11-13
categories: [flutter, dart]
tags: [flutter, dart, stream, async, streamcontroller, streamtransformer, broadcast, reactive]
---

Flutter 개발자라면 `Future`와 `async/await`은 익숙하지만, `Stream`의 내부 구조와 `StreamController`, `StreamTransformer`를 직접 다뤄본 경험은 많지 않은 경우가 많습니다. 이 글에서는 Dart의 `dart:async` 라이브러리가 제공하는 Stream API의 핵심 개념을 내부 구조까지 파고들어 완전히 이해하고, 실전에서 활용할 수 있는 패턴을 소개합니다.

## Stream이란? — 비동기 데이터의 흐름

`Stream`은 Dart에서 **시간이 지남에 따라 순차적으로 도착하는 비동기 이벤트의 시퀀스**입니다. `Future`가 "한 번만 발생하는 비동기 값"을 표현한다면, `Stream`은 "0번 이상 반복적으로 발생하는 비동기 이벤트의 흐름"을 표현합니다.

Dart의 Stream은 세 가지 종류의 이벤트를 전달합니다:

- **데이터 이벤트 (data event)**: 실제 값
- **에러 이벤트 (error event)**: 오류 정보와 스택 트레이스
- **완료 이벤트 (done event)**: 스트림이 종료되었음을 알리는 신호

Flutter에서는 WebSocket 메시지 수신, 사용자 입력 감지, 파일 읽기, BLE 디바이스 데이터 수신 등 다양한 곳에서 Stream이 활용됩니다.

### Single-subscription Stream vs Broadcast Stream

Dart Stream에는 두 가지 종류가 있습니다.

**Single-subscription Stream**: 오직 하나의 리스너(listener)만 허용합니다. 리스너가 등록되기 전까지 이벤트 전달이 시작되지 않으며, 리스너가 취소되면 더 이상 이벤트를 전달하지 않습니다. `File.openRead()`, HTTP 응답 body 등이 이에 해당합니다.

**Broadcast Stream**: 여러 리스너를 동시에 허용합니다. 리스너가 없어도 이벤트가 발생할 수 있으며(이 경우 이벤트는 버려집니다), 리스너는 중간에 구독을 시작하거나 취소할 수 있습니다. `StreamController.broadcast()`로 생성하며, Flutter의 `ValueNotifier`, BLoC 등 상태 관리 라이브러리의 내부에서 광범위하게 사용됩니다.

## 왜 StreamController가 필요한가?

Dart에서 Stream을 직접 생성하는 가장 일반적인 방법은 두 가지입니다:

1. `async*` 함수 (async generator)
2. `StreamController` 클래스

`async*`는 단순한 시퀀스를 표현할 때 적합하지만, 외부에서 임의의 시점에 이벤트를 주입하거나, 여러 리스너를 지원해야 하거나, 스트림의 생명주기(pause/resume/cancel)를 세밀하게 제어해야 할 때는 `StreamController`가 필요합니다.

`StreamController`는 Stream의 생산자(producer) 역할을 하는 싱크(sink)와 소비자(consumer)가 구독하는 Stream을 하나의 쌍으로 묶어 줍니다.

```
[ Producer ] → [ StreamController.sink ] → [ StreamController.stream ] → [ Consumer ]
```

## 실제 구현 예제 1: StreamController로 실시간 검색 디바운싱 구현

다음은 `StreamController`와 `Stream.transform`을 활용하여 사용자의 검색 입력에 디바운스(debounce)를 적용하는 완전한 Dart 예제입니다.

```dart
import 'dart:async';

/// 디바운스 StreamTransformer — 마지막 이벤트 후 [duration]이 지나야 이벤트를 전달
class DebounceTransformer<T> extends StreamTransformerBase<T, T> {
  final Duration duration;

  const DebounceTransformer(this.duration);

  @override
  Stream<T> bind(Stream<T> stream) {
    late StreamController<T> controller;
    Timer? timer;
    StreamSubscription<T>? subscription;

    void onData(T event) {
      timer?.cancel();
      timer = Timer(duration, () => controller.add(event));
    }

    void onError(Object error, StackTrace stack) {
      timer?.cancel();
      controller.addError(error, stack);
    }

    void onDone() {
      timer?.cancel();
      controller.close();
    }

    controller = StreamController<T>(
      onListen: () {
        subscription = stream.listen(
          onData,
          onError: onError,
          onDone: onDone,
          cancelOnError: false,
        );
      },
      onPause: () => subscription?.pause(),
      onResume: () => subscription?.resume(),
      onCancel: () {
        timer?.cancel();
        subscription?.cancel();
        subscription = null;
      },
    );

    return controller.stream;
  }
}

/// 검색 서비스 — StreamController를 통해 쿼리를 받고 디바운스 후 결과를 노출
class SearchService {
  final _queryController = StreamController<String>();

  /// UI에서 쿼리를 추가하는 싱크
  StreamSink<String> get querySink => _queryController.sink;

  /// 디바운스된 검색 결과 스트림
  late final Stream<List<String>> results = _queryController.stream
      .transform(const DebounceTransformer(Duration(milliseconds: 350)))
      .distinct() // 동일한 쿼리 연속 제출 방지
      .asyncMap(_fetchResults);

  Future<List<String>> _fetchResults(String query) async {
    if (query.isEmpty) return [];
    // 실제 앱에서는 HTTP 요청이나 로컬 DB 검색
    await Future.delayed(const Duration(milliseconds: 100));
    final db = ['Flutter', 'Dart', 'Android', 'Kotlin', 'Firebase', 'Riverpod'];
    return db
        .where((item) => item.toLowerCase().contains(query.toLowerCase()))
        .toList();
  }

  void dispose() {
    _queryController.close();
  }
}

// 사용 예시
Future<void> main() async {
  final service = SearchService();

  // 결과 스트림 구독
  final sub = service.results.listen(
    (results) => print('검색 결과: $results'),
    onError: (e) => print('에러: $e'),
  );

  // 빠르게 연속 입력 (디바운스로 마지막 입력만 처리)
  service.querySink.add('F');
  service.querySink.add('Fl');
  service.querySink.add('Flu');
  await Future.delayed(const Duration(milliseconds: 400)); // 'Flutter' 쿼리 처리됨
  service.querySink.add('D');
  service.querySink.add('Da');
  await Future.delayed(const Duration(milliseconds: 400)); // 'Dart' 쿼리 처리됨

  await sub.cancel();
  service.dispose();
}
```

이 예제에서 핵심 포인트는 다음과 같습니다:

- `StreamController`의 `onListen`, `onPause`, `onResume`, `onCancel` 콜백을 통해 업스트림 구독의 생명주기를 소비자의 구독 상태와 **정확히 동기화**합니다.
- `cancelOnError: false`로 설정하여 에러가 발생해도 스트림이 종료되지 않도록 합니다.
- `distinct()`를 추가하여 동일한 쿼리가 연속으로 처리되는 것을 방지합니다.

## 실제 구현 예제 2: Broadcast StreamController로 앱 전역 이벤트 버스 구현

단일 구독 스트림과 달리 여러 위젯이 동시에 이벤트를 수신해야 할 때는 Broadcast Stream이 필요합니다. 아래는 타입 안전한 이벤트 버스를 `StreamController.broadcast()`로 구현하는 예제입니다.

```dart
import 'dart:async';

// 이벤트 타입 정의
sealed class AppEvent {}

class UserLoggedIn extends AppEvent {
  final String userId;
  UserLoggedIn(this.userId);
}

class UserLoggedOut extends AppEvent {}

class CartUpdated extends AppEvent {
  final int itemCount;
  CartUpdated(this.itemCount);
}

/// 타입 안전한 글로벌 이벤트 버스
class EventBus {
  EventBus._();
  static final EventBus instance = EventBus._();

  final _controller = StreamController<AppEvent>.broadcast();

  /// 이벤트 발행
  void emit(AppEvent event) {
    if (!_controller.isClosed) {
      _controller.add(event);
    }
  }

  /// 특정 타입의 이벤트만 구독 (Dart 패턴 매칭 활용)
  Stream<T> on<T extends AppEvent>() {
    return _controller.stream.whereType<T>();
  }

  void dispose() => _controller.close();
}

// --- Widget A: 이벤트 발행 ---
class AuthRepository {
  void login(String userId) {
    // 로그인 처리 후
    EventBus.instance.emit(UserLoggedIn(userId));
  }

  void logout() {
    EventBus.instance.emit(UserLoggedOut());
  }
}

// --- Widget B: 사용자 로그인 이벤트 수신 ---
class ProfileWidget {
  late final StreamSubscription<UserLoggedIn> _sub;

  void init() {
    _sub = EventBus.instance.on<UserLoggedIn>().listen((event) {
      print('프로필 위젯: 사용자 ${event.userId} 로그인됨');
    });
  }

  void dispose() => _sub.cancel();
}

// --- Widget C: 장바구니 업데이트 이벤트 수신 ---
class CartBadgeWidget {
  late final StreamSubscription<CartUpdated> _sub;

  void init() {
    _sub = EventBus.instance.on<CartUpdated>().listen((event) {
      print('장바구니 뱃지: ${event.itemCount}개 아이템');
    });
  }

  void dispose() => _sub.cancel();
}

Future<void> main() async {
  final auth = AuthRepository();
  final profile = ProfileWidget()..init();
  final cart = CartBadgeWidget()..init();

  auth.login('user_123');
  EventBus.instance.emit(CartUpdated(3));
  auth.logout();

  await Future.delayed(Duration.zero); // 이벤트 처리 대기

  profile.dispose();
  cart.dispose();
  EventBus.instance.dispose();
}
```

**Broadcast Stream의 핵심 동작 방식**: `StreamController.broadcast()`로 생성된 스트림에서는 `add()`를 호출하는 시점에 현재 등록된 모든 리스너에게 **동기적**으로 이벤트가 전달됩니다(기본적으로 `sync: false` 옵션이므로 실제로는 마이크로태스크 큐를 통해 비동기 전달). 리스너가 없는 상태에서 발행된 이벤트는 아무에게도 전달되지 않고 소실됩니다.

## StreamTransformer 내부 구조 이해

`StreamTransformer<S, T>`는 `Stream<S>`를 받아 `Stream<T>`를 반환하는 변환기입니다. 내부적으로 `bind(Stream<S> stream) → Stream<T>` 메서드 하나만 구현하면 됩니다.

Dart SDK는 두 가지 편의 팩토리를 제공합니다:

```dart
// 간단한 경우: fromHandlers 팩토리
final upperCaseTransformer = StreamTransformer<String, String>.fromHandlers(
  handleData: (data, sink) => sink.add(data.toUpperCase()),
  handleError: (error, stack, sink) => sink.addError(error, stack),
  handleDone: (sink) => sink.close(),
);

// 복잡한 경우: StreamTransformerBase 상속 (예제 1의 DebounceTransformer)
```

`fromHandlers`는 내부적으로 `StreamController`를 생성하고 콜백을 연결합니다. 단, 이 팩토리는 single-subscription 스트림에 대한 pause/resume 신호를 자동으로 업스트림에 전파하지 않으므로, 백프레셔(backpressure)가 중요한 경우에는 직접 `StreamTransformerBase`를 구현하는 것이 더 안전합니다.

## 주의사항 및 실전 팁

### 1. StreamSubscription을 반드시 취소하라

가장 흔한 메모리 누수 원인입니다. `StatefulWidget`의 `dispose()`에서 반드시 취소해야 합니다.

```dart
class _MyWidgetState extends State<MyWidget> {
  StreamSubscription? _subscription;

  @override
  void initState() {
    super.initState();
    _subscription = myStream.listen((event) { /* ... */ });
  }

  @override
  void dispose() {
    _subscription?.cancel(); // 필수!
    super.dispose();
  }
}
```

### 2. `cancelOnError` 기본값에 주의

`Stream.listen()`의 `cancelOnError` 기본값은 `false`입니다. 하지만 `await for` 루프에서는 에러 발생 시 루프가 종료됩니다. 일관성을 위해 명시적으로 설정하세요.

### 3. `isClosed` 확인 후 `add` 호출

`StreamController.close()`를 호출한 후 `add()`를 호출하면 `StateError`가 발생합니다. 비동기 콜백에서 컨트롤러에 데이터를 추가할 때는 항상 확인하세요.

```dart
if (!_controller.isClosed) {
  _controller.add(value);
}
```

### 4. `sync: true` 옵션은 신중하게 사용

`StreamController(sync: true)`로 생성하면 `add()` 호출 즉시 동기적으로 리스너에게 이벤트가 전달됩니다. 이는 예상치 못한 재진입(reentrancy) 문제를 일으킬 수 있으므로, 성능이 절대적으로 중요한 내부 구현이 아니라면 기본값(async)을 사용하세요.

### 5. `StreamBuilder` 보다 `AsyncSnapshot` 상태를 명시적으로 처리

Flutter `StreamBuilder`는 `ConnectionState.waiting`, `ConnectionState.active`, `ConnectionState.done` 세 가지 상태를 가집니다. `snapshot.hasError`와 `snapshot.hasData`를 항상 함께 체크하는 방어적 코딩을 습관화하세요.

```dart
StreamBuilder<int>(
  stream: myStream,
  builder: (context, snapshot) {
    if (snapshot.connectionState == ConnectionState.waiting) {
      return const CircularProgressIndicator();
    }
    if (snapshot.hasError) {
      return Text('에러: ${snapshot.error}');
    }
    if (!snapshot.hasData) {
      return const Text('데이터 없음');
    }
    return Text('값: ${snapshot.data}');
  },
)
```

### 6. 여러 스트림 합성 시 `StreamGroup` 사용 고려

`async` 패키지의 `StreamGroup`은 여러 스트림을 하나의 스트림으로 병합할 때 유용합니다. 직접 `StreamController`를 생성하고 다수의 구독을 관리하는 것보다 훨씬 안전하게 생명주기를 처리해 줍니다.

## 결론

`dart:async`의 Stream 시스템은 Flutter 반응형 프로그래밍의 근간을 이루는 강력한 도구입니다. `StreamController`를 통해 생산자와 소비자를 명확히 분리하고, `StreamTransformer`로 파이프라인 패턴을 구성하면 복잡한 비동기 데이터 흐름도 선언적이고 테스트하기 쉬운 코드로 표현할 수 있습니다. Riverpod, BLoC 등 상위 레벨 라이브러리의 내부도 결국 Stream 위에 구축되어 있으므로, 이 근간을 깊이 이해하면 디버깅 역량과 아키텍처 설계 능력이 크게 향상됩니다.

## 참고 자료
- [Dart 공식 문서: Asynchronous programming — Streams](https://dart.dev/docs/tutorials/streams)
- [Dart API: StreamController class](https://api.dart.dev/stable/dart-async/StreamController-class.html)
- [Dart API: StreamTransformer class](https://api.dart.dev/stable/dart-async/StreamTransformer-class.html)
- [Dart API: dart:async library](https://api.dart.dev/stable/dart-async/dart-async-library.html)
