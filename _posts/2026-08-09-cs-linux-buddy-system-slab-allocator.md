---
layout: post
title: "Linux 커널 메모리 관리 완전 정복: Buddy System과 Slab Allocator의 내부 동작 원리"
date: 2026-08-09
categories: [cs, computer-science]
tags: [linux-kernel, buddy-system, slab-allocator, slub, memory-management, os, kernel]
---

## Linux 커널은 메모리를 어떻게 관리하는가

사용자 공간 프로그램이 `malloc()`을 호출하면 C 라이브러리(glibc의 ptmalloc, tcmalloc 등)가 처리한다. 그렇다면 커널 자체는 메모리가 필요할 때 어떻게 할당받을까? 답은 두 개의 계층적 할당자에 있다.

1. **버디 시스템(Buddy System)**: 물리적 페이지(4KB 단위) 수준의 할당을 담당한다.
2. **슬랩 할당자(Slab Allocator)**: 버디 시스템 위에 올라가 작은 커널 오브젝트(수십~수백 바이트) 단위의 할당을 담당한다.

이 두 자료구조는 Linux 커널의 심장부로, 1990년대부터 수십 년간 수백만 대의 서버, 스마트폰, 임베디드 장비를 지탱해왔다. 메모리 관리를 제대로 이해하면 커널 모듈 개발, 드라이버 작성, 시스템 성능 튜닝의 기반이 된다.

---

## 메모리 존(Zone)과 물리 메모리 구조

Linux는 물리 메모리를 **존(Zone)**으로 나누어 관리한다.

| 존 | 주소 범위 (x86-64) | 용도 |
|----|-------------------|------|
| ZONE_DMA | 0 ~ 16 MB | 레거시 ISA DMA 장치 (24비트 주소 제한) |
| ZONE_DMA32 | 0 ~ 4 GB | 32비트 DMA 장치 |
| ZONE_NORMAL | 4 GB ~ | 일반 커널 메모리 |
| ZONE_HIGHMEM | (32비트 전용) | 커널 주소 공간이 닿지 않는 고주소 영역 |

각 존은 독립적인 버디 시스템 풀(free list)을 가지며, `alloc_pages(GFP_KERNEL, order)` 같은 함수가 올바른 존에서 페이지를 할당한다.

---

## 버디 시스템 (Buddy System)

### 핵심 아이디어

버디 시스템은 메모리를 **2의 거듭제곱(power-of-two)** 크기의 블록으로 관리한다. Linux에서 최소 단위는 4KB 페이지(order 0)이며, 최대 order는 보통 11(4KB × 2¹¹ = 8MB)이다.

각 존(zone)은 order 0부터 MAX_ORDER-1까지의 **free_list**를 유지한다. 각 free_list는 해당 order 크기의 연속 물리 페이지 블록들의 링크드 리스트다.

```
free_list[0]: [4KB 블록들] ↔ [4KB 블록들] ↔ ...
free_list[1]: [8KB 블록들] ↔ [8KB 블록들] ↔ ...
free_list[2]: [16KB 블록들] ↔ ...
...
free_list[11]: [8MB 블록들] ↔ ...
```

### 할당 과정 (Allocation)

`alloc_pages(GFP_KERNEL, order=2)` (16KB 요청):

1. free_list[2]에 블록이 있으면 즉시 반환.
2. 없으면 free_list[3]에서 32KB 블록을 가져와 **절반으로 분할(split)**: 16KB 버디 쌍을 만든다.
3. 하나는 반환, 나머지 버디는 free_list[2]에 등록.
4. free_list[3]도 없으면 free_list[4]까지 올라가며 반복.

### 해제 과정 (Deallocation) — 병합(Coalescing)

페이지를 반환할 때 버디 시스템은 **버디(buddy)**를 찾아 병합한다.

```
버디 주소 계산: buddy_addr = block_addr XOR (block_size)
예) 4096 XOR 4096 = 8192  (4KB 블록의 버디는 8KB 위치에 있는 4KB 블록)
```

버디가 현재 free_list에 있으면 두 블록을 합쳐 상위 order free_list에 등록한다. 이 과정을 최대 order까지 반복(cascading coalescing)함으로써 메모리 단편화를 최소화한다.

### 예제 1: 커널 모듈에서 페이지 직접 할당

```c
#include <linux/module.h>
#include <linux/gfp.h>        // GFP 플래그
#include <linux/mm.h>         // alloc_pages, free_pages
#include <linux/slab.h>       // kmalloc, kfree
#include <linux/highmem.h>    // page_address

MODULE_LICENSE("GPL");

static int __init buddy_demo_init(void) {
    struct page *pages;
    void *vaddr;
    unsigned long order = 2; // 2^2 = 4 페이지 = 16KB

    /* ===== 버디 시스템으로 물리 페이지 직접 할당 ===== */
    pages = alloc_pages(GFP_KERNEL | __GFP_ZERO, order);
    if (!pages) {
        pr_err("alloc_pages 실패!\n");
        return -ENOMEM;
    }

    /* 첫 번째 페이지의 커널 가상 주소 획득 */
    vaddr = page_address(pages);
    pr_info("할당된 물리 주소: %pa, 가상 주소: %px\n",
            &page_to_phys(pages), vaddr);
    pr_info("할당된 크기: %lu KB (order=%lu, pages=%lu)\n",
            (1UL << order) * PAGE_SIZE / 1024, order, 1UL << order);

    /* 메모리 사용 */
    memset(vaddr, 0xAB, PAGE_SIZE * (1UL << order));
    pr_info("첫 바이트: 0x%02x\n", *(uint8_t *)vaddr);

    /* 해제 — 버디 병합이 자동으로 수행됨 */
    free_pages((unsigned long)vaddr, order);
    pr_info("페이지 해제 완료 (버디 병합 수행됨)\n");

    /* ===== kmalloc: 슬랩 할당자 래퍼 ===== */
    size_t size = 256;
    void *kbuf = kmalloc(size, GFP_KERNEL);
    if (!kbuf) {
        pr_err("kmalloc 실패!\n");
        return -ENOMEM;
    }
    pr_info("kmalloc(%zu bytes) = %px\n", size, kbuf);
    kfree(kbuf);

    return 0; // 바로 언로드됨 (데모 목적)
}

static void __exit buddy_demo_exit(void) {
    pr_info("버디 시스템 데모 모듈 언로드\n");
}

module_init(buddy_demo_init);
module_exit(buddy_demo_exit);
```

---

## 슬랩 할당자 (Slab Allocator)

### 왜 슬랩 할당자가 필요한가

커널은 `task_struct`(프로세스 디스크립터, ~수 KB), `inode`(파일 시스템 노드), `dentry`(디렉토리 항목) 같은 고정 크기 오브젝트를 **수만 개씩** 빈번하게 할당·해제한다. 버디 시스템으로 직접 관리하면:

1. **내부 단편화**: 100바이트 오브젝트를 위해 4KB 페이지 전체를 낭비
2. **느린 생성자 실행**: 매번 오브젝트를 초기화(생성자 실행)하는 오버헤드
3. **캐시 미스**: 자주 쓰는 오브젝트가 캐시에서 빠져나가 콜드 캐시 상태로 할당됨

슬랩 할당자는 이 세 가지 문제를 모두 해결한다.

### 슬랩 구조

```
[kmem_cache "task_struct"] 
  ├── [slab 0] = 한 페이지(또는 여러 페이지)
  │     ├── [object 0: task_struct] (FREE)
  │     ├── [object 1: task_struct] (USED)
  │     └── [object 2: task_struct] (FREE)
  ├── [slab 1] = ...
  └── [slab 2] = ...
```

각 `kmem_cache`는 **특정 오브젝트 타입** 전용이다. 슬랩은 하나 이상의 연속 페이지로 이루어지며, 고정 크기 오브젝트들이 배열처럼 배치된다. 각 슬랩은 세 가지 상태 중 하나다:

- **full**: 모든 오브젝트가 사용 중
- **partial**: 일부만 사용 중 (우선 할당 대상)
- **empty**: 모든 오브젝트가 free (일정 시간 후 버디 시스템에 반납)

### SLAB → SLUB → SLOB

Linux 커널에는 슬랩 할당자의 세 가지 구현이 있으며, 빌드 시 하나를 선택한다:

| 구현 | 특징 | 적합한 환경 |
|------|------|------------|
| SLAB | 원조, per-CPU 캐시·per-node 큐 구비, 메타데이터 오버헤드 있음 | 레거시 |
| **SLUB** | 현재 기본값, 메타데이터 최소화, 디버깅 편리 | 일반 서버·데스크톱 |
| SLOB | 초소형 임베디드용, 단순 first-fit 방식 | IoT, 임베디드 |

현재 대부분의 Linux 배포판은 **SLUB**을 사용한다 (`cat /boot/config-$(uname -r) | grep SLUB`로 확인 가능).

---

## 슬랩 할당자 API 상세

| 함수 | 용도 |
|------|------|
| `kmalloc(size, flags)` | 범용 커널 메모리 할당 (슬랩 사용) |
| `kzalloc(size, flags)` | kmalloc + 제로 초기화 |
| `kfree(ptr)` | kmalloc/kzalloc으로 할당된 메모리 해제 |
| `vmalloc(size)` | 가상 주소 연속 (물리 비연속) 대용량 할당 |
| `kmem_cache_create(...)` | 전용 슬랩 캐시 생성 |
| `kmem_cache_alloc(cache, flags)` | 캐시에서 오브젝트 할당 |
| `kmem_cache_free(cache, obj)` | 오브젝트를 캐시에 반환 |
| `kmem_cache_destroy(cache)` | 캐시 삭제 |

### GFP 플래그

| 플래그 | 의미 |
|--------|------|
| `GFP_KERNEL` | 일반 커널 컨텍스트, 슬립 허용 |
| `GFP_ATOMIC` | 인터럽트 핸들러 등 슬립 불허 컨텍스트 |
| `GFP_DMA` | DMA 가능 존에서 할당 |
| `__GFP_ZERO` | 할당된 메모리 제로 초기화 |
| `__GFP_NOWARN` | 실패해도 경고 메시지 없음 |

---

## 예제 2: 커스텀 kmem_cache 생성 및 사용

```c
#include <linux/module.h>
#include <linux/slab.h>
#include <linux/init.h>

MODULE_LICENSE("GPL");

/* 우리가 관리할 커스텀 오브젝트 */
struct my_object {
    int    id;
    char   name[64];
    u64    timestamp;
    struct list_head list;
};

static struct kmem_cache *my_cache;
static LIST_HEAD(object_list);

/* 생성자: 슬랩이 처음 만들어질 때 한 번 호출됨 */
static void my_object_ctor(void *obj) {
    struct my_object *o = (struct my_object *)obj;
    memset(o, 0, sizeof(*o));
    INIT_LIST_HEAD(&o->list);
    /* 공통 초기화를 여기서 하면 alloc 시 생략 가능 */
}

static int __init slab_demo_init(void) {
    int i;

    /* ===== 1. kmem_cache 생성 ===== */
    my_cache = kmem_cache_create(
        "my_object_cache",          // /proc/slabinfo에 표시되는 이름
        sizeof(struct my_object),   // 오브젝트 크기
        0,                          // 정렬 (0 = 자동)
        SLAB_HWCACHE_ALIGN |        // 캐시라인 정렬
        SLAB_POISON |               // 디버그: 해제된 메모리 오염 패턴
        SLAB_RED_ZONE,              // 디버그: 버퍼 오버플로 탐지
        my_object_ctor              // 생성자
    );

    if (!my_cache) {
        pr_err("kmem_cache_create 실패!\n");
        return -ENOMEM;
    }

    pr_info("슬랩 캐시 생성: 오브젝트 크기=%zu bytes\n",
            kmem_cache_size(my_cache));

    /* ===== 2. 오브젝트 할당 ===== */
    for (i = 0; i < 5; i++) {
        struct my_object *obj = kmem_cache_alloc(my_cache, GFP_KERNEL);
        if (!obj) {
            pr_err("kmem_cache_alloc 실패 (i=%d)\n", i);
            break;
        }

        obj->id = i;
        snprintf(obj->name, sizeof(obj->name), "object_%d", i);
        obj->timestamp = ktime_get_ns();

        list_add(&obj->list, &object_list);
        pr_info("오브젝트 %d 할당: %px\n", i, obj);
    }

    /* ===== 3. 오브젝트 일부 해제 ===== */
    {
        struct my_object *obj, *tmp;
        int freed = 0;
        list_for_each_entry_safe(obj, tmp, &object_list, list) {
            if (obj->id % 2 == 0) { // 짝수 id만 해제
                pr_info("오브젝트 %d 해제: %px\n", obj->id, obj);
                list_del(&obj->list);
                kmem_cache_free(my_cache, obj); // 슬랩으로 반환 (재사용 가능)
                freed++;
            }
        }
        pr_info("%d개 해제 완료\n", freed);
    }

    /* ===== 4. /proc/slabinfo 확인 포인트 =====
     * 실제 커널 모듈 실행 중: cat /proc/slabinfo | grep my_object_cache
     * 출력: active_objs, num_objs, objsize, objperslab 등 확인 가능
     */

    return 0;
}

static void __exit slab_demo_exit(void) {
    struct my_object *obj, *tmp;

    /* 남은 오브젝트 모두 해제 */
    list_for_each_entry_safe(obj, tmp, &object_list, list) {
        list_del(&obj->list);
        kmem_cache_free(my_cache, obj);
    }

    /* 캐시 삭제 — 반드시 모든 오브젝트를 반환한 후에 호출 */
    kmem_cache_destroy(my_cache);
    pr_info("슬랩 캐시 삭제 완료\n");
}

module_init(slab_demo_init);
module_exit(slab_demo_exit);
```

---

## 버디 시스템 vs 슬랩 할당자 선택 기준

```
요청 크기별 추천 함수:
├── 수 바이트 ~ 수 KB  → kmalloc()   (슬랩 사용, per-CPU 최적화)
├── 정해진 오브젝트 타입 → kmem_cache_alloc()  (전용 캐시, 생성자 재사용)
├── 물리 연속 필요 (DMA) → alloc_pages() + GFP_DMA
├── 물리 비연속 대용량   → vmalloc()  (가상 연속, 물리 단편화 무관)
└── 수 MB 이상 물리 연속 → alloc_pages() + order 지정
```

`vmalloc`은 가상 주소 공간은 연속이지만 물리 메모리는 비연속이다. TLB 엔트리를 많이 차지하므로 대용량 임시 버퍼 외에는 권장하지 않는다.

---

## 슬랩 통계 확인

```bash
# 슬랩 정보 전체 보기
cat /proc/slabinfo

# 활성 오브젝트 수, 크기, slab당 오브젝트 수
# 컬럼: name active_objs num_objs objsize objperslab pagesperslab
sudo slabtop         # 실시간 슬랩 모니터링 (top 유사)

# 특정 캐시 상세 정보 (SLUB 디버그 모드)
cat /sys/kernel/slab/kmalloc-256/alloc_calls

# 메모리 존별 버디 시스템 상태
cat /proc/buddyinfo
# 예시: Node 0, zone Normal  100 50 20 5 2 1 0 0 0 0 0
#        각 숫자가 order 0~10의 free 블록 수
```

---

## 주의사항 및 팁

### 1. 인터럽트 컨텍스트에서의 할당

인터럽트 핸들러나 소프트IRQ 컨텍스트에서는 **반드시 `GFP_ATOMIC`**을 사용해야 한다. `GFP_KERNEL`은 메모리 부족 시 슬립(sleep)을 허용하는데, 인터럽트 컨텍스트에서 슬립하면 커널 패닉이 발생한다.

```c
// 인터럽트 핸들러 내부
void *buf = kmalloc(size, GFP_ATOMIC); // GFP_KERNEL 절대 금지!
```

### 2. kmalloc 크기 오버헤드

kmalloc은 내부적으로 2의 거듭제곱 크기로 반올림해서 할당한다 (33바이트를 요청하면 64바이트 슬랩에서 할당). 비슷한 크기의 구조체가 많다면 `kmem_cache_create`로 전용 캐시를 만드는 것이 메모리 효율이 좋다.

### 3. 슬랩 디버깅

`CONFIG_SLUB_DEBUG=y` 빌드 옵션을 켜면 POISON 패턴(해제된 메모리를 0x6b로 채움), RED_ZONE(버퍼 경계 감지), 할당 추적을 사용할 수 있다. 개발 중 use-after-free나 double-free를 조기에 발견하는 데 유용하다.

### 4. 메모리 압박과 shrinker

슬랩 캐시는 시스템 메모리가 부족해지면 **shrinker** 메커니즘을 통해 빈 슬랩을 버디 시스템에 반납한다. 커널 모듈에서 대용량 캐시를 만들 때는 `register_shrinker()`로 커스텀 shrinker를 등록해 메모리 압박 상황에 대응하도록 설계해야 한다.

---

## 참고 자료

- [torvalds/linux — mm/page_alloc.c (버디 시스템 구현)](https://github.com/torvalds/linux/blob/master/mm/page_alloc.c)
- [torvalds/linux — mm/slub.c (SLUB 할당자 구현)](https://github.com/torvalds/linux/blob/master/mm/slub.c)
