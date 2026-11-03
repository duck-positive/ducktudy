---
layout: post
title: "Flutter get_it & injectable 심화: 서비스 로케이터와 코드 생성으로 의존성 주입 완전 정복"
date: 2026-11-03
categories: [android, flutter]
tags: [flutter, dart, get_it, injectable, dependency-injection, clean-architecture, service-locator]
---

Flutter 앱의 규모가 커질수록 클래스 간 의존 관계가 복잡해지고, 각 화면마다 동일한 객체를 직접 생성하는 코드가 반복됩니다. 이 문제를 근본적으로 해결하는 것이 **의존성 주입(Dependency Injection, DI)**이며, Flutter 생태계에서 가장 널리 쓰이는 조합이 `get_it`과 `injectable`입니다. 이 글에서는 두 패키지의 내부 동작 원리부터 실전 패턴까지 완전히 정복합니다.

---

## 의존성 주입이란 무엇인가?

의존성 주입이란 객체가 자신이 사용하는 의존 객체(Dependency)를 **직접 생성하지 않고 외부에서 주입받는** 설계 원칙입니다. 예를 들어 `UserRepository`가 `HttpClient`를 직접 `HttpClient()` 로 생성한다면, 테스트할 때 실제 HTTP 요청이 발생합니다. 반면 생성자 주입을 사용하면 테스트에서 Mock 객체를 쉽게 교체할 수 있습니다.

```dart
// ❌ 안티패턴: 직접 생성
class UserRepository {
  final _client = HttpClient(); // 테스트 불가

  Future<User> getUser(int id) async { ... }
}

// ✅ 의존성 주입: 외부에서 주입
class UserRepository {
  final HttpClient _client; // 외부에서 받음

  UserRepository(this._client);
}
```

의존성 주입의 핵심 이점은 세 가지입니다.

1. **테스트 용이성**: Mock 객체로 교체 가능
2. **관심사 분리**: 객체 생성과 사용 로직이 분리됨
3. **재사용성**: 같은 인스턴스를 여러 곳에서 공유 가능

---

## get_it: 서비스 로케이터 패턴

`get_it`은 서비스 로케이터(Service Locator) 패턴을 구현한 패키지입니다. 앱 시작 시 한 번만 객체를 등록하고, 이후 어디서든 `BuildContext` 없이 꺼내 쓸 수 있습니다.

```yaml
# pubspec.yaml
dependencies:
  get_it: ^9.3.0
  injectable: ^3.0.0

dev_dependencies:
  injectable_generator: ^3.1.3
  build_runner: ^2.4.0
```

`get_it`만 단독으로 사용하면 모든 등록 코드를 직접 작성해야 합니다.

```dart
final getIt = GetIt.instance;

void setupLocator() {
  getIt.registerLazySingleton<HttpClient>(() => HttpClient());
  getIt.registerLazySingleton<UserRepository>(() => UserRepository(getIt()));
  getIt.registerFactory<LoginViewModel>(() => LoginViewModel(getIt()));
}
```

앱 규모가 커지면 이 설정 파일이 수백 줄로 불어나고 유지보수가 힘들어집니다. 이 문제를 해결하는 것이 `injectable`입니다.

---

## injectable: 애노테이션 기반 코드 생성

`injectable`은 `get_it` 위에서 동작하는 코드 생성기입니다. 클래스에 애노테이션만 붙이면 `build_runner`가 자동으로 등록 코드를 생성해줍니다.

### 기본 설정

```dart
// injection.dart
import 'package:get_it/get_it.dart';
import 'package:injectable/injectable.dart';
import 'injection.config.dart'; // build_runner가 생성

final getIt = GetIt.instance;

@InjectableInit(
  initializerName: 'init',
  preferRelativeImports: true,
  asExtension: true,
)
Future<void> configureDependencies() async => getIt.init();
```

```dart
// main.dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await configureDependencies();
  runApp(const MyApp());
}
```

---

## 실전 예제 1: 클린 아키텍처와 injectable 통합

`injectable`의 핵심 애노테이션은 다음과 같습니다.

| 애노테이션 | get_it 등록 방식 | 설명 |
|---|---|---|
| `@injectable` | `registerFactory` | 호출마다 새 인스턴스 |
| `@singleton` | `registerSingleton` | 즉시 생성되는 단일 인스턴스 |
| `@lazySingleton` | `registerLazySingleton` | 첫 접근 시 생성되는 단일 인스턴스 |

다음은 클린 아키텍처 레이어를 injectable로 구성한 완전한 예제입니다.

```dart
// --- Data Layer ---

// HttpClient는 외부 라이브러리이므로 @module로 래핑
@module
abstract class NetworkModule {
  @lazySingleton
  Dio get dio => Dio(BaseOptions(baseUrl: 'https://api.example.com'));
}

@LazySingleton(as: UserRepository)
class UserRepositoryImpl implements UserRepository {
  final Dio _dio;

  UserRepositoryImpl(this._dio); // get_it이 자동으로 Dio 주입

  @override
  Future<User> getUser(int id) async {
    final response = await _dio.get('/users/$id');
    return User.fromJson(response.data as Map<String, dynamic>);
  }
}

// --- Domain Layer ---

@injectable
class GetUserUseCase {
  final UserRepository _repository;

  GetUserUseCase(this._repository);

  Future<User> execute(int id) => _repository.getUser(id);
}

// --- Presentation Layer ---

@injectable
class UserViewModel extends ChangeNotifier {
  final GetUserUseCase _getUserUseCase;

  UserViewModel(this._getUserUseCase);

  User? user;
  bool isLoading = false;

  Future<void> loadUser(int id) async {
    isLoading = true;
    notifyListeners();
    try {
      user = await _getUserUseCase.execute(id);
    } finally {
      isLoading = false;
      notifyListeners();
    }
  }
}
```

코드 생성 명령을 실행합니다.

```bash
dart run build_runner build --delete-conflicting-outputs
```

생성된 `injection.config.dart`는 자동으로 모든 의존성 관계를 등록합니다. 개발자는 등록 순서나 의존성 그래프를 직접 관리할 필요가 없습니다.

---

## 실전 예제 2: 환경별 등록과 비동기 초기화

실무에서는 dev/staging/prod 환경마다 다른 구현체를 사용하고, SharedPreferences처럼 비동기 초기화가 필요한 의존성을 다룰 때가 많습니다.

```dart
// --- 환경별 등록 ---

// dev 환경에서는 Mock 구현 사용
@Environment('dev')
@LazySingleton(as: AnalyticsService)
class MockAnalyticsService implements AnalyticsService {
  @override
  void track(String event) => debugPrint('[MOCK] event: $event');
}

// prod 환경에서는 실제 Firebase 구현 사용
@Environment('prod')
@LazySingleton(as: AnalyticsService)
class FirebaseAnalyticsService implements AnalyticsService {
  @override
  void track(String event) => FirebaseAnalytics.instance.logEvent(name: event);
}

// --- 비동기 의존성 ---

@module
abstract class StorageModule {
  // 비동기 초기화가 필요한 의존성: @preResolve로 표시
  @preResolve
  @lazySingleton
  Future<SharedPreferences> get prefs => SharedPreferences.getInstance();
}

@lazySingleton
class UserPreferences {
  final SharedPreferences _prefs;

  UserPreferences(this._prefs); // build_runner가 await된 값을 자동 주입

  String? get accessToken => _prefs.getString('access_token');

  Future<void> saveAccessToken(String token) =>
      _prefs.setString('access_token', token);
}

// --- 환경 지정하여 초기화 ---

// main.dart (prod)
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await configureDependencies(Environment.prod); // 환경 지정
  runApp(const MyApp());
}

// main_dev.dart (dev)
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await configureDependencies(Environment.dev);
  runApp(const MyApp());
}
```

### 스코프(Scope) 활용

로그인/로그아웃처럼 특정 사용자 세션에만 존재해야 하는 의존성은 스코프로 관리합니다.

```dart
// 로그인 성공 시 사용자 세션 스코프 생성
getIt.pushNewScope(scopeName: 'session');
getIt.registerLazySingleton<AuthToken>(() => AuthToken(token: jwt));

// 로그아웃 시 스코프 해제 (세션 관련 객체 전체 소멸)
await getIt.popScope();
```

---

## 주의사항 및 실전 팁

### 1. 코드 생성 파일을 .gitignore에서 제외하지 마세요

`injection.config.dart`는 커밋에 포함해야 합니다. `.gitignore`에 넣으면 CI/CD에서 `build_runner` 없이 빌드가 실패합니다.

### 2. `watch` 모드로 개발 효율 높이기

```bash
dart run build_runner watch --delete-conflicting-outputs
```

파일 저장 시 자동으로 코드를 재생성합니다. 개발 중에는 `build` 대신 `watch`를 사용하세요.

### 3. 테스트에서 의존성 교체

```dart
setUp(() {
  // 테스트용 getIt 인스턴스 초기화
  final testGetIt = GetIt.asNewInstance();
  testGetIt.registerFactory<UserRepository>(() => MockUserRepository());
});

tearDown(() => getIt.reset());
```

`getIt.reset()`으로 테스트 간 상태를 격리하는 것이 중요합니다.

### 4. `@injectable` vs `@singleton` 선택 기준

- **Repository, UseCase, Service**: 상태를 갖지 않으므로 `@lazySingleton`이 적합
- **ViewModel / Presenter**: 화면마다 독립 상태가 필요하므로 `@injectable` (Factory) 사용
- **공유 상태 (Cart, Auth)**: 전역 상태이므로 `@singleton` 사용

### 5. 순환 의존성 주의

A→B→A 형태의 순환 의존성이 있으면 get_it은 `StateError`를 던집니다. 이런 경우 의존성 방향을 재설계하거나 인터페이스를 도입해 결합도를 낮추세요.

---

## 마치며

`get_it`과 `injectable`의 조합은 Flutter 앱에서 의존성 주입을 구현하는 가장 실용적인 방법입니다. `injectable`의 코드 생성 덕분에 설정 코드의 반복 없이 수백 개의 의존성을 선언적으로 관리할 수 있습니다. 클린 아키텍처의 각 레이어가 인터페이스에만 의존하도록 구성하면, 단위 테스트가 쉬워지고 구현체 교체가 자유로운 유지보수하기 좋은 앱이 완성됩니다.

## 참고 자료
- [get_it 패키지 (pub.dev)](https://pub.dev/packages/get_it)
- [injectable 패키지 (pub.dev)](https://pub.dev/packages/injectable)
- [injectable_generator 패키지 (pub.dev)](https://pub.dev/packages/injectable_generator)
- [Mastering get_it: A Comprehensive Guide — Simform Engineering](https://medium.com/simform-engineering/mastering-get-it-a-comprehensive-guide-to-dependency-injection-in-flutter-db99bfb305bb)
