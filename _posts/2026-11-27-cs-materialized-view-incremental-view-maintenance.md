---
layout: post
title: "구체화 뷰와 증분 뷰 유지보수(IVM): 데이터베이스 집계 쿼리 최적화의 비밀"
date: 2026-11-27
categories: [cs, computer-science]
tags: [database, materialized-view, ivm, incremental-view-maintenance, postgresql, sql, olap]
---

## 개념 설명

**구체화 뷰(Materialized View)**는 데이터베이스에서 쿼리의 결과를 물리적으로 저장(캐싱)한 객체입니다. 일반 뷰(View)가 쿼리 실행 시마다 기반 테이블을 재조회하는 것과 달리, 구체화 뷰는 결과를 디스크에 테이블처럼 저장해 빠른 읽기를 가능하게 합니다.

```
일반 뷰 (Virtual View):
  SELECT time → 기반 테이블 풀 스캔 → 결과 반환  (매번 O(n))

구체화 뷰 (Materialized View):
  미리 계산된 결과가 디스크에 저장됨
  SELECT time → 저장된 결과 반환                (O(1) 혹은 O(결과 크기))
  단, 기반 테이블 변경 시 뷰가 "오래된(stale)" 상태가 됨
```

**증분 뷰 유지보수(Incremental View Maintenance, IVM)**는 기반 테이블이 변경될 때 구체화 뷰 전체를 재계산(full recomputation)하지 않고, 변경분(delta, Δ)만을 반영해 뷰를 최신 상태로 유지하는 기술입니다.

핵심 아이디어는 뷰 쿼리 `V = Q(T)`에서, 기반 테이블에 변경 `ΔT`가 발생했을 때 `ΔV = f(ΔT, V, T)`를 계산해 `V ← V + ΔV`로 업데이트하는 것입니다. 전체 `Q(T + ΔT)`를 재계산하지 않아도 됩니다.

---

## 왜 필요한가

대용량 집계 쿼리(GROUP BY, JOIN, 윈도우 함수 등)를 매번 실행하면 수백만 개의 행을 읽어야 합니다. OLAP(Online Analytical Processing) 환경에서는 이러한 쿼리가 보고서, 대시보드, API 응답에 반복적으로 사용됩니다.

**실제 사례:**

| 집계 | 기반 테이블 크기 | 단순 쿼리 시간 | 구체화 뷰 조회 |
|------|---------------|-------------|-------------|
| 월별 사용자 매출 | 주문 10억 행 | 수십 초 ~ 분 | 밀리초 |
| 상품별 재고 현황 | 재고이동 5억 행 | 수 초 | 밀리초 |
| 실시간 대시보드 | 이벤트 로그 1조 행 | 불가 | 밀리초 |

접근법 비교:

| 방법 | 읽기 속도 | 쓰기 오버헤드 | 복잡도 | 데이터 신선도 |
|------|---------|------------|--------|------------|
| 단순 뷰 (매번 재계산) | 느림 O(n) | 없음 | 낮음 | 항상 최신 |
| 전체 재계산 구체화 뷰 | 빠름 | 갱신 시 O(n) | 낮음 | 갱신 주기에 따름 |
| IVM (증분 유지보수) | 빠름 | O(Δ) | 높음 | 거의 실시간 |

---

## 실제 구현 예제

### 예제 1: PostgreSQL에서 구체화 뷰 생성과 활용

```sql
-- 기반 테이블
CREATE TABLE orders (
    id          BIGSERIAL PRIMARY KEY,
    user_id     INT          NOT NULL,
    product_id  INT          NOT NULL,
    amount      DECIMAL(10, 2) NOT NULL,
    status      VARCHAR(20)  NOT NULL DEFAULT 'pending',
    created_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

-- 구체화 뷰: 월별 사용자 매출 집계
CREATE MATERIALIZED VIEW monthly_user_sales AS
SELECT
    DATE_TRUNC('month', created_at)::DATE AS month,
    user_id,
    COUNT(*)                               AS order_count,
    SUM(amount)                            AS total_amount,
    AVG(amount)                            AS avg_amount
FROM orders
WHERE status = 'completed'
GROUP BY 1, 2
WITH DATA;  -- 즉시 데이터 채움 (WITH NO DATA는 비어있는 상태로 생성)

-- 구체화 뷰에도 인덱스 생성 가능
CREATE UNIQUE INDEX idx_muser_month_user
    ON monthly_user_sales (month, user_id);

CREATE INDEX idx_muser_user
    ON monthly_user_sales (user_id);

-- 빠른 집계 조회
EXPLAIN ANALYZE
SELECT month, total_amount
FROM monthly_user_sales
WHERE user_id = 42
ORDER BY month DESC;
-- Index Scan on idx_muser_user, 비용 ≈ 0.15 ms

-- 전체 재계산 갱신 (뷰 잠금 발생)
REFRESH MATERIALIZED VIEW monthly_user_sales;

-- 동시성 갱신 (조회를 차단하지 않음, UNIQUE 인덱스 필요)
REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_user_sales;

-- 갱신 스케줄링 (pg_cron 확장 사용)
SELECT cron.schedule(
    'refresh-monthly-sales',
    '0 * * * *',  -- 매 시간 정각
    'REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_user_sales'
);
```

---

### 예제 2: 트리거 기반 증분 뷰 유지보수 (PostgreSQL + Python)

전체 재계산 없이 INSERT/DELETE 시 델타만 반영하는 IVM을 직접 구현합니다.

```sql
-- 증분 유지보수용 집계 테이블 (구체화 뷰 역할)
CREATE TABLE mat_daily_product_sales (
    product_id   INT     NOT NULL,
    day          DATE    NOT NULL,
    order_count  BIGINT  NOT NULL DEFAULT 0,
    total_amount DECIMAL(12, 2) NOT NULL DEFAULT 0,
    PRIMARY KEY (product_id, day)
);

-- 델타 적용 함수 (upsert 방식)
CREATE OR REPLACE FUNCTION apply_product_delta(
    p_product_id  INT,
    p_day         DATE,
    p_count_delta BIGINT,
    p_amount_delta DECIMAL(12, 2)
) RETURNS VOID AS $$
BEGIN
    INSERT INTO mat_daily_product_sales
        (product_id, day, order_count, total_amount)
    VALUES
        (p_product_id, p_day, p_count_delta, p_amount_delta)
    ON CONFLICT (product_id, day) DO UPDATE SET
        order_count  = mat_daily_product_sales.order_count  + EXCLUDED.order_count,
        total_amount = mat_daily_product_sales.total_amount + EXCLUDED.total_amount;
END;
$$ LANGUAGE plpgsql;

-- INSERT 트리거 함수
CREATE OR REPLACE FUNCTION trg_orders_after_insert() RETURNS TRIGGER AS $$
BEGIN
    IF NEW.status = 'completed' THEN
        PERFORM apply_product_delta(
            NEW.product_id,
            DATE(NEW.created_at),
            +1,
            NEW.amount
        );
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- DELETE 트리거 함수
CREATE OR REPLACE FUNCTION trg_orders_after_delete() RETURNS TRIGGER AS $$
BEGIN
    IF OLD.status = 'completed' THEN
        PERFORM apply_product_delta(
            OLD.product_id,
            DATE(OLD.created_at),
            -1,
            -OLD.amount
        );
    END IF;
    RETURN OLD;
END;
$$ LANGUAGE plpgsql;

-- UPDATE 트리거 함수 (상태 변경 감지)
CREATE OR REPLACE FUNCTION trg_orders_after_update() RETURNS TRIGGER AS $$
BEGIN
    -- 기존 행의 효과를 되돌림
    IF OLD.status = 'completed' THEN
        PERFORM apply_product_delta(
            OLD.product_id, DATE(OLD.created_at), -1, -OLD.amount
        );
    END IF;
    -- 새 행의 효과를 반영
    IF NEW.status = 'completed' THEN
        PERFORM apply_product_delta(
            NEW.product_id, DATE(NEW.created_at), +1, NEW.amount
        );
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- 트리거 등록
CREATE TRIGGER orders_after_insert
    AFTER INSERT ON orders FOR EACH ROW
    EXECUTE FUNCTION trg_orders_after_insert();

CREATE TRIGGER orders_after_delete
    AFTER DELETE ON orders FOR EACH ROW
    EXECUTE FUNCTION trg_orders_after_delete();

CREATE TRIGGER orders_after_update
    AFTER UPDATE OF status, product_id, amount, created_at ON orders
    FOR EACH ROW EXECUTE FUNCTION trg_orders_after_update();
```

Python에서 이를 사용하는 클라이언트 코드:

```python
import psycopg2
from contextlib import contextmanager
from datetime import datetime
from decimal import Decimal

@contextmanager
def transaction(conn):
    try:
        yield conn.cursor()
        conn.commit()
    except Exception:
        conn.rollback()
        raise

def demo_ivm(dsn: str):
    conn = psycopg2.connect(dsn)

    with transaction(conn) as cur:
        # INSERT → 트리거가 자동으로 집계 갱신
        cur.execute("""
            INSERT INTO orders (user_id, product_id, amount, status, created_at)
            VALUES (%s, %s, %s, %s, %s)
            RETURNING id
        """, (42, 100, Decimal('29.99'), 'completed', datetime(2026, 11, 27, 10, 0, 0)))
        order_id = cur.fetchone()[0]
        print(f"주문 생성: id={order_id}")

    # 별도 갱신 없이 즉시 최신 집계 조회 가능
    with transaction(conn) as cur:
        cur.execute("""
            SELECT product_id, day, order_count, total_amount
            FROM mat_daily_product_sales
            WHERE product_id = 100
        """)
        for row in cur.fetchall():
            print(f"집계: product={row[0]}, day={row[1]}, "
                  f"count={row[2]}, total={row[3]}")

    # 주문 상태 변경 (pending → completed) → 집계 자동 반영
    with transaction(conn) as cur:
        cur.execute("""
            INSERT INTO orders (user_id, product_id, amount, status, created_at)
            VALUES (%s, %s, %s, %s, %s)
        """, (42, 100, Decimal('49.99'), 'pending', datetime(2026, 11, 27, 11, 0, 0)))

    with transaction(conn) as cur:
        cur.execute("UPDATE orders SET status = 'completed' WHERE user_id = 42 AND status = 'pending'")

    with transaction(conn) as cur:
        cur.execute("SELECT order_count, total_amount FROM mat_daily_product_sales WHERE product_id = 100")
        row = cur.fetchone()
        print(f"갱신 후 집계: count={row[0]}, total={row[1]}")
        # count=2, total=79.98 (즉각 반영)

    conn.close()
```

---

## 주의사항/팁

**1. 전체 재계산 vs 증분 갱신 선택 기준**

단순히 조회 성능이 목표라면 `REFRESH MATERIALIZED VIEW CONCURRENTLY`를 주기적으로 실행하는 것으로 충분한 경우가 많습니다. 트리거 기반 IVM은 복잡도를 높이므로 쓰기가 초당 수백 건 이상이고 실시간 집계가 필수인 경우에만 채택하세요.

**2. IVM을 지원하는 도구**

- **pg_ivm** (PostgreSQL 확장): 순수 SQL 뷰에 대한 증분 유지보수를 자동화
- **DBToaster**: 복잡한 중첩 집계 쿼리에 대해 고차 델타(higher-order delta) 코드를 자동 생성하는 연구 시스템
- **Snowflake Dynamic Tables**: 기반 테이블 변경을 지연 없이 반영하는 클라우드 DW 기능
- **Databricks Materialized Views**: Delta Lake 위에서 동작하는 증분 갱신
- **Apache Flink**: 스트리밍 집계로 IVM을 실시간 처리

**3. 증분 유지보수가 어려운 쿼리**

다음 연산은 델타 계산이 복잡하거나 불가능합니다:
- `DISTINCT` (비집계): 중복 제거 기준이 변경될 수 있어 전체 재계산 필요
- `MIN`/`MAX`: 최솟값 행이 삭제되면 전체 스캔 필요
- `LIMIT`/`OFFSET`: 순서 의존성
- 재귀 쿼리 (`WITH RECURSIVE`)
- 비결정적 함수 (`NOW()`, `RANDOM()`)

**4. 쓰기 오버헤드 주의**

트리거 기반 IVM은 모든 INSERT/UPDATE/DELETE마다 추가 작업이 발생합니다. 대용량 배치 INSERT 시 성능 저하를 유발할 수 있으므로, 배치 처리 중에는 트리거를 비활성화하고 완료 후 한 번에 집계를 재계산하는 방식을 고려하세요.

**5. 클라우드 환경에서의 활용**

Snowflake, Redshift, BigQuery 등 클라우드 DW에서 구체화 뷰는 쿼리 최적화기가 자동으로 선택하는 "자동 갱신 구체화 뷰(Auto-Refresh Materialized View)"를 지원합니다. 사용자가 명시적으로 조회하지 않아도 옵티마이저가 적합한 구체화 뷰를 자동으로 선택해 성능을 최적화합니다.

## 참고 자료
- [PostgreSQL Materialized Views 공식 문서](https://www.postgresql.org/docs/current/rules-materializedviews.html)
- [PostgreSQL CREATE MATERIALIZED VIEW 레퍼런스](https://www.postgresql.org/docs/current/sql-creatematerializedview.html)
- [pg_ivm: Incremental View Maintenance for PostgreSQL](https://github.com/sraoss/pg_ivm)
- [DBToaster: Higher-order Delta Processing for Dynamic, Frequently Fresh Views](https://dbtoaster.github.io/)
