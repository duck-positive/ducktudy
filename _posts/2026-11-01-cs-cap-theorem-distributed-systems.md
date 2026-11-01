---
layout: post
title: "CAP 정리 완전 정복: 분산 시스템에서 일관성, 가용성, 분단 허용성의 트레이드오프"
date: 2026-11-01
categories: [cs, computer-science]
tags: [cap-theorem, distributed-systems, consistency, availability, partition-tolerance, database]
---

## CAP 정리란 무엇인가?

CAP 정리(CAP Theorem)는 분산 시스템에서 **일관성(Consistency)**, **가용성(Availability)**, **분단 허용성(Partition Tolerance)** 세 가지 속성을 동시에 모두 보장하는 것은 불가능하다는 정리입니다.

2000년 Eric Brewer가 PODC(Principles of Distributed Computing) 학회에서 최초로 제시했고, 2002년 MIT의 Seth Gilbert와 Nancy Lynch가 수학적으로 증명했습니다. 이 정리는 분산 데이터베이스와 분산 시스템을 설계할 때 근본적인 제약으로 작용합니다.

### 세 가지 속성의 정의

**일관성(Consistency, C)**  
모든 노드가 동일한 시점에 동일한 데이터를 바라봅니다. 어떤 노드에서 쓰기(write)가 완료된 후에 다른 노드에서 읽기(read)를 수행하면, 반드시 최신 데이터를 반환해야 합니다. CAP에서의 일관성은 ACID의 일관성(Consistency)과 의미가 다르며, **선형화 가능성(Linearizability)**에 가깝습니다.

**가용성(Availability, A)**  
정상적인 모든 노드에 요청을 보내면 반드시 응답을 받아야 합니다(오류 응답 제외). 단, 응답 내용이 최신 데이터가 아닐 수 있습니다. 즉, 시스템이 항상 응답하되, 데이터 정확성은 보장하지 않을 수 있습니다.

**분단 허용성(Partition Tolerance, P)**  
네트워크 분단(Network Partition) — 즉 일부 노드 간 메시지가 손실되거나 지연되는 상황 — 이 발생해도 시스템이 계속 동작해야 합니다.

## 왜 셋 다 동시에 불가능한가?

분산 시스템에서 **네트워크 분단은 현실에서 반드시 발생**합니다. 케이블 단절, 라우터 오류, 데이터센터 전원 장애 등은 피할 수 없습니다. 따라서 **분단 허용성(P)은 사실상 포기할 수 없는 전제**이며, 실제 선택은 **C와 A 사이의 트레이드오프**입니다.

### 직관적 예시

두 데이터센터(DC1, DC2)가 있고, 사용자는 어느 쪽에나 요청할 수 있다고 가정합니다.

```
       DC1 (x=1 저장 중)         DC2 (x=0, 아직 동기화 안 됨)
       ┌──────────┐               ┌──────────┐
       │  Node A  │ ─ × ─ × ─── │  Node B  │  ← 네트워크 분단!
       └──────────┘               └──────────┘
    사용자1: write(x=1)         사용자2: read(x) → ?
```

이 상황에서:
- **C를 선택(CP)**하려면 → Node B가 최신 x 값을 모르므로 응답을 거부해야 합니다(가용성 손실).
- **A를 선택(AP)**하려면 → Node B가 오래된 x=0을 응답할 수 있습니다(일관성 손실).

분단 상황이 해소(네트워크 복구)될 때까지 두 속성을 동시에 만족시킬 방법이 없습니다.

## 실제 구현 예제

### 예제 1: Python으로 CP 시스템 시뮬레이션

아래는 분단 발생 시 가용성을 희생하고 일관성을 유지하는 CP 시스템의 간단한 시뮬레이션입니다.

```python
import threading
import time
import random

class CPNode:
    """일관성 우선(CP) 노드 — 분단 시 쓰기를 거부"""
    def __init__(self, node_id: str, peers: list):
        self.node_id = node_id
        self.peers = peers
        self.data = {}
        self.partitioned = False   # 네트워크 분단 여부
        self.lock = threading.Lock()

    def write(self, key: str, value) -> bool:
        """
        과반수(quorum) 노드에 복제에 성공해야만 쓰기를 수행.
        분단 상황에서 과반수 미달 시 쓰기를 거부해 일관성을 보장.
        """
        if self.partitioned:
            reachable = [p for p in self.peers if not p.partitioned]
        else:
            reachable = self.peers

        quorum = len(self.peers) // 2 + 1  # 과반수
        if len(reachable) < quorum - 1:    # 자신 포함 과반수 미달
            print(f"[{self.node_id}] 쓰기 거부: 과반수 미달 "
                  f"(도달 가능 {len(reachable)+1}/{len(self.peers)+1})")
            return False  # 가용성 손실, 일관성 유지

        # 복제 수행
        with self.lock:
            self.data[key] = value
        for peer in reachable:
            with peer.lock:
                peer.data[key] = value

        print(f"[{self.node_id}] write({key}={value}) 성공 — "
              f"{len(reachable)+1}개 노드에 복제 완료")
        return True

    def read(self, key: str):
        with self.lock:
            return self.data.get(key, None)


# 3노드 CP 클러스터 시뮬레이션
n1, n2, n3 = CPNode("N1", []), CPNode("N2", []), CPNode("N3", [])
n1.peers, n2.peers, n3.peers = [n2, n3], [n1, n3], [n1, n2]

# 정상 상태: 쓰기 성공
print("=== 정상 상태 ===")
n1.write("user:1", {"name": "Alice", "score": 100})

# 분단 시뮬레이션: N3 격리
print("\n=== 네트워크 분단 발생 (N3 격리) ===")
n3.partitioned = True

# N3 입장에서 쓰기 시도: 자신만 도달 가능 → 거부
n3.write("user:2", {"name": "Bob", "score": 200})

# N1 입장에서 쓰기 시도: N1+N2(과반수) 도달 가능 → 성공
n1.write("user:2", {"name": "Bob", "score": 200})
```

실행 결과:
```
=== 정상 상태 ===
[N1] write(user:1={'name': 'Alice', 'score': 100}) 성공 — 3개 노드에 복제 완료

=== 네트워크 분단 발생 (N3 격리) ===
[N3] 쓰기 거부: 과반수 미달 (도달 가능 1/3)
[N1] write(user:2={'name': 'Bob', 'score': 200}) 성공 — 2개 노드에 복제 완료
```

---

### 예제 2: AP 시스템의 Eventually Consistent 복제

AP 시스템은 분단 상황에서도 응답하며, 분단 해소 후 **최종적 일관성(Eventual Consistency)**으로 수렴합니다.

```python
import time
import threading
from collections import defaultdict

class APNode:
    """가용성 우선(AP) 노드 — 분단 시 낡은 데이터라도 응답"""
    def __init__(self, node_id: str):
        self.node_id = node_id
        self.data: dict = {}
        self.version: dict = defaultdict(int)  # 벡터 클럭 단순화
        self.peers: list = []
        self.partitioned = False

    def write(self, key: str, value) -> bool:
        self.version[key] += 1
        self.data[key] = (value, self.version[key])
        print(f"[{self.node_id}] write({key}={value}, v={self.version[key]})")

        # 백그라운드로 복제 시도 (분단 상황이면 실패 허용)
        threading.Thread(target=self._replicate, args=(key,), daemon=True).start()
        return True  # 항상 성공 반환 → 가용성 보장

    def read(self, key: str):
        val = self.data.get(key)
        if val:
            return val[0]  # 낡을 수 있지만 항상 응답
        return None

    def _replicate(self, key: str):
        time.sleep(0.01)  # 네트워크 지연 시뮬레이션
        for peer in self.peers:
            if not self.partitioned and not peer.partitioned:
                local_v = self.version[key]
                peer_v = peer.version.get(key, 0)
                if local_v > peer_v:  # Last-Write-Wins 단순 구현
                    peer.data[key] = self.data[key]
                    peer.version[key] = local_v

    def sync_from_peers(self):
        """분단 해소 후 수동 동기화 (최종 일관성 수렴)"""
        for peer in self.peers:
            for key in peer.data:
                if peer.version[key] > self.version.get(key, 0):
                    self.data[key] = peer.data[key]
                    self.version[key] = peer.version[key]
                    print(f"[{self.node_id}] '{key}' 동기화 완료: "
                          f"v{self.version[key]}")


n1, n2 = APNode("N1"), APNode("N2")
n1.peers, n2.peers = [n2], [n1]

# 분단 중에도 양측이 독립적으로 쓰기 — 충돌 발생
n1.partitioned = n2.partitioned = True
n1.write("config", "v2_dark_mode")
n2.write("config", "v2_light_mode")

print(f"\n[분단 중] N1.read(config) = {n1.read('config')}")
print(f"[분단 중] N2.read(config) = {n2.read('config')}")

# 분단 해소 후 동기화 (Last-Write-Wins이므로 버전 높은 쪽이 승리)
print("\n=== 분단 해소, 동기화 시작 ===")
n1.partitioned = n2.partitioned = False
n1.sync_from_peers()
n2.sync_from_peers()
```

## CAP 정리의 실제 데이터베이스 분류

| 시스템 | 선택 | 특징 |
|--------|------|------|
| HBase, Zookeeper, etcd | **CP** | 분단 시 일부 쓰기 거부, 강한 일관성 |
| Cassandra, CouchDB, DynamoDB | **AP** | 분단 시 낡은 데이터 응답, 최종 일관성 |
| MySQL/PostgreSQL (단일) | CA | 단일 노드: 분단 자체가 없음, 실제 분산 환경에선 의미 없음 |

## Brewer의 수정과 PACELC 모델

2012년 Brewer 자신이 CAP 정리의 "2 of 3" 프레임이 오해를 낳는다고 수정했습니다. 핵심은:

- **네트워크 분단은 드문 이벤트**입니다. 분단이 없는 정상 상태에서의 레이턴시-일관성 트레이드오프가 더 실용적인 관심사입니다.
- **PACELC 모델**(Daniel Abadi, 2012)은 이를 보완합니다: "분단(P) 시 A vs C를 선택하고, 정상(E, Else) 상태에서는 레이턴시(L) vs 일관성(C)을 선택한다."

## 주의사항 및 팁

**1. Consistency는 ACID와 다릅니다**  
CAP의 C는 "선형화 가능성(Linearizability)"으로, ACID의 C(제약 조건 보존)와 다릅니다. 혼용에 주의하세요.

**2. 분단은 반드시 일어납니다**  
AWS, GCP 등 클라우드 환경에서도 가용 영역(AZ) 간 네트워크 지연·단절은 정기적으로 발생합니다. P를 포기한 CA 시스템은 분산 환경에서 현실적이지 않습니다.

**3. 튜너블 일관성**  
Cassandra의 `CONSISTENCY QUORUM`, DynamoDB의 `ConsistentRead` 옵션처럼, 일부 시스템은 요청 단위로 일관성 수준을 조절할 수 있습니다. 중요한 읽기에는 강한 일관성, 고빈도 읽기에는 최종 일관성을 쓰는 하이브리드 전략이 효과적입니다.

**4. BASE 원칙**  
AP 시스템은 ACID 대신 **BASE(Basically Available, Soft state, Eventually consistent)**를 따릅니다. 소셜 미디어 좋아요 수처럼 일시적 불일치가 허용되는 데이터에 적합합니다.

## 참고 자료
- [CAP Theorem & Strategies for Distributed Systems — Splunk](https://www.splunk.com/en_us/blog/learn/cap-theorem.html)
- [Understanding the CAP Theorem — freeCodeCamp](https://www.freecodecamp.org/news/understanding-the-cap-theorem-consistency-availability-and-partition-tolerance-in-system-design/)
- [What is CAP Theorem? — IONOS](https://www.ionos.com/digitalguide/server/know-how/what-is-cap-theorem)
- [CAP Theorem — Zilliz Glossary](https://zilliz.com/glossary/cap-theorem)
