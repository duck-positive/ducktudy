---
layout: post
title: "AMQP 프로토콜과 RabbitMQ 아키텍처 완전 정복: Exchange·Queue·Binding부터 Quorum Queue까지"
date: 2026-09-28
categories: [cs, computer-science]
tags: [amqp, rabbitmq, message-broker, exchange, queue, routing, distributed-systems, messaging]
---

현대 분산 시스템에서 서비스 간 비동기 통신을 구현할 때 가장 많이 사용되는 도구 중 하나가 **메시지 브로커(Message Broker)**다. Kafka가 고처리량 스트리밍에 특화되어 있다면, RabbitMQ는 **복잡한 라우팅 규칙**, **요청-응답 패턴**, **낮은 지연 시간의 작업 큐(Task Queue)**에 강점을 갖는다. RabbitMQ는 **AMQP(Advanced Message Queuing Protocol)**를 핵심 프로토콜로 구현하며, 이 프로토콜의 설계 철학을 이해하는 것이 RabbitMQ의 동작 원리를 파악하는 열쇠다.

---

## 왜 메시지 브로커가 필요한가

직접 HTTP 통신 대신 메시지 브로커를 사용하는 이유:

1. **비동기 디커플링**: 생산자(Producer)는 소비자(Consumer)의 가용 여부를 신경 쓰지 않는다. 메시지를 브로커에 던지면 끝이다.
2. **탄력성(Resilience)**: 소비자가 일시적으로 다운되어도 메시지는 큐에 보존된다. 서비스가 복구되면 밀린 메시지를 처리한다.
3. **부하 분산**: 여러 소비자가 같은 큐를 소비하면 자동으로 워크로드가 분산된다(경쟁 소비자 패턴).
4. **유연한 라우팅**: 하나의 메시지를 여러 소비자에게 복사 전달(Pub/Sub), 특정 키에 따라 선택적 전달 등 복잡한 라우팅이 가능하다.

---

## AMQP 0-9-1 프로토콜 구조

AMQP는 **네트워크 프로토콜**(와이어 레벨)과 **브로커 모델** 양쪽을 모두 정의하는 표준이다. AMQP 0-9-1 스펙의 핵심 개념:

### Connection과 Channel

```
클라이언트 ←—TCP 소켓(1개)—→ RabbitMQ 브로커
               ↑
     Channel 1, Channel 2, Channel 3, ... (논리적 멀티플렉싱)
```

- **Connection**: 브로커와의 실제 TCP 소켓 연결. TLS를 통해 암호화.
- **Channel**: Connection 내의 가상 연결. 하나의 TCP 연결 위에서 수천 개의 채널이 독립적으로 동작하여 연결 비용을 절감한다.

```python
# 예제 1: Python(pika 라이브러리)로 RabbitMQ Connection과 Channel 관리
import pika
import json

# Connection 파라미터 설정
credentials = pika.PlainCredentials('user', 'password')
params = pika.ConnectionParameters(
    host='localhost',
    port=5672,
    virtual_host='/',
    credentials=credentials,
    heartbeat=600,          # 10분 heartbeat — 유휴 연결 감지
    blocked_connection_timeout=300
)

connection = pika.BlockingConnection(params)

# 하나의 Connection에서 여러 Channel 생성
channel1 = connection.channel()  # 생산자용
channel2 = connection.channel()  # 소비자용
# 두 채널은 같은 TCP 연결 공유 → 효율적

# Channel은 thread-safe하지 않음! 스레드마다 별도 Channel 필요
```

---

## Exchange: 메시지 라우팅의 핵심

AMQP의 가장 강력한 개념이 **Exchange**다. 생산자는 메시지를 **큐에 직접 보내지 않고 Exchange에 발행(publish)**한다. Exchange는 **Binding 규칙**에 따라 메시지를 하나 이상의 큐로 라우팅한다. 이 분리 덕분에 생산자는 어떤 큐가 존재하는지 알 필요가 없다.

### Exchange 타입

| 타입 | 라우팅 방식 | 사용 사례 |
|---|---|---|
| **direct** | 라우팅 키가 바인딩 키와 정확히 일치 | 작업 큐, 특정 서비스 라우팅 |
| **fanout** | 모든 바인딩된 큐에 복사 전달 | 브로드캐스트, 이벤트 알림 |
| **topic** | 와일드카드 패턴 매칭 (*, #) | 카테고리별 구독 |
| **headers** | 메시지 헤더 속성 기반 매칭 | 복잡한 조건부 라우팅 |

### Direct Exchange

```python
# 예제 2: Direct Exchange를 이용한 작업 큐 구현

import pika
import time
import random

# === 생산자 (Producer) ===
def producer():
    connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
    channel = connection.channel()

    # Exchange 선언
    channel.exchange_declare(
        exchange='task_exchange',
        exchange_type='direct',
        durable=True  # 브로커 재시작 후에도 Exchange 유지
    )

    # 큐 선언 (durable=True: 브로커 재시작 후에도 큐 유지)
    channel.queue_declare(queue='high_priority_tasks', durable=True)
    channel.queue_declare(queue='low_priority_tasks', durable=True)

    # 바인딩: Exchange → 큐
    channel.queue_bind(
        exchange='task_exchange',
        queue='high_priority_tasks',
        routing_key='high'
    )
    channel.queue_bind(
        exchange='task_exchange',
        queue='low_priority_tasks',
        routing_key='low'
    )

    for i in range(10):
        priority = 'high' if i % 3 == 0 else 'low'
        message = json.dumps({'task_id': i, 'payload': f'task_{i}'})

        channel.basic_publish(
            exchange='task_exchange',
            routing_key=priority,  # 라우팅 키로 큐 선택
            body=message,
            properties=pika.BasicProperties(
                delivery_mode=2,  # 메시지 영속화 (디스크 저장)
                content_type='application/json',
            )
        )
        print(f"[생산자] '{priority}' 큐로 메시지 발행: task_{i}")

    connection.close()


# === 소비자 (Consumer) ===
def consumer(queue_name: str):
    connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
    channel = connection.channel()

    # prefetch_count=1: 한 번에 하나의 메시지만 처리 (공정한 분배)
    channel.basic_qos(prefetch_count=1)

    def callback(ch, method, properties, body):
        task = json.loads(body)
        print(f"[소비자] {queue_name} 처리 중: {task['task_id']}")
        time.sleep(random.uniform(0.5, 2.0))  # 작업 처리 시뮬레이션

        # 명시적 ACK — 처리 완료 후 브로커에게 확인
        ch.basic_ack(delivery_tag=method.delivery_tag)
        print(f"[소비자] ACK 전송: {task['task_id']}")

    channel.basic_consume(
        queue=queue_name,
        on_message_callback=callback
    )
    print(f"[소비자] {queue_name} 대기 중...")
    channel.start_consuming()
```

### Topic Exchange

```python
# Topic Exchange: 와일드카드로 유연한 라우팅
# * = 단어 하나, # = 0개 이상의 단어

channel.exchange_declare(exchange='logs', exchange_type='topic', durable=True)

# 라우팅 키 예시: "order.created.payment" , "user.updated.email"

# 바인딩 예:
# "order.#"    → 모든 주문 이벤트 구독
# "*.created.*" → 모든 리소스의 생성 이벤트 구독
# "#.payment"  → 결제 관련 이벤트 모두 구독

channel.queue_bind(exchange='logs', queue='order_service', routing_key='order.#')
channel.queue_bind(exchange='logs', queue='audit_log',    routing_key='#')  # 전체 구독
channel.queue_bind(exchange='logs', queue='payment_svc',  routing_key='*.*.payment')

# 메시지 발행
channel.basic_publish(
    exchange='logs',
    routing_key='order.created.payment',
    # → order_service, audit_log, payment_svc 세 큐 모두 수신
    body='{"order_id": 123, "amount": 50000}'
)

channel.basic_publish(
    exchange='logs',
    routing_key='user.updated.email',
    # → audit_log 큐만 수신
    body='{"user_id": 456, "email": "new@example.com"}'
)
```

---

## Message Acknowledgement와 영속성

### 메시지 확인(ACK/NACK)

AMQP의 신뢰성 보장 핵심은 **명시적 확인(explicit acknowledgement)** 메커니즘이다:

- `basic_ack`: 메시지 처리 완료, 브로커는 큐에서 삭제
- `basic_nack`/`basic_reject`: 처리 실패, 브로커는 메시지를 재큐잉(requeue)하거나 DLX로 전송
- **Auto ACK 금지**: `auto_ack=True`는 메시지 손실 위험 → 프로덕션에서 절대 사용 금지

```python
def robust_callback(ch, method, properties, body):
    try:
        task = json.loads(body)
        process_task(task)  # 실제 작업
        ch.basic_ack(delivery_tag=method.delivery_tag)

    except ProcessingError as e:
        # 처리 실패 — 재큐잉 (일시적 오류)
        ch.basic_nack(
            delivery_tag=method.delivery_tag,
            requeue=True  # 큐 맨 앞으로 돌려보냄
        )
    except PermanentError as e:
        # 영구 실패 — 재큐잉 않음 (Dead Letter Exchange로 이동)
        ch.basic_nack(
            delivery_tag=method.delivery_tag,
            requeue=False
        )
```

### Dead Letter Exchange(DLX)

```python
# DLX: 처리 실패하거나 TTL 만료된 메시지를 별도 큐로 라우팅
channel.queue_declare(
    queue='main_tasks',
    durable=True,
    arguments={
        'x-dead-letter-exchange': 'dlx',    # 실패 시 DLX로
        'x-dead-letter-routing-key': 'dead', # DLX 라우팅 키
        'x-message-ttl': 30000,             # 30초 TTL — 이후 DLX로
        'x-max-length': 10000,              # 최대 10,000 메시지
    }
)

channel.exchange_declare(exchange='dlx', exchange_type='direct', durable=True)
channel.queue_declare(queue='dead_letter_queue', durable=True)
channel.queue_bind(exchange='dlx', queue='dead_letter_queue', routing_key='dead')

# DLQ에서 실패 메시지 모니터링 및 재처리 로직 구현 가능
```

---

## Quorum Queue: RabbitMQ 4.x의 고가용성 큐

전통적인 Classic Queue는 단일 노드에 데이터를 저장하여 노드 장애 시 메시지를 잃을 수 있다. RabbitMQ 3.8+에서 도입된 **Quorum Queue**는 **Raft 합의 알고리즘**을 기반으로 데이터를 클러스터 내 여러 노드에 복제하여 고가용성을 보장한다.

| 특성 | Classic Queue | Quorum Queue |
|---|---|---|
| 복제 방식 | 선택적 미러링(deprecated) | Raft 기반 자동 복제 |
| 장애 복구 | 수동 개입 필요 | 자동 리더 선출 |
| 메시지 순서 | 보장 | 보장 |
| 처리량 | 높음 | 약간 낮음 (복제 오버헤드) |
| 권장 사용처 | 레거시 | 신규 프로덕션 워크로드 |

```python
# Quorum Queue 선언
channel.queue_declare(
    queue='critical_tasks',
    durable=True,
    arguments={
        'x-queue-type': 'quorum',    # Quorum Queue 타입 지정
        'x-delivery-limit': 5,       # 최대 재전송 횟수 (초과 시 DLX)
    }
)
# Quorum Queue는 반드시 durable=True여야 하며, auto-delete 불가
# 클러스터 노드 중 과반수(quorum)가 살아있으면 데이터 보존
```

---

## 클러스터링과 vHost

```
RabbitMQ 클러스터 (3노드)
  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
  │   Node 1    │  │   Node 2    │  │   Node 3    │
  │  (Leader)   │  │ (Follower)  │  │ (Follower)  │
  │  Port:5672  │  │  Port:5672  │  │  Port:5672  │
  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
         └────────────────┼─────────────────┘
                     Erlang 분산 클러스터 통신 (Port: 25672)
```

**Virtual Host(vHost)**는 논리적 격리 단위다. 하나의 RabbitMQ 인스턴스 위에 여러 vHost를 만들어 서로 다른 환경(개발/스테이징/프로덕션) 또는 테넌트를 격리할 수 있다.

---

## 주의사항과 운영 팁

### 1. 큐 길이 모니터링

큐가 무한히 쌓이면 브로커가 OOM(Out of Memory)으로 죽는다. `x-max-length`와 `x-overflow` 정책을 설정하라.

### 2. Prefetch Count 조정

`basic_qos(prefetch_count=N)`은 소비자가 ACK 없이 보유할 수 있는 최대 메시지 수를 제한한다. 너무 크면 특정 소비자에게 부하가 몰리고, 너무 작으면(1) 처리량이 제한된다. 작업 시간과 소비자 수에 따라 실험적으로 조정하라.

### 3. 연결 풀링

Thread-per-connection 방식은 금지다. 애플리케이션당 Connection 1~2개를 사용하고, 스레드별로 Channel을 할당하라.

### 4. 영속 메시지와 처리량 트레이드오프

`delivery_mode=2`(영속 메시지)는 모든 메시지를 디스크에 fsync하므로 처리량이 크게 낮아진다. 손실을 허용할 수 있는 임시 작업은 `delivery_mode=1`(일시 메시지)를 사용하라.

---

## 정리

AMQP는 Exchange-Binding-Queue 구조를 통해 생산자와 소비자를 완전히 디커플링하고, 4가지 Exchange 타입(direct, fanout, topic, headers)으로 다양한 라우팅 시나리오를 지원한다. RabbitMQ는 이 프로토콜 위에 Quorum Queue(Raft 복제), DLX, TTL, 클러스터링을 더해 프로덕션 수준의 신뢰성을 제공한다. 메시지의 영속성, 명시적 ACK, 적절한 prefetch 설정이 신뢰할 수 있는 메시징 시스템의 세 가지 핵심 원칙이다.

## 참고 자료

- [AMQP 0-9-1 Model Explained - RabbitMQ 공식 문서](https://www.rabbitmq.com/tutorials/amqp-concepts)
- [What is AMQP and why is it used in RabbitMQ? - CloudAMQP](https://www.cloudamqp.com/blog/what-is-amqp-and-why-is-it-used-in-rabbitmq.html)
- [How RabbitMQ Works Internally: AMQP Protocol, Exchange Routing, and the Quorum Queue Engine](https://letsbuildsolutions.com/blog/system-design/how-rabbitmq-works-internally-amqp-protocol-exchange-routing-and-the-quorum-queue-engine-behind-reliable-messaging/)
- [RabbitMQ Architecture Explained: Exchanges, Queues, and Message Routing](https://pdtn.org/rabbitmq-architecture/)
