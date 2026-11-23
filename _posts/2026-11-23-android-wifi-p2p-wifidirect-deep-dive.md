---
layout: post
title: "Android Wi-Fi Direct(P2P) 심화: WifiP2pManager·BroadcastReceiver·소켓 통신으로 기기 간 직접 데이터 전송 완전 정복"
date: 2026-11-23
categories: [android]
tags: [android, wifi-p2p, wifi-direct, wifip2pmanager, kotlin, networking, p2p]
---

## Wi-Fi Direct(P2P)란 무엇인가?

Wi-Fi Direct(공식 명칭: Wi-Fi P2P)는 중간 액세스 포인트(AP) 없이 두 대 이상의 Android 기기를 직접 Wi-Fi로 연결하는 기술입니다. Android 4.0(API 14)부터 `android.net.wifi.p2p` 패키지로 제공되며, 최대 수백 Mbps의 전송 속도와 수십 미터에 달하는 범위를 지원합니다.

Bluetooth와 비교했을 때 Wi-Fi Direct는 훨씬 빠른 전송 속도와 긴 도달 범위를 제공합니다. NFC가 태그 기반 근접 통신에 특화된 반면, Wi-Fi Direct는 대용량 파일 전송, 멀티플레이어 게임, 화면 미러링 같은 고속 통신 시나리오에 적합합니다.

### Wi-Fi Direct가 필요한 상황

| 상황 | 추천 기술 |
|------|-----------|
| 대용량 파일(사진·영상) 공유 | **Wi-Fi Direct** |
| 멀티플레이어 모바일 게임 | **Wi-Fi Direct** |
| 소량 데이터 교환 | Bluetooth / NFC |
| 인터넷을 통한 원격 통신 | MQTT / WebSocket |

---

## 왜 Wi-Fi Direct인가? — Bluetooth 대비 장점

Bluetooth는 연결 설정 과정이 간단하지만, 최대 전송 속도가 Bluetooth 5.x 기준 약 2 Mbps에 불과합니다. Wi-Fi Direct는 WPA2 암호화를 기본 지원하면서도 최대 250 Mbps 이상의 속도를 제공합니다.

또한 Wi-Fi Direct는 **그룹 오너(Group Owner, GO)** 개념을 도입해 하나의 기기가 소프트 액세스 포인트 역할을 맡고 나머지 기기가 클라이언트로 접속하는 구조를 만듭니다. Android 시스템이 각 기기의 전력·UI·서비스 역량을 평가해 최적의 GO를 자동으로 선정하므로, 개발자가 직접 역할을 지정할 필요가 없습니다.

---

## 핵심 아키텍처

```
App ──→ WifiP2pManager
           │
           ├── initialize() → Channel (앱-프레임워크 통신 채널)
           ├── discoverPeers(Channel, ActionListener)
           ├── connect(Channel, WifiP2pConfig, ActionListener)
           └── requestConnectionInfo(Channel, ConnectionInfoListener)

System ──→ BroadcastReceiver
              ├── WIFI_P2P_STATE_CHANGED_ACTION   (P2P 활성화 여부)
              ├── WIFI_P2P_PEERS_CHANGED_ACTION   (피어 목록 변경)
              ├── WIFI_P2P_CONNECTION_CHANGED_ACTION (연결 상태 변경)
              └── WIFI_P2P_THIS_DEVICE_CHANGED_ACTION (내 기기 정보 변경)
```

전체 흐름은 **초기화 → 피어 탐색 → 연결 → 데이터 전송** 4단계로 구성됩니다.

---

## 권한 선언 (AndroidManifest.xml)

```xml
<!-- API 33+ 필수 -->
<uses-permission android:name="android.permission.NEARBY_WIFI_DEVICES"
    android:usesPermissionFlags="neverForLocation" />

<!-- API 32 이하 필수 -->
<uses-permission
    android:name="android.permission.ACCESS_FINE_LOCATION"
    android:maxSdkVersion="32" />

<uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
<uses-permission android:name="android.permission.CHANGE_WIFI_STATE" />
<uses-permission android:name="android.permission.CHANGE_NETWORK_STATE" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
<uses-permission android:name="android.permission.INTERNET" />
```

> **API 33 이상**: `discoverPeers()`·`connect()`·`requestPeers()` 호출 전 반드시 `NEARBY_WIFI_DEVICES` 런타임 권한을 획득해야 합니다. 위치 정보를 Wi-Fi API로 추론하지 않는다면 `usesPermissionFlags="neverForLocation"`을 추가해 위치 권한을 별도로 요청하지 않아도 됩니다.

---

## 코드 예제 1 — 초기화·피어 탐색·연결

아래 코드는 `Activity` 하나에 초기화, `BroadcastReceiver` 등록, 피어 탐색, 연결 요청까지 전 과정을 담았습니다.

```kotlin
class WifiDirectActivity : AppCompatActivity() {

    private val TAG = "WifiDirect"

    // WifiP2pManager와 Channel은 Activity 생명주기보다 길게 살아야 합니다.
    private val manager: WifiP2pManager by lazy {
        getSystemService(Context.WIFI_P2P_SERVICE) as WifiP2pManager
    }
    private lateinit var channel: WifiP2pManager.Channel
    private var receiver: BroadcastReceiver? = null

    private val peers = mutableListOf<WifiP2pDevice>()

    // 발견된 피어 목록 갱신 콜백
    private val peerListListener = WifiP2pManager.PeerListListener { peerList ->
        val refreshed = peerList.deviceList.toList()
        if (refreshed != peers) {
            peers.clear()
            peers.addAll(refreshed)
            Log.d(TAG, "Peers updated: ${peers.size} device(s) found")
        }
        if (peers.isEmpty()) Log.d(TAG, "No devices found")
    }

    // 연결 정보 콜백 — 그룹 오너 여부에 따라 서버/클라이언트 역할 결정
    private val connectionListener = WifiP2pManager.ConnectionInfoListener { info ->
        val ownerAddress = info.groupOwnerAddress?.hostAddress ?: return@ConnectionInfoListener
        Log.d(TAG, "Group owner address: $ownerAddress, isOwner: ${info.isGroupOwner}")

        if (info.groupFormed && info.isGroupOwner) {
            // 이 기기가 그룹 오너(서버 역할)
            startFileServer()
        } else if (info.groupFormed) {
            // 이 기기가 클라이언트 역할
            connectToServer(ownerAddress)
        }
    }

    // BroadcastReceiver: 시스템 P2P 이벤트를 수신
    private inner class WiFiDirectBroadcastReceiver : BroadcastReceiver() {
        override fun onReceive(context: Context, intent: Intent) {
            when (intent.action) {
                WifiP2pManager.WIFI_P2P_STATE_CHANGED_ACTION -> {
                    val state = intent.getIntExtra(WifiP2pManager.EXTRA_WIFI_STATE, -1)
                    val enabled = state == WifiP2pManager.WIFI_P2P_STATE_ENABLED
                    Log.d(TAG, "Wi-Fi P2P enabled: $enabled")
                }
                WifiP2pManager.WIFI_P2P_PEERS_CHANGED_ACTION -> {
                    // 피어 목록이 변경되면 최신 목록을 요청
                    manager.requestPeers(channel, peerListListener)
                }
                WifiP2pManager.WIFI_P2P_CONNECTION_CHANGED_ACTION -> {
                    // Android 10+ 에서는 non-sticky이므로 직접 requestConnectionInfo 호출
                    val networkInfo = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
                        manager.requestConnectionInfo(channel, connectionListener)
                        null
                    } else {
                        @Suppress("DEPRECATION")
                        intent.getParcelableExtra<NetworkInfo>(WifiP2pManager.EXTRA_NETWORK_INFO)
                    }
                    if (networkInfo?.isConnected == true) {
                        manager.requestConnectionInfo(channel, connectionListener)
                    }
                }
                WifiP2pManager.WIFI_P2P_THIS_DEVICE_CHANGED_ACTION -> {
                    // 내 기기 정보 변경 처리 (Android 10+ non-sticky)
                }
            }
        }
    }

    private val intentFilter = IntentFilter().apply {
        addAction(WifiP2pManager.WIFI_P2P_STATE_CHANGED_ACTION)
        addAction(WifiP2pManager.WIFI_P2P_PEERS_CHANGED_ACTION)
        addAction(WifiP2pManager.WIFI_P2P_CONNECTION_CHANGED_ACTION)
        addAction(WifiP2pManager.WIFI_P2P_THIS_DEVICE_CHANGED_ACTION)
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // Channel 초기화: 앱과 Wi-Fi P2P 프레임워크 사이의 통신 채널
        channel = manager.initialize(this, mainLooper, null)
    }

    override fun onResume() {
        super.onResume()
        receiver = WiFiDirectBroadcastReceiver()
        registerReceiver(receiver, intentFilter)
    }

    override fun onPause() {
        super.onPause()
        receiver?.let { unregisterReceiver(it) }
        receiver = null
    }

    // 피어 탐색 시작
    fun startDiscovery() {
        // API 33+: NEARBY_WIFI_DEVICES 권한 확인 후 호출해야 합니다.
        manager.discoverPeers(channel, object : WifiP2pManager.ActionListener {
            override fun onSuccess() {
                Log.d(TAG, "Discovery initiated successfully")
            }
            override fun onFailure(reason: Int) {
                Log.e(TAG, "Discovery failed: reason=$reason")
            }
        })
    }

    // 첫 번째 피어에 연결
    fun connectToFirstPeer() {
        if (peers.isEmpty()) {
            Log.w(TAG, "No peers available")
            return
        }
        val device = peers[0]
        val config = WifiP2pConfig().apply {
            deviceAddress = device.deviceAddress
            wps.setup = WpsInfo.PBC   // Push Button Configuration — 가장 일반적인 방식
        }
        manager.connect(channel, config, object : WifiP2pManager.ActionListener {
            override fun onSuccess() {
                Log.d(TAG, "Connection initiated to ${device.deviceName}")
            }
            override fun onFailure(reason: Int) {
                Log.e(TAG, "Connection failed: reason=$reason")
            }
        })
    }

    private fun startFileServer() { /* 예제 2 참고 */ }
    private fun connectToServer(ownerAddress: String) { /* 예제 2 참고 */ }
}
```

---

## 코드 예제 2 — 소켓 기반 파일 전송 (서버·클라이언트)

P2P 연결이 완료되면 `WifiP2pInfo`에서 그룹 오너 IP를 얻어 TCP 소켓으로 직접 통신합니다. 그룹 오너는 `ServerSocket`으로 대기하고, 나머지 기기는 클라이언트 `Socket`으로 접속합니다.

```kotlin
// 그룹 오너(서버) 역할 — 파일 수신
fun startFileServer(port: Int = 8988, onReceived: (File) -> Unit) {
    // 네트워크 작업은 반드시 백그라운드 스레드에서 수행해야 합니다.
    CoroutineScope(Dispatchers.IO).launch {
        var serverSocket: ServerSocket? = null
        try {
            serverSocket = ServerSocket(port)
            Log.d("WifiDirect", "Server: waiting for connection on port $port")

            // accept()는 클라이언트 연결까지 블로킹됩니다.
            val client = serverSocket.accept()
            Log.d("WifiDirect", "Server: client connected from ${client.inetAddress.hostAddress}")

            val outputDir = File(getExternalFilesDir(null), "wifip2p")
            outputDir.mkdirs()
            val outputFile = File(outputDir, "received_${System.currentTimeMillis()}.dat")

            // 스트림으로 데이터 수신
            client.getInputStream().use { input ->
                outputFile.outputStream().use { output ->
                    input.copyTo(output, bufferSize = 8192)
                }
            }
            Log.d("WifiDirect", "Server: file saved to ${outputFile.absolutePath}")

            withContext(Dispatchers.Main) { onReceived(outputFile) }
        } catch (e: IOException) {
            Log.e("WifiDirect", "Server error: ${e.message}")
        } finally {
            serverSocket?.close()
        }
    }
}

// 클라이언트 역할 — 파일 송신
fun sendFile(
    groupOwnerAddress: String,
    fileUri: Uri,
    port: Int = 8988,
    onComplete: (Boolean) -> Unit
) {
    CoroutineScope(Dispatchers.IO).launch {
        val socket = Socket()
        var success = false
        try {
            // 소켓 바인딩 후 그룹 오너 IP:포트로 연결 (500ms 타임아웃)
            socket.bind(null)
            socket.connect(InetSocketAddress(groupOwnerAddress, port), 5000)
            Log.d("WifiDirect", "Client: connected to $groupOwnerAddress:$port")

            // ContentResolver로 Uri에서 InputStream 획득 → 소켓 OutputStream으로 전송
            contentResolver.openInputStream(fileUri)?.use { fileStream ->
                socket.getOutputStream().use { socketStream ->
                    fileStream.copyTo(socketStream, bufferSize = 8192)
                }
            }
            success = true
            Log.d("WifiDirect", "Client: file sent successfully")
        } catch (e: IOException) {
            Log.e("WifiDirect", "Client error: ${e.message}")
        } finally {
            socket.close()
            withContext(Dispatchers.Main) { onComplete(success) }
        }
    }
}

// 사용 예시
// connectionListener 내부에서 호출
val connectionListener = WifiP2pManager.ConnectionInfoListener { info ->
    val ownerAddress = info.groupOwnerAddress?.hostAddress ?: return@ConnectionInfoListener
    if (info.groupFormed && info.isGroupOwner) {
        startFileServer { file ->
            Toast.makeText(this, "파일 수신 완료: ${file.name}", Toast.LENGTH_SHORT).show()
        }
    } else if (info.groupFormed) {
        val fileUri = Uri.parse("content://...") // 전송할 파일 URI
        sendFile(ownerAddress, fileUri) { success ->
            Toast.makeText(this, if (success) "전송 성공" else "전송 실패", Toast.LENGTH_SHORT).show()
        }
    }
}
```

---

## 연결 해제 및 정리

Wi-Fi P2P 그룹을 명시적으로 해제하지 않으면 다른 앱이 Wi-Fi를 사용할 때 간섭이 발생합니다. `onDestroy()`나 작업 완료 후 반드시 그룹을 제거해야 합니다.

```kotlin
fun disconnect() {
    manager.removeGroup(channel, object : WifiP2pManager.ActionListener {
        override fun onSuccess() = Log.d("WifiDirect", "P2P group removed")
        override fun onFailure(reason: Int) = Log.e("WifiDirect", "Remove group failed: $reason")
    })
    // 필요 시 채널도 닫기 (API 27+)
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O_MR1) {
        channel.close()
    }
}
```

---

## 주의사항 및 실전 팁

### 1. Android 10(Q) 이후 Non-sticky 브로드캐스트
`WIFI_P2P_CONNECTION_CHANGED_ACTION`과 `WIFI_P2P_THIS_DEVICE_CHANGED_ACTION`은 Android 10부터 non-sticky입니다. BroadcastReceiver 등록 시점에 이미 연결된 상태라면 인텐트를 받지 못합니다. Activity 시작 시 `requestConnectionInfo()`를 직접 호출해 현재 상태를 가져오도록 초기화 로직을 작성하세요.

### 2. `discoverPeers()` 성공은 발견 완료가 아님
`ActionListener.onSuccess()` 콜백은 탐색 *프로세스* 시작이 성공했음을 의미할 뿐, 실제 피어를 찾았다는 의미가 아닙니다. 실제 피어 목록은 `WIFI_P2P_PEERS_CHANGED_ACTION` 인텐트 수신 후 `requestPeers()`로 가져옵니다.

### 3. 위치 모드 활성화 필요
`discoverPeers()`, `discoverServices()`, `requestPeers()`는 기기의 위치 모드가 켜져 있어야 정상 동작합니다. 위치 모드가 꺼진 경우 `ERROR` 콜백이 반환되며, 사용자에게 위치 설정 화면으로 안내해야 합니다.

```kotlin
// 위치 모드 확인 (API 28+)
val lm = getSystemService(LOCATION_SERVICE) as LocationManager
val locationEnabled = LocationManagerCompat.isLocationEnabled(lm)
if (!locationEnabled) {
    startActivity(Intent(Settings.ACTION_LOCATION_SOURCE_SETTINGS))
}
```

### 4. 모든 기기가 Wi-Fi Direct를 지원하지 않을 수 있음
저가형 기기나 일부 태블릿은 Wi-Fi Direct 하드웨어를 지원하지 않습니다. `PackageManager.hasSystemFeature(PackageManager.FEATURE_WIFI_DIRECT)`로 지원 여부를 반드시 확인하세요.

```kotlin
val supportsP2P = packageManager.hasSystemFeature(PackageManager.FEATURE_WIFI_DIRECT)
if (!supportsP2P) {
    Log.w("WifiDirect", "This device does not support Wi-Fi Direct")
}
```

### 5. 그룹 오너 의도(groupOwnerIntent) 설정
`WifiP2pConfig.groupOwnerIntent`를 0~15 범위로 설정해 이 기기가 그룹 오너가 되려는 선호도를 조절할 수 있습니다. 15가 가장 높은 선호도이고, -1(기본값)은 시스템이 자동으로 결정합니다. 서버로 동작해야 하는 기기라면 15로 설정해 GO가 될 확률을 높이는 것이 좋습니다.

```kotlin
val config = WifiP2pConfig().apply {
    deviceAddress = device.deviceAddress
    wps.setup = WpsInfo.PBC
    groupOwnerIntent = 15 // 이 기기를 그룹 오너로 지정 선호
}
```

### 6. 배터리 소모
Wi-Fi Direct 탐색은 Wi-Fi 하드웨어를 지속적으로 활성화하므로 배터리를 많이 소모합니다. 필요할 때만 `discoverPeers()`를 호출하고, 연결 완료 후 또는 사용 완료 후에는 반드시 `removeGroup()`을 호출해 탐색을 중단하세요.

### 7. 소켓 포트 충돌 방지
`ServerSocket` 포트를 1024 이상의 비표준 포트로 선택하고, 앱 내에서 상수로 관리하세요. 여러 Wi-Fi P2P 기능을 동시에 사용한다면 각 기능마다 다른 포트를 할당합니다.

---

## 전체 흐름 정리

```
1. initialize()          → Channel 획득
2. BroadcastReceiver 등록
3. discoverPeers()       → 탐색 시작
4. PEERS_CHANGED 수신    → requestPeers() → WifiP2pDeviceList
5. connect(config)       → 연결 시작
6. CONNECTION_CHANGED 수신 → requestConnectionInfo()
7. ConnectionInfoListener  → isGroupOwner 판별
   ├── true  → ServerSocket 대기 (서버)
   └── false → Socket.connect(groupOwnerAddress) (클라이언트)
8. 데이터 전송 (Stream I/O)
9. removeGroup()         → 그룹 해제 및 정리
```

Wi-Fi Direct는 설정이 다소 복잡하지만, 중간 서버 없이 기기 간 고속 통신이 가능하다는 점에서 파일 공유 앱, 로컬 멀티플레이어 게임, 오프라인 협업 도구에 매우 유용합니다. Android 10 이후 non-sticky 브로드캐스트 변경, API 33 권한 체계 변화에 주의하면서 구현하면 견고한 P2P 통신 기능을 완성할 수 있습니다.

---

## 참고 자료
- [Wi-Fi Direct (P2P) 개요 — Android Developers](https://developer.android.com/develop/connectivity/wifi/wifip2p)
- [P2P 연결 생성 가이드 — Android Developers](https://developer.android.com/develop/connectivity/wifi/wifi-direct)
- [WifiP2pManager API Reference](https://developer.android.com/reference/kotlin/android/net/wifi/p2p/WifiP2pManager)
- [Wi-Fi Direct Demo 샘플 (AOSP)](https://android.googlesource.com/platform/development/+/master/samples/WiFiDirectDemo/)
