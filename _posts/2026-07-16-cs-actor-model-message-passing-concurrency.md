---
layout: post
title: "Actor 모델과 메시지 패싱: Erlang·Akka·Go가 선택한 동시성 패러다임 완전 정복"
date: 2026-07-16
categories: [cs, computer-science]
tags: [actor-model, concurrency, erlang, akka, go, message-passing, distributed-systems]
---

## Actor 모델이란 무엇인가

동시성 프로그래밍의 핵심 난제는 **공유 상태(shared state)**다. 여러 스레드가 같은 메모리를 동시에 읽고 쓸 때 발생하는 데이터 레이스(data race)와 교착 상태(deadlock)는 전통적인 락(lock) 기반 동시성의 고질적 문제다.

Actor 모델은 1973년 Carl Hewitt, Peter Bishop, Richard Steiger가 제안한 동시성 계산 모델로, 이 문제를 근본적으로 다른 방식으로 해결한다. **"공유하지 말고 통신하라(Don't communicate by sharing memory; share memory by communicating)"** 는 Go 언어의 격언이지만, Actor 모델의 철학을 정확히 표현한다.

### 모델의 세 가지 원칙

Actor 모델에서 모든 계산의 기본 단위는 **Actor**다. 각 Actor는 다음 세 가지만 할 수 있다:

1. **메시지를 보낸다** — 다른 Actor의 주소(Address)로 비동기 메시지를 전송
2. **새 Actor를 생성한다** — 자식 Actor를 스폰(spawn)
3. **다음 메시지 처리 방식을 결정한다** — 자신의 내부 상태(behavior)를 변경

중요한 것은 Actor들이 **메일박스(mailbox)**를 통해서만 소통한다는 점이다. 공유 메모리에 직접 접근하는 방법이 존재하지 않는다. 메시지는 비동기적으로 전달되고, Actor는 메일박스에서 메시지를 하나씩 순차적으로 처리한다. 이 순차 처리 덕분에 Actor 내부에서는 별도의 동기화가 필요 없다.

---

## 왜 Actor 모델이 필요한가

### 전통적 스레드·락 방식의 문제

```java
// 락 기반 동시성의 함정 — 교착 상태 예시
class BankAccount {
    private long balance;
    private final Object lock = new Object();

    // 계좌 이체: 두 락을 동시에 잡아야 함
    public void transfer(BankAccount target, long amount) {
        synchronized (this.lock) {          // lock A 획득
            synchronized (target.lock) {    // lock B 획득 시도
                // 다른 스레드가 target.transfer(this, ...)를 호출 중이라면?
                // → 서로 상대방의 락을 기다리는 교착 상태 발생
                this.balance -= amount;
                target.balance += amount;
            }
        }
    }
}
```

위 코드에서 Thread-1이 A→B 이체를, Thread-2가 B→A 이체를 동시에 시도하면 서로 상대방의 락을 기다리는 **교착 상태**가 발생한다. 락의 획득 순서를 고정하거나 타임아웃을 사용하는 방어 코드가 필요하지만, 이는 근본적 해결책이 아니다.

### Actor 방식의 해결

Actor 모델에서는 계좌 자체가 Actor다. 잔액은 Actor의 **프라이빗 상태**이며 외부에서 직접 접근할 수 없다. 모든 변경은 메시지를 통해서만 이루어진다.

```
Thread-1 → [Transfer(to=B, amount=100)] → Account A의 메일박스
Account A가 메시지를 처리 → [Debit(100)] → Account B의 메일박스
Account B가 메시지를 처리 → 잔액 증가
```

여러 전송 요청이 동시에 와도 Account A의 메일박스에 차례로 쌓이고, Actor는 하나씩 처리한다. 락이 필요 없다.

---

## 언어별 Actor 모델 구현

### Erlang: 원조 구현체

Erlang은 Actor 모델을 언어 설계에 내재화한 선구자다. Erlang 프로세스는 OS 스레드가 아닌 VM(BEAM) 레벨의 경량 프로세스로, 수십만 개를 동시에 생성할 수 있다.

```erlang
%% Erlang: 은행 계좌 Actor
-module(bank_account).
-export([start/1, deposit/2, withdraw/2, balance/1]).

%% Actor 시작: 초기 잔액으로 프로세스 생성
start(InitialBalance) ->
    spawn(fun() -> loop(InitialBalance) end).

%% 메시지 루프: 메일박스에서 패턴 매칭으로 메시지 처리
loop(Balance) ->
    receive
        {deposit, Amount, From} ->
            NewBalance = Balance + Amount,
            From ! {ok, NewBalance},
            loop(NewBalance);

        {withdraw, Amount, From} when Amount =< Balance ->
            NewBalance = Balance - Amount,
            From ! {ok, NewBalance},
            loop(NewBalance);

        {withdraw, _Amount, From} ->
            From ! {error, insufficient_funds},
            loop(Balance);

        {balance, From} ->
            From ! {balance, Balance},
            loop(Balance)
    end.

%% 클라이언트 API
deposit(Pid, Amount) ->
    Pid ! {deposit, Amount, self()},
    receive {ok, NewBalance} -> NewBalance end.

withdraw(Pid, Amount) ->
    Pid ! {withdraw, Amount, self()},
    receive
        {ok, NewBalance} -> {ok, NewBalance};
        {error, Reason} -> {error, Reason}
    end.

balance(Pid) ->
    Pid ! {balance, self()},
    receive {balance, B} -> B end.

%% 사용 예
%% Account = bank_account:start(1000).
%% bank_account:deposit(Account, 500).    % → 1500
%% bank_account:withdraw(Account, 200).   % → {ok, 1300}
%% bank_account:balance(Account).         % → 1300
```

`receive` 블록은 Erlang의 핵심이다. 메일박스에서 패턴 매칭으로 처리할 메시지를 선택하며, 일치하는 메시지가 없으면 블록된다. `loop/1`을 꼬리 재귀(tail recursion)로 호출함으로써 스택 오버플로우 없이 무한히 실행된다.

### Akka (Scala/Java): JVM 위의 Actor

Akka는 JVM 생태계에서 Actor 모델을 구현한 라이브러리다. Akka Typed API는 타입 안전성을 보장한다.

```scala
import akka.actor.typed._
import akka.actor.typed.scaladsl._

// 메시지 타입 정의 (sealed trait으로 타입 안전성 보장)
sealed trait AccountCommand
case class Deposit(amount: Long, replyTo: ActorRef[AccountResponse]) extends AccountCommand
case class Withdraw(amount: Long, replyTo: ActorRef[AccountResponse]) extends AccountCommand
case class GetBalance(replyTo: ActorRef[AccountResponse]) extends AccountCommand

sealed trait AccountResponse
case class BalanceUpdated(newBalance: Long) extends AccountResponse
case class InsufficientFunds(currentBalance: Long) extends AccountResponse

object BankAccount {
  def apply(initialBalance: Long): Behavior[AccountCommand] =
    active(initialBalance)

  // Actor의 Behavior: 메시지를 받아 다음 Behavior를 반환
  private def active(balance: Long): Behavior[AccountCommand] =
    Behaviors.receive { (ctx, message) =>
      message match {
        case Deposit(amount, replyTo) =>
          val newBalance = balance + amount
          ctx.log.info(s"Deposit $amount, balance: $balance → $newBalance")
          replyTo ! BalanceUpdated(newBalance)
          active(newBalance)  // 새 잔액으로 Behavior 교체 (불변 상태)

        case Withdraw(amount, replyTo) if amount <= balance =>
          val newBalance = balance - amount
          replyTo ! BalanceUpdated(newBalance)
          active(newBalance)

        case Withdraw(_, replyTo) =>
          replyTo ! InsufficientFunds(balance)
          Behaviors.same  // 상태 변화 없음

        case GetBalance(replyTo) =>
          replyTo ! BalanceUpdated(balance)
          Behaviors.same
      }
    }
}

// ActorSystem 생성 및 사용
val system = ActorSystem(BankAccount(1000L), "bank-system")
// system ! Deposit(500, ...)
```

Akka Typed의 핵심은 `Behavior[T]`다. Actor는 메시지를 처리할 때마다 **다음 Behavior를 반환**한다. `active(newBalance)`처럼 새 값을 담은 Behavior를 반환함으로써 가변 변수(`var`) 없이 상태를 갱신한다. 이는 함수형 프로그래밍의 불변성 원칙을 Actor 모델에 적용한 것이다.

---

## Actor 모델의 고급 패턴

### Supervision Tree: 장애 격리

Erlang과 Akka 모두 **감독자 트리(Supervisor Tree)**를 지원한다. 부모 Actor는 자식 Actor의 실패를 감지하고 재시작 전략을 결정한다.

```
RootSupervisor
├── UserService (one-for-one: 실패한 자식만 재시작)
│   ├── UserActor-1
│   └── UserActor-2
└── PaymentService (all-for-one: 하나 실패 시 전체 재시작)
    ├── PaymentActor
    └── FraudDetectionActor
```

이 구조 덕분에 Erlang 시스템은 **"Let it crash"** 철학을 채택한다. 오류를 복잡한 방어 코드로 막는 대신, Actor가 실패하면 감독자가 깨끗한 상태로 재시작한다. 이는 Erlang이 99.9999999% (나인 나인) 가용성을 달성하는 핵심 메커니즘이다.

### Location Transparency: 분산 환경

Actor의 주소(ActorRef)는 로컬인지 원격인지를 추상화한다. 같은 코드가 단일 프로세스에서도, 다른 서버의 Actor에 메시지를 보낼 때도 동일하게 동작한다.

```erlang
%% 로컬 프로세스 ID와 동일한 방식으로 원격 프로세스에 메시지 전송
RemotePid = {account_server, 'node@remote-server.com'},
RemotePid ! {deposit, 1000, self()}.
```

이것이 Actor 모델이 분산 시스템 구축에 자연스럽게 어울리는 이유다.

---

## CSP vs Actor 모델

Go의 고루틴(goroutine)과 채널(channel)은 Actor와 유사해 보이지만, 실제로는 **CSP(Communicating Sequential Processes)** 모델을 구현한다.

| 특성 | Actor 모델 | CSP |
|------|-----------|-----|
| 통신 방식 | 비동기, 수신자에게 전달 | 동기/비동기, 채널을 통해 |
| 메일박스 | Actor마다 내장 | 채널이 별도 존재 |
| 주소 지정 | Actor를 직접 지칭 | 채널을 지칭 |
| 대표 언어 | Erlang, Akka | Go, Clojure core.async |

```go
// Go: CSP 스타일의 동시성 (채널이 first-class citizen)
func bankAccount(initial int) (chan<- int, chan<- int, <-chan int) {
    deposit := make(chan int)
    withdraw := make(chan int)
    balance := make(chan int)

    go func() {
        bal := initial
        for {
            select {
            case amount := <-deposit:
                bal += amount
            case amount := <-withdraw:
                if amount <= bal {
                    bal -= amount
                }
            case balance <- bal:
                // 현재 잔액 응답
            }
        }
    }()

    return deposit, withdraw, balance
}

// 사용
dep, with, bal := bankAccount(1000)
dep <- 500        // 입금
with <- 200       // 출금
fmt.Println(<-bal) // 1300
```

Go에서는 채널 자체가 통신의 단위다. Actor 모델에서는 Actor가 주체이고 메일박스가 부속이지만, CSP에서는 채널이 일급 시민(first-class citizen)이다.

---

## 주의사항과 실전 팁

### 1. 메시지는 불변(Immutable)이어야 한다

Actor 간에 가변(mutable) 객체를 메시지로 전달하면 공유 상태가 다시 생긴다. 항상 불변 데이터 구조나 값 복사본을 전달해야 한다.

```scala
// 나쁜 예: 가변 리스트를 메시지로 전달
case class ProcessList(data: mutable.ListBuffer[Int])  // 위험!

// 좋은 예: 불변 컬렉션
case class ProcessList(data: List[Int])  // 안전
```

### 2. 메일박스 크기를 모니터링하라

Actor가 처리하는 속도보다 메시지가 쌓이는 속도가 빠르면 메일박스가 무한히 증가해 OOM이 발생한다. Back-pressure 전략(메일박스 크기 제한, 느린 생산자 조절)이 필요하다.

### 3. "fire-and-forget" vs 요청-응답

비동기 메시지는 기본적으로 응답을 보장하지 않는다. 응답이 필요하다면 `ask` 패턴과 타임아웃을 함께 사용해야 한다. 응답 없는 메시지("fire-and-forget")는 결과 확인이 불가능하므로 중요한 작업에는 부적합하다.

### 4. Actor 입도(Granularity) 설계

Actor 하나가 너무 많은 책임을 지면 단일 병목이 된다. 반대로 너무 잘게 쪼개면 메시지 전달 오버헤드가 커진다. 일반적으로 **상태와 생명 주기를 공유하는 논리적 단위**를 Actor로 설계하는 것이 적절하다.

---

## 참고 자료

- [Actor Model — Ergo Framework Documentation](https://docs.ergo.services/basics/actor-model)
- [CSP vs Actor model for concurrency — DEV Community](https://dev.to/karanpratapsingh/csp-vs-actor-model-for-concurrency-1cpg)
- [Akka Official Documentation](https://akka.io/docs/)
- [Erlang/OTP — Supervisor Behaviour](https://www.erlang.org/doc/design_principles/sup_princ)
