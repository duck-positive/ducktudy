---
layout: post
title: "JVM Virtual Threads (Project Loom) 완전 정복: 경량 스레드와 현대적 동시성 프로그래밍"
date: 2026-11-18
categories: [cs, computer-science]
tags: [java, jvm, virtual-threads, project-loom, concurrency, structured-concurrency, java21]
---

Java 21에서 정식 릴리스된 **Virtual Threads(JEP 444)**는 JVM 동시성 프로그래밍의 패러다임을 근본적으로 바꾸는 기능입니다. 수십 년간 Java 개발자들을 괴롭혀 온 "스레드는 비싸다"는 상식을 뒤집고, 수백만 개의 동시 작업을 단순한 블로킹 코드로 처리할 수 있게 합니다. 이 글에서는 Virtual Threads의 내부 구조부터 실전 활용 패턴까지 완전히 정복합니다.

## 왜 Virtual Threads가 필요한가

### 기존 플랫폼 스레드의 한계

전통적인 Java 스레드는 OS 스레드와 1:1로 대응됩니다. OS 스레드 하나는 보통 1~2MB의 스택 메모리를 차지하고, 컨텍스트 스위칭 비용이 상당합니다. 일반적인 서버에서 수천 개의 스레드를 생성하면 메모리 압박과 스케줄링 오버헤드가 급증합니다.

**스레드 풀의 함정**: 그래서 개발자들은 스레드 풀을 사용하고 풀 크기를 조정하는 복잡한 튜닝 작업을 해야 했습니다. I/O가 많은 서비스(HTTP 요청, DB 쿼리 등)에서는 스레드가 대부분의 시간을 블로킹 대기 상태로 보내므로, CPU 코어 수 대비 훨씬 많은 스레드가 필요하지만 그럴수록 메모리 낭비도 커집니다.

**리액티브 프로그래밍의 복잡성**: 이 문제를 해결하려고 등장한 것이 리액티브 프로그래밍입니다. Project Reactor, RxJava 같은 라이브러리는 비동기 비블로킹 I/O를 활용해 적은 스레드로 많은 요청을 처리합니다. 하지만 코드가 `flatMap`, `zipWith`, `switchIfEmpty` 같은 연산자 체인으로 가득 차 디버깅이 어렵고 스택 트레이스가 의미 없어지는 단점이 있습니다.

### Virtual Threads의 접근법

Virtual Threads는 완전히 다른 해답을 제시합니다. **블로킹 코드를 그대로 쓰되, JVM이 그것을 비블로킹처럼 실행한다**는 것입니다.

```
OS 스레드 (Carrier Thread): 적은 수 (CPU 코어 수 정도)
         ↕ 다중화(multiplexing)
Virtual Thread: 수백만 개 가능
```

Virtual Thread가 블로킹 작업(I/O, sleep, `synchronized` 등)을 만나면, JVM은 해당 Virtual Thread의 상태를 힙에 저장하고 Carrier Thread를 다른 Virtual Thread에게 양도합니다. 블로킹이 완료되면 Virtual Thread는 다시 어떤 Carrier Thread 위에서든 재개됩니다.

## Virtual Threads의 내부 구조

### Continuation 기반 구현

Virtual Thread는 내부적으로 `Continuation`을 사용합니다. Continuation은 실행 중인 코드의 상태(스택 프레임, 프로그램 카운터 등)를 힙 객체로 캡처하는 메커니즘입니다.

```
Virtual Thread 실행 흐름:
1. VT가 Carrier Thread 위에서 실행
2. VT가 블로킹 작업 호출
3. JVM이 VT의 Continuation을 힙에 저장
4. Carrier Thread가 다른 VT를 실행
5. 블로킹 완료 → VT가 ForkJoinPool 큐에 등록
6. 비어있는 Carrier Thread가 VT Continuation을 재개
```

### 스케줄러 구조

Virtual Thread의 스케줄러는 `ForkJoinPool`의 work-stealing 기반으로 동작합니다. 기본 Carrier Thread 풀 크기는 `Runtime.getRuntime().availableProcessors()`와 같습니다.

```java
// JVM 내부 개략적 구조 (실제 소스 단순화)
class VirtualThread extends BaseVirtualThread {
    private volatile int state;      // 실행 상태
    private Continuation cont;       // 실행 상태 저장
    private Thread carrierThread;    // 현재 실행 중인 OS 스레드

    void park() {
        // 블로킹 → continuation을 힙에 저장
        // carrierThread를 해방
        cont.yield();
    }

    void unpark() {
        // 큐에 재등록
        scheduler.execute(this::runContinuation);
    }
}
```

## 실전 코드 예제

### 예제 1: 대규모 동시 HTTP 요청 처리

다음은 Virtual Threads로 1만 개의 HTTP 요청을 동시에 처리하는 예제입니다:

```java
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class VirtualThreadHttpDemo {

    public static void main(String[] args) throws Exception {
        int taskCount = 10_000;
        List<String> results = new ArrayList<>(taskCount);

        // Virtual Thread를 사용하는 ExecutorService
        try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
            HttpClient client = HttpClient.newBuilder()
                .executor(executor)
                .build();

            List<java.util.concurrent.Future<String>> futures = new ArrayList<>();

            long start = System.currentTimeMillis();

            for (int i = 0; i < taskCount; i++) {
                final int taskId = i;
                var future = executor.submit(() -> {
                    // 블로킹 I/O를 그냥 작성 — Virtual Thread가 처리
                    HttpRequest request = HttpRequest.newBuilder()
                        .uri(URI.create("https://httpbin.org/get?id=" + taskId))
                        .GET()
                        .build();
                    try {
                        HttpResponse<String> response = client.send(
                            request,
                            HttpResponse.BodyHandlers.ofString()
                        );
                        return "Task " + taskId + ": HTTP " + response.statusCode();
                    } catch (Exception e) {
                        return "Task " + taskId + ": ERROR " + e.getMessage();
                    }
                });
                futures.add(future);
            }

            for (var f : futures) {
                results.add(f.get());
            }

            long elapsed = System.currentTimeMillis() - start;
            System.out.printf("완료: %d 요청, 소요시간: %dms%n", taskCount, elapsed);
        }
        // executor.close()는 try-with-resources로 자동 호출 (모든 태스크 완료 대기)
    }
}

// 기존 플랫폼 스레드로 10,000개 → OOM 혹은 수십 초 소요
// Virtual Thread로 10,000개 → 수초 내, 메모리 사용량 획기적 감소
```

### 예제 2: Structured Concurrency로 안전한 병렬 처리

Java 21의 `StructuredTaskScope`는 Virtual Thread와 함께 계층적 동시성을 구현합니다:

```java
import java.util.concurrent.StructuredTaskScope;
import java.util.concurrent.StructuredTaskScope.Subtask;

public class StructuredConcurrencyDemo {

    record UserProfile(String name, String email) {}
    record UserOrder(String orderId, double amount) {}
    record UserDashboard(UserProfile profile, UserOrder latestOrder) {}

    // 두 비동기 작업이 모두 성공해야 결과 반환
    static UserDashboard fetchDashboard(long userId) throws Exception {
        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {

            // 두 작업을 병렬로 시작 (각각 Virtual Thread에서 실행)
            Subtask<UserProfile> profileTask = scope.fork(() -> fetchProfile(userId));
            Subtask<UserOrder> orderTask = scope.fork(() -> fetchLatestOrder(userId));

            // 두 작업이 모두 완료될 때까지 대기
            scope.join();
            // 하나라도 실패하면 예외 전파
            scope.throwIfFailed();

            // 여기까지 오면 두 작업 모두 성공
            return new UserDashboard(profileTask.get(), orderTask.get());
        }
        // scope 종료 시 모든 하위 태스크도 자동으로 취소됨 (구조적 보장)
    }

    // ShutdownOnSuccess: 첫 번째 성공 결과만 필요한 경우
    static String fetchFromFastestReplica(long dataId) throws Exception {
        try (var scope = new StructuredTaskScope.ShutdownOnSuccess<String>()) {
            scope.fork(() -> fetchFromReplica("replica-1", dataId));
            scope.fork(() -> fetchFromReplica("replica-2", dataId));
            scope.fork(() -> fetchFromReplica("replica-3", dataId));

            scope.join();
            return scope.result(); // 가장 빠른 결과 반환
        }
    }

    private static UserProfile fetchProfile(long userId) throws InterruptedException {
        Thread.sleep(100); // DB 조회 시뮬레이션
        return new UserProfile("Alice", "alice@example.com");
    }

    private static UserOrder fetchLatestOrder(long userId) throws InterruptedException {
        Thread.sleep(150); // 다른 서비스 호출 시뮬레이션
        return new UserOrder("ORD-9999", 59900.0);
    }

    private static String fetchFromReplica(String replica, long dataId)
            throws InterruptedException {
        Thread.sleep((long)(Math.random() * 200));
        return replica + ":data:" + dataId;
    }

    public static void main(String[] args) throws Exception {
        // fetchProfile(100ms) + fetchLatestOrder(150ms) 를 직렬 처리 시 250ms
        // Structured Concurrency 사용 시 약 150ms (가장 긴 작업 시간)
        long start = System.currentTimeMillis();
        UserDashboard dashboard = fetchDashboard(42L);
        System.out.printf("Dashboard 로드: %dms, 사용자: %s%n",
            System.currentTimeMillis() - start, dashboard.profile().name());
    }
}
```

## 주의사항과 베스트 프랙티스

### 1. `synchronized`와 Virtual Thread 고정(Pinning)

가장 중요한 주의점입니다. Virtual Thread가 `synchronized` 블록 안에서 블로킹 작업을 만나면, **Carrier Thread에 고정(pin)** 됩니다. 즉, 블로킹 동안 Carrier Thread가 해방되지 않아 플랫폼 스레드와 같은 동작이 됩니다.

```java
// ❌ 나쁜 예: synchronized 블록 내 블로킹 I/O
synchronized (lock) {
    result = expensiveDbQuery(); // Virtual Thread가 고정됨
}

// ✅ 좋은 예: ReentrantLock 사용 (고정 없음)
lock.lock();
try {
    result = expensiveDbQuery(); // 블로킹 시 Carrier Thread 해방 가능
} finally {
    lock.unlock();
}
```

JDK 고정 감지: JVM 플래그 `-Djdk.tracePinnedThreads=full`로 고정 현상을 모니터링할 수 있습니다.

### 2. ThreadLocal vs ScopedValue

Virtual Thread는 수백만 개 생성 가능하므로 `ThreadLocal`에 큰 객체를 저장하면 메모리 문제가 생깁니다. Java 21에서 프리뷰로 도입된 `ScopedValue`를 사용하세요:

```java
// ScopedValue: 불변, 범위 한정, 상속 가능
static final ScopedValue<User> CURRENT_USER = ScopedValue.newInstance();

// 특정 범위에서만 값 바인딩
ScopedValue.where(CURRENT_USER, user)
    .run(() -> handleRequest(request));
```

### 3. CPU 집약적 작업은 여전히 플랫폼 스레드

Virtual Thread는 블로킹 I/O 작업에 최적화되어 있습니다. 순수 CPU 연산(암호화, 이미지 처리, 복잡한 계산)은 플랫폼 스레드 풀을 사용하는 것이 더 효율적입니다.

### 4. 스레드 풀 사용 패턴 변화

```java
// Java 21+ 권장 패턴
// 각 작업마다 하나의 Virtual Thread (스레드 풀 없이도 OK)
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(task1);
    executor.submit(task2);
    // ...
}

// 기존 패턴 (여전히 유효하지만 Virtual Thread 시대에는 불필요할 수 있음)
ExecutorService pool = Executors.newFixedThreadPool(100);
```

## 성능 특성과 모니터링

Virtual Thread의 메모리 오버헤드는 초기 스택 크기가 수백 바이트에 불과합니다(플랫폼 스레드는 1MB 이상). 블로킹 작업이 많은 I/O 집약적 서비스에서는 같은 메모리로 훨씬 더 많은 동시 연결을 처리할 수 있습니다.

JDK 플라이트 레코더(JFR)로 Virtual Thread 이벤트를 추적할 수 있습니다:
```
jcmd <pid> JFR.start name=vt-profile duration=60s
# jdk.VirtualThreadStart, jdk.VirtualThreadEnd, jdk.VirtualThreadPinned 이벤트 분석
```

## 참고 자료
- [JEP 444: Virtual Threads (OpenJDK)](https://openjdk.org/jeps/444)
- [Virtual Threads Explained - Sip of Java (inside.java)](https://inside.java/2023/10/30/sip086)
- [JEP 453: Structured Concurrency (OpenJDK)](https://openjdk.org/jeps/453)
- [Project Loom: Modern Scalable Concurrency for the Java Platform](https://www.infoworld.com/article/2337208/intro-to-virtual-threads-a-new-approach-to-java-concurrency.html)
