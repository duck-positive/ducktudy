---
layout: post
title: "AVL 트리 완전 정복: 자가 균형 이진 탐색 트리의 원조와 회전 연산"
date: 2026-06-28
categories: [cs, computer-science]
tags: [avl-tree, data-structure, binary-search-tree, rotation, balanced-tree, algorithm]
---

이진 탐색 트리(BST)는 검색·삽입·삭제를 평균 O(log n)에 처리하는 우아한 자료구조다. 그러나 정렬된 입력을 받으면 순식간에 편향 트리(skewed tree)로 퇴화해 모든 연산이 O(n)으로 느려진다. 이 문제를 1962년에 Georgy Adelson-Velsky와 Evgenii Landis가 논문 한 편으로 해결했다. 바로 **AVL 트리**다.

## 개념 설명

AVL 트리는 **모든 노드에서 왼쪽 서브트리와 오른쪽 서브트리의 높이 차이(balance factor)가 -1, 0, 1 중 하나를 유지하도록 강제**하는 자가 균형 이진 탐색 트리다.

```
balance_factor(node) = height(left_subtree) - height(right_subtree)
```

이 조건이 위반되는 순간(|balance_factor| > 1), 트리는 **회전(rotation)** 연산을 통해 즉시 균형을 회복한다. 이 엄격한 균형 조건 덕분에 AVL 트리의 높이는 항상 O(log n)으로 보장된다.

### 레드-블랙 트리와의 차이

레드-블랙 트리도 자가 균형 BST지만, AVL 트리보다 느슨한 균형 조건(높이 차이가 최대 2배까지 허용)을 사용한다. 덕분에 삽입·삭제 시 회전 횟수가 적어 **쓰기가 많은 워크로드**에서 유리하다. 반면 AVL 트리는 더 엄격한 균형으로 **검색이 빈번한 워크로드**에서 일반적으로 더 빠르다.

## 왜 필요한가

실제 시스템에서 BST의 편향 문제는 치명적이다.

- **예시**: 1부터 100만까지 순서대로 삽입하면 BST는 100만 깊이의 연결 리스트가 된다.
- 데이터베이스 인덱스가 편향 트리라면 단순 조회가 전체 테이블 스캔과 다를 바 없다.
- AVL 트리는 이 최악의 케이스를 원천 차단한다.

AVL 트리가 적합한 상황:
- 검색 연산이 삽입/삭제보다 압도적으로 많은 경우
- 엄격한 O(log n) 최악 보장이 필요한 실시간 시스템
- 메모리 내 사전(dictionary), 우선순위 기반 검색

## 핵심: 4가지 회전 케이스

불균형 노드를 `z`, 그 자식을 `y`, 손자를 `x`라고 할 때 4가지 경우가 발생한다.

| 케이스 | 조건 | 해결 |
|--------|------|------|
| LL | z의 왼쪽-왼쪽에 삽입 | z를 오른쪽으로 단순 회전 |
| RR | z의 오른쪽-오른쪽에 삽입 | z를 왼쪽으로 단순 회전 |
| LR | z의 왼쪽-오른쪽에 삽입 | y를 왼쪽 회전 후 z를 오른쪽 회전 |
| RL | z의 오른쪽-왼쪽에 삽입 | y를 오른쪽 회전 후 z를 왼쪽 회전 |

## 실제 구현 예제

### 예제 1: Python으로 구현하는 AVL 트리 전체 코드

```python
class Node:
    def __init__(self, key):
        self.key = key
        self.left = None
        self.right = None
        self.height = 1  # 새 노드는 높이 1

class AVLTree:
    def _height(self, node):
        return node.height if node else 0

    def _balance_factor(self, node):
        if not node:
            return 0
        return self._height(node.left) - self._height(node.right)

    def _update_height(self, node):
        node.height = 1 + max(self._height(node.left), self._height(node.right))

    # LL 케이스: 오른쪽 회전
    def _rotate_right(self, z):
        y = z.left
        T3 = y.right

        y.right = z
        z.left = T3

        self._update_height(z)
        self._update_height(y)
        return y  # 새로운 루트

    # RR 케이스: 왼쪽 회전
    def _rotate_left(self, z):
        y = z.right
        T2 = y.left

        y.left = z
        z.right = T2

        self._update_height(z)
        self._update_height(y)
        return y

    def _rebalance(self, node):
        self._update_height(node)
        bf = self._balance_factor(node)

        # LL 케이스
        if bf > 1 and self._balance_factor(node.left) >= 0:
            return self._rotate_right(node)

        # LR 케이스
        if bf > 1 and self._balance_factor(node.left) < 0:
            node.left = self._rotate_left(node.left)
            return self._rotate_right(node)

        # RR 케이스
        if bf < -1 and self._balance_factor(node.right) <= 0:
            return self._rotate_left(node)

        # RL 케이스
        if bf < -1 and self._balance_factor(node.right) > 0:
            node.right = self._rotate_right(node.right)
            return self._rotate_left(node)

        return node

    def insert(self, root, key):
        if not root:
            return Node(key)

        if key < root.key:
            root.left = self.insert(root.left, key)
        elif key > root.key:
            root.right = self.insert(root.right, key)
        else:
            return root  # 중복 키는 무시

        return self._rebalance(root)

    def _min_node(self, node):
        while node.left:
            node = node.left
        return node

    def delete(self, root, key):
        if not root:
            return root

        if key < root.key:
            root.left = self.delete(root.left, key)
        elif key > root.key:
            root.right = self.delete(root.right, key)
        else:
            # 삭제 대상 노드 발견
            if not root.left:
                return root.right
            elif not root.right:
                return root.left
            # 두 자식 모두 있을 때: 중위 후계자(inorder successor)로 대체
            successor = self._min_node(root.right)
            root.key = successor.key
            root.right = self.delete(root.right, successor.key)

        return self._rebalance(root)

    def inorder(self, root, result=None):
        if result is None:
            result = []
        if root:
            self.inorder(root.left, result)
            result.append(root.key)
            self.inorder(root.right, result)
        return result

# 사용 예시
avl = AVLTree()
root = None
for key in [10, 20, 30, 40, 50, 25]:
    root = avl.insert(root, key)

print("중위 순회:", avl.inorder(root))  # [10, 20, 25, 30, 40, 50]
print("트리 높이:", avl._height(root))  # 3 (일반 BST였다면 6)

root = avl.delete(root, 40)
print("40 삭제 후:", avl.inorder(root))  # [10, 20, 25, 30, 50]
```

코드에서 핵심은 `_rebalance` 함수다. 삽입·삭제 후 재귀 호출이 되감기면서(unwinding) 모든 조상 노드에서 `_rebalance`가 호출되어 불균형을 즉시 수정한다.

### 예제 2: 일반 BST와 AVL 트리의 높이 비교 실험

```python
import random

def bst_insert(root, key):
    """균형을 맞추지 않는 일반 BST 삽입"""
    node = Node(key)
    if not root:
        return node
    current = root
    while True:
        if key < current.key:
            if current.left is None:
                current.left = node
                break
            current = current.left
        else:
            if current.right is None:
                current.right = node
                break
            current = current.right
    return root

def bst_height(node):
    if not node:
        return 0
    return 1 + max(bst_height(node.left), bst_height(node.right))

# 순서대로 삽입 (최악의 케이스)
avl = AVLTree()
bst_root = None
avl_root = None

for i in range(1, 16):
    bst_root = bst_insert(bst_root, i)
    avl_root = avl.insert(avl_root, i)

print(f"일반 BST 높이 (1~15 순서 삽입): {bst_height(bst_root)}")  # 15
print(f"AVL 트리 높이 (1~15 순서 삽입): {avl._height(avl_root)}")   # 4

# 무작위 삽입
keys = list(range(1, 1001))
random.shuffle(keys)
bst_root2 = None
avl_root2 = None
for k in keys:
    bst_root2 = bst_insert(bst_root2, k)
    avl_root2 = avl.insert(avl_root2, k)

print(f"\n1000개 랜덤 삽입")
print(f"일반 BST 높이: {bst_height(bst_root2)}")      # 약 20~40 (랜덤)
print(f"AVL 트리 높이: {avl._height(avl_root2)}")      # 10 (ceil(log2(1001)) ≈ 10)
```

실험 결과가 모든 것을 말해준다. 1부터 15까지 순서 삽입 시 일반 BST의 높이는 **15**(연결 리스트), AVL 트리는 **4**다. 1000개 무작위 삽입에서도 AVL 트리는 항상 ceil(log₂ n) 근처의 최소 높이를 보장한다.

## 복잡도 분석

| 연산 | AVL 트리 | 일반 BST 평균 | 일반 BST 최악 |
|------|----------|----------------|----------------|
| 검색 | O(log n) | O(log n) | O(n) |
| 삽입 | O(log n) | O(log n) | O(n) |
| 삭제 | O(log n) | O(log n) | O(n) |
| 공간 | O(n) | O(n) | O(n) |

삽입 시 최대 1번, 삭제 시 O(log n)번의 회전이 발생할 수 있다. 각 회전 자체는 O(1)이다.

## 주의사항과 팁

**1. 높이 필드는 반드시 즉시 갱신하라**
회전 후 높이를 업데이트할 때 자식 노드를 먼저, 부모 노드를 나중에 갱신해야 한다. 순서를 뒤집으면 잘못된 높이로 회전 여부를 판단하게 된다.

**2. 삭제의 중위 후계자 선택**
삭제 시 두 자식이 모두 있는 경우, 오른쪽 서브트리의 최솟값(중위 후계자)으로 대체하는 것이 일반적이다. 균형 유지 측면에서 왼쪽 서브트리의 최댓값(중위 선행자)을 선택해도 동일한 결과지만, 일관성을 위해 한 가지 방식을 고수하는 것이 좋다.

**3. 재귀 vs 반복 구현**
재귀 구현이 직관적이지만, 깊이가 매우 깊어질 경우(n > 수백만) 스택 오버플로우 위험이 있다. 이 경우 부모 포인터를 저장하는 반복 구현을 고려하라.

**4. 언제 AVL 대신 레드-블랙 트리를 쓸까?**
- 삽입/삭제가 검색보다 많다면 레드-블랙 트리가 유리하다(회전 횟수 적음)
- Java의 `TreeMap`, C++의 `std::map`은 레드-블랙 트리를 사용
- 검색 집중적인 임베디드 시스템이나 DB 인덱스에는 AVL이 적합

AVL 트리는 자가 균형 BST의 시초로, 레드-블랙 트리·B트리 같은 현대적 자료구조의 철학적 뿌리다. 회전 메커니즘을 깊이 이해하면 균형 트리 전반의 동작 원리가 명쾌하게 보인다.

## 참고 자료
- [AVL Tree - Wikipedia](https://en.wikipedia.org/wiki/AVL_tree)
- [Introduction to AVL Tree - GeeksforGeeks](https://www.geeksforgeeks.org/introduction-to-avl-tree/)
- [AVL Trees - TutorialsPoint](https://www.tutorialspoint.com/data_structures_algorithms/avl_tree_algorithm.htm)
- [Lecture 16: AVL Trees - CMU 15-122](https://www.cs.cmu.edu/~15122/handouts/lectures/16-avl.pdf)
