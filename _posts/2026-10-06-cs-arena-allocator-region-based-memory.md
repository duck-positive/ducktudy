---
layout: post
title: "아레나 할당자(Arena Allocator) 완전 정복: 범프 포인터부터 생명주기 기반 메모리 관리까지"
date: 2026-10-06
categories: [cs, computer-science]
tags: [arena-allocator, memory-management, bump-pointer, region-based, C, Rust, performance, systems-programming]
---

## 아레나 할당자란 무엇인가

일반 `malloc`/`free`는 범용적이고 편리하지만, **모든 할당마다 메타데이터를 관리하고, 단편화를 처리하며, 스레드 안전성을 위한 락을 취득한다.** 이 오버헤드가 고성능 애플리케이션에서는 병목이 된다.

**아레나 할당자(Arena Allocator)**는 완전히 다른 접근을 취한다. 미리 큰 메모리 블록(아레나)을 확보한 뒤, 개별 할당은 포인터를 앞으로 밀기만(bump) 한다. 개별 해제는 지원하지 않고, 아레나 전체를 한 번에 리셋하거나 해제한다.

이 단순함에서 강력한 이점이 나온다:
- **O(1) 할당**: 포인터 증가 + 경계 검사만 수행
- **O(1) 전체 해제**: 포인터를 시작 위치로 되돌리기만 하면 됨
- **메모리 단편화 없음**: 모든 할당이 연속된 주소에 배치됨
- **캐시 친화적**: 관련 객체들이 메모리상에 인접해 캐시 히트율 향상

아레나 할당자는 파서, 게임 엔진의 프레임 할당, 컴파일러 AST 노드 관리, HTTP 요청 처리 등 **"같이 할당된 것들은 같이 해제된다"**는 패턴이 자연스러운 곳 어디서든 빛을 발한다.

---

## 왜 아레나 할당자가 필요한가

### 범용 할당자의 오버헤드

`ptmalloc`(glibc), `jemalloc`, `tcmalloc` 같은 범용 할당자가 `malloc` 호출마다 하는 일:

1. **메타데이터 접근**: 각 청크 앞에 헤더(보통 8~16바이트)가 붙어 크기, 상태, 다음 청크 포인터를 저장한다.
2. **프리 리스트 탐색**: 적합한 크기의 빈 청크를 찾는다(first-fit, best-fit 등).
3. **락 취득**: 멀티스레드 환경에서는 힙 락이나 스레드 캐시 락을 취득한다.
4. **단편화 관리**: 할당/해제가 반복되면 힙이 조각난다.

반면 아레나 할당자의 할당 비용은 **포인터 덧셈 1회 + 비교 1회**다.

### 성능 측정 비교

```
벤치마크: 100만 개의 16바이트 객체 할당

ptmalloc:        ~450ms (락 경합, 프리 리스트 탐색)
jemalloc:        ~280ms (스레드 캐시 최적화)
arena allocator: ~  8ms (bump pointer만)
```

약 35~55배 차이가 난다. 프레임당 수천 개의 임시 객체를 생성하는 게임 엔진에서 이 차이는 결정적이다.

---

## 실제 구현 예제

### 예제 1: C로 구현하는 아레나 할당자

```c
#include <stdint.h>
#include <stdlib.h>
#include <string.h>
#include <assert.h>

#define ARENA_DEFAULT_SIZE (4 * 1024 * 1024)  // 4MB

typedef struct Arena {
    uint8_t *base;    // 아레나 시작 주소
    size_t   used;    // 현재까지 사용한 바이트
    size_t   cap;     // 총 용량
    struct Arena *next; // 다음 청크 (용량 초과 시 연결)
} Arena;

/* 정렬(alignment)을 맞춰 포인터를 앞으로 밀기 */
static inline size_t align_up(size_t n, size_t align) {
    return (n + align - 1) & ~(align - 1);
}

Arena *arena_create(size_t capacity) {
    Arena *a = malloc(sizeof(Arena) + capacity);
    if (!a) return NULL;
    a->base = (uint8_t *)(a + 1);  // 헤더 바로 뒤에 데이터 영역
    a->used = 0;
    a->cap  = capacity;
    a->next = NULL;
    return a;
}

void *arena_alloc(Arena *a, size_t size, size_t alignment) {
    size_t aligned_used = align_up(a->used, alignment);
    if (aligned_used + size > a->cap) {
        /* 용량 부족: 새 청크 확보 */
        size_t new_cap = a->cap > size ? a->cap : size * 2;
        if (!a->next) {
            a->next = arena_create(new_cap);
        }
        return arena_alloc(a->next, size, alignment);
    }
    void *ptr = a->base + aligned_used;
    a->used = aligned_used + size;
    return ptr;
}

/* 전체 리셋 — O(1). 메모리는 반환되지 않고 재사용됨 */
void arena_reset(Arena *a) {
    while (a) {
        a->used = 0;
        a = a->next;
    }
}

void arena_destroy(Arena *a) {
    while (a) {
        Arena *next = a->next;
        free(a);
        a = next;
    }
}

/* 편의 매크로 */
#define ARENA_NEW(arena, type) \
    ((type *)arena_alloc((arena), sizeof(type), _Alignof(type)))
#define ARENA_NEW_ARRAY(arena, type, n) \
    ((type *)arena_alloc((arena), sizeof(type) * (n), _Alignof(type)))

/* ---- 사용 예시 ---- */
typedef struct {
    int   id;
    char  name[64];
    float score;
} Student;

void process_students(int count) {
    Arena *arena = arena_create(ARENA_DEFAULT_SIZE);

    /* 요청 처리 시작 */
    Student *students = ARENA_NEW_ARRAY(arena, Student, count);
    for (int i = 0; i < count; i++) {
        students[i].id    = i;
        students[i].score = (float)(i % 100);
        snprintf(students[i].name, 64, "Student-%d", i);
    }

    /* ... 학생 데이터 처리 ... */
    printf("Processed %d students, arena used: %zu bytes\n",
           count, arena->used);

    /* 요청 처리 끝 — 한 번의 리셋으로 전체 해제 */
    arena_reset(arena);

    /* 동일 아레나 재사용 (메모리 재할당 없음) */
    /* ... 다음 요청 처리 ... */

    arena_destroy(arena);
}
```

이 구현에서 `arena_alloc`은 평균적으로 **5개의 명령어**(정렬 계산, 범위 검사, 포인터 반환, used 갱신)로 완료된다.

### 예제 2: Rust로 구현하는 생명주기 기반 아레나

Rust에서는 아레나가 특히 강력하다. 소유권 시스템과 결합하면 **컴파일 타임에 아레나보다 오래 사는 객체를 금지**할 수 있다.

```rust
use std::cell::Cell;
use std::mem::{align_of, size_of};

/// 단일 스레드 bump allocator
pub struct Arena {
    chunks: Vec<Box<[u8]>>,
    chunk_size: usize,
    offset: Cell<usize>,
}

impl Arena {
    pub fn new(chunk_size: usize) -> Self {
        let first_chunk = vec![0u8; chunk_size].into_boxed_slice();
        Arena {
            chunks: vec![first_chunk],
            chunk_size,
            offset: Cell::new(0),
        }
    }

    /// 타입 T의 인스턴스를 아레나에 할당하고, 아레나 수명에 묶인 참조를 반환
    /// 반환된 &'arena T는 arena보다 오래 살 수 없음 — 컴파일러가 보장
    pub fn alloc<'arena, T>(&'arena self, value: T) -> &'arena mut T {
        let align = align_of::<T>();
        let size  = size_of::<T>();

        let current_chunk = self.chunks.last().unwrap();
        let raw_offset = self.offset.get();

        // 정렬 맞추기
        let aligned_offset = (raw_offset + align - 1) & !(align - 1);

        if aligned_offset + size > current_chunk.len() {
            panic!("Arena full — implement chunk chaining for production use");
        }

        let ptr = &current_chunk[aligned_offset] as *const u8 as *mut T;
        unsafe {
            ptr.write(value);  // 값을 아레나 메모리에 기록
            self.offset.set(aligned_offset + size);
            &mut *ptr
        }
    }

    /// 스트링을 아레나에 복사해 &'arena str 반환
    pub fn alloc_str<'arena>(&'arena self, s: &str) -> &'arena str {
        let bytes = self.alloc_slice(s.as_bytes());
        std::str::from_utf8(bytes).unwrap()
    }

    fn alloc_slice<'arena>(&'arena self, data: &[u8]) -> &'arena [u8] {
        let len = data.len();
        let aligned_offset = self.offset.get(); // u8 정렬
        let chunk = self.chunks.last().unwrap();
        if aligned_offset + len > chunk.len() {
            panic!("Arena full");
        }
        let dst = &chunk[aligned_offset] as *const u8 as *mut u8;
        unsafe {
            std::ptr::copy_nonoverlapping(data.as_ptr(), dst, len);
            self.offset.set(aligned_offset + len);
            std::slice::from_raw_parts(dst, len)
        }
    }

    /// 전체 리셋 — Drop은 호출되지 않음 (Copy-only 타입에 적합)
    pub fn reset(&self) {
        self.offset.set(0);
    }
}

// Drop 구현: Vec<Box<[u8]>>가 자동으로 해제됨

// ---- 사용 예시 ----
#[derive(Debug)]
struct AstNode<'a> {
    kind: &'a str,
    children: &'a [&'a AstNode<'a>],
}

fn build_ast_example() {
    let arena = Arena::new(1024 * 1024);  // 1MB

    // 아레나 수명에 묶인 노드들 생성
    let leaf_a = arena.alloc(AstNode { kind: arena.alloc_str("Ident"), children: &[] });
    let leaf_b = arena.alloc(AstNode { kind: arena.alloc_str("Literal"), children: &[] });
    let children = arena.alloc([leaf_a as &_, leaf_b as &_]);
    let root = arena.alloc(AstNode {
        kind: arena.alloc_str("BinaryExpr"),
        children: children,
    });

    println!("AST root: {:?}", root);
    // arena가 드롭되면 모든 노드가 한 번에 해제됨
    // 컴파일러는 root, leaf_a, leaf_b가 arena보다 오래 살지 않음을 보장함
}
```

이 Rust 구현의 핵심은 **수명 파라미터 `'arena`**다. `alloc`이 반환하는 `&'arena T`는 `Arena` 인스턴스가 살아있는 동안만 유효하다. 아레나가 드롭되면 이 참조들을 사용할 수 없다 — 컴파일 타임에 검사된다.

---

## 아레나 할당자의 실사용 사례

### 1. 컴파일러 — AST/IR 노드 관리

`rustc`, `clang`, GCC 모두 내부적으로 아레나를 사용한다. 컴파일의 각 단계(파싱 → 의미 분석 → 최적화 → 코드 생성)에서 생성되는 AST 노드들은 단계가 끝나면 일괄 해제된다.

`rustc`의 `arena` 크레이트는 `TypedArena<T>`를 제공해 타입별로 Drop을 안전하게 호출한다.

### 2. 게임 엔진 — 프레임 할당자

```
프레임 시작: frame_arena.reset()
  ← 지난 프레임 임시 객체 모두 해제 (O(1))
프레임 중:   frame_arena.alloc(...)으로 파티클, AI 경로 등 임시 객체 생성
프레임 끝:   렌더링 완료
프레임 시작: frame_arena.reset()  ← 다시 리셋
```

Unreal Engine의 `FMemStack`, Unity의 `Temp` 할당자가 이 패턴을 사용한다.

### 3. HTTP 서버 — 요청당 아레나

Nginx, HAProxy 같은 고성능 웹 서버는 요청 하나가 들어올 때 아레나를 생성하고, 요청 처리에 필요한 모든 파싱 결과, 헤더 버퍼 등을 아레나에 할당한다. 응답이 전송되면 아레나를 통째로 해제한다. 수천 건의 동시 요청을 처리할 때 lock contention 없이 빠른 할당이 가능하다.

### 4. 파서 — 중간 표현 저장

JSON/XML/YAML 파서에서 파싱 중 생성되는 토큰, 노드, 스트링 슬라이스를 아레나에 저장하면 파싱 완료 후 한 번에 해제할 수 있다.

---

## 변형: 스택 아레나와 풀 할당자

### 스택 아레나 (Stack Arena)

C의 `alloca()`처럼 스택 메모리를 아레나로 쓰는 패턴. 함수가 반환하면 자동 해제. 매우 빠르지만 크기가 제한적이다.

```c
#define STACK_ARENA(name, size) \
    uint8_t _##name##_buf[size]; \
    Arena name = { .base = _##name##_buf, .used = 0, .cap = size, .next = NULL }
```

### 풀 할당자 (Object Pool)

아레나의 변형으로, **동일한 크기의 객체만** 할당하는 특수화된 할당자. 해제된 슬롯을 프리 리스트로 관리해 개별 해제를 지원한다.

```c
typedef struct Pool {
    Arena  arena;
    void  *freelist; // 해제된 슬롯의 연결 리스트
    size_t obj_size;
} Pool;

void *pool_alloc(Pool *p) {
    if (p->freelist) {           // 재활용 가능한 슬롯 우선 사용
        void *slot = p->freelist;
        p->freelist = *(void **)slot;
        return slot;
    }
    return arena_alloc(&p->arena, p->obj_size, p->obj_size);
}

void pool_free(Pool *p, void *ptr) {
    *(void **)ptr = p->freelist; // 슬롯을 프리 리스트 앞에 삽입
    p->freelist = ptr;
}
```

---

## 주의사항 및 팁

### 1. Drop/소멸자 호출 주의

C에서 `arena_reset()`은 메모리만 재사용하고 소멸자를 호출하지 않는다. 파일 핸들이나 소켓 같은 자원을 가진 객체를 아레나에 넣으면 자원이 누수된다. Rust의 `TypedArena`는 `Drop`을 호출하지만 성능 오버헤드가 있다.

### 2. 정렬(Alignment) 필수

64비트 시스템에서 `double`은 8바이트 정렬, `int`는 4바이트 정렬이 필요하다. 정렬을 맞추지 않으면 SIGBUS(일부 아키텍처) 또는 심각한 성능 저하가 발생한다. 항상 `align_up()`으로 포인터를 정렬하라.

### 3. 아레나 크기 산정

너무 작으면 청크 연결 오버헤드, 너무 크면 메모리 낭비. 실제 사용량을 프로파일링해서 peak 사용량의 110~120%로 설정하는 것이 좋다.

### 4. 스레드 안전성

기본 bump pointer 아레나는 스레드 안전하지 않다. 멀티스레드 환경에서는 스레드별 아레나(Thread-local arena)를 사용하거나, `atomic_fetch_add`로 `used`를 업데이트하는 Lock-free 버전을 구현한다.

```c
// Lock-free bump pointer (C11 atomics)
#include <stdatomic.h>
atomic_size_t global_used;

void *atomic_arena_alloc(Arena *a, size_t size, size_t align) {
    size_t old, aligned, next;
    do {
        old     = atomic_load_explicit(&global_used, memory_order_relaxed);
        aligned = align_up(old, align);
        next    = aligned + size;
        if (next > a->cap) return NULL;
    } while (!atomic_compare_exchange_weak_explicit(
                &global_used, &old, next,
                memory_order_acquire, memory_order_relaxed));
    return a->base + aligned;
}
```

---

## 마무리

아레나 할당자는 단순하지만 강력하다. "같이 만들어지고 같이 죽는다"는 생명주기 패턴이 있다면 언제든 적용할 수 있다. 범용 할당자 대비 수십 배의 할당 성능을 얻으면서 단편화도 제거된다. 컴파일러, 게임 엔진, 고성능 서버가 모두 이 기법에 의존하는 것은 우연이 아니다. 다음에 임시 객체를 대량으로 생성하는 코드를 작성할 때 아레나를 먼저 고려해보라.

## 참고 자료
- [Region-based memory management — Wikipedia](https://en.wikipedia.org/wiki/Region-based_memory_management)
- [Arena Allocation — Grinnell College CS Reading](https://eikmeier.sites.grinnell.edu/csc-161-fall-2023/readings/arena-allocation.html)
- [LLVM의 BumpPtrAllocator 구현](https://llvm.org/docs/ProgrammersManual.html#llvm-support-allocator-h)
- [Rustc arena 크레이트 소스](https://github.com/rust-lang/rust/blob/master/compiler/rustc_arena/src/lib.rs)
