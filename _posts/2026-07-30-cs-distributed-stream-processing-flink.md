---
layout: post
title: "분산 스트림 처리 완전 정복: Apache Flink 아키텍처, 상태 관리, 체크포인팅"
date: 2026-07-30
categories: [cs, computer-science]
tags: [apache-flink, stream-processing, stateful-processing, checkpointing, exactly-once, distributed-systems, datastream-api, watermark]
---

현대 데이터 시스템은 배치(Batch) 처리만으로는 부족하다. 사용자의 클릭 스트림을 실시간 분석하고, 금융 거래의 이상 패턴을 밀리초 단위로 탐지하고, IoT 센서 데이터를 지속적으로 집계해야 한다. 이런 요구를 만족시키는 기술이 **분산 스트림 처리(Distributed Stream Processing)**다. 그리고 이 분야에서 가장 강력한 프레임워크 중 하나가 **Apache Flink**다.

## 배치 처리 vs 스트림 처리

전통적인 **배치 처리(Batch Processing)**는 데이터를 일정 시간 동안 모은 뒤 한꺼번에 처리한다. MapReduce, Apache Spark(배치 모드)가 대표적이다.

```
[데이터 축적] → [처리 시작] → [결과 출력]
 (수 시간 대기)                (처리 완료 후 사용 가능)
```

**스트림 처리(Stream Processing)**는 데이터가 들어오는 즉시, 연속적으로 처리한다. 데이터 파이프라인이 끊임없이 실행된다.

```
이벤트1 → [처리] → 결과1
이벤트2 → [처리] → 결과2
이벤트3 → [처리] → 결과3
(지연 없이 실시간)
```

Flink는 배치와 스트림을 **통합 모델**로 처리한다. "배치는 경계가 있는 스트림"으로 바라보는 것이다.

---

## Apache Flink 아키텍처

Flink의 아키텍처는 크게 두 계층으로 나뉜다.

### 클러스터 구성 요소

```
[Client]
    │  (Job Graph 제출)
    ▼
[JobManager]  ←→  [ResourceManager]
    │                    │
    │  (Task 배포)        │  (TaskManager 관리)
    ▼                    ▼
[TaskManager1]   [TaskManager2]   [TaskManager3]
  ┌─────────┐    ┌─────────┐    ┌─────────┐
  │ Task Slot│    │ Task Slot│    │ Task Slot│
  │ Task Slot│    │ Task Slot│    │ Task Slot│
  └─────────┘    └─────────┘    └─────────┘
```

- **JobManager**: 클러스터의 마스터 노드. 잡의 스케줄링, 체크포인트 조율, 장애 복구를 담당한다.
- **TaskManager**: 워커 노드. 실제 데이터를 처리하는 Task Slot을 가진다.
- **Task Slot**: 병렬 처리의 단위. 한 슬롯은 하나의 서브태스크(파이프라인의 일부)를 실행한다.

### 데이터 플로우 그래프

Flink 프로그램은 **Directed Acyclic Graph(DAG)**로 변환된다. 소스에서 시작하여, 여러 오퍼레이터를 거쳐, 싱크에서 끝난다.

```
[Source: Kafka]
     │
     ▼ (parallelism=4)
[Filter: 조건 필터링]
     │
     ▼ (parallelism=4)
[Map: 변환 처리]
     │
     ▼ (keyBy: userId, parallelism=8)
[Window Aggregation: 집계]
     │
     ▼
[Sink: PostgreSQL / Elasticsearch]
```

---

## DataStream API: 스트림 처리 코드 작성

Flink의 DataStream API로 실시간 이벤트를 처리하는 예제다.

```java
import org.apache.flink.api.common.eventtime.WatermarkStrategy;
import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.api.common.functions.ReduceFunction;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer;

import java.time.Duration;
import java.util.Properties;

public class RealtimeOrderAggregation {

    // 이벤트 POJO
    public static class OrderEvent {
        public String userId;
        public double amount;
        public long timestamp;

        public OrderEvent(String userId, double amount, long timestamp) {
            this.userId = userId;
            this.amount = amount;
            this.timestamp = timestamp;
        }
    }

    // 집계 결과 POJO
    public static class UserTotal {
        public String userId;
        public double totalAmount;
        public int orderCount;

        public UserTotal(String userId, double totalAmount, int orderCount) {
            this.userId = userId;
            this.totalAmount = totalAmount;
            this.orderCount = orderCount;
        }

        @Override
        public String toString() {
            return String.format("User=%s, Total=%.2f, Count=%d", userId, totalAmount, orderCount);
        }
    }

    public static void main(String[] args) throws Exception {
        // 1. 실행 환경 설정
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(4);  // 병렬 처리 수준 설정

        // 체크포인팅 활성화 (5초마다)
        env.enableCheckpointing(5000);

        // 2. Kafka 소스 설정
        Properties kafkaProps = new Properties();
        kafkaProps.setProperty("bootstrap.servers", "localhost:9092");
        kafkaProps.setProperty("group.id", "flink-order-consumer");

        // 3. 이벤트 시간 기반 워터마크 전략
        // 최대 10초의 지연을 허용 (늦게 도착하는 이벤트 처리)
        WatermarkStrategy<OrderEvent> watermarkStrategy = WatermarkStrategy
            .<OrderEvent>forBoundedOutOfOrderness(Duration.ofSeconds(10))
            .withTimestampAssigner((event, recordTimestamp) -> event.timestamp);

        // 4. 데이터 스트림 생성 및 처리 파이프라인 구성
        DataStream<OrderEvent> orderStream = env
            .fromElements(  // 예제에서는 fromElements 사용
                new OrderEvent("user1", 100.0, System.currentTimeMillis()),
                new OrderEvent("user2", 200.0, System.currentTimeMillis()),
                new OrderEvent("user1", 150.0, System.currentTimeMillis()),
                new OrderEvent("user3", 300.0, System.currentTimeMillis()),
                new OrderEvent("user2", 50.0, System.currentTimeMillis())
            )
            .assignTimestampsAndWatermarks(watermarkStrategy);

        // 5. keyBy로 userId 기준 파티셔닝 → 1분 텀블링 윈도우 집계
        DataStream<UserTotal> result = orderStream
            .filter(event -> event.amount > 0)          // 유효한 주문만
            .keyBy(event -> event.userId)               // userId 기준 파티셔닝
            .window(TumblingEventTimeWindows.of(Time.minutes(1)))  // 1분 윈도우
            .reduce(new ReduceFunction<OrderEvent>() {  // 집계 함수
                @Override
                public OrderEvent reduce(OrderEvent a, OrderEvent b) {
                    return new OrderEvent(a.userId, a.amount + b.amount,
                                         Math.max(a.timestamp, b.timestamp));
                }
            })
            .map(event -> new UserTotal(event.userId, event.amount, 1));

        // 6. 결과 출력
        result.print();

        // 7. 실행 시작
        env.execute("Realtime Order Aggregation");
    }
}
```

---

## 상태 관리 (Stateful Processing)

Flink의 가장 강력한 기능은 **상태(State)**다. 이전 이벤트를 기억하고, 현재 이벤트와 결합하여 처리할 수 있다.

### Keyed State vs Operator State

- **Keyed State**: `keyBy` 이후 각 키마다 독립적인 상태를 유지한다. 가장 일반적이다.
- **Operator State**: 병렬 오퍼레이터 인스턴스 전체에 공유되는 상태.

```java
import org.apache.flink.api.common.state.ValueState;
import org.apache.flink.api.common.state.ValueStateDescriptor;
import org.apache.flink.configuration.Configuration;
import org.apache.flink.streaming.api.functions.KeyedProcessFunction;
import org.apache.flink.util.Collector;

// 사용자별 연속 실패 로그인 횟수를 추적하는 KeyedProcessFunction
public class LoginFailureDetector extends KeyedProcessFunction<String, LoginEvent, Alert> {

    // 키(userId)마다 별도로 저장되는 상태
    private ValueState<Integer> failureCount;
    private ValueState<Long> firstFailureTime;

    @Override
    public void open(Configuration parameters) throws Exception {
        // 상태 정의: Integer 값을 저장하는 ValueState
        ValueStateDescriptor<Integer> countDescriptor =
            new ValueStateDescriptor<>("failure-count", Integer.class, 0);
        failureCount = getRuntimeContext().getState(countDescriptor);

        ValueStateDescriptor<Long> timeDescriptor =
            new ValueStateDescriptor<>("first-failure-time", Long.class, 0L);
        firstFailureTime = getRuntimeContext().getState(timeDescriptor);
    }

    @Override
    public void processElement(LoginEvent event, Context ctx, Collector<Alert> out)
            throws Exception {

        if (event.isFailure()) {
            int count = failureCount.value() + 1;
            failureCount.update(count);

            if (count == 1) {
                firstFailureTime.update(event.timestamp);
                // 60초 타이머 등록: 60초 내에 3회 실패하면 알림
                ctx.timerService().registerEventTimeTimer(event.timestamp + 60_000L);
            }

            if (count >= 3) {
                long firstTime = firstFailureTime.value();
                if (event.timestamp - firstTime < 60_000L) {
                    // 60초 내 3회 실패 → 알림 발생
                    out.collect(new Alert(event.userId,
                                         "60초 내 3회 연속 로그인 실패 감지", event.timestamp));
                    // 상태 초기화
                    failureCount.clear();
                    firstFailureTime.clear();
                }
            }
        } else {
            // 성공 로그인 시 상태 초기화
            failureCount.clear();
            firstFailureTime.clear();
        }
    }

    @Override
    public void onTimer(long timestamp, OnTimerContext ctx, Collector<Alert> out) throws Exception {
        // 타이머 만료: 60초 경과 후 상태 초기화
        failureCount.clear();
        firstFailureTime.clear();
    }
}
```

이 코드에서 핵심은 `failureCount`와 `firstFailureTime`이 **키(userId)별로 독립적으로 관리**된다는 것이다. userId "alice"의 실패 횟수와 "bob"의 실패 횟수는 서로 격리된다. 클러스터가 재시작되어도 체크포인트에서 상태가 복구된다.

---

## 체크포인팅과 정확히-한-번 처리 보장

Flink의 장애 복구 메커니즘은 **Chandy-Lamport 분산 스냅샷** 알고리즘에 기반한다.

### Barrier 기반 체크포인팅

```
Kafka Source        Operator A         Operator B
    │                   │                   │
    │  Event1           │                   │
    │──────────────────►│                   │
    │  Event2           │  처리된 Event1    │
    │──────────────────►│──────────────────►│
    │  [BARRIER n]      │                   │
    │──────────────────►│                   │
    │                   │  [BARRIER n]      │
    │                   │──────────────────►│
    │                   │  (A 상태 스냅샷)   │  (B 상태 스냅샷)
    │  Event3           │                   │
    │──────────────────►│                   │
```

1. **JobManager가 체크포인트 시작 신호를 보낸다**
2. **소스 노드가 Barrier를 이벤트 스트림에 삽입한다**
3. **Barrier가 오퍼레이터를 통과할 때 해당 오퍼레이터의 상태를 스냅샷으로 저장한다**
4. **모든 오퍼레이터가 Barrier를 처리하면 체크포인트 완료**

### Exactly-Once Semantics

Flink는 세 가지 수준의 처리 보장을 제공한다:

| 수준 | 설명 | 성능 |
|------|------|------|
| `AT_MOST_ONCE` | 장애 시 일부 이벤트 유실 가능 | 가장 빠름 |
| `AT_LEAST_ONCE` | 장애 시 이벤트 중복 처리 가능 | 중간 |
| `EXACTLY_ONCE` | 정확히 한 번 처리 보장 | 느림 (2PC 사용) |

```java
// 체크포인팅 설정
CheckpointConfig config = env.getCheckpointConfig();
config.setCheckpointInterval(5000);  // 5초마다 체크포인트
config.setMinPauseBetweenCheckpoints(1000);  // 체크포인트 사이 최소 1초
config.setCheckpointTimeout(60000);  // 체크포인트 타임아웃 60초
config.setMaxConcurrentCheckpoints(1);  // 동시 체크포인트 최대 1개

// Exactly-Once 시맨틱 설정
config.setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);

// 체크포인트 저장소 (HDFS, S3 등)
env.getCheckpointConfig().setCheckpointStorage("s3://my-bucket/flink-checkpoints");
```

---

## 이벤트 시간 vs 처리 시간: Watermark

스트림 처리에서 가장 어려운 문제 중 하나는 **늦게 도착하는 이벤트(Late Arriving Events)**다.

```
실제 발생 순서:   이벤트1(t=10) → 이벤트2(t=12) → 이벤트3(t=9, 늦게 도착)
Flink 수신 순서:  이벤트1(t=10) → 이벤트2(t=12) → 이벤트3(t=9)
```

이벤트3은 t=9에 발생했지만 늦게 도착했다. 만약 t=10~12에 대한 윈도우를 이미 닫아버렸다면?

Flink는 **워터마크(Watermark)**로 이 문제를 해결한다. 워터마크는 "이 시점 이후로는 t 이전의 이벤트가 더 이상 오지 않는다"는 진행 표시자다.

```java
// BoundedOutOfOrderness: 최대 5초 지연 허용
WatermarkStrategy<OrderEvent> strategy = WatermarkStrategy
    .<OrderEvent>forBoundedOutOfOrderness(Duration.ofSeconds(5))
    .withTimestampAssigner((event, ts) -> event.eventTime);

// 워터마크가 w일 때, [w-window_size, w] 범위의 윈도우를 닫는다
// 최대 5초 지연을 허용하므로, 5초 이내에 도착한 이벤트는 올바른 윈도우에 들어간다
```

---

## 상태 백엔드

Flink의 상태는 세 가지 백엔드에 저장할 수 있다:

| 백엔드 | 저장 위치 | 특징 |
|--------|-----------|------|
| `HashMapStateBackend` | JVM 힙 메모리 | 빠르지만 GC 부담, 대용량 부적합 |
| `EmbeddedRocksDBStateBackend` | 디스크 + 로컬 SSD | 대용량 상태 처리, 느리지만 확장성 |
| `EmbeddedRocksDBStateBackend` (incremental) | 디스크 + 증분 스냅샷 | 체크포인팅 속도 개선 |

```java
// RocksDB 상태 백엔드 설정 (대용량 상태에 권장)
env.setStateBackend(new EmbeddedRocksDBStateBackend(true)); // true = 증분 체크포인팅
```

---

## 실전 패턴: 슬라이딩 윈도우로 실시간 이상 감지

```java
// 1분 슬라이딩 윈도우로 10초마다 평균 주문 금액 계산
DataStream<Alert> anomalyAlerts = orderStream
    .keyBy(event -> event.merchantId)
    .window(SlidingEventTimeWindows.of(Time.minutes(1), Time.seconds(10)))
    .aggregate(new AggregateFunction<OrderEvent, Accumulator, Double>() {
        @Override
        public Accumulator createAccumulator() {
            return new Accumulator(0, 0.0);
        }

        @Override
        public Accumulator add(OrderEvent event, Accumulator acc) {
            return new Accumulator(acc.count + 1, acc.sum + event.amount);
        }

        @Override
        public Double getResult(Accumulator acc) {
            return acc.count == 0 ? 0.0 : acc.sum / acc.count;
        }

        @Override
        public Accumulator merge(Accumulator a, Accumulator b) {
            return new Accumulator(a.count + b.count, a.sum + b.sum);
        }
    })
    .filter(avgAmount -> avgAmount > 10000.0)  // 평균 1만원 초과 시 알림
    .map(avgAmount -> new Alert("고액 주문 이상 감지: 평균 " + avgAmount));
```

---

## Flink vs Kafka Streams vs Spark Streaming

| 특징 | Apache Flink | Kafka Streams | Spark Streaming |
|------|-------------|---------------|----------------|
| 처리 모델 | 네이티브 스트림 | 네이티브 스트림 | 마이크로 배치 |
| 상태 관리 | 강력 (RocksDB) | Kafka 기반 | 제한적 |
| 정확히-한-번 | 지원 | 지원 (Kafka EOS) | 지원 (제한적) |
| 지연 시간 | 밀리초 | 밀리초 | 수백 밀리초~수 초 |
| 복잡 이벤트 처리 | CEP 라이브러리 | 제한적 | 제한적 |
| 운영 복잡도 | 높음 | 낮음 (라이브러리) | 중간 |
| 배치+스트림 통합 | 완전 통합 | 스트림 전용 | 배치 중심 |

**Kafka Streams**는 클러스터 없이 라이브러리로 임베딩되므로 운영이 쉽다. Kafka 토픽과의 통합이 최우선이라면 좋은 선택이다. **Apache Flink**는 복잡한 상태 처리, 낮은 지연, 정확히-한-번 보장이 필요한 금융·결제·실시간 분석에 적합하다.

---

## 주의사항과 팁

1. **상태 크기 모니터링**: 상태가 무한정 커질 수 있다. `StateTtlConfig`로 상태에 TTL을 설정하여 오래된 상태가 자동 삭제되도록 하자.
2. **체크포인트 저장소 선택**: 프로덕션에서는 반드시 S3, HDFS, GCS 같은 분산 파일시스템을 체크포인트 저장소로 사용해야 한다. 로컬 파일시스템은 단일 장애점이 된다.
3. **백프레셔(Backpressure)**: 다운스트림 오퍼레이터가 느리면 업스트림이 자동으로 속도를 줄인다. Flink UI의 백프레셔 지표를 모니터링하여 병목을 찾자.
4. **Savepoint vs Checkpoint**: 체크포인트는 자동 장애 복구용, 세이브포인트는 수동 마이그레이션/업그레이드용이다. 잡 로직을 변경할 때는 세이브포인트를 만들고 재시작하자.
5. **parallelism 설계**: `keyBy` 이후 병렬성이 증가하면 네트워크 셔플(데이터 재분배) 비용이 커진다. 파티셔닝 키와 병렬성을 함께 설계하자.

---

## 참고 자료
- [Apache Flink - Stateful Stream Processing](https://nightlies.apache.org/flink/flink-docs-stable/docs/concepts/stateful-stream-processing/)
- [Apache Flink - Checkpointing](https://nightlies.apache.org/flink/flink-docs-stable/docs/dev/datastream/fault-tolerance/checkpointing/)
- [Apache Flink 공식 사이트](https://flink.apache.org/)
- [Apache Flink - Wikipedia](https://en.wikipedia.org/wiki/Apache_Flink)
