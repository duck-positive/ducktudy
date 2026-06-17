---
layout: post
title: "데이터베이스 트랜잭션 격리 수준과 MVCC: 동시성 제어의 핵심 원리"
date: 2026-06-17
categories: [cs, computer-science]
tags: [database, transaction, ACID, MVCC, isolation-level, PostgreSQL, concurrency]
---

## 트랜잭션과 ACID

데이터베이스에서 **트랜잭션(Transaction)**은 하나의 논리적 작업 단위다. 은행 계좌 이체를 예로 들면 "A 계좌에서 100만 원 차감"과 "B 계좌에 100만 원 추가"는 반드시 둘 다 성공하거나 둘 다 실패해야 한다.

트랜잭션의 4가지 특성 **ACID**는 다음과 같다:

- **원자성(Atomicity)**: 트랜잭션의 연산들은 전부 실행되거나 전혀 실행되지 않는다
- **일관성(Consistency)**: 트랜잭션 완료 후 데이터베이스는 항상 일관된 상태를 유지한다
- **격리성(Isolation)**: 동시에 실행되는 트랜잭션들은 서로 간섭하지 않는다
- **지속성(Durability)**: 커밋된 트랜잭션은 장애가 발생해도 영구적으로 반영된다

이 중 **격리성(Isolation)**이 가장 구현하기 어렵고, 성능과의 트레이드오프가 가장 크다.

---

## 동시성으로 인한 이상 현상

여러 트랜잭션이 동시에 실행될 때 격리가 제대로 되지 않으면 다음 이상 현상이 발생한다.

### 1. 더티 읽기 (Dirty Read)

트랜잭션 A가 아직 커밋하지 않은 데이터를 트랜잭션 B가 읽는 상황이다. A가 이후 롤백하면 B는 존재하지 않는 데이터를 읽은 것이 된다.

```
T1: UPDATE accounts SET balance = 0 WHERE id = 1;  -- 아직 커밋 안 함
T2: SELECT balance FROM accounts WHERE id = 1;     -- 0을 읽음!
T1: ROLLBACK;  -- T2는 없는 데이터를 읽었음
```

### 2. 반복 불가능한 읽기 (Non-Repeatable Read)

같은 트랜잭션 내에서 같은 행을 두 번 읽었을 때 다른 값이 나오는 현상이다.

```
T1: SELECT balance FROM accounts WHERE id = 1;  -- 1000 읽음
T2: UPDATE accounts SET balance = 500 WHERE id = 1; COMMIT;
T1: SELECT balance FROM accounts WHERE id = 1;  -- 500 읽음! (다른 값)
```

### 3. 팬텀 읽기 (Phantom Read)

범위 조회 시 처음에 없던 행이 두 번째 조회에서 나타나는 현상이다.

```
T1: SELECT * FROM orders WHERE amount > 1000;  -- 3건 반환
T2: INSERT INTO orders VALUES (4, 1500); COMMIT;
T1: SELECT * FROM orders WHERE amount > 1000;  -- 4건 반환! (팬텀)
```

---

## 4가지 트랜잭션 격리 수준

SQL 표준(ISO/IEC 9075)은 4가지 격리 수준을 정의한다. 격리 수준이 높을수록 동시성은 낮아지고 데이터 일관성은 높아진다.

| 격리 수준 | Dirty Read | Non-Repeatable Read | Phantom Read |
|---------|-----------|---------------------|-------------|
| READ UNCOMMITTED | 발생 | 발생 | 발생 |
| READ COMMITTED | 방지 | 발생 | 발생 |
| REPEATABLE READ | 방지 | 방지 | 발생 |
| SERIALIZABLE | 방지 | 방지 | 방지 |

### READ UNCOMMITTED
커밋되지 않은 데이터도 읽을 수 있다. 사실상 쓸 일이 없는 격리 수준이다.

### READ COMMITTED (PostgreSQL 기본값)
다른 트랜잭션이 커밋한 데이터만 읽는다. 각 `SELECT` 문이 실행될 때마다 최신 커밋 스냅샷을 참조한다. Oracle과 PostgreSQL의 기본 격리 수준이다.

### REPEATABLE READ (MySQL InnoDB 기본값)
트랜잭션 시작 시점의 스냅샷을 고정한다. 같은 행을 여러 번 읽어도 동일한 값을 반환한다. MySQL InnoDB의 기본 격리 수준이며, PostgreSQL에서는 팬텀 읽기까지 방지된다.

### SERIALIZABLE
모든 트랜잭션을 순차 실행한 것처럼 처리한다. 완벽한 격리지만 성능 저하가 크다. PostgreSQL은 SSI(Serializable Snapshot Isolation)를 사용해 실제 충돌이 발생하지 않으면 Lock 없이 처리한다.

---

## MVCC: 읽기와 쓰기가 서로를 막지 않는 비밀

**MVCC(Multi-Version Concurrency Control)**는 데이터의 여러 버전을 동시에 유지함으로써 읽기가 쓰기를 막지 않고, 쓰기가 읽기를 막지 않게 만드는 핵심 기술이다.

전통적인 락 기반 방식과의 차이:
- **락 기반**: 읽기 시 공유 락, 쓰기 시 배타 락 → 읽기/쓰기가 서로 블로킹
- **MVCC**: 쓰기 시 새 버전 생성 → 기존 읽기는 이전 버전을 그대로 사용

### PostgreSQL의 MVCC 구현

PostgreSQL은 각 행(튜플)에 두 개의 숨겨진 시스템 컬럼을 둔다:

```sql
-- 숨겨진 시스템 컬럼 확인
SELECT xmin, xmax, id, balance 
FROM accounts;

-- 결과 예시:
--  xmin | xmax | id | balance
-- ------+------+----+---------
--   100 |    0 |  1 |    1000   ← 트랜잭션 100이 생성, 아직 삭제 안됨
--   102 |    0 |  1 |     500   ← 트랜잭션 102가 업데이트 → 새 버전 생성
```

- `xmin`: 이 버전을 생성한 트랜잭션 ID
- `xmax`: 이 버전을 삭제(무효화)한 트랜잭션 ID (0이면 현재 유효)

UPDATE는 기존 행을 수정하는 것이 아니라, **기존 행의 xmax를 설정하고 새 행을 INSERT**한다. DELETE도 물리적으로 삭제하지 않고 xmax를 설정한다.

### 스냅샷과 가시성 판단

트랜잭션 시작 시 현재 활성 트랜잭션 목록을 담은 **스냅샷**을 생성한다. 행의 가시성은 다음 규칙으로 판단한다:

```
행이 보이는 조건:
  - xmin이 커밋됐고, 스냅샷 시점보다 이전에 시작됐음
  - xmax가 0이거나, 아직 커밋되지 않았거나, 스냅샷 시점보다 이후에 시작됐음
```

---

## 코드로 이해하는 격리 수준과 MVCC

### 격리 수준별 동작 시연 (PostgreSQL SQL)

```sql
-- ===== 테스트 환경 설정 =====
CREATE TABLE accounts (
    id      INT PRIMARY KEY,
    name    VARCHAR(50),
    balance DECIMAL(10,2)
);
INSERT INTO accounts VALUES (1, 'Alice', 1000.00);
INSERT INTO accounts VALUES (2, 'Bob',   2000.00);

-- ===== READ COMMITTED에서 Non-Repeatable Read 시연 =====
-- Session 1
BEGIN;
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
SELECT balance FROM accounts WHERE id = 1;  -- 1000.00

    -- Session 2 (동시 실행)
    BEGIN;
    UPDATE accounts SET balance = 500.00 WHERE id = 1;
    COMMIT;

SELECT balance FROM accounts WHERE id = 1;  -- 500.00 (다른 값! 비반복 읽기)
COMMIT;

-- ===== REPEATABLE READ에서 일관성 보장 =====
-- Session 1
BEGIN;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT balance FROM accounts WHERE id = 1;  -- 1000.00

    -- Session 2 (동시 실행)
    BEGIN;
    UPDATE accounts SET balance = 999.00 WHERE id = 1;
    COMMIT;

SELECT balance FROM accounts WHERE id = 1;  -- 1000.00 (여전히! 스냅샷 유지)
COMMIT;

-- ===== Serializable에서 Write Skew 방지 =====
-- Write Skew: 두 트랜잭션이 서로 다른 행을 읽고 쓰지만 결과가 비일관적
-- 의사가 최소 1명은 당직이어야 함 (Alice, Bob 둘 다 동시에 퇴근 시도)

CREATE TABLE doctors (id INT, name VARCHAR(50), on_call BOOLEAN);
INSERT INTO doctors VALUES (1, 'Alice', TRUE), (2, 'Bob', TRUE);

-- Session 1 (Alice가 퇴근 시도)
BEGIN;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SELECT COUNT(*) FROM doctors WHERE on_call = TRUE;  -- 2명
UPDATE doctors SET on_call = FALSE WHERE id = 1;

    -- Session 2 (Bob이 퇴근 시도)
    BEGIN;
    SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
    SELECT COUNT(*) FROM doctors WHERE on_call = TRUE;  -- 2명 (아직 Alice 변경 안 보임)
    UPDATE doctors SET on_call = FALSE WHERE id = 2;
    COMMIT;  -- 성공

COMMIT;  -- ERROR: could not serialize access due to read/write dependencies
         -- Serializable이 충돌 감지!
```

### MVCC 버전 체인 시뮬레이션 (Python)

```python
from dataclasses import dataclass, field
from typing import Optional
import threading

@dataclass
class RowVersion:
    """행의 한 버전을 나타냄"""
    data: dict
    xmin: int           # 이 버전을 생성한 트랜잭션 ID
    xmax: Optional[int] # 이 버전을 무효화한 트랜잭션 ID
    next_version: Optional['RowVersion'] = field(default=None, repr=False)

class MVCCTable:
    def __init__(self):
        self.rows: dict[int, RowVersion] = {}
        self.committed: set[int] = set()
        self.next_xid = 1
        self.lock = threading.Lock()

    def begin_transaction(self) -> int:
        with self.lock:
            xid = self.next_xid
            self.next_xid += 1
            # 스냅샷: 현재 커밋된 트랜잭션 집합 복사
            snapshot = frozenset(self.committed)
        return xid

    def commit(self, xid: int):
        with self.lock:
            self.committed.add(xid)
        print(f"  [XID {xid}] 커밋 완료")

    def insert(self, key: int, data: dict, xid: int):
        version = RowVersion(data=data, xmin=xid, xmax=None)
        self.rows[key] = version
        print(f"  [XID {xid}] INSERT key={key}, data={data}")

    def update(self, key: int, new_data: dict, xid: int):
        old = self.rows.get(key)
        if old is None:
            raise KeyError(f"key {key} not found")
        # 기존 버전의 xmax 설정 (논리적 삭제)
        old.xmax = xid
        # 새 버전 생성
        new_version = RowVersion(data=new_data, xmin=xid, xmax=None)
        old.next_version = new_version
        self.rows[key] = new_version
        print(f"  [XID {xid}] UPDATE key={key}: {old.data} → {new_data}")

    def read(self, key: int, snapshot: frozenset, xid: int):
        """스냅샷 기준으로 가시적인 버전 반환"""
        version = self.rows.get(key)
        # 실제 PostgreSQL은 버전 체인을 역방향으로 탐색하지만 여기선 단순화
        if version is None:
            return None
        # xmin이 커밋됐고, xmax가 없거나 아직 커밋 안 됐으면 가시적
        xmin_visible = version.xmin in snapshot or version.xmin == xid
        xmax_invisible = version.xmax is None or version.xmax not in snapshot
        if xmin_visible and xmax_invisible:
            return version.data
        return None

# MVCC 동작 시연
table = MVCCTable()

# T1: 데이터 삽입
xid1 = table.begin_transaction()
table.insert(1, {"name": "Alice", "balance": 1000}, xid1)
table.commit(xid1)

snapshot1_start = frozenset(table.committed)

# T2: 동시에 시작 (T3 커밋 전 스냅샷)
xid2 = table.begin_transaction()
print(f"\nT2 시작 (스냅샷: {snapshot1_start})")
print(f"T2 읽기: {table.read(1, snapshot1_start, xid2)}")  # 1000

# T3: 업데이트 후 커밋
xid3 = table.begin_transaction()
table.update(1, {"name": "Alice", "balance": 500}, xid3)
table.commit(xid3)

# T2는 여전히 이전 스냅샷으로 읽음 → REPEATABLE READ 효과
print(f"T2 재읽기 (동일 스냅샷): {table.read(1, snapshot1_start, xid2)}")  # 1000!

# 새 트랜잭션 T4는 T3 커밋 후 스냅샷으로 읽음
snapshot_after = frozenset(table.committed)
xid4 = table.begin_transaction()
print(f"\nT4 읽기 (T3 커밋 후): {table.read(1, snapshot_after, xid4)}")  # 500
```

---

## 주의사항과 실전 팁

### 격리 수준 선택 기준

| 상황 | 권장 격리 수준 |
|-----|--------------|
| 통계/분석 쿼리 (약간의 불일치 허용) | READ COMMITTED |
| 일반 OLTP (잔액 조회, 목록 조회) | READ COMMITTED |
| 금융 처리, 재고 관리 | REPEATABLE READ |
| 회계 마감, 복잡한 일관성 요구 | SERIALIZABLE |

### VACUUM과 Dead Tuple 문제
PostgreSQL MVCC는 업데이트/삭제 시 기존 버전을 물리적으로 남긴다. 시간이 지나면 **dead tuple**이 쌓여 테이블이 부풀어 오른다. `AUTOVACUUM`이 주기적으로 정리하지만, 대량 DML 작업 후에는 수동 `VACUUM`을 실행하는 것이 좋다.

```sql
-- 테이블의 dead tuple 수 확인
SELECT relname, n_dead_tup, n_live_tup, last_autovacuum
FROM pg_stat_user_tables
WHERE relname = 'accounts';

-- 수동 VACUUM
VACUUM ANALYZE accounts;
```

### 트랜잭션 ID 고갈 (XID Wraparound)
PostgreSQL의 트랜잭션 ID는 32비트 정수다. 약 21억 개의 트랜잭션이 실행되면 XID가 순환된다. `FREEZE` 작업으로 오래된 튜플의 xmin을 동결해야 하며, `VACUUM FREEZE`를 주기적으로 실행하거나 `autovacuum_freeze_max_age`를 적절히 설정해야 한다.

### 읽기 전용 트랜잭션 최적화

```sql
-- 읽기 전용 트랜잭션으로 명시 (최적화 힌트 제공)
BEGIN READ ONLY;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
-- 대용량 분석 쿼리
SELECT SUM(amount) FROM transactions WHERE created_at > NOW() - INTERVAL '1 month';
COMMIT;
```

### 명시적 Lock이 필요한 경우
MVCC만으로 해결할 수 없는 경우(재고 차감, 포인트 사용 등)에는 `SELECT ... FOR UPDATE`로 행 레벨 잠금을 사용한다.

```sql
BEGIN;
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;  -- 배타 락 획득
-- 이 시점부터 다른 트랜잭션은 이 행 수정 불가
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;
```

---

## 참고 자료
- [PostgreSQL Documentation — Concurrency Control (MVCC)](https://www.postgresql.org/docs/current/mvcc.html)
- [PostgreSQL Documentation — Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
- [MVCC in PostgreSQL — Postgres Professional Blog](https://postgrespro.com/blog/pgsql/5967856)
- [PostgreSQL Transaction Isolation Levels Guide — Mydbops](https://www.mydbops.com/blog/postgresql-transaction-isolation-levels-guide)
