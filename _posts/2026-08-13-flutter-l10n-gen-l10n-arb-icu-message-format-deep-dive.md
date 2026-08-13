---
layout: post
title: "Flutter 국제화(i18n) 심화: gen-l10n·ARB·ICU 메시지 포맷으로 다국어 앱 완전 정복"
date: 2026-08-13
categories: [android, flutter]
tags: [flutter, l10n, i18n, localization, internationalization, arb, gen-l10n, intl, icu, dart]
---

모바일 앱이 글로벌 시장에 진출하려면 단순한 번역 문자열 교체를 넘어, 복수형·날짜·통화·RTL 레이아웃까지 언어별로 정확하게 처리해야 합니다. Flutter는 `flutter_localizations` 패키지와 `gen-l10n` 코드 생성 도구를 통해 이 모든 것을 타입 안전하게 지원합니다. 이 글에서는 ARB 파일 설계부터 ICU 메시지 포맷, 런타임 언어 전환까지 실무 수준의 국제화 전략을 완전히 파헤칩니다.

---

## 왜 Flutter l10n이 어려운가?

국제화(i18n)를 처음 접하면 "그냥 Map에 문자열 넣으면 되지 않나?"라고 생각하기 쉽습니다. 하지만 실무에서는 다음 문제들이 기다리고 있습니다.

- **복수형(Plural)**: "1개 알림"과 "3개 알림"처럼 수량에 따라 문자열 형태가 달라집니다. 아랍어는 무려 6가지 복수형을 요구합니다.
- **성별(Gender/Select)**: "He joined"와 "She joined"처럼 문맥에 따른 분기가 필요합니다.
- **날짜·숫자 형식**: 한국은 `2026년 8월 13일`, 미국은 `August 13, 2026`, 독일은 `13. August 2026`처럼 형식이 다릅니다.
- **RTL(오른쪽→왼쪽) 레이아웃**: 아랍어·히브리어는 UI 방향 자체가 반전됩니다.
- **런타임 언어 전환**: 앱 재시작 없이 언어를 바꿔야 할 때 상태 관리와 연동이 필요합니다.

Flutter 공식 도구 체인(`gen-l10n` + `intl` 패키지)은 이 모든 문제를 **ARB 파일 하나**에서 출발해 **컴파일 타임에 타입 안전한 Dart 코드**로 해결합니다.

---

## 핵심 개념: ARB 파일이란?

ARB(Application Resource Bundle)는 Google이 정의한 JSON 기반 번역 파일 형식입니다. 각 키가 번역 문자열이며, `@키이름` 메타데이터로 플레이스홀더·설명·타입을 선언합니다.

```json
{
  "@@locale": "ko",
  "helloUser": "안녕하세요, {userName}님!",
  "@helloUser": {
    "description": "사용자를 환영하는 메시지",
    "placeholders": {
      "userName": {
        "type": "String",
        "example": "김철수"
      }
    }
  },
  "itemCount": "{count, plural, =0{항목 없음} =1{항목 1개} other{항목 {count}개}}",
  "@itemCount": {
    "description": "항목 수 표시",
    "placeholders": {
      "count": {
        "type": "int"
      }
    }
  }
}
```

`gen-l10n` 도구는 이 파일을 읽어 `AppLocalizations` 클래스를 자동 생성합니다. 개발자는 `AppLocalizations.of(context)!.helloUser(userName: '김철수')` 형태로 IDE 자동완성과 컴파일 타임 오류 검증을 모두 누릴 수 있습니다.

---

## 프로젝트 구조 설정

### 1. pubspec.yaml 의존성 추가

```yaml
dependencies:
  flutter:
    sdk: flutter
  flutter_localizations:
    sdk: flutter
  intl: ^0.20.3

flutter:
  generate: true  # gen-l10n 자동 실행 활성화
```

### 2. l10n.yaml 설정 파일

프로젝트 루트에 `l10n.yaml`을 생성하면 `gen-l10n`의 동작을 세밀하게 제어할 수 있습니다.

```yaml
arb-dir: lib/l10n
template-arb-file: app_ko.arb
output-localization-file: app_localizations.dart
output-class: AppLocalizations
synthetic-package: false          # Flutter 3.32+ 권장: 소스 디렉토리에 직접 생성
nullable-getter: false            # AppLocalizations.of(context)! 대신 non-null 반환
preferred-supported-locales:
  - ko
  - en
  - ja
  - ar
```

> **주의**: `synthetic-package: false`는 Flutter 3.32.0 이후 도입된 옵션입니다. 이를 사용하면 생성 코드가 `.dart_tool` 하위의 합성 패키지가 아닌 `lib/` 아래에 직접 생성되어 IDE 탐색과 Git 추적이 훨씬 편리해집니다.

### 3. ARB 파일 디렉토리

```
lib/
  l10n/
    app_ko.arb   ← 템플릿(기준 언어)
    app_en.arb
    app_ja.arb
    app_ar.arb
```

---

## 코드 예제 1: ICU 메시지 포맷 완전 활용

ICU(International Components for Unicode) 메시지 포맷은 복수형, 성별 분기, 중첩 치환 등을 하나의 문자열로 표현합니다.

### app_ko.arb

```json
{
  "@@locale": "ko",

  "welcomeTitle": "환영합니다",
  "@welcomeTitle": { "description": "앱 메인 화면 제목" },

  "newMessageAlert": "{count, plural, =0{새 메시지가 없습니다} =1{새 메시지 1건이 있습니다} other{새 메시지 {count}건이 있습니다}}",
  "@newMessageAlert": {
    "description": "새 메시지 수 알림",
    "placeholders": {
      "count": { "type": "int" }
    }
  },

  "userGreeting": "{gender, select, male{{name}님, 좋은 아침입니다} female{{name}님, 좋은 아침입니다} other{{name}님, 좋은 아침입니다}}",
  "@userGreeting": {
    "description": "성별에 따른 인사말 (한국어는 차이 없음, 영어 ARB에서 의미있음)",
    "placeholders": {
      "gender": { "type": "String" },
      "name": { "type": "String" }
    }
  },

  "lastSeen": "{name}님이 {date}에 마지막으로 접속했습니다",
  "@lastSeen": {
    "description": "마지막 접속 시간",
    "placeholders": {
      "name": { "type": "String" },
      "date": {
        "type": "DateTime",
        "format": "yMMMd",
        "isCustomDateFormat": false
      }
    }
  },

  "price": "가격: {amount}",
  "@price": {
    "placeholders": {
      "amount": {
        "type": "double",
        "format": "currency",
        "optionalParameters": {
          "symbol": "₩",
          "decimalDigits": 0
        }
      }
    }
  }
}
```

### app_en.arb

```json
{
  "@@locale": "en",
  "welcomeTitle": "Welcome",
  "newMessageAlert": "{count, plural, =0{No new messages} =1{1 new message} other{{count} new messages}}",
  "@newMessageAlert": {
    "placeholders": { "count": { "type": "int" } }
  },
  "userGreeting": "{gender, select, male{Good morning, Mr. {name}} female{Good morning, Ms. {name}} other{Good morning, {name}}}",
  "@userGreeting": {
    "placeholders": {
      "gender": { "type": "String" },
      "name": { "type": "String" }
    }
  },
  "lastSeen": "{name} was last seen on {date}",
  "@lastSeen": {
    "placeholders": {
      "name": { "type": "String" },
      "date": { "type": "DateTime", "format": "yMMMd" }
    }
  },
  "price": "Price: {amount}",
  "@price": {
    "placeholders": {
      "amount": {
        "type": "double",
        "format": "currency",
        "optionalParameters": { "symbol": "$", "decimalDigits": 2 }
      }
    }
  }
}
```

### Dart 사용 코드

```dart
import 'package:flutter/material.dart';
import 'package:flutter_gen/gen_l10n/app_localizations.dart';

class HomeScreen extends StatelessWidget {
  final int messageCount;
  final String userName;
  final String userGender; // 'male', 'female', 'other'
  final DateTime lastSeenAt;
  final double itemPrice;

  const HomeScreen({
    super.key,
    required this.messageCount,
    required this.userName,
    required this.userGender,
    required this.lastSeenAt,
    required this.itemPrice,
  });

  @override
  Widget build(BuildContext context) {
    final l10n = AppLocalizations.of(context);

    return Scaffold(
      appBar: AppBar(title: Text(l10n.welcomeTitle)),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            // ICU 복수형: 메시지 수에 따라 자동 선택
            Text(l10n.newMessageAlert(messageCount)),

            const SizedBox(height: 12),

            // ICU select: 성별 분기 (영어에서 Mr./Ms. 자동 선택)
            Text(l10n.userGreeting(userGender, userName)),

            const SizedBox(height: 12),

            // 날짜 자동 포맷: 한국어 → "2026년 8월 13일", 영어 → "Aug 13, 2026"
            Text(l10n.lastSeen(userName, lastSeenAt)),

            const SizedBox(height: 12),

            // 통화 자동 포맷: 한국어 → "₩39,000", 영어 → "$39.00"
            Text(l10n.price(itemPrice)),
          ],
        ),
      ),
    );
  }
}
```

---

## 코드 예제 2: 런타임 언어 전환

앱을 재시작하지 않고 언어를 바꾸려면 `MaterialApp`의 `locale`을 상태로 관리해야 합니다. Riverpod을 사용한 완전한 구현입니다.

```dart
import 'package:flutter/material.dart';
import 'package:flutter_localizations/flutter_localizations.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:shared_preferences/shared_preferences.dart';
import 'package:flutter_gen/gen_l10n/app_localizations.dart';

// 현재 로케일 상태 관리
final localeProvider = StateNotifierProvider<LocaleNotifier, Locale>((ref) {
  return LocaleNotifier();
});

class LocaleNotifier extends StateNotifier<Locale> {
  LocaleNotifier() : super(const Locale('ko')) {
    _loadSavedLocale();
  }

  static const _prefKey = 'app_locale';

  Future<void> _loadSavedLocale() async {
    final prefs = await SharedPreferences.getInstance();
    final code = prefs.getString(_prefKey);
    if (code != null) {
      state = Locale(code);
    }
  }

  Future<void> setLocale(Locale locale) async {
    state = locale;
    final prefs = await SharedPreferences.getInstance();
    await prefs.setString(_prefKey, locale.languageCode);
  }
}

// 지원 언어 목록
const supportedLocales = [
  Locale('ko'),
  Locale('en'),
  Locale('ja'),
  Locale('ar'),
];

class App extends ConsumerWidget {
  const App({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final locale = ref.watch(localeProvider);

    return MaterialApp(
      // 현재 선택된 로케일 적용
      locale: locale,

      // 지원 로케일 목록
      supportedLocales: supportedLocales,

      // 로컬라이제이션 대리자 (순서 중요)
      localizationsDelegates: const [
        AppLocalizations.delegate,             // 앱 자체 번역
        GlobalMaterialLocalizations.delegate,  // Material 위젯 번역
        GlobalWidgetsLocalizations.delegate,   // 위젯 기본값 (텍스트 방향)
        GlobalCupertinoLocalizations.delegate, // Cupertino 위젯 번역
      ],

      // 지원하지 않는 로케일 요청 시 처리
      localeResolutionCallback: (deviceLocale, supportedLocales) {
        if (deviceLocale == null) return supportedLocales.first;
        for (final locale in supportedLocales) {
          if (locale.languageCode == deviceLocale.languageCode) {
            return locale;
          }
        }
        return supportedLocales.first; // 기본값: 한국어
      },

      home: const HomeScreen(),
    );
  }
}

// 언어 선택 화면
class LanguageSettingsScreen extends ConsumerWidget {
  const LanguageSettingsScreen({super.key});

  static const _localeLabels = {
    'ko': '한국어',
    'en': 'English',
    'ja': '日本語',
    'ar': 'العربية',
  };

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final currentLocale = ref.watch(localeProvider);
    final notifier = ref.read(localeProvider.notifier);

    return Scaffold(
      appBar: AppBar(title: const Text('언어 설정')),
      body: ListView.builder(
        itemCount: supportedLocales.length,
        itemBuilder: (context, index) {
          final locale = supportedLocales[index];
          final isSelected = locale.languageCode == currentLocale.languageCode;

          return ListTile(
            title: Text(_localeLabels[locale.languageCode] ?? locale.languageCode),
            // RTL 언어(아랍어)는 trailing 아이콘도 자동으로 반전됨
            trailing: isSelected
                ? const Icon(Icons.check_circle, color: Colors.blue)
                : null,
            onTap: () => notifier.setLocale(locale),
          );
        },
      ),
    );
  }
}
```

### RTL 레이아웃 자동 지원

`GlobalWidgetsLocalizations.delegate`를 등록하면 아랍어(`ar`) 선택 시 `Directionality`가 자동으로 `rtl`로 설정됩니다. `Row`·`Padding`·`Align` 등 방향성 위젯들이 자동 반전되며, 명시적 좌/우 지정이 필요한 경우에는 `EdgeInsetsDirectional`과 `AlignmentDirectional`을 사용해야 합니다.

```dart
// ❌ 하드코딩된 방향 (RTL에서 의도와 반대로 적용됨)
Padding(padding: EdgeInsets.only(left: 16))

// ✅ 방향 무관한 패딩 (RTL에서 자동으로 오른쪽으로 전환)
Padding(padding: EdgeInsetsDirectional.only(start: 16))
```

---

## l10n.yaml 고급 옵션

| 옵션 | 설명 | 기본값 |
|---|---|---|
| `arb-dir` | ARB 파일 디렉토리 | `lib/l10n` |
| `template-arb-file` | 기준 ARB 파일명 | `app_en.arb` |
| `output-localization-file` | 생성 파일명 | `app_localizations.dart` |
| `synthetic-package` | 합성 패키지 vs 소스 파일 생성 | `true` |
| `nullable-getter` | `of()` 반환 타입 nullable 여부 | `true` |
| `use-escaping` | 중괄호 이스케이핑 활성화 | `false` |
| `required-resource-attributes` | 모든 키에 `@` 메타데이터 강제 | `false` |
| `untranslated-messages-file` | 미번역 메시지 출력 파일 | 없음 |

---

## 주의사항 및 실무 팁

### 1. 템플릿 ARB는 반드시 완전해야 한다

`gen-l10n`은 템플릿 ARB 파일을 기준으로 코드를 생성합니다. 다른 언어 ARB에 키가 없으면 **템플릿 언어 값이 폴백**으로 사용됩니다. 이는 편리하지만, 번역 누락을 알아차리기 어렵게 만듭니다. `untranslated-messages-file` 옵션으로 미번역 키를 파일에 기록해 CI에서 감지하세요.

```yaml
# l10n.yaml
untranslated-messages-file: untranslated.json
```

### 2. ICU 포맷 중괄호 이스케이핑

ICU 포맷에서 리터럴 `{`를 표시하고 싶다면 `use-escaping: true` 옵션과 함께 `'{'`처럼 작은따옴표로 감싸야 합니다.

```json
"literalBrace": "클릭하면 '{' 가 나타납니다"
```

### 3. 날짜 형식 커스터마이징

`intl` 패키지의 `DateFormat` 패턴을 ARB 플레이스홀더의 `format`에 지정합니다.

| 패턴 | 한국어 출력 | 영어 출력 |
|---|---|---|
| `yMMMd` | 2026년 8월 13일 | Aug 13, 2026 |
| `yMd` | 2026. 8. 13. | 8/13/2026 |
| `Hm` | 14:30 | 2:30 PM |
| `jms` | 오후 2:30:45 | 2:30:45 PM |

### 4. 코드 생성 자동화

`flutter.generate: true` 설정 시 `flutter pub get`이나 `flutter run` 실행 때 자동으로 코드가 생성됩니다. CI 환경에서는 명시적으로 실행하세요.

```bash
# CI에서 명시적 코드 생성
flutter gen-l10n

# 생성된 파일 확인 (synthetic-package: false 시)
ls lib/l10n/generated/
```

### 5. 아랍어·힌디어 복수형 주의

한국어와 영어는 복수형이 2종류지만, 아랍어는 CLDR 기준 6종류(`zero`, `one`, `two`, `few`, `many`, `other`)입니다. 아랍어를 지원한다면 ARB에 모든 케이스를 정의하세요.

```json
"itemCountAr": "{count, plural, =0{لا توجد عناصر} one{عنصر واحد} two{عنصران} few{{count} عناصر} many{{count} عنصرًا} other{{count} عنصر}}"
```

### 6. 테스트에서 로케일 주입

위젯 테스트에서는 `LocalizationsDelegate`를 직접 주입해 특정 언어로 테스트할 수 있습니다.

```dart
testWidgets('한국어 복수형 메시지 테스트', (tester) async {
  await tester.pumpWidget(
    MaterialApp(
      locale: const Locale('ko'),
      localizationsDelegates: AppLocalizations.localizationsDelegates,
      supportedLocales: AppLocalizations.supportedLocales,
      home: const HomeScreen(messageCount: 5, /* ... */),
    ),
  );

  expect(find.text('새 메시지 5건이 있습니다'), findsOneWidget);
});
```

---

## 마무리

Flutter의 `gen-l10n` + ARB + ICU 도구 체인은 단순한 문자열 치환을 넘어, 복수형·성별·날짜·통화·RTL 레이아웃까지 언어별 규칙을 **컴파일 타임 타입 안전성**과 함께 처리합니다. 핵심 정리:

1. **ARB 파일**에 ICU 메시지 포맷으로 플레이스홀더·복수형·성별 분기를 정의한다.
2. **`gen-l10n`**이 타입 안전한 `AppLocalizations` 클래스를 자동 생성한다.
3. **`localeProvider`(Riverpod)** 같은 상태 관리로 런타임 언어 전환을 구현한다.
4. **`EdgeInsetsDirectional`** 등 방향 무관 위젯으로 RTL을 지원한다.
5. **CI에서 미번역 키 감지**로 번역 누락 없이 글로벌 출시한다.

글로벌 앱 출시를 목표로 한다면, 기능 개발 초반부터 l10n 구조를 갖춰두는 것이 나중에 전체 코드베이스를 리팩터링하는 비용을 줄이는 가장 현명한 선택입니다.

---

## 참고 자료

- [intl 패키지 (pub.dev)](https://pub.dev/packages/intl)
- [intl 패키지 변경 이력 (pub.dev)](https://pub.dev/packages/intl/changelog)
- [Flutter Internationalization 공식 문서 (flutter.dev)](https://docs.flutter.dev/ui/internationalization)
- [Flutter gen-l10n synthetic-package 변경사항 (flutter.dev)](https://docs.flutter.dev/release/breaking-changes/flutter-generate-i10n-source)
