---
layout: post
title: "Futex 완전 정복: Linux 동기화 프리미티브의 숨겨진 심장"
date: 2026-08-05
categories: [cs, computer-science]
tags: [linux, futex, synchronization, kernel, mutex, pthread, concurrency, systems-programming]
---

여러분이 C++ `std::mutex`를 잠그거나 Java `synchronized` 블록에 진입할 때, 그 내부에서는 **futex(Fast Userspace Mutex)**라는 Linux 커널의 원시 동기화 인터페이스가 동작하고 있습니다. glibc의 pthread_mutex_t, pthread_cond_t, sem_t, Go 런타임의 mutex 등 현대 Linux 위에서 동작하는 거의 모든 고수준 동기화 프리미티브는 futex 위에 구축됩니다. 그러나 futex는 단순히 mutex의 하위 구현체가 아닙니다. 그 설계는 운영체제와 사용자 공간 사이의 경계를 최소화하는 정교한 철학을 담고 있습니다.

---

## 개념 설명

### 전통적인 동기화의 비효율

Futex 이전 시대(Linux 2.4 이전)에는 사용자 공간에서 동기화를 구현하는 방법이 두 가지밖에 없었습니다.

**방법 1: 완전 커널 기반 동기화**
매번 `semop()` 같은 시스템 콜을 호출해 커널이 락을 관리합니다. 항상 커널 진입이 발생하여 경쟁이 없는 경우에도 수백~수천 나노초의 오버헤드가 생깁니다.

**방법 2: 스핀락(Spinlock)**
사용자 공간에서 원자적 CAS 연산으로 바쁜 대기(busy-wait)를 수행합니다. CPU 자원을 낭비하며, 락을 오래 보유하는 경우 다른 스레드의 기아(starvation)를 유발합니다.

멀티스레드 애플리케이션에서 락의 대부분은 **경쟁 없이(uncontended)** 획득됩니다. 매번 커널을 호출하는 것은 거대한 낭비입니다.

### Futex의 핵심 통찰: 두 세계의 결합

Futex는 명확한 원칙을 갖습니다:
- **Fast path (경쟁 없음)**: 사용자 공간의 원자 연산만으로 처리. 커널 진입 없음.
- **Slow path (경쟁 있음)**: 커널의 wait queue에서 스레드를 재우고 깨움.

Futex는 두 컴포넌트로 구성됩니다:

1. **사용자 공간의 32비트 정수(futex word)**: 공유 메모리(mmap/스택/힙)에 위치하는 원자적 변수. 락 상태를 표현합니다.
2. **커널의 futex hash table**: 가상 주소를 키로 하는 wait queue의 해시 테이블. 대기 중인 스레드들을 관리합니다.

### Futex 시스템 콜 인터페이스

```c
#include <linux/futex.h>
#include <sys/syscall.h>

int futex(uint32_t *uaddr, int futex_op, uint32_t val,
          const struct timespec *timeout,
          uint32_t *uaddr2, uint32_t val3);
```

주요 연산:
- **FUTEX_WAIT**: `*uaddr == val`이면 스레드를 재움. 값이 이미 다르면 즉시 EAGAIN 반환.
- **FUTEX_WAKE**: `uaddr`에서 대기 중인 스레드 중 최대 val개를 깨움.
- **FUTEX_WAIT_BITSET / FUTEX_WAKE_BITSET**: 비트마스크로 선택적 wakeup (pthread_cond_t 구현에 사용).
- **FUTEX_REQUEUE**: 대기 스레드를 다른 futex로 이동 (pthread_cond_broadcast 최적화).

### 뮤텍스 상태 인코딩

실제 glibc의 pthread_mutex_t를 단순화한 구현에서 futex word는 세 가지 상태를 가집니다:
- **0**: 잠금 해제(unlocked)
- **1**: 잠금 획득, 대기자 없음
- **2**: 잠금 획득, 대기자 있음

이 세 상태 구분이 불필요한 FUTEX_WAKE 시스템 콜을 피하는 핵심입니다. 잠금 해제 시 상태가 1이면 (대기자 없음) FUTEX_WAKE를 호출하지 않아도 됩니다.

---

## 왜 필요한가

### 성능 측정: 시스템 콜 오버헤드의 현실

현대 x86-64 CPU에서 시스템 콜의 전환 비용은 다음과 같습니다:

| 연산 | 비용 |
|------|------|
| 사용자 공간 CAS (xchg) | ~5ns |
| 시스템 콜 오버헤드 | ~100~300ns |
| 컨텍스트 스위치 | ~1,000~10,000ns |

웹 서버처럼 초당 수백만 건의 mutex lock/unlock이 발생하는 환경에서 모든 경우에 시스템 콜을 호출한다면 성능은 치명적으로 저하됩니다.

실측: 경쟁 없는 `pthread_mutex_lock` + `pthread_mutex_unlock` 쌍의 실행 시간은 약 20~50ns로, 두 번의 시스템 콜(각 ~200ns)과 비교해 10배 이상 빠릅니다.

### Meltdown/Spectre 이후 증가한 시스템 콜 비용

2018년 Meltdown/Spectre 패치 이후 KPTI(Kernel Page Table Isolation)로 인해 시스템 콜 비용이 2~5배 증가했습니다. Futex의 fast path는 커널 진입이 없으므로 이 영향을 받지 않습니다.

---

## 실제 구현 예제

### 예제 1: Futex 시스템 콜을 직접 사용한 간단한 뮤텍스 구현 (C)

```c
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <unistd.h>
#include <syscall.h>
#include <linux/futex.h>
#include <stdatomic.h>
#include <pthread.h>
#include <assert.h>

/* futex 시스템 콜 래퍼 */
static int futex_wait(uint32_t *uaddr, uint32_t val) {
    return syscall(SYS_futex, uaddr, FUTEX_WAIT | FUTEX_PRIVATE_FLAG,
                   val, NULL, NULL, 0);
}

static int futex_wake(uint32_t *uaddr, uint32_t n) {
    return syscall(SYS_futex, uaddr, FUTEX_WAKE | FUTEX_PRIVATE_FLAG,
                   n, NULL, NULL, 0);
}

/*
 * 단순화된 뮤텍스
 * state == 0: 잠금 해제
 * state == 1: 잠금 획득, 대기자 없음
 * state == 2: 잠금 획득, 대기자 있음
 */
typedef struct {
    atomic_uint state;
} futex_mutex_t;

void futex_mutex_init(futex_mutex_t *m) {
    atomic_store(&m->state, 0);
}

void futex_mutex_lock(futex_mutex_t *m) {
    uint32_t expected = 0;

    /* Fast path: 0 -> 1 CAS 성공이면 즉시 락 획득 */
    if (atomic_compare_exchange_strong(&m->state, &expected, 1)) {
        return;
    }

    /* Slow path: 이미 잠긴 상태 */
    do {
        /* 대기자 있음을 표시하기 위해 2로 설정 */
        uint32_t old = atomic_exchange(&m->state, 2);

        /* 상태가 0이었다면 (락이 풀렸다면) 바로 진입 */
        if (old == 0) {
            return;
        }

        /* 커널에 상태가 2일 때만 재움 (spurious wakeup 방지) */
        futex_wait((uint32_t *)&m->state, 2);

        /* 깨어난 후 다시 시도 */
        expected = 0;
    } while (!atomic_compare_exchange_strong(&m->state, &expected, 2));
}

void futex_mutex_unlock(futex_mutex_t *m) {
    uint32_t old = atomic_fetch_sub(&m->state, 1);  /* 1 또는 2 -> 0 또는 1 */

    if (old == 2) {
        /* 대기자가 있었음: state를 0으로 만들고 하나를 깨움 */
        atomic_store(&m->state, 0);
        futex_wake((uint32_t *)&m->state, 1);
    }
    /* old == 1이면 대기자 없음: 이미 0이 됐으므로 WAKE 불필요 */
}

/* 검증용 공유 카운터 */
static futex_mutex_t g_mutex;
static volatile long g_counter = 0;
#define ITERATIONS 1000000
#define NUM_THREADS 4

void *worker(void *arg) {
    for (int i = 0; i < ITERATIONS; i++) {
        futex_mutex_lock(&g_mutex);
        g_counter++;
        futex_mutex_unlock(&g_mutex);
    }
    return NULL;
}

int main(void) {
    futex_mutex_init(&g_mutex);
    pthread_t threads[NUM_THREADS];

    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_create(&threads[i], NULL, worker, NULL);
    }
    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_join(threads[i], NULL);
    }

    long expected = (long)NUM_THREADS * ITERATIONS;
    assert(g_counter == expected);
    printf("카운터: %ld (기대값: %ld) - 정확성 검증 완료\n", g_counter, expected);
    return 0;
}
```

### 예제 2: Futex 기반 세마포어 구현 (C)

```c
#include <stdio.h>
#include <stdint.h>
#include <unistd.h>
#include <syscall.h>
#include <linux/futex.h>
#include <stdatomic.h>
#include <pthread.h>

typedef struct {
    atomic_int count;
} futex_semaphore_t;

void futex_sem_init(futex_semaphore_t *sem, int initial) {
    atomic_store(&sem->count, initial);
}

/* P 연산 (down/wait): count > 0이면 1 감소, 아니면 대기 */
void futex_sem_wait(futex_semaphore_t *sem) {
    while (1) {
        int c = atomic_load(&sem->count);
        if (c > 0) {
            if (atomic_compare_exchange_strong(&sem->count, &c, c - 1)) {
                return;  /* 성공적으로 획득 */
            }
            /* CAS 실패: 다른 스레드가 먼저 획득, 재시도 */
            continue;
        }
        /* count == 0: 커널에서 대기 */
        syscall(SYS_futex, &sem->count, FUTEX_WAIT | FUTEX_PRIVATE_FLAG,
                0, NULL, NULL, 0);
    }
}

/* V 연산 (up/post): count 증가 후 대기자 깨움 */
void futex_sem_post(futex_semaphore_t *sem) {
    int old = atomic_fetch_add(&sem->count, 1);
    if (old == 0) {
        /* count가 0이었다면 대기자가 있을 수 있으므로 깨움 */
        syscall(SYS_futex, &sem->count, FUTEX_WAKE | FUTEX_PRIVATE_FLAG,
                1, NULL, NULL, 0);
    }
}

/* 생산자-소비자 패턴 시연 */
#define BUFFER_SIZE 8
static int buffer[BUFFER_SIZE];
static int head = 0, tail = 0;
static futex_semaphore_t g_empty, g_full, g_mutex_sem;

void *producer(void *arg) {
    for (int i = 0; i < 20; i++) {
        futex_sem_wait(&g_empty);       /* 빈 슬롯 대기 */
        futex_sem_wait(&g_mutex_sem);   /* 버퍼 접근 보호 */

        buffer[tail] = i;
        tail = (tail + 1) % BUFFER_SIZE;
        printf("[생산자] %d 생산\n", i);

        futex_sem_post(&g_mutex_sem);
        futex_sem_post(&g_full);        /* 채운 슬롯 알림 */
        usleep(10000);
    }
    return NULL;
}

void *consumer(void *arg) {
    for (int i = 0; i < 20; i++) {
        futex_sem_wait(&g_full);        /* 채운 슬롯 대기 */
        futex_sem_wait(&g_mutex_sem);

        int val = buffer[head];
        head = (head + 1) % BUFFER_SIZE;
        printf("[소비자] %d 소비\n", val);

        futex_sem_post(&g_mutex_sem);
        futex_sem_post(&g_empty);       /* 빈 슬롯 알림 */
    }
    return NULL;
}

int main(void) {
    futex_sem_init(&g_empty, BUFFER_SIZE);  /* 처음엔 모두 비어있음 */
    futex_sem_init(&g_full, 0);
    futex_sem_init(&g_mutex_sem, 1);        /* 이진 세마포어 = 뮤텍스 */

    pthread_t prod_t, cons_t;
    pthread_create(&prod_t, NULL, producer, NULL);
    pthread_create(&cons_t, NULL, consumer, NULL);
    pthread_join(prod_t, NULL);
    pthread_join(cons_t, NULL);
    return 0;
}
```

---

## 주의사항 및 팁

### 허위 깨어남(Spurious Wakeup)

`FUTEX_WAIT`는 EINTR(시그널 인터럽트) 등으로 val과 *uaddr가 여전히 같아도 깨어날 수 있습니다. 반드시 루프로 감싸서 상태를 재확인해야 합니다:

```c
/* 잘못된 사용 */
futex_wait(addr, expected);  /* 한 번만 대기 - spurious wakeup에 취약 */
/* 올바른 사용 */
while (atomic_load(addr) == expected) {
    futex_wait(addr, expected);
}
```

### FUTEX_PRIVATE_FLAG 활용

같은 프로세스 내의 스레드 간에만 사용하는 futex는 `FUTEX_PRIVATE_FLAG`를 추가하면 커널이 물리 주소 변환 없이 가상 주소만으로 hash lookup을 수행해 약 10~15% 성능이 향상됩니다.

### Priority Inversion 문제

futex는 기본적으로 우선순위 역전(priority inversion) 문제에 취약합니다. 높은 우선순위 스레드가 낮은 우선순위 스레드가 잡은 락을 기다릴 때 문제가 발생합니다. Linux는 이를 위해 `FUTEX_LOCK_PI` (Priority Inheritance Futex)를 제공합니다. 실시간 시스템에서는 반드시 PI-futex를 사용해야 합니다.

### strace로 futex 동작 관찰

```bash
# pthread_mutex 내부의 futex 호출 관찰
strace -e trace=futex ./your_program 2>&1 | head -30

# 출력 예시:
# futex(0x..., FUTEX_WAIT_PRIVATE, 2, NULL) = 0
# futex(0x..., FUTEX_WAKE_PRIVATE, 1)       = 1
```

### 올바른 wait/wake 시퀀싱

Lock-Free 알고리즘에서 futex를 사용할 때 check-then-act 구간에서 TOCTOU(Time-of-Check-Time-of-Use) 경쟁이 발생할 수 있습니다. `FUTEX_WAIT`의 atomicity 보장 덕분에 `*uaddr`를 읽고 wait에 진입하는 동작이 원자적으로 처리됩니다. 커널은 진입 시 `*uaddr == val`을 재확인하여 이미 상태가 바뀐 경우 즉시 EAGAIN으로 반환합니다.

---

## 참고 자료
- [futex(7) - Linux manual page](https://man7.org/linux/man-pages/man7/futex.7.html)
- [futex(2) - Linux manual page](https://man7.org/linux/man-pages/man2/futex.2.html)
- [Futex - Wikipedia](https://en.wikipedia.org/wiki/Futex)
- [Basics of Futexes - Eli Bendersky](https://eli.thegreenplace.net/2018/basics-of-futexes/)
