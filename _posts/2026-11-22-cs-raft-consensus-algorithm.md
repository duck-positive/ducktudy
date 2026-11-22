---
layout: post
title: "Raft 합의 알고리즘 완전 정복: Paxos보다 이해하기 쉬운 분산 합의의 모든 것"
date: 2026-11-22
categories: [cs, computer-science]
tags: [raft, consensus, distributed-systems, leader-election, log-replication, etcd]
---

## 개요

분산 시스템에서 여러 노드가 동일한 상태를 유지하는 것은 근본적으로 어려운 문제입니다. 네트워크 파티션, 노드 장애, 메시지 지연이 발생하는 환경에서도 모든 노드가 "합의(consensus)"에 도달해야 합니다. 오랫동안 분산 합의의 표준은 **Paxos**였지만, 그 복잡성은 악명이 높았습니다. 2014년 Diego Ongaro와 John Ousterhout(Stanford)가 발표한 **Raft**는 "이해 가능성(understandability)"을 핵심 설계 목표로 삼아 등장했습니다.

이 글에서는 Raft의 세 핵심 메커니즘인 **리더 선출(Leader Election)**, **로그 복제(Log Replication)**, **안전성(Safety)**을 살펴보고, Go와 Python으로 핵심 로직을 구현합니다.

---

## 왜 Raft인가?

### Paxos의 문제

Paxos는 수학적으로 정확하지만, 실제 시스템으로 구현하기 위해서는 논문에 명시되지 않은 수많은 세부 결정이 필요합니다. Google의 Chubby 개발팀은 "Paxos는 실제 구현과 거리가 먼 알고리즘"이라고 언급했습니다.

### Raft의 목표

- **이해 가능성**: Ongaro의 PhD 논문과 함께 진행된 사용자 연구에서, 같은 내용을 배운 학생들이 Paxos보다 Raft에 대한 질문을 더 잘 답했습니다.
- **완전한 사양**: 리더 선출, 로그 복제, 멤버십 변경, 로그 압축(스냅샷)까지 하나의 알고리즘으로 명확하게 정의합니다.
- **실용성**: etcd(Kubernetes의 핵심 저장소), CockroachDB, TiKV, Consul 등 주요 분산 시스템이 Raft를 채택하고 있습니다.

---

## Raft의 세 가지 핵심 문제

### 1. 리더 선출 (Leader Election)

Raft 클러스터의 노드는 세 가지 상태 중 하나를 가집니다: **Follower**, **Candidate**, **Leader**

- 모든 노드는 Follower로 시작합니다.
- Follower가 일정 시간(election timeout, 보통 150~300ms) 동안 Leader로부터 하트비트를 받지 못하면 Candidate로 전환합니다.
- Candidate는 자신의 **term(임기 번호)**를 1 증가시키고, 다른 모든 노드에 `RequestVote` RPC를 보냅니다.
- **과반수(majority)** 의 투표를 얻으면 새 Leader가 됩니다.
- 두 Candidate가 동시에 선출을 시작하면 분열될 수 있으므로, timeout을 무작위로 설정하여 tie를 줄입니다.

### 2. 로그 복제 (Log Replication)

Leader는 클라이언트의 명령을 받아:
1. 자신의 로그에 항목(log entry)을 추가합니다.
2. 모든 Follower에 `AppendEntries` RPC로 항목을 전송합니다.
3. **과반수** Follower가 복제를 완료하면 항목을 **커밋(commit)** 합니다.
4. 커밋된 항목을 상태 머신(state machine)에 적용하고 클라이언트에 응답합니다.

### 3. 안전성 (Safety)

Raft는 다음 불변식을 보장합니다:
- **선출 안전성**: 각 term에 최대 하나의 Leader만 선출됩니다.
- **리더 로그 완전성**: 커밋된 항목은 모든 미래 Leader의 로그에 포함됩니다.
- **상태 머신 안전성**: 같은 인덱스에서 두 상태 머신이 다른 명령을 적용하지 않습니다.

---

## 구현 예제 1: Python으로 Raft 상태 머신 시뮬레이션

```python
import enum
import random
import time
from dataclasses import dataclass, field
from typing import Optional

class NodeState(enum.Enum):
    FOLLOWER = "follower"
    CANDIDATE = "candidate"
    LEADER = "leader"

@dataclass
class LogEntry:
    term: int
    command: str

@dataclass
class RaftNode:
    node_id: int
    peers: list
    # 영구 상태
    current_term: int = 0
    voted_for: Optional[int] = None
    log: list = field(default_factory=list)
    # 휘발성 상태
    state: NodeState = NodeState.FOLLOWER
    commit_index: int = -1
    last_applied: int = -1
    # 리더 전용
    next_index: dict = field(default_factory=dict)
    match_index: dict = field(default_factory=dict)
    votes_received: int = 0

    def start_election(self):
        """Candidate로 전환하여 선출 시작"""
        self.state = NodeState.CANDIDATE
        self.current_term += 1
        self.voted_for = self.node_id
        self.votes_received = 1  # 자기 자신에게 투표
        print(f"[Node {self.node_id}] Starting election for term {self.current_term}")

    def request_vote(self, candidate_id: int, candidate_term: int,
                     last_log_index: int, last_log_term: int) -> tuple[int, bool]:
        """투표 요청 처리"""
        if candidate_term < self.current_term:
            return self.current_term, False

        if candidate_term > self.current_term:
            self.current_term = candidate_term
            self.state = NodeState.FOLLOWER
            self.voted_for = None

        # 이미 다른 후보에게 투표했으면 거부
        if self.voted_for is not None and self.voted_for != candidate_id:
            return self.current_term, False

        # 후보의 로그가 자신보다 최신인지 확인 (로그 완전성 보장)
        my_last_log_term = self.log[-1].term if self.log else -1
        my_last_log_index = len(self.log) - 1

        log_ok = (candidate_term > my_last_log_term) or \
                 (candidate_term == my_last_log_term and last_log_index >= my_last_log_index)

        if log_ok:
            self.voted_for = candidate_id
            print(f"[Node {self.node_id}] Voted for Node {candidate_id} in term {candidate_term}")
            return self.current_term, True

        return self.current_term, False

    def append_entries(self, leader_id: int, term: int, prev_log_index: int,
                       prev_log_term: int, entries: list, leader_commit: int) -> tuple[int, bool]:
        """AppendEntries RPC 처리 (하트비트 + 로그 복제)"""
        if term < self.current_term:
            return self.current_term, False

        # 유효한 Leader로부터 메시지 수신 → Follower 유지
        self.current_term = term
        self.state = NodeState.FOLLOWER

        # 이전 로그 항목 일관성 확인
        if prev_log_index >= 0:
            if prev_log_index >= len(self.log):
                return self.current_term, False
            if self.log[prev_log_index].term != prev_log_term:
                # 충돌하는 항목 삭제
                self.log = self.log[:prev_log_index]
                return self.current_term, False

        # 새 항목 추가
        for i, entry in enumerate(entries):
            idx = prev_log_index + 1 + i
            if idx < len(self.log):
                if self.log[idx].term != entry.term:
                    self.log = self.log[:idx]
                    self.log.append(entry)
            else:
                self.log.append(entry)

        # 커밋 인덱스 업데이트
        if leader_commit > self.commit_index:
            self.commit_index = min(leader_commit, len(self.log) - 1)

        return self.current_term, True

    def become_leader(self, cluster_size: int):
        """Leader로 전환"""
        self.state = NodeState.LEADER
        self.next_index = {p: len(self.log) for p in self.peers}
        self.match_index = {p: -1 for p in self.peers}
        print(f"[Node {self.node_id}] Became LEADER for term {self.current_term}")

    def propose(self, command: str) -> Optional[int]:
        """클라이언트 명령 수락 (Leader만 가능)"""
        if self.state != NodeState.LEADER:
            return None
        entry = LogEntry(term=self.current_term, command=command)
        self.log.append(entry)
        print(f"[Node {self.node_id}] Appended to log: {command} (index={len(self.log)-1})")
        return len(self.log) - 1


# 간단한 선출 시뮬레이션
nodes = [RaftNode(node_id=i, peers=[j for j in range(5) if j != i]) for i in range(5)]

# 노드 0이 선출 시작
nodes[0].start_election()

# 다른 노드들에게 투표 요청 시뮬레이션
votes = 1  # 자신
for peer_id in [1, 2, 3, 4]:
    _, granted = nodes[peer_id].request_vote(
        candidate_id=0,
        candidate_term=nodes[0].current_term,
        last_log_index=len(nodes[0].log) - 1,
        last_log_term=nodes[0].log[-1].term if nodes[0].log else -1
    )
    if granted:
        votes += 1

if votes > len(nodes) // 2:
    nodes[0].become_leader(cluster_size=len(nodes))
    # 명령 제안
    nodes[0].propose("SET x = 1")
    nodes[0].propose("SET y = 2")
```

**출력 예시:**
```
[Node 0] Starting election for term 1
[Node 1] Voted for Node 0 in term 1
[Node 2] Voted for Node 0 in term 1
[Node 3] Voted for Node 0 in term 1
[Node 4] Voted for Node 0 in term 1
[Node 0] Became LEADER for term 1
[Node 0] Appended to log: SET x = 1 (index=0)
[Node 0] Appended to log: SET y = 2 (index=1)
```

---

## 구현 예제 2: Go로 Raft 핵심 구조 구현

```go
package raft

import (
	"math/rand"
	"sync"
	"time"
)

type State int

const (
	Follower State = iota
	Candidate
	Leader
)

type LogEntry struct {
	Term    int
	Command interface{}
}

type RequestVoteArgs struct {
	Term         int
	CandidateID  int
	LastLogIndex int
	LastLogTerm  int
}

type RequestVoteReply struct {
	Term        int
	VoteGranted bool
}

type AppendEntriesArgs struct {
	Term         int
	LeaderID     int
	PrevLogIndex int
	PrevLogTerm  int
	Entries      []LogEntry
	LeaderCommit int
}

type AppendEntriesReply struct {
	Term    int
	Success bool
}

type Raft struct {
	mu          sync.Mutex
	id          int
	peers       []int
	state       State
	currentTerm int
	votedFor    int
	log         []LogEntry
	commitIndex int
	lastApplied int
	nextIndex   map[int]int
	matchIndex  map[int]int
	electionTimer *time.Timer
}

func NewRaft(id int, peers []int) *Raft {
	r := &Raft{
		id:          id,
		peers:       peers,
		state:       Follower,
		currentTerm: 0,
		votedFor:    -1,
		log:         []LogEntry{},
		commitIndex: -1,
		lastApplied: -1,
	}
	r.resetElectionTimer()
	return r
}

func (r *Raft) resetElectionTimer() {
	if r.electionTimer != nil {
		r.electionTimer.Stop()
	}
	timeout := time.Duration(150+rand.Intn(150)) * time.Millisecond
	r.electionTimer = time.AfterFunc(timeout, r.startElection)
}

func (r *Raft) startElection() {
	r.mu.Lock()
	r.state = Candidate
	r.currentTerm++
	r.votedFor = r.id
	term := r.currentTerm
	lastLogIndex := len(r.log) - 1
	lastLogTerm := -1
	if lastLogIndex >= 0 {
		lastLogTerm = r.log[lastLogIndex].Term
	}
	r.mu.Unlock()

	votes := 1
	var voteMu sync.Mutex
	var wg sync.WaitGroup

	for _, peer := range r.peers {
		wg.Add(1)
		go func(peerID int) {
			defer wg.Done()
			args := RequestVoteArgs{
				Term:         term,
				CandidateID:  r.id,
				LastLogIndex: lastLogIndex,
				LastLogTerm:  lastLogTerm,
			}
			// 실제 구현에서는 RPC 호출
			reply := r.sendRequestVote(peerID, args)
			if reply.VoteGranted {
				voteMu.Lock()
				votes++
				voteMu.Unlock()
			}
		}(peer)
	}

	wg.Wait()
	r.mu.Lock()
	defer r.mu.Unlock()

	if r.state == Candidate && r.currentTerm == term && votes > (len(r.peers)+1)/2 {
		r.becomeLeader()
	}
}

func (r *Raft) RequestVote(args RequestVoteArgs, reply *RequestVoteReply) {
	r.mu.Lock()
	defer r.mu.Unlock()

	reply.Term = r.currentTerm
	reply.VoteGranted = false

	if args.Term < r.currentTerm {
		return
	}

	if args.Term > r.currentTerm {
		r.currentTerm = args.Term
		r.state = Follower
		r.votedFor = -1
	}

	if r.votedFor != -1 && r.votedFor != args.CandidateID {
		return
	}

	// 로그 최신성 검사
	lastLogIndex := len(r.log) - 1
	lastLogTerm := -1
	if lastLogIndex >= 0 {
		lastLogTerm = r.log[lastLogIndex].Term
	}

	logOk := args.LastLogTerm > lastLogTerm ||
		(args.LastLogTerm == lastLogTerm && args.LastLogIndex >= lastLogIndex)

	if logOk {
		r.votedFor = args.CandidateID
		r.resetElectionTimer()
		reply.VoteGranted = true
	}
}

func (r *Raft) AppendEntries(args AppendEntriesArgs, reply *AppendEntriesReply) {
	r.mu.Lock()
	defer r.mu.Unlock()

	reply.Term = r.currentTerm
	reply.Success = false

	if args.Term < r.currentTerm {
		return
	}

	r.currentTerm = args.Term
	r.state = Follower
	r.resetElectionTimer()

	// PrevLogIndex 일관성 확인
	if args.PrevLogIndex >= 0 {
		if args.PrevLogIndex >= len(r.log) ||
			r.log[args.PrevLogIndex].Term != args.PrevLogTerm {
			return
		}
	}

	// 로그 항목 추가
	for i, entry := range args.Entries {
		idx := args.PrevLogIndex + 1 + i
		if idx < len(r.log) {
			if r.log[idx].Term != entry.Term {
				r.log = r.log[:idx]
				r.log = append(r.log, entry)
			}
		} else {
			r.log = append(r.log, entry)
		}
	}

	if args.LeaderCommit > r.commitIndex {
		if args.LeaderCommit < len(r.log)-1 {
			r.commitIndex = args.LeaderCommit
		} else {
			r.commitIndex = len(r.log) - 1
		}
	}

	reply.Success = true
}

func (r *Raft) becomeLeader() {
	r.state = Leader
	r.nextIndex = make(map[int]int)
	r.matchIndex = make(map[int]int)
	for _, peer := range r.peers {
		r.nextIndex[peer] = len(r.log)
		r.matchIndex[peer] = -1
	}
	go r.sendHeartbeats()
}

func (r *Raft) sendHeartbeats() {
	ticker := time.NewTicker(50 * time.Millisecond)
	defer ticker.Stop()
	for range ticker.C {
		r.mu.Lock()
		if r.state != Leader {
			r.mu.Unlock()
			return
		}
		r.mu.Unlock()
		for _, peer := range r.peers {
			go r.replicateLog(peer)
		}
	}
}

// sendRequestVote는 실제 구현에서 RPC를 호출합니다.
func (r *Raft) sendRequestVote(peer int, args RequestVoteArgs) RequestVoteReply {
	// 실제 구현에서는 net/rpc 또는 gRPC 사용
	return RequestVoteReply{}
}

func (r *Raft) replicateLog(peer int) {
	// 실제 구현에서 AppendEntries RPC 전송
}
```

---

## Raft의 주요 엣지 케이스

### Term과 Split Vote

무작위 election timeout으로 split vote 가능성을 줄이지만 완전히 없앨 수는 없습니다. Split vote가 발생하면 모든 Candidate가 타임아웃 후 다시 선출을 시도합니다. 이 과정은 term이 계속 증가하면서 반복되며, 결국 하나의 Candidate가 과반수를 얻습니다.

### 로그 불일치 복구

Leader가 교체된 경우, 새 Leader는 Follower의 로그를 자신의 로그로 덮어씁니다. `nextIndex`를 통해 각 Follower에 맞는 위치부터 항목을 재전송합니다.

### 스냅샷 (Log Compaction)

로그가 무한정 커지면 디스크와 메모리 문제가 발생합니다. Raft는 주기적으로 현재 상태를 스냅샷으로 저장하고 그 이전 로그를 삭제합니다. `InstallSnapshot` RPC로 뒤처진 Follower에게 스냅샷을 전송합니다.

### Pre-Vote 최적화

네트워크 파티션이 복구될 때, 고립되었던 노드가 term을 계속 증가시킨 상태로 클러스터에 복귀하면 불필요한 리더 교체가 발생할 수 있습니다. **Pre-Vote** 단계에서 실제 선출 전에 과반수에서 지지 의사를 확인하면 이를 방지할 수 있습니다.

---

## Paxos vs Raft 비교

| 특성 | Paxos | Raft |
|---|---|---|
| 이해 용이성 | 낮음 | 높음 |
| 리더 | 없음 (Proposer) | 단일 Leader 필수 |
| 로그 순서 | 명시적 보장 없음 | 항상 순서대로 커밋 |
| 멤버십 변경 | 별도 명시 없음 | Joint Consensus 정의 |
| 채택 사례 | Chubby, Zookeeper(변형) | etcd, CockroachDB, TiKV, Consul |

---

## 참고 자료

- [Raft 논문 (USENIX ATC '14)](https://www.usenix.org/conference/atc14/technical-sessions/presentation/ongaro)
- [Raft 시각화 웹사이트](https://raft.github.io/)
- [etcd Raft 라이브러리 소스코드](https://github.com/etcd-io/raft)
- [The Secret Lives of Data: Raft 애니메이션](https://thesecretlivesofdata.com/raft/)
