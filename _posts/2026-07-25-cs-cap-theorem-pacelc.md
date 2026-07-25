---
layout: post
title: "CAP 정리와 PACELC: 분산 시스템 설계의 불가능한 트라이앵글 완전 정복"
date: 2026-07-25
categories: [cs, computer-science]
tags: [distributed-systems, cap-theorem, pacelc, consistency, availability, partition-tolerance, system-design]
---

분산 시스템을 설계할 때 가장 먼저 부딪히는 벽이 있다. "이 시스템은 항상 정확한 데이터를 보여줄 수 있는가?", "어떤 상황에서도 응답할 수 있는가?", "네트워크 장애가 나도 계속 동작할 수 있는가?" 이 세 가지 질문에 모두 "예스"라고 대답할 수 있는 시스템은 존재하지 않는다. 이것이 바로 **CAP 정리**의 핵심 통찰이다.

## CAP 정리란 무엇인가?

CAP 정리(CAP Theorem)는 2000년 Eric Brewer가 처음 제안하고 2002년 Seth Gilbert와 Nancy Lynch가 수학적으로 증명한 분산 시스템 이론이다. 이 정리는 분산 데이터 스토어가 다음 세 가지 속성을 동시에 완전히 보장하는 것은 불가능하다고 말한다.

- **C (Consistency, 일관성)**: 모든 읽기 연산은 가장 최근에 쓰여진 데이터를 반환한다. 즉, 어떤 노드에서 읽어도 동일한 최신 데이터를 볼 수 있어야 한다.
- **A (Availability, 가용성)**: 모든 요청은 (데이터가 최신인지 보장하지 못하더라도) 반드시 응답을 받는다. 즉, 시스템은 절대 다운타임 없이 모든 요청에 응답해야 한다.
- **P (Partition Tolerance, 파티션 허용성)**: 네트워크 파티션(노드 간 통신 단절)이 발생해도 시스템은 계속 작동한다.

핵심은 **파티션 허용성(P)은 포기할 수 없다**는 점이다. 실제 분산 시스템에서 네트워크 장애는 불가피하게 발생한다. 따라서 실질적인 선택은 **CP(일관성 우선)** 또는 **AP(가용성 우선)** 사이에서 이루어진다.

## 왜 CAP 정리가 필요한가?

2000년대 초반 대규모 인터넷 서비스가 등장하면서 단일 데이터베이스로는 수억 명의 요청을 처리할 수 없었다. 데이터를 여러 서버에 분산하기 시작했고, 그 순간부터 "네트워크 분리가 발생했을 때 어떻게 할 것인가?"라는 문제가 생겼다.

예를 들어, 두 데이터센터(DC-A, DC-B) 간의 네트워크 링크가 끊겼다고 하자. 이 순간 세 가지 행동 중 하나를 선택해야 한다:

1. **CP 선택**: DC-A와 DC-B 중 하나를 "리더"로 지정하고, 나머지는 쓰기를 거부한다. 데이터 일관성은 유지되지만 가용성이 감소한다.
2. **AP 선택**: 양쪽 모두 계속 쓰기를 받는다. 가용성은 유지되지만 나중에 충돌(conflict)이 발생할 수 있다.
3. **불가능한 선택**: 양쪽이 동기적으로 통신하면서 일관성과 가용성을 모두 유지한다. 그러나 네트워크가 끊겨 있으면 이것 자체가 불가능하다.

## CAP 정리 실제 구현 예제

다음 Python 예제는 CAP 정리의 상황을 시뮬레이션한다:

```python
import threading
import time
from typing import Optional

class Node:
    """분산 노드 시뮬레이터"""
    def __init__(self, node_id: str, mode: str = "CP"):
        self.node_id = node_id
        self.data: dict = {}
        self.mode = mode  # "CP" or "AP"
        self.is_partitioned = False
        self.peers: list['Node'] = []
        self.lock = threading.Lock()

    def write(self, key: str, value: str) -> bool:
        """데이터 쓰기"""
        if self.mode == "CP":
            # CP 모드: 파티션 중에는 쓰기 거부 (일관성 우선)
            if self.is_partitioned:
                print(f"[{self.node_id}] CP 모드: 파티션 감지, 쓰기 거부")
                return False

            # 모든 피어에게 동기 복제
            for peer in self.peers:
                if not peer.replicate(key, value):
                    print(f"[{self.node_id}] CP 모드: 복제 실패, 쓰기 롤백")
                    return False

            with self.lock:
                self.data[key] = value
            print(f"[{self.node_id}] CP 모드: 쓰기 성공 {key}={value}")
            return True

        else:  # AP 모드
            # AP 모드: 파티션 중에도 로컬에만 쓰기 (가용성 우선)
            with self.lock:
                self.data[key] = value
            print(f"[{self.node_id}] AP 모드: 로컬 쓰기 성공 {key}={value}")

            # 비동기로 피어에 복제 시도
            threading.Thread(
                target=self._async_replicate, args=(key, value)
            ).start()
            return True

    def read(self, key: str) -> Optional[str]:
        """데이터 읽기"""
        with self.lock:
            value = self.data.get(key)
        print(f"[{self.node_id}] 읽기: {key}={value}")
        return value

    def replicate(self, key: str, value: str) -> bool:
        """동기 복제"""
        if self.is_partitioned:
            return False
        with self.lock:
            self.data[key] = value
        return True

    def _async_replicate(self, key: str, value: str):
        """비동기 복제 (AP 모드)"""
        time.sleep(0.1)
        for peer in self.peers:
            if not peer.is_partitioned:
                peer.replicate(key, value)


def simulate_cap():
    # CP 시스템 시뮬레이션
    print("=== CP 시스템 (ZooKeeper 스타일) ===")
    node_a = Node("A", mode="CP")
    node_b = Node("B", mode="CP")
    node_a.peers = [node_b]
    node_b.peers = [node_a]

    node_a.write("user:1", "Alice")   # 정상 쓰기
    node_b.is_partitioned = True       # 파티션 발생
    node_a.write("user:2", "Bob")     # 거부됨

    print("\n=== AP 시스템 (Cassandra 스타일) ===")
    node_c = Node("C", mode="AP")
    node_d = Node("D", mode="AP")
    node_c.peers = [node_d]
    node_d.peers = [node_c]

    node_d.is_partitioned = True       # 파티션 발생
    node_c.write("user:1", "Alice")   # 로컬 쓰기 성공
    node_d.write("user:1", "Bob")     # 별도 로컬 쓰기 (충돌 발생!)
    time.sleep(0.2)
    # 파티션 해소 후 불일치 상태 확인
    print(f"C에서 읽기: {node_c.read('user:1')}")
    print(f"D에서 읽기: {node_d.read('user:1')}")  # 다를 수 있음


simulate_cap()
```

## PACELC: CAP 정리의 한계를 넘어서

CAP 정리는 훌륭한 이론이지만 실제 시스템 설계에는 한 가지 큰 맹점이 있다. **파티션이 없는 정상 동작 중에도 일관성과 지연(Latency) 사이의 트레이드오프가 존재한다**는 점이다.

Daniel Abadi가 2012년 제안한 **PACELC 정리**는 이 문제를 해결한다:

- **PAC**: 파티션(P)이 발생하면, 가용성(A)과 일관성(C) 중 선택
- **ELC**: 정상 운영(E, Else) 중에도, 지연(L, Latency)과 일관성(C) 사이에서 선택

즉, 시스템은 항상 **두 가지** 트레이드오프에 직면한다:

| 시스템 | 파티션 시 | 정상 시 | 분류 |
|--------|-----------|---------|------|
| DynamoDB | A | L | PA/EL |
| Cassandra | A | L | PA/EL |
| CRDB (CockroachDB) | C | C | PC/EC |
| Google Spanner | C | C | PC/EC |
| MongoDB | C | L | PC/EL |
| HBase | C | C | PC/EC |

## PACELC 실전 Java 구현 예제

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;
import java.util.*;

/**
 * PACELC 트레이드오프를 보여주는 분산 KV 스토어 시뮬레이터
 */
public class PACELCSimulator {

    enum ConsistencyLevel {
        STRONG,   // 선형화 일관성 (높은 지연, 강한 일관성)
        EVENTUAL  // 최종 일관성 (낮은 지연, 약한 일관성)
    }

    static class DistributedKVStore {
        private final int replicaCount;
        private final ConsistencyLevel consistencyLevel;
        private final List<Map<String, String>> replicas;
        private final ExecutorService asyncPool;

        public DistributedKVStore(int replicaCount, ConsistencyLevel level) {
            this.replicaCount = replicaCount;
            this.consistencyLevel = level;
            this.replicas = new ArrayList<>();
            this.asyncPool = Executors.newFixedThreadPool(replicaCount);
            for (int i = 0; i < replicaCount; i++) {
                replicas.add(new ConcurrentHashMap<>());
            }
        }

        /**
         * EL(낮은 지연) 쓰기: 과반수 응답 대기 후 나머지 비동기 복제
         * EC(강한 일관성) 쓰기: 모든 복제본에 동기 쓰기
         */
        public void write(String key, String value) throws Exception {
            if (consistencyLevel == ConsistencyLevel.STRONG) {
                // EC: 모든 복제본에 동기 쓰기 (높은 지연, 강한 일관성)
                long start = System.currentTimeMillis();
                for (Map<String, String> replica : replicas) {
                    Thread.sleep(10); // 네트워크 지연 시뮬레이션
                    replica.put(key, value);
                }
                System.out.printf("[EC/PC] 동기 쓰기 완료 (지연: %dms)%n",
                    System.currentTimeMillis() - start);
            } else {
                // EL: 첫 번째 복제본에만 쓰고 나머지 비동기 (낮은 지연, 약한 일관성)
                long start = System.currentTimeMillis();
                replicas.get(0).put(key, value); // 즉시 로컬 쓰기
                System.out.printf("[EL/PA] 비동기 쓰기 시작 (지연: %dms)%n",
                    System.currentTimeMillis() - start);

                // 나머지 복제본에 비동기 복제
                for (int i = 1; i < replicaCount; i++) {
                    final int idx = i;
                    asyncPool.submit(() -> {
                        try {
                            Thread.sleep(50); // 비동기 복제 지연
                            replicas.get(idx).put(key, value);
                        } catch (InterruptedException e) {
                            Thread.currentThread().interrupt();
                        }
                    });
                }
            }
        }

        /**
         * 읽기: 일관성 수준에 따라 다른 복제본 선택
         */
        public String read(String key, int replicaIndex) {
            return replicas.get(replicaIndex).get(key);
        }

        public void shutdown() {
            asyncPool.shutdown();
        }
    }

    public static void main(String[] args) throws Exception {
        System.out.println("=== EC (강한 일관성, 높은 지연) - Spanner 스타일 ===");
        DistributedKVStore strongStore =
            new DistributedKVStore(3, ConsistencyLevel.STRONG);
        strongStore.write("session:user1", "active");
        // 모든 복제본이 동일한 값을 반환
        for (int i = 0; i < 3; i++) {
            System.out.printf("  Replica %d: %s%n", i,
                strongStore.read("session:user1", i));
        }

        System.out.println("\n=== EL (낮은 지연, 최종 일관성) - Cassandra 스타일 ===");
        DistributedKVStore eventualStore =
            new DistributedKVStore(3, ConsistencyLevel.EVENTUAL);
        eventualStore.write("session:user2", "active");
        Thread.sleep(10); // 비동기 복제 완료 전
        for (int i = 0; i < 3; i++) {
            System.out.printf("  Replica %d (즉시): %s%n", i,
                eventualStore.read("session:user2", i));
        }
        Thread.sleep(100); // 복제 완료 후
        System.out.println("--- 100ms 후 ---");
        for (int i = 0; i < 3; i++) {
            System.out.printf("  Replica %d (이후): %s%n", i,
                eventualStore.read("session:user2", i));
        }

        strongStore.shutdown();
        eventualStore.shutdown();
    }
}
```

## 실제 데이터베이스 선택 가이드

CAP/PACELC 관점에서 데이터베이스를 선택할 때는 다음 질문에 답해야 한다:

**1. 금융/결제 시스템** → **CP/PC·EC 시스템 (CockroachDB, Spanner)**
- 잔액 불일치가 발생하면 심각한 문제이므로 일관성 최우선
- 지연이 다소 증가해도 허용 가능

**2. 소셜 미디어 좋아요/팔로워 수** → **AP/PA·EL 시스템 (Cassandra, DynamoDB)**
- 잠깐 다른 숫자가 보여도 사용자 경험에 큰 영향 없음
- 응답 속도와 항상 가용 가능한 상태가 더 중요

**3. 분산 설정/잠금** → **CP/PC·EC 시스템 (ZooKeeper, etcd)**
- 설정 값이나 잠금 상태가 불일치하면 심각한 버그 발생 가능

**4. 쇼핑몰 장바구니** → **AP 시스템 + 충돌 해결 (DynamoDB, Riak)**
- Amazon의 연구에 따르면 장바구니는 가용성 우선 + CRDT 기반 병합이 적합

## 주의사항 및 팁

**1. CAP은 이진 선택이 아니다**: 최신 시스템들은 조정 가능한(tunable) 일관성 수준을 제공한다. Cassandra의 `QUORUM`, `ALL`, `ONE` 같은 옵션이 대표적이다.

**2. 파티션은 드물지만 치명적이다**: 평소에는 지연-일관성 트레이드오프(PACELC의 E 부분)가 더 실질적인 문제다. 파티션은 드물지만 반드시 대비해야 한다.

**3. 최종 일관성은 "언젠간 같아진다"가 아니다**: 구체적인 수렴 시간(convergence time)과 충돌 해결 전략(conflict resolution)을 명확히 정의해야 한다.

**4. 단일 시스템 내 혼합 전략**: 실제 서비스에서는 데이터 유형에 따라 다른 일관성 수준을 적용하는 것이 일반적이다. 결제 데이터는 강한 일관성, 추천 피드는 최종 일관성으로 설계할 수 있다.

## 참고 자료
- [CAP Theorem - Wikipedia](https://en.wikipedia.org/wiki/CAP_theorem)
- [PACELC Theorem - Wikipedia](https://en.wikipedia.org/wiki/PACELC_theorem)
- [CAP and PACELC Theorems in Plain English - luminousmen.com](https://luminousmen.com/post/cap-and-pacelc-theorems-in-plain-english/)
- [CAP Theorem Explained - AlgoMaster](https://blog.algomaster.io/p/cap-theorem-explained)
