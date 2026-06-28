---
layout: post
title: "Flutter BLoC 심화: Cubit·BLoC·EventTransformer로 완전한 이벤트 기반 상태 관리 구현하기"
date: 2026-06-28
categories: [android, flutter]
tags: [flutter, bloc, cubit, state-management, event-transformer, dart]
---

Flutter 생태계에서 상태 관리 라이브러리를 고를 때 가장 많이 거론되는 것이 **BLoC(Business Logic Component)** 패턴입니다. Riverpod이 선언형 의존성 주입에 강점을 둔다면, BLoC은 **이벤트 → 상태의 단방향 흐름**과 **명시적인 이벤트 변환(EventTransformer)** 덕분에 복잡한 비즈니스 로직을 예측 가능하게 관리하는 데 탁월합니다. 이 글에서는 `Cubit`부터 `BLoC`, 그리고 실무에서 자주 쓰이는 `EventTransformer` 패턴까지 코드 중심으로 깊이 파헤칩니다.

---

## 개념 설명: BLoC 패턴의 핵심 구조

BLoC 패턴은 구글 I/O 2018에서 처음 소개되었으며, 그 핵심은 **UI와 비즈니스 로직의 완전한 분리**입니다. `bloc` 라이브러리(현재 9.x 버전)는 두 가지 핵심 클래스를 제공합니다.

### Cubit — 함수 기반의 단순한 BLoC

`Cubit`은 `BlocBase`를 상속하는 가장 단순한 형태로, **메서드를 직접 호출**해 상태를 변경합니다. 이벤트 클래스 없이 `emit()`으로 새 상태를 방출합니다.

```
Cubit<State>
  └── emit(newState)  ← 메서드 직접 호출
```

### BLoC — 이벤트 기반의 완전한 패턴

`BLoC`은 `Cubit`을 확장해 **이벤트(Event) → 처리(on<Event>) → 상태(State)** 흐름을 강제합니다. 이벤트를 Stream으로 받아 처리하기 때문에 `EventTransformer`를 통한 고급 제어(debounce, throttle, sequential 등)가 가능합니다.

```
BLoC<Event, State>
  └── add(Event) → on<Event>(handler, transformer) → emit(State)
```

### 상태의 불변성과 Equatable

BLoC에서 상태는 **불변(immutable)** 이어야 하며, `Equatable` 패키지를 사용해 객체 동등성을 올바르게 처리해야 합니다. 그렇지 않으면 참조가 달라도 동일한 데이터임을 BLoC이 감지하지 못해 불필요한 리빌드가 발생합니다.

---

## 왜 필요한가? BLoC 패턴을 선택하는 이유

### 1. 예측 가능한 단방향 데이터 흐름

UI에서 `add(Event)` 하나만 호출하면, 나머지 처리는 BLoC 내부에서만 일어납니다. 상태 변경의 원인이 항상 이벤트로 추적되므로, 버그 재현이 용이합니다.

### 2. 테스트 용이성

`bloc_test` 패키지를 이용하면 이벤트 시퀀스를 주입하고 예상 상태 흐름을 단위 테스트로 검증할 수 있습니다. UI 없이 순수 Dart로 비즈니스 로직 전체를 테스트합니다.

### 3. EventTransformer를 통한 이벤트 제어

`bloc_concurrency` 패키지는 네 가지 EventTransformer를 제공합니다.

| Transformer | 동작 |
|---|---|
| `concurrent()` | 모든 이벤트를 동시에 처리 (기본값) |
| `sequential()` | 이벤트를 순서대로 하나씩 처리 |
| `droppable()` | 처리 중인 이벤트가 있으면 새 이벤트 무시 |
| `restartable()` | 새 이벤트 도착 시 이전 처리를 취소하고 재시작 |

검색 기능 구현 시 `restartable()`을 사용하면 이전 API 호출을 자동으로 취소해 Race Condition을 방지합니다.

### 4. BlocObserver로 전역 로깅

앱 전체의 이벤트·상태·오류를 한 곳에서 관찰할 수 있어 디버깅과 분석에 유리합니다.

---

## 실제 구현 예제

### 예제 1: 로그인 BLoC — 이벤트 기반 인증 흐름

실무에서 가장 흔한 패턴인 로그인 상태 관리를 BLoC으로 구현합니다.

```dart
// login_event.dart
import 'package:equatable/equatable.dart';

abstract class LoginEvent extends Equatable {
  const LoginEvent();

  @override
  List<Object?> get props => [];
}

class LoginSubmitted extends LoginEvent {
  final String email;
  final String password;

  const LoginSubmitted({required this.email, required this.password});

  @override
  List<Object?> get props => [email, password];
}

class LoginReset extends LoginEvent {
  const LoginReset();
}

// login_state.dart
enum LoginStatus { initial, loading, success, failure }

class LoginState extends Equatable {
  final LoginStatus status;
  final String? errorMessage;
  final String? userToken;

  const LoginState({
    this.status = LoginStatus.initial,
    this.errorMessage,
    this.userToken,
  });

  LoginState copyWith({
    LoginStatus? status,
    String? errorMessage,
    String? userToken,
  }) {
    return LoginState(
      status: status ?? this.status,
      errorMessage: errorMessage ?? this.errorMessage,
      userToken: userToken ?? this.userToken,
    );
  }

  @override
  List<Object?> get props => [status, errorMessage, userToken];
}

// login_bloc.dart
import 'package:bloc/bloc.dart';
import 'package:bloc_concurrency/bloc_concurrency.dart';

class LoginBloc extends Bloc<LoginEvent, LoginState> {
  final AuthRepository _authRepository;

  LoginBloc({required AuthRepository authRepository})
      : _authRepository = authRepository,
        super(const LoginState()) {
    // droppable: 로그인 중 중복 요청 방지
    on<LoginSubmitted>(_onLoginSubmitted, transformer: droppable());
    on<LoginReset>(_onLoginReset);
  }

  Future<void> _onLoginSubmitted(
    LoginSubmitted event,
    Emitter<LoginState> emit,
  ) async {
    emit(state.copyWith(status: LoginStatus.loading));

    try {
      final token = await _authRepository.login(
        email: event.email,
        password: event.password,
      );
      emit(state.copyWith(
        status: LoginStatus.success,
        userToken: token,
      ));
    } on AuthException catch (e) {
      emit(state.copyWith(
        status: LoginStatus.failure,
        errorMessage: e.message,
      ));
    }
  }

  void _onLoginReset(LoginReset event, Emitter<LoginState> emit) {
    emit(const LoginState());
  }
}

// login_page.dart — UI 연결
class LoginPage extends StatelessWidget {
  const LoginPage({super.key});

  @override
  Widget build(BuildContext context) {
    return BlocProvider(
      create: (context) => LoginBloc(
        authRepository: context.read<AuthRepository>(),
      ),
      child: const LoginView(),
    );
  }
}

class LoginView extends StatefulWidget {
  const LoginView({super.key});

  @override
  State<LoginView> createState() => _LoginViewState();
}

class _LoginViewState extends State<LoginView> {
  final _emailController = TextEditingController();
  final _passwordController = TextEditingController();

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: BlocConsumer<LoginBloc, LoginState>(
        // listener: 부수효과 처리 (네비게이션, 스낵바 등)
        listener: (context, state) {
          if (state.status == LoginStatus.success) {
            Navigator.of(context).pushReplacementNamed('/home');
          } else if (state.status == LoginStatus.failure) {
            ScaffoldMessenger.of(context).showSnackBar(
              SnackBar(content: Text(state.errorMessage ?? '로그인 실패')),
            );
          }
        },
        // builder: 상태에 따른 UI 렌더링
        builder: (context, state) {
          return Padding(
            padding: const EdgeInsets.all(24.0),
            child: Column(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                TextField(
                  controller: _emailController,
                  decoration: const InputDecoration(labelText: '이메일'),
                ),
                const SizedBox(height: 16),
                TextField(
                  controller: _passwordController,
                  obscureText: true,
                  decoration: const InputDecoration(labelText: '비밀번호'),
                ),
                const SizedBox(height: 32),
                if (state.status == LoginStatus.loading)
                  const CircularProgressIndicator()
                else
                  ElevatedButton(
                    onPressed: () {
                      context.read<LoginBloc>().add(
                        LoginSubmitted(
                          email: _emailController.text,
                          password: _passwordController.text,
                        ),
                      );
                    },
                    child: const Text('로그인'),
                  ),
              ],
            ),
          );
        },
      ),
    );
  }

  @override
  void dispose() {
    _emailController.dispose();
    _passwordController.dispose();
    super.dispose();
  }
}
```

`BlocConsumer`를 사용하면 `builder`(UI 갱신)와 `listener`(네비게이션·스낵바 등 부수효과)를 하나의 위젯으로 처리할 수 있습니다. `droppable()` 트랜스포머 덕분에 버튼을 연속으로 눌러도 API 요청은 한 번만 실행됩니다.

---

### 예제 2: 실시간 검색 Bloc — restartable + debounce EventTransformer

실무에서 자주 쓰이는 검색 자동완성 기능을 `restartable()` + `debounce`로 구현합니다.

```dart
// search_event.dart
abstract class SearchEvent extends Equatable {
  const SearchEvent();
  @override
  List<Object?> get props => [];
}

class SearchQueryChanged extends SearchEvent {
  final String query;
  const SearchQueryChanged(this.query);
  @override
  List<Object?> get props => [query];
}

class SearchCleared extends SearchEvent {
  const SearchCleared();
}

// search_state.dart
abstract class SearchState extends Equatable {
  const SearchState();
  @override
  List<Object?> get props => [];
}

class SearchInitial extends SearchState {
  const SearchInitial();
}

class SearchLoading extends SearchState {
  const SearchLoading();
}

class SearchSuccess extends SearchState {
  final List<SearchResult> results;
  const SearchSuccess(this.results);
  @override
  List<Object?> get props => [results];
}

class SearchFailure extends SearchState {
  final String error;
  const SearchFailure(this.error);
  @override
  List<Object?> get props => [error];
}

// search_bloc.dart
import 'package:bloc/bloc.dart';
import 'package:bloc_concurrency/bloc_concurrency.dart';
import 'package:stream_transform/stream_transform.dart';

// 300ms debounce + restartable 조합 EventTransformer
EventTransformer<E> debounceRestartable<E>(Duration duration) {
  return (events, mapper) {
    return restartable<E>().call(events.debounce(duration), mapper);
  };
}

class SearchBloc extends Bloc<SearchEvent, SearchState> {
  final SearchRepository _searchRepository;

  SearchBloc({required SearchRepository searchRepository})
      : _searchRepository = searchRepository,
        super(const SearchInitial()) {
    on<SearchQueryChanged>(
      _onQueryChanged,
      // 300ms debounce: 타이핑이 멈춘 후 300ms 뒤에만 API 호출
      // restartable: 새 쿼리 입력 시 이전 API 호출 자동 취소
      transformer: debounceRestartable(const Duration(milliseconds: 300)),
    );
    on<SearchCleared>(_onSearchCleared);
  }

  Future<void> _onQueryChanged(
    SearchQueryChanged event,
    Emitter<SearchState> emit,
  ) async {
    final query = event.query.trim();

    if (query.isEmpty) {
      emit(const SearchInitial());
      return;
    }

    emit(const SearchLoading());

    try {
      // Emitter.forEach를 사용해 Stream 결과를 순차적으로 emit
      await emit.forEach<List<SearchResult>>(
        _searchRepository.search(query),
        onData: (results) => SearchSuccess(results),
        onError: (error, stackTrace) => SearchFailure(error.toString()),
      );
    } catch (e) {
      emit(SearchFailure(e.toString()));
    }
  }

  void _onSearchCleared(SearchCleared event, Emitter<SearchState> emit) {
    emit(const SearchInitial());
  }
}

// search_page.dart
class SearchPage extends StatelessWidget {
  const SearchPage({super.key});

  @override
  Widget build(BuildContext context) {
    return BlocProvider(
      create: (context) => SearchBloc(
        searchRepository: context.read<SearchRepository>(),
      ),
      child: const SearchView(),
    );
  }
}

class SearchView extends StatefulWidget {
  const SearchView({super.key});

  @override
  State<SearchView> createState() => _SearchViewState();
}

class _SearchViewState extends State<SearchView> {
  final _controller = TextEditingController();

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: TextField(
          controller: _controller,
          decoration: const InputDecoration(
            hintText: '검색어를 입력하세요...',
            border: InputBorder.none,
          ),
          onChanged: (query) {
            context.read<SearchBloc>().add(SearchQueryChanged(query));
          },
        ),
        actions: [
          IconButton(
            icon: const Icon(Icons.clear),
            onPressed: () {
              _controller.clear();
              context.read<SearchBloc>().add(const SearchCleared());
            },
          ),
        ],
      ),
      body: BlocBuilder<SearchBloc, SearchState>(
        builder: (context, state) {
          return switch (state) {
            SearchInitial() => const Center(child: Text('검색어를 입력하세요')),
            SearchLoading() => const Center(child: CircularProgressIndicator()),
            SearchSuccess(results: final results) => ListView.builder(
                itemCount: results.length,
                itemBuilder: (context, index) {
                  final result = results[index];
                  return ListTile(
                    leading: const Icon(Icons.search),
                    title: Text(result.title),
                    subtitle: Text(result.description),
                  );
                },
              ),
            SearchFailure(error: final error) => Center(
                child: Text('오류: $error', style: const TextStyle(color: Colors.red)),
              ),
            _ => const SizedBox.shrink(),
          };
        },
      ),
    );
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }
}
```

핵심 포인트는 커스텀 `debounceRestartable` EventTransformer입니다. `stream_transform` 패키지의 `debounce()`로 입력을 필터링한 뒤, `restartable()`로 새 이벤트가 들어오면 이전 처리를 자동 취소합니다. 사용자가 "flutter"를 타이핑하는 동안 "f", "fl", "flu", "flut", "flutt", "flutte" 요청은 모두 취소되고, 입력이 멈춘 300ms 후 "flutter"만 서버에 전송됩니다.

---

## 고급 패턴: BlocObserver와 글로벌 로깅

`BlocObserver`를 구현해 앱 전역에서 이벤트와 상태 전환을 로깅할 수 있습니다.

```dart
class AppBlocObserver extends BlocObserver {
  @override
  void onEvent(Bloc bloc, Object? event) {
    super.onEvent(bloc, event);
    debugPrint('[EVENT] ${bloc.runtimeType} ← $event');
  }

  @override
  void onTransition(Bloc bloc, Transition transition) {
    super.onTransition(bloc, transition);
    debugPrint(
      '[TRANSITION] ${bloc.runtimeType}: '
      '${transition.currentState.runtimeType} → ${transition.nextState.runtimeType}',
    );
  }

  @override
  void onError(BlocBase bloc, Object error, StackTrace stackTrace) {
    debugPrint('[ERROR] ${bloc.runtimeType}: $error');
    super.onError(bloc, error, stackTrace);
  }
}

// main.dart
void main() {
  Bloc.observer = AppBlocObserver();
  runApp(const App());
}
```

---

## 주의사항 및 실전 팁

### 1. `emit()` 호출 전 `isClosed` 확인은 불필요

`Emitter`의 `emit()`은 BLoC이 닫힌 상태에서 호출하면 자동으로 무시됩니다. 별도의 `if (!isClosed)` 가드는 필요 없습니다. 단, `BLoC.close()` 이후에도 참조를 들고 있는 곳에서는 메모리 누수가 발생할 수 있으니 `BlocProvider`를 통해 라이프사이클을 자동 관리하는 편이 안전합니다.

### 2. `listenWhen` / `buildWhen`으로 불필요한 리빌드 최소화

```dart
BlocBuilder<LoginBloc, LoginState>(
  buildWhen: (previous, current) =>
      previous.status != current.status, // 상태 타입이 바뀔 때만 리빌드
  builder: (context, state) => ...,
)
```

`BlocBuilder`와 `BlocListener`의 `buildWhen`/`listenWhen` 콜백을 활용하면 특정 필드가 변경될 때만 UI를 갱신해 성능을 최적화할 수 있습니다.

### 3. Equatable 없이 상태를 사용하면 항상 리빌드

BLoC은 `previousState == currentState`를 비교해 불필요한 emit을 차단합니다. `Equatable`을 상속하지 않는 클래스는 `==` 비교가 참조 기준으로 동작해, 동일한 값을 emit해도 매번 리빌드를 유발합니다.

### 4. MultiBlocProvider로 의존성 계층 선언

여러 BLoC이 필요한 화면은 `MultiBlocProvider`로 한 번에 선언합니다.

```dart
MultiBlocProvider(
  providers: [
    BlocProvider<LoginBloc>(
      create: (_) => LoginBloc(authRepository: authRepository),
    ),
    BlocProvider<UserBloc>(
      create: (context) => UserBloc(
        loginBloc: context.read<LoginBloc>(), // 다른 BLoC 주입 가능
      ),
    ),
  ],
  child: const HomeView(),
)
```

### 5. BLoC 테스트: bloc_test 패키지 활용

```dart
blocTest<SearchBloc, SearchState>(
  'emit [SearchLoading, SearchSuccess] when query is valid',
  build: () => SearchBloc(searchRepository: mockRepository),
  act: (bloc) => bloc.add(const SearchQueryChanged('flutter')),
  wait: const Duration(milliseconds: 300), // debounce 대기
  expect: () => [
    const SearchLoading(),
    isA<SearchSuccess>(),
  ],
);
```

`blocTest`는 `act`에서 이벤트를 주입하고 `expect`에서 방출된 상태 시퀀스를 검증합니다. `wait` 파라미터로 debounce 시간도 처리할 수 있습니다.

---

## 정리: Cubit vs BLoC 선택 기준

| 기준 | Cubit | BLoC |
|---|---|---|
| 복잡도 | 낮음 | 중~높음 |
| 이벤트 추적 | 불가 | 가능 |
| EventTransformer | 불가 | 가능 |
| 테스트 편의성 | 높음 | 매우 높음 |
| 추천 사용 | 단순 토글, 카운터, 로컬 UI 상태 | 검색, 인증, 페이지네이션, 복잡한 비즈니스 로직 |

BLoC 패턴은 초기 보일러플레이트가 더 많지만, 이벤트 추적 가능성·테스트 용이성·EventTransformer를 통한 이벤트 제어라는 강력한 장점을 제공합니다. 팀의 규모가 크거나 로직이 복잡한 앱에서는 BLoC의 명시적 구조가 장기적으로 유지보수 비용을 크게 낮춥니다.

## 참고 자료
- [flutter_bloc | Flutter package (pub.dev)](https://pub.dev/packages/flutter_bloc)
- [bloc | Dart package (pub.dev)](https://pub.dev/packages/bloc)
- [bloc_concurrency | Dart package (pub.dev)](https://pub.dev/packages/bloc_concurrency)
- [BLoC 공식 문서 (bloclibrary.dev)](https://bloclibrary.dev/)
