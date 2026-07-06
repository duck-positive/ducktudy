---
layout: post
title: "Kotlin Delegation 패턴 심화: by lazy부터 커스텀 위임까지 완전 정복"
date: 2026-07-06
categories: [android, flutter]
tags: [kotlin, delegation, property-delegate, by-lazy, observable, vetoable, android]
---

Kotlin의 `by` 키워드는 단순한 문법 설탕이 아니다. **프로퍼티 위임(Property Delegation)**과 **클래스 위임(Class Delegation)**이라는 두 가지 강력한 패턴을 하나의 키워드로 제공하며, 반복적인 보일러플레이트를 제거하고 재사용 가능한 동작을 캡슐화하는 Kotlin의 핵심 설계 철학을 담고 있다.

## 개념 설명

### 프로퍼티 위임 (Property Delegation)

```
val/var 프로퍼티명: 타입 by 위임객체
```

이 선언 하나로 컴파일러는 다음을 자동 생성한다:

- `val`의 경우: 위임 객체의 `getValue(thisRef, property)` 호출
- `var`의 경우: `getValue()` + `setValue(thisRef, property, value)` 호출

위임 객체는 별도의 인터페이스를 구현할 필요 없이 해당 시그니처의 연산자 함수만 제공하면 된다. 표준 라이브러리는 `ReadOnlyProperty<R, T>`와 `ReadWriteProperty<R, T>` 인터페이스를 제공해 타입 안전한 구현을 돕는다.

### 클래스 위임 (Class Delegation)

```kotlin
class A(val b: B) : SomeInterface by b
```

`SomeInterface`의 모든 멤버 구현이 `b`로 위임된다. A 본체에서 `override`한 멤버만 직접 처리하고 나머지는 자동으로 b에게 넘어간다. 이는 상속 없이 구성(Composition)으로 기능을 확장하는 Kotlin 관용 패턴이다.

---

## 왜 필요한가

Java 기반 Android 코드에서 흔히 보이는 패턴들을 떠올려보자:

```java
// SharedPreferences 읽기/쓰기 반복
String username = prefs.getString("username", "");
prefs.edit().putString("username", newValue).apply();

// null 체크 + 초기화 반복
if (mRepository == null) mRepository = new UserRepository(db);
return mRepository;

// 프로퍼티 변경 시 수동 통보
void setUsername(String value) {
    this.username = value;
    notifyDataSetChanged(); // 매번 수동 호출
}
```

Kotlin Delegation은 이 모든 패턴을 선언적으로 압축한다:

| 패턴 | 위임 방식 |
|------|----------|
| 지연 초기화 | `by lazy { }` |
| 변경 감지 | `by Delegates.observable()` |
| 입력 유효성 검사 | `by Delegates.vetoable()` |
| null 안전 초기화 | `by Delegates.notNull()` |
| 커스텀 저장소 추상화 | Custom `ReadWriteProperty` |
| 기능 확장(Decorator) | Class delegation `by` |

---

## 실제 구현 예제

### 예제 1: 표준 위임 — `lazy`, `observable`, `vetoable`, `notNull`

```kotlin
import kotlin.properties.Delegates

class UserViewModel {

    // lazy: 스레드 안전 초기화 (기본: LazyThreadSafetyMode.SYNCHRONIZED)
    // 최초 접근 시 딱 한 번만 람다 실행, 이후 캐시된 값 반환
    val heavyRepository: UserRepository by lazy {
        UserRepository(DatabaseModule.db)
    }

    // observable: 값 변경 이후(after assignment) 콜백 실행
    // handler(property, oldValue, newValue) 형태
    var username: String by Delegates.observable("") { prop, old, new ->
        if (old != new) {
            println("[${prop.name}] '$old' → '$new'")
            analyticsTracker.logPropertyChange(prop.name, new)
        }
    }

    // vetoable: 값 변경 이전(before assignment) 심사
    // handler가 false를 반환하면 대입 거부, 예외 없음, 이전 값 유지
    var age: Int by Delegates.vetoable(0) { _, _, new ->
        new in 0..150
    }

    // notNull: primitive 타입의 lateinit var
    // 초기화 전 접근 시 IllegalStateException 발생
    var sessionToken: String by Delegates.notNull()
}

fun main() {
    val vm = UserViewModel()

    vm.username = "duck"     // 출력: [username] '' → 'duck'
    vm.username = "duck"     // old == new → 콜백 내부 분기 통과

    vm.age = 25              // 허용 (0..150 범위)
    vm.age = 200             // 거부 → age 여전히 25
    vm.age = -1              // 거부 → age 여전히 25
    println(vm.age)          // 25

    // vm.sessionToken 접근 시 → IllegalStateException
}
```

**핵심 차이 정리**:

- `observable`: 변경 **후** 통보. 리스너 패턴처럼 사용. 반환값 없음.
- `vetoable`: 변경 **전** 심사. 게이트키퍼처럼 사용. `Boolean` 반환.
- `lazy`의 기본 동기화 모드는 `SYNCHRONIZED`(이중 확인 잠금)다. 단일 스레드 환경이라면 `LazyThreadSafetyMode.NONE`으로 잠금 오버헤드를 제거할 수 있다.

---

### 예제 2: 커스텀 프로퍼티 위임 — `SharedPreferences` 위임자

실무에서 가장 빛을 발하는 패턴이다. `SharedPreferences`의 읽기/쓰기 로직을 위임 객체에 봉인하고, 사용 측에서는 일반 프로퍼티처럼 접근한다.

```kotlin
import android.content.SharedPreferences
import kotlin.properties.ReadWriteProperty
import kotlin.reflect.KProperty

/**
 * SharedPreferences를 Kotlin 프로퍼티처럼 위임하는 범용 위임자.
 * 지원 타입: String, Int, Long, Float, Boolean
 */
class SharedPrefDelegate<T>(
    private val prefs: SharedPreferences,
    private val default: T,
    private val key: String? = null  // null이면 프로퍼티 이름을 키로 사용
) : ReadWriteProperty<Any?, T> {

    @Suppress("UNCHECKED_CAST")
    override fun getValue(thisRef: Any?, property: KProperty<*>): T {
        val k = key ?: property.name
        return when (default) {
            is String  -> prefs.getString(k, default) as T
            is Int     -> prefs.getInt(k, default) as T
            is Long    -> prefs.getLong(k, default) as T
            is Float   -> prefs.getFloat(k, default) as T
            is Boolean -> prefs.getBoolean(k, default) as T
            else       -> throw IllegalArgumentException("지원하지 않는 타입: ${default!!::class}")
        }
    }

    override fun setValue(thisRef: Any?, property: KProperty<*>, value: T) {
        val k = key ?: property.name
        prefs.edit().apply {
            when (value) {
                is String  -> putString(k, value)
                is Int     -> putInt(k, value)
                is Long    -> putLong(k, value)
                is Float   -> putFloat(k, value)
                is Boolean -> putBoolean(k, value)
                else       -> throw IllegalArgumentException("지원하지 않는 타입: ${value!!::class}")
            }
            apply()
        }
    }
}

// 확장 함수로 DSL처럼 활용
fun <T> SharedPreferences.delegate(default: T, key: String? = null) =
    SharedPrefDelegate(this, default, key)

// -------- 사용 예 --------
class AppSettings(prefs: SharedPreferences) {
    var isDarkMode: Boolean by prefs.delegate(false)
    var lastUserId: String  by prefs.delegate("", key = "user_id")
    var launchCount: Int    by prefs.delegate(0)
    var fontScale: Float    by prefs.delegate(1.0f)
}

class MainActivity : AppCompatActivity() {

    private val settings by lazy {
        AppSettings(getSharedPreferences("app_prefs", MODE_PRIVATE))
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        settings.launchCount++           // SharedPreferences 읽기/쓰기 자동 처리
        if (settings.isDarkMode) applyDarkTheme()
    }
}
```

`ReadWriteProperty<R, T>`를 구현하는 것만으로 위임 규약이 충족된다. `KProperty.name`을 키로 활용하므로 별도의 키 상수 파일이 필요 없고, 프로퍼티 이름 변경 시 키도 자동으로 변경된다는 점에 주의하자(마이그레이션 필요).

---

### 예제 3: 클래스 위임 — 상속 없이 Decorator 구현

```kotlin
interface Logger {
    fun log(msg: String)
    fun error(msg: String)
    fun warn(msg: String)
}

class ConsoleLogger : Logger {
    override fun log(msg: String)  = println("[LOG] $msg")
    override fun error(msg: String) = System.err.println("[ERROR] $msg")
    override fun warn(msg: String)  = println("[WARN] $msg")
}

// AnalyticsLogger는 Logger 전체를 ConsoleLogger에 위임하되,
// error()만 오버라이드해 Firebase에도 전송
class AnalyticsLogger(
    private val base: Logger = ConsoleLogger()
) : Logger by base {

    override fun error(msg: String) {
        base.error(msg)                               // 기존 동작 유지
        FirebaseCrashlytics.getInstance().log(msg)    // 추가 동작 주입
    }
    // log()와 warn()은 자동으로 base에 위임 → 보일러플레이트 제로
}

// 사용
val logger: Logger = AnalyticsLogger()
logger.log("앱 시작")          // ConsoleLogger.log() 호출
logger.error("크래시 발생!")   // ConsoleLogger.error() + Crashlytics
```

이 패턴은 `super`를 남발하지 않고 특정 메서드만 선택적으로 오버라이드할 수 있어, Kotlin 공식 문서가 상속보다 권장하는 구성(Composition over Inheritance) 방식이다.

---

## 주의사항 / 팁

### 1. `lazy` 스레드 안전성 모드 선택

| 모드 | 설명 | 적합한 상황 |
|------|------|------------|
| `SYNCHRONIZED` (기본) | 이중 확인 잠금, 멀티스레드 안전 | ViewModel, Repository 등 공유 객체 |
| `PUBLICATION` | 여러 스레드 동시 초기화 허용, 최초 완료 값 채택 | 동시 초기화가 허용되는 읽기 전용 데이터 |
| `NONE` | 잠금 없음, 단일 스레드 전용 | UI 스레드에서만 접근하는 View 관련 객체 |

### 2. `vetoable`의 조용한 거부

대입이 거부되어도 **예외가 발생하지 않는다**. 거부 사실을 호출자에게 알려야 한다면 `observable`과 조합하거나 커스텀 위임자로 예외를 던지는 것이 낫다.

```kotlin
// 거부 시 예외를 던지는 커스텀 위임자 패턴
var strictAge: Int by StrictRangeDelegate(0, 0..150, "나이는 0~150 사이여야 합니다")
```

### 3. 위임 객체 생성 시점

`by lazy { ... }`는 람다를 저장할 뿐이지만, `by SomeDelegateObject`처럼 위임 인스턴스를 직접 넘기면 **클래스 생성 시 즉시 평가**된다. 값비싼 초기화 로직을 위임 생성자 본문에 두지 말 것.

### 4. `provideDelegate` 연산자 (고급)

위임자를 프로퍼티에 바인딩하는 시점에 훅을 걸 수 있는 고급 기능이다. 바인딩 시점에 유효성 검사, 로깅, 의존성 주입이 필요한 경우 활용한다.

```kotlin
class ValidatedDelegate<T>(private val inner: ReadWriteProperty<Any?, T>) {

    operator fun provideDelegate(
        thisRef: Any?,
        prop: KProperty<*>
    ): ReadWriteProperty<Any?, T> {
        println("프로퍼티 '${prop.name}' 위임 바인딩 시점 훅")
        return inner
    }
}
```

Koin·Hilt 같은 DI 프레임워크나 Jetpack Compose의 `remember` 계열 위임이 이 방식을 내부적으로 활용한다.

### 5. Map 위임의 타입 안전성 주의

```kotlin
class Config(map: Map<String, Any?>) {
    val serverUrl: String by map
    val timeout: Int      by map  // 런타임에 ClassCastException 가능!
}
```

타입 안전성은 **런타임**에만 보장되므로, 구조화된 설정 데이터에는 `kotlinx.serialization`이나 명시적 커스텀 위임을 사용하자.

### 6. 클래스 위임과 self-type 문제

`class A : Interface by b`에서 `b` 내부가 `this.someMethod()`를 호출할 때, 그 `this`는 **A가 아닌 b 자신**이다. 위임은 상속이 아니므로 b는 A의 오버라이드된 메서드를 호출하지 않는다. 이 제약을 이해하지 못하면 예상과 다른 다형성 동작을 목격하게 된다.

---

Kotlin Delegation은 "반복되는 코드는 한 곳으로 모아라"는 원칙의 언어 차원 지원이다. `lazy` 하나만 써도 생산성이 오르지만, 커스텀 위임자와 클래스 위임까지 숙달하면 Android 코드베이스의 구조적 복잡성을 현저히 낮출 수 있다.

## 참고 자료
- [Delegated properties \| Kotlin Documentation](https://kotlinlang.org/docs/delegated-properties.html)
- [Delegation \| Kotlin Documentation](https://kotlinlang.org/docs/delegation.html)
- [Delegates \| Core API – Kotlin Programming Language](https://kotlinlang.org/api/core/kotlin-stdlib/kotlin.properties/-delegates/index.html)
