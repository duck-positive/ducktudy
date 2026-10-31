---
layout: post
title: "Flutter 국제화(i18n) 심화: gen-l10n·ARB 파일·복수형 처리·RTL 레이아웃 완전 정복"
date: 2026-10-31
categories: [android, flutter]
tags: [flutter, i18n, l10n, gen-l10n, arb, intl, localization, rtl, plurals, dart]
---

Flutter로 앱을 개발하다 보면 반드시 마주치는 과제가 하나 있습니다. 바로 **국제화(Internationalization, i18n)** 입니다. 단순히 문자열을 번역 파일에 빼두는 수준에서 끝나지 않습니다. 복수형 처리, 날짜·통화 형식, 아랍어·히브리어 같은 RTL(Right-to-Left) 레이아웃, 그리고 로케일에 따라 동적으로 바뀌는 복잡한 문장 구조까지 다뤄야 합니다. 이 글에서는 Flutter 공식 도구인 `gen-l10n`을 기반으로 실제 프로덕션 수준의 국제화 전략을 심도 있게 살펴봅니다.

## 1. 왜 국제화가 어려운가?

많은 개발자들이 처음에는 단순히 `Map<String, String>`을 만들어 키로 문자열을 조회하는 방식으로 시작합니다. 하지만 이 방법은 금방 한계에 부딪힙니다.

**문제 1 — 복수형(Plurals):** 한국어는 "1개", "2개" 모두 동일한 조사를 쓰지만, 영어는 "1 item"과 "2 items"처럼 단수·복수가 다릅니다. 아랍어는 무려 6가지 복수 형태를 가집니다. 단순 Map으로는 이를 표현할 수 없습니다.

**문제 2 — 성(Gender):** 프랑스어나 스페인어처럼 명사에 성이 있는 언어는 수식어도 성에 따라 달라집니다.

**문제 3 — RTL 레이아웃:** 아랍어, 히브리어, 페르시아어 사용자는 화면 전체가 오른쪽에서 왼쪽으로 읽혀야 합니다. 단순히 텍스트만 뒤집는 게 아니라 패딩, 아이콘 위치, 스크롤 방향까지 모두 미러링되어야 합니다.

**문제 4 — 날짜·숫자·통화 형식:** "2026년 10월 31일"(한국), "October 31, 2026"(미국), "31/10/2026"(영국)처럼 같은 날짜도 로케일마다 표현 방식이 다릅니다.

Flutter의 `gen-l10n` 도구와 `intl` 패키지는 이 모든 복잡성을 우아하게 처리합니다.

## 2. gen-l10n 기반 국제화 설정

### 2.1 의존성 추가

`pubspec.yaml`에 다음을 추가합니다.

```yaml
dependencies:
  flutter:
    sdk: flutter
  flutter_localizations:
    sdk: flutter
  intl: ^0.20.0

flutter:
  generate: true
```

`flutter_localizations`는 Flutter SDK에 내장된 패키지로, 78개 이상의 언어에 대한 날짜 피커, 방향 설정, 위젯 레이블 등을 제공합니다. `generate: true`는 `gen-l10n` 자동 실행을 활성화합니다.

### 2.2 l10n.yaml 설정 파일

프로젝트 루트에 `l10n.yaml`을 만듭니다.

```yaml
arb-dir: lib/l10n
template-arb-file: app_ko.arb
output-localization-file: app_localizations.dart
output-class: AppLocalizations
nullable-getter: false
use-escaping: true
synthetic-package: false
```

- `arb-dir`: ARB 파일들이 위치한 디렉토리
- `template-arb-file`: 기준 언어(보통 개발 언어) ARB 파일
- `synthetic-package`: `false`로 설정하면 `lib/` 하위에 직접 파일을 생성하여 IDE 코드 인식이 쉬워집니다
- `nullable-getter`: `false`로 설정하면 `AppLocalizations.of(context)!` 대신 `AppLocalizations.of(context)` 직접 사용 가능

### 2.3 ARB 파일 구조

ARB(Application Resource Bundle)는 JSON 기반의 메시지 파일입니다. `lib/l10n/app_ko.arb`를 작성합니다.

```json
{
  "@@locale": "ko",

  "appTitle": "덕스터디",
  "@appTitle": {
    "description": "앱의 기본 타이틀"
  },

  "welcomeMessage": "{name}님, 환영합니다!",
  "@welcomeMessage": {
    "description": "사용자 이름을 포함한 환영 메시지",
    "placeholders": {
      "name": {
        "type": "String",
        "example": "홍길동"
      }
    }
  },

  "itemCount": "{count, plural, =0{항목 없음} =1{항목 1개} other{{count}개 항목}}",
  "@itemCount": {
    "description": "항목 수에 따른 복수형 메시지",
    "placeholders": {
      "count": {
        "type": "int",
        "format": "decimalPattern"
      }
    }
  },

  "lastUpdated": "마지막 업데이트: {date}",
  "@lastUpdated": {
    "description": "마지막 업데이트 날짜",
    "placeholders": {
      "date": {
        "type": "DateTime",
        "format": "yMMMd",
        "isCustomDateFormat": "false"
      }
    }
  },

  "price": "가격: {amount}",
  "@price": {
    "description": "가격 표시",
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

영어 버전 `lib/l10n/app_en.arb`:

```json
{
  "@@locale": "en",

  "appTitle": "Ducktudy",

  "welcomeMessage": "Welcome, {name}!",

  "itemCount": "{count, plural, =0{No items} =1{1 item} other{{count} items}}",

  "lastUpdated": "Last updated: {date}",

  "price": "Price: {amount}",
  "@price": {
    "placeholders": {
      "amount": {
        "type": "double",
        "format": "currency",
        "optionalParameters": {
          "symbol": "$",
          "decimalDigits": 2
        }
      }
    }
  }
}
```

## 3. 실제 구현 예제 1 — 기본 국제화 적용

### 3.1 MaterialApp에 로케일 설정

```dart
import 'package:flutter/material.dart';
import 'package:flutter_localizations/flutter_localizations.dart';
import 'l10n/app_localizations.dart';

class MyApp extends StatefulWidget {
  const MyApp({super.key});

  @override
  State<MyApp> createState() => _MyAppState();
}

class _MyAppState extends State<MyApp> {
  Locale _locale = const Locale('ko');

  void _changeLocale(Locale locale) {
    setState(() => _locale = locale);
  }

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      // 지원할 로케일 목록
      supportedLocales: AppLocalizations.supportedLocales,
      // 로케일 결정 로직
      localeResolutionCallback: (deviceLocale, supportedLocales) {
        // 기기 언어가 지원 목록에 있으면 사용, 없으면 한국어로 폴백
        if (deviceLocale != null) {
          for (final supported in supportedLocales) {
            if (supported.languageCode == deviceLocale.languageCode) {
              return supported;
            }
          }
        }
        return const Locale('ko');
      },
      locale: _locale,
      localizationsDelegates: const [
        AppLocalizations.delegate,
        GlobalMaterialLocalizations.delegate,
        GlobalWidgetsLocalizations.delegate,
        GlobalCupertinoLocalizations.delegate,
      ],
      home: HomeScreen(onLocaleChange: _changeLocale),
    );
  }
}

class HomeScreen extends StatelessWidget {
  final void Function(Locale) onLocaleChange;

  const HomeScreen({super.key, required this.onLocaleChange});

  @override
  Widget build(BuildContext context) {
    final l10n = AppLocalizations.of(context);
    final now = DateTime.now();

    return Scaffold(
      appBar: AppBar(title: Text(l10n.appTitle)),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            // 플레이스홀더 치환
            Text(l10n.welcomeMessage('홍길동')),
            const SizedBox(height: 8),

            // 복수형 처리
            Text(l10n.itemCount(0)),   // "항목 없음"
            Text(l10n.itemCount(1)),   // "항목 1개"
            Text(l10n.itemCount(42)),  // "42개 항목"
            const SizedBox(height: 8),

            // 날짜 형식 (로케일에 따라 자동 변환)
            Text(l10n.lastUpdated(now)),
            const SizedBox(height: 8),

            // 통화 형식
            Text(l10n.price(29900)),
            const SizedBox(height: 24),

            // 로케일 전환 버튼
            Row(
              children: [
                ElevatedButton(
                  onPressed: () => onLocaleChange(const Locale('ko')),
                  child: const Text('한국어'),
                ),
                const SizedBox(width: 8),
                ElevatedButton(
                  onPressed: () => onLocaleChange(const Locale('en')),
                  child: const Text('English'),
                ),
                const SizedBox(width: 8),
                ElevatedButton(
                  onPressed: () => onLocaleChange(const Locale('ar')),
                  child: const Text('العربية'),
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

`flutter gen-l10n` 명령 실행 후 `lib/l10n/app_localizations.dart`와 각 언어별 구현 파일(`app_localizations_ko.dart`, `app_localizations_en.dart`)이 자동 생성됩니다. 생성된 코드는 `intl` 패키지의 `Intl.message`, `Intl.plural`을 활용하여 ICU 메시지 포맷을 런타임에 해석합니다.

## 4. 실제 구현 예제 2 — RTL 레이아웃과 방향 인식 위젯

RTL 지원은 단순히 아랍어 문자열을 넣는 것 이상입니다. Flutter는 `Directionality` 위젯을 통해 위젯 트리 전체의 텍스트 방향을 제어합니다.

```dart
import 'package:flutter/material.dart';

// RTL을 고려한 아이템 카드 위젯
class DirectionAwareCard extends StatelessWidget {
  final String title;
  final String subtitle;
  final IconData leadingIcon;
  final VoidCallback? onTap;

  const DirectionAwareCard({
    super.key,
    required this.title,
    required this.subtitle,
    required this.leadingIcon,
    this.onTap,
  });

  @override
  Widget build(BuildContext context) {
    // 현재 텍스트 방향 감지
    final isRtl = Directionality.of(context) == TextDirection.rtl;

    return Card(
      child: InkWell(
        onTap: onTap,
        child: Padding(
          padding: const EdgeInsets.all(16),
          child: Row(
            children: [
              // EdgeInsetsDirectional: 논리적 방향(start/end)을 사용
              // RTL에서는 start가 오른쪽, end가 왼쪽이 됨
              Padding(
                padding: const EdgeInsetsDirectional.only(end: 16),
                child: Icon(leadingIcon, size: 32),
              ),
              Expanded(
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Text(
                      title,
                      style: Theme.of(context).textTheme.titleMedium,
                    ),
                    Text(
                      subtitle,
                      style: Theme.of(context).textTheme.bodySmall,
                    ),
                  ],
                ),
              ),
              // RTL에서는 뒤로 가기 화살표 방향도 바뀌어야 함
              Icon(
                isRtl ? Icons.arrow_back_ios : Icons.arrow_forward_ios,
                size: 16,
                color: Colors.grey,
              ),
            ],
          ),
        ),
      ),
    );
  }
}

// RTL 대응 커스텀 패딩 위젯 예시
class DirectionAwareListTile extends StatelessWidget {
  final Widget leading;
  final Widget title;
  final Widget? trailing;

  const DirectionAwareListTile({
    super.key,
    required this.leading,
    required this.title,
    this.trailing,
  });

  @override
  Widget build(BuildContext context) {
    return Padding(
      // EdgeInsetsDirectional로 논리적 방향 패딩 적용
      padding: const EdgeInsetsDirectional.fromSTEB(16, 8, 16, 8),
      child: Row(
        textDirection: Directionality.of(context),
        children: [
          leading,
          const SizedBox(width: 12),
          Expanded(child: title),
          if (trailing != null) ...[
            const SizedBox(width: 12),
            trailing!,
          ],
        ],
      ),
    );
  }
}

// 아랍어 로케일 테스트 화면
class RtlDemoScreen extends StatelessWidget {
  const RtlDemoScreen({super.key});

  @override
  Widget build(BuildContext context) {
    final textDirection = Directionality.of(context);

    return Scaffold(
      appBar: AppBar(
        title: const Text('RTL Demo'),
        // 앱바 액션도 RTL에서 자동으로 위치 전환됨
        actions: [
          IconButton(
            icon: const Icon(Icons.settings),
            onPressed: () {},
          ),
        ],
      ),
      body: ListView(
        padding: const EdgeInsets.all(16),
        children: [
          // 방향 정보 표시
          Text('현재 방향: ${textDirection.name.toUpperCase()}'),
          const SizedBox(height: 16),

          // RTL 대응 카드들
          DirectionAwareCard(
            title: 'مرحبا بك',        // 아랍어: "환영합니다"
            subtitle: 'تعلم Flutter', // 아랍어: "Flutter 배우기"
            leadingIcon: Icons.school,
            onTap: () {},
          ),
          const SizedBox(height: 8),

          DirectionAwareCard(
            title: '한국어 제목',
            subtitle: '한국어 부제목',
            leadingIcon: Icons.book,
            onTap: () {},
          ),

          const SizedBox(height: 16),

          // BidiText: 혼합 방향 텍스트 처리
          // 아랍어와 영어가 섞인 텍스트는 Unicode Bidi Algorithm이 자동 처리
          const SelectableText(
            'Flutter يدعم النص ثنائي الاتجاه بشكل كامل.',
            textDirection: TextDirection.rtl,
          ),

          const SizedBox(height: 16),

          // AlignmentDirectional 사용 예시
          Container(
            alignment: AlignmentDirectional.centerStart,
            padding: const EdgeInsetsDirectional.only(start: 16),
            color: Colors.blue.shade50,
            height: 48,
            child: const Text('논리적 시작 정렬 (Start Alignment)'),
          ),
        ],
      ),
      // FloatingActionButton 위치도 RTL에서 자동 전환
      floatingActionButton: FloatingActionButton(
        onPressed: () {},
        child: const Icon(Icons.add),
      ),
      floatingActionButtonLocation: FloatingActionButtonLocation.endFloat,
    );
  }
}
```

### 4.1 RTL 대응 핵심 원칙

Flutter에서 RTL을 올바르게 지원하려면 다음 원칙을 따릅니다.

| LTR 전용 API | RTL 대응 API |
|---|---|
| `EdgeInsets.only(left: 16)` | `EdgeInsetsDirectional.only(start: 16)` |
| `Alignment.centerLeft` | `AlignmentDirectional.centerStart` |
| `Icons.arrow_forward` | `Icons.arrow_forward` (자동 미러링) |
| `TextAlign.left` | `TextAlign.start` |
| `CrossAxisAlignment.start` | `CrossAxisAlignment.start` (이미 논리적) |

Flutter는 `MaterialApp`에 `locale: Locale('ar')`을 설정하면 `Directionality` 위젯이 자동으로 RTL로 설정되고, `EdgeInsetsDirectional`, `AlignmentDirectional` 등 논리적 방향 API들이 자동으로 미러링됩니다.

## 5. ARB 고급 기능 — 복수형·성·선택형

ICU 메시지 포맷은 단순 복수형 이상의 복잡한 표현을 지원합니다.

```json
{
  "@@locale": "ar",

  "itemCount": "{count, plural, =0{لا توجد عناصر} =1{عنصر واحد} =2{عنصران} few{{count} عناصر} many{{count} عنصرًا} other{{count} عنصر}}",
  "@itemCount": {
    "description": "아랍어 복수형 (6가지 형태)",
    "placeholders": {
      "count": { "type": "int" }
    }
  },

  "genderMessage": "{gender, select, male{هو طالب} female{هي طالبة} other{هم طلاب}}",
  "@genderMessage": {
    "description": "성별에 따른 메시지",
    "placeholders": {
      "gender": { "type": "String" }
    }
  }
}
```

```dart
// 복수형과 성별 메시지 사용
class PluralGenderDemo extends StatelessWidget {
  const PluralGenderDemo({super.key});

  @override
  Widget build(BuildContext context) {
    final l10n = AppLocalizations.of(context);

    return Column(
      children: [
        // 복수형 자동 처리 (한국어, 영어, 아랍어 모두 다른 형태)
        for (final count in [0, 1, 2, 5, 100])
          Text('$count → ${l10n.itemCount(count)}'),

        const Divider(),

        // 성별 선택형 (아랍어에서만 의미 있음)
        Text(l10n.genderMessage('male')),
        Text(l10n.genderMessage('female')),
        Text(l10n.genderMessage('other')),
      ],
    );
  }
}
```

## 6. 로케일 상태 관리와 영속성

앱 재시작 후에도 사용자의 언어 설정이 유지되어야 합니다. `shared_preferences`와 결합한 패턴입니다.

```dart
import 'package:flutter/material.dart';
import 'package:shared_preferences/shared_preferences.dart';

class LocaleNotifier extends ChangeNotifier {
  static const _key = 'selected_locale';
  Locale _locale = const Locale('ko');

  Locale get locale => _locale;

  LocaleNotifier() {
    _loadLocale();
  }

  Future<void> _loadLocale() async {
    final prefs = await SharedPreferences.getInstance();
    final code = prefs.getString(_key);
    if (code != null) {
      _locale = Locale(code);
      notifyListeners();
    }
  }

  Future<void> setLocale(Locale locale) async {
    if (_locale == locale) return;
    _locale = locale;
    final prefs = await SharedPreferences.getInstance();
    await prefs.setString(_key, locale.languageCode);
    notifyListeners();
  }
}
```

## 7. 주의사항과 실전 팁

### 팁 1 — ARB 키 네이밍 컨벤션

ARB 키는 `lowerCamelCase`를 사용하고, 기능 단위로 접두사를 붙이면 관리가 쉬워집니다.

```
homeTitle, homeSubtitle       ← 홈 화면
profileEditButton             ← 프로필 화면
errorNetworkTimeout           ← 에러 메시지
```

### 팁 2 — 번역 누락 감지

`l10n.yaml`에 `untranslated-messages-file: missing_translations.txt`를 추가하면 번역이 빠진 키 목록을 파일로 출력합니다. CI 파이프라인에서 이 파일의 존재 여부를 체크하여 누락된 번역을 배포 전에 잡을 수 있습니다.

### 팁 3 — 긴 번역 텍스트 대비 레이아웃

독일어와 같이 영어보다 훨씬 긴 단어를 사용하는 언어("Donaudampfschifffahrtsgesellschaft")에 대비하여, 텍스트 위젯에는 항상 `overflow: TextOverflow.ellipsis` 또는 `softWrap: true`를 설정하고, 버튼 텍스트는 `FittedBox`로 감싸두는 것이 안전합니다.

```dart
// 긴 번역에 안전한 버튼
ElevatedButton(
  onPressed: onPressed,
  child: FittedBox(
    child: Text(l10n.someButtonLabel),
  ),
)
```

### 팁 4 — 숫자 형식 직접 제어

ARB의 자동 숫자 포맷이 맞지 않을 때는 `intl` 패키지를 직접 사용합니다.

```dart
import 'package:intl/intl.dart';

// 현재 로케일로 숫자 포맷
final formatted = NumberFormat.compact(
  locale: Localizations.localeOf(context).toString(),
).format(1234567);
// ko: "123만", en: "1.2M"

// 날짜 포맷 직접 제어
final dateStr = DateFormat('yyyy년 MM월 dd일 (E)', 'ko').format(DateTime.now());
```

### 팁 5 — Pseudolocalization으로 레이아웃 테스트

번역 파일 없이도 레이아웃이 다국어 대응이 되었는지 확인할 수 있습니다. `l10n.yaml`에 `pseudo-locale: en_XA`를 추가하면 영어 텍스트에 악센트를 붙이고 약 30% 길게 늘려주어, 긴 번역 텍스트에서의 레이아웃 깨짐을 조기에 발견할 수 있습니다.

## 마치며

Flutter의 `gen-l10n` 기반 국제화는 설정이 다소 복잡하게 느껴질 수 있지만, 한번 제대로 구축해두면 타입 안전한 코드로 번역 키 오타를 컴파일 타임에 잡고, 복수형·날짜·통화 같은 복잡한 로케일 처리를 인프라 수준에서 해결해줍니다. `EdgeInsetsDirectional`과 `AlignmentDirectional` 같은 논리적 방향 API를 습관화하면 추가 작업 없이 아랍어·히브리어 시장에도 대응할 수 있습니다. 글로벌 앱을 목표로 한다면 i18n은 마지막에 붙이는 기능이 아니라, 설계 단계부터 내재화해야 할 아키텍처입니다.

## 참고 자료

- [intl 패키지 (pub.dev)](https://pub.dev/packages/intl)
- [intl 패키지 API 문서](https://pub.dev/documentation/intl/latest/)
