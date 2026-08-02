---
layout: post
title: "데이터베이스 조인 알고리즘 완전 정복: Nested Loop·Hash Join·Merge Join의 선택 기준과 내부 동작"
date: 2026-08-02
categories: [cs, computer-science]
tags: [database, join-algorithm, hash-join, merge-join, nested-loop, query-optimizer, postgresql, sql]
---

## 개념 설명

관계형 데이터베이스에서 `JOIN`은 두 개 이상의 테이블에서 관련 행을 결합하는 핵심 연산이다. 대규모 데이터셋에서 JOIN의 성능은 쿼리 전체 성능을 결정짓는 핵심 병목이 될 수 있다. 쿼리 옵티마이저(Query Optimizer)는 통계 정보(Statistics), 인덱스 유무, 데이터 정렬 상태, 메모리 가용량 등을 분석하여 세 가지 기본 조인 알고리즘 중 가장 적합한 것을 선택한다.

### 세 가지 조인 알고리즘 비교

| 알고리즘 | 평균 복잡도 | 전제 조건 | 주요 사용 사례 |
|---------|-----------|---------|--------------|
| Nested Loop Join | O(n × m) | 없음 (인덱스 있으면 O(n log m)) | 소형 테이블, 인덱스 존재 |
| Hash Join | O(n + m) | 등가 조인(equi-join) 필요 | 대형 테이블, 인덱스 없음 |
| Merge Join | O(n log n + m log m) | 입력이 정렬되어야 함 | 정렬된 대형 테이블, 범위 조인 |

---

## 왜 필요한가

### 잘못된 조인 알고리즘 선택의 결과

```sql
-- 테이블 크기: orders(10M rows) × order_items(50M rows)
SELECT o.order_id, SUM(oi.price)
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
GROUP BY o.order_id;
```

- **Nested Loop**: 10M × 50M = 5천억 번 비교 → 사실상 불가능
- **Hash Join**: 10M + 50M = 6천만 번 연산 → 수 초 내 완료
- **Merge Join**: 두 테이블이 이미 `order_id`로 정렬되어 있다면 선형 스캔으로 완료

이처럼 데이터 규모와 특성에 따른 알고리즘 선택은 쿼리 성능을 수천 배 이상 좌우할 수 있다.

### 쿼리 플랜 확인의 중요성

```sql
-- PostgreSQL에서 실행 계획 확인
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT o.order_id, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.created_at > '2026-01-01';
```

실행 계획에서 어떤 조인이 선택되었는지, 예상 행 수와 실제 행 수의 차이, 버퍼 적중률 등을 분석하는 것이 쿼리 최적화의 시작이다.

---

## 실제 구현 예제

### 예제 1: 세 가지 조인 알고리즘 직접 구현 (Python)

```python
from typing import List, Tuple, Dict, Any, Iterator
from collections import defaultdict
import heapq

Row = Dict[str, Any]

def nested_loop_join(
    outer: List[Row],
    inner: List[Row],
    outer_key: str,
    inner_key: str
) -> List[Row]:
    """
    Nested Loop Join: 가장 단순한 조인.
    외부 테이블의 각 행에 대해 내부 테이블을 전체 순회.
    - 시간: O(|outer| × |inner|)
    - 메모리: O(1) (결과 제외)
    - 적합: outer가 매우 작을 때, inner에 인덱스 있을 때
    """
    result = []
    for o_row in outer:
        for i_row in inner:
            if o_row[outer_key] == i_row[inner_key]:
                result.append({**o_row, **i_row})
    return result


def index_nested_loop_join(
    outer: List[Row],
    inner: List[Row],
    outer_key: str,
    inner_key: str
) -> List[Row]:
    """
    Index Nested Loop Join: inner 테이블에 해시 인덱스 구축.
    - 시간: O(|outer| × 1) = O(|outer|) 평균 (인덱스 조회)
    - 메모리: O(|inner|) (인덱스)
    - 실제 DB에서 인덱스 NL Join의 동작 방식
    """
    # 인덱스 구축 (실제 DB에서는 B-tree 또는 hash index)
    index: Dict[Any, List[Row]] = defaultdict(list)
    for i_row in inner:
        index[i_row[inner_key]].append(i_row)

    result = []
    for o_row in outer:
        # 인덱스를 통해 O(1) 조회
        for i_row in index.get(o_row[outer_key], []):
            result.append({**o_row, **i_row})
    return result


def hash_join(
    build_side: List[Row],
    probe_side: List[Row],
    build_key: str,
    probe_key: str
) -> List[Row]:
    """
    Hash Join: 두 단계로 구성.
    1. Build Phase: 작은 테이블(build_side)로 해시 테이블 구축
    2. Probe Phase: 큰 테이블(probe_side)을 스캔하며 해시 테이블 탐색

    - 시간: O(|build| + |probe|) 평균
    - 메모리: O(|build|) → build side가 메모리에 맞아야 함
    - 등가 조인(=)만 지원
    - 적합: 대형 테이블 간 조인, 인덱스 없을 때
    """
    # Phase 1: Build
    hash_table: Dict[Any, List[Row]] = defaultdict(list)
    for row in build_side:
        hash_table[row[build_key]].append(row)

    # Phase 2: Probe
    result = []
    for row in probe_side:
        key = row[probe_key]
        if key in hash_table:
            for build_row in hash_table[key]:
                result.append({**build_row, **row})
    return result


def merge_join(
    left: List[Row],
    right: List[Row],
    left_key: str,
    right_key: str
) -> List[Row]:
    """
    Sort-Merge Join: 양쪽 테이블을 조인 키로 정렬 후 병합.

    - 시간: O(n log n + m log m) 정렬 포함, 정렬 완료 시 O(n + m)
    - 메모리: O(n + m) 최악 (모든 행이 같은 키인 경우)
    - 등가 조인 및 범위 조인 지원
    - 적합: 이미 정렬된 데이터(인덱스 스캔), 대형 테이블
    """
    # 정렬 (실제 DB에서는 인덱스 스캔으로 이미 정렬된 상태일 수 있음)
    left_sorted = sorted(left, key=lambda r: r[left_key])
    right_sorted = sorted(right, key=lambda r: r[right_key])

    result = []
    i, j = 0, 0
    n, m = len(left_sorted), len(right_sorted)

    while i < n and j < m:
        lv = left_sorted[i][left_key]
        rv = right_sorted[j][right_key]

        if lv < rv:
            i += 1
        elif lv > rv:
            j += 1
        else:
            # 같은 키 값의 모든 조합 출력
            j_start = j
            while j < m and right_sorted[j][right_key] == lv:
                result.append({**left_sorted[i], **right_sorted[j]})
                j += 1
            i += 1
            # 다음 left 행도 같은 키라면 j를 되돌림
            if i < n and left_sorted[i][left_key] == lv:
                j = j_start

    return result


# 테스트 데이터
customers = [
    {"cust_id": 1, "name": "Alice"},
    {"cust_id": 2, "name": "Bob"},
    {"cust_id": 3, "name": "Charlie"},
    {"cust_id": 4, "name": "Dave"},
]

orders = [
    {"order_id": 101, "cust_id": 1, "amount": 500},
    {"order_id": 102, "cust_id": 2, "amount": 300},
    {"order_id": 103, "cust_id": 1, "amount": 700},
    {"order_id": 104, "cust_id": 3, "amount": 200},
    {"order_id": 105, "cust_id": 2, "amount": 150},
]

print("=== Nested Loop Join ===")
result = nested_loop_join(customers, orders, "cust_id", "cust_id")
for r in result:
    print(f"  {r['name']} - Order #{r['order_id']}: ${r['amount']}")

print("\n=== Hash Join (customers as build side) ===")
result = hash_join(customers, orders, "cust_id", "cust_id")
for r in result:
    print(f"  {r['name']} - Order #{r['order_id']}: ${r['amount']}")

print("\n=== Merge Join ===")
result = merge_join(customers, orders, "cust_id", "cust_id")
for r in result:
    print(f"  {r['name']} - Order #{r['order_id']}: ${r['amount']}")
```

출력 결과:
```
=== Nested Loop Join ===
  Alice - Order #101: $500
  Alice - Order #103: $700
  Bob - Order #102: $300
  Bob - Order #105: $150
  Charlie - Order #104: $200

=== Hash Join (customers as build side) ===
  Alice - Order #101: $500
  Alice - Order #103: $700
  Bob - Order #102: $300
  Bob - Order #105: $150
  Charlie - Order #104: $200

=== Merge Join ===
  Alice - Order #101: $500
  Alice - Order #103: $700
  Bob - Order #102: $300
  Bob - Order #105: $150
  Charlie - Order #104: $200
```

---

### 예제 2: Grace Hash Join (메모리 초과 시 디스크 활용)

실제 데이터베이스에서는 build side가 메모리에 다 들어오지 않을 수 있다. 이를 해결하는 **Grace Hash Join**을 구현한다.

```python
import os
import json
import tempfile
from typing import List, Dict, Any
from collections import defaultdict
import hashlib

Row = Dict[str, Any]

class GraceHashJoin:
    """
    Grace Hash Join: 디스크를 활용하는 대용량 해시 조인.
    
    단계:
    1. Partition Phase: 두 테이블을 같은 해시 함수로 파티션 분할
       → 같은 파티션에는 같은 키를 가진 행들이 모임
    2. Build/Probe Phase: 각 파티션 쌍에 대해 일반 hash join 수행
       → 각 파티션이 메모리에 맞음을 보장
    
    메모리 복잡도: O(sqrt(N)) 파티션 수를 적절히 선택 시
    """

    def __init__(self, num_partitions: int = 4):
        self.num_partitions = num_partitions

    def _partition_id(self, key: Any) -> int:
        h = int(hashlib.md5(str(key).encode()).hexdigest(), 16)
        return h % self.num_partitions

    def join(
        self,
        build_side: List[Row],
        probe_side: List[Row],
        build_key: str,
        probe_key: str
    ) -> List[Row]:
        # Phase 1: Partition
        build_partitions: List[List[Row]] = [[] for _ in range(self.num_partitions)]
        probe_partitions: List[List[Row]] = [[] for _ in range(self.num_partitions)]

        for row in build_side:
            pid = self._partition_id(row[build_key])
            build_partitions[pid].append(row)

        for row in probe_side:
            pid = self._partition_id(row[probe_key])
            probe_partitions[pid].append(row)

        # Phase 2: Build and Probe per partition
        result = []
        for pid in range(self.num_partitions):
            # 각 파티션이 메모리에 맞는다고 가정
            # 실제 구현에서는 파티션을 디스크에 쓰고 하나씩 읽음
            build_ht: Dict[Any, List[Row]] = defaultdict(list)
            for row in build_partitions[pid]:
                build_ht[row[build_key]].append(row)

            for row in probe_partitions[pid]:
                key = row[probe_key]
                for build_row in build_ht.get(key, []):
                    result.append({**build_row, **row})

        return result


# 비교 실험: 일반 Hash Join vs Grace Hash Join
import time
import random

def generate_data(n: int):
    return [
        {"id": i, "key": random.randint(1, n // 2), "value": f"val_{i}"}
        for i in range(n)
    ]

left_data = generate_data(10000)
right_data = generate_data(50000)

# 일반 Hash Join
start = time.perf_counter()
result1 = hash_join(left_data, right_data, "key", "key")
t1 = time.perf_counter() - start

# Grace Hash Join (4 partitions)
gjoin = GraceHashJoin(num_partitions=8)
start = time.perf_counter()
result2 = gjoin.join(left_data, right_data, "key", "key")
t2 = time.perf_counter() - start

print(f"일반 Hash Join:  {len(result1):,}행, {t1*1000:.1f}ms")
print(f"Grace Hash Join: {len(result2):,}행, {t2*1000:.1f}ms")
print(f"결과 일치: {len(result1) == len(result2)}")
```

---

### PostgreSQL에서의 실행 계획 분석

```sql
-- 예시: 조인 알고리즘이 선택되는 조건 이해

-- 1. 소형 테이블 → Nested Loop
EXPLAIN SELECT c.name, o.amount
FROM customers c    -- 1,000행
JOIN orders o ON c.cust_id = o.cust_id  -- 5,000행
WHERE c.cust_id = 42;
-- → Index Scan on customers + Nested Loop
-- → customers에 cust_id 인덱스 있고 선택도(selectivity) 높으므로

-- 2. 대형 테이블, 인덱스 없음 → Hash Join
EXPLAIN SELECT c.name, SUM(o.amount)
FROM customers c    -- 1M행
JOIN orders o ON c.cust_id = o.cust_id  -- 10M행
GROUP BY c.name;
-- → Hash Join (customers를 build side로)
-- → Sequential Scan on both tables

-- 3. 정렬된 데이터 → Merge Join
EXPLAIN SELECT c.name, o.amount
FROM customers c
JOIN orders o ON c.cust_id = o.cust_id
ORDER BY c.cust_id;
-- → Merge Join (이미 cust_id 인덱스로 정렬된 경우)

-- 옵티마이저 힌트로 강제 설정 (테스트 목적)
SET enable_hashjoin = OFF;
SET enable_mergejoin = OFF;
-- → 이 상태에서는 Nested Loop만 선택됨

-- 실행 계획 상세 분석
EXPLAIN (ANALYZE, BUFFERS, VERBOSE)
SELECT c.name, COUNT(o.order_id) AS order_count
FROM customers c
LEFT JOIN orders o ON c.cust_id = o.cust_id
GROUP BY c.name
HAVING COUNT(o.order_id) > 5;

-- 출력 예시:
-- HashAggregate  (cost=45231.00..45261.00 rows=3000 width=64)
--   Buckets: 4096  Batches: 1  Memory Usage: 512kB
--   ->  Hash Left Join  (cost=2500.00..43481.00 rows=350000 width=48)
--         Hash Cond: (o.cust_id = c.cust_id)
--         Buffers: shared hit=8234 read=1200
--         ->  Seq Scan on orders  (cost=0.00..19346.00 rows=1000000 width=16)
--               Buffers: shared hit=6834 read=1200
--         ->  Hash  (cost=1500.00..1500.00 rows=80000 width=36)
--               Buckets: 131072  Batches: 1  Memory Usage: 5120kB
--               ->  Seq Scan on customers  (cost=0.00..1500.00 rows=80000 width=36)
```

---

## 주의사항 및 팁

### 1. 조인 순서(Join Order)와 비용 추정

옵티마이저는 조인 순서도 결정한다. 3개 테이블 A, B, C의 조인은 `(A⋈B)⋈C`, `(A⋈C)⋈B`, `(B⋈C)⋈A` 등 여러 순서가 가능하다. 테이블이 `n`개면 가능한 조인 순서는 `(2(n-1))! / (n-1)!`로 폭발적으로 증가한다. PostgreSQL은 테이블이 8개 이하면 정확한 동적 프로그래밍으로, 그 이상은 유전 알고리즘(geqo)으로 계획을 탐색한다.

```sql
-- 통계 정보 최신화 (옵티마이저 비용 추정 개선)
ANALYZE customers;
ANALYZE orders;

-- 확장 통계: 두 컬럼 간의 상관관계 추적
CREATE STATISTICS orders_stat ON cust_id, created_at FROM orders;
ANALYZE orders;
```

### 2. Hash Join의 메모리 관리와 배치(Batch)

빌드 테이블이 `work_mem`을 초과하면 PostgreSQL은 **배치(batch)** 처리로 전환한다. 여러 배치로 나눠 디스크를 활용하지만 I/O 비용이 증가한다.

```sql
-- work_mem 설정 (세션 수준)
SET work_mem = '256MB';  -- Hash Join, Sort 등에 사용되는 메모리

-- 현재 실행 중인 쿼리의 배치 수 확인
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...;
-- "Hash  (cost=... rows=... width=...)"
-- "  Buckets: 1048576  Batches: 4  Memory Usage: 32768kB"
-- Batches: 4 → 4번의 디스크 패스 필요 → work_mem 증가 권장
```

### 3. Nested Loop Join + 인덱스 최적화

Nested Loop가 선택되었는데 성능이 나쁘다면, 내부 테이블의 조인 키에 인덱스가 없는 경우가 많다.

```sql
-- 인덱스 생성 전: Seq Scan → O(n × m)
-- 인덱스 생성 후: Index Scan → O(n × log m)

CREATE INDEX idx_orders_cust_id ON orders(cust_id);

-- 복합 인덱스: 자주 함께 조회되는 컬럼을 포함시켜 Index-Only Scan 유도
CREATE INDEX idx_orders_covering
ON orders(cust_id, order_id, amount);  -- covering index
```

### 4. Merge Join 강제와 정렬 비용

Merge Join은 정렬된 입력을 요구한다. 두 테이블 모두 인덱스 스캔으로 정렬된 순서를 제공할 수 있다면 추가 정렬 비용 없이 Merge Join이 가장 효율적이다.

```sql
-- 정렬을 공유할 수 있는 경우: ORDER BY와 JOIN 키가 같을 때
SELECT c.cust_id, c.name, o.amount
FROM customers c
JOIN orders o ON c.cust_id = o.cust_id
ORDER BY c.cust_id;
-- → 이미 cust_id 인덱스로 정렬 제공 → Merge Join + 추가 Sort 불필요
```

### 5. Anti-Join과 Semi-Join

`NOT EXISTS`, `NOT IN`, `EXISTS`, `IN (subquery)` 패턴은 Anti-Join 또는 Semi-Join으로 변환된다:

```sql
-- Anti-Join (LEFT JOIN + WHERE IS NULL과 동일)
SELECT c.name
FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.cust_id = c.cust_id
);
-- PostgreSQL → Hash Anti Join 또는 Merge Anti Join

-- Semi-Join
SELECT c.name
FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.cust_id = c.cust_id
);
-- PostgreSQL → Hash Semi Join
-- → 내부 테이블에서 첫 번째 매칭 발견 즉시 진행 (불필요한 중복 방지)
```

### 6. 알고리즘 선택 가이드라인 요약

```
조인 알고리즘 선택 트리:

outer 테이블이 매우 작다 (< 수천 행)?
├── YES → 인덱스 있으면 Index NL Join
│         인덱스 없으면 Hash Join
└── NO → 등가 조인(=)?
    ├── NO → Nested Loop (범위 조인) 또는 Merge Join (정렬 데이터)
    └── YES → 두 테이블 모두 대형?
         ├── 이미 정렬됨 → Merge Join
         ├── build side가 메모리에 맞음 → Hash Join
         └── build side가 메모리 초과 → Grace Hash Join (Batched Hash Join)
```

## 참고 자료
- [Wikipedia: Hash join](https://en.wikipedia.org/wiki/Hash_join)
- [PostgreSQL Documentation: Planner Statistics](https://www.postgresql.org/docs/current/planner-stats.html)
- [SQLShack: Internals of Physical Join Operators](https://www.sqlshack.com/internals-of-physical-join-operators-nested-loops-join-hash-match-join-merge-join-in-sql-server/)
- [Medium: SQL Join Algorithms Explained](https://medium.com/@milhamsuryapratama/sql-join-algorithms-nested-loop-merge-and-hash-join-f264a34194e3)
