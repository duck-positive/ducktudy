---
layout: post
title: "Kademlia DHT 완전 정복: XOR 메트릭으로 설계한 분산 해시 테이블의 내부 구조"
date: 2026-10-31
categories: [cs, computer-science]
tags: [kademlia, dht, 분산시스템, p2p, 네트워크, 해시테이블, xor]
---

## 개요

Kademlia는 2002년 Petar Maymounkov와 David Mazières가 발표한 분산 해시 테이블(DHT, Distributed Hash Table) 프로토콜이다. BitTorrent의 트래커리스(trackerless) 네트워크, IPFS, Ethereum, Storj 등 수십억 노드 규모의 분산 시스템이 이 알고리즘에 의존한다. Kademlia의 독창성은 **XOR 거리 메트릭**이라는 수학적으로 우아한 개념에서 출발한다. 이 글에서는 Kademlia의 라우팅 알고리즘, k-버킷 구조, 노드 조회 프로토콜, 실제 구현을 깊이 다룬다.

---

## 왜 분산 해시 테이블이 필요한가?

중앙 서버 없이 수백만 피어가 데이터를 저장하고 조회하려면 다음 문제를 해결해야 한다.

- **어떤 노드가 어떤 데이터를 보유하는가?** (책임 분산)
- **원하는 데이터를 어떻게 효율적으로 찾는가?** (라우팅)
- **노드가 수시로 참여하고 떠날 때 어떻게 안정적으로 동작하는가?** (churn 대응)

Chord는 링 구조로, Pastry는 접두사 기반으로, CAN은 n차원 공간으로 접근했다. Kademlia는 이 중 가장 실용적이고 구현이 단순한 XOR 기반 트리 접근법을 택했다.

---

## XOR 거리 메트릭

Kademlia의 핵심은 노드 ID 공간을 이진 트리로 표현하고, 두 노드 간 거리를 **비트단위 XOR**로 정의하는 것이다.

```
distance(a, b) = a XOR b
```

이 정의는 다음 수학적 성질을 만족한다.

1. **d(x, y) = 0 iff x = y** — 자기 자신과의 거리는 0
2. **d(x, y) = d(y, x)** — 대칭성
3. **d(x, z) ≤ d(x, y) + d(y, z)** — 삼각 부등식 (XOR은 실제로 더 강한 d(x,z) ≤ max(d(x,y), d(y,z)) 만족)

XOR의 특별한 점은 **단방향성(unidirectionality)**이다: d(x, y)가 주어졌을 때 y는 유일하게 결정된다. Chord처럼 링에서 방향성 문제가 없고, 각 비트 위치에서 이진 트리를 내려가면 자연스럽게 대상 노드에 수렴한다.

---

## k-버킷 라우팅 테이블

각 Kademlia 노드는 **k-버킷(k-bucket)**이라는 160개의 리스트를 유지한다 (노드 ID가 160비트일 때). i번째 버킷은 자신과 XOR 거리가 [2^i, 2^(i+1)) 범위에 있는 노드 정보를 최대 k개 저장한다 (보통 k=20).

```
나 자신: 00110101...
버킷 0:  자신과 첫 159비트가 같고 마지막 비트가 다른 노드
버킷 1:  158번째 비트가 처음 다른 노드
버킷 2:  157번째 비트가 처음 다른 노드
...
버킷 i:  (160-i)번째 비트가 처음 다른 노드
```

### k-버킷 갱신 정책

새 노드를 만나면:
1. 해당 버킷에 빈자리가 있으면 바로 추가
2. 가득 찼으면 버킷의 가장 오래된 노드(head)에게 PING을 보낸다
3. 응답하면 기존 노드 유지, 새 노드 폐기 (오래된 노드가 더 안정적이라는 경험칙)
4. 응답 없으면 기존 노드 삭제 후 새 노드 추가

이 정책 덕분에 Kademlia의 라우팅 테이블은 **보수적이고 안정적**이다. 많이 연결될수록 더 오래 살아남은 노드들이 남는다.

---

## 네 가지 RPC

Kademlia 프로토콜은 단 4개의 원격 프로시저 호출로 이루어진다.

| RPC | 인자 | 반환 | 역할 |
|-----|------|------|------|
| PING | nodeID | nodeID | 생존 여부 확인 |
| STORE | key, value | — | 데이터 저장 |
| FIND_NODE | targetID | k개 노드 | 대상에 가장 가까운 k개 노드 |
| FIND_VALUE | key | value 또는 k개 노드 | 데이터 조회 |

---

## 노드 조회 알고리즘 (Iterative Lookup)

대상 노드 ID t를 찾는 과정:

1. 자신의 라우팅 테이블에서 t에 가장 가까운 α개 노드를 선택 (보통 α=3)
2. 선택된 노드들에게 병렬로 FIND_NODE(t) 전송
3. 응답 받은 k개 노드 중 이전보다 t에 더 가까운 노드가 있으면 그 노드들에게 다시 FIND_NODE 전송
4. 더 가까운 노드를 찾지 못할 때까지 반복
5. 최종적으로 t에 가장 가까운 k개 노드 반환

이 과정은 **O(log n)번의 라운드**로 수렴한다. 각 라운드에서 최소 한 비트 이상 대상에 가까워지기 때문이다.

---

## 구현 예제 1: 기본 Kademlia 노드 (Python)

```python
import hashlib
import os
import bisect
from dataclasses import dataclass, field
from typing import List, Optional, Dict, Tuple

K = 20       # k-버킷 크기
ALPHA = 3    # 동시 조회 수
ID_BITS = 160

def random_id() -> int:
    return int.from_bytes(os.urandom(20), 'big')

def xor_distance(a: int, b: int) -> int:
    return a ^ b

def bucket_index(self_id: int, other_id: int) -> int:
    """자신과 다른 노드 간의 XOR 거리의 최상위 비트 위치"""
    dist = xor_distance(self_id, other_id)
    if dist == 0:
        return -1  # 자기 자신
    return dist.bit_length() - 1  # 0-indexed, 0이 가장 가까움

@dataclass
class NodeInfo:
    node_id: int
    address: Tuple[str, int]  # (IP, port)
    
    def __hash__(self):
        return hash(self.node_id)
    
    def __eq__(self, other):
        return self.node_id == other.node_id

class KBucket:
    def __init__(self, k=K):
        self.k = k
        self.nodes: List[NodeInfo] = []  # 최신이 뒤쪽
        self.replacement_cache: List[NodeInfo] = []
    
    def add(self, node: NodeInfo) -> Optional[NodeInfo]:
        """
        노드 추가. 버킷이 가득 찼으면 가장 오래된 노드를 반환(PING 필요).
        PING 응답 없으면 remove_oldest()로 제거 후 재시도.
        """
        # 이미 존재하면 맨 뒤로 이동 (LRU 갱신)
        for i, n in enumerate(self.nodes):
            if n.node_id == node.node_id:
                self.nodes.pop(i)
                self.nodes.append(node)
                return None
        
        if len(self.nodes) < self.k:
            self.nodes.append(node)
            return None
        else:
            # 가장 오래된 노드(head) PING 필요
            return self.nodes[0]
    
    def remove_oldest(self):
        if self.nodes:
            self.nodes.pop(0)
    
    def get_closest(self, target_id: int, count: int) -> List[NodeInfo]:
        return sorted(self.nodes, key=lambda n: xor_distance(n.node_id, target_id))[:count]

class RoutingTable:
    def __init__(self, self_id: int):
        self.self_id = self_id
        self.buckets: List[KBucket] = [KBucket() for _ in range(ID_BITS)]
    
    def add_node(self, node: NodeInfo) -> Optional[NodeInfo]:
        idx = bucket_index(self.self_id, node.node_id)
        if idx < 0:
            return None  # 자기 자신
        return self.buckets[idx].add(node)
    
    def get_closest(self, target_id: int, count: int = K) -> List[NodeInfo]:
        """target_id에 가장 가까운 count개 노드"""
        candidates = []
        for bucket in self.buckets:
            candidates.extend(bucket.nodes)
        candidates.sort(key=lambda n: xor_distance(n.node_id, target_id))
        return candidates[:count]
    
    def find_node(self, target_id: int) -> Optional[NodeInfo]:
        idx = bucket_index(self.self_id, target_id)
        if idx < 0:
            return NodeInfo(self.self_id, ('localhost', 0))
        for node in self.buckets[idx].nodes:
            if node.node_id == target_id:
                return node
        return None

class KademliaNode:
    def __init__(self, address: Tuple[str, int]):
        self.node_id = random_id()
        self.address = address
        self.routing_table = RoutingTable(self.node_id)
        self.storage: Dict[int, bytes] = {}
    
    def store(self, key: bytes, value: bytes):
        """key를 SHA-1 해시하여 정수 ID로 저장"""
        key_id = int.from_bytes(hashlib.sha1(key).digest(), 'big')
        self.storage[key_id] = value
    
    def local_find_value(self, key: bytes) -> Optional[bytes]:
        key_id = int.from_bytes(hashlib.sha1(key).digest(), 'big')
        return self.storage.get(key_id)
    
    def info(self):
        total = sum(len(b.nodes) for b in self.routing_table.buckets)
        return f"NodeID={self.node_id:#x}, known peers={total}"

# 시뮬레이션: 3개 노드 간 라우팅 테이블 구성
node_a = KademliaNode(('127.0.0.1', 8468))
node_b = KademliaNode(('127.0.0.1', 8469))
node_c = KademliaNode(('127.0.0.1', 8470))

# 서로 알게 됨 (실제로는 PING 교환 후)
info_a = NodeInfo(node_a.node_id, node_a.address)
info_b = NodeInfo(node_b.node_id, node_b.address)
info_c = NodeInfo(node_c.node_id, node_c.address)

node_a.routing_table.add_node(info_b)
node_a.routing_table.add_node(info_c)
node_b.routing_table.add_node(info_a)
node_c.routing_table.add_node(info_a)

# A가 임의의 target_id에 가장 가까운 노드를 찾음
target = random_id()
closest = node_a.routing_table.get_closest(target, 2)
print(f"Target: {target:#x}")
for n in closest:
    print(f"  Closest: {n.node_id:#x}, dist={xor_distance(n.node_id, target):#x}")
```

---

## 구현 예제 2: Iterative FIND_NODE 시뮬레이션

```python
from collections import OrderedDict
import heapq

class KademliaNetwork:
    """테스트용 시뮬레이션 네트워크: 모든 노드가 같은 프로세스에 있음"""
    
    def __init__(self):
        self.nodes: Dict[int, KademliaNode] = {}
    
    def add_node(self, node: KademliaNode):
        self.nodes[node.node_id] = node
    
    def find_node_rpc(self, requester_id: int, responder_id: int, target_id: int) -> List[NodeInfo]:
        """FIND_NODE RPC 시뮬레이션"""
        responder = self.nodes.get(responder_id)
        if not responder:
            return []
        # 요청자를 라우팅 테이블에 추가 (실제 Kademlia의 부수 효과)
        requester = self.nodes.get(requester_id)
        if requester:
            responder.routing_table.add_node(NodeInfo(requester_id, requester.address))
        return responder.routing_table.get_closest(target_id, K)
    
    def iterative_find_node(self, start_id: int, target_id: int) -> List[NodeInfo]:
        """
        반복적 노드 조회. O(log n) 라운드로 수렴.
        반환: target_id에 가장 가까운 k개 노드
        """
        # (거리, NodeInfo) 힙으로 관리
        start = self.nodes[start_id]
        initial = start.routing_table.get_closest(target_id, ALPHA)
        
        # 조회 상태
        contacted = set()        # 이미 질의한 노드
        seen = {}                # node_id -> (distance, NodeInfo)
        
        for ni in initial:
            d = xor_distance(ni.node_id, target_id)
            seen[ni.node_id] = (d, ni)
        
        while True:
            # 아직 질의 안 한 노드 중 가장 가까운 ALPHA개 선택
            candidates = [
                (d, ni) for node_id, (d, ni) in seen.items()
                if node_id not in contacted
            ]
            candidates.sort(key=lambda x: x[0])
            to_query = candidates[:ALPHA]
            
            if not to_query:
                break
            
            closest_before = min(seen.values(), key=lambda x: x[0])[0] if seen else float('inf')
            
            found_closer = False
            for _, ni in to_query:
                contacted.add(ni.node_id)
                results = self.find_node_rpc(start_id, ni.node_id, target_id)
                for r in results:
                    d = xor_distance(r.node_id, target_id)
                    if r.node_id not in seen or seen[r.node_id][0] > d:
                        seen[r.node_id] = (d, r)
                        if d < closest_before:
                            found_closer = True
            
            if not found_closer:
                break  # 더 이상 가까운 노드를 찾지 못함
        
        # 최종 결과: target_id에 가장 가까운 k개
        all_nodes = sorted(seen.values(), key=lambda x: x[0])
        return [ni for _, ni in all_nodes[:K]]

# 50개 노드 네트워크 시뮬레이션
network = KademliaNetwork()
nodes = [KademliaNode(('127.0.0.1', 8000 + i)) for i in range(50)]
for node in nodes:
    network.add_node(node)

# 부트스트래핑: 각 노드가 첫 번째 노드를 알고 시작
bootstrap = nodes[0]
for node in nodes[1:]:
    node.routing_table.add_node(NodeInfo(bootstrap.node_id, bootstrap.address))
    bootstrap.routing_table.add_node(NodeInfo(node.node_id, node.address))

# 노드 0이 랜덤 타겟 조회
target_id = random_id()
result = network.iterative_find_node(nodes[0].node_id, target_id)
print(f"조회 완료. 가장 가까운 {len(result)}개 노드:")
for ni in result[:3]:
    print(f"  {ni.node_id:#042x}, dist={xor_distance(ni.node_id, target_id):#042x}")
```

---

## Kademlia의 수렴 증명

**정리**: n개 노드로 이루어진 Kademlia 네트워크에서 FIND_NODE 조회는 O(log n) 라운드 안에 수렴한다.

**증명 스케치**: XOR 거리 공간에서 각 라운드는 대상까지 남은 XOR 거리의 최상위 비트를 최소 1개 이상 제거한다. n개 노드의 ID는 160비트이므로 라운드 수는 최대 log₂(n)이다. 더 정확히는 α-병렬 조회 덕분에 (1+1/α)log n 라운드로 바운드된다.

---

## BitTorrent DHT와의 차이

BitTorrent DHT(BEP 5)는 Kademlia 기반이지만 몇 가지 변형이 있다.

| 항목 | Kademlia 원본 | BitTorrent DHT |
|------|----------------|----------------|
| STORE RPC | 있음 | 없음 (announce_peer 사용) |
| 토큰 기반 쓰기 | 없음 | 있음 (스팸 방지) |
| 노드 ID 비트 수 | 160 (SHA-1) | 160 |
| 데이터 재발행 | 원본 퍼블리셔 책임 | 없음 (announce만) |

---

## 주의사항과 팁

1. **Sybil 공격 취약성**: Kademlia는 노드 ID를 자유롭게 선택할 수 있어 공격자가 특정 키 공간을 장악할 수 있다. Ethereum은 ID를 공개키 해시로 강제한다.

2. **Eclipse 공격**: 악의적 노드가 피해자의 라우팅 테이블을 모두 채울 수 있다. 버킷 다양성 강제, 랜덤 키 발급으로 대응한다.

3. **Churn 대응**: k=20처럼 큰 버킷은 노드 이탈에도 충분한 백업 경로를 확보한다. 실제로 인터넷 노드의 평균 세션은 수시간이다.

4. **재발행(Republish)**: 저장된 데이터는 24시간마다 재발행해야 한다. 데이터를 저장한 k개 노드 중 일부가 떠났을 때를 대비한다.

5. **부트스트래핑**: 신규 노드는 미리 알려진 "부트스트랩 노드" 주소를 하드코딩하거나 DNS 시드를 통해 얻는다.

---

## 참고 자료
- [Kademlia — Wikipedia](https://en.wikipedia.org/wiki/Kademlia)
- [Kademlia: A Peer-to-peer Information System Based on the XOR Metric (원논문)](https://pdos.csail.mit.edu/6.824/papers/kademlia.pdf)
- [Kademlia and its use in decentralized platforms — Storj Blog](https://storj.io/blog/a-brief-overview-of-kademlia-and-its-use-in-various-decentralized-platforms)
- [BitTorrent DHT Protocol BEP 5](https://www.bittorrent.org/beps/bep_0005.html)
