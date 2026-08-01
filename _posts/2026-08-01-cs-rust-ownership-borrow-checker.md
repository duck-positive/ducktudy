---
layout: post
title: "Rust 소유권 모델과 빌림 검사기 완전 정복: GC 없이 메모리 안전을 보장하는 법"
date: 2026-08-01
categories: [cs, computer-science]
tags: [rust, ownership, borrow-checker, memory-safety, lifetime, systems-programming, zero-cost-abstraction]
---

메모리 버그는 소프트웨어 보안 취약점의 70% 이상을 차지한다. C/C++는 성능을 위해 개발자가 직접 메모리를 관리하지만, 이는 use-after-free, double free, null pointer dereference, buffer overflow 같은 치명적 버그를 유발한다. 반대로 Java, Python 같은 언어는 GC(가비지 컬렉터)로 메모리 안전을 보장하지만 런타임 오버헤드와 GC 일시 정지가 발생한다. Rust는 세 번째 길을 제시한다: **컴파일 타임 소유권 시스템으로 GC 없이 메모리 안전을 100% 보장한다.**

## 소유권(Ownership)의 세 가지 규칙

Rust의 모든 메모리 안전 보장은 단 세 가지 규칙에서 비롯된다.

**규칙 1**: Rust의 각 값은 단 하나의 *소유자(owner)*를 가진다.  
**규칙 2**: 동시에 단 하나의 소유자만 존재할 수 있다.  
**규칙 3**: 소유자가 스코프를 벗어나면 값은 즉시 드롭(drop)된다.

이 규칙은 컴파일러가 강제한다. 런타임 비용은 0이다.

```rust
fn main() {
    let s1 = String::from("hello");  // s1이 "hello" 문자열의 소유자
    let s2 = s1;                     // 소유권이 s1 → s2로 이동(Move)
    
    // println!("{}", s1);           // 컴파일 에러! s1은 더 이상 유효하지 않음
    // error[E0382]: borrow of moved value: `s1`
    
    println!("{}", s2);              // OK: s2가 현재 소유자
}  // 스코프 종료 → s2 드롭 → 힙 메모리 해제 (free 자동 호출)
```

이 Move 시맨틱은 C++의 `std::move`와 개념적으로 유사하지만, Rust에서는 Move 후 이전 변수 접근을 **컴파일러가 금지**한다는 점이 다르다.

### Copy 타입: 스택 할당 값들의 특별 처리

정수, 부동소수점, 불리언 같은 고정 크기 스택 타입들은 `Copy` 트레이트를 구현하며, Move 대신 자동으로 복사된다.

```rust
fn main() {
    let x: i32 = 42;
    let y = x;        // Copy: x의 값이 복사됨 (Move가 아님)
    
    println!("x={}, y={}", x, y);  // 둘 다 유효! Copy 타입은 Move되지 않음
    
    // Copy 트레이트를 구현하는 타입들:
    // - 모든 정수형 (i8, i16, i32, i64, i128, u8, u16, ...)
    // - 부동소수점 (f32, f64)
    // - bool, char
    // - 튜플 (모든 원소가 Copy인 경우)
    // - 배열 (원소가 Copy인 경우)
    
    // String은 Copy를 구현하지 않음 — 힙 할당이기 때문
    let s1 = String::from("world");
    let s2 = s1.clone();  // 명시적으로 깊은 복사(deep copy)를 원하면 clone() 사용
    println!("s1={}, s2={}", s1, s2);  // 둘 다 유효
}
```

## 빌림(Borrowing)과 참조(Reference)

값을 사용하면서 소유권을 넘기지 않으려면 **참조(reference)**를 사용한다. 이를 빌림이라고 한다.

### 불변 참조 (`&T`)

```rust
fn calculate_length(s: &String) -> usize {  // &String: String의 참조
    s.len()
    // s는 참조이므로 스코프 종료 시 드롭되지 않음
}

fn main() {
    let s1 = String::from("hello");
    
    // &s1: s1의 참조 생성. 소유권은 main()에 그대로 있음
    let len = calculate_length(&s1);
    
    println!("'{}' 길이: {}", s1, len);  // s1은 여전히 유효!
}
```

참조의 핵심 규칙: **불변 참조는 동시에 여러 개 가질 수 있다.**

```rust
fn main() {
    let s = String::from("hello");
    
    let r1 = &s;  // 불변 참조 1
    let r2 = &s;  // 불변 참조 2
    let r3 = &s;  // 불변 참조 3 — 전부 동시에 OK
    
    println!("{}, {}, {}", r1, r2, r3);  // 정상 동작
}
```

### 가변 참조 (`&mut T`)

```rust
fn change(s: &mut String) {
    s.push_str(", world");
}

fn main() {
    let mut s = String::from("hello");  // mut 선언 필수
    change(&mut s);                      // 가변 참조 전달
    println!("{}", s);  // "hello, world"
}
```

**빌림 검사기의 핵심 규칙**: 특정 시점에 **가변 참조는 단 하나만** 존재할 수 있다. 또한 불변 참조가 살아있는 동안 가변 참조를 만들 수 없다.

```rust
fn main() {
    let mut s = String::from("hello");
    
    let r1 = &s;      // 불변 참조 생성
    let r2 = &s;      // 불변 참조 또 생성 — OK
    // let r3 = &mut s; // 컴파일 에러! r1, r2가 살아있는 동안 가변 참조 불가
    // error[E0502]: cannot borrow `s` as mutable because it is also borrowed as immutable
    
    println!("{}, {}", r1, r2);  // r1, r2 마지막 사용
    // 이 시점 이후 r1, r2의 생존 범위(lifetime) 종료
    
    let r3 = &mut s;  // 이제는 OK — r1, r2가 더 이상 사용되지 않음
    r3.push_str("!");
    println!("{}", r3);
}
```

이 규칙이 **데이터 레이스(Data Race)를 컴파일 타임에 원천 차단**한다. 멀티스레드 프로그램에서 두 스레드가 동시에 같은 데이터에 접근하는데 그 중 하나 이상이 쓰기 작업이면 데이터 레이스가 발생한다. Rust의 빌림 규칙은 정확히 이 상황을 방지한다.

## 라이프타임(Lifetime): 컴파일러가 참조 유효성을 추적하는 법

### Dangling Reference(매달린 참조) 방지

```rust
// 이 함수는 컴파일되지 않는다
fn dangle() -> &String {        // 에러: String의 참조를 반환하려 하지만
    let s = String::from("hello");
    &s                           // s는 이 함수 종료 시 드롭됨
    // error[E0106]: missing lifetime specifier
    // 반환 후 s는 해제됨 → 반환된 참조는 해제된 메모리를 가리키게 됨 (use-after-free)
}

// 올바른 방법: 소유권을 직접 반환
fn no_dangle() -> String {
    let s = String::from("hello");
    s  // 소유권 이동으로 반환 — 드롭되지 않음
}
```

### 명시적 라이프타임 어노테이션

함수가 여러 참조를 받아 참조를 반환할 때, 반환값의 유효 기간이 입력 참조의 유효 기간과 어떻게 연관되는지 컴파일러에게 알려줘야 한다.

```rust
// 'a 는 라이프타임 파라미터 — "이 참조들은 적어도 'a 동안 유효해야 한다"
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
// 반환값의 라이프타임은 x와 y 중 더 짧은 것과 같다

fn main() {
    let string1 = String::from("long string is long");
    let result;
    {
        let string2 = String::from("xyz");
        result = longest(string1.as_str(), string2.as_str());
        println!("가장 긴 문자열: {}", result);  // OK: string2가 여기서 살아있음
    }
    // string2는 여기서 드롭됨
    // println!("{}", result);  // 에러! result는 string2의 수명에 묶여 있음
}
```

라이프타임은 런타임에 존재하지 않는다. 컴파일러가 코드를 분석하는 데만 사용되는 메타데이터다.

## 빌림 검사기 내부 동작 원리

### NLL(Non-Lexical Lifetimes)

Rust 2018 에디션부터 도입된 NLL은 라이프타임이 어휘적 스코프가 아닌 **실제 마지막 사용 시점**에 종료되도록 개선했다.

```rust
fn main() {
    let mut v = vec![1, 2, 3];
    
    // NLL 이전: r의 어휘적 스코프가 블록 끝까지이므로 push 불가
    // NLL 이후: r의 라이프타임은 println! 이후 즉시 종료
    let r = &v[0];
    println!("{}", r);  // r의 마지막 사용 → 여기서 라이프타임 종료
    
    v.push(4);  // NLL 덕분에 OK — r은 이미 사용이 끝남
}
```

### 빌림 검사기의 MIR 분석

Rust 컴파일러는 빌림 검사를 **MIR(Mid-level Intermediate Representation)** 단계에서 수행한다. MIR은 고수준 Rust 코드를 제어 흐름 그래프(CFG)로 표현한 중간 표현이다.

```
소스 코드 → AST → HIR → MIR → (빌림 검사) → LLVM IR → 기계어
                              ^
                              여기서 빌림 검사기가 동작
```

빌림 검사기(Polonius 기반)는 MIR의 각 기본 블록에서 어떤 참조가 살아있는지, 어떤 값이 이동됐는지를 데이터 플로우 분석으로 추적한다.

## 실전 패턴과 일반적인 함정

### 패턴 1: 구조체의 라이프타임

```rust
// 구조체가 참조를 필드로 가질 때 라이프타임 명시 필요
struct Important<'a> {
    content: &'a str,  // 이 참조는 구조체보다 오래 살아야 함
}

impl<'a> Important<'a> {
    fn announce(&self, announcement: &str) -> &str {
        println!("주목! {}", announcement);
        self.content  // self의 라이프타임을 반환값에 적용
    }
}

fn main() {
    let novel = String::from("오래 전 어느 별에서...");
    let first_sentence;
    {
        let i = Important {
            content: novel.split('.').next().expect("'.' 없음"),
        };
        first_sentence = i.announce("새 소설 출간!");
        println!("{}", first_sentence);
    }
}
```

### 패턴 2: Rc와 RefCell — 런타임 빌림 검사

컴파일 타임 빌림 검사가 너무 보수적인 경우, `Rc<RefCell<T>>`를 사용해 런타임으로 검사를 미룰 수 있다.

```rust
use std::rc::Rc;
use std::cell::RefCell;

fn main() {
    // Rc: 참조 카운팅 (단일 스레드 공유 소유권)
    // RefCell: 런타임 가변 빌림 (컴파일 타임 검사 불가한 경우)
    let shared = Rc::new(RefCell::new(vec![1, 2, 3]));
    
    let clone1 = Rc::clone(&shared);
    let clone2 = Rc::clone(&shared);
    
    // borrow_mut()은 런타임에 가변 빌림 규칙 확인
    clone1.borrow_mut().push(4);
    clone2.borrow_mut().push(5);
    
    println!("{:?}", shared.borrow());  // [1, 2, 3, 4, 5]
    
    // 런타임에 두 개의 가변 빌림을 동시에 시도하면 panic!
    // let _r1 = shared.borrow_mut();
    // let _r2 = shared.borrow_mut();  // panic: already mutably borrowed
}
```

멀티스레드 환경에서는 `Arc<Mutex<T>>`를 사용한다. `Arc`는 원자적 참조 카운팅이고 `Mutex`는 OS 수준의 상호 배제다.

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];
    
    for _ in 0..10 {
        let c = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            let mut num = c.lock().unwrap();  // Mutex 획득
            *num += 1;
            // num이 스코프 종료 시 Mutex 자동 해제 (RAII)
        });
        handles.push(handle);
    }
    
    for h in handles { h.join().unwrap(); }
    println!("최종 카운터: {}", *counter.lock().unwrap());  // 10
}
```

## 소유권 시스템의 성능적 의미

Rust의 소유권 모델은 단순히 메모리 안전성만 제공하지 않는다. **결정론적 메모리 해제**로 인해 다음과 같은 성능 이점도 얻는다:

1. **예측 가능한 성능**: GC 일시 정지가 없어 레이턴시가 일정하다.
2. **스택 우선 할당**: 소유권이 명확하면 컴파일러가 힙 할당을 스택 할당으로 최적화할 수 있다.
3. **캐시 친화성**: 불필요한 간접 참조(indirection)를 줄여 캐시 미스를 감소시킨다.
4. **병렬화 안전성**: 빌림 규칙이 데이터 레이스를 컴파일 타임에 방지하므로 안전한 병렬 코드를 더 자신 있게 작성할 수 있다.

## 주의사항과 팁

**팁 1**: `clone()`을 남발하지 않는다. 빌림 검사기를 만족시키기 위해 clone()을 쓰는 것은 빌림 규칙을 제대로 이해하지 못한 신호일 수 있다. 대부분의 경우 참조 구조를 재설계하면 clone() 없이 해결된다.

**팁 2**: 라이프타임 엘리전(Lifetime Elision) 규칙을 활용한다. 단순한 케이스에서는 컴파일러가 라이프타임을 자동으로 추론하므로 명시하지 않아도 된다.

**팁 3**: `unsafe` 블록은 최소화한다. 성능이 진짜 필요한 곳에서만, 그리고 안전 불변식을 직접 증명할 수 있을 때만 사용한다. unsafe 코드는 소유권 시스템이 적용되지 않으므로 직접 메모리 안전을 책임져야 한다.

## 참고 자료

- [The Rust Programming Language Book — 소유권 챕터](https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html)
- [The Rustonomicon — Unsafe Rust와 라이프타임](https://doc.rust-lang.org/nomicon/)
- [Rust Reference — 빌림 검사기 공식 명세](https://doc.rust-lang.org/reference/expressions.html)
- [Polonius: 차세대 Rust 빌림 검사기](https://github.com/rust-lang/polonius)
