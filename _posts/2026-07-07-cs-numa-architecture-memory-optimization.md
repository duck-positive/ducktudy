---
layout: post
title: "NUMA 아키텍처 완전 정복: 멀티소켓 서버에서 메모리 지역성으로 성능을 2배 올리는 법"
date: 2026-07-07
categories: [cs, computer-science]
tags: [numa, memory, cpu, architecture, linux, performance, hardware, cache, threading]
---

현대 고성능 서버는 대부분 2개 이상의 CPU 소켓을 장착한다. 이 구성에서 모든 CPU가 모든 메모리를 동일한 속도로 접근할 수 있다고 생각하면 큰 오산이다. NUMA(Non-Uniform Memory Access) 아키텍처를 이해하지 못하면 코드는 올바르게 동작하지만 성능은 이론치의 절반도 나오지 않을 수 있다. 이 글에서는 NUMA의 하드웨어 원리부터 Linux 커널의 처리 방식, 그리고 실제 성능 최적화 기법까지 다룬다.

## NUMA란 무엇인가

### UMA의 한계와 NUMA의 등장

전통적인 **UMA(Uniform Memory Access)** 아키텍처는 모든 CPU가 단일 공유 메모리 버스를 통해 메모리에 접근한다. CPU 수가 늘어날수록 이 버스가 병목이 된다. 8개, 16개 CPU가 하나의 버스를 놓고 경쟁하면 성능이 오히려 저하된다.

**NUMA**는 이 문제를 해결하기 위해 메모리를 CPU별로 분산 배치한다. 각 CPU(소켓)는 자신에게 물리적으로 가까운 **로컬 메모리(local memory)**를 빠르게 접근할 수 있고, 다른 CPU의 메모리인 **원격 메모리(remote memory)**는 인터커넥트(Intel의 QPI/UPI, AMD의 Infinity Fabric)를 통해 접근한다.

```
[NUMA Node 0]                [NUMA Node 1]
+----------+                 +----------+
|  CPU 0   |                 |  CPU 1   |
|  (코어    |                 |  (코어    |
|  0~7)    |                 |  8~15)   |
+----+-----+                 +-----+----+
     |                             |
+----+-----+   QPI/UPI/IF    +-----+----+
| 로컬 DRAM |<--------------->| 로컬 DRAM|
| (32 GB)  |  (고지연, 저대역폭) | (32 GB)  |
+----------+                 +----------+
```

### 접근 지연 시간의 차이

일반적인 2-소켓 서버에서의 메모리 접근 지연 시간:

| 접근 유형 | 지연 시간 |
|----------|---------|
| L1 캐시 | ~4 사이클 |
| L2 캐시 | ~12 사이클 |
| L3 캐시 | ~40 사이클 |
| 로컬 DRAM | ~70 ns (약 200 사이클) |
| 원격 DRAM (1 hop) | ~100~130 ns (약 300~400 사이클) |
| 원격 DRAM (2 hop) | ~150~200 ns (더 많은 노드) |

원격 메모리 접근은 로컬보다 **1.5~2.8배** 느리다. 메모리 집약적 워크로드에서는 이 차이가 전체 처리량을 결정한다.

## Linux 커널의 NUMA 지원

### NUMA 토폴로지 확인

```bash
# NUMA 노드 구성 확인
numactl --hardware

# 출력 예시:
# available: 2 nodes (0-1)
# node 0 cpus: 0 1 2 3 4 5 6 7 16 17 18 19 20 21 22 23
# node 0 size: 32752 MB
# node 0 free: 28100 MB
# node 1 cpus: 8 9 10 11 12 13 14 15 24 25 26 27 28 29 30 31
# node 1 size: 32768 MB
# node 1 free: 30200 MB
# node distances:
# node   0   1
#   0:  10  21
#   1:  21  10

# 시스템 토폴로지 시각화
lstopo-no-graphics

# NUMA 통계 확인
numastat -c
```

`node distances`에서 동일 노드 거리가 10이고 원격이 21이라면, 원격 접근은 약 2.1배 느리다는 의미다.

### Linux의 메모리 할당 정책

Linux 커널은 기본적으로 **first-touch 정책**을 따른다: 메모리 페이지에 처음 쓰기가 발생하는 순간, 해당 CPU의 NUMA 노드에서 물리 메모리를 할당한다. 따라서 **초기화 스레드가 어느 NUMA 노드에서 실행되느냐**가 전체 성능을 좌우한다.

```c
#include <numa.h>
#include <numaif.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

/* NUMA 정책별 메모리 할당 방법 */

void demonstrate_numa_allocation() {
    size_t size = 1024 * 1024 * 256; // 256 MB

    /* 1. 기본 할당 (first-touch 정책) */
    char *default_mem = malloc(size);

    /* 2. 특정 노드에 강제 할당 */
    char *node0_mem = numa_alloc_onnode(size, 0); // 노드 0에서만
    char *node1_mem = numa_alloc_onnode(size, 1); // 노드 1에서만

    /* 3. 인터리브 할당 (대역폭 집약적 워크로드에 유리) */
    struct bitmask *all_nodes = numa_all_nodes_ptr;
    char *interleaved = numa_alloc_interleaved_subset(size, all_nodes);

    /* 메모리 정책 확인 */
    int mode;
    struct bitmask *nodemask = numa_bitmask_alloc(numa_num_configured_nodes());
    get_mempolicy(&mode, nodemask->maskp, nodemask->size, NULL, 0);
    printf("Current memory policy: %d\n", mode);
    // MPOL_DEFAULT=0, MPOL_BIND=2, MPOL_INTERLEAVE=3, MPOL_PREFERRED=1

    /* 해제 */
    numa_free(node0_mem, size);
    numa_free(node1_mem, size);
    numa_free(interleaved, size);
    free(default_mem);
    numa_bitmask_free(nodemask);
}
```

### NUMA 불균형이 발생하는 대표적 시나리오

**시나리오 1: 마스터-워커 패턴의 함정**

```c
#include <pthread.h>
#include <numa.h>

#define ARRAY_SIZE (64 * 1024 * 1024)  // 64M integers

int *data;

/* 나쁜 패턴: 마스터 스레드(노드 0)가 모든 메모리를 초기화 */
void bad_initialization() {
    data = malloc(sizeof(int) * ARRAY_SIZE);
    // first-touch: 모든 페이지가 노드 0에 할당됨
    memset(data, 0, sizeof(int) * ARRAY_SIZE);
    // → 노드 1의 워커 스레드가 접근할 때마다 원격 메모리 접근 발생!
}

/* 좋은 패턴: 각 워커가 자신의 영역을 초기화 */
typedef struct { int start; int end; } WorkerArgs;

void *worker_init(void *arg) {
    WorkerArgs *wa = (WorkerArgs *)arg;
    // 이 스레드가 실행되는 NUMA 노드에서 first-touch 발생
    for (int i = wa->start; i < wa->end; i++) {
        data[i] = 0;
    }
    return NULL;
}

void good_initialization() {
    data = malloc(sizeof(int) * ARRAY_SIZE);

    int num_threads = 2;
    pthread_t threads[num_threads];
    WorkerArgs args[num_threads];

    // 스레드를 각 NUMA 노드에 바인딩
    for (int t = 0; t < num_threads; t++) {
        args[t].start = (ARRAY_SIZE / num_threads) * t;
        args[t].end   = (ARRAY_SIZE / num_threads) * (t + 1);

        cpu_set_t cpuset;
        CPU_ZERO(&cpuset);
        // 스레드 t를 NUMA 노드 t의 첫 번째 CPU에 바인딩
        int cpu = numa_node_to_cpus_ptr(t)->maskp[0];
        CPU_SET(__builtin_ctzl(cpu), &cpuset);

        pthread_create(&threads[t], NULL, worker_init, &args[t]);
        pthread_setaffinity_np(threads[t], sizeof(cpuset), &cpuset);
    }

    for (int t = 0; t < num_threads; t++) {
        pthread_join(threads[t], NULL);
    }
    // → 각 영역이 해당 노드 메모리에 위치, 워커가 로컬 접근
}
```

## 실전 최적화: numactl과 소프트웨어 설계

### numactl로 프로세스 바인딩

```bash
# 노드 0의 CPU와 메모리만 사용
numactl --cpunodebind=0 --membind=0 ./my_server

# CPU는 노드 0, 메모리는 인터리브 (대역폭 집약적 앱)
numactl --cpunodebind=0 --interleave=all ./my_app

# 특정 CPU 코어 집합 바인딩
numactl --physcpubind=0,1,2,3 --membind=0 ./worker

# 실시간 NUMA 접근 통계 모니터링
watch -n 1 numastat -c
```

### Java/JVM 환경에서의 NUMA 최적화

```bash
# JVM NUMA 인식 활성화 (Java 8+, Parallel GC와 함께)
java -XX:+UseNUMA -XX:+UseParallelGC -Xmx8g -jar application.jar

# G1GC와 NUMA (Java 15+ 에서 지원 강화)
java -XX:+UseG1GC -XX:+UseNUMA -Xmx16g -jar application.jar
```

```java
import java.lang.management.ManagementFactory;

public class NumaAwareApp {
    // JVM 레벨에서는 직접 NUMA를 제어하기 어렵지만,
    // 스레드 풀 설계 시 NUMA 토폴로지를 고려할 수 있다.

    private static final int NUMA_NODES = getNumaNodeCount();

    /**
     * NUMA 노드 수를 감지한다.
     * /sys/devices/system/node/online 파일을 파싱하거나
     * ManagementFactory의 OperatingSystemMXBean을 활용한다.
     */
    static int getNumaNodeCount() {
        try {
            String content = new String(
                java.nio.file.Files.readAllBytes(
                    java.nio.file.Paths.get("/sys/devices/system/node/online")
                )
            ).trim();
            // "0-3" 형식이면 4개 노드
            if (content.contains("-")) {
                String[] parts = content.split("-");
                return Integer.parseInt(parts[1]) - Integer.parseInt(parts[0]) + 1;
            }
            return 1;
        } catch (Exception e) {
            return 1;
        }
    }

    /**
     * NUMA 노드별 독립적인 스레드 풀을 구성한다.
     * 각 풀은 자신의 NUMA 노드 내 CPU 친화성을 갖고,
     * 해당 노드의 메모리를 우선 사용한다.
     */
    static java.util.concurrent.ExecutorService[] createNumaAwareThreadPools(
            int threadsPerNode) {
        java.util.concurrent.ExecutorService[] pools =
            new java.util.concurrent.ExecutorService[NUMA_NODES];
        for (int node = 0; node < NUMA_NODES; node++) {
            final int numaNode = node;
            pools[node] = java.util.concurrent.Executors.newFixedThreadPool(
                threadsPerNode,
                r -> {
                    Thread t = new Thread(r, "numa-node-" + numaNode + "-worker");
                    // 실제 CPU 친화성 설정은 JNI 또는 numactl 래핑으로 처리
                    return t;
                }
            );
        }
        return pools;
    }
}
```

### 메모리 접근 패턴 프로파일링

```bash
# perf로 NUMA 관련 이벤트 측정
perf stat -e numa-node-load-misses,numa-node-store-misses ./my_app

# Intel VTune에서 NUMA 분석 (상세)
# vtune -collect memory-access -result-dir ./results ./my_app

# /proc에서 NUMA 메모리 사용량 확인
cat /proc/buddyinfo  # 노드별 빈 메모리 블록
cat /proc/zoneinfo   # 노드/존별 상세 통계

# numastat으로 NUMA miss 확인
numastat
# 출력:
#                           node0           node1
# numa_hit               1234567          987654  ← 로컬 접근 성공
# numa_miss                12345           54321  ← 원격 접근 발생 (줄여야 함)
# numa_foreign              54321           12345
# interleave_hit             1234            1234
# local_node             1234567          987654
# other_node               12345           54321  ← 원격 노드에서 온 요청
```

## NUMA 설계 원칙 정리

### 원칙 1: 데이터와 연산을 같은 노드에

스레드가 처리할 데이터를 해당 스레드가 실행되는 NUMA 노드에서 할당하라. **first-touch 정책**을 활용하여 데이터를 처리할 스레드가 직접 초기화하도록 설계한다.

### 원칙 2: 워크로드 유형에 따른 메모리 정책 선택

- **지연 시간 민감(레이턴시 중심)**: `--membind`로 로컬 노드에 바인딩
- **처리량 중심(대역폭 집약적)**: `--interleave`로 모든 노드에 분산

### 원칙 3: 거짓 공유(False Sharing) 방지

서로 다른 NUMA 노드의 CPU들이 같은 캐시 라인을 공유하면 캐시 일관성 프로토콜이 노드 간 트래픽을 폭발적으로 증가시킨다. 자주 갱신되는 데이터는 노드별로 분리하고 캐시 라인 크기(보통 64 bytes)에 맞게 패딩하라.

```c
// 나쁜 패턴: 모든 코어가 같은 카운터를 경쟁
typedef struct {
    volatile long counter;  // 64 bytes 캐시 라인 전체를 오염
} SharedCounter;

// 좋은 패턴: NUMA 노드별 카운터 + 캐시 라인 패딩
typedef struct {
    volatile long counter;
    char padding[64 - sizeof(long)];  // 캐시 라인 격리
} __attribute__((aligned(64))) PerNodeCounter;

PerNodeCounter counters[MAX_NUMA_NODES];
```

### 원칙 4: 마이그레이션 비용을 고려

프로세스나 스레드가 NUMA 노드 간 이주(migration)하면 이미 할당된 메모리 페이지와 CPU가 분리된다. CPU 친화성(CPU affinity)을 설정하거나 `numactl --membind`로 메모리를 고정하면 이주를 제한할 수 있다.

NUMA는 "투명하게 동작하지만 성능은 투명하지 않다." 시스템의 하드웨어 토폴로지를 이해하고, 데이터 지역성을 의도적으로 설계하는 것이 고성능 멀티소켓 시스템 개발의 핵심이다.

## 참고 자료
- [Challenges of Memory Management on Modern NUMA Systems — ACM Queue](https://queue.acm.org/detail.cfm?id=2852078)
- [Expert Guide to NUMA Optimization in Linux — Medium](https://medium.com/@linuxgd/expert-guide-to-numa-optimization-in-linux-c188c5fb4d3d)
- [An Analysis of NUMA Architecture: From Hardware Topology to Linux Node Management — Medium](https://medium.com/@leohou1402/an-analysis-of-numa-architecture-from-hardware-topology-to-linux-node-management-a5945eb9880b)
- [Linux Beyond the Basics: NUMA — Medium](https://medium.com/@weidagang/linux-beyond-the-basics-numa-53b32ff60d6f)
