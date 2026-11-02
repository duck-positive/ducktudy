---
layout: post
title: "Flutter firebase_ai 심화: Gemini 2.5 Flash로 멀티모달 AI 채팅 앱 구현하기"
date: 2026-11-02
categories: [android, flutter]
tags: [flutter, dart, firebase, gemini, ai, llm, multimodal, function-calling, firebase_ai]
---

Flutter 앱에 생성형 AI를 직접 통합하는 수요가 급격히 늘어나고 있다. Google은 한때 `google_generative_ai` 패키지를 통해 Gemini API를 제공했지만, Gemini 2.0 출시 이후 클라이언트 SDK를 **Firebase AI Logic** (`firebase_ai`) 으로 통합했다. 이 패키지(현재 5.x)는 Firebase App Check를 통한 보안 강화, Vertex AI 백엔드 지원, 그리고 멀티모달·함수 호출 등 최신 기능을 모두 제공한다. 이 글에서는 `firebase_ai`를 사용해 텍스트 생성, 멀티턴 채팅, 이미지 이해(멀티모달), 그리고 함수 호출(Function Calling)까지 구현하는 방법을 단계별로 살펴본다.

---

## 왜 firebase_ai인가?

### google_generative_ai의 폐기

구글은 `google_generative_ai` 패키지(마지막 버전 0.4.7)의 추가 개발을 중단하고 `firebase_ai`로 이전할 것을 공식 권고하고 있다. `firebase_ai`는 다음 이점을 제공한다.

- **Firebase App Check 통합**: 클라이언트 API 키를 모바일 앱에 하드코딩하는 위험을 줄이고, App Check 토큰으로 요청을 검증한다.
- **Vertex AI 및 Google AI 백엔드 모두 지원**: 개발 단계는 `GoogleAIBackend`, 프로덕션은 `VertexAIBackend`로 손쉽게 전환 가능하다.
- **최신 Gemini 모델 지원**: Gemini 2.5 Flash, Gemini 2.0 Pro 등 최신 모델을 즉시 사용할 수 있다.
- **멀티모달, 함수 호출, 스트리밍, 라이브 세션 등 고급 기능** 제공.

### 언제 클라이언트 SDK를 쓰면 안 되나?

클라이언트 SDK는 개발/프로토타이핑에 적합하다. 프로덕션 앱에서 민감한 데이터를 다루거나 비용 제어가 중요한 경우에는 서버를 중간에 두는 아키텍처를 권장한다. 단, Firebase App Check를 활성화하면 허가받지 않은 클라이언트의 무분별한 API 호출을 차단할 수 있어 보안 수준이 크게 향상된다.

---

## 프로젝트 설정

### 1. 패키지 설치

```yaml
# pubspec.yaml
dependencies:
  flutter:
    sdk: flutter
  firebase_core: ^3.10.0
  firebase_ai: ^5.1.0
```

```bash
flutter pub get
```

### 2. Firebase 초기화

`main.dart`에서 Firebase를 초기화한다.

```dart
import 'package:firebase_core/firebase_core.dart';
import 'firebase_options.dart'; // FlutterFire CLI로 생성

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp(options: DefaultFirebaseOptions.currentPlatform);
  runApp(const MyApp());
}
```

---

## 구현 예제 1: 멀티턴 채팅 구현

가장 기본적인 사용 사례인 대화형 채팅을 구현한다. `ChatSession`을 이용하면 이전 대화 이력을 자동으로 관리해 준다.

```dart
import 'package:firebase_ai/firebase_ai.dart';
import 'package:flutter/material.dart';

class ChatRepository {
  late final ChatSession _session;

  ChatRepository() {
    final model = FirebaseAI.googleAI().generativeModel(
      model: 'gemini-2.5-flash',
      generationConfig: GenerationConfig(
        maxOutputTokens: 1024,
        temperature: 0.7,
      ),
      systemInstruction: Content.system(
        '당신은 Android와 Flutter 전문가입니다. '
        '기술적인 질문에 한국어로 친절하게 답변해 주세요.',
      ),
    );
    _session = model.startChat();
  }

  /// 메시지를 스트리밍으로 전송하고 응답 텍스트를 반환한다.
  Stream<String> sendMessage(String userText) async* {
    final response = _session.sendMessageStream(
      Content.text(userText),
    );
    await for (final chunk in response) {
      final text = chunk.text;
      if (text != null) yield text;
    }
  }

  /// 현재 대화 이력을 반환한다.
  List<Content> get history => _session.history;
}

// ──────────────────────────────────────────────────────────────────
// UI 레이어: StreamBuilder로 스트리밍 응답을 실시간으로 표시
// ──────────────────────────────────────────────────────────────────

class ChatScreen extends StatefulWidget {
  const ChatScreen({super.key});

  @override
  State<ChatScreen> createState() => _ChatScreenState();
}

class _ChatScreenState extends State<ChatScreen> {
  final _repo = ChatRepository();
  final _controller = TextEditingController();
  final _messages = <_Message>[];
  bool _isGenerating = false;

  Future<void> _send() async {
    final text = _controller.text.trim();
    if (text.isEmpty || _isGenerating) return;

    _controller.clear();
    setState(() {
      _messages.add(_Message(role: 'user', text: text));
      _messages.add(_Message(role: 'model', text: ''));
      _isGenerating = true;
    });

    final buffer = StringBuffer();
    await for (final chunk in _repo.sendMessage(text)) {
      buffer.write(chunk);
      setState(() {
        _messages.last = _Message(role: 'model', text: buffer.toString());
      });
    }
    setState(() => _isGenerating = false);
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Gemini 채팅')),
      body: Column(
        children: [
          Expanded(
            child: ListView.builder(
              padding: const EdgeInsets.all(12),
              itemCount: _messages.length,
              itemBuilder: (_, i) {
                final msg = _messages[i];
                final isUser = msg.role == 'user';
                return Align(
                  alignment:
                      isUser ? Alignment.centerRight : Alignment.centerLeft,
                  child: Container(
                    margin: const EdgeInsets.symmetric(vertical: 4),
                    padding: const EdgeInsets.all(12),
                    decoration: BoxDecoration(
                      color: isUser
                          ? Theme.of(context).colorScheme.primary
                          : Theme.of(context).colorScheme.surfaceVariant,
                      borderRadius: BorderRadius.circular(12),
                    ),
                    child: Text(
                      msg.text.isEmpty ? '...' : msg.text,
                      style: TextStyle(
                        color: isUser
                            ? Theme.of(context).colorScheme.onPrimary
                            : null,
                      ),
                    ),
                  ),
                );
              },
            ),
          ),
          Padding(
            padding: const EdgeInsets.all(8.0),
            child: Row(
              children: [
                Expanded(
                  child: TextField(
                    controller: _controller,
                    decoration: const InputDecoration(
                      hintText: '메시지를 입력하세요...',
                      border: OutlineInputBorder(),
                    ),
                    onSubmitted: (_) => _send(),
                  ),
                ),
                const SizedBox(width: 8),
                IconButton.filled(
                  icon: const Icon(Icons.send),
                  onPressed: _isGenerating ? null : _send,
                ),
              ],
            ),
          ),
        ],
      ),
    );
  }
}

class _Message {
  final String role;
  final String text;
  const _Message({required this.role, required this.text});
}
```

### 핵심 포인트

- `startChat()` 로 생성된 `ChatSession`은 내부적으로 `history` 리스트를 유지한다. 매 요청 시 이전 대화가 자동으로 포함된다.
- `sendMessageStream()`은 `Stream<GenerateContentResponse>`를 반환하며, `chunk.text`로 각 청크를 실시간으로 추출할 수 있다.
- `systemInstruction`을 통해 모델의 역할과 언어 설정을 초기화 시점에 고정할 수 있다.

---

## 구현 예제 2: 멀티모달 입력 + 함수 호출

이미지와 텍스트를 함께 보내고, 모델이 특정 함수를 호출하도록 하는 고급 패턴이다. 예시에서는 갤러리에서 이미지를 선택하면 모델이 이미지를 분석하고, 날씨 조회 함수를 호출해 추가 정보를 제공한다.

```dart
import 'dart:io';
import 'package:firebase_ai/firebase_ai.dart';
import 'package:image_picker/image_picker.dart';

// ── 함수 선언 정의 ──────────────────────────────────────────────────
final weatherTool = FunctionDeclaration(
  'get_current_weather',
  'Returns current weather for a given city.',
  parameters: {
    'city': Schema.string(description: '도시 이름 (예: Seoul)'),
    'unit': Schema.enumString(
      enumValues: ['celsius', 'fahrenheit'],
      description: '온도 단위',
    ),
  },
  requiredParameters: ['city'],
);

// ── 모델 생성 (툴 포함) ──────────────────────────────────────────────
GenerativeModel _buildModelWithTools() {
  return FirebaseAI.googleAI().generativeModel(
    model: 'gemini-2.5-flash',
    tools: [Tool.functionDeclarations([weatherTool])],
    toolConfig: ToolConfig(
      functionCallingConfig: FunctionCallingConfig(
        mode: FunctionCallingMode.auto,
      ),
    ),
  );
}

// ── 실제 함수 실행 (서버 호출 또는 로컬 로직) ────────────────────────
Map<String, Object?> _executeWeatherFunction(Map<String, Object?> args) {
  final city = args['city'] as String;
  final unit = args['unit'] as String? ?? 'celsius';
  // 실제 구현에서는 외부 API를 호출한다.
  return {
    'city': city,
    'temperature': unit == 'celsius' ? 22 : 71,
    'unit': unit,
    'condition': '맑음',
    'humidity': '55%',
  };
}

// ── 멀티모달 + 함수 호출 오케스트레이션 ──────────────────────────────
Future<String> analyzeImageWithWeather(File imageFile) async {
  final model = _buildModelWithTools();
  final imageBytes = await imageFile.readAsBytes();

  // 첫 번째 요청: 이미지 + 텍스트 전송
  var response = await model.generateContent([
    Content.multi([
      DataPart('image/jpeg', imageBytes),
      TextPart(
        '이 이미지를 분석하고, 사진 속 장소가 어느 도시인지 추측하여 '
        'get_current_weather 함수로 현재 날씨를 조회해 주세요.',
      ),
    ]),
  ]);

  // 함수 호출 루프: 모델이 함수 호출을 요청할 때까지 반복
  while (response.functionCalls.isNotEmpty) {
    final functionResults = <FunctionResponse>[];

    for (final call in response.functionCalls) {
      if (call.name == 'get_current_weather') {
        final result = _executeWeatherFunction(call.args);
        functionResults.add(FunctionResponse(call.name, result));
      }
    }

    // 함수 결과를 모델에 반환하여 최종 응답 생성
    response = await model.generateContent([
      Content.model(response.candidates.first.content.parts),
      Content.functionResponses(functionResults),
    ]);
  }

  return response.text ?? '응답을 생성할 수 없습니다.';
}

// ── 사용 예시 ─────────────────────────────────────────────────────
Future<void> pickAndAnalyze() async {
  final picker = ImagePicker();
  final picked = await picker.pickImage(source: ImageSource.gallery);
  if (picked == null) return;

  final result = await analyzeImageWithWeather(File(picked.path));
  debugPrint('분석 결과: $result');
}
```

### 함수 호출 흐름 설명

1. **모델에 Tool 정의를 등록**: `FunctionDeclaration`으로 함수의 이름, 설명, 파라미터 스키마를 정의한다.
2. **모델이 함수 호출을 결정**: 프롬프트 맥락에 따라 모델은 텍스트 응답 대신 `functionCalls`가 채워진 응답을 반환한다.
3. **앱에서 실제 함수를 실행**: `call.name`과 `call.args`를 사용해 로컬 또는 외부 API를 호출한다.
4. **결과를 다시 모델에 전달**: `Content.functionResponses()`로 결과를 모델에 피드백하면 최종 자연어 응답이 생성된다.

---

## 주의사항 및 팁

### 1. API 키 보안

클라이언트 앱에 API 키를 직접 포함하면 키가 노출될 위험이 있다. **Firebase App Check**를 활성화해 허가된 앱 빌드에서만 호출이 허용되도록 설정하는 것이 필수다.

```dart
// App Check 초기화 예시 (main.dart)
import 'package:firebase_app_check/firebase_app_check.dart';

await FirebaseAppCheck.instance.activate(
  androidProvider: AndroidProvider.playIntegrity,
  appleProvider: AppleProvider.appAttest,
);
```

프로덕션 환경이라면 서버를 중간에 두어 실제 Gemini API 호출을 서버에서 수행하는 아키텍처를 권장한다.

### 2. 비용과 토큰 관리

- `GenerationConfig`의 `maxOutputTokens`를 적절히 설정해 예상치 못한 과금을 방지한다.
- 이미지 입력은 텍스트보다 훨씬 많은 토큰을 소모한다. 이미지 크기를 전송 전에 압축할 것을 권장한다.
- `response.usageMetadata`로 각 요청의 `promptTokenCount`와 `candidatesTokenCount`를 모니터링한다.

```dart
final usage = response.usageMetadata;
debugPrint(
  'Prompt: ${usage?.promptTokenCount}, '
  'Output: ${usage?.candidatesTokenCount}',
);
```

### 3. 에러 처리

Gemini API는 안전 필터를 내장하고 있어 특정 콘텐츠는 `HARM_CATEGORY_*`로 차단될 수 있다.

```dart
try {
  final response = await model.generateContent([Content.text(prompt)]);
  if (response.candidates.isEmpty) {
    // 안전 필터에 의해 차단된 경우
    final reason = response.promptFeedback?.blockReason;
    throw Exception('응답 차단됨: $reason');
  }
  return response.text ?? '';
} on FirebaseException catch (e) {
  // 네트워크 오류, 쿼터 초과 등
  throw Exception('Firebase 오류: ${e.message}');
}
```

### 4. 스트리밍 vs 일반 응답

긴 텍스트 응답에는 반드시 `generateContentStream()`을 사용한다. 스트리밍을 쓰면 첫 번째 토큰이 출력될 때까지의 체감 대기 시간(TTFT)이 크게 줄어들어 UX가 향상된다.

### 5. VertexAI 백엔드로의 전환

개발 완료 후 프로덕션에서는 `GoogleAIBackend` 대신 `VertexAIBackend`를 사용해 엔터프라이즈급 SLA와 데이터 거버넌스를 적용할 수 있다.

```dart
// 개발용
final model = FirebaseAI.googleAI().generativeModel(model: 'gemini-2.5-flash');

// 프로덕션 (Vertex AI 백엔드)
final model = FirebaseAI.vertexAI().generativeModel(model: 'gemini-2.5-flash');
```

---

## 마치며

`firebase_ai` 패키지는 Flutter에서 Gemini API를 통합하는 공식 표준이 되었다. 스트리밍 채팅, 멀티모달 입력, 함수 호출이라는 세 가지 핵심 패턴을 익히면 AI 기반 앱의 대부분의 사용 사례를 커버할 수 있다. 프로덕션 배포 전에는 반드시 App Check를 활성화하고, 토큰 사용량 모니터링을 통해 비용을 관리하는 것을 잊지 말자.

## 참고 자료
- [firebase_ai 패키지 (pub.dev)](https://pub.dev/packages/firebase_ai)
- [google_generative_ai 패키지 - 폐기 안내 (pub.dev)](https://pub.dev/packages/google_generative_ai)
