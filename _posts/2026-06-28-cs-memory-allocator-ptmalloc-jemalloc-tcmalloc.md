---
layout: post
title: "메모리 할당자 내부 구조 완전 정복: ptmalloc, jemalloc, tcmalloc의 설계 철학"
date: 2026-06-28
categories: [cs, computer-science]
tags: [memory-allocator, ptmalloc, jemalloc, tcmalloc, heap, slab-allocator, fragmentation, malloc]
---

`malloc()`을 호출하면 어떤 일이 벌어질까? 운영체제에 직접 매번 시스템 콜을 보내는 게 아니라, C 런타임이 관리하는 **메모리 할당자(memory allocator)**가 중간에서 크기별로 분류된 블록을 효율적으로 관리한다. 이 할당자의 설계 품질이 애플리케이션의 처리량·메모리 사용량·지연 시간을 직접 결정한다.

## 왜 메모리 할당자가 복잡한가

단순히 "OS에서 큰 덩어리를 받아 나눠주면 되지 않느냐"고 생각할 수 있지만, 실제로는 세 가지 상충 목표를 동시에 만족해야 한다.

1. **속도**: 할당·해제가 나노초 단위로 빠야 한다. JVM의 `new Object()`는 초당 수억 번 호출될 수 있다.
2. **공간 효율**: 메모리 단편화(fragmentation)를 최소화해야 한다. 1GB를 할당했다 해제해도 실제 가용 메모리가 100MB밖에 없는 상황을 막아야 한다.
3. **멀티스레드 확장성**: 여러 스레드가 동시에 `malloc`을 호출해도 락 경합(lock contention)이 없어야 한다.

이 세 목표는 서로 상충한다. 빠르게 하려면 스레드별 캐시가 필요하지만 그러면 메모리를 더 쓴다. 공간을 아끼려면 통합(coalescing)이 필요하지만 그러면 락이 필요하다.

## 주요 개념: 할당자 공통 용어

**내부 단편화(Internal Fragmentation)**: 8바이트 요청에 16바이트 블록을 주면 8바이트가 낭비된다.

**외부 단편화(External Fragmentation)**: 100바이트 공간이 있지만 10바이트씩 10개 조각으로 나뉘어 있어 연속 60바이트를 할당할 수 없는 상황.

**Arena**: 독립적인 힙 영역. 각 아레나는 자체 락을 가지므로 서로 다른 아레나에서의 할당은 경합 없이 병렬로 진행된다.

**Slab**: 동일한 크기의 객체를 미리 여러 개 잘라둔 메모리 풀. 빠른 할당과 단편화 최소화를 동시에 달성한다.

## ptmalloc: glibc의 기본 할당자

ptmalloc(pthreads malloc)은 Doug Lea의 dlmalloc을 멀티스레드 환경으로 확장한 것으로, 리눅스 glibc의 기본 `malloc` 구현체다.

### 핵심 구조: Chunk

모든 메모리 블록은 **Chunk**라는 구조체로 관리된다.

```
+---------+---------+---------+--------+
| prev_sz |   size  |  data   |  ...   |
|  (8B)   |  (8B)   |  사용자  |       |
+---------+---------+---------+--------+
```

- `prev_sz`: 이전 chunk가 해제된 경우 그 크기 (이웃 chunk 합병에 사용)
- `size`의 하위 3비트: 플래그 (PREV_INUSE, IS_MMAPPED, NON_MAIN_ARENA)

### Bins: 크기별 프리 리스트

해제된 chunk는 크기에 따라 분류된다:

| Bin 종류 | 크기 범위 | 개수 | 특징 |
|----------|-----------|------|------|
| Fast bins | 16~176B | 10 | LIFO, 합병 없음, 가장 빠름 |
| Small bins | 32~512B | 62 | 고정 크기, FIFO, 이전/다음 합병 |
| Large bins | 512B+ | 63 | 가변 크기, best-fit |
| Unsorted bin | 모든 크기 | 1 | 새로 해제된 chunk의 임시 보관 |

### Arena 구조와 멀티스레드

```c
struct malloc_state {
    mutex_t mutex;          // 아레나 락
    mchunkptr top;          // 힙 끝 chunk
    mchunkptr last_remainder;
    mchunkptr fastbinsY[NFASTBINS];  // fast bins
    mchunkptr bins[NBINS * 2 - 2];  // small/large/unsorted bins
    // ...
};
```

메인 아레나는 `brk()` 시스템 콜로 힙을 확장하고, 추가 아레나는 `mmap()`으로 독립 영역을 만든다. 아레나 개수는 최대 `8 × CPU_CORES`로 제한된다.

**ptmalloc의 약점**: 스레드가 아레나에 동적으로 배정되는데, 인기 아레나에 스레드가 몰리면 락 경합이 발생한다. 또한 해제된 메모리를 OS에 즉시 반환하지 않아 RSS(Resident Set Size)가 비대해질 수 있다.

## jemalloc: Facebook과 FreeBSD가 선택한 할당자

jemalloc은 Jason Evans가 2006년 FreeBSD 커널용으로 설계했고, 이후 Firefox, Meta(Facebook), Redis가 채택했다.

### 예제 1: jemalloc 크기 클래스 시스템 이해

```c
// jemalloc의 size class 계산 시뮬레이션 (개념 코드)
#include <stdio.h>
#include <stddef.h>

// jemalloc은 크기를 약 10% 간격으로 구간화
// 실제 size class 예시 (일부)
size_t size_classes[] = {
    8, 16, 32, 48, 64, 80, 96, 112, 128,  // tiny (8B 단위)
    160, 192, 224, 256,                     // small (32B 단위)
    320, 384, 448, 512,                     // small (64B 단위)
    640, 768, 896, 1024,                    // small (128B 단위)
    1280, 1536, 1792, 2048,                 // ...
    2560, 3072, 3584, 4096,
};

size_t jemalloc_size_class(size_t requested) {
    for (size_t i = 0; i < sizeof(size_classes)/sizeof(size_t); i++) {
        if (requested <= size_classes[i])
            return size_classes[i];
    }
    return (requested + 4095) & ~4095;  // 4KB 정렬
}

int main() {
    // 실제 할당 크기 예측
    printf("요청 1B  → 할당 %zuB\n", jemalloc_size_class(1));    // 8
    printf("요청 17B → 할당 %zuB\n", jemalloc_size_class(17));   // 32
    printf("요청 100B → 할당 %zuB\n", jemalloc_size_class(100)); // 112
    printf("요청 500B → 할당 %zuB\n", jemalloc_size_class(500)); // 512

    // 내부 단편화 계산
    size_t req = 100, alloc = jemalloc_size_class(100);
    printf("내부 단편화: %.1f%%\n", (double)(alloc - req) / alloc * 100); // 10.7%
    return 0;
}
```

jemalloc의 크기 클래스는 약 87개로, 요청 크기와 실제 할당 크기의 차이(내부 단편화)를 최대 ~12.5%로 제한한다.

### Extent와 Slab 시스템

jemalloc은 OS에서 **Extent**(보통 2MB 또는 1GB 거대 페이지)를 받아와 내부적으로 **Slab**으로 분할한다. 같은 크기 클래스의 Slab들은 별도 arenas에 속하며, 각 arena는 특정 스레드 집합에 전용으로 배정된다.

```
Arena
├── tcache (스레드 로컬 캐시)
│   ├── size class 8B   → 최근 해제된 청크 LIFO 스택
│   ├── size class 16B  → ...
│   └── ...
├── Extent (2MB slab 모음)
│   ├── Slab (4KB, 32B 크기 클래스용 → 128개 슬롯)
│   ├── Slab (4KB, 64B 크기 클래스용 → 64개 슬롯)
│   └── ...
└── Large Extent (>4KB 직접 관리)
```

tcache가 가득 차거나 비면 해당 arena로 배치(flush/fill)되어 락 경합을 최소화한다.

## tcmalloc: Google이 설계한 스레드 캐시 할당자

tcmalloc(Thread-Caching Malloc)은 구글이 Chrome, Bigtable, Spanner에 사용하기 위해 설계한 할당자로, gperftools에 포함되어 오픈소스로 공개되었다. 현재 구글은 TCMalloc이라는 이름으로 완전히 재작성된 버전을 공개했다.

### 핵심: Per-CPU 캐시

최신 TCMalloc의 가장 큰 특징은 **Per-CPU Cache**다. 과거의 Per-Thread Cache는 스레드 수가 많아지면 메모리 낭비가 발생했다. Per-CPU Cache는 논리 CPU 수만큼의 캐시만 유지해 메모리를 훨씬 효율적으로 사용한다.

### 예제 2: 할당자 성능 벤치마크 (C++)

```cpp
#include <cstdlib>
#include <chrono>
#include <vector>
#include <thread>
#include <iostream>

// 단순 할당/해제 성능 테스트
void benchmark_malloc(int thread_id, int iterations, int alloc_size) {
    std::vector<void*> ptrs;
    ptrs.reserve(iterations);

    auto start = std::chrono::high_resolution_clock::now();

    for (int i = 0; i < iterations; i++) {
        ptrs.push_back(malloc(alloc_size));
    }
    for (void* p : ptrs) {
        free(p);
    }

    auto end = std::chrono::high_resolution_clock::now();
    auto us = std::chrono::duration_cast<std::chrono::microseconds>(end - start).count();
    std::cout << "Thread " << thread_id
              << " | " << iterations << "x " << alloc_size << "B"
              << " | " << us << "μs"
              << " | " << (double)us / iterations * 1000 << "ns/op\n";
}

int main() {
    const int THREADS = 8;
    const int ITER = 100000;

    std::cout << "=== 단일 스레드 ===\n";
    benchmark_malloc(0, ITER, 64);

    std::cout << "\n=== " << THREADS << " 스레드 동시 ===\n";
    std::vector<std::thread> threads;
    for (int t = 0; t < THREADS; t++) {
        threads.emplace_back(benchmark_malloc, t, ITER, 64);
    }
    for (auto& th : threads) th.join();

    // 크기별 성능 차이
    std::cout << "\n=== 크기별 할당 성능 ===\n";
    for (int sz : {8, 64, 512, 4096, 65536}) {
        benchmark_malloc(0, 10000, sz);
    }
    return 0;
}

/* 일반적인 결과 (ptmalloc 기준):
   단일 스레드: ~30ns/op
   8스레드 ptmalloc: ~150ns/op (락 경합)
   8스레드 jemalloc:  ~40ns/op (아레나 분리)
   8스레드 tcmalloc:  ~25ns/op (per-CPU 캐시)
*/
```

멀티스레드 환경에서 ptmalloc은 락 경합으로 성능이 크게 저하되지만, jemalloc과 tcmalloc은 스레드별·CPU별 캐시 덕분에 거의 선형으로 확장된다.

## 세 할당자 비교

| 항목 | ptmalloc | jemalloc | tcmalloc |
|------|----------|----------|----------|
| 기본 사용처 | glibc (Linux) | FreeBSD, Firefox, Redis | Chrome, Google 내부 |
| 단편화 제어 | 보통 | 우수 (크기 클래스 세분화) | 우수 |
| 멀티스레드 성능 | 보통 (아레나 경합) | 우수 | 매우 우수 (Per-CPU) |
| 메모리 반환 | 느림 | 빠름 (MADV_FREE) | 빠름 |
| 프로파일링 | `/proc/malloc_info` | `jemalloc.stats` | `pprof` 연동 |
| 적합한 워크로드 | 범용, 단순 앱 | 장기 실행 서버, DB | 고성능 멀티스레드 서비스 |

## 실전 적용 팁

### 할당자 교체 방법 (Linux)

```bash
# jemalloc으로 교체 (LD_PRELOAD 방식)
sudo apt-get install libjemalloc-dev
LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libjemalloc.so.2 ./my_server

# tcmalloc으로 교체
sudo apt-get install libgoogle-perftools-dev
LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libtcmalloc.so.4 ./my_server

# 실제 사용 중인 할당자 확인
ldd ./my_server | grep -E "jemalloc|tcmalloc"
```

### jemalloc 메모리 프로파일링

```bash
# jemalloc heap profiling 활성화
MALLOC_CONF="prof:true,prof_leak:true,lg_prof_sample:17" \
  LD_PRELOAD=libjemalloc.so ./my_app

# 힙 덤프 생성 후 분석
jeprof --pdf ./my_app jeprof.0.0.heap > heap_profile.pdf
```

## 주의사항

**1. glibc에서 메모리가 OS로 반환되지 않는 문제**
ptmalloc은 힙의 중간 chunk가 해제되어도 끝 chunk(top chunk)가 살아있으면 OS에 반환하지 않는다. 장기 실행 서버에서 RSS가 줄지 않는 이유다. 해결책: jemalloc으로 교체하거나 주기적으로 `malloc_trim(0)` 호출.

**2. false sharing과 캐시 라인**
Per-CPU 캐시가 아무리 좋아도, 할당된 메모리가 여러 스레드에서 동시에 접근되면 CPU 캐시 라인 무효화(cache line invalidation)로 성능이 저하된다. 할당자와는 별개의 문제다.

**3. 크기를 초과하는 할당**
jemalloc/tcmalloc 모두 특정 임계값(보통 4KB~32KB) 이상은 직접 `mmap()`으로 처리한다. 이 경우 tcache를 거치지 않으므로 소규모 할당보다 느릴 수 있다.

**4. Sanitizer와의 충돌**
AddressSanitizer(ASan)나 Valgrind는 자체 메모리 추적을 위해 기본 할당자를 교체한다. jemalloc/tcmalloc과 함께 사용하면 충돌이 발생하므로, 프로파일링 빌드와 디버그 빌드를 분리하라.

현대적인 고성능 서버(Redis, RocksDB, MySQL, Nginx)가 기본 ptmalloc 대신 jemalloc이나 tcmalloc을 선택한 것은 이유가 있다. 할당자를 바꾸는 것만으로 서버 처리량이 10~30% 향상되는 사례도 드물지 않다.

## 참고 자료
- [jemalloc 공식 문서 - jemalloc.net](https://jemalloc.net/)
- [TCMalloc Design - Google](https://google.github.io/tcmalloc/design.html)
- [Exploring Different Memory Allocators - DEV Community](https://dev.to/frosnerd/libmalloc-jemalloc-tcmalloc-mimalloc-exploring-different-memory-allocators-4lp3)
- [Testing Memory Allocators: ptmalloc2 vs tcmalloc vs hoard vs jemalloc - IT Hare](http://ithare.com/testing-memory-allocators-ptmalloc2-tcmalloc-hoard-jemalloc-while-trying-to-simulate-real-world-loads/)
