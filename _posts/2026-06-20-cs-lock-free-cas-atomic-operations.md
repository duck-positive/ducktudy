---
layout: post
title: "Lock-Free 자료구조와 CAS: 락 없이 동시성을 정복하는 법"
date: 2026-06-20
categories: [cs, computer-science]
tags: [lock-free, cas, compare-and-swap, concurrency, atomic, aba-problem, java, kotlin]
---

## 왜 락(Lock)은 느린가?

멀티코어 시대에 동시성(concurrency)은 피할 수 없습니다. 전통적인 방법은 **뮤텍스(Mutex)**나 **세마포어(Semaphore)** 같은 락을 사용해 공유 자원을 보호하는 것입니다. 하지만 락에는 치명적인 단점이 있습니다.

- **컨텍스트 스위칭(Context Switching)**: 락을 얻지 못한 스레드는 블록(block)되어 OS가 다른 스레드로 전환합니다. 이 전환 비용은 수천 나노초에 달합니다.
- **데드락(Deadlock)**: 둘 이상의 스레드가 서로의 락을 기다리면 영원히 멈춥니다.
- **우선순위 역전(Priority Inversion)**: 낮은 우선순위 스레드가 락을 쥔 상태에서 선점되면, 높은 우선순위 스레드도 기다려야 합니다.
- **컨보이 효과(Convoy Effect)**: 느린 스레드 때문에 빠른 스레드들이 줄을 서서 기다립니다.

이 모든 문제를 해결하는 접근법이 **Lock-Free(비차단) 프로그래밍**입니다.

---

## CAS(Compare-And-Swap)란?

**CAS**는 현대 CPU가 하드웨어 수준에서 지원하는 원자적(atomic) 연산입니다. 동작은 단순합니다:

```
CAS(memory_location, expected_value, new_value):
  if *memory_location == expected_value:
    *memory_location = new_value
    return SUCCESS
  else:
    return FAIL (현재 값 반환)
```

x86에서는 `LOCK CMPXCHG` 명령어, ARM에서는 `LDREX/STREX` 쌍으로 구현됩니다. 핵심은 **읽기-비교-쓰기가 단 하나의 불가분 연산**으로 이루어진다는 점입니다. 운영체제 개입 없이 하드웨어가 동시성을 보장합니다.

### Lock-Free의 정의

- **Wait-Free**: 모든 스레드가 유한 시간 내에 반드시 완료됩니다.
- **Lock-Free**: 시스템 전체로 보면 항상 일부 스레드가 진전(progress)합니다. 개별 스레드는 기아(starvation) 상태가 될 수 있지만, 전체적으로 교착 상태는 없습니다.
- **Obstruction-Free**: 방해받지 않는 스레드는 유한 시간 내에 완료됩니다.

---

## 실제 구현 예제

### 예제 1: Java로 구현하는 Lock-Free 카운터와 스택

```java
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.AtomicReference;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.CountDownLatch;

// ── Lock-Free 카운터 ──────────────────────────────────────────
class LockFreeCounter {
    private final AtomicInteger count = new AtomicInteger(0);

    public void increment() {
        // CAS 루프: 성공할 때까지 재시도
        int current, next;
        do {
            current = count.get();
            next = current + 1;
        } while (!count.compareAndSet(current, next));
        // 사실 AtomicInteger.incrementAndGet()이 내부적으로 동일하게 동작
    }

    public int get() { return count.get(); }
}

// ── Treiber 스택 (Lock-Free Stack) ───────────────────────────
class TreiberStack<T> {
    private static class Node<T> {
        final T value;
        final Node<T> next;
        Node(T v, Node<T> n) { value = v; next = n; }
    }

    private final AtomicReference<Node<T>> top = new AtomicReference<>(null);

    public void push(T value) {
        Node<T> newNode, currentTop;
        do {
            currentTop = top.get();
            newNode = new Node<>(value, currentTop);
        } while (!top.compareAndSet(currentTop, newNode));
        // CAS 실패 = 다른 스레드가 top을 변경했음 → 재시도
    }

    public T pop() {
        Node<T> currentTop, newTop;
        do {
            currentTop = top.get();
            if (currentTop == null) return null; // 빈 스택
            newTop = currentTop.next;
        } while (!top.compareAndSet(currentTop, newTop));
        return currentTop.value;
    }

    public boolean isEmpty() { return top.get() == null; }
}

// ── 벤치마크 ─────────────────────────────────────────────────
public class LockFreeDemo {
    public static void main(String[] args) throws InterruptedException {
        int threadCount = 8;
        int opsPerThread = 100_000;

        // 카운터 테스트
        LockFreeCounter counter = new LockFreeCounter();
        ExecutorService pool = Executors.newFixedThreadPool(threadCount);
        CountDownLatch latch = new CountDownLatch(threadCount);

        long start = System.nanoTime();
        for (int i = 0; i < threadCount; i++) {
            pool.submit(() -> {
                for (int j = 0; j < opsPerThread; j++) counter.increment();
                latch.countDown();
            });
        }
        latch.await();
        long elapsed = System.nanoTime() - start;

        System.out.printf("Lock-Free Counter 최종값: %d (기대값: %d)%n",
            counter.get(), (long) threadCount * opsPerThread);
        System.out.printf("소요 시간: %.2f ms%n", elapsed / 1_000_000.0);

        // 스택 테스트
        TreiberStack<Integer> stack = new TreiberStack<>();
        CountDownLatch latch2 = new CountDownLatch(threadCount);

        for (int i = 0; i < threadCount; i++) {
            final int id = i;
            pool.submit(() -> {
                for (int j = 0; j < 1000; j++) stack.push(id * 1000 + j);
                latch2.countDown();
            });
        }
        latch2.await();

        int poppedCount = 0;
        while (!stack.isEmpty()) { stack.pop(); poppedCount++; }
        System.out.printf("Treiber Stack: push %d개, pop %d개%n",
            threadCount * 1000, poppedCount);

        pool.shutdown();
    }
}
```

### 예제 2: Kotlin으로 구현하는 ABA 문제와 해결책

```kotlin
import java.util.concurrent.atomic.AtomicReference
import java.util.concurrent.atomic.AtomicStampedReference

// ── ABA 문제 시뮬레이션 ───────────────────────────────────────
data class Node<T>(val value: T, val next: Node<T>?)

class ABAProneStack<T> {
    val top = AtomicReference<Node<T>?>(null)

    fun push(value: T) {
        var current: Node<T>?
        val newNode: Node<T>
        do {
            current = top.get()
            newNode = Node(value, current)
        } while (!top.compareAndSet(current, newNode))
    }

    fun pop(): T? {
        var current: Node<T>?
        do {
            current = top.get() ?: return null
        } while (!top.compareAndSet(current, current.next))
        return current.value
    }
}

// ── ABA 문제 해결: 스탬프(버전) 사용 ─────────────────────────
class ABASafeStack<T> {
    // AtomicStampedReference는 참조 + 정수 스탬프를 원자적으로 관리
    private val top = AtomicStampedReference<Node<T>?>(null, 0)

    fun push(value: T) {
        val stamp = IntArray(1)
        var current: Node<T>?
        var newNode: Node<T>
        do {
            current = top.get(stamp)
            newNode = Node(value, current)
        } while (!top.compareAndSet(current, newNode, stamp[0], stamp[0] + 1))
        // 스탬프가 다르면 CAS 실패 → A→B→A 시나리오 방지
    }

    fun pop(): T? {
        val stamp = IntArray(1)
        var current: Node<T>?
        do {
            current = top.get(stamp) ?: return null
        } while (!top.compareAndSet(current, current.next, stamp[0], stamp[0] + 1))
        return current.value
    }
}

// ── ABA 문제 재현 시나리오 설명 ───────────────────────────────
fun explainABAProblem() {
    println("""
    ABA 문제 시나리오:
    
    초기 상태: top → A → B → null
    
    Thread 1: pop() 시작, current = A, next = B 읽음
    (Thread 1 선점됨)
    
    Thread 2: pop() → A 제거 (top → B)
    Thread 2: pop() → B 제거 (top → null)
    Thread 2: push(A) → A 재삽입 (top → A → null)
    
    Thread 1 재개: CAS(top, A, B) 성공!
    (top이 A인 건 맞지만, next가 null인 A)
    결과: top → B ... 하지만 B는 이미 해제된 메모리!
    
    해결책:
    - AtomicStampedReference: 버전 번호(stamp)로 변경 횟수 추적
    - AtomicMarkableReference: 삭제 표시 비트 사용
    - Hazard Pointers: GC 없는 언어(C/C++)에서 메모리 재사용 방지
    - Epoch-Based Reclamation: 안전한 메모리 해제 시점 결정
    """.trimIndent())
}

// ── 메모리 순서(Memory Ordering) 주의사항 ─────────────────────
fun memoryOrderingNote() {
    println("""
    Java/Kotlin의 volatile 키워드:
    - volatile 변수에 대한 읽기/쓰기는 happens-before 관계 보장
    - CPU와 컴파일러의 명령어 재순서화(reordering) 방지
    - AtomicXxx 클래스는 내부적으로 volatile 시맨틱 사용
    
    C++의 memory_order (std::atomic):
    - memory_order_relaxed: 순서 보장 없음, 최고 성능
    - memory_order_acquire: 읽기 후 명령어가 앞으로 이동 방지
    - memory_order_release: 쓰기 전 명령어가 뒤로 이동 방지
    - memory_order_seq_cst: 전체 순차 일관성 (기본값, 가장 안전)
    """.trimIndent())
}

fun main() {
    explainABAProblem()

    // ABA-safe 스택 테스트
    val safeStack = ABASafeStack<Int>()
    repeat(5) { safeStack.push(it * 10) }
    println("\nABA-Safe Stack pop 결과:")
    repeat(5) { print("${safeStack.pop()} ") }
    println()

    memoryOrderingNote()
}
```

---

## Lock-Free vs Lock-Based 성능 비교

| 경쟁 강도 | Lock-Based | Lock-Free |
|-----------|-----------|-----------|
| 경쟁 없음 | 빠름 | 비슷하거나 약간 느림 |
| 낮은 경쟁 | 보통 | 빠름 |
| 높은 경쟁 | 느림 (컨텍스트 스위칭) | 훨씬 빠름 |
| 데드락 위험 | 있음 | 없음 |
| 구현 복잡도 | 낮음 | 높음 |
| 디버깅 난이도 | 중간 | 매우 높음 |

---

## 주의사항 및 실전 팁

### 1. ABA 문제를 항상 고려하라

단순 포인터 CAS는 ABA 취약점이 있습니다. Java라면 `AtomicStampedReference`, C++라면 `std::atomic<tagged_ptr>`이나 Hazard Pointer를 사용하세요. 가비지 컬렉션이 있는 언어(Java, Go)에서는 ABA가 완화되지만 완전히 사라지지는 않습니다.

### 2. CAS 루프는 스핀(spin)이다

CAS 재시도 루프는 CPU를 계속 사용하는 **스핀 락**입니다. 경쟁이 극심하면 오히려 CPU를 낭비합니다. 이럴 때는 `Thread.yield()` 또는 지수 백오프(exponential backoff)를 도입하세요.

### 3. 메모리 순서(Memory Ordering)를 이해하라

Lock-Free 코드는 CPU와 컴파일러의 명령어 재순서화(reordering)에 취약합니다. Java에서는 `volatile`과 `Atomic` 클래스가 이를 처리해주지만, C/C++에서는 `std::memory_order`를 명시적으로 지정해야 합니다. 잘못된 메모리 순서는 특정 아키텍처(ARM, POWER)에서만 발현되는 하이젠버그(heisenbug)를 만듭니다.

### 4. 실전에서는 검증된 라이브러리를 사용하라

Lock-Free 자료구조를 직접 구현하는 것은 매우 어렵습니다. 실전에서는 Java의 `java.util.concurrent` 패키지(`ConcurrentLinkedQueue`, `ConcurrentHashMap`, `ConcurrentSkipListMap`)나 Go의 `sync/atomic` 패키지를 활용하는 것이 안전합니다.

### 5. 프로파일링 없이 Lock-Free를 도입하지 마라

Lock-Free는 "락이 없어서 무조건 빠를 것"이라는 오해가 있습니다. 실제로 락 경쟁이 적은 경우 Lock-Free의 CAS 루프와 메모리 배리어(memory barrier) 오버헤드가 더 클 수 있습니다. **반드시 프로파일링을 먼저 하고**, 락 경쟁이 실제 병목임을 확인한 뒤에 Lock-Free를 고려하세요.

---

## 참고 자료

- [Wikipedia - Compare-and-swap](https://en.wikipedia.org/wiki/Compare-and-swap)
- [Wikipedia - ABA Problem](https://en.wikipedia.org/wiki/ABA_problem)
- [Baeldung CS - The ABA Problem in Concurrency](https://www.baeldung.com/cs/aba-concurrency)
- [SEI CERT C - CON09-C: Avoid the ABA problem](https://wiki.sei.cmu.edu/confluence/display/c/CON09-C.+Avoid+the+ABA+problem+when+using+lock-free+algorithms)
