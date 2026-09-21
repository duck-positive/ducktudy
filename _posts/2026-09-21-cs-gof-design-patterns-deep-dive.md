---
layout: post
title: "GoF 디자인 패턴 심화: 23가지 패턴의 원리와 실전 적용"
date: 2026-09-21
categories: [cs, computer-science]
tags: [design-patterns, gof, oop, creational, structural, behavioral, java, python]
---

## 개념 설명

1994년 Erich Gamma, Richard Helm, Ralph Johnson, John Vlissides — 이른바 **GoF(Gang of Four)**가 출간한 *Design Patterns: Elements of Reusable Object-Oriented Software*는 소프트웨어 공학 역사상 가장 영향력 있는 책 중 하나다. 이 책은 객체지향 설계에서 반복적으로 등장하는 문제를 23개의 패턴으로 정리하여, 개발자들이 공통 어휘로 설계를 논의할 수 있게 했다.

GoF 패턴은 세 범주로 나뉜다.

- **생성 패턴(Creational)**: 객체 생성 메커니즘을 추상화. Singleton, Factory Method, Abstract Factory, Builder, Prototype
- **구조 패턴(Structural)**: 클래스·객체를 합성하여 더 큰 구조를 형성. Adapter, Bridge, Composite, Decorator, Facade, Flyweight, Proxy
- **행동 패턴(Behavioral)**: 알고리즘과 객체 간 책임 분배. Chain of Responsibility, Command, Iterator, Mediator, Memento, Observer, State, Strategy, Template Method, Visitor, Interpreter

패턴은 구현 코드가 아니라 **설계 템플릿**이다. 언어에 무관하게 적용 가능하며, 특정 상황의 구조적 해결책을 표현한다.

---

## 왜 필요한가

좋은 설계는 **변경에 강하고** 재사용 가능해야 한다. 패턴 없이 코드를 작성하면 다음 문제가 반복된다.

- **강한 결합**: 구체 클래스에 직접 의존하면 교체나 테스트가 어렵다
- **중복 로직**: 비슷한 생성·합성 코드가 여러 곳에 분산된다
- **변경 파급 효과**: 한 곳의 수정이 예상치 못한 곳까지 퍼진다

GoF 패턴은 수십 년간 검증된 해결책을 제공하므로, 바퀴를 다시 발명하지 않고 **팀 간 공통 언어**로 빠르게 소통할 수 있다.

---

## 실전 구현 예제

### 예제 1 — Builder 패턴 (Java)

복잡한 객체를 단계별로 구성한다. 생성자에 인자가 많을 때 가독성이 크게 개선된다.

```java
// Product
public class HttpRequest {
    private final String method;
    private final String url;
    private final Map<String, String> headers;
    private final String body;
    private final int timeoutMs;

    private HttpRequest(Builder builder) {
        this.method    = builder.method;
        this.url       = builder.url;
        this.headers   = Collections.unmodifiableMap(builder.headers);
        this.body      = builder.body;
        this.timeoutMs = builder.timeoutMs;
    }

    // Builder
    public static class Builder {
        private String method = "GET";
        private String url;
        private Map<String, String> headers = new HashMap<>();
        private String body;
        private int timeoutMs = 5000;

        public Builder url(String url)                         { this.url = url; return this; }
        public Builder method(String method)                   { this.method = method; return this; }
        public Builder header(String key, String value)        { headers.put(key, value); return this; }
        public Builder body(String body)                       { this.body = body; return this; }
        public Builder timeout(int ms)                         { this.timeoutMs = ms; return this; }

        public HttpRequest build() {
            if (url == null || url.isBlank()) throw new IllegalStateException("url is required");
            return new HttpRequest(this);
        }
    }
}

// 사용
HttpRequest req = new HttpRequest.Builder()
    .url("https://api.example.com/users")
    .method("POST")
    .header("Content-Type", "application/json")
    .header("Authorization", "Bearer token123")
    .body("{\"name\":\"Alice\"}")
    .timeout(3000)
    .build();
```

Builder의 핵심은 **불변 객체(immutable object)** 생성이다. `build()` 시점에 유효성 검사를 집중시켜 잘못된 상태를 조기에 차단할 수 있다.

---

### 예제 2 — Observer 패턴 (Python)

이벤트가 발생하면 의존 객체들에게 자동으로 알린다. GUI, 이벤트 시스템, 실시간 피드에 폭넓게 쓰인다.

```python
from __future__ import annotations
from abc import ABC, abstractmethod
from typing import List


class Observer(ABC):
    @abstractmethod
    def update(self, event: str, data: dict) -> None: ...


class Subject:
    def __init__(self) -> None:
        self._observers: List[Observer] = []

    def subscribe(self, observer: Observer) -> None:
        self._observers.append(observer)

    def unsubscribe(self, observer: Observer) -> None:
        self._observers.remove(observer)

    def notify(self, event: str, data: dict) -> None:
        for observer in self._observers:
            observer.update(event, data)


class OrderService(Subject):
    def place_order(self, order_id: str, amount: float) -> None:
        print(f"[OrderService] 주문 {order_id} 접수, 금액: {amount}")
        self.notify("ORDER_PLACED", {"order_id": order_id, "amount": amount})


class EmailNotifier(Observer):
    def update(self, event: str, data: dict) -> None:
        if event == "ORDER_PLACED":
            print(f"[Email] 주문 확인 메일 발송 → 주문번호: {data['order_id']}")


class InventoryService(Observer):
    def update(self, event: str, data: dict) -> None:
        if event == "ORDER_PLACED":
            print(f"[Inventory] 재고 차감 처리 → 주문번호: {data['order_id']}")


class AnalyticsService(Observer):
    def update(self, event: str, data: dict) -> None:
        print(f"[Analytics] 이벤트 기록: {event}, 데이터: {data}")


# 실행
order_service = OrderService()
order_service.subscribe(EmailNotifier())
order_service.subscribe(InventoryService())
order_service.subscribe(AnalyticsService())

order_service.place_order("ORD-001", 59_000)
```

```
출력:
[OrderService] 주문 ORD-001 접수, 금액: 59000
[Email] 주문 확인 메일 발송 → 주문번호: ORD-001
[Inventory] 재고 차감 처리 → 주문번호: ORD-001
[Analytics] 이벤트 기록: ORDER_PLACED, 데이터: {'order_id': 'ORD-001', 'amount': 59000}
```

OrderService는 EmailNotifier, InventoryService, AnalyticsService를 **직접 알지 못한다**. 새 구독자를 추가해도 OrderService 코드를 수정할 필요 없다 — **개방-폐쇄 원칙(OCP)**의 완벽한 구현이다.

---

### 예제 3 — Strategy 패턴 (Java) — 알고리즘 교체

실행 중에 알고리즘 계열을 교체할 수 있다. 결제 방법, 정렬 전략, 압축 알고리즘 선택 등에 유용하다.

```java
@FunctionalInterface
interface DiscountStrategy {
    double apply(double price);
}

class PricingService {
    private DiscountStrategy strategy;

    public PricingService(DiscountStrategy strategy) {
        this.strategy = strategy;
    }

    public void setStrategy(DiscountStrategy strategy) {
        this.strategy = strategy;
    }

    public double calculate(double price) {
        return strategy.apply(price);
    }
}

// 사용 — Java 람다로 전략 구현
DiscountStrategy vipDiscount      = price -> price * 0.7;      // 30% 할인
DiscountStrategy seasonalDiscount = price -> price - 5_000;    // 5000원 할인
DiscountStrategy noDiscount       = price -> price;

PricingService service = new PricingService(vipDiscount);
System.out.println(service.calculate(50_000));   // 35000.0

service.setStrategy(seasonalDiscount);
System.out.println(service.calculate(50_000));   // 45000.0
```

Java 8+ 에서는 `@FunctionalInterface`와 람다로 Strategy를 별도 클래스 없이 구현할 수 있어 코드가 크게 간결해진다.

---

## 패턴 선택 가이드

| 상황 | 추천 패턴 |
|---|---|
| 복잡한 객체를 단계별로 구성해야 할 때 | Builder |
| 어떤 클래스의 인스턴스를 생성할지 서브클래스가 결정해야 할 때 | Factory Method |
| 인터페이스 불일치를 해결해야 할 때 | Adapter |
| 동적으로 책임을 추가·제거해야 할 때 | Decorator |
| 하나의 전역 인스턴스가 필요할 때 | Singleton |
| 이벤트 기반 1:N 통보가 필요할 때 | Observer |
| 알고리즘을 런타임에 교체해야 할 때 | Strategy |
| 요청을 캡슐화하고 실행 취소가 필요할 때 | Command |

---

## 주의사항과 팁

**1. 오버엔지니어링 경계**  
모든 코드에 패턴을 적용하면 오히려 복잡도가 증가한다. "문제가 있을 때만" 패턴을 꺼내야 한다. 세 곳 이상에서 동일한 구조 문제를 발견하면 패턴 도입을 고려하라.

**2. Singleton의 함정**  
Singleton은 전역 상태를 만들어 단위 테스트를 어렵게 한다. 현대 개발에서는 DI 컨테이너(Spring, Hilt)를 통해 Singleton 스코프를 관리하는 것이 권장된다.

**3. Decorator vs Inheritance**  
상속 대신 Decorator를 쓰면 조합 폭발을 피할 수 있다. 예를 들어 `BufferedReader`는 `FileReader`에 버퍼링을 추가하는 Decorator다.

**4. 패턴은 언어에 따라 달라진다**  
일부 패턴(Iterator, Singleton, Observer)은 현대 언어에 내장 기능으로 흡수되었다. Python의 `__iter__`, Kotlin의 `object`, Rx 라이브러리가 그 예다.

**5. 공통 언어로 활용하라**  
코드 리뷰나 설계 토론에서 "이 부분은 Observer 패턴으로 분리하면 어떨까요?"처럼 패턴 이름을 쓰면 긴 설명 없이 의도를 전달할 수 있다.

---

## 참고 자료
- [GoF Pattern Reference — gofpattern.com](https://www.gofpattern.com/)
- [Gang of Four Design Patterns — Spring Framework Guru](https://springframework.guru/gang-of-four-design-patterns/)
- [GoF Design Patterns — GeeksforGeeks](https://www.geeksforgeeks.org/system-design/gang-of-four-gof-design-patterns/)
- [Gang of Four Patterns — Enterprise Architect User Guide (Sparx Systems)](https://sparxsystems.com/enterprise_architect_user_guide/17.1/modeling_domains/gof_patterns.html)
