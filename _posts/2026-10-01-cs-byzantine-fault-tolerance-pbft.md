---
layout: post
title: "비잔틴 장애 허용(BFT)과 PBFT: 악의적 노드가 존재하는 분산 합의 프로토콜 완전 정복"
date: 2026-10-01
categories: [cs, computer-science]
tags: [byzantine-fault-tolerance, pbft, distributed-systems, consensus, blockchain, security]
---

## 비잔틴 장애 허용이란

분산 시스템에서 노드(컴퓨터)는 언제든 고장날 수 있다. 일반적인 분산 시스템이 다루는 **크래시 장애(crash fault)**는 노드가 그냥 멈추는 경우다. 이 경우 조용히 응답이 없어지므로 다른 노드들이 "저 노드는 죽었구나"라고 판단할 수 있다.

그러나 **비잔틴 장애(Byzantine Fault)**는 훨씬 심각하다. 노드가 단순히 멈추는 것이 아니라, **의도적으로 잘못된 메시지를 보내거나, 서로 다른 노드에게 서로 다른 메시지를 보내거나, 응답을 선택적으로 지연시키는 등 악의적인 행동**을 할 수 있다.

이 개념은 1982년 Lamport, Shostak, Pease가 발표한 논문 "The Byzantine Generals Problem"에서 유래했다.

### 비잔틴 장군 문제

비잔틴 제국의 장군들이 도시를 포위하고 있다. 일부 장군과 전령은 반역자일 수 있다. 충성스러운 장군들은 서로 메시지를 주고받아 **공격 또는 후퇴**라는 동일한 결정에 도달해야 한다. 반역자는 어떤 메시지든 보낼 수 있다.

**결론**: n개 노드 중 f개가 비잔틴이면, 합의에 도달하려면 `n ≥ 3f + 1`이 필요하다.

왜 3f+1인가?
- 최악의 경우 f개 비잔틴 노드가 모두 거짓말을 한다
- 또한 f개 크래시 노드가 응답하지 않을 수 있다
- 남은 n - 2f개 노드 중 정직한 노드가 과반수여야 한다: `(n - 2f) > f` → `n > 3f`

---

## 왜 BFT가 필요한가

### 현실에서의 비잔틴 장애

1. **악의적 공격자**: 해킹된 서버가 잘못된 데이터를 전송
2. **소프트웨어 버그**: 비결정론적 버그로 인해 노드가 잘못된 상태 계산
3. **하드웨어 오류**: 메모리 비트 플립, 전자파 간섭으로 인한 잘못된 계산
4. **블록체인 네트워크**: 합의에 참여하는 밸리데이터 노드 중 일부가 공격자

블록체인(Bitcoin, Ethereum 등)이 BFT를 중심으로 설계된 이유도 여기에 있다. 검열 저항성과 탈중앙화를 위해서는 악의적인 참여자가 있어도 시스템이 올바르게 동작해야 한다.

### CFT vs BFT

| 구분 | CFT(Crash Fault Tolerant) | BFT(Byzantine Fault Tolerant) |
|------|--------------------------|-------------------------------|
| 장애 종류 | 노드 중단만 | 임의의 악의적 동작 |
| 필요 노드 수 | `n ≥ 2f + 1` | `n ≥ 3f + 1` |
| 대표 알고리즘 | Raft, Paxos | PBFT, Tendermint, HotStuff |
| 메시지 복잡도 | O(n) | O(n²) |
| 적용 분야 | 기업 내부 분산 시스템 | 블록체인, 적대적 환경 |

---

## PBFT(Practical Byzantine Fault Tolerance) 알고리즘

Miguel Castro와 Barbara Liskov가 1999년 발표한 PBFT는 이전 BFT 알고리즘들이 너무 이론적이어서 실용 불가능했던 문제를 해결한 **최초의 실용적 BFT 프로토콜**이다.

### 시스템 모델

- **n개 노드**, 그 중 최대 **f개 비잔틴 노드**가 존재
- `n = 3f + 1` (예: f=1이면 n=4, f=2이면 n=7)
- **Primary(리더)**와 나머지 **Replica**로 구성
- **부분 동기 네트워크** 가정: 메시지가 유한 시간 내에 도달하지만 정확한 시간은 모름

### 4단계 프로토콜

```
[클라이언트]
     │
     │ ① REQUEST(op, t, c)
     ↓
[Primary(p)]  ← 리더 노드
     │
     │ ② PRE-PREPARE(v, n, d) + 원본 REQUEST
     ↓
[모든 Replica]
     │
     │ ③ PREPARE(v, n, d, i)  ← 서로에게 전파
     ↓
[2f+1개 PREPARE 수신 후]
     │
     │ ④ COMMIT(v, n, D(m), i) ← 서로에게 전파
     ↓
[2f+1개 COMMIT 수신 후]
     │
     │ ⑤ REPLY(v, t, c, i, r) → 클라이언트
     ↓
[클라이언트]
```

**용어 정리**:
- `v`: 뷰 번호(view number) — 현재 Primary가 누구인지 식별
- `n`: 시퀀스 번호(sequence number) — 요청의 순서
- `d`: 메시지 다이제스트 (해시)
- `i`: 노드 ID

---

## 단계별 상세 설명

### Phase 1: Pre-prepare (Primary → Replicas)

Primary는 클라이언트의 요청을 받아 시퀀스 번호를 부여하고 모든 replica에 전파한다:

```
PRE-PREPARE(v, n, d, m)
- v: 현재 뷰 번호
- n: Primary가 할당한 시퀀스 번호
- d: m의 SHA-256 해시
- m: 클라이언트의 원본 요청
```

### Phase 2: Prepare (Replicas 상호 전파)

각 replica는 PRE-PREPARE를 검증한 뒤 다른 모든 replica에 PREPARE 메시지를 전송한다:

```
PREPARE(v, n, d, i)
- 검증 조건:
  1. 메시지 서명 유효
  2. 뷰 번호 v가 현재 뷰와 일치
  3. 시퀀스 번호 n이 유효 범위 내
  4. d가 m의 실제 해시와 일치
```

replica가 **2f개의 유효한 PREPARE** (자신 포함 2f+1개)를 수집하면 "prepared" 상태가 된다.

### Phase 3: Commit (Replicas 상호 전파)

"prepared" 상태에 도달한 replica는 COMMIT을 전파한다:

```
COMMIT(v, n, D(m), i)
```

**2f+1개의 COMMIT**을 수집하면 "committed" 상태가 되며, 요청을 실행한다.

### Phase 4: Reply (Replicas → Client)

클라이언트는 **f+1개의 동일한 응답**을 받으면 결과를 수락한다. f+1이면 충분한 이유: f개 비잔틴 노드가 모두 같은 거짓 응답을 보내더라도, 나머지 정직한 노드 1개가 다른 응답을 보내므로 구별 가능하다.

---

## Python으로 PBFT 시뮬레이션

```python
import hashlib
import json
from collections import defaultdict
from enum import Enum
from typing import Optional

class Phase(Enum):
    IDLE      = "idle"
    PREPARED  = "prepared"
    COMMITTED = "committed"

def digest(message: dict) -> str:
    """메시지 해시"""
    return hashlib.sha256(json.dumps(message, sort_keys=True).encode()).hexdigest()[:8]

class PBFTNode:
    def __init__(self, node_id: int, total: int, byzantine: bool = False):
        self.id = node_id
        self.n = total          # 총 노드 수
        self.f = (total - 1) // 3   # 허용 비잔틴 수
        self.byzantine = byzantine
        self.view = 0
        self.sequence = 0
        self.state = {}         # 시퀀스 번호 → 실행된 요청
        self.prepares = defaultdict(set)   # (v,n) → {replica_id 집합}
        self.commits  = defaultdict(set)   # (v,n) → {replica_id 집합}
        self.phase    = defaultdict(lambda: Phase.IDLE)
        self.log = []

    @property
    def is_primary(self):
        return self.id == (self.view % self.n)

    def receive_request(self, request: dict):
        """Primary: 요청 수신 후 PRE-PREPARE 발행"""
        if not self.is_primary:
            return None
        self.sequence += 1
        pre_prepare = {
            "type": "PRE-PREPARE",
            "view": self.view,
            "seq": self.sequence,
            "digest": digest(request),
            "request": request,
            "from": self.id,
        }
        self.log.append(f"[Node {self.id}] Primary: PRE-PREPARE(v={self.view}, n={self.sequence})")
        return pre_prepare

    def receive_pre_prepare(self, msg: dict) -> Optional[dict]:
        """Replica: PRE-PREPARE 수신 후 검증 및 PREPARE 발행"""
        if self.byzantine:
            # 비잔틴 노드: 잘못된 digest로 응답
            return {
                "type": "PREPARE",
                "view": msg["view"],
                "seq": msg["seq"],
                "digest": "INVALID_DIGEST",
                "from": self.id,
            }

        v, n = msg["view"], msg["seq"]
        # 검증: 뷰와 시퀀스 번호 유효성
        if v != self.view:
            return None

        prepare = {
            "type": "PREPARE",
            "view": v,
            "seq": n,
            "digest": msg["digest"],
            "from": self.id,
        }
        self.log.append(f"[Node {self.id}] PREPARE(v={v}, n={n}, d={msg['digest'][:4]}...)")
        return prepare

    def receive_prepare(self, msg: dict) -> Optional[dict]:
        """PREPARE 수신 및 quorum 체크"""
        if self.byzantine:
            return None  # 비잔틴: COMMIT 거부
            
        v, n, d = msg["view"], msg["seq"], msg["digest"]
        key = (v, n)
        
        if d != "INVALID_DIGEST":  # 정상 메시지만 카운트
            self.prepares[key].add(msg["from"])

        # 2f개 PREPARE 수집 시 prepared 상태
        if (len(self.prepares[key]) >= 2 * self.f and 
                self.phase[key] == Phase.IDLE):
            self.phase[key] = Phase.PREPARED
            commit = {
                "type": "COMMIT",
                "view": v,
                "seq": n,
                "digest": d,
                "from": self.id,
            }
            self.log.append(f"[Node {self.id}] → PREPARED! 발행: COMMIT(v={v}, n={n})")
            return commit
        return None

    def receive_commit(self, msg: dict) -> Optional[str]:
        """COMMIT 수신 및 실행"""
        if self.byzantine:
            return None
            
        v, n, d = msg["view"], msg["seq"], msg["digest"]
        key = (v, n)
        self.commits[key].add(msg["from"])

        # 2f+1개 COMMIT 수집 시 실행
        if (len(self.commits[key]) >= 2 * self.f + 1 and
                self.phase[key] == Phase.PREPARED):
            self.phase[key] = Phase.COMMITTED
            result = f"실행완료: seq={n}"
            self.state[n] = result
            self.log.append(f"[Node {self.id}] ✓ COMMITTED! seq={n} 실행")
            return result
        return None


def simulate_pbft(n_nodes: int, n_byzantine: int, request: str):
    print(f"\n{'='*60}")
    print(f"PBFT 시뮬레이션: 노드={n_nodes}, 비잔틴={n_byzantine}")
    print(f"요청: '{request}'")
    print(f"이론적 f: {(n_nodes-1)//3}, 실제 비잔틴: {n_byzantine}")
    print(f"{'='*60}")
    
    # 노드 생성 (마지막 n_byzantine개는 비잔틴)
    nodes = [
        PBFTNode(i, n_nodes, byzantine=(i >= n_nodes - n_byzantine))
        for i in range(n_nodes)
    ]
    
    req = {"op": request, "timestamp": 42, "client": "C1"}
    primary = nodes[0]  # 뷰 0에서 노드 0이 Primary
    
    # Phase 1: Primary → PRE-PREPARE
    pre_prepare = primary.receive_request(req)
    
    # Phase 2: 모든 노드 → PREPARE
    all_prepares = []
    for node in nodes:
        if not node.is_primary:
            prepare = node.receive_pre_prepare(pre_prepare)
            if prepare:
                all_prepares.append(prepare)
    
    # Phase 3: 모든 PREPARE 전파 → COMMIT
    all_commits = []
    for prepare_msg in all_prepares:
        for node in nodes:
            commit = node.receive_prepare(prepare_msg)
            if commit:
                all_commits.append(commit)
    
    # Phase 4: 모든 COMMIT 전파 → 실행
    results = []
    for commit_msg in all_commits:
        for node in nodes:
            result = node.receive_commit(commit_msg)
            if result:
                results.append((node.id, result))
    
    # 결과 출력
    print("\n[실행 로그]")
    for node in nodes:
        label = "(비잔틴)" if node.byzantine else ""
        for log in node.log:
            print(f"  {log} {label}")
    
    print(f"\n[완료된 노드 수]: {len(set(nid for nid, _ in results))}")
    committed = [nid for nid, _ in results if not nodes[nid].byzantine]
    print(f"[정직한 노드 커밋 수]: {len(committed)}")
    print(f"합의 {'성공 ✓' if len(committed) >= n_nodes - n_byzantine else '실패 ✗'}")


# 정상 케이스: n=4, f=1 (비잔틴 1개 허용)
simulate_pbft(n_nodes=4, n_byzantine=1, request="SET x=10")

# 한계 케이스: n=4, f=1이지만 비잔틴 2개 → 합의 실패
simulate_pbft(n_nodes=4, n_byzantine=2, request="SET y=20")
```

---

## 뷰 체인지: Primary 장애 대응

Primary 자체가 비잔틴이거나 크래시한 경우, **뷰 체인지(View Change)** 프로토콜을 통해 새 Primary를 선출한다.

```python
import time

class ViewChangeProtocol:
    def __init__(self, node_id: int, total: int):
        self.id = node_id
        self.n = total
        self.f = (total - 1) // 3
        self.view = 0
        self.view_change_votes = defaultdict(set)
        self.timeout_threshold = 2.0  # 2초 타임아웃

    def detect_timeout(self, request_time: float) -> bool:
        return time.time() - request_time > self.timeout_threshold

    def send_view_change(self) -> dict:
        """Primary 교체 요청"""
        new_view = self.view + 1
        msg = {
            "type": "VIEW-CHANGE",
            "new_view": new_view,
            "last_stable_checkpoint": 0,
            "prepared_messages": [],  # 준비된 메시지들의 증거
            "from": self.id,
        }
        return msg

    def receive_view_change(self, msg: dict) -> bool:
        """2f+1개 VIEW-CHANGE 수신 시 뷰 전환"""
        new_view = msg["new_view"]
        self.view_change_votes[new_view].add(msg["from"])
        
        if len(self.view_change_votes[new_view]) >= 2 * self.f + 1:
            self.view = new_view
            new_primary = new_view % self.n
            print(f"[Node {self.id}] 뷰 {new_view}로 전환, 새 Primary: Node {new_primary}")
            return True
        return False
```

---

## 주의사항 및 팁

### 1. O(n²) 메시지 복잡도

PBFT의 가장 큰 단점은 n개 노드에서 각 합의 라운드마다 O(n²)개의 메시지가 필요하다는 것이다. n=100이면 라운드당 약 10,000개의 메시지가 발생한다. 이 때문에 PBFT는 **소규모 검증자 집합**(10~100개 노드)에서 효과적이다.

### 2. 최신 BFT 알고리즘들

| 알고리즘 | 메시지 복잡도 | 특징 |
|---------|------------|------|
| PBFT | O(n²) | 최초 실용적 BFT |
| Tendermint | O(n²) | Cosmos 블록체인 사용 |
| HotStuff | O(n) | Facebook의 LibraBFT 기반, 선형 복잡도 |
| BLS-PBFT | O(n) | BLS 서명으로 메시지 집계 |

### 3. BFT vs 작업 증명(PoW)

Bitcoin의 PoW는 경제적 인센티브로 비잔틴 장애를 방지한다. 공격 비용이 이익보다 크면 공격하지 않는다. 반면 BFT는 **즉시 최종성(immediate finality)**을 제공한다: 한번 커밋되면 되돌릴 수 없다.

### 4. 안전성(Safety) vs 활성(Liveness)

PBFT는 네트워크 파티션 시 **안전성을 우선**한다(CAP 정리에서 CP). 동기화가 복구될 때까지 진행을 멈추더라도 잘못된 값을 커밋하지 않는다.

---

## 정리

| 개념 | 내용 |
|------|------|
| 비잔틴 장애 | 노드가 악의적으로 임의의 동작을 하는 장애 |
| BFT 조건 | n ≥ 3f + 1 (f개 비잔틴 허용) |
| PBFT 단계 | PRE-PREPARE → PREPARE → COMMIT → REPLY |
| Quorum | PREPARE: 2f개, COMMIT: 2f+1개 |
| 뷰 체인지 | Primary 장애 시 새 Primary 선출 |
| 한계 | O(n²) 메시지 복잡도 → 소규모 클러스터 적합 |

비잔틴 장애 허용은 적대적 환경(블록체인, 군사 시스템, 항공 제어)에서 필수적인 기술이다. PBFT의 원리를 이해하면 Tendermint, HotStuff 등 현대 BFT 알고리즘이 어떤 문제를 어떻게 개선했는지도 자연스럽게 파악할 수 있다.

## 참고 자료
- [Byzantine fault - Wikipedia](https://en.wikipedia.org/wiki/Byzantine_fault)
- [Practical Byzantine Fault Tolerance - Wikipedia](https://en.wikipedia.org/wiki/Practical_Byzantine_fault_tolerance)
- [The Byzantine Generals Problem - Lamport et al. (ACM TOPLAS 1982)](https://dl.acm.org/doi/10.1145/357172.357176)
- [HotStuff: BFT Consensus with Linearity and Responsiveness](https://arxiv.org/abs/1803.05069)
