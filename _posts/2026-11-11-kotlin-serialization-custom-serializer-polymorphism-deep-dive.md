---
layout: post
title: "kotlinx.serialization 심화: 커스텀 직렬화, 다형성 처리, SerializersModule 완전 정복"
date: 2026-11-11
categories: [android, kotlin]
tags: [kotlin, serialization, kotlinx-serialization, json, polymorphism, android, kmp]
---

`kotlinx.serialization`은 Kotlin 공식 직렬화 라이브러리입니다. 단순한 JSON 파싱 도구로 보기 쉽지만, 커스텀 직렬화기(KSerializer), 다형성 처리, SerializersModule을 이해하면 복잡한 API 응답도 타입 안전하게 처리할 수 있습니다. 이 글에서는 기초를 넘어 실무에서 마주치는 심화 패턴을 다룹니다.

## 왜 kotlinx.serialization인가?

Android 개발 초기부터 많이 쓰인 Gson과 Moshi에는 각각 한계가 있습니다.

**Gson의 문제점**
- Java 기반 설계로 Kotlin의 non-null 타입 보장 없음
- 런타임 리플렉션 사용으로 ProGuard/R8 적용 시 별도 keep 규칙 필요
- Kotlin data class의 기본 파라미터(default parameter)가 역직렬화 시 무시됨
- Kotlin Multiplatform(KMP) 미지원

**Moshi의 한계**
- Kotlin 친화적이지만 KSP/KAPT codegen을 별도로 설정해야 함
- KMP 지원이 kotlinx.serialization에 비해 제한적

**kotlinx.serialization의 강점**
- Kotlin 컴파일러 플러그인이 `@Serializable` 클래스의 직렬화 코드를 컴파일 타임에 생성
- Kotlin nullable, default parameter 완전 지원
- KMP 네이티브 지원 (JVM, JS, Native 모두 동작)
- JSON, Protobuf, CBOR, Properties 등 다양한 포맷을 하나의 API로 처리
- R8/ProGuard 추가 규칙 불필요

## 의존성 설정

```kotlin
// build.gradle.kts (Module)
plugins {
    kotlin("plugin.serialization") version "1.9.22"
}

dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.6.3")
    // Protobuf 지원이 필요하다면:
    // implementation("org.jetbrains.kotlinx:kotlinx-serialization-protobuf:1.6.3")
}
```

## 기본 사용법 복습

```kotlin
import kotlinx.serialization.*
import kotlinx.serialization.json.*

@Serializable
data class Product(
    val id: Int,
    val name: String,
    val price: Double,
    val inStock: Boolean = true  // 기본값 지원
)

fun main() {
    val product = Product(id = 1, name = "무선 이어폰", price = 89000.0)

    // 직렬화
    val jsonStr = Json.encodeToString(product)
    println(jsonStr)
    // {"id":1,"name":"무선 이어폰","price":89000.0}
    // inStock은 encodeDefaults=false(기본)이므로 포함 안 됨

    // 역직렬화
    val decoded = Json.decodeFromString<Product>(jsonStr)
    println(decoded.inStock) // true (기본값 복원)
}
```

## 커스텀 KSerializer 구현

기본 `@Serializable`로 처리할 수 없는 외부 라이브러리 타입(예: `java.util.UUID`, `java.time.LocalDate`)을 직렬화해야 할 때는 `KSerializer<T>` 인터페이스를 구현합니다.

**KSerializer의 세 가지 구성 요소**
1. `descriptor`: 직렬화할 타입의 구조를 선언
2. `serialize(encoder, value)`: Kotlin 객체 → 직렬화 포맷
3. `deserialize(decoder)`: 직렬화 포맷 → Kotlin 객체

```kotlin
import kotlinx.serialization.*
import kotlinx.serialization.descriptors.*
import kotlinx.serialization.encoding.*
import java.util.UUID
import java.time.LocalDate
import java.time.format.DateTimeFormatter

// UUID → JSON String
object UuidSerializer : KSerializer<UUID> {
    override val descriptor: SerialDescriptor =
        PrimitiveSerialDescriptor("UUID", PrimitiveKind.STRING)

    override fun serialize(encoder: Encoder, value: UUID) {
        encoder.encodeString(value.toString())
    }

    override fun deserialize(decoder: Decoder): UUID {
        return UUID.fromString(decoder.decodeString())
    }
}

// LocalDate → JSON String (ISO 8601 포맷)
object LocalDateSerializer : KSerializer<LocalDate> {
    private val formatter = DateTimeFormatter.ISO_LOCAL_DATE

    override val descriptor: SerialDescriptor =
        PrimitiveSerialDescriptor("LocalDate", PrimitiveKind.STRING)

    override fun serialize(encoder: Encoder, value: LocalDate) {
        encoder.encodeString(value.format(formatter))
    }

    override fun deserialize(decoder: Decoder): LocalDate {
        return LocalDate.parse(decoder.decodeString(), formatter)
    }
}

// 개별 프로퍼티에 직렬화기 지정
@Serializable
data class Order(
    @Serializable(with = UuidSerializer::class) val orderId: UUID,
    @Serializable(with = LocalDateSerializer::class) val orderDate: LocalDate,
    val amount: Long
)
```

**SerializersModule로 글로벌 등록 (반복 제거)**

매 프로퍼티마다 `@Serializable(with = ...)` 를 붙이는 대신, 모듈에 한 번만 등록하고 `@Contextual` 애노테이션으로 참조합니다:

```kotlin
val customTypesModule = SerializersModule {
    contextual(UUID::class, UuidSerializer)
    contextual(LocalDate::class, LocalDateSerializer)
}

val appJson = Json {
    serializersModule = customTypesModule
    ignoreUnknownKeys = true
}

// 이제 @Contextual만 붙이면 자동으로 커스텀 직렬화기 적용
@Serializable
data class Order(
    @Contextual val orderId: UUID,
    @Contextual val orderDate: LocalDate,
    val amount: Long
)

fun main() {
    val order = Order(
        orderId = UUID.randomUUID(),
        orderDate = LocalDate.of(2026, 11, 11),
        amount = 59_000L
    )

    val json = appJson.encodeToString(order)
    println(json)
    // {"orderId":"550e8400-...","orderDate":"2026-11-11","amount":59000}
}
```

## 다형성(Polymorphism) 처리: sealed class

API 응답이 여러 타입 중 하나일 때 sealed class와 다형성 직렬화를 조합하면 타입 안전하게 처리할 수 있습니다.

```kotlin
import kotlinx.serialization.*
import kotlinx.serialization.json.*

@Serializable
sealed class PaymentMethod {
    // @SerialName으로 JSON의 "type" 필드 값을 지정
    @Serializable
    @SerialName("credit_card")
    data class CreditCard(
        val cardNumber: String,
        val expiryMonth: Int,
        val expiryYear: Int
    ) : PaymentMethod()

    @Serializable
    @SerialName("bank_transfer")
    data class BankTransfer(
        val bankCode: String,
        val accountNumber: String,
        val accountHolder: String
    ) : PaymentMethod()

    @Serializable
    @SerialName("kakao_pay")
    data class KakaoPay(
        val userId: String,
        val token: String
    ) : PaymentMethod()
}

@Serializable
data class CheckoutRequest(
    val orderId: String,
    val amount: Long,
    val method: PaymentMethod
)

fun main() {
    val json = Json { prettyPrint = true }

    val request = CheckoutRequest(
        orderId = "ORD-2026-1111",
        amount = 59_000L,
        method = PaymentMethod.KakaoPay(userId = "user_123", token = "token_abc")
    )

    val encoded = json.encodeToString(request)
    println(encoded)
    /*
    {
        "orderId": "ORD-2026-1111",
        "amount": 59000,
        "method": {
            "type": "kakao_pay",
            "userId": "user_123",
            "token": "token_abc"
        }
    }
    */

    // 역직렬화 시 "type" 필드 값으로 올바른 서브클래스 자동 선택
    val decoded = json.decodeFromString<CheckoutRequest>(encoded)
    when (val method = decoded.method) {
        is PaymentMethod.KakaoPay -> println("카카오페이: ${method.userId}")
        is PaymentMethod.CreditCard -> println("신용카드: ${method.cardNumber}")
        is PaymentMethod.BankTransfer -> println("계좌이체: ${method.accountNumber}")
    }
}
```

## SerializersModule과 open class 다형성

sealed class가 아닌 open class 계층은 컴파일 타임에 모든 서브클래스를 알 수 없으므로, 반드시 `SerializersModule`에 명시적으로 등록해야 합니다.

```kotlin
import kotlinx.serialization.*
import kotlinx.serialization.json.*
import kotlinx.serialization.modules.*

@Serializable
abstract class AppEvent {
    abstract val timestamp: Long
}

@Serializable
@SerialName("screen_view")
data class ScreenViewEvent(
    override val timestamp: Long,
    val screenName: String,
    val userId: String? = null
) : AppEvent()

@Serializable
@SerialName("button_click")
data class ButtonClickEvent(
    override val timestamp: Long,
    val buttonId: String,
    val screenName: String
) : AppEvent()

@Serializable
@SerialName("purchase")
data class PurchaseEvent(
    override val timestamp: Long,
    val productId: String,
    val price: Double,
    val currency: String = "KRW"
) : AppEvent()

// 모든 서브클래스를 SerializersModule에 등록
val analyticsModule = SerializersModule {
    polymorphic(AppEvent::class) {
        subclass(ScreenViewEvent::class)
        subclass(ButtonClickEvent::class)
        subclass(PurchaseEvent::class)
    }
}

val analyticsJson = Json {
    serializersModule = analyticsModule
    classDiscriminator = "event_type"  // 기본값 "type"을 "event_type"으로 변경
    ignoreUnknownKeys = true
}

fun processEvents(rawEvents: List<String>) {
    rawEvents.forEach { raw ->
        when (val event = analyticsJson.decodeFromString<AppEvent>(raw)) {
            is ScreenViewEvent -> println("화면 조회: ${event.screenName}")
            is ButtonClickEvent -> println("버튼 클릭: ${event.buttonId}")
            is PurchaseEvent -> println("구매: ${event.productId}, ${event.price}원")
        }
    }
}
```

**모듈 합성**: 기능별로 분리한 모듈을 `+` 연산자로 합칩니다:

```kotlin
val paymentModule = SerializersModule {
    contextual(UUID::class, UuidSerializer)
    polymorphic(PaymentMethod::class) {
        subclass(PaymentMethod.CreditCard::class)
        subclass(PaymentMethod.BankTransfer::class)
        subclass(PaymentMethod.KakaoPay::class)
    }
}

// 두 모듈 합성
val combinedJson = Json {
    serializersModule = paymentModule + analyticsModule
    ignoreUnknownKeys = true
    coerceInputValues = true
}
```

## 주의사항 및 실전 팁

### 1. Json 인스턴스는 반드시 싱글턴으로

`Json { ... }` 객체 생성에는 비용이 따릅니다. 앱 전체에서 하나의 인스턴스만 사용하세요:

```kotlin
object AppJson {
    val Default = Json {
        ignoreUnknownKeys = true  // 서버에서 추가된 미知 필드 무시
        isLenient = true          // 따옴표 없는 문자열, 후행 쉼표 허용
        encodeDefaults = false    // 기본값 프로퍼티는 JSON에 미포함
        coerceInputValues = true  // 잘못된 타입/null을 기본값으로 강제 변환
        prettyPrint = false       // 프로덕션에서는 반드시 false
        serializersModule = combinedJson.serializersModule
    }
}
```

### 2. @Transient와 @Required 활용

```kotlin
@Serializable
data class CacheableResponse(
    val data: String,
    @Transient val cachedAt: Long = 0L,  // 직렬화 제외, 반드시 기본값 필요
    @Required val version: Int = 1        // 역직렬화 시 반드시 존재해야 하며 없으면 예외
)
```

### 3. JsonElement로 동적 타입 처리

API 응답의 특정 필드 타입이 가변적일 때 `JsonElement`를 활용합니다:

```kotlin
@Serializable
data class FlexibleResponse(
    val status: String,
    val payload: JsonElement
)

val response = AppJson.Default.decodeFromString<FlexibleResponse>(rawJson)
when (val p = response.payload) {
    is JsonObject -> handleAsObject(p)
    is JsonArray -> handleAsList(p)
    is JsonPrimitive -> handleAsPrimitive(p)
    JsonNull -> handleNull()
}
```

### 4. Retrofit 연동

```kotlin
// build.gradle.kts
implementation("com.jakewharton.retrofit:retrofit2-kotlinx-serialization-converter:1.0.0")

// Retrofit 설정
val retrofit = Retrofit.Builder()
    .baseUrl("https://api.example.com/")
    .addConverterFactory(
        AppJson.Default.asConverterFactory("application/json".toMediaType())
    )
    .build()
```

### 5. 버전 간 호환성 패턴

API 스키마 변경에 대비한 패턴:

```kotlin
@Serializable
data class UserProfile(
    val id: String,
    val name: String,
    val email: String? = null,         // 구버전 호환: nullable + null 기본값
    val phoneNumber: String? = null,    // 신규 필드
    @SerialName("display_name")
    val displayName: String = name     // 직렬화 이름이 바뀐 경우
)
```

## 마무리

`kotlinx.serialization`은 `@Serializable` 애노테이션으로 간단히 시작할 수 있지만, 실무에서는 다음 세 가지가 핵심입니다.

- **KSerializer**: 외부 타입이나 복잡한 직렬화 로직을 위한 커스텀 처리
- **SerializersModule**: contextual/polymorphic 등록으로 모듈화된 직렬화 구성
- **Json 설정**: ignoreUnknownKeys, coerceInputValues 등으로 API 호환성 확보

Gson에서 마이그레이션을 고민하는 팀이라면, 타입 안전성과 KMP 지원이라는 장기적인 이점을 위해 전환을 적극 검토하길 권장합니다.

## 참고 자료
- [Kotlin Serialization 공식 문서](https://kotlinlang.org/docs/serialization.html)
- [kotlinx.serialization GitHub 저장소](https://github.com/Kotlin/kotlinx.serialization)
- [Serialization Polymorphism 공식 가이드](https://kotlinlang.org/docs/serialization-polymorphism.html)
- [Gson에서 kotlinx.serialization으로 안전하게 마이그레이션하기](https://proandroiddev.com/safely-navigating-the-transition-from-gson-to-kotlinx-serialization-4b9107e557e8)
