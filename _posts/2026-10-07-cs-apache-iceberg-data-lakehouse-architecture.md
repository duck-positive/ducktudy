---
layout: post
title: "Apache Iceberg와 데이터 레이크하우스 아키텍처: 오픈 테이블 포맷의 내부 구조 완전 정복"
date: 2026-10-07
categories: [cs, computer-science]
tags: [iceberg, data-lakehouse, delta-lake, parquet, snapshot-isolation, schema-evolution, ACID, big-data]
---

## 데이터 레이크의 한계와 레이크하우스의 등장

2010년대 중반까지 빅데이터 처리의 표준은 단순했습니다. **데이터 웨어하우스(Data Warehouse)**는 정형 데이터를 다루며 강력한 SQL을 제공했지만 비용이 비쌌고, **데이터 레이크(Data Lake)**는 S3 같은 저렴한 오브젝트 스토리지에 모든 형식의 데이터를 모아뒀지만 ACID 트랜잭션이 없어 "데이터 늪(Data Swamp)"으로 전락하는 경우가 많았습니다.

이 두 세계의 장점을 합친 개념이 **데이터 레이크하우스(Data Lakehouse)**입니다. 핵심은 **오픈 테이블 포맷(Open Table Format)**입니다. Apache Iceberg, Delta Lake, Apache Hudi가 대표적이며, 이들은 오브젝트 스토리지 위에서 ACID 트랜잭션, 타임 트래블, 스키마 진화를 가능하게 합니다.

이 글에서는 Iceberg를 중심으로 그 내부 구조를 파헤칩니다.

## 왜 Apache Iceberg인가?

Iceberg는 Netflix가 내부 문제를 해결하기 위해 개발하여 2018년 오픈소스로 공개했습니다. 기존 Hive 메타스토어 기반의 파티션 테이블이 가진 문제들을 근본적으로 해결했습니다.

### Hive 파티션 테이블의 한계

```sql
-- Hive 방식: 파티션 = 디렉토리 구조
/warehouse/sales/year=2024/month=01/day=01/part-00000.parquet
/warehouse/sales/year=2024/month=01/day=02/part-00000.parquet
```

이 구조의 문제:
1. **파티션 값이 경로에 노출**: 스키마 변경 시 전체 데이터 재작성 필요
2. **파일 목록 조회 비용**: 수백만 개 파티션이 있으면 `LIST` 작업만으로 몇 분 소요
3. **ACID 없음**: 두 작업이 동시에 같은 파티션을 수정하면 데이터가 손상될 수 있음
4. **통계 없음**: 파일마다 어떤 데이터가 있는지 몰라 풀스캔 필수

## Iceberg 메타데이터 트리 구조

Iceberg의 핵심은 4계층으로 이루어진 **메타데이터 트리**입니다.

```
Catalog (테이블 → 현재 메타데이터 파일 포인터)
    │
    ▼
Table Metadata File (JSON)
  ├── schema 목록
  ├── partition spec 목록
  ├── sort order
  └── snapshots 목록
           │
           ▼
       Snapshot (스냅샷)
         ├── snapshot-id
         ├── parent-snapshot-id
         ├── timestamp-ms
         └── manifest-list 파일 경로
                   │
                   ▼
             Manifest List (Avro)
               ├── manifest 파일 경로들
               ├── 각 manifest의 파티션 범위
               └── 추가/삭제된 파일 수
                          │
                          ▼
                   Manifest File (Avro)
                     ├── 데이터 파일 경로
                     ├── 파일 포맷 (Parquet/ORC/Avro)
                     ├── 파일 크기
                     ├── 레코드 수
                     └── 컬럼별 통계 (min, max, null_count)
                                │
                                ▼
                         Data File (Parquet)
```

이 트리 구조가 Iceberg의 모든 강점을 가능하게 합니다.

### 스냅샷 기반 ACID 트랜잭션

```python
# PyIceberg를 사용한 기본 작업 예시
from pyiceberg.catalog import load_catalog
from pyiceberg.schema import Schema
from pyiceberg.types import NestedField, StringType, LongType, TimestampType
import pyarrow as pa

# 카탈로그 연결 (REST, Hive, Glue 등)
catalog = load_catalog("rest", **{
    "uri": "http://localhost:8181",
    "s3.endpoint": "http://localhost:9000",
})

# 스키마 정의
schema = Schema(
    NestedField(1, "order_id", LongType(), required=True),
    NestedField(2, "customer_id", LongType(), required=False),
    NestedField(3, "amount", LongType(), required=False),
    NestedField(4, "created_at", TimestampType(), required=False),
)

# 테이블 생성
table = catalog.create_table(
    identifier="analytics.orders",
    schema=schema,
    location="s3://my-bucket/warehouse/analytics/orders",
)

# 데이터 추가 — 원자적 커밋
df = pa.table({
    "order_id": [1, 2, 3],
    "customer_id": [101, 102, 103],
    "amount": [5000, 12000, 3500],
})
table.append(df)

# 이 시점에 새 스냅샷이 원자적으로 커밋됨
# 읽는 측은 커밋 완료 전 스냅샷을 봄 (격리 보장)
print(f"현재 스냅샷 ID: {table.current_snapshot().snapshot_id}")
```

커밋 과정:
1. 새 데이터 파일을 S3에 업로드
2. 새 Manifest 파일 작성 (데이터 파일 목록 + 통계)
3. 새 Manifest List 파일 작성
4. 새 Table Metadata JSON 작성 (새 스냅샷 포함)
5. **카탈로그에서 포인터를 원자적으로 교체** (낙관적 잠금)

다른 읽기 작업은 포인터 교체 전후에 따라 구 스냅샷 또는 신 스냅샷을 보며, 중간 상태는 절대 보이지 않습니다.

## 타임 트래블: 과거 스냅샷 조회

```python
# PySpark를 사용한 Iceberg 타임 트래블 예시
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .config("spark.sql.extensions",
            "org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions") \
    .config("spark.sql.catalog.spark_catalog",
            "org.apache.iceberg.spark.SparkSessionCatalog") \
    .getOrCreate()

# 특정 타임스탬프 시점 조회
df_past = spark.read.option(
    "as-of-timestamp", "2026-10-01T00:00:00"
).table("analytics.orders")

# 특정 스냅샷 ID로 조회
df_snapshot = spark.read.option(
    "snapshot-id", "3051729675574597004"
).table("analytics.orders")

# 두 스냅샷 간의 변경 이력 조회
changes = spark.sql("""
    SELECT *
    FROM analytics.orders.changes
    WHERE _change_type IN ('INSERT', 'DELETE')
    AND _commit_snapshot_id BETWEEN 1000 AND 2000
""")

# 스냅샷 이력 조회
spark.sql("SELECT * FROM analytics.orders.snapshots").show()
```

타임 트래블이 가능한 이유는 과거 스냅샷이 메타데이터에 여전히 남아있기 때문입니다. 보존 기간 이후에는 `EXPIRE SNAPSHOTS` 명령으로 오래된 메타데이터와 고아 파일을 정리합니다.

## 스키마 진화: 데이터 재작성 없는 안전한 변경

Iceberg의 스키마 진화는 **필드 ID** 시스템 덕분에 안전합니다. 각 컬럼에는 이름이 아닌 고유한 정수 ID가 부여되어 있어, 컬럼을 이름 변경하거나 재정렬해도 기존 파일을 건드리지 않습니다.

```sql
-- 안전하게 컬럼 추가 (기존 파일 재작성 없음)
ALTER TABLE analytics.orders ADD COLUMN discount BIGINT;

-- 컬럼 이름 변경 (내부 ID는 유지됨)
ALTER TABLE analytics.orders RENAME COLUMN amount TO total_amount;

-- 컬럼 타입 승격 (Int → Long 등 안전한 변환)
ALTER TABLE analytics.orders ALTER COLUMN customer_id TYPE BIGINT;

-- 파티션 스펙 변경 (새 데이터에만 적용)
ALTER TABLE analytics.orders
REPLACE PARTITION FIELD created_at WITH days(created_at);
```

Hive와 달리 파티션 스펙을 바꿔도 기존 파티션의 데이터는 그대로 유지됩니다. Iceberg는 각 데이터 파일이 어떤 파티션 스펙으로 만들어졌는지 추적하여 읽기 시점에 올바르게 해석합니다.

## 쿼리 최적화: 메타데이터 기반 파일 프루닝

Iceberg 쿼리 최적화기는 Manifest File에 저장된 컬럼별 통계를 활용합니다.

```
쿼리: SELECT * FROM orders WHERE amount > 100000

최적화 과정:
1. Table Metadata → 현재 Snapshot → Manifest List 읽기
2. Manifest List에서 파티션 범위로 1차 필터링 (불필요 Manifest 제외)
3. Manifest File에서 amount 컬럼의 max 통계 확인
   - max_amount < 100000인 데이터 파일은 완전히 건너뜀 (파일 열지 않음)
4. 살아남은 파일만 실제 읽기
```

데이터 파일 수가 100만 개라도 통계 기반 프루닝으로 실제로는 수십 개 파일만 읽는 경우가 흔합니다.

## 낙관적 동시성 제어와 충돌 해결

여러 쓰기 작업이 동시에 일어나면 Iceberg는 낙관적 잠금으로 처리합니다.

```
Writer A                   Writer B
  |                           |
  | 스냅샷 S1 기반으로 읽기     | 스냅샷 S1 기반으로 읽기
  |                           |
  | 새 파일 생성               | 새 파일 생성
  |                           |
  | S2 커밋 시도               |
  | (성공 → 카탈로그 S1→S2)    |
  |                           |
                              | S2 기반으로 S3 커밋 시도
                              | (실패 → S2 기반으로 재시도)
                              | S3 커밋 성공
```

충돌 시 재시도 전략은 작업 유형에 따라 다릅니다. `INSERT`는 항상 재시도 가능하지만 `UPDATE`(merge-on-read)는 충돌한 행을 다시 확인해야 합니다.

## 주의사항과 실무 팁

**1. 작은 파일 문제를 적극적으로 관리하라**
스트리밍 쓰기는 작은 파일을 많이 생성합니다. 주기적으로 `OPTIMIZE TABLE`(Spark) 또는 `rewrite_data_files` 프로시저를 실행하여 파일을 병합하세요.

**2. 메타데이터 파일도 축적된다**
스냅샷이 많아지면 메타데이터 파일도 커집니다. `expire_snapshots`와 `remove_orphan_files`를 주기적으로 실행하는 유지보수 작업을 스케줄링하세요.

**3. 카탈로그 선택이 중요하다**
REST Catalog(Iceberg REST 스펙), AWS Glue, Apache Polaris 중 운영 환경에 맞는 카탈로그를 선택하세요. 카탈로그가 포인터를 원자적으로 업데이트하므로 카탈로그 구현의 신뢰성이 ACID 보장의 핵심입니다.

**4. Iceberg vs Delta Lake vs Hudi**
세 포맷 모두 ACID와 타임 트래블을 제공하지만 세부 구현이 다릅니다. Delta Lake는 Databricks 생태계와 더 깊이 통합되고, Hudi는 레코드 수준 upsert에 강점이 있습니다. Iceberg는 멀티 엔진 호환성(Spark, Flink, Trino, Presto, BigQuery, Snowflake)이 가장 뛰어납니다.

## 참고 자료
- [Apache Iceberg Table Spec](https://iceberg.apache.org/spec/)
- [PyIceberg 공식 문서](https://py.iceberg.apache.org/)
- [Iceberg: A Modern Table Format for Huge Analytic Datasets (SIGMOD 2020)](https://dl.acm.org/doi/10.1145/3299869.3314034)
- [Apache Iceberg - GitHub](https://github.com/apache/iceberg)
