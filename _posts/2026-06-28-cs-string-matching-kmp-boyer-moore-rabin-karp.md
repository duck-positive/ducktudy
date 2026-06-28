---
layout: post
title: "문자열 매칭 알고리즘 완전 정복: KMP, Boyer-Moore, Rabin-Karp 구현과 비교"
date: 2026-06-28
categories: [cs, computer-science]
tags: [string-matching, kmp, boyer-moore, rabin-karp, algorithm, pattern-search, rolling-hash]
---

텍스트에서 패턴을 찾는 문자열 매칭(String Matching)은 컴파일러, 검색 엔진, DNA 분석, 네트워크 침입 탐지 시스템(IDS)에 이르기까지 소프트웨어 전반에 걸쳐 사용되는 핵심 알고리즘이다. 가장 단순한 방법(브루트 포스)은 O(n·m) 시간이 걸리지만, 1970년대에 등장한 세 가지 알고리즘이 이를 혁신적으로 개선했다.

## 왜 브루트 포스로는 부족한가

텍스트 길이가 n, 패턴 길이가 m일 때 나이브 알고리즘은 최악의 경우 n - m + 1개의 위치에서 m번씩 비교한다.

```
텍스트: "AAAAAAAAAB" (n=10)
패턴:   "AAAAB"      (m=5)
```

매 위치에서 4번 일치 후 1번 불일치가 반복되어 거의 O(n·m)에 수렴한다. DNA 서열(수십억 염기쌍)이나 대용량 로그 파일에서는 현실적으로 불가능하다.

## KMP 알고리즘 (Knuth-Morris-Pratt)

### 핵심 아이디어: 실패 함수(Failure Function)

KMP의 통찰은 **"불일치가 발생했을 때, 이미 비교한 정보를 버리지 말자"**는 것이다. 패턴에서 접두사(prefix)이면서 동시에 접미사(suffix)인 가장 긴 문자열의 길이를 미리 계산해두면, 불일치 시 텍스트 포인터를 되돌리지 않고도 패턴 포인터만 이동할 수 있다.

이 사전 계산 테이블을 **LPS(Longest Proper Prefix which is also Suffix)** 또는 **실패 함수(π)**라고 부른다.

```
패턴: "ABABCABAB"
인덱스: 0 1 2 3 4 5 6 7 8
LPS:    0 0 1 2 0 1 2 3 4
```

`LPS[4] = 0`: "ABABC"에서 접두사이면서 접미사인 최장 문자열의 길이가 0
`LPS[7] = 3`: "ABABCABA"에서 "ABA"가 접두사이자 접미사이므로 길이 3

### 예제 1: Python KMP 구현

```python
def compute_lps(pattern):
    """LPS(실패 함수) 테이블 계산: O(m)"""
    m = len(pattern)
    lps = [0] * m
    length = 0  # 이전 접두사-접미사의 길이
    i = 1

    while i < m:
        if pattern[i] == pattern[length]:
            length += 1
            lps[i] = length
            i += 1
        else:
            if length != 0:
                # 패턴을 되돌리지 않고 length를 lps[length-1]로 이동
                length = lps[length - 1]
                # i는 증가하지 않음
            else:
                lps[i] = 0
                i += 1
    return lps

def kmp_search(text, pattern):
    """KMP 문자열 매칭: O(n + m)"""
    n, m = len(text), len(pattern)
    if m == 0:
        return []

    lps = compute_lps(pattern)
    matches = []

    i = 0  # 텍스트 포인터
    j = 0  # 패턴 포인터

    while i < n:
        if text[i] == pattern[j]:
            i += 1
            j += 1

        if j == m:
            # 패턴 완전 일치 발견
            matches.append(i - j)
            j = lps[j - 1]  # 다음 매칭 탐색을 위해 패턴 포인터 이동
        elif i < n and text[i] != pattern[j]:
            if j != 0:
                j = lps[j - 1]  # 텍스트 포인터는 그대로, 패턴만 이동
            else:
                i += 1

    return matches

# 테스트
text    = "ABABDABACDABABCABAB"
pattern = "ABABCABAB"
result  = kmp_search(text, pattern)
print(f"패턴 발견 위치: {result}")  # [10]

text2 = "AAAAABAAAAB"
pattern2 = "AAAB"
print(f"반복 패턴 발견: {kmp_search(text2, pattern2)}")  # [2, 7]
```

KMP의 시간 복잡도는 **O(n + m)**이다. 텍스트 포인터 `i`는 절대 뒤로 가지 않으며, 패턴 포인터 `j`도 전체 합산 이동이 O(n)을 넘지 않는다.

## Boyer-Moore 알고리즘

### 핵심 아이디어: 뒤에서 앞으로, 점프하며 비교

Boyer-Moore는 두 가지 휴리스틱을 조합한다:
1. **Bad Character Rule**: 불일치 문자가 패턴에 존재하면 그 위치까지 패턴을 이동
2. **Good Suffix Rule**: 이미 일치한 접미사를 활용해 더 먼 위치로 이동

실전에서는 Bad Character Rule만 구현해도 대부분의 경우 빠르다. Boyer-Moore는 **평균 O(n/m)**, 최악 O(n·m), 최선 O(n/m)이며, 영어 자연어 처리에서 KMP보다 훨씬 빠른 경향이 있다.

## Rabin-Karp 알고리즘

### 핵심 아이디어: 롤링 해시(Rolling Hash)

Rabin-Karp는 패턴과 텍스트의 부분 문자열을 수치화(해싱)해서 비교한다. 해시가 같을 때만 실제 문자 비교를 수행한다. 핵심은 **롤링 해시**: 윈도우를 한 칸 이동할 때 전체를 재계산하지 않고 이전 해시에서 O(1)에 새 해시를 계산한다.

```
hash("abc") = a * p^2 + b * p^1 + c * p^0  (mod q)
윈도우 이동: 'd' 추가, 'a' 제거
hash("bcd") = (hash("abc") - a * p^2) * p + d  (mod q)
```

### 예제 2: Python Rabin-Karp 구현 (다중 패턴 매칭 포함)

```python
def rabin_karp_single(text, pattern, base=31, mod=10**9 + 7):
    """단일 패턴 Rabin-Karp: O(n + m) 평균"""
    n, m = len(text), len(pattern)
    if m > n:
        return []

    # 패턴과 첫 윈도우의 해시 계산
    pattern_hash = 0
    window_hash = 0
    power = 1  # base^(m-1)

    for i in range(m - 1):
        power = (power * base) % mod

    for i in range(m):
        pattern_hash = (pattern_hash * base + ord(pattern[i])) % mod
        window_hash  = (window_hash  * base + ord(text[i]))    % mod

    matches = []

    for i in range(n - m + 1):
        if window_hash == pattern_hash:
            # 해시 충돌 방지: 실제 문자 비교
            if text[i:i+m] == pattern:
                matches.append(i)

        if i < n - m:
            # 롤링 해시: 왼쪽 문자 제거, 오른쪽 문자 추가
            window_hash = (window_hash - ord(text[i]) * power) % mod
            window_hash = (window_hash * base + ord(text[i + m])) % mod
            window_hash = (window_hash + mod) % mod  # 음수 방지

    return matches

def rabin_karp_multi(text, patterns, base=31, mod=10**9 + 7):
    """다중 패턴 Rabin-Karp: 패턴이 많을 때 KMP보다 유리"""
    # 같은 길이 패턴들을 그룹화
    from collections import defaultdict
    by_length = defaultdict(list)
    for p in patterns:
        by_length[len(p)].append(p)

    results = defaultdict(list)

    for m, pats in by_length.items():
        pat_hashes = {}
        for p in pats:
            h = 0
            for ch in p:
                h = (h * base + ord(ch)) % mod
            pat_hashes[h] = p  # 해시 → 패턴 매핑

        n = len(text)
        if m > n:
            continue

        power = pow(base, m - 1, mod)
        window_hash = 0
        for i in range(m):
            window_hash = (window_hash * base + ord(text[i])) % mod

        for i in range(n - m + 1):
            if window_hash in pat_hashes:
                candidate = pat_hashes[window_hash]
                if text[i:i+m] == candidate:
                    results[candidate].append(i)
            if i < n - m:
                window_hash = (window_hash - ord(text[i]) * power) % mod
                window_hash = (window_hash * base + ord(text[i + m])) % mod
                window_hash = (window_hash + mod) % mod

    return dict(results)

# 단일 패턴
text = "GEEKSFORGEEKS"
print(rabin_karp_single(text, "GEEK"))   # [0, 8]
print(rabin_karp_single(text, "EEK"))    # [1, 9]

# 다중 패턴: IDS(침입 탐지) 시뮬레이션
log = "GET /admin HTTP/1.1 ... DROP TABLE users; SELECT * FROM ..."
signatures = ["DROP TABLE", "SELECT *", "UNION SELECT", "OR 1=1"]
found = rabin_karp_multi(log, signatures)
print("SQL 인젝션 패턴 탐지:", found)
```

다중 패턴 탐지는 Rabin-Karp의 강점이다. KMP는 패턴마다 별도의 LPS 테이블과 탐색 루프를 실행해야 하지만, Rabin-Karp는 같은 길이의 패턴들을 해시셋으로 O(1)에 조회한다.

## 세 알고리즘 비교

| 알고리즘 | 전처리 | 검색 평균 | 검색 최악 | 강점 |
|----------|--------|-----------|-----------|------|
| 나이브 | O(1) | O(n·m) | O(n·m) | 구현 단순 |
| KMP | O(m) | O(n) | O(n+m) | 최악 보장, 단일 패턴 |
| Boyer-Moore | O(m+σ) | O(n/m) | O(n·m) | 자연어 텍스트 검색 |
| Rabin-Karp | O(m) | O(n+m) | O(n·m) | 다중 패턴 동시 탐색 |

※ σ: 알파벳 크기 (예: ASCII = 256)

## 실제 사용처

- **KMP**: `grep`, 스트리밍 데이터에서 패턴 탐지, 압축 알고리즘(LZ)
- **Boyer-Moore**: GNU `grep`의 기본 엔진, 텍스트 에디터의 검색 기능
- **Rabin-Karp**: 표절 검사(rolling window 유사도), IDS/방화벽 시그니처 매칭, Git의 delta 압축

## 주의사항과 팁

**1. KMP LPS 계산의 off-by-one 오류**
LPS 계산에서 `i`를 1부터 시작하는 것을 잊으면 안 된다. `lps[0]`은 항상 0이다(길이 1 접두사에는 진부분 접두사가 없음).

**2. Rabin-Karp의 해시 충돌**
해시가 같아도 반드시 실제 문자열 비교를 수행해야 한다. 큰 소수(mod)를 사용하면 충돌 확률을 1/10⁹ 이하로 줄일 수 있다. double hashing(두 개의 서로 다른 해시 함수 사용)으로 충돌을 거의 제거할 수 있다.

**3. Boyer-Moore의 역설**
패턴이 짧거나 알파벳이 작으면(예: DNA의 A/T/G/C 4종) Bad Character Rule의 효과가 줄어든다. 이 경우 KMP가 더 안정적이다.

**4. 유니코드 처리**
멀티바이트 문자(한국어 등)를 다룰 때는 바이트 단위가 아닌 코드포인트 단위로 처리해야 한다. Python의 `str`은 이미 유니코드 코드포인트 기반이므로 위 코드가 그대로 동작한다.

## 참고 자료
- [Knuth–Morris–Pratt algorithm - Wikipedia](https://en.wikipedia.org/wiki/Knuth%E2%80%93Morris%E2%80%93Pratt_algorithm)
- [Boyer–Moore string-search algorithm - Wikipedia](https://en.wikipedia.org/wiki/Boyer%E2%80%93Moore_string-search_algorithm)
- [Rabin–Karp algorithm - Wikipedia](https://en.wikipedia.org/wiki/Rabin%E2%80%93Karp_algorithm)
- [String Matching Algorithms - CS Auckland Lecture Notes](https://www.cs.auckland.ac.nz/courses/compsci369s1c/lectures/GG-notes/CS369-StringAlgs.pdf)
