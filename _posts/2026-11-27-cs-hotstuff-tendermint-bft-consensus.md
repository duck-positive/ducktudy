---
layout: post
title: "HotStuff와 Tendermint: 선형 메시지 복잡도를 달성한 현대 BFT 합의 알고리즘"
date: 2026-11-27
categories: [cs, computer-science]
tags: [distributed-systems, consensus, bft, hotstuff, tendermint, blockchain, paxos]
---

## 개념 설명

**Byzantine Fault Tolerant(BFT) 합의**는 분산 시스템에서 일부 노드가 악의적으로 행동(메시지 조작, 거짓 응답, 침묵)하더라도 나머지 정직한 노드들이 올바른 값에 합의할 수 있도록 보장하는 프로토콜입니다.

**PBFT(Practical Byzantine Fault Tolerance)**는 1999년 Castro와 Liskov가 제안한 최초의 실용적 BFT 프로토콜로, 3f+1개 노드 중 최대 f개의 비잔틴 노드를 허용합니다. 그러나 O(n²) 메시지 복잡도로 인해 20~30개 이상의 노드로 확장이 어렵습니다.

```
PBFT 통신:
  PREPARE 단계: 리더 → 모든 복제자 (n-1)
                각 복제자 → 모든 복제자 (n-1)² → O(n²)

HotStuff 통신:
  각 단계: 복제자 → 리더  (팬인, n-1)
           리더 → 복제자  (팬아웃, n-1, 집계 서명)
  총 O(n) per 단계
```

**HotStuff**는 2018년 VMware Research의 Maofan Yin, Dahlia Malkhi 등이 제안한 프로토콜로, **선형 메시지 복잡도(O(n))**와 **낙관적 반응성(Optimistic Responsiveness)**을 동시에 만족하는 첫 번째 BFT 프로토콜입니다.

**Tendermint**는 블록체인(Cosmos 에코시스템)에서 사용되는 BFT 합의 프로토콜로, 비슷한 2단계 잠금 구조를 가지지만 리더 교체 시 최대 네트워크 지연(Δ)을 기다려야 하므로 낙관적 반응성이 없습니다.

---

## 왜 필요한가

기존 BFT 프로토콜의 한계와 각 프로토콜의 비교:

| 프로토콜 | 메시지 복잡도 | 낙관적 반응성 | 뷰 변경 복잡도 | 채택 사례 |
|---------|------------|--------------|--------------|---------|
| PBFT    | O(n²)      | 없음          | O(n²)        | HyperLedger Fabric |
| Tendermint | O(n²)   | 없음          | O(n)         | Cosmos, Binance Chain |
| HotStuff | O(n)      | 있음          | O(n)         | Diem/Libra, DiemBFT |
| Jolteon (HotStuff 변형) | O(n) | 있음 | O(n) | 연구 단계 |

**낙관적 반응성(Optimistic Responsiveness)**: 네트워크가 동기적으로 작동할 때, 리더가 최신 잠금 상태를 알기 위해 고정 타임아웃(Δ)을 기다리지 않고, 실제 메시지가 도착하는 즉시 진행할 수 있는 성질입니다.

**HotStuff의 세 가지 핵심 혁신:**

1. **선형 통신 (Linearity)**: BLS 집계 서명(Aggregate Signature)을 활용해 쿼럼의 모든 서명을 하나로 묶어 O(n) 통신을 달성합니다.
2. **낙관적 반응성 (Optimistic Responsiveness)**: 네트워크 지연 상한 Δ 없이 실제 네트워크 속도로 진행합니다.
3. **파이프라인화 (Chaining)**: Chained HotStuff는 단계들을 파이프라이닝하여 각 블록에 별도의 3단계를 두지 않고 "Three-Chain Rule"로 커밋합니다.

HotStuff는 Meta(前 Facebook)의 Diem/Libra 블록체인의 핵심 합의 엔진으로 채택되었고, DiemBFT, Jolteon, Fast-HotStuff 등 다양한 변형이 파생되었습니다.

---

## 실제 구현 예제

### 예제 1: HotStuff의 핵심 데이터 구조와 3단계 프로토콜 (Python)

```python
from __future__ import annotations
from dataclasses import dataclass, field
from typing import Optional, List, Dict
from hashlib import sha256
import json


@dataclass
class Block:
    parent_hash: str
    payload: str
    view: int

    @property
    def hash(self) -> str:
        data = json.dumps(
            {"parent": self.parent_hash, "payload": self.payload, "view": self.view},
            sort_keys=True
        )
        return sha256(data.encode()).hexdigest()[:16]

    def __str__(self):
        return f"Block(view={self.view}, hash={self.hash})"


@dataclass
class QuorumCertificate:
    """
    쿼럼 인증서(QC): n-f개 이상 노드의 BLS 집계 서명을 묶은 증거.
    실제 구현에서는 단일 집계 서명 1개로 O(1) 크기를 유지.
    """
    view: int
    block_hash: str
    signatures: List[str]  # 실제로는 BLS 집계 서명 1개

    def __str__(self):
        return f"QC(view={self.view}, block={self.block_hash})"


@dataclass
class VoteMessage:
    phase: str           # "PREPARE" | "PRE-COMMIT" | "COMMIT"
    view: int
    block_hash: str
    node_id: int
    signature: str


class HotStuffReplica:
    """
    HotStuff 복제자(Replica) - 리더 포함 모든 노드가 이 역할 수행
    4단계: PREPARE → PRE-COMMIT → COMMIT → DECIDE
    """

    def __init__(self, node_id: int, n: int, f: int):
        self.node_id = node_id
        self.n = n
        self.f = f
        self.view = 1

        # HotStuff의 두 가지 핵심 상태
        self.prepare_qc: Optional[QuorumCertificate] = None   # highQC: 가장 최신 Prepare QC
        self.locked_qc: Optional[QuorumCertificate] = None    # lockedQC: 안전 잠금

        # 커밋된 블록
        self.committed: List[Block] = []

        # 로컬 로그
        self.block_store: Dict[str, Block] = {"genesis": Block("", "GENESIS", 0)}

    @property
    def threshold(self) -> int:
        """쿼럼 임계값 = n - f = 2f+1"""
        return self.n - self.f

    def is_safe_node(self, b: Block, qc: QuorumCertificate) -> bool:
        """
        안전 규칙(Safety Rule):
        b가 locked_qc의 블록을 연장하거나,
        qc.view > locked_qc.view이면 안전 (Liveness Rule)
        """
        if self.locked_qc is None:
            return True
        # Liveness: 더 최신 QC가 있으면 허용
        if qc.view > self.locked_qc.view:
            return True
        # Safety: b가 locked 블록의 후손이면 허용
        return self._extends(b, self.locked_qc.block_hash)

    def _extends(self, b: Block, ancestor_hash: str) -> bool:
        """b가 ancestor_hash 블록의 자손인지 확인"""
        current = b
        visited = set()
        while current.parent_hash and current.parent_hash not in visited:
            if current.parent_hash == ancestor_hash:
                return True
            visited.add(current.parent_hash)
            current = self.block_store.get(current.parent_hash)
            if current is None:
                break
        return False

    # ─── 리더 로직 ───────────────────────────────────────────────

    def leader_on_view_start(self, payload: str) -> tuple[Block, QuorumCertificate]:
        """
        PREPARE 단계 시작:
        1. highQC(가장 높은 view의 Prepare QC)를 선택
        2. highQC의 블록을 부모로 하는 새 블록 제안
        """
        high_qc = self.prepare_qc or QuorumCertificate(view=0, block_hash="genesis", signatures=[])
        new_block = Block(
            parent_hash=high_qc.block_hash,
            payload=payload,
            view=self.view,
        )
        self.block_store[new_block.hash] = new_block
        print(f"[리더 {self.node_id}] PREPARE 제안: {new_block}, highQC={high_qc}")
        return new_block, high_qc

    def leader_make_qc(self, votes: List[VoteMessage]) -> Optional[QuorumCertificate]:
        """threshold 이상 투표가 모이면 QC 생성"""
        if len(votes) < self.threshold:
            return None
        return QuorumCertificate(
            view=votes[0].view,
            block_hash=votes[0].block_hash,
            signatures=[v.signature for v in votes[:self.threshold]]
        )

    # ─── 복제자 로직 ──────────────────────────────────────────────

    def replica_on_prepare(self, b: Block, high_qc: QuorumCertificate) -> Optional[VoteMessage]:
        """
        PREPARE 메시지 수신:
        안전 규칙을 만족하면 PREPARE 투표
        """
        if not self.is_safe_node(b, high_qc):
            print(f"[복제자 {self.node_id}] PREPARE 거부: 안전 규칙 위반")
            return None
        self.block_store[b.hash] = b
        # prepare_qc 업데이트 (highQC)
        if self.prepare_qc is None or high_qc.view > self.prepare_qc.view:
            self.prepare_qc = high_qc
        sig = f"sig_{self.node_id}_PREPARE_{b.hash}"
        print(f"[복제자 {self.node_id}] PREPARE 투표: {b}")
        return VoteMessage("PREPARE", self.view, b.hash, self.node_id, sig)

    def replica_on_prepare_qc(self, qc: QuorumCertificate) -> VoteMessage:
        """
        PREPARE QC 수신 → PRE-COMMIT 투표
        prepare_qc 업데이트 (알려진 가장 최신 QC)
        """
        if self.prepare_qc is None or qc.view > self.prepare_qc.view:
            self.prepare_qc = qc
        sig = f"sig_{self.node_id}_PRECOMMIT_{qc.block_hash}"
        print(f"[복제자 {self.node_id}] PRE-COMMIT 투표: QC={qc}")
        return VoteMessage("PRE-COMMIT", self.view, qc.block_hash, self.node_id, sig)

    def replica_on_precommit_qc(self, qc: QuorumCertificate) -> VoteMessage:
        """
        PRE-COMMIT QC 수신 → COMMIT 투표
        locked_qc 업데이트 (핵심: 이후 뷰에서 이 블록을 보호)
        """
        if self.locked_qc is None or qc.view > self.locked_qc.view:
            self.locked_qc = qc
            print(f"[복제자 {self.node_id}] 잠금 업데이트: {qc}")
        sig = f"sig_{self.node_id}_COMMIT_{qc.block_hash}"
        return VoteMessage("COMMIT", self.view, qc.block_hash, self.node_id, sig)

    def replica_on_commit_qc(self, qc: QuorumCertificate):
        """
        COMMIT QC 수신 → 블록 커밋 (상태 머신에 적용)
        """
        block = self.block_store.get(qc.block_hash)
        if block:
            self.committed.append(block)
            print(f"[복제자 {self.node_id}] 커밋 완료: {block}")
        self.view += 1


def simulate_hotstuff_round():
    """4개 노드 (n=4, f=1), 1라운드 HotStuff 시뮬레이션"""
    n, f = 4, 1
    nodes = [HotStuffReplica(i, n, f) for i in range(n)]
    leader = nodes[0]  # 뷰 1의 리더

    print("=" * 60)
    print("HotStuff 합의 시뮬레이션 (n=4, f=1, threshold=3)")
    print("=" * 60)

    # === PREPARE 단계 ===
    print("\n--- PREPARE 단계 ---")
    block, high_qc = leader.leader_on_view_start("트랜잭션: Alice → Bob 50 COIN")

    prepare_votes = []
    for node in nodes:
        vote = node.replica_on_prepare(block, high_qc)
        if vote:
            prepare_votes.append(vote)

    prepare_qc = leader.leader_make_qc(prepare_votes)
    print(f"\nPREPARE QC 생성: {prepare_qc}\n")

    # === PRE-COMMIT 단계 ===
    print("--- PRE-COMMIT 단계 ---")
    precommit_votes = [node.replica_on_prepare_qc(prepare_qc) for node in nodes]
    precommit_qc = leader.leader_make_qc(precommit_votes)
    print(f"\nPRE-COMMIT QC 생성: {precommit_qc}\n")

    # === COMMIT 단계 ===
    print("--- COMMIT 단계 ---")
    commit_votes = [node.replica_on_precommit_qc(precommit_qc) for node in nodes]
    commit_qc = leader.leader_make_qc(commit_votes)
    print(f"\nCOMMIT QC 생성: {commit_qc}\n")

    # === DECIDE (커밋) ===
    print("--- DECIDE 단계 ---")
    for node in nodes:
        node.replica_on_commit_qc(commit_qc)

    print(f"\n커밋된 블록: {nodes[0].committed}")


simulate_hotstuff_round()
```

---

### 예제 2: Chained HotStuff - Three-Chain Rule 구현

Chained HotStuff는 각 블록의 투표가 다음 블록의 단계를 겸합니다. 명시적 단계 구분 없이 세 개의 연속 블록이 직접 연결된 체인을 형성하면 가장 오래된 블록을 커밋합니다(Three-Chain Rule).

```python
@dataclass
class ChainedHotStuffNode:
    """
    Chained HotStuff: 파이프라이닝으로 처리량 극대화

    커밋 조건 (Three-Chain Rule):
        B_k ← B_{k+1} ← B_{k+2} 세 블록이 직접 연결되고 각각 QC가 있으면
        B_k 커밋

    비유: B_{k+2}의 QC = B_k의 Commit QC + B_{k+1}의 PreCommit QC + B_{k+2}의 Prepare QC 역할
    → 별도의 3라운드 없이 블록마다 1라운드로 전체 파이프라인 가동
    """
    node_id: int
    n: int
    f: int

    block_store: Dict[str, Block] = field(default_factory=lambda: {
        "genesis": Block("", "GENESIS", 0)
    })
    qc_store: Dict[str, QuorumCertificate] = field(default_factory=dict)

    b_lock: Optional[Block] = None    # 잠금 블록 (Pre-Commit에서 설정)
    b_exec: Optional[Block] = None    # 마지막 커밋 블록
    high_qc: Optional[QuorumCertificate] = None  # 가장 최신 QC

    @property
    def threshold(self):
        return self.n - self.f

    def on_receive_proposal(self, b: Block, qc: QuorumCertificate) -> Optional[VoteMessage]:
        """
        새 블록 b와 그 부모의 QC를 수신:
        1. 안전 규칙 확인
        2. Three-Chain 체크 (커밋 조건)
        3. 투표 발행
        """
        self.block_store[b.hash] = b
        self.qc_store[qc.block_hash] = qc

        # highQC 업데이트
        if self.high_qc is None or qc.view > self.high_qc.view:
            self.high_qc = qc

        # b1 = qc로 증명된 블록 (부모)
        b1 = self.block_store.get(qc.block_hash)
        # b2 = b1의 부모
        b2 = self.block_store.get(b1.parent_hash) if b1 else None
        b2_qc = self.qc_store.get(b1.parent_hash) if b1 else None
        # b3 = b2의 부모
        b3 = self.block_store.get(b2.parent_hash) if b2 else None
        b3_qc = self.qc_store.get(b2.parent_hash) if b2 else None

        # Liveness Rule: b2가 있으면 b_lock 업데이트
        if b2 and (self.b_lock is None or b2.view > self.b_lock.view):
            self.b_lock = b2
            print(f"[노드 {self.node_id}] 잠금 업데이트: {b2}")

        # Three-Chain Rule: b3 → b2 → b1 → b 직접 연결이면 b3 커밋
        if b3 and b2 and b1 and b2_qc and b3_qc:
            if (b2.parent_hash == b3.hash and  # b2가 b3의 직접 자식
                    b1.parent_hash == b2.hash and  # b1이 b2의 직접 자식
                    b.parent_hash == b1.hash):     # b가 b1의 직접 자식
                self._commit_chain(b3)

        # 안전 규칙 확인 후 투표
        if self._safe_node(b, qc):
            sig = f"sig_{self.node_id}_{b.hash[:8]}"
            return VoteMessage("VOTE", b.view, b.hash, self.node_id, sig)
        return None

    def _safe_node(self, b: Block, qc: QuorumCertificate) -> bool:
        """
        안전 규칙:
        - b_lock이 없으면 항상 안전
        - qc가 b_lock보다 최신이면 안전 (Liveness)
        - b가 b_lock의 자손이면 안전 (Safety)
        """
        if self.b_lock is None:
            return True
        if qc.view > self.b_lock.view:
            return True
        # b가 b_lock의 자손인지 확인
        current = b
        while current and current.parent_hash:
            if current.parent_hash == self.b_lock.hash:
                return True
            current = self.block_store.get(current.parent_hash)
        return False

    def _commit_chain(self, b: Block):
        """b와 그 조상들을 순서대로 커밋"""
        chain_to_commit = []
        current = b
        while current and (self.b_exec is None or current.view > self.b_exec.view):
            chain_to_commit.append(current)
            current = self.block_store.get(current.parent_hash)

        for blk in reversed(chain_to_commit):
            print(f"[노드 {self.node_id}] 커밋: {blk}")
            self.b_exec = blk
```

---

## 주의사항/팁

**1. BLS 집계 서명이 선형 통신의 비밀**

HotStuff가 O(n) 통신을 달성하는 핵심은 BLS(Boneh-Lynn-Shacham) 집계 서명입니다. 각 단계에서 복제자들은 리더에게만 투표를 보내고(팬인, n-1 메시지), 리더는 n-f개 서명을 하나의 집계 서명(QC)으로 합쳐 모두에게 브로드캐스트합니다(팬아웃, n-1 메시지). PBFT는 각 복제자가 다른 모든 복제자에게 직접 메시지를 보내므로 O(n²)입니다.

**2. Tendermint와의 결정적 차이점**

Tendermint는 각 뷰에서 `PREVOTE → PRECOMMIT` 2단계를 사용합니다. 리더 교체 시, 새 리더는 최소 Δ(최대 네트워크 지연)를 기다려야 충분한 `PREVOTE`를 수집할 수 있습니다. 반면 HotStuff의 새 리더는 복제자들에게 최신 `prepare_qc`를 요청해 즉시 진행할 수 있어 낙관적 반응성을 달성합니다.

**3. 안전성(Safety) vs 활성도(Liveness) 트레이드오프**

HotStuff는 `locked_qc`를 통해 안전성을 보장합니다. 복제자는 `locked_qc`보다 오래된 뷰의 블록에는 절대 투표하지 않습니다. 활성도를 위해서는 `qc.view > locked_qc.view`인 경우(더 최신 QC를 가진 리더)에는 잠금을 해제합니다. 이 두 규칙이 BFT의 핵심입니다.

**4. 뷰 동기화(View Synchronization)의 중요성**

HotStuff는 뷰 동기화 프로토콜이 별도로 필요합니다. 모든 정직한 노드가 같은 뷰(view)에 충분히 오래 머물러야 진행이 보장됩니다. 실제 구현(DiemBFT 등)에서는 `PaceMaker` 모듈이 이를 담당합니다. 타임아웃 메시지가 모이면 모든 노드가 다음 뷰로 이동합니다.

**5. 실제 채택 현황**

- **Diem/Libra (Meta)**: HotStuff 직접 구현, DiemBFT v4 → LibraBFT
- **Cosmos SDK**: Tendermint Core (별도 구현), CometBFT로 발전
- **Ethereum**: Gasper(Casper FFG + LMD-GHOST)는 HotStuff 아이디어를 일부 차용
- **엔터프라이즈**: IBFT2(PBFT 변형), Clique(PoA)

**6. BFT vs CFT의 선택**

BFT 합의는 CFT(Crash Fault Tolerance) 합의(Paxos, Raft)보다 복잡하고 메시지 오버헤드가 큽니다. 신뢰할 수 없는 환경(퍼블릭 블록체인, 다기업 컨소시엄)이 아니라면 Raft로 충분합니다. 프라이빗 클러스터에서 BFT를 선택하는 것은 과도한 복잡도를 야기합니다.

## 참고 자료
- [HotStuff: BFT Consensus in the Lens of Blockchain (arXiv:1803.05069)](https://arxiv.org/abs/1803.05069)
- [HotStuff at VMware Research](https://research.vmware.com/publications/hotstuff-bft-consensus-with-linearity-and-responsiveness)
- [HotStuff Three-Chain Rules — Dahlia Malkhi의 블로그](https://dahliamalkhi.github.io/posts/2019/08/hotstuff-three-chain-rules/)
- [Tendermint: Byzantine Fault Tolerance in the Age of Blockchains](https://docs.tendermint.com/v0.35/introduction/what-is-tendermint.html)
