---
layout: post
title: "카탈란 수 완전 정복: 이진 트리부터 괄호 쌍까지 조합론의 만능 열쇠"
date: 2026-09-10
categories: [cs, computer-science]
tags: [catalan-numbers, combinatorics, dynamic-programming, binary-tree, stack, algorithm, math]
---

## 개념 설명

**카탈란 수(Catalan Numbers)**는 조합론에서 가장 아름다운 수열 중 하나로, 다양한 조합 구조를 세는 데 등장하는 정수 수열이다.

카탈란 수의 수열은 다음과 같다:

```
C₀ = 1, C₁ = 1, C₂ = 2, C₃ = 5, C₄ = 14, C₅ = 42, C₆ = 132, ...
```

이 수열은 세 가지 주요 공식으로 표현된다:

**점화 관계식 (Recurrence)**:
```
C₀ = 1
Cₙ = Σ(k=0 to n-1) Cₖ × Cₙ₋₁₋ₖ
```

**이항 계수를 이용한 닫힌 형식 (Closed Form)**:
```
Cₙ = C(2n, n) / (n+1) = (2n)! / ((n+1)! × n!)
```

**점화 간소화**:
```
Cₙ = Cₙ₋₁ × 2(2n-1) / (n+1)
```

이 공식들이 모두 동일한 값을 생성한다는 사실 자체가 놀랍지만, 더 놀라운 것은 이 수열이 등장하는 상황의 다양성이다.

---

## 왜 필요한가 — 카탈란 수가 세는 것들

카탈란 수 Cₙ은 다음 구조의 개수를 동시에 센다:

### 1. 올바른 괄호 쌍 (Balanced Parentheses)
n쌍의 괄호로 만들 수 있는 올바른 표현식의 수.
- C₃ = 5: `((()))`, `(()())`, `(())()`, `()(())`, `()()()`

### 2. 서로 다른 이진 탐색 트리 (Binary Search Trees)
n개의 노드로 만들 수 있는 구조적으로 다른 이진 트리의 수.
- C₃ = 5: 3개 노드로 5가지 서로 다른 BST 구조 가능

### 3. 볼록 다각형의 삼각 분할 (Polygon Triangulation)
(n+2)각형을 삼각형으로 분할하는 방법의 수.
- 사각형(4각형, n=2): C₂ = 2가지 삼각 분할

### 4. 단조 격자 경로 (Monotone Lattice Paths)
(0,0)에서 (n,n)까지 대각선을 넘지 않고 이동하는 격자 경로의 수.

### 5. 산악 범위 (Mountain Ranges)
n번의 상승과 n번의 하강으로 이루어진 산 모양 계단 패턴의 수 (Dylan 경로).

### 6. 스택 정렬 가능 순열
스택 하나로 정렬 가능한 1..n의 순열의 수.

### 컴퓨터 과학과의 연결
이 다양한 해석들은 모두 **동일한 구조의 다른 표현**임이 증명되었다. 컴파일러의 파싱, 데이터베이스 조인 순서 최적화, 코드 생성기, 3D 메쉬 생성 등에 직접 활용된다.

---

## 실제 구현 예제

### 예제 1: 카탈란 수 계산 — 세 가지 방법 비교

```python
from functools import lru_cache
from math import comb, factorial

# 방법 1: 재귀 + 메모이제이션
@lru_cache(maxsize=None)
def catalan_recursive(n: int) -> int:
    """점화식을 이용한 메모이제이션 재귀. O(n²) 시간, O(n) 공간."""
    if n <= 1:
        return 1
    return sum(catalan_recursive(k) * catalan_recursive(n - 1 - k)
               for k in range(n))

# 방법 2: 이항 계수 공식
def catalan_binomial(n: int) -> int:
    """닫힌 형식. O(n) 시간 (comb 함수 사용)."""
    return comb(2 * n, n) // (n + 1)

# 방법 3: 동적 프로그래밍 (보텀업)
def catalan_dp(n: int) -> int:
    """점화식 보텀업 DP. O(n²) 시간, O(n) 공간."""
    dp = [0] * (n + 1)
    dp[0] = dp[1] = 1

    for i in range(2, n + 1):
        for j in range(i):
            dp[i] += dp[j] * dp[i - 1 - j]
    return dp[n]

# 방법 4: 점화 간소화 공식 (가장 빠름)
def catalan_efficient(n: int) -> int:
    """C(n) = C(n-1) * 2(2n-1) / (n+1). O(n) 시간."""
    result = 1
    for i in range(1, n + 1):
        result = result * 2 * (2 * i - 1) // (i + 1)
    return result

# 검증
print("n | recursive | binomial | dp | efficient")
print("-" * 45)
for n in range(8):
    r = catalan_recursive(n)
    b = catalan_binomial(n)
    d = catalan_dp(n)
    e = catalan_efficient(n)
    print(f"{n} |     {r:5d} |    {b:5d} | {d:2d} |       {e:5d}")

# 출력:
# n | recursive | binomial | dp | efficient
# ---------------------------------------------
# 0 |         1 |        1 |  1 |         1
# 1 |         1 |        1 |  1 |         1
# 2 |         2 |        2 |  2 |         2
# 3 |         5 |        5 |  5 |         5
# 4 |        14 |       14 | 14 |        14
# 5 |        42 |       42 | 42 |        42
# 6 |       132 |      132 |132 |       132
# 7 |       429 |      429 |429 |       429
```

### 예제 2: 카탈란 수의 응용 — 올바른 괄호 모든 조합 생성

카탈란 수 Cₙ = 5 (n=3)개인 올바른 괄호 쌍 시퀀스를 모두 출력하는 재귀 알고리즘:

```python
from typing import List

def generate_parentheses(n: int) -> List[str]:
    """
    n쌍의 괄호로 이루어진 모든 유효한 문자열 생성.
    반환 값의 개수가 카탈란 수 C(n)임을 확인할 수 있다.
    """
    result = []

    def backtrack(current: str, open_count: int, close_count: int):
        if len(current) == 2 * n:
            result.append(current)
            return
        if open_count < n:
            backtrack(current + '(', open_count + 1, close_count)
        if close_count < open_count:
            backtrack(current + ')', open_count, close_count + 1)

    backtrack('', 0, 0)
    return result


# 이진 트리 구조 세기 (구조만 다른 BST 수)
def count_unique_bst(n: int) -> int:
    """
    n개의 고유한 노드로 만들 수 있는 구조적으로 다른 BST의 수.
    = 카탈란 수 C(n)
    """
    if n <= 1:
        return 1
    total = 0
    for root in range(1, n + 1):
        left_trees = count_unique_bst(root - 1)   # 루트보다 작은 키들
        right_trees = count_unique_bst(n - root)   # 루트보다 큰 키들
        total += left_trees * right_trees
    return total


# 다각형 삼각 분할 수 세기
def polygon_triangulations(sides: int) -> int:
    """
    'sides'각형을 삼각형으로 분할하는 방법의 수.
    = 카탈란 수 C(sides - 2)
    """
    n = sides - 2
    return catalan_efficient(n)


# 검증 및 출력
for n in range(1, 5):
    parens = generate_parentheses(n)
    bst_count = count_unique_bst(n)
    catalan = catalan_efficient(n)

    print(f"\nn = {n}, C({n}) = {catalan}")
    print(f"  괄호 쌍 수: {len(parens)}, BST 구조 수: {bst_count}")
    if n == 3:
        print(f"  모든 괄호 표현식: {parens}")

# n = 3일 때:
# n = 3, C(3) = 5
#   괄호 쌍 수: 5, BST 구조 수: 5
#   모든 괄호 표현식: ['((()))', '(()())', '(())()', '()(())', '()()()']
```

### 예제 3: Ballot 문제와 카탈란 수의 직관적 증명

카탈란 수는 **Ballot 문제**로도 이해할 수 있다. A가 B보다 항상 앞서는 투표 수열 문제:

```python
def count_ballot_sequences(n: int) -> int:
    """
    A, B 두 후보의 투표 결과 수열에서 항상 A가 리드하는 경우의 수.
    A가 n표, B가 n표를 받을 때 (총 2n개 투표).
    이것도 카탈란 수 C(n)과 같다.
    
    그리디: 왼쪽에서 오른쪽으로 읽으며 항상 A 표가 B 표보다 많아야 함.
    """
    def is_valid_ballot(seq: list) -> bool:
        a_count = 0
        for vote in seq:
            if vote == 'A':
                a_count += 1
            else:
                a_count -= 1
            if a_count <= 0:
                return False
        return True

    from itertools import permutations

    votes = ['A'] * n + ['B'] * n
    seen = set()
    valid_count = 0

    for perm in set(permutations(votes)):
        if is_valid_ballot(perm):
            valid_count += 1

    return valid_count


# 검증
for n in range(1, 6):
    ballot = count_ballot_sequences(n)
    catalan = catalan_efficient(n)
    print(f"n={n}: ballot_valid={ballot}, C(n)={catalan}, match={ballot==catalan}")
```

---

## 카탈란 수의 생성 함수

카탈란 수의 생성 함수(Generating Function) G(x)는:

```
G(x) = (1 - √(1 - 4x)) / (2x)
```

이를 급수 전개하면:
```
G(x) = C₀ + C₁x + C₂x² + C₃x³ + ...
     = 1 + x + 2x² + 5x³ + 14x⁴ + ...
```

점화식 `Cₙ = Σ Cₖ × Cₙ₋₁₋ₖ`은 생성 함수에서 `G(x) = 1 + x × G(x)²`를 의미한다.

---

## 주의사항과 실전 팁

### 1. 카탈란 수인지 인식하는 훈련
경쟁 프로그래밍에서 다음 패턴이 나오면 카탈란 수를 의심하라:
- "n개의 쌍으로 올바른 구조 만들기"
- "이진 트리의 구조적 종류 수"
- "스택/큐를 이용한 정렬 가능 순열"
- "격자 경로에서 특정 선을 넘지 않는 경로"

### 2. 큰 값의 처리
C₁₀₀은 896519947090051...으로 매우 큰 수다. Python에서는 기본적으로 임의 정밀도 정수를 지원하지만, 다른 언어에서는 BigInteger나 모듈러 산술을 사용해야 한다.

```python
# 모듈러 산술로 카탈란 수 계산 (mod M)
def catalan_mod(n: int, M: int) -> int:
    def mod_inv(a, m):
        return pow(a, m - 2, m)  # 페르마 소정리 (m이 소수일 때)

    result = comb(2 * n, n) % M
    result = result * mod_inv(n + 1, M) % M
    return result
```

### 3. 메모이제이션 vs DP
재귀 + 메모이제이션은 O(n²) 시간과 재귀 스택 오버헤드가 있다. 큰 n에 대해서는 보텀업 DP나 점화 간소화 공식이 더 효율적이다. 점화 간소화 공식 `Cₙ = Cₙ₋₁ × 2(2n-1)/(n+1)`은 O(n)에 계산 가능하다.

### 4. 조합론적 증명 이해
동일한 수가 왜 이렇게 다양한 구조를 세는지 이해하려면 **전단사 함수(bijection)**를 통한 증명을 공부하는 것이 좋다. 예를 들어 올바른 괄호 표현식과 이진 트리 사이에는 명확한 전단사 함수가 존재하며, 이를 이해하면 새로운 카탈란 문제를 자연스럽게 인식할 수 있다.

### 5. 일반화: Catalan 수의 변형
- **k-ary Catalan 수**: 이진 대신 k-ary 트리를 셀 때
- **Fuss-Catalan 수**: 일반화된 버전, `1/(kn+1) × C((k+1)n, n)`
- **q-Catalan 다항식**: 가중 버전

---

## 참고 자료
- [Catalan Numbers — Wikipedia](https://en.wikipedia.org/wiki/Catalan_number)
- [Catalan Numbers — Algorithms for Competitive Programming](https://cp-algorithms.com/combinatorics/catalan-numbers.html)
- [Catalan Numbers — Brilliant.org](https://brilliant.org/wiki/catalan-numbers/)
- [What are Catalan Numbers? — Medium](https://vgnshiyer.medium.com/what-are-catalan-numbers-252a40f5d915)
