---
layout: post
title: "JVM 메모리 모델(JMM) 심층 분석: happens-before, volatile, synchronized의 메모리 가시성"
date: 2026-10-02
categories: [cs, computer-science]
tags: [jvm, java-memory-model, happens-before, volatile, synchronized, concurrency, thread-safety]
---

멀티스레드 프로그래밍에서 발생하는 가장 교묘한 버그들 중 상당수는 JVM 메모리 모델(Java Memory Model, JMM)에 대한 오해에서 비롯됩니다. 코드가 단일 스레드 환경에서는 완벽하게 동작하지만 멀티스레드 환경에서는 간헐적으로 오류를 일으킨다면, 그 원인은 십중팔구 메모리 가시성(visibility) 문제입니다. 이 글에서는 JMM의 핵심 개념인 happens-before 관계를 중심으로, `volatile`과 `synchronized`가 어떻게 메모리 가시성을 보장하는지 깊게 파헤칩니다.

## JVM 메모리 모델이란?

Java Memory Model(JMM)은 JLS(Java Language Specification) 17장에 명세된 규칙으로, 멀티스레드 프로그램에서 메모리 읽기·쓰기 연산의 가시성과 순서를 정의합니다. JMM은 물리적인 메모리 아키텍처(CPU 캐시, 레지스터, 메인 메모리)를 추상화하여 플랫폼 독립적인 동시성 보장을 제공합니다.

### 왜 메모리 모델이 필요한가?

현대 컴퓨터 시스템은 성능 최적화를 위해 다음과 같은 변환을 적극 활용합니다:

- **CPU 레지스터 캐싱**: 변수 값을 메인 메모리가 아닌 레지스터에 보관
- **CPU 캐시 (L1/L2/L3)**: 각 CPU 코어마다 독립적인 캐시 보유
- **명령어 재정렬(instruction reordering)**: 컴파일러와 CPU가 순서를 바꿔 실행
- **스토어 버퍼(store buffer)**: 쓰기 연산을 지연 처리

이로 인해 Thread A가 변수를 수정해도 Thread B는 그 변경을 즉시 볼 수 없습니다. JMM은 이 문제를 해결하기 위한 규칙 체계입니다.

## happens-before 관계: JMM의 핵심

happens-before는 두 연산 간의 메모리 가시성을 보장하는 편순서(partial order) 관계입니다. 연산 A가 연산 B보다 happens-before이면, A의 모든 메모리 쓰기 결과가 B에서 반드시 가시적(visible)입니다.

### happens-before 규칙 목록

1. **프로그램 순서 규칙**: 단일 스레드 내에서 앞의 연산은 뒤의 연산보다 happens-before
2. **모니터 잠금 규칙**: `synchronized` 블록의 unlock은 이후 같은 모니터의 lock보다 happens-before
3. **volatile 변수 규칙**: volatile 변수 쓰기는 이후 같은 변수 읽기보다 happens-before
4. **스레드 시작 규칙**: `Thread.start()`는 시작된 스레드의 모든 연산보다 happens-before
5. **스레드 종료 규칙**: 스레드의 모든 연산은 `Thread.join()` 반환보다 happens-before
6. **인터럽트 규칙**: `Thread.interrupt()` 호출은 인터럽트 감지보다 happens-before
7. **객체 생성자 규칙**: 생성자 완료는 `finalize()` 시작보다 happens-before
8. **전이 규칙(transitivity)**: A happens-before B이고 B happens-before C이면 A happens-before C

## 가시성 문제: volatile 없이 발생하는 버그

```java
public class VisibilityBug {
    // volatile 없는 일반 변수
    private boolean running = true;
    private int counter = 0;

    public void start() {
        Thread worker = new Thread(() -> {
            int localCounter = 0;
            while (running) {  // CPU가 running을 레지스터에 캐싱할 수 있음
                localCounter++;
            }
            counter = localCounter;
            System.out.println("Worker stopped. counter=" + counter);
        });
        worker.start();

        // 메인 스레드에서 1초 후 종료 신호
        try { Thread.sleep(1000); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        running = false;  // Worker 스레드는 이 변경을 보지 못할 수 있음!
        System.out.println("Main: set running=false");

        try { worker.join(); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        // 문제 1: worker가 무한 루프를 돌 수 있음 (running 캐싱)
        // 문제 2: counter 값이 0으로 보일 수 있음 (쓰기 가시성 보장 없음)
    }

    // 해결: volatile 키워드 사용
    private volatile boolean runningV = true;
    private volatile int counterV = 0;

    public void startFixed() {
        Thread worker = new Thread(() -> {
            int localCounter = 0;
            while (runningV) {  // 항상 최신 값을 메인 메모리에서 읽음
                localCounter++;
            }
            counterV = localCounter;  // volatile 쓰기 → 메인 메모리에 즉시 반영
            System.out.println("Worker stopped. counter=" + counterV);
        });
        worker.start();

        try { Thread.sleep(1000); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        runningV = false;  // happens-before: 이 쓰기는 worker의 읽기보다 먼저 보임
        System.out.println("Main: set runningV=false");

        try { worker.join(); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }

    public static void main(String[] args) {
        new VisibilityBug().startFixed();
    }
}
```

`volatile` 키워드는 다음 두 가지를 보장합니다:
- **가시성**: volatile 변수 읽기는 항상 다른 스레드의 최신 쓰기를 봄
- **부분적 순서 보장**: volatile 쓰기 전의 모든 연산은 volatile 읽기 후의 연산보다 happens-before

### volatile의 한계: 복합 연산에서의 원자성 미보장

```java
public class VolatileAtomicityProblem {
    private volatile int count = 0;

    // 이 메서드는 스레드 안전하지 않음!
    public void increment() {
        count++;  // 실제로는 3단계: read → increment → write (비원자적)
    }

    // 해결책 1: synchronized
    public synchronized void incrementSafe() {
        count++;
    }

    // 해결책 2: AtomicInteger (Lock-free, CAS 기반)
    private java.util.concurrent.atomic.AtomicInteger atomicCount =
        new java.util.concurrent.atomic.AtomicInteger(0);

    public void incrementAtomic() {
        atomicCount.incrementAndGet();  // CAS 기반 원자적 연산
    }

    // 복합 조건부 갱신: compare-and-set 패턴
    public boolean compareAndIncrement(int expected) {
        return atomicCount.compareAndSet(expected, expected + 1);
    }

    public static void main(String[] args) throws InterruptedException {
        VolatileAtomicityProblem prob = new VolatileAtomicityProblem();
        int THREAD_COUNT = 10;
        int ITER_COUNT = 100_000;

        // volatile count 테스트 (레이스 컨디션 발생)
        Thread[] threads = new Thread[THREAD_COUNT];
        for (int i = 0; i < THREAD_COUNT; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < ITER_COUNT; j++) prob.increment();
            });
        }
        for (Thread t : threads) t.start();
        for (Thread t : threads) t.join();
        // 예상: 1,000,000 / 실제: 더 작은 값 (레이스 컨디션으로 increment 소실)
        System.out.println("volatile count (expected 1000000): " + prob.count);

        // AtomicInteger 테스트 (정확한 결과)
        for (int i = 0; i < THREAD_COUNT; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < ITER_COUNT; j++) prob.incrementAtomic();
            });
        }
        for (Thread t : threads) t.start();
        for (Thread t : threads) t.join();
        System.out.println("AtomicInteger count (expected 1000000): " + prob.atomicCount.get());
    }
}
```

## synchronized의 happens-before 보장

`synchronized`는 volatile보다 강력한 보장을 제공합니다. 모니터 unlock → lock 사이에 happens-before가 성립하므로, synchronized 블록 내의 모든 읽기·쓰기가 다음 스레드에게 가시적입니다.

```java
public class SynchronizedVisibility {
    private int x = 0;
    private int y = 0;
    private final Object lock = new Object();

    public void writer() {
        synchronized (lock) {
            x = 42;      // (1)
            y = 100;     // (2)
        }  // unlock → happens-before → 다음 lock
    }

    public void reader() {
        synchronized (lock) {  // lock (writer의 unlock보다 happens-before 이후)
            // (1), (2)의 쓰기가 모두 가시적으로 보임
            System.out.println("x=" + x + ", y=" + y);  // 반드시 x=42, y=100
        }
    }

    // Double-Checked Locking (DCL) - volatile 필수
    private static volatile SynchronizedVisibility instance;

    public static SynchronizedVisibility getInstance() {
        if (instance == null) {                    // (A) 첫 번째 체크 (잠금 없음)
            synchronized (SynchronizedVisibility.class) {
                if (instance == null) {            // (B) 두 번째 체크 (잠금 있음)
                    instance = new SynchronizedVisibility();  // (C)
                }
            }
        }
        return instance;
        // volatile 없이는: (C)의 생성자 완료 전에 instance 참조가 공개될 수 있음
        // volatile 있으면: instance 쓰기 happens-before instance 읽기 → 안전
    }
}
```

### 명령어 재정렬과 메모리 배리어

JMM이 명령어 재정렬을 어떻게 제어하는지 이해하면 더 깊이 알 수 있습니다:

```java
// 재정렬 예시: 컴파일러/CPU는 아래 두 줄의 순서를 바꿀 수 있음
int a = 1;   // (1)
int b = 2;   // (2)
// (1)과 (2)는 서로 독립적이므로 재정렬 가능 → 단일 스레드에서는 문제 없음

// 하지만 아래는 문제가 됨:
class Publisher {
    int data;
    volatile boolean published;

    void publish(int value) {
        data = value;         // (1) 데이터 초기화
        published = true;     // (2) volatile 쓰기 → (1)이 반드시 (2) 이전에 완료
        // volatile 쓰기는 StoreStore 배리어 역할 → 위의 쓰기들이 메인 메모리에 반영됨
    }
}

class Subscriber {
    void consume(Publisher pub) {
        while (!pub.published) { /* spin */ }  // volatile 읽기
        // volatile 읽기는 LoadLoad 배리어 역할
        // published가 true를 보면 data 쓰기도 반드시 보임
        System.out.println("data=" + pub.data);  // 항상 올바른 값
    }
}
```

## 실전: JMM을 활용한 안전한 상태 공유 패턴

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;

public class ThreadSafePatterns {

    // 패턴 1: Immutable + volatile reference (가장 안전하고 빠름)
    static volatile ImmutableConfig config = new ImmutableConfig("default", 8080);

    static class ImmutableConfig {
        final String host;
        final int port;
        ImmutableConfig(String host, int port) { this.host = host; this.port = port; }
    }

    static void updateConfig(String host, int port) {
        config = new ImmutableConfig(host, port);  // atomic reference swap
        // 이전 config 객체는 GC가 처리
    }

    // 패턴 2: CopyOnWriteArrayList - 읽기 많고 쓰기 적은 경우
    static CopyOnWriteArrayList<String> listeners = new CopyOnWriteArrayList<>();

    // 패턴 3: StampedLock - 낙관적 읽기로 성능 향상
    static java.util.concurrent.locks.StampedLock stampedLock =
        new java.util.concurrent.locks.StampedLock();
    static double x, y;

    static double distanceFromOriginOptimistic() {
        long stamp = stampedLock.tryOptimisticRead();  // 잠금 없이 읽기 시도
        double currentX = x, currentY = y;
        if (!stampedLock.validate(stamp)) {  // 읽는 동안 쓰기가 있었으면
            stamp = stampedLock.readLock();  // 정식 읽기 잠금으로 재시도
            try {
                currentX = x;
                currentY = y;
            } finally {
                stampedLock.unlockRead(stamp);
            }
        }
        return Math.sqrt(currentX * currentX + currentY * currentY);
    }

    // 패턴 4: CompletableFuture - happens-before 체인
    static CompletableFuture<String> asyncPipeline() {
        return CompletableFuture
            .supplyAsync(() -> "raw data")         // 비동기 실행
            .thenApply(data -> data.toUpperCase()) // happens-before 보장
            .thenApply(data -> "[" + data + "]");  // 체인 전체 happens-before 보장
    }

    public static void main(String[] args) throws Exception {
        System.out.println(asyncPipeline().get());  // [RAW DATA]
    }
}
```

## 주의사항과 실무 팁

### 1. volatile은 64비트 타입에도 필수

```java
// JVM 스펙: 32비트가 아닌 플랫폼에서 long/double은 두 번의 32비트 연산으로 수행될 수 있음
private long bigNumber = 0L;          // 위험: 워드 tearing 가능
private volatile long safeNumber = 0L; // 안전: volatile이 원자성 보장
```

### 2. final 필드의 초기화 보장

```java
class SafePublication {
    final int value;
    SafePublication(int v) { this.value = v; }
    // final 필드는 생성자 완료 후 모든 스레드에서 올바른 값을 보장
    // 생성자 내 this 누출(this escape)만 없으면 안전
}
```

### 3. happens-before 체인 주의

```java
// volatile A → synchronized B가 연결되어 있으면 A도 B를 통해 가시성 보장
// 하지만 불필요한 의존성 체인은 성능 저하 → 명시적 설계 권장
```

### 4. 도구 활용

- **ThreadSanitizer (TSan)**: C/C++에서 데이터 레이스 감지
- **Java Concurrency Stress (jcstress)**: JVM 동시성 버그 검증 테스트 프레임워크
- **IntelliJ IDEA 동시성 검사기**: 정적 분석으로 잠재적 문제 탐지

## 요약

| 메커니즘 | 가시성 | 원자성 | 순서 보장 | 성능 |
|----------|--------|--------|-----------|------|
| `volatile` | ✅ | 단순 읽기/쓰기만 | 부분적 | 높음 |
| `synchronized` | ✅ | ✅ | 완전 | 중간 |
| `AtomicXxx` | ✅ | ✅ (CAS) | 부분적 | 높음 |
| `Lock` | ✅ | ✅ | 완전 | 중간 |

JMM은 복잡하지만 happens-before 규칙 8가지를 이해하면 대부분의 동시성 문제를 체계적으로 분석할 수 있습니다. volatile은 단순 플래그나 참조 교환에, synchronized 혹은 Lock은 복합 연산에, AtomicXxx는 숫자 카운터나 단일 참조 업데이트에 적합합니다.

## 참고 자료
- [JSR-133: Java Memory Model and Thread Specification](https://www.cs.umd.edu/~pugh/java/memoryModel/jsr133.pdf)
- [Java Language Specification §17 — Threads and Locks](https://docs.oracle.com/javase/specs/jls/se21/html/jls-17.html)
- [Java Concurrency in Practice (Goetz et al.)](https://jcip.net/)
- [jcstress: Java Concurrency Stress Tests](https://github.com/openjdk/jcstress)
