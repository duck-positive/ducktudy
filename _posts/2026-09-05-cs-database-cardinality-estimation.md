---
layout: post
title: "데이터베이스 카디널리티 추정 완전 정복: 쿼리 옵티마이저가 틀리는 이유와 해결책"
date: 2026-09-05
categories: [cs, computer-science]
tags: [database, cardinality-estimation, query-optimizer, histogram, statistics, postgresql, selectivity, execution-plan]
---

데이터베이스에서 `EXPLAIN` 명령을 실행해 본 적이 있다면 실제 행 수와 옵티마이저가 예측한 행 수(rows)가 수십~수천 배 차이 나는 상황을 목격했을 것이다. 이 수치의 정확도가 **쿼리 성능을 결정**한다. 쿼리 옵티마이저는 수백만 가지 가능한 실행 계획 중 하나를 선택하는데, 그 기준이 바로 **카디널리티 추정(Cardinality Estimation)**이다. 카디널리티를 잘못 추정하면 최악의 실행 계획이 선택되어 수초 쿼리가 수분으로 늘어난다.

## 카디널리티란 무엇인가

**카디널리티(Cardinality)**는 집합의 원소 수, 즉 데이터베이스 맥락에서는 **쿼리 연산자가 반환하는 행 수의 추정치**다. 카디널리티 추정은 다음 세 수준에서 이루어진다:

1. **기저 테이블 카디널리티**: `users` 테이블의 총 행 수 — 통계 테이블에서 직접 읽는다.
2. **선택도(Selectivity)**: 조건절 `WHERE age > 30`이 전체 행 중 몇 퍼센트를 통과시키는가.
3. **조인 카디널리티**: `users JOIN orders ON users.id = orders.user_id` 의 결과 행 수.

이 세 추정치를 결합하여 실행 계획 트리의 각 노드에서의 예상 행 수를 계산한다. 예상 행 수가 **실행 계획 선택의 핵심 입력**이다:

- 예상 행 수 < 수천 행 → 해시 조인 또는 네스티드 루프 조인 선호
- 예상 행 수 > 수만 행 → 머지 조인 또는 병렬 스캔 선호
- 예상 행 수가 너무 적게 추정됨 → 해시 테이블을 너무 작게 잡아 메모리 초과 발생

## 옵티마이저가 수집하는 통계

PostgreSQL을 기준으로 설명한다. `ANALYZE` 명령이 실행되면 `pg_statistic` 시스템 카탈로그에 다음 통계가 저장된다:

| 통계 항목 | 설명 |
|---|---|
| `null_frac` | NULL 비율 |
| `n_distinct` | 고유 값 수 (양수: 정확한 수, 음수: 전체 행 수 대비 비율) |
| `most_common_vals` | 가장 빈번한 값 목록 (기본 100개) |
| `most_common_freqs` | 각 빈번값의 출현 빈도 |
| `histogram_bounds` | 등깊이 히스토그램(equi-depth histogram)의 경계값 |
| `correlation` | 물리적 순서와 논리적 순서의 상관계수 |

```sql
-- 특정 컬럼의 통계 확인
SELECT
    attname,
    null_frac,
    n_distinct,
    most_common_vals,
    most_common_freqs,
    histogram_bounds
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'status';
```

## 히스토그램 기반 선택도 추정

가장 핵심적인 통계 구조는 **등깊이 히스토그램(Equi-Depth Histogram)**이다. 이 히스토그램은 각 버킷이 **동일한 수의 행**을 포함하도록 버킷 경계를 설정한다. PostgreSQL 기본 버킷 수는 100개다.

### 구현: Python으로 등깊이 히스토그램 만들기

```python
import statistics
from typing import List, Tuple, Optional
import bisect

class EquiDepthHistogram:
    """등깊이 히스토그램 — 각 버킷에 동일한 수의 행이 들어감."""

    def __init__(self, data: List[float], num_buckets: int = 100):
        if not data:
            raise ValueError("데이터가 비어 있습니다.")
        sorted_data = sorted(data)
        n = len(sorted_data)
        self.num_buckets = num_buckets
        self.total_rows = n

        # 버킷 경계 계산 (quantiles)
        # histogram_bounds: 버킷 사이의 경계값들 (num_buckets+1개)
        self.bounds = []
        for i in range(num_buckets + 1):
            idx = min(int(i * n / num_buckets), n - 1)
            self.bounds.append(sorted_data[idx])

        # 최빈값(MCVs) 추출 — 상위 25개
        from collections import Counter
        counter = Counter(sorted_data)
        mcv_cutoff = 25
        most_common = counter.most_common(mcv_cutoff)
        self.mcv_vals = [v for v, _ in most_common]
        self.mcv_freqs = [cnt / n for _, cnt in most_common]
        self.mcv_set = set(self.mcv_vals)

    def selectivity_equal(self, value: float) -> float:
        """= 조건의 선택도 추정."""
        # MCV에 있는 경우 정확한 빈도 반환
        if value in self.mcv_set:
            idx = self.mcv_vals.index(value)
            return self.mcv_freqs[idx]

        # 히스토그램 기반 추정: 해당 버킷 내 균등 분포 가정
        mcv_total_freq = sum(self.mcv_freqs)
        remaining_fraction = 1.0 - mcv_total_freq
        n_distinct_in_hist = self.num_buckets  # 버킷당 1개 고유값 가정
        return remaining_fraction / n_distinct_in_hist if n_distinct_in_hist > 0 else 0

    def selectivity_range(self, lo: Optional[float], hi: Optional[float],
                           lo_inclusive: bool = True, hi_inclusive: bool = False) -> float:
        """범위 조건 [lo, hi)의 선택도 추정."""
        def bucket_fraction(val: float, upper: bool) -> float:
            """val이 전체 히스토그램에서 차지하는 위치(0~1)를 반환."""
            if val <= self.bounds[0]:
                return 0.0
            if val >= self.bounds[-1]:
                return 1.0
            # 어떤 버킷에 속하는지 찾기
            idx = bisect.bisect_right(self.bounds, val) - 1
            idx = max(0, min(idx, self.num_buckets - 1))
            bucket_lo = self.bounds[idx]
            bucket_hi = self.bounds[idx + 1]
            # 버킷 내 선형 보간
            if bucket_hi == bucket_lo:
                frac_in_bucket = 1.0 if upper else 0.0
            else:
                frac_in_bucket = (val - bucket_lo) / (bucket_hi - bucket_lo)
            return (idx + frac_in_bucket) / self.num_buckets

        lo_frac = bucket_fraction(lo, False) if lo is not None else 0.0
        hi_frac = bucket_fraction(hi, True)  if hi is not None else 1.0

        # MCV 제외 후의 히스토그램 커버리지에서 비율 계산
        mcv_total_freq = sum(self.mcv_freqs)
        hist_fraction = max(0.0, hi_frac - lo_frac) * (1.0 - mcv_total_freq)

        # MCV 중 범위에 포함되는 것들 합산
        mcv_contribution = sum(
            freq for val, freq in zip(self.mcv_vals, self.mcv_freqs)
            if (lo is None or (val >= lo if lo_inclusive else val > lo)) and
               (hi is None or (val <= hi if hi_inclusive else val < hi))
        )

        return min(1.0, hist_fraction + mcv_contribution)


# 예시: 주문 금액 데이터 시뮬레이션
import random
random.seed(42)

# 대부분은 10~500달러, 일부는 소액(0~10), 일부는 고액(500~10000)
data = (
    [random.uniform(10, 500) for _ in range(9000)] +
    [random.uniform(0, 10) for _ in range(500)] +
    [random.uniform(500, 10000) for _ in range(500)]
)

hist = EquiDepthHistogram(data, num_buckets=100)

# 선택도 추정
sel_low  = hist.selectivity_range(0, 50)
sel_mid  = hist.selectivity_range(50, 200)
sel_high = hist.selectivity_range(200, None)

print(f"amount < 50     예상 선택도: {sel_low:.3f}  ({sel_low*10000:.0f}행 예상 / 10,000행)")
print(f"50 ≤ amount < 200 예상 선택도: {sel_mid:.3f}  ({sel_mid*10000:.0f}행 예상)")
print(f"amount ≥ 200    예상 선택도: {sel_high:.3f}  ({sel_high*10000:.0f}행 예상)")
```

## 조인 카디널리티 추정

조인 카디널리티 추정은 단순 범위 추정보다 훨씬 어렵다. PostgreSQL이 기본으로 사용하는 **독립성 가정(Independence Assumption)**은 두 칼럼이 서로 독립적으로 분포한다고 가정한다:

```
Card(R ⋈ S on R.a = S.b) ≈ Card(R) × Card(S) / max(NDV(R.a), NDV(S.b))
```

여기서 NDV는 Distinct Value 수다. 이 공식은 **포함 가정(Containment Assumption)**(한 쪽의 값이 반드시 다른 쪽에 포함됨)도 내포한다.

### 독립성 가정이 틀리는 경우들

```sql
-- 문제 1: 상관된 조건 (correlated predicates)
-- country = 'KR' AND city = 'Seoul'
-- 두 조건이 독립적이지 않음 — 서울은 한국에만 있으므로 실제 선택도 훨씬 높음
SELECT * FROM users WHERE country = 'KR' AND city = 'Seoul';

-- 문제 2: 데이터 스큐 (data skew)
-- 일부 user_id가 압도적으로 많은 주문을 가짐 (power-law distribution)
SELECT u.name, COUNT(*) FROM users u JOIN orders o ON u.id = o.user_id
GROUP BY u.name;

-- 문제 3: 조인 후 필터 (filter after join)
-- 옵티마이저는 조인 결과에 WHERE 적용 순서를 잘못 추정할 수 있음
SELECT * FROM orders o JOIN products p ON o.product_id = p.id
WHERE p.category = 'electronics' AND o.amount > 1000;
```

### 다차원 히스토그램

독립성 가정의 한계를 극복하기 위해 **다차원 히스토그램** 또는 **칼럼 그룹 통계**를 사용한다. PostgreSQL 12부터 `CREATE STATISTICS`로 명시적 다변량 통계를 생성할 수 있다:

```sql
-- (country, city) 쌍의 상관 통계 수집
CREATE STATISTICS users_country_city_stats (dependencies, ndistinct)
ON country, city FROM users;

ANALYZE users;

-- 통계 확인
SELECT * FROM pg_statistic_ext;
SELECT * FROM pg_statistic_ext_data;
```

## 샘플링 기반 추정

히스토그램은 사전 구축된 통계에 의존하지만, **쿼리 실행 시점에 실제 데이터를 샘플링**하여 추정하는 방식도 있다. SQL Server의 "Adaptive Query Processing"과 DuckDB의 일부 최적화에서 사용한다.

```python
import random
from typing import Callable, List

def sample_selectivity(
    table_size: int,
    condition: Callable[[dict], bool],
    data_generator: Callable[[int], dict],
    sample_rate: float = 0.01,
    confidence: float = 0.95,
) -> Tuple[float, float]:
    """
    샘플링으로 조건의 선택도를 추정하고 신뢰구간을 반환한다.
    Returns: (selectivity_estimate, margin_of_error)
    """
    import math
    sample_size = max(int(table_size * sample_rate), 1000)
    sample_size = min(sample_size, table_size)

    matches = sum(
        1 for _ in range(sample_size)
        if condition(data_generator(random.randint(0, table_size - 1)))
    )
    p_hat = matches / sample_size

    # Wilson score interval (이항 비율의 신뢰구간)
    z = 1.96  # 95% 신뢰수준
    margin = z * math.sqrt(p_hat * (1 - p_hat) / sample_size)

    return p_hat, margin


# 예시
orders = [{"amount": random.uniform(1, 10000), "status": random.choice(["paid", "pending", "cancelled"])}
          for _ in range(100_000)]

def condition(row):
    return row["amount"] > 500 and row["status"] == "paid"

sel, margin = sample_selectivity(
    table_size=100_000,
    condition=condition,
    data_generator=lambda i: orders[i],
    sample_rate=0.02,
)
print(f"추정 선택도: {sel:.4f} ± {margin:.4f}")
```

## 머신 러닝 기반 카디널리티 추정

최근 연구에서는 전통적인 히스토그램 대신 **학습 기반 카디널리티 추정(Learned Cardinality Estimation)**이 등장했다. 대표적인 접근법:

1. **Naru / NeuroCard** (MIT, 2019~): 데이터 분포를 신경망으로 모델링하여 임의의 다중 칼럼 조건부 선택도를 추정.
2. **BayesCard**: 베이즈 네트워크로 칼럼 간 상관관계를 포착.
3. **MSCN (Multi-Set Convolutional Network)**: 쿼리 구조와 통계를 함께 학습.

```python
# 개념적 구현: 선형 회귀 기반 단순 카디널리티 예측기
import numpy as np
from sklearn.linear_model import Ridge
from sklearn.preprocessing import StandardScaler

class SimpleCardinalityPredictor:
    """
    실제 실행 결과를 피드백으로 학습하는 단순 카디널리티 예측기.
    특성: [lo_bound, hi_bound, n_distinct, null_frac, mcv_coverage, table_rows]
    레이블: 실제 반환 행 수 (로그 스케일)
    """

    def __init__(self):
        self.model = Ridge(alpha=1.0)
        self.scaler = StandardScaler()
        self.is_fitted = False

    def _build_features(self, lo, hi, n_distinct, null_frac, mcv_coverage, table_rows):
        table_rows = max(table_rows, 1)
        range_frac = (hi - lo) / max(hi, 1) if hi and lo else 1.0
        return np.array([
            range_frac,
            np.log1p(n_distinct),
            null_frac,
            mcv_coverage,
            np.log1p(table_rows),
            range_frac * np.log1p(n_distinct),  # 교호 작용
        ])

    def fit(self, queries: list, actuals: list):
        """이전 쿼리 실행 결과로 모델 학습."""
        X = np.array([self._build_features(**q) for q in queries])
        y = np.log1p(np.array(actuals))  # 로그 스케일
        self.scaler.fit(X)
        self.model.fit(self.scaler.transform(X), y)
        self.is_fitted = True

    def predict(self, **features) -> int:
        if not self.is_fitted:
            raise RuntimeError("모델이 학습되지 않았습니다.")
        X = self._build_features(**features).reshape(1, -1)
        log_pred = self.model.predict(self.scaler.transform(X))[0]
        return max(1, int(np.expm1(log_pred)))
```

## 카디널리티 오추정 디버깅

### PostgreSQL에서 추정 오차 확인

```sql
-- 실제 행 수 vs 예측 행 수 비교
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
SELECT * FROM orders
WHERE user_id = 12345 AND created_at > '2026-01-01';

-- 출력 예시 (JSON 중 핵심 부분):
-- "Plan Rows": 150,    ← 옵티마이저 예측
-- "Actual Rows": 48250 ← 실제 행 수
-- 오차율: 321배 → 매우 나쁜 추정

-- 통계 갱신 (default_statistics_target 올리기)
ALTER TABLE orders ALTER COLUMN user_id SET STATISTICS 500;
ANALYZE orders;

-- 또는 세션 수준에서 통계 목표 설정
SET default_statistics_target = 500;
ANALYZE orders;
```

### 일반적인 원인과 해결책

| 증상 | 원인 | 해결책 |
|---|---|---|
| rows 과소 추정 | 데이터 스큐 (특정 값 집중) | MCV 수 증가, 선택적 통계 |
| rows 과대 추정 | 칼럼 상관관계 무시 | `CREATE STATISTICS (dependencies)` |
| 통계 자체가 오래됨 | VACUUM/ANALYZE 주기 문제 | autovacuum 튜닝, 수동 ANALYZE |
| 복잡한 표현식 조건 | 함수 적용 후 선택도 불명 | 표현식 인덱스 + 통계 |
| 조인 순서 문제 | 중간 결과 카디널리티 오추정 | `join_collapse_limit` 조절, 힌트 |

## 주의사항과 팁

**ANALYZE 주기를 튜닝하라.** PostgreSQL의 autovacuum은 기본적으로 행의 20%가 변경되면 ANALYZE를 트리거한다. 대용량 테이블에서는 이 임계값이 너무 늦게 걸릴 수 있다.

```sql
-- 대용량 테이블의 ANALYZE 임계값 낮추기
ALTER TABLE large_table SET (
    autovacuum_analyze_scale_factor = 0.01,  -- 1% 변경 시
    autovacuum_analyze_threshold = 1000       -- 최소 1000행
);
```

**extended statistics로 상관관계를 포착하라.** 두 칼럼이 항상 함께 필터링된다면 반드시 `CREATE STATISTICS ... (dependencies, ndistinct, mcv)`를 생성하라. 조건이 복잡할수록 효과가 크다.

**파티션 테이블에서는 각 파티션별 통계가 중요하다.** 전체 테이블 통계만으로는 개별 파티션의 분포를 포착하지 못한다.

카디널리티 추정은 데이터베이스 성능의 숨은 핵심이다. 쿼리가 느릴 때 인덱스만 보는 것은 절반의 진단이다. `EXPLAIN ANALYZE`로 예상 행 수와 실제 행 수의 차이를 먼저 확인하고, 그 차이가 크다면 통계와 카디널리티 추정을 의심해야 한다.

## 참고 자료
- [Row Estimation Examples — PostgreSQL 공식 문서](https://www.postgresql.org/docs/current/row-estimation-examples.html)
- [Cardinality Estimation in DBMS: A Comprehensive Benchmark Evaluation (VLDB 2022)](https://www.vldb.org/pvldb/vol15/p752-zhu.pdf)
- [Analyzing Query Optimizer Performance (arXiv 2023)](https://arxiv.org/abs/2311.17293)
- [pg_stats System View — PostgreSQL 공식 문서](https://www.postgresql.org/docs/current/view-pg-stats.html)
