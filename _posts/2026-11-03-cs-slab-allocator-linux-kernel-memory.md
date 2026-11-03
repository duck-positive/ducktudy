---
layout: post
title: "Slab Allocator: Linux 커널 메모리 관리 내부 구조 심층 분석"
date: 2026-11-03
categories: [cs, computer-science]
tags: [slab-allocator, linux-kernel, memory-management, slub, slob, kernel, systems-programming]
---

Linux 커널은 매 초 수십만 번의 메모리 할당/해제를 처리한다. 프로세스 생성(`task_struct`), 파일 열기(`file` 구조체), 네트워크 패킷(`sk_buff`) — 이 모든 것이 매우 빠르고 단편화 없이 처리되어야 한다. **Slab Allocator**는 이 문제를 **오브젝트 캐싱(Object Caching)**이라는 아이디어로 해결한 커널 메모리 할당자다. 1994년 Sun Microsystems의 Jeff Bonwick이 Solaris 커널을 위해 설계했고, Linux 커널에 1999년 도입된 이후 현재까지 발전해 왔다.

## 왜 일반적인 malloc으로는 부족한가

커널 수준의 메모리 할당에는 사용자 공간의 `malloc()`과 다른 요구사항이 있다.

**1. 고정 크기 반복 할당**: 커널은 `task_struct`(약 9KB), `inode`(약 600B), `dentry`(약 200B) 등 동일한 구조체를 수없이 할당/해제한다.

**2. 초기화 비용**: `task_struct` 같은 복잡한 구조체는 생성 시 뮤텍스, 리스트 헤드, 스핀락 등을 초기화해야 한다. Slab은 재사용 시 이 초기화를 **건너뛴다**.

**3. 하드웨어 캐시 정렬**: CPU 캐시 라인(일반적으로 64바이트)을 고려한 정렬로 캐시 미스를 줄인다.

**4. 메모리 단편화**: 작은 오브젝트를 페이지 단위로 할당하면 내부 단편화가 심각해진다. Slab은 페이지 내에 여러 오브젝트를 패킹한다.

## Slab Allocator의 구조

```
kmem_cache (캐시, 오브젝트 타입별)
├── slabs_full   (꽉 찬 슬랩들)
├── slabs_partial (일부 사용 중인 슬랩들)
└── slabs_free   (빈 슬랩들)
    └── slab (1개 이상의 페이지)
        ├── [obj][obj][obj][obj]...  ← 오브젝트들
        └── freelist (빈 슬롯 인덱스 배열)
```

### 핵심 용어
- **Cache (`kmem_cache`)**: 특정 타입 오브젝트를 관리하는 최상위 자료구조. `task_struct` 캐시, `inode` 캐시처럼 타입별로 존재한다.
- **Slab**: 한 개 이상의 연속된 물리 페이지. 동일 크기 오브젝트들로 채워진다.
- **Object**: 실제 할당 단위. `kmem_cache_alloc()`으로 반환되는 것.

## SLAB → SLUB 진화

Linux 커널의 Slab 구현체는 세 가지가 존재했다.

| 구현체 | 도입 버전 | 특징 |
|---|---|---|
| SLAB | 2.2 (1999) | Bonwick의 원래 설계, per-CPU/per-node 큐, 복잡한 코드 |
| SLUB | 2.6.22 (2007) | 단순화, per-CPU 슬랩 포인터, 더 나은 디버깅 |
| SLOB | 2.6.16 (2006) | 임베디드용 초소형, K&R 방식 |

현재 대부분의 Linux 시스템은 **SLUB**을 기본으로 사용한다.

### SLUB이 SLAB보다 나은 이유

SLAB의 가장 큰 문제는 per-CPU, per-node, 전역 큐 등 여러 계층의 큐를 관리하는 복잡성이었다. SLUB은 이를 단순화했다:

- **큐 제거**: SLAB의 `array_cache`(per-CPU 캐시 큐)를 제거하고 **per-CPU 슬랩 포인터** 하나로 대체
- **메타데이터 위치 변경**: SLAB은 슬랩 메타데이터를 별도 페이지에 두었지만, SLUB은 슬랩 첫 번째 오브젝트 앞에 내장
- **순서 있는 freelist**: SLUB의 freelist는 오브젝트 내부에 다음 빈 오브젝트 포인터를 저장 (인트루시브 연결 리스트)

## C로 이해하는 Slab 핵심 자료구조

```c
/* 실제 Linux 커널 코드를 단순화한 예시 */

struct kmem_cache {
    /* per-CPU 슬랩 포인터 (SLUB) */
    struct kmem_cache_cpu __percpu *cpu_slab;

    /* 오브젝트 크기 정보 */
    unsigned int size;         /* 각 오브젝트의 실제 크기 */
    unsigned int object_size;  /* 사용자가 요청한 크기 */
    unsigned int offset;       /* freelist 포인터 오프셋 */

    /* 슬랩 크기: 2^order 페이지 */
    unsigned int oo;           /* 최적 order */

    /* NUMA 노드별 슬랩 목록 */
    struct kmem_cache_node *node[MAX_NUMNODES];

    /* 캐시 이름 (디버깅용) */
    const char *name;
};

struct kmem_cache_cpu {
    void **freelist;         /* 다음 빈 오브젝트 포인터 */
    unsigned long tid;       /* 트랜잭션 ID (ABA 방지) */
    struct page *page;       /* 현재 슬랩 페이지 */
    struct page *partial;    /* partial 슬랩 목록 */
};

/* freelist: 오브젝트 내부에 다음 빈 슬롯 주소를 저장 */
static inline void *get_freepointer(struct kmem_cache *s, void *object) {
    /* object + offset 위치에 다음 빈 오브젝트의 주소가 저장됨 */
    return *(void **)(object + s->offset);
}

static inline void set_freepointer(struct kmem_cache *s, void *object, void *fp) {
    *(void **)(object + s->offset) = fp;
}
```

## 할당과 해제 흐름

```c
/*
 * kmem_cache_alloc 경로 (SLUB 단순화 버전)
 *
 * 빠른 경로: per-CPU 슬랩에서 freelist 포인터 하나 꺼내기
 * 느린 경로: 새 슬랩 할당 필요 시 buddy allocator에서 페이지 요청
 */

void *kmem_cache_alloc(struct kmem_cache *s, gfp_t gfpflags) {
    void *object;

    // 빠른 경로: per-CPU 슬랩에서 직접 할당 (인터럽트 비활성화 후)
    struct kmem_cache_cpu *c = this_cpu_ptr(s->cpu_slab);

    object = c->freelist;
    if (likely(object)) {
        // freelist에서 꺼내고 다음 빈 오브젝트로 포인터 이동
        c->freelist = get_freepointer(s, object);
        c->tid += TID_STEP;
        return object;
    }

    // 느린 경로: 새 슬랩이 필요
    return __slab_alloc(s, gfpflags, ...);
}

void kmem_cache_free(struct kmem_cache *s, void *object) {
    // 빠른 경로: 현재 per-CPU 슬랩에 반환
    struct kmem_cache_cpu *c = this_cpu_ptr(s->cpu_slab);

    if (likely(c->page == virt_to_head_page(object))) {
        // freelist 앞에 추가 (LIFO 방식)
        set_freepointer(s, object, c->freelist);
        c->freelist = object;
        c->tid += TID_STEP;
        return;
    }

    // 느린 경로: 다른 슬랩에 반환
    __slab_free(s, virt_to_head_page(object), object, ...);
}
```

## 직접 실험: /proc/slabinfo 분석

Linux 시스템에서 현재 slab 캐시 상태를 확인할 수 있다.

```bash
# /proc/slabinfo 주요 컬럼 확인
# <name> <active_objs> <num_objs> <objsize> <objperslab> <pagesperslab>

cat /proc/slabinfo | head -5
# slabinfo - version: 2.1
# # name    <active_objs> <num_objs> <objsize> <objperslab> : tunables ...
# task_struct    324    324   9152    3 : tunables    0    0    0
# mm_struct       84     84    832    7 : tunables    0    0    0
# filp           272    560    256   16 : tunables    0    0    0

# slabtop: 실시간 slab 캐시 모니터링 (상위 소비자 확인)
sudo slabtop -o | head -20

# 특정 캐시 통계 확인 (SLUB 디버그 정보)
cat /sys/kernel/slab/task_struct/alloc_calls 2>/dev/null || \
cat /sys/kernel/debug/slab/task_struct/alloc_calls 2>/dev/null
```

## 커널 모듈에서 커스텀 Slab 캐시 사용

커널 모듈 개발 시 자주 할당/해제하는 구조체는 전용 slab 캐시를 만드는 것이 좋다.

```c
#include <linux/slab.h>
#include <linux/module.h>

/* 반복 할당할 구조체 */
struct my_request {
    unsigned long id;
    char data[256];
    struct list_head list;
    spinlock_t lock;
};

/* 모듈 전역 캐시 포인터 */
static struct kmem_cache *my_cache;

/* 캐시 생성자: 오브젝트가 처음 슬랩에 들어올 때만 호출 */
static void my_request_ctor(void *p) {
    struct my_request *req = p;
    /* 한 번만 초기화하는 필드들 */
    INIT_LIST_HEAD(&req->list);
    spin_lock_init(&req->lock);
}

static int __init my_module_init(void) {
    /* slab 캐시 생성 */
    my_cache = kmem_cache_create(
        "my_request_cache",       /* 이름 */
        sizeof(struct my_request), /* 오브젝트 크기 */
        0,                        /* 정렬 (0 = 기본값) */
        SLAB_HWCACHE_ALIGN |      /* CPU 캐시 정렬 */
        SLAB_POISON,              /* 디버그: 해제 시 0x6b로 채움 */
        my_request_ctor           /* 생성자 콜백 */
    );

    if (!my_cache)
        return -ENOMEM;

    pr_info("Cache created: object_size=%u\n",
            kmem_cache_size(my_cache));

    /* 할당 */
    struct my_request *req = kmem_cache_alloc(my_cache, GFP_KERNEL);
    if (!req) {
        kmem_cache_destroy(my_cache);
        return -ENOMEM;
    }

    req->id = 42;
    pr_info("Allocated request id=%lu\n", req->id);

    /* 해제 */
    kmem_cache_free(my_cache, req);

    return 0;
}

static void __exit my_module_exit(void) {
    /* 모든 오브젝트가 해제된 후 캐시 삭제 */
    kmem_cache_destroy(my_cache);
}

module_init(my_module_init);
module_exit(my_module_exit);
MODULE_LICENSE("GPL");
```

## SLUB 디버깅

SLUB은 강력한 디버깅 기능을 내장하고 있다.

```bash
# 부트 파라미터로 SLUB 디버깅 활성화
# (커널 컴파일 시 CONFIG_SLUB_DEBUG=y 필요)
# slub_debug=F,task_struct  → task_struct 캐시에 freelist 검증 활성화
# slub_debug=FPUZ           → 전체 디버그 (Freelist, Poisoning, UsedObject, Red zones)

# 커널 빌드 시 옵션:
# CONFIG_SLUB_DEBUG_ON=y   → 항상 디버그 활성화
# CONFIG_SLAB_FREELIST_RANDOM=y → freelist 랜덤화 (보안)
# CONFIG_SLAB_FREELIST_HARDENED=y → freelist 포인터 암호화

# 런타임 통계
cat /sys/kernel/slab/kmalloc-64/alloc_fastpath
cat /sys/kernel/slab/kmalloc-64/alloc_slowpath
```

## kmalloc과 Slab의 관계

`kmalloc(size, flags)`는 내부적으로 size에 맞는 **일반 목적 slab 캐시**를 선택한다.

```
kmalloc-8, kmalloc-16, kmalloc-32, kmalloc-64, kmalloc-128, ...
kmalloc-1k, kmalloc-2k, kmalloc-4k, ... kmalloc-8M (최대)
```

요청 크기를 올림하여 가장 작은 적합한 kmalloc 캐시를 사용한다. 예: 70바이트 요청 → kmalloc-128 캐시 사용.

## 주의사항 및 실무 팁

**1. 캐시 오염(Cache Poisoning) 주의**: `SLAB_POISON` 플래그를 디버그용으로만 사용하라. 프로덕션에서는 성능 저하를 일으킨다.

**2. 생성자 함수 남용 금지**: 생성자는 오브젝트가 슬랩에 처음 배치될 때만 호출된다. 재사용 시에는 호출되지 않으므로, 매 할당마다 초기화가 필요한 필드는 `kmem_cache_alloc()` 후 직접 초기화하라.

**3. GFP 플래그 선택**: 인터럽트 컨텍스트에서는 `GFP_ATOMIC` (sleep 불가), 일반 컨텍스트에서는 `GFP_KERNEL` 사용. 잘못된 플래그는 데드락이나 BUG()를 유발한다.

**4. 메모리 누수 탐지**: `kmemleak` (CONFIG_DEBUG_KMEMLEAK=y)를 사용하면 커널 slab 메모리 누수를 탐지할 수 있다. `/sys/kernel/debug/kmemleak`에서 확인 가능.

**5. 크기 선택**: 오브젝트 크기가 페이지 크기(4KB)의 절반을 넘으면 `kmem_cache`보다 `alloc_pages()`를 직접 사용하는 것이 효율적이다.

Slab Allocator는 커널 메모리 관리의 가장 우아한 해결책 중 하나다. 오브젝트 캐싱이라는 단순한 아이디어에서 시작해, NUMA 인식, CPU 캐시 최적화, 디버깅 인프라까지 수십 년에 걸쳐 정교하게 발전했다. 고성능 시스템 프로그래밍을 이해하는 데 Slab 내부 구조 파악은 필수다.

## 참고 자료
- [The Slab Allocator: An Object-Caching Kernel Memory Allocator – Jeff Bonwick (1994)](https://www.usenix.org/legacy/publications/library/proceedings/bos94/bonwick.html)
- [Understanding the Linux Virtual Memory Manager – Gorman](https://kernel.org/doc/gorman/html/understand/understand011.html)
- [The SLUB allocator – LWN.net](https://lwn.net/Articles/229984/)
- [Slab allocators in the Linux Kernel: SLAB, SLOB, SLUB – Christoph Lameter](https://events.static.linuxfound.org/sites/events/files/slides/slaballocators.pdf)
