---
layout: post
title: "Paxos 합의 알고리즘 완전 정복: 분산 시스템의 철학자 식사 문제를 해결하는 법"
date: 2026-07-03
categories: [cs, computer-science]
tags: [paxos, distributed-systems, consensus, fault-tolerance, multi-paxos, lamport]
---

## Paxos 합의 알고리즘이란?

분산 시스템에서 가장 어려운 문제 중 하나는 **합의(Consensus)**입니다. 여러 노드가 하나의 값에 동의하도록 만들어야 하는데, 노드가 언제든 다운될 수 있고 네트워크는 메시지를 지연하거나 손실할 수 있습니다. 이 문제를 해결하기 위해 Leslie Lamport가 1989년에 고안하고 1998년에 논문으로 발표한 알고리즘이 바로 **Paxos**입니다.

Paxos는 단순히 "과반수가 동의하면 결정한다"는 원칙을 분산 환경에서 안전하게 구현합니다. Google의 Chubby 락 서비스, Apache Zookeeper의 ZAB 프로토콜, 그리고 수많은 분산 데이터베이스가 Paxos의 아이디어를 기반으로 만들어졌습니다.

## 왜 Paxos가 필요한가?

**FLP 불가능성 정리(FLP Impossibility)**에 따르면, 단 하나의 노드가 장애를 일으킬 수 있는 비동기 분산 시스템에서는 완전히 일관되고 항상 가용하며 장애를 허용하는 합의를 *결정론적으로* 달성할 수 없습니다.

하지만 현실에서는 "완벽한 보장" 대신 **확률적 종료**와 **안전성 보장**을 절충할 수 있습니다. Paxos는 이 절충을 현명하게 수행합니다:

- **Safety(안전성)**: 두 개의 다른 값이 동시에 결정되는 일은 절대 없습니다.
- **Liveness(활성성)**: 충분히 긴 시간이 주어지면 언젠가 합의에 도달합니다.
- **Fault Tolerance**: N개 중 floor(N/2)개의 노드가 실패해도 동작합니다.

단순한 "리더가 결정하면 된다"는 방식은 리더 자신이 실패했을 때 전체 시스템이 멈추는 문제가 있습니다. Paxos는 리더 없이도 안전하게 작동하며, 리더가 있을 때는 효율적으로 작동합니다.

## Paxos의 참여자와 역할

Paxos는 세 가지 역할을 정의합니다. 실제로 하나의 노드가 여러 역할을 동시에 수행할 수 있습니다.

- **Proposer(제안자)**: 합의할 값을 제안합니다. 클라이언트 요청을 받아 처리합니다.
- **Acceptor(수용자)**: 제안을 수락하거나 거부합니다. 과반수의 Acceptor가 수락해야 값이 결정됩니다.
- **Learner(학습자)**: 결정된 값을 학습하고 클라이언트에게 응답합니다.

## Single-Decree Paxos: 하나의 값에 합의하기

### Phase 1: Prepare

Proposer는 고유한 제안 번호 `n`을 선택하고, 과반수의 Acceptor에게 **Prepare(n)** 메시지를 보냅니다.

각 Acceptor는 다음을 검사합니다:
- `n`이 지금까지 본 어떤 Prepare 번호보다 크다면 → **Promise(n, v_prev)** 응답. 이전에 수락한 값 `v_prev`가 있다면 함께 전달.
- `n`이 이미 약속한 번호보다 작거나 같다면 → 응답 거부(Nack 또는 무시).

### Phase 2: Accept

Proposer가 과반수로부터 Promise를 받으면:
- Promise 중 **가장 큰 번호에 딸린 값**이 있다면 그 값을 사용.
- 그런 값이 없다면(모든 응답이 "이전 수락 없음") 원하는 값을 자유롭게 선택.
- **Accept(n, v)** 메시지를 과반수에게 전송.

Acceptor는 `n`이 자신이 약속한 번호 이상이면 값을 수락하고, Learner에게 알립니다.

```python
# Paxos 단순 구현 (교육용)
import threading
from typing import Optional, Tuple

class Acceptor:
    def __init__(self):
        self.promised_n: int = -1        # 약속한 최고 제안 번호
        self.accepted_n: int = -1        # 수락한 제안 번호
        self.accepted_v: Optional[str] = None  # 수락한 값
        self.lock = threading.Lock()

    def prepare(self, n: int) -> Optional[Tuple[int, Optional[str]]]:
        """Phase 1: Prepare 처리. Promise를 반환하거나 None(거부)."""
        with self.lock:
            if n > self.promised_n:
                self.promised_n = n
                # (이전에 수락한 번호, 이전에 수락한 값) 반환
                return (self.accepted_n, self.accepted_v)
            return None  # 거부

    def accept(self, n: int, v: str) -> bool:
        """Phase 2: Accept 처리."""
        with self.lock:
            if n >= self.promised_n:
                self.promised_n = n
                self.accepted_n = n
                self.accepted_v = v
                return True
            return False  # 거부


class Proposer:
    def __init__(self, node_id: int, acceptors: list):
        self.node_id = node_id
        self.acceptors = acceptors
        self.proposal_count = 0

    def _make_proposal_number(self) -> int:
        """단조 증가하는 고유 제안 번호 생성 (노드 ID를 인코딩)"""
        self.proposal_count += 1
        # 상위 비트에 카운터, 하위 비트에 노드 ID를 합쳐 고유성 보장
        return (self.proposal_count << 10) | self.node_id

    def propose(self, value: str) -> Optional[str]:
        n = self._make_proposal_number()
        quorum = len(self.acceptors) // 2 + 1

        # Phase 1: Prepare
        promises = []
        for acceptor in self.acceptors:
            result = acceptor.prepare(n)
            if result is not None:
                promises.append(result)

        if len(promises) < quorum:
            print(f"Phase 1 실패: 과반수({quorum}) 미달, {len(promises)}개만 응답")
            return None

        # 가장 최근에 수락된 값이 있다면 그것을 사용 (Safety 보장)
        highest_accepted = max(promises, key=lambda x: x[0])
        if highest_accepted[1] is not None:
            value = highest_accepted[1]
            print(f"이전에 수락된 값 발견: {value}, 이 값을 사용합니다")

        # Phase 2: Accept
        accepts = 0
        for acceptor in self.acceptors:
            if acceptor.accept(n, value):
                accepts += 1

        if accepts >= quorum:
            print(f"합의 성공! 결정된 값: '{value}' (제안 번호: {n})")
            return value
        else:
            print(f"Phase 2 실패: 과반수({quorum}) 미달, {accepts}개만 수락")
            return None


# 실행 예시
acceptors = [Acceptor() for _ in range(5)]  # 5개 Acceptor (3개 과반수)
proposer_a = Proposer(node_id=1, acceptors=acceptors)
proposer_b = Proposer(node_id=2, acceptors=acceptors)

result = proposer_a.propose("값-A")
print(f"최종 결과: {result}")
```

이 구현의 핵심은 Phase 1에서 Acceptor가 "더 큰 번호가 오면 이전 약속을 깨지 않겠다"고 약속하는 것입니다. 이 약속이 Safety를 보장합니다.

## Multi-Paxos: 연속된 값의 합의

Single-Decree Paxos는 하나의 값만 결정합니다. 데이터베이스의 로그처럼 **연속된 명령어들**에 합의하려면 Multi-Paxos가 필요합니다.

Multi-Paxos의 핵심 최적화는 **리더 선출**입니다. 한 Proposer가 리더로 선출되면 매번 Phase 1을 반복할 필요 없이 Phase 2만 실행합니다. RTT(Round Trip Time)가 2에서 1로 줄어들어 성능이 2배 향상됩니다.

```python
class MultiPaxosLog:
    """Multi-Paxos를 이용한 분산 로그 구현 (단순화)"""
    
    def __init__(self, node_id: int, peers: list):
        self.node_id = node_id
        self.peers = peers
        self.log: list = []           # 결정된 명령어 로그
        self.is_leader = False
        self.current_term = 0         # Raft의 term과 유사한 개념
        self.leader_id: Optional[int] = None

    def try_become_leader(self) -> bool:
        """Phase 1을 전체 로그 슬롯에 대해 한 번에 실행하여 리더가 됨"""
        new_term = self.current_term + 1
        quorum = len(self.peers) // 2 + 1
        promises_received = 0

        for peer in self.peers:
            # 실제로는 네트워크 RPC
            if peer.receive_prepare_leader(new_term, self.node_id):
                promises_received += 1

        if promises_received >= quorum:
            self.current_term = new_term
            self.is_leader = True
            self.leader_id = self.node_id
            print(f"노드 {self.node_id}가 Term {new_term}의 리더로 선출됨")
            return True
        return False

    def append(self, command: str) -> Optional[int]:
        """리더만 새 명령어를 제안할 수 있음"""
        if not self.is_leader:
            print(f"리더({self.leader_id})에게 요청을 전달하세요")
            return None

        log_index = len(self.log)
        quorum = len(self.peers) // 2 + 1
        accepts = 0

        # Phase 2만 실행 (리더 선출 시 Phase 1을 이미 완료했으므로)
        for peer in self.peers:
            if peer.receive_accept(self.current_term, log_index, command):
                accepts += 1

        if accepts >= quorum:
            self.log.append(command)
            print(f"[{log_index}] '{command}' 커밋 완료")
            return log_index
        return None

    def receive_prepare_leader(self, term: int, candidate_id: int) -> bool:
        """다른 노드의 리더 선출 Prepare 처리"""
        if term > self.current_term:
            self.current_term = term
            self.is_leader = False
            self.leader_id = candidate_id
            return True
        return False

    def receive_accept(self, term: int, log_index: int, command: str) -> bool:
        """다른 노드의 Accept 처리"""
        if term >= self.current_term:
            # 로그 슬롯에 값 기록
            while len(self.log) <= log_index:
                self.log.append(None)
            self.log[log_index] = command
            return True
        return False


# 3-노드 클러스터 시뮬레이션
nodes = [MultiPaxosLog(node_id=i, peers=[]) for i in range(3)]
# peers 설정 (자기 자신 제외)
for i, node in enumerate(nodes):
    node.peers = [n for j, n in enumerate(nodes) if j != i]

# 노드 0이 리더가 됨
nodes[0].try_become_leader()

# 리더를 통해 명령어 추가
nodes[0].append("SET x = 1")
nodes[0].append("SET y = 2")
nodes[0].append("DELETE z")

print(f"\n노드 1의 로그: {nodes[1].log}")
print(f"노드 2의 로그: {nodes[2].log}")
```

## Paxos의 라이브니스 문제와 해결책

Paxos는 이론적으로 **라이브락(Livelock)**에 빠질 수 있습니다. 두 Proposer가 서로의 Prepare를 무효화하며 무한히 경쟁하는 상황입니다.

```
Proposer A: Prepare(n=1) → 과반수 약속
Proposer B: Prepare(n=2) → 과반수 약속 (A의 약속 무효화)
Proposer A: Prepare(n=3) → 과반수 약속 (B의 약속 무효화)
... 무한 반복
```

해결책:
1. **리더 선출**: 한 번에 하나의 Proposer만 활성화되도록 제어합니다.
2. **랜덤 백오프**: 충돌 시 무작위 시간만큼 대기 후 재시도합니다.
3. **Raft 알고리즘**: 리더 선출을 프로토콜의 핵심으로 삼아 Paxos의 복잡성을 줄인 대안입니다.

## Paxos vs. Raft: 설계 철학의 차이

| 항목 | Paxos | Raft |
|------|-------|------|
| 복잡도 | 높음 (여러 변형 존재) | 낮음 (이해 가능성 우선) |
| 리더 | 선택적 (Multi-Paxos에서 권장) | 필수 |
| 로그 순서 | 유연함 (갭 허용) | 엄격함 (순차적) |
| 멤버십 변경 | 복잡 | 단계적 변경(Joint Consensus) |
| 실제 사용 | Chubby, Spanner | etcd, CockroachDB, TiKV |

Paxos가 더 일반적이고 이론적으로 강력하지만, 구현의 복잡성 때문에 현대 시스템은 Raft를 선호하는 경향이 있습니다.

## 실제 시스템에서의 Paxos

**Google Spanner**는 TrueTime API와 Paxos를 결합하여 전 세계에 분산된 데이터베이스에서 외부 일관성(External Consistency)을 달성합니다. 각 데이터 샤드마다 Paxos 그룹이 존재하며, 2PC(Two-Phase Commit)와 결합하여 분산 트랜잭션을 처리합니다.

**Apache Zookeeper**의 ZAB(Zookeeper Atomic Broadcast) 프로토콜은 Paxos와 유사한 원리로 동작하지만, 리더 에포크(epoch)를 명시적으로 관리하고 로그의 연속성을 강하게 보장합니다.

## 주의사항과 구현 팁

**네트워크 분리(Network Partition) 처리**: Paxos는 네트워크가 분리될 때 어떤 파티션도 과반수를 갖지 못하면 새로운 합의를 달성할 수 없습니다. 이는 Safety를 Availability보다 우선시하는 의도적인 설계입니다(CAP 정리의 CP 선택).

**제안 번호 지속성**: Proposer는 충돌 후 재시작할 때 이전에 사용한 제안 번호보다 큰 번호를 사용해야 합니다. 이를 위해 마지막 제안 번호를 디스크에 저장해야 합니다.

**멱등성(Idempotency)**: 메시지가 중복 전달될 수 있으므로 동일한 Prepare/Accept 메시지를 여러 번 받아도 안전하게 처리해야 합니다.

**배치 처리**: 실제 시스템에서는 여러 요청을 하나의 로그 엔트리로 묶어 처리하여(batching) 성능을 높입니다.

Paxos는 구현이 복잡하지만, 분산 합의의 수학적 기반을 깊이 이해하고 싶다면 반드시 공부해야 할 알고리즘입니다. Raft를 사용하든 ZAB를 사용하든, 그 밑에는 항상 Paxos의 철학이 담겨 있습니다.

## 참고 자료
- [Paxos Made Simple — Leslie Lamport (2001)](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf)
- [Paxos Algorithm in Distributed System — GeeksforGeeks](https://www.geeksforgeeks.org/operating-systems/paxos-algorithm-in-distributed-system/)
- [Paxos: A Distributed Consensus Algorithm — Medium](https://medium.com/designing-distributed-systems/paxos-a-distributed-consensus-algorithm-41946d5d7d9)
- [Wikipedia: Paxos (computer science)](https://en.wikipedia.org/wiki/Paxos_(computer_science))
