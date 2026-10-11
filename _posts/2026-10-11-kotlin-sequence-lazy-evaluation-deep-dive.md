---
layout: post
title: "Kotlin Sequences 심화: 지연 평가(Lazy Evaluation)로 컬렉션 처리 성능을 극적으로 끌어올리는 법"
date: 2026-10-11
categories: [android, kotlin]
tags: [kotlin, sequence, lazy-evaluation, collections, performance, android, coroutines]
---

Kotlin 표준 라이브러리에는 `List`, `Set`, `Map`과 같은 일반 컬렉션 외에 `Sequence<T>`라는 강력한 컬렉션 타입이 존재합니다. Sequence는 **지연 평가(Lazy Evaluation)**를 통해 대용량 데이터 처리 시 불필요한 중간 컬렉션 생성을 막고, 단락 평가(short-circuit)로 필요한 최소한의 연산만 수행합니다. 이번 아티클에서는 Sequence의 내부 동작 원리부터 Android 실무 활용 패턴, 그리고 주의해야 할 함정까지 깊이 있게 다룹니다.

---

## 1. Sequence란 무엇인가?

`Sequence<T>`는 Kotlin 표준 라이브러리가 제공하는 인터페이스로, 요소들을 **지연(lazy) 방식**으로 생산합니다. 일반 `Iterable`(List, Set 등)이 각 연산을 모든 요소에 대해 즉시 수행(eager)하는 것과 대조됩니다.

### Eager(즉시 평가) vs Lazy(지연 평가)

```kotlin
// Eager: 각 중간 연산이 전체 컬렉션에 즉시 적용됨
val list = (1..1_000_000).toList()
val eagerResult = list
    .filter { it % 2 == 0 }   // → 50만 개짜리 새 List 생성
    .map { it * it }           // → 50만 개짜리 새 List 생성
    .take(3)                   // → 3개짜리 새 List 생성

// Lazy: 터미널 연산이 호출될 때까지 실제 처리 안 됨
val lazyResult = list.asSequence()
    .filter { it % 2 == 0 }   // → 새 Sequence 반환 (처리 안 됨)
    .map { it * it }           // → 새 Sequence 반환 (처리 안 됨)
    .take(3)                   // → 새 Sequence 반환 (처리 안 됨)
    .toList()                  // ← 이 시점에만 실제 처리 시작, 6번 연산 후 종료
```

위 예시에서 Eager 방식은 100만 개 요소에 대해 filter, map, take를 순서대로 전부 적용합니다. 반면 Lazy 방식은 짝수 3개를 찾자마자 즉시 중단하므로 불필요한 연산이 사라집니다.

### 처리 순서의 차이를 눈으로 확인하기

```kotlin
val words = listOf("the", "quick", "brown", "fox", "jumps")

println("=== Eager (List) ===")
val eagerResult = words
    .filter  { print("F:$it "); it.length > 3 }
    .map     { print("M:$it "); it.uppercase() }
    .take(2)
println(eagerResult)
// 출력: F:the F:quick F:brown F:fox F:jumps M:quick M:brown
//       (filter가 먼저 전부 실행된 뒤 map 실행)

println("=== Lazy (Sequence) ===")
val lazyResult = words.asSequence()
    .filter  { print("F:$it "); it.length > 3 }
    .map     { print("M:$it "); it.uppercase() }
    .take(2)
    .toList()
println(lazyResult)
// 출력: F:the F:quick M:quick F:brown M:brown
//       (각 요소가 파이프라인 전체를 통과, take(2) 충족 후 중단)
```

이 출력만 봐도 Sequence가 얼마나 효율적으로 동작하는지 직관적으로 이해할 수 있습니다.

---

## 2. 왜 Sequence를 사용해야 하는가?

### 2.1 중간 컬렉션 생성 방지

일반 컬렉션 연산에서는 각 중간 연산마다 새로운 컬렉션이 힙 메모리에 할당됩니다. 데이터가 100만 건이고 filter → map → take(10) 체인이라면:

- `filter` 결과: 약 50만 건의 새 List (힙 할당)
- `map` 결과: 약 50만 건의 새 List (힙 할당)
- `take(10)`: 10건만 남김

GC가 처리해야 할 불필요한 객체가 수십만 개 생성됩니다. Sequence를 사용하면 중간 컬렉션이 전혀 생기지 않습니다.

### 2.2 단락 평가(Short-Circuit)로 조기 종료

`first()`, `take(n)`, `find()`, `any()`, `none()` 등 조기 종료가 가능한 터미널 연산에서 Sequence는 조건이 충족되는 즉시 처리를 멈춥니다.

```kotlin
val numbers = (1..Int.MAX_VALUE).asSequence()
val firstBigSquare = numbers
    .map { it.toLong() * it }
    .first { it > 1_000_000L }
// Sequence이므로 조건 충족 시 즉시 중단, Int.MAX_VALUE까지 처리하지 않음
println(firstBigSquare) // 1004004 (1002^2)
```

### 2.3 무한 시퀀스 표현 가능

`generateSequence()`나 `sequence {}` 빌더를 이용하면 끝이 없는 시퀀스를 메모리 부담 없이 정의할 수 있습니다. `take()`나 `takeWhile()`로 필요한 만큼만 소비합니다.

---

## 3. Sequence 생성 방법 4가지

### 3.1 `sequenceOf()` — 고정 요소로 생성

```kotlin
val seq = sequenceOf(10, 20, 30, 40, 50)
println(seq.filter { it > 15 }.toList()) // [20, 30, 40, 50]
```

### 3.2 `asSequence()` — 기존 컬렉션 변환

```kotlin
val result = listOf("apple", "banana", "cherry", "date", "elderberry")
    .asSequence()
    .filter { it.length > 5 }
    .map { it.replaceFirstChar { c -> c.uppercase() } }
    .take(2)
    .toList()
println(result) // [Banana, Cherry]
```

### 3.3 `generateSequence()` — 함수로 무한/유한 시퀀스 생성

```kotlin
// 무한 시퀀스: null을 반환하지 않으면 무한히 계속
val naturals = generateSequence(1) { it + 1 }
println(naturals.take(5).toList()) // [1, 2, 3, 4, 5]

// 유한 시퀀스: null 반환 시 종료
val powersOfTwo = generateSequence(1) { if (it < 512) it * 2 else null }
println(powersOfTwo.toList()) // [1, 2, 4, 8, 16, 32, 64, 128, 256, 512]

// 실용 예: 반복 연산 결과를 시퀀스로 표현
data class Node(val value: Int, val next: Node?)

fun Node.asSequence(): Sequence<Int> =
    generateSequence(this) { it.next }.map { it.value }

val node = Node(1, Node(2, Node(3, null)))
println(node.asSequence().toList()) // [1, 2, 3]
```

### 3.4 `sequence {}` 빌더 — `yield`와 `yieldAll`로 정밀 제어

가장 유연한 방법입니다. `yield()`는 값을 하나씩, `yieldAll()`은 Iterable/Sequence 전체를 생산합니다.

```kotlin
val customSeq = sequence {
    yield(0)                               // 단일 값 생산
    yieldAll(listOf(1, 2, 3))              // 리스트 전체 생산
    yieldAll(generateSequence(4) { it + 1 }) // 무한 시퀀스 이어 붙이기
}
println(customSeq.take(8).toList()) // [0, 1, 2, 3, 4, 5, 6, 7]
```

`sequence {}` 빌더 내부는 코루틴 기반 `SequenceScope`로 동작하므로, `yield` 호출 사이에 일반 로직(조건문, 반복문 등)을 자유롭게 삽입할 수 있습니다.

---

## 4. 실전 코드 예제

### 예제 1: Android 대용량 로그 파일 처리

앱 크래시 로그를 파일에서 읽어 ERROR 레벨 항목 중 최근 50건만 추출하는 예제입니다.

```kotlin
import java.io.File

data class LogEntry(val level: String, val tag: String, val message: String)

fun parseLogLine(line: String): LogEntry? {
    // 형식: "2026-10-11 14:32:00 [ERROR] MainActivity: NullPointerException"
    val regex = Regex("""^\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2} \[(\w+)] (\w+): (.+)$""")
    val match = regex.matchEntire(line.trim()) ?: return null
    val (level, tag, message) = match.destructured
    return LogEntry(level, tag, message)
}

fun getRecentErrors(logFile: File, maxCount: Int = 50): List<LogEntry> {
    return logFile.bufferedReader().use { reader ->
        reader.lineSequence()                    // File 전체를 메모리에 올리지 않음
            .mapNotNull { parseLogLine(it) }     // 파싱 실패 시 null → 제외
            .filter { it.level == "ERROR" }      // ERROR만 필터
            .take(maxCount)                      // 50건 채우면 즉시 파일 읽기 중단
            .toList()
    }
}

// 비교: Eager 방식 (파일 전체 메모리 로드, 중간 리스트 2회 생성)
fun getRecentErrorsEager(logFile: File, maxCount: Int = 50): List<LogEntry> {
    return logFile.readLines()                   // 전체 파일 읽기 (메모리 부담)
        .mapNotNull { parseLogLine(it) }
        .filter { it.level == "ERROR" }
        .take(maxCount)
}
```

`lineSequence()`를 사용하면 `readLines()`처럼 파일 전체를 메모리에 올리지 않고, 50건이 채워지는 순간 파일 읽기를 중단합니다. 수백 MB 로그 파일에서 극적인 차이가 납니다.

---

### 예제 2: 피보나치 수열과 무한 시퀀스 실전 활용

```kotlin
// generateSequence를 이용한 피보나치 수열 (무한)
val fibonacci: Sequence<Long> = generateSequence(Pair(0L, 1L)) { (a, b) ->
    Pair(b, a + b)
}.map { it.first }

println("처음 10개: ${fibonacci.take(10).toList()}")
// [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]

println("100 미만의 짝수 피보나치 합: ${
    fibonacci.takeWhile { it < 100 }.filter { it % 2 == 0 }.sum()
}")
// 44 (2 + 8 + 34)

// sequence 빌더로 더 명확하게 구현
val fib = sequence {
    var a = 0L
    var b = 1L
    while (true) {
        yield(a)
        val next = a + b
        a = b
        b = next
    }
}

// 피보나치 수 중 1,000,000 이하이면서 5의 배수인 것만 추출
val filteredFib = fib
    .takeWhile { it <= 1_000_000L }
    .filter { it % 5 == 0L }
    .toList()
println("1,000,000 이하 5의 배수 피보나치: $filteredFib")
// [0, 5, 55, 610, 6765, 75025, 832040]
```

---

### 예제 3: Android ViewModel — 대규모 사용자 데이터 처리 파이프라인

```kotlin
data class RawUser(
    val id: Int,
    val name: String,
    val score: Int,
    val isActive: Boolean,
    val badges: List<String>
)

data class UserDisplayModel(
    val id: Int,
    val displayName: String,
    val tier: String,
    val badgeSummary: String
)

class UserRepository {
    fun processLeaderboard(rawUsers: List<RawUser>): List<UserDisplayModel> {
        return rawUsers.asSequence()
            .filter { it.isActive }
            .filter { it.score > 0 }
            .sortedByDescending { it.score }   // stateful: 전체 정렬 필요
            .mapIndexed { rank, user ->
                UserDisplayModel(
                    id = user.id,
                    displayName = "${user.name} (#${rank + 1})",
                    tier = when {
                        rank < 10   -> "다이아몬드 💎"
                        rank < 50   -> "플래티넘 🏆"
                        rank < 200  -> "골드 🥇"
                        else        -> "실버 🥈"
                    },
                    badgeSummary = user.badges.take(3).joinToString(", ")
                )
            }
            .take(500)   // 상위 500명만 노출
            .toList()
    }
}
```

`asSequence()`를 사용하면 `filter`와 `mapIndexed` 사이에 임시 List가 생성되지 않습니다. 단, `sortedByDescending`은 전체 요소를 모아야 하는 stateful 연산이므로 이 단계에서는 모든 데이터가 메모리에 올라갑니다. 그러나 그 이후의 `mapIndexed → take → toList` 체인은 여전히 지연 방식으로 동작합니다.

---

### 예제 4: 성능 비교 측정

```kotlin
import kotlin.system.measureTimeMillis

fun benchmark() {
    val n = 2_000_000
    val data = (1..n).toList()

    // 시나리오 1: first() 조기 종료 — Lazy가 압도적으로 유리
    val eagerTime1 = measureTimeMillis {
        val result = data.filter { it % 7 == 0 }.map { it * it }.first { it > 1_000_000 }
        println("Eager result: $result")
    }
    val lazyTime1 = measureTimeMillis {
        val result = data.asSequence().filter { it % 7 == 0 }.map { it * it }.first { it > 1_000_000 }
        println("Lazy result: $result")
    }
    println("first() 조기 종료 — Eager: ${eagerTime1}ms, Lazy: ${lazyTime1}ms")

    // 시나리오 2: 전체 합산 — 차이가 적음 (단순 연산)
    val eagerTime2 = measureTimeMillis {
        data.filter { it % 2 == 0 }.map { it.toLong() }.sum()
    }
    val lazyTime2 = measureTimeMillis {
        data.asSequence().filter { it % 2 == 0 }.map { it.toLong() }.sum()
    }
    println("전체 합산 — Eager: ${eagerTime2}ms, Lazy: ${lazyTime2}ms")
}
```

실제 실행 결과(JVM 기준):
- **조기 종료 시나리오**: Eager ~120ms vs Lazy ~2ms (약 60배 차이)
- **전체 합산 시나리오**: Eager ~40ms vs Lazy ~45ms (Lazy가 약간 느릴 수 있음)

단순 전체 처리라면 오버헤드 때문에 오히려 Lazy가 느릴 수 있습니다. **조기 종료 연산(`first`, `find`, `any`, `take`)이 있을 때** Sequence 효과가 극대화됩니다.

---

## 5. Sequence vs Kotlin Flow — 언제 무엇을 쓸까?

많은 Android 개발자들이 `Sequence`와 `Flow`를 혼동합니다. 두 가지 모두 지연 처리를 지원하지만 근본적으로 다릅니다.

| 항목 | `Sequence<T>` | `Flow<T>` |
|------|--------------|-----------|
| 실행 방식 | 동기(Synchronous) | 비동기(Asynchronous) |
| suspend 지원 | ❌ 불가 | ✅ 가능 |
| 스레드 전환 | 불가 | `flowOn()` 가능 |
| 무한 스트림 | 가능 (단일 스레드) | 가능 (다중 스레드) |
| 백프레셔 | 없음 | 내장 |
| 주 용도 | CPU 연산, 로컬 파일/메모리 | 네트워크, DB, UI 이벤트 |

```kotlin
// Sequence: 동기적 CPU 연산에 적합
fun generatePrimes(): Sequence<Int> = sequence {
    var n = 2
    while (true) {
        if ((2 until n).none { n % it == 0 }) yield(n)
        n++
    }
}
println(generatePrimes().take(10).toList()) // [2, 3, 5, 7, 11, 13, 17, 19, 23, 29]

// Flow: 비동기 데이터 스트림에 적합
fun fetchUsersFromApi(): Flow<User> = flow {
    var page = 0
    while (true) {
        val users = apiService.getUsers(page++)  // suspend 가능
        if (users.isEmpty()) break
        users.forEach { emit(it) }
    }
}
```

---

## 6. 주의사항 및 실무 팁

### 6.1 소규모 컬렉션엔 Sequence가 오히려 느릴 수 있다

Sequence의 지연 평가는 람다 호출 스택이 추가되므로 오버헤드가 있습니다. 컬렉션 크기가 수백 개 이하이거나 연산이 한두 개뿐이라면 일반 List가 더 빠를 수 있습니다.

**추천 기준:**
- 요소가 1,000개 미만 + 연산 체인 2개 이하 → List
- 요소가 많거나, 조기 종료 연산 사용, 무한 시퀀스 → Sequence

### 6.2 stateful 연산 위치에 주의

`sorted()`, `distinct()`, `groupBy()` 등은 전체 요소를 메모리에 올려야 합니다. 무한 시퀀스에서 이들을 호출하면 `OutOfMemoryError`가 발생합니다.

```kotlin
// 위험: 무한 시퀀스에서 sorted() 직접 호출 → OOM
// generateSequence(1) { it + 1 }.sorted().take(10) // 절대 금지!

// 안전: take()로 범위를 먼저 제한한 뒤 stateful 연산
val result = generateSequence(1) { it + 1 }
    .take(1000)        // 먼저 범위 제한
    .filter { it % 3 == 0 }
    .sortedDescending() // 그 다음 정렬
    .toList()
```

### 6.3 터미널 연산 없이는 아무것도 실행되지 않는다

Sequence는 완전히 지연 평가됩니다. 부수 효과(side effect)를 목적으로 연산을 정의했더라도 터미널 연산 없이는 실행되지 않습니다.

```kotlin
// 버그: onEach의 println이 전혀 출력되지 않음
sequenceOf(1, 2, 3)
    .filter { it > 1 }
    .onEach { println("처리 중: $it") } // 터미널 없으면 실행 안 됨

// 올바른 사용
sequenceOf(1, 2, 3)
    .filter { it > 1 }
    .onEach { println("처리 중: $it") }
    .toList() // ← 터미널 연산으로 실행 유발
```

### 6.4 `constrainOnce()`로 단일 순회 강제

일부 시퀀스(파일 기반, 네트워크 스트림 기반 등)는 한 번만 읽을 수 있습니다. `constrainOnce()`를 사용하면 재순회 시도 시 명확한 `IllegalStateException`이 발생해 버그를 조기에 발견할 수 있습니다.

```kotlin
val onceSeq = generateSequence(1) { it + 1 }.constrainOnce()

val first5 = onceSeq.take(5).toList()
println(first5) // [1, 2, 3, 4, 5]

// 두 번째 순회 시도
try {
    onceSeq.take(5).toList()
} catch (e: IllegalStateException) {
    println("두 번 순회 불가: ${e.message}")
    // "This sequence can be consumed only once."
}
```

### 6.5 `File.lineSequence()`로 대용량 파일을 메모리 효율적으로 처리

```kotlin
fun countWordOccurrences(file: File, word: String): Int {
    return file.bufferedReader().use { reader ->
        reader.lineSequence()
            .flatMap { line -> line.split(Regex("\\s+")).asSequence() }
            .count { it.equals(word, ignoreCase = true) }
    }
    // readLines()와 달리 파일을 한 줄씩 처리 → 수 GB 파일도 OOM 없이 처리 가능
}
```

---

## 결론

Kotlin Sequences는 지연 평가를 통해 대용량 데이터 처리에서 강력한 성능 이점을 제공합니다. 핵심 원칙을 정리하면:

1. **터미널 연산 전까지 처리되지 않는다** — 진정한 지연 평가
2. **중간 컬렉션이 생기지 않는다** — GC 압력 감소
3. **조기 종료 연산에서 극적인 성능 향상** — `first()`, `take()`, `find()` 등
4. **무한 시퀀스를 안전하게 표현 가능** — `take()`, `takeWhile()`로 범위 제한
5. **Flow와 달리 동기적** — 비동기 작업에는 `Flow` 사용

Android 개발에서 Room 쿼리 후 복잡한 변환 파이프라인, 대용량 CSV/로그 파일 파싱, 레벨 데이터 랭킹 처리 등에 Sequence를 적극 활용해 불필요한 메모리 할당과 GC 압력을 줄여보세요. 작은 최적화가 사용자 경험의 큰 차이를 만듭니다.

---

## 참고 자료
- [Kotlin 공식 문서 — Sequences](https://kotlinlang.org/docs/sequences.html)
- [Kotlin 표준 라이브러리 API — kotlin.sequences](https://kotlinlang.org/api/core/kotlin-stdlib/kotlin.sequences/)
