---
layout: post
title: "리저버 샘플링(Reservoir Sampling) 완전 정복: 무한 데이터 스트림에서 공정한 표본 추출"
date: 2026-08-07
categories: [cs, computer-science]
tags: [알고리즘, 확률, 리저버샘플링, 스트리밍, 랜덤샘플링, 빅데이터, 분산시스템]
---

## 개념 설명

다음 상황을 상상해 보세요. 인터넷 트래픽 로그가 초당 수백만 건씩 쏟아지고 있습니다. 이 중에서 1,000개를 무작위로 뽑아 분석하고 싶습니다. 단, 로그 전체를 저장할 메모리는 없고, 전체 로그의 수도 모릅니다. 한 번 지나간 로그는 다시 볼 수 없습니다.

이것이 **리저버 샘플링(Reservoir Sampling)**이 해결하는 문제입니다.

> 크기를 모르는 스트림에서, 각 원소를 **동일한 확률**로, **단 한 번의 패스**로, **제한된 메모리**로 k개를 샘플링하라.

리저버 샘플링은 Alan Waterman이 1975년에 제안하고 Jeffrey Scott Vitter가 1985년 논문 "Random Sampling with a Reservoir"에서 체계화한 알고리즘군입니다.

### 핵심 불변 조건(Invariant)

알고리즘이 n번째 원소를 처리한 후, **지금까지 본 n개의 원소 중 어떤 원소든 동일한 확률 k/n으로 리저버에 들어 있어야** 합니다.

이 조건이 스트림이 끝날 때까지 유지되면, 전체 N개 중 k개를 균등하게 샘플링한 것과 동일합니다.

---

### Algorithm R: 기본 리저버 샘플링

가장 단순한 버전인 **Algorithm R** (Vitter, 1985)의 동작 원리:

1. 처음 k개 원소를 **리저버(reservoir)**에 넣습니다.
2. i번째 원소(i > k)에 대해:
   - `[1, i]` 범위의 난수 j를 생성합니다.
   - j ≤ k이면, 리저버의 j번째 원소를 현재 원소로 교체합니다.
   - j > k이면, 현재 원소를 버립니다.

**정확성 증명 (귀납법)**:

- **기저 사례**: i = k일 때, 각 원소가 리저버에 있을 확률 = k/k = 1. 성립.
- **귀납 단계**: i-1번째까지 각 원소가 확률 k/(i-1)로 리저버에 있다고 가정.
  - i번째 원소가 리저버에 들어갈 확률: k/i
  - 기존 원소 x가 리저버에서 살아남을 확률:
    - i번째 원소가 선택되지 않을 확률: (i - k)/i
    - 선택되더라도 x가 교체 대상이 아닐 확률: (k-1)/k
    - 합산: k/(i-1) × [1 - (k/i)(1/k)] = k/(i-1) × (i-1)/i = k/i ✓

각 원소가 정확히 k/i 확률로 리저버에 있음이 증명됩니다.

---

### Algorithm L: 건너뛰기 최적화

Algorithm R는 모든 원소를 방문합니다. 하지만 i번째 원소에서 교체가 발생할 확률은 k/i로 점점 낮아집니다. 대부분의 원소는 그냥 버려집니다.

**Algorithm L** (Li, 1994)은 다음 교체가 일어나는 위치를 확률적으로 **미리 계산**하여 불필요한 원소를 건너뜁니다.

현재 i번째 원소를 처리한 후, 다음 교체가 발생하는 인덱스를 기하 분포(geometric distribution)에서 샘플링합니다. 이로 인해 Algorithm L의 시간 복잡도는 기대 O(k(1 + log(N/k)))로 Algorithm R의 O(N)보다 훨씬 빠릅니다. N이 매우 크거나 k가 작을 때 특히 효과적입니다.

---

## 왜 필요한가?

### 전통적 샘플링의 한계

**균등 무작위 샘플링의 전제 조건**:
1. 전체 크기 N을 알아야 한다.
2. 모든 원소에 랜덤 접근이 가능해야 한다.
3. 필요하다면 여러 번 반복할 수 있다.

리저버 샘플링은 이 세 가지 전제 없이도 동일한 통계적 보장을 제공합니다.

### 현대 시스템에서의 활용

1. **네트워크 패킷 샘플링**: Cisco NetFlow, sFlow 등의 프로토콜이 패킷 샘플링에 리저버 샘플링을 사용합니다.
2. **데이터베이스 TABLESAMPLE**: PostgreSQL과 Oracle은 대용량 테이블에서 행을 샘플링할 때 유사한 원리를 활용합니다.
3. **A/B 테스트 사용자 배정**: 스트리밍 사용자 트래픽에서 실험군/대조군을 동적으로 배정합니다.
4. **온라인 머신러닝**: 무한 데이터 스트림에서 훈련 데이터를 메모리 효율적으로 유지합니다.
5. **분산 집계**: Apache Spark의 `sample()`, `takeSample()` 메서드가 내부적으로 리저버 샘플링 변형을 사용합니다.

### 가중치 리저버 샘플링

각 원소에 가중치가 있는 경우(Weighted Reservoir Sampling)에도 확장 가능합니다. 원소 i의 가중치 w_i에 비례하여 선택될 확률을 조정합니다. Efraimidis & Spirakis (2006)의 Algorithm A-Res가 대표적으로, 각 원소에 `u^(1/w)` (u는 [0,1] 균등 난수)를 할당하여 가장 큰 값 k개를 유지합니다.

---

## 실제 구현 예제

### 예제 1: Algorithm R과 Algorithm L 구현 및 비교 (Python)

```python
import random
import math
from typing import Iterator, TypeVar

T = TypeVar('T')


def reservoir_sample_r(stream: Iterator[T], k: int) -> list[T]:
    """
    Algorithm R (Vitter 1985) — 기본 리저버 샘플링
    시간 복잡도: O(N), 공간 복잡도: O(k)
    """
    reservoir = []

    for i, item in enumerate(stream):
        if i < k:
            reservoir.append(item)
        else:
            # [0, i] 범위 난수. j < k이면 교체
            j = random.randint(0, i)
            if j < k:
                reservoir[j] = item

    return reservoir


def reservoir_sample_l(stream: Iterator[T], k: int) -> list[T]:
    """
    Algorithm L (Li 1994) — 건너뛰기 최적화 버전
    시간 복잡도: O(k * (1 + log(N/k))), 공간 복잡도: O(k)
    """
    reservoir = []
    it = iter(stream)

    # 처음 k개 원소로 리저버 초기화
    for _ in range(k):
        try:
            reservoir.append(next(it))
        except StopIteration:
            return reservoir

    # W: 다음 교체 확률 기반 가중치
    W = math.exp(math.log(random.random()) / k)
    skip = math.floor(math.log(random.random()) / math.log(1 - W))

    for i, item in enumerate(it, start=k):
        if i == k + skip:
            # 리저버의 랜덤 위치에 현재 원소 교체
            reservoir[random.randint(0, k - 1)] = item
            # 다음 건너뛸 위치 계산
            W *= math.exp(math.log(random.random()) / k)
            skip += math.floor(math.log(random.random()) / math.log(1 - W)) + 1
        else:
            skip -= 1 if i < k + skip else 0

    return reservoir


def verify_uniformity(population_size: int, k: int, trials: int = 10000) -> dict:
    """샘플링 균등성 검증: 각 원소가 선택될 실험적 확률"""
    count = {i: 0 for i in range(population_size)}

    for _ in range(trials):
        sample = reservoir_sample_r(iter(range(population_size)), k)
        for item in sample:
            count[item] += 1

    return {i: count[i] / trials for i in range(population_size)}


# 균등성 검증
N, k = 10, 3
empirical_probs = verify_uniformity(N, k)
expected_prob = k / N
print(f"기대 확률: {expected_prob:.4f}")
print("실험적 확률:")
for i, p in empirical_probs.items():
    deviation = abs(p - expected_prob)
    print(f"  원소 {i}: {p:.4f} (편차: {deviation:.4f})")

# 실제 스트리밍 시뮬레이션
def infinite_log_stream():
    """무한 로그 스트림 시뮬레이터"""
    import itertools
    for i in itertools.count(1):
        yield f"log_{i:010d}"

stream = infinite_log_stream()
# 처음 100만 건에서 10개 샘플링
LIMIT = 1_000_000
sample = reservoir_sample_r(
    (next(stream) for _ in range(LIMIT)),
    k=10
)
print(f"\n{LIMIT:,}건 스트림에서 10개 샘플:")
for s in sample:
    print(f"  {s}")
```

### 예제 2: 가중치 리저버 샘플링과 분산 스트림 병합 (Python)

```python
import random
import math
import heapq
from dataclasses import dataclass, field
from typing import Any


@dataclass(order=True)
class WeightedItem:
    """가중치 리저버 샘플링을 위한 원소"""
    priority: float         # u^(1/w), 비교 기준
    item: Any = field(compare=False)
    weight: float = field(compare=False)


def weighted_reservoir_sample(
    stream: list[tuple[Any, float]],
    k: int
) -> list[Any]:
    """
    Algorithm A-Res (Efraimidis & Spirakis 2006)
    가중치에 비례하여 원소를 샘플링 (비복원 추출)
    stream: [(item, weight), ...]
    """
    heap = []  # 최소 힙 (크기 k 유지)

    for item, weight in stream:
        if weight <= 0:
            continue
        # 키: u^(1/weight), u ~ Uniform(0, 1)
        u = random.random()
        priority = u ** (1.0 / weight)

        if len(heap) < k:
            heapq.heappush(heap, WeightedItem(priority, item, weight))
        elif priority > heap[0].priority:
            heapq.heapreplace(heap, WeightedItem(priority, item, weight))

    return [wi.item for wi in heap]


def merge_reservoir_samples(
    samples: list[list],
    weights: list[float],
    k: int
) -> list:
    """
    분산 환경에서 여러 리저버 샘플 병합
    각 파티션의 샘플을 결합하여 전체를 대표하는 샘플 생성
    """
    combined = []
    for sample, weight in zip(samples, weights):
        for item in sample:
            combined.append((item, weight / len(sample)))

    return weighted_reservoir_sample(combined, k)


# 가중치 샘플링 테스트
items_with_weights = [
    ("apple", 10),    # 높은 가중치
    ("banana", 5),
    ("cherry", 2),
    ("date", 1),      # 낮은 가중치
    ("elderberry", 8),
    ("fig", 3),
    ("grape", 6),
]

N_TRIALS = 50_000
k = 3
selection_count = {name: 0 for name, _ in items_with_weights}

for _ in range(N_TRIALS):
    sample = weighted_reservoir_sample(items_with_weights, k)
    for item in sample:
        selection_count[item] += 1

total_weight = sum(w for _, w in items_with_weights)
print("가중치 샘플링 결과 (기대 비율 vs 실험 비율):")
for name, weight in items_with_weights:
    expected = weight / total_weight
    actual = selection_count[name] / N_TRIALS
    print(f"  {name:12s}: 기대={expected:.3f}, 실험={actual:.3f}, 편차={abs(expected-actual):.4f}")


# 분산 병합 시뮬레이션
print("\n분산 리저버 샘플 병합:")
partition1 = list(range(0, 50))
partition2 = list(range(50, 100))
partition3 = list(range(100, 150))

sample1 = reservoir_sample_r(iter(partition1), 10)
sample2 = reservoir_sample_r(iter(partition2), 10)
sample3 = reservoir_sample_r(iter(partition3), 10)

# 각 파티션은 50개 원소 → 동일 가중치 50
merged = merge_reservoir_samples(
    [sample1, sample2, sample3],
    [50.0, 50.0, 50.0],
    k=10
)
print(f"병합된 샘플 (10개): {sorted(merged)}")
```

---

## 주의사항 및 팁

### 통계적 정확성 보장

1. **난수 품질이 중요합니다**: `random.randint`는 편향 없는 의사난수 생성기(PRNG)를 사용합니다. 암호학적으로 안전한 난수가 필요하다면 `secrets` 모듈을 사용하세요. 단, 성능이 100배 이상 느려집니다.

2. **정수 오버플로 주의**: Algorithm R에서 스트림의 원소 수 i가 매우 크면(>2³¹) `random.randint(0, i)`가 Python에서는 문제없지만, C/Java로 구현 시 정수 오버플로가 발생할 수 있습니다.

3. **병렬/분산 환경**: 멀티스레드 환경에서 리저버를 공유하면 경쟁 조건이 발생합니다. 각 스레드가 독립 리저버를 유지한 뒤 가중치 병합을 수행하거나, `threading.Lock`으로 접근을 직렬화하세요.

### 알고리즘 선택 가이드

| 상황 | 추천 알고리즘 | 이유 |
|------|-------------|------|
| 스트림 속도가 매우 빠를 때 | Algorithm L | O(k log(N/k))로 건너뛰기 최적화 |
| 구현 단순성 우선 | Algorithm R | 코드 10줄 이내 구현 가능 |
| 가중치 샘플링 필요 | Algorithm A-Res | 가중치 비례 확률 보장 |
| 분산 환경 병합 필요 | 가중치 병합 방식 | 각 파티션을 독립 처리 후 합산 |
| 시계열/시간 감쇠 필요 | 지수 가중 리저버 | 최근 원소에 높은 가중치 부여 |

### 흔한 실수

```python
# 잘못된 예시 1: 0-indexed와 1-indexed 혼동
# random.randint(0, i-1)을 써야 할 곳에 random.randint(0, i)를 쓰면
# 마지막 원소가 선택될 확률이 낮아짐 (비균등 분포)

# 잘못된 예시 2: k개 이하 스트림 처리 미비
def bad_sample(stream, k):
    reservoir = list(stream)  # 전체를 메모리에 적재! 목적에 어긋남
    return random.sample(reservoir, min(k, len(reservoir)))

# 올바른 예시: 스트림이 k개보다 짧을 경우 그대로 반환
def good_sample(stream, k):
    reservoir = []
    for i, item in enumerate(stream):
        if i < k:
            reservoir.append(item)
        else:
            j = random.randint(0, i)
            if j < k:
                reservoir[j] = item
    return reservoir  # 스트림이 k개 미만이면 전체 반환
```

### 성능 벤치마크 팁

실제 시스템에서 리저버 샘플링 성능을 측정할 때:
- 난수 생성이 가장 큰 병목입니다. `numpy.random`의 벡터화된 난수 생성을 활용하면 순수 Python 대비 10~50배 빨라집니다.
- Algorithm L의 건너뛰기 계산에도 `numpy`를 활용하면 대규모 스트림에서 효과적입니다.
- Java의 `ThreadLocalRandom`이나 C++의 `<random>` 라이브러리가 `rand()`보다 훨씬 품질 높은 난수를 생성합니다.

리저버 샘플링은 단순하지만 강력한 알고리즘입니다. 데이터 엔지니어링, 시스템 모니터링, 온라인 학습 등 다양한 분야에서 "어떻게 무한한 데이터를 유한한 메모리로 대표할 것인가"라는 질문에 답하는 핵심 도구입니다.

## 참고 자료
- [Reservoir Sampling - Wikipedia](https://en.wikipedia.org/wiki/Reservoir_sampling)
- [Random Sampling with a Reservoir - Vitter (1985), cs.umd.edu](https://www.cs.umd.edu/~samir/498/vitter.pdf)
- [Reservoir Sampling - GeeksforGeeks](https://www.geeksforgeeks.org/dsa/reservoir-sampling/)
- [Efficient Random Sampling - Parallel, Vectorized, Cache-Efficient (arXiv)](https://arxiv.org/abs/1610.05141)
