---
layout: post
title: "Android BiometricPrompt 심화: CryptoObject·KeyStore 연동부터 Class 3 강력 인증 구현까지"
date: 2026-07-31
categories: [android]
tags: [android, biometric, biometricprompt, cryptoobject, keystore, security, kotlin]
---

현대 모바일 앱에서 생체 인증은 선택이 아닌 필수입니다. 단순히 지문 버튼을 띄우는 것을 넘어, **CryptoObject**와 **Android Keystore**를 연동한 암호학적으로 안전한 인증 체계를 구현해야 비로소 "강력한 인증"이라 부를 수 있습니다. 이 글에서는 `BiometricPrompt` API의 내부 구조를 파헤치고, Class 3 생체 인증을 활용한 실전 보안 패턴을 완전히 마스터합니다.

## 생체 인증 클래스 분류: Class 1 / 2 / 3

Android는 Android 11(API 30)부터 생체 인증 센서를 세 가지 보안 등급으로 분류합니다.

| 클래스 | 이름 | 설명 | 대표 예시 |
|--------|------|------|-----------|
| Class 1 | Convenience | 가장 낮은 보안. 앱이 직접 사용 불가 | 일부 저가형 얼굴 인식 |
| Class 2 | Weak | 중간 보안. CryptoObject 연동 **불가** | 소프트웨어 얼굴 인식 |
| Class 3 | Strong | 최고 보안. CryptoObject 연동 **가능** | 지문 센서, 하드웨어 얼굴 인식 |

Class 3 (`BIOMETRIC_STRONG`)만이 Keystore와 암호학적으로 결합할 수 있습니다. 금융 거래, 결제, 민감 데이터 암·복호화에는 반드시 Class 3을 사용해야 합니다.

## 왜 CryptoObject 연동이 필요한가

단순한 `BiometricPrompt`는 **"사용자가 인증에 성공했다"**는 사실만 앱에 알려줍니다. 이 방식의 약점은 다음과 같습니다.

- 루팅된 기기에서 인증 성공 신호를 위조할 수 있습니다.
- 앱 서버 측에서 "이 인증이 진짜인가"를 검증할 방법이 없습니다.
- 인증 성공과 민감 작업 수행 사이의 간극을 악용할 수 있습니다.

`CryptoObject`는 이 문제를 해결합니다. 생체 인증이 성공해야만 Keystore의 키를 사용할 수 있게 잠금을 겁니다. 인증 없이는 **암호화 연산 자체가 불가능**하므로 위조가 원천 차단됩니다.

```
[생체 인증 성공]
        ↓
[Android Keystore 키 잠금 해제]
        ↓
[Cipher / Signature / Mac 사용 가능]
        ↓
[데이터 암호화 또는 서명 수행]
```

이 흐름이 바로 "암호학적으로 결합된 인증(Cryptographically-bound authentication)"입니다.

## 실전 구현 예제 1: 기본 BiometricPrompt 설정

먼저 의존성을 추가합니다.

```kotlin
// build.gradle.kts
dependencies {
    implementation("androidx.biometric:biometric:1.1.0")
}
```

가용성 확인부터 시작합니다. 생체 인증이 지원되지 않는 기기에서 오류가 발생하지 않도록 사전에 점검해야 합니다.

```kotlin
import androidx.biometric.BiometricManager
import androidx.biometric.BiometricManager.Authenticators.BIOMETRIC_STRONG
import androidx.biometric.BiometricManager.Authenticators.DEVICE_CREDENTIAL

class AuthCheckHelper(private val context: Context) {

    enum class AuthStatus {
        AVAILABLE,
        NOT_ENROLLED,
        NOT_SUPPORTED,
        UNKNOWN_ERROR
    }

    fun checkBiometricAvailability(): AuthStatus {
        val manager = BiometricManager.from(context)
        return when (manager.canAuthenticate(BIOMETRIC_STRONG or DEVICE_CREDENTIAL)) {
            BiometricManager.BIOMETRIC_SUCCESS -> AuthStatus.AVAILABLE
            BiometricManager.BIOMETRIC_ERROR_NONE_ENROLLED -> AuthStatus.NOT_ENROLLED
            BiometricManager.BIOMETRIC_ERROR_NO_HARDWARE,
            BiometricManager.BIOMETRIC_ERROR_HW_UNAVAILABLE -> AuthStatus.NOT_SUPPORTED
            else -> AuthStatus.UNKNOWN_ERROR
        }
    }
}
```

`BIOMETRIC_ERROR_NONE_ENROLLED`가 반환되면 사용자를 생체 인증 등록 화면으로 안내해야 합니다. Android 11 이상에서는 `Settings.ACTION_BIOMETRIC_ENROLL` Intent를 사용합니다.

```kotlin
fun navigateToEnrollment(activity: Activity) {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
        val intent = Intent(Settings.ACTION_BIOMETRIC_ENROLL).apply {
            putExtra(
                Settings.EXTRA_BIOMETRIC_AUTHENTICATORS_ALLOWED,
                BIOMETRIC_STRONG or DEVICE_CREDENTIAL
            )
        }
        activity.startActivity(intent)
    } else {
        activity.startActivity(Intent(Settings.ACTION_SECURITY_SETTINGS))
    }
}
```

## 실전 구현 예제 2: CryptoObject + KeyStore 강력 인증

이것이 핵심입니다. AES-GCM 대칭키를 Android Keystore에 생성하고, 생체 인증으로만 잠금을 해제하는 패턴을 구현합니다.

```kotlin
import android.security.keystore.KeyGenParameterSpec
import android.security.keystore.KeyProperties
import androidx.biometric.BiometricPrompt
import androidx.core.content.ContextCompat
import androidx.fragment.app.FragmentActivity
import java.security.KeyStore
import javax.crypto.Cipher
import javax.crypto.KeyGenerator
import javax.crypto.SecretKey
import javax.crypto.spec.GCMParameterSpec

class BiometricCryptoManager(private val activity: FragmentActivity) {

    companion object {
        private const val KEYSTORE_PROVIDER = "AndroidKeyStore"
        private const val KEY_ALIAS = "ducktudy_biometric_key"
        private const val AES_KEY_SIZE = 256
        private const val GCM_TAG_LENGTH = 128
        private const val TRANSFORMATION = "AES/GCM/NoPadding"
    }

    // --- KeyStore에 생체 인증 전용 키 생성 ---
    fun generateSecretKey() {
        val keyStore = KeyStore.getInstance(KEYSTORE_PROVIDER)
        keyStore.load(null)

        // 이미 키가 있으면 재생성하지 않음
        if (keyStore.containsAlias(KEY_ALIAS)) return

        val keyGenerator = KeyGenerator.getInstance(
            KeyProperties.KEY_ALGORITHM_AES,
            KEYSTORE_PROVIDER
        )
        keyGenerator.init(
            KeyGenParameterSpec.Builder(
                KEY_ALIAS,
                KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT
            )
                .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
                .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
                .setKeySize(AES_KEY_SIZE)
                // 핵심: 매 사용마다 생체 인증 필요
                .setUserAuthenticationRequired(true)
                // Android 11+: 생체 인증 타입 지정 (BIOMETRIC_STRONG만 허용)
                .setUserAuthenticationParameters(
                    0, // timeout 0 = per-use key (매번 인증)
                    KeyProperties.AUTH_BIOMETRIC_STRONG
                )
                // 새 생체 인증 등록 시 키 무효화 (보안 강화)
                .setInvalidatedByBiometricEnrollment(true)
                .build()
        )
        keyGenerator.generateKey()
    }

    // --- 암호화용 Cipher 초기화 (인증 전 단계) ---
    fun getEncryptCipher(): Cipher {
        val keyStore = KeyStore.getInstance(KEYSTORE_PROVIDER)
        keyStore.load(null)
        val secretKey = keyStore.getKey(KEY_ALIAS, null) as SecretKey

        val cipher = Cipher.getInstance(TRANSFORMATION)
        cipher.init(Cipher.ENCRYPT_MODE, secretKey)
        return cipher
    }

    // --- 복호화용 Cipher 초기화 (저장된 IV 필요) ---
    fun getDecryptCipher(iv: ByteArray): Cipher {
        val keyStore = KeyStore.getInstance(KEYSTORE_PROVIDER)
        keyStore.load(null)
        val secretKey = keyStore.getKey(KEY_ALIAS, null) as SecretKey

        val cipher = Cipher.getInstance(TRANSFORMATION)
        cipher.init(Cipher.DECRYPT_MODE, secretKey, GCMParameterSpec(GCM_TAG_LENGTH, iv))
        return cipher
    }

    // --- BiometricPrompt 생성 ---
    fun createBiometricPrompt(
        onSuccess: (BiometricPrompt.AuthenticationResult) -> Unit,
        onError: (Int, CharSequence) -> Unit,
        onFailed: () -> Unit
    ): BiometricPrompt {
        val executor = ContextCompat.getMainExecutor(activity)
        return BiometricPrompt(
            activity,
            executor,
            object : BiometricPrompt.AuthenticationCallback() {
                override fun onAuthenticationSucceeded(result: BiometricPrompt.AuthenticationResult) {
                    super.onAuthenticationSucceeded(result)
                    onSuccess(result)
                }

                override fun onAuthenticationError(errorCode: Int, errString: CharSequence) {
                    super.onAuthenticationError(errorCode, errString)
                    onError(errorCode, errString)
                }

                override fun onAuthenticationFailed() {
                    super.onAuthenticationFailed()
                    // 인증 시도는 했지만 매칭 실패 (연속 실패 시 잠금)
                    onFailed()
                }
            }
        )
    }

    // --- PromptInfo 생성 ---
    fun buildPromptInfo(title: String, subtitle: String): BiometricPrompt.PromptInfo {
        return BiometricPrompt.PromptInfo.Builder()
            .setTitle(title)
            .setSubtitle(subtitle)
            .setAllowedAuthenticators(
                BiometricManager.Authenticators.BIOMETRIC_STRONG
            )
            // BIOMETRIC_STRONG 단독 사용 시 반드시 NegativeButton 설정
            .setNegativeButtonText("취소")
            .setConfirmationRequired(true) // 명시적 확인 요구 (고보안 작업)
            .build()
    }

    // --- 암호화 흐름: 데이터를 암호화하여 반환 ---
    fun encryptData(
        plainText: String,
        onSuccess: (encryptedData: ByteArray, iv: ByteArray) -> Unit,
        onError: (Int, CharSequence) -> Unit
    ) {
        generateSecretKey()
        val cipher = getEncryptCipher()
        val cryptoObject = BiometricPrompt.CryptoObject(cipher)

        val prompt = createBiometricPrompt(
            onSuccess = { result ->
                val authenticatedCipher = result.cryptoObject?.cipher
                    ?: run { onError(-1, "CryptoObject가 null입니다"); return@createBiometricPrompt }
                val encrypted = authenticatedCipher.doFinal(plainText.toByteArray(Charsets.UTF_8))
                val iv = authenticatedCipher.iv
                onSuccess(encrypted, iv)
            },
            onError = onError,
            onFailed = { /* UI 피드백 처리 */ }
        )

        val promptInfo = buildPromptInfo(
            title = "데이터 암호화",
            subtitle = "저장 전 생체 인증이 필요합니다"
        )
        prompt.authenticate(promptInfo, cryptoObject)
    }

    // --- 복호화 흐름 ---
    fun decryptData(
        encryptedData: ByteArray,
        iv: ByteArray,
        onSuccess: (String) -> Unit,
        onError: (Int, CharSequence) -> Unit
    ) {
        val cipher = getDecryptCipher(iv)
        val cryptoObject = BiometricPrompt.CryptoObject(cipher)

        val prompt = createBiometricPrompt(
            onSuccess = { result ->
                val authenticatedCipher = result.cryptoObject?.cipher
                    ?: run { onError(-1, "CryptoObject가 null입니다"); return@createBiometricPrompt }
                val decrypted = authenticatedCipher.doFinal(encryptedData)
                onSuccess(String(decrypted, Charsets.UTF_8))
            },
            onError = onError,
            onFailed = { }
        )

        val promptInfo = buildPromptInfo(
            title = "데이터 복호화",
            subtitle = "접근을 위해 생체 인증이 필요합니다"
        )
        prompt.authenticate(promptInfo, cryptoObject)
    }
}
```

### 실제 사용 예시 (Activity)

```kotlin
class SecureNoteActivity : AppCompatActivity() {

    private lateinit var cryptoManager: BiometricCryptoManager
    private var encryptedNote: ByteArray? = null
    private var noteIv: ByteArray? = null

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        cryptoManager = BiometricCryptoManager(this)
    }

    fun saveNote(noteText: String) {
        cryptoManager.encryptData(
            plainText = noteText,
            onSuccess = { encrypted, iv ->
                encryptedNote = encrypted
                noteIv = iv
                // SharedPreferences나 Room에 encrypted + iv를 저장
                saveToStorage(encrypted, iv)
                Toast.makeText(this, "메모가 안전하게 저장되었습니다", Toast.LENGTH_SHORT).show()
            },
            onError = { errorCode, message ->
                when (errorCode) {
                    BiometricPrompt.ERROR_NEGATIVE_BUTTON -> {
                        // 사용자가 취소 버튼을 누름
                    }
                    BiometricPrompt.ERROR_LOCKOUT -> {
                        Toast.makeText(this, "인증 시도 횟수 초과. 잠시 후 다시 시도하세요", Toast.LENGTH_LONG).show()
                    }
                    BiometricPrompt.ERROR_LOCKOUT_PERMANENT -> {
                        Toast.makeText(this, "PIN으로 잠금을 해제해 주세요", Toast.LENGTH_LONG).show()
                    }
                    else -> {
                        Toast.makeText(this, "오류: $message", Toast.LENGTH_SHORT).show()
                    }
                }
            }
        )
    }

    fun loadNote() {
        val (encrypted, iv) = loadFromStorage() ?: return

        cryptoManager.decryptData(
            encryptedData = encrypted,
            iv = iv,
            onSuccess = { plainText ->
                binding.noteTextView.text = plainText
            },
            onError = { _, message ->
                Toast.makeText(this, "복호화 실패: $message", Toast.LENGTH_SHORT).show()
            }
        )
    }
}
```

## KeyGenParameterSpec 핵심 파라미터 해설

| 파라미터 | 설명 | 권장값 |
|---------|------|--------|
| `setUserAuthenticationRequired(true)` | 키 사용에 인증 필수 | `true` |
| `setUserAuthenticationParameters(0, AUTH_BIOMETRIC_STRONG)` | 타임아웃 0 = per-use key | `0` (매번 인증) |
| `setInvalidatedByBiometricEnrollment(true)` | 새 지문 등록 시 키 무효화 | `true` |
| `setUserAuthenticationValidityDurationSeconds` | (구버전) 인증 유효 시간 | **사용 비권장** |

### Per-Use Key vs Time-Based Key

`setUserAuthenticationParameters(0, ...)` 처럼 타임아웃을 `0`으로 설정하면 **per-use key**가 됩니다. 키를 사용할 때마다 반드시 인증해야 합니다. 이것이 금융·결제 앱에 적합한 설정입니다.

반면 타임아웃을 양수로 설정하면 일정 시간 동안 재인증 없이 키를 사용할 수 있지만, 보안이 약화됩니다. 이 방식은 `setUserAuthenticationValidityDurationSeconds`를 통해 구버전 API와 연동됩니다만, Android 11+ 에서는 `setUserAuthenticationParameters`를 사용하는 것이 권장됩니다.

## 생체 인증 무효화 처리

`setInvalidatedByBiometricEnrollment(true)` 설정 후 사용자가 새 지문을 등록하면, 기존에 생성된 키는 즉시 무효화됩니다. 이후 Cipher 초기화 시 `KeyPermanentlyInvalidatedException`이 발생합니다.

```kotlin
fun getEncryptCipherSafe(): Cipher? {
    return try {
        getEncryptCipher()
    } catch (e: KeyPermanentlyInvalidatedException) {
        // 기존 키 삭제 후 재생성
        deleteKey()
        generateSecretKey()
        null // 사용자에게 재등록 안내
    }
}

private fun deleteKey() {
    val keyStore = KeyStore.getInstance("AndroidKeyStore")
    keyStore.load(null)
    keyStore.deleteEntry(KEY_ALIAS)
}
```

사용자에게 "새 생체 정보가 등록되어 재인증이 필요합니다" 메시지를 표시하고, 다음 사용 시 새 키로 데이터를 다시 암호화하도록 안내해야 합니다.

## ECDSA Signature를 활용한 서버 인증

서버와의 인증에는 `Cipher`(대칭키) 대신 `Signature`(비대칭키)를 사용합니다. 클라이언트에서 개인키로 챌린지에 서명하고, 서버는 공개키로 검증합니다.

```kotlin
import android.security.keystore.KeyProperties
import java.security.KeyPairGenerator
import java.security.Signature

class BiometricSignatureManager {

    companion object {
        private const val KEY_ALIAS = "ducktudy_ec_key"
        private const val KEYSTORE_PROVIDER = "AndroidKeyStore"
    }

    fun generateKeyPair() {
        val keyStore = KeyStore.getInstance(KEYSTORE_PROVIDER)
        keyStore.load(null)
        if (keyStore.containsAlias(KEY_ALIAS)) return

        val kpg = KeyPairGenerator.getInstance(
            KeyProperties.KEY_ALGORITHM_EC,
            KEYSTORE_PROVIDER
        )
        kpg.initialize(
            KeyGenParameterSpec.Builder(
                KEY_ALIAS,
                KeyProperties.PURPOSE_SIGN or KeyProperties.PURPOSE_VERIFY
            )
                .setDigests(KeyProperties.DIGEST_SHA256)
                .setUserAuthenticationRequired(true)
                .setUserAuthenticationParameters(0, KeyProperties.AUTH_BIOMETRIC_STRONG)
                .setInvalidatedByBiometricEnrollment(true)
                .build()
        )
        kpg.generateKeyPair()
    }

    // 서버에 등록할 공개키 반환
    fun getPublicKeyBase64(): String {
        val keyStore = KeyStore.getInstance(KEYSTORE_PROVIDER)
        keyStore.load(null)
        val publicKey = keyStore.getCertificate(KEY_ALIAS).publicKey
        return android.util.Base64.encodeToString(publicKey.encoded, android.util.Base64.NO_WRAP)
    }

    // 서버 챌린지를 생체 인증 후 서명
    fun signChallenge(
        challenge: ByteArray,
        activity: FragmentActivity,
        onSuccess: (signatureBytes: ByteArray) -> Unit,
        onError: (Int, CharSequence) -> Unit
    ) {
        generateKeyPair()

        val keyStore = KeyStore.getInstance(KEYSTORE_PROVIDER)
        keyStore.load(null)
        val privateKey = keyStore.getKey(KEY_ALIAS, null) as java.security.PrivateKey

        val signature = Signature.getInstance("SHA256withECDSA")
        signature.initSign(privateKey)
        signature.update(challenge) // 서명할 챌린지 데이터 입력
        val cryptoObject = BiometricPrompt.CryptoObject(signature)

        val executor = ContextCompat.getMainExecutor(activity)
        val prompt = BiometricPrompt(
            activity,
            executor,
            object : BiometricPrompt.AuthenticationCallback() {
                override fun onAuthenticationSucceeded(result: BiometricPrompt.AuthenticationResult) {
                    val signedBytes = result.cryptoObject?.signature?.sign()
                        ?: return
                    onSuccess(signedBytes)
                }

                override fun onAuthenticationError(errorCode: Int, errString: CharSequence) {
                    onError(errorCode, errString)
                }
            }
        )

        val promptInfo = BiometricPrompt.PromptInfo.Builder()
            .setTitle("로그인 확인")
            .setSubtitle("생체 인증으로 서버 요청에 서명합니다")
            .setAllowedAuthenticators(BiometricManager.Authenticators.BIOMETRIC_STRONG)
            .setNegativeButtonText("취소")
            .build()

        prompt.authenticate(promptInfo, cryptoObject)
    }
}
```

이 패턴은 FIDO2/WebAuthn의 기본 원리와 동일합니다. 서버는 챌린지를 발급하고, 클라이언트는 생체 인증 후 개인키로 서명하여 응답합니다. 비밀 정보는 네트워크를 통해 전송되지 않으므로 중간자 공격에 안전합니다.

## 주의사항 및 실전 팁

### 1. `onAuthenticationFailed()` vs `onAuthenticationError()` 구분

- `onAuthenticationFailed()`: 지문이 일치하지 않았을 때 호출됩니다. 여러 번 실패 후 `ERROR_LOCKOUT`으로 전환됩니다. 이 콜백에서는 **프롬프트가 아직 표시 중**이므로 별도로 닫지 않아도 됩니다.
- `onAuthenticationError()`: 취소, 잠금, 하드웨어 오류 등 최종 오류 상태입니다. 프롬프트가 자동으로 닫힙니다.

### 2. `ERROR_LOCKOUT` 처리

5회 연속 실패 시 30초 잠금, 이후에도 실패하면 영구 잠금(`ERROR_LOCKOUT_PERMANENT`)이 발생합니다. 영구 잠금은 사용자가 PIN/패턴/비밀번호로 기기를 잠금 해제해야 풀립니다. 앱에서 별도 처리가 필요합니다.

### 3. `setConfirmationRequired()` 사용 지침

- **금융 거래, 결제** → `setConfirmationRequired(true)`: 사용자가 명시적으로 "확인" 버튼을 눌러야 인증 완료
- **잠금 해제, 화면 접근** → `setConfirmationRequired(false)`: 지문 접촉 즉시 인증 완료 (UX 향상)

### 4. 키 무효화와 데이터 마이그레이션

`setInvalidatedByBiometricEnrollment(true)` 사용 시 새 지문 등록으로 키가 무효화되면 해당 키로 암호화된 모든 데이터에 접근할 수 없게 됩니다. 데이터 손실을 방지하려면:

1. 새 키 생성 전 서버에서 데이터를 복구하거나
2. 키 무효화 전에 백업 메커니즘을 제공하거나
3. `setInvalidatedByBiometricEnrollment(false)`로 설정하되 보안 위험을 인지합니다.

### 5. FragmentActivity 필수

`BiometricPrompt`는 `FragmentActivity` 또는 `ComponentActivity`를 요구합니다. `Activity`만으로는 동작하지 않습니다. Jetpack Biometric 1.4.x-alpha부터는 `ComponentActivity`도 지원합니다.

### 6. 테스트 환경에서의 생체 인증

에뮬레이터에서 생체 인증을 테스트하려면:
```bash
# 에뮬레이터에서 가상 지문 인증 트리거
adb -e emu finger touch 1
```

실기기 없이도 기본 동작을 검증할 수 있습니다.

## 전체 흐름 요약

```
앱 시작
  ├─ BiometricManager.canAuthenticate() 확인
  ├─ NOT_ENROLLED → 등록 화면 안내
  └─ AVAILABLE
       ├─ KeyStore에 키 생성 (최초 1회)
       ├─ Cipher / Signature 초기화 (인증 전)
       ├─ CryptoObject(cipher) 생성
       ├─ BiometricPrompt.authenticate(promptInfo, cryptoObject)
       │
       ├─ onAuthenticationSucceeded
       │    └─ result.cryptoObject?.cipher?.doFinal(data) → 암·복호화
       │
       ├─ onAuthenticationError(ERROR_LOCKOUT) → 30초 대기 안내
       ├─ onAuthenticationError(ERROR_LOCKOUT_PERMANENT) → PIN 해제 안내
       └─ onAuthenticationFailed → (자동 재시도, 별도 처리 불필요)
```

## 마무리

`BiometricPrompt`와 `CryptoObject`의 조합은 Android 보안 생태계에서 가장 강력한 로컬 인증 수단입니다. 단순 "로그인 확인"을 넘어, 암호학적으로 데이터를 잠그고 생체 인증 없이는 접근 자체가 불가능한 구조를 만들 수 있습니다.

핵심 원칙을 다시 정리합니다:

1. **Class 3 (`BIOMETRIC_STRONG`) + `CryptoObject`**: 금융·결제 앱의 최소 요건
2. **Per-use key** (`timeout = 0`): 가장 안전한 설정. 매번 인증
3. **`setInvalidatedByBiometricEnrollment(true)`**: 새 생체 정보 등록 시 자동 무효화로 보안 유지
4. **`KeyPermanentlyInvalidatedException` 처리**: 키 무효화를 우아하게 처리해야 데이터 손실 방지

## 참고 자료
- [Show a biometric authentication dialog - Android Developers](https://developer.android.com/identity/sign-in/biometric-auth)
- [BiometricPrompt API Reference - Android Developers](https://developer.android.com/reference/android/hardware/biometrics/BiometricPrompt)
- [Jetpack Biometric Library Releases](https://developer.android.com/jetpack/androidx/releases/biometric)
- [Secure user authentication - Android Developers](https://developer.android.com/security/fraud-prevention/authentication)
