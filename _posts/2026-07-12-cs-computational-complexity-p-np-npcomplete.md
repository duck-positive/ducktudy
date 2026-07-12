---
layout: post
title: "계산 복잡도 이론 완전 정복: P, NP, NP-완전 문제의 원리와 실전 적용"
date: 2026-07-12
categories: [cs, computer-science]
tags: [complexity-theory, P-NP, NP-complete, algorithms, computational-theory, SAT, TSP]
---

## 계산 복잡도 이론이란?

우리가 알고리즘을 설계할 때 항상 따라오는 질문이 있습니다. "이 문제를 빠르게 풀 수 있는 방법이 존재하는가?" 계산 복잡도 이론(Computational Complexity Theory)은 이 질문에 체계적으로 답하기 위한 이론적 토대입니다.

복잡도 이론은 문제를 "얼마나 빠르게 풀 수 있는가"에 따라 분류합니다. 핵심 개념은 다음 세 가지입니다.

- **시간 복잡도(Time Complexity):** 입력 크기 n에 대해 알고리즘이 수행하는 연산의 수
- **공간 복잡도(Space Complexity):** 알고리즘이 사용하는 메모리의 양
- **복잡도 클래스(Complexity Class):** 비슷한 복잡도를 가진 문제들의 집합

오늘은 가장 중요한 복잡도 클래스인 **P, NP, NP-완전(NP-Complete), NP-하드(NP-Hard)**를 깊이 파헤칩니다.

---

## P 클래스: 효율적으로 풀 수 있는 문제

**P(Polynomial time)**는 결정론적 튜링 머신에서 **다항 시간(Polynomial time)** 내에 풀 수 있는 문제들의 집합입니다. 쉽게 말하면, 입력 크기 n에 대해 O(n^k) 형태의 시간이 걸리는 알고리즘이 존재하는 문제들입니다.

P에 속하는 대표적인 문제들:
- 정렬(Sorting): O(n log n)
- 최단 경로(Dijkstra): O((V + E) log V)
- 행렬 곱셈: O(n^3) 또는 Strassen O(n^2.81)
- 소수 판별(AKS): O(log^6 n)

---

## NP 클래스: 검증은 빠르지만 풀기는 어려울 수 있는 문제

**NP(Nondeterministic Polynomial time)**는 "주어진 해답이 올바른지 다항 시간 내에 검증(verify)할 수 있는 문제"들의 집합입니다.

중요한 오해: NP는 "빠르게 풀 수 없는 문제"가 아닙니다. P⊆NP이기 때문에, P에 속하는 모든 문제는 당연히 NP에도 속합니다. 핵심 질문은 "P=NP인가 P≠NP인가"입니다.

### P vs NP 문제가 중요한 이유

만약 P=NP라면:
- 현재 사용하는 모든 공개키 암호 시스템(RSA, ECC 등)이 깨집니다
- 단백질 접힘 예측, 신약 개발이 극적으로 빨라집니다
- 최적화 문제 대부분이 효율적으로 풀립니다

현재 대부분의 컴퓨터 과학자들은 P≠NP라고 믿지만, 수학적 증명은 아직 나오지 않았습니다. 이 문제는 클레이 수학 연구소(Clay Mathematics Institute)의 밀레니엄 문제 중 하나로, 100만 달러의 상금이 걸려 있습니다.

---

## NP-완전(NP-Complete): NP 중 가장 어려운 문제들

**NP-완전 문제**는 두 가지 조건을 동시에 만족하는 문제입니다.

1. **NP에 속한다**: 해답을 다항 시간에 검증할 수 있다
2. **NP-하드이다**: NP의 모든 문제를 이 문제로 다항 시간 내에 환원(reduce)할 수 있다

만약 NP-완전 문제 중 하나라도 다항 시간 알고리즘을 찾는다면, 모든 NP 문제가 다항 시간에 풀리게 됩니다(P=NP 증명).

### 대표적인 NP-완전 문제들

| 문제 | 설명 |
|------|------|
| SAT (Boolean Satisfiability) | 주어진 불리언 수식을 참으로 만드는 변수 값 할당이 존재하는가 |
| 3-SAT | 각 절이 정확히 3개의 리터럴을 갖는 CNF 형식의 SAT |
| 클리크(Clique) | 그래프에서 크기 k인 완전 부분 그래프가 존재하는가 |
| 정점 커버(Vertex Cover) | 크기 k 이하의 정점 집합으로 모든 간선을 커버할 수 있는가 |
| 해밀턴 경로(Hamiltonian Path) | 모든 정점을 정확히 한 번씩 방문하는 경로가 존재하는가 |
| 배낭 문제(0/1 Knapsack) | 용량 W를 초과하지 않으면서 최대 가치를 갖도록 물건을 선택할 수 있는가 |
| 부분집합 합(Subset Sum) | 집합에서 합이 정확히 T인 부분집합이 존재하는가 |
| 여행하는 세일즈맨(TSP) 결정 버전 | 비용 k 이하로 모든 도시를 방문하는 순환 경로가 존재하는가 |

### Cook-Levin 정리: 최초의 NP-완전 문제

1971년 스티븐 쿡(Stephen Cook)은 **SAT 문제**가 NP-완전임을 증명했습니다. 이것이 인류가 발견한 최초의 NP-완전 문제입니다. 이후 카프(Richard Karp)는 1972년 논문에서 21개의 고전적인 NP-완전 문제를 열거했습니다(카프의 21 문제).

---

## 왜 복잡도 이론이 중요한가?

### 1. 불가능한 최적해 탐색을 멈출 수 있다

어떤 문제가 NP-완전임을 안다면, 다항 시간 알고리즘을 찾으려는 시도를 멈추고 **근사 알고리즘(Approximation Algorithm)** 또는 **휴리스틱(Heuristic)**을 사용하는 방향으로 전환할 수 있습니다.

### 2. 암호화의 이론적 토대

RSA, Elliptic Curve 암호화는 **큰 수의 인수 분해**와 **이산 로그 문제**가 NP에 속한다는 사실(더 정확히는 NP-hard로 여겨진다는 사실)에 기반합니다. P=NP가 증명되면 이 암호 시스템들은 즉시 무너집니다.

### 3. 알고리즘 설계 방향 결정

NP-완전 문제를 실제로 다루는 세 가지 전략:
- **근사 알고리즘**: 최적해의 일정 비율 이내의 해를 보장
- **파라미터화 알고리즘**: 문제 파라미터가 작을 때 빠른 알고리즘 사용
- **랜덤화 알고리즘**: 확률적으로 좋은 해를 찾음

---

## 실제 구현 예제

### 예제 1: 부분집합 합 문제 (Subset Sum) — 브루트 포스와 동적 프로그래밍 비교

```python
from itertools import combinations
import time

def subset_sum_brute_force(numbers: list[int], target: int) -> list[int] | None:
    """
    브루트 포스: 모든 부분집합 탐색 - O(2^n)
    NP-완전 문제의 지수 시간 특성을 보여줌
    """
    n = len(numbers)
    for r in range(1, n + 1):
        for combo in combinations(range(n), r):
            if sum(numbers[i] for i in combo) == target:
                return [numbers[i] for i in combo]
    return None

def subset_sum_dp(numbers: list[int], target: int) -> list[int] | None:
    """
    동적 프로그래밍: O(n * target) — 의사 다항 시간(Pseudo-polynomial)
    입력 크기가 아닌 입력 값에 의존하므로 진정한 다항 시간이 아님
    """
    n = len(numbers)
    # dp[i][j] = True: numbers[:i]에서 합이 j인 부분집합 존재
    dp = [[False] * (target + 1) for _ in range(n + 1)]
    dp[0][0] = True

    for i in range(1, n + 1):
        num = numbers[i - 1]
        for j in range(target + 1):
            dp[i][j] = dp[i - 1][j]
            if j >= num:
                dp[i][j] |= dp[i - 1][j - num]

    if not dp[n][target]:
        return None

    # 역추적으로 실제 부분집합 복원
    result = []
    j = target
    for i in range(n, 0, -1):
        if not dp[i - 1][j]:
            result.append(numbers[i - 1])
            j -= numbers[i - 1]
    return result

# 성능 비교
import random
random.seed(42)

numbers = random.sample(range(1, 50), 20)
target = sum(numbers[:5])  # 부분집합이 반드시 존재하도록 설정

print(f"배열: {numbers}")
print(f"목표 합: {target}")

# 브루트 포스 (n=20에서 이미 느려짐)
start = time.perf_counter()
result_bf = subset_sum_brute_force(numbers, target)
elapsed_bf = time.perf_counter() - start
print(f"\n브루트 포스 결과: {result_bf}")
print(f"브루트 포스 시간: {elapsed_bf:.4f}초 (최대 2^20 = {2**20:,}번 탐색)")

# 동적 프로그래밍
start = time.perf_counter()
result_dp = subset_sum_dp(numbers, target)
elapsed_dp = time.perf_counter() - start
print(f"\nDP 결과: {result_dp}")
print(f"DP 시간: {elapsed_dp:.6f}초 (O(n * target) = O({len(numbers)} * {target}))")

print(f"\n속도 향상: {elapsed_bf / elapsed_dp:.1f}배")
```

출력 예시:
```
브루트 포스 시간: 0.3421초 (최대 2^20 = 1,048,576번 탐색)
DP 시간: 0.000812초 (O(n * target) = O(20 * 87))
속도 향상: 421.3배
```

주목할 점은 DP가 훨씬 빠르지만, 이것은 **의사 다항 시간(Pseudo-polynomial)**입니다. target 값이 매우 크면 DP도 느려집니다. 이것이 Subset Sum이 NP-완전인 이유입니다.

---

### 예제 2: 여행하는 세일즈맨 (TSP) — 정확한 해 vs 근사 알고리즘

```python
from itertools import permutations
import math

def tsp_exact(dist: list[list[int]]) -> tuple[int, list[int]]:
    """
    TSP 정확 해: O(n!) — n=20이면 2.4 * 10^18번 연산
    실용적으로는 n=10 이상부터 사용 불가
    """
    n = len(dist)
    cities = list(range(1, n))
    best_cost = float('inf')
    best_path = None

    for perm in permutations(cities):
        path = [0] + list(perm) + [0]
        cost = sum(dist[path[i]][path[i+1]] for i in range(n))
        if cost < best_cost:
            best_cost = cost
            best_path = path

    return best_cost, best_path

def tsp_nearest_neighbor(dist: list[list[int]], start: int = 0) -> tuple[int, list[int]]:
    """
    최근접 이웃 휴리스틱: O(n^2)
    최적해 보장 없음, 최악의 경우 최적의 log(n)배 이상 가능
    실제로는 최적해의 약 20-25% 이내를 달성
    """
    n = len(dist)
    unvisited = set(range(n))
    path = [start]
    unvisited.remove(start)
    current = start

    while unvisited:
        nearest = min(unvisited, key=lambda c: dist[current][c])
        path.append(nearest)
        unvisited.remove(nearest)
        current = nearest

    path.append(start)
    cost = sum(dist[path[i]][path[i+1]] for i in range(n))
    return cost, path

def tsp_2opt(dist: list[list[int]], path: list[int]) -> tuple[int, list[int]]:
    """
    2-opt 개선: 두 간선을 교환하여 경로를 반복적으로 개선
    국소 최적해(Local optimum)에 수렴, O(n^2) per iteration
    """
    n = len(path) - 1  # 마지막 원소는 시작점 반복

    def total_cost(p):
        return sum(dist[p[i]][p[i+1]] for i in range(len(p) - 1))

    improved = True
    current_path = path[:]

    while improved:
        improved = False
        for i in range(1, n - 1):
            for j in range(i + 1, n):
                # i와 j 사이 구간을 뒤집기
                new_path = current_path[:i] + current_path[i:j+1][::-1] + current_path[j+1:]
                if total_cost(new_path) < total_cost(current_path):
                    current_path = new_path
                    improved = True
                    break
            if improved:
                break

    return total_cost(current_path), current_path

# 10개 도시 랜덤 좌표 생성
import random
random.seed(7)
n = 10
cities = [(random.randint(0, 100), random.randint(0, 100)) for _ in range(n)]

def euclidean(a, b):
    return int(math.sqrt((a[0]-b[0])**2 + (a[1]-b[1])**2))

dist = [[euclidean(cities[i], cities[j]) for j in range(n)] for i in range(n)]

# 정확 해 (n=10이라 실용적)
exact_cost, exact_path = tsp_exact(dist)
print(f"정확 해 비용: {exact_cost}")
print(f"정확 해 경로: {exact_path}")

# 최근접 이웃
nn_cost, nn_path = tsp_nearest_neighbor(dist)
print(f"\n최근접 이웃 비용: {nn_cost} (최적 대비 {nn_cost/exact_cost*100:.1f}%)")

# 2-opt 개선
opt_cost, opt_path = tsp_2opt(dist, nn_path)
print(f"2-opt 개선 비용: {opt_cost} (최적 대비 {opt_cost/exact_cost*100:.1f}%)")
```

이 예제는 NP-완전 문제를 다루는 실전 전략을 보여줍니다. 정확한 해는 지수 시간이 걸리지만, 근사 알고리즘과 국소 탐색을 조합하면 실용적인 시간 내에 충분히 좋은 해를 얻을 수 있습니다.

---

## NP-하드(NP-Hard): NP보다 더 어려울 수 있는 문제

**NP-하드**는 "NP의 모든 문제를 이 문제로 환원할 수 있는" 문제들입니다. NP-하드는 NP에 속할 필요가 없습니다. 즉, 해답을 검증하는 것조차 어려울 수 있습니다.

```
          ┌──────────────────────────────────┐
          │           NP-Hard                │
          │    ┌─────────────────────┐       │
          │    │   NP-Complete       │       │
          │    │ ┌─────────────────┐ │       │
          │    │ │       NP        │ │       │
          │    │ │  ┌───────────┐  │ │       │
          │    │ │  │     P     │  │ │       │
          │    │ │  └───────────┘  │ │       │
          │    │ └─────────────────┘ │       │
          │    └─────────────────────┘       │
          └──────────────────────────────────┘
          (P≠NP를 가정할 때)
```

TSP의 최적화 버전(최소 비용 경로를 찾아라)은 NP-하드이지만, 결정 버전(비용 k 이하의 경로가 존재하는가)은 NP-완전입니다.

---

## 복잡도 이론의 다른 중요한 클래스들

| 클래스 | 설명 | 예시 |
|--------|------|------|
| **co-NP** | NP의 여집합; "아니오" 답의 검증이 쉬운 문제 | PRIMES (소수인지 판별, P에도 속함) |
| **PSPACE** | 다항 공간으로 풀 수 있는 문제 | 양화 불리언 공식(QBF) |
| **EXP** | 지수 시간으로 풀 수 있는 문제 | 체스 완벽 게임 |
| **BPP** | 확률적 다항 시간 알고리즘이 존재하는 문제 | Miller-Rabin 소수 판별 |

---

## 주의사항 및 실전 팁

### 팁 1: NP-완전 문제를 만났을 때의 전략

```
NP-완전 문제 발견
       │
       ├─→ 입력이 작은가? (n < 30)
       │         YES → 지수 시간 브루트 포스 사용
       │         NO  ↓
       ├─→ 특수 구조가 있는가? (트리, 평면 그래프 등)
       │         YES → 다항 시간 특수 알고리즘 탐색
       │         NO  ↓
       ├─→ 근사해로 충분한가?
       │         YES → 근사 알고리즘 (PTAS, FPTAS 탐색)
       │         NO  ↓
       └─→ 실용적 해법: 메타휴리스틱 (SA, GA, TS 등)
```

### 팁 2: 환원(Reduction)으로 NP-완전성 증명

새로운 문제 Y가 NP-완전인지 증명하려면:
1. Y가 NP에 속함을 보인다 (다항 시간 검증기 제시)
2. 이미 알려진 NP-완전 문제 X를 Y로 환원한다 (X ≤_p Y)

이 두 조건이 만족되면 Y는 NP-완전입니다.

### 팁 3: 파라미터화 복잡도 (Parameterized Complexity)

NP-완전 문제라도 특정 파라미터 k가 작으면 빠르게 풀 수 있습니다. FPT(Fixed-Parameter Tractable) 알고리즘은 f(k) · n^O(1) 시간에 실행됩니다.

예를 들어, 정점 커버 문제는 일반적으로 NP-완전이지만, k-정점 커버(크기 k 이하의 정점 커버 존재 여부)는 O(2^k · n) FPT 알고리즘이 존재합니다.

---

## 마무리

계산 복잡도 이론은 "이 문제는 왜 빠르게 풀기 어려운가?"를 이해하는 열쇠입니다. P=NP 문제는 수십 년째 미해결로 남아 있지만, 이 이론 덕분에 우리는 불가능한 효율성을 추구하는 대신 근사 알고리즘, 휴리스틱, 파라미터화 알고리즘 등 실용적인 해법을 선택할 수 있게 되었습니다. 알고리즘을 설계할 때 복잡도 클래스를 인식하는 것이 곧 엔지니어링의 출발점입니다.

## 참고 자료

- [P versus NP problem - Wikipedia](https://en.wikipedia.org/wiki/P_versus_NP_problem)
- [P vs NP - Britannica](https://www.britannica.com/science/P-versus-NP-problem)
- [The P versus NP Problem - Clay Mathematics Institute (Stephen Cook)](https://www.claymath.org/wp-content/uploads/2022/06/pvsnp.pdf)
- [NP-completeness - Wikipedia](https://en.wikipedia.org/wiki/NP-completeness)
