---
layout: post
title: "Google Spanner와 TrueTime 완전 정복: 전 세계 규모 분산 데이터베이스의 외부 일관성 구현 원리"
date: 2026-09-17
categories: [cs, computer-science]
tags: [spanner, truetime, distributed-database, external-consistency, linearizability, 분산시스템, 데이터베이스]
---

## 개요

데이터베이스 세계에서 "글로벌 규모의 강한 일관성"은 오랫동안 이론적 불가능에 가까운 목표로 여겨졌다. CAP 정리는 분산 시스템이 일관성(Consistency), 가용성(Availability), 파티션 내성(Partition Tolerance) 중 두 가지만 선택할 수 있다고 명시한다. 그러나 Google은 2012년 OSDI에서 **Spanner**를 발표하며 이 관념에 정면으로 도전했다. Spanner는 전 세계 수십 개의 데이터센터에 분산된 데이터에 대해 **외부 일관성(External Consistency)** — Linearizability라고도 불리는 가장 강한 일관성 보장 — 을 제공한다. 그 핵심에는 원자 시계와 GPS 수신기를 결합한 **TrueTime API**가 있다.

## 왜 필요한가: 분산 시스템에서의 시간 문제

분산 데이터베이스에서 트랜잭션 순서를 결정하는 것은 놀랍도록 어려운 문제다.

### 논리적 클록의 한계

Lamport 타임스탬프나 벡터 클록을 사용하면 이벤트 간의 인과 관계(causality)를 추적할 수 있다. 하지만 이것으로는 부족하다. 두 트랜잭션이 서로 인과 관계가 없는 경우(concurrent transactions), 전역적인 순서를 결정할 수 없기 때문이다.

예를 들어, 서울 데이터센터에서 트랜잭션 T1이 완료되고, 그 결과를 본 사용자가 도쿄 데이터센터에서 트랜잭션 T2를 시작했다고 하자. T2는 T1에 causally 의존하지만, 논리 클록만으로는 T1의 커밋 타임스탬프가 T2의 시작 타임스탬프보다 앞서도록 보장할 방법이 없다.

### 물리 시계의 한계

NTP(Network Time Protocol)로 동기화된 물리 시계를 사용하면 어떨까? 실제로 NTP는 수 밀리초에서 수십 밀리초의 오차를 허용하므로, 서로 다른 서버의 시계가 역전될 수 있다. 서버 A가 트랜잭션을 t=100ms에 커밋했는데, 서버 B의 시계가 살짝 빠르게 흘러 t=98ms라고 기록할 수 있다. 이 경우 B에서 t=99ms에 시작한 트랜잭션이 A의 트랜잭션보다 앞선 것으로 보일 수 있어 일관성이 깨진다.

Spanner는 이 문제를 TrueTime을 통해 해결한다.

## TrueTime: 시간 불확실성을 명시적으로 다루기

TrueTime의 핵심 아이디어는 단순하다: **시간을 단일 값이 아니라 구간(interval)으로 표현한다.**

### TrueTime API

```
TT.now()   → TTinterval { earliest: t_lo, latest: t_hi }
TT.after(t) → bool  // t가 확실히 과거인지
TT.before(t) → bool // t가 확실히 미래인지
```

`TT.now()`를 호출하면 `[t_lo, t_hi]` 구간이 반환된다. 이 구간의 의미는 **실제 절대 시각이 이 구간 안에 반드시 존재한다**는 것이다. Google의 데이터센터에서 이 구간의 폭(epsilon, ε)은 통상 1~7ms 수준이다.

### 하드웨어 인프라

각 Google 데이터센터는 **타임마스터(timemaster)** 서버를 운영하며, 이 서버는 두 종류의 하드웨어를 사용한다:

- **GPS 수신기**: 나노초 수준의 정확도로 UTC 시간을 제공하지만, 건물 내 신호 차단이나 재밍(jamming) 공격에 취약
- **원자 시계(Atomic Clock)**: GPS 신호 없이도 수십 마이크로초/초 이하의 드리프트로 작동. GPS 장애 시 약 수천 초간 ε을 작게 유지 가능

각 서버 데몬은 주기적으로 여러 타임마스터와 동기화하며, 측정 불확실성과 통신 RTT를 합산하여 ε를 계산한다.

## Spanner의 트랜잭션 프로토콜

### 외부 일관성 정의

트랜잭션 T1이 T2보다 먼저 커밋되었다면, T2의 커밋 타임스탬프는 T1의 커밋 타임스탬프보다 반드시 크다:

```
commit(T1) happens-before commit(T2)
  ⟹ s(T1) < s(T2)
```

여기서 s(T)는 트랜잭션 T에 할당된 타임스탬프다.

### 읽기-쓰기 트랜잭션 (Commit Wait)

Spanner가 외부 일관성을 보장하는 핵심 메커니즘은 **Commit Wait**이다. 트랜잭션 커밋 과정은 다음과 같다:

1. **Prepare 단계**: 2PC(Two-Phase Commit)의 prepare 메시지를 관련 파티션에 전송
2. **타임스탬프 선택**: 코디네이터 리더가 `s = TT.now().latest`를 커밋 타임스탬프로 선택
3. **Commit Wait**: `TT.after(s)`가 참이 될 때까지 대기 (즉, 실제 시각이 s를 넘을 때까지 대기)
4. **커밋 실행**: Commit 메시지를 전송하고 결과를 클라이언트에 반환

```python
# Commit Wait 슈도코드
def commit_transaction(txn):
    prepare_all_participants(txn)
    
    s = TT.now().latest  # 가장 늦은 가능 시각을 타임스탬프로
    
    # 실제 시각이 s를 확실히 지날 때까지 대기
    while not TT.after(s):
        time.sleep(1_ms)
    
    apply_commit(txn, timestamp=s)
    return s
```

왜 이 대기가 외부 일관성을 보장할까?

T1이 커밋을 완료하는 **절대 시각**을 t_commit1이라 하면, T1은 `TT.after(s1)`이 참일 때만 커밋을 완료하므로:
```
t_commit1 ≥ s1
```

그 후 T2가 시작되면, T2의 코디네이터는 `s2 = TT.now().latest ≥ TT.now().earliest ≥ 현재_절대시각 ≥ t_commit1 ≥ s1`이 되어:
```
s2 > s1
```

외부 일관성이 성립한다.

### 읽기 전용 트랜잭션 (Lock-Free Snapshot Read)

Spanner의 또 다른 혁신은 **잠금 없는 읽기 전용 트랜잭션**이다.

각 데이터 항목은 다중 버전으로 저장되며, 각 버전에는 쓰기 트랜잭션의 커밋 타임스탬프가 붙는다. 읽기 전용 트랜잭션은 `s_read = TT.now().earliest`를 선택하고, 이 타임스탬프 이하의 최신 버전을 읽는다.

```python
# 읽기 전용 트랜잭션
def read_only_txn(keys):
    s_read = TT.now().earliest  # 가장 이른 가능 시각
    results = {}
    for key in keys:
        # s_read 이하의 가장 최신 커밋 버전 읽기
        results[key] = read_at_timestamp(key, s_read)
    return results
```

이 방식은 **잠금이 전혀 필요 없으므로** 읽기 트랜잭션이 쓰기 트랜잭션을 차단하지 않는다. MVCC(Multi-Version Concurrency Control)를 TrueTime 타임스탬프로 구현한 것이다.

## Spanner 아키텍처

### 계층 구조

```
Universe (전 세계 Spanner 배포)
  └─ Zone (하나의 데이터센터)
       ├─ Zonemaster: Tablet 할당 관리
       ├─ Location Proxy: Tablet 위치 탐색
       └─ Spanserver (수백~수천 대)
            ├─ Tablet (데이터 파티션)
            │    └─ Paxos State Machine
            └─ Lock Table, Transaction Manager
```

### Paxos를 통한 복제

각 데이터 파티션(Tablet)은 여러 Zone에 Paxos 그룹으로 복제된다. 보통 5개의 복제본을 유지하며, 리더 Spanserver가 읽기/쓰기를 처리한다.

**디렉토리(Directory)**: Spanner는 연속된 키를 가진 데이터를 디렉토리 단위로 그룹화하며, 같은 디렉토리의 데이터는 항상 같은 Paxos 그룹에 위치한다. 이 설계로 단일 데이터센터 내 트랜잭션이 분산 2PC 없이 처리될 수 있다.

## 실제 구현 예제

### 1. Spanner 데이터 모델과 스키마

Spanner는 계층형 테이블 구조를 지원한다. 부모-자식 관계의 테이블은 물리적으로 같은 디렉토리에 저장(인터리빙)될 수 있어 JOIN 성능이 향상된다.

```sql
-- 부모 테이블
CREATE TABLE Singers (
  SingerId   INT64 NOT NULL,
  FirstName  STRING(1024),
  LastName   STRING(1024),
  BirthDate  DATE,
) PRIMARY KEY (SingerId);

-- 자식 테이블 (인터리빙)
CREATE TABLE Albums (
  SingerId     INT64 NOT NULL,
  AlbumId      INT64 NOT NULL,
  AlbumTitle   STRING(MAX),
) PRIMARY KEY (SingerId, AlbumId),
  INTERLEAVE IN PARENT Singers ON DELETE CASCADE;
-- Albums의 데이터가 Singers와 물리적으로 인접 저장됨
```

### 2. Go 클라이언트로 Spanner 트랜잭션 사용

```go
package main

import (
    "context"
    "fmt"
    "log"

    "cloud.google.com/go/spanner"
    "google.golang.org/api/iterator"
)

func transferBalance(ctx context.Context, client *spanner.Client,
    fromAccount, toAccount string, amount int64) error {
    
    // 읽기-쓰기 트랜잭션: 외부 일관성 보장
    _, err := client.ReadWriteTransaction(ctx, func(ctx context.Context, txn *spanner.ReadWriteTransaction) error {
        // 잔액 읽기
        fromRow, err := txn.ReadRow(ctx, "Accounts",
            spanner.Key{fromAccount}, []string{"Balance"})
        if err != nil {
            return err
        }
        var fromBalance int64
        if err := fromRow.Columns(&fromBalance); err != nil {
            return err
        }

        if fromBalance < amount {
            return fmt.Errorf("insufficient balance: %d < %d", fromBalance, amount)
        }

        // 잔액 차감 및 추가 (원자적 실행, Commit Wait 포함)
        txn.BufferWrite([]*spanner.Mutation{
            spanner.Update("Accounts",
                []string{"AccountId", "Balance"},
                []interface{}{fromAccount, fromBalance - amount}),
            spanner.Update("Accounts",
                []string{"AccountId", "Balance"},
                []interface{}{toAccount, spanner.CommitTimestamp}), // 갱신
        })
        return nil
    })
    return err
}

func readAtTimestamp(ctx context.Context, client *spanner.Client, t time.Time) {
    // 스냅샷 읽기: 잠금 불필요, 과거 시점 읽기
    ro := client.ReadOnlyTransaction().WithTimestampBound(
        spanner.ReadTimestamp(t)) // 특정 시각 기준
    defer ro.Close()

    iter := ro.Read(ctx, "Singers", spanner.AllKeys(), []string{"SingerId", "FirstName"})
    defer iter.Stop()
    for {
        row, err := iter.Next()
        if err == iterator.Done {
            break
        }
        if err != nil {
            log.Fatal(err)
        }
        var id int64
        var name string
        row.Columns(&id, &name)
        fmt.Printf("Singer %d: %s\n", id, name)
    }
}
```

### 3. TrueTime 기반 Commit Wait 시뮬레이션

실제 TrueTime API는 Google 내부 전용이지만, 개념을 시뮬레이션할 수 있다:

```python
import time
import random
from dataclasses import dataclass

@dataclass
class TTInterval:
    earliest: float  # Unix timestamp
    latest: float    # Unix timestamp

class SimulatedTrueTime:
    """epsilon(ε)을 고려한 TrueTime 시뮬레이션"""
    
    def __init__(self, epsilon_ms=4):
        self.epsilon = epsilon_ms / 1000.0  # 초 단위
    
    def now(self) -> TTInterval:
        """현재 시각을 불확실성 구간으로 반환"""
        t = time.time()
        # 실제 시각은 [t - ε/2, t + ε/2] 구간 어딘가
        return TTInterval(
            earliest=t - self.epsilon / 2,
            latest=t + self.epsilon / 2
        )
    
    def after(self, t: float) -> bool:
        """t가 확실히 과거인지 (현재 최소 가능 시각 > t)"""
        return self.now().earliest > t
    
    def before(self, t: float) -> bool:
        """t가 확실히 미래인지 (현재 최대 가능 시각 < t)"""
        return self.now().latest < t

class SpannerCoordinator:
    def __init__(self, truetime: SimulatedTrueTime):
        self.truetime = truetime
        self.committed_txns = []
    
    def commit_rw_transaction(self, txn_id: str, mutations: list) -> float:
        """
        읽기-쓰기 트랜잭션 커밋:
        1. 타임스탬프 = TT.now().latest
        2. Commit Wait: TT.after(s)가 될 때까지 대기
        """
        # 타임스탬프 선택
        s = self.truetime.now().latest
        print(f"[{txn_id}] 커밋 타임스탬프 선택: {s:.6f}")
        
        # Commit Wait
        wait_start = time.time()
        while not self.truetime.after(s):
            time.sleep(0.001)  # 1ms 대기
        
        wait_duration = time.time() - wait_start
        print(f"[{txn_id}] Commit Wait 완료 (대기: {wait_duration*1000:.1f}ms)")
        
        # 실제 커밋
        self.committed_txns.append({
            'txn_id': txn_id,
            'timestamp': s,
            'mutations': mutations
        })
        
        return s

# 외부 일관성 검증
def verify_external_consistency():
    tt = SimulatedTrueTime(epsilon_ms=4)
    coord = SpannerCoordinator(tt)
    
    # T1 먼저 커밋
    s1 = coord.commit_rw_transaction("T1", [{"key": "A", "value": 1}])
    
    # T2는 T1 커밋 후 시작 (인과 관계 존재)
    time.sleep(0.001)  # 약간의 처리 시간
    s2 = coord.commit_rw_transaction("T2", [{"key": "B", "value": 2}])
    
    print(f"\n외부 일관성 검증: s1={s1:.6f}, s2={s2:.6f}")
    print(f"s1 < s2: {s1 < s2}")  # 반드시 True

verify_external_consistency()
```

## Spanner vs. 전통적 분산 데이터베이스

| 특성 | 전통 RDBMS (클러스터) | Spanner |
|------|----------------------|---------|
| 일관성 모델 | 강한 일관성 (단일 리전) | 외부 일관성 (글로벌) |
| 확장 방식 | 수직 확장 위주 | 수평 자동 분산 |
| 트랜잭션 | ACID | ACID + 글로벌 |
| 읽기 지연 | 수ms | 수ms~수십ms (리전 내) |
| 쓰기 지연 | 수ms | 수ms + Commit Wait (5~10ms) |
| 가용성 SLA | 99.9% | 99.999% |

## 주의사항과 Commit Wait의 비용

### 쓰기 지연 증가

Commit Wait 때문에 읽기-쓰기 트랜잭션의 최소 커밋 지연은 약 **ε (4~14ms)**이다. Google은 이를 "외부 일관성의 가격"으로 명시한다. 대부분의 애플리케이션에서 이 지연은 허용 가능하지만, 초저지연이 요구되는 시스템에서는 고려가 필요하다.

### Hotspot 방지

모든 트랜잭션이 동일한 키에 집중되면 단일 Paxos 리더에 부하가 집중된다. Spanner는 자동 분산을 지원하지만, 스키마 설계 시 순차 증가 기본키보다 **해시 기반 키**를 사용하는 것이 좋다.

```sql
-- 나쁜 예: 순차 ID는 핫스팟 유발
CREATE TABLE Events (
  EventId INT64 NOT NULL,  -- 순차 증가
  ...
) PRIMARY KEY (EventId);

-- 좋은 예: UUID나 해시 기반 키
CREATE TABLE Events (
  EventId STRING(36) NOT NULL,  -- UUID
  ...
) PRIMARY KEY (EventId);
```

### Stale Read로 성능 최적화

외부 일관성이 필요하지 않은 읽기는 **Stale Read**를 사용해 지연과 부하를 줄일 수 있다. 15초 이전 스냅샷 읽기는 Paxos 리더가 아닌 팔로워에서도 처리 가능하다:

```go
// 15초 이전 데이터로 충분한 경우
ro := client.ReadOnlyTransaction().WithTimestampBound(
    spanner.ExactStaleness(15 * time.Second))
```

## 참고 자료
- [Spanner: Google's Globally Distributed Database (OSDI 2012)](https://www.usenix.org/conference/osdi12/technical-sessions/presentation/corbett)
- [Google Cloud Spanner 공식 문서](https://cloud.google.com/spanner/docs)
- [Spanner, TrueTime and the CAP Theorem (Google 공식 블로그)](https://cloud.google.com/blog/products/databases/inside-cloud-spanner-and-the-cap-theorem)
- [Spanner: Becoming a SQL System (SIGMOD 2017)](https://dl.acm.org/doi/10.1145/3035918.3056103)
