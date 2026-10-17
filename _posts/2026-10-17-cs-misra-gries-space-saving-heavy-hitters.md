---
layout: post
title: "Misra-Gries와 Space-Saving: 데이터 스트림에서 Top-K 빈번 아이템을 찾는 결정론적 알고리즘"
date: 2026-10-17
categories: [cs, computer-science]
tags: [스트리밍알고리즘, Misra-Gries, Space-Saving, Heavy-Hitters, Top-K, 데이터스트림, 확률적자료구조]
---

## 스트리밍 알고리즘이 필요한 이유

하루에 수십억 건의 검색어가 처리되는 검색 엔진, 초당 수백만 패킷이 지나가는 네트워크 라우터, 수천만 명이 동시에 사용하는 소셜미디어 플랫폼. 이 시스템들에서 "가장 많이 검색된 키워드 100개", "가장 많은 트래픽을 차지하는 IP 주소 10개", "지금 가장 화제인 해시태그"를 구하려면 어떻게 해야 할까요?

**핵심 제약:** 데이터는 무한히 흐르는 스트림이며, 모든 데이터를 메모리에 저장할 수 없습니다.

전통적 접근법인 해시맵은 O(1)로 빠르지만 데이터 종류가 무한히 늘어날 수 있어 메모리가 부족합니다. 정렬은 전체 데이터를 봐야 해서 스트림에 적용할 수 없습니다.

이 문제를 **Heavy Hitters Problem** 또는 **Frequent Items Problem** 이라고 합니다. 이 글에서는 이를 해결하는 두 가지 결정론적(deterministic) 알고리즘인 **Misra-Gries**와 **Space-Saving**을 깊이 있게 다룹니다.

## 문제 정의

**φ-Frequent Items 문제:**
- 스트림 σ = ⟨a₁, a₂, ..., aₙ⟩ (n은 스트림 길이, 사전에 알 수 없음)
- 각 원소 aᵢ는 우주 U에 속함 (|U|는 매우 크거나 무한)
- 목표: 등장 횟수 f(e) ≥ φn 인 모든 원소 e를 찾기 (φ는 빈도 임계값)

**Top-K 문제:**
- 전체 스트림에서 빈도 상위 K개 원소를 찾기

이를 메모리 O(1/φ) 또는 O(K)에 해결할 수 있어야 합니다. **HyperLogLog**가 "고유 원소 수"를 추정한다면, Misra-Gries와 Space-Saving은 "빈번한 원소"를 찾습니다.

## Misra-Gries 알고리즘 (1982)

J. Misra와 David Gries가 1982년 발표한 알고리즘입니다. 핵심 아이디어는 놀랍도록 단순합니다.

### 핵심 아이디어: 취소(Cancellation)

k개의 카운터를 유지합니다. 새 원소가 들어올 때:
1. 이미 카운터가 있으면 해당 카운터 증가
2. 없고 빈 슬롯이 있으면 새 카운터 생성 (카운트 = 1)
3. 없고 슬롯이 꽉 찼으면 **모든 카운터를 1 감소, 0이 된 것 제거**

이 "취소" 과정이 핵심입니다. k개의 서로 다른 원소가 서로를 취소하면, 빈번하지 않은 원소들이 자연스럽게 제거됩니다.

**보장:** k개의 카운터를 유지할 때, 실제 빈도 f(e) > n/k 인 원소는 반드시 최종 카운터에 포함됩니다.

```python
from collections import OrderedDict
from typing import Iterator, List, Tuple, Any

class MisraGries:
    """
    Misra-Gries 빈번 아이템 알고리즘.
    k-1개의 카운터로 빈도 > n/k인 원소를 보장하여 탐지.
    """
    def __init__(self, k: int):
        if k < 2:
            raise ValueError("k는 2 이상이어야 합니다")
        self.k = k            # 추적할 최대 카운터 수 (실제 보장: k-1개 빈번 원소)
        self.counters: dict = {}
        self.total = 0        # 처리한 원소 총 수

    def process(self, element: Any) -> None:
        """스트림 원소 하나를 처리"""
        self.total += 1

        if element in self.counters:
            # 케이스 1: 이미 추적 중 → 카운터 증가
            self.counters[element] += 1
        elif len(self.counters) < self.k - 1:
            # 케이스 2: 빈 슬롯 있음 → 새 카운터 생성
            self.counters[element] = 1
        else:
            # 케이스 3: 꽉 참 → 모든 카운터 1 감소, 0이면 제거
            to_delete = []
            for key in self.counters:
                self.counters[key] -= 1
                if self.counters[key] == 0:
                    to_delete.append(key)
            for key in to_delete:
                del self.counters[key]
            # 현재 원소는 등록하지 않음 (취소로 소멸)

    def get_frequent(self, min_count: int = 0) -> List[Tuple[Any, int]]:
        """추정 빈도가 min_count 이상인 원소 반환"""
        return sorted(
            [(k, v) for k, v in self.counters.items() if v >= min_count],
            key=lambda x: -x[1]
        )

    def error_bound(self) -> float:
        """빈도 추정 오차 상한: n / k"""
        return self.total / self.k

    def query(self, element: Any) -> int:
        """
        특정 원소의 빈도 추정값 반환.
        실제 빈도 f(e)에 대해: counter[e] ≤ f(e) ≤ counter[e] + n/k
        """
        return self.counters.get(element, 0)


# 사용 예시
stream = ['a', 'b', 'a', 'c', 'a', 'b', 'a', 'd', 'a', 'b',
          'a', 'e', 'a', 'f', 'a', 'g', 'a', 'h', 'a', 'b']

mg = MisraGries(k=4)  # 3개의 카운터 유지 (k-1 = 3)
for item in stream:
    mg.process(item)

print(f"총 원소 수: {mg.total}")
print(f"오차 상한: {mg.error_bound():.2f}")
print(f"빈번 원소 후보: {mg.get_frequent()}")
# 출력 예시: 빈번 원소 후보: [('a', 9), ('b', 3)]
```

### 정확도 보장 증명

**정리:** Misra-Gries 알고리즘(k개 카운터)에서
1. 실제 빈도 f(e) > n/k 인 원소는 반드시 카운터에 남습니다.
2. 카운터 값 c̃(e)는 과소추정입니다: c̃(e) ≤ f(e)
3. 추정 오차는 n/k 이하입니다: f(e) - c̃(e) ≤ n/k

**증명 스케치:**
- 케이스 3(취소)이 발생할 때마다 정확히 k개의 원소가 카운터에서 제거됩니다 (k-1개 기존 원소 + 1개 현재 원소).
- 전체 취소 횟수를 d라 하면, 총 제거 원소 수 = k*d ≤ n이므로 d ≤ n/k
- 어떤 원소 e의 카운터 감소량은 d 이하이므로: f(e) - c̃(e) ≤ d ≤ n/k

## Space-Saving 알고리즘 (Metwally, 2005)

Ahmed Metwally, Divyakant Agrawal, Amr El Abbadi가 2005년 발표한 알고리즘입니다. Misra-Gries의 단점을 보완하여 모든 원소를 항상 추적합니다.

### 핵심 차이: 대체(Replacement) 방식

Space-Saving은 카운터가 꽉 찼을 때 원소를 **제거하지 않고 최소 카운터 원소를 현재 원소로 교체**합니다.

```
새 원소 e 처리:
  if e in counters:
    counters[e] += 1
  else if |counters| < k:
    counters[e] = 1
  else:
    // 최소 카운터 원소 m을 찾음
    min_elem, min_count = min(counters)
    counters.remove(min_elem)
    counters[e] = min_count + 1  // 교체 후 증가
    error[e] = min_count         // 오차 추적
```

```python
import heapq
from collections import defaultdict

class SpaceSaving:
    """
    Space-Saving Top-K 알고리즘.
    k개의 카운터로 상위 K개 빈번 원소를 추적.
    """
    def __init__(self, k: int):
        self.k = k
        self.counters: dict = {}    # {element: count}
        self.errors: dict = {}      # {element: max_error} (교체 시 오차)
        self.total = 0
        # 최솟값 빠른 접근을 위한 힙 (count, element)
        self._heap: list = []       # min-heap
        self._heap_dict: dict = {}  # {element: heap_entry}

    def _push(self, element: Any, count: int) -> None:
        entry = [count, element]
        self._heap_dict[element] = entry
        heapq.heappush(self._heap, entry)

    def _invalidate(self, element: Any) -> None:
        """힙에서 원소를 무효화 (lazy deletion)"""
        entry = self._heap_dict.pop(element, None)
        if entry:
            entry[0] = float('inf')  # 힙 정렬에서 제외

    def _get_min(self) -> Tuple[int, Any]:
        """현재 최솟값 원소 반환"""
        while self._heap:
            count, elem = self._heap[0]
            if elem in self._heap_dict and self._heap_dict[elem][0] == count:
                return count, elem
            heapq.heappop(self._heap)
        return None, None

    def process(self, element: Any) -> None:
        self.total += 1

        if element in self.counters:
            # 이미 추적 중: 카운터 증가
            self._invalidate(element)
            self.counters[element] += 1
            self._push(element, self.counters[element])

        elif len(self.counters) < self.k:
            # 빈 슬롯: 새 원소 추가
            self.counters[element] = 1
            self.errors[element] = 0
            self._push(element, 1)

        else:
            # 슬롯 꽉 참: 최소 카운터 원소를 현재 원소로 교체
            min_count, min_elem = self._get_min()
            if min_elem is None:
                return

            # 최솟값 원소 제거
            self._invalidate(min_elem)
            del self.counters[min_elem]
            del self.errors[min_elem]

            # 현재 원소를 (min_count + 1)로 추가
            new_count = min_count + 1
            self.counters[element] = new_count
            self.errors[element] = min_count  # 오차: 교체 전 최소 카운터
            self._push(element, new_count)

    def top_k(self, k: int = None) -> List[Tuple[Any, int, int]]:
        """
        상위 k개 원소 반환.
        반환: [(원소, 추정_빈도, 최대_오차), ...]
        """
        if k is None:
            k = self.k
        results = []
        for elem, count in self.counters.items():
            error = self.errors.get(elem, 0)
            results.append((elem, count, error))
        return sorted(results, key=lambda x: -x[1])[:k]

    def query(self, element: Any) -> Tuple[int, int]:
        """
        특정 원소 (추정_빈도, 최대_오차) 반환.
        실제 빈도 f(e)에 대해:
          count - error ≤ f(e) ≤ count
        """
        count = self.counters.get(element, 0)
        error = self.errors.get(element, 0)
        return count, error


# Space-Saving 사용 예시
import random
random.seed(42)

# Zipf 분포 스트림 생성 (현실적인 분포)
def generate_zipf_stream(n: int, vocab_size: int, alpha: float = 1.5):
    weights = [1.0 / (i ** alpha) for i in range(1, vocab_size + 1)]
    total = sum(weights)
    probs = [w / total for w in weights]
    items = [f"item_{i}" for i in range(1, vocab_size + 1)]
    return random.choices(items, weights=probs, k=n)

stream = generate_zipf_stream(n=10000, vocab_size=1000)
true_counts = {}
for item in stream:
    true_counts[item] = true_counts.get(item, 0) + 1
true_top10 = sorted(true_counts.items(), key=lambda x: -x[1])[:10]

# Space-Saving으로 Top-10 추정
ss = SpaceSaving(k=50)
for item in stream:
    ss.process(item)

estimated_top10 = ss.top_k(10)

print("=== 실제 Top-10 vs Space-Saving 추정 ===")
print(f"{'순위':<4} {'실제 원소':<12} {'실제 빈도':<10} {'추정 빈도':<10} {'최대 오차':<10}")
for i, ((true_elem, true_cnt), (est_elem, est_cnt, error)) in \
        enumerate(zip(true_top10, estimated_top10)):
    print(f"{i+1:<4} {true_elem:<12} {true_cnt:<10} {est_cnt:<10} {error:<10}")
```

### Space-Saving의 이론적 보장

| 속성 | Misra-Gries | Space-Saving |
|------|------------|-------------|
| 메모리 | O(k) | O(k) |
| 처리 시간 (원소당) | O(k) 최악, O(1) 평균 | O(log k) (힙 사용) |
| 항상 k개 유지 | 아니오 (줄어들 수 있음) | 예 |
| 상위 K 순위 보존 | 부분적 | 완전 보장 |
| 오차 상한 | n/k | n/k |
| 빈도 과대추정 가능 | 아니오 (과소추정) | 예 (과대추정 가능) |

**Space-Saving 핵심 보장:**
- 카운터에 있는 원소는 실제 빈도 > 0
- counter[e] - error[e] ≤ f(e) ≤ counter[e]
- 실제 빈도 f(e) > n/k 이면 반드시 Top-K에 포함

## 두 알고리즘의 실전 비교

```python
def compare_algorithms(stream_size: int = 50000,
                        vocab_size: int = 5000,
                        k: int = 100):
    """두 알고리즘의 정밀도/재현율 비교"""
    stream = generate_zipf_stream(stream_size, vocab_size)

    # 실제 빈도
    true_counts = {}
    for item in stream:
        true_counts[item] = true_counts.get(item, 0) + 1

    threshold = stream_size / k  # φ = 1/k
    true_heavy_hitters = {e for e, c in true_counts.items() if c > threshold}

    # Misra-Gries
    mg = MisraGries(k=k)
    for item in stream:
        mg.process(item)
    mg_results = {e for e, c in mg.get_frequent(min_count=1)}

    # Space-Saving
    ss = SpaceSaving(k=k)
    for item in stream:
        ss.process(item)
    ss_results = {e for e, _, _ in ss.top_k(k)}

    def metrics(found: set, true: set) -> dict:
        tp = len(found & true)
        fp = len(found - true)
        fn = len(true - found)
        precision = tp / (tp + fp) if (tp + fp) > 0 else 0
        recall    = tp / (tp + fn) if (tp + fn) > 0 else 0
        f1 = 2 * precision * recall / (precision + recall) \
             if (precision + recall) > 0 else 0
        return {'precision': precision, 'recall': recall, 'f1': f1,
                'found': len(found), 'true_heavy': len(true)}

    mg_metrics = metrics(mg_results, true_heavy_hitters)
    ss_metrics = metrics(ss_results, true_heavy_hitters)

    print(f"\n스트림 크기: {stream_size}, 어휘: {vocab_size}, k: {k}")
    print(f"실제 Heavy Hitters: {mg_metrics['true_heavy']}개")
    print(f"\n{'알고리즘':<20} {'정밀도':>8} {'재현율':>8} {'F1':>8} {'탐지':>6}")
    print("-" * 55)
    print(f"{'Misra-Gries':<20} {mg_metrics['precision']:>8.3f} "
          f"{mg_metrics['recall']:>8.3f} {mg_metrics['f1']:>8.3f} "
          f"{mg_metrics['found']:>6}")
    print(f"{'Space-Saving':<20} {ss_metrics['precision']:>8.3f} "
          f"{ss_metrics['recall']:>8.3f} {ss_metrics['f1']:>8.3f} "
          f"{ss_metrics['found']:>6}")

compare_algorithms()
```

## 실제 시스템에서의 활용

### 분산 환경에서의 병합 가능성(Mergeability)

대형 시스템에서는 여러 노드에서 각자 Misra-Gries나 Space-Saving을 실행하고, 나중에 결과를 병합해야 합니다.

```python
def merge_misra_gries(summaries: List[MisraGries], k: int) -> MisraGries:
    """여러 Misra-Gries 요약을 하나로 병합"""
    merged = MisraGries(k=k)

    # 모든 카운터 합산
    combined = {}
    for mg in summaries:
        for elem, count in mg.counters.items():
            combined[elem] = combined.get(elem, 0) + count

    # 다시 k-1개로 줄이기: 가장 작은 값을 기준으로 취소
    if len(combined) > k - 1:
        # 취소 임계값 계산
        sorted_counts = sorted(combined.values())
        # 상위 k-1개만 남기도록 임계값 설정
        threshold = sorted_counts[-(k-1)] if len(sorted_counts) >= k-1 else 0
        adjust = threshold - 1 if threshold > 0 else 0
        for elem in list(combined.keys()):
            combined[elem] -= adjust
            if combined[elem] <= 0:
                del combined[elem]

    merged.counters = combined
    merged.total = sum(mg.total for mg in summaries)
    return merged
```

### Apache DataSketches: 프로덕션 구현

Apache DataSketches 라이브러리는 Misra-Gries 계열 알고리즘의 최적화된 Java/Python 구현을 제공합니다.

```python
# pip install apache-datasketches
# from datasketches import frequent_strings_sketch

# 실제 프로덕션에서는 다음과 같이 사용:
# sketch = frequent_strings_sketch(lg_max_map_size=10)  # 2^10 = 1024 슬롯
# for item in stream:
#     sketch.update(item)
# results = sketch.get_frequent_items(frequent_items_error_type.NO_FALSE_POSITIVES)
```

## 다른 스트리밍 알고리즘과의 비교

| 알고리즘 | 목적 | 메모리 | 오류 유형 |
|---------|------|-------|---------|
| HyperLogLog | 고유 원소 수 추정 | O(1/ε²) | 확률적, 상대 오차 |
| Count-Min Sketch | 임의 원소 빈도 추정 | O((1/ε) log(1/δ)) | 확률적, 과대추정 |
| **Misra-Gries** | **Heavy Hitters 탐지** | **O(1/φ)** | **결정론적, 과소추정** |
| **Space-Saving** | **Top-K 빈번 원소** | **O(K)** | **결정론적, 과대추정** |
| Bloom Filter | 집합 멤버십 테스트 | O(n log(1/ε)) | 확률적, 거짓 양성 |

**핵심 차이:** Misra-Gries와 Space-Saving은 **결정론적(deterministic)** 입니다. 같은 스트림에 대해 항상 같은 결과를 보장합니다. Count-Min Sketch는 더 일반적인 빈도 추정에 사용되지만 무작위성을 활용합니다.

## 주의사항과 팁

**1. k 값 선택**
φ-Heavy Hitters를 찾으려면 k ≥ 1/φ + 1 이어야 합니다. Top-K를 원한다면 k를 충분히 크게 (K보다 4-10배 크게) 설정해야 정확도가 올라갑니다.

**2. Zipf 분포와 실제 데이터**
대부분의 실제 데이터는 Zipf 분포를 따릅니다 (상위 20%가 80%의 빈도). 이 경우 두 알고리즘 모두 실제보다 높은 정밀도/재현율을 보입니다.

**3. 시간 윈도우 처리**
슬라이딩 윈도우 Heavy Hitters가 필요하다면 Window Misra-Gries나 WCSS(Window-based Space-Saving) 등의 변형을 검토하세요.

**4. Space-Saving의 과대추정 위험**
Space-Saving은 교체 시 카운터를 높게 시작하므로, 실제 빈도를 과대추정할 수 있습니다. error 값과 함께 구간으로 해석하세요.

**5. 처리 속도**
Misra-Gries의 케이스 3(취소)은 최악 O(k)로 느립니다. 실제 구현에서는 Python dict 대신 LRU 형태의 자료구조나 링크드 버킷을 사용하여 O(1) amortized로 개선합니다.

## 정리

Misra-Gries와 Space-Saving은 1980-2000년대에 제안된 알고리즘이지만, 빅데이터·스트리밍 시대가 되면서 그 가치가 재조명되었습니다. 결정론적 보장과 O(1/φ) 메모리라는 특성은 네트워크 트래픽 분석, 실시간 트렌드 탐지, 분산 시스템 모니터링 등 현대 인프라의 핵심 요소가 되었습니다. HyperLogLog, Count-Min Sketch와 함께 "스트리밍 알고리즘 삼총사"로 기억해두면 좋습니다.

## 참고 자료
- [Finding Repeated Elements (Misra & Gries, 1982)](https://www.sciencedirect.com/science/article/pii/0167642382900148)
- [Efficient Computation of Frequent and Top-k Elements in Data Streams (Metwally et al., 2005)](https://link.springer.com/chapter/10.1007/978-3-540-30570-5_15)
- [A High-Performance Algorithm for Identifying Frequent Items in Data Streams (arXiv:1705.07001)](https://arxiv.org/abs/1705.07001)
- [Misra-Gries Heavy Hitters Algorithm - Wikipedia](https://en.wikipedia.org/wiki/Misra%E2%80%93Gries_heavy_hitters_algorithm)
