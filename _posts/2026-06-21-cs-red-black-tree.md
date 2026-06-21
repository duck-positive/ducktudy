---
layout: post
title: "레드-블랙 트리 완전 정복: 자가 균형 이진 탐색 트리의 핵심 원리"
date: 2026-06-21
categories: [cs, computer-science]
tags: [red-black-tree, data-structure, algorithm, binary-search-tree, self-balancing, java, c]
---

레드-블랙 트리(Red-Black Tree)는 C++ STL의 `std::map`, Java의 `TreeMap`, Linux 커널의 프로세스 스케줄러까지 현대 소프트웨어 곳곳에 사용되는 자가 균형 이진 탐색 트리(Self-Balancing BST)다. 이름에서 알 수 있듯이 각 노드에 빨간색 또는 검정색 색상을 부여하고, 이 색상 규칙을 지키도록 트리를 유지함으로써 **O(log n)** 의 탐색·삽입·삭제 성능을 보장한다.

이번 아티클에서는 레드-블랙 트리의 다섯 가지 불변 조건부터 회전(Rotation)과 색 반전(Color Flip) 연산, 실제 Java/C 구현 예제까지 완전히 파헤친다.

---

## 왜 레드-블랙 트리가 필요한가?

일반 이진 탐색 트리(BST)는 최악의 경우 삽입 순서가 정렬된 데이터라면 편향 트리(Skewed Tree)가 되어 탐색이 **O(n)** 으로 저하된다. AVL 트리는 엄격한 균형(높이 차 ≤1)으로 이를 해결하지만, 삽입·삭제 시 회전이 빈번해 쓰기 비용이 높다.

레드-블랙 트리는 **"완벽한 균형보다는 충분한 균형"** 전략을 택한다. 가장 긴 경로가 가장 짧은 경로의 두 배를 넘지 않도록 색상 규칙으로 느슨하게 균형을 유지하여, AVL보다 삽입·삭제 시 회전 횟수를 줄이면서도 탐색은 여전히 O(log n)을 보장한다.

| 비교 항목 | BST (최악) | AVL 트리 | 레드-블랙 트리 |
|---|---|---|---|
| 탐색 | O(n) | O(log n) | O(log n) |
| 삽입 | O(n) | O(log n), 많은 회전 | O(log n), 최대 2회 회전 |
| 삭제 | O(n) | O(log n), 많은 회전 | O(log n), 최대 3회 회전 |
| 균형 엄격도 | 없음 | 엄격 | 느슨 |

---

## 다섯 가지 불변 조건 (Red-Black Properties)

레드-블랙 트리는 아래 다섯 가지 규칙을 **항상** 만족해야 한다. 이 규칙이 깨지면 즉시 회전 또는 색 반전으로 복구한다.

1. **색상 규칙**: 모든 노드는 빨간색(Red) 또는 검정색(Black)이다.
2. **루트 규칙**: 루트 노드는 반드시 검정색이다.
3. **리프 규칙**: 모든 리프(NIL 노드)는 검정색이다.
4. **빨간색 규칙**: 빨간색 노드의 자식은 반드시 검정색이다 (빨간색 노드가 연속될 수 없다).
5. **블랙-높이 규칙**: 임의의 노드에서 하위 리프까지의 모든 경로에서 검정색 노드 수(블랙-높이)가 동일하다.

규칙 4와 5가 핵심이다. 규칙 4는 빨간색 노드를 연속 배치하지 못하게 하고, 규칙 5는 모든 경로의 검정색 노드 수가 같아야 하므로 경로 길이 차이가 최대 2배 이내로 제한된다.

```
예시 트리:
        13(B)
       /      \
     8(R)    17(R)
    /   \    /   \
  1(B) 11(B)15(B)25(B)
        \         \
        11.5(R)   27(R)
```

---

## 핵심 연산: 회전과 색 반전

### 좌회전 (Left Rotation)

```
    x                y
   / \              / \
  A   y    →      x   C
     / \         / \
    B   C       A   B
```

`x`를 `y`의 왼쪽 자식으로 내리고, `y`가 `x`의 자리를 차지한다. 우회전은 반대 방향이다.

### 삽입 후 균형 복구 경우의 수

새 노드는 항상 **빨간색**으로 삽입한다 (블랙-높이를 바꾸지 않기 위해). 이후 규칙 위반 여부에 따라 세 가지 케이스를 처리한다.

- **Case 1**: 삼촌(Uncle) 노드가 빨간색 → 부모와 삼촌을 검정색으로, 조부모를 빨간색으로 바꾼다 (색 반전).
- **Case 2**: 삼촌이 검정색, 새 노드가 부모의 오른쪽 자식 → 부모 기준 좌회전 후 Case 3으로.
- **Case 3**: 삼촌이 검정색, 새 노드가 부모의 왼쪽 자식 → 조부모 기준 우회전 후 색 교환.

---

## 구현 예제 1: Java로 레드-블랙 트리 구현

```java
public class RedBlackTree {
    private static final boolean RED   = true;
    private static final boolean BLACK = false;

    private class Node {
        int key;
        Node left, right, parent;
        boolean color;

        Node(int key) {
            this.key   = key;
            this.color = RED;
        }
    }

    private Node root;
    private final Node NIL; // 센티널 노드

    public RedBlackTree() {
        NIL = new Node(0);
        NIL.color = BLACK;
        root = NIL;
    }

    private void leftRotate(Node x) {
        Node y = x.right;
        x.right = y.left;
        if (y.left != NIL) y.left.parent = x;
        y.parent = x.parent;
        if (x.parent == null)      root = y;
        else if (x == x.parent.left) x.parent.left  = y;
        else                          x.parent.right = y;
        y.left   = x;
        x.parent = y;
    }

    private void rightRotate(Node y) {
        Node x = y.left;
        y.left = x.right;
        if (x.right != NIL) x.right.parent = y;
        x.parent = y.parent;
        if (y.parent == null)      root = x;
        else if (y == y.parent.left) y.parent.left  = x;
        else                          y.parent.right = x;
        x.right  = y;
        y.parent = x;
    }

    public void insert(int key) {
        Node z = new Node(key);
        z.left = z.right = z.parent = NIL;

        Node y = null, x = root;
        while (x != NIL) {
            y = x;
            x = (z.key < x.key) ? x.left : x.right;
        }
        z.parent = y;
        if (y == null)          root = z;
        else if (z.key < y.key) y.left  = z;
        else                    y.right = z;

        insertFixup(z);
    }

    private void insertFixup(Node z) {
        while (z.parent != NIL && z.parent.color == RED) {
            if (z.parent == z.parent.parent.left) {
                Node uncle = z.parent.parent.right;
                if (uncle.color == RED) {           // Case 1
                    z.parent.color         = BLACK;
                    uncle.color            = BLACK;
                    z.parent.parent.color  = RED;
                    z = z.parent.parent;
                } else {
                    if (z == z.parent.right) {      // Case 2
                        z = z.parent;
                        leftRotate(z);
                    }
                    z.parent.color        = BLACK;  // Case 3
                    z.parent.parent.color = RED;
                    rightRotate(z.parent.parent);
                }
            } else {
                // 대칭 케이스 (오른쪽 방향)
                Node uncle = z.parent.parent.left;
                if (uncle.color == RED) {
                    z.parent.color        = BLACK;
                    uncle.color           = BLACK;
                    z.parent.parent.color = RED;
                    z = z.parent.parent;
                } else {
                    if (z == z.parent.left) {
                        z = z.parent;
                        rightRotate(z);
                    }
                    z.parent.color        = BLACK;
                    z.parent.parent.color = RED;
                    leftRotate(z.parent.parent);
                }
            }
        }
        root.color = BLACK; // 루트는 항상 검정색
    }

    public boolean search(int key) {
        Node x = root;
        while (x != NIL) {
            if      (key == x.key) return true;
            else if (key <  x.key) x = x.left;
            else                   x = x.right;
        }
        return false;
    }
}
```

삽입 시 `insertFixup`이 빨간색 규칙 위반을 올라가며 수정하고, 루트에서 멈추면 루트를 강제로 검정색으로 바꾸어 종료한다.

---

## 구현 예제 2: C로 구현한 간단한 레드-블랙 트리 탐색/높이 검증

```c
#include <stdio.h>
#include <stdlib.h>
#include <assert.h>

#define RED   1
#define BLACK 0

typedef struct Node {
    int key, color;
    struct Node *left, *right, *parent;
} Node;

static Node NIL_NODE = {0, BLACK, NULL, NULL, NULL};
static Node *NIL = &NIL_NODE;

Node *newNode(int key) {
    Node *n   = malloc(sizeof(Node));
    n->key    = key;
    n->color  = RED;
    n->left   = n->right = n->parent = NIL;
    return n;
}

/* 블랙-높이 검증: 모든 경로의 검정 노드 수가 동일한지 확인 */
int blackHeight(Node *n) {
    if (n == NIL) return 1;
    int lh = blackHeight(n->left);
    int rh = blackHeight(n->right);
    assert(lh == rh); // 블랙-높이 규칙 위반 시 assert 실패
    return lh + (n->color == BLACK ? 1 : 0);
}

/* 빨간색 규칙 검증: 빨간 노드의 자식은 검정 */
void validateRedRule(Node *n) {
    if (n == NIL) return;
    if (n->color == RED) {
        assert(n->left->color  == BLACK);
        assert(n->right->color == BLACK);
    }
    validateRedRule(n->left);
    validateRedRule(n->right);
}

void validate(Node *root) {
    assert(root->color == BLACK); // 루트 규칙
    blackHeight(root);
    validateRedRule(root);
    printf("✓ 모든 레드-블랙 트리 불변 조건 통과\n");
}

int main() {
    /* 수동으로 작은 RB 트리 구성 후 검증 */
    Node *root = newNode(13); root->color = BLACK;
    Node *n8   = newNode(8);
    Node *n17  = newNode(17);
    Node *n1   = newNode(1);  n1->color = BLACK;
    Node *n11  = newNode(11); n11->color = BLACK;
    Node *n15  = newNode(15); n15->color = BLACK;
    Node *n25  = newNode(25); n25->color = BLACK;

    root->left = n8;  root->right = n17;
    n8->left   = n1;  n8->right   = n11;
    n17->left  = n15; n17->right  = n25;

    validate(root);
    return 0;
}
```

`blackHeight`와 `validateRedRule` 함수로 트리가 레드-블랙 불변 조건을 충족하는지 검증할 수 있다.

---

## 주의사항 및 팁

### 1. NIL 센티널 노드 사용
`NULL` 대신 고정 `NIL` 센티널 노드를 사용하면 엣지 케이스(리프 처리)를 단순화할 수 있다. 센티널은 색상이 항상 BLACK이며 실제 데이터를 담지 않는다.

### 2. 삭제는 삽입보다 훨씬 복잡
삭제 시에는 **더블 블랙(Double Black)** 문제가 발생할 수 있다. 삭제되는 노드가 검정이면 블랙-높이가 깨지므로, 형제 노드의 색상과 자식 구성에 따라 6가지 케이스를 처리해야 한다. 실무에서는 검증된 라이브러리(Java `TreeMap`, C++ `std::map`)를 활용하는 편이 안전하다.

### 3. AVL vs 레드-블랙 선택 기준
- **탐색이 훨씬 빈번**: AVL 트리가 유리 (더 엄격한 균형 → 탐색 트리 깊이 더 작음)
- **삽입·삭제가 빈번**: 레드-블랙 트리가 유리 (회전 횟수 적음)
- **일반적인 맵/셋 자료구조**: 레드-블랙 트리가 기본 선택 (대부분의 언어 표준 라이브러리)

### 4. 메모리 레이아웃 최적화
각 노드에 포인터 3개(left, right, parent)와 색상 1비트가 필요하다. 실제 구현에서는 color를 포인터의 하위 1비트에 저장하는 태그드 포인터(tagged pointer) 기법으로 메모리를 절약하기도 한다.

### 5. 실무 활용처 파악
- **Linux Kernel**: `struct rb_root` / `struct rb_node` — 프로세스 스케줄링, 가상 메모리 영역 관리
- **Java**: `java.util.TreeMap`, `java.util.TreeSet`
- **C++ STL**: `std::map`, `std::set`, `std::multimap`
- **Nginx**: 타이머 관리

---

## 참고 자료
- [Red–black tree - Wikipedia](https://en.wikipedia.org/wiki/Red%E2%80%93black_tree)
- [Introduction to Red-Black Tree - GeeksforGeeks](https://www.geeksforgeeks.org/dsa/introduction-to-red-black-tree/)
- [Red-Black Tree - Programiz](https://www.programiz.com/dsa/red-black-tree)
- [Red-Black Trees - EECS Michigan](https://www.eecs.umich.edu/courses/eecs380/ALG/red_black.html)
