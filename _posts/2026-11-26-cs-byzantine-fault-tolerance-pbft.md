---
layout: post
title: "Byzantine Fault Tolerance 완전 정복: PBFT 알고리즘으로 배우는 분산 시스템 비잔틴 장애 허용"
date: 2026-11-26
categories: [cs, computer-science]
tags: [BFT, PBFT, 분산시스템, 합의알고리즘, 블록체인, Hyperledger, 비잔틴장애]
---

분산 시스템에서 노드가 단순히 죽거나(Crash Fault) 멈추는 것은 쉽게 처리할 수 있습니다. 하지만 노드가 거짓 정보를 보내거나, 일부 노드에게는 다른 메시지를 보내거나, 아예 악의적으로 행동한다면 어떻게 될까요? 이것이 바로 **비잔틴 장애(Byzantine Fault)**이며, 이를 허용(Tolerate)하는 시스템 설계가 Byzantine Fault Tolerance(BFT)입니다.

## 비잔틴 장군 문제란?

1982년 Leslie Lamport, Robert Shostak, Marshall Pease가 제안한 사고 실험으로, 비잔틴 제국의 장군들이 통신하면서 배신자가 섞여 있을 때 올바른 합의를 이룰 수 있는가를 묻습니다.

**문제 설정:**
- N명의 장군이 있고, 그 중 F명은 반역자(Byzantine node)일 수 있습니다.
- 반역자는 임의의 메시지를 보내거나, 다른 장군에게 서로 다른 메시지를 보낼 수 있습니다.
- 나머지 충성스러운 장군들은 반드시 같은 결정(공격 or 후퇴)에 도달해야 합니다.

**핵심 결과:** 비잔틴 장애를 허용하려면 **최소 3f+1개의 노드**가 필요합니다. f개의 불량 노드를 허용하려면 그 3배보다 1 많은 정상 노드가 있어야 합니다.

### Crash Fault vs Byzantine Fault

| 구분 | Crash Fault (CFT) | Byzantine Fault (BFT) |
|---|---|---|
| 노드 행동 | 멈춤/침묵 | 임의/악의적 행동 |
| 필요 복제본 | 2f+1 | 3f+1 |
| 대표 알고리즘 | Paxos, Raft | PBFT, Tendermint, HotStuff |
| 적용 환경 | 신뢰 가능한 내부망 | 퍼블릭 네트워크, 블록체인 |

## PBFT(Practical Byzantine Fault Tolerance)

1999년 Miguel Castro와 Barbara Liskov가 OSDI에서 발표한 PBFT는 비잔틴 장애를 현실적으로 처리하는 최초의 실용적 알고리즘입니다. 비동기 네트워크에서도 안전성(Safety)을 보장하고, 약한 동기성(Weak Synchrony) 가정 아래 활성성(Liveness)을 제공합니다.

### PBFT 프로토콜 단계

PBFT는 세 단계 프로토콜(Three-Phase Protocol)로 작동합니다:

1. **Pre-Prepare**: 리더(Primary) 노드가 클라이언트 요청에 순서 번호(sequence number)를 부여하고 모든 복제본에 전달.
2. **Prepare**: 각 복제본이 Pre-Prepare를 검증하고 2f+1개의 Prepare 메시지를 수집하면 "준비 완료" 상태로 전환.
3. **Commit**: 준비 완료된 복제본들이 Commit 메시지를 교환하고 2f+1개를 받으면 실행(Execute).

리더가 Byzantine하면 **View Change** 프로토콜로 새 리더를 선출합니다.

## 직접 구현으로 이해하는 PBFT

### 1. PBFT 메시지 타입과 상태 기계 (Python)

```python
from enum import Enum
from dataclasses import dataclass
from typing import Dict, List, Optional
import hashlib
import json

class MessageType(Enum):
    REQUEST = "REQUEST"
    PRE_PREPARE = "PRE_PREPARE"
    PREPARE = "PREPARE"
    COMMIT = "COMMIT"
    REPLY = "REPLY"
    VIEW_CHANGE = "VIEW_CHANGE"

@dataclass
class PBFTMessage:
    msg_type: MessageType
    view: int           # 현재 뷰 번호 (리더 교체 시 증가)
    sequence: int       # 요청 순서 번호
    digest: str         # 요청 메시지의 해시
    node_id: int        # 발신 노드
    payload: dict       # 실제 데이터

    def to_json(self) -> str:
        return json.dumps({
            "type": self.msg_type.value,
            "view": self.view,
            "seq": self.sequence,
            "digest": self.digest,
            "from": self.node_id,
            "payload": self.payload
        })


class PBFTState(Enum):
    PRE_PREPARED = "pre_prepared"
    PREPARED = "prepared"
    COMMITTED = "committed"
    EXECUTED = "executed"


class PBFTReplica:
    """
    단순화된 PBFT 복제본 (단일 노드 시뮬레이션용)
    실제 구현에서는 네트워크 통신 레이어가 필요합니다.
    """
    def __init__(self, node_id: int, total_nodes: int):
        self.node_id = node_id
        self.n = total_nodes
        self.f = (total_nodes - 1) // 3   # 허용 Byzantine 노드 수
        self.view = 0
        self.sequence = 0

        # 메시지 로그: (view, seq) → 수집된 메시지들
        self.prepare_log: Dict[tuple, List[PBFTMessage]] = {}
        self.commit_log: Dict[tuple, List[PBFTMessage]] = {}
        self.state_log: Dict[tuple, PBFTState] = {}

        # 실행된 요청 로그
        self.executed: Dict[int, str] = {}

    @property
    def primary(self) -> int:
        return self.view % self.n

    def is_primary(self) -> bool:
        return self.node_id == self.primary

    def _digest(self, data: dict) -> str:
        return hashlib.sha256(json.dumps(data, sort_keys=True).encode()).hexdigest()[:16]

    def on_client_request(self, operation: str) -> Optional[PBFTMessage]:
        """리더(Primary)만 Pre-Prepare 생성"""
        if not self.is_primary():
            print(f"  Node {self.node_id}: Not primary, forwarding to primary {self.primary}")
            return None

        self.sequence += 1
        payload = {"op": operation, "ts": self.sequence}
        digest = self._digest(payload)

        msg = PBFTMessage(
            msg_type=MessageType.PRE_PREPARE,
            view=self.view,
            sequence=self.sequence,
            digest=digest,
            node_id=self.node_id,
            payload=payload
        )
        print(f"  [Primary {self.node_id}] PRE-PREPARE seq={self.sequence} op='{operation}'")
        return msg

    def on_pre_prepare(self, msg: PBFTMessage) -> Optional[PBFTMessage]:
        """복제본이 Pre-Prepare를 받으면 검증 후 Prepare 발송"""
        key = (msg.view, msg.sequence)

        # 검증: 올바른 뷰, 올바른 primary에서 왔는지 확인
        if msg.view != self.view:
            print(f"  Node {self.node_id}: Wrong view ({msg.view} != {self.view}), rejected")
            return None

        if msg.node_id != self.primary:
            print(f"  Node {self.node_id}: Pre-prepare not from primary, rejected")
            return None

        self.state_log[key] = PBFTState.PRE_PREPARED

        prepare = PBFTMessage(
            msg_type=MessageType.PREPARE,
            view=self.view,
            sequence=msg.sequence,
            digest=msg.digest,
            node_id=self.node_id,
            payload={}
        )
        print(f"  [Node {self.node_id}] PREPARE seq={msg.sequence}")
        return prepare

    def on_prepare(self, msg: PBFTMessage) -> Optional[PBFTMessage]:
        """2f+1개의 Prepare 수집 → Prepared 상태 → Commit 발송"""
        key = (msg.view, msg.sequence)
        if key not in self.prepare_log:
            self.prepare_log[key] = []
        self.prepare_log[key].append(msg)

        # 준비 완료 조건: Pre-prepared 상태 + 2f Prepare (자신 포함 2f+1)
        quorum = 2 * self.f  # 자신 제외 2f개
        if (self.state_log.get(key) == PBFTState.PRE_PREPARED and
                len(self.prepare_log[key]) >= quorum):

            self.state_log[key] = PBFTState.PREPARED
            commit = PBFTMessage(
                msg_type=MessageType.COMMIT,
                view=self.view,
                sequence=msg.sequence,
                digest=msg.digest,
                node_id=self.node_id,
                payload={}
            )
            print(f"  [Node {self.node_id}] COMMIT seq={msg.sequence} (quorum={quorum+1})")
            return commit
        return None

    def on_commit(self, msg: PBFTMessage) -> bool:
        """2f+1개의 Commit 수집 → 실행"""
        key = (msg.view, msg.sequence)
        if key not in self.commit_log:
            self.commit_log[key] = []
        self.commit_log[key].append(msg)

        quorum = 2 * self.f + 1  # f+1개의 정직한 노드 보장
        if (self.state_log.get(key) == PBFTState.PREPARED and
                len(self.commit_log[key]) >= quorum):

            self.state_log[key] = PBFTState.COMMITTED
            self.executed[msg.sequence] = msg.digest
            print(f"  [Node {self.node_id}] EXECUTED seq={msg.sequence} ✓")
            return True
        return False


# PBFT 시뮬레이션 (N=4, f=1 → 1개의 Byzantine 노드 허용)
def simulate_pbft():
    N = 4  # 3*1+1 = 4개 노드 → 1개 Byzantine 허용
    nodes = [PBFTReplica(i, N) for i in range(N)]
    primary = nodes[0]  # view=0, primary=node0

    print("=== PBFT 시뮬레이션 (N=4, f=1) ===\n")
    print("1. 클라이언트 요청: 'transfer(A→B, 100)'")

    # Phase 1: Pre-Prepare
    pre_prepare = primary.on_client_request("transfer(A→B, 100)")

    # Phase 2: Prepare (Byzantine node 1개 제외한 나머지 3개 응답)
    print("\n2. Prepare 단계:")
    prepares = []
    for node in nodes[1:]:  # 복제본들이 Prepare 발송
        prep = node.on_pre_prepare(pre_prepare)
        if prep:
            prepares.append(prep)

    # 각 노드가 다른 노드의 Prepare 수집
    print("\n3. Commit 단계:")
    commits = []
    for node in nodes:
        for prep in prepares:
            if prep.node_id != node.node_id:
                commit = node.on_prepare(prep)
                if commit:
                    commits.append(commit)
                    break

    # Phase 3: Commit 수집 → 실행
    print("\n4. 실행:")
    for node in nodes:
        for commit in commits:
            if node.on_commit(commit):
                break

simulate_pbft()
```

### 2. Byzantine 노드 감지 패턴

```python
from collections import Counter

class ByzantineDetector:
    """
    실제 시스템에서 Byzantine 노드를 감지하는 투표 기반 방어
    """
    def __init__(self, total: int, threshold_ratio: float = 2/3):
        self.total = total
        self.threshold = int(total * threshold_ratio)
        self.votes: Dict[str, List[int]] = {}  # value → [node_ids who voted]

    def vote(self, node_id: int, value: str):
        if value not in self.votes:
            self.votes[value] = []
        if node_id not in self.votes[value]:
            self.votes[value].append(node_id)

    def get_consensus(self) -> Optional[str]:
        """
        2/3 이상의 노드가 동의한 값을 반환.
        이 조건이 만족되면 Byzantine 노드들이 조작할 수 없음.
        """
        for value, voters in self.votes.items():
            if len(voters) >= self.threshold:
                return value
        return None  # 아직 합의 도달 못함

    def detect_byzantine_nodes(self) -> List[int]:
        """합의된 값과 다른 값을 보낸 노드 = 잠재적 Byzantine"""
        consensus = self.get_consensus()
        if not consensus:
            return []
        suspects = []
        for value, voters in self.votes.items():
            if value != consensus:
                suspects.extend(voters)
        return suspects


# Byzantine 감지 시뮬레이션
detector = ByzantineDetector(total=7, threshold_ratio=2/3)  # 최소 5개 동의 필요

# 정상 노드 5개는 "COMMIT" 투표
for i in range(5):
    detector.vote(i, "COMMIT")

# Byzantine 노드 2개는 "ABORT" 투표 (또는 서로 다른 값)
detector.vote(5, "ABORT")
detector.vote(6, "ABORT")

print("합의 결과:", detector.get_consensus())  # COMMIT
print("의심 노드:", detector.detect_byzantine_nodes())  # [5, 6]
```

## PBFT의 한계와 현대적 대안

**PBFT의 약점:**
- **O(n²) 메시지 복잡도**: 각 단계마다 모든 노드가 모든 노드에 메시지를 보내므로 노드 수가 늘면 급격히 느려집니다. 100개 노드 → 10,000개 메시지.
- **리더 교체 비용**: View Change 프로토콜이 복잡하고 느립니다.

**현대적 BFT 알고리즘들:**

| 알고리즘 | 메시지 복잡도 | 특징 |
|---|---|---|
| PBFT (1999) | O(n²) | 최초 실용적 BFT |
| Tendermint (2014) | O(n²) | 블록체인 BFT, 리더 순환 |
| HotStuff (2018) | O(n) | Facebook의 Diem 채택, 선형 복잡도 |
| SBFT | O(n) | 임계 서명으로 집계 |

## 주의사항과 실전 팁

**1. BFT가 필요한가?** BFT는 CFT(Crash Fault Tolerance)보다 훨씬 비쌉니다. 신뢰할 수 있는 내부망(데이터센터)에서는 Raft나 Paxos로 충분합니다. BFT는 외부 참여자가 있는 블록체인, 퍼블릭 컨소시엄, 보안이 중요한 항공/금융 시스템에 적합합니다.

**2. 3f+1 규칙 위반 주의:** 비용을 아끼려고 노드를 줄이면 안전성이 무너집니다. f=1 허용에 N=3 노드를 쓰면 (3*1+1=4 필요) 단 하나의 Byzantine 노드로도 합의가 불가능해집니다.

**3. 네트워크 파티션과 BFT:** 비잔틴 장애와 네트워크 파티션이 동시에 발생하면 더 복잡합니다. 현실 시스템에서는 타임아웃과 View Change를 신중하게 설계하세요.

**4. 암호학적 서명 사용:** 메시지에 디지털 서명을 추가하면 Byzantine 노드가 다른 노드를 사칭(spoofing)하는 것을 방지할 수 있습니다. PBFT 구현 시 반드시 적용하세요.

**5. 실제 구현은 라이브러리를 사용하세요:** Hyperledger Fabric(PBFT 기반), Tendermint Core, Diem의 HotStuff 구현 등 검증된 라이브러리를 활용하고, 직접 구현은 충분한 형식 검증(formal verification) 없이는 위험합니다.

## 참고 자료
- [Wikipedia: Byzantine fault](https://en.wikipedia.org/wiki/Byzantine_fault)
- [PBFT Lecture Slides — NUS CS6234](https://www.comp.nus.edu.sg/~rahul/allfiles/cs6234-16-pbft.pdf)
- [On the Practicality of 'Practical' Byzantine Fault Tolerance (arXiv)](https://arxiv.org/pdf/1110.4854)
- [Implementing PBFT in Blockchain (Medium)](https://medium.com/coinmonks/implementing-pbft-in-blockchain-12368c6c9548)
