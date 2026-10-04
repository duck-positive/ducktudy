---
layout: post
title: "Jetpack Compose Snapshot System 심화 — State, MutableState, SnapshotStateList 내부 원리"
date: 2026-10-04
categories: [android, flutter]
tags: [android, jetpack-compose, snapshot, mutablestate, recomposition, mvcc, thread-safety]
---

Jetpack Compose를 매일 사용하면서도 `mutableStateOf()`나 `mutableStateListOf()` 내부에서 무슨 일이 벌어지는지 깊이 이해하는 개발자는 많지 않습니다. 이 글에서는 Compose 런타임의 핵심 기반인 **Snapshot System**을 해부하고, 그것이 어떻게 스레드 안전성과 원자적 상태 갱신, 효율적인 리컴포지션을 동시에 가능하게 하는지 살펴봅니다.

---

## 1. Snapshot System이란 무엇인가

Jetpack Compose의 상태 관리는 겉으로는 단순해 보입니다.

```kotlin
var count by mutableStateOf(0)
```

하지만 이 한 줄 뒤에는 데이터베이스의 MVCC(Multi-Version Concurrency Control)에서 영감을 받은 정교한 **Snapshot System**이 동작하고 있습니다.

Snapshot System의 핵심 개념은 다음과 같습니다.

- **Snapshot**: 특정 시점의 모든 상태 값을 논리적으로 캡처한 뷰(View). 실제 데이터를 복사하는 것이 아니라, "이 스냅샷 ID 시점에 어떤 값이 유효한가"를 추적합니다.
- **GlobalSnapshot**: 항상 열려 있는 전역 스냅샷. Compose 프레임 시작 전 `apply()`를 호출해 변경된 상태를 반영합니다.
- **MutableSnapshot**: 고립된(isolated) 쓰기 작업을 수행할 수 있는 스냅샷. 명시적으로 `apply()`를 호출해야 GlobalSnapshot에 반영됩니다.
- **StateRecord**: 각 상태 객체 내부에 연결 리스트(Linked List) 형태로 유지되는 값 기록. 스냅샷 ID와 쌍을 이룹니다.

### 상태 읽기와 StateRecord

`mutableStateOf(0)`은 내부적으로 `SnapshotMutableStateImpl`을 생성합니다. 이 객체는 `StateRecord` 연결 리스트를 가지고 있으며, 값을 읽을 때는 현재 스냅샷의 ID보다 작거나 같은 ID를 가진 레코드 중 가장 최신 것을 반환합니다.

```
StateRecord 연결 리스트 예시:
  [snapshotId=1, value=0] -> [snapshotId=3, value=5] -> [snapshotId=7, value=10]

스냅샷 ID=4에서 읽으면 → value=5 반환 (ID 3이 4 이하이며 가장 최신)
스냅샷 ID=2에서 읽으면 → value=0 반환 (ID 1이 2 이하이며 가장 최신)
```

이 구조 덕분에 **다른 스레드가 값을 써도, 현재 스냅샷 범위 밖의 변경은 보이지 않습니다**. 별도의 Lock 없이 스레드 안전 읽기가 가능한 이유입니다.

---

## 2. 왜 Snapshot System이 필요한가

### 2-1. 리컴포지션과 변경 추적

Compose가 리컴포지션을 효율적으로 수행하려면 "어떤 컴포저블이 어떤 상태에 의존하는가"를 정확히 알아야 합니다. Snapshot System은 상태 읽기 시점에 현재 `RecomposeScope`를 구독자로 자동 등록합니다. 값이 변경되면 구독 중인 스코프만 재실행됩니다. `derivedStateOf`나 `remember(key)` 같은 최적화도 이 메커니즘 위에서 동작합니다.

### 2-2. 원자적 다중 상태 갱신

UI 상태가 여러 `MutableState` 필드로 구성될 때, 일부만 바뀐 중간 상태가 리컴포지션을 트리거하면 화면이 깜빡이거나 잘못된 UI가 순간 노출될 수 있습니다. Snapshot System은 `Snapshot.withMutableSnapshot { }` 블록을 통해 여러 상태를 하나의 트랜잭션으로 묶어 **원자적으로 커밋**할 수 있습니다.

### 2-3. 백그라운드 스레드에서의 안전한 읽기

`StateFlow`를 `collectAsStateWithLifecycle()`로 수집할 때, 실제 값 계산이 백그라운드 스레드에서 이루어질 수 있습니다. GlobalSnapshot은 특정 시점에 "동결"되므로, 백그라운드 계산 중 메인 스레드가 상태를 변경해도 현재 계산에 영향을 주지 않습니다.

---

## 3. 실제 구현 예제

### 예제 1: MutableSnapshot을 이용한 원자적 트랜잭션

여러 개의 `MutableState`를 동시에 갱신할 때 중간 상태로 인한 리컴포지션을 방지하는 코드입니다.

```kotlin
import androidx.compose.runtime.*
import androidx.compose.runtime.snapshots.Snapshot

// 상태 정의
var firstName by mutableStateOf("")
var lastName by mutableStateOf("")
var isLoading by mutableStateOf(false)

// ❌ 잘못된 방법: 각 상태 변경마다 리컴포지션 트리거
fun updateUserBad(first: String, last: String) {
    firstName = first   // 1번째 리컴포지션
    lastName = last     // 2번째 리컴포지션
    isLoading = false   // 3번째 리컴포지션
}

// ✅ 올바른 방법: 모든 변경을 하나의 트랜잭션으로 묶기
fun updateUserGood(first: String, last: String) {
    Snapshot.withMutableSnapshot {
        firstName = first
        lastName = last
        isLoading = false
    }
    // apply() 이후 단 한 번만 리컴포지션 발생
}

@Composable
fun UserProfile() {
    // LaunchedEffect 내부에서 네트워크 응답 처리 예시
    LaunchedEffect(Unit) {
        val result = fetchUserFromNetwork()
        // 백그라운드 코루틴에서 원자적으로 UI 상태 갱신
        Snapshot.withMutableSnapshot {
            firstName = result.firstName
            lastName = result.lastName
            isLoading = false
        }
    }

    Column {
        if (isLoading) {
            CircularProgressIndicator()
        } else {
            Text("$firstName $lastName")
        }
    }
}
```

`Snapshot.withMutableSnapshot { }` 블록은 내부적으로 `MutableSnapshot`을 생성하고, 블록 종료 시 `apply()`를 자동 호출합니다. GlobalSnapshot에 변경이 커밋되는 시점은 한 번이므로, 리컴포지션도 한 번만 트리거됩니다.

---

### 예제 2: SnapshotStateList와 커스텀 SnapshotMutationPolicy

`SnapshotStateList`의 내부 동작을 이해하고, 커스텀 `SnapshotMutationPolicy`로 불필요한 리컴포지션을 억제하는 예제입니다.

```kotlin
import androidx.compose.runtime.*
import androidx.compose.runtime.snapshots.SnapshotStateList

// --- SnapshotStateList 동작 이해 ---

@Composable
fun TaskList() {
    // mutableStateListOf는 내부적으로 SnapshotStateList<T>를 반환
    val tasks: SnapshotStateList<String> = remember { mutableStateListOf() }

    // SnapshotStateList는 구조적 변경(add/remove)과 요소 변경 모두를
    // 스냅샷 시스템을 통해 추적합니다.
    // ArrayList와 동일한 시간 복잡도를 가지며, 변경 시 관련 컴포저블만 리컴포지션됩니다.

    LaunchedEffect(Unit) {
        // 여러 요소를 한꺼번에 추가할 때 addAll을 사용하면
        // 내부적으로 단일 알림으로 처리됩니다.
        tasks.addAll(listOf("Task A", "Task B", "Task C"))
    }

    LazyColumn {
        items(tasks) { task ->
            TaskItem(task)
        }
    }
}

// --- 커스텀 SnapshotMutationPolicy ---

// 기본 structuralEqualityPolicy()는 `==` 비교를 사용합니다.
// 비용이 큰 equals() 연산이 있는 클래스에는 referentialEqualityPolicy()가 적합합니다.

data class HeavyData(val id: Int, val payload: ByteArray) {
    override fun equals(other: Any?): Boolean {
        if (this === other) return true
        if (other !is HeavyData) return false
        // ByteArray 비교는 O(n) 비용
        return id == other.id && payload.contentEquals(other.payload)
    }
    override fun hashCode() = id
}

@Composable
fun HeavyDataView() {
    // referentialEqualityPolicy: 동일 참조일 때만 변경 없음으로 판단
    // → equals() 호출 비용 없이 참조 비교만 수행
    var data by remember {
        mutableStateOf(
            HeavyData(1, ByteArray(1024 * 1024)),
            policy = referentialEqualityPolicy()
        )
    }

    Button(onClick = {
        // 새 객체를 할당해야 리컴포지션 트리거
        data = HeavyData(data.id + 1, ByteArray(1024 * 1024))
    }) {
        Text("Update: ${data.id}")
    }
}

// --- 커스텀 정책 구현 예시 ---

// 특정 필드만 비교하는 커스텀 정책
fun <T : Any> idOnlyPolicy(getId: (T) -> Any): SnapshotMutationPolicy<T> =
    object : SnapshotMutationPolicy<T> {
        override fun equivalent(a: T, b: T): Boolean = getId(a) == getId(b)
        // merge: MutableSnapshot 충돌 해결 (null = 충돌로 처리)
        override fun merge(previous: T, current: T, applied: T): T? = applied
    }

data class User(val id: Long, val name: String, val avatar: ByteArray)

@Composable
fun UserCard() {
    // id만 다를 때 리컴포지션, name/avatar만 다르면 리컴포지션 안 함
    var user by remember {
        mutableStateOf(
            User(1L, "Alice", ByteArray(0)),
            policy = idOnlyPolicy { it.id }
        )
    }

    Text(user.name)
}
```

---

## 4. Snapshot System의 생명주기와 GlobalSnapshot 관리

```
프레임 시작
    │
    ▼
GlobalSnapshot.sendApplyNotifications()
    │  ← 직전 프레임 이후 변경된 상태 알림
    ▼
Recomposer: 변경된 상태를 구독하는 RecomposeScope 수집
    │
    ▼
대상 컴포저블 리컴포지션 실행
    │  ← 이 시점의 상태 읽기는 현재 GlobalSnapshot ID 기준
    ▼
렌더링
```

`GlobalSnapshot`은 **스냅샷 ID를 증가**시키며 전진합니다. 새 스냅샷이 적용될 때마다 `applyObservers` 리스너가 호출되고, `Recomposer`는 이 콜백을 통해 다음 프레임에서 처리할 변경 목록을 수집합니다.

### GlobalSnapshotManager

```kotlin
// Compose UI 내부 구현 (단순화)
object GlobalSnapshotManager {
    fun ensureStarted() {
        GlobalScope.launch(Dispatchers.Main) {
            val channel = Channel<Unit>(Channel.CONFLATED)
            Snapshot.registerGlobalWriteObserver {
                channel.trySend(Unit)
            }
            for (token in channel) {
                Snapshot.sendApplyNotifications()
            }
        }
    }
}
```

백그라운드 스레드에서 상태가 바뀌면 `GlobalWriteObserver`가 호출되고, 메인 스레드의 `sendApplyNotifications()`가 리컴포저에 신호를 보냅니다. CONFLATED 채널 덕분에 연속적인 상태 변경은 하나로 합쳐져 불필요한 프레임이 발생하지 않습니다.

---

## 5. 주의사항과 실전 팁

### 5-1. SnapshotStateList vs List in MutableState

```kotlin
// ❌ 이렇게 하면 리컴포지션이 트리거되지 않을 수 있음
var items by mutableStateOf(mutableListOf<String>())
items.add("new item")  // MutableState 자체는 변경 안 됨, 참조 동일

// ✅ SnapshotStateList 사용
val items = mutableStateListOf<String>()
items.add("new item")  // 내부 변경이 스냅샷 시스템에 전달됨

// ✅ 또는 새 리스트로 교체
var items by mutableStateOf(listOf<String>())
items = items + "new item"  // 새 참조 → 리컴포지션 트리거
```

### 5-2. 코루틴에서의 상태 접근

코루틴에서 Compose 상태를 읽거나 쓸 때는 반드시 스냅샷 컨텍스트를 인지해야 합니다.

```kotlin
// ❌ withContext(Dispatchers.IO)에서 직접 읽기는 스냅샷 없는 컨텍스트일 수 있음
// ✅ snapshotFlow를 사용해 안전하게 관찰
val countFlow = snapshotFlow { count }
    .distinctUntilChanged()

// snapshotFlow는 현재 스냅샷에서 블록을 실행하고
// 관찰된 상태가 변경될 때마다 새 값을 emit합니다.
LaunchedEffect(Unit) {
    countFlow.collect { value ->
        // 백그라운드 처리
        processCount(value)
    }
}
```

### 5-3. derivedStateOf로 불필요한 리컴포지션 차단

```kotlin
// ❌ items가 바뀔 때마다 isNotEmpty 계산 + 리컴포지션
val isNotEmpty = items.isNotEmpty()

// ✅ isNotEmpty 결과가 실제로 변경될 때만 리컴포지션
val isNotEmpty by remember { derivedStateOf { items.isNotEmpty() } }
// items에 요소가 추가/제거될 때, isNotEmpty 값(true/false)이 바뀐 경우만 트리거
```

### 5-4. Snapshot.observe로 테스트 지원

```kotlin
// 테스트 코드에서 상태 변경을 동기적으로 검증
@Test
fun testStateChange() {
    val state = mutableStateOf(0)
    val changed = mutableListOf<Int>()

    val handle = Snapshot.observe(readObserver = null) { stateObjects ->
        // 쓰기 완료 후 호출됨
    }

    Snapshot.withMutableSnapshot {
        state.value = 42
    }

    Snapshot.sendApplyNotifications()
    handle.dispose()
}
```

---

## 6. 성능 관점에서의 Snapshot System

| 연산 | 비용 | 비고 |
|------|------|------|
| `mutableStateOf` 읽기 | O(k) | k = StateRecord 수, 보통 2~3개 |
| `mutableStateOf` 쓰기 | O(1) amortized | 새 StateRecord 추가 |
| `SnapshotStateList` add | O(1) amortized | ArrayList 기반 |
| `SnapshotStateList` 변경 알림 | O(구독자 수) | 보통 극소수 |
| `derivedStateOf` 계산 | O(의존 상태 수) | 의존 상태 변경 시만 재계산 |

`StateRecord` 연결 리스트는 오래된 레코드가 자동 정리(GC)되므로 일반적으로 k는 2~3 수준을 유지합니다. 극단적으로 많은 MutableSnapshot을 동시에 열어두면 리스트가 길어질 수 있으니 주의하세요.

---

## 결론

Jetpack Compose의 Snapshot System은 단순한 관찰자 패턴(Observer Pattern)을 훨씬 넘어선, MVCC에 기반한 정교한 상태 동기화 시스템입니다. 이를 이해하면:

- **불필요한 리컴포지션을 예방**하는 코드를 작성할 수 있고
- **멀티스레드 환경에서 안전한 상태 갱신** 전략을 수립할 수 있으며
- `derivedStateOf`, `snapshotFlow`, `SnapshotMutationPolicy` 같은 고급 API를 **목적에 맞게** 사용할 수 있습니다

Compose를 "마법처럼" 사용하는 것에서 벗어나, 내부 원리를 파악하면 성능 문제가 생겼을 때 훨씬 빠르게 근본 원인을 찾을 수 있습니다.

---

## 참고 자료
- [State and Jetpack Compose — Android Developers](https://developer.android.com/jetpack/compose/state)
- [androidx.compose.runtime.snapshots — API Reference](https://developer.android.com/reference/kotlin/androidx/compose/runtime/snapshots/package-summary)
- [SnapshotStateList API Reference](https://developer.android.com/reference/kotlin/androidx/compose/runtime/snapshots/SnapshotStateList)
- [Jetpack Compose Runtime Part 5: Snapshots — The Invisible State System](https://blog.devgenius.io/jetpack-compose-runtime-part-5-snapshots-the-invisible-state-system-ecbad83384e6)
