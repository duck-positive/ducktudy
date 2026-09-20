---
layout: post
title: "프로세스 메모리 레이아웃 완전 정복: Text·Data·BSS·Heap·Stack 세그먼트의 내부 구조"
date: 2026-09-20
categories: [cs, computer-science]
tags: [memory, process, stack, heap, bss, virtual-memory, linux, os, systems-programming]
---

## 개요

프로그램이 실행될 때 운영체제는 각 프로세스에 독립적인 **가상 주소 공간(Virtual Address Space)**을 할당합니다. 이 공간은 여러 세그먼트로 나뉘며, 각 세그먼트는 서로 다른 종류의 데이터와 코드를 담당합니다. 이 구조를 깊이 이해하면 메모리 관련 버그 디버깅(스택 오버플로우, 힙 손상, 세그멘테이션 폴트), 성능 최적화, 보안 취약점 분석이 훨씬 쉬워집니다.

Linux x86-64 기준으로 32비트 프로세스의 가상 주소 공간(4GB) 레이아웃을 살펴보겠습니다.

---

## 가상 주소 공간의 전체 구조

```
높은 주소 (0xFFFFFFFF)
┌──────────────────────────────┐
│       커널 공간               │  ← 커널 코드, 데이터 (사용자 접근 불가)
│       (1GB ~ 4GB)            │
├──────────────────────────────┤ 0xC0000000
│       Stack                  │  ← 지역 변수, 함수 호출 스택 (↓ 아래로 성장)
│         ↓                    │
│    (비어있는 공간)             │
│         ↑                    │
│       Heap                   │  ← 동적 메모리 (↑ 위로 성장)
├──────────────────────────────┤
│       BSS Segment            │  ← 초기화되지 않은 전역/정적 변수
├──────────────────────────────┤
│       Data Segment           │  ← 초기화된 전역/정적 변수
├──────────────────────────────┤
│       Text Segment           │  ← 실행 코드 (읽기 전용)
└──────────────────────────────┘ 0x00000000 (낮은 주소)
```

64비트 리눅스에서는 사용자 공간이 일반적으로 0x0000000000000000 ~ 0x00007FFFFFFFFFFF(128TB)를, 커널 공간이 0xFFFF800000000000 이상을 차지합니다.

---

## 각 세그먼트 상세 분석

### 1. Text Segment (코드 세그먼트)

프로그램의 **실행 가능한 기계어 코드**가 저장됩니다.

**특징:**
- **읽기 전용(Read-Only)**: 코드 자기 수정을 방지하고 보안을 강화합니다. 쓰기 시도 시 `SIGSEGV` 발생
- **공유 가능(Sharable)**: 같은 프로그램의 여러 인스턴스가 물리 메모리에서 동일한 텍스트 세그먼트를 공유합니다 (Copy-on-Write)
- 컴파일러가 생성한 상수 문자열 리터럴("hello world")도 이 영역에 저장되는 경우가 많습니다

```c
// 이 코드는 텍스트 세그먼트에 기계어로 저장됨
int add(int a, int b) {
    return a + b;  // → ADD 명령어로 컴파일됨
}

// 주의: 함수 포인터를 통한 코드 수정 시도는 SIGSEGV 유발
void modify_code() {
    char *ptr = (char *)add;
    *ptr = 0x90;  // ← SIGSEGV! 텍스트 세그먼트는 쓰기 불가
}
```

### 2. Data Segment (초기화된 데이터 세그먼트)

**초기화된 전역 변수와 정적 변수**가 저장됩니다.

```c
// 모두 Data Segment에 저장
int global_initialized = 42;           // 전역 초기화 변수
static float static_pi = 3.14159f;    // 정적 초기화 변수

// 읽기-쓰기 영역 (수정 가능)
void modify() {
    global_initialized = 100;  // OK
}

// 읽기 전용 영역 (컴파일러에 따라 .rodata 섹션)
const char *greeting = "Hello";  // 포인터는 Data, 문자열은 .rodata
```

Data 세그먼트는 실행 파일(ELF)에 초기값이 직접 포함됩니다. 따라서 `int arr[10000] = {1};`처럼 큰 배열을 초기화하면 실행 파일 크기가 커집니다.

### 3. BSS Segment (Block Started by Symbol)

**초기화되지 않은 전역 변수와 정적 변수**를 위한 공간입니다.

```c
// BSS Segment에 저장 (실행 파일에 크기만 기록됨)
int global_uninitialized;          // 자동으로 0으로 초기화됨
static char buffer[1024 * 1024];   // 1MB이지만 실행 파일 크기 증가 없음

// 반면 Data Segment (실행 파일에 실제 데이터 포함)
int data_initialized[1024 * 1024] = {0};  // 실행 파일이 4MB 커짐!
```

**핵심 최적화**: BSS는 실행 파일에 크기만 기록하고 실제 데이터를 저장하지 않습니다. OS 로더가 프로세스 시작 시 해당 크기만큼 메모리를 할당하고 0으로 초기화합니다. 이 덕분에 큰 전역 배열을 선언해도 실행 파일 크기가 늘어나지 않습니다.

### 4. Heap (힙)

`malloc()`, `calloc()`, `new` 등 **동적 메모리 할당**을 위한 영역입니다. 낮은 주소에서 높은 주소 방향(↑)으로 성장합니다.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

void heap_demonstration() {
    // sbrk() 또는 mmap()으로 OS에서 메모리 획득
    int *arr = (int *)malloc(10 * sizeof(int));
    if (!arr) {
        perror("malloc failed");
        return;
    }

    // 힙 메모리 사용
    for (int i = 0; i < 10; i++) {
        arr[i] = i * i;
    }

    // calloc: 할당 + 0 초기화 (memset 불필요)
    char *buf = (char *)calloc(256, sizeof(char));

    // realloc: 크기 조정 (내부적으로 복사 발생 가능)
    arr = (int *)realloc(arr, 20 * sizeof(int));

    // 반드시 해제! 해제 안 하면 메모리 누수
    free(arr);
    free(buf);
    arr = NULL;  // dangling pointer 방지
    buf = NULL;
}
```

**힙 단편화**: 동적 할당/해제를 반복하면 작은 빈 조각들이 여기저기 흩어져 실제로 사용 가능한 메모리가 줄어드는 **단편화(Fragmentation)**가 발생합니다. tcmalloc, jemalloc 같은 고성능 할당기는 이를 최소화하는 알고리즘을 사용합니다.

### 5. Stack (스택)

**함수 호출과 지역 변수**를 관리합니다. 높은 주소에서 낮은 주소 방향(↓)으로 성장합니다.

각 함수 호출마다 **스택 프레임(Stack Frame)**이 생성됩니다:
- 지역 변수
- 함수 인자 (일부는 레지스터로 전달)
- 반환 주소(Return Address)
- 저장된 레지스터 값
- 이전 프레임 포인터(Frame Pointer)

```c
#include <stdio.h>

// 스택 프레임 구조를 보여주는 예제
int multiply(int a, int b) {
    // a, b: 스택에 저장된 인자 (x86-64에서는 레지스터로 전달 가능)
    int result = a * b;  // result: 스택의 지역 변수
    return result;       // 반환 후 이 프레임은 소멸
}

int factorial(int n) {
    if (n <= 1) return 1;
    // 재귀 호출마다 새 스택 프레임 생성
    return n * factorial(n - 1);
    // factorial(100000) → Stack Overflow!
}

// 스택 주소 확인 (감소하는 방향 확인)
void show_stack_direction() {
    int local1 = 1;
    printf("local1 주소: %p\n", (void *)&local1);
    
    int local2 = 2;
    printf("local2 주소: %p\n", (void *)&local2);
    // local2의 주소 < local1의 주소 (스택은 ↓ 성장)
}
```

---

## 실전 코드 예제: 메모리 레이아웃 시각화

### 예제 1: 각 변수가 어느 세그먼트에 위치하는지 확인

```c
#include <stdio.h>
#include <stdlib.h>

/* ---- 전역 영역 ---- */
int g_init = 100;           // Data Segment
int g_uninit;               // BSS Segment
static char g_static[64];   // BSS Segment
const int g_const = 999;    // Data Segment (읽기 전용)

void print_segment_info() {
    /* ---- 스택 영역 ---- */
    int local_var = 42;
    char local_arr[16];

    /* ---- 힙 영역 ---- */
    int *heap_var = (int *)malloc(sizeof(int));
    *heap_var = 777;

    printf("=== 프로세스 메모리 레이아웃 ===\n\n");

    printf("[Text Segment - 코드]\n");
    printf("  함수 포인터 주소: %p\n\n", (void *)print_segment_info);

    printf("[Data Segment - 초기화된 전역]\n");
    printf("  g_init(%d) 주소:  %p\n", g_init, (void *)&g_init);
    printf("  g_const(%d) 주소: %p\n\n", g_const, (void *)&g_const);

    printf("[BSS Segment - 미초기화 전역]\n");
    printf("  g_uninit(%d) 주소: %p\n", g_uninit, (void *)&g_uninit);
    printf("  g_static 주소:    %p\n\n", (void *)g_static);

    printf("[Heap - 동적 할당]\n");
    printf("  heap_var(%d) 주소: %p\n\n", *heap_var, (void *)heap_var);

    printf("[Stack - 지역 변수]\n");
    printf("  local_var(%d) 주소: %p\n", local_var, (void *)&local_var);
    printf("  local_arr 주소:    %p\n\n", (void *)local_arr);

    printf("주소 순서 확인:\n");
    printf("  Text < Data < BSS < Heap < Stack?\n");
    printf("  %p < %p < %p < %p < %p\n",
           (void *)print_segment_info,
           (void *)&g_init,
           (void *)&g_uninit,
           (void *)heap_var,
           (void *)&local_var);

    free(heap_var);
}

int main() {
    print_segment_info();
    return 0;
}
```

실행 결과 (예시):
```
=== 프로세스 메모리 레이아웃 ===

[Text Segment - 코드]
  함수 포인터 주소: 0x401156

[Data Segment - 초기화된 전역]
  g_init(100) 주소:  0x404020
  g_const(999) 주소: 0x402010

[BSS Segment - 미초기화 전역]
  g_uninit(0) 주소:  0x404048
  g_static 주소:     0x40404c

[Heap - 동적 할당]
  heap_var(777) 주소: 0x20eb2a0

[Stack - 지역 변수]
  local_var(42) 주소: 0x7ffd3a8b1c3c
  local_arr 주소:     0x7ffd3a8b1c20
```

### 예제 2: /proc/self/maps로 실제 메모리 맵 읽기

```c
#include <stdio.h>
#include <stdlib.h>

void print_memory_map() {
    FILE *maps = fopen("/proc/self/maps", "r");
    if (!maps) {
        perror("fopen /proc/self/maps");
        return;
    }

    printf("=== /proc/self/maps ===\n");
    printf("%-30s %-6s %-6s %s\n", "주소 범위", "권한", "오프셋", "설명");
    printf("%s\n", "─────────────────────────────────────────────────────");

    char line[512];
    while (fgets(line, sizeof(line), maps)) {
        // 형식: start-end perms offset dev inode pathname
        unsigned long start, end;
        char perms[8], offset[16], dev[16], inode_str[16], path[256];
        path[0] = '\0';
        sscanf(line, "%lx-%lx %s %s %s %s %255[^\n]",
               &start, &end, perms, offset, dev, inode_str, path);
        printf("%012lx-%012lx %-6s %s\n", start, end, perms, path);
    }
    fclose(maps);
}

int main() {
    int *heap = (int *)malloc(1024);  // 힙 할당
    print_memory_map();
    free(heap);
    return 0;
}
```

출력 예시:
```
주소 범위                      권한   오프셋 설명
──────────────────────────────────────────────────────
555555554000-555555555000 r--p  /usr/bin/myprogram    ← ELF 헤더
555555555000-555555556000 r-xp  /usr/bin/myprogram    ← Text (실행)
555555556000-555555557000 r--p  /usr/bin/myprogram    ← .rodata
555555557000-555555558000 rw-p  /usr/bin/myprogram    ← Data/BSS
555555558000-555555579000 rw-p  [heap]                ← Heap
7ffff7d84000-7ffff7daa000 r--p  /lib/x86_64-linux-gnu/libc.so.6
7ffff7daa000-7ffff7f3f000 r-xp  /lib/x86_64-linux-gnu/libc.so.6
7ffffffde000-7ffffffff000 rw-p  [stack]               ← Stack
```

권한 필드 해석: `r`=읽기, `w`=쓰기, `x`=실행, `p`=프라이빗(COW), `s`=공유

---

## 주요 개념: 스택 vs 힙 비교

| 항목 | Stack | Heap |
|------|-------|------|
| 관리 주체 | 컴파일러/CPU 자동 관리 | 프로그래머 (malloc/free) |
| 크기 | 제한됨 (보통 1~8MB) | 프로세스 가용 메모리까지 |
| 성능 | 매우 빠름 (레지스터 연산) | 상대적으로 느림 (할당기 오버헤드) |
| 생존 기간 | 함수 스코프 내 | 명시적 해제까지 |
| 단편화 | 없음 | 있음 |
| 위험 | 스택 오버플로우 | 메모리 누수, 이중 해제, use-after-free |

---

## 주의사항과 팁

**1. 스택 오버플로우 방지**
재귀가 깊어지면 스택이 고갈됩니다. `ulimit -s`로 스택 크기를 확인하고, 깊은 재귀는 반복문으로 변환하거나 힙을 사용하는 명시적 스택 자료구조로 교체하세요.

**2. BSS vs Data: 파일 크기 영향**
초기화된 큰 전역 배열(`int arr[1000000] = {0}`)은 Data 세그먼트에 들어가 실행 파일을 크게 만듭니다. `int arr[1000000]`(미초기화)는 BSS에 들어가 파일 크기 영향이 없습니다.

**3. ASLR (Address Space Layout Randomization)**
현대 리눅스는 보안을 위해 스택, 힙, 라이브러리의 기본 주소를 실행마다 무작위화합니다. `cat /proc/sys/kernel/randomize_va_space`로 활성화 여부를 확인할 수 있습니다.

**4. 메모리 보호 레이어**
- **NX bit (No-eXecute)**: 힙/스택을 실행 불가로 표시해 쉘코드 실행 방지
- **Stack Canary**: 함수 반환 전 스택 쿠키 값을 검증해 버퍼 오버플로우 탐지
- **PIE (Position Independent Executable)**: 텍스트 세그먼트도 ASLR 적용

**5. 공유 라이브러리의 메모리 공유**
`libc.so.6`의 텍스트 세그먼트는 모든 프로세스가 물리 메모리에서 공유합니다. 데이터 수정은 Copy-on-Write로 각 프로세스가 독립적인 복사본을 갖습니다.

---

## 참고 자료

- [Anatomy of a Program in Memory - Many But Finite](https://manybutfinite.com/post/anatomy-of-a-program-in-memory/)
- [Linux Processes Memory Layout - The Geek Stuff](https://www.thegeekstuff.com/2012/03/linux-processes-memory-layout/)
- [Linux process address space layout](https://celery1124.github.io/Linux-process-address-space-layout/)
- [Process Address Space - Linux Kernel Internals](https://kernel-internals.org/mm/mmap/)
