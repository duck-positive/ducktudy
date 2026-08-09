---
layout: post
title: "링크-컷 트리(Link-Cut Tree) 완전 정복: 동적 트리 경로 쿼리를 O(log N) 분할 상환으로 해결하기"
date: 2026-08-09
categories: [cs, computer-science]
tags: [link-cut-tree, splay-tree, dynamic-tree, data-structure, graph, competitive-programming]
---

## 링크-컷 트리란 무엇인가

링크-컷 트리(Link-Cut Tree, 이하 LCT)는 **동적 포레스트(dynamic forest)**를 관리하기 위한 고급 자료구조다. 트리에서 간선을 추가(link)하거나 삭제(cut)하는 동적 연산과, 두 노드 사이의 경로에 대한 집계 쿼리(aggregate query)를 **분할 상환 O(log N)** 시간에 처리할 수 있다.

1983년 Sleator와 Tarjan이 발표한 이 자료구조는, 내부적으로 **스플레이 트리(Splay Tree)**를 사용하여 경로를 묶고 관리한다. 이해하기 쉬운 자료구조는 아니지만, 한 번 익히면 경쟁 프로그래밍의 최상급 문제와 분산 시스템 알고리즘 설계에서 매우 강력한 도구가 된다.

---

## 왜 링크-컷 트리가 필요한가

정적 트리에서 두 노드 사이 경로의 최솟값, 합, XOR 등을 구하는 문제는 **Heavy-Light Decomposition(HLD)**이나 **Euler Tour + 세그먼트 트리**로 O(log²N) 혹은 O(log N)에 해결 가능하다. 그러나 이 접근법들은 **트리 구조가 고정**되어 있을 때만 유효하다.

실제 시스템에서는 트리 구조가 동적으로 변하는 상황이 많다:

- **동적 연결성 문제**: 간선이 추가·삭제되며 두 노드가 같은 컴포넌트에 속하는지 판단해야 할 때
- **동적 MST(최소 신장 트리)**: 간선 삽입·삭제 후 MST를 효율적으로 유지할 때
- **오프라인 LCA**: 쿼리마다 트리 루트가 바뀌는 문제

이런 문제들을 O(N)씩 재계산하면 총 복잡도가 O(NQ)가 되어 대규모 입력에서 시간 초과가 난다. 링크-컷 트리는 이 모든 연산을 분할 상환 O(log N)으로 처리해 O(Q log N)을 달성한다.

---

## 핵심 개념: 선호 경로 분해 (Preferred Path Decomposition)

LCT의 핵심 아이디어는 트리의 모든 노드를 **선호 경로(preferred path)**라는 수직 경로들로 분리하는 것이다.

### 선호 간선(Preferred Edge)

각 노드는 자식 중 최대 하나와 **선호 간선**으로 연결된다. 선호 간선으로 연결된 노드들의 연속된 체인이 선호 경로를 이룬다. 선호 간선이 아닌 나머지 간선은 **경로 간 간선(path-parent pointer)**이라 부른다.

### 보조 트리(Auxiliary Tree)

각 선호 경로는 **스플레이 트리**로 표현된다. 스플레이 트리의 키는 원본 트리에서의 **깊이(depth)**로, 같은 경로에 속한 노드들이 깊이 순으로 정렬된 BST를 이룬다.

- **실선(solid edge)**: 같은 선호 경로 안의 간선 (스플레이 트리 내부 간선)
- **허선(dashed edge)**: 서로 다른 선호 경로를 연결하는 경로 간 간선

### access(v) 연산

LCT의 모든 연산은 `access(v)`를 중심으로 구성된다. `access(v)`는 루트에서 v까지의 경로를 하나의 선호 경로로 만들고, 해당 경로를 표현하는 스플레이 트리에서 v를 루트로 스플레이하는 연산이다.

```
access(v):
  1. v를 v가 속한 보조 트리에서 스플레이한다.
  2. v의 오른쪽 자식을 잘라낸다 (v 아래쪽 경로를 분리).
  3. 경로 간 간선을 따라 부모 u로 이동한다.
  4. u를 u의 보조 트리에서 스플레이한다.
  5. u의 오른쪽 자식을 v로 교체한다 (u~v를 같은 경로로 합침).
  6. u에서 3~5를 반복하며 루트까지 올라간다.
```

`access`를 반복하면 루트~v 경로 전체가 하나의 스플레이 트리에 담기며, v가 그 트리의 루트(최대 깊이 노드)가 된다.

---

## 구현 예제

### 예제 1: LCT 핵심 구조 (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

struct Node {
    int ch[2], p;   // 좌/우 자식, 부모
    int val, agg;   // 노드 값, 경로 집계 값 (예: XOR)
    bool rev;       // 반전 지연 플래그

    Node() : ch{0, 0}, p(0), val(0), agg(0), rev(false) {}
};

const int MAXN = 200005;
Node t[MAXN];

// v가 보조 트리의 루트인지 확인 (부모가 없거나 부모의 자식이 아님)
bool isRoot(int v) {
    int p = t[v].p;
    return t[p].ch[0] != v && t[p].ch[1] != v;
}

// 집계 값 업데이트
void pushUp(int v) {
    t[v].agg = t[t[v].ch[0]].agg ^ t[v].val ^ t[t[v].ch[1]].agg;
}

// 반전 플래그 전파
void pushRev(int v) {
    if (!v) return;
    swap(t[v].ch[0], t[v].ch[1]);
    t[v].rev ^= 1;
}

// 지연 플래그 하향 전파
void pushDown(int v) {
    if (t[v].rev) {
        pushRev(t[v].ch[0]);
        pushRev(t[v].ch[1]);
        t[v].rev = false;
    }
}

// 스플레이 회전
void rotate(int v) {
    int u = t[v].p, g = t[u].p;
    int k = (t[u].ch[1] == v);
    if (!isRoot(u)) t[g].ch[t[g].ch[1] == u] = v;
    t[v].p = g;
    t[u].ch[k] = t[v].ch[!k];
    if (t[v].ch[!k]) t[t[v].ch[!k]].p = u;
    t[v].ch[!k] = u;
    t[u].p = v;
    pushUp(u);
    pushUp(v);
}

// 스플레이 (v를 보조 트리 루트로)
void splay(int v) {
    static int stk[MAXN];
    int top = 0, u = v;
    stk[top++] = u;
    while (!isRoot(u)) stk[top++] = u = t[u].p;
    while (top) pushDown(stk[--top]);
    while (!isRoot(v)) {
        int u = t[v].p, g = t[u].p;
        if (!isRoot(u)) {
            (t[g].ch[0] == u) == (t[u].ch[0] == v)
                ? rotate(u) : rotate(v);
        }
        rotate(v);
    }
    pushUp(v);
}

// access: 루트~v 경로를 하나의 보조 트리로
void access(int v) {
    int last = 0;
    for (int u = v; u; u = t[u].p) {
        splay(u);
        t[u].ch[1] = last;
        pushUp(u);
        last = u;
    }
    splay(v);
}

// makeRoot: v를 트리의 루트로 만들기
void makeRoot(int v) {
    access(v);
    pushRev(v);
}

// findRoot: v가 속한 트리의 루트 반환
int findRoot(int v) {
    access(v);
    while (t[v].ch[0]) {
        pushDown(v);
        v = t[v].ch[0];
    }
    splay(v);
    return v;
}

// link: u와 v 사이 간선 추가 (u, v가 다른 트리에 속해야 함)
void link(int u, int v) {
    makeRoot(u);
    if (findRoot(v) != u) {
        t[u].p = v;
    }
}

// cut: u와 v 사이 간선 제거
void cut(int u, int v) {
    makeRoot(u);
    access(v);
    // 이 시점에서 u는 v의 왼쪽 자식이어야 함
    if (t[v].ch[0] == u && !t[u].ch[1]) {
        t[v].ch[0] = 0;
        t[u].p = 0;
        pushUp(v);
    }
}

// query: u~v 경로의 집계 값 (예: XOR) 반환
int query(int u, int v) {
    makeRoot(u);
    access(v);
    return t[v].agg;
}
```

### 예제 2: 동적 연결성 및 경로 XOR 쿼리 활용

```cpp
int main() {
    ios_base::sync_with_stdio(false);
    cin.tie(nullptr);

    int n, q;
    cin >> n >> q;

    // 초기 노드 값 설정
    for (int i = 1; i <= n; i++) {
        cin >> t[i].val;
        t[i].agg = t[i].val;
    }

    while (q--) {
        int op, u, v;
        cin >> op >> u >> v;

        if (op == 1) {
            // 간선 추가 (단, 사이클을 만들지 않는 경우만)
            link(u, v);
        } else if (op == 2) {
            // 간선 제거
            cut(u, v);
        } else if (op == 3) {
            // u~v 경로 XOR 합 쿼리
            // 같은 트리에 속하는지 먼저 확인
            makeRoot(u);
            if (findRoot(v) == u) {
                cout << query(u, v) << '\n';
            } else {
                cout << -1 << '\n'; // 연결되지 않음
            }
        } else {
            // 노드 값 업데이트
            splay(u);
            t[u].val = v;
            pushUp(u);
        }
    }

    return 0;
}
```

위 코드는 BOJ 13726 "LCT" 유형의 문제를 풀 수 있는 완전한 링크-컷 트리 구현이다. `query(u, v)`에서 `makeRoot(u)`, `access(v)` 후 `t[v].agg`가 u~v 경로의 집계 값이 된다.

---

## 복잡도 분석

| 연산 | 시간 복잡도 |
|------|-------------|
| access(v) | 분할 상환 O(log N) |
| link(u, v) | 분할 상환 O(log N) |
| cut(u, v) | 분할 상환 O(log N) |
| findRoot(v) | 분할 상환 O(log N) |
| query(u, v) | 분할 상환 O(log N) |

`access`의 분할 상환 분석은 스플레이 트리의 포텐셜 함수 분석과 유사하다. 각 노드의 포텐셜을 `Φ(v) = log(size(v))`로 정의하면, `access`에서 선호 간선이 바뀌는 횟수의 합이 O(Q log N)으로 수렴함을 증명할 수 있다.

---

## 주의사항 및 팁

### 1. `isRoot` 판별의 중요성

LCT에서 가장 흔한 버그는 스플레이 트리의 루트 판별이다. 보조 트리의 루트는 **부모 포인터는 있지만, 부모의 자식이 아닌** 상태다. 따라서 `isRoot(v) = (t[t[v].p].ch[0] != v && t[t[v].p].ch[1] != v)`로 판별해야 한다.

### 2. `pushDown` 순서

`splay`에서 회전하기 전에 반드시 루트부터 v까지의 경로에 있는 모든 노드에 대해 `pushDown`을 먼저 적용해야 한다. 그렇지 않으면 반전 플래그가 올바르게 전파되지 않는다. 스택을 이용해 경로를 저장한 뒤 역순으로 `pushDown`하는 방식이 안전하다.

### 3. 비연결 그래프에서의 `link`

`link(u, v)` 전에 반드시 u와 v가 서로 다른 트리에 속하는지 확인해야 한다. 같은 트리에 있는 노드를 link하면 사이클이 생기고 LCT의 불변 조건이 깨진다. `findRoot`를 이용해 연결 여부를 먼저 확인하자.

### 4. `makeRoot` + `access`의 의미

`query(u, v)`에서 `makeRoot(u)`를 호출하면 u가 트리의 루트가 된다. 이어서 `access(v)`를 호출하면 u~v 경로 전체가 하나의 보조 스플레이 트리에 담기고, v가 해당 트리의 루트가 된다. 이때 `t[v].agg`가 u~v 경로의 집계 값이다.

### 5. Heavy-Light Decomposition과의 선택 기준

- 트리가 정적이라면 HLD나 세그먼트 트리가 구현이 훨씬 단순하다.
- 트리가 동적이어야 하거나 포레스트 상에서 연결성 유지가 필요하다면 LCT를 선택한다.
- LCT의 상수 인자는 꽤 크므로, N이 작다면 O(N log²N) HLD도 충분히 빠를 수 있다.

---

## 참고 자료

- [PetarV- / Algorithms — Link-cut Tree.cpp (C++ 구현 레퍼런스)](https://github.com/PetarV-/Algorithms/blob/master/Data%20Structures/Link-cut%20Tree.cpp)
- [saadtaame / link-cut-tree (동적 연결성 응용 구현)](https://github.com/saadtaame/link-cut-tree)
