---
layout: post
title: "Flutter Add-to-App 심화: FlutterEngine·FlutterActivity·FlutterFragment로 기존 Android 앱에 Flutter를 점진적으로 도입하는 완전 가이드"
date: 2026-09-28
categories: [android, flutter]
tags: [flutter, android, add-to-app, flutterengine, flutteractivity, flutterfragment, flutter-module, kotlin, dart, migration]
---

대규모 레거시 Android 앱을 완전히 Flutter로 재작성하는 것은 현실적으로 불가능한 경우가 많습니다. 기존 팀의 역량, 비즈니스 일정, 검증되지 않은 위험 부담이 동시에 쌓입니다. **Add-to-App**은 이 딜레마를 풀어주는 Flutter 공식 전략입니다. 기존 Android 코드베이스를 그대로 유지하면서 일부 화면이나 기능만 Flutter로 점진적으로 교체할 수 있습니다.

이 글에서는 단순한 "Hello World" 수준을 넘어, 실제 프로덕션에서 Add-to-App을 안정적으로 운영하기 위한 FlutterEngine 생명주기 관리, 성능 최적화, 네이티브↔Flutter 통신 패턴까지 깊이 있게 다룹니다.

## Add-to-App이란 무엇인가

Flutter의 Add-to-App은 **기존 Native Android(또는 iOS) 앱 안에 Flutter 모듈을 라이브러리 형태로 삽입**하는 방식입니다. 일반적인 Flutter 앱과 달리 `main()` 진입점이 없고, 대신 `flutter create --template=module` 명령으로 생성된 Flutter 모듈이 별도의 Gradle 모듈 또는 AAR 파일로 Host 앱에 포함됩니다.

### 왜 Add-to-App이 필요한가

1. **단계적 마이그레이션**: 수년간 쌓인 네이티브 코드를 한 번에 재작성하지 않고, 새 기능부터 Flutter로 추가할 수 있습니다.
2. **리스크 분산**: Flutter 화면과 네이티브 화면이 공존하므로, Flutter 버그가 전체 앱을 망가뜨리지 않습니다.
3. **팀 병행 개발**: Flutter 팀과 Android 팀이 독립적으로 작업하고 각자의 빌드 산출물(AAR)을 통합할 수 있습니다.
4. **코드 재사용**: Dart로 작성한 비즈니스 로직을 KMP(Kotlin Multiplatform) 없이도 iOS와 Android 모두에서 공유할 수 있습니다.

---

## 프로젝트 구조 설정

### Flutter 모듈 생성

```bash
# Host Android 앱과 같은 레벨에 Flutter 모듈 생성
flutter create --template=module my_flutter_module

# 디렉터리 구조
.
├── my_android_app/          # 기존 Android 앱
│   ├── app/
│   └── settings.gradle.kts
└── my_flutter_module/       # Flutter 모듈
    ├── lib/
    │   └── main.dart
    ├── .android/            # 자동 생성 (직접 수정 금지)
    └── pubspec.yaml
```

### Android 앱에 Flutter 모듈 통합 (소스 의존성 방식)

`settings.gradle.kts`에 Flutter 모듈 경로를 추가합니다.

```kotlin
// my_android_app/settings.gradle.kts
pluginManagement {
    repositories {
        gradlePluginPortal()
        google()
        mavenCentral()
    }
}

// Flutter 모듈의 .android 디렉터리를 Gradle 프로젝트에 포함
val flutterProjectRoot = rootProject.projectDir.parentFile.toPath()
val flutterModulePath = flutterProjectRoot.resolve("my_flutter_module/.android/include_flutter.groovy")

apply(from = flutterModulePath.toFile())

include(":app")
```

그리고 앱 모듈의 `build.gradle.kts`에 의존성을 추가합니다.

```kotlin
// my_android_app/app/build.gradle.kts
dependencies {
    implementation(project(":flutter"))
    // 기타 의존성
}
```

> **AAR 방식**: 팀이 분리되어 있거나 Flutter 빌드 환경 없이 Android 팀만 작업할 때는 `flutter build aar` 명령으로 생성된 AAR 파일을 Maven 저장소에 배포해 사용합니다. 소스 의존성 방식보다 빌드 시간이 짧지만 Flutter 코드 수정 사이클이 느립니다.

---

## FlutterEngine: Add-to-App의 핵심

### FlutterEngine의 생명주기와 비용

`FlutterEngine`은 Dart VM, Flutter 렌더링 파이프라인, 플러그인 레지스트리를 포함하는 **무거운 객체**입니다. 기본적으로 `FlutterActivity`나 `FlutterFragment`를 열 때마다 새로운 `FlutterEngine`이 생성되는데, 이 초기화 과정에는 **수백 밀리초**가 소요됩니다. 사용자 입장에서는 화면이 열릴 때 눈에 띄는 지연이 느껴집니다.

해결책은 **앱 시작 시 FlutterEngine을 미리 워밍업(pre-warm)하고 캐시에 보관**하는 것입니다.

### FlutterEngine 사전 워밍업 구현

```kotlin
// MyApplication.kt
class MyApplication : Application() {

    companion object {
        const val ENGINE_ID = "main_flutter_engine"
    }

    override fun onCreate() {
        super.onCreate()
        
        // Dart 코드 실행을 위한 FlutterEngine 초기화
        val flutterEngine = FlutterEngine(this)
        
        // 기본 엔트리포인트 실행 (lib/main.dart의 main() 함수)
        flutterEngine.dartExecutor.executeDartEntrypoint(
            DartExecutor.DartEntrypoint.createDefault()
        )
        
        // 특정 초기 라우트 지정 (선택사항)
        // flutterEngine.navigationChannel.setInitialRoute("/product/123")
        
        // FlutterEngineCache에 등록
        FlutterEngineCache.getInstance().put(ENGINE_ID, flutterEngine)
    }
}
```

```xml
<!-- AndroidManifest.xml -->
<application
    android:name=".MyApplication"
    ...>
```

> **중요**: `FlutterEngine`을 캐시에 보관하면 Dart 코드는 `FlutterActivity`/`FlutterFragment`가 화면에 표시되기 전부터 이미 실행 중입니다. 이 점을 활용하면 Flutter 화면이 열리기 전에 데이터를 미리 로드할 수도 있지만, 동시에 메모리 소비가 증가하므로 주의가 필요합니다.

---

## FlutterActivity로 Flutter 화면 띄우기

`FlutterActivity`는 `Activity` 전체를 Flutter로 대체하는 가장 단순한 방식입니다.

### 기본 사용법 (캐시된 엔진 활용)

```kotlin
// SomeNativeActivity.kt
class SomeNativeActivity : AppCompatActivity() {
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_some_native)
        
        binding.openFlutterButton.setOnClickListener {
            startActivity(
                FlutterActivity
                    .withCachedEngine(MyApplication.ENGINE_ID)
                    .build(this)
            )
        }
    }
}
```

```xml
<!-- AndroidManifest.xml: FlutterActivity 등록 -->
<activity
    android:name="io.flutter.embedding.android.FlutterActivity"
    android:theme="@style/LaunchTheme"
    android:configChanges="orientation|keyboardHidden|keyboard|screenSize|smallestScreenSize|locale|layoutDirection|fontScale|screenLayout|density|uiMode"
    android:hardwareAccelerated="true"
    android:windowSoftInputMode="adjustResize"
    android:exported="false" />
```

### 커스텀 FlutterActivity 서브클래스

플러그인 등록이나 엔진 설정을 커스터마이즈하려면 서브클래스를 만듭니다.

```kotlin
// CustomFlutterActivity.kt
class CustomFlutterActivity : FlutterActivity() {

    // 캐시된 엔진 ID 지정
    override fun getCachedEngineId(): String = MyApplication.ENGINE_ID

    // 특정 라우트로 시작
    override fun getInitialRoute(): String {
        return intent.getStringExtra("route") ?: "/"
    }

    // FlutterEngine에 추가 플러그인 등록
    override fun configureFlutterEngine(flutterEngine: FlutterEngine) {
        super.configureFlutterEngine(flutterEngine)
        // 여기서 추가 MethodChannel 설정 등 가능
        setupMethodChannel(flutterEngine)
    }

    private fun setupMethodChannel(flutterEngine: FlutterEngine) {
        MethodChannel(
            flutterEngine.dartExecutor.binaryMessenger,
            "com.example.app/native_bridge"
        ).setMethodCallHandler { call, result ->
            when (call.method) {
                "getUserName" -> {
                    val userName = getUserNameFromSession()
                    result.success(userName)
                }
                "logout" -> {
                    performLogout()
                    result.success(null)
                }
                else -> result.notImplemented()
            }
        }
    }

    private fun getUserNameFromSession(): String = "홍길동" // 실제로는 세션에서 읽음
    private fun performLogout() { /* 로그아웃 처리 */ }
}
```

---

## FlutterFragment: 부분 화면에 Flutter 삽입

`FlutterFragment`는 기존 `Activity`의 **일부 영역**에 Flutter UI를 렌더링합니다. 내비게이션 바, 탭 레이아웃, 또는 화면 하단 절반만 Flutter로 구현해야 할 때 적합합니다.

### FlutterFragment 기본 사용

```kotlin
// HybridActivity.kt
class HybridActivity : AppCompatActivity() {

    private val FLUTTER_FRAGMENT_TAG = "flutter_fragment"

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_hybrid)

        // 이미 존재하면 재사용, 없으면 생성
        val existingFragment = supportFragmentManager
            .findFragmentByTag(FLUTTER_FRAGMENT_TAG) as? FlutterFragment

        if (existingFragment == null) {
            val flutterFragment = FlutterFragment
                .withCachedEngine(MyApplication.ENGINE_ID)
                .shouldAttachEngineToActivity(true)
                .renderMode(FlutterView.RenderMode.surface)   // SurfaceView (성능 우선)
                // .renderMode(FlutterView.RenderMode.texture) // TextureView (Z-order 제어 필요 시)
                .transparencyMode(FlutterView.TransparencyMode.opaque)
                .build<FlutterFragment>()

            supportFragmentManager
                .beginTransaction()
                .add(R.id.flutter_container, flutterFragment, FLUTTER_FRAGMENT_TAG)
                .commit()
        }
    }
}
```

```xml
<!-- activity_hybrid.xml -->
<LinearLayout
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <!-- 네이티브 UI 영역 -->
    <TextView
        android:id="@+id/native_header"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="네이티브 헤더"
        android:padding="16dp" />

    <!-- Flutter UI 삽입 영역 -->
    <FrameLayout
        android:id="@+id/flutter_container"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1" />

</LinearLayout>
```

### RenderMode 선택 기준

| 항목 | `SurfaceView` (기본) | `TextureView` |
|------|---------------------|---------------|
| 성능 | 우수 (하드웨어 오버레이) | 보통 (소프트웨어 합성) |
| Z-order 제어 | 제한적 (항상 최상위 또는 최하위) | 자유로움 (중간 삽입 가능) |
| 투명도 | 불가 | 가능 |
| 권장 상황 | 일반적인 경우 | View 계층 중간 삽입 필요 시 |

---

## Flutter↔Android 양방향 통신

Add-to-App에서 가장 중요한 부분 중 하나는 **네이티브 코드와 Flutter 코드 간의 데이터 교환**입니다.

### MethodChannel: 요청-응답 패턴

```kotlin
// Android 측 (Kotlin) - 채널 설정
val channel = MethodChannel(
    flutterEngine.dartExecutor.binaryMessenger,
    "com.example.app/product_service"
)

// Android → Flutter 호출
channel.invokeMethod(
    "fetchProductDetails",
    mapOf("productId" to "PROD_001"),
    object : MethodChannel.Result {
        override fun success(result: Any?) {
            val productJson = result as? String
            // UI 업데이트
        }
        override fun error(errorCode: String, errorMessage: String?, errorDetails: Any?) {
            Log.e("Flutter", "오류: $errorCode - $errorMessage")
        }
        override fun notImplemented() { /* 미구현 */ }
    }
)

// Flutter → Android 호출 처리
channel.setMethodCallHandler { call, result ->
    when (call.method) {
        "addToCart" -> {
            val productId = call.argument<String>("productId") ?: return@setMethodCallHandler
            val quantity = call.argument<Int>("quantity") ?: 1
            cartRepository.addItem(productId, quantity)
            result.success(true)
        }
        else -> result.notImplemented()
    }
}
```

```dart
// Flutter 측 (Dart)
class ProductService {
  static const _channel = MethodChannel('com.example.app/product_service');

  // Flutter → Android 데이터 요청
  Future<void> addToCart(String productId, int quantity) async {
    try {
      final success = await _channel.invokeMethod<bool>('addToCart', {
        'productId': productId,
        'quantity': quantity,
      });
      if (success == true) {
        debugPrint('장바구니에 추가 완료');
      }
    } on PlatformException catch (e) {
      debugPrint('장바구니 추가 실패: ${e.message}');
    }
  }

  // Android로부터 호출받기
  void listenForNativeEvents() {
    _channel.setMethodCallHandler((call) async {
      if (call.method == 'fetchProductDetails') {
        final productId = call.arguments['productId'] as String;
        final product = await _localRepository.getProduct(productId);
        return jsonEncode(product.toJson());
      }
      throw PlatformException(code: 'NOT_IMPLEMENTED');
    });
  }
}
```

### EventChannel: 연속 데이터 스트림

Android의 위치 업데이트, 센서 데이터, 소켓 메시지 등 **지속적인 스트림**을 Flutter로 전달할 때 사용합니다.

```kotlin
// Android 측 - 위치 스트림 제공
EventChannel(
    flutterEngine.dartExecutor.binaryMessenger,
    "com.example.app/location_stream"
).setStreamHandler(object : EventChannel.StreamHandler {
    private var locationCallback: LocationCallback? = null

    override fun onListen(arguments: Any?, events: EventChannel.EventSink) {
        locationCallback = object : LocationCallback() {
            override fun onLocationResult(result: LocationResult) {
                val location = result.lastLocation ?: return
                // UI 스레드에서 호출 보장
                Handler(Looper.getMainLooper()).post {
                    events.success(mapOf(
                        "latitude" to location.latitude,
                        "longitude" to location.longitude,
                        "accuracy" to location.accuracy
                    ))
                }
            }
        }
        fusedLocationClient.requestLocationUpdates(
            locationRequest, locationCallback!!, Looper.getMainLooper()
        )
    }

    override fun onCancel(arguments: Any?) {
        locationCallback?.let { fusedLocationClient.removeLocationUpdates(it) }
        locationCallback = null
    }
})
```

```dart
// Flutter 측 - 위치 스트림 구독
class LocationWidget extends StatefulWidget {
  const LocationWidget({super.key});

  @override
  State<LocationWidget> createState() => _LocationWidgetState();
}

class _LocationWidgetState extends State<LocationWidget> {
  static const _channel = EventChannel('com.example.app/location_stream');
  StreamSubscription? _subscription;
  Map<String, dynamic>? _location;

  @override
  void initState() {
    super.initState();
    _subscription = _channel
        .receiveBroadcastStream()
        .listen((event) {
          setState(() {
            _location = Map<String, dynamic>.from(event as Map);
          });
        }, onError: (error) {
          debugPrint('위치 오류: $error');
        });
  }

  @override
  void dispose() {
    _subscription?.cancel();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final lat = _location?['latitude']?.toStringAsFixed(6) ?? '-';
    final lng = _location?['longitude']?.toStringAsFixed(6) ?? '-';
    return Text('현재 위치: $lat, $lng');
  }
}
```

---

## 복수 FlutterEngine 운용과 메모리 관리

한 앱에서 여러 Flutter 화면을 독립적으로 실행해야 할 때(예: 독립된 상태의 상품 상세 화면과 채팅 화면) **복수의 FlutterEngine**을 사용할 수 있습니다.

```kotlin
// MultiEngineApplication.kt
class MultiEngineApplication : Application() {

    companion object {
        const val PRODUCT_ENGINE_ID = "product_engine"
        const val CHAT_ENGINE_ID = "chat_engine"
    }

    override fun onCreate() {
        super.onCreate()
        
        // 상품 화면용 엔진 (즉시 워밍업)
        createAndCacheEngine(PRODUCT_ENGINE_ID, "/product")
        
        // 채팅 화면용 엔진은 실제 필요 시 생성 (지연 초기화)
    }

    fun ensureChatEngineReady() {
        if (FlutterEngineCache.getInstance().contains(CHAT_ENGINE_ID)) return
        createAndCacheEngine(CHAT_ENGINE_ID, "/chat")
    }

    private fun createAndCacheEngine(id: String, route: String) {
        val engine = FlutterEngine(this)
        engine.navigationChannel.setInitialRoute(route)
        engine.dartExecutor.executeDartEntrypoint(
            DartExecutor.DartEntrypoint.createDefault()
        )
        FlutterEngineCache.getInstance().put(id, engine)
    }

    // 앱 메모리 부족 시 엔진 해제
    override fun onLowMemory() {
        super.onLowMemory()
        // 사용 중이지 않은 엔진만 해제
        FlutterEngineCache.getInstance().get(CHAT_ENGINE_ID)?.destroy()
        FlutterEngineCache.getInstance().remove(CHAT_ENGINE_ID)
    }
}
```

> **주의**: 각 `FlutterEngine`은 약 40~60MB의 메모리를 사용합니다. 저사양 기기에서는 2개 이상의 엔진을 동시에 유지하면 OOM(Out of Memory) 위험이 높아집니다. 반드시 `onLowMemory()`와 `onTrimMemory()`에서 미사용 엔진을 해제하는 전략을 구현하세요.

---

## Dart 측 진입점 분리 (`@pragma('vm:entry-point')`)

하나의 Flutter 모듈에서 여러 네이티브 엔트리포인트를 지원하려면 Dart에서 `@pragma` 애노테이션을 사용합니다.

```dart
// lib/main.dart
import 'package:flutter/material.dart';

// 기본 진입점 (상품 화면)
void main() {
  runApp(const ProductApp());
}

// 채팅 진입점
@pragma('vm:entry-point')
void chatMain() {
  runApp(const ChatApp());
}

// 독립 실행 위젯 진입점
@pragma('vm:entry-point')
void miniWidgetMain() {
  runApp(const MiniWidget());
}

class ProductApp extends StatelessWidget {
  const ProductApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: '상품',
      home: const ProductListScreen(),
    );
  }
}
```

```kotlin
// Android 측에서 특정 Dart 진입점 사용
val chatEngine = FlutterEngine(this)
chatEngine.dartExecutor.executeDartEntrypoint(
    DartExecutor.DartEntrypoint(
        FlutterInjector.instance().flutterLoader().findAppBundlePath(),
        "chatMain"  // Dart 함수명
    )
)
```

---

## 주의사항 및 실전 팁

### 1. 플러그인 호환성 확인 필수
모든 Flutter 플러그인이 Add-to-App을 지원하는 것은 아닙니다. 특히 `GeneratedPluginRegistrant`가 Add-to-App 환경에서 올바르게 동작하는지 확인하세요. 공식적으로 지원하지 않는 플러그인은 네이티브 코드와 MethodChannel로 직접 연동해야 합니다.

### 2. 첫 프레임 렌더링 전 블랙 스크린 방지
FlutterEngine이 사전 워밍업되어 있더라도 첫 프레임 렌더링에 약간의 시간이 필요합니다. 전환 시 스플래시 화면 또는 스켈레톤 UI를 네이티브로 먼저 보여주고, Flutter 준비가 완료되면 전환하는 패턴을 권장합니다.

```kotlin
// FlutterActivity를 열기 전 스플래시를 네이티브로 표시
binding.loadingView.isVisible = true
Handler(Looper.getMainLooper()).postDelayed({
    binding.loadingView.isVisible = false
    startActivity(FlutterActivity.withCachedEngine(ENGINE_ID).build(this))
}, 300) // 엔진이 충분히 준비될 시간
```

### 3. 백 스택과 생명주기 관리
`FlutterFragment`를 사용하는 `Activity`에서 `onBackPressedDispatcher`를 올바르게 위임하지 않으면 Flutter의 팝 내비게이션이 동작하지 않습니다.

```kotlin
class HybridActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // Flutter의 back 처리를 Activity에 위임
        onBackPressedDispatcher.addCallback(this) {
            val fragment = supportFragmentManager
                .findFragmentByTag(FLUTTER_FRAGMENT_TAG) as? FlutterFragment
            
            // Flutter가 back을 처리할 수 있으면 Flutter에 위임
            if (fragment?.onBackPressed() == false) {
                isEnabled = false
                onBackPressedDispatcher.onBackPressed()
            }
        }
    }
}
```

### 4. 빌드 캐시 활용으로 빌드 시간 단축
Add-to-App은 일반 Flutter 앱보다 빌드가 복잡합니다. `flutter build aar --debug` 결과를 로컬 Maven 저장소에 캐시하면 Android 팀은 Flutter 환경 없이 개발할 수 있습니다.

```kotlin
// app/build.gradle.kts - 로컬 AAR 사용
repositories {
    maven {
        url = uri("${rootProject.projectDir}/../my_flutter_module/build/host/outputs/repo")
    }
}

dependencies {
    debugImplementation("com.example:flutter_debug:1.0")
    releaseImplementation("com.example:flutter_release:1.0")
}
```

### 5. ProGuard/R8 규칙
Add-to-App 빌드에서 Flutter 클래스가 난독화되지 않도록 규칙을 추가하세요.

```proguard
# proguard-rules.pro
-keep class io.flutter.** { *; }
-keep class io.flutter.plugins.** { *; }
-dontwarn io.flutter.**
```

---

## 마이그레이션 전략: 점진적 도입 로드맵

Add-to-App을 실제 프로젝트에 도입할 때 권장되는 단계별 전략입니다.

1. **1단계 (파일럿)**: 신규 기능 하나를 Flutter로 개발하고 Add-to-App으로 삽입합니다. 빌드 파이프라인, CI/CD, 팀 워크플로를 검증합니다.
2. **2단계 (확장)**: 검증된 패턴으로 더 많은 화면을 Flutter로 이관합니다. MethodChannel 계약을 명확히 문서화합니다.
3. **3단계 (통합)**: 기존 네이티브 화면이 점점 줄어들면, 결국 네이티브 화면이 Add-to-App의 껍데기가 됩니다. 이 시점에 전체 Flutter 앱으로 전환을 검토합니다.

---

## 마무리

Flutter Add-to-App은 **"지금 당장 전체를 바꾸기 어렵다"는 현실적인 제약**을 인정하면서도 Flutter의 생산성과 성능을 점진적으로 누릴 수 있는 강력한 전략입니다. 핵심은 `FlutterEngine`을 앱 시작 시 사전 워밍업하여 성능 지연을 없애고, MethodChannel/EventChannel로 명확한 계약 기반의 양방향 통신을 설계하며, 메모리 관리에 세심하게 신경 쓰는 것입니다.

레거시 앱을 가진 팀이라면 작은 기능 하나부터 Add-to-App으로 시작해 보세요. 그 경험이 팀 전체의 Flutter 전환에 대한 확신을 만들어 줄 것입니다.

## 참고 자료
- [Add Flutter to existing Android app — Flutter Docs](https://docs.flutter.dev/add-to-app)
- [Add a Flutter Fragment to an Android app — Flutter Docs](https://docs.flutter.dev/add-to-app/android/add-flutter-fragment)
- [Android Fragments — Android Developers](https://developer.android.com/guide/components/fragments)
- [Flutter GitHub Wiki: Add Flutter to existing apps](https://github.com/flutter/flutter/wiki/Add-Flutter-to-existing-apps)
