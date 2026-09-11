---
layout: post
title: "LMAX Disruptor 완전 정복: 락 없는 초고성능 이벤트 처리 아키텍처"
date: 2026-09-11
categories: [cs, computer-science]
tags: [disruptor, ring-buffer, lock-free, concurrency, java, high-performance, event-processing]
---

초당 수백만 건의 금융 거래를 단일 서버에서 처리하려면 어떻게 해야 할까요? LMAX Exchange의 엔지니어들은 이 문제를 해결하기 위해 2011년 **Disruptor**라는 혁신적인 라이브러리를 발표했습니다. 전통적인 락 기반 큐를 완전히 버리고, CPU와 메모리 아키텍처의 특성을 극한까지 활용한 이 설계는 당시 업계에 큰 충격을 주었습니다. 이 글에서는 Disruptor의 내부 구조와 그것이 왜 이렇게 빠른지 깊이 파헤칩니다.

## Disruptor란 무엇인가

Disruptor는 **생산자-소비자 패턴을 구현하는 고성능 링 버퍼(Ring Buffer)** 라이브러리입니다. Java의 전통적인 `BlockingQueue`를 대체하며, 스레드 간 데이터 교환에서 압도적인 성능을 제공합니다.

LMAX는 Disruptor를 자사의 전자 거래 플랫폼 핵심 컴포넌트로 사용하며, 단일 스레드로 초당 **6백만 건 이상의 주문**을 처리하는 성능을 달성했습니다. 이는 `ArrayBlockingQueue` 대비 8배 이상 빠른 수치입니다.

### 전통적 큐의 문제점

Java의 `LinkedBlockingQueue`나 `ArrayBlockingQueue`는 스레드 안전성을 위해 내부적으로 `ReentrantLock`이나 `synchronized`를 사용합니다.

```java
// 전통적 BlockingQueue의 내부 (단순화)
public class ArrayBlockingQueue<E> {
    final Object[] items;
    int putIndex, takeIndex, count;
    final ReentrantLock lock = new ReentrantLock(); // 병목!
    
    public void put(E e) throws InterruptedException {
        lock.lockInterruptibly(); // 모든 연산이 여기서 직렬화됨
        try {
            while (count == items.length)
                notFull.await();
            enqueue(e);
        } finally {
            lock.unlock();
        }
    }
}
```

락 경쟁(lock contention)이 발생하면 스레드들이 대기 상태에 들어가고 컨텍스트 스위칭 오버헤드가 발생합니다. 또한 `LinkedBlockingQueue`의 경우 노드 객체를 계속 할당하고 해제하면서 GC 압박도 커집니다.

## 왜 Disruptor가 필요한가: CPU 아키텍처의 활용

Disruptor의 성능 비밀은 **현대 CPU와 메모리 시스템의 특성**을 최대한 활용하는 데 있습니다.

### 1. 캐시 라인 최적화와 False Sharing 방지

CPU 캐시는 64바이트(일반적으로) 단위인 **캐시 라인** 으로 데이터를 관리합니다. 서로 다른 스레드가 같은 캐시 라인의 다른 필드를 수정하면 **False Sharing**이 발생해 성능이 크게 저하됩니다.

```java
// False Sharing 발생 예시 (나쁜 패턴)
class BadCounter {
    volatile long producer = 0; // 같은 캐시 라인에 있을 수 있음!
    volatile long consumer = 0;
}

// Disruptor의 해법: 패딩으로 캐시 라인 격리
// (Java 8+ 이전 방식)
class RingBufferPad {
    protected long p1, p2, p3, p4, p5, p6, p7; // 앞 패딩 56바이트
}

class RingBufferFields<E> extends RingBufferPad {
    private long indexMask;          // 8바이트 → 합계 64바이트 = 1 캐시 라인
    private Object[] entries;
    // ...
}

class RingBuffer<E> extends RingBufferFields<E> {
    protected long p1, p2, p3, p4, p5, p6, p7; // 뒤 패딩 56바이트
    // cursor 시퀀스가 다음 캐시 라인에 위치하도록 보장
}

// Java 8+ 공식 방식: @Contended
@sun.misc.Contended
class Sequence {
    volatile long value = -1L;
}
```

이렇게 하면 `producer`와 `consumer` 시퀀스가 서로 다른 캐시 라인에 위치하여 False Sharing을 완전히 방지합니다.

### 2. 링 버퍼의 장점

링 버퍼는 고정 크기의 배열을 순환하며 사용합니다. 크기는 반드시 **2의 거듭제곱**으로 설정합니다.

```
링 버퍼 구조 (크기 = 8):

인덱스: [0][1][2][3][4][5][6][7]
          ^                 ^
          consumer          producer
          cursor            cursor

모듈로 연산 대신 비트마스킹 사용:
index = sequence & (BUFFER_SIZE - 1)
= sequence & 7  (BUFFER_SIZE=8 → mask=7=0b0111)

이 연산은 % 연산보다 수십 배 빠름!
```

**메모리 사전 할당**: 링 버퍼의 모든 슬롯을 처음부터 생성해 놓습니다. 실제 데이터 교환 시에는 객체를 생성하는 대신 기존 객체의 필드만 업데이트합니다. 이로써 GC 부담이 거의 없어집니다.

### 3. 배리어(Barrier) 기반 의존성 관리

Disruptor는 여러 생산자와 소비자, 그리고 소비자 간 의존성을 **시퀀스 배리어(Sequence Barrier)**로 관리합니다.

```
         생산자
           │
      [RingBuffer]
           │
    SequenceBarrier
           │
     ┌─────┴────────────┐
     │                  │
  소비자A             소비자B
  (로깅)           (복제)
     │                  │
     └────── SequenceBarrier ──────┐
                                   │
                                소비자C
                               (비즈니스 로직)
                               A와 B 완료 후 처리
```

## Disruptor 구현: 실전 코드

### 기본 설정과 사용법

```java
import com.lmax.disruptor.*;
import com.lmax.disruptor.dsl.Disruptor;
import com.lmax.disruptor.dsl.ProducerType;
import java.util.concurrent.Executors;

// 1. 이벤트 정의 - 미리 할당될 객체
public class OrderEvent {
    private long orderId;
    private String symbol;
    private double price;
    private int quantity;
    private char side; // 'B' = Buy, 'S' = Sell
    
    // getter/setter
    public void set(long orderId, String symbol, double price, 
                    int quantity, char side) {
        this.orderId = orderId;
        this.symbol = symbol;
        this.price = price;
        this.quantity = quantity;
        this.side = side;
    }
    
    public long getOrderId() { return orderId; }
    public String getSymbol() { return symbol; }
    // ...
}

// 2. 이벤트 팩토리 - 미리 할당
public class OrderEventFactory implements EventFactory<OrderEvent> {
    @Override
    public OrderEvent newInstance() {
        return new OrderEvent(); // 한 번만 호출됨!
    }
}

// 3. 이벤트 핸들러 - 소비자 로직
public class OrderProcessor implements EventHandler<OrderEvent> {
    private final String name;
    private long processedCount = 0;
    
    public OrderProcessor(String name) { this.name = name; }
    
    @Override
    public void onEvent(OrderEvent event, long sequence, 
                        boolean endOfBatch) {
        // 객체 생성 없음! 링 버퍼의 기존 객체를 읽음
        processedCount++;
        System.out.printf("[%s] seq=%d orderId=%d %s %s@%.2f x%d%n",
            name, sequence, event.getOrderId(),
            event.getSide() == 'B' ? "BUY" : "SELL",
            event.getSymbol(), event.getPrice(), 
            event.getQuantity());
        
        // endOfBatch: 현재 배치의 마지막 이벤트인지 여부
        // true이면 I/O 플러시 등 배치 종료 작업 수행
        if (endOfBatch) {
            flush();
        }
    }
    
    private void flush() { /* DB flush, 로그 플러시 등 */ }
}

// 4. Disruptor 조립 및 실행
public class TradingSystem {
    private static final int BUFFER_SIZE = 1024; // 반드시 2의 거듭제곱
    
    public static void main(String[] args) throws Exception {
        OrderEventFactory factory = new OrderEventFactory();
        
        // SINGLE: 생산자가 1개, MULTI: 생산자가 여러 개
        // SINGLE이 훨씬 빠름 (CAS 불필요)
        Disruptor<OrderEvent> disruptor = new Disruptor<>(
            factory,
            BUFFER_SIZE,
            Executors.defaultThreadFactory(),
            ProducerType.SINGLE,
            new YieldingWaitStrategy() // 낮은 지연 vs CPU 사용량 트레이드오프
        );
        
        // 소비자 체인 설정: logger와 replicator가 병렬로, 그 다음 processor
        OrderProcessor logger = new OrderProcessor("Logger");
        OrderProcessor replicator = new OrderProcessor("Replicator");
        OrderProcessor businessLogic = new OrderProcessor("BusinessLogic");
        
        disruptor
            .handleEventsWith(logger, replicator)  // 병렬 처리
            .then(businessLogic);                   // 그 다음에 처리
        
        // 시작
        RingBuffer<OrderEvent> ringBuffer = disruptor.start();
        
        // 생산자: 이벤트 발행
        long startTime = System.nanoTime();
        for (int i = 0; i < 10_000_000; i++) {
            // claim → populate → publish 3단계
            long sequence = ringBuffer.next(); // 다음 슬롯 예약
            try {
                OrderEvent event = ringBuffer.get(sequence); // 슬롯 참조
                event.set(i, "AAPL", 150.0 + (i % 100) * 0.01,
                           100 + (i % 50), i % 2 == 0 ? 'B' : 'S');
            } finally {
                ringBuffer.publish(sequence); // 발행 (소비자가 볼 수 있게)
            }
        }
        
        long elapsed = System.nanoTime() - startTime;
        System.out.printf("10M events in %.2f ms (%.1f M/s)%n",
            elapsed / 1_000_000.0, 10_000.0 / (elapsed / 1_000_000.0));
        
        disruptor.shutdown();
    }
}
```

### 배치 생산자를 활용한 더 높은 처리량

```java
// 여러 이벤트를 한 번에 예약하여 오버헤드 절감
public class BatchPublisher {
    private final RingBuffer<OrderEvent> ringBuffer;
    
    public void publishBatch(List<OrderData> orders) {
        int batchSize = orders.size();
        
        // 연속된 시퀀스 범위를 한 번에 예약
        long highSequence = ringBuffer.next(batchSize);
        long lowSequence = highSequence - (batchSize - 1);
        
        try {
            for (int i = 0; i < batchSize; i++) {
                long seq = lowSequence + i;
                OrderEvent event = ringBuffer.get(seq);
                OrderData data = orders.get(i);
                event.set(data.getId(), data.getSymbol(), 
                          data.getPrice(), data.getQty(), data.getSide());
            }
        } finally {
            // 전체 배치를 한 번에 발행
            ringBuffer.publish(lowSequence, highSequence);
        }
    }
}
```

## 대기 전략(Wait Strategy) 선택

소비자가 새 이벤트를 기다리는 방식을 `WaitStrategy`로 결정합니다. 트레이드오프가 명확하므로 요구사항에 맞게 선택해야 합니다.

| 전략 | 지연 | CPU 사용 | 용도 |
|------|------|----------|------|
| `BusySpinWaitStrategy` | 매우 낮음 | 100% (1코어) | 초지연 요구, 코어 독점 가능할 때 |
| `YieldingWaitStrategy` | 낮음 | 높음 | 지연 민감, 멀티코어 |
| `SleepingWaitStrategy` | 중간 | 낮음 | 처리량 중요, 지연 덜 중요 |
| `BlockingWaitStrategy` | 높음 | 매우 낮음 | CPU 공유 환경, 처리량 중심 |
| `LiteBlockingWaitStrategy` | 높음 | 매우 낮음 | BlockingWaitStrategy의 경량 버전 |

```java
// 초저지연 요구 시 (HFT, 게이밍)
new BusySpinWaitStrategy()

// 균형 잡힌 선택 (대부분의 경우)
new YieldingWaitStrategy()

// CPU 효율 우선 (배치 처리, 백오피스)
new SleepingWaitStrategy(0, 1_000_000) // 1ms sleep
```

## 주의사항 및 팁

### 1. 예외 처리: ExceptionHandler 설정 필수

이벤트 핸들러에서 예외가 발생하면 기본적으로 스레드가 종료됩니다. 반드시 `ExceptionHandler`를 설정하세요.

```java
disruptor.setDefaultExceptionHandler(new ExceptionHandler<OrderEvent>() {
    @Override
    public void handleEventException(Throwable ex, long sequence, 
                                     OrderEvent event) {
        // 로깅, 알람, 재시도 로직
        log.error("Failed to process event seq={} event={}", sequence, event, ex);
        // Dead Letter Queue에 기록
        deadLetterQueue.offer(event);
    }
    
    @Override
    public void handleOnStartException(Throwable ex) { /* startup error */ }
    
    @Override
    public void handleOnShutdownException(Throwable ex) { /* shutdown error */ }
});
```

### 2. 링 버퍼 크기: 백프레셔 없는 주의점

Disruptor에는 기본적으로 **백프레셔 메커니즘이 없습니다**. 생산자가 소비자보다 빠르면 링 버퍼가 가득 차고, `ringBuffer.next()`가 블로킹됩니다(소비자가 처리할 때까지 스핀). 버퍼 크기는 트래픽 버스트를 흡수할 수 있을 만큼 충분히 크게 설정하되, L3 캐시에 들어갈 수 있는 크기가 이상적입니다.

### 3. SINGLE vs MULTI 생산자

`ProducerType.SINGLE`은 CAS 연산이 불필요해 `MULTI`보다 훨씬 빠릅니다. 생산자가 실제로 하나임을 보장할 수 있을 때만 SINGLE을 사용하세요. 여러 스레드에서 SINGLE을 사용하면 데이터 손상이 발생합니다.

### 4. JVM 튜닝

Disruptor의 진가는 GC 최소화에 있습니다. JVM 옵션으로 이를 보완하세요.

```bash
# GC 로깅으로 불필요한 GC 확인
-Xlog:gc*:gc.log

# 이벤트 객체 외 할당 최소화
# 핸들러 내에서 새 객체 생성 금지

# CPU 어피니티 설정 (Linux)
taskset -c 0,1,2 java -jar trading-system.jar
```

### 5. 성능 측정: 올바른 벤치마크

JMH(Java Microbenchmark Harness)를 사용해 JIT 워밍업을 포함한 정확한 성능을 측정하세요. 단순한 System.nanoTime()만으로는 JIT 효과를 반영하지 못합니다.

## 참고 자료
- [LMAX Disruptor GitHub Repository](https://github.com/LMAX-Exchange/disruptor)
- [Apache ZooKeeper GitHub Repository](https://github.com/apache/zookeeper)
