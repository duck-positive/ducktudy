---
layout: post
title: "ARIES 트랜잭션 복구 알고리즘: Steal/No-Force 정책과 3단계 복구"
date: 2026-08-22
categories: [cs, computer-science]
tags: [aries, database, recovery, wal, transaction, steal, no-force, lsn, checkpoint]
---

데이터베이스 시스템이 갑작스러운 장애(크래시, 정전 등)로 인해 종료되었을 때, 시스템을 일관된 상태로 복원하는 것은 핵심 과제입니다. IBM의 C. Mohan이 1992년에 발표한 **ARIES**(Algorithms for Recovery and Isolation Exploiting Semantics)는 오늘날 PostgreSQL, DB2, SQL Server 등 대부분의 상용 DBMS가 직접 채택하거나 변형하여 사용하는 표준 복구 알고리즘입니다.

---

## 왜 ARIES가 필요한가?

데이터베이스 복구의 핵심 목표는 **ACID의 원자성(Atomicity)과 내구성(Durability)** 을 장애 상황에서도 보장하는 것입니다.

장애 발생 시 두 가지 상황이 문제가 됩니다:

1. **커밋되지 않은 트랜잭션의 변경이 디스크에 기록된 경우** → 롤백(Undo) 필요
2. **커밋된 트랜잭션의 변경이 디스크에 반영되지 않은 경우** → 재실행(Redo) 필요

단순하게 생각하면 "커밋 시점에 모든 데이터를 디스크에 강제 기록(Force)하고, 커밋 전에는 절대 디스크에 쓰지 않는다(No-Steal)"는 전략을 쓸 수도 있습니다. 하지만 이 방식은 성능이 너무 나쁩니다.

ARIES는 성능과 안전성 모두를 잡기 위해 **Steal + No-Force** 정책을 사용합니다.

---

## 버퍼 관리 정책: Steal과 Force

### Steal vs. No-Steal

- **No-Steal**: 커밋되지 않은 트랜잭션의 변경 페이지를 버퍼에만 유지하고 디스크에 내보내지 않음. 안전하지만 메모리를 낭비하고 버퍼 부족 시 문제가 생김.
- **Steal**: 버퍼 매니저가 필요에 따라 커밋되지 않은 페이지를 디스크에 내보낼 수 있음. 메모리 효율이 높지만 크래시 시 디스크에 미완료 변경이 남아 Undo가 필요.

### Force vs. No-Force

- **Force**: 커밋 시 해당 트랜잭션이 변경한 모든 페이지를 즉시 디스크에 기록. 내구성을 보장하지만 동기적 랜덤 I/O가 많아 느림.
- **No-Force**: 커밋 후에도 페이지가 메모리에 머물 수 있음. I/O를 배치 처리하여 성능이 좋지만 크래시 시 Redo가 필요.

ARIES는 최고 성능을 위해 **Steal + No-Force** 조합을 채택하고, 이로 인한 Undo/Redo 필요성을 WAL 로그로 해결합니다.

---

## Write-Ahead Logging(WAL) 원칙

ARIES의 기반은 **WAL(Write-Ahead Logging)** 입니다. 두 가지 핵심 규칙이 있습니다:

1. **Undo 규칙**: 데이터 페이지를 디스크에 쓰기 전에 해당 변경에 대한 Undo 로그 레코드를 먼저 디스크에 기록해야 한다.
2. **Redo 규칙**: 트랜잭션 커밋 전에 해당 트랜잭션의 모든 로그 레코드를 디스크에 기록해야 한다.

로그는 절대 제자리에서 수정되지 않고(append-only) 순차적으로 기록되어 I/O 효율이 매우 높습니다.

---

## LSN(Log Sequence Number)

모든 로그 레코드는 단조 증가하는 **LSN(Log Sequence Number)** 을 가집니다. LSN은 복구의 핵심 역할을 합니다:

```
로그 레코드 구조:
┌─────┬─────────────┬────────────┬───────────┬─────────────────────────┐
│ LSN │ Transaction │  Prev LSN  │ Operation │ Before/After Image (데이터) │
│     │     ID      │ (이전 LSN)  │ (타입)     │                         │
└─────┴─────────────┴────────────┴───────────┴─────────────────────────┘
```

**pageLSN**: 각 데이터 페이지에는 해당 페이지를 마지막으로 수정한 로그 레코드의 LSN이 저장됩니다. Redo 시 `pageLSN >= record LSN`이면 해당 변경은 이미 반영된 것이므로 건너뜁니다(멱등성).

---

## ARIES 3단계 복구 과정

크래시 후 재시작 시 ARIES는 다음 세 단계를 순서대로 실행합니다.

### 1단계: Analysis (분석)

체크포인트부터 로그 끝까지 순방향으로 스캔하여 두 가지 정보를 구축합니다:

- **ATT(Active Transaction Table)**: 크래시 시점에 아직 커밋되지 않은 트랜잭션 목록
- **DPT(Dirty Page Table)**: 크래시 시점에 메모리에는 변경되었지만 디스크에 아직 반영되지 않은 페이지 목록과 각 페이지의 `recLSN`(해당 페이지를 처음 더티하게 만든 LSN)

### 2단계: Redo (재실행)

DPT의 가장 작은 `recLSN`부터 로그 끝까지 순방향으로 스캔하며, 디스크의 `pageLSN`보다 큰 LSN을 가진 모든 변경을 재실행합니다. 이를 통해 크래시 직전 상태를 완전히 재현합니다.

**중요**: Redo는 커밋 여부와 무관하게 모든 변경을 재실행합니다. 이후 Undo 단계에서 미완료 트랜잭션을 정리합니다.

### 3단계: Undo (롤백)

ATT에 있는 미완료 트랜잭션을 `prevLSN` 체인을 역방향으로 따라가며 모두 Undo합니다. Undo 시에는 **CLR(Compensation Log Record)** 를 기록하여 Undo 자체가 재시작되어도 안전하게 처리됩니다.

```
로그 체인 예시:
LSN 100: BEGIN T1
LSN 110: UPDATE page_5, T1 (prevLSN=100, before=A, after=B)
LSN 120: BEGIN T2
LSN 130: UPDATE page_7, T2 (prevLSN=120, before=C, after=D)
LSN 140: COMMIT T2
LSN 150: UPDATE page_3, T1 (prevLSN=110, before=E, after=F)
--- CRASH ---

Analysis: ATT={T1}, DPT={page_5: recLSN=110, page_3: recLSN=150}
Redo:     LSN 110 ~ 150 재실행
Undo:     T1의 LSN 150 → CLR(undo page_3), LSN 110 → CLR(undo page_5)
```

---

## 코드 예제 1: 간단한 WAL 시뮬레이터 (Python)

```python
import json
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class LogRecord:
    lsn: int
    txn_id: str
    prev_lsn: Optional[int]
    op: str  # 'begin', 'update', 'commit', 'abort', 'clr'
    page_id: Optional[str] = None
    before: Optional[str] = None
    after: Optional[str] = None
    undo_next_lsn: Optional[int] = None  # CLR용

class WALManager:
    def __init__(self):
        self.log: list[LogRecord] = []
        self.next_lsn = 1
        self.txn_table: dict[str, int] = {}  # txn_id -> last_lsn
        self.buffer_pool: dict[str, dict] = {}  # page_id -> {data, page_lsn}
        self.disk: dict[str, dict] = {}         # page_id -> {data, page_lsn}

    def _append(self, record: LogRecord):
        self.log.append(record)
        print(f"  [LOG] LSN={record.lsn} txn={record.txn_id} op={record.op}"
              f"{f' page={record.page_id}' if record.page_id else ''}"
              f"{f' {record.before!r}->{record.after!r}' if record.before else ''}")
        return record.lsn

    def begin(self, txn_id: str) -> int:
        lsn = self.next_lsn; self.next_lsn += 1
        rec = LogRecord(lsn, txn_id, None, 'begin')
        self.txn_table[txn_id] = lsn
        return self._append(rec)

    def update(self, txn_id: str, page_id: str, new_val: str) -> int:
        # 페이지를 버퍼에 로드
        if page_id not in self.buffer_pool:
            self.buffer_pool[page_id] = self.disk.get(page_id, {'data': 'NULL', 'page_lsn': 0}).copy()
        old_val = self.buffer_pool[page_id]['data']
        lsn = self.next_lsn; self.next_lsn += 1
        rec = LogRecord(lsn, txn_id, self.txn_table[txn_id], 'update',
                        page_id, old_val, new_val)
        # WAL 원칙: 로그 먼저, 그 다음 버퍼 수정
        self._append(rec)
        self.buffer_pool[page_id] = {'data': new_val, 'page_lsn': lsn}
        self.txn_table[txn_id] = lsn
        return lsn

    def commit(self, txn_id: str) -> int:
        lsn = self.next_lsn; self.next_lsn += 1
        rec = LogRecord(lsn, txn_id, self.txn_table[txn_id], 'commit')
        self._append(rec)
        # 커밋 로그가 디스크에 도달했다고 가정 (No-Force이므로 데이터 페이지는 안 써도 됨)
        del self.txn_table[txn_id]
        return lsn

    def flush_page(self, page_id: str):
        """버퍼의 페이지를 디스크에 내보냄 (Steal 가능)"""
        if page_id in self.buffer_pool:
            self.disk[page_id] = self.buffer_pool[page_id].copy()
            print(f"  [DISK] page={page_id} flushed (page_lsn={self.disk[page_id]['page_lsn']})")

    def recover(self):
        print("\n=== ARIES 복구 시작 ===")
        # Analysis: 로그 전체 스캔
        att = {}   # txn_id -> last_lsn
        dpt = {}   # page_id -> rec_lsn

        print("\n[1단계: Analysis]")
        for rec in self.log:
            if rec.op == 'begin':
                att[rec.txn_id] = rec.lsn
            elif rec.op in ('update', 'clr'):
                att[rec.txn_id] = rec.lsn
                if rec.page_id and rec.page_id not in dpt:
                    page_lsn = self.disk.get(rec.page_id, {}).get('page_lsn', 0)
                    if page_lsn < rec.lsn:
                        dpt[rec.page_id] = rec.lsn
            elif rec.op == 'commit':
                att.pop(rec.txn_id, None)
        print(f"  ATT: {att}")
        print(f"  DPT: {dpt}")

        # Redo
        print("\n[2단계: Redo]")
        redo_start = min(dpt.values()) if dpt else self.next_lsn
        for rec in self.log:
            if rec.lsn < redo_start or rec.op not in ('update',):
                continue
            page = self.disk.get(rec.page_id, {'data': 'NULL', 'page_lsn': 0})
            if page['page_lsn'] < rec.lsn:
                print(f"  Redo LSN={rec.lsn}: page={rec.page_id} {rec.before!r}->{rec.after!r}")
                self.disk[rec.page_id] = {'data': rec.after, 'page_lsn': rec.lsn}

        # Undo
        print("\n[3단계: Undo]")
        # 각 미완료 트랜잭션의 마지막 LSN부터 역방향 Undo
        undo_set = {txn: last_lsn for txn, last_lsn in att.items()}
        lsn_map = {rec.lsn: rec for rec in self.log}
        while undo_set:
            # 가장 큰 LSN부터 처리
            txn_id = max(undo_set, key=lambda t: undo_set[t])
            lsn = undo_set[txn_id]
            rec = lsn_map[lsn]
            if rec.op == 'update':
                print(f"  Undo LSN={rec.lsn}: page={rec.page_id} {rec.after!r}->{rec.before!r} (CLR 기록)")
                if rec.page_id in self.disk:
                    self.disk[rec.page_id] = {'data': rec.before, 'page_lsn': rec.lsn}
            if rec.prev_lsn is not None:
                undo_set[txn_id] = rec.prev_lsn
            else:
                del undo_set[txn_id]
        print("\n=== 복구 완료 ===")
        print("디스크 상태:", {k: v['data'] for k, v in self.disk.items()})


# 시뮬레이션
wal = WALManager()
print("=== 정상 실행 ===")
wal.begin("T1")
wal.update("T1", "page_A", "Hello")
wal.begin("T2")
wal.update("T2", "page_B", "World")
wal.flush_page("page_A")  # Steal: T1이 아직 커밋 안 했지만 디스크에 내보냄
wal.commit("T2")
wal.update("T1", "page_C", "Temp")
# T1은 커밋 안 된 상태로 크래시

wal.recover()
```

---

## 코드 예제 2: Checkpoint 구현 (Java 스타일 의사코드)

```java
// ARIES 퍼지 체크포인트 (Fuzzy Checkpoint)
// 체크포인트 시 버퍼를 모두 비울 필요 없이 ATT와 DPT만 로그에 기록

public class ARIESCheckpoint {

    // BEGIN_CHECKPOINT 레코드 기록 후 END_CHECKPOINT에 스냅샷 저장
    public void takeCheckpoint(LogManager log, BufferPool buffer) {
        // 1. BEGIN_CHECKPOINT 로그 기록 (즉시, 버퍼 플러시 없이)
        long beginLSN = log.append(new LogRecord(
            LogType.BEGIN_CHECKPOINT, null, null, null, null
        ));

        // 2. 현재 ATT 스냅샷 (진행 중인 트랜잭션)
        Map<String, Long> attSnapshot = txnTable.snapshot();

        // 3. 현재 DPT 스냅샷 (더티 페이지 목록과 recLSN)
        Map<PageId, Long> dptSnapshot = buffer.getDirtyPageTable();

        // 4. END_CHECKPOINT 로그 기록 (ATT + DPT 포함)
        long endLSN = log.append(new LogRecord(
            LogType.END_CHECKPOINT, null, attSnapshot, dptSnapshot, null
        ));

        // 5. master record에 BEGIN_CHECKPOINT LSN 기록
        //    (재시작 시 여기서부터 Analysis를 시작)
        masterRecord.setCheckpointLSN(beginLSN);
        log.flush(endLSN);  // END_CHECKPOINT만 디스크에 플러시

        System.out.printf("Checkpoint: beginLSN=%d, endLSN=%d, ATT=%s%n",
            beginLSN, endLSN, attSnapshot.keySet());
    }

    // 복구 시 체크포인트 정보로 ATT/DPT 초기화
    public RecoveryState loadCheckpoint(LogManager log) {
        long ckptLSN = masterRecord.getCheckpointLSN();
        LogRecord beginRec = log.read(ckptLSN);
        // BEGIN_CHECKPOINT에서 END_CHECKPOINT를 찾아 ATT, DPT 로드
        LogRecord endRec = log.findEndCheckpoint(ckptLSN);

        Map<String, Long> att = new HashMap<>(endRec.attSnapshot);
        Map<PageId, Long> dpt = new HashMap<>(endRec.dptSnapshot);
        System.out.printf("체크포인트 로드: %d개 진행 중 트랜잭션, %d개 더티 페이지%n",
            att.size(), dpt.size());
        return new RecoveryState(ckptLSN, att, dpt);
    }
}
```

---

## 주의사항 및 팁

### CLR(Compensation Log Record)의 중요성
Undo 단계에서 시스템이 다시 크래시되면 Undo 작업도 재실행해야 합니다. CLR은 Undo된 변경을 로그로 남겨 재시작 시 다시 Undo하지 않게 합니다. CLR은 자신보다 이전의 `undoNextLSN`을 가리켜 이미 Undo된 레코드를 건너뜁니다.

### Redo의 멱등성(Idempotency)
ARIES의 Redo는 `pageLSN`을 통해 중복 적용을 방지합니다. 동일한 Redo를 여러 번 수행해도 결과가 동일합니다. 이 덕분에 복구 도중 다시 크래시가 발생해도 안전합니다.

### 세분화된 잠금(Fine-Granularity Locking)
ARIES는 페이지 단위가 아닌 레코드(행) 단위 잠금을 지원합니다. 로그 레코드에 물리적 변경뿐 아니라 논리적 연산 정보도 저장하여 잠금 범위를 최소화합니다.

### 실무 적용
- **PostgreSQL**: WAL 기반 복구를 사용하며 ARIES와 유사하지만 독자적인 구현
- **InnoDB(MySQL)**: 더블 라이트 버퍼와 결합하여 partial write 문제도 해결
- **RocksDB**: LSM 트리 특성상 ARIES 대신 WAL + Compaction 기반 복구 사용

---

## 참고 자료

- [Write-ahead logging and the ARIES crash recovery algorithm — Kevin Sookocheff](https://sookocheff.com/post/databases/write-ahead-logging/)
- [Database Recovery Demystified: Understanding ARIES from First Principles](https://yashagw.github.io/blog/db-recovery/)
- [ARIES: A Transaction Recovery Method — the morning paper (Adrian Colyer)](https://blog.acolyer.org/2016/01/08/aries/)
- [Crash Recovery — WAL and ARIES — Socratopia](https://www.socratopia.app/library/database-systems-en/chapter-17)
