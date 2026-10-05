---
layout: post
title: "2PC와 3PC 완전 정복: 분산 트랜잭션 원자 커밋 프로토콜의 원리와 한계"
date: 2026-10-05
categories: [cs, computer-science]
tags: [distributed-systems, 2PC, 3PC, atomic-commit, consensus, transaction, fault-tolerance]
---

## 개요

은행 이체를 생각해보자. A 계좌에서 100만 원을 차감하고 B 계좌에 100만 원을 추가하는 작업이다. 두 작업이 하나의 DB에 있다면 ACID 트랜잭션으로 간단히 해결된다. 그런데 A 계좌는 서울 데이터센터의 DB에, B 계좌는 부산 데이터센터의 DB에 있다면 어떻게 해야 할까?

이것이 **분산 원자 커밋(Distributed Atomic Commit)** 문제다. 여러 노드에 걸쳐 있는 작업을 "**모두 커밋(commit) or 모두 롤백(abort)**"으로 처리해야 한다. 일부 노드만 커밋하고 나머지가 중단되면 데이터 불일치가 발생한다.

**Two-Phase Commit(2PC)**과 **Three-Phase Commit(3PC)**은 이 문제를 해결하는 고전적인 프로토콜이다. 각각의 작동 원리, 한계, 그리고 실제 시스템에서의 대안을 깊이 살펴본다.

---

## 왜 필요한가

### 분산 환경의 새로운 문제

단일 노드 트랜잭션과 달리 분산 환경에서는 다음 장애가 추가로 발생한다:

- **노드 충돌(Node Crash)**: 참여 노드가 커밋 도중 다운될 수 있다
- **네트워크 파티션(Network Partition)**: 노드 간 통신이 일시적으로 끊길 수 있다
- **메시지 손실/지연**: 네트워크가 불안정하면 메시지가 도착하지 않거나 늦게 도착한다

단순히 각 노드에 개별 커밋을 보내면:

```
Coordinator → NodeA: COMMIT
Coordinator → NodeB: COMMIT (← 이 메시지가 전달되기 전에 Coordinator 다운!)
```

NodeA는 커밋했고, NodeB는 여전히 PENDING 상태. 데이터 불일치 발생.

---

## Two-Phase Commit (2PC)

### 프로토콜 구성

2PC는 **코디네이터(Coordinator)**와 **참여자(Participant)** 역할로 구성된다. 코디네이터는 트랜잭션 전체를 조율하고, 참여자는 각 노드에서 실제 작업을 수행한다.

#### Phase 1: Prepare (투표 단계)

```
Coordinator → 모든 Participant: PREPARE

각 Participant:
  - 트랜잭션 작업을 수행할 수 있는지 검사
  - 가능하면: 작업 결과를 WAL(Write-Ahead Log)에 기록 후 잠금 유지
  - 응답: YES or NO

Participant → Coordinator: YES (or NO)
```

#### Phase 2: Commit or Abort (결정 단계)

```
모든 YES 수신 → Coordinator → 모든 Participant: COMMIT
하나라도 NO    → Coordinator → 모든 Participant: ABORT

각 Participant:
  - COMMIT: 실제로 변경사항 적용, 잠금 해제
  - ABORT:  변경사항 롤백, 잠금 해제
```

### Python 시뮬레이션

```python
import time
import random
from enum import Enum

class Vote(Enum):
    YES = "YES"
    NO = "NO"

class Decision(Enum):
    COMMIT = "COMMIT"
    ABORT = "ABORT"

class Participant:
    def __init__(self, name: str, fail_rate: float = 0.0):
        self.name = name
        self.fail_rate = fail_rate
        self.state = "INITIAL"  # INITIAL → PREPARED → COMMITTED/ABORTED
        self.pending_work = None

    def prepare(self, transaction: dict) -> Vote:
        """Phase 1: 트랜잭션 수행 가능 여부 투표"""
        if random.random() < self.fail_rate:
            print(f"  [{self.name}] 장애 발생! 투표 불가")
            return Vote.NO

        # 실제 작업 준비 (WAL에 기록, 잠금 획득)
        self.pending_work = transaction
        self.state = "PREPARED"
        print(f"  [{self.name}] PREPARED → YES 투표")
        return Vote.YES

    def commit(self):
        """Phase 2: 커밋 수행"""
        if self.state != "PREPARED":
            raise RuntimeError(f"{self.name}: PREPARED 상태가 아님")
        self.state = "COMMITTED"
        self.pending_work = None
        print(f"  [{self.name}] COMMITTED ✓")

    def abort(self):
        """Phase 2: 롤백 수행"""
        self.state = "ABORTED"
        self.pending_work = None
        print(f"  [{self.name}] ABORTED ✗")


class TwoPhaseCommitCoordinator:
    def __init__(self, participants: list[Participant]):
        self.participants = participants

    def run_transaction(self, transaction: dict) -> bool:
        print("\n=== Phase 1: PREPARE ===")
        votes = {}
        for p in self.participants:
            vote = p.prepare(transaction)
            votes[p.name] = vote
            if vote == Vote.NO:
                break  # 하나라도 NO면 즉시 중단

        decision = (Decision.COMMIT
                    if all(v == Vote.YES for v in votes.values())
                    else Decision.ABORT)

        print(f"\n=== Phase 2: {decision.value} ===")
        success = True
        for p in self.participants:
            if p.state == "PREPARED":
                if decision == Decision.COMMIT:
                    p.commit()
                else:
                    p.abort()
            elif p.state == "INITIAL":
                # YES를 받지 못한 참여자는 이미 롤백 상태
                p.abort()

        print(f"\n결과: 트랜잭션 {'성공' if decision == Decision.COMMIT else '실패'}")
        return decision == Decision.COMMIT


# 사용 예시
tx = {"type": "transfer", "from": "A", "to": "B", "amount": 1_000_000}

# 정상 시나리오
print("=== 정상 시나리오 ===")
participants = [Participant("Seoul-DB"), Participant("Busan-DB")]
coordinator = TwoPhaseCommitCoordinator(participants)
coordinator.run_transaction(tx)

# 노드 장애 시나리오
print("\n=== 노드 장애 시나리오 ===")
participants = [Participant("Seoul-DB"), Participant("Busan-DB", fail_rate=1.0)]
coordinator = TwoPhaseCommitCoordinator(participants)
coordinator.run_transaction(tx)
```

### 2PC의 한계: 블로킹 프로토콜

2PC의 치명적 약점은 **코디네이터 단일 장애점(SPOF)**이다.

```
시나리오: 코디네이터가 COMMIT 메시지를 일부에만 보내고 다운

Coordinator → NodeA: COMMIT  ← NodeA는 커밋 완료
Coordinator [CRASH]
NodeB: Phase 2 메시지를 기다리는 중... (UNCERTAIN 상태)
```

NodeB는 커밋해야 할지 롤백해야 할지 알 수 없다. 코디네이터가 복구될 때까지 **무한정 블록**된다. 잠금을 해제하지 못하므로 다른 트랜잭션도 대기한다.

**2PC는 blocking protocol이다**: 코디네이터 장애 시 참여자들이 잠금을 유지한 채 블록된다.

---

## Three-Phase Commit (3PC)

3PC는 2PC의 블로킹 문제를 해결하기 위해 **PreCommit** 단계를 추가한다.

### 핵심 아이디어

2PC가 블로킹인 이유: Phase 1 이후 참여자들은 코디네이터의 결정을 알 수 없다. Phase 1에서 YES를 투표했다는 것은 "커밋할 수 있다"는 의미이지, "커밋이 확정됐다"는 의미가 아니다.

3PC는 다음 두 가지 불변식을 보장함으로써 비블로킹을 달성한다:

1. `PreCommit` 메시지를 받은 참여자가 없다면 → 안전하게 ABORT 가능
2. `PreCommit` 메시지를 받은 참여자가 있다면 → 반드시 COMMIT (롤백 불가)

### 프로토콜

#### Phase 1: CanCommit (투표)
```
Coordinator → 모든 Participant: CAN_COMMIT?
Participant → Coordinator: YES or NO
```

#### Phase 2: PreCommit (의향 공표)
```
모든 YES → Coordinator → 모든 Participant: PRE_COMMIT
           각 Participant: "커밋 직전 상태" 로그 기록, ACK 전송

하나라도 NO → Coordinator → 모든 Participant: ABORT
```

#### Phase 3: DoCommit (최종 커밋)
```
모든 ACK 수신 → Coordinator → 모든 Participant: DO_COMMIT
               각 Participant: 실제 커밋 수행
```

### Python 시뮬레이션 (3PC 상태 기계)

```python
from enum import Enum, auto

class ParticipantState3PC(Enum):
    INITIAL = auto()
    WAITING = auto()      # Phase 1 투표 후
    PREPARED = auto()     # Phase 2 PRE_COMMIT 수신 후
    COMMITTED = auto()
    ABORTED = auto()

class Participant3PC:
    def __init__(self, name: str):
        self.name = name
        self.state = ParticipantState3PC.INITIAL
        self.timeout = 3.0  # 타임아웃 (초)

    def on_can_commit(self) -> bool:
        """Phase 1: 커밋 가능 여부 응답"""
        self.state = ParticipantState3PC.WAITING
        can = True  # 실제로는 리소스 가용성 체크
        print(f"  [{self.name}] CanCommit → {'YES' if can else 'NO'}")
        return can

    def on_pre_commit(self):
        """Phase 2: PreCommit 수신 - 이 시점부터 커밋 의향 확정"""
        self.state = ParticipantState3PC.PREPARED
        print(f"  [{self.name}] PreCommit 수신 → PREPARED (이제 COMMIT 방향)")

    def on_do_commit(self):
        """Phase 3: 실제 커밋"""
        self.state = ParticipantState3PC.COMMITTED
        print(f"  [{self.name}] DoCommit → COMMITTED ✓")

    def on_abort(self):
        """어느 단계에서나 ABORT 수신 시"""
        self.state = ParticipantState3PC.ABORTED
        print(f"  [{self.name}] ABORTED ✗")

    def handle_coordinator_timeout(self):
        """
        코디네이터 응답 타임아웃 처리.
        3PC의 핵심: 상태에 따라 안전하게 결정 가능.
        """
        if self.state == ParticipantState3PC.WAITING:
            # PreCommit을 받지 못함 → 코디네이터는 아직 결정 안 함
            # 안전하게 ABORT 가능
            print(f"  [{self.name}] 타임아웃 (WAITING) → 안전하게 ABORT")
            self.on_abort()
        elif self.state == ParticipantState3PC.PREPARED:
            # PreCommit을 받았음 → 모든 참여자가 YES 투표함
            # 다른 살아있는 참여자들도 PREPARED 상태
            # → 과반수 프로토콜로 COMMIT 진행 가능
            print(f"  [{self.name}] 타임아웃 (PREPARED) → 다른 노드와 협의 후 COMMIT")
            # 실제 구현: 다른 참여자에게 직접 연락하여 state 확인
            self.on_do_commit()


class ThreePhaseCommitCoordinator:
    def __init__(self, participants: list[Participant3PC]):
        self.participants = participants

    def run(self) -> bool:
        # Phase 1: CanCommit
        print("\n=== Phase 1: CanCommit ===")
        votes = [p.on_can_commit() for p in self.participants]

        if not all(votes):
            print("=== ABORT (투표 실패) ===")
            for p in self.participants:
                p.on_abort()
            return False

        # Phase 2: PreCommit
        print("\n=== Phase 2: PreCommit ===")
        for p in self.participants:
            p.on_pre_commit()

        # Phase 3: DoCommit
        print("\n=== Phase 3: DoCommit ===")
        for p in self.participants:
            p.on_do_commit()

        return True


# 실행 예시
print("=== 3PC 정상 실행 ===")
participants = [Participant3PC("Node-A"), Participant3PC("Node-B"), Participant3PC("Node-C")]
coord = ThreePhaseCommitCoordinator(participants)
coord.run()

# 타임아웃 시뮬레이션: WAITING 상태에서 코디네이터 다운
print("\n=== 코디네이터 타임아웃 (WAITING 상태) ===")
p = Participant3PC("Node-X")
p.on_can_commit()  # WAITING 상태로 전환
p.handle_coordinator_timeout()  # → 안전하게 ABORT

# 타임아웃 시뮬레이션: PREPARED 상태에서 코디네이터 다운
print("\n=== 코디네이터 타임아웃 (PREPARED 상태) ===")
p2 = Participant3PC("Node-Y")
p2.on_can_commit()
p2.on_pre_commit()  # PREPARED 상태로 전환
p2.handle_coordinator_timeout()  # → COMMIT 진행
```

---

## 2PC vs 3PC 비교

| 특성 | 2PC | 3PC |
|------|-----|-----|
| 단계 수 | 2 | 3 |
| 블로킹 여부 | **블로킹** | 비블로킹 |
| 메시지 수 | 4n | 6n |
| 네트워크 파티션 | 블로킹 | **여전히 불일치 가능** |
| 코디네이터 SPOF | 있음 | 줄어듦 |
| 성능 | 빠름 | 느림 |
| 실제 사용 | **흔함** | 드묾 |

### 3PC도 완벽하지 않다

3PC는 **비동기 네트워크(asynchronous network)**에서 여전히 불일치가 발생할 수 있다. 네트워크 파티션이 발생하면:

```
파티션 발생: [Node-A, Node-B] | [Node-C, Node-D]

Node-A: PREPARED 상태, 타임아웃 → COMMIT 결정
Node-C: WAITING 상태, 타임아웃 → ABORT 결정

→ Node-A,B는 COMMIT, Node-C,D는 ABORT → 불일치!
```

이는 **FLP 불가능성 정리(FLP Impossibility)**의 직접적인 결과다: 완전 비동기 네트워크에서 결정론적으로 합의를 달성하는 것은 불가능하다.

---

## 실제 시스템의 접근 방식

### 1. Paxos/Raft 기반 분산 커밋

Google Spanner, CockroachDB, TiDB는 Paxos 또는 Raft를 사용하여 2PC의 코디네이터 SPOF 문제를 해결한다. 코디네이터 자체를 Raft 그룹으로 복제하여 코디네이터 장애 시 새 리더가 취임하고 진행 중인 트랜잭션을 이어받는다.

### 2. Percolator (Google)

BigTable 위에서 2PC를 구현한 방식. 코디네이터가 다운되면 다른 트랜잭션이 충돌 감지 시 **롤포워드(roll forward)** 또는 **롤백**을 대신 수행한다. 코디네이터 없이도 진행 가능.

### 3. Saga 패턴

마이크로서비스에서 2PC 대신 사용하는 장기 실행 트랜잭션 패턴. 각 로컬 트랜잭션이 성공하면 이벤트를 발행하고, 실패하면 보상(compensating) 트랜잭션을 실행한다. 강한 일관성 대신 최종적 일관성을 제공한다.

### 4. 실용적 2PC 구현 전략

```python
# 실무에서 2PC의 블로킹을 완화하는 방법

class RobustTwoPC:
    """
    - 코디네이터 상태를 영속 저장소에 기록
    - 참여자 타임아웃 처리
    - 코디네이터 재시작 후 미완료 트랜잭션 복구
    """

    def prepare_and_log(self, tx_id: str, participants: list):
        """
        PREPARE 전 코디네이터 로그에 기록.
        재시작 후 이 로그로 미완료 트랜잭션을 식별.
        """
        # 1. 영속 저장소에 "PREPARING" 상태 기록
        self.log_store.write(tx_id, state="PREPARING", participants=participants)

        votes = []
        for p in participants:
            try:
                vote = p.prepare(timeout=30)  # 타임아웃 설정
                votes.append(vote)
            except TimeoutError:
                votes.append(Vote.NO)

        decision = Decision.COMMIT if all(v == Vote.YES for v in votes) else Decision.ABORT

        # 2. 결정을 영속 저장소에 기록 (이 시점이 커밋 포인트)
        self.log_store.write(tx_id, state=decision.value)

        # 3. 모든 참여자에게 결정 전송 (실패해도 재시도 가능)
        for p in participants:
            self._deliver_decision_with_retry(p, decision, max_retries=10)

    def recover_on_startup(self):
        """재시작 후 미완료 트랜잭션 처리"""
        for tx_id, entry in self.log_store.get_pending():
            if entry.state == "PREPARING":
                # 결정 로그 없음 → ABORT
                self._broadcast_abort(tx_id, entry.participants)
            elif entry.state in ("COMMIT", "ABORT"):
                # 결정은 났지만 전달 실패 → 재전송
                self._redeliver_decision(tx_id, entry)
```

---

## 주의사항 및 팁

### 1. 타임아웃은 반드시 설정하라
2PC에서 참여자가 코디네이터 응답을 무한정 기다리는 것은 위험하다. 실무에서는 타임아웃 후 운영자 알림 또는 자동 ABORT를 구현한다.

### 2. 멱등성(Idempotency) 보장
COMMIT 메시지가 중복 전달될 수 있다. 참여자는 같은 트랜잭션 ID에 대한 중복 커밋을 무시해야 한다.

### 3. 로그는 커밋 포인트 전후로 분리
코디네이터의 결정 로그(`state=COMMIT`)가 **커밋 포인트**다. 이 로그가 디스크에 영속화되기 전에 코디네이터가 다운되면 ABORT, 이후에 다운되면 복구 후 COMMIT이 보장된다.

### 4. 분산 데드락 감지
2PC 중 분산 데드락이 발생할 수 있다. Wait-for 그래프를 전역으로 관리하거나 타임아웃 기반으로 데드락을 탐지한다.

### 5. 실무에서의 선택 가이드

```
같은 데이터센터 내 소수의 DB 서버 → 2PC (낮은 레이턴시, 단순함)
지역 간 분산 시스템                → Paxos/Raft 기반 커밋 (높은 가용성)
마이크로서비스 환경                → Saga 패턴 (서비스 자율성 보장)
단순 내구성 요구사항               → 로컬 트랜잭션 + 메시지 큐 아웃박스 패턴
```

---

## 마무리

2PC는 분산 트랜잭션의 근본적인 해법을 제공하지만 코디네이터 장애 시 블로킹 문제가 있다. 3PC는 이를 개선했지만 네트워크 파티션 하에서는 여전히 불일치 가능성이 있고 메시지 비용이 크다. 실제 대규모 시스템은 Paxos/Raft 기반 분산 커밋이나 Saga 패턴 같은 대안을 사용하는 경우가 많다. 중요한 것은 자신의 시스템이 얼마나 강한 일관성을 필요로 하는지, 그리고 그 비용(레이턴시, 가용성)을 감당할 수 있는지 명확히 이해하고 프로토콜을 선택하는 것이다.

## 참고 자료
- [Wikipedia: Two-phase commit protocol](https://en.wikipedia.org/wiki/Two-phase_commit_protocol)
- [The Paper Trail: Consensus Protocols - Three-Phase Commit](https://www.the-paper-trail.org/post/2008-11-29-consensus-protocols-three-phase-commit/)
- [Fischer, Lynch, Paterson (1985). Impossibility of Distributed Consensus with One Faulty Process.](https://doi.org/10.1145/3149.214121)
- [Bernstein, Hadzilacos, Goodman (1987). Concurrency Control and Recovery in Database Systems.](https://www.microsoft.com/en-us/research/publication/concurrency-control-recovery-database-systems/)
