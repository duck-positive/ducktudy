---
layout: post
title: "학습 인덱스(Learned Index Structures): 머신러닝으로 구축하는 차세대 데이터베이스 인덱스"
date: 2026-10-28
categories: [cs, computer-science]
tags: [learned-index, database, machine-learning, B-tree, RMI, PGM-index, data-structure]
---

데이터베이스의 핵심은 인덱스다. 수십억 건의 레코드에서 특정 키를 찾을 때 풀스캔(O(n)) 대신 B-트리(O(log n))가 동작할 수 있는 것은 바로 인덱스 덕분이다. 그런데 2018년 구글 딥마인드의 Tim Kraska 팀이 SIGMOD에서 충격적인 논문을 발표했다. "인덱스 자체를 머신러닝 모델로 대체할 수 있다"는 것이다. 이것이 **학습 인덱스(Learned Index Structures)**의 시작이다.

## 인덱스란 무엇인가 — 새로운 관점

전통적인 B-트리 인덱스의 역할을 수식으로 표현하면 다음과 같다:

```
f(key) → position  (디스크 상의 레코드 위치)
```

B-트리는 이 함수를 노드 트리로 구현한다. 탐색마다 노드를 내려가며 비교 연산을 수행하고, 최종적으로 데이터가 저장된 물리적 주소를 반환한다.

Kraska 팀의 핵심 통찰은 이렇다: **"정렬된 데이터의 키 분포는 누적 분포 함수(CDF)와 같다."** 키가 정렬된 배열에서 키 `k`의 위치는 `k`보다 작은 키의 개수, 즉 CDF에 비례한다. 따라서 다음을 근사할 수 있다:

```
f(key) ≈ CDF(key) × N   (N은 전체 레코드 수)
```

이 CDF를 뉴럴 네트워크나 선형 회귀 같은 머신러닝 모델로 학습시킨다면, 트리 탐색 없이 한 번의 모델 추론으로 위치를 예측할 수 있다.

## 왜 필요한가 — B-트리의 한계

B-트리는 훌륭한 자료구조지만 몇 가지 비효율이 있다:

**1. 범용성으로 인한 오버헤드:** B-트리는 어떤 데이터 분포에서도 동작해야 하므로, 실제 데이터의 분포를 전혀 활용하지 못한다. 단조 증가하는 타임스탬프 키가 있어도 B-트리는 이를 모른다.

**2. 메모리 비효율:** B-트리 노드는 포인터, 메타데이터, 페이지 헤더를 포함한다. 수억 건의 레코드에서 인덱스만 수 GB를 차지하는 일이 흔하다.

**3. 캐시 미스:** 트리 순회는 다수의 포인터 체이싱을 수반하며, CPU 캐시를 비효율적으로 사용한다.

학습 인덱스는 데이터의 통계적 특성을 학습하여 이러한 한계를 극복하려 한다.

## RMI: Recursive Model Index 구현

Kraska 논문의 핵심 기여는 **RMI(Recursive Model Index)**다. 단일 모델로 전체 CDF를 정확히 근사하기 어렵기 때문에, 계층적 모델 구조를 사용한다.

```python
import numpy as np
from dataclasses import dataclass
from typing import List, Optional

@dataclass
class LinearModel:
    """단순 선형 회귀 모델 (y = ax + b)"""
    slope: float = 0.0
    intercept: float = 0.0

    def predict(self, key: float) -> float:
        return self.slope * key + self.intercept

    def train(self, keys: np.ndarray, positions: np.ndarray):
        if len(keys) < 2:
            self.slope = 0
            self.intercept = positions[0] if len(positions) > 0 else 0
            return
        # 최소제곱법으로 선형 회귀
        A = np.vstack([keys, np.ones(len(keys))]).T
        result = np.linalg.lstsq(A, positions, rcond=None)
        self.slope, self.intercept = result[0]


class RMI:
    """
    2단계 Recursive Model Index 구현
    
    Stage 1: 전체 키 공간을 커버하는 단일 모델
    Stage 2: Stage 1의 예측값을 기반으로 선택되는 전문가 모델 집합
    """

    def __init__(self, n_models: int = 100):
        self.n_models = n_models
        self.stage1 = LinearModel()
        self.stage2: List[LinearModel] = [LinearModel() for _ in range(n_models)]
        self.min_errors: List[int] = []
        self.max_errors: List[int] = []

    def build(self, keys: np.ndarray):
        """정렬된 키 배열로부터 RMI 구축"""
        n = len(keys)
        positions = np.arange(n, dtype=float)

        # Stage 1 학습: 키 → 대략적 위치 예측
        self.stage1.train(keys, positions / n)  # [0, 1] 범위로 정규화

        # 각 Stage-2 모델에 키 할당
        model_assignments = [[] for _ in range(self.n_models)]
        pos_assignments = [[] for _ in range(self.n_models)]

        for i, key in enumerate(keys):
            # Stage 1 예측으로 어떤 Stage-2 모델을 사용할지 결정
            pred = self.stage1.predict(key)
            model_idx = int(np.clip(pred * self.n_models, 0, self.n_models - 1))
            model_assignments[model_idx].append(key)
            pos_assignments[model_idx].append(i)

        # Stage 2 모델 학습 및 오류 범위 계산
        self.min_errors = []
        self.max_errors = []

        for i in range(self.n_models):
            if model_assignments[i]:
                k = np.array(model_assignments[i], dtype=float)
                p = np.array(pos_assignments[i], dtype=float)
                self.stage2[i].train(k, p)

                # 각 모델의 최대/최소 오류 기록 (바이너리 서치 범위)
                preds = np.array([self.stage2[i].predict(key) for key in k])
                errors = p - preds
                self.min_errors.append(int(np.floor(errors.min())))
                self.max_errors.append(int(np.ceil(errors.max())))
            else:
                self.min_errors.append(0)
                self.max_errors.append(0)

        self.keys = keys

    def lookup(self, key: float) -> Optional[int]:
        """키의 위치를 예측하고 로컬 바이너리 서치로 정확한 위치 반환"""
        # Stage 1: 대략적 모델 인덱스 예측
        pred1 = self.stage1.predict(key)
        model_idx = int(np.clip(pred1 * self.n_models, 0, self.n_models - 1))

        # Stage 2: 정확한 위치 예측
        pred2 = int(self.stage2[model_idx].predict(key))

        # 오류 범위 내에서 바이너리 서치
        lo = max(0, pred2 + self.min_errors[model_idx])
        hi = min(len(self.keys) - 1, pred2 + self.max_errors[model_idx])

        # 바이너리 서치로 정확한 위치 찾기
        while lo <= hi:
            mid = (lo + hi) // 2
            if self.keys[mid] == key:
                return mid
            elif self.keys[mid] < key:
                lo = mid + 1
            else:
                hi = mid - 1

        return None  # 키 없음


# 사용 예시
if __name__ == "__main__":
    import random
    import time

    N = 1_000_000
    keys = np.sort(np.random.randint(0, 10 * N, N).astype(float))

    rmi = RMI(n_models=1000)

    start = time.perf_counter()
    rmi.build(keys)
    build_time = time.perf_counter() - start
    print(f"RMI 구축 시간: {build_time:.3f}s")

    # 조회 성능 측정
    test_keys = random.choices(list(keys), k=1000)
    start = time.perf_counter()
    for k in test_keys:
        rmi.lookup(k)
    rmi_time = time.perf_counter() - start

    # B-트리(이진 탐색)와 비교
    start = time.perf_counter()
    for k in test_keys:
        np.searchsorted(keys, k)
    bsearch_time = time.perf_counter() - start

    print(f"RMI 조회 시간 (1000회): {rmi_time*1000:.2f}ms")
    print(f"이진 탐색 시간 (1000회): {bsearch_time*1000:.2f}ms")
    print(f"속도 향상: {bsearch_time/rmi_time:.2f}x")
```

## PGM-Index: 실용적인 학습 인덱스

RMI는 혁신적이지만 학습 과정이 복잡하고, 잘못 튜닝하면 성능이 저하된다. 2020년 Ferragina와 Vinciguerra가 발표한 **PGM-Index(Piecewise Geometric Model)**은 더 실용적인 접근을 제시한다.

```python
from bisect import bisect_left
from typing import List, Tuple

def build_pgm_index(keys: List[int], epsilon: int = 64) -> List[Tuple[int, float, float]]:
    """
    PGM-Index 구축
    
    각 선형 세그먼트는 (첫 번째 키, slope, intercept)로 표현된다.
    epsilon은 허용 오차: 예측 위치 ± epsilon 범위 내에 실제 위치가 있음을 보장.
    
    반환: 세그먼트 목록 [(key, slope, intercept), ...]
    """
    n = len(keys)
    if n == 0:
        return []

    segments = []
    seg_start = 0
    slope_lo = float('-inf')  # 허용 기울기 하한
    slope_hi = float('inf')   # 허용 기울기 상한

    for i in range(1, n):
        # 현재 점 (keys[i], i)이 오차 범위 내에 있는지 확인
        # 오차 범위: epsilon 이내로 예측 위치가 실제 위치와 달라도 됨
        dx = keys[i] - keys[seg_start]
        if dx == 0:
            continue

        # 허용 기울기 범위 계산
        new_slope_lo = (i - seg_start - epsilon) / dx
        new_slope_hi = (i - seg_start + epsilon) / dx

        # 허용 범위 교집합
        slope_lo = max(slope_lo, new_slope_lo)
        slope_hi = min(slope_hi, new_slope_hi)

        if slope_lo > slope_hi:
            # 더 이상 단일 선형 모델로 커버 불가 → 새 세그먼트 시작
            # 이전 세그먼트 확정
            mid_slope = (slope_lo + slope_hi) / 2  # 이전 교집합의 중간값
            # 실제로는 이전 i-1 시점의 기울기를 사용
            best_slope = (slope_lo + slope_hi) / 2
            intercept = seg_start - best_slope * keys[seg_start]
            segments.append((keys[seg_start], best_slope, intercept))

            # 새 세그먼트 초기화
            seg_start = i - 1
            slope_lo = float('-inf')
            slope_hi = float('inf')

    # 마지막 세그먼트 추가
    if slope_lo == float('-inf') or slope_hi == float('inf'):
        best_slope = (n - seg_start - 1) / max(keys[-1] - keys[seg_start], 1)
    else:
        best_slope = (slope_lo + slope_hi) / 2

    intercept = seg_start - best_slope * keys[seg_start]
    segments.append((keys[seg_start], best_slope, intercept))

    return segments


def pgm_lookup(keys: List[int], segments: List[Tuple[int, float, float]],
               target: int, epsilon: int = 64) -> int:
    """
    PGM-Index로 target 위치 예측 후 바이너리 서치로 정확한 위치 반환
    """
    # 어떤 세그먼트를 사용할지 이진 탐색 (세그먼트 키는 정렬되어 있음)
    seg_keys = [s[0] for s in segments]
    seg_idx = bisect_left(seg_keys, target)
    if seg_idx > 0 and (seg_idx == len(segments) or seg_keys[seg_idx] > target):
        seg_idx -= 1

    # 해당 세그먼트로 위치 예측
    key0, slope, intercept = segments[seg_idx]
    pred = int(slope * target + intercept)

    # epsilon 범위 내에서 이진 탐색
    lo = max(0, pred - epsilon)
    hi = min(len(keys) - 1, pred + epsilon)

    # 바이너리 서치
    idx = bisect_left(keys, target, lo, hi + 1)
    if idx < len(keys) and keys[idx] == target:
        return idx
    return -1  # 없는 키


# 사용 예시
if __name__ == "__main__":
    import random

    data = sorted(random.sample(range(10_000_000), 1_000_000))
    EPS = 64

    segments = build_pgm_index(data, epsilon=EPS)
    print(f"PGM 세그먼트 수: {len(segments)}")
    print(f"세그먼트당 평균 커버 키 수: {len(data) / len(segments):.0f}")
    print(f"메모리 사용 (세그먼트): {len(segments) * 3 * 8} bytes (slope+intercept+key)")

    # 검증
    test_keys = random.sample(data, 100)
    success = sum(1 for k in test_keys if pgm_lookup(data, segments, k, EPS) != -1)
    print(f"조회 성공률: {success}/100")
```

## 주의사항과 현실적 고려

**1. 갱신 문제가 치명적이다.** 학습 인덱스는 기본적으로 정적이다. 새로운 키가 삽입되면 모델의 CDF 가정이 깨진다. Updatable Learned Index(ALEX, DILI) 등이 이를 해결하려 하지만, 아직 B-트리의 갱신 유연성에 미치지 못한다.

**2. 쓰기가 많은 워크로드에는 부적합하다.** 읽기가 압도적으로 많고(read-heavy), 데이터 분포가 비교적 안정적인 시나리오(로그 데이터, 시계열 데이터, 키-값 스토어)에서 가장 유효하다.

**3. 최악의 경우 성능 보장이 없다.** B-트리는 O(log n)을 보장하지만, 학습 인덱스는 분포를 잘못 학습하면 예측이 크게 벗어날 수 있다. 로컬 바이너리 서치 범위(epsilon)를 크게 설정하면 보완할 수 있지만 성능 이점이 줄어든다.

**4. SOSD 벤치마크 결과를 주목하라.** 2020년 Marcus et al.이 공개한 SOSD(Suite of Searched Data structures) 벤치마크에서, 실제 데이터셋에서 학습 인덱스가 B-트리를 항상 압도하지는 않았다. 데이터 분포에 따라 성능 차이가 크게 난다.

**5. 하이브리드 접근이 현실적이다.** PostgreSQL의 인덱스 어드바이저, MySQL의 쿼리 최적화기처럼, 학습 인덱스를 기존 B-트리의 완전한 대체제보다는 특수 목적 인덱스 또는 쿼리 최적화 힌트로 활용하는 방향이 산업계에서 더 활발히 연구되고 있다.

학습 인덱스는 "데이터베이스 자료구조도 학습될 수 있다"는 패러다임 전환을 보여준 점에서 의미가 크다. B-트리가 1970년 등장한 이후 반세기 만에 나온 진지한 도전이다.

## 참고 자료
- [The Case for Learned Index Structures (Kraska et al., SIGMOD 2018)](https://arxiv.org/abs/2103.00858)
- [The Morning Paper: The case for learned index structures](https://blog.acolyer.org/2018/01/09/the-case-for-learned-index-structures-part-ii/)
- [SOSD: A Benchmark for Learned Indexes](https://learned.systems/papers/sosd.pdf)
- [PGM-Index: Official Repository](https://pgm.di.unipi.it/)
