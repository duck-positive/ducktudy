---
layout: post
title: "CSP(Communicating Sequential Processes) 완전 정복: Go·Kotlin 채널의 수학적 토대"
date: 2026-09-02
categories: [cs, computer-science]
tags: [csp, concurrency, channels, go, goroutines, process-algebra, deadlock, select]
---

## 개념 설명

CSP(Communicating Sequential Processes)는 1978년 Tony Hoare가 CACM(Communications of the ACM)에 발표한 동시성 이론입니다. 이후 1985년 동명의 책으로 발전하였으며, Go 언어의 동시성 모델을 비롯해 Erlang, Kotlin Coroutines, Clojure core.async의 설계 철학에 직접적인 영향을 주었습니다.

CSP의 핵심 철학은 단 한 문장으로 요약됩니다:

> **"메모리를 공유하여 통신하지 말고, 통신하여 메모리를 공유하라."**
> — Rob Pike (Go 언어 공동 창시자)

### CSP의 기본 구성 요소

CSP는 세 가지 추상 개념으로 구성됩니다.

| 구성 요소 | 설명 | Go에서의 대응 |
|-----------|------|--------------|
| **프로세스(Process)** | 순차적으로 실행되는 독립 계산 단위 | goroutine |
| **채널(Channel)** | 프로세스 간 메시지 전달 매체 | channel (`chan`) |
| **이벤트(Event)** | 채널을 통한 동기화 지점 | send/receive 연산 |

### 랑데부(Rendezvous) 동기화

CSP의 채널 통신은 기본적으로 **동기식(synchronous)** 입니다. 송신자와 수신자가 모두 준비되었을 때 비로소 통신(이벤트)이 발생합니다. 이를 랑데부(rendezvous)라고 합니다.

```
프로세스 A          채널 C          프로세스 B
    |                |                  |
  send(C, v) ------→[블록]             |
    |         (수신자 대기)             |
    |                ←-------------- recv(C)
    |                v                  |
  [해제]       이벤트 발생         v 수신 완료
    |                                   |
```

비동기 채널(버퍼 채널)은 이 이론 위에 버퍼링 레이어를 추가한 확장입니다.

## 왜 필요한가

### 공유 메모리 방식의 문제점

전통적인 멀티스레드 프로그래밍은 공유 메모리(shared memory)와 락(mutex)으로 동시성을 제어합니다. 이는 근본적인 복잡성을 낳습니다.

```java
// 전통적 공유 메모리 방식: 레이스 컨디션 위험
class BankAccount {
    private int balance = 1000;
    private final Object lock = new Object();

    public void withdraw(int amount) {
        synchronized (lock) {
            if (balance >= amount) {
                balance -= amount;   // 임계 구역
            }
        }
    }

    public void deposit(int amount) {
        synchronized (lock) {
            balance += amount;   // 임계 구역
        }
    }
}
```

- **데드락** — 두 스레드가 서로 상대방의 락을 기다리는 교착 상태
- **레이스 컨디션** — 실행 순서에 따라 결과가 달라지는 비결정론적 동작
- **락 조합의 어려움** — 여러 락을 올바른 순서로 획득·해제하는 코드 작성이 매우 어려움
- **테스트 어려움** — 동시성 버그는 재현성이 낮아 디버깅이 극도로 어려움

CSP 방식은 **상태를 특정 goroutine이 소유**하도록 설계하여, 공유 상태 자체를 제거합니다.

### CSP의 장점

1. **구성 가능성(Composability)** — 독립적인 프로세스를 채널로 연결해 복잡한 시스템을 조립
2. **이해 가능성** — 각 goroutine이 무엇을 소유하는지 명확하게 추론 가능
3. **형식 검증** — CSP 이론 기반의 도구(FDR, SPIN)로 데드락·라이브락을 수학적으로 검증
4. **선형 확장** — 프로세스 수를 늘리는 것만으로 처리량을 늘릴 수 있는 구조

## 실제 구현 예제

### 예제 1: Go의 기본 채널과 goroutine

```go
package main

import (
    "fmt"
    "sync"
)

// Pipeline 패턴: 생산자 → 필터 → 소비자
func producer(out chan<- int, nums ...int) {
    defer close(out)
    for _, n := range nums {
        out <- n  // CSP 이벤트: 채널에 값 전송
    }
}

func square(in <-chan int, out chan<- int) {
    defer close(out)
    for n := range in {
        out <- n * n  // 입력 채널에서 읽어 변환 후 출력 채널로 전송
    }
}

func printer(in <-chan int, wg *sync.WaitGroup) {
    defer wg.Done()
    for v := range in {
        fmt.Printf("결과: %d\n", v)
    }
}

func main() {
    nums := make(chan int)
    squares := make(chan int)
    var wg sync.WaitGroup

    wg.Add(1)
    go producer(nums, 1, 2, 3, 4, 5)
    go square(nums, squares)
    go printer(squares, &wg)

    wg.Wait()
    // 결과: 1, 4, 9, 16, 25
}
```

### 예제 2: select로 구현하는 CSP 가드 커맨드(Guarded Command)

Hoare의 원래 CSP 논문에서 핵심 개념 중 하나는 **가드된 커맨드(Guarded Command)**입니다. Go의 `select`문이 이를 직접 구현합니다.

```go
package main

import (
    "fmt"
    "time"
)

// 두 채널 중 먼저 준비되는 곳에서 수신하는 비결정론적 선택
func fanIn(ch1, ch2 <-chan string) <-chan string {
    merged := make(chan string)
    go func() {
        defer close(merged)
        for {
            select {
            case v, ok := <-ch1:
                if !ok {
                    ch1 = nil  // 닫힌 채널 비활성화
                } else {
                    merged <- v
                }
            case v, ok := <-ch2:
                if !ok {
                    ch2 = nil
                } else {
                    merged <- v
                }
            }
            if ch1 == nil && ch2 == nil {
                return
            }
        }
    }()
    return merged
}

// 타임아웃 패턴: CSP의 "타임아웃 이벤트"
func requestWithTimeout(ch <-chan string, timeout time.Duration) (string, bool) {
    select {
    case result := <-ch:
        return result, true
    case <-time.After(timeout):
        return "", false  // 타임아웃
    }
}

// Done 채널 패턴: CSP의 취소(cancellation) 이벤트
func worker(done <-chan struct{}, jobs <-chan int, results chan<- int) {
    for {
        select {
        case <-done:
            fmt.Println("워커 취소됨")
            return
        case j, ok := <-jobs:
            if !ok {
                return
            }
            results <- j * j
        }
    }
}

func main() {
    done := make(chan struct{})
    jobs := make(chan int, 5)
    results := make(chan int, 5)

    go worker(done, jobs, results)

    for i := 1; i <= 3; i++ {
        jobs <- i
    }
    close(jobs)

    for i := 0; i < 3; i++ {
        fmt.Printf("결과: %d\n", <-results)
    }
    close(done)
}
```

### 예제 3: 상태 소유권 패턴으로 뮤텍스 대체

```go
package main

import "fmt"

// BankAccount 상태를 goroutine이 단독 소유
// 외부에서는 채널로만 접근 → 뮤텍스 불필요
type BankAccount struct {
    deposit  chan int
    withdraw chan int
    balance  chan int
}

func NewBankAccount(initial int) *BankAccount {
    acc := &BankAccount{
        deposit:  make(chan int),
        withdraw: make(chan int),
        balance:  make(chan int),
    }

    // 상태를 독점하는 goroutine — 이 goroutine만 balance 변수에 접근
    go func() {
        balance := initial
        for {
            select {
            case amount := <-acc.deposit:
                balance += amount
            case amount := <-acc.withdraw:
                if balance >= amount {
                    balance -= amount
                }
            case acc.balance <- balance:
                // 잔액 쿼리: 채널로 현재 잔액 전달
            }
        }
    }()

    return acc
}

func (a *BankAccount) Deposit(amount int) { a.deposit <- amount }
func (a *BankAccount) Withdraw(amount int) { a.withdraw <- amount }
func (a *BankAccount) Balance() int        { return <-a.balance }

func main() {
    acc := NewBankAccount(1000)
    acc.Deposit(500)
    acc.Withdraw(200)
    fmt.Printf("잔액: %d원\n", acc.Balance()) // 1300원
}
```

### 예제 4: FDR을 이용한 데드락 형식 검증

CSP의 가장 강력한 장점 중 하나는 **모델 검사(model checking)** 도구를 사용해 데드락을 수학적으로 검증할 수 있다는 것입니다. FDR(Failures-Divergences Refinement)은 CSP의 공식 모델 검사기입니다.

```
-- FDR/CSP# 언어로 작성한 간단한 프로토콜 검증 예시
-- 식사하는 철학자 문제 (교착 상태 검증)

channel pickup, putdown : {0..4}

PHIL(i) = pickup.i → pickup.((i+1)%5) → putdown.i → putdown.((i+1)%5) → PHIL(i)

FORK(i) = pickup.i → putdown.i → FORK(i)
        [] pickup.((i+1)%5) → putdown.((i+1)%5) → FORK(i)

DINING = (|| i : {0..4} @ PHIL(i)) [| {|pickup, putdown|} |]
         (|| i : {0..4} @ FORK(i))

-- 데드락 자유 검증:
-- assert DINING :[deadlock free [F]]
-- FDR 결과: FAILED — 교착 상태 발견
-- 반례: phi0, phi1, phi2, phi3, phi4가 모두 왼쪽 포크를 집으면 교착

-- 해결책: 하나의 철학자가 오른쪽 포크를 먼저 집도록 수정
PHIL_FIXED(i) =
    if i == 0
    then pickup.((i+1)%5) → pickup.i → putdown.i → putdown.((i+1)%5) → PHIL_FIXED(i)
    else pickup.i → pickup.((i+1)%5) → putdown.i → putdown.((i+1)%5) → PHIL_FIXED(i)

-- assert PHIL_FIXED(i) 기반 DINING :[deadlock free [F]]
-- FDR 결과: PASSED — 데드락 없음
```

## CSP 이론의 의미론

CSP는 세 가지 수학적 의미론을 가집니다.

### 1. 추적(Trace) 의미론

프로세스가 수행한 이벤트 시퀀스의 집합. 가장 단순하며 안전성(safety) 속성 검증에 사용됩니다.

### 2. 실패(Failure) 의미론

프로세스가 수행한 추적과, 그 상태에서 거부할 수 있는 이벤트 집합의 쌍. 데드락 검증에 사용됩니다.

### 3. 실패-발산(Failure-Divergence) 의미론

실패 의미론에 무한 내부 행동(liveleak)을 추가한 의미론. FDR의 기본 검증 도메인입니다.

## CSP와 Actor 모델의 차이

Go의 CSP와 Erlang/Akka의 Actor 모델은 모두 메시지 패싱 기반이지만 구조적 차이가 있습니다.

| 특성 | CSP (Go) | Actor Model (Erlang/Akka) |
|------|----------|--------------------------|
| 통신 대상 | **채널** (익명 통신) | **액터** (주소 지정 통신) |
| 동기화 | 기본 동기식, 옵션 비동기 | 항상 비동기 |
| 메일박스 | 없음 (채널이 큐) | 액터마다 내장 메일박스 |
| 취소 | `done` 채널 패턴 | `PoisonPill` 메시지 |
| 위치 투명성 | 없음 (주로 단일 프로세스) | 네트워크 투명 |

## 주의사항과 팁

1. **goroutine 누수** — 채널이 닫히지 않으면 수신 goroutine이 영원히 블록됩니다. 항상 `defer close(ch)` 패턴을 사용하세요.

2. **방향성 채널** — `chan<- T` (송신 전용), `<-chan T` (수신 전용)을 함수 시그니처에 명시하면 컴파일러가 잘못된 사용을 잡아줍니다.

3. **nil 채널** — `nil` 채널에 대한 송수신은 영원히 블록됩니다. `select`문에서 채널을 비활성화할 때 유용하게 활용할 수 있습니다.

4. **버퍼 크기 결정** — 버퍼 채널의 크기는 "생산자와 소비자 사이의 최대 허용 지연"을 의미합니다. 과도한 버퍼는 역압(backpressure) 신호를 숨기는 부작용이 있습니다.

5. **`context` 패키지** — Go 1.7부터 도입된 `context.Context`는 `done` 채널 패턴을 표준화한 것입니다. 타임아웃·취소 전파에 항상 사용하세요.

## 참고 자료

- [Go Concurrency Patterns — Rob Pike Talk](https://github.com/nicholasjackson/gopher-examples)
- [Z3 Prover (SMT Solver for CSP 검증)](https://github.com/Z3Prover/z3)
- [angr Framework — 프로세스 분석 도구](https://github.com/angr/angr)
- [profunctor-optics Haskell 패키지 (프로세스 대수 관련 타입 이론)](https://hackage.haskell.org/package/profunctor-optics)
