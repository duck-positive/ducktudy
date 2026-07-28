---
layout: post
title: "벡터화 실행 엔진 완전 정복: Volcano 모델의 한계와 DuckDB가 선택한 Column-at-a-Time 처리"
date: 2026-07-28
categories: [cs, computer-science]
tags: [database, query-execution, vectorized, duckdb, olap, simd, volcano-model, columnar]
---

데이터베이스 쿼리 실행 방식에는 조용한 혁명이 일어나고 있습니다. 수십 년간 표준이었던 Volcano 모델을 대체하는 **벡터화 실행 엔진(Vectorized Execution Engine)**이 DuckDB, ClickHouse, Snowflake, Databricks Photon의 심장이 되었습니다. 이 글에서는 두 모델의 차이를 코드 수준에서 분석하고, 왜 벡터화가 분석 쿼리에서 4~30배 빠른지 그 원리를 해부합니다.

## 전통적인 Volcano 모델

1994년 Goetz Graefe의 논문 "Volcano — An Extensible and Parallel Query Evaluation System"에서 제안된 Volcano 모델(이터레이터 모델이라고도 불림)은 매우 우아한 설계입니다.

모든 쿼리 오퍼레이터(Scan, Filter, Join, Aggregate 등)는 동일한 인터페이스 `next()` 를 구현합니다. 상위 오퍼레이터가 `next()`를 호출하면, 하위 오퍼레이터의 `next()`를 호출하고, 이것이 연쇄적으로 트리 아래까지 전파되어 결국 스토리지에서 한 row를 읽어 반환합니다.

```
SELECT sum(price) FROM orders WHERE amount > 100;

실행 트리:
Aggregate (sum)
    └── Filter (amount > 100)
            └── Scan (orders)

실행 흐름:
1. Aggregate.next()가 Filter.next()를 호출
2. Filter.next()가 Scan.next()를 호출
3. Scan이 row 1을 반환 → Filter가 조건 검사 → 통과하면 Aggregate에 반환
4. 1~3을 모든 row가 소진될 때까지 반복
```

이 모델의 장점은 명확합니다. 구현이 단순하고, 파이프라인 방식이라 모든 데이터를 메모리에 올릴 필요가 없습니다.

그런데 문제가 있습니다.

## Volcano 모델의 병목: 함수 호출 오버헤드

10억 개의 row를 스캔하는 쿼리를 생각해봅시다. Volcano 모델에서는 `next()` 가 10억 번 호출됩니다. CPU에서 가상 함수 호출(virtual function call)은 분기 예측 실패, 명령어 캐시 미스, 함수 프롤로그/에필로그 비용 등으로 수십 사이클이 걸립니다.

**10억 × 수십 사이클 = 수백억 사이클 = 수십 초**

더 심각한 것은 SIMD 최적화가 거의 불가능하다는 점입니다. CPU의 SIMD 명령어(SSE, AVX 등)는 여러 데이터를 동시에 처리하지만, Volcano 모델에서는 한 번에 row 하나만 처리하므로 SIMD가 낄 자리가 없습니다.

## 벡터화 실행 엔진의 원리

2005년 MonetDB/X100 논문(Boncz, Zukowski, Nes)에서 제안된 벡터화 실행은 간단한 아이디어에서 출발합니다.

> **한 번에 row 하나가 아닌, 한 번에 행 N개(벡터)씩 처리하자.**

`next()` 대신 `next_batch()`를 호출하면 오퍼레이터는 1개가 아닌 1024~4096개의 row를 한 번에 반환합니다. DuckDB는 이 크기를 **2048개**로 고정합니다.

```
벡터화 실행 흐름:
1. Aggregate가 Filter에게 next_batch() 호출
2. Filter가 Scan에게 next_batch() 호출
3. Scan이 2048개의 amount 값 배열을 반환
4. Filter가 2048개 배열에 대해 amount > 100 조건을 **일괄** 검사
   → SIMD로 한 번에 8~32개씩 비교 가능
5. 통과한 row의 price 값 배열을 Aggregate에게 반환
6. Aggregate가 배열 전체를 한 번에 합산
```

함수 호출 횟수가 10억 번에서 `10억 / 2048 ≈ 50만 번`으로 줄어듭니다. 그리고 각 오퍼레이터 내부 루프는 컴파일러가 SIMD로 자동 벡터화할 수 있습니다.

## 실제 구현 예제

### 예제 1: Python으로 Volcano 모델 vs 벡터화 실행 성능 비교

```python
import time
import random
from typing import Iterator, List

# ─────────────────────────────────────────────────────────────────
# 1. Volcano 모델 구현 (Row-at-a-time)
# ─────────────────────────────────────────────────────────────────

class VolcanoScan:
    def __init__(self, data: List[dict]):
        self.data = data
        self.idx = 0

    def next(self):
        if self.idx >= len(self.data):
            return None
        row = self.data[self.idx]
        self.idx += 1
        return row


class VolcanoFilter:
    def __init__(self, child, col: str, threshold: float):
        self.child = child
        self.col = col
        self.threshold = threshold

    def next(self):
        while True:
            row = self.child.next()
            if row is None:
                return None
            if row[self.col] > self.threshold:
                return row


class VolcanoAggregate:
    def __init__(self, child, col: str):
        self.child = child
        self.col = col

    def execute(self) -> float:
        total = 0.0
        while True:
            row = self.child.next()
            if row is None:
                break
            total += row[self.col]
        return total


# ─────────────────────────────────────────────────────────────────
# 2. 벡터화 실행 구현 (Batch 처리)
# ─────────────────────────────────────────────────────────────────

VECTOR_SIZE = 2048

class VectorizedScan:
    def __init__(self, amounts: List[float], prices: List[float]):
        self.amounts = amounts
        self.prices = prices
        self.idx = 0

    def next_batch(self):
        if self.idx >= len(self.amounts):
            return None, None
        end = min(self.idx + VECTOR_SIZE, len(self.amounts))
        a_batch = self.amounts[self.idx:end]
        p_batch = self.prices[self.idx:end]
        self.idx = end
        return a_batch, p_batch


class VectorizedFilter:
    def __init__(self, child, threshold: float):
        self.child = child
        self.threshold = threshold

    def next_batch(self):
        while True:
            amounts, prices = self.child.next_batch()
            if amounts is None:
                return None, None
            # 선택 벡터(selection vector): 조건을 만족하는 인덱스
            sel = [i for i, a in enumerate(amounts) if a > self.threshold]
            if sel:
                return (
                    [amounts[i] for i in sel],
                    [prices[i] for i in sel]
                )


class VectorizedAggregate:
    def __init__(self, child):
        self.child = child

    def execute(self) -> float:
        total = 0.0
        while True:
            _, prices = self.child.next_batch()
            if prices is None:
                break
            total += sum(prices)  # 컴파일러가 SIMD로 최적화 가능
        return total


# ─────────────────────────────────────────────────────────────────
# 3. 성능 비교
# ─────────────────────────────────────────────────────────────────

def benchmark():
    N = 5_000_000
    random.seed(42)
    amounts = [random.uniform(0, 200) for _ in range(N)]
    prices  = [random.uniform(1, 1000) for _ in range(N)]
    rows = [{"amount": a, "price": p} for a, p in zip(amounts, prices)]

    # Volcano 모델
    t0 = time.perf_counter()
    scan   = VolcanoScan(rows)
    filt   = VolcanoFilter(scan, "amount", 100)
    aggr   = VolcanoAggregate(filt, "price")
    result_v = aggr.execute()
    t1 = time.perf_counter()
    volcano_time = t1 - t0

    # 벡터화 실행
    t0 = time.perf_counter()
    vscan  = VectorizedScan(amounts, prices)
    vfilt  = VectorizedFilter(vscan, 100)
    vaggr  = VectorizedAggregate(vfilt)
    result_vec = vaggr.execute()
    t1 = time.perf_counter()
    vec_time = t1 - t0

    print(f"데이터 행 수: {N:,}")
    print(f"Volcano 모델:     {volcano_time:.3f}s  결과={result_v:.2f}")
    print(f"벡터화 실행:       {vec_time:.3f}s  결과={result_vec:.2f}")
    print(f"속도 향상: {volcano_time / vec_time:.1f}x")

    # 배치 처리 횟수 차이
    next_calls_volcano = N  # 모든 row마다 next() 호출
    next_calls_vec = (N + VECTOR_SIZE - 1) // VECTOR_SIZE
    print(f"\nnext() 호출 횟수 — Volcano: {next_calls_volcano:,}, "
          f"벡터화: {next_calls_vec:,} ({next_calls_volcano//next_calls_vec}x 감소)")


if __name__ == "__main__":
    benchmark()
```

Python에서도 벡터화 버전이 수 배 빠르게 측정됩니다. C나 Java에서 SIMD 최적화까지 더해지면 속도 차이는 더욱 극적으로 벌어집니다.

### 예제 2: C로 SIMD를 활용한 벡터화 필터 구현

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <immintrin.h>  // AVX2 인트린직

#define N 8000000
#define VECTOR_SIZE 256

// 스칼라 버전 (기존 방식)
long scalar_filter_sum(float *amounts, float *prices, int n, float threshold) {
    double total = 0.0;
    for (int i = 0; i < n; i++) {
        if (amounts[i] > threshold) {
            total += prices[i];
        }
    }
    return (long)total;
}

// 벡터화 버전 (AVX2 사용)
// AVX2: 256비트 레지스터로 8개 float 동시 처리
long avx2_filter_sum(float *amounts, float *prices, int n, float threshold) {
    __m256 thresh_vec = _mm256_set1_ps(threshold);
    __m256 sum_vec = _mm256_setzero_ps();

    int i = 0;
    // 8개씩 처리 (AVX2: 256bit / 32bit = 8개 float)
    for (; i <= n - 8; i += 8) {
        __m256 a = _mm256_loadu_ps(&amounts[i]);
        __m256 p = _mm256_loadu_ps(&prices[i]);

        // amounts > threshold 마스크 생성
        __m256 mask = _mm256_cmp_ps(a, thresh_vec, _CMP_GT_OS);

        // 마스크된 price만 누적
        __m256 masked_p = _mm256_and_ps(p, mask);
        sum_vec = _mm256_add_ps(sum_vec, masked_p);
    }

    // 256비트 레지스터를 수평 합산
    float result[8];
    _mm256_storeu_ps(result, sum_vec);
    double total = 0.0;
    for (int j = 0; j < 8; j++) total += result[j];

    // 나머지 처리 (8의 배수가 아닌 끝부분)
    for (; i < n; i++) {
        if (amounts[i] > threshold) total += prices[i];
    }
    return (long)total;
}

// 선택 벡터(Selection Vector) 기반 벡터화 — DuckDB 방식
// 조건을 만족하는 인덱스 배열을 먼저 생성한 뒤 gather 연산
int build_selection_vector(float *amounts, int n, float threshold, int *sel) {
    int count = 0;
    for (int i = 0; i < n; i++) {
        if (amounts[i] > threshold) {
            sel[count++] = i;
        }
    }
    return count;
}

double sum_with_selection(float *prices, int *sel, int count) {
    double total = 0.0;
    for (int i = 0; i < count; i++) {
        total += prices[sel[i]];
    }
    return total;
}

int main() {
    srand(42);
    float *amounts = (float*)malloc(N * sizeof(float));
    float *prices  = (float*)malloc(N * sizeof(float));
    int   *sel     = (int*)malloc(N * sizeof(int));

    for (int i = 0; i < N; i++) {
        amounts[i] = (float)(rand() % 200);
        prices[i]  = (float)(rand() % 1000 + 1);
    }

    float threshold = 100.0f;
    struct timespec t0, t1;

    // 스칼라 실행
    clock_gettime(CLOCK_MONOTONIC, &t0);
    long r1 = scalar_filter_sum(amounts, prices, N, threshold);
    clock_gettime(CLOCK_MONOTONIC, &t1);
    double scalar_ms = (t1.tv_sec - t0.tv_sec)*1000.0
                     + (t1.tv_nsec - t0.tv_nsec)/1e6;

    // AVX2 벡터화 실행
    clock_gettime(CLOCK_MONOTONIC, &t0);
    long r2 = avx2_filter_sum(amounts, prices, N, threshold);
    clock_gettime(CLOCK_MONOTONIC, &t1);
    double avx2_ms = (t1.tv_sec - t0.tv_sec)*1000.0
                   + (t1.tv_nsec - t0.tv_nsec)/1e6;

    // 선택 벡터 방식
    clock_gettime(CLOCK_MONOTONIC, &t0);
    int count = build_selection_vector(amounts, N, threshold, sel);
    double r3 = sum_with_selection(prices, sel, count);
    clock_gettime(CLOCK_MONOTONIC, &t1);
    double selvec_ms = (t1.tv_sec - t0.tv_sec)*1000.0
                     + (t1.tv_nsec - t0.tv_nsec)/1e6;

    printf("데이터 행 수: %d\n", N);
    printf("스칼라:        %.2fms  결과=%ld\n", scalar_ms, r1);
    printf("AVX2 벡터화:   %.2fms  결과=%ld  (%.1fx 향상)\n",
           avx2_ms, r2, scalar_ms/avx2_ms);
    printf("선택벡터:       %.2fms  결과=%.0f (%.1fx 향상)\n",
           selvec_ms, r3, scalar_ms/selvec_ms);

    // 컴파일: gcc -O2 -mavx2 -o vec_query vec_query.c
    free(amounts); free(prices); free(sel);
    return 0;
}
```

AVX2를 사용하면 8개의 float를 동시에 비교·합산하므로 이론적으로 8배의 SIMD 가속이 가능합니다. 실제로는 메모리 대역폭이 병목이 되어 3~6배 향상이 일반적입니다.

## DuckDB의 내부 구조 심화

DuckDB는 벡터화 실행을 가장 정교하게 구현한 오픈소스 OLAP 데이터베이스입니다. 주목할 특징들을 살펴봅니다.

### DataChunk와 Vector 타입

DuckDB의 기본 데이터 단위는 `DataChunk`입니다. 이는 최대 2048개의 row를 **열(column)** 단위로 저장하는 컬렉션입니다.

```
DataChunk (2048 rows 최대):
  Column 0 (amount, FLOAT):  [73.2, 120.5, 45.0, ... 2048개]
  Column 1 (price, DOUBLE):  [500, 1200, 300, ... 2048개]
  Column 2 (user_id, INT64): [1001, 1002, 1003, ... 2048개]
```

각 `Vector`는 단순 배열이 아니라 **물리적 타입**이 있습니다.

- `FLAT_VECTOR`: 단순 배열 (가장 일반적)
- `DICTIONARY_VECTOR`: 카디널리티가 낮은 문자열 (예: 성별, 국가코드) 딕셔너리 인코딩
- `CONSTANT_VECTOR`: 모든 값이 동일 (예: `WHERE 1=1` 결과) — 배열 대신 단일 값 저장
- `SEQUENCE_VECTOR`: 연속적인 정수 시퀀스 — 배열 없이 시작값+스텝만 저장

### 선택 벡터 (Selection Vector)

DuckDB의 Filter 오퍼레이터는 데이터를 실제로 복사하지 않습니다. 대신 조건을 만족하는 행의 **인덱스 배열**인 Selection Vector를 생성하고, 이후 오퍼레이터들이 이 인덱스 배열을 참조해 데이터에 접근합니다.

```
원본 배열:      [73.2, 120.5, 45.0, 200.1, 30.0]
Filter (>100):
Selection Vec:  [1, 3]  ← 인덱스만 추적, 복사 없음

다음 오퍼레이터가 price[sel[0]], price[sel[1]]을 접근
```

이 방식은 선택률이 낮을 때(조건에 맞는 행이 적을 때) 특히 효과적입니다.

### 적응형 실행 (Adaptive Execution)

DuckDB는 실행 중 통계를 수집해 플랜을 적응적으로 변경합니다. 예를 들어 작은 테이블과 조인할 때 Hash Join을 선택했다가, 런타임에 실제 크기가 예상보다 크다는 걸 감지하면 Grace Hash Join으로 전환합니다.

## 벡터화 vs. 코드 생성 (Code Generation)

벡터화 실행과 경쟁하는 또 다른 접근법이 **Just-In-Time 코드 생성**입니다. HyPer, Umbra 같은 시스템은 쿼리별로 C++ 코드나 LLVM IR을 생성·컴파일해 실행합니다.

| 방식 | 장점 | 단점 |
|------|------|------|
| 벡터화 | 구현 단순, SIMD 친화적 | 인터프리터 오버헤드 존재 |
| 코드 생성 | 최대 성능, 분기 제거 | 컴파일 레이턴시, 구현 복잡 |

DuckDB는 벡터화를 택했고, 많은 쿼리에서 JIT 시스템과 경쟁적인 성능을 보입니다. ClickHouse는 두 접근법을 혼용합니다.

## 주의사항과 팁

**1. 컬럼형 스토리지와 함께 사용해야 효과가 극대화됩니다**: 벡터화 실행은 같은 컬럼의 값들이 메모리에 연속적으로 배치될 때 CPU 캐시 효율이 높아집니다. 행(row) 기반 스토리지에서는 컬럼값이 흩어져 있어 캐시 미스가 발생합니다.

**2. NULL 처리**: 벡터에 NULL이 많으면 SIMD 적용이 복잡해집니다. DuckDB는 Validity Bitmap을 사용해 NULL을 추적하고, NULL이 없는 경우 더 빠른 코드 경로를 선택합니다.

**3. 벡터 크기(Vector Size) 튜닝**: 벡터 크기가 크면 함수 호출 오버헤드가 줄지만 L1 캐시를 초과하면 오히려 느려집니다. DuckDB의 2048은 float 기준 8KB로 L1 캐시(일반적으로 32KB)의 1/4에 해당해 최적화된 값입니다.

**4. OLAP vs OLTP**: 벡터화 실행은 분석(OLAP) 쿼리에 최적입니다. 포인트 조회(OLTP)는 한 번에 row 하나를 가져오므로 벡터화의 이점이 거의 없고, Volcano 모델이 충분히 빠릅니다.

## 참고 자료
- [Why DuckDB — DuckDB 공식 문서](https://duckdb.org/why_duckdb)
- [Vectorized Execution in DuckDB — Medium](https://medium.com/@connect.hashblock/vectorized-execution-in-duckdb-55679d6874f6)
- [What is vectorized query execution? — ClickHouse](https://clickhouse.com/resources/engineering/vectorized-query-execution)
- [The Hidden Power of DuckDB's Vectorized Engine — Medium](https://medium.com/@ThinkingLoop/d4-6-the-hidden-power-of-duckdbs-vectorized-engine-1f719d0c499e)
