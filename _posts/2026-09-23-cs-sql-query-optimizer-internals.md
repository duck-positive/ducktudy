---
layout: post
title: "SQL 쿼리 옵티마이저 완전 정복: 비용 기반 최적화와 조인 순서 결정의 내부 원리"
date: 2026-09-23
categories: [cs, computer-science]
tags: [database, sql, query-optimizer, cost-based-optimizer, cardinality, join-ordering, postgresql, execution-plan]
---

데이터베이스를 사용하다 보면 같은 결과를 반환하는 두 쿼리의 실행 시간이 수십 배 차이 나는 경험을 하게 됩니다. 인덱스를 걸었는데도 왜 느린지, EXPLAIN을 봐도 무슨 말인지 모르겠는 순간이 옵니다. 이 모든 것의 열쇠는 **쿼리 옵티마이저(Query Optimizer)**에 있습니다.

## 개념 설명: 쿼리 옵티마이저란?

SQL 쿼리 옵티마이저는 데이터베이스 관리 시스템(DBMS)에서 SQL 쿼리를 받아 **가장 효율적인 실행 계획(Execution Plan)**을 찾아내는 핵심 모듈입니다. 사용자가 "어떤 데이터를 가져올까(WHAT)"를 SQL로 기술하면, 옵티마이저는 "어떻게 그 데이터를 가져올까(HOW)"를 결정합니다.

SQL이 **선언적 언어(Declarative Language)**인 이유가 여기 있습니다. 개발자는 결과만 기술하고, 최적의 처리 경로는 옵티마이저에게 맡기는 것입니다.

쿼리 옵티마이저는 크게 두 가지 유형으로 나뉩니다:

- **규칙 기반 옵티마이저(RBO, Rule-Based Optimizer)**: 미리 정의된 휴리스틱 규칙에 따라 실행 계획을 선택합니다. 예를 들어 "인덱스가 있으면 무조건 사용"처럼 작동합니다. Oracle 초기 버전에서 사용했습니다.
- **비용 기반 옵티마이저(CBO, Cost-Based Optimizer)**: 테이블 통계 정보와 수학적 비용 모델을 사용하여 최적의 실행 계획을 선택합니다. 현대 DBMS(PostgreSQL, MySQL InnoDB, Oracle, SQL Server)는 모두 CBO를 사용합니다.

### 옵티마이저 파이프라인

CBO는 다음 단계를 순서대로 실행합니다:

1. **파싱(Parsing)**: SQL 텍스트를 파스 트리(Parse Tree)로 변환합니다.
2. **논리 계획 생성(Logical Planning)**: 파스 트리를 관계형 대수(Relational Algebra) 표현으로 변환합니다. 이 단계에서 불필요한 조건 제거, 뷰 전개(View Expansion) 등 논리적 변환이 이루어집니다.
3. **계획 공간 탐색(Plan Space Exploration)**: 가능한 물리적 실행 계획들을 열거합니다. 같은 논리 계획도 Hash Join, Nested Loop, Merge Join 등 다양한 방식으로 실행될 수 있습니다.
4. **비용 추정(Cost Estimation)**: 각 계획의 예상 비용을 계산합니다.
5. **최적 계획 선택(Plan Selection)**: 가장 낮은 비용의 계획을 실행자(Executor)에게 전달합니다.

## 왜 필요한가: 조인 순서의 폭발적 복잡도

n개의 테이블을 조인할 때, 가능한 조인 순서는 이론적으로 **n! (n 팩토리얼)**개입니다. 10개 테이블을 조인하면 3,628,800가지 순서가 가능합니다. 각 조인 순서에서 어떤 조인 알고리즘을 사용할지까지 고려하면 탐색 공간은 훨씬 커집니다.

사람이 이를 직접 최적화하는 것은 불가능합니다. 옵티마이저가 수학적 비용 모델과 효율적인 탐색 알고리즘으로 이 문제를 해결합니다.

### 카디널리티 추정 (Cardinality Estimation)

카디널리티(Cardinality)는 각 연산자가 처리하거나 반환하는 **행(row)의 수**입니다. 정확한 카디널리티 추정은 효율적인 실행 계획 선택의 핵심입니다. 카디널리티를 10배 잘못 추정하면 완전히 잘못된 계획이 선택될 수 있습니다.

옵티마이저는 **통계 정보(Statistics)**를 활용합니다:
- **테이블 통계**: 전체 행 수(`n_live_tup`), 페이지 수(`relpages`)
- **컬럼 통계**: 유니크 값의 수(NDV, Number of Distinct Values), NULL 비율, 최솟값/최댓값, 히스토그램(Most Common Values + 나머지 분포)
- **인덱스 통계**: 인덱스 높이, 클러스터링 팩터(데이터가 인덱스 순서와 얼마나 일치하는지)

**선택도(Selectivity)**는 WHERE 절이 전체 행 중 몇 %를 필터링하는지를 나타냅니다:
- 동등 조건(`col = val`): `sel = 1 / NDV`
- 범위 조건(`col > val`): `sel = (max - val) / (max - min)`
- LIKE 조건: MCV(Most Common Values) 히스토그램 기반
- 다중 컬럼 조건: 독립성 가정 → 각 선택도의 곱 (상관관계가 있으면 부정확해짐)

PostgreSQL에서 통계를 확인하려면:
```sql
SELECT * FROM pg_stats WHERE tablename = 'orders' AND attname = 'status';
```

## 실제 구현 예제

### 예제 1: PostgreSQL EXPLAIN ANALYZE로 실행 계획 분석하기

```sql
-- 테이블 생성
CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INT NOT NULL,
    product_id INT NOT NULL,
    amount DECIMAL(10,2),
    status VARCHAR(20) DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    country CHAR(2)
);

CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,
    category VARCHAR(50),
    price DECIMAL(10,2)
);

-- 인덱스 생성
CREATE INDEX idx_orders_customer ON orders(customer_id);
CREATE INDEX idx_orders_product  ON orders(product_id);
CREATE INDEX idx_orders_created  ON orders(created_at);

-- 통계 최신화 (ANALYZE 실행)
ANALYZE orders;
ANALYZE customers;
ANALYZE products;

-- 실행 계획 분석: 버퍼 사용량, 실제 실행 시간까지 확인
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT
    c.name,
    p.category,
    SUM(o.amount)  AS total_amount,
    COUNT(*)       AS order_count
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN products p  ON o.product_id  = p.product_id
WHERE
    c.country    = 'KR'
    AND o.created_at >= '2026-01-01'
    AND p.price  > 50000
GROUP BY c.name, p.category
ORDER BY total_amount DESC;

/*
예시 출력 (단순화):
HashAggregate  (cost=15234.56..15289.56 rows=5500)
  ->  Hash Join  (cost=1234.56..14934.56 rows=30000)
        Hash Cond: (o.product_id = p.product_id)
        ->  Hash Join  (cost=567.89..12123.45 rows=45000)
              Hash Cond: (o.customer_id = c.customer_id)
              ->  Bitmap Heap Scan on orders o
                    Recheck Cond: (created_at >= '2026-01-01')
                    ->  Bitmap Index Scan on idx_orders_created
              ->  Hash  (cost=456.78..456.78 rows=8888)
                    ->  Seq Scan on customers c
                          Filter: (country = 'KR')
        ->  Hash  (cost=345.67..345.67 rows=7200)
              ->  Seq Scan on products p
                    Filter: (price > 50000)
*/

-- 다중 컬럼 상관관계를 위한 확장 통계 (PostgreSQL 10+)
-- 예: city와 zip_code는 강하게 상관되어 있음
CREATE STATISTICS orders_stat (dependencies, ndistinct, mcv)
    ON customer_id, status FROM orders;
ANALYZE orders;

-- 조인 순서 강제 (실험용 - 운영에서는 주의)
SET join_collapse_limit = 1;  -- FROM 절 순서 그대로 조인
SET enable_hashjoin = off;    -- Hash Join 비활성화 (Nested Loop, Merge Join만 허용)
```

계획을 해석하는 핵심 포인트:
- `cost=시작비용..전체비용`: 첫 행까지의 비용과 모든 행의 비용
- `rows=추정행수`: 옵티마이저가 예측한 결과 행 수
- `actual time=X..Y rows=Z`: 실제 실행 시간과 실제 행 수
- 추정 행 수와 실제 행 수가 크게 다르면 통계가 오래됐거나 상관관계 문제

### 예제 2: Python으로 구현하는 비용 기반 조인 순서 탐색 시뮬레이터

동적 계획법(DP)으로 최적 조인 순서를 찾는 핵심 알고리즘을 구현합니다.

```python
from __future__ import annotations
from dataclasses import dataclass
from itertools import combinations

# ── 통계 모델 ────────────────────────────────────────────────────────────────

@dataclass
class TableStats:
    name: str
    row_count: int
    page_count: int  # 디스크 페이지 수

@dataclass
class JoinPredicate:
    table_a: str
    table_b: str
    selectivity: float  # 조인 결과 선택도 (FK-PK 조인: 1/NDV)

# ── 비용 상수 (PostgreSQL 기본값 참고) ──────────────────────────────────────

SEQ_PAGE_COST   = 1.0   # 순차 I/O 비용 단위
RAND_PAGE_COST  = 4.0   # 랜덤 I/O 비용 단위 (SSD면 1.1 정도로 낮춤)
CPU_TUPLE_COST  = 0.01  # 행당 CPU 처리 비용
CPU_JOIN_COST   = 0.025 # 조인 연산당 CPU 비용

# ── 비용 추정 함수 ────────────────────────────────────────────────────────────

def seq_scan_cost(stats: TableStats, selectivity: float = 1.0) -> tuple[float, int]:
    """순차 스캔 비용과 출력 행 수를 반환합니다."""
    io_cost  = stats.page_count * SEQ_PAGE_COST
    cpu_cost = stats.row_count  * CPU_TUPLE_COST
    out_rows = max(1, int(stats.row_count * selectivity))
    return io_cost + cpu_cost, out_rows

def hash_join_cost(
    outer_rows: int, inner_rows: int, inner_pages: int, join_sel: float
) -> tuple[float, int]:
    """Hash Join 비용 추정: Build(내부) + Probe(외부) 단계."""
    build_cost = inner_pages * SEQ_PAGE_COST + inner_rows * CPU_TUPLE_COST
    probe_cost = outer_rows  * CPU_JOIN_COST
    out_rows   = max(1, int(outer_rows * inner_rows * join_sel))
    return build_cost + probe_cost, out_rows

def nested_loop_cost(
    outer_rows: int, inner_pages: int, join_sel: float
) -> tuple[float, int]:
    """Nested Loop Join 비용 추정: 외부 행마다 내부 테이블 스캔."""
    cost     = outer_rows * (inner_pages * RAND_PAGE_COST + CPU_JOIN_COST)
    out_rows = max(1, int(outer_rows * join_sel))
    return cost, out_rows

# ── 핵심: DP 기반 최적 조인 순서 탐색 ───────────────────────────────────────

def find_best_join_order(
    tables: list[str],
    table_stats: dict[str, TableStats],
    predicates: list[JoinPredicate],
) -> tuple[list[str], float]:
    """
    동적 계획법으로 최적 조인 순서를 찾습니다.
    PostgreSQL의 standard_join_search()에 해당하는 알고리즘입니다.
    시간복잡도: O(3^n) — n=10일 때 약 59,049번 연산
    """
    # 조인 선택도 인덱스 구축
    sel_index: dict[frozenset, float] = {}
    for p in predicates:
        key = frozenset([p.table_a, p.table_b])
        sel_index[key] = p.selectivity

    # 단일 테이블 초기화
    best_cost: dict[frozenset, float] = {}
    best_rows: dict[frozenset, int]   = {}
    best_plan: dict[frozenset, list]  = {}

    for t in tables:
        s           = frozenset([t])
        cost, rows  = seq_scan_cost(table_stats[t])
        best_cost[s] = cost
        best_rows[s] = rows
        best_plan[s] = [t]

    # DP: 크기 2, 3, ..., n인 집합에 대해 최적 계획 탐색
    for size in range(2, len(tables) + 1):
        for subset_list in combinations(tables, size):
            subset = frozenset(subset_list)
            min_cost, min_rows, min_plan = float("inf"), 0, []

            # 가능한 모든 (left, right) 분할 시도
            for split_size in range(1, size):
                for left_list in combinations(subset_list, split_size):
                    left  = frozenset(left_list)
                    right = subset - left

                    if left not in best_cost or right not in best_cost:
                        continue

                    # 두 집합 사이에 조인 조건이 있는지 확인 (Cartesian Product 방지)
                    join_sel = _find_join_sel(left, right, sel_index)
                    if join_sel is None:
                        join_sel = 0.01  # Cross join은 매우 비쌈

                    l_cost, l_rows = best_cost[left],  best_rows[left]
                    r_cost, r_rows = best_cost[right], best_rows[right]
                    r_pages = sum(table_stats[t].page_count for t in right)

                    # Hash Join과 Nested Loop 중 더 저렴한 것 선택
                    hj_cost, hj_rows = hash_join_cost(l_rows, r_rows, r_pages, join_sel)
                    nl_cost, nl_rows = nested_loop_cost(l_rows, r_pages, join_sel)

                    if hj_cost <= nl_cost:
                        join_cost, join_rows = hj_cost, hj_rows
                        join_type = "HashJoin"
                    else:
                        join_cost, join_rows = nl_cost, nl_rows
                        join_type = "NestedLoop"

                    total_cost = l_cost + r_cost + join_cost

                    if total_cost < min_cost:
                        min_cost = total_cost
                        min_rows = join_rows
                        min_plan = best_plan[left] + best_plan[right]

            best_cost[subset] = min_cost
            best_rows[subset] = min_rows
            best_plan[subset] = min_plan

    all_tables = frozenset(tables)
    return best_plan[all_tables], best_cost[all_tables]


def _find_join_sel(
    left: frozenset, right: frozenset, sel_index: dict[frozenset, float]
) -> float | None:
    for l in left:
        for r in right:
            key = frozenset([l, r])
            if key in sel_index:
                return sel_index[key]
    return None


# ── 사용 예시 ─────────────────────────────────────────────────────────────────

if __name__ == "__main__":
    stats = {
        "orders":    TableStats("orders",    1_000_000, 10_000),
        "customers": TableStats("customers",    50_000,    500),
        "products":  TableStats("products",     10_000,    100),
    }

    predicates = [
        JoinPredicate("orders", "customers", 1.0 / 50_000),  # PK-FK
        JoinPredicate("orders", "products",  1.0 / 10_000),  # PK-FK
    ]

    order, cost = find_best_join_order(
        tables=["orders", "customers", "products"],
        table_stats=stats,
        predicates=predicates,
    )

    print("=== 쿼리 옵티마이저 시뮬레이터 ===")
    print(f"최적 조인 순서 : {' → '.join(order)}")
    print(f"예상 총 비용    : {cost:,.2f}")
    print()
    print("✅ 작은 테이블(customers, products)을 먼저 조인하여")
    print("   orders 테이블의 대량 데이터 스캔을 최소화합니다.")
```

## 주의사항 및 팁

**1. 통계를 주기적으로 최신화하라**

대량의 INSERT/UPDATE/DELETE 이후에는 반드시 `ANALYZE`를 실행하세요. PostgreSQL의 `autovacuum`이 자동으로 실행하지만, 대용량 일괄 작업 후에는 수동 실행이 필요합니다.

```sql
-- 특정 테이블 통계 업데이트
ANALYZE orders;
-- 통계 샘플 크기 늘리기 (정확도 향상, 비용 증가)
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 500;
```

**2. 실행 계획을 이해하고 힌트 사용을 최소화하라**

강제 힌트(Force Index, Join Hint)는 통계가 변할 때 오히려 비효율적이 될 수 있습니다. 힌트 대신 통계를 정확하게 유지하고, 필요하면 Extended Statistics를 사용하세요.

**3. 파라미터 스니핑(Parameter Sniffing) 문제**

컴파일된 실행 계획은 최초 실행 시의 파라미터로 최적화됩니다. 특정 파라미터에서만 비효율적이라면 해당 쿼리의 실행 계획 캐시를 초기화하거나, 동적 SQL로 전환하는 것을 고려하세요.

**4. 카디널리티 추정 오류를 EXPLAIN으로 진단하라**

`rows=예측` 과 `actual rows=실제` 차이가 크면 통계 문제입니다. 10배 이상 차이나면 즉시 조사하세요. PostgreSQL의 `pg_stats`, `pg_statistic` 뷰로 통계 품질을 확인할 수 있습니다.

**5. GEQO 임계값 조정**

PostgreSQL은 기본적으로 테이블이 12개 이상이면 유전 알고리즘(GEQO)을 사용합니다. 복잡한 쿼리에서 최적 계획을 찾지 못한다면 `geqo_threshold`를 높이거나 낮추어 실험해보세요.

## 참고 자료

- [PostgreSQL: Planner/Optimizer 공식 문서](https://www.postgresql.org/docs/current/planner-optimizer.html)
- [PostgreSQL: Query Planning 설정](https://www.postgresql.org/docs/current/runtime-config-query.html)
- [PostgreSQL 내부 구조 - Query Planning (GitHub)](https://github.com/postgres/postgres/tree/master/src/backend/optimizer)
- [CMU 15-445 Database Systems Lecture Notes](https://15445.courses.cs.cmu.edu/fall2023/notes/15-optimization.pdf)
