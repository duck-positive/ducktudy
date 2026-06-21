---
layout: post
title: "동적 프로그래밍 심화: 메모이제이션·테이블화·최적 부분구조 완전 정복"
date: 2026-06-21
categories: [cs, computer-science]
tags: [dynamic-programming, memoization, tabulation, algorithm, optimization, python, java]
---

동적 프로그래밍(Dynamic Programming, DP)은 1950년대 리처드 벨만(Richard Bellman)이 정립한 알고리즘 설계 패러다임으로, 복잡한 문제를 작은 하위 문제로 분해하고 그 결과를 재활용하여 전체 최적해를 구한다. "동적(Dynamic)"이라는 명칭은 사실 마케팅적 이유로 붙은 것으로, 기술적으로는 **메모이제이션(Memoization)** 또는 **테이블화(Tabulation)** 두 가지 구현 방식으로 대별된다.

이 아티클에서는 DP의 두 핵심 조건(최적 부분구조·겹치는 하위 문제), 탑다운과 바텀업 접근법의 차이, 그리고 클래식 문제(LCS, 배낭 문제, 편집 거리)의 실전 구현까지 완전히 파헤친다.

---

## DP의 두 핵심 조건

### 1. 최적 부분구조 (Optimal Substructure)

문제의 최적해가 하위 문제들의 최적해로 구성될 수 있어야 한다. 즉, 전체 최적 경로는 구간 내 최적 경로를 포함한다.

```
최단 경로: A → B → C → D 가 최단이라면
A → B → C 도 A에서 C까지의 최단 경로여야 한다
```

그리니디(Greedy)도 최적 부분구조를 요구하지만, DP는 추가로 하위 문제들이 서로 겹쳐야(Overlapping Subproblems) 한다.

### 2. 겹치는 하위 문제 (Overlapping Subproblems)

분할 정복(Divide and Conquer)과 달리, DP의 하위 문제들은 독립적이지 않고 여러 번 재계산된다. 피보나치 수열이 대표적이다.

```
fib(5) = fib(4) + fib(3)
fib(4) = fib(3) + fib(2)  ← fib(3) 중복!
fib(3) = fib(2) + fib(1)  ← fib(2) 중복!
...

재귀 호출 트리:
         fib(5)
        /      \
    fib(4)    fib(3)
    /    \    /    \
fib(3) fib(2) fib(2) fib(1)
...

겹치는 하위 문제 없으면 DP 불필요 (분할 정복으로 충분)
```

DP는 이미 계산한 하위 문제의 결과를 저장(메모이제이션)하거나 순서대로 테이블을 채워(테이블화) 중복 계산을 제거한다.

---

## 탑다운 vs 바텀업: 두 구현 방식 비교

### 탑다운 (Top-Down, 메모이제이션)

재귀로 큰 문제부터 시작해 필요할 때 하위 문제를 계산하고 캐시에 저장한다.

- **장점**: 구현이 직관적, 필요한 하위 문제만 계산 (Lazy Evaluation)
- **단점**: 재귀 호출 스택 오버플로 위험, 함수 호출 오버헤드

### 바텀업 (Bottom-Up, 테이블화)

가장 작은 하위 문제부터 순서대로 테이블을 채워 올라간다.

- **장점**: 스택 오버플로 없음, 캐시 효율 좋음 (순차 메모리 접근)
- **단점**: 모든 하위 문제를 계산 (불필요한 계산 포함 가능)

---

## 구현 예제 1: Python으로 세 가지 DP 클래식 문제 구현

### 피보나치: 메모이제이션 vs 테이블화

```python
import sys
from functools import lru_cache
from time import perf_counter

# 방법 1: 순수 재귀 (지수 시간, O(2^n))
def fib_naive(n: int) -> int:
    if n <= 1:
        return n
    return fib_naive(n - 1) + fib_naive(n - 2)

# 방법 2: 탑다운 메모이제이션 (O(n) 시간, O(n) 공간)
@lru_cache(maxsize=None)
def fib_memo(n: int) -> int:
    if n <= 1:
        return n
    return fib_memo(n - 1) + fib_memo(n - 2)

# 방법 3: 바텀업 테이블화 (O(n) 시간, O(n) 공간)
def fib_tab(n: int) -> int:
    if n <= 1:
        return n
    dp = [0] * (n + 1)
    dp[1] = 1
    for i in range(2, n + 1):
        dp[i] = dp[i-1] + dp[i-2]
    return dp[n]

# 방법 4: 공간 최적화 (O(n) 시간, O(1) 공간)
def fib_optimal(n: int) -> int:
    if n <= 1:
        return n
    a, b = 0, 1
    for _ in range(2, n + 1):
        a, b = b, a + b
    return b


# 성능 비교
n = 35
t0 = perf_counter(); r0 = fib_naive(n);   naive_ms = (perf_counter()-t0)*1000
t0 = perf_counter(); r1 = fib_memo(n);    memo_ms  = (perf_counter()-t0)*1000
t0 = perf_counter(); r2 = fib_tab(n);     tab_ms   = (perf_counter()-t0)*1000

print(f"n={n} 피보나치 결과: {r0}")
print(f"순수 재귀:    {naive_ms:.2f}ms")
print(f"메모이제이션: {memo_ms:.4f}ms  (약 {naive_ms/max(memo_ms,0.001):.0f}배 빠름)")
print(f"테이블화:     {tab_ms:.4f}ms")
```

### 최장 공통 부분 수열 (LCS, Longest Common Subsequence)

```python
def lcs(s1: str, s2: str) -> tuple[int, str]:
    """
    두 문자열의 LCS 길이와 실제 문자열을 반환한다.
    시간: O(m*n), 공간: O(m*n)
    """
    m, n = len(s1), len(s2)
    # dp[i][j] = s1[:i]와 s2[:j]의 LCS 길이
    dp = [[0] * (n + 1) for _ in range(m + 1)]

    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if s1[i-1] == s2[j-1]:
                dp[i][j] = dp[i-1][j-1] + 1
            else:
                dp[i][j] = max(dp[i-1][j], dp[i][j-1])

    # 역추적으로 실제 LCS 문자열 복원
    result = []
    i, j = m, n
    while i > 0 and j > 0:
        if s1[i-1] == s2[j-1]:
            result.append(s1[i-1])
            i -= 1; j -= 1
        elif dp[i-1][j] > dp[i][j-1]:
            i -= 1
        else:
            j -= 1

    return dp[m][n], ''.join(reversed(result))


# 편집 거리 (Levenshtein Distance)
def edit_distance(s1: str, s2: str) -> int:
    """
    s1을 s2로 바꾸는 최소 편집 연산(삽입, 삭제, 교체) 횟수
    시간: O(m*n), 공간: O(m*n) → O(n)으로 최적화 가능
    """
    m, n = len(s1), len(s2)
    dp = [[0] * (n + 1) for _ in range(m + 1)]

    for i in range(m + 1): dp[i][0] = i  # s2를 빈 문자열로 만들기 → 삭제 i번
    for j in range(n + 1): dp[0][j] = j  # 빈 문자열을 s2로 만들기 → 삽입 j번

    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if s1[i-1] == s2[j-1]:
                dp[i][j] = dp[i-1][j-1]           # 같으면 연산 불필요
            else:
                dp[i][j] = 1 + min(
                    dp[i-1][j],    # 삭제 (s1에서 s1[i-1] 제거)
                    dp[i][j-1],    # 삽입 (s2[j-1]을 s1에 추가)
                    dp[i-1][j-1],  # 교체 (s1[i-1]을 s2[j-1]로)
                )

    return dp[m][n]


# 실행 예시
s1, s2 = "ABCBDAB", "BDCAB"
length, seq = lcs(s1, s2)
print(f"LCS('{s1}', '{s2}'): 길이={length}, 문자열='{seq}'")
# 출력: LCS 길이=4, 문자열='BCAB' 또는 'BDAB'

print(f"편집거리('kitten', 'sitting'): {edit_distance('kitten', 'sitting')}")
# 출력: 3 (kitten → sitten → sittin → sitting)
```

---

## 구현 예제 2: Java로 0/1 배낭 문제와 공간 최적화

```java
import java.util.Arrays;

public class Knapsack {

    /**
     * 0/1 배낭 문제 - 바텀업 2D DP
     * 시간: O(n * W), 공간: O(n * W)
     * items[i] = {weight, value}
     */
    public static int knapsack2D(int[][] items, int capacity) {
        int n = items.length;
        // dp[i][w] = 처음 i개 아이템으로 용량 w 배낭을 채울 때의 최대 가치
        int[][] dp = new int[n + 1][capacity + 1];

        for (int i = 1; i <= n; i++) {
            int w = items[i-1][0]; // 현재 아이템 무게
            int v = items[i-1][1]; // 현재 아이템 가치
            for (int c = 0; c <= capacity; c++) {
                dp[i][c] = dp[i-1][c]; // 현재 아이템 미포함
                if (c >= w) {
                    dp[i][c] = Math.max(dp[i][c], dp[i-1][c-w] + v);
                }
            }
        }

        // 선택된 아이템 역추적
        System.out.print("선택된 아이템: ");
        int c = capacity;
        for (int i = n; i >= 1; i--) {
            if (dp[i][c] != dp[i-1][c]) {
                System.out.printf("아이템%d(무게:%d, 가치:%d) ",
                    i, items[i-1][0], items[i-1][1]);
                c -= items[i-1][0];
            }
        }
        System.out.println();

        return dp[n][capacity];
    }

    /**
     * 0/1 배낭 문제 - 공간 최적화 1D DP
     * 시간: O(n * W), 공간: O(W)
     * 2D 배열의 i행이 i-1행에만 의존함을 이용
     */
    public static int knapsack1D(int[][] items, int capacity) {
        int[] dp = new int[capacity + 1];

        for (int[] item : items) {
            int w = item[0], v = item[1];
            // 역방향 순회: dp[c-w]가 아직 현재 아이템을 포함하지 않은 값임을 보장
            for (int c = capacity; c >= w; c--) {
                dp[c] = Math.max(dp[c], dp[c - w] + v);
            }
        }

        return dp[capacity];
    }

    /**
     * 동전 교환 문제 (Coin Change) - 무한 개 사용 가능
     * 최소 동전 수로 amount를 만들기
     * 시간: O(amount * |coins|), 공간: O(amount)
     */
    public static int coinChange(int[] coins, int amount) {
        int[] dp = new int[amount + 1];
        Arrays.fill(dp, amount + 1); // 불가능한 큰 값으로 초기화
        dp[0] = 0;

        for (int i = 1; i <= amount; i++) {
            for (int coin : coins) {
                if (i >= coin) {
                    dp[i] = Math.min(dp[i], dp[i - coin] + 1);
                }
            }
        }

        return dp[amount] > amount ? -1 : dp[amount];
    }

    public static void main(String[] args) {
        // 배낭 문제 테스트
        int[][] items = {
            {2, 6},  // 무게 2, 가치 6
            {2, 10}, // 무게 2, 가치 10
            {3, 12}, // 무게 3, 가치 12
        };
        int capacity = 5;

        int result2D = knapsack2D(items, capacity);
        int result1D = knapsack1D(items, capacity);
        System.out.printf("최대 가치 (2D): %d%n", result2D); // 22
        System.out.printf("최대 가치 (1D): %d%n", result1D); // 22

        // 동전 교환 테스트
        int[] coins  = {1, 5, 11};
        int amount   = 15;
        System.out.printf("동전 %s으로 %d원 만들기: %d개%n",
            Arrays.toString(coins), amount, coinChange(coins, amount)); // 3개 (5+9?)
        // 실제: 11+1+1+1+1=5개 vs 5+5+5=3개 → 3개
    }
}
```

배낭 문제의 핵심은 1D DP에서 **역방향 순회**다. 정방향으로 순회하면 같은 아이템을 여러 번 사용하게 되어 배낭 문제가 아니라 완전 배낭 문제(Unbounded Knapsack)로 바뀐다.

---

## 주의사항 및 팁

### 1. 점화식 설계가 전부다

DP 문제를 푸는 핵심은 **점화식(Recurrence Relation)** 과 **상태(State) 정의**다.

- **상태**: `dp[i][j]`가 정확히 무엇을 의미하는지 명확히 정의
- **전이**: `dp[i][j]`를 이전 상태들로부터 어떻게 계산하는지
- **기저 조건**: 가장 작은 하위 문제의 초기값

LCS의 `dp[i][j] = s1[:i]와 s2[:j]의 LCS 길이` 처럼 상태 정의가 명확하면 점화식은 자연스럽게 따라온다.

### 2. 공간 복잡도 최적화

대부분의 2D DP는 현재 행이 이전 행에만 의존하므로 1D 배열로 줄일 수 있다:
- 배낭 문제: O(n·W) → O(W)
- LCS: O(m·n) → O(min(m, n))
- 편집 거리: O(m·n) → O(n)

### 3. 메모이제이션 vs 테이블화 선택 기준

| 기준 | 메모이제이션 | 테이블화 |
|---|---|---|
| 구현 난이도 | 쉬움 (재귀 + 캐시) | 순서 파악 필요 |
| 필요 하위 문제만 계산 | O (Lazy) | X (모두 계산) |
| 스택 오버플로 위험 | 있음 | 없음 |
| 캐시 친화성 | 낮음 | 높음 (배열 순차 접근) |
| 반복 횟수가 매우 많을 때 | 느림 | 빠름 |

### 4. 그리디와의 혼동 주의

배낭 문제는 DP가 필요하지만, 분수 배낭 문제(Fractional Knapsack)는 그리디로 해결 가능하다. 최적 부분구조는 같지만 겹치는 하위 문제가 없기 때문이다. 동전 교환 문제도 특정 동전 집합(예: {1, 5, 10, 50})에서는 그리디가 동작하지만 임의의 동전 집합에서는 DP가 필요하다.

### 5. 상태 공간 폭발 방지

2D 이상의 DP에서 상태 수가 너무 많아지는 경우 **비트마스킹 DP**, **구간 DP(Interval DP)**, **트리 DP** 등 특수 기법을 고려한다. 예: 외판원 순회 문제(TSP)의 비트마스킹 DP는 O(n! ) → O(2^n · n^2)으로 개선한다.

### 6. 실전 DP 풀이 프레임워크

1. 최적 부분구조와 겹치는 하위 문제 확인
2. 상태 명확히 정의 (`dp[i]`가 정확히 무엇인지)
3. 점화식 도출 (경우의 수를 나열해서)
4. 기저 조건 설정
5. 탑다운 메모이제이션으로 먼저 구현
6. 바텀업으로 변환 후 공간 최적화

---

## 참고 자료
- [Dynamic Programming (DP) Introduction - GeeksforGeeks](https://www.geeksforgeeks.org/dsa/introduction-to-dynamic-programming-data-structures-and-algorithm-tutorials/)
- [Dynamic Programming: Mastering Tabulation and Memoization - AlgoCademy](https://algocademy.com/blog/dynamic-programming-mastering-tabulation-and-memoization/)
- [DSA Tabulation - W3Schools](https://www.w3schools.com/dsa/dsa_ref_tabulation.php)
- [Intro to Dynamic Programming - Fordham University CIS](https://storm.cis.fordham.edu/zhang/cs4080/slides/DynamicProgramming.pdf)
