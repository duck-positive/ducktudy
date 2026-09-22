---
layout: post
title: "OSPF 링크 상태 라우팅 프로토콜 완전 정복: 인터넷 내부를 연결하는 동적 라우팅의 핵심"
date: 2026-09-22
categories: [cs, computer-science]
tags: [ospf, routing, network, link-state, dijkstra, lsa, distributed-systems]
---

## 개념 설명

OSPF(Open Shortest Path First)는 인터넷 내부 라우팅(IGP, Interior Gateway Protocol)의 표준으로, RFC 2328에 정의된 **링크 상태(Link-State)** 라우팅 프로토콜입니다. 같은 목적지로의 경로를 찾는 프로토콜이지만, BGP가 AS(Autonomous System) 간 경로를 교환하는 경로 벡터 프로토콜인 반면, OSPF는 AS 내부의 라우터들이 서로의 연결 상태를 공유하고 각자가 독립적으로 최적 경로를 계산하는 방식입니다.

### 링크 상태 vs 거리 벡터

초기 IGP인 RIP(Routing Information Protocol)는 **거리 벡터(Distance Vector)** 방식이었습니다. 각 라우터는 이웃에게 "나는 X 네트워크까지 N 홉이야"라는 정보만 전달합니다. 이 방식은 구현이 단순하지만 수렴 속도가 느리고 라우팅 루프가 발생하기 쉽습니다.

OSPF는 이를 해결합니다. 각 라우터는 **자신에게 직접 연결된 링크의 상태(Link State)**를 광고(advertise)하고, 이 정보가 전체 AS에 전파되면 모든 라우터가 동일한 토폴로지 맵을 가집니다. 그 위에서 Dijkstra 알고리즘으로 최단 경로 트리(SPT)를 직접 계산합니다.

## 왜 OSPF가 필요한가

### 빠른 수렴(Fast Convergence)

네트워크 장애 발생 시 거리 벡터 프로토콜은 모든 라우터가 순차적으로 정보를 업데이트해야 합니다. OSPF는 변경 사항이 발생하면 즉시 LSA(Link State Advertisement)를 플러딩(Flooding)해 수십 초가 아닌 수 초 이내에 수렴합니다.

### 정확한 토폴로지 인식

OSPF의 모든 라우터는 동일한 LSDB(Link State Database)를 가집니다. 전체 토폴로지를 알기 때문에 루프 없이 최적 경로를 계산합니다. BGP처럼 정책 기반 경로 선택이 아니라 메트릭(대역폭 기반 코스트)에 따른 순수 최단 경로를 구합니다.

### 계층적 설계 — Area

대규모 네트워크에서 모든 라우터가 전체 토폴로지 정보를 유지하면 메모리와 CPU 부담이 커집니다. OSPF는 **Area** 개념으로 이를 해결합니다. Area 0(백본 Area)을 중심으로 다른 Area들이 연결되며, Area 내부의 세부 토폴로지는 외부 Area로 노출되지 않습니다.

- **ABR(Area Border Router)**: 두 Area를 연결하는 라우터
- **ASBR(AS Boundary Router)**: 외부 라우팅 도메인과 연결하는 라우터

## OSPF 동작 원리

### 1단계: 이웃 발견 (Neighbor Discovery)

OSPF 라우터는 Hello 패킷을 멀티캐스트(224.0.0.5)로 주기적으로 전송합니다. Hello 패킷에는 라우터 ID, Area ID, Hello/Dead 인터벌, 인증 정보 등이 포함됩니다. 이웃 라우터가 Hello를 받고 파라미터가 일치하면 **Neighbor 관계**가 형성됩니다.

브로드캐스트 네트워크(이더넷)에서는 N×(N-1)/2개의 Adjacency 수를 줄이기 위해 **DR(Designated Router)**와 **BDR(Backup DR)**을 선출합니다. 다른 라우터들은 DR/BDR과만 Adjacency를 맺고, 나머지와는 단순 Neighbor 관계를 유지합니다.

### 2단계: LSDB 동기화 (Database Exchange)

새 라우터가 네트워크에 참여하면 DBD(Database Description) 패킷을 교환해 어떤 LSA를 가지고 있는지 비교합니다. 없는 LSA는 LSR(Link State Request)로 요청하고 LSU(Link State Update)로 받습니다. 마지막으로 LSAck로 확인하면 LSDB가 동기화됩니다.

### 3단계: SPF 계산

LSDB가 완성되면 각 라우터는 자신을 루트로 하는 **Shortest Path Tree(SPT)**를 Dijkstra 알고리즘으로 계산합니다.

```python
import heapq
from collections import defaultdict

class OSPFRouter:
    def __init__(self, router_id):
        self.router_id = router_id
        self.lsdb = {}  # router_id -> {neighbor: cost}
    
    def add_lsa(self, router_id, links):
        """링크 상태 광고 수신"""
        self.lsdb[router_id] = links
    
    def calculate_spf(self):
        """Dijkstra로 최단 경로 트리 계산"""
        dist = defaultdict(lambda: float('inf'))
        dist[self.router_id] = 0
        prev = {}
        pq = [(0, self.router_id)]
        visited = set()
        
        while pq:
            cost, node = heapq.heappop(pq)
            if node in visited:
                continue
            visited.add(node)
            
            for neighbor, link_cost in self.lsdb.get(node, {}).items():
                new_cost = cost + link_cost
                if new_cost < dist[neighbor]:
                    dist[neighbor] = new_cost
                    prev[neighbor] = node
                    heapq.heappush(pq, (new_cost, neighbor))
        
        return dict(dist), prev
    
    def build_routing_table(self):
        dist, prev = self.calculate_spf()
        table = {}
        for dest, cost in dist.items():
            if dest == self.router_id:
                continue
            # 다음 홉 찾기
            path = []
            cur = dest
            while cur in prev:
                path.append(cur)
                cur = prev[cur]
            path.reverse()
            next_hop = path[0] if path else dest
            table[dest] = {'cost': cost, 'next_hop': next_hop}
        return table


# 시뮬레이션: R1-R2-R3-R4로 구성된 네트워크
router = OSPFRouter("R1")

# LSDB 구성 (각 라우터의 LSA 수신)
router.add_lsa("R1", {"R2": 10, "R3": 30})
router.add_lsa("R2", {"R1": 10, "R3": 15, "R4": 5})
router.add_lsa("R3", {"R1": 30, "R2": 15, "R4": 20})
router.add_lsa("R4", {"R2": 5, "R3": 20})

table = router.build_routing_table()
for dest, info in table.items():
    print(f"{dest}: cost={info['cost']}, next_hop={info['next_hop']}")
# R2: cost=10, next_hop=R2
# R3: cost=25, next_hop=R2  (R1->R2->R3: 10+15=25, R1->R3: 30)
# R4: cost=15, next_hop=R2  (R1->R2->R4: 10+5=15)
```

### 4단계: LSA 플러딩 (LSA Flooding)

OSPF의 핵심 메커니즘입니다. 링크 상태가 변경되면 해당 라우터는 새 LSA를 생성하고 모든 인접 라우터에게 전송합니다. LSA를 받은 라우터는 자신의 LSDB를 업데이트하고 다른 인접 라우터들에게 다시 전송합니다. 이 과정이 전체 AS에 퍼집니다.

LSA에는 **Sequence Number**가 있어 오래된 정보를 버립니다. **Age** 필드는 1800초(30분)까지 증가하며, MaxAge에 도달하면 LSDB에서 삭제됩니다. 라우터는 30분마다 자신의 LSA를 새로 생성해 플러딩합니다.

```python
class LSA:
    """Link State Advertisement"""
    def __init__(self, router_id, sequence_num, links):
        self.router_id = router_id
        self.sequence_num = sequence_num  # 단조 증가
        self.age = 0
        self.links = links  # {neighbor: cost}
        self.MAX_AGE = 3600  # 1시간
        self.LS_REFRESH_TIME = 1800  # 30분

class OSPFLSDB:
    def __init__(self):
        self.db = {}  # (router_id) -> LSA
    
    def process_lsa(self, lsa: LSA) -> bool:
        """
        LSA 수신 처리. 반환값: True면 플러딩 필요
        """
        existing = self.db.get(lsa.router_id)
        
        if existing is None:
            self.db[lsa.router_id] = lsa
            return True  # 새 LSA → 플러딩
        
        # Sequence Number 비교
        if lsa.sequence_num > existing.sequence_num:
            self.db[lsa.router_id] = lsa
            return True  # 더 새로운 LSA → 플러딩
        elif lsa.sequence_num == existing.sequence_num:
            return False  # 이미 알고 있음 → 플러딩 불필요
        else:
            # 자신이 더 최신 → 원본 발신자에게 최신 버전 전송
            return False

# 플러딩 시뮬레이션
lsdb = OSPFLSDB()
lsa1 = LSA("R2", sequence_num=1, links={"R1": 10, "R3": 15, "R4": 5})
lsa2 = LSA("R2", sequence_num=2, links={"R1": 10, "R3": 20, "R4": 5})  # 링크 비용 변경

print(lsdb.process_lsa(lsa1))  # True - 플러딩 필요
print(lsdb.process_lsa(lsa1))  # False - 이미 처리됨
print(lsdb.process_lsa(lsa2))  # True - 더 새로운 LSA, 플러딩 필요
```

## OSPF 메트릭 — 코스트(Cost)

OSPF 메트릭은 **참조 대역폭 / 인터페이스 대역폭**으로 계산합니다. 기본 참조 대역폭은 100Mbps입니다.

| 인터페이스 속도 | 코스트 |
|---|---|
| 10Mbps | 10 |
| 100Mbps | 1 |
| 1Gbps | 1 (주의!) |
| 10Gbps | 1 (주의!) |

기본 참조 대역폭으로는 1Gbps와 10Gbps가 같은 코스트가 되어버립니다. 실무에서는 `auto-cost reference-bandwidth 10000` (10Gbps 기준)으로 설정하는 것이 일반적입니다.

## OSPF LSA 타입

OSPF에는 여러 종류의 LSA가 있습니다.

- **Type 1 (Router LSA)**: 각 라우터가 생성. 자신의 인터페이스와 인접 라우터 정보
- **Type 2 (Network LSA)**: DR이 생성. 브로드캐스트 네트워크의 라우터 목록
- **Type 3 (Summary LSA)**: ABR이 생성. Area 간 경로 요약
- **Type 4 (ASBR Summary LSA)**: ASBR의 위치를 알리는 LSA
- **Type 5 (External LSA)**: ASBR이 생성. 외부 AS 경로 정보
- **Type 7 (NSSA External LSA)**: Not-So-Stubby Area에서 외부 경로 광고

## 주의사항과 실전 팁

### 1. Hello/Dead 인터벌 일치 확인

이웃 관계가 맺어지지 않는 가장 흔한 원인입니다. 두 라우터의 Hello Interval과 Dead Interval이 다르면 Neighbor가 형성되지 않습니다. 기본값은 P2P에서 10초/40초, 브로드캐스트에서 10초/40초입니다.

### 2. MTU 불일치

DBD 교환 시 MTU가 일치하지 않으면 ExStart 단계에서 Neighbor 관계가 멈춥니다. 인터페이스 MTU를 맞추거나 `ip ospf mtu-ignore` 명령으로 우회합니다.

### 3. Stub Area 활용

외부 경로가 필요 없는 Area는 Stub Area로 설정해 Type 5 LSA를 차단합니다. LSDB 크기와 SPF 계산 부담이 크게 줄어듭니다. 또한 Totally Stubby Area로 설정하면 Type 3 LSA도 차단합니다.

### 4. SPF 계산 타이머 조정

네트워크 불안정 시 너무 자주 SPF를 계산하면 CPU 부하가 급증합니다. Cisco IOS의 `timers throttle spf` 명령으로 SPF 계산 지연을 설정합니다.

```
timers throttle spf 5 1000 90000
# spf-start=5ms, spf-hold=1000ms, spf-max=90000ms
```

### 5. OSPFv3 (IPv6)

OSPFv2가 IPv4 전용인 반면, OSPFv3(RFC 5340)은 IPv6를 지원합니다. 주소 패밀리를 분리해 동일한 LSA 구조를 유지하되 IPv6 주소를 사용합니다.

## 참고 자료

- [RFC 2328 - OSPF Version 2](https://www.rfc-editor.org/rfc/rfc2328.html)
- [OSPF - Wikipedia](https://en.wikipedia.org/wiki/Open_Shortest_Path_First)
- [FRRouting OSPFv2 Documentation](https://docs.frrouting.org/en/stable-8.5/ospfd.html)
- [Dijkstra's Algorithm - Wikipedia](https://en.wikipedia.org/wiki/Dijkstra%27s_algorithm)
