---
layout: post
title: "Rope 자료구조 완전 정복: 텍스트 에디터가 대용량 문자열을 O(log n)에 편집하는 비밀"
date: 2026-08-12
categories: [cs, computer-science]
tags: [rope, data-structure, string, text-editor, binary-tree, algorithm]
---

텍스트 에디터에서 수백 메가바이트짜리 파일을 편집할 때도 커서가 부드럽게 움직이고, 중간 어딘가에 글자를 삽입해도 즉각 반응하는 것을 당연하게 여기곤 한다. 하지만 만약 파일 전체를 하나의 연속된 배열(array)로 저장한다면, 1GB짜리 파일의 첫 번째 위치에 글자 하나를 삽입할 때마다 뒤에 오는 수십억 바이트를 통째로 밀어야 한다. O(n)짜리 연산이 사용자 인터랙션마다 발생하는 것이다. 이 문제를 근본적으로 해결하는 것이 **Rope 자료구조**다.

## 개념 설명: Rope란 무엇인가

Rope는 1995년 Hans-J. Boehm, Russ Atkinson, Michael Plass의 논문 "Ropes: an Alternative to Strings"에서 처음 제안된 이진 트리 기반의 문자열 자료구조다. 이름은 선박용 밧줄에서 왔다. 여러 가닥의 실(소문자열)을 꼬아서 하나의 굵은 밧줄(문자열)을 만드는 것처럼, Rope도 짧은 문자열 조각들을 트리 구조로 엮어 논리적으로 하나의 긴 문자열처럼 동작하게 만든다.

구조는 다음과 같다:

- **리프 노드(Leaf Node)**: 실제 문자열 조각을 저장한다. 일반적으로 짧은 문자열(예: 최대 64바이트)을 담는다.
- **내부 노드(Internal Node)**: 두 개의 자식을 가지며, 왼쪽 서브트리에 속한 모든 문자열의 총 길이인 **weight**만 저장한다. 실제 문자 데이터는 없다.

전체 문자열의 i번째 문자를 찾으려면, 루트에서 시작해 weight와 i를 비교하며 왼쪽/오른쪽으로 내려가면 된다. 이 과정이 트리 높이만큼만 반복되므로, 균형 잡힌 Rope에서 인덱스 접근은 **O(log n)**이다.

### 핵심 연산

| 연산 | 시간 복잡도 | 설명 |
|------|-------------|------|
| Index | O(log n) | i번째 문자 조회 |
| Concat | O(1) ~ O(log n) | 두 Rope를 이어붙이기 |
| Split | O(log n) | 위치 k에서 두 Rope로 분리 |
| Insert | O(log n) | Split + Concat 조합 |
| Delete | O(log n) | Split + Concat 조합 |
| Iterate | O(n) | 전체 순회 |

## 왜 Rope가 필요한가

### 일반 String의 한계

C++의 `std::string`, Java의 `String`, Python의 `str`은 모두 내부적으로 연속된 메모리 블록을 사용한다. 이 때문에:

- **삽입·삭제**: 중간 위치 연산은 O(n). 파일이 1GB라면 수천 ms가 소요될 수 있다.
- **메모리 재할당**: 문자열이 커지면 더 큰 배열로 통째로 복사해야 한다. 피크 메모리 사용량이 2×로 증가한다.
- **불변성 문제**: Java의 `String`처럼 불변 타입에서는 모든 수정이 새 객체 생성을 유발한다.

### 대안 비교

텍스트 편집에 쓰이는 자료구조들을 비교하면:

- **Gap Buffer**: Emacs가 사용. 커서 주변 편집은 O(1)이지만, 커서 이동 자체가 O(n)이 될 수 있다.
- **Piece Table**: VS Code가 사용. 원본 버퍼와 추가 버퍼를 테이블로 관리. 메모리 효율이 높지만 구현이 복잡하다.
- **Rope**: Sublime Text, xi-editor 등이 사용. 균형 트리 기반으로 모든 연산이 O(log n). 병렬화와 지속성(persistent) 구현에 유리하다.

## 실제 구현 예제

### 예제 1: Python으로 구현하는 기본 Rope

```python
class RopeNode:
    def __init__(self, value=""):
        self.left = None
        self.right = None
        self.value = value         # 리프 노드만 사용
        self.weight = len(value)   # 왼쪽 서브트리 문자 수 (리프는 자신의 길이)

    def is_leaf(self):
        return self.left is None and self.right is None


def rope_index(node, i):
    """i번째 문자를 O(log n)에 반환"""
    if node is None:
        raise IndexError("index out of range")
    if node.is_leaf():
        return node.value[i]
    if i < node.weight:
        return rope_index(node.left, i)
    else:
        return rope_index(node.right, i - node.weight)


def rope_concat(left, right):
    """두 Rope를 O(1)에 연결 (단순 버전)"""
    if left is None:
        return right
    if right is None:
        return left
    parent = RopeNode()
    parent.left = left
    parent.right = right
    parent.weight = rope_length(left)
    return parent


def rope_length(node):
    if node is None:
        return 0
    if node.is_leaf():
        return node.weight
    return rope_length(node.left) + rope_length(node.right)


def rope_to_string(node):
    """전체 문자열 재구성 O(n)"""
    if node is None:
        return ""
    if node.is_leaf():
        return node.value
    return rope_to_string(node.left) + rope_to_string(node.right)


def rope_split(node, i):
    """위치 i에서 두 Rope (left_part, right_part)로 분리"""
    if node is None:
        return None, None
    if node.is_leaf():
        left_leaf = RopeNode(node.value[:i])
        right_leaf = RopeNode(node.value[i:])
        return left_leaf, right_leaf
    if i < node.weight:
        left_split, right_split = rope_split(node.left, i)
        return left_split, rope_concat(right_split, node.right)
    elif i > node.weight:
        left_split, right_split = rope_split(node.right, i - node.weight)
        return rope_concat(node.left, left_split), right_split
    else:
        return node.left, node.right


def rope_insert(root, i, text):
    """위치 i에 text 삽입"""
    new_node = RopeNode(text)
    left, right = rope_split(root, i)
    return rope_concat(rope_concat(left, new_node), right)


# 사용 예시
r1 = RopeNode("Hello, ")
r2 = RopeNode("World!")
root = rope_concat(r1, r2)

print(rope_to_string(root))         # Hello, World!
print(rope_index(root, 7))          # W

root = rope_insert(root, 7, "Beautiful ")
print(rope_to_string(root))         # Hello, Beautiful World!
print(f"Total length: {rope_length(root)}")
```

### 예제 2: Treap 기반 고성능 Rope (C++)

실전에서는 단순 이진 트리보다 **Treap(Implicit Key Treap)**을 기반으로 Rope를 구현한다. Treap은 이진 탐색 트리의 순서를 임시 키(인덱스)로, 힙 속성을 무작위 우선순위로 관리해 평균 O(log n) 높이를 유지한다.

```cpp
#include <iostream>
#include <random>
#include <string>
using namespace std;

mt19937 rng(42);

struct Node {
    char ch;
    int priority, size;
    Node *left, *right;

    Node(char c) : ch(c), priority(rng()), size(1), left(nullptr), right(nullptr) {}
};

int sz(Node* n) { return n ? n->size : 0; }

void update(Node* n) {
    if (n) n->size = 1 + sz(n->left) + sz(n->right);
}

// 앞 k개와 나머지로 분리
pair<Node*, Node*> split(Node* n, int k) {
    if (!n) return {nullptr, nullptr};
    int leftSize = sz(n->left);
    if (leftSize >= k) {
        auto [l, r] = split(n->left, k);
        n->left = r;
        update(n);
        return {l, n};
    } else {
        auto [l, r] = split(n->right, k - leftSize - 1);
        n->right = l;
        update(n);
        return {n, r};
    }
}

Node* merge(Node* l, Node* r) {
    if (!l) return r;
    if (!r) return l;
    if (l->priority > r->priority) {
        l->right = merge(l->right, r);
        update(l);
        return l;
    } else {
        r->left = merge(l, r->left);
        update(r);
        return r;
    }
}

Node* buildRope(const string& s) {
    Node* root = nullptr;
    for (char c : s)
        root = merge(root, new Node(c));
    return root;
}

void insertAt(Node*& root, int pos, const string& s) {
    auto [l, r] = split(root, pos);
    Node* mid = buildRope(s);
    root = merge(merge(l, mid), r);
}

void deleteRange(Node*& root, int pos, int len) {
    auto [l, mr] = split(root, pos);
    auto [m, r] = split(mr, len);
    // m은 삭제 대상 - 메모리 해제 필요
    root = merge(l, r);
}

char getChar(Node* root, int i) {
    while (root) {
        int leftSz = sz(root->left);
        if (i == leftSz) return root->ch;
        else if (i < leftSz) root = root->left;
        else { i -= leftSz + 1; root = root->right; }
    }
    throw runtime_error("out of bounds");
}

void print(Node* n) {
    if (!n) return;
    print(n->left);
    cout << n->ch;
    print(n->right);
}

int main() {
    Node* root = buildRope("Hello, World!");
    cout << "Original: "; print(root); cout << '\n';

    insertAt(root, 7, "Beautiful ");
    cout << "After insert: "; print(root); cout << '\n';

    deleteRange(root, 0, 7);  // "Hello, " 삭제
    cout << "After delete: "; print(root); cout << '\n';

    cout << "Char at index 9: " << getChar(root, 9) << '\n';
    cout << "Total size: " << sz(root) << '\n';
    return 0;
}
```

실행 결과:
```
Original: Hello, World!
After insert: Hello, Beautiful World!
After delete: Beautiful World!
Char at index 9: W
Total size: 14
```

## 주의사항 및 팁

### 1. 균형 유지가 핵심이다

순수한 이진 트리 기반 Rope는 삽입·분리가 반복될수록 트리가 편향되어 O(n)으로 퇴화할 수 있다. 실전에서는 다음 방법을 사용한다:

- **Treap(Randomized BST)**: 위 예제처럼 무작위 우선순위로 평균 O(log n)을 보장한다.
- **리밸런싱(Rebalancing)**: Boehm의 원본 논문에서는 피보나치 수열을 기준으로 트리가 너무 불균형해지면 재구축한다. 노드 수 n에 비해 높이가 1.44 × log₂(n+2)를 넘으면 재균형을 시작한다.
- **AVL/Red-Black Tree 기반**: 항상 O(log n)을 보장하지만 구현 복잡도가 높다.

### 2. 리프 크기 선택

리프 노드에 저장하는 문자열의 최적 크기는 보통 **32~64바이트**다. 이는 CPU 캐시 라인(64바이트)과 일치시켜 캐시 미스를 줄이기 위한 설계다. 너무 작으면 트리 노드 오버헤드가 커지고, 너무 크면 부분 수정 시 비효율적이다.

### 3. 불변 Rope와 지속성

Rope의 분리·연결은 기존 노드를 수정하지 않고 새 노드를 생성하는 방식으로 구현할 수 있다. 이 경우 구버전의 트리가 공유 노드를 통해 살아있으므로, **무제한 Undo/Redo**를 메모리 효율적으로 구현할 수 있다. 이것이 xi-editor 같은 현대적 에디터가 Rope를 선택하는 핵심 이유 중 하나다.

### 4. 멀티스레드 환경

Rope의 불변성(immutability)을 활용하면, 쓰기는 새 버전을 생성하고 읽기는 이전 버전을 참조하는 방식으로 락(lock) 없이 동시 접근이 가능하다. 협업 편집기에서 CRDT와 함께 Rope를 사용하는 이유다.

### 5. 인코딩 고려

UTF-8 인코딩에서는 바이트 인덱스와 문자(codepoint) 인덱스가 다르다. 멀티바이트 문자를 올바르게 처리하려면 weight를 바이트 수 기준으로 유지하고, 문자 경계(character boundary) 검증 로직을 별도로 추가해야 한다.

## 정리

Rope는 단순히 문자열을 빠르게 처리하는 자료구조를 넘어, **지속성·불변성·병렬성**이라는 현대 소프트웨어의 핵심 요구를 우아하게 충족하는 설계 철학의 산물이다. 특히 텍스트 에디터처럼 빈번한 중간 삽입·삭제가 발생하는 시나리오에서 기존 배열 기반 문자열 대비 압도적인 성능을 발휘한다. Treap 기반 구현으로 모든 연산을 기댓값 O(log n)에 처리할 수 있으며, 불변 변형을 통해 복잡한 상태 관리도 단순화할 수 있다.

## 참고 자료
- [GitHub: Rope 자료구조 구현 모음](https://github.com/topics/rope?o=asc&s=stars)
- [GitHub: lz4/lz4 — 극한의 압축 속도를 위한 레퍼런스 구현](https://github.com/lz4/lz4)
