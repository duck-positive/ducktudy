---
layout: post
title: "Z-알고리즘과 Manacher's 알고리즘 완전 정복: 문자열 처리를 O(N)으로 끝내는 선형 시간 비밀"
date: 2026-08-07
categories: [cs, computer-science]
tags: [알고리즘, 문자열, Z-알고리즘, 마나커, 팰린드롬, 패턴매칭, 선형시간]
---

## 개념 설명

문자열 처리는 컴퓨터 과학의 가장 고전적인 문제 영역 중 하나입니다. 텍스트 편집기의 "찾기" 기능, DNA 서열 분석, 침입 탐지 시스템, 검색 엔진의 쿼리 매칭 — 이 모든 것이 문자열 알고리즘 위에서 동작합니다. 이번 글에서는 두 가지 강력한 선형 시간 문자열 알고리즘인 **Z-알고리즘(Z-function)**과 **Manacher's 알고리즘**을 깊이 파고듭니다.

### Z-함수(Z-function)란?

문자열 `S`의 길이를 `n`이라 할 때, Z-배열 `Z[i]`는 다음을 의미합니다:

> `Z[i]` = `S`와 `S[i:]` (즉, i번째 위치부터 시작하는 접미사)의 **가장 긴 공통 접두사(Longest Common Prefix, LCP)**의 길이

예를 들어 `S = "aabxaab"` 이라면:
- `Z[0]` = 7 (전체 문자열과 자신의 LCP이므로 통상 7 또는 0으로 정의)
- `Z[1]` = 1 (`"abxaab"` vs `"aabxaab"` → 'a'까지 일치, LCP=1)
- `Z[4]` = 3 (`"aab"` vs `"aabxaab"` → 'aab' 일치, LCP=3)

Z-배열을 순진하게 계산하면 최악의 경우 O(n²)이 소요됩니다. Z-알고리즘은 이를 **O(n)**으로 줄입니다.

### Z-박스(Z-box) 최적화 원리

Z-알고리즘의 핵심 아이디어는 **Z-박스** `[l, r]`를 유지하는 것입니다. `[l, r]`은 지금까지 계산된 Z-박스 중 오른쪽 끝점 `r`이 가장 큰 구간을 의미하며, `S[l..r] = S[0..r-l]`이 성립합니다.

새로운 인덱스 `i`에서 Z[i]를 계산할 때:
1. `i > r`이면: `S[i]`부터 나이브하게 확장하고 새로운 Z-박스 갱신
2. `i <= r`이면: 미러 위치 `i' = i - l`에서 이미 계산된 `Z[i']`를 활용
   - `Z[i'] < r - i + 1`이면 `Z[i] = Z[i']` (박스 안에 완전히 포함)
   - 그 외이면 `r - i + 1`부터 확장 시작

이 과정에서 오른쪽 포인터 `r`은 단조증가하므로 전체 시간 복잡도는 O(n)입니다.

---

### Manacher's 알고리즘이란?

Manacher's 알고리즘은 Glenn Manacher가 1975년에 고안한 알고리즘으로, 문자열의 모든 팰린드롬 부분 문자열을 **O(n)** 에 찾아냅니다.

가장 긴 팰린드롬 부분 문자열 문제를 나이브하게 풀면 각 중심에서 확장을 시도하여 O(n²)이 됩니다. Manacher's 알고리즘은 Z-알고리즘과 유사한 **중심 재사용(center reuse)** 원리로 이를 O(n)으로 단축합니다.

#### 짝수/홀수 팰린드롬 통합

짝수 길이(`"abba"`)와 홀수 길이(`"aba"`) 팰린드롬을 통일하여 처리하기 위해 문자 사이에 특수 구분자 `#`를 삽입합니다:

```
"abba"  →  "#a#b#b#a#"
"aba"   →  "#a#b#a#"
```

변환된 문자열의 길이는 항상 2n+1이고 모든 팰린드롬은 홀수 길이가 됩니다.

#### P-배열과 중심 재사용

`P[i]` = 변환된 문자열에서 위치 `i`를 중심으로 하는 팰린드롬의 반지름

현재까지 알려진 **가장 오른쪽 팰린드롬**의 중심 `c`와 오른쪽 경계 `r`을 유지합니다. 새로운 위치 `i`에서:
- 미러 위치 `mirror = 2*c - i`에서 `P[mirror]`를 초기값으로 사용
- `P[i] = min(P[mirror], r - i)` (경계를 넘지 않는 선에서)
- 그 후 실제 확장 시도

---

## 왜 필요한가?

### KMP를 Z-알고리즘으로 대체하기

패턴 `P`가 텍스트 `T`에서 등장하는 모든 위치를 찾는 문제를 생각해 봅시다. KMP 알고리즘이 대표적인 O(n+m) 해법이지만, Z-알고리즘을 이용하면 더 직관적으로 같은 복잡도를 달성할 수 있습니다.

**트릭**: `P + "$" + T` 형태의 문자열 S를 만들어 Z-배열을 계산합니다. `"$"`는 P에도 T에도 등장하지 않는 구분자입니다. `Z[i] == len(P)`인 위치 `i`가 텍스트 T에서 P가 시작하는 위치입니다.

### 가장 긴 팰린드롬 부분 문자열 (LeetCode #5)

Manacher's 알고리즘의 핵심 응용입니다. 면접에서 자주 나오는 문제이며, O(n²) DP나 O(n²) 중심 확장보다 훨씬 효율적인 O(n) 해법을 제공합니다.

### DNA 서열 분석에서의 활용

Z-알고리즘은 생물정보학에서 DNA/RNA 서열의 모티프 탐색에 광범위하게 사용됩니다. 수백만 염기쌍 서열에서 특정 패턴을 선형 시간에 탐색하는 것은 임상적으로 매우 중요합니다.

---

## 실제 구현 예제

### 예제 1: Z-알고리즘을 이용한 패턴 탐색 (Python)

```python
def z_function(s: str) -> list[int]:
    n = len(s)
    z = [0] * n
    z[0] = n
    l, r = 0, 0

    for i in range(1, n):
        if i < r:
            z[i] = min(r - i, z[i - l])
        while i + z[i] < n and s[z[i]] == s[i + z[i]]:
            z[i] += 1
        if i + z[i] > r:
            l, r = i, i + z[i]

    return z


def find_pattern(text: str, pattern: str) -> list[int]:
    """패턴이 텍스트에서 등장하는 모든 시작 인덱스 반환 (O(n+m))"""
    if not pattern or not text:
        return []

    combined = pattern + "$" + text
    z = z_function(combined)
    p_len = len(pattern)
    result = []

    for i in range(p_len + 1, len(combined)):
        if z[i] == p_len:
            result.append(i - p_len - 1)

    return result


# 사용 예시
text = "aababcabababc"
pattern = "abab"
positions = find_pattern(text, pattern)
print(f"Pattern '{pattern}' found at positions: {positions}")
# Pattern 'abab' found at positions: [1, 6, 8]

# 검증
for pos in positions:
    assert text[pos:pos + len(pattern)] == pattern
print("All positions verified!")
```

이 구현에서 핵심은 `z[0] = n`으로 초기화한 뒤, `l`과 `r`로 Z-박스를 추적하는 것입니다. 내부 while 루프가 여러 번 반복되더라도 `r`은 단조증가하므로 총 반복 횟수가 O(n)을 넘지 않습니다.

---

### 예제 2: Manacher's 알고리즘으로 가장 긴 팰린드롬 찾기 (Python)

```python
def manacher(s: str) -> tuple[str, int, int]:
    """
    가장 긴 팰린드롬 부분 문자열과 원본 문자열에서의 시작/끝 인덱스 반환
    시간 복잡도: O(n), 공간 복잡도: O(n)
    """
    # 변환: "abc" -> "#a#b#c#"
    t = "#" + "#".join(s) + "#"
    n = len(t)
    p = [0] * n  # 각 위치의 팰린드롬 반지름
    c, r = 0, 0  # 현재 가장 오른쪽 팰린드롬의 중심과 오른쪽 경계

    for i in range(n):
        mirror = 2 * c - i
        if i < r:
            p[i] = min(r - i, p[mirror])

        # 실제 확장
        left, right = i - (p[i] + 1), i + (p[i] + 1)
        while left >= 0 and right < n and t[left] == t[right]:
            p[i] += 1
            left -= 1
            right += 1

        # Z-박스 갱신
        if i + p[i] > r:
            c, r = i, i + p[i]

    # 가장 긴 팰린드롬의 중심 찾기
    max_len_idx = p.index(max(p))
    max_radius = p[max_len_idx]

    # 원본 문자열에서의 인덱스 변환
    start = (max_len_idx - max_radius) // 2
    end = start + max_radius - 1

    return s[start:end + 1], start, end


def find_all_palindromes(s: str) -> list[tuple[int, int, str]]:
    """길이 2 이상인 모든 팰린드롬 부분 문자열 탐색 (중복 포함)"""
    t = "#" + "#".join(s) + "#"
    n = len(t)
    p = [0] * n
    c, r = 0, 0

    for i in range(n):
        mirror = 2 * c - i
        if i < r:
            p[i] = min(r - i, p[mirror])
        left, right = i - (p[i] + 1), i + (p[i] + 1)
        while left >= 0 and right < n and t[left] == t[right]:
            p[i] += 1
            left -= 1
            right += 1
        if i + p[i] > r:
            c, r = i, i + p[i]

    results = []
    for i, radius in enumerate(p):
        if radius == 0:
            continue
        start = (i - radius) // 2
        end = start + radius - 1
        substr = s[start:end + 1]
        if len(substr) >= 2:
            results.append((start, end, substr))

    return results


# 테스트
s = "babad"
palindrome, start, end = manacher(s)
print(f"가장 긴 팰린드롬: '{palindrome}' (인덱스 {start}~{end})")
# 가장 긴 팰린드롬: 'bab' (인덱스 0~2)

s2 = "racecar"
palindrome2, start2, end2 = manacher(s2)
print(f"가장 긴 팰린드롬: '{palindrome2}' (인덱스 {start2}~{end2})")
# 가장 긴 팰린드롬: 'racecar' (인덱스 0~6)

# 모든 팰린드롬 탐색
s3 = "abacaba"
all_pals = find_all_palindromes(s3)
print(f"'{s3}'의 모든 팰린드롬: {[(p[2], p[0], p[1]) for p in all_pals]}")
```

`t[i]`가 `#`이면 짝수 길이 팰린드롬의 중심, 원래 문자이면 홀수 길이 팰린드롬의 중심에 해당합니다. `p[i]`를 원본 인덱스로 변환할 때 `(i - p[i]) // 2`를 사용하는 이유는 변환 문자열에서 각 원본 문자가 두 칸씩 차지하기 때문입니다.

---

## 주의사항 및 팁

### Z-알고리즘 구현 시 주의점

1. **Z[0] 처리**: Z[0]은 문자열 전체와의 LCP이므로 n으로 초기화하거나, 문제에 따라 0으로 처리합니다. 패턴 탐색 응용에서는 `Z[0] = n`으로 두면 `P+"$"+T` 방식에서 패턴 위치와 혼동이 생기므로 주의하세요.

2. **구분자 선택**: `P+"$"+T` 트릭에서 구분자 `"$"`는 반드시 P와 T에 등장하지 않아야 합니다. 알파벳 문자열이면 `"$"`나 `"#"`, 숫자 배열이면 `-1` 같은 값을 사용합니다.

3. **경계 조건**: `l = r = 0`으로 초기화할 때 첫 번째 Z-박스가 비어 있음을 명심하세요. `i < r` 조건이 `i <= r`이 아니어야 무한 루프를 피할 수 있습니다.

### Manacher's 알고리즘 구현 시 주의점

1. **변환 문자열의 길이**: 원본 문자열 길이가 n이면 변환 후 2n+1입니다. 배열 인덱스를 혼동하지 않도록 변환 함수를 별도로 분리하는 것이 좋습니다.

2. **원본 인덱스 역변환**: 변환 문자열의 인덱스 `i`와 반지름 `p[i]`로부터 원본 인덱스를 구하는 공식은 `start = (i - p[i]) // 2`, `length = p[i]`입니다. 이를 직접 도출해 보고 몇 가지 예시로 검증해 두면 실수를 예방할 수 있습니다.

3. **문자열 전체가 팰린드롬인 경우**: `p[i] = n` (n은 원본 길이)일 때 Z-박스가 문자열 전체를 커버하므로, 이 케이스를 명시적으로 테스트하세요.

### 성능 비교

| 알고리즘 | 시간 복잡도 | 공간 복잡도 | 특징 |
|---------|-----------|-----------|------|
| 나이브 패턴 탐색 | O(nm) | O(1) | 구현 단순 |
| KMP | O(n+m) | O(m) | 실패 함수 사전 계산 |
| Z-알고리즘 | O(n+m) | O(n+m) | Z-배열 기반 |
| 나이브 팰린드롬 | O(n²) | O(1) | — |
| DP 팰린드롬 | O(n²) | O(n²) | 부분 문제 저장 |
| Manacher's | O(n) | O(n) | 선형 시간 |

Z-알고리즘과 Manacher's 알고리즘은 모두 "이미 계산한 정보를 재사용하여 중복 비교를 제거"한다는 동일한 패러다임 위에 있습니다. 두 알고리즘을 이해하면 접미사 배열, Aho-Corasick 같은 고급 문자열 자료구조를 이해하는 데도 큰 도움이 됩니다.

## 참고 자료
- [Z-function - Algorithms for Competitive Programming](https://cp-algorithms.com/string/z-function.html)
- [Manacher's Algorithm - Algorithms for Competitive Programming](https://cp-algorithms.com/string/manacher.html)
- [Z algorithm (Linear time pattern searching) - GeeksforGeeks](https://www.geeksforgeeks.org/dsa/z-algorithm-linear-time-pattern-searching-algorithm/)
- [Manacher's Algorithm - GeeksforGeeks](https://www.geeksforgeeks.org/dsa/manachers-algorithm-linear-time-longest-palindromic-substring-part-1/)
