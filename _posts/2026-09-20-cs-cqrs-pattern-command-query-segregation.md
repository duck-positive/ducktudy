---
layout: post
title: "CQRS 패턴 심화: 커맨드와 쿼리의 분리, 프로젝션, 그리고 이벤트 소싱과의 결합"
date: 2026-09-20
categories: [cs, computer-science]
tags: [cqrs, architecture, design-pattern, event-sourcing, distributed-systems, ddd, microservices]
---

## 개요

**CQRS(Command Query Responsibility Segregation)**는 Greg Young이 CQS(Command Query Separation) 원칙을 아키텍처 레벨로 발전시킨 패턴입니다. 핵심 아이디어는 단순합니다: **시스템 상태를 변경하는 작업(Command)과 상태를 조회하는 작업(Query)을 완전히 분리된 모델로 처리하라**는 것입니다.

이 패턴은 복잡한 도메인, 높은 트래픽, 읽기/쓰기 성능 요구사항이 비대칭적인 시스템에서 강력한 위력을 발휘합니다. 단, 잘못 적용하면 불필요한 복잡성만 추가되는 과한 설계가 될 수 있습니다.

---

## CQRS의 개념적 기반: CQS 원칙

**CQS(Command Query Separation)**는 Bertrand Meyer가 Eiffel 언어에서 제시한 원칙입니다:
> "모든 메서드는 Command(상태를 변경하되 값을 반환하지 않음) 또는 Query(값을 반환하되 상태를 변경하지 않음) 중 하나여야 한다."

```python
# ❌ CQS 위반: 상태 변경 + 반환 동시 수행
def pop_and_return(stack: list) -> int:
    return stack.pop()  # 수정하면서 반환

# ✅ CQS 준수: 분리
def peek(stack: list) -> int:   # Query: 읽기만
    return stack[-1]

def pop(stack: list) -> None:   # Command: 쓰기만
    stack.pop()
```

CQRS는 이 원칙을 메서드 수준에서 **시스템 아키텍처 수준**으로 끌어올립니다.

---

## CQRS 아키텍처의 핵심 구조

### 기본 CQRS 모델

```
                    ┌─────────────────────────────────────────┐
                    │              Application                  │
                    └─────┬─────────────────────┬─────────────┘
                          │                     │
                   Command │               Query │
                          ↓                     ↓
            ┌─────────────────────┐  ┌─────────────────────┐
            │   Command Handler   │  │   Query Handler      │
            │  (비즈니스 로직)     │  │  (읽기 최적화)       │
            └──────────┬──────────┘  └──────────┬──────────┘
                       │                         │
                       ↓                         ↓
            ┌─────────────────────┐  ┌─────────────────────┐
            │    Write Model      │  │    Read Model        │
            │  (도메인 집계체)     │  │  (뷰/프로젝션)       │
            └──────────┬──────────┘  └──────────┬──────────┘
                       │                         │
            ┌──────────┴──────────┐  ┌───────────┴─────────┐
            │   Write Database    │  │   Read Database      │
            │   (PostgreSQL)      │  │   (Redis, Elastic)   │
            └─────────────────────┘  └─────────────────────┘
                       │ 이벤트 발행                ↑
                       └──────────────────────────┘
                              비동기 동기화
```

### Command 모델

Command는 "무언가를 하라"는 의도를 담은 불변(immutable) 객체입니다. Command는 실패할 수 있으므로, 성공/실패 결과를 반환할 수 있습니다(Greg Young의 순수 CQRS에서는 void이지만, 실용적으로는 ID 정도는 반환).

### Query 모델

Query는 현재 상태를 읽는 작업입니다. 읽기 전용이므로 여러 데이터 소스(캐시, 검색 엔진, 읽기 전용 DB 복제본)를 활용하여 성능을 극대화합니다.

---

## 실전 코드 예제

### 예제 1: Python으로 CQRS 패턴 구현 (주문 시스템)

```python
from dataclasses import dataclass, field
from datetime import datetime
from typing import Optional
from uuid import uuid4
import copy

# ─── Command 정의 ───────────────────────────────────────────

@dataclass(frozen=True)
class PlaceOrderCommand:
    """주문 생성 커맨드: 불변 객체"""
    customer_id: str
    items: tuple  # frozenset으로도 가능
    shipping_address: str

@dataclass(frozen=True)
class CancelOrderCommand:
    """주문 취소 커맨드"""
    order_id: str
    reason: str

# ─── 도메인 이벤트 ───────────────────────────────────────────

@dataclass(frozen=True)
class OrderPlacedEvent:
    order_id: str
    customer_id: str
    items: tuple
    total_amount: float
    occurred_at: datetime = field(default_factory=datetime.utcnow)

@dataclass(frozen=True)
class OrderCancelledEvent:
    order_id: str
    reason: str
    occurred_at: datetime = field(default_factory=datetime.utcnow)

# ─── Write Model (도메인 집계체) ─────────────────────────────

class Order:
    """쓰기 모델: 비즈니스 규칙 검증 + 이벤트 발행"""

    ITEM_PRICES = {"BOOK": 15000, "PEN": 2000, "NOTEBOOK": 8000}

    def __init__(self, order_id: str, customer_id: str,
                 items: tuple, shipping_address: str):
        self.order_id = order_id
        self.customer_id = customer_id
        self.items = items
        self.shipping_address = shipping_address
        self.status = "PLACED"
        self.total_amount = sum(self.ITEM_PRICES.get(i, 0) for i in items)
        self._events: list = []

    def cancel(self, reason: str) -> None:
        """비즈니스 규칙 검증 후 취소"""
        if self.status != "PLACED":
            raise ValueError(f"Cannot cancel order in status: {self.status}")
        self.status = "CANCELLED"
        self._events.append(OrderCancelledEvent(
            order_id=self.order_id,
            reason=reason
        ))

    def pop_events(self) -> list:
        events = list(self._events)
        self._events.clear()
        return events

# ─── Command Handler ─────────────────────────────────────────

class OrderCommandHandler:
    """커맨드를 받아 도메인 로직을 실행하고 이벤트를 발행"""

    def __init__(self, write_repo, event_bus):
        self._repo = write_repo
        self._bus = event_bus

    def handle_place_order(self, cmd: PlaceOrderCommand) -> str:
        order_id = str(uuid4())[:8]
        order = Order(order_id, cmd.customer_id, cmd.items, cmd.shipping_address)
        self._repo.save(order)

        # 도메인 이벤트 발행 → Read Model 동기화 트리거
        event = OrderPlacedEvent(
            order_id=order_id,
            customer_id=cmd.customer_id,
            items=cmd.items,
            total_amount=order.total_amount
        )
        self._bus.publish(event)
        return order_id  # 실용적 CQRS: 생성된 ID는 반환

    def handle_cancel_order(self, cmd: CancelOrderCommand) -> None:
        order = self._repo.find_by_id(cmd.order_id)
        if not order:
            raise ValueError(f"Order {cmd.order_id} not found")
        order.cancel(cmd.reason)
        self._repo.save(order)

        for event in order.pop_events():
            self._bus.publish(event)

# ─── Read Model & Query Handler ──────────────────────────────

class OrderSummaryView:
    """읽기 전용 뷰 모델: 쿼리에 최적화된 구조"""
    def __init__(self):
        self._store: dict[str, dict] = {}

    def on_order_placed(self, event: OrderPlacedEvent):
        """이벤트 핸들러: Write 이벤트를 받아 Read Model 업데이트"""
        self._store[event.order_id] = {
            "order_id": event.order_id,
            "customer_id": event.customer_id,
            "items": list(event.items),
            "total_amount": event.total_amount,
            "status": "PLACED",
            "placed_at": event.occurred_at.isoformat()
        }

    def on_order_cancelled(self, event: OrderCancelledEvent):
        if event.order_id in self._store:
            self._store[event.order_id]["status"] = "CANCELLED"
            self._store[event.order_id]["cancel_reason"] = event.reason

    # Query 메서드들: 읽기 전용, 부작용 없음
    def get_order(self, order_id: str) -> Optional[dict]:
        return copy.deepcopy(self._store.get(order_id))

    def get_customer_orders(self, customer_id: str) -> list[dict]:
        return [copy.deepcopy(o) for o in self._store.values()
                if o["customer_id"] == customer_id]

# ─── 간단한 인프라 구현 (데모용) ─────────────────────────────

class InMemoryOrderRepo:
    def __init__(self):
        self._store: dict = {}

    def save(self, order: Order):
        self._store[order.order_id] = order

    def find_by_id(self, order_id: str) -> Optional[Order]:
        return self._store.get(order_id)

class SimpleEventBus:
    def __init__(self):
        self._handlers: dict[type, list] = {}

    def subscribe(self, event_type: type, handler):
        self._handlers.setdefault(event_type, []).append(handler)

    def publish(self, event):
        for handler in self._handlers.get(type(event), []):
            handler(event)

# ─── 사용 예 ──────────────────────────────────────────────────

def demo():
    repo = InMemoryOrderRepo()
    bus = SimpleEventBus()
    view = OrderSummaryView()

    # 이벤트 구독 (Read Model 동기화)
    bus.subscribe(OrderPlacedEvent, view.on_order_placed)
    bus.subscribe(OrderCancelledEvent, view.on_order_cancelled)

    handler = OrderCommandHandler(repo, bus)

    # Command 실행
    order_id = handler.handle_place_order(PlaceOrderCommand(
        customer_id="customer-1",
        items=("BOOK", "PEN", "NOTEBOOK"),
        shipping_address="서울시 강남구"
    ))
    print(f"주문 생성: {order_id}")

    # Query 실행 (Read Model에서 직접 조회)
    summary = view.get_order(order_id)
    print(f"주문 조회: {summary}")

    # 취소 Command
    handler.handle_cancel_order(CancelOrderCommand(order_id, "마음이 바뀜"))
    summary = view.get_order(order_id)
    print(f"취소 후: {summary['status']}")

demo()
```

### 예제 2: Event Sourcing과 CQRS 결합 (이벤트 스토어 기반)

CQRS의 Write 모델을 이벤트 소싱으로 구현하면, 상태(State)를 저장하는 대신 **이벤트 스트림**을 저장합니다.

```python
from dataclasses import dataclass, field
from typing import Any
from datetime import datetime

# ─── 이벤트 스토어 ────────────────────────────────────────────

@dataclass
class EventRecord:
    """이벤트 스토어의 단일 레코드"""
    stream_id: str          # Aggregate ID
    version: int            # 낙관적 동시성 제어용
    event_type: str
    payload: dict
    occurred_at: datetime = field(default_factory=datetime.utcnow)

class EventStore:
    """단순 인메모리 이벤트 스토어"""
    def __init__(self):
        self._streams: dict[str, list[EventRecord]] = {}

    def append(self, stream_id: str, events: list[dict],
               expected_version: int = -1) -> None:
        """낙관적 동시성 제어로 이벤트 저장"""
        current = self._streams.get(stream_id, [])
        current_version = len(current) - 1

        if expected_version != -1 and current_version != expected_version:
            raise Exception(
                f"Concurrency conflict: expected v{expected_version}, "
                f"got v{current_version}"
            )

        for i, event in enumerate(events):
            record = EventRecord(
                stream_id=stream_id,
                version=len(current) + i,
                event_type=event["type"],
                payload=event["payload"]
            )
            current.append(record)
        self._streams[stream_id] = current

    def load(self, stream_id: str, from_version: int = 0) -> list[EventRecord]:
        return [r for r in self._streams.get(stream_id, [])
                if r.version >= from_version]

# ─── 이벤트 소싱 기반 집계체 ──────────────────────────────────

class BankAccount:
    """이벤트 소싱으로 구현된 은행 계좌 집계체"""

    def __init__(self, account_id: str):
        self.account_id = account_id
        self.balance = 0
        self.owner = ""
        self.is_open = False
        self.version = -1
        self._pending_events: list[dict] = []

    # ─ Command 처리 (비즈니스 규칙 검증 → 이벤트 발행)

    def open(self, owner: str, initial_deposit: float) -> None:
        if self.is_open:
            raise ValueError("Account already open")
        if initial_deposit < 0:
            raise ValueError("Initial deposit cannot be negative")
        self._emit("AccountOpened", {
            "owner": owner, "initial_deposit": initial_deposit
        })

    def deposit(self, amount: float) -> None:
        if not self.is_open:
            raise ValueError("Account is closed")
        if amount <= 0:
            raise ValueError("Deposit amount must be positive")
        self._emit("MoneyDeposited", {"amount": amount})

    def withdraw(self, amount: float) -> None:
        if not self.is_open:
            raise ValueError("Account is closed")
        if amount > self.balance:
            raise ValueError(f"Insufficient funds: {self.balance} < {amount}")
        self._emit("MoneyWithdrawn", {"amount": amount})

    # ─ 이벤트 적용 (상태 재구성)

    def _apply(self, event_type: str, payload: dict) -> None:
        """이벤트를 현재 상태에 반영 (순수 함수, 부작용 없음)"""
        if event_type == "AccountOpened":
            self.owner = payload["owner"]
            self.balance = payload["initial_deposit"]
            self.is_open = True
        elif event_type == "MoneyDeposited":
            self.balance += payload["amount"]
        elif event_type == "MoneyWithdrawn":
            self.balance -= payload["amount"]

    def _emit(self, event_type: str, payload: dict) -> None:
        self._pending_events.append({"type": event_type, "payload": payload})
        self._apply(event_type, payload)  # 즉시 상태에 반영

    def pop_events(self) -> list[dict]:
        events = list(self._pending_events)
        self._pending_events.clear()
        return events

    @classmethod
    def from_events(cls, account_id: str,
                    records: list[EventRecord]) -> "BankAccount":
        """이벤트 스트림으로 집계체 재구성 (Event Replay)"""
        account = cls(account_id)
        for record in records:
            account._apply(record.event_type, record.payload)
            account.version = record.version
        return account

# ─── 프로젝션 (Read Model 생성기) ─────────────────────────────

class AccountBalanceProjection:
    """이벤트 스트림에서 잔액 조회용 Read Model을 생성"""

    def __init__(self):
        self._balances: dict[str, dict] = {}

    def handle(self, record: EventRecord) -> None:
        sid = record.stream_id
        if record.event_type == "AccountOpened":
            self._balances[sid] = {
                "account_id": sid,
                "owner": record.payload["owner"],
                "balance": record.payload["initial_deposit"],
                "transaction_count": 1
            }
        elif record.event_type == "MoneyDeposited":
            if sid in self._balances:
                self._balances[sid]["balance"] += record.payload["amount"]
                self._balances[sid]["transaction_count"] += 1
        elif record.event_type == "MoneyWithdrawn":
            if sid in self._balances:
                self._balances[sid]["balance"] -= record.payload["amount"]
                self._balances[sid]["transaction_count"] += 1

    def get_balance(self, account_id: str) -> dict:
        return dict(self._balances.get(account_id, {}))

    def get_all_accounts(self) -> list[dict]:
        return list(self._balances.values())

# ─── 통합 데모 ────────────────────────────────────────────────

def event_sourcing_demo():
    store = EventStore()
    projection = AccountBalanceProjection()
    account_id = "account-001"

    # Command 처리
    account = BankAccount(account_id)
    account.open("김철수", 100_000)
    account.deposit(50_000)
    account.withdraw(30_000)

    # 이벤트 저장
    events = account.pop_events()
    store.append(account_id, events)

    # 프로젝션 업데이트
    for record in store.load(account_id):
        projection.handle(record)

    # Query: Read Model에서 조회
    balance_view = projection.get_balance(account_id)
    print(f"계좌 잔액: {balance_view['balance']:,}원")
    print(f"거래 횟수: {balance_view['transaction_count']}회")

    # Event Replay: 이벤트에서 집계체 재구성
    records = store.load(account_id)
    restored = BankAccount.from_events(account_id, records)
    print(f"재구성된 잔액: {restored.balance:,}원")
    assert restored.balance == balance_view["balance"]
    print("Write Model ↔ Read Model 일치 확인!")

event_sourcing_demo()
```

---

## CQRS의 장단점과 적용 기준

### 장점

**1. 읽기/쓰기 독립적 최적화**
- Write: 정규화된 관계형 DB (데이터 무결성 보장)
- Read: 비정규화된 Redis/Elasticsearch/MongoDB (빠른 조회)

**2. 확장성**
- 대부분의 시스템은 읽기 트래픽이 쓰기보다 10~100배 많음
- Read Model 서버만 수평 확장 가능

**3. 복잡한 쿼리 단순화**
- Write Model의 복잡한 도메인 집계체와 관계없이 쿼리용 뷰를 별도로 유지

**4. 이벤트 소싱과의 자연스러운 결합**
- 완전한 감사 이력 (Audit Trail)
- 시간 여행 디버깅 (특정 시점 상태 재현)

### 단점

| 단점 | 설명 |
|------|------|
| 최종 일관성 | Write 후 Read 모델 업데이트까지 지연 발생 |
| 복잡성 증가 | 두 모델 동기화, 이벤트 처리 등 추가 인프라 필요 |
| 중복 코드 | Write/Read 모델이 유사한 데이터를 다름 방식으로 표현 |
| 학습 비용 | 개념적 전환이 필요하고 온보딩이 어려움 |

### CQRS를 도입해야 할 때

- 읽기와 쓰기의 성능 요구사항이 크게 다를 때
- 복잡한 도메인 규칙이 있고 다양한 읽기 뷰가 필요할 때
- 이벤트 드리븐 마이크로서비스 아키텍처를 구축할 때
- 완전한 감사 이력이 필요할 때 (금융, 의료, 법률)

### CQRS를 피해야 할 때

- 단순한 CRUD 애플리케이션
- 팀이 작고 복잡성을 감당하기 어려울 때
- 강한 일관성(Strong Consistency)이 필수적일 때

---

## 주의사항과 팁

**1. 최종 일관성(Eventual Consistency)을 UI에 반영하세요**
Command 성공 후 즉시 변경된 내용을 Query하면 아직 Read Model에 반영되지 않을 수 있습니다. UI 레벨에서 "낙관적 업데이트(Optimistic Update)"를 적용하거나, 사용자에게 "처리 중" 상태를 보여주세요.

**2. 단계적으로 도입하세요**
처음부터 완전한 CQRS를 도입하지 말고, 먼저 같은 DB에서 Read/Write 모델을 논리적으로 분리하는 것부터 시작하세요. 나중에 물리적으로 분리할 수 있습니다.

**3. 프로젝션을 재생 가능하게 설계하세요**
프로젝션은 항상 이벤트 스트림 처음부터 재생하여 재구성할 수 있어야 합니다. 이를 통해 새로운 Read 요구사항이 생기면 새 프로젝션을 만들어 과거 이벤트로 초기 상태를 구성할 수 있습니다.

**4. Command와 Event를 혼동하지 마세요**
- Command: "이것을 해달라"(요청, 거절될 수 있음, 명령형) → `PlaceOrderCommand`
- Event: "이것이 일어났다"(사실, 변경 불가, 과거형) → `OrderPlacedEvent`

---

## 참고 자료

- [CQRS - Martin Fowler's Bliki](https://martinfowler.com/bliki/CQRS.html)
- [Command and Query Responsibility Segregation (CQRS) - Microsoft Learn](https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs)
- [CQRS and Event Sourcing - Confluent Developer](https://developer.confluent.io/courses/event-sourcing/cqrs/)
- [Command Query Responsibility Segregation - Akka Guide](https://doc.akka.io/libraries/guide/concepts/cqrs.html)
