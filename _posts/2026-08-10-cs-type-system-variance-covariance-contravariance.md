---
layout: post
title: "타입 시스템 공변성·반공변성·불변성 완전 정복: 제네릭이 안전하면서도 유연한 이유"
date: 2026-08-10
categories: [cs, computer-science]
tags: [type-system, covariance, contravariance, invariance, generics, type-theory, kotlin, java, scala]
---

"왜 `List<Dog>`는 `List<Animal>`의 서브타입이 아닌가?" 자바나 코틀린으로 제네릭 코드를 처음 작성하다 보면 반드시 마주치는 질문입니다. 직관적으로는 `Dog`가 `Animal`의 서브타입이므로 `List<Dog>`도 `List<Animal>`의 서브타입이어야 할 것 같지만, 컴파일러는 이를 거부합니다. 이 글에서는 **공변성(Covariance)**, **반공변성(Contravariance)**, **불변성(Invariance)**의 정의와 원리를 타입 이론부터 실제 코드 예제까지 단계적으로 설명합니다.

---

## 1. 리스코프 치환 원칙과 서브타이핑

타입 S가 타입 T의 서브타입(S <: T)이라는 것은, **T를 기대하는 모든 문맥에서 S를 안전하게 사용할 수 있음**을 의미합니다. 이것이 리스코프 치환 원칙(LSP)입니다.

단순 타입에서는 직관과 잘 맞습니다: `Dog <: Animal`이면 `Animal`을 기대하는 곳에 `Dog`를 넣어도 안전합니다.

그런데 타입 생성자(type constructor) `F<T>`가 있을 때, `S <: T`이면 `F<S> <: F<T>`도 성립하는가? 이 질문이 **분산(Variance)**의 핵심입니다.

---

## 2. 세 가지 분산

### 2.1 공변(Covariant, +)

`S <: T → F<S> <: F<T>` 가 성립하면 F는 T에 대해 **공변**합니다. 즉, 타입 인자의 서브타입 관계가 그대로 유지됩니다.

**읽기 전용(생산자) 컨테이너**가 공변의 전형입니다. `List<Dog>`에서 꺼낸 원소는 반드시 `Dog`이므로 `Animal`로 사용해도 안전합니다.

### 2.2 반공변(Contravariant, -)

`S <: T → F<T> <: F<S>` (방향이 반대!)가 성립하면 F는 **반공변**합니다.

**소비자(Consumer) 함수**가 반공변의 전형입니다. `Animal`을 받아들이는 함수는 `Dog`도 받아들일 수 있습니다. 따라서 `(Animal) → Unit`을 기대하는 곳에 `(Dog) → Unit`을 넣으면 **안전하지 않지만**, `(Animal) → Unit`을 `(Dog) → Unit` 위치에 넣는 것은 안전합니다.

### 2.3 불변(Invariant)

`S <: T`여도 `F<S>`와 `F<T>` 사이에 서브타입 관계가 없으면 **불변**입니다. 읽기·쓰기가 모두 가능한 가변 컨테이너가 대표적입니다.

---

## 3. 왜 가변 리스트는 불변이어야 하는가

자바의 `List<T>`(가변)가 공변이라면 다음 코드가 컴파일될 것입니다:

```java
// 이 코드가 컴파일된다면 어떤 일이 생길까?
List<Dog> dogs = new ArrayList<>();
List<Animal> animals = dogs;   // 가정: List<Dog> <: List<Animal>

// animals를 통해 Cat을 추가
animals.add(new Cat());

// dogs에서 꺼내면 Dog라고 기대하지만 실제로는 Cat!
Dog d = dogs.get(0);  // ClassCastException at runtime
```

`List<Dog>`를 `List<Animal>`로 취급하면, `Animal`로의 업캐스트를 통해 `Cat`을 삽입할 수 있고, 이후 `Dog`로 꺼낼 때 런타임 예외가 발생합니다. 컴파일러는 이를 **타입 시스템 레벨에서 차단**하기 위해 가변 제네릭을 불변으로 처리합니다.

---

## 4. Kotlin의 선언 지점 분산 (Declaration-Site Variance)

코틀린은 타입 파라미터 선언 위치에서 분산을 명시합니다. `out`은 공변, `in`은 반공변입니다.

```kotlin
// out T: T를 반환(생산)만 할 수 있음 → 공변
interface Producer<out T> {
    fun produce(): T
    // fun consume(t: T) // 컴파일 에러: out 위치에서 T는 반환 타입에만 허용
}

// in T: T를 소비만 할 수 있음 → 반공변
interface Consumer<in T> {
    fun consume(t: T)
    // fun produce(): T // 컴파일 에러: in 위치에서 T는 파라미터 타입에만 허용
}

open class Animal(val name: String)
class Dog(name: String) : Animal(name)
class Cat(name: String) : Animal(name)

fun main() {
    // 공변: Producer<Dog>를 Producer<Animal>로 사용 가능
    val dogProducer: Producer<Dog> = object : Producer<Dog> {
        override fun produce() = Dog("Buddy")
    }
    val animalProducer: Producer<Animal> = dogProducer  // OK!
    println(animalProducer.produce().name)  // "Buddy"

    // 반공변: Consumer<Animal>을 Consumer<Dog>로 사용 가능
    val animalConsumer: Consumer<Animal> = object : Consumer<Animal> {
        override fun consume(t: Animal) = println("Got: ${t.name}")
    }
    val dogConsumer: Consumer<Dog> = animalConsumer  // OK!
    dogConsumer.consume(Dog("Max"))  // "Got: Max"
}
```

코틀린 표준 라이브러리의 `List<out E>`가 공변으로 선언된 이유가 바로 이것입니다. 코틀린의 `List`는 읽기 전용이므로, `List<Dog>`를 `List<Animal>`로 안전하게 사용할 수 있습니다.

---

## 5. 자바의 사용 지점 분산 (Use-Site Variance / Wildcards)

자바는 선언 지점 분산을 지원하지 않습니다. 대신 와일드카드(wildcard)를 통한 **사용 지점 분산**을 제공합니다.

```java
import java.util.ArrayList;
import java.util.List;

class Animal { String name; Animal(String name) { this.name = name; } }
class Dog extends Animal { Dog(String name) { super(name); } }
class Cat extends Animal { Cat(String name) { super(name); } }

public class VarianceDemo {

    // ? extends T → 공변 (생산자, 읽기만 허용)
    static double sumWeights(List<? extends Animal> animals) {
        // animals.add(new Dog("X")); // 컴파일 에러: 쓰기 불가 (타입 불확실)
        return animals.stream().count();
    }

    // ? super T → 반공변 (소비자, 쓰기 허용)
    static void addAnimals(List<? super Dog> list) {
        list.add(new Dog("Buddy"));
        list.add(new Dog("Max"));
        // Animal a = list.get(0); // 컴파일 에러: 읽기는 Object로만 가능
    }

    public static void main(String[] args) {
        List<Dog> dogs = new ArrayList<>();

        // List<? super Dog>는 List<Dog>, List<Animal>, List<Object> 모두 허용
        addAnimals(dogs);

        // List<? extends Animal>은 List<Dog>, List<Cat> 모두 허용 (읽기 안전)
        sumWeights(dogs);  // List<Dog> → List<? extends Animal>
        System.out.println("Size: " + dogs.size()); // 2
    }
}
```

**PECS 원칙(Producer Extends, Consumer Super)**은 자바 와일드카드 사용의 황금률입니다:
- 데이터를 **꺼내는(생산)** 곳 → `? extends T`
- 데이터를 **넣는(소비)** 곳 → `? super T`
- 둘 다 하는 곳 → 와일드카드 사용 불가, 명시적 타입 사용

---

## 6. 함수 타입의 분산

함수 타입 `(A) → B`에서:
- **입력 타입 A**는 반공변: `A' <: A`이면 `(A) → B <: (A') → B`
- **반환 타입 B**는 공변: `B <: B'`이면 `(A) → B <: (A) → B'`

직관적으로, 더 일반적인 입력을 받는 함수는 더 구체적인 입력을 받는 함수 자리에 사용할 수 있고, 더 구체적인 값을 반환하는 함수는 더 일반적인 반환 타입 자리에 사용할 수 있습니다.

코틀린의 함수 타입 `Function1<in P, out R>`에 이 규칙이 그대로 인코딩되어 있습니다.

---

## 7. Scala의 타입 시스템: 공변 컬렉션

Scala는 `List[+A]`(공변)와 `Array[A]`(불변)를 명확히 구분합니다:

```scala
// Scala
val dogs: List[Dog] = List(new Dog("Buddy"))
val animals: List[Animal] = dogs  // OK: List[+A]는 공변

val dogArr: Array[Dog] = Array(new Dog("Max"))
// val animalArr: Array[Animal] = dogArr  // 컴파일 에러: Array[A]는 불변
```

Scala의 `List`는 불변(immutable) 컬렉션이므로 공변이 안전합니다. `Array`는 가변(mutable)이므로 불변(invariant) 타입 파라미터를 사용합니다.

---

## 8. 주의사항과 실전 팁

1. **읽기 전용 = 공변, 쓰기 전용 = 반공변, 읽기·쓰기 = 불변**이 핵심 규칙입니다.
2. 코틀린 `MutableList<T>`는 불변(invariant)입니다. `MutableList<Dog>`는 `MutableList<Animal>`의 서브타입이 아닙니다.
3. **배열(Array)은 대부분의 언어에서 불변**입니다. 자바의 배열은 런타임에 예외를 발생시킬 수 있는 공변 배열을 허용하는 역사적 실수가 있습니다.
4. API 설계 시 인터페이스를 생산자/소비자로 분리하면 분산 관계를 명확히 할 수 있습니다.
5. **타입 경계 체크는 컴파일 타임에**: 분산 규칙을 어기면 런타임 캐스트 예외가 아니라 컴파일 에러가 발생하도록 설계하는 것이 올바른 접근입니다.

---

## 9. 정리

공변성·반공변성·불변성은 제네릭 타입 시스템이 유연성과 타입 안전성을 동시에 제공하기 위한 핵심 메커니즘입니다. "데이터를 꺼내기만 하면 공변, 넣기만 하면 반공변, 둘 다 하면 불변"이라는 원칙을 기억하면 복잡한 제네릭 코드에서도 타입 오류를 빠르게 진단할 수 있습니다. 리스코프 치환 원칙을 타입 생성자 레벨로 확장한 것이 분산 이론의 본질이며, 이를 이해하면 코틀린의 `in`/`out`, 자바의 와일드카드, Scala의 `+`/`-` 표기가 모두 같은 개념의 서로 다른 문법임을 알 수 있습니다.

## 참고 자료
- [Covariance and Contravariance (Computer Science) - Wikipedia](https://en.wikipedia.org/wiki/Covariance_and_contravariance_(computer_science))
- [Kotlin Generics: in, out, where - Kotlin Docs](https://kotlinlang.org/docs/generics.html)
- [Java Wildcards - Oracle Docs](https://docs.oracle.com/javase/tutorial/java/generics/wildcards.html)
- [Variance in Type Systems - Alexander Guz](https://guzalexander.com/2020/02/04/variance-in-type-systems.html)
