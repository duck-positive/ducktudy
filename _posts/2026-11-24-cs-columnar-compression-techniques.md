---
layout: post
title: "열 지향 스토리지 압축 기법 심층 분석: Dictionary, RLE, Delta, Bit-Packing"
date: 2026-11-24
categories: [cs, computer-science]
tags: [database, compression, columnar, parquet, arrow, dictionary-encoding, rle, delta-encoding, bit-packing, analytics]
---

열 지향(Columnar) 데이터베이스와 파일 포맷(Parquet, ORC, Arrow)에서 압축은 단순한 저장 공간 절약이 아닙니다. 압축률이 곧 쿼리 성능입니다. 이 글에서는 현대 분석 시스템의 핵심 압축 기법들을 코드 수준에서 파헤칩니다.

## 개념 설명: 왜 열 지향에서 압축이 강력한가

행 지향(Row-oriented) 스토리지는 다양한 타입의 값이 혼재되어 있어 압축 효율이 낮습니다. 반면 열 지향 스토리지는 같은 타입, 같은 의미의 값들이 연속으로 저장되기 때문에 압축에 훨씬 유리합니다.

```
행 지향:  [KR, Alice, 30] [US, Bob, 25] [KR, Charlie, 28]
열 지향:  country=[KR, US, KR]  name=[Alice, Bob, Charlie]  age=[30, 25, 28]
```

`country` 컬럼은 반복값이 많아 극단적인 압축이 가능합니다. `age` 컬럼은 작은 정수 범위라 비트 패킹이 효과적입니다.

### 주요 압축 기법 개요

| 기법 | 적합한 경우 | 압축 원리 |
|------|------------|-----------|
| Dictionary Encoding | 저카디널리티 컬럼 (국가, 상태코드 등) | 고유값 사전 + 정수 인덱스 |
| Run-Length Encoding (RLE) | 연속 반복값이 많은 컬럼 | (값, 반복횟수) 쌍으로 저장 |
| Bit-Packing | 작은 정수 범위 컬럼 | 불필요한 상위 비트 제거 |
| Delta Encoding | 단조증가 시퀀스 (타임스탬프, ID) | 차이값만 저장 |
| Frame of Reference (FOR) | 좁은 범위의 정수 묶음 | 기준값 빼서 작은 수로 변환 |

## 왜 필요한가

### I/O가 병목인 분석 쿼리

OLAP 쿼리는 수억 행에서 소수 컬럼만 읽습니다. 이때 압축의 효과는 두 배입니다.

1. **디스크/네트워크 I/O 감소**: 압축된 데이터를 읽어오므로 I/O 양이 줄어듭니다.
2. **Late Materialization**: 압축 상태에서 필터를 적용한 뒤, 조건을 만족하는 행만 압축 해제합니다. 불필요한 데이터는 압축 해제조차 하지 않습니다.

### Parquet의 하이브리드 인코딩

Apache Parquet은 여러 인코딩을 조합합니다.

1. 컬럼의 고유값이 적으면 → **Dictionary Encoding** 적용
2. 사전 크기가 임계값을 넘으면 → **Plain Encoding**으로 폴백
3. Dictionary 인덱스(정수 배열)는 → **RLE + Bit-Packing 하이브리드**로 추가 압축
   - 같은 값이 8번 이상 연속: RLE 사용
   - 그 미만: Bit-Packing 사용

```
원본: [KR, US, KR, KR, KR, KR, KR, KR, KR, JP]
         ↓ Dictionary Encoding
사전: {KR:0, US:1, JP:2}
인덱스: [0, 1, 0, 0, 0, 0, 0, 0, 0, 2]
         ↓ RLE/Bit-Packing Hybrid
결과: RLE(0, 1번), Plain(1), RLE(0, 8번), Plain(2)
       → RLE 임계값(8)에 걸려 혼합 적용
```

## 실제 구현 예제

### 예제 1: 사전 인코딩 + RLE + 비트 패킹

```python
from __future__ import annotations
from typing import Any
import math


def dictionary_encode(values: list[Any]) -> tuple[list[Any], list[int]]:
    """
    사전 인코딩: 고유값 사전 + 인덱스 배열로 변환.

    [KR, US, KR, JP, KR] → 사전:[KR, US, JP], 인덱스:[0,1,0,2,0]
    """
    seen: dict[Any, int] = {}
    indices: list[int] = []
    for v in values:
        if v not in seen:
            seen[v] = len(seen)
        indices.append(seen[v])
    dictionary = sorted(seen, key=lambda k: seen[k])
    return dictionary, indices


def rle_encode(values: list[int]) -> list[tuple[int, int]]:
    """
    런 길이 인코딩: 연속 같은 값을 (값, 개수) 쌍으로 압축.

    [0,0,0,1,1,2,0,0] → [(0,3),(1,2),(2,1),(0,2)]
    """
    if not values:
        return []
    runs: list[tuple[int, int]] = []
    cur, cnt = values[0], 1
    for v in values[1:]:
        if v == cur:
            cnt += 1
        else:
            runs.append((cur, cnt))
            cur, cnt = v, 1
    runs.append((cur, cnt))
    return runs


def bit_pack_width(max_value: int) -> int:
    """주어진 최댓값을 표현하는 데 필요한 비트 수."""
    return max(1, max_value.bit_length())


def bit_pack(values: list[int], width: int) -> bytes:
    """
    비트 패킹: 각 값을 width 비트로 압축.

    width=3이면 0~7을 3비트로 저장 → 8개 값 = 3바이트 (원래 8바이트)
    """
    result = bytearray()
    buf, bits = 0, 0
    for v in values:
        buf |= (v << bits)
        bits += width
        while bits >= 8:
            result.append(buf & 0xFF)
            buf >>= 8
            bits -= 8
    if bits > 0:
        result.append(buf & 0xFF)
    return bytes(result)


def bit_unpack(data: bytes, width: int, count: int) -> list[int]:
    """비트 패킹된 데이터를 복원."""
    mask = (1 << width) - 1
    result = []
    buf, bits = 0, 0
    byte_iter = iter(data)
    for _ in range(count):
        while bits < width:
            buf |= next(byte_iter) << bits
            bits += 8
        result.append(buf & mask)
        buf >>= width
        bits -= width
    return result


# 압축 효과 측정
def compression_demo():
    import random
    random.seed(42)

    # 국가 코드 컬럼 (저카디널리티, 반복 많음)
    country_col = (['KR'] * 50_000 + ['US'] * 30_000 +
                   ['JP'] * 15_000 + ['DE'] * 5_000)
    random.shuffle(country_col)

    original_bytes = sum(len(c.encode()) for c in country_col)
    dictionary, indices = dictionary_encode(country_col)
    rle_runs = rle_encode(indices)

    bit_w = bit_pack_width(len(dictionary) - 1)
    packed = bit_pack(indices, bit_w)

    print("=== 국가 코드 컬럼 압축 효과 ===")
    print(f"원본:         {original_bytes:>10,} bytes")
    print(f"사전 크기:    {len(dictionary):>10} 항목 ({sum(len(d.encode()) for d in dictionary)} bytes)")
    print(f"RLE 엔트리:   {len(rle_runs):>10} 쌍")
    print(f"비트 패킹:    {len(packed):>10,} bytes  ({bit_w}비트/값)")
    print(f"압축률(BP):   {(1 - len(packed)/original_bytes)*100:>9.1f}%")

    # 정수 ID 컬럼 (작은 범위)
    id_col = [random.randint(0, 1023) for _ in range(100_000)]
    orig_id = len(id_col) * 4  # int32
    packed_id = bit_pack(id_col, 10)  # 0~1023 = 10비트

    print("\n=== 정수 ID 컬럼 (0~1023) 압축 효과 ===")
    print(f"원본(int32):  {orig_id:>10,} bytes")
    print(f"비트 패킹:    {len(packed_id):>10,} bytes  (10비트/값)")
    print(f"압축률:       {(1 - len(packed_id)/orig_id)*100:>9.1f}%")


compression_demo()
```

### 예제 2: 델타 인코딩 + Frame of Reference

```python
from __future__ import annotations
import struct
import time


def delta_encode(values: list[int]) -> tuple[int, list[int]]:
    """
    델타 인코딩: 연속 차이만 저장.

    타임스탬프, 자동 증가 ID 등 단조 증가 시퀀스에 최적.
    [1000, 1001, 1003, 1002, 1008] → 첫값:1000, 델타:[1,2,-1,6]
    """
    if not values:
        return 0, []
    deltas = [values[i] - values[i - 1] for i in range(1, len(values))]
    return values[0], deltas


def frame_of_reference_encode(values: list[int]) -> tuple[int, list[int]]:
    """
    FOR(Frame of Reference) 인코딩: 최솟값을 기준으로 변환.

    모든 값에서 최솟값을 빼면 범위가 작아져 비트 패킹 효율 증가.
    [1000000, 1000003, 1000001, 1000007]
      → 기준:1000000, 나머지:[0,3,1,7]  (3비트로 표현 가능!)
    """
    if not values:
        return 0, []
    base = min(values)
    return base, [v - base for v in values]


def measure_timestamp_compression(n: int = 100_000) -> None:
    """1초 간격 타임스탬프 압축 비교."""
    base_ts = int(time.time())
    # 1초 간격에 약간의 지터(jitter) 추가
    import random
    random.seed(0)
    timestamps = sorted(
        base_ts + i + random.randint(-2, 2) for i in range(n)
    )

    # 원본
    orig_bytes = n * 8  # int64

    # 델타 인코딩
    first_val, deltas = delta_encode(timestamps)
    max_delta = max(abs(d) for d in deltas)
    delta_width = max_delta.bit_length() + 1  # 부호 비트 포함

    # FOR + 비트 패킹
    for_base, for_values = frame_of_reference_encode(timestamps)
    for_max = max(for_values)
    for_width = for_max.bit_length()

    # 델타 비트 패킹 (부호 처리: zigzag encoding)
    def zigzag_encode(v: int) -> int:
        return (v << 1) ^ (v >> 63)

    zz_deltas = [zigzag_encode(d) for d in deltas]
    zz_max = max(zz_deltas)
    zz_width = zz_max.bit_length()
    delta_packed_size = math.ceil(len(zz_deltas) * zz_width / 8) + 8

    for_packed_size = math.ceil(len(for_values) * for_width / 8) + 8

    print(f"=== 타임스탬프 {n:,}개 압축 비교 ===")
    print(f"원본(int64):        {orig_bytes:>10,} bytes")
    print(f"최대 델타:          {max_delta:>10,}  ({delta_width}비트 필요)")
    print(f"Delta + ZigZag BP:  {delta_packed_size:>10,} bytes  ({zz_width}비트/값)")
    print(f"FOR + BP:           {for_packed_size:>10,} bytes  ({for_width}비트/값)")
    print(f"Delta 압축률:       {(1-delta_packed_size/orig_bytes)*100:>9.1f}%")
    print(f"FOR 압축률:         {(1-for_packed_size/orig_bytes)*100:>9.1f}%")


import math
measure_timestamp_compression()
```

## 주의사항/팁

### Parquet RLE/Bit-Packing 하이브리드 임계값

Parquet 스펙에 따르면 사전 인덱스 압축 시:
- 연속 같은 값이 **8번 이상**: RLE 사용
- **8번 미만**: Bit-Packing 사용

이 임계값은 "8개를 Bit-Pack하는 것이 (값, 카운트) RLE 쌍 1개와 비슷한 크기"에서 나왔습니다. 데이터 분포에 따라 성능 차이가 크게 납니다.

### Late Materialization (지연 구체화)

압축 기법의 진정한 힘은 Late Materialization과 결합할 때 나옵니다.

```
-- 나이가 25세인 국가 코드 조회
SELECT country FROM users WHERE age = 25

행 지향 처리:
  → 모든 행을 읽어 압축 해제 → age=25 필터 → country 추출

열 지향 + Late Materialization:
  → age 컬럼만 압축 해제 → age=25인 행 ID 목록 획득
  → country 컬럼에서 해당 행 ID만 압축 해제 → 완료
  → 나머지 컬럼(name, email 등)은 압축 해제조차 안 함!
```

### 압축 계층 구조

현대 열 지향 시스템은 두 층으로 압축합니다.

1. **경량 인코딩 (Light-weight Encoding)**: Dictionary, RLE, Bit-Packing, Delta — CPU 비용이 낮고, 쿼리 처리 중에도 활용 가능
2. **블록 압축 (Block Compression)**: Snappy, ZSTD, LZ4 — 범용 압축, 디스크 I/O 절감에 더 집중

경량 인코딩 후 블록 압축을 추가로 적용하면 압축률이 더 높아지지만, CPU 오버헤드도 증가합니다. ZSTD는 Snappy보다 느리지만 압축률이 훨씬 높습니다.

### 카디널리티에 따른 인코딩 선택

```
저카디널리티 (고유값 < 수백): Dictionary Encoding → 인덱스에 RLE/BP
중간 카디널리티: Dictionary + 블록 압축
고카디널리티 + 단조증가: Delta Encoding → Bit-Packing
고카디널리티 + 좁은 범위: Frame of Reference → Bit-Packing
무작위 고카디널리티: 블록 압축만 적용
```

### SIMD와의 결합

Bit-Packing과 Bit-Unpacking은 SIMD 명령어로 병렬화하면 성능이 대폭 향상됩니다. Apache Arrow의 C++ 구현은 AVX2 SIMD를 활용하여 압축 해제 속도를 크게 높입니다. 이것이 Arrow가 쿼리 처리용으로 설계된 이유 중 하나입니다(Arrow는 기본적으로 경량 인코딩만 사용하여 압축 해제 없이 쿼리를 처리할 수 있도록 합니다).

## 참고 자료

- [Apache Parquet Encodings 공식 문서](https://parquet.apache.org/docs/file-format/data-pages/encodings/)
- [An Empirical Evaluation of Columnar Storage Formats - arXiv 2304.05028](https://arxiv.org/abs/2304.05028)
- [ClickHouse: Database Compression Encodings](https://clickhouse.com/engineering-resources/database-compression)
- [Data Formats in Analytical DBMSs - arXiv 2411.14331](https://arxiv.org/abs/2411.14331)
