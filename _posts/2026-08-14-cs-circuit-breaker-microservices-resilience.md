---
layout: post
title: "서킷 브레이커 패턴 완전 정복: 마이크로서비스 장애 전파를 막는 복원력 설계"
date: 2026-08-14
categories: [cs, computer-science]
tags: [circuit-breaker, microservices, resilience, resilience4j, fault-tolerance, design-pattern, java, spring-boot]
---

## 서킷 브레이커 패턴이란

마이크로서비스 아키텍처에서는 수십 개의 서비스가 네트워크를 통해 서로 호출합니다. 여기에는 근본적인 위험이 내재해 있습니다: 결제 서비스가 느려지거나 응답을 멈추면 어떤 일이 벌어질까요?

주문 서비스가 결제 서비스를 호출하고 30초 타임아웃을 기다리는 동안 스레드 풀의 스레드 하나가 블록됩니다. 동시에 수천 개의 주문 요청이 들어오면, 결제 서비스 호출 대기로 인해 주문 서비스의 전체 스레드 풀이 고갈됩니다. 이제 주문 서비스도 응답 불능이 되고, 주문 서비스를 호출하던 API 게이트웨이도 연쇄적으로 마비됩니다. 단 하나의 서비스 장애가 전체 시스템을 무너뜨리는 **장애 전파(Cascading Failure)**입니다.

**서킷 브레이커(Circuit Breaker)** 패턴은 전기 회로 차단기에서 영감을 얻었습니다. 2007년 마이클 나이가드(Michael Nygard)가 저서 "Release It!"에서 체계화했으며, 마틴 파울러(Martin Fowler)의 2014년 블로그 포스트로 광범위하게 알려졌습니다. 하위 서비스 호출이 일정 횟수 이상 실패하면 회로를 **차단(Open)**하여 더 이상 호출하지 않고 즉시 실패를 반환합니다.

## 왜 단순 Retry만으로는 부족한가

재시도(Retry) 패턴은 서킷 브레이커와 자주 함께 언급되지만, 혼자서는 장애 전파를 막기 어렵습니다:

**Retry Storm**: 결제 서비스가 장애 상태일 때, 수천 개의 클라이언트가 각자 3번씩 재시도하면 정상 상태보다 3배 많은 트래픽이 이미 과부하 상태인 서비스로 쏟아집니다.

**스레드 고갈**: 재시도마다 타임아웃을 기다리며 스레드를 점유합니다. 타임아웃이 30초라면 재시도 3회에 최장 90초 동안 스레드 하나가 블록됩니다.

**레이턴시 폭발**: 사용자 입장에서 각 재시도 + 타임아웃 시간의 합만큼 응답이 지연됩니다.

서킷 브레이커는 **빠른 실패(Fail Fast)** 원칙을 구현합니다. 장애 중인 서비스에 계속 호출하는 것보다, 즉시 실패 응답을 반환하고 복구를 기다리는 것이 전체 시스템 안정성을 높입니다.

## 서킷 브레이커의 세 가지 상태 전이

```
                실패율 >= 임계값
  ┌─ CLOSED ──────────────────────► OPEN ──────────────────┐
  │  (정상 운영)                   (완전 차단)              │
  │                                    │ 대기 시간 경과     │
  │                              HALF_OPEN                  │
  │                              (부분 통과)                 │
  │        성공률 >= 임계값             │                   │
  └────────────────────────────────────┘                   │
           실패 → 다시 OPEN ─────────────────────────────►┘
```

### CLOSED (정상 운영)

모든 요청이 실제 서비스로 전달됩니다. 성공/실패를 슬라이딩 윈도우(카운트 기반 또는 시간 기반)로 집계합니다. 실패율이 임계값(예: 50%)을 초과하면 **OPEN** 상태로 전환합니다. 최소 호출 횟수 미달 시에는 임계값 계산을 하지 않아 콜드 스타트 시 오탐을 방지합니다.

### OPEN (완전 차단)

모든 요청을 즉시 차단하고 `CallNotPermittedException`을 반환합니다. 실제 서비스 호출 없이 Fallback 로직만 실행되므로 응답 지연이 거의 없습니다. 설정된 대기 시간(예: 60초)이 경과하면 **HALF_OPEN** 상태로 전환합니다.

### HALF_OPEN (탐색)

제한된 수의 요청(예: 5개)만 실제 서비스로 통과시켜 복구 여부를 확인합니다. 통과된 요청 중 성공 비율이 임계값 이상이면 **CLOSED**로 복귀하고, 아니면 다시 **OPEN**으로 전환합니다.

## Resilience4j 실제 구현

```groovy
// build.gradle
dependencies {
    implementation 'io.github.resilience4j:resilience4j-spring-boot3:2.2.0'
    implementation 'org.springframework.boot:spring-boot-starter-aop'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'io.micrometer:micrometer-registry-prometheus'
}
```

```yaml
# application.yml
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        sliding-window-type: COUNT_BASED
        sliding-window-size: 100
        minimum-number-of-calls: 20
        failure-rate-threshold: 50
        wait-duration-in-open-state: 60000
        permitted-number-of-calls-in-half-open-state: 5
        slow-call-duration-threshold: 2000ms
        slow-call-rate-threshold: 80
        record-exceptions:
          - java.io.IOException
          - java.net.SocketTimeoutException
          - feign.FeignException.ServiceUnavailable
        ignore-exceptions:
          - com.example.BusinessValidationException
```

```java
import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry;
import org.springframework.stereotype.Service;

@Service
public class OrderService {

    private final PaymentServiceClient paymentClient;
    private final PendingPaymentQueue pendingQueue;

    @Autowired
    private CircuitBreakerRegistry circuitBreakerRegistry;

    public OrderService(PaymentServiceClient paymentClient,
                        PendingPaymentQueue pendingQueue) {
        this.paymentClient = paymentClient;
        this.pendingQueue  = pendingQueue;
    }

    @CircuitBreaker(name = "paymentService", fallbackMethod = "processPaymentFallback")
    public PaymentResult processPayment(Order order) {
        return paymentClient.charge(new ChargeRequest(
            order.getAmount(),
            order.getPaymentInfo()
        ));
    }

    private PaymentResult processPaymentFallback(Order order,
                                                  CallNotPermittedException ex) {
        log.warn("결제 서비스 서킷 OPEN. 주문 큐에 저장: orderId={}", order.getId());
        pendingQueue.enqueue(order);
        return PaymentResult.pending(
            "결제 서비스 점검 중입니다. 주문은 저장되었으며 잠시 후 자동 처리됩니다.");
    }

    private PaymentResult processPaymentFallback(Order order,
                                                  java.net.SocketTimeoutException ex) {
        return PaymentResult.retryable("결제 처리 시간이 초과되었습니다. 다시 시도해 주세요.");
    }

    private PaymentResult processPaymentFallback(Order order, Throwable ex) {
        log.error("결제 처리 실패: orderId={}", order.getId(), ex);
        return PaymentResult.failure("결제 처리 중 오류가 발생했습니다.");
    }

    public CircuitBreakerStatus getCircuitBreakerStatus() {
        io.github.resilience4j.circuitbreaker.CircuitBreaker cb =
            circuitBreakerRegistry.circuitBreaker("paymentService");
        io.github.resilience4j.circuitbreaker.CircuitBreaker.Metrics m = cb.getMetrics();
        return CircuitBreakerStatus.builder()
            .state(cb.getState().name())
            .failureRate(m.getFailureRate())
            .slowCallRate(m.getSlowCallRate())
            .successfulCalls(m.getNumberOfSuccessfulCalls())
            .failedCalls(m.getNumberOfFailedCalls())
            .notPermittedCalls(m.getNumberOfNotPermittedCalls())
            .build();
    }
}
```

## Retry, Bulkhead, TimeLimiter와의 조합

실제 운영 환경에서는 서킷 브레이커를 단독으로 사용하지 않고, 여러 복원력 패턴을 레이어로 조합합니다:

```yaml
resilience4j:
  retry:
    instances:
      externalApi:
        max-attempts: 3
        wait-duration: 500ms
        exponential-backoff-multiplier: 2
        retry-exceptions:
          - java.net.ConnectException
          - java.net.SocketTimeoutException
        ignore-exceptions:
          - com.example.ClientErrorException

  bulkhead:
    instances:
      externalApi:
        max-concurrent-calls: 25
        max-wait-duration: 500ms

  timelimiter:
    instances:
      externalApi:
        timeout-duration: 3s
        cancel-running-future: true
```

```java
@Service
public class ExternalApiService {

    // 실행 순서 (바깥쪽 → 안쪽):
    //   Bulkhead → CircuitBreaker → TimeLimiter → Retry → 실제 호출
    @Bulkhead(name = "externalApi", type = Bulkhead.Type.THREADPOOL)
    @CircuitBreaker(name = "externalApi", fallbackMethod = "apiFallback")
    @TimeLimiter(name = "externalApi")
    @Retry(name = "externalApi")
    public CompletableFuture<String> callExternalApi(String request) {
        return CompletableFuture.supplyAsync(() ->
            externalApiClient.call(request)
        );
    }

    private CompletableFuture<String> apiFallback(String request, Exception ex) {
        String cached = cacheService.getLastKnownGood(request);
        return CompletableFuture.completedFuture(
            cached != null ? cached : "서비스 임시 불가 - 기본값 반환"
        );
    }
}
```

## 메트릭과 모니터링

서킷 브레이커의 상태를 실시간으로 관찰해야 임계값 튜닝과 장애 조기 감지가 가능합니다:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, metrics, circuitbreakers, circuitbreakerevents
  health:
    circuitbreakers:
      enabled: true
```

```bash
curl http://localhost:8080/actuator/health | jq '.components.circuitBreakers'

# Prometheus 메트릭
resilience4j_circuitbreaker_state{name="paymentService"} 0.0
# 0=CLOSED, 1=OPEN, 2=HALF_OPEN

resilience4j_circuitbreaker_failure_rate{name="paymentService"} 12.5
resilience4j_circuitbreaker_calls_total{name="paymentService",kind="successful"} 87
resilience4j_circuitbreaker_calls_total{name="paymentService",kind="failed"} 13
resilience4j_circuitbreaker_calls_total{name="paymentService",kind="not_permitted"} 0
```

```java
@PostConstruct
public void setupListeners() {
    io.github.resilience4j.circuitbreaker.CircuitBreaker cb =
        registry.circuitBreaker("paymentService");

    cb.getEventPublisher()
        .onStateTransition(event -> {
            log.warn("[서킷 브레이커] 상태 전환: {} → {}",
                event.getStateTransition().getFromState(),
                event.getStateTransition().getToState());
            alertService.notify(
                "결제 서비스 서킷 브레이커 상태 변경: "
                + event.getStateTransition().getToState()
            );
        })
        .onFailureRateExceeded(event ->
            log.error("[서킷 브레이커] 실패율 임계값 초과: {}%",
                event.getFailureRate())
        );
}
```

## 주의사항과 팁

### 1. 임계값 설정의 함정

**너무 낮은 임계값**: 일시적인 네트워크 지터(jitter)에도 서킷이 열려 False Positive 발생. 정상 서비스를 불필요하게 차단.

**너무 높은 임계값**: 실제 장애가 심각해져도 서킷이 열리지 않아 장애 전파를 막지 못함.

권장 접근법:
1. 운영 환경에서 최소 2주간 오류율을 측정해 베이스라인 확보
2. `minimum-number-of-calls`를 트래픽의 5~10%에 해당하는 수치로 설정
3. 임계값을 베이스라인의 3~5배로 시작해 점진적으로 조정

### 2. Fallback 설계 원칙

**Fallback 내에서 외부 서비스를 다시 호출하지 마세요.** Fallback이 또 다른 실패 지점이 되어서는 안 됩니다:

```java
// 나쁜 예: fallback에서 또 다른 외부 서비스 호출
private PaymentResult fallback(Order order, Exception ex) {
    return backupPaymentService.charge(order);  // 이것도 실패하면?
}

// 좋은 예: 의존성 없는 로컬 로직
private PaymentResult fallback(Order order, Exception ex) {
    localQueue.save(order);
    return PaymentResult.pending("잠시 후 처리됩니다");
}
```

### 3. 분산 서킷 브레이커의 한계

각 서비스 인스턴스가 독립적인 서킷 브레이커를 유지하면, 10개의 인스턴스 중 하나만 서킷이 열려도 나머지 9개는 계속 호출합니다. 이를 해결하려면:
- **Redis 기반 공유 카운터**: 전체 인스턴스의 실패율을 중앙 집계
- **Service Mesh (Istio 등)**: 사이드카 프록시에서 서킷 브레이커를 처리해 코드 변경 없이 적용

### 4. 서킷 브레이커가 필요 없는 경우

모든 외부 호출에 서킷 브레이커를 적용하는 것은 과잉 설계일 수 있습니다. 다음 경우에는 단순 Retry나 타임아웃만으로도 충분합니다:
- 동일 프로세스 내 함수 호출
- 동기적 DB 쿼리 (커넥션 풀로 이미 격리)
- 매우 빠른 응답이 보장된 내부 마이크로서비스

## 참고 자료
- [Martin Fowler - Circuit Breaker](https://martinfowler.com/bliki/CircuitBreaker.html)
- [Resilience4j 공식 문서](https://resilience4j.readme.io/docs/circuitbreaker)
- [Spring Cloud Circuit Breaker](https://docs.spring.io/spring-cloud-circuitbreaker/docs/current/reference/html/)
- [Release It! - Michael T. Nygard](https://pragprog.com/titles/mnee2/release-it-second-edition/)
