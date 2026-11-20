---
layout: post
title: "Flutter RxDart 심화: BehaviorSubject, PublishSubject와 반응형 프로그래밍 패턴 완전 정복"
date: 2026-11-20
categories: [android, flutter]
tags: [flutter, rxdart, reactive-programming, behaviorsubject, publishsubject, streams, dart, state-management]
---

Flutter와 Dart는 기본적으로 `Stream`을 통한 비동기 데이터 흐름을 지원합니다. 그러나 복잡한 앱을 개발하다 보면 여러 Stream을 조합하거나, 마지막 값을 "기억"하거나, 이벤트를 멀티캐스트해야 하는 상황이 빈번하게 발생합니다. 이때 **RxDart**가 그 해결책이 됩니다.

RxDart는 ReactiveX API를 Dart로 구현한 라이브러리로, Dart의 네이티브 Stream을 완전히 대체하지 않고 그 위에 풍부한 연산자와 Subject 타입들을 얹어 줍니다. 이 글에서는 RxDart의 핵심 Subject 타입들과 실전 패턴을 깊이 있게 살펴봅니다.

---

## 왜 RxDart가 필요한가?

Dart의 기본 `StreamController`는 강력하지만 몇 가지 한계가 있습니다.

### 기본 Stream의 한계

```dart
// 일반 StreamController의 문제: 마지막 값을 기억하지 못함
final controller = StreamController<int>.broadcast();
controller.sink.add(1);
controller.sink.add(2);
controller.sink.add(3);

// 이 시점에 구독한 리스너는 아무 값도 받지 못함
controller.stream.listen((value) {
  print(value); // 구독 이후의 값만 출력
});
```

UI에서는 보통 ViewModel이 이미 상태를 가지고 있는데, 새로 위젯이 빌드되면 현재 상태를 즉시 받아야 합니다. 기본 `broadcast()` 스트림은 이를 지원하지 않습니다.

또한 여러 스트림을 선언적으로 조합하는 것도 번거롭습니다. `combineLatest`, `switchMap`, `debounceTime` 같은 고급 연산자를 직접 구현하면 코드가 복잡해지고 버그가 생기기 쉽습니다.

**RxDart가 제공하는 것:**
- **Subject 타입들**: `BehaviorSubject`, `PublishSubject`, `ReplaySubject`
- **70가지 이상의 스트림 연산자**: `debounceTime`, `switchMap`, `combineLatest`, `throttleTime` 등
- **Rx 정적 팩토리**: `Rx.merge()`, `Rx.combineLatest2()`, `Rx.zip2()` 등

---

## Subject 타입 완전 정복

RxDart의 Subject는 `Stream`이자 `Sink`입니다. 즉, 값을 발행(sink)하면서 동시에 그 스트림을 구독(stream)할 수 있습니다.

### 1. BehaviorSubject — 마지막 값을 기억하는 Subject

`BehaviorSubject`는 가장 최근에 추가된 값을 캐싱하고, 새 구독자에게 즉시 그 값을 전달합니다. 초기값을 `seeded()` 팩토리로 지정할 수 있습니다.

```
새 구독자 → 즉시 현재값 수신 → 이후 모든 이벤트 수신
```

**적합한 사용 사례:**
- ViewModel의 UI 상태 (`UiState`)
- 로그인 여부, 테마, 설정값 등 앱 전역 상태
- 검색 결과, 목록 데이터

### 2. PublishSubject — 현재 이후의 이벤트만 전달

`PublishSubject`는 구독 이후에 발행된 이벤트만 구독자에게 전달합니다. 과거 값을 캐싱하지 않습니다.

```
구독 전 이벤트 → 무시됨
구독 후 이벤트 → 구독자에게 전달
```

**적합한 사용 사례:**
- 버튼 클릭, 사용자 액션
- 단발성 이벤트 (스낵바 메시지, 화면 전환 명령)
- 로그, 애널리틱스 이벤트

### 3. ReplaySubject — 과거 N개의 값을 재방출

`ReplaySubject`는 지정된 개수(`maxSize`)만큼의 과거 이벤트를 버퍼에 저장하고, 새 구독자에게 그 버퍼를 재방출합니다.

**적합한 사용 사례:**
- 최근 N개 로그/오류 메시지 표시
- 오프라인 큐 패턴
- 히스토리 기능

---

## 실제 구현 예제

### 예제 1: 검색 자동완성 — debounceTime + switchMap

실제 앱에서 가장 많이 사용되는 패턴입니다. 사용자가 타이핑할 때마다 API를 호출하면 불필요한 요청이 발생합니다. `debounceTime`으로 입력을 300ms 지연시키고, `switchMap`으로 최신 요청만 유지합니다.

```dart
import 'package:rxdart/rxdart.dart';

class SearchBloc {
  final _searchQuery = BehaviorSubject<String>.seeded('');
  final _searchResults = BehaviorSubject<AsyncValue<List<String>>>.seeded(
    const AsyncValue.data([]),
  );
  final _isLoading = BehaviorSubject<bool>.seeded(false);

  Stream<AsyncValue<List<String>>> get results => _searchResults.stream;
  Stream<bool> get isLoading => _isLoading.stream;
  String get currentQuery => _searchQuery.value;

  late final _subscription = _searchQuery.stream
      .debounceTime(const Duration(milliseconds: 300))
      .distinctUntilChanged()
      .doOnData((_) => _isLoading.add(true))
      .switchMap((query) {
        if (query.isEmpty) {
          return Stream.value(const AsyncValue<List<String>>.data([]));
        }
        return _performSearch(query)
            .asStream()
            .map((results) => AsyncValue.data(results))
            .onErrorReturn(const AsyncValue.error('검색 실패'));
      })
      .doOnData((_) => _isLoading.add(false))
      .listen(_searchResults.add);

  Future<List<String>> _performSearch(String query) async {
    // 실제 API 호출 대체
    await Future.delayed(const Duration(milliseconds: 200));
    final mockData = ['Flutter', 'Dart', 'RxDart', 'BehaviorSubject', 'Stream'];
    return mockData
        .where((item) => item.toLowerCase().contains(query.toLowerCase()))
        .toList();
  }

  void search(String query) {
    _searchQuery.add(query);
  }

  void dispose() {
    _subscription.cancel();
    _searchQuery.close();
    _searchResults.close();
    _isLoading.close();
  }
}

// 간단한 AsyncValue sealed class (Riverpod 없이 사용)
sealed class AsyncValue<T> {
  const AsyncValue();

  const factory AsyncValue.data(T value) = AsyncData<T>;
  const factory AsyncValue.error(String message) = AsyncError<T>;
}

class AsyncData<T> extends AsyncValue<T> {
  final T value;
  const AsyncData(this.value);
}

class AsyncError<T> extends AsyncValue<T> {
  final String message;
  const AsyncError(this.message);
}
```

이 패턴의 핵심은 `switchMap`입니다. 새 검색어가 들어오면 이전 API 요청은 자동으로 취소(구독 해제)되고 최신 요청만 유지됩니다.

---

### 예제 2: 대시보드 상태 조합 — Rx.combineLatest와 멀티 Subject

여러 독립적인 데이터 스트림을 조합하여 하나의 UI 상태를 만드는 패턴입니다.

```dart
import 'package:rxdart/rxdart.dart';
import 'package:flutter/material.dart';

// 데이터 모델
class User {
  final String name;
  final String email;
  const User({required this.name, required this.email});
}

class AppNotification {
  final String title;
  final bool isRead;
  const AppNotification({required this.title, this.isRead = false});
}

class DashboardState {
  final User? user;
  final List<AppNotification> notifications;
  final int unreadCount;
  final bool isOnline;

  const DashboardState({
    this.user,
    this.notifications = const [],
    this.unreadCount = 0,
    this.isOnline = false,
  });

  DashboardState copyWith({
    User? user,
    List<AppNotification>? notifications,
    int? unreadCount,
    bool? isOnline,
  }) {
    return DashboardState(
      user: user ?? this.user,
      notifications: notifications ?? this.notifications,
      unreadCount: unreadCount ?? this.unreadCount,
      isOnline: isOnline ?? this.isOnline,
    );
  }
}

class DashboardBloc {
  // 각 도메인별 Subject
  final _user = BehaviorSubject<User?>.seeded(null);
  final _notifications = BehaviorSubject<List<AppNotification>>.seeded([]);
  final _isOnline = BehaviorSubject<bool>.seeded(false);

  // 단발성 이벤트용 PublishSubject
  final _errorEvents = PublishSubject<String>();
  final _navigationEvents = PublishSubject<String>();

  // 여러 Subject를 결합하여 단일 DashboardState 스트림 생성
  late final Stream<DashboardState> dashboardState = Rx.combineLatest3(
    _user.stream,
    _notifications.stream,
    _isOnline.stream,
    (User? user, List<AppNotification> notifications, bool isOnline) {
      final unreadCount =
          notifications.where((n) => !n.isRead).length;
      return DashboardState(
        user: user,
        notifications: notifications,
        unreadCount: unreadCount,
        isOnline: isOnline,
      );
    },
  ).distinct();

  // 단발성 이벤트 스트림 (BuildContext 없이 화면 전환 명령)
  Stream<String> get errors => _errorEvents.stream;
  Stream<String> get navigationCommands => _navigationEvents.stream;

  // 현재 상태 동기적 읽기 (BehaviorSubject.value)
  User? get currentUser => _user.valueOrNull;
  int get currentUnreadCount =>
      _notifications.value.where((n) => !n.isRead).length;

  void setUser(User user) => _user.add(user);

  void setOnlineStatus(bool isOnline) => _isOnline.add(isOnline);

  void addNotification(AppNotification notification) {
    final current = List<AppNotification>.from(_notifications.value);
    current.insert(0, notification);
    _notifications.add(current);
  }

  void markAllAsRead() {
    final updated = _notifications.value
        .map((n) => AppNotification(title: n.title, isRead: true))
        .toList();
    _notifications.add(updated);
  }

  void navigateTo(String route) => _navigationEvents.add(route);
  void showError(String message) => _errorEvents.add(message);

  void dispose() {
    _user.close();
    _notifications.close();
    _isOnline.close();
    _errorEvents.close();
    _navigationEvents.close();
  }
}

// Flutter 위젯에서 사용
class DashboardScreen extends StatefulWidget {
  const DashboardScreen({super.key});

  @override
  State<DashboardScreen> createState() => _DashboardScreenState();
}

class _DashboardScreenState extends State<DashboardScreen> {
  final _bloc = DashboardBloc();

  @override
  void initState() {
    super.initState();
    // 단발성 이벤트 구독 (ScaffoldMessenger 사용)
    _bloc.errors.listen((msg) {
      if (mounted) {
        ScaffoldMessenger.of(context)
            .showSnackBar(SnackBar(content: Text(msg)));
      }
    });
    // 초기 데이터 로드
    _loadInitialData();
  }

  void _loadInitialData() {
    _bloc.setUser(const User(name: 'Duck', email: 'duck@example.com'));
    _bloc.setOnlineStatus(true);
    _bloc.addNotification(
      const AppNotification(title: '새 메시지가 도착했습니다'),
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('대시보드')),
      body: StreamBuilder<DashboardState>(
        stream: _bloc.dashboardState,
        builder: (context, snapshot) {
          if (!snapshot.hasData) {
            return const CircularProgressIndicator();
          }
          final state = snapshot.data!;
          return Column(
            children: [
              ListTile(
                title: Text(state.user?.name ?? '비로그인'),
                subtitle: Text('알림 ${state.unreadCount}개'),
                trailing: Icon(
                  state.isOnline ? Icons.wifi : Icons.wifi_off,
                  color: state.isOnline ? Colors.green : Colors.red,
                ),
              ),
              Expanded(
                child: ListView.builder(
                  itemCount: state.notifications.length,
                  itemBuilder: (context, index) {
                    final notification = state.notifications[index];
                    return ListTile(
                      leading: Icon(
                        notification.isRead
                            ? Icons.notifications_none
                            : Icons.notifications,
                        color:
                            notification.isRead ? Colors.grey : Colors.blue,
                      ),
                      title: Text(notification.title),
                    );
                  },
                ),
              ),
            ],
          );
        },
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: _bloc.markAllAsRead,
        child: const Icon(Icons.done_all),
      ),
    );
  }

  @override
  void dispose() {
    _bloc.dispose();
    super.dispose();
  }
}
```

---

## 핵심 연산자 레퍼런스

| 연산자 | 용도 | 대표 사용 사례 |
|--------|------|----------------|
| `debounceTime(Duration)` | N밀리초 동안 새 이벤트 없을 때만 방출 | 검색 입력 |
| `throttleTime(Duration)` | N밀리초 간격으로 첫 이벤트만 방출 | 버튼 중복 클릭 방지 |
| `distinctUntilChanged()` | 이전 값과 동일하면 무시 | 불필요한 API 호출 방지 |
| `switchMap` | 최신 내부 스트림만 유지 | 검색, 탭 전환 |
| `flatMap` / `mergeMap` | 모든 내부 스트림 병합 | 병렬 요청 |
| `concatMap` | 내부 스트림을 순서대로 처리 | 순서 보장이 필요한 작업 |
| `scan` | 누적 집계 (reduce의 스트림 버전) | 카운터, 누적 합계 |
| `withLatestFrom` | 다른 스트림의 최신값과 결합 | 버튼 클릭 + 현재 상태 |
| `Rx.combineLatest2~9` | 여러 스트림의 최신값을 결합 | 복합 UI 상태 |
| `Rx.merge` | 여러 스트림의 이벤트를 하나로 합침 | 다양한 소스의 이벤트 통합 |

---

## 주의사항과 실전 팁

### 1. Subject는 반드시 close() 해야 합니다

Subject는 내부적으로 `StreamController`를 사용하므로, 사용이 끝나면 반드시 닫아야 합니다. StatefulWidget이라면 `dispose()`에서, ViewModel이라면 `onCleared()`에서 처리합니다.

```dart
@override
void dispose() {
  _querySubject.close();
  _resultsSubject.close();
  super.dispose();
}
```

`close()`를 호출하지 않으면 `StreamController`가 GC되지 않아 메모리 누수가 발생합니다.

### 2. Subject를 직접 노출하지 마세요

외부에 Subject를 직접 노출하면 아무나 값을 발행할 수 있어 캡슐화가 깨집니다.

```dart
// 나쁜 예
final subject = BehaviorSubject<int>.seeded(0); // public

// 좋은 예: Stream만 노출
final _subject = BehaviorSubject<int>.seeded(0); // private
Stream<int> get stream => _subject.stream;        // public
```

### 3. valueOrNull vs value

`BehaviorSubject`는 `.value`로 현재값을 동기적으로 읽을 수 있지만, `close()` 이후에 접근하면 예외가 발생합니다. `valueOrNull`을 사용하면 안전하게 null을 반환합니다.

```dart
// 안전한 현재값 읽기
final currentValue = _subject.valueOrNull; // close() 후에도 null 반환

// 위험
final unsafeValue = _subject.value; // close() 후 StateError 발생 가능
```

### 4. Hot vs Cold Stream 구분

- **Cold Stream**: 구독할 때마다 새로운 데이터 시퀀스 시작 (HTTP 요청, `Stream.fromFuture`)
- **Hot Stream**: 구독 여부와 무관하게 데이터를 발행 (Subject, `broadcast()`)

`BehaviorSubject`는 Hot Stream입니다. 주의할 점은 `switchMap` 내부에서 Cold Stream(예: `http.get()`)을 반환하면, `switchMap`이 이전 구독을 취소하고 새 구독을 시작합니다. 이 동작이 의도와 다를 경우 `flatMap`이나 `concatMap`을 사용하세요.

### 5. 에러 처리

RxDart의 Subject에 에러를 추가(`addError()`)하면 구독자의 `onError` 핸들러가 호출되고 **스트림이 종료됩니다**. 에러 후에도 스트림을 계속 사용하려면:

```dart
_subject.stream
    .handleError((e) => print('에러 처리: $e'))
    // handleError 후 스트림이 종료되지 않음
    .listen((value) => print(value));
```

또는 `onErrorReturn`, `onErrorReturnWith`, `onErrorResumeNext` 연산자를 사용해 에러를 값으로 변환하세요.

### 6. StreamSubscription 취소

`switchMap`의 반환 스트림처럼, 내부에서 생성되는 구독은 RxDart가 자동으로 관리합니다. 그러나 `.listen()`으로 직접 구독을 만든 경우에는 반드시 `cancel()`을 호출해야 합니다.

```dart
class MyBloc {
  late final StreamSubscription _sub;

  MyBloc() {
    _sub = someSubject.stream
        .debounceTime(Duration(milliseconds: 200))
        .listen(_handleData);
  }

  void dispose() {
    _sub.cancel(); // 반드시 취소
    someSubject.close();
  }
}
```

---

## 마치며

RxDart는 Dart의 Stream API를 대체하지 않고 보완합니다. 기본 Stream으로 충분한 단순한 비동기 작업에는 굳이 RxDart를 사용할 필요가 없습니다. 그러나 다음과 같은 상황에서 RxDart는 코드를 획기적으로 단순하고 선언적으로 만들어 줍니다:

- 여러 데이터 소스를 조합해야 할 때
- 현재 상태를 "기억"해야 할 때
- `debounce`, `switchMap` 같은 복잡한 비동기 로직이 필요할 때
- BLoC 패턴이나 MVI 아키텍처를 구현할 때

`BehaviorSubject`는 UI 상태 관리의 핵심이고, `PublishSubject`는 단발성 이벤트 전달에 최적입니다. 두 Subject의 차이를 명확히 이해하고 목적에 맞게 사용하는 것이 RxDart 활용의 핵심입니다.

## 참고 자료
- [RxDart - pub.dev](https://pub.dev/packages/rxdart)
- [RxDart Rx class API 문서](https://pub.dev/documentation/rxdart/latest/rx/Rx-class.html)
- [RxDart 0.28.0 Changelog](https://pub.dev/packages/rxdart/changelog)
