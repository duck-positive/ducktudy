---
layout: post
title: "Android Binder IPC와 AIDL 심화: 프로세스 간 통신의 핵심 원리 완전 정복"
date: 2026-10-20
categories: [android]
tags: [android, binder, aidl, ipc, bound-service, kotlin, parcelable]
---

Android 앱 개발을 하다 보면 "IPC"나 "Binder"라는 단어를 수없이 접하게 됩니다. 시스템 서비스와의 통신, 별도 프로세스로 실행되는 음악 플레이어, 다른 앱의 기능을 활용하는 컴포넌트 — 이 모든 것의 뒤에는 **Binder IPC** 메커니즘이 있습니다. 이 글에서는 Android의 프로세스 간 통신(IPC)이 어떻게 동작하는지를 밑바닥부터 이해하고, 이를 앱에서 실제로 활용하는 방법을 완전히 정복합니다.

## 1. 왜 IPC가 필요한가: Android 프로세스 격리 모델

Linux 커널 기반의 Android는 각 앱을 **독립된 프로세스**에서 실행합니다. 각 프로세스는 자신만의 가상 메모리 공간을 갖고 있어 다른 프로세스의 메모리에 직접 접근할 수 없습니다. 이 격리(isolation)는 보안과 안정성의 핵심이지만, 동시에 프로세스 간 데이터 교환을 어렵게 만듭니다.

Android가 Linux IPC(Socket, Pipe, Shared Memory)를 그대로 사용하지 않고 **Binder**를 별도로 개발한 이유는 세 가지입니다:

1. **성능**: 일반적인 Socket IPC는 데이터를 커널에 복사 후 다시 사용자 공간으로 복사하는 두 번의 복사(copy)가 발생합니다. Binder는 **mmap**을 활용해 한 번의 복사만 수행합니다.
2. **보안**: Binder는 UID/PID를 자동으로 전달해 호출자 신원을 검증할 수 있습니다.
3. **객체 지향**: Binder는 원격 객체를 마치 로컬 객체처럼 사용하는 인터페이스를 제공합니다.

### Binder 드라이버의 구조

```
[App Process A]          [Kernel: /dev/binder]          [App Process B]
   Client                   Binder Driver                   Service
     │                           │                              │
     │──── ioctl(BC_TRANSACTION) ▶│                              │
     │                           │──── mmap 공유 ──────────────▶│
     │                           │                              │
     │◀─── ioctl(BR_REPLY) ──────│◀─── ioctl(BC_REPLY) ────────│
```

Binder 드라이버(`/dev/binder`)는 커널 모듈로 동작하며, 데이터를 **한 번만 복사**해 수신 프로세스의 Binder 버퍼에 직접 매핑합니다. Binder 트랜잭션은 **동기(synchronous)** 방식이 기본으로, 클라이언트 스레드는 서버가 응답할 때까지 블로킹됩니다.

---

## 2. IPC 통신 방식 3가지 선택 기준

Android에서 프로세스 간 통신을 구현하는 방법은 세 가지입니다. 상황에 맞는 방법을 선택해야 합니다.

| 방식 | 프로세스 | 멀티스레드 | 복잡도 |
|------|----------|------------|--------|
| Binder 직접 상속 | 동일 프로세스 | 불필요 | 낮음 |
| Messenger | 다른 프로세스 | 단일 스레드 큐 | 중간 |
| AIDL | 다른 프로세스 | 병렬 처리 필요 | 높음 |

**Messenger**는 내부적으로 AIDL을 사용하지만 메시지를 단일 `Handler` 큐에 순차 처리합니다. 동시 요청이 많지 않다면 Messenger가 단순하고 안전합니다. 반면 **AIDL**은 Binder 스레드 풀에서 메서드가 병렬로 호출될 수 있어 서비스 구현이 **스레드 안전(thread-safe)**해야 합니다.

---

## 3. AIDL 실전 구현: 백그라운드 계산 서비스

실제 동작하는 예제를 통해 AIDL의 전체 흐름을 살펴봅니다. 별도 프로세스에서 실행되는 수학 계산 서비스를 만들어보겠습니다.

### 3.1 AIDL 인터페이스 정의

`src/main/aidl/com/example/calculator/ICalculatorService.aidl`:

```aidl
// ICalculatorService.aidl
package com.example.calculator;

// parcelable을 전달하려면 별도 .aidl 파일에서 선언해야 합니다
import com.example.calculator.CalculationResult;

interface ICalculatorService {
    // in: 서비스로만 전달, out: 서비스에서 채워서 반환, inout: 양방향
    CalculationResult calculate(in String expression);

    // oneway: 비동기 호출 (반환값 없음, 클라이언트 블로킹 없음)
    oneway void submitBatch(in List<String> expressions);

    int getPid();
}
```

`src/main/aidl/com/example/calculator/CalculationResult.aidl`:

```aidl
// CalculationResult.aidl - Parcelable 선언
package com.example.calculator;

parcelable CalculationResult;
```

### 3.2 Parcelable 데이터 클래스 구현

AIDL을 통해 커스텀 객체를 전달하려면 반드시 `Parcelable`을 구현해야 합니다. Kotlin에서는 `@Parcelize` 애노테이션을 활용하면 됩니다.

```kotlin
// CalculationResult.kt
import android.os.Parcelable
import kotlinx.parcelize.Parcelize

@Parcelize
data class CalculationResult(
    val expression: String,
    val result: Double,
    val isError: Boolean,
    val errorMessage: String = ""
) : Parcelable
```

### 3.3 서비스 구현 (별도 프로세스)

```kotlin
// CalculatorService.kt
class CalculatorService : Service() {

    // Binder Stub을 구현 — 이 메서드들은 Binder 스레드 풀에서 병렬 호출됨
    private val binder = object : ICalculatorService.Stub() {

        override fun calculate(expression: String): CalculationResult {
            // Binder.getCallingPid()/getCallingUid()로 호출자 검증 가능
            val callerUid = Binder.getCallingUid()
            Log.d(TAG, "calculate() called from UID: $callerUid")

            return try {
                val result = evaluateExpression(expression)
                CalculationResult(expression, result, false)
            } catch (e: Exception) {
                CalculationResult(expression, 0.0, true, e.message ?: "Unknown error")
            }
        }

        override fun submitBatch(expressions: List<String>) {
            // oneway 메서드: 클라이언트는 이 결과를 기다리지 않음
            // 서비스는 원하는 대로 비동기 처리 가능
            expressions.forEach { expr ->
                Log.d(TAG, "Batch processing: $expr")
                // 실제로는 Worker 또는 Coroutine으로 처리
            }
        }

        override fun getPid(): Int = Process.myPid()
    }

    override fun onBind(intent: Intent): IBinder = binder

    private fun evaluateExpression(expr: String): Double {
        // 실제 계산 로직 (간략화)
        val tokens = expr.split("+")
        return tokens.sumOf { it.trim().toDouble() }
    }

    companion object {
        private const val TAG = "CalculatorService"
    }
}
```

`AndroidManifest.xml`에서 서비스를 **별도 프로세스**로 선언합니다:

```xml
<service
    android:name=".CalculatorService"
    android:process=":calculator_process"
    android:exported="false" />
```

`android:process=":calculator_process"`는 앱의 메인 프로세스와 별개로 실행됩니다. `:` 접두어는 해당 앱에 비공개 프로세스임을 의미합니다.

### 3.4 클라이언트 연결 구현

```kotlin
// MainActivity.kt
class MainActivity : AppCompatActivity() {

    private var calculatorService: ICalculatorService? = null
    private var isServiceBound = false

    private val serviceConnection = object : ServiceConnection {
        override fun onServiceConnected(name: ComponentName, service: IBinder) {
            // asInterface(): 같은 프로세스면 Stub 직접 반환, 다른 프로세스면 Proxy 반환
            calculatorService = ICalculatorService.Stub.asInterface(service)
            isServiceBound = true
            Log.d(TAG, "Service connected. Service PID: ${calculatorService?.pid}")
            Log.d(TAG, "My PID: ${Process.myPid()}")
        }

        override fun onServiceDisconnected(name: ComponentName) {
            // 예기치 않은 연결 끊김 (정상 unbind 시에는 호출 안 됨)
            calculatorService = null
            isServiceBound = false
        }
    }

    override fun onStart() {
        super.onStart()
        val intent = Intent(this, CalculatorService::class.java)
        bindService(intent, serviceConnection, Context.BIND_AUTO_CREATE)
    }

    override fun onStop() {
        super.onStop()
        if (isServiceBound) {
            unbindService(serviceConnection)
            isServiceBound = false
        }
    }

    private fun performCalculation() {
        val service = calculatorService ?: return

        // 주의: AIDL 원격 호출은 메인 스레드에서 호출하면 ANR 위험!
        // 반드시 백그라운드 스레드에서 호출해야 합니다
        lifecycleScope.launch(Dispatchers.IO) {
            try {
                val result = service.calculate("10.5 + 20.3 + 5.2")
                withContext(Dispatchers.Main) {
                    if (result.isError) {
                        Log.e(TAG, "Calculation error: ${result.errorMessage}")
                    } else {
                        Log.d(TAG, "Result: ${result.result}") // 36.0
                    }
                }
            } catch (e: DeadObjectException) {
                // 서비스 프로세스가 예기치 않게 종료된 경우
                Log.e(TAG, "Service died: ${e.message}")
                isServiceBound = false
            } catch (e: RemoteException) {
                Log.e(TAG, "Remote call failed: ${e.message}")
            }
        }
    }

    companion object { private const val TAG = "MainActivity" }
}
```

---

## 4. Binder 보안: 호출자 신원 검증

Binder IPC의 강력한 기능 중 하나는 **호출자의 UID/PID**를 신뢰할 수 있게 전달받는 것입니다. 악의적인 앱이 다른 앱인 척 위장할 수 없습니다.

```kotlin
private val binder = object : ICalculatorService.Stub() {

    override fun calculate(expression: String): CalculationResult {
        val callerPid = Binder.getCallingPid()
        val callerUid = Binder.getCallingUid()

        // 특정 앱만 허용하려면 패키지명으로 검증
        val pm = applicationContext.packageManager
        val callerPackages = pm.getPackagesForUid(callerUid)

        val isAllowed = callerPackages?.contains("com.example.trusted.app") == true
        if (!isAllowed) {
            throw SecurityException("Caller UID $callerUid is not authorized")
        }

        // clearCallingIdentity(): 이 서비스 자신의 UID로 작업 수행 (권한 승격)
        val token = Binder.clearCallingIdentity()
        try {
            return doCalculate(expression)
        } finally {
            Binder.restoreCallingIdentity(token) // 반드시 복원해야 함
        }
    }
}
```

`Binder.clearCallingIdentity()`는 서비스가 자신의 권한으로 시스템 리소스에 접근해야 할 때 사용합니다. `restoreCallingIdentity()`로 반드시 복원해야 하며, `finally` 블록에서 호출하는 것이 안전합니다.

---

## 5. Messenger를 이용한 단방향 IPC (간단한 방법)

AIDL보다 단순한 시나리오에서는 `Messenger`가 더 나은 선택입니다. 내부적으로 AIDL을 사용하지만 메시지 큐 방식으로 순차 처리합니다.

```kotlin
// MessengerService.kt
class MessengerService : Service() {

    // 메시지를 처리할 Handler (메인 스레드 또는 HandlerThread에서 실행)
    private val handler = object : Handler(Looper.getMainLooper()) {
        override fun handleMessage(msg: Message) {
            when (msg.what) {
                MSG_GREET -> {
                    val name = msg.data.getString("name") ?: "Unknown"
                    Log.d(TAG, "Hello, $name!")

                    // 클라이언트에게 응답 보내기
                    val reply = Message.obtain(null, MSG_REPLY)
                    reply.data = Bundle().apply {
                        putString("greeting", "Hello from service, $name!")
                    }
                    try {
                        msg.replyTo?.send(reply) // replyTo: 클라이언트의 Messenger
                    } catch (e: RemoteException) {
                        Log.e(TAG, "Failed to send reply: ${e.message}")
                    }
                }
                else -> super.handleMessage(msg)
            }
        }
    }

    private val messenger = Messenger(handler)

    override fun onBind(intent: Intent): IBinder = messenger.binder

    companion object {
        const val MSG_GREET = 1
        const val MSG_REPLY = 2
        private const val TAG = "MessengerService"
    }
}

// 클라이언트에서 사용
class ClientActivity : AppCompatActivity() {

    private var serviceMessenger: Messenger? = null

    // 서비스로부터 응답을 받을 Messenger
    private val replyMessenger = Messenger(object : Handler(Looper.getMainLooper()) {
        override fun handleMessage(msg: Message) {
            if (msg.what == MessengerService.MSG_REPLY) {
                val greeting = msg.data.getString("greeting")
                Log.d("Client", "Received: $greeting")
            }
        }
    })

    private val connection = object : ServiceConnection {
        override fun onServiceConnected(name: ComponentName, service: IBinder) {
            serviceMessenger = Messenger(service)
            sendGreeting("World")
        }
        override fun onServiceDisconnected(name: ComponentName) {
            serviceMessenger = null
        }
    }

    private fun sendGreeting(name: String) {
        val msg = Message.obtain(null, MessengerService.MSG_GREET)
        msg.data = Bundle().apply { putString("name", name) }
        msg.replyTo = replyMessenger // 응답받을 Messenger 등록
        try {
            serviceMessenger?.send(msg)
        } catch (e: RemoteException) {
            Log.e("Client", "Failed to send message: ${e.message}")
        }
    }
}
```

---

## 6. 주의사항과 실전 팁

### 6.1 메인 스레드에서 원격 호출 금지

원격 Binder 호출은 **동기적으로 블로킹**됩니다. 서비스가 응답하는 데 시간이 걸리면 클라이언트 스레드가 그만큼 블로킹됩니다. 메인 스레드에서 호출하면 **ANR(Application Not Responding)**이 발생합니다. 반드시 `Dispatchers.IO` 또는 별도의 스레드에서 호출하세요.

### 6.2 AIDL 인터페이스 하위 호환성 유지

앱을 배포한 후에는 AIDL 인터페이스를 변경할 때 **하위 호환성**을 반드시 유지해야 합니다. 다른 앱이 이 인터페이스에 의존하고 있을 수 있기 때문입니다. 새 메서드는 추가할 수 있지만, 기존 메서드의 매개변수 순서나 타입을 변경하면 안 됩니다.

### 6.3 Stub.asInterface()의 동작 이해

```kotlin
// 같은 프로세스 내부라면: Stub 객체 자체를 반환 (IPC 없음)
// 다른 프로세스라면: Stub.Proxy 객체를 반환 (실제 Binder IPC 발생)
val service = ICalculatorService.Stub.asInterface(binder)
```

`Stub.asInterface()`는 IBinder가 동일 프로세스인지 자동 감지합니다. 같은 프로세스면 Proxy 없이 직접 호출해 불필요한 오버헤드를 피합니다.

### 6.4 서비스 연결 해제 시 NullPointerException 방지

`onServiceDisconnected()`는 예기치 않은 연결 끊김에만 호출됩니다. 정상적인 `unbindService()` 후에는 호출되지 않습니다. 항상 `calculatorService?.calculate()`처럼 null-safe 호출을 사용하세요.

### 6.5 Binder 스레드 풀 한도

Android의 Binder 스레드 풀은 **프로세스당 최대 16개**의 스레드로 제한됩니다. 동시에 많은 클라이언트가 동기 호출을 날리면 스레드 풀이 고갈될 수 있습니다. 고처리량 시나리오에서는 `oneway` 비동기 메서드 사용을 고려하세요.

---

## 7. 실제 활용 사례

Android 프레임워크 자체가 AIDL과 Binder로 가득합니다:

- **ActivityManagerService**: 액티비티 생명주기, 태스크 관리
- **WindowManagerService**: 윈도우 그리기, 터치 이벤트 분배
- **PackageManagerService**: 앱 설치, 권한 확인
- **MediaPlayerService**: 미디어 재생 (미디어 앱의 MediaSession도 Binder 기반)

앱 개발에서 AIDL이 유용한 시나리오:

- **백그라운드 다운로드 서비스**: 다운로드 진행률을 다른 앱에 실시간으로 제공
- **SDK/Library**: 라이브러리를 별도 프로세스로 격리해 메모리 안정성 확보
- **플러그인 아키텍처**: 동적으로 로드되는 플러그인 앱과의 통신

---

## 결론

Android Binder IPC와 AIDL은 처음에는 복잡하게 느껴지지만, 핵심 개념을 이해하면 강력한 도구가 됩니다. 핵심 요약:

1. **Binder**는 mmap 기반의 단일 복사 IPC로 성능과 보안을 동시에 제공합니다.
2. **AIDL**은 복잡한 마샬링 코드를 자동 생성해주는 인터페이스 정의 언어입니다.
3. 원격 호출은 반드시 **백그라운드 스레드**에서 수행하고, `DeadObjectException`을 처리해야 합니다.
4. **Binder.getCallingUid()**로 호출자 신원을 신뢰할 수 있게 검증할 수 있습니다.
5. 단순 시나리오에서는 **Messenger**가 AIDL보다 간단하고 안전합니다.

프로세스 격리가 필요한 서비스 아키텍처를 설계할 때 Binder/AIDL은 Android 생태계에서 가장 견고하고 효율적인 IPC 메커니즘입니다.

## 참고 자료
- [Android Interface Definition Language (AIDL) - Android Developers](https://developer.android.com/guide/components/aidl)
- [Bound Services 개요 - Android Developers](https://developer.android.com/guide/components/bound-services)
