---
layout: post
title: "폴리노미얼 롤링 해시 완전 정복: 충돌 없는 고속 문자열 비교와 이중 해싱 전략"
date: 2026-08-29
categories: [cs, computer-science]
tags: [algorithms, string, hashing, rolling-hash, rabin-karp, competitive-programming]
---

문자열 처리에서 가장 기본적인 연산은 "두 문자열이 같은가?"이다. 길이 N인 문자열 두 개를 직접 비교하면 O(N)이 걸린다. 1000개의 문자열 쌍을 비교하면 O(1000N)이 된다. 그런데 **폴리노미얼 롤링 해시(Polynomial Rolling Hash)**를 사용하면 전처리 O(N) 후 임의 부분 문자열의 해시를 O(1)에 계산해 비교할 수 있다.

더 강력한 점은 **슬라이딩 윈도우**에서 해시를 O(1)에 업데이트할 수 있다는 것이다. 길이 K인 패턴을 길이 N인 텍스트에서 찾는 Rabin-Karp 알고리즘이 바로 이 성질을 활용해 O(N + M)의 평균 복잡도를 달성한다. 이 기법은 경쟁 프로그래밍의 팰린드롬 탐색, 최장 공통 부분 문자열(LCS), 문자열 해시 집합 관리 등 수많은 문제에 필수적으로 사용된다.

---

## 폴리노미얼 해시의 수학적 정의

문자열 S = s₀s₁…s_{n-1}의 해시를 다음과 같이 정의한다.

```
H(S) = (s₀ · p⁰ + s₁ · p¹ + s₂ · p² + … + s_{n-1} · p^{n-1}) mod M
```

여기서:
- **p**: 기저(base). 문자 집합 크기보다 큰 소수를 사용. 소문자 알파벳이면 31 또는 37이 일반적
- **M**: 모듈러. 충돌 확률을 낮추려면 큰 소수 사용. 10⁹+7이나 10⁹+9가 흔함
- **s_i**: 문자의 숫자 값. 'a' = 1, 'b' = 2, …, 'z' = 26 (또는 ASCII 코드)

이 정의에서 부분 문자열 S[l..r]의 해시를 프리픽스 해시 배열로 O(1)에 계산할 수 있다.

---

## 기본 구현: 프리픽스 해시와 부분 문자열 해시

```python
class StringHash:
    """
    폴리노미얼 롤링 해시 (단일 모듈러 버전)
    s[l..r] (0-indexed, 양 끝 포함)의 해시를 O(1)에 반환
    """
    BASE = 131          # 127보다 큰 소수 (ASCII 범위 커버)
    MOD  = 10**9 + 7
    
    def __init__(self, s: str):
        n = len(s)
        self.h = [0] * (n + 1)  # prefix hash: h[i] = H(s[0..i-1])
        self.pw = [1] * (n + 1) # pw[i] = BASE^i mod MOD
        
        for i in range(n):
            self.h[i+1] = (self.h[i] * self.BASE + ord(s[i])) % self.MOD
            self.pw[i+1] = self.pw[i] * self.BASE % self.MOD
    
    def get(self, l: int, r: int) -> int:
        """s[l..r]의 해시 반환 (0-indexed, 양 끝 포함)"""
        # h[r+1] - h[l] * BASE^(r-l+1)
        return (self.h[r+1] - self.h[l] * self.pw[r-l+1]) % self.MOD

# 사용 예시
s = "abcabcabc"
sh = StringHash(s)

print(sh.get(0, 2))  # "abc"의 해시
print(sh.get(3, 5))  # "abc"의 해시 (같아야 함)
print(sh.get(6, 8))  # "abc"의 해시 (같아야 함)
print(sh.get(0, 2) == sh.get(3, 5) == sh.get(6, 8))  # True

# 패턴 매칭: "bc"가 s에서 등장하는 모든 위치 찾기
pattern = "bc"
ph = StringHash(pattern)
pat_hash = ph.get(0, len(pattern) - 1)
pat_len = len(pattern)

positions = []
for i in range(len(s) - pat_len + 1):
    if sh.get(i, i + pat_len - 1) == pat_hash:
        positions.append(i)
        
print(f"'bc' 등장 위치: {positions}")  # [1, 4, 7]
```

---

## 충돌 문제와 이중 해싱(Double Hashing)

단일 해시는 **해시 충돌(hash collision)** 문제가 있다. 서로 다른 두 문자열이 같은 해시 값을 가질 수 있다. 단일 소수 M을 사용하면 충돌 확률은 약 1/M ≈ 10⁻⁹이지만, 경쟁 프로그래밍에서는 해커가 의도적으로 충돌을 유발하는 입력(안티-해시 테스트)을 만들 수 있다.

**이중 해싱**은 두 개의 독립적인 (BASE, MOD) 쌍을 사용해 충돌 확률을 M₁ × M₂ ≈ 10¹⁸ 분의 1로 줄인다.

```python
class DoubleHash:
    """
    이중 해싱: 두 독립 해시를 tuple로 관리
    해시 충돌 확률 ≈ 1/(10^9 * 10^9) = 10^{-18}
    """
    BASE1, MOD1 = 131, 10**9 + 7
    BASE2, MOD2 = 137, 10**9 + 9
    
    def __init__(self, s: str):
        n = len(s)
        self.h1 = [0] * (n + 1)
        self.h2 = [0] * (n + 1)
        self.pw1 = [1] * (n + 1)
        self.pw2 = [1] * (n + 1)
        
        for i in range(n):
            c = ord(s[i])
            self.h1[i+1] = (self.h1[i] * self.BASE1 + c) % self.MOD1
            self.h2[i+1] = (self.h2[i] * self.BASE2 + c) % self.MOD2
            self.pw1[i+1] = self.pw1[i] * self.BASE1 % self.MOD1
            self.pw2[i+1] = self.pw2[i] * self.BASE2 % self.MOD2
    
    def get(self, l: int, r: int) -> tuple:
        """(hash1, hash2) 튜플 반환"""
        h1 = (self.h1[r+1] - self.h1[l] * self.pw1[r-l+1]) % self.MOD1
        h2 = (self.h2[r+1] - self.h2[l] * self.pw2[r-l+1]) % self.MOD2
        return (h1, h2)
    
    def equal(self, l1: int, r1: int, l2: int, r2: int) -> bool:
        """s[l1..r1] == s[l2..r2] 여부 확인 (길이가 같아야 함)"""
        if r1 - l1 != r2 - l2:
            return False
        return self.get(l1, r1) == self.get(l2, r2)

# 활용: 문자열 내에서 가장 긴 반복 부분 문자열 찾기
def longest_repeated_substring(s: str) -> str:
    """이분 탐색 + 해싱으로 O(N log N)에 가장 긴 반복 부분 문자열 탐색"""
    n = len(s)
    dh = DoubleHash(s)
    
    def has_repeat_of_length(length: int) -> int:
        """길이 length인 반복 부분 문자열의 시작 위치 반환, 없으면 -1"""
        seen = {}
        for i in range(n - length + 1):
            h = dh.get(i, i + length - 1)
            if h in seen:
                # 실제 문자열도 비교 (이중 해싱이라 거의 불필요하지만 안전)
                prev = seen[h]
                if s[prev:prev+length] == s[i:i+length]:
                    return i
            seen[h] = i
        return -1
    
    # 이분 탐색: 가능한 최대 길이 탐색
    lo, hi = 0, n - 1
    result_pos, result_len = -1, 0
    
    while lo <= hi:
        mid = (lo + hi) // 2
        pos = has_repeat_of_length(mid)
        if pos != -1:
            result_pos, result_len = pos, mid
            lo = mid + 1
        else:
            hi = mid - 1
    
    if result_pos == -1:
        return ""
    return s[result_pos:result_pos + result_len]

# 테스트
test_cases = [
    "banana",       # "ana"
    "abcabcabc",    # "abcabc"
    "aabcaab",      # "aab"
    "aaaa",         # "aaa"
]
for s in test_cases:
    result = longest_repeated_substring(s)
    print(f"'{s}' → 가장 긴 반복: '{result}'")
```

---

## Rabin-Karp 알고리즘: 슬라이딩 윈도우 해시

Rabin-Karp는 패턴을 해시로 변환한 뒤, 텍스트에서 윈도우를 한 칸씩 이동하며 해시를 O(1)에 업데이트해 비교하는 알고리즘이다. 윈도우 이동 시 해시 업데이트 공식은 다음과 같다.

```
H(s[i+1..i+m]) = (H(s[i..i+m-1]) - s[i] * p^{m-1}) * p + s[i+m]
```

```python
def rabin_karp(text: str, pattern: str) -> list[int]:
    """
    Rabin-Karp 다중 패턴 호환 구현
    Returns: 패턴이 시작되는 0-indexed 위치 목록
    """
    n, m = len(text), len(pattern)
    if m > n:
        return []
    
    BASE, MOD = 131, 10**9 + 7
    
    # 패턴 해시와 첫 윈도우 해시 계산
    pat_hash = 0
    win_hash = 0
    leading = pow(BASE, m - 1, MOD)  # BASE^(m-1) mod MOD
    
    for i in range(m):
        pat_hash = (pat_hash * BASE + ord(pattern[i])) % MOD
        win_hash = (win_hash * BASE + ord(text[i])) % MOD
    
    results = []
    
    for i in range(n - m + 1):
        if win_hash == pat_hash:
            # 해시 충돌 방지: 실제 문자열 비교
            if text[i:i+m] == pattern:
                results.append(i)
        
        # 윈도우 슬라이드
        if i < n - m:
            win_hash = (
                (win_hash - ord(text[i]) * leading) * BASE
                + ord(text[i + m])
            ) % MOD
    
    return results

# 테스트
text    = "ababcababcababc"
pattern = "ababc"
positions = rabin_karp(text, pattern)
print(f"패턴 '{pattern}' 위치: {positions}")  # [0, 5, 10]

# 성능 비교 (Python naive vs Rabin-Karp)
import time

big_text    = "a" * 100_000 + "b"
big_pattern = "a" * 1000 + "b"

start = time.time()
naive_result = [i for i in range(len(big_text) - len(big_pattern) + 1)
                if big_text[i:i+len(big_pattern)] == big_pattern]
print(f"Naive: {len(naive_result)}개 매치, {time.time()-start:.3f}s")

start = time.time()
rk_result = rabin_karp(big_text, big_pattern)
print(f"Rabin-Karp: {len(rk_result)}개 매치, {time.time()-start:.3f}s")
```

---

## 고급 활용: 최장 공통 부분 문자열 (LCS)

두 문자열 A와 B의 최장 공통 부분 문자열(Longest Common Substring) 문제를 이분 탐색 + 해싱으로 O((N+M) log(N+M))에 해결한다.

```python
def longest_common_substring(a: str, b: str) -> str:
    """
    두 문자열의 최장 공통 부분 문자열
    이분 탐색 + 이중 해싱: O((N+M) log(N+M))
    """
    BASE1, MOD1 = 131, 10**9 + 7
    BASE2, MOD2 = 137, 10**9 + 9
    
    ha = DoubleHash(a)
    hb = DoubleHash(b)
    
    def has_common_of_length(length: int):
        if length == 0:
            return 0, 0  # (위치a, 위치b)
        
        # A의 모든 길이 length 부분 문자열 해시를 Set에
        hash_set = {}
        for i in range(len(a) - length + 1):
            h = ha.get(i, i + length - 1)
            if h not in hash_set:
                hash_set[h] = i
        
        # B에서 매칭 탐색
        for j in range(len(b) - length + 1):
            h = hb.get(j, j + length - 1)
            if h in hash_set:
                i = hash_set[h]
                # 실제 비교로 충돌 배제
                if a[i:i+length] == b[j:j+length]:
                    return i, j
        
        return -1, -1
    
    lo, hi = 0, min(len(a), len(b))
    best_len, best_pos_a = 0, 0
    
    while lo <= hi:
        mid = (lo + hi) // 2
        ia, ib = has_common_of_length(mid)
        if ia != -1:
            best_len = mid
            best_pos_a = ia
            lo = mid + 1
        else:
            hi = mid - 1
    
    return a[best_pos_a:best_pos_a + best_len]

# 테스트
a = "OldSite:GeeksforGeeks.org"
b = "NewSite:GeeksQuiz.com"
print(f"LCS: '{longest_common_substring(a, b)}'")  # "Site:Geeks"

a2 = "abcde"
b2 = "abfcde"
print(f"LCS: '{longest_common_substring(a2, b2)}'")  # "cde"
```

---

## BASE와 MOD 선택 전략

해시 충돌 확률과 성능은 BASE와 MOD 선택에 크게 의존한다.

**BASE 선택 원칙:**
- 문자 집합 크기보다 커야 한다. 소문자 영문자만 사용하면 26보다 큰 소수(31, 37), ASCII 전체면 128보다 큰 소수(131, 137, 257)
- 소수여야 충돌이 적다
- 너무 크면 중간 계산 값이 오버플로우 위험

**MOD 선택 원칙:**
- 충분히 큰 소수. 10⁹+7, 10⁹+9, 10¹⁸+9 등
- 이중 해싱 시 두 MOD는 서로 다른 소수를 선택
- 64비트 정수 범위 내에서 작동해야 함

**안티-해시 방어:**
일부 경쟁 문제에서는 고정 BASE/MOD에 대한 충돌 케이스를 의도적으로 만든다. 방어책으로는:
1. **이중 해싱**: 두 해시가 동시에 충돌할 확률은 극히 낮다
2. **랜덤 BASE**: 실행 시마다 BASE를 무작위로 선택한다

```python
import random

class SecureHash:
    """실행마다 다른 BASE를 사용해 안티-해시 공격 방어"""
    def __init__(self, s: str):
        self.MOD = 10**9 + 7
        # 실행마다 다른 BASE (충분히 큰 소수 범위에서 랜덤 선택)
        self.BASE = random.randint(10**8, 10**9 - 1) | 1  # 홀수로 설정
        
        n = len(s)
        self.h = [0] * (n + 1)
        self.pw = [1] * (n + 1)
        for i in range(n):
            self.h[i+1] = (self.h[i] * self.BASE + ord(s[i])) % self.MOD
            self.pw[i+1] = self.pw[i] * self.BASE % self.MOD
    
    def get(self, l, r):
        return (self.h[r+1] - self.h[l] * self.pw[r-l+1]) % self.MOD
```

---

## 주의사항과 팁

**1. 음수 모듈러 처리**  
Python에서는 문제없지만 C++에서 `(h[r+1] - h[l] * pw[r-l+1] % MOD)` 는 음수가 될 수 있다. 반드시 `(... % MOD + MOD) % MOD`로 보정해야 한다.

**2. 해시 후 반드시 실제 비교**  
단일 해싱에서 충돌이 실제로 발생하면 오답이 난다. 해시 매칭 후 문자열 직접 비교를 추가하면 완전한 정확성을 보장할 수 있다. 이중 해싱이면 충돌 확률이 극히 낮으므로 생략 가능하다.

**3. 전처리 배열의 크기**  
길이 N 문자열의 프리픽스 해시 배열은 N+1 크기가 필요하다. `h[0] = 0`, `h[i]`는 s[0..i-1]의 해시임을 헷갈리지 말 것.

**4. 슬라이딩 윈도우에서의 정확한 공식**  
부분 문자열 해시 공식 `H(s[l..r]) = (h[r+1] - h[l] * pw[r-l+1]) % MOD`에서 `pw[r-l+1]`이 곱해지는 이유: `h[l]`은 s[0..l-1]의 정보를 p^(r-l+1) 자리만큼 앞에 위치한 것처럼 스케일링해야 h[r+1]에서 빼면 s[l..r]만 남기 때문이다.

**5. 팰린드롬 검사 응용**  
문자열 S와 S의 역순 S_rev에 대해 프리픽스 해시를 모두 구해두면, S[l..r]이 팰린드롬인지를 O(1)에 확인할 수 있다. `H(S[l..r]) == H(S_rev[n-1-r..n-1-l])` 조건을 확인하면 된다. 이를 이용하면 Manacher's Algorithm 없이도 최장 팰린드롬 부분 문자열을 이분 탐색 + 해싱으로 O(N log N)에 풀 수 있다.

---

## 참고 자료
- [CP-Algorithms: String Hashing](https://cp-algorithms.com/string/string-hashing.html)
- [CP-Algorithms: Rabin-Karp Algorithm](https://cp-algorithms.com/string/rabin-karp.html)
- [Codeforces Blog: Anti-Hash Test](https://codeforces.com/blog/entry/4898)
- [Wikipedia: Rabin-Karp Algorithm](https://en.wikipedia.org/wiki/Rabin%E2%80%93Karp_algorithm)
