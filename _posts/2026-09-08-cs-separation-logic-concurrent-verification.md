---
layout: post
title: "분리 논리(Separation Logic) 완전 정복: 포인터 안전성과 동시성을 수학적으로 증명하는 법"
date: 2026-09-08
categories: [cs, computer-science]
tags: [separation-logic, Hoare-logic, concurrent-verification, memory-safety, formal-verification, Iris, Coq, Rust]
---

## 개념 설명

**분리 논리(Separation Logic)**는 John C. Reynolds와 Peter O'Hearn이 2002년 제안한 **Hoare 논리의 확장**이다. 포인터와 동적 메모리를 다루는 프로그램의 정확성을 수학적으로 증명하기 위해 설계되었다. 전통적인 Hoare 논리가 힙(heap) 메모리를 다루기 어렵다는 한계를 극복한다.

### Hoare 논리의 한계

고전적인 Hoare 논리에서 프로그램 검증은 삼중쌍(triple)으로 표현된다.

```
{P} C {Q}
```

"사전 조건 P가 성립할 때, 명령 C를 실행하면 사후 조건 Q가 성립한다"는 의미다.

하지만 힙 메모리와 포인터를 다룰 때 문제가 생긴다.

```c
// 두 포인터가 같은 메모리를 가리키면?
void increment(int *x, int *y) {
    *x += 1;
    *y += 1;
    // *x == *y + 1을 가정했지만 x == y이면 *x == *y가 된다!
}
```

이런 **앨리어싱(aliasing)** 문제를 다루려면 전통 Hoare 논리에서는 "x ≠ y"같은 조건을 모든 사전 조건에 명시해야 한다. 코드가 복잡해질수록 이런 조건이 폭발적으로 늘어난다.

### 분리 합성(Separating Conjunction)

분리 논리의 핵심은 새로운 논리 연결자 **`*` (separating conjunction)**다.

```
P * Q
```

이는 "힙이 두 **분리된** 부분으로 나뉘며, 한쪽에서는 P가 성립하고 다른 쪽에서는 Q가 성립한다"를 뜻한다. 두 술어(predicate)가 **서로 겹치지 않는 메모리 영역**에 대해 참이라는 것을 보장한다.

기본 개념들:

| 표기 | 의미 |
|------|------|
| `emp` | 힙이 비어있다 |
| `x ↦ v` | 주소 x에 값 v가 저장되어 있다 (정확히 이 한 셀) |
| `P * Q` | P와 Q가 서로 다른(분리된) 힙 영역에서 성립 |
| `P -* Q` | 현재 힙에 P를 만족하는 영역을 합치면 Q가 성립 (마법 지팡이, magic wand) |

**분리 합성의 힘**: `x ↦ 1 * y ↦ 2`를 증명하면 x와 y가 **반드시 다른 주소**임이 자동으로 보장된다. 앨리어싱 조건을 별도로 명시할 필요가 없다.

### 주요 증명 규칙

**로컬 액션(Local Action) — 프레임 규칙**:

```
    {P} C {Q}
───────────────────── (Frame Rule)
  {P * R} C {Q * R}
```

프로그램 C가 P가 성립하는 힙 영역만 수정할 때, C를 실행해도 나머지 영역 R은 그대로 유지된다는 것을 보장한다. 이것이 분리 논리의 **모듈성(modularity)** 핵심이다.

**힙 할당**:

```
{emp} x := alloc(v) {x ↦ v}
```

**힙 읽기**:

```
{x ↦ v} y := *x {x ↦ v ∧ y = v}
```

**힙 쓰기**:

```
{x ↦ _} *x := v {x ↦ v}
```

**힙 해제**:

```
{x ↦ _} free(x) {emp}
```

---

## 왜 필요한가

### 메모리 안전성 오류의 현실

메모리 관련 버그는 소프트웨어 취약점의 약 70%를 차지한다(Microsoft 통계). 주요 유형:

- **Use-after-free**: 해제된 메모리에 접근
- **Double-free**: 같은 메모리를 두 번 해제
- **Buffer overflow**: 배열 경계 초과 접근
- **Null dereference**: 널 포인터 역참조
- **Data race**: 동시 접근으로 인한 경쟁 조건

분리 논리는 이런 오류들을 **컴파일 타임 또는 검증 타임**에 수학적으로 배제할 수 있다.

### Rust의 소유권 시스템과 분리 논리

Rust의 **소유권(ownership) + 빌림(borrowing)** 시스템은 분리 논리의 직관을 타입 시스템으로 구현한 것으로 볼 수 있다.

- `T` (소유): `x ↦ v` — 유일한 소유
- `&mut T` (가변 빌림): 분리된 읽기-쓰기 접근
- `&T` (불변 빌림): 공유 읽기 (여러 참조 가능)

Rust 컴파일러의 **borrow checker**는 사실상 분리 논리의 자동 증명기다.

---

## 실제 구현 예제

### 예제 1: 연결 리스트 검증 (C 의사코드 + 분리 논리 주석)

```c
/*
 * 분리 논리로 연결 리스트 append 함수 검증
 *
 * 리스트 술어 정의:
 *   list(p, xs) — p가 값 xs를 담은 단일 연결 리스트의 첫 노드 주소
 *
 *   list(NULL, []) = emp
 *   list(p, x::xs) = ∃q. p ↦ (x, q) * list(q, xs)
 *
 * append의 명세:
 *   {list(p, xs) * list(q, ys)} append(p, q) {list(ret, xs ++ ys)}
 */

typedef struct Node {
    int val;
    struct Node *next;
} Node;

// 사전 조건: {list(p, xs) * list(q, ys)}
// 사후 조건: {list(결과, xs ++ ys)}
Node *append(Node *p, Node *q) {
    if (p == NULL) {
        // xs = [], 따라서 xs ++ ys = ys
        // {list(q, ys)} → list(q, [] ++ ys) = list(q, ys) ✓
        return q;
    }
    // p ↦ (p->val, p->next) * list(p->next, xs') * list(q, ys)
    // 여기서 xs = p->val :: xs'

    // 재귀 호출:
    // {list(p->next, xs') * list(q, ys)}
    // append(p->next, q)
    // {list(결과', xs' ++ ys)}
    p->next = append(p->next, q);

    // 프레임 규칙 적용:
    // {p ↦ (p->val, 결과')} * {list(결과', xs' ++ ys)}
    // = list(p, p->val :: xs' ++ ys)
    // = list(p, xs ++ ys) ✓
    return p;
}

/*
 * 검증 핵심 포인트:
 * 1. 재귀 호출 전후 프레임 규칙으로 p 노드의 소유권 보존 자동 증명
 * 2. NULL 체크가 list 술어의 기저 사례(emp)와 정확히 매핑
 * 3. p->next 수정은 p ↦ _ 영역만 변경 — 다른 노드 불변
 */
```

### 예제 2: 동시 분리 논리(CSL) — 락 기반 공유 자원 검증

```rust
// Rust로 구현한 뮤텍스 보호 카운터
// 동시 분리 논리(Concurrent Separation Logic) 관점에서 주석 포함
use std::sync::{Arc, Mutex};
use std::thread;

/*
 * [CSL 불변식 정의]
 * 뮤텍스 M에 대한 불변식 I:
 *   I(counter) = counter ↦ n ∧ n ≥ 0
 *
 * 락 획득 규칙:
 *   {owns_lock(M)} acquire(M) {I(counter) * owns_lock(M)}
 *
 * 락 해제 규칙:
 *   {I(counter) * owns_lock(M)} release(M) {emp}
 *
 * 분리 논리의 보장:
 * - 락 없이 counter에 접근하는 코드는 컴파일 타임에 거부(Rust borrow checker)
 * - 두 스레드가 동시에 counter를 수정하는 것은 불가능 (불변식 보장)
 */

fn concurrent_counter_verified() {
    // Arc<Mutex<i32>>: 소유권 공유(Arc) + 상호 배제(Mutex)
    // CSL에서: shared_ptr = {M : Mutex | I(counter)}
    let counter = Arc::new(Mutex::new(0i32));

    let handles: Vec<_> = (0..10).map(|i| {
        let counter_clone = Arc::clone(&counter);
        thread::spawn(move || {
            // [CSL: {own(counter_clone)} acquire(M) {I * own}]
            let mut guard = counter_clone.lock().unwrap();

            // 이 시점: guard는 counter ↦ n 의 독점적 소유권
            // 다른 스레드는 이 영역에 접근 불가 (CSL 프레임 규칙)
            *guard += 1;
            println!("스레드 {}: counter = {}", i, *guard);

            // [CSL: {I * own} release(M) {emp}]
            // guard 드롭 시 자동으로 락 해제 + 불변식 복원
        })
        // Rust: guard의 범위(scope)가 CSL의 임계 구역과 1:1 대응
    }).collect();

    for h in handles {
        h.join().unwrap();
    }

    // 최종 값은 반드시 10 (CSL로 증명 가능)
    println!("최종 counter = {}", *counter.lock().unwrap());
}

/*
 * CSL이 보장하는 것:
 * 1. Data race 자유 (race freedom): 락 없이는 공유 데이터에 접근 불가
 * 2. 불변식 보존: 락 해제 시 항상 I가 성립
 * 3. 교착 상태 자유(deadlock freedom): 락 획득 순서를 CSL 레벨에서 증명 가능
 *
 * Rust + Tokio의 async/await도 유사한 소유권 기반 보장 제공
 */

fn main() {
    concurrent_counter_verified();
}
```

---

## 주의사항 및 팁

### 1. 분리 논리 도구 생태계

| 도구 | 언어 | 특징 |
|------|------|------|
| **Iris** | Coq | 고계 동시 분리 논리, 가장 강력한 이론 기반 |
| **VeriFast** | C/Java | 실용적 C 프로그램 검증, 산업 적용 사례 있음 |
| **RustBelt** | Rust (Coq 증명) | Rust 타입 시스템의 안전성을 Iris로 기계 증명 |
| **Infer** | C/Java/ObjC | Meta의 정적 분석 도구, 분리 논리 기반, Facebook 프로덕션 사용 |
| **Bedrock2** | C (Coq) | MIT의 RISC-V 커널 검증 프로젝트 |

### 2. Meta Infer — 프로덕션 적용 사례

**Meta Infer**는 분리 논리를 기반으로 한 **자동화된 정적 분석기**다. Null 역참조, use-after-free, 메모리 누수를 자동 탐지하여 2013년부터 Facebook/Instagram/WhatsApp 코드베이스에 적용 중이다. 분리 논리의 **framing**과 **bi-abduction** 기법으로 복잡한 포인터 코드도 분석할 수 있다.

```bash
# Infer로 C 코드 분석
infer run -- gcc -c mycode.c

# 보고서 예시:
# mycode.c:42: error: NULL_DEREFERENCE
#   pointer p could be null at line 42.
# mycode.c:87: error: USE_AFTER_FREE
#   memory was freed at line 85.
```

### 3. 프레임 규칙의 한계

분리 논리의 프레임 규칙은 **로컬 액션(local actions)** 가정 하에 성립한다. 즉, 프로그램이 사전 조건에 명시된 힙 영역 **밖**을 건드리지 않아야 한다. 글로벌 변수를 임의로 수정하거나, 복잡한 동시성 패턴(RCU, hazard pointer 등)은 고급 CSL 확장(Iris의 ghost 상태, invariant 등)이 필요하다.

### 4. Rust에서 Unsafe의 의미

Rust의 `unsafe` 블록은 분리 논리적으로 "이 영역은 타입 시스템이 보장하지 않으니 프로그래머가 직접 분리 논리 명세를 머릿속에서 검증해야 한다"는 선언이다. `unsafe` 코드의 정확성 증명이 필요하면 **Verus**, **Prusti** 같은 Rust 전용 검증 도구를 활용할 수 있다.

### 5. 함수 명세 작성 팁

좋은 분리 논리 명세는 다음을 명시해야 한다:

1. **소유권(ownership)**: 어떤 메모리 영역을 함수가 소유(수정 가능)하는가
2. **공유 접근(shared access)**: 읽기 전용으로 접근하는 영역은?
3. **불변식(invariant)**: 함수 호출 전후로 반드시 성립해야 할 조건
4. **자원 반환**: 할당한 메모리는 반드시 반환하는가 (메모리 누수 방지)

```
// 좋은 명세 예시
// 사전 조건: {list(p, xs) * n > 0}
// 사후 조건: {list(결과, take(n, xs)) * list(dropped, drop(n, xs))}
// → 입력 리스트를 정확히 두 부분으로 분리, 메모리 누수 없음
```

## 참고 자료
- [A Beginner Guide to Iris, Coq and Separation Logic (arXiv)](https://arxiv.org/abs/2105.12077)
- [Revisiting Concurrent Separation Logic (ScienceDirect)](https://www.sciencedirect.com/science/article/pii/S235222081630058X)
- [Modular Verification of Safe Memory Reclamation in CSL (ResearchGate)](https://www.researchgate.net/publication/374770176_Modular_Verification_of_Safe_Memory_Reclamation_in_Concurrent_Separation_Logic)
- [RustBelt: Securing the Foundations of the Rust Programming Language](https://plv.mpi-sws.org/rustbelt/)
