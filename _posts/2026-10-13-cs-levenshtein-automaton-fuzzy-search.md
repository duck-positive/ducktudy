---
layout: post
title: "레벤슈타인 오토마톤 — 편집 거리 기반 퍼지 문자열 검색의 내부 구조"
date: 2026-10-13
categories: [cs, computer-science]
tags: [레벤슈타인, 편집거리, 퍼지검색, 오토마톤, NFA, DFA, 맞춤법검사]
---

## 개요

검색창에 "알고리듬"을 입력했을 때 "알고리즘"을 제안하거나, Elasticsearch에서 오타가 있는 단어를 검색해도 올바른 결과를 찾아주는 기능 — 이 모든 것의 배경에는 **레벤슈타인 오토마톤(Levenshtein Automaton)**이 있습니다.

레벤슈타인 오토마톤은 주어진 단어와 편집 거리(Edit Distance)가 k 이하인 모든 문자열을 정확히 인식하는 유한 오토마톤입니다. 단순히 편집 거리를 반복 계산하는 것보다 훨씬 효율적으로 대규모 사전에서 퍼지 검색을 수행할 수 있습니다.

---

## 레벤슈타인 편집 거리 복습

편집 거리(Levenshtein Distance)는 한 문자열을 다른 문자열로 변환하는 데 필요한 최소 편집 연산 수입니다. 허용 연산은:

- **삽입(Insert)**: 문자 하나 추가
- **삭제(Delete)**: 문자 하나 제거
- **대체(Replace)**: 문자 하나를 다른 문자로 교체

```python
def levenshtein_distance(s1: str, s2: str) -> int:
    """동적 프로그래밍으로 레벤슈타인 거리 계산 — O(|s1| × |s2|)"""
    m, n = len(s1), len(s2)
    dp = [[0] * (n + 1) for _ in range(m + 1)]

    # 초기화: 빈 문자열과의 편집 거리
    for i in range(m + 1):
        dp[i][0] = i
    for j in range(n + 1):
        dp[0][j] = j

    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if s1[i - 1] == s2[j - 1]:
                dp[i][j] = dp[i - 1][j - 1]  # 일치: 비용 없음
            else:
                dp[i][j] = 1 + min(
                    dp[i - 1][j],      # 삭제
                    dp[i][j - 1],      # 삽입
                    dp[i - 1][j - 1]   # 대체
                )

    return dp[m][n]


# 테스트
print(levenshtein_distance("kitten", "sitting"))  # 3
print(levenshtein_distance("알고리즘", "알고리듬"))    # 1
print(levenshtein_distance("", "abc"))             # 3
```

이 방식으로 대규모 사전(예: 100만 단어)에서 퍼지 검색을 하면 O(|query| × |word|) 계산을 단어마다 반복해야 합니다. 레벤슈타인 오토마톤은 이를 훨씬 빠르게 만듭니다.

---

## 레벤슈타인 오토마톤의 핵심 아이디어

레벤슈타인 오토마톤은 **쿼리 단어 `p`**와 **최대 허용 편집 거리 `k`**를 바탕으로 구성됩니다. 이 오토마톤은 정확히 `edit_distance(p, w) ≤ k`인 모든 문자열 `w`를 인식합니다.

### NFA (비결정론적 유한 오토마톤)로 구성

상태 `(i, e)`는 다음을 의미합니다:
- `i`: 쿼리 패턴의 현재 매칭 위치 (0 ≤ i ≤ |p|)
- `e`: 지금까지의 편집 비용 (0 ≤ e ≤ k)

전이 규칙:
- **일치**: `(i, e) --p[i]--> (i+1, e)` (비용 0)
- **삽입**: `(i, e) --*--> (i, e+1)` (어떤 문자든)
- **삭제**: `(i, e) --ε--> (i+1, e+1)` (ε-전이)
- **대체**: `(i, e) --*--> (i+1, e+1)` (일치하지 않는 문자)

```python
from typing import FrozenSet, Dict, Set, Tuple

class LevenshteinNFA:
    """쿼리 패턴 p에 대한 레벤슈타인 NFA (편집 거리 k 이하)"""

    def __init__(self, pattern: str, max_edits: int):
        self.pattern = pattern
        self.max_edits = max_edits
        self.n = len(pattern)

    def start_state(self) -> FrozenSet[Tuple[int, int]]:
        """NFA의 시작 상태 집합 (ε-closure 포함)"""
        # (position, edits) 쌍의 집합
        states = set()
        # ε-전이(삭제)로 도달 가능한 초기 상태들
        for e in range(self.max_edits + 1):
            if e <= self.n:
                states.add((e, e))  # 처음 e글자를 삭제로 처리
        return frozenset(states)

    def transitions(self, states: FrozenSet[Tuple[int, int]], char: str) -> FrozenSet[Tuple[int, int]]:
        """현재 상태 집합에서 문자 char를 읽었을 때의 다음 상태 집합"""
        next_states = set()

        for (pos, edits) in states:
            if pos > self.n:
                continue

            # 현재 패턴 문자와 비교
            matched = (pos < self.n and self.pattern[pos] == char)

            # 1. 일치: 비용 없이 전진
            if matched and edits <= self.max_edits:
                next_states.add((pos + 1, edits))

            # 2. 대체: 다른 문자 → 비용 1
            if not matched and edits + 1 <= self.max_edits:
                next_states.add((pos + 1, edits + 1))

            # 3. 삽입: 패턴 위치 유지, 비용 1 (입력 문자 소비)
            if edits + 1 <= self.max_edits:
                next_states.add((pos, edits + 1))

        # ε-closure (삭제 연산: 입력 소비 없이 패턴 전진)
        expanded = set(next_states)
        for (pos, edits) in next_states:
            e = edits + 1
            p = pos + 1
            while e <= self.max_edits and p <= self.n:
                expanded.add((p, e))
                e += 1
                p += 1

        return frozenset(expanded)

    def is_accepting(self, states: FrozenSet[Tuple[int, int]]) -> bool:
        """현재 상태 집합이 수락 상태를 포함하는지 확인"""
        for (pos, edits) in states:
            if pos == self.n and edits <= self.max_edits:
                return True
        return False

    def match(self, word: str) -> bool:
        """단어가 편집 거리 k 이하인지 판별"""
        states = self.start_state()
        for char in word:
            states = self.transitions(states, char)
            if not states:
                return False
        return self.is_accepting(states)


# 테스트
nfa = LevenshteinNFA("algorithm", max_edits=2)
test_words = ["algorithm", "algoritm", "algortihm", "logarithm", "xyz"]
for word in test_words:
    match = nfa.match(word)
    dist = levenshtein_distance("algorithm", word)
    print(f"  '{word}': distance={dist}, match={match}")
```

---

## DFA 변환 및 사전 퍼지 검색

NFA를 DFA로 변환하면 상태가 결정론적이 되어 각 문자당 O(1) 전이가 가능합니다. 이를 트라이(Trie) 사전과 결합하면 효율적인 퍼지 검색이 가능합니다.

```python
class FuzzySearchDict:
    """트라이 + 레벤슈타인 NFA 기반 퍼지 사전 검색"""

    def __init__(self, words: list):
        self.trie = {}
        for word in words:
            node = self.trie
            for char in word:
                node = node.setdefault(char, {})
            node['$'] = True  # 단어 끝 표시

    def search(self, query: str, max_edits: int) -> list:
        """편집 거리 max_edits 이하의 모든 단어를 사전에서 검색"""
        nfa = LevenshteinNFA(query, max_edits)
        results = []
        initial_states = nfa.start_state()

        def dfs(node: dict, current_word: str, nfa_states: FrozenSet):
            """트라이를 DFS로 탐색하며 NFA 상태를 동시에 관리"""
            if not nfa_states:
                return

            # 현재 노드가 단어 끝이고 NFA가 수락 상태면 결과에 추가
            if '$' in node and nfa.is_accepting(nfa_states):
                results.append(current_word)

            # 자식 노드 탐색
            for char, child_node in node.items():
                if char == '$':
                    continue
                next_states = nfa.transitions(nfa_states, char)
                if next_states:  # 유효한 전이가 있을 때만 탐색
                    dfs(child_node, current_word + char, next_states)

        dfs(self.trie, "", initial_states)
        return sorted(results)


# 한국어/영어 혼합 사전 예시
dictionary = [
    "algorithm", "algorithms", "logarithm", "altruism",
    "alligator", "alignment", "alimentation",
    "python", "patron", "proton", "photon",
]

fsd = FuzzySearchDict(dictionary)

print("퍼지 검색 결과:")
print(f"  'algoritm' (편집거리 ≤1): {fsd.search('algoritm', 1)}")
print(f"  'algoritm' (편집거리 ≤2): {fsd.search('algoritm', 2)}")
print(f"  'pyton' (편집거리 ≤1): {fsd.search('pyton', 1)}")
print(f"  'proton' (편집거리 ≤2): {fsd.search('proton', 2)}")
```

---

## 시간 복잡도 분석

| 방법 | 전처리 | 단어당 검색 | 전체 사전 검색 |
|------|--------|------------|----------------|
| 단순 반복 (DP) | O(1) | O(\|q\| × \|w\|) | O(N × \|q\| × avg_w) |
| 레벤슈타인 오토마톤 + 트라이 | O(\|q\| × k²) | — | O(\|q\| × k × trie_nodes) |

레벤슈타인 오토마톤의 NFA 상태 수는 O(|q| × k)이며, DFA로 변환하면 최악의 경우 지수적으로 증가할 수 있지만 실제로는 훨씬 작습니다. Lucene/Elasticsearch는 `k ≤ 2`일 때 DFA를 미리 생성하여 사용합니다.

---

## 실전 활용: Elasticsearch Fuzzy Query

```json
{
  "query": {
    "fuzzy": {
      "title": {
        "value": "algoritm",
        "fuzziness": "AUTO",
        "max_expansions": 50,
        "prefix_length": 1,
        "transpositions": true
      }
    }
  }
}
```

`fuzziness: "AUTO"`는 단어 길이에 따라 자동으로 허용 편집 거리를 결정합니다:
- 1-2글자: 0 (정확 일치)
- 3-5글자: 1
- 6글자 이상: 2

`transpositions: true`는 인접 문자 교환도 하나의 편집 연산으로 처리하는 **Damerau-Levenshtein** 거리를 사용합니다 (예: "ab" → "ba"가 거리 1).

---

## 주의사항 및 팁

**1. 편집 거리 k의 선택**: k=2 이상이면 너무 많은 결과가 나올 수 있습니다. 실제 맞춤법 검사기는 k=1~2가 적당하며, 단어 길이에 따라 동적으로 설정하는 것이 좋습니다.

**2. prefix_length 최적화**: Elasticsearch의 `prefix_length` 파라미터처럼, 앞 몇 글자는 정확히 일치해야 한다는 제약을 추가하면 검색 공간을 크게 줄일 수 있습니다.

**3. 한국어 처리 주의**: 한국어는 자모 분리 여부에 따라 편집 거리가 크게 달라집니다. "알고리즘"을 자모 단위로 처리하면 "알고리듬"과의 거리가 다르게 계산됩니다. 목적에 맞는 토크나이저를 선택하세요.

**4. 메모리 vs 속도 트레이드오프**: DFA를 미리 빌드하면 검색이 빠르지만 메모리를 더 사용합니다. 대용량 사전이라면 FST(Finite State Transducer)와 결합하는 것이 효율적입니다.

---

## 정리

레벤슈타인 오토마톤은 편집 거리 계산을 유한 오토마톤으로 표현하여, 대규모 사전에서 효율적인 퍼지 문자열 검색을 가능하게 합니다. 트라이와 결합하면 NFA 상태를 관리하면서 사전을 탐색할 수 있어, 단순한 편집 거리 반복 계산에 비해 훨씬 빠른 성능을 달성합니다. 맞춤법 검사기, 검색 엔진, 자동 완성 기능 등 실용적인 분야에서 광범위하게 활용됩니다.

## 참고 자료
- [Wikipedia: Levenshtein automaton](https://en.wikipedia.org/wiki/Levenshtein_automaton)
- [Lucene FuzzyQuery Documentation](https://lucene.apache.org/core/9_0_0/core/org/apache/lucene/search/FuzzyQuery.html)
- [Elasticsearch Fuzzy Query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-fuzzy-query.html)
- [Damn Cool Algorithms: Levenshtein Automata (Nick Johnson)](https://blog.notdot.net/2010/07/Damn-Cool-Algorithms-Levenshtein-Automata)
