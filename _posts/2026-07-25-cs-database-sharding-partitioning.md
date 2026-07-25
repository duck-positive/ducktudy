---
layout: post
title: "데이터베이스 샤딩과 파티셔닝 완전 정복: 수십억 레코드를 다루는 수평 확장 전략"
date: 2026-07-25
categories: [cs, computer-science]
tags: [database, sharding, partitioning, horizontal-scaling, distributed-database, system-design]
---

수억 건의 사용자 데이터를 다루는 서비스를 구축한다고 상상해 보자. 단일 PostgreSQL 인스턴스는 하드웨어의 한계에 부딪히고, CPU, 메모리, 디스크 I/O 어디에서든 병목이 생긴다. 이 문제를 해결하는 가장 강력한 방법이 바로 **데이터베이스 샤딩(Sharding)** 과 **파티셔닝(Partitioning)** 이다.

## 파티셔닝 vs 샤딩: 무엇이 다른가?

두 개념은 자주 혼용되지만 중요한 차이가 있다.

**파티셔닝(Partitioning)**은 **단일 데이터베이스 인스턴스 내에서** 테이블을 논리적으로 또는 물리적으로 분할하는 기법이다. 물리적 저장소는 분리되지만 단일 DB 프로세스가 관리한다.

**샤딩(Sharding)**은 **여러 데이터베이스 인스턴스(샤드)에** 데이터를 분산 저장하는 수평 분할 기법이다. 각 샤드는 완전히 독립적인 DB 인스턴스다.

또한 두 가지 파티셔닝 방향이 있다:
- **수직 파티셔닝(Vertical Partitioning)**: 컬럼을 기준으로 테이블을 분리. 예: `users` 테이블에서 자주 쓰는 `id, name, email`과 거의 안 쓰는 `bio, settings` 분리
- **수평 파티셔닝(Horizontal Partitioning = Sharding)**: 행(Row)을 기준으로 분리. 같은 스키마를 유지하면서 데이터를 여러 노드에 분산

## 샤딩이 필요한 이유

단일 데이터베이스의 한계는 명확하다:

- **쓰기 처리량**: 단일 노드는 물리적 디스크 I/O 한계가 있다. PostgreSQL 기준으로도 초당 수만 건의 쓰기가 한계다.
- **저장 용량**: 수십 테라바이트 이상의 데이터는 단일 서버에서 관리가 어렵다.
- **읽기 확장**: 수평으로 레플리카를 추가해도 쓰기는 여전히 마스터 한 대에 집중된다.

Instagram은 사용자 수가 폭발적으로 증가하면서 PostgreSQL을 샤딩했고, Pinterest는 MySQL을 8192개의 가상 샤드로 운영한 사례가 있다.

## 3가지 핵심 샤딩 전략

### 1. 범위 기반 샤딩 (Range-Based Sharding)

샤드 키(shard key)의 값 범위에 따라 샤드를 결정한다.

```
샤드 0: user_id 1 ~ 1,000,000
샤드 1: user_id 1,000,001 ~ 2,000,000
샤드 2: user_id 2,000,001 ~ 3,000,000
```

**장점**: 범위 쿼리(`WHERE user_id BETWEEN 100 AND 500`)가 효율적이다.
**단점**: 특정 범위에 트래픽이 집중되는 **핫스팟(hotspot)** 이 발생할 수 있다. 신규 사용자는 항상 마지막 샤드에 집중된다.

### 2. 해시 기반 샤딩 (Hash-Based Sharding)

샤드 키에 해시 함수를 적용해 샤드를 결정한다.

```
shard_id = hash(user_id) % num_shards
```

**장점**: 데이터가 균등하게 분산되어 핫스팟이 거의 없다.
**단점**: 범위 쿼리 시 모든 샤드를 스캔해야 한다(scatter-gather). 샤드 수를 변경하면 거의 모든 키를 재배치해야 한다.

### 3. 디렉터리 기반 샤딩 (Directory-Based Sharding)

별도의 조회 테이블(lookup table)이 각 키가 어느 샤드에 있는지 관리한다.

```
lookup_table:
  user_id=1001 → shard_2
  user_id=1002 → shard_0
  user_id=1003 → shard_1
```

**장점**: 가장 유연하다. 데이터 이동 없이 재배치 가능.
**단점**: 조회 테이블 자체가 단일 장애점(SPOF)이 될 수 있다. 모든 쿼리에 추가 홉이 발생한다.

## Python 샤딩 라우터 구현 예제

```python
import hashlib
from dataclasses import dataclass
from typing import Optional


@dataclass
class Shard:
    shard_id: int
    host: str
    port: int
    data: dict  # 실제로는 DB 연결


class ShardRouter:
    """해시 기반 샤드 라우터 (Consistent Hashing 적용)"""

    def __init__(self, shards: list[Shard]):
        self.shards = shards
        self.num_shards = len(shards)
        # 가상 노드(virtual node) 링 구성 - 100개씩
        self.ring: list[tuple[int, Shard]] = []
        for shard in shards:
            for i in range(100):
                key = f"{shard.shard_id}:vnode:{i}"
                hash_val = self._hash(key)
                self.ring.append((hash_val, shard))
        self.ring.sort(key=lambda x: x[0])

    def _hash(self, key: str) -> int:
        return int(hashlib.md5(key.encode()).hexdigest(), 16)

    def get_shard(self, shard_key: str) -> Shard:
        """일관성 해싱으로 샤드 결정"""
        hash_val = self._hash(shard_key)
        for ring_hash, shard in self.ring:
            if hash_val <= ring_hash:
                return shard
        return self.ring[0][1]  # 링을 돌아서 첫 번째

    def write(self, user_id: int, data: dict) -> bool:
        shard = self.get_shard(str(user_id))
        shard.data[user_id] = data
        print(f"  user_id={user_id} → Shard {shard.shard_id} ({shard.host})")
        return True

    def read(self, user_id: int) -> Optional[dict]:
        shard = self.get_shard(str(user_id))
        return shard.data.get(user_id)

    def range_query(self, start_id: int, end_id: int) -> list[dict]:
        """범위 쿼리: 모든 샤드에 scatter-gather 필요"""
        results = []
        for shard in self.shards:
            for uid, data in shard.data.items():
                if start_id <= uid <= end_id:
                    results.append(data)
        return sorted(results, key=lambda x: x.get('user_id', 0))


# 시뮬레이션
shards = [
    Shard(0, "db-0.example.com", 5432, {}),
    Shard(1, "db-1.example.com", 5432, {}),
    Shard(2, "db-2.example.com", 5432, {}),
]
router = ShardRouter(shards)

print("=== 샤드 라우팅 결과 ===")
for user_id in [1001, 1002, 1003, 5000, 9999, 100000]:
    router.write(user_id, {"user_id": user_id, "name": f"User{user_id}"})

print(f"\n=== user_id=1001 읽기: {router.read(1001)}")
print(f"\n=== 범위 쿼리 (1001~5000): {len(router.range_query(1001, 5000))}건")

# 데이터 분포 확인
print("\n=== 샤드별 데이터 분포 ===")
for shard in shards:
    print(f"  Shard {shard.shard_id}: {len(shard.data)}건")
```

## PostgreSQL 테이블 파티셔닝 실전 예제

단일 DB 내 파티셔닝은 수십억 건의 시계열 데이터를 관리할 때 특히 효과적이다.

```sql
-- 범위 파티셔닝: 월별 주문 데이터
CREATE TABLE orders (
    order_id    BIGSERIAL,
    user_id     BIGINT NOT NULL,
    amount      DECIMAL(10, 2),
    created_at  TIMESTAMPTZ NOT NULL,
    status      VARCHAR(20)
) PARTITION BY RANGE (created_at);

-- 월별 파티션 생성
CREATE TABLE orders_2026_01 PARTITION OF orders
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

CREATE TABLE orders_2026_02 PARTITION OF orders
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');

CREATE TABLE orders_2026_07 PARTITION OF orders
    FOR VALUES FROM ('2026-07-01') TO ('2026-08-01');

-- 기본 파티션 (위 범위에 속하지 않는 데이터용)
CREATE TABLE orders_default PARTITION OF orders DEFAULT;

-- 파티션별 인덱스 (자동으로 파티션에 적용됨)
CREATE INDEX ON orders (user_id, created_at);
CREATE INDEX ON orders (created_at);

-- 쿼리 (PostgreSQL이 자동으로 관련 파티션만 스캔)
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders
WHERE created_at BETWEEN '2026-07-01' AND '2026-07-31'
  AND user_id = 12345;
-- → orders_2026_07 파티션만 스캔 (Partition Pruning)

-- 해시 파티셔닝: 사용자 데이터 균등 분산
CREATE TABLE users_sharded (
    user_id   BIGINT NOT NULL,
    username  VARCHAR(50),
    email     VARCHAR(100)
) PARTITION BY HASH (user_id);

CREATE TABLE users_shard_0 PARTITION OF users_sharded
    FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE users_shard_1 PARTITION OF users_sharded
    FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE users_shard_2 PARTITION OF users_sharded
    FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE users_shard_3 PARTITION OF users_sharded
    FOR VALUES WITH (MODULUS 4, REMAINDER 3);

-- 파티션 상태 조회
SELECT
    child.relname AS partition_name,
    pg_size_pretty(pg_relation_size(child.oid)) AS size,
    pg_stat_user_tables.n_live_tup AS row_count
FROM pg_inherits
JOIN pg_class parent ON pg_inherits.inhparent = parent.oid
JOIN pg_class child ON pg_inherits.inhrelid = child.oid
LEFT JOIN pg_stat_user_tables ON child.relname = pg_stat_user_tables.relname
WHERE parent.relname = 'orders'
ORDER BY child.relname;
```

## 샤딩의 핵심 난제들

### 1. 크로스 샤드 쿼리

여러 샤드에 걸친 `JOIN`, `GROUP BY`, `ORDER BY`는 매우 비용이 크다. 각 샤드에서 부분 결과를 가져와 애플리케이션 레이어에서 병합해야 한다.

**해결책**: 자주 함께 조회되는 데이터는 같은 샤드에 배치하는 **데이터 지역성(data locality)** 을 고려해 샤드 키를 설계한다. 예를 들어 소셜 피드 서비스라면 `user_id` 대신 `tenant_id`나 `region`을 샤드 키로 선택할 수 있다.

### 2. 분산 트랜잭션

A 샤드에서 돈을 빼고, B 샤드에서 돈을 넣는 트랜잭션은 단일 DB처럼 ACID를 보장하기 어렵다. **2PC(Two-Phase Commit)** 또는 **Saga 패턴**이 필요하다.

### 3. 리샤딩(Re-sharding)

샤드 수를 변경할 때 데이터를 재배치하는 과정이 서비스 중단 없이 진행되어야 한다. **일관성 해싱(Consistent Hashing)** 이 이 문제를 크게 완화한다. 또한 **가상 샤드(virtual shard)** 를 사용해 물리 샤드보다 훨씬 많은 가상 단위를 만들어 두면 리샤딩 시 데이터 이동량을 최소화할 수 있다.

### 4. 핫스팟과 불균형

유명 사용자(celebrity effect)나 특정 시간대에 데이터가 집중될 수 있다. 이때 **샤드 키 설계**가 핵심이다. Twitter에서 유명인 트윗을 별도로 캐싱하거나, 타임스탬프를 그대로 샤드 키로 쓰지 않고 `(timestamp, random_suffix)` 조합을 사용하는 것이 좋은 예다.

## 주의사항 및 팁

- **샤딩은 마지막 수단이다**: 먼저 인덱스 최적화, 쿼리 튜닝, 읽기 레플리카, 캐싱 레이어를 도입하라. 샤딩은 운영 복잡도를 크게 높인다.
- **샤드 키 선택이 전부다**: 나중에 바꾸기 매우 어렵다. 접근 패턴을 충분히 분석하고 결정하라.
- **PlanetScale, Vitess, Citus** 같은 미들웨어를 활용하면 애플리케이션 코드 변경 없이 MySQL/PostgreSQL을 샤딩할 수 있다.
- **글로벌 고유 ID**가 필요하다: 여러 샤드에서 동일한 auto-increment를 쓸 수 없다. Snowflake ID, UUID v7, ULID 등을 사용하라.

## 참고 자료
- [Sharding Pattern - Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/patterns/sharding)
- [Database Sharding - Wikipedia](https://en.wikipedia.org/wiki/Shard_(database_architecture))
- [PostgreSQL Table Partitioning - Official Docs](https://www.postgresql.org/docs/current/ddl-partitioning.html)
- [Database Sharding Explained - ProxySQL Blog](https://proxysql.com/blog/database-sharding/)
