---
layout: post
title: "PostgreSQL 확장 인덱스 심화: GIN, GiST, BRIN으로 B-Tree를 넘어서"
date: 2026-10-25
categories: [cs, computer-science]
tags: [postgresql, database, index, gin, gist, brin, full-text-search, spatial-index]
---

## 개요

PostgreSQL이 강력한 이유 중 하나는 B-Tree를 넘어서는 다양한 인덱스 타입을 제공하기 때문이다. **GIN(Generalized Inverted iNdex)**, **GiST(Generalized Search Tree)**, **BRIN(Block Range INdex)**은 각각 매우 다른 내부 구조와 적합한 사용 사례를 가진다. 이 글에서는 세 인덱스의 내부 구조를 깊이 파고들고, 어떤 상황에서 무엇을 선택해야 하는지를 코드 예제와 함께 설명한다.

---

## B-Tree의 한계

B-Tree는 등호 및 범위 비교에 탁월하지만, 다음 상황에서는 적합하지 않다:

- **복합 값**: 배열, JSONB, tsvector처럼 하나의 컬럼이 여러 값을 포함할 때
- **기하학적 데이터**: 포함 여부, 교차 여부, 최근접 검색
- **순차 데이터**: 타임스탬프처럼 물리적 저장 순서와 논리적 순서가 일치하는 대형 테이블

이 세 가지 문제를 각각 GIN, GiST, BRIN이 해결한다.

---

## GIN (Generalized Inverted iNdex)

### 내부 구조

GIN은 **역인덱스(Inverted Index)**다. 각 컬럼 값을 "구성 요소(items)"로 분해하고, 각 item에 대해 그 item이 등장하는 행 ID(TID) 목록을 저장한다.

```
원본 데이터:
  row 1: {python, database, index}
  row 2: {python, algorithm}
  row 3: {database, transaction}

GIN 구조:
  "algorithm"  → [row 2]
  "database"   → [row 1, row 3]
  "index"      → [row 1]
  "python"     → [row 1, row 2]
  "transaction"→ [row 3]
```

TID 목록이 짧으면 **posting list**(압축된 배열), 길면 **posting tree**(B-Tree)로 저장된다.

### Fast Update 메커니즘

GIN에 행을 삽입할 때마다 모든 item을 인덱스에 즉시 반영하면 매우 느리다. PostgreSQL의 GIN은 **Fast Update** 기능을 기본으로 활성화한다. 새 삽입은 메인 GIN 구조 대신 **pending list**에 누적되고, 이후 `gin_pending_list_limit`(기본 4MB)을 초과하거나 VACUUM이 실행될 때 한꺼번에 병합된다.

```sql
-- Fast Update ON/OFF 제어
CREATE INDEX idx_tags ON posts USING gin(tags)
WITH (fastupdate = on);  -- 기본값

-- 대량 삽입 후 pending list 수동 플러시
SELECT gin_clean_pending_list('idx_tags'::regclass);
```

### 실전 예제: 전문 검색 (Full-Text Search)

```sql
-- 테이블 및 tsvector 컬럼 생성
CREATE TABLE articles (
    id SERIAL PRIMARY KEY,
    title TEXT,
    body TEXT,
    search_vector TSVECTOR
);

-- tsvector 자동 업데이트 트리거
CREATE FUNCTION update_search_vector() RETURNS TRIGGER AS $$
BEGIN
    NEW.search_vector :=
        setweight(to_tsvector('korean', COALESCE(NEW.title, '')), 'A') ||
        setweight(to_tsvector('korean', COALESCE(NEW.body, '')), 'B');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_search_vector
BEFORE INSERT OR UPDATE ON articles
FOR EACH ROW EXECUTE FUNCTION update_search_vector();

-- GIN 인덱스 생성
CREATE INDEX idx_articles_fts ON articles USING gin(search_vector);

-- 전문 검색 쿼리 (인덱스 활용)
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, title
FROM articles
WHERE search_vector @@ plainto_tsquery('korean', '데이터베이스 인덱스');
```

### JSONB와 배열에 GIN 적용

```sql
-- JSONB 인덱스
CREATE TABLE events (
    id SERIAL,
    metadata JSONB
);

-- JSONB 전체 키/값에 GIN
CREATE INDEX idx_metadata ON events USING gin(metadata);

-- 중첩 키 경로 조회
SELECT * FROM events
WHERE metadata @> '{"user": {"role": "admin"}}';

-- 배열 컬럼
CREATE TABLE products (
    id SERIAL,
    tags TEXT[]
);

CREATE INDEX idx_product_tags ON products USING gin(tags);

-- 배열 포함 여부 검색
SELECT * FROM products
WHERE tags @> ARRAY['database', 'nosql'];

-- 배열 교집합 여부
SELECT * FROM products
WHERE tags && ARRAY['redis', 'postgresql'];
```

---

## GiST (Generalized Search Tree)

### 내부 구조

GiST는 인덱스 구조를 위한 **프레임워크(Framework)**다. B-Tree처럼 균형 트리를 유지하지만, 각 노드에 저장되는 "키"가 무엇인지를 사용자가 정의한 operator class가 결정한다. 이를 통해 기하학적, 텍스트 유사도, 범위 등 다양한 데이터 타입에 대한 인덱싱이 가능하다.

GiST 노드의 각 키는 해당 서브트리에 포함된 모든 행의 "bounding predicate"(경계 술어)다. 예를 들어 공간 인덱스에서 각 내부 노드의 키는 자식 노드들 전체를 포함하는 MBR(Minimum Bounding Rectangle)이다.

```
GiST R-Tree 구조 (공간 인덱스):

           [MBR: 전체 공간]
           /              \
  [MBR: 서울 지역]    [MBR: 부산 지역]
    /      \              /       \
[강남구]  [마포구]  [해운대구]  [서면]
```

### Operator Classes

GiST는 operator class를 통해 다음 연산을 추상화한다:

| 메서드 | 역할 |
|-------|-----|
| `consistent` | 노드의 키가 쿼리를 만족할 수 있는지 판단 |
| `union` | 여러 키의 합집합 경계 계산 |
| `compress/decompress` | 키를 인덱스 저장 형식으로 변환 |
| `penalty` | 새 키 삽입 시 어느 서브트리에 넣을지 비용 계산 |
| `picksplit` | 노드 오버플로우 시 분할 방법 결정 |
| `distance` | KNN 검색 시 거리 계산 |

### 실전 예제: 지리공간 쿼리 (PostGIS)

```sql
-- PostGIS 활성화 (설치 필요)
CREATE EXTENSION IF NOT EXISTS postgis;

CREATE TABLE restaurants (
    id SERIAL PRIMARY KEY,
    name TEXT,
    location GEOMETRY(Point, 4326)
);

-- GiST 공간 인덱스
CREATE INDEX idx_restaurant_location
ON restaurants USING gist(location);

-- 특정 지점에서 반경 1km 내 식당 검색
SELECT name,
       ST_Distance(
           location::geography,
           ST_SetSRID(ST_MakePoint(126.9780, 37.5665), 4326)::geography
       ) AS distance_m
FROM restaurants
WHERE ST_DWithin(
    location::geography,
    ST_SetSRID(ST_MakePoint(126.9780, 37.5665), 4326)::geography,
    1000  -- 1000m
)
ORDER BY distance_m;

-- KNN (K-Nearest Neighbor): 가장 가까운 5개
SELECT name,
       location <-> ST_SetSRID(ST_MakePoint(126.9780, 37.5665), 4326) AS dist
FROM restaurants
ORDER BY dist
LIMIT 5;
```

### 범위 타입과 GiST

```sql
-- 예약 시스템에서 시간 겹침 방지
CREATE TABLE reservations (
    id SERIAL PRIMARY KEY,
    room_id INT,
    period TSRANGE  -- 시간 범위 타입
);

-- GiST로 범위 인덱스
CREATE INDEX idx_reservations_period
ON reservations USING gist(period);

-- 특정 시간에 예약 가능한 방 조회 (겹치지 않는 것)
SELECT DISTINCT room_id FROM reservations
WHERE NOT EXISTS (
    SELECT 1 FROM reservations r
    WHERE r.room_id = reservations.room_id
    AND r.period && '[2026-10-25 14:00, 2026-10-25 16:00)'::tsrange
);

-- 제약조건으로 중복 예약 방지 (Exclusion Constraint with GiST)
ALTER TABLE reservations
ADD CONSTRAINT no_overlap
EXCLUDE USING gist(room_id WITH =, period WITH &&);
```

---

## BRIN (Block Range INdex)

### 내부 구조

BRIN은 "대형 테이블에서 물리적 저장 순서와 쿼리 조건이 상관관계가 높을 때" 사용하는 매우 작은 인덱스다. 테이블의 물리적 블록들을 일정 크기(기본 128 블록 = 1,024KB)의 범위로 묶고, 각 범위의 MIN/MAX 값만 저장한다.

```
테이블 물리 블록:
블록 1-128:   created_at = 2024-01-01 ~ 2024-01-05
블록 129-256: created_at = 2024-01-05 ~ 2024-01-10
블록 257-384: created_at = 2024-01-10 ~ 2024-01-15
...

BRIN 메타데이터 (극소량):
range 1: min=2024-01-01, max=2024-01-05
range 2: min=2024-01-05, max=2024-01-10
range 3: min=2024-01-10, max=2024-01-15
```

쿼리 `WHERE created_at = '2024-01-07'`가 들어오면 BRIN은 range 2만 스캔하면 됨을 알 수 있다. 나머지 범위는 순차 스캔에서 제외된다(Bitmap Heap Scan).

### 상관관계(Correlation)가 핵심

BRIN은 컬럼 값의 물리적 저장 순서와 논리적 순서가 일치할 때만 효과적이다. 이 상관관계는 `pg_stats.correlation`으로 확인할 수 있다:

```sql
-- 컬럼의 물리적 상관관계 확인
SELECT attname, correlation
FROM pg_stats
WHERE tablename = 'events'
ORDER BY abs(correlation) DESC;

-- correlation = 1.0  → 완벽한 정순 정렬 (BRIN 매우 효과적)
-- correlation = -1.0 → 완벽한 역순 정렬 (BRIN 효과적)
-- correlation ≈ 0.0  → 무작위 순서 (BRIN 거의 무효)
```

### 실전 예제: 대용량 로그 테이블

```sql
-- 10억 건 로그 테이블 (날짜순 삽입 → 높은 correlation)
CREATE TABLE access_logs (
    id BIGSERIAL PRIMARY KEY,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    user_id INT,
    endpoint TEXT,
    status_code SMALLINT,
    response_ms INT
) PARTITION BY RANGE (created_at);  -- 파티셔닝과 병행

-- BRIN 인덱스 (B-Tree 대비 수백 배 작음)
CREATE INDEX idx_logs_brin ON access_logs
USING brin(created_at)
WITH (pages_per_range = 64);  -- 기본 128보다 작은 값 = 더 세밀한 범위

-- 시계열 범위 쿼리
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT COUNT(*), AVG(response_ms)
FROM access_logs
WHERE created_at BETWEEN '2026-10-25 00:00' AND '2026-10-25 01:00';

-- 인덱스 크기 비교
SELECT
    indexrelid::regclass AS index_name,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_index
WHERE indrelid = 'access_logs'::regclass;
```

### BRIN의 자동 요약(Auto-summarize)

새 블록이 추가될 때 BRIN 메타데이터를 자동으로 갱신하려면 `autosummarize` 옵션을 사용한다:

```sql
CREATE INDEX idx_ts ON access_logs
USING brin(created_at)
WITH (pages_per_range = 128, autosummarize = on);

-- 수동 요약 갱신
SELECT brin_summarize_new_values('idx_ts');

-- 특정 범위 재계산
SELECT brin_desummarize_range('idx_ts', 0);
SELECT brin_summarize_range('idx_ts', 0);
```

---

## 인덱스 타입 선택 가이드

```
데이터 유형별 권장 인덱스:

등호/범위 비교 (숫자, 날짜, 문자열 prefix)
  → B-Tree (기본)

전문 검색 (tsvector, trigram)
  → GIN (gin_trgm_ops)

배열/JSONB 포함 여부
  → GIN

공간 데이터 (Point, Polygon, Geometry)
  → GiST (with PostGIS)

범위 겹침/포함 (tsrange, int4range)
  → GiST (Exclusion Constraint)

텍스트 유사도 검색 (LIKE '%keyword%')
  → GIN with pg_trgm extension

순차 삽입 대형 테이블, 범위 쿼리
  → BRIN (correlation ≥ 0.9 권장)

IP 범위, 트리 구조
  → SP-GiST (Space-Partitioned GiST)
```

### 성능 벤치마크 참고

| 인덱스 | 크기 | 빌드 시간 | 검색 성능 | 갱신 오버헤드 |
|-------|-----|----------|---------|------------|
| B-Tree | 보통 | 빠름 | 최고(등호/범위) | 보통 |
| GIN | 큼 | 느림 | 최고(다중값) | 높음(fast update 완화) |
| GiST | 보통 | 보통 | 좋음(공간/범위) | 보통 |
| BRIN | 매우 작음 | 매우 빠름 | 보통(상관관계 의존) | 매우 낮음 |

---

## 주의사항 및 팁

### GIN 사용 시
- `gin_pending_list_limit` 기본값(4MB)은 쓰기가 많은 환경에서 병합 지연을 유발할 수 있음
- 대량 삽입 전 인덱스를 삭제하고 후에 재생성하는 것이 빠를 수 있음
- `pg_trgm` 확장과 함께 사용하면 LIKE/ILIKE 패턴 검색도 인덱스 활용 가능

### GiST 사용 시
- PostGIS 없이도 내장 `point`, `box`, `circle` 타입에 사용 가능
- KNN 검색(`<->` 연산자)은 GiST만 지원 (GIN 불가)
- Exclusion Constraint는 반드시 GiST 또는 SP-GiST 필요

### BRIN 사용 시
- correlation이 낮은 컬럼에 BRIN을 걸면 오히려 순차 스캔보다 느림
- `pages_per_range`를 줄이면 정밀도 향상이지만 인덱스 크기 증가
- IoT 시계열, 로그 테이블처럼 시간순 삽입 패턴에 최적

---

## 참고 자료
- [PostgreSQL Docs: GIN Indexes](https://www.postgresql.org/docs/current/gin.html)
- [PostgreSQL Docs: GiST Indexes](https://www.postgresql.org/docs/current/gist.html)
- [PostgreSQL Docs: BRIN Indexes](https://www.postgresql.org/docs/current/brin.html)
- [PostgreSQL Index Types Overview](https://www.postgresql.org/docs/current/indexes-types.html)
