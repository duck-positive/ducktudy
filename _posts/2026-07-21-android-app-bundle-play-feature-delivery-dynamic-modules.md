---
layout: post
title: "Android App Bundle과 Play Feature Delivery 심화: 동적 기능 모듈과 SplitInstallManager 완전 정복"
date: 2026-07-21
categories: [android]
tags: [android, app-bundle, dynamic-feature-modules, play-feature-delivery, split-install-manager, kotlin, gradle]
---

앱이 커질수록 첫 설치 크기와 기능 복잡도 사이의 긴장이 커집니다. Android App Bundle과 Play Feature Delivery는 "필요할 때 필요한 기능만 다운로드한다"는 원칙으로 이 문제를 해결합니다. 이 글에서는 동적 기능 모듈의 내부 동작 원리부터 `SplitInstallManager` 실전 구현, 그리고 Compose 기반 네비게이션 통합까지 심층적으로 살펴봅니다.

---

## 1. 개념 설명

### Android App Bundle이란?

Android App Bundle(`.aab`)은 구글이 2018년 도입한 새로운 앱 배포 형식입니다. 기존 APK와 달리 App Bundle은 앱의 모든 컴파일된 코드와 리소스를 하나의 파일로 묶되, 기기별 최적화된 APK를 구글 플레이가 직접 생성해 서빙하는 **Dynamic Delivery** 방식을 사용합니다.

기존 APK 방식에서는 모든 화면 밀도(mdpi, hdpi, xhdpi 등)의 이미지, 모든 언어 문자열 리소스, 모든 CPU 아키텍처(arm64, x86 등) 네이티브 코드가 하나의 파일에 포함되어 불필요한 용량을 차지했습니다. App Bundle을 사용하면 사용자 기기에 실제로 필요한 리소스만 담긴 최적화된 APK가 제공되므로, 앱 설치 크기가 평균 15~35% 감소합니다.

Dynamic Delivery의 핵심 구성 요소는 세 가지입니다.

- **Base APK**: 앱의 핵심 코드와 리소스. 항상 설치됩니다.
- **Configuration APKs**: 기기 화면 밀도, 언어, CPU 아키텍처에 따라 자동 분할된 APK. 구글 플레이가 자동 처리합니다.
- **Dynamic Feature APKs**: 앱 실행 중 필요할 때 런타임에 다운로드 가능한 기능 모듈 APK.

### Play Feature Delivery란?

Play Feature Delivery는 App Bundle의 확장 기능으로, 앱의 특정 기능을 별도의 모듈로 분리하여 조건에 따라 다르게 배포할 수 있게 해줍니다. 배포 방식은 세 가지입니다.

| 배포 방식 | 설명 | 사용 예 |
|---|---|---|
| **Install-time** | 앱 설치 시 함께 설치 (기본값) | 핵심 온보딩 화면 |
| **On-demand** | 앱 실행 중 사용자 요청 시 다운로드 | 결제 모듈, AR 기능 |
| **Conditional** | API 레벨·국가·기기 기능 조건 충족 시 자동 설치 | ARCore 전용 기능, 특정 국가 결제 수단 |

각 동적 기능 모듈은 독립적인 Gradle 모듈로 구성되며, `com.android.dynamic-feature` 플러그인을 적용합니다.

---

## 2. 왜 필요한가?

실제 앱 개발에서 모든 기능이 첫 실행부터 필요한 것은 아닙니다.

- **결제 모듈**: 무료 사용자에게는 전혀 필요 없음
- **AR 카메라 필터**: ARCore를 지원하는 기기에만 필요
- **다국어 OCR 엔진**: 특정 지역 사용자에게만 필요
- **어드민 패널**: 관리자 계정에서만 사용

이런 기능들을 온디맨드 모듈로 분리하면 다음과 같은 이점이 있습니다.

- **설치 크기 감소**: 첫 다운로드 크기가 줄어들어 이탈률이 낮아집니다.
- **기능별 독립 업데이트**: 특정 모듈만 업데이트하여 배포할 수 있습니다.
- **조건부 기능 제공**: 기기 능력이나 사용자 속성에 따른 차별화된 경험을 제공합니다.
- **개발 모듈화**: 기능별 팀 분리와 병렬 개발, 빌드 시간 단축이 가능합니다.

---

## 3. 실제 구현 예제

### 3.1 프로젝트 구성 및 Gradle 설정

먼저 `app/build.gradle.kts`에서 Play Feature Delivery 라이브러리를 추가하고, App Bundle 분리 옵션을 설정합니다.

```kotlin
// app/build.gradle.kts
android {
    bundle {
        language { enableSplit = true }
        density { enableSplit = true }
        abi { enableSplit = true }
    }
}

dependencies {
    implementation("com.google.android.play:feature-delivery:2.1.0")
    implementation("com.google.android.play:feature-delivery-ktx:2.1.0")
}
```

동적 기능 모듈(예: `payment` 모듈)의 `build.gradle.kts`는 다음과 같습니다.

```kotlin
// payment/build.gradle.kts
plugins {
    id("com.android.dynamic-feature")
    id("org.jetbrains.kotlin.android")
}

android {
    namespace = "com.example.app.payment"
    compileSdk = 34
    defaultConfig {
        minSdk = 21
    }
}

dependencies {
    // base 모듈을 구현 의존성으로 참조
    implementation(project(":app"))
}
```

모듈의 `AndroidManifest.xml`에서 배포 방식을 선언합니다.

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:dist="http://schemas.android.com/apk/distribution">

    <dist:module
        dist:instant="false"
        dist:title="@string/title_payment">
        <dist:delivery>
            <!-- 온디맨드 배포 선언 -->
            <dist:on-demand />
        </dist:delivery>
        <!-- fusing: API 20 이하 기기에서 base APK에 합산 포함 여부 -->
        <dist:fusing dist:include="true" />
    </dist:module>
</manifest>
```

### 3.2 SplitInstallManager로 온디맨드 모듈 로드

`SplitInstallManager`를 ViewModel에서 관리하면 상태를 Compose UI에 깔끔하게 연결할 수 있습니다.

```kotlin
class DynamicFeatureViewModel(application: Application) : AndroidViewModel(application) {

    private val splitInstallManager = SplitInstallManagerFactory.create(application)

    private val _installState = MutableStateFlow<InstallState>(InstallState.Idle)
    val installState: StateFlow<InstallState> = _installState.asStateFlow()

    private val listener = SplitInstallStateUpdatedListener { state ->
        when (state.status()) {
            SplitInstallSessionStatus.DOWNLOADING -> {
                val progress = if (state.totalBytesToDownload() > 0) {
                    (state.bytesDownloaded() * 100 / state.totalBytesToDownload()).toInt()
                } else 0
                _installState.value = InstallState.Downloading(progress)
            }
            SplitInstallSessionStatus.INSTALLING ->
                _installState.value = InstallState.Installing

            SplitInstallSessionStatus.INSTALLED ->
                _installState.value = InstallState.Installed

            SplitInstallSessionStatus.REQUIRES_USER_CONFIRMATION ->
                _installState.value = InstallState.RequiresConfirmation(state)

            SplitInstallSessionStatus.FAILED ->
                _installState.value = InstallState.Failed(state.errorCode())
        }
    }

    init {
        splitInstallManager.registerListener(listener)
    }

    fun installPaymentModule() {
        // 이미 설치된 경우 즉시 Installed 상태로 전환
        if (splitInstallManager.installedModules.contains("payment")) {
            _installState.value = InstallState.Installed
            return
        }

        val request = SplitInstallRequest.newBuilder()
            .addModule("payment")
            .build()

        splitInstallManager.startInstall(request)
            .addOnSuccessListener { sessionId ->
                Log.d("DFM", "Install session started: $sessionId")
            }
            .addOnFailureListener { exception ->
                _installState.value = InstallState.Failed(-1)
                Log.e("DFM", "Install failed", exception)
            }
    }

    // Wi-Fi 연결 시 미리 백그라운드 다운로드를 예약하는 선제적 전략
    fun prefetchPaymentModule() {
        if (!splitInstallManager.installedModules.contains("payment")) {
            splitInstallManager.deferredInstall(listOf("payment"))
        }
    }

    fun uninstallPaymentModule() {
        splitInstallManager.deferredUninstall(listOf("payment"))
    }

    override fun onCleared() {
        splitInstallManager.unregisterListener(listener)
    }
}

sealed class InstallState {
    object Idle : InstallState()
    data class Downloading(val progress: Int) : InstallState()
    object Installing : InstallState()
    object Installed : InstallState()
    data class RequiresConfirmation(val state: SplitInstallSessionState) : InstallState()
    data class Failed(val errorCode: Int) : InstallState()
}
```

### 3.3 Compose UI와 연동한 설치 흐름

```kotlin
@Composable
fun HomeScreen(
    viewModel: DynamicFeatureViewModel = viewModel(),
    activity: ComponentActivity
) {
    val installState by viewModel.installState.collectAsStateWithLifecycle()

    // RequiresUserConfirmation: 10MB 초과 모듈 다운로드 시 사용자 동의 다이얼로그 표시
    LaunchedEffect(installState) {
        if (installState is InstallState.RequiresConfirmation) {
            val state = (installState as InstallState.RequiresConfirmation).state
            SplitInstallManagerFactory.create(activity)
                .startConfirmationDialogForResult(state, activity, REQUEST_CODE)
        }
    }

    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(16.dp),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        when (val state = installState) {
            is InstallState.Idle ->
                Button(onClick = { viewModel.installPaymentModule() }) {
                    Text("결제 기능 로드")
                }

            is InstallState.Downloading ->
                Column(horizontalAlignment = Alignment.CenterHorizontally) {
                    CircularProgressIndicator(progress = state.progress / 100f)
                    Spacer(Modifier.height(8.dp))
                    Text("다운로드 중... ${state.progress}%")
                }

            is InstallState.Installing ->
                CircularProgressIndicator()

            is InstallState.Installed -> {
                Text("결제 모듈 로드 완료!", color = MaterialTheme.colorScheme.primary)
                Spacer(Modifier.height(12.dp))
                Button(onClick = { /* PaymentActivity 실행 */ }) {
                    Text("결제 화면으로 이동")
                }
            }

            is InstallState.Failed ->
                Text(
                    "오류 발생 (코드: ${state.errorCode})",
                    color = MaterialTheme.colorScheme.error
                )

            else -> {}
        }
    }
}
```

### 3.4 SplitCompat 적용으로 재시작 없이 모듈 사용

동적 모듈이 설치된 직후 앱 재시작 없이 코드와 리소스를 사용하려면 `SplitCompat`을 반드시 적용해야 합니다.

```kotlin
// Application에 전역 적용
class MyApplication : Application() {
    override fun attachBaseContext(base: Context) {
        super.attachBaseContext(base)
        SplitCompat.install(this)
    }
}

// 동적 모듈의 Activity에도 개별 적용
class PaymentActivity : AppCompatActivity() {
    override fun attachBaseContext(newBase: Context) {
        super.attachBaseContext(newBase)
        SplitCompat.installActivity(this)
    }
}

// Base 모듈에서 동적 모듈의 Activity를 클래스 이름으로 실행
fun launchPaymentActivity(context: Context) {
    try {
        val intent = Intent().setClassName(
            context.packageName,
            "com.example.app.payment.PaymentActivity"
        )
        context.startActivity(intent)
    } catch (e: ActivityNotFoundException) {
        // 모듈이 아직 설치되지 않은 경우 처리
        Log.e("DFM", "Payment module not installed yet", e)
    }
}
```

---

## 4. 주의사항 및 실전 팁

### 4.1 SplitCompat을 절대 빠뜨리지 말 것

`SplitCompat`을 Application, Activity 어느 하나라도 누락하면 설치 직후 모듈의 클래스나 리소스를 로드할 때 `ClassNotFoundException`, `Resources.NotFoundException` 같은 런타임 오류가 발생합니다. Application 전역 설정만으로는 부족하고, 동적 모듈에서 실행되는 모든 Activity에 `attachBaseContext`를 개별 적용해야 합니다.

### 4.2 모듈 수 제한 준수

구글 플레이는 하나의 앱 번들에서 최대 약 50개의 동적 기능 모듈을 권장합니다. 또한 `dist:removable="true"`로 설정된 제거 가능한 모듈은 10개 이하로 유지하는 것이 좋습니다. 과도한 모듈 분리는 빌드 복잡성과 관리 부담을 높이며, 동적 모듈 간 리소스 공유 시 네임스페이스 충돌에도 주의해야 합니다.

### 4.3 로컬 테스트: bundletool 활용

Play Store를 거치지 않고 로컬에서 App Bundle을 테스트하려면 구글의 `bundletool`을 사용합니다.

```bash
# Debug App Bundle 빌드
./gradlew bundleDebug

# 로컬 테스트 모드로 기기에 설치할 APK 세트 생성
bundletool build-apks \
  --bundle=app/build/outputs/bundle/debug/app-debug.aab \
  --output=app.apks \
  --local-testing

# 연결된 기기에 설치
bundletool install-apks --apks=app.apks
```

`--local-testing` 플래그를 사용하면 구글 플레이 없이도 `SplitInstallManager`가 정상 동작합니다. 이 플래그 없이는 디버그 빌드에서 `SplitInstallManager.startInstall()`이 항상 실패합니다.

### 4.4 에러 코드 핸들링

`SplitInstallSessionStatus.FAILED` 상태의 `errorCode`에는 다양한 원인이 있습니다.

| 에러 코드 | 의미 | 권장 처리 |
|---|---|---|
| `SplitInstallErrorCode.NETWORK_ERROR` | 네트워크 오류 | 재시도 UI 제공 |
| `SplitInstallErrorCode.MODULE_UNAVAILABLE` | 해당 기기에서 모듈 미지원 | 기능 비활성화 |
| `SplitInstallErrorCode.INSUFFICIENT_STORAGE` | 저장 공간 부족 | 사용자 알림 |
| `SplitInstallErrorCode.ACTIVE_SESSIONS_LIMIT_EXCEEDED` | 동시 세션 초과 | 잠시 후 재시도 |

### 4.5 deferredInstall로 선제적 캐싱 전략

사용자가 아직 요청하지 않았지만 곧 사용할 가능성이 높은 모듈은 `deferredInstall()`로 백그라운드 다운로드를 미리 예약할 수 있습니다. Wi-Fi 연결 시점이나 앱 백그라운드 진입 시 호출하는 것이 이상적입니다. 이 방식은 사용자 경험을 해치지 않으면서 필요한 순간에 즉시 기능을 제공하는 최적 전략입니다.

```kotlin
// NetworkCallback에서 Wi-Fi 연결 시 미리 다운로드 예약
val networkCallback = object : ConnectivityManager.NetworkCallback() {
    override fun onCapabilitiesChanged(
        network: Network,
        capabilities: NetworkCapabilities
    ) {
        if (capabilities.hasTransport(NetworkCapabilities.TRANSPORT_WIFI)) {
            viewModel.prefetchPaymentModule()
        }
    }
}
```

### 4.6 Instant 모듈의 크기 제한

`dist:instant="true"`로 설정된 인스턴트 모듈은 Google Play Instant를 통해 앱 설치 없이 체험할 수 있습니다. 다만 인스턴트 모듈은 base 모듈 포함 전체 크기가 **15MB**를 초과할 수 없다는 엄격한 제약이 있습니다. 이 한계를 초과하면 구글 플레이 콘솔에서 업로드 자체가 거부됩니다.

---

## 참고 자료
- [Overview of Play Feature Delivery - Android Developers](https://developer.android.com/guide/playcore/feature-delivery)
- [Configure on demand delivery - Android Developers](https://developer.android.com/guide/playcore/feature-delivery/on-demand)
- [SplitInstallManager API Reference - Android Developers](https://developer.android.com/reference/com/google/android/play/core/splitinstall/SplitInstallManager)
