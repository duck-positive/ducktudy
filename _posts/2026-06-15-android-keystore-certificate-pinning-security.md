---
layout: post
title: "Android 보안 심화: Android Keystore와 Certificate Pinning 완전 정복"
date: 2026-06-15
categories: [android, flutter]
tags: [android, security, keystore, certificate-pinning, okhttp, kotlin, tls, biometric]
---

## 개요

현대의 Android 앱은 사용자의 민감한 데이터를 다루는 경우가 많습니다. 금융 앱, 헬스케어 앱, 기업용 앱에서 보안은 선택이 아닌 필수입니다. 특히 두 가지 핵심 보안 기법인 **Android Keystore**와 **Certificate Pinning**은 앱의 데이터 보안과 네트워크 통신 보안을 책임지는 핵심 기술입니다.

이 글에서는 두 기술의 개념부터 실제 구현까지 심층적으로 다루며, 실전에서 놓치기 쉬운 운영 이슈까지 정리합니다.

---

## 1. Android Keystore System

### 1.1 Keystore란 무엇인가?

Android Keystore는 OS가 제공하는 하드웨어/소프트웨어 기반의 암호화 키 저장소입니다. 일반적인 SharedPreferences나 데이터베이스에 키를 저장하는 것과는 근본적으로 다릅니다.

AES 키를 앱 내에 하드코딩하거나 SharedPreferences에 저장한다면, 루팅된 기기에서 간단히 추출됩니다. 반면 Android Keystore는 다음을 보장합니다:

- **키 소재(Key Material) 비추출성**: 앱 프로세스에서 키 소재에 직접 접근 불가
- **하드웨어 기반 보안 (TEE/SE)**: Android 9(API 28)+ 에서는 StrongBox를 통해 별도 보안 요소에 키 저장 가능
- **사용 제한(Authorization)**: 허용 알고리즘, 인증 요구사항, 유효 기간 등 세밀한 제어

### 1.2 왜 Keystore가 필요한가?

```kotlin
// 위험한 방식 — 하드코딩된 키 (절대 사용 금지)
val secretKey = "my_secret_key_1234"  // 역공학으로 쉽게 추출 가능
val sharedPrefs = context.getSharedPreferences("keys", Context.MODE_PRIVATE)
sharedPrefs.edit().putString("api_key", secretKey).apply() // 루팅 기기에서 직접 읽기 가능
```

이 방식은 다음 위협에 취약합니다:
- APK 디컴파일로 하드코딩된 키 추출
- 루팅 기기에서 SharedPreferences 파일 직접 읽기
- 메모리 덤프를 통한 키 탈취

### 1.3 AES-GCM Keystore 구현

```kotlin
import android.security.keystore.KeyGenParameterSpec
import android.security.keystore.KeyProperties
import java.security.KeyStore
import javax.crypto.Cipher
import javax.crypto.KeyGenerator
import javax.crypto.SecretKey
import javax.crypto.spec.GCMParameterSpec
import android.util.Base64

class SecureDataManager {
    private val KEY_ALIAS = "my_secure_key"
    private val ANDROID_KEYSTORE = "AndroidKeyStore"
    private val TRANSFORMATION = "AES/GCM/NoPadding"
    private val GCM_TAG_LENGTH = 128

    private fun getOrCreateSecretKey(): SecretKey {
        val keyStore = KeyStore.getInstance(ANDROID_KEYSTORE).also { it.load(null) }
        keyStore.getKey(KEY_ALIAS, null)?.let { return it as SecretKey }

        val keyGenerator = KeyGenerator.getInstance(
            KeyProperties.KEY_ALGORITHM_AES,
            ANDROID_KEYSTORE
        )
        val keyGenSpec = KeyGenParameterSpec.Builder(
            KEY_ALIAS,
            KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT
        )
            .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
            .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
            .setKeySize(256)
            // 생체인증 연동 시 활성화
            // .setUserAuthenticationRequired(true)
            // .setUserAuthenticationValidityDurationSeconds(30)
            .build()

        keyGenerator.init(keyGenSpec)
        return keyGenerator.generateKey()
    }

    fun encrypt(plainText: String): String {
        val cipher = Cipher.getInstance(TRANSFORMATION)
        cipher.init(Cipher.ENCRYPT_MODE, getOrCreateSecretKey())

        val iv = cipher.iv
        val encryptedBytes = cipher.doFinal(plainText.toByteArray(Charsets.UTF_8))

        // IV(12바이트) + 암호문을 연결하여 저장
        val combined = ByteArray(iv.size + encryptedBytes.size)
        System.arraycopy(iv, 0, combined, 0, iv.size)
        System.arraycopy(encryptedBytes, 0, combined, iv.size, encryptedBytes.size)

        return Base64.encodeToString(combined, Base64.DEFAULT)
    }

    fun decrypt(encryptedData: String): String {
        val combined = Base64.decode(encryptedData, Base64.DEFAULT)
        val iv = combined.copyOfRange(0, 12)
        val encryptedBytes = combined.copyOfRange(12, combined.size)

        val cipher = Cipher.getInstance(TRANSFORMATION)
        val spec = GCMParameterSpec(GCM_TAG_LENGTH, iv)
        cipher.init(Cipher.DECRYPT_MODE, getOrCreateSecretKey(), spec)

        return String(cipher.doFinal(encryptedBytes), Charsets.UTF_8)
    }
}
```

### 1.4 StrongBox 활용 및 생체인증 연동

Android 9+에서는 StrongBox(별도 Secure Element)를 시도하고 미지원 기기는 TEE로 폴백할 수 있습니다.

```kotlin
import android.os.Build
import android.security.keystore.KeyGenParameterSpec
import android.security.keystore.KeyProperties
import android.security.keystore.StrongBoxUnavailableException
import androidx.biometric.BiometricPrompt
import androidx.core.content.ContextCompat
import androidx.fragment.app.FragmentActivity
import javax.crypto.Cipher
import javax.crypto.KeyGenerator
import javax.crypto.SecretKey

class StrongBoxKeyManager(private val activity: FragmentActivity) {

    fun createStrongBoxKey(alias: String): SecretKey {
        val keyGenerator = KeyGenerator.getInstance(
            KeyProperties.KEY_ALGORITHM_AES, "AndroidKeyStore"
        )

        val specBuilder = KeyGenParameterSpec.Builder(
            alias,
            KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT
        )
            .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
            .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
            .setKeySize(256)
            .setUserAuthenticationRequired(true)     // 생체인증 강제
            .setInvalidatedByBiometricEnrollment(true) // 새 지문 등록 시 키 무효화

        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.P) {
            try {
                specBuilder.setIsStrongBoxBacked(true)
                keyGenerator.init(specBuilder.build())
                return keyGenerator.generateKey()
            } catch (e: StrongBoxUnavailableException) {
                // 미지원 기기 → TEE 폴백
            }
        }

        keyGenerator.init(specBuilder.build())
        return keyGenerator.generateKey()
    }

    fun encryptWithBiometric(
        key: SecretKey,
        data: String,
        onSuccess: (ByteArray) -> Unit,
        onError: (String) -> Unit
    ) {
        val cipher = Cipher.getInstance("AES/GCM/NoPadding").apply {
            init(Cipher.ENCRYPT_MODE, key)
        }

        val promptInfo = BiometricPrompt.PromptInfo.Builder()
            .setTitle("보안 인증")
            .setSubtitle("데이터 암호화를 위해 생체인증이 필요합니다")
            .setNegativeButtonText("취소")
            .build()

        BiometricPrompt(
            activity,
            ContextCompat.getMainExecutor(activity),
            object : BiometricPrompt.AuthenticationCallback() {
                override fun onAuthenticationSucceeded(result: BiometricPrompt.AuthenticationResult) {
                    val c = result.cryptoObject?.cipher ?: return onError("Cipher 없음")
                    val iv = c.iv
                    val encrypted = c.doFinal(data.toByteArray())
                    onSuccess(iv + encrypted) // IV 앞에 붙여서 반환
                }
                override fun onAuthenticationError(code: Int, msg: CharSequence) = onError(msg.toString())
                override fun onAuthenticationFailed() = onError("인증 실패")
            }
        ).authenticate(promptInfo, BiometricPrompt.CryptoObject(cipher))
    }
}
```

---

## 2. Certificate Pinning (인증서 피닝)

### 2.1 왜 Certificate Pinning이 필요한가?

일반 TLS는 시스템에 설치된 루트 CA를 기반으로 신뢰를 검증합니다. 이 방식의 문제:

- **MITM 공격**: 공격자가 자신의 CA를 기기에 심고 트래픽 감청
- **CA 침해**: 신뢰 CA가 해킹당하면 임의 도메인에 가짜 인증서 발급 가능
- **기업 MDM 프록시**: 관리되는 기기에서 모든 TLS 트래픽 가로챌 수 있음

Certificate Pinning은 앱이 **특정 공개키 해시만** 신뢰하도록 강제하여 이 문제를 원천 차단합니다.

### 2.2 방법 1: Network Security Configuration (권장)

Android 7.0(API 24)+ 에서 코드 변경 없이 XML만으로 구현합니다.

**res/xml/network_security_config.xml**:

```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <domain-config>
        <domain includeSubdomains="true">api.myapp.com</domain>
        <pin-set expiration="2027-06-15">
            <!-- 현재 운영 인증서의 공개키 SHA-256 해시 -->
            <pin digest="SHA-256">AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=</pin>
            <!-- 백업 핀: 인증서 교체 시 연속성 보장 (필수) -->
            <pin digest="SHA-256">BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=</pin>
        </pin-set>
        <trust-anchors>
            <certificates src="system"/>
        </trust-anchors>
    </domain-config>

    <!-- 디버그 빌드에서만 프록시 도구 허용 -->
    <debug-overrides>
        <trust-anchors>
            <certificates src="user"/>
            <certificates src="system"/>
        </trust-anchors>
    </debug-overrides>
</network-security-config>
```

**AndroidManifest.xml**에 등록:

```xml
<application
    android:networkSecurityConfig="@xml/network_security_config"
    ...>
```

**공개키 해시 추출 명령어**:

```bash
openssl s_client -connect api.myapp.com:443 2>/dev/null \
  | openssl x509 -pubkey -noout \
  | openssl pkey -pubin -outform der \
  | openssl dgst -sha256 -binary \
  | base64
```

### 2.3 방법 2: OkHttp CertificatePinner

동적 핀 설정이나 런타임 제어가 필요할 때 사용합니다.

```kotlin
import okhttp3.CertificatePinner
import okhttp3.OkHttpClient
import retrofit2.Retrofit
import retrofit2.converter.gson.GsonConverterFactory
import java.util.concurrent.TimeUnit

object NetworkModule {

    private const val HOST = "api.myapp.com"
    private const val BASE_URL = "https://$HOST/"

    private val certificatePinner = CertificatePinner.Builder()
        .add(HOST, "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=") // 운영 핀
        .add(HOST, "sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=") // 백업 핀
        .build()

    private val okHttpClient = OkHttpClient.Builder()
        .certificatePinner(certificatePinner)
        .connectTimeout(30, TimeUnit.SECONDS)
        .readTimeout(30, TimeUnit.SECONDS)
        .writeTimeout(30, TimeUnit.SECONDS)
        .build()

    val retrofit: Retrofit = Retrofit.Builder()
        .baseUrl(BASE_URL)
        .client(okHttpClient)
        .addConverterFactory(GsonConverterFactory.create())
        .build()
}
```

**개발 중 실제 핀 값 확인용 디버그 Interceptor**:

```kotlin
// DEBUG 빌드에서만 사용 — 프로덕션 절대 포함 금지
class PinLoggingInterceptor : okhttp3.Interceptor {
    override fun intercept(chain: okhttp3.Interceptor.Chain): okhttp3.Response {
        val response = chain.proceed(chain.request())
        response.handshake?.peerCertificates?.forEachIndexed { i, cert ->
            val hash = java.security.MessageDigest.getInstance("SHA-256")
                .digest(cert.publicKey.encoded)
            val b64 = android.util.Base64.encodeToString(hash, android.util.Base64.NO_WRAP)
            android.util.Log.d("CertPin", "[$i] sha256/$b64")
        }
        return response
    }
}
```

---

## 3. 실전 통합: Keystore + Certificate Pinning 완성 패턴

API 토큰을 Keystore로 암호화 보관하고, Certificate Pinning이 적용된 OkHttp 클라이언트에서 자동 주입하는 완성형 패턴입니다.

```kotlin
import android.content.Context
import android.security.keystore.KeyGenParameterSpec
import android.security.keystore.KeyProperties
import okhttp3.Interceptor
import okhttp3.OkHttpClient
import okhttp3.Request
import retrofit2.Retrofit
import retrofit2.converter.gson.GsonConverterFactory
import java.security.KeyStore
import javax.crypto.Cipher
import javax.crypto.KeyGenerator
import javax.crypto.SecretKey
import javax.crypto.spec.GCMParameterSpec

class SecureApiClient(private val context: Context) {

    private val keyAlias = "api_token_key"
    private val keyStore = KeyStore.getInstance("AndroidKeyStore").apply { load(null) }
    private val prefs = context.getSharedPreferences("secure_prefs", Context.MODE_PRIVATE)

    fun saveToken(token: String) {
        val cipher = Cipher.getInstance("AES/GCM/NoPadding")
        cipher.init(Cipher.ENCRYPT_MODE, resolveKey())
        val iv = cipher.iv
        val encrypted = cipher.doFinal(token.toByteArray())
        prefs.edit()
            .putString("iv", android.util.Base64.encodeToString(iv, android.util.Base64.DEFAULT))
            .putString("token", android.util.Base64.encodeToString(encrypted, android.util.Base64.DEFAULT))
            .apply()
    }

    private fun loadToken(): String {
        val iv = android.util.Base64.decode(prefs.getString("iv", null)!!, android.util.Base64.DEFAULT)
        val enc = android.util.Base64.decode(prefs.getString("token", null)!!, android.util.Base64.DEFAULT)
        val cipher = Cipher.getInstance("AES/GCM/NoPadding")
        cipher.init(Cipher.DECRYPT_MODE, resolveKey(), GCMParameterSpec(128, iv))
        return String(cipher.doFinal(enc))
    }

    private fun resolveKey(): SecretKey {
        keyStore.getKey(keyAlias, null)?.let { return it as SecretKey }
        val kg = KeyGenerator.getInstance(KeyProperties.KEY_ALGORITHM_AES, "AndroidKeyStore")
        kg.init(
            KeyGenParameterSpec.Builder(
                keyAlias,
                KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT
            )
                .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
                .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
                .setKeySize(256)
                .build()
        )
        return kg.generateKey()
    }

    private inner class AuthInterceptor : Interceptor {
        override fun intercept(chain: Interceptor.Chain): okhttp3.Response {
            val req: Request = chain.request().newBuilder()
                .addHeader("Authorization", "Bearer ${loadToken()}")
                .build()
            return chain.proceed(req)
        }
    }

    fun buildRetrofit(): Retrofit {
        val pinner = okhttp3.CertificatePinner.Builder()
            .add("api.myapp.com", "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=")
            .add("api.myapp.com", "sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=")
            .build()

        val client = OkHttpClient.Builder()
            .certificatePinner(pinner)
            .addInterceptor(AuthInterceptor())
            .build()

        return Retrofit.Builder()
            .baseUrl("https://api.myapp.com/")
            .client(client)
            .addConverterFactory(GsonConverterFactory.create())
            .build()
    }
}
```

---

## 4. 주의사항 및 실전 팁

### 4.1 인증서 만료 — 가장 흔한 장애 원인

Certificate Pinning의 최대 위험은 **인증서 갱신 시 앱 전체 통신 차단**입니다.

대응 전략:
1. **백업 핀 필수 포함**: 최소 2개 핀 설정 (운영 핀 + 백업 핀)
2. **만료일 설정**: `expiration` 속성으로 만료 전 앱 업데이트 유도
3. **갱신 절차**: 인증서 갱신 최소 30일 전에 새 핀을 백업 핀으로 배포 → 갱신 완료 후 이전 핀 제거

### 4.2 디버그 환경 Charles/Proxy 허용

Network Security Config의 `<debug-overrides>`를 활용하면 릴리즈 빌드 보안을 유지하면서 개발 중 프록시 도구 사용이 가능합니다. 위 XML 예제처럼 `src="user"` 항목을 debug-overrides 안에만 넣어두세요.

### 4.3 키 버전 관리 (Key Rotation)

장기 운영 앱은 Keystore 키도 주기적으로 교체해야 합니다.

```kotlin
// 버전 접미사로 키 관리
private val currentKeyAlias = "secure_key_v2"

fun migrateFromV1() {
    val oldAlias = "secure_key_v1"
    val ks = KeyStore.getInstance("AndroidKeyStore").apply { load(null) }
    if (!ks.containsAlias(oldAlias)) return

    // 1. 기존 키로 복호화
    val plainText = decryptWithAlias(oldAlias, loadEncryptedData())
    // 2. 새 키로 재암호화 저장
    saveEncryptedData(encryptWithAlias(currentKeyAlias, plainText))
    // 3. 이전 키 삭제
    ks.deleteEntry(oldAlias)
}
```

### 4.4 루팅 기기 탐지와의 조합

Keystore와 Certificate Pinning은 강력하지만, 루팅된 기기에서는 Xposed/Frida 등으로 우회될 수 있습니다. Play Integrity API와 함께 사용하면 기기 무결성 검증까지 추가할 수 있습니다.

---

## 마치며

Android Keystore와 Certificate Pinning은 현대 Android 보안의 두 핵심 축입니다. Keystore는 암호화 키를 하드웨어 수준에서 보호하고, Certificate Pinning은 네트워크 통신의 신뢰성을 강제합니다. 두 기법을 함께 적용하면 루팅 기기나 MITM 공격에 대한 방어 깊이(Defense in Depth)가 확보됩니다.

운영 측면에서는 인증서 만료 관리, 백업 핀 관리, 키 교체 전략이 반드시 사전에 설계되어야 합니다. 보안은 구현만큼이나 **지속적인 유지관리**가 중요합니다.

## 참고 자료

- [Android Keystore System — Android Developers](https://developer.android.com/privacy-and-security/keystore)
- [Network Security Configuration — Android Developers](https://developer.android.com/privacy-and-security/security-config)
- [Android Security Checklist — Android Developers](https://developer.android.com/privacy-and-security/security-tips)
