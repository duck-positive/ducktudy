---
layout: post
title: "쌍시제 데이터 모델링(Bitemporal Data Modeling) 완전 정복"
date: 2026-11-24
categories: [cs, computer-science]
tags: [database, temporal, bitemporal, sql, data-modeling, valid-time, transaction-time]
---

데이터베이스 설계에서 "언제"를 추적하는 방법은 생각보다 복잡합니다. 단순히 `created_at`, `updated_at`을 추가하는 것만으로는 충분하지 않은 경우가 많습니다. 금융, 의료, 보험, 회계 시스템에서는 두 가지 독립적인 시간 차원을 모두 추적해야 합니다. 이것이 바로 쌍시제 데이터 모델링(Bitemporal Data Modeling)입니다.

## 개념 설명: 두 가지 시간 차원

쌍시제 모델은 다음 두 차원을 독립적으로 추적합니다.

### 유효 시간(Valid Time, Application Time)

현실 세계에서 데이터가 참(true)인 기간입니다. 예를 들어 Alice가 마케팅팀에 소속된 실제 기간은 2024-01-15부터 2024-08-31까지입니다. 이 사실이 데이터베이스에 언제 입력되었든 상관없이, 현실 세계에서의 유효 기간을 나타냅니다.

### 트랜잭션 시간(Transaction Time, System Time)

데이터베이스가 해당 사실을 알고 있던(기록된) 기간입니다. 위 예시에서 이 정보를 2024-02-10에야 시스템에 입력했다면, 트랜잭션 시간은 2024-02-10부터 시작됩니다.

### 왜 두 차원이 모두 필요한가

단일 시간 차원으로는 대답하기 어려운 질문들이 있습니다.

- **"2024년 1월 1일 기준으로 조직도는 어떻게 됐나요?"** → 유효 시간 쿼리
- **"지난달 우리 데이터베이스가 2024년 1월 1일 조직도를 어떻게 알고 있었나요?"** → 트랜잭션 시간 + 유효 시간 쌍시제 쿼리
- **"입력 오류로 인해 잘못된 데이터가 있었을 때, 과거 특정 시점에 DB가 무엇을 알고 있었는지 재현해 주세요"** → 감사(audit) 목적의 쌍시제 쿼리

이런 요구사항은 금융 시스템의 거래 이력, 의료 기록, 보험 계약 이력 등에서 흔합니다.

## 왜 필요한가

### 단순 `updated_at` 방식의 한계

```sql
-- 흔히 볼 수 있는 단순한 이력 테이블
CREATE TABLE employee_history (
    id           SERIAL PRIMARY KEY,
    employee_id  BIGINT NOT NULL,
    department   TEXT NOT NULL,
    updated_at   TIMESTAMPTZ DEFAULT now()
);
```

이 방식은 "현재 최신 상태"는 알 수 있지만, 다음을 알 수 없습니다:
- 2024-01-01에 이 직원이 어느 부서에 있었는가?
- 그 사실을 DB가 처음 알게 된 시점은?
- 만약 입력 오류로 잘못된 데이터가 들어갔다가 수정됐다면, 오류 전 상태는?

### SQL:2011 표준

SQL:2011은 세 가지 시간 테이블 종류를 표준화했습니다.

| 종류 | 설명 |
|------|------|
| Application-time period tables | 유효 시간만 추적 |
| System-versioned tables | 트랜잭션 시간만 추적 (DB가 자동 관리) |
| System-versioned application-time tables | 두 차원 모두 추적 (쌍시제) |

Microsoft SQL Server 2016+, PostgreSQL 13+의 `SYSTEM_VERSIONING`, IBM DB2 등이 이를 지원합니다.

## 실제 구현 예제

### 예제 1: PostgreSQL에서 직접 구현한 쌍시제 테이블

```sql
-- 직원 소속 부서 이력을 쌍시제로 관리
CREATE TABLE employee_dept_history (
    employee_id   BIGINT NOT NULL,
    department_id BIGINT NOT NULL,
    
    -- 유효 시간: 현실 세계에서 사실인 기간
    valid_from    DATE NOT NULL,
    valid_to      DATE NOT NULL DEFAULT 'infinity',
    
    -- 트랜잭션 시간: DB에 기록된 기간 (시스템이 관리)
    sys_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
    sys_to        TIMESTAMPTZ NOT NULL DEFAULT 'infinity',
    
    role          TEXT,
    
    PRIMARY KEY (employee_id, valid_from, sys_from),
    CHECK (valid_from < valid_to),
    CHECK (sys_from < sys_to)
);

-- 현재 기준 현재 사실 조회 (가장 흔한 쿼리)
SELECT *
FROM employee_dept_history
WHERE employee_id = 42
  AND valid_from  <= CURRENT_DATE AND valid_to  > CURRENT_DATE
  AND sys_from    <= now()        AND sys_to    > now();

-- 2024-01-01 기준 현실 세계의 상태 (유효 시간 쿼리)
SELECT *
FROM employee_dept_history
WHERE employee_id = 42
  AND valid_from  <= '2024-01-01' AND valid_to  > '2024-01-01'
  AND sys_from    <= now()        AND sys_to    > now();

-- 어제 DB가 알던 현재 상태 (쌍시제 타임 트래블 쿼리)
SELECT *
FROM employee_dept_history
WHERE employee_id = 42
  AND valid_from  <= CURRENT_DATE              AND valid_to  > CURRENT_DATE
  AND sys_from    <= now() - INTERVAL '1 day'  AND sys_to   > now() - INTERVAL '1 day';

-- 편의를 위한 현재 뷰 생성
CREATE VIEW employee_dept_current AS
SELECT employee_id, department_id, valid_from, valid_to, role
FROM employee_dept_history
WHERE sys_from <= now() AND sys_to > now()
  AND valid_from <= CURRENT_DATE AND valid_to > CURRENT_DATE;
```

### 예제 2: 쌍시제 업데이트 패턴 (Python)

쌍시제 테이블에서 단순 `UPDATE`를 쓰면 과거 기록이 사라집니다. 항상 "기존 레코드 종료 → 새 레코드 삽입"의 append-only 패턴을 사용해야 합니다.

```python
from datetime import date, datetime, timezone
from contextlib import contextmanager
import psycopg2
from psycopg2.extras import RealDictCursor


def bitemporal_update(
    conn,
    employee_id: int,
    new_department_id: int,
    new_role: str,
    valid_from: date,
    valid_to: date | None = None,
) -> None:
    """
    쌍시제 업데이트.

    기존 활성 레코드의 sys_to를 현재 시각으로 닫고,
    새 레코드를 삽입합니다. 트랜잭션으로 묶어 원자성 보장.
    """
    now = datetime.now(timezone.utc)
    v_to = valid_to or date(9999, 12, 31)

    with conn.cursor() as cur:
        # 1단계: 겹치는 유효 기간의 기존 활성 레코드 sys_to 종료
        cur.execute(
            """
            UPDATE employee_dept_history
            SET    sys_to = %s
            WHERE  employee_id = %s
              AND  valid_from  < %s          -- 새 유효 기간과 겹치는 레코드
              AND  valid_to    > %s
              AND  sys_to      = 'infinity'  -- 현재 활성 레코드만
            """,
            (now, employee_id, v_to, valid_from),
        )

        # 2단계: 새 버전 삽입
        cur.execute(
            """
            INSERT INTO employee_dept_history
                (employee_id, department_id, valid_from, valid_to,
                 sys_from, sys_to, role)
            VALUES (%s, %s, %s, %s, %s, 'infinity', %s)
            """,
            (employee_id, new_department_id, valid_from, v_to, now, new_role),
        )

    conn.commit()


def bitemporal_correction(
    conn,
    employee_id: int,
    correct_department_id: int,
    correct_role: str,
    valid_from: date,
    valid_to: date | None = None,
) -> None:
    """
    과거 데이터 수정 (Correction).

    유효 시간을 소급하여 수정합니다.
    기존 DB 기록은 트랜잭션 시간으로 보존되므로
    "수정 전 DB가 뭘 알고 있었는지"는 여전히 조회 가능합니다.
    """
    # 수정도 동일한 패턴: 기존 레코드 종료 후 새 레코드 삽입
    bitemporal_update(conn, employee_id, correct_department_id,
                      correct_role, valid_from, valid_to)


# 사용 예시
if __name__ == "__main__":
    conn = psycopg2.connect("dbname=mydb")

    # Alice(id=42)가 2024-01-15부터 마케팅팀(id=5)에 소속
    bitemporal_update(
        conn,
        employee_id=42,
        new_department_id=5,
        new_role="Manager",
        valid_from=date(2024, 1, 15),
    )

    # 실제로는 2024-01-01부터였다고 소급 수정
    bitemporal_correction(
        conn,
        employee_id=42,
        correct_department_id=5,
        correct_role="Manager",
        valid_from=date(2024, 1, 1),  # 소급 수정
    )

    # 수정 전 DB가 알고 있던 내용 조회 (트랜잭션 시간으로 타임 트래블)
    with conn.cursor(cursor_factory=RealDictCursor) as cur:
        cur.execute(
            """
            SELECT * FROM employee_dept_history
            WHERE employee_id = 42
              AND sys_from <= %s AND sys_to > %s  -- 소급 수정 전 시점
            ORDER BY valid_from
            """,
            (datetime(2024, 2, 9, tzinfo=timezone.utc),) * 2,
        )
        rows = cur.fetchall()
        for row in rows:
            print(row)
```

## 주의사항/팁

### 성능 최적화

**인덱스 전략**: 쌍시제 쿼리는 항상 4개 컬럼을 범위 조건으로 사용합니다. 복합 인덱스 또는 PostgreSQL의 BRIN 인덱스가 효과적입니다.

```sql
-- 현재 활성 레코드를 빠르게 찾는 부분 인덱스
CREATE INDEX idx_emp_dept_active
    ON employee_dept_history (employee_id, valid_from, valid_to)
    WHERE sys_to = 'infinity';

-- 트랜잭션 시간 기반 타임 트래블을 위한 인덱스
CREATE INDEX idx_emp_dept_sys_time
    ON employee_dept_history (employee_id, sys_from, sys_to);
```

**파티셔닝**: 트랜잭션 시간 기준으로 범위 파티셔닝을 적용하면 오래된 히스토리 쿼리와 현재 쿼리의 I/O를 분리할 수 있습니다.

### 데이터 모델 설계 원칙

1. **무한대 표현**: `'infinity'` (PostgreSQL) 또는 `9999-12-31` 센티넬 값으로 열린 기간을 표현합니다. `infinity`를 지원하는 DB에서는 `infinity`가 더 명확합니다.
2. **트랜잭션 시간은 직접 수정 금지**: `sys_from`, `sys_to`는 애플리케이션이 직접 쓰면 안 됩니다. 가능하면 트리거나 DB 레벨에서 관리하세요.
3. **공백과 겹침 방지**: 유효 시간 구간에 공백(gap)이나 겹침(overlap)이 없어야 하는 경우 PostgreSQL의 `EXCLUDE` 제약이나 애플리케이션 레벨 검증이 필요합니다.
4. **쿼리 추상화**: 뷰(VIEW)나 저장 프로시저로 쌍시제 필터링 로직을 캡슐화하면 쿼리 오류를 줄일 수 있습니다.
5. **쌍시제가 필요 없는 곳에는 사용 금지**: 모든 테이블에 적용하면 복잡성만 증가합니다. 감사 추적, 소급 수정, 타임 트래블 쿼리가 명확히 필요한 테이블에만 사용하세요.

### 현대적 대안

- **SQL Server 2016+**: `SYSTEM_VERSIONING = ON`으로 트랜잭션 시간을 DB가 자동 관리
- **MariaDB 10.3+**: 시스템 버전 테이블 지원
- **Hibernate Envers / Debezium**: ORM 레벨 또는 CDC 기반 이력 추적

## 참고 자료

- [Wikipedia: Transaction time](https://en.wikipedia.org/wiki/Transaction_time)
- [TDWI: What is Temporal Data Modeling (2026)](https://tdwi.org/blogs/data-101/2026/05/what-is-temporal-data-modeling.aspx)
- [An Empirical Evaluation of Columnar Storage Formats - arXiv](https://arxiv.org/abs/2304.05028)
