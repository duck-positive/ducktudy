---
layout: post
title: "CPU 캐시와 메모리 계층 구조: 캐시 친화적 코드가 성능을 바꾸는 이유"
date: 2026-06-24
categories: [cs, computer-science]
tags: [cpu-cache, memory-hierarchy, cache-coherence, MESI, NUMA, performance, cache-friendly]
---

## 메모리 계층 구조와 속도 격차

현대 CPU와 메인 메모리(RAM) 사이에는 엄청난 속도 격차가 존재한다. CPU는 1 사이클에 연산을 수행하지만, RAM에서 데이터를 읽어오는 데는 수백 사이클이 걸린다. 이 격차를 채우기 위해 등장한 것이 **CPU 캐시** 다.

| 메모리 종류 | 접근 지연 (대략) | 용량 (일반적) |
|-----------|----------------|-------------|
| CPU 레지스터 | ~1 사이클 | 수백 바이트 |
| L1 캐시 | ~4 사이클 | 32~64 KB (코어당) |
| L2 캐시 | ~12 사이클 | 256 KB ~ 1 MB (코어당) |
| L3 캐시 | ~40 사이클 | 8~64 MB (CPU 공유) |
| DRAM (RAM) | ~200 사이클 | 수 GB ~ 수십 GB |
| NVMe SSD | ~100,000 사이클 | 수 TB |

이 숫자를 시간 단위로 환산하면, L1 캐시 접근이 "부엌에서 냉장고 열기"라면 DRAM 접근은 "마트에 다녀오기"와 같다는 비유가 성립한다.

## 캐시 라인(Cache Line): 캐시의 기본 단위

캐시는 **바이트 단위**가 아니라 **캐시 라인(Cache Line)** 단위로 데이터를 가져온다. x86-64 아키텍처에서 캐시 라인 크기는 **64 바이트** 다.

즉, `int arr[16]`(64바이트)의 첫 번째 원소를 읽으면, 나머지 15개 원소도 자동으로 L1 캐시에 올라온다. 이것이 배열의 순차 접근이 랜덤 접근보다 월등히 빠른 이유다.

```
메모리 주소:  0x1000  0x1040  0x1080  ...
캐시 라인:  [64바이트][64바이트][64바이트]

arr[0] 접근 → 캐시 미스 → 0x1000~0x103F (64바이트) 전체를 RAM에서 캐시로 로드
arr[1] 접근 → 캐시 히트 (이미 캐시에 있음)
...
arr[15] 접근 → 캐시 히트
arr[16] 접근 → 캐시 미스 (다음 캐시 라인)
```

## 코드 예제: 행 우선 vs 열 우선 접근

### 예제 1: 행렬 순회 순서에 따른 성능 차이 (C)

```c
#include <stdio.h>
#include <time.h>
#include <string.h>

#define N 4096

static double matrix[N][N];

// 행 우선(row-major) 순회 — 캐시 친화적
double row_major_sum() {
    double sum = 0.0;
    for (int i = 0; i < N; i++)
        for (int j = 0; j < N; j++)
            sum += matrix[i][j];  // 연속된 메모리 위치 접근
    return sum;
}

// 열 우선(column-major) 순회 — 캐시 비친화적
double col_major_sum() {
    double sum = 0.0;
    for (int j = 0; j < N; j++)
        for (int i = 0; i < N; i++)
            sum += matrix[i][j];  // 행 간격(N*8 바이트)마다 점프 → 캐시 미스
    return sum;
}

int main() {
    // 초기화
    for (int i = 0; i < N; i++)
        for (int j = 0; j < N; j++)
            matrix[i][j] = (double)(i * N + j);

    struct timespec t0, t1;

    clock_gettime(CLOCK_MONOTONIC, &t0);
    double s1 = row_major_sum();
    clock_gettime(CLOCK_MONOTONIC, &t1);
    double row_ms = (t1.tv_sec - t0.tv_sec) * 1000.0
                  + (t1.tv_nsec - t0.tv_nsec) / 1e6;

    clock_gettime(CLOCK_MONOTONIC, &t0);
    double s2 = col_major_sum();
    clock_gettime(CLOCK_MONOTONIC, &t1);
    double col_ms = (t1.tv_sec - t0.tv_sec) * 1000.0
                  + (t1.tv_nsec - t0.tv_nsec) / 1e6;

    printf("Row-major: %.1f ms (sum=%.0f)\n", row_ms, s1);
    printf("Col-major: %.1f ms (sum=%.0f)\n", col_ms, s2);
    printf("Speedup: %.1fx\n", col_ms / row_ms);
    return 0;
}

// 실행 결과 (예시, Intel Core i7 기준):
// Row-major:  18.3 ms
// Col-major: 142.7 ms
// Speedup: 7.8x
```

C는 **행 우선(Row-Major)** 레이아웃을 사용하므로, `matrix[i][j]`와 `matrix[i][j+1]`은 메모리에서 연속이다. 열 우선 순회를 하면 매 접근마다 4096 × 8 = 32,768 바이트를 건너뛰어 캐시 미스가 연속 발생한다.

## 캐시 일관성(Cache Coherence)과 MESI 프로토콜

멀티코어 CPU에서 각 코어는 독립적인 L1/L2 캐시를 가진다. 코어 A가 `x`를 수정했을 때, 코어 B의 캐시에 있는 `x`의 복사본은 구식(stale)이 된다. 이 문제를 해결하는 것이 **캐시 일관성 프로토콜(Cache Coherence Protocol)** 이다.

가장 널리 쓰이는 프로토콜이 **MESI** 다. 각 캐시 라인은 네 가지 상태 중 하나다:

| 상태 | 의미 |
|------|------|
| **M (Modified)** | 이 코어만 갖고 있으며, RAM과 값이 다름 (dirty). 다른 코어에 없음 |
| **E (Exclusive)** | 이 코어만 갖고 있으며, RAM과 값이 같음 (clean) |
| **S (Shared)** | 여러 코어가 동일한 값을 가짐 (read-only 공유) |
| **I (Invalid)** | 이 캐시 라인은 유효하지 않음 (비어있거나 무효화됨) |

```
코어 A가 캐시 라인 읽기:
  I → E (다른 코어에 없으면) 또는 I → S (다른 코어가 가지고 있으면)

코어 A가 캐시 라인 쓰기:
  E → M (이미 Exclusive면 바로)
  S → M (다른 모든 코어에 Invalidate 신호 전송 → I로 전환 후 M)
  I → M (캐시 미스 → 로드 → 다른 코어 무효화 → M)
```

이 프로토콜 때문에 **거짓 공유(False Sharing)** 문제가 발생한다.

## 거짓 공유(False Sharing): 멀티스레드 성능의 숨은 적

두 스레드가 **서로 다른 변수**를 독립적으로 수정하는데, 그 변수들이 같은 캐시 라인에 있다면?

```
캐시 라인 (64 바이트):
[counter_A (8B)][counter_B (8B)][...나머지 패딩...]

코어 0 스레드: counter_A++ → 캐시 라인을 M 상태로
코어 1 스레드: counter_B++ → 코어 0의 캐시 라인을 I로 무효화하고 가져옴
코어 0 스레드: counter_A++ → 다시 코어 1의 캐시 라인을 I로...
→ 두 코어가 같은 캐시 라인을 계속 무효화하며 핑퐁
```

실제 코드에서는 논리적으로 완전히 독립적인 카운터인데, 물리적 배치가 같은 캐시 라인이라 심각한 성능 저하가 발생한다.

### 예제 2: 거짓 공유 탐지 및 수정 (C++)

```cpp
#include <thread>
#include <atomic>
#include <chrono>
#include <iostream>
#include <vector>

static const int ITERATIONS = 100'000'000;

// ===== 거짓 공유 발생 구조 =====
struct FalseSharingCounters {
    std::atomic<long> a{0};  // 같은 캐시 라인에 공존
    std::atomic<long> b{0};
};

// ===== 패딩으로 거짓 공유 방지 =====
struct alignas(64) PaddedCounter {
    std::atomic<long> value{0};
    // 캐시 라인 나머지를 패딩으로 채워 다음 카운터와 라인을 분리
    char padding[64 - sizeof(std::atomic<long>)];
};

struct AlignedCounters {
    PaddedCounter a;
    PaddedCounter b;
};

template<typename T>
auto benchmark(T& counters) {
    auto start = std::chrono::high_resolution_clock::now();
    std::thread t1([&]{ for (int i = 0; i < ITERATIONS; i++) counters.a.value++; });
    std::thread t2([&]{ for (int i = 0; i < ITERATIONS; i++) counters.b.value++; });
    // FalseSharing 버전은 counters.a / counters.b로 직접 접근
    t1.join(); t2.join();
    return std::chrono::high_resolution_clock::now() - start;
}

int main() {
    FalseSharingCounters bad;
    std::thread t1_bad([&]{ for(int i=0;i<ITERATIONS;i++) bad.a++; });
    std::thread t2_bad([&]{ for(int i=0;i<ITERATIONS;i++) bad.b++; });
    auto t_bad_start = std::chrono::high_resolution_clock::now();
    t1_bad.join(); t2_bad.join();
    auto bad_time = std::chrono::high_resolution_clock::now() - t_bad_start;

    AlignedCounters good;
    auto good_time = benchmark(good);

    std::cout << "False sharing:  "
              << std::chrono::duration_cast<std::chrono::milliseconds>(bad_time).count()
              << " ms\n";
    std::cout << "Cache-aligned:  "
              << std::chrono::duration_cast<std::chrono::milliseconds>(good_time).count()
              << " ms\n";
    return 0;
}

// 예상 결과:
// False sharing:  1842 ms
// Cache-aligned:   221 ms  (약 8배 차이)
```

## 캐시 프리페치(Prefetching)

CPU는 앞으로 어떤 메모리 주소에 접근할지 예측해 미리 캐시로 가져오는 **하드웨어 프리페처(Hardware Prefetcher)** 를 내장한다. 순차 접근 패턴은 하드웨어 프리페처가 잘 예측하지만, 포인터 체이싱(linked list 순회 등) 패턴은 예측이 어렵다.

소프트웨어에서 명시적으로 프리페치를 지시할 수도 있다:

```c
// GCC 빌트인 프리페치 힌트
// locality: 0=L3, 1=L2, 2=L1, 3=레지스터 (높을수록 가까운 캐시)
__builtin_prefetch(&arr[i + 16], 0, 1);  // 16 원소 앞을 미리 L2에 올려둠
```

## NUMA(Non-Uniform Memory Access)

멀티 소켓(Multi-Socket) 서버에서는 각 CPU 소켓이 자신의 DRAM에 직접 연결되어 있다. 다른 소켓의 메모리에 접근하려면 **인터커넥트(QPI/UPI)** 를 거쳐야 해 지연이 2~3배 증가한다.

```
소켓 0              소켓 1
┌─────────────┐    ┌─────────────┐
│  CPU 코어들  │    │  CPU 코어들  │
│  L1/L2/L3  │    │  L1/L2/L3  │
│  ↕ 빠름     │    │  ↕ 빠름     │
│  DRAM 0     │──QPI──│  DRAM 1     │
└─────────────┘    └─────────────┘
  로컬 접근 빠름     원격 접근 느림
```

Linux에서 NUMA를 고려한 메모리 할당:

```bash
# NUMA 토폴로지 확인
numactl --hardware

# 특정 NUMA 노드에서 프로세스 실행
numactl --cpunodebind=0 --membind=0 ./myapp

# Java JVM에 NUMA 인식 활성화
java -XX:+UseNUMA -XX:+UseParallelGC MyApp
```

## 실전 캐시 최적화 체크리스트

### 1. 데이터 레이아웃: AoS vs SoA

```cpp
// Array of Structures (AoS) — 각 객체의 모든 필드가 붙어있음
// 객체 전체를 자주 처리할 때 유리
struct Particle { float x, y, z, mass, vx, vy, vz; };
Particle particles[1000];

// Structure of Arrays (SoA) — 같은 종류의 필드끼리 모음
// 특정 필드만 SIMD로 한꺼번에 처리할 때 유리 (물리 엔진, 게임 등)
struct Particles {
    float x[1000], y[1000], z[1000];
    float mass[1000];
    float vx[1000], vy[1000], vz[1000];
};

// 위치 업데이트만 할 때: SoA가 유리
// → x, y, z 배열만 캐시에 로드; mass/velocity는 접근 안 함
```

### 2. 핫/콜드 데이터 분리

자주 접근하는 데이터(hot)와 드물게 접근하는 데이터(cold)를 분리해, hot 데이터만 캐시에 들어오게 한다.

```cpp
// 비효율: 자주 쓰는 name/age와 드물게 쓰는 address가 같은 구조체
struct User {
    char name[64];      // hot
    int age;            // hot
    char address[256];  // cold — 쿼리 시 드물게 접근
    char bio[512];      // cold
};

// 개선: 핫/콜드 분리
struct UserHot { char name[64]; int age; };
struct UserCold { char address[256]; char bio[512]; };
UserHot hot_users[N];     // 자주 접근하는 배열
UserCold* cold_users[N];  // 필요할 때만 포인터 역참조
```

### 3. 분기 예측과 캐시 미스의 교차 효과

```c
// 분기 예측 실패 + 캐시 미스가 겹치면 최악
// 미리 정렬해두면 두 가지 모두 개선됨
void process_items(Item* items, int n) {
    // 정렬 없이 처리: 랜덤 is_valid 분포 → 분기 예측 실패
    for (int i = 0; i < n; i++)
        if (items[i].is_valid) do_work(&items[i]);

    // 개선: is_valid == true인 항목을 앞으로 파티셔닝
    // → 분기 예측 개선 + 핫 데이터가 연속 배치되어 캐시 히트율 증가
}
```

## 성능 측정 도구

```bash
# Linux perf로 캐시 미스 확인
perf stat -e cache-misses,cache-references,L1-dcache-load-misses ./myapp

# 예시 출력:
# 142,847,291      cache-misses              # 15.23% of all cache refs
#   937,482,021      cache-references
#   89,234,121      L1-dcache-load-misses     # 2.14% of all L1-dcache accesses

# Valgrind cachegrind로 상세 캐시 시뮬레이션
valgrind --tool=cachegrind ./myapp
cg_annotate cachegrind.out.* --auto=yes
```

## 마치며

CPU 캐시는 현대 컴퓨터 성능의 숨은 중재자다. 알고리즘의 시간 복잡도가 같아도, 메모리 접근 패턴에 따라 실제 성능은 10배 이상 차이 날 수 있다. 핵심을 정리하면:

1. **연속된 메모리를 순서대로 접근하라** — 캐시 라인 단위로 미리 올라온다.
2. **거짓 공유를 피하라** — 독립적으로 수정하는 데이터는 다른 캐시 라인에 배치하라.
3. **핫 데이터와 콜드 데이터를 분리하라** — 자주 쓰는 데이터만 캐시에 들어오게 하라.
4. **NUMA를 의식하라** — 멀티 소켓 서버에서 원격 메모리 접근은 비싸다.

코드의 로직이 아닌 **메모리 레이아웃**을 바꾸는 것만으로도 CPU가 진정한 속도를 낼 수 있다.

## 참고 자료
- [Wikipedia: CPU cache](https://en.wikipedia.org/wiki/CPU_cache)
- [Wikipedia: MESI protocol](https://en.wikipedia.org/wiki/MESI_protocol)
- [Wikipedia: Non-uniform memory access](https://en.wikipedia.org/wiki/Non-uniform_memory_access)
- [Linux perf wiki — Tutorial](https://perf.wiki.kernel.org/index.php/Tutorial)
