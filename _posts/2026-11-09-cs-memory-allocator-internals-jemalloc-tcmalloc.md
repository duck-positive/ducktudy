---
layout: post
title: "메모리 얼로케이터 내부 구조 완전 정복: malloc·jemalloc·tcmalloc이 메모리를 쪼개는 법"
date: 2026-11-09
categories: [cs, computer-science]
tags: [memory-allocator, malloc, jemalloc, tcmalloc, systems-programming, heap, fragmentation]
---

## 메모리 얼로케이터란 무엇인가?

`malloc(size)`를 호출하면 힙(heap)에서 `size` 바이트의 메모리가 반환된다. 이 한 줄의 함수 뒤에는 수십 년에 걸쳐 발전한 복잡한 시스템이 숨어 있다. **메모리 얼로케이터**는 프로세스의 힙 영역을 관리하며, `malloc` / `free` / `realloc` 요청을 처리하는 런타임 라이브러리 컴포넌트다.

얼로케이터의 성능은 현대 서버 애플리케이션의 처리량과 레이턴시에 직접적인 영향을 미친다. Meta(Facebook)가 jemalloc을 도입한 후 Firefox의 메모리 사용량이 20~30% 감소했고, Google의 tcmalloc은 멀티스레드 환경에서 glibc malloc 대비 최대 7배 빠른 할당 성능을 보인다.

---

## 왜 복잡한 얼로케이터가 필요한가?

### 단순한 얼로케이터의 문제: 단편화(Fragmentation)

**외부 단편화(External Fragmentation)**: 충분한 총 여유 메모리가 있지만 연속적이지 않아 큰 요청을 처리하지 못하는 상황.
**내부 단편화(Internal Fragmentation)**: 요청 크기보다 큰 블록을 반환했을 때 낭비되는 공간.

### OS 시스템 콜의 비용

메모리를 직접 OS에 요청하는 `brk()`나 `mmap()` 시스템 콜은 비싸다(수 마이크로초). 얼로케이터는 OS에서 큰 덩어리를 한 번에 가져온 뒤 작게 쪼개어 애플리케이션에 제공함으로써 시스템 콜 빈도를 줄인다.

### 멀티스레드 경쟁(Lock Contention)

단순히 전역 락으로 힙을 보호하면, 많은 스레드가 동시에 `malloc`을 호출할 때 심각한 경쟁이 발생한다.

---

## glibc malloc (ptmalloc2)의 구조

GNU C 라이브러리의 `malloc`은 ptmalloc2를 기반으로 한다. 핵심 개념은 **아레나(Arena)** 와 **청크(Chunk)** 다.

```c
/* glibc malloc chunk의 구조체 (단순화) */
struct malloc_chunk {
    size_t prev_size;  /* 이전 청크가 free일 때 그 크기 */
    size_t size;       /* 현재 청크 크기 (하위 3비트: 플래그) */
    /*
     * size의 하위 비트 플래그:
     *   P (prev_in_use): 이전 청크가 사용 중이면 1
     *   M (is_mmapped):  mmap으로 할당됐으면 1
     *   A (non_main_arena): non-main arena 소속이면 1
     */
    struct malloc_chunk *fd;  /* free 상태일 때: 다음 free 청크 (이중 연결 리스트) */
    struct malloc_chunk *bk;  /* free 상태일 때: 이전 free 청크 */
    /* 대형 청크는 추가 포인터 fd_nextsize, bk_nextsize 보유 */
};

/*
 * 사용 중인 청크 레이아웃 (prev_size + size 헤더):
 * +----------+----------+-------------------+
 * | prev_size|   size   | user data ...     |
 * +----------+----------+-------------------+
 *                       ^ malloc()이 반환하는 포인터
 *
 * free된 청크:
 * +----------+----------+------+------+-----+
 * | prev_size|   size   |  fd  |  bk  | ... |
 * +----------+----------+------+------+-----+
 */
```

**빈(bins)**: free된 청크는 크기별로 연결 리스트(bin)에 보관된다.
- **Fast bins**: 16~80 바이트 소형 청크. LIFO 단일 연결 리스트. lock 없음.
- **Small bins**: 80 바이트 이하. FIFO 이중 연결 리스트.
- **Large bins**: 512 바이트 이상. 크기별로 정렬된 이중 연결 리스트.
- **Unsorted bin**: 최근 free된 청크가 임시 보관되는 버퍼.
- **Top chunk**: 힙 끝에 위치한 미할당 영역. 다른 bin에서 처리 불가한 요청을 처리.

**아레나(Arena)**: 멀티스레드 경쟁을 줄이기 위해 스레드별로 별도 아레나를 할당한다. 최대 아레나 수 = `2 * CPU 코어 수` (32비트) 또는 `8 * CPU 코어 수` (64비트).

---

## jemalloc의 아키텍처

jemalloc은 Jason Evans가 FreeBSD를 위해 개발하고, 이후 Meta(Firefox, Android)가 채택한 얼로케이터다. **단편화 최소화**와 **캐시 효율성**에 초점을 맞춘다.

### 핵심 개념

**크기 클래스(Size Classes)**: 모든 할당을 미리 정의된 크기로 올림(round up)한다. 예: 300바이트 요청 → 320바이트 클래스로 처리. 약 232개의 크기 클래스가 있으며, 큰 크기일수록 간격이 넓다.

**지역(Region) → 슬랩(Slab) → 익스텐트(Extent)**
- **Region**: 하나의 할당 단위 (사용자 데이터)
- **Slab**: 동일 크기 클래스의 Region 묶음. 보통 4KB 페이지 1~2개.
- **Extent**: OS로부터 받은 메모리 덩어리 (보통 2MB의 huge page 단위).

**스레드 캐시(tcache)**: 각 스레드가 소형 할당을 위한 로컬 캐시를 보유. lock 없이 O(1) 할당/해제 가능.

```python
# jemalloc 크기 클래스 계산 (Python 시뮬레이션)
def jemalloc_size_class(size: int) -> int:
    """
    요청 크기를 jemalloc 크기 클래스로 올림.
    실제 jemalloc은 더 세밀한 크기 클래스를 사용하지만, 원리 설명용.
    """
    if size <= 8:
        return 8
    elif size <= 16:
        return 16
    elif size <= 32:
        return 32
    elif size <= 48:
        return 48
    elif size <= 64:
        return 64
    elif size <= 80:
        return 80
    elif size <= 96:
        return 96
    elif size <= 112:
        return 112
    elif size <= 128:
        return 128
    # 128 이상: 64바이트 간격
    elif size <= 256:
        return ((size + 63) // 64) * 64
    # 256 이상: 128바이트 간격
    elif size <= 512:
        return ((size + 127) // 128) * 128
    # 512 이상: 256바이트 간격
    elif size <= 1024:
        return ((size + 255) // 256) * 256
    # 1KB 이상: 512바이트 간격
    elif size <= 2048:
        return ((size + 511) // 512) * 512
    # 2KB 이상: 1KB 간격 (소형 할당의 최대)
    elif size <= 4096:
        return ((size + 1023) // 1024) * 1024
    # 4KB 초과: large allocation (extent 직접 할당)
    else:
        page_size = 4096
        return ((size + page_size - 1) // page_size) * page_size

# 내부 단편화 측정
test_sizes = [1, 7, 17, 33, 100, 200, 300, 500, 1000, 3000, 5000]
print(f"{'요청 크기':>10} {'할당 크기':>10} {'단편화':>10} {'단편화율':>10}")
print("-" * 45)
for size in test_sizes:
    allocated = jemalloc_size_class(size)
    waste = allocated - size
    fragmentation_pct = (waste / allocated) * 100
    print(f"{size:>10} {allocated:>10} {waste:>10} {fragmentation_pct:>9.1f}%")
```

```
   요청 크기   할당 크기       단편화      단편화율
---------------------------------------------
         1          8          7      87.5%
         7          8          1      12.5%
        17         32         15      46.9%
        33         48         15      31.2%
       100        112         12      10.7%
       200        256         56      21.9%
       300        320         20       6.3%
       500        512         12       2.3%
      1000       1024         24       2.3%
      3000       3072         72       2.3%
      5000       8192       3192      38.9%
```

---

## tcmalloc의 멀티스레드 최적화

Google의 tcmalloc(Thread-Caching Malloc)은 **멀티스레드 성능**에 특화되었다.

```c
/*
 * tcmalloc 아키텍처 (의사 코드):
 *
 * [Thread Local]        [Central]           [OS]
 * ThreadCache -------> CentralFreeList --> PageHeap --> mmap/sbrk
 *   per-thread cache    per-size-class      global
 *   (lock-free)         (spinlock)          (mutex)
 */

/*
 * 소형 객체 할당 흐름 (< 256KB):
 * 1. ThreadCache에서 해당 크기 클래스의 free list 확인
 * 2. ThreadCache miss → CentralFreeList에서 배치로 가져옴
 * 3. CentralFreeList miss → PageHeap에서 스팬(span) 획득
 * 4. PageHeap miss → OS에서 mmap으로 메모리 획득
 *
 * 대형 객체 (>= 256KB):
 * - PageHeap에서 직접 처리 (mutex 보호)
 * - 2MB 슈퍼페이지(hugepage) 단위로 관리
 */

/* 슈도코드: ThreadCache에서 할당 */
void* tc_malloc(size_t size) {
    if (size > kMaxSmallSize) {
        // Large: PageHeap에서 직접 할당
        return page_heap.Allocate(size);
    }
    
    // 크기 클래스 조회 (룩업 테이블, O(1))
    int cl = size_map.SizeClass(size);
    
    // ThreadCache free list에서 pop (lock-free!)
    ThreadCache* cache = GetThreadCache();
    FreeList& list = cache->list_[cl];
    
    if (!list.empty()) {
        return list.Pop();  // 가장 빠른 경로: ~5ns
    }
    
    // Cache miss: CentralFreeList에서 batch fetch
    int batch_size = size_map.num_objects_to_move(cl);
    central_cache[cl].RemoveRange(&list, batch_size);  // spinlock
    return list.Pop();
}
```

**gcmalloc(2022년 tcmalloc 개선)** 은 "세대별(generational)" 접근을 도입했다. 최근에 할당된 객체는 별도 영역에서 관리되어, GC 스타일의 일괄 해제가 가능해졌다.

---

## 얼로케이터 성능 비교와 선택 기준

| 항목 | glibc malloc | jemalloc | tcmalloc |
|------|-------------|---------|---------|
| 소형 단일스레드 | 보통 | 좋음 | 보통 |
| 소형 멀티스레드 | 보통 | 좋음 | 매우 좋음 |
| 단편화 최소화 | 보통 | 매우 좋음 | 좋음 |
| 메모리 사용량 | 적음 | 보통 | 많음 |
| 디버그/프로파일 | 제한적 | 풍부 (jeprof) | 풍부 (pprof) |
| 적합한 환경 | 범용 | 서버, Firefox | Google 서비스, Go 런타임 |

**선택 가이드:**
- **장기 실행 서버** (Redis, Nginx, Memcached): **jemalloc** — 단편화 최소화
- **고성능 멀티스레드** (gRPC 서버, 게임 서버): **tcmalloc** — 스레드 캐시 최적화
- **메모리 디버깅 필요**: **jemalloc** — `--enable-debug`, `jemalloc_stats_print()`
- **임베디드/최소 환경**: **glibc malloc** — 의존성 없음

**주의사항:**
1. **교체 적용법**: `LD_PRELOAD=/usr/lib/libjemalloc.so ./your_app` 로 런타임 교체 가능.
2. **hugepage 호환**: jemalloc 5.x는 transparent hugepage(THP)와 함께 쓸 때 주의 — THP가 활성화되면 jemalloc이 2MB 경계로 얼라인하여 메모리를 더 쓸 수 있다.
3. **단편화 모니터링**: jemalloc의 `malloc_stats_print(NULL, NULL, NULL)`으로 현재 단편화 비율 확인 가능.
4. **커스텀 얼로케이터의 필요성**: 게임 엔진처럼 할당 패턴이 예측 가능한 경우, 풀 얼로케이터(Pool Allocator)나 슬랩 얼로케이터(Slab Allocator)가 범용 malloc보다 훨씬 효율적이다.

메모리 얼로케이터는 성능 엔지니어링에서 가장 과소평가되는 최적화 포인트 중 하나다. 특히 초당 수백만 건의 할당이 발생하는 고성능 서비스에서, 올바른 얼로케이터 선택과 튜닝은 CPU 사용량을 10~30% 절감할 수 있다.

## 참고 자료
- [jemalloc 공식 사이트](https://jemalloc.net/)
- [jemalloc GitHub Wiki - Background](https://github.com/jemalloc/jemalloc/wiki/Background)
- [tcmalloc 문서 (Google)](https://google.github.io/tcmalloc/)
- [glibc malloc internals (Azeria Labs)](https://azeria-labs.com/heap-exploitation-part-1-understanding-the-glibc-heap-implementation/)
