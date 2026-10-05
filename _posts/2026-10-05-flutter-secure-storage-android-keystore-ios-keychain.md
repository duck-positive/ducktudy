---
layout: post
title: "Flutter Secure Storage 심화: flutter_secure_storage로 민감 데이터를 Android Keystore와 iOS Keychain에 안전하게 저장하는 완전 가이드"
date: 2026-10-05
categories: [android, flutter]
tags: [flutter, security, flutter_secure_storage, android_keystore, ios_keychain, dart, biometric]
---

모바일 앱에서 사용자 토큰, 비밀번호, API 키와 같은 민감한 데이터를 저장하는 것은 피할 수 없는 요구사항입니다. `SharedPreferences`나 일반 파일 시스템에 이런 데이터를 저장하면 루팅(Rooting)된 기기나 백업 분석을 통해 데이터가 노출될 수 있습니다. `flutter_secure_storage` 패키지는 이 문제를 해결하기 위해 각 플랫폼의 보안 저장소를 활용합니다.

## 1. 개념 설명: flutter_secure_storage란?

`flutter_secure_storage`(v11.2.0)는 Flutter 앱에서 민감한 데이터를 키-값 쌍으로 안전하게 저장할 수 있게 해주는 공식적으로 검증된 패키지입니다. 각 플랫폼의 네이티브 보안 저장소를 추상화합니다.

| 플랫폼 | 사용 저장소 | 암호화 방식 |
|---|---|---|
| Android | EncryptedSharedPreferences | AES-256-GCM + Android Keystore |
| iOS / macOS | Keychain Services | AES-256 (하드웨어 기반) |
| Windows | DPAPI | 시스템 자격증명 |
| Linux | libsecret | GNOME Keyring / KWallet |

핵심은 암호화 키(마스터 키)를 앱의 메모리나 파일 시스템이 아니라 **하드웨어 보안 모듈(TEE, Trusted Execution Environment)** 또는 **Secure Element**에 안전하게 보관하고, 실제 암호화/복호화 연산도 해당 보안 영역 안에서만 수행된다는 점입니다. 앱 프로세스는 키 자체를 꺼낼 수 없으므로 공격 표면이 현저히 줄어듭니다.

### Android Keystore의 동작 원리

Android Keystore는 단순한 키 저장소가 아닙니다. 앱은 키 핸들(alias)만 보유하고, 실제 키 소재(key material)는 TEE나 Secure Element 안에 갇혀 있습니다. `EncryptedSharedPreferences`가 데이터를 저장할 때 다음 과정을 거칩니다:

1. Keystore에서 마스터 AES-256-GCM 키 참조를 가져옴
2. 해당 키 핸들로 암호화 연산을 시스템 프로세스(TEE)에 위임
3. 암호문만 `SharedPreferences`에 저장
4. 읽을 때 역순으로 TEE에서 복호화 수행

루팅된 기기에서 `/data/data/<패키지명>/` 디렉토리에 접근해도 암호문만 볼 수 있고, 키는 꺼낼 수 없습니다.

## 2. 왜 필요한가? — 일반 저장소의 위험성

### SharedPreferences / NSUserDefaults의 취약점

```
# 루팅된 Android 기기에서 SharedPreferences 파일 직접 읽기
adb shell
su
cat /data/data/com.example.app/shared_prefs/prefs.xml
# → <string name="access_token">eyJhbGci...</string> 그대로 노출
```

- **Android**: `data/data/<패키지명>/shared_prefs/*.xml`에 평문 저장. 루팅된 기기에서 바로 읽을 수 있음
- **iOS**: `.plist` 파일로 저장되며 iTunes 백업(암호화되지 않은 경우)에 포함
- 앱 내 악성 서드파티 라이브러리가 동일 프로세스에서 접근 가능

### 실제 공격 시나리오

1. **루팅된 Android 기기**: ADB나 파일 관리자로 SharedPreferences XML 직접 읽기
2. **iTunes 백업 분석**: 암호화되지 않은 백업에서 plist 파일 추출 후 토큰 탈취
3. **메모리 덤프**: 프로세스 메모리 덤프로 런타임에 존재하는 키값 탈취
4. **APK 리버스 엔지니어링**: 하드코딩된 시크릿 키 탈취

`flutter_secure_storage`를 사용하면 위의 공격들이 모두 무력화됩니다. 실제 키 소재가 TEE 밖으로 나오지 않기 때문입니다.

## 3. 실제 구현 예제

### 설치 및 기본 설정

`pubspec.yaml`에 의존성을 추가합니다:

```yaml
dependencies:
  flutter_secure_storage: ^11.2.0
```

Android에서는 `minSdkVersion`이 반드시 23 이상이어야 합니다:

```gradle
// android/app/build.gradle
android {
    defaultConfig {
        minSdkVersion 23
    }
}
```

### 예제 1: SecureStorageService 추상화 레이어

전역에서 `FlutterSecureStorage` 인스턴스를 직접 사용하는 것보다, 서비스 클래스로 추상화하면 테스트 가능성과 유지보수성이 크게 향상됩니다. 특히 `Future.wait`으로 병렬 I/O를 처리하는 것이 핵심입니다.

```dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

class SecureStorageService {
  static const _storage = FlutterSecureStorage(
    aOptions: AndroidOptions(
      encryptedSharedPreferences: true, // Android 6.0+ (API 23+) 필수
    ),
    iOptions: IOSOptions(
      // 첫 번째 잠금 해제 후 접근 가능 → 백그라운드 작업(push 갱신 등) 대응
      accessibility: KeychainAccessibility.first_unlock,
      synchronizable: false, // iCloud 동기화 비활성화 (민감 데이터 전용)
    ),
  );

  static const _accessTokenKey = 'access_token';
  static const _refreshTokenKey = 'refresh_token';
  static const _userIdKey = 'user_id';

  /// 로그인 성공 후 토큰 저장 (병렬 I/O)
  static Future<void> saveTokens({
    required String accessToken,
    required String refreshToken,
  }) async {
    await Future.wait([
      _storage.write(key: _accessTokenKey, value: accessToken),
      _storage.write(key: _refreshTokenKey, value: refreshToken),
    ]);
  }

  /// 저장된 토큰 읽기 (병렬 I/O, Dart 3 Records 반환)
  static Future<(String?, String?)> readTokens() async {
    final results = await Future.wait([
      _storage.read(key: _accessTokenKey),
      _storage.read(key: _refreshTokenKey),
    ]);
    return (results[0], results[1]);
  }

  /// 로그아웃 시 모든 보안 데이터 삭제
  static Future<void> clearAll() async {
    await _storage.deleteAll();
  }

  /// 유효한 세션 존재 여부 확인
  static Future<bool> hasValidSession() async {
    final accessToken = await _storage.read(key: _accessTokenKey);
    return accessToken != null && accessToken.isNotEmpty;
  }

  /// userId 저장
  static Future<void> saveUserId(String userId) async {
    await _storage.write(key: _userIdKey, value: userId);
  }
}
```

사용 예시:

```dart
// 로그인 처리
await SecureStorageService.saveTokens(
  accessToken: response.accessToken,
  refreshToken: response.refreshToken,
);

// 앱 시작 시 세션 확인
final hasSession = await SecureStorageService.hasValidSession();
if (!hasSession) {
  // 로그인 화면으로 이동
}

// 토큰 읽기 (Dart 3 구조 분해)
final (accessToken, refreshToken) = await SecureStorageService.readTokens();
```

### 예제 2: Riverpod + 생체 인증 연동 보안 저장소

실제 프로덕션 앱에서 Riverpod과 함께 생체 인증을 강제하는 패턴입니다. Android에서 지문/얼굴 인증 없이는 저장된 데이터에 접근 불가능하게 할 수 있습니다.

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:local_auth/local_auth.dart';

// DI: 테스트 시 override 가능
final secureStorageProvider = Provider<FlutterSecureStorage>((ref) {
  return const FlutterSecureStorage(
    aOptions: AndroidOptions(
      encryptedSharedPreferences: true,
      storageCipherAlgorithm: StorageCipherAlgorithm.AES_GCM_NoPadding,
    ),
    iOptions: IOSOptions(
      // 기기에서만 접근 가능 + 첫 번째 잠금 해제 후
      accessibility: KeychainAccessibility.first_unlock_this_device,
      synchronizable: false,
    ),
  );
});

final authNotifierProvider =
    AsyncNotifierProvider<AuthNotifier, AuthState>(AuthNotifier.new);

class AuthNotifier extends AsyncNotifier<AuthState> {
  late final FlutterSecureStorage _storage;
  final _localAuth = LocalAuthentication();

  @override
  Future<AuthState> build() async {
    _storage = ref.read(secureStorageProvider);
    final token = await _storage.read(key: 'access_token');
    return token != null
        ? AuthState.authenticated(token: token)
        : const AuthState.unauthenticated();
  }

  Future<bool> _authenticate() async {
    final canBio = await _localAuth.canCheckBiometrics;
    final canAuth = await _localAuth.isDeviceSupported();
    if (!canBio && !canAuth) return true; // 생체 인증 불가 기기는 통과

    return _localAuth.authenticate(
      localizedReason: '인증이 필요합니다',
      options: const AuthenticationOptions(
        biometricOnly: false, // PIN/패턴도 허용
        stickyAuth: true,     // 앱이 백그라운드 갔다 와도 재인증 불필요
      ),
    );
  }

  Future<void> login({
    required String accessToken,
    required String refreshToken,
  }) async {
    state = const AsyncValue.loading();
    try {
      await Future.wait([
        _storage.write(key: 'access_token', value: accessToken),
        _storage.write(key: 'refresh_token', value: refreshToken),
      ]);
      state = AsyncValue.data(AuthState.authenticated(token: accessToken));
    } catch (e, st) {
      state = AsyncValue.error(e, st);
    }
  }

  /// 생체 인증 후 민감 작업 수행 (예: 결제 정보 읽기)
  Future<String?> readWithBiometric(String key) async {
    final authenticated = await _authenticate();
    if (!authenticated) throw Exception('인증 실패');
    return _storage.read(key: key);
  }

  Future<void> logout() async {
    await _storage.deleteAll();
    state = const AsyncValue.data(AuthState.unauthenticated());
  }
}

// 불변 상태 모델 (Freezed 사용 권장)
sealed class AuthState {
  const AuthState();
  const factory AuthState.authenticated({required String token}) =
      _Authenticated;
  const factory AuthState.unauthenticated() = _Unauthenticated;
}

class _Authenticated extends AuthState {
  final String token;
  const _Authenticated({required this.token});
}

class _Unauthenticated extends AuthState {
  const _Unauthenticated();
}
```

Widget에서의 사용:

```dart
class HomeScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final authState = ref.watch(authNotifierProvider);

    return authState.when(
      data: (state) => switch (state) {
        _Authenticated(:final token) => MainContent(token: token),
        _Unauthenticated() => const LoginScreen(),
      },
      loading: () => const CircularProgressIndicator(),
      error: (e, _) => ErrorView(message: e.toString()),
    );
  }
}
```

## 4. 주의사항 및 팁

### 4.1 백업에서 보안 데이터 제외 (Android 필수)

`AndroidManifest.xml`에서 자동 백업을 비활성화하거나 secure storage 경로를 제외해야 합니다:

```xml
<!-- AndroidManifest.xml -->
<application
    android:allowBackup="false"
    android:fullBackupContent="false"
    ...>
```

Android 12 이상에서는 `dataExtractionRules.xml`을 사용합니다:

```xml
<!-- res/xml/data_extraction_rules.xml -->
<?xml version="1.0" encoding="utf-8"?>
<data-extraction-rules>
    <cloud-backup>
        <exclude domain="sharedpref" path="FlutterSecureStorage"/>
    </cloud-backup>
    <device-transfer>
        <exclude domain="sharedpref" path="FlutterSecureStorage"/>
    </device-transfer>
</data-extraction-rules>
```

`AndroidManifest.xml`에서 이 파일을 참조합니다:

```xml
<application
    android:dataExtractionRules="@xml/data_extraction_rules"
    ...>
```

### 4.2 iOS Keychain Accessibility 선택 가이드

| Accessibility 레벨 | 언제 사용? |
|---|---|
| `first_unlock` | 앱 재시작 후 백그라운드 작업에서도 접근 필요 시 (기본 권장) |
| `first_unlock_this_device` | iCloud 동기화 없이 기기 전용, 백그라운드 접근 필요 시 |
| `when_unlocked` | 보안 최우선, 백그라운드 접근 불필요 시 |
| `when_unlocked_this_device` | 가장 엄격한 보안, 동기화 없음 |
| `always_this_device` | 잠금 상태에서도 접근 필요(BackgroundFetch 등) — 보안 위험 |

대부분의 앱에서는 `first_unlock` 또는 `first_unlock_this_device`가 적절합니다.

### 4.3 테스트 환경에서의 Mock 처리

Flutter 단위 테스트에서는 플랫폼 채널이 없으므로 인메모리 구현을 사용합니다:

```dart
// test/helpers/fake_secure_storage.dart
import 'package:flutter_secure_storage_platform_interface/flutter_secure_storage_platform_interface.dart';

class FakeFlutterSecureStoragePlatform extends FlutterSecureStoragePlatform {
  final Map<String, String> _storage = {};

  @override
  Future<void> write({
    required String key,
    required String value,
    required Map<String, String> options,
  }) async => _storage[key] = value;

  @override
  Future<String?> read({
    required String key,
    required Map<String, String> options,
  }) async => _storage[key];

  @override
  Future<void> delete({
    required String key,
    required Map<String, String> options,
  }) async => _storage.remove(key);

  @override
  Future<void> deleteAll({required Map<String, String> options}) async =>
      _storage.clear();

  @override
  Future<bool> containsKey({
    required String key,
    required Map<String, String> options,
  }) async => _storage.containsKey(key);

  @override
  Future<Map<String, String>> readAll({
    required Map<String, String> options,
  }) async => Map.unmodifiable(_storage);
}

// test/auth_test.dart에서 setUp
setUp(() {
  FlutterSecureStoragePlatform.instance = FakeFlutterSecureStoragePlatform();
});
```

### 4.4 v9 → v11 마이그레이션 주의사항

버전 10부터 Android는 RSA+AES 방식에서 `EncryptedSharedPreferences`로 완전 전환됩니다. **기존 사용자가 업데이트하면 이전 방식으로 저장된 데이터를 읽을 수 없게 됩니다.** v10에서는 자동 마이그레이션 도구가 제공되었으나 v11부터는 제거되었으므로, 반드시 v10을 거쳐 마이그레이션 후 v11로 업그레이드해야 합니다.

업그레이드 순서: `v9.x → v10.x(마이그레이션) → v11.x`

### 4.5 성능 최적화

- `_storage.read()` / `_storage.write()`는 네이티브 I/O 호출이므로 반드시 `await`로 처리
- 앱 시작 시 꼭 필요한 키만 읽고, 나머지는 필요 시 lazy하게 읽기
- 동일 키를 반복적으로 읽는 경우 메모리 캐시 레이어 추가 고려
- `readAll()`은 전체 항목을 읽으므로 최소화 (앱 이관 등 특수 케이스에만 사용)
- Android에서 `EncryptedSharedPreferences`는 첫 초기화 시 Keystore 키 생성으로 약간의 지연이 있음 — `main()` 초기에 미리 초기화 권장

```dart
// main.dart에서 미리 초기화
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  // 첫 접근 시 Keystore 키 생성을 main() 단계에서 처리
  await SecureStorageService.hasValidSession();
  runApp(const MyApp());
}
```

### 4.6 키 존재 여부 확인 vs null 체크

`containsKey()`와 `read()` 후 null 체크는 동작이 같지만, 값의 존재만 확인할 때는 `containsKey()`가 더 명시적입니다:

```dart
// 덜 명확
final token = await _storage.read(key: 'token');
if (token != null) { ... }

// 더 명확 (키 존재 여부만 확인할 때)
final hasToken = await _storage.containsKey(key: 'token');
if (hasToken) { ... }
```

## 마무리

`flutter_secure_storage`는 단순한 암호화 래퍼가 아니라, 각 플랫폼의 하드웨어 보안 기능을 Flutter 앱에서 손쉽게 활용할 수 있게 해주는 필수 보안 레이어입니다. 사용자 토큰, 비밀번호, 생체 인증 결과, API 시크릿 등 앱에서 다루는 모든 민감한 데이터는 반드시 이 레이어를 통해 저장해야 합니다.

핵심 정리:
- **Android**: `encryptedSharedPreferences: true` + `minSdkVersion 23` 설정 필수
- **iOS**: 사용 사례에 맞는 `KeychainAccessibility` 선택
- **백업 제외** 설정으로 데이터 노출 방지
- **서비스 레이어 추상화** + **Riverpod DI**로 테스트 가능성 확보
- **버전 업그레이드 시** 마이그레이션 경로 반드시 확인

## 참고 자료
- [flutter_secure_storage - pub.dev](https://pub.dev/packages/flutter_secure_storage)
- [Android Keystore System - Android Developers](https://developer.android.com/privacy-and-security/keystore)
- [Keychain Services - Apple Developer](https://developer.apple.com/documentation/security/keychain_services)
