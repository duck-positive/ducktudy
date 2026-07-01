---
layout: post
title: "Android R8 심화: 코드 축소·난독화·최적화·매핑 파일 완전 정복"
date: 2026-07-01
categories: [android]
tags: [android, r8, proguard, shrinking, obfuscation, optimization, keep-rules, mapping]
---

Android 앱을 릴리스 빌드할 때 APK 크기를 줄이고 성능을 향상시키는 핵심 도구가 **R8**입니다. R8은 구글이 ProGuard를 대체하기 위해 개발한 컴파일러로, 코드 축소(Shrinking)·난독화(Obfuscation)·최적화(Optimization)·DEX 변환을 단일 패스에서 처리합니다. Android Studio 3.4(AGP 3.4.0)부터 기본 도구로 채택되었으며, ProGuard에 비해 빌드 속도가 빠르고 최적화 수준이 더 높습니다.

이 아티클에서는 R8의 내부 동작 원리부터 실전 Keep Rules 작성법, Full Mode 마이그레이션, 매핑 파일 활용까지 깊이 있게 다룹니다.

---

## 왜 R8가 필요한가?

릴리스 앱에서 R8을 사용하지 않으면 다음과 같은 문제가 발생합니다.

**APK 크기 팽창**: 개발 과정에서 포함된 사용되지 않는 클래스, 메서드, 필드가 모두 포함됩니다. 특히 대형 라이브러리(Retrofit, OkHttp, Room 등)를 사용할 경우 수십 MB의 불필요한 코드가 포함될 수 있습니다.

**DEX 메서드 수 제한**: Android는 단일 DEX 파일에서 최대 65,536개의 메서드만 참조할 수 있습니다(64K 제한). R8의 코드 축소 없이는 대형 프로젝트에서 Multidex 설정이 필수가 됩니다.

**역공학 노출**: 난독화 없이 배포된 앱은 jadx, apktool 같은 도구로 소스 코드가 그대로 복원될 수 있습니다. API 키, 비즈니스 로직, 서버 엔드포인트 등 민감 정보가 쉽게 추출됩니다.

**성능 저하**: 인라이닝, 불필요한 체크 제거, 클래스 병합 같은 최적화를 거치지 않으면 런타임 성능이 저하됩니다.

R8은 이 네 가지 문제를 단일 컴파일 패스에서 해결합니다.

---

## R8의 세 가지 핵심 기능

### 1. 코드 축소 (Code Shrinking / Tree Shaking)

R8은 앱의 진입점(Entry Point)인 `AndroidManifest.xml`에 등록된 Activity, Service, BroadcastReceiver, ContentProvider부터 시작하여 참조 그래프를 탐색합니다. 이 탐색 과정에서 한 번도 참조되지 않는 클래스와 멤버는 최종 DEX에서 제거됩니다.

### 2. 난독화 (Obfuscation)

제거 후 남은 클래스·메서드·필드명을 `a`, `b`, `c` 같은 짧은 이름으로 치환합니다. 이는 APK 크기를 추가로 줄이고 역공학을 어렵게 만듭니다. 매핑 파일(`mapping.txt`)은 원본 이름과 난독화된 이름의 대응 관계를 기록하여 크래시 스택 트레이스 복원에 사용됩니다.

### 3. 최적화 (Optimization)

메서드 인라이닝, 불변 필드 상수 폴딩, 빈 클래스 제거, 클래스 병합(class merging), 불필요한 null 체크 제거 등 수십 가지 최적화를 적용합니다. Full Mode에서는 이 최적화 범위가 더욱 넓어집니다.

---

## 실전 예제 1: build.gradle 설정과 proguard-rules.pro

R8을 활성화하는 기본 설정과 실전 Keep Rules입니다.

```kotlin
// app/build.gradle.kts
android {
    buildTypes {
        release {
            isMinifyEnabled = true       // R8 코드 축소·난독화·최적화 활성화
            isShrinkResources = true     // 미사용 리소스도 함께 제거
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
}
```

```proguard
# proguard-rules.pro

# Gson 직렬화 모델: 필드명이 보존되어야 역직렬화 가능
-keep class com.example.app.data.model.** { *; }
-keepclassmembers class com.example.app.data.model.** {
    <fields>;
}

# Retrofit 인터페이스: 메서드 시그니처 보존
-keep interface com.example.app.data.api.** { *; }

# Enum 클래스: values()와 valueOf() 메서드 보존
-keepclassmembers enum * {
    public static **[] values();
    public static ** valueOf(java.lang.String);
}

# Parcelable: CREATOR 필드 보존
-keepclassmembers class * implements android.os.Parcelable {
    public static final android.os.Parcelable$Creator CREATOR;
}

# JNI 네이티브 메서드: 이름이 바뀌면 링크 실패
-keepclasseswithmembernames class * {
    native <methods>;
}

# allowshrinking 수식어: 클래스는 keep하되 멤버가 미사용이면 제거 허용
-keep,allowshrinking class com.example.app.util.ReflectionHelper { *; }

# 불필요한 경고 억제 (특정 라이브러리 전용으로만 사용)
-dontwarn org.bouncycastle.**
-dontwarn okio.**
```

`proguard-android-optimize.txt`는 `proguard-android.txt`보다 더 공격적인 최적화를 포함합니다. 신규 프로젝트라면 항상 `-optimize` 버전을 사용하세요.

---

## Keep Rule 수식어(Modifier) 정리

| 수식어 | 의미 |
|---|---|
| (없음) | 클래스·멤버 모두 keep, 리네임·제거 불가 |
| `allowshrinking` | 미사용 시 제거 허용, 사용 시 이름 유지 |
| `allowobfuscation` | 이름 변경 허용, 제거는 불가 |
| `allowoptimization` | R8 내부 최적화 적용 허용 |

---

## 실전 예제 2: @Keep 어노테이션과 Kotlin 코드 수준 보존

proguard-rules.pro 파일 대신 코드 레벨에서 보존을 선언할 수도 있습니다.

```kotlin
import androidx.annotation.Keep

// 클래스 전체 보존: 이름과 모든 멤버 유지
@Keep
data class UserDto(
    val id: Long,
    val name: String,
    val email: String
)

// 특정 메서드만 보존: 리플렉션으로 호출되는 메서드만 이름 고정
class AnalyticsHelper {

    @Keep
    fun trackEvent(eventName: String, params: Map<String, Any>) {
        val payload = buildPayload(params)
        // ... 전송 로직
    }

    // @Keep 없음 → R8이 인라이닝 또는 제거 가능
    private fun buildPayload(params: Map<String, Any>): String {
        return params.entries.joinToString("&") { "${it.key}=${it.value}" }
    }
}

// Gson 직렬화 대상: @SerializedName이 있어도 필드가 제거되면 null 반환
data class ApiResponse<T>(
    @com.google.gson.annotations.SerializedName("data")
    val data: T?,
    @com.google.gson.annotations.SerializedName("error_code")
    val errorCode: Int,
    @com.google.gson.annotations.SerializedName("message")
    val message: String
)

// Hilt 모듈: R8이 제거하지 않도록 @InstallIn이 있으면 자동 보존
// (Hilt의 consumer-rules.pro에 이미 포함되어 있음)
@dagger.Module
@dagger.hilt.InstallIn(dagger.hilt.components.SingletonComponent::class)
object NetworkModule {
    @dagger.Provides
    @javax.inject.Singleton
    fun provideOkHttpClient(): okhttp3.OkHttpClient = okhttp3.OkHttpClient.Builder().build()
}
```

---

## R8 Full Mode 적용

R8 Full Mode는 호환 모드보다 더 공격적인 최적화를 수행합니다. AGP 8.0+에서는 기본값이 Full Mode로 변경되고 있습니다.

```properties
# gradle.properties
# Full Mode 명시적 활성화 (AGP 7.x에서 명시적으로 켜는 경우)
android.enableR8.fullMode=true
```

Full Mode에서 가장 많이 발생하는 문제는 **기본 생성자 미보존**입니다. 호환 모드에서는 클래스를 Keep하면 인수 없는 생성자도 암묵적으로 보존되었지만, Full Mode에서는 명시적으로 선언해야 합니다.

```proguard
# Full Mode: 기본 생성자 명시적 보존
-keepclassmembers class com.example.app.data.model.** {
    <init>();
}

# Full Mode: Kotlin data class copy() 메서드 보존
-keepclassmembers class com.example.app.data.model.** {
    public ** copy(...);
}

# Full Mode: 어노테이션 속성 보존 (호환 모드에서는 자동 보존됨)
-keepattributes *Annotation*
-keepattributes Signature
-keepattributes InnerClasses
-keepattributes EnclosingMethod
```

---

## 매핑 파일과 크래시 분석

릴리스 빌드 후 `app/build/outputs/mapping/release/mapping.txt`가 생성됩니다.

**구글 플레이 콘솔 업로드**: 앱 번들 업로드 시 매핑 파일도 함께 업로드하면 Firebase Crashlytics와 Android Vitals에서 자동으로 역난독화된 스택 트레이스를 제공합니다.

**수동 역난독화 (retrace)**:
```bash
# Android SDK의 retrace 도구 사용
java -jar retrace.jar mapping.txt stacktrace.txt

# Firebase Crashlytics CLI로 역난독화
firebase crashlytics:mappingfile:upload --app=APP_ID mapping.txt
```

**버전별 매핑 파일 보관**: 각 릴리스의 매핑 파일은 버전 코드와 함께 Git LFS나 내부 스토리지에 보관하세요. 오래된 매핑 파일이 없으면 해당 버전의 크래시를 분석할 수 없습니다.

---

## 주의사항 및 실전 팁

**1. 리플렉션 코드 전수 조사**: Gson, Retrofit 인터페이스, Room Entity, Hilt 주입 대상, `Class.forName()` 호출 부분은 모두 Keep 규칙이 필요합니다. 동적으로 클래스를 로드하는 코드는 R8이 참조 없다고 판단해 제거할 수 있습니다.

**2. `-dontwarn` 남용 금지**: 경고를 무시하는 것은 임시방편입니다. 근본 원인을 파악하고 적절한 `-keep` 규칙으로 해결하세요. `-dontwarn **` 같은 와일드카드 사용은 실제 문제를 숨깁니다.

**3. 라이브러리 consumer-rules.pro 확인**: Retrofit, OkHttp, Gson, Room, Hilt, Coil 등 주요 라이브러리는 AAR 내부에 `consumer-rules.pro`를 포함해 자동으로 Keep Rules가 적용됩니다. 라이브러리 업그레이드 시 이 파일이 함께 업데이트되었는지 확인하세요.

**4. CI/CD에 릴리스 빌드 테스트 포함**: 개발 중 디버그 빌드만 테스트하다가 릴리스 직전에야 R8 관련 버그를 발견하는 경우가 많습니다. PR 머지 후 릴리스 빌드를 자동으로 실행하고 스모크 테스트를 돌리는 파이프라인을 구성하세요.

**5. APK Analyzer로 Before/After 비교**: Android Studio의 **Build > Analyze APK** 기능으로 R8 적용 전후의 클래스·메서드 수와 APK 크기를 비교할 수 있습니다. DEX 탭에서 어떤 클래스가 최종 APK에 포함되었는지 직접 확인하세요.

**6. `-printseeds`, `-printusage` 활용**:
```proguard
# R8이 Keep한 진입점 목록 출력
-printseeds seeds.txt
# R8이 제거한 코드 목록 출력
-printusage usage.txt
```
이 두 파일을 분석하면 의도치 않게 제거되거나 보존된 코드를 찾아낼 수 있습니다.

---

## 마치며

R8은 단순한 코드 압축 도구가 아니라 앱 품질을 전반적으로 끌어올리는 컴파일러 최적화 파이프라인입니다. 올바른 Keep Rules를 작성하고, Full Mode의 장점을 활용하며, 매핑 파일을 체계적으로 관리한다면 APK 크기와 런타임 성능 모두에서 의미 있는 개선을 얻을 수 있습니다. 특히 AGP 8.x 이상에서는 Full Mode가 기본값으로 채택되는 방향으로 나아가고 있으므로, 지금부터 호환성을 확보해두는 것이 중요합니다.

## 참고 자료
- [Enable app optimization with R8 - Android Developers](https://developer.android.com/topic/performance/app-optimization/enable-app-optimization)
- [Use R8 in full mode - Android Developers](https://developer.android.com/topic/performance/app-optimization/full-mode)
- [Adopt optimizations incrementally - Android Developers](https://developer.android.com/topic/performance/app-optimization/adopt-optimizations-incrementally)
