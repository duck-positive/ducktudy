---
layout: post
title: "메모리 정렬과 구조체 패딩 완전 정복: CPU가 데이터 경계를 맞추는 이유와 성능 최적화"
date: 2026-09-17
categories: [cs, computer-science]
tags: [memory-alignment, struct-padding, cache-line, cpu-architecture, c, 시스템프로그래밍, 메모리최적화]
---

## 개요

C나 C++로 구조체를 정의할 때 멤버 변수를 선언한 순서와 크기만 봐도 구조체의 전체 크기를 계산할 수 있을 것 같다. 하지만 실제로는 그렇지 않다. 컴파일러는 성능을 위해 구조체 멤버 사이에 보이지 않는 **패딩(padding) 바이트**를 삽입하고, 구조체 전체 크기도 특정 배수로 맞춘다. 이를 **메모리 정렬(Memory Alignment)**이라 한다. 이 개념을 이해하지 못하면 메모리를 낭비하거나, 미묘한 성능 저하를 겪거나, 심지어 잘못된 동작(버스 에러)을 맞닥뜨릴 수 있다.

## 왜 메모리 정렬이 필요한가

### CPU와 메모리 버스의 물리적 제약

현대 CPU는 메모리를 **워드(word)** 단위로 읽는다. 64비트 시스템의 워드는 8바이트다. CPU가 메모리에서 데이터를 읽을 때는 버스를 통해 정렬된 주소로부터 워드 단위로 데이터를 가져온다.

만약 8바이트 `double` 값이 주소 `0x1001`(홀수 주소, 즉 8의 배수가 아님)에 저장되어 있다면, CPU는 이를 읽기 위해 두 번의 메모리 접근이 필요하다:
- 첫 번째: `0x1000`부터 8바이트를 읽어 상위 7바이트 획득
- 두 번째: `0x1008`부터 8바이트를 읽어 하위 1바이트 획득
- 두 조각을 합쳐서 완성

이는 정렬된 접근보다 2배의 메모리 접근이 필요하며, 일부 구형 아키텍처(SPARC, MIPS 등)는 비정렬 접근 시 **SIGBUS** 신호를 발생시켜 프로그램을 종료시키기도 한다. x86은 비정렬 접근을 허용하지만 성능 패널티가 발생한다.

### 각 타입의 정렬 요구사항

```c
#include <stdio.h>
#include <stddef.h>

void print_alignof() {
    printf("char     정렬: %zu바이트\n", _Alignof(char));     // 1
    printf("short    정렬: %zu바이트\n", _Alignof(short));    // 2
    printf("int      정렬: %zu바이트\n", _Alignof(int));      // 4
    printf("long     정렬: %zu바이트\n", _Alignof(long));     // 8 (64bit)
    printf("float    정렬: %zu바이트\n", _Alignof(float));    // 4
    printf("double   정렬: %zu바이트\n", _Alignof(double));   // 8
    printf("pointer  정렬: %zu바이트\n", _Alignof(void*));    // 8 (64bit)
}
```

규칙은 단순하다: **타입의 정렬 요구사항은 그 타입의 크기와 같다** (최대 플랫폼 워드 크기까지). `int`(4바이트)는 4의 배수 주소에, `double`(8바이트)는 8의 배수 주소에 위치해야 한다.

## 구조체 패딩의 동작 원리

컴파일러는 구조체 멤버를 선언 순서대로 배치하되, 각 멤버의 정렬 요구사항을 만족시키기 위해 **패딩을 삽입**한다. 구조체 전체 크기는 **가장 큰 멤버의 정렬 요구사항의 배수**로 맞춰진다.

### 실제 패딩 분석

```c
#include <stdio.h>

// 나쁜 순서: 패딩이 많이 발생
struct Bad {
    char  a;    // 1바이트, 오프셋 0
                // 패딩 3바이트 (int의 4바이트 정렬을 위해)
    int   b;    // 4바이트, 오프셋 4
    char  c;    // 1바이트, 오프셋 8
                // 패딩 7바이트 (double의 8바이트 정렬을 위해)
    double d;   // 8바이트, 오프셋 16
    char  e;    // 1바이트, 오프셋 24
                // 패딩 7바이트 (구조체 크기를 8의 배수로 맞추기 위해)
};              // 총 32바이트!

// 좋은 순서: 패딩 최소화
struct Good {
    double d;   // 8바이트, 오프셋 0
    int    b;   // 4바이트, 오프셋 8
    char   a;   // 1바이트, 오프셋 12
    char   c;   // 1바이트, 오프셋 13
    char   e;   // 1바이트, 오프셋 14
                // 패딩 1바이트 (크기를 4의 배수로...)
                // 실제론 int 정렬(4)에 맞게 전체가 16
};              // 총 16바이트 (Bad의 절반!)

void analyze_structs() {
    printf("Bad 구조체 크기:  %zu바이트\n", sizeof(struct Bad));   // 32
    printf("Good 구조체 크기: %zu바이트\n", sizeof(struct Good));  // 16
    
    // 오프셋 확인
    printf("\nBad 멤버 오프셋:\n");
    printf("  a: %zu\n", offsetof(struct Bad, a));  // 0
    printf("  b: %zu\n", offsetof(struct Bad, b));  // 4
    printf("  c: %zu\n", offsetof(struct Bad, c));  // 8
    printf("  d: %zu\n", offsetof(struct Bad, d));  // 16
    printf("  e: %zu\n", offsetof(struct Bad, e));  // 24
}
```

### 패딩 계산 알고리즘

컴파일러가 패딩을 계산하는 알고리즘은 다음과 같다:

```python
def calculate_struct_layout(members):
    """
    members: [(name, size, alignment), ...]
    """
    offsets = {}
    current_offset = 0
    max_alignment = 1
    
    for name, size, alignment in members:
        max_alignment = max(max_alignment, alignment)
        
        # 현재 오프셋이 정렬 요구사항의 배수인지 확인
        # 아니라면 패딩 삽입
        remainder = current_offset % alignment
        if remainder != 0:
            padding = alignment - remainder
            print(f"  -> {name} 앞에 패딩 {padding}바이트 삽입")
            current_offset += padding
        
        offsets[name] = current_offset
        current_offset += size
        print(f"  {name}: 오프셋 {offsets[name]}, 크기 {size}")
    
    # 구조체 전체 크기를 max_alignment의 배수로 맞추기
    remainder = current_offset % max_alignment
    if remainder != 0:
        tail_padding = max_alignment - remainder
        print(f"  -> 구조체 끝에 패딩 {tail_padding}바이트 삽입")
        current_offset += tail_padding
    
    print(f"총 크기: {current_offset}바이트")
    return offsets, current_offset

print("=== Bad 구조체 레이아웃 ===")
members_bad = [
    ("char a",   1, 1),
    ("int b",    4, 4),
    ("char c",   1, 1),
    ("double d", 8, 8),
    ("char e",   1, 1),
]
calculate_struct_layout(members_bad)

print("\n=== Good 구조체 레이아웃 ===")
members_good = [
    ("double d", 8, 8),
    ("int b",    4, 4),
    ("char a",   1, 1),
    ("char c",   1, 1),
    ("char e",   1, 1),
]
calculate_struct_layout(members_good)
```

## 캐시 라인과 정렬

### 캐시 라인 크기

현대 CPU 캐시는 데이터를 **캐시 라인(Cache Line)** 단위로 관리하며, 대부분의 x86-64 CPU에서 캐시 라인은 **64바이트**다.

```c
#include <stdlib.h>
#include <stdio.h>
#include <time.h>

#define N 1000
#define ITERATIONS 10000000

// 구조체가 두 캐시 라인에 걸쳐 있는 경우 (Cache Line Splitting)
struct __attribute__((packed)) BadAligned {
    // 의도적으로 63바이트 오프셋에 int 배치 (두 캐시 라인에 걸침)
    char padding[63];
    int value;  // 오프셋 63: 캐시 라인 경계에 걸침
};

// 캐시 라인에 맞게 정렬
struct __attribute__((aligned(64))) GoodAligned {
    int value;  // 캐시 라인 시작에 배치
    char padding[60];
};

// False Sharing 방지: 각 스레드 데이터를 별도 캐시 라인에
#define CACHE_LINE_SIZE 64
struct ThreadCounter {
    long count;
    char pad[CACHE_LINE_SIZE - sizeof(long)];  // 패딩으로 캐시 라인 채우기
} __attribute__((aligned(CACHE_LINE_SIZE)));
```

### False Sharing: 멀티스레드에서의 정렬 함정

멀티스레드 프로그램에서 서로 다른 스레드가 같은 캐시 라인에 있는 다른 변수를 수정하면 **False Sharing**이 발생한다. 서로 다른 데이터를 수정하지만 같은 캐시 라인을 무효화시켜 성능이 급락한다.

```c
#include <pthread.h>
#include <stdio.h>
#include <stdint.h>

#define ITERATIONS 100000000
#define CACHE_LINE 64

// False Sharing 발생: 두 카운터가 같은 캐시 라인에 위치
struct WithFalseSharing {
    volatile long counter1;  // 스레드 1 사용
    volatile long counter2;  // 스레드 2 사용 (같은 캐시 라인!)
};

// False Sharing 방지: 각 카운터를 별도 캐시 라인에
struct WithoutFalseSharing {
    volatile long counter1 __attribute__((aligned(CACHE_LINE)));
    char pad1[CACHE_LINE - sizeof(long)];
    volatile long counter2 __attribute__((aligned(CACHE_LINE)));
    char pad2[CACHE_LINE - sizeof(long)];
};

// 성능 차이 측정 (일반적으로 2~10배 차이)
void* increment_counter1_bad(void* arg) {
    struct WithFalseSharing* s = arg;
    for (long i = 0; i < ITERATIONS; i++) s->counter1++;
    return NULL;
}

void* increment_counter2_bad(void* arg) {
    struct WithFalseSharing* s = arg;
    for (long i = 0; i < ITERATIONS; i++) s->counter2++;
    return NULL;
}

void benchmark_false_sharing() {
    struct WithFalseSharing bad = {0, 0};
    pthread_t t1, t2;
    
    struct timespec start, end;
    clock_gettime(CLOCK_MONOTONIC, &start);
    
    pthread_create(&t1, NULL, increment_counter1_bad, &bad);
    pthread_create(&t2, NULL, increment_counter2_bad, &bad);
    pthread_join(t1, NULL);
    pthread_join(t2, NULL);
    
    clock_gettime(CLOCK_MONOTONIC, &end);
    double elapsed = (end.tv_sec - start.tv_sec) + 
                     (end.tv_nsec - start.tv_nsec) / 1e9;
    printf("False Sharing 있음: %.3f초\n", elapsed);
}
```

## 정렬 제어 기법

### __attribute__((packed)): 패딩 제거

```c
// 네트워크 패킷 헤더: 패딩 없이 정확한 바이트 레이아웃 필요
struct __attribute__((packed)) IPv4Header {
    uint8_t  version_ihl;      // 버전(4비트) + IHL(4비트)
    uint8_t  tos;
    uint16_t total_length;     // 빅엔디안
    uint16_t identification;
    uint16_t flags_fragment;
    uint8_t  ttl;
    uint8_t  protocol;
    uint16_t checksum;
    uint32_t src_ip;
    uint32_t dst_ip;
};  // packed 없으면 패딩으로 24바이트, packed이면 정확히 20바이트

// 주의: packed 구조체 멤버의 포인터를 직접 역참조하면
// 비정렬 접근 위험! memcpy 사용 권장
uint16_t get_total_length(const struct IPv4Header* hdr) {
    uint16_t len;
    memcpy(&len, &hdr->total_length, sizeof(len));  // 안전
    return ntohs(len);
}
```

### __attribute__((aligned(N))): 강제 정렬

```c
// SIMD 연산을 위한 16/32/64바이트 정렬
float array[256] __attribute__((aligned(32)));  // AVX 256비트 SIMD용

// 구조체 특정 멤버 정렬
struct Optimized {
    int   small;
    float data[16] __attribute__((aligned(64)));  // 캐시 라인 정렬
};
```

### C11 alignas / C++11 alignas

```cpp
#include <stdalign.h>  // C11
// #include <memory>  // C++

// C11
alignas(64) char cache_friendly_buffer[256];

struct alignas(16) SimdVector {
    float x, y, z, w;  // 16바이트 정렬 보장
};

// C++17: std::aligned_storage 대신 std::byte 배열 + alignas
alignas(SimdVector) std::byte storage[sizeof(SimdVector)];
SimdVector* vec = new(storage) SimdVector{1, 2, 3, 4};
```

## 실전 최적화 팁

### 구조체 멤버 정렬 순서 최적화

**원칙: 크기 내림차순으로 선언하면 패딩이 최소화된다.**

```c
// 메모리 사용량 최적화: 큰 것부터 작은 것 순서
struct OptimalOrder {
    double  d;    // 8바이트
    long    l;    // 8바이트
    int     i;    // 4바이트
    short   s;    // 2바이트
    char    c1;   // 1바이트
    char    c2;   // 1바이트
    // 패딩 없이 24바이트
};

// 최악의 순서
struct WorstOrder {
    char   c1;    // 1바이트 + 패딩 7
    double d;     // 8바이트
    char   c2;    // 1바이트 + 패딩 3
    int    i;     // 4바이트
    char   c3;    // 1바이트 + 패딩 1
    short  s;     // 2바이트
    char   c4;    // 1바이트 + 패딩 7
    long   l;     // 8바이트
    // 총 48바이트! (OptimalOrder의 2배)
};
```

### pahole로 패딩 분석

Linux의 `pahole` 도구는 구조체의 패딩을 시각화해준다:

```bash
# 컴파일 (디버그 심볼 포함)
$ gcc -g -o myprogram myprogram.c

# 구조체 레이아웃 분석
$ pahole myprogram

struct Bad {
    char a;                  /*     0     1 */
    /* XXX 3 bytes hole, try to pack */
    int  b;                  /*     4     4 */
    char c;                  /*     8     1 */
    /* XXX 7 bytes hole, try to pack */
    double d;                /*    16     8 */
    char e;                  /*    24     1 */
    /* XXX 7 bytes padding  */

    /* size: 32, cachelines: 1, members: 5 */
    /* sum members: 15, holes: 2, sum holes: 10 */
    /* padding: 7 */
    /* last cacheline: 32 bytes */
};
```

### 컴파일러 경고 활용

```bash
# GCC/Clang으로 패딩 경고 활성화
$ gcc -Wpadded myprogram.c

# 출력 예:
myprogram.c:5:14: warning: padding struct 'Bad' with 3 bytes to align 'b'
myprogram.c:7:14: warning: padding struct 'Bad' with 7 bytes to align 'd'
```

## 데이터 지향 설계와 정렬

게임 엔진이나 고성능 시스템에서는 **배열의 구조체(Array of Structs, AoS)** 대신 **구조체의 배열(Struct of Arrays, SoA)** 패턴을 사용해 정렬과 캐시 효율을 동시에 개선한다:

```c
#define N 10000

// AoS: 각 엔티티의 모든 데이터가 함께
struct Entity_AoS {
    float x, y, z;     // 위치
    float vx, vy, vz;  // 속도
    int   health;
    float mass;
};
struct Entity_AoS entities_aos[N];

// SoA: 같은 종류의 데이터가 연속 배치 (SIMD 친화적)
struct Entities_SoA {
    float x[N], y[N], z[N];      // 위치 배열
    float vx[N], vy[N], vz[N];   // 속도 배열
    int   health[N];
    float mass[N];
} entities_soa;

// SoA + SIMD: 8개 엔티티의 x 좌표를 동시에 업데이트
#include <immintrin.h>
void update_positions_simd(float dt) {
    __m256 dt_vec = _mm256_set1_ps(dt);
    
    for (int i = 0; i < N; i += 8) {
        __m256 x  = _mm256_load_ps(&entities_soa.x[i]);
        __m256 vx = _mm256_load_ps(&entities_soa.vx[i]);
        x = _mm256_fmadd_ps(vx, dt_vec, x);  // x += vx * dt
        _mm256_store_ps(&entities_soa.x[i], x);
    }
}
```

## 참고 자료
- [cppreference: alignof, alignas](https://en.cppreference.com/w/c/language/_Alignof)
- [Linux kernel: Data alignment](https://www.kernel.org/doc/html/latest/process/coding-style.html)
- [What Every Programmer Should Know About Memory (Ulrich Drepper)](https://people.freebsd.org/~lstewart/articles/cpumemory.pdf)
- [pahole 도구 문서](https://linux.die.net/man/1/pahole)
