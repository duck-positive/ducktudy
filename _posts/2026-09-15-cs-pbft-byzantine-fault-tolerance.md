---
layout: post
title: "PBFT 완전 정복: 비잔틴 장애 허용 합의 알고리즘과 블록체인의 수학적 토대"
date: 2026-09-15
categories: [cs, computer-science]
tags: [distributed-systems, consensus, pbft, byzantine-fault-tolerance, blockchain, cryptography]
---

분산 시스템에서 합의(Consensus)란 여러 노드가 하나의 값에 동의하는 과정이다. Raft나 Paxos 같은 전통적 합의 알고리즘은 노드가 **결함(Crash Fault)**을 가질 수 있다고 가정하지만, 노드가 **악의적으로 거짓 정보를 보내는 상황(Byzantine Fault)**은 고려하지 않는다. 전자는 노드가 멈추거나 응답하지 않는 경우이고, 후자는 노드가 의도적으로 잘못된 데이터를 전파하는 경우다.

**PBFT(Practical Byzantine Fault Tolerance)**는 1999년 Miguel Castro와 Barbara Liskov가 OSDI에 발표한 알고리즘으로, 비잔틴 노드가 존재해도 안전하게 합의에 도달하는 최초의 **실용적** 방법을 제시했다. 이전 BFT 알고리즘들은 이론적으로 옳지만 성능이 너무 나빠 실제 사용이 불가능했다.

## 비잔틴 장군 문제

PBFT를 이해하려면 먼저 **비잔틴 장군 문제(Byzantine Generals Problem)**를 알아야 한다. 1982년 Lamport, Shostak, Pease가 제시한 이 문제는 다음과 같다:

여러 장군이 적의 성을 공격하거나 후퇴하기로 결정해야 한다. 장군들은 서로 메시지를 보내 소통하지만, 일부 장군은 반역자(비잔틴 노드)여서 다른 장군들에게 서로 다른 메시지를 보낼 수 있다. 모든 충성스러운 장군이 같은 행동을 취하려면 어떤 프로토콜이 필요한가?

핵심 정리: **최소 3f+1개의 노드**가 있어야 **f개의 비잔틴 노드**를 허용하면서 합의에 도달할 수 있다. 예를 들어 비잔틴 노드 1개를 허용하려면 최소 4개 노드가 필요하다.

이 한계는 정보 이론적으로 증명된 것이다. 3개 노드에서 1개가 비잔틴이면, 2개의 정직한 노드가 각자 다른 값을 받을 때 어느 것이 진짜인지 판단할 방법이 없다.

## PBFT의 3단계 프로토콜

PBFT는 **Primary(리더) 노드**와 여러 **Replica(복제본) 노드**로 구성된 클라이언트-서버 모델에서 작동한다. 정상 케이스 프로토콜은 세 단계로 이루어진다:

```
Client   Primary    Replica1   Replica2   Replica3
  │         │           │          │          │
  │─REQUEST─▶          │          │          │
  │         │──PRE-PREPARE──────▶  │          │
  │         │──PRE-PREPARE──────────────────▶ │
  │         │           │──PREPARE──▶         │
  │         │           │──PREPARE──────────▶ │
  │         │◀──PREPARE─│          │          │
  │         │           │◀──────PREPARE───────│
  │         │           │──────COMMIT──▶      │
  │         │           │──────COMMIT──────▶  │
  │         │◀──────COMMIT──────────────────  │
  │◀REPLY───│◀REPLY─────│◀REPLY────│◀REPLY────│
```

### 1단계: Pre-prepare

Primary 노드가 클라이언트 요청 `m`을 받으면, 시퀀스 번호 `n`과 뷰 번호 `v`를 포함한 Pre-prepare 메시지를 모든 Replica에 전파한다:

```
<<PRE-PREPARE, v, n, d>, m>
```

여기서 `d`는 요청 m의 다이제스트(해시)다.

### 2단계: Prepare

각 Replica는 Pre-prepare를 받으면 유효성을 검사(시퀀스 번호 범위, 다이제스트 일치)한 후, 다른 모든 Replica에게 Prepare 메시지를 보낸다:

```
<PREPARE, v, n, d, i>
```

어떤 노드가 **같은 (v, n, d)에 대해 2f개 이상**의 유효한 Prepare 메시지를 받으면 **prepared 상태**가 된다.

### 3단계: Commit

prepared 상태의 노드는 Commit 메시지를 전파한다:

```
<COMMIT, v, n, d, i>
```

**2f+1개 이상**의 유효한 Commit 메시지를 받으면 요청을 실행하고 클라이언트에 응답한다.

### 안전성 보장 원리

**Safety**: 어떤 두 정직한 노드도 서로 다른 요청을 같은 시퀀스 번호에 실행하지 않는다. Commit phase에서 2f+1개의 메시지를 모아야 하므로, f개의 비잔틴 노드가 있어도 최소 f+1개의 정직한 노드가 동의한 것이 보장된다.

**Liveness**: Primary가 장애가 나면 **View Change** 프로토콜로 새 Primary를 선출한다.

## 코드 예제 1: PBFT 메시지 검증 로직 구현 (Python)

```python
import hashlib
import json
from collections import defaultdict
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class Message:
    msg_type: str      # PRE-PREPARE, PREPARE, COMMIT, REPLY
    view: int          # 뷰 번호 (현재 Primary 식별)
    seq_num: int       # 시퀀스 번호
    digest: str        # 요청 메시지의 SHA-256 해시
    node_id: int       # 발송 노드 ID
    request: Optional[dict] = None  # PRE-PREPARE에만 포함

def compute_digest(request: dict) -> str:
    """요청의 SHA-256 다이제스트 계산"""
    serialized = json.dumps(request, sort_keys=True).encode()
    return hashlib.sha256(serialized).hexdigest()


class PBFTNode:
    """간소화된 PBFT 노드 구현 (네트워크 레이어 생략)"""
    
    def __init__(self, node_id: int, total_nodes: int):
        self.node_id = node_id
        self.n = total_nodes
        self.f = (total_nodes - 1) // 3  # 허용 가능한 최대 비잔틴 노드 수
        self.view = 0
        self.seq_num = 0
        
        # 상태 저장소
        self.pre_prepares: dict = {}        # (view, seq) -> Message
        self.prepares: dict = defaultdict(set)   # (view, seq, digest) -> {node_ids}
        self.commits: dict = defaultdict(set)    # (view, seq, digest) -> {node_ids}
        self.prepared: set = set()          # prepared된 (view, seq) 집합
        self.executed: set = set()          # 실행된 seq_num 집합
        self.log: list = []                 # 실행된 요청 로그
        
        print(f"Node {node_id}: n={total_nodes}, f={self.f} (최대 {self.f}개 비잔틴 허용)")
    
    @property
    def is_primary(self) -> bool:
        return self.node_id == self.view % self.n
    
    def handle_pre_prepare(self, msg: Message) -> bool:
        """PRE-PREPARE 처리"""
        key = (msg.view, msg.seq_num)
        
        # 유효성 검사
        if msg.view != self.view:
            print(f"Node {self.node_id}: 뷰 불일치 {msg.view} != {self.view}")
            return False
        if key in self.pre_prepares:
            existing = self.pre_prepares[key]
            if existing.digest != msg.digest:
                print(f"Node {self.node_id}: 같은 (v,n)에 다른 다이제스트! 비잔틴 Primary 감지")
                return False
        if msg.request and compute_digest(msg.request) != msg.digest:
            print(f"Node {self.node_id}: 다이제스트 불일치")
            return False
        
        self.pre_prepares[key] = msg
        print(f"Node {self.node_id}: PRE-PREPARE 수락 (v={msg.view}, n={msg.seq_num})")
        return True
    
    def handle_prepare(self, msg: Message) -> bool:
        """PREPARE 처리 - 2f개 이상 모이면 prepared 상태"""
        key = (msg.view, msg.seq_num, msg.digest)
        self.prepares[key].add(msg.node_id)
        
        state_key = (msg.view, msg.seq_num)
        if (len(self.prepares[key]) >= 2 * self.f and
                state_key in self.pre_prepares and
                state_key not in self.prepared):
            self.prepared.add(state_key)
            print(f"Node {self.node_id}: PREPARED (v={msg.view}, n={msg.seq_num}) "
                  f"[{len(self.prepares[key])} prepares]")
            return True
        return False
    
    def handle_commit(self, msg: Message) -> bool:
        """COMMIT 처리 - 2f+1개 이상 모이면 실행"""
        key = (msg.view, msg.seq_num, msg.digest)
        self.commits[key].add(msg.node_id)
        
        if (len(self.commits[key]) >= 2 * self.f + 1 and
                msg.seq_num not in self.executed):
            # 이전 시퀀스들이 모두 실행됐는지 확인 (gap 없는 실행 보장)
            if all(i in self.executed for i in range(msg.seq_num)):
                self.executed.add(msg.seq_num)
                pp = self.pre_prepares.get((msg.view, msg.seq_num))
                if pp and pp.request:
                    self.log.append(pp.request)
                print(f"Node {self.node_id}: 요청 실행 (seq={msg.seq_num}) "
                      f"[{len(self.commits[key])} commits]")
                return True
        return False


def simulate_pbft():
    """7개 노드 클러스터에서 PBFT 시뮬레이션 (f=2)"""
    N = 7
    nodes = [PBFTNode(i, N) for i in range(N)]
    primary = nodes[0]
    
    request = {"operation": "SET", "key": "counter", "value": 42, "timestamp": 1000}
    digest = compute_digest(request)
    seq = 1
    view = 0
    
    print(f"\n=== 클라이언트 요청: {request} ===\n")
    
    # Phase 1: Primary가 PRE-PREPARE 브로드캐스트
    pp_msg = Message("PRE-PREPARE", view, seq, digest, primary.node_id, request)
    for node in nodes[1:]:  # Replica들에게 전송
        node.handle_pre_prepare(pp_msg)
    
    print()
    # Phase 2: 각 Replica가 PREPARE 브로드캐스트 (비잔틴 노드 2개 제외)
    honest_replicas = nodes[1:5]  # 정직한 Replica 4개
    for sender in honest_replicas:
        prep_msg = Message("PREPARE", view, seq, digest, sender.node_id)
        for receiver in nodes:
            receiver.handle_prepare(prep_msg)
    
    print()
    # Phase 3: COMMIT 브로드캐스트
    for sender in nodes[:5]:  # 5개 노드가 commit (2f+1 = 5)
        commit_msg = Message("COMMIT", view, seq, digest, sender.node_id)
        for receiver in nodes:
            receiver.handle_commit(commit_msg)
    
    print(f"\n=== 결과: {sum(1 for n in nodes if seq in n.executed)}/{N} 노드가 실행 완료 ===")


simulate_pbft()
```

## 코드 예제 2: View Change (리더 교체) 트리거 감지

```python
import time
import threading
from enum import Enum

class NodeStatus(Enum):
    NORMAL = "normal"
    VIEW_CHANGE = "view_change"

class ViewChangeDetector:
    """Primary 타임아웃 감지 및 View Change 트리거"""
    
    def __init__(self, node_id: int, n: int, timeout_sec: float = 5.0):
        self.node_id = node_id
        self.n = n
        self.view = 0
        self.status = NodeStatus.NORMAL
        self.timeout = timeout_sec
        self.last_primary_activity = time.time()
        self.view_change_votes: dict = defaultdict(set)  # new_view -> {node_ids}
        self.f = (n - 1) // 3
    
    def record_primary_activity(self):
        """Primary로부터 활동이 감지될 때 타이머 리셋"""
        self.last_primary_activity = time.time()
    
    def check_timeout(self) -> bool:
        """타임아웃 초과 여부 확인"""
        return (time.time() - self.last_primary_activity) > self.timeout
    
    def trigger_view_change(self) -> dict:
        """View Change 메시지 생성"""
        new_view = self.view + 1
        vc_msg = {
            "type": "VIEW-CHANGE",
            "new_view": new_view,
            "node_id": self.node_id,
            "prepared_proofs": [],  # 실제로는 prepared 상태의 증명 포함
            "timestamp": time.time()
        }
        self.status = NodeStatus.VIEW_CHANGE
        print(f"Node {self.node_id}: View Change 트리거! v={self.view} → v={new_view}")
        return vc_msg
    
    def receive_view_change(self, vc_msg: dict):
        """다른 노드의 View Change 메시지 수신"""
        new_view = vc_msg["new_view"]
        sender = vc_msg["node_id"]
        self.view_change_votes[new_view].add(sender)
        
        # f+1개 이상 받으면 자신도 View Change 트리거
        if (len(self.view_change_votes[new_view]) >= self.f + 1 and
                self.status == NodeStatus.NORMAL):
            print(f"Node {self.node_id}: f+1 View Change 수신, 동참")
            self.trigger_view_change()
        
        # 2f+1개 이상 받으면 새 Primary 확정
        if len(self.view_change_votes[new_view]) >= 2 * self.f + 1:
            self.view = new_view
            new_primary = new_view % self.n
            self.status = NodeStatus.NORMAL
            print(f"Node {self.node_id}: 새 Primary = Node {new_primary} (view={new_view})")
    
    def run_timeout_monitor(self):
        """백그라운드 타임아웃 모니터"""
        while True:
            if self.status == NodeStatus.NORMAL and self.check_timeout():
                vc_msg = self.trigger_view_change()
                # 실제로는 다른 노드들에게 vc_msg 브로드캐스트
            time.sleep(0.5)


# 시뮬레이션: Primary 장애 → View Change
print("=== View Change 시뮬레이션 ===")
detectors = [ViewChangeDetector(i, 7, timeout_sec=2.0) for i in range(7)]

# Primary(Node 0)가 응답 없음 → 2초 후 타임아웃
print("Primary(Node 0) 장애 발생 시뮬레이션...")
time.sleep(0.1)  # 짧은 대기

# 다른 노드들이 View Change 시작 (f+1 = 3개)
for i in range(1, 4):
    vc = detectors[i].trigger_view_change()
    for j in range(7):
        if j != i:
            detectors[j].receive_view_change(vc)

print(f"\n새 Primary: Node {detectors[1].view % 7} (view={detectors[1].view})")
```

## PBFT의 복잡도와 한계

### 메시지 복잡도 O(n²)

PBFT의 가장 큰 약점은 **통신 복잡도**다. Prepare 단계에서 각 노드는 다른 모든 노드에게 메시지를 보내므로, n개 노드에서 메시지 수는 O(n²)이다. 10개 노드면 약 100개, 100개 노드면 약 10,000개의 메시지가 필요하다. 이 때문에 PBFT는 수십 개 이하의 소규모 클러스터에서만 실용적이다.

### 현대의 BFT 알고리즘

PBFT의 한계를 개선한 알고리즘들이 등장했다:

- **Tendermint (2014)**: 블록체인 최적화, 라운드-로빈 리더 선출
- **HotStuff (2018)**: Meta(Facebook)가 개발한 선형 BFT, O(n) 메시지 복잡도. Diem(Libra) 블록체인에 사용
- **Streamlet (2020)**: 3단계를 2단계로 줄인 간소화 BFT
- **PBFT vs Raft**: PBFT는 악의적 노드를 처리하지만 Raft는 비잔틴이 아닌 결함만 처리. 허가형 블록체인은 PBFT 계열, 공개 블록체인은 PoW/PoS

## 블록체인에서의 활용

허가형(Permissioned) 블록체인인 Hyperledger Fabric과 IBM Food Trust는 PBFT 기반 합의를 사용한다. 참여자가 알려져 있고 신뢰 경계가 명확한 금융·공급망 시스템에 적합하다.

반면 비트코인의 PoW나 이더리움의 PoS는 네크워크 참여자 전체를 알 수 없으므로, 비잔틴 내성을 **경제적 인센티브** 설계로 해결한다는 점에서 PBFT와 철학적으로 다르다.

## 정리

PBFT는 이론으로만 존재하던 비잔틴 장애 허용을 실용적으로 구현했다는 점에서 역사적 의의가 크다. 핵심 아이디어는 세 가지다:
1. 3f+1개 노드로 f개 비잔틴 노드를 허용
2. 2단계 투표(Prepare + Commit)로 Safety 보장
3. View Change로 Liveness(활성도) 보장

O(n²) 복잡도 한계는 HotStuff 같은 후속 알고리즘이 개선했지만, PBFT의 기본 구조—Pre-prepare, Prepare, Commit 3단계—는 여전히 현대 BFT 알고리즘의 설계 기반이 되고 있다.

## 참고 자료
- [Castro & Liskov: Practical Byzantine Fault Tolerance (OSDI 1999)](https://pmg.csail.mit.edu/papers/osdi99.pdf)
- [HotStuff: BFT Consensus with Linearity and Responsiveness (PODC 2019)](https://arxiv.org/abs/1803.05069)
- [Tendermint: Byzantine Fault Tolerance in the Age of Blockchains](https://arxiv.org/abs/1807.04938)
- [PBFT Enhanced: NCBIBio](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC10780467/)
