---
layout: post
title: "Apache Arrow와 컬럼형 메모리 포맷 완전 정복: Pandas·DuckDB·Spark가 10배 빠른 이유"
date: 2026-11-07
categories: [cs, computer-science]
tags: [apache-arrow, columnar-storage, parquet, duckdb, simd, zero-copy, data-engineering]
---

## 컬럼형 메모리 포맷이란?

데이터를 메모리에 배치하는 방법에는 크게 두 가지가 있습니다. 행 지향(row-oriented)과 열 지향(columnar, column-oriented)입니다.

**행 지향 레이아웃** (전통적인 RDBMS, 대부분의 ORM)
```
메모리:  [id=1, name="Alice", age=30] [id=2, name="Bob", age=25] [id=3, ...]
         ← 한 레코드가 연속된 메모리에 ──────────────────────────────────── →
```

**컬럼 지향 레이아웃** (Apache Arrow, DuckDB, ClickHouse, Parquet)
```
메모리:  [1, 2, 3, 4, ...]        ← id 컬럼
         ["Alice", "Bob", ...]     ← name 컬럼
         [30, 25, 28, ...]         ← age 컬럼
```

분석 쿼리에서 왜 컬럼 지향이 압도적으로 빠른지 한 가지 예로 살펴봅시다. `SELECT AVG(age) FROM users WHERE country = 'KR'` 쿼리를 처리할 때:

- **행 지향:** 모든 행을 읽어 country와 age 필드를 추출해야 함. 필요 없는 name, email 등의 필드도 메모리에서 읽어야 함.
- **컬럼 지향:** country 컬럼과 age 컬럼만 읽으면 됨. I/O가 최소화되고, 같은 타입의 연속된 데이터는 SIMD 벡터 연산으로 처리 가능.

## Apache Arrow: 표준 인메모리 컬럼 포맷

Apache Arrow는 2016년에 시작된 오픈소스 프로젝트로, 여러 시스템 간에 데이터를 **제로 카피(zero-copy)**로 공유할 수 있는 표준 컬럼형 인메모리 포맷을 정의합니다. 2024년 현재 Pandas, Polars, DuckDB, PySpark, Dask, cuDF, Flink 등 주요 데이터 도구들이 Arrow를 기본 포맷으로 채택했습니다.

### Arrow가 해결하는 문제

Arrow 이전에는 Pandas DataFrame을 PySpark로 보내려면:
1. Pandas DataFrame → Python dict/list 변환
2. dict/list → Pickle/JSON 직렬화
3. 역직렬화 → Spark Row 객체 생성

이 과정에서 메모리 복사가 여러 번 일어나고, Python GIL 때문에 단일 스레드로 처리됩니다. Arrow를 사용하면 동일한 메모리 버퍼를 Pandas와 Spark가 직접 가리킬 수 있습니다.

## Arrow 메모리 레이아웃의 내부 구조

Arrow의 컬럼형 배열은 다음 세 가지 버퍼로 구성됩니다:

```
Fixed-size primitive array (예: int64):
  validity bitmap:  [1, 1, 0, 1, ...]   ← null 여부 (1=유효, 0=null)
  values buffer:    [1, 2, 0, 4, ...]   ← 실제 값 (null이어도 슬롯 존재)

Variable-size binary array (예: string):
  validity bitmap:  [1, 1, 1, ...]
  offsets buffer:   [0, 5, 8, 15, ...]  ← 각 문자열의 시작 오프셋
  data buffer:      "AliceBobCharlie..." ← 실제 문자열 데이터 연속 저장

List array (예: array<int>):
  validity bitmap
  offsets buffer:   [0, 3, 5, ...]      ← 각 리스트의 시작 위치
  child array:      [1,2,3, 4,5, ...]   ← 중첩 배열의 값
```

Arrow는 버퍼를 **64바이트 정렬**하여 AVX-512 SIMD 명령어가 정렬된 로드(aligned load)를 사용할 수 있도록 합니다. 또한 버퍼 크기를 64바이트의 배수로 패딩하여 마지막 청크도 완전한 SIMD 레지스터로 처리 가능합니다.

## 실제 구현 예제

### PyArrow로 Arrow 메모리 레이아웃 직접 조작

```python
import pyarrow as pa
import pyarrow.compute as pc
import numpy as np
import time

# ── 1. Arrow 배열 생성과 버퍼 구조 확인 ──────────────────────────────

# 정수 배열 (null 포함)
int_array = pa.array([1, 2, None, 4, 5], type=pa.int64())
print(f"정수 배열: {int_array}")
print(f"버퍼 수: {len(int_array.buffers())}")

validity_buf = int_array.buffers()[0]  # validity bitmap
values_buf   = int_array.buffers()[1]  # 값 버퍼

if validity_buf:
    # validity bitmap은 비트 단위: 각 바이트에 8개 원소의 null 여부
    print(f"Validity bitmap (hex): {validity_buf.to_pybytes().hex()}")
    # 출력: ef (11101111 → 비트 2가 0 = 3번째 원소가 null)

print(f"Values buffer (hex): {values_buf.to_pybytes().hex()}")

# 문자열 배열의 오프셋 구조
str_array = pa.array(["Alice", "Bob", "Charlie"], type=pa.string())
offsets_buf = str_array.buffers()[1]  # 오프셋 버퍼
data_buf    = str_array.buffers()[2]  # 실제 문자열 데이터

offsets = np.frombuffer(offsets_buf, dtype=np.int32)
print(f"\n문자열 오프셋: {offsets}")   # [0, 5, 8, 15]
print(f"문자열 데이터: {data_buf.to_pybytes()}")  # b'AliceBobCharlie'

# ── 2. 제로 카피 슬라이싱 ───────────────────────────────────────────

large_array = pa.array(range(10_000_000), type=pa.int64())

# 슬라이스는 메모리를 복사하지 않고 오프셋과 길이만 조정
slice_start = time.perf_counter()
sliced = large_array.slice(1_000_000, 500_000)
slice_elapsed = time.perf_counter() - slice_start

print(f"\n슬라이스 시간 (10M 원소 중 500K): {slice_elapsed*1e6:.1f} μs (제로 카피)")
print(f"슬라이스 길이: {len(sliced)}")

# ── 3. SIMD 가속 컴퓨팅 ─────────────────────────────────────────────

# pyarrow.compute는 C++ Arrow 계산 엔진을 호출 (SIMD 자동 활용)
arr = pa.array(np.random.randint(0, 1000, size=10_000_000, dtype=np.int64))

start = time.perf_counter()
result_arrow = pc.sum(arr)
arrow_elapsed = time.perf_counter() - start

start = time.perf_counter()
result_numpy = np.sum(arr.to_pylist())  # Python 리스트 변환 후 합계 (비효율)
numpy_slow_elapsed = time.perf_counter() - start

start = time.perf_counter()
result_numpy2 = np.sum(np.array(arr))  # NumPy 배열로 직접 변환 (제로 카피 가능)
numpy_fast_elapsed = time.perf_counter() - start

print(f"\n1,000만 원소 합계:")
print(f"  Arrow compute (SIMD):  {arrow_elapsed*1000:.2f} ms → {result_arrow}")
print(f"  Python list sum:        {numpy_slow_elapsed*1000:.2f} ms")
print(f"  NumPy (via Arrow buf):  {numpy_fast_elapsed*1000:.2f} ms")

# ── 4. RecordBatch: 여러 컬럼을 하나의 단위로 묶기 ──────────────────

schema = pa.schema([
    pa.field("id",      pa.int64()),
    pa.field("name",    pa.string()),
    pa.field("score",   pa.float64()),
    pa.field("active",  pa.bool_()),
])

batch = pa.record_batch({
    "id":     pa.array([1, 2, 3, 4, 5], type=pa.int64()),
    "name":   pa.array(["Alice", "Bob", "Carol", "Dave", "Eve"]),
    "score":  pa.array([92.5, 87.0, 95.3, 78.1, 89.9], type=pa.float64()),
    "active": pa.array([True, True, False, True, True]),
}, schema=schema)

print(f"\nRecordBatch: {batch.num_rows} rows × {batch.num_columns} cols")
print(f"Schema: {batch.schema}")

# 필터 조건 적용 (SIMD 가속)
mask = pc.greater(batch.column("score"), pa.scalar(88.0))
filtered = batch.filter(mask)
print(f"score > 88 필터 후: {filtered.num_rows} rows")
print(filtered.to_pydict())
```

### DuckDB + Arrow: 제로 카피 쿼리 엔진

```python
import duckdb
import pyarrow as pa
import pyarrow.parquet as pq
import numpy as np
import time

# ── DuckDB에서 Arrow Table을 직접 쿼리 (제로 카피) ──────────────────

np.random.seed(42)
N = 10_000_000

# Arrow Table 생성 (인메모리)
table = pa.table({
    "user_id":    pa.array(np.random.randint(1, 100_000, N), type=pa.int64()),
    "product_id": pa.array(np.random.randint(1, 10_000, N), type=pa.int64()),
    "amount":     pa.array(np.random.uniform(0.5, 500.0, N), type=pa.float64()),
    "country":    pa.array(np.random.choice(["KR", "US", "JP", "DE"], N)),
    "timestamp":  pa.array(np.random.randint(1_700_000_000, 1_750_000_000, N), type=pa.int64()),
})

print(f"Arrow Table: {N:,} rows, {table.nbytes / 1e6:.1f} MB")

# DuckDB 연결 (인메모리)
con = duckdb.connect()

# DuckDB는 Arrow Table을 직접 스캔 — 메모리 복사 없음!
con.register("orders", table)

# 복잡한 집계 쿼리
query = """
SELECT
    country,
    COUNT(*)           AS order_count,
    SUM(amount)        AS total_amount,
    AVG(amount)        AS avg_amount,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY amount) AS p95_amount
FROM orders
WHERE amount > 10.0
GROUP BY country
ORDER BY total_amount DESC
"""

start = time.perf_counter()
result = con.execute(query).arrow()  # 결과도 Arrow Table로 반환
elapsed = time.perf_counter() - start

print(f"\n집계 쿼리 ({N:,} rows): {elapsed*1000:.1f} ms")
print(result.to_pandas().to_string())

# ── Parquet 파일 읽기와 프레디케이트 푸시다운 ───────────────────────

# Parquet 파일 저장 (실제 환경에서는 이미 존재하는 파일)
pq.write_table(table, "/tmp/orders.parquet",
               compression="snappy",
               row_group_size=100_000)

# 프레디케이트 푸시다운: Parquet 레벨에서 필터링하므로 불필요한 데이터를 디스크에서 읽지 않음
start = time.perf_counter()
filtered_table = pq.read_table(
    "/tmp/orders.parquet",
    filters=[("country", "=", "KR"), ("amount", ">", 100.0)],
    columns=["user_id", "amount"],  # 필요한 컬럼만 읽기 (컬럼 프루닝)
)
parquet_elapsed = time.perf_counter() - start

print(f"\nParquet 필터 읽기 (KR, amount>100): {parquet_elapsed*1000:.1f} ms")
print(f"결과 행 수: {len(filtered_table):,}")

# ── Arrow IPC: 프로세스 간 제로 카피 공유 ───────────────────────────

import pyarrow.ipc as ipc
import io

# Arrow IPC 형식으로 직렬화 (Feather v2 / Arrow IPC Stream)
sink = io.BytesIO()
writer = ipc.new_stream(sink, table.schema)
# RecordBatch 단위로 청크 전송 (스트리밍 처리 가능)
for batch in table.to_batches(max_chunksize=500_000):
    writer.write_batch(batch)
writer.close()

ipc_bytes = len(sink.getvalue())
print(f"\nArrow IPC 크기: {ipc_bytes / 1e6:.1f} MB (원본 {table.nbytes / 1e6:.1f} MB)")

# 역직렬화: shared_memory나 소켓을 통해 받은 경우 제로 카피 가능
sink.seek(0)
reader = ipc.open_stream(sink)
recovered_table = reader.read_all()
print(f"IPC 역직렬화 후 행 수: {len(recovered_table):,} (동일: {len(recovered_table) == N})")
```

## Arrow Flight: 고성능 데이터 전송 프로토콜

Arrow Flight는 HTTP/2(gRPC) 위에서 Arrow 데이터를 직접 스트리밍하는 RPC 프레임워크입니다. 직렬화/역직렬화 오버헤드 없이 Arrow RecordBatch를 그대로 전송합니다.

```python
# Arrow Flight 서버 예시 (간략화)
import pyarrow.flight as flight

class SimpleFlightServer(flight.FlightServerBase):
    def __init__(self, location, **kwargs):
        super().__init__(location, **kwargs)
        self._tables = {}

    def do_put(self, context, descriptor, reader, writer):
        """클라이언트가 데이터를 업로드할 때 호출"""
        key = descriptor.path[0].decode()
        self._tables[key] = reader.read_all()
        print(f"수신: {key}, {len(self._tables[key])} rows")

    def do_get(self, context, ticket):
        """클라이언트가 데이터를 요청할 때 호출"""
        key = ticket.ticket.decode()
        table = self._tables[key]
        # RecordBatch 스트림으로 전송 (제로 카피)
        return flight.RecordBatchStream(table)

    def list_flights(self, context, criteria):
        for key, table in self._tables.items():
            descriptor = flight.FlightDescriptor.for_path(key)
            endpoint = flight.FlightEndpoint(key, [self._server_location])
            yield flight.FlightInfo(table.schema, descriptor, [endpoint],
                                    table.num_rows, table.nbytes)

# 서버 시작
# server = SimpleFlightServer("grpc://localhost:8815")
# server.serve()

# 클라이언트 사용 예시
# client = flight.connect("grpc://localhost:8815")
# 업로드: client.do_put(descriptor, table.schema) → writer.write_table(table)
# 다운로드: reader = client.do_get(ticket) → table = reader.read_all()
```

## Arrow vs Parquet: 무엇이 다른가?

혼동하기 쉬운 Arrow와 Parquet의 차이를 명확히 정리합니다:

| 특성 | Apache Arrow | Apache Parquet |
|------|-------------|----------------|
| 목적 | 인메모리 처리, 프로세스 간 공유 | 디스크 저장, 장기 보관 |
| 압축 | 없거나 최소화 (처리 속도 우선) | 강력한 압축 (저장 공간 절약) |
| 접근 | 임의 접근 O(1) | 순차 접근 최적화 (Row Group 단위) |
| 직렬화 비용 | 거의 없음 (메모리 버퍼 직접 공유) | 디코딩 필요 |
| 행 그룹 | 없음 (단순 컬럼 버퍼) | Row Group (기본 128MB) |
| 사용 사례 | Pandas ↔ Spark 교환, 쿼리 엔진 내부 | S3/HDFS 저장, 배치 로드 |

현대 데이터 파이프라인은 두 포맷을 조합합니다: Parquet으로 저장 → Arrow로 메모리 로드 → 쿼리 처리.

## 주의사항과 성능 팁

**1. 작은 배치 크기 피하기:** Arrow의 SIMD 가속은 충분히 큰 배열에서 효과를 발휘합니다. RecordBatch 크기를 최소 65,536행 이상으로 설정하세요 (기본 권장: 1M행 또는 128MB).

**2. 딕셔너리 인코딩 활용:** 카디널리티가 낮은 문자열 컬럼(국가코드, 카테고리 등)은 `pa.dictionary(pa.int32(), pa.string())`으로 인코딩하면 메모리 사용량과 처리 속도 모두 개선됩니다.

**3. 스키마 진화:** Arrow와 Parquet은 하위 호환 스키마 진화를 지원하지만, 필드 삭제나 타입 변경은 위험합니다. Avro의 스키마 레지스트리 패턴을 Arrow와 함께 사용하는 것을 고려하세요.

**4. Python GIL 우회:** `pyarrow.compute`의 대부분 연산은 GIL을 릴리스하므로 멀티스레드에서 안전하게 사용할 수 있습니다. 단, Arrow Table을 Python 객체로 변환하는 작업(`to_pylist()`, `to_pandas()`)은 GIL을 획득합니다.

**5. CUDA Arrow:** NVIDIA의 cuDF(RAPIDS)는 GPU 메모리에 Arrow 호환 컬럼형 버퍼를 유지하고 GPU 커널로 처리합니다. CPU Arrow와 GPU Arrow 간 전송도 Arrow IPC를 통해 최소화할 수 있습니다.

## 참고 자료
- [Apache Arrow Columnar Format Specification](https://arrow.apache.org/docs/format/Columnar.html)
- [Apache Arrow 공식 문서](https://arrow.apache.org/docs/)
- [DuckDB: An in-process SQL OLAP database](https://duckdb.org/why_duckdb)
- [Apache Parquet Format Specification](https://parquet.apache.org/docs/file-format/)
