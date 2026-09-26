---
layout: post
title: "CDC(Change Data Capture) 완전 정복: Debezium으로 데이터베이스 변경을 실시간 스트리밍하는 법"
date: 2026-09-26
categories: [cs, computer-science]
tags: [cdc, change-data-capture, debezium, kafka, database, event-streaming, mysql, postgresql]
---

마이크로서비스 환경에서 가장 어려운 문제 중 하나는 여러 서비스에 걸친 데이터 동기화다. 한 서비스의 데이터베이스가 변경되면 캐시를 무효화하거나 검색 인덱스를 갱신하거나 다른 서비스에 알림을 보내야 한다. 이를 애플리케이션 코드에서 직접 처리하면 복잡성이 폭발한다. **CDC(Change Data Capture)**는 이 문제를 데이터베이스 레벨에서 우아하게 해결한다.

## 1. CDC란 무엇인가

CDC(Change Data Capture)는 데이터베이스에서 발생하는 모든 변경(INSERT, UPDATE, DELETE)을 감지하고 이를 다른 시스템으로 스트리밍하는 기술이다. 핵심은 데이터 변경을 **이벤트 스트림**으로 추출하여, 이를 구독하는 시스템들이 실시간으로 반응할 수 있게 하는 것이다.

CDC의 주요 사용 사례는 다음과 같다.

- **캐시 무효화**: DB 레코드가 변경되면 Redis 캐시를 즉시 제거
- **검색 인덱스 동기화**: MySQL 변경을 Elasticsearch에 실시간 반영
- **감사 로그(Audit Log)**: 모든 데이터 변경 이력 추적
- **마이크로서비스 이벤트 발행**: 주문 상태 변경을 다른 서비스에 알림
- **데이터 웨어하우스 ETL**: 변경된 레코드만 증분 로드

## 2. CDC의 두 가지 방식

### 방식 1: 폴링 기반(Polling-Based) CDC

가장 단순한 방식으로, 주기적으로 `updated_at` 컬럼이나 시퀀스 번호를 확인하여 변경된 레코드를 쿼리한다.

```sql
-- 마지막 처리 시각 이후에 변경된 레코드를 폴링
SELECT * FROM orders
WHERE updated_at > '2026-09-26 10:00:00'
ORDER BY updated_at;
```

**단점**: DELETE 이벤트를 감지하기 어렵고, 폴링 주기 동안의 지연이 있으며, 데이터베이스에 추가적인 쿼리 부하를 준다.

### 방식 2: 로그 기반(Log-Based) CDC

데이터베이스의 **트랜잭션 로그**를 직접 읽는 방식이다. MySQL의 바이너리 로그(binlog), PostgreSQL의 WAL(Write-Ahead Log), Oracle의 Redo Log 등이 해당한다. 모든 변경 연산이 로그에 기록되므로 INSERT/UPDATE/DELETE 모두 캡처 가능하고, 애플리케이션 성능에 거의 영향을 주지 않는다. **Debezium**은 로그 기반 CDC를 구현한 대표적인 오픈소스 플랫폼이다.

## 3. Debezium 아키텍처

Debezium은 Apache Kafka Connect 기반의 소스 커넥터 집합이다. 아키텍처는 다음과 같다.

```
┌──────────────┐    트랜잭션 로그     ┌──────────────────────┐
│   MySQL /    │ ─────────────────→ │  Debezium Connector  │
│ PostgreSQL / │                    │  (Kafka Connect)     │
│    Oracle    │                    └──────────┬───────────┘
└──────────────┘                               │ CDC 이벤트
                                               ↓
                                    ┌──────────────────────┐
                                    │    Apache Kafka      │
                                    │  (토픽: db.table명)  │
                                    └──────────┬───────────┘
                          ┌──────────────────┬─┴──────────────────┐
                          ↓                  ↓                    ↓
                   ┌────────────┐  ┌──────────────┐  ┌──────────────────┐
                   │   Redis    │  │Elasticsearch │  │  Analytics DB    │
                   │ 캐시 무효화 │  │ 인덱스 동기화 │  │ 데이터 웨어하우스  │
                   └────────────┘  └──────────────┘  └──────────────────┘
```

Debezium 커넥터는 데이터베이스의 트랜잭션 로그를 구독하고, 각 변경 이벤트를 JSON 또는 Avro 형식으로 직렬화하여 Kafka 토픽에 발행한다. 토픽 이름은 기본적으로 `{서버명}.{데이터베이스}.{테이블명}` 형식이다.

## 4. 실제 구현 예제

### 예제 1: Debezium MySQL 커넥터 설정

Kafka Connect REST API를 통해 Debezium MySQL 커넥터를 등록하는 JSON 설정이다.

```json
{
  "name": "mysql-orders-connector",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "tasks.max": "1",
    "database.hostname": "mysql",
    "database.port": "3306",
    "database.user": "debezium",
    "database.password": "dbz_password",
    "database.server.id": "184054",
    "topic.prefix": "shop",
    "database.include.list": "ecommerce",
    "table.include.list": "ecommerce.orders,ecommerce.products",
    "schema.history.internal.kafka.bootstrap.servers": "kafka:9092",
    "schema.history.internal.kafka.topic": "schema-changes.ecommerce",
    "include.schema.changes": "true",
    "decimal.handling.mode": "string",
    "snapshot.mode": "initial",
    "heartbeat.interval.ms": "10000",
    "transforms": "unwrap",
    "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
    "transforms.unwrap.drop.tombstones": "false",
    "transforms.unwrap.delete.handling.mode": "rewrite"
  }
}
```

커넥터를 등록하면 Debezium은 먼저 기존 데이터를 **스냅샷**하여 Kafka에 발행하고(`snapshot.mode: initial`), 이후 binlog를 스트리밍한다. `ExtractNewRecordState` 트랜스폼은 Debezium의 복잡한 이벤트 봉투(envelope)에서 실제 레코드 데이터만 추출한다.

Debezium이 생성하는 원본 CDC 이벤트 구조는 다음과 같다.

```json
{
  "op": "u",
  "ts_ms": 1727340000000,
  "before": {
    "id": 1001,
    "status": "PENDING",
    "amount": 5000
  },
  "after": {
    "id": 1001,
    "status": "SHIPPED",
    "amount": 5000
  },
  "source": {
    "db": "ecommerce",
    "table": "orders",
    "server_id": 184054,
    "binlog_file": "mysql-bin.000003",
    "binlog_pos": 154
  }
}
```

`op` 필드는 `c`(create), `u`(update), `d`(delete), `r`(read/snapshot)을 나타낸다.

### 예제 2: Python Kafka 컨슈머로 CDC 이벤트 처리

```python
import json
from kafka import KafkaConsumer
import redis

# Redis 클라이언트 (캐시 무효화 용)
cache = redis.Redis(host='localhost', port=6379, db=0)

# Kafka 컨슈머 설정
consumer = KafkaConsumer(
    'shop.ecommerce.orders',
    bootstrap_servers=['kafka:9092'],
    group_id='order-cache-invalidator',
    auto_offset_reset='earliest',
    value_deserializer=lambda m: json.loads(m.decode('utf-8')),
    key_deserializer=lambda m: m.decode('utf-8') if m else None
)

def handle_order_created(record: dict):
    order_id = record['id']
    print(f"[CREATE] 새 주문 #{order_id} 생성됨")
    cache.delete(f"user:{record['user_id']}:orders")

def handle_order_updated(before: dict, after: dict):
    order_id = after['id']
    old_status = before.get('status')
    new_status = after.get('status')
    if old_status != new_status:
        print(f"[UPDATE] 주문 #{order_id} 상태 변경: {old_status} → {new_status}")
        # 캐시 무효화
        cache.delete(f"order:{order_id}")
        cache.delete(f"user:{after['user_id']}:orders")
        # 상태가 SHIPPED로 변경되면 배송 알림 서비스 트리거
        if new_status == 'SHIPPED':
            print(f"  → 배송 알림 발행 (주문 #{order_id})")

def handle_order_deleted(record: dict):
    order_id = record['id']
    print(f"[DELETE] 주문 #{order_id} 삭제됨")
    cache.delete(f"order:{order_id}")

print("CDC 이벤트 컨슈머 시작...")
for message in consumer:
    event = message.value

    # ExtractNewRecordState 적용 후 단순화된 이벤트
    # 원본 포맷 처리 시
    op = event.get('op')

    if op == 'c':
        handle_order_created(event.get('after', {}))
    elif op == 'u':
        handle_order_updated(event.get('before', {}), event.get('after', {}))
    elif op == 'd':
        handle_order_deleted(event.get('before', {}))
    elif op == 'r':
        print(f"[SNAPSHOT] 초기 데이터 로드: 주문 #{event.get('after', {}).get('id')}")
```

## 5. 주의사항 및 고급 팁

### 스냅샷과 스트리밍 일관성

Debezium이 처음 시작될 때 기존 데이터를 스냅샷하는 동안 새 변경도 발생한다. Debezium은 스냅샷 완료 후 binlog를 특정 포지션부터 이어 읽어 일관성을 보장한다. 이 과정에서 Kafka 토픽에 중복 이벤트가 발생할 수 있으므로 컨슈머는 **멱등성(Idempotency)**을 보장해야 한다.

### at-least-once 보장

Debezium은 기본적으로 **at-least-once** 전달을 보장한다. 즉, 장애 복구 시 동일 이벤트가 중복 발행될 수 있다. 컨슈머에서 이벤트의 `binlog_pos`나 레코드의 기본 키를 기반으로 중복 처리를 방지해야 한다.

### Outbox 패턴과의 결합

CDC를 애플리케이션 이벤트 발행에 사용할 때는 **Transactional Outbox 패턴**이 효과적이다. 비즈니스 로직과 이벤트 레코드 삽입을 하나의 트랜잭션으로 묶고, Debezium이 Outbox 테이블을 읽어 Kafka에 발행한다. 이렇게 하면 "이중 쓰기(Dual Write)" 문제를 해결하고 트랜잭션 보장을 얻을 수 있다.

### 스키마 변경 처리

테이블 컬럼이 추가/삭제될 때 Debezium은 스키마 변경을 감지하고 Kafka Schema Registry(Avro 사용 시)와 함께 하위 호환성을 관리한다. 컨슈머는 스키마 버전을 주의 깊게 관리해야 한다.

### Debezium Server와 임베디드 모드

Kafka Connect 없이 Debezium을 사용하고 싶다면 **Debezium Server**를 사용할 수 있다. Amazon Kinesis, Google Pub/Sub, Apache Pulsar 등으로 직접 이벤트를 전달한다. 소규모 환경에서는 애플리케이션에 **임베디드 엔진**으로 통합할 수도 있다.

## 참고 자료

- [Debezium 공식 문서](https://debezium.io/documentation/reference/stable/index.html)
- [Debezium Features Guide](https://debezium.io/documentation/reference/stable/features.html)
- [Debezium FAQ](https://debezium.io/documentation/faq/)
- [Apache Kafka Connect 문서](https://kafka.apache.org/documentation/#connect)
