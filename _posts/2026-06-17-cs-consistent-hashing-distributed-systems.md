---
layout: post
title: "일관성 해싱(Consistent Hashing): 분산 시스템 로드 밸런싱의 핵심 알고리즘"
date: 2026-06-17
categories: [cs, computer-science]
tags: [consistent-hashing, 분산시스템, 해싱, 로드밸런싱, 시스템설계, 알고리즘]
---

## 개요

분산 시스템에서 데이터를 여러 서버에 고르게 분산시키는 것은 핵심 과제입니다. 서버를 추가하거나 제거할 때 기존 데이터의 재분배 비용을 최소화하는 것이 특히 중요합니다.

전통적인 해시 기반 분산 방식(예: `hash(key) % N`)은 서버 수 N이 변경될 때마다 거의 모든 데이터를 재배치해야 합니다. 이 문제를 해결하기 위해 1997년 David Karger와 동료들이 제안한 것이 **일관성 해싱(Consistent Hashing)**입니다.

Amazon DynamoDB, Apache Cassandra, Discord의 분산 캐시, Memcached 등 수많은 대규모 서비스가 이 알고리즘을 핵심적으로 활용합니다.

---

## 1. 전통적 해싱의 문제점

서버 3대(S0, S1, S2)에 키를 분산한다고 가정해봅시다.

```python
def traditional_hash(key: str, num_servers: int) -> int:
    return hash(key) % num_servers

servers = ["S0", "S1", "S2"]
keys = [f"user:{i}" for i in range(1, 7)]

print("=== 서버 3대 ====")
for key in keys:
    idx = traditional_hash(key, 3)
    print(f"  {key} → {servers[idx]}")

# 서버 1대 추가 (S3)
print("\n=== 서버 4대 (S3 추가) ====" )
for key in keys:
    idx_before = traditional_hash(key, 3)
    idx_after  = traditional_hash(key, 4)
    moved = "← 이동!" if idx_before != idx_after else ""
    print(f"  {key}: S{idx_before} → S{idx_after} {moved}")
```

서버가 3대에서 4대로 늘어나면 `hash(key) % 4`의 결과가 대부분 달라집니다. 이론적으로 서버가 N대일 때 1대를 추가하면 약 **(N-1)/N**의 키가 재배치됩니다. 서버 100대 환경에서 1대를 추가할 경우 **약 99%의 키가 이동**하는 셈입니다.

이는 캐시 서버에서 치명적입니다. 서버 재배치 직후 거의 모든 요청이 캐시 미스(Cache Miss)가 되어 원본 데이터베이스에 엄청난 부하를 줍니다.

---

## 2. 일관성 해싱의 원리: 해시 링

일관성 해싱은 해시 공간을 **원형 링(Hash Ring)**으로 만들어 이 문제를 해결합니다.

### 동작 방식

1. 해시 함수의 출력 범위(예: 0 ~ 2³²-1)를 원형으로 배치합니다.
2. 각 서버를 해시 함수로 링 위의 특정 위치에 배치합니다.
3. 키를 해시하여 링 위의 위치를 구한 뒤, **시계 방향으로 가장 가까운 서버**에 배치합니다.

서버가 추가/제거될 때, 해당 서버와 바로 이전 서버 사이의 키만 재배치됩니다. N개의 서버 중 1개가 변경될 때 평균 **K/N**개의 키만 이동합니다(K: 전체 키 수). 이것이 바로 일관성 해싱의 핵심 장점입니다.

---

## 3. 코드 예제 1 — 기본 일관성 해싱 구현

```python
import hashlib
from bisect import bisect, insort

class ConsistentHashRing:
    def __init__(self):
        self.ring = {}           # 해시값 → 서버 이름
        self.sorted_keys = []    # 정렬된 해시값 목록

    def _hash(self, key: str) -> int:
        return int(hashlib.md5(key.encode()).hexdigest(), 16)

    def add_node(self, node: str):
        h = self._hash(node)
        self.ring[h] = node
        insort(self.sorted_keys, h)

    def remove_node(self, node: str):
        h = self._hash(node)
        del self.ring[h]
        self.sorted_keys.remove(h)

    def get_node(self, key: str) -> str:
        if not self.ring:
            return None
        h = self._hash(key)
        # 시계 방향으로 가장 가까운 서버 탐색
        idx = bisect(self.sorted_keys, h) % len(self.sorted_keys)
        return self.ring[self.sorted_keys[idx]]

# 사용 예시
ring = ConsistentHashRing()
for server in ["Server-A", "Server-B", "Server-C"]:
    ring.add_node(server)

# 키 분산 확인
keys = [f"user:{i}" for i in range(1, 11)]
dist = {}
for key in keys:
    server = ring.get_node(key)
    dist[server] = dist.get(server, 0) + 1
    print(f"  {key} → {server}")

print("\n분산 현황:", dist)

# Server-D 추가 후 재배치 비율 확인
before = {k: ring.get_node(k) for k in keys}
ring.add_node("Server-D")
after  = {k: ring.get_node(k) for k in keys}
moved = sum(1 for k in keys if before[k] != after[k])
print(f"\nServer-D 추가 후 재배치된 키: {moved}/{len(keys)}개 ({moved/len(keys)*100:.0f}%)")
# 출력 예: 재배치된 키: 3/10개 (30%) — 이론적으로 1/N = 25%에 근접
```

이진 탐색(bisect)을 통해 get_node 연산의 시간 복잡도는 **O(log N)**입니다.

---

## 4. 코드 예제 2 — 가상 노드(Virtual Nodes)로 부하 균형 개선

기본 구현의 문제점은 서버가 링 위에서 균일하게 분포되지 않을 수 있다는 것입니다. 우연히 한 서버가 링의 절반을 차지한다면 해당 서버에 50%의 요청이 집중됩니다.

**가상 노드(Virtual Nodes, VNode)**는 각 물리 서버를 링 위에 여러 개의 가상 지점으로 분산 배치하여 이 문제를 해결합니다.

```python
import hashlib
from bisect import bisect, insort
import random

class ConsistentHashRingWithVNodes:
    def __init__(self, virtual_nodes: int = 150):
        self.ring = {}
        self.sorted_keys = []
        self.virtual_nodes = virtual_nodes

    def _hash(self, key: str) -> int:
        return int(hashlib.md5(key.encode()).hexdigest(), 16)

    def add_node(self, node: str):
        """각 서버를 virtual_nodes개의 가상 지점으로 링에 분산 배치"""
        for i in range(self.virtual_nodes):
            virtual_key = f"{node}#vnode{i}"
            h = self._hash(virtual_key)
            self.ring[h] = node
            insort(self.sorted_keys, h)

    def remove_node(self, node: str):
        for i in range(self.virtual_nodes):
            virtual_key = f"{node}#vnode{i}"
            h = self._hash(virtual_key)
            del self.ring[h]
            self.sorted_keys.remove(h)

    def get_node(self, key: str) -> str:
        if not self.ring:
            return None
        h = self._hash(key)
        idx = bisect(self.sorted_keys, h) % len(self.sorted_keys)
        return self.ring[self.sorted_keys[idx]]

# 10,000개 키로 분산 시뮬레이션
vring = ConsistentHashRingWithVNodes(virtual_nodes=150)
for server in ["Server-A", "Server-B", "Server-C"]:
    vring.add_node(server)

dist = {}
for i in range(10_000):
    key = f"item:{random.randint(1, 1_000_000)}"
    server = vring.get_node(key)
    dist[server] = dist.get(server, 0) + 1

print("=== 가상 노드 150개 — 분산 결과 ===")
for server, count in sorted(dist.items()):
    bar = "█" * (count // 200)
    print(f"  {server}: {count:5d}건 ({count/100:.1f}%) {bar}")

# 예상 출력 (각 서버 약 33% 균등 분배):
#   Server-A:  3312건 (33.1%) ████████████████
#   Server-B:  3345건 (33.4%) ████████████████
#   Server-C:  3343건 (33.4%) ████████████████
```

가상 노드 수를 늘릴수록 분산이 더 균일해집니다. 실제 서비스에서는 통상 **100~300개**의 가상 노드를 사용합니다.

---

## 5. 전통적 해싱 vs 일관성 해싱 재배치 비율 비교

```python
# 전통적 해싱: 서버 3대 → 4대
keys_10k = [f"item:{i}" for i in range(10_000)]
traditional_moved = sum(
    1 for k in keys_10k
    if hash(k) % 3 != hash(k) % 4
)
print(f"전통적 해싱  — 서버 추가 시 재배치: {traditional_moved/100:.1f}%")

# 일관성 해싱: Server-A/B/C → Server-A/B/C/D 추가
ring3 = ConsistentHashRingWithVNodes(150)
for s in ["S0", "S1", "S2"]:
    ring3.add_node(s)

ring4 = ConsistentHashRingWithVNodes(150)
for s in ["S0", "S1", "S2", "S3"]:
    ring4.add_node(s)

ch_moved = sum(
    1 for k in keys_10k
    if ring3.get_node(k) != ring4.get_node(k)
)
print(f"일관성 해싱 — 서버 추가 시 재배치: {ch_moved/100:.1f}%")

# 출력 예시:
# 전통적 해싱  — 서버 추가 시 재배치: 75.0%
# 일관성 해싱 — 서버 추가 시 재배치: 25.3%  (이론값 25%에 근접)
```

---

## 6. 실제 서비스 적용 사례

### Amazon DynamoDB
DynamoDB는 파티션 키(Partition Key)를 일관성 해싱으로 분산 저장합니다. 각 파티션에는 3개의 복제본이 자동 배치됩니다.

### Apache Cassandra
Cassandra는 가상 노드를 기본적으로 활용하며(기본값: 노드당 128개 VNode), 새 노드 추가 시 자동으로 데이터를 재분배합니다.

### Nginx 업스트림 일관성 해싱
```nginx
upstream backend {
    hash $request_uri consistent;
    server backend1.example.com;
    server backend2.example.com;
    server backend3.example.com;
}
```
동일한 URI는 항상 동일한 서버로 라우팅되어 백엔드 캐시 효율이 높아집니다.

---

## 7. 주의사항과 실무 팁

1. **가상 노드 수를 적절히 설정하세요.** 너무 적으면(< 50) 불균형, 너무 많으면(> 500) 메모리 및 노드 관리 오버헤드가 발생합니다. 100~200을 권장합니다.

2. **복제(Replication)와 함께 사용하세요.** 일관성 해싱만으로는 내결함성을 보장하지 않습니다. 링 위에서 시계 방향으로 후속 N개 노드에 복제본을 저장하는 방식을 결합하세요.

3. **해시 함수의 균일성을 검증하세요.** MD5 외에 MurmurHash3, xxHash 등의 고속 해시 함수도 고려하세요. 해시 충돌을 최소화하는 것이 중요합니다.

4. **핫키(Hot Key) 문제는 별도로 처리해야 합니다.** 특정 키에 요청이 집중될 경우 가상 노드만으로는 해결되지 않습니다. 이 경우 해당 키에 대한 추가 캐시 레이어나 키 샤딩 전략이 필요합니다.

5. **노드 장애(Failover) 시나리오를 설계하세요.** 노드가 갑자기 제거될 경우 해당 노드의 키들이 다음 노드로 쏠립니다. 이를 대비해 복제 팩터와 타임아웃 정책을 함께 설계해야 합니다.

---

## 참고 자료
- [Wikipedia: Consistent hashing](https://en.wikipedia.org/wiki/Consistent_hashing)
- [Ably Blog: Implementing Efficient Consistent Hashing](https://ably.com/blog/implementing-efficient-consistent-hashing)
- [Toptal: The Ultimate Guide to Consistent Hashing](https://www.toptal.com/big-data/consistent-hashing)
- [High Scalability: Consistent hashing algorithm](https://highscalability.com/consistent-hashing-algorithm/)
