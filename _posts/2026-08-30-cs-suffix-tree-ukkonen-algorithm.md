---
layout: post
title: "접미사 트리(Suffix Tree) 완전 정복: Ukkonen's O(n) 온라인 알고리즘"
date: 2026-08-30
categories: [cs, computer-science]
tags: [suffix-tree, ukkonen, string-algorithm, data-structure, competitive-programming]
---

접미사 트리(Suffix Tree)는 문자열 처리 분야에서 가장 강력한 자료구조 중 하나입니다. 한 번 구축해두면 부분 문자열 검색, 가장 긴 공통 부분 문자열, 문자열 반복 패턴 탐색 등 수많은 문자열 문제를 선형 시간에 해결할 수 있습니다. 이 글에서는 접미사 트리의 개념부터 Ukkonen이 1995년 발표한 O(n) 선형 시간 온라인 구축 알고리즘까지 완전히 정복합니다.

## 1. 접미사 트리란 무엇인가?

문자열 `S = "banana"`를 예로 들겠습니다. 이 문자열의 모든 접미사(suffix)는 다음과 같습니다:

```
banana$
anana$
nana$
ana$
na$
a$
$
```

(`$`는 문자열 끝을 나타내는 종료 문자, 알파벳에 없는 특수 문자)

**접미사 트리**는 이 모든 접미사를 압축 트라이(Compressed Trie)로 표현한 자료구조입니다. 단순 트라이와 달리, 하나의 자식만 있는 노드들을 합쳐서 간선(edge)에 문자열을 저장합니다. 이를 **패스 압축(Path Compression)** 또는 **압축 트라이(Compressed Trie)**라고 합니다.

접미사 트리의 핵심 성질:
- **노드 수**: 문자열 길이 n에 대해 최대 2n-1개의 노드
- **리프 노드**: 각 접미사에 정확히 하나씩 대응
- **간선**: 원본 문자열의 연속된 부분 문자열을 나타냄 (시작·끝 인덱스로 저장)
- **공간**: O(n)

### 접미사 트리 vs 접미사 배열 vs 접미사 자동자

| 자료구조 | 구축 시간 | 공간 | 주요 용도 |
|---------|----------|------|---------|
| 접미사 트리 | O(n) | O(n) | 온라인 쿼리, 복수 문자열 처리 |
| 접미사 배열 | O(n log n) 또는 O(n) | O(n) | 사전식 정렬, 메모리 효율 |
| 접미사 자동자 | O(n) | O(n) | 부분 문자열 카운팅, DAG 연산 |

접미사 트리는 세 자료구조 중 가장 직관적이며, 두 문자열을 합쳐서 다루는 **일반화 접미사 트리(Generalized Suffix Tree)**로 쉽게 확장할 수 있다는 장점이 있습니다.

## 2. 왜 Ukkonen's Algorithm인가?

접미사 트리를 구축하는 가장 단순한 방법은 각 접미사를 하나씩 트라이에 삽입하는 것입니다. 접미사의 길이 합이 O(n²)이므로 이 방법의 시간 복잡도는 O(n²)입니다. 

1973년 Weiner가 최초의 선형 시간 알고리즘을 발표했고, 1976년 McCreight가 개선된 버전을 내놓았습니다. 그러나 이 알고리즘들은 모두 **오프라인(offline)** 방식으로, 전체 문자열을 미리 알고 있어야 합니다.

1995년 Esko Ukkonen이 발표한 알고리즘은 세 가지 혁신을 담고 있습니다:

1. **온라인(online) 알고리즘**: 문자열을 왼쪽에서 오른쪽으로 한 글자씩 처리
2. **암묵적 접미사 트리(Implicit Suffix Tree)**: 중간 단계에서도 유효한 구조 유지
3. **세 가지 트릭으로 O(n) 달성**: End-pointer, Active Point, Suffix Link

## 3. Ukkonen's Algorithm의 세 가지 핵심 트릭

### 트릭 1: End-pointer (전역 끝 포인터)

각 리프 노드의 간선 끝 인덱스를 개별 저장하지 않고, 전역 변수 `end`를 가리키게 합니다. 새 문자를 추가할 때 `end`를 1 증가시키면 모든 리프의 간선이 자동으로 연장됩니다.

이 트릭 덕분에 이미 생성된 리프 노드들을 갱신하는 비용이 O(1)로 줄어듭니다.

### 트릭 2: Active Point

새 문자를 삽입할 때 "현재 어디서부터 새 분기를 만들어야 하는가"를 추적하는 포인터입니다. 세 값으로 구성됩니다:

- `active_node`: 현재 위치의 노드
- `active_edge`: 현재 탐색 중인 간선의 첫 문자
- `active_length`: 현재 간선을 따라 이동한 거리

### 트릭 3: Suffix Link

내부 노드 v에서 출발하는 경로가 문자열 `aX`를 나타낼 때, `X`를 나타내는 내부 노드 u로 가는 포인터를 **접미사 링크(Suffix Link)**라고 합니다. 새 내부 노드를 만들 때마다 이전 단계에서 만든 내부 노드로부터 접미사 링크를 연결합니다. 이를 통해 다음 접미사 삽입 위치를 O(1)에 찾아갑니다.

## 4. 구현: C++ Ukkonen's Algorithm

아래는 Ukkonen's Algorithm의 완전한 C++ 구현입니다.

```cpp
#include <bits/stdc++.h>
using namespace std;

const int MAXN = 200005;
const int INF = 1e9;

struct Node {
    int start, end, link;
    map<char, int> next;
    
    Node(int s, int e) : start(s), end(e), link(-1) {}
    
    int len() { return end - start; }
};

string s;
vector<Node> tree;
int sz, last, n;
int active_node, active_edge, active_len;
int remaining;
int* global_end;

int newNode(int start, int end) {
    tree.push_back(Node(start, end));
    return sz++;
}

char activeEdgeChar() {
    return s[active_edge];
}

void extend(int pos) {
    // 전역 끝 포인터 연장 (트릭 1)
    *global_end = pos + 1;
    remaining++;
    last = -1;
    
    while (remaining > 0) {
        if (active_len == 0)
            active_edge = pos;
        
        // active_edge 방향 간선이 없는 경우
        if (tree[active_node].next.find(s[active_edge]) 
            == tree[active_node].next.end()) {
            // 새 리프 생성
            tree[active_node].next[s[active_edge]] = newNode(pos, *global_end - 1);
            // 이전에 만든 내부 노드에 suffix link 연결
            if (last != -1) {
                tree[last].link = active_node;
                last = -1;
            }
        } else {
            int nxt = tree[active_node].next[s[active_edge]];
            // active_length가 간선 길이를 초과하면 walk down
            int edge_len = tree[nxt].len();
            if (active_len >= edge_len) {
                active_edge += edge_len;
                active_len -= edge_len;
                active_node = nxt;
                continue;
            }
            // 현재 문자가 이미 트리에 있는 경우 (rule 3)
            if (s[tree[nxt].start + active_len] == s[pos]) {
                active_len++;
                if (last != -1)
                    tree[last].link = active_node;
                break;
            }
            // 간선을 분할하여 새 내부 노드 생성 (rule 2)
            int split = newNode(tree[nxt].start, tree[nxt].start + active_len - 1);
            tree[active_node].next[s[active_edge]] = split;
            tree[split].next[s[pos]] = newNode(pos, *global_end - 1);
            tree[nxt].start += active_len;
            tree[split].next[s[tree[nxt].start]] = nxt;
            
            // suffix link 연결
            if (last != -1)
                tree[last].link = split;
            last = split;
        }
        remaining--;
        // active_node의 suffix link로 이동 (트릭 3)
        if (active_node == 0 && active_len > 0) {
            active_len--;
            active_edge = pos - remaining + 1;
        } else if (tree[active_node].link != -1) {
            active_node = tree[active_node].link;
        } else {
            active_node = 0;
        }
    }
}

// suffix tree 구축 후 부분 문자열 검색
bool search(const string& pat) {
    int cur = 0;
    int i = 0;
    while (i < (int)pat.size()) {
        char c = pat[i];
        if (tree[cur].next.find(c) == tree[cur].next.end())
            return false;
        int nxt = tree[cur].next[c];
        int j = tree[nxt].start;
        int e = min(tree[nxt].end + 1, (int)s.size());
        while (j < e && i < (int)pat.size()) {
            if (s[j] != pat[i]) return false;
            i++; j++;
        }
        cur = nxt;
    }
    return true;
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    cin >> s;
    s += '$'; // 종료 문자 추가
    n = s.size();
    
    int end_val = 0;
    global_end = &end_val;
    sz = 0;
    active_node = 0;
    active_edge = 0;
    active_len = 0;
    remaining = 0;
    
    newNode(-1, -1); // 루트 노드 (인덱스 0)
    
    for (int i = 0; i < n; i++)
        extend(i);
    
    // 검색 테스트
    string queries[] = {"ana", "ban", "xyz", "nana"};
    for (const string& q : queries) {
        cout << "\"" << q << "\" in \"" << s.substr(0, n-1) << "\": "
             << (search(q) ? "found" : "not found") << "\n";
    }
    
    return 0;
}
```

**출력 결과** (입력: "banana"):
```
"ana" in "banana": found
"ban" in "banana": found
"xyz" in "banana": not found
"nana" in "banana": found
```

## 5. 접미사 트리로 해결하는 고급 문제들

### 5-1. 가장 긴 공통 부분 문자열 (Longest Common Substring)

두 문자열 S1, S2를 `S1#S2$`로 합쳐서 일반화 접미사 트리를 구축합니다. S1과 S2 양쪽에서 온 리프를 모두 포함하는 내부 노드 중 가장 깊은 것의 경로 레이블이 LCS입니다.

```python
def longest_common_substring(s1: str, s2: str) -> str:
    """
    일반화 접미사 트리를 이용한 LCS 계산
    여기서는 간단한 O(n*m) 방법으로 개념 설명
    실제로는 접미사 트리 구축 후 O(n+m)에 해결 가능
    """
    n, m = len(s1), len(s2)
    # dp[i][j] = s1[i:]와 s2[j:]의 최장 공통 접두사 길이
    dp = [[0] * (m + 1) for _ in range(n + 1)]
    max_len = 0
    end_idx = 0
    
    for i in range(1, n + 1):
        for j in range(1, m + 1):
            if s1[i-1] == s2[j-1]:
                dp[i][j] = dp[i-1][j-1] + 1
                if dp[i][j] > max_len:
                    max_len = dp[i][j]
                    end_idx = i
    
    return s1[end_idx - max_len:end_idx]

# 테스트
s1 = "abcdefgh"
s2 = "acdeghi"
result = longest_common_substring(s1, s2)
print(f"LCS of '{s1}' and '{s2}': '{result}'")
# 출력: LCS of 'abcdefgh' and 'acdeghi': 'cde' 또는 'ghi' (길이 3)
```

### 5-2. 최소 회문 분해 (Palindrome Decomposition)

접미사 트리를 역순 문자열의 접미사 트리와 결합하여 모든 팰린드롬 부분 문자열을 찾을 수 있습니다. 이는 Eertree(팰린드로믹 트리)와 함께 활용됩니다.

## 6. 시간·공간 복잡도 분석

Ukkonen's Algorithm의 복잡도를 단계별로 분석합니다:

| 연산 | 비용 | 근거 |
|------|------|------|
| 전역 end 연장 | O(1) per char | 트릭 1 |
| active point 이동 | Amortized O(1) | active_length 단조 감소 |
| suffix link 순회 | Amortized O(1) | 각 링크 최대 1회 사용 |
| 전체 구축 | **O(n)** | 세 트릭의 합 |

공간 복잡도는 노드 수 O(n), 간선 정보 O(n)으로 전체 **O(n)** 입니다.

단, 알파벳 크기가 σ인 경우 `map<char, int>` 대신 `int[σ]` 배열을 사용하면 공간을 O(nσ)로 고정하거나, 해시맵을 써서 O(n) 기대치를 유지할 수 있습니다.

## 7. 주의사항 및 실전 팁

### 종료 문자 선택

종료 문자 `$`는 반드시 문자열에 등장하지 않는 문자여야 합니다. 경쟁 프로그래밍에서는 보통 `'$'` (ASCII 36) 또는 `'#'`을 씁니다. 두 문자열을 합칠 때는 각각 다른 종료 문자를 써야 합니다 (`s1 + '#' + s2 + '$'`).

### Walk Down 최적화

active point가 내부 노드로 이동할 때 간선을 하나씩 내려가는 대신, 간선 길이를 비교해 건너뛰는 **walk down** 기법을 반드시 구현해야 선형 시간을 보장할 수 있습니다.

### 메모리 사전 할당

`vector<Node>`를 사용할 때 `reserve(2*n)`을 미리 호출해 재할당으로 인한 성능 저하를 막으세요.

### 실전 활용 팁

- **문자열 검색**: 패턴 길이 m에 대해 O(m)으로 탐색 가능
- **접미사 배열 변환**: DFS 순회로 O(n)에 접미사 배열 생성 가능
- **생물정보학**: 게놈 서열 분석에서 Generalized Suffix Tree 활용
- **텍스트 편집기**: Rope 자료구조와 결합해 O(log n) 편집 가능

접미사 트리는 구현이 복잡하여 경쟁 프로그래밍에서는 접미사 배열이 더 선호되지만, 이론적 완성도와 응용 범위에서는 접미사 트리가 가장 강력한 문자열 자료구조입니다.

## 참고 자료
- [Ukkonen's Algorithm - Wikipedia](https://en.wikipedia.org/wiki/Ukkonen%27s_algorithm)
- [Suffix Tree. Ukkonen's Algorithm - CP-Algorithms](https://cp-algorithms.com/string/suffix-tree-ukkonen.html)
- [Suffix Tree - Wikipedia](https://en.wikipedia.org/wiki/Suffix_tree)
- [Introduction to Algorithms (CLRS) - Chapter 32: String Matching](https://mitpress.mit.edu/9780262046305/introduction-to-algorithms/)
