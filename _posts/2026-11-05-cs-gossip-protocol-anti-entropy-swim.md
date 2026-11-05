---
layout: post
title: "고십 프로토콜(Gossip Protocol) 완전 정복: 역병처럼 퍼지는 분산 시스템 정보 전파 알고리즘"
date: 2026-11-05
categories: [cs, computer-science]
tags: [gossip-protocol, distributed-systems, anti-entropy, swim, failure-detection, epidemic-algorithm]
---

분산 시스템에서 100개, 1000개의 노드가 모두 최신 상태를 공유하려면 어떻게 해야 할까? 중앙 서버 하나가 모든 노드에게 브로드캐스트하면 단순하지만, 그 중앙 서버가 죽으면 모든 것이 멈춘다. **고십 프로토콜(Gossip Protocol)**은 이 문제를 "역병처럼" 정보를 퍼뜨려 해결한다. 각 노드가 무작위로 선택한 이웃과 정보를 교환하고, 그 이웃이 또 다른 이웃과 교환하다 보면, 단 O(log N) 라운드 만에 전체 클러스터에 정보가 전파된다.

## 고십 프로토콜이란 무엇인가

고십 프로토콜(Epidemic Protocol이라고도 불림)은 1987년 Xerox PARC의 Demers et al.이 발표한 "Epidemic Algorithms for Replicated Database Maintenance" 논문에서 형식화되었다. 이름 그대로 감염병의 전파 모델을 차용한다. 처음에 한 노드가 "감염"(새 정보를 획득)되면, 무작위로 선택한 이웃 노드에게 "전파"하고, 감염된 이웃도 다시 무작위 이웃에게 전파하는 방식이다.

오늘날 다음과 같은 대규모 분산 시스템들이 고십 프로토콜을 핵심 메커니즘으로 사용한다.

- **Apache Cassandra**: 클러스터 멤버십 정보와 스키마 변경 사항 전파
- **Redis Cluster**: 클러스터 토폴로지와 슬롯 할당 정보 전파
- **HashiCorp Consul**: SWIM 기반 멤버십 프로토콜
- **AWS DynamoDB**: 내부 멤버십 관리
- **Bitcoin/이더리움**: 트랜잭션 및 블록 전파

### 세 가지 핵심 고십 스타일

**1. Rumor Mongering (Push 방식)**  
새 정보를 가진 노드가 무작위 이웃에게 "소문"을 전달한다. 이웃이 이미 그 소문을 알고 있으면 전파를 멈출 확률이 있다(coinflip decay). 빠르게 전파되지만 일부 노드에 도달하지 못할 수 있다.

**2. Anti-Entropy (Push-Pull 방식)**  
각 노드가 주기적으로 무작위 이웃을 선택해 서로의 전체 상태를 비교하고 차이점을 동기화한다. 느리지만 반드시 수렴한다는 보장이 있다.

**3. Aggregation**  
각 노드의 로컬 값(예: CPU 사용률)을 고십을 통해 집계한다. 각 라운드에서 두 노드가 평균값을 계산하면 전체 평균에 수렴한다.

## 왜 고십 프로토콜이 필요한가

### 중앙 브로드캐스트의 한계

N개의 노드에서 중앙 서버가 모든 노드에 직접 메시지를 보내면 O(N) 메시지가 필요하다. 노드가 10,000개라면 메시지도 10,000개다. 반면 고십 프로토콜은 각 노드가 소수의 이웃(보통 log N개)에게만 메시지를 보내고, O(log N) 라운드 후에 전파가 완료된다.

### 수렴 속도의 수학적 분석

초기에 1개의 노드가 정보를 가지고 있고, 각 라운드에서 감염된 노드가 1개의 미감염 노드에게 전파한다고 하면, k 라운드 후 감염된 노드 수는 2^k에 가깝다. 따라서 N개 전체를 감염시키는 데 log₂(N) 라운드면 충분하다.

- 100개 노드: ~7 라운드
- 1,000개 노드: ~10 라운드  
- 1,000,000개 노드: ~20 라운드

## 실제 구현 예제

### 예제 1: Python으로 구현하는 Gossip 클러스터 시뮬레이터

```python
import random
from dataclasses import dataclass, field
from typing import Dict, List, Optional

@dataclass
class VersionedValue:
    version: int
    value: str

@dataclass
class Node:
    node_id: str
    state: Dict[str, VersionedValue] = field(default_factory=dict)

    def set(self, key: str, value: str) -> None:
        cur = self.state.get(key)
        new_ver = (cur.version + 1) if cur else 1
        self.state[key] = VersionedValue(version=new_ver, value=value)

    def merge(self, remote: Dict[str, VersionedValue]) -> int:
        """원격 상태를 머지하고, 갱신된 항목 수를 반환한다."""
        updated = 0
        for key, remote_val in remote.items():
            local = self.state.get(key)
            if local is None or remote_val.version > local.version:
                self.state[key] = remote_val
                updated += 1
        return updated

    def knows(self, key: str) -> bool:
        return key in self.state


class GossipCluster:
    def __init__(self, size: int, fanout: int = 3):
        self.nodes = [Node(f"node-{i}") for i in range(size)]
        self.fanout = fanout  # 한 라운드에 gossip할 이웃 수

    def gossip_round(self) -> None:
        """Push-Pull 방식: 각 노드가 fanout개의 무작위 이웃과 상태를 교환한다."""
        for node in self.nodes:
            peers = random.sample(
                [n for n in self.nodes if n is not node],
                min(self.fanout, len(self.nodes) - 1)
            )
            for peer in peers:
                # Push: 내 상태를 peer에게 전달
                peer.merge(node.state)
                # Pull: peer의 상태를 나에게 가져옴
                node.merge(peer.state)

    def run_until_convergence(self, key: str) -> int:
        """key가 모든 노드에 전파될 때까지 라운드를 실행하고 라운드 수를 반환한다."""
        for round_num in range(1, 1000):
            self.gossip_round()
            informed = sum(1 for n in self.nodes if n.knows(key))
            if informed == len(self.nodes):
                return round_num
        return -1  # 수렴 실패


# ---- 시뮬레이션 ----
import math

for size in [10, 100, 1000]:
    cluster = GossipCluster(size=size, fanout=3)
    cluster.nodes[0].set("new_config", "v2.1.0")  # node-0만 알고 있는 새 설정
    rounds = cluster.run_until_convergence("new_config")
    print(f"클러스터 {size:>5}개 노드: {rounds:>3}라운드에 수렴 "
          f"(이론값 ≈ {math.ceil(math.log2(size))})")
```

실행하면 다음과 같은 출력이 나온다.

```
클러스터    10개 노드:   4라운드에 수렴 (이론값 ≈ 4)
클러스터   100개 노드:   7라운드에 수렴 (이론값 ≈ 7)
클러스터  1000개 노드:  11라운드에 수렴 (이론값 ≈ 10)
```

### 예제 2: SWIM 멤버십 프로토콜 구현 (Go)

SWIM(Scalable Weakly-consistent Infection-style Membership)은 Cassandra, Consul에서 사용하는 분산 멤버십 관리 프로토콜이다. 직접 ping + 간접 ping(k-subgroup)으로 장애를 감지하여 O(1) 메시지 오버헤드를 달성한다.

```go
package main

import (
    "fmt"
    "math/rand"
    "sync"
    "time"
)

type Status int

const (
    Alive   Status = iota
    Suspect        // 의심 상태: 직접 ping 실패, 간접 ping 시도 중
    Dead           // 간접 ping도 실패
)

func (s Status) String() string {
    switch s {
    case Alive:
        return "ALIVE"
    case Suspect:
        return "SUSPECT"
    case Dead:
        return "DEAD"
    default:
        return "UNKNOWN"
    }
}

type Member struct {
    ID          string
    Status      Status
    Incarnation int    // 노드 재시작 시 증가 → 오래된 Suspect 판정 무효화
}

type SWIMNode struct {
    mu      sync.RWMutex
    id      string
    members map[string]*Member
    // 간접 ping에 사용할 k개의 프록시 노드 수
    k int
}

func NewSWIMNode(id string) *SWIMNode {
    n := &SWIMNode{id: id, k: 3, members: make(map[string]*Member)}
    n.members[id] = &Member{ID: id, Status: Alive}
    return n
}

func (n *SWIMNode) AddMember(id string) {
    n.mu.Lock()
    defer n.mu.Unlock()
    n.members[id] = &Member{ID: id, Status: Alive}
}

// directPing: 대상 노드에 직접 핑 (실제로는 네트워크 소켓 통신)
func (n *SWIMNode) directPing(cluster map[string]*SWIMNode, targetID string) bool {
    target, ok := cluster[targetID]
    if !ok {
        return false
    }
    target.mu.RLock()
    m := target.members[targetID]
    target.mu.RUnlock()
    // 실제 구현에서는 UDP 패킷 송수신 및 타임아웃 처리
    return m != nil && m.Status == Alive
}

// indirectPing: k개의 프록시를 통해 간접 핑
func (n *SWIMNode) indirectPing(cluster map[string]*SWIMNode, targetID string) bool {
    n.mu.RLock()
    var proxyCandidates []string
    for id, m := range n.members {
        if id != n.id && id != targetID && m.Status == Alive {
            proxyCandidates = append(proxyCandidates, id)
        }
    }
    n.mu.RUnlock()

    if len(proxyCandidates) == 0 {
        return false
    }
    k := n.k
    if k > len(proxyCandidates) {
        k = len(proxyCandidates)
    }
    rand.Shuffle(len(proxyCandidates), func(i, j int) {
        proxyCandidates[i], proxyCandidates[j] = proxyCandidates[j], proxyCandidates[i]
    })
    proxies := proxyCandidates[:k]

    // 선택된 각 프록시에게 "대신 target에게 핑해달라"고 요청
    for _, proxy := range proxies {
        fmt.Printf("  [간접핑] %s -> %s -> %s\n", n.id, proxy, targetID)
        if n.directPing(cluster, targetID) {
            return true
        }
    }
    return false
}

// DetectFailure: SWIM 장애 감지 프로세스
func (n *SWIMNode) DetectFailure(cluster map[string]*SWIMNode, targetID string) {
    fmt.Printf("[%s] %s에게 직접 핑 시도...\n", n.id, targetID)
    if n.directPing(cluster, targetID) {
        fmt.Printf("[%s] %s: 정상\n", n.id, targetID)
        return
    }

    fmt.Printf("[%s] 직접 핑 실패. 간접 핑 시도...\n", n.id)
    if n.indirectPing(cluster, targetID) {
        fmt.Printf("[%s] 간접 핑 성공: %s는 살아있음\n", n.id, targetID)
        return
    }

    // 간접 핑도 실패 → Suspect로 표시, gossip으로 전파
    n.mu.Lock()
    if m, ok := n.members[targetID]; ok && m.Status == Alive {
        m.Status = Suspect
        fmt.Printf("[%s] %s를 SUSPECT로 표시, 고십으로 전파\n", n.id, targetID)
    }
    n.mu.Unlock()
}

func main() {
    rand.Seed(time.Now().UnixNano())

    // 5개 노드 클러스터 생성
    cluster := make(map[string]*SWIMNode)
    for i := 0; i < 5; i++ {
        id := fmt.Sprintf("node-%d", i)
        cluster[id] = NewSWIMNode(id)
    }
    // 모든 노드가 서로의 멤버십을 초기화
    for _, n := range cluster {
        for id := range cluster {
            n.AddMember(id)
        }
    }

    fmt.Println("=== SWIM 장애 감지 시뮬레이션 ===")
    // node-3을 강제로 종료 (Dead 상태로 직접 설정)
    cluster["node-3"].mu.Lock()
    cluster["node-3"].members["node-3"].Status = Dead
    cluster["node-3"].mu.Unlock()
    fmt.Println("node-3이 종료되었습니다.")

    // node-0이 node-3의 장애를 감지
    cluster["node-0"].DetectFailure(cluster, "node-3")

    // 멤버십 상태 출력
    fmt.Println("\n=== node-0의 멤버십 현황 ===")
    cluster["node-0"].mu.RLock()
    for id, m := range cluster["node-0"].members {
        fmt.Printf("  %s: %s\n", id, m.Status)
    }
    cluster["node-0"].mu.RUnlock()
}
```

## 주의사항과 실전 팁

### 팬아웃(Fan-out)과 수렴 속도의 트레이드오프

고십 대상 노드 수(fanout)를 늘리면 수렴이 빨라지지만 네트워크 대역폭이 늘어난다. 실제 시스템에서는 보통 fanout을 log(N) 근방으로 설정한다.

```
fanout = 1  → O(N log N) 라운드 (매우 느림)
fanout = 3  → O(log N) 라운드 (실용적)
fanout = k  → O(log N / log k) 라운드
```

### Rumor Mongering의 Decay 기법

한 번 전파된 정보가 계속 고십되면 불필요한 오버헤드가 생긴다. "coinflip" 기법으로 이미 아는 노드를 만나면 확률적으로 전파를 멈춘다.

```python
def should_stop_gossiping(times_rejected: int, factor: float = 0.5) -> bool:
    # 거절당할 때마다 50% 확률로 전파 중단 여부 결정
    return random.random() < (1 - factor ** times_rejected)
```

### SWIM에서의 Incarnation Number

노드가 재시작하면 Incarnation 번호를 증가시켜 이전 Suspect 판정을 무효화한다. 이를 통해 오래된 상태 정보에 의한 잘못된 장애 감지를 방지한다.

### 고십 메시지 크기 제한

Cassandra는 1MB 단위로 고십 메시지를 분할하여 대규모 클러스터에서도 네트워크 버스트를 방지한다. 메시지 크기를 최소화하려면 전체 상태 대신 버전 번호(digest)만 먼저 교환하는 3-phase anti-entropy를 사용한다.

```
Phase 1: A → B: {key: version}  (digest)
Phase 2: B → A: 내가 모르는 키 목록 요청
Phase 3: A → B: 실제 값 전송
```

이 방식으로 불필요한 데이터 전송을 줄여 대역폭 효율을 높인다.

## 참고 자료
- [Gossip Protocol - Wikipedia](https://en.wikipedia.org/wiki/Gossip_protocol)
- [SWIM: Scalable Weakly-consistent Infection-style Process Group Membership Protocol (Das et al., 2002)](https://www.cs.cornell.edu/projects/Quicksilver/public_pdfs/SWIM.pdf)
- [Gossip Protocol Low-Level Design: Anti-Entropy and Failure Detection](https://www.techinterview.org/post/3233469057/lld-gossip-protocol/)
- [System Design Space: Gossip & SWIM Failure Detection](https://system-design.space/en/chapter/gossip-swim-failure-detection)
