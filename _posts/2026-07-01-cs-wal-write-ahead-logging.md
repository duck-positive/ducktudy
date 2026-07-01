---
layout: post
title: "WAL(Write-Ahead Logging) 완전 정복: PostgreSQL이 데이터를 절대 잃지 않는 비밀"
date: 2026-07-01
categories: [cs, computer-science]
tags: [WAL, write-ahead-logging, database, postgresql, sqlite, crash-recovery, durability, ACID]
---

데이터베이스의 가장 중요한 속성 중 하나는 **내구성(Durability)**입니다. "트랜잭션이 커밋되었다면, 그 결과는 시스템이 충돌하더라도 반드시 살아남아야 한다"는 ACID의 D가 바로 그것입니다. 이 내구성을 실제로 구현하는 핵심 메커니즘이 **WAL(Write-Ahead Logging)**입니다. PostgreSQL, SQLite, MySQL InnoDB, 심지어 ZooKeeper와 Kafka까지 수많은 시스템이 WAL에 의존합니다.

## WAL이란 무엇인가?

WAL의 핵심 원칙은 딱 하나입니다:

> **데이터 파일을 수정하기 전에, 반드시 로그를 먼저 디스크에 기록하라.**

데이터베이스에 `UPDATE accounts SET balance = 500 WHERE id = 1`이라는 쿼리가 들어온다고 상상해봅시다. 이 변경을 즉시 데이터 파일에 반영하면 어떻게 될까요? 쓰기가 완료되기 직전에 전원이 나가면, 디스크의 데이터는 절반만 수정된 불일치 상태가 됩니다. 이것이 바로 **torn write** 문제입니다.

WAL은 다음 순서로 작동합니다:

1. **변경 내용을 WAL 레코드로 기록** (append-only, 순차 쓰기)
2. **WAL 버퍼를 디스크에 fsync** (이 시점에서 커밋 완료를 클라이언트에 알림)
3. **실제 데이터 파일(힙, 인덱스)에 변경 반영** (비동기적으로, 나중에)

1~2번이 완료된 순간 트랜잭션은 "커밋되었다"고 볼 수 있습니다. 3번이 완료되기 전에 시스템이 충돌해도, 재시작 시 WAL을 재생(replay)하여 데이터 파일을 복구할 수 있습니다.

## 왜 WAL이 필요한가?

### 순차 쓰기 vs 랜덤 쓰기

전통적인 HDD에서 랜덤 I/O는 순차 I/O보다 수십~수백 배 느립니다. 트랜잭션이 발생할 때마다 변경된 모든 데이터 페이지(8KB 단위)를 디스크의 임의 위치에 fsync하면 성능이 처참해집니다.

WAL은 이 문제를 해결합니다:
- WAL 파일은 **순차적(append-only)**으로 기록됩니다. 디스크 헤드 이동 없이 연속적으로 씁니다.
- 커밋 시 WAL 파일만 fsync하면 됩니다. 실제 데이터 파일은 나중에 **체크포인트(checkpoint)** 시점에 일괄 반영합니다.

### 세 가지 핵심 기능

| 기능 | 설명 |
|------|------|
| **Crash Recovery** | 재시작 시 WAL을 재생하여 커밋된 트랜잭션 복구 |
| **Streaming Replication** | WAL을 네트워크로 전송하여 스탠바이 서버 동기화 |
| **Point-in-Time Recovery (PITR)** | 특정 시점으로 데이터베이스 복구 |

## PostgreSQL WAL 내부 구조

### LSN (Log Sequence Number)

PostgreSQL WAL의 모든 레코드는 **LSN**으로 식별됩니다. LSN은 WAL 파일 내의 바이트 오프셋을 나타내는 64비트 정수입니다.

```sql
-- 현재 WAL 위치 확인
SELECT pg_current_wal_lsn();
-- 결과: 0/3A1B2C0

-- WAL 파일 목록 확인
SELECT * FROM pg_ls_waldir() ORDER BY modification DESC LIMIT 5;

-- WAL 레코드 내용 분석 (pg_waldump 사용)
-- $ pg_waldump -p /var/lib/postgresql/data/pg_wal 000000010000000000000003
```

### WAL 레코드 구조

각 WAL 레코드는 다음 정보를 포함합니다:

```
┌─────────────────────────────────────────────┐
│ XLogRecord Header                           │
│   xl_tot_len: 레코드 전체 길이              │
│   xl_xid:     트랜잭션 ID                  │
│   xl_prev:    이전 레코드의 LSN            │
│   xl_info:    레코드 타입 (INSERT/UPDATE..) │
│   xl_rmid:    리소스 매니저 ID             │
│   xl_crc:     CRC32 체크섬                 │
├─────────────────────────────────────────────┤
│ Data Block (변경된 페이지의 이미지 또는 델타) │
└─────────────────────────────────────────────┘
```

### 체크포인트

WAL이 계속 쌓이면 디스크가 가득 찹니다. **체크포인트**는 주기적으로 WAL의 내용을 실제 데이터 파일에 반영하고, 반영이 완료된 WAL 파일을 삭제하는 과정입니다.

```sql
-- 강제 체크포인트 실행
CHECKPOINT;

-- 체크포인트 설정 확인
SHOW checkpoint_timeout;    -- 기본값: 5min
SHOW max_wal_size;          -- 기본값: 1GB
SHOW checkpoint_completion_target;  -- 기본값: 0.9
```

## 실제 구현 예제

### 예제 1: SQLite WAL 모드 활성화

SQLite는 기본적으로 **Rollback Journal** 방식을 사용하지만, WAL 모드로 전환하면 읽기-쓰기 동시성이 크게 향상됩니다.

```python
import sqlite3
import threading
import time

def setup_wal_mode(db_path: str) -> sqlite3.Connection:
    """SQLite WAL 모드 활성화"""
    conn = sqlite3.connect(db_path, check_same_thread=False)
    
    # WAL 모드 활성화
    conn.execute("PRAGMA journal_mode = WAL;")
    # WAL 파일 자동 체크포인트 임계값 (페이지 수)
    conn.execute("PRAGMA wal_autocheckpoint = 1000;")
    # 동기화 수준 (NORMAL: WAL 모드에서 충분히 안전)
    conn.execute("PRAGMA synchronous = NORMAL;")
    
    print(f"Journal mode: {conn.execute('PRAGMA journal_mode').fetchone()[0]}")
    return conn

def writer(conn: sqlite3.Connection, writer_id: int):
    """쓰기 스레드"""
    for i in range(10):
        conn.execute(
            "INSERT INTO logs (writer_id, value, ts) VALUES (?, ?, datetime('now'))",
            (writer_id, i)
        )
        conn.commit()
        time.sleep(0.01)
    print(f"Writer {writer_id} done")

def reader(conn: sqlite3.Connection, reader_id: int):
    """읽기 스레드 — WAL 모드에서는 쓰기와 동시에 읽기 가능"""
    for _ in range(5):
        rows = conn.execute("SELECT COUNT(*) FROM logs").fetchone()
        print(f"Reader {reader_id}: {rows[0]} rows")
        time.sleep(0.05)

# 데이터베이스 설정
import tempfile, os
db_file = tempfile.mktemp(suffix=".db")
conn = setup_wal_mode(db_file)
conn.execute("""
    CREATE TABLE IF NOT EXISTS logs (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        writer_id INTEGER,
        value INTEGER,
        ts TEXT
    )
""")
conn.commit()

# 다중 쓰기/읽기 스레드 실행
threads = []
for wid in range(3):
    # 각 스레드는 별도 연결 (WAL은 다중 연결 지원)
    wconn = sqlite3.connect(db_file, check_same_thread=False)
    wconn.execute("PRAGMA journal_mode = WAL;")
    threads.append(threading.Thread(target=writer, args=(wconn, wid)))

rconn = sqlite3.connect(db_file, check_same_thread=False)
rconn.execute("PRAGMA journal_mode = WAL;")
threads.append(threading.Thread(target=reader, args=(rconn, 0)))

for t in threads:
    t.start()
for t in threads:
    t.join()

# WAL 파일 확인 (.db-wal 파일 생성됨)
print(f"\nWAL file exists: {os.path.exists(db_file + '-wal')}")
print(f"SHM file exists: {os.path.exists(db_file + '-shm')}")

# 정리
conn.execute("PRAGMA wal_checkpoint(FULL);")  # 강제 체크포인트
conn.close()
```

### 예제 2: 미니 WAL 구현으로 원리 이해하기

WAL의 핵심 원리를 Python으로 직접 구현해봅니다.

```python
import struct
import os
import json
import fcntl
from dataclasses import dataclass, field
from typing import Any

MAGIC = 0xDEADBEEF
RECORD_HEADER_FMT = "!IIQ"  # magic(4) + length(4) + lsn(8)
RECORD_HEADER_SIZE = struct.calcsize(RECORD_HEADER_FMT)

@dataclass
class WALRecord:
    lsn: int
    operation: str  # 'INSERT', 'UPDATE', 'DELETE', 'COMMIT', 'ROLLBACK'
    data: dict

class SimpleWAL:
    """교육용 미니 WAL 구현"""
    
    def __init__(self, wal_path: str, data_path: str):
        self.wal_path = wal_path
        self.data_path = data_path
        self._lsn = 0
        self._wal_fd = open(wal_path, 'ab+')
        
        # 데이터 파일 (실제 DB 역할)
        self._data: dict[str, Any] = {}
        if os.path.exists(data_path):
            with open(data_path) as f:
                self._data = json.load(f)
    
    def _write_record(self, operation: str, data: dict) -> int:
        """WAL에 레코드 기록 (항상 append)"""
        self._lsn += 1
        record = WALRecord(lsn=self._lsn, operation=operation, data=data)
        payload = json.dumps({
            "lsn": record.lsn,
            "op": record.operation,
            "data": record.data
        }).encode()
        
        header = struct.pack(RECORD_HEADER_FMT, MAGIC, len(payload), self._lsn)
        self._wal_fd.write(header + payload)
        self._wal_fd.flush()
        os.fsync(self._wal_fd.fileno())  # 핵심: 디스크에 강제 동기화
        return self._lsn
    
    def put(self, key: str, value: Any, txn_id: int) -> int:
        """WAL에 먼저 쓴 뒤, 메모리에 반영"""
        lsn = self._write_record("PUT", {"txn": txn_id, "key": key, "value": value})
        # 메모리(버퍼)에 반영 (아직 데이터 파일에는 미반영)
        self._data[key] = value
        return lsn
    
    def commit(self, txn_id: int) -> int:
        """COMMIT 레코드 기록 — 이 시점에 트랜잭션 내구성 보장"""
        lsn = self._write_record("COMMIT", {"txn": txn_id})
        print(f"  [WAL] COMMIT txn={txn_id} at LSN={lsn} — durability guaranteed!")
        return lsn
    
    def checkpoint(self):
        """WAL 내용을 데이터 파일에 반영 (체크포인트)"""
        print("\n  [CHECKPOINT] Flushing data to disk...")
        with open(self.data_path, 'w') as f:
            json.dump(self._data, f, indent=2)
        print(f"  [CHECKPOINT] Done. Data file updated.")
    
    def recover(self):
        """충돌 복구: WAL을 재생하여 데이터 파일 복원"""
        print("\n  [RECOVERY] Replaying WAL...")
        self._wal_fd.seek(0)
        committed_txns = set()
        records = []
        
        while True:
            header_bytes = self._wal_fd.read(RECORD_HEADER_SIZE)
            if not header_bytes:
                break
            magic, length, lsn = struct.unpack(RECORD_HEADER_FMT, header_bytes)
            if magic != MAGIC:
                print("  [RECOVERY] Corrupted record detected, stopping")
                break
            payload = json.loads(self._wal_fd.read(length))
            records.append(payload)
            if payload["op"] == "COMMIT":
                committed_txns.add(payload["data"]["txn"])
        
        # 커밋된 트랜잭션만 재적용
        for rec in records:
            if rec["op"] == "PUT" and rec["data"]["txn"] in committed_txns:
                self._data[rec["data"]["key"]] = rec["data"]["value"]
                print(f"  [RECOVERY] Applied: {rec['data']['key']} = {rec['data']['value']}")
        
        print(f"  [RECOVERY] Replayed {len(records)} records, "
              f"{len(committed_txns)} committed transactions")
        self.checkpoint()

# 사용 예시
import tempfile
wal_file = tempfile.mktemp(suffix=".wal")
data_file = tempfile.mktemp(suffix=".json")

wal = SimpleWAL(wal_file, data_file)
print("=== 트랜잭션 1: 커밋 ===")
wal.put("user:1", {"name": "Alice", "balance": 1000}, txn_id=1)
wal.put("user:2", {"name": "Bob", "balance": 500}, txn_id=1)
wal.commit(txn_id=1)

print("\n=== 트랜잭션 2: 충돌 (커밋 없음) ===")
wal.put("user:1", {"name": "Alice", "balance": 0}, txn_id=2)
# 여기서 "충돌" 발생 — COMMIT 없음

print("\n=== 시스템 재시작 후 복구 ===")
recovered_wal = SimpleWAL(wal_file, data_file)
recovered_wal.recover()
print(f"\n최종 데이터: {recovered_wal._data}")
# user:1의 balance는 1000 (txn 2의 미커밋 변경은 무시됨)
```

## 주요 설정과 트레이드오프

### fsync 설정의 중요성

PostgreSQL의 `synchronous_commit` 설정은 성능과 내구성 사이의 트레이드오프를 조절합니다:

```sql
-- 가장 안전 (기본값): WAL이 디스크에 완전히 기록될 때까지 대기
SET synchronous_commit = 'on';

-- 성능 향상 (약 ~0.6ms 이내 데이터 손실 가능):
SET synchronous_commit = 'off';

-- 로컬 WAL은 확인, 원격 복제는 확인 안 함:
SET synchronous_commit = 'local';

-- 스탠바이 서버의 WAL 수신까지 대기:
SET synchronous_commit = 'remote_write';
```

### WAL 수준 설정

```sql
SHOW wal_level;
-- minimal: 크래시 복구만
-- replica: 스트리밍 복제 (기본값)
-- logical: 논리 복제 (데이터 변환 가능)
```

## 주의사항 및 팁

**1. full_page_writes 비활성화 주의**  
체크포인트 이후 첫 번째 페이지 수정 시 PostgreSQL은 전체 페이지 이미지를 WAL에 기록합니다(`full_page_writes = on`). 이를 끄면 WAL 크기와 I/O가 줄지만, 하드웨어 수준의 partial write가 발생하면 복구가 불가능해집니다. EBS, ZFS처럼 원자적 쓰기를 보장하는 스토리지에서만 비활성화를 고려하세요.

**2. WAL 파일 크기 조정**  
기본 16MB인 WAL 파일(`wal_segment_size`)은 컴파일 타임에만 변경 가능합니다. 대용량 트랜잭션이 많은 환경에서는 처음 빌드 시 조정하세요.

**3. PITR은 베이스 백업 + WAL 아카이빙**  
Point-in-Time Recovery를 위해서는 `pg_basebackup`으로 베이스 백업을 찍고, `archive_command`로 WAL 파일을 안전한 저장소에 보관해야 합니다.

**4. `wal_compression`으로 I/O 절감**  
PostgreSQL 15+에서는 `wal_compression = lz4` 설정으로 full-page 이미지를 압축하여 WAL I/O를 최대 50% 줄일 수 있습니다.

## 참고 자료

- [PostgreSQL Documentation: Write-Ahead Logging](https://www.postgresql.org/docs/current/wal-intro.html)
- [PostgreSQL Documentation: WAL Configuration](https://www.postgresql.org/docs/current/runtime-config-wal.html)
- [SQLite WAL Mode](https://www.sqlite.org/wal.html)
- [Martin Fowler: Event Sourcing Pattern](https://martinfowler.com/eaaDev/EventSourcing.html)
