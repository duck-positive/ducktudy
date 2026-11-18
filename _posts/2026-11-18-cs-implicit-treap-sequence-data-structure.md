---
layout: post
title: "Implicit Treap 완전 정복: 위치 기반 연산과 구간 처리의 최강 자료구조"
date: 2026-11-18
categories: [cs, computer-science]
tags: [data-structures, treap, implicit-treap, range-queries, competitive-programming, algorithms]
---

**Implicit Treap(암묵적 트립)**은 순서가 있는 시퀀스(배열)에 대해 임의 위치의 삽입·삭제·구간 연산을 모두 O(log N)에 지원하는 강력한 자료구조입니다. 일반 Treap이 키(key)로 비교하는 반면, Implicit Treap은 **노드의 위치(rank)**를 묵시적으로 사용합니다. 가변 배열에서 `std::vector`는 중간 삽입/삭제에 O(N)이 걸리지만, Implicit Treap은 모든 위치에서 O(log N)을 보장합니다.

## Treap 복습

Treap은 Binary Search Tree(BST)와 Heap의 성질을 동시에 만족하는 랜덤화 자료구조입니다:
- **BST 조건**: 각 노드의 key는 왼쪽 서브트리의 key보다 크고, 오른쪽 서브트리의 key보다 작습니다.
- **Heap 조건**: 각 노드의 priority(우선순위, 무작위 값)는 자식들의 priority보다 큽니다(max-heap).

랜덤 priority 덕분에 Treap은 기대값 O(log N) 높이를 가집니다. 핵심 연산은 **Split**과 **Merge**입니다.

## Implicit Treap이란?

일반 Treap의 key 대신 **서브트리 크기(subtree size)**를 활용해 위치를 계산합니다. 어떤 노드의 암묵적 key는 "왼쪽 서브트리의 크기 + 1"입니다. 이를 계산하여 노드의 순서상 위치(rank)를 알 수 있습니다.

```
배열 [A, B, C, D, E]를 Implicit Treap으로 표현하면:

              C (priority=9, size=5)
            /         \
     A (p=7, sz=2)   D (p=5, sz=2)
       \               \
     B (p=3, sz=1)   E (p=2, sz=1)

각 노드의 암묵적 key:
- A: leftSize(0) + 1 = 1 → 배열의 1번 위치
- B: leftSize(0) + 1 = 2 → 배열의 2번 위치 (A.rightChild)
- C: leftSize(2) + 1 = 3 → 배열의 3번 위치 (루트)
- D: leftSize(2+1) + 1 = 4 → 배열의 4번 위치
- E: leftSize(0) + 1 = 5 → 배열의 5번 위치
```

## 핵심 연산 구현

### 예제 1: 기본 Implicit Treap 구현 (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

mt19937 rng(chrono::steady_clock::now().time_since_epoch().count());

struct Node {
    int val;           // 저장값
    int priority;      // 랜덤 우선순위 (Heap 조건용)
    int size;          // 서브트리 크기 (암묵적 key 계산에 사용)
    bool rev;          // 구간 뒤집기 lazy flag
    long long sum;     // 서브트리 합 (구간 쿼리용)
    Node* left;
    Node* right;

    Node(int v) :
        val(v), priority(rng()), size(1),
        rev(false), sum(v), left(nullptr), right(nullptr) {}
};

// 서브트리 크기 반환 (null-safe)
int sz(Node* t) { return t ? t->size : 0; }
long long sm(Node* t) { return t ? t->sum : 0; }

// 자식 정보로 현재 노드 갱신
void update(Node* t) {
    if (!t) return;
    t->size = 1 + sz(t->left) + sz(t->right);
    t->sum  = t->val + sm(t->left) + sm(t->right);
}

// 뒤집기 lazy 태그 적용
void applyRev(Node* t) {
    if (!t) return;
    swap(t->left, t->right);
    t->rev = !t->rev;
}

// lazy 태그 하위 노드에 전파
void pushDown(Node* t) {
    if (!t || !t->rev) return;
    applyRev(t->left);
    applyRev(t->right);
    t->rev = false;
}

// Split: t를 앞의 k개와 나머지로 분리
// l: 앞 k개, r: 나머지
void split(Node* t, int k, Node*& l, Node*& r) {
    if (!t) { l = r = nullptr; return; }
    pushDown(t);  // lazy 전파 먼저
    int leftSize = sz(t->left);
    if (leftSize >= k) {
        // k번째가 왼쪽 서브트리에 있음
        split(t->left, k, l, t->left);
        r = t;
    } else {
        // k번째가 오른쪽 서브트리에 있음
        split(t->right, k - leftSize - 1, t->right, r);
        l = t;
    }
    update(t);
}

// Merge: l과 r을 합쳐 반환
Node* merge(Node* l, Node* r) {
    if (!l) return r;
    if (!r) return l;
    pushDown(l); pushDown(r);
    if (l->priority > r->priority) {
        // l의 루트가 더 높은 우선순위 → l이 루트
        l->right = merge(l->right, r);
        update(l);
        return l;
    } else {
        r->left = merge(l, r->left);
        update(r);
        return r;
    }
}

// 위치 pos에 값 val 삽입 (0-indexed)
Node* insert(Node* t, int pos, int val) {
    Node* l, *r;
    split(t, pos, l, r);
    return merge(merge(l, new Node(val)), r);
}

// 위치 pos의 원소 삭제
Node* erase(Node* t, int pos) {
    Node *l, *m, *r;
    split(t, pos, l, r);
    split(r, 1, m, r);
    delete m;
    return merge(l, r);
}

// [l, r] 구간의 합 (0-indexed)
long long querySum(Node* t, int l, int r) {
    Node *left, *mid, *right;
    split(t, l, left, right);
    split(right, r - l + 1, mid, right);
    long long result = sm(mid);
    t = merge(merge(left, mid), right);  // 주의: t는 로컬 복사본
    return result;
}

// [l, r] 구간 뒤집기
Node* reverse(Node* t, int l, int r) {
    Node *left, *mid, *right;
    split(t, l, left, right);
    split(right, r - l + 1, mid, right);
    applyRev(mid);
    return merge(merge(left, mid), right);
}

// 중위 순회로 배열 출력
void print(Node* t) {
    if (!t) return;
    pushDown(t);
    print(t->left);
    cout << t->val << " ";
    print(t->right);
}

int main() {
    Node* root = nullptr;

    // 초기 배열: [1, 2, 3, 4, 5]
    for (int i = 1; i <= 5; i++)
        root = merge(root, new Node(i));

    cout << "초기: "; print(root); cout << "\n";
    // 출력: 초기: 1 2 3 4 5

    // 위치 2에 99 삽입 → [1, 2, 99, 3, 4, 5]
    root = insert(root, 2, 99);
    cout << "삽입 후: "; print(root); cout << "\n";
    // 출력: 삽입 후: 1 2 99 3 4 5

    // [1, 4] 구간 뒤집기 (0-indexed) → [1, 3, 99, 2, 4, 5]
    root = reverse(root, 1, 4);
    cout << "뒤집기 후: "; print(root); cout << "\n";
    // 출력: 뒤집기 후: 1 3 99 2 4 5

    return 0;
}
```

### 예제 2: Cut-and-Paste 연산 (CSES 문제 유형)

Implicit Treap의 대표적인 응용: 배열의 특정 구간을 잘라내어 다른 위치에 붙여넣기

```cpp
#include <bits/stdc++.h>
using namespace std;

// (위의 Node 구조체와 split/merge 함수 그대로 사용)
// ... (생략, 위와 동일)

// cut: [l, r] 구간을 잘라서 position p 뒤에 삽입
// 예: [1,2,3,4,5]에서 [1,3](0-indexed) 잘라서 p=1 뒤에 붙이기
//     → 잘린 구간: [2,3,4], 나머지: [1,5], 붙이기 위치: 1
//     → 결과: [1, 2, 3, 4, 5] (실제론 p 계산이 필요)
Node* cutAndPaste(Node* t, int l, int r, int p) {
    // 1단계: [l, r] 구간 분리
    Node *left, *mid, *right;
    split(t, l, left, right);
    split(right, r - l + 1, mid, right);
    // 현재: left=[0,l-1], mid=[l,r], right=[r+1,end]

    // 2단계: left와 right 합치기
    Node* rest = merge(left, right);
    // rest 길이 = 원래 길이 - (r-l+1)

    // 3단계: p 위치에 mid 삽입
    Node *a, *b;
    split(rest, p, a, b);
    return merge(merge(a, mid), b);
}

// 구간 최솟값 쿼리 (lazy 태그와 함께 활용)
struct NodeMin {
    int val, priority, size;
    int minVal;    // 서브트리 최솟값
    int add;       // 구간 덧셈 lazy 태그
    NodeMin *left, *right;

    NodeMin(int v) :
        val(v), priority(rand()), size(1),
        minVal(v), add(0), left(nullptr), right(nullptr) {}
};

void pushAdd(NodeMin* t, int delta) {
    if (!t) return;
    t->val    += delta;
    t->minVal += delta;
    t->add    += delta;
}

void pushDownMin(NodeMin* t) {
    if (!t || t->add == 0) return;
    pushAdd(t->left,  t->add);
    pushAdd(t->right, t->add);
    t->add = 0;
}

void updateMin(NodeMin* t) {
    if (!t) return;
    t->size   = 1 + (t->left ? t->left->size : 0) + (t->right ? t->right->size : 0);
    t->minVal = t->val;
    if (t->left)  t->minVal = min(t->minVal, t->left->minVal);
    if (t->right) t->minVal = min(t->minVal, t->right->minVal);
}

// 사용 패턴: 구간 [l, r]에 val 더하기
// split → pushAdd(mid, val) → merge

int main() {
    // Cut-and-Paste 예시
    // 배열: [1, 2, 3, 4, 5]
    Node* root = nullptr;
    for (int v : {1, 2, 3, 4, 5})
        root = merge(root, new Node(v));

    // [2, 4] 구간 (0-indexed, 즉 값 3,4,5)을 위치 0 뒤에 붙이기
    // 기대 결과: [1, 2, 3, 4, 5] → [1, 3, 4, 5, 2] (p=1에 [2,4] 삽입)
    // 잘라낸 구간 [2,4]: [3, 4, 5], 나머지: [1, 2], p=1
    root = cutAndPaste(root, 2, 4, 1);
    cout << "Cut-and-Paste 결과: ";
    print(root);
    cout << "\n";
    // 출력: Cut-and-Paste 결과: 1 3 4 5 2

    return 0;
}
```

## Implicit Treap의 복잡도 분석

| 연산 | 시간복잡도 | 비고 |
|------|-----------|------|
| 임의 위치 삽입 | O(log N) 기댓값 | Split + Merge |
| 임의 위치 삭제 | O(log N) 기댓값 | Split + Split + Merge |
| 구간 쿼리 (합, 최솟값 등) | O(log N) 기댓값 | Split + 계산 + Merge |
| 구간 업데이트 (lazy) | O(log N) 기댓값 | Split + pushAdd + Merge |
| 구간 뒤집기 | O(log N) 기댓값 | Split + applyRev + Merge |
| Cut-and-Paste | O(log N) 기댓값 | 2회 Split + Merge |

**공간복잡도**: O(N) — N개 노드

## 언제 Implicit Treap을 써야 하는가?

**Implicit Treap이 적합한 경우**:
- 배열 중간에 빈번한 삽입/삭제가 필요할 때
- 구간 뒤집기(reversal)가 필요할 때 (세그먼트 트리로는 어려움)
- Cut-and-Paste 연산이 필요할 때
- 구간 업데이트 + 구간 쿼리 + 삽입/삭제를 동시에 지원해야 할 때

**세그먼트 트리가 더 나은 경우**:
- 배열 크기가 고정 (삽입/삭제 불필요)
- 구간 뒤집기가 필요 없는 단순 구간 연산

**`std::vector`가 더 나은 경우**:
- 삽입/삭제가 주로 끝에서만 발생 (amortized O(1))
- N이 작아서 O(N) 연산도 빠름

## 주의사항과 팁

### 1. Split/Merge 후 반드시 루트 갱신

Split과 Merge는 트리 구조를 바꾸므로, 반드시 반환된 새 루트를 사용해야 합니다:

```cpp
// ❌ 잘못된 사용: root가 갱신되지 않음
Node *l, *r;
split(root, k, l, r);
// root는 이제 무효 상태

// ✅ 올바른 사용
root = merge(l, r);  // 사용 후 반드시 재결합
```

### 2. 메모리 풀 사용

경쟁 프로그래밍에서는 `new/delete` 오버헤드를 줄이기 위해 정적 배열 기반 메모리 풀을 사용합니다:

```cpp
const int MAXN = 300005;
Node pool[MAXN];
int poolIdx = 0;

Node* newNode(int val) {
    pool[poolIdx] = Node(val);
    return &pool[poolIdx++];
}
```

### 3. pushDown 타이밍

`split`과 `merge` 진입 시 반드시 `pushDown`을 먼저 호출해야 lazy 태그가 올바르게 전파됩니다. 이를 빠뜨리면 뒤집기나 구간 업데이트가 잘못 적용됩니다.

### 4. 순수하게 C++ `rope` 활용 (간단한 경우)

GNU C++ 확장의 `__gnu_pbds::tree` 나 `std::rope`를 사용할 수도 있지만, 커스텀 lazy 태그가 필요한 경우에는 직접 구현이 필요합니다.

## 실전 적용 사례

- **텍스트 에디터**: 문서를 Implicit Treap으로 관리 → 임의 위치 삽입/삭제 O(log N)
- **버전 관리 자료구조**: Persistent Implicit Treap → 구조 공유로 이전 버전 보존
- **온라인 알고리즘**: 스트림에서 k번째 원소 삽입/삭제 쿼리

## 참고 자료
- [Treaps / Implicit Treap - USACO Guide (Advanced)](https://usaco.guide/adv/treaps?lang=cpp)
- [Implicit Treap - mtuann Competitive Programming](https://mtuann.github.io/competitive-programming-cpp/topics/data-structures/treap-implicit/)
- [Codeforces: Treap with Implicit Keys - codemia.io](https://codemia.io/knowledge-hub/path/treap_with_implicit_keys)
- [CP-algorithms: Treap](https://cp-algorithms.com/data_structures/treap.html)
