---
layout: post
title: "Android Photo Picker 심화: PickVisualMedia·READ_MEDIA_VISUAL_USER_SELECTED·Embedded Photo Picker 완전 정복"
date: 2026-09-15
categories: [android, flutter]
tags: [android, photo-picker, PickVisualMedia, storage, permissions, media]
---

## 개요

Android 13(API 33)부터 Google은 기존의 `Intent.ACTION_PICK` 또는 Storage Access Framework(SAF) 방식 대신, **Photo Picker**라는 시스템 제공 미디어 선택 UI를 도입했습니다. Photo Picker는 별도 프로세스에서 실행되어 앱이 사용자의 전체 갤러리에 접근할 수 없게 하고, 사용자가 직접 선택한 파일 URI만을 앱에 돌려주는 프라이버시 중심 설계입니다.

이 글에서는 Photo Picker의 내부 동작 원리, `PickVisualMedia` / `PickMultipleVisualMedia` 계약 사용법, Android 14에서 새로 도입된 `READ_MEDIA_VISUAL_USER_SELECTED` 권한 처리, 그리고 최신 Embedded Photo Picker까지 실제 코드와 함께 완전히 파헤칩니다.

---

## Photo Picker란 무엇인가?

Photo Picker는 운영체제가 직접 제공하는 이미지·동영상 선택기입니다. 앱 프로세스 외부에서 독립적으로 실행되므로, 사용자가 선택한 미디어의 URI만 앱에 전달되며 나머지 파일에는 접근이 불가능합니다.

### 기존 방식과의 비교

| 방식 | 권한 필요 | 접근 범위 | 프라이버시 |
|---|---|---|---|
| `Intent.ACTION_PICK` | `READ_EXTERNAL_STORAGE` | 갤러리 전체 | 낮음 |
| SAF (`ACTION_OPEN_DOCUMENT`) | 없음(선택 가능) | 선택한 파일만 | 보통 |
| **Photo Picker** | **없음** | **선택한 파일만** | **높음** |

Photo Picker를 사용하면 `READ_EXTERNAL_STORAGE`, `READ_MEDIA_IMAGES`, `READ_MEDIA_VIDEO` 권한을 **전혀 요청하지 않아도** 됩니다. 이는 Google Play 정책의 민감 권한 심사 부담을 줄여주고 사용자 신뢰를 높이는 핵심 이점입니다.

### 지원 범위

- **Android 13(API 33)** 이상: OS 기본 내장
- **Android 11–12(API 30–32)**: Google Play 서비스를 통한 모듈형 시스템 구성 요소로 제공
- **Android 4.4(API 19)** 이상: Jetpack `androidx.activity:activity` 1.7.0+를 통해 백포트

---

## 왜 Photo Picker를 써야 하는가?

### 1. 권한 없이 미디어 선택

기존에는 갤러리 접근을 위해 `READ_EXTERNAL_STORAGE`를 선언해야 했으나, Photo Picker는 권한 없이 동작합니다. Android 13+에서 `READ_MEDIA_IMAGES`/`READ_MEDIA_VIDEO`를 사용하는 경우에도 Photo Picker를 쓰면 이를 대체할 수 있습니다.

### 2. 사용자 경험 일관성

시스템 제공 UI이므로 Android 버전별 일관된 디자인(Material You 지원)을 보장하며, 별도 UI 구현 비용이 없습니다.

### 3. 클라우드 미디어 통합

Google Photos 등 클라우드 미디어 제공자와 연동하여 로컬과 클라우드 파일을 동시에 탐색·선택할 수 있습니다.

### 4. Android 14의 Partial Access 자동 처리

Android 14(API 34)부터 사용자가 "일부 사진만 허용"을 선택하더라도 Photo Picker는 이 제한을 우아하게 처리합니다.

---

## 실제 구현 예제

### 의존성 추가

```kotlin
// build.gradle.kts
dependencies {
    implementation("androidx.activity:activity-ktx:1.9.3")
}
```

### 예제 1: 단일 미디어 선택 (PickVisualMedia)

```kotlin
import android.net.Uri
import android.os.Bundle
import android.util.Log
import androidx.activity.result.PickVisualMediaRequest
import androidx.activity.result.contract.ActivityResultContracts.PickVisualMedia
import androidx.appcompat.app.AppCompatActivity
import coil.load

class SinglePickerActivity : AppCompatActivity() {

    // PickVisualMedia 계약으로 런처 등록
    private val pickMedia = registerForActivityResult(PickVisualMedia()) { uri: Uri? ->
        if (uri != null) {
            Log.d("PhotoPicker", "선택된 URI: $uri")
            // URI를 이미지뷰에 로드 (Coil 사용 예시)
            binding.imageView.load(uri)
            // 앱 재시작 후에도 접근하려면 영구 권한 획득
            contentResolver.takePersistableUriPermission(
                uri,
                android.content.Intent.FLAG_GRANT_READ_URI_PERMISSION
            )
        } else {
            Log.d("PhotoPicker", "선택 취소")
        }
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // ...

        binding.btnPickImage.setOnClickListener {
            // 이미지만 선택
            pickMedia.launch(
                PickVisualMediaRequest(PickVisualMedia.ImageOnly)
            )
        }

        binding.btnPickVideo.setOnClickListener {
            // 동영상만 선택
            pickMedia.launch(
                PickVisualMediaRequest(PickVisualMedia.VideoOnly)
            )
        }

        binding.btnPickGif.setOnClickListener {
            // 특정 MIME 타입(GIF)만 선택
            pickMedia.launch(
                PickVisualMediaRequest(PickVisualMedia.SingleMimeType("image/gif"))
            )
        }
    }
}
```

`PickVisualMediaRequest`의 `mediaType` 파라미터로 `ImageOnly`, `VideoOnly`, `ImageAndVideo`, `SingleMimeType(mimeType)` 중 하나를 지정할 수 있습니다.

### 예제 2: 다중 미디어 선택 (PickMultipleVisualMedia)

```kotlin
import android.net.Uri
import android.os.Bundle
import android.util.Log
import androidx.activity.result.PickVisualMediaRequest
import androidx.activity.result.contract.ActivityResultContracts.PickMultipleVisualMedia
import androidx.activity.result.contract.ActivityResultContracts.PickVisualMedia
import androidx.appcompat.app.AppCompatActivity
import androidx.recyclerview.widget.RecyclerView

class MultiplePickerActivity : AppCompatActivity() {

    // 최대 5개 선택 (시스템 한도 MediaStore.getPickImagesMaxLimit() 초과 불가)
    private val pickMultipleMedia =
        registerForActivityResult(PickMultipleVisualMedia(maxItems = 5)) { uris: List<Uri> ->
            if (uris.isNotEmpty()) {
                Log.d("PhotoPicker", "선택된 항목 수: ${uris.size}")
                uris.forEach { uri ->
                    Log.d("PhotoPicker", "URI: $uri")
                }
                // RecyclerView 어댑터에 반영
                mediaAdapter.submitList(uris)
            } else {
                Log.d("PhotoPicker", "선택 취소 또는 빈 선택")
            }
        }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // ...

        // 기기가 Photo Picker를 지원하는지 확인 (Jetpack이 자동 폴백하지만 명시적 체크도 가능)
        val isAvailable = PickVisualMedia.isPhotoPickerAvailable(this)
        Log.d("PhotoPicker", "Photo Picker 지원 여부: $isAvailable")

        binding.btnPickMultiple.setOnClickListener {
            pickMultipleMedia.launch(
                PickVisualMediaRequest(PickVisualMedia.ImageAndVideo)
            )
        }
    }

    private fun getSystemMaxLimit(): Int {
        return if (android.os.Build.VERSION.SDK_INT >= android.os.Build.VERSION_CODES.TIRAMISU) {
            android.provider.MediaStore.getPickImagesMaxLimit()
        } else {
            100 // 폴백 기본값
        }
    }
}
```

`maxItems` 값은 `MediaStore.getPickImagesMaxLimit()`이 반환하는 시스템 최대 한도를 초과할 수 없습니다. 초과 시 런타임 예외가 발생합니다.

---

## Android 14의 READ_MEDIA_VISUAL_USER_SELECTED 처리

Photo Picker를 사용하지 않고 커스텀 갤러리 UI를 사용해야 하는 경우(예: 앱 내 갤러리 UI), Android 14부터 사용자가 "일부 사진만 허용"할 수 있는 **Partial Access**를 처리해야 합니다.

```kotlin
// AndroidManifest.xml
// <uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />
// <uses-permission android:name="android.permission.READ_MEDIA_VIDEO" />
// <uses-permission android:name="android.permission.READ_MEDIA_VISUAL_USER_SELECTED" />

import android.Manifest.permission.*
import android.content.pm.PackageManager.PERMISSION_GRANTED
import android.os.Build
import androidx.activity.result.contract.ActivityResultContracts.RequestMultiplePermissions
import androidx.core.content.ContextCompat

class GalleryPermissionActivity : AppCompatActivity() {

    private val requestPermissions =
        registerForActivityResult(RequestMultiplePermissions()) { grants ->
            val fullImage = grants[READ_MEDIA_IMAGES] == true
            val fullVideo = grants[READ_MEDIA_VIDEO] == true
            val userSelected = grants[READ_MEDIA_VISUAL_USER_SELECTED] == true

            when {
                fullImage || fullVideo -> loadAllMedia()
                userSelected -> loadUserSelectedMedia()  // 부분 접근
                else -> showPermissionDeniedMessage()
            }
        }

    private fun checkAndRequestPermissions() {
        val required = when {
            Build.VERSION.SDK_INT >= Build.VERSION_CODES.UPSIDE_DOWN_CAKE -> {
                // Android 14+: 부분 접근 권한 추가
                arrayOf(READ_MEDIA_IMAGES, READ_MEDIA_VIDEO, READ_MEDIA_VISUAL_USER_SELECTED)
            }
            Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU -> {
                // Android 13: 미디어 유형별 권한
                arrayOf(READ_MEDIA_IMAGES, READ_MEDIA_VIDEO)
            }
            else -> {
                // Android 12L 이하: 통합 읽기 권한
                arrayOf(READ_EXTERNAL_STORAGE)
            }
        }
        requestPermissions.launch(required)
    }

    private fun getAccessLevel(): AccessLevel {
        return when {
            Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU &&
            (ContextCompat.checkSelfPermission(this, READ_MEDIA_IMAGES) == PERMISSION_GRANTED ||
             ContextCompat.checkSelfPermission(this, READ_MEDIA_VIDEO) == PERMISSION_GRANTED) ->
                AccessLevel.FULL

            Build.VERSION.SDK_INT >= Build.VERSION_CODES.UPSIDE_DOWN_CAKE &&
            ContextCompat.checkSelfPermission(this, READ_MEDIA_VISUAL_USER_SELECTED) == PERMISSION_GRANTED ->
                AccessLevel.PARTIAL

            ContextCompat.checkSelfPermission(this, READ_EXTERNAL_STORAGE) == PERMISSION_GRANTED ->
                AccessLevel.FULL

            else -> AccessLevel.DENIED
        }
    }

    enum class AccessLevel { FULL, PARTIAL, DENIED }
}
```

**핵심**: `READ_MEDIA_VISUAL_USER_SELECTED`가 부여된 상태(Partial Access)에서는 사용자가 명시적으로 허용한 파일만 MediaStore 쿼리 결과에 포함됩니다. `onResume`에서 접근 수준을 재확인하고 UI를 갱신해야 합니다.

---

## Embedded Photo Picker (Android 16)

Android 16(API 36)과 Jetpack PhotoPicker 라이브러리 1.0.0-alpha02에서는 **Embedded Photo Picker**가 추가되었습니다. 기존 Photo Picker가 전체 화면 팝업이었다면, Embedded Photo Picker는 앱의 뷰 계층 구조 안에 인라인으로 삽입할 수 있습니다.

```kotlin
// build.gradle.kts
implementation("androidx.photopicker:photopicker-compose:1.0.0-alpha02")
```

```kotlin
// Jetpack Compose에서 Embedded Photo Picker 사용
@Composable
fun EmbeddedPickerScreen() {
    val state = rememberEmbeddedPhotoPickerState()

    Column {
        // 앱 UI의 일부로 포토 피커가 삽입됨
        EmbeddedPhotopicker(
            state = state,
            onUriPermissionGranted = { uris ->
                // 선택된 URI 처리
                uris.forEach { uri ->
                    Log.d("EmbeddedPicker", "선택: $uri")
                }
            }
        )
        Button(onClick = { /* 추가 액션 */ }) {
            Text("선택 확인")
        }
    }
}
```

Embedded Photo Picker는 별도 프로세스(process isolation)를 유지하면서도 앱 UI에 자연스럽게 통합되어, 커스텀 갤러리 UI 대비 보안 수준을 높일 수 있습니다. 단, API 34 이상에서만 동작하고 현재(2026년 9월) alpha 단계이므로 프로덕션 적용 시 주의가 필요합니다.

---

## 주의사항 및 팁

### 1. URI 유효 기간

Photo Picker가 반환하는 URI는 **세션 기반**입니다. 앱 재시작 후에도 파일에 접근하려면 반드시 `ContentResolver.takePersistableUriPermission()`을 호출해 영구 권한을 확보해야 합니다. SAF URI와 마찬가지로 영구 권한에는 시스템 한도가 있으며, 초과 시 가장 오래된 권한이 자동 해제됩니다.

### 2. maxItems 한도 초과 방지

```kotlin
val maxLimit = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
    android.provider.MediaStore.getPickImagesMaxLimit()
} else {
    Int.MAX_VALUE
}
val safeMax = minOf(requestedMax, maxLimit)
registerForActivityResult(PickMultipleVisualMedia(safeMax)) { ... }
```

### 3. 폴백 전략 불필요

Jetpack의 `PickVisualMedia` 계약은 기기가 Photo Picker를 지원하지 않으면 자동으로 `ACTION_OPEN_DOCUMENT`로 폴백합니다. 개발자가 별도 분기 처리를 할 필요가 없습니다.

### 4. onResume에서 권한 재확인

커스텀 갤러리를 사용하는 경우, 사용자가 설정에서 권한을 변경하면 `onResume`이 호출됩니다. 이때 접근 수준을 재확인하고 표시 목록을 갱신해야 합니다.

### 5. HDR 동영상 트랜스코딩 (Android 13+)

HDR 동영상을 선택했을 때 기기가 해당 포맷을 지원하지 않으면, 시스템이 SDR로 자동 변환하도록 `MediaCapabilities`를 지정할 수 있습니다.

```kotlin
val request = PickVisualMediaRequest.Builder()
    .setMediaType(PickVisualMedia.VideoOnly)
    .setMediaCapabilitiesForTranscoding(
        MediaCapabilities.Builder()
            .addSupportedHdrType(MediaCapabilities.HdrType.TYPE_HLG10)
            .addSupportedHdrType(MediaCapabilities.HdrType.TYPE_HDR10)
            .build()
    )
    .build()
pickMedia.launch(request)
```

---

## 정리

| 기능 | API | 최소 요구사항 |
|---|---|---|
| 단일 미디어 선택 | `PickVisualMedia` | activity-ktx 1.7.0 |
| 다중 미디어 선택 | `PickMultipleVisualMedia(max)` | activity-ktx 1.7.0 |
| 부분 접근 권한 | `READ_MEDIA_VISUAL_USER_SELECTED` | Android 14 (API 34) |
| 임베디드 피커 (Compose) | `EmbeddedPhotopicker` | Android 16 (API 36), photopicker-compose |
| 영구 URI 권한 | `takePersistableUriPermission` | 모든 버전 |

Photo Picker는 단순한 API 변경이 아니라 **앱이 불필요하게 전체 갤러리에 접근하는 관행을 구조적으로 차단**하는 플랫폼 수준의 프라이버시 강화입니다. 신규 Android 앱은 물론 레거시 앱도 Photo Picker로 전환하면 권한 심사 부담이 줄어들고 사용자 신뢰도가 높아집니다.

## 참고 자료
- [Android 공식 Photo Picker 가이드](https://developer.android.com/training/data-storage/shared/photo-picker)
- [Android 14: 사진·동영상 부분 접근 권한](https://developer.android.com/about/versions/14/changes/partial-photo-video-access)
- [Jetpack PhotoPicker 라이브러리 릴리즈 노트](https://developer.android.com/jetpack/androidx/releases/photopicker)
