---
layout: post
title: "회로 차단기(Circuit Breaker) 패턴 완전 정복: 마이크로서비스 장애 격리의 핵심 전략"
date: 2026-07-14
categories: [cs, computer-science]
tags: [circuit-breaker, 회로차단기, 마이크로서비스, resilience4j, 장애격리, 분산시스템, 폴백]
---

## 개요: 연쇄 장애(Cascading Failure)의 공포

마이크로서비스 아키텍처에서 서비스 A가 서비스 B를 호출하고, B는 C를 호출하는 체인이 있다고 하자. 서비스 C가 느려지기 시작하면:

1. B → C 호출이 타임아웃까지 스레드를 점유
2. B의 스레드 풀이 고갈되어 B도 응답 불가
3. A → B 호출 역시 타임아웃
4. A도 스레드 풀 고갈
5. A에 의존하는 서비스들도 연쇄 장애...

이것이 **연쇄 장애(Cascading Failure)**다. 작은 문제 하나가 전체 시스템을 마비시킨다.

**회로 차단기(Circuit Breaker) 패턴**은 전기 회로의 누전 차단기에서 영감을 받은 소프트웨어 패턴이다. 외부 서비스 호출을 감싸(wrap) 실패가 반복될 때 회로를 "열어(Open)" 이후 호출을 즉시 차단함으로써 연쇄 장애를 방지한다.

Martin Fowler가 2014년 [martinfowler.com](https://martinfowler.com/bliki/CircuitBreaker.html)에서 이 패턴을 설명했으며, Netflix Hystrix를 거쳐 오늘날 Resilience4j의 핵심 패턴이 되었다.

## 회로 차단기의 3가지 상태

### 상태 전이 다이어그램

```
                  실패율 > 임계값
   [CLOSED] ──────────────────────→ [OPEN]
      ↑                                │
      │      성공률 > 임계값           │ 대기 시간 경과
      │                               ↓
      └────────────────────── [HALF-OPEN]
            일부 요청 허용 후 판단
```

### 1. CLOSED (닫힘): 정상 동작

모든 요청이 실제 서비스로 전달된다. 실패 횟수/비율을 카운팅하며, 임계값을 초과하면 OPEN으로 전환.

```
CLOSED 상태:
  요청 → [Circuit Breaker] → 실제 서비스
  실패 시: 카운터 증가
  실패율 ≥ threshold → OPEN으로 전환
```

### 2. OPEN (열림): 차단

모든 요청을 즉시 거부한다(실제 서비스 호출 없음). `CallNotPermittedException`을 던지거나 폴백(fallback)을 실행한다. 설정된 대기 시간 후 HALF-OPEN으로 전환.

```
OPEN 상태:
  요청 → [Circuit Breaker] → 즉시 차단 (서비스 호출 없음)
  → 폴백 실행 또는 예외 발생
  대기 시간 경과 → HALF-OPEN으로 전환
```

### 3. HALF-OPEN (반열림): 복구 탐지

제한된 수의 요청만 실제 서비스로 전달한다. 성공률이 임계값을 넘으면 CLOSED로, 실패가 많으면 다시 OPEN으로 전환.

```
HALF-OPEN 상태:
  첫 N개 요청만 → 실제 서비스로 전달
  나머지 → 즉시 차단 (OPEN처럼)
  N개 중 성공률이 높으면 → CLOSED
  N개 중 실패율이 높으면 → 다시 OPEN
```

## 직접 구현: Circuit Breaker from Scratch

```python
import time
import threading
from enum import Enum
from dataclasses import dataclass, field
from collections import deque
from typing import Callable, Any, Optional
import functools

class CircuitState(Enum):
    CLOSED = "CLOSED"
    OPEN = "OPEN"
    HALF_OPEN = "HALF_OPEN"

@dataclass
class CircuitBreakerConfig:
    failure_rate_threshold: float = 50.0     # 실패율 임계값 (%)
    slow_call_rate_threshold: float = 100.0  # 느린 호출 비율 임계값 (%)
    slow_call_duration_threshold: float = 2.0  # 느린 호출 기준 (초)
    minimum_number_of_calls: int = 10        # 통계 수집 최소 호출 수
    sliding_window_size: int = 20            # 슬라이딩 윈도우 크기
    wait_duration_in_open_state: float = 30.0  # OPEN 상태 대기 시간 (초)
    permitted_calls_in_half_open: int = 5    # HALF_OPEN에서 허용 호출 수

@dataclass
class CallResult:
    success: bool
    duration: float  # 초 단위
    timestamp: float = field(default_factory=time.time)

class CircuitBreaker:
    def __init__(self, name: str, config: CircuitBreakerConfig = None):
        self.name = name
        self.config = config or CircuitBreakerConfig()
        self._state = CircuitState.CLOSED
        self._lock = threading.RLock()
        self._sliding_window: deque = deque(maxlen=self.config.sliding_window_size)
        self._open_timestamp: Optional[float] = None
        self._half_open_calls = 0  # HALF_OPEN에서 시도한 호출 수
        self._half_open_success = 0

    @property
    def state(self) -> CircuitState:
        with self._lock:
            if self._state == CircuitState.OPEN:
                elapsed = time.time() - self._open_timestamp
                if elapsed >= self.config.wait_duration_in_open_state:
                    print(f"[{self.name}] OPEN → HALF_OPEN (대기 시간 경과: {elapsed:.1f}s)")
                    self._state = CircuitState.HALF_OPEN
                    self._half_open_calls = 0
                    self._half_open_success = 0
            return self._state

    def _is_call_permitted(self) -> bool:
        s = self.state
        if s == CircuitState.CLOSED:
            return True
        elif s == CircuitState.OPEN:
            return False
        else:  # HALF_OPEN
            return self._half_open_calls < self.config.permitted_calls_in_half_open

    def _record_call(self, result: CallResult):
        with self._lock:
            self._sliding_window.append(result)

            if self._state == CircuitState.HALF_OPEN:
                self._half_open_calls += 1
                if result.success:
                    self._half_open_success += 1

                if self._half_open_calls >= self.config.permitted_calls_in_half_open:
                    success_rate = self._half_open_success / self._half_open_calls * 100
                    if success_rate >= (100 - self.config.failure_rate_threshold):
                        print(f"[{self.name}] HALF_OPEN → CLOSED (성공률: {success_rate:.1f}%)")
                        self._state = CircuitState.CLOSED
                        self._sliding_window.clear()
                    else:
                        print(f"[{self.name}] HALF_OPEN → OPEN (성공률: {success_rate:.1f}%)")
                        self._state = CircuitState.OPEN
                        self._open_timestamp = time.time()
                return

            # CLOSED 상태: 임계값 확인
            if len(self._sliding_window) < self.config.minimum_number_of_calls:
                return

            failures = sum(1 for r in self._sliding_window if not r.success)
            failure_rate = failures / len(self._sliding_window) * 100

            slow_calls = sum(
                1 for r in self._sliding_window
                if r.duration >= self.config.slow_call_duration_threshold
            )
            slow_rate = slow_calls / len(self._sliding_window) * 100

            should_open = (
                failure_rate >= self.config.failure_rate_threshold or
                slow_rate >= self.config.slow_call_rate_threshold
            )

            if should_open:
                print(f"[{self.name}] CLOSED → OPEN "
                      f"(실패율: {failure_rate:.1f}%, 느린호출: {slow_rate:.1f}%)")
                self._state = CircuitState.OPEN
                self._open_timestamp = time.time()

    def call(self, func: Callable, *args, fallback: Callable = None, **kwargs) -> Any:
        if not self._is_call_permitted():
            print(f"[{self.name}] 호출 차단됨 (상태: {self._state.value})")
            if fallback:
                return fallback(*args, **kwargs)
            raise RuntimeError(f"Circuit breaker '{self.name}' is OPEN")

        start = time.time()
        try:
            result = func(*args, **kwargs)
            duration = time.time() - start
            self._record_call(CallResult(success=True, duration=duration))
            return result
        except Exception as e:
            duration = time.time() - start
            self._record_call(CallResult(success=False, duration=duration))
            if fallback:
                return fallback(*args, **kwargs)
            raise

    def decorator(self, fallback: Callable = None):
        """데코레이터 방식으로 사용"""
        def wrapper(func):
            @functools.wraps(func)
            def inner(*args, **kwargs):
                return self.call(func, *args, fallback=fallback, **kwargs)
            return inner
        return wrapper

    def get_metrics(self) -> dict:
        with self._lock:
            if not self._sliding_window:
                return {"state": self._state.value, "calls": 0}
            failures = sum(1 for r in self._sliding_window if not r.success)
            return {
                "state": self._state.value,
                "total_calls": len(self._sliding_window),
                "failure_rate": failures / len(self._sliding_window) * 100,
                "avg_duration": sum(r.duration for r in self._sliding_window) / len(self._sliding_window),
            }


# === 시뮬레이션 ===
import random

config = CircuitBreakerConfig(
    failure_rate_threshold=50.0,
    minimum_number_of_calls=5,
    sliding_window_size=10,
    wait_duration_in_open_state=3.0,
    permitted_calls_in_half_open=3,
)
cb = CircuitBreaker("payment-service", config)

def payment_service_call(amount: float, fail: bool = False) -> str:
    if fail:
        raise ConnectionError("Payment service unavailable")
    time.sleep(0.01)
    return f"결제 완료: {amount}원"

def fallback_handler(amount: float, fail: bool = False) -> str:
    return f"결제 대기열에 추가: {amount}원 (나중에 처리)"

print("=== 정상 요청 5회 ===")
for i in range(5):
    result = cb.call(payment_service_call, 1000, fallback=fallback_handler)
    print(f"  {result}")

print("\n=== 연속 실패 7회 (차단기 개방 예상) ===")
for i in range(7):
    result = cb.call(payment_service_call, 1000, fail=True, fallback=fallback_handler)
    print(f"  {result}")

print(f"\n현재 메트릭: {cb.get_metrics()}")

print("\n=== OPEN 상태에서 호출 시도 ===")
result = cb.call(payment_service_call, 1000, fallback=fallback_handler)
print(f"  {result}")

print("\n=== 대기 후 HALF-OPEN 복구 시뮬레이션 ===")
time.sleep(3.1)  # wait_duration_in_open_state 경과
for i in range(3):
    result = cb.call(payment_service_call, 1000, fallback=fallback_handler)
    print(f"  {result}")

print(f"\n최종 메트릭: {cb.get_metrics()}")
```

## Resilience4j를 사용한 Spring Boot 실전 구현

실제 프로덕션에서는 검증된 라이브러리를 사용하는 것이 좋다.

### 의존성 설정 (build.gradle)

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-aop'
    implementation 'io.github.resilience4j:resilience4j-spring-boot3:2.2.0'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
}
```

### application.yml 설정

```yaml
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        # 슬라이딩 윈도우: COUNT_BASED 또는 TIME_BASED
        slidingWindowType: COUNT_BASED
        slidingWindowSize: 20
        minimumNumberOfCalls: 10
        
        # 임계값
        failureRateThreshold: 50          # 실패율 50% 이상이면 OPEN
        slowCallRateThreshold: 80         # 느린 호출 비율 80% 이상이면 OPEN
        slowCallDurationThreshold: 3000   # 3초 이상 = 느린 호출 (ms)
        
        # 상태 전환
        waitDurationInOpenState: 30s      # OPEN 유지 시간
        permittedNumberOfCallsInHalfOpenState: 5
        automaticTransitionFromOpenToHalfOpenEnabled: true
        
        # 예외 설정
        recordExceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
          - org.springframework.web.client.HttpServerErrorException
        ignoreExceptions:
          - com.example.BusinessException  # 비즈니스 예외는 회로 차단 카운트 제외

  retry:
    instances:
      paymentService:
        maxAttempts: 3
        waitDuration: 500ms
        retryExceptions:
          - java.io.IOException
```

### 서비스 구현

```java
package com.example.payment;

import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import io.github.resilience4j.retry.annotation.Retry;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;

@Slf4j
@Service
@RequiredArgsConstructor
public class PaymentService {

    private final RestTemplate restTemplate;
    private static final String CIRCUIT_BREAKER_NAME = "paymentService";

    /**
     * Circuit Breaker + Retry 조합
     * Retry가 먼저 재시도하고, 계속 실패하면 Circuit Breaker가 열림
     */
    @CircuitBreaker(name = CIRCUIT_BREAKER_NAME, fallbackMethod = "paymentFallback")
    @Retry(name = CIRCUIT_BREAKER_NAME)
    public PaymentResponse processPayment(PaymentRequest request) {
        log.info("결제 처리 요청: {}", request.getAmount());
        
        String url = "http://external-payment-gateway/api/pay";
        return restTemplate.postForObject(url, request, PaymentResponse.class);
    }

    /**
     * 폴백 메서드: 같은 파라미터 + Throwable
     * Circuit Breaker가 열리거나 예외 발생 시 호출
     */
    private PaymentResponse paymentFallback(PaymentRequest request, Throwable ex) {
        log.warn("결제 서비스 폴백 실행: {} - {}", 
                 ex.getClass().getSimpleName(), ex.getMessage());

        // 전략 1: 캐시된 응답 반환
        // 전략 2: 대기열에 추가 후 나중에 처리
        // 전략 3: 에러 응답 반환
        return PaymentResponse.builder()
            .status("QUEUED")
            .message("결제가 대기열에 등록되었습니다. 곧 처리됩니다.")
            .requestId(request.getRequestId())
            .build();
    }

    /**
     * 프로그래매틱 방식 (어노테이션 외 세밀한 제어 필요 시)
     */
    public PaymentResponse processPaymentProgrammatic(PaymentRequest request) {
        io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry registry =
            io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry.ofDefaults();
        
        io.github.resilience4j.circuitbreaker.CircuitBreaker cb =
            registry.circuitBreaker(CIRCUIT_BREAKER_NAME);

        // 이벤트 리스너 등록
        cb.getEventPublisher()
            .onStateTransition(event -> 
                log.info("상태 변경: {} → {}", 
                    event.getStateTransition().getFromState(),
                    event.getStateTransition().getToState()))
            .onError(event ->
                log.error("오류 발생: {}", event.getThrowable().getMessage()));

        return io.github.resilience4j.circuitbreaker.CircuitBreaker
            .decorateSupplier(cb, () -> callExternalService(request))
            .get();
    }

    private PaymentResponse callExternalService(PaymentRequest request) {
        // 외부 서비스 호출
        return new PaymentResponse();
    }
}
```

### Actuator로 메트릭 모니터링

```bash
# 현재 회로 차단기 상태 확인
curl http://localhost:8080/actuator/circuitbreakers

# 응답 예시:
{
  "circuitBreakers": {
    "paymentService": {
      "state": "CLOSED",
      "metrics": {
        "failureRate": "12.50%",
        "slowCallRate": "0.00%",
        "numberOfBufferedCalls": 20,
        "numberOfFailedCalls": 2,
        "numberOfSlowCalls": 0
      }
    }
  }
}
```

## Bulkhead 패턴과 함께 사용하기

Circuit Breaker만으로는 부족하다. **Bulkhead(격벽) 패턴**을 함께 사용하면 스레드 풀을 서비스별로 격리해 연쇄 장애를 더 철저히 방지할 수 있다.

```yaml
resilience4j:
  bulkhead:
    instances:
      paymentService:
        maxConcurrentCalls: 20      # 동시 호출 최대 20개
        maxWaitDuration: 500ms      # 대기 최대 500ms

  thread-pool-bulkhead:
    instances:
      paymentService:
        maxThreadPoolSize: 10       # 전용 스레드 풀 크기
        coreThreadPoolSize: 5
        queueCapacity: 20
```

```java
@CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
@Bulkhead(name = "paymentService", type = Bulkhead.Type.THREADPOOL)
@TimeLimiter(name = "paymentService")
public CompletableFuture<PaymentResponse> processPaymentAsync(PaymentRequest request) {
    return CompletableFuture.supplyAsync(() -> callExternalService(request));
}
```

## 주의사항과 실전 팁

### 1. 임계값 설정의 함정

```
잘못된 설정:
  minimum_calls = 2, failure_threshold = 50%
  → 2번 호출 중 1번 실패하면 즉시 OPEN
  → 정상적인 일시적 오류에도 차단

올바른 설정:
  minimum_calls = 10~20 (충분한 샘플)
  failure_threshold = 50~60% (환경에 맞게)
  sliding_window = 20~100 (호출 빈도 고려)
```

### 2. 폴백 전략 계층화

```
1단계: 캐시된 이전 응답 반환
2단계: 기본값(default value) 반환
3단계: 대기열에 추가 후 비동기 처리
4단계: 사용자에게 명확한 오류 메시지 반환
   (차단 이유 숨기지 않기: "결제 서비스 일시 중단, 10분 후 재시도")
```

### 3. Circuit Breaker ≠ Retry의 무분별한 조합

```
잘못된 패턴:
  @Retry(maxAttempts=5)
  @CircuitBreaker(threshold=50%, minCalls=10)
  → 5번 재시도 × N개 요청 = 실제로는 10배 요청
  → 이미 죽어가는 서비스에 더 많은 부하

올바른 패턴:
  Retry maxAttempts=2~3 (빠른 재시도)
  Circuit Breaker가 열리면 Retry는 실행되지 않음 (Resilience4j가 처리)
```

### 4. 헬스체크와 연동

OPEN 상태에서 주기적으로 대상 서비스를 헬스체크해 복구를 빨리 감지하려면 `automaticTransitionFromOpenToHalfOpenEnabled: true`를 활성화하고, 별도 헬스체크 엔드포인트(`/health`)를 활용한다.

### 5. 분산 환경에서의 상태 공유

여러 인스턴스가 뜬 경우 각자 독립적인 Circuit Breaker 상태를 갖는다. Redis 같은 공유 저장소를 이용해 상태를 동기화할 수 있지만, 오히려 복잡성이 증가하는 경우가 많다. 대부분의 경우 인스턴스별 독립 상태로 충분하다.

## 마무리

Circuit Breaker 패턴은 마이크로서비스의 **탄력성(Resilience)**을 위한 핵심 도구다. 장애를 막는 것이 아니라, 장애가 전파되는 것을 막는다는 철학을 기억해야 한다. 

Resilience4j의 Circuit Breaker, Retry, Bulkhead, TimeLimiter, RateLimiter를 조합하면 대부분의 분산 시스템 장애 시나리오를 커버할 수 있다. 중요한 것은 각 패턴을 이해하고, 실제 서비스 특성(호출 빈도, 허용 가능한 오류율, 복구 시간)에 맞게 임계값을 튜닝하는 것이다.

## 참고 자료

- [Circuit Breaker — Martin Fowler](https://martinfowler.com/bliki/CircuitBreaker.html)
- [Resilience4j Circuit Breaker 공식 문서](https://resilience4j.readme.io/docs/circuitbreaker)
- [Spring Boot Circuit Breaker with Resilience4j — GeeksforGeeks](https://www.geeksforgeeks.org/advance-java/spring-boot-circuit-breaker-pattern-with-resilience4j/)
- [Comprehensive Guide to Resilience4j — Medium](https://medium.com/@bolot.89/comprehensive-guide-to-resilience4j-and-the-circuit-breaker-pattern-85c6349d3535)
