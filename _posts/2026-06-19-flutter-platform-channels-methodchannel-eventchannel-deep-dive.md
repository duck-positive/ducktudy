---
layout: post
title: "Flutter Platform Channels 심화: MethodChannel·EventChannel·BasicMessageChannel 네이티브 통신 완전 정복"
date: 2026-06-19
categories: [android, flutter]
tags: [flutter, platform-channels, methodchannel, eventchannel, basicmessagechannel, kotlin, native, android]
---

Flutter는 "하나의 코드베이스로 모든 플랫폼"이라는 슬로건을 내세우지만, 현실 세계의 앱은 반드시 플랫폼 고유의 API를 호출해야 하는 순간이 찾아옵니다. 배터리 정보, 센서 스트림, Bluetooth, 생체 인증, 카메라 HAL…. 이 모든 기능은 Flutter 엔진이 직접 제공하지 않으며, Android SDK 혹은 iOS SDK를 통해서만 접근할 수 있습니다. 이때 Flutter와 네이티브 코드를 잇는 다리가 바로 **Platform Channels**입니다.

이 글에서는 세 가지 채널(MethodChannel, EventChannel, BasicMessageChannel)의 내부 동작 원리를 파악하고, Dart와 Kotlin 코드를 직접 작성하면서 실전 감각을 익혀봅니다.

---

## 왜 Platform Channels가 필요한가

Flutter 엔진 자체는 Dart VM 위에서 실행됩니다. 그런데 Android의 배터리 상태(`BatteryManager`), 가속도 센서(`SensorManager`), Wi-Fi 상태(`ConnectivityManager`) 같은 API는 Android SDK에만 존재합니다. Flutter 패키지 생태계가 풍부하긴 하지만, 기업 내부 라이브러리, 레거시 SDK, 혹은 아직 pub.dev에 없는 신규 API를 써야 할 때는 직접 브릿지를 만들어야 합니다.

Platform Channels는 **비동기 메시지 전달(Asynchronous Message Passing)** 방식을 사용합니다. Dart 측에서 메시지를 보내면 Flutter 엔진이 JNI(Android) 또는 Obj-C 런타임(iOS)을 거쳐 플랫폼 스레드로 전달하고, 결과를 다시 Dart로 돌려줍니다. 이 과정이 모두 비동기로 처리되기 때문에 UI 스레드가 블로킹되지 않습니다.

### 세 가지 채널 한눈에 비교

| 채널 | 통신 방향 | 대표 용도 |
|------|-----------|-----------|
| `MethodChannel` | Dart → Native (응답 반환) | 단발성 API 호출 (배터리 수준, 권한 요청) |
| `EventChannel` | Native → Dart (스트림) | 지속적 데이터 (센서, GPS, 네트워크 변화) |
| `BasicMessageChannel` | 양방향 | 커스텀 코덱이 필요한 메시지 교환 |

---

## 1. MethodChannel: 단발성 네이티브 호출

### 개념

`MethodChannel`은 가장 많이 쓰이는 채널입니다. Dart에서 메서드 이름과 인자를 넘기면 네이티브가 이를 처리하고 결과를 반환합니다. 마치 원격 프로시저 호출(RPC)처럼 동작합니다.

### Dart 측 구현

```dart
import 'package:flutter/services.dart';

class BatteryService {
  static const _channel = MethodChannel('dev.ducktudy/battery');

  Future<int> getBatteryLevel() async {
    try {
      final level = await _channel.invokeMethod<int>('getBatteryLevel');
      return level ?? -1;
    } on PlatformException catch (e) {
      // 네이티브에서 던진 예외를 Dart에서 캐치
      debugPrint('배터리 조회 실패: ${e.code} - ${e.message}');
      rethrow;
    }
  }

  Future<String> getDeviceInfo() async {
    // 인자를 Map으로 전달하는 예
    final result = await _channel.invokeMapMethod<String, dynamic>(
      'getDeviceInfo',
      {'includeModel': true, 'includeSdk': true},
    );
    return '${result?['model']} (SDK ${result?['sdk']})';
  }
}
```

채널 이름(`dev.ducktudy/battery`)은 앱 전역에서 고유해야 합니다. 관례상 `{도메인}/{기능}` 형태의 역-DNS 스타일을 사용합니다.

### Android(Kotlin) 측 구현

```kotlin
// android/app/src/main/kotlin/.../MainActivity.kt
import android.content.Context
import android.content.Intent
import android.content.IntentFilter
import android.os.BatteryManager
import android.os.Build
import io.flutter.embedding.android.FlutterActivity
import io.flutter.embedding.engine.FlutterEngine
import io.flutter.plugin.common.MethodChannel

class MainActivity : FlutterActivity() {

    private val CHANNEL = "dev.ducktudy/battery"

    override fun configureFlutterEngine(flutterEngine: FlutterEngine) {
        super.configureFlutterEngine(flutterEngine)

        MethodChannel(flutterEngine.dartExecutor.binaryMessenger, CHANNEL)
            .setMethodCallHandler { call, result ->
                when (call.method) {
                    "getBatteryLevel" -> {
                        val level = getBatteryLevel()
                        if (level != -1) {
                            result.success(level)
                        } else {
                            result.error(
                                "UNAVAILABLE",
                                "배터리 정보를 가져올 수 없습니다.",
                                null
                            )
                        }
                    }
                    "getDeviceInfo" -> {
                        val includeModel = call.argument<Boolean>("includeModel") ?: false
                        val includeSdk   = call.argument<Boolean>("includeSdk")   ?: false
                        val info = mutableMapOf<String, Any>()
                        if (includeModel) info["model"] = Build.MODEL
                        if (includeSdk)   info["sdk"]   = Build.VERSION.SDK_INT
                        result.success(info)
                    }
                    else -> result.notImplemented()
                }
            }
    }

    private fun getBatteryLevel(): Int {
        return if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
            val bm = getSystemService(Context.BATTERY_SERVICE) as BatteryManager
            bm.getIntProperty(BatteryManager.BATTERY_PROPERTY_CAPACITY)
        } else {
            val intent = ContextWrapper(applicationContext)
                .registerReceiver(null, IntentFilter(Intent.ACTION_BATTERY_CHANGED))
            val level  = intent?.getIntExtra(BatteryManager.EXTRA_LEVEL, -1) ?: -1
            val scale  = intent?.getIntExtra(BatteryManager.EXTRA_SCALE, -1) ?: -1
            if (scale == 0) -1 else (level * 100 / scale)
        }
    }
}
```

네이티브 콜백(`setMethodCallHandler`)은 **플랫폼 메인 스레드**에서 실행됩니다. 따라서 오래 걸리는 작업은 코루틴이나 스레드풀로 오프로드하고, 완료 후 `result.success()`를 호출해야 합니다.

---

## 2. EventChannel: 지속적인 네이티브 스트림

### 개념

배터리 수준처럼 단발로 끝나는 값이 아니라, 가속도 센서나 네트워크 상태처럼 **지속적으로 변하는 데이터**를 Dart로 흘려보내야 할 때 `EventChannel`을 사용합니다. Dart 측에서는 `Stream`으로 데이터를 받을 수 있습니다.

### Dart 측 구현

```dart
import 'package:flutter/services.dart';

class AccelerometerService {
  static const _channel = EventChannel('dev.ducktudy/accelerometer');

  // 스트림은 lazy하게 생성됨 — listen() 호출 시점에 네이티브 구독 시작
  Stream<Map<String, double>> get accelerometerStream {
    return _channel
        .receiveBroadcastStream()
        .map((event) {
          final data = Map<String, dynamic>.from(event as Map);
          return {
            'x': (data['x'] as num).toDouble(),
            'y': (data['y'] as num).toDouble(),
            'z': (data['z'] as num).toDouble(),
          };
        });
  }
}

// 사용 예시
class SensorScreen extends StatefulWidget {
  const SensorScreen({super.key});

  @override
  State<SensorScreen> createState() => _SensorScreenState();
}

class _SensorScreenState extends State<SensorScreen> {
  final _service = AccelerometerService();
  StreamSubscription? _subscription;
  Map<String, double> _values = {'x': 0, 'y': 0, 'z': 0};

  @override
  void initState() {
    super.initState();
    _subscription = _service.accelerometerStream.listen(
      (data) => setState(() => _values = data),
      onError: (error) => debugPrint('센서 오류: $error'),
    );
  }

  @override
  void dispose() {
    _subscription?.cancel(); // 반드시 구독 해제
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('가속도 센서')),
      body: Center(
        child: Text(
          'X: ${_values['x']!.toStringAsFixed(3)}\n'
          'Y: ${_values['y']!.toStringAsFixed(3)}\n'
          'Z: ${_values['z']!.toStringAsFixed(3)}',
          style: Theme.of(context).textTheme.headlineMedium,
          textAlign: TextAlign.center,
        ),
      ),
    );
  }
}
```

### Android(Kotlin) 측 구현

```kotlin
import android.content.Context
import android.hardware.Sensor
import android.hardware.SensorEvent
import android.hardware.SensorEventListener
import android.hardware.SensorManager
import io.flutter.embedding.android.FlutterActivity
import io.flutter.embedding.engine.FlutterEngine
import io.flutter.plugin.common.EventChannel

class MainActivity : FlutterActivity() {

    private val EVENT_CHANNEL = "dev.ducktudy/accelerometer"
    private var sensorManager: SensorManager? = null
    private var sensorEventListener: SensorEventListener? = null

    override fun configureFlutterEngine(flutterEngine: FlutterEngine) {
        super.configureFlutterEngine(flutterEngine)

        sensorManager = getSystemService(Context.SENSOR_SERVICE) as SensorManager

        EventChannel(flutterEngine.dartExecutor.binaryMessenger, EVENT_CHANNEL)
            .setStreamHandler(object : EventChannel.StreamHandler {

                override fun onListen(arguments: Any?, events: EventChannel.EventSink) {
                    val sensor = sensorManager
                        ?.getDefaultSensor(Sensor.TYPE_ACCELEROMETER)

                    if (sensor == null) {
                        events.error("NO_SENSOR", "가속도 센서가 없습니다.", null)
                        return
                    }

                    sensorEventListener = object : SensorEventListener {
                        override fun onSensorChanged(event: SensorEvent) {
                            // UI 스레드에서 호출되므로 바로 sink로 전달
                            events.success(mapOf(
                                "x" to event.values[0],
                                "y" to event.values[1],
                                "z" to event.values[2]
                            ))
                        }
                        override fun onAccuracyChanged(sensor: Sensor, accuracy: Int) {}
                    }

                    sensorManager?.registerListener(
                        sensorEventListener,
                        sensor,
                        SensorManager.SENSOR_DELAY_UI
                    )
                }

                override fun onCancel(arguments: Any?) {
                    // Dart 측에서 cancel() 호출 시 리소스 해제
                    sensorManager?.unregisterListener(sensorEventListener)
                    sensorEventListener = null
                }
            })
    }
}
```

`onCancel()`은 Dart에서 `StreamSubscription.cancel()`을 호출하거나 위젯이 `dispose`될 때 실행됩니다. 여기서 리스너를 반드시 해제해야 배터리 소모를 방지할 수 있습니다.

---

## 3. BasicMessageChannel: 양방향 커스텀 메시지

`BasicMessageChannel`은 `MethodChannel`이나 `EventChannel`로 표현하기 어려운 **양방향 대화형 메시지**에 적합합니다. 주로 커스텀 직렬화 코덱(`StandardMessageCodec`, `JSONMessageCodec`, `BinaryCodec`, `StringCodec`)과 함께 사용합니다.

```dart
// Dart 측
final channel = BasicMessageChannel<String>(
  'dev.ducktudy/chat',
  StringCodec(),
);

// 네이티브로 보내고 응답 받기
Future<void> sendMessage(String text) async {
  final reply = await channel.send(text);
  debugPrint('네이티브 응답: $reply');
}

// 네이티브에서 오는 메시지 받기
void setupReceiver() {
  channel.setMessageHandler((message) async {
    debugPrint('네이티브 → Dart: $message');
    return '수신 확인: $message';
  });
}
```

```kotlin
// Kotlin 측
val basicChannel = BasicMessageChannel<String>(
    flutterEngine.dartExecutor.binaryMessenger,
    "dev.ducktudy/chat",
    StringCodec.INSTANCE
)

basicChannel.setMessageHandler { message, reply ->
    Log.d("BasicChannel", "Dart → Native: $message")
    reply.reply("네이티브 응답: $message")
}

// 네이티브에서 Dart로 먼저 메시지 보내기
basicChannel.send("안녕 Dart!") { response ->
    Log.d("BasicChannel", "Dart 응답: $response")
}
```

---

## 4. 주의사항 및 실전 팁

### 채널 이름 충돌 방지
앱이 여러 플러그인을 사용하면 채널 이름이 충돌할 수 있습니다. 반드시 **역-DNS 표기법**으로 고유한 이름을 지정하세요. 예: `com.mycompany.myapp/battery`.

### 스레드 안전성
`result.success()` / `result.error()` / `events.success()`는 반드시 **플랫폼 메인 스레드**에서 호출해야 합니다. 백그라운드 스레드에서 처리했다면 `Handler(Looper.getMainLooper()).post { result.success(data) }` 패턴을 사용하거나, Kotlin 코루틴에서 `withContext(Dispatchers.Main)`으로 전환하세요.

```kotlin
// 코루틴에서 백그라운드 작업 후 메인 스레드로 전환 예시
"heavyTask" -> {
    lifecycleScope.launch {
        val data = withContext(Dispatchers.IO) {
            performHeavyComputation()
        }
        result.success(data) // 코루틴이 Main dispatcher로 돌아온 후 호출
    }
}
```

### result는 정확히 한 번만 호출
`result.success()`, `result.error()`, `result.notImplemented()` 중 하나는 **반드시 단 한 번**만 호출해야 합니다. 두 번 호출하면 `IllegalStateException`이 발생합니다. 조건 분기가 복잡하다면 `try-finally`로 보장하는 것이 안전합니다.

### null 안전성과 타입 변환
네이티브에서 `Map`을 반환할 때 Dart에서 `Map<dynamic, dynamic>`으로 수신됩니다. 반드시 `Map<String, dynamic>.from(raw)`로 캐스팅하고, `num`으로 오는 숫자 값은 `.toDouble()` 또는 `.toInt()`로 명시적 변환하세요.

### FlutterEngine 생명주기와 채널 등록
`configureFlutterEngine()`은 `FlutterActivity` 또는 `FlutterFragment`가 엔진을 초기화할 때 한 번 호출됩니다. 여러 화면에서 같은 채널을 사용한다면 `FlutterEngineCache`를 이용해 엔진을 공유하세요. 동일한 채널 이름으로 `setMethodCallHandler()`를 두 번 호출하면 두 번째 핸들러가 첫 번째를 덮어씁니다.

### 통합 테스트
`MethodChannel`은 `TestWidgetsFlutterBinding`의 `defaultBinaryMessenger.setMockMethodCallHandler()`로 목킹할 수 있습니다. 이를 통해 실제 디바이스 없이도 채널 호출을 단위 테스트할 수 있습니다.

```dart
// 테스트 코드 예시
testWidgets('getBatteryLevel returns correct value', (tester) async {
  TestDefaultBinaryMessengerBinding.instance.defaultBinaryMessenger
      .setMockMethodCallHandler(
    const MethodChannel('dev.ducktudy/battery'),
    (call) async {
      if (call.method == 'getBatteryLevel') return 85;
      return null;
    },
  );

  final service = BatteryService();
  expect(await service.getBatteryLevel(), 85);
});
```

---

## 마치며

Platform Channels는 Flutter 앱이 네이티브 세계와 소통하는 핵심 메커니즘입니다. `MethodChannel`로 단발성 API를, `EventChannel`로 지속 스트림을, `BasicMessageChannel`로 양방향 메시지를 처리한다는 역할 분리를 명확히 이해하고 나면, 어떤 네이티브 기능이라도 Flutter 앱에 안전하게 통합할 수 있습니다. 특히 스레드 안전성과 `result`의 단 한 번 호출 규칙은 반드시 기억하세요.

---

## 참고 자료
- [Flutter 공식 문서 — Writing custom platform-specific code](https://docs.flutter.dev/platform-integration/platform-channels)
- [LogRocket — Using Flutter's MethodChannel to invoke Kotlin code for Android](https://blog.logrocket.com/using-flutters-methodchannel-invoke-kotlin-code-android/)
- [Flutter Platform Channels Complete Guide with Code Examples (2026)](https://www.appsonair.com/blogs/flutter-platform-channels-explained-with-real-examples)
- [Platform Channels in Flutter: A Comprehensive Guide with Examples](https://medium.com/@mohitarora7272/platform-channels-in-flutter-a-comprehensive-guide-with-examples-73ceda798bf1)
