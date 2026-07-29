---
layout: post
title: "RCU (Read-Copy-Update) — 리눅스 커널의 고성능 동기화 메커니즘"
date: 2026-07-29
categories: [cs, computer-science]
tags: [rcu, linux-kernel, synchronization, concurrency, lock-free, read-copy-update]
---

## 개념 설명

RCU(Read-Copy-Update)는 읽기 작업이 쓰기보다 훨씬 빈번한 상황에서 뛰어난 성능을 발휘하는 독특한 동기화 메커니즘이다. 리눅스 커널에서 광범위하게 사용되며, 멀티코어 환경에서 확장성을 극대화하는 데 핵심적인 역할을 한다.

전통적인 Reader-Writer Lock(RWLock)은 다수의 독자가 동시에 읽을 수 있지만, 독자들도 락을 획득해야 한다는 점에서 한계가 있다. 반면 RCU는 읽기 측에서 어떠한 락도, 원자적 연산도, 메모리 배리어도 필요하지 않다(대부분의 아키텍처에서). 이는 읽기 작업의 오버헤드를 사실상 0에 가깝게 만든다.

RCU의 명칭은 업데이트 메커니즘을 그대로 반영한다:

1. **Read**: 독자는 언제든 자유롭게 데이터를 읽는다. 락 없이, 원자적 연산 없이.
2. **Copy**: 업데이터는 수정할 데이터의 복사본을 만든다.
3. **Update**: 복사본을 수정한 뒤, 원자적 포인터 교체로 새 버전을 발행한다.

발행 이후에는 구 버전을 즉시 해제할 수 없다. 발행 이전부터 읽기를 시작한 독자들이 여전히 구 버전을 참조하고 있을 수 있기 때문이다. 이 때 **그레이스 피리어드(Grace Period)** 개념이 등장한다.

### 그레이스 피리어드와 정지 상태

**정지 상태(Quiescent State)**란 CPU가 RCU 읽기 측 임계 구역 밖에 있는 상태를 말한다. 컨텍스트 스위칭, 유저 모드 실행, 유휴 상태 등이 정지 상태의 예다.

**그레이스 피리어드**는 모든 온라인 CPU가 최소 한 번씩 정지 상태를 거친 후에 완료된다. 이 시점이 되면 업데이트 이전부터 읽기를 시작했던 모든 독자가 완료되었음이 보장되므로, 구 버전의 메모리를 안전하게 해제할 수 있다.

```
시간 ──────────────────────────────────────────────>

         [그레이스 피리어드 시작]
         ↓
Reader A: |__읽기중__|
Reader B:       |__읽기중__|
Reader C:             |__읽기중__|

         그레이스 피리어드 종료: Reader A, B, C 모두 완료
                                 ↓
                              구 버전 해제 가능
```

## 왜 필요한가?

### RWLock의 스케일링 문제

멀티코어 시스템에서 RWLock의 문제점:

- **캐시 라인 더럽힘**: 독자가 락을 획득할 때 카운터 업데이트로 캐시 라인이 더럽혀진다. 코어 수가 늘어날수록 캐시 일관성 트래픽이 폭발적으로 증가한다.
- **락 경쟁 증가**: 코어 수에 비례해 락 획득 비용이 증가한다.
- **실시간 지연 예측 불가**: 쓰기 측이 독자들을 기다리는 동안 지연이 발생한다.

### RCU의 성능 이점

리눅스 커널 벤치마크에 따르면, 읽기 작업에 한해 RCU는 RWLock 대비:
- 단일 코어: 약 2~3배 빠름
- 48코어: 약 100배 이상 빠름

이 차이가 코어 수에 따라 폭발적으로 커지는 이유가 바로 캐시 라인 경합의 제거다.

### 리눅스 커널에서의 활용

RCU는 리눅스 커널 전반에 걸쳐 사용된다:
- **네트워크 라우팅 테이블**: 패킷이 들어올 때마다 조회가 발생하는 핫 경로
- **프로세스 자격증명(credentials)**: setuid 등 업데이트가 드문 데이터
- **VFS 덴트리 캐시**: 파일 경로 조회에 사용
- **모듈 목록**: 커널 모듈 로드/언로드

## 실제 구현 예제

### 예제 1: RCU를 사용한 연결 리스트 순회

```c
#include <linux/rcupdate.h>
#include <linux/rculist.h>
#include <linux/slab.h>
#include <linux/spinlock.h>

struct my_data {
    int value;
    struct list_head list;
    struct rcu_head rcu;   /* RCU 콜백 등록용 */
};

static LIST_HEAD(my_list);       /* RCU로 보호되는 전역 리스트 */
static DEFINE_SPINLOCK(list_lock); /* 업데이터 간 상호 배제 */

/* 읽기 측: 락 없이 순회 */
void reader_function(void)
{
    struct my_data *entry;

    rcu_read_lock();   /* 선점 비활성화 (CONFIG_PREEMPT_RCU에서는 실제 락) */
    list_for_each_entry_rcu(entry, &my_list, list) {
        printk(KERN_INFO "value: %d\n", entry->value);
    }
    rcu_read_unlock();
}

/* 쓰기 측: 새 항목 추가 */
int writer_add(int new_value)
{
    struct my_data *new_entry;

    new_entry = kmalloc(sizeof(*new_entry), GFP_KERNEL);
    if (!new_entry)
        return -ENOMEM;

    new_entry->value = new_value;

    spin_lock(&list_lock);
    list_add_rcu(&new_entry->list, &my_list);  /* 원자적으로 발행 */
    spin_unlock(&list_lock);

    return 0;
}

/* 해제 콜백: 그레이스 피리어드 완료 후 호출됨 */
static void my_data_free_rcu(struct rcu_head *rcu)
{
    struct my_data *entry = container_of(rcu, struct my_data, rcu);
    kfree(entry);
}

/* 쓰기 측: 항목 삭제 */
void writer_delete(int target_value)
{
    struct my_data *entry;

    spin_lock(&list_lock);
    list_for_each_entry(entry, &my_list, list) {
        if (entry->value == target_value) {
            list_del_rcu(&entry->list);  /* 리스트에서 제거 (독자는 아직 참조 가능) */
            spin_unlock(&list_lock);
            /* 그레이스 피리어드 후 비동기적으로 해제 */
            call_rcu(&entry->rcu, my_data_free_rcu);
            return;
        }
    }
    spin_unlock(&list_lock);
}
```

### 예제 2: RCU를 사용한 포인터 교체 패턴

```c
#include <linux/rcupdate.h>
#include <linux/slab.h>
#include <linux/mutex.h>

struct server_config {
    int max_connections;
    int timeout_ms;
    char server_name[64];
};

/* __rcu 어노테이션: sparse 정적 분석 도구가 잘못된 접근 탐지 */
static struct server_config __rcu *g_config;
static DEFINE_MUTEX(config_mutex);

/* 설정 읽기: 락 없이 */
void use_config(void)
{
    struct server_config *cfg;

    rcu_read_lock();
    cfg = rcu_dereference(g_config);  /* 올바른 메모리 순서 보장 */
    if (cfg) {
        printk(KERN_INFO "max_conn=%d, timeout=%dms, name=%s\n",
               cfg->max_connections,
               cfg->timeout_ms,
               cfg->server_name);
    }
    rcu_read_unlock();
    /* cfg는 rcu_read_unlock() 이후 절대 사용 불가! */
}

/* 설정 업데이트 */
int update_config(int new_max, int new_timeout, const char *name)
{
    struct server_config *old_cfg;
    struct server_config *new_cfg;

    new_cfg = kmalloc(sizeof(*new_cfg), GFP_KERNEL);
    if (!new_cfg)
        return -ENOMEM;

    new_cfg->max_connections = new_max;
    new_cfg->timeout_ms = new_timeout;
    strscpy(new_cfg->server_name, name, sizeof(new_cfg->server_name));

    mutex_lock(&config_mutex);

    /* 원자적 포인터 교체: 이 순간부터 새 독자는 new_cfg를 본다 */
    old_cfg = rcu_replace_pointer(g_config, new_cfg,
                                   lockdep_is_held(&config_mutex));

    mutex_unlock(&config_mutex);

    /*
     * 그레이스 피리어드 완료까지 블로킹 대기.
     * 이후 old_cfg를 참조하는 독자가 없음이 보장됨.
     */
    synchronize_rcu();
    kfree(old_cfg);

    return 0;
}
```

### RCU API 요약

| API | 설명 |
|-----|------|
| `rcu_read_lock()` | 읽기 측 임계 구역 시작 (선점 비활성화) |
| `rcu_read_unlock()` | 읽기 측 임계 구역 종료 |
| `rcu_dereference(p)` | 보호된 포인터 역참조 (메모리 배리어 포함) |
| `rcu_assign_pointer(p, v)` | 포인터를 원자적으로 발행 |
| `synchronize_rcu()` | 그레이스 피리어드 완료까지 블로킹 |
| `call_rcu(head, func)` | 그레이스 피리어드 후 비동기 콜백 등록 |
| `kfree_rcu(ptr, field)` | 그레이스 피리어드 후 kfree (단순화 매크로) |

## 주의사항 및 팁

### 수면 금지 (클래식 RCU)
`rcu_read_lock()` 임계 구역 내에서 수면(sleep)이 가능한 함수를 호출하면 커널 경고 또는 데드락이 발생한다. 수면이 필요한 경우 `srcu_read_lock()`/`srcu_read_unlock()`을 사용하는 SRCU(Sleepable RCU)를 사용해야 한다.

### rcu_dereference() 필수 사용
컴파일러 재정렬과 프로세서 추측 실행을 방지하려면 반드시 `rcu_dereference()`를 통해 포인터를 읽어야 한다. 직접 역참조는 희귀하지만 실제로 발생하는 경쟁 조건을 만든다.

```c
/* 잘못된 방법 — 컴파일러가 두 번 로드할 수 있음 */
if (g_config != NULL)
    printk("%d\n", g_config->max_connections);  /* g_config가 NULL이 될 수 있음 */

/* 올바른 방법 */
rcu_read_lock();
cfg = rcu_dereference(g_config);
if (cfg != NULL)
    printk("%d\n", cfg->max_connections);  /* 안전 */
rcu_read_unlock();
```

### 업데이터 간 동기화
RCU는 독자-작성자 동기화를 제공하지만, 여러 업데이터 간에는 별도의 스핀락 또는 뮤텍스가 필요하다.

### 메모리 해제 지연
`call_rcu()`는 그레이스 피리어드 후에 메모리를 해제하므로, 업데이트 빈도가 매우 높은 경우 미해제 메모리가 일시적으로 누적될 수 있다. `synchronize_rcu()`를 주기적으로 호출하거나 RCU 배치 크기를 조정해 대응할 수 있다.

### 유저 스페이스 RCU
커널 외부에서도 `liburcu` 라이브러리를 통해 RCU를 활용할 수 있다. 사용자 공간 프로그램에서 읽기 집약적인 공유 데이터 구조를 다룰 때 유용하다.

```c
/* liburcu 사용 예시 (userspace) */
#include <urcu.h>

rcu_read_lock();
struct config *cfg = rcu_dereference(global_cfg);
/* 데이터 사용 */
rcu_read_unlock();

/* 업데이트 측 */
struct config *new_cfg = /* 새 구성 생성 */;
struct config *old_cfg = rcu_xchg_pointer(&global_cfg, new_cfg);
synchronize_rcu();
free(old_cfg);
```

RCU는 복잡한 메커니즘이지만, 읽기 집약적인 동시성 문제에서 락 기반 접근보다 월등한 성능을 제공한다. 리눅스 커널에서 수천 곳에 사용되며, 현대 멀티코어 시스템을 가능하게 만든 핵심 기술 중 하나다.

## 참고 자료
- [What is RCU? — The Linux Kernel documentation](https://www.kernel.org/doc/html/next/RCU/whatisRCU.html)
- [What is RCU, Fundamentally? — LWN.net](https://lwn.net/Articles/262464/)
- [Read-copy-update — Wikipedia](https://en.wikipedia.org/wiki/Read-copy-update)
- [RCU Linux Kernel Newbies](https://kernelnewbies.org/Documentation/Subsystems/ReadCopyUpdate)
