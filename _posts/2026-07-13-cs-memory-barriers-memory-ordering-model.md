---
layout: post
title: "메모리 배리어와 메모리 순서 모델 완전 정복: CPU가 코드를 멋대로 재배열하는 이유"
date: 2026-07-13
categories: [cs, computer-science]
tags: [memory-barrier, memory-ordering, cpu, concurrency, c++, atomic, happens-before, sequential-consistency]
---

멀티스레드 프로그램을 작성할 때 가장 많이 놓치는 함정 중 하나는 **"코드가 작성된 순서대로 실행된다"는 가정**이다. 사실 CPU와 컴파일러는 성능 최적화를 위해 메모리 읽기/쓰기 순서를 적극적으로 바꾼다. 메모리 배리어(Memory Barrier)와 메모리 순서 모델(Memory Ordering Model)은 이 재배열을 제어해 멀티스레드 프로그램의 정확성을 보장하는 핵심 도구다.

## 왜 메모리 연산이 재배열되는가

### 컴파일러 재배열 (Compiler Reordering)

컴파일러는 최적화 과정에서 독립적인 연산의 순서를 자유롭게 바꿀 수 있다. 예를 들어 다음 코드에서:

```c
int x = 0, ready = 0;

// 스레드 A
x = 42;
ready = 1;  // 컴파일러가 x=42보다 먼저 실행할 수 있다

// 스레드 B
while (!ready) {}
int val = x;  // val이 0일 수 있다!
```

컴파일러 관점에서 `x = 42`와 `ready = 1`은 독립적인 연산이므로 순서를 바꿔도 단일 스레드 시맨틱을 위반하지 않는다. 하지만 스레드 B가 `ready == 1`을 보고 `x`를 읽을 때 `x`가 아직 0일 수 있다.

### CPU 재배열 (CPU Reordering)

현대 CPU는 파이프라인 효율성을 높이기 위해 **비순서 실행(Out-of-Order Execution)**을 수행한다. CPU 내부에는 실행 유닛, 로드/스토어 버퍼, 라이트-백 큐 등이 있어 실제 실행 순서가 프로그램 순서와 다를 수 있다.

x86-64는 **TSO(Total Store Order)** 모델을 따라 상대적으로 강한 순서를 보장한다:
- 스토어-스토어 순서: 보장됨
- 로드-로드 순서: 보장됨
- 로드 이후 스토어: **보장됨**
- **스토어 이후 로드**: 재배열 가능 (스토어 버퍼 때문)

반면 ARM은 **Weak Memory Model**을 따라 네 가지 방향 모두 재배열이 가능하다. 이 때문에 x86에서 잘 동작하던 lock-free 코드가 ARM에서 깨지는 경우가 실제로 존재한다.

## 메모리 배리어의 종류

메모리 배리어는 재배열을 막는 울타리(fence)다. 방향에 따라 세 종류로 나뉜다.

| 배리어 종류 | 의미 | 비유 |
|---|---|---|
| **Load Barrier (읽기 장벽)** | 배리어 이전 로드는 이후 로드보다 먼저 완료 | 다음 페이지를 펼치기 전에 이 페이지를 다 읽어야 함 |
| **Store Barrier (쓰기 장벽)** | 배리어 이전 스토어는 이후 스토어보다 먼저 완료 | 다음 작업 시작 전 이 작업을 완료해야 함 |
| **Full Barrier (전체 장벽)** | 배리어를 기준으로 양방향 재배열 금지 | 검문소: 앞의 차가 모두 지나간 뒤 뒤의 차가 진입 |

## C++의 메모리 순서 모델

C++11부터 `std::atomic`과 `std::memory_order`로 메모리 순서를 명시적으로 제어할 수 있다.

```
memory_order_seq_cst   (순차 일관성, 가장 강함)
memory_order_acq_rel   (acquire + release)
memory_order_acquire   (읽기 장벽)
memory_order_release   (쓰기 장벽)
memory_order_consume   (데이터 의존 순서, 거의 사용 안 함)
memory_order_relaxed   (재배열 제한 없음, 가장 약함)
```

### Happens-Before 관계

C++ 메모리 모델의 핵심은 **happens-before** 관계다. `A happens-before B`가 성립하면, B가 실행되기 전에 A의 모든 효과가 B에게 보인다.

`release`와 `acquire`의 쌍이 이 관계를 형성한다:
- 스레드 A에서 `store(val, release)`
- 스레드 B에서 `load(acquire)`가 A의 값을 읽으면
- A의 release store **이전** 모든 쓰기가 B의 acquire load **이후**에 보임

## 실제 구현 예제

### 예제 1: memory_order로 제어하는 생산자-소비자 패턴 (C++)

```cpp
#include <atomic>
#include <thread>
#include <iostream>
#include <cassert>

std::atomic<int> data{0};
std::atomic<bool> ready{false};

void producer() {
    data.store(42, std::memory_order_relaxed);
    // release: 위의 store(data=42)가 반드시 먼저 완료된 뒤
    // 이 플래그를 발행한다
    ready.store(true, std::memory_order_release);
}

void consumer() {
    // acquire: 이 load가 true를 읽으면
    // producer의 release 이전 쓰기가 모두 보임
    while (!ready.load(std::memory_order_acquire)) {
        std::this_thread::yield();
    }
    // 이 시점에서 data == 42 가 보장됨
    assert(data.load(std::memory_order_relaxed) == 42);
    std::cout << "data: " << data << std::endl;
}

int main() {
    std::thread t1(producer);
    std::thread t2(consumer);
    t1.join();
    t2.join();
    return 0;
}
// 컴파일: g++ -O2 -std=c++17 -pthread -o fence_demo fence_demo.cpp
```

`data` 읽기/쓰기에 `relaxed`를 사용해도 `ready`의 `release`/`acquire` 쌍이 happens-before를 형성하므로 안전하다. 모든 연산에 `seq_cst`를 쓰는 것보다 훨씬 효율적이다.

### 예제 2: 이중 검사 잠금(DCLP)의 올바른 구현 (C++)

```cpp
#include <atomic>
#include <mutex>
#include <memory>
#include <iostream>

class Singleton {
public:
    static Singleton* getInstance() {
        // 첫 번째 검사: 잠금 없이 acquire load
        Singleton* ptr = instance_.load(std::memory_order_acquire);
        if (ptr == nullptr) {
            std::lock_guard<std::mutex> lock(mutex_);
            // 두 번째 검사: 잠금 획득 후 재확인
            ptr = instance_.load(std::memory_order_relaxed);
            if (ptr == nullptr) {
                ptr = new Singleton();
                // release: 생성자 완료 후 포인터 발행
                instance_.store(ptr, std::memory_order_release);
            }
        }
        return ptr;
    }

    void greet() { std::cout << "Singleton instance @ " << this << std::endl; }

private:
    Singleton() = default;
    static std::atomic<Singleton*> instance_;
    static std::mutex mutex_;
};

std::atomic<Singleton*> Singleton::instance_{nullptr};
std::mutex Singleton::mutex_;

// 잘못된 구현 예시 (volatile만 사용한 경우)
// volatile은 컴파일러 최적화는 막지만 CPU 재배열은 막지 않는다!
class BadSingleton {
public:
    static BadSingleton* getInstance() {
        if (instance_ == nullptr) {               // ← 여기서 반쯤 초기화된 포인터를 볼 수 있음
            static std::mutex m;
            std::lock_guard<std::mutex> lock(m);
            if (instance_ == nullptr) {
                instance_ = new BadSingleton();   // ARM에서 포인터 쓰기가 생성자보다 먼저 보일 수 있음
            }
        }
        return instance_;
    }
private:
    static volatile BadSingleton* instance_;
};
volatile BadSingleton* BadSingleton::instance_ = nullptr;

int main() {
    Singleton::getInstance()->greet();
    Singleton::getInstance()->greet();
    return 0;
}
```

`volatile`은 단일 스레드 환경의 컴파일러 최적화만 막는다. 멀티스레드 환경에서 메모리 순서를 보장하려면 반드시 `std::atomic`을 사용해야 한다.

## 순차 일관성(Sequential Consistency)이란

`memory_order_seq_cst`(기본값)는 가장 강한 보장을 제공한다. 모든 스레드가 **전역적으로 동일한 순서로 원자적 연산을 관찰**한다는 것을 의미한다. 이 모델은 개발자가 이해하기 가장 쉽지만, 강한 동기화 명령(x86의 `MFENCE`, ARM의 `dmb ish`)을 생성해 성능 비용이 크다.

실제 성능 차이는 하드웨어에 따라 크게 다르다. x86에서는 `seq_cst`와 `acquire/release`의 성능 차이가 크지 않지만, ARM에서는 `relaxed` 연산이 `seq_cst`보다 수배 빠를 수 있다.

## 언어별 메모리 모델 요약

| 언어 | 모델 | 특징 |
|---|---|---|
| C++ (C++11+) | DRF-SC | data-race-free 코드에 seq. consistency 보장 |
| Java | JMM | volatile로 happens-before, synchronized로 모니터 락 |
| Go | Go Memory Model | channel, sync.Mutex, atomic으로 happens-before |
| Rust | C++ 모델 차용 | `std::sync::atomic::Ordering`으로 동일한 6가지 순서 |

Java의 `volatile`은 C++의 `seq_cst`와 유사하며, Go의 채널 통신은 묵시적으로 happens-before를 형성한다.

## 주의사항 및 팁

**"단순히 seq_cst를 쓰면 안전하지 않나요?"** — 안전하지만 불필요한 성능 비용이 발생한다. 핫 패스에서 고성능이 필요하다면 acquire/release 쌍을 쓰고, 그것도 과도하다면 relaxed + 명시적 fence를 고려한다.

**`volatile`을 메모리 배리어로 착각하지 말 것** — C/C++의 `volatile`은 컴파일러 최적화만 막는다. ARM 등 약한 메모리 모델 CPU에서 volatile만으로는 재배열을 막을 수 없다. 반드시 `std::atomic` 또는 명시적 배리어를 사용하라.

**데이터 레이스는 정의되지 않은 동작(UB)** — C++에서 두 스레드가 동기화 없이 같은 메모리를 읽고 쓰면 Undefined Behavior다. AddressSanitizer의 `-fsanitize=thread` 옵션(TSan)으로 데이터 레이스를 검출할 수 있다.

**Consume은 피할 것** — `memory_order_consume`은 데이터 의존 순서만 보장하는 최약한 모델이지만, 구현이 어려워 현재 모든 주요 컴파일러가 `acquire`로 업그레이드해 처리한다.

## 참고 자료
- [std::memory_order - cppreference.com](https://en.cppreference.com/cpp/atomic/memory_order)
- [Memory barrier - Wikipedia](https://en.wikipedia.org/wiki/Memory_barrier)
- [Memory Barriers Are Like Source Control Operations - Preshing on Programming](https://preshing.com/20120710/memory-barriers-are-like-source-control-operations/)
- [Memory ordering - Wikipedia](https://en.wikipedia.org/wiki/Memory_ordering)
