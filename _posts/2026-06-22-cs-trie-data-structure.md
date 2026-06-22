---
layout: post
title: "트라이(Trie) 완전 정복: 문자열 검색의 최강 자료구조"
date: 2026-06-22
categories: [cs, computer-science]
tags: [trie, data-structure, string, autocomplete, algorithm, prefix-tree]
---

트라이(Trie)는 문자열 집합을 다루는 트리 기반 자료구조로, 자동완성·맞춤법 검사·IP 라우팅 등 실생활 곳곳에 숨어 있다. 해시 테이블과 비교했을 때 접두사 검색에서 압도적으로 유리하며, 사전(Dictionary) 구현의 표준 선택지 중 하나다.

## 개념: 트라이란 무엇인가?

Trie라는 이름은 re**trie**val(검색)에서 유래했다. "트리" 또는 "트라이"라고 읽는다. 각 노드는 문자 하나를 나타내며, 루트에서 특정 노드까지의 경로가 하나의 문자열(접두사)을 의미한다.

```
삽입 단어: ["apple", "app", "ap", "banana"]

          (root)
         /       \
        a         b
        |         |
        p         a
       / \        |
      (*)  p      n
           |      |
           l      a
           |      |
           e      n
           |      |
          (*)     a
                  |
                 (*)
```

- `(*)` 표시된 노드: `isEnd = true` — 여기까지가 하나의 완성된 단어임을 의미
- 각 노드는 최대 알파벳 수(26개 또는 해시맵)의 자식 포인터를 가짐

### 핵심 속성

| 연산 | 시간복잡도 | 설명 |
|------|-----------|------|
| 삽입 | O(L) | L = 문자열 길이 |
| 검색 | O(L) | |
| 접두사 검색 | O(P) | P = 접두사 길이 |
| 삭제 | O(L) | |

해시맵 기반 검색이 O(1)이지만, "apple로 시작하는 단어 모두 찾기" 같은 접두사 쿼리는 O(N·L)이 된다. 트라이는 이를 O(P + K)로(K = 결과 수) 해결한다.

---

## 왜 트라이가 필요한가?

### 1. 자동완성 (Autocomplete)

검색창에 "py"를 입력하면 "python", "pytorch", "pypi"를 제안하는 기능. 트라이에서 "py" 노드를 찾은 뒤 그 서브트리의 모든 완성 단어를 DFS로 수집하면 된다.

### 2. 맞춤법 검사 (Spell Check)

단어 사전 전체를 트라이에 올려두면, 입력 단어의 각 문자를 따라 내려가며 O(L)만에 존재 여부를 확인한다. 유사 단어 제안은 허용 편집거리(Levenshtein Distance)만큼 분기를 탐색하면 된다.

### 3. IP 라우팅 (Longest Prefix Match)

IPv4 라우팅 테이블은 트라이(정확히는 Radix Tree/Patricia Tree)로 구현된다. 패킷의 목적지 IP를 비트 단위로 트라이에서 탐색해 가장 긴 매칭 접두사를 찾는 것이 핵심이다.

### 4. 단어 빈도 추적

각 노드에 카운터를 달면 단어의 빈도를 O(L)로 추적할 수 있다. 빅데이터 스트림에서 Top-K 빈출 단어를 찾는 데 활용된다.

---

## 실제 구현 예제

### 예제 1: Python으로 기본 트라이 구현

```python
class TrieNode:
    def __init__(self):
        self.children: dict[str, 'TrieNode'] = {}
        self.is_end: bool = False
        self.count: int = 0  # 이 단어의 삽입 횟수

class Trie:
    def __init__(self):
        self.root = TrieNode()

    def insert(self, word: str) -> None:
        node = self.root
        for ch in word:
            if ch not in node.children:
                node.children[ch] = TrieNode()
            node = node.children[ch]
        node.is_end = True
        node.count += 1

    def search(self, word: str) -> bool:
        """단어가 정확히 존재하는지 확인"""
        node = self._traverse(word)
        return node is not None and node.is_end

    def starts_with(self, prefix: str) -> bool:
        """해당 접두사로 시작하는 단어가 있는지 확인"""
        return self._traverse(prefix) is not None

    def autocomplete(self, prefix: str) -> list[str]:
        """접두사로 시작하는 모든 단어 반환"""
        node = self._traverse(prefix)
        if node is None:
            return []
        result = []
        self._dfs(node, list(prefix), result)
        return result

    def _traverse(self, s: str):
        node = self.root
        for ch in s:
            if ch not in node.children:
                return None
            node = node.children[ch]
        return node

    def _dfs(self, node: TrieNode, path: list[str], result: list[str]) -> None:
        if node.is_end:
            result.append(''.join(path))
        for ch, child in node.children.items():
            path.append(ch)
            self._dfs(child, path, result)
            path.pop()


# 사용 예시
trie = Trie()
words = ["python", "pytorch", "pypi", "java", "javascript", "golang"]
for w in words:
    trie.insert(w)

print(trie.search("python"))       # True
print(trie.search("pyth"))         # False
print(trie.starts_with("py"))      # True
print(trie.autocomplete("java"))   # ['java', 'javascript']
print(trie.autocomplete("go"))     # ['golang']
```

### 예제 2: 메모리 최적화 — 배열 기반 트라이 (Java)

해시맵 대신 고정 크기 배열을 쓰면 메모리 오버헤드를 줄이고 캐시 효율을 높일 수 있다. 알파벳 소문자만 처리하는 경우 크기 26 배열이면 충분하다.

```java
class Trie {
    private static final int ALPHA = 26;
    private int[][] children;  // children[node][c] = 자식 노드 번호
    private boolean[] isEnd;
    private int size;

    public Trie(int maxNodes) {
        children = new int[maxNodes][ALPHA];
        isEnd = new boolean[maxNodes];
        size = 1;  // 루트는 노드 0
        // -1: 자식 없음
        for (int[] row : children) java.util.Arrays.fill(row, -1);
    }

    public void insert(String word) {
        int node = 0;
        for (char ch : word.toCharArray()) {
            int c = ch - 'a';
            if (children[node][c] == -1) {
                children[node][c] = size++;
            }
            node = children[node][c];
        }
        isEnd[node] = true;
    }

    public boolean search(String word) {
        int node = traverse(word);
        return node != -1 && isEnd[node];
    }

    public boolean startsWith(String prefix) {
        return traverse(prefix) != -1;
    }

    private int traverse(String s) {
        int node = 0;
        for (char ch : s.toCharArray()) {
            int c = ch - 'a';
            if (children[node][c] == -1) return -1;
            node = children[node][c];
        }
        return node;
    }
}

// 사용 예시
public class Main {
    public static void main(String[] args) {
        Trie trie = new Trie(100_000);
        trie.insert("apple");
        trie.insert("app");
        trie.insert("application");

        System.out.println(trie.search("app"));         // true
        System.out.println(trie.search("ap"));          // false
        System.out.println(trie.startsWith("appl"));    // true
    }
}
```

**배열 vs 해시맵 비교:**

| | 배열 기반 | 해시맵 기반 |
|--|---------|----------|
| 접근 속도 | O(1), 캐시 친화적 | O(1) 평균, 해시 충돌 가능 |
| 메모리 | 고정 (빈 슬롯 낭비) | 동적 (실제 사용량에 비례) |
| 적합 케이스 | 알파벳 고정, 대용량 | 유니코드, 가변 문자셋 |

---

## 고급 변형: Compressed Trie (Radix Tree)

일반 트라이는 긴 단일 경로("golang"을 삽입할 때 g→o→l→a→n→g 6개 노드)가 메모리를 낭비한다. Radix Tree(Patricia Tree)는 단일 자식인 경로를 하나의 엣지로 압축한다.

```
일반 트라이:          Radix Tree:
g → o → l → a → n → g    "golang"
g → o                     "go" → "lang"
                                   ↓
                                  (end)
```

Go 언어의 `net/http` 라우터, Nginx 설정 파싱 등이 이 방식을 쓴다.

---

## 주의사항 및 팁

### 1. 메모리 사용량에 주의

단어 수가 N, 평균 길이가 L일 때, 최악의 경우(공유 접두사 없음) 노드 수는 O(N·L)이다. 100만 단어 × 평균 길이 10 = 1000만 노드. 노드당 26 포인터 × 8바이트 = 약 2GB. 대용량에서는 Radix Tree나 해시 기반 압축이 필수다.

### 2. 삭제 구현 시 주의

단순히 `is_end = False`만 하면 중간 노드들이 고아로 남는다. 재귀적으로 "자식이 없고 is_end가 false인 노드"를 역방향으로 제거해야 메모리가 해제된다.

```python
def delete(self, word: str) -> bool:
    def _delete(node, word, depth):
        if depth == len(word):
            if not node.is_end:
                return False
            node.is_end = False
            return len(node.children) == 0  # 자식 없으면 이 노드 삭제 가능
        ch = word[depth]
        if ch not in node.children:
            return False
        should_delete = _delete(node.children[ch], word, depth + 1)
        if should_delete:
            del node.children[ch]
            return not node.is_end and len(node.children) == 0
        return False
    return _delete(self.root, word, 0)
```

### 3. 유니코드 처리

한글, 중국어, 이모지가 포함된 텍스트라면 배열 기반 대신 해시맵 기반을 써야 한다. Python의 경우 `dict[str, TrieNode]`가 자연스럽게 멀티바이트 문자를 처리한다.

### 4. 실제로 쓰이는 라이브러리

- **Python**: `pygtrie`, `datrie` (DAWG 기반)
- **Java**: Apache Commons의 `PatriciaTrie`
- **Go**: `github.com/derekparker/trie`
- **Redis**: Sorted Set을 활용한 lexicographic range 쿼리로 트라이를 에뮬레이션 가능

---

## 마무리

트라이는 단순 검색을 넘어 "접두사가 같은 모든 항목을 효율적으로 처리"하는 상황에서 독보적인 성능을 낸다. 면접 단골 문제인 LeetCode 208 (Implement Trie), 212 (Word Search II)도 이 구조가 기반이다. Radix Tree로의 확장까지 이해하면 Nginx, Linux 커널 네트워크 스택의 구현 원리도 읽힌다.

## 참고 자료
- [Trie - Wikipedia](https://en.wikipedia.org/wiki/Trie)
- [Introduction to Trie - GeeksforGeeks](https://www.geeksforgeeks.org/dsa/introduction-to-trie-data-structure-and-algorithm-tutorials/)
- [Trie Data Structure - Toptal Engineering Blog](https://www.toptal.com/java/the-trie-a-neglected-data-structure)
- [208. Implement Trie - LeetCode](https://leetcode.com/problems/implement-trie-prefix-tree/)
