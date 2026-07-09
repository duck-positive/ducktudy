---
layout: post
title: "비잔틴 장애 허용(BFT)과 PBFT 완전 정복: 악의적 노드가 있어도 합의하는 방법"
date: 2026-07-09
categories: [cs, computer-science]
tags: [distributed-systems, consensus, byzantine-fault-tolerance, pbft, fault-tolerance]
---

분산 시스템에서 일부 노드가 고장나거나 악의적으로 동작하더라도 전체 시스템이 올바른 결과에 합의할 수 있을까? 이 질문에 답하는 것이 바로 **비잔틴 장애 허용(Byzantine Fault Tolerance, BFT)** 이론이다. 본 글에서는 비잔틴 장군 문제부터 실제로 구현 가능한 PBFT(Practical Byzantine Fault Tolerance) 알고리즘까지 핵심 원리와 구현 방법을 깊이 있게 살펴본다.

---

## 비잔틴 장군 문제란?

1982년 Leslie Lamport, Robert Shostak, Marshall Pease가 제안한 **비잔틴 장군 문제(Byzantine Generals Problem)** 는 분산 합의의 한계를 수학적으로 정의한 고전 문제다.

여러 장군이 각기 다른 위치에서 도시를 포위하고 있다. 공격이 성공하려면 모든 장군이 **동시에** 공격하거나 **모두** 후퇴해야 한다. 문제는 일부 장군이 반역자일 수 있다는 점이다. 반역 장군은 일부에게는 "공격"이라고, 다른 이에게는 "후퇴"라고 전령을 보낼 수 있다.

이 문제는 분산 컴퓨팅에서 정확히 동일하게 나타난다:
- **노드(Node)** = 장군
- **메시지(Message)** = 전령
- **비잔틴 결함(Byzantine Fault)** = 거짓 정보를 보내는 반역 장군

> **핵심 정리**: `n`개의 노드 중 `f`개의 비잔틴 결함 노드가 있을 때, 합의를 보장하려면 **`n ≥ 3f + 1`** 이어야 한다.

즉, 1개의 악성 노드를 허용하려면 최소 4개의 노드가 필요하다. 2개를 허용하려면 7개, 이런 식으로 요구하는 노드 수가 빠르게 증가한다.

### 비잔틴 결함 vs. 크래시 결함

일반적인 **크래시 결함(Crash Fault)** 은 노드가 단순히 응답하지 않는 경우다. Raft나 Paxos 같은 알고리즘은 크래시 결함만 처리하며 `n ≥ 2f + 1`만 있으면 된다. 비잔틴 결함은 훨씬 더 강력한 적대적 조건으로, **거짓 메시지를 능동적으로 생성하는 악성 행위자**까지 가정한다.

| 속성 | 크래시 결함 | 비잔틴 결함 |
|------|-----------|-----------|
| 실패 유형 | 응답 없음 | 잘못된 응답 |
| 최소 노드 수 | 2f + 1 | 3f + 1 |
| 대표 알고리즘 | Raft, Paxos | PBFT, Tendermint |
| 성능 오버헤드 | 낮음 | 높음 (O(n²) 메시지) |

---

## PBFT 알고리즘의 등장

1999년 Miguel Castro와 Barbara Liskov가 발표한 **PBFT(Practical Byzantine Fault Tolerance)** 는 처음으로 실용적인 수준에서 비잔틴 장애를 허용하는 합의 알고리즘을 제시했다. 이전 BFT 알고리즘들이 이론에만 머물렀던 것과 달리, PBFT는 실제 분산 시스템에서 수백 밀리초의 레이턴시로 동작할 수 있었다.

### PBFT의 3-Phase 프로토콜

PBFT는 **Primary(리더)**와 **Backup(팔로워)** 노드로 구성된다. 클라이언트 요청이 들어오면 다음 세 단계를 거쳐 합의에 도달한다.

```
Client → Primary: REQUEST
Primary → All: PRE-PREPARE(v, n, d)
    모든 노드 → All: PREPARE(v, n, d, i)
    모든 노드 → All: COMMIT(v, n, d, i)
    모든 노드 → Client: REPLY
```

**Phase 1 — Pre-Prepare**: Primary가 클라이언트 요청을 받아 순서 번호 `n`을 부여하고, 모든 Backup에게 브로드캐스트한다. Backup은 메시지의 시퀀스 번호가 유효한지, 다이제스트가 일치하는지 확인한다.

**Phase 2 — Prepare**: Pre-Prepare를 받은 각 노드는 자신이 동의한다는 PREPARE 메시지를 **모든 다른 노드에게** 전송한다. 노드는 `2f`개의 유효한 PREPARE 메시지를 받으면 **prepared 상태**가 된다.

**Phase 3 — Commit**: prepared 상태의 노드는 COMMIT 메시지를 브로드캐스트한다. `2f + 1`개의 COMMIT 메시지를 받으면 요청을 실행하고 클라이언트에게 응답한다.

---

## 코드 예제 1: Python으로 PBFT 핵심 로직 구현

다음은 PBFT의 3-Phase 프로토콜 핵심을 Python으로 구현한 예제다. 실제 네트워크 통신은 생략하고 상태 머신 로직에 집중한다.

```python
import hashlib
import json
from enum import Enum
from collections import defaultdict

class Phase(Enum):
    IDLE = "idle"
    PRE_PREPARED = "pre_prepared"
    PREPARED = "prepared"
    COMMITTED = "committed"

class PBFTNode:
    def __init__(self, node_id: int, total_nodes: int):
        self.node_id = node_id
        self.n = total_nodes
        self.f = (total_nodes - 1) // 3  # 허용 가능한 비잔틴 노드 수
        self.view = 0
        self.sequence = 0
        self.phase = Phase.IDLE
        
        # 메시지 로그: (view, seq) → set of node_ids
        self.prepare_log = defaultdict(set)
        self.commit_log = defaultdict(set)
        self.executed = set()
        
    def digest(self, request: dict) -> str:
        serialized = json.dumps(request, sort_keys=True)
        return hashlib.sha256(serialized.encode()).hexdigest()
    
    def is_primary(self) -> bool:
        return self.node_id == self.view % self.n
    
    # --- Phase 1: Pre-Prepare ---
    def handle_request(self, request: dict) -> dict | None:
        """Primary만 실행: 클라이언트 요청을 Pre-Prepare 메시지로 변환"""
        if not self.is_primary():
            return None
        
        self.sequence += 1
        d = self.digest(request)
        msg = {
            "type": "PRE-PREPARE",
            "view": self.view,
            "seq": self.sequence,
            "digest": d,
            "request": request,
        }
        self.phase = Phase.PRE_PREPARED
        print(f"[Node {self.node_id}] PRE-PREPARE: seq={self.sequence}, digest={d[:8]}...")
        return msg
    
    # --- Phase 2: Prepare ---
    def handle_pre_prepare(self, msg: dict) -> dict | None:
        """Backup 노드: Pre-Prepare 검증 후 Prepare 메시지 생성"""
        v, n, d = msg["view"], msg["seq"], msg["digest"]
        
        # 검증: 뷰 번호, 다이제스트 일치 여부
        if v != self.view:
            return None
        if self.digest(msg["request"]) != d:
            print(f"[Node {self.node_id}] WARN: digest mismatch!")
            return None
        
        self.phase = Phase.PRE_PREPARED
        prepare_msg = {
            "type": "PREPARE",
            "view": v,
            "seq": n,
            "digest": d,
            "node_id": self.node_id,
        }
        print(f"[Node {self.node_id}] PREPARE: seq={n}")
        return prepare_msg
    
    def handle_prepare(self, msg: dict) -> bool:
        """Prepare 메시지 수신 및 quorum 확인"""
        key = (msg["view"], msg["seq"])
        self.prepare_log[key].add(msg["node_id"])
        
        # 2f 개의 Prepare = prepared 상태 (자신 포함 2f+1)
        if len(self.prepare_log[key]) >= 2 * self.f:
            self.phase = Phase.PREPARED
            print(f"[Node {self.node_id}] PREPARED: seq={msg['seq']}, "
                  f"votes={len(self.prepare_log[key])}")
            return True
        return False
    
    # --- Phase 3: Commit ---
    def create_commit(self, view: int, seq: int, digest: str) -> dict:
        return {
            "type": "COMMIT",
            "view": view,
            "seq": seq,
            "digest": digest,
            "node_id": self.node_id,
        }
    
    def handle_commit(self, msg: dict) -> bool:
        """Commit 메시지 수신 및 quorum 확인"""
        key = (msg["view"], msg["seq"])
        self.commit_log[key].add(msg["node_id"])
        
        # 2f+1 개의 Commit = committed 상태
        if len(self.commit_log[key]) >= 2 * self.f + 1:
            if key not in self.executed:
                self.executed.add(key)
                self.phase = Phase.COMMITTED
                print(f"[Node {self.node_id}] COMMITTED & EXECUTED: seq={msg['seq']}")
                return True
        return False


# --- 시뮬레이션 ---
def simulate_pbft(n_nodes=4, byzantine_node_id=3):
    nodes = [PBFTNode(i, n_nodes) for i in range(n_nodes)]
    primary = nodes[0]  # view=0, primary=node 0
    
    request = {"operation": "SET", "key": "x", "value": 42, "timestamp": 1720500000}
    
    print(f"=== PBFT 시뮬레이션 (노드 수: {n_nodes}, 비잔틴: node {byzantine_node_id}) ===\n")
    
    # Phase 1: Pre-Prepare
    pre_prepare = primary.handle_request(request)
    
    # Phase 2: Backup들이 Prepare
    prepare_msgs = []
    for node in nodes[1:]:
        if node.node_id == byzantine_node_id:
            # 비잔틴 노드: 잘못된 다이제스트로 응답
            bad_msg = dict(pre_prepare)
            bad_msg["digest"] = "deadbeef" * 8
            result = node.handle_pre_prepare(bad_msg)
        else:
            result = node.handle_pre_prepare(pre_prepare)
        if result:
            prepare_msgs.append(result)
    
    # 모든 노드가 Prepare 메시지 처리
    print()
    commit_msgs = []
    for node in nodes:
        for pmsg in prepare_msgs:
            if node.handle_prepare(pmsg):
                # Prepared → Commit 생성
                cmsg = node.create_commit(
                    pmsg["view"], pmsg["seq"], pmsg["digest"]
                )
                commit_msgs.append(cmsg)
                break
    
    # Phase 3: Commit
    print()
    for node in nodes:
        for cmsg in commit_msgs:
            if node.handle_commit(cmsg):
                break

simulate_pbft(n_nodes=4, byzantine_node_id=3)
```

실행하면 비잔틴 노드(node 3)의 잘못된 다이제스트는 검증에서 거부되고, 나머지 정직한 노드들이 합의를 완성하는 과정을 확인할 수 있다.

---

## 코드 예제 2: View Change 프로토콜 — Primary 교체

PBFT에서 Primary가 악의적으로 동작하거나 타임아웃이 발생하면 **View Change** 를 통해 새 Primary를 선출한다.

```python
import time
from threading import Timer

class PBFTViewChangeManager:
    """PBFT View Change 프로토콜 핵심 로직"""
    
    def __init__(self, node_id: int, n: int, timeout_sec: float = 2.0):
        self.node_id = node_id
        self.n = n
        self.f = (n - 1) // 3
        self.view = 0
        self.timeout_sec = timeout_sec
        self.view_change_log = defaultdict(set)  # new_view → set of node_ids
        self._timer: Timer | None = None
        
    def current_primary(self) -> int:
        return self.view % self.n
    
    def start_request_timer(self, seq: int):
        """요청 처리 타임아웃 타이머 시작"""
        def on_timeout():
            print(f"[Node {self.node_id}] TIMEOUT for seq={seq}, "
                  f"initiating view change from view {self.view}")
            self.send_view_change()
        
        self._timer = Timer(self.timeout_sec, on_timeout)
        self._timer.start()
    
    def cancel_timer(self):
        if self._timer:
            self._timer.cancel()
    
    def send_view_change(self) -> dict:
        """VIEW-CHANGE 메시지 생성 및 브로드캐스트"""
        new_view = self.view + 1
        msg = {
            "type": "VIEW-CHANGE",
            "new_view": new_view,
            "node_id": self.node_id,
            # 실제 구현에서는 검증용 체크포인트 증명 포함
            "last_stable_checkpoint": self._get_checkpoint(),
            "prepared_proofs": [],
        }
        print(f"[Node {self.node_id}] VIEW-CHANGE → view {new_view}")
        return msg
    
    def handle_view_change(self, msg: dict) -> dict | None:
        """VIEW-CHANGE 메시지 수신 처리"""
        new_view = msg["new_view"]
        self.view_change_log[new_view].add(msg["node_id"])
        
        # f+1개 이상의 VIEW-CHANGE를 받으면 자신도 view change에 합류
        if len(self.view_change_log[new_view]) == self.f + 1:
            if self.view < new_view:
                print(f"[Node {self.node_id}] Joining view change to {new_view}")
                return self.send_view_change()
        
        # 2f+1개 이상이면 새 Primary가 NEW-VIEW 메시지 생성
        if len(self.view_change_log[new_view]) >= 2 * self.f + 1:
            if self.node_id == new_view % self.n:
                return self._create_new_view_message(new_view)
        
        return None
    
    def handle_new_view(self, msg: dict):
        """NEW-VIEW 메시지 처리 → 뷰 전환 완료"""
        new_view = msg["new_view"]
        # 실제 구현에서는 VIEW-CHANGE 증명들을 검증
        self.view = new_view
        print(f"[Node {self.node_id}] Switched to view {self.view}, "
              f"new primary: node {self.current_primary()}")
    
    def _create_new_view_message(self, new_view: int) -> dict:
        print(f"[Node {self.node_id}] I am new primary for view {new_view}, "
              f"broadcasting NEW-VIEW")
        return {
            "type": "NEW-VIEW",
            "new_view": new_view,
            "primary": self.node_id,
            "view_change_proofs": list(self.view_change_log[new_view]),
        }
    
    def _get_checkpoint(self) -> int:
        return 0  # 단순화


# 시뮬레이션: Primary(node 0)가 비잔틴 장애 → View Change
print("=== View Change 시뮬레이션 ===")
managers = [PBFTViewChangeManager(i, n=4) for i in range(4)]

# node 1, 2, 3이 타임아웃 → View Change 시작
vc_msgs = [managers[i].send_view_change() for i in [1, 2, 3]]

# 각 노드가 VIEW-CHANGE 메시지를 처리
new_view_msg = None
for manager in managers:
    for vc in vc_msgs:
        result = manager.handle_view_change(vc)
        if result and result.get("type") == "NEW-VIEW":
            new_view_msg = result

# 모든 노드가 NEW-VIEW 메시지 처리
if new_view_msg:
    for manager in managers:
        manager.handle_new_view(new_view_msg)
```

---

## PBFT의 복잡도와 한계

### 메시지 복잡도 O(n²)

PBFT의 가장 큰 단점은 Prepare 단계에서 각 노드가 모든 노드에게 메시지를 보내야 하므로 메시지 수가 **O(n²)** 이라는 점이다. 노드가 100개라면 Prepare 메시지만 약 10,000개가 필요하다.

```
메시지 수 ≈ 2n² + 5n  (n = 전체 노드 수)

n=4:  약 48개
n=10: 약 250개
n=100: 약 20,500개
n=1000: 약 2,005,000개  ← 사실상 불가능
```

이 때문에 PBFT는 **소규모 클러스터(보통 10~30 노드)** 에서만 실용적이다.

### PBFT 이후 개선 알고리즘

| 알고리즘 | 개선 포인트 | 메시지 복잡도 |
|---------|-----------|-----------|
| PBFT | 기준선 | O(n²) |
| Tendermint | 라운드 기반, 블록체인 친화 | O(n²) |
| HotStuff | 선형 메시지, 파이프라인화 | O(n) |
| Streamlet | 간단한 체인 기반 | O(n²) |

특히 **HotStuff** (Facebook이 Libra/Diem에서 채택)는 Prepare/Pre-Commit/Commit의 3-phase와 임계 서명(Threshold Signature)을 활용해 O(n) 메시지로 BFT를 구현해 큰 주목을 받았다.

---

## 주의사항과 실전 팁

**1. 비잔틴 결함은 크래시 결함보다 훨씬 비싸다**
Raft를 쓸 수 있는 환경에서 PBFT를 선택하지 말 것. 악의적 행위자가 없는 내부 시스템이라면 Raft로 충분하다. PBFT는 블록체인, 금융 결제, 군사 시스템 같이 **적대적 환경** 이 명시된 경우에만 도입하라.

**2. 3f+1 노드 수를 반드시 확보하라**
f=1을 허용하려면 4개 노드가 필요하다. 4개 노드에서 노드 2개가 크래시되면 `n=2 < 3f+1=4`이 되어 PBFT가 멈춘다. 크래시 결함도 고려한다면 여유분을 더 확보해야 한다.

**3. 클라이언트 인증 필수**
클라이언트 요청에 MAC 또는 디지털 서명을 첨부해야 한다. Primary가 클라이언트 요청을 위변조하는 것을 방지하기 위해 Backup 노드들이 원본 요청의 다이제스트를 직접 검증한다.

**4. 체크포인트로 로그를 정리하라**
매 K번째 시퀀스마다 체크포인트를 찍어 이전 메시지 로그를 가비지 컬렉션하지 않으면 메모리 사용량이 무한히 증가한다. Castro & Liskov는 K=100을 권장했다.

**5. 현실적인 배포 방식**
순수 PBFT를 직접 구현하기보다 검증된 라이브러리를 활용하라:
- **Hyperledger Fabric**: PBFT 기반의 BFT-SMaRt 라이브러리 지원
- **Tendermint**: Go 구현, Cosmos SDK의 합의 엔진
- **HotStuff (LibraBFT)**: Facebook이 Diem 프로젝트에서 사용

---

비잔틴 장애 허용은 분산 시스템 보안성의 최전선에 있다. Raft와 Paxos가 "신뢰할 수 있는 노드들이 때로 죽을 수 있다"는 가정에서 출발한다면, PBFT는 "어떤 노드도 믿지 않아도 된다"는 근본적으로 다른 세계관을 갖는다. 블록체인의 확산과 함께 BFT 계열 알고리즘의 중요성은 더욱 높아지고 있다.

## 참고 자료
- [Bytepawn - PBFT Algorithm Explained](https://bytepawn.com/practical-byzantine-fault-tolerance.html)
- [Byzantine Fault Tolerance: BFT Concepts and PBFT Algorithm](https://developers-heaven.net/blog/byzantine-fault-tolerance-bft-concepts-and-algorithms-practical-bft-pbft/)
- [NCBI: Grouped Multilayer PBFT Consensus Algorithm](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC10649370/)
