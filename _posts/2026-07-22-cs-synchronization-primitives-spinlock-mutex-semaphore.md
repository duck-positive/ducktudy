---
layout: post
title: "동기화 프리미티브 심화: 스핀락, 뮤텍스, 세마포어, 컨디션 변수"
date: 2026-07-22
categories: [cs, computer-science]
tags: [spinlock, mutex, semaphore, condition-variable, concurrency, synchronization, os, linux-kernel]
---

멀티스레드 프로그래밍에서 가장 중요한 문제 중 하나는 **공유 자원에 대한 동시 접근을 제어**하는 것입니다. 잘못된 동기화는 데이터 경쟁(data race), 데드락(deadlock), 라이브락(livelock) 등의 심각한 버그를 초래합니다. 이 글에서는 스핀락, 뮤텍스, 세마포어, 컨디션 변수의 내부 동작 원리와 각각의 적합한 사용 시나리오를 깊이 있게 다룹니다.

## 개념 설명: 왜 동기화가 필요한가

두 스레드가 동시에 `counter++`를 실행하는 상황을 생각해봅시다.

```
Thread 1: LOAD counter → REG1 (값: 5)
Thread 2: LOAD counter → REG2 (값: 5)
Thread 1: REG1 = REG1 + 1 (값: 6)
Thread 2: REG2 = REG2 + 1 (값: 6)
Thread 1: STORE REG1 → counter (counter = 6)
Thread 2: STORE REG2 → counter (counter = 6)  ← 갱신 손실!
```

원래 기대값인 7이 아니라 6이 저장됩니다. 이런 **갱신 손실(lost update)** 문제를 해결하기 위해 임계 구역(critical section)을 보호하는 동기화 프리미티브가 필요합니다.

---

## 스핀락 (Spinlock)

### 개념

스핀락은 잠금이 해제될 때까지 **바쁜 대기(busy-wait)** 방식으로 CPU를 점유하며 계속 검사하는 락입니다. 커널 내부와 같이 컨텍스트 스위칭 비용이 락 대기 시간보다 클 때 적합합니다.

### 내부 구현: Test-and-Set

가장 단순한 스핀락은 `test_and_set` 원자 명령어로 구현됩니다.

```c
#include <stdatomic.h>
#include <stdio.h>
#include <pthread.h>

typedef struct {
    atomic_flag flag;
} spinlock_t;

void spinlock_init(spinlock_t *lock) {
    atomic_flag_clear(&lock->flag);
}

void spinlock_lock(spinlock_t *lock) {
    /* atomic_flag_test_and_set이 false를 반환할 때까지 스핀 */
    while (atomic_flag_test_and_set_explicit(&lock->flag, memory_order_acquire)) {
        /* CPU 힌트: 다른 하이퍼스레드에 양보 */
        __asm__ volatile("pause" ::: "memory");
    }
}

void spinlock_unlock(spinlock_t *lock) {
    atomic_flag_clear_explicit(&lock->flag, memory_order_release);
}

/* 사용 예시 */
static spinlock_t counter_lock;
static long shared_counter = 0;

void *increment_worker(void *arg) {
    for (int i = 0; i < 1000000; i++) {
        spinlock_lock(&counter_lock);
        shared_counter++;
        spinlock_unlock(&counter_lock);
    }
    return NULL;
}

int main(void) {
    spinlock_init(&counter_lock);

    pthread_t t1, t2;
    pthread_create(&t1, NULL, increment_worker, NULL);
    pthread_create(&t2, NULL, increment_worker, NULL);
    pthread_join(t1, NULL);
    pthread_join(t2, NULL);

    printf("Expected: 2000000, Got: %ld\n", shared_counter);
    return 0;
}
```

위 코드에서 `memory_order_acquire`와 `memory_order_release`는 메모리 배리어를 삽입하여 컴파일러/CPU 재순서를 방지합니다. `pause` 명령어는 Intel CPU에서 스핀 루프임을 프로세서에 알려 하이퍼스레딩 효율을 높이고 메모리 버스 점유를 줄입니다.

### 리눅스 커널의 스핀락

리눅스 커널에서는 `raw_spinlock_t`를 사용하며, PREEMPT_RT 패치 적용 여부에 따라 동작이 달라집니다. 실시간 커널에서는 우선순위 역전을 막기 위해 스핀락이 내부적으로 뮤텍스로 전환되기도 합니다.

---

## 뮤텍스 (Mutex)

### 개념

뮤텍스(Mutual Exclusion)는 잠금 획득에 실패한 스레드를 **sleep 상태로 전환**시켜 CPU를 다른 작업에 양보하는 슬리핑 락입니다. 대기 시간이 길거나 사용자 공간 프로그램에서 주로 사용합니다.

### 내부 구현 원리

Linux의 futex(Fast Userspace muTEX)를 활용한 뮤텍스의 핵심은 **빠른 경로는 유저 공간, 경합 발생 시에만 시스템 콜**을 호출하는 구조입니다.

```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

/*
 * pthreads의 pthread_mutex는 내부적으로 futex를 사용합니다.
 * 아래는 futex를 직접 활용한 간략한 뮤텍스 구현입니다.
 */
#include <linux/futex.h>
#include <sys/syscall.h>
#include <unistd.h>
#include <stdatomic.h>

typedef struct {
    atomic_int state;  /* 0: 잠금 해제, 1: 잠금, 2: 경합 중 */
} futex_mutex_t;

static long futex_wait(atomic_int *uaddr, int val) {
    return syscall(SYS_futex, uaddr, FUTEX_WAIT_PRIVATE, val, NULL, NULL, 0);
}

static long futex_wake(atomic_int *uaddr, int count) {
    return syscall(SYS_futex, uaddr, FUTEX_WAKE_PRIVATE, count, NULL, NULL, 0);
}

void futex_mutex_lock(futex_mutex_t *m) {
    int c = 0;
    /* 빠른 경로: 잠금이 해제된 상태면 즉시 획득 */
    if (atomic_compare_exchange_strong_explicit(
            &m->state, &c, 1,
            memory_order_acquire, memory_order_relaxed)) {
        return;
    }
    /* 느린 경로: 경합 상태로 전환하고 커널에 대기 요청 */
    do {
        if (c == 2 ||
            atomic_compare_exchange_strong_explicit(
                &m->state, &c, 2,
                memory_order_acquire, memory_order_relaxed)) {
            futex_wait(&m->state, 2);
        }
        c = 0;
    } while (!atomic_compare_exchange_strong_explicit(
                 &m->state, &c, 2,
                 memory_order_acquire, memory_order_relaxed));
}

void futex_mutex_unlock(futex_mutex_t *m) {
    /* 경합 중인 스레드가 있으면 하나를 깨움 */
    if (atomic_fetch_sub_explicit(&m->state, 1, memory_order_release) != 1) {
        atomic_store_explicit(&m->state, 0, memory_order_release);
        futex_wake(&m->state, 1);
    }
}
```

이 구현의 핵심은 **경합이 없을 때 시스템 콜 없이 유저 공간에서 모든 처리를 완료**한다는 것입니다. 실제 `glibc`의 `pthread_mutex_t`도 유사한 구조로 동작합니다.

---

## 세마포어 (Semaphore)

### 개념

세마포어는 **정수 카운터**를 기반으로 동작하며, 이진 세마포어(카운터=1)와 카운팅 세마포어(카운터>1)로 구분됩니다. 뮤텍스와 달리 소유권 개념이 없어 한 스레드가 잠그고 다른 스레드가 해제할 수 있습니다.

```c
#include <semaphore.h>
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

#define BUFFER_SIZE 5
#define NUM_ITEMS   20

int buffer[BUFFER_SIZE];
int in_idx = 0, out_idx = 0;

sem_t empty;   /* 빈 슬롯 개수 */
sem_t full;    /* 채워진 슬롯 개수 */
sem_t mutex;   /* 버퍼 접근 상호 배제 */

void *producer(void *arg) {
    for (int i = 0; i < NUM_ITEMS; i++) {
        sem_wait(&empty);   /* 빈 슬롯을 기다림 */
        sem_wait(&mutex);

        buffer[in_idx] = i;
        in_idx = (in_idx + 1) % BUFFER_SIZE;
        printf("[Producer] produced: %d\n", i);

        sem_post(&mutex);
        sem_post(&full);    /* 채워진 슬롯 증가 */
    }
    return NULL;
}

void *consumer(void *arg) {
    for (int i = 0; i < NUM_ITEMS; i++) {
        sem_wait(&full);    /* 채워진 슬롯을 기다림 */
        sem_wait(&mutex);

        int item = buffer[out_idx];
        out_idx = (out_idx + 1) % BUFFER_SIZE;
        printf("[Consumer] consumed: %d\n", item);

        sem_post(&mutex);
        sem_post(&empty);   /* 빈 슬롯 증가 */
    }
    return NULL;
}

int main(void) {
    sem_init(&empty, 0, BUFFER_SIZE);
    sem_init(&full,  0, 0);
    sem_init(&mutex, 0, 1);

    pthread_t prod, cons;
    pthread_create(&prod, NULL, producer, NULL);
    pthread_create(&cons, NULL, consumer, NULL);
    pthread_join(prod, NULL);
    pthread_join(cons, NULL);

    sem_destroy(&empty);
    sem_destroy(&full);
    sem_destroy(&mutex);
    return 0;
}
```

이 코드는 **생산자-소비자 문제(Producer-Consumer Problem)**를 세마포어로 해결하는 고전적인 예제입니다. `empty`와 `full` 세마포어가 버퍼 슬롯 가용성을 나타내며, `mutex` 세마포어가 버퍼에 대한 상호 배제를 보장합니다.

---

## 컨디션 변수 (Condition Variable)

컨디션 변수는 특정 조건이 충족될 때까지 스레드를 효율적으로 대기시키는 프리미티브입니다. 반드시 뮤텍스와 함께 사용해야 합니다.

```c
#include <pthread.h>
#include <stdio.h>
#include <stdbool.h>

pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t  cond = PTHREAD_COND_INITIALIZER;
bool data_ready = false;

void *waiter_thread(void *arg) {
    pthread_mutex_lock(&lock);
    /* spurious wakeup을 방지하기 위해 while로 조건 재검사 */
    while (!data_ready) {
        pthread_cond_wait(&cond, &lock);  /* 원자적으로 잠금 해제 + 대기 */
    }
    printf("[Waiter] Data is ready, processing...\n");
    pthread_mutex_unlock(&lock);
    return NULL;
}

void *notifier_thread(void *arg) {
    /* 데이터 준비 작업 시뮬레이션 */
    printf("[Notifier] Preparing data...\n");
    pthread_mutex_lock(&lock);
    data_ready = true;
    pthread_cond_signal(&cond);  /* 대기 중인 스레드 하나를 깨움 */
    pthread_mutex_unlock(&lock);
    return NULL;
}
```

`pthread_cond_wait`은 내부적으로 뮤텍스 해제와 대기를 **원자적으로** 수행하기 때문에 경쟁 조건 없이 안전하게 조건을 기다릴 수 있습니다.

---

## 주의사항 및 팁

### 1. 어떤 프리미티브를 선택할까?

| 상황 | 추천 |
|------|------|
| 임계 구역이 매우 짧고 커널 내부 코드 | **스핀락** |
| 대기 시간이 길거나 유저 공간 코드 | **뮤텍스** |
| 소유권 없이 신호 전달이 필요한 경우 | **세마포어** |
| 특정 조건이 만족될 때까지 대기 | **컨디션 변수** |

### 2. Priority Inversion (우선순위 역전)

낮은 우선순위 스레드가 뮤텍스를 보유한 상태에서 높은 우선순위 스레드가 대기하면 문제가 발생합니다. Linux의 `PTHREAD_PRIO_INHERIT` 프로토콜이나 실시간 OS의 Priority Inheritance 메커니즘으로 해결합니다.

```c
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_setprotocol(&attr, PTHREAD_PRIO_INHERIT);
pthread_mutex_init(&my_mutex, &attr);
```

### 3. Double-Checked Locking 안티패턴

```c
/* 잘못된 예: 메모리 모델 고려 없이 double-checked locking 사용 */
if (instance == NULL) {         /* 첫 번째 검사 */
    pthread_mutex_lock(&lock);
    if (instance == NULL) {     /* 두 번째 검사 */
        instance = create();    /* CPU 재순서로 인해 불완전한 객체 노출 가능 */
    }
    pthread_mutex_unlock(&lock);
}

/* 올바른 예: atomic으로 안전하게 처리 */
void *get_instance(void) {
    void *p = atomic_load_explicit(&instance, memory_order_acquire);
    if (!p) {
        pthread_mutex_lock(&lock);
        p = atomic_load_explicit(&instance, memory_order_relaxed);
        if (!p) {
            p = create();
            atomic_store_explicit(&instance, p, memory_order_release);
        }
        pthread_mutex_unlock(&lock);
    }
    return p;
}
```

### 4. 락 경합을 줄이는 전략

- **Lock Striping**: 단일 락 대신 여러 락으로 분할 (예: `ConcurrentHashMap`의 버킷별 락)
- **Read-Write Lock**: 읽기는 병렬, 쓰기만 독점으로 처리
- **Lock-Free 알고리즘**: CAS 기반으로 락 없이 구현

동기화 프리미티브는 멀티코어 시대에 올바른 병렬 프로그램을 작성하기 위한 필수 도구입니다. 각 프리미티브의 내부 동작을 이해하면 성능과 정확성을 모두 갖춘 코드를 작성할 수 있습니다.

## 참고 자료
- [Linux Kernel Mutex Design — The Linux Kernel documentation](https://docs.kernel.org/locking/mutex-design.html)
- [Linux Kernel Synchronization Primitives Part 4 — linux-insides](https://0xax.gitbooks.io/linux-insides/content/SyncPrim/linux-sync-4.html)
- [Real-time Ubuntu: Kernel Locks Explained](https://documentation.ubuntu.com/real-time/latest/explanation/locks/)
- [Mutex vs Semaphore Internal Implementation — Medium](https://medium.com/@lakshminath_alamuru/mutex-vs-semaphore-and-its-internal-implementation-1500a9d58242)
