---
layout: post
title: "Chord DHT 완전 정복: 중앙 서버 없이 수백만 노드가 키를 찾는 방법"
date: 2026-07-10
categories: [cs, computer-science]
tags: [chord, dht, distributed-hash-table, peer-to-peer, consistent-hashing, distributed-systems, finger-table]
---

## Chord DHT란?

**Chord**는 2001년 MIT의 Ion Stoica, Robert Morris 등이 SIGCOMM에서 발표한 **분산 해시 테이블(Distributed Hash Table, DHT)** 프로토콜입니다. 2011년 SIGCOMM Test-of-Time Award를 수상했을 만큼 분산 시스템 분야에서 가장 영향력 있는 논문 중 하나로 손꼽힙니다.

Chord의 핵심 아이디어는 단순합니다: **중앙 서버 없이 N개의 노드가 키-값 쌍을 저장하고, 어떤 노드에서든 O(log N) 번의 메시지 교환만으로 원하는 키를 찾을 수 있다.**

### 문제 의식: 왜 DHT가 필요한가?

전통적인 분산 시스템에서 "어떤 서버가 키 K를 갖고 있는가?"를 알기 위해서는 세 가지 방법이 있습니다:

1. **중앙 디렉터리 서버**: SPOF(단일 장애 지점)가 됨. Napster가 이 방식이었고, 결국 서버가 내려가며 서비스 종료
2. **브로드캐스트**: 모든 노드에 질의 → 트래픽 폭증, O(N) 비용
3. **모든 노드가 모든 정보를 저장**: 노드 수에 비례한 O(N) 라우팅 테이블 → 확장 불가

Chord는 **링 구조 + 핑거 테이블**로 이 문제를 O(log N) 라우팅 테이블과 O(log N) 조회 시간으로 해결합니다.

---

## 왜 Chord가 중요한가?

Chord는 BitTorrent, IPFS, Amazon Dynamo, Apache Cassandra, Ethereum의 피어 발견(peer discovery) 등 현대 분산 시스템 전반에 영향을 미쳤습니다.

### 핵심 특성

- **탈중앙화(Decentralization)**: 모든 노드가 동등
- **확장성(Scalability)**: 노드가 늘어도 각 노드의 라우팅 테이블은 O(log N)
- **가용성(Availability)**: 노드가 떠나거나 합류해도 시스템 지속 동작
- **부하 분산(Load Balancing)**: 일관성 해싱으로 키가 균등하게 분산

---

## 내부 구조: 링과 핑거 테이블

### 1. 식별자 링 (Identifier Ring)

Chord는 0부터 2^m - 1까지의 식별자 공간을 **원형 링**으로 구성합니다 (보통 m = 160, SHA-1 사용).

- 각 **노드**는 IP 주소를 해싱한 **노드 ID**를 가집니다
- 각 **키**는 해싱한 **키 ID**를 가집니다
- 키 K는 K 이상의 ID를 가진 가장 첫 번째 노드가 책임집니다 → 이를 **successor(K)**라 부릅니다

```
링 예시 (m=4, 16개의 슬롯):
0 → 1 → 2 → ... → 15 → 0 (순환)

노드: N1(id=1), N4(id=4), N7(id=7), N11(id=11), N14(id=14)

키 K=6의 successor → N7 (6 이상인 첫 번째 노드)
키 K=12의 successor → N14
키 K=15의 successor → N1 (링을 한 바퀴 돌아)
```

### 2. 핑거 테이블 (Finger Table)

단순 successor 연결만 사용하면 최악의 경우 N번의 홉이 필요합니다. Chord는 각 노드마다 **m개의 핑거(finger)**를 저장해 조회를 O(log N)으로 단축합니다.

노드 n의 i번째 핑거:

```
finger[i] = successor(n + 2^(i-1))  mod 2^m
            (i = 1, 2, ..., m)
```

즉, 1/2, 1/4, 1/8, ... 거리에 해당하는 위치의 successor를 미리 알고 있어, 이진 탐색처럼 목적지에 빠르게 접근할 수 있습니다.

---

## 실제 구현 예제

### 예제 1: Chord 링 시뮬레이션 (Python)

```python
import hashlib
import bisect

M = 6  # 6비트 링 → 0~63
RING_SIZE = 2 ** M


def hash_key(key: str) -> int:
    return int(hashlib.sha1(key.encode()).hexdigest(), 16) % RING_SIZE


class ChordNode:
    def __init__(self, node_id: int):
        self.id = node_id
        self.finger = [None] * (M + 1)  # 1-indexed
        self.predecessor = None
        self.data = {}  # 이 노드가 담당하는 키-값 저장소

    def find_successor(self, key_id: int, ring: 'ChordRing') -> 'ChordNode':
        """key_id의 successor 노드를 찾는다 — O(log N)"""
        if self._in_range(key_id, self.id, self.finger[1].id, inclusive_right=True):
            return self.finger[1]
        # 가장 가깝고 key_id보다 앞선 핑거로 점프
        closest = self._closest_preceding_finger(key_id, ring)
        if closest.id == self.id:
            return self.finger[1]
        return closest.find_successor(key_id, ring)

    def _closest_preceding_finger(self, key_id: int, ring: 'ChordRing') -> 'ChordNode':
        for i in range(M, 0, -1):
            f = self.finger[i]
            if f and self._in_range(f.id, self.id, key_id, inclusive_right=False):
                return f
        return self

    def _in_range(self, x: int, start: int, end: int, inclusive_right: bool) -> bool:
        """x가 (start, end] 또는 (start, end) 범위에 있는지 — 링 구조 고려"""
        if start == end:
            return True
        if start < end:
            if inclusive_right:
                return start < x <= end
            return start < x < end
        else:  # 링을 가로질러
            if inclusive_right:
                return x > start or x <= end
            return x > start or x < end

    def __repr__(self):
        return f"Node({self.id})"


class ChordRing:
    def __init__(self):
        self.nodes = {}  # id -> ChordNode
        self.sorted_ids = []

    def add_node(self, node_id: int):
        node = ChordNode(node_id)
        self.nodes[node_id] = node
        bisect.insort(self.sorted_ids, node_id)
        self._rebuild_fingers()
        return node

    def get_successor(self, key_id: int) -> ChordNode:
        """key_id 이상인 가장 가까운 노드 반환"""
        idx = bisect.bisect_left(self.sorted_ids, key_id)
        if idx == len(self.sorted_ids):
            idx = 0
        return self.nodes[self.sorted_ids[idx]]

    def _rebuild_fingers(self):
        """핑거 테이블 전체 재구성 (시뮬레이션 편의상)"""
        for node_id, node in self.nodes.items():
            for i in range(1, M + 1):
                start = (node_id + 2 ** (i - 1)) % RING_SIZE
                node.finger[i] = self.get_successor(start)

    def put(self, key: str, value: str):
        key_id = hash_key(key)
        responsible = self.get_successor(key_id)
        responsible.data[key] = value
        print(f"PUT '{key}' (id={key_id}) → {responsible}")

    def get(self, key: str, query_from: ChordNode) -> str:
        key_id = hash_key(key)
        responsible = query_from.find_successor(key_id, self)
        value = responsible.data.get(key)
        print(f"GET '{key}' (id={key_id}) → {responsible}: '{value}'")
        return value


# --- 사용 예시 ---
ring = ChordRing()
for nid in [1, 14, 28, 42, 54]:
    ring.add_node(nid)

ring.put("alice", "engineer")
ring.put("bob", "designer")
ring.put("charlie", "manager")

# 임의 노드에서 조회
start_node = ring.nodes[14]
ring.get("alice", start_node)
ring.get("charlie", start_node)
```

### 예제 2: 노드 합류(Join)와 안정화(Stabilize) 프로토콜 (Go)

```go
package chord

import (
	"crypto/sha1"
	"fmt"
	"math/big"
)

const M = 160 // SHA-1 비트 수
var ringSize = new(big.Int).Exp(big.NewInt(2), big.NewInt(M), nil)

// NodeID는 SHA-1 해시로 생성된 160비트 정수
func NodeID(addr string) *big.Int {
	h := sha1.Sum([]byte(addr))
	return new(big.Int).SetBytes(h[:])
}

// inRange는 x가 (start, end] 범위인지 확인 (링 구조)
func inRange(x, start, end *big.Int) bool {
	if start.Cmp(end) < 0 {
		return x.Cmp(start) > 0 && x.Cmp(end) <= 0
	}
	// 링 경계를 넘는 경우
	return x.Cmp(start) > 0 || x.Cmp(end) <= 0
}

type Node struct {
	ID          *big.Int
	Addr        string
	Successor   *Node
	Predecessor *Node
	Finger      [M + 1]*Node // 1-indexed
}

func NewNode(addr string) *Node {
	return &Node{
		ID:   NodeID(addr),
		Addr: addr,
	}
}

// Join: 기존 링에 참여 — n0가 링의 임의 노드
func (n *Node) Join(n0 *Node) {
	if n0 == nil {
		// 첫 번째 노드: 혼자서 링 형성
		n.Predecessor = n
		n.Successor = n
		for i := 1; i <= M; i++ {
			n.Finger[i] = n
		}
		fmt.Printf("Node %s created a new ring\n", n.Addr)
		return
	}
	// 기존 노드를 통해 successor 탐색
	n.Successor = n0.FindSuccessor(n.ID)
	n.Predecessor = nil
	fmt.Printf("Node %s joined: successor = %s\n", n.Addr, n.Successor.Addr)
}

// FindSuccessor: key_id의 담당 노드를 찾는다
func (n *Node) FindSuccessor(id *big.Int) *Node {
	if inRange(id, n.ID, n.Successor.ID) {
		return n.Successor
	}
	closest := n.closestPrecedingFinger(id)
	if closest == n {
		return n.Successor
	}
	return closest.FindSuccessor(id)
}

func (n *Node) closestPrecedingFinger(id *big.Int) *Node {
	for i := M; i >= 1; i-- {
		if n.Finger[i] != nil && inRange(n.Finger[i].ID, n.ID, id) {
			return n.Finger[i]
		}
	}
	return n
}

// Stabilize: 주기적으로 실행하여 successor/predecessor 포인터를 최신 상태로 유지
func (n *Node) Stabilize() {
	// Successor의 predecessor를 확인
	x := n.Successor.Predecessor
	if x != nil && inRange(x.ID, n.ID, n.Successor.ID) {
		n.Successor = x
	}
	// Successor에게 우리가 predecessor임을 알림
	n.Successor.Notify(n)
}

// Notify: n0가 우리의 predecessor일 수 있음을 통보받음
func (n *Node) Notify(n0 *Node) {
	if n.Predecessor == nil || inRange(n0.ID, n.Predecessor.ID, n.ID) {
		n.Predecessor = n0
	}
}

// FixFingers: 핑거 테이블을 주기적으로 갱신
func (n *Node) FixFingers(next int) int {
	// finger[next] = successor(n + 2^(next-1))
	start := new(big.Int).Add(n.ID, new(big.Int).Exp(big.NewInt(2), big.NewInt(int64(next-1)), nil))
	start.Mod(start, ringSize)
	n.Finger[next] = n.FindSuccessor(start)
	next++
	if next > M {
		next = 1
	}
	return next
}

// Example usage:
func Example() {
	// 최초 노드 생성
	node1 := NewNode("192.168.1.1:8080")
	node1.Join(nil) // 새 링 생성

	// 추가 노드들 합류
	node2 := NewNode("192.168.1.2:8080")
	node2.Join(node1)

	node3 := NewNode("192.168.1.3:8080")
	node3.Join(node1)

	// 주기적으로 호출되어 링을 안정적으로 유지
	node1.Stabilize()
	node2.Stabilize()
	node3.Stabilize()

	// 키 조회
	key := NodeID("some-data-key")
	responsible := node1.FindSuccessor(key)
	fmt.Printf("Key responsible node: %s\n", responsible.Addr)
}
```

---

## Chord의 핵심 알고리즘 상세

### 노드 합류 과정

새 노드 n이 링에 합류할 때:

1. 기존 노드 n0에게 연락하여 `n.successor = n0.FindSuccessor(n.id)` 설정
2. **안정화(Stabilize)**: 주기적으로 실행 — successor의 predecessor를 확인하고, 자신이 더 가까우면 successor를 갱신
3. **핑거 테이블 갱신(FixFingers)**: 각 핑거를 한 번씩 순차적으로 갱신
4. 기존 노드들도 Stabilize를 통해 새 노드를 자신의 successor/predecessor로 인식

합류 완료까지 O(log² N)의 Stabilize 라운드가 필요하다고 논문에서 증명합니다.

### 노드 이탈과 장애 처리

Chord는 각 노드가 **successor list**(r개의 successor)를 유지하도록 권장합니다. 주 successor가 장애를 일으켜도 다음 successor로 대체하여 링 연속성을 보장합니다. r = O(log N)이면 N개의 노드 중 절반이 동시에 장애를 일으켜도 올바른 조회가 가능합니다.

---

## 실제 시스템에서의 Chord

| 시스템 | Chord 활용 방식 |
|--------|----------------|
| **BitTorrent** | Kademlia DHT (Chord에서 영감)로 피어 탐색 |
| **IPFS** | Kademlia 기반 분산 콘텐츠 주소 지정 |
| **Amazon Dynamo** | 일관성 해싱 + 가상 노드로 데이터 분산 |
| **Apache Cassandra** | 토큰 링으로 데이터 파티셔닝 |
| **Ethereum** | Kademlia 기반 P2P 피어 발견 프로토콜 |

---

## 주의사항과 실전 팁

### Chord의 한계

1. **단조로운 하나의 링**: 여러 복제본이 필요한 경우 별도의 복제 전략이 필요합니다 (Dynamo는 preference list 사용)
2. **네트워크 지역성 무시**: SHA-1 해싱 결과는 지리적으로 가까운 노드를 고려하지 않음. Coral DHT는 이를 개선
3. **비잔틴 장애 취약**: 악의적인 노드가 잘못된 라우팅 정보를 제공할 수 있음. S/Kademlia가 이를 개선
4. **핫스팟 문제**: 특정 키가 과도하게 요청될 경우 담당 노드 부하 집중 → 복제 또는 캐싱으로 완화

### Chord vs Kademlia

실제 시스템에서 더 널리 쓰이는 것은 **Kademlia**입니다. Chord는 단방향 링을 사용하고, Kademlia는 XOR 메트릭을 사용해 양방향 조회가 가능하고 병렬 조회 지원이 쉽습니다. 그러나 Chord는 학습 목적으로 더 직관적이며, 두 시스템 모두 O(log N) 조회라는 핵심 특성을 공유합니다.

---

## 참고 자료

- [Chord: A Scalable Peer-to-peer Lookup Service (원본 논문 요약)](https://arxiv.org/abs/1502.06461)
- [Chord DHT - Wikipedia](https://en.wikipedia.org/wiki/Chord_(peer-to-peer))
- [Building a DHT in Golang - Medium](https://medium.com/techlog/chord-building-a-dht-distributed-hash-table-in-golang-67c3ce17417b)
- [CS 3410: Chord DHT Assignment - Dixie State University](https://cit.dixie.edu/cs/3410/asst_chord.html)
