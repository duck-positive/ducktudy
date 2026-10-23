---
layout: post
title: "Flutter Local Notifications 심화: 스케줄 알림·그룹화·액션 버튼·Android 정확 알람 권한 완전 정복"
date: 2026-10-23
categories: [android, flutter]
tags: [flutter, local-notifications, android, ios, scheduled-notifications, notification-channels, exact-alarm]
---

Flutter 앱에서 FCM(Firebase Cloud Messaging)이 아닌 **기기 자체에서 발생시키는 알림**이 필요한 경우는 생각보다 많습니다. 리마인더 앱, 약 복용 알림, 할 일 목록 마감 알림, 오프라인 상태에서도 동작해야 하는 캘린더 이벤트 등이 그 예입니다. 이 모든 시나리오에 답을 주는 것이 `flutter_local_notifications` 패키지입니다. 이 글에서는 기본 초기화부터 정확 스케줄 알림, 그룹화, 액션 버튼, Android 12+ 권한 처리까지 실무에서 바로 쓸 수 있는 수준으로 파고듭니다.

---

## 1. 개념 설명: 로컬 알림 vs 푸시 알림

**푸시 알림(Push Notification)**은 원격 서버에서 FCM 또는 APNs를 통해 기기로 전달됩니다. 서버가 트리거를 담당하므로 네트워크 연결이 필수입니다.

**로컬 알림(Local Notification)**은 앱 코드가 직접 OS 알림 시스템에 요청합니다. 인터넷이 끊겨 있어도, 앱이 백그라운드나 종료 상태여도 OS가 정해진 시각에 알림을 발생시킵니다.

`flutter_local_notifications` 패키지(현재 최신 안정 버전 22.3.x)는 Android, iOS, macOS, Linux, Windows, Web을 단일 Dart API로 지원합니다. 내부적으로 Android는 `NotificationCompat`, iOS/macOS는 `UNUserNotificationCenter`를 사용합니다.

---

## 2. 왜 필요한가

### FCM만으로 부족한 이유

- **오프라인 시나리오**: 기기가 인터넷에 연결되지 않은 상태에서도 알림이 와야 하는 경우
- **정밀한 시각 제어**: FCM 딜리버리 타임스탬프는 보장되지 않지만, 로컬 스케줄은 OS 수준의 정확도를 제공
- **반복 알림**: 매일 오전 9시, 매주 월요일 같은 규칙적 알림은 서버 비용 없이 기기에서 처리 가능
- **개인화된 타이밍**: 사용자가 앱 내에서 직접 설정한 시각에 맞춘 알림

---

## 3. 설정 및 초기화

### pubspec.yaml

```yaml
dependencies:
  flutter_local_notifications: ^22.3.2
  timezone: ^0.9.4          # 시간대 기반 스케줄에 필수
  flutter_timezone: ^2.0.0  # 현재 기기 시간대 감지
```

### AndroidManifest.xml 설정

Android 12+(API 31+)에서 정확 알람을 사용하려면 권한 선언이 필수입니다.

```xml
<manifest ...>
  <!-- 기본 알림 권한 (Android 13+) -->
  <uses-permission android:name="android.permission.POST_NOTIFICATIONS"/>

  <!-- 정확 알람 권한 (Android 12+). 
       USE_EXACT_ALARM은 자동 부여, SCHEDULE_EXACT_ALARM은 사용자 승인 필요 -->
  <uses-permission android:name="android.permission.USE_EXACT_ALARM"/>
  <!-- 또는: android.permission.SCHEDULE_EXACT_ALARM -->

  <!-- 부팅 후 알람 복원을 위해 -->
  <uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED"/>

  <application ...>
    <!-- 알람 수신 리시버 (패키지 내부 등록됨, 명시적 지정 권장) -->
    <receiver
      android:name="com.dexterous.flutterlocalnotifications.ScheduledNotificationReceiver"
      android:exported="false"/>
    <receiver
      android:name="com.dexterous.flutterlocalnotifications.ScheduledNotificationBootReceiver"
      android:exported="false">
      <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED"/>
        <action android:name="android.intent.action.MY_PACKAGE_REPLACED"/>
      </intent-filter>
    </receiver>
  </application>
</manifest>
```

---

## 4. 실제 구현 예제

### 예제 1: 초기화, 권한 요청, 기본/스케줄 알림

아래는 `NotificationService` 싱글턴으로 패키지를 감싼 실용적인 구현입니다.

```dart
import 'package:flutter_local_notifications/flutter_local_notifications.dart';
import 'package:timezone/data/latest_all.dart' as tz;
import 'package:timezone/timezone.dart' as tz;
import 'package:flutter_timezone/flutter_timezone.dart';

class NotificationService {
  NotificationService._();
  static final instance = NotificationService._();

  final _plugin = FlutterLocalNotificationsPlugin();

  /// 앱 시작 시 한 번 호출
  Future<void> init() async {
    // 타임존 데이터 초기화
    tz.initializeTimeZones();
    final tzName = await FlutterTimezone.getLocalTimezone();
    tz.setLocalLocation(tz.getLocation(tzName));

    const androidSettings = AndroidInitializationSettings('@mipmap/ic_launcher');
    const iosSettings = DarwinInitializationSettings(
      requestAlertPermission: false, // 권한은 별도 요청
      requestBadgePermission: false,
      requestSoundPermission: false,
    );

    await _plugin.initialize(
      const InitializationSettings(android: androidSettings, iOS: iosSettings),
      onDidReceiveNotificationResponse: _onTap,
      onDidReceiveBackgroundNotificationResponse: _onBackgroundTap,
    );
  }

  /// Android 13+ 알림 권한, iOS 권한 통합 요청
  Future<bool> requestPermissions() async {
    final android = _plugin.resolvePlatformSpecificImplementation<
        AndroidFlutterLocalNotificationsPlugin>();
    final ios = _plugin.resolvePlatformSpecificImplementation<
        IOSFlutterLocalNotificationsPlugin>();

    final androidGranted = await android?.requestNotificationsPermission() ?? true;
    final exactAlarmGranted = await android?.requestExactAlarmsPermission() ?? true;
    final iosGranted = await ios?.requestPermissions(
      alert: true,
      badge: true,
      sound: true,
    ) ?? true;

    return androidGranted && exactAlarmGranted && iosGranted;
  }

  /// 즉시 알림 표시
  Future<void> show({
    required int id,
    required String title,
    required String body,
    String? payload,
  }) async {
    await _plugin.show(
      id,
      title,
      body,
      NotificationDetails(
        android: AndroidNotificationDetails(
          'default_channel',
          '기본 알림',
          channelDescription: '앱의 기본 알림 채널',
          importance: Importance.high,
          priority: Priority.high,
          icon: '@mipmap/ic_launcher',
        ),
        iOS: const DarwinNotificationDetails(
          presentAlert: true,
          presentBadge: true,
          presentSound: true,
        ),
      ),
      payload: payload,
    );
  }

  /// 특정 날짜/시각에 한 번 발송 (정확 알람)
  Future<void> scheduleOnce({
    required int id,
    required String title,
    required String body,
    required DateTime scheduledDate,
    String? payload,
  }) async {
    await _plugin.zonedSchedule(
      id,
      title,
      body,
      tz.TZDateTime.from(scheduledDate, tz.local), // 로컬 타임존 적용
      NotificationDetails(
        android: AndroidNotificationDetails(
          'reminder_channel',
          '리마인더',
          channelDescription: '사용자 설정 리마인더 알림',
          importance: Importance.max,
          priority: Priority.high,
        ),
        iOS: const DarwinNotificationDetails(),
      ),
      payload: payload,
      androidScheduleMode: AndroidScheduleMode.exactAllowWhileIdle,
      // Doze 모드에서도 정확히 발송 (USE_EXACT_ALARM 또는 SCHEDULE_EXACT_ALARM 필요)
    );
  }

  /// 매일 특정 시각 반복 알림
  Future<void> scheduleDailyAt({
    required int id,
    required String title,
    required String body,
    required Time time, // flutter_local_notifications의 Time 타입
  }) async {
    await _plugin.zonedSchedule(
      id,
      title,
      body,
      _nextInstanceOfTime(time),
      NotificationDetails(
        android: AndroidNotificationDetails(
          'daily_channel',
          '일일 반복 알림',
          importance: Importance.high,
          priority: Priority.high,
        ),
        iOS: const DarwinNotificationDetails(),
      ),
      androidScheduleMode: AndroidScheduleMode.exactAllowWhileIdle,
      matchDateTimeComponents: DateTimeComponents.time, // 매일 같은 시각 반복
    );
  }

  tz.TZDateTime _nextInstanceOfTime(Time time) {
    final now = tz.TZDateTime.now(tz.local);
    var scheduled = tz.TZDateTime(
      tz.local,
      now.year, now.month, now.day,
      time.hour, time.minute, time.second,
    );
    if (scheduled.isBefore(now)) {
      scheduled = scheduled.add(const Duration(days: 1));
    }
    return scheduled;
  }

  Future<void> cancel(int id) => _plugin.cancel(id);
  Future<void> cancelAll() => _plugin.cancelAll();

  Future<List<PendingNotificationRequest>> getPending() =>
      _plugin.pendingNotificationRequests();
}

// 최상위 함수로 선언 (백그라운드 콜백은 최상위여야 함)
@pragma('vm:entry-point')
void _onBackgroundTap(NotificationResponse response) {
  // 백그라운드에서 알림 탭 시 처리
  print('Background tap: ${response.payload}');
}

void _onTap(NotificationResponse response) {
  // 포그라운드/앱 재시작 시 탭 처리
  print('Foreground tap: ${response.payload}');
}
```

`main()`에서 초기화:

```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await NotificationService.instance.init();
  runApp(const MyApp());
}
```

---

### 예제 2: 그룹 알림 + 액션 버튼 (Android 전용 고급 기능)

실무에서 자주 쓰이는 두 가지 고급 패턴입니다. 메시징 앱처럼 여러 알림을 하나로 묶는 **InboxStyle 그룹 알림**과, 알림에서 직접 작업할 수 있는 **액션 버튼**을 함께 구현합니다.

```dart
import 'package:flutter_local_notifications/flutter_local_notifications.dart';

extension GroupNotifications on NotificationService {
  static const _groupKey = 'com.example.app.GROUP_MESSAGES';
  static const _groupChannelId = 'message_group_channel';
  static const _groupChannelName = '메시지 그룹 알림';

  /// 메시지 알림 추가 후 서머리 업데이트
  Future<void> showGroupedMessage({
    required int id,
    required String sender,
    required String message,
    required List<String> allMessages, // 그룹에 쌓인 전체 메시지 목록
  }) async {
    final androidDetails = AndroidNotificationDetails(
      _groupChannelId,
      _groupChannelName,
      importance: Importance.high,
      priority: Priority.high,
      groupKey: _groupKey,
      // 개별 알림은 그룹의 멤버
      setAsGroupSummary: false,
      styleInformation: BigTextStyleInformation(message),
      actions: [
        // 액션 버튼 1: 답장 (RemoteInput으로 인라인 텍스트 입력)
        AndroidNotificationAction(
          'reply_action',
          '답장',
          icon: DrawableResourceAndroidBitmap('@drawable/ic_reply'),
          inputs: [
            AndroidNotificationActionInput(
              label: '메시지 입력...',
              allowedMimeTypes: {'text/plain'},
            ),
          ],
          showsUserInterface: false, // 답장 후 앱 열지 않음
        ),
        // 액션 버튼 2: 읽음 처리
        AndroidNotificationAction(
          'mark_read_action',
          '읽음',
          cancelNotification: true, // 탭 시 알림 자동 닫기
        ),
      ],
    );

    // 개별 메시지 알림 표시
    await _plugin.show(
      id,
      sender,
      message,
      NotificationDetails(android: androidDetails),
      payload: 'message:$id',
    );

    // 그룹 서머리 알림 (항상 고정 ID 사용)
    final summaryDetails = AndroidNotificationDetails(
      _groupChannelId,
      _groupChannelName,
      importance: Importance.low, // 서머리는 낮은 중요도
      priority: Priority.low,
      groupKey: _groupKey,
      setAsGroupSummary: true, // ← 이것이 서머리 알림
      styleInformation: InboxStyleInformation(
        allMessages,
        contentTitle: '${allMessages.length}개의 새 메시지',
        summaryText: '${allMessages.length}개',
      ),
    );

    await _plugin.show(
      0, // 서머리는 항상 ID 0
      '새 메시지',
      '${allMessages.length}개의 읽지 않은 메시지',
      NotificationDetails(android: summaryDetails),
    );
  }
}

// 액션 응답 처리 (onDidReceiveNotificationResponse 콜백에서)
void handleNotificationAction(NotificationResponse response) {
  switch (response.actionId) {
    case 'reply_action':
      final replyText = response.input; // 인라인 입력 텍스트
      if (replyText != null && replyText.isNotEmpty) {
        // API 호출로 답장 전송
        _sendReply(response.payload, replyText);
      }
      break;
    case 'mark_read_action':
      // 읽음 처리 API 호출
      _markAsRead(response.payload);
      break;
    default:
      // 알림 탭 (본문 탭) → 앱 열기
      _navigateToChat(response.payload);
  }
}

void _sendReply(String? payload, String text) {
  // TODO: 실제 구현
}

void _markAsRead(String? payload) {
  // TODO: 실제 구현
}

void _navigateToChat(String? payload) {
  // TODO: 실제 구현
}
```

---

## 5. 주의사항 및 팁

### Android 정확 알람 권한 흐름

Android 12+(API 31+)부터 정확 알람 사용 전에 반드시 `canScheduleExactAlarms()`를 확인해야 합니다.

- **`USE_EXACT_ALARM`**: 자동 부여, 사용자가 직접 설정하지 않아도 됩니다. Google Play 정책 제한이 있으므로 알람/달력 앱 같이 핵심 기능으로 사용하는 경우에만 사용하세요.
- **`SCHEDULE_EXACT_ALARM`**: 사용자가 시스템 설정에서 직접 허용해야 합니다. Android 14에서 기기 복원 시 기본 거부 상태가 됩니다.

```dart
Future<void> ensureExactAlarmPermission(BuildContext context) async {
  final android = NotificationService.instance._plugin
      .resolvePlatformSpecificImplementation<
          AndroidFlutterLocalNotificationsPlugin>();

  if (android == null) return;

  // 정확 알람 권한 현재 상태 확인
  final canSchedule = await android.canScheduleExactNotifications();
  if (canSchedule == false) {
    // 사용자에게 이유를 설명한 후 시스템 설정으로 안내
    if (context.mounted) {
      showDialog(
        context: context,
        builder: (_) => AlertDialog(
          title: const Text('정확한 알림을 위해 권한이 필요합니다'),
          content: const Text(
            '정확한 시각에 리마인더를 받으려면 "정확한 알람 및 리마인더" 권한을 허용해 주세요.',
          ),
          actions: [
            TextButton(
              onPressed: () => Navigator.pop(context),
              child: const Text('나중에'),
            ),
            TextButton(
              onPressed: () {
                Navigator.pop(context);
                android.requestExactAlarmsPermission();
              },
              child: const Text('설정으로 이동'),
            ),
          ],
        ),
      );
    }
  }
}
```

### 부팅 후 알람 복원

기기가 재부팅되면 모든 예약 알림이 취소됩니다. `RECEIVE_BOOT_COMPLETED`를 선언하면 패키지의 `ScheduledNotificationBootReceiver`가 자동으로 복원을 처리하지만, 복원 데이터는 기기 스토리지에 직렬화됩니다. 알림 ID와 예약 시각을 앱 DB(SharedPreferences, Hive, Isar 등)에 별도로 저장하고, 부팅 복원 시 재스케줄하는 방어 로직을 추가하는 것이 안전합니다.

```dart
// boot_receiver_handler.dart (실제 앱 로직)
Future<void> rescheduleAllOnBoot() async {
  final pendingItems = await MyRepository.getPendingReminders();
  for (final item in pendingItems) {
    if (item.scheduledAt.isAfter(DateTime.now())) {
      await NotificationService.instance.scheduleOnce(
        id: item.id,
        title: item.title,
        body: item.body,
        scheduledDate: item.scheduledAt,
      );
    } else {
      // 이미 지난 알림은 DB에서 제거
      await MyRepository.deleteReminder(item.id);
    }
  }
}
```

### iOS NotificationCenter 통합

iOS 15+에서는 `UNNotificationCategory`를 사용해 액션 버튼을 등록합니다. `DarwinInitializationSettings`의 `notificationCategories` 파라미터로 전달합니다.

```dart
const iosSettings = DarwinInitializationSettings(
  notificationCategories: [
    DarwinNotificationCategory(
      'message_category',
      actions: [
        DarwinNotificationAction.plain('reply_action', '답장',
            options: {DarwinNotificationActionOption.foreground}),
        DarwinNotificationAction.plain('mark_read_action', '읽음'),
      ],
    ),
  ],
);
```

### 알림 채널 중요도 설계

Android 8.0(API 26) 이상에서 채널 중요도(Importance)는 **생성 후 변경 불가**입니다. 앱 릴리스 전에 채널 계층을 설계하세요:

| 채널 | Importance | 사용 사례 |
|------|-----------|---------|
| `urgent_channel` | `Importance.max` | 전화, 긴급 알림 |
| `reminder_channel` | `Importance.high` | 사용자 설정 리마인더 |
| `update_channel` | `Importance.default_` | 업데이트, 배지 |
| `silent_channel` | `Importance.low` | 진행 표시줄, 서머리 |

채널 중요도를 바꿔야 한다면 새 ID의 채널을 생성하고 이전 채널을 삭제하는 마이그레이션 로직이 필요합니다.

---

## 6. 실전 체크리스트

- [ ] `tz.initializeTimeZones()` + `FlutterTimezone`으로 기기 시간대 적용 완료
- [ ] Android 12+: `USE_EXACT_ALARM` 또는 `SCHEDULE_EXACT_ALARM` 권한 선언
- [ ] Android 13+: `POST_NOTIFICATIONS` 런타임 권한 요청
- [ ] `RECEIVE_BOOT_COMPLETED` 선언 + 부팅 후 재스케줄 로직 구현
- [ ] 알림 ID 충돌 방지: 고유 ID 생성 전략 수립 (예: DB auto-increment)
- [ ] iOS `notificationCategories`로 액션 버튼 등록
- [ ] 앱 종료 상태에서의 알림 탭 처리: `getNotificationAppLaunchDetails()` 확인

---

## 참고 자료
- [flutter_local_notifications 패키지 (pub.dev)](https://pub.dev/packages/flutter_local_notifications)
- [Android 정확 알람 스케줄링 공식 문서](https://developer.android.com/training/scheduling/alarms)
- [timezone Dart 패키지](https://pub.dev/packages/timezone)
- [flutter_timezone 패키지](https://pub.dev/packages/flutter_timezone)
