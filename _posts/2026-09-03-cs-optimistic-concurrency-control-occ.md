---
layout: post
title: "낙관적 동시성 제어(OCC) 완전 정복: 충돌이 드문 환경을 위한 락 없는 트랜잭션 처리"
date: 2026-09-03
categories: [cs, computer-science]
tags: [database, concurrency-control, OCC, transaction, pessimistic-locking, MVCC, serialization, postgresql]
---

데이터베이스 동시성 제어의 두 가지 큰 패러다임은 **비관적(Pessimistic)**과 **낙관적(Optimistic)**이다. 비관적 동시성 제어(2PL)는 "충돌이 일어날 것"을 가정해 미리 락을 걸지만, **낙관적 동시성 제어(Optimistic Concurrency Control, OCC)**는 "충돌이 드물 것"을 가정해 트랜잭션을 락 없이 실행하고, 커밋 시점에만 충돌을 검사한다.

OCC는 1981년 H.T. Kung과 John T. Robinson의 논문 "On Optimistic Methods for Concurrency Control"에서 처음 체계화되었으며, 오늘날 PostgreSQL의 SSI(Serializable Snapshot Isolation), TiDB의 낙관적 트랜잭션 모드, CockroachDB, FoundationDB 등 다양한 분산 데이터베이스에서 핵심 메커니즘으로 활용된다.

## 왜 낙관적 동시성 제어가 필요한가?

비관적 동시성 제어(2PL, Two-Phase Locking)는 직렬성(serializability)을 보장하지만 대가가 있다:

**락 경합(Lock Contention)**: 트래픽이 몰리면 대기 시간이 길어진다. 수천 개의 트랜잭션이 같은 레코드의 락을 기다리면 처리량이 급감한다.

**데드락(Deadlock)**: T1이 A를 잠그고 B를 기다리는 동안, T2가 B를 잠그고 A를 기다리면 교착 상태가 발생한다. 데드락 탐지와 희생자 선택의 오버헤드도 만만치 않다.

**읽기 성능 저하**: 읽기 전용 트랜잭션도 공유 락을 획득해야 한다. 읽기가 쓰기를 차단하거나 반대 상황이 발생한다.

```
2PL 문제 시나리오:
T1: LOCK(row A) → LOCK(row B) → 처리 → UNLOCK
T2:              LOCK(row B) → 대기... (A 락 기다리며)
T3:                           LOCK(row A) → 대기... (B 락 기다리며)
→ T2와 T3 데드락!
```

읽기 비율이 높고 실제 충돌이 드문 시스템 — e-commerce 상품 조회, 소셜 미디어 피드, 분석 쿼리 — 에서 OCC는 극적인 성능 향상을 제공한다.

## OCC의 세 단계

모든 OCC 구현의 핵심은 세 단계 프로토콜이다:

```
타임스탬프: T_begin          T_validate        T_commit
            │                │                 │
T1: [──────── 읽기 단계 ────────][검증][──쓰기──]
T2:     [──────── 읽기 단계 ───────────][검증][쓰기]
T3:           [────── 읽기 단계 ──────────][검증 실패]→ 재시도
```

### 1단계: 읽기 단계 (Read Phase)

- 트랜잭션이 시작되며 **시작 타임스탬프**(T_begin)를 부여받는다.
- 필요한 데이터를 읽고, 읽은 레코드의 버전/타임스탬프를 **읽기 집합(Read Set)**에 기록한다.
- 쓸 데이터는 로컬 버퍼(스냅샷)에만 기록한다 — **쓰기 집합(Write Set)**.
- 이 단계에서 다른 트랜잭션을 차단하거나 충돌을 검사하지 않는다.

### 2단계: 검증 단계 (Validation Phase)

- 커밋 시점에 **검증 타임스탬프**(T_validate)를 원자적으로 부여.
- 읽기 단계 이후(T_begin ~ T_validate) 다른 트랜잭션이 읽기 집합을 수정했는지 검사.
- **충돌 없음** → 3단계 진행.
- **충돌 감지** → 현재 트랜잭션 중단(abort), 재시도.

### 3단계: 쓰기 단계 (Write Phase)

- 검증을 통과한 트랜잭션의 로컬 버퍼를 실제 DB에 원자적으로 반영.
- **커밋 타임스탬프**(T_commit) 부여.

## 코드 예제 1: Python으로 구현하는 OCC 인메모리 DB

```python
import threading
from typing import Dict, Any, Optional
from dataclasses import dataclass, field

@dataclass
class VersionedRecord:
    value: Any
    commit_ts: int  # 이 버전이 커밋된 타임스탬프

class OccDatabase:
    """낙관적 동시성 제어 인메모리 데이터베이스"""
    
    def __init__(self):
        self._store: Dict[str, VersionedRecord] = {}
        self._ts_lock = threading.Lock()
        self._commit_lock = threading.Lock()  # 검증+쓰기 직렬화
        self._ts = 0
    
    def _next_ts(self) -> int:
        with self._ts_lock:
            self._ts += 1
            return self._ts
    
    def begin(self) -> 'OccTransaction':
        return OccTransaction(self, self._next_ts())
    
    def _read_at(self, key: str, ts: int) -> Optional[VersionedRecord]:
        """ts 시점에 커밋된 레코드 반환"""
        with self._ts_lock:
            rec = self._store.get(key)
            if rec and rec.commit_ts <= ts:
                return rec
        return None
    
    def _validate_and_write(self, txn: 'OccTransaction') -> bool:
        """검증과 쓰기를 원자적으로 수행 (직렬화 임계 구역)"""
        with self._commit_lock:
            commit_ts = self._next_ts()
            
            # ── 검증 단계 ──
            # 읽기 집합의 각 키가 begin_ts 이후 변경되었는지 검사
            for key, read_rec in txn.read_set.items():
                current = self._store.get(key)
                
                if current is None and read_rec is not None:
                    return False  # 다른 트랜잭션이 삭제함
                if current is not None:
                    if read_rec is None:
                        return False  # 우리가 없다고 읽었는데 생겼음
                    if current.commit_ts > txn.begin_ts:
                        return False  # begin_ts 이후 다른 트랜잭션이 수정함
            
            # ── 쓰기 단계 ──
            for key, value in txn.write_set.items():
                self._store[key] = VersionedRecord(
                    value=value,
                    commit_ts=commit_ts
                )
            
            return True

@dataclass
class OccTransaction:
    db: OccDatabase
    begin_ts: int
    read_set: Dict[str, Optional[VersionedRecord]] = field(default_factory=dict)
    write_set: Dict[str, Any] = field(default_factory=dict)
    _aborted: bool = False
    
    def read(self, key: str) -> Optional[Any]:
        if self._aborted:
            raise RuntimeError("중단된 트랜잭션")
        # 쓰기 집합 우선 (읽기-쓰기 일관성)
        if key in self.write_set:
            return self.write_set[key]
        # DB에서 begin_ts 기준 읽기
        rec = self.db._read_at(key, self.begin_ts)
        self.read_set[key] = rec  # 읽기 집합에 버전 기록
        return rec.value if rec else None
    
    def write(self, key: str, value: Any):
        if self._aborted:
            raise RuntimeError("중단된 트랜잭션")
        self.write_set[key] = value
    
    def commit(self) -> bool:
        if self._aborted:
            return False
        return self.db._validate_and_write(self)
    
    def abort(self):
        self._aborted = True
        self.read_set.clear()
        self.write_set.clear()

# ── 사용 예제: 계좌 이체 ──
import random, time

db = OccDatabase()

# 초기 데이터 설정
setup = db.begin()
setup.write("alice", 1000)
setup.write("bob", 500)
setup.commit()

def transfer_with_retry(from_acc: str, to_acc: str, amount: int,
                        max_retries: int = 5) -> bool:
    for attempt in range(max_retries):
        txn = db.begin()
        
        from_bal = txn.read(from_acc)
        to_bal   = txn.read(to_acc)
        
        if from_bal is None or from_bal < amount:
            txn.abort()
            return False
        
        txn.write(from_acc, from_bal - amount)
        txn.write(to_acc, to_bal + amount)
        
        if txn.commit():
            print(f"이체 성공 ({attempt + 1}번째 시도): "
                  f"{from_acc} {from_bal}→{from_bal - amount}, "
                  f"{to_acc} {to_bal}→{to_bal + amount}")
            return True
        
        # 충돌 발생 → 지수 백오프 후 재시도
        wait = (2 ** attempt) * 0.01 + random.uniform(0, 0.01)
        print(f"충돌, {wait:.3f}초 후 재시도... ({attempt + 1}/{max_retries})")
        time.sleep(wait)
    
    return False

# 병렬 이체 테스트
threads = [
    threading.Thread(target=transfer_with_retry,
                     args=("alice", "bob", 100)),
    threading.Thread(target=transfer_with_retry,
                     args=("bob", "alice", 50)),
]
for t in threads: t.start()
for t in threads: t.join()

# 결과 확인
check = db.begin()
print(f"최종 잔액 — alice: {check.read('alice')}, bob: {check.read('bob')}")
```

## 검증 전략: 후방 검증 vs 전방 검증

OCC 검증 알고리즘은 두 가지 방향으로 설계할 수 있다:

```python
# ── 후방 검증 (Backward Validation) ──
# 현재 트랜잭션의 읽기 집합 vs 최근 커밋된 트랜잭션들의 쓰기 집합
# "내가 읽은 것을 남이 이미 수정했나?"
def backward_validate(current_txn, recently_committed):
    for committed in recently_committed:
        # committed.write_keys ∩ current_txn.read_keys ≠ ∅ → 충돌
        overlap = set(committed.write_set) & set(current_txn.read_set)
        if overlap:
            return False, f"충돌 키: {overlap}"
    return True, "검증 통과"

# ── 전방 검증 (Forward Validation) ──
# 현재 트랜잭션의 쓰기 집합 vs 동시 실행 중인 트랜잭션들의 읽기 집합
# "내가 쓸 것을 남이 이미 읽고 있나?"
def forward_validate(current_txn, active_txns):
    for active in active_txns:
        if active.begin_ts > current_txn.begin_ts:
            # 나보다 늦게 시작한 트랜잭션이 내 쓰기 집합을 읽었으면 충돌 가능
            overlap = set(active.read_set) & set(current_txn.write_set)
            if overlap:
                return False, f"전방 충돌 키: {overlap}"
    return True, "검증 통과"

# 후방 검증: 커밋 시점에 과거를 확인 — 구현 단순
# 전방 검증: 커밋 시점에 미래(동시 트랜잭션)를 확인 — 더 일찍 충돌 탐지 가능
```

## 코드 예제 2: PostgreSQL SSI (직렬성 스냅샷 격리)

PostgreSQL 9.1부터 도입된 SSI는 MVCC 위에 OCC 스타일의 위험 감지를 추가해 완전한 직렬성을 보장하는 가장 정교한 구현 중 하나다.

```sql
-- ── 예제: Write Skew 이상 현상과 SSI 해결 ──

-- 의사 코드 시나리오: 두 의사 중 적어도 한 명은 항상 당직이어야 한다
-- T1: Alice가 퇴근 (Bob이 당직 중인 줄 알고)
-- T2: Bob이 퇴근 (Alice가 당직 중인 줄 알고)
-- 결과: 아무도 당직이 아닌 Write Skew 발생!

-- 2PL이나 기본 스냅샷 격리에서는 이 이상 현상을 막지 못한다.
-- SSI는 rw-anti-dependency 사이클을 탐지하여 차단한다.

-- 직렬화 격리 수준 설정
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- T1 트랜잭션 (Alice가 실행)
SELECT COUNT(*) FROM doctors WHERE on_call = true;  -- 2명 읽음 (읽기 집합에 기록)
UPDATE doctors SET on_call = false WHERE name = 'Alice';  -- Alice 퇴근
COMMIT;  
-- SSI가 T1의 읽기와 T2의 쓰기 사이의 rw-anti-dependency를 탐지
-- 직렬화 사이클 발견 → 한 트랜잭션 abort

-- ── PostgreSQL의 SSI 내부 동작 ──
-- SIREAD 락: 특정 행의 "읽기 종속성"을 추적
-- rw-anti-dependency 그래프:
--   T1 →(읽기)→ doctors
--   T2 →(쓰기)→ doctors (T1이 읽은 후)
--   T2 →(읽기)→ doctors  
--   T1 →(쓰기)→ doctors (T2가 읽은 후)
-- → T1 → T2 → T1 사이클 발견 → 충돌!

-- 오류 메시지:
-- ERROR:  could not serialize access due to read/write dependencies
--         among transactions
-- DETAIL: Reason code: Canceled on identification as a pivot,
--         during commit attempt.
-- HINT:   The transaction might succeed if retried.

-- ── 실용적인 재시도 패턴 ──
-- Python psycopg2 예제
import psycopg2
from psycopg2 import errors

def serializable_transfer(conn, from_id: int, to_id: int, amount: float):
    max_retries = 5
    for attempt in range(max_retries):
        try:
            with conn.cursor() as cur:
                conn.set_isolation_level(
                    psycopg2.extensions.ISOLATION_LEVEL_SERIALIZABLE
                )
                cur.execute(
                    "SELECT balance FROM accounts WHERE id = %s FOR NO KEY UPDATE",
                    (from_id,)
                )
                from_bal = cur.fetchone()[0]
                
                if from_bal < amount:
                    conn.rollback()
                    return False, "잔액 부족"
                
                cur.execute(
                    "UPDATE accounts SET balance = balance - %s WHERE id = %s",
                    (amount, from_id)
                )
                cur.execute(
                    "UPDATE accounts SET balance = balance + %s WHERE id = %s",
                    (amount, to_id)
                )
                conn.commit()
                return True, f"성공 (시도 {attempt + 1}회)"
        
        except errors.SerializationFailure:
            conn.rollback()
            if attempt < max_retries - 1:
                time.sleep(0.1 * (2 ** attempt))
                continue
        except Exception as e:
            conn.rollback()
            raise
    
    return False, "최대 재시도 초과"
```

## OCC vs 2PL vs MVCC 비교

| 특성 | OCC | 2PL (비관적) | MVCC |
|------|-----|-------------|------|
| 충돌 가정 | 낙관적 (드물다) | 비관적 (잦다) | 중립 |
| 읽기 성능 | 매우 높음 | 락 경합에 따라 저하 | 높음 |
| 쓰기 충돌 처리 | Abort & Retry | 대기 (Blocking) | 버전 분기 |
| 데드락 가능성 | 없음 | 있음 | 없음 |
| 오버헤드 종류 | 검증 비용, Abort 비용 | 락 관리, 데드락 탐지 | 버전 저장 공간 |
| 최적 워크로드 | 읽기 중심, 충돌 낮음 | 쓰기 중심, 충돌 높음 | 범용 |
| 데이터베이스 예시 | TiDB 낙관적 모드, FoundationDB | MySQL InnoDB 기본 | PostgreSQL, Oracle |

## 분산 환경에서의 OCC: Percolator와 TiDB

Google의 Percolator(2010)는 분산 스토리지(Bigtable)에서 OCC를 구현한 획기적인 시스템이다. 2단계 커밋(2PC)과 낙관적 잠금을 결합해 수십억 개의 웹 페이지 인덱싱에 사용된다.

```
Percolator 커밋 프로토콜:
1. 읽기 단계: 시작 타임스탬프(start_ts)로 스냅샷 읽기
2. 예비 커밋(Prewrite):
   - 쓰기 집합의 각 키에 잠금(lock) 기록
   - 기본 키(primary key)를 앵커로 사용
3. 커밋:
   - 기본 키의 잠금을 커밋 타임스탬프(commit_ts)로 교체
   - 나머지 키들의 잠금 해제
4. 충돌: 다른 트랜잭션의 잠금 발견 시 abort

TiDB 낙관적 트랜잭션 설정:
SET tidb_disable_txn_auto_retry = OFF;
BEGIN OPTIMISTIC;  -- 낙관적 트랜잭션 시작
-- ... 읽기/쓰기 ...
COMMIT;  -- 검증 + 쓰기
```

## 주의사항과 팁

### 1. 충돌률이 높으면 OCC가 역효과

OCC는 충돌 시 트랜잭션 전체를 재시작해야 한다. 읽기 단계 동안 수행한 모든 작업이 낭비된다.

```
충돌률 < 5%: OCC 유리 (낮은 락 오버헤드 > Abort 비용)
충돌률 5~20%: 상황에 따라 다름 (트랜잭션 길이 고려)
충돌률 > 20%: 2PL 유리 (대기가 Abort+Retry보다 효율적)

핫 키(Hot Key) 문제: 특정 키에 쓰기가 집중되면
OCC Abort율이 급증 → 카운터, 인기 상품 재고 등에 주의
```

### 2. 읽기 집합을 최소화하라

읽기 집합이 클수록 충돌 가능성이 높아진다. 필요한 컬럼만 SELECT하고, 인덱스를 활용해 스캔 범위를 줄여라.

```sql
-- 나쁜 예: 전체 테이블 읽기 → 읽기 집합이 거대해짐
SELECT * FROM products;

-- 좋은 예: 필요한 것만 읽기 → 읽기 집합 최소화
SELECT id, price FROM products WHERE category_id = 5;
```

### 3. 재시도 로직에 지터(Jitter) 추가

여러 트랜잭션이 동시에 충돌하면 동시에 재시도해서 또 충돌한다. 지수 백오프에 무작위 지터를 추가해 재시도를 분산시켜라.

```python
import random, time

def retry_with_backoff(operation, max_retries=5, base_delay=0.1):
    for attempt in range(max_retries):
        try:
            return operation()
        except SerializationError:
            if attempt == max_retries - 1:
                raise
            # 지수 백오프 + 무작위 지터
            delay = base_delay * (2 ** attempt) + random.uniform(0, base_delay)
            time.sleep(delay)
```

### 4. 트랜잭션 길이를 짧게 유지하라

읽기 단계가 길수록 그 사이 다른 트랜잭션이 같은 데이터를 수정할 확률이 높아진다. OCC 환경에서는 트랜잭션을 최대한 짧게 설계하는 것이 핵심이다.

## 결론

낙관적 동시성 제어는 락 기반 방식의 경합 오버헤드를 제거하여, 충돌이 드문 워크로드에서 극적인 처리량 향상을 제공한다. 전통적인 OLTP에서는 2PL과 MVCC의 조합이 일반적이지만, 마이크로서비스 간 분산 트랜잭션이나 읽기 중심 API 레이어에서 OCC는 확실한 성능 이점을 제공한다. 워크로드의 충돌 패턴을 실측하고(충돌률, 핫 키 분포), 그에 맞는 동시성 제어 전략을 선택하는 것이 성능 최적화의 핵심이다.

## 참고 자료
- [Kung & Robinson (1981) — On Optimistic Methods for Concurrency Control (ACM)](https://dl.acm.org/doi/10.1145/319566.319567)
- [PostgreSQL 공식 문서 — 트랜잭션 격리 (Transaction Isolation)](https://www.postgresql.org/docs/current/transaction-iso.html)
- [Google Percolator 논문 — Large-scale Incremental Processing](https://research.google/pubs/large-scale-incremental-processing-using-distributed-transactions-and-notifications/)
- [TiDB 낙관적 트랜잭션 공식 문서](https://docs.pingcap.com/tidb/stable/optimistic-transaction)
