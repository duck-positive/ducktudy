---
layout: post
title: "Aho-Corasick 알고리즘 완전 정복: 다중 패턴 문자열 검색의 선형 시간 해법"
date: 2026-10-05
categories: [cs, computer-science]
tags: [algorithm, string-matching, trie, automaton, aho-corasick, pattern-matching]
---

## 개요

텍스트에서 하나의 패턴을 찾는 문제는 KMP(Knuth-Morris-Pratt) 알고리즘으로 O(n + m) 시간에 해결할 수 있다. 그런데 수천 개의 패턴을 동시에 찾아야 한다면 어떻게 해야 할까? 바이러스 백신 스캐너는 수십만 개의 악성코드 시그니처를 동시에 파일 내에서 탐색한다. 네트워크 침입 탐지 시스템(IDS)은 패킷 스트림에서 수백 개의 위험 패턴을 실시간으로 찾는다.

단순하게 패턴마다 KMP를 반복 실행하면 O(n × k + Σm_i) 시간이 걸린다. k가 10만이고 n이 1GB 텍스트라면 현실적으로 불가능하다.

**Aho-Corasick 알고리즘**은 1975년 Alfred V. Aho와 Margaret J. Corasick이 발표한 알고리즘으로, k개의 패턴을 유한 오토마톤(finite automaton)으로 컴파일한 뒤 텍스트를 단 한 번만 스캔하여 모든 패턴의 등장 위치를 O(n + Σm_i + 결과 수) 시간에 찾는다.

---

## 왜 필요한가

### 문제의 규모

- **안티바이러스**: ClamAV는 수십만 개의 바이러스 시그니처를 동시에 스캔
- **검색 엔진**: Elasticsearch의 다중 키워드 하이라이팅
- **네트워크 IDS**: Snort/Suricata의 패킷 페이로드 패턴 매칭
- **생물정보학**: DNA 서열에서 수천 개의 유전자 모티프 동시 탐색
- **스팸 필터**: 수백 개의 금지 키워드 동시 감지

### 기존 방법의 한계

패턴이 k개이고 텍스트 길이가 n, 패턴 평균 길이가 m이라 할 때:

| 방법 | 시간 복잡도 | 설명 |
|------|------------|------|
| 단순 순회 | O(n × k × m) | 각 위치마다 각 패턴 비교 |
| KMP 반복 | O((n + m) × k) | 패턴별 KMP 실행 |
| **Aho-Corasick** | **O(n + Σm_i + 결과 수)** | 한 번의 텍스트 스캔 |

---

## 핵심 개념: 세 가지 함수

Aho-Corasick 오토마톤은 세 가지 함수로 구성된다.

### 1. Goto 함수 (트라이)

모든 패턴을 트라이(Trie) 자료구조로 삽입한다. 루트에서 시작하여 패턴의 각 문자를 따라가는 경로가 트라이를 이룬다.

```
패턴: ["he", "she", "his", "hers"]

        root
       /    \
      h      s
     / \      \
    e   i      h
    |   |      |
    r   s      e
    |
    s
```

각 노드는 현재까지 매칭된 문자열의 접두사를 나타낸다.

### 2. Failure 함수 (실패 링크)

KMP의 실패 함수(failure function)를 트라이 전체로 일반화한 것이다.

**failure[v]**: 노드 v에서 나타내는 문자열의 가장 긴 **진접두사**이면서 동시에 어떤 패턴의 접두사인 문자열을 나타내는 노드.

쉽게 말해, 현재 상태에서 입력 문자가 매칭에 실패했을 때 되돌아갈 노드다.

- 루트의 자식들의 failure 링크는 루트를 가리킨다.
- 깊이 > 1인 노드는 BFS로 계산한다.

### 3. Output 함수 (출력 함수)

노드 v에서 매칭이 완료되는 패턴들의 집합. 단, failure 링크를 따라가다가 출력이 있는 노드들의 패턴도 포함한다 (output 링크 체인).

---

## 실제 구현 예제

### 예제 1: Python으로 구현하는 Aho-Corasick

```python
from collections import deque

class AhoCorasick:
    def __init__(self):
        # 각 노드: {char: next_node_id}
        self.goto = [{}]
        # 실패 링크
        self.fail = [0]
        # 출력: 각 노드에서 매칭되는 패턴 인덱스들
        self.output = [[]]

    def add_pattern(self, pattern: str, pattern_id: int):
        """트라이에 패턴 삽입"""
        cur = 0
        for ch in pattern:
            if ch not in self.goto[cur]:
                self.goto[cur][ch] = len(self.goto)
                self.goto.append({})
                self.fail.append(0)
                self.output.append([])
            cur = self.goto[cur][ch]
        self.output[cur].append(pattern_id)

    def build(self):
        """BFS로 실패 링크 및 출력 링크 계산"""
        q = deque()
        # 루트의 직접 자식들: 실패 링크 = 루트(0)
        for ch, nxt in self.goto[0].items():
            self.fail[nxt] = 0
            q.append(nxt)

        while q:
            u = q.popleft()
            for ch, v in self.goto[u].items():
                # v의 실패 링크 계산
                f = self.fail[u]
                while f != 0 and ch not in self.goto[f]:
                    f = self.fail[f]
                self.fail[v] = self.goto[f].get(ch, 0)
                if self.fail[v] == v:
                    self.fail[v] = 0
                # 출력 링크 체인: fail 노드의 출력도 포함
                self.output[v] = self.output[v] + self.output[self.fail[v]]
                q.append(v)

    def search(self, text: str):
        """
        텍스트에서 모든 패턴 매칭 결과 반환
        Returns: list of (end_position, pattern_id)
        """
        results = []
        cur = 0
        for i, ch in enumerate(text):
            # 현재 상태에서 ch로 이동 불가능하면 실패 링크를 따라감
            while cur != 0 and ch not in self.goto[cur]:
                cur = self.fail[cur]
            cur = self.goto[cur].get(ch, 0)
            # 현재 노드의 출력 수집
            for pid in self.output[cur]:
                results.append((i, pid))
        return results


# ── 사용 예시 ──────────────────────────────────────────
patterns = ["he", "she", "his", "hers"]
ac = AhoCorasick()
for i, p in enumerate(patterns):
    ac.add_pattern(p, i)
ac.build()

text = "ushers"
matches = ac.search(text)
for end_pos, pid in matches:
    p = patterns[pid]
    start = end_pos - len(p) + 1
    print(f"패턴 '{p}' 발견: [{start}:{end_pos+1}] → '{text[start:end_pos+1]}'")

# 출력:
# 패턴 'she' 발견: [1:4] → 'she'
# 패턴 'he' 발견: [2:4] → 'he'
# 패턴 'hers' 발견: [2:6] → 'hers'
```

### 예제 2: 실전 응용 - 다중 키워드 필터링

```python
class KeywordFilter:
    """
    금지어 목록으로 텍스트를 검사하고 하이라이팅하는 필터.
    Aho-Corasick을 활용하여 O(n) 시간에 처리.
    """
    def __init__(self, keywords: list[str]):
        self.keywords = keywords
        self.ac = AhoCorasick()
        for i, kw in enumerate(keywords):
            self.ac.add_pattern(kw.lower(), i)
        self.ac.build()

    def find_all(self, text: str) -> list[dict]:
        results = []
        lowered = text.lower()
        seen = set()  # 중복 제거
        for end_pos, pid in self.ac.search(lowered):
            kw = self.keywords[pid]
            start = end_pos - len(kw) + 1
            key = (start, end_pos)
            if key not in seen:
                seen.add(key)
                results.append({
                    "keyword": kw,
                    "start": start,
                    "end": end_pos + 1,
                    "matched": text[start:end_pos + 1],
                })
        return sorted(results, key=lambda x: x["start"])

    def highlight(self, text: str, mark="**") -> str:
        """매칭된 키워드를 마크업으로 감싸기"""
        matches = self.find_all(text)
        if not matches:
            return text
        result = []
        prev = 0
        for m in matches:
            result.append(text[prev:m["start"]])
            result.append(f"{mark}{m['matched']}{mark}")
            prev = m["end"]
        result.append(text[prev:])
        return "".join(result)


# 사용 예시
banned_words = ["spam", "casino", "free money", "click here"]
f = KeywordFilter(banned_words)

email_body = "Get Free Money now! Click Here for Casino deals."
print(f.highlight(email_body))
# → "Get **Free Money** now! **Click Here** for **Casino** deals."

print(f.find_all(email_body))
# → [{'keyword': 'free money', 'start': 4, ...}, ...]
```

---

## 시간 복잡도 분석

| 단계 | 시간 복잡도 | 공간 복잡도 |
|------|------------|------------|
| 트라이 구축 | O(Σ|p_i|) | O(Σ|p_i| × σ) |
| 실패 링크 계산 | O(Σ|p_i| × σ) | O(Σ|p_i|) |
| 텍스트 검색 | O(n + 결과 수) | O(1) 추가 |

σ는 알파벳 크기(보통 256). 실패 링크 계산을 goto 함수를 완전히 채운 형태(전이 함수)로 미리 계산하면 검색 단계가 순수 O(n)이 된다.

### Goto 함수를 완전 전이 테이블로 최적화

```python
def build_goto_table(self):
    """모든 문자에 대한 전이를 미리 계산 (σ = 26 소문자 예시)"""
    ALPHA = 26
    # goto_full[node][char] = next_node
    size = len(self.goto)
    goto_full = [[0] * ALPHA for _ in range(size)]

    q = deque()
    for c in range(ALPHA):
        ch = chr(ord('a') + c)
        if ch in self.goto[0]:
            goto_full[0][c] = self.goto[0][ch]
            q.append(self.goto[0][ch])
        # goto_full[0][c]가 없으면 0(루트)으로 유지

    while q:
        u = q.popleft()
        for c in range(ALPHA):
            ch = chr(ord('a') + c)
            if ch in self.goto[u]:
                v = self.goto[u][ch]
                goto_full[u][c] = v
                q.append(v)
            else:
                # 실패 링크를 통한 전이 (간접 goto)
                goto_full[u][c] = goto_full[self.fail[u]][c]
    return goto_full
```

이렇게 하면 검색 시 while 루프 없이 단순 배열 인덱싱으로 상태 전이가 가능하여 캐시 효율도 좋아진다.

---

## 실제 구현 라이브러리

실무에서는 직접 구현보다 검증된 라이브러리를 사용하는 것이 좋다.

- **Python**: `pyahocorasick` 라이브러리 - C 확장으로 고성능
- **C++**: `AhoCorasickAutomaton` (Boost 미포함 헤더 구현 다수)
- **Java**: `org.apache.lucene.util.automaton` 또는 `ahocorasick-java`
- **Go**: `github.com/cloudflare/ahocorasick`

```python
# pyahocorasick 라이브러리 사용 예시
import ahocorasick

A = ahocorasick.Automaton()
patterns = ["he", "she", "his", "hers"]
for i, key in enumerate(patterns):
    A.add_word(key, (i, key))
A.make_automaton()

text = "ushers"
for end_idx, (pat_idx, original_value) in A.iter(text):
    start_idx = end_idx - len(original_value) + 1
    print(f"'{original_value}' at [{start_idx}:{end_idx+1}]")
```

---

## 주의사항 및 팁

### 1. 중복 매칭 처리
같은 위치에서 여러 패턴이 매칭될 수 있다(예: "he"와 "hers"가 "hers"에서 동시에 매칭). 용도에 따라 가장 긴 매칭만 선택할지, 모든 매칭을 수집할지 결정해야 한다.

### 2. 대소문자 처리
검색 전에 텍스트와 패턴 모두 같은 케이스로 정규화하면 대소문자 무관 검색이 가능하다. 단, 원본 텍스트의 위치 정보를 유지해야 하므로 정규화된 복사본으로 검색하고 원본 인덱스를 반환하는 방식을 권장한다.

### 3. 유니코드 지원
유니코드 텍스트는 알파벳 크기가 커진다. 전이 테이블을 `dict`로 표현하면 공간을 아낄 수 있지만, 속도는 배열 방식보다 느리다. `trie + dict` 방식이 메모리 절약과 속도의 균형점이다.

### 4. 스트리밍 텍스트
텍스트가 스트림으로 도착하는 경우 현재 오토마톤 상태(cur)만 유지하면 되므로 Aho-Corasick이 특히 유리하다. 청크 단위로 처리해도 상태를 이어갈 수 있어 네트워크 IDS에 이상적이다.

### 5. 패턴 추가/삭제
한 번 구축한 오토마톤에 패턴을 추가하면 전체를 재빌드해야 한다. 동적으로 패턴이 변경되는 시스템에서는 주기적 재빌드(offline rebuild)나 다층 Aho-Corasick 구조를 고려하라.

---

## 마무리

Aho-Corasick 알고리즘은 KMP를 다중 패턴으로 일반화한 우아한 알고리즘이다. 핵심은 트라이에 실패 링크를 더함으로써 단 한 번의 텍스트 스캔으로 모든 패턴을 동시에 매칭하는 것이다. 패턴 수가 늘어나도 텍스트 스캔 횟수는 단 한 번으로 고정되므로, 대규모 패턴 집합과 긴 텍스트를 다루는 시스템에서 필수 도구다.

## 참고 자료
- [Aho, A. V., & Corasick, M. J. (1975). Efficient string matching. Communications of the ACM.](https://doi.org/10.1145/360825.360855)
- [CP-Algorithms: Aho-Corasick algorithm](https://cp-algorithms.com/string/aho_corasick.html)
- [Wikipedia: Aho–Corasick algorithm](https://en.wikipedia.org/wiki/Aho%E2%80%93Corasick_algorithm)
- [pyahocorasick 라이브러리 문서](https://pyahocorasick.readthedocs.io/)
