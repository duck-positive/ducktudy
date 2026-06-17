---
layout: post
title: "B-트리와 B+트리: 데이터베이스 인덱싱의 핵심 자료구조"
date: 2026-06-17
categories: [cs, computer-science]
tags: [b-tree, b+tree, 자료구조, 데이터베이스, 인덱스, 알고리즘]
---

## 개요

데이터베이스에서 수백만, 수억 건의 데이터를 빠르게 검색하려면 어떻게 해야 할까요? 테이블 전체를 순차적으로 탐색하는 Full Table Scan은 데이터가 늘어날수록 선형적으로 느려집니다. 이 문제를 해결하기 위해 등장한 것이 **인덱스(Index)**이며, 현대 관계형 데이터베이스(MySQL, PostgreSQL, Oracle 등)의 인덱스는 대부분 **B+트리(B+ Tree)** 구조로 구현되어 있습니다.

이 글에서는 B-트리의 기본 개념부터 B+트리의 개선 사항, 실제 구현 예제, 그리고 실무에서 인덱스를 어떻게 활용해야 하는지까지 심도 있게 다룹니다.

---

## 1. B-트리란 무엇인가?

**B-트리(B-Tree, Balanced Tree)**는 1970년 Rudolf Bayer와 Ed McCreight가 고안한 자가 균형(Self-Balancing) 트리 자료구조입니다. 이진 탐색 트리(BST)를 일반화한 구조로, 하나의 노드에 여러 개의 키(Key)와 자식 포인터를 저장할 수 있습니다.

### B-트리의 특성 (차수 m인 경우)

1. 루트 노드는 최소 2개의 자식을 가집니다.
2. 루트와 리프를 제외한 모든 내부 노드는 최소 ⌈m/2⌉개, 최대 m개의 자식을 가집니다.
3. 모든 리프 노드는 동일한 깊이(depth)에 위치합니다.
4. n개의 자식을 가진 노드는 n-1개의 키를 가집니다.
5. 각 노드의 키는 오름차순으로 정렬되어 있습니다.

### 왜 B-트리인가? — 디스크 I/O 최적화

B-트리가 데이터베이스 인덱스로 선택된 가장 큰 이유는 **디스크 I/O 횟수 최소화**입니다.

하드 디스크와 SSD는 메모리보다 수십~수백 배 느립니다. 데이터를 읽을 때 한 번의 I/O 연산은 보통 페이지(Page) 단위로 이루어집니다(일반적으로 4KB~16KB). B-트리는 하나의 노드를 정확히 한 페이지에 맞게 설계하여, 트리의 높이(Depth)를 최소화하고 I/O 횟수를 줄입니다.

예를 들어, 차수가 1,000인 B-트리는 10억 개의 키를 단 3~4 레벨 안에 저장할 수 있습니다. 이는 이진 트리(Binary Tree)의 약 30레벨에 비해 압도적으로 효율적입니다.

---

## 2. B+트리란 무엇인가?

**B+트리(B+ Tree)**는 B-트리의 변형으로, 현대 RDBMS(관계형 데이터베이스 관리 시스템)에서 가장 널리 쓰이는 인덱스 구조입니다.

### B-트리와 B+트리의 핵심 차이

| 특성 | B-트리 | B+트리 |
|------|--------|--------|
| 데이터 저장 위치 | 내부 노드 + 리프 노드 | 리프 노드에만 저장 |
| 리프 노드 연결 | 없음 | 이중 연결 리스트로 연결 |
| 범위 검색(Range Query) | 비효율적 | 매우 효율적 |
| 내부 노드 크기 | 키 + 데이터 | 키만 저장 (더 많은 키 수용) |
| 중복 키 | 없음 | 내부 노드에 키가 중복될 수 있음 |

B+트리에서 내부(Internal) 노드는 오직 **라우팅 키(Routing Key)**만 저장하고, 실제 레코드는 모두 리프(Leaf) 노드에 저장됩니다. 리프 노드들은 이중 연결 리스트(Doubly Linked List)로 연결되어 있어, 범위 쿼리(`BETWEEN`, `>`, `<`) 시 순차 탐색이 가능합니다.

---

## 3. 코드 예제 1 — B+트리 노드 구조와 탐색

아래는 Python으로 구현한 B+트리의 기본 구조와 키 탐색(Search), 범위 검색(Range Query) 기능입니다.

```python
class BPlusTreeNode:
    def __init__(self, is_leaf=False):
        self.keys = []          # 정렬된 키 목록
        self.children = []      # 자식 노드 포인터 (내부 노드)
        self.values = []        # 실제 데이터 (리프 노드)
        self.next = None        # 다음 리프 노드 포인터
        self.is_leaf = is_leaf

class BPlusTree:
    def __init__(self, order=4):
        self.root = BPlusTreeNode(is_leaf=True)
        self.order = order

    def search(self, key):
        """키에 해당하는 값을 O(log n)에 탐색"""
        node = self._find_leaf(key)
        for i, k in enumerate(node.keys):
            if k == key:
                return node.values[i]
        return None

    def _find_leaf(self, key):
        """루트에서 시작하여 키가 속한 리프 노드 탐색"""
        node = self.root
        while not node.is_leaf:
            i = 0
            while i < len(node.keys) and key >= node.keys[i]:
                i += 1
            node = node.children[i]
        return node

    def range_search(self, start_key, end_key):
        """범위 검색: B+트리의 핵심 강점"""
        result = []
        node = self._find_leaf(start_key)
        while node is not None:
            for i, key in enumerate(node.keys):
                if start_key <= key <= end_key:
                    result.append((key, node.values[i]))
                elif key > end_key:
                    return result
            node = node.next  # 다음 리프 노드로 이동 (연결 리스트 활용)
        return result
```

`range_search`는 시작 키가 속한 리프 노드를 먼저 찾은 뒤, 리프 노드 연결 리스트를 따라 순차 탐색합니다. 이것이 B+트리가 `BETWEEN`, `ORDER BY` 쿼리에서 압도적으로 빠른 이유입니다.

---

## 4. 코드 예제 2 — B+트리 삽입과 노드 분할

B+트리의 핵심 연산인 삽입(Insert)과 노드 분할(Node Split)입니다. 삽입 후 노드가 가득 차면 분할이 발생하며, 분할 키가 부모 노드로 올라갑니다.

```python
    def insert(self, key, value):
        leaf = self._find_leaf(key)
        self._insert_in_leaf(leaf, key, value)
        if len(leaf.keys) >= self.order:
            new_leaf, push_up_key = self._split_leaf(leaf)
            self._insert_in_parent(leaf, push_up_key, new_leaf)

    def _insert_in_leaf(self, node, key, value):
        i = 0
        while i < len(node.keys) and node.keys[i] < key:
            i += 1
        node.keys.insert(i, key)
        node.values.insert(i, value)

    def _split_leaf(self, leaf):
        mid = len(leaf.keys) // 2
        new_leaf = BPlusTreeNode(is_leaf=True)
        new_leaf.keys = leaf.keys[mid:]
        new_leaf.values = leaf.values[mid:]
        leaf.keys = leaf.keys[:mid]
        leaf.values = leaf.values[:mid]
        # 이중 연결 리스트 갱신
        new_leaf.next = leaf.next
        leaf.next = new_leaf
        return new_leaf, new_leaf.keys[0]

    def _insert_in_parent(self, left, key, right):
        if left == self.root:
            # 루트 분할 → 트리 높이 1 증가
            new_root = BPlusTreeNode(is_leaf=False)
            new_root.keys = [key]
            new_root.children = [left, right]
            self.root = new_root
            return
        parent = self._find_parent(self.root, left)
        i = parent.children.index(left)
        parent.keys.insert(i, key)
        parent.children.insert(i + 1, right)
        if len(parent.keys) >= self.order:
            self._split_internal(parent)
```

분할(Split)은 재귀적으로 부모 노드까지 전파될 수 있으며, 루트가 분할되면 트리의 높이가 1 증가합니다. 이처럼 B+트리는 항상 모든 리프 노드가 동일한 깊이를 유지하는 **균형 트리(Balanced Tree)**입니다.

---

## 5. MySQL InnoDB에서의 실제 활용

### 클러스터드 인덱스와 세컨더리 인덱스

MySQL InnoDB는 기본 키(Primary Key)를 기준으로 **클러스터드 인덱스(Clustered Index)**를 구성합니다. 테이블의 실제 데이터 행(Row)이 B+트리의 리프 노드에 기본 키 순서대로 물리적으로 저장됩니다.

```sql
-- 클러스터드 인덱스 (기본 키)
CREATE TABLE users (
    id     INT PRIMARY KEY,
    name   VARCHAR(50),
    age    INT,
    email  VARCHAR(100)
);

-- 세컨더리 인덱스: 리프에 기본 키 값만 저장
CREATE INDEX idx_age ON users(age);

-- 실행 계획 확인: type=range, key=idx_age
EXPLAIN SELECT * FROM users WHERE age BETWEEN 20 AND 30;

-- 커버링 인덱스: 이중 탐색(Double Lookup) 완전 회피
CREATE INDEX idx_age_name ON users(age, name);
EXPLAIN SELECT name FROM users WHERE age BETWEEN 20 AND 30;
-- Extra: Using index (인덱스만으로 결과 반환)
```

세컨더리 인덱스 조회 시 리프 노드의 기본 키로 클러스터드 인덱스를 다시 탐색하는 **이중 탐색(Double Lookup)**이 발생합니다. 커버링 인덱스를 활용하면 이를 완전히 회피할 수 있습니다.

---

## 6. 주의사항과 실무 팁

1. **선택성(Selectivity)이 높은 컬럼에 인덱스를 생성하세요.** 성별(M/F)처럼 카디널리티가 낮은 컬럼은 인덱스 효과가 미미합니다. 컬럼 내 고유 값의 비율이 높을수록 인덱스가 효과적입니다.

2. **복합 인덱스의 컬럼 순서가 결정적입니다.** `(a, b, c)` 인덱스는 `WHERE a = ?`, `WHERE a = ? AND b = ?`에는 효과적이지만, `WHERE b = ?` 단독으로는 사용되지 않습니다(좌측 접두사 규칙, Leftmost Prefix Rule).

3. **인덱스는 쓰기 비용입니다.** INSERT/UPDATE/DELETE 시 B+트리를 항상 갱신해야 합니다. 실제로 조회에 사용되지 않는 인덱스는 정기적으로 제거하세요.

4. **EXPLAIN으로 반드시 실행 계획을 확인하세요.** 옵티마이저는 데이터 분포에 따라 인덱스 대신 Full Table Scan을 선택하기도 합니다.

5. **컬럼에 함수를 적용하면 인덱스를 사용할 수 없습니다.** `WHERE YEAR(created_at) = 2024`는 인덱스를 활용하지 못하므로 `WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'`로 작성하세요.

---

## 참고 자료
- [Wikipedia: B+ tree](https://en.wikipedia.org/wiki/B%2B_tree)
- [Wikipedia: B-tree](https://en.wikipedia.org/wiki/B-tree)
- [Use The Index, Luke — SQL Indexing and Tuning](https://use-the-index-luke.com/)
- [MySQL 공식 문서: How MySQL Uses Indexes](https://dev.mysql.com/doc/refman/8.0/en/mysql-indexes.html)
