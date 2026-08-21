---
layout: post
title: "Aho-Corasick 알고리즘 완전 정복: 트라이와 실패 링크로 구현하는 다중 패턴 동시 탐색"
date: 2026-08-21
categories: [cs, computer-science]
tags: [string, pattern-matching, Aho-Corasick, trie, KMP, algorithm, 문자열매칭]
---

## 개념 설명: 하나의 패스로 수천 개 패턴을 동시에 찾다

텍스트에서 특정 단어를 찾을 때 KMP나 Boyer-Moore 같은 단일 패턴 매칭 알고리즘은 패턴 하나당 한 번의 선형 탐색이 필요하다. 패턴이 1,000개라면 텍스트를 1,000번 스캔해야 한다. 바이러스 시그니처가 수백만 개에 달하는 안티바이러스 소프트웨어나 수십만 개의 금칙어를 검사하는 채팅 필터가 이런 방식을 쓴다면 실시간 처리는 불가능하다.

**Aho-Corasick 알고리즘**은 이 문제를 완전히 해결한다. 1975년 Alfred V. Aho와 Margaret J. Corasick이 발표한 이 알고리즘은 모든 패턴으로 트라이(Trie)를 구성하고, 여기에 **실패 링크(Failure Link)** 를 추가해 유한 오토마톤(Finite Automaton)을 만든다. 이 오토마톤으로 텍스트를 단 한 번만 스캔하면 모든 패턴의 출현 위치를 O(n + m + k) 시간에 찾아낼 수 있다.

- **n**: 텍스트 길이
- **m**: 모든 패턴 길이의 합
- **k**: 총 매칭 횟수

패턴이 아무리 많아도 텍스트 탐색 비용은 O(n + k)에 고정된다는 점이 핵심이다.

### 세 가지 함수로 이해하는 Aho-Corasick

원 논문은 알고리즘을 세 함수로 정의한다:

1. **goto 함수**: 현재 상태에서 문자를 읽었을 때 전이할 다음 상태를 정의한다. 트라이의 간선에 해당한다.
2. **failure 함수**: goto 함수에 정의된 전이가 없을 때 돌아갈 "접미사 일치" 상태를 정의한다. KMP의 실패 함수와 유사하다.
3. **output 함수**: 현재 상태에서 매칭이 완료된 패턴의 집합을 반환한다.

---

## 왜 필요한가?

### 단순 접근법의 한계

패턴 P₁, P₂, …, Pₖ를 텍스트 T에서 찾는 가장 단순한 방법은 각 패턴에 대해 KMP를 독립적으로 실행하는 것이다. 시간 복잡도는 O(k·n + m)이 된다.

- 안티바이러스: 패턴 100만 개, 텍스트 10MB → 100만 × 10MB = 10TB 연산
- 스팸 필터: 금칙어 50만 개를 실시간 채팅 메시지에 적용

Aho-Corasick은 이를 O(n + m + k)로 줄인다. 패턴 수가 늘어도 텍스트 탐색 비용은 그대로다.

### 실제 사용 사례

| 시스템 | 활용 |
|--------|------|
| **Snort** (IDS/IPS) | 네트워크 패킷에서 수천 개의 침입 시그니처 동시 탐색 |
| **Grep / fgrep** | 원조 Unix `fgrep`이 Aho-Corasick 기반 |
| **안티바이러스** | 악성코드 시그니처 수백만 개 동시 매칭 |
| **채팅 필터** | 금칙어, 스팸 패턴 실시간 검사 |
| **검색 엔진** | 역색인 구축 시 다중 용어 동시 인식 |

---

## 실제 구현 예제

### 구현 1: Aho-Corasick 오토마톤 (Python)

```python
from collections import deque

class AhoCorasick:
    def __init__(self):
        # 상태 0이 루트
        self.goto = [{}]      # goto[state][char] = next_state
        self.fail = [0]       # failure link
        self.output = [[]]    # 각 상태에서 매칭 완료된 패턴 인덱스 목록

    def _new_state(self):
        self.goto.append({})
        self.fail.append(0)
        self.output.append([])
        return len(self.goto) - 1

    def add_pattern(self, pattern, idx):
        """트라이에 패턴 삽입"""
        cur = 0
        for ch in pattern:
            if ch not in self.goto[cur]:
                self.goto[cur][ch] = self._new_state()
            cur = self.goto[cur][ch]
        self.output[cur].append(idx)

    def build(self):
        """BFS로 failure link 구성 — KMP fail 함수의 다차원 확장"""
        queue = deque()
        # 깊이 1 노드의 failure link는 루트(0)
        for ch, s in self.goto[0].items():
            self.fail[s] = 0
            queue.append(s)

        while queue:
            r = queue.popleft()
            for ch, s in self.goto[r].items():
                queue.append(s)
                # s의 failure link 계산
                state = self.fail[r]
                while state != 0 and ch not in self.goto[state]:
                    state = self.fail[state]
                self.fail[s] = self.goto[state].get(ch, 0)
                if self.fail[s] == s:
                    self.fail[s] = 0  # 자기 참조 방지
                # output 병합: failure link가 가리키는 상태의 output도 포함
                self.output[s] = self.output[s] + self.output[self.fail[s]]

    def search(self, text, patterns):
        """
        텍스트에서 모든 패턴 검색
        반환: [(시작_위치, 패턴_문자열), ...]
        """
        results = []
        cur = 0
        for i, ch in enumerate(text):
            # goto 전이 없으면 failure link 따라가기
            while cur != 0 and ch not in self.goto[cur]:
                cur = self.fail[cur]
            cur = self.goto[cur].get(ch, 0)
            # 현재 상태에서 매칭 완료된 패턴 기록
            for idx in self.output[cur]:
                start = i - len(patterns[idx]) + 1
                results.append((start, patterns[idx]))
        return results


# ── 사용 예시 ─────────────────────────────────────────────
patterns = ["he", "she", "his", "hers"]
text = "ahershers"
#        0123456789

ac = AhoCorasick()
for i, p in enumerate(patterns):
    ac.add_pattern(p, i)
ac.build()

matches = ac.search(text, patterns)
for pos, pat in sorted(matches):
    print(f"위치 {pos}: '{pat}' 발견")
# 출력:
# 위치 1: 'he' 발견
# 위치 1: 'hers' 발견
# 위치 5: 'he' 발견
# 위치 5: 'hers' 발견
```

### 구현 2: 배열 기반 고성능 구현 (C++)

딕셔너리 기반 Python 구현은 이해하기 쉽지만 실전에서는 알파벳 크기를 배열 인덱스로 고정해 성능을 높인다.

```cpp
#include <bits/stdc++.h>
using namespace std;

const int MAXN = 100005;
const int ALPHA = 26;  // 소문자 알파벳

struct AhoCorasick {
    int next[MAXN][ALPHA], fail[MAXN], output[MAXN];
    int cnt;

    void init() {
        cnt = 0;
        memset(next[0], -1, sizeof(next[0]));
        output[0] = 0;
    }

    int newNode() {
        ++cnt;
        memset(next[cnt], -1, sizeof(next[cnt]));
        output[cnt] = 0;
        fail[cnt] = 0;
        return cnt;
    }

    void addPattern(const string& s, int id) {
        int cur = 0;
        for (char c : s) {
            int ch = c - 'a';
            if (next[cur][ch] == -1)
                next[cur][ch] = newNode();
            cur = next[cur][ch];
        }
        output[cur] |= (1 << id);  // 비트마스크로 패턴 ID 기록
    }

    void build() {
        queue<int> q;
        for (int c = 0; c < ALPHA; c++) {
            if (next[0][c] == -1)
                next[0][c] = 0;  // 루트 미정의 전이는 루트로
            else {
                fail[next[0][c]] = 0;
                q.push(next[0][c]);
            }
        }
        while (!q.empty()) {
            int r = q.front(); q.pop();
            output[r] |= output[fail[r]];  // output 병합
            for (int c = 0; c < ALPHA; c++) {
                if (next[r][c] == -1) {
                    next[r][c] = next[fail[r]][c];  // 고정점 방식
                } else {
                    fail[next[r][c]] = next[fail[r]][c];
                    q.push(next[r][c]);
                }
            }
        }
    }

    vector<pair<int,int>> search(const string& text, const vector<string>& patterns) {
        vector<pair<int,int>> res;
        int cur = 0;
        for (int i = 0; i < (int)text.size(); i++) {
            cur = next[cur][text[i] - 'a'];
            if (output[cur]) {
                for (int j = 0; j < (int)patterns.size(); j++) {
                    if (output[cur] & (1 << j)) {
                        res.push_back({i - (int)patterns[j].size() + 1, j});
                    }
                }
            }
        }
        return res;
    }
} ac;

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);

    ac.init();
    vector<string> patterns = {"he", "she", "his", "hers"};
    for (int i = 0; i < (int)patterns.size(); i++)
        ac.addPattern(patterns[i], i);
    ac.build();

    string text = "ahershers";
    auto matches = ac.search(text, patterns);
    for (auto [pos, idx] : matches)
        cout << "위치 " << pos << ": '" << patterns[idx] << "' 발견\n";
    return 0;
}
```

C++ 구현의 핵심 개선점은 **goto 함수를 완전 DFA로 전환**하는 것이다. 미정의 전이를 `next[fail[r]][c]`로 채워넣어 탐색 시 failure link를 따라가는 while 루프를 제거하고 항상 O(1)에 전이한다.

---

## 알고리즘 상세 분석

### 실패 링크 계산의 직관

실패 링크는 **"현재 매칭된 접두사의 가장 긴 진짜 접미사이면서 어떤 패턴의 접두사인 상태"** 를 가리킨다. KMP의 실패 함수와 정확히 동일한 개념을 트라이 전체로 확장한 것이다.

예를 들어 `"she"`와 `"he"` 두 패턴이 있을 때, `"she"`의 마지막 `e`를 처리한 상태에서는 `"he"`의 `e`에 해당하는 상태도 동시에 output에 포함된다. 왜냐하면 `"she"`는 `"he"`를 접미사로 포함하기 때문이다.

### output 링크(사전 링크)

BFS 전파 시 `output[s] |= output[fail[s]]`를 통해 failure chain 상의 모든 패턴을 현재 상태의 output으로 병합한다. 이를 **출력 링크(Output/Dictionary Link)** 라고도 부른다. 이 덕분에 탐색 시 매 상태에서 failure chain을 따라가지 않아도 된다.

---

## 주의사항 및 팁

### 1. 패턴 수가 많을 때 메모리 관리
패턴이 수백만 개라면 트라이 노드 수가 수십억에 달할 수 있다. 이때는 해시맵 기반 goto 또는 압축 트라이(Compact Trie)를 사용해야 한다.

### 2. 대소문자 무관 매칭
`ch.lower()`로 정규화하거나 알파벳 크기를 확장(a-z + A-Z = 52)해서 처리한다.

### 3. 유니코드 지원
한국어 등 다국어 패턴을 처리할 때는 알파벳 크기가 수만에 달하므로 반드시 딕셔너리(해시맵) 기반 goto를 사용해야 한다.

### 4. 검색엔진 확장: 단어 경계 처리
패턴 매칭 후 앞뒤 문자를 확인해 단어 경계를 검사하면 부분 문자열 오검출을 방지할 수 있다.

### 5. 스트리밍 텍스트 처리
오토마톤 상태를 유지하면 청크 단위로 들어오는 스트림 데이터를 처리할 수 있다. 청크 경계에서 패턴이 잘려도 `cur` 상태가 유지되므로 다음 청크에서 이어서 처리된다.

---

## 참고 자료

- [Aho–Corasick algorithm – Wikipedia](https://en.wikipedia.org/wiki/Aho%E2%80%93Corasick_algorithm)
- [Aho-Corasick algorithm – cp-algorithms.com](https://cp-algorithms.com/string/aho_corasick.html)
- [Aho-Corasick Algorithm for Pattern Searching – GeeksforGeeks](https://www.geeksforgeeks.org/dsa/aho-corasick-algorithm-pattern-searching/)
- [pyahocorasick – Python 라이브러리 공식 문서](https://pyahocorasick.readthedocs.io/)
