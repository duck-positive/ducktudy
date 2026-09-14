---
layout: post
title: "SWIM 프로토콜 완전 정복: 분산 클러스터에서 장애를 감지하고 멤버십을 관리하는 법"
date: 2026-09-14
categories: [cs, computer-science]
tags: [swim, distributed-systems, membership, failure-detection, gossip, consul, hashicorp]
---

## 개념 설명: 분산 시스템의 멤버십 문제

분산 시스템을 운영하다 보면 가장 기초적이면서도 어려운 문제에 부딪힌다. "지금 클러스터에 살아 있는 노드가 몇 개이고, 어떤 노드가 죽었는가?"

이 질문에 답하는 것이 **멤버십 프로토콜(Membership Protocol)**의 임무다. 각 노드는 현재 클러스터 멤버 목록(membership list)을 유지하고, 새 노드가 합류하거나 기존 노드가 이탈/장애 시 이를 전파해야 한다.

전통적인 방식인 **중앙집중식 하트비팅**은 각 노드가 중앙 모니터에게 주기적으로 생존 신호를 보내는 방식이다. 노드가 수십 개일 때는 잘 동작하지만 수천 개가 되면 모니터가 병목이 된다. **전통적인 All-to-All 하트비팅**은 모든 노드가 모든 노드에게 핑을 보내는 방식으로, 네트워크 트래픽이 O(n²)으로 폭증한다.

이 문제를 해결하기 위해 2002년 Das, Gupta, Motivala가 제안한 것이 **SWIM(Scalable Weakly-consistent Infection-style Process Group Membership)** 프로토콜이다.

### SWIM의 핵심 아이디어

SWIM은 두 컴포넌트로 이루어진다.

1. **실패 감지기(Failure Detector)**: 다른 노드의 생사를 능동적으로 탐지
2. **전파 컴포넌트(Dissemination Component)**: 멤버십 변경 사항을 감염(infection) 방식으로 전파

핵심은 **직접 핑(Direct Ping) + 간접 핑-요청(Indirect Ping-Request)**의 조합이다. 단순히 "응답 없으면 죽은 것"으로 판단하지 않고 여러 경로를 통해 확인함으로써 일시적 네트워크 장애로 인한 오탐(False Positive)을 크게 줄인다.

---

## 왜 SWIM이 필요한가?

### 기존 방식의 문제

**하트비팅 기반 방식의 한계:**

| 방식 | 트래픽 복잡도 | 감지 시간 | 확장성 |
|------|--------------|-----------|--------|
| 중앙집중식 | O(n) → 중앙 병목 | 낮음 | 나쁨 |
| All-to-All | O(n²) | 낮음 | 나쁨 |
| SWIM | O(n) | 조정 가능 | 매우 좋음 |

**SWIM의 수학적 보장:**

- 노드 수 n이 증가해도 각 노드가 보내는 메시지 수는 프로토콜 주기당 O(1)이다.
- 거짓 양성률(false positive rate)은 멤버 수와 무관하게 일정하게 유지된다.
- 멤버십 변경의 전파 시간은 O(log n) 라운드다 (감염 방식의 특성).

### 실제 사용 사례

SWIM 또는 SWIM 변형을 사용하는 대표적인 시스템:
- **HashiCorp Consul**: 서비스 디스커버리 및 헬스 체크
- **HashiCorp Serf**: 분산 오케스트레이션
- **Apache Cassandra**: gossip 기반 클러스터 관리
- **Kubernetes**: 일부 내부 클러스터 관리 컴포넌트

---

## 실제 구현 예제

### 예제 1: SWIM 프로토콜 핵심 로직 시뮬레이션 (Python)

```python
import random
import time
from dataclasses import dataclass, field
from enum import Enum
from typing import Optional

class NodeState(Enum):
    ALIVE = "alive"
    SUSPECT = "suspect"
    DEAD = "dead"

@dataclass
class Member:
    node_id: str
    address: str
    state: NodeState = NodeState.ALIVE
    incarnation: int = 0      # 논리적 타임스탬프 (잘못된 SUSPECT 신호 반박용)
    last_updated: float = field(default_factory=time.time)

class SWIMNode:
    """SWIM 프로토콜 노드 시뮬레이션"""
    
    PROTOCOL_PERIOD = 1.0        # T' : 프로토콜 주기 (초)
    SUSPECT_TIMEOUT = 3.0        # 의심 상태 타임아웃
    PING_TIMEOUT = 0.5           # 핑 응답 대기 시간
    INDIRECT_PING_COUNT = 3      # 간접 핑 요청에 사용할 노드 수
    
    def __init__(self, node_id: str, address: str):
        self.node_id = node_id
        self.address = address
        self.members: dict[str, Member] = {}
        self.incarnation = 0
        
        # 자기 자신을 멤버로 추가
        self.members[node_id] = Member(node_id, address)
        
        # 전파할 이벤트 큐 (gossip buffer)
        self.gossip_buffer: list[dict] = []
    
    def join(self, seed_address: str):
        """클러스터에 조인 (시드 노드에게 알림)"""
        print(f"[{self.node_id}] 클러스터 조인 요청 → {seed_address}")
        # 실제로는 네트워크 통신이 필요하지만 시뮬레이션에서는 직접 처리
    
    def _pick_probe_target(self) -> Optional[Member]:
        """무작위로 핑할 멤버 선택 (자신 제외)"""
        candidates = [
            m for m in self.members.values()
            if m.node_id != self.node_id and m.state != NodeState.DEAD
        ]
        return random.choice(candidates) if candidates else None
    
    def _pick_indirect_helpers(self, exclude: str, k: int) -> list[Member]:
        """간접 핑을 도울 k개의 노드 선택"""
        candidates = [
            m for m in self.members.values()
            if m.node_id != self.node_id and m.node_id != exclude
               and m.state == NodeState.ALIVE
        ]
        return random.sample(candidates, min(k, len(candidates)))
    
    def protocol_period(self, network_simulator):
        """프로토콜 주기 실행: 핑 → (실패 시) 핑-요청 → 판정"""
        target = self._pick_probe_target()
        if not target:
            return
        
        print(f"[{self.node_id}] 직접 핑 → {target.node_id}")
        alive = network_simulator.ping(self.node_id, target.node_id)
        
        if alive:
            # 응답 성공: 멤버 상태 갱신
            self._on_alive(target.node_id)
        else:
            # 응답 없음: 간접 핑으로 확인
            print(f"[{self.node_id}] {target.node_id} 응답 없음 → 간접 핑 시도")
            helpers = self._pick_indirect_helpers(target.node_id, self.INDIRECT_PING_COUNT)
            confirmed_alive = False
            
            for helper in helpers:
                print(f"[{self.node_id}] 핑 요청 → {helper.node_id} → {target.node_id}")
                result = network_simulator.ping_req(
                    self.node_id, helper.node_id, target.node_id
                )
                if result:
                    confirmed_alive = True
                    break
            
            if confirmed_alive:
                self._on_alive(target.node_id)
            else:
                # 여전히 응답 없음: SUSPECT으로 전환
                self._on_suspect(target.node_id)
    
    def _on_alive(self, node_id: str):
        if node_id in self.members:
            if self.members[node_id].state != NodeState.ALIVE:
                print(f"[{self.node_id}] {node_id} ALIVE로 복귀")
            self.members[node_id].state = NodeState.ALIVE
            self.members[node_id].last_updated = time.time()
    
    def _on_suspect(self, node_id: str):
        if node_id in self.members and self.members[node_id].state == NodeState.ALIVE:
            print(f"[{self.node_id}] {node_id} SUSPECT으로 표시")
            self.members[node_id].state = NodeState.SUSPECT
            self.members[node_id].last_updated = time.time()
            # gossip으로 전파
            self.gossip_buffer.append({
                "type": "suspect",
                "node_id": node_id,
                "incarnation": self.members[node_id].incarnation
            })
    
    def _on_dead(self, node_id: str):
        if node_id in self.members and self.members[node_id].state != NodeState.DEAD:
            print(f"[{self.node_id}] {node_id} DEAD 선언")
            self.members[node_id].state = NodeState.DEAD
            self.gossip_buffer.append({
                "type": "dead",
                "node_id": node_id
            })
    
    def check_suspect_timeout(self):
        """SUSPECT 상태 노드가 타임아웃되면 DEAD으로 전환"""
        now = time.time()
        for member in self.members.values():
            if (member.state == NodeState.SUSPECT and
                    now - member.last_updated > self.SUSPECT_TIMEOUT):
                self._on_dead(member.node_id)
    
    def gossip(self, target_node: 'SWIMNode'):
        """버퍼에 있는 이벤트를 대상 노드에게 전파 (piggyback)"""
        events_to_send = self.gossip_buffer[:3]  # 최대 3개씩 piggybacking
        for event in events_to_send:
            target_node.receive_gossip(event)
        # 전파된 이벤트 제거 (람다 감염 횟수 초과 시)
        self.gossip_buffer = self.gossip_buffer[3:]
    
    def receive_gossip(self, event: dict):
        node_id = event.get("node_id")
        if not node_id or node_id not in self.members:
            return
        
        if event["type"] == "suspect":
            if self.members[node_id].incarnation <= event.get("incarnation", 0):
                self._on_suspect(node_id)
        elif event["type"] == "dead":
            self._on_dead(node_id)


class NetworkSimulator:
    """네트워크 시뮬레이터 (패킷 손실, 노드 장애 시뮬레이션)"""
    
    def __init__(self, nodes: dict[str, SWIMNode], failure_rate: float = 0.1):
        self.nodes = nodes
        self.failure_rate = failure_rate
        self.crashed_nodes: set[str] = set()
    
    def crash(self, node_id: str):
        self.crashed_nodes.add(node_id)
        print(f"[NETWORK] 노드 {node_id} 장애 발생!")
    
    def ping(self, from_id: str, to_id: str) -> bool:
        if to_id in self.crashed_nodes:
            return False
        if random.random() < self.failure_rate:  # 패킷 손실
            return False
        return True
    
    def ping_req(self, requester: str, helper: str, target: str) -> bool:
        if helper in self.crashed_nodes:
            return False
        return self.ping(helper, target)


# 시뮬레이션 실행
nodes = {
    f"node-{i}": SWIMNode(f"node-{i}", f"10.0.0.{i}") for i in range(1, 6)
}

# 초기 멤버십 세팅 (실제로는 gossip으로 전파되지만 시뮬레이션에서는 직접)
for node in nodes.values():
    for other in nodes.values():
        if other.node_id != node.node_id:
            node.members[other.node_id] = Member(other.node_id, other.address)

net = NetworkSimulator(nodes)
net.crash("node-3")  # node-3 장애 발생

# 프로토콜 주기 실행
for _ in range(5):
    for node_id, node in nodes.items():
        if node_id not in net.crashed_nodes:
            node.protocol_period(net)
    print("---")
```

### 예제 2: Incarnation 번호를 이용한 거짓 SUSPECT 반박 (Go 스타일 의사코드)

SWIM의 중요한 개선 사항 중 하나는 노드가 자신에 대한 거짓 SUSPECT 메시지를 반박할 수 있다는 것이다. Incarnation 번호를 증가시켜 "나는 살아있다"는 ALIVE 메시지를 더 높은 우선순위로 전파한다.

```go
// Go 스타일 의사코드 (실제 동작 코드 아님)

type State int
const (
    Alive   State = iota
    Suspect
    Dead
)

type Member struct {
    ID          string
    State       State
    Incarnation uint32   // 논리적 타임스탬프
}

type Node struct {
    ID          string
    Incarnation uint32
    Members     map[string]*Member
    EventBuf    []Event    // gossip 전파용 이벤트 버퍼
}

// 자신이 Suspect 메시지를 받으면 호출
func (n *Node) handleSuspect(msg SuspectMsg) {
    if msg.NodeID == n.ID {
        // 나 자신에 대한 Suspect → incarnation 증가 후 Alive 전파
        n.Incarnation++
        n.EventBuf = append(n.EventBuf, Event{
            Type:        "alive",
            NodeID:      n.ID,
            Incarnation: n.Incarnation,
        })
        fmt.Printf("[%s] 거짓 Suspect 반박: incarnation %d\n", n.ID, n.Incarnation)
        return
    }
    
    m, ok := n.Members[msg.NodeID]
    if !ok { return }
    
    // incarnation이 더 높은 경우에만 업데이트 (오래된 메시지 무시)
    if msg.Incarnation >= m.Incarnation {
        m.State = Suspect
        m.Incarnation = msg.Incarnation
        // 전파
        n.EventBuf = append(n.EventBuf, Event{
            Type:        "suspect",
            NodeID:      msg.NodeID,
            Incarnation: msg.Incarnation,
        })
    }
}

// Alive 메시지를 받으면 호출
func (n *Node) handleAlive(msg AliveMsg) {
    m, ok := n.Members[msg.NodeID]
    if !ok { return }
    
    // 더 최신 정보만 반영
    if msg.Incarnation > m.Incarnation ||
        (msg.Incarnation == m.Incarnation && m.State != Alive) {
        m.State = Alive
        m.Incarnation = msg.Incarnation
    }
}

// 매 프로토콜 주기마다 실행: gossip piggybacking
func (n *Node) doProtocolPeriod(target *Node) {
    // 핑 메시지에 최대 λ개의 gossip 이벤트 첨부
    gossipPayload := n.EventBuf[:min(3, len(n.EventBuf))]
    
    // ... 핑/핑-요청 로직 ...
    
    // gossip 이벤트 target에게 전달
    for _, event := range gossipPayload {
        target.receiveGossip(event)
    }
}
```

**Incarnation 번호의 역할:**
- `Suspect(node_x, incarnation=k)` 메시지가 퍼지면, node_x는 `Alive(node_x, incarnation=k+1)`을 전파해 반박할 수 있다
- incarnation이 더 낮은 메시지는 무시되므로 오래된 Suspect가 되살아나는 문제를 방지한다

---

## 주의사항과 팁

### 1. 오탐 vs 지연 감지 트레이드오프

SWIM에서 프로토콜 주기 T'와 SUSPECT 타임아웃을 어떻게 설정하느냐가 핵심이다.
- T'가 짧으면: 빠른 감지, 높은 네트워크 부하
- T'가 길면: 느린 감지, 낮은 부하

HashiCorp Consul의 기본값은 약 200ms ~ 1000ms 수준의 주기를 사용한다.

### 2. 네트워크 파티션 처리

SWIM은 네트워크 파티션(network partition) 상황에서 파티션 양쪽이 서로를 DEAD로 선언할 수 있다. 이를 완화하기 위해 HashiCorp의 **Lifeguard 확장**이 도입되었다. Lifeguard는 로컬 헬스 점수(LHA: Local Health Awareness)를 추적하고, 응답 지연이 일관되면 타임아웃을 늘려 불필요한 DEAD 선언을 줄인다.

### 3. Piggybacking의 전파 속도

SWIM의 전파는 감염(infection) 모델을 따른다. 각 노드가 프로토콜 주기마다 λ log n개의 이벤트를 피기백한다면 O(log n) 라운드 후 전체 클러스터에 전파된다.

직관: n명 중 k명이 이미 알고 있을 때, 다음 라운드에서 새로 아는 사람 수는 약 k(n-k)/n이다. 이는 SIR 전염병 모델과 동일한 수학이다.

### 4. HashiCorp Memberlist 실무 활용

실제 프로덕션에서는 직접 구현보다 **HashiCorp Memberlist**를 사용하는 것을 권장한다. SWIM 위에 전체 상태 동기화, Lifeguard 확장 등이 추가된 강건한 라이브러리다.

```go
import "github.com/hashicorp/memberlist"

config := memberlist.DefaultLocalConfig()
config.Name = "node-1"
config.BindAddr = "0.0.0.0"
config.BindPort = 7946

list, err := memberlist.Create(config)
if err != nil { panic(err) }

// 기존 노드에 조인
_, err = list.Join([]string{"10.0.0.1:7946"})

// 현재 살아 있는 멤버 목록
for _, member := range list.Members() {
    fmt.Printf("Member: %s %s\n", member.Name, member.Addr)
}
```

---

## 정리

SWIM 프로토콜은 분산 시스템에서 멤버십 관리라는 고전적인 문제를 우아하게 해결한다. O(n²) 네트워크 부하 없이 O(n)으로 확장 가능하면서, 간접 핑과 Incarnation 번호를 통해 거짓 긍정을 크게 줄였다. Consul, Serf, Cassandra 같은 실전 시스템이 이 프로토콜을 선택한 이유가 바로 여기에 있다. 클러스터 규모가 수십 노드를 넘어선다면, SWIM 기반 멤버십 라이브러리 도입을 진지하게 고려하라.

## 참고 자료
- [GitHub - hashicorp/memberlist: SWIM 기반 Golang 멤버십 라이브러리](https://github.com/hashicorp/memberlist)
- [GitHub - marciamart/Real-time-scheduling-Algorithm: 스케줄링 알고리즘 구현 참고](https://github.com/marciamart/Real-time-scheduling-Algorithm)
