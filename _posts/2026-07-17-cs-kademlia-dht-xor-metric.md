---
layout: post
title: "Kademlia DHT 완전 정복: XOR 메트릭으로 P2P 네트워크를 정복하는 분산 해시 테이블"
date: 2026-07-17
categories: [cs, computer-science]
tags: [kademlia, DHT, distributed-hash-table, P2P, XOR, BitTorrent, IPFS, ethereum]
---

## 개념 설명

비트토렌트(BitTorrent)로 파일을 내려받을 때, 또는 IPFS에서 콘텐츠를 검색할 때, 이더리움 노드들이 서로를 발견할 때 — 이 모든 과정의 이면에는 **Kademlia**가 동작하고 있습니다. Kademlia는 2002년 Petar Maymounkov와 David Mazières가 발표한 논문 "Kademlia: A Peer-to-peer Information System Based on the XOR Metric"에서 소개된 분산 해시 테이블(DHT, Distributed Hash Table)입니다.

이전에 살펴본 **Chord DHT**가 링(ring) 위에서 일방향 거리를 사용한다면, Kademlia는 **XOR 거리**라는 독창적인 메트릭을 통해 노드 탐색을 수행합니다. 이 선택이 Chord 대비 구조적으로 더 단순하고, 실용적으로 더 견고하며, 구현하기 쉬운 DHT를 탄생시켰습니다.

### 핵심 개념 1: XOR 거리(XOR Metric)

Kademlia에서 모든 노드와 모든 키는 동일한 160비트(또는 256비트) 식별자 공간에 존재합니다. 두 식별자 A, B 사이의 "거리"는 단순히 `A XOR B`입니다. XOR 메트릭의 수학적 특성:

1. **대칭성**: `d(A, B) = d(B, A)` (비트토렌트 Chord와의 결정적 차이)
2. **삼각 부등식**: `d(A, B) ≤ d(A, C) XOR d(C, B)`
3. **단방향성**: A에서 봤을 때 자신에게 "가까운" 노드들이, 그 노드들 관점에서도 A에 가깝습니다.

이 대칭성 덕분에 Kademlia는 노드 탐색 경로가 양방향으로 일관되어, 실패한 경로를 쉽게 캐시에서 제거할 수 있습니다.

### 핵심 개념 2: k-버킷 라우팅 테이블

각 노드는 자신과의 XOR 거리를 기준으로 **160개의 버킷(k-bucket)**을 가집니다. 버킷 i는 XOR 거리가 [2^i, 2^(i+1)) 범위인 노드들을 저장하며, 각 버킷에는 최대 k개(보통 k=20)의 노드를 LRU 순서로 유지합니다.

```
버킷 0: 거리 [1, 2)    → 1개 노드  (자신과 최상위 비트 제외 동일)
버킷 1: 거리 [2, 4)    → 최대 k개
버킷 2: 거리 [4, 8)    → 최대 k개
버킷 7: 거리 [128, 256)→ 최대 k개
...
버킷 159: 가장 먼 노드 그룹
```

**가까운 노드에 대해 더 정밀한 정보를 유지**하고, 먼 노드에 대해서는 대략적인 정보만 가집니다. 이는 자연스러운 정보 지역성입니다.

### 핵심 개념 3: 네 가지 RPC

Kademlia 노드는 네 가지 원격 프로시저 호출(RPC)만으로 운영됩니다.

| RPC | 설명 |
|-----|------|
| `PING` | 노드 생존 확인 |
| `STORE(key, value)` | 특정 노드에 키-값 쌍 저장 |
| `FIND_NODE(target_id)` | target_id에 가장 가까운 k개 노드 반환 |
| `FIND_VALUE(key)` | 키에 해당하는 값 반환, 없으면 FIND_NODE처럼 동작 |

---

## 왜 필요한가

분산 시스템에서 "이 키를 어느 노드가 담당하는가?"라는 질문에 답하려면 전역 상태를 관리하는 중앙 서버가 필요합니다. 하지만 중앙 서버는 단일 장애점(SPOF)이며, 수억 명이 사용하는 BitTorrent에서는 비용도 불가능합니다.

**DHT의 목표**: 중앙 서버 없이, N개의 노드가 분산적으로 데이터를 저장·조회하되, O(log N) 메시지로 임의의 키를 찾는 것.

Kademlia가 선택받은 이유:
- **대칭 XOR 거리**: 노드 탐색 경로를 양방향으로 최적화
- **k-버킷 + LRU 교체**: 오래된 노드 정보를 자동으로 신선한 정보로 교체
- **동시 병렬 탐색(α-parallel)**: α개 노드에 병렬 FIND_NODE를 보내 지연 시간 단축
- **노드 이탈에 강함(churn tolerance)**: 복제(replication) + 주기적 STORE로 데이터 지속성 보장

실제 채택 사례:
- **BitTorrent DHT**: 마그넷 링크에서 토렌트 메타데이터 위치를 추적
- **IPFS**: 콘텐츠 주소(CID) 기반 데이터 위치 탐색
- **Ethereum**: devp2p 프로토콜의 노드 발견(Node Discovery v4/v5)
- **I2P 네트워크**: 익명 P2P 통신
- **OpenDHT (Jami)**: 오픈소스 분산 전화번호부

---

## 실제 구현 예제

### 예제 1: Kademlia 핵심 구조 Python 구현

```python
import hashlib
import random
from dataclasses import dataclass, field
from typing import Optional

# 식별자 길이 (비트)
ID_BITS = 160

def sha1_id(data: str) -> int:
    """문자열의 SHA-1 해시를 160비트 정수로 반환"""
    return int(hashlib.sha1(data.encode()).hexdigest(), 16)

def xor_distance(a: int, b: int) -> int:
    return a ^ b

def common_prefix_length(a: int, b: int) -> int:
    """a XOR b의 선행 0비트 수 (= 공통 상위 비트 수)"""
    d = xor_distance(a, b)
    if d == 0:
        return ID_BITS
    return ID_BITS - d.bit_length()


@dataclass
class KNode:
    node_id: int
    address: str  # "ip:port" 형식 (시뮬레이션용)

    def __hash__(self):
        return self.node_id

    def __eq__(self, other):
        return self.node_id == other.node_id

    def __repr__(self):
        return f"Node({self.node_id:#x}, {self.address})"


class KBucket:
    """단일 k-버킷: 최대 k개 노드를 LRU 순서로 유지"""

    def __init__(self, k: int = 20):
        self.k = k
        self.nodes: list[KNode] = []  # 가장 오래된 것이 앞

    def add(self, node: KNode) -> bool:
        """노드 추가. 이미 있으면 맨 뒤로 이동(최근 사용). 
        가득 찼으면 False 반환 (실제 구현: 맨 앞 노드에 PING 후 응답 없으면 교체)"""
        for i, n in enumerate(self.nodes):
            if n.node_id == node.node_id:
                self.nodes.pop(i)
                self.nodes.append(node)
                return True
        if len(self.nodes) < self.k:
            self.nodes.append(node)
            return True
        return False  # 가득 참, 맨 앞 노드 PING 필요

    def get_closest(self, n: int) -> list[KNode]:
        return self.nodes[-n:]  # 가장 최근에 본 노드들

    def remove(self, node_id: int):
        self.nodes = [n for n in self.nodes if n.node_id != node_id]


class RoutingTable:
    """160개의 k-버킷으로 구성된 라우팅 테이블"""

    def __init__(self, own_id: int, k: int = 20):
        self.own_id = own_id
        self.k = k
        self.buckets: list[KBucket] = [KBucket(k) for _ in range(ID_BITS)]

    def _bucket_index(self, node_id: int) -> int:
        """어느 버킷에 속하는지 결정 (XOR의 최상위 비트 위치)"""
        dist = xor_distance(self.own_id, node_id)
        if dist == 0:
            return 0
        return dist.bit_length() - 1

    def add_node(self, node: KNode) -> bool:
        if node.node_id == self.own_id:
            return False
        idx = self._bucket_index(node.node_id)
        return self.buckets[idx].add(node)

    def find_closest(self, target_id: int, count: int = 20) -> list[KNode]:
        """target_id에 가장 가까운 count개 노드 반환"""
        candidates: list[KNode] = []
        for bucket in self.buckets:
            candidates.extend(bucket.nodes)
        candidates.sort(key=lambda n: xor_distance(n.node_id, target_id))
        return candidates[:count]


class KademliaNode:
    """단순화된 Kademlia 노드 (네트워크 없이 로컬 시뮬레이션)"""

    def __init__(self, address: str):
        self.id = sha1_id(address + str(random.random()))
        self.address = address
        self.routing_table = RoutingTable(self.id)
        self.storage: dict[int, str] = {}

    def __repr__(self):
        return f"KademliaNode({self.id:#x}, {self.address})"

    def ping(self, other: 'KademliaNode') -> bool:
        """다른 노드에 PING → 라우팅 테이블 갱신"""
        self.routing_table.add_node(KNode(other.id, other.address))
        other.routing_table.add_node(KNode(self.id, self.address))
        return True

    def find_node(self, target_id: int, network: dict[int, 'KademliaNode'],
                  alpha: int = 3, k: int = 20) -> list[KNode]:
        """FIND_NODE: target_id에 가장 가까운 k개 노드를 반복적으로 탐색"""
        visited: set[int] = {self.id}
        closest = self.routing_table.find_closest(target_id, k)
        
        while True:
            # α개씩 병렬 쿼리 (시뮬레이션: 순차)
            candidates = [n for n in closest if n.node_id not in visited][:alpha]
            if not candidates:
                break
            improved = False
            for contact in candidates:
                visited.add(contact.node_id)
                remote = network.get(contact.node_id)
                if remote is None:
                    continue
                new_nodes = remote.routing_table.find_closest(target_id, k)
                for nn in new_nodes:
                    self.routing_table.add_node(nn)
                # closest 갱신
                all_seen = {n.node_id: n for n in closest}
                for nn in new_nodes:
                    if nn.node_id not in all_seen:
                        all_seen[nn.node_id] = nn
                        improved = True
                closest = sorted(all_seen.values(),
                                 key=lambda n: xor_distance(n.node_id, target_id))[:k]
            if not improved:
                break
        return closest

    def store(self, key: str, value: str, network: dict[int, 'KademliaNode'], k: int = 20):
        """STORE: key의 해시에 가장 가까운 k개 노드에 값을 저장"""
        key_id = sha1_id(key)
        closest = self.find_node(key_id, network, k=k)
        for contact in closest:
            target = network.get(contact.node_id)
            if target:
                target.storage[key_id] = value

    def find_value(self, key: str, network: dict[int, 'KademliaNode']) -> Optional[str]:
        """FIND_VALUE: key에 해당하는 값 탐색"""
        key_id = sha1_id(key)
        if key_id in self.storage:
            return self.storage[key_id]
        closest = self.find_node(key_id, network)
        for contact in closest:
            target = network.get(contact.node_id)
            if target and key_id in target.storage:
                return target.storage[key_id]
        return None


# 시뮬레이션
def build_network(size: int = 50) -> dict[int, KademliaNode]:
    nodes = [KademliaNode(f"10.0.0.{i}:{8000+i}") for i in range(size)]
    network = {n.id: n for n in nodes}
    
    # 부트스트랩: 각 노드가 부트스트랩 노드와 PING
    bootstrap = nodes[0]
    for node in nodes[1:]:
        node.ping(bootstrap)
        # FIND_NODE로 자신 주변 채우기
        node.find_node(node.id, network)
    return network

network = build_network(50)
nodes = list(network.values())

# 데이터 저장/조회 테스트
publisher = nodes[0]
publisher.store("bitcoin:block:850000", "hash=000000000000000000034fa...", network)

finder = nodes[25]
result = finder.find_value("bitcoin:block:850000", network)
print(f"조회 결과: {result}")

# XOR 거리 예시
a = sha1_id("node-alpha")
b = sha1_id("node-beta")
print(f"XOR 거리: {xor_distance(a, b):#x}")
print(f"공통 상위 비트: {common_prefix_length(a, b)}")
```

---

### 예제 2: 실제 BitTorrent DHT 패킷 구조 분석 (Python + bencode)

```python
import struct
import socket

# BitTorrent DHT는 BEP-5 표준을 따릅니다
# 메시지 형식은 bencoding으로 직렬화됩니다

def bencode_str(s: str) -> bytes:
    b = s.encode()
    return f"{len(b)}:".encode() + b

def bencode_dict(d: dict) -> bytes:
    result = b"d"
    for k, v in sorted(d.items()):
        result += bencode_str(k)
        if isinstance(v, dict):
            result += bencode_dict(v)
        elif isinstance(v, str):
            result += bencode_str(v)
        elif isinstance(v, bytes):
            result += f"{len(v)}:".encode() + v
        elif isinstance(v, int):
            result += f"i{v}e".encode()
        elif isinstance(v, list):
            result += b"l" + b"".join(bencode_str(i) for i in v) + b"e"
    result += b"e"
    return result


def build_ping_query(transaction_id: bytes, node_id: bytes) -> bytes:
    """PING 쿼리 패킷 생성 (BEP-5)"""
    msg = {
        "t": transaction_id,  # 트랜잭션 ID
        "y": "q",             # 쿼리
        "q": "ping",          # 메서드명
        "a": {"id": node_id}, # 인자
    }
    # 실제로는 bencode 라이브러리 사용
    return (
        f"d1:t2:{transaction_id.decode('latin1')}1:y1:q1:q4:ping"
        f"1:ad2:id20:{node_id.decode('latin1')}ee"
    ).encode('latin1')


def build_find_node_query(txn_id: bytes, sender_id: bytes, target_id: bytes) -> dict:
    """FIND_NODE 쿼리 메시지 구조 (dict 형태)"""
    return {
        "t": txn_id.hex(),       # 트랜잭션 ID (2바이트)
        "y": "q",                 # 쿼리 타입
        "q": "find_node",         # RPC 이름
        "a": {
            "id": sender_id.hex(),   # 보내는 노드 ID (20바이트)
            "target": target_id.hex() # 찾고자 하는 노드 ID (20바이트)
        }
    }

def parse_compact_node_info(data: bytes) -> list[tuple[bytes, str, int]]:
    """
    Compact Node Info 파싱 (BEP-5):
    각 노드 = 20바이트 NodeID + 4바이트 IP + 2바이트 PORT = 26바이트
    """
    nodes = []
    for i in range(0, len(data), 26):
        chunk = data[i:i+26]
        if len(chunk) < 26:
            break
        node_id = chunk[:20]
        ip = socket.inet_ntoa(chunk[20:24])
        port = struct.unpack(">H", chunk[24:26])[0]
        nodes.append((node_id, ip, port))
    return nodes

# compact node info 파싱 예시
sample_compact = (
    b"\x00" * 19 + b"\x01"  # NodeID 1
    + socket.inet_aton("192.168.1.1")
    + struct.pack(">H", 6881)
    + b"\x00" * 19 + b"\x02"  # NodeID 2
    + socket.inet_aton("10.0.0.1")
    + struct.pack(">H", 6882)
)

parsed = parse_compact_node_info(sample_compact)
for nid, ip, port in parsed:
    print(f"Node ID: {nid.hex()}, 주소: {ip}:{port}")

# FIND_NODE 쿼리 예시 출력
import json
query = build_find_node_query(
    txn_id=bytes.fromhex("aa00"),
    sender_id=bytes.fromhex("a" * 40),
    target_id=bytes.fromhex("b" * 40),
)
print("\nFIND_NODE 쿼리:")
print(json.dumps(query, indent=2))
```

---

## 주의사항 및 팁

**1. 버킷 분할(Bucket Splitting) 전략**

구현 시 흔히 저지르는 실수는 단순히 160개 버킷을 처음부터 다 만드는 것입니다. 실용적인 구현은 처음에 단일 버킷에서 시작하고, 자신의 ID가 속한 버킷이 가득 차면 두 개로 분할합니다. 이를 **Bucket Splitting**이라 하며, 메모리와 라우팅 효율 모두를 높입니다.

**2. 노드 이탈(Churn) 대응**

피어 투 피어 네트워크에서는 노드가 수시로 접속/이탈합니다. Kademlia는 세 가지로 대응합니다.
- **k-버킷 LRU**: PING에 응답하는 오래된 노드를 새 노드보다 선호 → 안정적인 노드 유지
- **복제 계수(replication)**: 동일 키를 k개 노드에 복제 저장
- **재발행(republish)**: 저장된 데이터를 24시간마다 재발행하여 노드 이탈로 인한 데이터 손실 방지

**3. Sybil 공격 취약성**

공격자가 특정 키 주변에 다수의 가짜 노드를 배치하면 그 키를 "점령"할 수 있습니다. 이더리움의 Node Discovery v5(discv5)는 ENR(Ethereum Node Record) + 서명 검증으로 이를 부분적으로 방지합니다. IPFS는 별도의 콘텐츠 검증(CID = 콘텐츠 해시)으로 데이터 무결성을 보장합니다.

**4. Eclipse 공격**

공격자가 피해자의 라우팅 테이블을 악의적 노드로 가득 채워 피해자를 네트워크에서 격리하는 공격입니다. 대응책으로는 라우팅 테이블 다양성 유지(AS 단위 다양성), bootstrap 노드 신뢰 기반 초기화, 하드코딩된 시드 노드 활용이 있습니다.

**5. 조회 종료 조건**

"가장 가까운 k개 노드"가 한 라운드에서 더 이상 가까워지지 않으면 종료합니다. 단, 실용 구현에서는 일정 횟수(예: 3회) 연속 개선 없음을 기다리거나, RTT(Round Trip Time) 기반 타임아웃을 사용합니다.

**6. 실용 라이브러리**

- Python: `kademlia` (asyncio 기반), `opendht-python`
- Go: `github.com/libp2p/go-libp2p-kad-dht`
- Rust: `libp2p::kad`
- JavaScript: `@libp2p/kad-dht`

---

## 참고 자료

- [Kademlia - Wikipedia](https://en.wikipedia.org/wiki/Kademlia)
- [Distributed Hash Tables (DHT) - IPFS Docs](https://docs.ipfs.tech/concepts/dht/)
- [BitTorrent DHT Protocol - BEP-5](https://www.bittorrent.org/beps/bep_0005.html)
- [Kademlia DHT implementation - OpenDHT Wiki](https://github.com/savoirfairelinux/opendht/wiki/What-are-Distributed-Hash-Tables-%3F)
