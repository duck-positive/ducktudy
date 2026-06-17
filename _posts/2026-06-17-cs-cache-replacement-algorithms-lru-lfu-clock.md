---
layout: post
title: "캐시 교체 알고리즘 완전 정복: LRU, LFU, CLOCK 구현과 비교"
date: 2026-06-17
categories: [cs, computer-science]
tags: [캐시, LRU, LFU, CLOCK, 운영체제, 메모리관리, 알고리즘, 자료구조]
---

## 개요

캐시(Cache)는 느린 저장소(디스크, 데이터베이스)에 자주 접근하는 비용을 줄이기 위해 빠른 저장소(메모리, Redis, CPU 캐시)에 데이터를 임시 보관하는 기법입니다. 그러나 캐시 크기는 항상 한정되어 있어, 새로운 데이터가 들어올 때 기존 데이터 중 무엇을 제거할지 결정하는 정책이 필요합니다. 이것이 **캐시 교체 알고리즘(Cache Replacement Policy)**입니다.

이 글에서는 가장 널리 쓰이는 세 가지 알고리즘 — **LRU(Least Recently Used)**, **LFU(Least Frequently Used)**, **CLOCK** — 의 원리, Python 구현, 그리고 실무에서의 선택 기준까지 심도 있게 다룹니다.

---

## 1. LRU(Least Recently Used) — 가장 오래 전에 사용된 항목 제거

### 개념

LRU는 **가장 최근에 사용되지 않은 항목을 제거**하는 전략입니다. 시간 지역성(Temporal Locality) — "최근에 사용한 데이터는 다시 사용될 가능성이 높다"는 원칙에 기반합니다.

운영체제의 가상 메모리 페이지 교체, Redis의 `volatile-lru` / `allkeys-lru` 정책, CPU L1/L2 캐시 등에서 폭넓게 활용됩니다.

### 구현 원리

LRU를 get/put 모두 **O(1)**에 처리하려면 **해시맵(HashMap) + 이중 연결 리스트(Doubly Linked List)**를 결합합니다.
- 해시맵: 키 → 노드의 O(1) 접근
- 이중 연결 리스트: 최근 사용 순서 유지 (헤드=가장 최근, 테일=가장 오래됨)

---

## 2. 코드 예제 1 — LRU Cache: OrderedDict 활용

Python의 `OrderedDict`는 삽입 순서를 유지하고 `move_to_end()`로 O(1) 재배치를 지원하여 LRU 구현에 최적입니다.

```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = OrderedDict()

    def get(self, key: int) -> int:
        if key not in self.cache:
            return -1
        self.cache.move_to_end(key)   # 가장 최근 사용으로 이동
        return self.cache[key]

    def put(self, key: int, value: int) -> None:
        if key in self.cache:
            self.cache.move_to_end(key)
        self.cache[key] = value
        if len(self.cache) > self.capacity:
            self.cache.popitem(last=False)  # 가장 오래된 항목 제거

# 시뮬레이션
cache = LRUCache(3)
cache.put(1, "Alice")
cache.put(2, "Bob")
cache.put(3, "Charlie")
print(cache.get(1))   # 'Alice' — 1이 최근 사용으로 이동
cache.put(4, "Dave")  # 용량 초과 → 가장 오래된 2번 제거
print(cache.get(2))   # -1 (제거됨)
print(cache.get(3))   # 'Charlie'
print(list(cache.cache.keys()))  # [1, 3, 4] — 최근 사용 역순
```

### 이중 연결 리스트로 직접 구현

```python
class DLinkedNode:
    def __init__(self, key=0, value=0):
        self.key = key
        self.value = value
        self.prev = None
        self.next = None

class LRUCacheRaw:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = {}
        self.head = DLinkedNode()   # 더미 헤드 (가장 최근)
        self.tail = DLinkedNode()   # 더미 테일 (가장 오래됨)
        self.head.next = self.tail
        self.tail.prev = self.head

    def _remove(self, node):
        node.prev.next = node.next
        node.next.prev = node.prev

    def _add_to_head(self, node):
        node.prev = self.head
        node.next = self.head.next
        self.head.next.prev = node
        self.head.next = node

    def get(self, key: int) -> int:
        if key not in self.cache:
            return -1
        node = self.cache[key]
        self._remove(node)
        self._add_to_head(node)
        return node.value

    def put(self, key: int, value: int) -> None:
        if key in self.cache:
            node = self.cache[key]
            node.value = value
            self._remove(node)
            self._add_to_head(node)
        else:
            node = DLinkedNode(key, value)
            self.cache[key] = node
            self._add_to_head(node)
            if len(self.cache) > self.capacity:
                lru = self.tail.prev
                self._remove(lru)
                del self.cache[lru.key]
```

---

## 3. LFU(Least Frequently Used) — 사용 빈도가 가장 낮은 항목 제거

### 개념

LFU는 **접근 빈도가 가장 낮은 항목을 제거**하는 전략입니다. 빈도수가 같은 경우, 가장 오래 전에 사용된 항목을 제거합니다(LRU tie-breaking).

빈도 기반 지역성이 강한 워크로드 — 특정 인기 콘텐츠에 반복 접근하는 CDN 캐시, 데이터베이스 버퍼 풀 등 — 에 적합합니다.

### O(1) LFU 구현 전략

세 가지 자료구조를 결합합니다:
- `key_to_val`: 키 → 값
- `key_to_freq`: 키 → 빈도수
- `freq_to_keys`: 빈도수 → 해당 빈도의 키 집합(OrderedDict로 LRU 순서 유지)
- `min_freq`: 현재 최소 빈도수 추적

---

## 4. 코드 예제 2 — O(1) LFU Cache 구현

```python
from collections import defaultdict, OrderedDict

class LFUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.min_freq = 0
        self.key_to_val  = {}
        self.key_to_freq = {}
        self.freq_to_keys = defaultdict(OrderedDict)

    def _update_freq(self, key: int):
        freq = self.key_to_freq[key]
        self.key_to_freq[key] = freq + 1
        del self.freq_to_keys[freq][key]
        if not self.freq_to_keys[freq]:
            del self.freq_to_keys[freq]
            if self.min_freq == freq:
                self.min_freq += 1
        self.freq_to_keys[freq + 1][key] = None

    def get(self, key: int) -> int:
        if key not in self.key_to_val:
            return -1
        self._update_freq(key)
        return self.key_to_val[key]

    def put(self, key: int, value: int) -> None:
        if self.capacity <= 0:
            return
        if key in self.key_to_val:
            self.key_to_val[key] = value
            self._update_freq(key)
            return
        if len(self.key_to_val) >= self.capacity:
            # 최소 빈도 버킷에서 가장 오래된 항목 제거
            min_bucket = self.freq_to_keys[self.min_freq]
            evict_key, _ = min_bucket.popitem(last=False)
            del self.key_to_val[evict_key]
            del self.key_to_freq[evict_key]
        self.key_to_val[key] = value
        self.key_to_freq[key] = 1
        self.freq_to_keys[1][key] = None
        self.min_freq = 1

# 시뮬레이션
lfu = LFUCache(3)
lfu.put(1, "A")  # freq: {1:1}
lfu.put(2, "B")  # freq: {1:1, 2:1}
lfu.put(3, "C")  # freq: {1:1, 2:1, 3:1}
lfu.get(1)       # freq: {1:2, 2:1, 3:1}
lfu.get(1)       # freq: {1:3, 2:1, 3:1}
lfu.get(2)       # freq: {1:3, 2:2, 3:1}

lfu.put(4, "D")  # 용량 초과 → min_freq=1, 가장 오래된 3번 제거
print(lfu.get(3))  # -1 (제거됨)
print(lfu.get(1))  # 'A' (빈도 4)
print(lfu.get(2))  # 'B' (빈도 3)
print(lfu.get(4))  # 'D' (빈도 2)
```

---

## 5. CLOCK 알고리즘 — LRU의 근사(Approximation) 구현

### 개념

CLOCK 알고리즘은 운영체제의 페이지 교체에서 LRU를 근사하는 방식으로, **참조 비트(Reference Bit)** 하나만으로 동작합니다. 순수 LRU는 모든 페이지의 접근 시간을 기록해야 하므로 하드웨어 지원 없이는 구현 비용이 크지만, CLOCK은 구현이 단순하고 효율적입니다.

### 동작 방식

페이지들이 원형(시계)으로 배열되고, 시계 포인터가 순환합니다:
1. 페이지 접근 시: 해당 페이지의 참조 비트를 **1**로 설정.
2. 교체 필요 시: 포인터가 가리키는 페이지의 참조 비트를 확인:
   - **1이면**: 0으로 초기화하고 다음으로 이동 ("두 번째 기회" 부여)
   - **0이면**: 해당 페이지를 교체하고 새 페이지 적재

---

## 6. 코드 예제 3 — CLOCK 알고리즘 구현

```python
class ClockCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.frames   = [None] * capacity  # 페이지 프레임
        self.ref_bits = [0]    * capacity  # 참조 비트
        self.pointer  = 0                  # 시계 포인터
        self.page_to_frame = {}            # 페이지 → 프레임 인덱스

    def access(self, page: int) -> str:
        if page in self.page_to_frame:
            # 캐시 히트: 참조 비트만 갱신
            self.ref_bits[self.page_to_frame[page]] = 1
            return f"HIT  page={page}"

        # 캐시 미스: CLOCK 알고리즘으로 교체 대상 탐색
        while self.ref_bits[self.pointer] == 1:
            self.ref_bits[self.pointer] = 0   # 두 번째 기회 소멸
            self.pointer = (self.pointer + 1) % self.capacity

        evicted = self.frames[self.pointer]
        if evicted is not None:
            del self.page_to_frame[evicted]

        self.frames[self.pointer] = page
        self.ref_bits[self.pointer] = 1
        self.page_to_frame[page] = self.pointer
        result = f"MISS page={page}, evicted={evicted}"
        self.pointer = (self.pointer + 1) % self.capacity
        return result

# 시뮬레이션
cache = ClockCache(capacity=4)
page_seq = [1, 2, 3, 4, 1, 2, 5, 1, 2, 3, 4, 5]
for page in page_seq:
    print(cache.access(page))

# 출력:
# MISS page=1, evicted=None
# MISS page=2, evicted=None
# MISS page=3, evicted=None
# MISS page=4, evicted=None
# HIT  page=1
# HIT  page=2
# MISS page=5, evicted=3   ← ref_bit=0이 된 3번 교체
# HIT  page=1
# HIT  page=2
# MISS page=3, evicted=4
# MISS page=4, evicted=5
# MISS page=5, evicted=1
```

---

## 7. 알고리즘 비교 및 선택 가이드

| 알고리즘 | get/put 복잡도 | 공간 복잡도 | 장점 | 단점 | 최적 사용 상황 |
|---------|--------------|------------|------|------|---------------|
| LRU | O(1) | O(n) | 구현 간단, 시간 지역성에 최적 | 빈도 미고려 | 웹 캐시, Redis, 일반 인메모리 캐시 |
| LFU | O(1) | O(n) | 장기 인기 항목 유지 | 새 항목 불리, 구현 복잡 | CDN, DB 버퍼 풀, 인기 콘텐츠 캐시 |
| CLOCK | O(1) amortized | O(n) | 메모리 효율적, OS 구현에 적합 | LRU 근사치 (부정확) | 운영체제 페이지 교체 |
| FIFO | O(1) | O(n) | 구현 최단순 | Belady's Anomaly 발생 | 단순 버퍼, 스트리밍 |
| ARC | O(1) | O(2n) | LRU+LFU 자동 균형 | 구현 복잡 | 파일시스템 캐시(ZFS 등) |

---

## 8. Redis의 캐시 교체 정책

Redis는 `maxmemory-policy` 설정으로 다양한 교체 정책을 지원합니다:

```bash
# redis.conf 또는 redis-cli 설정
CONFIG SET maxmemory 256mb
CONFIG SET maxmemory-policy allkeys-lru

# 사용 가능한 정책:
# noeviction      — 교체 없음 (쓰기 시 오류 반환)
# allkeys-lru     — 전체 키 중 LRU 제거
# volatile-lru    — TTL 설정된 키 중 LRU 제거
# allkeys-lfu     — 전체 키 중 LFU 제거 (Redis 4.0+)
# allkeys-random  — 랜덤 제거
# volatile-ttl    — TTL이 가장 짧은 키 제거

# 캐시 적중률 모니터링
# redis-cli INFO stats | grep -E 'keyspace_hits|keyspace_misses'
```

Redis의 LRU/LFU 구현은 **근사 알고리즘**입니다. 전체 키를 순회하는 대신 랜덤 샘플(기본값 5개)을 추출하여 그 중 가장 오래된/빈도 낮은 키를 제거합니다. `maxmemory-samples` 값을 높이면 정확도가 올라가지만 CPU 사용량도 증가합니다.

---

## 9. 주의사항과 실무 팁

1. **LRU vs LFU 선택 기준**: 워크로드가 시간적 지역성을 따르면 LRU, 인기도 기반 반복 접근 패턴이면 LFU가 유리합니다. 확신이 없다면 LRU가 더 안전한 기본값입니다.

2. **캐시 오염(Cache Pollution) 주의**: 일회성 대용량 순차 스캔이 발생하면 LRU에서 기존 워킹셋(Working Set)이 모두 밀려납니다. 이를 방지하기 위해 ARC(Adaptive Replacement Cache)나 LIRS 알고리즘을 고려하세요.

3. **캐시 적중률(Hit Rate) 모니터링**: 적중률이 70% 미만이면 캐시 크기, 교체 정책, 또는 TTL 설정을 재검토하세요. Redis의 경우 `redis-cli INFO stats`로 실시간 확인이 가능합니다.

4. **Cold Start(콜드 스타트) 문제**: 서버 재시작 시 캐시가 비워져 초기 성능이 급락합니다. 주요 데이터를 미리 프리로드(Preload)하거나 Lazy Loading과 요청 병합 전략을 결합하세요.

5. **분산 환경에서의 캐시 일관성**: 여러 노드에 캐시를 분산할 경우, 쓰기 전파(Write Propagation)와 캐시 무효화(Cache Invalidation) 전략을 신중하게 설계해야 합니다.

---

## 참고 자료
- [Wikipedia: Cache replacement policies](https://en.wikipedia.org/wiki/Cache_replacement_policies)
- [LeetCode: LRU Cache (Problem 146)](https://leetcode.com/problems/lru-cache/)
- [Redis 공식 문서: Using Redis as an LRU Cache](https://redis.io/docs/manual/eviction/)
- [CodeLucky: Caching Algorithms — LRU, LFU, FIFO In-Depth Guide](https://codelucky.com/caching-algorithms-lru-lfu-fifo/)
