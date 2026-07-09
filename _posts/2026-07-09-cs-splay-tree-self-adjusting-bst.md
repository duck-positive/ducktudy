---
layout: post
title: "스플레이 트리(Splay Tree) 완전 정복: 자주 쓰는 노드가 루트로 올라오는 자기 조정 이진 탐색 트리"
date: 2026-07-09
categories: [cs, computer-science]
tags: [data-structure, tree, splay-tree, amortized-analysis, binary-search-tree, cache]
---

이진 탐색 트리(BST)에서 같은 데이터에 반복 접근하는 패턴은 매우 흔하다. AVL 트리나 레드-블랙 트리는 균형을 엄격하게 유지하지만, **최근 접근한 데이터가 다시 접근될 가능성이 높다**는 참조 지역성(Temporal Locality)을 활용하지는 않는다. **스플레이 트리(Splay Tree)** 는 이 원리를 정면에서 활용한다. 접근할 때마다 해당 노드를 루트로 끌어올려, 자주 쓰는 노드는 항상 빠르게 찾을 수 있도록 스스로를 재구성한다.

---

## 스플레이 트리란?

1985년 Daniel D. Sleator와 Robert E. Tarjan이 논문 "Self-Adjusting Binary Search Trees"에서 제안한 자료구조다. 핵심 아이디어는 단순하다:

> **접근(탐색, 삽입, 삭제)이 발생할 때마다 해당 노드를 Splay(회전) 연산으로 루트로 끌어올린다.**

이를 통해:
- 자주 접근되는 노드는 루트 근처에 유지된다
- 최악의 경우 O(n) 비용이 들 수 있지만, 분할 상환 분석(Amortized Analysis)으로 **작업 당 O(log n)**이 보장된다
- AVL/레드-블랙 트리보다 구현이 단순하다 (색상 정보나 높이 정보 불필요)

### Splay 연산의 세 가지 케이스

노드 x를 루트로 끌어올리는 과정에서 x의 부모 p, 조부모 g의 관계에 따라 세 가지 회전 케이스가 있다.

**Case 1: Zig** — p가 루트인 경우 (x가 p의 왼쪽 또는 오른쪽 자식)
```
     p               x
    / \             / \
   x   C    →     A   p
  / \                 / \
 A   B               B   C
```
단순 우회전(또는 좌회전) 1회 수행.

**Case 2: Zig-Zig** — x와 p가 같은 방향 자식인 경우 (둘 다 왼쪽 또는 둘 다 오른쪽)
```
       g               x
      / \             / \
     p   D           A   p
    / \       →         / \
   x   C               B   g
  / \                      / \
 A   B                    C   D
```
p 먼저 회전한 후 x를 회전 (순서가 중요: Zig-Zag와 다름).

**Case 3: Zig-Zag** — x와 p가 다른 방향 자식인 경우 (p가 왼쪽 자식인데 x는 오른쪽 자식)
```
     g               x
    / \             / \
   p   D           p   g
  / \       →    / \ / \
 A   x           A  B C  D
    / \
   B   C
```
x를 두 번 회전 (x 기준으로 두 번 AVL 회전과 동일).

---

## 분할 상환 분석: 왜 O(log n)인가?

스플레이 트리의 단일 연산은 O(n)이 될 수 있다. 예를 들어 완전히 편향된 트리(오른쪽으로만 늘어진)에서 가장 깊은 노드를 접근하면 O(n) 회전이 필요하다. 그러나 그 결과로 트리가 균형에 가까워지므로, **이후 연산들은 빠르게 처리**된다.

Sleator와 Tarjan은 **퍼텐셜 함수(Potential Function)** φ를 정의해 이를 증명했다.

φ(T) = Σ rank(v), 여기서 rank(v) = log₂(size(v))

**분할 상환 분석 결론**: m번의 연산을 n개 노드 트리에서 수행할 때 총 비용은 **O((m + n) log n)**. 즉 연산 당 O(log n).

**Access Lemma**: 노드 t에 접근할 때 드는 분할 상환 비용은 최대 1 + 3(rank(root) - rank(t)) = O(log n).

---

## 코드 예제 1: Python으로 완전한 Splay Tree 구현

```python
from __future__ import annotations
from typing import Generic, TypeVar, Optional

K = TypeVar('K')
V = TypeVar('V')

class Node(Generic[K, V]):
    __slots__ = ('key', 'value', 'left', 'right', 'parent')
    
    def __init__(self, key: K, value: V):
        self.key = key
        self.value = value
        self.left: Optional['Node[K, V]'] = None
        self.right: Optional['Node[K, V]'] = None
        self.parent: Optional['Node[K, V]'] = None

class SplayTree(Generic[K, V]):
    """
    Splay Tree 구현
    - 탐색, 삽입, 삭제: 분할 상환 O(log n)
    - 공간: O(n)
    """
    
    def __init__(self):
        self.root: Optional[Node[K, V]] = None
        self._size = 0
    
    def __len__(self) -> int:
        return self._size
    
    # --- 회전 연산 ---
    def _rotate_right(self, x: Node) -> None:
        """x를 우회전: x의 왼쪽 자식 p가 x의 자리로"""
        p = x.left
        if p is None:
            return
        
        # p의 오른쪽 자식을 x의 왼쪽으로
        x.left = p.right
        if p.right:
            p.right.parent = x
        
        # p가 x의 부모를 이어받음
        p.parent = x.parent
        if x.parent is None:
            self.root = p
        elif x is x.parent.left:
            x.parent.left = p
        else:
            x.parent.right = p
        
        p.right = x
        x.parent = p
    
    def _rotate_left(self, x: Node) -> None:
        """x를 좌회전: x의 오른쪽 자식 p가 x의 자리로"""
        p = x.right
        if p is None:
            return
        
        x.right = p.left
        if p.left:
            p.left.parent = x
        
        p.parent = x.parent
        if x.parent is None:
            self.root = p
        elif x is x.parent.left:
            x.parent.left = p
        else:
            x.parent.right = p
        
        p.left = x
        x.parent = p
    
    # --- Splay 핵심 연산 ---
    def _splay(self, x: Node) -> None:
        """x를 루트로 끌어올림"""
        while x.parent is not None:
            p = x.parent
            g = p.parent
            
            if g is None:
                # Zig: p가 루트
                if x is p.left:
                    self._rotate_right(p)
                else:
                    self._rotate_left(p)
            elif x is p.left and p is g.left:
                # Zig-Zig (왼쪽-왼쪽)
                self._rotate_right(g)
                self._rotate_right(p)
            elif x is p.right and p is g.right:
                # Zig-Zig (오른쪽-오른쪽)
                self._rotate_left(g)
                self._rotate_left(p)
            elif x is p.right and p is g.left:
                # Zig-Zag (왼쪽-오른쪽)
                self._rotate_left(p)
                self._rotate_right(g)
            else:
                # Zig-Zag (오른쪽-왼쪽)
                self._rotate_right(p)
                self._rotate_left(g)
    
    # --- 탐색 ---
    def get(self, key: K) -> Optional[V]:
        node = self._find(key)
        if node is None:
            return None
        self._splay(node)  # 접근 → 루트로 끌어올림
        return node.value
    
    def _find(self, key: K) -> Optional[Node]:
        """BST 탐색 (splay 없이)"""
        node = self.root
        last = None
        while node:
            last = node
            if key == node.key:
                return node
            elif key < node.key:
                node = node.left
            else:
                node = node.right
        # 탐색 실패 시 마지막 방문 노드를 splay (캐시 효과)
        if last:
            self._splay(last)
        return None
    
    # --- 삽입 ---
    def put(self, key: K, value: V) -> None:
        if self.root is None:
            self.root = Node(key, value)
            self._size += 1
            return
        
        # BST 삽입 위치 탐색
        node = self.root
        while True:
            if key == node.key:
                node.value = value  # 업데이트
                self._splay(node)
                return
            elif key < node.key:
                if node.left is None:
                    node.left = Node(key, value)
                    node.left.parent = node
                    self._splay(node.left)
                    self._size += 1
                    return
                node = node.left
            else:
                if node.right is None:
                    node.right = Node(key, value)
                    node.right.parent = node
                    self._splay(node.right)
                    self._size += 1
                    return
                node = node.right
    
    # --- 삭제 ---
    def delete(self, key: K) -> bool:
        node = self._find(key)
        if node is None:
            return False
        
        self._splay(node)  # 삭제할 노드를 루트로
        
        # 루트를 제거하고 왼쪽/오른쪽 서브트리 합병
        if self.root.left is None:
            self.root = self.root.right
            if self.root:
                self.root.parent = None
        elif self.root.right is None:
            self.root = self.root.left
            self.root.parent = None
        else:
            # 왼쪽 서브트리의 최댓값을 찾아 splay → 루트로
            left_tree = self.root.left
            left_tree.parent = None
            right_tree = self.root.right
            right_tree.parent = None
            
            # 왼쪽 서브트리에서 최댓값 찾기
            max_left = left_tree
            while max_left.right:
                max_left = max_left.right
            
            # 임시로 루트를 왼쪽 서브트리로 설정하고 최댓값 splay
            self.root = left_tree
            self._splay(max_left)  # max_left가 왼쪽 서브트리의 루트가 됨
            
            # 오른쪽 서브트리 붙이기
            self.root.right = right_tree
            right_tree.parent = self.root
        
        self._size -= 1
        return True
    
    def inorder(self) -> list:
        result = []
        def _inorder(node):
            if node:
                _inorder(node.left)
                result.append((node.key, node.value))
                _inorder(node.right)
        _inorder(self.root)
        return result
    
    def depth_of(self, key: K) -> int:
        """루트에서 키까지의 깊이"""
        node = self.root
        depth = 0
        while node:
            if key == node.key:
                return depth
            elif key < node.key:
                node = node.left
            else:
                node = node.right
            depth += 1
        return -1


# --- 동작 확인 ---
def demo():
    tree = SplayTree()
    
    # 삽입
    for i in [10, 5, 15, 3, 7, 12, 18]:
        tree.put(i, f"val-{i}")
    
    print("In-order:", [k for k, v in tree.inorder()])
    
    # 탐색: 7에 반복 접근 → 루트 근처로 이동
    print(f"\n7 접근 전 깊이: {tree.depth_of(7)}")
    for _ in range(5):
        tree.get(7)
    print(f"7 반복 접근 후 깊이: {tree.depth_of(7)} (루트={tree.root.key})")
    
    # 15에 접근 → 루트로 이동
    tree.get(15)
    print(f"15 접근 후 루트: {tree.root.key}")
    
    # 삭제
    tree.delete(10)
    print(f"\n10 삭제 후 in-order: {[k for k, v in tree.inorder()]}")

demo()
```

---

## 코드 예제 2: 스플레이 트리로 LRU 캐시 구현

스플레이 트리는 참조 지역성이 있는 데이터에서 LRU(Least Recently Used) 캐시와 유사하게 동작한다. 실제 LRU 캐시도 스플레이 트리로 효율적으로 구현할 수 있다.

```python
from collections import OrderedDict
import time
import random

class SplayCache:
    """
    스플레이 트리 기반 캐시
    - 최근 접근 항목이 자동으로 루트 근처로 이동
    - O(log n) 탐색, O(1) 비슷한 반복 접근 효과
    """
    
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.tree = SplayTree()
        self._access_count = {}
    
    def get(self, key: int) -> Optional[str]:
        self._access_count[key] = self._access_count.get(key, 0) + 1
        result = self.tree.get(key)
        return result
    
    def put(self, key: int, value: str):
        if key not in [k for k, _ in self.tree.inorder()]:
            if len(self.tree) >= self.capacity:
                # 단순화: 가장 깊은 노드(최소 접근) 제거
                # 실제 LRU라면 접근 시간 기반으로 제거
                pass
        self.tree.put(key, value)
    
    def hot_keys(self, top_n: int = 5) -> list:
        """자주 접근된 키 반환"""
        sorted_keys = sorted(
            self._access_count.items(), 
            key=lambda x: x[1], 
            reverse=True
        )
        return sorted_keys[:top_n]

def benchmark_splay_vs_dict(n_ops: int = 100_000):
    """Splay Tree vs Python dict 성능 비교 (지역성 있는 접근 패턴)"""
    
    # 접근 패턴: 80/20 법칙 — 전체 키의 20%가 80% 접근을 받음
    all_keys = list(range(1000))
    hot_keys = all_keys[:200]   # 상위 20%
    cold_keys = all_keys[200:]  # 하위 80%
    
    # 스플레이 트리 초기화
    tree = SplayTree()
    for k in all_keys:
        tree.put(k, f"value-{k}")
    
    # Python dict 초기화
    d = {k: f"value-{k}" for k in all_keys}
    
    # 워크로드 생성: 80% hot, 20% cold
    workload = []
    for _ in range(n_ops):
        if random.random() < 0.8:
            workload.append(random.choice(hot_keys))
        else:
            workload.append(random.choice(cold_keys))
    
    # 스플레이 트리 벤치마크
    start = time.perf_counter()
    hit = 0
    for k in workload:
        if tree.get(k) is not None:
            hit += 1
    splay_time = time.perf_counter() - start
    
    # dict 벤치마크
    start = time.perf_counter()
    for k in workload:
        _ = d.get(k)
    dict_time = time.perf_counter() - start
    
    print(f"Operations: {n_ops:,}")
    print(f"SplayTree: {splay_time:.3f}s ({n_ops/splay_time:,.0f} ops/s)")
    print(f"dict:      {dict_time:.3f}s ({n_ops/dict_time:,.0f} ops/s)")
    print(f"dict is {splay_time/dict_time:.1f}x faster (expected: dict wins)")
    
    # 스플레이 트리의 진짜 장점: 핫 키 접근 깊이
    hot_depths = [tree.depth_of(k) for k in hot_keys[:10]]
    cold_depths = [tree.depth_of(k) for k in cold_keys[:10]]
    print(f"\nHot keys avg depth: {sum(hot_depths)/len(hot_depths):.1f}")
    print(f"Cold keys avg depth: {sum(cold_depths)/len(cold_depths):.1f}")
    print("→ Hot keys가 루트에 더 가까움!")

benchmark_splay_vs_dict()
```

---

## 스플레이 트리의 실제 활용 사례

### 1. GCC의 컴파일러 내부
GCC의 인터프리터 및 링커에서 심볼 테이블 관리에 스플레이 트리를 사용한다. 자주 참조되는 심볼(변수, 함수명)이 루트 근처에 위치하게 되어 반복 컴파일 시 성능이 향상된다.

### 2. Windows NT 커널의 가상 메모리 관리
Windows의 메모리 관리자(Mm)는 VAD(Virtual Address Descriptor) 트리에 스플레이 트리를 사용한다. 최근 접근된 메모리 영역이 루트 근처에 위치해 페이지 폴트 처리 속도를 높인다.

### 3. 네트워크 라우터의 IP 테이블
일부 라우터 구현에서 최근 라우팅된 IP 주소를 스플레이 트리로 관리해 Hot Path(자주 사용되는 경로)를 빠르게 처리한다.

---

## AVL / 레드-블랙 트리 vs. 스플레이 트리 비교

| 속성 | AVL 트리 | 레드-블랙 트리 | 스플레이 트리 |
|------|---------|--------------|-------------|
| 탐색 (균등 분포) | O(log n) | O(log n) | O(log n) 분할 상환 |
| 탐색 (지역성 있음) | O(log n) | O(log n) | O(1) ~ O(log n) |
| 삽입 | O(log n) | O(log n) | O(log n) 분할 상환 |
| 삭제 | O(log n) | O(log n) | O(log n) 분할 상환 |
| 최악 단일 연산 | O(log n) | O(log n) | O(n) |
| 추가 메모리 | 높이 정보 | 색상 비트 | 없음 |
| 구현 복잡도 | 중간 | 높음 | 낮음 |
| 캐시 효율성 | 보통 | 보통 | 우수 (지역성 활용) |

---

## 주의사항과 실전 팁

**1. 최악의 경우 O(n)을 허용할 수 없다면 사용하지 마라**
실시간 시스템이나 응답 시간이 엄격하게 제한된 환경에서는 AVL 또는 레드-블랙 트리가 더 안전하다. 스플레이 트리는 **분할 상환 분석**으로 O(log n)이지 각 연산을 보장하지는 않는다.

**2. 멀티스레드 환경에서 주의하라**
탐색 연산이 트리 구조를 변경(Splay)하므로, 읽기 전용처럼 보이는 `get()`도 실제로는 **쓰기 연산**이다. 읽기-쓰기 잠금(RWLock)으로는 충분하지 않으며, 쓰기 잠금이 필요하다.

**3. 접근 패턴에 따른 성능 차이를 이해하라**
- **Zipf 분포** (80/20 법칙): 스플레이 트리 최적, hot 데이터가 루트에 집중
- **균등 분포**: AVL/RB와 유사, 이점 없음
- **역순 순차 접근** (1, 2, 3, ..., n 순서): O(n²) 총 비용 발생 가능

**4. 구현 시 부모 포인터 관리에 주의하라**
Splay 연산 중 부모 포인터 업데이트를 빠뜨리면 트리 구조가 망가진다. 포인터 업데이트 순서를 신중하게 처리해야 한다. 위의 구현처럼 `__slots__`를 사용해 메모리도 절약하고 실수를 줄이는 것을 권장한다.

**5. 탑-다운(Top-Down) Splay로 성능 개선 가능**
위 구현은 바텀-업(Bottom-Up) 방식이다. Top-Down Splay는 탐색과 회전을 동시에 수행해 부모 포인터가 없어도 동작하고, 캐시 효율도 더 높다. Sleator & Tarjan의 원 논문에서 이 방식을 권장한다.

---

스플레이 트리는 **단순성과 적응성**이 강점이다. 복잡한 균형 조건을 유지하는 대신, 실제 사용 패턴에 적응해 스스로를 최적화한다. 참조 지역성이 강한 애플리케이션에서는 이론적으로 균등 분포를 가정하는 AVL 트리보다 실질적으로 더 빠를 수 있다. 알고리즘의 설계가 "미래를 예측하지 않고 과거로부터 배운다"는 철학을 담고 있는 우아한 자료구조다.

## 참고 자료
- [Splay Tree — Wikipedia](https://en.wikipedia.org/wiki/Splay_tree)
- [Self-Adjusting Binary Search Trees (Sleator & Tarjan, 1985)](https://www.cs.cmu.edu/~sleator/papers/self-adjusting.pdf)
- [Amortized Analysis of Splay Trees — CMU](https://www.cs.cmu.edu/~runmingl/student/kebuladze-splay.pdf)
