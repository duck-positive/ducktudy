---
layout: post
title: "접미사 배열(Suffix Array)과 LCP 배열 완전 정복: SA-IS와 Kasai 알고리즘으로 문자열을 정복하는 법"
date: 2026-08-31
categories: [cs, computer-science]
tags: [suffix-array, lcp-array, string-algorithm, sa-is, kasai, competitive-programming]
---

## 개념 설명

**접미사 배열(Suffix Array, SA)**은 문자열 `s`의 모든 접미사(suffix)를 사전순으로 정렬했을 때, 각 접미사가 시작하는 인덱스를 순서대로 저장한 배열입니다. 예를 들어 문자열 `"banana"`의 접미사는 다음과 같습니다:

| 인덱스 | 접미사      |
|--------|------------|
| 0      | banana     |
| 1      | anana      |
| 2      | nana       |
| 3      | ana        |
| 4      | na         |
| 5      | a          |

사전순으로 정렬하면 `a < ana < anana < banana < na < nana`이므로, 접미사 배열은 `SA = [5, 3, 1, 0, 4, 2]`가 됩니다.

**LCP 배열(Longest Common Prefix Array)**은 접미사 배열에서 인접한 두 접미사 사이의 가장 긴 공통 접두사의 길이를 저장한 배열입니다. `LCP[i]`는 `SA[i-1]`번째 접미사와 `SA[i]`번째 접미사 사이의 최장 공통 접두사 길이를 나타냅니다 (`LCP[0] = 0`으로 정의).

`"banana"` 예시에서 LCP 배열은 `[0, 1, 3, 0, 0, 2]`가 됩니다.

---

## 왜 필요한가?

접미사 배열과 LCP 배열은 문자열 처리 분야에서 **접미사 트리(Suffix Tree)와 동등한 정보를 훨씬 적은 메모리로** 표현합니다. 접미사 트리는 이론적으로 강력하지만 구현이 복잡하고 실제 메모리 사용량이 매우 큽니다. 반면 접미사 배열은 간결한 배열 구조로 다음 문제들을 효율적으로 해결합니다:

- **부분 문자열 검색**: 패턴 P가 문자열 S에 존재하는지 O(|P| log |S|)에 이분 탐색으로 확인
- **최장 반복 부분 문자열**: LCP 배열의 최댓값
- **서로 다른 부분 문자열의 수**: `|S|*(|S|+1)/2 - sum(LCP)`
- **최장 공통 부분 문자열(LCS)**: 두 문자열을 특수 구분자로 연결한 뒤 SA + LCP 활용
- **DNA 서열 분석**, **데이터 압축(BWT)**, **검색 엔진 인덱싱** 등

---

## 실제 구현 예제

### 예제 1: O(n log²n) 접미사 배열 구축 (Python)

가장 직관적인 구현은 비교 기반 정렬입니다. 각 접미사를 직접 정렬하면 O(n² log n)이지만, **배가(doubling) 전략**을 쓰면 O(n log²n)으로 줄일 수 있습니다.

```python
def build_suffix_array(s: str) -> list[int]:
    """배가 기법으로 접미사 배열 구축 O(n log^2 n)"""
    n = len(s)
    # (rank, index) 쌍을 초기화: 첫 번째 문자 기준
    sa = sorted(range(n), key=lambda i: s[i])
    rank = [0] * n
    tmp  = [0] * n

    rank[sa[0]] = 0
    for i in range(1, n):
        rank[sa[i]] = rank[sa[i-1]] + (s[sa[i]] != s[sa[i-1]])

    k = 1
    while k < n:
        # k 길이 기준 rank로 2k 길이 정렬
        def key(i):
            return (rank[i], rank[i + k] if i + k < n else -1)
        sa = sorted(range(n), key=key)
        tmp[sa[0]] = 0
        for i in range(1, n):
            tmp[sa[i]] = tmp[sa[i-1]] + (key(sa[i]) != key(sa[i-1]))
        rank = tmp[:]
        if rank[sa[-1]] == n - 1:
            break   # 모든 순위가 유일하면 종료
        k *= 2
    return sa


def build_lcp_array(s: str, sa: list[int]) -> list[int]:
    """Kasai 알고리즘으로 LCP 배열 구축 O(n)"""
    n = len(s)
    rank = [0] * n
    for i, v in enumerate(sa):
        rank[v] = i

    lcp = [0] * n
    h = 0
    for i in range(n):
        if rank[i] > 0:
            j = sa[rank[i] - 1]
            while i + h < n and j + h < n and s[i + h] == s[j + h]:
                h += 1
            lcp[rank[i]] = h
            if h > 0:
                h -= 1
    return lcp


# 테스트
s = "banana"
sa  = build_suffix_array(s)
lcp = build_lcp_array(s, sa)

print("접미사 배열:", sa)        # [5, 3, 1, 0, 4, 2]
print("LCP 배열:  ", lcp)       # [0, 1, 3, 0, 0, 2]
print("정렬된 접미사:")
for idx in sa:
    print(f"  SA[{idx}] = {s[idx:]!r}")
```

**Kasai 알고리즘의 핵심 아이디어**: 인덱스 `i`에서 시작하는 접미사의 LCP가 `h`라면, 인덱스 `i+1`에서 시작하는 접미사의 LCP는 최소 `h-1`임을 이용해 h를 단조 감소시키면서 전체 배열을 O(n)에 계산합니다.

---

### 예제 2: 접미사 배열로 서로 다른 부분 문자열 개수 세기 (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

// SA-IS 대신 간결한 O(n log^2 n) 구현
vector<int> buildSA(const string& s) {
    int n = s.size();
    vector<int> sa(n), rank_(n), tmp(n);
    iota(sa.begin(), sa.end(), 0);
    for (int i = 0; i < n; i++) rank_[i] = s[i];

    for (int k = 1; k < n; k <<= 1) {
        auto cmp = [&](int a, int b) {
            if (rank_[a] != rank_[b]) return rank_[a] < rank_[b];
            int ra = a + k < n ? rank_[a + k] : -1;
            int rb = b + k < n ? rank_[b + k] : -1;
            return ra < rb;
        };
        sort(sa.begin(), sa.end(), cmp);
        tmp[sa[0]] = 0;
        for (int i = 1; i < n; i++)
            tmp[sa[i]] = tmp[sa[i-1]] + cmp(sa[i-1], sa[i]);
        rank_ = tmp;
    }
    return sa;
}

vector<int> buildLCP(const string& s, const vector<int>& sa) {
    int n = s.size();
    vector<int> rank_(n), lcp(n, 0);
    for (int i = 0; i < n; i++) rank_[sa[i]] = i;
    for (int i = 0, h = 0; i < n; i++) {
        if (rank_[i] > 0) {
            int j = sa[rank_[i] - 1];
            while (i + h < n && j + h < n && s[i+h] == s[j+h]) h++;
            lcp[rank_[i]] = h;
            if (h) h--;
        }
    }
    return lcp;
}

long long countDistinctSubstrings(const string& s) {
    int n = s.size();
    auto sa  = buildSA(s);
    auto lcp = buildLCP(s, sa);

    // 전체 부분 문자열 수 - 중복(LCP 합)
    long long total = (long long)n * (n + 1) / 2;
    long long dup   = 0;
    for (int x : lcp) dup += x;
    return total - dup;
}

int main() {
    string s = "abcabc";
    cout << "문자열: " << s << "\n";
    cout << "서로 다른 부분 문자열 수: " << countDistinctSubstrings(s) << "\n";
    // "abcabc"는 전체 21개 - 중복 6개 = 15개
    return 0;
}
```

---

## 주의사항 및 팁

### 1. SA-IS 알고리즘 (실전 권장)
경쟁 프로그래밍 실전에서는 위의 O(n log²n) 대신 **SA-IS(Suffix Array - Induced Sorting)**를 사용하면 O(n)으로 구축할 수 있습니다. 구현 복잡도는 높지만 수백만 자 이상의 문자열에서 유의미한 차이를 만듭니다.

### 2. 문자 집합 주의
SA 구축 시 문자 집합을 정수로 변환(좌표 압축)하면 다양한 알파벳에 적용 가능합니다. 특수 구분자(보통 `$`, ASCII 코드가 가장 작은 문자)를 사용할 때는 두 문자열의 경계를 명확히 해야 합니다.

### 3. LCP Range Minimum Query
LCP 배열에 **Sparse Table**을 올리면 임의 구간 [l, r]의 최솟값을 O(1)에 구할 수 있어, 임의의 두 접미사 사이의 LCP를 O(1)에 계산합니다 (접미사 배열 상 두 위치의 LCP = 해당 구간의 LCP 최솟값).

### 4. 메모리 레이아웃
접미사 배열 탐색은 메모리 접근 패턴이 불규칙하므로, 대형 문자열에서는 **캐시 미스**가 잦습니다. 정수 배열이므로 접미사 트리 대비 캐시 친화적이지만, 수백 MB 이상의 텍스트에서는 외부 메모리 SA 구축 알고리즘이 필요합니다.

---

## 참고 자료

- [Suffix Array - CP-Algorithms](https://cp-algorithms.com/string/suffix-array.html)
- [Kasai's Algorithm for LCP Array - GeeksforGeeks](https://www.geeksforgeeks.org/dsa/kasais-algorithm-for-construction-of-lcp-array-from-suffix-array/)
- [Suffix Array - Wikipedia](https://en.wikipedia.org/wiki/Suffix_array)
- [SA-IS 원논문: Linear Suffix Array Construction by Almost Pure Induced-Sorting (Nong, Zhang, Chan, 2009)](https://ieeexplore.ieee.org/document/4976463)
