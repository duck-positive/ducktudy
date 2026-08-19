---
layout: post
title: "반응형 스트림과 백프레셔(Reactive Streams & Backpressure) 완전 정복: 빠른 생산자가 느린 소비자를 압도하지 않게 하는 비동기 흐름 제어"
date: 2026-08-19
categories: [cs, computer-science]
tags: [reactive-streams, backpressure, project-reactor, rxjava, async, non-blocking, java]
---

## 반응형 스트림이란?

전통적인 동기 프로그래밍에서는 메서드가 반환할 때까지 호출자가 블로킹된다. 비동기 프로그래밍으로 넘어가면 콜백, Future/Promise 등을 통해 블로킹을 피하지만, 새로운 문제가 등장한다. **생산자(Producer)가 소비자(Consumer)보다 훨씬 빠르게 데이터를 만들어내는 경우다.**

예를 들어 초당 100만 건의 주식 호가 이벤트를 생성하는 시스템에서, 소비자가 초당 10만 건밖에 처리하지 못한다면 나머지 90만 건은 어디로 가야 할까? 버퍼에 무한정 쌓으면 OutOfMemoryError가 발생하고, 버리면 데이터 유실이 생긴다.

**반응형 스트림(Reactive Streams)**은 이 문제를 해결하기 위한 표준 사양(specification)으로, 2015년 Netflix, Pivotal, Red Hat, Twitter 등이 공동 제정했다. 핵심 목표는 **논블로킹 백프레셔(non-blocking backpressure)**로 비동기 스트림 처리를 표준화하는 것이다.

이 사양은 Java 9의 `java.util.concurrent.Flow` API로 채택되었고, Project Reactor(Spring WebFlux의 핵심), RxJava 2/3, Akka Streams 등 주요 라이브러리의 토대가 되었다.

---

## 핵심 인터페이스: 4개의 계약

반응형 스트림 사양은 네 가지 인터페이스로 정의된다.

```java
// 데이터를 발행하는 생산자
public interface Publisher<T> {
    void subscribe(Subscriber<? super T> subscriber);
}

// 데이터를 소비하는 구독자
public interface Subscriber<T> {
    void onSubscribe(Subscription s);  // 구독 시작 시 호출
    void onNext(T t);                  // 데이터 항목 수신
    void onError(Throwable t);         // 오류 발생 시 호출
    void onComplete();                 // 스트림 완료 시 호출
}

// 생산자-소비자 간 흐름 제어
public interface Subscription {
    void request(long n);  // n개의 아이템을 요청 (백프레셔의 핵심!)
    void cancel();         // 구독 취소
}

// Publisher이자 Subscriber인 중간 처리자
public interface Processor<T, R> extends Subscriber<T>, Publisher<R> {}
```

### 백프레셔의 작동 원리

백프레셔는 `Subscription.request(n)` 메서드를 통해 구현된다. 소비자가 처리할 수 있는 만큼만 요청하고, 생산자는 그 수량만큼만 발행한다. 이 **pull 기반 프로토콜**이 흐름 제어의 핵심이다.

```
[Publisher] <───────── request(10) ─────── [Subscriber]
[Publisher] ──── onNext(item1..10) ──────> [Subscriber]
[Publisher] <───────── request(5) ──────── [Subscriber]
[Publisher] ──── onNext(item11..15) ─────> [Subscriber]
```

소비자가 준비될 때까지 생산자는 추가 데이터를 보내지 않는다. 이를 **demand-driven(수요 기반) 흐름 제어**라고 한다.

---

## Project Reactor 심화: Flux와 Mono

**Project Reactor**는 반응형 스트림 사양의 가장 완성도 높은 구현체로, Spring WebFlux의 기반이다.

- **Mono<T>**: 0개 또는 1개의 아이템을 발행
- **Flux<T>**: 0개 이상의 아이템을 발행 (무한 스트림 포함)

### 직접 구현: BaseSubscriber로 백프레셔 제어

```java
import reactor.core.publisher.BaseSubscriber;
import reactor.core.publisher.Flux;

public class BackpressureDemo {
    public static void main(String[] args) throws InterruptedException {
        Flux<Integer> fastProducer = Flux.range(1, 100)
            .doOnNext(n -> System.out.println("생산: " + n));
        
        fastProducer.subscribe(new BaseSubscriber<Integer>() {
            private int processedCount = 0;
            
            @Override
            protected void hookOnSubscribe(org.reactivestreams.Subscription subscription) {
                System.out.println("구독 시작 - 초기 5개 요청");
                request(5); // 처음에 5개만 요청
            }
            
            @Override
            protected void hookOnNext(Integer value) {
                System.out.println("소비: " + value);
                processedCount++;
                
                // 5개 처리 후 다음 5개 요청
                if (processedCount % 5 == 0) {
                    System.out.println("--- 다음 배치 요청 ---");
                    request(5);
                }
                
                // 25개 처리 후 취소
                if (processedCount >= 25) {
                    System.out.println("25개 처리 완료, 구독 취소");
                    cancel();
                }
            }
            
            @Override
            protected void hookOnError(Throwable throwable) {
                System.err.println("오류: " + throwable.getMessage());
            }
            
            @Override
            protected void hookOnComplete() {
                System.out.println("스트림 완료");
            }
        });
        
        Thread.sleep(1000);
    }
}
```

---

## 백프레셔 처리 전략 4가지

소비자가 따라가지 못할 때 어떻게 처리할지 결정하는 것이 **백프레셔 전략**이다.

```java
import reactor.core.publisher.Flux;
import reactor.core.publisher.BufferOverflowStrategy;
import java.time.Duration;

public class BackpressureStrategies {
    
    // 생산자: 10ms마다 이벤트 발생 (빠름)
    static Flux<Long> fastSource() {
        return Flux.interval(Duration.ofMillis(10));
    }
    
    // 소비자: 100ms마다 처리 (느림 - 10배 느린 소비자)
    static void slowConsumer(long item) throws InterruptedException {
        Thread.sleep(100);
        System.out.println("처리: " + item);
    }
    
    public static void main(String[] args) throws InterruptedException {
        
        // 전략 1: BUFFER - 처리되지 않은 아이템을 버퍼에 저장 (기본값)
        // 주의: 메모리 무제한 사용 가능
        System.out.println("=== 전략 1: BUFFER ===");
        fastSource()
            .onBackpressureBuffer(10,  // 최대 버퍼 크기
                dropped -> System.out.println("버퍼 초과, 드롭: " + dropped),
                BufferOverflowStrategy.DROP_OLDEST) // 오래된 것부터 제거
            .take(5)
            .subscribe(item -> {
                try { slowConsumer(item); }
                catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            });
        
        Thread.sleep(2000);
        
        // 전략 2: DROP - 처리 능력 초과 아이템을 버림
        System.out.println("\n=== 전략 2: DROP ===");
        fastSource()
            .onBackpressureDrop(dropped -> 
                System.out.println("드롭: " + dropped))
            .take(5)
            .subscribe(item -> {
                try { slowConsumer(item); }
                catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            });
        
        Thread.sleep(2000);
        
        // 전략 3: LATEST - 가장 최신 아이템만 유지
        System.out.println("\n=== 전략 3: LATEST ===");
        fastSource()
            .onBackpressureLatest()
            .take(5)
            .subscribe(item -> {
                try { slowConsumer(item); }
                catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            });
        
        Thread.sleep(2000);
        
        // 전략 4: ERROR - 초과 시 MissingBackpressureException 발생
        System.out.println("\n=== 전략 4: ERROR ===");
        fastSource()
            .onBackpressureError()
            .take(5)
            .subscribe(
                item -> { try { slowConsumer(item); } 
                          catch (InterruptedException e) { Thread.currentThread().interrupt(); } },
                error -> System.out.println("오류 발생: " + error.getClass().getSimpleName())
            );
        
        Thread.sleep(2000);
    }
}
```

---

## 실전 패턴: 파이프라인 설계

실제 서비스에서 백프레셔를 활용하는 전형적인 패턴을 살펴보자.

```java
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;
import reactor.core.scheduler.Schedulers;
import java.time.Duration;
import java.util.List;

public class ReactivePipeline {
    
    // 시뮬레이션: 데이터베이스에서 대량 데이터 조회
    static Flux<String> readFromDatabase() {
        return Flux.range(1, 10_000)
            .map(i -> "record-" + i)
            .delayElements(Duration.ofMillis(1)); // DB 지연 시뮬레이션
    }
    
    // 시뮬레이션: 외부 API 호출
    static Mono<String> callExternalApi(String record) {
        return Mono.just("processed-" + record)
            .delayElement(Duration.ofMillis(50)); // API 지연 시뮬레이션
    }
    
    // 시뮬레이션: 결과 저장
    static Mono<Void> saveToStorage(List<String> batch) {
        return Mono.fromRunnable(() -> 
            System.out.println("배치 저장: " + batch.size() + "개"));
    }
    
    public static void main(String[] args) throws InterruptedException {
        readFromDatabase()
            // 1단계: 16개씩 병렬 처리 (flatMap 동시성 제어)
            .flatMap(
                record -> callExternalApi(record)
                    .subscribeOn(Schedulers.boundedElastic()),
                16, // 최대 동시 구독 수 (백프레셔 제어)
                256 // prefetch 크기
            )
            // 2단계: 100개씩 배치로 묶기
            .buffer(100)
            // 3단계: 배치 저장 (하나씩 순차 처리)
            .concatMap(batch -> saveToStorage(batch))
            // 4단계: 완료 대기
            .blockLast();
        
        System.out.println("전체 파이프라인 완료");
    }
}
```

### 핵심 연산자의 의미

```
readFromDatabase()  →  flatMap(16, 256)  →  buffer(100)  →  concatMap(saveToStorage)
   [소스]                [동시성 16개]       [100개 묶음]      [순차 저장]
   빠름                   병렬/백프레셔       집계              블로킹 방지
```

- **flatMap(concurrency, prefetch)**: 내부 Publisher를 최대 16개 동시 구독. prefetch=256은 각 내부 Publisher에서 미리 가져올 아이템 수.
- **concatMap**: 순서를 보장하는 flatMap으로, 이전 내부 Publisher가 완료될 때까지 다음 것을 구독하지 않음.

---

## 내부 동작: Publisher-Subscriber 계약 규칙

반응형 스트림 사양은 총 37개의 규칙(Rule)을 정의한다. 핵심 규칙들:

### Publisher 규칙
- **Rule 1**: `request(n)`으로 요청한 수보다 많이 `onNext`를 호출해서는 안 된다.
- **Rule 7**: 오류 발생 시 반드시 `onError`를 호출해야 한다.

### Subscriber 규칙
- **Rule 1**: `onSubscribe`, `onNext`, `onError`, `onComplete`는 순차적으로 호출되어야 한다.
- **Rule 9**: `onSubscribe` 호출 후에만 `request`를 호출할 수 있다.

### Subscription 규칙
- **Rule 1**: `request(n)`에서 n ≤ 0이면 `IllegalArgumentException`이 전파되어야 한다.
- **Rule 6**: `cancel()` 후에는 더 이상 신호를 보내서는 안 된다.

```java
// 직접 Publisher 구현 예시 (교육용)
import org.reactivestreams.*;
import java.util.concurrent.atomic.AtomicLong;

public class RangePublisher implements Publisher<Integer> {
    private final int start, count;
    
    public RangePublisher(int start, int count) {
        this.start = start;
        this.count = count;
    }
    
    @Override
    public void subscribe(Subscriber<? super Integer> subscriber) {
        subscriber.onSubscribe(new Subscription() {
            private int current = start;
            private final AtomicLong requested = new AtomicLong(0);
            private volatile boolean cancelled = false;
            
            @Override
            public void request(long n) {
                if (n <= 0) {
                    subscriber.onError(new IllegalArgumentException("n must be positive"));
                    return;
                }
                
                long prev = requested.getAndAdd(n);
                if (prev > 0) return; // 이미 루프 실행 중
                
                // demand-driven 루프
                while (!cancelled && current < start + count && requested.get() > 0) {
                    subscriber.onNext(current++);
                    requested.decrementAndGet();
                }
                
                if (!cancelled && current >= start + count) {
                    subscriber.onComplete();
                }
            }
            
            @Override
            public void cancel() {
                cancelled = true;
            }
        });
    }
}
```

---

## 백프레셔 없는 시스템과의 비교

| 측면 | 전통적 비동기 (콜백/Promise) | 반응형 스트림 |
|------|---------------------------|-------------|
| 흐름 제어 | 없음 (push 기반) | 있음 (pull 기반, demand-driven) |
| 메모리 안전성 | 버퍼 폭발 위험 | 소비자 속도에 맞춤 |
| 오류 처리 | 콜백 지옥, 분산된 try-catch | onError 시그널로 통합 |
| 조합 가능성 | 낮음 | 풍부한 연산자 조합 |
| 표준화 | 없음 | Reactive Streams 표준 |
| 구현체 | 다양, 호환 불가 | Reactor, RxJava, Akka (상호 운용 가능) |

---

## 주의사항과 팁

### 1. 블로킹 작업과의 혼용 금지

반응형 파이프라인 내에서 블로킹 코드를 직접 호출하면 전체 스레드가 블로킹된다. 반드시 `Schedulers.boundedElastic()`으로 격리해야 한다.

```java
// 잘못된 예
Flux.range(1, 100)
    .map(i -> {
        Thread.sleep(100); // 블로킹! 이벤트 루프 스레드를 잡아먹음
        return i * 2;
    });

// 올바른 예
Flux.range(1, 100)
    .flatMap(i -> Mono.fromCallable(() -> {
        Thread.sleep(100);
        return i * 2;
    }).subscribeOn(Schedulers.boundedElastic()));
```

### 2. Context 전파

Spring Security, MDC(Mapped Diagnostic Context) 등은 ThreadLocal에 의존한다. 반응형 파이프라인에서는 스레드가 바뀌므로 `Context` API를 사용해야 한다.

```java
Flux.just("data")
    .contextWrite(Context.of("userId", "user-123"))
    .flatMap(data -> 
        Mono.deferContextual(ctx -> {
            String userId = ctx.get("userId");
            return processWithUser(data, userId);
        })
    );
```

### 3. Hot vs Cold Publisher

- **Cold Publisher**: 각 구독자마다 처음부터 데이터를 발행. HTTP 요청 같은 스트림.
- **Hot Publisher**: 구독 시점부터의 데이터만 수신. 주식 시세, 실시간 이벤트.

```java
// Cold: 각 구독자가 독립적인 시퀀스를 받음
Flux<Integer> cold = Flux.range(1, 5);

// Hot: 구독 시점 이후 이벤트만 수신
Flux<Long> hot = Flux.interval(Duration.ofSeconds(1)).share();
```

### 4. 무한 스트림 처리

백프레셔는 무한 스트림을 처리할 때 특히 중요하다. `take()`, `limitRate()`, `takeUntil()` 등으로 소비량을 명시적으로 제한하라.

```java
Flux.interval(Duration.ofMillis(10))
    .limitRate(100)    // 한 번에 최대 100개씩 요청
    .take(1000)        // 총 1000개 후 완료
    .subscribe(System.out::println);
```

반응형 스트림과 백프레셔는 현대 마이크로서비스, 실시간 데이터 파이프라인, 고성능 API 서버에서 필수적인 기술이다. 흐름 제어 없이 단순히 비동기 코드를 작성하면 시스템은 높은 부하에서 조용히 무너진다. 백프레셔는 시스템이 **자신이 처리할 수 있는 만큼만 받는** 것을 보장하는 안전 장치다.

## 참고 자료
- [Reactive Streams Specification](https://www.reactive-streams.org/)
- [Project Reactor: Introduction to Reactive Programming](https://projectreactor.io/docs/core/release/reference/reactiveProgramming.html)
- [Backpressure Mechanism in Spring WebFlux - Baeldung](https://www.baeldung.com/spring-webflux-backpressure)
- [Backpressure in Reactive Systems - Nicolas Fränkel](https://blog.frankel.ch/backpressure-reactive-systems/)
