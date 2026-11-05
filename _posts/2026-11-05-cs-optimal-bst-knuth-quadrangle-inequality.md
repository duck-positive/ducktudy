---
layout: post
title: "최적 이진 탐색 트리(Optimal BST)와 Knuth의 DP 최적화: 사각형 부등식으로 O(N³)을 O(N²)으로"
date: 2026-11-05
categories: [cs, computer-science]
tags: [optimal-bst, dynamic-programming, knuth-optimization, quadrangle-inequality, dp, algorithm]
---

이진 탐색 트리(BST)에서 "균형"은 최악의 탐색 비용을 O(log N)으로 보장한다. 그러나 모든 키가 동일한 빈도로 접근되지 않는다면 어떨까? 자주 접근하는 키를 루트 근처에, 드물게 접근하는 키를 잎 근처에 배치하면 평균 탐색 비용을 더 줄일 수 있다. 이것이 **최적 이진 탐색 트리(Optimal Binary Search Tree, Optimal BST)**의 핵심 아이디어다.

1971년 Donald Knuth는 이 문제를 O(N²) 시간에 해결하는 동적 프로그래밍 알고리즘을 발표했다. 순진한 DP는 O(N³)이지만, **단조성 조건(사각형 부등식, Quadrangle Inequality)**을 이용해 최적 루트 선택 범위를 좁히면 O(N²)으로 감소한다. 이 최적화 기법은 Optimal BST 외에도 행렬 체인 곱셈, 볼록 껍질 트릭 등 다양한 DP 문제에 적용된다.

## Optimal BST란 무엇인가

### 문제 정의

N개의 키 k₁ < k₂ < ... < kₙ와 각 키에 대한 **성공 탐색 확률** pᵢ (kᵢ를 탐색할 확률), 그리고 각 키 사이 범위에 대한 **실패 탐색 확률** qᵢ (kᵢ와 kᵢ₊₁ 사이를 탐색하다 실패할 확률)가 주어진다.

```
q₀: k₁보다 작은 키 탐색 실패 확률
p₁, q₁, p₂, q₂, ..., pₙ, qₙ: 각 키와 간격의 확률
```

BST에서 kᵢ를 탐색하는 비용은 `depth(kᵢ) + 1`이다. 평균 탐색 비용을 최소화하는 BST 구조를 찾는 것이 목표다.

**기대 탐색 비용** =  Σᵢ (depth(kᵢ) + 1) × pᵢ + Σᵢ (depth(dᵢ) + 1) × qᵢ

여기서 dᵢ는 실패 탐색에 대응하는 더미 노드다.

### 직관적 예시

4개의 키 `{do, if, read, while}`이 있고 접근 확률이 다를 때:

| 키 | 접근 확률 |
|----|----------|
| do | 0.15 |
| if | 0.10 |
| read | 0.05 |
| while | 0.35 |

단순 균형 BST가 아닌, `while`을 루트 근처에 두는 Optimal BST가 평균 탐색 비용이 더 낮다.

## 왜 이 문제가 중요한가

### 컴파일러의 심볼 테이블

프로그래밍 언어의 키워드(`if`, `while`, `for` 등)는 소스 코드에서 출현 빈도가 매우 다르다. 컴파일러가 키워드를 정적 BST로 관리할 때 Optimal BST를 사용하면 토큰화 속도를 개선할 수 있다.

### 허프만 코딩과의 비교

허프만 코딩이 "최적 접두사 없는 코드"를 생성하듯, Optimal BST는 "최적 탐색 트리"를 생성한다. 그러나 허프만은 Greedy O(N log N)으로 해결되는 반면, Optimal BST는 DP O(N²)이 필요하다. BST는 in-order 순서 제약이 있어 더 복잡하다.

### Knuth 최적화의 범용성

사각형 부등식을 만족하는 DP 점화식은 모두 Knuth 최적화를 적용할 수 있다. 행렬 체인 곱셈, 구간 DP(Interval DP), 1D/1D DP 등 광범위한 문제에서 O(N³) → O(N²) 개선이 가능하다.

## 핵심 알고리즘

### 순진한 DP (O(N³))

```
e[i][j] = i부터 j까지 키를 포함하는 최적 BST의 기대 탐색 비용
w[i][j] = p[i] + p[i+1] + ... + p[j] + q[i-1] + q[i] + ... + q[j]
        = 서브트리 i..j의 총 확률 가중치 (루트를 한 단계 내릴 때 추가되는 비용)

점화식:
e[i][j] = min over r in [i,j] { e[i][r-1] + e[r+1][j] + w[i][j] }
e[i][i-1] = q[i-1]  (빈 서브트리의 비용)
```

r을 모든 경우에 대해 탐색하면 O(N) 루프가 N² 개의 서브 문제에 적용되어 총 O(N³)이다.

### Knuth의 단조성 최적화 (O(N²))

Knuth는 최적 루트가 단조성을 가짐을 증명했다:

```
root[i][j-1] ≤ root[i][j] ≤ root[i+1][j]
```

즉, 구간이 늘어날수록 최적 루트도 단조 증가한다. 이 성질 덕분에 각 서브 문제에서 탐색해야 하는 r의 범위가 O(1) 평균으로 줄어들고, 전체 복잡도가 O(N²)이 된다.

이 단조성은 w[i][j]가 **사각형 부등식(Quadrangle Inequality)**을 만족하기 때문에 성립한다:

```
w[a][c] + w[b][d] ≤ w[a][d] + w[b][c]  (a ≤ b ≤ c ≤ d)
```

## 실제 구현 예제

### 예제 1: Optimal BST DP 완전 구현 (Python)

```python
from typing import List, Tuple
import math

def optimal_bst(
    p: List[float],  # p[1..n]: 키 k_i의 성공 탐색 확률
    q: List[float],  # q[0..n]: 실패 탐색 확률
) -> Tuple[float, List[List[int]]]:
    """
    Optimal BST 알고리즘 (Knuth 1971, CLRS 15.5).
    p: 길이 n+1 (p[0]은 사용하지 않음, p[1..n] 사용)
    q: 길이 n+1 (q[0..n])
    반환: (최소 기대 비용, root 테이블)
    """
    n = len(p) - 1  # 키의 수

    # e[i][j]: 키 i부터 j까지의 최적 서브트리 기대 비용
    # w[i][j]: 서브트리 i..j의 총 가중치
    # root[i][j]: 최적 서브트리의 루트 인덱스
    INF = float('inf')
    e = [[INF] * (n + 2) for _ in range(n + 2)]
    w = [[0.0] * (n + 2) for _ in range(n + 2)]
    root = [[0] * (n + 1) for _ in range(n + 2)]

    # 기저 사례: 빈 서브트리
    for i in range(1, n + 2):
        e[i][i - 1] = q[i - 1]
        w[i][i - 1] = q[i - 1]

    # 길이 l의 서브트리부터 bottom-up으로 채움
    for l in range(1, n + 1):       # 서브트리 크기
        for i in range(1, n - l + 2):  # 시작 인덱스
            j = i + l - 1               # 끝 인덱스
            e[i][j] = INF
            w[i][j] = w[i][j - 1] + p[j] + q[j]

            # Knuth 최적화 적용:
            # r 탐색 범위를 root[i][j-1] ~ root[i+1][j]로 제한
            lo = root[i][j - 1] if j > 1 else i
            hi = root[i + 1][j] if i <= n else j

            # lo, hi 경계 보정
            lo = max(lo, i)
            hi = min(hi, j)

            for r in range(lo, hi + 1):
                cost = e[i][r - 1] + e[r + 1][j] + w[i][j]
                if cost < e[i][j]:
                    e[i][j] = cost
                    root[i][j] = r

    return e[1][n], root


def print_optimal_bst(root: List[List[int]], keys: List[str], i: int, j: int, parent: str, side: str):
    """재귀적으로 Optimal BST 구조를 출력한다."""
    if i > j:
        print(f"  d{i-1} (실패 노드) {'→' if side else ''} {parent}의 {side}")
        return
    r = root[i][j]
    if parent == "":
        print(f"  {keys[r]} (루트)")
    else:
        print(f"  {keys[r]} → {parent}의 {side}")
    print_optimal_bst(root, keys, i, r - 1, keys[r], "왼쪽")
    print_optimal_bst(root, keys, r + 1, j, keys[r], "오른쪽")


# ─────────────────────────────────────────────────────
#  예시: 5개의 키
#  키: k1=10, k2=20, k3=30, k4=40, k5=50
#  p  = [_, 0.15, 0.10, 0.05, 0.10, 0.20]
#  q  = [0.05, 0.10, 0.05, 0.05, 0.05, 0.10]
# ─────────────────────────────────────────────────────
keys = ['', 'k1(10)', 'k2(20)', 'k3(30)', 'k4(40)', 'k5(50)']
p = [0, 0.15, 0.10, 0.05, 0.10, 0.20]
q = [0.05, 0.10, 0.05, 0.05, 0.05, 0.10]

print(f"총 확률 합: {sum(p) + sum(q):.2f}")  # 반드시 1.0

cost, root_table = optimal_bst(p, q)
print(f"\n최소 기대 탐색 비용: {cost:.4f}")
print("\n=== Optimal BST 구조 ===")
print_optimal_bst(root_table, keys, 1, 5, "", "")

# ─────────────────────────────────────────────────────
#  비교: 균형 BST와 Optimal BST의 기대 비용 비교
# ─────────────────────────────────────────────────────
def balanced_bst_cost(p: List[float], q: List[float]) -> float:
    """완전 균형 BST의 기대 탐색 비용 근사 계산"""
    n = len(p) - 1
    # 단순화: 각 키의 depth = floor(log2(i))로 근사
    import math
    cost = 0.0
    for i in range(1, n + 1):
        depth = math.floor(math.log2(i)) + 1  # 1-indexed depth
        cost += (depth) * p[i]
    for i in range(0, n + 1):
        depth = math.floor(math.log2(i + 1)) + 2 if i > 0 else 2
        cost += depth * q[i]
    return cost

balanced_cost = balanced_bst_cost(p, q)
print(f"\n균형 BST 기대 비용 (근사): {balanced_cost:.4f}")
print(f"Optimal BST 기대 비용:      {cost:.4f}")
print(f"개선율: {(balanced_cost - cost) / balanced_cost * 100:.1f}%")
```

### 예제 2: Knuth 최적화 효과 측정 (C++)

```cpp
#include <iostream>
#include <vector>
#include <climits>
#include <chrono>
#include <random>
#include <iomanip>

using namespace std;

// O(N³) 순진한 DP
double naive_optimal_bst(const vector<double>& p, const vector<double>& q, int n) {
    vector<vector<double>> e(n + 2, vector<double>(n + 1, 1e18));
    vector<vector<double>> w(n + 2, vector<double>(n + 1, 0.0));

    for (int i = 1; i <= n + 1; ++i) {
        e[i][i - 1] = q[i - 1];
        w[i][i - 1] = q[i - 1];
    }

    for (int l = 1; l <= n; ++l) {
        for (int i = 1; i <= n - l + 1; ++i) {
            int j = i + l - 1;
            w[i][j] = w[i][j - 1] + p[j] + q[j];
            for (int r = i; r <= j; ++r) {  // O(N) 탐색
                double cost = e[i][r - 1] + e[r + 1][j] + w[i][j];
                if (cost < e[i][j]) e[i][j] = cost;
            }
        }
    }
    return e[1][n];
}

// O(N²) Knuth 최적화 DP
double knuth_optimal_bst(const vector<double>& p, const vector<double>& q, int n) {
    vector<vector<double>> e(n + 2, vector<double>(n + 1, 1e18));
    vector<vector<double>> w(n + 2, vector<double>(n + 1, 0.0));
    vector<vector<int>> root(n + 2, vector<int>(n + 1, 0));

    for (int i = 1; i <= n + 1; ++i) {
        e[i][i - 1] = q[i - 1];
        w[i][i - 1] = q[i - 1];
        root[i][i - 1] = i - 1;  // 더미 노드
    }

    for (int l = 1; l <= n; ++l) {
        for (int i = 1; i <= n - l + 1; ++i) {
            int j = i + l - 1;
            w[i][j] = w[i][j - 1] + p[j] + q[j];

            // Knuth 최적화: 탐색 범위를 단조성으로 제한
            int lo = (j > 1) ? root[i][j - 1] : i;
            int hi = (i <= n) ? root[i + 1][j] : j;
            lo = max(lo, i);
            hi = min(hi, j);

            for (int r = lo; r <= hi; ++r) {
                double cost = e[i][r - 1] + e[r + 1][j] + w[i][j];
                if (cost < e[i][j]) {
                    e[i][j] = cost;
                    root[i][j] = r;
                }
            }
        }
    }
    return e[1][n];
}

int main() {
    mt19937 rng(42);
    uniform_real_distribution<double> dist(0.0, 1.0);

    cout << fixed << setprecision(4);
    cout << "N\t순진한DP(ms)\tKnuth최적화(ms)\t속도향상\n";
    cout << string(60, '-') << "\n";

    for (int n : {100, 300, 500, 1000, 2000}) {
        // 무작위 확률 생성 (합이 1이 되도록 정규화)
        vector<double> p(n + 1), q(n + 1);
        double total = 0;
        for (int i = 1; i <= n; ++i) { p[i] = dist(rng); total += p[i]; }
        for (int i = 0; i <= n; ++i) { q[i] = dist(rng); total += q[i]; }
        for (int i = 1; i <= n; ++i) p[i] /= total;
        for (int i = 0; i <= n; ++i) q[i] /= total;

        // 순진한 DP (N이 너무 크면 스킵)
        double naive_ms = -1;
        if (n <= 500) {
            auto t1 = chrono::high_resolution_clock::now();
            naive_optimal_bst(p, q, n);
            auto t2 = chrono::high_resolution_clock::now();
            naive_ms = chrono::duration<double, milli>(t2 - t1).count();
        }

        // Knuth 최적화
        auto t1 = chrono::high_resolution_clock::now();
        knuth_optimal_bst(p, q, n);
        auto t2 = chrono::high_resolution_clock::now();
        double knuth_ms = chrono::duration<double, milli>(t2 - t1).count();

        cout << n;
        if (naive_ms >= 0) {
            cout << "\t" << naive_ms << "\t\t" << knuth_ms
                 << "\t\t" << (naive_ms / knuth_ms) << "x";
        } else {
            cout << "\t(생략)\t\t\t" << knuth_ms;
        }
        cout << "\n";
    }
    return 0;
}
```

컴파일 및 실행: `g++ -O2 -o obst optimal_bst.cpp && ./obst`

예상 출력:

```
N    순진한DP(ms)   Knuth최적화(ms)   속도향상
------------------------------------------------------------
100  0.8            0.1               8x
300  22.1           1.0               22x
500  102.3          2.8               37x
1000 (생략)         11.2
2000 (생략)         44.6
```

N이 커질수록 Knuth 최적화의 효과가 뚜렷해진다.

## 사각형 부등식(Quadrangle Inequality) 심화

Knuth 최적화가 적용 가능한 조건은 비용 함수 w[i][j]가 **사각형 부등식(Quadrangle Inequality, QI)**을 만족하는 것이다:

```
a ≤ b ≤ c ≤ d 이면: w[a][c] + w[b][d] ≤ w[b][c] + w[a][d]
```

이 조건이 성립하면 Optimal BST처럼 최적 루트(또는 분할 점)가 단조성을 가진다:

```
root[i][j-1] ≤ root[i][j] ≤ root[i+1][j]
```

**다른 적용 예시:**

| 문제 | 비용 함수 | 복잡도 개선 |
|------|----------|------------|
| Optimal BST | w[i][j] = 구간 확률 합 | O(N³) → O(N²) |
| 행렬 체인 곱셈 | w[i][j] = 차원 곱 | O(N³) → O(N²) |
| 볼록 다각형 삼각분할 | w[i][j] = 삼각형 비용 | O(N³) → O(N²) |
| 1D Facility Location | w[i][j] = 구간 클라이언트 비용 | O(N³) → O(N²) |

## 주의사항과 실전 팁

### Knuth 최적화의 경계 조건

`root[i][j-1]`과 `root[i+1][j]`의 초기값 설정에 주의해야 한다. `j = i` (길이 1) 경우 `root[i][i-1]`은 존재하지 않으므로 하한을 `i`로, `root[i+1][i]`는 `i+1-1 = i`이므로 상한도 `i`로 맞춘다.

### 동적 BST vs 정적 BST

Optimal BST는 **정적(static)** 사용 패턴을 가정한다. 접근 빈도가 시간에 따라 변하면 Splay Tree처럼 동적으로 자기 조정하는 BST가 더 적합하다. 오랜 연구에서 Splay Tree의 "Working Set Theorem"이 Optimal BST와 경쟁적임이 증명되었다.

### 공간 최적화

e 테이블은 O(N²) 공간이지만, w 테이블은 1D 배열로 계산하면서 갱신하면 메모리를 절약할 수 있다. 단, root 테이블은 BST 구조 복원을 위해 반드시 O(N²)을 유지해야 한다.

### 실제 소프트웨어 적용

현실에서는 접근 빈도를 사전에 정확히 알기 어렵다. 실무에서는 프로파일링으로 접근 빈도를 측정 후 주기적으로 BST를 재구성하거나, Splay Tree처럼 적응형 자료구조를 사용한다. 컴파일러에서는 키워드 집합이 고정되어 있어 Optimal BST를 미리 계산하여 하드코딩하기도 한다.

## 참고 자료
- [Optimal Binary Search Tree - Wikipedia](https://en.wikipedia.org/wiki/Optimal_binary_search_tree)
- [Optimal Binary Search Trees - UC Irvine Lecture Notes](https://www.ics.uci.edu/~dan/class/165/notes/OptBST.html)
- [Optimal Search Tree - TryAlgo](https://tryalgo.org/en/trees/2022/11/03/optimal-search-tree/)
- [Introduction to Algorithms (CLRS) - Chapter 15.5: Optimal Binary Search Trees](https://mitpress.mit.edu/9780262046305/introduction-to-algorithms/)
