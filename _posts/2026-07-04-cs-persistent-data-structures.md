---
layout: post
title: "영속성 자료구조(Persistent Data Structures) 완전 정복: 불변성과 구조 공유로 함수형 프로그래밍 실현하기"
date: 2026-07-04
categories: [cs, computer-science]
tags: [persistent-data-structures, functional-programming, immutable, structural-sharing, clojure, haskell, concurrency]
---

## 개요: 변경하지 않고 변경하는 자료구조

일반적인 자료구조는 값을 수정하면 이전 상태가 사라집니다. 연결 리스트의 노드를 삭제하거나, 배열의 원소를 교체하면, 수정 전 상태로 돌아갈 방법이 없습니다. 이를 **에페머럴(Ephemeral) 자료구조**라고 합니다.

반면 **영속성 자료구조(Persistent Data Structures)** 는 수정 연산을 수행할 때마다 이전 버전을 보존하면서 새로운 버전을 만들어냅니다. "이 리스트에서 첫 번째 원소를 제거한 리스트"는 기존 리스트를 파괴하지 않고, 수정된 새 리스트를 반환합니다. 이전 버전도 여전히 유효하고, 접근 가능합니다.

이 특성은 함수형 프로그래밍의 **불변성(Immutability)** 원칙과 완벽하게 맞아 떨어지며, 동시성 프로그래밍, 버전 관리 시스템(Git의 내부), 실행 취소(Undo) 기능 구현 등 폭넓은 분야에서 활용됩니다.

---

## 왜 영속성 자료구조가 필요한가

### 문제 1: 공유 가변 상태의 위험

멀티스레드 환경에서 가변(mutable) 자료구조를 공유하려면 반드시 잠금(Lock)이 필요합니다:

```java
// 위험한 코드: 경쟁 조건 발생
List<Integer> shared = new ArrayList<>();
// Thread A: shared.add(1)
// Thread B: shared.add(2)
// → 결과 불확정
```

Lock을 사용하면 동시성이 저하됩니다. 반면 불변 자료구조는 수정이 불가능하므로, 어떤 스레드가 언제 읽더라도 일관된 뷰를 제공합니다. Lock이 필요 없습니다.

### 문제 2: 완전 복사(Deep Copy)의 비용

불변성을 구현하는 단순한 방법은 수정 시 전체를 복사하는 것입니다:

```python
# 매번 O(n) 복사 — 비효율적
original = list(range(1_000_000))
modified = original.copy()  # 100만 원소 전체 복사!
modified.append(1_000_001)
```

100만 원소 리스트를 한 번 수정할 때마다 전체를 복사하면 메모리와 시간 모두 O(n)이 소모됩니다. 영속성 자료구조는 **구조 공유(Structural Sharing)** 를 통해 이 문제를 해결합니다.

---

## 구조 공유: 핵심 원리

영속성 자료구조의 핵심은 **변경된 부분만 새로 만들고, 나머지는 이전 버전과 공유**한다는 것입니다.

### 영속성 연결 리스트

```
# 원본 리스트: [3, 2, 1]
v1: 3 → 2 → 1 → nil

# v1 앞에 4를 추가한 v2 = cons(4, v1)
v2: 4 → [3 → 2 → 1 → nil]
          ↑
          v1과 공유 (복사 없음!)

# v1과 v2는 독립적으로 유효
```

연결 리스트의 prepend 연산은 O(1)이고, 이전 리스트와 메모리를 공유합니다. 이것이 함수형 언어 Haskell, Clojure, Erlang에서 리스트가 앞에서 추가/제거를 선호하는 이유입니다.

---

## 핵심 자료구조: Path Copying 이진 트리

임의 위치 수정이 가능한 영속성 자료구조를 위해 **경로 복사(Path Copying)** 기법이 사용됩니다. 트리에서 특정 노드를 수정할 때, 해당 노드에서 루트까지의 경로만 새로 복사하고, 나머지 서브트리는 공유합니다.

```
원본 트리 v1:        수정 후 v2 (노드 2 값 변경):

        5                    5'  ← 새로 복사
       / \                  / \
      3   7               3'   7  ← 공유
     / \                 / \
    2   4               2'   4  ← 공유
                       (값 수정)

수정된 노드: 5', 3', 2'  (루트까지의 경로 = O(log n))
공유 노드:  7, 4, 그 외 모든 노드
```

O(log n) 수준의 노드만 새로 생성하므로 메모리와 시간 모두 효율적입니다.

---

## 실제 구현 예제

### 예제 1: 영속성 이진 탐색 트리 (Python)

```python
from __future__ import annotations
from dataclasses import dataclass
from typing import Optional, TypeVar, Generic

T = TypeVar('T')

@dataclass(frozen=True)  # frozen=True로 불변 노드 생성
class Node(Generic[T]):
    key: int
    value: T
    left: Optional['Node[T]'] = None
    right: Optional['Node[T]'] = None

class PersistentBST(Generic[T]):
    """
    영속성 이진 탐색 트리.
    insert/delete 연산은 항상 새로운 루트를 반환하고,
    기존 트리는 그대로 보존된다.
    """
    def __init__(self, root: Optional[Node[T]] = None):
        self._root = root

    @property
    def root(self):
        return self._root

    def insert(self, key: int, value: T) -> 'PersistentBST[T]':
        """O(log n): 루트까지의 경로만 새 노드 생성"""
        new_root = self._insert_node(self._root, key, value)
        return PersistentBST(new_root)

    def _insert_node(self, node: Optional[Node[T]], key: int, value: T) -> Node[T]:
        if node is None:
            return Node(key=key, value=value)
        if key < node.key:
            # 왼쪽 서브트리만 재귀적으로 변경, 오른쪽은 공유
            new_left = self._insert_node(node.left, key, value)
            return Node(key=node.key, value=node.value,
                        left=new_left, right=node.right)  # right는 공유!
        elif key > node.key:
            new_right = self._insert_node(node.right, key, value)
            return Node(key=node.key, value=node.value,
                        left=node.left, right=new_right)  # left는 공유!
        else:
            # 키 업데이트: 새 노드 생성, 좌우 자식은 공유
            return Node(key=key, value=value,
                        left=node.left, right=node.right)

    def get(self, key: int) -> Optional[T]:
        node = self._root
        while node:
            if key == node.key:
                return node.value
            node = node.left if key < node.key else node.right
        return None

    def inorder(self) -> list:
        result = []
        def _traverse(node):
            if node:
                _traverse(node.left)
                result.append((node.key, node.value))
                _traverse(node.right)
        _traverse(self._root)
        return result


# 사용 예시
if __name__ == "__main__":
    # 버전 0: 빈 트리
    v0 = PersistentBST()

    # 버전 1: {5: 'five'} 삽입
    v1 = v0.insert(5, 'five')

    # 버전 2: {3: 'three'} 삽입
    v2 = v1.insert(3, 'three')

    # 버전 3: {7: 'seven'} 삽입
    v3 = v2.insert(7, 'seven')

    # 버전 4: 키 3의 값 업데이트
    v4 = v3.insert(3, 'THREE_UPDATED')

    print("v3:", v3.inorder())  # [(3, 'three'), (5, 'five'), (7, 'seven')]
    print("v4:", v4.inorder())  # [(3, 'THREE_UPDATED'), (5, 'five'), (7, 'seven')]
    print("v3은 여전히 유효:", v3.get(3))  # 'three' — 버전 3 보존됨!

    # 구조 공유 확인: v3와 v4의 루트 노드는 다르지만 서브트리를 공유
    print("v3.root.right is v4.root.right:", v3.root.right is v4.root.right)  # True!
    print("v3.root.left is v4.root.left:",   v3.root.left  is v4.root.left)   # False (업데이트됨)
```

### 예제 2: 영속성 해시 배열 맵 트라이 (Hash Array Mapped Trie, Clojure 방식) in Python

Clojure의 `PersistentHashMap`이 사용하는 **Hash Array Mapped Trie (HAMT)** 를 단순화한 버전입니다. 키의 해시 비트를 단계별로 사용해 트라이를 탐색하며, 수정 시 경로만 복사합니다.

```python
from typing import Any, Dict, Tuple, Optional
import copy

BRANCHING_FACTOR = 4  # 실제 Clojure는 32-way trie

class HAMTNode:
    """불변 트라이 노드 (복사 전략으로 영속성 구현)"""
    __slots__ = ('children', 'value', 'key')

    def __init__(self, children=None, key=None, value=None):
        self.children: Dict[int, 'HAMTNode'] = children or {}
        self.key = key
        self.value = value

    def is_leaf(self) -> bool:
        return self.key is not None

    def copy(self) -> 'HAMTNode':
        """얕은 복사: children dict를 새로 만들지만 자식 노드는 공유"""
        return HAMTNode(
            children=dict(self.children),  # dict는 새로 생성
            key=self.key,
            value=self.value
        )


class PersistentHashMap:
    """
    각 assoc()/dissoc() 연산이 새 루트를 반환하며,
    변경 경로의 O(log n) 노드만 복사한다.
    """
    def __init__(self, root: Optional[HAMTNode] = None, size: int = 0):
        self._root = root or HAMTNode()
        self._size = size

    def __len__(self) -> int:
        return self._size

    def _hash_bits(self, key, depth: int) -> int:
        """해시값에서 특정 깊이의 비트 그룹을 추출"""
        h = hash(key)
        shift = depth * 2  # BRANCHING_FACTOR=4 → 2비트씩
        return (h >> shift) & (BRANCHING_FACTOR - 1)

    def get(self, key: Any, default=None) -> Any:
        node = self._root
        depth = 0
        while not node.is_leaf():
            bit = self._hash_bits(key, depth)
            if bit not in node.children:
                return default
            node = node.children[bit]
            depth += 1
        return node.value if node.key == key else default

    def assoc(self, key: Any, value: Any) -> 'PersistentHashMap':
        """새 맵 버전을 반환. 수정 경로만 복사."""
        new_root, is_new = self._assoc_node(
            self._root, key, value, depth=0
        )
        delta = 1 if is_new else 0
        return PersistentHashMap(new_root, self._size + delta)

    def _assoc_node(self, node: HAMTNode, key, value, depth: int
                    ) -> Tuple[HAMTNode, bool]:
        if node.is_leaf():
            if node.key == key:
                # 값만 업데이트: 새 리프 노드 생성
                new_leaf = HAMTNode(key=key, value=value)
                return new_leaf, False  # 신규가 아닌 갱신
            # 해시 충돌 해결: 중간 노드로 교체 후 재삽입
            new_internal = HAMTNode()
            new_internal, _ = self._assoc_node(
                new_internal, node.key, node.value, depth
            )
            new_internal, _ = self._assoc_node(
                new_internal, key, value, depth
            )
            return new_internal, True

        bit = self._hash_bits(key, depth)
        new_node = node.copy()  # 이 노드만 복사, 나머지 자식은 공유!

        if bit in new_node.children:
            child, is_new = self._assoc_node(
                new_node.children[bit], key, value, depth + 1
            )
            new_node.children[bit] = child
            return new_node, is_new
        else:
            new_node.children[bit] = HAMTNode(key=key, value=value)
            return new_node, True


# 사용 예시
if __name__ == "__main__":
    m0 = PersistentHashMap()

    m1 = m0.assoc('name', 'Alice')
    m2 = m1.assoc('age', 30)
    m3 = m2.assoc('city', 'Seoul')
    m4 = m3.assoc('name', 'Bob')  # name 업데이트

    print(f"m3 name: {m3.get('name')}")    # Alice
    print(f"m4 name: {m4.get('name')}")    # Bob
    print(f"m3 size: {len(m3)}")           # 3
    print(f"m4 size: {len(m4)}")           # 3 (갱신이므로 동일)

    # m3는 여전히 'Alice'를 가리킴 — 영속성 보장!
    print(f"m3 여전히 유효: {m3.get('name')}")   # Alice
    print(f"m4['city']: {m4.get('city')}")         # Seoul — 공유 서브트리에서 읽음
```

---

## 영속성의 종류

| 종류 | 설명 | 예시 |
|------|------|------|
| **Partial Persistence** | 모든 버전 읽기 가능, 최신 버전만 수정 가능 | 단순 로그 기반 이력 |
| **Full Persistence** | 모든 버전 읽기/쓰기 가능 | Git (브랜치로 임의 버전에서 분기) |
| **Confluent Persistence** | 두 버전을 합쳐 새 버전 생성 가능 | Git merge |
| **Functional Persistence** | Full Persistence + 순수 함수 인터페이스 | Clojure의 persistent collections |

---

## 주의사항과 실무 팁

### 1. GC 압력에 주의
매번 새 버전을 생성하므로 단명(short-lived) 객체가 많이 생성됩니다. JVM 환경에서는 Young Generation GC 횟수가 늘어날 수 있습니다. 실제 성능은 측정을 통해 검증하세요.

### 2. 캐시 지역성 저하 가능성
트리 기반 구조 공유는 포인터 추적이 많아 배열 기반 자료구조보다 캐시 미스가 잦을 수 있습니다. 성능이 중요한 경로에서는 벤치마크 필수입니다.

### 3. 언어별 기본 지원 현황
- **Clojure**: PersistentVector, PersistentHashMap 등 언어 내장
- **Haskell**: 모든 기본 자료구조가 불변 (Data.Map, Data.Set 등)
- **Scala**: `scala.collection.immutable` 패키지
- **Python**: `pyrsistent` 라이브러리 (PVector, PMap, PSet)
- **Java**: `Vavr` 라이브러리의 `io.vavr.collection`

### 4. Git이 바로 영속성 자료구조
Git의 내부 객체 모델이 완벽한 영속성 트리입니다. 커밋(Commit)은 트리 스냅샷이고, 새 커밋은 변경된 파일/디렉터리만 새 객체로 만들고 나머지는 이전 커밋의 트리 객체를 공유합니다. `git cat-file -p HEAD~1^{tree}`로 직접 확인할 수 있습니다.

---

## 정리

영속성 자료구조는 **구조 공유**를 통해 불변성을 O(log n) 비용으로 구현합니다. 함수형 프로그래밍의 핵심 도구이자, 멀티스레드 환경에서 Lock 없이 안전한 공유를 가능하게 하는 강력한 패턴입니다. Clojure의 HAMT, Haskell의 Data.Map, Git의 객체 모델이 모두 이 원리 위에 서 있습니다.

## 참고 자료
- [pyrsistent: Python용 영속성 자료구조 라이브러리 (GitHub)](https://github.com/tobgu/pyrsistent)
- [Rich Hickey - Persistent Data Structures and Managed References (강연 트랜스크립트)](https://github.com/matthiasn/talk-transcripts/blob/master/Hickey_Rich/PersistentDataStructure.md)
