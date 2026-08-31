---
layout: post
title: "Flutter Background Service 심화: workmanager와 flutter_background_service로 앱 종료 후에도 살아남는 백그라운드 작업 완전 정복"
date: 2026-08-31
categories: [android, flutter]
tags: [flutter, android, background-service, workmanager, dart, background-execution, isolate]
---

모바일 앱을 개발하다 보면 반드시 맞닥뜨리는 요구사항이 있습니다. "앱이 종료된 상태에서도 주기적으로 서버와 동기화해야 해요", "사용자가 앱을 닫아도 위치 추적을 계속해야 해요"처럼, **앱 프로세스 밖에서 실행되는 작업** 이 필요한 상황입니다. Flutter에서 이를 구현하는 방법은 크게 두 가지입니다. 일회성·주기성 작업에 최적화된 **workmanager** 패키지와, 앱이 닫혀도 지속적으로 동작하는 서비스가 필요할 때 쓰는 **flutter_background_service** 패키지입니다. 이 글에서는 두 패키지의 내부 동작 원리부터 실전 구현, 그리고 Android 배터리 최적화 정책의 함정까지 심층적으로 다룹니다.

---

## 1. 왜 백그라운드 실행이 이렇게 어려운가

Flutter 앱이 포그라운드를 떠나는 순간, iOS와 Android는 각자의 방식으로 앱을 제한합니다.

**Android의 제약**

Android 8.0(API 26)부터 백그라운드 서비스 실행에 강력한 제한이 생겼습니다. 앱이 백그라운드로 전환된 후 몇 분 이내에 시스템은 해당 앱의 백그라운드 서비스를 강제 종료합니다. Android 12(API 31)부터는 `Exact Alarms`도 권한이 필요해졌고, Doze 모드와 App Standby Bucket이 적극적으로 작업 실행을 지연시킵니다.

Google이 이 문제를 해결하기 위해 제안한 공식 솔루션이 바로 **WorkManager**입니다. WorkManager는 작업을 OS 수준에서 스케줄링하여 앱 프로세스와 독립적으로 실행을 보장합니다. Doze 모드, 기기 재부팅, 앱 업데이트에도 작업이 유지됩니다.

**iOS의 제약**

iOS는 Android보다 더욱 엄격합니다. 기본적으로 백그라운드에서 코드를 실행할 수 없으며, `BGTaskScheduler`를 통해 제한된 시간 동안의 실행만 허용합니다. 지속적인 서비스 형태의 실행은 iOS에서 공식적으로 지원되지 않으며, flutter_background_service 역시 iOS에서는 Background Fetch 방식으로 15분~30분 간격의 짧은 실행만 가능합니다.

---

## 2. workmanager: 주기적·일회성 작업의 표준

### 2-1. 내부 동작 원리

`workmanager` 패키지는 Flutter의 `callbackDispatcher` 메커니즘을 활용합니다. Dart VM은 기본적으로 UI 스레드에서 동작하지만, 백그라운드에서 실행될 때는 **별도의 Flutter Engine 인스턴스**가 생성됩니다. 이 Engine은 UI 없이 Dart 코드만 실행하는 헤드리스(headless) 모드로 동작합니다.

Android 내부적으로는 `CoroutineWorker`가 WorkManager와 연결되어 있으며, iOS에서는 `BGProcessingTask` 또는 `BGAppRefreshTask`에 매핑됩니다.

### 2-2. 기본 설정

`pubspec.yaml`에 패키지를 추가합니다.

```yaml
dependencies:
  workmanager: ^0.10.9
```

Android의 경우 `AndroidManifest.xml`에 WorkManager가 사용하는 리시버를 등록해야 합니다. 패키지가 자동으로 처리하지만, 커스텀 설정이 필요하면 직접 선언할 수 있습니다.

### 2-3. 핵심 구현 예제: 주기적 데이터 동기화

아래 코드는 앱이 완전히 종료된 상태에서도 15분마다 서버와 동기화하는 작업을 등록하는 예제입니다.

```dart
import 'package:flutter/material.dart';
import 'package:workmanager/workmanager.dart';
import 'package:http/http.dart' as http;
import 'dart:convert';

// 최상위 레벨 함수여야 합니다. 클래스 메서드 불가.
@pragma('vm:entry-point')
void callbackDispatcher() {
  Workmanager().executeTask((taskName, inputData) async {
    switch (taskName) {
      case SyncTasks.periodicSync:
        await _performDataSync(inputData);
        break;
      case SyncTasks.oneTimeUpload:
        await _uploadPendingData(inputData);
        break;
    }
    return Future.value(true); // true = 성공, false = 실패(재시도), throw = 실패(재시도)
  });
}

class SyncTasks {
  static const periodicSync = 'periodicSyncTask';
  static const oneTimeUpload = 'oneTimeUploadTask';
}

Future<void> _performDataSync(Map<String, dynamic>? inputData) async {
  try {
    // SharedPreferences나 Hive 등 로컬 DB 접근 가능
    final response = await http.get(
      Uri.parse('https://api.example.com/sync'),
      headers: {'Authorization': 'Bearer ${inputData?['token'] ?? ''}'},
    ).timeout(const Duration(seconds: 30));

    if (response.statusCode == 200) {
      final data = jsonDecode(response.body);
      // 로컬 DB에 저장하는 로직
      print('[BackgroundSync] 동기화 완료: ${data['updatedAt']}');
    }
  } catch (e) {
    print('[BackgroundSync] 동기화 실패: $e');
    // false를 반환하면 WorkManager가 지수 백오프로 재시도
    return Future.error(e);
  }
}

Future<void> _uploadPendingData(Map<String, dynamic>? inputData) async {
  // 오프라인 상태에서 쌓인 데이터를 업로드하는 로직
  print('[BackgroundUpload] 업로드 작업 시작');
}

class MyApp extends StatefulWidget {
  const MyApp({super.key});

  @override
  State<MyApp> createState() => _MyAppState();
}

class _MyAppState extends State<MyApp> {
  @override
  void initState() {
    super.initState();
    _initWorkmanager();
  }

  Future<void> _initWorkmanager() async {
    await Workmanager().initialize(
      callbackDispatcher,
      isInDebugMode: false, // 디버그 모드에서는 즉시 실행
    );

    // 주기적 작업 등록 (최소 간격 15분)
    await Workmanager().registerPeriodicTask(
      'sync-task-unique-id',
      SyncTasks.periodicSync,
      frequency: const Duration(minutes: 15),
      constraints: Constraints(
        networkType: NetworkType.connected, // Wi-Fi 또는 데이터 연결 시에만 실행
        requiresBatteryNotLow: true,        // 배터리 부족 시 실행 안 함
        requiresCharging: false,
      ),
      inputData: {'token': 'user_access_token'},
      existingWorkPolicy: ExistingWorkPolicy.keep, // 이미 등록된 경우 유지
    );

    // 일회성 작업 등록 (즉시 또는 지연 실행)
    await Workmanager().registerOneOffTask(
      'upload-task-unique-id',
      SyncTasks.oneTimeUpload,
      initialDelay: const Duration(seconds: 10),
      constraints: Constraints(
        networkType: NetworkType.connected,
      ),
      inputData: {'retryCount': 0},
    );
  }

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      home: Scaffold(
        appBar: AppBar(title: const Text('Background Workmanager Demo')),
        body: Center(
          child: ElevatedButton(
            onPressed: () async {
              // 특정 작업 취소
              await Workmanager().cancelByUniqueName('sync-task-unique-id');
              // 모든 작업 취소
              await Workmanager().cancelAll();
            },
            child: const Text('모든 백그라운드 작업 취소'),
          ),
        ),
      ),
    );
  }
}
```

**핵심 포인트**

- `callbackDispatcher`는 반드시 **최상위 레벨 함수**여야 합니다. 별도 Flutter Engine 인스턴스에서 실행되기 때문에 앱의 상태나 싱글톤에 접근할 수 없습니다.
- `@pragma('vm:entry-point')` 어노테이션은 R8/ProGuard가 이 함수를 제거하지 않도록 보호합니다. Flutter 앱을 릴리스 빌드하면 트리 쉐이킹으로 미사용 코드가 제거되는데, 백그라운드 진입점은 앱 코드에서 직접 호출되지 않기 때문에 이 어노테이션 없이는 릴리스 빌드에서 크래시가 발생합니다.
- `ExistingWorkPolicy.keep`을 사용하면 동일한 ID로 이미 작업이 예약된 경우 새 작업을 무시합니다. 앱 재시작 시 중복 등록을 방지하려면 이 정책을 활용하세요.

---

## 3. flutter_background_service: 지속형 백그라운드 서비스

workmanager가 "정해진 시점에 짧은 작업을 실행"하는 방식이라면, **flutter_background_service**는 앱이 닫혀도 계속 살아있는 Android의 `Foreground Service`를 Flutter에서 제어할 수 있게 해줍니다. 실시간 위치 추적, 지속적인 BLE 통신, 음악 재생 제어 등 중단 없는 서비스가 필요할 때 사용합니다.

### 3-1. Android Foreground Service의 이해

Android 정책상, 백그라운드에서 장시간 실행되는 작업은 반드시 **Foreground Service** 형태여야 하며, 사용자에게 알림(Notification)으로 실행 중임을 알려야 합니다. Android 14(API 34)부터는 포그라운드 서비스 유형(dataSync, location, health, remoteMessaging 등)을 명시적으로 선언해야 합니다.

### 3-2. 설정

```yaml
dependencies:
  flutter_background_service: ^5.1.0
```

`AndroidManifest.xml`에 아래 권한과 서비스를 추가합니다.

```xml
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_DATA_SYNC" />

<application>
  <service
    android:name="id.flutter.flutter_background_service.BackgroundService"
    android:foregroundServiceType="dataSync"
    android:exported="false" />
</application>
```

### 3-3. 핵심 구현 예제: 실시간 센서 모니터링 서비스

아래 예제는 앱이 닫혀도 계속 동작하며, UI와 양방향으로 데이터를 교환하는 완전한 백그라운드 서비스를 구현합니다.

```dart
import 'dart:async';
import 'dart:ui';
import 'package:flutter/material.dart';
import 'package:flutter_background_service/flutter_background_service.dart';
import 'package:flutter_local_notifications/flutter_local_notifications.dart';

// 서비스 진입점: 반드시 최상위 레벨 함수
@pragma('vm:entry-point')
void onStart(ServiceInstance service) async {
  // DartPluginRegistrant.ensureInitialized()를 통해 플러그인 사용 가능
  DartPluginRegistrant.ensureInitialized();

  // 서비스 → UI 방향 메시지 스트림
  final FlutterLocalNotificationsPlugin notificationsPlugin =
      FlutterLocalNotificationsPlugin();

  // Android 포그라운드 서비스 알림 업데이트
  if (service is AndroidServiceInstance) {
    service.on('setAsForeground').listen((event) {
      service.setAsForegroundService();
    });
    service.on('setAsBackground').listen((event) {
      service.setAsBackgroundService();
    });
  }

  // UI에서 서비스로 보내는 stop 이벤트 수신
  service.on('stopService').listen((event) {
    service.stopSelf();
  });

  // 1초마다 센서 데이터를 읽어 UI로 전송
  int tick = 0;
  Timer.periodic(const Duration(seconds: 1), (timer) async {
    if (service is AndroidServiceInstance) {
      // 포그라운드 서비스 알림 텍스트 업데이트
      if (await service.isForegroundService()) {
        notificationsPlugin.show(
          888,
          '센서 모니터링 중',
          '경과 시간: ${tick}초 | CPU: ${_getFakeCpuUsage()}%',
          const NotificationDetails(
            android: AndroidNotificationDetails(
              'sensor_monitor_channel',
              '센서 모니터 채널',
              importance: Importance.low,
              ongoing: true,
              playSound: false,
              enableVibration: false,
            ),
          ),
        );
      }
    }

    // UI로 데이터 전송 (UI가 없으면 무시됨)
    service.invoke('update', {
      'tick': tick,
      'timestamp': DateTime.now().toIso8601String(),
      'cpuUsage': _getFakeCpuUsage(),
      'memoryMB': _getFakeMemoryUsage(),
    });

    tick++;
  });
}

double _getFakeCpuUsage() => (tick % 100).toDouble();
int _getFakeMemoryUsage() => 120 + (tick % 50);
int tick = 0;

// 앱 초기화 시 서비스 설정
Future<void> initializeService() async {
  final service = FlutterBackgroundService();

  // 알림 채널 생성 (Android 8.0+)
  const AndroidNotificationChannel channel = AndroidNotificationChannel(
    'sensor_monitor_channel',
    '센서 모니터 채널',
    description: '백그라운드에서 센서를 모니터링합니다.',
    importance: Importance.low,
  );

  final FlutterLocalNotificationsPlugin notificationsPlugin =
      FlutterLocalNotificationsPlugin();
  await notificationsPlugin
      .resolvePlatformSpecificImplementation<
          AndroidFlutterLocalNotificationsPlugin>()
      ?.createNotificationChannel(channel);

  await service.configure(
    androidConfiguration: AndroidConfiguration(
      onStart: onStart,
      autoStart: true,           // 앱 재시작 시 자동으로 서비스 시작
      isForegroundMode: true,    // 포그라운드 서비스로 실행
      notificationChannelId: 'sensor_monitor_channel',
      initialNotificationTitle: '센서 모니터링',
      initialNotificationContent: '시작 중...',
      foregroundServiceNotificationId: 888,
      foregroundServiceTypes: [AndroidForegroundType.dataSync],
    ),
    iosConfiguration: IosConfiguration(
      autoStart: true,
      onForeground: onStart,
      // iOS에서는 Background Fetch 방식으로 제한적 실행
      onBackground: _onIosBackground,
    ),
  );
}

@pragma('vm:entry-point')
Future<bool> _onIosBackground(ServiceInstance service) async {
  // iOS 백그라운드: 30초 이내 완료해야 함
  WidgetsFlutterBinding.ensureInitialized();
  DartPluginRegistrant.ensureInitialized();
  print('[iOS] 백그라운드 실행 중');
  return true;
}

// UI 레이어: 서비스와 통신
class SensorMonitorScreen extends StatefulWidget {
  const SensorMonitorScreen({super.key});

  @override
  State<SensorMonitorScreen> createState() => _SensorMonitorScreenState();
}

class _SensorMonitorScreenState extends State<SensorMonitorScreen> {
  final service = FlutterBackgroundService();
  Map<String, dynamic>? _latestData;
  StreamSubscription? _subscription;

  @override
  void initState() {
    super.initState();
    // 서비스에서 오는 데이터 구독
    _subscription = service.on('update').listen((data) {
      if (mounted) {
        setState(() => _latestData = data);
      }
    });
  }

  @override
  void dispose() {
    _subscription?.cancel();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('센서 모니터링')),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            StreamBuilder<bool>(
              stream: Stream.periodic(const Duration(seconds: 1))
                  .asyncMap((_) => service.isRunning()),
              builder: (context, snapshot) {
                final isRunning = snapshot.data ?? false;
                return Row(
                  children: [
                    Icon(
                      isRunning ? Icons.circle : Icons.circle_outlined,
                      color: isRunning ? Colors.green : Colors.red,
                      size: 16,
                    ),
                    const SizedBox(width: 8),
                    Text(isRunning ? '서비스 실행 중' : '서비스 중지됨'),
                  ],
                );
              },
            ),
            const SizedBox(height: 24),
            if (_latestData != null) ...[
              Text('경과 시간: ${_latestData!['tick']}초',
                  style: const TextStyle(fontSize: 20)),
              Text('CPU 사용률: ${_latestData!['cpuUsage']}%'),
              Text('메모리: ${_latestData!['memoryMB']} MB'),
              Text('업데이트: ${_latestData!['timestamp']}',
                  style: const TextStyle(fontSize: 12, color: Colors.grey)),
            ] else
              const Text('서비스 데이터 대기 중...'),
            const Spacer(),
            Row(
              children: [
                Expanded(
                  child: ElevatedButton(
                    onPressed: () => service.startService(),
                    child: const Text('서비스 시작'),
                  ),
                ),
                const SizedBox(width: 8),
                Expanded(
                  child: OutlinedButton(
                    onPressed: () => service.invoke('stopService'),
                    child: const Text('서비스 중지'),
                  ),
                ),
              ],
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## 4. workmanager vs flutter_background_service: 언제 무엇을 쓸까

| 구분 | workmanager | flutter_background_service |
|------|------------|---------------------------|
| 실행 방식 | 스케줄 기반 (OS가 시점 결정) | 즉시 시작, 지속 실행 |
| Android 구현 | WorkManager (CoroutineWorker) | Foreground Service |
| iOS 지원 | BGTaskScheduler (제한적) | Background Fetch (매우 제한적) |
| 배터리 영향 | 낮음 | 높음 (포그라운드 알림 표시) |
| 사용 사례 | 주기적 동기화, 파일 업로드 | 위치 추적, BLE 연결, 음악 재생 |
| 최소 간격 | 15분 (Android) | 제한 없음 |
| 알림 필요 여부 | 불필요 | Android에서 필수 |

---

## 5. 주의사항과 실전 팁

### 5-1. 배터리 최적화 예외 처리

삼성, Xiaomi, Huawei 등 일부 제조사는 기본 Android 위에 자체 배터리 최적화 레이어를 추가합니다. 이 때문에 WorkManager 작업이 제때 실행되지 않거나 Foreground Service가 강제 종료될 수 있습니다. 앱 설정에서 사용자가 직접 배터리 최적화 예외를 허용하도록 안내해야 합니다.

```dart
import 'package:permission_handler/permission_handler.dart';

Future<void> requestBatteryOptimizationExemption() async {
  // Android 6.0+ 에서 배터리 최적화 예외 요청
  final status = await Permission.ignoreBatteryOptimizations.status;
  if (!status.isGranted) {
    // 사용자에게 설정 화면으로 이동하도록 안내
    await Permission.ignoreBatteryOptimizations.request();
  }
}
```

### 5-2. 릴리스 빌드에서의 R8 설정

앞서 언급한 `@pragma('vm:entry-point')` 외에도, `proguard-rules.pro`에 아래 규칙을 추가하면 안전합니다.

```
-keep class id.flutter.flutter_background_service.** { *; }
-keep class dev.flutter.pigeon.** { *; }
```

### 5-3. isolate 통신의 한계

백그라운드 실행 컨텍스트는 Flutter의 메인 Isolate와 **완전히 격리**된 별도 Isolate입니다. 따라서 앱에서 사용하던 싱글톤 인스턴스, Provider, Riverpod의 상태 등에 직접 접근할 수 없습니다. 백그라운드에서는 별도의 DB 연결을 열거나 SharedPreferences를 직접 읽어야 합니다.

flutter_background_service의 `invoke`/`on` 메커니즘이나 workmanager의 `inputData`/`outputData`는 JSON으로 직렬화 가능한 타입만 지원합니다. 복잡한 객체는 직렬화 후 전달해야 합니다.

### 5-4. iOS에서의 현실적인 기대치

iOS 앱스토어 정책상, 명확한 기능적 이유(위치 서비스, 오디오 재생, VoIP 등) 없이 백그라운드 실행을 사용하면 심사 거부될 수 있습니다. flutter_background_service의 iOS 지원은 Background Fetch에 의존하므로 최소 15분 간격이며 30초 이내에 작업을 완료해야 합니다. iOS에서 지속적인 백그라운드 실행이 필요하다면 오디오 세션, 위치 서비스 등 허용된 백그라운드 모드를 사용해야 합니다.

### 5-5. 디버깅 전략

WorkManager 작업은 `adb shell` 명령으로 직접 트리거할 수 있어 개발 중 유용합니다.

```bash
# 등록된 작업 목록 확인
adb shell dumpsys jobscheduler

# WorkManager 진단 정보 확인
adb shell dumpsys activity service WorkManagerService

# workmanager의 isInDebugMode: true 설정 시 즉시 실행
```

flutter_background_service는 `isForegroundService()`를 주기적으로 체크하여 서비스가 예상대로 실행 중인지 확인하세요.

---

## 마치며

Flutter에서 백그라운드 실행은 단순히 패키지를 추가하는 것으로 끝나지 않습니다. Android의 진화하는 배터리 정책, 제조사별 커스터마이징, iOS의 엄격한 제한을 모두 이해하고 대응해야 합니다. workmanager는 배터리 친화적인 주기 작업에, flutter_background_service는 사용자에게 명확한 가치를 제공하는 지속 서비스에 각각 사용하고, 두 패키지 모두 릴리스 빌드에서 반드시 검증하는 습관을 들이시기 바랍니다.

## 참고 자료
- [workmanager Flutter 패키지 (pub.dev)](https://pub.dev/packages/workmanager)
- [flutter_background_service Flutter 패키지 (pub.dev)](https://pub.dev/packages/flutter_background_service)
- [Android WorkManager 공식 문서](https://developer.android.com/develop/background-work/background-tasks/persistent/getting-started)
