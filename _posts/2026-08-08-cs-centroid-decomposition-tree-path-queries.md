---
layout: post
title: "센트로이드 분해(Centroid Decomposition) 완전 정복: 트리 경로 쿼리를 O(N log²N)에 정복하는 분할 정복"
date: 2026-08-08
categories: [cs, computer-science]
tags: [algorithm, centroid-decomposition, tree, divide-and-conquer, graph-theory, competitive-programming]
---

## 개념 설명: 센트로이드 분해란?

**센트로이드 분해(Centroid Decomposition)**는 트리에서의 분할 정복(Divide and Conquer) 기법으로, 트리를 **센트로이드(centroid)** 를 기준으로 재귀적으로 분해해 트리 경로와 관련된 다양한 쿼리를 효율적으로 처리하는 알고리즘이다.

### 센트로이드(Centroid)란?

N개의 노드로 이루어진 트리에서, 노드 C를 제거했을 때 남은 연결 요소(connected component) 중 어느 것도 ⌊N/2⌋개를 초과하지 않는 노드를 **센트로이드**라 한다. 모든 트리에는 항상 1개 또는 2개의 센트로이드가 존재함이 알려져 있다.

**핵심 성질**: 트리를 센트로이드로 분해할 때 생성되는 **센트로이드 분해 트리**의 높이는 항상 **O(log N)** 이다. 이는 센트로이드를 제거할 때마다 각 서브트리의 크기가 절반 이하로 줄어들기 때문이다.

### 센트로이드 분해 과정

1. 현재 트리(또는 서브트리)에서 센트로이드 C를 찾는다.
2. C를 센트로이드 분해 트리의 루트(또는 해당 서브트리의 루트)로 설정한다.
3. C를 "제거"(방문 완료 표시)한 뒤, 남은 각 연결 요소에서 재귀적으로 과정을 반복한다.

이때 "제거"는 실제 그래프 수정이 아니라 `removed[C] = true` 표시로 구현한다. 이후 센트로이드를 찾는 DFS는 `removed` 표시된 노드를 무시한다.

### 왜 높이가 O(log N)인가?

센트로이드 C를 제거하면, 남은 모든 서브트리의 크기는 원래 트리 크기의 절반 이하다. 따라서 재귀 호출 깊이(센트로이드 분해 트리의 높이)는 T(N) ≤ T(N/2) + 1 = O(log N)이 된다.

---

## 왜 필요한가?

트리에서의 경로 관련 문제는 흔히 두 노드 u, v 사이의 경로에 관한 쿼리를 다수 처리하는 형태로 나타난다. 예를 들어:

- **두 노드 사이 최단 거리 쿼리**: 각 쿼리를 단순 LCA로 처리하면 O(log N)이지만, 거리 조건이 붙으면 복잡해진다.
- **경로 위 특정 조건 만족 경로 수 세기**: "거리가 정확히 K인 쌍의 수는?"
- **경로 위 값의 합/최대/최소**: 업데이트 포함 쿼리

이런 문제들을 세그먼트 트리나 HLD(Heavy-Light Decomposition)로 풀 수 있는 경우도 있지만, **"트리에서 특정 길이의 경로 수"** 같은 문제는 센트로이드 분해가 가장 자연스럽고 효율적인 해법을 제공한다.

센트로이드 분해의 핵심 관찰: **트리의 임의 경로 (u, v)는 반드시 해당 경로 위에 있는 가장 위쪽 센트로이드를 지난다**. 센트로이드 분해 트리에서 모든 노드는 O(log N)개의 조상을 가지므로, 각 노드에서 O(log N)번만 처리하면 모든 경로를 커버할 수 있다.

---

## 실제 구현 예제

### 예제 1: 거리가 K인 경로 쌍 수 세기 (C++)

N개의 노드로 이루어진 가중치 없는 트리에서 두 노드 사이 거리(엣지 수)가 정확히 K인 쌍 (u, v)의 수를 구하라.

```cpp
#include <bits/stdc++.h>
using namespace std;

const int MAXN = 100005;
vector<int> adj[MAXN];
int subtree_size[MAXN];
bool removed[MAXN];
int n, k;
long long ans = 0;

// 서브트리 크기 계산
void calc_size(int v, int parent) {
    subtree_size[v] = 1;
    for (int u : adj[v]) {
        if (u == parent || removed[u]) continue;
        calc_size(u, v);
        subtree_size[v] += subtree_size[u];
    }
}

// 센트로이드 찾기: 서브트리 크기가 tree_size/2 이하인 노드
int find_centroid(int v, int parent, int tree_size) {
    for (int u : adj[v]) {
        if (u == parent || removed[u]) continue;
        if (subtree_size[u] > tree_size / 2)
            return find_centroid(u, v, tree_size);
    }
    return v;
}

// 센트로이드 C에서 깊이 depth인 노드들의 깊이 목록 수집
void collect_depths(int v, int parent, int depth, vector<int>& depths) {
    depths.push_back(depth);
    for (int u : adj[v]) {
        if (u == parent || removed[u]) continue;
        collect_depths(u, v, depth + 1, depths);
    }
}

// 깊이 목록에서 합이 K인 쌍 수 세기
long long count_pairs(vector<int>& depths) {
    sort(depths.begin(), depths.end());
    long long cnt = 0;
    int l = 0, r = (int)depths.size() - 1;
    while (l < r) {
        int s = depths[l] + depths[r];
        if (s == k) {
            if (depths[l] == depths[r]) {
                long long len = r - l + 1;
                cnt += len * (len - 1) / 2;
                break;
            }
            int cl = 1, cr = 1;
            while (l + cl < r && depths[l + cl] == depths[l]) cl++;
            while (r - cr > l && depths[r - cr] == depths[r]) cr++;
            cnt += (long long)cl * cr;
            l += cl; r -= cr;
        } else if (s < k) l++;
        else r--;
    }
    return cnt;
}

// 센트로이드 분해 메인
void decompose(int v) {
    calc_size(v, -1);
    int c = find_centroid(v, -1, subtree_size[v]);

    removed[c] = true;

    // c 자체(깊이 0)를 포함한 전체 깊이 목록
    vector<int> all_depths = {0};
    for (int u : adj[c]) {
        if (removed[u]) continue;
        vector<int> sub_depths;
        collect_depths(u, c, 1, sub_depths);

        // 포함-배제: c를 통과하는 경로 수 = 전체 - c를 통과하지 않는 수
        // 먼저 sub_depths와 all_depths의 쌍을 구하면 c를 통과하는 경로 포함
        for (int d : sub_depths) all_depths.push_back(d);
    }
    ans += count_pairs(all_depths);

    // 포함-배제: 같은 서브트리 내 경로(c를 경유하지 않음) 제거
    all_depths = {0};
    for (int u : adj[c]) {
        if (removed[u]) continue;
        vector<int> sub_depths;
        collect_depths(u, c, 1, sub_depths);
        ans -= count_pairs(sub_depths);  // 오버카운팅 제거
    }

    // 재귀적으로 각 서브트리 처리
    for (int u : adj[c]) {
        if (!removed[u]) decompose(u);
    }
}

int main() {
    ios_base::sync_with_stdio(false);
    cin.tie(NULL);

    cin >> n >> k;
    for (int i = 0; i < n - 1; i++) {
        int u, v;
        cin >> u >> v;
        adj[u].push_back(v);
        adj[v].push_back(u);
    }

    decompose(1);
    cout << ans << "\n";
    return 0;
}
```

**시간복잡도 분석**:
- 센트로이드 분해: O(N log N)
- 각 레벨에서 깊이 수집 및 정렬: O(N log N) (각 노드가 O(log N) 레벨에 참여)
- 전체: **O(N log² N)**

---

### 예제 2: 센트로이드 분해 트리 구축 및 온라인 쿼리 처리 (C++)

센트로이드 분해 트리를 실제로 빌드해 "각 노드의 k번째 조상" 형태 쿼리를 지원하는 구조를 만든다.

```cpp
#include <bits/stdc++.h>
using namespace std;

const int MAXN = 100005;
vector<int> adj[MAXN];
int sz[MAXN], centroid_parent[MAXN];
bool removed[MAXN];
int n;

void calc_size(int v, int p) {
    sz[v] = 1;
    for (int u : adj[v]) {
        if (u == p || removed[u]) continue;
        calc_size(u, v);
        sz[v] += sz[u];
    }
}

int find_centroid(int v, int p, int tree_sz) {
    for (int u : adj[v]) {
        if (u == p || removed[u]) continue;
        if (sz[u] > tree_sz / 2) return find_centroid(u, v, tree_sz);
    }
    return v;
}

// 센트로이드 분해 트리 빌드
// centroid_parent[v] = 센트로이드 분해 트리에서 v의 부모
void build(int v, int parent_centroid) {
    calc_size(v, -1);
    int c = find_centroid(v, -1, sz[v]);

    centroid_parent[c] = parent_centroid;
    removed[c] = true;

    for (int u : adj[c]) {
        if (!removed[u]) build(u, c);
    }
}

// 센트로이드 분해 트리에서 v에서 루트까지 조상 목록 반환
vector<int> get_ancestors(int v) {
    vector<int> ancestors;
    int cur = v;
    while (cur != -1) {
        ancestors.push_back(cur);
        cur = centroid_parent[cur];
    }
    return ancestors;  // v, 부모, 조부모, ... 순
}

// 활용: 두 노드 u, v 사이 경로 위의 센트로이드 분해 트리 LCA 찾기
int centroid_lca(int u, int v) {
    // 두 노드의 조상 집합을 구하고 교집합의 가장 낮은 공통 조상을 찾는 방식
    set<int> ancestors_u;
    for (int a : get_ancestors(u)) ancestors_u.insert(a);
    for (int a : get_ancestors(v))
        if (ancestors_u.count(a)) return a;
    return -1;
}

int main() {
    ios_base::sync_with_stdio(false);
    cin.tie(NULL);

    cin >> n;
    for (int i = 0; i < n - 1; i++) {
        int u, v; cin >> u >> v;
        adj[u].push_back(v);
        adj[v].push_back(u);
    }

    fill(centroid_parent, centroid_parent + n + 1, -1);
    build(1, -1);

    // 각 노드의 센트로이드 분해 트리 조상 출력
    for (int i = 1; i <= n; i++) {
        cout << "노드 " << i << "의 c-tree 부모: "
             << (centroid_parent[i] == -1 ? "루트" : to_string(centroid_parent[i])) << "\n";
    }
    return 0;
}
```

이 구조를 기반으로 각 노드에 대한 정보를 센트로이드 분해 트리를 따라 조상별로 집계하면, 다양한 트리 경로 쿼리를 O(log N) ~ O(log² N)에 처리할 수 있다.

---

## 주의사항 및 팁

### 1. 포함-배제 원리의 적용

센트로이드 분해에서 경로 수를 셀 때 흔한 실수는 **같은 서브트리 안에서 센트로이드를 경유하지 않는 경로를 이중 계산**하는 것이다. 위 예제처럼 전체 카운트에서 각 서브트리 내부 카운트를 빼는 포함-배제 기법을 반드시 적용해야 한다.

### 2. removed 배열 초기화 불필요

`removed[]` 배열은 분해 과정에서 한번 설정되면 재설정하지 않는다. 분해가 끝난 센트로이드는 영구히 제거 상태로 표시된다. 이는 알고리즘의 특성이지 버그가 아니다.

### 3. calc_size의 루트 설정

`calc_size(v, -1)`을 호출할 때 루트는 분해 중인 컴포넌트의 임의 노드여도 무방하다. 센트로이드 탐색은 올바른 결과를 보장한다. 단, `removed` 노드는 무시해야 크기 계산이 정확해진다.

### 4. 업데이트 포함 버전

정적 쿼리뿐 아니라 **노드 값 업데이트**가 있는 경우에도 센트로이드 분해 트리를 활용할 수 있다. 각 노드 v에서 센트로이드 분해 트리 조상들을 따라 올라가며 정보를 갱신하면 O(log N)에 업데이트가 가능하다. 이 패턴을 활용하면 트리 경로 합, 최소값 등의 쿼리를 O(log² N)에 처리하는 동적 트리 문제를 풀 수 있다.

### 5. 시간복잡도 정리

| 연산 | 복잡도 |
|------|--------|
| 센트로이드 분해 트리 구축 | O(N log N) |
| 정적 경로 쿼리 (단순 집계) | O(N log N) 전처리, O(log N) 쿼리 |
| 동적 업데이트 + 경로 쿼리 | O(log N) 업데이트, O(log² N) 쿼리 |
| 거리 K인 경로 수 (정렬 포함) | O(N log² N) |

### 6. HLD와의 비교

HLD는 경로 위 구간 쿼리(구간 최소/최대/합)에 특화된 반면, 센트로이드 분해는 **두 노드 사이 거리 조건**이나 **특정 합을 갖는 경로 수** 같은 문제에 더 자연스럽다. 문제 유형에 따라 두 기법을 선택하거나 조합할 수 있다.

---

## 참고 자료

- [davidedellagiustina - Linear Centroid Decomposition (O(N) 선형 시간 알고리즘 구현)](https://github.com/davidedellagiustina/linear-centroid-decomposition)
- [mochow13 - Centroid Decomposition Sample 구현 (GCD 경로 쿼리 적용)](https://github.com/mochow13/competitive-programming-library/blob/master/Data%20Structures/Centroid%20Decomposition%20Sample.cpp)
- [hoanghai1803 - ITK21NBK 커리큘럼 (센트로이드 분해 포함)](https://github.com/hoanghai1803/ITK21NBK)
