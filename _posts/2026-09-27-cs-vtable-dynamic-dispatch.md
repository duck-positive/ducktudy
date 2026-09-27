---
layout: post
title: "vtable과 동적 디스패치: 다형성의 구현 원리와 최적화"
date: 2026-09-27
categories: [cs, computer-science]
tags: [vtable, dynamic-dispatch, polymorphism, c++, oop, devirtualization, virtual-functions, interface]
---

## 다형성과 동적 디스패치의 필요성

객체지향 프로그래밍의 핵심은 **다형성(polymorphism)**입니다. 같은 인터페이스를 통해 서로 다른 동작을 실행할 수 있어야 합니다. C++에서 `Animal* a = new Dog(); a->speak();`를 호출했을 때, 컴파일 타임에는 `a`가 `Dog`인지 `Cat`인지 알 수 없습니다. 런타임에 실제 객체 타입을 확인하고 올바른 함수를 호출하는 메커니즘이 필요합니다. 이것이 **동적 디스패치(dynamic dispatch)**입니다.

C++ 표준은 동적 디스패치의 동작만 명세하며 구현 방식을 강제하지 않습니다. 그러나 GCC, Clang, MSVC를 포함한 사실상 모든 C++ 컴파일러가 **vtable(virtual function table)**을 사용하여 이를 구현합니다.

---

## vtable의 구조

### 컴파일러가 생성하는 것들

`virtual` 함수를 포함하는 클래스가 있을 때, 컴파일러는 두 가지를 자동으로 생성합니다.

1. **vtable**: 클래스마다 하나씩 생성되는 함수 포인터 배열. 프로그램의 읽기 전용 데이터 세그먼트에 위치합니다.
2. **vptr**: 각 객체의 첫 번째 필드로 숨겨지는 포인터. 해당 클래스의 vtable을 가리킵니다.

```
┌─────────────────────┐
│   Animal 객체       │
│   vptr ─────────────┼──► Animal's vtable
│   name (멤버)       │    [0]: &Animal::speak
└─────────────────────┘    [1]: &Animal::move

┌─────────────────────┐
│   Dog 객체          │
│   vptr ─────────────┼──► Dog's vtable
│   name (멤버)       │    [0]: &Dog::speak   ← 오버라이드됨
│   breed (멤버)      │    [1]: &Animal::move ← 상속됨
└─────────────────────┘
```

가상 함수 호출 `a->speak()`는 다음 단계로 이루어집니다:
1. `a`가 가리키는 객체에서 `vptr`를 읽는다 (메모리 역참조 1회)
2. `vptr`가 가리키는 vtable에서 `speak`의 슬롯 오프셋 위치의 함수 포인터를 읽는다 (메모리 역참조 1회)
3. 그 포인터로 함수를 간접 호출한다

어셈블리 수준에서는 대략 다음과 같습니다:
```asm
mov    rax, [rdi]        ; rdi = this 포인터, rax = vptr
call   [rax + 0]         ; vtable[0] 호출 (speak의 슬롯)
```

---

## 코드 예제 1: C++ vtable 동작 관찰

```cpp
#include <iostream>
#include <cstdint>

class Animal {
public:
    virtual void speak() { std::cout << "...\n"; }
    virtual void move()  { std::cout << "moving\n"; }
    virtual ~Animal() = default;
    int id = 0;
};

class Dog : public Animal {
public:
    void speak() override { std::cout << "Woof!\n"; }
    // move()는 오버라이드하지 않음 → Animal::move가 vtable에 남음
    int breed_id = 1;
};

class Cat : public Animal {
public:
    void speak() override { std::cout << "Meow!\n"; }
    void move()  override { std::cout << "tiptoeing\n"; }
};

void make_noise(Animal* a) {
    // 컴파일 시점에 a의 실제 타입을 모름
    // 런타임에 vtable을 통해 올바른 speak() 호출
    a->speak();
}

// vptr 직접 탐색 (교육 목적, UB를 포함하므로 프로덕션 코드에서 사용 금지)
void inspect_vptr(Animal* obj) {
    // 객체의 첫 8바이트가 vptr
    void** vptr = *reinterpret_cast<void***>(obj);
    std::cout << "vtable 주소: " << vptr << "\n";
    std::cout << "vtable[0] (speak): " << vptr[0] << "\n";
    std::cout << "vtable[1] (move):  " << vptr[1] << "\n";
}

int main() {
    Dog d;
    Cat c;
    Animal* animals[] = {&d, &c};

    for (auto* a : animals) {
        make_noise(a);   // 런타임에 Dog::speak 또는 Cat::speak 호출
    }

    std::cout << "\n--- vtable 탐색 ---\n";
    std::cout << "Dog:\n";
    inspect_vptr(&d);
    std::cout << "Cat:\n";
    inspect_vptr(&c);

    // sizeof 확인: vptr 때문에 Animal이 8바이트 커짐
    std::cout << "\nsizeof(Animal) = " << sizeof(Animal) << "\n"; // 16 (vptr 8 + id 4 + padding 4)
    std::cout << "sizeof(Dog) = " << sizeof(Dog) << "\n";         // 24
    return 0;
}
```

컴파일하고 실행하면 `Dog`와 `Cat`의 vtable 주소가 서로 다른 것을 확인할 수 있습니다. `Dog::speak`와 `Cat::speak`가 vtable의 같은 슬롯(인덱스 0)에 위치하지만, 각 vtable이 다른 함수 포인터를 담고 있습니다.

---

## 코드 예제 2: C로 vtable 직접 구현

vtable의 본질은 함수 포인터 테이블입니다. C에서 수동으로 구현하면 그 원리가 명확해집니다.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

// 인터페이스 정의 (vtable 구조체)
typedef struct ShapeVTable {
    double (*area)(const void* self);
    double (*perimeter)(const void* self);
    void   (*print)(const void* self);
    void   (*destroy)(void* self);
} ShapeVTable;

// 기본 "클래스" 구조체 (vptr을 첫 번째 필드에)
typedef struct Shape {
    const ShapeVTable* vptr;  // 모든 Shape의 첫 필드
} Shape;

// Circle "클래스"
typedef struct Circle {
    const ShapeVTable* vptr;  // Shape과 동일한 레이아웃
    double radius;
} Circle;

static double circle_area(const void* self) {
    const Circle* c = (const Circle*)self;
    return 3.14159265 * c->radius * c->radius;
}
static double circle_perimeter(const void* self) {
    const Circle* c = (const Circle*)self;
    return 2.0 * 3.14159265 * c->radius;
}
static void circle_print(const void* self) {
    const Circle* c = (const Circle*)self;
    printf("Circle(radius=%.2f)\n", c->radius);
}
static void circle_destroy(void* self) { free(self); }

// Circle의 vtable (정적으로 초기화, 프로그램 전체에서 하나만 존재)
static const ShapeVTable circle_vtable = {
    .area      = circle_area,
    .perimeter = circle_perimeter,
    .print     = circle_print,
    .destroy   = circle_destroy,
};

Circle* circle_new(double radius) {
    Circle* c = malloc(sizeof(Circle));
    c->vptr   = &circle_vtable;  // vptr 초기화: C++ 생성자가 하는 일
    c->radius = radius;
    return c;
}

// Rectangle "클래스"
typedef struct Rectangle {
    const ShapeVTable* vptr;
    double width, height;
} Rectangle;

static double rect_area(const void* self) {
    const Rectangle* r = (const Rectangle*)self;
    return r->width * r->height;
}
static double rect_perimeter(const void* self) {
    const Rectangle* r = (const Rectangle*)self;
    return 2.0 * (r->width + r->height);
}
static void rect_print(const void* self) {
    const Rectangle* r = (const Rectangle*)self;
    printf("Rectangle(%.2f x %.2f)\n", r->width, r->height);
}

static const ShapeVTable rect_vtable = {
    .area = rect_area, .perimeter = rect_perimeter,
    .print = rect_print, .destroy = free,
};

Rectangle* rect_new(double w, double h) {
    Rectangle* r = malloc(sizeof(Rectangle));
    r->vptr = &rect_vtable;
    r->width = w; r->height = h;
    return r;
}

// 다형성 함수: Shape* (= void* with vptr 첫 필드)를 통해 동작
void describe_shape(Shape* s) {
    s->vptr->print(s);
    printf("  넓이: %.4f, 둘레: %.4f\n", s->vptr->area(s), s->vptr->perimeter(s));
}

int main(void) {
    Shape* shapes[3] = {
        (Shape*)circle_new(5.0),
        (Shape*)rect_new(4.0, 6.0),
        (Shape*)circle_new(2.5),
    };

    for (int i = 0; i < 3; i++) {
        describe_shape(shapes[i]);
        shapes[i]->vptr->destroy(shapes[i]);
    }
    return 0;
}
```

C++ 컴파일러가 자동으로 처리하는 모든 것—vtable 생성, vptr 초기화, 간접 함수 호출—을 손으로 직접 작성했습니다. `circle_vtable`과 `rect_vtable`은 각각 C++의 클래스 vtable에 해당하며, `c->vptr = &circle_vtable`은 생성자에서 컴파일러가 삽입하는 vptr 초기화에 해당합니다.

---

## 다중 상속과 vtable

C++의 다중 상속이 있을 때 vtable은 더 복잡해집니다. 파생 클래스가 여러 기반 클래스를 상속하면, 각 기반 클래스에 대한 별도의 vtable 포인터가 필요할 수 있습니다. 이때 **thunk**라는 짧은 코드 조각이 주소를 조정합니다.

```
class A { virtual void f(); };
class B { virtual void g(); };
class C : public A, public B { void f() override; void g() override; };

// C 객체의 메모리 레이아웃:
// [vptr_A ][A의 필드][vptr_B ][B의 필드][C의 필드]
//    │                   │
//    ▼ C의 A-part vtable  ▼ C의 B-part vtable
```

---

## 성능: 동적 디스패치의 비용

가상 함수 호출은 비가상 함수 호출보다 느립니다. 주요 비용 요인:

1. **간접 호출 오버헤드**: 메모리 역참조가 2회 추가됩니다 (vptr 로드, vtable 엔트리 로드).
2. **인라이닝 불가**: 간접 호출은 컴파일러가 인라인화하기 어렵습니다.
3. **분기 예측 어려움**: 호출될 함수 주소가 런타임에 결정되므로 CPU의 간접 분기 예측기에 의존합니다.
4. **캐시 미스**: vtable 자체가 콜드 캐시라면 추가 캐시 미스가 발생합니다.

### 탈가상화(Devirtualization)

컴파일러는 객체의 실제 타입을 정적으로 추론할 수 있을 때 동적 디스패치를 직접 호출로 최적화합니다.

```cpp
void foo() {
    Dog d;                // 스택에 생성, 타입이 명확
    d.speak();            // 컴파일러가 Animal*가 아닌 Dog 타입을 알고 있음
                          // → Dog::speak()로 직접 호출, vtable 우회 가능
}
```

PGO(Profile-Guided Optimization)와 LTO(Link-Time Optimization)를 함께 사용하면 런타임 프로파일 정보를 기반으로 "이 호출의 95%는 Dog::speak"라는 사실을 인식하여 추측 탈가상화(speculative devirtualization)를 수행합니다.

---

## Java와 Go의 인터페이스 디스패치

**Java**는 JVM의 `invokeinterface`와 `invokevirtual` 바이트코드를 사용합니다. 인터페이스 호출은 vtable 기반이 아닌 **itable(interface table)**을 통해 이루어지며, JIT 컴파일러가 런타임에 인라인 캐시(Inline Cache) 최적화를 적용합니다.

**Go**는 `interface`를 `(type, data)` 두 포인터로 구현합니다. 타입 포인터가 가리키는 테이블에 인터페이스 메서드의 구현 포인터가 들어 있습니다. Go의 인터페이스는 암묵적으로 구현되므로(structural typing), 빈 인터페이스 `interface{}`는 어떤 타입도 담을 수 있습니다.

---

## 주의사항과 팁

**가상 소멸자**: 기본 클래스 포인터로 파생 클래스 객체를 `delete`할 때, 소멸자가 `virtual`이 아니라면 파생 클래스의 소멸자가 호출되지 않아 자원 누수가 발생합니다. 다형적으로 사용될 클래스는 반드시 `virtual ~Base() = default;`를 선언하세요.

**순수 가상 함수**: `virtual void f() = 0;`으로 선언된 순수 가상 함수는 vtable에 특별한 "pure virtual called" 핸들러 주소가 들어갑니다. 추상 클래스의 객체를 직접 생성할 수 없는 이유입니다.

**성능이 중요한 경우**: 가상 함수 대신 CRTP(Curiously Recurring Template Pattern)를 사용한 정적 다형성, 또는 `std::variant` + `std::visit`을 고려하세요. 컴파일 타임에 디스패치를 해결하여 vtable 오버헤드를 제거합니다.

## 참고 자료
- [Dynamic dispatch (vtable) example in C - GitHub Gist](https://gist.github.com/hbobenicio/240a64f6c7c4726fc28741e68aa56745)
- [vtable topics - GitHub](https://github.com/topics/vtable?o=asc&s=forks)
- [Comprehensive Guide to ABI in C and C++ - GitHub Gist](https://gist.github.com/MangaD/506a0f3273724ef3af26b8c085accdcb)
