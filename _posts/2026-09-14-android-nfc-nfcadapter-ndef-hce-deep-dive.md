---
layout: post
title: "Android NFC 심화: NfcAdapter·NDEF 메시지·HCE로 근거리 무선 통신 완전 정복"
date: 2026-09-14
categories: [android]
tags: [android, nfc, nfcadapter, ndef, hce, kotlin, connectivity]
---

NFC(Near Field Communication)는 13.56 MHz 주파수를 이용해 약 4cm 이내의 거리에서 데이터를 교환하는 무선 통신 기술이다. Android는 API 10(Android 2.3)부터 NFC를 공식 지원하기 시작했으며, 오늘날 모바일 결제, 대중교통 카드, 스마트 잠금장치, 디바이스 간 콘텐츠 공유 등 광범위한 영역에서 활용된다. 이 글에서는 NFC의 핵심 구성요소인 `NfcAdapter`, `NDEF` 메시지 처리, `Foreground Dispatch`, 그리고 `Host-based Card Emulation(HCE)`를 Kotlin으로 구현하는 과정을 단계적으로 살펴본다.

---

## NFC의 핵심 개념

### NfcAdapter와 태그 디스패치 시스템

`NfcAdapter`는 Android NFC 기능에 진입하는 단일 창구다. 기기에 NFC 컨트롤러가 하나 존재하듯, 앱도 `NfcAdapter.getDefaultAdapter(context)`로 단 하나의 어댑터 인스턴스를 가져온다.

Android는 NFC 태그를 감지하면 **태그 디스패치 시스템(Tag Dispatch System)**을 통해 적절한 앱·액티비티를 자동으로 결정한다. 우선순위는 다음과 같다:

1. **`ACTION_NDEF_DISCOVERED`** — NDEF 데이터가 담긴 태그를 인식했을 때 가장 먼저 발생
2. **`ACTION_TECH_DISCOVERED`** — 특정 NFC 기술(MifareClassic 등)을 처리할 수 있는 앱이 있을 때
3. **`ACTION_TAG_DISCOVERED`** — 위 두 인텐트로 처리할 앱이 없을 때 최후 수단

### NDEF(NFC Data Exchange Format)

NDEF는 NFC 태그에 저장·교환하는 표준 데이터 형식이다. `NdefMessage`는 하나 이상의 `NdefRecord`로 구성된다. 각 레코드는 **TNF(Type Name Format)**와 **type**, **id**, **payload**로 이루어진다. 대표적인 TNF 값은 다음과 같다:

| TNF | 의미 |
|-----|------|
| `TNF_WELL_KNOWN` | RTD_TEXT, RTD_URI 등 잘 알려진 타입 |
| `TNF_ABSOLUTE_URI` | 절대 URI 포맷 |
| `TNF_MIME_MEDIA` | MIME 타입 데이터 |
| `TNF_EXTERNAL_TYPE` | 외부 타입 (도메인:경로 형식) |

### Host-based Card Emulation(HCE)

HCE는 앱이 NFC 카드처럼 동작하도록 해주는 기능이다. 별도의 Secure Element 없이 Android OS와 앱만으로 ISO-DEP(ISO 14443-4) 기반의 APDU 명령을 처리할 수 있어, 간단한 디지털 명함 교환, 멤버십 카드, 사내 출입증 등을 구현할 때 유용하다.

---

## 왜 NFC를 직접 구현해야 하는가?

모바일 결제는 이미 OS 수준에서 처리되지만, 다음 시나리오에서는 개발자가 직접 NFC를 다뤄야 한다:

- **스마트 태그 자동화**: NFC 태그를 탭하면 Wi-Fi 연결, 앱 실행, 설정 변경 등을 자동 수행
- **디바이스 간 데이터 교환**: 명함·쿠폰·좌석 정보를 태그에 기록하고 다른 기기에서 읽기
- **자체 카드 에뮬레이션**: 사내 출입증, 포인트 카드 같은 맞춤형 NFC 카드 구현
- **IoT 디바이스 페어링**: 블루투스 스피커나 스마트 가전의 NFC 탭-페어링 지원

라이브러리 의존성 없이 Android SDK만으로 구현할 수 있다는 점도 장점이다.

---

## 실제 구현 예제

### 준비: Manifest 설정

```xml
<!-- AndroidManifest.xml -->
<uses-permission android:name="android.permission.NFC" />
<uses-feature android:name="android.hardware.nfc" android:required="true" />

<activity
    android:name=".NfcActivity"
    android:exported="true"
    android:launchMode="singleTop">

    <!-- NDEF 태그를 인식했을 때 이 액티비티를 실행 -->
    <intent-filter>
        <action android:name="android.nfc.action.NDEF_DISCOVERED" />
        <category android:name="android.intent.category.DEFAULT" />
        <data android:mimeType="application/vnd.example.nfc" />
    </intent-filter>
</activity>
```

---

### 예제 1: NDEF 태그 읽기 (Foreground Dispatch)

Foreground Dispatch를 사용하면 앱이 포그라운드에 있을 때 태그를 최우선으로 인터셉트할 수 있다. 백그라운드 실행 중인 다른 앱보다 현재 앱이 먼저 인텐트를 받는다.

```kotlin
import android.app.Activity
import android.app.PendingIntent
import android.content.Intent
import android.content.IntentFilter
import android.nfc.NdefMessage
import android.nfc.NfcAdapter
import android.nfc.Tag
import android.os.Bundle
import android.widget.TextView

class NfcReaderActivity : Activity() {

    private lateinit var nfcAdapter: NfcAdapter
    private lateinit var pendingIntent: PendingIntent
    private lateinit var intentFiltersArray: Array<IntentFilter>
    private lateinit var tvResult: TextView

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_nfc_reader)
        tvResult = findViewById(R.id.tvResult)

        nfcAdapter = NfcAdapter.getDefaultAdapter(this)
            ?: run { finish(); return } // NFC 미지원 기기

        // Foreground Dispatch를 위한 PendingIntent (FLAG_MUTABLE 필수 — Android 12+)
        pendingIntent = PendingIntent.getActivity(
            this, 0,
            Intent(this, javaClass).addFlags(Intent.FLAG_ACTIVITY_SINGLE_TOP),
            PendingIntent.FLAG_MUTABLE
        )

        // NDEF 및 일반 태그 인텐트 필터 등록
        intentFiltersArray = arrayOf(
            IntentFilter(NfcAdapter.ACTION_NDEF_DISCOVERED).apply {
                addDataType("*/*") // 모든 MIME 타입 허용
            },
            IntentFilter(NfcAdapter.ACTION_TAG_DISCOVERED)
        )
    }

    override fun onResume() {
        super.onResume()
        // 포그라운드 디스패치 활성화
        nfcAdapter.enableForegroundDispatch(this, pendingIntent, intentFiltersArray, null)
    }

    override fun onPause() {
        super.onPause()
        // 포그라운드 디스패치 반드시 해제 — 메모리 누수 방지
        nfcAdapter.disableForegroundDispatch(this)
    }

    override fun onNewIntent(intent: Intent) {
        super.onNewIntent(intent)
        handleNfcIntent(intent)
    }

    private fun handleNfcIntent(intent: Intent) {
        val rawMessages = intent.getParcelableArrayExtra(NfcAdapter.EXTRA_NDEF_MESSAGES)
        if (rawMessages != null) {
            // NDEF 메시지 파싱
            val messages = rawMessages.map { it as NdefMessage }
            val sb = StringBuilder()
            messages.forEachIndexed { msgIdx, message ->
                message.records.forEachIndexed { recIdx, record ->
                    val payload = record.payload
                    // TNF_WELL_KNOWN + RTD_TEXT: 첫 바이트는 상태 바이트, 이후 언어 코드 + 텍스트
                    if (record.tnf == android.nfc.NdefRecord.TNF_WELL_KNOWN &&
                        record.type.contentEquals(android.nfc.NdefRecord.RTD_TEXT)
                    ) {
                        val languageCodeLength = payload[0].toInt() and 0x3F
                        val text = String(
                            payload,
                            1 + languageCodeLength,
                            payload.size - 1 - languageCodeLength,
                            Charsets.UTF_8
                        )
                        sb.appendLine("메시지[$msgIdx] 레코드[$recIdx]: $text")
                    } else {
                        sb.appendLine("메시지[$msgIdx] 레코드[$recIdx] RAW: ${payload.size} bytes")
                    }
                }
            }
            tvResult.text = sb.toString()
        } else {
            // NDEF 아닌 태그 처리
            val tag: Tag? = intent.getParcelableExtra(NfcAdapter.EXTRA_TAG)
            tvResult.text = "태그 감지 (NDEF 없음): ${tag?.id?.joinToString(":") { "%02X".format(it) }}"
        }
    }
}
```

**핵심 포인트:**
- `onResume`/`onPause`에서 반드시 `enable`/`disable`을 대칭적으로 호출해야 한다. `onPause`를 빠뜨리면 다른 앱의 NFC 기능이 막힌다.
- `PendingIntent.FLAG_MUTABLE`은 Android 12(API 31) 이상에서 필수다.
- RTD_TEXT 레코드의 페이로드 첫 바이트는 언어 코드 길이와 UTF-8/UTF-16 여부를 인코딩한 상태 바이트다.

---

### 예제 2: NDEF 태그 쓰기

읽기와 달리 쓰기는 `Ndef` TagTechnology를 통해 직접 태그에 데이터를 기록한다. I/O 작업이므로 반드시 코루틴이나 스레드로 메인 스레드를 벗어나야 한다.

```kotlin
import android.nfc.NdefMessage
import android.nfc.NdefRecord
import android.nfc.Tag
import android.nfc.tech.Ndef
import android.nfc.tech.NdefFormatable
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext

/**
 * NFC 태그에 텍스트와 URI 레코드를 함께 기록한다.
 * @return 성공 여부와 오류 메시지 Pair
 */
suspend fun writeNdefTag(tag: Tag, text: String, uri: String): Pair<Boolean, String?> =
    withContext(Dispatchers.IO) {
        // 텍스트 레코드 생성 (TNF_WELL_KNOWN + RTD_TEXT)
        val textRecord = createTextRecord(text, "ko")

        // URI 레코드 생성 (createUri 헬퍼 사용 — API 14+)
        val uriRecord = NdefRecord.createUri(uri)

        val ndefMessage = NdefMessage(arrayOf(textRecord, uriRecord))
        val messageSize = ndefMessage.toByteArray().size

        // 1단계: Ndef 기술이 지원되면 바로 쓰기
        Ndef.get(tag)?.use { ndef ->
            if (!ndef.isWritable) return@withContext Pair(false, "태그가 읽기 전용입니다")
            if (ndef.maxSize < messageSize) {
                return@withContext Pair(false, "태그 용량 부족: ${ndef.maxSize} < $messageSize bytes")
            }
            ndef.connect()
            ndef.writeNdefMessage(ndefMessage)
            return@withContext Pair(true, null)
        }

        // 2단계: NDEF 미포맷 태그는 NdefFormatable로 포맷 후 쓰기
        NdefFormatable.get(tag)?.use { formatable ->
            formatable.connect()
            formatable.format(ndefMessage)
            return@withContext Pair(true, null)
        }

        Pair(false, "이 태그는 NDEF를 지원하지 않습니다")
    }

/** RTD_TEXT 레코드 수동 생성 */
private fun createTextRecord(text: String, languageCode: String): NdefRecord {
    val langBytes = languageCode.toByteArray(Charsets.US_ASCII)
    val textBytes = text.toByteArray(Charsets.UTF_8)
    // 상태 바이트: 비트7=0(UTF-8), 비트6=0(예약), 비트5~0=언어코드 길이
    val statusByte = langBytes.size.toByte()
    val payload = ByteArray(1 + langBytes.size + textBytes.size).apply {
        this[0] = statusByte
        langBytes.copyInto(this, 1)
        textBytes.copyInto(this, 1 + langBytes.size)
    }
    return NdefRecord(NdefRecord.TNF_WELL_KNOWN, NdefRecord.RTD_TEXT, ByteArray(0), payload)
}

// Activity에서의 사용 예
// lifecycleScope.launch {
//     val (success, error) = writeNdefTag(tag, "안녕하세요!", "https://example.com")
//     if (success) showToast("태그 쓰기 성공") else showToast("실패: $error")
// }
```

**핵심 포인트:**
- `Ndef.get(tag)`은 태그가 이미 NDEF 포맷이면 non-null을 반환하고, 포맷되지 않은 태그에는 `NdefFormatable.get(tag)`을 사용한다.
- `use { }` 블록으로 `TagTechnology`를 자동으로 `close()`한다. 연결을 닫지 않으면 다음 태그 접근이 실패한다.
- 쓰기는 최소 수십 ms의 I/O 시간이 소요되므로 반드시 `Dispatchers.IO`에서 실행해야 한다.

---

## 주의사항 및 실전 팁

### 1. NFC 지원 여부 확인 및 활성화 유도

```kotlin
val nfcAdapter = NfcAdapter.getDefaultAdapter(context)
when {
    nfcAdapter == null -> showError("이 기기는 NFC를 지원하지 않습니다")
    !nfcAdapter.isEnabled -> {
        // Android 10 이하: 설정 화면으로 이동
        startActivity(Intent(android.provider.Settings.ACTION_NFC_SETTINGS))
    }
}
```

### 2. 포그라운드 디스패치 vs. 인텐트 필터

- **인텐트 필터**: 앱이 백그라운드이거나 완전히 종료된 상태에서도 태그 감지 시 앱 실행
- **포그라운드 디스패치**: 앱이 포그라운드에 있을 때 다른 앱/필터보다 우선 처리, `onNewIntent`로 수신

두 방법을 병용하면 앱이 닫혀 있을 때는 인텐트 필터로 실행되고, 열려 있을 때는 포그라운드 디스패치가 처리한다.

### 3. Android 12+ PendingIntent Mutability

Android 12부터 `PendingIntent` 생성 시 `FLAG_MUTABLE` 또는 `FLAG_IMMUTABLE`을 반드시 명시해야 한다. Foreground Dispatch용 `PendingIntent`는 시스템이 인텐트를 채워 넣어야 하므로 `FLAG_MUTABLE`을 사용해야 한다.

### 4. 태그 탈착 중 예외 처리

사용자가 태그를 너무 빨리 떼어내면 `TagLostException`이 발생한다. 쓰기 작업에서는 반드시 이를 캐치해 사용자에게 재시도를 안내해야 한다.

```kotlin
try {
    ndef.writeNdefMessage(ndefMessage)
} catch (e: android.nfc.TagLostException) {
    // 태그가 중간에 벗어남 — 재시도 요청
} catch (e: java.io.IOException) {
    // 일반 I/O 오류
}
```

### 5. 태그 용량과 NDEF 메시지 크기

일반 NFC 스티커(NTAG213)는 최대 137 바이트만 저장 가능하다. 대용량 데이터가 필요하다면 태그에는 URL만 저장하고 실제 데이터는 서버에서 가져오는 패턴이 현실적이다.

### 6. Android 16+ 앱 허용 목록

Android 16부터 사용자가 특정 앱의 NFC 태그 스캔을 차단할 수 있다. 앱은 `NfcAdapter.isTagIntentAllowed()`로 허용 여부를 확인하고, 거부된 경우 `ACTION_CHANGE_TAG_INTENT_PREFERENCE` 인텐트로 설정 화면을 열어 사용자에게 허용을 요청할 수 있다.

---

## 정리

Android NFC는 단순한 결제 기술을 넘어, IoT 자동화·디지털 명함·출입 통제·자산 추적까지 다양한 실무 시나리오에 활용된다. 핵심은 세 가지다:

1. **Foreground Dispatch**로 포그라운드 앱이 태그를 최우선 처리
2. **NDEF TagTechnology**로 표준 데이터 포맷을 읽고 쓰기
3. **TagLostException**·용량·mutability 등 엣지 케이스를 철저히 방어

HCE까지 결합하면 앱 자체가 NFC 카드처럼 동작할 수 있어, 별도 하드웨어 없이 순수 소프트웨어만으로 NFC 기반 서비스를 완성할 수 있다.

## 참고 자료
- [NFC Basics — Android Developers](https://developer.android.com/develop/connectivity/nfc/nfc)
- [Advanced NFC Overview — Android Developers](https://developer.android.com/develop/connectivity/nfc/advanced-nfc)
- [Host-based Card Emulation Overview — Android Developers](https://developer.android.com/develop/connectivity/nfc/hce)
