---
layout: post
title: "Android Credential Manager & Passkey 심화: FIDO2 기반 패스워드리스 인증 완전 정복"
date: 2026-07-10
categories: [android, security]
tags: [android, credential-manager, passkey, fido2, webauthn, kotlin, authentication, security]
---

## 패스워드의 한계와 새로운 인증 패러다임

패스워드 기반 인증은 수십 년 동안 디지털 세계의 표준이었지만, 오늘날 끊임없는 보안 취약점에 노출되어 있습니다. 피싱, 자격 증명 스터핑(Credential Stuffing), 데이터 침해, 패스워드 재사용 문제 등은 사용자와 서비스 제공자 모두에게 심각한 위협입니다. Verizon의 데이터 침해 보고서에 따르면 전체 보안 침해의 86% 이상이 도난된 자격 증명과 관련되어 있습니다.

FIDO Alliance와 W3C가 공동으로 개발한 **Passkey(패스키)**는 이 문제를 근본적으로 해결하는 차세대 인증 방식입니다. 패스키는 공개키 암호화(Public-Key Cryptography)를 기반으로 하며, 사용자 기기에 비밀키를 안전하게 저장하고 서버에는 공개키만 보관합니다. 이 구조 덕분에 서버가 해킹되어도 사용자의 인증 정보는 유출되지 않습니다.

Android 생태계에서 패스키를 구현하는 공식 방법은 **Credential Manager API**입니다. `androidx.credentials` 라이브러리를 통해 제공되며, 패스워드·패스키·연합 인증(Federated Identity)을 하나의 통합된 UI와 API로 관리합니다.

---

## Passkey의 작동 원리: FIDO2와 WebAuthn

패스키는 **FIDO2 표준**을 기반으로 합니다. FIDO2는 두 가지 핵심 사양으로 구성됩니다.

- **WebAuthn (Web Authentication)**: W3C 표준으로, 브라우저·서버 간 인증 프로토콜을 정의합니다.
- **CTAP (Client to Authenticator Protocol)**: 클라이언트(앱)와 인증기(기기, 보안 키) 간의 통신을 정의합니다.

패스키 인증 흐름은 **등록(Registration)**과 **인증(Authentication)** 두 단계로 나뉩니다.

**등록 단계:**
1. 서버가 사용자에게 `challenge`(무작위 바이트)를 전송합니다.
2. 사용자 기기에서 비공개키(Private Key)와 공개키(Public Key) 쌍을 생성합니다.
3. 비공개키는 기기의 보안 엔클레이브(Android Keystore)에 안전하게 저장됩니다.
4. 공개키와 함께 `challenge`에 서명한 값을 서버에 전송합니다.
5. 서버는 공개키를 저장하고 서명을 검증하여 등록을 완료합니다.

**인증 단계:**
1. 서버가 새로운 `challenge`를 전송합니다.
2. 사용자가 생체 인식(지문·얼굴 인식) 또는 PIN으로 로컬 인증을 수행합니다.
3. 기기가 비공개키로 `challenge`에 서명합니다.
4. 서버는 저장된 공개키로 서명을 검증합니다.

이 과정에서 **비공개키는 절대 기기를 벗어나지 않습니다.** 서버에는 공개키만 보관하므로 서버 해킹으로 인한 자격 증명 유출이 원천적으로 불가능합니다. 또한 패스키는 특정 도메인(RP ID)에 종속되기 때문에 피싱 사이트에서는 사용 자체가 불가능합니다.

---

## 왜 지금 Credential Manager로 전환해야 하는가

Android에서 FIDO2 기반 인증은 기존에 `com.google.android.gms:play-services-fido` 라이브러리의 **Fido2ApiClient**를 통해 구현했습니다. Google은 2024년부터 이 API를 deprecated로 표시하고 **Credential Manager**로의 마이그레이션을 공식 권장하고 있습니다.

Credential Manager로 전환해야 하는 주요 이유:

| 항목 | 기존 Fido2ApiClient | Credential Manager |
|---|---|---|
| 지원 인증 방식 | 패스키만 지원 | 패스워드·패스키·Google Sign-In 통합 |
| UI | 별도 액티비티 런처 | 통합 바텀 시트 UI |
| Android 버전 | API 24+ | 패스워드는 API 19+, 패스키는 API 28+ |
| 3rd Party 지원 | 미지원 | Android 14+에서 1Password·Bitwarden 등 지원 |
| 유지보수 상태 | Deprecated | 활발히 개발 중인 공식 API |

현재 최신 안정 버전은 `1.6.0`(2026년 5월)이며 의존성 추가는 다음과 같습니다:

```kotlin
// build.gradle.kts
dependencies {
    implementation("androidx.credentials:credentials:1.6.0")
    // Google Play Services를 통한 구버전 Android 지원
    implementation("androidx.credentials:credentials-play-services-auth:1.6.0")
}
```

---

## 실제 구현: Credential Manager로 Passkey 등록하기

### 프로젝트 설정

`AndroidManifest.xml`에 Digital Asset Links 연결을 위한 메타데이터를 추가합니다. 패스키 등록 시 `rpId`(서버 도메인)와 앱 패키지를 연결하는 중요한 설정입니다.

```xml
<manifest>
    <application>
        <meta-data
            android:name="asset_statements"
            android:resource="@string/asset_statements" />
    </application>
</manifest>
```

`res/values/strings.xml`에 서버 도메인을 등록합니다:

```xml
<string name="asset_statements" translatable="false">
    [{
        "include": "https://example.com/.well-known/assetlinks.json"
    }]
</string>
```

서버의 `/.well-known/assetlinks.json`에는 앱의 패키지명과 SHA-256 서명 해시를 등록해야 합니다:

```json
[{
  "relation": [
    "delegate_permission/common.handle_all_urls",
    "delegate_permission/common.get_login_creds"
  ],
  "target": {
    "namespace": "android_app",
    "package_name": "com.example.myapp",
    "sha256_cert_fingerprints": ["AA:BB:CC:DD:EE:FF:..."]
  }
}]
```

### Passkey 등록 구현

서버로부터 받은 WebAuthn `PublicKeyCredentialCreationOptions` JSON을 이용해 패스키를 등록합니다.

```kotlin
import androidx.credentials.CreatePublicKeyCredentialRequest
import androidx.credentials.CreatePublicKeyCredentialResponse
import androidx.credentials.CredentialManager
import androidx.credentials.exceptions.CreateCredentialCancellationException
import androidx.credentials.exceptions.CreateCredentialException
import androidx.credentials.exceptions.publickeycredential.CreatePublicKeyCredentialDomException

class PasskeyManager(private val context: Context) {

    private val credentialManager = CredentialManager.create(context)

    /**
     * 서버로부터 받은 JSON을 사용해 패스키를 등록합니다.
     *
     * @param activity UI 컨텍스트가 필요한 Activity
     * @param requestJson 서버에서 발급한 WebAuthn PublicKeyCredentialCreationOptions JSON
     * @return 서버로 전송할 등록 응답 JSON (성공 시) 또는 오류
     */
    suspend fun registerPasskey(
        activity: Activity,
        requestJson: String
    ): Result<String> {
        val request = CreatePublicKeyCredentialRequest(
            requestJson = requestJson,
            // false: USB 보안 키 등 외부 인증기도 허용
            preferImmediatelyAvailableCredentials = false
        )

        return try {
            val result = credentialManager.createCredential(
                context = activity,
                request = request
            )
            // 성공 시 서버로 전송할 응답 JSON 반환
            val response = result as CreatePublicKeyCredentialResponse
            Result.success(response.registrationResponseJson)

        } catch (e: CreatePublicKeyCredentialDomException) {
            // WebAuthn DOM 오류: 잘못된 RP ID, 서버 challenge 불일치 등
            Result.failure(RuntimeException("등록 DOM 오류: ${e.domError}", e))

        } catch (e: CreateCredentialCancellationException) {
            // 사용자가 직접 등록 취소
            Result.failure(RuntimeException("사용자가 등록을 취소했습니다."))

        } catch (e: CreateCredentialException) {
            // 잠금 화면 미설정, 기기 미지원, Google Play Services 오류 등
            Result.failure(RuntimeException("패스키 등록 불가: ${e.message}", e))
        }
    }
}
```

서버가 내려주는 `requestJson`의 표준 형식입니다:

```json
{
  "challenge": "base64url_encoded_random_bytes",
  "rp": { "name": "My App", "id": "example.com" },
  "user": {
    "id": "base64url_user_id",
    "name": "user@example.com",
    "displayName": "홍길동"
  },
  "pubKeyCredParams": [
    { "alg": -7, "type": "public-key" },
    { "alg": -257, "type": "public-key" }
  ],
  "timeout": 1800000,
  "attestation": "none",
  "authenticatorSelection": {
    "authenticatorAttachment": "platform",
    "requireResidentKey": true,
    "userVerification": "required"
  }
}
```

---

## 실제 구현: Passkey + 패스워드 통합 로그인

패스키와 기존 패스워드 모두를 지원하는 통합 로그인을 구현합니다. 사용자는 하나의 바텀 시트에서 원하는 방식을 선택할 수 있습니다.

```kotlin
import androidx.credentials.CredentialManager
import androidx.credentials.GetCredentialRequest
import androidx.credentials.GetPasswordOption
import androidx.credentials.GetPublicKeyCredentialOption
import androidx.credentials.PasswordCredential
import androidx.credentials.PublicKeyCredential
import androidx.credentials.exceptions.GetCredentialCancellationException
import androidx.credentials.exceptions.NoCredentialException

sealed class SignInResult {
    data class Passkey(val responseJson: String) : SignInResult()
    data class Password(val username: String, val password: String) : SignInResult()
    data object NoCredential : SignInResult()
    data object Cancelled : SignInResult()
    data class Error(val message: String) : SignInResult()
}

class PasskeyManager(private val context: Context) {

    private val credentialManager = CredentialManager.create(context)

    /**
     * 패스키 또는 패스워드로 로그인합니다.
     *
     * @param activity UI 컨텍스트가 필요한 Activity
     * @param authRequestJson 서버에서 발급한 WebAuthn PublicKeyCredentialRequestOptions JSON
     */
    suspend fun signIn(activity: Activity, authRequestJson: String): SignInResult {
        val getPublicKeyCredentialOption = GetPublicKeyCredentialOption(
            requestJson = authRequestJson
        )
        // 기존 패스워드 사용자를 위한 폴백 옵션
        val getPasswordOption = GetPasswordOption()

        val request = GetCredentialRequest(
            credentialOptions = listOf(getPublicKeyCredentialOption, getPasswordOption)
        )

        return try {
            val result = credentialManager.getCredential(
                context = activity,
                request = request
            )

            when (val credential = result.credential) {
                is PublicKeyCredential -> {
                    // 패스키 인증 성공 — authenticationResponseJson을 서버로 전송
                    SignInResult.Passkey(credential.authenticationResponseJson)
                }
                is PasswordCredential -> {
                    // 패스워드 인증 성공
                    SignInResult.Password(
                        username = credential.id,
                        password = credential.password
                    )
                }
                else -> SignInResult.Error("지원하지 않는 자격증명 유형")
            }

        } catch (e: NoCredentialException) {
            // 기기에 저장된 자격증명 없음 → 회원가입 유도
            SignInResult.NoCredential
        } catch (e: GetCredentialCancellationException) {
            SignInResult.Cancelled
        } catch (e: Exception) {
            SignInResult.Error(e.message ?: "알 수 없는 오류")
        }
    }
}
```

### ViewModel과 연동

```kotlin
@HiltViewModel
class AuthViewModel @Inject constructor(
    private val passkeyManager: PasskeyManager,
    private val authRepository: AuthRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow<AuthUiState>(AuthUiState.Idle)
    val uiState: StateFlow<AuthUiState> = _uiState.asStateFlow()

    fun signIn(activity: Activity) {
        viewModelScope.launch {
            _uiState.value = AuthUiState.Loading

            // 1. 서버에서 challenge 획득
            val challengeJson = authRepository.getAuthChallenge()
                .getOrElse {
                    _uiState.value = AuthUiState.Error("서버 연결 실패")
                    return@launch
                }

            // 2. Credential Manager로 로그인
            when (val result = passkeyManager.signIn(activity, challengeJson)) {
                is SignInResult.Passkey -> {
                    authRepository.verifyPasskeyAssertion(result.responseJson)
                        .onSuccess { _uiState.value = AuthUiState.Success }
                        .onFailure { _uiState.value = AuthUiState.Error("패스키 인증 실패") }
                }
                is SignInResult.Password -> {
                    authRepository.signInWithPassword(result.username, result.password)
                        .onSuccess { _uiState.value = AuthUiState.Success }
                        .onFailure { _uiState.value = AuthUiState.Error("비밀번호 오류") }
                }
                SignInResult.NoCredential -> {
                    _uiState.value = AuthUiState.NeedSignUp
                }
                SignInResult.Cancelled -> {
                    _uiState.value = AuthUiState.Idle
                }
                is SignInResult.Error -> {
                    _uiState.value = AuthUiState.Error(result.message)
                }
            }
        }
    }

    fun registerPasskey(activity: Activity) {
        viewModelScope.launch {
            _uiState.value = AuthUiState.Loading

            val registrationOptions = authRepository.getRegistrationOptions()
                .getOrElse {
                    _uiState.value = AuthUiState.Error("등록 옵션 획득 실패")
                    return@launch
                }

            passkeyManager.registerPasskey(activity, registrationOptions)
                .onSuccess { responseJson ->
                    authRepository.verifyRegistration(responseJson)
                        .onSuccess { _uiState.value = AuthUiState.PasskeyRegistered }
                        .onFailure { _uiState.value = AuthUiState.Error("서버 검증 실패") }
                }
                .onFailure { e ->
                    _uiState.value = AuthUiState.Error(e.message ?: "등록 실패")
                }
        }
    }
}

sealed class AuthUiState {
    data object Idle : AuthUiState()
    data object Loading : AuthUiState()
    data object Success : AuthUiState()
    data object NeedSignUp : AuthUiState()
    data object PasskeyRegistered : AuthUiState()
    data class Error(val message: String) : AuthUiState()
}
```

---

## 주의사항 및 실전 팁

### 1. RP ID는 반드시 실제 도메인이어야 합니다

패스키의 `rpId`는 서버의 실제 도메인(예: `example.com`)이어야 하며, IP 주소나 `localhost`는 프로덕션에서 사용 불가합니다. 개발 환경에서는 ngrok이나 실제 스테이징 도메인을 활용하세요.

### 2. 잠금 화면 설정이 필수입니다

패스키는 사용자 검증(User Verification)을 요구합니다. 기기에 PIN·패턴·지문 등의 잠금 화면이 설정되어 있지 않으면 `CreateCredentialException`이 발생합니다. 이 경우 사용자에게 잠금 화면 설정을 안내하는 UI를 제공해야 합니다.

```kotlin
catch (e: CreateCredentialException) {
    if (e.type == CreateCredentialException.TYPE_NO_CREATE_OPTIONS) {
        // 잠금 화면 미설정 또는 플랫폼 미지원
        showLockScreenSetupGuide()
    }
}
```

### 3. `preferImmediatelyAvailableCredentials` 옵션 선택

`true`로 설정하면 기기에서 즉시 사용 가능한 플랫폼 인증기(지문·얼굴 인식)만 표시합니다. `false`로 설정하면 USB 보안 키 등 외부 인증기도 허용합니다. 일반 소비자 앱에서는 `false`가 더 유연한 경험을 제공합니다.

### 4. 패스키·패스워드 공존 마이그레이션 전략

모든 사용자가 즉시 패스키로 전환하기는 어렵습니다. 점진적 마이그레이션을 권장합니다:

- 기존 패스워드 사용자는 `GetPasswordOption`으로 계속 로그인 허용
- 패스워드 로그인 성공 후 패스키 등록을 권유하는 UI 표시 (인앱 프로모션)
- 패스키 등록 성공 후 서버에서 패스워드 비활성화 (선택)

### 5. Google Password Manager와의 자동 동기화

Android의 Credential Manager는 기본적으로 **Google Password Manager**와 연동됩니다. 한 기기에서 등록한 패스키는 동일 Google 계정의 모든 Android 기기에서 자동으로 사용 가능합니다. iOS의 iCloud Keychain, Windows Hello와도 크로스 플랫폼 동기화가 지원됩니다.

### 6. `credentials-play-services-auth`의 필요성

이 라이브러리는 Google Play Services가 설치된 Android 9~13 기기에서 패스키를 지원하기 위한 호환성 레이어입니다. Android 14부터는 OS 수준에서 패스키를 지원하지만, 구버전 호환을 위해 항상 함께 추가하는 것을 권장합니다.

---

## 마치며

Credential Manager API와 패스키는 단순한 새로운 API가 아니라 **인증 패러다임의 전환**입니다. 피싱 공격에 면역이며 사용자 경험이 뛰어난 패스키는 빠르게 업계 표준이 되고 있습니다. Apple·Google·Microsoft가 모두 FIDO2 표준을 지지하는 상황에서, Android 앱에서 Credential Manager를 통한 패스키 지원은 선택이 아닌 필수가 되어가고 있습니다.

기존 FIDO2 API를 사용 중이라면 공식 마이그레이션 가이드를 참고하여 Credential Manager로 전환하고, 새 프로젝트라면 처음부터 Credential Manager를 사용하는 것을 강력히 권장합니다.

## 참고 자료

- [About Credential Manager — Android Developers](https://developer.android.com/identity/credential-manager)
- [Migrate from FIDO2 to Credential Manager — Android Developers](https://developer.android.com/identity/sign-in/fido2-migration)
- [CredentialManager API Reference — AndroidX](https://developer.android.com/reference/androidx/credentials/CredentialManager)
- [androidx.credentials 릴리즈 노트 — Jetpack](https://developer.android.com/jetpack/androidx/releases/credentials)
