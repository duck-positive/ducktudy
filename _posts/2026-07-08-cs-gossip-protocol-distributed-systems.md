---
layout: post
title: "Gossip 프로토콜 완전 정복: 리더 없이 수천 개의 노드가 서로를 아는 방법"
date: 2026-07-08
categories: [cs, computer-science]
tags: [gossip-protocol, distributed-systems, cassandra, membership, failure-detection, epidemic-protocol, phi-accrual]
---

## 개념 설명

**Gossip 프로토콜(Gossip Protocol)**은 분산 시스템에서 노드들이 중앙 코디네이터 없이 정보를 전파하는 방식입니다. 사람들이 소문을 퍼뜨리는 방식처럼, 각 노드가 주기적으로 랜덤한 이웃 노드들에게 자신이 알고 있는 상태를 전달합니다. 전염병이 퍼지는 것과 유사하여 **epidemic 프로토콜**이라고도 부릅니다.

Gossip의 가장 중요한 특성은 **결과적 일관성(eventual consistency)**입니다. 네트워크의 일부가 실패해도, 충분한 시간이 지나면 모든 노드가 동일한 상태를 갖게 됩니다. Cassandra, DynamoDB, Redis Cluster, Consul, Serf 등 현대적인 분산 시스템 대부분이 Gossip을 클러스터 멤버십 관리의 기반으로 사용합니다.

### 핵심 동작 원리

각 노드는 다른 노드들에 대한 **상태 벡터(state vector)**를 유지합니다. 상태에는 노드 ID, IP 주소, 포트, 세대 번호(generation), 버전(heartbeat counter) 등이 포함됩니다.

**Gossip 한 라운드의 절차:**
1. 각 노드가 주기적으로(Cassandra는 기본 1초) 타이머를 울림
2. 클러스터 내 랜덤한 1~3개의 노드를 선택
3. 자신이 알고 있는 전체 멤버십 상태를 SYN 메시지로 전송
4. 수신 노드가 ACK로 자신의 상태를 응답
5. 양쪽이 서로의 최신 정보로 상태를 업데이트

정보가 N개의 노드에 퍼지는 데 걸리는 시간은 **O(log N)**입니다. 노드 수가 1000개에서 100만 개로 늘어도 수십 라운드 내에 모든 노드가 업데이트됩니다.

### 장단점

| 항목 | 내용 |
|------|------|
| 내장애성 | 노드/네트워크 장애에 강함 (단일 장애점 없음) |
| 확장성 | 노드 추가가 O(log N) 라운드 내 전파 |
| 수렴 보장 | 충분한 시간 후 모든 노드가 동일 상태 |
| 정확도 | 결과적 일관성 — 일시적 불일치 발생 가능 |
| 대역폭 | 노드 수에 비례하는 지속적 백그라운드 트래픽 |

---

## 왜 필요한가

**전통적인 중앙집중형 멤버십 관리의 문제점:**

- **단일 장애점(SPOF)**: 중앙 레지스트리 서버가 죽으면 전체 클러스터가 멈춤
- **확장성 병목**: 1000개 노드가 중앙 서버에 하트비트를 보내면 초당 1000건의 요청 처리 필요
- **네트워크 파티션 취약**: 중앙 서버와의 연결이 끊기면 정상 노드도 "죽은 것"으로 판단

**Gossip이 해결하는 문제:**

1. **멤버십 관리**: 클러스터에 새 노드가 합류하거나 나갈 때 전체에 전파
2. **장애 감지**: 특정 노드가 응답하지 않으면 여러 노드가 관찰하여 합의로 결정
3. **스키마/설정 전파**: Cassandra의 keyspace, 테이블 스키마 변경을 전체에 배포
4. **Anti-Entropy**: 복제본 간 데이터 불일치를 감지하고 수리 (Merkle 트리와 함께 사용)

---

## 실제 구현 예제

### 예제 1: 기본 Gossip 노드 구현 (Python)

```python
import random
import time
import threading
from dataclasses import dataclass, field
from typing import Dict, List

@dataclass
class NodeState:
    node_id: str
    address: str
    generation: int = 0   # 노드 재시작 시 증가
    heartbeat: int = 0    # 매 gossip 라운드마다 증가
    is_alive: bool = True

class GossipNode:
    def __init__(self, node_id: str, address: str, seed_nodes: List[str]):
        self.state = NodeState(node_id=node_id, address=address)
        self.membership: Dict[str, NodeState] = {node_id: self.state}
        self.seed_nodes = seed_nodes
        self.lock = threading.Lock()
        self._running = False
    
    def start(self):
        self._running = True
        # Bootstrap: seed 노드에서 초기 멤버십 획득
        for seed in self.seed_nodes:
            self._bootstrap(seed)
        # 주기적 gossip 시작
        threading.Thread(target=self._gossip_loop, daemon=True).start()
        threading.Thread(target=self._heartbeat_loop, daemon=True).start()
    
    def _heartbeat_loop(self):
        """매 초마다 자신의 heartbeat 증가"""
        while self._running:
            with self.lock:
                self.state.heartbeat += 1
            time.sleep(1.0)
    
    def _gossip_loop(self):
        """매 1초마다 랜덤 노드에 상태 전파"""
        while self._running:
            time.sleep(1.0)
            with self.lock:
                peers = [nid for nid in self.membership
                         if nid != self.state.node_id
                         and self.membership[nid].is_alive]
            
            # 팬아웃: 최대 3개 노드에 전파 (fanout = 3)
            gossip_targets = random.sample(peers, min(3, len(peers)))
            for target_id in gossip_targets:
                self._send_gossip(target_id)
    
    def _send_gossip(self, target_id: str):
        """타겟 노드에 현재 멤버십 상태 전송 (실제 구현에서는 네트워크 호출)"""
        with self.lock:
            snapshot = {nid: NodeState(**vars(state))
                        for nid, state in self.membership.items()}
        # 실제로는 gRPC, HTTP 등으로 전송
        # 여기서는 직접 merge 호출로 시뮬레이션
        target = self.membership.get(target_id)
        if target:
            # target._receive_gossip(snapshot) 형태
            pass
    
    def _receive_gossip(self, remote_states: Dict[str, NodeState]):
        """수신한 상태를 자신의 멤버십 테이블에 병합"""
        with self.lock:
            for node_id, remote_state in remote_states.items():
                local_state = self.membership.get(node_id)
                
                if local_state is None:
                    # 새로운 노드 발견
                    self.membership[node_id] = remote_state
                    print(f"[{self.state.node_id}] 새 노드 발견: {node_id}")
                elif remote_state.generation > local_state.generation:
                    # 재시작된 노드 (세대 번호가 더 높음)
                    self.membership[node_id] = remote_state
                elif (remote_state.generation == local_state.generation
                      and remote_state.heartbeat > local_state.heartbeat):
                    # 더 최신 heartbeat
                    local_state.heartbeat = remote_state.heartbeat
                    local_state.is_alive = True
    
    def _bootstrap(self, seed_address: str):
        """시드 노드에서 초기 멤버십 정보 요청"""
        print(f"[{self.state.node_id}] 시드 노드 {seed_address}에 bootstrap 요청")
    
    def mark_suspected(self, node_id: str):
        """특정 노드를 의심 상태로 표시"""
        with self.lock:
            if node_id in self.membership:
                self.membership[node_id].is_alive = False
                print(f"[{self.state.node_id}] 노드 {node_id} 의심 상태로 표시")
    
    def get_live_nodes(self) -> List[str]:
        with self.lock:
            return [nid for nid, state in self.membership.items()
                    if state.is_alive]
```

---

### 예제 2: Phi Accrual 장애 감지기 (Cassandra 방식)

단순 타임아웃 기반 장애 감지의 문제는 네트워크 지연이 가변적일 때 오탐(false positive)이 많다는 것입니다. Cassandra가 사용하는 **φ (phi) accrual detector**는 최근 heartbeat 도착 간격을 통계적으로 분석하여 "의심 수준"을 연속적인 실수 값으로 표현합니다.

```python
import math
from collections import deque

class PhiAccrualDetector:
    """
    φ 값 해석:
      φ < 1:  거의 확실히 살아있음
      φ = 8:  Cassandra 기본 임계값 (99.9% 신뢰도로 죽었다 판단)
      φ = 16: 더 높은 확신 (false positive 거의 없음)
    """
    
    def __init__(self, threshold: float = 8.0, window_size: int = 1000,
                 min_std_dev_ms: float = 200):
        self.threshold = threshold
        self.min_std_dev_ms = min_std_dev_ms
        self.intervals: deque = deque(maxlen=window_size)
        self.last_arrival_time: float = None
    
    def heartbeat(self, current_time_ms: float):
        """새 heartbeat 수신 시 호출"""
        if self.last_arrival_time is not None:
            interval = current_time_ms - self.last_arrival_time
            self.intervals.append(interval)
        self.last_arrival_time = current_time_ms
    
    def _mean(self) -> float:
        return sum(self.intervals) / len(self.intervals)
    
    def _std_dev(self) -> float:
        if len(self.intervals) < 2:
            return self.min_std_dev_ms
        mean = self._mean()
        variance = sum((x - mean) ** 2 for x in self.intervals) / len(self.intervals)
        return max(math.sqrt(variance), self.min_std_dev_ms)
    
    def _probability_later(self, elapsed_ms: float) -> float:
        """
        elapsed 시간 이후에도 heartbeat가 도착하지 않을 확률 (지수 분포 가정)
        정규분포로 근사하여 계산
        """
        if not self.intervals:
            return 1.0
        mean = self._mean()
        std = self._std_dev()
        # 정규분포의 누적분포함수(CDF) 활용: P(X > elapsed)
        y = (elapsed_ms - mean) / std
        # erfc(y/√2)/2 ≈ P(정규변수 > y)
        return 0.5 * math.erfc(y / math.sqrt(2))
    
    def phi(self, current_time_ms: float) -> float:
        """
        현재 시각에서의 φ 값 반환.
        마지막 heartbeat로부터 얼마나 오래됐는지 기반으로 계산.
        """
        if self.last_arrival_time is None or not self.intervals:
            return 0.0
        elapsed = current_time_ms - self.last_arrival_time
        prob = self._probability_later(elapsed)
        # 확률이 매우 작아지면 φ가 커짐
        if prob == 0:
            return float('inf')
        return -math.log10(prob)
    
    def is_available(self, current_time_ms: float) -> bool:
        return self.phi(current_time_ms) < self.threshold

# 사용 예
detector = PhiAccrualDetector(threshold=8.0)
now = 1000.0

# 정상 heartbeat 시뮬레이션 (1초 간격)
for i in range(20):
    detector.heartbeat(now + i * 1000)

# 21번째 heartbeat 지연 — 2.5초 경과 후 φ 확인
check_time = now + 20 * 1000 + 2500
phi_val = detector.phi(check_time)
print(f"φ = {phi_val:.2f}")                      # 약 2~3 (아직 살아있다 판단)
print(f"살아있음: {detector.is_available(check_time)}")  # True

# 10초 경과 후
check_time2 = now + 20 * 1000 + 10000
phi_val2 = detector.phi(check_time2)
print(f"φ = {phi_val2:.2f}")                       # 8 이상 (죽었다 판단)
print(f"살아있음: {detector.is_available(check_time2)}") # False
```

φ가 1씩 커질 때마다 "살아있을 확률"이 10배씩 낮아집니다. φ=8이면 10^-8, 즉 1억 분의 1 확률로 살아있다는 의미입니다.

---

## 주의사항 및 팁

**1. 시드(Seed) 노드 설계**

시드 노드는 부트스트랩 진입점일 뿐, 특별한 역할을 갖지 않습니다. 모든 노드가 시드가 될 수 있으며, 최소 2~3개를 다른 랙/AZ에 배치해야 합니다. 시드 노드가 모두 죽어있으면 새 노드가 합류할 수 없습니다.

**2. 팬아웃(Fanout) 수치 선택**

팬아웃이 클수록 수렴이 빠르지만 트래픽이 증가합니다. 팬아웃 k일 때 정보는 O(log_k N) 라운드에 퍼집니다. 실전에서는 k=3이 흔합니다.

**3. 세대 번호(Generation)로 재시작 노드 구별**

같은 IP의 노드가 재시작되면 heartbeat 카운터가 0으로 초기화됩니다. 세대 번호(generation = 현재 Unix timestamp)를 이용하면 이전 인스턴스와 새 인스턴스를 구분할 수 있습니다.

**4. 가십 vs. 반가십(Anti-Entropy)**

- **가십 전파**: 새 정보를 빠르게 퍼뜨림
- **Anti-entropy**: 주기적으로 두 노드가 전체 상태를 비교하여 불일치를 수정

Cassandra는 가십으로 멤버십을 관리하고, Merkle 트리 기반 anti-entropy로 데이터 복제본을 수리합니다.

**5. 네트워크 파티션 시 동작**

두 파티션이 분리되면 각자 정상적으로 동작하다가 파티션이 해소되면 상태 벡터의 타임스탬프/버전 번호로 충돌을 해소합니다. "최신 버전이 이긴다(last-write-wins)" 정책을 많이 씁니다.

**6. 보안 고려사항**

Gossip 트래픽은 인증/암호화 없이 주고받으면 악의적인 노드가 거짓 상태를 전파할 수 있습니다. 실제 구현에서는 TLS와 상호 TLS(mTLS) 인증을 적용하세요.

---

## 참고 자료
- [Gossip Protocol Explained - High Scalability](https://highscalability.com/gossip-protocol-explained/)
- [Gossip Protocol in Distributed Systems - GeeksforGeeks](https://www.geeksforgeeks.org/distributed-systems/gossip-protocol-in-disrtibuted-systems/)
- [Cassandra Gossip Protocol and Internode Messaging - AxonOps](https://axonops.com/docs/data-platforms/cassandra/architecture/cluster-management/gossip/)
- [Gossip Protocols: How Services Discover and Share State - Medium](https://hosseinnejati.medium.com/gossip-protocols-how-services-discover-and-share-state-f4479bc6ac50)
