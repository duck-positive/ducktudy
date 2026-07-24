---
layout: post
title: "Dart 3 Records · Sealed Classes · Patterns 심화: 강력한 타입 시스템으로 더 안전한 Flutter 앱 만들기"
date: 2026-07-24
categories: [android, flutter]
tags: [dart3, records, sealed-classes, pattern-matching, flutter, type-system, exhaustiveness]
---

Dart 3는 단순한 마이너 업데이트가 아니었습니다. **Records**, **Sealed Classes**, **Patterns** 세 가지 기능이 동시에 도입되면서 Dart 언어의 표현력과 타입 안전성이 근본적으로 달라졌습니다. 이 글에서는 세 기능의 동작 원리부터 실전 Flutter 패턴까지 깊이 있게 다룹니다.

---

## 1. 왜 이 세 기능이 필요했는가

Dart 2.x 시대에는 함수에서 여러 값을 반환할 때 `Map<String, dynamic>`이나 별도의 클래스를 만드는 수밖에 없었습니다. 상태 모델링에서는 `abstract class`와 `extends`로 타입 계층을 만들어도 **컴파일러가 분기 누락을 잡아주지 못했습니다**. `switch` 문은 `enum` 외에는 완전성 검사가 없어서 런타임 오류가 숨어 있었습니다.

Dart 3는 이 세 가지 고통 포인트를 한 번에 해결합니다.

| 문제 | Dart 2.x 해결책 | Dart 3 해결책 |
|---|---|---|
| 함수에서 복수 반환 | `Map` 또는 전용 클래스 | **Records** |
| 닫힌 타입 계층 | `abstract class` + 컨벤션 | **Sealed Classes** |
| 구조 분해 및 완전성 | if/else 체인 | **Patterns** |

---

## 2. Records: 일회성 익명 데이터 묶음

### 2.1 기본 문법

Records는 `(값1, 값2)` 형태의 **익명 불변 튜플**입니다. 필드에 이름을 붙일 수도 있습니다.

```dart
// 위치 필드 (positional)
(int, String) getPair() => (42, 'hello');

// 명명 필드 (named)
({int code, String message}) getResult() => (code: 200, message: 'OK');

// 혼합
(int, {String label}) getMixed() => (1, label: 'first');
```

Record 타입은 **구조적 동등성(structural equality)**을 갖습니다. 같은 타입과 값이면 `==`이 `true`를 반환합니다.

```dart
final a = (1, 'a');
final b = (1, 'a');
print(a == b); // true — 별도 equals() 구현 불필요
```

### 2.2 Records로 다중 반환값 처리하기

아래는 HTTP 응답을 파싱하는 실제 코드 예제입니다. Dart 2.x였다면 `ParsedResponse` 클래스를 따로 만들었을 것입니다.

```dart
import 'dart:convert';

/// HTTP 응답을 파싱하여 상태 코드와 데이터를 함께 반환합니다.
({int statusCode, Map<String, dynamic> data, String? error})
    parseHttpResponse(String rawBody, int httpCode) {
  if (httpCode >= 400) {
    return (statusCode: httpCode, data: {}, error: 'HTTP Error $httpCode');
  }

  try {
    final decoded = jsonDecode(rawBody) as Map<String, dynamic>;
    return (statusCode: httpCode, data: decoded, error: null);
  } on FormatException catch (e) {
    return (statusCode: httpCode, data: {}, error: 'Parse error: $e');
  }
}

void main() {
  final result = parseHttpResponse('{"id": 1, "name": "Alice"}', 200);

  // 명명 필드는 .fieldName으로 접근
  print(result.statusCode); // 200
  print(result.data['name']); // Alice
  print(result.error); // null

  // 패턴으로 한 번에 구조 분해
  final (:statusCode, :data, :error) = result;
  if (error != null) {
    print('오류 발생: $error');
  } else {
    print('성공: $statusCode, 데이터: $data');
  }
}
```

### 2.3 Records와 컬렉션 조합

Records는 `List`, `Map`, `Iterable`과 자연스럽게 결합됩니다.

```dart
List<(String, int)> leaderboard = [
  ('Alice', 1200),
  ('Bob', 980),
  ('Charlie', 1050),
];

// 점수 기준 정렬
leaderboard.sort((a, b) => b.$2.compareTo(a.$2));

for (final (name, score) in leaderboard) {
  print('$name: $score점');
}
// 출력:
// Alice: 1200점
// Charlie: 1050점
// Bob: 980점
```

---

## 3. Sealed Classes: 컴파일러가 보장하는 닫힌 타입 계층

### 3.1 `sealed` 키워드의 의미

`sealed class`는 세 가지 제약을 동시에 가집니다.

1. **같은 라이브러리(파일) 내에서만** 직접 상속/구현 가능
2. **인스턴스 직접 생성 불가** (암묵적으로 `abstract`)
3. **컴파일러가 모든 하위 타입을 파악** → 스위치 완전성 검사 활성화

```dart
// result.dart
sealed class ApiResult<T> {}

class Success<T> extends ApiResult<T> {
  final T data;
  const Success(this.data);
}

class Failure<T> extends ApiResult<T> {
  final String message;
  final int? code;
  const Failure(this.message, {this.code});
}

class Loading<T> extends ApiResult<T> {
  const Loading();
}
```

`ApiResult`가 `sealed`이므로, 이 파일 밖에서 새로운 서브클래스를 추가하는 것은 **컴파일 에러**입니다. 덕분에 컴파일러는 "ApiResult의 구체적인 타입은 오직 Success, Failure, Loading 뿐"이라는 사실을 압니다.

### 3.2 완전성 검사(Exhaustiveness Checking)

`switch` **expression**에서 sealed 타입의 모든 서브클래스를 처리하지 않으면 **컴파일 오류**가 발생합니다. 이것이 sealed class의 핵심 가치입니다.

```dart
Widget buildWidget<T>(ApiResult<T> result) {
  // switch expression — 모든 케이스를 처리해야 컴파일 통과
  return switch (result) {
    Loading() => const CircularProgressIndicator(),
    Success(:final data) => Text(data.toString()),
    Failure(:final message, :final code) =>
      Text('오류 $code: $message', style: const TextStyle(color: Colors.red)),
    // 만약 여기서 Failure 케이스를 빠뜨리면 → 컴파일 에러!
    // "The type 'ApiResult<T>' is not exhaustively matched by the switch cases"
  };
}
```

`switch` **statement**에서는 `default` 없이 모든 케이스를 다루어도 경고가 발생할 수 있습니다. 하지만 `switch expression`(값을 반환하는 형태)은 항상 완전성 검사를 강제합니다.

### 3.3 실전: Flutter BLoC 상태를 Sealed Class로 모델링

```dart
// user_state.dart
sealed class UserState {}

class UserInitial extends UserState {}

class UserLoading extends UserState {}

class UserLoaded extends UserState {
  final String name;
  final String email;
  final Uri? avatarUrl;
  const UserLoaded({
    required this.name,
    required this.email,
    this.avatarUrl,
  });
}

class UserError extends UserState {
  final String message;
  const UserError(this.message);
}

// user_bloc.dart
class UserBloc extends Bloc<UserEvent, UserState> {
  UserBloc(this._repository) : super(UserInitial()) {
    on<FetchUser>(_onFetchUser);
  }

  final UserRepository _repository;

  Future<void> _onFetchUser(FetchUser event, Emitter<UserState> emit) async {
    emit(UserLoading());
    try {
      final user = await _repository.getUser(event.userId);
      emit(UserLoaded(name: user.name, email: user.email));
    } catch (e) {
      emit(UserError(e.toString()));
    }
  }
}

// user_page.dart — switch expression으로 UI 분기
class UserPage extends StatelessWidget {
  const UserPage({super.key});

  @override
  Widget build(BuildContext context) {
    return BlocBuilder<UserBloc, UserState>(
      builder: (context, state) => switch (state) {
        UserInitial() => const Center(child: Text('사용자를 불러오세요')),
        UserLoading() => const Center(child: CircularProgressIndicator()),
        UserLoaded(:final name, :final email, :final avatarUrl) => Column(
            children: [
              if (avatarUrl != null) Image.network(avatarUrl.toString()),
              Text(name, style: Theme.of(context).textTheme.headlineMedium),
              Text(email),
            ],
          ),
        UserError(:final message) => Center(
            child: Text('오류: $message', style: const TextStyle(color: Colors.red)),
          ),
      },
    );
  }
}
```

나중에 `UserState`에 새로운 서브클래스 `UserOffline`을 추가하면, 위 `switch`를 업데이트하지 않은 모든 파일에서 즉시 **컴파일 에러**가 발생합니다. 런타임 버그 대신 **컴파일 타임 안전망**이 생깁니다.

---

## 4. Patterns: 구조 분해와 매칭의 통합

### 4.1 패턴의 종류

Dart 3의 패턴은 크게 세 가지 상황에서 사용됩니다.

- **변수 선언**: `var (x, y) = point;`
- **변수 할당**: `(a, b) = (b, a);` (swap!)
- **switch 표현식/문**

패턴 종류별 예시:

```dart
// 1. 객체 패턴 (Object Pattern)
switch (point) {
  case Point(x: > 0, y: > 0): print('1사분면');
  case Point(x: < 0, y: > 0): print('2사분면');
  default: print('기타');
}

// 2. 리스트 패턴 (List Pattern)
final [first, second, ...rest] = [1, 2, 3, 4, 5];
print(first);  // 1
print(second); // 2
print(rest);   // [3, 4, 5]

// 3. 맵 패턴 (Map Pattern)
final {'name': name, 'age': int age} = {'name': 'Alice', 'age': 30};
print('$name, $age세');

// 4. 논리 OR 패턴
switch (statusCode) {
  case 200 || 201 || 204: print('성공');
  case 400 || 422: print('클라이언트 오류');
  case 500 || 502 || 503: print('서버 오류');
}

// 5. 가드 절 (Guard Clause) — when
switch (user) {
  case User(:final age) when age >= 18: print('성인');
  case User(:final age) when age < 18: print('미성년자');
}
```

### 4.2 실전 예제: JSON 파싱을 Patterns로 안전하게

기존 JSON 파싱은 `as`, `!` 연산자로 가득한 위험한 코드였습니다. Patterns를 활용하면 구조를 명확히 표현하고 안전하게 처리할 수 있습니다.

```dart
import 'dart:convert';

sealed class ParsedJson {}

class JsonObject extends ParsedJson {
  final Map<String, dynamic> fields;
  const JsonObject(this.fields);
}

class JsonArray extends ParsedJson {
  final List<dynamic> items;
  const JsonArray(this.items);
}

class JsonPrimitive extends ParsedJson {
  final Object? value;
  const JsonPrimitive(this.value);
}

class JsonParseError extends ParsedJson {
  final String reason;
  const JsonParseError(this.reason);
}

/// JSON 문자열을 안전하게 파싱하여 sealed 타입으로 반환합니다.
ParsedJson safeParseJson(String raw) {
  try {
    final decoded = jsonDecode(raw);
    return switch (decoded) {
      Map<String, dynamic> m => JsonObject(m),
      List<dynamic> l => JsonArray(l),
      _ => JsonPrimitive(decoded),
    };
  } on FormatException catch (e) {
    return JsonParseError('잘못된 JSON 형식: ${e.message}');
  }
}

/// 파싱 결과로부터 특정 경로의 값을 추출합니다.
String extractField(ParsedJson parsed, String fieldName) {
  return switch (parsed) {
    JsonObject(:final fields) when fields.containsKey(fieldName) =>
      fields[fieldName].toString(),
    JsonObject() => '필드 "$fieldName"가 존재하지 않습니다',
    JsonArray(:final items) => '배열 (${items.length}개 항목)',
    JsonPrimitive(:final value) => '원시값: $value',
    JsonParseError(:final reason) => '파싱 실패: $reason',
  };
}

void main() {
  final inputs = [
    '{"name": "Alice", "age": 30}',
    '[1, 2, 3]',
    '"hello"',
    '{invalid json}',
  ];

  for (final input in inputs) {
    final result = safeParseJson(input);
    print(extractField(result, 'name'));
  }
  // 출력:
  // Alice
  // 배열 (3개 항목)
  // 원시값: hello
  // 파싱 실패: 잘못된 JSON 형식: ...
}
```

### 4.3 Records + Patterns: 완벽한 조합

Records의 구조 분해와 Patterns의 매칭을 결합하면 매우 표현력 있는 코드가 만들어집니다.

```dart
/// 두 점 사이의 거리와 중점을 한 번에 계산합니다.
({double distance, (double, double) midpoint})
    analyzePoints((double, double) p1, (double, double) p2) {
  final (x1, y1) = p1;
  final (x2, y2) = p2;
  final dist = ((x2 - x1) * (x2 - x1) + (y2 - y1) * (y2 - y1))
      .abs()
      .toDouble();
  return (
    distance: dart_math_sqrt(dist), // import 'dart:math' show sqrt;
    midpoint: ((x1 + x2) / 2, (y1 + y2) / 2),
  );
}

// 사용:
final result = analyzePoints((0.0, 0.0), (3.0, 4.0));
final (:distance, midpoint: (mx, my)) = result;
print('거리: $distance'); // 5.0
print('중점: ($mx, $my)'); // (1.5, 2.0)
```

---

## 5. 세 기능의 시너지: Flutter 실전 아키텍처

### 5.1 Repository 계층에서의 활용

```dart
// network_result.dart
sealed class NetworkResult<T> {
  const NetworkResult();
}

final class NetworkSuccess<T> extends NetworkResult<T> {
  final T data;
  final int statusCode;
  const NetworkSuccess(this.data, {this.statusCode = 200});
}

final class NetworkError<T> extends NetworkResult<T> {
  final String message;
  final NetworkErrorType type;
  const NetworkError(this.message, this.type);
}

final class NetworkLoading<T> extends NetworkResult<T> {
  const NetworkLoading();
}

enum NetworkErrorType { timeout, noConnection, serverError, unauthorized, unknown }

// user_repository.dart
class UserRepository {
  final ApiClient _client;
  UserRepository(this._client);

  Future<NetworkResult<User>> fetchUser(String userId) async {
    try {
      final (:statusCode, :body) = await _client.get('/users/$userId');

      return switch (statusCode) {
        200 => NetworkSuccess(User.fromJson(jsonDecode(body))),
        401 => NetworkError('인증이 필요합니다', NetworkErrorType.unauthorized),
        int s when s >= 500 =>
          NetworkError('서버 오류: $s', NetworkErrorType.serverError),
        _ => NetworkError('알 수 없는 오류: $statusCode', NetworkErrorType.unknown),
      };
    } on TimeoutException {
      return NetworkError('요청 시간 초과', NetworkErrorType.timeout);
    } on SocketException {
      return NetworkError('네트워크 연결 없음', NetworkErrorType.noConnection);
    }
  }
}
```

### 5.2 UI 계층에서의 완전성 보장

```dart
// 리포지토리에서 받은 결과를 switch로 처리 — 컴파일러가 누락 케이스를 잡아줍니다.
Future<void> loadUser(String userId) async {
  final result = await _repository.fetchUser(userId);

  switch (result) {
    case NetworkLoading():
      // 이 케이스는 실제로 발생하지 않지만, sealed class이므로 처리해야 함
      break;
    case NetworkSuccess(:final data, :final statusCode):
      _showUserProfile(data);
      _analytics.track('user_loaded', {'status': statusCode});
    case NetworkError(:final message, type: NetworkErrorType.unauthorized):
      _navigateToLogin();
    case NetworkError(:final message, type: NetworkErrorType.noConnection):
      _showOfflineSnackbar();
    case NetworkError(:final message, :final type):
      _showErrorDialog(message);
  }
}
```

---

## 6. 주의사항과 마이그레이션 팁

### 6.1 Records는 클래스가 아닙니다

Records는 익명 불변 구조체입니다. **메서드나 상속이 없습니다**. 복잡한 로직이 필요한 경우에는 여전히 클래스를 사용해야 합니다. Records는 "함수에서 잠깐 묶어 반환할 데이터"에 최적입니다.

```dart
// 좋은 사용 예
(String, int) getNameAndAge() => ('Alice', 30);

// 부적절한 사용 예 — 메서드가 필요하다면 클래스를 써야 합니다
// Records에 toJson() 같은 메서드를 추가할 수 없습니다
```

### 6.2 `sealed`와 `final`, `base`, `interface` 구분

Dart 3는 클래스 한정자(class modifier)를 더 세분화했습니다.

| 한정자 | 의미 |
|---|---|
| `sealed` | 같은 라이브러리에서만 확장 가능, 완전성 검사 활성화 |
| `final` | 어디서도 확장/구현 불가 |
| `base` | 확장은 가능하지만 구현(implements)은 불가 |
| `interface` | 구현(implements)은 가능하지만 확장(extends)은 불가 |

상태 모델링에는 `sealed`, 변경을 막으려면 `final`을 사용합니다.

### 6.3 switch expression vs switch statement

`switch expression`은 항상 값을 반환해야 하므로 완전성 검사가 강제됩니다. `switch statement`는 `default`를 생략하면 일부 케이스를 처리하지 않을 수 있습니다. **UI 렌더링 로직**에는 `switch expression`을 선호하세요.

```dart
// 권장: switch expression (완전성 강제)
final widget = switch (state) {
  Loading() => const Spinner(),
  Loaded(:final data) => DataView(data: data),
  Error(:final msg) => ErrorView(message: msg),
};

// 덜 안전: switch statement (실수로 케이스를 빠뜨릴 수 있음)
switch (state) {
  case Loading():
    showSpinner();
  // 만약 Error를 빠뜨려도 컴파일러가 경고하지 않을 수 있습니다
}
```

### 6.4 Dart 버전 요구사항

Records, Sealed Classes, Patterns는 모두 **Dart 3.0** (Flutter 3.10) 이상이 필요합니다. `pubspec.yaml`의 `environment.sdk`를 확인하세요.

```yaml
environment:
  sdk: '>=3.0.0 <4.0.0'
```

### 6.5 기존 코드 점진적 마이그레이션

한 번에 모든 코드를 바꿀 필요는 없습니다. 다음 순서를 권장합니다.

1. 새로 추가하는 상태 클래스부터 `sealed class`로 설계합니다.
2. 여러 값을 반환하는 헬퍼 함수에 Records를 적용합니다.
3. 기존 `if-else` 타입 분기를 `switch expression`으로 교체합니다.
4. 컴파일러가 잡아주는 누락 케이스를 하나씩 해결합니다.

---

## 마치며

Dart 3의 **Records**, **Sealed Classes**, **Patterns**는 개별 기능이 아닌 하나의 통합된 언어 기능 묶음으로 설계되었습니다. Records로 데이터를 묶고, Sealed Classes로 타입 계층을 닫고, Patterns로 구조를 분해하는 흐름이 자연스럽게 이어집니다.

이 세 기능을 제대로 활용하면 런타임 오류 대신 컴파일 에러로, `if-else` 체인 대신 선언적 패턴으로, 임시 Map 대신 타입 안전한 Records로 코드가 진화합니다. Flutter 앱의 상태 관리 계층에서 이 패턴을 먼저 적용해보면 효과를 바로 체감할 수 있습니다.

---

## 참고 자료

- [Dart Language: Records](https://dart.dev/language/records)
- [Dart Language: Patterns](https://dart.dev/language/patterns)
- [Dart Language: Class Modifiers (sealed)](https://dart.dev/language/class-modifiers#sealed)
- [Announcing Dart 3 (Dart Blog)](https://medium.com/dartlang/announcing-dart-3-53f065a10635)
