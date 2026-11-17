---
layout: post
title: "Consistent Hashing 완전 정복: 분산 시스템에서 노드 추가·삭제 시 최소한의 리밸런싱으로 부하를 분산하는 방법"
date: 2026-11-17
categories: [cs, computer-science]
tags: [consistent-hashing, distributed-systems, load-balancing, virtual-nodes, dynamo, cassandra]
---

서버 한 대가 죽었을 때, 전체 캐시가 무효화되어 데이터베이스로 쏠리는 "캐시 스탬피드(Cache Stampede)"를 경험해본 적이 있는가? 단순한 모듈러 해싱(`key % n`)에서 노드 하나가 사라지면 거의 모든 키의 담당 노드가 바뀐다. Consistent Hashing은 이 문제를 O(K/N)의 리밸런싱만으로 해결한다. Amazon Dynamo, Apache Cassandra, Redis Cluster, CDN 로드 밸런서가 모두 이 알고리즘을 채택한 이유다.

## 모듈러 해싱의 한계

총 N개의 서버가 있을 때, 키 k는 `server = hash(k) % N`번 서버에 저장된다. N=3일 때 `hash(k)=10`이면 서버 1에 저장된다. 서버 하나가 추가되어 N=4가 되면 `10 % 4 = 2`로, 담당 서버가 1에서 2로 바뀐다. 최악의 경우 전체 키의 (N-1)/N ≈ 100%가 리밸런싱된다.

이 문제는 단순히 성능 저하가 아니다. 캐시 클러스터에서는 모든 캐시 미스가 동시에 발생하여 원본 DB에 폭발적인 부하를 유발한다.

## Consistent Hashing의 핵심 아이디어: 해시 링

Consistent Hashing은 1997년 Karger et al.의 논문 "Consistent Hashing and Random Trees"에서 처음 제안됐다. 핵심 개념은 다음과 같다:

1. **해시 공간을 원형 링으로 표현한다.** 해시값의 범위(예: 0 ~ 2^32-1)를 원형으로 배치한다.
2. **서버도 해시 링 위에 배치한다.** `hash(서버ID)`로 링의 한 지점에 위치시킨다.
3. **키는 링에서 시계 방향으로 가장 가까운 서버에 할당된다.**

노드 추가 시: 새 노드와 그 다음 노드 사이의 키만 재배치된다 → 평균 K/N개 키만 이동.  
노드 제거 시: 해당 노드의 키만 다음 노드로 이동된다 → 역시 평균 K/N개.

## 실제 구현 예제 1: Python으로 Consistent Hash Ring 구현

```python
import hashlib
import bisect
from typing import Optional

class ConsistentHashRing:
    def __init__(self, virtual_nodes: int = 150):
        """
        virtual_nodes: 각 물리 노드당 가상 노드 수
        가상 노드가 많을수록 부하가 균등하게 분산됨
        """
        self.virtual_nodes = virtual_nodes
        self.ring: dict[int, str] = {}   # hash → 노드 이름
        self.sorted_keys: list[int] = []  # 정렬된 해시 키 목록

    def _hash(self, key: str) -> int:
        return int(hashlib.md5(key.encode()).hexdigest(), 16)

    def add_node(self, node: str) -> None:
        for i in range(self.virtual_nodes):
            virtual_key = f"{node}#vnode{i}"
            h = self._hash(virtual_key)
            self.ring[h] = node
            bisect.insort(self.sorted_keys, h)
        print(f"Added node '{node}' with {self.virtual_nodes} virtual nodes")

    def remove_node(self, node: str) -> None:
        for i in range(self.virtual_nodes):
            virtual_key = f"{node}#vnode{i}"
            h = self._hash(virtual_key)
            if h in self.ring:
                del self.ring[h]
                self.sorted_keys.remove(h)
        print(f"Removed node '{node}'")

    def get_node(self, key: str) -> Optional[str]:
        if not self.ring:
            return None
        h = self._hash(key)
        # 링에서 시계 방향으로 가장 가까운 노드 찾기
        idx = bisect.bisect(self.sorted_keys, h)
        # 링의 끝을 넘으면 처음으로 (wrap-around)
        if idx == len(self.sorted_keys):
            idx = 0
        return self.ring[self.sorted_keys[idx]]

    def get_distribution(self, keys: list[str]) -> dict[str, int]:
        distribution: dict[str, int] = {}
        for key in keys:
            node = self.get_node(key)
            if node:
                distribution[node] = distribution.get(node, 0) + 1
        return distribution


# 사용 예시
ring = ConsistentHashRing(virtual_nodes=100)

# 노드 3개로 시작
ring.add_node("server-A")
ring.add_node("server-B")
ring.add_node("server-C")

# 키 1000개 분배 확인
test_keys = [f"user:{i}" for i in range(1000)]
dist = ring.get_distribution(test_keys)
print("초기 분배:", dist)

# 특정 키 조회
print("user:42 →", ring.get_node("user:42"))
print("product:99 →", ring.get_node("product:99"))

# 노드 추가 (D 추가 시 얼마나 리밸런싱?)
ring.add_node("server-D")
dist_after = ring.get_distribution(test_keys)
print("\n노드 추가 후 분배:", dist_after)

# 리밸런싱된 키 수 계산
moved = sum(
    1 for k in test_keys
    if ring.get_node(k) != dist.get(ring.get_node(k))
)
print(f"이론상 리밸런싱 비율: {1/4:.1%} (실제 약 {sum(1 for k in test_keys if ring.get_node(k) != 'server-D' and k in [kk for kk in test_keys if ring.get_node(kk) == 'server-D'])/1000:.1%})")
```

## 가상 노드(Virtual Nodes)가 필요한 이유

물리 노드를 링에 하나만 배치하면 편향(skew)이 발생한다. 노드가 적을 때 운 나쁘게 한쪽에 몰릴 수 있다. 가상 노드는 각 물리 노드를 링 위의 여러 지점에 분산시켜 부하 균형을 보장한다.

Amazon Dynamo는 이를 "token" 개념으로 발전시켜, 각 노드가 담당하는 링의 범위를 명시적으로 조절한다. 하드웨어 사양이 다른 서버에 가중치를 줄 수도 있다.

## 실제 구현 예제 2: Java로 멀티 레플리케이션 지원 Consistent Hash 구현

```java
import java.security.MessageDigest;
import java.util.*;

public class ConsistentHashWithReplica {
    private final int virtualNodes;
    private final int replicaCount;
    private final TreeMap<Long, String> ring = new TreeMap<>();

    public ConsistentHashWithReplica(int virtualNodes, int replicaCount) {
        this.virtualNodes = virtualNodes;
        this.replicaCount = replicaCount;
    }

    private long hash(String key) {
        try {
            MessageDigest md = MessageDigest.getInstance("MD5");
            byte[] digest = md.digest(key.getBytes());
            // 처음 8바이트를 long으로 변환
            long h = 0;
            for (int i = 0; i < 8; i++) {
                h = (h << 8) | (digest[i] & 0xFF);
            }
            return h & Long.MAX_VALUE; // 양수 보장
        } catch (Exception e) {
            return key.hashCode() & Long.MAX_VALUE;
        }
    }

    public void addNode(String node) {
        for (int i = 0; i < virtualNodes; i++) {
            ring.put(hash(node + "#" + i), node);
        }
    }

    public void removeNode(String node) {
        for (int i = 0; i < virtualNodes; i++) {
            ring.remove(hash(node + "#" + i));
        }
    }

    // 레플리케이션을 위한 N개의 담당 노드 반환
    public List<String> getNodes(String key) {
        if (ring.isEmpty()) return Collections.emptyList();

        long h = hash(key);
        Set<String> uniqueNodes = new LinkedHashSet<>();
        NavigableMap<Long, String> tail = ring.tailMap(h);

        // 시계 방향으로 replicaCount개의 고유 노드 수집
        for (String node : tail.values()) {
            uniqueNodes.add(node);
            if (uniqueNodes.size() >= replicaCount) break;
        }
        // 부족하면 처음부터 계속
        if (uniqueNodes.size() < replicaCount) {
            for (String node : ring.values()) {
                uniqueNodes.add(node);
                if (uniqueNodes.size() >= replicaCount) break;
            }
        }

        return new ArrayList<>(uniqueNodes);
    }

    public static void main(String[] args) {
        // 레플리케이션 팩터 3 (Cassandra/Dynamo 기본)
        ConsistentHashWithReplica ch = new ConsistentHashWithReplica(100, 3);
        ch.addNode("node-1");
        ch.addNode("node-2");
        ch.addNode("node-3");
        ch.addNode("node-4");
        ch.addNode("node-5");

        String[] testKeys = {"order:1001", "user:9999", "product:555", "session:abc"};
        for (String key : testKeys) {
            List<String> nodes = ch.getNodes(key);
            System.out.println(key + " → " + nodes);
        }

        // 노드 장애 시뮬레이션
        System.out.println("\n--- node-3 장애 발생 ---");
        ch.removeNode("node-3");
        for (String key : testKeys) {
            List<String> nodes = ch.getNodes(key);
            System.out.println(key + " → " + nodes + " (변경됐나요?)");
        }
    }
}
```

## Consistent Hashing의 현실적 한계와 해결책

### 핫스팟(Hotspot) 문제
특정 키에 트래픽이 집중되면 가상 노드로도 해결이 안 된다. Cassandra는 "Paxos-based Light-weight Transactions"으로 쓰기 충돌을 조정하고, 읽기는 여러 레플리카에 분산한다.

### 순서가 있는 범위 쿼리의 어려움
해시 기반이므로 연속된 키가 다른 노드에 흩어진다. 범위 스캔이 필요하면 DynamoDB처럼 파티션 키 + 정렬 키를 분리하거나, MongoDB처럼 샤딩 키를 범위 기반으로 선택한다.

### 데이터 이관 중 일관성
노드 추가 시 새 노드로 데이터를 복사하는 동안 일시적으로 구 노드와 신 노드 양쪽에 데이터가 있을 수 있다. Cassandra의 `nodetool repair`와 Dynamo의 "read repair"가 이를 처리한다.

## Consistent Hashing vs 다른 분산 전략

| 전략 | 리밸런싱 비용 | 범위 쿼리 | 사용 사례 |
|------|-------------|-----------|----------|
| **Modular Hashing** | O(K) ≈ 100% | 불가 | 단일 서버 |
| **Consistent Hashing** | O(K/N) | 불가 | 캐시, NoSQL |
| **Range Partitioning** | 수동 조정 필요 | 가능 | CockroachDB, TiKV |
| **Directory-based** | O(1) | 가능 | HDFS NameNode |

## 참고 자료

- [원본 논문: Consistent Hashing and Random Trees (Karger et al., 1997)](https://courses.cs.duke.edu/fall21/compsci514/internal/readings/ConsistentHashing.pdf)
- [High Scalability: Consistent Hashing Algorithm](https://highscalability.com/consistent-hashing-algorithm/)
- [Wikipedia: Consistent Hashing](https://en.wikipedia.org/wiki/Consistent_hashing)
- [Amazon Dynamo Paper](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)
