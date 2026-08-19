---
layout: post
title: "Android Custom Lint Rules 심화: UAST Detector로 팀 코딩 규칙을 컴파일 타임에 강제하는 정적 분석 도구 만들기"
date: 2026-08-15
categories: [android]
tags: [android, lint, static-analysis, kotlin, uast, code-quality, ci]
---

Android 개발팀이 성장하면서 가장 먼저 부딪히는 문제 중 하나는 **코드 품질의 일관성 유지**입니다. "Log.d 대신 팀 전용 Logger를 써야 한다", "ViewModel에서 직접 Context를 참조하면 안 된다", "특정 deprecated API는 절대 사용하지 말라" 같은 규칙은 코드 리뷰만으로는 실수를 막기 어렵습니다. 이런 규칙을 컴파일 타임 또는 CI 빌드 단계에서 자동으로 강제할 수 있다면 어떨까요?

**Android Custom Lint Rules**는 바로 그 해법입니다. 구글이 제공하는 Lint 프레임워크를 활용해 팀만의 정적 분석 규칙을 만들고, 기존 Android Lint와 동일한 방식으로 동작하게 할 수 있습니다.

---

## 1. Android Lint란 무엇인가

Android Lint는 Android SDK에 내장된 정적 코드 분석 도구입니다. 실행 없이 소스 코드와 리소스를 스캔하여 잠재적인 버그, 성능 문제, 보안 취약점, 접근성 문제 등을 찾아냅니다.

```bash
# 린트 실행
./gradlew lint
./gradlew lintRelease
```

Lint는 `lint.xml` 파일이나 Gradle DSL로 심각도를 제어할 수 있습니다:

```kotlin
// build.gradle.kts (app 모듈)
android {
    lint {
        abortOnError = true
        error += "CustomIssueId"
        baseline = file("lint-baseline.xml")
    }
}
```

기본으로 제공되는 수백 개의 규칙 외에도, **개발팀이 직접 규칙을 정의**할 수 있습니다. 이것이 Custom Lint Rules의 핵심입니다.

---

## 2. 왜 Custom Lint Rules가 필요한가

### 코드 리뷰의 한계

코드 리뷰는 팀 규칙을 강제하는 좋은 방법이지만 한계가 있습니다:

- 리뷰어가 놓칠 수 있습니다
- 동일한 지적을 반복하는 피로도가 쌓입니다
- PR 머지 후 발견되면 수정 비용이 큽니다

### Custom Lint로 해결할 수 있는 대표적인 시나리오

| 시나리오 | 커스텀 규칙 |
|----------|------------|
| `Log.d/e` 직접 사용 금지 | 팀 Logger API 사용 강제 |
| `ViewModel`에서 `Context` 직접 참조 | `ApplicationContext` 또는 DI 사용 권고 |
| 특정 라이브러리 API deprecated 처리 | 새 API로 마이그레이션 강제 |
| 하드코딩된 문자열 탐지 | 리소스 string 사용 권고 |
| Jetpack Compose `@Preview` 이름 규칙 | 네이밍 컨벤션 강제 |

### 비용 대비 효과

Custom Lint는 일회성 구현으로 **모든 팀원의 모든 커밋**을 자동으로 검사합니다. 인간 리뷰어의 주의력과 달리 절대 놓치지 않습니다.

---

## 3. 프로젝트 구조 설정

Custom Lint Rules는 별도 모듈로 분리하는 것이 표준 패턴입니다.

```
my-project/
├── app/
├── library/          ← AAR로 배포되는 라이브러리 모듈
│   └── build.gradle.kts
├── lint-checks/      ← Lint 규칙 구현 모듈 (JAR)
│   └── build.gradle.kts
└── build.gradle.kts
```

### lint-checks 모듈 설정

```kotlin
// lint-checks/build.gradle.kts
plugins {
    id("java-library")
    alias(libs.plugins.kotlin.jvm)
}

// Lint 버전 = AGP 버전 + 23
// AGP 8.3.x → lint 31.3.x
val lintVersion = "31.3.2"

dependencies {
    compileOnly("com.android.tools.lint:lint-api:$lintVersion")
    compileOnly("com.android.tools.lint:lint-checks:$lintVersion")

    testImplementation("com.android.tools.lint:lint-tests:$lintVersion")
    testImplementation("junit:junit:4.13.2")
}

// Lint가 이 JAR에서 IssueRegistry를 찾을 수 있도록 매니페스트 설정
tasks.withType<Jar> {
    manifest {
        attributes["Lint-Registry-v2"] =
            "com.example.lintchecks.MyIssueRegistry"
    }
}
```

### library 모듈 설정

```kotlin
// library/build.gradle.kts
plugins {
    id("com.android.library")
    alias(libs.plugins.kotlin.android)
}

dependencies {
    // lintPublish: AAR에 lint JAR을 포함시켜 의존하는 프로젝트에 자동 배포
    lintPublish(project(":lint-checks"))
}
```

---

## 4. 첫 번째 Detector 구현: Log 사용 금지 규칙

### 4-1. Issue 정의

`Issue`는 Lint 규칙 하나를 나타내는 메타데이터 객체입니다.

```kotlin
// lint-checks/src/main/java/com/example/lintchecks/LogUsageDetector.kt
package com.example.lintchecks

import com.android.tools.lint.detector.api.*

class LogUsageDetector : Detector(), SourceCodeScanner {

    companion object {
        val ISSUE = Issue.create(
            id = "DirectLogUsage",
            briefDescription = "android.util.Log를 직접 사용하지 마세요",
            explanation = """
                `android.util.Log`를 직접 사용하면 릴리스 빌드에서 로그가 유출될 수 있습니다.
                팀 전용 `AppLogger` 유틸리티를 대신 사용하세요.
                
                **잘못된 예:**
                `Log.d("TAG", "message")`
                
                **올바른 예:**
                `AppLogger.d("TAG", "message")`
            """.trimIndent(),
            category = Category.CORRECTNESS,
            priority = 8,
            severity = Severity.ERROR,
            implementation = Implementation(
                LogUsageDetector::class.java,
                Scope.JAVA_FILE_SCOPE
            )
        )
    }

    // 어떤 UAST 노드 타입을 스캔할지 지정
    override fun getApplicableMethodNames(): List<String> =
        listOf("d", "e", "w", "i", "v", "wtf")

    // 메서드 호출이 감지될 때마다 호출되는 콜백
    override fun visitMethodCall(
        context: JavaContext,
        node: UCallExpression,
        method: PsiMethod
    ) {
        // 호출된 클래스가 android.util.Log인지 확인
        if (context.evaluator.isMemberInClass(method, "android.util.Log")) {
            context.report(
                issue = ISSUE,
                scope = node,
                location = context.getCallLocation(node, includeReceiver = true, includeArguments = false),
                message = "`android.util.Log.${method.name}()` 대신 `AppLogger.${method.name}()`을 사용하세요"
            )
        }
    }
}
```

### 4-2. IssueRegistry 등록

```kotlin
// lint-checks/src/main/java/com/example/lintchecks/MyIssueRegistry.kt
package com.example.lintchecks

import com.android.tools.lint.client.api.IssueRegistry
import com.android.tools.lint.client.api.Vendor
import com.android.tools.lint.detector.api.CURRENT_API
import com.android.tools.lint.detector.api.Issue

class MyIssueRegistry : IssueRegistry() {

    override val issues: List<Issue> = listOf(
        LogUsageDetector.ISSUE,
        // 추가 규칙들을 여기에 등록
    )

    override val api: Int = CURRENT_API

    override val minApi: Int = 8

    override val vendor: Vendor = Vendor(
        vendorName = "My Team",
        feedbackUrl = "https://github.com/my-org/my-project/issues"
    )
}
```

---

## 5. 두 번째 Detector 구현: ViewModel에서 Context 금지

UAST의 강력한 기능을 활용해 더 복잡한 규칙을 만들어 보겠습니다. `ViewModel` 하위 클래스에서 `android.content.Context`를 파라미터로 받는 생성자나 함수를 금지하는 규칙입니다.

```kotlin
// lint-checks/src/main/java/com/example/lintchecks/ViewModelContextDetector.kt
package com.example.lintchecks

import com.android.tools.lint.detector.api.*
import com.intellij.psi.PsiMethod
import org.jetbrains.uast.*

class ViewModelContextDetector : Detector(), SourceCodeScanner {

    companion object {
        val ISSUE = Issue.create(
            id = "ViewModelContextLeak",
            briefDescription = "ViewModel에서 Context를 직접 참조하지 마세요",
            explanation = """
                `ViewModel`은 Activity나 Fragment보다 오래 살아남습니다.
                일반 `Context`를 `ViewModel`에 저장하면 메모리 누수가 발생합니다.
                
                - `Context`가 필요하다면 `AndroidViewModel`을 상속하고 `getApplication()`을 사용하세요.
                - 또는 `SavedStateHandle`이나 Repository 패턴을 통해 Context 의존을 제거하세요.
            """.trimIndent(),
            category = Category.PERFORMANCE,
            priority = 9,
            severity = Severity.WARNING,
            implementation = Implementation(
                ViewModelContextDetector::class.java,
                Scope.JAVA_FILE_SCOPE
            )
        )

        private const val VIEWMODEL_CLASS = "androidx.lifecycle.ViewModel"
        private const val CONTEXT_CLASS = "android.content.Context"
    }

    override fun applicableSuperClasses(): List<String> =
        listOf(VIEWMODEL_CLASS)

    override fun visitClass(context: JavaContext, declaration: UClass) {
        // ViewModel 하위 클래스인 경우에만 검사
        if (!context.evaluator.extendsClass(declaration.javaPsi, VIEWMODEL_CLASS, true)) {
            return
        }

        // 모든 필드를 검사
        declaration.fields.forEach { field ->
            val fieldType = field.type.canonicalText
            if (isContextType(fieldType)) {
                context.report(
                    issue = ISSUE,
                    scope = field,
                    location = context.getLocation(field),
                    message = "ViewModel 필드 `${field.name}`이 Context를 참조합니다. " +
                            "메모리 누수가 발생할 수 있습니다."
                )
            }
        }

        // 생성자 파라미터 검사
        declaration.methods
            .filter { it.isConstructor }
            .forEach { constructor ->
                constructor.uastParameters.forEach { param ->
                    val paramType = param.type.canonicalText
                    if (isContextType(paramType)) {
                        context.report(
                            issue = ISSUE,
                            scope = param,
                            location = context.getLocation(param),
                            message = "ViewModel 생성자가 Context 파라미터 `${param.name}`을 받습니다. " +
                                    "`AndroidViewModel`을 상속하고 `getApplication()`을 사용하세요."
                        )
                    }
                }
            }
    }

    private fun isContextType(typeName: String): Boolean =
        typeName == CONTEXT_CLASS ||
                typeName == "android.app.Activity" ||
                typeName == "android.app.Fragment" ||
                typeName == "androidx.fragment.app.Fragment"
}
```

---

## 6. Lint 규칙 테스트

Lint 규칙 테스트는 `LintDetectorTest`를 상속하여 작성합니다. 실제 코드를 작성하고 예상 출력과 비교하는 방식입니다.

```kotlin
// lint-checks/src/test/java/com/example/lintchecks/LogUsageDetectorTest.kt
package com.example.lintchecks

import com.android.tools.lint.checks.infrastructure.LintDetectorTest
import com.android.tools.lint.detector.api.Detector
import com.android.tools.lint.detector.api.Issue
import org.junit.Test

class LogUsageDetectorTest : LintDetectorTest() {

    override fun getDetector(): Detector = LogUsageDetector()

    override fun getIssues(): List<Issue> = listOf(LogUsageDetector.ISSUE)

    @Test
    fun `Log_d 직접 사용 시 에러 보고`() {
        lint()
            .files(
                kotlin("""
                    package com.example

                    import android.util.Log

                    class MyRepository {
                        fun fetchData() {
                            Log.d("MyRepository", "fetching data")
                        }
                    }
                """).indented()
            )
            .run()
            .expect("""
                src/com/example/MyRepository.kt:7: Error: `android.util.Log.d()` 대신 `AppLogger.d()`을 사용하세요 [DirectLogUsage]
                            Log.d("MyRepository", "fetching data")
                            ~~~~~
                1 errors, 0 warnings
            """.trimIndent())
    }

    @Test
    fun `AppLogger 사용 시 정상 통과`() {
        lint()
            .files(
                kotlin("""
                    package com.example

                    class MyRepository {
                        fun fetchData() {
                            AppLogger.d("MyRepository", "fetching data")
                        }
                    }
                """).indented()
            )
            .run()
            .expectClean()
    }

    @Test
    fun `@SuppressLint 어노테이션으로 억제 가능`() {
        lint()
            .files(
                kotlin("""
                    package com.example

                    import android.annotation.SuppressLint
                    import android.util.Log

                    class LegacyCode {
                        @SuppressLint("DirectLogUsage")
                        fun oldMethod() {
                            Log.d("LegacyCode", "legacy log")
                        }
                    }
                """).indented()
            )
            .run()
            .expectClean()
    }
}
```

테스트는 `./gradlew :lint-checks:test` 명령으로 실행합니다.

---

## 7. CI 파이프라인에 통합하기

Custom Lint는 일반 Lint와 동일하게 Gradle로 실행됩니다. CI에서 린트 실패 시 빌드를 중단하도록 설정합니다.

```kotlin
// app/build.gradle.kts
android {
    lint {
        abortOnError = true
        warningsAsErrors = false    // 경고는 허용, 에러만 중단
        htmlReport = true
        htmlOutput = file("${project.rootDir}/lint-results.html")
        
        // 팀 커스텀 규칙 에러로 격상
        error += "DirectLogUsage"
        error += "ViewModelContextLeak"
    }
}
```

GitHub Actions 예시:

```yaml
# .github/workflows/lint.yml
name: Lint Check
on: [pull_request]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
      - name: Run Lint
        run: ./gradlew lint
      - name: Upload Lint Report
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: lint-results
          path: app/build/reports/lint-results*.html
```

---

## 8. 주의사항 및 실전 팁

### Lint API 버전 맞추기

Lint 버전은 반드시 사용 중인 AGP(Android Gradle Plugin) 버전과 맞춰야 합니다.

```
lint 버전 = AGP 버전 + 23
AGP 8.3.x → lint 31.3.x
AGP 8.4.x → lint 31.4.x
```

버전이 맞지 않으면 런타임에 `IllegalArgumentException`이 발생하거나 규칙이 무시됩니다.

### `compileOnly` vs `implementation`

lint-api, lint-checks 의존성은 반드시 `compileOnly`로 선언해야 합니다. `implementation`으로 선언하면 JAR 크기가 불필요하게 커지고 충돌이 발생합니다.

### UAST vs PSI 선택

- **UAST** (`SourceCodeScanner`): Java/Kotlin 공통 코드 분석. 대부분의 경우 권장.
- **PSI** (`JavaPsiScanner`): Kotlin 전용 기능이 필요할 때.
- **XML** (`XmlScanner`): 레이아웃, 매니페스트 등 XML 파일 분석.
- **Gradle** (`GradleScanner`): `build.gradle` 파일 분석.

### 빠른 디버깅: PSI 뷰어

Android Studio에서 `idea.properties` 파일에 `idea.is.internal=true`를 추가하면 **Tools > View PSI Structure** 메뉴가 활성화됩니다. 어떤 UAST 노드 타입을 타겟팅해야 할지 파악하는 데 매우 유용합니다.

### 기존 코드에 점진적 적용: Baseline

이미 오래된 코드베이스에 Custom Lint를 적용할 때는 Baseline 파일을 활용하면 됩니다. 기존 위반은 무시하고 새 코드에만 적용됩니다.

```bash
# 현재 위반 사항으로 baseline 생성
./gradlew lintDebug -Dlint.baselines.continue=true
```

생성된 `lint-baseline.xml`을 버전 관리에 커밋하면 기존 레거시 코드에서는 경고가 발생하지 않습니다.

---

## 9. 마치며

Custom Lint Rules는 초기 설정 비용 대비 팀 전체의 코드 품질 향상 효과가 뛰어난 도구입니다. 코드 리뷰에서 반복되는 지적사항, deprecated API 마이그레이션, 팀 내부 아키텍처 규칙 강제 등 다양한 곳에 활용할 수 있습니다. 

무엇보다 중요한 것은 규칙의 `briefDescription`과 `explanation`을 명확하게 작성하는 것입니다. 팀원이 린트 에러 메시지만 보고도 무엇을 어떻게 고쳐야 하는지 알 수 있어야 합니다.

작은 규칙 하나부터 시작해서, 팀 코드 리뷰에서 반복적으로 등장하는 주제를 차례대로 자동화해 보세요.

## 참고 자료
- [Android Lint 공식 문서 - 코드 작성 및 Lint 실행](https://developer.android.com/studio/write/lint)
- [Google Samples: Android Custom Lint Rules (GitHub)](https://github.com/googlesamples/android-custom-lint-rules)
- [AndroidX Lint Guide (AOSP 소스)](https://android.googlesource.com/platform/frameworks/support/+/refs/heads/androidx-main/docs/lint_guide.md)
