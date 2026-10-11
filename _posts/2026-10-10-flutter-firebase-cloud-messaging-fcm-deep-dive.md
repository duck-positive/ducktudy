---
layout: post
title: "Flutter Firebase Cloud Messaging(FCM) 심화: 포그라운드·백그라운드·종료 상태 푸시 알림 완전 정복"
date: 2026-10-10
categories: [android, flutter]
tags: [flutter, firebase, fcm, push-notification, firebase_messaging, flutter_local_notifications, dart]
---

푸시 알림은 사용자를 앱으로 다시 끌어오는 가장 강력한 채널이다. Firebase Cloud Messaging(FCM)은 Google이 제공하는 크로스플랫폼 메시지 전송 솔루션으로, Android와 iOS 모두를 단일 API로 지원한다. 그러나 Flutter에서 FCM을 제대로 다루려면 단순히 패키지를 추가하는 것을 넘어, 앱의 세 가지 생명주기 상태—포그라운드(Foreground), 백그라운드(Background), 종료(Terminated)—에서의 메시지 처리 차이, isolate 경계, Android/iOS 플랫폼별 구성의 세부 차이를 완전히 이해해야 한다. 이 글에서는 `firebase_messaging ^16.7.0`과 `flutter_local_notifications ^22.3.1`을 기준으로 실제 프로덕션 수준의 구현을 단계별로 완전히 정복한다.

---

## 1. FCM 메시지 타입과 앱 상태 매트릭스

FCM은 크게 두 종류의 메시지를 전송한다.

| 타입 | 설명 | 처리 주체 |
|---|---|---|
| **Notification Message** | `notification` 페이로드 포함. OS가 자동으로 알림 표시 | 백그라운드/종료: OS. 포그라운드: 앱 |
| **Data Message** | `data` 페이로드만 포함. OS가 알림을 표시하지 않음 | 항상 앱 코드 |

앱 상태와 메시지 타입에 따라 호출되는 Flutter 핸들러가 달라진다:

| 앱 상태 | Notification | Data | Notification+Data |
|---|---|---|---|
| **포그라운드** | `onMessage` | `onMessage` | `onMessage` |
| **백그라운드** | OS 자동 표시 | `onBackgroundMessage` | OS 표시 + `onBackgroundMessage` |
| **종료(Terminated)** | OS 자동 표시 | `onBackgroundMessage` | OS 표시 + `onBackgroundMessage` |

이 매트릭스를 이해하지 못하면 "왜 백그라운드에서 내 핸들러가 안 불리나?" 같은 디버깅 지옥에 빠지게 된다.

---

## 2. 왜 별도의 Isolate가 필요한가

Flutter의 Dart 코드는 기본적으로 단일 isolate(메인 isolate)에서 실행된다. 그런데 Android에서 앱이 종료된 상태로 FCM 데이터 메시지가 도착하면, OS는 앱의 메인 UI를 띄우지 않고 백그라운드에서만 메시지를 처리해야 한다.

이를 위해 `firebase_messaging`은 **별도의 백그라운드 isolate**를 생성한다. 이 isolate는 메인 isolate와 메모리를 공유하지 않으며, 따라서 메인 isolate에서 초기화한 상태(예: ProviderContainer, GetIt 등)가 전혀 보이지 않는다. 백그라운드 핸들러 안에서 Firebase를 다시 초기화하고, 필요한 의존성을 모두 재설정해야 하는 이유가 바로 여기에 있다.

또한 이 함수는 반드시 **top-level 함수**여야 한다. 클래스 메서드나 람다는 사용할 수 없다. 릴리즈 빌드에서 트리 쉐이킹(tree shaking)에 의해 제거되는 것을 막으려면 `@pragma('vm:entry-point')` 애노테이션도 필수다.

---

## 3. 프로젝트 설정

### 3.1 pubspec.yaml

```yaml
dependencies:
  firebase_core: ^3.13.1
  firebase_messaging: ^16.7.0
  flutter_local_notifications: ^22.3.1
```

### 3.2 Android 설정

`android/app/build.gradle`의 `minSdkVersion`을 23 이상으로 설정한다. FCM의 최신 기능(안드로이드 13+ 알림 권한 등)을 쓰려면 26 이상이 권장된다.

`android/app/src/main/AndroidManifest.xml`에 다음을 추가한다:

```xml
<!-- 안드로이드 13 이상에서 알림 권한 요청을 위한 선언 -->
<uses-permission android:name="android.permission.POST_NOTIFICATIONS"/>

<application ...>
    <!-- FCM 기본 채널 ID 지정 -->
    <meta-data
        android:name="com.google.firebase.messaging.default_notification_channel_id"
        android:value="high_importance_channel" />
</application>
```

### 3.3 iOS 설정

`ios/Runner/AppDelegate.swift`에서 `UNUserNotificationCenterDelegate` 설정이 FlutterFire에 의해 자동으로 처리된다. 다만 Xcode에서 **Push Notifications** 및 **Background Modes > Remote notifications** capability를 반드시 추가해야 한다.

---

## 4. 코드 예제 1 — 핵심 FCM 초기화 및 세 가지 상태 핸들러

```dart
import 'dart:io';
import 'package:firebase_core/firebase_core.dart';
import 'package:firebase_messaging/firebase_messaging.dart';
import 'package:flutter/material.dart';
import 'package:flutter_local_notifications/flutter_local_notifications.dart';

// ① 최상위 함수 + entry-point 애노테이션: 릴리즈 빌드에서 트리 쉐이킹 방지
@pragma('vm:entry-point')
Future<void> _firebaseMessagingBackgroundHandler(RemoteMessage message) async {
  // ② 별도 isolate이므로 Firebase를 다시 초기화
  await Firebase.initializeApp();

  // ③ 데이터 메시지일 때만 로컬 알림 직접 표시 (Notification 메시지는 OS가 표시)
  if (message.notification == null) {
    await _showLocalNotification(message);
  }

  debugPrint('[Background] messageId=${message.messageId}');
}

// 로컬 알림 플러그인 인스턴스 (전역 싱글턴)
final FlutterLocalNotificationsPlugin _localNotifications =
    FlutterLocalNotificationsPlugin();

const AndroidNotificationChannel _channel = AndroidNotificationChannel(
  'high_importance_channel',
  '중요 알림',
  description: '앱의 중요 알림에 사용됩니다.',
  importance: Importance.max,
  playSound: true,
);

Future<void> _showLocalNotification(RemoteMessage message) async {
  final notification = message.notification;
  final title = notification?.title ?? message.data['title'] ?? '새 알림';
  final body = notification?.body ?? message.data['body'] ?? '';

  await _localNotifications.show(
    message.hashCode,
    title,
    body,
    NotificationDetails(
      android: AndroidNotificationDetails(
        _channel.id,
        _channel.name,
        channelDescription: _channel.description,
        importance: Importance.max,
        priority: Priority.high,
        icon: '@mipmap/ic_launcher',
      ),
      iOS: const DarwinNotificationDetails(
        presentAlert: true,
        presentBadge: true,
        presentSound: true,
      ),
    ),
    payload: message.data.toString(),
  );
}

class FcmService {
  final FirebaseMessaging _messaging = FirebaseMessaging.instance;

  Future<void> initialize() async {
    // ④ 백그라운드 핸들러 등록: main()보다 먼저 or 최상단에 등록해야 함
    FirebaseMessaging.onBackgroundMessage(_firebaseMessagingBackgroundHandler);

    // ⑤ 로컬 알림 플러그인 초기화
    await _initLocalNotifications();

    // ⑥ iOS 포그라운드 알림 표시 옵션 설정
    await _messaging.setForegroundNotificationPresentationOptions(
      alert: true,
      badge: true,
      sound: true,
    );

    // ⑦ 알림 권한 요청 (Android 13+, iOS)
    final settings = await _messaging.requestPermission(
      alert: true,
      badge: true,
      sound: true,
      provisional: false, // iOS: true면 조용한 알림으로 시작
    );

    if (settings.authorizationStatus == AuthorizationStatus.authorized) {
      debugPrint('알림 권한 허용됨');
    }

    // ⑧ FCM 토큰 획득 및 갱신 리스너
    final token = await _messaging.getToken();
    debugPrint('FCM Token: $token');

    _messaging.onTokenRefresh.listen((newToken) {
      // 서버에 새 토큰 전송
      _sendTokenToServer(newToken);
    });

    // ⑨ 포그라운드 메시지 리스너
    FirebaseMessaging.onMessage.listen(_handleForegroundMessage);

    // ⑩ 백그라운드에서 알림 탭 → 앱 열기 핸들러
    FirebaseMessaging.onMessageOpenedApp.listen(_handleNotificationTap);

    // ⑪ 앱이 종료 상태에서 알림 탭으로 시작된 경우
    final initialMessage = await _messaging.getInitialMessage();
    if (initialMessage != null) {
      _handleNotificationTap(initialMessage);
    }
  }

  Future<void> _initLocalNotifications() async {
    // Android 초기화 설정
    const androidSettings = AndroidInitializationSettings('@mipmap/ic_launcher');
    // iOS 초기화 설정
    const iosSettings = DarwinInitializationSettings(
      requestAlertPermission: false, // FCM에서 직접 요청하므로 false
      requestBadgePermission: false,
      requestSoundPermission: false,
    );

    await _localNotifications.initialize(
      const InitializationSettings(
        android: androidSettings,
        iOS: iosSettings,
      ),
      onDidReceiveNotificationResponse: (response) {
        // 로컬 알림 탭 처리
        debugPrint('로컬 알림 탭: payload=${response.payload}');
      },
    );

    // Android 고중요도 채널 생성
    await _localNotifications
        .resolvePlatformSpecificImplementation<
            AndroidFlutterLocalNotificationsPlugin>()
        ?.createNotificationChannel(_channel);
  }

  void _handleForegroundMessage(RemoteMessage message) {
    debugPrint('[Foreground] ${message.notification?.title}');
    // 포그라운드에서는 OS가 알림을 자동으로 표시하지 않으므로 직접 표시
    _showLocalNotification(message);
  }

  void _handleNotificationTap(RemoteMessage message) {
    // 알림 탭 시 라우팅 처리
    final screen = message.data['screen'];
    debugPrint('[Tap] screen=$screen');
    // NavigationService.navigateTo(screen);
  }

  void _sendTokenToServer(String token) {
    // 실제 서버 API 호출
    debugPrint('서버로 토큰 전송: $token');
  }

  // 토픽 구독/해제
  Future<void> subscribeToTopic(String topic) =>
      _messaging.subscribeToTopic(topic);

  Future<void> unsubscribeFromTopic(String topic) =>
      _messaging.unsubscribeFromTopic(topic);
}
```

---

## 5. 코드 예제 2 — 알림 탭 → 화면 이동 (딥링크 연동)

실무에서 가장 중요한 요구사항 중 하나는 "알림을 탭했을 때 특정 화면으로 이동"이다. 앱 상태에 따라 처리 방식이 다르기 때문에 중앙화된 라우팅 로직이 필요하다.

```dart
import 'package:firebase_messaging/firebase_messaging.dart';
import 'package:go_router/go_router.dart';

class NotificationRouter {
  // 앱이 아직 초기화되기 전의 대기 메시지를 저장
  static RemoteMessage? _pendingMessage;

  /// main()에서 Firebase 초기화 직후, runApp() 이전에 호출
  static Future<void> captureLaunchMessage() async {
    _pendingMessage = await FirebaseMessaging.instance.getInitialMessage();
  }

  /// MaterialApp 또는 GoRouter가 준비된 후 라우팅 실행
  static void handlePendingNavigation(BuildContext context) {
    if (_pendingMessage == null) return;
    _navigateFromMessage(context, _pendingMessage!);
    _pendingMessage = null;
  }

  static void _navigateFromMessage(BuildContext context, RemoteMessage message) {
    final data = message.data;
    final screen = data['screen'] as String?;
    final id = data['id'] as String?;

    switch (screen) {
      case 'order_detail':
        if (id != null) context.push('/orders/$id');
      case 'chat':
        final roomId = data['room_id'];
        if (roomId != null) context.push('/chat/$roomId');
      case 'promotion':
        context.push('/promotions');
      default:
        // 알 수 없는 화면은 홈으로
        context.go('/home');
    }
  }
}

// main.dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp();

  // 종료 상태에서 탭된 초기 메시지를 즉시 캡처
  await NotificationRouter.captureLaunchMessage();

  // 백그라운드 핸들러는 FirebaseMessaging.onMessage 리스너보다 먼저 등록해야 함
  FirebaseMessaging.onBackgroundMessage(_firebaseMessagingBackgroundHandler);

  runApp(const MyApp());
}

class MyApp extends StatefulWidget {
  const MyApp({super.key});

  @override
  State<MyApp> createState() => _MyAppState();
}

class _MyAppState extends State<MyApp> {
  final FcmService _fcmService = FcmService();

  @override
  void initState() {
    super.initState();
    _fcmService.initialize();

    // 백그라운드 탭 메시지 리스너 등록
    FirebaseMessaging.onMessageOpenedApp.listen((message) {
      if (mounted) {
        NotificationRouter._navigateFromMessage(context, message);
      }
    });
  }

  @override
  void didChangeDependencies() {
    super.didChangeDependencies();
    // 라우터가 준비된 후 종료 상태 탭 처리
    WidgetsBinding.instance.addPostFrameCallback((_) {
      NotificationRouter.handlePendingNavigation(context);
    });
  }

  @override
  Widget build(BuildContext context) {
    return MaterialApp.router(
      routerConfig: _router,
    );
  }
}

final _router = GoRouter(
  routes: [
    GoRoute(path: '/home', builder: (_, __) => const HomeScreen()),
    GoRoute(path: '/orders/:id', builder: (_, state) =>
        OrderDetailScreen(id: state.pathParameters['id']!)),
    GoRoute(path: '/chat/:roomId', builder: (_, state) =>
        ChatScreen(roomId: state.pathParameters['roomId']!)),
    GoRoute(path: '/promotions', builder: (_, __) => const PromotionScreen()),
  ],
);
```

---

## 6. FCM HTTP v1 API로 서버에서 메시지 보내기

2024년 6월부터 구 레거시 FCM HTTP API(정적 서버 키 방식)가 완전 종료됐다. 서버에서는 반드시 **HTTP v1 API**를 OAuth 2.0 액세스 토큰과 함께 사용해야 한다.

```kotlin
// Kotlin 서버 측 예제 (Firebase Admin SDK)
import com.google.firebase.messaging.FirebaseMessaging
import com.google.firebase.messaging.Message
import com.google.firebase.messaging.Notification
import com.google.firebase.messaging.AndroidConfig
import com.google.firebase.messaging.AndroidNotification

fun sendPushToUser(
    fcmToken: String,
    title: String,
    body: String,
    screen: String,
    id: String? = null
) {
    val message = Message.builder()
        // 알림 페이로드 (OS가 표시)
        .setNotification(
            Notification.builder()
                .setTitle(title)
                .setBody(body)
                .build()
        )
        // 데이터 페이로드 (앱 코드가 처리)
        .putData("screen", screen)
        .apply { if (id != null) putData("id", id) }
        // Android 전용 채널 및 우선순위 설정
        .setAndroidConfig(
            AndroidConfig.builder()
                .setNotification(
                    AndroidNotification.builder()
                        .setChannelId("high_importance_channel") // AndroidManifest와 일치해야 함
                        .setColor("#FF5252")
                        .setIcon("ic_notification")
                        .build()
                )
                .setPriority(AndroidConfig.Priority.HIGH)
                .build()
        )
        // 특정 기기로 전송
        .setToken(fcmToken)
        .build()

    val response = FirebaseMessaging.getInstance().send(message)
    println("Successfully sent message: $response")
}

// 토픽으로 전송하는 예제
fun sendToTopic(topic: String, title: String, body: String) {
    val message = Message.builder()
        .setNotification(
            Notification.builder()
                .setTitle(title)
                .setBody(body)
                .build()
        )
        .setTopic(topic) // 토픽 구독자 전원에게 전송
        .build()

    FirebaseMessaging.getInstance().send(message)
}

// 여러 기기에 동시 전송 (Multicast, 최대 500개)
fun sendMulticast(tokens: List<String>, title: String, body: String) {
    val multicastMessage = com.google.firebase.messaging.MulticastMessage.builder()
        .setNotification(
            Notification.builder()
                .setTitle(title)
                .setBody(body)
                .build()
        )
        .addAllTokens(tokens)
        .build()

    val response = FirebaseMessaging.getInstance().sendEachForMulticast(multicastMessage)
    println("${response.successCount} messages were sent successfully")
    // 실패한 토큰 처리 (DB에서 삭제 등)
    response.responses.forEachIndexed { index, sendResponse ->
        if (!sendResponse.isSuccessful) {
            val failedToken = tokens[index]
            println("Failed token: $failedToken, error: ${sendResponse.exception?.message}")
        }
    }
}
```

---

## 7. 주의사항 및 실전 팁

### 7.1 백그라운드 핸들러의 실행 시간 제한

Android에서 백그라운드 핸들러는 **최대 약 10초** 내에 완료해야 한다. 이 시간을 초과하면 OS가 강제 종료한다. 무거운 작업(API 호출, DB 쓰기)은 WorkManager로 위임하는 것이 안전하다.

```dart
@pragma('vm:entry-point')
Future<void> _firebaseMessagingBackgroundHandler(RemoteMessage message) async {
  await Firebase.initializeApp();
  // 무거운 작업은 WorkManager로 위임 (android_alarm_manager_plus 또는 workmanager 패키지)
  // 여기서는 알림 표시만 빠르게 처리
  await _showLocalNotification(message);
}
```

### 7.2 Android 13+ 알림 권한 처리

Android 13(API 33)부터 `POST_NOTIFICATIONS` 권한이 런타임 권한으로 변경됐다. `firebase_messaging`의 `requestPermission()`이 내부적으로 처리하지만, 사용자가 거부한 경우 앱 설정 화면으로 유도해야 한다.

```dart
final settings = await FirebaseMessaging.instance.requestPermission();
if (settings.authorizationStatus == AuthorizationStatus.denied) {
  // 사용자에게 설정 화면 유도 UI 표시
  showPermissionRationaleDialog();
}
```

### 7.3 토큰 만료와 갱신 전략

FCM 토큰은 다음 상황에서 갱신된다:
- 앱 데이터 삭제
- 새 기기 복원
- Google Play Services 업데이트
- 앱 재설치

반드시 `onTokenRefresh` 리스너를 등록하고, 갱신된 토큰을 즉시 서버에 업로드해야 한다. 토큰을 서버에 저장할 때는 **userId + deviceId + token** 조합으로 관리해 멀티 디바이스 지원과 중복 전송을 방지하라.

### 7.4 포그라운드에서 알림이 보이지 않는 문제

Android에서 포그라운드 상태의 Notification 메시지는 기본적으로 OS 알림이 표시되지 않는다. `onMessage` 리스너에서 반드시 `flutter_local_notifications`로 직접 알림을 표시해야 한다. iOS에서는 `setForegroundNotificationPresentationOptions(alert: true, ...)`를 호출하면 OS가 자동으로 표시한다.

### 7.5 릴리즈 빌드 점검 체크리스트

- [ ] `google-services.json` (Android), `GoogleService-Info.plist` (iOS) 포함 여부
- [ ] `@pragma('vm:entry-point')` 애노테이션 확인
- [ ] ProGuard/R8 규칙에 Firebase 예외 추가 (`-keep class com.google.firebase.**`)
- [ ] iOS Provisioning Profile에 Push Notifications entitlement 포함
- [ ] Android 채널 ID가 AndroidManifest의 `meta-data`와 일치

---

## 8. 마치며

Flutter FCM의 완전한 구현은 단순히 `onMessage`를 구독하는 것이 아니다. 세 가지 앱 상태 각각에 맞는 핸들러를 올바르게 등록하고, isolate 경계를 이해하며, Android/iOS 플랫폼별 설정 차이를 파악해야 한다. 특히 `@pragma('vm:entry-point')` 누락, 백그라운드 핸들러 내 Firebase 미초기화, Android 채널 ID 불일치는 가장 흔한 실수다. 위의 패턴을 기반으로 구현하면 어떤 앱 상태에서도 안정적으로 메시지를 수신하고, 알림 탭 시 올바른 화면으로 이동하는 완전한 FCM 구현을 갖출 수 있다.

## 참고 자료
- [firebase_messaging 패키지 (pub.dev)](https://pub.dev/packages/firebase_messaging)
- [flutter_local_notifications 패키지 (pub.dev)](https://pub.dev/packages/flutter_local_notifications)
