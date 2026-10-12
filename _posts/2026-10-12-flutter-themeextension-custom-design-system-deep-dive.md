---
layout: post
title: "Flutter ThemeExtension 심화: 커스텀 디자인 시스템으로 멀티브랜드 앱 완전 정복"
date: 2026-10-12
categories: [android, flutter]
tags: [flutter, theme, themeextension, design-system, material3, dart]
---

Flutter 앱이 커질수록 `ThemeData`만으로는 부족한 순간이 온다. 브랜드별로 다른 색상 팔레트, 앱마다 다른 그림자 강도, 컴포넌트별 고유한 간격 시스템… Material 3가 제공하는 `ColorScheme`과 `TextTheme`은 강력하지만, 실제 프로덕션 앱의 디자인 요구사항을 100% 커버하지는 못한다. 이 문제를 해결하기 위해 Flutter 3.0에서 공식 도입된 `ThemeExtension`은 개발자가 `ThemeData`를 원하는 방향으로 확장할 수 있는 메커니즘을 제공한다. 이 아티클에서는 `ThemeExtension`의 내부 동작 원리부터 멀티브랜드 디자인 시스템 구현까지 심화 내용을 다룬다.

---

## ThemeExtension이란?

`ThemeExtension<T>`는 Flutter의 `material` 패키지에 포함된 추상 클래스로, `ThemeData.extensions`에 커스텀 데이터를 담아 위젯 트리 어디서나 `Theme.of(context).extension<T>()`로 꺼낼 수 있게 해준다. 핵심 계약은 단 두 메서드다:

- **`copyWith(...)`**: 일부 필드만 변경한 새 인스턴스를 반환한다.
- **`lerp(ThemeExtension<T>? other, double t)`**: 두 확장값 사이를 `t`(0.0~1.0) 기준으로 보간한다. 이 메서드 덕분에 `ThemeData.lerp`가 호출될 때 커스텀 값도 부드럽게 전환된다.

```
abstract class ThemeExtension<T extends ThemeExtension<T>> {
  const ThemeExtension();
  T copyWith();
  T lerp(ThemeExtension<T>? other, double t);
}
```

Flutter 내부적으로 `AnimatedTheme` 위젯이 두 `ThemeData` 사이를 보간할 때 각 extension의 `lerp`를 호출한다. 즉, `lerp`를 올바르게 구현하지 않으면 다크 모드 전환이나 테마 애니메이션이 어색하게 끊기게 된다.

---

## 왜 ThemeExtension이 필요한가?

### Material ThemeData의 한계

`ThemeData`는 Material Design 사양에 맞는 값들만 보유한다. 실제 서비스 앱에서 자주 필요한 다음 항목들은 기본 제공되지 않는다:

| 필요한 값 | Material 3 기본 제공 여부 |
|---|---|
| 브랜드 전용 그라디언트 | ✗ |
| 커스텀 간격 토큰 (spacing-4, spacing-8 등) | ✗ |
| 컴포넌트별 고유 Border Radius | ✗ |
| 브랜드 전용 그림자(elevation shadow) | ✗ |
| 다크/라이트 외 3번째 테마 (예: 하이콘트라스트) | ✗ |

기존 해결책으로는 `InheritedWidget`을 직접 구현하거나 `BuildContext` 확장 함수로 전역 상수를 제공하는 방식이 있었다. 하지만 이 방식들은 `ThemeData.lerp`와 연동되지 않아 애니메이션 전환 시 값이 즉시 바뀌는 문제가 있었다.

`ThemeExtension`은 이 모든 문제를 해결한다. `ThemeData.extensions`에 등록하면 Material 테마 시스템과 완전히 통합되어 애니메이션, 다크 모드, 테마 오버라이드까지 자동으로 처리된다.

---

## 실제 구현 예제 1: 기본 커스텀 컬러 팔레트

아래는 브랜드 전용 색상과 그라디언트를 `ThemeExtension`으로 정의하는 예제다.

```dart
import 'package:flutter/material.dart';

@immutable
class AppColorExtension extends ThemeExtension<AppColorExtension> {
  const AppColorExtension({
    required this.brandPrimary,
    required this.brandSecondary,
    required this.success,
    required this.warning,
    required this.brandGradient,
  });

  final Color brandPrimary;
  final Color brandSecondary;
  final Color success;
  final Color warning;
  final LinearGradient brandGradient;

  // 라이트 모드 기본값
  static const light = AppColorExtension(
    brandPrimary: Color(0xFF0057FF),
    brandSecondary: Color(0xFF00C2FF),
    success: Color(0xFF00C853),
    warning: Color(0xFFFFAB00),
    brandGradient: LinearGradient(
      colors: [Color(0xFF0057FF), Color(0xFF00C2FF)],
      begin: Alignment.centerLeft,
      end: Alignment.centerRight,
    ),
  );

  // 다크 모드 기본값
  static const dark = AppColorExtension(
    brandPrimary: Color(0xFF4D8FFF),
    brandSecondary: Color(0xFF40D4FF),
    success: Color(0xFF69F0AE),
    warning: Color(0xFFFFD740),
    brandGradient: LinearGradient(
      colors: [Color(0xFF4D8FFF), Color(0xFF40D4FF)],
      begin: Alignment.centerLeft,
      end: Alignment.centerRight,
    ),
  );

  @override
  AppColorExtension copyWith({
    Color? brandPrimary,
    Color? brandSecondary,
    Color? success,
    Color? warning,
    LinearGradient? brandGradient,
  }) {
    return AppColorExtension(
      brandPrimary: brandPrimary ?? this.brandPrimary,
      brandSecondary: brandSecondary ?? this.brandSecondary,
      success: success ?? this.success,
      warning: warning ?? this.warning,
      brandGradient: brandGradient ?? this.brandGradient,
    );
  }

  @override
  AppColorExtension lerp(ThemeExtension<AppColorExtension>? other, double t) {
    if (other is! AppColorExtension) return this;
    return AppColorExtension(
      brandPrimary: Color.lerp(brandPrimary, other.brandPrimary, t)!,
      brandSecondary: Color.lerp(brandSecondary, other.brandSecondary, t)!,
      success: Color.lerp(success, other.success, t)!,
      warning: Color.lerp(warning, other.warning, t)!,
      // Gradient는 직접 보간 불필요 시 t < 0.5 기준으로 전환
      brandGradient: t < 0.5 ? brandGradient : other.brandGradient,
    );
  }
}
```

이 `AppColorExtension`을 `MaterialApp`에 등록하고 사용하는 방법은 다음과 같다:

```dart
class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      theme: ThemeData.light(useMaterial3: true).copyWith(
        extensions: const [AppColorExtension.light],
      ),
      darkTheme: ThemeData.dark(useMaterial3: true).copyWith(
        extensions: const [AppColorExtension.dark],
      ),
      home: const HomePage(),
    );
  }
}

// 위젯에서 사용
class BrandButton extends StatelessWidget {
  const BrandButton({super.key, required this.label});

  final String label;

  @override
  Widget build(BuildContext context) {
    final appColors = Theme.of(context).extension<AppColorExtension>()!;

    return DecoratedBox(
      decoration: BoxDecoration(
        gradient: appColors.brandGradient,
        borderRadius: BorderRadius.circular(12),
      ),
      child: Padding(
        padding: const EdgeInsets.symmetric(horizontal: 24, vertical: 12),
        child: Text(
          label,
          style: const TextStyle(color: Colors.white, fontWeight: FontWeight.bold),
        ),
      ),
    );
  }
}
```

---

## 실제 구현 예제 2: 멀티브랜드 디자인 시스템

화이트라벨 앱이나 OEM 배포 시나리오에서는 같은 코드베이스로 여러 브랜드를 지원해야 한다. `ThemeExtension`과 `AppThemeProvider` 패턴을 조합하면 런타임에 브랜드를 전환하는 시스템을 깔끔하게 구현할 수 있다.

```dart
// 브랜드 정의 열거형
enum AppBrand { alpha, beta, gamma }

// 간격 토큰 확장
@immutable
class SpacingExtension extends ThemeExtension<SpacingExtension> {
  const SpacingExtension({
    required this.xs,
    required this.sm,
    required this.md,
    required this.lg,
    required this.xl,
    required this.cardRadius,
    required this.buttonRadius,
  });

  final double xs;   // 4
  final double sm;   // 8
  final double md;   // 16
  final double lg;   // 24
  final double xl;   // 32
  final double cardRadius;
  final double buttonRadius;

  @override
  SpacingExtension copyWith({
    double? xs, double? sm, double? md, double? lg, double? xl,
    double? cardRadius, double? buttonRadius,
  }) {
    return SpacingExtension(
      xs: xs ?? this.xs,
      sm: sm ?? this.sm,
      md: md ?? this.md,
      lg: lg ?? this.lg,
      xl: xl ?? this.xl,
      cardRadius: cardRadius ?? this.cardRadius,
      buttonRadius: buttonRadius ?? this.buttonRadius,
    );
  }

  @override
  SpacingExtension lerp(ThemeExtension<SpacingExtension>? other, double t) {
    if (other is! SpacingExtension) return this;
    // lerpDouble 사용으로 부드러운 보간
    return SpacingExtension(
      xs: lerpDouble(xs, other.xs, t)!,
      sm: lerpDouble(sm, other.sm, t)!,
      md: lerpDouble(md, other.md, t)!,
      lg: lerpDouble(lg, other.lg, t)!,
      xl: lerpDouble(xl, other.xl, t)!,
      cardRadius: lerpDouble(cardRadius, other.cardRadius, t)!,
      buttonRadius: lerpDouble(buttonRadius, other.buttonRadius, t)!,
    );
  }
}

// 브랜드별 테마 팩토리
class BrandTheme {
  static ThemeData forBrand(AppBrand brand, {bool dark = false}) {
    final base = dark
        ? ThemeData.dark(useMaterial3: true)
        : ThemeData.light(useMaterial3: true);

    final colorExt = _colorExtension(brand, dark);
    final spacingExt = _spacingExtension(brand);

    return base.copyWith(
      colorScheme: _colorScheme(brand, dark),
      extensions: [colorExt, spacingExt],
    );
  }

  static AppColorExtension _colorExtension(AppBrand brand, bool dark) {
    switch (brand) {
      case AppBrand.alpha:
        return dark ? AppColorExtension.dark : AppColorExtension.light;
      case AppBrand.beta:
        return dark
            ? const AppColorExtension(
                brandPrimary: Color(0xFFFF5252),
                brandSecondary: Color(0xFFFF8A80),
                success: Color(0xFF69F0AE),
                warning: Color(0xFFFFD740),
                brandGradient: LinearGradient(
                  colors: [Color(0xFFFF5252), Color(0xFFFF8A80)],
                ),
              )
            : const AppColorExtension(
                brandPrimary: Color(0xFFD50000),
                brandSecondary: Color(0xFFFF5252),
                success: Color(0xFF00C853),
                warning: Color(0xFFFFAB00),
                brandGradient: LinearGradient(
                  colors: [Color(0xFFD50000), Color(0xFFFF5252)],
                ),
              );
      case AppBrand.gamma:
        return dark
            ? const AppColorExtension(
                brandPrimary: Color(0xFF69F0AE),
                brandSecondary: Color(0xFFB9F6CA),
                success: Color(0xFF69F0AE),
                warning: Color(0xFFFFD740),
                brandGradient: LinearGradient(
                  colors: [Color(0xFF00BFA5), Color(0xFF69F0AE)],
                ),
              )
            : const AppColorExtension(
                brandPrimary: Color(0xFF00897B),
                brandSecondary: Color(0xFF00BFA5),
                success: Color(0xFF00C853),
                warning: Color(0xFFFFAB00),
                brandGradient: LinearGradient(
                  colors: [Color(0xFF00897B), Color(0xFF00BFA5)],
                ),
              );
    }
  }

  static SpacingExtension _spacingExtension(AppBrand brand) {
    // 브랜드마다 다른 간격/반경 토큰
    switch (brand) {
      case AppBrand.alpha:
        return const SpacingExtension(
          xs: 4, sm: 8, md: 16, lg: 24, xl: 32,
          cardRadius: 12, buttonRadius: 8,
        );
      case AppBrand.beta:
        return const SpacingExtension(
          xs: 4, sm: 8, md: 16, lg: 24, xl: 32,
          cardRadius: 4, buttonRadius: 2, // 더 각진 디자인
        );
      case AppBrand.gamma:
        return const SpacingExtension(
          xs: 4, sm: 8, md: 16, lg: 24, xl: 32,
          cardRadius: 24, buttonRadius: 24, // 더 둥근 디자인
        );
    }
  }

  static ColorScheme _colorScheme(AppBrand brand, bool dark) {
    // 브랜드별 colorScheme 구성 (생략)
    return dark
        ? const ColorScheme.dark()
        : const ColorScheme.light();
  }
}

// BuildContext 편의 확장
extension ThemeExtensionContext on BuildContext {
  AppColorExtension get appColors =>
      Theme.of(this).extension<AppColorExtension>()!;

  SpacingExtension get spacing =>
      Theme.of(this).extension<SpacingExtension>()!;
}

// 사용 예시
class BrandCard extends StatelessWidget {
  const BrandCard({super.key, required this.title, required this.subtitle});

  final String title;
  final String subtitle;

  @override
  Widget build(BuildContext context) {
    final spacing = context.spacing;
    final colors = context.appColors;

    return Container(
      padding: EdgeInsets.all(spacing.md),
      decoration: BoxDecoration(
        borderRadius: BorderRadius.circular(spacing.cardRadius),
        color: Theme.of(context).colorScheme.surface,
        boxShadow: [
          BoxShadow(
            color: colors.brandPrimary.withOpacity(0.12),
            blurRadius: spacing.sm,
            offset: const Offset(0, 4),
          ),
        ],
      ),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Text(
            title,
            style: Theme.of(context).textTheme.titleMedium?.copyWith(
              color: colors.brandPrimary,
            ),
          ),
          SizedBox(height: spacing.xs),
          Text(subtitle),
        ],
      ),
    );
  }
}
```

---

## 주의사항 및 팁

### 1. `lerp`에서 null 처리를 명확히 하라

`lerp`의 `other` 파라미터는 nullable이다. `other is! T`인 경우 `this`를 반환하는 것이 안전하다. `Color.lerp`는 null을 허용하지만, 반환값도 nullable이므로 `!`로 강제 언래핑하거나 `??` 폴백을 꼭 처리해야 한다.

```dart
// 잘못된 예 - NullPointerException 위험
Color.lerp(brandPrimary, other.brandPrimary, t); // Color? 반환

// 올바른 예
Color.lerp(brandPrimary, other.brandPrimary, t) ?? brandPrimary;
```

### 2. `extension<T>()`는 null을 반환할 수 있다

`ThemeData.extensions`에 등록하지 않으면 `null`이 반환된다. 프로덕션 코드에서는 `!` 강제 언래핑보다 null-safe 처리 또는 테스트에서 항상 확장을 포함하도록 보장하는 게 낫다.

```dart
// 개발 중 빠른 실패를 원할 때
final colors = Theme.of(context).extension<AppColorExtension>()!;

// 방어적 코드가 필요할 때
final colors = Theme.of(context).extension<AppColorExtension>()
    ?? AppColorExtension.light;
```

### 3. `const` 생성자를 활용하라

ThemeExtension 인스턴스는 불변(immutable)이어야 하며, `@immutable` 어노테이션을 붙이는 것이 권장된다. 값이 컴파일 타임에 결정된다면 `const`로 선언해 성능을 최적화할 수 있다.

### 4. 타입 계층을 납작하게 유지하라

하나의 거대한 `ThemeExtension`보다 책임별로 분리된 여러 확장이 낫다. `AppColorExtension`, `SpacingExtension`, `TypographyExtension` 식으로 나누면 테스트와 오버라이드가 쉬워진다.

### 5. 위젯 테스트에서 확장을 주입하라

```dart
testWidgets('BrandCard uses brand colors', (tester) async {
  await tester.pumpWidget(
    MaterialApp(
      theme: ThemeData.light().copyWith(
        extensions: const [
          AppColorExtension.light,
          SpacingExtension(
            xs: 4, sm: 8, md: 16, lg: 24, xl: 32,
            cardRadius: 12, buttonRadius: 8,
          ),
        ],
      ),
      home: const BrandCard(title: 'Test', subtitle: 'Subtitle'),
    ),
  );
  expect(find.text('Test'), findsOneWidget);
});
```

### 6. `AnimatedTheme`을 활용한 부드러운 브랜드 전환

`MaterialApp`의 `theme`을 교체할 때 Flutter는 내부적으로 `AnimatedTheme`을 통해 보간한다. `lerp`를 올바르게 구현해 두면 브랜드 전환 시 색상과 간격이 자연스럽게 애니메이션된다. 전환 속도는 `ThemeAnimationDuration` 파라미터로 조정할 수 있다.

---

## 정리

`ThemeExtension`은 Flutter의 테마 시스템에서 빠져 있던 퍼즐 조각이다. Material 3의 `ColorScheme`과 `TextTheme`을 유지하면서, 앱 고유의 디자인 토큰(간격, 그라디언트, 컴포넌트별 반경)을 동일한 인프라 위에 올릴 수 있다. 특히 멀티브랜드 화이트라벨 앱이나 디자인 시스템 라이브러리를 만들 때 `ThemeExtension`은 사실상 필수 도구다. `lerp`와 `copyWith`를 제대로 구현하면 다크 모드 전환, `AnimatedTheme`, 테마 오버라이드가 모두 무료로 따라온다.

## 참고 자료
- [Flutter ThemeExtension API 레퍼런스](https://api.flutter.dev/flutter/material/ThemeExtension-class.html)
- [design_tokens_builder: Figma 토큰을 ThemeExtension으로 자동 생성](https://pub.dev/packages/design_tokens_builder)
- [design_builder: W3C Design Tokens JSON에서 ThemeExtension 코드 생성](https://pub.dev/packages/design_builder)
