---
layout: post
title: "Treap 완전 정복: BST와 힙의 결합으로 만드는 확률적 균형 이진 탐색 트리"
date: 2026-07-07
categories: [cs, computer-science]
tags: [treap, bst, heap, data-structure, algorithm, randomized, balanced-tree, split-merge]
---

이진 탐색 트리(BST)의 가장 큰 약점은 편향(skew)이다. 정렬된 순서로 원소를 삽입하면 트리가 연결 리스트처럼 변해 O(n) 탐색으로 퇴화한다. AVL 트리와 레드-블랙 트리는 명시적인 회전 연산으로 균형을 강제하지만, 구현이 복잡하다. **Treap**(트립)은 무작위 우선순위를 도입하여 확률적으로 균형을 보장하는 놀랍도록 우아한 해법이다. 기대 높이가 O(log n)이고, 구현은 AVL 트리의 절반 수준이다.

## 핵심 아이디어: 두 가지 성질의 공존

Treap의 각 노드는 두 가지 값을 가진다:
- **key**: BST 성질을 만족 (왼쪽 서브트리 < key < 오른쪽 서브트리)
- **priority**: 힙 성질을 만족 (부모의 priority ≥ 자식의 priority, max-heap)

priority는 삽입 시 **무작위로 배정**된다. 이 두 성질이 동시에 만족되는 트리는 주어진 key 집합에 대해 **유일하게 결정**된다(동점 없다면). 무작위 priority로 인해 이 트리는 n개의 무작위 키를 삽입한 BST와 동일한 분포를 가지며, 기대 높이는 O(log n)이다.

```
삽입 순서: 5, 3, 7, 1, 4 (priority는 무작위 배정)

key=5, pri=9
├── key=3, pri=7
│   ├── key=1, pri=4
│   └── key=4, pri=6
└── key=7, pri=8

BST 성질: inorder 순회 → 1, 3, 4, 5, 7 ✓
힙 성질: 부모 priority ≥ 자식 priority ✓
```

## 두 가지 구현 방식

Treap은 **회전 기반**과 **분할-병합(split-merge) 기반** 두 가지 방식으로 구현할 수 있다. 분할-병합 방식이 더 강력하고 응용 범위가 넓으므로 두 방식 모두 살펴본다.

### 방식 1: 회전 기반 Treap

```python
import random

class TreapNode:
    def __init__(self, key):
        self.key = key
        self.priority = random.random()  # 무작위 우선순위 [0, 1)
        self.left = None
        self.right = None
        self.size = 1  # 서브트리 크기 (선택적)


class RotationTreap:
    def __init__(self):
        self.root = None

    def _size(self, node):
        return node.size if node else 0

    def _update(self, node):
        if node:
            node.size = 1 + self._size(node.left) + self._size(node.right)

    def _rotate_right(self, y):
        """y를 오른쪽으로 회전. x가 새 루트."""
        x = y.left
        y.left = x.right
        x.right = y
        self._update(y)
        self._update(x)
        return x

    def _rotate_left(self, x):
        """x를 왼쪽으로 회전. y가 새 루트."""
        y = x.right
        x.right = y.left
        y.left = x
        self._update(x)
        self._update(y)
        return y

    def _insert(self, node, key):
        if node is None:
            return TreapNode(key)

        if key < node.key:
            node.left = self._insert(node.left, key)
            if node.left.priority > node.priority:
                node = self._rotate_right(node)
        elif key > node.key:
            node.right = self._insert(node.right, key)
            if node.right.priority > node.priority:
                node = self._rotate_left(node)
        # key == node.key: 중복 무시 (또는 카운터 증가)

        self._update(node)
        return node

    def _delete(self, node, key):
        if node is None:
            return None

        if key < node.key:
            node.left = self._delete(node.left, key)
        elif key > node.key:
            node.right = self._delete(node.right, key)
        else:
            # 삭제할 노드 발견: 자식이 없을 때까지 아래로 회전
            if node.left is None:
                return node.right
            elif node.right is None:
                return node.left
            elif node.left.priority > node.right.priority:
                node = self._rotate_right(node)
                node.right = self._delete(node.right, key)
            else:
                node = self._rotate_left(node)
                node.left = self._delete(node.left, key)

        self._update(node)
        return node

    def _search(self, node, key):
        if node is None:
            return False
        if key == node.key:
            return True
        elif key < node.key:
            return self._search(node.left, key)
        else:
            return self._search(node.right, key)

    def insert(self, key): self.root = self._insert(self.root, key)
    def delete(self, key): self.root = self._delete(self.root, key)
    def search(self, key): return self._search(self.root, key)

    def inorder(self):
        result = []
        def _inorder(node):
            if node:
                _inorder(node.left)
                result.append(node.key)
                _inorder(node.right)
        _inorder(self.root)
        return result


# 테스트
treap = RotationTreap()
for k in [5, 3, 7, 1, 4, 6, 8, 2]:
    treap.insert(k)

print(treap.inorder())      # [1, 2, 3, 4, 5, 6, 7, 8]
print(treap.search(4))      # True
print(treap.search(9))      # False
treap.delete(5)
print(treap.inorder())      # [1, 2, 3, 4, 6, 7, 8]
```

### 방식 2: 분할-병합 기반 Treap (더 강력)

분할-병합 방식은 두 핵심 연산으로 모든 것을 처리한다:
- **split(T, k)**: 트리 T를 key ≤ k인 부분과 key > k인 부분으로 분리
- **merge(L, R)**: 두 트리를 하나로 합병 (L의 모든 key < R의 모든 key 가정)

```python
import random

class SplitMergeTreapNode:
    __slots__ = ['key', 'priority', 'left', 'right', 'size']

    def __init__(self, key):
        self.key = key
        self.priority = random.getrandbits(32)  # 정수 우선순위가 비교가 더 빠름
        self.left = None
        self.right = None
        self.size = 1


def size(node):
    return node.size if node else 0

def update(node):
    if node:
        node.size = 1 + size(node.left) + size(node.right)


def split(node, key):
    """
    key를 기준으로 트리를 분할한다.
    반환: (key <= key 인 트리, key > key 인 트리)
    """
    if node is None:
        return None, None

    if node.key <= key:
        # 루트는 왼쪽 부분에 속함
        left, right = split(node.right, key)
        node.right = left
        update(node)
        return node, right
    else:
        # 루트는 오른쪽 부분에 속함
        left, right = split(node.left, key)
        node.left = right
        update(node)
        return left, node


def merge(left, right):
    """
    두 트리를 병합한다 (left의 모든 key < right의 모든 key).
    우선순위가 높은 쪽이 루트가 된다.
    """
    if left is None:
        return right
    if right is None:
        return left

    if left.priority > right.priority:
        left.right = merge(left.right, right)
        update(left)
        return left
    else:
        right.left = merge(left, right.left)
        update(right)
        return right


class SplitMergeTreap:
    def __init__(self):
        self.root = None

    def insert(self, key):
        """split 후 새 노드 삽입, merge로 재결합."""
        left, right = split(self.root, key - 1)
        new_node = SplitMergeTreapNode(key)
        self.root = merge(merge(left, new_node), right)

    def delete(self, key):
        """key를 기준으로 3분할 후 해당 노드를 제외하고 merge."""
        left, mid_right = split(self.root, key - 1)
        _, right = split(mid_right, key)  # mid에서 key 노드 버림
        self.root = merge(left, right)

    def search(self, key):
        node = self.root
        while node:
            if key == node.key:
                return True
            elif key < node.key:
                node = node.left
            else:
                node = node.right
        return False

    def kth_smallest(self, k):
        """k번째 작은 원소 (1-indexed). size 필드 활용."""
        node = self.root
        while node:
            left_size = size(node.left)
            if k == left_size + 1:
                return node.key
            elif k <= left_size:
                node = node.left
            else:
                k -= left_size + 1
                node = node.right
        return None

    def count_less_than(self, key):
        """key보다 작은 원소의 수."""
        left, right = split(self.root, key - 1)
        result = size(left)
        self.root = merge(left, right)
        return result

    def range_query(self, lo, hi):
        """[lo, hi] 범위의 원소 개수."""
        left, mid_right = split(self.root, lo - 1)
        mid, right = split(mid_right, hi)
        result = size(mid)
        self.root = merge(left, merge(mid, right))
        return result

    def inorder(self):
        result = []
        def _inorder(node):
            if node:
                _inorder(node.left)
                result.append(node.key)
                _inorder(node.right)
        _inorder(self.root)
        return result


# 테스트
t = SplitMergeTreap()
for k in [5, 3, 7, 1, 4, 6, 8, 2]:
    t.insert(k)

print(t.inorder())              # [1, 2, 3, 4, 5, 6, 7, 8]
print(t.kth_smallest(3))        # 3
print(t.kth_smallest(6))        # 6
print(t.count_less_than(5))     # 4 (1, 2, 3, 4)
print(t.range_query(3, 6))      # 4 (3, 4, 5, 6)
t.delete(4)
print(t.inorder())              # [1, 2, 3, 5, 6, 7, 8]
print(t.kth_smallest(3))        # 3
print(t.kth_smallest(4))        # 5 (4가 삭제됨)
```

## Treap의 균형 보장: 수학적 근거

**정리**: 무작위 priority를 사용하는 Treap의 기대 높이는 O(log n)이다.

**직관적 증명**: priority가 모두 다르다면(확률 1로 성립), Treap의 구조는 key를 priority 내림차순으로 삽입했을 때의 BST와 동일하다. 무작위 priority는 key의 삽입 순서를 무작위화하는 효과를 가진다. n개의 원소를 무작위 순서로 삽입한 BST의 기대 높이는 약 **2.5 ln n**이다.

이는 결정론적 균형 트리(AVL: 1.44 log₂n, Red-Black: 2 log₂n)보다 약간 높지만, 실제 성능은 유사하며 구현 복잡도가 훨씬 낮다.

| 자료구조 | 기대 높이 | 삽입 구현 복잡도 | 삭제 구현 복잡도 |
|---------|----------|---------------|---------------|
| 일반 BST (최악) | O(n) | 낮음 | 낮음 |
| AVL Tree | 1.44 log₂n | 높음 (4가지 회전) | 높음 |
| Red-Black Tree | 2 log₂n | 매우 높음 | 매우 높음 |
| Treap | ~2.5 ln n | 중간 | 중간 |
| Skip List | O(log n) | 중간 | 중간 |

## 응용: 암묵적 Treap (Implicit Treap)

**암묵적 Treap**은 key 대신 **서브트리 크기(인덱스)**를 BST 성질로 사용한다. 배열처럼 인덱스로 접근하면서도 O(log n)에 원소 삽입/삭제/역전/이동이 가능하다. 이는 일반 배열로는 불가능한 기능이다.

```python
def split_by_size(node, k):
    """앞에서 k개의 원소와 나머지를 분리한다."""
    if node is None:
        return None, None

    left_size = size(node.left)

    if left_size >= k:
        left, right = split_by_size(node.left, k)
        node.left = right
        update(node)
        return left, node
    else:
        left, right = split_by_size(node.right, k - left_size - 1)
        node.right = left
        update(node)
        return node, right


def reverse_range(root, l, r):
    """
    인덱스 [l, r] 범위를 역전시킨다 (lazy propagation 필요).
    여기서는 개념 시연용 단순 버전.
    """
    left, mid_right = split_by_size(root, l)
    mid, right = split_by_size(mid_right, r - l + 1)

    # mid 트리 역전 (lazy flip으로 O(1) 가능)
    def flip(node):
        if node:
            node.left, node.right = node.right, node.left
            flip(node.left)
            flip(node.right)
    flip(mid)

    return merge(left, merge(mid, right))
```

암묵적 Treap은 문자열 편집 자료구조, 오프라인 LCT(Link-Cut Tree) 대체, 구간 뒤집기/이동 등 복잡한 시퀀스 연산에 활용된다.

## 주의사항과 팁

**1. 재귀 스택 오버플로우**: Python의 기본 재귀 깊이 제한(1000)에 걸릴 수 있다. n > 10만이면 `sys.setrecursionlimit`을 늘리거나 반복적(iterative) 버전을 구현하라.

**2. 동점 priority 처리**: 같은 priority를 가진 노드가 생기면 BST+힙 성질이 충돌한다. 64비트 무작위 정수를 사용하면 충돌 확률이 거의 0에 가깝다.

**3. split의 부등호 방향**: `split(node, key)`에서 "key 이하"와 "key 미만"을 명확히 정의하라. 중복 키 처리에 따라 부등호가 달라진다.

**4. Treap vs 스킵 리스트**: 두 자료구조 모두 확률적 균형을 사용하며 유사한 복잡도를 가진다. Treap은 순서 통계(order statistics)와 분할-병합 연산이 더 자연스럽고, 스킵 리스트는 동시성(concurrent) 환경에서 더 유리하다.

**5. 실무 활용**: Treap은 경쟁 프로그래밍에서 폭넓게 사용된다. 실제 시스템에서는 레드-블랙 트리(C++ STL, Java TreeMap)나 B+트리(데이터베이스)가 더 보편적이지만, 유연한 분할-병합이 필요한 경우 Treap이 강점을 발휘한다.

Treap은 "무작위성을 도구로 사용하는 알고리즘 설계" 철학의 훌륭한 예시다. 복잡한 케이스 분석 대신 확률의 힘을 빌려 단순하면서도 효율적인 구조를 만들어낸다.

## 참고 자료
- [Treap — Algorithms for Competitive Programming (cp-algorithms.com)](https://cp-algorithms.com/data_structures/treap.html)
- [Treap Data Structures — Baeldung on Computer Science](https://www.baeldung.com/cs/treaps-data-structure)
- [7.2 Treap: A Randomized Binary Search Tree — Open Data Structures](https://opendatastructures.org/ods-java/7_2_Treap_Randomized_Binary.html)
- [Treap (A Randomized Binary Search Tree) — GeeksforGeeks](https://www.geeksforgeeks.org/dsa/treap-a-randomized-binary-search-tree/)
