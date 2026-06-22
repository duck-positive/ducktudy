---
layout: post
title: "Kafka 아키텍처 심화와 이벤트 소싱 패턴: 불변 로그로 만드는 견고한 시스템"
date: 2026-06-22
categories: [cs, computer-science]
tags: [kafka, event-sourcing, message-queue, distributed-systems, CQRS, architecture, event-driven]
---

"데이터베이스 레코드를 직접 업데이트하지 말고, 모든 변화를 이벤트로 기록하라." 이벤트 소싱(Event Sourcing)은 이 단순한 아이디어로 감사 추적, 시간 여행 디버깅, 시스템 복원을 동시에 해결한다. Apache Kafka는 이 패턴의 인프라적 구현체로, 분산 환경에서 초당 수백만 건의 이벤트를 처리한다.

## Kafka 아키텍처 심화

### 핵심 구성 요소

```
Producer → [Topic/Partition] → Consumer Group
               ↑
           Broker Cluster
           (KRaft/ZooKeeper)
```

**Topic**: 이벤트의 논리적 카테고리. `orders`, `user-events` 같은 이름을 가진다.

**Partition**: 토픽을 물리적으로 분할한 단위. 각 파티션은 **append-only 로그**다. 파티션 수가 병렬 처리 능력을 결정한다.

**Broker**: Kafka 서버 인스턴스. 여러 브로커가 클러스터를 이루고, 각 파티션의 리더/팔로워를 분산 배치한다.

**Consumer Group**: 동일한 `group.id`를 가진 컨슈머들의 집합. 하나의 파티션은 그룹 내 하나의 컨슈머에게만 할당된다.

### 오프셋(Offset) 메커니즘

Kafka의 가장 중요한 특징은 **메시지를 소비해도 삭제하지 않는다**는 것이다. 각 컨슈머 그룹은 파티션별로 "어디까지 읽었는지(offset)"를 별도로 관리한다.

```
Partition 0: [msg0] [msg1] [msg2] [msg3] [msg4] [msg5]
                                    ↑                ↑
                            GroupA offset=3    GroupB offset=6

- GroupA는 msg3부터 다시 읽을 수 있음
- GroupB는 최신 메시지까지 읽음
- msg0~msg2는 retention 기간(기본 7일) 동안 보존
```

이 구조 덕분에:
1. **재처리(Replay)**: 오프셋을 0으로 되돌려 전체 이벤트를 재처리할 수 있다
2. **다중 소비자**: 여러 서비스가 같은 토픽을 독립적으로 소비한다
3. **장애 복구**: 컨슈머가 죽어도 마지막 커밋 오프셋부터 재시작한다

### KRaft: ZooKeeper 없는 Kafka

Kafka 4.0부터 ZooKeeper 의존성이 완전히 제거되고 **KRaft(Kafka Raft Metadata)** 모드가 기본이 됐다. 메타데이터(토픽 설정, 파티션 리더 정보)를 Kafka 자체 내부 토픽에 저장하고 Raft 합의 알고리즘으로 관리한다.

```
Before (ZooKeeper):              After (KRaft):
Kafka Brokers ←→ ZooKeeper       Kafka Brokers (Controller 내장)
운영 복잡도: 높음                  운영 복잡도: 낮음
별도 앙상블 필요                   단일 프로세스로 통합
```

---

## 이벤트 소싱 패턴 (Event Sourcing)

### 전통적 상태 저장 vs 이벤트 소싱

```
[전통적 방식 - State Storage]
주문 테이블: { id: 1, status: "delivered", amount: 50000 }
→ UPDATE orders SET status='delivered' WHERE id=1;
→ 이전 상태(주문 생성, 결제, 발송)는 사라짐

[이벤트 소싱 - Event Storage]
이벤트 로그:
  t=0: OrderPlaced   { orderId: 1, amount: 50000 }
  t=1: PaymentDone   { orderId: 1, method: "card" }
  t=2: OrderShipped  { orderId: 1, trackingId: "K123" }
  t=3: OrderDelivered{ orderId: 1 }
→ 현재 상태는 이벤트를 순서대로 적용(fold)해서 도출
```

이벤트 소싱에서 **이벤트 스트림이 유일한 진실의 원천(Single Source of Truth)**이다.

### 핵심 장점

1. **완전한 감사 로그**: 누가 언제 무엇을 했는지 모든 이력이 남는다 (금융/의료/규제 산업 필수)
2. **시간 여행 디버깅**: 특정 시점의 시스템 상태를 이벤트를 재생해 재현할 수 있다
3. **이벤트 드리븐 아키텍처**: 이벤트 발행으로 다른 서비스를 느슨하게 연동한다
4. **읽기 모델 재구성**: 집계 방식(쿼리 모델)이 바뀌어도 이벤트를 재처리해 새 뷰를 만들 수 있다

---

## 실제 구현 예제

### 예제 1: Python — Kafka Producer/Consumer로 이벤트 소싱

```python
from kafka import KafkaProducer, KafkaConsumer
from dataclasses import dataclass, asdict
from datetime import datetime
import json
import uuid

# 이벤트 타입 정의
@dataclass
class Event:
    event_id: str
    event_type: str
    aggregate_id: str
    timestamp: str
    payload: dict

def create_event(event_type: str, aggregate_id: str, payload: dict) -> Event:
    return Event(
        event_id=str(uuid.uuid4()),
        event_type=event_type,
        aggregate_id=aggregate_id,
        timestamp=datetime.utcnow().isoformat(),
        payload=payload,
    )

# --- Producer: 이벤트 발행 ---
class OrderEventProducer:
    def __init__(self, bootstrap_servers: str):
        self.producer = KafkaProducer(
            bootstrap_servers=bootstrap_servers,
            value_serializer=lambda v: json.dumps(v).encode('utf-8'),
            key_serializer=lambda k: k.encode('utf-8'),
        )

    def publish(self, event: Event):
        # aggregate_id를 키로 사용 → 같은 주문의 이벤트는 같은 파티션에 기록
        self.producer.send(
            topic='orders',
            key=event.aggregate_id,
            value=asdict(event),
        )
        self.producer.flush()
        print(f"Published: {event.event_type} for order {event.aggregate_id}")


# --- Consumer: 이벤트 소비 & 상태 재구성 ---
class OrderProjection:
    """이벤트를 소비해 현재 주문 상태를 메모리에 유지"""

    def __init__(self):
        self.orders: dict[str, dict] = {}

    def apply(self, event: Event):
        aid = event.aggregate_id
        match event.event_type:
            case 'OrderPlaced':
                self.orders[aid] = {
                    'id': aid,
                    'status': 'placed',
                    'amount': event.payload['amount'],
                }
            case 'PaymentDone':
                if aid in self.orders:
                    self.orders[aid]['status'] = 'paid'
            case 'OrderShipped':
                if aid in self.orders:
                    self.orders[aid]['status'] = 'shipped'
                    self.orders[aid]['tracking'] = event.payload.get('trackingId')
            case 'OrderDelivered':
                if aid in self.orders:
                    self.orders[aid]['status'] = 'delivered'

    def run_consumer(self, bootstrap_servers: str):
        consumer = KafkaConsumer(
            'orders',
            bootstrap_servers=bootstrap_servers,
            group_id='order-projection',
            value_deserializer=lambda v: json.loads(v.decode('utf-8')),
            auto_offset_reset='earliest',  # 처음부터 재생
        )
        for msg in consumer:
            data = msg.value
            event = Event(**data)
            self.apply(event)
            print(f"[offset {msg.offset}] Applied: {event.event_type}")


# --- 사용 시나리오 ---
if __name__ == '__main__':
    producer = OrderEventProducer('localhost:9092')
    order_id = str(uuid.uuid4())

    producer.publish(create_event('OrderPlaced',   order_id, {'amount': 59000}))
    producer.publish(create_event('PaymentDone',   order_id, {'method': 'card'}))
    producer.publish(create_event('OrderShipped',  order_id, {'trackingId': 'K456'}))
    producer.publish(create_event('OrderDelivered',order_id, {}))
```

### 예제 2: CQRS 패턴과 결합 — Kotlin

CQRS(Command Query Responsibility Segregation)는 쓰기(Command)와 읽기(Query) 모델을 분리하는 패턴이다. 이벤트 소싱과 결합하면 각 모델을 독립적으로 최적화할 수 있다.

```kotlin
import org.apache.kafka.clients.producer.KafkaProducer
import org.apache.kafka.clients.producer.ProducerRecord
import com.fasterxml.jackson.module.kotlin.jacksonObjectMapper
import java.util.Properties

data class DomainEvent(
    val eventId: String,
    val eventType: String,
    val aggregateId: String,
    val timestamp: Long,
    val payload: Map<String, Any>
)

// --- Command Side: 명령 처리 후 이벤트 저장 ---
class OrderCommandHandler(private val eventStore: EventStore) {

    fun placeOrder(orderId: String, amount: Int): DomainEvent {
        // 비즈니스 로직 검증...
        val event = DomainEvent(
            eventId = java.util.UUID.randomUUID().toString(),
            eventType = "OrderPlaced",
            aggregateId = orderId,
            timestamp = System.currentTimeMillis(),
            payload = mapOf("amount" to amount)
        )
        eventStore.append(event)
        return event
    }

    fun shipOrder(orderId: String, trackingId: String): DomainEvent {
        val event = DomainEvent(
            eventId = java.util.UUID.randomUUID().toString(),
            eventType = "OrderShipped",
            aggregateId = orderId,
            timestamp = System.currentTimeMillis(),
            payload = mapOf("trackingId" to trackingId)
        )
        eventStore.append(event)
        return event
    }
}

// --- Event Store: Kafka 토픽에 이벤트 추가 ---
class EventStore(bootstrapServers: String) {
    private val mapper = jacksonObjectMapper()
    private val producer: KafkaProducer<String, String>

    init {
        val props = Properties().apply {
            put("bootstrap.servers", bootstrapServers)
            put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer")
            put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer")
            // 정확히 1번 전달 보장 (Exactly-Once Semantics)
            put("enable.idempotence", "true")
            put("acks", "all")
        }
        producer = KafkaProducer(props)
    }

    fun append(event: DomainEvent) {
        val record = ProducerRecord(
            "order-events",
            event.aggregateId,                // 파티션 키
            mapper.writeValueAsString(event)   // 직렬화
        )
        producer.send(record).get()  // 동기 전송으로 at-least-once 보장
    }
}

// --- Query Side: 이벤트를 소비해 읽기 모델 구성 ---
data class OrderReadModel(
    val id: String,
    var status: String,
    var amount: Int = 0,
    var trackingId: String? = null
)

class OrderReadModelProjector {
    val orders = mutableMapOf<String, OrderReadModel>()

    fun handle(event: DomainEvent) {
        when (event.eventType) {
            "OrderPlaced" -> orders[event.aggregateId] = OrderReadModel(
                id = event.aggregateId,
                status = "placed",
                amount = event.payload["amount"] as Int
            )
            "OrderShipped" -> orders[event.aggregateId]?.apply {
                status = "shipped"
                trackingId = event.payload["trackingId"] as String
            }
            "OrderDelivered" -> orders[event.aggregateId]?.status = "delivered"
        }
    }
}
```

---

## 스냅샷(Snapshot) 전략

이벤트가 수백만 건 쌓이면 매번 전체를 재생하는 건 비효율적이다. **스냅샷**은 특정 시점의 집계 상태를 별도 저장소에 저장해, 그 이후 이벤트만 재생하면 되도록 한다.

```
이벤트 스트림:  [e1][e2]...[e999][e1000][e1001]...[e2000]
                              ↑                     ↑
                       Snapshot@1000          Snapshot@2000

현재 상태 = load(Snapshot@2000) + replay(e2001..eN)
```

- 스냅샷 빈도: 100~1000 이벤트마다 한 번이 일반적
- 저장 위치: Redis, S3, PostgreSQL (이벤트 스토어와 별도)

---

## 주의사항 및 팁

### 1. 이벤트 스키마 불변성

이벤트는 한번 발행되면 절대 수정/삭제하지 않는다. 스키마가 바뀌면 새 버전의 이벤트 타입을 추가하고, 프로젝터(읽기 모델 구성기)에서 v1/v2를 모두 처리하는 Upcaster 패턴을 쓴다.

### 2. Exactly-Once 처리

Kafka의 기본 보장은 **At-Least-Once** (중복 가능). 정확히 한 번 처리가 필요하면:
- Producer: `enable.idempotence=true` + `transactional.id` 설정
- Consumer: 컨슈머 사이드 멱등성(Idempotency) 구현 (이벤트 ID로 중복 체크)

### 3. 파티션 수는 신중하게 설정

파티션 수는 늘리기는 쉽지만 줄이기는 어렵다(토픽 재생성 필요). 초기에는 `컨슈머 인스턴스 수 × 2` 수준으로 설정하고, 처리량을 모니터링해 조정한다.

### 4. Consumer Lag 모니터링

컨슈머가 처리하는 속도보다 프로듀서가 빠르면 **Consumer Lag**이 쌓인다. `kafka-consumer-groups.sh --describe` 또는 Prometheus + Grafana로 실시간 감시가 필수다.

### 5. 이벤트 소싱이 적합하지 않은 경우

- 단순 CRUD 애플리케이션: 오버엔지니어링
- 이벤트 히스토리가 필요 없는 캐시성 데이터
- 쿼리 패턴이 매우 단순한 경우

---

## 마무리

Kafka와 이벤트 소싱은 독립적으로도 쓸 수 있지만, 함께 쓸 때 시너지가 폭발한다. Kafka의 append-only 로그 구조는 이벤트 소싱의 "변경 불가한 이벤트 스트림" 철학과 완벽히 일치한다. 스냅샷 전략과 CQRS를 더하면, 초대형 트래픽을 감당하면서도 전체 이력을 보존하는 시스템을 만들 수 있다. 넷플릭스, 우버, 링크드인이 이 아키텍처를 실제 운영하고 있다.

## 참고 자료
- [Apache Kafka Official Documentation](https://kafka.apache.org/documentation/)
- [Event Sourcing Patterns with Kafka - Conduktor](https://www.conduktor.io/glossary/event-sourcing-patterns-with-kafka)
- [Event-Driven Architecture with Apache Kafka - Redpanda](https://www.redpanda.com/guides/kafka-use-cases-event-driven-architecture)
- [Event Sourcing with Kafka - Tinybird Blog](https://www.tinybird.co/blog/event-sourcing-with-kafka)
