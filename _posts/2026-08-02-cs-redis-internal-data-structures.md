---
layout: post
title: "Redis 내부 자료구조 완전 정복: SDS·listpack·quicklist·skiplist가 메모리를 최적화하는 방법"
date: 2026-08-02
categories: [cs, computer-science]
tags: [redis, data-structures, sds, skiplist, quicklist, listpack, memory-optimization, nosql]
---

## 개념 설명

Redis는 "Remote Dictionary Server"의 약자로, 단순한 키-값 캐시처럼 보이지만 내부적으로는 정교하게 설계된 자료구조들의 집합체다. 사용자가 `ZADD`, `LPUSH`, `HSET` 같은 명령을 사용할 때, Redis는 데이터의 **크기와 특성**에 따라 내부 인코딩을 자동으로 선택하여 메모리와 CPU 사용을 최적화한다.

예를 들어 `Sorted Set`은 데이터가 128개 이하이고 모든 값의 길이가 64바이트 이하면 `listpack`으로 저장되지만, 그 이상이 되면 `skiplist + hashtable` 조합으로 자동 변환된다. 이런 적응형 인코딩(adaptive encoding)이 Redis가 극도의 메모리 효율성을 달성하는 핵심이다.

### 주요 내부 자료구조 목록

| 자료구조 | 역할 | 사용되는 타입 |
|---------|------|--------------|
| SDS | 문자열 저장 | String, Hash key/value |
| listpack | 소형 컬렉션 연속 저장 | List, Hash, ZSet (소형) |
| quicklist | 대형 리스트 | List (대형) |
| hashtable | O(1) 키 조회 | Hash, Set (대형) |
| skiplist | 정렬된 집합 | ZSet (대형) |
| intset | 정수 집합 | Set (정수만 있는 경우) |

---

## 왜 필요한가

### C 문자열의 한계와 SDS

Redis는 C 언어로 작성되었지만 C의 기본 문자열(`char*`)을 직접 사용하지 않는다. C 문자열은 다음 문제가 있다:

1. **길이 계산 O(n)**: `strlen()`은 `\0`을 만날 때까지 순회해야 한다
2. **바이너리 비안전**: `\0`이 중간에 오면 문자열이 잘린다
3. **버퍼 오버플로우**: 크기 추적이 없어 메모리 오염 위험

### 컬렉션 타입의 이중 압력

- **메모리 압력**: Redis는 가능한 한 적은 메모리를 사용해야 한다. 빈 링크드 리스트 노드 하나도 포인터 2개(16바이트)를 차지한다
- **성능 압력**: 메모리가 연속적일수록 CPU 캐시 효율이 높아 빠르다

이 두 가지를 동시에 만족시키기 위해 Redis는 크기에 따른 **이중 인코딩 전략**을 사용한다.

---

## 실제 구현 예제

### 예제 1: SDS(Simple Dynamic String) 구조 이해

SDS는 Redis의 기본 문자열 구현이다. C 문자열과의 하위 호환성을 유지하면서 길이 추적과 바이너리 안전성을 추가한다.

```c
/* sds.h에서 발췌 (Redis 7.x 기준) */

/* 헤더 타입: 문자열 길이에 따라 다른 크기의 헤더 사용 */
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t  len;    /* 현재 사용 중인 길이 (최대 255바이트) */
    uint8_t  alloc;  /* 할당된 전체 공간 (헤더와 null terminator 제외) */
    unsigned char flags; /* 헤더 타입 식별자 (하위 3비트) */
    char     buf[];  /* 실제 문자열 데이터 */
};

struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len;
    uint32_t alloc;
    unsigned char flags;
    char     buf[];
};

/* SDS 포인터는 항상 buf를 가리킨다.
 * 헤더에 접근하려면 포인터를 뒤로 이동시킨다.
 *
 *  [sdshdr8 header] [buf...] [\0]
 *                   ^
 *            sds 포인터가 여기를 가리킴
 */

/* 주요 매크로 */
#define sdslen(s)    (((struct sdshdr8 *)(s - sizeof(struct sdshdr8)))->len)
#define sdsavail(s)  (sdsalloc(s) - sdslen(s))

/* 문자열 추가 시 공간 부족하면 2배 확장 (1MB 미만),
 * 1MB 이상이면 1MB씩 증가 (성장 전략) */
sds sdsMakeRoomFor(sds s, size_t addlen) {
    size_t avail = sdsavail(s);
    if (avail >= addlen) return s;  /* 이미 충분한 공간 */

    size_t len = sdslen(s);
    size_t newlen = len + addlen;
    if (newlen < SDS_MAX_PREALLOC)      /* 1MB 미만 */
        newlen *= 2;                     /* 2배 증가 */
    else
        newlen += SDS_MAX_PREALLOC;      /* 1MB씩 증가 */

    /* 새로운 크기에 맞는 헤더 타입 선택 후 realloc */
    /* ... */
}
```

**SDS 사용의 장점 시뮬레이션 (Python으로 개념 검증)**:

```python
# SDS의 핵심 아이디어를 Python으로 표현

class SDS:
    """Simple Dynamic String 개념 구현"""
    
    def __init__(self, s: str = ""):
        self._buf = bytearray(s.encode())
        self._len = len(self._buf)
        self._alloc = self._len  # 처음에는 딱 맞게
    
    @property
    def len(self) -> int:
        return self._len  # O(1) 길이 반환
    
    def append(self, s: str) -> None:
        data = s.encode()
        add_len = len(data)
        
        # 공간 확보 (growth strategy)
        if self._alloc - self._len < add_len:
            new_alloc = self._len + add_len
            if new_alloc < 1024 * 1024:  # 1MB 미만
                new_alloc *= 2
            else:
                new_alloc += 1024 * 1024
            # realloc에 해당하는 동작
            new_buf = bytearray(new_alloc)
            new_buf[:self._len] = self._buf[:self._len]
            self._buf = new_buf
            self._alloc = new_alloc
        
        # 데이터 추가
        self._buf[self._len:self._len + add_len] = data
        self._len += add_len
    
    def to_str(self) -> str:
        return self._buf[:self._len].decode()
    
    def __repr__(self):
        return (f"SDS(len={self._len}, alloc={self._alloc}, "
                f"free={self._alloc - self._len}, "
                f"content={self.to_str()!r})")


# 데모
s = SDS("Hello")
print(s)
# SDS(len=5, alloc=5, free=0, content='Hello')

s.append(", World!")
print(s)
# SDS(len=13, alloc=26, free=13, content='Hello, World!')
# → 2배 할당으로 잦은 realloc 방지

# 바이너리 안전: \x00도 저장 가능
s2 = SDS()
s2._buf = bytearray(b"bin\x00ary")
s2._len = 7
print(f"바이너리 길이: {s2.len}")  # 7 (올바름, C strlen은 3을 반환)
```

---

### 예제 2: Skiplist 구현과 Redis Sorted Set

Redis의 `ZADD`/`ZRANGE`/`ZRANGEBYSCORE` 같은 명령은 내부적으로 **스킵 리스트(skiplist)**를 사용한다. 스킵 리스트는 레드-블랙 트리보다 구현이 단순하면서도 평균 O(log n) 성능을 제공한다.

Redis의 스킵 리스트는 표준 스킵 리스트에서 두 가지를 추가했다:
1. **backward 포인터**: 역방향 순회(`ZREVRANGE`)를 위해 각 노드에 이전 노드 포인터 추가
2. **span**: 각 레벨의 forward 포인터가 몇 개의 노드를 건너뛰는지 기록 → `ZRANK` O(log n) 구현

```c
/* Redis의 zskiplistNode 구조 (t_zset.c) */
typedef struct zskiplistNode {
    sds ele;                  /* 멤버 문자열 (SDS) */
    double score;             /* 정렬 기준 점수 */
    struct zskiplistNode *backward; /* 역방향 포인터 */
    struct zskiplistLevel {
        struct zskiplistNode *forward; /* 다음 노드 */
        unsigned long span;            /* 건너뛴 노드 수 */
    } level[];                /* 레벨 배열 (가변 길이) */
} zskiplistNode;

typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;    /* 노드 수 */
    int level;               /* 현재 최대 레벨 */
} zskiplist;
```

**Python으로 Redis 스킵 리스트 핵심 로직 구현**:

```python
import random
import math
from dataclasses import dataclass, field
from typing import Optional, List

MAX_LEVEL = 32
P = 0.25  # Redis의 skiplist 확률 (표준 0.5보다 낮아 메모리 절약)


@dataclass
class SkiplistNode:
    score: float
    ele: str
    backward: Optional['SkiplistNode'] = field(default=None, repr=False)
    forward: List[Optional['SkiplistNode']] = field(default_factory=list, repr=False)
    span: List[int] = field(default_factory=list, repr=False)


class RedisSkiplist:
    def __init__(self):
        self.header = SkiplistNode(score=-math.inf, ele="")
        self.header.forward = [None] * MAX_LEVEL
        self.header.span = [0] * MAX_LEVEL
        self.tail = None
        self.level = 1
        self.length = 0

    def _random_level(self) -> int:
        """랜덤 레벨 결정: 확률 P로 레벨 증가"""
        level = 1
        while random.random() < P and level < MAX_LEVEL:
            level += 1
        return level

    def insert(self, score: float, ele: str) -> SkiplistNode:
        """O(log n) 삽입"""
        update = [None] * MAX_LEVEL
        rank = [0] * MAX_LEVEL

        x = self.header
        for i in range(self.level - 1, -1, -1):
            rank[i] = rank[i + 1] if i < self.level - 1 else 0
            while (x.forward[i] is not None and
                   (x.forward[i].score < score or
                    (x.forward[i].score == score and x.forward[i].ele < ele))):
                rank[i] += x.span[i]
                x = x.forward[i]
            update[i] = x

        new_level = self._random_level()
        if new_level > self.level:
            for i in range(self.level, new_level):
                rank[i] = 0
                update[i] = self.header
                update[i].span[i] = self.length
            self.level = new_level

        x = SkiplistNode(score=score, ele=ele)
        x.forward = [None] * new_level
        x.span = [0] * new_level

        for i in range(new_level):
            x.forward[i] = update[i].forward[i]
            update[i].forward[i] = x
            # span 업데이트: 건너뛴 노드 수 기록
            x.span[i] = update[i].span[i] - (rank[0] - rank[i])
            update[i].span[i] = (rank[0] - rank[i]) + 1

        # backward 포인터 설정 (ZREVRANGE용)
        x.backward = None if update[0] is self.header else update[0]
        if x.forward[0]:
            x.forward[0].backward = x
        else:
            self.tail = x

        self.length += 1
        return x

    def get_rank(self, score: float, ele: str) -> int:
        """O(log n) 랭크 조회 - span 덕분에 가능"""
        rank = 0
        x = self.header
        for i in range(self.level - 1, -1, -1):
            while (x.forward[i] is not None and
                   (x.forward[i].score < score or
                    (x.forward[i].score == score and x.forward[i].ele <= ele))):
                rank += x.span[i]
                x = x.forward[i]
            if x.ele == ele:
                return rank
        return -1  # 미존재

    def range_by_score(self, min_score: float, max_score: float) -> List[str]:
        """O(log n + k) 범위 조회 - ZRANGEBYSCORE 구현"""
        result = []
        x = self.header
        for i in range(self.level - 1, -1, -1):
            while x.forward[i] and x.forward[i].score < min_score:
                x = x.forward[i]
        x = x.forward[0]
        while x and x.score <= max_score:
            result.append((x.ele, x.score))
            x = x.forward[0]
        return result


# 데모
sl = RedisSkiplist()
data = [("Alice", 100.0), ("Bob", 200.0), ("Charlie", 150.0),
        ("Dave", 200.0), ("Eve", 50.0)]

for name, score in data:
    sl.insert(score, name)

print(f"총 멤버 수: {sl.length}")  # 5
print(f"Bob의 랭크: {sl.get_rank(200.0, 'Bob')}")  # 4
print(f"100~200점 범위: {sl.range_by_score(100, 200)}")
# [('Alice', 100.0), ('Charlie', 150.0), ('Bob', 200.0), ('Dave', 200.0)]
```

---

### Redis 인코딩 전환 임계값

```
# Redis CLI로 직접 확인하는 방법
redis-cli> ZADD myset 1 "one" 2 "two" 3 "three"
(integer) 3
redis-cli> OBJECT ENCODING myset
"listpack"   ← 128개 이하, 64바이트 이하이므로 listpack 사용

# 128개 초과하거나 64바이트 초과하면 자동 변환
redis-cli> ZADD myset 4 "four-very-long-string-exceeding-64bytes-1234567890abcdefghij"
redis-cli> OBJECT ENCODING myset
"skiplist"   ← 자동으로 skiplist+hashtable로 전환

# Hash 타입의 인코딩 전환
redis-cli> HSET myhash field1 val1
redis-cli> OBJECT ENCODING myhash
"listpack"   ← 128개 이하, 64바이트 이하

# List 타입
redis-cli> RPUSH mylist a b c
redis-cli> OBJECT ENCODING mylist
"listpack"   ← 512개 이하, 64바이트 이하 (Redis 7.0+)
```

**인코딩 전환 임계값 (redis.conf)**:
```
# Sorted Set
zset-max-listpack-entries 128   # 엔트리 수 임계값
zset-max-listpack-value 64      # 값 길이 임계값 (바이트)

# Hash
hash-max-listpack-entries 128
hash-max-listpack-value 64

# List
list-max-listpack-size 128
list-max-ziplist-size 8192      # quicklist 노드당 최대 크기

# Set
set-max-intset-entries 512      # intset 최대 크기
set-max-listpack-entries 128
```

---

## 주의사항 및 팁

### 1. listpack vs. ziplist 혼동 주의
Redis 7.0부터 `ziplist`는 `listpack`으로 대체되었다. 주요 차이는 listpack이 이전 항목으로의 역방향 포인터를 제거하여 메모리를 절약하고 캐스케이드 업데이트(cascade update) 문제를 해결했다. RDB 파일 역호환성을 위해 내부적으로 ziplist 디코딩은 여전히 지원된다.

### 2. 인코딩 임계값 튜닝
기본값은 일반적인 워크로드에 최적화되어 있지만, 특정 사용 패턴에서는 조정이 필요할 수 있다:
- **메모리 우선**: 임계값을 낮게 설정하면 listpack 사용 증가 → 메모리 절약, CPU 증가
- **성능 우선**: 임계값을 높게 설정하면 skiplist/hashtable 빠른 전환 → CPU 절약, 메모리 증가

### 3. quicklist의 fill factor
quicklist는 내부적으로 listpack 노드들의 이중 연결 리스트다. 각 노드의 크기는 `list-max-listpack-size`로 제어하며, 양수이면 엔트리 수, 음수이면 바이트 크기 제한(`-1`=4KB, `-2`=8KB, `-3`=16KB, `-4`=32KB, `-5`=64KB)이다. `-2`(8KB)가 기본값으로, 대부분의 경우 최적이다.

### 4. OBJECT ENCODING으로 실시간 모니터링
```bash
# 메모리 사용량과 인코딩을 함께 확인
redis-cli> OBJECT ENCODING key
redis-cli> OBJECT IDLETIME key  # 마지막 접근 이후 초
redis-cli> MEMORY USAGE key     # 바이트 단위 메모리 사용량
```

### 5. AOF rewrite와 RDB 저장 시 인코딩
AOF와 RDB 파일은 내부 인코딩이 아닌 논리적 명령/데이터를 저장한다. 따라서 Redis 버전이 달라도 데이터 이식성이 보장된다. 단, RDB 버전 간 호환성은 `rdbchecksum` 설정으로 관리된다.

### 6. HyperLogLog와 Stream의 내부
- **HyperLogLog**: 문자열 타입으로 저장되지만 내부적으로 희소(sparse)와 밀집(dense) 두 가지 표현 사용. 12KB 이하의 고정 메모리로 수억 개의 고유 값을 0.81% 오차로 추정
- **Stream**: Radix Tree(Trie의 변형) 위에 listpack 노드가 붙은 복합 구조로, 시계열 메시지를 압축 저장

## 참고 자료
- [Redis Official Docs: Data Types](https://redis.io/docs/latest/develop/data-types/)
- [Redis Internals: Data Structures Deep Dive (Calmops)](https://calmops.com/database/redis/redis-internals/)
- [DeepWiki: Redis Compact Data Structures](https://deepwiki.com/redis/redis/3.4-compact-data-structures)
- [Redis Source Code: src/sds.h, src/t_zset.c (GitHub)](https://github.com/redis/redis)
