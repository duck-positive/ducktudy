---
layout: post
title: "Michael-Scott 무잠금 큐와 ABA 문제: CAS로 구현하는 비차단 자료구조"
date: 2026-11-11
categories: [cs, computer-science]
tags: [lock-free, concurrent, queue, michael-scott, aba-problem, cas, hazard-pointers, rust, java]
---

다중 스레드 환경에서 공유 자료구조에 접근할 때 뮤텍스를 사용하면 안전하지만, 경쟁이 심할수록 성능이 급격히 저하됩니다. 뮤텍스를 보유한 스레드가 스케줄러에 의해 선점(preempted)되면 다른 모든 스레드가 블록되기 때문입니다. **무잠금(lock-free) 자료구조**는 이 문제를 CAS(Compare-And-Swap) 같은 원자적 명령어로 해결하여, 뮤텍스 없이도 스레드 안전성을 보장합니다.

1996년 Maged Michael과 Michael Scott이 PODC(Principles of Distributed Computing)에서 발표한 **Michael-Scott 큐**는 가장 영향력 있는 무잠금 FIFO 큐 알고리즘입니다. Java의 `ConcurrentLinkedQueue`, .NET의 `ConcurrentQueue`, 그리고 많은 메시지 패싱 시스템의 내부에서 이 알고리즘이 활용됩니다.

## CAS(Compare-And-Swap)의 기초

Michael-Scott 큐를 이해하려면 먼저 CAS를 이해해야 합니다. CAS는 현대 CPU가 하드웨어 수준에서 지원하는 원자적 연산으로, 세 개의 인수를 받습니다:

```
CAS(메모리_위치, 예상_값, 새_값) → 성공_여부
```

- 메모리 위치의 현재 값이 예상 값과 같으면 → 새 값으로 교체하고 `true` 반환
- 그렇지 않으면 → 아무것도 하지 않고 `false` 반환

이 연산은 원자적(atomic)으로 수행되므로, 두 스레드가 동시에 CAS를 수행할 때 반드시 하나만 성공하고 나머지는 실패합니다. 무잠금 알고리즘은 이 성질을 이용하여 "실패 시 재시도" 패턴으로 뮤텍스를 대체합니다.

## Michael-Scott 큐의 구조

Michael-Scott 큐는 연결 리스트(linked list)를 기반으로 합니다:

- **sentinel 노드**: 항상 큐의 앞에 존재하는 더미 노드
- **head 포인터**: sentinel을 가리키며, 디큐 시 갱신
- **tail 포인터**: 마지막 노드를 가리키며, 인큐 시 갱신
- **각 노드의 next 포인터**: 다음 노드를 가리키는 원자적 포인터

핵심 불변식(invariant):
1. `tail.next == null`이면 tail이 실제로 마지막 노드 (일관된 상태)
2. `tail.next != null`이면 다른 스레드가 인큐 중 (불일치 상태, 보조 필요)

## 코드 예제 1: C++17로 구현하는 Michael-Scott 큐

```cpp
#include <atomic>
#include <memory>
#include <optional>

template<typename T>
class MichaelScottQueue {
private:
    struct Node {
        T value;
        std::atomic<Node*> next{nullptr};
        
        Node() = default;
        explicit Node(T val) : value(std::move(val)) {}
    };

    std::atomic<Node*> head;
    std::atomic<Node*> tail;

public:
    MichaelScottQueue() {
        // sentinel 노드 생성
        Node* sentinel = new Node();
        head.store(sentinel, std::memory_order_relaxed);
        tail.store(sentinel, std::memory_order_relaxed);
    }

    ~MichaelScottQueue() {
        // 모든 노드 해제 (단일 스레드 정리 시 사용)
        Node* curr = head.load(std::memory_order_relaxed);
        while (curr) {
            Node* next = curr->next.load(std::memory_order_relaxed);
            delete curr;
            curr = next;
        }
    }

    // 인큐: 새 노드를 tail에 추가
    void enqueue(T value) {
        Node* new_node = new Node(std::move(value));
        
        while (true) {
            Node* t = tail.load(std::memory_order_acquire);
            Node* next = t->next.load(std::memory_order_acquire);
            
            // tail이 여전히 마지막 노드인지 확인
            if (t != tail.load(std::memory_order_acquire)) continue;
            
            if (next == nullptr) {
                // tail이 실제로 마지막 → new_node를 next로 연결 시도
                if (t->next.compare_exchange_weak(
                        next, new_node,
                        std::memory_order_release,
                        std::memory_order_relaxed)) {
                    // CAS 성공: tail 포인터를 new_node로 갱신 (실패해도 무방)
                    tail.compare_exchange_weak(t, new_node,
                        std::memory_order_release,
                        std::memory_order_relaxed);
                    return;
                }
            } else {
                // tail이 뒤처져 있음: 다른 스레드가 인큐 중
                // tail을 앞으로 당겨 주는 보조(help) 수행
                tail.compare_exchange_weak(t, next,
                    std::memory_order_release,
                    std::memory_order_relaxed);
            }
        }
    }

    // 디큐: head의 다음 노드를 꺼내 반환
    std::optional<T> dequeue() {
        while (true) {
            Node* h = head.load(std::memory_order_acquire);
            Node* t = tail.load(std::memory_order_acquire);
            Node* next = h->next.load(std::memory_order_acquire);
            
            if (h != head.load(std::memory_order_acquire)) continue;
            
            if (h == t) {
                // 큐가 비었거나 tail이 뒤처진 상태
                if (next == nullptr) {
                    return std::nullopt;  // 큐 비어 있음
                }
                // tail 보조
                tail.compare_exchange_weak(t, next,
                    std::memory_order_release,
                    std::memory_order_relaxed);
            } else {
                // next에서 값 읽기 (CAS 성공 전에 읽어야 함)
                T value = next->value;
                // head를 next로 이동 (sentinel 역할을 next로 위임)
                if (head.compare_exchange_weak(h, next,
                        std::memory_order_release,
                        std::memory_order_relaxed)) {
                    // 이전 head(= 이전 sentinel) 해제는 Hazard Pointer로 처리해야 안전
                    delete h;  // 주의: ABA 문제 발생 가능 (아래 설명 참고)
                    return value;
                }
            }
        }
    }
};
```

## ABA 문제: 무잠금 알고리즘의 가장 큰 함정

위 코드의 `delete h` 줄에 주석으로 경고를 남겼습니다. 이 지점에서 **ABA 문제**가 발생할 수 있습니다.

### ABA 문제란?

스레드 T1이 CAS 연산을 수행하려는데, 조건을 확인하고 실제 CAS를 수행하는 사이에 다른 스레드들이 값을 A → B → A로 변경하면, T1은 값이 여전히 A인 것을 보고 CAS를 성공시키지만, 실제로는 상태가 변경된 후입니다.

구체적 시나리오:

```
초기 상태: head -> [sentinel] -> [A] -> [B] -> null, tail -> [B]

스레드 T1: 디큐 시작
  → h = head = sentinel, next = A, value = A.value
  → (선점됨: 스케줄러가 T1을 멈춤)

스레드 T2: 디큐 → A 반환, [sentinel] 삭제, head -> [A] (새 sentinel)
스레드 T2: 디큐 → B 반환, [A] 삭제, head -> [B] (새 sentinel)
스레드 T2: [sentinel] 메모리 해제 → 그 주소를 메모리 풀에 반환
스레드 T3: 새 노드 C 인큐 → 메모리 할당 시 우연히 [sentinel]의 옛 주소를 받음
  → head -> [B] -> [C], tail -> [C]

스레드 T1: 재개
  → CAS(head, old_sentinel_ptr, A_ptr)
  → old_sentinel_ptr == head의 현재 값(= B)? → 아니다! CAS 실패
  (이 경우는 괜찮음)

하지만 만약 T3의 C 노드가 old_sentinel_ptr와 같은 주소를 받았다면?
  → head -> [B] -> [C=old_sentinel_ptr_재활용], tail -> [C]
  → T1의 CAS(head, old_sentinel_ptr, A_ptr) 성공!
  → head = A_ptr (이미 해제된 메모리!)
  → 메모리 오염(use-after-free)
```

이처럼 ABA 문제는 메모리를 직접 관리하는 환경(C/C++)에서 무잠금 자료구조가 해제된 메모리를 재활용할 때 발생합니다. 가비지 컬렉션(GC)이 있는 언어(Java, Go)에서는 GC가 메모리 재활용 시점을 제어하므로 ABA 문제가 대부분 발생하지 않습니다. 이것이 `java.util.concurrent.ConcurrentLinkedQueue`가 C++ 버전보다 구현이 단순한 이유 중 하나입니다.

### ABA 해결책 1: 태그드 포인터 (Tagged Pointer)

포인터와 함께 버전 카운터(version counter)를 원자적으로 저장합니다. 128비트 DCAS(Double-Width CAS) 또는 플랫폼별 DWCAS 명령어를 활용합니다:

```cpp
// 64비트 포인터 + 64비트 카운터를 __int128으로 패킹
struct TaggedPointer {
    Node* ptr;
    uint64_t tag;
};

std::atomic<TaggedPointer> head;

// CAS 시 tag도 함께 비교하므로 동일 주소라도 다른 태그면 실패
bool cas_head(TaggedPointer expected, Node* new_ptr) {
    TaggedPointer new_val{new_ptr, expected.tag + 1};
    return head.compare_exchange_weak(expected, new_val, ...);
}
```

태그 카운터가 오버플로우되면 ABA가 재발할 수 있으나, 64비트 카운터를 사용하면 현실적으로 불가능합니다.

## 코드 예제 2: Rust에서 crossbeam을 활용한 무잠금 큐

Rust에서는 `crossbeam` 라이브러리가 `SegQueue`(무잠금 세그먼트 큐)를 제공합니다. 내부적으로 Epoch 기반 메모리 회수(EBR)로 ABA 문제를 해결합니다:

```rust
use crossbeam::queue::SegQueue;
use std::sync::Arc;
use std::thread;

fn main() {
    // Arc로 여러 스레드에서 공유
    let queue = Arc::new(SegQueue::new());
    let mut handles = vec![];

    // 생산자 스레드 4개
    for i in 0..4 {
        let q = Arc::clone(&queue);
        handles.push(thread::spawn(move || {
            for j in 0..1000 {
                q.push(i * 1000 + j);
            }
        }));
    }

    // 소비자 스레드 2개
    let total_expected = 4 * 1000;
    let counter = Arc::new(std::sync::atomic::AtomicUsize::new(0));
    
    for _ in 0..2 {
        let q = Arc::clone(&queue);
        let cnt = Arc::clone(&counter);
        handles.push(thread::spawn(move || {
            loop {
                match q.pop() {
                    Some(_val) => {
                        cnt.fetch_add(1, std::sync::atomic::Ordering::Relaxed);
                    }
                    None => {
                        // 큐가 일시적으로 비어 있을 수 있음 (스핀 대기)
                        if cnt.load(std::sync::atomic::Ordering::Relaxed) >= total_expected {
                            break;
                        }
                        thread::yield_now();
                    }
                }
            }
        }));
    }

    for h in handles {
        h.join().unwrap();
    }

    println!("모든 항목 처리 완료. 남은 항목: {}", queue.len());
}
```

### Epoch 기반 메모리 회수 (EBR)

`crossbeam`은 ABA 문제를 **Epoch 기반 메모리 회수(Epoch-Based Reclamation)**로 해결합니다:

1. 전역 에폭(epoch) 카운터와 각 스레드의 로컬 에폭을 유지
2. 스레드가 임계 구역(critical section)에 진입할 때 현재 전역 에폭을 로컬에 기록
3. 노드를 삭제할 때 즉시 `delete`하지 않고, 현재 에폭에 연결된 "쓰레기 목록"에 추가
4. 모든 활성 스레드의 로컬 에폭이 삭제 시의 에폭을 넘어서면 그때 실제로 메모리 해제
5. 이 시점에는 어떤 스레드도 그 노드를 참조하지 않음이 보장됨

이 방식은 가비지 컬렉션의 아이디어를 무잠금 알고리즘에 적용한 것으로, ABA 문제를 완전히 해결하면서도 성능 오버헤드가 작습니다.

## Hazard Pointers: ABA 해결의 또 다른 방법

Michael이 2002년에 제안한 **Hazard Pointers**는 각 스레드가 현재 접근 중인 노드를 "위험 포인터" 목록에 등록하는 방식입니다:

```cpp
// 개념 의사 코드
thread_local HazardPointer hp[MAX_HAZARD];

T* read_ptr(std::atomic<T*>& p) {
    while (true) {
        T* ptr = p.load(std::memory_order_acquire);
        hp[0].store(ptr, std::memory_order_release);  // 위험 포인터 등록
        // 등록 후 포인터가 변경되지 않았는지 재확인
        if (ptr == p.load(std::memory_order_acquire)) {
            return ptr;  // 안전하게 사용 가능
        }
    }
}

void retire(T* ptr) {
    // 모든 스레드의 Hazard Pointer 목록에 ptr이 없으면 즉시 삭제
    // 있으면 "은퇴 목록"에 추가 후 나중에 삭제
}
```

Hazard Pointers는 EBR보다 메모리 효율이 좋지만, 구현이 복잡하고 위험 포인터 등록/해제로 인한 메모리 배리어 오버헤드가 있습니다.

## 무잠금 vs 뮤텍스 성능 비교

| 항목 | 뮤텍스 큐 | 무잠금 큐 |
|------|-----------|----------|
| 경쟁 없음 | 빠름 | 약간 빠름 |
| 경쟁 낮음 | 적당 | 빠름 |
| 경쟁 높음 | 급격히 느려짐 | 더 나은 확장성 |
| 스레드 선점 | 나머지 블록 | 진행 보장 |
| 구현 복잡도 | 낮음 | 높음 |
| 메모리 관리 | 단순 | 복잡(ABA) |

무잠금 자료구조는 스레드가 많거나 경쟁이 심한 환경, 또는 실시간 시스템처럼 블로킹을 허용하지 않는 환경에서 뮤텍스보다 유리합니다. 그러나 경쟁이 적은 환경에서는 뮤텍스 오버헤드가 작아 차이가 미미합니다.

## 주의사항과 팁

### 1. "무잠금"이 "빠름"을 의미하지는 않음

무잠금 자료구조는 최악의 경우 블로킹을 방지하는 **진행 보장(progress guarantee)**이 목적입니다. 경쟁이 없는 환경에서는 단순한 뮤텍스보다 느릴 수도 있습니다. 항상 프로파일링으로 검증하세요.

### 2. 메모리 순서(Memory Ordering) 주의

무잠금 코드에서 `memory_order_acquire`, `memory_order_release` 등을 잘못 지정하면 소리 없이 데이터 레이스가 발생합니다. 의심스러울 때는 `memory_order_seq_cst`(순차 일관성)를 사용하고, 성능이 중요한 경우에만 완화된 순서를 적용하세요.

### 3. 스핀 루프의 위험성

CAS 실패 시 무한 루프로 재시도하면 단일 스레드 환경이나 스레드 수 > CPU 코어 수인 환경에서 CPU를 낭비합니다. `std::thread::yield_now()` 또는 적절한 백오프(exponential backoff) 전략을 사용하세요.

### 4. 직접 구현보다 검증된 라이브러리 선호

무잠금 자료구조는 미묘한 버그가 많습니다. Rust에서는 `crossbeam`, Java에서는 `java.util.concurrent`의 구현처럼 검증된 라이브러리를 사용하는 것이 훨씬 안전합니다.

### 5. ABA 문제는 반드시 고려할 것

가비지 컬렉션이 없는 언어(C, C++, Rust)에서 직접 무잠금 자료구조를 구현한다면, 메모리 회수 전략(EBR, Hazard Pointers, Tagged Pointers 등)을 반드시 포함해야 합니다. 이를 빠뜨리면 간헐적인 메모리 오염 버그로 디버깅에 수백 시간이 걸릴 수 있습니다.

Michael-Scott 큐는 발표된 지 30년이 가까워지는 오늘날에도 여전히 현역으로 사용되는 알고리즘입니다. CAS 하나로 뮤텍스의 한계를 넘어선 이 아이디어는 현대 고성능 동시성 시스템의 근간이 되었으며, 무잠금 프로그래밍을 배우는 첫 걸음으로 가장 좋은 사례입니다.

## 참고 자료

- [crossbeam: Rust 고성능 동시성 라이브러리](https://github.com/crossbeam-rs/crossbeam)
- [rayon: Rust 데이터 병렬 처리 라이브러리](https://github.com/rayon-rs/rayon)
