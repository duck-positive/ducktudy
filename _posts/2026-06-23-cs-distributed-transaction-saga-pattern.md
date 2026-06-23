---
layout: post
title: "분산 트랜잭션과 Saga 패턴: 마이크로서비스 데이터 일관성 완전 정복"
date: 2026-06-23
categories: [cs, computer-science]
tags: [distributed-systems, saga-pattern, microservices, distributed-transactions, 2pc, eventual-consistency, choreography, orchestration]
---

## 개념 설명: 분산 트랜잭션이란

모놀리식(Monolithic) 아키텍처에서는 단일 데이터베이스의 ACID 트랜잭션으로 데이터 일관성을 보장할 수 있다. 주문과 결제와 재고를 한 `BEGIN` ~ `COMMIT` 블록 안에 묶으면 된다.

그러나 마이크로서비스 아키텍처에서는 각 서비스가 **독립된 데이터베이스**를 소유한다(Database-per-Service 패턴). 주문 서비스(OrderDB), 결제 서비스(PaymentDB), 재고 서비스(InventoryDB)가 분리된 상황에서 "주문 → 결제 → 재고 차감"이 모두 성공하거나 모두 실패해야 하는 요구사항을 어떻게 처리할까?

이것이 **분산 트랜잭션(Distributed Transaction)** 문제다.

### 2PC(Two-Phase Commit): 고전적 해결책과 그 한계

분산 트랜잭션의 전통적 해결책은 **2단계 커밋(2PC)** 이다.

```
Phase 1 — Prepare (투표):
  코디네이터 → 참여자1: "커밋 준비됐나요?"
  코디네이터 → 참여자2: "커밋 준비됐나요?"
  코디네이터 → 참여자3: "커밋 준비됐나요?"

Phase 2 — Commit (결정):
  모두 Yes → 코디네이터 → 모두: "커밋하세요"
  하나라도 No → 코디네이터 → 모두: "롤백하세요"
```

2PC가 마이크로서비스에서 실패하는 이유:

1. **단일 장애점**: 코디네이터가 Phase 1 후, Phase 2 전에 죽으면 참여자들은 락을 걸고 영원히 대기 (blocking protocol)
2. **성능 병목**: 모든 참여자가 응답할 때까지 대기 → 느린 서비스 하나가 전체를 지연
3. **네트워크 파티션**: CAP 정리에 의해 파티션 상황에서 일관성과 가용성을 동시에 보장 불가
4. **서비스 결합 증가**: 모든 서비스가 동일한 트랜잭션 프로토콜을 지원해야 함

---

## Saga 패턴: 장기 실행 트랜잭션의 현대적 해결책

**Saga**는 1987년 Hector Garcia-Molina와 Kenneth Salem이 논문 "Sagas"에서 처음 제안한 개념이다. 마이크로서비스 맥락에서는 Chris Richardson이 대중화시켰다.

### 핵심 아이디어

하나의 큰 ACID 트랜잭션을 **여러 개의 작은 로컬 트랜잭션**으로 분해하고, 각 단계 실패 시 **보상 트랜잭션(Compensating Transaction)**으로 이미 커밋된 결과를 의미론적으로 되돌린다.

```
일반 흐름 (Happy Path):
T₁ (주문 생성) → T₂ (결제 처리) → T₃ (재고 차감) → T₄ (배송 시작)

실패 흐름 (T₃ 재고 부족):
T₁ → T₂ → T₃ 실패
      ↓
C₂ (결제 환불) → C₁ (주문 취소)
```

각 Tᵢ에 대응하는 보상 트랜잭션 Cᵢ가 존재한다. 보상 트랜잭션은 단순 DB 롤백이 아닌 **비즈니스 의미의 되돌리기**다(예: "결제 취소"는 환불 요청, "주문 취소"는 취소 이벤트 발행).

### Saga의 격리성 부재 — ACD 트랜잭션

Saga는 ACID에서 **I(격리성, Isolation)을 포기**한다. 다른 트랜잭션이 Saga의 중간 상태를 볼 수 있다(Dirty Read). 이를 허용하는 대신 **최종 일관성(Eventual Consistency)**을 보장한다.

따라서 Saga는 **ACD 트랜잭션**이라고도 부른다:
- **A**tomicity: 보상 트랜잭션으로 논리적 원자성 보장
- **C**onsistency: 비즈니스 규칙 일관성 유지
- **D**urability: 각 로컬 트랜잭션의 영속성

---

## 왜 Saga가 필요한가

마이크로서비스로 전환한 실제 시스템에서:

- **e커머스**: 주문 → 결제 → 재고 → 포인트 적립 → 배송 지시 (5개 서비스, 5개 DB)
- **여행 예약**: 항공권 → 호텔 → 렌터카 예약 (3개 외부 API)
- **금융 이체**: 출금 계좌 차감 → 수취 계좌 증가 → 이체 내역 기록

이 모든 케이스에서 2PC는 확장성과 장애 대응 측면에서 현실적이지 않다. Saga는 **느슨한 결합**을 유지하면서 분산된 일관성을 달성한다.

---

## 두 가지 구현 방식

### 1. Choreography(코레오그래피) Saga

각 서비스가 자신의 로컬 트랜잭션 완료 후 이벤트를 발행하고, 다음 서비스가 그 이벤트를 구독하여 자신의 작업을 수행한다. **중앙 오케스트레이터가 없다.**

```
OrderService     PaymentService    InventoryService   ShippingService
    │                                                      │
    │ OrderCreated ───▶                                    │
    │                  PaymentProcessed ──▶                │
    │                                     StockReserved ──▶│
    │                                                      │ ShipmentCreated
    │ ◀──────────────────────────────────────────────────  │
```

**장점**: 서비스 간 결합 최소, 단일 장애점 없음, 구현 단순  
**단점**: 전체 플로우를 한눈에 파악하기 어렵고, 사이클 의존성이 생기기 쉬움

### 2. Orchestration(오케스트레이션) Saga

중앙의 **Saga 오케스트레이터**가 각 서비스에 명령을 보내고 응답을 받아 다음 단계를 결정한다.

```
                    SagaOrchestrator
                    ┌─────────────┐
                    │ 1. Create   │──▶ OrderService
                    │ 2. Pay      │──▶ PaymentService
                    │ 3. Reserve  │──▶ InventoryService
                    │ 4. Ship     │──▶ ShippingService
                    │ [실패 시]   │
                    │ C3. Release │──▶ InventoryService
                    │ C2. Refund  │──▶ PaymentService
                    │ C1. Cancel  │──▶ OrderService
                    └─────────────┘
```

**장점**: 전체 플로우 가시성 높음, 테스트 용이, 보상 로직 한 곳에 집중  
**단점**: 오케스트레이터가 단일 장애점 가능성, 오케스트레이터와 서비스 간 결합

---

## 실제 구현 예제

### 예제 1: Choreography Saga — Spring Boot + 이벤트 (Java)

```java
// ──────────────────────────────────────────────────
// OrderService.java
// ──────────────────────────────────────────────────
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepo;
    private final ApplicationEventPublisher eventPublisher;

    @Transactional
    public Order createOrder(CreateOrderCommand cmd) {
        Order order = new Order(cmd.userId(), cmd.productId(), cmd.quantity());
        order.setStatus(OrderStatus.PENDING);
        orderRepo.save(order);

        // 로컬 트랜잭션 커밋 후 이벤트 발행
        eventPublisher.publishEvent(new OrderCreatedEvent(
            order.getId(), cmd.userId(), cmd.productId(),
            cmd.quantity(), cmd.totalAmount()
        ));
        return order;
    }

    // 보상 트랜잭션: 결제 실패 시 주문 취소
    @EventListener
    @Transactional
    public void onPaymentFailed(PaymentFailedEvent event) {
        Order order = orderRepo.findById(event.orderId()).orElseThrow();
        order.setStatus(OrderStatus.CANCELLED);
        orderRepo.save(order);
        System.out.println("Order " + order.getId() + " cancelled due to payment failure.");
    }
}

// ──────────────────────────────────────────────────
// PaymentService.java
// ──────────────────────────────────────────────────
@Service
@RequiredArgsConstructor
public class PaymentService {

    private final PaymentRepository paymentRepo;
    private final ApplicationEventPublisher eventPublisher;

    @EventListener
    @Transactional
    public void onOrderCreated(OrderCreatedEvent event) {
        try {
            // 결제 처리 시도
            Payment payment = new Payment(event.orderId(), event.amount());
            processPayment(payment); // 실제 PG 연동
            paymentRepo.save(payment);

            eventPublisher.publishEvent(new PaymentCompletedEvent(
                event.orderId(), payment.getId()
            ));
        } catch (PaymentException e) {
            // 결제 실패 — 보상 이벤트 발행
            eventPublisher.publishEvent(new PaymentFailedEvent(
                event.orderId(), e.getMessage()
            ));
        }
    }

    // 보상 트랜잭션: 재고 부족 시 환불
    @EventListener
    @Transactional
    public void onInventoryReservationFailed(InventoryReservationFailedEvent event) {
        Payment payment = paymentRepo.findByOrderId(event.orderId()).orElseThrow();
        payment.setStatus(PaymentStatus.REFUNDED);
        paymentRepo.save(payment);
        eventPublisher.publishEvent(new PaymentFailedEvent(event.orderId(), "Inventory unavailable"));
    }

    private void processPayment(Payment payment) {
        // PG API 호출 로직 (생략)
        if (payment.getAmount() <= 0) throw new PaymentException("Invalid amount");
    }
}

// ──────────────────────────────────────────────────
// InventoryService.java
// ──────────────────────────────────────────────────
@Service
@RequiredArgsConstructor
public class InventoryService {

    private final InventoryRepository inventoryRepo;
    private final ApplicationEventPublisher eventPublisher;

    @EventListener
    @Transactional
    public void onPaymentCompleted(PaymentCompletedEvent event) {
        Inventory inventory = inventoryRepo.findByProductId(event.productId())
            .orElseThrow(() -> new InventoryException("Product not found"));

        if (inventory.getStock() < event.quantity()) {
            eventPublisher.publishEvent(
                new InventoryReservationFailedEvent(event.orderId(), "Insufficient stock")
            );
            return;
        }

        inventory.decreaseStock(event.quantity());
        inventoryRepo.save(inventory);
        eventPublisher.publishEvent(
            new InventoryReservedEvent(event.orderId(), event.productId(), event.quantity())
        );
    }
}
```

### 예제 2: Orchestration Saga — 상태 머신 오케스트레이터 (Python)

```python
from enum import Enum, auto
from dataclasses import dataclass
from typing import Optional
import uuid

class SagaState(Enum):
    STARTED        = auto()
    ORDER_CREATED  = auto()
    PAYMENT_DONE   = auto()
    STOCK_RESERVED = auto()
    COMPLETED      = auto()
    # 보상 상태들
    COMPENSATING_STOCK   = auto()
    COMPENSATING_PAYMENT = auto()
    COMPENSATING_ORDER   = auto()
    FAILED               = auto()

@dataclass
class SagaContext:
    saga_id:    str
    order_id:   Optional[str] = None
    payment_id: Optional[str] = None
    state:      SagaState = SagaState.STARTED

class OrderSagaOrchestrator:
    """주문 생성 Saga 오케스트레이터 — 상태 머신 기반"""

    def __init__(self, order_svc, payment_svc, inventory_svc):
        self.order_svc     = order_svc
        self.payment_svc   = payment_svc
        self.inventory_svc = inventory_svc

    def execute(self, user_id: str, product_id: str, quantity: int, amount: float):
        ctx = SagaContext(saga_id=str(uuid.uuid4()))
        print(f"[Saga {ctx.saga_id}] 시작")

        # Step 1: 주문 생성
        ctx = self._step_create_order(ctx, user_id, product_id, quantity)
        if ctx.state == SagaState.FAILED:
            return ctx

        # Step 2: 결제 처리
        ctx = self._step_process_payment(ctx, amount)
        if ctx.state == SagaState.FAILED:
            return ctx

        # Step 3: 재고 차감
        ctx = self._step_reserve_stock(ctx, product_id, quantity)
        if ctx.state == SagaState.FAILED:
            return ctx

        ctx.state = SagaState.COMPLETED
        print(f"[Saga {ctx.saga_id}] 완료!")
        return ctx

    def _step_create_order(self, ctx, user_id, product_id, quantity):
        try:
            order_id = self.order_svc.create(user_id, product_id, quantity)
            ctx.order_id = order_id
            ctx.state    = SagaState.ORDER_CREATED
            print(f"[Saga] 주문 생성 완료: {order_id}")
        except Exception as e:
            print(f"[Saga] 주문 생성 실패: {e}")
            ctx.state = SagaState.FAILED
        return ctx

    def _step_process_payment(self, ctx, amount):
        try:
            payment_id = self.payment_svc.charge(ctx.order_id, amount)
            ctx.payment_id = payment_id
            ctx.state      = SagaState.PAYMENT_DONE
            print(f"[Saga] 결제 완료: {payment_id}")
        except Exception as e:
            print(f"[Saga] 결제 실패: {e} → 주문 취소 보상 시작")
            self._compensate_order(ctx)
            ctx.state = SagaState.FAILED
        return ctx

    def _step_reserve_stock(self, ctx, product_id, quantity):
        try:
            self.inventory_svc.reserve(ctx.order_id, product_id, quantity)
            ctx.state = SagaState.STOCK_RESERVED
            print(f"[Saga] 재고 예약 완료")
        except Exception as e:
            print(f"[Saga] 재고 예약 실패: {e} → 결제/주문 보상 시작")
            self._compensate_payment(ctx)
            self._compensate_order(ctx)
            ctx.state = SagaState.FAILED
        return ctx

    # ── 보상 트랜잭션들 ──────────────────────────────
    def _compensate_payment(self, ctx):
        if ctx.payment_id:
            try:
                self.payment_svc.refund(ctx.payment_id)
                print(f"[Saga] 결제 환불 완료: {ctx.payment_id}")
                ctx.state = SagaState.COMPENSATING_PAYMENT
            except Exception as e:
                print(f"[Saga] 환불 실패 (수동 처리 필요): {e}")

    def _compensate_order(self, ctx):
        if ctx.order_id:
            try:
                self.order_svc.cancel(ctx.order_id)
                print(f"[Saga] 주문 취소 완료: {ctx.order_id}")
                ctx.state = SagaState.COMPENSATING_ORDER
            except Exception as e:
                print(f"[Saga] 주문 취소 실패 (수동 처리 필요): {e}")


# ── 서비스 스텁 ──────────────────────────────────────
class MockOrderService:
    def create(self, user_id, product_id, quantity):
        return f"order_{uuid.uuid4().hex[:8]}"
    def cancel(self, order_id):
        print(f"  OrderService: {order_id} 취소됨")

class MockPaymentService:
    def __init__(self, fail=False):
        self.fail = fail
    def charge(self, order_id, amount):
        if self.fail:
            raise Exception("잔액 부족")
        return f"payment_{uuid.uuid4().hex[:8]}"
    def refund(self, payment_id):
        print(f"  PaymentService: {payment_id} 환불됨")

class MockInventoryService:
    def __init__(self, fail=False):
        self.fail = fail
    def reserve(self, order_id, product_id, quantity):
        if self.fail:
            raise Exception("재고 부족")
        print(f"  InventoryService: {product_id} x{quantity} 예약됨")


# ── 실행 ──────────────────────────────────────────────
print("=== 정상 케이스 ===")
saga = OrderSagaOrchestrator(
    MockOrderService(),
    MockPaymentService(fail=False),
    MockInventoryService(fail=False)
)
result = saga.execute("user1", "prod_A", 2, 50000.0)
print(f"최종 상태: {result.state}\n")

print("=== 재고 부족 케이스 (보상 트랜잭션 발동) ===")
saga2 = OrderSagaOrchestrator(
    MockOrderService(),
    MockPaymentService(fail=False),
    MockInventoryService(fail=True)
)
result2 = saga2.execute("user2", "prod_B", 10, 200000.0)
print(f"최종 상태: {result2.state}")
```

출력 예시:
```
=== 정상 케이스 ===
[Saga abc123] 시작
[Saga] 주문 생성 완료: order_f1a2b3c4
[Saga] 결제 완료: payment_9e8d7c6b
  InventoryService: prod_A x2 예약됨
[Saga] 재고 예약 완료
[Saga abc123] 완료!
최종 상태: SagaState.COMPLETED

=== 재고 부족 케이스 (보상 트랜잭션 발동) ===
[Saga] 주문 생성 완료: order_a1b2c3d4
[Saga] 결제 완료: payment_5f6g7h8i
[Saga] 재고 예약 실패: 재고 부족 → 결제/주문 보상 시작
  PaymentService: payment_5f6g7h8i 환불됨
  OrderService: order_a1b2c3d4 취소됨
최종 상태: SagaState.FAILED
```

---

## 주의사항 및 팁

### 1. 멱등성(Idempotency) 설계

네트워크 장애로 메시지가 중복 전달될 수 있다. 모든 보상 트랜잭션과 로컬 트랜잭션은 **멱등하게** 설계해야 한다. 중복 실행해도 결과가 같아야 한다. UUID 기반 이벤트 ID 체크, `ON CONFLICT DO NOTHING`, 처리 기록 테이블 등을 활용한다.

### 2. Outbox 패턴 — 이중 쓰기 문제 해결

이벤트 발행과 DB 저장을 단일 트랜잭션으로 묶지 않으면 이중 쓰기 문제가 발생한다:
- DB 저장 성공 + 이벤트 발행 실패 → 다음 서비스가 모름
- 이벤트 발행 성공 + DB 저장 실패 → 불일치

**Outbox 패턴**: DB 저장과 동일한 트랜잭션으로 `outbox` 테이블에 이벤트를 기록하고, 별도의 Outbox 폴러가 이를 메시지 브로커(Kafka 등)에 발행한다.

### 3. 격리 수준 부재 대응 — 카운터메저

Saga는 격리성을 포기하므로 다음 문제가 생길 수 있다:
- **Dirty Read**: 다른 트랜잭션이 아직 진행 중인 Saga의 중간 상태를 읽음
- **Lost Update**: 동시 Saga가 서로의 업데이트를 덮어씀

대응책:
- **낙관적 잠금(Optimistic Locking)**: 버전 번호(version) 필드로 충돌 감지
- **Semantic Lock**: Saga 진행 중 레코드를 "PENDING" 상태로 표시, 다른 Saga가 PENDING 레코드 건드리지 못하도록
- **Commutative Update**: 순서 무관한 연산 설계 (덧셈/뺄셈 등)

### 4. Saga State Store의 영속성

오케스트레이션 Saga는 Saga 상태를 DB에 반드시 저장해야 한다. 오케스트레이터 재시작 후에도 중단된 Saga를 복구할 수 있어야 한다. 실무에서는 **Temporal**, **Conductor**, **Axon Framework** 같은 워크플로 엔진을 활용한다.

### 5. 언제 Saga를 쓰지 말아야 하나

Saga는 복잡성을 수반하므로 다음 경우에는 단순한 2PC나 단일 DB 트랜잭션을 고려하자:
- 서비스가 2~3개이고 모두 동일 DB 사용 가능한 경우
- 즉각적인 강한 일관성이 비즈니스 요구사항인 경우 (금융 원장 등)
- 팀 규모가 작고 마이크로서비스 전환이 불필요한 경우

---

## 정리

Saga 패턴은 마이크로서비스 아키텍처에서 2PC의 한계를 극복하는 표준 해결책이다. 코레오그래피는 단순하고 결합도가 낮지만 전체 흐름 추적이 어렵고, 오케스트레이션은 가시성이 높지만 오케스트레이터가 복잡해진다. 멱등성, Outbox 패턴, 낙관적 잠금을 함께 설계해야 실제 프로덕션 환경에서 안전하다.

## 참고 자료
- [Pattern: Saga — microservices.io (Chris Richardson)](https://microservices.io/patterns/data/saga.html)
- [Saga Design Pattern — Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/patterns/saga)
- [Saga Pattern in Distributed Systems — Orkes](https://orkes.io/blog/saga-pattern-in-distributed-systems/)
- [Saga Choreography Pattern — AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/saga-choreography.html)
