---
layout: post
title: "구조적 동시성(Structured Concurrency) 완전 정복: 코루틴·가상 스레드·고루틴의 수명을 부모가 관리하는 법"
date: 2026-10-06
categories: [cs, computer-science]
tags: [structured-concurrency, coroutines, kotlin, java, goroutine, concurrency, loom, virtual-threads]
---

## 구조적 동시성이란 무엇인가

소프트웨어 개발에서 "동시성(concurrency)"은 수십 년간 골칫덩어리였다. 스레드를 생성해 작업을 분산하면 성능은 오르지만, 스레드가 *언제*, *어디서*, *어떻게* 끝나는지를 추적하는 일은 극도로 어렵다. 함수 하나가 백그라운드 스레드를 띄우고 반환하면, 그 스레드의 수명은 함수의 호출 스택과 완전히 분리된다. 에러가 발생해도 호출자는 알 수 없고, 프로그램이 종료될 때까지 좀비 스레드가 자원을 잡아먹는다.

**구조적 동시성(Structured Concurrency)**은 이 문제를 해결하기 위한 패러다임이다. 핵심 아이디어는 단순하다: **동시 작업의 수명은 반드시 그 작업을 생성한 스코프(scope)보다 길 수 없다.** 마치 구조적 프로그래밍이 `goto`를 없애고 `if/for/while`로 제어 흐름을 명확히 한 것처럼, 구조적 동시성은 병렬 실행 흐름을 스코프로 묶어 수명을 제어한다.

이 아이디어는 2016년 Nathaniel J. Smith가 Python의 `trio` 라이브러리를 만들면서 체계화했고, 이후 Kotlin Coroutines의 `CoroutineScope`, Java 21의 `StructuredTaskScope`, Go의 `errgroup` 패키지 등에 영향을 주었다.

---

## 왜 구조적 동시성이 필요한가

### 기존 비구조적 동시성의 문제

전통적인 스레드 기반 코드의 문제를 예시로 살펴보자.

```java
// 비구조적 동시성의 문제 예시 (Java)
ExecutorService executor = Executors.newCachedThreadPool();

Future<String> userFuture = executor.submit(() -> fetchUser(userId));
Future<String> orderFuture = executor.submit(() -> fetchOrder(orderId));

String user = userFuture.get();   // 블로킹
String order = orderFuture.get(); // 블로킹
```

위 코드의 문제점은 다음과 같다:

1. **수명 불일치**: `executor`의 스레드들은 이 메서드가 반환한 후에도 살아있을 수 있다.
2. **에러 전파 누락**: `fetchUser`가 예외를 던지면 `fetchOrder`는 계속 실행된다. 자원 낭비다.
3. **취소 불가**: `userFuture`를 취소해도 `orderFuture`는 취소되지 않는다.
4. **데드라인 적용 어려움**: 두 작업 전체에 타임아웃을 걸기가 복잡하다.

구조적 동시성은 **"자식 작업은 부모 스코프가 닫히기 전에 반드시 완료되어야 한다"**는 불변식을 언어나 라이브러리 차원에서 강제함으로써 이 문제들을 해결한다.

### 제어 흐름의 명확성

구조적 프로그래밍이 소스 코드의 위에서 아래로 읽으면 실행 순서를 알 수 있게 해주듯, 구조적 동시성은 **블록 안에서 생성된 모든 병렬 흐름이 블록을 벗어나기 전에 끝난다**는 보장을 준다. 이는 코드 가독성, 디버깅, 테스트를 획기적으로 개선한다.

---

## 실제 구현 예제

### 예제 1: Kotlin Coroutines — coroutineScope와 async

Kotlin은 언어 차원에서 구조적 동시성을 지원한다. `coroutineScope { }` 블록은 내부에서 생성된 모든 코루틴이 완료될 때까지 suspend되며, 하나라도 실패하면 나머지를 자동 취소한다.

```kotlin
import kotlinx.coroutines.*

data class UserInfo(val id: String, val name: String)
data class OrderInfo(val orderId: String, val total: Double)
data class Dashboard(val user: UserInfo, val order: OrderInfo)

// suspend 함수 — 네트워크 I/O 시뮬레이션
suspend fun fetchUser(userId: String): UserInfo {
    delay(200)  // 200ms 지연
    return UserInfo(userId, "홍길동")
}

suspend fun fetchOrder(orderId: String): OrderInfo {
    delay(300)  // 300ms 지연
    return OrderInfo(orderId, 59_900.0)
}

// 두 요청을 병렬로 실행하되, 하나 실패 시 나머지도 취소
suspend fun buildDashboard(userId: String, orderId: String): Dashboard =
    coroutineScope {
        val userDeferred = async { fetchUser(userId) }
        val orderDeferred = async { fetchOrder(orderId) }

        // 두 결과를 기다림 — 총 소요 시간 ≈ max(200, 300) = 300ms
        Dashboard(userDeferred.await(), orderDeferred.await())
    }
// coroutineScope 블록을 벗어나는 순간 userDeferred, orderDeferred는 반드시 완료(또는 취소) 상태

fun main() = runBlocking {
    val start = System.currentTimeMillis()
    val dashboard = buildDashboard("user-1", "order-42")
    val elapsed = System.currentTimeMillis() - start
    println("Dashboard: $dashboard (${elapsed}ms)") // ~300ms
}
```

`coroutineScope`의 동작 원리:
- `async { }` 블록은 코루틴을 시작하고 `Deferred<T>`를 반환한다.
- 두 `Deferred`는 부모 `coroutineScope`에 자식으로 등록된다.
- `userDeferred`가 예외를 던지면, 부모 스코프가 `orderDeferred`를 `cancel()`한다.
- `coroutineScope { }` 표현식은 모든 자식이 끝날 때까지 suspend된다.

이처럼 **스코프의 생명주기 = 자식 코루틴들의 생명주기**라는 불변식이 컴파일러 수준에서 보장된다.

### 예제 2: Java 21 StructuredTaskScope

Java 21에서 정식 도입된 `StructuredTaskScope`는 JVM 위에서 구조적 동시성을 제공한다. 가상 스레드(Virtual Thread, Project Loom)와 결합해 수백만 개의 동시 작업을 OS 스레드 생성 없이 처리한다.

```java
import java.util.concurrent.*;
import java.util.concurrent.StructuredTaskScope.*;

record UserInfo(String id, String name) {}
record OrderInfo(String orderId, double total) {}
record Dashboard(UserInfo user, OrderInfo order) {}

public class DashboardService {

    static UserInfo fetchUser(String userId) throws InterruptedException {
        Thread.sleep(200);
        return new UserInfo(userId, "홍길동");
    }

    static OrderInfo fetchOrder(String orderId) throws InterruptedException {
        Thread.sleep(300);
        return new OrderInfo(orderId, 59_900.0);
    }

    // ShutdownOnFailure: 하나라도 실패하면 나머지 작업을 즉시 취소
    static Dashboard buildDashboard(String userId, String orderId)
            throws InterruptedException, ExecutionException {

        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
            Subtask<UserInfo> userTask = scope.fork(() -> fetchUser(userId));
            Subtask<OrderInfo> orderTask = scope.fork(() -> fetchOrder(orderId));

            scope.join();           // 모든 subtask 완료 대기
            scope.throwIfFailed();  // 실패한 subtask가 있으면 예외 전파

            return new Dashboard(userTask.get(), orderTask.get());
        }
        // try-with-resources: 블록 탈출 시 scope가 close() → 모든 미완료 task 취소
    }

    public static void main(String[] args) throws Exception {
        long start = System.currentTimeMillis();
        Dashboard dashboard = buildDashboard("user-1", "order-42");
        long elapsed = System.currentTimeMillis() - start;
        System.out.println("Dashboard: " + dashboard + " (" + elapsed + "ms)");
    }
}
```

`ShutdownOnFailure` 외에도 `ShutdownOnSuccess`(하나라도 성공하면 나머지 취소, 레이스 패턴)가 내장되어 있으며, `StructuredTaskScope`를 직접 상속해 커스텀 정책을 구현할 수도 있다.

### 예제 3: Go의 errgroup — 구조적 동시성 근사 구현

Go는 언어 수준에서 구조적 동시성을 제공하지 않지만, `golang.org/x/sync/errgroup` 패키지로 유사한 패턴을 구현할 수 있다.

```go
package main

import (
    "context"
    "fmt"
    "golang.org/x/sync/errgroup"
    "time"
)

type UserInfo struct{ ID, Name string }
type OrderInfo struct{ OrderID string; Total float64 }

func fetchUser(ctx context.Context, uid string) (UserInfo, error) {
    select {
    case <-ctx.Done():
        return UserInfo{}, ctx.Err()
    case <-time.After(200 * time.Millisecond):
        return UserInfo{uid, "홍길동"}, nil
    }
}

func fetchOrder(ctx context.Context, oid string) (OrderInfo, error) {
    select {
    case <-ctx.Done():
        return OrderInfo{}, ctx.Err()
    case <-time.After(300 * time.Millisecond):
        return OrderInfo{oid, 59900.0}, nil
    }
}

func buildDashboard(ctx context.Context, uid, oid string) error {
    g, ctx := errgroup.WithContext(ctx)

    var user UserInfo
    var order OrderInfo

    g.Go(func() error {
        var err error
        user, err = fetchUser(ctx, uid)
        return err
    })
    g.Go(func() error {
        var err error
        order, err = fetchOrder(ctx, oid)
        return err
    })

    // g.Wait(): 모든 고루틴 완료 대기, 하나 실패 시 context 취소
    if err := g.Wait(); err != nil {
        return fmt.Errorf("dashboard build failed: %w", err)
    }
    fmt.Printf("Dashboard: user=%v order=%v\n", user, order)
    return nil
}
```

`errgroup.WithContext`는 첫 번째 에러 발생 시 `context`를 취소해 다른 고루틴들이 조기 종료할 수 있게 한다. 단, Go의 경우 스코프를 벗어나도 고루틴이 여전히 살아있을 수 있어 진정한 구조적 동시성과는 차이가 있다. `g.Wait()` 호출을 잊으면 고루틴이 누수된다.

---

## 구조적 동시성의 핵심 보장과 이점

### 1. 에러 전파의 명확성

자식 작업 중 하나가 실패하면:
- **Kotlin**: 부모 `CoroutineScope`가 나머지 자식을 모두 취소하고 예외를 전파한다.
- **Java**: `scope.throwIfFailed()`가 첫 번째 실패 예외를 래핑해서 던진다.
- 호출자는 항상 명확한 실패 이유를 받는다.

### 2. 자원 누수 방지

스코프가 닫힐 때 내부의 모든 자식 작업은 완료 또는 취소 상태임이 보장된다. 좀비 스레드나 고루틴이 남지 않는다.

### 3. 취소의 전파

부모 스코프가 취소되면 자동으로 모든 자식이 취소된다. Kotlin에서는 `Job.cancel()`, Java에서는 `scope.shutdown()`이 이 역할을 한다.

### 4. 관찰 가능성(Observability)

구조적 동시성에서는 "이 요청을 처리하는 동안 생성된 모든 작업"을 트리 구조로 추적할 수 있다. 분산 트레이싱이나 로깅에서 컨텍스트를 자연스럽게 전파할 수 있다.

---

## 주의사항 및 팁

### 스코프와 디스패처 분리 (Kotlin)

```kotlin
// 나쁜 예: GlobalScope 사용 — 구조적 동시성 파괴
GlobalScope.launch { /* 부모와 수명이 연결되지 않음 */ }

// 좋은 예: 주입받은 스코프 사용
class UserRepository(private val scope: CoroutineScope) {
    fun refreshAsync() = scope.async { fetchUsers() }
}
```

`GlobalScope`는 구조적 동시성을 파괴한다. 가능하면 `coroutineScope { }` 또는 주입받은 스코프를 사용하라.

### SupervisorScope — 형제 코루틴 독립성

일반 `coroutineScope`는 자식 중 하나가 실패하면 형제도 모두 취소한다. 형제들이 서로 독립적이어야 한다면 `supervisorScope`를 사용한다.

```kotlin
supervisorScope {
    val a = async { task1() }  // task1 실패해도
    val b = async { task2() }  // task2는 계속 실행됨
}
```

### Java StructuredTaskScope의 가상 스레드 이점

`StructuredTaskScope.fork()`는 내부적으로 가상 스레드를 사용한다. 가상 스레드는 OS 스레드 1개가 수천 개의 가상 스레드를 다중화하므로, 블로킹 I/O를 사용하더라도 메모리 오버헤드가 매우 낮다. `Thread.sleep()`나 JDBC 같은 블로킹 API를 그대로 써도 된다.

### 타임아웃 적용

```kotlin
// Kotlin: withTimeout으로 전체 스코프에 타임아웃
withTimeout(1000L) {  // 1초 안에 완료되지 않으면 TimeoutCancellationException
    coroutineScope {
        async { fetchUser(userId) }
        async { fetchOrder(orderId) }
    }
}
```

```java
// Java: join(Duration)으로 타임아웃
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    scope.fork(() -> fetchUser(userId));
    scope.fork(() -> fetchOrder(orderId));
    scope.joinUntil(Instant.now().plusSeconds(1));  // 1초 타임아웃
    scope.throwIfFailed();
}
```

---

## 마무리

구조적 동시성은 동시 프로그래밍의 복잡성을 근본적으로 줄여주는 패러다임이다. "자식의 수명은 부모 스코프를 벗어날 수 없다"는 단순한 규칙 하나가 에러 전파, 자원 누수, 취소 전파라는 오랜 난제를 한꺼번에 해결한다. Kotlin Coroutines는 이미 실전에서 광범위하게 쓰이고 있으며, Java 21의 `StructuredTaskScope`는 JVM 생태계 전반으로 이 패턴을 확산시키고 있다. 새 동시성 코드를 작성한다면 반드시 구조적 동시성 원칙을 적용하라.

## 참고 자료
- [Java StructuredTaskScope 공식 문서 (JEP 453)](https://openjdk.org/jeps/453)
- [Kotlin Coroutines 구조적 동시성 가이드](https://kotlinlang.org/docs/coroutines-basics.html#structured-concurrency)
- [Structured Concurrency: Java vs Kotlin 비교 연구](https://www2.tvz.hr/?p=43140)
- [golang.org/x/sync/errgroup](https://pkg.go.dev/golang.org/x/sync/errgroup)
