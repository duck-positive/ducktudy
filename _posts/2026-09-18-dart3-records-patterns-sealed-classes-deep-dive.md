---
layout: post
title: "Dart 3 심화: Records, Patterns, Sealed Classes로 타입 안전한 상태 모델링하기"
date: 2026-09-18
categories: [android, flutter]
tags: [dart3, records, patterns, sealed-class, pattern-matching, flutter, dart, state-management]
---

Dart 3.0은 Flutter 3.10과 함께 2023년 5월에 공개되었으며, 언어 수준에서 타입 시스템을 한층 강화하는 세 가지 핵심 기능을 도입했습니다: **Records**, **Pattern Matching**, **Sealed Classes**. 이 기능들은 단순한 문법 설탕이 아니라, 컴파일 타임에 버그를 잡고 코드의 의도를 명확히 표현할 수 있게 해주는 강력한 도구입니다.

이 아티클에서는 각 기능의 내부 원리와 실전 활용법, 그리고 Flutter 앱 아키텍처에 어떻게 통합하는지를 심층적으로 다룹니다.

---

## 왜 필요한가: 기존 Dart 2 코드의 한계

### 1) 여러 값 반환의 번거로움

Dart 2에서 함수가 여러 값을 반환하려면 전용 클래스나 `Map`을 사용해야 했습니다:

```dart
// Dart 2 방식 — 불필요한 클래스 정의 필요
class UserValidationResult {
  final bool isValid;
  final String? errorMessage;
  UserValidationResult({required this.isValid, this.errorMessage});
}

UserValidationResult validate(String email) { ... }
```

이 방식은 일회성 데이터를 위해 클래스를 정의해야 하므로 보일러플레이트가 늘어납니다.

### 2) 상태 분기의 안전성 부재

가장 큰 문제는 상태를 표현하는 클래스 계층에서 컴파일러가 모든 케이스를 처리했는지 검증하지 못한다는 점입니다:

```dart
// Dart 2: AuthState의 새 하위 타입을 추가해도 컴파일 오류 없음
abstract class AuthState {}
class AuthLoading extends AuthState {}
class AuthAuthenticated extends AuthState { final User user; AuthAuthenticated(this.user); }
class AuthError extends AuthState { final String message; AuthError(this.message); }

// AuthLoading을 처리하지 않아도 컴파일러가 경고하지 않음
Widget buildAuthWidget(AuthState state) {
  if (state is AuthAuthenticated) return HomeScreen(user: state.user);
  if (state is AuthError) return ErrorScreen(message: state.message);
  return const SizedBox(); // AuthLoading을 몰래 무시
}
```

나중에 `AuthRefreshing` 같은 새 상태를 추가해도, 처리하지 않은 모든 `switch` / `if-else` 체인을 직접 찾아 수정해야 합니다.

Dart 3는 **Records**와 **Sealed Classes**로 이 두 문제를 우아하게 해결합니다.

---

## Records: 익명 경량 구조체

Records는 고정된 필드 수를 가진 불변 익명 타입입니다. 클래스 정의 없이 여러 값을 타입 안전하게 묶어서 반환할 수 있으며, 구조적 동등성(structural equality)을 기본 지원합니다.

### 기본 문법

```dart
// 위치(positional) 필드
(String, int) getNameAndAge() => ('Alice', 30);

// 명명(named) 필드
({String name, int age}) getUser() => (name: 'Alice', age: 30);

// 혼합 사용
(String, {int age, String email}) getProfile() =>
    ('Alice', age: 30, email: 'alice@example.com');

// 사용법
final (name, age) = getNameAndAge();   // 위치 필드 구조 분해
final (:name, :age) = getUser();       // 명명 필드 구조 분해
```

### 구조적 동등성

Records는 필드 값 기준으로 `==`를 자동 구현합니다:

```dart
var a = ('hello', 42);
var b = ('hello', 42);
print(a == b);           // true
print(a.hashCode == b.hashCode); // true

// 클래스라면 동일 인스턴스가 아니면 false
```

### typedef로 가독성 향상

자주 사용하는 Record 타입은 `typedef`로 이름을 붙일 수 있습니다:

```dart
typedef ValidationResult = ({bool isValid, String? errorMessage});
typedef Coordinate = (double lat, double lng);
```

---

## Pattern Matching: 데이터를 분해하는 새로운 방법

Dart 3의 패턴은 `switch` 표현식, `if-case` 문, 그리고 변수 선언에서 데이터를 매칭·분해하는 데 사용됩니다.

### switch 표현식 (값을 반환하는 switch)

Dart 3에서 `switch`는 표현식이 될 수 있습니다. 화살표 구문으로 각 케이스의 결과를 반환합니다:

```dart
String describeHttpStatus(int code) => switch (code) {
  200 => '성공',
  201 => '생성됨',
  400 => '잘못된 요청',
  401 => '인증 필요',
  403 => '접근 금지',
  404 => '찾을 수 없음',
  500 => '서버 내부 오류',
  >= 500 => '서버 오류',
  >= 400 => '클라이언트 오류',
  _ => '알 수 없는 상태 코드',
};
```

### Guard Clause (when 절)

`when` 절로 패턴에 추가 조건을 붙일 수 있습니다:

```dart
String classifyTemperature(double celsius) => switch (celsius) {
  < 0 => '결빙',
  var t when t < 10 => '매우 차가움',
  var t when t < 20 => '시원함',
  var t when t < 30 => '적당함',
  _ => '더움',
};
```

### List와 Map 패턴

```dart
String describeList(List<int> nums) => switch (nums) {
  [] => '빈 리스트',
  [var only] => '원소 하나: $only',
  [var first, var second] => '두 원소: $first, $second',
  [var first, ..., var last] => '첫 원소: $first, 마지막: $last (총 ${nums.length}개)',
};
```

### if-case 문

`if-case`는 단일 패턴을 검사하면서 동시에 구조 분해할 수 있습니다:

```dart
void processShape(Object shape) {
  if (shape case Circle(radius: var r) when r > 0) {
    print('유효한 원: 반지름 $r, 넓이 ${3.14 * r * r}');
  }
}
```

---

## Sealed Classes: 닫힌 클래스 계층과 소진성 검사

`sealed` 키워드로 선언된 클래스는 **같은 파일(library) 내에서만** 하위 타입을 가질 수 있습니다. 이를 통해 컴파일러가 `switch`에서 모든 케이스가 처리되었는지 **소진성 검사(exhaustiveness checking)**를 수행합니다.

### sealed vs abstract 비교

| 항목 | `abstract` | `sealed` |
|---|---|---|
| 외부 라이브러리 상속 | 가능 | 불가능 |
| 소진성 검사 | 없음 | 있음 |
| 인스턴스 생성 | 불가 | 불가 |
| 용도 | 공개 확장 가능 타입 | 닫힌 상태 집합 |

---

## 코드 예제 1: Sealed Classes + Pattern Matching으로 네트워크 상태 관리

실제 Flutter 앱에서 API 호출 상태를 타입 안전하게 모델링하는 예제입니다.

```dart
// api_state.dart

sealed class ApiState<T> {}

final class ApiLoading<T> extends ApiState<T> {
  const ApiLoading();
}

final class ApiSuccess<T> extends ApiState<T> {
  final T data;
  const ApiSuccess(this.data);
}

final class ApiError<T> extends ApiState<T> {
  final String message;
  final int? statusCode;
  const ApiError(this.message, {this.statusCode});
}

final class ApiEmpty<T> extends ApiState<T> {
  const ApiEmpty();
}
```

```dart
// user_notifier.dart (Riverpod AsyncNotifier 예시)

class UserNotifier extends AutoDisposeAsyncNotifier<User?> {
  @override
  Future<User?> build() async => null;

  Future<ApiState<User>> fetchUser(int id) async {
    try {
      final response = await ref.read(dioProvider).get('/users/$id');
      if (response.data == null) return const ApiEmpty();
      return ApiSuccess(User.fromJson(response.data as Map<String, dynamic>));
    } on DioException catch (e) {
      return ApiError(
        e.message ?? '알 수 없는 오류가 발생했습니다',
        statusCode: e.response?.statusCode,
      );
    }
  }
}
```

```dart
// user_screen.dart

class UserScreen extends ConsumerWidget {
  final int userId;
  const UserScreen({super.key, required this.userId});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final notifier = ref.watch(userNotifierProvider.notifier);
    return FutureBuilder<ApiState<User>>(
      future: notifier.fetchUser(userId),
      builder: (context, snapshot) {
        if (!snapshot.hasData) return const CircularProgressIndicator();

        // sealed class의 소진성 검사:
        // ApiLoading, ApiSuccess, ApiError, ApiEmpty 중 하나라도 빠지면 컴파일 오류
        return switch (snapshot.data!) {
          ApiLoading() => const Center(
              child: CircularProgressIndicator(),
            ),
          ApiSuccess(:final data) => UserCard(user: data),
          ApiError(:final message, :final statusCode) => Column(
              children: [
                Icon(Icons.error, color: Colors.red),
                Text('오류 ${statusCode ?? ''}: $message'),
                ElevatedButton(
                  onPressed: () {},
                  child: const Text('다시 시도'),
                ),
              ],
            ),
          ApiEmpty() => const Center(
              child: Text('사용자 정보가 없습니다'),
            ),
        };
      },
    );
  }
}
```

`ApiState`에 새로운 하위 타입(예: `ApiRefreshing`)을 추가하면 `switch` 표현식에서 즉시 컴파일 오류가 발생합니다. 처리 누락이 불가능합니다.

---

## 코드 예제 2: Records를 활용한 폼 검증 결과 반환

여러 필드를 검증하고 결과를 Records로 반환하는 실전 예제입니다.

```dart
// form_validators.dart

typedef ValidationResult = ({bool isValid, String? errorMessage});

ValidationResult validateEmail(String email) {
  if (email.trim().isEmpty) {
    return (isValid: false, errorMessage: '이메일을 입력해주세요.');
  }
  final regex = RegExp(r'^[\w\-\.]+@([\w\-]+\.)+[\w\-]{2,4}$');
  if (!regex.hasMatch(email.trim())) {
    return (isValid: false, errorMessage: '올바른 이메일 형식이 아닙니다.');
  }
  return (isValid: true, errorMessage: null);
}

ValidationResult validatePassword(String password) {
  if (password.length < 8) {
    return (isValid: false, errorMessage: '비밀번호는 8자 이상이어야 합니다.');
  }
  if (!password.contains(RegExp(r'[A-Z]'))) {
    return (isValid: false, errorMessage: '대문자를 하나 이상 포함해야 합니다.');
  }
  if (!password.contains(RegExp(r'[0-9]'))) {
    return (isValid: false, errorMessage: '숫자를 하나 이상 포함해야 합니다.');
  }
  return (isValid: true, errorMessage: null);
}

ValidationResult validateConfirmPassword(String password, String confirm) {
  if (confirm.isEmpty) {
    return (isValid: false, errorMessage: '비밀번호 확인을 입력해주세요.');
  }
  if (password != confirm) {
    return (isValid: false, errorMessage: '비밀번호가 일치하지 않습니다.');
  }
  return (isValid: true, errorMessage: null);
}
```

```dart
// signup_form_state.dart

class SignUpFormState {
  final String email;
  final String password;
  final String confirmPassword;

  const SignUpFormState({
    this.email = '',
    this.password = '',
    this.confirmPassword = '',
  });

  // 단일 Records로 전체 검증 결과 반환
  ({bool canSubmit, Map<String, String> fieldErrors}) validate() {
    final (:isValid as emailValid, errorMessage: emailError) =
        validateEmail(email);
    final (:isValid as passValid, errorMessage: passError) =
        validatePassword(password);
    final (:isValid as confirmValid, errorMessage: confirmError) =
        validateConfirmPassword(password, confirmPassword);

    final fieldErrors = <String, String>{
      if (!emailValid && emailError != null) 'email': emailError,
      if (!passValid && passError != null) 'password': passError,
      if (!confirmValid && confirmError != null) 'confirmPassword': confirmError,
    };

    return (
      canSubmit: emailValid && passValid && confirmValid,
      fieldErrors: fieldErrors,
    );
  }

  SignUpFormState copyWith({
    String? email,
    String? password,
    String? confirmPassword,
  }) => SignUpFormState(
        email: email ?? this.email,
        password: password ?? this.password,
        confirmPassword: confirmPassword ?? this.confirmPassword,
      );
}
```

```dart
// signup_screen.dart (사용 예시)

void onSignUpPressed(SignUpFormState formState) {
  final (:canSubmit, :fieldErrors) = formState.validate();

  if (!canSubmit) {
    // 각 필드의 오류 메시지를 직접 참조
    if (fieldErrors.containsKey('email')) {
      showFieldError('email', fieldErrors['email']!);
    }
    if (fieldErrors.containsKey('password')) {
      showFieldError('password', fieldErrors['password']!);
    }
    return;
  }

  submitSignUp(formState.email, formState.password);
}
```

---

## 고급 패턴: 객체 패턴과 클래스 구조 분해

클래스의 getter를 패턴으로 직접 분해할 수 있습니다:

```dart
class Point {
  final double x, y;
  const Point(this.x, this.y);
}

String describePoint(Point p) => switch (p) {
  Point(x: 0, y: 0) => '원점',
  Point(x: var x, y: 0) => 'x축 위 ($x, 0)',
  Point(x: 0, y: var y) => 'y축 위 (0, $y)',
  Point(:var x, :var y) when x == y => '대각선 위 ($x, $y)',
  Point(:var x, :var y) => '일반 점 ($x, $y)',
};
```

중첩 패턴도 지원합니다:

```dart
sealed class Expr {}
final class Num extends Expr { final double value; Num(this.value); }
final class Add extends Expr { final Expr left, right; Add(this.left, this.right); }
final class Mul extends Expr { final Expr left, right; Mul(this.left, this.right); }

double evaluate(Expr expr) => switch (expr) {
  Num(:var value) => value,
  Add(:var left, :var right) => evaluate(left) + evaluate(right),
  Mul(:var left, :var right) => evaluate(left) * evaluate(right),
};
```

이 패턴은 컴파일러 설계, AST 처리, 게임 로직 등 복잡한 도메인 모델에 유용합니다.

---

## 주의사항 및 팁

### 1. Dart SDK 버전 명시

Records, Patterns, Sealed Classes는 **Dart 3.0 이상**에서만 사용 가능합니다. `pubspec.yaml`에 최소 버전을 명시하세요:

```yaml
environment:
  sdk: '>=3.0.0 <4.0.0'
```

Flutter 3.10 이상을 사용하면 Dart 3.0이 자동으로 포함됩니다.

### 2. sealed는 같은 파일 내에서만 확장 가능

`sealed class`의 모든 하위 타입은 반드시 **같은 `.dart` 파일** 안에 있어야 합니다. 다른 파일에서 하위 타입을 추가하려 하면 컴파일 오류가 발생합니다. 상태 계층을 단일 파일에 응집시키는 이점도 있습니다.

### 3. Records는 const를 지원하지 않음

Records 인스턴스는 `const`로 선언할 수 없습니다. 불변 상수가 필요하다면 `@immutable` 클래스를 사용하세요. 단, Records는 값 동등성을 기본 지원하므로 `const` 없이도 비교가 올바르게 동작합니다.

### 4. 소진성 검사는 sealed, enum, bool에만 적용

`switch`의 소진성 검사는 `sealed class`, `enum`, `bool` 타입에만 적용됩니다. 일반 클래스나 `int`, `String` 타입의 `switch`에서는 `_` (와일드카드) 또는 `default` 케이스를 반드시 추가해야 컴파일러 경고를 피할 수 있습니다.

### 5. OR 패턴으로 케이스 통합

여러 케이스를 동일하게 처리할 때 `|` (OR 패턴)으로 통합할 수 있습니다:

```dart
Widget buildIcon(ApiState state) => switch (state) {
  ApiLoading() || ApiEmpty() => const CircularProgressIndicator(),
  ApiSuccess() => const Icon(Icons.check),
  ApiError() => const Icon(Icons.error),
};
```

### 6. 마이그레이션 전략

기존 Dart 2 코드를 마이그레이션할 때는 **UI 레이어부터 점진적으로 적용**하는 것을 권장합니다. 상태 클래스를 `sealed`로 변경하면 컴파일러가 처리하지 않은 케이스를 즉시 알려주므로, 수정이 필요한 지점을 빠르게 파악할 수 있습니다.

---

## 결론

Dart 3의 Records, Pattern Matching, Sealed Classes는 함께 사용할 때 시너지가 극대화됩니다. 상태를 `sealed class`로 모델링하고, `switch` 표현식으로 안전하게 분기하며, 반환값을 `Record`로 간결하게 표현하면 **컴파일 타임에 안전성이 보장되는 견고한 Flutter 앱**을 만들 수 있습니다.

특히 팀 협업에서 새로운 상태를 추가할 때 처리 누락이 컴파일 오류로 즉시 드러나므로, 코드 리뷰 없이도 일관성을 유지할 수 있다는 것이 가장 큰 실용적 이점입니다.

---

## 참고 자료

- [Dart Records Feature Specification (dart-lang/language)](https://github.com/dart-lang/language/blob/main/accepted/3.0/records/feature-specification.md)
- [Dart Patterns Feature Specification (dart-lang/language)](https://github.com/dart-lang/language/blob/main/accepted/3.0/patterns/feature-specification.md)
- [Dart Class Modifiers Feature Specification (dart-lang/language)](https://github.com/dart-lang/language/blob/main/accepted/3.0/class-modifiers/feature-specification.md)
