---
layout: post
title: "메모리 일관성 모델 완전 정복: x86 TSO, ARM, C++ memory_order로 이해하는 메모리 순서 보장"
date: 2026-10-31
categories: [cs, computer-science]
tags: [메모리모델, memory-ordering, TSO, ARM, 동시성, 멀티코어, C++, 메모리펜스, atomic]
---

## 개요

멀티코어 프로세서에서 두 스레드가 공유 변수를 읽고 쓸 때, 한 스레드가 쓴 값이 다른 스레드에게 언제, 어떤 순서로 보이는가? 이 질문의 답이 **메모리 일관성 모델(Memory Consistency Model)**이다. 단일 CPU에서 직렬로 실행되던 시절에는 자명했던 이 문제가, 멀티코어 시대에는 CPU 파이프라인, 스토어 버퍼, 비순서 실행, 캐시 라인 전파 지연으로 인해 극히 복잡해졌다. x86의 TSO(Total Store Order), ARM/POWER의 완화된 모델, C++11의 `std::atomic` memory_order를 완전히 이해해 보자.

---

## 왜 메모리 순서가 보장되지 않는가?

### 하드웨어 수준에서의 재배열

현대 CPU는 성능을 위해 다음 최적화를 수행한다.

1. **스토어 버퍼(Store Buffer)**: 캐시 미스가 발생한 저장(Store) 연산을 버퍼에 임시 보관하고 다음 명령을 먼저 실행
2. **비순서 실행(Out-of-Order Execution)**: 의존성 없는 명령어들을 데이터 준비 순서대로 병렬 실행
3. **캐시 일관성 지연**: MESI 프로토콜 메시지가 다른 코어에 도달하기까지의 전파 지연
4. **인벌리드큐(Invalidation Queue)**: 캐시 라인 무효화 요청을 버퍼링

이로 인해 **컴파일러가 코드를 작성한 순서**와 **CPU가 실제로 실행하는 순서**와 **다른 코어가 관측하는 순서** 세 가지가 모두 달라질 수 있다.

### 컴파일러 수준에서의 재배열

GCC, Clang, MSVC 모두 의미상 동등하다고 판단되면 명령어 순서를 변경한다. 단일 스레드 관점에서는 등가이지만 멀티스레드에서는 심각한 버그를 유발한다.

```c
// 원래 코드
int data = compute();
ready = 1;  // data를 다른 스레드에 알림

// 컴파일러가 재배열할 수 있음
ready = 1;  // data가 설정되기 전에!
int data = compute();
```

---

## 메모리 재배열의 네 가지 종류

두 인접한 메모리 연산의 관측 가능한 순서 변경은 다음 4종류다.

| 재배열 유형 | 설명 | x86 허용 여부 | ARM 허용 여부 |
|------------|------|---------------|---------------|
| Store → Load | 쓰기 후 읽기 순서 변경 | **O (허용)** | O |
| Load → Load | 읽기 후 읽기 순서 변경 | X (금지) | O |
| Store → Store | 쓰기 후 쓰기 순서 변경 | X | O |
| Load → Store | 읽기 후 쓰기 순서 변경 | X | O |

**x86(TSO)**은 Store→Load 재배열만 허용한다. 이것이 x86이 "강한(strong)" 메모리 모델이라 불리는 이유다.
**ARM/POWER**는 네 가지 모두 허용한다. 명시적 배리어 없이는 어떤 순서도 보장되지 않는다.

---

## x86 TSO(Total Store Order) 모델

x86 TSO는 각 코어에 **FIFO 스토어 버퍼**가 하나씩 있다고 직관적으로 이해할 수 있다.

```
Core 0:        Core 1:
  Store Buf      Store Buf
      ↓              ↓
  +--------------------------+
  |     공유 메모리 (RAM)      |
  +--------------------------+
```

- 코어가 **저장(Store)**을 할 때: 스토어 버퍼에 먼저 기록, 나중에 메모리에 전파
- 코어가 **적재(Load)**를 할 때: 자신의 스토어 버퍼를 먼저 확인, 없으면 메모리에서 읽음

결과적으로 **자신의 최신 Store는 즉시 보임(Store Forwarding)**, 하지만 **다른 코어에서는 아직 보이지 않을 수 있다**.

이것이 **Store→Load 재배열**이 허용되는 이유다.

---

## 구현 예제 1: Store Buffer가 만드는 재배열 (C++로 관찰)

```cpp
#include <atomic>
#include <thread>
#include <iostream>
#include <cassert>

// 순수 int를 사용 — memory_order 없음, 재배열 허용
int x = 0, y = 0;
int r1 = 0, r2 = 0;

void thread0() {
    x = 1;       // S1
    r1 = y;      // L1
}

void thread1() {
    y = 1;       // S2
    r2 = x;      // L2
}

int main() {
    int reorder_count = 0;
    int total = 1'000'000;
    
    for (int i = 0; i < total; ++i) {
        x = y = r1 = r2 = 0;
        
        std::thread t0(thread0);
        std::thread t1(thread1);
        t0.join();
        t1.join();
        
        // 두 스레드가 모두 상대방의 쓰기를 보지 못하는 경우:
        // r1 == 0 && r2 == 0 은 순차 일관성에서 불가능하다.
        // 하지만 TSO에서는 S1이 스토어 버퍼에 있는 동안 L1이 y=0을 읽고,
        // S2가 스토어 버퍼에 있는 동안 L2가 x=0을 읽으면 발생한다.
        if (r1 == 0 && r2 == 0) {
            ++reorder_count;
        }
    }
    
    std::cout << "재배열 발생 횟수: " << reorder_count
              << " / " << total << " ("
              << (100.0 * reorder_count / total) << "%)\n";
    // 실제 실행 시 수 %의 재배열이 관측됨
    return 0;
}
```

위 코드는 **순차 일관성(Sequential Consistency)** 모델에서는 `r1==0 && r2==0`이 절대 발생하지 않는다. 그러나 x86에서도 스토어 버퍼로 인해 실제로 발생한다.

---

## 메모리 펜스(Memory Fence/Barrier)

펜스는 CPU에게 **이 명령 앞/뒤의 메모리 연산 순서를 보장하라**고 명시적으로 지시한다.

| 명령어 | 아키텍처 | 효과 |
|--------|----------|------|
| `MFENCE` | x86 | Store+Load 양방향 순서 보장 |
| `SFENCE` | x86 | Store 순서만 보장 |
| `LFENCE` | x86 | Load 순서만 보장 (+ 투기 실행 방지) |
| `DMB ISH` | ARM | 내부 공유 도메인 양방향 배리어 |
| `DSB` | ARM | 더 강한 보장 (+ 파이프라인 플러시) |
| `ISB` | ARM | 명령 파이프라인 동기화 |

x86에서 `LOCK` 접두사(예: `LOCK XCHG`, `LOCK CMPXCHG`)는 MFENCE를 암묵적으로 포함한다.

---

## C++ memory_order와 Happens-Before

C++11부터 `std::atomic<T>`와 6가지 `memory_order`로 플랫폼 독립적 메모리 순서를 지정할 수 있다.

```
memory_order_relaxed    — 순서 보장 없음, 원자성만 보장
memory_order_consume    — (거의 사용 안 됨, release와 묶임)
memory_order_acquire    — 이후 Load/Store가 앞으로 올라오지 않음
memory_order_release    — 이전 Load/Store가 뒤로 내려가지 않음
memory_order_acq_rel    — acquire + release
memory_order_seq_cst    — 완전 순차 일관성 (기본값)
```

### Acquire-Release 패턴

```cpp
#include <atomic>
#include <thread>
#include <cassert>

std::atomic<int> flag{0};
int data = 0;

void producer() {
    data = 42;                               // (1) data 쓰기
    flag.store(1, std::memory_order_release); // (2) release: (1)이 반드시 앞에
}

void consumer() {
    while (flag.load(std::memory_order_acquire) == 0) {} // (3) acquire
    // (3)이 성공하면 (2)와 synchronized-with 관계가 성립
    // happens-before: (1) → (2) → (3) → (4)
    assert(data == 42);                      // (4) 반드시 42
}

int main() {
    std::thread t1(producer);
    std::thread t2(consumer);
    t1.join();
    t2.join();
}
```

**Synchronized-with**: `release` store가 `acquire` load에 보이면, release 이전의 모든 쓰기가 acquire 이후에 보인다. 이것이 C++ happens-before의 핵심이다.

---

## 구현 예제 2: 세 가지 메모리 모델로 구현한 스핀락

```cpp
#include <atomic>
#include <thread>
#include <vector>
#include <iostream>

class SpinLockRelaxed {
    std::atomic<bool> locked_{false};
public:
    void lock() {
        // 취득 전: 바쁜 대기
        while (true) {
            // relaxed로 확인 (캐시 친화적)
            if (!locked_.load(std::memory_order_relaxed)) {
                // exchange로 취득 시도: acq_rel로 순서 보장
                if (!locked_.exchange(true, std::memory_order_acquire)) {
                    return;
                }
            }
            // CPU 힌트: 하이퍼스레딩 양보
            __builtin_ia32_pause();
        }
    }
    
    void unlock() {
        // release: unlock 이전의 모든 쓰기가 다음 lock 취득자에게 보임
        locked_.store(false, std::memory_order_release);
    }
};

class SpinLockSeqCst {
    std::atomic<bool> locked_{false};
public:
    void lock() {
        while (locked_.exchange(true, std::memory_order_seq_cst)) {
            __builtin_ia32_pause();
        }
    }
    void unlock() {
        locked_.store(false, std::memory_order_seq_cst);
    }
};

// 성능 비교 테스트
template<typename Lock>
long benchmark(int threads, int iterations) {
    Lock lk;
    long counter = 0;
    
    auto worker = [&]() {
        for (int i = 0; i < iterations; ++i) {
            lk.lock();
            ++counter;
            lk.unlock();
        }
    };
    
    std::vector<std::thread> ts;
    for (int i = 0; i < threads; ++i) ts.emplace_back(worker);
    for (auto& t : ts) t.join();
    return counter;
}

int main() {
    const int T = 4, N = 100'000;
    
    auto r1 = benchmark<SpinLockRelaxed>(T, N);
    auto r2 = benchmark<SpinLockSeqCst>(T, N);
    
    std::cout << "Relaxed/Acquire-Release: counter=" << r1
              << (r1 == T * N ? " (정확)" : " (버그!)") << "\n";
    std::cout << "SeqCst: counter=" << r2
              << (r2 == T * N ? " (정확)" : " (버그!)") << "\n";
    return 0;
}
```

x86에서는 두 구현이 생성하는 어셈블리 차이가 적지만, ARM에서 Relaxed 버전이 훨씬 빠르다. `__builtin_ia32_pause()`는 ARM에서 `__asm__ volatile("yield")` 또는 `std::this_thread::yield()`로 대체한다.

---

## 순차 일관성(SC)이 기본값인 이유

`memory_order_seq_cst`는 가장 비용이 비싸지만 가장 이해하기 쉽다. "모든 스레드가 같은 순서로 메모리 연산을 관측한다"는 강한 보장이 프로그램 정확성 추론을 단순하게 만든다.

x86에서 SC store는 `MOV` 대신 `XCHG`나 `MOV+MFENCE`를 사용하고, SC load는 보통 그대로다. ARM에서는 SC store에 `STLR`, SC load에 `LDAR`을 사용하며, 양방향 배리어 효과가 있다.

---

## Happens-Before와 Data Race

C++ 표준에 따르면 **데이터 레이스(Data Race)**는 두 스레드가 동기화 없이 같은 위치를 동시에 접근하고 그중 하나 이상이 쓰기일 때 발생한다. 이는 **정의되지 않은 동작(Undefined Behavior)**이다.

happens-before 관계로 data race를 배제하는 방법:
1. **Mutex**: `unlock()` happens-before `lock()` 성공
2. **Atomic acquire-release**: `release store` happens-before `acquire load`
3. **Thread join**: `thread::join()` happens-before join 이후 코드
4. **Semaphore/Condvar**: 신호 happens-before 대기 해제

---

## 아키텍처별 펜스 매핑 표

| C++ memory_order | x86 | ARM64 |
|-----------------|-----|-------|
| relaxed | (없음) | (없음) |
| acquire load | (없음, TSO로 충분) | `LDAR` |
| release store | (없음, TSO로 충분) | `STLR` |
| seq_cst store | `LOCK XCHG` 또는 `MOV+MFENCE` | `STLR` |
| seq_cst load | `MOV` | `LDAR` |
| acq_rel fence | `MFENCE` | `DMB ISH` |

x86에서 acquire/release가 "공짜"인 이유: TSO가 이미 Load→Load, Load→Store, Store→Store를 금지하기 때문에 소프트웨어 펜스가 필요 없다. 오직 Store→Load(스토어 버퍼 플러시)만 MFENCE가 필요하다.

---

## 주의사항과 팁

1. **가장 약한 모델부터 시작하지 말 것**: `relaxed`는 극히 특수한 경우(통계 카운터 등)에만 안전하다. 처음에는 `seq_cst`로 짜고, 프로파일링 후 필요시에만 약화한다.

2. **Atomic은 레이스의 해결책이 아닌 도구**: `std::atomic<int>`를 쓴다고 해서 프로그램 논리가 자동으로 맞아지지 않는다. happens-before를 직접 설계해야 한다.

3. **ARM에서의 테스트**: x86에서 통과한 코드가 ARM에서 실패하는 경우가 많다. CI에서 ARM(Apple Silicon M시리즈나 AWS Graviton)을 반드시 포함한다.

4. **volatile은 메모리 순서를 보장하지 않는다**: C의 `volatile`은 컴파일러 재배열만 막고 CPU 재배열과 캐시 일관성은 전혀 보장하지 않는다. 멀티스레딩에서 `volatile`만으로는 부족하다.

5. **Lock-free 코드는 리뷰가 어렵다**: acquire-release를 잘못 짠 코드는 드물게만 오작동해 재현이 거의 불가능하다. ThreadSanitizer(`-fsanitize=thread`)를 필수로 사용한다.

6. **C++ memory_order_consume은 사용하지 말 것**: 이론상 `acquire`보다 약하지만 모든 컴파일러가 `acquire`와 동일하게 구현한다. C++26에서 재설계 예정이다.

---

## 참고 자료
- [A Better x86 Memory Model: x86-TSO (Owens, Sarkar, Sewell 논문)](https://www.cl.cam.ac.uk/~pes20/weakmemory/x86tso-paper.tphols.pdf)
- [std::memory_order — cppreference.com](https://en.cppreference.com/w/cpp/atomic/memory_order)
- [Memory Barriers: a Hardware View for Software Hackers (Paul McKenney)](https://www.kernel.org/doc/Documentation/memory-barriers.txt)
- [Preshing on Programming: Memory Ordering](https://preshing.com/20120710/memory-barriers-are-like-source-control-operations/)
