---
layout: post
title: "헥사고날 아키텍처(Hexagonal Architecture) 완전 정복: Ports & Adapters로 테스트 가능하고 유연한 시스템 설계하기"
date: 2026-09-29
categories: [cs, computer-science]
tags: [hexagonal-architecture, ports-adapters, clean-architecture, software-design, dependency-inversion, testability, ddd]
---

## 헥사고날 아키텍처란 무엇인가

헥사고날 아키텍처(Hexagonal Architecture)는 2005년 앨리스터 코크번(Alistair Cockburn)이 제안한 소프트웨어 아키텍처 패턴으로, **Ports and Adapters 패턴**이라고도 불린다. 이 아키텍처의 근본 목표는 **애플리케이션의 비즈니스 로직(코어)을 외부 세계로부터 완전히 격리**하는 것이다.

코크번은 소프트웨어가 외부 시스템—UI, 데이터베이스, 메시지 큐, 외부 API—에 직접 의존할 때 발생하는 문제를 관찰했다. 테스트하려면 실제 데이터베이스가 필요하고, UI가 바뀌면 비즈니스 로직도 수정해야 하며, 기술 스택 변경이 핵심 도메인 코드에 파급된다. 이 모든 문제의 근원은 **의존성의 방향**이다.

헥사고날 아키텍처는 외부 시스템에서 코어로, 코어에서 외부 시스템으로 향하는 두 방향의 상호작용을 각각 **Primary Port(Driving Side)**와 **Secondary Port(Driven Side)**로 체계화하고, 이를 **Adapter**를 통해 연결한다.

이름이 "헥사고날"인 이유는 특정 의미가 있는 6각형이 아니라, 애플리케이션의 각 면(side)에 여러 포트를 꽂을 수 있다는 아이디어를 시각화하기 위해 다각형을 선택했기 때문이다.

## 왜 헥사고날 아키텍처가 필요한가

전통적인 레이어드 아키텍처(Layered Architecture)는 Presentation → Business Logic → Persistence로 단방향 의존성을 갖는다. 이 구조의 문제점은 다음과 같다.

**테스트의 어려움**: 비즈니스 로직을 테스트하려면 그 아래 레이어인 데이터베이스, 외부 서비스가 필요하다. 통합 테스트는 느리고 환경 의존성이 높다.

**기술 결합**: 비즈니스 로직 코드 안에 `SELECT * FROM users WHERE ...` 같은 SQL이나 HTTP 클라이언트 코드가 섞인다. ORM을 교체하거나 REST를 gRPC로 바꾸면 비즈니스 코드를 수정해야 한다.

**병렬 개발의 한계**: UI와 DB가 결정되지 않으면 비즈니스 로직 개발을 시작하기 어렵다.

헥사고날 아키텍처는 **의존성 역전 원칙(Dependency Inversion Principle)**을 철저히 적용해 이 모든 문제를 해결한다.

## 핵심 구성 요소

### 애플리케이션 코어(Application Core)

비즈니스 로직, 도메인 모델, 유스케이스가 여기에 위치한다. **외부 라이브러리나 프레임워크에 대한 의존성이 없다.** 순수 언어 코드로만 작성된다.

### 포트(Port)

코어가 외부와 소통하는 인터페이스다. 두 종류가 있다.

- **Primary Port (Inbound Port)**: 외부에서 코어를 호출하는 인터페이스. UseCase 인터페이스가 대표적이다.
- **Secondary Port (Outbound Port)**: 코어가 외부 시스템을 호출하기 위해 정의하는 인터페이스. Repository, EmailSender 등이 대표적이다.

### 어댑터(Adapter)

포트를 구현하는 실제 코드다.

- **Primary Adapter**: REST 컨트롤러, CLI 핸들러, 이벤트 소비자 등 — 외부 요청을 Primary Port로 변환한다.
- **Secondary Adapter**: JPA Repository 구현체, SMTP 이메일 전송, Kafka Producer 등 — Secondary Port를 실제 외부 시스템과 연결한다.

## 실전 구현: 주문 도메인

다음은 헥사고날 아키텍처를 Python으로 구현한 주문 처리 시스템 예시다. 레이어를 명확히 분리한다.

```python
# ============================================================
# 도메인 계층: 순수 비즈니스 로직, 외부 의존성 없음
# ============================================================
from dataclasses import dataclass, field
from typing import List, Optional
from enum import Enum
import uuid

class OrderStatus(Enum):
    PENDING = "pending"
    CONFIRMED = "confirmed"
    CANCELLED = "cancelled"

@dataclass
class OrderItem:
    product_id: str
    quantity: int
    unit_price: float

    @property
    def subtotal(self) -> float:
        return self.quantity * self.unit_price

@dataclass
class Order:
    id: str
    customer_id: str
    items: List[OrderItem] = field(default_factory=list)
    status: OrderStatus = OrderStatus.PENDING

    @classmethod
    def create(cls, customer_id: str) -> "Order":
        return cls(id=str(uuid.uuid4()), customer_id=customer_id)

    def add_item(self, product_id: str, quantity: int, unit_price: float) -> None:
        if self.status != OrderStatus.PENDING:
            raise ValueError("확정된 주문에 상품을 추가할 수 없습니다")
        self.items.append(OrderItem(product_id, quantity, unit_price))

    def confirm(self) -> None:
        if not self.items:
            raise ValueError("빈 주문은 확정할 수 없습니다")
        self.status = OrderStatus.CONFIRMED

    @property
    def total(self) -> float:
        return sum(item.subtotal for item in self.items)


# ============================================================
# 포트(Port): 코어가 정의하는 인터페이스
# ============================================================
from abc import ABC, abstractmethod

# Secondary Port: 코어가 필요로 하는 외부 기능
class OrderRepository(ABC):
    @abstractmethod
    def save(self, order: Order) -> None: ...

    @abstractmethod
    def find_by_id(self, order_id: str) -> Optional[Order]: ...

class NotificationPort(ABC):
    @abstractmethod
    def send_confirmation(self, customer_id: str, order_id: str, total: float) -> None: ...

# Primary Port: 코어가 제공하는 유스케이스 인터페이스
class PlaceOrderUseCase(ABC):
    @abstractmethod
    def execute(self, customer_id: str, items: list) -> str: ...

class GetOrderUseCase(ABC):
    @abstractmethod
    def execute(self, order_id: str) -> Optional[Order]: ...


# ============================================================
# 애플리케이션 서비스(Application Service): 유스케이스 구현
# — Primary Port를 구현하고, Secondary Port에만 의존
# ============================================================
class OrderApplicationService(PlaceOrderUseCase, GetOrderUseCase):
    def __init__(self, order_repo: OrderRepository, notifier: NotificationPort):
        self._order_repo = order_repo
        self._notifier = notifier

    def execute(self, customer_id: str, items: list) -> str:
        order = Order.create(customer_id)
        for item in items:
            order.add_item(item["product_id"], item["quantity"], item["unit_price"])
        order.confirm()
        self._order_repo.save(order)
        self._notifier.send_confirmation(customer_id, order.id, order.total)
        return order.id

    def execute(self, order_id: str) -> Optional[Order]:  # GetOrderUseCase 구현
        return self._order_repo.find_by_id(order_id)


# ============================================================
# Secondary Adapters: Secondary Port 구현체
# — 외부 시스템과의 실제 연결
# ============================================================

# 인메모리 구현 (테스트용)
class InMemoryOrderRepository(OrderRepository):
    def __init__(self):
        self._store: dict = {}

    def save(self, order: Order) -> None:
        self._store[order.id] = order

    def find_by_id(self, order_id: str) -> Optional[Order]:
        return self._store.get(order_id)


# 콘솔 알림 (개발용)
class ConsoleNotificationAdapter(NotificationPort):
    def send_confirmation(self, customer_id: str, order_id: str, total: float) -> None:
        print(f"[알림] 고객 {customer_id}의 주문 {order_id} 확정. 총액: {total:,.0f}원")


# 이메일 알림 (프로덕션용)
class EmailNotificationAdapter(NotificationPort):
    def __init__(self, smtp_host: str):
        self._smtp_host = smtp_host

    def send_confirmation(self, customer_id: str, order_id: str, total: float) -> None:
        # 실제 이메일 발송 로직
        print(f"[SMTP:{self._smtp_host}] 이메일 발송 → 고객 {customer_id}: 주문 {order_id} ({total:,.0f}원)")


# ============================================================
# Primary Adapter: REST 컨트롤러 (Primary Port 호출)
# ============================================================
class OrderRestController:
    def __init__(self, place_order: PlaceOrderUseCase):
        self._place_order = place_order

    def post_order(self, request_body: dict) -> dict:
        """POST /orders 엔드포인트 처리"""
        try:
            order_id = self._place_order.execute(
                customer_id=request_body["customer_id"],
                items=request_body["items"],
            )
            return {"status": 201, "order_id": order_id}
        except ValueError as e:
            return {"status": 400, "error": str(e)}


# ============================================================
# 조립(Composition Root): 의존성 주입
# ============================================================
def create_app(use_real_email: bool = False):
    repo = InMemoryOrderRepository()

    if use_real_email:
        notifier = EmailNotificationAdapter("smtp.example.com")
    else:
        notifier = ConsoleNotificationAdapter()

    app_service = OrderApplicationService(repo, notifier)
    controller = OrderRestController(app_service)
    return controller, app_service


# 실행
controller, service = create_app(use_real_email=False)
response = controller.post_order({
    "customer_id": "cust-001",
    "items": [
        {"product_id": "prod-A", "quantity": 2, "unit_price": 25000},
        {"product_id": "prod-B", "quantity": 1, "unit_price": 15000},
    ],
})
print(f"응답: {response}")
```

## 테스트 가능성: 헥사고날 아키텍처의 핵심 이점

헥사고날 아키텍처의 가장 강력한 이점은 **단위 테스트의 용이성**이다. 코어는 포트(인터페이스)에만 의존하므로, 테스트에서는 실제 DB나 외부 서비스 없이 가짜(Fake) 구현체를 주입할 수 있다.

```python
import pytest

# 테스트용 Fake Adapter — 실제 DB 없이 도메인 로직만 테스트
class FakeOrderRepository(OrderRepository):
    def __init__(self):
        self.saved_orders: List[Order] = []

    def save(self, order: Order) -> None:
        self.saved_orders.append(order)

    def find_by_id(self, order_id: str) -> Optional[Order]:
        return next((o for o in self.saved_orders if o.id == order_id), None)


class FakeNotificationAdapter(NotificationPort):
    def __init__(self):
        self.sent: List[dict] = []

    def send_confirmation(self, customer_id: str, order_id: str, total: float) -> None:
        self.sent.append({"customer_id": customer_id, "order_id": order_id, "total": total})


def test_place_order_success():
    # Given
    repo = FakeOrderRepository()
    notifier = FakeNotificationAdapter()
    service = OrderApplicationService(repo, notifier)

    # When
    order_id = service.execute(
        customer_id="cust-123",
        items=[{"product_id": "prod-1", "quantity": 3, "unit_price": 10000}],
    )

    # Then
    assert len(repo.saved_orders) == 1
    saved = repo.saved_orders[0]
    assert saved.id == order_id
    assert saved.status == OrderStatus.CONFIRMED
    assert saved.total == 30000

    assert len(notifier.sent) == 1
    assert notifier.sent[0]["customer_id"] == "cust-123"
    assert notifier.sent[0]["total"] == 30000


def test_place_order_empty_items_raises():
    repo = FakeOrderRepository()
    notifier = FakeNotificationAdapter()
    service = OrderApplicationService(repo, notifier)

    with pytest.raises(ValueError, match="빈 주문"):
        service.execute(customer_id="cust-456", items=[])


# 실행 (pytest 환경이 아니라면 직접 호출)
test_place_order_success()
test_place_order_empty_items_raises()
print("모든 테스트 통과!")
```

이 테스트는 데이터베이스, 이메일 서버, HTTP 스택 없이 **순수 Python만으로 비즈니스 로직을 검증**한다. 테스트 속도는 수 밀리초 수준이며, CI/CD 환경 의존성도 최소화된다.

## 레이어드 아키텍처 vs 헥사고날 아키텍처

| 기준 | 레이어드 | 헥사고날 |
|------|---------|---------|
| 의존성 방향 | 단방향 (위→아래) | 코어를 향한 단방향 |
| 외부 시스템 교체 | 어려움 (코드 수정 필요) | 쉬움 (Adapter만 교체) |
| 단위 테스트 | 어려움 (Mock 복잡) | 쉬움 (Fake 주입) |
| 기술 결합 | 높음 | 낮음 |
| 학습 곡선 | 낮음 | 중간 |

## 클린 아키텍처와의 관계

로버트 C. 마틴(Uncle Bob)의 클린 아키텍처(Clean Architecture)는 헥사고날 아키텍처를 발전시킨 개념이다. 헥사고날의 "Port & Adapter" 개념이 클린 아키텍처에서는 "Interface & Interactor"로 표현되며, 계층이 더 세분화(Entities → Use Cases → Interface Adapters → Frameworks & Drivers)된다. 핵심 원칙은 동일하다: **의존성은 항상 안쪽(도메인)을 향한다.**

## 주의사항과 실전 팁

**모든 프로젝트에 적용할 필요는 없다**: CRUD 중심의 단순한 관리 시스템이나 소규모 스크립트에 헥사고날 아키텍처를 적용하면 보일러플레이트가 과도해진다. 비즈니스 로직이 복잡하고, 여러 외부 시스템과 통합이 필요한 경우에 가장 효과적이다.

**포트는 도메인 언어로 정의하라**: `OrderRepository`는 `database_connector`가 아니다. 포트는 기술 용어가 아닌 비즈니스 의도를 표현해야 한다.

**어댑터는 얇게 유지하라**: 어댑터에 비즈니스 로직을 넣으면 테스트 이점이 사라진다. 어댑터는 형식 변환(HTTP 요청 → 도메인 객체, 도메인 객체 → DB 레코드)만 수행해야 한다.

**조립은 한 곳에서**: 의존성 주입(DI)과 객체 조립은 애플리케이션 진입점(main 함수, DI 컨테이너)에서 수행한다. 코어 코드에 `new`가 없어야 테스트가 쉬워진다.

**점진적 도입**: 기존 레이어드 아키텍처 코드베이스에서 헥사고날로 전환할 때는 가장 핵심적인 도메인 로직부터 포트와 어댑터로 분리한다. 전체를 한 번에 바꾸려 하지 말라.

## 참고 자료
- [Alistair Cockburn - Hexagonal Architecture](https://alistair.cockburn.us/hexagonal-architecture/)
- [Martin Fowler - Ports and Adapters](https://martinfowler.com/articles/injection.html)
- [Clean Architecture by Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Netflix Tech Blog - Hexagonal Architecture](https://netflixtechblog.com/)
