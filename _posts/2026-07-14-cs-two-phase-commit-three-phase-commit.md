---
layout: post
title: "2PC와 3PC 완전 정복: 분산 시스템 원자적 커밋 프로토콜의 핵심 원리"
date: 2026-07-14
categories: [cs, computer-science]
tags: [2PC, 3PC, 분산트랜잭션, 원자적커밋, 분산시스템, 데이터베이스, 합의알고리즘]
---

## 개요: 분산 트랜잭션의 근본 문제

분산 데이터베이스에서 트랜잭션을 수행한다고 생각해보자. 은행 이체 시스템에서 A 노드에서 잔액을 차감하고 B 노드에서 잔액을 증가시켜야 한다. 이때:

- A는 성공했지만 B가 실패하면? → 돈이 사라진다
- A와 B 중간에 코디네이터(Coordinator)가 죽으면? → 트랜잭션이 불확정 상태에 빠진다
- 네트워크가 단절되면? → 각 노드가 독립적으로 결정을 내릴 수도 없다

이것이 **분산 원자적 커밋(Distributed Atomic Commit)** 문제다. **2PC(Two-Phase Commit)**와 **3PC(Three-Phase Commit)**는 이 문제를 해결하기 위한 고전적 프로토콜이며, 모든 현대 분산 데이터베이스 시스템의 기반이 된다.

## 2PC (Two-Phase Commit) 상세 분석

### 구성 요소

- **코디네이터(Coordinator)**: 트랜잭션을 시작하고 최종 결정(커밋/어보트)을 내리는 노드
- **참여자(Participant)**: 코디네이터의 지시에 따라 실제 데이터 변경을 수행하는 노드들

### 2단계 동작 과정

#### Phase 1: 투표(Voting/Prepare) 단계

```
코디네이터                    참여자1        참여자2
    |                           |              |
    |──── PREPARE ───────────→  |              |
    |──── PREPARE ─────────────────────────→  |
    |                           |              |
    |     (redo/undo 로그 기록, 리소스 잠금)   |
    |                           |              |
    |←─── VOTE_YES ────────────  |              |
    |←─── VOTE_YES ─────────────────────────  |
```

코디네이터가 모든 참여자에게 **PREPARE** 메시지를 보낸다. 각 참여자는:
1. 트랜잭션을 실행할 수 있으면 → WAL(Write-Ahead Log)에 기록 후 **VOTE_YES**
2. 실행할 수 없으면 → **VOTE_NO** (즉시 로컬 어보트)

#### Phase 2: 커밋(Commit) 단계

```
코디네이터                    참여자1        참여자2
    |                           |              |
    | (모두 YES면 COMMIT 결정)   |              |
    | (하나라도 NO면 ABORT 결정) |              |
    |                           |              |
    |──── COMMIT ────────────→  |              |
    |──── COMMIT ──────────────────────────→  |
    |                           |              |
    |←─── ACK ─────────────────  |              |
    |←─── ACK ────────────────────────────── |
```

### 2PC 구현 (Python 시뮬레이션)

```python
import threading
import time
import random
from enum import Enum
from dataclasses import dataclass, field
from typing import Dict, List, Optional

class ParticipantState(Enum):
    IDLE = "IDLE"
    PREPARED = "PREPARED"
    COMMITTED = "COMMITTED"
    ABORTED = "ABORTED"

class CoordinatorState(Enum):
    INITIAL = "INITIAL"
    WAITING = "WAITING"
    COMMIT = "COMMIT"
    ABORT = "ABORT"
    COMPLETED = "COMPLETED"

@dataclass
class Transaction:
    txn_id: str
    operations: Dict[str, str]  # participant_id -> operation

class Participant:
    def __init__(self, pid: str, fail_rate: float = 0.0):
        self.pid = pid
        self.state = ParticipantState.IDLE
        self.fail_rate = fail_rate
        self.pending_log: Optional[str] = None
        self.lock = threading.Lock()

    def prepare(self, txn: Transaction) -> bool:
        """Phase 1: PREPARE 요청 처리"""
        with self.lock:
            if random.random() < self.fail_rate:
                print(f"  [{self.pid}] 장애 발생 → VOTE_NO")
                return False

            op = txn.operations.get(self.pid, "no-op")
            # WAL에 기록 (실제로는 디스크에 fsync)
            self.pending_log = f"txn={txn.txn_id}, op={op}"
            self.state = ParticipantState.PREPARED
            print(f"  [{self.pid}] 준비 완료 (로그: {self.pending_log}) → VOTE_YES")
            return True

    def commit(self, txn_id: str):
        """Phase 2: COMMIT 요청 처리"""
        with self.lock:
            assert self.state == ParticipantState.PREPARED, \
                f"잘못된 상태: {self.state}"
            # 실제 데이터 반영
            self.state = ParticipantState.COMMITTED
            print(f"  [{self.pid}] 커밋 완료")

    def abort(self, txn_id: str):
        """Phase 2: ABORT 요청 처리"""
        with self.lock:
            # PREPARED 상태면 언두, IDLE이면 아무것도 안 함
            if self.state == ParticipantState.PREPARED:
                self.pending_log = None  # 언두
            self.state = ParticipantState.ABORTED
            print(f"  [{self.pid}] 어보트 완료")

class TwoPhaseCommitCoordinator:
    def __init__(self, participants: List[Participant]):
        self.participants = participants
        self.state = CoordinatorState.INITIAL

    def execute(self, txn: Transaction) -> bool:
        print(f"\n[코디네이터] 트랜잭션 {txn.txn_id} 시작")
        self.state = CoordinatorState.WAITING

        # Phase 1: 투표
        print("[코디네이터] Phase 1: PREPARE 브로드캐스트")
        votes = []
        for p in self.participants:
            vote = p.prepare(txn)
            votes.append(vote)

        # 결정
        if all(votes):
            self.state = CoordinatorState.COMMIT
            print("[코디네이터] 모든 참여자 동의 → COMMIT 결정")
            # Phase 2: 커밋 (코디네이터 로그에 COMMIT 기록 후 전송)
            print("[코디네이터] Phase 2: COMMIT 브로드캐스트")
            for p in self.participants:
                p.commit(txn.txn_id)
        else:
            self.state = CoordinatorState.ABORT
            print("[코디네이터] 일부 참여자 거부 → ABORT 결정")
            print("[코디네이터] Phase 2: ABORT 브로드캐스트")
            for p in self.participants:
                p.abort(txn.txn_id)

        self.state = CoordinatorState.COMPLETED
        result = self.state == CoordinatorState.COMPLETED and all(
            p.state == ParticipantState.COMMITTED for p in self.participants
        )
        print(f"[코디네이터] 트랜잭션 {txn.txn_id} 완료: {'성공' if all(votes) else '어보트'}")
        return all(votes)

# 시뮬레이션
participants = [
    Participant("DB_Seoul", fail_rate=0.0),
    Participant("DB_Busan", fail_rate=0.0),
    Participant("DB_Jeju",  fail_rate=0.0),
]
coordinator = TwoPhaseCommitCoordinator(participants)

txn = Transaction(
    txn_id="TXN-001",
    operations={
        "DB_Seoul": "UPDATE accounts SET balance = balance - 1000 WHERE id = 1",
        "DB_Busan": "UPDATE accounts SET balance = balance + 1000 WHERE id = 2",
        "DB_Jeju":  "INSERT INTO audit_log VALUES ('transfer', 1000)",
    }
)

success = coordinator.execute(txn)
print(f"\n결과: {'성공' if success else '실패'}")

# 장애 시뮬레이션
print("\n=== 장애 시뮬레이션 ===")
participants_with_failure = [
    Participant("DB_Seoul", fail_rate=0.0),
    Participant("DB_Busan", fail_rate=1.0),  # 항상 실패
]
coordinator2 = TwoPhaseCommitCoordinator(participants_with_failure)
coordinator2.execute(Transaction("TXN-002", {"DB_Seoul": "op1", "DB_Busan": "op2"}))
```

## 2PC의 치명적 약점: 블로킹 문제

2PC는 특정 상황에서 **무한 블로킹(Blocking)**이 발생할 수 있다:

### 시나리오: 코디네이터 장애

```
시간축
  T1: 코디네이터 → 참여자A PREPARE 전송
  T2: 참여자A → VOTE_YES (PREPARED 상태로 잠금 유지)
  T3: 코디네이터가 COMMIT/ABORT 보내기 전에 코디네이터 죽음!
  T4: 참여자A는 잠금을 유지한 채 무한 대기...
      (다른 트랜잭션들이 해당 리소스에 접근 불가)
```

PREPARED 상태의 참여자는:
- 독자적으로 커밋도, 어보트도 할 수 없다 (코디네이터의 결정을 기다려야 함)
- 다른 참여자들과 직접 통신해도 어보트했는지 알 수 없음
- 코디네이터가 복구될 때까지 잠금 보유 (긴 서비스 중단)

## 3PC (Three-Phase Commit): 블로킹 해결 시도

3PC는 2PC의 블로킹 문제를 해결하기 위해 **PRE-COMMIT 단계**를 추가한다.

### 3단계 동작 과정

```
Phase 1: CanCommit
코디네이터 → 참여자: "커밋할 수 있나요? (데이터 잠금 아직 안 함)"
참여자 → 코디네이터: YES/NO

Phase 2: PreCommit
코디네이터 → 참여자: "커밋 준비하세요" (이제 잠금 획득, WAL 기록)
참여자 → 코디네이터: ACK

Phase 3: DoCommit
코디네이터 → 참여자: "커밋하세요"
참여자 → 코디네이터: ACK
```

### 3PC의 핵심 차이점

```python
class ThreePhaseCommitParticipant:
    def __init__(self, pid: str):
        self.pid = pid
        self.state = "IDLE"
        self.timeout = 5.0  # 타임아웃 (초)

    def can_commit(self, txn_id: str) -> bool:
        """Phase 1: 가능 여부만 확인 (잠금 없음)"""
        # 아직 리소스 잠금을 획득하지 않음
        self.state = "UNCERTAIN"
        print(f"  [{self.pid}] 리소스 확인 (잠금 없음) → YES")
        return True

    def pre_commit(self, txn_id: str) -> bool:
        """Phase 2: 실제 준비 (잠금 획득, WAL 기록)"""
        # PRE-COMMITTED 상태에서 타임아웃 시 자동 커밋 가능!
        self.state = "PRE_COMMITTED"
        print(f"  [{self.pid}] 잠금 획득, WAL 기록 → ACK (PRE_COMMITTED)")
        return True

    def do_commit(self, txn_id: str):
        """Phase 3: 최종 커밋"""
        assert self.state == "PRE_COMMITTED"
        self.state = "COMMITTED"
        print(f"  [{self.pid}] 커밋 완료")

    def handle_coordinator_timeout(self):
        """코디네이터 타임아웃 시 독자적 결정"""
        if self.state == "PRE_COMMITTED":
            # PRE_COMMITTED 상태: 코디네이터가 커밋 결정을 했다고 유추 가능
            # (모든 참여자가 PRE_COMMITTED가 됐을 때만 이 상태에 도달)
            print(f"  [{self.pid}] 타임아웃 → PRE_COMMITTED이므로 자동 커밋")
            self.state = "COMMITTED"
        elif self.state == "UNCERTAIN":
            # UNCERTAIN 상태: 아직 잠금도 없음, 안전하게 어보트
            print(f"  [{self.pid}] 타임아웃 → UNCERTAIN이므로 자동 어보트")
            self.state = "ABORTED"
```

### 3PC의 비차단성(Non-blocking) 보장 원리

**핵심 불변식**: 참여자가 PRE_COMMITTED 상태에 있다면, **모든** 참여자가 YES를 투표했다는 것을 안다. 따라서 코디네이터 없이도 커밋해도 안전하다.

```
UNCERTAIN 상태 참여자: "아직 아무것도 모름 → 어보트 (안전)"
PRE_COMMITTED 참여자: "모두 동의했음 → 커밋 (안전)"
```

## 3PC의 한계: 네트워크 파티션

3PC는 **네트워크 파티션**에는 취약하다:

```
시나리오:
  참여자A: PRE_COMMITTED → 타임아웃 → 커밋
  참여자B: 네트워크 분리 → UNCERTAIN → 타임아웃 → 어보트

결과: A는 커밋, B는 어보트 → 불일치!
```

이것이 3PC가 실제로 거의 사용되지 않는 이유다. CAP 정리에 따르면 네트워크 파티션 상황에서 일관성과 가용성을 동시에 보장할 수 없다.

## 실제 시스템들의 선택

### PostgreSQL의 2PC 구현

PostgreSQL은 `PREPARE TRANSACTION` / `COMMIT PREPARED` SQL 명령으로 2PC를 지원한다:

```sql
-- 참여자1 (PostgreSQL)
BEGIN;
UPDATE accounts SET balance = balance - 1000 WHERE id = 1;
PREPARE TRANSACTION 'txn_transfer_001';
-- 이 시점에서 트랜잭션은 PREPARED 상태

-- 코디네이터가 모든 참여자의 PREPARE 완료를 확인 후

-- Phase 2: 커밋
COMMIT PREPARED 'txn_transfer_001';
-- 또는 어보트
ROLLBACK PREPARED 'txn_transfer_001';
```

준비된 트랜잭션은 `pg_prepared_xacts` 시스템 뷰로 확인:

```sql
SELECT gid, prepared, owner, database
FROM pg_prepared_xacts;
```

### XA 트랜잭션 (Java / JDBC)

```java
import javax.transaction.xa.*;
import java.sql.*;

// XA 코디네이터 시뮬레이션
public class XACoordinator {
    public boolean executeDistributedTransaction(
            XAConnection conn1, XAConnection conn2, 
            String sql1, String sql2) throws Exception {
        
        XAResource xa1 = conn1.getXAResource();
        XAResource xa2 = conn2.getXAResource();
        
        Xid xid = new MyXid(100, new byte[]{0x01}, new byte[]{0x02});
        
        try {
            // Phase 1: PREPARE
            xa1.start(xid, XAResource.TMNOFLAGS);
            conn1.getConnection().prepareStatement(sql1).execute();
            xa1.end(xid, XAResource.TMSUCCESS);
            int vote1 = xa1.prepare(xid);

            xa2.start(xid, XAResource.TMNOFLAGS);
            conn2.getConnection().prepareStatement(sql2).execute();
            xa2.end(xid, XAResource.TMSUCCESS);
            int vote2 = xa2.prepare(xid);

            // Phase 2: COMMIT
            if (vote1 != XAResource.XA_RDONLY) xa1.commit(xid, false);
            if (vote2 != XAResource.XA_RDONLY) xa2.commit(xid, false);
            return true;

        } catch (XAException e) {
            // ABORT
            try { xa1.rollback(xid); } catch (Exception ignored) {}
            try { xa2.rollback(xid); } catch (Exception ignored) {}
            return false;
        }
    }
}
```

## 현대적 대안들

### 1. Saga 패턴

2PC의 블로킹 문제를 피하면서 분산 트랜잭션을 구현하는 대표적 패턴. 각 서비스는 로컬 트랜잭션만 수행하고, 실패 시 **보상 트랜잭션(Compensating Transaction)**을 실행한다.

### 2. Percolator (Google BigTable)

Google이 설계한 낙관적 잠금 기반 분산 트랜잭션 모델. TiKV(TiDB의 저장 엔진)가 채택했다. 타임스탬프 오라클(TSO)을 활용한 MVCC로 2PC를 구현하되, 잠금을 데이터와 함께 인라인으로 저장해 코디네이터 의존도를 줄였다.

### 3. Calvin / Deterministic Databases

트랜잭션을 미리 전역 순서로 정렬한 후 결정론적으로 실행 → 잠금 불필요. VoltDB, FaunaDB 등이 채택.

## 주의사항과 실전 팁

### 1. 블로킹 방지를 위한 휴리스틱 완료(Heuristic Completion)

코디네이터 장애 시 너무 오래 블로킹이 지속되면, 참여자는 **휴리스틱 결정**을 내릴 수 있다. 이는 일관성을 포기하는 비상수단이며, PostgreSQL에서는 `max_prepared_transactions` 설정과 `ROLLBACK PREPARED`로 운영자가 수동으로 처리해야 한다.

### 2. 타임아웃 설정

```python
# 코디네이터 타임아웃: Phase 1 투표 응답 대기
PREPARE_TIMEOUT_SECONDS = 30

# 참여자 타임아웃: Phase 2 결정 대기 (길게 설정)
COMMIT_DECISION_TIMEOUT_SECONDS = 300  # 코디네이터가 복구될 시간

# 코디네이터가 로그에 결정을 기록하는 것이 가장 중요
# 복구 시 로그에서 결정을 재전송
```

### 3. 2PC를 피해야 할 때

- **고가용성이 최우선**인 서비스: 블로킹 위험
- **느슨한 결합 마이크로서비스**: Saga 패턴 선호
- **성능이 매우 중요한 경우**: 최소 2라운드 통신 = 높은 레이턴시

## 마무리

2PC는 분산 환경에서 원자성을 보장하는 가장 검증된 프로토콜이지만, 코디네이터 장애 시 블로킹이라는 근본적 한계를 갖는다. 3PC는 이를 이론적으로 해결하지만 네트워크 파티션에 취약하여 실용적이지 않다.

실무에서는 2PC를 사용할 때 코디네이터 고가용성(레플리케이션), 짧은 트랜잭션 유지, 그리고 장애 복구 절차를 함께 설계해야 한다. 마이크로서비스 환경에서는 Saga 패턴이나 이벤트 소싱을 통해 분산 트랜잭션을 대체하는 것이 현대적인 접근법이다.

## 참고 자료

- [Two-phase commit protocol — Wikipedia](https://en.wikipedia.org/wiki/Two-phase_commit_protocol)
- [Three-phase commit protocol — Wikipedia](https://en.wikipedia.org/wiki/Three-phase_commit_protocol)
- [TiKV Distributed Algorithms — TiKV Deep Dive](https://tikv.org/deep-dive/distributed-transaction/distributed-algorithms/)
- [Two-Phase Commit Protocol Explained — Endgrate](https://endgrate.com/blog/two-phase-commit-protocol-explained)
