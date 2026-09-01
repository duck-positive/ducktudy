---
layout: post
title: "데이터베이스 커넥션 풀링 완전 정복: PgBouncer와 HikariCP의 내부 동작 원리"
date: 2026-09-01
categories: [cs, computer-science]
tags: [database, connection-pooling, pgbouncer, hikaricp, postgresql, performance, backend]
---

## 개요

데이터베이스 연결(Connection)을 맺는 일은 생각보다 훨씬 무겁다. PostgreSQL 기준으로 새 연결 하나를 수립하는 데 TCP 3-way 핸드셰이크, TLS 협상, 인증, 백엔드 프로세스 fork 등 총 **약 25~45ms**가 소요된다. 초당 1,000개의 짧은 쿼리를 처리하는 서비스에서 매번 새 연결을 만든다면, 연결 비용만으로 CPU와 시간의 상당 부분이 낭비된다.

**커넥션 풀링(Connection Pooling)**은 미리 생성해둔 연결 집합을 재사용함으로써 이 오버헤드를 제거하는 기법이다. 현대 웹 서비스에서 커넥션 풀링 없이 높은 동시성을 달성하기는 사실상 불가능하다.

이 아티클에서는 두 가지 대표적인 풀링 도구를 심층 분석한다.

- **HikariCP**: JVM 애플리케이션 내부의 인프로세스(in-process) 풀링
- **PgBouncer**: 데이터베이스 앞단에 위치하는 독립 프록시 방식 풀링

---

## 왜 커넥션 풀이 필요한가

### PostgreSQL 연결 모델의 구조적 특성

PostgreSQL은 클라이언트 연결마다 **별도의 OS 프로세스**를 fork한다. 이는 동시 연결 수가 곧 운영체제 프로세스 수로 직결됨을 의미한다.

```
클라이언트 → PostgreSQL postmaster 프로세스
               ├── 백엔드 프로세스 1 (연결 #1)
               ├── 백엔드 프로세스 2 (연결 #2)
               └── 백엔드 프로세스 N (연결 #N)
```

연결 수가 수백 개를 넘으면:
- 프로세스 간 컨텍스트 스위칭 오버헤드 급증
- 공유 메모리(Shared Buffer) 경합 심화
- `max_connections` 파라미터 초과 시 연결 거부 오류

일반적으로 PostgreSQL의 `max_connections`는 100~400개로 설정한다. 그러나 마이크로서비스 아키텍처에서는 수십 개의 서비스 인스턴스 × 인스턴스당 풀 크기 = 손쉽게 1,000개를 넘어선다.

### 커넥션 풀의 두 가지 위치

```
[앱 서버 1] ─┐
[앱 서버 2] ─┤── [PgBouncer] ──────── [PostgreSQL]
[앱 서버 3] ─┘   (독립 프록시)         (백엔드 연결 수 제한)

[앱 서버 1 내부]
  [HikariCP Pool] ──────────────────── [PostgreSQL]
  (JVM 내부 풀)
```

---

## HikariCP 심층 분석

HikariCP는 "스위스 제일의 JDBC 연결 풀"이라는 별명으로, 극단적인 성능과 단순함을 목표로 설계되었다.

### 내부 구조: ConcurrentBag

HikariCP의 핵심 자료구조는 `ConcurrentBag`이다. 일반적인 `BlockingQueue`와 달리:

1. **스레드 로컬 캐싱**: 각 스레드가 최근 사용한 연결을 `ThreadLocal<List<WeakReference>>` 에 캐시한다. 동일 스레드가 재요청 시 큐 경합 없이 바로 획득.
2. **손 오프(Handoff) 메커니즘**: 반납된 연결을 대기 중인 스레드에 직접 전달한다. Semaphore 신호 대신 `SynchronousQueue`와 유사한 패턴.

```
연결 획득 흐름:
1. ThreadLocal 캐시에서 Available 연결 확인
2. 없으면 공유 CopyOnWriteArrayList에서 스캔
3. 없으면 waiters 큐에 등록 후 대기 (maxWait까지)
4. 타임아웃 → SQLTimeoutException
```

### 핵심 설정 파라미터

```java
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:postgresql://localhost:5432/mydb");
config.setUsername("user");
config.setPassword("password");

// 풀 크기 — 핵심 공식: core_count * 2 + effective_spindle_count
// SSD 기준 약 10, HDD 포함 시 +1
config.setMaximumPoolSize(10);
config.setMinimumIdle(5);           // 유휴 연결 최소 유지 수

// 타임아웃 설정
config.setConnectionTimeout(30_000);    // 연결 획득 대기 최대 30초
config.setIdleTimeout(600_000);         // 유휴 연결 제거까지 10분
config.setMaxLifetime(1_800_000);       // 연결 강제 교체 주기 30분

// 연결 유효성 검사
config.setConnectionTestQuery("SELECT 1");    // JDBC 4.0 미만 드라이버용
// JDBC 4.0+ 드라이버는 isValid() 자동 사용

// Postgres 최적화
config.addDataSourceProperty("prepStmtCacheSize", "250");
config.addDataSourceProperty("prepStmtCacheSqlLimit", "2048");
config.addDataSourceProperty("cachePrepStmts", "true");

DataSource ds = new HikariDataSource(config);
```

### Pool Size 계산 공식

HikariCP 문서의 권장 공식:

```
pool_size = (core_count × 2) + effective_spindle_count
```

- `core_count`: CPU 코어 수 (하이퍼스레딩 제외한 물리 코어)
- `effective_spindle_count`: I/O 병렬성 — SSD = 1, HDD = 스핀들 수

**예시**: 4-core CPU + SSD → `pool_size = 4*2 + 1 = 9` → **10개** 설정

이 공식의 근거는 CPU 코어 수를 초과하는 스레드를 만들어도 컨텍스트 스위칭 비용만 늘어나기 때문이다.

```java
// 잘못된 예 — "연결이 더 많을수록 빠르다"는 오해
config.setMaximumPoolSize(200);  // CPU 코어가 4개인 서버에서 역효과

// 올바른 예 — Hikari의 권장값
config.setMaximumPoolSize(10);
```

---

## PgBouncer 심층 분석

PgBouncer는 PostgreSQL 전용 경량 커넥션 풀 프록시다. C로 작성되어 메모리 사용량이 극히 낮으며(연결당 약 2KB), 단일 프로세스로 수만 개의 클라이언트 연결을 처리한다.

### 3가지 풀링 모드

```ini
# pgbouncer.ini
[databases]
mydb = host=127.0.0.1 port=5432 dbname=mydb

[pgbouncer]
listen_addr = 127.0.0.1
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt

pool_mode = transaction    # 핵심 설정: session, transaction, statement 중 선택
max_client_conn = 10000    # 클라이언트 최대 연결 수
default_pool_size = 20     # PostgreSQL으로의 실제 연결 수
min_pool_size = 5
reserve_pool_size = 5
```

**모드 비교**:

| 모드 | 연결 반환 시점 | 특징 | 권장 사용 |
|------|---------------|------|-----------|
| `session` | 클라이언트 세션 종료 시 | PostgreSQL 모든 기능 지원 | `LISTEN/NOTIFY`, `SET` 사용 시 |
| `transaction` | 트랜잭션 종료 시 | **가장 높은 연결 다중화율** | 일반적인 CRUD API |
| `statement` | SQL 문장 종료 시 | 트랜잭션 불가 | 특수 목적만 |

### Transaction 모드의 한계와 해결책

Transaction 모드는 가장 효율적이지만 PostgreSQL의 세션 기반 기능들이 동작하지 않는다.

```sql
-- ❌ Transaction 모드에서 작동하지 않는 것들

-- 1. PREPARE/EXECUTE — 서버 사이드 prepared statement
PREPARE my_stmt (INT) AS SELECT * FROM users WHERE id = $1;
EXECUTE my_stmt(42);  -- 다른 서버 연결로 라우팅될 수 있어 실패

-- 2. 세션 변수 설정
SET search_path = my_schema;  -- 다음 트랜잭션은 다른 연결 사용

-- 3. Advisory Lock
SELECT pg_advisory_lock(12345);  -- 세션 수명과 결합되어 있음
```

**해결책**:

```ini
# 1. PgBouncer 1.21+ 서버 사이드 prepared statement 지원
server_reset_query = DISCARD ALL
ignore_startup_parameters = extra_float_digits

# 2. 클라이언트가 세션 모드 필요 시 별도 포트로 분리
[databases]
mydb_tx = host=127.0.0.1 port=5432 pool_mode=transaction
mydb_sess = host=127.0.0.1 port=5432 pool_mode=session
```

```java
// 애플리케이션에서 기능별 DataSource 분리
@Bean
@Qualifier("transactionPool")
public DataSource transactionPoolDataSource() {
    HikariConfig config = new HikariConfig();
    config.setJdbcUrl("jdbc:postgresql://pgbouncer:6432/mydb_tx");
    // ...
    return new HikariDataSource(config);
}

@Bean
@Qualifier("sessionPool")
public DataSource sessionPoolDataSource() {
    HikariConfig config = new HikariConfig();
    config.setJdbcUrl("jdbc:postgresql://pgbouncer:6433/mydb_sess");
    // ...
    return new HikariDataSource(config);
}
```

---

## 두 도구의 조합 아키텍처

현대적인 배포에서는 HikariCP와 PgBouncer를 **함께** 사용한다.

```
[App Instance 1]          [App Instance 2]
  HikariCP (5개 연결)       HikariCP (5개 연결)
       │                         │
       └──────────┬──────────────┘
                  ▼
           [PgBouncer]          ← transaction mode
           (20개 PostgreSQL 연결)
                  │
                  ▼
           [PostgreSQL]
           (max_connections = 100)
```

이 구성에서:
- 앱 인스턴스 10개 × HikariCP 5개 = **클라이언트 50개 연결**
- PgBouncer가 PostgreSQL에는 **20개 연결**만 유지
- PostgreSQL 입장에서는 프로세스 20개만 관리

연결 수를 극적으로 줄이면서, 각 인스턴스의 `HikariCP`가 스레드 로컬 캐싱으로 빠른 획득을 보장한다.

---

## 연결 풀 모니터링

### HikariCP JMX 메트릭

```java
// Spring Boot Actuator + Micrometer 통합
management.metrics.enable.hikaricp=true

// 주요 메트릭
hikaricp_connections_active     // 현재 사용 중인 연결 수
hikaricp_connections_idle       // 유휴 연결 수
hikaricp_connections_pending    // 대기 중인 스레드 수
hikaricp_connections_timeout_total  // 타임아웃 횟수 (이게 증가하면 풀 크기 재검토)
hikaricp_connection_acquire_ms  // 연결 획득 평균 시간
```

### PgBouncer 통계

```bash
# psql로 PgBouncer 관리 인터페이스 접속
psql -h 127.0.0.1 -p 6432 -U pgbouncer pgbouncer

SHOW POOLS;
# database | user | cl_active | cl_waiting | sv_active | sv_idle | sv_used
# mydb     | app  |        45 |          0 |        20 |       0 |       0

SHOW STATS;
# total_query_count | total_query_time | avg_query_time | ...

# 연결 강제 종료
KILL mydb;

# 설정 리로드
RELOAD;
```

---

## 주의사항과 팁

### 1. 연결 누수(Connection Leak) 감지

```java
// HikariCP 연결 누수 감지
config.setLeakDetectionThreshold(2000);  // 2초 이상 반납 안 되면 경고

// 반드시 try-with-resources 사용
try (Connection conn = dataSource.getConnection();
     PreparedStatement stmt = conn.prepareStatement("SELECT 1")) {
    // ... 자동으로 반납됨
}
// ❌ 수동 close() 없이 참조만 버리는 코드는 누수 발생
```

### 2. `max_connections` 계획

```
PostgreSQL max_connections 계산:
  = default_pool_size × PgBouncer 인스턴스 수
  + 모니터링 도구 연결 수 (5~10개)
  + 관리자 연결 수 (superuser_reserved_connections = 3)

예: pool 20 × PgBouncer 2개 + 15 = 55 → max_connections = 100 설정
```

### 3. 트랜잭션 내 긴 대기 방지

PgBouncer transaction 모드에서는 트랜잭션이 열린 채로 서버 연결을 점유한다. 긴 트랜잭션은 풀 고갈의 주범이다.

```java
// ❌ 나쁜 패턴: 트랜잭션 안에서 외부 I/O
@Transactional
public void process(Long id) {
    User user = userRepo.findById(id).orElseThrow();
    String result = externalApiClient.call(user);  // 수백ms 외부 호출
    user.setResult(result);
    userRepo.save(user);
}

// ✅ 좋은 패턴: 외부 I/O는 트랜잭션 밖으로
public void process(Long id) {
    User user = userRepo.findById(id).orElseThrow();       // 짧은 TX
    String result = externalApiClient.call(user);           // TX 밖 호출
    updateUserResult(id, result);                           // 짧은 TX
}
```

### 4. PgBouncer TLS 설정 (프로덕션 필수)

```ini
[pgbouncer]
client_tls_sslmode = require
client_tls_key_file = /etc/pgbouncer/server.key
client_tls_cert_file = /etc/pgbouncer/server.crt

server_tls_sslmode = require
server_tls_ca_file = /etc/ssl/certs/ca-certificates.crt
```

---

## 정리

커넥션 풀링은 고성능 데이터베이스 계층의 필수 구성 요소다. **HikariCP**는 JVM 애플리케이션 내부에서 스레드 로컬 캐싱과 효율적인 ConcurrentBag으로 나노초 단위의 연결 획득을 구현한다. **PgBouncer**는 독립 프록시로서 여러 서비스 인스턴스의 연결을 통합 관리하며, transaction 모드로 수십 배의 연결 다중화를 달성한다. 두 도구를 조합하면 수천 개의 애플리케이션 스레드가 수십 개의 실제 PostgreSQL 연결을 안전하게 공유하는 아키텍처를 구현할 수 있다. 핵심은 **풀 크기를 CPU 코어 수 기반으로 설정하고, 트랜잭션을 최대한 짧게 유지하며, 연결 누수를 모니터링**하는 것이다.

## 참고 자료
- [HikariCP GitHub 저장소 — 설정 파라미터 공식 문서](https://github.com/brettwooldridge/HikariCP)
- [pgbouncer/pgbouncer GitHub 저장소](https://github.com/pgbouncer/pgbouncer)
