---
layout: post
title: "스킵 리스트(Skip List) 완전 정복: Redis가 선택한 확률적 자료구조"
date: 2026-06-20
categories: [cs, computer-science]
tags: [skip-list, data-structure, algorithm, redis, probabilistic, linked-list]
---

## 스킵 리스트란 무엇인가?

스킵 리스트(Skip List)는 1990년 William Pugh가 고안한 확률적(probabilistic) 자료구조입니다. 정렬된 연결 리스트(linked list)를 기반으로 하되, 여러 단계(레벨)의 레인(lane)을 두어 탐색 속도를 O(log n)으로 끌어올리는 것이 핵심 아이디어입니다. AVL 트리나 레드-블랙 트리 같은 균형 이진 탐색 트리(balanced BST)와 동일한 점근적 성능을 내면서도 **구현이 훨씬 단순**하고 **동시성(concurrency) 친화적**이라는 장점 때문에 Redis의 Sorted Set, LevelDB, MemSQL 등 다양한 프로덕션 시스템에서 채택하고 있습니다.

### 구조 이해: 레인(Lane)의 개념

스킵 리스트의 가장 아래 레벨(Level 0)은 모든 요소를 포함하는 일반적인 정렬 연결 리스트입니다. 그 위에 더 높은 레벨들이 쌓이는데, 각 레벨은 확률 p(보통 0.5)에 따라 아래 레벨 원소의 절반 정도만 포함합니다.

```
Level 3: -∞ ────────────────── 30 ────────── +∞
Level 2: -∞ ────── 10 ──────── 30 ── 50 ──── +∞
Level 1: -∞ ── 5 ── 10 ── 20 ── 30 ── 50 ── +∞
Level 0: -∞ ── 5 ── 10 ── 15 ── 20 ── 25 ── 30 ── 40 ── 50 ── +∞
```

탐색 시 가장 높은 레벨에서 시작해 목표 값에 가까워질수록 낮은 레벨로 내려오면서 불필요한 노드를 건너뜁니다. 이 덕분에 순차 탐색의 O(n)이 아니라 O(log n)에 원하는 원소를 찾을 수 있습니다.

---

## 왜 스킵 리스트가 필요한가?

### AVL / 레드-블랙 트리의 한계

균형 이진 탐색 트리는 삽입·삭제 시 **회전(rotation)** 연산을 통해 균형을 유지합니다. 회전은 노드 포인터를 복잡하게 갱신하므로, 멀티스레드 환경에서 락(lock) 없이 구현하기가 매우 어렵습니다. 전체 서브트리를 잠가야 할 수 있어 동시성 성능이 저하됩니다.

### 스킵 리스트의 강점

1. **구현 단순성**: 삽입/삭제 시 회전이 없고 전임 노드(predecessor) 포인터만 관리하면 됩니다.
2. **Lock-Free 친화성**: CAS(Compare-And-Swap) 한 번으로 포인터를 교체할 수 있어 비차단(non-blocking) 동시 자료구조 구현이 용이합니다.
3. **순서 보장**: 정렬 순서가 항상 유지되므로 범위 쿼리(range query)에 최적입니다.
4. **캐시 지역성**: 연속적인 메모리 접근 패턴이 BST보다 캐시에 유리한 경우가 있습니다.

Redis가 Sorted Set에 스킵 리스트를 사용하는 이유도 바로 이것입니다. `ZRANGEBYSCORE`, `ZRANK` 같은 범위 연산이 O(log n + k)에 동작하며, 코드 유지보수가 쉽습니다.

---

## 실제 구현 예제

### 예제 1: Python으로 스킵 리스트 구현

```python
import random
import math

MAX_LEVEL = 16
P = 0.5

class SkipNode:
    def __init__(self, key, value, level):
        self.key = key
        self.value = value
        # forward[i]는 레벨 i에서 다음 노드를 가리키는 포인터
        self.forward = [None] * (level + 1)

class SkipList:
    def __init__(self):
        self.max_level = MAX_LEVEL
        self.level = 0
        # 헤드 노드: 키를 -∞로 설정 (실제론 None)
        self.head = SkipNode(None, None, MAX_LEVEL)

    def _random_level(self):
        """확률 p에 따라 노드의 레벨을 무작위로 결정"""
        lvl = 0
        while random.random() < P and lvl < self.max_level:
            lvl += 1
        return lvl

    def insert(self, key, value):
        # update[i]: 레벨 i에서 새 노드 바로 이전에 위치할 노드
        update = [None] * (self.max_level + 1)
        current = self.head

        for i in range(self.level, -1, -1):
            while current.forward[i] and current.forward[i].key < key:
                current = current.forward[i]
            update[i] = current

        # 이미 존재하는 키면 값 업데이트
        if current.forward[0] and current.forward[0].key == key:
            current.forward[0].value = value
            return

        new_level = self._random_level()
        if new_level > self.level:
            for i in range(self.level + 1, new_level + 1):
                update[i] = self.head
            self.level = new_level

        new_node = SkipNode(key, value, new_level)
        for i in range(new_level + 1):
            new_node.forward[i] = update[i].forward[i]
            update[i].forward[i] = new_node

    def search(self, key):
        current = self.head
        for i in range(self.level, -1, -1):
            while current.forward[i] and current.forward[i].key < key:
                current = current.forward[i]

        current = current.forward[0]
        if current and current.key == key:
            return current.value
        return None

    def delete(self, key):
        update = [None] * (self.max_level + 1)
        current = self.head

        for i in range(self.level, -1, -1):
            while current.forward[i] and current.forward[i].key < key:
                current = current.forward[i]
            update[i] = current

        target = current.forward[0]
        if target and target.key == key:
            for i in range(self.level + 1):
                if update[i].forward[i] != target:
                    break
                update[i].forward[i] = target.forward[i]
            while self.level > 0 and self.head.forward[self.level] is None:
                self.level -= 1
            return True
        return False

    def display(self):
        for i in range(self.level, -1, -1):
            node = self.head.forward[i]
            print(f"Level {i}: ", end="")
            while node:
                print(f"{node.key}({node.value})", end=" -> ")
                node = node.forward[i]
            print("None")
```

### 예제 2: 스킵 리스트 동작 시연 및 성능 측정

```python
import time
import random

def benchmark_skip_list():
    sl = SkipList()
    n = 100_000

    # 삽입 벤치마크
    keys = list(range(n))
    random.shuffle(keys)

    start = time.perf_counter()
    for k in keys:
        sl.insert(k, f"value_{k}")
    insert_time = time.perf_counter() - start
    print(f"삽입 {n:,}개 완료: {insert_time:.3f}초")

    # 검색 벤치마크
    search_keys = random.sample(keys, 1000)
    start = time.perf_counter()
    hits = sum(1 for k in search_keys if sl.search(k) is not None)
    search_time = time.perf_counter() - start
    print(f"검색 1,000개 완료: {search_time:.6f}초 (적중 {hits}/1000)")

    # 삭제 벤치마크
    delete_keys = random.sample(keys, 10_000)
    start = time.perf_counter()
    deleted = sum(1 for k in delete_keys if sl.delete(k))
    delete_time = time.perf_counter() - start
    print(f"삭제 10,000개 완료: {delete_time:.3f}초 (실제 삭제 {deleted}개)")

    # 구조 확인 (소규모)
    mini_sl = SkipList()
    for v in [3, 6, 7, 9, 12, 19, 21, 25]:
        mini_sl.insert(v, str(v))
    print("\n--- 스킵 리스트 구조 ---")
    mini_sl.display()

if __name__ == "__main__":
    benchmark_skip_list()
```

실행 결과 예시:
```
삽입 100,000개 완료: 0.412초
검색 1,000개 완료: 0.000231초 (적중 1000/1000)
삭제 10,000개 완료: 0.041초 (실제 삭제 10000개)

--- 스킵 리스트 구조 ---
Level 3: 19(19) -> None
Level 2: 6(6) -> 19(19) -> 25(25) -> None
Level 1: 3(3) -> 6(6) -> 12(12) -> 19(19) -> 25(25) -> None
Level 0: 3(3) -> 6(6) -> 7(7) -> 9(9) -> 12(12) -> 19(19) -> 21(21) -> 25(25) -> None
```

---

## 시간/공간 복잡도

| 연산 | 평균 | 최악 |
|------|------|------|
| 탐색 | O(log n) | O(n) |
| 삽입 | O(log n) | O(n) |
| 삭제 | O(log n) | O(n) |
| 공간 | O(n log n) | O(n log n) |

최악의 경우 O(n)이 가능하지만, 이는 랜덤 레벨 생성이 극단적으로 편향될 때만 발생하며, 그 확률은 지수적으로 작습니다. 실용적으로는 O(log n)을 보장한다고 봐도 무방합니다.

---

## 주의사항 및 실전 팁

### 1. 확률 파라미터 p 선택

p = 0.5가 가장 일반적이지만, 검색 위주 워크로드에서는 p = 0.25를 쓰면 공간을 절약하면서 성능을 유지할 수 있습니다. Redis는 실험을 통해 p = 0.25를 채택했습니다.

### 2. 최대 레벨(MAX_LEVEL) 설정

최적 MAX_LEVEL은 `log_{1/p}(n)`입니다. 예상 원소 수가 100만 개이고 p = 0.5이면 `log₂(1,000,000) ≈ 20`이 적절합니다.

### 3. 동시성 처리

멀티스레드 환경에서는 각 레벨의 포인터를 CAS 연산으로 원자적(atomic)으로 교체하면 Lock-Free 스킵 리스트를 구현할 수 있습니다. Java의 `ConcurrentSkipListMap`이 이 방식을 사용합니다.

### 4. 스킵 리스트 vs 레드-블랙 트리

메모리 사용량은 스킵 리스트가 약간 더 크지만, 범위 쿼리와 동시성 친화성에서 스킵 리스트가 우위를 가집니다. 단순 키-값 조회만 필요하다면 해시 테이블이 더 빠릅니다.

---

## 참고 자료

- [Wikipedia - Skip List](https://en.wikipedia.org/wiki/Skip_list)
- [Baeldung CS - The Skip List Data Structure](https://www.baeldung.com/cs/skip-lists)
- [OpenDSA - Skip Lists](https://opendsa-server.cs.vt.edu/ODSA/Books/CS3/html/SkipList.html)
- [William Pugh 원본 논문 (1990)](https://www.epaperpress.com/sortsearch/download/skiplist.pdf)
