---
layout: post
title: "순서 통계 트리(Order Statistic Tree) 완전 정복: k번째 원소 탐색과 순위 계산을 O(log N)에 처리하기"
date: 2026-08-19
categories: [cs, computer-science]
tags: [order-statistic-tree, augmented-bst, data-structure, algorithm, red-black-tree, cpp]
---

## 순서 통계 트리(Order Statistic Tree)란?

일반적인 이진 탐색 트리(BST)는 원소의 삽입, 삭제, 탐색을 O(log N)에 수행한다. 그러나 "정렬된 순서에서 k번째 원소는 무엇인가?" 또는 "원소 x는 몇 번째로 작은가?"라는 질문에는 답할 수 없다. 이를 위해서는 모든 원소를 순회해야 하므로 O(N)이 걸린다.

**순서 통계 트리(Order Statistic Tree, OST)**는 일반 BST를 **증강(augmentation)**하여 두 가지 추가 연산을 O(log N)에 지원하는 자료구조다:

- **SELECT(k)**: 정렬 순서에서 k번째로 작은 원소를 반환한다.
- **RANK(x)**: 원소 x가 정렬 순서에서 몇 번째인지(순위)를 반환한다.

이 자료구조는 CLRS(Introduction to Algorithms) 교재의 "증강 자료구조" 챕터에서 대표적인 사례로 소개되며, 알고리즘 교육의 핵심 주제 중 하나다.

---

## 왜 필요한가?

순서 통계 트리가 필요한 상황은 생각보다 많다.

### 실시간 순위 시스템

게임 리더보드에서 특정 점수를 가진 플레이어의 순위를 실시간으로 조회해야 한다고 가정하자. 배열 정렬은 삽입마다 O(N)이 걸리고, 일반 BST는 순위 계산에 O(N)이 필요하다. OST는 삽입 O(log N), 순위 조회 O(log N)으로 모두 처리한다.

### k번째 최솟값 반복 조회

데이터 스트림에서 중앙값(median)을 실시간으로 추적하거나, "상위 10%에 해당하는 값은?"처럼 특정 백분위수를 구해야 하는 경우에도 OST가 유용하다.

### 역전 쌍(Inversion Count) 계산

배열에서 i < j이지만 A[i] > A[j]인 쌍의 수를 세는 문제(Merge Sort로도 풀 수 있지만)를 OST로 O(N log N)에 해결할 수 있다.

---

## 핵심 아이디어: 서브트리 크기 증강

OST의 구현 원리는 단순하다. 각 노드에 **해당 노드를 루트로 하는 서브트리의 크기(size)**를 추가로 저장한다.

```
노드 구조:
┌─────────────────────────────────────┐
│  key   │  left  │  right  │  size  │
└─────────────────────────────────────┘
        size = size(left) + size(right) + 1
```

이 불변식(invariant)이 항상 유지되도록 삽입/삭제/회전 시 size 값을 갱신한다.

### SELECT 알고리즘

k번째 원소를 찾는 알고리즘:

```python
def select(node, k):
    """k번째로 작은 원소를 반환 (1-indexed)"""
    if node is None:
        return None
    
    # 왼쪽 서브트리의 크기
    left_size = node.left.size if node.left else 0
    
    if k == left_size + 1:
        # 현재 노드가 k번째
        return node.key
    elif k <= left_size:
        # k번째는 왼쪽 서브트리에 있음
        return select(node.left, k)
    else:
        # k번째는 오른쪽 서브트리에 있음 (오른쪽에서의 순위 조정 필요)
        return select(node.right, k - left_size - 1)
```

### RANK 알고리즘

원소 x의 순위를 계산:

```python
def rank(root, x):
    """원소 x의 순위를 반환 (x보다 작거나 같은 원소의 수)"""
    node = root
    r = 0
    
    while node is not None:
        if x < node.key:
            # x는 왼쪽 서브트리에 있음
            node = node.left
        elif x > node.key:
            # 현재 노드와 왼쪽 서브트리는 모두 x보다 작음
            left_size = node.left.size if node.left else 0
            r += left_size + 1
            node = node.right
        else:
            # 현재 노드가 x
            left_size = node.left.size if node.left else 0
            r += left_size + 1
            return r
    
    return -1  # x가 트리에 없음
```

---

## C++ 실전 구현: Policy-Based Tree 활용

경쟁 프로그래밍에서는 직접 구현하지 않고 **C++ STL의 Policy-Based Data Structures**를 활용하는 것이 일반적이다. GCC의 `ext/pb_ds` 라이브러리를 사용하면 된다.

```cpp
#include <bits/stdc++.h>
#include <ext/pb_ds/assoc_container.hpp>
#include <ext/pb_ds/tree_policy.hpp>
using namespace std;
using namespace __gnu_pbds;

// Order Statistic Tree 타입 정의
// tree<Key, Mapped, Cmp, Tag, Node_Update>
typedef tree<
    int,                        // Key 타입
    null_type,                  // Mapped (set이므로 null_type)
    less<int>,                  // 비교 함수
    rb_tree_tag,                // Red-Black Tree 사용
    tree_order_statistics_node_update  // OST 기능 활성화
> ordered_set;

int main() {
    ordered_set s;
    
    // 원소 삽입
    s.insert(10);
    s.insert(30);
    s.insert(20);
    s.insert(50);
    s.insert(40);
    
    // k번째 원소 조회 (0-indexed)
    cout << "0번째(최소): " << *s.find_by_order(0) << "\n"; // 10
    cout << "2번째: "       << *s.find_by_order(2) << "\n"; // 30
    cout << "4번째(최대): " << *s.find_by_order(4) << "\n"; // 50
    
    // 원소의 순위 조회 (몇 개가 더 작은지, 0-indexed)
    cout << "10의 순위: " << s.order_of_key(10) << "\n"; // 0
    cout << "35의 순위: " << s.order_of_key(35) << "\n"; // 3 (10,20,30이 더 작음)
    cout << "50의 순위: " << s.order_of_key(50) << "\n"; // 4
    
    // 삭제 후 재조회
    s.erase(20);
    cout << "\n20 삭제 후:\n";
    cout << "2번째: " << *s.find_by_order(2) << "\n"; // 40
    
    return 0;
}
```

출력 결과:
```
0번째(최소): 10
2번째: 30
4번째(최대): 50
10의 순위: 0
35의 순위: 3
50의 순위: 4

20 삭제 후:
2번째: 40
```

---

## 완전한 직접 구현: Red-Black Tree 기반 OST

실제로 어떻게 동작하는지 이해하기 위해 간단한 직접 구현을 살펴보자. 아래는 레드-블랙 트리 없이 AVL 트리 기반으로 size 증강을 적용한 예시다.

```cpp
#include <bits/stdc++.h>
using namespace std;

struct Node {
    int key, size, height;
    Node *left, *right;
    
    Node(int k) : key(k), size(1), height(1), left(nullptr), right(nullptr) {}
};

int getSize(Node* n) { return n ? n->size : 0; }
int getHeight(Node* n) { return n ? n->height : 0; }

void update(Node* n) {
    if (n) {
        n->size = 1 + getSize(n->left) + getSize(n->right);
        n->height = 1 + max(getHeight(n->left), getHeight(n->right));
    }
}

Node* rotateRight(Node* y) {
    Node* x = y->left;
    Node* T2 = x->right;
    x->right = y;
    y->left = T2;
    update(y);
    update(x);
    return x;
}

Node* rotateLeft(Node* x) {
    Node* y = x->right;
    Node* T2 = y->left;
    y->left = x;
    x->right = T2;
    update(x);
    update(y);
    return y;
}

int getBalance(Node* n) {
    return n ? getHeight(n->left) - getHeight(n->right) : 0;
}

Node* insert(Node* node, int key) {
    if (!node) return new Node(key);
    
    if (key < node->key)
        node->left = insert(node->left, key);
    else if (key > node->key)
        node->right = insert(node->right, key);
    else
        return node; // 중복 허용 안 함
    
    update(node);
    
    int balance = getBalance(node);
    
    // LL 회전
    if (balance > 1 && key < node->left->key)
        return rotateRight(node);
    // RR 회전
    if (balance < -1 && key > node->right->key)
        return rotateLeft(node);
    // LR 회전
    if (balance > 1 && key > node->left->key) {
        node->left = rotateLeft(node->left);
        return rotateRight(node);
    }
    // RL 회전
    if (balance < -1 && key < node->right->key) {
        node->right = rotateRight(node->right);
        return rotateLeft(node);
    }
    
    return node;
}

// k번째 원소 반환 (1-indexed)
int select(Node* node, int k) {
    if (!node) return -1;
    int leftSize = getSize(node->left);
    
    if (k == leftSize + 1) return node->key;
    else if (k <= leftSize) return select(node->left, k);
    else return select(node->right, k - leftSize - 1);
}

// key보다 작은 원소의 수 반환 (0-indexed rank)
int rank(Node* node, int key) {
    if (!node) return 0;
    
    if (key <= node->key)
        return rank(node->left, key);
    else
        return 1 + getSize(node->left) + rank(node->right, key);
}

int main() {
    Node* root = nullptr;
    vector<int> vals = {5, 3, 7, 1, 4, 6, 8, 2};
    
    for (int v : vals)
        root = insert(root, v);
    
    // 정렬 순서: 1, 2, 3, 4, 5, 6, 7, 8
    cout << "전체 원소 수: " << getSize(root) << "\n";
    
    for (int k = 1; k <= 8; k++)
        cout << k << "번째 원소: " << select(root, k) << "\n";
    
    cout << "\n5의 순위(0-indexed): " << rank(root, 5) << "\n"; // 4
    cout << "1의 순위(0-indexed): " << rank(root, 1) << "\n"; // 0
    cout << "8의 순위(0-indexed): " << rank(root, 8) << "\n"; // 7
    
    return 0;
}
```

---

## 응용: 역전 쌍(Inversion Count) 계산

배열 `[3, 1, 4, 1, 5, 9, 2, 6]`에서 역전 쌍의 수를 OST로 계산하는 예시:

```cpp
#include <bits/stdc++.h>
#include <ext/pb_ds/assoc_container.hpp>
#include <ext/pb_ds/tree_policy.hpp>
using namespace std;
using namespace __gnu_pbds;

typedef tree<pair<int,int>, null_type, less<pair<int,int>>,
             rb_tree_tag, tree_order_statistics_node_update> ordered_set;

long long countInversions(vector<int>& arr) {
    ordered_set s;
    long long inversions = 0;
    int n = arr.size();
    
    for (int i = 0; i < n; i++) {
        // arr[i]보다 큰 원소 중 이미 삽입된 것의 수
        // = (현재까지 삽입된 수) - (arr[i] 이하인 수의 수)
        int inserted = (int)s.size();
        int lessOrEqual = (int)s.order_of_key({arr[i], i}); // arr[i]보다 작거나 같은 수
        
        // 실제로 arr[i]보다 같은 값이 있으면 order_of_key({arr[i], i})는
        // arr[i]보다 먼저 삽입된 같은 값을 포함하지 않음
        // pair를 사용해 중복 원소를 구별
        inversions += inserted - lessOrEqual;
        s.insert({arr[i], i});
    }
    
    return inversions;
}

int main() {
    vector<int> arr = {3, 1, 4, 1, 5, 9, 2, 6};
    cout << "역전 쌍의 수: " << countInversions(arr) << "\n";
    // (3,1), (3,1), (3,2), (4,1), (4,2), (9,2), (9,6) → 7쌍
    
    return 0;
}
```

---

## 주의사항과 팁

### 1. 중복 원소 처리

Policy-based tree는 기본적으로 집합(set)처럼 동작하므로 중복 원소를 저장할 수 없다. 중복이 필요하다면 `pair<int, int>`나 `pair<int, unique_id>`를 키로 사용해야 한다.

```cpp
// 중복 허용: 값과 삽입 순서를 쌍으로 사용
ordered_set s;
s.insert({value, counter++}); // counter는 고유 ID
```

### 2. 범위 쿼리

"값이 [l, r] 범위에 속하는 원소의 수"도 O(log N)에 계산할 수 있다:

```cpp
int countInRange(ordered_set& s, int l, int r) {
    return s.order_of_key(r + 1) - s.order_of_key(l);
}
```

### 3. 경쟁 프로그래밍에서의 한계

Policy-based tree는 GCC 전용이다. MSVC나 Clang에서는 직접 구현해야 한다. 또한 Fenwick Tree(BIT)를 좌표 압축과 결합하면 유사한 기능을 더 간단하게 구현할 수 있는 경우도 많다.

### 4. 복잡도 요약

| 연산 | 시간 복잡도 | 공간 복잡도 |
|------|------------|------------|
| 삽입 | O(log N) | - |
| 삭제 | O(log N) | - |
| SELECT(k) | O(log N) | O(1) 추가 |
| RANK(x) | O(log N) | O(1) 추가 |
| 전체 공간 | - | O(N) |

OST는 "증강(augmentation)"이라는 개념을 가장 명확하게 보여주는 자료구조다. 기존 자료구조에 부가 정보를 추가하고 연산 시 이를 유지하는 패턴은 Interval Tree, Segment Tree 등 다양한 자료구조에도 동일하게 적용된다.

## 참고 자료
- [Order statistic tree - Wikipedia](https://en.wikipedia.org/wiki/Order_statistic_tree)
- [CMU 15-210: Ordered Sets and Augmented BSTs](https://www.cs.cmu.edu/afs/cs/academic/class/15210-s14/www/lectures/ordered.pdf)
- [PBDS (Policy-Based Data Structures) - Codeforces](https://codeforces.com/blog/entry/11080)
- [CLRS Chapter 14: Augmenting Data Structures](https://mitpress.mit.edu/9780262046305/introduction-to-algorithms/)
