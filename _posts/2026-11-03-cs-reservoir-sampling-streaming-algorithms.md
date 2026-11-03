---
layout: post
title: "Reservoir Sampling과 온라인 스트리밍 알고리즘 심층 분석"
date: 2026-11-03
categories: [cs, computer-science]
tags: [reservoir-sampling, streaming-algorithms, randomized-algorithms, probability, statistics, algorithm]
---

데이터 스트림은 끝을 알 수 없는 흐름이다. 트위터 피드, 실시간 로그, IoT 센서 데이터는 메모리에 모두 올릴 수 없을 정도로 방대하게 쏟아진다. **Reservoir Sampling(저수지 표본 추출)**은 전체 크기를 모르는 스트림에서 단 **한 번의 패스**로 균등하게 무작위 샘플을 추출하는 알고리즘이다. 이 포스트에서는 기본 알고리즘부터 최적화 변형, 그리고 실무 응용까지 깊이 파고든다.

## 왜 스트리밍 알고리즘이 필요한가

전통적인 알고리즘은 데이터를 RAM에 모두 올린다고 가정하지만, 현실은 다르다.

- **데이터 크기가 메모리를 초과**: 수십억 개의 로그 레코드를 모두 저장할 수 없다
- **스트림 길이를 미리 알 수 없음**: 실시간 데이터는 언제 끝날지 알 수 없다
- **단일 패스 처리**: 데이터를 여러 번 읽는 비용이 너무 크다

스트리밍 알고리즘은 **O(k) 공간** (k = 샘플 크기)과 **O(1) 또는 O(log n) 시간** per item으로 처리하는 것을 목표로 한다.

## 알고리즘 R: 기본 Reservoir Sampling

1980년 Alan Waterman이 제안하고 Jeffrey Vitter가 1985년 "Random sampling with a reservoir" 논문에서 정형화했다.

**핵심 아이디어**: n번째 원소는 확률 k/n으로 저수지(reservoir)에 선택된다.

**귀납적 증명**:
- k개 원소 이후, 각 원소가 저수지에 있을 확률 = 1 (k/k = 1)
- (k+1)번째 원소: 확률 k/(k+1)로 선택됨. 기존 원소 하나가 확률 1/(k+1)로 제거되므로, 기존 각 원소가 남을 확률 = k/(k+1) × (1 - 1/k) = k/(k+1) × (k-1)/k = (k-1)/(k+1) ... 
  
  좀 더 정확히: 원소 i (i ≤ k)가 n번째 스텝 후에도 저수지에 있을 확률을 귀납법으로 보이면, 항상 k/n임을 증명할 수 있다.

```python
import random
from typing import List, TypeVar, Iterator

T = TypeVar('T')

def reservoir_sample_r(stream: Iterator[T], k: int) -> List[T]:
    """
    Algorithm R: Vitter(1985)
    시간복잡도: O(n), 공간복잡도: O(k)
    """
    reservoir = []

    for i, item in enumerate(stream):
        if i < k:
            # 처음 k개는 무조건 저수지에 채움
            reservoir.append(item)
        else:
            # i번째 원소를 [0, i] 중 랜덤 위치에 삽입 시도
            j = random.randint(0, i)
            if j < k:
                reservoir[j] = item

    return reservoir


# 검증: 1~1000 중 10개 샘플, 각 숫자의 선택 확률이 균등한지 확인
from collections import Counter
import statistics

N_TRIALS = 100_000
k = 10
population_size = 1000
counts = Counter()

for _ in range(N_TRIALS):
    sample = reservoir_sample_r(iter(range(1, population_size + 1)), k)
    counts.update(sample)

# 이론적으로 각 숫자가 선택될 기댓값 = N_TRIALS * k / population_size = 1000
expected = N_TRIALS * k / population_size
actual_values = list(counts.values())
print(f"Expected count per item: {expected:.1f}")
print(f"Mean actual: {statistics.mean(actual_values):.1f}")
print(f"Std dev: {statistics.stdev(actual_values):.1f}")
# 출력: 평균 ≈ 1000, 표준편차는 낮음 → 균등 분포 확인
```

## 알고리즘 L: O(n/k) 건너뜀으로 최적화

Algorithm R는 n - k번의 `randint` 호출과 조건 분기가 필요하다. **Algorithm L**은 다음 원소가 저수지에 들어올 때까지 몇 개를 건너뛸지를 기하 분포(Geometric Distribution)로 한 번에 계산한다.

```python
import math

def reservoir_sample_l(stream: Iterator[T], k: int) -> List[T]:
    """
    Algorithm L: Kim-Park(1994), Vitter의 개선
    시간복잡도: O(k + k·log(n/k)), 공간복잡도: O(k)
    """
    reservoir = []
    stream_iter = iter(stream)

    # 처음 k개 채우기
    for item in stream_iter:
        reservoir.append(item)
        if len(reservoir) == k:
            break

    if len(reservoir) < k:
        return reservoir

    # W: 다음 교체까지 건너뛸 간격을 계산하기 위한 키
    W = math.exp(math.log(random.random()) / k)
    # 다음 교체 위치
    i = k - 1 + math.floor(math.log(random.random()) / math.log(1 - W)) + 1

    current_index = k - 1
    for item in stream_iter:
        current_index += 1
        if current_index == int(i):
            # 저수지의 랜덤한 위치와 교체
            reservoir[random.randint(0, k - 1)] = item
            # 다음 교체 위치 갱신
            W *= math.exp(math.log(random.random()) / k)
            i += math.floor(math.log(random.random()) / math.log(1 - W)) + 1

    return reservoir


# Algorithm R vs Algorithm L 성능 비교
import time

large_stream_size = 10_000_000
k = 1000

# Algorithm R
start = time.perf_counter()
result_r = reservoir_sample_r(iter(range(large_stream_size)), k)
time_r = time.perf_counter() - start

# Algorithm L
start = time.perf_counter()
result_l = reservoir_sample_l(iter(range(large_stream_size)), k)
time_l = time.perf_counter() - start

print(f"Algorithm R: {time_r:.3f}s, sample size: {len(result_r)}")
print(f"Algorithm L: {time_l:.3f}s, sample size: {len(result_l)}")
# Algorithm L이 약 2~5배 빠름 (건너뜀 덕분에 루프 횟수 감소)
```

## 가중치 있는 Reservoir Sampling

각 원소가 서로 다른 가중치를 가질 때, 가중치에 비례한 확률로 샘플링해야 하는 경우가 있다. **Efraimidis-Spirakis (2006)** 알고리즘은 각 원소에 키 `u^(1/w_i)` (u는 균등 난수, w_i는 가중치)를 할당하고 상위 k개 키를 가진 원소를 선택한다.

```python
import heapq

def weighted_reservoir_sample(
    items_weights: Iterator[tuple],  # (item, weight) 쌍
    k: int
) -> List:
    """
    Efraimidis-Spirakis Algorithm (2006)
    가중치에 비례하는 확률로 k개 샘플링
    """
    heap = []  # (key, item) 의 min-heap

    for item, weight in items_weights:
        if weight <= 0:
            continue
        # 키 = u^(1/weight), u ~ Uniform(0,1)
        u = random.random()
        key = u ** (1.0 / weight)

        if len(heap) < k:
            heapq.heappush(heap, (key, item))
        elif key > heap[0][0]:
            heapq.heapreplace(heap, (key, item))

    return [item for _, item in heap]


# 예시: 가중치가 높은 원소가 더 자주 선택되는지 확인
items = [('A', 1), ('B', 5), ('C', 10), ('D', 1), ('E', 3)]
counts = Counter()

for _ in range(50_000):
    sample = weighted_reservoir_sample(iter(items), k=2)
    counts.update(sample)

print("Weighted sampling frequencies:")
total = sum(counts.values())
for item, cnt in sorted(counts.items()):
    weight = dict(items)[item]
    print(f"  {item} (weight={weight}): {cnt/total*100:.1f}%")
# C(weight=10)가 약 50%, B(weight=5)가 약 25%, E(weight=3)가 약 15% 등으로 나타남
```

## Count-Min Sketch와의 조합: 스트리밍 Heavy Hitter 탐지

실무에서 Reservoir Sampling은 다른 스트리밍 알고리즘과 조합해 사용된다. 예를 들어 **Count-Min Sketch**로 빈도가 높은 원소를 식별한 후, 해당 원소들을 우선적으로 샘플에 유지할 수 있다.

```python
import hashlib

class CountMinSketch:
    def __init__(self, width: int = 1000, depth: int = 5):
        self.width = width
        self.depth = depth
        self.table = [[0] * width for _ in range(depth)]
        self.seeds = [i * 2654435761 for i in range(1, depth + 1)]

    def _hash(self, item: str, seed: int) -> int:
        h = hashlib.md5(f"{seed}{item}".encode()).hexdigest()
        return int(h, 16) % self.width

    def add(self, item: str, count: int = 1):
        for d, seed in enumerate(self.seeds):
            col = self._hash(item, seed)
            self.table[d][col] += count

    def estimate(self, item: str) -> int:
        return min(
            self.table[d][self._hash(item, seed)]
            for d, seed in enumerate(self.seeds)
        )


# 스트림에서 상위 빈도 원소 탐지 예시
cms = CountMinSketch()
stream = ['apple'] * 500 + ['banana'] * 200 + ['cherry'] * 50 + ['date'] * 10
random.shuffle(stream)

for item in stream:
    cms.add(item)

for word in ['apple', 'banana', 'cherry', 'date']:
    print(f"  {word}: estimated count = {cms.estimate(word)}")
```

## 분산 환경에서의 Reservoir Sampling

여러 노드에서 병렬로 Reservoir Sampling을 실행하고 결과를 병합하는 방법이 필요하다. **Merge 가능한 Reservoir**는 각 노드에서 k개씩 샘플을 추출하고, 전체 스트림 크기에 비례하여 가중치를 부여해 병합한다.

- 노드 i가 n_i개의 원소를 처리했다면 전체 N = Σn_i
- 노드 i의 샘플에서 각 원소가 최종 샘플에 포함될 확률은 k·n_i/N
- 전체 샘플을 `k·n_i/(k·N)` 비율로 병합하면 균등 분포 유지

## 주의사항 및 실무 팁

**1. 유사 난수 품질**: Python의 `random.random()`은 Mersenne Twister 기반으로 대부분 충분하지만, 보안이 필요한 샘플링에는 `secrets` 모듈을 사용하라.

**2. 부동소수점 정밀도**: Algorithm L에서 `math.log(0.0)`이 발생할 수 있다. `random.random()`이 0을 반환할 확률은 매우 낮지만, 방어 코드(`max(1e-10, u)`)를 추가하라.

**3. 스트림이 비어있을 때**: 스트림이 k개 미만이면 모든 원소가 저수지에 들어간다. 반환값의 크기가 k보다 작을 수 있음을 처리해야 한다.

**4. 재현 가능성**: 디버깅을 위해 `random.seed(42)`로 시드를 고정하라. 단, 프로덕션에서는 시드 고정이 보안 취약점이 될 수 있다.

Reservoir Sampling은 "단순하지만 강력한" 알고리즘의 교과서적 사례다. 수학적으로 우아하고, 구현이 직관적이며, 실무에서 즉시 활용 가능하다. 빅데이터 파이프라인, A/B 테스트 샘플 선택, 실시간 모니터링 등 어디서나 만날 수 있다.

## 참고 자료
- [Reservoir Sampling – Wikipedia](https://en.wikipedia.org/wiki/Reservoir_sampling)
- [Random Sampling with a Reservoir – Jeffrey Vitter (1985)](https://dl.acm.org/doi/10.1145/3147.3165)
- [Weighted random sampling with a reservoir – Efraimidis & Spirakis (2006)](https://www.sciencedirect.com/science/article/pii/S002001900500298X)
- [Streaming Algorithms – Duke University Lecture Notes](https://www.cs.duke.edu/~reif/courses/alglectures/jain.lectures/StreamingAlgorithms.pdf)
