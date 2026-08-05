---
layout: post
title: "OLAP vs OLTP 완전 정복: 컬럼형 데이터베이스와 행형 데이터베이스의 설계 철학"
date: 2026-08-05
categories: [cs, computer-science]
tags: [database, olap, oltp, columnar-storage, row-storage, duckdb, clickhouse, data-warehouse, vectorized-execution]
---

"SELECT COUNT(*) FROM orders WHERE amount > 1000"을 실행할 때, PostgreSQL은 수억 개의 행을 모두 읽어야 할 수 있습니다. 반면 ClickHouse나 DuckDB 같은 컬럼형 데이터베이스는 `amount` 컬럼 하나만 읽고 끝냅니다. 이 차이가 어디서 오는지 이해하려면 **스토리지 레이아웃**과 **워크로드 특성**의 근본적인 차이를 파악해야 합니다. OLTP와 OLAP은 단순히 사용 목적이 다른 것이 아니라, 하드웨어의 물리적 특성에서 시작된 설계 철학의 차이입니다.

---

## 개념 설명

### OLTP: 행 기반 스토리지와 트랜잭션

**OLTP(Online Transaction Processing)**는 사용자의 실시간 요청을 빠르게 처리하는 워크로드입니다. 쇼핑몰 결제, 은행 이체, SNS 글쓰기 등이 해당합니다.

OLTP 쿼리의 특징:
- **단일 행 또는 소량의 행** 조회/수정
- **높은 동시성**: 수천 개의 트랜잭션이 동시 실행
- **짧은 응답 지연**: 수 밀리초 이내 응답 요구
- **쓰기 집중**: INSERT/UPDATE/DELETE가 자주 발생

```sql
-- 전형적인 OLTP 쿼리
SELECT * FROM users WHERE user_id = 12345;
UPDATE orders SET status = 'shipped' WHERE order_id = 98765;
INSERT INTO payments VALUES (nextval('seq'), 12345, 50000, NOW());
```

**행 기반 스토리지(Row-Oriented Storage)**는 이 워크로드에 최적입니다. 한 행의 모든 컬럼 데이터가 디스크에 연속으로 저장됩니다.

```
디스크 레이아웃:
페이지 1: [id=1][name=Alice][age=30][salary=50000]
          [id=2][name=Bob  ][age=25][salary=45000]
          [id=3][name=Carol][age=28][salary=60000]
```

`user_id = 12345` 조회 시 B-Tree 인덱스로 해당 행의 디스크 오프셋을 찾고, **단 하나의 페이지 I/O**로 그 행의 모든 컬럼을 읽어옵니다. 쓰기도 마찬가지로 한 위치에 연속된 데이터를 쓰면 됩니다.

### OLAP: 컬럼 기반 스토리지와 분석

**OLAP(Online Analytical Processing)**는 대규모 데이터셋에서 집계, 통계, 추세 분석을 수행하는 워크로드입니다. BI 대시보드, 매출 분석, 로그 분석이 해당합니다.

OLAP 쿼리의 특징:
- **전체 테이블 또는 대부분의 행** 스캔
- **소수의 컬럼**만 선택
- **낮은 동시성**: 복잡하지만 적은 수의 쿼리
- **읽기 집중**: 집계, 그룹화, 조인이 주를 이룸

```sql
-- 전형적인 OLAP 쿼리 (수억 행 스캔)
SELECT 
    date_trunc('month', order_date) AS month,
    product_category,
    SUM(amount) AS total_sales,
    COUNT(DISTINCT user_id) AS unique_buyers
FROM orders
WHERE order_date >= '2025-01-01'
GROUP BY 1, 2
ORDER BY 1, 3 DESC;
```

**컬럼 기반 스토리지(Column-Oriented Storage)**는 이 워크로드에 최적입니다. 각 컬럼이 독립된 저장 단위로 분리됩니다.

```
디스크 레이아웃:
id 컬럼:     [1][2][3][4][5]...
name 컬럼:   [Alice][Bob][Carol][Dave][Eve]...
age 컬럼:    [30][25][28][32][27]...
salary 컬럼: [50000][45000][60000][55000][48000]...
```

`SUM(salary)` 계산 시 salary 컬럼만 읽으면 됩니다. 나머지 컬럼은 디스크에서 전혀 읽지 않습니다.

---

## 왜 필요한가

### 컬럼형 스토리지의 세 가지 핵심 이점

#### 1. I/O 절감: 필요한 컬럼만 읽기

10개 컬럼, 10억 행 테이블에서 2개 컬럼만 집계하는 경우:

| 방식 | 읽는 데이터 |
|------|-------------|
| 행 기반 | 전체 테이블 (10 × 10억 행) |
| 컬럼 기반 | 2개 컬럼 (2 × 10억 행) |

이론적으로 5배 적은 I/O, 실제로는 페이지 크기와 접근 패턴에 따라 더 큰 차이를 보입니다.

#### 2. 압축 효율: 동일한 타입의 데이터가 연속으로

컬럼 스토리지는 같은 타입의 데이터가 연속으로 저장되므로 압축률이 극적으로 향상됩니다.

- **RLE(Run-Length Encoding)**: `[Male, Male, Male, ..., Female, Female]` 같은 낮은 카디널리티 컬럼에서 10~100배 압축
- **Delta Encoding**: 타임스탬프, 주가처럼 순차적으로 증가하는 데이터에서 효과적
- **Dictionary Encoding**: `product_category` 같은 카테고리 컬럼을 작은 정수 코드로 치환

ClickHouse 실측: 100GB 행 기반 데이터 → 컬럼 기반으로 전환 시 5~20GB로 압축 가능.

#### 3. 벡터화 실행(Vectorized Execution): SIMD와의 시너지

컬럼 데이터는 연속된 배열이므로 CPU의 SIMD(Single Instruction Multiple Data) 명령어를 자연스럽게 활용할 수 있습니다.

```
SIMD AVX-512: 한 번에 int64_t 8개 덧셈 = 8개 salary를 동시에 합산
```

DuckDB는 내부적으로 2048개의 값을 하나의 벡터 배치로 처리합니다. 행 기반 시스템의 Volcano 모델(한 번에 1행)과 비교해 캐시 효율과 분기 예측 성능이 수십 배 향상됩니다.

---

## 실제 구현 예제

### 예제 1: Python으로 행 기반 vs 컬럼 기반 성능 비교

```python
import time
import random
import statistics

# 데이터 생성: 100만 개의 주문 레코드
N = 1_000_000
random.seed(42)

# 행 기반 저장
rows = [
    {
        'order_id': i,
        'user_id': random.randint(1, 100000),
        'product': random.choice(['A', 'B', 'C', 'D', 'E']),
        'amount': random.uniform(1000, 100000),
        'status': random.choice(['pending', 'shipped', 'delivered', 'cancelled']),
        'created_at': f'2025-{random.randint(1,12):02d}-{random.randint(1,28):02d}',
    }
    for i in range(N)
]

# 컬럼 기반 저장
columns = {
    'order_id': [r['order_id'] for r in rows],
    'user_id':  [r['user_id']  for r in rows],
    'product':  [r['product']  for r in rows],
    'amount':   [r['amount']   for r in rows],
    'status':   [r['status']   for r in rows],
    'created_at': [r['created_at'] for r in rows],
}

def benchmark(name, func, repeat=5):
    times = []
    for _ in range(repeat):
        start = time.perf_counter()
        result = func()
        times.append(time.perf_counter() - start)
    avg_ms = statistics.mean(times) * 1000
    print(f"{name}: {avg_ms:.1f}ms (결과={result:.2f})")
    return avg_ms

# 쿼리: SUM(amount) WHERE status == 'delivered'
def row_sum_with_filter():
    return sum(r['amount'] for r in rows if r['status'] == 'delivered')

def col_sum_with_filter():
    statuses = columns['status']
    amounts = columns['amount']
    return sum(amounts[i] for i in range(N) if statuses[i] == 'delivered')

# 더 공정한 비교: NumPy 컬럼형 처리
try:
    import numpy as np
    amounts_arr = np.array(columns['amount'])
    status_arr = np.array(columns['status'])

    def numpy_sum_with_filter():
        mask = status_arr == 'delivered'
        return float(amounts_arr[mask].sum())

    t_row = benchmark("행 기반 (dict 리스트)", row_sum_with_filter)
    t_col = benchmark("컬럼 기반 (Python)", col_sum_with_filter)
    t_np  = benchmark("컬럼 기반 (NumPy)", numpy_sum_with_filter)

    print(f"\n성능 비율:")
    print(f"  행 기반 대비 NumPy: {t_row/t_np:.1f}x 빠름")
except ImportError:
    benchmark("행 기반 (dict 리스트)", row_sum_with_filter)
    benchmark("컬럼 기반 (Python)", col_sum_with_filter)
```

### 예제 2: DuckDB로 컬럼형 OLAP 분석 (Python)

```python
import duckdb
import time

# DuckDB는 완전한 컬럼형 임베디드 OLAP 데이터베이스
con = duckdb.connect(':memory:')

# 샘플 데이터 생성 (DuckDB SQL로 직접 생성)
con.execute("""
    CREATE TABLE orders AS
    SELECT
        range AS order_id,
        (random() * 100000)::INT AS user_id,
        ['electronics', 'clothing', 'food', 'books', 'toys'][1 + (random()*4)::INT] AS category,
        (random() * 99000 + 1000)::DECIMAL(10,2) AS amount,
        ['pending', 'shipped', 'delivered', 'cancelled'][1 + (random()*3)::INT] AS status,
        DATE '2024-01-01' + (random() * 365)::INT AS order_date
    FROM range(5000000)  -- 500만 건
""")

print("=== OLAP 쿼리 성능 테스트 ===\n")

queries = [
    ("월별 카테고리별 매출 집계", """
        SELECT 
            date_trunc('month', order_date) AS month,
            category,
            SUM(amount) AS total_sales,
            COUNT(*) AS order_count,
            AVG(amount) AS avg_amount
        FROM orders
        WHERE status = 'delivered'
        GROUP BY 1, 2
        ORDER BY 1, 3 DESC
    """),
    ("상위 10개 카테고리 매출", """
        SELECT 
            category,
            SUM(amount) AS total,
            COUNT(DISTINCT user_id) AS unique_buyers
        FROM orders
        GROUP BY category
        ORDER BY total DESC
        LIMIT 10
    """),
    ("Rolling 7일 매출 추이", """
        SELECT
            order_date,
            SUM(amount) AS daily_sales,
            SUM(SUM(amount)) OVER (
                ORDER BY order_date 
                ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
            ) AS rolling_7d
        FROM orders
        WHERE status != 'cancelled'
        GROUP BY order_date
        ORDER BY order_date
        LIMIT 30
    """),
]

for name, sql in queries:
    start = time.perf_counter()
    result = con.execute(sql).fetchall()
    elapsed = (time.perf_counter() - start) * 1000
    print(f"[{name}]")
    print(f"  실행시간: {elapsed:.1f}ms, 결과 행수: {len(result)}")
    if result:
        print(f"  첫 번째 결과: {result[0]}")
    print()

# EXPLAIN으로 실행 계획 확인
print("=== 실행 계획 (벡터화 스캔 확인) ===")
explain = con.execute("EXPLAIN " + queries[0][1]).fetchall()
for row in explain:
    print(row[1][:500])  # 처음 500자만 출력

con.close()
```

출력 예시:
```
[월별 카테고리별 매출 집계]
  실행시간: 85.3ms, 결과 행수: 48

[상위 10개 카테고리 매출]
  실행시간: 42.1ms, 결과 행수: 5

[Rolling 7일 매출 추이]
  실행시간: 63.7ms, 결과 행수: 30
```

500만 행을 대상으로 복잡한 집계 쿼리가 수십 밀리초 이내에 완료됩니다.

---

## 주의사항 및 팁

### 언제 OLTP를, 언제 OLAP을 선택하는가

| 기준 | OLTP (행 기반) | OLAP (컬럼 기반) |
|------|---------------|-----------------|
| 주요 연산 | 단일 행 조회/수정 | 대량 집계 |
| 트랜잭션 | ACID 필수 | 완화 가능 |
| 쓰기 패턴 | 빈번한 삽입/수정 | 배치 삽입 |
| 쿼리 패턴 | 특정 행 조회 | 컬럼 집계 |
| 대표 시스템 | PostgreSQL, MySQL | ClickHouse, DuckDB, Redshift |

### HTAP: 두 마리 토끼를 잡으려는 시도

최신 트렌드는 OLTP와 OLAP을 하나의 시스템에서 처리하는 **HTAP(Hybrid Transactional/Analytical Processing)**입니다.

- **TiDB**: 행 기반 TiKV와 컬럼 기반 TiFlash를 동시 운영
- **MySQL HeatWave**: InnoDB(행 기반) + 인메모리 컬럼형 가속 레이어
- **SingleStore**: OLTP용 행 기반 + OLAP용 컬럼 기반 혼합 스토리지

그러나 최적화 방향이 근본적으로 다르기 때문에 완벽한 HTAP은 어렵고, 일반적으로 각각에서 최적 성능에 비해 절충점이 존재합니다.

### 파티셔닝과 인덱스 전략

컬럼형 데이터베이스에서도 모든 컬럼을 매번 스캔하면 비효율적입니다. ClickHouse의 **Sparse Index**, DuckDB의 **Zone Map**, Parquet의 **Row Group Statistics** 같은 메타데이터 기반 프루닝이 핵심입니다:

```sql
-- ClickHouse: 파티션 + 정렬 키로 데이터 프루닝
CREATE TABLE orders (
    order_date Date,
    category LowCardinality(String),
    amount Float64
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(order_date)  -- 월별 파티션
ORDER BY (category, order_date);   -- 정렬 키로 스킵 인덱스 활성화
```

### Late Materialization(늦은 구체화)

컬럼 기반 시스템의 고급 최적화 기법으로, 필터 컬럼을 먼저 평가하여 일치하는 행 번호(row ID)만 추린 뒤, SELECT 대상 컬럼을 마지막에 읽는 방식입니다. 선택성이 낮은(많은 행이 제거되는) 필터에서 I/O를 극적으로 줄입니다.

---

## 참고 자료
- [Column-oriented DBMS - Wikipedia](https://en.wikipedia.org/wiki/Column-oriented_DBMS)
- [Online analytical processing - Wikipedia](https://en.wikipedia.org/wiki/Online_analytical_processing)
- [Why DuckDB - DuckDB Documentation](https://duckdb.org/why_duckdb.html)
- [OLTP vs OLAP - ClickHouse](https://clickhouse.com/resources/engineering/oltp-vs-olap)
