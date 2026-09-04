---
layout: post
title: "스플레이 트리(Splay Tree) 완전 정복: 자기 조정 BST의 분할 상환 분석과 구현"
date: 2026-09-04
categories: [cs, computer-science]
tags: [splay-tree, data-structure, binary-search-tree, amortized-analysis, algorithm]
---

## 스플레이 트리란 무엇인가

스플레이 트리(Splay Tree)는 1985년 Daniel Sleator와 Robert Tarjan이 발표한 **자기 조정(self-adjusting) 이진 탐색 트리**입니다. AVL 트리나 레드-블랙 트리가 추가적인 균형 정보(높이, 색상 플래그 등)를 노드에 저장하며 엄격한 균형을 유지하는 것과 달리, 스플레이 트리는 그런 보조 정보 없이 동작합니다.

스플레이 트리의 핵심 아이디어는 단순합니다. **최근에 접근한 노드를 루트 근처로 끌어올린다**는 것입니다. 이 "스플레이(splay)" 연산 덕분에 자주 접근하는 원소는 항상 루트 가까이에 위치하게 되고, 반복 접근 시 O(1)에 가까운 성능을 발휘합니다.

어떤 순간의 트리 형태는 최악의 경우 연결 리스트처럼 깊어질 수 있지만, **m번의 연산에 대해 분할 상환(amortized) O(m log n)** 을 보장합니다. 즉, 개별 연산은 비싸더라도 전체 평균은 O(log n)입니다.

---

## 왜 스플레이 트리가 필요한가

### 작업 집합 속성 (Working Set Property)

실제 응용에서 데이터 접근 패턴은 균일하지 않습니다. 전체 원소 중 일부(작업 집합)에 반복적으로 접근하는 경향이 있습니다. 캐시, 텍스트 편집기의 최근 명령어, 언어 파서의 자주 쓰는 심볼 테이블이 좋은 예입니다.

스플레이 트리는 이런 지역성(locality)을 자동으로 포착합니다. t번 전에 마지막으로 접근한 원소 x에 접근하는 비용은 O(log t)입니다. 이를 **작업 집합 정리(Working Set Theorem)** 라 하며, 이론적으로 최적 정적 트리보다도 우수할 수 있습니다.

### 구현의 단순성

AVL 트리와 레드-블랙 트리는 정확한 구현이 까다롭습니다. 스플레이 트리는 노드에 추가 필드가 필요 없고, 스플레이라는 단일 연산으로 삽입/삭제/탐색을 모두 처리합니다. 코드 크기가 작고 실수할 여지가 줄어듭니다.

### 캐시 친화성

최근에 접근한 노드들이 루트 근처에 모여 있기 때문에, 연속 접근 시 CPU 캐시 히트율이 높아집니다. 포인터 추적 깊이가 줄어들어 메모리 계층 구조를 잘 활용합니다.

---

## 스플레이 연산의 세 가지 케이스

스플레이는 노드 x를 루트로 끌어올리는 일련의 회전(rotation) 연산입니다. x의 현재 위치에 따라 세 가지 케이스로 나뉩니다.

**Zig**: x의 부모 p가 루트일 때, x를 향해 한 번 회전합니다.

**Zig-Zig**: x와 p가 같은 방향 자식(둘 다 왼쪽, 또는 둘 다 오른쪽)일 때, **p를 먼저** 조부모 방향으로 회전한 뒤, **x를 다시** 회전합니다. 단순히 x를 두 번 회전하는 것과 달리, p를 먼저 회전함으로써 균형 특성이 개선됩니다.

**Zig-Zag**: x와 p가 다른 방향 자식(x는 오른쪽, p는 왼쪽, 혹은 그 반대)일 때, **x를 두 번** 회전합니다. 처음 회전 후 x가 p 자리로 올라가고, 두 번째 회전으로 x가 조부모 자리까지 올라갑니다.

---

## 실제 구현 예제

### 예제 1: 스플레이 트리 Python 구현

```python
class Node:
    def __init__(self, key):
        self.key = key
        self.left = None
        self.right = None
        self.parent = None

class SplayTree:
    def __init__(self):
        self.root = None

    def _rotate_right(self, x):
        y = x.left
        x.left = y.right
        if y.right:
            y.right.parent = x
        y.parent = x.parent
        if not x.parent:
            self.root = y
        elif x == x.parent.right:
            x.parent.right = y
        else:
            x.parent.left = y
        y.right = x
        x.parent = y

    def _rotate_left(self, x):
        y = x.right
        x.right = y.left
        if y.left:
            y.left.parent = x
        y.parent = x.parent
        if not x.parent:
            self.root = y
        elif x == x.parent.left:
            x.parent.left = y
        else:
            x.parent.right = y
        y.left = x
        x.parent = y

    def _splay(self, x):
        while x.parent:
            p = x.parent
            g = p.parent  # 조부모
            if not g:
                # Zig: 부모가 루트
                if x == p.left:
                    self._rotate_right(p)
                else:
                    self._rotate_left(p)
            elif x == p.left and p == g.left:
                # Zig-Zig (왼쪽-왼쪽)
                self._rotate_right(g)
                self._rotate_right(p)
            elif x == p.right and p == g.right:
                # Zig-Zig (오른쪽-오른쪽)
                self._rotate_left(g)
                self._rotate_left(p)
            elif x == p.right and p == g.left:
                # Zig-Zag (오른쪽-왼쪽)
                self._rotate_left(p)
                self._rotate_right(g)
            else:
                # Zig-Zag (왼쪽-오른쪽)
                self._rotate_right(p)
                self._rotate_left(g)

    def insert(self, key):
        node = Node(key)
        if not self.root:
            self.root = node
            return
        cur = self.root
        while True:
            if key < cur.key:
                if not cur.left:
                    cur.left = node
                    node.parent = cur
                    break
                cur = cur.left
            elif key > cur.key:
                if not cur.right:
                    cur.right = node
                    node.parent = cur
                    break
                cur = cur.right
            else:
                return  # 중복 키 무시
        self._splay(node)

    def search(self, key):
        cur = self.root
        last = None
        while cur:
            last = cur
            if key == cur.key:
                self._splay(cur)
                return True
            elif key < cur.key:
                cur = cur.left
            else:
                cur = cur.right
        if last:
            self._splay(last)  # 탐색 실패 시 마지막 노드 스플레이
        return False

    def delete(self, key):
        if not self.search(key):
            return  # 루트로 탐색 대상이 올라옴
        # 루트 = 삭제 대상
        left_tree = self.root.left
        right_tree = self.root.right
        if left_tree:
            left_tree.parent = None
        if right_tree:
            right_tree.parent = None
        if not left_tree:
            self.root = right_tree
        elif not right_tree:
            self.root = left_tree
        else:
            # 왼쪽 트리의 최댓값을 루트로 스플레이
            self.root = left_tree
            cur = left_tree
            while cur.right:
                cur = cur.right
            self._splay(cur)
            self.root.right = right_tree
            right_tree.parent = self.root

    def inorder(self, node=None, result=None):
        if result is None:
            result = []
            node = self.root
        if node:
            self.inorder(node.left, result)
            result.append(node.key)
            self.inorder(node.right, result)
        return result


# 사용 예
tree = SplayTree()
for k in [5, 3, 7, 1, 4, 6, 8]:
    tree.insert(k)

print("중위 순회:", tree.inorder())  # [1, 3, 4, 5, 6, 7, 8]

tree.search(1)
print("1 접근 후 루트:", tree.root.key)  # 1이 루트로 올라옴

tree.delete(5)
print("5 삭제 후:", tree.inorder())  # [1, 3, 4, 6, 7, 8]
```

### 예제 2: 분할 상환 비용 시각화 — 반복 접근 패턴

```python
import random
import time

def benchmark_splay_vs_bst(n=10000, queries=50000, hot_ratio=0.8):
    """
    스플레이 트리 vs 일반 BST의 지역성 처리 비교
    hot_ratio: 전체 쿼리 중 상위 10%의 키에 집중되는 비율
    """
    tree = SplayTree()
    keys = list(range(1, n + 1))
    random.shuffle(keys)
    for k in keys:
        tree.insert(k)

    hot_keys = keys[:n // 10]    # 상위 10% 키
    cold_keys = keys[n // 10:]   # 나머지 90% 키

    # 쿼리 생성: hot_ratio 확률로 hot_keys에서 선택
    query_list = []
    for _ in range(queries):
        if random.random() < hot_ratio:
            query_list.append(random.choice(hot_keys))
        else:
            query_list.append(random.choice(cold_keys))

    # 스플레이 트리 탐색
    t0 = time.perf_counter()
    for q in query_list:
        tree.search(q)
    splay_time = time.perf_counter() - t0

    print(f"원소 수: {n}, 쿼리 수: {queries}")
    print(f"핫 키 비율: {hot_ratio*100:.0f}%")
    print(f"스플레이 트리 탐색 시간: {splay_time*1000:.2f}ms")
    print(f"평균 쿼리 시간: {splay_time/queries*1e6:.2f}μs")

    # 핫 키의 현재 루트까지 깊이 확인
    def depth(tree, key):
        cur = tree.root
        d = 0
        while cur:
            if key == cur.key:
                return d
            elif key < cur.key:
                cur = cur.left
            else:
                cur = cur.right
            d += 1
        return -1

    avg_hot_depth = sum(depth(tree, k) for k in hot_keys[:5]) / 5
    print(f"핫 키 평균 깊이: {avg_hot_depth:.1f} (자주 접근할수록 루트에 가까워짐)")

benchmark_splay_vs_bst()
```

---

## 분할 상환 분석: 포텐셜 함수 접근법

스플레이 트리의 분할 상환 분석은 **포텐셜 함수 Φ**를 이용합니다. 각 노드 x에 대해 **rank**를 다음과 같이 정의합니다.

```
size(x) = x를 루트로 하는 서브트리의 노드 수
rank(x) = floor(log2(size(x)))
Φ = Σ rank(x) (모든 노드에 대한 합)
```

이 정의 하에, 스플레이 연산의 분할 상환 비용은 **최대 1 + 3(rank(root) - rank(x))** 임을 증명할 수 있습니다. n개의 노드가 있을 때 rank(root) ≤ log₂(n)이므로, 분할 상환 비용은 O(log n)입니다.

**직관적 설명**: 스플레이가 트리를 불균형하게 만들면(포텐셜 증가) 다음 연산들이 더 빠르게 처리됩니다(실제 비용 감소). 반대로, 스플레이로 트리가 균형 잡히면(포텐셜 감소) 그 감소분이 실제 비용의 일부를 상쇄합니다.

---

## 주의사항 및 실전 팁

### 멀티스레딩 환경에서의 주의

스플레이 트리는 **읽기 연산도 트리 구조를 변경**합니다(스플레이). 따라서 다중 스레드 환경에서 읽기-쓰기 락(reader-writer lock)만으로는 부족합니다. 읽기 연산도 독점 잠금(exclusive lock)이 필요합니다. 동시성이 중요한 환경에서는 다른 자료구조를 고려하세요.

### 최악 케이스 단일 연산

개별 연산은 최악 O(n)이 될 수 있습니다. 실시간 시스템처럼 **단일 연산의 최악 케이스 시간이 중요**한 경우 스플레이 트리는 적합하지 않습니다. 이런 경우 AVL 트리나 레드-블랙 트리가 적합합니다.

### 균일 접근 패턴일 때의 성능

접근 패턴에 지역성이 없고 완전히 균일하다면(uniformly random), 스플레이 트리는 레드-블랙 트리보다 상수 배 느릴 수 있습니다. 스플레이가 최적인 상황은 **편향된(skewed) 접근 패턴**이 존재할 때입니다.

### 활용 사례

- **캐시/메모이제이션**: 최근 접근 원소가 빠르게 재접근되는 경우
- **텍스트 편집기의 piece table**: 연속된 구간 편집에 최적
- **네트워크 라우팅 테이블**: 자주 접근되는 경로가 빠르게 처리됨
- **링크-컷 트리(Link-Cut Tree)**: 스플레이 트리를 내부 구조로 사용하는 고급 자료구조

---

## 참고 자료

- [Splay Tree - Wikipedia](https://en.wikipedia.org/wiki/Splay_tree)
- [Self-Adjusting Binary Search Trees - Sleator & Tarjan (1985)](https://www.cs.cmu.edu/~sleator/papers/self-adjusting.pdf)
- [Splay Trees - CP Algorithms](https://cp-algorithms.com/data_structures/splay_tree.html)
- [Guide to Splay Trees - Baeldung CS](https://www.baeldung.com/cs/splay-trees)
