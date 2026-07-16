---
layout: post
title: "Van Emde Boas 트리 완전 정복: O(log log U)로 정수 집합을 정복하는 자료구조"
date: 2026-07-16
categories: [cs, computer-science]
tags: [van-emde-boas, vEB-tree, data-structure, algorithm, integer-set, priority-queue]
---

## Van Emde Boas 트리란 무엇인가

자료구조를 배울 때 우리는 보통 다음과 같은 연산 복잡도에 익숙해진다:
- 이진 탐색 트리(BST): O(log n)
- 해시 테이블: 평균 O(1), 최악 O(n)

그런데 여기, **O(log log U)**라는 놀라운 복잡도를 달성하는 자료구조가 있다. **Van Emde Boas 트리(vEB tree)**다.

vEB 트리는 1975년 네덜란드의 컴퓨터 과학자 Peter van Emde Boas가 제안했다. 키가 정수(integer)이고, 그 범위가 **[0, U-1]**로 제한될 때, 다음 연산을 모두 **O(log log U)** 시간에 처리한다:

- **Insert(x)**: 원소 삽입
- **Delete(x)**: 원소 삭제
- **Member(x)**: 포함 여부 확인
- **Minimum()**: 최솟값
- **Maximum()**: 최댓값
- **Successor(x)**: x의 다음으로 큰 원소
- **Predecessor(x)**: x의 다음으로 작은 원소

Successor와 Predecessor를 O(log log U)에 처리하는 자료구조는 매우 드물다. 이 덕분에 vEB 트리는 **정수 키를 다루는 우선순위 큐**로 활용될 때 특히 강력하다.

---

## 왜 O(log log U)인가

### 핵심 아이디어: 재귀적 분할

vEB 트리의 핵심은 **유니버스를 반복해서 제곱근으로 분할**하는 것이다.

유니버스 크기가 U일 때, √U 크기의 클러스터 √U개로 나눈다. 각 클러스터도 같은 방식으로 재귀적으로 분할된다. 분할 깊이는 U → √U → U^(1/4) → ... → 2가 되므로 **log log U** 단계가 된다.

```
U = 16 (4비트 정수)의 예:
최상위 vEB(16)
├── cluster[0] → vEB(4)  [0..3  범위]
├── cluster[1] → vEB(4)  [4..7  범위]
├── cluster[2] → vEB(4)  [8..11 범위]
├── cluster[3] → vEB(4)  [12..15 범위]
└── summary   → vEB(4)  [어느 cluster가 비어있지 않은지]
```

`summary`는 각 클러스터가 **비어 있지 않은지**를 추적하는 보조 vEB 트리다. Successor 연산 시 현재 클러스터에서 찾지 못하면 summary를 통해 다음 비어 있지 않은 클러스터로 O(log log U) 만에 점프할 수 있다.

### 재귀 관계와 복잡도 분석

vEB 연산의 시간 복잡도 T(U)는 다음 점화식을 만족한다:

```
T(U) = T(√U) + O(1)
```

각 연산마다 크기가 √U인 하위 문제 하나를 풀고 O(1) 추가 작업만 한다. 이를 치환법으로 풀면:

```
T(U) = T(U^(1/2)) + O(1)
     = T(U^(1/4)) + O(1) + O(1)
     = ...
     = O(1) × (log log U 단계)
     = O(log log U)
```

---

## vEB 트리 구현 (Python)

### 기본 구조 정의

```python
import math

class VEBTree:
    """Van Emde Boas Tree 구현 (유니버스 크기 U는 2의 거듭제곱 가정)"""

    def __init__(self, universe_size):
        self.u = universe_size      # 유니버스 크기
        self.min = None             # 최솟값 (클러스터에 저장 안 됨)
        self.max = None             # 최댓값

        if universe_size <= 2:
            # 기저 사례: 유니버스가 2이하면 재귀 없이 처리
            self.clusters = None
            self.summary = None
        else:
            sqrt_u = self._upper_sqrt()
            # √U개의 클러스터, 각 크기는 √U
            self.clusters = [VEBTree(sqrt_u) for _ in range(sqrt_u)]
            # summary: 어느 클러스터가 비어있지 않은지 추적
            self.summary = VEBTree(sqrt_u)

    def _upper_sqrt(self):
        """2^(ceil(log2(U)/2)) — 상위 절반 비트"""
        return 2 ** math.ceil(math.log2(self.u) / 2)

    def _lower_sqrt(self):
        """2^(floor(log2(U)/2)) — 하위 절반 비트"""
        return 2 ** math.floor(math.log2(self.u) / 2)

    def _high(self, x):
        """x의 상위 비트 → 클러스터 인덱스"""
        return x // self._lower_sqrt()

    def _low(self, x):
        """x의 하위 비트 → 클러스터 내 위치"""
        return x % self._lower_sqrt()

    def _index(self, cluster_idx, offset):
        """클러스터 인덱스와 오프셋으로 전체 인덱스 복원"""
        return cluster_idx * self._lower_sqrt() + offset

    # ─────────────────────────────────────────────
    # 핵심 연산 구현
    # ─────────────────────────────────────────────

    def insert(self, x):
        """O(log log U) 삽입"""
        if self.min is None:
            # 빈 트리: min과 max를 x로 설정 (클러스터에 저장 안 함)
            self.min = x
            self.max = x
            return

        if x < self.min:
            x, self.min = self.min, x  # x를 min과 교환

        if self.u > 2:
            cluster_idx = self._high(x)
            pos = self._low(x)

            if self.clusters[cluster_idx].min is None:
                # 해당 클러스터가 비어있었음: summary에도 등록
                self.summary.insert(cluster_idx)
                # 클러스터에 직접 삽입 (재귀 깊이 1회로 끝남)
                self.clusters[cluster_idx].min = pos
                self.clusters[cluster_idx].max = pos
            else:
                self.clusters[cluster_idx].insert(pos)

        if x > self.max:
            self.max = x

    def member(self, x):
        """O(log log U) 포함 여부 확인"""
        if x == self.min or x == self.max:
            return True
        if self.u <= 2:
            return False
        return self.clusters[self._high(x)].member(self._low(x))

    def successor(self, x):
        """O(log log U) 후계자(x보다 큰 가장 작은 원소) 탐색"""
        if self.u <= 2:
            if x == 0 and self.max == 1:
                return 1
            return None

        if self.min is not None and x < self.min:
            return self.min

        cluster_idx = self._high(x)
        pos = self._low(x)
        cluster = self.clusters[cluster_idx]

        # Case 1: 현재 클러스터 내에 후계자가 있음
        if cluster.max is not None and pos < cluster.max:
            offset = cluster.successor(pos)
            return self._index(cluster_idx, offset)

        # Case 2: 다음 비어있지 않은 클러스터로 점프
        next_cluster = self.summary.successor(cluster_idx)
        if next_cluster is None:
            return None
        offset = self.clusters[next_cluster].min
        return self._index(next_cluster, offset)

    def minimum(self):
        return self.min

    def maximum(self):
        return self.max
```

### 동작 확인 예제

```python
# vEB 트리 테스트
veb = VEBTree(16)  # 4비트 정수 (0~15)

# 삽입
for x in [2, 3, 7, 11, 14]:
    veb.insert(x)

print(f"Min: {veb.minimum()}")        # Min: 2
print(f"Max: {veb.maximum()}")        # Max: 14
print(f"Member(7): {veb.member(7)}")  # Member(7): True
print(f"Member(5): {veb.member(5)}")  # Member(5): False

# Successor 연산
x = 0
print("정렬된 순서로 출력:")
while True:
    x = veb.successor(x - 1) if x == 0 else veb.successor(x)
    if x is None:
        break
    print(x, end=" ")
# 출력: 2 3 7 11 14

# 성능 비교 (U=2^20, n=100,000 원소)
import random, time

U = 1 << 20  # 1,048,576
veb_large = VEBTree(U)
elements = random.sample(range(U), 100_000)

start = time.time()
for e in elements:
    veb_large.insert(e)
print(f"\nvEB insert 100k 원소: {time.time()-start:.3f}s")

start = time.time()
queries = random.sample(elements, 10_000)
for q in queries:
    veb_large.successor(q)
print(f"vEB successor 10k 쿼리: {time.time()-start:.3f}s")
```

---

## 공간 복잡도와 트레이드오프

vEB 트리의 가장 큰 단점은 **공간 복잡도 O(U)**다. 원소 수(n)에 비례하지 않고, 유니버스 크기(U)에 비례한다. U = 2^32이면 약 40억 개의 노드가 필요하므로 실용적이지 않다.

이를 개선한 변형이 **Proto-vEB 트리**와 **y-fast trie** 등이다. y-fast trie는 공간을 O(n)으로 줄이면서도 연산을 O(log log U) expected time으로 유지한다.

| 자료구조 | Insert | Successor | 공간 |
|---------|--------|-----------|------|
| 이진 힙 | O(log n) | O(n) | O(n) |
| AVL/RB 트리 | O(log n) | O(log n) | O(n) |
| vEB 트리 | O(log log U) | O(log log U) | **O(U)** |
| y-fast trie | O(log log U) expected | O(log log U) expected | O(n) |

---

## 실전 응용 사례

### 1. 네트워크 라우팅 (IP Lookup)

IPv4 주소는 32비트 정수다. 라우터에서 패킷의 목적지 IP에 대한 다음 홉(next-hop)을 찾으려면 포워딩 테이블에서 **가장 긴 프리픽스 매칭(LPM)**을 해야 한다. vEB 트리 기반의 정수 집합 연산으로 이를 O(log log U) = O(log 32) = O(5) 만에 처리할 수 있다.

### 2. 동적 그래프 알고리즘

정수 키를 다루는 우선순위 큐(Dijkstra, Prim 등)에서 vEB 트리를 사용하면 전체 알고리즘 복잡도를 개선할 수 있다. Dijkstra를 vEB 트리로 구현하면 O((V + E) log log V)가 된다.

### 3. 데이터베이스 인덱스 (정수형 키)

유니버스가 적당히 작은 정수 칼럼(예: 사용자 ID, 타임스탬프)에서 범위 쿼리와 후계자 탐색을 빠르게 처리할 수 있다.

---

## 주의사항과 실전 팁

### 1. 유니버스 크기가 핵심 제약

vEB 트리는 키 범위가 미리 알려져 있어야 한다. 동적으로 범위가 늘어나는 경우 재구축이 필요하다. 실제로는 유니버스를 2의 거듭제곱으로 패딩하여 구현한다.

### 2. 캐시 비효율성

트리 구조가 깊고 랜덤한 메모리 접근 패턴 때문에 CPU 캐시를 효율적으로 활용하지 못한다. 이론적 복잡도와 달리 실측 성능이 레드-블랙 트리보다 느릴 수 있다. Cache-oblivious 변형을 사용하면 이를 개선할 수 있다.

### 3. 구현 복잡도

단순 이진 탐색 트리에 비해 구현이 복잡하다. 실무에서는 B-트리나 skip list처럼 검증된 라이브러리를 우선 고려하고, 정수 키 + 낮은 레이턴시 요구사항이 명확할 때만 vEB를 선택한다.

---

## 참고 자료

- [Van Emde Boas tree — Wikipedia](https://en.wikipedia.org/wiki/Van_Emde_Boas_tree)
- [van Emde Boas Tree — OpenGenus IQ](https://iq.opengenus.org/van-emde-boas-tree/)
- [Mastering Van Emde Boas Tree — NumberAnalytics](https://www.numberanalytics.com/blog/van-emde-boas-tree-ultimate-guide)
- [van Emde Boas Trees Formal Verification (Isabelle/AFP)](https://isa-afp.org/browser_info/current/AFP/Van_Emde_Boas_Trees/outline.pdf)
