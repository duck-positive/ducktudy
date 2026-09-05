---
layout: post
title: "LCS와 편집 거리(Edit Distance) 완전 정복: diff·맞춤법 교정·DNA 정렬의 수학적 토대"
date: 2026-09-05
categories: [cs, computer-science]
tags: [lcs, edit-distance, levenshtein, dynamic-programming, string-algorithms, diff, hirschberg, alignment]
---

두 텍스트 파일의 차이를 보여주는 `git diff`, 검색창의 맞춤법 자동 교정, 생물정보학의 DNA 서열 정렬 — 이 모두가 하나의 공통된 알고리즘 토대 위에 서 있다. **최장 공통 부분 수열(LCS, Longest Common Subsequence)**과 **편집 거리(Edit Distance, Levenshtein Distance)**다. 두 알고리즘은 깊이 연관되어 있으며, 동적 프로그래밍(DP)의 정수를 보여주는 대표적인 문제다.

## 최장 공통 부분 수열(LCS)이란 무엇인가

**부분 수열(subsequence)**이란 원래 순서를 유지하되 연속일 필요는 없는 부분 원소 집합이다.

- `ABCDE`의 부분 수열: `ACE`, `BD`, `ABCDE`, `""`, `A`, ...
- `"ACGT"`와 `"CATG"`의 LCS: `"AT"` (길이 2)

LCS는 두 수열에서 공통으로 등장하는 부분 수열 중 가장 긴 것을 찾는 문제다.

**부분 문자열(substring)**과 혼동하지 말 것: 부분 문자열은 반드시 연속이어야 한다.

### LCS의 점화식

두 문자열 `X[1..m]`과 `Y[1..n]`에 대해, `dp[i][j]`를 `X[1..i]`와 `Y[1..j]`의 LCS 길이라 하면:

```
dp[0][j] = 0, dp[i][0] = 0          (기저 조건)

dp[i][j] = dp[i-1][j-1] + 1         if X[i] == Y[j]
dp[i][j] = max(dp[i-1][j], dp[i][j-1])  otherwise
```

마지막 문자가 같으면 그 문자를 LCS에 포함시키고, 다르면 한 쪽을 줄인 경우 중 큰 것을 택한다.

## 편집 거리(Edit Distance / Levenshtein Distance)란 무엇인가

편집 거리는 한 문자열을 다른 문자열로 바꾸는 데 필요한 **최소 편집 연산 횟수**다. Levenshtein이 1965년 제안한 표준 정의에서는 세 연산을 허용한다:

1. **삽입(Insertion)**: 문자 하나 삽입
2. **삭제(Deletion)**: 문자 하나 삭제
3. **대체(Substitution)**: 문자 하나를 다른 문자로 교체

예: `"kitten"` → `"sitting"`
```
kitten
→ sitten   (k→s 대체, 비용 1)
→ sittin   (e→i 대체, 비용 1)
→ sitting  (g 삽입, 비용 1)
편집 거리 = 3
```

### 편집 거리의 점화식

`dp[i][j]`를 `X[1..i]`를 `Y[1..j]`로 바꾸는 최소 편집 횟수라 하면:

```
dp[0][j] = j          (Y의 j번째 접두사로 만들려면 j번 삽입)
dp[i][0] = i          (X의 i번째 접두사를 빈 문자열로 만들려면 i번 삭제)

dp[i][j] = dp[i-1][j-1]                          if X[i] == Y[j]  (비용 0)
dp[i][j] = 1 + min(dp[i-1][j],    # 삭제
                   dp[i][j-1],    # 삽입
                   dp[i-1][j-1])  # 대체      otherwise
```

### LCS와 편집 거리의 관계

편집 거리(삽입·삭제만 허용, 대체 없음)는 LCS와 직접 연결된다:

```
edit_distance_no_sub(X, Y) = len(X) + len(Y) - 2 * LCS(X, Y)
```

두 수열 중 LCS에 해당하는 문자는 건드리지 않고, 나머지는 삭제하거나 삽입하면 되기 때문이다. 이 관계는 diff 알고리즘의 핵심이기도 하다.

## 왜 필요한가 — 실제 적용 사례

| 분야 | LCS/Edit Distance 활용 |
|---|---|
| **버전 관리 (git diff)** | 두 파일의 줄(line) 단위 LCS → 추가/삭제된 줄 표시 |
| **맞춤법 교정** | 사전 단어와의 편집 거리 ≤ k인 단어 후보 제시 |
| **DNA 서열 정렬** | 유전자 유사도 측정, 진화적 거리 계산 |
| **표절 탐지** | 두 문서의 유사도 측정 |
| **자연어 처리** | 단어 임베딩 품질 평가, 의미적 유사도 기준 |
| **데이터 정제** | 중복 레코드 식별 (fuzzy matching) |

## 구현: LCS와 편집 거리

### Python: O(m×n) DP 구현

```python
from typing import List, Tuple

def lcs_length(X: str, Y: str) -> int:
    """두 문자열의 LCS 길이를 계산한다. O(m*n) 시간, O(m*n) 공간."""
    m, n = len(X), len(Y)
    dp = [[0] * (n + 1) for _ in range(m + 1)]

    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if X[i - 1] == Y[j - 1]:
                dp[i][j] = dp[i - 1][j - 1] + 1
            else:
                dp[i][j] = max(dp[i - 1][j], dp[i][j - 1])

    return dp[m][n]


def lcs_string(X: str, Y: str) -> str:
    """실제 LCS 문자열을 역추적으로 복원한다."""
    m, n = len(X), len(Y)
    dp = [[0] * (n + 1) for _ in range(m + 1)]

    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if X[i - 1] == Y[j - 1]:
                dp[i][j] = dp[i - 1][j - 1] + 1
            else:
                dp[i][j] = max(dp[i - 1][j], dp[i][j - 1])

    # 역추적 (backtracking)
    result = []
    i, j = m, n
    while i > 0 and j > 0:
        if X[i - 1] == Y[j - 1]:
            result.append(X[i - 1])
            i -= 1
            j -= 1
        elif dp[i - 1][j] >= dp[i][j - 1]:
            i -= 1
        else:
            j -= 1

    return "".join(reversed(result))


def edit_distance(X: str, Y: str) -> int:
    """Levenshtein 편집 거리를 계산한다. O(m*n) 시간, O(m*n) 공간."""
    m, n = len(X), len(Y)
    dp = [[0] * (n + 1) for _ in range(m + 1)]

    # 기저 조건
    for i in range(m + 1):
        dp[i][0] = i
    for j in range(n + 1):
        dp[0][j] = j

    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if X[i - 1] == Y[j - 1]:
                dp[i][j] = dp[i - 1][j - 1]
            else:
                dp[i][j] = 1 + min(
                    dp[i - 1][j],      # 삭제
                    dp[i][j - 1],      # 삽입
                    dp[i - 1][j - 1],  # 대체
                )

    return dp[m][n]


def edit_operations(X: str, Y: str) -> List[Tuple[str, str]]:
    """편집 연산 목록을 역추적으로 복원한다."""
    m, n = len(X), len(Y)
    dp = [[0] * (n + 1) for _ in range(m + 1)]
    for i in range(m + 1):
        dp[i][0] = i
    for j in range(n + 1):
        dp[0][j] = j
    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if X[i - 1] == Y[j - 1]:
                dp[i][j] = dp[i - 1][j - 1]
            else:
                dp[i][j] = 1 + min(dp[i-1][j], dp[i][j-1], dp[i-1][j-1])

    # 역추적
    ops = []
    i, j = m, n
    while i > 0 or j > 0:
        if i > 0 and j > 0 and X[i-1] == Y[j-1]:
            ops.append(("match", X[i-1]))
            i -= 1; j -= 1
        elif i > 0 and j > 0 and dp[i][j] == dp[i-1][j-1] + 1:
            ops.append(("replace", f"{X[i-1]}→{Y[j-1]}"))
            i -= 1; j -= 1
        elif i > 0 and dp[i][j] == dp[i-1][j] + 1:
            ops.append(("delete", X[i-1]))
            i -= 1
        else:
            ops.append(("insert", Y[j-1]))
            j -= 1

    return list(reversed(ops))


# 테스트
if __name__ == "__main__":
    X, Y = "AGGTAB", "GXTXAYB"
    print(f"LCS({X!r}, {Y!r}) = {lcs_string(X, Y)!r}  (길이: {lcs_length(X, Y)})")
    # LCS('AGGTAB', 'GXTXAYB') = 'GTAB'  (길이: 4)

    A, B = "kitten", "sitting"
    print(f"edit_distance({A!r}, {B!r}) = {edit_distance(A, B)}")
    for op in edit_operations(A, B):
        print(f"  {op}")
```

### C++: 공간 최적화 O(min(m,n))

DP 테이블 전체를 저장할 필요 없이 **두 행만** 유지하면 공간을 O(min(m,n))으로 줄일 수 있다:

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <algorithm>

// 공간 최적화 편집 거리: O(min(m,n)) 공간
int editDistance(const std::string& X, const std::string& Y) {
    int m = X.size(), n = Y.size();

    // 항상 짧은 쪽을 Y로 (열 방향)
    if (m < n) return editDistance(Y, X);

    std::vector<int> prev(n + 1), curr(n + 1);

    // 첫 행 초기화
    for (int j = 0; j <= n; ++j) prev[j] = j;

    for (int i = 1; i <= m; ++i) {
        curr[0] = i;
        for (int j = 1; j <= n; ++j) {
            if (X[i - 1] == Y[j - 1]) {
                curr[j] = prev[j - 1];
            } else {
                curr[j] = 1 + std::min({prev[j],      // 삭제
                                         curr[j - 1],  // 삽입
                                         prev[j - 1]}); // 대체
            }
        }
        std::swap(prev, curr);
    }
    return prev[n];
}

// LCS 길이 (공간 최적화 버전)
int lcsLength(const std::string& X, const std::string& Y) {
    int m = X.size(), n = Y.size();
    std::vector<int> prev(n + 1, 0), curr(n + 1, 0);

    for (int i = 1; i <= m; ++i) {
        for (int j = 1; j <= n; ++j) {
            if (X[i - 1] == Y[j - 1])
                curr[j] = prev[j - 1] + 1;
            else
                curr[j] = std::max(prev[j], curr[j - 1]);
        }
        std::swap(prev, curr);
        std::fill(curr.begin(), curr.end(), 0);
    }
    return prev[n];
}

int main() {
    std::string X = "ABCBDAB", Y = "BDCAB";
    std::cout << "LCS 길이: " << lcsLength(X, Y) << "\n";  // 4

    std::cout << "편집 거리: " << editDistance("saturday", "sunday") << "\n";  // 3
    return 0;
}
```

## Hirschberg's Algorithm: LCS를 O(min(m,n)) 공간에 복원하기

LCS **길이**는 O(min(m,n)) 공간으로 계산 가능하지만, **실제 LCS 문자열 복원**은 DP 테이블 전체가 필요해 O(m×n) 공간이 든다. Hirschberg(1975)는 분할 정복 기법으로 이를 O(min(m,n)) 공간에서도 LCS 문자열을 복원하는 알고리즘을 고안했다.

```python
def lcs_row(X: str, Y: str) -> list:
    """X와 Y의 마지막 DP 행만 반환 (O(n) 공간)."""
    n = len(Y)
    prev = [0] * (n + 1)
    for ch in X:
        curr = [0] * (n + 1)
        for j, cy in enumerate(Y, 1):
            if ch == cy:
                curr[j] = prev[j - 1] + 1
            else:
                curr[j] = max(prev[j], curr[j - 1])
        prev = curr
    return prev


def hirschberg(X: str, Y: str) -> str:
    """Hirschberg 알고리즘: O(mn) 시간, O(min(m,n)) 공간에서 LCS 복원."""
    if not X or not Y:
        return ""
    if len(X) == 1:
        return X[0] if X[0] in Y else ""
    if len(Y) == 1:
        return Y[0] if Y[0] in X else ""

    mid = len(X) // 2
    # 앞쪽 절반과 Y의 LCS 마지막 행
    score_top = lcs_row(X[:mid], Y)
    # 뒤쪽 절반의 역순과 Y의 역순의 LCS 마지막 행
    score_bot = lcs_row(X[mid:][::-1], Y[::-1])

    # 최적 분할점 찾기
    split = max(range(len(Y) + 1), key=lambda k: score_top[k] + score_bot[len(Y) - k])

    # 재귀적으로 좌우 분할
    return hirschberg(X[:mid], Y[:split]) + hirschberg(X[mid:], Y[split:])


# 검증
X, Y = "AGTACGCA", "TATGC"
print(f"Hirschberg LCS: {hirschberg(X, Y)!r}")   # 'ATGC' 또는 동일 길이의 다른 LCS
print(f"참조 LCS:       {lcs_string(X, Y)!r}")
```

## diff 알고리즘과의 연결: Myers 알고리즘

실제 `git diff`에 사용되는 **Myers diff 알고리즘(1986)**은 LCS를 기반으로 하되, `O(D)` 공간(D=편집 횟수)으로 동작한다. 핵심 아이디어는 "가장 짧은 편집 스크립트(Shortest Edit Script)"를 찾는 것으로, 대각선 방향 이동(공통 문자)을 최대화하는 그리디 BFS 탐색이다.

```python
def myers_diff(old: list, new: list):
    """Myers diff 알고리즘 — 줄(line) 단위 diff.
    반환: ('+', line), ('-', line), ('=', line) 튜플 리스트."""
    n, m = len(old), len(new)
    max_d = n + m

    # v[k] = k 대각선에서 도달한 최대 x 좌표
    v = {1: 0}
    trace = []

    for d in range(max_d + 1):
        trace.append(dict(v))
        for k in range(-d, d + 1, 2):
            if k == -d or (k != d and v.get(k - 1, -1) < v.get(k + 1, -1)):
                x = v.get(k + 1, 0)  # 삽입 (아래로 이동)
            else:
                x = v.get(k - 1, 0) + 1  # 삭제 (오른쪽 이동)
            y = x - k
            # 대각선 이동 (공통 문자)
            while x < n and y < m and old[x] == new[y]:
                x += 1
                y += 1
            v[k] = x
            if x >= n and y >= m:
                # 역추적
                return _backtrack(trace, old, new, d)

    return []


def _backtrack(trace, old, new, d):
    x, y = len(old), len(new)
    result = []
    for cur_d in range(d, -1, -1):
        v = trace[cur_d]
        k = x - y
        if k == -cur_d or (k != cur_d and v.get(k - 1, -1) < v.get(k + 1, -1)):
            prev_k = k + 1
        else:
            prev_k = k - 1
        prev_x = v.get(prev_k, 0)
        prev_y = prev_x - prev_k
        # 대각선 이동 (공통)
        while x > prev_x and y > prev_y:
            result.append(("=", old[x - 1]))
            x -= 1; y -= 1
        if cur_d > 0:
            if x == prev_x:
                result.append(("+", new[y - 1]))
                y -= 1
            else:
                result.append(("-", old[x - 1]))
                x -= 1
    return list(reversed(result))


# 예시
old_lines = ["apple", "banana", "cherry", "date"]
new_lines = ["apple", "blueberry", "cherry", "elderberry", "date"]
for op, line in myers_diff(old_lines, new_lines):
    prefix = {"=": " ", "+": "+", "-": "-"}[op]
    print(f"{prefix} {line}")
```

## 주의사항과 팁

### 시간 복잡도

두 알고리즘 모두 기본 DP는 **O(m×n)** 시간이다. 짧은 문자열(수백 자 이내)에서는 문제 없지만, 긴 문자열(수만 자 이상)이나 대용량 파일 비교에는 최적화가 필요하다:
- **비트 벡터 최적화**: 문자 집합이 작으면 비트 연산으로 상수 인수 개선 가능
- **4-러시안 알고리즘**: O(n²/log n)으로 개선
- **근사 알고리즘**: 정확도를 약간 포기하고 O(n log n)으로 개선

### 가중치 편집 거리

실제 맞춤법 교정에서는 **연산마다 비용이 다를 수** 있다. 예를 들어 키보드에서 인접한 키의 대체는 비용 0.5, 먼 키는 비용 1.5로 설정하면 더 자연스러운 교정이 된다. 점화식에서 비용 1 대신 `cost(op, char)` 함수로 대체하면 된다.

### Jaro-Winkler: 짧은 문자열 비교

이름·주소 같은 짧은 문자열의 유사도를 측정할 때는 Jaro-Winkler 거리가 Levenshtein보다 사람의 직관에 더 잘 맞는 경우가 많다. fuzzy matching 라이브러리(예: Python `thefuzz`)는 이를 지원한다.

LCS와 편집 거리는 단순해 보이지만, DP 설계의 정수를 담고 있으며 수십 년간 파서·컴파일러·데이터베이스·생물정보학에서 핵심 역할을 해왔다. 이 두 알고리즘을 완전히 이해하면, 문자열 처리 문제 대부분의 해법이 자연스럽게 도출된다.

## 참고 자료
- [Longest Common Subsequence — Wikipedia](https://en.wikipedia.org/wiki/Longest_common_subsequence)
- [Edit Distance — Wikipedia](https://en.wikipedia.org/wiki/Edit_distance)
- [An O(ND) Difference Algorithm (Myers, 1986)](https://www.researchgate.net/publication/220184833_An_ONP_Difference_Algorithm)
- [A Linear Space Algorithm for Computing Longest Common Subsequences (Hirschberg, 1975)](https://dl.acm.org/doi/10.1145/360825.360861)
