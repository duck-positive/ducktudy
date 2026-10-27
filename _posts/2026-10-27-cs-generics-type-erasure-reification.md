---
layout: post
title: "제네릭 타입 소거와 구체화: Java·Kotlin·C++·Rust의 접근 방식 비교"
date: 2026-10-27
categories: [cs, computer-science]
tags: [generics, type-erasure, reification, java, kotlin, cpp, rust, type-system]
---

제네릭(Generics)은 현대 프로그래밍 언어에서 타입 안전성과 코드 재사용성을 동시에 달성하는 핵심 메커니즘이다. 그런데 동일한 "제네릭"이라는 개념이 언어마다 전혀 다른 방식으로 구현된다. Java는 **타입 소거(Type Erasure)**를, C++과 Rust는 **단형화(Monomorphization)**를, Kotlin은 두 가지를 모두 지원하는 혼합 방식을 채택했다. 이 차이는 런타임 성능, 이진 크기, 리플렉션 능력에 큰 영향을 미친다. 각 언어의 접근 방식을 깊이 파고들어 보자.

## 타입 소거(Type Erasure)란?

타입 소거는 제네릭 타입 정보를 **컴파일 시점에만 사용**하고, 런타임에는 제거하는 방식이다. Java가 1.5(2004년)에 제네릭을 도입할 때 하위 호환성을 위해 선택한 설계이다.

```java
// 소스 코드
List<String> strings = new ArrayList<String>();
strings.add("hello");
String s = strings.get(0);

// 컴파일 후 바이트코드 (타입 소거 적용)
List strings = new ArrayList();      // raw type
strings.add("hello");
String s = (String) strings.get(0);  // 컴파일러가 캐스팅 삽입
```

런타임에는 `List<String>`과 `List<Integer>`가 모두 그냥 `List`다. `instanceof List<String>`은 컴파일 오류가 되고, `List.class`와 `List<String>.class`는 같은 객체를 가리킨다.

### 타입 소거의 한계

```java
import java.util.ArrayList;
import java.util.List;

public class ErasureLimitations {
    
    // 1. 런타임에 제네릭 타입 검사 불가
    public static <T> boolean isListOfString(List<T> list) {
        // return list instanceof List<String>;  // 컴파일 오류!
        return true; // 타입 정보 없음
    }
    
    // 2. 제네릭 배열 생성 불가
    public static <T> T[] makeArray(int size) {
        // return new T[size];  // 컴파일 오류!
        return null; // 우회 불가
    }
    
    // 3. 오버로딩 충돌 — 소거 후 시그니처가 같아짐
    // void process(List<String> list) { }
    // void process(List<Integer> list) { }  // 컴파일 오류!
    
    // 4. static 필드로 제네릭 타입 사용 불가
    // static T instance;  // 오류
    
    public static void main(String[] args) {
        List<String> strings = new ArrayList<>();
        List<Integer> integers = new ArrayList<>();
        
        // 런타임에는 동일한 클래스
        System.out.println(strings.getClass() == integers.getClass()); // true
        System.out.println(strings.getClass().getName()); // java.util.ArrayList
    }
}
```

## 구체화(Reification): C++의 템플릿 방식

C++ 템플릿은 **컴파일 시 각 타입 인수마다 별도의 코드를 생성**한다. `vector<int>`와 `vector<double>`은 완전히 별개의 클래스다.

```cpp
#include <iostream>
#include <vector>
#include <typeinfo>

// 컴파일러는 이 함수를 int, double, string 버전으로 각각 생성
template<typename T>
T findMax(const std::vector<T>& vec) {
    if (vec.empty()) throw std::runtime_error("empty vector");
    T maxVal = vec[0];
    for (const T& item : vec) {
        if (item > maxVal) maxVal = item;
    }
    return maxVal;
}

// 런타임 타입 정보 접근 가능
template<typename T>
void printTypeInfo(const T& value) {
    std::cout << "Type: " << typeid(T).name() 
              << ", Value: " << value << std::endl;
}

// SFINAE (Substitution Failure Is Not An Error) — 타입 조건부 특수화
template<typename T, typename = std::enable_if_t<std::is_integral_v<T>>>
T bitwiseAnd(T a, T b) {
    return a & b;
}

int main() {
    std::vector<int> ints = {3, 1, 4, 1, 5, 9};
    std::vector<double> doubles = {3.14, 2.71, 1.41};
    
    // 각각 별도의 인스턴스화된 함수 호출
    std::cout << "Max int: " << findMax(ints) << std::endl;
    std::cout << "Max double: " << findMax(doubles) << std::endl;
    
    printTypeInfo(42);
    printTypeInfo(3.14);
    
    // int와 double 버전은 완전히 다른 함수
    std::cout << "sizeof(vector<int>): " << sizeof(std::vector<int>) << std::endl;
    std::cout << "sizeof(vector<double>): " << sizeof(std::vector<double>) << std::endl;
    
    return 0;
}
```

C++ 템플릿은 덕 타이핑(duck typing)처럼 동작한다. 타입이 필요한 연산(`operator>` 등)을 지원하기만 하면 템플릿에 사용할 수 있다. 오류는 템플릿 인스턴스화 시점에 발견된다.

**이진 크기 팽창(Code Bloat)**: 모든 타입 인수마다 별도 코드가 생성되므로 바이너리 크기가 커진다. `std::sort<int*>`와 `std::sort<double*>`는 서로 다른 기계어 코드를 가진다.

## Rust의 단형화와 트레이트 객체

Rust는 C++과 유사한 **단형화(Monomorphization)** 방식을 사용하면서, 런타임 다형성이 필요한 경우 **트레이트 객체(Trait Objects)**를 통해 동적 디스패치를 지원한다.

```rust
use std::fmt::Display;

// 제네릭 함수 — 컴파일 시 각 타입마다 별도 코드 생성
fn largest<T: PartialOrd>(list: &[T]) -> &T {
    let mut largest = &list[0];
    for item in list {
        if item > largest {
            largest = item;
        }
    }
    largest
}

// 구조체 제네릭
struct Pair<T> {
    first: T,
    second: T,
}

impl<T: Display + PartialOrd> Pair<T> {
    fn cmp_display(&self) {
        if self.first >= self.second {
            println!("첫 번째 값이 더 큼: {}", self.first);
        } else {
            println!("두 번째 값이 더 큼: {}", self.second);
        }
    }
}

// 트레이트 객체: 런타임 다형성 (동적 디스패치, vtable 사용)
// Box<dyn Trait>는 Java의 인터페이스와 유사하지만 타입 정보를 런타임에 vtable로 관리
trait Shape {
    fn area(&self) -> f64;
    fn name(&self) -> &str;
}

struct Circle { radius: f64 }
struct Rectangle { width: f64, height: f64 }

impl Shape for Circle {
    fn area(&self) -> f64 { std::f64::consts::PI * self.radius * self.radius }
    fn name(&self) -> &str { "Circle" }
}

impl Shape for Rectangle {
    fn area(&self) -> f64 { self.width * self.height }
    fn name(&self) -> &str { "Rectangle" }
}

fn print_shape_info(shape: &dyn Shape) {
    println!("{}: 넓이 = {:.2}", shape.name(), shape.area());
}

fn main() {
    let numbers = vec![34, 50, 25, 100, 65];
    println!("가장 큰 수: {}", largest(&numbers));
    
    let chars = vec!['y', 'm', 'a', 'q'];
    println!("가장 큰 문자: {}", largest(&chars));
    
    // 트레이트 객체 — 런타임에 타입 결정
    let shapes: Vec<Box<dyn Shape>> = vec![
        Box::new(Circle { radius: 3.0 }),
        Box::new(Rectangle { width: 4.0, height: 5.0 }),
    ];
    
    for shape in &shapes {
        print_shape_info(shape.as_ref()); // vtable을 통한 동적 디스패치
    }
}
```

**단형화 vs 트레이트 객체 성능 비교:**

| 특성 | 제네릭 (단형화) | 트레이트 객체 (`dyn Trait`) |
|------|----------------|--------------------------|
| 디스패치 방식 | 정적 (컴파일 시 결정) | 동적 (vtable 통해 런타임) |
| 성능 | 더 빠름 (인라이닝 가능) | 간접 호출 오버헤드 |
| 이진 크기 | 타입별로 증가 | 하나의 코드 |
| 이기종 컬렉션 | 불가 | 가능 (`Vec<Box<dyn Trait>>`) |

## Kotlin: 타입 소거 + reified 인라인 함수

Kotlin은 JVM 위에서 실행되므로 기본적으로 Java와 동일한 타입 소거를 사용한다. 하지만 `inline` 함수 + `reified` 타입 파라미터를 통해 런타임에 제네릭 타입 정보를 사용할 수 있다.

```kotlin
import kotlin.reflect.KClass

// 일반 제네릭 함수 (타입 소거 적용)
fun <T> List<T>.firstOrNull(): T? = if (isEmpty()) null else this[0]

// reified 제네릭: 인라인 함수에서만 사용 가능
// 컴파일러가 호출 지점에 함수 본체를 인라이닝하고 T를 실제 타입으로 대체
inline fun <reified T> List<*>.filterIsInstance(): List<T> {
    return this.filter { it is T }.map { it as T }
}

inline fun <reified T> Any.isInstanceOf(): Boolean = this is T

inline fun <reified T> createInstance(): T {
    return T::class.java.getDeclaredConstructor().newInstance()
}

// 실용 예: JSON 역직렬화에서 reified 활용
inline fun <reified T> String.fromJson(): T {
    // 런타임에 T::class.java를 실제 타입으로 접근 가능
    val targetClass = T::class.java
    println("역직렬화 대상 클래스: ${targetClass.simpleName}")
    // return objectMapper.readValue(this, targetClass)  // 실제로는 Jackson 등 사용
    return targetClass.getDeclaredConstructor().newInstance()
}

// 타입 소거의 한계를 reified로 극복
inline fun <reified T : Any> safeListCast(list: List<*>): List<T>? {
    return if (list.all { it is T }) {
        @Suppress("UNCHECKED_CAST")
        list as List<T>
    } else null
}

data class User(val name: String = "Default")
data class Order(val id: Int = 0)

fun main() {
    val mixed: List<Any> = listOf("hello", 42, "world", 3.14, "!")
    
    // reified 덕분에 런타임 타입 정보 사용
    val strings = mixed.filterIsInstance<String>()
    println("문자열만: $strings")  // [hello, world, !]
    
    val numbers = mixed.filterIsInstance<Int>()
    println("정수만: $numbers")  // [42]
    
    // 런타임 타입 체크
    println("\"hello\" is String: " + "hello".isInstanceOf<String>())  // true
    println("42 is String: " + 42.isInstanceOf<String>())  // false
    
    // 제네릭 타입 소거의 현실
    val intList = listOf(1, 2, 3)
    val strList = listOf("a", "b", "c")
    println("같은 클래스? ${intList::class == strList::class}")  // true
    println("클래스: ${intList::class}")  // class java.util.Arrays$ArrayList
}
```

### reified가 가능한 이유

`inline` 함수는 컴파일러가 호출 지점에 함수 본문을 **그대로 붙여넣기**한다. 이때 `T`가 실제 타입으로 대체되므로, 바이트코드 레벨에서는 타입 정보가 실제 클래스 리터럴로 존재한다. 이것이 런타임 타입 소거를 우회하는 원리다.

```kotlin
// 호출 코드
val result = list.filterIsInstance<String>()

// 인라이닝 후 실제 생성되는 바이트코드 (의사 코드)
val result = list.filter { it is String }.map { it as String }
// T가 String으로 완전히 대체됨!
```

## 네 언어의 접근 방식 비교

```
언어        | 방식           | 런타임 타입 정보 | 이진 크기 | 리플렉션
-----------|----------------|----------------|---------|--------
Java       | 타입 소거       | 없음           | 작음    | 제한적
Kotlin     | 소거 + reified | inline에서 가능 | 중간    | 우수
C++        | 단형화/템플릿   | 있음(typeid)   | 큼      | 없음
Rust       | 단형화 + dyn   | dyn Trait만    | 큼(단형) | 없음
```

### 각 언어가 선택한 이유

- **Java**: 1.4 이하 코드와의 100% 하위 호환성. 기존 `raw type` 코드가 경고는 나지만 동작해야 했다.
- **Kotlin**: JVM 생태계를 유지하면서 `reified`로 주요 불편함을 해결. 실용적 타협점.
- **C++**: 최대 성능과 유연성. 템플릿 메타프로그래밍(TMP)을 통해 컴파일 시 복잡한 계산도 수행.
- **Rust**: 제로-코스트 추상화 원칙. 단형화로 성능 보장, `dyn Trait`으로 필요할 때만 동적 디스패치.

## 주의사항과 팁

### 1. Java에서 타입 토큰(TypeToken) 패턴

런타임에 제네릭 타입 정보가 필요할 때 사용하는 우회 기법이다.

```java
// Gson, Jackson 등의 라이브러리에서 사용하는 TypeToken 패턴
import java.lang.reflect.ParameterizedType;
import java.lang.reflect.Type;

// 익명 서브클래스를 통해 제네릭 타입 정보를 런타임에 보존
abstract class TypeToken<T> {
    private final Type type;
    
    protected TypeToken() {
        // 슈퍼클래스의 타입 파라미터에서 실제 타입 추출
        Type superClass = getClass().getGenericSuperclass();
        this.type = ((ParameterizedType) superClass).getActualTypeArguments()[0];
    }
    
    public Type getType() { return type; }
}

// 사용법:
// TypeToken<List<String>> token = new TypeToken<List<String>>() {};
// Type type = token.getType();  // java.util.List<java.lang.String>
```

### 2. Kotlin에서 reified의 제한

- `inline` 함수에서만 사용 가능
- 클래스의 일반 메서드에는 적용 불가
- `reified` 타입 파라미터로 인스턴스 생성 시 기본 생성자가 필요

### 3. C++ 개념(Concepts, C++20)

C++20의 `concept`는 템플릿 타입 제약을 명확하게 표현해 컴파일 오류 메시지를 개선한다.

```cpp
#include <concepts>
#include <string>

// 개념 정의: T는 산술 타입이어야 함
template<typename T>
concept Arithmetic = std::is_arithmetic_v<T>;

// 제약된 템플릿
template<Arithmetic T>
T add(T a, T b) { return a + b; }

// std::string으로 호출하면 명확한 오류 메시지 발생
// add(std::string("a"), std::string("b"));
// error: ... does not satisfy Arithmetic
```

### 4. 런타임 성능 비교 요약

타입 소거(Java) 방식은 제네릭 컬렉션에서 원시 타입 사용 시 **오토박싱(autoboxing)** 오버헤드가 발생한다. `List<Integer>`는 내부적으로 `int`가 아닌 `Integer` 객체로 저장된다. 성능이 중요한 경우 Eclipse Collections, Trove, Koloboke 같은 원시 타입 특화 컬렉션 라이브러리를 고려하라.

Rust와 C++의 단형화는 컴파일러가 각 타입별 최적화(인라이닝, SIMD 등)를 수행할 수 있어 성능 면에서 우위를 가진다. 단, 컴파일 시간과 바이너리 크기가 증가한다.

## 참고 자료
- [Java Generics FAQ — Angelika Langer](http://www.angelikalanger.com/GenericsFAQ/JavaGenericsFAQ.html)
- [Kotlin Inline Functions and Reified Type Parameters 공식 문서](https://kotlinlang.org/docs/inline-functions.html)
- [The Rust Programming Language — Generics](https://doc.rust-lang.org/book/ch10-00-generics.html)
- [C++ Templates: The Complete Guide (2nd Ed.) — Josuttis, Vandevoorde, Gregor](https://www.oreilly.com/library/view/c-templates-the/9780134778808/)
