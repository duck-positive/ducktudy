---
layout: post
title: "Adaptive Radix Tree(ART) 완전 정복: 현대 인메모리 데이터베이스의 초고속 인덱스"
date: 2026-07-17
categories: [cs, computer-science]
tags: [adaptive-radix-tree, ART, trie, index, in-memory-database, data-structures]
---

## 개념 설명

데이터베이스 인덱스의 왕좌는 오랫동안 **B+트리**가 차지해 왔습니다. B+트리는 디스크 I/O를 최소화하도록 설계된 자료구조로, 페이지 크기(4KB~16KB)에 맞춰 다수의 키를 하나의 노드에 담아 I/O 횟수를 줄입니다. 그러나 메인 메모리가 수십~수백 GB에 달하는 현대 인메모리 데이터베이스 환경에서는 B+트리의 설계 철학이 오히려 **캐시 비효율**을 낳습니다. 노드 하나가 수백 개의 키를 담기 때문에 필요 없는 데이터까지 캐시에 올라오고, 노드 내 이진 탐색도 캐시 미스를 유발합니다.

**Adaptive Radix Tree(ART)**는 2013년 Viktor Leis, Alfons Kemper, Thomas Neumann이 ICDE에서 발표한 논문 "The Adaptive Radix Tree: ARTful Indexing for Main-Memory Databases"에서 소개된 인덱스 자료구조입니다. ART는 라딕스 트리(Radix Tree, 압축 트라이)의 공간 효율을 극적으로 개선하기 위해 **노드 크기를 자식 수에 따라 동적으로 조정**하는 것이 핵심입니다.

### 라딕스 트리의 문제점

일반 라딕스 트리는 각 노드가 256개(1바이트 청크 기준)의 자식 포인터 배열을 가집니다. 자식이 1개뿐인 노드도 2KB(포인터 8바이트 × 256)를 낭비합니다. ART는 이 문제를 네 가지 노드 타입으로 해결합니다.

### ART의 네 가지 노드 타입

| 노드 타입 | 최대 자식 수 | 내부 구조 | 크기 |
|-----------|------------|-----------|------|
| Node4 | 4 | 키 배열 4개 + 포인터 배열 4개 | 52B |
| Node16 | 16 | 키 배열 16개 + 포인터 배열 16개 (SIMD 탐색 가능) | 160B |
| Node48 | 48 | 인덱스 배열 256개(1B) + 포인터 배열 48개 | 656B |
| Node256 | 256 | 포인터 배열 256개 (직접 인덱싱) | 2KB |

- **Node4**: 선형 탐색 (자식 ≤ 4개)
- **Node16**: SIMD(SSE2) _mm_cmpestri 명령으로 16개를 한 번에 비교
- **Node48**: 256바이트 인덱스 테이블로 O(1) 조회, 포인터는 48개 슬롯
- **Node256**: 순수 배열 직접 접근, O(1)

노드는 자식 수가 임계치를 넘으면 **자동 확장(expand)**, 임계치 이하로 줄면 **자동 축소(shrink)**됩니다.

### 경로 압축(Path Compression)

라딕스 트리의 또 다른 비효율은 단일 자식만 가진 긴 체인입니다. ART는 두 가지 압축 기법을 사용합니다.

1. **Lazy Expansion**: 단일 자식 경로를 끝까지 압축하지 않고, 노드에 "남은 키의 일부"를 포인터와 함께 저장합니다. 삽입 시 충돌 발생 시점에만 노드를 분할합니다.
2. **Pessimistic Path Compression**: 각 노드에 압축된 경로의 최대 8바이트를 인라인으로 저장(partial key). 8바이트를 초과하면 원본 키를 참조합니다.

---

## 왜 필요한가

**인메모리 데이터베이스의 병목은 I/O가 아닌 CPU 캐시 미스**입니다. DRAM 접근 레이턴시는 ~100ns, L1 캐시는 ~1ns, L2는 ~5ns입니다. B+트리의 큰 노드는 캐시라인(64B)을 여러 개 걸쳐 읽어야 하므로 캐시 효율이 낮습니다.

ART의 장점:
- **캐시 친화적**: Node4, Node16은 단일 캐시라인 또는 2~3개로 수용
- **O(k) 조회**: k는 키의 길이(바이트 수). B+트리의 O(log N)과 달리 데이터 크기에 무관
- **공간 효율**: 해시 테이블 대비 정렬 순서 보존, B+트리 대비 공간 사용량 절감
- **범위 쿼리 지원**: 트라이 구조 덕분에 정렬된 순회가 자연스러움

HyPer(TU München), LeanStore, HyRise, DuckDB, TiKV(일부) 등 최신 인메모리 엔진들이 ART를 채택했습니다.

---

## 실제 구현 예제

### 예제 1: ART 노드 타입 및 삽입 구현 (Python)

```python
from dataclasses import dataclass, field
from typing import Optional

# 리프 노드
@dataclass
class LeafNode:
    key: bytes
    value: object

# 내부 노드 기반 클래스
class InnerNode:
    def __init__(self):
        self.partial: bytes = b""  # 경로 압축된 부분 키

    def find_child(self, byte: int) -> Optional['ArtNode']:
        raise NotImplementedError

    def add_child(self, byte: int, child: 'ArtNode'):
        raise NotImplementedError

ArtNode = object  # Union[InnerNode, LeafNode]


class Node4(InnerNode):
    MAX_CHILDREN = 4

    def __init__(self):
        super().__init__()
        self.keys: list[int] = []
        self.children: list[ArtNode] = []

    def find_child(self, byte: int) -> Optional[ArtNode]:
        for i, k in enumerate(self.keys):
            if k == byte:
                return self.children[i]
        return None

    def add_child(self, byte: int, child: ArtNode):
        # 정렬 삽입
        idx = next((i for i, k in enumerate(self.keys) if k > byte), len(self.keys))
        self.keys.insert(idx, byte)
        self.children.insert(idx, child)

    def is_full(self) -> bool:
        return len(self.keys) >= self.MAX_CHILDREN

    def grow(self) -> 'Node16':
        n16 = Node16()
        n16.partial = self.partial
        n16.keys = self.keys[:]
        n16.children = self.children[:]
        return n16


class Node16(InnerNode):
    MAX_CHILDREN = 16

    def __init__(self):
        super().__init__()
        self.keys: list[int] = []
        self.children: list[ArtNode] = []

    def find_child(self, byte: int) -> Optional[ArtNode]:
        # 실제 구현에서는 SIMD 사용; 여기서는 이진 탐색
        lo, hi = 0, len(self.keys)
        while lo < hi:
            mid = (lo + hi) // 2
            if self.keys[mid] == byte:
                return self.children[mid]
            elif self.keys[mid] < byte:
                lo = mid + 1
            else:
                hi = mid
        return None

    def add_child(self, byte: int, child: ArtNode):
        idx = next((i for i, k in enumerate(self.keys) if k > byte), len(self.keys))
        self.keys.insert(idx, byte)
        self.children.insert(idx, child)

    def is_full(self) -> bool:
        return len(self.keys) >= self.MAX_CHILDREN

    def grow(self) -> 'Node48':
        n48 = Node48()
        n48.partial = self.partial
        for i, (k, c) in enumerate(zip(self.keys, self.children)):
            n48.index[k] = i
            n48.children[i] = c
        n48.num_children = len(self.keys)
        return n48


class Node48(InnerNode):
    MAX_CHILDREN = 48

    def __init__(self):
        super().__init__()
        self.index = [-1] * 256   # byte -> slot index
        self.children = [None] * 48
        self.num_children = 0

    def find_child(self, byte: int) -> Optional[ArtNode]:
        slot = self.index[byte]
        return self.children[slot] if slot != -1 else None

    def add_child(self, byte: int, child: ArtNode):
        slot = self.num_children
        self.index[byte] = slot
        self.children[slot] = child
        self.num_children += 1

    def is_full(self) -> bool:
        return self.num_children >= self.MAX_CHILDREN


class Node256(InnerNode):
    def __init__(self):
        super().__init__()
        self.children = [None] * 256

    def find_child(self, byte: int) -> Optional[ArtNode]:
        return self.children[byte]

    def add_child(self, byte: int, child: ArtNode):
        self.children[byte] = child

    def is_full(self) -> bool:
        return False  # 항상 여유 있음


class ART:
    def __init__(self):
        self.root: Optional[ArtNode] = None

    def _check_prefix(self, node: InnerNode, key: bytes, depth: int) -> int:
        """노드의 partial key와 일치하는 바이트 수 반환"""
        mismatch = 0
        for i, b in enumerate(node.partial):
            if depth + i >= len(key) or key[depth + i] != b:
                break
            mismatch += 1
        return mismatch

    def search(self, key: bytes) -> Optional[object]:
        node = self.root
        depth = 0
        while node is not None:
            if isinstance(node, LeafNode):
                return node.value if node.key == key else None
            # 경로 압축 확인
            match_len = self._check_prefix(node, key, depth)
            if match_len < len(node.partial):
                return None
            depth += len(node.partial)
            if depth >= len(key):
                return None
            node = node.find_child(key[depth])
            depth += 1
        return None

    def insert(self, key: bytes, value: object):
        if self.root is None:
            self.root = LeafNode(key, value)
            return
        self._insert(self.root, key, value, 0, None, 0)

    def _insert(self, node, key, value, depth, parent, parent_byte):
        if isinstance(node, LeafNode):
            if node.key == key:
                node.value = value
                return
            # 리프 분할: 새 Node4 생성
            new_inner = Node4()
            # 공통 접두사 계산
            common = 0
            while depth + common < len(key) and depth + common < len(node.key):
                if key[depth + common] != node.key[depth + common]:
                    break
                common += 1
            new_inner.partial = key[depth:depth + common]
            new_depth = depth + common
            b1 = key[new_depth] if new_depth < len(key) else 256
            b2 = node.key[new_depth] if new_depth < len(node.key) else 256
            new_inner.add_child(b1, LeafNode(key, value))
            new_inner.add_child(b2, node)
            if parent is None:
                self.root = new_inner
            else:
                parent.children[parent.keys.index(parent_byte)] = new_inner
            return

        # 내부 노드: 경로 압축 매칭
        match_len = self._check_prefix(node, key, depth)
        if match_len < len(node.partial):
            # 분할 필요
            new_inner = Node4()
            new_inner.partial = node.partial[:match_len]
            old_byte = node.partial[match_len]
            node.partial = node.partial[match_len + 1:]
            new_byte = key[depth + match_len] if depth + match_len < len(key) else 256
            new_inner.add_child(old_byte, node)
            new_inner.add_child(new_byte, LeafNode(key, value))
            if parent is None:
                self.root = new_inner
            return

        depth += len(node.partial)
        if depth >= len(key):
            return
        b = key[depth]
        child = node.find_child(b)
        if child is None:
            if node.is_full():
                node = node.grow()
                if parent is None:
                    self.root = node
            node.add_child(b, LeafNode(key, value))
        else:
            self._insert(child, key, value, depth + 1, node, b)


# 사용 예시
art = ART()
data = [("apple", 1), ("application", 2), ("apt", 3), ("banana", 4), ("band", 5)]
for k, v in data:
    art.insert(k.encode(), v)

for k, _ in data:
    print(f"search('{k}') = {art.search(k.encode())}")
```

---

### 예제 2: ART vs B+트리 캐시 성능 비교 실험

```python
import time
import random
import sys

def bench_dict_vs_art(n: int = 100_000):
    """Python dict(해시테이블)과 ART 삽입/조회 성능 비교"""
    keys = [f"user:{i:010d}".encode() for i in range(n)]
    random.shuffle(keys)
    values = list(range(n))

    # Python dict 기준선
    d = {}
    t0 = time.perf_counter()
    for k, v in zip(keys, values):
        d[k] = v
    t1 = time.perf_counter()
    for k in keys[:10_000]:
        _ = d[k]
    t2 = time.perf_counter()
    print(f"dict  삽입 {n}개: {(t1-t0)*1000:.1f}ms, 조회 10k: {(t2-t1)*1000:.1f}ms")

    # ART
    art = ART()
    t0 = time.perf_counter()
    for k, v in zip(keys, values):
        art.insert(k, v)
    t1 = time.perf_counter()
    for k in keys[:10_000]:
        _ = art.search(k)
    t2 = time.perf_counter()
    print(f"ART   삽입 {n}개: {(t1-t0)*1000:.1f}ms, 조회 10k: {(t2-t1)*1000:.1f}ms")
    print("(순수 Python 구현으로 C++ ART의 성능을 직접 비교하기 어렵지만,")
    print(" 구조적 이해를 위한 참고 수치)")

bench_dict_vs_art(50_000)
```

실제 C++ 구현(Viktor Leis의 참조 구현)에서는 10억 건 정수 키 기준으로 B+트리 대비 **2~4배** 빠른 조회 성능을 보였습니다.

---

## 주의사항 및 팁

**1. 키의 비교 가능성**

ART는 키를 바이트 배열로 처리합니다. 정수형의 경우 빅엔디안 변환 후 저장해야 사전순(lexicographic) 순서가 숫자 순서와 일치합니다. `int32`라면 `struct.pack(">I", value)`로 변환합니다. 부동소수점은 NaN, ±Inf 처리와 부호 비트 반전이 필요합니다.

**2. 멀티버전 동시성 제어(MVCC)와의 통합**

Viktor Leis의 후속 논문(ARTful Synchronization, 2016)에서는 OLC(Optimistic Lock Coupling)와 ROWEX(Read-Optimized Write EXclusion) 두 가지 동시성 프로토콜을 제안합니다. 멀티코어 환경에서 락 경합을 최소화하는 핵심입니다.

**3. 삭제 연산의 복잡성**

삽입보다 삭제가 복잡합니다. 노드 축소(shrink) 로직과 경로 압축 재조정이 필요합니다. 실용 구현에서는 지연 삭제(lazy deletion, 마크만 하고 나중에 정리)를 병행합니다.

**4. 메모리 할당자의 영향**

Node4, Node16, Node48, Node256은 크기가 다릅니다. 고성능 구현에서는 크기별 메모리 풀(pool allocator)을 사용해 단편화를 줄이고 캐시 지역성을 높입니다. jemalloc이나 TCMalloc과의 조합이 효과적입니다.

**5. 가변 길이 키 vs 고정 길이 키**

고정 길이 키(UUID, 타임스탬프 등)는 경로 압축 없이도 트리 깊이가 일정하여 성능이 매우 안정적입니다. 가변 길이 문자열 키는 경로 압축이 핵심이며, 패딩(null terminator)을 올바르게 처리해야 합니다.

---

## 참고 자료

- [The Adaptive Radix Tree: ARTful Indexing for Main-Memory Databases (원본 논문)](https://www.db.in.tum.de/~leis/papers/ART.pdf)
- [Radix tree - Wikipedia](https://en.wikipedia.org/wiki/Radix_tree)
- [ARTful Synchronization Paper - Viktor Leis et al.](https://db.in.tum.de/~leis/papers/artsync.pdf)
- [Viktor Leis - TU Munich Database Group](https://db.in.tum.de/~leis/)
