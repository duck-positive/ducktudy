---
layout: post
title: "Go 런타임 GMP 스케줄러 완전 정복: 고루틴이 CPU를 점유하는 모든 과정"
date: 2026-09-13
categories: [cs, computer-science]
tags: [go, goroutine, scheduler, concurrency, runtime, gmp, work-stealing]
---

Go 언어는 수십만 개의 고루틴(goroutine)을 적은 메모리로 동시에 실행할 수 있다는 점에서 이례적인 언어다. 이를 가능하게 하는 핵심이 바로 **GMP 스케줄러**다. 운영체제 스레드보다 훨씬 가볍고, 사용자 공간에서 동작하며, 작업 훔치기(Work Stealing) 알고리즘으로 CPU 유휴 시간을 최소화한다. 이 글에서는 GMP 모델의 내부 동작 원리를 코드 수준까지 파헤친다.

## GMP 모델의 세 주인공

Go 스케줄러는 세 가지 핵심 개념으로 구성된다.

**G (Goroutine)**: 고루틴은 Go 런타임이 관리하는 경량 실행 단위다. `go func()` 키워드 하나로 생성되며, 초기 스택 크기는 단 2~8KB에 불과하다. 필요에 따라 동적으로 스택이 확장(최대 1GB)되므로 OS 스레드의 고정 스택(보통 1~8MB)과 비교해 훨씬 메모리 효율적이다. G의 상태는 `Grunnable`(실행 준비), `Grunning`(실행 중), `Gwaiting`(대기), `Gdead`(종료) 등으로 세분화된다.

**M (Machine)**: OS 스레드와 1:1 대응되는 실행 엔진이다. 실제 CPU에서 코드를 실행하는 주체로, M은 반드시 P를 획득해야만 고루틴을 실행할 수 있다. Go 런타임은 M의 수를 동적으로 조절하며, 기본 최대값은 10,000개다(`runtime/debug.SetMaxThreads`로 변경 가능). 시스템 콜로 M이 블로킹되면 런타임은 새 M을 생성해 P를 넘겨준다.

**P (Processor)**: 논리적 프로세서로, M과 G를 연결하는 컨텍스트다. `GOMAXPROCS` 환경 변수로 개수를 제어하며, 기본값은 CPU 코어 수다. 각 P는 **로컬 런큐(Local Run Queue, LRQ)**를 갖고 있어 최대 256개의 실행 가능한 고루틴을 보유할 수 있다. P의 수가 실제 병렬 실행 가능한 고루틴의 최대 수를 결정한다.

## 왜 GMP가 필요한가

OS 스레드를 직접 사용하는 방식(Java의 전통적 Thread, C의 pthread)과 비교해 GMP 모델이 필요한 이유를 살펴보자.

**문제 1: OS 스레드의 무거운 컨텍스트 스위칭**  
OS 스레드 간 스위칭은 커널 모드 진입, 레지스터 저장·복원, 페이지 테이블 갱신 등의 작업을 포함해 수 마이크로초가 소요된다. 반면 Go의 고루틴 스위칭은 사용자 공간에서 일어나고 레지스터 저장량도 훨씬 적어 수백 나노초 수준이다.

**문제 2: 스레드 개수의 제한**  
OS 스레드는 커널이 관리하므로 수천 개를 초과하면 시스템 리소스가 고갈된다. Go 런타임은 소수의 OS 스레드(M)로 수십만 개의 고루틴(G)을 멀티플렉싱하여 이 한계를 극복한다.

**문제 3: I/O 블로킹 시 CPU 낭비**  
네트워크 I/O를 기다리는 스레드는 CPU를 점유한 채 블로킹된다. Go 런타임은 netpoller(epoll/kqueue/IOCP 추상화)를 통해 I/O 대기 중인 고루틴을 `Gwaiting` 상태로 전환하고, 완료 시 자동으로 `Grunnable`로 복귀시킨다. 따라서 M은 다른 고루틴을 계속 실행할 수 있다.

## GMP 스케줄러의 실제 작동 원리

### 고루틴 생성과 큐 배치

`go func()` 호출 시 런타임은 `newproc` 함수를 통해 G를 생성하고 현재 P의 로컬 런큐에 넣는다. 로컬 런큐가 가득 차면 절반을 전역 런큐(Global Run Queue, GRQ)로 이동한다.

```go
package main

import (
	"fmt"
	"runtime"
	"sync"
	"time"
)

func main() {
	// CPU 코어 수만큼 P 설정 (기본값)
	numP := runtime.GOMAXPROCS(0)
	fmt.Printf("논리 프로세서(P) 수: %d\n", numP)

	var wg sync.WaitGroup
	results := make([]int, 10)

	// 10개의 고루틴을 생성 — 각 G는 P의 LRQ에 배치됨
	for i := 0; i < 10; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			// CPU 바운드 작업: 이 고루틴은 M 위에서 직접 실행
			sum := 0
			for j := 0; j < 1_000_000; j++ {
				sum += j
			}
			results[id] = sum
		}(i)
	}

	wg.Wait()
	fmt.Printf("완료. 첫 번째 결과: %d\n", results[0])

	// 시스템 콜 블로킹 예시: M이 분리되고 새 M이 P를 인계받음
	done := make(chan struct{})
	go func() {
		// time.Sleep은 내부적으로 gopark를 호출해 G를 Gwaiting으로 전환
		// M은 다른 G를 실행할 수 있음
		time.Sleep(10 * time.Millisecond)
		close(done)
	}()
	<-done
}
```

### 작업 훔치기(Work Stealing) 알고리즘

스케줄러의 핵심 특징은 **작업 훔치기**다. P가 자신의 로컬 런큐를 비우면, 다른 P의 로컬 런큐에서 절반의 고루틴을 훔쳐온다. 이를 통해 CPU 편중을 방지하고 모든 코어를 균등하게 활용한다.

스케줄러가 다음 G를 선택하는 순서는 다음과 같다:

1. **로컬 런큐 확인**: P의 LRQ에서 G 꺼내기  
2. **전역 런큐 확인**: 61번 스케줄링마다 GRQ에서 한 번씩 G 가져오기 (기아 방지)  
3. **Netpoller 확인**: I/O 완료된 G 가져오기  
4. **다른 P에서 훔치기**: 무작위로 선택한 P의 LRQ 절반 훔치기

```go
package main

import (
	"fmt"
	"runtime"
	"sync"
	"sync/atomic"
	"time"
)

// Work Stealing 동작을 관찰하기 위한 예제
func main() {
	// P를 4개로 제한
	runtime.GOMAXPROCS(4)

	var totalProcessed int64
	jobs := make(chan int, 1000)
	var wg sync.WaitGroup

	// 8개의 워커 고루틴 — 4개의 P가 8개의 G를 스케줄링
	// 일부 P의 LRQ가 비면 다른 P에서 G를 훔쳐옴
	for w := 0; w < 8; w++ {
		wg.Add(1)
		go func(workerID int) {
			defer wg.Done()
			localCount := 0
			for job := range jobs {
				// 작업량을 불균등하게 분배하여 Work Stealing 유발
				if job%3 == 0 {
					// 무거운 작업
					time.Sleep(time.Millisecond)
				}
				localCount++
				atomic.AddInt64(&totalProcessed, 1)
			}
			fmt.Printf("워커 %d: %d개 처리\n", workerID, localCount)
		}(w)
	}

	// 1000개 작업 전송
	for i := 0; i < 1000; i++ {
		jobs <- i
	}
	close(jobs)

	wg.Wait()
	fmt.Printf("\n총 처리 건수: %d\n", atomic.LoadInt64(&totalProcessed))
}
```

### 선점형 스케줄링 (Go 1.14+)

초기 Go 스케줄러는 협력형(cooperative)이었다. 고루틴이 함수 호출, 시스템 콜, 채널 작업 등의 안전 지점에서만 스케줄링 기회를 양보했기 때문에, CPU 바운드 고루틴 하나가 P를 독점할 수 있었다.

Go 1.14부터 **비동기 선점(Asynchronous Preemption)**이 도입됐다. `SIGURG` 시그널을 주기적으로 전송해 어떤 지점에서든 고루틴을 강제 전환할 수 있게 됐다. 이를 통해 `for { }` 무한 루프도 다른 고루틴의 실행을 방해하지 않는다.

## 시스템 콜과 M의 분리

시스템 콜(파일 I/O, 소켓 등)이 발생하면 다음 과정이 일어난다:

1. **핸드오프(Handoff)**: M은 자신의 P를 다른 M(대기 중인 M이 없으면 새로 생성)에게 양도한다.
2. **블로킹**: 원래 M은 시스템 콜과 함께 블로킹된다.
3. **복귀**: 시스템 콜 완료 후 M은 새 P를 획득하려 시도한다. 가용 P가 없으면 G를 전역 런큐에 넣고 M은 스레드 캐시로 복귀한다.

netpoller를 통한 비동기 I/O는 이 과정과 다르다. `net.Conn.Read()` 등의 호출은 내부적으로 고루틴을 `Gwaiting`으로 전환한다. M은 분리되지 않고 다른 G를 계속 실행한다. I/O 완료가 감지되면 해당 G는 다시 `Grunnable`로 전환된다.

## 주의사항과 성능 팁

**1. GOMAXPROCS 튜닝**  
CPU 바운드 워크로드는 코어 수와 동일하게 설정한다. I/O 바운드 워크로드는 더 높게 설정해도 되지만, 너무 크면 스케줄러 오버헤드가 증가한다.

**2. 고루틴 누수 주의**  
`Gwaiting` 상태로 영영 깨어나지 못하는 고루틴은 메모리를 점유한다. `goleak` 라이브러리로 테스트 시 누수를 탐지할 수 있다.

**3. `runtime.LockOSThread()` 사용 주의**  
OpenGL, CGO 콜백 등 특정 OS 스레드에 고정이 필요한 경우에만 사용한다. 이 함수는 G와 M을 1:1로 고정해 스케줄러의 유연성을 제거한다.

**4. 채널 크기와 고루틴 수 균형**  
고루틴을 무제한 생성하면 GC 오버헤드와 스케줄러 메타데이터 비용이 증가한다. 워커 풀 패턴으로 고루틴 수를 제한하는 것이 좋다.

**5. `sync.Pool`로 G 할당 최소화**  
고루틴이 반복적으로 생성하는 임시 객체는 `sync.Pool`을 사용해 재활용한다. GC 압력을 줄이면 스케줄러가 GC 중단으로 인한 지연을 겪지 않는다.

## 참고 자료
- [The Go Memory Model](https://go.dev/ref/mem)
- [Go Runtime Scheduler - Golang Documentation](https://pkg.go.dev/runtime)
- [Scalable Go Scheduler Design Doc (Dmitry Vyukov)](https://docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw/edit)
- [Go: Goroutine and Preemption - Vincent Blanchon](https://medium.com/a-journey-with-go/go-goroutine-and-preemption-d6bc2aa2f4b7)
