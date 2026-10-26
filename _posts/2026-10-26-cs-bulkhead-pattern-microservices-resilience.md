---
layout: post
title: "Bulkhead 패턴: 마이크로서비스 격벽으로 장애 전파 차단하기"
date: 2026-10-26
categories: [cs, computer-science]
tags: [bulkhead, microservices, resilience, fault-tolerance, resilience4j, thread-pool, semaphore]
---

## Bulkhead(격벽) 패턴이란

Bulkhead는 선박 공학에서 유래한 개념입니다. 대형 선박은 내부를 여러 개의 수밀 격벽(Watertight Compartment)으로 나눕니다. 한 구획에 구멍이 뚫려 물이 차더라도 격벽이 막아 다른 구획으로 물이 넘치지 않습니다. 타이타닉호의 격벽이 충분하지 않았던 것이 침몰의 원인 중 하나였습니다.

소프트웨어 아키텍처에서 Bulkhead 패턴은 동일한 원리를 적용합니다. **서비스의 리소스(스레드, 커넥션 등)를 기능 또는 의존성 단위로 분리**하여, 한 부분의 장애가 전체 시스템으로 전파되지 않도록 격리합니다. Michael Nygard의 저서 *Release It!*(2007)에서 소프트웨어 패턴으로 처음 제시되었습니다.

## 왜 격리가 필요한가: 공유 스레드 풀의 위험

일반적인 단일 스레드 풀 구조에서는 하나의 의존성 장애가 전체 서비스를 마비시킬 수 있습니다.

### 장애 시나리오

```
[단일 스레드 풀 - 100개 스레드]

정상 시:
  - 결제 서비스 호출: 30개 스레드 사용
  - 재고 서비스 호출: 30개 스레드 사용
  - 알림 서비스 호출: 10개 스레드 사용
  - 여유 스레드: 30개

결제 서비스가 응답 지연(30초 timeout):
  - 결제 서비스 호출: 30 → 100개 스레드 점유 (요청 대기 큐에 쌓임)
  - 재고 서비스 호출: 스레드 없음 → 타임아웃
  - 알림 서비스 호출: 스레드 없음 → 타임아웃
  - 결과: 결제 서비스 1개의 지연으로 전체 서비스 장애!
```

하나의 느린 의존성이 스레드 풀을 점령해 다른 멀쩡한 기능까지 모두 불능 상태로 만드는 **캐스케이딩 장애(Cascading Failure)** 입니다. Bulkhead 패턴은 이 문제를 해결합니다.

## Bulkhead의 두 가지 구현 방식

### 1. 스레드 풀 격리 (Thread Pool Isolation)

각 의존성 또는 기능 그룹에 **독립적인 스레드 풀**을 할당합니다.

```
[Bulkhead 적용 후]

결제 서비스 스레드 풀:  30개 스레드
재고 서비스 스레드 풀:  20개 스레드
알림 서비스 스레드 풀:  10개 스레드
공통 작업 스레드 풀:    40개 스레드

결제 서비스 장애 시:
  - 결제 스레드 풀: 30개 전부 점유 → 결제 서비스만 차단
  - 재고 서비스: 독립 스레드 풀 사용 → 정상 동작
  - 알림 서비스: 독립 스레드 풀 사용 → 정상 동작
  - 결과: 장애 격리 성공!
```

장점: 완전한 격리, 각 풀별 독립적인 모니터링  
단점: 스레드 컨텍스트 스위칭 오버헤드, 스레드 수 증가

### 2. 세마포어 격리 (Semaphore Isolation)

공유 스레드를 유지하되, 특정 의존성에 **동시 접근 가능한 수를 제한**합니다.

장점: 스레드 오버헤드 없음, 구현 간단  
단점: 타임아웃 설정이 어렵고, 스레드 풀만큼 강한 격리 불가

## 실제 구현 예제

### 예제 1: Resilience4j Bulkhead (Java/Kotlin)

Resilience4j는 Java 생태계에서 Hystrix를 대체하는 표준 라이브러리입니다.

```java
import io.github.resilience4j.bulkhead.Bulkhead;
import io.github.resilience4j.bulkhead.BulkheadConfig;
import io.github.resilience4j.bulkhead.BulkheadFullException;
import io.github.resilience4j.bulkhead.ThreadPoolBulkhead;
import io.github.resilience4j.bulkhead.ThreadPoolBulkheadConfig;

import java.time.Duration;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.CompletionStage;
import java.util.function.Supplier;

public class PaymentServiceClient {

    // 세마포어 Bulkhead: 동시 호출 수 제한
    private final Bulkhead semaphoreBulkhead = Bulkhead.of(
        "payment-service",
        BulkheadConfig.custom()
            .maxConcurrentCalls(20)        // 최대 동시 호출 20개
            .maxWaitDuration(Duration.ofMillis(100))  // 대기 최대 100ms
            .build()
    );

    // 스레드 풀 Bulkhead: 전용 스레드 풀 격리
    private final ThreadPoolBulkhead threadPoolBulkhead = ThreadPoolBulkhead.of(
        "payment-service-async",
        ThreadPoolBulkheadConfig.custom()
            .maxThreadPoolSize(10)         // 최대 스레드 10개
            .coreThreadPoolSize(5)         // 코어 스레드 5개
            .queueCapacity(20)             // 대기 큐 20개
            .keepAliveDuration(Duration.ofMillis(20))
            .build()
    );

    /**
     * 세마포어 Bulkhead를 이용한 동기 호출
     */
    public PaymentResult processPayment(PaymentRequest request) {
        // Bulkhead로 호출 래핑
        Supplier<PaymentResult> decoratedSupplier = Bulkhead.decorateSupplier(
            semaphoreBulkhead,
            () -> callPaymentAPI(request)
        );

        try {
            return decoratedSupplier.get();
        } catch (BulkheadFullException e) {
            // Bulkhead 포화 시: fallback 처리
            log.warn("Payment bulkhead full, falling back. Active calls: {}",
                semaphoreBulkhead.getMetrics().getAvailableConcurrentCalls());
            return PaymentResult.deferred(request.getOrderId());
        }
    }

    /**
     * 스레드 풀 Bulkhead를 이용한 비동기 호출
     */
    public CompletableFuture<InventoryResult> checkInventoryAsync(String itemId) {
        // 비동기 실행 - 결제 서비스와 완전히 다른 스레드 풀 사용
        CompletionStage<InventoryResult> stage = threadPoolBulkhead.executeSupplier(
            () -> callInventoryService(itemId)
        );

        return stage.toCompletableFuture()
            .exceptionally(ex -> {
                if (ex.getCause() instanceof BulkheadFullException) {
                    log.warn("Inventory bulkhead full for item: {}", itemId);
                    return InventoryResult.unknown(itemId);
                }
                throw new RuntimeException(ex);
            });
    }

    // Bulkhead 메트릭 모니터링
    public BulkheadMetrics getMetrics() {
        return BulkheadMetrics.builder()
            .availableCalls(semaphoreBulkhead.getMetrics().getAvailableConcurrentCalls())
            .maxCalls(semaphoreBulkhead.getBulkheadConfig().getMaxConcurrentCalls())
            .queueDepth(threadPoolBulkhead.getMetrics().getQueueDepth())
            .build();
    }

    private PaymentResult callPaymentAPI(PaymentRequest req) {
        // 외부 결제 API 호출 로직
        return PaymentResult.success(req.getOrderId());
    }

    private InventoryResult callInventoryService(String itemId) {
        // 재고 서비스 호출 로직
        return InventoryResult.available(itemId, 100);
    }
}
```

### 예제 2: Spring Boot + Resilience4j 설정 파일 방식

```yaml
# application.yml
resilience4j:
  bulkhead:
    instances:
      payment-service:
        maxConcurrentCalls: 20
        maxWaitDuration: 100ms
      
      notification-service:
        maxConcurrentCalls: 5
        maxWaitDuration: 50ms
  
  thread-pool-bulkhead:
    instances:
      inventory-service:
        maxThreadPoolSize: 10
        coreThreadPoolSize: 5
        queueCapacity: 20
        keepAliveDuration: 20ms
      
      analytics-service:
        maxThreadPoolSize: 5
        coreThreadPoolSize: 2
        queueCapacity: 10
        keepAliveDuration: 20ms
```

```java
// Spring Boot Service 계층에서 어노테이션으로 적용
@Service
public class OrderService {
    
    @Bulkhead(name = "payment-service", fallbackMethod = "paymentFallback")
    public PaymentResult processPayment(PaymentRequest request) {
        return paymentClient.process(request);
    }
    
    // Bulkhead 포화 시 자동 호출되는 폴백 메서드
    public PaymentResult paymentFallback(PaymentRequest request, BulkheadFullException ex) {
        log.warn("Payment service bulkhead full: {}", ex.getMessage());
        // 큐에 넣거나 나중에 재시도
        retryQueue.enqueue(request);
        return PaymentResult.queued(request.getOrderId());
    }
    
    @Bulkhead(name = "inventory-service", type = Bulkhead.Type.THREADPOOL)
    public CompletableFuture<InventoryResult> checkInventory(String itemId) {
        return CompletableFuture.supplyAsync(() ->
            inventoryClient.check(itemId)
        );
    }
    
    @Bulkhead(name = "notification-service", fallbackMethod = "notificationFallback")
    public void sendNotification(NotificationRequest request) {
        notificationClient.send(request);
    }
    
    public void notificationFallback(NotificationRequest request, BulkheadFullException ex) {
        // 알림은 비핵심 기능 - 조용히 드롭하고 로깅만
        log.info("Notification dropped (bulkhead full): userId={}", request.getUserId());
    }
}
```

## Circuit Breaker와의 조합

Bulkhead는 **동시성을 제한**하고, Circuit Breaker는 **실패율이 높을 때 호출 자체를 차단**합니다. 두 패턴을 함께 사용하면 시너지가 발생합니다.

```java
// Bulkhead + Circuit Breaker 조합
@CircuitBreaker(name = "payment-cb", fallbackMethod = "paymentCbFallback")
@Bulkhead(name = "payment-bh", fallbackMethod = "paymentBhFallback")
public PaymentResult processPayment(PaymentRequest request) {
    return paymentAPI.process(request);
}

// Circuit Breaker 오픈 상태일 때
public PaymentResult paymentCbFallback(PaymentRequest req, CallNotPermittedException ex) {
    return PaymentResult.circuitOpen(req.getOrderId());
}

// Bulkhead 포화 시 (Circuit Breaker보다 먼저 발동)
public PaymentResult paymentBhFallback(PaymentRequest req, BulkheadFullException ex) {
    return PaymentResult.rateLimited(req.getOrderId());
}
```

실행 순서: `Bulkhead → Circuit Breaker → Rate Limiter → Retry → Timeout → 실제 호출`

## 모니터링과 Bulkhead 크기 결정

Bulkhead의 크기를 어떻게 설정하느냐가 핵심입니다.

### Little's Law 활용

```
L = λ × W
L: 동시 처리 수 (maxConcurrentCalls)
λ: 초당 요청 수 (throughput)
W: 평균 처리 시간 (latency)

예시: 초당 100 요청, 평균 200ms 처리 → L = 100 × 0.2 = 20개
따라서 maxConcurrentCalls = 20 (여유분 1.5배 → 30)
```

### Actuator로 런타임 모니터링

```java
// Spring Actuator를 통한 메트릭 노출
// /actuator/metrics/resilience4j.bulkhead.available.concurrent.calls

// Prometheus + Grafana 대시보드 쿼리:
// resilience4j_bulkhead_available_concurrent_calls{name="payment-service"}
// resilience4j_bulkhead_max_allowed_concurrent_calls{name="payment-service"}
// resilience4j_bulkhead_buffered_calls{name="payment-service", kind="successful"}
// resilience4j_bulkhead_buffered_calls{name="payment-service", kind="failed"}
```

## 주의사항과 안티패턴

**1. 너무 작은 Bulkhead 크기**  
과도하게 작은 크기는 정상 트래픽도 차단합니다. 부하 테스트로 정상 동시성을 측정하고, 여유율 20~50%를 더해 설정하세요.

**2. 모든 의존성에 동일한 크기 적용**  
중요도와 호출 패턴이 다른 서비스에 동일한 제한을 적용하면 비효율적입니다. 핵심 서비스(결제)는 크게, 부가 서비스(알림)는 작게 설정하세요.

**3. Bulkhead만 있고 Fallback이 없는 경우**  
Bulkhead가 포화되면 예외가 발생합니다. 반드시 의미 있는 Fallback을 구현하세요. 결제는 큐에 넣기, 알림은 무시하기 등 비즈니스 맥락에 맞는 전략이 필요합니다.

**4. 스레드 풀 크기와 JVM 힙의 균형**  
스레드 풀 격리는 스레드 수를 늘립니다. 각 스레드는 기본 1MB 스택을 사용합니다. 100개 서비스에 각 10개 스레드를 할당하면 1000개 스레드 = 1GB 스택입니다. 세마포어 방식과 혼합해 균형을 잡으세요.

Bulkhead 패턴은 분산 시스템의 **방어 깊이(Defense in Depth)** 전략의 핵심입니다. 완벽한 시스템은 없습니다. 장애는 반드시 발생합니다. 중요한 것은 **장애가 얼마나 퍼지느냐**입니다. 격벽 하나가 전체 선박을 구하듯, Bulkhead 패턴 하나가 전체 서비스를 살릴 수 있습니다.

## 참고 자료
- [Microsoft Azure Architecture: Bulkhead Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/bulkhead)
- [Resilience4j Bulkhead Documentation](https://resilience4j.readme.io/docs/bulkhead)
- [Netflix Hystrix - Thread and Semaphore Isolation](https://github.com/Netflix/Hystrix/wiki/How-it-Works#Isolation)
- [Martin Fowler: Patterns of Enterprise Application Architecture](https://martinfowler.com/eaaCatalog/)
