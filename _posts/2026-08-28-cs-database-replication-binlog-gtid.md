---
layout: post
title: "데이터베이스 복제(Replication) 완전 정복: Binlog·GTID·리더-팔로워와 복제 지연 해결"
date: 2026-08-28
categories: [cs, computer-science]
tags: [database, replication, binlog, gtid, leader-follower, mysql, postgresql, replication-lag, high-availability, distributed-systems]
---

단일 데이터베이스 서버로는 읽기 부하, 고가용성(HA), 지역 분산의 세 가지 요구를 동시에 만족하기 어렵다. **데이터베이스 복제(Replication)**는 하나의 데이터베이스(리더/Primary)의 변경사항을 하나 이상의 복제본(팔로워/Replica)에 실시간으로 전파하여 이 문제를 해결한다. MySQL의 Binlog 기반 복제부터 GTID, 반동기 복제, 그리고 PostgreSQL의 WAL 스트리밍까지 복제의 내부 동작을 완전히 이해하면 데이터베이스 장애에 자신 있게 대응할 수 있다.

## 개념 설명: 복제가 필요한 이유

### 복제가 해결하는 문제

**읽기 확장(Read Scaling)**: 실제 서비스에서 읽기(SELECT)는 쓰기(INSERT/UPDATE/DELETE)보다 압도적으로 많다. 팔로워에게 읽기 부하를 분산하면 리더의 CPU와 I/O를 쓰기에 집중할 수 있다.

**고가용성(High Availability)**: 리더가 장애를 일으키면 팔로워를 자동으로 리더로 승격(Failover)하여 서비스 중단 시간을 최소화한다.

**지역 분산(Geo-Distribution)**: 사용자에게 가까운 지역의 팔로워에서 읽기를 처리하면 지연 시간(latency)을 줄인다.

**분석 쿼리 격리**: 무거운 분석 쿼리(OLAP)를 팔로워에서 실행하여 리더의 OLTP 성능에 영향을 주지 않는다.

### 동기 복제 vs. 비동기 복제

| 구분 | 동기 복제 | 비동기 복제 |
|------|-----------|------------|
| 데이터 손실 | 없음 | 가능 (복제 지연 중 리더 장애 시) |
| 쓰기 지연 | 증가 (팔로워 응답 대기) | 최소 |
| 가용성 | 팔로워 장애 시 쓰기도 중단 | 팔로워 장애와 무관 |
| 사용 사례 | 금융, 결제 | 일반 웹 서비스 |

MySQL의 기본값은 **비동기 복제**다. **반동기 복제(Semi-Synchronous Replication)**는 그 중간으로, 리더가 최소 1개의 팔로워가 릴레이 로그를 수신했다는 ACK를 받은 후 클라이언트에게 성공을 응답한다.

## MySQL Binlog 복제의 내부 동작

### Binlog란?

**Binary Log(Binlog)**는 MySQL 리더에서 발생한 모든 데이터 변경 이벤트를 순서대로 기록한 로그 파일이다. Binlog는 크래시 복구가 목적인 InnoDB의 Redo Log와 다르다—Binlog는 복제와 Point-in-Time Recovery에 사용된다.

Binlog 포맷 3가지:
- **STATEMENT**: SQL 문장 그대로 기록. 파일 크기 작음. `NOW()`, `UUID()` 같은 비결정적 함수에서 복제 불일치 발생.
- **ROW**: 실제 변경된 행 데이터를 기록. 가장 안전하고 권장됨. 파일 크기 큼.
- **MIXED**: 기본적으로 STATEMENT, 비결정적 상황에서는 ROW 사용.

### 복제 스레드

팔로워는 두 개의 스레드로 복제를 처리한다.

1. **I/O Thread**: 리더의 Binlog Dump 스레드에 연결하여 Binlog 이벤트를 수신하고 로컬의 **Relay Log**에 기록한다.
2. **SQL Thread**: Relay Log에서 이벤트를 읽어 팔로워 데이터베이스에 적용한다.

이 2단계 파이프라인으로 인해 복제 지연(Replication Lag)이 발생한다.

### GTID(Global Transaction Identifier) 복제

전통적인 Binlog 복제는 파일명과 오프셋(`(binlog.000003, 1234)`)으로 위치를 추적한다. 이 방식은 Failover 시 새로운 리더에서 어떤 파일, 어떤 오프셋부터 복제해야 하는지 수동으로 계산해야 한다.

**GTID(Global Transaction Identifier)**는 각 트랜잭션에 전역적으로 고유한 ID를 부여한다.

```
GTID 형식: {source_id}:{transaction_id}
예시: 3E11FA47-71CA-11E1-9E33-C80AA9429562:23
```

- `source_id`: 리더의 `server_uuid` (MySQL 서버 고유 UUID)
- `transaction_id`: 해당 서버에서 커밋된 순서 번호

GTID의 장점:
- 팔로워는 "어느 GTID까지 적용했는가"만 알면 됨. Failover 시 자동으로 올바른 위치부터 복제 재개.
- `EXECUTED_GTID_SET` 집합 연산으로 누락된 트랜잭션 자동 탐지.

## 실제 구현 예제

### 예제 1: Binlog 파싱 및 복제 시뮬레이터 (Python)

```python
import time
import threading
import queue
from dataclasses import dataclass, field
from enum import Enum
from typing import Optional
import uuid as uuid_mod

class BinlogEventType(Enum):
    WRITE_ROWS = "WRITE_ROWS"    # INSERT
    UPDATE_ROWS = "UPDATE_ROWS"  # UPDATE
    DELETE_ROWS = "DELETE_ROWS"  # DELETE
    QUERY = "QUERY"              # DDL

@dataclass
class GTIDEvent:
    source_uuid: str
    transaction_id: int

    @property
    def gtid(self) -> str:
        return f"{self.source_uuid}:{self.transaction_id}"

@dataclass
class BinlogEvent:
    gtid: Optional[GTIDEvent]
    event_type: BinlogEventType
    table: str
    data: dict
    timestamp: float = field(default_factory=time.time)

    def __str__(self):
        gtid_str = self.gtid.gtid if self.gtid else "N/A"
        return (f"[{self.event_type.value}] table={self.table} "
                f"data={self.data} gtid={gtid_str}")

class MySQLLeader:
    """리더 데이터베이스: 트랜잭션을 처리하고 Binlog에 기록"""

    def __init__(self, server_uuid: Optional[str] = None):
        self.server_uuid = server_uuid or str(uuid_mod.uuid4())
        self.tables: dict[str, dict] = {}
        self.binlog: list[BinlogEvent] = []
        self.gtid_counter = 0
        self._lock = threading.Lock()

        # 팔로워들의 Relay Log 큐 (브로드캐스트 시뮬레이션)
        self.follower_queues: list[queue.Queue] = []

        print(f"  Leader started. server_uuid={self.server_uuid[:8]}...")

    def _next_gtid(self) -> GTIDEvent:
        self.gtid_counter += 1
        return GTIDEvent(self.server_uuid, self.gtid_counter)

    def _broadcast(self, event: BinlogEvent):
        for q in self.follower_queues:
            q.put(event)

    def insert(self, table: str, row_id: int, data: dict):
        with self._lock:
            if table not in self.tables:
                self.tables[table] = {}
            self.tables[table][row_id] = data
            event = BinlogEvent(
                gtid=self._next_gtid(),
                event_type=BinlogEventType.WRITE_ROWS,
                table=table,
                data={"id": row_id, **data}
            )
            self.binlog.append(event)
            self._broadcast(event)
            print(f"  [LEADER] INSERT {table} id={row_id} "
                  f"→ {event.gtid.gtid}")

    def update(self, table: str, row_id: int, data: dict):
        with self._lock:
            if table not in self.tables or row_id not in self.tables[table]:
                print(f"  [LEADER] UPDATE failed: {table} id={row_id} not found")
                return
            self.tables[table][row_id].update(data)
            event = BinlogEvent(
                gtid=self._next_gtid(),
                event_type=BinlogEventType.UPDATE_ROWS,
                table=table,
                data={"id": row_id, **data}
            )
            self.binlog.append(event)
            self._broadcast(event)
            print(f"  [LEADER] UPDATE {table} id={row_id} "
                  f"→ {event.gtid.gtid}")

    def delete(self, table: str, row_id: int):
        with self._lock:
            if table in self.tables:
                self.tables[table].pop(row_id, None)
            event = BinlogEvent(
                gtid=self._next_gtid(),
                event_type=BinlogEventType.DELETE_ROWS,
                table=table,
                data={"id": row_id}
            )
            self.binlog.append(event)
            self._broadcast(event)
            print(f"  [LEADER] DELETE {table} id={row_id} "
                  f"→ {event.gtid.gtid}")

    def register_follower(self) -> queue.Queue:
        q: queue.Queue = queue.Queue()
        self.follower_queues.append(q)
        return q

class MySQLFollower:
    """팔로워: Relay Log를 수신하고 SQL Thread가 적용"""

    def __init__(self, name: str, leader: MySQLLeader,
                 apply_delay: float = 0.0):
        self.name = name
        self.apply_delay = apply_delay  # 복제 지연 시뮬레이션 (초)
        self.tables: dict[str, dict] = {}
        self.executed_gtids: set[str] = set()
        self.relay_log_queue = leader.register_follower()
        self._running = True

        # SQL Thread 시작
        self._thread = threading.Thread(
            target=self._sql_thread, daemon=True
        )
        self._thread.start()
        print(f"  Follower {name} connected "
              f"(delay={apply_delay}s)")

    def _apply_event(self, event: BinlogEvent):
        """SQL Thread: 이벤트를 팔로워 DB에 적용"""
        if event.gtid and event.gtid.gtid in self.executed_gtids:
            return  # 이미 적용된 트랜잭션 스킵

        if event.event_type == BinlogEventType.WRITE_ROWS:
            table = event.table
            if table not in self.tables:
                self.tables[table] = {}
            row_id = event.data["id"]
            self.tables[table][row_id] = {
                k: v for k, v in event.data.items() if k != "id"
            }

        elif event.event_type == BinlogEventType.UPDATE_ROWS:
            table = event.table
            row_id = event.data["id"]
            if table in self.tables and row_id in self.tables[table]:
                self.tables[table][row_id].update(
                    {k: v for k, v in event.data.items() if k != "id"}
                )

        elif event.event_type == BinlogEventType.DELETE_ROWS:
            table = event.table
            row_id = event.data["id"]
            if table in self.tables:
                self.tables[table].pop(row_id, None)

        if event.gtid:
            self.executed_gtids.add(event.gtid.gtid)
            print(f"  [{self.name}] Applied {event.gtid.gtid} "
                  f"({event.event_type.value} on {event.table})")

    def _sql_thread(self):
        while self._running:
            try:
                event = self.relay_log_queue.get(timeout=0.5)
                time.sleep(self.apply_delay)  # 복제 지연 시뮬레이션
                self._apply_event(event)
            except queue.Empty:
                continue

    def stop(self):
        self._running = False

    def replication_lag(self, leader: "MySQLLeader") -> int:
        """리더와의 GTID 차이 (미적용 트랜잭션 수)"""
        leader_gtids = {
            e.gtid.gtid for e in leader.binlog if e.gtid
        }
        return len(leader_gtids - self.executed_gtids)


# 시뮬레이션
leader = MySQLLeader()
follower1 = MySQLFollower("replica-1", leader, apply_delay=0.01)
follower2 = MySQLFollower("replica-2", leader, apply_delay=0.05)

time.sleep(0.1)

leader.insert("users", 1, {"name": "Alice", "email": "alice@example.com"})
leader.insert("users", 2, {"name": "Bob", "email": "bob@example.com"})
leader.update("users", 1, {"email": "alice@new.com"})
leader.delete("users", 2)
leader.insert("orders", 100, {"user_id": 1, "total": 9900})

time.sleep(0.3)  # 복제 완료 대기

print(f"\n  Leader users: {leader.tables.get('users', {})}")
print(f"  replica-1 users: {follower1.tables.get('users', {})}")
print(f"  replica-2 users: {follower2.tables.get('users', {})}")
print(f"  replica-1 lag: {follower1.replication_lag(leader)} txns")
print(f"  replica-2 lag: {follower2.replication_lag(leader)} txns")

follower1.stop()
follower2.stop()
```

### 예제 2: 복제 지연 모니터링 및 자동 Failover 시뮬레이션 (Python)

```python
import time
import random
from dataclasses import dataclass
from typing import Optional

@dataclass
class ReplicaStatus:
    name: str
    lag_seconds: float
    executed_gtid_count: int
    is_alive: bool

class ReplicationMonitor:
    """
    복제 지연 모니터링 및 자동 Failover 시뮬레이터
    실제 환경에서는 Orchestrator, MHA, ProxySQL 등이 이 역할을 담당한다.
    """

    MAX_ALLOWED_LAG = 10.0   # 최대 허용 복제 지연 (초)
    HEARTBEAT_TIMEOUT = 30.0  # 이 이상 지연 시 리더 장애로 판단

    def __init__(self):
        self.leader: Optional[str] = "db-primary"
        self.replicas: dict[str, ReplicaStatus] = {}
        self.leader_alive = True
        self.failover_history: list[dict] = []

    def register_replica(self, name: str):
        self.replicas[name] = ReplicaStatus(
            name=name, lag_seconds=0.0,
            executed_gtid_count=0, is_alive=True
        )
        print(f"  Replica {name} registered")

    def update_replica_status(self, name: str, lag: float,
                              gtid_count: int, alive: bool = True):
        if name in self.replicas:
            self.replicas[name].lag_seconds = lag
            self.replicas[name].executed_gtid_count = gtid_count
            self.replicas[name].is_alive = alive

    def check_replication_health(self) -> list[str]:
        """복제 지연 경고 발생 목록 반환"""
        warnings = []
        for name, status in self.replicas.items():
            if not status.is_alive:
                warnings.append(
                    f"CRITICAL: Replica {name} is DOWN!"
                )
            elif status.lag_seconds > self.HEARTBEAT_TIMEOUT:
                warnings.append(
                    f"CRITICAL: Replica {name} lag={status.lag_seconds:.1f}s "
                    f"(HEARTBEAT TIMEOUT)"
                )
            elif status.lag_seconds > self.MAX_ALLOWED_LAG:
                warnings.append(
                    f"WARNING: Replica {name} lag={status.lag_seconds:.1f}s "
                    f"(exceeds {self.MAX_ALLOWED_LAG}s threshold)"
                )
        return warnings

    def select_failover_candidate(self) -> Optional[str]:
        """
        Failover 후보 선택:
        1. 살아있는 팔로워 중
        2. 가장 많은 GTID를 적용한 (가장 최신 데이터를 가진) 팔로워
        """
        candidates = [
            s for s in self.replicas.values()
            if s.is_alive
        ]
        if not candidates:
            return None
        # 가장 많은 트랜잭션을 처리한 팔로워 선택
        best = max(candidates, key=lambda s: s.executed_gtid_count)
        return best.name

    def failover(self, reason: str = "Manual"):
        """자동 Failover 실행"""
        old_leader = self.leader
        candidate = self.select_failover_candidate()
        if not candidate:
            print("  [FAILOVER FAILED] No available candidates!")
            return

        print(f"\n  *** FAILOVER START ***")
        print(f"  Reason: {reason}")
        print(f"  Old leader: {old_leader}")
        print(f"  New leader: {candidate}")

        # Failover 실행: 후보를 새 리더로 승격
        self.leader = candidate
        del self.replicas[candidate]

        # 나머지 팔로워들의 리더를 변경
        print(f"  Redirecting {len(self.replicas)} replicas "
              f"to new leader {candidate}...")
        for name in self.replicas:
            print(f"    {name} → now replicating from {candidate}")

        self.failover_history.append({
            "timestamp": time.time(),
            "old_leader": old_leader,
            "new_leader": candidate,
            "reason": reason,
        })
        print(f"  *** FAILOVER COMPLETE ***\n")

    def print_status(self):
        print(f"\n  === Replication Status ===")
        print(f"  Leader: {self.leader} "
              f"({'ALIVE' if self.leader_alive else 'DOWN'})")
        print(f"  {'Replica':<15} {'Lag(s)':>8} {'GTIDs':>8} {'Status':>8}")
        print(f"  {'-'*45}")
        for name, status in self.replicas.items():
            health = "OK" if status.lag_seconds <= self.MAX_ALLOWED_LAG \
                          else "LAGGING"
            if not status.is_alive:
                health = "DOWN"
            print(f"  {name:<15} {status.lag_seconds:>8.1f} "
                  f"{status.executed_gtid_count:>8} {health:>8}")


# 시뮬레이션
monitor = ReplicationMonitor()
monitor.register_replica("db-replica-1")
monitor.register_replica("db-replica-2")
monitor.register_replica("db-replica-3")

print("\n--- 정상 상태 ---")
monitor.update_replica_status("db-replica-1", lag=0.2, gtid_count=1000)
monitor.update_replica_status("db-replica-2", lag=0.5, gtid_count=999)
monitor.update_replica_status("db-replica-3", lag=1.1, gtid_count=998)
monitor.print_status()
warnings = monitor.check_replication_health()
print(f"  Warnings: {warnings if warnings else 'None'}")

print("\n--- 복제 지연 발생 ---")
monitor.update_replica_status("db-replica-2", lag=15.0, gtid_count=985)
monitor.print_status()
for w in monitor.check_replication_health():
    print(f"  {w}")

print("\n--- 리더 장애 발생 → 자동 Failover ---")
monitor.leader_alive = False
monitor.failover(reason="Leader heartbeat timeout (60s)")
monitor.print_status()
```

## 주의사항과 팁

### 복제 지연(Replication Lag)의 주요 원인과 해결책

**1. 리더의 병렬 쓰기 vs. 팔로워의 단일 스레드 적용**

리더에서는 여러 트랜잭션이 동시에 실행되지만, 전통적인 팔로워의 SQL Thread는 단일 스레드로 직렬 적용한다. MySQL 5.7부터 도입된 **병렬 복제(Parallel Replication)**는 같은 스키마의 서로 다른 그룹에 속한 트랜잭션을 병렬 적용한다.

```sql
-- MySQL 병렬 복제 활성화
SET GLOBAL slave_parallel_type = 'LOGICAL_CLOCK';
SET GLOBAL slave_parallel_workers = 8;
```

**2. 대용량 트랜잭션**

수백만 행을 수정하는 `UPDATE users SET ...` 같은 대용량 트랜잭션은 팔로워에서 적용하는 데 오래 걸린다. 배치 작업은 작은 단위로 나눠 실행하자.

**3. `SHOW REPLICA STATUS`로 복제 상태 진단**

```sql
SHOW REPLICA STATUS\G
-- 주요 항목:
-- Seconds_Behind_Source: 복제 지연 초
-- Replica_IO_Running: I/O Thread 상태
-- Replica_SQL_Running: SQL Thread 상태
-- Last_Error: 오류 메시지
-- Executed_Gtid_Set: 적용 완료된 GTID 집합
```

### Read-Your-Writes 일관성 문제

사용자가 쓰기를 수행한 직후, 읽기 요청을 팔로워로 보내면 방금 쓴 데이터가 보이지 않을 수 있다. 이를 해결하는 방법:
- **Sticky Session**: 쓰기 직후 일정 시간 동안은 리더에서 읽기
- **GTID 기반 동기화**: 클라이언트가 받은 GTID를 팔로워가 적용할 때까지 대기
- **Monotonic Read**: 항상 같은 팔로워에서 읽도록 라우팅

### PostgreSQL의 WAL 스트리밍 복제

PostgreSQL은 MySQL의 Binlog 대신 **WAL(Write-Ahead Log)**를 복제에 사용한다. WAL 레코드를 스탠바이 서버에 스트리밍하는 방식으로 MySQL과 유사한 리더-팔로워 구조를 구현한다.

```sql
-- postgresql.conf (리더)
wal_level = replica
max_wal_senders = 5
wal_keep_size = 1GB

-- pg_hba.conf
host replication replicator replica-ip/32 md5

-- 팔로워 초기화
pg_basebackup -h leader-ip -U replicator -D /var/lib/postgresql/data -P --wal-method=stream
```

PostgreSQL 10+의 **Logical Replication**은 선택적 테이블만 복제하거나 이기종 PostgreSQL 버전 간 복제에 활용된다.

## 참고 자료
- [MySQL Replication – Delayed Replication (Official Docs)](https://dev.mysql.com/doc/mysql-replication-excerpt/8.0/en/replication-delayed.html)
- [Using GTID-based replication – Amazon RDS](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/mysql-replication-gtid.html)
- [Replicating MySQL: A Look at the Binlog and GTIDs – Airbyte](https://airbyte.com/blog/replicating-mysql-a-look-at-the-binlog-and-gtids)
- [What to Look for if Your MySQL Replication is Lagging – Severalnines](https://severalnines.com/blog/what-look-if-your-mysql-replication-lagging/)
