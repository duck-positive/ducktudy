---
layout: post
title: "Kotlin Serialization 심화: 다형성 직렬화·커스텀 Serializer·KMP 환경 완전 정복"
date: 2026-08-02
categories: [android, flutter]
tags: [kotlin, serialization, kotlinx-serialization, polymorphism, kmp, android, json]
---

kotlinx.serialization은 JetBrains가 공식 제공하는 Kotlin 직렬화 라이브러리로, Android·KMP·서버 환경에서 동일한 API로 JSON, CBOR, ProtoBuf 등 다양한 포맷을 처리할 수 있습니다. 단순한 `@Serializable` 어노테이션 수준을 넘어, **다형성(Polymorphic) 직렬화**와 **커스텀 Serializer** 작성을 이해하면 실무의 복잡한 API 구조도 타입 안전하게 모델링할 수 있습니다.

## kotlinx.serialization이란 무엇인가

kotlinx.serialization은 리플렉션 기반이 아니라 **컴파일 타임 코드 생성** 방식을 사용합니다. KSP(Kotlin Symbol Processing)를 통해 `@Serializable`이 붙은 클래스를 분석하고, 해당 클래스의 `KSerializer<T>`를 자동으로 생성합니다. 그 결과 런타임 비용이 거의 없으며, R8/ProGuard 환경에서도 별도 규칙 없이 안전하게 동작합니다.

지원하는 직렬화 포맷은 다음과 같습니다.

- **JSON** (`kotlinx-serialization-json`) — Android·KMP에서 가장 많이 사용
- **ProtoBuf** (`kotlinx-serialization-protobuf`) — 바이너리, 스키마 필요
- **CBOR** (`kotlinx-serialization-cbor`) — 바이너리, 스키마 불필요
- **Properties** (`kotlinx-serialization-properties`) — JVM 전용

```kotlin
// build.gradle.kts
plugins {
    kotlin("plugin.serialization") version "2.0.21"
}

dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.7.3")
}
```

## 왜 다형성(Polymorphic) 직렬화가 필요한가

실제 API 응답에는 같은 필드에 여러 타입이 들어오는 경우가 흔합니다. 예를 들어 알림 시스템에서 `type` 필드에 따라 페이로드 구조가 달라지거나, UI 컴포넌트 트리를 서버에서 내려받아 렌더링하는 경우가 대표적입니다.

```json
// 서버가 반환하는 알림 페이로드 예시
{ "type": "message", "senderId": "u123", "text": "안녕하세요" }
{ "type": "payment", "amount": 9900, "currency": "KRW" }
```

이런 구조를 `Map<String, Any?>`나 `JsonObject`로 받아 수동으로 분기하면 타입 안전성이 없고 코드가 지저분해집니다. kotlinx.serialization의 다형성 직렬화를 사용하면 클래스 계층으로 깔끔하게 표현할 수 있습니다.

## Sealed Class + 다형성 직렬화 구현

### Sealed Class를 이용한 닫힌 다형성(Closed Polymorphism)

Sealed class를 사용하면 가장 간단하게 다형성 직렬화를 구현할 수 있습니다. `@Serializable`과 `classDiscriminator`를 활용합니다.

```kotlin
import kotlinx.serialization.*
import kotlinx.serialization.json.*

@Serializable
@JsonClassDiscriminator("type")  // 타입 판별자 필드명 지정
sealed class Notification {
    abstract val id: String
}

@Serializable
@SerialName("message")  // "type" 필드에 들어갈 값
data class MessageNotification(
    override val id: String,
    val senderId: String,
    val text: String,
) : Notification()

@Serializable
@SerialName("payment")
data class PaymentNotification(
    override val id: String,
    val amount: Long,
    val currency: String,
) : Notification()

fun main() {
    val json = Json { prettyPrint = true }

    // 직렬화
    val notification: Notification = MessageNotification(
        id = "notif-001",
        senderId = "u123",
        text = "안녕하세요",
    )
    val serialized = json.encodeToString(notification)
    println(serialized)
    // {
    //   "type": "message",
    //   "id": "notif-001",
    //   "senderId": "u123",
    //   "text": "안녕하세요"
    // }

    // 역직렬화: 런타임에 올바른 서브클래스 인스턴스가 생성됨
    val jsonString = """{"type":"payment","id":"notif-002","amount":9900,"currency":"KRW"}"""
    val decoded = json.decodeFromString<Notification>(jsonString)
    check(decoded is PaymentNotification)
    println("금액: ${decoded.amount} ${decoded.currency}")
}
```

`@JsonClassDiscriminator`는 클래스 단위로 타입 판별자 필드명을 지정합니다. 전역 설정은 `Json { classDiscriminator = "type" }`으로 할 수 있습니다.

### 열린 다형성(Open Polymorphism) — 추상 클래스와 SerializersModule

서드파티 라이브러리의 클래스를 상속받거나, 모듈 간 독립적으로 서브타입을 등록해야 할 때는 `SerializersModule`을 사용합니다. 이것이 "열린(Open) 다형성"입니다.

```kotlin
import kotlinx.serialization.*
import kotlinx.serialization.json.*
import kotlinx.serialization.modules.*

// 공통 모듈: 추상 베이스만 정의
@Serializable
abstract class UiComponent {
    abstract val key: String
}

// 기능 모듈 A: 텍스트 컴포넌트
@Serializable
@SerialName("text")
data class TextComponent(
    override val key: String,
    val content: String,
    val fontSize: Int = 16,
) : UiComponent()

// 기능 모듈 B: 이미지 컴포넌트
@Serializable
@SerialName("image")
data class ImageComponent(
    override val key: String,
    val url: String,
    val aspectRatio: Float = 1.0f,
) : UiComponent()

// 모듈 등록: 각 기능 모듈이 자신의 타입을 등록
val uiModule = SerializersModule {
    polymorphic(UiComponent::class) {
        subclass(TextComponent::class)
        subclass(ImageComponent::class)
    }
}

// Json 인스턴스에 모듈 주입
val json = Json {
    serializersModule = uiModule
    prettyPrint = true
}

@Serializable
data class Screen(
    val title: String,
    @Polymorphic val components: List<UiComponent>,
)

fun main() {
    val screen = Screen(
        title = "홈 화면",
        components = listOf(
            TextComponent(key = "title", content = "환영합니다", fontSize = 24),
            ImageComponent(key = "banner", url = "https://example.com/banner.webp", aspectRatio = 16f / 9f),
        ),
    )

    val encoded = json.encodeToString(screen)
    println(encoded)

    val decoded = json.decodeFromString<Screen>(encoded)
    decoded.components.forEach { component ->
        when (component) {
            is TextComponent -> println("텍스트: ${component.content} (${component.fontSize}sp)")
            is ImageComponent -> println("이미지: ${component.url}")
        }
    }
}
```

`@Polymorphic` 어노테이션은 해당 프로퍼티가 다형적으로 직렬화되어야 함을 명시합니다. `SerializersModule`에 등록되지 않은 서브타입을 직렬화하면 `SerializationException`이 발생하므로, DI 컨테이너를 통해 모듈을 한 곳에서 조합하는 패턴이 권장됩니다.

## 커스텀 KSerializer 작성

서드파티 클래스나 특수한 직렬화 로직이 필요할 때는 `KSerializer<T>`를 직접 구현합니다. 가장 흔한 사례는 `java.util.Date`, `java.util.UUID`, 또는 서버가 특이한 포맷을 사용하는 경우입니다.

### 기본형 커스텀 Serializer — UUID를 String으로

```kotlin
import kotlinx.serialization.*
import kotlinx.serialization.descriptors.*
import kotlinx.serialization.encoding.*
import kotlinx.serialization.json.*
import java.util.UUID

object UUIDSerializer : KSerializer<UUID> {

    // 직렬화 디스크립터: "UUID"라는 이름의 String 원시 타입으로 표현
    override val descriptor: SerialDescriptor =
        PrimitiveSerialDescriptor("UUID", PrimitiveKind.STRING)

    override fun serialize(encoder: Encoder, value: UUID) {
        encoder.encodeString(value.toString())
    }

    override fun deserialize(decoder: Decoder): UUID {
        return UUID.fromString(decoder.decodeString())
    }
}

@Serializable
data class User(
    @Serializable(with = UUIDSerializer::class)
    val id: UUID,
    val name: String,
)

fun main() {
    val user = User(id = UUID.randomUUID(), name = "홍길동")
    val jsonString = Json.encodeToString(user)
    println(jsonString)
    // {"id":"550e8400-e29b-41d4-a716-446655440000","name":"홍길동"}

    val decoded = Json.decodeFromString<User>(jsonString)
    println(decoded.id.javaClass.name) // java.util.UUID
}
```

자주 쓰는 Serializer는 파일 상단에 `typealias` 패턴으로 묶거나, `@file:UseSerializers(UUIDSerializer::class)`를 사용하면 파일 전체에 적용할 수 있습니다.

### 복합형 커스텀 Serializer — Pair를 JSON 배열로

```kotlin
import kotlinx.serialization.*
import kotlinx.serialization.descriptors.*
import kotlinx.serialization.encoding.*
import kotlinx.serialization.json.*

class PairSerializer<A, B>(
    private val aSerializer: KSerializer<A>,
    private val bSerializer: KSerializer<B>,
) : KSerializer<Pair<A, B>> {

    override val descriptor: SerialDescriptor = buildClassSerialDescriptor("Pair") {
        element("first", aSerializer.descriptor)
        element("second", bSerializer.descriptor)
    }

    override fun serialize(encoder: Encoder, value: Pair<A, B>) {
        // CompositeEncoder를 사용해 구조체 형식으로 인코딩
        val composite = encoder.beginStructure(descriptor)
        composite.encodeSerializableElement(descriptor, 0, aSerializer, value.first)
        composite.encodeSerializableElement(descriptor, 1, bSerializer, value.second)
        composite.endStructure(descriptor)
    }

    override fun deserialize(decoder: Decoder): Pair<A, B> {
        val composite = decoder.beginStructure(descriptor)
        var first: A? = null
        var second: B? = null
        loop@ while (true) {
            when (val index = composite.decodeElementIndex(descriptor)) {
                CompositeDecoder.DECODE_DONE -> break@loop
                0 -> first = composite.decodeSerializableElement(descriptor, 0, aSerializer)
                1 -> second = composite.decodeSerializableElement(descriptor, 1, bSerializer)
                else -> throw SerializationException("Unknown index: $index")
            }
        }
        composite.endStructure(descriptor)
        return requireNotNull(first) to requireNotNull(second)
    }
}

@Serializable
data class Coordinate(
    @Serializable(with = DoubleDoubleSerializer::class)
    val point: Pair<Double, Double>,
    val label: String,
)

// 타입 파라미터가 고정된 편의용 object
object DoubleDoubleSerializer : KSerializer<Pair<Double, Double>> by PairSerializer(
    Double.serializer(), Double.serializer()
)

fun main() {
    val coord = Coordinate(point = 37.5665 to 126.9780, label = "서울시청")
    val jsonString = Json.encodeToString(coord)
    println(jsonString)
    // {"point":{"first":37.5665,"second":126.978},"label":"서울시청"}

    val decoded = Json.decodeFromString<Coordinate>(jsonString)
    println("위도: ${decoded.point.first}, 경도: ${decoded.point.second}")
}
```

## KMP 환경에서의 직렬화 전략

kotlinx.serialization은 Kotlin Multiplatform에서 동일한 코드로 Android, iOS, Desktop에서 동작하는 핵심 라이브러리입니다. 주의할 점은 다음과 같습니다.

**플랫폼별 expect/actual 직렬화**

`java.util.Date`처럼 플랫폼 의존적인 타입을 직렬화할 때는 `expect`/`actual`과 커스텀 Serializer를 조합합니다.

```kotlin
// commonMain: 플랫폼 독립 표현
@Serializable(with = InstantSerializer::class)
expect class PlatformInstant

// commonMain: 공통 Serializer 인터페이스
expect object InstantSerializer : KSerializer<PlatformInstant>

// androidMain: Android 구현
actual typealias PlatformInstant = java.time.Instant
actual object InstantSerializer : KSerializer<PlatformInstant> {
    override val descriptor = PrimitiveSerialDescriptor("Instant", PrimitiveKind.STRING)
    override fun serialize(encoder: Encoder, value: PlatformInstant) {
        encoder.encodeString(value.toString())
    }
    override fun deserialize(decoder: Decoder): PlatformInstant {
        return java.time.Instant.parse(decoder.decodeString())
    }
}
```

실제로는 `kotlinx-datetime` 라이브러리가 `Instant`, `LocalDate`, `LocalDateTime`에 대한 직렬화를 기본으로 제공하므로, KMP 프로젝트에서는 이를 활용하는 것이 권장됩니다.

**Json 인스턴스는 싱글톤으로**

`Json { ... }` 생성은 내부적으로 직렬화 캐시를 구성하는 비용이 있습니다. Android에서는 `object` 또는 DI(Hilt/Koin)를 통해 싱글톤으로 관리하세요.

```kotlin
// DI 모듈에서 한 번만 생성
val json = Json {
    ignoreUnknownKeys = true    // 서버 API 변경 시 앱 크래시 방지
    isLenient = true            // 따옴표 없는 문자열 허용
    coerceInputValues = true    // null 불허 필드에 null이 오면 기본값으로 처리
    encodeDefaults = false      // 기본값 프로퍼티는 JSON에서 생략
    prettyPrint = BuildConfig.DEBUG
    serializersModule = combinedModule  // 다형성 모듈 조합
}
```

## 주의사항과 실전 팁

### 1. `@Transient` vs `@kotlinx.serialization.Transient`

Java의 `transient` 키워드, `@java.beans.Transient`, kotlinx.serialization의 `@Transient`는 서로 다릅니다. kotlinx.serialization에서 특정 프로퍼티를 직렬화 대상에서 제외하려면 반드시 `@kotlinx.serialization.Transient`를 사용하고, 해당 프로퍼티에 기본값을 지정해야 합니다.

```kotlin
@Serializable
data class Session(
    val userId: String,
    val token: String,
    @Transient val isActive: Boolean = false,  // 직렬화에서 제외, 기본값 필수
)
```

### 2. 서버 API 버전 변경에 대응하는 `@SerialName` + `ignoreUnknownKeys`

서버가 필드명을 변경하거나 새 필드를 추가할 때를 대비합니다.

```kotlin
@Serializable
data class Product(
    @SerialName("product_id") val productId: String,  // snake_case → camelCase 매핑
    val name: String,
    val price: Long,
    // 서버에서 새로 추가된 필드: ignoreUnknownKeys = true 설정 시 에러 없이 무시됨
)
```

`ignoreUnknownKeys = true`는 기본값이 `false`입니다. 프로덕션 앱에서는 대부분 `true`로 설정해야 서버 API 변경 시 앱 크래시를 막을 수 있습니다.

### 3. `sealed class` 서브클래스에 `@Serializable` 누락 주의

`sealed class` 계층에서 서브클래스에 `@Serializable`을 빠뜨리면 컴파일은 통과하지만 런타임에 `SerializationException`이 발생합니다. Lint 규칙이나 별도의 컴파일 체크가 없으므로 주의가 필요합니다.

### 4. `List<@Polymorphic BaseType>` vs `List<BaseType>`

다형성 리스트를 직렬화할 때, sealed class는 `@Polymorphic` 없이도 동작하지만 추상 클래스를 사용할 경우 `@Polymorphic`을 명시하거나 `SerializersModule`의 컨텍스트 직렬화를 활성화해야 합니다.

```kotlin
// Sealed class: @Polymorphic 불필요
@Serializable
data class Container(val items: List<SealedBase>)

// Abstract class: @Polymorphic 필요
@Serializable
data class Container2(val items: List<@Polymorphic AbstractBase>)
```

### 5. 성능: `encodeToString` vs `encodeToStream`

대용량 데이터를 처리할 때는 `encodeToString`보다 `encodeToStream(outputStream)`을 사용해 중간 문자열 객체 생성을 피하세요. Android에서 네트워크 인터셉터 레이어에 적용하면 GC 압박을 줄일 수 있습니다.

## 마무리

kotlinx.serialization의 다형성 직렬화와 커스텀 Serializer를 이해하면, 복잡한 서버 API 구조를 타입 안전하게 매핑하고 KMP 프로젝트에서도 단일 코드베이스로 직렬화 로직을 유지할 수 있습니다. 핵심은 **sealed class로 닫힌 다형성**, **SerializersModule로 열린 다형성**, 그리고 **KSerializer로 완전한 제어**라는 세 가지 도구를 상황에 맞게 선택하는 것입니다.

## 참고 자료
- [Serialization | Kotlin Documentation](https://kotlinlang.org/docs/serialization.html)
- [Polymorphism — Kotlin Serialization Guide](https://github.com/Kotlin/kotlinx.serialization/blob/master/docs/polymorphism.md)
- [Custom Serializers — Kotlin Serialization Guide](https://github.com/Kotlin/kotlinx.serialization/blob/master/docs/serializers.md)
- [PolymorphicSerializer API Reference](https://kotlinlang.org/api/kotlinx.serialization/kotlinx-serialization-core/kotlinx.serialization/-polymorphic-serializer/)
