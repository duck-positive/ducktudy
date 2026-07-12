---
layout: post
title: "Kotlin Value Class 심화: 타입 안전성과 제로 비용 추상화를 동시에 잡는 법"
date: 2026-07-12
categories: [android, kotlin]
tags: [kotlin, value-class, inline-class, jvm, android, performance, type-safety]
---

Android 앱을 개발하다 보면 동일한 원시 타입(`Long`, `String`)을 여러 의미로 혼용하여 미묘한 버그를 만드는 경우가 잦습니다. Kotlin의 **Value Class(값 클래스)**는 이 문제를 컴파일 타임에 차단하면서도 런타임 성능 저하를 일으키지 않는 강력한 도구입니다. 이번 글에서는 Value Class의 내부 동작 원리부터 Android 실무 적용 패턴까지 심층적으로 살펴봅니다.

---

## 1. Value Class란 무엇인가

Value Class(이전 명칭: Inline Class)는 Kotlin 1.5에서 안정화된 기능으로, **단일 프로퍼티를 가진 클래스를 JVM 바이트코드에서 해당 프로퍼티 타입으로 대체**할 수 있게 해주는 특수한 클래스입니다.

컴파일 타임에는 완전한 타입으로 동작하여 타입 안전성을 제공하지만, JVM 바이트코드에서는 래퍼 객체를 생성하지 않고 내부 값을 직접 사용합니다. JVM 백엔드에서는 `@JvmInline` 애노테이션과 `value` 변경자를 함께 사용해야 합니다.

```kotlin
@JvmInline
value class UserId(val value: Long)
```

Kotlin 컴파일러는 `UserId`를 사용하는 곳에서 가능한 경우 래퍼 객체를 제거하고 `Long`을 직접 사용합니다. 이것이 "인라이닝(inlining)"이며, 이로 인해 **힙 할당이 발생하지 않아** 원시 타입과 동일한 수준의 성능을 유지합니다.

### Type Alias와의 결정적 차이

`typealias`는 진정한 타입 안전성을 제공하지 않습니다. Type Alias는 별칭일 뿐, 원본 타입과 완전히 호환됩니다.

```kotlin
typealias UserId = Long
typealias MessageId = Long

fun sendMessage(from: UserId, to: UserId, message: MessageId) { ... }

val userId: UserId = 1001L
val messageId: MessageId = 999L

// 컴파일 성공! 타입 안전성이 전혀 없다
sendMessage(messageId, userId, userId)
```

반면 Value Class는 서로 호환되지 않는 진짜 별개의 타입을 만들어 위와 같은 실수를 **컴파일 에러**로 차단합니다.

---

## 2. 왜 Value Class가 필요한가

### 문제: 원시 타입의 무분별한 혼용

실제 프로젝트에서는 다음과 같은 코드가 흔합니다.

```kotlin
// 모든 파라미터가 Long — 실수하기 너무 쉽다
fun placeOrder(userId: Long, productId: Long, couponId: Long): Long { ... }

// 순서를 잘못 전달해도 컴파일러가 잡지 못한다
val orderId = placeOrder(product.id, coupon.id, user.id) // 버그!
```

이런 문제를 일반 클래스로 해결하면 **힙 할당과 GC 압박**이 증가합니다. 핫 패스(hot path)에서 수천 번 호출되는 함수라면 성능에 직결됩니다.

### 해결: Value Class로 타입 안전성과 성능을 동시에

Value Class는 컴파일 타임에 타입 검사를 수행하고, 런타임에는 원시 타입처럼 동작합니다.

```kotlin
@JvmInline
value class UserId(val value: Long) {
    init { require(value > 0) { "UserId는 양수여야 합니다" } }
}

@JvmInline
value class ProductId(val value: Long) {
    init { require(value > 0) { "ProductId는 양수여야 합니다" } }
}

@JvmInline
value class CouponId(val value: Long)

@JvmInline
value class OrderId(val value: Long)

fun placeOrder(userId: UserId, productId: ProductId, couponId: CouponId): OrderId { ... }

// 이제 순서를 바꾸면 컴파일 에러!
// placeOrder(ProductId(1L), UserId(2L), CouponId(3L)) // Type mismatch
val orderId = placeOrder(UserId(1L), ProductId(2L), CouponId(3L)) // OK
```

`init` 블록을 통해 유효성 검증도 자동으로 수행됩니다. `UserId(-1L)`을 만들면 즉시 `IllegalArgumentException`이 발생합니다.

---

## 3. 실제 구현 예제

### 예제 1: 도메인 모델의 타입 안전성 완성

멤버 함수, 계산 프로퍼티, 연산자 오버로딩까지 Value Class 내에서 모두 사용할 수 있습니다.

```kotlin
@JvmInline
value class Email(val value: String) {
    init {
        require(value.contains('@') && value.contains('.')) {
            "유효하지 않은 이메일 형식: $value"
        }
    }

    val domain: String get() = value.substringAfter('@')
    val localPart: String get() = value.substringBefore('@')

    fun isCompanyEmail(companyDomain: String): Boolean =
        domain.equals(companyDomain, ignoreCase = true)

    fun masked(): String {
        val local = localPart
        return "${local.first()}***@$domain"
    }
}

@JvmInline
value class PhoneNumber(val value: String) {
    init {
        val digits = value.filter { it.isDigit() }
        require(digits.length in 10..11) { "유효하지 않은 전화번호: $value" }
    }

    val digitsOnly: String get() = value.filter { it.isDigit() }

    fun formatted(): String {
        val d = digitsOnly
        return if (d.length == 11) {
            "${d.substring(0, 3)}-${d.substring(3, 7)}-${d.substring(7)}"
        } else {
            "${d.substring(0, 3)}-${d.substring(3, 6)}-${d.substring(6)}"
        }
    }
}

// 사용 예
fun registerUser(email: Email, phone: PhoneNumber) {
    println("이메일: ${email.masked()}")    // u***@example.com
    println("전화번호: ${phone.formatted()}") // 010-1234-5678
    println("회사 이메일 여부: ${email.isCompanyEmail("company.com")}")
}

// registerUser("not-email", "01012345678") // 컴파일 에러
registerUser(Email("user@example.com"), PhoneNumber("01012345678")) // OK
```

### 예제 2: Android UI 단위 안전성 — Dp, Px, Sp 혼용 방지

Android 개발에서 픽셀(px), dp, sp 단위를 혼동하는 것은 레이아웃 버그의 흔한 원인입니다. Value Class로 이를 컴파일 타임에 차단할 수 있습니다.

```kotlin
@JvmInline
value class Dp(val value: Float) {
    operator fun plus(other: Dp) = Dp(value + other.value)
    operator fun minus(other: Dp) = Dp(value - other.value)
    operator fun times(factor: Float) = Dp(value * factor)
    operator fun div(factor: Float) = Dp(value / factor)
    operator fun compareTo(other: Dp) = value.compareTo(other.value)
    override fun toString() = "${value}dp"
}

@JvmInline
value class Sp(val value: Float) {
    operator fun plus(other: Sp) = Sp(value + other.value)
    override fun toString() = "${value}sp"
}

@JvmInline
value class Px(val value: Float) {
    operator fun plus(other: Px) = Px(value + other.value)
    override fun toString() = "${value}px"
}

// 단위 변환 확장 함수
fun Dp.toPx(density: Float): Px = Px(value * density)
fun Px.toDp(density: Float): Dp = Dp(value / density)
fun Sp.toPx(scaledDensity: Float): Px = Px(value * scaledDensity)

// Android View 설정 유틸리티
fun View.setWidthDp(width: Dp) {
    val density = context.resources.displayMetrics.density
    val params = layoutParams ?: ViewGroup.LayoutParams(
        ViewGroup.LayoutParams.WRAP_CONTENT,
        ViewGroup.LayoutParams.WRAP_CONTENT
    )
    params.width = width.toPx(density).value.toInt()
    layoutParams = params
}

fun View.setPaddingDp(horizontal: Dp, vertical: Dp) {
    val density = context.resources.displayMetrics.density
    val h = horizontal.toPx(density).value.toInt()
    val v = vertical.toPx(density).value.toInt()
    setPadding(h, v, h, v)
}

// 사용 예 — 컴파일 타임에 단위 혼용 차단
fun setupButton(button: Button) {
    val width = Dp(200f)
    val padding = Dp(16f)
    val extraPadding = Dp(8f)

    button.setWidthDp(width)
    button.setPaddingDp(padding + extraPadding, Dp(12f)) // Dp + Dp = Dp

    // button.setWidthDp(Px(200f)) // 컴파일 에러! Px ≠ Dp
    // button.setPaddingDp(Sp(16f), Dp(12f)) // 컴파일 에러! Sp ≠ Dp
}
```

Jetpack Compose가 내부적으로 `Dp`, `Color`, `TextUnit`을 Value Class로 구현한 것도 이 패턴의 실전 검증입니다.

---

## 4. JVM 내부 동작: 박싱이 발생하는 경우

Value Class가 항상 인라이닝되는 것은 아닙니다. 다음 상황에서는 래퍼 객체가 생성(박싱)됩니다.

```kotlin
@JvmInline
value class UserId(val value: Long)

// 박싱 발생 ①: 제네릭 타입으로 사용될 때
val list: List<UserId> = listOf(UserId(1L), UserId(2L)) // 각 UserId가 박싱됨

// 박싱 발생 ②: nullable 타입으로 사용될 때
val nullableId: UserId? = UserId(1L) // 박싱됨

// 박싱 발생 ③: 인터페이스 타입으로 업캐스팅될 때
interface Identifiable { val id: Long }

@JvmInline
value class OrderId(override val id: Long) : Identifiable
val identifiable: Identifiable = OrderId(42L) // 박싱됨

// 인라이닝 유지 ①: 직접 타입으로 선언된 파라미터
fun processUser(userId: UserId) {
    println(userId.value)
    // JVM 바이트코드에서는: processUser(J)V — Long 직접 사용
}

// 인라이닝 유지 ②: 로컬 변수
fun example() {
    val id = UserId(100L) // 힙 할당 없음
    processUser(id)       // 인라이닝
}
```

### 네임 맹글링(Name Mangling)

Value Class를 파라미터로 받는 함수는 JVM에서 이름이 변경됩니다. Java와의 이름 충돌을 방지하기 위한 조치입니다.

```kotlin
fun processUser(id: UserId) { ... }
// JVM에서: processUser-<해시값>(J)V 로 컴파일됨
```

Java에서 호출이 필요하면 `@JvmName` 또는 Kotlin 2.2+의 `@JvmExposeBoxed`를 사용하세요.

```kotlin
// @JvmName으로 Java 친화적 이름 노출
@JvmName("processUserById")
fun processUser(id: UserId) { ... }

// Kotlin 2.2+: 박싱된 타입을 Java에 노출
@JvmExposeBoxed
@JvmInline
value class UserId(val value: Long)
```

---

## 5. 주의사항 및 실무 팁

### ① 직렬화 라이브러리와의 통합

`kotlinx.serialization`은 Value Class를 공식 지원합니다. 단, `@Serializable` 애노테이션을 명시해야 합니다.

```kotlin
@Serializable
@JvmInline
value class UserId(val value: Long)

@Serializable
data class User(
    val id: UserId,
    val name: String
)

// JSON: {"id": 1001, "name": "홍길동"} — id가 Long으로 직렬화됨
```

Gson이나 Moshi 사용 시에는 별도 커스텀 어댑터가 필요합니다.

### ② Room 데이터베이스와의 통합

Room에서는 `TypeConverter`를 제공해야 합니다.

```kotlin
object Converters {
    @TypeConverter
    fun fromUserId(id: UserId): Long = id.value

    @TypeConverter
    fun toUserId(value: Long): UserId = UserId(value)
}

@Database(entities = [OrderEntity::class], version = 1)
@TypeConverters(Converters::class)
abstract class AppDatabase : RoomDatabase()
```

### ③ Jetpack Compose와의 조합

Compose 안정성 시스템과 통합하면 불필요한 리컴포지션을 방지할 수 있습니다.

```kotlin
@Immutable
@JvmInline
value class ThemeColor(val argb: Int) {
    val alpha: Int get() = (argb shr 24) and 0xFF
    val red: Int get() = (argb shr 16) and 0xFF
    val green: Int get() = (argb shr 8) and 0xFF
    val blue: Int get() = argb and 0xFF
}
```

### ④ 언제 Value Class를 사용해야 하는가

| 사용 권장 | 사용 비권장 |
|---|---|
| 도메인 식별자 (`UserId`, `OrderId`) | 두 개 이상의 프로퍼티가 필요한 경우 |
| 측정 단위 (`Dp`, `Px`, `Sp`) | 자주 박싱이 발생하는 컨텍스트 |
| 검증이 필요한 원시값 (이메일, 전화번호) | Gson 등 리플렉션 기반 직렬화 |
| 민감한 값 (`Password`, `AuthToken`) | Java 코드와의 광범위한 상호운용 |

---

## 마치며

Kotlin Value Class는 **"비용 없는 추상화(zero-cost abstraction)"** 의 완벽한 구현입니다. 컴파일 타임에는 타입 안전성을, 런타임에는 원시 타입의 성능을 동시에 제공합니다. 특히 Android 앱에서 도메인 모델을 설계할 때 Value Class를 적극 활용하면 코드의 표현력이 높아지고, 실수로 인한 런타임 버그를 사전에 차단할 수 있습니다.

Jetpack Compose가 내부적으로 `Dp`, `Color`, `TextUnit`을 모두 Value Class로 구현한 것은 이 패턴의 실전 검증이며, 우리의 도메인 코드에도 같은 철학을 적용할 충분한 이유가 됩니다.

## 참고 자료

- [Kotlin 공식 문서 — Inline value classes](https://kotlinlang.org/docs/inline-classes.html)
- [Kotlin API 레퍼런스 — @JvmInline](https://kotlinlang.org/api/core/kotlin-stdlib/kotlin.jvm/-jvm-inline/)
- [Android Kotlin 스타일 가이드](https://developer.android.com/kotlin/style-guide)
