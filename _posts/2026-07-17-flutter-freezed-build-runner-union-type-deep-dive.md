---
layout: post
title: "Flutter Freezed 심화: build_runner로 불변 데이터 모델·Union 타입·JSON 직렬화 완전 정복"
date: 2026-07-17
categories: [flutter]
tags: [flutter, dart, freezed, build_runner, union-type, json-serialization, immutable, code-generation]
---

Flutter 앱을 개발하다 보면 필연적으로 마주치는 문제가 있습니다. 데이터 모델 클래스를 작성할 때마다 반복되는 `==`, `hashCode`, `copyWith`, `toString` 구현의 지루한 반복, 그리고 API 응답 상태를 안전하게 표현하기 위한 복잡한 sealed class 작성입니다. Freezed는 이 모든 문제를 단 하나의 어노테이션으로 해결합니다. 이번 포스트에서는 Freezed 3.x와 build_runner를 활용해 프로덕션 수준의 코드를 작성하는 방법을 깊이 있게 살펴봅니다.

## 1. Freezed란 무엇인가?

Freezed는 Dart/Flutter 생태계에서 가장 널리 사용되는 코드 생성 패키지입니다. `@freezed` 어노테이션 하나만으로 다음을 자동 생성합니다.

- **불변(immutable) 데이터 클래스**: 모든 필드가 `final`인 클래스
- **`copyWith` 메서드**: 특정 필드만 변경한 새 인스턴스 반환
- **값 기반 `==` / `hashCode`**: 참조가 아닌 내용으로 동등성 비교
- **`toString` 오버라이드**: 디버깅에 유용한 문자열 표현
- **Union 타입 (Sealed Class)**: 타입 안전한 다중 상태 표현
- **JSON 직렬화**: `json_serializable`과 통합된 `fromJson` / `toJson`

Dart 3에서 `sealed class` 키워드가 도입되었지만, Freezed는 이를 훨씬 더 강력하게 확장합니다. 특히 Generic Union 타입, 자동 JSON 직렬화, `copyWith`의 deeply nested 지원 등은 Dart 네이티브 sealed class로는 구현하기 까다롭습니다.

## 2. 왜 필요한가? — Dart 데이터 클래스의 한계

Dart는 Kotlin의 `data class`나 Java의 Lombok에 해당하는 내장 기능이 없습니다. 순수하게 손으로 불변 데이터 클래스를 작성하면 어떤 일이 벌어지는지 살펴봅시다.

```dart
// ❌ Freezed 없이 작성하는 불변 데이터 클래스
class User {
  final String id;
  final String name;
  final int age;
  final String? email;
  final List<String> roles;

  const User({
    required this.id,
    required this.name,
    required this.age,
    this.email,
    this.roles = const [],
  });

  // 수동 copyWith — 필드가 늘어날수록 폭발적으로 증가
  User copyWith({
    String? id,
    String? name,
    int? age,
    String? email,
    List<String>? roles,
  }) {
    return User(
      id: id ?? this.id,
      name: name ?? this.name,
      age: age ?? this.age,
      email: email ?? this.email,
      roles: roles ?? this.roles,
    );
  }

  // 수동 == 연산자 — 필드 누락 시 버그 발생
  @override
  bool operator ==(Object other) =>
      identical(this, other) ||
      other is User &&
          runtimeType == other.runtimeType &&
          id == other.id &&
          name == other.name &&
          age == other.age &&
          email == other.email &&
          listEquals(roles, other.roles);

  @override
  int get hashCode =>
      id.hashCode ^ name.hashCode ^ age.hashCode ^ email.hashCode ^ roles.hashCode;

  @override
  String toString() =>
      'User(id: $id, name: $name, age: $age, email: $email, roles: $roles)';

  // JSON 직렬화도 수동으로...
  Map<String, dynamic> toJson() => {
        'id': id,
        'name': name,
        'age': age,
        'email': email,
        'roles': roles,
      };

  factory User.fromJson(Map<String, dynamic> json) => User(
        id: json['id'] as String,
        name: json['name'] as String,
        age: json['age'] as int,
        email: json['email'] as String?,
        roles: (json['roles'] as List?)?.cast<String>() ?? [],
      );
}
```

필드 5개짜리 클래스에 이미 50줄 이상의 코드가 필요합니다. 필드가 늘어날수록, 중첩 객체가 추가될수록 유지보수 비용은 기하급수적으로 증가합니다. 또한 `==` 연산자에서 필드 하나를 빠뜨리는 실수는 테스트에서도 잡기 어려운 버그로 이어집니다.

## 3. 설치 및 프로젝트 설정

### 3.1 pubspec.yaml 설정

```yaml
dependencies:
  flutter:
    sdk: flutter
  freezed_annotation: ^2.4.1

dev_dependencies:
  build_runner: ^2.4.8
  freezed: ^2.5.3
  json_serializable: ^6.7.1  # JSON 직렬화가 필요한 경우에만
```

`freezed_annotation`은 `dependencies`에, `freezed`와 `build_runner`는 `dev_dependencies`에 넣어야 합니다. 런타임에는 어노테이션만 필요하고 코드 생성 도구는 개발 시에만 필요하기 때문입니다.

### 3.2 코드 생성 명령

```bash
# 일회성 빌드 (CI/CD 환경)
dart run build_runner build --delete-conflicting-outputs

# 파일 변경 감시 모드 (개발 중 권장)
dart run build_runner watch --delete-conflicting-outputs
```

`--delete-conflicting-outputs` 플래그는 이전에 생성된 파일과 충돌 시 자동으로 삭제 후 재생성합니다. 없으면 충돌 에러로 빌드가 멈추므로 항상 붙여두는 것을 권장합니다.

## 4. 실제 구현 예제

### 4.1 예제 1: 불변 데이터 모델 + copyWith + JSON 직렬화

프로덕션 앱에서 흔히 볼 수 있는 User 프로필 모델을 Freezed로 구현합니다.

```dart
// lib/models/user.dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'user.freezed.dart'; // Freezed가 생성
part 'user.g.dart';       // json_serializable이 생성

@freezed
class User with _$User {
  const factory User({
    required String id,
    required String name,
    required int age,
    String? email,
    @Default([]) List<String> roles,
    @JsonKey(name: 'created_at') required DateTime createdAt,
    @JsonKey(name: 'is_active', defaultValue: true) required bool isActive,
  }) = _User;

  factory User.fromJson(Map<String, dynamic> json) => _$UserFromJson(json);
}
```

위 선언만으로 build_runner가 수백 줄의 코드를 자동 생성합니다. 실제 사용 방법을 봅시다.

```dart
// 생성된 코드 활용 예시
void demonstrateFreezed() {
  final user = User(
    id: 'user-001',
    name: '김철수',
    age: 28,
    email: 'chulsoo@example.com',
    roles: ['admin', 'editor'],
    createdAt: DateTime(2024, 1, 15),
    isActive: true,
  );

  // 1. copyWith: 불변성을 유지하면서 특정 필드만 변경
  final updatedUser = user.copyWith(
    age: 29,
    email: 'chulsoo.new@example.com',
    roles: [...user.roles, 'moderator'],
  );

  print(user == updatedUser); // false (다른 값)
  print(user.id == updatedUser.id); // true (변경하지 않은 필드)

  // 2. null을 명시적으로 설정하려면 copyWith.call 사용
  // 단순 copyWith(email: null)은 "변경하지 않음"으로 해석됩니다
  // Freezed는 이를 위해 특수 문법을 제공합니다

  // 3. JSON 직렬화
  final json = user.toJson();
  // 결과: {id: user-001, name: 김철수, age: 28, email: chulsoo@example.com,
  //        roles: [admin, editor], created_at: 2024-01-15T00:00:00.000, is_active: true}

  // 4. JSON 역직렬화
  final userFromJson = User.fromJson(json);
  print(user == userFromJson); // true — 값 기반 비교

  // 5. toString
  print(user);
  // User(id: user-001, name: 김철수, age: 28, email: chulsoo@example.com,
  //      roles: [admin, editor], createdAt: 2024-01-15 00:00:00.000, isActive: true)
}
```

`@JsonKey(name: 'created_at')`처럼 JSON 키 이름을 매핑하거나, `@Default([])`로 기본값을 지정하는 것도 매우 직관적입니다.

### 4.2 예제 2: Union 타입으로 API 상태 관리

Freezed의 真髓는 Union 타입입니다. API 응답을 상태별로 타입 안전하게 표현하고, Dart 3 Pattern Matching과 결합하면 exhaustive 체크까지 가능합니다.

```dart
// lib/models/api_result.dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'api_result.freezed.dart';

/// API 요청의 세 가지 상태를 타입 안전하게 표현
@freezed
sealed class ApiResult<T> with _$ApiResult<T> {
  const factory ApiResult.loading() = Loading<T>;

  const factory ApiResult.success({
    required T data,
    @Default(false) bool isFromCache,
    DateTime? cachedAt,
  }) = Success<T>;

  const factory ApiResult.failure({
    required String message,
    int? statusCode,
    Object? exception,
  }) = Failure<T>;
}

// lib/providers/user_provider.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:freezed_annotation/freezed_annotation.dart';

part 'user_provider.freezed.dart';

@freezed
class UserScreenState with _$UserScreenState {
  const factory UserScreenState({
    @Default(ApiResult.loading()) ApiResult<User> userResult,
    @Default(false) bool isEditing,
    String? pendingName,
  }) = _UserScreenState;
}

class UserScreenNotifier extends Notifier<UserScreenState> {
  @override
  UserScreenState build() => const UserScreenState();

  Future<void> fetchUser(String userId) async {
    // 로딩 상태로 전환
    state = state.copyWith(userResult: const ApiResult.loading());

    try {
      final user = await ref.read(userRepositoryProvider).getUser(userId);
      state = state.copyWith(
        userResult: ApiResult.success(data: user),
      );
    } on NetworkException catch (e) {
      state = state.copyWith(
        userResult: ApiResult.failure(
          message: '네트워크 오류: ${e.message}',
          statusCode: e.statusCode,
          exception: e,
        ),
      );
    } catch (e) {
      state = state.copyWith(
        userResult: ApiResult.failure(
          message: '예기치 않은 오류가 발생했습니다.',
          exception: e,
        ),
      );
    }
  }

  void startEditing() {
    state = state.copyWith(isEditing: true);
  }

  void cancelEditing() {
    state = state.copyWith(isEditing: false, pendingName: null);
  }
}

// lib/widgets/user_profile_widget.dart
class UserProfileWidget extends ConsumerWidget {
  const UserProfileWidget({super.key, required this.userId});
  final String userId;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final screenState = ref.watch(userScreenNotifierProvider);
    final notifier = ref.read(userScreenNotifierProvider.notifier);

    return switch (screenState.userResult) {
      // Dart 3 Destructuring Pattern Matching
      Loading() => const Center(child: CircularProgressIndicator()),

      Success(:final data, :final isFromCache, :final cachedAt) => Column(
          children: [
            if (isFromCache)
              CachedDataBanner(cachedAt: cachedAt),
            if (screenState.isEditing)
              UserEditForm(
                user: data,
                onCancel: notifier.cancelEditing,
              )
            else
              UserCard(
                user: data,
                onEdit: notifier.startEditing,
              ),
          ],
        ),

      Failure(:final message, :final statusCode) => ErrorView(
          message: message,
          statusCode: statusCode,
          onRetry: () => notifier.fetchUser(userId),
        ),
    };
  }
}
```

`switch` expression이 `ApiResult`의 모든 케이스를 처리하지 않으면 컴파일 타임에 에러가 발생합니다. 이것이 바로 Union 타입의 핵심 가치입니다. 런타임에 처리되지 않은 상태로 인한 크래시가 원천적으로 차단됩니다.

### 4.3 커스텀 메서드와 Generic Union 타입

실제 앱에서 자주 필요한 고급 패턴들입니다.

```dart
// lib/models/product.dart

@freezed
class Product with _$Product {
  // private 기본 생성자 추가 → 커스텀 멤버 정의 가능
  const Product._();

  const factory Product({
    required String id,
    required String name,
    required double price,
    required double discountRate,
    required ProductStatus status,
  }) = _Product;

  factory Product.fromJson(Map<String, dynamic> json) => _$ProductFromJson(json);

  // 커스텀 getter — const Product._() 없이는 에러
  double get finalPrice => price * (1 - discountRate);
  bool get isOnSale => discountRate > 0;
  bool get isAvailable => status == ProductStatus.available;
}

// Generic pagination wrapper
@Freezed(genericArgumentFactories: true)
class PagedResponse<T> with _$PagedResponse<T> {
  const factory PagedResponse({
    required List<T> items,
    required int totalCount,
    required int page,
    required int perPage,
    @Default(false) bool hasMore,
  }) = _PagedResponse<T>;

  factory PagedResponse.fromJson(
    Map<String, dynamic> json,
    T Function(Object? json) fromJsonT,
  ) => _$PagedResponseFromJson(json, fromJsonT);
}

// 실제 사용
Future<PagedResponse<Product>> fetchProducts(int page) async {
  final json = await apiClient.get('/products?page=$page');
  return PagedResponse<Product>.fromJson(
    json,
    (item) => Product.fromJson(item as Map<String, dynamic>),
  );
}
```

## 5. 주의사항과 실전 팁

### 5.1 커스텀 멤버를 추가하려면 private 생성자가 필수

```dart
@freezed
class Counter with _$Counter {
  const Counter._(); // ← 이 줄이 없으면 아래 getter에서 컴파일 에러

  const factory Counter({@Default(0) int count}) = _Counter;

  bool get isZero => count == 0;
  bool get isPositive => count > 0;
  Counter increment() => copyWith(count: count + 1);
}
```

### 5.2 null 값을 명시적으로 설정하는 방법

단순히 `copyWith(email: null)`은 "변경하지 않음"으로 해석됩니다. 명시적으로 null을 설정하려면 `@freezed` 클래스에 `@JsonSerializable` 설정과 함께 별도 접근법이 필요합니다.

```dart
// Freezed의 copyWith은 null을 "변경 없음"으로 해석
// null로 설정하고 싶다면 래퍼를 활용
final cleared = user.copyWith(email: const Value(null).value);
// 또는 직접 팩토리 생성자로 새 인스턴스 만들기
final cleared = User(
  id: user.id,
  name: user.name,
  age: user.age,
  email: null, // 명시적 null
  roles: user.roles,
  createdAt: user.createdAt,
  isActive: user.isActive,
);
```

### 5.3 `part` 파일 충돌 방지

여러 Freezed 클래스를 같은 파일에 정의하면 하나의 `.freezed.dart`가 생성됩니다. 하지만 JSON 직렬화가 필요한 클래스와 그렇지 않은 클래스를 같은 파일에 섞으면 관리가 복잡해집니다. 원칙적으로 **파일 하나에 Freezed 클래스 하나**를 권장합니다.

### 5.4 Dart 3 Pattern Matching 권장, `when()`/`map()` 지양

```dart
// ✅ Dart 3 스타일 — 권장
final message = switch (result) {
  Loading() => '로딩 중...',
  Success(:final data) => '성공: ${data.name}',
  Failure(:final message) => '오류: $message',
};

// ⚠️ Freezed 레거시 스타일 — 지양 (하지만 여전히 동작함)
final message = result.when(
  loading: () => '로딩 중...',
  success: (data, isFromCache, cachedAt) => '성공: ${data.name}',
  failure: (message, statusCode, exception) => '오류: $message',
);
```

Dart 3 `switch` expression은 컴파일러 레벨에서 exhaustive 체크를 수행하며, Destructuring으로 필요한 필드만 추출할 수 있어 더 간결합니다.

### 5.5 build_runner 성능 최적화

대규모 프로젝트에서 build_runner 실행 시간이 길어질 때 `build.yaml`로 최적화합니다.

```yaml
# build.yaml (프로젝트 루트)
targets:
  $default:
    builders:
      freezed:
        options:
          # Union 타입에서 generate_all: false로 미사용 메서드 생성 억제
          # (Freezed 버전에 따라 옵션명이 다를 수 있음)
          copy_with: true
          equal: true
          to_string: true
```

또한 `flutter pub run build_runner watch` 대신 `dart run build_runner watch`를 사용하면 약 20% 더 빠릅니다. `dart` 명령이 Flutter wrapper 오버헤드를 거치지 않기 때문입니다.

### 5.6 .gitignore 설정

생성된 파일은 일반적으로 `.gitignore`에 추가하지 않는 것을 권장합니다. CI 환경에서 코드 생성 단계를 별도로 실행하거나, 생성 파일을 커밋해 PR 리뷰에서 생성된 코드를 확인할 수 있어야 하기 때문입니다.

하지만 팀 규모와 CI 파이프라인에 따라 생성 파일을 `.gitignore`에 추가하고 CI 빌드 단계에 `build_runner build`를 포함시키는 방식도 유효합니다.

## 정리

Freezed와 build_runner는 Flutter 앱 개발에서 보일러플레이트 코드를 극적으로 줄이고, 타입 안전성을 높이는 핵심 도구입니다.

| 기능 | Dart 수동 구현 | Freezed |
|------|--------------|---------|
| 불변 데이터 클래스 | 수십 줄 보일러플레이트 | 5~10줄 선언 |
| copyWith | 수동 작성, 필드 누락 위험 | 자동 생성, 안전 |
| 값 기반 동등성 | 수동 `==` / `hashCode` | 자동 생성 |
| Union 타입 | sealed class + 수동 match | 어노테이션 하나 |
| JSON 직렬화 | 수동 fromJson/toJson | json_serializable 통합 |
| Dart 3 Pattern Matching | 부분 지원 | 완전 지원 |

특히 Riverpod의 `Notifier`/`AsyncNotifier`나 BLoC 패턴과 Freezed를 결합하면 상태 관리 레이어가 매우 견고하고 읽기 쉬워집니다. 새 Flutter 프로젝트를 시작할 때 가장 먼저 도입을 고려해야 할 라이브러리 중 하나입니다.

## 참고 자료

- [freezed \| pub.dev 공식 문서](https://pub.dev/packages/freezed)
- [json\_serializable \| pub.dev 공식 문서](https://pub.dev/packages/json_serializable)
- [build\_runner \| pub.dev 공식 문서](https://pub.dev/packages/build_runner)
