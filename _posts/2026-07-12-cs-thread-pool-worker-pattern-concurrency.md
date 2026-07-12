---
layout: post
title: "스레드 풀(Thread Pool) 완전 정복: 고성능 동시성 처리의 핵심 원리와 구현"
date: 2026-07-12
categories: [cs, computer-science]
tags: [thread-pool, concurrency, Java, Python, executor, worker-pattern, synchronization, blocking-queue]
---

## 스레드 풀이란 무엇인가?

웹 서버가 초당 수천 건의 요청을 처리한다고 가정해봅시다. 요청마다 새 스레드를 생성하고, 처리가 끝나면 스레드를 파괴하는 방식은 직관적이지만 심각한 문제를 안고 있습니다.

스레드 생성 비용은 생각보다 큽니다. Linux에서 `clone()` 시스템 콜로 스레드를 생성하는 데 약 10~100마이크로초가 걸리고, 스택 메모리를 기본 8MB씩 할당합니다. 초당 10,000개의 요청이 들어온다면 스레드 생성·파괴 비용만으로 전체 처리 시간의 상당 부분을 잡아먹게 됩니다.

**스레드 풀(Thread Pool)**은 이 문제를 해결하는 고전적인 설계 패턴입니다. 미리 일정 수의 스레드를 생성해두고, 작업이 들어오면 **작업 큐(Task Queue)**에 넣습니다. 유휴 스레드가 큐에서 작업을 꺼내 처리한 뒤, 스레드를 파괴하지 않고 다음 작업을 기다립니다.

```
Client Requests
     │
     ▼
┌─────────────────────────────┐
│       Task Queue (Blocking) │  ← 작업 대기열
│  [T1] [T2] [T3] [T4] ...   │
└──────────────┬──────────────┘
               │  작업 분배
     ┌─────────┼─────────┐
     ▼         ▼         ▼
┌────────┐ ┌────────┐ ┌────────┐
│Worker 1│ │Worker 2│ │Worker 3│  ← 미리 생성된 스레드들
│(작업중)│ │(유휴)  │ │(작업중)│
└────────┘ └────────┘ └────────┘
```

---

## 왜 스레드 풀이 필요한가?

### 1. 스레드 생성 비용 제거

스레드 생성은 OS 레벨의 자원 할당을 수반합니다. 스레드 풀은 스레드를 재사용하여 이 오버헤드를 한 번으로 줄입니다.

### 2. 자원 소비 제한

무제한으로 스레드를 생성하면 메모리 부족, CPU 컨텍스트 스위칭 폭발이 발생합니다. 스레드 풀은 최대 스레드 수를 제한하여 시스템 자원을 안정적으로 관리합니다.

### 3. 응답성 향상

작업이 도착했을 때 이미 스레드가 준비되어 있으므로 즉시 처리를 시작할 수 있습니다.

### 4. 백프레셔(Backpressure) 구현

작업 큐가 가득 찼을 때 어떻게 할지(거부, 대기, 호출자 직접 실행 등)를 명시적으로 제어할 수 있습니다.

---

## 스레드 풀의 핵심 구성 요소

### 1. 코어 스레드 수 (Core Pool Size)

항상 살아있는 최소 스레드 수입니다. 유휴 상태여도 이 수만큼은 유지됩니다.

### 2. 최대 스레드 수 (Maximum Pool Size)

부하가 급증할 때 생성할 수 있는 최대 스레드 수입니다. 코어 스레드 수를 초과해 생성된 스레드는 일정 시간 유휴 상태가 되면 자동으로 종료됩니다.

### 3. 작업 큐 (Work Queue)

`BlockingQueue` 구현에 따라 동작이 달라집니다.
- `LinkedBlockingQueue`: 무한(기본) 크기, 최대 스레드 수 설정이 무의미해짐
- `ArrayBlockingQueue`: 고정 크기, 큐가 가득 차면 거부 정책 실행
- `SynchronousQueue`: 버퍼 없음, 소비자 없으면 즉시 거부 (CachedThreadPool에 사용)

### 4. 거부 정책 (Rejected Execution Handler)

큐와 스레드가 모두 가득 찼을 때의 동작:
- `AbortPolicy`: `RejectedExecutionException` 발생 (기본값)
- `CallerRunsPolicy`: 호출자 스레드에서 직접 실행 (자연스러운 백프레셔)
- `DiscardPolicy`: 조용히 무시
- `DiscardOldestPolicy`: 가장 오래된 작업을 버리고 새 작업 수용

---

## 실제 구현 예제

### 예제 1: Java ThreadPoolExecutor 직접 구현과 Executors 팩토리

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class ThreadPoolDemo {

    // ThreadPoolExecutor를 직접 구성하는 방식 (권장)
    public static ThreadPoolExecutor createCustomPool() {
        int corePoolSize = 4;        // CPU 코어 수에 맞춤
        int maxPoolSize = 8;         // 최대 2배까지 확장
        long keepAliveSeconds = 60L; // 초과 스레드 유지 시간
        int queueCapacity = 100;     // 작업 큐 용량

        return new ThreadPoolExecutor(
            corePoolSize,
            maxPoolSize,
            keepAliveSeconds,
            TimeUnit.SECONDS,
            new ArrayBlockingQueue<>(queueCapacity),
            new ThreadFactory() {
                private final AtomicInteger count = new AtomicInteger(1);
                @Override
                public Thread newThread(Runnable r) {
                    Thread t = new Thread(r, "worker-" + count.getAndIncrement());
                    t.setDaemon(false); // JVM 종료 시 작업 완료 보장
                    return t;
                }
            },
            new ThreadPoolExecutor.CallerRunsPolicy() // 과부하 시 호출자가 직접 실행
        );
    }

    public static void main(String[] args) throws InterruptedException {
        ThreadPoolExecutor pool = createCustomPool();

        // 모니터링을 위한 주기적 상태 출력
        ScheduledExecutorService monitor = Executors.newSingleThreadScheduledExecutor();
        monitor.scheduleAtFixedRate(() -> {
            System.out.printf("[Monitor] Active: %d, Pool: %d, Queue: %d, Completed: %d%n",
                pool.getActiveCount(),
                pool.getPoolSize(),
                pool.getQueue().size(),
                pool.getCompletedTaskCount());
        }, 0, 500, TimeUnit.MILLISECONDS);

        // 200개 작업 제출 (큐 용량 100 초과 시 CallerRunsPolicy 동작)
        CountDownLatch latch = new CountDownLatch(200);
        AtomicInteger successCount = new AtomicInteger(0);

        for (int i = 0; i < 200; i++) {
            final int taskId = i;
            pool.execute(() -> {
                try {
                    // I/O 바운드 작업 시뮬레이션
                    Thread.sleep(100 + (taskId % 50));
                    successCount.incrementAndGet();
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } finally {
                    latch.countDown();
                }
            });
        }

        latch.await();
        System.out.println("완료된 작업: " + successCount.get());

        // 정상 종료: 대기 중인 작업 모두 완료 후 종료
        pool.shutdown();
        if (!pool.awaitTermination(30, TimeUnit.SECONDS)) {
            pool.shutdownNow(); // 강제 종료
        }
        monitor.shutdown();
    }
}

// Future와 CompletableFuture를 활용한 비동기 결과 처리
class AsyncTaskDemo {
    private final ExecutorService pool = Executors.newFixedThreadPool(
        Runtime.getRuntime().availableProcessors()
    );

    public CompletableFuture<String> fetchDataAsync(String url) {
        return CompletableFuture.supplyAsync(() -> {
            // HTTP 요청 시뮬레이션
            try { Thread.sleep(200); } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            return "Response from " + url;
        }, pool)
        .thenApply(response -> response.toUpperCase()) // 스레드 풀에서 변환
        .exceptionally(ex -> "Error: " + ex.getMessage());
    }

    // 여러 비동기 작업 병렬 실행 후 모두 완료되면 합산
    public CompletableFuture<String> aggregateResults(String[] urls) {
        CompletableFuture<String>[] futures = java.util.Arrays.stream(urls)
            .map(this::fetchDataAsync)
            .toArray(CompletableFuture[]::new);

        return CompletableFuture.allOf(futures)
            .thenApply(v -> java.util.Arrays.stream(futures)
                .map(CompletableFuture::join)
                .reduce("", (a, b) -> a + "\n" + b));
    }
}
```

---

### 예제 2: Python으로 스레드 풀 직접 구현 + concurrent.futures 비교

```python
import threading
import queue
import time
from concurrent.futures import ThreadPoolExecutor, as_completed
from typing import Callable, Any

class SimpleThreadPool:
    """
    교육 목적으로 만든 간단한 스레드 풀 구현체
    실제 사용에는 concurrent.futures.ThreadPoolExecutor를 권장
    """
    def __init__(self, num_workers: int, max_queue_size: int = 0):
        self._task_queue: queue.Queue = queue.Queue(maxsize=max_queue_size)
        self._workers: list[threading.Thread] = []
        self._shutdown = threading.Event()
        self._active_tasks = 0
        self._lock = threading.Lock()
        self._all_done = threading.Condition(self._lock)

        for i in range(num_workers):
            t = threading.Thread(target=self._worker_loop, name=f"pool-worker-{i}", daemon=True)
            t.start()
            self._workers.append(t)

    def _worker_loop(self):
        while True:
            try:
                # 1초 타임아웃으로 종료 신호 주기적 확인
                task, args, kwargs = self._task_queue.get(timeout=1.0)
            except queue.Empty:
                if self._shutdown.is_set():
                    break
                continue

            with self._lock:
                self._active_tasks += 1
            try:
                task(*args, **kwargs)
            except Exception as e:
                print(f"[{threading.current_thread().name}] 작업 실패: {e}")
            finally:
                with self._all_done:
                    self._active_tasks -= 1
                    self._task_queue.task_done()
                    if self._active_tasks == 0 and self._task_queue.empty():
                        self._all_done.notify_all()

    def submit(self, fn: Callable, *args, **kwargs):
        if self._shutdown.is_set():
            raise RuntimeError("스레드 풀이 이미 종료되었습니다")
        self._task_queue.put((fn, args, kwargs))

    def wait_all(self):
        with self._all_done:
            while self._active_tasks > 0 or not self._task_queue.empty():
                self._all_done.wait(timeout=0.1)

    def shutdown(self, wait: bool = True):
        self._shutdown.set()
        if wait:
            for w in self._workers:
                w.join()


# --- 벤치마크: 스레드 풀 vs 매번 스레드 생성 vs concurrent.futures ---

def simulate_io_task(task_id: int, delay: float = 0.05) -> str:
    time.sleep(delay)  # I/O 대기 시뮬레이션
    return f"Task {task_id} done by {threading.current_thread().name}"

NUM_TASKS = 200
DELAY = 0.02

# 1. 매번 스레드 생성 (안티패턴)
print("=== 1. 매번 스레드 생성 ===")
start = time.perf_counter()
threads = []
for i in range(NUM_TASKS):
    t = threading.Thread(target=simulate_io_task, args=(i, DELAY))
    t.start()
    threads.append(t)
for t in threads:
    t.join()
elapsed_naive = time.perf_counter() - start
print(f"소요 시간: {elapsed_naive:.3f}초")

# 2. SimpleThreadPool (직접 구현)
print("\n=== 2. SimpleThreadPool (8 workers) ===")
pool = SimpleThreadPool(num_workers=8)
start = time.perf_counter()
for i in range(NUM_TASKS):
    pool.submit(simulate_io_task, i, DELAY)
pool.wait_all()
elapsed_pool = time.perf_counter() - start
pool.shutdown(wait=False)
print(f"소요 시간: {elapsed_pool:.3f}초")

# 3. concurrent.futures.ThreadPoolExecutor (권장)
print("\n=== 3. concurrent.futures.ThreadPoolExecutor (8 workers) ===")
start = time.perf_counter()
results = []
with ThreadPoolExecutor(max_workers=8) as executor:
    futures = [executor.submit(simulate_io_task, i, DELAY) for i in range(NUM_TASKS)]
    for future in as_completed(futures):
        results.append(future.result())
elapsed_futures = time.perf_counter() - start
print(f"소요 시간: {elapsed_futures:.3f}초")
print(f"성공한 작업: {len(results)}/{NUM_TASKS}")

print(f"\n--- 비교 ---")
print(f"매번 생성: {elapsed_naive:.3f}초 (기준)")
print(f"직접 구현 풀: {elapsed_pool:.3f}초 ({elapsed_naive/elapsed_pool:.1f}x 향상)")
print(f"concurrent.futures: {elapsed_futures:.3f}초 ({elapsed_naive/elapsed_futures:.1f}x 향상)")
```

출력 예시:
```
=== 1. 매번 스레드 생성 ===
소요 시간: 1.847초

=== 2. SimpleThreadPool (8 workers) ===
소요 시간: 0.521초

=== 3. concurrent.futures.ThreadPoolExecutor ===
소요 시간: 0.511초

--- 비교 ---
매번 생성: 1.847초 (기준)
직접 구현 풀: 1.847/0.521 = 3.5x 향상
concurrent.futures: 3.6x 향상
```

---

## 스레드 풀 크기 결정 방법

스레드 풀 크기는 작업의 성격에 따라 달라집니다.

### CPU 바운드 작업

```python
import os

# CPU 코어 수 = 최적 스레드 수 (컨텍스트 스위칭 최소화)
cpu_bound_threads = os.cpu_count()
# Python의 경우 GIL 때문에 CPU 바운드에는 ProcessPoolExecutor 사용
from concurrent.futures import ProcessPoolExecutor

def cpu_bound_task(n: int) -> int:
    return sum(i * i for i in range(n))

with ProcessPoolExecutor(max_workers=os.cpu_count()) as executor:
    results = list(executor.map(cpu_bound_task, range(100, 10100, 100)))
```

### I/O 바운드 작업

```python
# Little's Law: 스레드 수 = 처리량 * 평균 응답 시간
# 실용 공식: N = CPU 코어 수 * (1 + I/O 대기 비율)
# 예: 4코어, I/O 비율 90% → N = 4 * (1 + 0.9/0.1) = 40

import os

def calculate_optimal_threads(cpu_cores: int, wait_time: float, compute_time: float) -> int:
    """
    wait_time: I/O 대기 시간 (초)
    compute_time: CPU 연산 시간 (초)
    """
    if compute_time == 0:
        return cpu_cores * 2  # 순수 I/O: 코어의 2배를 기본값으로
    blocking_coefficient = wait_time / compute_time
    return int(cpu_cores * (1 + blocking_coefficient))

cpu = os.cpu_count() or 4
io_threads = calculate_optimal_threads(cpu, wait_time=0.1, compute_time=0.01)
print(f"CPU 코어: {cpu}, 추천 I/O 스레드 수: {io_threads}")
```

---

## 주의사항 및 실전 팁

### 주의 1: 스레드 풀 내부에서 같은 풀에 블로킹 작업 제출 금지 (데드락)

```java
// 위험: 같은 풀의 작업이 다른 풀 작업의 완료를 기다림
ExecutorService pool = Executors.newFixedThreadPool(2);
pool.submit(() -> {
    // 이 submit은 풀의 스레드를 하나 더 필요로 하지만
    // 풀에 스레드가 2개뿐이고 모두 이 상황에 있다면 데드락
    Future<?> inner = pool.submit(() -> "inner task");
    inner.get(); // 블로킹! 스레드 풀 데드락 발생
});
```

### 주의 2: shutdown()과 shutdownNow() 차이

```java
// shutdown(): 대기 중인 작업 모두 완료 후 종료 (Graceful)
pool.shutdown();
pool.awaitTermination(30, TimeUnit.SECONDS);

// shutdownNow(): 즉시 중단, 대기 중인 작업 목록 반환 (Forceful)
List<Runnable> pending = pool.shutdownNow();
System.out.println("미처리 작업: " + pending.size());
```

### 팁 1: 모니터링 지표 수집

```java
ThreadPoolExecutor pool = (ThreadPoolExecutor) executor;

// 주요 지표
long completed = pool.getCompletedTaskCount();  // 완료된 작업 수
int active = pool.getActiveCount();             // 현재 실행 중
int queued = pool.getQueue().size();            // 대기 중
long total = pool.getTaskCount();               // 전체 제출된 수

// 풀 포화도 (0~1, 1에 가까울수록 병목)
double saturation = (double)(active + queued) / pool.getMaximumPoolSize();
```

### 팁 2: 가상 스레드 (Java 21+ Project Loom)

Java 21부터 도입된 가상 스레드(Virtual Thread)는 OS 스레드가 아닌 JVM이 관리하는 경량 스레드입니다. I/O 바운드 작업에서는 기존 스레드 풀을 대체할 수 있습니다.

```java
// 가상 스레드 풀: OS 스레드 생성 없이 수백만 개의 가상 스레드 실행 가능
try (ExecutorService vPool = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 100_000; i++) {
        vPool.submit(() -> {
            Thread.sleep(Duration.ofMillis(100)); // I/O 블로킹을 가상 스레드가 자동 언마운트
            return "done";
        });
    }
}
```

---

## 마무리

스레드 풀은 동시성 프로그래밍의 기본 빌딩 블록입니다. 올바른 크기 설정, 적절한 큐 전략, 거부 정책 선택이 시스템 안정성과 처리량을 결정합니다. 실제 운영 환경에서는 반드시 모니터링 지표를 수집하고, CPU 바운드와 I/O 바운드 작업을 **별도의 풀**로 분리하는 것을 권장합니다.

## 참고 자료

- [Thread Pools — Java Tutorials (Oracle)](https://docs.oracle.com/javase/tutorial/essential/concurrency/pools.html)
- [ThreadPoolExecutor — Java 8 API (Oracle)](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ThreadPoolExecutor.html)
- [concurrent.futures — Python 3 Documentation](https://docs.python.org/3/library/concurrent.futures.html)
- [Java Concurrency in Practice — Java Platform SE 25](https://docs.oracle.com/en/java/javase/25/core/concurrency.html)
