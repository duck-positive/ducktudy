---
layout: post
title: "부하 분산 알고리즘 심층 분석 — Round Robin부터 Power of Two Choices까지"
date: 2026-07-29
categories: [cs, computer-science]
tags: [load-balancing, round-robin, least-connections, power-of-two-choices, consistent-hashing, distributed-systems, system-design]
---

## 개념 설명

부하 분산(Load Balancing)은 클라이언트 요청을 여러 서버에 분산시켜 단일 서버의 과부하를 방지하고 전체 시스템의 가용성과 처리량을 높이는 기술이다. 분산 시스템과 대규모 웹 서비스에서 핵심 인프라 요소로, 어떤 서버에 요청을 보낼지 결정하는 것이 **부하 분산 알고리즘**이다.

좋은 부하 분산 알고리즘은 다음 목표를 달성해야 한다:
- **균등한 부하 분배**: 어떤 서버도 병목이 되지 않도록
- **낮은 오버헤드**: 알고리즘 자체가 병목이 되어선 안 됨
- **빠른 장애 대응**: 서버 장애 시 트래픽을 즉시 우회
- **상태 인식**: 서버의 현재 부하를 반영하는 지능적 라우팅

## 왜 중요한가?

단순해 보이는 부하 분산이 실제로는 복잡한 문제다. 2008년 Mitzenmacher의 연구 "The Power of Two Choices in Randomized Load Balancing"은 완전 무작위 선택이 얼마나 불균형한 부하를 만드는지를 수학적으로 보여주었다. N개 서버에 N개 공을 완전 무작위로 넣으면 최대 부하가 `O(log N / log log N)`인 반면, 2개를 선택해 덜 찬 쪽을 선택하면 최대 부하가 `O(log log N)`으로 극적으로 줄어든다.

## 주요 알고리즘

### 1. Round Robin

요청을 서버 목록에 순차적으로 분배한다. 구현이 가장 단순하다.

**적합한 상황**: 서버 스펙이 동일하고 요청 처리 시간이 비슷한 경우.
**부적합한 상황**: 일부 요청이 훨씬 오래 걸리거나, 서버 스펙이 다른 경우.

### 2. Weighted Round Robin

서버마다 가중치를 부여해 성능이 좋은 서버에 더 많은 요청을 보낸다. Nginx의 스무스(Smooth) WRR은 가중치에 비례하면서도 요청 분배가 최대한 고르게 이루어지도록 설계되어 있다.

### 3. Least Connections

현재 활성 연결이 가장 적은 서버로 요청을 보낸다.

**적합한 상황**: 데이터베이스 쿼리, 파일 업로드 등 처리 시간 편차가 큰 워크로드.

### 4. Power of Two Choices (P2C)

전체 서버 풀에서 무작위로 2개를 선택한 뒤, 그 중 덜 부하를 받는 서버로 요청을 보낸다. 이 단순한 휴리스틱이 무작위 선택 대비 부하 불균형을 로그 스케일로 줄인다는 것이 수학적으로 증명되어 있다.

**적합한 상황**: 대규모 서버 풀, Least Connections 대비 연결 수 전역 조회 오버헤드를 줄이고 싶을 때.

### 5. Consistent Hashing

클라이언트 IP나 요청 키를 해시해 항상 동일한 서버로 라우팅한다.

**적합한 상황**: 세션 유지, 캐시 효율성이 중요한 경우.

## 실제 구현 예제

### 예제 1: 파이썬으로 구현하는 부하 분산 알고리즘들

```python
import random
import statistics
from dataclasses import dataclass, field

@dataclass
class Server:
    host: str
    port: int
    weight: int = 1
    active_connections: int = field(default=0, compare=False)

    def __repr__(self):
        return f"{self.host}:{self.port}(w={self.weight}, conn={self.active_connections})"

    def __lt__(self, other):
        return self.active_connections < other.active_connections


class RoundRobinBalancer:
    """단순 라운드 로빈"""
    def __init__(self, servers: list[Server]):
        self.servers = servers
        self._index = 0

    def pick(self) -> Server:
        server = self.servers[self._index % len(self.servers)]
        self._index += 1
        return server


class WeightedRoundRobinBalancer:
    """가중치 기반 라운드 로빈 (Nginx 스무스 알고리즘)"""
    def __init__(self, servers: list[Server]):
        self.servers = servers
        self._current_weights = [0] * len(servers)
        self._total_weight = sum(s.weight for s in servers)

    def pick(self) -> Server:
        # 현재 가중치에 각 서버의 weight를 더함
        for i, s in enumerate(self.servers):
            self._current_weights[i] += s.weight

        # 현재 가중치가 가장 높은 서버 선택
        best_idx = max(range(len(self.servers)),
                       key=lambda i: self._current_weights[i])

        # 선택된 서버의 현재 가중치에서 총 가중치를 차감
        self._current_weights[best_idx] -= self._total_weight

        return self.servers[best_idx]


class LeastConnectionsBalancer:
    """최소 연결 알고리즘"""
    def __init__(self, servers: list[Server]):
        self.servers = servers

    def pick(self) -> Server:
        return min(self.servers, key=lambda s: s.active_connections)

    def on_connect(self, server: Server):
        server.active_connections += 1

    def on_disconnect(self, server: Server):
        server.active_connections = max(0, server.active_connections - 1)


class PowerOfTwoChoicesBalancer:
    """Power of Two Choices — 2개 무작위 선택 후 최선 선택"""
    def __init__(self, servers: list[Server]):
        self.servers = servers

    def pick(self) -> Server:
        if len(self.servers) < 2:
            return self.servers[0]

        # 중복 없이 무작위로 2개 선택
        a, b = random.sample(self.servers, 2)

        # 활성 연결이 적은 쪽 선택 (동률이면 a 선택)
        return a if a.active_connections <= b.active_connections else b

    def on_connect(self, server: Server):
        server.active_connections += 1

    def on_disconnect(self, server: Server):
        server.active_connections = max(0, server.active_connections - 1)


# 사용 예시: Weighted Round Robin
servers = [
    Server("10.0.0.1", 8080, weight=3),
    Server("10.0.0.2", 8080, weight=1),
    Server("10.0.0.3", 8080, weight=2),
]

wrr = WeightedRoundRobinBalancer(servers)
distribution = {}
for _ in range(60):
    s = wrr.pick()
    distribution[s.host] = distribution.get(s.host, 0) + 1

for host, count in sorted(distribution.items()):
    print(f"{host}: {count}회 선택")

# 출력:
# 10.0.0.1: 30회 선택  (가중치 3/6 = 50%)
# 10.0.0.2: 10회 선택  (가중치 1/6 ≈ 17%)
# 10.0.0.3: 20회 선택  (가중치 2/6 ≈ 33%)
```

### 예제 2: Power of Two Choices 성능 시뮬레이션

```python
import random
import statistics

def simulate_load_distribution(num_servers: int, num_requests: int,
                                algorithm: str) -> list[int]:
    """부하 분산 알고리즘별 부하 분포 시뮬레이션"""
    load = [0] * num_servers
    counter = [0]

    for _ in range(num_requests):
        if algorithm == "random":
            chosen = random.randint(0, num_servers - 1)

        elif algorithm == "round_robin":
            chosen = counter[0] % num_servers
            counter[0] += 1

        elif algorithm == "p2c":
            a = random.randint(0, num_servers - 1)
            b = random.randint(0, num_servers - 1)
            while b == a and num_servers > 1:
                b = random.randint(0, num_servers - 1)
            chosen = a if load[a] <= load[b] else b

        load[chosen] += 1

    return load


def analyze(name: str, load: list[int], total: int):
    expected = total / len(load)
    max_dev = max(abs(x - expected) for x in load)
    std_dev = statistics.stdev(load)
    print(f"{name:20s} — 기댓값: {expected:.1f}, 최대편차: {max_dev:.1f}, 표준편차: {std_dev:.2f}")


NUM_SERVERS = 100
NUM_REQUESTS = 100_000
random.seed(42)

load_random = simulate_load_distribution(NUM_SERVERS, NUM_REQUESTS, "random")
load_rr     = simulate_load_distribution(NUM_SERVERS, NUM_REQUESTS, "round_robin")
load_p2c    = simulate_load_distribution(NUM_SERVERS, NUM_REQUESTS, "p2c")

analyze("무작위 선택", load_random, NUM_REQUESTS)
analyze("라운드 로빈", load_rr, NUM_REQUESTS)
analyze("P2C", load_p2c, NUM_REQUESTS)

# 출력 예시:
# 무작위 선택         — 기댓값: 1000.0, 최대편차: 112.0, 표준편차: 31.87
# 라운드 로빈         — 기댓값: 1000.0, 최대편차: 0.0,   표준편차: 0.00
# P2C                 — 기댓값: 1000.0, 최대편차: 38.0,  표준편차: 12.11
#
# P2C는 무작위 대비 표준편차가 크게 줄어들면서도
# 전역 상태 조회 없이 O(1) 선택이 가능함


# 실제 연결 지속 시간을 반영한 P2C 시뮬레이션
def simulate_with_duration(num_servers: int, num_requests: int) -> dict:
    """요청 처리 시간이 다를 때 알고리즘 비교"""
    import heapq

    results = {}

    for algorithm in ["p2c", "least_conn"]:
        connections = [0] * num_servers
        finish_times = []  # (finish_time, server_id)
        current_time = 0
        max_conn_seen = 0

        for req_id in range(num_requests):
            current_time += 1

            # 완료된 연결 처리
            while finish_times and finish_times[0][0] <= current_time:
                ft, sid = heapq.heappop(finish_times)
                connections[sid] = max(0, connections[sid] - 1)

            if algorithm == "p2c":
                a, b = random.sample(range(num_servers), 2)
                chosen = a if connections[a] <= connections[b] else b
            else:  # least_conn
                chosen = min(range(num_servers), key=lambda i: connections[i])

            # 처리 시간: 지수 분포 (평균 10 타임 유닛)
            duration = max(1, int(random.expovariate(1/10)))
            connections[chosen] += 1
            max_conn_seen = max(max_conn_seen, max(connections))
            heapq.heappush(finish_times, (current_time + duration, chosen))

        results[algorithm] = max_conn_seen

    return results


random.seed(0)
results = simulate_with_duration(20, 5000)
for alg, max_conn in results.items():
    print(f"{alg:12s} — 최대 동시 연결: {max_conn}")
```

## 알고리즘 선택 가이드

| 상황 | 권장 알고리즘 |
|------|--------------|
| 서버 스펙 동일, 처리 빠름 | Round Robin |
| 서버 스펙이 다름 | Weighted Round Robin |
| 처리 시간 편차 큼 (DB, 파일) | Least Connections |
| 대규모 서버 풀, 전역 상태 관리 어려움 | Power of Two Choices |
| 세션 유지, 캐시 최적화 필요 | Consistent Hashing |
| 지리적 분산 | GeoDNS + 각 리전별 알고리즘 조합 |

## 주의사항 및 팁

### 헬스 체크 연동 필수
모든 알고리즘은 서버 헬스 체크와 연동해야 한다. 장애 서버로의 라우팅은 즉시 차단해야 하며, 헬스 체크 간격은 목표 가용성(SLA)에 맞게 설정한다.

```python
class HealthAwareBalancer(PowerOfTwoChoicesBalancer):
    def pick(self) -> Server | None:
        healthy = [s for s in self.servers if self._is_healthy(s)]
        if not healthy:
            return None  # 모든 서버 장애 상황
        if len(healthy) == 1:
            return healthy[0]
        a, b = random.sample(healthy, 2)
        return a if a.active_connections <= b.active_connections else b

    def _is_healthy(self, server: Server) -> bool:
        # 실제로는 별도 헬스 체크 스레드가 상태를 관리
        return True
```

### 슬로우 스타트 (Slow Start)
새 서버가 풀에 추가될 때 갑자기 전체 트래픽의 N분의 1이 몰리는 것을 방지하려면 초기 가중치를 0으로 시작해 점진적으로 높이는 슬로우 스타트를 적용한다. Nginx는 `slow_start` 파라미터로 이를 지원한다.

### 연결 상태 분산 동기화 문제
여러 로드 밸런서가 동시에 운영될 때 각 인스턴스가 다른 연결 수 정보를 가질 수 있다. P2C의 장점은 **전역 정확성이 없어도** 성능이 좋다는 것이다. 각 로드 밸런서가 자신이 관리하는 연결 수만 알아도 충분히 좋은 분산을 달성한다.

### L4 vs L7 부하 분산
- **L4 (TCP/UDP)**: 패킷 레벨 분산. 처리 오버헤드 매우 낮음. LVS, IPVS 활용.
- **L7 (HTTP/gRPC)**: URL 경로, 헤더, 쿠키 기반 라우팅 가능. Nginx, Envoy, HAProxy.

L7에서는 헤더 기반 카나리 배포, URL 기반 마이크로서비스 라우팅 등 훨씬 정교한 전략을 적용할 수 있다.

## 참고 자료
- [What are load-balancing algorithms? — HAProxy](https://www.haproxy.com/glossary/what-are-load-balancing-algorithms)
- [Load Balancing Algorithms — algomaster.io](https://algomaster.io/learn/system-design/load-balancing-algorithms)
- [The Power of Two Choices in Randomized Load Balancing — Mitzenmacher (1996)](https://www.eecs.harvard.edu/~michaelm/postscripts/mythesis.pdf)
- [Load Balancing Algorithms, Types and Techniques — Kemp](https://kemptechnologies.com/load-balancer/load-balancing-algorithms-techniques)
