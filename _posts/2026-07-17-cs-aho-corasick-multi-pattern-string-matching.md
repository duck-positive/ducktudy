---
layout: post
title: "Aho-Corasick 알고리즘 완전 정복: 수천 개의 패턴을 한 번에 찾는 멀티패턴 문자열 매칭"
date: 2026-07-17
categories: [cs, computer-science]
tags: [aho-corasick, string-matching, automaton, trie, algorithms, pattern-matching]
---

## 개념 설명

문자열 검색은 소프트웨어 개발에서 가장 빈번하게 등장하는 문제 중 하나입니다. 단일 패턴을 텍스트에서 찾는 KMP(Knuth-Morris-Pratt)나 Boyer-Moore 알고리즘은 이미 O(N + M) 수준의 효율을 달성했지만, **N개의 패턴을 동시에 찾아야 할 때**는 이야기가 달라집니다. 패턴마다 개별적으로 알고리즘을 돌리면 전체 복잡도는 O((N + M) × P)로 늘어납니다. (P는 패턴 수)

**Aho-Corasick 알고리즘**은 1975년 Alfred V. Aho와 Margaret J. Corasick이 발표한 논문에서 소개된 멀티패턴 문자열 검색 알고리즘입니다. 핵심 아이디어는 모든 패턴을 트라이(Trie)로 구성한 뒤, 실패 링크(Failure Link)와 출력 링크(Output Link)를 추가하여 **유한 상태 오토마톤(Finite State Automaton, FSA)**을 만드는 것입니다. 이 오토마톤 위에서 텍스트를 단 한 번 순회하면 **모든 패턴의 모든 출현 위치**를 O(N + M + Z) 시간에 찾을 수 있습니다. (N은 텍스트 길이, M은 전체 패턴 길이 합, Z는 매칭 횟수)

### 핵심 구성 요소

**1. 트라이(Goto 함수)**

모든 패턴을 트라이에 삽입합니다. 각 패턴의 끝 노드에는 해당 패턴 식별자를 기록합니다. 트라이는 FSA의 기반 구조입니다.

**2. 실패 링크(Failure Link)**

현재 상태에서 다음 문자로 전이가 불가능할 때 이동하는 링크입니다. KMP의 실패 함수(failure function)와 동일한 개념을 트라이 전체에 적용한 것으로, BFS 순서로 계산합니다. 실패 링크는 **현재 상태에 해당하는 문자열의 최장 진짜 접미사** 중 트라이 내에 존재하는 것을 가리킵니다.

**3. 출력 링크(Output Link)**

실패 링크를 따라가면서 만나는 패턴 종료 노드들을 한 번에 접근할 수 있도록 연결하는 링크입니다. 이를 통해 겹치는 패턴(예: "he"와 "she"를 동시에 매칭)을 누락 없이 보고합니다.

---

## 왜 필요한가

Aho-Corasick 알고리즘이 필수적으로 쓰이는 영역을 살펴보면 그 중요성을 실감할 수 있습니다.

- **네트워크 침입 탐지 시스템(IDS)**: Snort, Suricata 같은 도구는 수천 개의 악성 패턴 시그니처를 패킷 페이로드에서 동시에 검색합니다. 패킷당 수천 번의 개별 검색을 하면 실시간 처리가 불가능합니다.
- **안티바이러스 엔진**: 수백만 개의 바이러스 시그니처 데이터베이스를 파일 내용에서 한 번에 스캔합니다.
- **검색 엔진 필터링**: 스팸, 혐오 표현, 금지어 등 다수의 키워드를 텍스트에서 동시에 감지합니다.
- **생물정보학**: DNA/RNA 서열에서 다수의 유전자 모티프를 동시에 탐색합니다.
- **grep -F (fixed string)**: GNU grep은 `-F` 옵션(고정 문자열 다중 검색)에 Aho-Corasick 기반 알고리즘을 활용합니다.

단일 패턴 알고리즘으로는 P개의 패턴을 찾기 위해 텍스트를 P번 순회해야 하지만, Aho-Corasick은 텍스트를 **단 한 번** 스캔합니다. 패턴 수가 늘어날수록 이점이 기하급수적으로 커집니다.

---

## 실제 구현 예제

### 예제 1: Aho-Corasick 오토마톤 Python 구현

```python
from collections import deque, defaultdict

class AhoCorasick:
    def __init__(self):
        # goto[state][char] = next_state
        self.goto = [{}]
        # fail[state] = failure_link_state
        self.fail = [-1]
        # output[state] = list of pattern indices
        self.output = [[]]
        self.patterns = []

    def add_pattern(self, pattern: str) -> int:
        pid = len(self.patterns)
        self.patterns.append(pattern)
        state = 0
        for ch in pattern:
            if ch not in self.goto[state]:
                self.goto[state][ch] = len(self.goto)
                self.goto.append({})
                self.fail.append(-1)
                self.output.append([])
            state = self.goto[state][ch]
        self.output[state].append(pid)
        return pid

    def build(self):
        q = deque()
        # 루트(0)의 자식들: 실패 링크 = 0
        for ch, s in self.goto[0].items():
            self.fail[s] = 0
            q.append(s)

        while q:
            r = q.popleft()
            for ch, s in self.goto[r].items():
                q.append(s)
                state = self.fail[r]
                # 실패 링크를 따라가며 ch 전이 가능한 상태 탐색
                while state != -1 and ch not in self.goto[state]:
                    state = self.fail[state]
                self.fail[s] = self.goto[state][ch] if state != -1 else 0
                if self.fail[s] == s:
                    self.fail[s] = 0
                # 출력 링크: 실패 링크 노드의 출력을 병합
                self.output[s] = self.output[s] + self.output[self.fail[s]]

    def search(self, text: str) -> list[tuple[int, int, str]]:
        """반환: (패턴인덱스, 종료위치, 패턴문자열) 리스트"""
        results = []
        state = 0
        for i, ch in enumerate(text):
            while state != -1 and ch not in self.goto[state]:
                state = self.fail[state]
            state = self.goto[state][ch] if state != -1 else 0
            for pid in self.output[state]:
                p = self.patterns[pid]
                start = i - len(p) + 1
                results.append((pid, start, p))
        return results


# 사용 예시
ac = AhoCorasick()
ac.add_pattern("he")
ac.add_pattern("she")
ac.add_pattern("his")
ac.add_pattern("hers")
ac.build()

text = "ushers"
matches = ac.search(text)
for pid, start, pattern in matches:
    print(f"패턴 '{pattern}' 발견: 위치 {start}~{start + len(pattern) - 1}")

# 출력:
# 패턴 'she' 발견: 위치 1~3
# 패턴 'he' 발견: 위치 2~3
# 패턴 'hers' 발견: 위치 2~5
```

이 구현에서 `build()`는 O(M × Σ) (Σ는 알파벳 크기), `search()`는 O(N + Z) 복잡도를 가집니다.

---

### 예제 2: 실패 링크 계산 과정 시각화 + 고성능 배열 기반 구현

```python
class AhoCorasickFast:
    """배열 기반의 고성능 구현 (딕셔너리 대신 배열 사용)"""
    
    ALPHA = 26  # 소문자 알파벳만 사용
    
    def __init__(self, max_nodes: int = 100_000):
        self.goto = [[-1] * self.ALPHA for _ in range(max_nodes)]
        self.fail = [-1] * max_nodes
        self.output = [[] for _ in range(max_nodes)]
        self.size = 1  # 루트 = 0
        self.patterns = []
    
    def _ord(self, ch: str) -> int:
        return ord(ch) - ord('a')
    
    def add_pattern(self, pattern: str):
        pid = len(self.patterns)
        self.patterns.append(pattern)
        cur = 0
        for ch in pattern:
            c = self._ord(ch)
            if self.goto[cur][c] == -1:
                self.goto[cur][c] = self.size
                self.size += 1
            cur = self.goto[cur][c]
        self.output[cur].append(pid)
    
    def build(self):
        from collections import deque
        q = deque()
        
        # 루트 자식 초기화
        for c in range(self.ALPHA):
            s = self.goto[0][c]
            if s == -1:
                self.goto[0][c] = 0  # 루트로 self-loop
            else:
                self.fail[s] = 0
                q.append(s)
        
        while q:
            r = q.popleft()
            # 출력 링크 병합
            self.output[r] = self.output[r] + self.output[self.fail[r]]
            for c in range(self.ALPHA):
                s = self.goto[r][c]
                if s == -1:
                    # goto 보완: 실패 링크 기반
                    self.goto[r][c] = self.goto[self.fail[r]][c]
                else:
                    self.fail[s] = self.goto[self.fail[r]][c]
                    q.append(s)
    
    def search(self, text: str) -> list:
        results = []
        cur = 0
        for i, ch in enumerate(text):
            c = self._ord(ch)
            cur = self.goto[cur][c]
            for pid in self.output[cur]:
                p = self.patterns[pid]
                results.append((i - len(p) + 1, i, p))
        return results


# 다중 패턴 검색 벤치마크
import time

patterns = [
    "attack", "malware", "exploit", "shellcode",
    "injection", "overflow", "privilege", "escalation"
]

ac_fast = AhoCorasickFast()
for p in patterns:
    ac_fast.add_pattern(p)
ac_fast.build()

# 네트워크 페이로드 시뮬레이션
payload = "this is a buffer overflow attempt with shellcode injection targeting privilege escalation"
start = time.perf_counter()
for _ in range(100_000):
    ac_fast.search(payload)
elapsed = time.perf_counter() - start

matches = ac_fast.search(payload)
print(f"발견된 패턴: {[(m[2], m[0]) for m in matches]}")
print(f"10만 회 검색 시간: {elapsed:.3f}초")
```

배열 기반 구현은 딕셔너리 기반 대비 캐시 친화성이 높아 실전 환경에서 3~5배 빠른 성능을 보입니다.

---

## 주의사항 및 팁

**1. 알파벳 크기(Σ)와 메모리 트레이드오프**

알파벳이 크면(예: Unicode) 배열 기반 노드가 메모리를 과다하게 소비합니다. 이 경우 딕셔너리 기반 goto나 압축된 노드 표현을 사용해야 합니다. 실용적으로는 ASCII 범위(256)로 제한하거나, 해시맵 기반 전이를 사용합니다.

**2. 겹치는 패턴 처리**

"he"와 "she"처럼 한 패턴이 다른 패턴의 접미사인 경우, 출력 링크(Output Link)를 통해 둘 다 보고됩니다. 이 동작이 의도한 것인지 확인해야 합니다. 중복 보고가 불필요하면 출력 링크 병합을 생략합니다.

**3. 스트리밍 처리**

텍스트를 스트리밍으로 처리할 때는 현재 상태(state)를 유지하면서 청크(chunk)를 순차적으로 처리할 수 있습니다. Aho-Corasick 오토마톤은 **상태 기계**이므로 이전 상태를 보존하면 자연스럽게 스트리밍을 지원합니다.

**4. 사전 컴파일된 오토마톤 직렬화**

안티바이러스처럼 패턴이 거의 변하지 않는 환경에서는 `build()` 결과(goto 테이블, fail 배열)를 직렬화하여 저장해 두고, 런타임에 재로드해 사용합니다. 패턴 수가 수백만 개라면 빌드 시간 자체가 수 초에 달할 수 있기 때문입니다.

**5. 멀티스레드 안전성**

`build()` 이후의 오토마톤은 읽기 전용이므로 여러 스레드가 `search()`를 동시에 호출해도 안전합니다. 단, 각 스레드는 독립적인 현재 상태(state 변수)를 유지해야 합니다.

**6. 실용 라이브러리**

- Python: `pyahocorasick` (C 확장, 매우 빠름)
- C++: `ahocorasick` (Boost 미포함, 헤더 온리)
- Java: `org.ahocorasick:ahocorasick`
- Rust: `aho-corasick` crate (regex 크레이트 내부에서도 사용)

---

## 참고 자료

- [Aho-Corasick algorithm - Algorithms for Competitive Programming](https://cp-algorithms.com/string/aho_corasick.html)
- [Aho–Corasick algorithm - Wikipedia](https://en.wikipedia.org/wiki/Aho%E2%80%93Corasick_algorithm)
- [Original Paper: Efficient String Matching - Alfred V. Aho & Margaret J. Corasick (1975)](https://dl.acm.org/doi/10.1145/360825.360855)
- [pyahocorasick Python Library](https://github.com/WojciechMula/pyahocorasick)
