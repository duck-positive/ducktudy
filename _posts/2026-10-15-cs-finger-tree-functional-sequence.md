---
layout: post
title: "핑거 트리: 범용 함수형 시퀀스 자료구조의 정수"
date: 2026-10-15
categories: [cs, computer-science]
tags: [finger-tree, functional-programming, data-structure, Haskell, sequence, 2-3-tree, monoid, persistent]
---

## 핑거 트리(Finger Tree)란?

**핑거 트리(Finger Tree)**는 2006년 Ralf Hinze와 Ross Paterson이 제안한 순수 함수형 자료구조로, 범용 **영속적 시퀀스(Persistent Sequence)**를 O(1) 분할상환 복잡도로 양쪽 끝에서 접근하고, O(log n)으로 분리·결합할 수 있다.

이름의 "Finger"는 자료구조의 **양쪽 끝(손가락 끝)**에 빠른 접근 포인터가 있다는 의미다. 내부적으로 **2-3 트리**를 기반으로 하며, 각 내부 노드에 **모노이드(Monoid)** 값을 캐시해 다양한 용도로 활용한다.

### 핵심 복잡도

| 연산 | 복잡도 |
|---|---|
| 앞/뒤에 추가 (prepend/append) | O(1) 분할상환 |
| 앞/뒤 원소 제거 (viewL/viewR) | O(1) 분할상환 |
| 두 시퀀스 결합 (concat) | O(log n) |
| 임의 위치 분리 (split) | O(log n) |
| 임의 위치 접근 (index) | O(log n) |
| 길이 조회 | O(1) (모노이드 캐시) |

---

## 왜 필요한가: 표준 자료구조의 한계

```
            prepend  append   concat   split   index
배열         O(n)     O(1)*   O(n)     O(n)    O(1)
연결 리스트   O(1)     O(n)    O(n)     O(n)    O(n)
이진 탐색 트리 O(log n) O(log n) O(n)  O(log n) O(log n)
이중 연결 큐  O(1)     O(1)    O(n)     O(n)    O(n)
핑거 트리    O(1)*    O(1)*   O(log n) O(log n) O(log n)
```

*(* = 분할상환)*

핑거 트리는 이 모든 연산을 한꺼번에 효율적으로 지원하는 거의 유일한 순수 함수형 자료구조다. Haskell의 `Data.Sequence`가 이를 기반으로 구현돼 있다.

---

## 자료구조의 구조

핑거 트리는 세 가지 케이스로 귀납적으로 정의된다.

```haskell
data FingerTree v a
  = Empty                                      -- 빈 트리
  | Single a                                   -- 원소 하나
  | Deep v (Digit a) (FingerTree v (Node v a)) (Digit a)
  --       ^^^^^^^^^ 왼쪽 손가락    ^^^^^^^^^^  오른쪽 손가락
  --                               ^^^^^^^^^^^^^^^^^^^^^ 재귀적인 내부 트리

data Digit a = One a | Two a a | Three a a a | Four a a a a

data Node v a = Node2 v a a | Node3 v a a a

-- v: 모노이드 값 (크기, 최댓값, 우선순위 등 — 목적에 따라 달라짐)
-- a: 원소 타입
```

`Deep` 노드는 왼쪽 손가락(1~4개 원소), 재귀 내부 트리(원소가 `Node`로 묶임), 오른쪽 손가락(1~4개 원소)으로 구성된다. 내부 트리는 깊이가 1 늘어날 때마다 원소가 `Node`로 그루핑되어 자연스럽게 2-3 트리를 형성한다.

**모노이드 캐시**: 각 `Deep`, `Node` 노드는 자신이 대표하는 모든 원소의 모노이드 합산 값을 캐시한다. 이를 통해 임의 분기점을 O(log n)에 찾는다.

---

## 코드 예제 1: Python으로 핵심 연산 구현

```python
from __future__ import annotations
from dataclasses import dataclass, field
from typing import Generic, TypeVar, Optional, Callable, Tuple
from functools import reduce

V = TypeVar("V")   # 모노이드 어노테이션
A = TypeVar("A")   # 원소 타입

# ─── 모노이드 클래스 ───────────────────────────────────────────
class Monoid:
    """모노이드: 항등원 mempty와 이항 연산 mappend"""
    def __init__(self, mempty, mappend: Callable):
        self.mempty  = mempty
        self.mappend = mappend

    def mconcat(self, items):
        return reduce(self.mappend, items, self.mempty)

# 크기 모노이드: 전체 원소 개수 추적
SIZE_MON = Monoid(0, lambda a, b: a + b)

# ─── 노드 ──────────────────────────────────────────────────────
@dataclass
class Node2:
    v: int            # 모노이드 값
    a: object
    b: object

@dataclass
class Node3:
    v: int
    a: object
    b: object
    c: object

# ─── 핑거 트리 ────────────────────────────────────────────────
class FT:
    """단순화된 핑거 트리 (크기 모노이드 사용, 순수 Python)"""

    @staticmethod
    def empty():
        return []

    @staticmethod
    def measure(tree):
        return _size(tree)

def _size(tree):
    if isinstance(tree, list):
        return len(tree)
    if isinstance(tree, tuple):
        kind = tree[0]
        if kind == "Empty":    return 0
        if kind == "Single":   return 1
        if kind == "Deep":     return tree[1]
    return 1

class FingerTree:
    """
    단순 구현: 이중 연결 덱과 재귀 내부 트리
    실제 구현은 Digit(1~4원소)와 Node(2~3원소)를 사용하지만
    여기서는 개념 전달을 위해 단순화
    """
    def __init__(self):
        self._left  = []   # 왼쪽 손가락 (최대 4원소)
        self._right = []   # 오른쪽 손가락 (최대 4원소)
        self._inner: Optional[FingerTree] = None  # 내부 트리
        self._size  = 0

    @classmethod
    def from_list(cls, items):
        ft = cls()
        for x in items:
            ft = ft.append(x)
        return ft

    def __len__(self):
        return self._size

    def is_empty(self):
        return self._size == 0

    def prepend(self, x):
        """앞에 원소 추가 — O(1) 분할상환"""
        new = FingerTree()
        new._size  = self._size + 1
        new._right = self._right[:]
        new._inner = self._inner

        if len(self._left) < 4:
            new._left = [x] + self._left
        else:
            # 왼쪽 손가락이 꽉 참: 앞 3개를 내부 트리에 내려보냄
            node = tuple(self._left[:3])   # Node3 단순화
            inner = self._inner or FingerTree()
            new._inner = _inner_prepend(inner, node)
            new._left  = [x, self._left[3]]
        return new

    def append(self, x):
        """뒤에 원소 추가 — O(1) 분할상환"""
        new = FingerTree()
        new._size  = self._size + 1
        new._left  = self._left[:]
        new._inner = self._inner

        if len(self._right) < 4:
            new._right = self._right + [x]
        else:
            node = tuple(self._right[-3:])
            inner = self._inner or FingerTree()
            new._inner = _inner_append(inner, node)
            new._right = [self._right[0], x]
        return new

    def peek_left(self):
        """왼쪽 끝 원소 조회"""
        if self._left:
            return self._left[0]
        raise IndexError("empty")

    def peek_right(self):
        """오른쪽 끝 원소 조회"""
        if self._right:
            return self._right[-1]
        if self._left:
            return self._left[-1]
        raise IndexError("empty")

    def to_list(self):
        result = list(self._left)
        if self._inner:
            for node in self._inner.to_list():
                if isinstance(node, tuple):
                    result.extend(node)
                else:
                    result.append(node)
        result.extend(self._right)
        return result


def _inner_prepend(tree: FingerTree, node):
    return tree.prepend(node)

def _inner_append(tree: FingerTree, node):
    return tree.append(node)


# ─── 데모 ─────────────────────────────────────────────────────
ft = FingerTree.from_list(range(20))
print(f"길이: {len(ft)}")
print(f"왼쪽 끝: {ft.peek_left()}")
print(f"오른쪽 끝: {ft.peek_right()}")
print(f"내용: {ft.to_list()}")

# 새 원소 추가 (영속성 — 원본 불변)
ft2 = ft.prepend(-1).prepend(-2)
print(f"\nft2 (앞에 -1, -2 추가): {ft2.to_list()[:5]} ...")
print(f"원본 ft 불변: {ft.to_list()[:3]} ...")
```

---

## 코드 예제 2: Haskell의 Data.Sequence 실사용

Haskell의 `Data.Sequence`는 핑거 트리 위에 구축된 표준 시퀀스 라이브러리다.

```haskell
import qualified Data.Sequence as Seq
import Data.Sequence (Seq, (|>), (<|), (><))

-- 핑거 트리 기반 Seq의 핵심 연산들
main :: IO ()
main = do
  -- 생성: O(n)
  let s = Seq.fromList [1..10 :: Int]
  putStrLn $ "길이: " ++ show (Seq.length s)   -- O(1) — 모노이드 캐시

  -- 앞/뒤 추가: O(1) 분할상환
  let s1 = 0 <| s        -- prepend
  let s2 = s |> 11       -- append
  putStrLn $ "prepend 0: " ++ show (Seq.index s1 0)
  putStrLn $ "append 11: " ++ show (Seq.index s2 10)

  -- 결합: O(log n)
  let sa = Seq.fromList [1..5 :: Int]
  let sb = Seq.fromList [6..10 :: Int]
  let sc = sa >< sb      -- concat
  putStrLn $ "결합 후 길이: " ++ show (Seq.length sc)

  -- 분리: O(log n)
  let (left, right) = Seq.splitAt 5 sc
  putStrLn $ "splitAt 5 — 왼쪽: " ++ show (Seq.index left 4)
  putStrLn $ "splitAt 5 — 오른쪽: " ++ show (Seq.index right 0)

  -- 임의 인덱스 접근: O(log n)
  putStrLn $ "인덱스 7: " ++ show (Seq.index sc 7)

  -- 우선순위 큐로 활용 (Priority 모노이드 적용)
  -- Data.Sequence는 크기 모노이드를 기본 사용하지만
  -- fingertree 패키지를 사용하면 임의 모노이드를 직접 지정 가능
  putStrLn "모노이드 교환으로 우선순위 큐, 구간 트리로 변환 가능"
```

모노이드를 교체하기만 하면 핑거 트리 하나로 여러 자료구조를 구현할 수 있다.

| 모노이드 | 용도 |
|---|---|
| `Sum` (크기) | 임의 인덱스 접근, 표준 Seq |
| `Max` (최댓값) | 최대 우선순위 큐 |
| `(Min, Max)` | 구간 최솟값/최댓값 쿼리 |
| `(Ordered, Set)` | 정렬된 집합, 삽입·삭제 |
| `Interval` | 구간 트리 |

이 "측정(measure) 바꾸기" 패턴 덕분에 핑거 트리는 단일 구현으로 다양한 고급 자료구조의 기반이 된다.

---

## split: 핑거 트리의 가장 강력한 연산

`split` 연산은 모노이드 값이 특정 조건(predicate)을 만족하는 위치에서 시퀀스를 분리한다.

```
split p t = (l, x, r)   where:
  p (measure l) = false
  p (measure (l ++ [x])) = true
```

예를 들어 크기 모노이드와 `p = (>= k)` 조건을 사용하면 인덱스 k에서의 분리가 된다. 우선순위 모노이드와 `p = (> maxPriority)` 조건을 사용하면 최대 우선순위 원소 추출이 된다.

---

## 영속성(Persistence)의 이점

핑거 트리는 순수 함수형이므로 **영속적(Persistent)**이다. 모든 연산이 기존 트리를 수정하지 않고 새 트리를 반환한다. 이는 다음을 가능하게 한다.

- **버전 관리**: 이전 상태를 비용 없이 보존
- **병렬 접근**: 불변 공유로 락 불필요
- **스냅샷**: 상태를 저장해두고 롤백

구조 공유(structural sharing) 덕분에 각 연산은 O(log n)개의 새 노드만 할당하며, 나머지는 이전 트리와 공유한다.

---

## 주의사항과 팁

**상수 인자**: 이론적 복잡도는 우수하지만 노드당 수십 바이트의 오버헤드가 있다. 수백 개 미만의 소규모 데이터에서는 단순 배열이 더 빠를 수 있다.

**캐시 친화성 부족**: 포인터 기반 트리는 연속 메모리 배열보다 캐시 미스가 많다. 인덱스 위주 접근이 주된 사용 사례라면 `Vector`나 `Array`가 더 적합하다.

**GC 압박**: 영속적 자료구조는 많은 단기 노드를 생성한다. GC 압박이 심한 환경에서는 `Data.Sequence`보다 가변 자료구조가 더 실용적일 수 있다.

**Haskell 외 지원**: Scala의 `cats` 라이브러리, Clojure의 `PersistentVector`(개념적으로 유사), Rust의 `im` 크레이트 등에서 유사한 구조를 제공한다.

---

## 참고 자료
- [Finger Trees: A Simple General-purpose Data Structure — Hinze & Paterson (2006)](https://www.staff.city.ac.uk/~ross/papers/FingerTree.pdf)
- [Finger Tree — Wikipedia](https://en.wikipedia.org/wiki/Finger_tree)
- [fingertree: Generic finger-tree structure (Hackage)](https://hackage-content-origin.haskell.org/package/fingertree-0.1.4.2/candidate)
- [Data.Sequence — Haskell Platform Docs](https://hackage-content-origin.haskell.org/package/hw-fingertree-0.1.0.0/candidate/src/HaskellWorks/Data/FingerTree.hs)
