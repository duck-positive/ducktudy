---
layout: post
title: "Raft 합의 알고리즘 완전 정복: 분산 시스템에서 리더를 선출하는 법"
date: 2026-06-18
categories: [cs, computer-science]
tags: [raft, consensus, distributed-systems, leader-election, log-replication, paxos, etcd, kubernetes]
---

## 개념 설명

분산 시스템에서 가장 어려운 문제 중 하나는 **합의(Consensus)**: 네트워크로 연결된 여러 노드가 특정 값이나 순서에 동의하게 만드는 것입니다. 노드는 언제든지 다운될 수 있고, 네트워크 패킷은 지연·손실·순서 바뀜이 발생할 수 있습니다.

**Raft**는 2014년 Diego Ongaro와 John Ousterhout가 스탠퍼드에서 설계한 합의 알고리즘입니다. 기존의 Paxos 알고리즘이 이해하기 어렵고 구현이 복잡하다는 비판을 받자, *"Understandability"*를 핵심 설계 목표로 삼아 개발되었습니다. 오늘날 etcd(쿠버네티스의 상태 저장소), CockroachDB, TiKV, Consul 등이 Raft를 채택합니다.

### Raft의 세 가지 핵심 서브문제

**1. 리더 선출 (Leader Election)**

Raft 클러스터의 모든 노드는 세 가지 상태 중 하나입니다: **Follower**, **Candidate**, **Leader**.

- 시작 시 모든 노드는 Follower 상태
- Follower가 일정 시간(election timeout, 보통 150~300ms) 동안 Leader의 **하트비트**를 받지 못하면 Candidate로 전환
- Candidate는 자신의 **Term 번호**를 1 올리고, 다른 노드에게 `RequestVote` RPC를 보냄
- 과반수 득표 시 Leader가 됨. Leader는 주기적으로 하트비트(`AppendEntries` RPC with empty entries)를 보내 자신의 존재를 알림

각 노드는 Term당 **한 번만** 투표할 수 있고, 먼저 요청한 Candidate에게 투표합니다. 이로써 동시에 두 명의 Leader가 선출되는 **Split-Brain**을 방지합니다.

**2. 로그 복제 (Log Replication)**

Leader는 클라이언트 요청을 받으면 자신의 로그에 추가하고, `AppendEntries` RPC로 모든 Follower에게 전파합니다. **과반수 노드가 로그에 기록 완료**했을 때 해당 엔트리는 *커밋(Committed)* 상태가 되고, Leader는 클라이언트에 응답합니다.

네트워크 분리 등으로 일부 Follower가 뒤처질 경우, Leader는 해당 Follower의 `nextIndex`를 줄여가며 일치하는 지점을 찾고, 그 이후의 엔트리를 재전송하여 로그를 일치시킵니다.

**3. 안전성 (Safety)**

Raft는 **"한 번 커밋된 엔트리는 절대 덮어쓰이지 않는다"**는 불변식을 보장합니다. 투표 시 후보의 로그가 투표자의 로그보다 최신이어야 한다는 조건이 이를 보장합니다(로그 최신성 비교: Term 번호 → Index 순으로 비교).

---

## 왜 필요한가

단일 서버 데이터베이스는 그 서버가 다운되면 전체 시스템이 중단됩니다. 고가용성(HA)을 위해 여러 노드를 운영할 때, **"어느 노드가 현재의 진실(Source of Truth)인가?"**를 결정해야 합니다. 이 과정에서 합의 없이는:

- 두 노드가 동시에 서로 다른 값을 커밋 (데이터 불일치)
- 네트워크 분리 후 두 쪽이 독립적으로 동작 (Split-Brain)

Raft는 이 문제를 **단일 Leader를 통한 직렬화**로 해결합니다. 모든 쓰기는 Leader를 경유하므로, 순서가 항상 일관되게 결정됩니다.

---

## 실제 구현 예제

### 예제 1: Go로 구현한 Raft 상태 머신 스켈레톤

```go
package raft

import (
    "math/rand"
    "sync"
    "time"
)

type NodeState int

const (
    Follower  NodeState = iota
    Candidate
    Leader
)

type LogEntry struct {
    Term    int
    Command interface{}
}

type RaftNode struct {
    mu          sync.Mutex
    id          int
    peers       []int
    state       NodeState
    currentTerm int
    votedFor    int // -1이면 아직 투표 안 함
    log         []LogEntry
    commitIndex int
    lastApplied int

    electionTimer *time.Timer
}

func NewRaftNode(id int, peers []int) *RaftNode {
    n := &RaftNode{
        id:          id,
        peers:       peers,
        state:       Follower,
        currentTerm: 0,
        votedFor:    -1,
    }
    n.resetElectionTimer()
    return n
}

// Election Timeout 랜덤화: 150~300ms 범위로 동시 선출 방지
func (n *RaftNode) electionTimeout() time.Duration {
    return time.Duration(150+rand.Intn(150)) * time.Millisecond
}

func (n *RaftNode) resetElectionTimer() {
    if n.electionTimer != nil {
        n.electionTimer.Stop()
    }
    n.electionTimer = time.AfterFunc(n.electionTimeout(), n.startElection)
}

func (n *RaftNode) startElection() {
    n.mu.Lock()
    n.state = Candidate
    n.currentTerm++
    n.votedFor = n.id
    term := n.currentTerm
    n.mu.Unlock()

    votes := 1 // 자기 자신에게 투표
    majority := (len(n.peers)+1)/2 + 1

    for _, peer := range n.peers {
        go func(p int) {
            granted := n.sendRequestVote(p, term)
            n.mu.Lock()
            defer n.mu.Unlock()
            if granted && n.state == Candidate && n.currentTerm == term {
                votes++
                if votes >= majority {
                    n.becomeLeader()
                }
            }
        }(peer)
    }
    // 타이머 재설정 (과반수 못 얻으면 다시 시도)
    n.resetElectionTimer()
}

func (n *RaftNode) becomeLeader() {
    n.state = Leader
    // Leader는 즉시 하트비트를 전송해 권한을 확립
    go n.sendHeartbeats()
}

func (n *RaftNode) sendRequestVote(peer int, term int) bool {
    // 실제 구현에서는 gRPC/RPC 호출
    // 여기서는 개념 설명용 스텁
    return false
}

func (n *RaftNode) sendHeartbeats() {
    ticker := time.NewTicker(50 * time.Millisecond)
    for range ticker.C {
        n.mu.Lock()
        if n.state != Leader {
            n.mu.Unlock()
            ticker.Stop()
            return
        }
        n.mu.Unlock()
        for _, peer := range n.peers {
            go n.sendAppendEntries(peer, nil) // empty = heartbeat
        }
    }
}

func (n *RaftNode) sendAppendEntries(peer int, entries []LogEntry) {
    // gRPC 등으로 실제 전송
}
```

### 예제 2: Python으로 RequestVote 로직 시뮬레이션

```python
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class LogEntry:
    term: int
    command: str

@dataclass
class RaftPeer:
    node_id: int
    current_term: int = 0
    voted_for: Optional[int] = None
    log: list = field(default_factory=list)

    def last_log_term(self) -> int:
        return self.log[-1].term if self.log else 0

    def last_log_index(self) -> int:
        return len(self.log) - 1

    def request_vote(self, candidate_id: int, candidate_term: int,
                     last_log_index: int, last_log_term: int) -> bool:
        """
        RequestVote RPC 수신 처리.
        후보의 로그가 나보다 최신이어야 투표.
        """
        # 1) 오래된 Term이면 거부
        if candidate_term < self.current_term:
            print(f"  Node {self.node_id}: 거부 (term {candidate_term} < {self.current_term})")
            return False

        # 2) Term 업데이트 (더 높은 Term을 보면 Follower로 전환)
        if candidate_term > self.current_term:
            self.current_term = candidate_term
            self.voted_for = None

        # 3) 이미 투표했으면 거부
        if self.voted_for is not None and self.voted_for != candidate_id:
            print(f"  Node {self.node_id}: 거부 (이미 {self.voted_for}에게 투표)")
            return False

        # 4) 로그 최신성 확인 (Raft §5.4.1)
        my_last_term = self.last_log_term()
        my_last_idx = self.last_log_index()
        log_ok = (last_log_term > my_last_term) or \
                 (last_log_term == my_last_term and last_log_index >= my_last_idx)

        if log_ok:
            self.voted_for = candidate_id
            print(f"  Node {self.node_id}: 찬성 → Candidate {candidate_id}")
            return True
        else:
            print(f"  Node {self.node_id}: 거부 (로그 뒤처짐)")
            return False


# 시뮬레이션: 3노드 클러스터에서 선출
nodes = [RaftPeer(node_id=i, current_term=1) for i in range(3)]
# Node 0이 Candidate 선언 (Term=2, 로그 없음)
votes = sum(
    1 for n in nodes
    if n.request_vote(candidate_id=0, candidate_term=2,
                      last_log_index=-1, last_log_term=0)
)
print(f"\n총 득표: {votes} / {len(nodes)} → {'리더 선출 성공' if votes > len(nodes)//2 else '실패'}")
```

---

## 주의사항 및 실무 팁

**Election Timeout 튜닝**: Timeout이 너무 짧으면 불필요한 선거가 자주 발생하고, 너무 길면 Leader 장애 감지가 느려집니다. 보통 Heartbeat Interval의 10배 이상으로 설정합니다.

**Pre-Vote 최적화**: 네트워크 분리된 노드가 복구 후 높은 Term으로 선거를 시작해 클러스터를 방해하는 문제를 막기 위해, etcd는 **Pre-Vote** 단계를 추가합니다. 실제 선거 전 "내가 이기겠냐"를 먼저 타진합니다.

**읽기 성능**: 모든 읽기가 Leader를 통과하면 Leader 부하가 집중됩니다. **ReadIndex**나 **Lease Read** 기법을 사용하면 Follower가 안전하게 선형화 읽기를 제공할 수 있습니다.

**홀수 노드 구성**: 과반수 원칙 때문에 노드 수는 홀수(3, 5, 7)로 구성하는 것이 표준입니다. 4노드 클러스터는 2노드 장애를 허용하지 못해(과반수 = 3) 3노드 대비 가용성이 나아지지 않습니다.

**실무 라이브러리**:
- **etcd/raft** (Go): https://github.com/etcd-io/etcd/tree/main/raft
- **hashicorp/raft** (Go): https://github.com/hashicorp/raft
- **ratis** (Java): Apache Ratis, HBase/Ozone에서 사용

## 참고 자료
- [Raft Consensus Algorithm - 공식 사이트](https://raft.github.io/)
- [Raft Consensus Algorithm - GeeksforGeeks](https://www.geeksforgeeks.org/system-design/raft-consensus-algorithm/)
- [Understanding Raft Consensus in Distributed Systems - PingCAP](https://www.pingcap.com/article/understanding-raft-consensus-in-distributed-systems-with-tidb/)
- [Raft Protocol: What is the Raft Consensus Algorithm? - YugaByte](https://www.yugabyte.com/key-concepts/raft-consensus-algorithm/)
