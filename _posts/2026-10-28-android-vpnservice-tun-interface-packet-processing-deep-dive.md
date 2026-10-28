---
layout: post
title: "Android VpnService 심화: TUN 인터페이스와 패킷 처리 완전 분석"
date: 2026-10-28
categories: [android]
tags: [android, vpn, vpnservice, tun, networking, kotlin, security]
---

Android에서 VPN 앱을 직접 개발하려면 운영체제 수준의 네트워크 스택 이해가 필수입니다. `VpnService` API는 단순한 네트워크 연결을 넘어, TUN(Tunnel) 가상 인터페이스를 통해 디바이스의 모든 IP 패킷을 앱 레벨에서 가로채고 처리할 수 있는 강력한 메커니즘을 제공합니다. 이 글에서는 VPN 앱의 내부 동작 원리부터 실제 TUN 인터페이스를 활용한 패킷 처리까지 심층적으로 다룹니다.

## VpnService란 무엇인가

`VpnService`는 Android 4.0 (API 14)부터 제공되는 추상 서비스 클래스로, 앱이 가상 네트워크 인터페이스(TUN 인터페이스)를 생성하고 시스템의 IP 라우팅 테이블을 수정하여 트래픽을 가로챌 수 있게 합니다.

### TUN 인터페이스의 동작 원리

TUN(TUNnel) 인터페이스는 Linux 커널의 네트워크 스택에서 동작하는 가상 네트워크 장치입니다. 물리적 이더넷이나 Wi-Fi 인터페이스와 달리, TUN 인터페이스는 파일 디스크립터로 표현되며 유저스페이스 프로세스가 직접 IP 패킷을 읽고 쓸 수 있습니다.

```
[앱] → [TUN fd.write()] → [커널 TUN 인터페이스] → [시스템 IP 스택] → [앱의 소켓]
[앱 소켓] → [시스템 IP 스택] → [TUN 인터페이스] → [TUN fd.read()] → [VPN 서비스]
```

`VpnService.Builder.establish()`를 호출하면 시스템은:
1. `tun0` (또는 유사한) 가상 인터페이스를 생성
2. 지정된 IP 주소와 라우팅 규칙을 적용
3. 해당 인터페이스에 대응하는 파일 디스크립터(`ParcelFileDescriptor`)를 앱에 반환

이후 라우팅 규칙에 해당하는 모든 아웃바운드 패킷은 TUN 인터페이스로 전달되고, VPN 서비스는 이를 `FileInputStream`으로 읽어 처리할 수 있습니다.

## 왜 VpnService가 필요한가

### 1. 프로토콜 다양성

Android의 내장 VPN 클라이언트는 IPSec, L2TP, PPTP, IKEv2만 지원합니다. WireGuard, OpenVPN, VLESS, Shadowsocks 같은 현대적인 프로토콜을 사용하려면 `VpnService`를 활용한 커스텀 구현이 필수입니다.

### 2. 트래픽 필터링과 모니터링

광고 차단, 콘텐츠 필터링, 데이터 사용량 추적 앱들은 `VpnService`를 이용해 DNS 쿼리를 가로채거나 TCP/UDP 패킷을 분석합니다. 루트 권한 없이 패킷 수준 처리가 가능한 유일한 방법입니다.

### 3. 기업 보안 솔루션

BYOD(Bring Your Own Device) 환경에서 특정 앱만 기업 네트워크로 라우팅하는 Per-App VPN, 네트워크 정책 강제 적용, 트래픽 감사 로그 등 엔터프라이즈 기능 구현에 필수적입니다.

### 4. Always-On VPN (Android 7.0+)

Android 7.0부터 시스템 레벨에서 VPN을 항상 활성화하고, VPN 연결이 끊어지면 모든 네트워크 트래픽을 차단하는 Lockdown 모드를 지원합니다.

## 필수 설정: Manifest와 권한

VPN 서비스를 사용하려면 반드시 `BIND_VPN_SERVICE` 퍼미션이 필요하며, 이는 시스템만이 서비스에 바인드할 수 있도록 제한합니다.

```xml
<!-- AndroidManifest.xml -->
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_SPECIAL_USE" />

<service
    android:name=".vpn.MyVpnService"
    android:permission="android.permission.BIND_VPN_SERVICE"
    android:exported="true"
    android:foregroundServiceType="specialUse">
    <intent-filter>
        <action android:name="android.net.VpnService" />
    </intent-filter>
    <meta-data
        android:name="android.net.VpnService.SUPPORTS_ALWAYS_ON"
        android:value="true" />
</service>
```

Android 14 (API 34)부터는 포그라운드 서비스 타입에 `specialUse`를 명시해야 하며, Play Store 심사를 위한 별도 선언이 필요합니다.

## 실제 구현 예제 1: VPN 서비스 기초 구조

다음은 TUN 인터페이스를 생성하고 패킷을 읽어 처리하는 완전한 VPN 서비스의 기초 구조입니다.

```kotlin
import android.net.VpnService
import android.os.ParcelFileDescriptor
import kotlinx.coroutines.*
import java.io.FileInputStream
import java.io.FileOutputStream
import java.net.DatagramSocket
import java.net.InetAddress
import java.nio.ByteBuffer

class MyVpnService : VpnService() {

    private var vpnInterface: ParcelFileDescriptor? = null
    private val serviceScope = CoroutineScope(Dispatchers.IO + SupervisorJob())
    private var tunnelSocket: DatagramSocket? = null

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        when (intent?.action) {
            ACTION_START -> startVpn(intent)
            ACTION_STOP -> stopVpn()
        }
        return START_STICKY
    }

    private fun startVpn(intent: Intent) {
        val serverAddress = intent.getStringExtra(EXTRA_SERVER_ADDRESS) ?: return
        val serverPort = intent.getIntExtra(EXTRA_SERVER_PORT, 8388)

        // 포그라운드 서비스 알림 시작 (Android 8.0+ 필수)
        startForeground(NOTIFICATION_ID, buildNotification())

        serviceScope.launch {
            try {
                establishTunnel(serverAddress, serverPort)
            } catch (e: Exception) {
                stopSelf()
            }
        }
    }

    private suspend fun establishTunnel(serverAddress: String, serverPort: Int) {
        // 1. UDP 소켓으로 VPN 게이트웨이에 연결
        val socket = DatagramSocket()
        tunnelSocket = socket

        // 2. 소켓을 protect()로 VPN 루프백 방지
        // protect()를 호출하지 않으면 이 소켓의 트래픽도 TUN으로 들어가 무한루프 발생
        if (!protect(socket)) {
            throw IllegalStateException("소켓 protect() 실패 - VPN 퍼미션을 확인하세요")
        }

        socket.connect(InetAddress.getByName(serverAddress), serverPort)

        // 3. TUN 인터페이스 구성 및 생성
        val builder = Builder()
            .setSession("MyVPN")
            .addAddress("10.0.0.2", 24)           // 가상 인터페이스 IP
            .addRoute("0.0.0.0", 0)               // 모든 IPv4 트래픽 라우팅
            .addRoute("::", 0)                     // 모든 IPv6 트래픽 라우팅
            .addDnsServer("1.1.1.1")              // Cloudflare DNS
            .addDnsServer("8.8.8.8")              // Google DNS
            .setMtu(1500)
            .setBlocking(false)                    // 논블로킹 모드
            .setConfigureIntent(buildConfigIntent())

        // Per-App VPN: 특정 앱만 터널 사용
        // builder.addAllowedApplication("com.example.targetapp")

        vpnInterface = builder.establish()
            ?: throw IllegalStateException("TUN 인터페이스 생성 실패")

        // 4. 패킷 처리 루프 시작
        runPacketLoop(socket)
    }

    private suspend fun runPacketLoop(socket: DatagramSocket) {
        val tunFd = vpnInterface ?: return
        val inputStream = FileInputStream(tunFd.fileDescriptor)
        val outputStream = FileOutputStream(tunFd.fileDescriptor)
        val packet = ByteArray(32767)

        // Inbound(VPN 게이트웨이 → 디바이스)와 Outbound(디바이스 → VPN 게이트웨이)를
        // 별도 코루틴으로 처리
        val outboundJob = serviceScope.launch {
            processOutbound(inputStream, socket, packet)
        }
        val inboundJob = serviceScope.launch {
            processInbound(socket, outputStream)
        }

        outboundJob.join()
        inboundJob.join()
    }

    // 아웃바운드: TUN에서 읽어 VPN 서버로 전송
    private suspend fun processOutbound(
        input: FileInputStream,
        socket: DatagramSocket,
        buffer: ByteArray
    ) = withContext(Dispatchers.IO) {
        val packetBuffer = DatagramPacket(buffer, buffer.size)

        while (isActive) {
            val length = input.read(buffer)
            if (length <= 0) {
                delay(1) // CPU 과부하 방지
                continue
            }

            // 실제 구현에서는 여기서 암호화 수행
            val encryptedData = encryptPacket(buffer, length)

            packetBuffer.data = encryptedData
            packetBuffer.length = encryptedData.size
            packetBuffer.address = socket.inetAddress
            packetBuffer.port = socket.port

            try {
                socket.send(packetBuffer)
            } catch (e: Exception) {
                // 연결 실패 시 재연결 로직
                break
            }
        }
    }

    // 인바운드: VPN 서버에서 수신하여 TUN에 쓰기
    private suspend fun processInbound(
        socket: DatagramSocket,
        output: FileOutputStream
    ) = withContext(Dispatchers.IO) {
        val buffer = ByteArray(32767)
        val packetBuffer = DatagramPacket(buffer, buffer.size)

        while (isActive) {
            try {
                socket.receive(packetBuffer)
                val decryptedData = decryptPacket(
                    buffer,
                    packetBuffer.length
                )
                output.write(decryptedData)
            } catch (e: Exception) {
                break
            }
        }
    }

    override fun onRevoke() {
        // 사용자가 VPN을 비활성화하거나 다른 VPN이 시작될 때 호출
        stopVpn()
    }

    private fun stopVpn() {
        serviceScope.cancel()
        tunnelSocket?.close()
        vpnInterface?.close()
        vpnInterface = null
        stopForeground(STOP_FOREGROUND_REMOVE)
        stopSelf()
    }

    override fun onDestroy() {
        stopVpn()
        super.onDestroy()
    }

    // 스텁 메서드 (실제 구현 필요)
    private fun encryptPacket(data: ByteArray, length: Int): ByteArray =
        data.copyOf(length)

    private fun decryptPacket(data: ByteArray, length: Int): ByteArray =
        data.copyOf(length)

    companion object {
        const val ACTION_START = "action_start"
        const val ACTION_STOP = "action_stop"
        const val EXTRA_SERVER_ADDRESS = "server_address"
        const val EXTRA_SERVER_PORT = "server_port"
        const val NOTIFICATION_ID = 1001
    }
}
```

## 실제 구현 예제 2: IP 패킷 파서 및 DNS 필터링

VPN 앱에서 광고 차단이나 DNS 필터링을 구현하려면 IP 패킷 구조를 직접 파싱해야 합니다. 다음은 IPv4 헤더와 UDP DNS 쿼리를 파싱하는 구현입니다.

```kotlin
import java.nio.ByteBuffer
import java.nio.ByteOrder

/**
 * IPv4 패킷 파서
 * RFC 791 기반 IPv4 헤더 파싱
 */
data class IPv4Header(
    val version: Int,           // IP 버전 (4)
    val ihl: Int,               // 헤더 길이 (32비트 워드 단위)
    val tos: Int,               // 서비스 타입
    val totalLength: Int,       // 전체 패킷 길이
    val identification: Int,    // 단편화 식별자
    val flags: Int,             // 단편화 플래그
    val fragmentOffset: Int,    // 단편화 오프셋
    val ttl: Int,               // Time to Live
    val protocol: Int,          // 프로토콜 (6=TCP, 17=UDP, 1=ICMP)
    val headerChecksum: Int,    // 헤더 체크섬
    val sourceAddress: ByteArray,      // 출발지 IP
    val destinationAddress: ByteArray  // 목적지 IP
) {
    val headerLength: Int get() = ihl * 4

    val sourceIp: String get() =
        sourceAddress.joinToString(".") { (it.toInt() and 0xFF).toString() }

    val destinationIp: String get() =
        destinationAddress.joinToString(".") { (it.toInt() and 0xFF).toString() }

    companion object {
        const val PROTOCOL_ICMP = 1
        const val PROTOCOL_TCP = 6
        const val PROTOCOL_UDP = 17

        fun parse(buffer: ByteBuffer): IPv4Header? {
            if (buffer.remaining() < 20) return null

            buffer.order(ByteOrder.BIG_ENDIAN)
            val versionAndIhl = buffer.get().toInt() and 0xFF
            val version = versionAndIhl shr 4
            val ihl = versionAndIhl and 0x0F

            if (version != 4 || ihl < 5) return null

            val tos = buffer.get().toInt() and 0xFF
            val totalLength = buffer.short.toInt() and 0xFFFF
            val identification = buffer.short.toInt() and 0xFFFF
            val flagsAndOffset = buffer.short.toInt() and 0xFFFF
            val flags = flagsAndOffset shr 13
            val fragmentOffset = flagsAndOffset and 0x1FFF
            val ttl = buffer.get().toInt() and 0xFF
            val protocol = buffer.get().toInt() and 0xFF
            val headerChecksum = buffer.short.toInt() and 0xFFFF
            val sourceAddress = ByteArray(4).also { buffer.get(it) }
            val destinationAddress = ByteArray(4).also { buffer.get(it) }

            // 추가 옵션 필드 건너뛰기
            val optionsLength = (ihl - 5) * 4
            if (optionsLength > 0) {
                buffer.position(buffer.position() + optionsLength)
            }

            return IPv4Header(
                version, ihl, tos, totalLength, identification,
                flags, fragmentOffset, ttl, protocol, headerChecksum,
                sourceAddress, destinationAddress
            )
        }
    }
}

/**
 * DNS 패킷 분석기 - 광고 차단 구현 예시
 */
class DnsFilter(private val blocklist: Set<String>) {

    data class DnsQuery(
        val transactionId: Int,
        val domain: String,
        val queryType: Int  // 1=A, 28=AAAA, 5=CNAME 등
    )

    /**
     * UDP 페이로드에서 DNS 쿼리 도메인을 추출합니다.
     * DNS 패킷 형식: RFC 1035 준수
     */
    fun parseDnsQuery(udpPayload: ByteArray): DnsQuery? {
        if (udpPayload.size < 12) return null

        val buffer = ByteBuffer.wrap(udpPayload).order(ByteOrder.BIG_ENDIAN)
        val transactionId = buffer.short.toInt() and 0xFFFF
        val flags = buffer.short.toInt() and 0xFFFF

        // QR 비트: 0=쿼리, 1=응답
        if (flags and 0x8000 != 0) return null

        val questionCount = buffer.short.toInt() and 0xFFFF
        if (questionCount == 0) return null

        // Answer, Authority, Additional RR 카운트 건너뛰기
        buffer.position(buffer.position() + 6)

        // QNAME 파싱 (레이블 인코딩)
        val domainBuilder = StringBuilder()
        while (buffer.hasRemaining()) {
            val labelLength = buffer.get().toInt() and 0xFF
            if (labelLength == 0) break  // 루트 레이블

            if (domainBuilder.isNotEmpty()) domainBuilder.append('.')

            repeat(labelLength) {
                domainBuilder.append(buffer.get().toChar())
            }
        }

        if (!buffer.hasRemaining()) return null
        val queryType = buffer.short.toInt() and 0xFFFF

        return DnsQuery(transactionId, domainBuilder.toString(), queryType)
    }

    /**
     * DNS 쿼리를 검사하여 차단 여부를 반환합니다.
     * 차단 대상이면 NXDOMAIN(3) 응답 패킷을 생성합니다.
     */
    fun shouldBlock(query: DnsQuery): Boolean {
        val domain = query.domain.lowercase()
        return blocklist.any { blocked ->
            domain == blocked || domain.endsWith(".$blocked")
        }
    }

    /**
     * NXDOMAIN 응답 패킷 생성 (차단 응답)
     */
    fun createBlockResponse(originalQuery: ByteArray, transactionId: Int): ByteArray {
        val response = ByteArray(originalQuery.size)
        originalQuery.copyInto(response)

        val buffer = ByteBuffer.wrap(response)
        buffer.order(ByteOrder.BIG_ENDIAN)

        // Transaction ID 유지
        buffer.position(2)

        // Flags: QR=1(응답), OPCODE=0, AA=0, TC=0, RD=1, RA=1, RCODE=3(NXDOMAIN)
        buffer.putShort(0x8183.toShort())

        return response
    }
}

/**
 * VPN 서비스에서 DNS 필터링 통합 예시
 */
class FilteringVpnService : VpnService() {

    private val dnsFilter = DnsFilter(
        blocklist = setOf(
            "ads.example.com",
            "tracker.analytics.io",
            "doubleclick.net"
        )
    )

    fun processPacket(rawPacket: ByteArray, length: Int): ProcessResult {
        val buffer = ByteBuffer.wrap(rawPacket, 0, length)
        val ipHeader = IPv4Header.parse(buffer) ?: return ProcessResult.Forward(rawPacket, length)

        // UDP 패킷이고 목적지 포트가 53(DNS)인 경우
        if (ipHeader.protocol == IPv4Header.PROTOCOL_UDP) {
            val udpBuffer = ByteBuffer.wrap(rawPacket, ipHeader.headerLength, length - ipHeader.headerLength)
            udpBuffer.order(ByteOrder.BIG_ENDIAN)

            val srcPort = udpBuffer.short.toInt() and 0xFFFF
            val dstPort = udpBuffer.short.toInt() and 0xFFFF
            val udpLength = udpBuffer.short.toInt() and 0xFFFF
            udpBuffer.short // checksum skip

            if (dstPort == 53) {
                val dnsPayload = ByteArray(udpLength - 8)
                udpBuffer.get(dnsPayload)

                val dnsQuery = dnsFilter.parseDnsQuery(dnsPayload)
                if (dnsQuery != null && dnsFilter.shouldBlock(dnsQuery)) {
                    // 차단 응답 생성 및 TUN에 직접 주입
                    val blockResponse = dnsFilter.createBlockResponse(dnsPayload, dnsQuery.transactionId)
                    return ProcessResult.Block(buildDnsResponse(rawPacket, ipHeader, srcPort, blockResponse))
                }
            }
        }

        return ProcessResult.Forward(rawPacket, length)
    }

    sealed class ProcessResult {
        data class Forward(val data: ByteArray, val length: Int) : ProcessResult()
        data class Block(val syntheticResponse: ByteArray) : ProcessResult()
    }

    private fun buildDnsResponse(
        originalPacket: ByteArray,
        ipHeader: IPv4Header,
        clientPort: Int,
        dnsResponse: ByteArray
    ): ByteArray {
        // 응답 패킷 구성: IP 헤더 + UDP 헤더 + DNS 응답
        // 실제 구현에서는 IP/UDP 체크섬 재계산 필요
        val udpLength = 8 + dnsResponse.size
        val totalLength = ipHeader.headerLength + udpLength
        val response = ByteArray(totalLength)
        val buf = ByteBuffer.wrap(response).order(ByteOrder.BIG_ENDIAN)

        // IP 헤더 (출발지/목적지 주소 교환)
        buf.put(rawPacket, 0, ipHeader.headerLength)
        buf.position(12)
        buf.put(ipHeader.destinationAddress)  // 새 출발지 = 원래 목적지(DNS 서버)
        buf.put(ipHeader.sourceAddress)       // 새 목적지 = 원래 출발지(클라이언트)

        // IP 전체 길이 수정
        buf.position(2)
        buf.putShort(totalLength.toShort())

        // UDP 헤더
        buf.position(ipHeader.headerLength)
        buf.putShort(53.toShort())          // 출발지 포트: DNS
        buf.putShort(clientPort.toShort())  // 목적지 포트: 클라이언트
        buf.putShort(udpLength.toShort())
        buf.putShort(0)                     // 체크섬 (0=비활성화)

        // DNS 응답 데이터
        buf.put(dnsResponse)

        return response
    }
}
```

## Always-On VPN과 Lockdown 모드 구현

Android 7.0 (API 24)부터 지원하는 Always-On VPN은 디바이스 부팅 시 자동으로 VPN을 시작하고, 연결이 끊어지면 모든 네트워크 접근을 차단합니다.

```kotlin
// Always-On VPN 감지 및 재연결 처리
override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
    val isAlwaysOn = intent == null || intent.component == null

    if (isAlwaysOn) {
        // 시스템이 시작한 Always-On VPN
        // 저장된 설정으로 자동 연결 시도
        val savedConfig = loadSavedConfig()
        if (savedConfig != null) {
            startVpnWithConfig(savedConfig)
        } else {
            // 설정이 없으면 사용자에게 알림
            showConfigRequiredNotification()
            stopSelf()
        }
    }
    return START_STICKY
}
```

Always-On VPN 지원을 매니페스트에 선언하고, 불필요하면 비활성화할 수 있습니다:

```xml
<!-- Always-On 지원 선언 -->
<meta-data
    android:name="android.net.VpnService.SUPPORTS_ALWAYS_ON"
    android:value="true" />
```

## 주의사항 및 핵심 팁

### 1. protect() 호출 누락은 치명적

VPN 게이트웨이와 통신하는 소켓에 `protect()`를 호출하지 않으면, 해당 소켓의 트래픽도 TUN 인터페이스를 통해 라우팅되어 무한 루프가 발생합니다. `protect()`는 소켓을 VPN 라우팅 규칙에서 제외하여 실제 물리 인터페이스로 전송되게 합니다.

```kotlin
// 반드시 connect() 전에 protect() 호출
val socket = DatagramSocket()
if (!protect(socket)) {
    throw IllegalStateException("VPN protect() 실패")
}
socket.connect(serverAddress, serverPort)
```

### 2. VpnService.prepare() 사전 호출 필수

서비스 시작 전 Activity에서 반드시 사용자 동의를 얻어야 합니다:

```kotlin
// Activity에서
private val vpnPermissionLauncher = registerForActivityResult(
    ActivityResultContracts.StartActivityForResult()
) { result ->
    if (result.resultCode == RESULT_OK) {
        startVpnService()
    }
}

fun requestVpnPermission() {
    val intent = VpnService.prepare(this)
    if (intent != null) {
        // 사용자 동의 다이얼로그 표시
        vpnPermissionLauncher.launch(intent)
    } else {
        // 이미 권한 있음
        startVpnService()
    }
}
```

### 3. MTU 설정과 IP 단편화

TUN 인터페이스의 MTU를 물리 인터페이스의 MTU보다 작게 설정해야 암호화 헤더 오버헤드를 수용할 수 있습니다. 일반적으로 물리 MTU(1500)에서 암호화 헤더 크기를 뺀 값을 사용합니다.

```kotlin
// WireGuard 기준: 1500 - 60(IPv6) - 8(UDP) - 32(WireGuard) = 1400
builder.setMtu(1400)
```

### 4. 배터리와 성능 최적화

패킷 처리 루프에서 busy-waiting을 피하려면 `setBlocking(false)` 대신 `setBlocking(true)`로 설정하고 별도 스레드에서 블로킹 읽기를 수행하거나, NIO `Selector`를 활용하세요:

```kotlin
// 블로킹 모드 사용 (CPU 효율적)
builder.setBlocking(true)

// 전용 스레드에서 처리
thread(name = "vpn-outbound") {
    val buffer = ByteArray(32767)
    while (!Thread.interrupted()) {
        val length = inputStream.read(buffer) // 블로킹
        if (length > 0) processPacket(buffer, length)
    }
}
```

### 5. 연결 끊김 감지와 재연결

네트워크 전환(Wi-Fi → LTE)이나 절전 모드 후 재개 시 VPN 터널이 끊길 수 있습니다. `ConnectivityManager.NetworkCallback`과 함께 사용하면 자동 재연결 로직을 구현할 수 있습니다.

### 6. Android 14+ 보안 강화

Android 14(API 34)부터 포그라운드 서비스 타입 선언이 엄격하게 요구됩니다. `specialUse` 타입을 사용하면 Google Play 심사 과정에서 VPN 목적임을 명확히 설명해야 합니다. 사용자 데이터 보호 정책을 준수하고 개인정보 처리방침에 패킷 처리 내용을 반드시 명시해야 합니다.

### 7. TUN 인터페이스 재사용 (연결 복원)

`Builder.establish()`에 `allowFamilies()`, `allowRoutes()` 등을 동일하게 설정하면 `Builder` 재호출 시 기존 인터페이스를 재사용할 수 있어 트래픽 중단 없이 재연결이 가능합니다.

## 참고 자료

- [Android VPN 개발 가이드 (공식)](https://developer.android.com/develop/connectivity/vpn)
- [VpnService API 레퍼런스](https://developer.android.com/reference/android/net/VpnService)
