---
layout: post
title: "Android ConnectivityManager & NetworkCallback 심화: Kotlin Flow로 네트워크 상태를 실시간 감지하는 완전한 가이드"
date: 2026-10-01
categories: [android, flutter]
tags: [android, connectivity, networkcallback, kotlin, flow, coroutines, hilt]
---

네트워크 연결 상태 감지는 모든 Android 앱의 핵심 기능 중 하나입니다. 그러나 전통적인 `BroadcastReceiver` 방식은 Android 7.0(API 24)부터 명시적으로 제한되기 시작했고, `ConnectivityManager`의 `NetworkCallback` API와 Kotlin `callbackFlow`를 결합한 현대적인 방식이 표준으로 자리 잡았습니다. 이 글에서는 네트워크 상태 감지의 내부 동작 원리부터 Hilt 기반의 실전 구현까지 완전히 분해합니다.

## 왜 BroadcastReceiver를 버려야 하는가

Android 7.0 이전에는 `CONNECTIVITY_CHANGE` 액션의 `BroadcastReceiver`를 AndroidManifest에 선언하는 방식이 일반적이었습니다. 문제는 앱이 백그라운드에 있을 때도 이 인텐트가 전달되어 불필요한 배터리 소모를 유발했다는 점입니다. 구글은 이를 해결하기 위해 다음과 같은 단계적 제한을 도입했습니다.

- **Android 7.0 (API 24)**: 암시적 `CONNECTIVITY_CHANGE` 브로드캐스트를 Manifest 등록으로는 수신 불가
- **Android 8.0 (API 26)**: 백그라운드 브로드캐스트 제한 대폭 강화
- **Android 12 (API 31)**: `registerReceiver()` 동적 등록 시 `RECEIVER_EXPORTED` / `RECEIVER_NOT_EXPORTED` 플래그 필수

이러한 변화의 대안으로 `ConnectivityManager.NetworkCallback`이 API 21부터 제공되었고, API 24의 `registerDefaultNetworkCallback()`을 거쳐 현재의 완성된 형태가 되었습니다.

## ConnectivityManager 핵심 API 해부

### NetworkRequest

네트워크를 요청하거나 구독할 때는 `NetworkRequest`를 먼저 구성합니다. 두 가지 핵심 축이 있습니다.

**Transport Type (전송 방식)**
- `TRANSPORT_WIFI` — Wi-Fi
- `TRANSPORT_CELLULAR` — 모바일 데이터
- `TRANSPORT_ETHERNET` — 이더넷
- `TRANSPORT_BLUETOOTH` — 블루투스 PAN

**Network Capability (네트워크 능력)**
- `NET_CAPABILITY_INTERNET` — 인터넷 라우팅 설정이 되어 있음 (실제 연결 ≠ 보장)
- `NET_CAPABILITY_VALIDATED` — DNS/HTTP 검증을 통과한 실제 인터넷 연결
- `NET_CAPABILITY_NOT_METERED` — 무제한 요금제(Wi-Fi 등)
- `NET_CAPABILITY_NOT_VPN` — VPN이 아님

`NET_CAPABILITY_INTERNET`과 `NET_CAPABILITY_VALIDATED`의 차이가 중요합니다. 전자는 "인터넷에 가는 경로가 설정됨"을 의미하고, 후자는 "실제로 공공 인터넷에 도달 가능함"을 의미합니다. 포털이 있는 Wi-Fi에 연결됐을 때는 `INTERNET`은 true지만 `VALIDATED`는 false입니다.

### NetworkCallback 메서드

```kotlin
object : ConnectivityManager.NetworkCallback() {
    // 요청 조건을 만족하는 네트워크가 생겼을 때
    override fun onAvailable(network: Network) {}

    // 네트워크 연결이 끊겼을 때
    override fun onLost(network: Network) {}

    // 네트워크의 capabilities가 변경됐을 때 (VALIDATED 상태 변화 포함)
    override fun onCapabilitiesChanged(
        network: Network,
        networkCapabilities: NetworkCapabilities
    ) {}

    // 링크 속성(IP, DNS 등)이 변경됐을 때
    override fun onLinkPropertiesChanged(network: Network, linkProperties: LinkProperties) {}

    // 사용 가능한 네트워크가 없어서 블로킹됐을 때
    override fun onUnavailable() {}
}
```

`registerDefaultNetworkCallback()`은 시스템이 선택한 기본 네트워크(가장 선호되는 네트워크)의 변화만 알려줍니다. `requestNetwork()`는 특정 조건의 네트워크를 "요청"하며, `registerNetworkCallback()`은 조건에 맞는 모든 네트워크 변화를 알려줍니다.

## Kotlin callbackFlow로 Flow 변환

콜백 기반 API를 Flow로 래핑하는 표준 패턴은 `callbackFlow`입니다. `trySend()`와 `awaitClose()`를 사용해 생명주기를 안전하게 관리합니다.

```kotlin
// domain/NetworkObserver.kt
interface NetworkObserver {
    val networkStatus: Flow<NetworkStatus>
}

sealed class NetworkStatus {
    data class Available(val isValidated: Boolean) : NetworkStatus()
    object Lost : NetworkStatus()
    object Unavailable : NetworkStatus()
}
```

```kotlin
// data/AndroidNetworkObserver.kt
@Singleton
class AndroidNetworkObserver @Inject constructor(
    @ApplicationContext private val context: Context
) : NetworkObserver {

    private val connectivityManager =
        context.getSystemService(ConnectivityManager::class.java)

    override val networkStatus: Flow<NetworkStatus> = callbackFlow {
        val callback = object : ConnectivityManager.NetworkCallback() {
            override fun onAvailable(network: Network) {
                val caps = connectivityManager.getNetworkCapabilities(network)
                val isValidated = caps
                    ?.hasCapability(NetworkCapabilities.NET_CAPABILITY_VALIDATED) == true
                trySend(NetworkStatus.Available(isValidated))
            }

            override fun onLost(network: Network) {
                trySend(NetworkStatus.Lost)
            }

            override fun onUnavailable() {
                trySend(NetworkStatus.Unavailable)
            }

            override fun onCapabilitiesChanged(
                network: Network,
                networkCapabilities: NetworkCapabilities
            ) {
                val isValidated = networkCapabilities
                    .hasCapability(NetworkCapabilities.NET_CAPABILITY_VALIDATED)
                trySend(NetworkStatus.Available(isValidated))
            }
        }

        val request = NetworkRequest.Builder()
            .addCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET)
            .build()

        connectivityManager.registerNetworkCallback(request, callback)

        // Flow가 cancel되면 콜백 해제
        awaitClose {
            connectivityManager.unregisterNetworkCallback(callback)
        }
    }.distinctUntilChanged()
        .shareIn(
            scope = CoroutineScope(SupervisorJob() + Dispatchers.IO),
            started = SharingStarted.WhileSubscribed(5_000),
            replay = 1
        )
}
```

`shareIn`의 세 가지 핵심 파라미터를 주목하세요. `SharingStarted.WhileSubscribed(5_000)`는 마지막 구독자가 사라진 후 5초 뒤에 업스트림을 종료합니다. 이렇게 하면 화면 회전 같은 짧은 생명주기 변화에도 네트워크 재등록 없이 캐시된 값을 즉시 제공합니다. `replay = 1`은 새 구독자에게 가장 최근 상태를 즉시 전달합니다.

## ViewModel과 UI 연동

```kotlin
@HiltViewModel
class HomeViewModel @Inject constructor(
    private val networkObserver: NetworkObserver
) : ViewModel() {

    val networkStatus: StateFlow<NetworkStatus> = networkObserver.networkStatus
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            initialValue = NetworkStatus.Unavailable
        )

    // 오프라인 큐: 연결 복구 시 재시도
    private val pendingActions = ArrayDeque<suspend () -> Unit>()

    fun submitAction(action: suspend () -> Unit) {
        viewModelScope.launch {
            networkObserver.networkStatus
                .filter { it is NetworkStatus.Available }
                .first()
            action()
        }
    }
}
```

```kotlin
// UI (Compose)
@Composable
fun HomeScreen(viewModel: HomeViewModel = hiltViewModel()) {
    val networkStatus by viewModel.networkStatus.collectAsStateWithLifecycle()

    // 네트워크 배너 표시
    AnimatedVisibility(visible = networkStatus is NetworkStatus.Lost) {
        Surface(color = MaterialTheme.colorScheme.errorContainer) {
            Text(
                text = "인터넷 연결이 끊겼습니다",
                modifier = Modifier
                    .fillMaxWidth()
                    .padding(horizontal = 16.dp, vertical = 8.dp),
                color = MaterialTheme.colorScheme.onErrorContainer,
                style = MaterialTheme.typography.bodySmall
            )
        }
    }
}
```

## 현재 네트워크 상태 즉시 조회

Flow를 구독하기 전에 현재 상태를 즉시 알고 싶다면 `getActiveNetwork()`와 `getNetworkCapabilities()`를 사용합니다.

```kotlin
fun isCurrentlyConnected(): Boolean {
    val network = connectivityManager.activeNetwork ?: return false
    val caps = connectivityManager.getNetworkCapabilities(network) ?: return false
    return caps.hasCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET) &&
           caps.hasCapability(NetworkCapabilities.NET_CAPABILITY_VALIDATED)
}
```

단, 이 방식은 스냅샷이므로 실시간 모니터링이 필요하면 항상 `NetworkCallback`을 사용해야 합니다.

## Hilt 모듈 설정

```kotlin
@Module
@InstallIn(SingletonComponent::class)
abstract class NetworkModule {

    @Binds
    @Singleton
    abstract fun bindNetworkObserver(
        impl: AndroidNetworkObserver
    ): NetworkObserver
}
```

## 주의사항 및 팁

### 1. 포어그라운드 서비스와 백그라운드 제한
백그라운드에서 네트워크 콜백을 계속 받으려면 포어그라운드 서비스를 사용해야 합니다. `ViewModel`의 `viewModelScope`는 앱이 백그라운드로 가면 `collectAsStateWithLifecycle`에 의해 자동으로 일시 중지됩니다—이는 의도된 동작입니다.

### 2. API 26+ `registerDefaultNetworkCallback` 활용
단순히 "현재 사용 중인 네트워크"만 감시하면 된다면 `NetworkRequest` 없이 `registerDefaultNetworkCallback(callback)`을 사용하세요. 더 가볍고 시스템 부담도 적습니다.

### 3. 멀티 네트워크 시나리오
Wi-Fi와 모바일 데이터가 동시에 연결되어 있을 때 `registerNetworkCallback()`은 두 네트워크 각각에 대한 이벤트를 받습니다. 이를 적절히 집계해 "하나라도 사용 가능하면 연결됨"으로 처리할 때는 `network` 파라미터로 구분하고 Set로 관리합니다.

```kotlin
// 멀티 네트워크 관리 예시
private val availableNetworks = mutableSetOf<Network>()

override fun onAvailable(network: Network) {
    availableNetworks.add(network)
    trySend(NetworkStatus.Available(isValidated = true))
}

override fun onLost(network: Network) {
    availableNetworks.remove(network)
    if (availableNetworks.isEmpty()) {
        trySend(NetworkStatus.Lost)
    }
}
```

### 4. 테스트 전략
단위 테스트에서 `NetworkObserver`는 인터페이스로 추상화되어 있으므로 Fake 구현체로 교체하기 쉽습니다.

```kotlin
class FakeNetworkObserver(
    private val statusFlow: Flow<NetworkStatus> = flowOf(NetworkStatus.Available(true))
) : NetworkObserver {
    override val networkStatus: Flow<NetworkStatus> = statusFlow
}

// ViewModel 테스트
@Test
fun `오프라인 상태에서 대기 후 온라인 복구 시 액션 실행`() = runTest {
    val statusChannel = Channel<NetworkStatus>()
    val observer = FakeNetworkObserver(statusChannel.receiveAsFlow())
    val viewModel = HomeViewModel(observer)

    var actionExecuted = false
    viewModel.submitAction { actionExecuted = true }

    statusChannel.send(NetworkStatus.Available(true))
    advanceUntilIdle()

    assertTrue(actionExecuted)
}
```

### 5. 권한 확인
`ACCESS_NETWORK_STATE` 권한이 `AndroidManifest.xml`에 반드시 선언되어야 합니다. 런타임 권한이 아니므로 사용자 허가를 요청할 필요는 없지만, 누락 시 `SecurityException`이 발생합니다.

```xml
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
```

## 마무리

`ConnectivityManager` + `callbackFlow` + `shareIn`의 조합은 Android 네트워크 상태 모니터링의 현대적 표준입니다. 단순한 연결 여부 확인을 넘어 `NET_CAPABILITY_VALIDATED`로 실제 인터넷 도달 가능 여부를 판단하고, 멀티 네트워크 시나리오를 Set로 관리하며, Hilt로 의존성을 주입해 테스트 가능한 구조를 만드는 것이 핵심입니다. 한 번 제대로 구축해두면 앱 전반에서 일관된 오프라인 UX를 제공하는 견고한 인프라가 됩니다.

## 참고 자료
- [Monitor connectivity status and connection metering — Android Developers](https://developer.android.com/training/monitoring-device-state/connectivity-monitoring)
- [Read network state — Android Developers](https://developer.android.com/training/basics/network-ops/reading-network-state)
