---
layout: post
title: "Apache Cassandra 아키텍처 완전 정복: 마스터리스 분산 데이터베이스의 내부 동작 원리"
date: 2026-09-25
categories: [cs, computer-science]
tags: [cassandra, distributed-systems, nosql, database, consistent-hashing, compaction, wide-column]
---

Netflix, Apple, Instagram이 수십억 건의 쓰기 요청을 처리하기 위해 선택한 데이터베이스가 Apache Cassandra입니다. 단일 장애점이 없고, 글로벌 다중 데이터센터를 지원하며, 선형적인 수평 확장이 가능한 Cassandra의 비결은 무엇일까요? 이 글에서는 Cassandra의 마스터리스 링 아키텍처부터 쓰기·읽기 경로, 압축 전략, 일관성 모델까지 내부 동작을 완전히 해부합니다.

## 왜 Cassandra인가

관계형 데이터베이스는 수직 확장(더 좋은 서버)과 리더-팔로워 복제에 의존합니다. 초당 수백만 건의 쓰기, 페타바이트 규모의 데이터, 글로벌 멀티리전 요구사항 앞에서 이 구조는 한계를 드러냅니다.

Cassandra는 Amazon Dynamo(파티셔닝, 복제)와 Google Bigtable(Wide-Column 데이터 모델)의 아이디어를 결합해 이를 해결합니다:

- **마스터리스(Masterless)**: 모든 노드가 동등한 역할 수행, 단일 장애점 없음
- **튜닝 가능한 일관성**: 쓰기·읽기마다 `ONE`, `QUORUM`, `ALL` 선택
- **선형 확장**: 노드 추가 시 처리량이 선형적으로 증가
- **멀티 데이터센터 복제**: 지역 간 데이터 자동 복제

## 링 아키텍처와 일관성 해싱

Cassandra의 핵심은 **토큰 링(Token Ring)**입니다. 각 노드는 0 ~ 2¹²⁷ 범위의 토큰 값을 할당받아 가상의 링을 형성합니다. 파티션 키는 Murmur3 해시 함수로 해싱되어 링 위의 위치가 결정되고, 그 위치를 시계 방향으로 처음 만나는 노드가 해당 파티션의 코디네이터가 됩니다.

```
토큰 링 (6개 노드 예시):

                    Token 0
                  Node A (0~42)
            ↗                    ↘
  Node F (213~255)              Node B (43~85)
  |                                         |
  Node E (170~212)              Node C (86~128)
            ↖                    ↗
                  Node D (129~169)
```

### Virtual Nodes (Vnodes)

초기 Cassandra는 각 물리 노드에 단일 토큰을 할당했습니다. 이 방식은 노드 추가/제거 시 수동으로 토큰을 재분배해야 하는 문제가 있었습니다. Cassandra 1.2부터 도입된 **Vnodes**는 각 물리 노드가 링 전체에 걸쳐 수백 개(기본값 256개)의 가상 토큰을 소유합니다:

```
Physical Node A: Token [5, 38, 72, 105, 142, ...] (256개)
Physical Node B: Token [3, 19, 64, 99, 131, ...]  (256개)
Physical Node C: Token [8, 44, 88, 120, 158, ...]  (256개)
```

이로써 노드가 추가되면 기존 모든 노드에서 균등하게 데이터를 자동 이관하고, 노드가 제거되면 그 토큰들이 나머지 노드에 자동 분배됩니다.

## 복제 전략

데이터는 Replication Factor(RF)에 따라 여러 노드에 복제됩니다. `RF=3`이면 각 파티션이 3개의 노드에 저장됩니다.

```cql
-- keyspace 생성 시 복제 전략 지정
CREATE KEYSPACE my_app
WITH REPLICATION = {
    'class': 'NetworkTopologyStrategy',
    'us-east': 3,     -- 미국 동부 데이터센터에 3개 복제본
    'eu-west': 2      -- 유럽 서부에 2개 복제본
};

-- 예시 테이블: 사용자 세션 데이터
CREATE TABLE sessions (
    user_id   UUID,
    session_id TIMEUUID,
    ip_address TEXT,
    user_agent TEXT,
    created_at TIMESTAMP,
    PRIMARY KEY (user_id, session_id)  -- user_id: 파티션 키, session_id: 클러스터링 키
) WITH CLUSTERING ORDER BY (session_id DESC)
  AND default_time_to_live = 86400;   -- 24시간 TTL
```

## 쓰기 경로: 속도를 위한 설계

Cassandra는 쓰기 성능을 최우선으로 설계되었습니다. 쓰기 요청은 네 단계로 처리됩니다:

```
Client
  │
  ▼
Coordinator Node (코디네이터: 클라이언트가 접속한 아무 노드)
  │   일관성 레벨에 따라 필요한 노드 수 결정 (QUORUM = ⌊RF/2⌋ + 1)
  ├──→ Replica Node 1
  ├──→ Replica Node 2   ← 일관성 레벨 QUORUM 충족 시 ACK 반환
  └──→ Replica Node 3
```

각 복제 노드의 쓰기 처리:

1. **CommitLog 기록**: 모든 쓰기를 먼저 순차적 디스크 파일에 기록 (내구성 보장)
2. **MemTable 기록**: 정렬된 인메모리 구조에 데이터 저장
3. **MemTable 플러시**: 메모리 임계값(기본 256MB) 도달 시 SSTable로 플러시

```
쓰기 요청 → CommitLog (append-only, fast) → MemTable (sorted in-memory)
                                                  │
                                   임계값 도달 시  ▼
                                              SSTable (불변 디스크 파일)
                                              SSTable
                                              SSTable ...
```

### SSTable 구조

SSTable(Sorted String Table)은 불변(Immutable)의 정렬된 디스크 파일입니다. 하나의 SSTable은 여러 파일로 구성됩니다:

```
Data.db          -- 실제 행 데이터 (압축 저장)
Index.db         -- 파티션 키 → Data.db 오프셋 인덱스
Filter.db        -- Bloom Filter (파티션 존재 여부 빠른 확인)
Statistics.db    -- 통계 정보 (파티션 수, 토큰 범위, 히스토그램)
Summary.db       -- Index.db의 샘플 (Index.db를 메모리에서 이진 탐색하기 위한 인덱스)
CompressionInfo.db -- 청크별 압축 메타데이터
```

## 읽기 경로: 여러 레이어 병합

쓰기가 MemTable과 여러 SSTable에 분산되어 있으므로 읽기는 이를 병합해야 합니다:

```
Client → Coordinator → Replica Node (읽기)
                              │
                     1. MemTable 조회
                     2. SSTable Bloom Filter 확인 (파티션 존재 가능성 O(1))
                     3. Partition Summary → Partition Index → Data 위치 확인
                     4. Row Cache 확인 (활성화된 경우)
                     5. 여러 SSTable 데이터 병합 (Merge Iterator)
                     6. 삭제 표시(Tombstone) 처리
```

`QUORUM` 읽기의 경우 코디네이터는 여러 복제 노드에서 데이터를 읽어 **Read Repair**를 수행합니다. 오래된 복제본은 최신 타임스탬프 기준으로 업데이트됩니다.

### 코드 예제 1: Python cassandra-driver로 Cassandra CRUD

```python
from cassandra.cluster import Cluster, ExecutionProfile, EXEC_PROFILE_DEFAULT
from cassandra.policies import DCAwareRoundRobinPolicy, TokenAwarePolicy
from cassandra.query import SimpleStatement, PreparedStatement, ConsistencyLevel
from uuid import uuid4
from datetime import datetime

def create_cluster() -> Cluster:
    # 토큰 인식 부하 분산: 코디네이터 홉 최소화
    profile = ExecutionProfile(
        load_balancing_policy=TokenAwarePolicy(
            DCAwareRoundRobinPolicy(local_dc='us-east')
        ),
        consistency_level=ConsistencyLevel.LOCAL_QUORUM,
        request_timeout=10.0
    )
    cluster = Cluster(
        contact_points=['cassandra-node1', 'cassandra-node2'],
        execution_profiles={EXEC_PROFILE_DEFAULT: profile}
    )
    return cluster

# Prepared Statement: 파싱 비용 1회만 지불
INSERT_SESSION = None
SELECT_SESSIONS = None

def setup_prepared_statements(session):
    global INSERT_SESSION, SELECT_SESSIONS
    INSERT_SESSION = session.prepare("""
        INSERT INTO my_app.sessions (user_id, session_id, ip_address, user_agent, created_at)
        VALUES (?, now(), ?, ?, toTimestamp(now()))
        USING TTL 86400
    """)
    SELECT_SESSIONS = session.prepare("""
        SELECT session_id, ip_address, created_at
        FROM my_app.sessions
        WHERE user_id = ?
        LIMIT 10
    """)

def write_session(session, user_id: str, ip: str, user_agent: str):
    # QUORUM 일관성: 과반수 복제 노드에 쓰기 확인 (가용성과 일관성 균형)
    bound = INSERT_SESSION.bind((user_id, ip, user_agent))
    session.execute(bound)

def read_sessions(session, user_id: str) -> list[dict]:
    bound = SELECT_SESSIONS.bind((user_id,))
    rows = session.execute(bound)
    return [
        {
            'session_id': str(row.session_id),
            'ip': row.ip_address,
            'created_at': row.created_at.isoformat()
        }
        for row in rows
    ]

def batch_write_example(session, events: list[dict]):
    """
    주의: Cassandra BATCH는 원자성 보장 목적으로만 사용
    성능 향상 목적의 배치 사용은 오히려 핫스팟 유발
    """
    from cassandra.query import BatchStatement, BatchType
    batch = BatchStatement(batch_type=BatchType.UNLOGGED)
    for event in events[:30]:  # 배치 크기 제한 권장
        batch.add(INSERT_SESSION, (event['user_id'], event['ip'], event['ua']))
    session.execute(batch)

if __name__ == "__main__":
    cluster = create_cluster()
    cass_session = cluster.connect()
    setup_prepared_statements(cass_session)
    
    user_id = str(uuid4())
    write_session(cass_session, user_id, "203.0.113.42", "Mozilla/5.0 ...")
    sessions = read_sessions(cass_session, user_id)
    print(f"Found {len(sessions)} sessions for user {user_id}")
    cluster.shutdown()
```

## 압축(Compaction) 전략

쓰기가 쌓일수록 SSTable이 늘어나고 읽기 성능이 저하됩니다. Compaction은 여러 SSTable을 병합하고 Tombstone(삭제 표시)을 제거하는 백그라운드 프로세스입니다.

### STCS (SizeTieredCompactionStrategy)
동일한 크기의 SSTable 4개가 쌓이면 하나로 병합합니다. 쓰기 집약적 워크로드에 적합하지만 공간 증폭이 큽니다.

### LCS (LeveledCompactionStrategy)
SSTable을 여러 레벨(L0 ~ L6)로 관리합니다. 각 레벨의 SSTable은 파티션 키 범위가 겹치지 않아 읽기 효율이 높습니다. 읽기 집약적 워크로드에 적합합니다.

### TWCS (TimeWindowCompactionStrategy)
시계열 데이터에 최적화됩니다. 시간 윈도우(예: 1시간) 단위로 SSTable을 분리 관리해 TTL 만료 데이터를 통째로 삭제 가능합니다.

```cql
-- 시계열 이벤트 테이블: TWCS 적용
CREATE TABLE events (
    sensor_id  TEXT,
    event_time TIMESTAMP,
    value      DOUBLE,
    PRIMARY KEY (sensor_id, event_time)
) WITH CLUSTERING ORDER BY (event_time DESC)
  AND default_time_to_live = 2592000  -- 30일 TTL
  AND compaction = {
      'class': 'TimeWindowCompactionStrategy',
      'compaction_window_unit': 'HOURS',
      'compaction_window_size': 1
  };
```

## 코드 예제 2: 일관성 레벨에 따른 가용성 트레이드오프

```python
from cassandra.query import ConsistencyLevel
from cassandra.cluster import Cluster

"""
RF = 3, 노드 3개 클러스터에서의 일관성 레벨 비교:

| 일관성 레벨 | 쓰기 필요 ACK | 읽기 필요 응답 | 내결함성 | 지연시간 |
|------------|--------------|---------------|---------|---------|
| ONE        | 1개          | 1개           | 낮음     | 최저    |
| QUORUM     | 2개          | 2개           | 중간     | 중간    |
| ALL        | 3개          | 3개           | 없음     | 최고    |
| LOCAL_ONE  | 로컬 DC 1개  | 로컬 DC 1개   | 낮음     | 최저    |
| LOCAL_QUORUM| 로컬 DC 과반| 로컬 DC 과반  | 중간     | 중간    |

강한 일관성 보장: WRITE(QUORUM) + READ(QUORUM) 조합
  → 2 + 2 > 3 (RF), 교집합 보장으로 항상 최신 데이터 읽기 가능
"""

def demo_consistency_levels(cluster: Cluster):
    session = cluster.connect('my_app')
    
    # 실시간 결제: 강한 일관성 필수
    payment_stmt = session.prepare("INSERT INTO payments ... VALUES (?)")
    payment_stmt.consistency_level = ConsistencyLevel.QUORUM
    
    # 사용자 피드: 약간의 지연 허용 가능
    feed_stmt = session.prepare("SELECT ... FROM feed WHERE user_id = ?")
    feed_stmt.consistency_level = ConsistencyLevel.LOCAL_ONE
    
    # 멀티 DC: 로컬 DC 우선 처리 후 비동기 원격 복제
    profile_stmt = session.prepare("INSERT INTO user_profiles ... VALUES (?)")
    profile_stmt.consistency_level = ConsistencyLevel.LOCAL_QUORUM

# Lightweight Transaction (LWT): 비교 후 설정 (선형화 가능성 보장)
def create_unique_username(session, username: str, user_id: str) -> bool:
    """Paxos 기반 LWT로 유니크 사용자명 생성 보장"""
    result = session.execute("""
        INSERT INTO usernames (username, user_id)
        VALUES (%s, %s)
        IF NOT EXISTS
    """, (username, user_id))
    # [applied] = True: 삽입 성공, False: 이미 존재
    return result.one().applied
```

## Gossip 프로토콜: 노드 간 통신

Cassandra는 중앙 코디네이터 없이 **Gossip 프로토콜**로 노드 상태를 전파합니다. 매 초마다 각 노드는 최대 3개의 랜덤 노드에게 자신과 알고 있는 다른 노드들의 상태를 전송합니다. 모든 노드가 O(log N) 라운드 내에 전체 클러스터 상태를 파악합니다.

```bash
# 노드 상태 확인
nodetool status
# UN: Up/Normal (정상)
# DN: Down/Normal (다운됨)
# UJ: Up/Joining (링 참여 중)
# UL: Up/Leaving (링 이탈 중)

# 링 토큰 분포 확인
nodetool ring

# 압축 진행 현황
nodetool compactionstats

# 읽기/쓰기 지연시간 모니터링
nodetool tpstats
```

## 주의사항과 안티패턴

**데이터 모델링은 쿼리 중심으로:**
- Cassandra에는 서버 사이드 JOIN이 없습니다. 조회 패턴에 맞게 테이블을 설계하세요 (Query-First Design).
- 동일한 데이터를 여러 테이블에 비정규화하는 것이 일반적입니다.

**파티션 크기 제한:**
- 단일 파티션 내 행 수는 수백만 개 이하 유지 권장
- `SELECT COUNT(*) FROM table WHERE partition_key = x` 대신 카운터 테이블 사용

**Tombstone 관리:**
- `DELETE` 또는 TTL 만료는 데이터를 즉시 지우지 않고 Tombstone을 생성합니다
- Tombstone이 과도하게 쌓이면 읽기 성능이 급격히 저하됩니다
- `gc_grace_seconds`(기본 10일) 후 Compaction이 Tombstone을 제거합니다

**ALLOW FILTERING 금지:**
```cql
-- 절대 사용하지 말 것 (전체 클러스터 스캔)
SELECT * FROM sessions WHERE ip_address = '1.2.3.4' ALLOW FILTERING;
-- 대신 보조 인덱스 또는 별도 조회 테이블 생성
CREATE INDEX ON sessions (ip_address);  -- 소규모 클러스터에만 적합
```

## 참고 자료
- [Apache Cassandra 공식 아키텍처 문서](https://cassandra.apache.org/doc/latest/cassandra/architecture/dynamo.html)
- [DataStax 아키텍처 심화 가이드](https://docs.datastax.com/en/cassandra-oss/3.0/cassandra/architecture/archDataDistributeHashing.html)
- [Cassandra: The Definitive Guide (O'Reilly)](https://www.oreilly.com/library/view/cassandra-the-definitive/9781098115159/)
- [Instagram Engineering: Cassandra at Scale](https://engineering.instagram.com/open-sourcing-a-10x-reduction-in-apache-cassandra-tail-latency-d64f86b43589)
