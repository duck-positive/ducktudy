---
layout: post
title: "데이터베이스 쿼리 옵티마이저와 실행 계획: SQL이 어떻게 빠른 코드가 되는가"
date: 2026-06-24
categories: [cs, computer-science]
tags: [database, query-optimizer, execution-plan, join-algorithms, statistics, postgresql, sql]
---

## 쿼리 옵티마이저란 무엇인가

SQL은 **선언적(Declarative)** 언어다. 우리는 "무엇을 원하는지"만 기술하고, "어떻게 가져올지"는 데이터베이스에게 맡긴다. 이 "어떻게"를 결정하는 핵심 컴포넌트가 바로 **쿼리 옵티마이저(Query Optimizer)** 다.

```sql
SELECT u.name, COUNT(o.id) AS order_count
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.country = 'KR'
GROUP BY u.name
ORDER BY order_count DESC;
```

이 쿼리를 처리하는 방법은 수십 가지다. `users`를 먼저 필터링할 수도 있고, `orders`를 먼저 스캔할 수도 있다. 조인 방식도 Nested Loop, Hash Join, Merge Join 중 하나를 선택해야 한다. 인덱스를 쓸지 순차 스캔을 할지도 결정해야 한다. 옵티마이저는 이 모든 대안 중 **예상 비용이 가장 낮은 실행 계획**을 찾아낸다.

## 옵티마이저의 동작 과정

### 1단계: 파싱(Parsing)과 의미 분석(Semantic Analysis)

SQL 텍스트를 **파스 트리(Parse Tree)** 로 변환하고, 테이블/컬럼 존재 여부, 타입 호환성, 권한 등을 검증한다.

### 2단계: 쿼리 재작성(Query Rewriting)

파스 트리를 **논리 쿼리 계획(Logical Query Plan)** 으로 변환한 후, 의미를 유지하면서 더 효율적인 형태로 변환(rewrite)한다.

대표적인 재작성 규칙:
- **조건 푸시다운(Predicate Pushdown)**: `WHERE` 조건을 가능한 한 아래쪽(테이블 스캔에 가깝게) 이동해 처리 행 수를 조기에 줄인다.
- **서브쿼리를 조인으로 변환**: 상관 서브쿼리를 조인으로 변환하면 한 번의 스캔으로 처리 가능해진다.
- **중복 조건 제거**: `a = 1 AND a = 1`을 `a = 1`로 단순화.

```sql
-- 재작성 전 (비효율적)
SELECT * FROM orders o
WHERE o.user_id IN (
  SELECT id FROM users WHERE country = 'KR'
);

-- 조건 푸시다운 + 서브쿼리 → 조인으로 재작성
SELECT o.* FROM orders o
JOIN users u ON o.user_id = u.id
WHERE u.country = 'KR';
```

### 3단계: 비용 기반 최적화(Cost-Based Optimization)

핵심 단계. 옵티마이저는 **카탈로그(Catalog)에 저장된 통계 정보**를 사용해 각 실행 계획의 비용을 추정한다.

**통계 정보의 구성요소:**
- `n_distinct`: 컬럼의 고유값 수 (카디널리티)
- 히스토그램: 값 분포 정보 (어떤 범위에 행이 몰려있는지)
- `correlation`: 컬럼 값과 물리적 저장 순서의 상관관계
- 행 수(row count), 페이지 수(page count)

```sql
-- PostgreSQL에서 통계 확인
SELECT attname, n_distinct, correlation
FROM pg_stats
WHERE tablename = 'users' AND attname = 'country';

-- 통계 업데이트
ANALYZE users;
```

비용은 주로 **디스크 I/O 횟수 + CPU 연산량**으로 모델링된다. PostgreSQL은 `seq_page_cost = 1.0`, `random_page_cost = 4.0`(기본값)처럼 상대적 비용 단위를 사용한다.

## 세 가지 조인 알고리즘

### Nested Loop Join

```
for each row r1 in outer_table:
    for each row r2 in inner_table:
        if r1.key == r2.key:
            emit(r1, r2)
```

- **복잡도**: O(N × M)
- **최적 조건**: outer 테이블이 작고, inner 테이블에 인덱스가 있을 때.
- **인덱스 Nested Loop**: inner 테이블의 조인 컬럼에 인덱스가 있으면 O(N × log M)으로 줄어든다.

### Hash Join

```
-- Build phase: 작은 테이블로 해시 테이블 구축
for each row r1 in build_table:
    hash_table[hash(r1.key)] = r1

-- Probe phase: 큰 테이블로 해시 테이블 탐색
for each row r2 in probe_table:
    if hash_table[hash(r2.key)] exists:
        emit(match, r2)
```

- **복잡도**: O(N + M) — 단, 해시 테이블이 메모리에 들어올 때
- **최적 조건**: 테이블이 크고, 동등 조인(`=`)이며, 인덱스가 없을 때.
- **Grace Hash Join**: 메모리에 안 들어오면 디스크로 분할(partition)한 후 합산.

### Merge Join (Sort-Merge Join)

```
sort(table1, key)
sort(table2, key)
i, j = 0, 0
while i < len(table1) and j < len(table2):
    if table1[i].key == table2[j].key:
        emit(table1[i], table2[j])
        j++
    elif table1[i].key < table2[j].key:
        i++
    else:
        j++
```

- **복잡도**: O(N log N + M log M) (정렬 포함) / O(N + M) (이미 정렬된 경우)
- **최적 조건**: 두 테이블이 이미 정렬되어 있거나(인덱스 스캔), 범위 조건 조인.

## 실행 계획 읽기 (EXPLAIN ANALYZE)

### 예제 1: PostgreSQL EXPLAIN ANALYZE

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT u.name, COUNT(o.id)
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.country = 'KR'
GROUP BY u.name;
```

```
HashAggregate  (cost=4521.00..4621.00 rows=10000) (actual time=45.2..48.3 rows=8921)
  Group Key: u.name
  Buffers: shared hit=320 read=1821
  ->  Hash Join  (cost=800.00..4021.00 rows=100000) (actual time=8.1..38.7 rows=87432)
        Hash Cond: (o.user_id = u.id)
        Buffers: shared hit=280 read=1621
        ->  Seq Scan on orders o  (cost=0.00..2200.00 rows=200000) (actual time=0.1..15.3 rows=200000)
              Buffers: shared read=1200
        ->  Hash  (cost=750.00..750.00 rows=4000) (actual time=7.8..7.8 rows=4150)
              Buckets: 4096  Batches: 1  Memory Usage: 312kB
              ->  Seq Scan on users u  (cost=0.00..750.00 rows=4000)  (actual time=0.1..5.2 rows=4150)
                    Filter: ((country)::text = 'KR')
                    Rows Removed by Filter: 45850
```

**읽는 법:**
- `cost=A..B`: A = 첫 행 반환까지 비용, B = 전체 비용 (예상)
- `actual time=X..Y`: 실제 측정된 밀리초
- `rows=N`: 예상 행 수 vs 실제 행 수 — **두 수치 차이가 크면 통계가 오래됐다는 신호**
- `Buffers: shared hit=X read=Y`: X = 캐시 히트, Y = 디스크 읽기

### 예제 2: 인덱스 미스로 인한 성능 저하 감지 및 수정 (Python + psycopg2)

```python
import psycopg2
import json

conn = psycopg2.connect("dbname=mydb user=postgres")
cur = conn.cursor()

# 실행 계획을 JSON으로 받아서 분석
cur.execute("""
    EXPLAIN (ANALYZE, FORMAT JSON)
    SELECT * FROM orders WHERE user_id = 12345 AND status = 'pending'
""")
plan = cur.fetchone()[0][0]

# 실행 계획에서 Seq Scan 탐지
def find_seq_scans(node, path=""):
    issues = []
    node_type = node.get("Node Type", "")
    if "Seq Scan" in node_type:
        rows = node.get("Actual Rows", 0)
        cost = node.get("Total Cost", 0)
        # 큰 테이블에서 Seq Scan은 경고
        if node.get("Actual Rows", 0) > 1000 or cost > 500:
            issues.append({
                "path": path,
                "node_type": node_type,
                "table": node.get("Relation Name"),
                "filter": node.get("Filter"),
                "rows": rows,
                "cost": cost
            })
    for child in node.get("Plans", []):
        issues.extend(find_seq_scans(child, path + " -> " + node_type))
    return issues

issues = find_seq_scans(plan["Plan"])
for issue in issues:
    print(f"[경고] Seq Scan on {issue['table']}")
    print(f"  필터: {issue['filter']}")
    print(f"  비용: {issue['cost']:.1f}, 실제 행 수: {issue['rows']}")
    print(f"  권장: CREATE INDEX ON {issue['table']}(컬럼)")
```

## 옵티마이저가 틀리는 경우

### 카디널리티 추정 오류

통계가 오래됐거나 데이터 왜곡(skew)이 심하면 옵티마이저는 잘못된 조인 알고리즘을 선택한다.

```sql
-- 특정 user_id에 주문이 몰린 경우 (VIP 사용자)
-- 옵티마이저는 "orders WHERE user_id=1 → 50행 예상"
-- 실제로는 50만 행 → Hash Join 대신 Nested Loop 선택 → 재앙
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 1;

-- 해결책 1: 통계 업데이트
ANALYZE orders;

-- 해결책 2: 히스토그램 버킷 증가
ALTER TABLE orders ALTER COLUMN user_id SET STATISTICS 500;
ANALYZE orders;

-- 해결책 3: 힌트 (PostgreSQL은 공식 힌트 미지원, pg_hint_plan 확장 사용)
-- /*+ HashJoin(o u) */ SELECT ...
```

### 플랜 캐싱의 함정

Prepared Statement는 첫 실행 시 생성된 실행 계획을 캐시한다. 그 시점의 파라미터로 최적화된 계획이 다른 파라미터에는 비효율적일 수 있다.

```sql
-- PostgreSQL: generic plan vs custom plan
PREPARE user_orders AS
  SELECT * FROM orders WHERE user_id = $1;

-- 첫 5회: custom plan (파라미터 값을 보고 최적화)
-- 이후: generic plan (파라미터 무관, 평균 비용 기반)
-- 특정 user_id가 이상치면 generic plan이 재앙이 될 수 있음

-- 플랜 캐시 강제 초기화
DEALLOCATE user_orders;
```

## 실전 최적화 팁

1. **EXPLAIN ANALYZE는 실제 실행**: `EXPLAIN`만 쓰면 예상값만 나온다. `ANALYZE`를 붙여야 실제 값을 얻는다. (단, `DELETE/UPDATE/INSERT`에는 주의 — 실제로 실행됨)

2. **통계를 최신으로 유지**: 대량 INSERT/DELETE 후에는 `ANALYZE`를 실행하거나 autovacuum 설정을 검토하라.

3. **선택도(selectivity)가 높은 컬럼에 인덱스**: `WHERE country = 'KR'`처럼 전체 데이터의 10~50%를 선택하는 조건은 Seq Scan이 오히려 빠를 수 있다. 인덱스는 선택도가 낮을 때(≤5%) 효과적이다.

4. **복합 인덱스 컬럼 순서**: `(a, b)` 인덱스는 `WHERE a = ?`와 `WHERE a = ? AND b = ?`에 사용되지만, `WHERE b = ?` 단독으로는 사용되지 않는다.

5. **부분 인덱스(Partial Index)**: 자주 쿼리되는 부분 집합만 인덱싱해 인덱스 크기를 줄인다.

```sql
-- 미처리 주문만 인덱싱 (전체의 1%라 가정)
CREATE INDEX idx_pending_orders ON orders(created_at)
WHERE status = 'pending';
```

## 마치며

쿼리 옵티마이저는 수십 년의 연구가 집약된 복잡한 시스템이지만, 핵심 원리는 단순하다: **통계를 기반으로 비용을 추정하고, 비용이 가장 낮은 실행 계획을 선택한다.** 개발자가 해야 할 일은 옵티마이저가 올바른 결정을 내릴 수 있도록 — 통계를 최신으로 유지하고, 적절한 인덱스를 제공하고, 실행 계획을 주기적으로 검토하는 것이다.

## 참고 자료
- [PostgreSQL Documentation — Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)
- [PostgreSQL Documentation — Planner/Optimizer](https://www.postgresql.org/docs/current/planner-optimizer.html)
- [Wikipedia: Query optimization](https://en.wikipedia.org/wiki/Query_optimization)
- [Wikipedia: Hash join](https://en.wikipedia.org/wiki/Hash_join)
