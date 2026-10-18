---
layout: post
title: "팰린드롬 트리(Eertree) 완전 정복: 모든 팰린드롬 부분 문자열을 선형 시간에 찾는 자료구조"
date: 2026-10-18
categories: [cs, computer-science]
tags: [palindrome-tree, eertree, string-algorithm, data-structure, competitive-programming]
---

## 개요

문자열 처리 알고리즘 중 **팰린드롬(palindrome)**—앞으로 읽으나 뒤로 읽으나 같은 문자열—에 관한 문제는 빈번하게 등장한다. 단순히 최장 팰린드롬 부분 문자열을 구하는 Manacher 알고리즘은 잘 알려져 있지만, "서로 다른 팰린드롬 부분 문자열이 몇 개인가?", "팰린드롬 부분 문자열의 개수의 합은?", "팰린드롬 분리(palindrome factorization) 최솟값은?" 같은 더 어려운 질문에는 충분하지 않다.

2014년 Mikhail Rubinchik과 Arseny Shur가 고안한 **팰린드롬 트리(Palindrome Tree)**, 혹은 **Eertree**는 길이 n인 문자열의 모든 *서로 다른* 팰린드롬 부분 문자열을 O(n) 시간·공간에 저장하는 자료구조다. 팰린드롬 트리 하나만으로 위의 복잡한 질문들을 효율적으로 해결할 수 있다.

---

## 왜 팰린드롬 트리가 필요한가

길이 n인 문자열에는 최대 O(n²)개의 팰린드롬 부분 문자열 **출현(occurrence)**이 있다. 그러나 서로 다른 팰린드롬 부분 문자열의 수는 최대 n+1개임이 알려져 있다. 이 놀라운 성질 덕분에 팰린드롬 트리는 선형 공간에 모든 정보를 압축할 수 있다.

기존 접근법의 한계를 살펴보자.

- **Manacher 알고리즘**: 각 위치를 중심으로 하는 최대 팰린드롬 반경을 O(n)에 구한다. 그러나 서로 다른 팰린드롬의 목록이나 개수 합은 직접 구하기 어렵다.
- **접미사 배열 / 접미사 자동자**: 일반 문자열 처리에 강력하지만 팰린드롬 특성을 직접 활용하지 않는다.
- **브루트-포스 DP**: O(n²) 혹은 O(n² log n) 시간이 필요해 n ≤ 10⁵ 이상에서는 TLE를 면치 못한다.

팰린드롬 트리는 이 모든 문제를 O(n) 시간 + O(n·|Σ|) 공간으로 해결한다.

---

## 팰린드롬 트리의 구조

팰린드롬 트리는 사실 *두 개의 루트*를 가진 방향 그래프다.

| 노드 | 의미 | 길이(`len`) |
|------|------|------------|
| 루트 0 (odd root) | 가상 노드 | -1 |
| 루트 1 (even root) | 빈 팰린드롬 | 0 |
| 일반 노드 v | 어떤 팰린드롬 p | len(p) ≥ 1 |

각 노드는 다음 필드를 갖는다.

- **`len`**: 이 노드가 나타내는 팰린드롬의 길이.
- **`suf_link`**: *접미사 링크(suffix link)*. 현재 팰린드롬의 가장 긴 진짜 팰린드롬 접미사(proper palindromic suffix)를 나타내는 노드를 가리킨다.
- **`children[c]`**: 문자 c를 양쪽에 붙였을 때 새로운 팰린드롬이 되는 자식 노드.

문자열 "abaab"를 처리하면 다음 팰린드롬들이 트리에 저장된다: "a", "b", "aba", "aa", "baab", "abaab"(이 중 일부).

---

## 구현 예제 1 — Python: 팰린드롬 트리 기본 구현

```python
class Node:
    def __init__(self, length, suf_link):
        self.len = length
        self.suf_link = suf_link
        self.children = {}
        self.cnt = 0          # 이 팰린드롬이 나타나는 횟수 (접미사 링크 전파 전)
        self.series_link = None  # 같은 시리즈(AABAA 계열) 링크
        self.diff = 0         # len - suf_link.len
        self.slink_cnt = 0    # 시리즈 내 누적 등장 횟수

class PalindromeTree:
    def __init__(self):
        # 가상 루트(-1)와 빈 팰린드롬(0) 루트
        self.odd  = Node(-1, None)   # 루트 0
        self.even = Node(0, None)    # 루트 1
        self.odd.suf_link  = self.odd
        self.even.suf_link = self.odd

        self.last = self.even
        self.s = []           # 처리된 문자열
        self.size = 2         # 노드 수 (루트 2개 포함)

    def _get_link(self, node):
        """현재 접미사에 이 노드를 붙일 수 있는지 확인."""
        i = len(self.s) - 1
        while True:
            j = i - node.len - 1
            if j >= 0 and self.s[j] == self.s[i]:
                return node
            node = node.suf_link

    def add(self, c: str):
        self.s.append(c)
        cur = self._get_link(self.last)

        if c not in cur.children:
            # 새 팰린드롬 노드 생성
            new_len = cur.len + 2
            if new_len == 1:
                suf = self.even
            else:
                suf = self._get_link(cur.suf_link).children[c]
            new_node = Node(new_len, suf)
            cur.children[c] = new_node
            self.size += 1
        else:
            new_node = cur.children[c]

        new_node.cnt += 1
        self.last = new_node

    def count_distinct(self):
        """서로 다른 팰린드롬 부분 문자열 수 (빈 문자열 제외)."""
        return self.size - 2   # 가상 루트 2개 제외

    def propagate_counts(self):
        """각 노드의 실제 등장 횟수를 접미사 링크를 통해 전파."""
        # 노드를 길이 기준 내림차순으로 처리해야 한다.
        # 여기서는 BFS 대신 간단히 DFS로 수집 후 역순 처리
        all_nodes = []
        stack = [self.odd, self.even]
        while stack:
            v = stack.pop()
            all_nodes.append(v)
            for child in v.children.values():
                stack.append(child)

        all_nodes.sort(key=lambda n: -n.len)
        for v in all_nodes:
            if v.suf_link and v.suf_link is not v:
                v.suf_link.cnt += v.cnt


# --- 사용 예시 ---
def solve(s: str):
    pt = PalindromeTree()
    for c in s:
        pt.add(c)
    pt.propagate_counts()
    print(f"문자열: {s}")
    print(f"서로 다른 팰린드롬 부분 문자열 수: {pt.count_distinct()}")

solve("abaab")   # -> 6: a, b, aba, aa, baab, abaab
solve("abacaba") # -> 7: a, b, c, aba, aca, bacab, abacaba
```

---

## 구현 예제 2 — C++: 빠른 팰린드롬 트리로 최소 팰린드롬 분리 수 구하기

팰린드롬 분리(palindrome factorization) 문제: 문자열 s를 팰린드롬들의 연결로 분리할 때 최소 조각 수를 구하라.

```cpp
#include <bits/stdc++.h>
using namespace std;

struct Node {
    int len, suf_link, diff, series_link;
    long long cnt, slink_cnt;
    map<char, int> ch;
};

struct PalindromicTree {
    vector<Node> t;
    string s;
    int last;

    PalindromicTree() {
        t.push_back({-1, 0, 0, 0, 0, 0, {}}); // 루트0: odd
        t.push_back({ 0, 0, 0, 0, 0, 0, {}}); // 루트1: even
        t[0].suf_link = 0;
        t[1].suf_link = 0;
        last = 1;
    }

    int getLink(int v) {
        while (true) {
            int j = (int)s.size() - t[v].len - 2;
            if (j >= 0 && s[j] == s.back()) return v;
            v = t[v].suf_link;
        }
    }

    void addChar(char c) {
        s += c;
        int cur = getLink(last);
        char ch = s.back();

        if (!t[cur].ch.count(ch)) {
            int newLen = t[cur].len + 2;
            int sufLink;
            if (newLen == 1) {
                sufLink = 1;
            } else {
                sufLink = t[getLink(t[cur].suf_link)].ch[ch];
            }
            Node nw;
            nw.len = newLen;
            nw.suf_link = sufLink;
            nw.diff = newLen - t[sufLink].len;
            if (nw.diff == t[sufLink].diff)
                nw.series_link = t[sufLink].series_link;
            else
                nw.series_link = sufLink;
            nw.cnt = nw.slink_cnt = 0;
            t[cur].ch[ch] = (int)t.size();
            t.push_back(nw);
        }

        last = t[cur].ch[ch];
        t[last].cnt++;
    }
};

// dp[i] = s[0..i-1]를 팰린드롬들로 분리하는 최솟값
vector<int> minPalindromeSplit(const string& s) {
    int n = s.size();
    PalindromicTree pt;
    vector<int> dp(n + 1, INT_MAX);
    dp[0] = 0;

    for (int i = 0; i < n; i++) {
        pt.addChar(s[i]);
        // series_link를 활용한 O(n log n) 최적화
        for (int v = pt.last; v > 1; v = pt.t[v].series_link) {
            pt.t[v].slink_cnt = dp[i + 1 - pt.t[pt.t[v].series_link].len - pt.t[v].diff];
            if (pt.t[v].diff == pt.t[pt.t[v].suf_link].diff)
                pt.t[v].slink_cnt = min(pt.t[v].slink_cnt,
                                        pt.t[pt.t[v].suf_link].slink_cnt);
            if (pt.t[v].slink_cnt != INT_MAX)
                dp[i + 1] = min(dp[i + 1], pt.t[v].slink_cnt + 1);
        }
    }
    return dp;
}

int main() {
    string s = "abaab";
    auto dp = minPalindromeSplit(s);
    cout << "최소 팰린드롬 분리 수: " << dp[s.size()] << "\n";
    // "a" + "baab" = 2조각
    return 0;
}
```

---

## 주요 알고리즘 설명

### `getLink(v)` 함수의 동작

현재까지 처리된 문자열의 *가장 긴 팰린드롬 접미사*의 노드 v에서 시작하여, 새로 추가된 문자 c를 양쪽에 붙였을 때 여전히 팰린드롬이 되는 위치를 찾는다. 불일치하면 접미사 링크를 따라 올라간다. 이 과정은 amortized O(n)임이 증명되어 있다 — 접미사 링크는 항상 길이가 더 짧은 팰린드롬으로 연결되기 때문이다.

### 접미사 링크 (Suffix Link)

팰린드롬 트리에서 접미사 링크는 현재 노드가 나타내는 팰린드롬의 **가장 긴 진짜 팰린드롬 접미사**를 가리킨다. 예를 들어 "abaaba"의 접미사 링크는 "aba" → "a" → (even root)로 이어진다.

접미사 링크를 통해 `cnt` 전파(propagation)를 하면 각 팰린드롬 부분 문자열의 정확한 등장 횟수를 O(n)에 구할 수 있다.

### Series Link를 이용한 DP 최적화

팰린드롬 분리 DP를 단순 팰린드롬 트리로 구현하면 O(n²) 이 된다. 그러나 **같은 차이(diff = len - suf_link.len)가 반복되는 시리즈**를 묶는 `series_link`를 도입하면 O(n log n) 혹은 O(n)까지 최적화할 수 있다.

---

## 복잡도 분석

| 연산 | 시간 | 공간 |
|------|------|------|
| 문자 삽입 (전체) | O(n · |Σ|) amortized | O(n · |Σ|) |
| 서로 다른 팰린드롬 수 | O(1) | — |
| 등장 횟수 전파 | O(n) | — |
| 팰린드롬 분리 DP | O(n log n) | O(n) |

---

## 주의사항 및 팁

1. **루트가 두 개**인 이유: 홀수 길이 팰린드롬의 부모 역할을 하는 `-1` 길이 가상 루트와 짝수 길이 팰린드롬의 시작점인 `0` 길이 루트가 분리되어야 팰린드롬 확장 로직이 단순해진다.

2. **children을 `map` vs `array`**: 알파벳이 소문자 26자라면 `int ch[26]`을 쓰는 것이 `map` 대비 훨씬 빠르다. 경쟁 프로그래밍에서는 고정 크기 배열로 구현하는 것이 일반적이다.

3. **등장 횟수 계산 시 정렬 필요**: 접미사 링크를 통해 cnt를 전파하려면 길이 내림차순으로 노드를 처리해야 한다 (자식 → 부모 방향).

4. **online 처리 가능**: 팰린드롬 트리는 문자 추가를 online으로 처리한다. 스트리밍 입력이나 동적 문자열 처리에 적합하다. 다만 삭제는 지원하지 않는다.

5. **관련 문제 유형**: BOJ 16564 (팰린드롬 분리), Codeforces의 팰린드롬 시리즈 문제들, ICPC 지역 대회의 문자열 문제에서 자주 출제된다.

---

## 참고 자료
- [Palindrome tree - Wikipedia](https://en.wikipedia.org/wiki/Palindrome_tree)
- [Eertree: An Efficient Data Structure for Processing Palindromes in Strings (arXiv:1506.04862)](https://arxiv.org/abs/1506.04862)
- [Eertree implementation - Rosetta Code](https://rosettacode.org/wiki/Eertree)
- [Min-cost flow and palindrome applications - CP-Algorithms](https://cp-algorithms.com/graph/min_cost_flow.html)
