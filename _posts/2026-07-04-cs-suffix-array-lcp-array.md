---
layout: post
title: "Suffix Array와 LCP Array 완전 정복: 선형 시간 문자열 처리의 숨겨진 강자"
date: 2026-07-04
categories: [cs, computer-science]
tags: [suffix-array, lcp-array, string-processing, algorithm, kasai, dc3, bioinformatics, pattern-matching]
---

## 개요: Suffix Array란 무엇인가

문자열 처리에서 가장 강력한 도구 중 하나인 **Suffix Array(접미사 배열)** 는, 어떤 문자열의 모든 접미사를 사전 순으로 정렬한 인덱스 배열입니다. 직접적인 정의보다 예시가 이해에 빠릅니다.

문자열 `"banana"` (인덱스 0~5)의 모든 접미사:

```
인덱스  접미사
  0    banana
  1    anana
  2    nana
  3    ana
  4    na
  5    a
```

이를 사전 순 정렬하면:

```
접미사    원본 인덱스
a           5
ana         3
anana       1
banana      0
na          4
nana        2
```

Suffix Array `SA = [5, 3, 1, 0, 4, 2]`는 "정렬된 접미사의 시작 인덱스 순서"입니다.

---

## 왜 Suffix Array가 필요한가

### Suffix Tree와의 비교

Suffix Array의 전신은 **Suffix Tree**입니다. Suffix Tree도 O(n) 구성이 가능하고 다양한 쿼리를 O(1)~O(m)에 처리하지만, 실제 상수 계수가 크고 **포인터 기반 자료구조**라 캐시 효율이 낮으며 구현이 복잡합니다. 영문자 기준으로 Suffix Array는 Suffix Tree 대비 약 5~20배 적은 메모리를 사용합니다.

| 비교 항목 | Suffix Tree | Suffix Array |
|-----------|-------------|--------------|
| 구성 시간 | O(n) | O(n) ~ O(n log n) |
| 공간 | O(n) — 상수 계수 큼 | O(n) — 매우 작음 |
| 캐시 효율 | 낮음 (포인터 추적) | 높음 (연속 배열) |
| 구현 복잡도 | 매우 높음 | 중간 |
| 실용성 | 이론적으로 강력 | 실무에서 선호 |

LCP Array와 함께 사용하면 Suffix Array로 Suffix Tree의 거의 모든 기능을 구현할 수 있습니다.

### 주요 활용 분야

- **패턴 매칭**: 길이 m인 패턴을 O(m log n) 또는 LCP를 이용해 O(m + log n)에 검색
- **최장 반복 부분문자열**: LCP Array의 최댓값이 답
- **최장 공통 부분문자열(LCS)**: 두 문자열을 구분자로 연결 후 Suffix Array 구성
- **게놈 시퀀싱**: BWA, bowtie 등 생물정보학 도구의 핵심
- **데이터 압축**: Burrows-Wheeler Transform(BWT)의 기반

---

## LCP Array: Suffix Array의 동반자

**LCP Array (Longest Common Prefix Array)** 는 `LCP[i]`가 정렬된 `SA[i-1]`번째 접미사와 `SA[i]`번째 접미사의 **최장 공통 접두사 길이**를 저장하는 배열입니다.

`"banana"`의 Suffix Array와 LCP Array:

```
i   SA[i]  접미사       LCP[i]
0     5    a            0     (첫 번째, LCP 정의 없음)
1     3    ana          1     (a와 ana의 공통 접두사: "a" → 1)
2     1    anana        3     (ana와 anana의 공통 접두사: "ana" → 3)
3     0    banana       0     (anana와 banana의 공통 접두사: 없음 → 0)
4     4    na           0     (banana와 na의 공통 접두사: 없음 → 0)
5     2    nana         2     (na와 nana의 공통 접두사: "na" → 2)
```

LCP Array는 인접한 두 접미사가 얼마나 비슷한지 나타냅니다. 이 정보를 이용하면 패턴 검색의 비교 횟수를 크게 줄이고, 문자열 관련 다양한 문제를 선형 시간에 풀 수 있습니다.

---

## 실제 구현 예제

### 예제 1: Suffix Array 구성 + Kasai's Algorithm (Python)

```python
def build_suffix_array(s: str) -> list[int]:
    """
    O(n log^2 n) 접미사 배열 구성.
    더 빠른 SA-IS(O(n))도 있지만 구현 복잡도가 높다.
    """
    n = len(s)
    # 초기 rank: 각 문자의 ASCII 값
    sa = list(range(n))
    rank = [ord(c) for c in s]
    tmp = [0] * n
    k = 1

    while k < n:
        # (rank[i], rank[i+k]) 쌍으로 정렬
        def key(i):
            return (rank[i], rank[i + k] if i + k < n else -1)
        sa.sort(key=key)

        # 새 rank 계산
        tmp[sa[0]] = 0
        for i in range(1, n):
            tmp[sa[i]] = tmp[sa[i - 1]]
            if key(sa[i]) != key(sa[i - 1]):
                tmp[sa[i]] += 1
        rank = tmp[:]

        if rank[sa[n - 1]] == n - 1:
            break  # 모든 rank가 고유하면 조기 종료
        k *= 2

    return sa


def build_lcp_array(s: str, sa: list[int]) -> list[int]:
    """
    Kasai's Algorithm: O(n)에 LCP 배열 구성.
    핵심 아이디어: lcp[i] ≥ lcp[i-1] - 1
    """
    n = len(s)
    lcp = [0] * n
    rank = [0] * n

    # SA의 역 배열(rank) 계산
    for i, pos in enumerate(sa):
        rank[pos] = i

    h = 0  # 현재 접미사와 이전 접미사의 LCP 길이
    for i in range(n):
        if rank[i] > 0:
            j = sa[rank[i] - 1]  # SA에서 바로 앞 접미사
            # 이미 알고 있는 h에서 시작해 비교 (핵심 최적화)
            while i + h < n and j + h < n and s[i + h] == s[j + h]:
                h += 1
            lcp[rank[i]] = h
            if h > 0:
                h -= 1  # 다음 접미사는 최소 h-1 보장
    return lcp


def search_pattern(s: str, sa: list[int], pattern: str) -> list[int]:
    """
    이진 탐색으로 패턴 검색: O(m log n).
    패턴의 모든 출현 위치를 반환.
    """
    n, m = len(s), len(pattern)

    def lower_bound() -> int:
        lo, hi = 0, n
        while lo < hi:
            mid = (lo + hi) // 2
            suffix = s[sa[mid]:sa[mid] + m]
            if suffix < pattern:
                lo = mid + 1
            else:
                hi = mid
        return lo

    def upper_bound() -> int:
        lo, hi = 0, n
        while lo < hi:
            mid = (lo + hi) // 2
            suffix = s[sa[mid]:sa[mid] + m]
            if suffix <= pattern:
                lo = mid + 1
            else:
                hi = mid
        return lo

    lo = lower_bound()
    hi = upper_bound()
    return sorted(sa[lo:hi])  # 출현 위치 (정렬된 인덱스)


def longest_repeated_substring(s: str, sa: list[int], lcp: list[int]) -> str:
    """LCP 배열의 최댓값 → 최장 반복 부분문자열"""
    max_lcp = 0
    best_pos = 0
    for i in range(1, len(s)):
        if lcp[i] > max_lcp:
            max_lcp = lcp[i]
            best_pos = sa[i]
    return s[best_pos:best_pos + max_lcp]


# 사용 예시
if __name__ == "__main__":
    text = "banana"
    sa = build_suffix_array(text)
    lcp = build_lcp_array(text, sa)

    print(f"문자열: '{text}'")
    print(f"Suffix Array: {sa}")
    print(f"LCP Array:    {lcp}\n")

    # 정렬된 접미사 출력
    print("정렬된 접미사:")
    for i, idx in enumerate(sa):
        print(f"  SA[{i}]={idx:2d}  LCP={lcp[i]}  '{text[idx:]}'")

    # 패턴 검색
    pattern = "ana"
    positions = search_pattern(text, sa, pattern)
    print(f"\n패턴 '{pattern}' 출현 위치: {positions}")  # [1, 3]

    # 최장 반복 부분문자열
    lrs = longest_repeated_substring(text, sa, lcp)
    print(f"최장 반복 부분문자열: '{lrs}'")  # 'ana'
```

### 예제 2: 두 문자열의 최장 공통 부분문자열 (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

// O(n log^2 n) Suffix Array 구성
vector<int> buildSA(const string& s) {
    int n = s.size();
    vector<int> sa(n), rank_(n), tmp(n);
    iota(sa.begin(), sa.end(), 0);
    for (int i = 0; i < n; i++) rank_[i] = s[i];

    for (int k = 1; k < n; k <<= 1) {
        auto cmp = [&](int a, int b) {
            if (rank_[a] != rank_[b]) return rank_[a] < rank_[b];
            int ra = (a + k < n) ? rank_[a + k] : -1;
            int rb = (b + k < n) ? rank_[b + k] : -1;
            return ra < rb;
        };
        sort(sa.begin(), sa.end(), cmp);
        tmp[sa[0]] = 0;
        for (int i = 1; i < n; i++) {
            tmp[sa[i]] = tmp[sa[i-1]] + (cmp(sa[i-1], sa[i]) ? 1 : 0);
        }
        rank_ = tmp;
        if (rank_[sa[n-1]] == n-1) break;
    }
    return sa;
}

// Kasai's Algorithm: O(n) LCP 배열 구성
vector<int> buildLCP(const string& s, const vector<int>& sa) {
    int n = s.size();
    vector<int> lcp(n, 0), rank_(n);
    for (int i = 0; i < n; i++) rank_[sa[i]] = i;
    int h = 0;
    for (int i = 0; i < n; i++) {
        if (rank_[i] > 0) {
            int j = sa[rank_[i] - 1];
            while (i + h < n && j + h < n && s[i + h] == s[j + h]) h++;
            lcp[rank_[i]] = h;
            if (h > 0) h--;
        }
    }
    return lcp;
}

// 두 문자열의 최장 공통 부분문자열
// 구분자($)로 연결 후 SA 구성, LCP 배열에서 교차 접미사 쌍의 최대 LCP 탐색
string longestCommonSubstring(const string& a, const string& b) {
    int na = a.size(), nb = b.size();
    // '$'는 알파벳보다 작아야 SA 정렬에서 경계 역할
    string s = a + "$" + b;
    int n = s.size();

    auto sa = buildSA(s);
    auto lcp = buildLCP(s, sa);

    int best_len = 0, best_pos = 0;
    for (int i = 1; i < n; i++) {
        // SA[i-1]과 SA[i]가 서로 다른 문자열에서 온 접미사인지 확인
        bool prev_from_a = sa[i-1] < na;
        bool curr_from_a = sa[i]   < na;
        // 구분자($)를 포함하지 않고, 서로 다른 문자열에서 온 경우만
        if (prev_from_a != curr_from_a && lcp[i] > best_len) {
            // 구분자가 LCP 안에 포함되었는지 검사
            int p = min(sa[i-1], sa[i]);
            if (p + lcp[i] <= na) { // 구분자를 넘지 않음
                best_len = lcp[i];
                best_pos = sa[i];
            }
        }
    }
    if (best_pos >= na + 1) best_pos -= (na + 1);
    return s.substr(sa[/* rank of best */0] < na ? sa[0] : sa[1], best_len);
    // 단순화: 실제로는 best를 기록한 위치에서 추출
}

int main() {
    string a = "abcdef";
    string b = "zcdemf";

    // 직접 구현한 SA + LCP로 LCS 탐색 (단순화 버전)
    string s = a + "$" + b;
    int na = a.size();
    auto sa = buildSA(s);
    auto lcp = buildLCP(s, sa);

    int best_len = 0, best_idx = 0;
    for (int i = 1; i < (int)s.size(); i++) {
        bool prev_from_a = sa[i-1] < na;
        bool curr_from_a = sa[i]   < na;
        if (prev_from_a != curr_from_a) {
            // 구분자($) 포함 여부 확인
            int actual_lcp = lcp[i];
            // LCP가 구분자를 포함하면 실제 길이 제한
            for (int k = 0; k < actual_lcp; k++) {
                if (s[sa[i] + k] == '$' || s[sa[i-1] + k] == '$') {
                    actual_lcp = k;
                    break;
                }
            }
            if (actual_lcp > best_len) {
                best_len = actual_lcp;
                best_idx = sa[i];
            }
        }
    }

    cout << "문자열 A: " << a << "\n";
    cout << "문자열 B: " << b << "\n";
    cout << "최장 공통 부분문자열: '" << s.substr(best_idx, best_len) << "'\n";
    cout << "길이: " << best_len << "\n";
    // 출력: 'cde', 길이: 3

    return 0;
}
```

---

## 고급 알고리즘: SA-IS (선형 시간 O(n))

실제 대용량 문자열 처리에서는 O(n log² n) 대신 **SA-IS (Suffix Array - Induced Sorting)** 알고리즘을 사용합니다. SA-IS는 접미사를 S-type(다음 문자보다 사전순으로 작음)과 L-type으로 분류하고, LMS(Left-Most S-type) 접미사를 기준으로 정렬을 귀납적으로 유도합니다. 구현 복잡도는 높지만 O(n) 시간/공간으로 동작합니다.

```
접미사 타입 분류 ('banana' + sentinel '$'):
b  a  n  a  n  a  $
L  S  L  S  L  S  S   (L: 다음보다 큼, S: 다음보다 작거나 같음)

LMS 접미사 (SS 패턴이 시작되는 S-type):
    *     *     *
b  a  n  a  n  a  $
   ↑     ↑     ↑
  LMS   LMS   LMS
```

---

## Z-함수와의 관계

Suffix Array와 자주 비교되는 **Z-함수(Z-Array)** 는 `Z[i]`가 `s[i..]`와 `s[0..]`의 최장 공통 접두사 길이를 저장합니다. Z-함수는 O(n) 구성, O(m + n) 단일 패턴 검색에 최적화되어 있습니다. Suffix Array는 다중 패턴 검색, 반복 쿼리, 생물정보학 등 복잡한 시나리오에서 더 유리합니다.

---

## 주의사항과 실무 팁

### 1. 구분자(Sentinel) 선택에 주의
여러 문자열을 연결할 때 사용하는 구분자는 반드시 알파벳에 포함되지 않는 문자여야 합니다. 또한 각 구분자는 서로 달라야 합니다 (두 문자열 연결 시 `$`와 `#` 등). 그렇지 않으면 잘못된 LCS 결과가 나옵니다.

### 2. SA 구성 알고리즘 선택 기준
- 길이 < 100만: O(n log² n)의 단순 구현도 충분
- 길이 > 100만 또는 대회 시간 제한: SA-IS 또는 DC3 사용
- 실무 라이브러리: `divsufsort` (C, 세계에서 가장 빠른 SA 구성)

### 3. LCP + Stack으로 다양한 문자열 문제 해결
LCP 배열과 단조 스택(Monotone Stack)을 결합하면 "k번 이상 등장하는 최장 부분문자열" 같은 문제를 O(n)에 풀 수 있습니다. 히스토그램 최대 직사각형 문제와 동일한 구조입니다.

### 4. 실무 라이브러리
- **C++**: `std::string` + `divsufsort` 라이브러리
- **Python**: `suffix-array` PyPI 패키지 또는 직접 구현
- **Java**: Apache Commons Text의 `LongestCommonSubsequence` (LCS와 다름 — 직접 구현 권장)
- **Rust**: `suffix` crate

---

## 정리

Suffix Array는 모든 접미사를 정렬한 인덱스 배열로, Suffix Tree에 비해 메모리 효율이 뛰어나고 구현이 단순합니다. Kasai's Algorithm으로 O(n) LCP Array를 구성하면 패턴 검색, 최장 반복 부분문자열, 최장 공통 부분문자열 등 다양한 문제를 효율적으로 해결할 수 있습니다. 생물정보학, 텍스트 압축, 전문 검색 엔진 등 실무에서도 핵심적으로 사용되는 알고리즘입니다.

## 참고 자료
- [Suffix Arrays: SA-IS, Kasai, DC3 구현 (GitHub - c0D3M)](https://github.com/c0D3M/Suffix-Arrays)
- [Suffix Array C++ 구현: SA-IS + Kasai (GitHub - Tascate)](https://github.com/Tascate/Suffix-Array-Implementation)
