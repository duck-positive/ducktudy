---
layout: post
title: "C++ 템플릿 메타프로그래밍(TMP) 완전 정복: SFINAE, Concepts, constexpr로 컴파일 타임 계산 구현하기"
date: 2026-09-01
categories: [cs, computer-science]
tags: [cpp, template-metaprogramming, SFINAE, concepts, constexpr, compile-time, generic-programming]
---

## 개요

C++ 템플릿 메타프로그래밍(Template Metaprogramming, TMP)은 **컴파일 타임에 프로그램을 실행하는 기법**이다. 일반적인 코드가 런타임에 실행되는 반면, TMP는 컴파일러가 타입과 상수를 분석하는 과정에서 계산을 완료한다. 그 결과 생성된 바이너리는 이미 최적화된 특수화 코드를 담게 되어, 런타임 오버헤드가 사실상 0이 된다.

1994년 Erwin Unruh가 소수를 나열하는 프로그램을 컴파일 경고 메시지로만 출력하는 코드를 공개하면서, C++ 템플릿이 튜링 완전(Turing-complete)하다는 사실이 알려졌다. 이후 Andrei Alexandrescu의 저서 *Modern C++ Design*(2001)을 통해 TMP는 본격적인 설계 기법으로 발전했고, C++11/14/17/20을 거치며 `constexpr`, `if constexpr`, Concepts 등 언어 차원의 지원이 대폭 강화되었다.

---

## 왜 TMP가 필요한가

### 1. 제로 비용 추상화(Zero-Cost Abstraction)

```cpp
// 런타임 조건 분기 — 분기 예측 실패 가능성 존재
int abs_runtime(int x) { return x < 0 ? -x : x; }

// 컴파일 타임 타입 분기 — 조건 자체가 바이너리에 없음
template <typename T>
constexpr T abs_compile(T x) { return x < 0 ? -x : x; }
```

두 번째 함수는 `constexpr`로 선언되어, 인자가 컴파일 타임 상수라면 컴파일러가 결과를 바이너리 상수로 치환한다. 분기 코드 자체가 사라지는 것이다.

### 2. 타입 기반 알고리즘 선택

정렬 알고리즘을 예로 들면, `RandomAccessIterator`에는 퀵소트를, `ForwardIterator`에는 병합 정렬을 자동 선택하는 코드를 TMP로 작성할 수 있다. STL의 `std::sort`가 이런 방식으로 구현된다.

### 3. 도메인 특화 언어(DSL) 구축

Eigen(선형대수 라이브러리), Boost.Spirit(파서 생성기) 등은 TMP로 C++ 문법 안에서 수학 표기법이나 BNF 문법을 직접 표현한다.

---

## 핵심 개념 1: SFINAE (Substitution Failure Is Not An Error)

SFINAE는 "치환 실패는 에러가 아니다"라는 뜻이다. 컴파일러가 함수 오버로드 후보를 고를 때, 템플릿 인자를 대입(substitution)하는 과정에서 타입 에러가 발생하면 해당 후보를 **조용히 제외**한다. 이를 이용해 특정 타입에만 활성화되는 함수를 작성할 수 있다.

```cpp
#include <type_traits>
#include <iostream>

// 정수 타입일 때만 활성화
template <typename T>
std::enable_if_t<std::is_integral_v<T>, void>
print_type(T value) {
    std::cout << "정수: " << value << "\n";
}

// 부동소수점 타입일 때만 활성화
template <typename T>
std::enable_if_t<std::is_floating_point_v<T>, void>
print_type(T value) {
    std::cout << "실수: " << value << "\n";
}

int main() {
    print_type(42);      // 출력: 정수: 42
    print_type(3.14);    // 출력: 실수: 3.14
    // print_type("hi"); // 컴파일 에러: 후보 없음
}
```

`std::enable_if_t<조건, 반환타입>`은 조건이 참일 때만 `반환타입`으로 평가된다. 조건이 거짓이면 타입 자체가 존재하지 않아 치환 실패가 발생하고, 그 오버로드 후보가 제거된다.

### `decltype`과 `std::void_t`를 활용한 메서드 감지

```cpp
#include <type_traits>

// T에 .serialize() 메서드가 있는지 컴파일 타임에 검사
template <typename T, typename = void>
struct has_serialize : std::false_type {};

template <typename T>
struct has_serialize<T, std::void_t<decltype(std::declval<T>().serialize())>>
    : std::true_type {};

struct Foo { std::string serialize() const { return "{}"; } };
struct Bar {};

static_assert(has_serialize<Foo>::value);   // 통과
static_assert(!has_serialize<Bar>::value);  // 통과
```

`std::void_t`는 C++17 유틸리티로, 임의 개수의 타입 표현식이 모두 유효할 때만 `void`로 평가된다. 하나라도 유효하지 않으면 치환 실패로 `false_type` 특수화가 선택된다.

---

## 핵심 개념 2: `constexpr`와 컴파일 타임 계산

C++11에서 도입된 `constexpr`는 함수나 변수를 컴파일 타임에 평가 가능하도록 선언한다. C++14부터는 루프와 지역 변수도 허용되어 훨씬 유연해졌다.

```cpp
#include <array>
#include <cstddef>

// 컴파일 타임 피보나치 수열 (C++17 constexpr)
constexpr std::size_t fib(std::size_t n) {
    if (n <= 1) return n;
    std::size_t a = 0, b = 1;
    for (std::size_t i = 2; i <= n; ++i) {
        std::size_t tmp = a + b;
        a = b;
        b = tmp;
    }
    return b;
}

// 컴파일 타임에 배열 크기 결정
constexpr auto fib_table = []() {
    std::array<std::size_t, 20> arr{};
    for (std::size_t i = 0; i < 20; ++i)
        arr[i] = fib(i);
    return arr;
}();

// static_assert로 컴파일 타임 검증
static_assert(fib_table[10] == 55);
static_assert(fib_table[19] == 4181);
```

C++17의 `if constexpr`는 조건에 따라 **분기 자체를 컴파일 타임에 제거**하므로, SFINAE보다 훨씬 읽기 쉬운 코드를 작성할 수 있다.

```cpp
#include <string>
#include <type_traits>

template <typename T>
std::string to_str(T value) {
    if constexpr (std::is_same_v<T, bool>) {
        return value ? "true" : "false";
    } else if constexpr (std::is_integral_v<T>) {
        return std::to_string(value);
    } else if constexpr (std::is_floating_point_v<T>) {
        return std::to_string(value);
    } else {
        // T가 std::string이거나 변환 가능한 타입이라고 가정
        return std::string(value);
    }
}
```

---

## 핵심 개념 3: C++20 Concepts — SFINAE의 진화

Concepts는 C++20에서 정식 도입된 기능으로, 타입 제약(constraint)을 **명시적이고 읽기 쉬운 문법**으로 표현한다. SFINAE와 달리 컴파일 에러 메시지가 명확하고, 제약 조건 자체를 재사용할 수 있다.

```cpp
#include <concepts>
#include <numeric>
#include <vector>
#include <iostream>

// Concept 정의: 산술 연산이 가능한 타입
template <typename T>
concept Arithmetic = std::is_arithmetic_v<T>;

// Concept 정의: 반복 가능하고, 값 타입이 산술인 컨테이너
template <typename C>
concept NumericRange =
    requires(C c) {
        { c.begin() } -> std::input_iterator;
        { c.end() }   -> std::input_iterator;
    } &&
    Arithmetic<typename C::value_type>;

// Concept으로 제약된 함수 템플릿
template <NumericRange C>
auto sum(const C& container) {
    return std::accumulate(container.begin(), container.end(),
                           typename C::value_type{});
}

int main() {
    std::vector<int> v = {1, 2, 3, 4, 5};
    std::cout << sum(v) << "\n";  // 15

    // std::vector<std::string> vs = {"a", "b"};
    // sum(vs);  // 컴파일 에러: "constraints not satisfied" — 명확한 메시지
}
```

### `requires` 절의 4가지 형태

```cpp
// 1. 단순 표현식 요구 — 표현식이 유효한지만 확인
requires { t.serialize(); }

// 2. 타입 요구 — 중첩 타입이 존재하는지 확인
requires { typename T::value_type; }

// 3. 복합 요구 — 표현식 유효성 + 반환 타입 제약
requires { { t.size() } -> std::convertible_to<std::size_t>; }

// 4. 중첩 요구 — requires 안에 requires
requires { requires std::copyable<T>; }
```

---

## 실전 예제: 컴파일 타임 단위 시스템

TMP의 강력한 응용 사례로, 물리 단위를 타입으로 인코딩하여 단위 불일치를 **컴파일 에러**로 잡는 시스템을 구현할 수 있다.

```cpp
#include <ratio>
#include <type_traits>

// 단위를 차원으로 인코딩: [길이, 시간, 질량]을 정수 지수로 표현
template <int L, int T, int M>
struct Unit {
    static constexpr int length = L;
    static constexpr int time   = T;
    static constexpr int mass   = M;
};

using Meter    = Unit<1, 0, 0>;
using Second   = Unit<0, 1, 0>;
using Kilogram = Unit<0, 0, 1>;
using MeterPerSec = Unit<1, -1, 0>;  // m/s
using Newton   = Unit<1, -2, 1>;     // kg·m/s²

template <typename U>
struct Quantity {
    double value;
    explicit constexpr Quantity(double v) : value(v) {}
};

// 곱셈: 지수 덧셈
template <typename U1, typename U2>
auto operator*(Quantity<U1> a, Quantity<U2> b) {
    using ResultUnit = Unit<U1::length + U2::length,
                             U1::time   + U2::time,
                             U1::mass   + U2::mass>;
    return Quantity<ResultUnit>{a.value * b.value};
}

// 덧셈: 단위가 같아야만 컴파일 가능
template <typename U>
Quantity<U> operator+(Quantity<U> a, Quantity<U> b) {
    return Quantity<U>{a.value + b.value};
}

int main() {
    Quantity<Kilogram> mass{70.0};
    Quantity<MeterPerSec> accel{9.8};  // 단위: m/s (가속도는 m/s²이지만 예시)

    auto result = mass * accel;  // 컴파일 타임에 단위 추론

    Quantity<Meter> d1{100.0}, d2{50.0};
    auto total = d1 + d2;  // OK: 같은 단위

    // Quantity<Second> s{1.0};
    // d1 + s;  // 컴파일 에러: 단위 불일치
}
```

---

## 주의사항과 팁

### 1. 컴파일 시간 폭발 문제

재귀적 TMP는 인스턴스화 깊이가 깊어질수록 컴파일 시간이 기하급수적으로 증가한다. GCC/Clang 기본 제한은 약 900개 수준이다. `constexpr` 루프로 대체하면 컴파일 시간을 크게 줄일 수 있다.

```cpp
// 느린 방법 — 재귀 인스턴스화
template <int N>
struct Factorial { static constexpr long long value = N * Factorial<N-1>::value; };
template <> struct Factorial<0> { static constexpr long long value = 1; };

// 빠른 방법 — constexpr 루프
constexpr long long factorial(int n) {
    long long result = 1;
    for (int i = 2; i <= n; ++i) result *= i;
    return result;
}
```

### 2. 에러 메시지 가독성

SFINAE 실패 시 에러 메시지는 악명 높게 길다. C++20 Concepts을 사용하거나, `static_assert`에 메시지를 추가하면 개발 경험이 크게 향상된다.

```cpp
template <typename T>
void process(T value) {
    static_assert(std::is_trivially_copyable_v<T>,
                  "T must be trivially copyable for zero-copy transfer");
    // ...
}
```

### 3. `if constexpr` vs SFINAE 선택 기준

- **함수 본문 내부** 에서 타입 분기가 필요하면 `if constexpr` 사용
- **함수 시그니처 수준** 에서 오버로드 선택이 필요하면 Concepts(C++20) 또는 SFINAE 사용
- C++17 미만 환경에서만 SFINAE를 마지못해 사용

### 4. 타입 특성(Type Traits) 라이브러리 활용

`<type_traits>` 헤더의 `std::is_*`, `std::has_*` 계열을 숙지하면 TMP 코드의 절반은 이미 작성된 셈이다. Abseil 등 현대적 라이브러리도 추가적인 타입 특성 유틸리티를 제공한다.

---

## 정리

C++ TMP는 런타임 비용 없이 타입 안전성과 고성능을 동시에 달성하는 강력한 기법이다. SFINAE → `if constexpr` → Concepts 순으로 C++ 표준이 발전하면서 코드의 표현력과 가독성이 크게 향상되었다. 현대 C++ 개발에서 TMP를 완전히 피하기는 어렵지만, 항상 가장 새로운 언어 기능을 우선 검토하는 것이 유지보수성 측면에서 현명하다.

## 참고 자료
- [modern-cpp-features: C++ 버전별 언어/라이브러리 기능 치트시트](https://github.com/AnthonyCalandra/modern-cpp-features)
- [abseil-cpp: Google의 C++ 공통 라이브러리 (타입 유틸리티 참고)](https://github.com/abseil/abseil-cpp)
- [awesome-hpp: 헤더 온리 C++ 라이브러리 목록 (TMP 활용 사례)](https://github.com/p-ranav/awesome-hpp)
