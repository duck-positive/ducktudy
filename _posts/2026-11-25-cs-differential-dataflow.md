---
layout: post
title: "차분 데이터플로우(Differential Dataflow) 완전 정복: 스트리밍 쿼리를 증분으로 유지하는 혁신적 계산 모델"
date: 2026-11-25
categories: [cs, computer-science]
tags: [differential-dataflow, streaming, incremental-computation, timely-dataflow, materialize, dataflow-programming, distributed-systems]
---

## 차분 데이터플로우란 무엇인가

**차분 데이터플로우(Differential Dataflow, DD)**는 Frank McSherry, Derek Murray, Rebecca Isaacs, Michael Isard가 2013년 CIDR에서 발표한 계산 모델이다. 한 문장으로 요약하면: **입력 컬렉션이 변할 때 출력 컬렉션을 최소 비용으로 갱신하는, 반복(iteration)까지 지원하는 증분 데이터플로우 프레임워크.**

기존 증분 계산의 가장 큰 장벽은 **반복 알고리즘**이었다. 그래프의 연결 컴포넌트, 최단 경로, PageRank는 모두 수렴할 때까지 계산을 반복해야 한다. 이 반복을 증분으로 처리하려면 "현재 반복의 출력이 다음 반복의 입력이 된다"는 순환 의존을 올바르게 다루어야 한다. DD는 이를 **다중 타임스탬프(multi-dimensional timestamps)**라는 우아한 메커니즘으로 해결했다.

DD는 현재 Rust 라이브러리 `timely-dataflow`와 `differential-dataflow`로 구현되어 있으며, 스트리밍 SQL 엔진 **Materialize**의 핵심 엔진이기도 하다.

## 왜 차분 데이터플로우가 필요한가

### 기존 스트리밍 시스템의 한계

Kafka Streams, Flink, Spark Streaming 같은 스트리밍 시스템들은 **윈도우 집계(windowed aggregation)** 에는 강하지만, **반복적 쿼리(iterative query)**에서 취약점을 보인다.

예를 들어 "Twitter 팔로워 그래프에서 PageRank 상위 100명을 실시간으로 구하라"는 쿼리를 생각해보자. 팔로우 관계가 초당 수백 건 변한다. 기존 시스템에서는 두 가지 선택지만 있다:

1. **완전 재계산**: 주기적으로 전체 그래프를 재처리 → 수십 분~수 시간 지연
2. **근사 스트리밍**: 정확한 PageRank 대신 근사값을 추적 → 정확성 포기

DD는 세 번째 선택지를 제공한다: **변경된 엣지만 처리하여 정확한 PageRank를 수 초 이내에 갱신.**

### 차분(Difference)의 개념

DD의 핵심 직관은 모든 컬렉션을 **"무엇이 추가/제거되었는가"의 집합**으로 표현하는 것이다. 각 데이터 아이템은 `(데이터, 타임스탬프, 가중치)` 튜플로 표현된다. 가중치 +1은 "추가", -1은 "제거"를 의미한다.

```
초기 상태: {A:+1, B:+1, C:+1}  → 컬렉션 {A, B, C}
변경 1:   {B:-1, D:+1}          → 컬렉션 {A, C, D}
변경 2:   {A:-1}                → 컬렉션 {C, D}
```

이런 "차분 스트림(difference stream)"을 연산자들이 소비하고 생산한다. `join`, `map`, `filter`, `reduce` 같은 관계 대수 연산자들이 모두 차분-aware하게 구현된다.

## 차분 데이터플로우의 핵심 개념

### 1. 타임스탬프와 부분 순서(Partial Order)

DD에서 타임스탬프는 단순한 정수가 아닌 **격자(lattice)** 위의 원소다. 선형 시간(`t₁ < t₂ < t₃`)과 반복 카운터(`iter₁, iter₂, ...`)를 결합한 2차원 타임스탬프 `(time, iteration)`를 사용한다.

```
(0, 0) → (0, 1) → (0, 2) → (0, converged)
                              ↓
(1, 0) → (1, 1) → (1, 2) → (1, converged)
```

이 2차원 구조 덕분에 "반복 내 계산"과 "다음 배치 처리"를 동시에 올바르게 처리할 수 있다.

### 2. Arrangement: 인덱싱된 차분 컬렉션

단순 차분 스트림을 매번 처음부터 집계하면 여전히 O(전체 데이터) 비용이 든다. DD는 **Arrangement**라는 인덱스 구조로 이를 해결한다.

Arrangement는 `(key, value, time, diff)` 형태의 차분 스트림을 `key`로 정렬된 B-트리 같은 구조에 유지하면서, **컴팩션(compaction)**으로 오래된 차분을 정리한다. 조인 연산은 두 Arrangement를 "차분별로" 매칭하여 출력 차분을 생산한다.

### 3. 반복과 고정점(Fixed Point)

DD에서 반복은 `iterate` 블록으로 표현된다:

```rust
// Rust의 differential-dataflow 스타일 (의사코드)
let result = nodes.iterate(|inner| {
    // inner: 현재 반복의 도달 가능 집합
    inner.join(&edges)
         .map(|(from, to)| to)
         .concat(&inner)
         .distinct()
    // 이전 반복과 동일한 집합이 되면(= 차분이 모두 0) 자동으로 수렴 감지
});
```

타임스탬프의 반복 차원이 증가하면서 차분이 전파되고, 최종적으로 모든 차분이 0이 되면 고정점에 도달한다. 중요한 점은 **입력 그래프의 엣지가 바뀌면, 수렴한 고정점이 최소한의 반복으로 갱신된다**는 것이다.

## 구현 예제 1: 차분 데이터플로우의 연산자 시뮬레이션

```python
from collections import defaultdict
from typing import Any, Callable

# 타임스탬프 타입: (batch_time, iteration)
Timestamp = tuple[int, int]
# 차분 아이템: (데이터, 타임스탬프, 가중치)
DiffItem = tuple[Any, Timestamp, int]

class DiffCollection:
    """차분 아이템의 컬렉션. DD의 핵심 데이터 타입 시뮬레이션"""
    
    def __init__(self, items: list = None):
        self._items: list = items or []

    def consolidate(self) -> 'DiffCollection':
        """같은 (데이터, 타임스탬프)의 가중치를 합산하고 0인 항목 제거"""
        counts: dict = defaultdict(int)
        for data, ts, diff in self._items:
            counts[(data, ts)] += diff
        result = [(d, ts, w) for (d, ts), w in counts.items() if w != 0]
        return DiffCollection(result)

    def map(self, fn: Callable) -> 'DiffCollection':
        """각 아이템에 함수 적용 (가중치와 타임스탬프 보존)"""
        return DiffCollection([(fn(data), ts, diff) for data, ts, diff in self._items])

    def filter(self, pred: Callable) -> 'DiffCollection':
        return DiffCollection([(d, ts, w) for d, ts, w in self._items if pred(d)])

    def to_collection_at(self, ts: tuple) -> dict:
        """주어진 타임스탬프까지의 차분을 집계하여 현재 컬렉션 상태 반환"""
        counts: dict = defaultdict(int)
        for data, item_ts, diff in self._items:
            if item_ts[0] <= ts[0] and item_ts[1] <= ts[1]:
                counts[data] += diff
        return {k: v for k, v in counts.items() if v > 0}


# --- 예제: 차분으로 그래프 엣지 갱신 추적 ---

# 그래프 엣지 컬렉션 (초기 배치 t=0)
edges = DiffCollection([
    (('A', 'B'), (0, 0), +1),
    (('B', 'C'), (0, 0), +1),
    (('A', 'C'), (0, 0), +1),
    (('C', 'D'), (0, 0), +1),
])

print("초기 엣지 집합 (t=0):")
print(edges.to_collection_at((0, 0)))
# {('A','B'):1, ('B','C'):1, ('A','C'):1, ('C','D'):1}

# 배치 t=1: 엣지 (B,C) 제거, (D,E) 추가
update = DiffCollection([
    (('B', 'C'), (1, 0), -1),
    (('D', 'E'), (1, 0), +1),
])

all_diffs = DiffCollection(edges._items + update._items)

print("\n갱신 후 엣지 집합 (t=1):")
print(all_diffs.to_collection_at((1, 0)))
# {('A','B'):1, ('A','C'):1, ('C','D'):1, ('D','E'):1} — B→C 사라짐, D→E 추가

reachable_from_a = all_diffs.filter(lambda e: e[0] == 'A')
print("\n'A'에서 나가는 엣지 (t=1):")
print(reachable_from_a.to_collection_at((1, 0)))
# {('A','B'):1, ('A','C'):1}
```

## 구현 예제 2: 증분 BFS — 그래프 변화 시 최단 경로 갱신

```python
from collections import defaultdict, deque
from dataclasses import dataclass, field

@dataclass
class IncrementalBFS:
    """
    차분 데이터플로우 스타일의 증분 BFS
    
    그래프 엣지가 추가/제거될 때 모든 노드의 거리를 최소 재계산으로 갱신.
    실제 DD의 반복(iterate) + fixed-point 메커니즘을 단순화하여 구현.
    """
    source: str
    _distances: dict = field(default_factory=dict)
    _graph: dict = field(default_factory=lambda: defaultdict(set))
    _reverse_graph: dict = field(default_factory=lambda: defaultdict(set))
    _recompute_count: int = 0

    def _full_bfs(self):
        self._recompute_count += 1
        dist = {self.source: 0}
        queue = deque([self.source])
        while queue:
            node = queue.popleft()
            for neighbor in self._graph[node]:
                if neighbor not in dist:
                    dist[neighbor] = dist[node] + 1
                    queue.append(neighbor)
        self._distances = dist

    def add_edge(self, u: str, v: str):
        if v in self._graph[u]:
            return
        self._graph[u].add(v)
        self._reverse_graph[v].add(u)
        
        if u in self._distances:
            new_dist_v = self._distances[u] + 1
            if v not in self._distances or new_dist_v < self._distances[v]:
                self._incremental_update(v, new_dist_v)

    def remove_edge(self, u: str, v: str):
        if v not in self._graph[u]:
            return
        self._graph[u].discard(v)
        self._reverse_graph[v].discard(u)
        
        if v in self._distances:
            other_dists = [
                self._distances[parent] + 1
                for parent in self._reverse_graph[v]
                if parent in self._distances
            ]
            if other_dists:
                new_dist = min(other_dists)
            elif v == self.source:
                new_dist = 0
            else:
                new_dist = float('inf')
            
            if new_dist != self._distances.get(v, float('inf')):
                self._incremental_update(v, new_dist)

    def _incremental_update(self, start_node: str, new_dist):
        self._recompute_count += 1
        
        if new_dist == float('inf'):
            to_remove = {start_node}
            queue = deque([start_node])
            while queue:
                node = queue.popleft()
                for neighbor in self._graph[node]:
                    if neighbor in self._distances and neighbor not in to_remove:
                        alt = min(
                            (self._distances[p] + 1 for p in self._reverse_graph[neighbor]
                             if p in self._distances and p not in to_remove),
                            default=float('inf')
                        )
                        if alt == float('inf'):
                            to_remove.add(neighbor)
                            queue.append(neighbor)
            for node in to_remove:
                del self._distances[node]
        else:
            queue = deque([(start_node, new_dist)])
            while queue:
                node, dist = queue.popleft()
                if self._distances.get(node, float('inf')) <= dist:
                    continue
                self._distances[node] = dist
                for neighbor in self._graph[node]:
                    if self._distances.get(neighbor, float('inf')) > dist + 1:
                        queue.append((neighbor, dist + 1))


# --- 테스트 ---
bfs = IncrementalBFS(source='S')

for u, v in [('S','A'), ('S','B'), ('A','C'), ('B','C'), ('C','D'), ('B','D')]:
    bfs._graph[u].add(v)
    bfs._reverse_graph[v].add(u)
bfs._full_bfs()

print("초기 거리:")
print({k: v for k, v in sorted(bfs._distances.items())})
# {'A': 1, 'B': 1, 'C': 2, 'D': 2, 'S': 0}

bfs.add_edge('S', 'D')
print("\nS→D 추가 후 거리:")
print({k: v for k, v in sorted(bfs._distances.items())})
# D의 거리: 2 → 1

bfs.remove_edge('S', 'B')
print("\nS→B 제거 후 거리:")
print({k: v for k, v in sorted(bfs._distances.items())})

print(f"\n총 증분 갱신 횟수: {bfs._recompute_count}")
print("(완전 재계산이었다면 3번 전체 BFS가 필요했을 것)")
```

## 실제 활용: Materialize DB와 차분 데이터플로우

**Materialize**는 DD를 기반으로 구축된 스트리밍 SQL 엔진이다. 사용자가 SQL 뷰를 정의하면, 그 뷰를 DD 연산자 그래프로 컴파일하고, 입력 데이터가 변할 때마다 뷰를 증분으로 갱신한다.

```sql
-- Materialize SQL (기본 개념 예시)
-- 이 뷰는 항상 최신 상태로 유지된다 (증분 갱신)
CREATE MATERIALIZED VIEW order_stats AS
  SELECT
    customer_id,
    COUNT(*) AS order_count,
    SUM(amount) AS total_amount
  FROM orders
  GROUP BY customer_id;

-- orders에 새 행이 INSERT/DELETE되면
-- order_stats는 해당 customer_id의 집계만 업데이트 (전체 재집계 없음)
SELECT * FROM order_stats WHERE customer_id = 42;
```

DD의 `join`, `group_by`, `distinct` 연산자가 모두 차분-aware하게 구현되어 있기 때문에 이것이 가능하다.

## 차분 데이터플로우 vs 다른 접근법 비교

| 특성 | 완전 재계산 | Kafka/Flink 스트리밍 | Differential Dataflow |
|------|------------|---------------------|----------------------|
| 반복 알고리즘 지원 | O | △ (복잡) | O |
| 입력 삭제 처리 | O | △ (어려움) | O (음수 차분) |
| 증분 갱신 | X | △ (윈도우 한정) | O |
| 쿼리 표현력 | 최고 | 중간 | 관계 대수 완전 |
| 구현 복잡도 | 낮음 | 중간 | 높음 |

## 주의사항과 실전 팁

**1. 메모리 압박 — Arrangement 크기**: DD의 Arrangement는 모든 과거 차분을 (컴팩션 전까지) 메모리에 유지한다. 긴 시간 동안 많은 변경이 쌓이면 메모리가 급증한다. `compact_frontiers`를 적절히 전진시켜 오래된 차분을 정리해야 한다.

**2. 고기수 키(High-cardinality keys)에서의 조인**: 두 거대한 Arrangement를 조인할 때 출력 차분 수가 폭발할 수 있다. `semijoin`이나 필터링을 먼저 적용하여 Arrangement 크기를 줄이는 것이 핵심 최적화다.

**3. 반복 수렴 보장**: DD의 `iterate`는 모노토닉 연산자여야 고정점 수렴이 보장된다. 집합의 증가(`union + distinct`)는 모노토닉이지만, 임의의 수정은 사이클을 만들 수 있다. `min_reduce` 같은 단조 집계를 사용하라.

**4. Rust 생태계**: 현재 가장 성숙한 DD 구현은 Rust의 `differential-dataflow` 크레이트다. Python 바인딩이 아직 미성숙하므로, 프로덕션 사용은 Rust API 또는 Materialize를 통한 SQL 인터페이스를 권장한다.

**5. 타임스탬프 설계**: 애플리케이션의 시간 개념을 DD 타임스탬프에 올바르게 매핑하는 것이 가장 어려운 설계 결정이다. 단조 증가 타임스탬프를 보장하지 못하면 out-of-order 처리 비용이 급증한다. Kafka 오프셋을 타임스탬프로 쓰는 패턴이 안정적이다.

차분 데이터플로우는 "어떻게 변했는가"를 "어떤 상태인가"만큼 1급 시민으로 다루는 패러다임 전환이다. 복잡한 그래프 쿼리나 반복 알고리즘을 실시간으로 유지해야 하는 시스템에서 DD의 가치는 압도적이며, Materialize를 통해 SQL 개발자도 이 힘을 사용할 수 있게 되었다.

## 참고 자료
- [GitHub — TimelyDataflow/differential-dataflow](https://github.com/TimelyDataflow/differential-dataflow)
- [Differential dataflow — Microsoft Research (CIDR 2013)](https://www.microsoft.com/en-us/research/publication/differential-dataflow/)
- [Frank McSherry's Blog: Differential Dataflow Introduction](https://github.com/frankmcsherry/blog/blob/master/posts/2015-04-07.md)
- [Naiad: A Timely Dataflow System — Microsoft Research](https://www.microsoft.com/en-us/research/project/naiad/)
