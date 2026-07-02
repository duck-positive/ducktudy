---
layout: post
title: "Kotlin KSP 심화: 커스텀 애노테이션 프로세서와 코드 생성 완전 정복"
date: 2026-07-02
categories: [android, kotlin]
tags: [ksp, kotlin, annotation-processing, code-generation, kapt, android]
---

## Kotlin Symbol Processing(KSP)이란?

**Kotlin Symbol Processing(KSP)**은 Kotlin 소스 코드를 분석하고, 애노테이션을 기반으로 새로운 코드를 자동 생성하는 **경량 컴파일러 플러그인 프레임워크**다. Google이 개발했으며, 기존 KAPT(Kotlin Annotation Processing Tool)의 성능 문제를 해결하기 위해 탄생했다.

Room, Hilt, Moshi, Retrofit 같은 유명 라이브러리들은 모두 내부적으로 애노테이션 프로세서를 사용해 보일러플레이트 코드를 자동 생성한다. KSP를 이해하면 이 "마법"의 작동 원리를 파악할 수 있고, 팀 내 반복 코드를 자동화하는 커스텀 도구를 만들 수 있다.

### KSP vs KAPT: 왜 전환해야 하는가?

KAPT는 Kotlin 코드를 **Java 스텁(stub)으로 변환**한 뒤 Java 애노테이션 프로세서를 실행하는 방식이다. 이 두 단계 변환 과정이 빌드 속도의 발목을 잡는다.

| 항목 | KAPT | KSP |
|------|------|-----|
| 처리 방식 | Kotlin → Java 스텁 → 프로세서 | Kotlin 소스 직접 분석 |
| 빌드 속도 | 느림 (스텁 생성 오버헤드) | 최대 2배 빠름 |
| Kotlin 멀티플랫폼 | 미지원 | 지원 |
| Kotlin 언어 이해 | 제한적 | 네이티브 (확장 함수, data class 등 인식) |
| 현재 상태 | 유지보수 모드(deprecated 예정) | 적극 개발 중 (KSP2 기본값) |

KSP 2.0.0부터 **KSP2**가 기본 활성화되었으며, KSP1은 deprecated 상태다. 새 프로젝트라면 반드시 KSP2를 사용해야 한다.

---

## KSP의 핵심 구조: 어떻게 동작하는가?

KSP 프로세서는 컴파일 단계에서 Kotlin 컴파일러에 의해 호출된다. 동작 흐름은 다음과 같다.

```
소스 코드 (.kt)
    ↓
KSP API → KSResolver로 심볼 탐색
    ↓
SymbolProcessor.process() 호출
    ↓
KSPLogger로 오류/경고 로그
    ↓
CodeGenerator로 .kt 파일 생성
    ↓
컴파일러가 생성된 코드 포함하여 최종 빌드
```

핵심 인터페이스는 세 가지다:
- **`SymbolProcessorProvider`**: 프로세서 인스턴스를 생성하는 팩토리
- **`SymbolProcessor`**: 실제 코드 분석과 생성 로직
- **`KSPLogger`**: 컴파일 오류/경고 출력

---

## 프로젝트 구조 설정

KSP 프로세서를 만들려면 3개의 모듈이 필요하다:

```
my-project/
├── app/                     # 메인 앱 (프로세서 소비자)
├── annotations/             # 커스텀 애노테이션 정의
└── processor/               # KSP 프로세서 구현
```

각 모듈의 `build.gradle.kts` 설정:

```kotlin
// annotations/build.gradle.kts
plugins {
    kotlin("jvm")
}

// processor/build.gradle.kts
plugins {
    kotlin("jvm")
}
dependencies {
    implementation("com.google.devtools.ksp:symbol-processing-api:2.1.0-1.0.29")
    implementation(project(":annotations"))
}

// app/build.gradle.kts
plugins {
    id("com.android.application")
    kotlin("android")
    id("com.google.devtools.ksp") version "2.1.0-1.0.29"
}
dependencies {
    implementation(project(":annotations"))
    ksp(project(":processor"))  // kapt 대신 ksp 사용
}
```

---

## 실전 예제 1: `@AutoFactory` — 팩토리 패턴 자동 생성

ViewModel이나 Repository 객체를 생성할 때 팩토리 클래스를 일일이 작성하는 것은 번거롭다. `@AutoFactory` 애노테이션을 붙이면 팩토리 클래스가 자동 생성되도록 구현해보자.

### Step 1: 애노테이션 정의

```kotlin
// annotations/src/main/kotlin/AutoFactory.kt
@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.SOURCE)
annotation class AutoFactory
```

`AnnotationRetention.SOURCE`로 설정하면 컴파일 후 바이트코드에 애노테이션이 남지 않아 런타임 오버헤드가 없다.

### Step 2: SymbolProcessor 구현

```kotlin
// processor/src/main/kotlin/AutoFactoryProcessor.kt
import com.google.devtools.ksp.processing.*
import com.google.devtools.ksp.symbol.*
import com.google.devtools.ksp.validate
import java.io.OutputStream

class AutoFactoryProcessor(
    private val codeGenerator: CodeGenerator,
    private val logger: KSPLogger
) : SymbolProcessor {

    override fun process(resolver: Resolver): List<KSAnnotated> {
        // @AutoFactory가 붙은 클래스 심볼 탐색
        val symbols = resolver
            .getSymbolsWithAnnotation("com.example.annotations.AutoFactory")
            .filterIsInstance<KSClassDeclaration>()

        // 아직 처리 불가한 심볼은 다음 라운드로 미룸
        val unprocessed = symbols.filter { !it.validate() }.toList()

        symbols.filter { it.validate() }.forEach { classDecl ->
            generateFactory(classDecl)
        }

        return unprocessed
    }

    private fun generateFactory(classDecl: KSClassDeclaration) {
        val className = classDecl.simpleName.asString()
        val packageName = classDecl.packageName.asString()
        val factoryName = "${className}Factory"

        // 생성자 파라미터 추출
        val primaryConstructor = classDecl.primaryConstructor
            ?: run {
                logger.error("@AutoFactory 클래스는 주 생성자가 필요합니다: $className")
                return
            }

        val params = primaryConstructor.parameters
        val paramList = params.joinToString(", ") { param ->
            val paramName = param.name?.asString() ?: "param"
            val paramType = param.type.resolve().declaration.qualifiedName?.asString() ?: "Any"
            "$paramName: $paramType"
        }
        val argList = params.joinToString(", ") { param ->
            param.name?.asString() ?: "param"
        }

        // 파일 생성 — Dependencies에 원본 파일 기록 (증분 빌드 지원)
        val file: OutputStream = codeGenerator.createNewFile(
            dependencies = Dependencies(aggregating = false, classDecl.containingFile!!),
            packageName = packageName,
            fileName = factoryName
        )

        file.write(
            """
            |package $packageName
            |
            |object $factoryName {
            |    fun create($paramList): $className {
            |        return $className($argList)
            |    }
            |}
            """.trimMargin().toByteArray()
        )
        file.close()

        logger.info("$factoryName 생성 완료 (패키지: $packageName)")
    }
}

// SymbolProcessorProvider — KSP가 찾는 진입점
class AutoFactoryProcessorProvider : SymbolProcessorProvider {
    override fun create(environment: SymbolProcessorEnvironment): SymbolProcessor {
        return AutoFactoryProcessor(
            codeGenerator = environment.codeGenerator,
            logger = environment.logger
        )
    }
}
```

### Step 3: META-INF 서비스 등록

KSP가 프로세서를 자동으로 발견하려면 서비스 파일을 등록해야 한다:

```
processor/src/main/resources/META-INF/services/
    com.google.devtools.ksp.processing.SymbolProcessorProvider
```

파일 내용:
```
com.example.processor.AutoFactoryProcessorProvider
```

### Step 4: 앱 코드에서 사용

```kotlin
// app에서 애노테이션 적용
@AutoFactory
class UserRepository(
    private val api: UserApi,
    private val db: UserDatabase
)
```

빌드 후 `build/generated/ksp/` 폴더에 다음 코드가 자동 생성된다:

```kotlin
// 자동 생성된 파일 (수동 수정 불가)
package com.example.app

object UserRepositoryFactory {
    fun create(api: UserApi, db: UserDatabase): UserRepository {
        return UserRepository(api, db)
    }
}
```

---

## 실전 예제 2: `@Singleton` — 싱글톤 래퍼 자동 생성

두 번째 예제로, `@Singleton` 애노테이션이 붙은 클래스의 싱글톤 홀더를 자동 생성한다. Kotlin `object`를 직접 쓰지 않고 **스레드 안전한 lazy 싱글톤**을 생성해보자.

```kotlin
// annotations/src/main/kotlin/Singleton.kt
@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.SOURCE)
annotation class Singleton
```

```kotlin
// processor/src/main/kotlin/SingletonProcessor.kt
class SingletonProcessor(
    private val codeGenerator: CodeGenerator,
    private val logger: KSPLogger
) : SymbolProcessor {

    override fun process(resolver: Resolver): List<KSAnnotated> {
        val symbols = resolver
            .getSymbolsWithAnnotation("com.example.annotations.Singleton")
            .filterIsInstance<KSClassDeclaration>()
            .filter { it.validate() }

        symbols.forEach { classDecl ->
            val className = classDecl.simpleName.asString()
            val packageName = classDecl.packageName.asString()
            val holderName = "${className}Holder"

            // 주 생성자 파라미터가 있는지 검증
            val hasParams = (classDecl.primaryConstructor?.parameters?.size ?: 0) > 0
            if (hasParams) {
                logger.error(
                    "@Singleton은 파라미터 없는 주 생성자를 가진 클래스에만 적용 가능합니다: $className",
                    classDecl
                )
                return@forEach
            }

            val file = codeGenerator.createNewFile(
                dependencies = Dependencies(false, classDecl.containingFile!!),
                packageName = packageName,
                fileName = holderName
            )

            // Double-checked locking 패턴으로 스레드 안전 싱글톤 생성
            file.write(
                """
                |package $packageName
                |
                |object $holderName {
                |    @Volatile
                |    private var instance: $className? = null
                |
                |    fun getInstance(): $className {
                |        return instance ?: synchronized(this) {
                |            instance ?: $className().also { instance = it }
                |        }
                |    }
                |
                |    fun resetForTest() {
                |        synchronized(this) { instance = null }
                |    }
                |}
                """.trimMargin().toByteArray()
            )
            file.close()

            logger.info("$holderName 생성 (스레드 안전 싱글톤)")
        }

        return emptyList()
    }
}

class SingletonProcessorProvider : SymbolProcessorProvider {
    override fun create(environment: SymbolProcessorEnvironment): SymbolProcessor =
        SingletonProcessor(environment.codeGenerator, environment.logger)
}
```

이 프로세서는 몇 가지 중요한 패턴을 보여준다:
- `logger.error(..., classDecl)`처럼 **심볼 위치를 같이 전달**하면 IDE에서 오류 위치가 정확히 표시된다.
- `resetForTest()`를 생성해 **테스트 격리**를 지원한다.
- `@Volatile` + `synchronized` 조합으로 **DCL(Double-Checked Locking)** 패턴을 적용한다.

---

## 다중 라운드 처리 이해하기

KSP는 **다중 라운드(multi-round processing)**를 지원한다. 1라운드에서 생성된 코드에 애노테이션이 있으면 2라운드에서 다시 처리된다.

`process()`가 반환하는 `List<KSAnnotated>`는 "아직 처리 못한 심볼" 목록이다. 처리하지 못한 심볼은 반환해서 다음 라운드에 재시도하게 해야 한다.

```kotlin
override fun process(resolver: Resolver): List<KSAnnotated> {
    val symbols = resolver
        .getSymbolsWithAnnotation("com.example.annotations.AutoFactory")
        .filterIsInstance<KSClassDeclaration>()

    // validate()가 false인 심볼은 의존하는 타입이 아직 처리되지 않은 상태
    // → 다음 라운드로 미뤄야 함
    val deferred = symbols.filter { !it.validate() }.toList()
    
    symbols.filter { it.validate() }.forEach { generateFactory(it) }

    return deferred  // 빈 리스트 반환 시 더 이상 처리할 것 없음을 알림
}
```

---

## 증분 빌드(Incremental Build) 최적화

KSP는 증분 빌드를 지원하지만, `Dependencies` 설정을 올바르게 해야 실제로 효과를 볼 수 있다.

```kotlin
// aggregating = false: 특정 파일에만 의존 (빠름, 대부분의 경우)
// aggregating = true: 모든 파일에 의존 (느림, 전체 스캔이 필요할 때)
val deps = Dependencies(
    aggregating = false,        // 소스 파일 변경 시에만 재생성
    classDecl.containingFile!!  // 의존하는 소스 파일 등록
)

// 여러 파일에 의존할 경우
val deps = Dependencies(
    aggregating = true,
    *resolver.getAllFiles().toList().toTypedArray()
)
```

`aggregating = false`로 설정하면 변경된 파일과 관련된 코드만 재생성하므로 빌드가 훨씬 빠르다. `@AutoFactory`처럼 **클래스 1개 → 파일 1개** 매핑이면 반드시 `aggregating = false`를 사용하라.

---

## 주의사항 및 실전 팁

### 1. 순환 참조 방지
생성된 코드가 다시 애노테이션을 포함하면 무한 루프가 발생할 수 있다. 생성 파일에는 애노테이션을 붙이지 않는 것이 원칙이다.

### 2. `qualifiedName` vs `simpleName`
타입을 임포트할 때는 `simpleName`이 아닌 `qualifiedName`을 사용해야 한다. 동일한 이름의 클래스가 다른 패키지에 있을 경우 충돌이 발생한다.

```kotlin
// 위험: simpleName만 사용
val typeName = param.type.resolve().declaration.simpleName.asString()

// 안전: qualifiedName 사용
val typeName = param.type.resolve().declaration.qualifiedName?.asString()
    ?: error("타입의 qualified name을 찾을 수 없습니다")
```

### 3. 테스트 작성
KSP 프로세서는 `kotlin-compile-testing-ksp` 라이브러리를 사용해 단위 테스트를 작성할 수 있다.

```kotlin
// processor/build.gradle.kts
testImplementation("com.github.tschuchortdev:kotlin-compile-testing-ksp:1.6.0")
```

```kotlin
// AutoFactoryProcessorTest.kt
@Test
fun `@AutoFactory가 붙은 클래스에 팩토리가 생성된다`() {
    val source = SourceFile.kotlin("User.kt", """
        package com.example
        import com.example.annotations.AutoFactory
        
        @AutoFactory
        class User(val name: String, val age: Int)
    """)

    val result = KotlinCompilation().apply {
        sources = listOf(source)
        symbolProcessorProviders = listOf(AutoFactoryProcessorProvider())
        inheritClassPath = true
    }.compile()

    assertEquals(KotlinCompilation.ExitCode.OK, result.exitCode)
    assertTrue(result.generatedFiles.any { it.name == "UserFactory.kt" })
}
```

### 4. KSP 옵션으로 동작 제어
`build.gradle.kts`에서 프로세서에 옵션을 전달할 수 있다.

```kotlin
// app/build.gradle.kts
ksp {
    arg("autoFactory.generateToString", "true")
    arg("autoFactory.packageSuffix", ".generated")
}
```

프로세서에서 옵션 읽기:
```kotlin
val generateToString = environment.options["autoFactory.generateToString"] == "true"
```

### 5. 생성 코드 위치 확인
빌드 후 생성된 파일은 다음 경로에 위치한다:
```
build/generated/ksp/main/kotlin/   # JVM/Android
build/generated/ksp/debug/kotlin/  # Android debug variant
```

Android Studio에서는 **Project 뷰 → Generated Sources** 폴더에서 확인 가능하다.

---

## 마이그레이션 체크리스트: KAPT → KSP

프로젝트에 KAPT가 남아있다면 다음 순서로 마이그레이션하자:

1. 사용 중인 라이브러리의 KSP 지원 여부 확인 (`Room`, `Hilt`, `Moshi` 모두 KSP 지원)
2. `build.gradle.kts`에 KSP 플러그인 추가
3. `kapt(...)` 의존성을 `ksp(...)`로 교체
4. `annotationProcessor(...)` 도 `ksp(...)`로 교체
5. 모든 모듈에서 `apply plugin: 'kotlin-kapt'` 제거
6. `kaptGenerateStubsDebug` 태스크가 빌드에서 사라졌는지 확인

---

## 마치며

KSP는 단순한 빌드 속도 개선 도구를 넘어, **반복 코드 자동화의 철학**을 코드베이스에 적용할 수 있게 해준다. `@AutoFactory`, `@Singleton`처럼 팀 내 공통 패턴을 애노테이션으로 추상화하면 코드 일관성이 높아지고 리뷰 비용도 줄어든다.

KSP2 전환으로 API가 더욱 정교해진 만큼, 지금이 커스텀 프로세서 작성을 시작하기 가장 좋은 시점이다.

## 참고 자료
- [Kotlin Symbol Processing API 공식 문서](https://kotlinlang.org/docs/ksp-overview.html)
- [Getting Started with KSP](https://kotlinlang.org/docs/ksp-quickstart.html)
- [Migrate from kapt to KSP — Android Developers](https://developer.android.com/build/migrate-to-ksp)
- [KSP GitHub Repository (google/ksp)](https://github.com/google/ksp)
