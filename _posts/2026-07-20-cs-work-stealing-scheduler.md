---
layout: post
title: "Work-Stealing 스케줄러 완전 정복: Go·Java ForkJoin·Tokio가 선택한 병렬 태스크 분배 알고리즘"
date: 2026-07-20
categories: [cs, computer-science]
tags: [work-stealing, scheduler, concurrency, go, java, forkjoin, tokio, parallel-computing]
---

현대의 멀티코어 CPU를 최대한 활용하려면 스레드(또는 고루틴)들이 골고루 바쁘게 일해야 합니다. 그런데 작업을 균등하게 나눠놓아도 런타임에 불균형이 생기는 일은 피할 수 없습니다. 어떤 스레드는 큰 서브트리를 처리하느라 한참 걸리고, 다른 스레드는 이미 손을 놓고 쉬고 있을 수도 있죠. **Work-Stealing(작업 훔치기)** 스케줄러는 이 문제를 근본적으로 해결합니다. 놀고 있는 스레드가 직접 바쁜 스레드의 큐에서 작업을 가져오는 발상으로, Go의 goroutine 스케줄러, Java의 `ForkJoinPool`, Rust의 Tokio 런타임, Intel TBB 등이 이 알고리즘을 채택하고 있습니다.

## 개념 설명

### Work-Sharing vs Work-Stealing

병렬 스케줄링 전략은 크게 두 가지로 나뉩니다.

- **Work-Sharing(작업 나눠주기):** 새 태스크가 생겨날 때 스케줄러가 중앙에서 다른 유휴 프로세서에 태스크를 *밀어넣는(push)* 방식입니다. 태스크를 밀어보낼 때마다 커뮤니케이션 비용이 발생합니다.
- **Work-Stealing(작업 훔치기):** 각 프로세서는 자신만의 데크(deque, double-ended queue)를 갖고 자신의 큐에서 작업을 처리합니다. 유휴 상태가 되면 *다른 프로세서의 큐 끝에서 작업을 가져옵니다(steal)*. 훔치는 행위 자체가 이미 있어도 되고 없어도 되는 예외적 상황이므로, 경쟁이 적습니다.

이론적으로 Work-Stealing은 Work-Sharing보다 통신 비용이 최대 2배 적습니다 (Blumofe & Leiserson, 1999). 직관적으로도, 데크의 소유자는 앞쪽(head)에서 추가·삭제하고, 도둑(thief)은 반대쪽(tail)에서 훔쳐가므로 **CAS(Compare-And-Swap) 원자 연산 한 번**으로 충돌 없이 처리할 수 있습니다.

### 데크(Deque) 구조

```
소유자 스레드 (head 쪽):
  push(new_task) → head로 넣기
  pop()          → head에서 꺼내기 (LIFO)

도둑 스레드 (tail 쪽):
  steal()        → tail에서 훔치기 (FIFO)
```

소유자가 LIFO로 동작하는 이유는 **캐시 지역성(cache locality)** 때문입니다. 가장 최근에 생성된 태스크는 보통 부모 태스크와 같은 데이터를 공유하므로, 캐시에 따뜻하게 남아 있을 가능성이 높습니다. 도둑이 tail(오래된 태스크)을 가져가므로 두 스레드 간의 작업 집합(working set) 충돌도 최소화됩니다.

### Chase-Lev 알고리즘

현대 Work-Stealing 구현의 표준은 Chase와 Lev(2005)가 제안한 락-프리(lock-free) 데크입니다. 인덱스를 원형 배열(circular array)로 관리하며, 소유자의 push/pop은 아토믹 읽기/쓰기로, 도둑의 steal은 CAS로 동시성을 보장합니다. 이를 통해 경쟁이 없는 경우 push/pop은 O(1)이며, 충돌 시에도 retry만 하면 됩니다.

---

## 왜 필요한가

### 재귀 분할 정복(Divide-and-Conquer) 에서의 불균형

병렬 퀵소트, 병렬 머지소트, 트리 탐색 등은 런타임 전에 각 서브태스크의 크기를 예측하기 어렵습니다. 정적으로 N개 스레드에 N등분하면 어떤 파티션이 편향(skewed)되는 순간 부하 불균형이 발생합니다.

Work-Stealing은 이런 동적 불균형을 스스로 해소합니다. 한 스레드가 큰 서브트리를 처리하는 동안, 다른 스레드들이 그 큐에서 남은 서브트리를 훔쳐 병렬 처리합니다.

### I/O 바운드 혼합 워크로드

goroutine이나 async task처럼 I/O를 기다리는 태스크가 섞여 있으면, 특정 스레드에 I/O 완료 콜백이 몰리거나 반대로 CPU 집약적 태스크가 한 쪽에 쏠릴 수 있습니다. Work-Stealing은 이 불균형도 자동으로 완화합니다.

---

## 실제 구현 예제

### 예제 1 — Go: GOMAXPROCS와 goroutine 스케줄러

Go 런타임의 스케줄러(GMP 모델)는 Work-Stealing을 핵심으로 삼습니다.
- **G(Goroutine)**: 실행 단위
- **M(Machine)**: OS 스레드
- **P(Processor)**: 논리 프로세서로, 각각 로컬 runqueue를 보유

```go
package main

import (
	"fmt"
	"runtime"
	"sync"
	"sync/atomic"
	"time"
)

func parallelSum(data []int64) int64 {
	n := len(data)
	if n == 0 {
		return 0
	}
	// 충분히 작으면 직접 계산 (베이스 케이스)
	if n <= 512 {
		var s int64
		for _, v := range data {
			s += v
		}
		return s
	}

	mid := n / 2
	var left, right int64
	var wg sync.WaitGroup
	wg.Add(2)

	go func() {
		defer wg.Done()
		// 이 goroutine은 현재 P의 로컬 큐에 쌓임
		// 다른 P가 유휴 상태면 steal 해감
		left = parallelSum(data[:mid])
	}()
	go func() {
		defer wg.Done()
		right = parallelSum(data[mid:])
	}()

	wg.Wait()
	return left + right
}

func main() {
	// P의 수 = CPU 코어 수 (Work-Stealing pool 크기)
	procs := runtime.GOMAXPROCS(0)
	fmt.Printf("GOMAXPROCS = %d\n", procs)

	const N = 10_000_000
	data := make([]int64, N)
	for i := range data {
		data[i] = int64(i + 1)
	}

	start := time.Now()
	result := parallelSum(data)
	elapsed := time.Since(start)

	// 등차수열 합 공식으로 검증
	expected := int64(N) * int64(N+1) / 2
	fmt.Printf("합계: %d (기댓값: %d), 소요시간: %v\n", result, expected, elapsed)

	// goroutine 스케줄러 통계 출력 (Go 1.23+)
	var stats runtime.MemStats
	runtime.ReadMemStats(&stats)
	_ = atomic.LoadInt64(&result) // 컴파일러 최적화 방지
	fmt.Printf("GC 횟수: %d\n", stats.NumGC)
}
```

Go 런타임 내부에서 P가 로컬 runqueue를 비우면 `runtime.schedule()` 함수가 글로벌 큐나 다른 P의 큐에서 **절반(half)** 의 goroutine을 훔쳐옵니다(`stealWork` 함수, `proc.go` 참조). 무작위 순서로 P를 순회하여 핫스팟이 생기지 않도록 설계되어 있습니다.

### 예제 2 — Java: ForkJoinPool로 병렬 병합 정렬

Java 7부터 포함된 `ForkJoinPool`은 Work-Stealing을 표준 라이브러리 수준에서 제공합니다. `RecursiveTask<T>`를 상속해 분할-정복 태스크를 표현합니다.

```java
import java.util.Arrays;
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.RecursiveAction;

public class ParallelMergeSort extends RecursiveAction {
    private static final int THRESHOLD = 2048;
    private final int[] array;
    private final int from;
    private final int to;

    public ParallelMergeSort(int[] array, int from, int to) {
        this.array = array;
        this.from  = from;
        this.to    = to;
    }

    @Override
    protected void compute() {
        int length = to - from;
        if (length <= THRESHOLD) {
            // 작은 부분 배열은 순차 정렬
            Arrays.sort(array, from, to);
            return;
        }
        int mid = from + length / 2;

        // fork(): 현재 스레드의 데크 head에 push
        // 다른 스레드가 steal 가능한 상태가 됨
        ParallelMergeSort left  = new ParallelMergeSort(array, from, mid);
        ParallelMergeSort right = new ParallelMergeSort(array, mid, to);

        left.fork();    // 왼쪽 서브태스크를 큐에 넣고 비동기 실행
        right.compute();// 오른쪽은 현재 스레드가 직접 처리 (tail 자리 차지 방지)
        left.join();    // 왼쪽 결과 대기

        merge(array, from, mid, to);
    }

    private static void merge(int[] arr, int from, int mid, int to) {
        int[] temp = Arrays.copyOfRange(arr, from, to);
        int i = 0, j = mid - from, k = from;
        while (i < mid - from && j < to - from) {
            arr[k++] = temp[i] <= temp[j] ? temp[i++] : temp[j++];
        }
        while (i < mid - from) arr[k++] = temp[i++];
        while (j < to   - from) arr[k++] = temp[j++];
    }

    public static void main(String[] args) {
        int[] data = new int[10_000_000];
        java.util.Random rng = new java.util.Random(42);
        for (int i = 0; i < data.length; i++) data[i] = rng.nextInt();

        // commonPool은 CPU 코어 수 - 1 개의 worker thread 사용
        ForkJoinPool pool = ForkJoinPool.commonPool();
        System.out.printf("Pool 병렬도: %d%n", pool.getParallelism());

        long start = System.currentTimeMillis();
        pool.invoke(new ParallelMergeSort(data, 0, data.length));
        long elapsed = System.currentTimeMillis() - start;

        // 정렬 검증
        boolean sorted = true;
        for (int i = 1; i < data.length; i++) {
            if (data[i] < data[i-1]) { sorted = false; break; }
        }
        System.out.printf("정렬 완료: %b, 소요시간: %dms%n", sorted, elapsed);
        System.out.printf("Steal 횟수: %d%n", pool.getStealCount());
    }
}
```

`fork()`와 `compute()`의 순서가 중요합니다. **오른쪽을 `fork()`하고 왼쪽을 `compute()`하는 것보다**, 왼쪽을 `fork()`하고 오른쪽을 `compute()`하면 현재 스레드가 즉시 처리할 일이 생겨 스케줄러 부담이 줄어듭니다. `pool.getStealCount()`로 실제 Work-Stealing 횟수를 확인할 수 있습니다.

---

## 주의사항 및 팁

### 1. THRESHOLD 튜닝이 성능을 좌우한다

분할 임계값(THRESHOLD)이 너무 작으면 태스크 생성/스케줄링 오버헤드가 지배적이 됩니다. 너무 크면 work-stealing 기회가 줄어들어 부하 불균형이 생깁니다. 일반적으로 **수 KB~수십 KB의 실제 처리량** 단위를 한 태스크로 설정하는 것이 경험적으로 적합합니다.

### 2. 태스크가 블로킹 I/O를 포함하면 위험하다

ForkJoinPool의 기본 스레드 수는 CPU 코어 수 기반입니다. 태스크 안에서 `Thread.sleep()`이나 블로킹 소켓 I/O를 호출하면 모든 worker thread가 블로킹되어 데드락에 가까운 상황이 발생할 수 있습니다. 이런 경우 `ForkJoinPool.ManagedBlocker`를 사용하거나, 블로킹 작업은 별도 `ExecutorService`에 위임해야 합니다.

### 3. Go의 goroutine 파킹과 Work-Stealing

Go에서 고루틴이 `syscall`에 진입하면 M(OS 스레드)이 P에서 분리됩니다. 런타임이 대기 중인 M을 깨우거나 새 M을 생성해 P를 재할당합니다. 이 과정에서도 P의 로컬 큐는 유지되어 Work-Stealing이 계속 동작합니다.

### 4. 메모리 사용 패턴

Work-Stealing 데크는 동적으로 크기를 늘려갑니다(Chase-Lev의 circular array doubling). 태스크를 매우 빠르게 생성하는 경우 메모리 사용량이 예상치 못하게 증가할 수 있습니다. Go의 경우 goroutine 스택도 동적으로 확장되므로 수백만 goroutine 생성은 메모리 압력을 유발합니다. **배압(backpressure)** 을 위해 세마포어 패턴(`chan struct{}` 버퍼 채널)으로 동시 goroutine 수를 제한하는 것이 좋습니다.

### 5. False Sharing 주의

Work-Stealing 큐의 top/bottom 인덱스가 같은 캐시 라인에 있으면 소유자와 도둑이 매번 캐시 라인을 경쟁합니다. 실제 구현체들은 패딩(padding)으로 이를 방지합니다.

```java
// 잘못된 예: top과 bottom이 같은 객체에 연속 배치
class BadDeque { volatile int top; volatile int bottom; }

// 올바른 예: 캐시 라인(64바이트) 패딩
@Contended  // Java 8+, -XX:-RestrictContended 필요
class GoodDeque { volatile int top; volatile int bottom; }
```

Work-Stealing 스케줄러는 "놀고 있는 것이 비용"이라는 인식에서 출발합니다. 중앙 집중식 분배 대신 각 스레드가 주도적으로 일을 찾아 나서도록 설계함으로써, 동적이고 불규칙한 병렬 워크로드에서도 CPU 활용률을 극대화합니다.

## 참고 자료

- [Go runtime package — pkg.go.dev](https://pkg.go.dev/runtime)
- [Work stealing — Wikipedia](https://en.wikipedia.org/wiki/Work_stealing)
- [Guide to Work Stealing in Java — Baeldung](https://www.baeldung.com/java-work-stealing)
- [Go's work-stealing scheduler — rakyll.org](https://rakyll.org/scheduler/)
