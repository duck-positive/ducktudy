---
layout: post
title: "하이브리드 논리 클락 완전 정복: 분산 시스템에서 인과율과 물리 시간을 동시에 잡는 방법"
date: 2026-10-23
categories: [cs, computer-science]
tags: [distributed-systems, clocks, hybrid-logical-clock, HLC, causality, CockroachDB, vector-clock, lamport-clock]
---

## 개요

분산 시스템에서 "이 이벤트가 저 이벤트보다 먼저 발생했는가?"를 판단하는 일은 예상보다 훨씬 어렵습니다. 각 서버의 물리 시계는 클럭 드리프트로 미묘하게 다르고, NTP로 동기화해도 수십~수백 밀리초의 오차가 남습니다. 반면 Lamport 클락이나 벡터 클락 같은 논리 클락은 **인과관계(causality)**를 정확히 추적하지만, 사람이 이해하는 실제 시간과 동떨어져 디버깅이 어렵습니다.

**하이브리드 논리 클락(HLC, Hybrid Logical Clocks)**은 2014년 Kulkarni, Demirbas 등이 제안한 방식으로, 논리 클락의 인과성 보장과 물리 클락의 직관적인 타임스탬프를 동시에 달성합니다. CockroachDB, YugabyteDB, MongoDB 클러스터 타임이 HLC를 채택하고 있습니다.

---

## 1. 배경: 기존 시계 방식의 한계

### 1.1 물리 시계 (Physical Clocks): 시간은 알지만 인과성이 없다

NTP(Network Time Protocol)로 동기화된 물리 시계는 실제 시간을 나타내지만, 동기화 오차와 클럭 드리프트로 인해 분산 시스템에서 **이벤트 순서를 신뢰할 수 없습니다**.

```
서버 A: event₁ @ 10:00:00.050  →  서버 B에 전달
서버 B의 시계: 10:00:00.040     ← 동기화 지연으로 A보다 10ms 느림
서버 B: event₂ @ 10:00:00.045  (B 기준으로는 event₁보다 이전!)
```

Google Spanner는 TrueTime API(원자시계+GPS)로 이 문제를 해결하지만, 일반 시스템에서는 불가능합니다.

### 1.2 Lamport 클락: 인과성은 있지만 시간이 없다

Lamport(1978)의 논리 클락은 다음 규칙으로 인과율을 보장합니다:
- 이벤트 발생 시: `l ← l + 1`
- 메시지 전송 시: `l ← l + 1`, 타임스탬프 `l`을 첨부
- 메시지 수신 시: `l ← max(l, 수신_타임스탬프) + 1`

**Lamport 클락이 같거나 작으면 인과적으로 앞선다**는 의미이지만, 역은 성립하지 않습니다. 또한 값이 물리 시간과 전혀 관련이 없어 "언제 발생한 이벤트인지" 알 수 없습니다.

### 1.3 HLC의 목표

| 속성 | 물리 시계 | Lamport 클락 | HLC |
|------|-----------|-------------|-----|
| 인과성 추적 | ✗ | ✓ | ✓ |
| 물리 시간 근사 | ✓ | ✗ | ✓ |
| NTP 호환 64비트 표현 | ✓ | ✗ | ✓ |
| 추가 상태 | 없음 | 카운터 1개 | 카운터 1개 |

---

## 2. HLC 알고리즘 설계

### 2.1 HLC 타임스탬프 구조

HLC 타임스탬프는 `(l, c)` 쌍으로 표현됩니다:
- **l (logical physical time)**: 물리 시간과 동기화된 값. 현재 물리 시각보다 절대 미래로 가지 않습니다. NTP 시간과의 최대 오차 범위 `ε`을 벗어나지 않도록 설계합니다.
- **c (counter)**: 물리 시간이 같은 이벤트 간의 인과 순서를 구별하는 카운터.

비교 규칙: `(l₁, c₁) < (l₂, c₂)` iff `l₁ < l₂` 또는 (`l₁ = l₂` 이고 `c₁ < c₂`)

### 2.2 세 가지 연산

**이벤트 발생 (local event)**
```
l' = max(l, pt())   # pt(): 현재 물리 시각
if l' == l:
    c = c + 1
else:
    l = l'; c = 0
```

**메시지 전송**:
먼저 이벤트 발생 연산을 수행한 후 현재 `(l, c)`를 메시지에 첨부합니다.

**메시지 수신 (수신 타임스탬프 l_m, c_m)**:
```
l' = max(l, l_m, pt())
if l' == l == l_m:
    c = max(c, c_m) + 1
elif l' == l:
    c = c + 1
elif l' == l_m:
    c = c_m + 1
else:
    c = 0
l = l'
```

핵심: `l`이 물리 시간에서 너무 멀리 달아나면 그 노드는 오류를 발생시켜야 합니다 (시스템 보호 장치).

---

## 3. Python 구현

### 예제 1: HLC 기본 구현

```python
import time
import threading
from dataclasses import dataclass, field
from typing import Optional

@dataclass(order=True)
class HLCTimestamp:
    l: int  # 물리 시간 (나노초 단위)
    c: int  # 논리 카운터

    def __repr__(self):
        import datetime
        t = datetime.datetime.fromtimestamp(self.l / 1e9)
        return f"HLC({t.strftime('%H:%M:%S.%f')}, c={self.c})"

class HybridLogicalClock:
    """Kulkarni & Demirbas (2014) HLC 구현"""

    MAX_DRIFT_NS = 500_000_000  # NTP 오차 허용 범위: 500ms

    def __init__(self, node_id: str):
        self.node_id = node_id
        self._l: int = 0
        self._c: int = 0
        self._lock = threading.Lock()

    def _physical_time_ns(self) -> int:
        return int(time.time() * 1e9)

    @property
    def timestamp(self) -> HLCTimestamp:
        with self._lock:
            return HLCTimestamp(self._l, self._c)

    def tick(self) -> HLCTimestamp:
        """로컬 이벤트 발생 — 타임스탬프 갱신 후 반환"""
        with self._lock:
            pt = self._physical_time_ns()
            l_prime = max(self._l, pt)

            if l_prime == self._l:
                self._c += 1
            else:
                self._l = l_prime
                self._c = 0

            return HLCTimestamp(self._l, self._c)

    def receive(self, msg_l: int, msg_c: int) -> HLCTimestamp:
        """메시지 수신 — 수신된 HLC 타임스탬프로 자신을 갱신"""
        with self._lock:
            pt = self._physical_time_ns()
            l_prime = max(self._l, msg_l, pt)

            drift = l_prime - pt
            if drift > self.MAX_DRIFT_NS:
                raise ValueError(
                    f"[{self.node_id}] HLC drift {drift/1e9:.3f}s > 허용 범위 "
                    f"{self.MAX_DRIFT_NS/1e9:.3f}s — 시스템 시계 동기화 필요"
                )

            if l_prime == self._l == msg_l:
                self._c = max(self._c, msg_c) + 1
            elif l_prime == self._l:
                self._c += 1
            elif l_prime == msg_l:
                self._c = msg_c + 1
            else:
                self._c = 0

            self._l = l_prime
            return HLCTimestamp(self._l, self._c)

# 시뮬레이션: 세 노드 간의 이벤트 흐름
hlc_a = HybridLogicalClock("A")
hlc_b = HybridLogicalClock("B")
hlc_c = HybridLogicalClock("C")

ts_a1 = hlc_a.tick()
print(f"A: 이벤트 발생 @ {ts_a1}")

time.sleep(0.001)

ts_b_recv = hlc_b.receive(ts_a1.l, ts_a1.c)
print(f"B: A 메시지 수신 @ {ts_b_recv}")

ts_b1 = hlc_b.tick()
print(f"B: 추가 이벤트 @ {ts_b1}")

ts_c_recv = hlc_c.receive(ts_b1.l, ts_b1.c)
print(f"C: B 메시지 수신 @ {ts_c_recv}")

print(f"\n인과 순서 검증:")
print(f"  ts_a1 < ts_b_recv: {ts_a1 < ts_b_recv}")
print(f"  ts_b_recv < ts_b1: {ts_b_recv < ts_b1}")
print(f"  ts_b1 < ts_c_recv: {ts_b1 < ts_c_recv}")
```

### 예제 2: HLC 기반 분산 KV 스토어의 낙관적 읽기 (CockroachDB 스타일)

```python
from typing import Dict, Any
import copy

@dataclass
class VersionedValue:
    value: Any
    ts: HLCTimestamp
    node: str

class DistributedKVStore:
    """HLC를 이용한 다중 버전 KV 스토어 (MVCC 시뮬레이션)"""

    def __init__(self, node_id: str):
        self.node_id = node_id
        self.hlc = HybridLogicalClock(node_id)
        self._store: Dict[str, list] = {}

    def put(self, key: str, value: Any) -> HLCTimestamp:
        ts = self.hlc.tick()
        if key not in self._store:
            self._store[key] = []
        self._store[key].append(VersionedValue(value, ts, self.node_id))
        self._store[key].sort(key=lambda v: v.ts)
        print(f"[{self.node_id}] PUT {key}={value!r} @ {ts}")
        return ts

    def get(self, key: str, read_ts: Optional[HLCTimestamp] = None) -> Optional[VersionedValue]:
        if key not in self._store:
            return None
        versions = self._store[key]
        if read_ts is None:
            return versions[-1]
        eligible = [v for v in versions if v.ts <= read_ts]
        return eligible[-1] if eligible else None

    def replicate_from(self, remote_key: str, remote_value: Any,
                       remote_ts: HLCTimestamp, remote_node: str):
        self.hlc.receive(remote_ts.l, remote_ts.c)
        if remote_key not in self._store:
            self._store[remote_key] = []
        entry = VersionedValue(remote_value, remote_ts, remote_node)
        self._store[remote_key].append(entry)
        self._store[remote_key].sort(key=lambda v: v.ts)
        print(f"[{self.node_id}] REPLICATE {remote_key}={remote_value!r} @ {remote_ts} from {remote_node}")

# 시뮬레이션
node1 = DistributedKVStore("node1")
node2 = DistributedKVStore("node2")

ts1 = node1.put("user:1", {"name": "Alice", "balance": 1000})
time.sleep(0.001)
ts2 = node1.put("user:1", {"name": "Alice", "balance": 1500})

node2.replicate_from("user:1", {"name": "Alice", "balance": 1000}, ts1, "node1")
node2.replicate_from("user:1", {"name": "Alice", "balance": 1500}, ts2, "node1")

snapshot = node2.get("user:1", read_ts=ts1)
print(f"\n[node2] ts1 시점 스냅샷: {snapshot.value}")

latest = node2.get("user:1")
print(f"[node2] 최신 데이터: {latest.value}")
```

---

## 4. CockroachDB와 YugabyteDB의 HLC 활용

### CockroachDB

CockroachDB는 모든 트랜잭션에 HLC 타임스탬프를 부여합니다:
- **쓰기**: HLC를 tick하여 타임스탬프 획득
- **읽기**: 지정된 HLC 타임스탬프의 스냅샷 읽기 (MVCC)
- **클락 스큐 처리**: 타임스탬프 불확실성 구간(`[now, now + maxOffset]`)에 충돌하는 읽기는 "uncertain read restart"로 재시도

`kv.clock.max_offset`(기본 500ms)를 초과하는 클락 드리프트는 CockroachDB가 노드를 종료해 데이터 정합성을 보호합니다.

### YugabyteDB

YugabyteDB도 HLC를 기반으로 하는 HybridTime을 사용합니다. 특히 지리 분산 환경에서 리전 간 읽기의 인과 일관성을 보장하는 데 HLC가 핵심 역할을 합니다.

---

## 5. 주의사항과 설계 지침

### NTP 동기화 품질이 핵심

HLC의 `l` 값은 물리 시간과 최대 `ε`만큼 떨어질 수 있습니다. NTP 동기화가 나쁠수록 `l`이 물리 시간을 크게 앞서게 되고, 시스템이 오류를 발생시킵니다. 프로덕션 시스템에서는:
- **chrony** 또는 **ntpd**로 정기 동기화
- 클락 드리프트 허용 한계를 환경에 맞게 튜닝 (보통 250ms~500ms)
- 드리프트 모니터링 알람 설정

### HLC는 전역 순서가 아닌 인과 순서

HLC는 **인과적으로 연관된 이벤트** 간의 순서만 보장합니다. 완전히 독립적인 두 이벤트의 `l`이 같고 `c`도 같을 수 있으며, 이 경우 타이-브레이킹(예: 노드 ID 비교)이 필요합니다.

### 타임스탬프 압축: 64비트 NTP 포맷 호환

HLC는 NTP의 64비트 타임스탬프 형식과 호환되도록 설계할 수 있습니다:
- 상위 48비트: 물리 시간 (초 단위, NTP 에포크 기준)
- 하위 16비트: 논리 카운터

이를 통해 기존 NTP 인프라를 수정 없이 활용하면서 인과성 보장을 추가할 수 있습니다.

### 클락 워프(Clock Warp) 처리

물리 시계가 앞으로 점프(leap second, NTP 큰 조정)하거나 뒤로 가는 경우, HLC의 `l`은 절대 뒤로 가지 않으므로 단조증가(monotonically increasing) 속성이 유지됩니다. 이는 분산 스냅샷과 일관된 읽기에 필수적입니다.

---

## 참고 자료

- [Kulkarni et al. (2014). "Logical Physical Clocks" — HLC 원본 논문](https://www.cse.buffalo.edu/~demirbas/publications/hlc.pdf)
- [CockroachDB — Living Without Atomic Clocks](https://www.cockroachlabs.com/blog/living-without-atomic-clocks/)
- [ar5iv — HLC와 NTP 64비트 포맷 호환성 연구](https://ar5iv.arxiv.org/html/1612.05205)
- [YugabyteDB — HybridTime: An HLC-Based Read-at-a-Timestamp](https://docs.yugabyte.com/preview/architecture/transactions/single-row-transactions/)
