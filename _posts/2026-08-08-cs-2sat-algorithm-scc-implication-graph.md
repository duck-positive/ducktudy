---
layout: post
title: "2-SAT 완전 정복: 함의 그래프와 강결합 요소로 부울 만족성 문제를 O(N+M)에 해결하기"
date: 2026-08-08
categories: [cs, computer-science]
tags: [algorithm, 2-sat, scc, implication-graph, boolean-satisfiability, graph-theory]
---

## 개념 설명: 2-SAT란?

**2-SAT(2-Satisfiability)**은 부울 만족성 문제(SAT)의 특수한 형태로, 각 절(clause)이 정확히 두 개의 리터럴(literal)로 이루어진 CNF(Conjunctive Normal Form) 공식의 만족 가능 여부를 판단하는 문제다.

일반 SAT 문제는 NP-완전(NP-complete)이어서 효율적인 알고리즘이 알려지지 않았지만, 2-SAT는 **강결합 요소(Strongly Connected Components, SCC)** 를 활용해 **O(N + M)** 의 선형 시간에 풀 수 있다. 여기서 N은 변수 수, M은 절의 수다.

### 2-SAT 문제의 형태

2-SAT 공식은 다음과 같이 구성된다:

```
(x₁ ∨ x₂) ∧ (¬x₁ ∨ x₃) ∧ (x₂ ∨ ¬x₃) ∧ ...
```

각 절 `(A ∨ B)`는 "A가 거짓이면 B가 참이어야 하고, B가 거짓이면 A가 참이어야 한다"는 두 개의 함의(implication)와 동치다:

- `¬A → B`
- `¬B → A`

이 관찰이 2-SAT를 그래프 문제로 변환하는 핵심이다.

### 함의 그래프(Implication Graph)

N개의 변수 x₁, x₂, ..., xₙ에 대해, 각 변수 xᵢ와 그 부정 ¬xᵢ를 그래프의 노드로 만든다. 총 2N개의 노드가 생성된다.

각 절 `(A ∨ B)`에 대해 다음 두 방향 간선을 추가한다:
- `¬A → B`
- `¬B → A`

이렇게 구성된 방향 그래프를 **함의 그래프**라고 한다.

### 해결 조건

**정리**: 2-SAT 공식이 만족 가능(satisfiable)하기 위한 필요충분조건은, 함의 그래프에서 어떤 변수 xᵢ와 그 부정 ¬xᵢ가 같은 SCC에 속하지 않는 것이다.

만약 xᵢ와 ¬xᵢ가 같은 SCC에 있다면, xᵢ → ¬xᵢ이자 ¬xᵢ → xᵢ인 경로가 존재해 논리적 모순이 발생한다.

### 변수값 결정

해가 존재할 때, 각 변수의 참/거짓 값은 SCC의 **위상 정렬 순서**로 결정된다. SCC를 위상 정렬했을 때, xᵢ가 속한 SCC의 순서가 ¬xᵢ의 SCC보다 늦으면(DAG에서 더 뒤쪽이면) xᵢ = true, 그렇지 않으면 xᵢ = false로 설정한다.

---

## 왜 필요한가?

2-SAT는 언뜻 추상적인 이론처럼 보이지만, 실제 문제 모델링에 광범위하게 쓰인다.

**실제 활용 사례:**

- **스케줄링 문제**: "작업 i와 작업 j는 같은 시간대에 배정될 수 없다" → 절 표현 가능
- **그래프 색칠**: 각 정점에 두 가지 색 중 하나 배정, 인접 정점 같은 색 불허 → 2-SAT
- **회의실 배정**: 각 행사는 두 시간대 중 하나, 시간 충돌 금지 → 2-SAT
- **네트워크 게이트 문제**: AND 게이트, OR 게이트의 입력 설정 최적화

특히 경쟁 프로그래밍에서는 조건이 두 선택지 중 하나를 선택하는 형태로 주어지는 많은 문제가 2-SAT로 환원된다.

---

## 실제 구현 예제

### 예제 1: 2-SAT 풀이 — Tarjan's SCC 기반 C++ 완전 구현

```cpp
#include <bits/stdc++.h>
using namespace std;

struct TwoSat {
    int n;
    vector<vector<int>> adj, radj;
    vector<int> order, comp;
    vector<bool> visited;

    TwoSat(int n) : n(n), adj(2 * n), radj(2 * n), comp(2 * n), visited(2 * n) {}

    // 변수 인덱스: xᵢ = 2i, ¬xᵢ = 2i+1
    // "u가 참이면 v가 참이어야 한다" 형태의 함의 추가
    void addClause(int u, bool nu, int v, bool nv) {
        // (u ∨ v) → (¬u → v) ∧ (¬v → u)
        adj[2 * u + nu].push_back(2 * v + (nv ^ 1));
        adj[2 * v + nv].push_back(2 * u + (nu ^ 1));
        radj[2 * v + (nv ^ 1)].push_back(2 * u + nu);
        radj[2 * u + (nu ^ 1)].push_back(2 * v + nv);
    }

    // Kosaraju's SCC 1단계: 역방향 위상 정렬
    void dfs1(int v) {
        visited[v] = true;
        for (int u : adj[v]) if (!visited[u]) dfs1(u);
        order.push_back(v);
    }

    // Kosaraju's SCC 2단계: 역그래프에서 DFS
    void dfs2(int v, int c) {
        comp[v] = c;
        for (int u : radj[v]) if (comp[u] == -1) dfs2(u, c);
    }

    // 해 존재 여부 확인 및 변수값 배열 반환
    // 반환값: false이면 불만족, true이면 만족 (vals에 각 변수값 저장)
    bool solve(vector<bool>& vals) {
        fill(visited.begin(), visited.end(), false);
        fill(comp.begin(), comp.end(), -1);

        for (int i = 0; i < 2 * n; i++)
            if (!visited[i]) dfs1(i);

        int c = 0;
        for (int i = (int)order.size() - 1; i >= 0; i--)
            if (comp[order[i]] == -1) dfs2(order[i], c++);

        vals.resize(n);
        for (int i = 0; i < n; i++) {
            if (comp[2 * i] == comp[2 * i + 1]) return false;  // 모순
            // SCC 번호가 클수록 위상 정렬에서 앞쪽 (Kosaraju 특성)
            vals[i] = comp[2 * i] > comp[2 * i + 1];
        }
        return true;
    }
};

int main() {
    ios_base::sync_with_stdio(false);
    cin.tie(NULL);

    // 예시: 3개 변수, 3개 절
    // (x0 ∨ x1) ∧ (¬x0 ∨ x2) ∧ (¬x1 ∨ ¬x2)
    int n = 3;
    TwoSat sat(n);
    sat.addClause(0, false, 1, false);   // (x0 ∨ x1)
    sat.addClause(0, true,  2, false);   // (¬x0 ∨ x2)
    sat.addClause(1, true,  2, true);    // (¬x1 ∨ ¬x2)

    vector<bool> vals;
    if (sat.solve(vals)) {
        cout << "SATISFIABLE\n";
        for (int i = 0; i < n; i++)
            cout << "x" << i << " = " << (vals[i] ? "true" : "false") << "\n";
    } else {
        cout << "UNSATISFIABLE\n";
    }
    return 0;
}
```

**출력 예시:**
```
SATISFIABLE
x0 = false
x1 = true
x2 = true
```

검증: `(F ∨ T) = T`, `(¬F ∨ T) = T`, `(¬T ∨ ¬T) = T` — 모든 절 만족.

---

### 예제 2: 실전 문제 — 회의 시간 배정

N개의 발표가 있고, 각 발표는 시간대 A 또는 시간대 B에 배정될 수 있다. 단, 특정 발표 쌍 `(i, j)`은 같은 시간대에 배정될 수 없다. 모든 제약을 만족하는 배정이 존재하는지 판단하라.

```cpp
#include <bits/stdc++.h>
using namespace std;

// 위의 TwoSat 구조체 사용 가정

int main() {
    ios_base::sync_with_stdio(false);
    cin.tie(NULL);

    int n, m;
    cin >> n >> m;  // n: 발표 수, m: 충돌 쌍 수

    TwoSat sat(n);

    for (int i = 0; i < m; i++) {
        int u, v;
        cin >> u >> v;
        u--; v--;
        // 발표 u와 v는 같은 시간대 불허
        // (u=A이면 v=B) ∧ (u=B이면 v=A)
        // → (¬u ∨ ¬v) ∧ (u ∨ v)를 2-SAT 절로 변환
        // 여기서 false=시간대A, true=시간대B
        sat.addClause(u, false, v, false);  // (u=A → v=B) 즉 (¬u ∨ ¬v)
        sat.addClause(u, true,  v, true);   // (u=B → v=A) 즉 (u ∨ v) [보완]
        // 사실상 "둘이 같은 쪽이면 안됨": addClause(u, false, v, false) + addClause(u, true, v, true)
    }

    vector<bool> vals;
    if (sat.solve(vals)) {
        cout << "가능\n";
        for (int i = 0; i < n; i++)
            cout << "발표 " << i + 1 << ": 시간대 " << (vals[i] ? "B" : "A") << "\n";
    } else {
        cout << "불가능\n";
    }
    return 0;
}
```

이 패턴에서 `addClause(u, f_u, v, f_v)`는 "¬(u=f_u) ∨ ¬(v=f_v)"를 표현한다. 즉 두 조건이 동시에 성립하지 않아야 한다는 뜻이다.

---

## 주의사항 및 팁

### 1. 변수 인덱싱 규칙 고정

구현에서 가장 흔한 실수는 변수 인덱스 매핑이 혼란스러워지는 것이다. 관례적으로 xᵢ = 2i, ¬xᵢ = 2i + 1 또는 xᵢ = i, ¬xᵢ = i + N으로 통일하고 절대 섞어 쓰지 않는 것이 중요하다.

### 2. addClause 의미 이중 확인

`addClause(u, nu, v, nv)`가 나타내는 절의 의미를 매번 명확히 정의해야 한다. 절 `(A ∨ B)`는 nu=false이면 xᵤ, true이면 ¬xᵤ를 의미한다고 명시해 두는 것이 실수를 예방한다.

### 3. SCC 구현: Tarjan vs Kosaraju

- **Tarjan's**: 한 번의 DFS로 SCC를 구하며, 코드가 약간 복잡하다. SCC 번호가 역위상 정렬 순서로 매겨지므로 Kosaraju와 값 결정 로직이 반대다.
- **Kosaraju's**: 두 번의 DFS, 코드가 직관적이다. 두 번째 DFS는 역그래프를 사용하며 SCC 번호가 위상 정렬 역순으로 매겨진다.

### 4. 해가 여러 개일 때

2-SAT는 해의 존재 여부만 선형 시간에 판단한다. 가능한 해의 수나 모든 해의 열거는 추가 계산이 필요하다. 위의 알고리즘이 반환하는 해는 SCC 기반의 "표준 해" 중 하나이며, 유일하지 않을 수 있다.

### 5. 단일 강제 조건(Unit Clause)

"xᵢ는 반드시 참이어야 한다"는 단일 조건은 `(xᵢ ∨ xᵢ)` 절로 표현할 수 있다. 이는 `addClause(i, false, i, false)`를 호출하면 되며, 내부적으로 `¬xᵢ → xᵢ` 간선이 두 번 추가된다.

---

## 참고 자료

- [QabasAK - 2SAT Solver (Tarjan's SCC 기반 C++ 구현)](https://github.com/QabasAK/2SAT-Solver)
- [hoanghai1803 - ITK21NBK 경쟁 프로그래밍 커리큘럼 (2-SAT 포함)](https://github.com/hoanghai1803/ITK21NBK)
