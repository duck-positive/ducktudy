---
layout: post
title: "접미사 자동자(Suffix Automaton) 완전 정복: 모든 부분 문자열을 O(n) 공간에 압축하는 알고리즘"
date: 2026-08-02
categories: [cs, computer-science]
tags: [suffix-automaton, string-algorithm, automaton, dynamic-programming, competitive-programming]
---

## 개념 설명

**접미사 자동자(Suffix Automaton, SAM)**는 문자열 `s`의 모든 부분 문자열(substring)을 인식하는 최소 결정적 유한 오토마톤(Minimal DFA)이다. 길이 `n`인 문자열에 대해 **O(n) 시간과 O(n) 공간**으로 구축할 수 있으며, 이 자동자를 통해 부분 문자열 검색, 최장 공통 부분 문자열, 서로 다른 부분 문자열 수 계산 등 다양한 문자열 처리 문제를 극도로 효율적으로 해결할 수 있다.

### 핵심 개념: endpos 등가 클래스

접미사 자동자를 이해하는 가장 중요한 개념은 **endpos(끝 위치 집합)**다. 문자열 `s`의 어떤 부분 문자열 `t`에 대해 `endpos(t)`는 `t`가 `s` 안에서 끝나는 위치들의 집합이다.

예를 들어 `s = "abcbc"`에서:
- `endpos("bc") = {3, 5}` → `s[2..3]`과 `s[4..5]`에서 끝남
- `endpos("c") = {3, 5}` → 동일!
- `endpos("abc") = {3}` → 한 번만 등장

두 부분 문자열의 `endpos`가 같다면, 하나는 항상 다른 하나의 접미사이고, 이 둘은 같은 **등가 클래스(equivalence class)**에 속한다. 접미사 자동자의 각 **상태(state)**가 바로 이 등가 클래스 하나에 대응한다.

### 상태의 속성

각 상태 `v`는 다음 정보를 가진다:

| 속성 | 설명 |
|------|------|
| `len` | 이 상태가 표현하는 부분 문자열 중 가장 긴 것의 길이 |
| `link` | 접미사 링크(suffix link) - 더 짧은 등가 클래스로의 포인터 |
| `next` | 전이 함수 - 문자 `c`를 붙였을 때 이동하는 상태 |

**접미사 링크(suffix link)**는 현재 상태의 등가 클래스에서 가장 짧은 문자열의 진접미사(proper suffix)가 속하는 상태를 가리킨다. 모든 접미사 링크를 따라가면 초기 상태 `root`에 도달하며, 이 링크들은 **접미사 링크 트리(suffix link tree)**를 형성한다.

### 크기 보장

길이 `n`인 문자열에 대해 접미사 자동자는:
- **상태 수**: 최대 `2n - 1`
- **전이 수**: 최대 `3n - 4`

이 두 경계는 tight하며, `n ≥ 3`일 때 `s = "abbb...b"` 형태의 문자열에서 달성된다. 이처럼 접미사 트리(suffix tree)가 `O(n)` 노드를 가지는 것처럼, 접미사 자동자 역시 선형 공간에 모든 부분 문자열 정보를 압축한다.

---

## 왜 필요한가

### 기존 방법의 한계

문자열 `s`의 모든 부분 문자열을 저장하는 가장 단순한 방법은 길이 `1`부터 `n`까지 모든 부분 문자열을 명시적으로 열거하는 것인데, 이는 **O(n²) 공간**이 필요하다. 길이 10만인 문자열은 약 50억 개의 부분 문자열을 가질 수 있다.

다른 방법인 접미사 배열(Suffix Array)은 O(n) 공간이지만, 임의의 패턴 매칭에 O(m log n) 시간이 걸린다. 접미사 트리(Suffix Tree)는 O(n) 구축과 O(m) 검색이 가능하지만 구현이 매우 복잡하다.

접미사 자동자는:
- **구축**: O(n) 시간, O(n) 공간
- **패턴 검색**: O(m) 시간 (패턴 길이 m)
- **서로 다른 부분 문자열 수**: O(n) 전처리 후 O(1) 쿼리
- **구현 난이도**: 접미사 트리보다 현저히 낮음

### 실전 활용 사례

- **바이오인포매틱스**: DNA/RNA 서열에서 반복 패턴 분석
- **검색 엔진**: 접미사 자동자 기반 인덱스 구조
- **표절 검출**: 두 문서의 최장 공통 부분 문자열 계산
- **텍스트 압축**: 반복 패턴 인식 기반 압축 (LZ류 알고리즘과 연관)
- **경쟁 프로그래밍**: 복잡한 문자열 DP 문제 해결

---

## 실제 구현 예제

### 예제 1: 접미사 자동자 구축 (C++)

온라인 알고리즘으로, 문자를 하나씩 추가하며 자동자를 점진적으로 확장한다.

```cpp
#include <bits/stdc++.h>
using namespace std;

struct SuffixAutomaton {
    struct State {
        int len;                  // 이 상태의 longest 부분 문자열 길이
        int link;                 // 접미사 링크
        map<char, int> next;      // 전이 함수
        long long cnt;            // 이 상태를 끝으로 하는 부분 문자열 수 (출현 횟수)
        bool isClone;             // clone된 상태인지 여부
    };

    vector<State> st;
    int last;

    SuffixAutomaton() {
        st.push_back({0, -1, {}, 0, false}); // 초기 상태 (root)
        last = 0;
    }

    void extend(char c) {
        // 새 상태 cur 생성: 현재 last보다 길이 1 긴 문자열을 표현
        int cur = st.size();
        st.push_back({st[last].len + 1, -1, {}, 1, false});
        
        int p = last;
        // cur로 향하는 전이를 p에서 시작하여 root까지 추가
        while (p != -1 && !st[p].next.count(c)) {
            st[p].next[c] = cur;
            p = st[p].link;
        }

        if (p == -1) {
            // c로 시작하는 suffix가 없음 → root에서 시작
            st[cur].link = 0;
        } else {
            int q = st[p].next[c];
            if (st[p].len + 1 == st[q].len) {
                // q가 이미 올바른 상태
                st[cur].link = q;
            } else {
                // q를 clone하여 두 상태로 분리
                int clone = st.size();
                st.push_back({st[p].len + 1, st[q].link, st[q].next, 0, true});
                
                // p에서 시작하여 q를 가리키던 전이들을 clone으로 변경
                while (p != -1 && st[p].next[c] == q) {
                    st[p].next[c] = clone;
                    p = st[p].link;
                }
                st[q].link = clone;
                st[cur].link = clone;
            }
        }
        last = cur;
    }

    // 각 상태의 출현 횟수 계산 (len 내림차순 토포 정렬 후 링크 따라 전파)
    void computeCounts() {
        int sz = st.size();
        vector<int> order(sz);
        iota(order.begin(), order.end(), 0);
        sort(order.begin(), order.end(), [&](int a, int b) {
            return st[a].len > st[b].len;
        });
        for (int v : order) {
            if (st[v].link >= 0)
                st[st[v].link].cnt += st[v].cnt;
        }
    }
};

// 사용 예시
int main() {
    string s = "abcbc";
    SuffixAutomaton sam;
    for (char c : s) sam.extend(c);
    sam.computeCounts();

    // 패턴 "bc"가 s에 몇 번 등장하는지 확인
    string pattern = "bc";
    int cur = 0;
    bool found = true;
    for (char c : pattern) {
        if (!sam.st[cur].next.count(c)) { found = false; break; }
        cur = sam.st[cur].next[c];
    }
    if (found)
        cout << "\"" << pattern << "\" 등장 횟수: " << sam.st[cur].cnt << "\n"; // 2
    else
        cout << "패턴을 찾을 수 없습니다\n";

    return 0;
}
```

빌드 및 실행:
```
g++ -O2 -o sam sam.cpp
./sam
# "bc" 등장 횟수: 2
```

### 예제 2: 서로 다른 부분 문자열 수 계산 및 두 문자열의 최장 공통 부분 문자열

```cpp
#include <bits/stdc++.h>
using namespace std;

// 위의 SuffixAutomaton 구조체를 재사용한다고 가정

// 1. 서로 다른 부분 문자열의 수
// 각 상태 v는 [link.len + 1, v.len] 범위의 길이를 가진 부분 문자열을 표현한다.
// 따라서 상태 v가 기여하는 서로 다른 부분 문자열 수 = v.len - v.link.len
long long countDistinctSubstrings(const string& s) {
    SuffixAutomaton sam;
    for (char c : s) sam.extend(c);

    long long result = 0;
    for (int i = 1; i < (int)sam.st.size(); i++) {
        result += sam.st[i].len - sam.st[sam.st[i].link].len;
    }
    return result;
}

// 2. 두 문자열의 최장 공통 부분 문자열 (Longest Common Substring)
// s1의 접미사 자동자를 구축한 후, s2의 각 문자를 자동자에서 매칭하며
// 현재 매칭 길이를 추적한다.
string longestCommonSubstring(const string& s1, const string& s2) {
    SuffixAutomaton sam;
    for (char c : s1) sam.extend(c);

    int cur = 0, curLen = 0;    // 현재 상태와 매칭 길이
    int bestLen = 0, bestEnd = 0; // 최장 매칭 정보

    for (int i = 0; i < (int)s2.size(); i++) {
        char c = s2[i];

        // c로 전이 가능할 때까지 접미사 링크를 타고 올라감
        while (cur != 0 && !sam.st[cur].next.count(c)) {
            cur = sam.st[cur].link;
            curLen = sam.st[cur].len;
        }

        if (sam.st[cur].next.count(c)) {
            cur = sam.st[cur].next[c];
            curLen++;
        }

        if (curLen > bestLen) {
            bestLen = curLen;
            bestEnd = i; // s2에서 공통 부분 문자열이 끝나는 위치
        }
    }

    return s2.substr(bestEnd - bestLen + 1, bestLen);
}

int main() {
    string s = "abcde";
    cout << "\"" << s << "\"의 서로 다른 부분 문자열 수: "
         << countDistinctSubstrings(s) << "\n"; // 15

    string s1 = "ababc", s2 = "babcab";
    cout << "\"" << s1 << "\"와 \"" << s2 << "\"의 최장 공통 부분 문자열: \""
         << longestCommonSubstring(s1, s2) << "\"\n"; // "babc"

    return 0;
}
```

### 내부 동작 시각화: "abb" 처리 과정

```
초기: root(len=0, link=-1)

extend('a'):
  cur=1 (len=1), p=root
  root->next['a'] = 1
  cur.link = root
  상태: root -a-> 1

extend('b'):
  cur=2 (len=2), p=1 (last)
  1->next['b'] = 2, p=root
  root->next['b'] = 2, p=-1
  cur.link = root
  상태: root -b-> 2 -? ; 1 -b-> 2

extend('b'):
  cur=3 (len=3), p=2 (last)
  2->next['b'] = 3, p=root
  root->next['b'] = 2 (이미 존재!) → q=2
  root.len+1=1, q.len=2 → 1 ≠ 2 → clone 필요!
  clone=4 (len=1, link=root, next=2의 next={})
  root->next['b'] = clone=4
  2.link = 4, 3.link = 4
  최종: root -a-> 1 -b-> 3
        root -b-> 4(clone) → 2 -b-> 3
        접미사 링크: 1→root, 2→4, 3→4, 4→root
```

---

## 주의사항 및 팁

### 1. clone 상태 처리
Clone은 `cnt`를 0으로 초기화해야 한다. clone은 새로운 문자를 추가한 것이 아니라 기존 상태를 분리한 것이므로, 출현 횟수 계산 시 직접적인 기여가 없다.

### 2. 메모리 최적화
`map<char, int>` 대신 알파벳 크기가 작을 때(예: a-z만 사용) `int next[26]`으로 선언하면 캐시 친화적이고 훨씬 빠르다.

```cpp
struct State {
    int len, link;
    int next[26]; // map 대신 배열 사용
    State() : len(0), link(-1) { fill(next, next + 26, -1); }
};
```

### 3. 출현 횟수 vs. 부분 문자열 수
- `cnt` 계산: 각 상태의 문자열이 원본에서 몇 번 끝나는지 → 접미사 링크 트리에서 리프에서 루트 방향으로 전파
- 서로 다른 부분 문자열 수: 각 상태의 기여 = `len - link.len`

### 4. 시간 복잡도 주의
`extend` 함수 내부의 while 루프는 worst case에서 O(n) 번 반복할 수 있지만, 전체 `n`번의 `extend` 호출에 대한 총 반복 횟수는 상각(amortized) O(n)임이 증명된다. 이는 `last`의 `len` 값이 단조 증가하고 접미사 링크를 타고 올라가는 횟수가 총합 O(n)이기 때문이다.

### 5. 다중 문자열 처리
여러 문자열의 접미사 자동자를 구축하려면 구분자(예: `$`, `#`)를 삽입하거나, 일반화 접미사 자동자(Generalized SAM)를 사용한다. 각 문자열 처리 후 `last = 0`으로 리셋하는 방식으로 단순 구현이 가능하다.

```cpp
// 일반화 SAM: 새 문자열 시작 시 last를 root로 리셋
void addString(const string& s) {
    last = 0; // root로 돌아감
    for (char c : s) {
        if (!st[last].next.count(c)) {
            extend(c);
        } else {
            int q = st[last].next[c];
            if (st[last].len + 1 == st[q].len) {
                last = q;
            } else {
                int clone = st.size();
                st.push_back({st[last].len + 1, st[q].link, st[q].next, 0, true});
                while (last != -1 && st[last].next.count(c) && st[last].next[c] == q) {
                    st[last].next[c] = clone;
                    last = st[last].link;
                }
                st[q].link = clone;
                last = clone;
            }
        }
    }
}
```

### 6. 경쟁 프로그래밍 팁
- SAM으로 해결할 수 있는 대표 문제: 서로 다른 부분 문자열 수, K번째 부분 문자열, 가장 짧은 비등장 부분 문자열, 두 문자열의 LCS
- SAM + DP 조합: 접미사 링크 트리 위에서 DP를 수행하면 복잡한 문자열 문제를 해결할 수 있다
- `cnt`를 올바르게 계산하면 각 상태의 부분 문자열이 몇 번 등장하는지 O(n)에 구할 수 있다

## 참고 자료
- [CP-Algorithms: Suffix Automaton](https://cp-algorithms.com/string/suffix-automaton.html)
- [Wikipedia: Suffix automaton](https://en.wikipedia.org/wiki/Suffix_automaton)
- [Codeforces Tutorial: SAM Problems](https://codeforces.com/blog/entry/80913)
- [An Illustrated Tutorial on Suffix Automata (Medium)](https://medium.com/@olivermatislill/an-illustrated-tutorial-on-suffix-automata-d7ac89f06cc4)
