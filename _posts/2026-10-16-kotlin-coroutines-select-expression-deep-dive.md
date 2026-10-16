---
layout: post
title: "Kotlin Coroutines select 표현식 심화: 다중 비동기 소스 경쟁 처리와 우선순위 제어 완전 정복"
date: 2026-10-16
categories: [android, flutter]
tags: [kotlin, coroutines, select, channel, async, concurrency]
---

코루틴을 어느 정도 다뤄본 개발자라면 `async/await`, `Flow`, `Channel`에는 익숙할 것이다. 그러나 **여러 비동기 소스 중 가장 먼저 완료되는 것을 선택**해야 하는 상황에서 어떻게 처리할지 막막했던 경험이 있을 것이다. 바로 이 문제를 우아하게 해결해 주는 것이 `select` 표현식이다.

## 1. `select` 표현식이란?

`select`는 `kotlinx.coroutines`가 제공하는 특별한 코루틴 빌더로, **여러 일시 중단(suspend) 연산을 동시에 대기**하다가 **가장 먼저 준비된 하나의 절(clause)만 실행**하고 결과를 반환한다.

```kotlin
val result = select<String> {
    deferredA.onAwait { "A 완료: $it" }
    deferredB.onAwait { "B 완료: $it" }
    onTimeout(2000L) { "시간 초과" }
}
```

위 코드에서 `select`는 `deferredA`, `deferredB` 중 먼저 완료되는 것의 결과를 반환한다. 2초 안에 아무것도 완료되지 않으면 타임아웃 블록이 실행된다.

### `select`가 지원하는 절(clause) 목록

| 대상 | 일시 중단 함수 | select 절 |
|---|---|---|
| `Job` | `join()` | `onJoin` |
| `Deferred<T>` | `await()` | `onAwait` |
| `SendChannel<E>` | `send()` | `onSend` |
| `ReceiveChannel<E>` | `receive()` | `onReceive` |
| `ReceiveChannel<E>` | `receiveCatching()` | `onReceiveCatching` |
| (없음) | `delay()` | `onTimeout` |

---

## 2. 왜 `select`가 필요한가?

### 문제 상황: 두 API 중 빠른 쪽 우선 사용

분산 시스템이나 마이크로서비스 환경에서는 **동일한 데이터를 서로 다른 엔드포인트에서 제공**하는 경우가 흔하다. 예를 들어 캐시 서버와 원본 서버가 각각 응답을 줄 수 있을 때, 더 빨리 오는 쪽을 사용하고 싶다면 어떻게 해야 할까?

단순하게 구현한다면:

```kotlin
// 순차 실행 — 느리다
val cacheResult = cacheApi.getData(id) // 완료될 때까지 기다림
val originResult = originApi.getData(id) // 캐시 실패 시에만 호출
```

이 방식은 캐시가 느리거나 실패할 경우 불필요한 대기가 발생한다.

`async`를 두 개 띄워도 문제가 있다:

```kotlin
val cacheDeferred = async { cacheApi.getData(id) }
val originDeferred = async { originApi.getData(id) }
// 어느 쪽이 먼저 완료됐는지 어떻게 알 수 있을까?
val result = ??? // 첫 번째 완료된 것을 받아야 한다
```

바로 여기서 `select`가 빛을 발한다.

---

## 3. 실제 구현 예제

### 예제 1: 다중 API 경쟁 — 가장 빠른 응답 사용

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.selects.select

data class UserProfile(val id: String, val name: String, val source: String)

// 캐시 서버에서 조회 (일반적으로 빠름)
suspend fun fetchFromCache(userId: String): UserProfile {
    delay(50L + (0..200L).random()) // 50~250ms
    return UserProfile(userId, "홍길동 (캐시)", "cache")
}

// 원본 DB에서 조회 (신뢰할 수 있지만 느릴 수 있음)
suspend fun fetchFromOrigin(userId: String): UserProfile {
    delay(100L + (0..300L).random()) // 100~400ms
    return UserProfile(userId, "홍길동 (원본)", "origin")
}

suspend fun fetchUserFastest(userId: String): UserProfile = coroutineScope {
    val cacheDeferred = async { fetchFromCache(userId) }
    val originDeferred = async { fetchFromOrigin(userId) }

    val winner = select<UserProfile> {
        cacheDeferred.onAwait { profile ->
            println("[select] 캐시 서버가 먼저 응답: ${profile.source}")
            profile
        }
        originDeferred.onAwait { profile ->
            println("[select] 원본 서버가 먼저 응답: ${profile.source}")
            profile
        }
    }

    // 패배한 coroutine은 명시적으로 취소해야 리소스 낭비를 막는다
    cacheDeferred.cancel()
    originDeferred.cancel()

    winner
}

fun main() = runBlocking {
    repeat(5) { i ->
        println("--- 시도 #${i + 1} ---")
        val profile = fetchUserFastest("user_001")
        println("최종 사용 소스: ${profile.source}\n")
    }
}
```

**실행 결과 예시:**
```
--- 시도 #1 ---
[select] 캐시 서버가 먼저 응답: cache
최종 사용 소스: cache

--- 시도 #2 ---
[select] 원본 서버가 먼저 응답: origin
최종 사용 소스: origin
```

랜덤 지연 덕분에 캐시와 원본 서버 중 어느 쪽이 먼저 응답할지 매번 달라진다. `select`는 이를 완전히 투명하게 처리한다.

---

### 예제 2: 채널 Round-Robin — 여러 생산자로부터 공정하게 수신

다음은 `select`의 **편향(bias)** 문제를 해결하기 위해 `selectUnbiased`를 사용하는 예제다. 기본 `select`는 여러 절이 동시에 준비됐을 때 항상 **첫 번째 절을 우선** 선택한다. 여러 채널을 공정하게 처리하려면 `selectUnbiased`를 써야 한다.

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*
import kotlinx.coroutines.selects.selectUnbiased

fun CoroutineScope.sensor(name: String, interval: Long): ReceiveChannel<String> = produce {
    var count = 0
    while (true) {
        delay(interval)
        send("[$name] 측정값: ${++count}")
    }
}

fun main() = runBlocking {
    val tempSensor = sensor("온도", 100L)   // 100ms마다 전송
    val pressureSensor = sensor("기압", 100L) // 100ms마다 전송
    val humiditySensor = sensor("습도", 100L) // 100ms마다 전송

    val tally = mutableMapOf("온도" to 0, "기압" to 0, "습도" to 0)

    // biased select를 쓰면 온도 센서가 거의 항상 먼저 선택된다
    // selectUnbiased는 무작위로 선택해 공정성을 보장한다
    repeat(30) {
        val reading = selectUnbiased<String> {
            tempSensor.onReceive { value ->
                tally["온도"] = tally["온도"]!! + 1
                value
            }
            pressureSensor.onReceive { value ->
                tally["기압"] = tally["기압"]!! + 1
                value
            }
            humiditySensor.onReceive { value ->
                tally["습도"] = tally["습도"]!! + 1
                value
            }
        }
        println(reading)
    }

    println("\n=== 센서별 선택 횟수 ===")
    tally.forEach { (sensor, count) ->
        println("$sensor: $count회")
    }

    coroutineContext.cancelChildren()
}
```

**예상 출력 (selectUnbiased 사용 시):**
```
[온도] 측정값: 1
[기압] 측정값: 1
[습도] 측정값: 1
...

=== 센서별 선택 횟수 ===
온도: 10회
기압: 10회
습도: 10회
```

`select` (편향)를 쓰면 온도 센서가 20회 이상 선택되는 반면, `selectUnbiased`는 거의 균등하게 분배한다.

---

### 예제 3: 채널 닫힘 처리 + 타임아웃 결합

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*
import kotlinx.coroutines.selects.select

fun CoroutineScope.dataStream(): ReceiveChannel<Int> = produce {
    for (i in 1..5) {
        delay(200L)
        send(i)
    }
    // 자동 닫힘
}

fun main() = runBlocking {
    val stream = dataStream()

    while (true) {
        val result = select<String> {
            stream.onReceiveCatching { channelResult ->
                when {
                    channelResult.isSuccess ->
                        "데이터 수신: ${channelResult.getOrThrow()}"
                    channelResult.isClosed ->
                        "스트림이 닫혔습니다"
                    else ->
                        "오류: ${channelResult.exceptionOrNull()?.message}"
                }
            }
            onTimeout(500L) {
                "타임아웃: 500ms 내 데이터 없음"
            }
        }

        println(result)
        if (result == "스트림이 닫혔습니다") break
    }
}
```

**출력:**
```
데이터 수신: 1
데이터 수신: 2
데이터 수신: 3
데이터 수신: 4
데이터 수신: 5
스트림이 닫혔습니다
```

`onReceiveCatching`은 채널이 닫혀도 예외를 던지지 않고 `ChannelResult`를 반환하므로, 채널 종료를 깔끔하게 처리할 수 있다.

---

## 4. 주의사항과 실전 팁

### ⚠️ 패배한 코루틴은 반드시 취소하라

`select`는 하나의 절만 실행하고 반환하지만, **나머지 `async` 코루틴들은 계속 실행 중**이다. 메모리 누수와 불필요한 네트워크 호출을 막으려면 명시적으로 취소해야 한다.

```kotlin
val deferredA = async { ... }
val deferredB = async { ... }

val result = select<String> {
    deferredA.onAwait { it }
    deferredB.onAwait { it }
}

// ✅ 반드시 취소!
deferredA.cancel()
deferredB.cancel()
```

`coroutineScope` 내에서 실행하면 스코프 종료 시 자동으로 취소되지만, 명시적 취소가 더 안전하다.

### ⚠️ 편향(Bias)을 이해하고 의도적으로 활용하라

```kotlin
select<String> {
    highPriorityChannel.onReceive { "우선순위 높음: $it" } // 항상 먼저 시도됨
    lowPriorityChannel.onReceive { "우선순위 낮음: $it" }
}
```

기본 `select`는 첫 번째 절을 우선시하므로 **의도적인 우선순위 큐**를 구현할 때 유용하다. 공정한 처리가 필요하면 `selectUnbiased`를 쓰자.

### ⚠️ `select`는 실험적(Experimental) API다

`kotlinx.coroutines`에서 `select`는 아직 실험적 API로 분류되어 있어, 마이너 버전 업데이트에서도 파괴적 변경이 일어날 수 있다. 프로덕션에 사용할 때는 의존성 버전을 고정하고, 업그레이드 시 변경 로그를 반드시 확인하자.

```kotlin
// build.gradle.kts
@OptIn(ExperimentalCoroutinesApi::class) // 컴파일 경고 억제
suspend fun mySelectFunc() { ... }
```

### ✅ `onTimeout`으로 데드라인 보장

원격 API 호출에서 전체 응답 시간에 상한선을 두려면 `onTimeout`을 활용하자:

```kotlin
val result = select<ApiResult> {
    apiDeferred.onAwait { ApiResult.Success(it) }
    onTimeout(3000L) { ApiResult.Timeout }
}
```

`withTimeout`은 예외를 던지지만, `select` + `onTimeout`은 정상적인 값을 반환하므로 예외 처리 없이 간결하게 작성할 수 있다.

### ✅ `switchMap` 패턴 — 최신 요청만 처리

사용자가 검색어를 빠르게 입력할 때 이전 검색 결과를 무시하고 최신 요청만 처리하고 싶다면:

```kotlin
suspend fun CoroutineScope.switchMapSearch(
    queries: ReceiveChannel<String>
): ReceiveChannel<List<Result>> = produce {
    var currentSearch: Deferred<List<Result>>? = null

    while (true) {
        select<Unit> {
            queries.onReceive { query ->
                currentSearch?.cancel()
                currentSearch = async { searchApi.search(query) }
            }
            currentSearch?.onAwait { results ->
                send(results)
                currentSearch = null
            }
        }
    }
}
```

---

## 5. `select` vs 다른 패턴 비교

| 상황 | 권장 패턴 |
|---|---|
| 두 API 중 빠른 쪽 선택 | `select` + `onAwait` |
| 여러 채널 중 데이터 있는 쪽 수신 | `select` + `onReceive` |
| 공정한 다중 채널 처리 | `selectUnbiased` + `onReceive` |
| 단순 타임아웃 | `withTimeout { }` |
| 채널 닫힘 포함 처리 | `select` + `onReceiveCatching` |
| Flow 경쟁 | `merge()` 또는 `Flow.raceWith()` |

---

## 마무리

`select` 표현식은 Kotlin 코루틴의 **숨겨진 강점** 중 하나다. 단순한 순차 실행이나 병렬 실행을 넘어, **여러 비동기 소스 중 가장 먼저 준비된 것을 선택하는 경쟁(race) 패턴**을 간결하고 읽기 쉬운 코드로 표현할 수 있게 해준다.

핵심 요약:
- `select`는 여러 절 중 **가장 먼저 준비된 하나**만 실행한다.
- `onAwait`, `onReceive`, `onSend`, `onTimeout`을 절로 사용할 수 있다.
- 기본 `select`는 **편향(bias)**되어 있으며, 공정성이 필요하면 `selectUnbiased`를 쓴다.
- **패배한 코루틴은 명시적으로 취소**해야 리소스 누수가 없다.
- 실험적 API이므로 버전 업그레이드 시 변경 로그를 확인하자.

## 참고 자료
- [Kotlin 공식 문서: select expression](https://kotlinlang.org/docs/select-expression.html)
- [kotlinx.coroutines API 레퍼런스: select](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.selects/select.html)
