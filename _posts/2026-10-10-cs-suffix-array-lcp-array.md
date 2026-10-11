---
layout: post
title: "서픽스 배열(Suffix Array)과 LCP 배열 완전 정복: 문자열 처리의 최강 도구"
date: 2026-10-10
categories: [cs, computer-science]
tags: [suffix-array, lcp-array, string-algorithm, kasai-algorithm, data-structure]
---

문자열 처리는 컴퓨터 과학에서 가장 오래되고 중요한 분야 중 하나입니다. 텍스트 검색, 생물정보학의 DNA 서열 분석, 데이터 압축까지 수많은 응용이 문자열을 효율적으로 처리하는 기술에 의존합니다. 오늘은 그 핵심에 있는 **서픽스 배열(Suffix Array)**과 **LCP 배열(Longest Common Prefix Array)**을 깊이 파헤칩니다.

## 개념 설명

### 서픽스(Suffix)란?

문자열 `S = "banana"`가 있을 때, `S`의 서픽스는 `S`의 특정 위치 `i`부터 끝까지의 부분 문자열입니다.

```
i=0: "banana"
i=1: "anana"
i=2: "nana"
i=3: "ana"
i=4: "na"
i=5: "a"
```

### 서픽스 배열이란?

**서픽스 배열(SA, Suffix Array)**은 문자열의 모든 서픽스를 사전순으로 정렬했을 때, 각 서픽스의 **시작 인덱스**를 저장한 배열입니다. `"banana"` 예시를 보면:

```
정렬된 서픽스:
0: "a"        → i=5
1: "ana"      → i=3
2: "anana"    → i=1
3: "banana"   → i=0
4: "na"       → i=4
5: "nana"     → i=2

SA = [5, 3, 1, 0, 4, 2]
```

단순히 서픽스 문자열을 생성해 정렬하면 O(n² log n) 시간이 걸리지만, **접두사 배가(Prefix Doubling)** 기법을 사용하면 O(n log n), SA-IS 알고리즘을 사용하면 O(n)에 구성 가능합니다.

### LCP 배열이란?

**LCP 배열(Longest Common Prefix Array)**은 서픽스 배열에서 **인접한 두 서픽스의 가장 긴 공통 접두사 길이**를 저장합니다.

```
SA = [5, 3, 1, 0, 4, 2]
정렬된 서픽스:
"a", "ana", "anana", "banana", "na", "nana"

LCP[0] = 0  (정의에 따라 첫 번째는 0)
LCP[1] = 1  ("a"와 "ana"의 공통 접두사: "a" → 길이 1)
LCP[2] = 3  ("ana"와 "anana"의 공통 접두사: "ana" → 길이 3)
LCP[3] = 0  ("anana"와 "banana"의 공통 접두사: 없음 → 길이 0)
LCP[4] = 0  ("banana"와 "na"의 공통 접두사: 없음 → 길이 0)
LCP[5] = 2  ("na"와 "nana"의 공통 접두사: "na" → 길이 2)
```

---

## 왜 필요한가?

### 1. 패턴 검색을 O(m log n)에 수행

나이브한 방법으로 길이 n의 텍스트에서 길이 m의 패턴을 찾으면 O(nm)이 걸립니다. 서픽스 배열을 이용한 이분 탐색으로 **O(m log n)**에 처리할 수 있습니다.

### 2. 가장 긴 반복 부분 문자열 탐색

LCP 배열의 최댓값이 가장 긴 반복 부분 문자열의 길이입니다. O(n)에 탐색 가능합니다.

### 3. 가장 긴 공통 부분 문자열 탐색

두 문자열을 특수 구분자로 이어 붙인 뒤 서픽스 배열과 LCP 배열을 구성하면, 두 문자열 사이의 가장 긴 공통 부분 문자열을 O(n)에 찾을 수 있습니다.

### 4. 데이터 압축 (BWT, LZ)

BWT(Burrows-Wheeler Transform)은 서픽스 배열을 이용해 구성되며, 반복 패턴이 많은 문자열로 변환해 압축률을 높입니다. 실제로 gzip, bzip2 등의 압축 도구가 이 원리를 사용합니다.

### 5. 생물정보학

유전체 서열 분석에서 DNA나 RNA 서열의 패턴을 빠르게 검색하는 핵심 도구입니다. 수 기가바이트짜리 유전체 데이터를 처리할 때 서픽스 배열 없이는 현실적인 분석이 불가능합니다.

---

## 실제 구현 예제

### 예제 1: 서픽스 배열 구성 (접두사 배가 O(n log n))

```python
def build_suffix_array(s):
    """접두사 배가(Prefix Doubling) 기법으로 서픽스 배열 구성 O(n log²n)"""
    n = len(s)
    # 초기 랭크: 각 문자의 ASCII 코드
    sa = list(range(n))
    rank = [ord(c) for c in s]
    tmp = [0] * n
    
    gap = 1
    while gap < n:
        # 현재 gap 기준으로 (rank[i], rank[i+gap]) 쌍으로 정렬
        def sort_key(i):
            r1 = rank[i]
            r2 = rank[i + gap] if i + gap < n else -1
            return (r1, r2)
        
        sa.sort(key=sort_key)
        
        # 새로운 랭크 부여
        tmp[sa[0]] = 0
        for i in range(1, n):
            prev, curr = sa[i-1], sa[i]
            same = (rank[prev] == rank[curr] and
                    (rank[prev+gap] if prev+gap < n else -1) ==
                    (rank[curr+gap] if curr+gap < n else -1))
            tmp[curr] = tmp[prev] if same else tmp[prev] + 1
        
        rank = tmp[:]
        if rank[sa[-1]] == n - 1:
            break  # 모든 랭크가 유일해지면 종료
        gap *= 2
    
    return sa


def test_suffix_array():
    s = "banana"
    sa = build_suffix_array(s)
    print(f"문자열: {s}")
    print(f"서픽스 배열: {sa}")
    print("정렬된 서픽스:")
    for rank, i in enumerate(sa):
        print(f"  [{rank}] SA[{rank}]={i}: \"{s[i:]}\"")


test_suffix_array()
```

출력:
```
문자열: banana
서픽스 배열: [5, 3, 1, 0, 4, 2]
정렬된 서픽스:
  [0] SA[0]=5: "a"
  [1] SA[1]=3: "ana"
  [2] SA[2]=1: "anana"
  [3] SA[3]=0: "banana"
  [4] SA[4]=4: "na"
  [5] SA[5]=2: "nana"
```

---

### 예제 2: Kasai 알고리즘으로 LCP 배열 O(n) 구성 및 활용

```python
def build_lcp_array(s, sa):
    """Kasai 알고리즘: O(n)에 LCP 배열 구성"""
    n = len(s)
    rank = [0] * n
    for i, suffix_start in enumerate(sa):
        rank[suffix_start] = i  # 역배열 (서픽스 시작위치 → SA 인덱스)
    
    lcp = [0] * n
    h = 0  # 현재 LCP 길이 (h는 최대 1씩만 감소 → 총 O(n))
    
    for i in range(n):
        if rank[i] > 0:
            j = sa[rank[i] - 1]  # SA에서 바로 앞에 있는 서픽스
            while i + h < n and j + h < n and s[i + h] == s[j + h]:
                h += 1
            lcp[rank[i]] = h
            if h > 0:
                h -= 1  # 핵심: h는 매 스텝 최대 1 감소
    
    return lcp


def longest_repeated_substring(s):
    """서픽스 배열 + LCP 배열로 가장 긴 반복 부분문자열 탐색"""
    sa = build_suffix_array(s)
    lcp = build_lcp_array(s, sa)
    
    max_lcp = max(lcp)
    if max_lcp == 0:
        return ""
    
    max_idx = lcp.index(max_lcp)
    return s[sa[max_idx]:sa[max_idx] + max_lcp]


def pattern_search(text, pattern):
    """서픽스 배열을 이용한 이분 탐색 패턴 검색 O(m log n)"""
    sa = build_suffix_array(text)
    n, m = len(text), len(pattern)
    
    # 하한(lower bound): pattern <= suffix[sa[mid]]
    lo, hi = 0, n
    while lo < hi:
        mid = (lo + hi) // 2
        if text[sa[mid]:sa[mid]+m] < pattern:
            lo = mid + 1
        else:
            hi = mid
    start = lo
    
    # 상한(upper bound): suffix[sa[mid]] starts with pattern
    lo, hi = 0, n
    while lo < hi:
        mid = (lo + hi) // 2
        if text[sa[mid]:sa[mid]+m] <= pattern:
            lo = mid + 1
        else:
            hi = mid
    end = lo
    
    # [start, end) 범위의 SA 값이 패턴 등장 위치
    return sorted(sa[start:end])


# 테스트
s = "abracadabra"
sa = build_suffix_array(s)
lcp = build_lcp_array(s, sa)

print(f"문자열: {s}")
print(f"SA:  {sa}")
print(f"LCP: {lcp}")
print(f"가장 긴 반복 부분문자열: \"{longest_repeated_substring(s)}\"")

positions = pattern_search(s, "abr")
print(f"패턴 'abr' 등장 위치: {positions}")
```

출력:
```
문자열: abracadabra
SA:  [10, 7, 0, 3, 5, 8, 1, 4, 6, 9, 2]
LCP: [0, 1, 4, 1, 1, 0, 3, 0, 0, 0, 2]
가장 긴 반복 부분문자열: "abra"
패턴 'abr' 등장 위치: [0, 7]
```

---

## 주의사항 및 팁

### 1. 문자열 끝에 구분자 추가

서픽스 배열을 구성할 때 문자열 끝에 `$`와 같이 알파벳에 포함되지 않는 **최솟값 구분자**를 추가하면 두 서픽스가 완전히 동일해지는 상황을 방지할 수 있습니다. 이는 SA-IS처럼 선형 시간 알고리즘에서 특히 중요합니다.

### 2. Kasai 알고리즘의 핵심 불변식

Kasai 알고리즘이 O(n)인 이유는 `h` 값이 매 스텝에서 최대 1씩 감소하기 때문입니다. `h`는 증가하거나 1 감소하므로 전체 증가량이 최대 n, 전체 감소량도 최대 n이 되어 총 O(n)이 보장됩니다.

### 3. 두 서픽스의 임의 LCP 계산

LCP 배열과 **구간 최솟값 쿼리(RMQ)** 자료구조를 결합하면, 임의의 두 서픽스 `SA[i]`와 `SA[j]` (`i < j`)의 LCP를 `min(LCP[i+1], ..., LCP[j])`로 O(1)에 계산할 수 있습니다. Sparse Table을 사용하면 전처리 O(n log n), 쿼리 O(1)입니다.

### 4. 두 문자열의 가장 긴 공통 부분문자열

두 문자열 `A`와 `B`를 `A + '#' + B`로 이어 붙인 후(단, `#`은 A에도 B에도 없는 구분자), 서픽스 배열과 LCP 배열을 구성합니다. 이후 LCP 배열을 순회하며, **SA 인덱스가 A 구간과 B 구간에 각각 속하는 인접 쌍** 중 최대 LCP값을 구하면 됩니다.

### 5. 성능 비교

| 알고리즘 | 시간 복잡도 | 공간 복잡도 |
|---------|-----------|-----------|
| 나이브 정렬 | O(n² log n) | O(n²) |
| 접두사 배가 | O(n log² n) | O(n) |
| DC3 / Skew | O(n) | O(n) |
| SA-IS | O(n) | O(n) |
| Kasai (LCP) | O(n) | O(n) |

실무에서는 구현 복잡도와 성능의 균형을 고려해 **접두사 배가 기법**이 많이 쓰이며, 경쟁 프로그래밍에서는 SA-IS나 DC3를 구현 라이브러리로 활용합니다.

---

## 참고 자료

- [CP-Algorithms: Suffix Array](https://cp-algorithms.com/string/suffix-array.html)
- [Wikipedia: Suffix Array](https://en.wikipedia.org/wiki/Suffix_array)
- [Wikipedia: LCP Array](https://en.wikipedia.org/wiki/LCP_array)
- [Stanford CS166 강의자료: LCP Array 및 RMQ](https://web.stanford.edu/class/archive/cs/cs166/cs166.1226/lectures/13/Small13.pdf)
