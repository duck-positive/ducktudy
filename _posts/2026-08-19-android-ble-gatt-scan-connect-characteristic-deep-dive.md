---
layout: post
title: "Android Bluetooth Low Energy(BLE) 심화: GATT 프로파일·스캔·연결·Characteristic 읽기·쓰기 완전 정복"
date: 2026-08-19
categories: [android, flutter]
tags: [android, bluetooth, ble, gatt, kotlin, iot, connectivity]
---

## 1. BLE란 무엇인가

Bluetooth Low Energy(BLE)는 스마트폰과 IoT 기기, 헬스케어 장치, 스마트홈 기기를 저전력으로 연결하는 무선 프로토콜입니다. 2010년 Bluetooth 4.0 스펙에 포함된 이후 Android는 API 18(Android 4.3)부터 BLE 중앙(Central) 역할을 지원했으며, API 21부터는 주변(Peripheral) 역할도 지원합니다.

### 핵심 용어: GATT, GAP, Service, Characteristic

BLE를 다루려면 프로토콜 계층 구조를 먼저 이해해야 합니다.

- **GAP(Generic Access Profile)**: 기기 발견과 연결 절차를 정의합니다. 광고(Advertising)와 스캔(Scanning) 동작이 여기에 속합니다.
- **GATT(Generic Attribute Profile)**: 연결된 BLE 기기 간 데이터 교환 방식을 정의합니다.
- **Profile**: 특정 사용 사례를 위한 서비스 집합(예: Heart Rate Profile, Battery Service Profile).
- **Service**: 논리적으로 연관된 Characteristic의 묶음. UUID로 식별됩니다.
- **Characteristic**: BLE 기기가 노출하는 데이터의 최소 단위. 값(Value), 속성(Properties: READ, WRITE, NOTIFY 등), 설명자(Descriptor)로 구성됩니다.

```
Android 앱(GATT Client) ←→ BLE 기기(GATT Server)
            └─ Service (UUID: 0x180D Heart Rate)
                 └─ Characteristic (UUID: 0x2A37)
                      ├─ Value (최대 512 bytes)
                      ├─ Properties (READ / WRITE / NOTIFY)
                      └─ Descriptor (CCCD 0x2902: 알림 활성화 토글)
```

---

## 2. 왜 BLE가 필요한가

현대 Android 앱에서 BLE가 요구되는 주요 시나리오는 다음과 같습니다.

- **헬스케어**: 혈당계, 혈압계, 산소포화도 측정기와 실시간 데이터 교환
- **스마트홈**: 조명 제어, 도어락, 온도·습도 센서 통신
- **웨어러블**: 스마트워치·이어폰과 앱 연동
- **산업 IoT**: 공장 센서 데이터 수집, 현장 장비 제어

Classic Bluetooth와 달리 BLE는 데이터 전송량이 적지만 연결을 짧게 끊고 이어 붙일 수 있어 배터리 소모가 극히 적습니다. 심박수 모니터처럼 초당 수 바이트만 전송하는 기기에 최적화된 방식입니다.

---

## 3. 권한 및 초기 설정

Android 12(API 31) 이상에서는 `BLUETOOTH_SCAN`, `BLUETOOTH_CONNECT` 권한을 런타임에 요청해야 합니다. API 30 이하에서는 스캔에 `ACCESS_FINE_LOCATION`이 필요합니다.

**`AndroidManifest.xml`**:
```xml
<!-- Android 12+ (API 31 이상) -->
<uses-permission android:name="android.permission.BLUETOOTH_SCAN"
    android:usesPermissionFlags="neverForLocation" />
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />

<!-- Android 11 이하 (API 30 이하) -->
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />

<!-- BLE 필수 기능 선언 -->
<uses-feature android:name="android.hardware.bluetooth_le" android:required="true" />
```

---

## 4. 실제 구현 예제

### 예제 1: BLE 스캔 — BluetoothLeScanner + ScanFilter

특정 서비스 UUID를 광고하는 기기만 필터링하여 스캔합니다. 불필요한 기기를 걸러내면 배터리와 콜백 부하를 크게 줄일 수 있습니다.

```kotlin
class BleScanner(private val context: Context) {

    private val bluetoothManager =
        context.getSystemService(Context.BLUETOOTH_SERVICE) as BluetoothManager
    private val bluetoothAdapter: BluetoothAdapter? = bluetoothManager.adapter
    private val scanner: BluetoothLeScanner? get() = bluetoothAdapter?.bluetoothLeScanner

    // 찾고자 하는 BLE 서비스 UUID (Heart Rate Service: 0x180D)
    private val targetServiceUuid = UUID.fromString("0000180D-0000-1000-8000-00805F9B34FB")

    private val scanCallback = object : ScanCallback() {
        override fun onScanResult(callbackType: Int, result: ScanResult) {
            val device = result.device
            val deviceName = result.scanRecord?.deviceName ?: "Unknown"
            val rssi = result.rssi
            Log.d("BLE", "발견: $deviceName (${device.address}), RSSI: $rssi dBm")
            // 원하는 기기를 찾으면 스캔을 중지하고 연결 시도
        }

        override fun onScanFailed(errorCode: Int) {
            Log.e("BLE", "스캔 실패: errorCode=$errorCode")
        }
    }

    @SuppressLint("MissingPermission")
    fun startScan() {
        val filters = listOf(
            ScanFilter.Builder()
                .setServiceUuid(ParcelUuid(targetServiceUuid))
                .build()
        )
        val settings = ScanSettings.Builder()
            .setScanMode(ScanSettings.SCAN_MODE_LOW_LATENCY)   // 전경 스캔: 빠른 발견
            .setCallbackType(ScanSettings.CALLBACK_TYPE_FIRST_MATCH)
            .build()

        scanner?.startScan(filters, settings, scanCallback)
        Log.d("BLE", "스캔 시작")
    }

    @SuppressLint("MissingPermission")
    fun stopScan() {
        scanner?.stopScan(scanCallback)
        Log.d("BLE", "스캔 중지")
    }
}
```

`SCAN_MODE_LOW_LATENCY`는 화면 활성 상태(전경)에서 사용하고, 백그라운드에서는 `SCAN_MODE_LOW_POWER`로 전환해야 배터리 소모를 최소화할 수 있습니다.

---

### 예제 2: GATT 연결 및 Characteristic 알림 수신

스캔으로 발견한 기기에 GATT 연결을 맺고, Heart Rate Characteristic의 알림(Notification)을 구독하여 실시간으로 심박수를 수신합니다.

```kotlin
class BleGattManager(private val context: Context) {

    private var bluetoothGatt: BluetoothGatt? = null

    // Heart Rate 서비스 및 Characteristic UUID (Bluetooth SIG 표준)
    private val HR_SERVICE_UUID = UUID.fromString("0000180D-0000-1000-8000-00805F9B34FB")
    private val HR_CHAR_UUID    = UUID.fromString("00002A37-0000-1000-8000-00805F9B34FB")
    private val CCCD_UUID       = UUID.fromString("00002902-0000-1000-8000-00805F9B34FB")

    private val gattCallback = object : BluetoothGattCallback() {

        @SuppressLint("MissingPermission")
        override fun onConnectionStateChange(gatt: BluetoothGatt, status: Int, newState: Int) {
            when (newState) {
                BluetoothProfile.STATE_CONNECTED -> {
                    Log.d("BLE", "GATT 연결됨. 서비스 탐색 시작")
                    gatt.discoverServices()
                }
                BluetoothProfile.STATE_DISCONNECTED -> {
                    Log.d("BLE", "GATT 연결 해제됨 (status=$status)")
                    bluetoothGatt?.close()
                    bluetoothGatt = null
                }
            }
        }

        override fun onServicesDiscovered(gatt: BluetoothGatt, status: Int) {
            if (status != BluetoothGatt.GATT_SUCCESS) {
                Log.e("BLE", "서비스 탐색 실패: status=$status")
                return
            }
            val characteristic = gatt
                .getService(HR_SERVICE_UUID)
                ?.getCharacteristic(HR_CHAR_UUID) ?: run {
                    Log.e("BLE", "Heart Rate Characteristic을 찾을 수 없음")
                    return
                }
            enableNotification(gatt, characteristic)
        }

        // API 33+ 새로운 콜백 시그니처 (ByteArray 직접 전달)
        override fun onCharacteristicChanged(
            gatt: BluetoothGatt,
            characteristic: BluetoothGattCharacteristic,
            value: ByteArray
        ) {
            val heartRate = parseHeartRate(value)
            Log.d("BLE", "심박수: $heartRate bpm")
        }

        // API 32 이하 deprecated 콜백 (하위 호환성)
        @Suppress("DEPRECATION")
        override fun onCharacteristicChanged(
            gatt: BluetoothGatt,
            characteristic: BluetoothGattCharacteristic
        ) {
            if (Build.VERSION.SDK_INT < Build.VERSION_CODES.TIRAMISU) {
                val heartRate = parseHeartRate(characteristic.value ?: return)
                Log.d("BLE", "심박수(레거시): $heartRate bpm")
            }
        }
    }

    @SuppressLint("MissingPermission")
    fun connect(device: BluetoothDevice) {
        // autoConnect=false: 범위 내 기기에 즉시 연결 시도
        bluetoothGatt = device.connectGatt(
            context,
            /* autoConnect = */ false,
            gattCallback,
            BluetoothDevice.TRANSPORT_LE
        )
    }

    @SuppressLint("MissingPermission")
    private fun enableNotification(
        gatt: BluetoothGatt,
        characteristic: BluetoothGattCharacteristic
    ) {
        // 1단계: 로컬(Android 스택)에서 알림 활성화
        gatt.setCharacteristicNotification(characteristic, true)

        // 2단계: CCCD Descriptor에 ENABLE_NOTIFICATION_VALUE를 써서 원격 기기에도 활성화
        val cccd = characteristic.getDescriptor(CCCD_UUID) ?: return
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
            gatt.writeDescriptor(cccd, BluetoothGattDescriptor.ENABLE_NOTIFICATION_VALUE)
        } else {
            @Suppress("DEPRECATION")
            cccd.value = BluetoothGattDescriptor.ENABLE_NOTIFICATION_VALUE
            @Suppress("DEPRECATION")
            gatt.writeDescriptor(cccd)
        }
    }

    @SuppressLint("MissingPermission")
    fun disconnect() {
        bluetoothGatt?.disconnect()
    }

    // Heart Rate Measurement 포맷 파싱:
    // 첫 바이트의 bit0 = 0 → HR 값은 1바이트(UInt8), bit0 = 1 → 2바이트(UInt16)
    private fun parseHeartRate(data: ByteArray): Int {
        return if (data[0].toInt() and 0x01 == 0) {
            data[1].toInt() and 0xFF
        } else {
            ((data[2].toInt() and 0xFF) shl 8) or (data[1].toInt() and 0xFF)
        }
    }
}
```

---

## 5. 주의사항 및 실전 팁

### 5.1 GATT 작업은 반드시 직렬로 처리하라

BLE GATT 연산(`readCharacteristic`, `writeCharacteristic`, `writeDescriptor`)은 **하나가 완료(콜백 수신)된 후에 다음을 시작**해야 합니다. 동시에 여러 요청을 보내면 `false`를 반환하거나 콜백이 누락됩니다. 아래처럼 큐를 구현하거나 Kotlin 코루틴의 `Mutex`로 직렬화하세요.

```kotlin
private val operationQueue = ArrayDeque<() -> Unit>()
private var isOperationInProgress = false

fun enqueueOperation(operation: () -> Unit) {
    operationQueue.add(operation)
    if (!isOperationInProgress) executeNext()
}

private fun executeNext() {
    if (operationQueue.isEmpty()) {
        isOperationInProgress = false
        return
    }
    isOperationInProgress = true
    operationQueue.removeFirst().invoke()
}
// 각 콜백(onCharacteristicRead, onDescriptorWrite 등) 마지막에 executeNext() 호출
```

### 5.2 close()를 반드시 호출하라

`bluetoothGatt.close()`를 호출하지 않으면 BLE 연결 슬롯이 해제되지 않습니다. Android는 동시에 약 7개의 BLE 연결만 지원하며, `close()` 없이 반복 연결하면 **GATT 133 오류**(`GATT_ERROR`)가 발생합니다. 연결 해제 콜백 내에서 항상 `close()`를 호출하세요.

### 5.3 MTU 협상으로 처리량 향상

기본 MTU는 20바이트입니다. 큰 데이터를 전송할 때는 연결 직후 `gatt.requestMtu(512)`를 호출하면 최대 512바이트까지 확장 가능합니다. `onMtuChanged(gatt, mtu, status)` 콜백에서 실제 협상된 크기를 확인하고 그에 맞게 데이터를 분할(chunking)하세요.

### 5.4 실기기 테스트와 BLE 분석 도구 활용

BLE는 하드웨어·펌웨어와의 협력이 필수인 영역입니다. **nRF Connect**(Nordic Semiconductor)나 **LightBlue** 같은 BLE 분석 앱으로 대상 기기의 GATT 구조(서비스·Characteristic UUID, 속성)를 미리 파악하면 개발 속도를 크게 높일 수 있습니다.

### 5.5 Android 12+ 권한 분기 처리

Android 12(API 31)부터 `BLUETOOTH_SCAN`과 `BLUETOOTH_CONNECT`가 분리됐습니다. 스캔만 하는 기능은 `BLUETOOTH_SCAN`만, 기기 이름 조회나 GATT 연결 시에는 `BLUETOOTH_CONNECT`도 요청해야 합니다. `@SuppressLint("MissingPermission")` 사용 전에 런타임 권한 확인 로직을 반드시 배치하세요.

---

이 글의 예제들은 Android 12 이상을 기준으로 작성됐으며, API 30 이하를 지원할 때는 `ACCESS_FINE_LOCATION` 권한 추가와 deprecated API 분기 처리가 필요합니다. BLE 개발은 하드웨어 변수가 많으므로 반드시 실기기 환경에서 충분히 검증하세요.

## 참고 자료
- [Bluetooth Low Energy 개요 - Android Developers](https://developer.android.com/develop/connectivity/bluetooth/ble/ble-overview)
- [GATT 서버에 연결하기 - Android Developers](https://developer.android.com/develop/connectivity/bluetooth/ble/connect-gatt-server)
- [BLE 데이터 전송 - Android Developers](https://developer.android.com/develop/connectivity/bluetooth/ble/transfer-ble-data)
