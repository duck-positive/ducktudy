---
layout: post
title: "Flutter Isolate & Compute 심화: 메인 스레드를 지키는 병렬처리 전략"
date: 2026-06-10
categories: [flutter]
tags: [flutter, dart, isolate, compute, concurrency, performance, parallel]
---

Flutter 앱에서 무거운 연산을 실행할 때 UI가 버벅거리는 경험을 해본 적 있나요? 대용량 JSON 파싱, 이미지 처리, 암호화 연산 등 CPU 집약적인 작업은 메인 Isolate의 프레임 타임을 초과하여 앱 반응성을 저하시킵니다. Flutter/Dart에서 이 문제를 해결하는 핵심 기술이 바로 **Isolate**와 **compute** 함수입니다.

## Dart Isolate란?

Dart는 싱글 스레드 모델 위에 구축되었지만, 실제로는 **Isolate**라는 독립적인 실행 단위를 통해 병렬 처리를 지원합니다. Java의 Thread와 달리, Isolate는 메모리를 공유하지 않습니다. 각 Isolate는 완전히 격리된(isolated) 힙 메모리를 가지며, Isolate 간 통신은 오직 **메시지 패싱(Message Passing)**을 통해서만 이루어집니다.

이 설계는 Race Condition이나 Deadlock 같은 멀티스레드 문제를 원천적으로 차단하지만, 동시에 데이터를 공유하는 방식이 기존 스레드 모델과 다릅니다.

### Flutter의 Isolate 구조

Flutter 앱은 기본적으로 다음 Isolate들 위에서 동작합니다:

- **Main Isolate (UI Isolate)**: UI 렌더링과 사용자 입력 처리를 담당. 초당 60프레임(또는 120프레임)을 유지해야 합니다.
- **Background Isolate**: 개발자가 생성하는 작업용 Isolate입니다.
- **Flutter Engine Isolate**: 네이티브 코드와 통신하는 엔진 레벨 Isolate입니다.

## 왜 Isolate가 필요한가?

Flutter의 프레임 렌더링 예산은 **16ms(60fps 기준)**입니다. 이 시간 내에 모든 UI 계산과 렌더링을 완료해야 합니다. `async/await`만으로는 CPU 집약적 작업을 해결할 수 없습니다.

`async/await`는 I/O 작업(네트워크, 파일)에서는 효과적이지만, CPU를 지속적으로 점유하는 계산 작업은 이벤트 루프를 차단합니다. 다음과 같은 시나리오에서 Isolate가 필수입니다:

- 수만 개 항목이 담긴 대용량 JSON 파싱
- 이미지 필터/압축 처리
- 암호화/복호화 연산
- 복잡한 수학적 계산 (ML 추론 전처리 등)
- 대용량 데이터 정렬 및 필터링

### async/await의 한계 확인

```dart
// ❌ async/await는 CPU 집약적 작업을 차단합니다
Future<int> badExample() async {
  // 이 작업은 메인 스레드를 약 1초간 차단합니다
  // UI 프리즈 발생!
  int sum = 0;
  for (int i = 0; i < 100000000; i++) {
    sum += i;
  }
  return sum;
}

// ✅ Isolate를 사용하면 메인 스레드를 차단하지 않습니다
Future<int> goodExample() async {
  return Isolate.run(() {
    int sum = 0;
    for (int i = 0; i < 100000000; i++) {
      sum += i;
    }
    return sum;
  });
}
```

## 기본 구현: Isolate.run()

Dart 2.19부터 `Isolate.run()`이 도입되어 단발성 병렬 작업을 훨씬 간결하게 처리할 수 있습니다. Flutter에서는 동일하게 `compute()` 함수를 사용하며, 이는 내부적으로 `Isolate.run()`을 감쌉니다.

## 실제 구현 예제 1: compute()로 대용량 JSON 파싱

실무에서 가장 흔히 만나는 Isolate 활용 사례는 대용량 JSON 파싱입니다. 수천 개의 상품 목록을 API에서 받아와 파싱해야 한다면 메인 스레드에서 처리할 경우 UI가 순간적으로 멈춥니다.

```dart
import 'dart:convert';
import 'package:flutter/foundation.dart';

// Top-level 또는 static 함수여야 합니다
List<Product> _parseProducts(String jsonString) {
  final List<dynamic> jsonList = jsonDecode(jsonString) as List<dynamic>;
  return jsonList
      .map((json) => Product.fromJson(json as Map<String, dynamic>))
      .toList();
}

class ProductRepository {
  Future<List<Product>> fetchProducts() async {
    // 네트워크 요청 (I/O → async/await로 충분)
    final response = await httpClient.get('/api/products');

    // JSON 파싱 (CPU 집약 → Isolate 필요)
    // compute는 별도 Isolate에서 _parseProducts를 실행합니다
    return compute(_parseProducts, response.body);
  }
}

class Product {
  final int id;
  final String name;
  final double price;
  final String imageUrl;

  Product({
    required this.id,
    required this.name,
    required this.price,
    required this.imageUrl,
  });

  factory Product.fromJson(Map<String, dynamic> json) {
    return Product(
      id: json['id'] as int,
      name: json['name'] as String,
      price: (json['price'] as num).toDouble(),
      imageUrl: json['image_url'] as String,
    );
  }
}

// Flutter Widget에서 사용 예시
class ProductListScreen extends StatelessWidget {
  const ProductListScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return FutureBuilder<List<Product>>(
      future: ProductRepository().fetchProducts(),
      builder: (context, snapshot) {
        if (snapshot.connectionState == ConnectionState.waiting) {
          return const CircularProgressIndicator();
        }
        if (snapshot.hasError) {
          return Text('오류: ${snapshot.error}');
        }
        final products = snapshot.data!;
        return ListView.builder(
          itemCount: products.length,
          itemBuilder: (context, index) => ListTile(
            title: Text(products[index].name),
            subtitle: Text('₩${products[index].price}'),
          ),
        );
      },
    );
  }
}
```

`compute()` 함수는 두 가지 중요한 제약을 가집니다:

1. **Top-level 또는 static 함수**: Isolate 간에 클로저를 직접 전달할 수 없으므로, 반드시 최상위 또는 정적 함수여야 합니다.
2. **직렬화 가능한 메시지**: Isolate 간 메시지는 복사되므로, 직렬화 가능한 타입이어야 합니다.

## 실제 구현 예제 2: 장기 실행 Isolate — SendPort & ReceivePort

단발성 작업이 아니라 지속적으로 통신이 필요한 경우(예: 실시간 데이터 처리 파이프라인), `SendPort`와 `ReceivePort`를 사용해 양방향 통신 채널을 구성합니다. 이미지 필터를 여러 장 연속 처리하는 Worker 패턴을 예로 들겠습니다.

```dart
import 'dart:async';
import 'dart:isolate';
import 'dart:typed_data';

// ─── 메시지 타입 정의 ───────────────────────────────────

class _ImageTask {
  final int taskId;
  final Uint8List imageBytes;
  _ImageTask({required this.taskId, required this.imageBytes});
}

class _ImageResult {
  final int taskId;
  final Uint8List processedBytes;
  _ImageResult({required this.taskId, required this.processedBytes});
}

// ─── Worker Isolate 진입점 ──────────────────────────────

// Release 모드에서 트리 쉐이킹으로 제거되지 않도록 어노테이션 필수
@pragma('vm:entry-point')
void _imageWorkerEntryPoint(SendPort mainSendPort) {
  final workerReceivePort = ReceivePort();

  // 1단계: Worker의 SendPort를 Main에 전달 (초기 핸드셰이크)
  mainSendPort.send(workerReceivePort.sendPort);

  // 2단계: Main에서 오는 작업 메시지 수신 대기
  workerReceivePort.listen((message) {
    if (message is _ImageTask) {
      final processed = _applyGrayscaleFilter(message.imageBytes);
      mainSendPort.send(_ImageResult(
        taskId: message.taskId,
        processedBytes: processed,
      ));
    } else if (message == 'shutdown') {
      workerReceivePort.close();
    }
  });
}

// CPU 집약적 이미지 처리 (RGBA 포맷 가정)
Uint8List _applyGrayscaleFilter(Uint8List bytes) {
  final result = Uint8List.fromList(bytes);
  for (int i = 0; i < result.length; i += 4) {
    final r = result[i];
    final g = result[i + 1];
    final b = result[i + 2];
    // BT.601 루미난스 공식으로 그레이스케일 변환
    final gray = (0.299 * r + 0.587 * g + 0.114 * b).round().clamp(0, 255);
    result[i] = gray;
    result[i + 1] = gray;
    result[i + 2] = gray;
    // Alpha 채널(result[i+3])은 그대로 유지
  }
  return result;
}

// ─── Main Isolate의 Worker 관리 클래스 ─────────────────

class ImageProcessingWorker {
  Isolate? _isolate;
  SendPort? _workerSendPort;
  late ReceivePort _mainReceivePort;
  final Map<int, Completer<Uint8List>> _pendingTasks = {};
  int _taskCounter = 0;
  bool _initialized = false;

  Future<void> initialize() async {
    _mainReceivePort = ReceivePort();

    // Worker Isolate 스폰
    _isolate = await Isolate.spawn(
      _imageWorkerEntryPoint,
      _mainReceivePort.sendPort,
    );

    // Main ReceivePort에서 메시지 수신 처리
    final completer = Completer<void>();
    _mainReceivePort.listen((message) {
      if (message is SendPort) {
        // 핸드셰이크 완료: Worker의 SendPort 저장
        _workerSendPort = message;
        _initialized = true;
        completer.complete();
      } else if (message is _ImageResult) {
        // 처리 완료된 이미지 결과 수신
        final taskCompleter = _pendingTasks.remove(message.taskId);
        taskCompleter?.complete(message.processedBytes);
      }
    });

    // 초기화 완료 대기
    await completer.future;
  }

  Future<Uint8List> processImage(Uint8List imageBytes) async {
    if (!_initialized) throw StateError('Worker가 초기화되지 않았습니다.');

    final taskId = _taskCounter++;
    final completer = Completer<Uint8List>();
    _pendingTasks[taskId] = completer;

    _workerSendPort!.send(_ImageTask(
      taskId: taskId,
      imageBytes: imageBytes,
    ));

    return completer.future;
  }

  void dispose() {
    _workerSendPort?.send('shutdown');
    _mainReceivePort.close();
    _isolate?.kill(priority: Isolate.immediate);
    _pendingTasks.clear();
  }
}

// ─── Flutter Widget에서 사용 예시 ─────────────────────

class ImageFilterScreen extends StatefulWidget {
  const ImageFilterScreen({super.key});

  @override
  State<ImageFilterScreen> createState() => _ImageFilterScreenState();
}

class _ImageFilterScreenState extends State<ImageFilterScreen> {
  final _worker = ImageProcessingWorker();
  Uint8List? _processedImage;

  @override
  void initState() {
    super.initState();
    _worker.initialize();
  }

  Future<void> _applyFilter(Uint8List originalBytes) async {
    final result = await _worker.processImage(originalBytes);
    setState(() => _processedImage = result);
  }

  @override
  void dispose() {
    _worker.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('이미지 필터')),
      body: _processedImage != null
          ? Image.memory(_processedImage!)
          : const Center(child: Text('이미지를 선택하세요')),
    );
  }
}
```

이 패턴의 핵심은 Isolate를 매번 생성하지 않고 **재활용**한다는 점입니다. 이미지 100장을 처리해야 한다면 Isolate 생성 오버헤드 없이 Worker에 작업만 순차적으로 전달할 수 있습니다.

## 주의사항 및 팁

### 1. Top-level 함수 제약과 클로저 불가

```dart
// ❌ 잘못된 예 - 클로저 캡처 불가
final threshold = 128;
// 컴파일은 되지만 런타임 오류 발생
final result = await compute((bytes) => _applyThreshold(bytes, threshold), imageBytes);

// ✅ 올바른 예 - 데이터를 메시지에 포함하거나 top-level 함수 사용
class _FilterParams {
  final Uint8List bytes;
  final int threshold;
  _FilterParams(this.bytes, this.threshold);
}

Uint8List _applyThresholdFilter(_FilterParams params) {
  // params.threshold 사용
  return params.bytes; // 실제 처리 로직
}

final result = await compute(
  _applyThresholdFilter,
  _FilterParams(imageBytes, 128),
);
```

### 2. 직렬화 가능한 타입만 전달 가능

Isolate 간 메시지는 **복사(copy)**됩니다. 전달 가능한 타입:

| 타입 | 전달 가능 여부 | 특이사항 |
|------|--------------|---------|
| `int`, `double`, `bool`, `String` | ✅ | - |
| `List`, `Map` | ✅ | 내부 원소도 직렬화 가능해야 함 |
| `Uint8List`, `Float32List` 등 TypedData | ✅ | 효율적 전송 |
| `SendPort` | ✅ | 통신 채널 전달 시 사용 |
| 사용자 정의 클래스 | ⚠️ | Dart 3.x 이전엔 직접 직렬화 필요 |
| `Stream`, `Future` | ❌ | 전달 불가 |
| Widget, BuildContext | ❌ | 절대 전달 불가 |

### 3. Flutter 플러그인은 Background Isolate에서 직접 사용 불가

일반 `Isolate.spawn()`으로 생성된 Isolate에서는 `shared_preferences`, `path_provider` 같은 Flutter 플러그인을 직접 호출할 수 없습니다. Dart 3.x부터는 `BackgroundIsolateBinaryMessenger`로 해결할 수 있습니다:

```dart
import 'dart:ui';
import 'package:flutter/widgets.dart';

@pragma('vm:entry-point')
void backgroundIsolateWithPlugin(List<Object?> args) async {
  final sendPort = args[0] as SendPort;

  // RootIsolateToken으로 플러그인 바이너리 메신저 초기화
  final token = args[1] as RootIsolateToken;
  BackgroundIsolateBinaryMessenger.ensureInitialized(token);

  // 이제 Flutter 플러그인 사용 가능
  final prefs = await SharedPreferences.getInstance();
  final value = prefs.getString('key') ?? 'default';

  sendPort.send(value);
}

// Main Isolate에서 호출
Future<void> spawnWithPluginSupport() async {
  final receivePort = ReceivePort();
  final token = RootIsolateToken.instance!; // Main Isolate에서만 얻을 수 있음

  await Isolate.spawn(
    backgroundIsolateWithPlugin,
    [receivePort.sendPort, token],
  );
}
```

### 4. Isolate 생성 비용 인식

Isolate 생성에는 약 **5~15ms**의 오버헤드가 있습니다. 작은 연산을 자주 Isolate로 보내면 오히려 성능이 저하될 수 있습니다. 기준선:

- 연산 시간 < 1ms → 메인 Isolate에서 처리
- 연산 시간 > 16ms → Isolate 사용 고려
- 반복 작업 → Worker 패턴으로 Isolate 재활용

### 5. DevTools로 Isolate 성능 모니터링

Flutter DevTools의 **Timeline** 탭에서 각 Isolate의 CPU 사용률과 실행 타임라인을 시각적으로 확인할 수 있습니다. **Memory** 탭에서는 Isolate별 힙 사용량도 추적 가능합니다. Isolate 목록에서 메인 Isolate 외에 추가 Isolate가 활성 상태인지, 메모리를 과도하게 소비하진 않는지 확인하세요.

## Isolate 방법별 비교 정리

| 방법 | 적합한 상황 | Dart 버전 | 특징 |
|------|------------|-----------|------|
| `compute()` | 단발성 CPU 집약 작업 | Flutter 전용 | 가장 간단, Isolate 풀 관리 |
| `Isolate.run()` | 단발성, Dart 코드만 | 2.19+ | compute와 유사, Dart 표준 |
| `Isolate.spawn()` | 장기 실행, 양방향 통신 | 모든 버전 | SendPort/ReceivePort로 완전 제어 |
| `flutter_isolate` | 플러그인 필요한 Background 작업 | Flutter 전용 | 플러그인 지원 포함 |

Flutter의 Isolate는 메인 스레드를 보호하는 강력한 도구입니다. `compute()`로 시작해 필요에 따라 양방향 통신 패턴으로 확장하는 전략을 권장합니다. 무엇보다 중요한 것은 **실제로 UI 프리즈가 발생하는지 DevTools로 먼저 측정**하고, 병목이 확인된 경우에만 Isolate를 도입하는 것입니다. 섣부른 최적화는 오히려 코드 복잡성만 높입니다.

## 참고 자료

- [Concurrency and isolates — Flutter 공식 문서](https://docs.flutter.dev/perf/isolates)
- [Isolates — Dart 공식 문서](https://dart.dev/language/isolates)
- [flutter_isolate — pub.dev 패키지](https://pub.dev/packages/flutter_isolate)
