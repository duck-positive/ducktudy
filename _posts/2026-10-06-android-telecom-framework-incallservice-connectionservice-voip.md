---
layout: post
title: "Android Telecom Framework 심화: InCallService와 ConnectionService로 VoIP 전화 앱 구현하기"
date: 2026-10-06
categories: [android]
tags: [android, telecom, voip, incallservice, connectionservice, kotlin]
---

## 개요

스마트폰의 핵심 기능 중 하나인 '전화'는 Android에서 단순히 하드웨어 통화만을 의미하지 않습니다. Zoom, Google Meet, WhatsApp, LINE처럼 인터넷을 통해 통화하는 VoIP(Voice over IP) 앱들도 모두 Android OS와 유기적으로 통합되어 있습니다. 이 통합의 핵심이 바로 **Android Telecom Framework**입니다.

Telecom Framework는 API 21(Android 5.0)부터 제공되어 왔으며, 기기의 모든 통화(PSTN 기반 이동통신 및 VoIP)를 중앙에서 조율하는 운영 체제 수준의 인프라입니다. 이 프레임워크를 제대로 이해하면 일반 다이얼러 앱 대체, 스팸 차단, VoIP 통합 등 전화와 관련된 고급 기능을 구현할 수 있습니다.

---

## Android Telecom Framework란 무엇인가?

Android Telecom Framework(이하 Telecom)는 기기에서 발생하는 오디오/비디오 통화를 관리하는 시스템 서비스입니다. PSTN(이동통신망)과 VoIP를 모두 아우르며, 세 가지 주요 컴포넌트로 구성됩니다.

### 핵심 컴포넌트

| 컴포넌트 | 역할 |
|---|---|
| **Telecom (시스템)** | 교환기(switchboard) 역할. ConnectionService와 InCallService를 연결 |
| **ConnectionService** | 실제 통화 연결을 담당 (VoIP 엔진, SIP, WebRTC 등) |
| **InCallService** | 통화 중 UI를 담당 (다이얼러 화면, 버튼 등) |

Telecom은 교환기처럼 동작합니다. ConnectionService가 통화를 생성하면, Telecom이 이를 InCallService에 전달해 UI에 표시합니다. 개발자는 이 둘 중 하나 또는 둘 다 구현할 수 있습니다.

### 세 가지 통합 방식

1. **Self-Managed ConnectionService**: 독립형 VoIP 앱 (자체 UI 포함). 기본 다이얼러 앱에 표시되지 않음
2. **Managed ConnectionService**: VoIP 엔진만 제공, UI는 기기 기본 다이얼러 앱 사용
3. **InCallService + ConnectionService**: 완전한 다이얼러 앱 대체

---

## 왜 Telecom Framework가 필요한가?

### 1. 시스템 수준 오디오 충돌 방지

여러 앱이 동시에 통화를 시도하면 오디오 충돌이 발생할 수 있습니다. Telecom은 통화 포커스를 중앙에서 관리해 이를 방지합니다. VoIP 앱이 `AudioManager`를 직접 제어하면 시스템 전화나 다른 VoIP 앱과 충돌하지만, Telecom을 통하면 OS가 이를 안전하게 중재합니다.

### 2. 잠금 화면 통합

Telecom에 등록된 통화는 잠금 화면에 자동으로 표시됩니다. 별도의 잠금 화면 처리 로직 없이도 수신 전화 UI가 올바르게 동작합니다.

### 3. Do Not Disturb / 방해 금지 모드 준수

사용자가 방해 금지 모드를 설정했을 때, Telecom 기반 앱은 이를 자동으로 존중합니다. 직접 AudioManager를 다루면 방해 금지 모드를 우회할 수 있어 사용자 경험이 나빠집니다.

### 4. Android Auto / Wear OS 연동

Telecom 기반 통화는 Android Auto나 Wear OS 기기에 자동으로 전파되어 핸즈프리 통화가 가능합니다.

### 5. 스팸 필터링 통합

`CallScreeningService`를 등록하면 수신 전화를 인터셉트해 스팸 여부를 판단하고 차단 또는 경고를 표시할 수 있습니다.

---

## 실제 구현 예제

### 예제 1: Self-Managed ConnectionService 구현 (VoIP 앱)

가장 일반적인 VoIP 앱 구조입니다. Zoom, Teams 같은 앱이 이 방식을 사용합니다.

#### 1단계: 권한 및 Manifest 설정

```xml
<!-- AndroidManifest.xml -->
<uses-permission android:name="android.permission.MANAGE_OWN_CALLS" />
<uses-permission android:name="android.permission.RECORD_AUDIO" />

<application>
    <!-- ConnectionService 등록 -->
    <service
        android:name=".telecom.VoipConnectionService"
        android:permission="android.permission.BIND_TELECOM_CONNECTION_SERVICE"
        android:exported="true">
        <intent-filter>
            <action android:name="android.telecom.ConnectionService" />
        </intent-filter>
    </service>
</application>
```

#### 2단계: ConnectionService 구현

```kotlin
// VoipConnectionService.kt
class VoipConnectionService : ConnectionService() {

    override fun onCreateOutgoingConnection(
        connectionManagerPhoneAccount: PhoneAccountHandle,
        request: ConnectionRequest
    ): Connection {
        val connection = VoipConnection()
        connection.setAddress(request.address, TelecomManager.PRESENTATION_ALLOWED)
        connection.setCallerDisplayName(
            request.address.schemeSpecificPart,
            TelecomManager.PRESENTATION_ALLOWED
        )
        // 자체 VoIP 엔진(WebRTC 등) 호출을 여기서 시작
        connection.startVoipCall(request.address.schemeSpecificPart)
        return connection
    }

    override fun onCreateIncomingConnection(
        connectionManagerPhoneAccount: PhoneAccountHandle,
        request: ConnectionRequest
    ): Connection {
        val connection = VoipConnection()
        val callerNumber = request.extras.getString("caller_number") ?: "Unknown"
        connection.setAddress(
            Uri.parse("tel:$callerNumber"),
            TelecomManager.PRESENTATION_ALLOWED
        )
        connection.setCallerDisplayName(callerNumber, TelecomManager.PRESENTATION_ALLOWED)
        // RINGING 상태로 전환 → 시스템이 수신 알림 표시
        connection.setRinging()
        return connection
    }
}

// VoipConnection.kt
class VoipConnection : Connection() {

    private var webRtcEngine: WebRtcEngine? = null

    fun startVoipCall(remoteNumber: String) {
        webRtcEngine = WebRtcEngine()
        webRtcEngine?.connect(remoteNumber) { connected ->
            if (connected) {
                setActive() // ACTIVE 상태로 전환
            } else {
                setDisconnected(DisconnectCause(DisconnectCause.ERROR))
                destroy()
            }
        }
    }

    override fun onAnswer() {
        // 사용자가 수신 전화를 받음
        webRtcEngine?.accept()
        setActive()
    }

    override fun onReject() {
        // 사용자가 수신 전화를 거절
        webRtcEngine?.reject()
        setDisconnected(DisconnectCause(DisconnectCause.REJECTED))
        destroy()
    }

    override fun onHold() {
        // 통화 보류
        webRtcEngine?.hold()
        setOnHold()
    }

    override fun onUnhold() {
        // 보류 해제
        webRtcEngine?.resume()
        setActive()
    }

    override fun onDisconnect() {
        // 통화 종료
        webRtcEngine?.disconnect()
        webRtcEngine = null
        setDisconnected(DisconnectCause(DisconnectCause.LOCAL))
        destroy()
    }

    override fun onPlayDtmfTone(c: Char) {
        webRtcEngine?.sendDtmf(c)
    }
}
```

#### 3단계: PhoneAccount 등록 및 통화 시작

```kotlin
// TelecomHelper.kt
class TelecomHelper(private val context: Context) {

    private val telecomManager = context.getSystemService(TelecomManager::class.java)

    companion object {
        private val PHONE_ACCOUNT_HANDLE = PhoneAccountHandle(
            ComponentName("com.example.voipapp", "VoipConnectionService"),
            "voip_account_id"
        )
    }

    fun registerPhoneAccount() {
        val phoneAccount = PhoneAccount.builder(PHONE_ACCOUNT_HANDLE, "My VoIP App")
            .setCapabilities(PhoneAccount.CAPABILITY_SELF_MANAGED)
            .addSupportedUriScheme(PhoneAccount.SCHEME_SIP)
            .addSupportedUriScheme(PhoneAccount.SCHEME_TEL)
            .build()
        telecomManager.registerPhoneAccount(phoneAccount)
    }

    fun placeOutgoingCall(number: String) {
        val uri = Uri.parse("tel:$number")
        val extras = Bundle().apply {
            putParcelable(TelecomManager.EXTRA_PHONE_ACCOUNT_HANDLE, PHONE_ACCOUNT_HANDLE)
        }
        // 권한 확인 후 통화
        if (ActivityCompat.checkSelfPermission(context, Manifest.permission.CALL_PHONE)
            == PackageManager.PERMISSION_GRANTED) {
            telecomManager.placeCall(uri, extras)
        }
    }

    fun notifyIncomingCall(callerNumber: String) {
        val extras = Bundle().apply {
            putString("caller_number", callerNumber)
        }
        // 수신 전화 알림 → 시스템이 ConnectionService.onCreateIncomingConnection() 호출
        telecomManager.addNewIncomingCall(PHONE_ACCOUNT_HANDLE, extras)
    }
}
```

---

### 예제 2: InCallService 구현 (커스텀 다이얼러 앱)

기기의 기본 다이얼러 앱을 대체하고 싶다면 `InCallService`를 구현합니다.

#### 1단계: Manifest 설정

```xml
<!-- AndroidManifest.xml -->
<uses-permission android:name="android.permission.READ_PHONE_STATE" />

<application>
    <service
        android:name=".dialer.CustomInCallService"
        android:permission="android.permission.BIND_INCALL_SERVICE"
        android:exported="true">
        <intent-filter>
            <action android:name="android.telecom.InCallService" />
        </intent-filter>
        <!-- 기본 다이얼러로 등록될 수 있음을 표시 -->
        <meta-data
            android:name="android.telecom.IN_CALL_SERVICE_UI"
            android:value="true" />
    </service>

    <!-- 기본 다이얼러로 지정될 Activity -->
    <activity android:name=".dialer.InCallActivity">
        <intent-filter>
            <action android:name="android.intent.action.DIAL" />
            <category android:name="android.intent.category.DEFAULT" />
            <category android:name="android.intent.category.BROWSABLE" />
        </intent-filter>
    </activity>
</application>
```

#### 2단계: InCallService 구현

```kotlin
// CustomInCallService.kt
class CustomInCallService : InCallService() {

    companion object {
        val calls = MutableStateFlow<List<Call>>(emptyList())
    }

    override fun onCallAdded(call: Call) {
        super.onCallAdded(call)
        // 통화 상태 변화 감지를 위한 콜백 등록
        call.registerCallback(callCallback)
        calls.value = calls.value + call

        // 통화 화면 Activity 실행
        val intent = Intent(this, InCallActivity::class.java).apply {
            flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_SINGLE_TOP
        }
        startActivity(intent)
    }

    override fun onCallRemoved(call: Call) {
        super.onCallRemoved(call)
        call.unregisterCallback(callCallback)
        calls.value = calls.value - call
    }

    private val callCallback = object : Call.Callback() {
        override fun onStateChanged(call: Call, state: Int) {
            // UI 업데이트
            calls.value = calls.value.toList() // StateFlow 갱신 트리거
        }

        override fun onDetailsChanged(call: Call, details: Call.Details) {
            calls.value = calls.value.toList()
        }
    }
}

// InCallViewModel.kt
class InCallViewModel : ViewModel() {

    val activeCalls = CustomInCallService.calls.asStateFlow()

    fun answer(call: Call) {
        call.answer(VideoProfile.STATE_AUDIO_ONLY)
    }

    fun reject(call: Call) {
        call.reject(false, null)
    }

    fun disconnect(call: Call) {
        call.disconnect()
    }

    fun hold(call: Call) {
        call.hold()
    }

    fun unhold(call: Call) {
        call.unhold()
    }

    fun mute(muted: Boolean) {
        // InCallService.setMuted()를 통해 마이크 뮤트
        // Activity에서 service 참조를 통해 직접 호출 필요
    }

    fun getCallState(call: Call): String {
        return when (call.state) {
            Call.STATE_RINGING -> "수신 중"
            Call.STATE_DIALING -> "발신 중"
            Call.STATE_ACTIVE -> "통화 중"
            Call.STATE_HOLDING -> "보류 중"
            Call.STATE_DISCONNECTED -> "통화 종료"
            Call.STATE_CONNECTING -> "연결 중"
            else -> "알 수 없음"
        }
    }

    fun getCallerInfo(call: Call): String {
        val details = call.details
        return details.handle?.schemeSpecificPart
            ?: details.callerDisplayName
            ?: "알 수 없는 번호"
    }
}
```

#### 3단계: 기본 다이얼러 요청

```kotlin
// MainActivity.kt (앱 설치 후 초기 설정)
class MainActivity : AppCompatActivity() {

    private val requestDefaultDialer =
        registerForActivityResult(ActivityResultContracts.StartActivityForResult()) { result ->
            if (result.resultCode == RESULT_OK) {
                Toast.makeText(this, "기본 다이얼러로 설정되었습니다.", Toast.LENGTH_SHORT).show()
            }
        }

    private fun requestToBeDefaultDialer() {
        val roleManager = getSystemService(RoleManager::class.java)
        if (roleManager.isRoleAvailable(RoleManager.ROLE_DIALER)) {
            if (!roleManager.isRoleHeld(RoleManager.ROLE_DIALER)) {
                val intent = roleManager.createRequestRoleIntent(RoleManager.ROLE_DIALER)
                requestDefaultDialer.launch(intent)
            }
        }
    }
}
```

---

### 예제 3: Core-Telecom 라이브러리 활용 (권장, Jetpack)

Android Jetpack의 `core-telecom` 라이브러리는 Self-Managed ConnectionService를 더 간단하게 구현할 수 있도록 도와줍니다.

```kotlin
// build.gradle.kts
dependencies {
    implementation("androidx.core:core-telecom:1.0.0")
}
```

```kotlin
// VoipApp.kt
class VoipApp : Application() {

    lateinit var callsManager: CallsManager

    override fun onCreate() {
        super.onCreate()
        callsManager = CallsManager(this)
        // CAPABILITY_SUPPORTS_CALL_STREAMING: Android Auto 지원
        callsManager.registerAppWithTelecom(
            CallsManager.CAPABILITY_SUPPORTS_VIDEO_CALLING or
            CallsManager.CAPABILITY_SUPPORTS_CALL_STREAMING
        )
    }
}

// CallRepository.kt
class CallRepository(private val callsManager: CallsManager) {

    suspend fun startOutgoingCall(number: String) {
        val callAttributes = CallAttributesCompat(
            displayName = number,
            address = Uri.parse("tel:$number"),
            direction = CallAttributesCompat.DIRECTION_OUTGOING,
            callType = CallAttributesCompat.CALL_TYPE_AUDIO_CALL,
            callCapabilities = CallAttributesCompat.SUPPORTS_SET_INACTIVE or
                               CallAttributesCompat.SUPPORTS_TRANSFER
        )

        callsManager.addCall(
            callAttributes = callAttributes,
            onAnswer = { videoState ->
                // 수신 통화 응답 처리
            },
            onDisconnect = { disconnectCause ->
                // 통화 종료 처리
            },
            onSetActive = {
                // 통화 활성화
            },
            onSetInactive = {
                // 통화 비활성화 (보류)
            }
        ) { callControlScope ->
            // 이 블록 내에서 CallControlScope를 통해 통화를 제어
            // 오디오 엔드포인트 변경 감지
            callControlScope.currentCallEndpoint.collect { endpoint ->
                when (endpoint.endpointType) {
                    CallEndpointCompat.TYPE_EARPIECE -> { /* 이어폰 */ }
                    CallEndpointCompat.TYPE_SPEAKER -> { /* 스피커 */ }
                    CallEndpointCompat.TYPE_BLUETOOTH -> { /* 블루투스 */ }
                    CallEndpointCompat.TYPE_WIRED_HEADSET -> { /* 유선 헤드셋 */ }
                }
            }
        }
    }
}
```

---

## 주의사항 및 팁

### 1. MANAGE_OWN_CALLS vs READ_PHONE_STATE 권한

| 권한 | 용도 |
|---|---|
| `MANAGE_OWN_CALLS` | Self-Managed ConnectionService 등록 |
| `READ_PHONE_STATE` | 통화 상태 감지 (InCallService용) |
| `CALL_PHONE` | `placeCall()` 호출 시 필요 |

`CALL_PHONE`은 위험 권한이므로 런타임에 요청해야 합니다. Self-Managed 방식의 경우 `MANAGE_OWN_CALLS`만으로도 통화 추가가 가능합니다.

### 2. 포그라운드 서비스 필수

Android 10 이상에서는 통화 추가 후 5초 이내에 포그라운드 서비스 알림을 표시해야 합니다. 이를 어기면 `SecurityException`이 발생합니다.

```kotlin
// 수신 전화 알림 시 포그라운드 서비스 시작
class VoipForegroundService : Service() {
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        val notification = buildCallNotification()
        startForeground(NOTIFICATION_ID, notification)
        return START_STICKY
    }
}
```

### 3. AudioManager 직접 사용 금지

Telecom Framework를 사용할 때는 `AudioManager.requestAudioFocus()`를 직접 호출하지 마세요. Telecom이 자동으로 오디오 포커스를 관리합니다. 직접 호출하면 시스템 통화와의 충돌 및 예측 불가능한 동작이 발생할 수 있습니다. 오디오 라우팅은 반드시 `CallControlScope.requestEndpointChange()`를 통해 처리하세요.

### 4. Connection 상태 전환 규칙

Connection의 상태는 정해진 순서대로 전환해야 합니다.

```
INITIALIZING → RINGING → ACTIVE → HOLDING → ACTIVE → DISCONNECTED
                   ↓
               DISCONNECTED (거절)
```

`destroy()`를 호출하지 않으면 Connection이 메모리에 계속 남아 있어 리소스 누수가 발생합니다. 반드시 `setDisconnected()` 후 `destroy()`를 호출하세요.

### 5. Self-Managed 통화와 시스템 전화 공존

Self-Managed 통화 진행 중 시스템 전화(PSTN)가 수신되면, Telecom이 자동으로 Self-Managed 통화에 `onCallAudioStateChanged()`를 호출해 보류 처리를 요청합니다. 이를 제대로 처리하지 않으면 두 통화 오디오가 동시에 재생될 수 있습니다.

```kotlin
override fun onConnectionEvent(event: String, extras: Bundle) {
    if (event == Connection.EVENT_CALL_HOLD) {
        // 시스템이 보류를 요청했을 때 처리
        onHold()
    }
}
```

### 6. 테스트 시 주의사항

- **에뮬레이터**: 에뮬레이터에서는 PSTN 통화가 제한되며, Self-Managed VoIP 테스트는 실제 기기에서 진행하는 것이 권장됩니다.
- **권한**: 기본 다이얼러 권한 요청은 에뮬레이터에서도 동작하지만, 실 기기 경험과 다를 수 있습니다.
- **다중 통화**: 보류 및 다중 통화 시나리오는 반드시 실 기기에서 검증하세요.

---

## 정리

Android Telecom Framework는 진입 장벽이 높지만, 이를 활용하면 OS와 완벽하게 통합된 VoIP 앱을 구현할 수 있습니다.

- **VoIP 전용 앱**: Self-Managed ConnectionService + core-telecom 라이브러리
- **다이얼러 대체 앱**: InCallService 구현 + RoleManager 요청
- **VoIP + 커스텀 UI**: ConnectionService + InCallService 함께 구현

Telecom Framework를 올바르게 사용하면 Android Auto 연동, 잠금 화면 통합, 방해 금지 모드 준수 등의 시스템 기능을 무료로 얻을 수 있으며, 사용자 입장에서는 네이티브 전화와 동일한 경험을 VoIP 앱에서도 누릴 수 있게 됩니다.

## 참고 자료
- [Android Telecom Framework 공식 개요](https://developer.android.com/develop/connectivity/telecom)
- [Core-Telecom (Self-Managed) 가이드](https://developer.android.com/develop/connectivity/telecom/selfManaged)
- [android.telecom 패키지 레퍼런스](https://developer.android.com/reference/android/telecom/package-summary)
