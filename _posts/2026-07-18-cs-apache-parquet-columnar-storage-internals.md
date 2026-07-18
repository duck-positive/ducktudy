---
layout: post
title: "Apache Parquet 내부 구조: 컬럼형 스토리지가 빅데이터를 10배 빠르게 읽는 이유"
date: 2026-07-18
categories: [cs, computer-science]
tags: [parquet, columnar-storage, big-data, encoding, compression, rle, dictionary-encoding, predicate-pushdown, olap]
---

데이터 엔지니어링 파이프라인을 구축하다 보면 CSV 대신 Parquet 형식을 사용하라는 말을 자주 듣는다. Parquet은 단순히 "압축이 잘 되는 파일 형식"이 아니다. 데이터를 읽는 방식 자체를 바꾸는 **컬럼 지향(Columnar)** 저장 구조를 채택해, 분석 쿼리 성능을 수십 배까지 향상시킨다. 이 글에서는 Parquet 파일의 내부 구조, 인코딩 기법, 그리고 쿼리 엔진이 어떻게 이를 활용하는지 구체적으로 살펴본다.

---

## 개념 설명

### 행 지향 vs 컬럼 지향

전통적인 CSV, JSON, RDBMS 테이블은 **행(Row) 지향**으로 데이터를 저장한다.

```
행 지향 저장:
[user_id=1, name="Alice", age=30, country="KR"]
[user_id=2, name="Bob",   age=25, country="US"]
[user_id=3, name="Carol", age=28, country="JP"]
```

`SELECT AVG(age) FROM users`를 실행하면, `name`과 `country`는 전혀 필요 없음에도 모든 행을 읽어야 한다.

**컬럼 지향** 저장에서는 같은 컬럼의 값들이 연속적으로 배치된다.

```
컬럼 지향 저장:
user_id: [1, 2, 3, ...]
name:    ["Alice", "Bob", "Carol", ...]
age:     [30, 25, 28, ...]
country: ["KR", "US", "JP", ...]
```

`AVG(age)`를 계산하려면 `age` 컬럼만 읽으면 된다. 컬럼이 수십 개, 행이 수억 개라면 I/O 절감 효과는 압도적이다.

### Parquet 파일의 3계층 구조

Apache Parquet 파일은 계층적으로 구성된다.

```
┌─────────────────────────────────┐
│  Magic Number (4 bytes: "PAR1") │
├─────────────────────────────────┤
│  Row Group 1                    │
│  ├── Column Chunk (col A)       │
│  │   ├── Page 1 (data)          │
│  │   ├── Page 2 (data)          │
│  │   └── Dictionary Page        │
│  └── Column Chunk (col B)       │
│      └── Page 1 (data)          │
├─────────────────────────────────┤
│  Row Group 2                    │
│  └── ...                        │
├─────────────────────────────────┤
│  Footer (File Metadata)         │
│  ├── Schema                     │
│  ├── Row Group 메타데이터       │
│  └── Statistics (min/max/null)  │
├─────────────────────────────────┤
│  Footer Length (4 bytes)        │
│  Magic Number (4 bytes: "PAR1") │
└─────────────────────────────────┘
```

- **Row Group**: 데이터의 수평 파티션. 기본 128MB ~ 1GB. 각 Row Group은 독립적으로 읽을 수 있다.
- **Column Chunk**: Row Group 내에서 하나의 컬럼 데이터. 연속된 디스크 위치에 저장된다.
- **Page**: Column Chunk의 최소 단위. 압축과 인코딩이 적용되는 레벨. 보통 1MB.
- **Footer**: 파일 끝에 위치. 스키마, 각 컬럼의 통계(min, max, null count), Row Group 오프셋 정보 포함.

---

## 왜 필요한가

### OLAP 워크로드의 특성

OLAP(Online Analytical Processing) 쿼리는 아래 특성을 가진다.

- **컬럼 선택적 접근**: 수십~수백 개 컬럼 중 몇 개만 읽음
- **집계 연산**: SUM, AVG, COUNT, GROUP BY
- **대량 스캔**: 수십억 행을 읽되, 특정 조건으로 필터링

이런 패턴에서 행 지향 포맷의 문제는 명확하다. 필요 없는 컬럼까지 모두 읽어야 하고, 압축 효율도 낮다(같은 타입의 연속 값이 아니므로). Parquet은 이 두 가지를 모두 해결한다.

### 동일 타입 데이터의 압축 효율

같은 컬럼의 값들은 동일한 타입이고, 실제 데이터에서는 **카디널리티가 낮은(반복이 많은)** 경우가 많다.

```
country 컬럼: ["KR","KR","KR","US","KR","JP","KR","KR","US","KR"]
age 컬럼:     [25, 25, 26, 25, 30, 30, 30, 30, 25, 26]
```

이런 패턴은 **RLE, 사전 인코딩** 등으로 극적인 압축이 가능하다.

---

## 실제 구현 예제

### 예제 1: PyArrow로 Parquet 파일 생성 및 내부 구조 탐색 (Python)

```python
import pyarrow as pa
import pyarrow.parquet as pq
import numpy as np

# 샘플 데이터 생성 (낮은 카디널리티 컬럼 포함)
n = 1_000_000
data = {
    "user_id": pa.array(range(n), type=pa.int64()),
    "country": pa.array(
        np.random.choice(["KR", "US", "JP", "CN", "DE"], n),
        type=pa.string()
    ),
    "age": pa.array(np.random.randint(18, 80, n), type=pa.int32()),
    "score": pa.array(np.random.random(n).astype(np.float32)),
    "active": pa.array(np.random.choice([True, False], n), type=pa.bool_()),
}
table = pa.table(data)

# 1) 기본 Parquet 저장 (Snappy 압축)
pq.write_table(
    table,
    "users.parquet",
    compression="snappy",
    row_group_size=200_000,     # Row Group당 행 수
    use_dictionary=True,         # Dictionary Encoding 활성화
    write_statistics=True,       # 통계 정보 저장
)

# 2) ZSTD 압축으로 저장 (더 높은 압축률)
pq.write_table(
    table,
    "users_zstd.parquet",
    compression="zstd",
    compression_level=3,
)

# 3) 파일 내부 메타데이터 탐색
pf = pq.ParquetFile("users.parquet")
meta = pf.metadata

print(f"총 행 수: {meta.num_rows:,}")
print(f"Row Group 수: {meta.num_row_groups}")
print(f"컬럼 수: {meta.num_columns}")
print()

# Row Group 0의 통계 출력
rg = meta.row_group(0)
for i in range(meta.num_columns):
    col = rg.column(i)
    stats = col.statistics
    print(f"[{col.path_in_schema}]")
    print(f"  압축 크기: {col.total_compressed_size:,} bytes")
    print(f"  비압축 크기: {col.total_uncompressed_size:,} bytes")
    if stats:
        print(f"  min: {stats.min}, max: {stats.max}")
        print(f"  null count: {stats.null_count}")
    print()

# 4) Predicate Pushdown 적용 읽기 (KR 국가만, age 컬럼만)
import pyarrow.compute as pc
filtered = pq.read_table(
    "users.parquet",
    columns=["age", "country"],
    filters=[("country", "=", "KR")],  # 필터 조건을 파일 레벨에서 적용
)
print(f"KR 사용자 수: {len(filtered):,}")
print(f"KR 평균 나이: {pc.mean(filtered['age']).as_py():.1f}")
```

### 예제 2: Parquet 인코딩 기법 시각화 및 직접 구현 (Python)

Parquet의 핵심 인코딩인 **RLE/Bit-Packing**과 **Dictionary Encoding**을 직접 구현해보자.

```python
import struct
from typing import List, Any

class RLEEncoder:
    """
    Run-Length Encoding: 연속 반복 값을 (횟수, 값) 쌍으로 압축
    예: [1,1,1,2,2,3] -> [(3,1),(2,2),(1,3)]
    """
    @staticmethod
    def encode(values: List[int]) -> bytes:
        if not values:
            return b""
        result = []
        i = 0
        while i < len(values):
            run_val = values[i]
            run_len = 1
            while i + run_len < len(values) and values[i + run_len] == run_val:
                run_len += 1
            # (run_length << 1 | 0)은 RLE 모드를 표시 (LSB=0)
            result.append(struct.pack("<IH", (run_len << 1), run_val))
            i += run_len
        return b"".join(result)
    
    @staticmethod
    def decode(data: bytes) -> List[int]:
        result = []
        offset = 0
        while offset < len(data):
            header, val = struct.unpack_from("<IH", data, offset)
            run_len = header >> 1
            result.extend([val] * run_len)
            offset += 6
        return result


class DictionaryEncoder:
    """
    Dictionary Encoding: 고유 값들을 딕셔너리로, 실제 값은 정수 인덱스로 저장
    낮은 카디널리티 컬럼에서 극적인 압축 효과
    """
    def encode(self, values: List[Any]):
        # 딕셔너리 구축
        dictionary = []
        dict_map = {}
        for v in values:
            if v not in dict_map:
                dict_map[v] = len(dictionary)
                dictionary.append(v)
        # 값을 인덱스로 변환
        indices = [dict_map[v] for v in values]
        return dictionary, indices
    
    def decode(self, dictionary: List[Any], indices: List[int]) -> List[Any]:
        return [dictionary[i] for i in indices]


# 시연: country 컬럼 압축 비율 비교
import random
countries = ["KR", "US", "JP", "CN", "DE"]
country_data = [random.choice(countries) for _ in range(100_000)]

# 원본 크기 (UTF-8)
raw_size = sum(len(c.encode()) for c in country_data)
print(f"원본 크기: {raw_size:,} bytes")

# Dictionary Encoding 후 크기
enc = DictionaryEncoder()
dictionary, indices = enc.encode(country_data)
dict_size = sum(len(d.encode()) for d in dictionary)  # 딕셔너리 자체 크기
# 인덱스는 2비트(5개 값이면 3비트, 여기선 1바이트로 단순화)
indices_size = len(indices) * 1
encoded_size = dict_size + indices_size
print(f"Dictionary Encoding 후: {encoded_size:,} bytes")
print(f"압축률: {raw_size / encoded_size:.1f}x")

# RLE로 인덱스 추가 압축 (정렬된 데이터가 아니면 효과 없음)
sorted_indices = sorted(indices)  # 정렬된 경우 시뮬레이션
rle_encoded = RLEEncoder.encode(sorted_indices)
print(f"RLE + Dictionary 후: {len(rle_encoded):,} bytes")
print(f"최종 압축률: {raw_size / len(rle_encoded):.1f}x")

# 복원 검증
decoded_indices = RLEEncoder.decode(rle_encoded)
assert decoded_indices == sorted_indices, "RLE 복원 실패"
print("RLE 복원 검증 완료")
```

---

## Predicate Pushdown와 Column Pruning

Parquet의 Footer에는 각 Row Group, 각 Column Chunk의 **통계 정보(min/max/null_count)**가 저장된다. 쿼리 엔진은 이 정보를 활용해 데이터를 아예 읽지 않을 수 있다.

```
예: WHERE age > 60 AND country = 'KR'

Footer 검사:
- Row Group 1: age 범위 [18, 55] → age > 60 조건 불만족 → 건너뜀
- Row Group 2: age 범위 [50, 80], country에 'KR' 없음 → 건너뜀  
- Row Group 3: age 범위 [55, 79], country에 'KR' 있음 → 읽어야 함

→ Row Group 3만 읽어서 실제 필터 적용
```

이 메커니즘을 **Row Group Filtering** 또는 **Predicate Pushdown**이라 한다. Spark, Hive, DuckDB, Presto 등의 엔진이 모두 이를 활용한다.

---

## 중첩 데이터 표현: Dremel 알고리즘

Parquet은 Google Dremel 논문에서 영감받아 **Repetition Level**과 **Definition Level**로 중첩/반복 구조(Nested Schema)를 컬럼 형식으로 표현한다.

```
스키마:
message user {
  required string name;
  repeated group address {
    optional string city;
    required string country;
  }
}

레코드 1: {name: "Alice", address: [{city: "Seoul", country: "KR"}]}
레코드 2: {name: "Bob", address: [{country: "US"}, {city: "London", country: "GB"}]}

address.city 컬럼 (Repetition, Definition, Value):
R=0, D=2, "Seoul"    # 새 레코드, address 존재, city 존재
R=0, D=1, NULL       # 새 레코드, address 존재, city 없음
R=1, D=2, "London"   # 같은 레코드, 두 번째 address, city 존재
```

---

## 주의사항 및 팁

### Row Group 크기 선택

- **너무 작으면**: Footer 메타데이터 오버헤드 비율 증가, 병렬 처리 효율 저하
- **너무 크면**: 메모리 압박, 필요 없는 Row Group도 많이 읽힘
- **권장**: Spark의 경우 128MB ~ 1GB, Pandas의 경우 수십만 ~ 수백만 행

### 파티셔닝과 Parquet의 조합

Hive 스타일 파티셔닝(`country=KR/date=2026-07-18/part-0.parquet`)과 Parquet의 Row Group Filtering을 함께 사용하면 쿼리 엔진이 디렉토리 레벨에서 먼저 필터링한 후 파일 레벨에서 Row Group을 선택한다.

### 스키마 진화(Schema Evolution)

Parquet은 컬럼 추가, 타입 업캐스팅(int32 → int64) 등의 스키마 변화를 지원하지만, 컬럼 이름 변경이나 타입 다운캐스팅은 지원하지 않는다. 프로덕션에서는 스키마 레지스트리를 함께 운용하는 것이 좋다.

### Dictionary Encoding 비활성화 대상

카디널리티가 높은 컬럼(UUID, 타임스탬프, 랜덤 해시 등)에 Dictionary Encoding을 적용하면 딕셔너리가 커져 오히려 overhead가 발생한다. `use_dictionary=False` 또는 컬럼별로 설정하는 것이 성능에 유리하다.

---

## 참고 자료
- [Apache Parquet 공식 사이트](https://parquet.apache.org/)
- [Apache Parquet Format Specification - GitHub](https://github.com/apache/parquet-format/)
- [What is Parquet? - Databricks](https://www.databricks.com/blog/what-is-parquet)
- [Columnar storage formats: Parquet, ORC, and Arrow explained - ClickHouse](https://clickhouse.com/resources/engineering/columnar-storage-formats)
