---
layout: post
title: "Work Stealing 스케줄러 완전 정복: Java ForkJoinPool과 Go Runtime이 선택한 병렬 부하 분산 전략"
date: 2026-09-15
categories: [cs, computer-science]
tags: [concurrency, scheduler, work-stealing, java, go, forkjoin, parallelism, algorithm]
---

멀티코어 CPU를 최대한 활용하는 병렬 프로그래밍에서 가장 어려운 문제 중 하나는 **부하 분산(Load Balancing)**이다. 재귀적으로 분해되는 작업들을 여러 CPU 코어에 골고루 배분하면서, 동기화 오버헤드는 최소화해야 한다. **Work Stealing(작업 훔치기)**은 이 문제에 대한 우아하고 효율적인 해답으로, Java의 ForkJoinPool, Go의 GMP 스케줄러, Rust의 Tokio, Cilk 병렬 언어 등 현대 런타임 시스템의 핵심 기술이 되었다.

## Work Stealing이란

Work Stealing은 1994년 Blumofe와 Leiserson이 고안한 알고리즘으로, 각 워커 스레드가 자체 **이중 종단 큐(Deque, 덱)**를 가지는 방식이다. 핵심 아이디어는 다음과 같다:

- **작업 생성(Push)**: 자신의 덱 앞(head)에 새 서브태스크를 추가
- **작업 소비(Pop)**: 자신의 덱 앞(head)에서 다음 작업을 꺼냄 (LIFO 순서)
- **작업 훔치기(Steal)**: 덱이 비면 랜덤하게 선택한 다른 워커의 덱 **뒤(tail)**에서 작업을 가져옴

왜 자신은 앞에서, 도둑은 뒤에서 가져갈까? 이것이 Work Stealing의 핵심 통찰이다.

### LIFO 소비와 FIFO 훔치기

**자기 자신 (LIFO, 스택처럼)**:
- 최근에 생성된 "작은" 서브태스크를 먼저 처리
- 캐시 지역성(cache locality) 극대화: 방금 분해한 데이터가 CPU 캐시에 있을 가능성이 높다
- 재귀 트리의 깊이를 먼저 탐색 → 메모리 사용량 최소화

**도둑 (FIFO, 큐처럼)**:
- 덱의 반대쪽 끝에서 "오래된" 작업을 훔침
- 오래된 작업은 아직 분해되지 않은 "크고 굵은" 서브트리일 가능성이 높다
- 훔친 작업 하나로 도둑 스레드가 오랫동안 바쁘게 일할 수 있음 → 훔치기 빈도 감소

이 비대칭 접근이 Work Stealing의 효율성을 만든다.

## 왜 Work Stealing이 필요한가

### 정적 분할의 문제

단순한 병렬화 전략은 입력을 N등분하여 N개 스레드에 배분하는 것이다. 하지만 재귀적 알고리즘(퀵소트, 피보나치, 트리 탐색)이나 입력 크기를 사전에 예측할 수 없는 경우, 각 파티션의 작업량이 불균등해진다. 한 코어가 90%의 작업을 처리하는 동안 다른 코어들이 놀고 있다면 병렬화의 의미가 없다.

### 동적 스틸 전략의 이점

Work Stealing은 **풀 방식(pull-based)** 동적 부하 분산이다. 바쁜 워커는 자신의 작업에 집중하고(push 오버헤드 없음), 할 일이 없는 워커만 능동적으로 작업을 가져온다. 중앙 집중형 큐(모든 워커가 하나의 큐에서 경쟁)와 달리, 덱은 워커별로 분리되어 있어 대부분의 경우 **무락(lock-free) 연산**으로 작동한다.

## 코드 예제 1: Java ForkJoinPool로 병렬 병합 정렬

```java
import java.util.Arrays;
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.RecursiveAction;

public class ParallelMergeSort {
    
    static class MergeSortTask extends RecursiveAction {
        private static final int THRESHOLD = 1024; // 시퀀셜 임계값
        private final int[] array;
        private final int lo, hi;
        
        MergeSortTask(int[] array, int lo, int hi) {
            this.array = array;
            this.lo = lo;
            this.hi = hi;
        }
        
        @Override
        protected void compute() {
            if (hi - lo <= THRESHOLD) {
                // 작은 범위는 시퀀셜 정렬 (오버헤드 방지)
                Arrays.sort(array, lo, hi);
                return;
            }
            
            int mid = (lo + hi) >>> 1;
            
            // 왼쪽 절반을 Fork: 현재 워커의 덱 앞에 추가
            MergeSortTask left  = new MergeSortTask(array, lo, mid);
            MergeSortTask right = new MergeSortTask(array, mid, hi);
            
            // fork(): 현재 스레드의 덱에 left 태스크 추가
            left.fork();
            
            // 오른쪽은 현재 스레드에서 직접 실행 (최적화)
            right.compute();
            
            // join(): left 완료 대기. 완료 전이면 해당 스레드는
            // 덱에서 다른 작업을 꺼내 실행 (대기하지 않고 일함!)
            left.join();
            
            // 두 정렬된 부분 합병
            merge(array, lo, mid, hi);
        }
        
        private void merge(int[] arr, int lo, int mid, int hi) {
            int[] temp = Arrays.copyOfRange(arr, lo, hi);
            int i = 0, j = mid - lo, k = lo;
            
            while (i < mid - lo && j < hi - lo) {
                if (temp[i] <= temp[j]) arr[k++] = temp[i++];
                else                    arr[k++] = temp[j++];
            }
            while (i < mid - lo) arr[k++] = temp[i++];
            while (j < hi - lo)  arr[k++] = temp[j++];
        }
    }
    
    public static void main(String[] args) {
        int size = 10_000_000;
        int[] data = new int[size];
        java.util.Random rng = new java.util.Random(42);
        for (int i = 0; i < size; i++) data[i] = rng.nextInt();
        
        // 기본 commonPool: CPU 코어 수 - 1개의 워커 스레드
        ForkJoinPool pool = ForkJoinPool.commonPool();
        System.out.printf("워커 스레드 수: %d%n", pool.getParallelism());
        
        long start = System.currentTimeMillis();
        pool.invoke(new MergeSortTask(data, 0, size));
        long elapsed = System.currentTimeMillis() - start;
        
        // 정렬 검증
        boolean sorted = true;
        for (int i = 1; i < size; i++) {
            if (data[i] < data[i-1]) { sorted = false; break; }
        }
        System.out.printf("정렬 완료: %s (%d ms)%n", sorted ? "성공" : "실패", elapsed);
        
        // ForkJoinPool 통계 (디버깅용)
        System.out.println("풀 상태:");
        System.out.printf("  훔치기 횟수: %d%n", pool.getStealCount());
        System.out.printf("  큐 크기: %d%n", pool.getQueuedTaskCount());
    }
}
```

## 코드 예제 2: Work Stealing 덱 직접 구현 (Python)

```python
import threading
import random
import time
from collections import deque
from typing import Optional, Callable

class WorkStealingDeque:
    """
    Chase-Lev(2005) 스타일 Work Stealing 덱의 간소화 구현.
    실제 구현은 Lock-free CAS를 사용하지만, 여기서는 lock으로 단순화.
    """
    
    def __init__(self):
        self._deque = deque()
        self._lock = threading.Lock()
    
    def push(self, task):
        """소유 스레드만 호출: 앞에 추가 (O(1))"""
        with self._lock:
            self._deque.appendleft(task)
    
    def pop(self) -> Optional[Callable]:
        """소유 스레드만 호출: 앞에서 꺼냄 (LIFO)"""
        with self._lock:
            if self._deque:
                return self._deque.popleft()
            return None
    
    def steal(self) -> Optional[Callable]:
        """다른 스레드가 호출: 뒤에서 훔침 (FIFO)"""
        with self._lock:
            if len(self._deque) > 1:  # 1개 이상 있을 때만 훔침
                return self._deque.pop()
            return None
    
    def __len__(self):
        return len(self._deque)


class WorkStealingScheduler:
    """멀티스레드 Work Stealing 스케줄러"""
    
    def __init__(self, num_workers: int):
        self.num_workers = num_workers
        self.deques = [WorkStealingDeque() for _ in range(num_workers)]
        self.results = []
        self.results_lock = threading.Lock()
        self._running = True
        self.steal_count = [0] * num_workers
        self.exec_count = [0] * num_workers
    
    def submit(self, task: Callable, worker_id: int = 0):
        """초기 태스크 제출"""
        self.deques[worker_id].push(task)
    
    def _worker_loop(self, worker_id: int):
        """워커 스레드 메인 루프"""
        my_deque = self.deques[worker_id]
        
        while self._running:
            # 1. 자신의 덱에서 작업 꺼내기
            task = my_deque.pop()
            
            if task is None:
                # 2. 빈 경우: 랜덤 피해자 선택하여 훔치기
                victim_id = random.randint(0, self.num_workers - 1)
                if victim_id == worker_id:
                    victim_id = (worker_id + 1) % self.num_workers
                
                task = self.deques[victim_id].steal()
                if task:
                    self.steal_count[worker_id] += 1
                else:
                    # 모든 덱이 비었으면 잠시 대기 (스핀 방지)
                    time.sleep(0.0001)
                    continue
            
            # 3. 작업 실행
            result = task()
            self.exec_count[worker_id] += 1
            
            if result is not None:
                with self.results_lock:
                    self.results.append(result)
    
    def run(self, timeout: float = 5.0) -> list:
        """스케줄러 실행"""
        threads = []
        for i in range(self.num_workers):
            t = threading.Thread(target=self._worker_loop, args=(i,), daemon=True)
            t.start()
            threads.append(t)
        
        time.sleep(timeout)
        self._running = False
        
        for t in threads:
            t.join(timeout=1.0)
        
        return self.results
    
    def stats(self):
        total_exec = sum(self.exec_count)
        total_steal = sum(self.steal_count)
        print(f"\n=== Work Stealing 스케줄러 통계 ===")
        for i in range(self.num_workers):
            print(f"  Worker {i}: 실행={self.exec_count[i]}, 훔치기={self.steal_count[i]}")
        print(f"  총 실행: {total_exec}, 총 훔치기: {total_steal}")
        if total_exec > 0:
            print(f"  훔치기 비율: {total_steal/total_exec*100:.1f}%")


def fibonacci_task(n: int, scheduler: WorkStealingScheduler, worker_id: int):
    """Work Stealing 방식의 피보나치 계산 태스크 팩토리"""
    if n <= 1:
        return lambda: n
    
    results = []
    lock = threading.Lock()
    
    def compute():
        if n <= 10:  # 임계값 이하: 시퀀셜 계산
            def fib_seq(k):
                a, b = 0, 1
                for _ in range(k): a, b = b, a + b
                return a
            return fib_seq(n)
        
        # 큰 경우: 서브태스크로 분해하여 덱에 추가
        sub_results = []
        
        def left_task():
            val = fibonacci_task(n - 1, scheduler, worker_id)()
            sub_results.append(val)
            return None
        
        def right_task():
            val = fibonacci_task(n - 2, scheduler, worker_id)()
            sub_results.append(val)
            return None
        
        scheduler.deques[worker_id].push(left_task)
        right_task()  # 오른쪽은 즉시 실행
        
        # 간단한 대기 (실제로는 join 메커니즘 사용)
        while len(sub_results) < 2:
            time.sleep(0.00001)
        
        return sum(sub_results)
    
    return compute


# 데모: Work Stealing으로 소수 계산
def demo_prime_sieve():
    """Work Stealing으로 범위를 분할하여 소수 계산"""
    scheduler = WorkStealingScheduler(num_workers=4)
    
    def is_prime(n: int) -> bool:
        if n < 2: return False
        if n == 2: return True
        if n % 2 == 0: return False
        for i in range(3, int(n**0.5) + 1, 2):
            if n % i == 0: return False
        return True
    
    def make_range_task(start: int, end: int):
        def task():
            primes = [n for n in range(start, end) if is_prime(n)]
            return primes
        return task
    
    # 1~100000을 25개 청크로 분할하여 Worker 0에게 할당
    chunk_size = 4000
    for i in range(25):
        start = i * chunk_size + 2
        end = (i + 1) * chunk_size + 2
        scheduler.submit(make_range_task(start, end), worker_id=i % 4)
    
    results = scheduler.run(timeout=2.0)
    
    all_primes = sorted(set(p for chunk in results if isinstance(chunk, list) for p in chunk))
    print(f"발견된 소수 개수: {len(all_primes)}")
    print(f"처음 10개: {all_primes[:10]}")
    print(f"마지막 10개: {all_primes[-10:]}")
    scheduler.stats()


if __name__ == "__main__":
    demo_prime_sieve()
```

## Java ForkJoinPool 내부 구조

### 덱 구현: Chase-Lev 동적 배열 덱

실제 ForkJoinPool은 Chase-Lev(2005) 알고리즘의 변형을 사용한다. 핵심은 **원형 배열(Circular Array)** 기반 덱이다:

- `top` 인덱스: 소유 워커가 push/pop에 사용 (배열 앞쪽)
- `base` 인덱스: 도둑이 steal에 사용 (배열 뒤쪽)
- 배열이 꽉 차면 두 배로 확장 (동적 크기 조정)

push/pop은 단일 스레드(소유자)만 접근하므로 **원자적 연산 없이** 가능하다. steal만 CAS(Compare-And-Swap)가 필요하다:

```
steal():
  b = base
  array = deque.array
  task = array[b % array.size]
  if CAS(base, b, b+1):  // 원자적으로 base 증가
    return task           // 성공: task 반환
  return null             // 실패: 다른 도둑과 경쟁 패배
```

### Go GMP 스케줄러의 Work Stealing

Go 런타임의 스케줄러는 **M:N 모델**이다: M개의 OS 스레드가 N개의 고루틴을 P개의 프로세서 큐에서 실행한다. P(Processor)마다 로컬 고루틴 큐(LRQ, Local Run Queue)가 있고, 전역 큐(GRQ)가 백업으로 존재한다.

Work Stealing 규칙:
1. 자신의 LRQ에서 고루틴 실행
2. LRQ가 비면: GRQ에서 가져오기 (전역 균형)
3. GRQ도 비면: 랜덤 P의 LRQ 절반을 훔치기

"절반 훔치기(steal-half)" 정책이 중요하다. 하나만 훔치면 자주 훔쳐야 하고, 전부 훔치면 피해자가 즉시 할 일이 없어진다. 절반이 최적의 균형이다.

## 임계값 조정: 작업 분해의 적정 크기

Work Stealing의 성능은 **작업 분해 임계값(Granularity Threshold)**에 크게 의존한다.

너무 작은 임계값:
- 너무 많은 서브태스크 → 태스크 객체 생성 오버헤드
- 훔치기 경쟁 증가 → 캐시 무효화(cache invalidation)

너무 큰 임계값:
- 부하 불균형 → 일부 코어가 오래 유휴 상태

경험적 가이드라인:
- Java ForkJoinPool: 배열 크기 기준 1024~4096개 요소
- 태스크 하나당 작업량이 ~100μs 이상이어야 오버헤드 대비 이득

```java
// ForkJoinPool 커스텀 설정
ForkJoinPool customPool = new ForkJoinPool(
    Runtime.getRuntime().availableProcessors(), // 병렬도
    ForkJoinPool.defaultForkJoinWorkerThreadFactory,
    null,   // UncaughtExceptionHandler
    false   // asyncMode: false=LIFO(기본), true=FIFO
);
```

## 주의사항

### 1. 블로킹 태스크 주의

ForkJoinPool의 워커 수는 CPU 코어 수에 맞춰져 있다. ForkJoinTask 안에서 I/O 블로킹이 발생하면 다른 태스크들이 모두 대기해야 한다. `ManagedBlocker` 인터페이스로 이 문제를 완화할 수 있다:

```java
ForkJoinPool.managedBlock(new ForkJoinPool.ManagedBlocker() {
    @Override
    public boolean block() throws InterruptedException {
        result = blockingCall(); // I/O 작업
        return true;
    }
    @Override
    public boolean isReleasable() { return result != null; }
});
```

### 2. ThreadLocal 함정

Work Stealing은 같은 태스크가 다른 스레드에서 실행될 수 있다. `ThreadLocal`에 저장된 상태가 서브태스크에서 보이지 않을 수 있으므로, 태스크 간 공유 상태는 명시적으로 전달해야 한다.

### 3. 예외 처리

`ForkJoinTask.get()`은 `ExecutionException`을 발생시키므로 체계적인 예외 처리가 필요하다. `RecursiveTask<V>` 대신 `RecursiveAction`을 사용하면 반환값이 없어 예외 처리가 단순해진다.

## 정리

Work Stealing은 재귀적 병렬 알고리즘의 이상적인 스케줄링 전략이다. 핵심 아이디어를 기억하자:
- **소유자는 앞(LIFO)**: 캐시 지역성과 재귀 깊이 최소화
- **도둑은 뒤(FIFO)**: 크고 굵은 잔여 작업 훔치기
- **대부분 무경쟁**: 각자의 덱에서 작업하므로 락 경쟁 최소
- **절반 훔치기**: Go식 steal-half로 불균형 신속 해소

Java의 `ForkJoinPool`은 `parallelStream()`, `CompletableFuture.supplyAsync()`, `Arrays.parallelSort()`의 내부 엔진이다. Go의 모든 고루틴 스케줄링은 Work Stealing 위에서 돌아간다. 병렬 프로그래밍을 한다면 이 알고리즘을 이해하는 것은 필수다.

## 참고 자료
- [Baeldung: Guide to Work Stealing in Java](https://www.baeldung.com/java-work-stealing)
- [Go's Work-Stealing Scheduler — rakyll.org](https://rakyll.org/scheduler/)
- [Blumofe & Leiserson: Scheduling Multithreaded Computations by Work Stealing (JACM 1999)](https://dl.acm.org/doi/10.1145/324133.324234)
- [Chase & Lev: Dynamic Circular Work-Stealing Deque (SPAA 2005)](https://dl.acm.org/doi/10.1145/1073970.1073974)
