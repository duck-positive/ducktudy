---
layout: post
title: "데이터베이스 버퍼 풀 매니저 완전 정복: PostgreSQL이 디스크 I/O를 최소화하는 방법"
date: 2026-07-15
categories: [cs, computer-science]
tags: [buffer-pool, database, postgresql, clock-sweep, page-replacement, storage, internals]
---

## 개념 설명

데이터베이스에서 가장 느린 연산은 디스크 I/O다. 현대 SSD도 메모리보다 수십~수백 배 느리고, HDD는 그보다 훨씬 느리다. **버퍼 풀 매니저(Buffer Pool Manager)**는 디스크 페이지를 메모리에 캐싱하여 디스크 I/O를 최소화하는 데이터베이스의 핵심 컴포넌트다. DBMS의 성능은 버퍼 풀 히트율(Buffer Pool Hit Rate)에 크게 의존하며, 이 값이 99%를 넘으면 거의 모든 읽기가 메모리에서 처리된다.

### 핵심 구성 요소

PostgreSQL의 버퍼 매니저는 세 가지 핵심 구조로 이루어진다:

```
┌─────────────────────────────────────────────────────┐
│                  공유 메모리 영역                     │
│                                                     │
│  ┌──────────────────┐  ┌─────────────────────────┐  │
│  │   Buffer Table    │  │   Buffer Descriptors    │  │
│  │ (Hash Table)      │  │   (Metadata 배열)       │  │
│  │                   │  │  ┌───────────────────┐  │  │
│  │ BufferTag →       │  │  │ refcount (핀 수)  │  │  │
│  │ buffer_id         │  │  │ usage_count       │  │  │
│  │                   │  │  │ dirty flag        │  │  │
│  │ O(1) 조회         │  │  │ buffer_tag        │  │  │
│  └──────────────────┘  │  └───────────────────┘  │  │
│                         └─────────────────────────┘  │
│  ┌──────────────────────────────────────────────┐    │
│  │              Buffer Pool                      │    │
│  │  [page_0][page_1][page_2]...[page_N-1]        │    │
│  │  각 슬롯 = 8KB (기본 페이지 크기)              │    │
│  └──────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘
```

**Buffer Table**: 버퍼 태그(파일 OID + 포크 번호 + 블록 번호)를 버퍼 ID로 매핑하는 해시 테이블. O(1)로 "이 페이지가 메모리에 있는가?"를 확인한다.

**Buffer Descriptor**: 각 버퍼 슬롯에 대한 메타데이터. `refcount`(현재 접근 중인 프로세스 수), `usage_count`(최근 접근 빈도), `dirty`(수정 여부) 등을 저장한다.

**Buffer Pool**: 실제 페이지 데이터가 저장되는 배열. 기본값은 `shared_buffers`(PostgreSQL 기본값: 128MB, 권장: 물리 메모리의 25%).

## 왜 필요한가

운영체제도 Page Cache를 통해 파일 I/O를 캐싱하지만, DBMS가 자체 버퍼 풀을 관리하는 이유가 있다:

1. **트랜잭션 제어**: DBMS는 언제 더티 페이지를 디스크에 플러시할지를 WAL과 연계해 정확히 제어해야 한다. 운영체제 캐시는 이를 보장하지 않는다.
2. **교체 정책 제어**: DBMS는 쿼리 계획을 알고 있으므로 "이 페이지는 곧 필요 없을 것"이라는 힌트(VACUUM, 순차 스캔 등)를 교체 정책에 반영할 수 있다.
3. **직접 I/O(O_DIRECT)**: 일부 DBMS는 커널 Page Cache를 완전히 우회해 이중 캐싱을 방지하고, 버퍼 풀만으로 캐싱을 단일화한다.

## 페이지 교체 알고리즘: Clock-Sweep

PostgreSQL은 LRU의 근사 알고리즘인 **Clock-Sweep**을 사용한다. LRU는 페이지 접근마다 연결 리스트를 업데이트해야 하므로 잠금 경합이 심해진다. Clock-Sweep은 단일 포인터를 회전시키는 방식으로 잠금 없이 거의 동등한 효과를 낸다.

```
버퍼 풀 (원형 배열):

       ↑ nextVictimBuffer (시계 방향 회전)
  [0]  [1]  [2]  [3]  [4]  [5]
  u=2  u=0  u=1  u=3  u=0  u=1
             ↑현재 위치

교체 후보 탐색 규칙:
  - usage_count == 0 이고 refcount == 0 → 즉시 교체 대상
  - usage_count > 0  → usage_count-- 하고 다음으로 넘어감
  - refcount > 0     → 다른 프로세스가 사용 중 → 건너뜀
```

## 실제 구현 예제

### 예제 1: Clock-Sweep 페이지 교체 알고리즘 구현

```python
from dataclasses import dataclass, field
from typing import Optional, Dict, Any
import threading

@dataclass
class BufferDescriptor:
    buffer_id: int
    page_tag: Optional[str] = None  # 실제로는 (filenode, fork, block)
    usage_count: int = 0
    refcount: int = 0
    dirty: bool = False
    data: Optional[bytes] = None

    def is_evictable(self) -> bool:
        return self.refcount == 0 and self.usage_count == 0

    def pin(self):
        """페이지 접근 시작: 핀을 꽂아 교체 방지"""
        self.refcount += 1
        if self.usage_count < 5:  # PostgreSQL: 최대 5
            self.usage_count += 1

    def unpin(self):
        """페이지 접근 완료: 핀 제거"""
        assert self.refcount > 0, "unpin without pin"
        self.refcount -= 1


class BufferPoolManager:
    def __init__(self, pool_size: int):
        self.pool_size = pool_size
        self.descriptors = [BufferDescriptor(i) for i in range(pool_size)]
        self.page_table: Dict[str, int] = {}  # tag -> buffer_id
        self.clock_hand = 0
        self.lock = threading.Lock()
        self.disk_reads = 0
        self.disk_writes = 0
        self.cache_hits = 0
        self.cache_misses = 0

    def _fetch_from_disk(self, page_tag: str) -> bytes:
        """디스크에서 페이지 읽기 시뮬레이션"""
        self.disk_reads += 1
        return f"[Page data: {page_tag}]".encode()

    def _write_to_disk(self, desc: BufferDescriptor):
        """더티 페이지를 디스크에 기록 시뮬레이션"""
        self.disk_writes += 1
        desc.dirty = False

    def _find_victim(self) -> int:
        """Clock-Sweep으로 교체 대상 버퍼 탐색"""
        attempts = 0
        while attempts < self.pool_size * 2:
            desc = self.descriptors[self.clock_hand]

            if desc.is_evictable():
                victim = self.clock_hand
                self.clock_hand = (self.clock_hand + 1) % self.pool_size
                return victim
            elif desc.refcount == 0 and desc.usage_count > 0:
                # usage_count 감소 (LRU 근사)
                desc.usage_count -= 1

            self.clock_hand = (self.clock_hand + 1) % self.pool_size
            attempts += 1

        raise RuntimeError("모든 버퍼가 핀 상태 — 버퍼 풀 소진")

    def get_page(self, page_tag: str) -> BufferDescriptor:
        """페이지를 버퍼 풀에서 반환 (없으면 디스크에서 로드)"""
        with self.lock:
            # 1. 버퍼 테이블에서 조회 (캐시 히트)
            if page_tag in self.page_table:
                buf_id = self.page_table[page_tag]
                desc = self.descriptors[buf_id]
                desc.pin()
                self.cache_hits += 1
                return desc

            # 2. 캐시 미스 → 교체 대상 탐색
            self.cache_misses += 1
            victim_id = self._find_victim()
            victim = self.descriptors[victim_id]

            # 3. 교체 대상 더티 페이지 플러시
            if victim.dirty:
                self._write_to_disk(victim)

            # 4. 기존 버퍼 테이블 엔트리 제거
            if victim.page_tag:
                del self.page_table[victim.page_tag]

            # 5. 디스크에서 새 페이지 로드
            victim.page_tag = page_tag
            victim.data = self._fetch_from_disk(page_tag)
            victim.dirty = False
            victim.usage_count = 1
            victim.refcount = 1

            self.page_table[page_tag] = victim_id
            return victim

    def release_page(self, desc: BufferDescriptor, dirty: bool = False):
        with self.lock:
            if dirty:
                desc.dirty = True
            desc.unpin()

    def stats(self) -> dict:
        hit_rate = (self.cache_hits /
                    max(self.cache_hits + self.cache_misses, 1) * 100)
        return {
            "cache_hits": self.cache_hits,
            "cache_misses": self.cache_misses,
            "hit_rate": f"{hit_rate:.1f}%",
            "disk_reads": self.disk_reads,
            "disk_writes": self.disk_writes,
        }


# 사용 예시
bpm = BufferPoolManager(pool_size=4)

# 5개 페이지 접근 (풀 크기=4이므로 교체 발생)
pages = ["rel1/blk0", "rel1/blk1", "rel1/blk2", "rel1/blk3", "rel1/blk0"]
for tag in pages:
    desc = bpm.get_page(tag)
    print(f"  접근: {tag:15s} → buf_id={desc.buffer_id}, usage={desc.usage_count}")
    bpm.release_page(desc)

print("\n버퍼 풀 상태:")
for d in bpm.descriptors:
    print(f"  buf[{d.buffer_id}]: tag={d.page_tag or 'empty':15s} "
          f"usage={d.usage_count} dirty={d.dirty}")

print(f"\n통계: {bpm.stats()}")
```

### 예제 2: 더블 버퍼링과 순차 스캔 최적화 (Ring Buffer)

대용량 순차 스캔(VACUUM, `pg_dump`, 전체 테이블 스캔)은 많은 페이지를 한 번씩만 읽는다. 이때 일반 버퍼 풀을 사용하면 자주 쓰이는 핫 페이지가 쫓겨나는 **캐시 오염(Cache Pollution)** 현상이 발생한다. PostgreSQL은 이를 막기 위해 순차 스캔에 소규모 **Ring Buffer**를 사용한다.

```python
class RingBuffer:
    """순차 스캔 전용 소규모 순환 버퍼 (캐시 오염 방지)"""
    RING_SIZE = 256  # 약 2MB (256 * 8KB)

    def __init__(self):
        self.slots = [None] * self.RING_SIZE
        self.head = 0
        self.evicted_count = 0
        self.loaded_count = 0

    def load_next(self, page_tag: str) -> str:
        """다음 슬롯에 페이지 적재 (이전 페이지 자동 퇴출)"""
        slot = self.head % self.RING_SIZE
        old_page = self.slots[slot]
        self.slots[slot] = page_tag
        self.head += 1
        self.loaded_count += 1
        if old_page is not None:
            self.evicted_count += 1
        return page_tag

    def current_pages(self) -> list:
        return [s for s in self.slots if s]


class BufferStrategy:
    """버퍼 전략 선택기"""

    @staticmethod
    def choose_strategy(scan_type: str, table_size_mb: float,
                        shared_buffers_mb: float):
        """
        PostgreSQL의 버퍼 전략 선택 로직
        - 소규모 테이블 (< shared_buffers / 4): 일반 버퍼 풀
        - 대규모 순차 스캔: Ring Buffer
        - VACUUM: 별도 Ring Buffer (크기 다름)
        """
        if scan_type == "sequential" and table_size_mb > shared_buffers_mb / 4:
            strategy = "RING_BUFFER"
            ring_size = min(RingBuffer.RING_SIZE, int(table_size_mb * 128))
        elif scan_type == "vacuum":
            strategy = "VACUUM_RING"
            ring_size = 6  # VACUUM은 매우 작은 링 사용
        else:
            strategy = "DEFAULT"
            ring_size = None

        return {
            "strategy": strategy,
            "ring_size": ring_size,
            "reason": {
                "RING_BUFFER": "대용량 순차 스캔 — 캐시 오염 방지",
                "VACUUM_RING": "VACUUM — 사용자 쿼리 방해 최소화",
                "DEFAULT": "소규모 접근 — 재사용 가능성 높음",
            }[strategy]
        }


# 전략 선택 시뮬레이션
print("버퍼 전략 선택 예시:")
configs = [
    ("sequential", 5, 128),     # 5MB 테이블, 128MB 버퍼
    ("sequential", 500, 128),   # 500MB 테이블, 128MB 버퍼
    ("random",     50, 128),    # 무작위 접근
    ("vacuum",     1000, 128),  # VACUUM
]
for scan, tbl_mb, buf_mb in configs:
    strategy = BufferStrategy.choose_strategy(scan, tbl_mb, buf_mb)
    print(f"  {scan:12s} {tbl_mb:6}MB → {strategy['strategy']:15s}: {strategy['reason']}")

# Ring Buffer 동작 시뮬레이션
print("\nRing Buffer 순차 스캔 (크기=4 축소 예시):")
ring = RingBuffer()
ring.RING_SIZE = 4
ring.slots = [None] * 4

for i in range(8):
    ring.load_next(f"table_blk_{i}")
    print(f"  blk_{i} 로드 → 현재 슬롯: {ring.slots}")
```

## 버퍼 관리와 WAL의 관계

버퍼 풀의 더티 페이지를 디스크에 기록하는 시점은 단순하지 않다. **WAL(Write-Ahead Logging)** 원칙에 의해 반드시 해당 변경 사항을 기록한 WAL 레코드가 먼저 디스크에 플러시된 이후에야 더티 페이지를 기록할 수 있다.

```
트랜잭션 커밋 순서:
1. WAL 버퍼에 변경 사항 기록
2. WAL 파일을 fsync() — 이것이 "내구성" 보장의 핵심
3. 트랜잭션 완료 반환 (클라이언트에 OK 응답)
4. 버퍼 풀의 더티 페이지는 나중에 bgwriter가 플러시

만약 순서가 바뀌면:
1. 더티 페이지를 디스크에 기록
2. 시스템 크래시 발생!
3. WAL 레코드 없음 → 복구 불가능한 불일치 상태
```

PostgreSQL의 `bgwriter`(백그라운드 라이터) 프로세스가 주기적으로 더티 페이지를 디스크에 기록하고, 체크포인트(checkpoint) 시에는 모든 더티 페이지가 플러시된다.

## 주의사항 및 팁

### 1. `shared_buffers` 크기 설정
일반적으로 물리 메모리의 25%를 권장한다. 너무 크게 설정하면 운영체제 Page Cache가 부족해 전체 성능이 오히려 떨어질 수 있다. PostgreSQL은 운영체제 Page Cache와 이중으로 동작하므로, 나머지 메모리는 OS 캐시에 위임하는 것이 효과적이다.

### 2. Hit Rate 모니터링
```sql
-- PostgreSQL 버퍼 히트율 확인
SELECT
    relname,
    heap_blks_read,
    heap_blks_hit,
    round(heap_blks_hit::numeric /
          nullif(heap_blks_hit + heap_blks_read, 0) * 100, 2) AS hit_rate
FROM pg_statio_user_tables
ORDER BY heap_blks_read DESC;
```
히트율이 99% 미만이면 `shared_buffers` 증가를 검토한다.

### 3. Clock-Sweep의 한계: Correlated Scan
여러 쿼리가 동시에 순차 스캔을 수행하면 같은 페이지를 서로 교체하는 **경쟁 상태**가 발생할 수 있다. PostgreSQL은 순차 스캔을 동기화해 같은 방향으로 진행하도록 하는 **동기화 스캔(Synchronized Scan)** 기법으로 이를 완화한다.

### 4. 핀(Pin) 관리 주의
`LockBufferForCleanup()` 같은 잠금을 오래 보유하면 다른 프로세스가 해당 버퍼를 사용할 수 없어 성능이 저하된다. 확장 플러그인을 작성할 때는 반드시 핀을 가능한 한 빨리 해제해야 한다.

### 5. MySQL InnoDB의 버퍼 풀과 비교
InnoDB는 LRU 변형인 **Midpoint LRU**를 사용한다. LRU 리스트를 Old(37%)와 New(63%) 서브리스트로 나누어, 새로 읽은 페이지는 Old 쪽에 삽입하고 짧은 시간 내 재접근이 이루어지면 New로 승격시킨다. 대용량 스캔이 New 리스트에 있는 핫 페이지를 밀어내지 못하도록 설계되어 있다.

## 참고 자료
- [postgres/postgres — Buffer Manager Source](https://github.com/postgres/postgres/blob/master/src/backend/storage/buffer/README)
- [CMU 15-445/645 — Buffer Pools (Lecture Notes)](https://15445.courses.cs.cmu.edu/fall2023/notes/05-bufferpool.pdf)
- [The Internals of PostgreSQL — Chapter 8: Buffer Manager](https://www.interdb.jp/pg/pgsql08.html)
