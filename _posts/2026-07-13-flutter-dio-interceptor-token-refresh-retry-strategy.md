---
layout: post
title: "Flutter Dio 인터셉터 심화: 토큰 갱신, 재시도 전략, 네트워크 계층 설계"
date: 2026-07-13
categories: [flutter, android]
tags: [flutter, dio, interceptor, http, token-refresh, retry, network, rest-api]
---

현업 Flutter 앱에서 네트워크 레이어가 얼마나 견고한지는 곧 앱의 안정성과 직결됩니다. 사용자가 모르는 사이에 만료된 토큰이 자동으로 갱신되고, 일시적인 네트워크 오류는 조용히 재시도되어야 합니다. 이 모든 것을 깔끔하게 처리할 수 있는 것이 바로 Dio의 **인터셉터(Interceptor)** 입니다. 이 글에서는 Dio 5.x 기준으로 인터셉터의 내부 동작 원리부터, 실전에서 바로 쓸 수 있는 JWT 토큰 자동 갱신 패턴, 지수 백오프 재시도 전략까지 심층적으로 다룹니다.

## Dio 인터셉터란 무엇인가

Dio는 Flutter/Dart 생태계에서 가장 널리 쓰이는 HTTP 클라이언트 라이브러리입니다(pub.dev 최신 버전: 5.10.0). 인터셉터는 요청이 서버로 전송되기 전, 그리고 응답이 앱 코드에 도달하기 전에 끼어들어 데이터를 가로채거나 수정할 수 있는 미들웨어입니다.

`Interceptor` 클래스는 세 가지 핵심 콜백을 제공합니다.

| 메서드 | 시점 | 주요 용도 |
|--------|------|-----------|
| `onRequest` | 요청 전송 직전 | 헤더 주입, 로깅, 요청 취소 |
| `onResponse` | 응답 수신 직후 | 데이터 변환, 캐싱, 성공 로깅 |
| `onError` | 예외 발생 시 | 토큰 갱신, 재시도, 에러 변환 |

각 콜백은 `handler` 파라미터를 통해 체인의 흐름을 제어합니다. `handler.next()`로 다음 인터셉터나 실제 응답으로 진행하거나, `handler.resolve()`로 즉시 성공 응답을 반환하거나, `handler.reject()`로 강제 에러를 발생시킬 수 있습니다.

## 왜 인터셉터가 필요한가

인터셉터 없이 API 호출을 작성하면 어떻게 될까요? 모든 Repository나 DataSource 클래스마다 다음 코드가 중복됩니다.

- Authorization 헤더 주입
- 401 응답 감지 후 토큰 갱신 로직
- 네트워크 오류 시 재시도
- 요청/응답 디버그 로깅

이는 DRY(Don't Repeat Yourself) 원칙을 심각하게 위반합니다. 인터셉터를 사용하면 이 모든 횡단 관심사(cross-cutting concerns)를 한 곳에서 처리할 수 있으며, 비즈니스 로직과 완전히 분리됩니다.

또 다른 핵심 이유는 **동시 요청 문제**입니다. 앱 초기화 시점에 여러 API가 동시에 호출되다가 모두 401을 받는 상황을 상상해 보세요. 인터셉터가 없다면 토큰 갱신 요청이 N번 중복 발생합니다. 올바르게 설계된 인터셉터는 이 문제를 큐(queue) 기반으로 해결합니다.

## 실제 구현 예제

### 예제 1: JWT 토큰 자동 갱신 인터셉터 (`QueuedInterceptor` 사용)

동시 401 문제를 해결하는 핵심은 `QueuedInterceptor`입니다. 일반 `Interceptor`와 달리 `QueuedInterceptor`는 `onError` 등의 콜백을 **직렬(serial)** 로 실행하므로, 첫 번째 401 처리가 완료될 때까지 나머지 요청들이 대기합니다.

```dart
import 'package:dio/dio.dart';

class AuthInterceptor extends QueuedInterceptor {
  final Dio _dio;
  final TokenStorage _tokenStorage;

  AuthInterceptor(this._dio, this._tokenStorage);

  @override
  void onRequest(
    RequestOptions options,
    RequestInterceptorHandler handler,
  ) async {
    final accessToken = await _tokenStorage.getAccessToken();
    if (accessToken != null) {
      options.headers['Authorization'] = 'Bearer $accessToken';
    }
    handler.next(options);
  }

  @override
  void onError(DioException err, ErrorInterceptorHandler handler) async {
    // 401이 아니거나 이미 토큰 갱신 요청 자체라면 그냥 통과
    if (err.response?.statusCode != 401 ||
        err.requestOptions.path.contains('/auth/refresh')) {
      return handler.next(err);
    }

    try {
      final refreshToken = await _tokenStorage.getRefreshToken();
      if (refreshToken == null) {
        await _tokenStorage.clear();
        return handler.reject(err);
      }

      // 토큰 갱신 요청 (인터셉터 체인을 우회하기 위해 별도 Dio 인스턴스 사용)
      final refreshDio = Dio(BaseOptions(baseUrl: _dio.options.baseUrl));
      final response = await refreshDio.post('/auth/refresh', data: {
        'refresh_token': refreshToken,
      });

      final newAccessToken = response.data['access_token'] as String;
      final newRefreshToken = response.data['refresh_token'] as String;

      await _tokenStorage.saveTokens(
        accessToken: newAccessToken,
        refreshToken: newRefreshToken,
      );

      // 원래 요청의 헤더를 새 토큰으로 교체하고 재시도
      final retryOptions = err.requestOptions
        ..headers['Authorization'] = 'Bearer $newAccessToken';

      final retryResponse = await _dio.fetch(retryOptions);
      handler.resolve(retryResponse);
    } on DioException catch (e) {
      // 갱신도 실패했다면 로그아웃 처리
      await _tokenStorage.clear();
      handler.reject(e);
    }
  }
}

// 토큰 저장소 추상화 (실제 구현은 FlutterSecureStorage 등 사용)
abstract class TokenStorage {
  Future<String?> getAccessToken();
  Future<String?> getRefreshToken();
  Future<void> saveTokens({
    required String accessToken,
    required String refreshToken,
  });
  Future<void> clear();
}
```

`QueuedInterceptor`를 사용하면 동시에 여러 401이 발생하더라도 토큰 갱신은 정확히 한 번만 수행되고, 이후 대기 중인 요청들은 새로 발급된 토큰으로 자동 재시도됩니다.

---

### 예제 2: 지수 백오프(Exponential Backoff) 재시도 인터셉터

네트워크 연결 불안정이나 서버 일시 장애(5xx)는 잠시 후 재시도하면 해결되는 경우가 많습니다. 무조건 즉시 재시도하면 서버 부하를 악화시킬 수 있으므로 **지수 백오프** 전략이 필요합니다.

```dart
import 'dart:math';
import 'package:dio/dio.dart';

class RetryInterceptor extends Interceptor {
  final Dio _dio;
  final int maxRetries;
  final Duration initialDelay;

  RetryInterceptor({
    required Dio dio,
    this.maxRetries = 3,
    this.initialDelay = const Duration(milliseconds: 500),
  }) : _dio = dio;

  @override
  void onError(DioException err, ErrorInterceptorHandler handler) async {
    final attempt = _getAttemptCount(err.requestOptions);

    if (!_shouldRetry(err, attempt)) {
      return handler.next(err);
    }

    // 지수 백오프: 500ms → 1000ms → 2000ms (최대 8초 캡)
    final delay = _calculateDelay(attempt);
    await Future.delayed(delay);

    final retryOptions = err.requestOptions
      ..extra['retryCount'] = attempt + 1;

    try {
      final response = await _dio.fetch(retryOptions);
      handler.resolve(response);
    } on DioException catch (retryError) {
      handler.next(retryError);
    }
  }

  bool _shouldRetry(DioException err, int attempt) {
    if (attempt >= maxRetries) return false;

    // 재시도할 오류 유형: 연결 타임아웃, 수신 타임아웃, 서버 오류(5xx)
    final isNetworkError = err.type == DioExceptionType.connectionTimeout ||
        err.type == DioExceptionType.receiveTimeout ||
        err.type == DioExceptionType.connectionError;
    final isServerError = (err.response?.statusCode ?? 0) >= 500;

    // 클라이언트 오류(4xx)나 인증 오류(401)는 재시도하지 않음
    final isClientError = (err.response?.statusCode ?? 0) >= 400 &&
        (err.response?.statusCode ?? 0) < 500;

    return (isNetworkError || isServerError) && !isClientError;
  }

  int _getAttemptCount(RequestOptions options) {
    return options.extra['retryCount'] as int? ?? 0;
  }

  Duration _calculateDelay(int attempt) {
    final exponent = pow(2, attempt).toInt();
    final delayMs = initialDelay.inMilliseconds * exponent;
    // 최대 8초로 제한
    return Duration(milliseconds: min(delayMs, 8000));
  }
}

// Dio 설정 예시
Dio createDio({
  required String baseUrl,
  required TokenStorage tokenStorage,
}) {
  final dio = Dio(BaseOptions(
    baseUrl: baseUrl,
    connectTimeout: const Duration(seconds: 10),
    receiveTimeout: const Duration(seconds: 30),
    headers: {'Content-Type': 'application/json'},
  ));

  // 인터셉터 순서가 중요: LogInterceptor → RetryInterceptor → AuthInterceptor
  // AuthInterceptor는 QueuedInterceptor이므로 마지막에 추가
  dio.interceptors.addAll([
    LogInterceptor(
      requestBody: true,
      responseBody: true,
      logPrint: (obj) => debugPrint(obj.toString()),
    ),
    RetryInterceptor(dio: dio),
    AuthInterceptor(dio, tokenStorage),
  ]);

  return dio;
}
```

---

## 인터셉터 체인의 실행 순서

인터셉터는 등록된 순서대로 FIFO(First In, First Out) 방식으로 실행됩니다. 단, 요청(`onRequest`)과 응답(`onResponse`/`onError`)의 방향이 반대라는 점을 이해해야 합니다.

```
요청 흐름:   LogInterceptor → RetryInterceptor → AuthInterceptor → 서버
응답 흐름:   서버 → AuthInterceptor → RetryInterceptor → LogInterceptor
```

`LogInterceptor`를 첫 번째에 배치하면, 토큰 갱신 후 재시도된 요청도 모두 로깅됩니다. 반대로 마지막에 배치하면 최종 완성된 요청(Authorization 헤더 포함)만 기록됩니다. 목적에 맞게 순서를 선택하세요.

## 주의사항 및 실전 팁

### 1. 토큰 갱신 전용 Dio 인스턴스를 분리하라

`AuthInterceptor` 안에서 토큰 갱신 요청을 할 때 **반드시 인터셉터가 등록되지 않은 별도 Dio 인스턴스**를 사용해야 합니다. 같은 인스턴스를 재사용하면 갱신 요청 자체도 인터셉터를 통과하다가 무한 재귀(Infinite Loop)에 빠질 수 있습니다.

### 2. handler.next()를 빠뜨리지 마라

`onRequest`, `onResponse`, `onError` 모두 반드시 핸들러 메서드(`next`, `resolve`, `reject`) 중 하나를 호출해야 합니다. 호출하지 않으면 해당 요청이 영구히 보류 상태로 남아 앱이 응답하지 않는 것처럼 보입니다. `async` 콜백에서 예외가 발생했을 때 빠져나오는 경로(catch 블록)에서도 핸들러 호출을 보장해야 합니다.

### 3. QueuedInterceptor는 직렬 처리임을 명심하라

`QueuedInterceptor`의 콜백들은 이전 콜백이 완료될 때까지 다음 콜백이 실행되지 않습니다. 토큰 갱신처럼 직렬 처리가 필요한 경우에는 탁월하지만, 단순 로깅이나 헤더 주입처럼 빠른 작업에 `QueuedInterceptor`를 남용하면 성능 병목이 생깁니다.

### 4. 무한 재시도 방지

`RetryInterceptor`는 재시도 횟수를 `RequestOptions.extra`에 보관합니다. 이 값이 제대로 전달되지 않으면 재시도 카운터가 초기화되어 무한 루프가 발생할 수 있습니다. `_dio.fetch(retryOptions)`에서 `retryOptions`는 원본 `requestOptions`의 참조이므로 `extra` 값이 그대로 전달됩니다.

### 5. 테스트에서 MockInterceptor 활용

인터셉터를 별도 클래스로 분리하면 단위 테스트가 쉬워집니다. 테스트 환경에서는 `AuthInterceptor` 대신 `MockAuthInterceptor`를 주입하여 401 → 토큰 갱신 → 재시도 시나리오를 쉽게 검증할 수 있습니다.

```dart
// 테스트용 Interceptor
class AlwaysUnauthorizedInterceptor extends Interceptor {
  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) {
    handler.reject(
      DioException(
        requestOptions: options,
        response: Response(
          requestOptions: options,
          statusCode: 401,
        ),
        type: DioExceptionType.badResponse,
      ),
    );
  }
}
```

## 마치며

Dio 인터셉터는 단순한 편의 기능이 아니라, 프로덕션 수준의 Flutter 앱을 위한 핵심 설계 패턴입니다. `QueuedInterceptor`로 동시 401 문제를 방지하고, 지수 백오프 재시도로 일시적 네트워크 오류를 투명하게 처리하면, 비즈니스 로직 코드는 네트워크 복잡성을 전혀 신경 쓰지 않아도 됩니다. 인터셉터 체인의 실행 순서와 핸들러 호출 규칙만 확실히 이해한다면, 어떤 복잡한 인증 흐름도 우아하게 구현할 수 있습니다.

## 참고 자료
- [Dio Package on pub.dev](https://pub.dev/packages/dio)
- [Interceptor class - Dio API docs](https://pub.dev/documentation/dio/latest/dio/Interceptor-class.html)
- [QueuedInterceptor - Dio API docs](https://pub.dev/documentation/dio/latest/dio/QueuedInterceptor-class.html)
