---
layout: post
title: "Kotlin Coroutines Flow 연산자 심화: combine·zip·flatMapLatest·buffer·conflate 완전 정복"
date: 2026-08-06
categories: [android, flutter]
tags: [kotlin, coroutines, flow, combine, zip, flatMapLatest, buffer, conflate, android, 비동기]
---

Kotlin Coroutines의 `Flow`는 비동기 데이터 스트림을 선언형으로 처리하는 강력한 도구입니다. 그러나 수십 가지 연산자 중 어느 것을 언제 써야 하는지, 특히 **combine과 zip의 차이**, **flatMapLatest가 왜 검색에 반드시 필요한지**, **배압(Backpressure) 상황에서 buffer와 conflate를 어떻게 구분하는지**는 경험 없이는 혼동하기 쉽습니다. 이 글에서는 실무에서 가장 중요한 다섯 연산자를 코드 예제와 함께 완전히 정복합니다.

---

## 1. 개념 설명: Flow 연산자의 두 종류

Flow 연산자는 크게 두 가지로 나뉩니다.

- **중간 연산자(Intermediate Operators)**: 새로운 Flow를 반환하는 lazy 연산자입니다. `collect()`가 호출되기 전까지 아무것도 실행하지 않습니다. `map`, `filter`, `combine`, `zip`, `flatMapLatest`, `buffer`, `conflate` 등이 여기에 속합니다.
- **종단 연산자(Terminal Operators)**: Flow를 소비하고 결과를 반환하는 `suspend` 함수입니다. `collect`, `toList`, `first`, `single`, `fold` 등이 있습니다.

중간 연산자는 체이닝(chaining)이 가능하며, 최종적으로 종단 연산자가 호출될 때 비로소 전체 파이프라인이 실행됩니다. 이 **지연 평가(Lazy Evaluation)** 특성이 Flow를 메모리 효율적으로 만드는 핵심입니다.

---

## 2. 왜 필요한가: 실무에서 직면하는 세 가지 문제

### 문제 1: 두 상태를 동시에 감시해야 할 때
사용자가 입력한 **검색어**와 선택한 **카테고리** 두 가지를 모두 감시하여 API를 호출해야 합니다. 하나만 바뀌어도 즉시 반응해야 합니다. → `combine` vs `zip`

### 문제 2: 빠른 입력에 대해 최신 값만 처리하고 싶을 때
사용자가 검색창에 글자를 빠르게 입력합니다. 매 글자마다 네트워크 API를 호출하면 낭비이고, 이전 요청이 새 요청보다 늦게 응답하면 **Race Condition**이 발생합니다. → `flatMapLatest`

### 문제 3: 생산자가 소비자보다 빠를 때 (배압, Backpressure)
실시간 센서 데이터나 소켓 이벤트를 emit하는 속도가 UI 렌더링 속도보다 훨씬 빠를 때, 중간 값들을 대기열에 쌓거나 건너뛰어야 합니다. → `buffer` / `conflate`

---

## 3. 실제 구현 예제

### 3.1 combine vs zip: 결정적인 차이

두 연산자 모두 여러 Flow를 하나로 합치지만, 동작 방식이 근본적으로 다릅니다.

**`zip`** 은 두 Flow를 **1:1 쌍(pair)**으로 묶습니다. 양쪽 모두에서 새 값이 emit되어야 결합된 값이 방출되고, 한쪽이 더 빠르면 느린 쪽을 기다립니다. 가장 짧은 Flow가 완료되면 전체 zip도 완료됩니다.

**`combine`** 은 어느 한쪽에서 새 값이 emit될 때마다 나머지의 **최신 값**과 결합하여 방출합니다. 양쪽 모두 첫 값이 도착하기 전까지만 대기하며, 이후엔 어느 한쪽이 바뀌면 즉시 반응합니다.

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking {
    val names = flowOf("Alice", "Bob", "Charlie").onEach { delay(100) }
    val scores = flowOf(95, 80).onEach { delay(150) }

    println("=== zip: 1:1 쌍 결합 ===")
    names.zip(scores) { name, score ->
        "$name: $score점"
    }.collect { println(it) }
    // Alice: 95점
    // Bob: 80점
    // Charlie는 scores가 먼저 완료되어 방출 안 됨

    println("\n=== combine: 최신값 결합 ===")
    val query = MutableStateFlow("")
    val filterType = MutableStateFlow("ALL")

    // 실무 패턴: ViewModel에서 검색어 + 필터 동시 감시
    combine(query, filterType) { q, f ->
        "query='$q', filter='$f'"
    }.take(4).collect { println(it) }

    // 시뮬레이션: 각 StateFlow를 독립적으로 변경
    launch {
        delay(50); query.value = "안드로이드"
        delay(50); filterType.value = "KOTLIN"
        delay(50); query.value = "플러터"
    }
}
```

**언제 무엇을 쓸까?**

| 상황 | 권장 연산자 |
|------|------------|
| 두 리스트를 줄 단위로 병합 (CSV 결합 등) | `zip` |
| 여러 상태 중 하나라도 바뀌면 UI 갱신 | `combine` |
| ViewModel에서 검색어 + 필터 + 정렬 동시 감시 | `combine` |
| 페이지와 사용자 정보를 쌍으로 매핑 | `zip` |

---

### 3.2 flatMapLatest: 검색 기능 구현의 정석

`flatMap` 계열에는 세 가지가 있습니다.

- **flatMapConcat**: 이전 inner Flow가 완료될 때까지 다음 것을 기다림 (순서 보장, 느림)
- **flatMapMerge**: 동시에 여러 inner Flow를 실행 (병렬, 순서 불보장, Race Condition 위험)
- **flatMapLatest**: 새 값이 오면 이전 inner Flow를 **즉시 취소**하고 새 것을 시작

검색창 구현에는 반드시 `flatMapLatest`를 써야 합니다. "안드로이드"를 치다가 "플러터"로 바꾸면, 이미 진행 중인 "안드로이드" 검색 API 호출을 취소하고 "플러터" 검색만 남겨야 하기 때문입니다. `flatMapMerge`를 사용하면 두 요청이 동시에 진행되어, 먼저 시작된 요청이 나중에 응답하면 화면에 잘못된 결과가 표시됩니다.

```kotlin
// SearchViewModel.kt
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

class SearchViewModel(
    private val repository: SearchRepository
) : ViewModel() {

    private val _query = MutableStateFlow("")
    val query: StateFlow<String> = _query.asStateFlow()

    val searchResults: StateFlow<SearchUiState> =
        _query
            .debounce(300L)           // 300ms 내 연속 입력 무시 (debouncing)
            .filter { it.length >= 2 } // 2글자 미만 쿼리 무시
            .distinctUntilChanged()    // 같은 값 반복 emit 무시
            .flatMapLatest { query ->
                // 새 query가 오면 이 flow 블록이 즉시 취소되고 새로 시작됨
                flow {
                    emit(SearchUiState.Loading)
                    try {
                        val results = repository.search(query)
                        emit(SearchUiState.Success(results))
                    } catch (e: CancellationException) {
                        throw e // 취소 예외는 반드시 재전파!
                    } catch (e: Exception) {
                        emit(SearchUiState.Error(e.message ?: "검색 실패"))
                    }
                }
            }
            .stateIn(
                scope = viewModelScope,
                started = SharingStarted.WhileSubscribed(5_000),
                initialValue = SearchUiState.Idle
            )

    fun onQueryChanged(newQuery: String) {
        _query.value = newQuery
    }
}

sealed class SearchUiState {
    object Idle : SearchUiState()
    object Loading : SearchUiState()
    data class Success(val data: List<SearchResult>) : SearchUiState()
    data class Error(val message: String) : SearchUiState()
}
```

`flatMapLatest`의 취소 메커니즘은 Kotlin 코루틴의 **구조적 동시성(Structured Concurrency)** 에 기반합니다. 새 query가 오면 이전 `flow { ... }` 블록의 코루틴 스코프에 `CancellationException`이 전달되어, 실행 중인 `repository.search(query)` suspend 함수가 즉시 취소됩니다. Retrofit을 사용한다면 내부적으로 OkHttp Call도 취소(cancel)됩니다.

> **주의**: `catch (e: Exception)`으로 모든 예외를 잡으면 `CancellationException`까지 삼켜버려 취소가 동작하지 않습니다. `CancellationException`은 반드시 `throw e`로 재전파해야 합니다.

---

### 3.3 buffer와 conflate: 배압(Backpressure) 처리

Flow는 기본적으로 **순차(sequential)** 로 동작합니다. collector가 한 값의 처리를 끝내야 producer가 다음 값을 emit합니다. 생산자가 소비자보다 훨씬 빠른 상황에서 이 동기 구조는 병목이 됩니다.

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import kotlin.system.measureTimeMillis

fun fastProducer(): Flow<Int> = flow {
    for (i in 1..5) {
        delay(100)   // 100ms마다 emit (빠른 생산자)
        println("[Producer] emit $i")
        emit(i)
    }
}

fun main() = runBlocking {
    println("=== 기본 (순차) ===")
    val t1 = measureTimeMillis {
        fastProducer().collect { value ->
            delay(300) // 300ms 처리 (느린 소비자)
            println("[Collector] processed $value")
        }
    }
    println("총 시간: ${t1}ms  ← ≈ (100+300)×5 = 2000ms\n")

    println("=== buffer: 생산자·소비자 분리 ===")
    val t2 = measureTimeMillis {
        fastProducer()
            .buffer(capacity = 10) // 별도 코루틴에서 emit, 최대 10개 대기열
            .collect { value ->
                delay(300)
                println("[Collector] processed $value")
            }
    }
    println("총 시간: ${t2}ms  ← ≈ 300×5 = 1500ms (emit이 collect를 기다리지 않음)\n")

    println("=== conflate: 최신 값만 처리 ===")
    val t3 = measureTimeMillis {
        fastProducer()
            .conflate() // 처리 중 쌓인 값들을 건너뜀, 항상 최신 값만 수신
            .collect { value ->
                delay(300)
                println("[Collector] processed $value  ← 일부 값 skip됨")
            }
    }
    println("총 시간: ${t3}ms  ← 가장 빠르지만 일부 값 유실\n")
}
```

**실행 결과 비교 (conflate)**:
```
[Producer] emit 1
[Collector] processed 1  ← 처리 시작
[Producer] emit 2        ← 처리 중 도착 → 버림
[Producer] emit 3        ← 처리 중 도착 → 버림
[Producer] emit 4        ← 처리 중 도착 → 최신값 보관
[Collector] processed 4  ← 4만 처리 (2,3 skip)
[Producer] emit 5
[Collector] processed 5
```

**buffer vs conflate 선택 기준**

| 기준 | `buffer` | `conflate` |
|------|---------|-----------|
| 값 유실 | 없음 (버퍼 용량 내) | 있음 (중간 값 skip) |
| 메모리 | 버퍼 크기만큼 사용 | 최소 (최신 1개만 유지) |
| 처리 순서 | 보장 | 중간 값 건너뜀 |
| 적합한 사례 | 금융 거래, 로그 이벤트, 순서 중요한 메시지 | GPS 위치 추적 UI, 실시간 센서 게이지, 주가 현재값 표시 |

---

## 4. 주의사항과 실전 팁

### 팁 1: combine 초기화 타이밍 — 첫 emit을 보장하라

`combine`은 모든 입력 Flow가 **최소 1개** 이상 emit한 뒤에야 첫 결합 값을 방출합니다. 일반 cold flow(`flow { ... }`)를 사용하면 첫 emit이 늦을 수 있습니다. `StateFlow`를 사용하거나 `.onStart { emit(defaultValue) }`를 추가하면 안전합니다.

```kotlin
// 위험: 두 Flow 중 하나라도 늦게 emit하면 결합이 지연됨
val result = combine(coldFlow1, coldFlow2) { a, b -> "$a + $b" }

// 안전: StateFlow는 초기값이 즉시 emit됨
val state1 = MutableStateFlow("초기값1")
val state2 = MutableStateFlow("초기값2")
val result = combine(state1, state2) { a, b -> "$a + $b" }
```

### 팁 2: flatMapLatest 안에서의 예외 처리 패턴

앞서 설명한 것처럼, `CancellationException`은 재전파해야 합니다. Kotlin 1.8+ 에서는 코루틴 취소 관련 예외 처리를 위해 `ensureActive()`나 `isActive` 확인도 좋은 방법입니다.

```kotlin
.flatMapLatest { query ->
    flow {
        emit(UiState.Loading)
        runCatching { repository.search(query) }
            .onSuccess { emit(UiState.Success(it)) }
            .onFailure { e ->
                if (e is CancellationException) throw e
                emit(UiState.Error(e.message ?: "오류"))
            }
    }
}
```

### 팁 3: flowOn과 buffer의 연산자 융합(Operator Fusion)

`flowOn`은 내부적으로 채널(Channel)을 사용하는데, 이것 자체가 일종의 버퍼입니다. `flowOn` 뒤에 `buffer`를 추가하면 Kotlin이 **연산자 융합(Operator Fusion)** 을 적용하여 두 버퍼를 하나로 합칩니다. 아래 두 코드는 동일하게 동작합니다.

```kotlin
// 명시적으로 buffer + flowOn
flow.buffer(10).flowOn(Dispatchers.IO)

// flowOn만 사용 (내부적으로 동일)
flow.flowOn(Dispatchers.IO)
```

### 팁 4: zip은 완료 조건에 주의

`zip`은 두 Flow 중 **먼저 완료되는 쪽**이 기준입니다. 한쪽이 무한 스트림(`StateFlow` 등)이어도 다른 쪽이 끝나면 zip 전체가 완료됩니다. 반드시 두 Flow의 수명(lifetime)을 고려해 설계하세요.

### 팁 5: distinctUntilChanged로 불필요한 재처리 방지

`combine`이나 `flatMapLatest` 앞에 `.distinctUntilChanged()`를 추가하면 동일한 값이 연속으로 emit될 때 중복 처리를 막을 수 있습니다. 특히 `StateFlow`는 같은 값을 set하면 emit하지 않지만, 중간 변환을 거친 결과가 같을 경우를 위해 명시적으로 추가하는 것이 좋습니다.

---

## 마치며

Kotlin Flow 연산자를 올바르게 선택하면 복잡한 비동기 로직을 선언적이고 읽기 쉬운 코드로 표현할 수 있습니다. 핵심 원칙을 정리하면 다음과 같습니다.

- **여러 스트림 결합** → `combine`(어느 쪽이 바뀌어도 반응) vs `zip`(1:1 쌍 대응)
- **최신 값만 처리** → `flatMapLatest`(이전 작업 자동 취소, 검색 UI 필수)
- **생산자 > 소비자 속도** → `buffer`(순서·값 보장) vs `conflate`(최신 값만, 빠름)

이 원칙들을 `ViewModel + StateFlow + stateIn(WhileSubscribed)` 패턴과 결합하면, 생명주기를 안전하게 관리하면서도 반응형으로 동작하는 견고한 Android 앱을 구축할 수 있습니다.

## 참고 자료
- [Kotlin 공식 문서: Flows](https://kotlinlang.org/docs/coroutines-flow.html)
- [Kotlin 공식 문서: Flow Operators](https://kotlinlang.org/docs/coroutines-flow-operators.html)
- [Android Developers: Kotlin flows on Android](https://developer.android.com/kotlin/flow)
- [flatMapLatest API Reference](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flat-map-latest.html)
