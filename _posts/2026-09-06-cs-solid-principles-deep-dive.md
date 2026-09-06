---
layout: post
title: "SOLID 원칙 완전 정복: 실제 코드로 이해하는 객체지향 설계의 다섯 가지 원칙"
date: 2026-09-06
categories: [cs, computer-science]
tags: [solid, oop, design-principles, single-responsibility, open-closed, liskov, interface-segregation, dependency-inversion]
---

## SOLID 원칙이란

SOLID는 Robert C. Martin(Uncle Bob)이 2000년대 초에 정립한 **객체지향 설계 5대 원칙**의 두문자어입니다.

- **S** — Single Responsibility Principle (단일 책임 원칙)
- **O** — Open/Closed Principle (개방/폐쇄 원칙)
- **L** — Liskov Substitution Principle (리스코프 치환 원칙)
- **I** — Interface Segregation Principle (인터페이스 분리 원칙)
- **D** — Dependency Inversion Principle (의존성 역전 원칙)

이 원칙들은 "변경에 강하고, 확장이 쉬우며, 테스트하기 좋은" 소프트웨어를 만들기 위한 설계 지침입니다. 각 원칙을 위반한 코드와 준수한 코드를 나란히 비교하며 살펴보겠습니다.

## S — 단일 책임 원칙 (SRP)

> "클래스는 변경되어야 할 이유가 오직 하나여야 한다."

**위반 예시**: 하나의 클래스가 데이터 처리, 포맷팅, 저장, 알림을 모두 담당합니다.

```python
# SRP 위반: Order 클래스가 너무 많은 책임을 가짐
class Order:
    def __init__(self, items, customer_email):
        self.items = items
        self.customer_email = customer_email

    def calculate_total(self):
        return sum(item['price'] * item['qty'] for item in self.items)

    def format_receipt(self):
        # 책임 2: 영수증 포맷팅
        lines = [f"{i['name']}: {i['price'] * i['qty']:,}원" for i in self.items]
        return "\n".join(lines) + f"\n합계: {self.calculate_total():,}원"

    def save_to_db(self, conn):
        # 책임 3: 데이터베이스 저장
        conn.execute("INSERT INTO orders ...", (self.calculate_total(),))

    def send_confirmation_email(self, smtp):
        # 책임 4: 이메일 발송
        smtp.send(self.customer_email, "주문 확인", self.format_receipt())
```

이 클래스는 영수증 형식이 바뀌거나, DB 스키마가 변하거나, 이메일 라이브러리가 교체될 때마다 변경됩니다. **변경 이유가 4가지**입니다.

```python
# SRP 준수: 각 책임을 분리
class Order:
    def __init__(self, items):
        self.items = items

    def calculate_total(self):
        return sum(item['price'] * item['qty'] for item in self.items)

class ReceiptFormatter:
    def format(self, order: Order) -> str:
        lines = [f"{i['name']}: {i['price'] * i['qty']:,}원" for i in order.items]
        return "\n".join(lines) + f"\n합계: {order.calculate_total():,}원"

class OrderRepository:
    def __init__(self, conn):
        self._conn = conn

    def save(self, order: Order):
        self._conn.execute("INSERT INTO orders ...", (order.calculate_total(),))

class OrderNotifier:
    def __init__(self, smtp):
        self._smtp = smtp

    def send_confirmation(self, email: str, receipt: str):
        self._smtp.send(email, "주문 확인", receipt)

# 조립: 각 클래스는 자신의 책임만 가짐
order = Order(items)
receipt = ReceiptFormatter().format(order)
OrderRepository(conn).save(order)
OrderNotifier(smtp).send_confirmation(customer_email, receipt)
```

각 클래스는 변경 이유가 하나뿐이고 독립적으로 테스트할 수 있습니다.

## O — 개방/폐쇄 원칙 (OCP)

> "소프트웨어 요소는 확장에는 열려 있어야 하고, 수정에는 닫혀 있어야 한다."

새로운 기능을 추가할 때 기존 코드를 수정하지 않고 **확장**만으로 해결해야 합니다.

```python
# OCP 위반: 새 결제 수단이 추가될 때마다 process_payment를 수정해야 함
class PaymentProcessor:
    def process_payment(self, method: str, amount: float):
        if method == 'credit_card':
            print(f"신용카드로 {amount}원 결제")
        elif method == 'bank_transfer':
            print(f"계좌이체로 {amount}원 결제")
        elif method == 'crypto':          # 새 수단 추가 시 여기를 수정
            print(f"암호화폐로 {amount}원 결제")
        # 간편결제가 추가되면 또 수정...
```

```python
# OCP 준수: 추상화로 확장점을 열어 둠
from abc import ABC, abstractmethod

class PaymentMethod(ABC):
    @abstractmethod
    def pay(self, amount: float) -> bool:
        pass

class CreditCardPayment(PaymentMethod):
    def pay(self, amount: float) -> bool:
        print(f"신용카드로 {amount:,}원 결제 완료")
        return True

class BankTransferPayment(PaymentMethod):
    def pay(self, amount: float) -> bool:
        print(f"계좌이체로 {amount:,}원 결제 완료")
        return True

# 새 결제 수단 추가 시 기존 코드 무수정
class KakaoPayPayment(PaymentMethod):
    def pay(self, amount: float) -> bool:
        print(f"카카오페이로 {amount:,}원 결제 완료")
        return True

class PaymentProcessor:
    def process(self, method: PaymentMethod, amount: float) -> bool:
        return method.pay(amount)  # 이 코드는 절대 수정되지 않음

# 사용
processor = PaymentProcessor()
processor.process(CreditCardPayment(), 50_000)
processor.process(KakaoPayPayment(), 30_000)  # 신규 추가, 기존 코드 무수정
```

**전략 패턴(Strategy Pattern)**이 OCP의 전형적인 구현입니다.

## L — 리스코프 치환 원칙 (LSP)

> "프로그램에서 서브타입의 인스턴스는 부모 타입으로 교체해도 프로그램이 올바르게 동작해야 한다."

Barbara Liskov가 1987년에 정의한 이 원칙은 **상속 계층이 올바른 IS-A 관계**를 나타내야 함을 의미합니다.

```python
# LSP 위반: Rectangle → Square 상속이 계약을 깨뜨림
class Rectangle:
    def __init__(self, width, height):
        self._width = width
        self._height = height

    def set_width(self, w): self._width = w
    def set_height(self, h): self._height = h
    def area(self): return self._width * self._height

class Square(Rectangle):
    def set_width(self, w):
        # 정사각형은 너비와 높이가 항상 같아야 함
        self._width = w
        self._height = w  # 부모의 계약을 어김!

    def set_height(self, h):
        self._width = h
        self._height = h

def assert_area(rect: Rectangle):
    rect.set_width(5)
    rect.set_height(10)
    expected = 50
    actual = rect.area()
    assert actual == expected, f"기대: {expected}, 실제: {actual}"

rect = Rectangle(2, 3)
assert_area(rect)   # 통과

sq = Square(4, 4)
assert_area(sq)     # 실패! area = 100 (10*10), 기대는 50 → LSP 위반
```

```python
# LSP 준수: 공통 조상으로 리팩토링
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self) -> float:
        pass

class Rectangle(Shape):
    def __init__(self, width: float, height: float):
        self._width = width
        self._height = height

    def area(self) -> float:
        return self._width * self._height

class Square(Shape):
    def __init__(self, side: float):
        self._side = side

    def area(self) -> float:
        return self._side ** 2

# 이제 Rectangle과 Square는 독립적이며 LSP를 만족함
def print_area(shape: Shape):
    print(f"넓이: {shape.area()}")

print_area(Rectangle(5, 10))  # 넓이: 50
print_area(Square(7))          # 넓이: 49
```

LSP 위반의 대표적 신호: 서브클래스에서 부모 메서드를 오버라이드하며 예외를 던지거나, 사전 조건을 강화하거나, 사후 조건을 약화시키는 경우입니다.

## I — 인터페이스 분리 원칙 (ISP)

> "클라이언트는 자신이 사용하지 않는 메서드에 의존하도록 강요받아서는 안 된다."

큰 인터페이스 하나보다 **작고 명확한 인터페이스 여러 개**가 낫습니다.

```python
# ISP 위반: 모든 클래스가 하나의 거대한 인터페이스를 구현해야 함
from abc import ABC, abstractmethod

class Worker(ABC):
    @abstractmethod
    def work(self): pass

    @abstractmethod
    def eat(self): pass   # 로봇에게는 불필요!

    @abstractmethod
    def sleep(self): pass  # 로봇에게는 불필요!

class HumanWorker(Worker):
    def work(self): print("인간이 일합니다")
    def eat(self): print("인간이 식사합니다")
    def sleep(self): print("인간이 잠을 잡니다")

class RobotWorker(Worker):
    def work(self): print("로봇이 일합니다")
    def eat(self): raise NotImplementedError("로봇은 식사하지 않습니다")  # 위반!
    def sleep(self): raise NotImplementedError("로봇은 자지 않습니다")    # 위반!
```

```python
# ISP 준수: 역할별로 인터페이스 분리
class Workable(ABC):
    @abstractmethod
    def work(self): pass

class Eatable(ABC):
    @abstractmethod
    def eat(self): pass

class Sleepable(ABC):
    @abstractmethod
    def sleep(self): pass

class HumanWorker(Workable, Eatable, Sleepable):
    def work(self): print("인간이 일합니다")
    def eat(self): print("인간이 식사합니다")
    def sleep(self): print("인간이 잠을 잡니다")

class RobotWorker(Workable):  # 필요한 인터페이스만 구현
    def work(self): print("로봇이 일합니다")

def manage_workers(workers: list[Workable]):
    for w in workers:
        w.work()

# 로봇과 인간 모두 Workable이므로 함께 사용 가능
team = [HumanWorker(), RobotWorker()]
manage_workers(team)  # 정상 동작
```

실전에서는 Java의 `Serializable`, `Comparable`, `Runnable`, Python의 `Iterable`, `Sized`, `Callable`처럼 **단일 목적 인터페이스**가 ISP의 좋은 예입니다.

## D — 의존성 역전 원칙 (DIP)

> "고수준 모듈은 저수준 모듈에 의존해서는 안 된다. 둘 다 추상화에 의존해야 한다."

DIP는 의존성의 방향을 뒤집어 **추상화에 의존**하게 만듭니다.

```python
# DIP 위반: 고수준 모듈 UserService가 저수준 구현 MySQL에 직접 의존
class MySQLDatabase:
    def save_user(self, user: dict):
        print(f"MySQL에 저장: {user}")

class UserService:
    def __init__(self):
        self._db = MySQLDatabase()  # 구체적 구현에 직접 의존!

    def register(self, name: str, email: str):
        user = {'name': name, 'email': email}
        self._db.save_user(user)
        # MySQL을 PostgreSQL로 바꾸려면 UserService 코드를 수정해야 함
```

```python
# DIP 준수: 추상화를 통한 의존성 역전
from abc import ABC, abstractmethod

class UserRepository(ABC):
    """추상화 계층 — 고수준과 저수준 모두 이것에 의존"""
    @abstractmethod
    def save(self, user: dict) -> None: pass

    @abstractmethod
    def find_by_email(self, email: str) -> dict | None: pass

# 저수준 모듈들 (구현체)
class MySQLUserRepository(UserRepository):
    def save(self, user: dict) -> None:
        print(f"MySQL에 저장: {user}")

    def find_by_email(self, email: str) -> dict | None:
        print(f"MySQL에서 {email} 조회")
        return None

class InMemoryUserRepository(UserRepository):
    def __init__(self):
        self._store = {}

    def save(self, user: dict) -> None:
        self._store[user['email']] = user

    def find_by_email(self, email: str) -> dict | None:
        return self._store.get(email)

# 고수준 모듈 — 추상화에만 의존
class UserService:
    def __init__(self, repo: UserRepository):  # 의존성 주입(DI)
        self._repo = repo

    def register(self, name: str, email: str) -> bool:
        if self._repo.find_by_email(email):
            raise ValueError(f"{email}은 이미 등록된 이메일입니다")
        self._repo.save({'name': name, 'email': email})
        return True

# 프로덕션 환경
prod_service = UserService(MySQLUserRepository())
prod_service.register("Alice", "alice@example.com")

# 테스트 환경: DB 없이도 완벽한 단위 테스트 가능
test_service = UserService(InMemoryUserRepository())
test_service.register("Bob", "bob@example.com")
```

### 의존성 주입 컨테이너

대규모 애플리케이션에서는 의존성을 수동으로 조립하는 것이 복잡해집니다. **DI 컨테이너**가 이를 자동화합니다.

```python
# 간단한 DI 컨테이너 구현
class DIContainer:
    def __init__(self):
        self._bindings = {}
        self._singletons = {}

    def bind(self, abstract, concrete, singleton=False):
        self._bindings[abstract] = (concrete, singleton)

    def make(self, abstract):
        if abstract not in self._bindings:
            raise KeyError(f"{abstract} is not registered")
        concrete, is_singleton = self._bindings[abstract]
        if is_singleton:
            if abstract not in self._singletons:
                self._singletons[abstract] = concrete()
            return self._singletons[abstract]
        return concrete()

# 설정
container = DIContainer()
container.bind(UserRepository, MySQLUserRepository, singleton=True)
container.bind(UserService, lambda: UserService(container.make(UserRepository)))

# 사용
service = container.make(UserService)
service.register("Charlie", "charlie@example.com")
```

실제 프레임워크에서는 Python의 `dependency-injector`, Java의 Spring, Kotlin의 Hilt 등이 이 역할을 합니다.

## SOLID 원칙 간의 관계

SOLID 원칙은 서로 독립적이지 않습니다.

- **SRP + OCP**: 책임이 분리된 클래스는 자연스럽게 확장에 열려 있습니다.
- **OCP + DIP**: 추상화에 의존할 때 OCP를 쉽게 달성할 수 있습니다.
- **LSP + ISP**: 작고 명확한 인터페이스일수록 LSP를 만족하기 쉽습니다.
- **DIP**: 나머지 네 원칙을 가능하게 하는 **기반 원칙**입니다.

```
DIP (추상화에 의존)
    ↓ 가능하게 함
OCP + ISP (확장 가능한 작은 계약)
    ↓ 가능하게 함
SRP + LSP (단일 책임, 안전한 상속)
```

## 언제 SOLID를 적용하지 않아야 하는가

SOLID는 도구이지 목표가 아닙니다. **과도한 추상화**는 오히려 코드를 이해하기 어렵게 만듭니다.

- **프로토타입/MVP**: 요구사항이 불명확할 때는 YAGNI(You Aren't Gonna Need It)가 우선입니다.
- **작은 스크립트**: 50줄짜리 유틸리티 스크립트에 DI 컨테이너는 과잉입니다.
- **성능이 최우선인 경우**: 추상화 레이어는 경우에 따라 인라인 최적화를 막습니다.

"코드가 두 번 이상 변경 이유가 생겼을 때" SOLID 원칙을 적용하는 **리팩토링 트리거**로 사용하는 것이 실용적입니다.

## 마무리

SOLID 원칙은 단순한 이론이 아니라 실제 소프트웨어 유지보수성을 높이는 설계 지침입니다. SRP로 변경 이유를 하나로 줄이고, OCP로 기존 코드 수정 없이 확장하며, LSP로 안전한 상속 계층을 설계하고, ISP로 인터페이스를 날카롭게 다듬고, DIP로 테스트 가능하고 유연한 의존성 구조를 만들 수 있습니다. 이 원칙들을 내면화하면 "언제 리팩토링이 필요한가"를 코드 냄새(Code Smell)로 직관적으로 감지하게 됩니다.

## 참고 자료
- [Robert C. Martin — Design Principles and Design Patterns (원본 논문)](https://fi.ort.edu.uy/innovaportal/file/2032/1/design_principles.pdf)
- [Martin Fowler — Refactoring: Improving the Design of Existing Code](https://refactoring.com/)
- [SourceMaking — SOLID Principles](https://sourcemaking.com/design_patterns)
- [Wikipedia — SOLID](https://en.wikipedia.org/wiki/SOLID)
