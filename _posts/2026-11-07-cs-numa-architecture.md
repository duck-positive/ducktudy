---
layout: post
title: "NUMA 아키텍처 완전 정복: 멀티소켓 서버에서 메모리 지역성이 성능을 결정하는 방법"
date: 2026-11-07
categories: [cs, computer-science]
tags: [numa, non-uniform-memory-access, linux, cpu, performance, memory-optimization, hwloc]
---

## NUMA란 무엇인가?

NUMA(Non-Uniform Memory Access, 비균일 메모리 접근)는 현대 멀티소켓 서버 시스템에서 사용하는 메모리 아키텍처입니다. NUMA에서는 각 CPU 소켓(프로세서)이 자신에게 직접 연결된 로컬 메모리(local memory)와 다른 소켓에 연결된 원격 메모리(remote memory)에 서로 다른 지연 시간으로 접근합니다.

단일 소켓 시스템이나 2세대 이전 서버에서는 모든 CPU 코어가 단일 메모리 버스를 통해 같은 속도로 메모리에 접근하는 UMA(Uniform Memory Access) 방식을 사용했습니다. 그러나 CPU 코어 수가 늘어날수록 단일 버스는 병목이 되고, 이를 해결하기 위해 각 소켓마다 메모리 컨트롤러를 두는 NUMA 아키텍처가 등장했습니다.

전형적인 2소켓 NUMA 서버에서 접근 지연 시간 차이는 다음과 같습니다:
- **로컬 메모리 접근:** 약 60~80 ns
- **원격 메모리 접근 (1홉):** 약 100~150 ns (약 1.5~2배)
- **원격 메모리 접근 (2홉, 4소켓 서버):** 약 180~250 ns (약 2.5~3배)

이 차이는 소규모 애플리케이션에서는 무시할 수 있지만, 메모리 대역폭 집약적인 데이터베이스, 인-메모리 캐시, 과학 계산 워크로드에서는 전체 성능에 30~50%의 영향을 미칠 수 있습니다.

## NUMA 하드웨어 구조

현대 Intel Xeon이나 AMD EPYC 서버는 여러 소켓을 고속 인터커넥트로 연결합니다:

- **Intel:** UPI(Ultra Path Interconnect), 이전에는 QPI(QuickPath Interconnect)
- **AMD:** Infinity Fabric — EPYC "Rome"부터 소켓 내에서도 여러 CCD(Core Complex Die)가 별도 NUMA 도메인을 형성하는 "sub-NUMA" 구조 도입

Linux 커널은 이 NUMA 토폴로지를 `numactl`, `/proc/cpuinfo`, `/sys/devices/system/node/` 를 통해 노출합니다.

```bash
# NUMA 토폴로지 확인
numactl --hardware

# 출력 예시 (2소켓 서버):
# available: 2 nodes (0-1)
# node 0 cpus: 0 1 2 3 4 5 6 7 8 9 10 11 24 25 26 27 28 29 30 31 32 33 34 35
# node 0 size: 128720 MB
# node 0 free: 95431 MB
# node 1 cpus: 12 13 14 15 16 17 18 19 20 21 22 23 36 37 38 39 40 41 42 43 44 45 46 47
# node 1 size: 128760 MB
# node 1 free: 102318 MB
# node distances:
# node   0   1
#   0:  10  21    ← node 0에서 node 0 접근 비용: 10 (기준), node 1 접근: 21
#   1:  21  10    ← 비대칭이면 NUMA 간 접근 비용 차이가 있음을 의미
```

## 왜 NUMA가 성능에 중요한가?

NUMA 비인식(NUMA-unaware) 애플리케이션이 겪는 대표적인 성능 문제:

1. **원격 메모리 접근:** 스레드가 다른 소켓의 메모리에 반복 접근하면 지연 시간이 2배
2. **메모리 대역폭 경합:** 모든 스레드가 한 소켓의 메모리 컨트롤러를 집중 사용하면 대역폭 포화
3. **캐시 무효화 비용:** 소켓 간 캐시 일관성 유지를 위한 MESI 프로토콜 메시지가 UPI/Infinity Fabric을 통해 전달되어 추가 지연 발생

## 실제 구현: NUMA 인식 프로그래밍

### C/C++에서 libnuma 활용

```c
#include <stdio.h>
#include <stdlib.h>
#include <numa.h>
#include <numaif.h>
#include <pthread.h>
#include <string.h>
#include <time.h>

#define BUFFER_SIZE (256 * 1024 * 1024)  // 256MB
#define NUM_THREADS 4
#define ITERATIONS  100

typedef struct {
    int thread_id;
    int preferred_node;
    char *buffer;
    size_t size;
    double elapsed_ms;
} ThreadArg;

static void *memory_access_benchmark(void *arg) {
    ThreadArg *targ = (ThreadArg *)arg;
    volatile long sum = 0;
    struct timespec start, end;

    clock_gettime(CLOCK_MONOTONIC, &start);

    for (int iter = 0; iter < ITERATIONS; iter++) {
        for (size_t i = 0; i < targ->size / sizeof(long); i++) {
            sum += ((long *)targ->buffer)[i];
        }
    }

    clock_gettime(CLOCK_MONOTONIC, &end);
    targ->elapsed_ms = (end.tv_sec - start.tv_sec) * 1000.0
                     + (end.tv_nsec - start.tv_nsec) / 1e6;

    printf("Thread %d (NUMA node %d): sum=%ld, elapsed=%.2f ms\n",
           targ->thread_id, targ->preferred_node, sum, targ->elapsed_ms);
    return NULL;
}

int main(void) {
    if (numa_available() < 0) {
        fprintf(stderr, "NUMA not available\n");
        return 1;
    }

    int num_nodes = numa_num_configured_nodes();
    printf("NUMA 노드 수: %d\n", num_nodes);

    // === 테스트 1: NUMA 비인식 할당 (일반 malloc) ===
    printf("\n[테스트 1] 일반 malloc (NUMA 비인식)\n");
    char *bad_buf = malloc(BUFFER_SIZE);
    memset(bad_buf, 1, BUFFER_SIZE);  // First-touch: 할당 시 소켓 결정
    // 문제: 메인 스레드가 실행된 소켓에 메모리가 배치되지만,
    // 다른 소켓의 스레드들은 원격 접근을 하게 됨

    // === 테스트 2: NUMA 인식 할당 (numa_alloc_onnode) ===
    printf("\n[테스트 2] numa_alloc_onnode (NUMA 인식)\n");

    ThreadArg args[NUM_THREADS];
    pthread_t threads[NUM_THREADS];

    for (int i = 0; i < NUM_THREADS; i++) {
        int node = i % num_nodes;  // 각 스레드를 다른 NUMA 노드에 배치

        // 해당 NUMA 노드에 직접 메모리 할당
        char *buf = numa_alloc_onnode(BUFFER_SIZE / NUM_THREADS, node);
        if (!buf) {
            perror("numa_alloc_onnode");
            return 1;
        }
        memset(buf, 1, BUFFER_SIZE / NUM_THREADS);

        args[i].thread_id = i;
        args[i].preferred_node = node;
        args[i].buffer = buf;
        args[i].size = BUFFER_SIZE / NUM_THREADS;
    }

    // 각 스레드를 해당 NUMA 노드의 CPU에 바인딩
    for (int i = 0; i < NUM_THREADS; i++) {
        int node = args[i].preferred_node;
        cpu_set_t cpuset;
        CPU_ZERO(&cpuset);

        // node에 속한 CPU들을 cpuset에 추가
        struct bitmask *node_cpus = numa_allocate_cpumask();
        numa_node_to_cpus(node, node_cpus);
        for (int cpu = 0; cpu < numa_num_configured_cpus(); cpu++) {
            if (numa_bitmask_isbitset(node_cpus, cpu)) {
                CPU_SET(cpu, &cpuset);
            }
        }
        numa_free_cpumask(node_cpus);

        pthread_create(&threads[i], NULL, memory_access_benchmark, &args[i]);

        // 스레드를 특정 CPU 집합에 바인딩 (CPU affinity)
        pthread_setaffinity_np(threads[i], sizeof(cpu_set_t), &cpuset);
    }

    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_join(threads[i], NULL);
        numa_free(args[i].buffer, BUFFER_SIZE / NUM_THREADS);
    }

    free(bad_buf);
    return 0;
}

// 컴파일: gcc -O2 -o numa_bench numa_bench.c -lnuma -lpthread
```

### Java에서 NUMA 최적화: JVM 설정과 Unsafe API

```java
import sun.misc.Unsafe;
import java.lang.reflect.Field;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicLong;

/**
 * NUMA 인식 Java 프로그래밍 예제.
 * JVM은 기본적으로 NUMA를 인식하지 못하지만,
 * JVM 플래그와 Unsafe를 통해 최적화할 수 있다.
 */
public class NumaAwareDemo {

    // JVM NUMA 최적화 플래그 (JVM 시작 시 적용):
    // -XX:+UseNUMA              : NUMA 인식 메모리 할당 활성화 (Eden 영역을 노드별 분할)
    // -XX:+UseParallelGC        : NUMA와 가장 잘 호환되는 GC
    // -XX:+UseNUMAInterleaving  : 대용량 힙을 NUMA 노드 간 인터리빙 (균등 분산)
    // -XX:NUMAChunkSize=4m      : 노드별 힙 청크 크기

    private static final int BUFFER_SIZE = 64 * 1024 * 1024;  // 64MB
    private static final Unsafe UNSAFE;

    static {
        try {
            Field f = Unsafe.class.getDeclaredField("theUnsafe");
            f.setAccessible(true);
            UNSAFE = (Unsafe) f.get(null);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

    /**
     * Off-heap 메모리를 직접 할당하여 GC 압력 없이 대규모 버퍼를 관리.
     * OS의 NUMA 정책을 직접 따르므로, numactl로 JVM 프로세스를 감싸면
     * off-heap 메모리도 NUMA 노드에 바인딩됨.
     */
    static class NumaBuffer implements AutoCloseable {
        private final long address;
        private final long size;

        NumaBuffer(long size) {
            this.size = size;
            this.address = UNSAFE.allocateMemory(size);
            UNSAFE.setMemory(address, size, (byte) 0);
        }

        long get(long index) {
            return UNSAFE.getLong(address + index * Long.BYTES);
        }

        void set(long index, long value) {
            UNSAFE.putLong(address + index * Long.BYTES, value);
        }

        @Override
        public void close() {
            UNSAFE.freeMemory(address);
        }
    }

    /**
     * NUMA 토폴로지를 고려한 스레드 풀 설계.
     * 각 NUMA 노드마다 별도의 스레드 풀을 두고,
     * 작업을 해당 노드의 데이터와 가까운 풀에 제출한다.
     */
    static class NumaAwareThreadPool {
        private final ExecutorService[] nodePools;
        private final int numNodes;
        private final AtomicLong counter = new AtomicLong(0);

        NumaAwareThreadPool(int numNodes, int threadsPerNode) {
            this.numNodes = numNodes;
            this.nodePools = new ExecutorService[numNodes];
            for (int i = 0; i < numNodes; i++) {
                final int nodeId = i;
                nodePools[i] = Executors.newFixedThreadPool(
                    threadsPerNode,
                    r -> {
                        Thread t = new Thread(r, "numa-node-" + nodeId + "-worker-" + counter.incrementAndGet());
                        // 실제 환경에서는 JNI를 통해 pthread_setaffinity_np 호출
                        return t;
                    }
                );
            }
        }

        /**
         * 데이터의 NUMA 노드에 가장 가까운 풀에 작업 제출.
         * 간단한 라운드로빈 대신, 실제로는 데이터 주소의 NUMA 노드를
         * mbind/get_mempolicy 시스템 콜로 조회하여 결정.
         */
        Future<?> submit(Runnable task, int dataNode) {
            return nodePools[dataNode % numNodes].submit(task);
        }

        void shutdown() throws InterruptedException {
            for (ExecutorService pool : nodePools) {
                pool.shutdown();
                pool.awaitTermination(5, TimeUnit.SECONDS);
            }
        }
    }

    public static void main(String[] args) throws Exception {
        System.out.println("NUMA 인식 Java 데모");
        System.out.println("JVM 플래그: -XX:+UseNUMA -XX:+UseParallelGC");

        int numNodes = 2;  // 2소켓 서버 가정
        NumaAwareThreadPool pool = new NumaAwareThreadPool(numNodes, 4);

        // Off-heap 버퍼 할당 (각 노드에 하나씩)
        try (NumaBuffer node0Buffer = new NumaBuffer(BUFFER_SIZE);
             NumaBuffer node1Buffer = new NumaBuffer(BUFFER_SIZE)) {

            // 각 버퍼에 대한 작업을 해당 노드의 풀에 제출
            Future<?> f0 = pool.submit(() -> {
                long sum = 0;
                for (long i = 0; i < BUFFER_SIZE / Long.BYTES; i++) {
                    sum += node0Buffer.get(i);
                }
                System.out.println("Node 0 처리 완료, sum=" + sum);
            }, 0);

            Future<?> f1 = pool.submit(() -> {
                long sum = 0;
                for (long i = 0; i < BUFFER_SIZE / Long.BYTES; i++) {
                    sum += node1Buffer.get(i);
                }
                System.out.println("Node 1 처리 완료, sum=" + sum);
            }, 1);

            f0.get();
            f1.get();
        }

        pool.shutdown();
    }
}
```

## Linux NUMA 정책과 커널 메모리 할당

Linux 커널은 `mmap` 시스템 콜과 페이지 폴트 처리 시 **First-Touch 정책**을 기본으로 사용합니다. 메모리를 할당하는 시점이 아닌, 해당 페이지에 처음 접근하는 시점에 그 CPU가 위치한 NUMA 노드에 물리 페이지를 배치합니다.

```bash
# numactl: 프로세스를 특정 NUMA 노드에 바인딩
numactl --cpunodebind=0 --membind=0 ./my_app      # CPU와 메모리를 모두 node 0에 바인딩
numactl --interleave=all ./my_app                   # 메모리를 모든 노드에 균등 분산 (대역폭 최대화)
numactl --preferred=0 ./my_app                      # node 0 선호, 부족하면 다른 노드 사용

# 실행 중인 프로세스의 NUMA 메모리 사용 확인
numastat -p <PID>

# 시스템 전체 NUMA 통계
numastat

# NUMA 발란싱 상태 확인 (Linux NUMA Balancing: 자동 페이지 마이그레이션)
cat /proc/sys/kernel/numa_balancing
# 1이면 활성화: 커널이 원격 접근 패턴을 감지하고 페이지를 자동으로 로컬 노드로 이동
```

## NUMA와 데이터베이스 성능

PostgreSQL, MySQL, Redis 같은 데이터베이스는 NUMA의 영향을 크게 받습니다.

```bash
# PostgreSQL NUMA 최적화
# 1. 인터리빙 모드로 실행 (균등한 메모리 대역폭)
numactl --interleave=all postgres -D /var/lib/postgresql/data

# 2. 또는 특정 소켓에 바인딩 (대용량 버퍼 풀이 한 노드에 맞는 경우)
numactl --cpunodebind=0 --membind=0 postgres -D /var/lib/postgresql/data

# Redis NUMA 최적화 (단일 스레드 특성상 단일 노드 바인딩이 유리)
numactl --cpunodebind=0 --membind=0 redis-server /etc/redis/redis.conf

# Java 힙 NUMA 최적화 (G1GC는 UseNUMA와 호환성 문제가 있으므로 ParallelGC 권장)
java -XX:+UseNUMA -XX:+UseParallelGC -Xmx128g -jar app.jar
```

## 주의사항과 운영 팁

**1. NUMA 발란싱(Automatic NUMA Balancing)의 양면성:** Linux의 자동 NUMA 발란싱(`/proc/sys/kernel/numa_balancing=1`)은 편리하지만, 대용량 데이터를 다루는 데이터베이스나 HPC 워크로드에서 페이지 마이그레이션 오버헤드가 성능을 오히려 저하시킬 수 있습니다. `numastat`로 `numa_miss`와 `numa_foreign` 카운터를 모니터링하세요.

**2. Transparent Huge Pages (THP):** NUMA와 THP를 함께 사용하면 huge page 할당이 특정 NUMA 노드에 편중될 수 있습니다. Redis 같은 애플리케이션은 THP를 비활성화하도록 권장합니다.

**3. AMD EPYC의 Sub-NUMA:** AMD EPYC "Milan" (3세대)부터 단일 소켓 내에서도 여러 CCD(Core Complex Die)가 별도 NUMA 도메인을 형성합니다. 이른바 Sub-NUMA Clustering(SNC)으로, `numactl --hardware`에서 4소켓 처럼 보이는 2소켓 서버가 될 수 있습니다. 이 토폴로지를 `hwloc-lstopo`로 시각화해 확인하세요.

**4. 메모리 오버커밋 금지:** NUMA 노드당 메모리가 균등하지 않은 경우, 한 노드의 메모리를 소진하면 커널이 자동으로 원격 노드 메모리를 사용합니다. 이를 방지하려면 `numactl --membind`로 엄격하게 제한하세요.

**5. 모니터링 도구:** `perf stat -e cache-misses,cache-references,node-load-misses,node-store-misses` 명령으로 NUMA 관련 HW 카운터를 측정하고, `lstopo`(hwloc)로 NUMA 토폴로지를 시각화하세요.

## 참고 자료
- [Linux NUMA Support (kernel.org OLS 2003)](https://www.kernel.org/doc/ols/2003/ols2003-pages-169-184.pdf)
- [numa(7) Man Page - Linux Manual](https://man.archlinux.org/man/numa.7.en.txt)
- [An Overview of Non-Uniform Memory Access - ACM Communications](https://cacm.acm.org/magazines/2013/9/167139-an-overview-of-non-uniform-memory-access/fulltext)
- [hwloc: Portable Hardware Locality](https://www.open-mpi.org/projects/hwloc/)
