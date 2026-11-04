---
layout: post
title: "Material You & Material 3 심화: Dynamic Color·토널 팔레트·색상 조화로 Jetpack Compose & Flutter 앱 테마 완전 정복"
date: 2026-11-04
categories: [android, flutter]
tags: [material3, material-you, dynamic-color, jetpack-compose, flutter, theming, colorscheme, tonal-palette, android12]
---

안드로이드 12(API 31)와 함께 등장한 **Material You**는 단순한 디자인 업데이트가 아닙니다. 사용자의 배경화면에서 색상을 추출해 앱 전체에 개인화된 팔레트를 적용하는 _운영체제 수준의 테마 엔진_ 입니다. 이 글에서는 Material 3의 색상 시스템 내부 동작 원리부터 Jetpack Compose와 Flutter에서의 완전한 구현까지 심층적으로 다룹니다.

## Material You란 무엇인가

Material You(내부 코드명 Monet)는 Google이 Android 12에서 발표한 디자인 시스템으로, **Material Design 3(M3)** 의 핵심 기능입니다. 기존 Material Design이 앱 개발자가 고정된 색상 팔레트를 직접 정의해야 했다면, Material You는 사용자의 선호(배경화면, 색상 선택기)로부터 _실시간으로_ 색상 팔레트를 생성합니다.

### 핵심 개념: HCT 색공간

Material You의 색상 과학은 **HCT(Hue-Chroma-Tone)** 색공간을 기반으로 합니다. 이는 Google이 Cam16(인지적 색상 외관 모델)과 L\*a\*b\* 색공간을 결합해 설계한 것으로, 세 가지 주요 특성을 가집니다.

- **Hue(색조)**: 0–360도의 색상환에서 색의 종류를 나타냄
- **Chroma(채도)**: 색상의 선명도. 0이면 무채색, 높을수록 선명
- **Tone(명도)**: 0(완전한 검정)~100(완전한 흰색)으로 밝기를 표현

기존 RGB나 HSL과 달리 HCT는 _인간의 시각적 인식_ 에 더 가깝게 설계되어, 같은 Tone 값이면 색조가 달라도 인지적으로 비슷한 밝기로 보입니다. 이 덕분에 접근성 기준(WCAG 대비율)을 보장하면서도 아름다운 팔레트를 자동 생성할 수 있습니다.

### 색상 생성 파이프라인

배경화면 → 개인화 색상이 되는 과정은 다음 4단계입니다.

1. **소스 색상 추출**: 배경화면 이미지에서 대표 색상 1개를 추출
2. **5가지 키 색상 생성**: Primary, Secondary, Tertiary, Neutral, Neutral Variant
3. **토널 팔레트 생성**: 각 키 색상마다 13개 Tone 값(0, 10, 20, 30, 40, 50, 60, 70, 80, 90, 95, 99, 100)의 팔레트 생성
4. **색상 역할(Color Role) 매핑**: 팔레트의 특정 Tone을 primary, onPrimary, primaryContainer 등의 역할에 할당

Android 12에서는 _Tonal Spot_ 변형만 지원했지만, Android 13부터는 Neutral, Vibrant Tonal, Expressive 등 4가지 변형을 사용자가 직접 선택할 수 있습니다.

## 왜 Material 3 Dynamic Color가 필요한가

### 1. 사용자 개인화와 감성적 연결

정적인 브랜드 색상만 사용하는 앱은 모든 사용자에게 동일한 시각적 경험을 제공합니다. Dynamic Color를 도입하면 앱이 사용자의 기기와 _조화롭게_ 어우러져 "내 앱"처럼 느껴지는 감성적 연결을 만들어냅니다.

### 2. 자동화된 접근성 보장

HCT 기반의 토널 팔레트는 색상 역할 간 대비율을 자동으로 보장합니다. `primary`와 `onPrimary`는 항상 WCAG AA 기준(4.5:1 이상)을 만족하도록 설계되어 있어, 개발자가 수동으로 접근성을 검증하는 부담이 줄어듭니다.

### 3. 라이트/다크 모드 일관성

Dynamic Color는 라이트와 다크 테마 각각에 최적화된 별도 색상 역할을 제공합니다. 개발자는 `isSystemInDarkTheme()` 하나로 모드를 감지하고, 색상 선택은 시스템에 위임하면 됩니다.

### 4. Material 3 Expressive와의 통합

2025년 Google I/O에서 발표된 **Material 3 Expressive**는 M3의 다음 단계로, 모션·컴포넌트·타이포그래피를 더욱 표현력 있게 발전시킨 버전입니다. Android 16 시각 스타일과 함께 작동하며, 기존 M3 Dynamic Color 인프라 위에서 동작합니다.

## Jetpack Compose에서 Dynamic Color 완전 구현

### 의존성 설정

```kotlin
// build.gradle.kts (app)
dependencies {
    implementation("androidx.compose.material3:material3:1.3.1")
    implementation("androidx.compose.material3:material3-window-size-class:1.3.1")
}
```

### 완전한 Dynamic Color 테마 구현

다음은 Dynamic Color를 지원하면서 구형 기기(Android 12 미만)에서는 정적 팔레트로 폴백하는 완전한 테마 구현입니다.

```kotlin
// ui/theme/Color.kt
import androidx.compose.ui.graphics.Color

// 정적 폴백용 브랜드 색상 (Material Theme Builder로 생성)
val md_theme_light_primary = Color(0xFF476810)
val md_theme_light_onPrimary = Color(0xFFFFFFFF)
val md_theme_light_primaryContainer = Color(0xFFC7F089)
val md_theme_light_onPrimaryContainer = Color(0xFF0F2000)
val md_theme_light_secondary = Color(0xFF57624A)
val md_theme_light_onSecondary = Color(0xFFFFFFFF)
val md_theme_light_secondaryContainer = Color(0xFFDAE7C8)
val md_theme_light_onSecondaryContainer = Color(0xFF151E0C)
val md_theme_light_surface = Color(0xFFFDFCF5)
val md_theme_light_onSurface = Color(0xFF1B1C17)
val md_theme_light_surfaceVariant = Color(0xFFE1E4D5)
val md_theme_light_onSurfaceVariant = Color(0xFF44483D)

val md_theme_dark_primary = Color(0xFFACD370)
val md_theme_dark_onPrimary = Color(0xFF213600)
val md_theme_dark_primaryContainer = Color(0xFF324F00)
val md_theme_dark_onPrimaryContainer = Color(0xFFC7F089)
val md_theme_dark_secondary = Color(0xFFBECBAD)
val md_theme_dark_onSecondary = Color(0xFF293420)
val md_theme_dark_secondaryContainer = Color(0xFF3F4A33)
val md_theme_dark_onSecondaryContainer = Color(0xFFDAE7C8)
val md_theme_dark_surface = Color(0xFF131410)
val md_theme_dark_onSurface = Color(0xFFE4E3DC)
val md_theme_dark_surfaceVariant = Color(0xFF44483D)
val md_theme_dark_onSurfaceVariant = Color(0xFFC4C8BA)
```

```kotlin
// ui/theme/Theme.kt
import android.os.Build
import androidx.compose.foundation.isSystemInDarkTheme
import androidx.compose.material3.*
import androidx.compose.runtime.Composable
import androidx.compose.ui.platform.LocalContext

private val LightColorScheme = lightColorScheme(
    primary = md_theme_light_primary,
    onPrimary = md_theme_light_onPrimary,
    primaryContainer = md_theme_light_primaryContainer,
    onPrimaryContainer = md_theme_light_onPrimaryContainer,
    secondary = md_theme_light_secondary,
    onSecondary = md_theme_light_onSecondary,
    secondaryContainer = md_theme_light_secondaryContainer,
    onSecondaryContainer = md_theme_light_onSecondaryContainer,
    surface = md_theme_light_surface,
    onSurface = md_theme_light_onSurface,
    surfaceVariant = md_theme_light_surfaceVariant,
    onSurfaceVariant = md_theme_light_onSurfaceVariant,
)

private val DarkColorScheme = darkColorScheme(
    primary = md_theme_dark_primary,
    onPrimary = md_theme_dark_onPrimary,
    primaryContainer = md_theme_dark_primaryContainer,
    onPrimaryContainer = md_theme_dark_onPrimaryContainer,
    secondary = md_theme_dark_secondary,
    onSecondary = md_theme_dark_onSecondary,
    secondaryContainer = md_theme_dark_secondaryContainer,
    onSecondaryContainer = md_theme_dark_onSecondaryContainer,
    surface = md_theme_dark_surface,
    onSurface = md_theme_dark_onSurface,
    surfaceVariant = md_theme_dark_surfaceVariant,
    onSurfaceVariant = md_theme_dark_onSurfaceVariant,
)

@Composable
fun MyAppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    // Android 12 이상에서 Dynamic Color 활성화 여부 (기본값 true)
    dynamicColor: Boolean = true,
    content: @Composable () -> Unit
) {
    val colorScheme = when {
        // Dynamic Color는 Android 12(API 31) 이상에서만 지원
        dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S -> {
            val context = LocalContext.current
            if (darkTheme) dynamicDarkColorScheme(context)
            else dynamicLightColorScheme(context)
        }
        darkTheme -> DarkColorScheme
        else -> LightColorScheme
    }

    MaterialTheme(
        colorScheme = colorScheme,
        typography = AppTypography,
        shapes = AppShapes,
        content = content
    )
}
```

### 색상 역할 올바르게 사용하기

M3의 색상 역할 시스템에서 가장 중요한 원칙은 **컨테이너-온컨테이너 쌍** 입니다.

```kotlin
@Composable
fun ArticleCard(
    title: String,
    summary: String,
    isHighlighted: Boolean,
    onClick: () -> Unit
) {
    Card(
        onClick = onClick,
        colors = CardDefaults.cardColors(
            // 강조 카드: primaryContainer 배경, onPrimaryContainer 텍스트
            // 일반 카드: surfaceVariant 배경, onSurfaceVariant 텍스트
            containerColor = if (isHighlighted)
                MaterialTheme.colorScheme.primaryContainer
            else
                MaterialTheme.colorScheme.surfaceVariant,
            contentColor = if (isHighlighted)
                MaterialTheme.colorScheme.onPrimaryContainer
            else
                MaterialTheme.colorScheme.onSurfaceVariant
        ),
        elevation = CardDefaults.cardElevation(defaultElevation = 2.dp)
    ) {
        Column(modifier = Modifier.padding(16.dp)) {
            Text(
                text = title,
                style = MaterialTheme.typography.titleMedium,
                // contentColor를 통해 자동 적용되므로 color 파라미터 불필요
            )
            Spacer(modifier = Modifier.height(8.dp))
            Text(
                text = summary,
                style = MaterialTheme.typography.bodyMedium,
                // 덜 중요한 텍스트는 같은 색상의 70% 불투명도 대신
                // onSurface 계열 색상 사용 (M3 권장)
                color = if (isHighlighted)
                    MaterialTheme.colorScheme.onPrimaryContainer.copy(alpha = 0.8f)
                else
                    MaterialTheme.colorScheme.onSurfaceVariant
            )
        }
    }
}

// 버튼 계층 구조: 강조도에 따른 올바른 사용
@Composable
fun ActionButtons() {
    Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
        // 가장 중요한 액션: FilledButton (primary 색상)
        Button(onClick = {}) {
            Text("저장")
        }
        // 중간 강조: FilledTonalButton (secondaryContainer 색상)
        FilledTonalButton(onClick = {}) {
            Text("임시저장")
        }
        // 낮은 강조: OutlinedButton
        OutlinedButton(onClick = {}) {
            Text("취소")
        }
    }
}
```

### 토널 Elevation과 Surface 색상

M3에서는 어두운 테마의 elevation을 그림자 대신 **토널 오버레이** 로 표현합니다. `Surface` 컴포저블의 `tonalElevation` 파라미터가 이를 처리합니다.

```kotlin
@Composable
fun ElevationShowcase() {
    Column(verticalArrangement = Arrangement.spacedBy(8.dp)) {
        // tonalElevation이 높을수록 primary 색상이 surface에 더 많이 혼합
        listOf(0.dp, 1.dp, 3.dp, 6.dp, 8.dp, 12.dp).forEach { elevation ->
            Surface(
                tonalElevation = elevation,
                shape = MaterialTheme.shapes.medium,
                modifier = Modifier
                    .fillMaxWidth()
                    .height(48.dp)
            ) {
                Box(contentAlignment = Alignment.Center) {
                    Text(
                        text = "Tonal Elevation: $elevation",
                        style = MaterialTheme.typography.labelMedium
                    )
                }
            }
        }
    }
}
```

## Flutter에서 Material 3 Dynamic Color 구현

Flutter에서는 `dynamic_color` 패키지를 통해 Android와 일부 데스크탑 플랫폼(Linux, macOS, Windows)의 Dynamic Color를 지원합니다.

### 의존성 설정

```yaml
# pubspec.yaml
dependencies:
  flutter:
    sdk: flutter
  dynamic_color: ^1.7.0
  material_color_utilities: ^0.11.1
```

### 완전한 Dynamic Color 앱 구현

```dart
import 'package:dynamic_color/dynamic_color.dart';
import 'package:flutter/material.dart';

// 정적 폴백용 시드 색상
const _brandColor = Color(0xFF476810);

// 커스텀 브랜드 색상 (동적 팔레트와 조화롭게 조정될 색상)
const _customWarningColor = Color(0xFFF4B942);
const _customSuccessColor = Color(0xFF34C759);

void main() {
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    // DynamicColorBuilder: OS의 Dynamic Color를 감지하여 콜백으로 제공
    return DynamicColorBuilder(
      builder: (ColorScheme? lightDynamic, ColorScheme? darkDynamic) {
        ColorScheme lightColorScheme;
        ColorScheme darkColorScheme;

        if (lightDynamic != null && darkDynamic != null) {
          // OS Dynamic Color 사용 가능 (Android 12+)
          // harmonized()로 커스텀 색상을 동적 팔레트와 조화롭게 조정
          lightColorScheme = lightDynamic.harmonized();
          darkColorScheme = darkDynamic.harmonized();
        } else {
          // 폴백: 브랜드 색상에서 M3 색상 체계 생성
          lightColorScheme = ColorScheme.fromSeed(
            seedColor: _brandColor,
            brightness: Brightness.light,
          );
          darkColorScheme = ColorScheme.fromSeed(
            seedColor: _brandColor,
            brightness: Brightness.dark,
          );
        }

        return MaterialApp(
          title: 'Dynamic Color Demo',
          theme: ThemeData(
            colorScheme: lightColorScheme,
            useMaterial3: true,
          ),
          darkTheme: ThemeData(
            colorScheme: darkColorScheme,
            useMaterial3: true,
          ),
          themeMode: ThemeMode.system,
          home: const DemoScreen(),
        );
      },
    );
  }
}

// 색상 조화(harmonization) 활용 예제
class DemoScreen extends StatelessWidget {
  const DemoScreen({super.key});

  @override
  Widget build(BuildContext context) {
    final colorScheme = Theme.of(context).colorScheme;

    // 커스텀 색상을 현재 Dynamic Color와 조화롭게 조정
    // harmonizeWith: 색상의 색조를 primary 색상 쪽으로 약간 이동
    final harmonizedWarning = _customWarningColor.harmonizeWith(colorScheme.primary);
    final harmonizedSuccess = _customSuccessColor.harmonizeWith(colorScheme.primary);

    return Scaffold(
      backgroundColor: colorScheme.surface,
      appBar: AppBar(
        // AppBar는 자동으로 colorScheme.surface + tonal elevation 사용
        title: const Text('Material You 데모'),
        actions: [
          IconButton(
            icon: const Icon(Icons.palette_outlined),
            onPressed: () {},
          ),
        ],
      ),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            // Primary Container 카드
            Card(
              color: colorScheme.primaryContainer,
              child: Padding(
                padding: const EdgeInsets.all(16.0),
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Text(
                      '동적 색상 활성화됨',
                      style: Theme.of(context).textTheme.titleMedium?.copyWith(
                        color: colorScheme.onPrimaryContainer,
                      ),
                    ),
                    const SizedBox(height: 4),
                    Text(
                      '이 색상은 배경화면에서 추출됩니다',
                      style: Theme.of(context).textTheme.bodySmall?.copyWith(
                        color: colorScheme.onPrimaryContainer.withOpacity(0.8),
                      ),
                    ),
                  ],
                ),
              ),
            ),
            const SizedBox(height: 16),
            // 조화된 커스텀 색상 표시
            Row(
              children: [
                _ColorChip(
                  label: '경고',
                  color: harmonizedWarning,
                  onColor: Colors.white,
                ),
                const SizedBox(width: 8),
                _ColorChip(
                  label: '성공',
                  color: harmonizedSuccess,
                  onColor: Colors.white,
                ),
                const SizedBox(width: 8),
                _ColorChip(
                  label: '오류',
                  color: colorScheme.error,
                  onColor: colorScheme.onError,
                ),
              ],
            ),
          ],
        ),
      ),
      floatingActionButton: FloatingActionButton.extended(
        onPressed: () {},
        // FAB는 자동으로 primaryContainer 색상 사용
        icon: const Icon(Icons.add),
        label: const Text('새 항목'),
      ),
    );
  }
}

class _ColorChip extends StatelessWidget {
  final String label;
  final Color color;
  final Color onColor;

  const _ColorChip({
    required this.label,
    required this.color,
    required this.onColor,
  });

  @override
  Widget build(BuildContext context) {
    return Chip(
      label: Text(label, style: TextStyle(color: onColor)),
      backgroundColor: color,
    );
  }
}
```

### ColorScheme.fromSeed의 고급 활용

Flutter 3.22 이후부터 `ColorScheme.fromSeed`는 `dynamicSchemeVariant` 파라미터를 지원합니다. Android 13의 색상 변형과 동일한 효과를 Flutter에서도 구현할 수 있습니다.

```dart
// 다양한 DynamicSchemeVariant 비교
final schemes = {
  'Tonal Spot (기본)': ColorScheme.fromSeed(
    seedColor: _brandColor,
    dynamicSchemeVariant: DynamicSchemeVariant.tonalSpot,
  ),
  'Vibrant': ColorScheme.fromSeed(
    seedColor: _brandColor,
    dynamicSchemeVariant: DynamicSchemeVariant.vibrant,
  ),
  'Expressive': ColorScheme.fromSeed(
    seedColor: _brandColor,
    dynamicSchemeVariant: DynamicSchemeVariant.expressive,
  ),
  'Neutral': ColorScheme.fromSeed(
    seedColor: _brandColor,
    dynamicSchemeVariant: DynamicSchemeVariant.neutral,
  ),
  'Fidelity (시드 색상 충실)': ColorScheme.fromSeed(
    seedColor: _brandColor,
    dynamicSchemeVariant: DynamicSchemeVariant.fidelity,
  ),
};

// 고대비 모드 지원
final highContrastScheme = ColorScheme.fromSeed(
  seedColor: _brandColor,
  // 0.0 = 표준, 0.5 = 중간 대비, 1.0 = 고대비
  contrastLevel: 1.0,
);
```

## 주의사항과 실전 팁

### 1. 색상 역할 혼용 금지

M3에서 가장 흔한 실수는 **컨테이너와 온컨테이너 색상을 뒤바꾸는 것** 입니다.

```kotlin
// ❌ 잘못된 사용: tertiaryContainer 위에 primaryContainer 텍스트
Button(
    colors = ButtonDefaults.buttonColors(
        containerColor = MaterialTheme.colorScheme.tertiaryContainer,
        contentColor = MaterialTheme.colorScheme.primaryContainer // 대비율 보장 안 됨!
    )
) { Text("잘못된 버튼") }

// ✅ 올바른 사용: container와 onContainer는 항상 쌍으로
Button(
    colors = ButtonDefaults.buttonColors(
        containerColor = MaterialTheme.colorScheme.primary,
        contentColor = MaterialTheme.colorScheme.onPrimary
    )
) { Text("올바른 버튼") }
```

### 2. 브랜드 색상과 Dynamic Color의 공존

기업 앱에서는 브랜드 색상을 완전히 포기하기 어렵습니다. 이 경우 `HarmonizedColors`(Android Views) 또는 `harmonizeWith`(Flutter)를 사용해 브랜드 색상이 Dynamic Color 팔레트와 자연스럽게 어우러지도록 합니다.

```kotlin
// Android Views: 애플리케이션 레벨에서 색상 조화 적용
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        // Dynamic Color를 모든 Activity에 적용
        DynamicColors.applyToActivitiesIfAvailable(this)
    }
}

// 특정 커스텀 색상을 Dynamic Color와 조화롭게 조정
// (XML 테마에서 사용하는 방식)
// HarmonizedColors.applyToContextIfAvailable(
//     context,
//     HarmonizedColorsOptions.Builder()
//         .setColorAttributes(HarmonizedColorAttributes.createMaterialDefaults())
//         .build()
// )
```

### 3. Android 12 미만 기기 지원

Dynamic Color는 API 31 이상에서만 동작합니다. `Build.VERSION.SDK_INT >= Build.VERSION_CODES.S` 조건 없이 `dynamicLightColorScheme()`을 호출하면 런타임 크래시가 발생합니다.

```kotlin
// ❌ 위험: API 체크 없이 Dynamic Color 호출
val colors = dynamicLightColorScheme(context) // API 31 미만에서 크래시!

// ✅ 안전: 버전 확인 후 폴백
val colors = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
    if (darkTheme) dynamicDarkColorScheme(context)
    else dynamicLightColorScheme(context)
} else {
    if (darkTheme) DarkColorScheme
    else LightColorScheme
}
```

### 4. 미리보기에서 Dynamic Color 테스트

Compose Preview는 실제 기기가 아니므로 Dynamic Color를 사전에 적용할 수 없습니다. 대신 `dynamicColor = false`를 명시해 정적 테마로 미리보기하거나, Android Studio의 UI Mode를 활용하세요.

```kotlin
@Preview(name = "Light Mode", uiMode = UI_MODE_NIGHT_NO)
@Preview(name = "Dark Mode", uiMode = UI_MODE_NIGHT_YES)
@Composable
private fun ArticleCardPreview() {
    MyAppTheme(dynamicColor = false) { // 미리보기에서는 정적 팔레트 사용
        ArticleCard(
            title = "미리보기 제목",
            summary = "미리보기 요약 텍스트",
            isHighlighted = true,
            onClick = {}
        )
    }
}
```

### 5. Surface Container 색상 역할 (M3 1.1 이후)

Material 3 사양이 업데이트되면서 기존 `background`/`surface`에 더해 `surfaceContainerLowest`, `surfaceContainerLow`, `surfaceContainer`, `surfaceContainerHigh`, `surfaceContainerHighest` 등 더 세분화된 Surface 색상 역할이 추가됐습니다. 카드, 모달, 바텀시트 등의 계층 구조를 표현할 때 활용하세요.

```kotlin
@Composable
fun LayeredContainers() {
    val cs = MaterialTheme.colorScheme
    // 배경: surfaceContainerLowest (가장 낮은 계층)
    // 카드: surfaceContainer
    // 모달 바텀시트: surfaceContainerHigh
    // 다이얼로그: surfaceContainerHighest

    Surface(color = cs.surfaceContainerLowest) {
        Surface(
            color = cs.surfaceContainer,
            shape = MaterialTheme.shapes.medium,
            modifier = Modifier.padding(16.dp)
        ) {
            Text(
                text = "계층 구조 예시",
                modifier = Modifier.padding(16.dp),
                style = MaterialTheme.typography.bodyMedium
            )
        }
    }
}
```

### 6. Material Theme Builder 활용

[Material Theme Builder](https://material.io/material-theme-builder)를 사용하면 브랜드 색상을 입력해 M3 호환 `Color.kt`와 `Theme.kt`를 자동 생성할 수 있습니다. Figma 플러그인도 지원하므로 디자이너-개발자 협업 시 특히 유용합니다.

---

Material You와 Material 3의 Dynamic Color 시스템은 단순한 색상 변경이 아니라 _사용자와 앱 사이의 감성적 연결_ 을 만드는 UX 혁신입니다. HCT 색공간 기반의 토널 팔레트가 접근성을 자동으로 보장하고, Jetpack Compose와 Flutter 모두 간결한 API로 이를 지원합니다. 브랜드 색상과 Dynamic Color의 조화로운 공존 전략을 함께 적용하면, 개성 있으면서도 모든 사용자에게 접근 가능한 앱을 만들 수 있습니다.

## 참고 자료

- [Material Design 3 in Compose 공식 문서](https://developer.android.com/jetpack/compose/designsystems/material3)
- [Android Dynamic Colors 공식 가이드](https://developer.android.com/develop/ui/views/theming/dynamic-colors)
- [dynamic_color Flutter 패키지 (pub.dev)](https://pub.dev/packages/dynamic_color)
