---
layout: post
title: "Android Runtime Permissions 심화: ActivityResultLauncher·Permission Rationale·다중 권한 처리 패턴 완전 정복"
date: 2026-08-29
categories: [android, flutter]
tags: [android, permissions, activityresultlauncher, runtime-permissions, kotlin, scoped-storage]
---

모든 Android 앱은 카메라, 마이크, 위치, 연락처 같은 민감한 자원에 접근하기 위해 **런타임 퍼미션(Runtime Permission)** 을 사용자에게 요청해야 합니다. Android 6.0(API 23)에서 도입된 이 체계는 이후 버전을 거치며 꾸준히 변화했고, Android 13·14에서는 세분화된 미디어 권한과 사진 선택기(Photo Picker)가 추가되어 처리 방식이 한층 복잡해졌습니다.

이 아티클에서는 최신 `ActivityResultLauncher` API를 중심으로 권한 요청의 전체 흐름을 완전하게 파악하고, 프로덕션 수준의 다중 권한 처리 패턴까지 구현합니다.

---

## 개념 설명: Runtime Permission이란 무엇인가

Android는 권한을 크게 세 가지로 분류합니다.

| 유형 | 설명 | 처리 방식 |
|------|------|-----------|
| **Install-time (Normal)** | INTERNET, VIBRATE 등 위험도 낮음 | 설치 시 자동 부여 |
| **Runtime (Dangerous)** | 카메라, 위치, 연락처 등 민감 데이터 | 실행 중 사용자 동의 필요 |
| **Special** | MANAGE_EXTERNAL_STORAGE 등 | 시스템 설정 화면으로 이동 |

Runtime Permission의 핵심은 **"필요한 시점에, 맥락과 함께 요청하라"** 는 원칙입니다. 앱 시작 시 모든 권한을 한꺼번에 요청하는 패턴은 Google Play 정책 위반이자 UX 측면에서도 최악입니다.

### 권한 처리 상태 흐름

```
checkSelfPermission()
    ├── GRANTED → 기능 실행
    └── DENIED
          ├── shouldShowRequestPermissionRationale() == true
          │     → 설명 다이얼로그 표시 후 요청
          └── shouldShowRequestPermissionRationale() == false
                ├── 최초 요청 → 바로 요청
                └── 영구 거부 → 앱 설정 화면 안내
```

---

## 왜 필요한가: 역사적 맥락과 현재의 중요성

**Android 6.0 이전**에는 앱 설치 시 `AndroidManifest.xml`에 선언된 모든 권한이 한꺼번에 부여되었습니다. 사용자는 권한 목록을 확인하고 설치를 수락하거나 거부하는 것밖에 선택지가 없었습니다.

Runtime Permission 체계 도입 이후:
- 사용자는 개별 권한을 선택적으로 거부할 수 있습니다.
- Android 11부터는 **One-time Permission**이 생겨 "이번 한 번만" 허용이 가능합니다.
- Android 13부터는 `READ_EXTERNAL_STORAGE` 하나로 묶였던 미디어 접근 권한이 이미지/비디오/오디오로 세분화되었습니다.
- Android 14에서는 사용자가 **일부 사진/동영상만 선택**해서 공유하는 부분 접근 모드가 추가되었습니다.

이처럼 권한 정책은 앱 대상 API 레벨에 따라 다르게 적용되므로, 단순한 보일러플레이트 코드가 아닌 체계적인 처리 전략이 필수입니다.

---

## 실제 구현 예제

### 예제 1: ActivityResultLauncher로 단일·다중 권한 요청

과거에는 `requestPermissions()` → `onRequestPermissionsResult()` 콜백 방식을 사용했지만, 현재 권장 방식은 `ActivityResultLauncher`입니다. 요청과 콜백이 같은 위치에 있어 코드 흐름이 훨씬 명확합니다.

```kotlin
import android.Manifest
import android.content.Intent
import android.net.Uri
import android.os.Build
import android.provider.Settings
import androidx.activity.result.contract.ActivityResultContracts
import androidx.appcompat.app.AlertDialog
import androidx.appcompat.app.AppCompatActivity
import androidx.core.content.ContextCompat
import android.content.pm.PackageManager

class CameraActivity : AppCompatActivity() {

    // 단일 권한 요청
    private val requestCameraPermission =
        registerForActivityResult(ActivityResultContracts.RequestPermission()) { isGranted ->
            if (isGranted) {
                openCamera()
            } else {
                handlePermissionDenied(Manifest.permission.CAMERA)
            }
        }

    // 다중 권한 요청
    private val requestMediaPermissions =
        registerForActivityResult(ActivityResultContracts.RequestMultiplePermissions()) { permissions ->
            val allGranted = permissions.values.all { it }
            if (allGranted) {
                loadMediaFiles()
            } else {
                val denied = permissions.filterValues { !it }.keys
                handleMultiplePermissionsDenied(denied)
            }
        }

    fun onCameraButtonClick() {
        when {
            ContextCompat.checkSelfPermission(
                this, Manifest.permission.CAMERA
            ) == PackageManager.PERMISSION_GRANTED -> {
                openCamera()
            }
            shouldShowRequestPermissionRationale(Manifest.permission.CAMERA) -> {
                showRationaleDialog(
                    message = "카메라 권한은 사진 촬영 기능에 필요합니다.",
                    onConfirm = { requestCameraPermission.launch(Manifest.permission.CAMERA) }
                )
            }
            else -> {
                requestCameraPermission.launch(Manifest.permission.CAMERA)
            }
        }
    }

    fun onMediaAccessButtonClick() {
        // Android 13+ 세분화 권한, 이전 버전 호환 처리
        val permissions = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
            arrayOf(
                Manifest.permission.READ_MEDIA_IMAGES,
                Manifest.permission.READ_MEDIA_VIDEO
            )
        } else {
            arrayOf(Manifest.permission.READ_EXTERNAL_STORAGE)
        }

        val notGranted = permissions.filter {
            ContextCompat.checkSelfPermission(this, it) != PackageManager.PERMISSION_GRANTED
        }

        if (notGranted.isEmpty()) {
            loadMediaFiles()
        } else {
            requestMediaPermissions.launch(notGranted.toTypedArray())
        }
    }

    private fun handlePermissionDenied(permission: String) {
        if (!shouldShowRequestPermissionRationale(permission)) {
            // 영구 거부: 설정 화면으로 안내
            showGoToSettingsDialog()
        } else {
            // 일반 거부
            showSnackbar("카메라 권한이 필요합니다.")
        }
    }

    private fun handleMultiplePermissionsDenied(denied: Set<String>) {
        val permanentlyDenied = denied.none { shouldShowRequestPermissionRationale(it) }
        if (permanentlyDenied) {
            showGoToSettingsDialog()
        }
    }

    private fun showRationaleDialog(message: String, onConfirm: () -> Unit) {
        AlertDialog.Builder(this)
            .setTitle("권한 안내")
            .setMessage(message)
            .setPositiveButton("허용") { _, _ -> onConfirm() }
            .setNegativeButton("취소", null)
            .show()
    }

    private fun showGoToSettingsDialog() {
        AlertDialog.Builder(this)
            .setTitle("권한 설정 필요")
            .setMessage("권한이 영구적으로 거부되었습니다. 설정에서 직접 허용해 주세요.")
            .setPositiveButton("설정으로 이동") { _, _ ->
                val intent = Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS).apply {
                    data = Uri.fromParts("package", packageName, null)
                }
                startActivity(intent)
            }
            .setNegativeButton("취소", null)
            .show()
    }

    private fun openCamera() { /* 카메라 구현 */ }
    private fun loadMediaFiles() { /* 미디어 로드 구현 */ }
    private fun showSnackbar(msg: String) { /* Snackbar 표시 */ }
}
```

---

### 예제 2: 재사용 가능한 PermissionManager 유틸리티

실무에서는 여러 화면에서 반복적으로 동일한 권한 처리 로직이 필요합니다. `PermissionManager`를 만들어 Activity·Fragment 어디서든 재사용 가능한 구조를 만들 수 있습니다.

```kotlin
import android.content.Context
import android.content.Intent
import android.content.pm.PackageManager
import android.net.Uri
import android.provider.Settings
import androidx.activity.ComponentActivity
import androidx.activity.result.ActivityResultLauncher
import androidx.activity.result.contract.ActivityResultContracts
import androidx.core.content.ContextCompat
import androidx.lifecycle.DefaultLifecycleObserver
import androidx.lifecycle.LifecycleOwner

/**
 * 단일/다중 권한 요청을 추상화한 재사용 가능한 권한 관리자.
 * Activity의 onCreate() 이전에 반드시 초기화해야 합니다.
 */
class PermissionManager(
    private val activity: ComponentActivity
) : DefaultLifecycleObserver {

    sealed class PermissionResult {
        object Granted : PermissionResult()
        data class Denied(val permissions: List<String>) : PermissionResult()
        data class PermanentlyDenied(val permissions: List<String>) : PermissionResult()
    }

    private var pendingCallback: ((PermissionResult) -> Unit)? = null

    private val multiplePermissionsLauncher: ActivityResultLauncher<Array<String>> =
        activity.registerForActivityResult(
            ActivityResultContracts.RequestMultiplePermissions()
        ) { results ->
            val granted = results.filterValues { it }.keys.toList()
            val denied = results.filterValues { !it }.keys.toList()

            val permanentlyDenied = denied.filter { permission ->
                !activity.shouldShowRequestPermissionRationale(permission)
            }
            val temporaryDenied = denied - permanentlyDenied.toSet()

            val result = when {
                denied.isEmpty() -> PermissionResult.Granted
                permanentlyDenied.isNotEmpty() -> PermissionResult.PermanentlyDenied(permanentlyDenied)
                else -> PermissionResult.Denied(temporaryDenied)
            }
            pendingCallback?.invoke(result)
            pendingCallback = null
        }

    init {
        activity.lifecycle.addObserver(this)
    }

    /**
     * 권한 요청 진입점.
     * 이미 허용된 경우 즉시 Granted 콜백을 반환합니다.
     */
    fun requestPermissions(
        vararg permissions: String,
        onResult: (PermissionResult) -> Unit
    ) {
        val notGranted = permissions.filter { permission ->
            ContextCompat.checkSelfPermission(
                activity, permission
            ) != PackageManager.PERMISSION_GRANTED
        }

        if (notGranted.isEmpty()) {
            onResult(PermissionResult.Granted)
            return
        }

        pendingCallback = onResult
        multiplePermissionsLauncher.launch(notGranted.toTypedArray())
    }

    fun openAppSettings() {
        val intent = Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS).apply {
            data = Uri.fromParts("package", activity.packageName, null)
        }
        activity.startActivity(intent)
    }

    override fun onDestroy(owner: LifecycleOwner) {
        pendingCallback = null
    }
}

// --- 사용 예시 ---

class ProfileActivity : ComponentActivity() {

    private val permissionManager = PermissionManager(this)

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        binding.btnPickPhoto.setOnClickListener {
            val permissions = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
                arrayOf(Manifest.permission.READ_MEDIA_IMAGES)
            } else {
                arrayOf(Manifest.permission.READ_EXTERNAL_STORAGE)
            }

            permissionManager.requestPermissions(*permissions) { result ->
                when (result) {
                    is PermissionManager.PermissionResult.Granted -> {
                        pickPhotoFromGallery()
                    }
                    is PermissionManager.PermissionResult.Denied -> {
                        showToast("사진 접근 권한이 필요합니다.")
                    }
                    is PermissionManager.PermissionResult.PermanentlyDenied -> {
                        AlertDialog.Builder(this)
                            .setMessage("사진 권한이 차단되었습니다. 설정에서 허용해 주세요.")
                            .setPositiveButton("설정") { _, _ -> permissionManager.openAppSettings() }
                            .setNegativeButton("취소", null)
                            .show()
                    }
                }
            }
        }
    }
}
```

---

## 주의사항 및 실전 팁

### 1. Android 13+ 세분화된 미디어 권한

Android 13(API 33)부터 `READ_EXTERNAL_STORAGE` 권한이 세 가지로 분리되었습니다.

| 권한 | 접근 대상 |
|------|-----------|
| `READ_MEDIA_IMAGES` | JPEG, PNG, WebP 등 이미지 |
| `READ_MEDIA_VIDEO` | MP4, MKV 등 동영상 |
| `READ_MEDIA_AUDIO` | MP3, FLAC 등 오디오 |

앱이 targetSdkVersion 33 이상이면서 Android 13 기기에서 실행 중이라면 이전 `READ_EXTERNAL_STORAGE`를 요청해도 부여되지 않습니다. `Build.VERSION.SDK_INT` 분기가 필수입니다.

### 2. Android 14+ 부분 미디어 접근 (선택적 사진 공유)

Android 14에서는 사용자가 "전체 허용" 대신 특정 사진/동영상만 선택해서 앱에 공유할 수 있는 **부분 접근(Partial access)** 모드가 추가되었습니다. `READ_MEDIA_VISUAL_USER_SELECTED` 권한을 함께 선언해야 이 모드를 처리할 수 있습니다.

```xml
<!-- AndroidManifest.xml -->
<uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />
<uses-permission android:name="android.permission.READ_MEDIA_VIDEO" />
<!-- Android 14+ 부분 접근 지원 -->
<uses-permission android:name="android.permission.READ_MEDIA_VISUAL_USER_SELECTED" />
```

### 3. Photo Picker 사용 권장

미디어 파일을 선택하는 기능이라면 권한 없이도 동작하는 **Photo Picker API**를 최우선으로 고려하세요. `ActivityResultContracts.PickVisualMedia`를 사용하면 별도 권한 없이 시스템 UI를 통해 안전하게 미디어를 선택할 수 있습니다.

```kotlin
private val pickMedia =
    registerForActivityResult(ActivityResultContracts.PickVisualMedia()) { uri ->
        if (uri != null) {
            imageView.setImageURI(uri)
        }
    }

fun onPickPhotoClick() {
    pickMedia.launch(PickVisualMediaRequest(ActivityResultContracts.PickVisualMedia.ImageOnly))
}
```

### 4. One-Time Permission (Android 11+)

Android 11부터 위치, 카메라, 마이크에 대해 "이번만 허용" 옵션이 생겼습니다. 앱이 백그라운드로 이동하거나 일정 시간이 지나면 권한이 자동으로 취소됩니다. 따라서 **앱 포그라운드 복귀 시 매번 권한 상태를 재확인**하는 로직이 필수입니다.

```kotlin
override fun onResume() {
    super.onResume()
    if (!hasCameraPermission()) {
        updateCameraButtonState(enabled = false)
    }
}

private fun hasCameraPermission() =
    ContextCompat.checkSelfPermission(this, Manifest.permission.CAMERA) ==
        PackageManager.PERMISSION_GRANTED
```

### 5. shouldShowRequestPermissionRationale()의 한계

이 메서드는 **권한이 영구 거부된 경우를 직접 감지하지 못합니다.** 영구 거부 여부는 "권한 요청 후 결과가 DENIED이고 `shouldShowRequestPermissionRationale()`가 false를 반환할 때"로만 판단할 수 있습니다. 따라서 권한 요청 이력을 `SharedPreferences` 또는 `DataStore`에 별도로 저장해 두면 더 정밀한 판단이 가능합니다.

### 6. Android 12 블루투스 권한 변경

Android 12(API 31)부터 블루투스 권한이 세분화되었습니다.

| 권한 | 용도 |
|------|------|
| `BLUETOOTH_SCAN` | 주변 기기 스캔 |
| `BLUETOOTH_CONNECT` | 페어링된 기기 연결 |
| `BLUETOOTH_ADVERTISE` | BLE 어드버타이징 |

API 31 미만에서는 `BLUETOOTH`와 `BLUETOOTH_ADMIN`을 사용하므로 반드시 버전 분기 처리가 필요합니다.

---

## 정리

Android Runtime Permission 처리는 단순 보일러플레이트처럼 보이지만, OS 버전별 정책 변화와 UX 고려 사항이 촘촘하게 얽혀 있습니다.

핵심 원칙을 정리하면 다음과 같습니다.

1. **맥락 있는 요청**: 권한이 필요한 기능을 사용하려는 시점에만 요청하세요.
2. **Rationale 표시**: `shouldShowRequestPermissionRationale()`가 true면 사용자에게 이유를 먼저 설명하세요.
3. **영구 거부 대응**: 설정 화면으로 안내하는 UX를 반드시 준비하세요.
4. **버전 분기**: `Build.VERSION.SDK_INT`를 사용해 Android 13/14 세분화 권한을 처리하세요.
5. **Photo Picker 우선**: 미디어 선택은 권한 없이 동작하는 Photo Picker를 먼저 고려하세요.

`PermissionManager`처럼 재사용 가능한 추상화 레이어를 두면 팀 전체가 일관된 패턴으로 권한 처리를 구현할 수 있어, 유지보수성과 UX 품질 모두를 높일 수 있습니다.

---

## 참고 자료
- [Android Developers - Request runtime permissions](https://developer.android.com/training/permissions/requesting)
- [Android Developers - Declare app permissions](https://developer.android.com/training/permissions/declaring)
- [Android Developers - Access media files from shared storage](https://developer.android.com/training/data-storage/shared/media)
- [Android Developers - Behavior changes: Android 13 granular media permissions](https://developer.android.com/about/versions/13/behavior-changes-13#granular-media-permissions)
