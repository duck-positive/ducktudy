---
layout: post
title: "접미사 배열(Suffix Array)과 LCP 배열 완전 정복: 선형 시간 문자열 검색을 가능하게 하는 고급 자료구조"
date: 2026-11-17
categories: [cs, computer-science]
tags: [suffix-array, lcp-array, string-algorithm, kasai, sa-is, competitive-programming]
---

접미사 배열(Suffix Array)은 접미사 트리(Suffix Tree)의 메모리 효율적인 대안으로, 문자열 처리의 거의 모든 문제를 선형 또는 O(N log N) 시간에 해결할 수 있게 한다. 최장 공통 부분 문자열, 가장 긴 반복 부분 문자열, 문자열 압축, DNA 서열 분석, 전문 검색 엔진까지 — 접미사 배열이 없다면 불가능하거나 훨씬 느렸을 것이다.

## 접미사 배열이란?

문자열 `S = "banana"`의 모든 접미사를 나열하면:

```
0: banana
1: anana
2: nana
3: ana
4: na
5: a
```

이 접미사들을 사전 순으로 정렬하면:

```
5: a
3: ana
1: anana
0: banana
4: na
2: nana
```

**접미사 배열(SA)** 은 정렬된 접미사의 시작 인덱스 배열이다: `SA = [5, 3, 1, 0, 4, 2]`

단순히 "정렬된 접미사의 인덱스 목록"이지만, 이 배열 위에 이진 탐색을 적용하면 길이 N의 문자열에서 패턴 검색을 O(M log N)에 수행할 수 있다. (M은 패턴 길이)

## LCP 배열이란?

**LCP(Longest Common Prefix) 배열**은 접미사 배열에서 인접한 두 접미사의 가장 긴 공통 접두사 길이를 저장한다.

`SA = [5, 3, 1, 0, 4, 2]`에 대해:

```
SA[0]=5: a       → LCP[0] = 0 (첫 번째는 항상 0)
SA[1]=3: ana     → LCP[1] = 1 (a vs ana → 공통: "a")
SA[2]=1: anana   → LCP[2] = 3 (ana vs anana → 공통: "ana")
SA[3]=0: banana  → LCP[3] = 0 (anana vs banana → 공통: "")
SA[4]=4: na      → LCP[4] = 0 (banana vs na → 공통: "")
SA[5]=2: nana    → LCP[5] = 2 (na vs nana → 공통: "na")
```

`LCP = [0, 1, 3, 0, 0, 2]`

SA와 LCP를 함께 쓰면 접미사 트리와 동등한 능력을 발휘하면서 공간은 O(N)만 쓴다.

## 왜 필요한가?

| 문제 | 단순 접근법 | SA+LCP |
|------|-----------|--------|
| 패턴 검색 | O(N×M) | O(M log N) |
| 가장 긴 반복 부분 문자열 | O(N²) | O(N) |
| 두 문자열의 최장 공통 부분 문자열 | O(N×M) DP | O(N+M) |
| 문자열 압축률 추정 | O(N²) | O(N) |

전문 검색 엔진의 역인덱스(Inverted Index)에서 쿼리 자동완성, 오타 교정도 LCP 배열로 효율화된다.

## 실제 구현 예제 1: O(N log²N) 접미사 배열 구축 (Python)

```python
def build_suffix_array(s: str) -> list[int]:
    """
    접미사 배열 구축 - 이중 정렬(doubling) 방법
    시간복잡도: O(N log² N)
    """
    n = len(s)
    # 초기 순위: 각 문자의 ASCII 값
    sa = list(range(n))
    rank = [ord(c) for c in s]
    tmp = [0] * n

    gap = 1
    while gap < n:
        def sort_key(i):
            r2 = rank[i + gap] if i + gap < n else -1
            return (rank[i], r2)

        sa.sort(key=sort_key)

        # 순위 갱신
        tmp[sa[0]] = 0
        for i in range(1, n):
            tmp[sa[i]] = tmp[sa[i-1]]
            if sort_key(sa[i]) != sort_key(sa[i-1]):
                tmp[sa[i]] += 1

        rank = tmp[:]
        if rank[sa[-1]] == n - 1:
            break  # 모든 순위가 고유하면 완료
        gap *= 2

    return sa


def build_lcp_array(s: str, sa: list[int]) -> list[int]:
    """
    Kasai 알고리즘으로 LCP 배열 구축
    시간복잡도: O(N)
    """
    n = len(s)
    rank = [0] * n
    for i, v in enumerate(sa):
        rank[v] = i  # 각 접미사의 SA에서의 위치

    lcp = [0] * n
    h = 0  # 현재 LCP 값

    for i in range(n):
        if rank[i] > 0:
            j = sa[rank[i] - 1]  # SA에서 바로 앞 접미사
            # s[i:]와 s[j:]의 공통 접두사 측정
            while i + h < n and j + h < n and s[i + h] == s[j + h]:
                h += 1
            lcp[rank[i]] = h
            if h > 0:
                h -= 1  # 핵심: h는 1씩만 감소 → 전체 O(N) 보장

    return lcp


def longest_repeated_substring(s: str) -> str:
    """LCP 최댓값 위치가 가장 긴 반복 부분 문자열"""
    sa = build_suffix_array(s)
    lcp = build_lcp_array(s, sa)
    max_len = max(lcp)
    if max_len == 0:
        return ""
    idx = lcp.index(max_len)
    return s[sa[idx]:sa[idx] + max_len]


# 테스트
s = "banana"
sa = build_suffix_array(s)
lcp = build_lcp_array(s, sa)

print(f"문자열: {s}")
print(f"SA: {sa}")   # [5, 3, 1, 0, 4, 2]
print(f"LCP: {lcp}") # [0, 1, 3, 0, 0, 2]
print()

for i, idx in enumerate(sa):
    print(f"SA[{i}]={idx}: {s[idx]:15s} LCP={lcp[i]}")

print(f"\n가장 긴 반복 부분 문자열: '{longest_repeated_substring(s)}'")  # 'ana'

# 더 긴 예시
s2 = "abcabcabc"
print(f"\n'{s2}'의 가장 긴 반복: '{longest_repeated_substring(s2)}'")  # 'abcabc'
```

## Kasai 알고리즘의 핵심 아이디어

Kasai 알고리즘이 O(N)인 이유는 `h` 변수의 단조 감소 성질 때문이다.

접미사 `S[i:]`의 LCP를 계산했다면, 다음 접미사 `S[i+1:]`의 LCP는 적어도 `LCP(S[i:]) - 1`이다. 왜냐하면 `S[i:]`와 `S[j:]`가 `h`개의 공통 접두사를 가진다면, `S[i+1:]`와 `S[j+1:]`는 적어도 `h-1`개의 공통 접두사를 가지기 때문이다. 따라서 `h`는 한 번에 1씩만 감소하고, 전체 반복 중 최대 N번만 증가할 수 있으므로 총 시간복잡도는 O(N)이다.

## 실제 구현 예제 2: SA를 이용한 패턴 검색 및 최장 공통 부분 문자열 (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

// O(N log^2 N) 접미사 배열
vector<int> buildSA(const string& s) {
    int n = s.size();
    vector<int> sa(n), rank_(n), tmp(n);
    iota(sa.begin(), sa.end(), 0);
    for (int i = 0; i < n; i++) rank_[i] = s[i];

    for (int gap = 1; gap < n; gap *= 2) {
        auto cmp = [&](int a, int b) {
            if (rank_[a] != rank_[b]) return rank_[a] < rank_[b];
            int ra = a + gap < n ? rank_[a + gap] : -1;
            int rb = b + gap < n ? rank_[b + gap] : -1;
            return ra < rb;
        };
        sort(sa.begin(), sa.end(), cmp);
        tmp[sa[0]] = 0;
        for (int i = 1; i < n; i++)
            tmp[sa[i]] = tmp[sa[i-1]] + (cmp(sa[i-1], sa[i]) ? 1 : 0);
        rank_ = tmp;
        if (rank_[sa[n-1]] == n-1) break;
    }
    return sa;
}

// Kasai O(N) LCP 배열
vector<int> buildLCP(const string& s, const vector<int>& sa) {
    int n = s.size();
    vector<int> rank_(n), lcp(n);
    for (int i = 0; i < n; i++) rank_[sa[i]] = i;
    int h = 0;
    for (int i = 0; i < n; i++) {
        if (rank_[i] > 0) {
            int j = sa[rank_[i] - 1];
            while (i + h < n && j + h < n && s[i+h] == s[j+h]) h++;
            lcp[rank_[i]] = h;
            if (h) h--;
        }
    }
    return lcp;
}

// 두 문자열의 최장 공통 부분 문자열 (LCS)
string longestCommonSubstring(const string& a, const string& b) {
    // 두 문자열을 '$' 구분자로 연결
    string s = a + "$" + b;
    int na = a.size();
    int n = s.size();
    auto sa = buildSA(s);
    auto lcp = buildLCP(s, sa);

    int best_len = 0, best_pos = 0;
    for (int i = 1; i < n; i++) {
        // SA에서 인접한 두 접미사가 서로 다른 원본 문자열에서 왔는지 확인
        bool from_a_prev = sa[i-1] < na;
        bool from_a_curr = sa[i] < na;
        if (from_a_prev != from_a_curr && lcp[i] > best_len) {
            best_len = lcp[i];
            best_pos = sa[i];
        }
    }
    return s.substr(best_pos, best_len);
}

// SA로 패턴 검색: O(M log N)
pair<int,int> searchPattern(const string& text, const string& pattern,
                             const vector<int>& sa) {
    int n = text.size(), m = pattern.size();
    // 하한 (pattern 이상인 첫 위치)
    int lo = 0, hi = n;
    while (lo < hi) {
        int mid = (lo + hi) / 2;
        if (text.compare(sa[mid], m, pattern) < 0) lo = mid + 1;
        else hi = mid;
    }
    int left = lo;
    // 상한
    hi = n;
    while (lo < hi) {
        int mid = (lo + hi) / 2;
        if (text.compare(sa[mid], m, pattern) <= 0) lo = mid + 1;
        else hi = mid;
    }
    return {left, lo}; // SA[left..lo) 가 모든 매칭 위치

}

int main() {
    string text = "abracadabra";
    auto sa = buildSA(text);
    auto lcp = buildLCP(text, sa);

    cout << "Text: " << text << "\n";
    cout << "SA: ";
    for (int x : sa) cout << x << " ";
    cout << "\nLCP: ";
    for (int x : lcp) cout << x << " ";
    cout << "\n\n";

    // 패턴 검색
    string pattern = "abr";
    auto [l, r] = searchPattern(text, pattern, sa);
    cout << "Pattern '" << pattern << "' found at positions: ";
    for (int i = l; i < r; i++) cout << sa[i] << " ";
    cout << "\n";

    // 최장 공통 부분 문자열
    string a = "ABABC", b = "BABCAB";
    cout << "LCS of '" << a << "' and '" << b << "': '"
         << longestCommonSubstring(a, b) << "'\n"; // BABC
    return 0;
}
```

## 고급 주제: SA-IS 알고리즘 (O(N) 선형 시간)

SA-IS(Suffix Array - Induced Sorting)는 2009년 Nong, Zhang, Chan이 발표한 선형 시간 접미사 배열 구축 알고리즘이다. 핵심 아이디어는 접미사를 S-type(다음 접미사보다 사전순 작음)과 L-type(크거나 같음)으로 분류하고, LMS(Left-Most S-type) 접미사만 먼저 정렬한 뒤 나머지를 유도(induction)하는 것이다.

구현이 복잡하지만 대용량 텍스트(DNA 서열, 대형 로그 파일) 처리 시 반드시 고려해야 한다.

## 응용: 최장 반복 부분 문자열, Burrows-Wheeler 변환

**BWT(Burrows-Wheeler Transform)** 는 SA를 이용해 텍스트를 변환하는데, 이 변환은 bzip2 압축과 DNA 서열 정렬 도구(BWA, Bowtie)의 핵심이다. SA의 각 위치에서 텍스트의 한 칸 앞 문자를 모으면 BWT가 완성된다.

```python
def bwt(s: str) -> str:
    s += '$'  # 끝 표시자 (어떤 문자보다도 작음)
    n = len(s)
    sa = build_suffix_array(s)
    return ''.join(s[i - 1] for i in sa)

print(bwt("banana"))  # "annb$aa" 형태의 압축 친화적 문자열
```

## 주의사항과 팁

**경쟁 프로그래밍**: O(N log²N) 구현으로 충분한 경우가 많다. N이 50만을 초과하면 SA-IS를 고려한다.

**0-indexed vs 1-indexed**: 구현 시 인덱스 체계를 일관되게 유지하라. 특히 LCP 배열의 `lcp[0] = 0` 처리와 SA 인덱스 오프셋에서 버그가 많이 발생한다.

**정수 알파벳**: SA-IS 등 선형 알고리즘은 알파벳을 정수 범위 `[0, σ)`로 정규화하는 단계가 선행되어야 한다.

**메모리 사용**: SA + LCP = 2N 정수 → N이 100만이면 약 8MB. 대형 게놈 서열(수십억 염기쌍)을 처리할 때는 압축 접미사 배열(CSA) 연구 분야가 별도로 존재한다.

## 참고 자료

- [Wikipedia: Suffix Array](https://en.wikipedia.org/wiki/Suffix_array)
- [Wikipedia: LCP Array](https://en.wikipedia.org/wiki/LCP_array)
- [GeeksforGeeks: Kasai's Algorithm](https://www.geeksforgeeks.org/kasais-algorithm-for-construction-of-lcp-array-from-suffix-array/)
- [VisuAlgo: Suffix Array Visualization](https://visualgo.net/en/suffixarray)
