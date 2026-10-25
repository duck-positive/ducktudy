---
layout: post
title: "스플레이 트리: 자가 조정 이진 탐색 트리의 원리와 구현"
date: 2026-10-24
categories: [cs, computer-science]
tags: [splay-tree, binary-search-tree, data-structure, amortized-analysis, algorithm]
---

## 개요

스플레이 트리(Splay Tree)는 1985년 Daniel Sleator와 Robert Tarjan이 제안한 자가 조정(Self-Adjusting) 이진 탐색 트리입니다. AVL 트리나 레드-블랙 트리와 달리 명시적인 균형 정보를 노드에 저장하지 않으며, 대신 **최근에 접근한 요소를 루트 근처로 이동**시키는 스플레이(Splay) 연산을 통해 동적으로 균형을 유지합니다.

가장 큰 특징은 **지역성(Locality)** 활용입니다. 실제 애플리케이션에서 데이터 접근 패턴은 균등하지 않고, 최근 접근한 데이터가 곧 다시 접근될 가능성이 높습니다(Temporal Locality). 스플레이 트리는 이 성질을 구조적으로 활용하여 반복 접근 시 O(1)에 가까운 성능을 냅니다.

---

## 왜 스플레이 트리가 필요한가?

### 기존 자료구조의 한계

AVL 트리와 레드-블랙 트리는 모든 연산에 O(log n)을 보장하지만 다음과 같은 한계가 있습니다:

- **고정된 균형 비용**: 접근 패턴에 무관하게 항상 같은 비용을 지불합니다.
- **추가 메모리**: 각 노드에 균형 인수(balance factor) 또는 색(color) 정보를 저장합니다.
- **구현 복잡도**: 회전 케이스가 많아 구현이 복잡합니다.

### 스플레이 트리의 장점

1. **상각 O(log n)**: 단일 연산은 O(n)이 될 수 있지만 m번 연산의 총 비용은 O(m log n)입니다.
2. **캐시 친화적**: 자주 접근하는 요소가 루트 근처에 위치합니다.
3. **단순한 구현**: 별도의 균형 정보 없이 회전만으로 동작합니다.
4. **구간 연산 지원**: 분리(Split)와 합치기(Merge) 연산이 효율적입니다.

---

## 핵심 연산: Splay

스플레이 연산은 목표 노드 `x`를 루트로 올리는 과정입니다. 세 가지 케이스로 나뉩니다.

### Zig (1단계 회전)

`x`의 부모 `p`가 루트인 경우, 단순 회전으로 `x`를 루트로 올립니다.

```
    p            x
   / \          / \
  x   C   →   A   p
 / \              / \
A   B            B   C
```

### Zig-Zig (같은 방향 2단계 회전)

`x`와 부모 `p`가 조부모 `g`에 대해 같은 방향(둘 다 왼쪽 또는 둘 다 오른쪽)인 경우:

```
      g              x
     / \            / \
    p   D          A   p
   / \                / \
  x   C              B   g
 / \                    / \
A   B                  C   D
```

이 경우 **g를 먼저 회전**시키고 그 다음 p를 회전시킵니다. 단순히 x를 두 번 올리는 것(Zig-Zag x2)보다 트리가 더 균형잡힙니다.

### Zig-Zag (다른 방향 2단계 회전)

`x`가 `p`의 오른쪽 자식이고 `p`가 `g`의 왼쪽 자식인 경우(또는 그 반대):

```
    g              x
   / \            / \
  p   D          p   g
 / \            / \ / \
A   x          A  B C  D
   / \
  B   C
```

이 경우 `x`를 두 번 연속 회전시킵니다.

---

## 구현 예제 1: C++ 스플레이 트리

```cpp
#include <iostream>
using namespace std;

struct Node {
    int key;
    Node *left, *right, *parent;
    Node(int k) : key(k), left(nullptr), right(nullptr), parent(nullptr) {}
};

class SplayTree {
    Node* root;

    void setParent(Node* child, Node* par) {
        if (child) child->parent = par;
    }

    void rotateRight(Node* x) {
        Node* p = x->parent;
        Node* g = p ? p->parent : nullptr;
        Node* b = x->right;

        x->right = p;
        p->left = b;

        setParent(b, p);
        setParent(p, x);
        setParent(x, g);

        if (g) {
            if (g->left == p) g->left = x;
            else g->right = x;
        }
    }

    void rotateLeft(Node* x) {
        Node* p = x->parent;
        Node* g = p ? p->parent : nullptr;
        Node* b = x->left;

        x->left = p;
        p->right = b;

        setParent(b, p);
        setParent(p, x);
        setParent(x, g);

        if (g) {
            if (g->left == p) g->left = x;
            else g->right = x;
        }
    }

    void splay(Node* x) {
        while (x->parent != nullptr) {
            Node* p = x->parent;
            Node* g = p->parent;

            if (g == nullptr) {
                // Zig
                if (p->left == x) rotateRight(x);
                else rotateLeft(x);
            } else if (g->left == p && p->left == x) {
                // Zig-Zig (left-left)
                rotateRight(p);
                rotateRight(x);
            } else if (g->right == p && p->right == x) {
                // Zig-Zig (right-right)
                rotateLeft(p);
                rotateLeft(x);
            } else if (g->left == p && p->right == x) {
                // Zig-Zag (left-right)
                rotateLeft(x);
                rotateRight(x);
            } else {
                // Zig-Zag (right-left)
                rotateRight(x);
                rotateLeft(x);
            }
        }
        root = x;
    }

    Node* findNode(int key) {
        Node* cur = root;
        Node* last = nullptr;
        while (cur) {
            last = cur;
            if (key == cur->key) break;
            else if (key < cur->key) cur = cur->left;
            else cur = cur->right;
        }
        if (last) splay(last);
        return (root && root->key == key) ? root : nullptr;
    }

public:
    SplayTree() : root(nullptr) {}

    void insert(int key) {
        if (!root) {
            root = new Node(key);
            return;
        }
        Node* cur = root;
        while (true) {
            if (key == cur->key) { splay(cur); return; }
            if (key < cur->key) {
                if (!cur->left) {
                    cur->left = new Node(key);
                    cur->left->parent = cur;
                    splay(cur->left);
                    return;
                }
                cur = cur->left;
            } else {
                if (!cur->right) {
                    cur->right = new Node(key);
                    cur->right->parent = cur;
                    splay(cur->right);
                    return;
                }
                cur = cur->right;
            }
        }
    }

    bool search(int key) {
        return findNode(key) != nullptr;
    }

    void inorder(Node* node) {
        if (!node) return;
        inorder(node->left);
        cout << node->key << " ";
        inorder(node->right);
    }

    void print() { inorder(root); cout << "\n"; }
    int getRoot() { return root ? root->key : -1; }
};

int main() {
    SplayTree st;
    for (int x : {10, 20, 30, 40, 50}) st.insert(x);
    
    cout << "Inorder: ";
    st.print();
    
    st.search(20);
    cout << "Root after search(20): " << st.getRoot() << "\n";
    
    st.search(20);
    st.search(20);
    cout << "Root after 2 more search(20): " << st.getRoot() << "\n";
    
    return 0;
}
```

실행 결과:
```
Inorder: 10 20 30 40 50
Root after search(20): 20
Root after 2 more search(20): 20
```

`search(20)`을 반복할수록 20이 루트에 유지됩니다. 이것이 스플레이 트리의 지역성 활용입니다.

---

## 구현 예제 2: Python 구간 합 스플레이 트리

스플레이 트리는 구간 연산에 특히 강합니다. Split/Merge를 이용해 구간 합을 O(log n) 상각 시간에 처리합니다.

```python
class SplayNode:
    def __init__(self, val):
        self.val = val
        self.sum = val
        self.size = 1
        self.left = None
        self.right = None
        self.parent = None

def update(node):
    if not node:
        return
    node.sum = node.val
    node.size = 1
    if node.left:
        node.sum += node.left.sum
        node.size += node.left.size
    if node.right:
        node.sum += node.right.sum
        node.size += node.right.size

def set_parent(child, par):
    if child:
        child.parent = par

def rotate(x):
    p = x.parent
    g = p.parent if p else None

    if p.left == x:
        p.left = x.right
        set_parent(x.right, p)
        x.right = p
    else:
        p.right = x.left
        set_parent(x.left, p)
        x.left = p

    set_parent(p, x)
    set_parent(x, g)

    if g:
        if g.left == p:
            g.left = x
        else:
            g.right = x

    update(p)
    update(x)

def splay(x, goal=None):
    while x.parent != goal:
        p = x.parent
        g = p.parent if p else None
        if g == goal:
            rotate(x)
        elif (g.left == p) == (p.left == x):
            rotate(p)
            rotate(x)
        else:
            rotate(x)
            rotate(x)
    return x

def split(root, k):
    """k번째 이후로 분리 (1-indexed)"""
    if not root:
        return None, None
    node = find_kth(root, k)
    splay(node)
    right = node.right
    node.right = None
    set_parent(right, None)
    update(node)
    return node, right

def merge(left, right):
    if not left:
        return right
    if not right:
        return left
    node = left
    while node.right:
        node = node.right
    splay(node)
    node.right = right
    set_parent(right, node)
    update(node)
    return node

def find_kth(root, k):
    """k번째 노드 찾기 (1-indexed)"""
    node = root
    while True:
        left_size = node.left.size if node.left else 0
        if k <= left_size:
            node = node.left
        elif k == left_size + 1:
            return node
        else:
            k -= left_size + 1
            node = node.right

def range_sum(root, l, r):
    """[l, r] 구간 합 (1-indexed)"""
    left, mid_right = split(root, l - 1)
    mid, right = split(mid_right, r - l + 1)
    result = mid.sum if mid else 0
    mid_right = merge(mid, right)
    root = merge(left, mid_right)
    return result, root

# 테스트
nodes = [SplayNode(i) for i in [1, 2, 3, 4, 5]]
for i in range(1, len(nodes)):
    nodes[i].parent = nodes[i - 1]
    nodes[i - 1].right = nodes[i]

root = nodes[0]
for node in reversed(nodes):
    update(node)

total, root = range_sum(root, 2, 4)
print(f"구간 합 [2, 4]: {total}")  # 2+3+4 = 9

total, root = range_sum(root, 1, 5)
print(f"구간 합 [1, 5]: {total}")  # 1+2+3+4+5 = 15
```

---

## 상각 분석 (Amortized Analysis)

스플레이 트리의 분석에는 **포텐셜 함수(Potential Function)** 를 사용합니다.

각 노드 `x`의 `rank`를 `log(size(x))`로 정의하고, 트리 전체의 포텐셜을 모든 노드 rank의 합으로 설정합니다.

**Access Lemma**: 노드 `x`를 루트 `t`로 스플레이하는 상각 비용은 최대 `3(rank(t) - rank(x)) + 1`입니다.

이를 통해 m회 연산의 총 비용이 `O((m + n) log n)`임을 증명할 수 있습니다.

---

## 주의사항 및 팁

### 1. 상수 인수 주의
스플레이 트리는 상각 O(log n)이지만 상수 인수가 AVL, 레드-블랙 트리보다 큽니다. 균등 분포 접근에서는 성능이 떨어질 수 있습니다.

### 2. 최악 케이스 대비
단일 연산의 최악 케이스는 O(n)입니다. 실시간 응답이 중요한 시스템(경성 실시간)에는 적합하지 않습니다.

### 3. 멀티스레드 주의
스플레이는 읽기(search) 연산도 트리를 수정합니다. 멀티스레드 환경에서는 단순 읽기 락(read lock)으로 충분하지 않고 쓰기 락이 필요합니다.

### 4. 실전 활용 패턴
- **구간 쿼리**: 분리/합치기 연산으로 구간 덮어쓰기, 구간 뒤집기 등을 지원합니다.
- **LRU 캐시**: 최근 접근 원소가 루트에 모이는 성질을 캐시 구현에 활용합니다.
- **동적 순서 통계**: k번째 원소를 O(log n) 상각으로 찾습니다.

---

## 정리

스플레이 트리는 접근 패턴의 지역성을 자료구조 수준에서 활용하는 아이디어입니다. 추가 메모리 없이 상각 O(log n)을 달성하며, 구간 연산 등에서 높은 표현력을 보입니다. 균등 분포보다 편향된 접근 패턴이 있는 애플리케이션에서 실질적 성능 이득을 얻을 수 있습니다.

## 참고 자료
- [Wikipedia - Splay Tree](https://en.wikipedia.org/wiki/Splay_tree)
- [Wikipedia - Amortized Analysis](https://en.wikipedia.org/wiki/Amortized_analysis)
- [CP-Algorithms - Splay Tree](https://cp-algorithms.com/data_structures/splay-tree.html)
