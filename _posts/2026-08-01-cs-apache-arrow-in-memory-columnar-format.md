---
layout: post
title: "Apache Arrow 인메모리 컬럼형 포맷 완전 정복: 제로 카피로 분석 시스템을 연결하는 표준"
date: 2026-08-01
categories: [cs, computer-science]
tags: [apache-arrow, columnar-format, in-memory, zero-copy, analytics, data-engineering, pandas, polars]
---

현대 데이터 엔지니어링 생태계에는 수십 개의 분석 도구가 공존한다. Pandas, Spark, DuckDB, Polars, Snowflake, BigQuery… 이들이 서로 데이터를 주고받을 때 매번 포맷을 변환한다면 엄청난 성능 낭비가 발생한다. Apache Arrow는 이 문제를 해결하기 위해 탄생한 언어 독립적 인메모리 컬럼형 데이터 포맷 표준이다. 이 글에서는 Arrow의 내부 구조와 동작 원리, 그리고 실제 활용 방법을 깊이 있게 다룬다.

## 왜 Apache Arrow가 필요한가

### 기존 방식의 문제: 데이터 변환 비용

데이터 파이프라인에서 가장 큰 병목 중 하나는 데이터 직렬화/역직렬화 비용이다. 예를 들어 Pandas DataFrame을 Spark에 전달하거나, DuckDB 결과를 Polars로 가져올 때 각 시스템은 자체적인 메모리 표현 방식을 사용하기 때문에 다음과 같은 오버헤드가 발생한다.

1. **직렬화 비용**: 시스템 A의 메모리 레이아웃 → 공통 중간 포맷(JSON, CSV 등)으로 변환
2. **역직렬화 비용**: 중간 포맷 → 시스템 B의 메모리 레이아웃으로 변환
3. **메모리 복사 비용**: 각 단계에서 추가적인 메모리 할당과 복사 발생
4. **타입 변환 비용**: 시스템마다 다른 수치 타입 처리

이런 비용은 대용량 데이터셋에서 전체 처리 시간의 80%를 차지하는 경우도 있다. "데이터를 실제로 처리하는 시간보다 이동시키는 시간이 더 오래 걸린다"는 문제는 Wes McKinney(Pandas 창시자)가 2016년에 Arrow 프로젝트를 제안하면서 가장 먼저 지적한 핵심 문제였다.

### Arrow의 핵심 아이디어: 공통 인메모리 표준

Apache Arrow는 모든 분석 시스템이 동일한 메모리 레이아웃을 사용하면 데이터 복사 없이 포인터만 전달할 수 있다는 아이디어에서 출발한다. 이를 **제로 카피(Zero-Copy)** 데이터 공유라고 한다.

```
전통적 방식:
Pandas → 직렬화 → 전송 → 역직렬화 → Spark
                  [복사1]          [복사2]

Arrow 방식:
Pandas (Arrow 버퍼) → 포인터 전달 → Spark (Arrow 버퍼)
                      [복사 없음]
```

## Apache Arrow의 컬럼형 메모리 레이아웃

### 행(Row) vs 컬럼(Column) 저장 방식

관계형 데이터베이스 전통의 행 저장 방식(Row-oriented)과 Arrow의 컬럼 저장 방식(Column-oriented)의 차이는 분석 쿼리 성능에 극적인 영향을 미친다.

```
행 저장 방식 (Row-oriented):
메모리: [id=1, name="Alice", age=30] [id=2, name="Bob", age=25] [id=3, ...]

컬럼 저장 방식 (Column-oriented, Arrow):
id 버퍼:   [1, 2, 3, 4, 5, ...]
name 버퍼: ["Alice", "Bob", "Charlie", ...]
age 버퍼:  [30, 25, 35, ...]
```

"모든 사람의 평균 나이를 구하라"는 쿼리는 `age` 컬럼만 읽으면 된다. 컬럼 저장 방식에서는 연속된 메모리 블록 하나만 읽지만, 행 저장 방식에서는 모든 행을 읽으면서 age 필드만 추출해야 한다. CPU 캐시 효율성과 SIMD 벡터 연산 활용도 면에서 컬럼 방식이 압도적으로 유리하다.

### Arrow IPC 포맷의 물리적 레이아웃

Arrow 컬럼형 포맷의 핵심은 **버퍼(Buffer)** 개념이다. 각 컬럼은 최대 3종류의 연속된 바이트 버퍼로 구성된다.

#### 1. Validity Bitmap (유효성 비트맵)

NULL 값을 표현하기 위한 비트맵. 1비트가 1행을 나타내며, 1이면 유효한 값, 0이면 NULL.

```
컬럼 값: [1, NULL, 3, 4, NULL]
비트맵:  [1,    0, 1, 1,    0]  → 0b00011101 = 0x1D (바이트로 패킹)
```

NULL이 없는 컬럼은 비트맵을 생략하여 메모리를 아낄 수 있다.

#### 2. Offsets Buffer (오프셋 버퍼, 가변 길이 타입용)

문자열이나 바이너리처럼 각 원소의 크기가 다른 타입을 위해 사용된다.

```
문자열 컬럼: ["hello", "world", "!"]
값 버퍼(Values): h e l l o w o r l d !
오프셋 버퍼:     [0, 5, 10, 11]
                  ^  ^   ^   ^
                  |  |   |   +-- "!" 끝 인덱스
                  |  |   +------ "world" 끝 인덱스
                  |  +---------- "hello" 끝 인덱스
                  +------------- 시작 인덱스

i번째 문자열 = 값 버퍼[offsets[i] .. offsets[i+1]]
```

이 방식 덕분에 각 문자열을 별도의 힙 객체로 할당하지 않고 하나의 연속된 버퍼에 모두 저장할 수 있다. 메모리 단편화가 없고 캐시 효율이 높다.

#### 3. Values Buffer (값 버퍼)

고정 길이 타입(int32, float64 등)의 경우 값 자체를 연속된 메모리에 저장한다. C 배열과 동일한 레이아웃이므로 SIMD 연산에 직접 사용 가능하다.

### 메모리 정렬 요구사항

Arrow 명세는 버퍼를 **64바이트 경계**에 정렬하도록 권장하고, 버퍼 길이도 64바이트의 배수로 패딩한다. 이는 AVX-512 SIMD 레지스터(64바이트)에 최적화된 것이다.

```
int32 배열 [1, 2, 3] 의 Values 버퍼:
실제 데이터: [1, 2, 3] = 12바이트
패딩 후:     [1, 2, 3, 0, 0, ..., 0] = 64바이트 (52바이트 패딩)
```

### RecordBatch와 Schema

Arrow에서 데이터의 기본 단위는 **RecordBatch**다. RecordBatch는 동일한 길이를 가진 컬럼들의 집합이며, 각 컬럼은 **ChunkedArray**로 구성된다.

```
Schema: [
  Field("id",   Int32,   nullable=False),
  Field("name", Utf8,    nullable=True),
  Field("score", Float64, nullable=False)
]

RecordBatch (3행):
  id 컬럼:    [1, 2, 3]           → Int32 버퍼
  name 컬럼:  ["Alice", NULL, "Charlie"] → Validity 비트맵 + Offset + Values
  score 컬럼: [95.5, 87.0, 92.3]  → Float64 버퍼
```

## Python으로 Apache Arrow 실습

### 기본 사용법

```python
import pyarrow as pa
import pyarrow.compute as pc

# 1. RecordBatch 직접 생성
data = {
    "id":    pa.array([1, 2, 3, 4, 5], type=pa.int32()),
    "name":  pa.array(["Alice", "Bob", None, "Dave", "Eve"]),
    "score": pa.array([95.5, 87.0, 92.3, 78.1, 88.9], type=pa.float64()),
}
table = pa.table(data)
print(table.schema)
# id: int32
# name: string
# score: double

# 2. 컬럼 접근 및 연산 (제로 카피)
scores = table.column("score")
avg = pc.mean(scores)
print(f"평균 점수: {avg.as_py():.2f}")  # 88.36

# 3. 필터링 (컬럼 단위 연산 — SIMD 가속)
mask = pc.greater_equal(scores, pa.scalar(90.0))
filtered = table.filter(mask)
print(filtered)
# id: [[1,3]]
# name: [["Alice","null"]]  -- null은 그대로 포함됨
# score: [[95.5,92.3]]

# 4. 메모리 레이아웃 직접 확인
id_array = table.column("id")
buf = id_array.buffers()[1]  # values 버퍼 (index 0은 validity bitmap)
print(f"버퍼 주소: 0x{buf.address:x}")
print(f"버퍼 크기: {buf.size} bytes")  # 5 * 4 = 20바이트 (+ 패딩)

# numpy와 제로 카피 공유
import numpy as np
np_arr = id_array.to_pylist()  # 복사 발생
np_arr_zero_copy = id_array.to_numpy(zero_copy_only=True)  # 복사 없음!
print(f"numpy 배열 주소 == Arrow 버퍼 주소: {np_arr_zero_copy.ctypes.data == buf.address}")
```

### Arrow IPC를 통한 프로세스 간 제로 카피 데이터 전송

Arrow의 IPC(Inter-Process Communication) 포맷은 공유 메모리를 통해 프로세스 간에 데이터를 복사 없이 전달할 수 있다.

```python
import pyarrow as pa
import pyarrow.ipc as ipc
import multiprocessing.shared_memory as shm
import io

# === 생산자 프로세스 ===
def producer():
    table = pa.table({
        "x": pa.array(range(1_000_000), type=pa.int64()),
        "y": pa.array([i * 2.5 for i in range(1_000_000)], type=pa.float64()),
    })
    
    # IPC 스트림으로 직렬화 (Arrow 자체 포맷 — 메타데이터 포함)
    sink = io.BytesIO()
    writer = ipc.new_stream(sink, table.schema)
    for batch in table.to_batches(max_chunksize=100_000):
        writer.write_batch(batch)
    writer.close()
    
    arrow_bytes = sink.getvalue()
    print(f"직렬화 크기: {len(arrow_bytes) / 1024 / 1024:.2f} MB")
    return arrow_bytes

# === 소비자 프로세스 ===
def consumer(arrow_bytes: bytes):
    reader = ipc.open_stream(io.BytesIO(arrow_bytes))
    table = reader.read_all()
    
    # 역직렬화 후 즉시 Arrow 컬럼 연산 사용 가능
    import pyarrow.compute as pc
    total = pc.sum(table.column("x"))
    print(f"합계: {total.as_py()}")  # 499999500000

arrow_data = producer()
consumer(arrow_data)

# 실제 제로 카피: 공유 메모리 사용
# mmap이나 plasma store를 통해 포인터만 전달하면
# 소비자 프로세스는 복사 없이 동일 메모리에 접근
```

### Pandas와 Polars의 Arrow 기반 상호운용

```python
import pyarrow as pa
import pandas as pd
import polars as pl

# Arrow RecordBatch → Pandas (제로 카피 가능)
arrow_table = pa.table({
    "timestamp": pa.array([1720000000, 1720000001, 1720000002], type=pa.int64()),
    "value":     pa.array([3.14, 2.71, 1.41], type=pa.float64()),
})

# Pandas로 변환 (내부적으로 Arrow 버퍼 재사용 시도)
df_pandas = arrow_table.to_pandas(zero_copy_only=False)
# zero_copy_only=True이면 복사가 필요한 경우 ArrowInvalid 예외 발생

# Polars로 변환 (Polars는 내부적으로 Arrow 기반)
df_polars = pl.from_arrow(arrow_table)

# Polars → Arrow (실질적 제로 카피 — Polars의 내부 표현이 Arrow 그 자체)
arrow_back = df_polars.to_arrow()
print(f"원본과 동일한 메모리? {arrow_table.equals(arrow_back)}")  # True

# DuckDB도 Arrow 지원 — SQL로 Arrow 테이블 직접 쿼리
import duckdb
result = duckdb.query("SELECT AVG(value) FROM arrow_table").arrow()
print(result)  # Arrow Table 반환 — 직렬화 없음
```

## Arrow Flight: 네트워크를 통한 고성능 데이터 전송

Arrow IPC가 같은 머신의 프로세스 간 통신이라면, **Arrow Flight**는 네트워크를 통한 대용량 데이터 전송 프로토콜이다. gRPC 위에서 동작하며 Arrow 포맷을 직접 스트리밍한다.

```
전통적 REST API:
클라이언트 → HTTP 요청 → 서버 JSON 생성 → 전송 → 클라이언트 JSON 파싱 → Pandas

Arrow Flight:
클라이언트 → Flight 요청 → 서버 Arrow 버퍼 스트리밍 → 클라이언트 Arrow 버퍼 수신 → 즉시 사용
```

Flight는 멀티스레드 스트리밍과 서버 사이드 필터링(Predicate Pushdown)을 지원한다.

## Apache Arrow와 Apache Parquet의 차이

자주 혼동되는 두 포맷의 차이를 명확히 정리한다.

| 특성 | Apache Arrow | Apache Parquet |
|------|-------------|----------------|
| 대상 | 인메모리 표현 | 디스크 저장 포맷 |
| 목적 | 제로 카피 프로세스 간 공유 | 압축·장기 보관 |
| 압축 | 없음 (빠른 접근 우선) | 강력한 압축 지원 |
| 랜덤 접근 | O(1) | 느림 (Row Group 단위) |
| 타입 매핑 | 1:1 대응 가능 | 필요시 Arrow로 변환 |

실제 워크플로에서는 둘을 함께 사용한다: Parquet으로 데이터를 저장하고, 분석 시에 Arrow 포맷으로 읽어 처리한다.

## 주의사항과 팁

### 1. 딕셔너리 인코딩으로 고카디널리티 컬럼 최적화

반복되는 문자열 값이 많은 컬럼은 Dictionary Encoding을 사용하면 메모리와 처리 속도를 크게 개선할 수 있다.

```python
import pyarrow as pa

# 일반 문자열 배열 (각 값을 그대로 저장)
plain = pa.array(["low", "high", "low", "medium", "low"] * 200_000)
print(f"일반: {plain.nbytes / 1024 / 1024:.2f} MB")  # ~5.7 MB

# 딕셔너리 인코딩 (고유값 인덱스로 저장)
encoded = plain.dictionary_encode()
print(f"인코딩: {encoded.nbytes / 1024 / 1024:.2f} MB")  # ~0.8 MB
print(f"딕셔너리: {encoded.dictionary}")  # ["low", "high", "medium"]
```

### 2. Chunked Array vs Contiguous Array

대용량 데이터를 처리할 때 하나의 연속된 버퍼 대신 여러 청크로 나눠 관리하면 메모리 재할당을 피할 수 있다.

```python
# 스트리밍 데이터를 점진적으로 추가할 때
chunks = [pa.array([1, 2, 3]), pa.array([4, 5, 6])]
chunked = pa.chunked_array(chunks)
# 메모리 복사 없이 청크들을 논리적으로 연결
print(chunked[4].as_py())  # 5 (두 번째 청크 인덱스 1)
```

### 3. 메모리 풀 관리

Arrow는 자체 메모리 풀을 사용한다. 시스템 기본 할당자 대신 jemalloc이나 mimalloc을 지정하면 성능을 향상시킬 수 있다.

```python
import pyarrow as pa

pool = pa.system_memory_pool()  # 기본값
print(f"현재 할당: {pool.bytes_allocated() / 1024:.1f} KB")

# 커스텀 풀로 메모리 추적
logging_pool = pa.logging_memory_pool(pool)
arr = pa.array([1, 2, 3], memory_pool=logging_pool)
```

## 참고 자료

- [Arrow Columnar Format Specification](https://arrow.apache.org/docs/format/Columnar.html)
- [Apache Arrow 공식 홈페이지](https://arrow.apache.org/)
- [Apache Arrow Python Cookbook](https://arrow.apache.org/cookbook/py/)
- [Wes McKinney - Apache Arrow and the 10 Things I Hate About Pandas](https://wesmckinney.com/blog/apache-arrow-pandas-internals/)
