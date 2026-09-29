---
layout: post
title: "도메인 주도 설계(DDD) 완전 정복: Aggregate, Bounded Context, Domain Event로 복잡한 비즈니스 로직 정복하기"
date: 2026-09-29
categories: [cs, computer-science]
tags: [ddd, domain-driven-design, architecture, aggregate, bounded-context, domain-event, repository, value-object, entity]
---

## 도메인 주도 설계란 무엇인가

도메인 주도 설계(Domain-Driven Design, DDD)는 2003년 에릭 에반스(Eric Evans)가 저서 "Domain-Driven Design: Tackling Complexity in the Heart of Software"에서 체계화한 소프트웨어 설계 방법론이다. DDD의 핵심 철학은 **소프트웨어의 복잡성은 기술적 문제가 아닌 비즈니스 도메인의 복잡성에서 비롯된다**는 인식이다.

DDD는 두 가지 큰 축으로 구성된다. **전략적 설계(Strategic Design)**는 시스템 전체 구조와 팀 간 경계를 다루며, **전술적 설계(Tactical Design)**는 단일 바운디드 컨텍스트 내부의 구현 패턴을 다룬다.

## 왜 DDD가 필요한가

전통적인 데이터 중심 설계(data-centric design)에서는 데이터베이스 테이블이 시스템의 중심이 된다. 이 접근법은 초기에는 단순해 보이지만, 비즈니스 규칙이 복잡해질수록 여러 문제가 발생한다.

**빈혈 도메인 모델(Anemic Domain Model)**: 모든 비즈니스 로직이 서비스 계층에 집중되고, 도메인 객체는 단순히 데이터를 운반하는 DTO로 전락한다. 결과적으로 절차적 코드가 난무하며, 코드 재사용성과 테스트 가능성이 낮아진다.

**언어 불일치**: 개발자가 사용하는 기술 용어(User, Record, Row)와 비즈니스 전문가가 사용하는 도메인 언어(Customer, Order, Invoice)가 달라지면, 요구사항 해석 오류가 반복된다.

**경계 없는 시스템**: 단일 모델이 시스템 전체를 표현하려 할 때, 모델 하나의 변경이 예측 불가능한 사이드 이펙트를 일으킨다.

DDD는 이런 문제를 해결하기 위해 **유비쿼터스 언어(Ubiquitous Language)**—개발자와 도메인 전문가가 공유하는 공통 어휘—를 중심에 놓고, 명확한 경계와 책임을 가진 모델을 구축한다.

## 전술적 설계 패턴

### 엔티티(Entity)와 값 객체(Value Object)

**엔티티**는 고유한 식별자를 가지며, 식별자가 같으면 속성이 달라도 동일한 객체로 간주한다. 예를 들어 사용자는 이름이 변경되어도 같은 사용자다.

**값 객체**는 식별자가 없고, 속성 값 자체가 동등성을 결정한다. `Money(100, "KRW")`와 `Money(100, "KRW")`는 동일하다. 값 객체는 불변(immutable)으로 설계하는 것이 원칙이다.

```python
from dataclasses import dataclass
from typing import Optional
import uuid

# 값 객체: 불변, 속성으로 동등성 비교
@dataclass(frozen=True)
class Money:
    amount: int
    currency: str

    def __add__(self, other: "Money") -> "Money":
        if self.currency != other.currency:
            raise ValueError(f"통화 불일치: {self.currency} vs {other.currency}")
        return Money(self.amount + other.amount, self.currency)

    def __str__(self) -> str:
        return f"{self.amount:,} {self.currency}"


# 값 객체: 이메일
@dataclass(frozen=True)
class Email:
    value: str

    def __post_init__(self):
        if "@" not in self.value:
            raise ValueError(f"유효하지 않은 이메일: {self.value}")


# 엔티티: 고유 ID를 가짐, 속성이 변해도 동일 객체
class Customer:
    def __init__(self, customer_id: str, email: Email, name: str):
        self._id = customer_id
        self._email = email
        self._name = name
        self._credit_limit = Money(0, "KRW")

    @classmethod
    def create(cls, email: str, name: str) -> "Customer":
        return cls(
            customer_id=str(uuid.uuid4()),
            email=Email(email),
            name=name,
        )

    def change_email(self, new_email: str) -> None:
        self._email = Email(new_email)  # 이메일이 바뀌어도 같은 Customer

    def set_credit_limit(self, limit: Money) -> None:
        if limit.amount < 0:
            raise ValueError("신용 한도는 0 이상이어야 합니다")
        self._credit_limit = limit

    def __eq__(self, other) -> bool:
        if not isinstance(other, Customer):
            return False
        return self._id == other._id  # ID로만 동등성 비교

    def __hash__(self) -> int:
        return hash(self._id)

    @property
    def id(self) -> str:
        return self._id

    @property
    def credit_limit(self) -> Money:
        return self._credit_limit


# 사용 예시
customer = Customer.create("alice@example.com", "Alice Kim")
customer.set_credit_limit(Money(500_000, "KRW"))
print(f"고객 ID: {customer.id}")
print(f"신용 한도: {customer.credit_limit}")

m1 = Money(100_000, "KRW")
m2 = Money(200_000, "KRW")
print(f"합계: {m1 + m2}")
```

### Aggregate와 Aggregate Root

**Aggregate**는 일관성 경계를 형성하는 관련 객체들의 클러스터다. **Aggregate Root**는 외부에서 Aggregate 내부에 접근하는 유일한 진입점이며, 불변식(invariant)을 보호할 책임이 있다.

Aggregate 설계의 핵심 규칙:
- Aggregate 외부에서는 Root를 통해서만 내부에 접근한다
- Aggregate 간 참조는 ID로만 한다 (직접 객체 참조 금지)
- 하나의 트랜잭션은 하나의 Aggregate만 수정한다

```python
from dataclasses import dataclass, field
from typing import List
from enum import Enum
import uuid

class OrderStatus(Enum):
    PENDING = "pending"
    CONFIRMED = "confirmed"
    SHIPPED = "shipped"
    CANCELLED = "cancelled"

@dataclass(frozen=True)
class ProductId:
    value: str

@dataclass(frozen=True)
class OrderId:
    value: str

    @classmethod
    def generate(cls) -> "OrderId":
        return cls(str(uuid.uuid4()))


# Order Line: Order Aggregate 내부 엔티티
class OrderLine:
    def __init__(self, line_id: str, product_id: ProductId, quantity: int, unit_price: Money):
        if quantity <= 0:
            raise ValueError("수량은 1 이상이어야 합니다")
        if unit_price.amount <= 0:
            raise ValueError("단가는 0보다 커야 합니다")
        self._id = line_id
        self._product_id = product_id
        self._quantity = quantity
        self._unit_price = unit_price

    @property
    def subtotal(self) -> Money:
        return Money(self._unit_price.amount * self._quantity, self._unit_price.currency)

    @property
    def product_id(self) -> ProductId:
        return self._product_id

    @property
    def quantity(self) -> int:
        return self._quantity


# Order: Aggregate Root — 불변식 보호 책임
class Order:
    MAX_LINES = 10

    def __init__(self, order_id: OrderId, customer_id: str):
        self._id = order_id
        self._customer_id = customer_id  # Customer를 ID로만 참조
        self._lines: List[OrderLine] = []
        self._status = OrderStatus.PENDING
        self._domain_events: List[dict] = []

    @classmethod
    def create(cls, customer_id: str) -> "Order":
        order = cls(OrderId.generate(), customer_id)
        order._record_event("OrderCreated", {"order_id": order._id.value})
        return order

    def add_item(self, product_id: str, quantity: int, unit_price: Money) -> None:
        # 불변식 보호: 확정된 주문에는 아이템 추가 불가
        if self._status != OrderStatus.PENDING:
            raise ValueError("확정된 주문에는 상품을 추가할 수 없습니다")
        # 불변식 보호: 최대 라인 수 초과 금지
        if len(self._lines) >= self.MAX_LINES:
            raise ValueError(f"주문 라인은 최대 {self.MAX_LINES}개까지만 추가할 수 있습니다")
        line = OrderLine(
            line_id=str(uuid.uuid4()),
            product_id=ProductId(product_id),
            quantity=quantity,
            unit_price=unit_price,
        )
        self._lines.append(line)

    def confirm(self) -> None:
        if not self._lines:
            raise ValueError("주문 항목이 없어 확정할 수 없습니다")
        if self._status != OrderStatus.PENDING:
            raise ValueError("대기 상태의 주문만 확정할 수 있습니다")
        self._status = OrderStatus.CONFIRMED
        self._record_event("OrderConfirmed", {
            "order_id": self._id.value,
            "total": self.total_amount.amount,
        })

    def cancel(self) -> None:
        if self._status == OrderStatus.SHIPPED:
            raise ValueError("배송된 주문은 취소할 수 없습니다")
        self._status = OrderStatus.CANCELLED
        self._record_event("OrderCancelled", {"order_id": self._id.value})

    @property
    def total_amount(self) -> Money:
        if not self._lines:
            return Money(0, "KRW")
        total = sum(line.subtotal.amount for line in self._lines)
        return Money(total, "KRW")

    @property
    def status(self) -> OrderStatus:
        return self._status

    def pop_domain_events(self) -> List[dict]:
        events = list(self._domain_events)
        self._domain_events.clear()
        return events

    def _record_event(self, event_type: str, payload: dict) -> None:
        self._domain_events.append({"type": event_type, "payload": payload})


# 사용 예시
order = Order.create(customer_id="customer-123")
order.add_item("product-A", quantity=2, unit_price=Money(15_000, "KRW"))
order.add_item("product-B", quantity=1, unit_price=Money(35_000, "KRW"))
print(f"총액: {order.total_amount}")

order.confirm()
print(f"상태: {order.status.value}")

events = order.pop_domain_events()
for event in events:
    print(f"도메인 이벤트: {event['type']} → {event['payload']}")
```

### Repository 패턴

Repository는 Aggregate의 영속성을 추상화하며, 컬렉션처럼 Aggregate를 저장하고 조회하는 인터페이스를 제공한다. 인프라 계층의 구현 세부사항이 도메인 계층에 노출되지 않도록 인터페이스를 분리한다.

```python
from abc import ABC, abstractmethod
from typing import Optional

# 도메인 계층: 인터페이스만 정의
class OrderRepository(ABC):
    @abstractmethod
    def find_by_id(self, order_id: OrderId) -> Optional[Order]:
        pass

    @abstractmethod
    def save(self, order: Order) -> None:
        pass

    @abstractmethod
    def next_id(self) -> OrderId:
        pass


# 인프라 계층: 실제 구현 (인메모리 예시)
class InMemoryOrderRepository(OrderRepository):
    def __init__(self):
        self._store: dict = {}

    def find_by_id(self, order_id: OrderId) -> Optional[Order]:
        return self._store.get(order_id.value)

    def save(self, order: Order) -> None:
        self._store[order._id.value] = order

    def next_id(self) -> OrderId:
        return OrderId.generate()


# 애플리케이션 서비스: 유스케이스 조율
class OrderService:
    def __init__(self, order_repo: OrderRepository):
        self._order_repo = order_repo

    def place_order(self, customer_id: str, items: list) -> str:
        order = Order.create(customer_id)
        for item in items:
            order.add_item(item["product_id"], item["quantity"],
                           Money(item["unit_price"], "KRW"))
        order.confirm()
        self._order_repo.save(order)
        return order._id.value


repo = InMemoryOrderRepository()
service = OrderService(repo)
order_id = service.place_order("customer-123", [
    {"product_id": "prod-1", "quantity": 2, "unit_price": 20_000},
])
print(f"생성된 주문 ID: {order_id}")
```

## 전략적 설계: Bounded Context

**Bounded Context**는 특정 도메인 모델이 적용되는 명시적인 경계다. 대규모 시스템에서 "Customer"라는 단어는 판매 시스템에서는 "구매 이력이 있는 사람", CRM에서는 "연락처 정보가 있는 사람", 배송 시스템에서는 "배송지 주소를 가진 사람"으로 다르게 정의될 수 있다.

각 Bounded Context는:
- 자신만의 유비쿼터스 언어를 갖는다
- 독립적으로 배포될 수 있다 (마이크로서비스와 자연스럽게 정렬)
- 다른 컨텍스트와 Anti-Corruption Layer를 통해 통신한다

**Context Map**은 여러 Bounded Context 사이의 관계를 문서화한다. 관계 패턴으로는 Shared Kernel(공유 코어), Customer-Supplier(상하 의존), Conformist(순응자), Anti-Corruption Layer(부패 방지 계층) 등이 있다.

## 도메인 이벤트(Domain Event)

도메인 이벤트는 도메인에서 의미 있는 사건이 발생했음을 나타내는 불변 객체다. 이벤트는 과거 시제로 이름을 짓는다 (`OrderConfirmed`, `PaymentProcessed`). 이벤트 기반 통신은 Bounded Context 간 결합도를 낮추는 핵심 수단이다.

## 주의사항과 실전 팁

**DDD는 항상 정답이 아니다**: CRUD 중심의 단순한 시스템이나 도메인 로직이 거의 없는 프로젝트에 DDD를 적용하면 오버엔지니어링이 된다. 도메인 복잡성이 높은 핵심 서브도메인(Core Domain)에만 집중적으로 적용하라.

**Aggregate 크기**: Aggregate는 작게 유지하라. 많은 엔티티를 하나의 Aggregate에 넣으면 동시성 충돌과 성능 문제가 발생한다. 하나의 Aggregate에는 최소한의 불변식만 포함시켜라.

**ID 참조 원칙**: Aggregate 간에는 직접 객체 참조가 아닌 ID로만 참조한다. 이를 어기면 트랜잭션 경계가 모호해지고 결합도가 높아진다.

**Repository는 Aggregate Root 단위로**: 각 Aggregate Root마다 Repository를 하나씩 만든다. OrderLine에 대한 Repository는 만들지 않는다.

**이벤트 스토밍(Event Storming)**: 도메인 전문가와 개발자가 함께 Domain Event를 발견하고 Aggregate, Command, Policy를 식별하는 협업 워크숍 기법이다. DDD 초기 도입 시 가장 효과적인 방법 중 하나다.

## 참고 자료
- [Eric Evans - Domain-Driven Design Reference](https://www.domainlanguage.com/ddd/reference/)
- [Martin Fowler - DomainDrivenDesign](https://martinfowler.com/bliki/DomainDrivenDesign.html)
- [Vaughn Vernon - Implementing Domain-Driven Design](https://vaughnvernon.com/)
- [Domain-Driven Design Community](https://dddcommunity.org/)
