---
layout: post
title: "Kotlin 타입 안전 DSL 빌더 완전 정복: 람다 수신자·@DslMarker·Gradle KTS까지"
date: 2026-08-27
categories: [android, flutter]
tags: [kotlin, dsl, builder-pattern, dslmarker, gradle-kts, android, lambda-receiver]
---

## 개요

Kotlin이 단순한 Java 대체재를 넘어 **도메인 특화 언어(DSL, Domain-Specific Language)** 구축의 일류 도구로 자리 잡은 이유는 무엇일까요? Jetpack Compose의 `Column { ... }`, Gradle KTS의 `dependencies { ... }`, Ktor의 `routing { get("/") { ... } }` — 이 모두가 Kotlin DSL의 산물입니다. 이 글에서는 타입 안전 DSL을 처음부터 직접 설계하고, `@DslMarker`로 범위를 제어하며, 실제 Android 프로젝트에서 어떻게 활용되는지까지 낱낱이 파헤칩니다.

---

## 왜 DSL이 필요한가?

전통적인 빌더 패턴(Java Builder Pattern)을 떠올려 보세요.

```java
// Java 빌더 패턴 — 장황하고 실수가 많다
Dialog dialog = new Dialog.Builder()
    .setTitle("확인")
    .setMessage("계속 진행하시겠습니까?")
    .setPositiveButton("예", listener)
    .setNegativeButton("아니오", null)
    .create();
```

Kotlin DSL을 사용하면 이렇게 됩니다.

```kotlin
val dialog = dialog {
    title = "확인"
    message = "계속 진행하시겠습니까?"
    positiveButton("예") { /* 처리 */ }
    negativeButton("아니오")
}
```

코드가 구조를 **그대로** 반영합니다. 중첩된 데이터 구조, 선언적 UI, 설정 스크립트처럼 *읽는 사람이 형태를 바로 이해해야 하는 곳*에서 DSL이 빛을 발합니다.

---

## 핵심 개념 1: 람다 수신자 (Lambda with Receiver)

Kotlin DSL의 심장은 **함수 리터럴 수신자(Function Literal with Receiver)** 입니다. 일반 람다와 달리, 수신자가 있는 람다 안에서는 수신자 객체의 멤버를 `this.` 없이 직접 호출할 수 있습니다.

```kotlin
// 일반 람다: it 또는 명시적 파라미터 사용
val greet: (String) -> String = { name -> "안녕, $name" }

// 수신자 람다: this = String
val greet: String.() -> String = { "안녕, $this" }
greet("Kotlin")  // → "안녕, Kotlin"
```

빌더 함수에서는 이 패턴을 다음처럼 활용합니다.

```kotlin
class HtmlBuilder {
    private val content = StringBuilder()

    fun h1(text: String) { content.append("<h1>$text</h1>\n") }
    fun p(text: String)  { content.append("<p>$text</p>\n") }
    fun build() = content.toString()
}

// 수신자 람다를 인자로 받는 최상위 함수
fun html(block: HtmlBuilder.() -> Unit): String {
    val builder = HtmlBuilder()
    builder.block()        // HtmlBuilder가 this가 됨
    return builder.build()
}

// 사용
val page = html {
    h1("Kotlin DSL")
    p("람다 수신자로 만드는 HTML 빌더입니다.")
}
println(page)
// <h1>Kotlin DSL</h1>
// <p>람다 수신자로 만드는 HTML 빌더입니다.</p>
```

`html { ... }` 블록 안에서 `this`는 `HtmlBuilder` 인스턴스이므로, `h1()`과 `p()`를 직접 호출할 수 있습니다.

---

## 핵심 개념 2: 중첩 DSL과 @DslMarker

한 단계 더 들어가 중첩 구조를 만들어 봅시다. `html > body > div > p` 같은 계층이 필요하다면 각 레벨마다 별도 빌더 클래스를 만들어야 합니다. 그런데 여기서 **치명적인 함정**이 있습니다.

```kotlin
class DivBuilder {
    fun p(text: String) { /* ... */ }
}

class BodyBuilder {
    fun div(block: DivBuilder.() -> Unit) { /* ... */ }
}

class HtmlBuilder2 {
    fun body(block: BodyBuilder.() -> Unit) { /* ... */ }
}

fun html2(block: HtmlBuilder2.() -> Unit): String { /* ... */; return "" }

// 문제 발생!
html2 {
    body {
        div {
            body {  // ← 이게 컴파일된다! DivBuilder 안에서 BodyBuilder.body()를 호출
                p("오염된 스코프")
            }
        }
    }
}
```

`div { }` 블록 안에서 바깥 `BodyBuilder`의 `body()`를 그대로 호출할 수 있습니다. 이는 Kotlin이 **암묵적 수신자(implicit receiver)**를 스택 방식으로 쌓기 때문입니다. 이 문제를 해결하는 것이 바로 `@DslMarker`입니다.

### @DslMarker 적용

```kotlin
@DslMarker
annotation class HtmlDsl

@HtmlDsl
class DivBuilder2 {
    private val items = mutableListOf<String>()
    fun p(text: String) { items.add("<p>$text</p>") }
    fun build() = items.joinToString("\n") { "  $it" }
}

@HtmlDsl
class BodyBuilder2 {
    private val divs = mutableListOf<DivBuilder2>()

    fun div(block: DivBuilder2.() -> Unit) {
        val builder = DivBuilder2()
        builder.block()
        divs.add(builder)
    }

    fun build() = divs.joinToString("\n") { "<div>\n${it.build()}\n</div>" }
}

@HtmlDsl
class HtmlBuilder3 {
    private var body: BodyBuilder2? = null

    fun body(block: BodyBuilder2.() -> Unit) {
        val builder = BodyBuilder2()
        builder.block()
        body = builder
    }

    fun build() = "<html>\n<body>\n${body?.build()}\n</body>\n</html>"
}

fun html3(block: HtmlBuilder3.() -> Unit): String {
    val b = HtmlBuilder3()
    b.block()
    return b.build()
}

// 이제 스코프 오염이 컴파일 에러로 막힌다!
val result = html3 {
    body {
        div {
            p("올바른 단락")
            // body { } ← 컴파일 에러: 'body' can't be called in this context
        }
    }
}
println(result)
```

`@DslMarker`는 메타 어노테이션입니다. 동일한 마커로 표시된 수신자는 **직전(가장 안쪽) 수신자만 암묵적으로 접근**할 수 있고, 바깥 수신자의 메서드는 명시적으로 `this@HtmlBuilder3.body { }` 형태로만 접근 가능합니다. 컴파일러가 스코프를 강제하므로 런타임 오류 없이 잘못된 사용을 막을 수 있습니다.

---

## 실전 예제 1: Android 알림 DSL 만들기

실제 Android 프로젝트에서 알림(Notification)을 생성하는 DSL을 구축해 봅시다.

```kotlin
@DslMarker
annotation class NotificationDsl

@NotificationDsl
class NotificationActionBuilder(private val context: Context) {
    var label: String = ""
    var icon: Int = 0
    private var pendingIntent: PendingIntent? = null

    fun intent(block: () -> PendingIntent) {
        pendingIntent = block()
    }

    fun build(): NotificationCompat.Action =
        NotificationCompat.Action.Builder(icon, label, pendingIntent).build()
}

@NotificationDsl
class NotificationBuilder(private val context: Context) {
    var title: String = ""
    var message: String = ""
    var channelId: String = "default"
    var smallIcon: Int = android.R.drawable.ic_dialog_info
    var priority: Int = NotificationCompat.PRIORITY_DEFAULT
    var autoCancel: Boolean = true

    private val actions = mutableListOf<NotificationCompat.Action>()

    fun action(block: NotificationActionBuilder.() -> Unit) {
        val actionBuilder = NotificationActionBuilder(context)
        actionBuilder.block()
        actions.add(actionBuilder.build())
    }

    fun build(): Notification {
        val builder = NotificationCompat.Builder(context, channelId)
            .setContentTitle(title)
            .setContentText(message)
            .setSmallIcon(smallIcon)
            .setPriority(priority)
            .setAutoCancel(autoCancel)

        actions.forEach { builder.addAction(it) }
        return builder.build()
    }
}

fun Context.buildNotification(block: NotificationBuilder.() -> Unit): Notification {
    val builder = NotificationBuilder(this)
    builder.block()
    return builder.build()
}

// 사용 예
fun sendDownloadNotification(context: Context, notificationManager: NotificationManagerCompat) {
    val notification = context.buildNotification {
        channelId = "download_channel"
        title = "다운로드 완료"
        message = "파일이 성공적으로 저장되었습니다."
        smallIcon = R.drawable.ic_download_done
        priority = NotificationCompat.PRIORITY_HIGH
        autoCancel = true

        action {
            label = "열기"
            icon = R.drawable.ic_open
            intent {
                PendingIntent.getActivity(
                    context, 0,
                    Intent(context, MainActivity::class.java),
                    PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
                )
            }
        }
    }

    notificationManager.notify(1001, notification)
}
```

알림 생성 로직이 DSL 덕분에 선언적이고 읽기 쉬워졌습니다. `action { }` 블록 안에서 `title = "..."` 같은 `NotificationBuilder`의 속성에는 접근할 수 없습니다(`@DslMarker` 덕분에). 컴파일러가 이를 보장합니다.

---

## 실전 예제 2: 타입 안전 쿼리 빌더

데이터베이스 쿼리나 API 필터를 DSL로 표현하면 SQL 인젝션이나 잘못된 필터 조합 같은 오류를 컴파일 타임에 방지할 수 있습니다.

```kotlin
@DslMarker
annotation class QueryDsl

enum class SortOrder { ASC, DESC }

@QueryDsl
class WhereClauseBuilder {
    private val conditions = mutableListOf<String>()

    infix fun String.eq(value: Any): WhereClauseBuilder {
        conditions.add("$this = '$value'")
        return this@WhereClauseBuilder
    }

    infix fun String.gt(value: Number): WhereClauseBuilder {
        conditions.add("$this > $value")
        return this@WhereClauseBuilder
    }

    infix fun String.like(pattern: String): WhereClauseBuilder {
        conditions.add("$this LIKE '$pattern'")
        return this@WhereClauseBuilder
    }

    fun build() = if (conditions.isEmpty()) "" else "WHERE ${conditions.joinToString(" AND ")}"
}

@QueryDsl
class QueryBuilder(private val table: String) {
    private var whereClause = ""
    private var limitValue: Int? = null
    private var orderByField: String? = null
    private var sortOrder: SortOrder = SortOrder.ASC
    private val selectedFields = mutableListOf<String>()

    fun select(vararg fields: String) {
        selectedFields.addAll(fields)
    }

    fun where(block: WhereClauseBuilder.() -> Unit) {
        val builder = WhereClauseBuilder()
        builder.block()
        whereClause = builder.build()
    }

    fun orderBy(field: String, order: SortOrder = SortOrder.ASC) {
        orderByField = field
        sortOrder = order
    }

    fun limit(count: Int) {
        limitValue = count
    }

    fun build(): String {
        val fields = if (selectedFields.isEmpty()) "*" else selectedFields.joinToString(", ")
        val order = orderByField?.let { " ORDER BY $it ${sortOrder.name}" } ?: ""
        val limit = limitValue?.let { " LIMIT $it" } ?: ""
        return "SELECT $fields FROM $table $whereClause$order$limit".trim()
    }
}

fun query(table: String, block: QueryBuilder.() -> Unit): String {
    val builder = QueryBuilder(table)
    builder.block()
    return builder.build()
}

// 사용 예
val sql = query("users") {
    select("id", "name", "email")
    where {
        "age" gt 18
        "status" eq "active"
        "name" like "김%"
    }
    orderBy("name", SortOrder.ASC)
    limit(50)
}

println(sql)
// SELECT id, name, email FROM users WHERE age > 18 AND status = 'active' AND name LIKE '김%' ORDER BY name ASC LIMIT 50
```

`infix` 함수와 수신자 람다의 조합으로 SQL과 닮은 직관적인 쿼리 DSL이 완성됩니다. `where { }` 블록 안에서 `orderBy()`나 `limit()` 같은 외부 빌더 함수를 실수로 호출할 수 없습니다.

---

## Gradle KTS — 실전 DSL의 현재

Android 프로젝트의 `build.gradle.kts`는 Kotlin DSL의 가장 대중적인 응용 사례입니다. Gradle의 `Project` 객체가 암묵적 수신자가 되어, 우리가 작성하는 모든 `plugins { }`, `dependencies { }`, `android { }` 블록이 수신자 람다로 동작합니다.

```kotlin
// build.gradle.kts
plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.android)
}

android {
    namespace = "com.example.myapp"
    compileSdk = 35

    defaultConfig {
        applicationId = "com.example.myapp"
        minSdk = 26
        targetSdk = 35
        versionCode = 1
        versionName = "1.0.0"
    }

    buildTypes {
        release {
            isMinifyEnabled = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
        debug {
            isDebuggable = true
            applicationIdSuffix = ".debug"
        }
    }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
}

dependencies {
    implementation(libs.androidx.core.ktx)
    implementation(libs.androidx.lifecycle.runtime.ktx)
    testImplementation(libs.junit)
}
```

`buildTypes { release { ... } }` 같은 중첩 블록이 자연스럽게 읽히는 이유는 각 블록마다 수신자 타입이 바뀌기 때문입니다. `release { }` 안의 `isMinifyEnabled`는 `BuildType` 수신자의 속성이고, `android { }` 안의 `compileSdk`는 `CommonExtension` 수신자의 속성입니다.

---

## 주의사항 및 팁

**1. 과도한 DSL 설계를 피할 것**

DSL은 *자주 쓰이고, 구조가 명확하며, 다양한 사람이 작성하는* 부분에 적합합니다. 한두 번만 쓰이는 설정에 DSL을 만들면 배우는 비용이 더 큽니다.

**2. `@DslMarker`는 항상 적용할 것**

중첩 수신자가 두 개 이상이라면 반드시 `@DslMarker`로 스코프를 제한하세요. 없으면 컴파일은 되지만 런타임 버그나 혼란스러운 동작이 생깁니다.

**3. 기본값을 최대한 제공할 것**

DSL 사용자는 필요한 것만 명시하고 나머지는 합리적인 기본값을 기대합니다. 필수 필드는 생성자 파라미터로, 선택 필드는 프로퍼티 기본값으로 분리하세요.

**4. IDE 자동완성을 염두에 두고 설계할 것**

각 블록 안에서 노출되는 메서드가 적을수록 자동완성이 명확해집니다. 수신자 클래스에 불필요한 공개 API를 두지 마세요.

**5. `operator fun invoke` 활용**

싱글턴 객체에서 DSL처럼 보이는 API를 만들 때는 `invoke` 연산자를 활용할 수 있습니다.

```kotlin
object Logger {
    private var level: String = "INFO"
    private var tag: String = "App"

    operator fun invoke(block: Logger.() -> Unit) {
        block()
    }

    fun level(l: String) { level = l }
    fun tag(t: String)   { tag = t }
    fun log(msg: String) = println("[$level][$tag] $msg")
}

Logger {
    level("DEBUG")
    tag("MainActivity")
    log("DSL 설정 완료")
}
```

---

## 마무리

Kotlin 타입 안전 DSL 빌더는 세 가지 언어 기능의 조합으로 완성됩니다.

| 기능 | 역할 |
|---|---|
| 람다 수신자 (`T.() -> Unit`) | 블록 안에서 `this` 없이 멤버 접근 가능 |
| `@DslMarker` | 중첩 스코프 오염 방지, 컴파일러 강제 |
| `infix` / `operator` 함수 | 읽기 좋은 DSL 문법 구현 |

Compose의 `@Composable`, Ktor의 `routing { }`, Gradle KTS 모두 이 세 가지를 기반으로 동작합니다. 직접 DSL을 설계하면 팀 코드베이스의 가독성과 안전성이 동시에 올라갑니다. 다음 번에 중첩 빌더 패턴이 필요하다면, `@DslMarker`와 람다 수신자로 팀원 모두가 실수 없이 쓸 수 있는 API를 만들어 보세요.

---

## 참고 자료

- [Type-safe builders — Kotlin 공식 문서](https://kotlinlang.org/docs/type-safe-builders.html)
- [Gradle build configuration을 Kotlin DSL로 마이그레이션하기 — Android Developers](https://developer.android.com/build/migrate-to-kotlin-dsl)
- [Gradle Kotlin DSL Primer — Gradle 공식 문서](https://docs.gradle.org/current/userguide/kotlin_dsl.html)
- [Kotlin DSL Deep Dive — ProAndroidDev](https://proandroiddev.com/kotlin-dsl-deep-dive-from-gradle-scripts-to-building-your-own-8e08a1471467)
