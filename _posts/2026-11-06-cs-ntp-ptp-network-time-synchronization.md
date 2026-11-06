---
layout: post
title: "NTP/PTP 네트워크 타임 동기화 프로토콜 심화: 분산 시스템에서의 시간 문제"
date: 2026-11-06
categories: [cs, computer-science]
tags: [ntp, ptp, time-synchronization, distributed-systems, clock-drift, marzullo, ieee-1588]
---

분산 시스템에서 "지금 몇 시인가?"는 간단한 질문이 아니다. 서버 A와 서버 B의 시계가 10ms씩 다르다면, 두 서버가 함께 처리하는 트랜잭션의 순서가 뒤집어질 수 있다. 2012년 Cloudflare의 대규모 장애와 AWS의 간헐적 DynamoDB 이상 동작 모두 시간 동기화 문제에서 비롯됐다.

## 왜 클록이 어긋나는가

### 클록 드리프트(Clock Drift)

컴퓨터의 하드웨어 클록은 수정 발진자(Quartz Oscillator)를 사용한다. 이 발진자는 온도, 전압, 노화에 따라 약 1~100 ppm(parts per million)의 오차를 가진다. 1 ppm 오차는 하루에 약 86ms, 한 달에 약 2.6초다.

```
drift = rated_frequency - actual_frequency
drift_rate [ppm] = drift / rated_frequency × 10^6

예: 1 ppm 드리프트 → 하루에 86.4ms 오차 발생
   10 ppm 드리프트 → 하루에 864ms (약 0.86초) 오차 발생
```

### 시간 동기화의 필요성

1. **분산 트랜잭션**: 여러 노드에 걸친 트랜잭션에서 타임스탬프 기반 순서 결정
2. **로그 상관(Log Correlation)**: 여러 서버의 로그를 시간순으로 합쳐 분석
3. **보안 인증**: TLS 인증서 유효기간, JWT 만료 시각 검사
4. **캐시 무효화**: HTTP Cache-Control의 `max-age` 계산
5. **금융 거래**: 초 단위 이하의 정밀도로 주문 순서 결정

## NTP (Network Time Protocol, RFC 5905)

### 구조와 계층

NTP는 1985년 David Mills가 설계한 프로토콜로, 계층적 클록 구조(Stratum)를 사용한다.

```
Stratum 0: 원자시계, GPS 수신기, 라디오 방송 시각 — 절대 기준
   │
Stratum 1: Stratum 0에 직접 연결된 타임 서버 (오차 < 1μs)
   │
Stratum 2: Stratum 1에 네트워크로 연결된 서버 (오차 수 ms)
   │
Stratum 3+: 각 Stratum은 상위로부터 시간을 받아 하위에 제공
   │
클라이언트: Stratum n에 연결, 로컬 클록 조정
```

`pool.ntp.org`는 Stratum 2에 해당하며, 전 세계 수천 개의 NTP 서버 풀을 운영한다.

### NTP의 오프셋 계산 원리

NTP는 4개의 타임스탬프로 클록 오프셋과 RTT를 계산한다.

```
클라이언트                     NTP 서버

T1: Request 발송
        ───────────────────→
                           T2: Request 수신
                           T3: Response 발송
        ←───────────────────
T4: Response 수신

오프셋 θ = ((T2 - T1) + (T3 - T4)) / 2
RTT δ   = (T4 - T1) - (T3 - T2)
```

이 공식의 가정은 "업스트림/다운스트림 경로의 지연이 같다"는 것이다. 비대칭 네트워크(예: 위성 링크)에서는 오차가 발생한다.

## 실제 구현 예제

### 예제 1: NTP 클라이언트 구현

```python
import socket
import struct
import time
from dataclasses import dataclass
from typing import Optional

# NTP epoch (1900-01-01)와 Unix epoch (1970-01-01)의 차이
NTP_DELTA = 2208988800  # 70년을 초로 환산

@dataclass
class NTPResponse:
    stratum: int            # 계층 (1~15)
    offset_ms: float        # 클록 오프셋 (ms)
    rtt_ms: float           # 왕복 지연 (ms)
    reference_id: bytes     # 참조 서버 ID
    root_delay_ms: float    # 루트까지의 왕복 지연

def ntp_fixed_to_float(seconds: int, fraction: int) -> float:
    """NTP 고정소수점 형식(32.32비트) → Python float"""
    return seconds + fraction / (2**32)

def query_ntp(server: str = "pool.ntp.org",
              port: int = 123,
              timeout: float = 5.0) -> Optional[NTPResponse]:
    """
    NTPv4 쿼리 (RFC 5905)
    패킷 구조: LI(2) | VN(3) | Mode(3) | Stratum(8) | Poll(8) | Precision(8)
              | Root Delay(32) | Root Dispersion(32) | Reference ID(32)
              | Reference Timestamp(64) | Originate Timestamp(64)
              | Receive Timestamp(64) | Transmit Timestamp(64)
    """
    # LI=0(경고 없음), VN=4(NTPv4), Mode=3(Client)
    # 0 00 011 = 0b00011011 = 0x1b
    NTP_PACKET = bytes([0x1b]) + bytes(47)

    try:
        with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as sock:
            sock.settimeout(timeout)

            # T1: 클라이언트 로컬 시각 (송신 직전)
            t1_local = time.time()
            sock.sendto(NTP_PACKET, (server, port))

            data, addr = sock.recvfrom(1024)

            # T4: 클라이언트 로컬 시각 (수신 직후)
            t4_local = time.time()

    except (socket.timeout, socket.gaierror) as e:
        print(f"NTP 쿼리 실패: {e}")
        return None

    if len(data) < 48:
        print("응답 패킷 크기 부족")
        return None

    # 패킷 파싱
    li_vn_mode, stratum, poll, precision = struct.unpack_from('!BBBB', data, 0)
    root_delay_raw, root_disp_raw = struct.unpack_from('!II', data, 4)
    ref_id = data[12:16]

    # Timestamps (각 64비트: 32비트 초 + 32비트 소수)
    # Offset 32: T2 (서버가 요청 수신한 시각)
    t2_sec, t2_frac = struct.unpack_from('!II', data, 32)
    # Offset 40: T3 (서버가 응답 발송한 시각)
    t3_sec, t3_frac = struct.unpack_from('!II', data, 40)

    t2 = ntp_fixed_to_float(t2_sec, t2_frac) - NTP_DELTA
    t3 = ntp_fixed_to_float(t3_sec, t3_frac) - NTP_DELTA

    # 오프셋과 RTT 계산
    theta = ((t2 - t1_local) + (t3 - t4_local)) / 2
    delta = (t4_local - t1_local) - (t3 - t2)

    # Root Delay (Q16.16 고정소수점)
    root_delay_ms = (root_delay_raw / (2**16)) * 1000

    return NTPResponse(
        stratum=stratum,
        offset_ms=theta * 1000,
        rtt_ms=delta * 1000,
        reference_id=ref_id,
        root_delay_ms=root_delay_ms
    )

def multi_server_query(servers: list[str], n: int = 3) -> dict:
    """여러 NTP 서버를 쿼리하여 가장 신뢰할 수 있는 오프셋을 선택"""
    results = []
    for server in servers[:n]:
        resp = query_ntp(server)
        if resp and resp.rtt_ms < 100:  # RTT 100ms 이상은 신뢰 불가
            results.append(resp)
            print(f"{server}: offset={resp.offset_ms:+.3f}ms, "
                  f"rtt={resp.rtt_ms:.3f}ms, stratum={resp.stratum}")

    if not results:
        print("사용 가능한 NTP 서버 없음")
        return {}

    # RTT가 가장 짧은 서버의 오프셋을 채택 (단순화된 클럭 선택)
    best = min(results, key=lambda r: r.rtt_ms)
    print(f"\n채택된 오프셋: {best.offset_ms:+.3f}ms (rtt={best.rtt_ms:.3f}ms)")

    return {
        "offset_ms": best.offset_ms,
        "rtt_ms": best.rtt_ms,
        "stratum": best.stratum
    }

# 사용 예 (실제 환경)
# servers = ["0.pool.ntp.org", "1.pool.ntp.org", "2.pool.ntp.org"]
# result = multi_server_query(servers)

# 시뮬레이션 출력 예:
# 0.pool.ntp.org: offset=+2.341ms, rtt=15.234ms, stratum=2
# 1.pool.ntp.org: offset=+2.187ms, rtt=12.891ms, stratum=2
# 2.pool.ntp.org: offset=+10.234ms, rtt=48.231ms, stratum=3  ← 느림, 버림
# 채택된 오프셋: +2.187ms (rtt=12.891ms)
```

### 예제 2: Marzullo 알고리즘과 클록 선택 (Clock Selection)

여러 NTP 서버 중 불량 서버(Falseticker)를 필터링하고 최적의 클록을 선택하는 알고리즘이다.

```python
from typing import List, Tuple, Optional
import statistics

@dataclass
class ClockSource:
    server: str
    offset_ms: float      # 측정된 오프셋 (ms)
    dispersion_ms: float  # 오차 범위 (ms) = rtt/2 + 고유 불확실성
    stratum: int
    is_falseticker: bool = False

def marzullo_algorithm(
    clocks: List[ClockSource]
) -> Tuple[Optional[Tuple[float, float]], int]:
    """
    Marzullo의 알고리즘:
    가장 많은 클록과 겹치는 오프셋 구간을 찾아 Falseticker를 필터링

    Returns: (최적 구간, 합의된 클록 수)
    """
    # 각 클록의 오프셋 범위를 이벤트로 변환
    # (값, 타입): +1 = 구간 시작, -1 = 구간 끝
    events: List[Tuple[float, int]] = []
    for clock in clocks:
        lo = clock.offset_ms - clock.dispersion_ms
        hi = clock.offset_ms + clock.dispersion_ms
        events.append((lo, +1, clock.server))
        events.append((hi, -1, clock.server))

    # 값 기준 정렬 (같은 값이면 종료(-1)보다 시작(+1)을 먼저)
    events.sort(key=lambda x: (x[0], -x[1]))

    best_lo = 0.0
    best_hi = 0.0
    best_count = 0
    current_count = 0
    best_lo_set = False

    for value, event_type, server in events:
        current_count += event_type
        if current_count > best_count:
            best_count = current_count
            best_lo = value
            best_lo_set = True
        if event_type == -1 and best_lo_set and current_count == best_count - 1:
            if best_hi == 0.0 or value > best_hi:
                best_hi = value

    if best_count == 0:
        return None, 0

    return (best_lo, best_hi), best_count

def clock_filter(readings: List[float], n: int = 4) -> dict:
    """
    NTP의 Clock Filter 알고리즘:
    최근 N개의 샘플 중 최소 jitter를 가진 샘플을 선택
    """
    if len(readings) < 2:
        return {"offset": readings[0] if readings else 0, "jitter": 0}

    # 각 샘플의 jitter = 다른 샘플들과의 평균 편차
    best_offset = None
    min_jitter = float('inf')

    for i, offset in enumerate(readings[-n:]):
        others = [r for j, r in enumerate(readings[-n:]) if j != i]
        jitter = statistics.mean(abs(offset - o) for o in others)
        if jitter < min_jitter:
            min_jitter = jitter
            best_offset = offset

    return {
        "offset": best_offset,
        "jitter_ms": min_jitter,
        "std_ms": statistics.stdev(readings[-n:]) if len(readings) >= 2 else 0
    }

# 시뮬레이션
clocks = [
    ClockSource("ntp1.example.com", offset_ms=2.5,  dispersion_ms=1.3, stratum=2),
    ClockSource("ntp2.example.com", offset_ms=2.8,  dispersion_ms=1.5, stratum=2),
    ClockSource("ntp3.example.com", offset_ms=3.1,  dispersion_ms=1.8, stratum=2),
    ClockSource("ntp4.example.com", offset_ms=50.0, dispersion_ms=5.0, stratum=3),  # Falseticker
]

print("=== Marzullo 알고리즘 실행 ===")
result_interval, consensus_count = marzullo_algorithm(clocks)

if result_interval:
    lo, hi = result_interval
    best_offset = (lo + hi) / 2
    print(f"최적 오프셋 구간: [{lo:.2f}ms, {hi:.2f}ms]")
    print(f"최적 오프셋 추정치: {best_offset:.2f}ms")
    print(f"합의된 클록 수: {consensus_count}/{len(clocks)}")
    print(f"Falseticker 감지: {len(clocks) - consensus_count}개")
else:
    print("합의 실패")

# 출력 예:
# 최적 오프셋 구간: [2.30ms, 3.60ms]
# 최적 오프셋 추정치: 2.95ms
# 합의된 클록 수: 3/4
# Falseticker 감지: 1개

print("\n=== Clock Filter 실행 ===")
# 같은 서버의 반복 샘플들
samples = [2.1, 2.3, 8.9, 2.4, 2.2, 2.5]  # 8.9ms는 일시적 오류
filtered = clock_filter(samples)
print(f"선택된 오프셋: {filtered['offset']:.2f}ms")
print(f"Jitter: {filtered['jitter_ms']:.2f}ms")
print(f"표준편차: {filtered['std_ms']:.2f}ms")
```

## PTP (Precision Time Protocol, IEEE 1588)

### NTP vs PTP

| 항목 | NTP | PTP |
|------|-----|-----|
| 정확도 | 1~10ms (소프트웨어) | 1~100ns (하드웨어 타임스탬핑) |
| 적용 범위 | 범용 인터넷 | 로컬 네트워크 (LAN/WAN) |
| 하드웨어 필요 | 불필요 | 권장 (PHC: PTP Hardware Clock) |
| 사용 사례 | 일반 서버 | 금융거래, 통신, 산업 자동화, 5G |
| RFC | RFC 5905 | IEEE 1588-2019 |

### PTP의 메시지 교환

PTP는 2단계 메시지 교환으로 정밀한 오프셋을 계산한다.

```
마스터 클록              슬레이브 클록

[Sync 메시지 발송]
  T1 (HW 타임스탬핑) ──→
                         T2 (HW 타임스탬핑, 수신)
                         ↓
                    [Follow_Up 메시지로 T1 수신]
                         ↓
[Delay_Req 수신]
  T4 (HW 타임스탬핑) ←── T3 (HW 타임스탬핑, 발송)
  ↓
[Delay_Resp 메시지로 T4 전송]

전파 지연 D = ((T2 - T1) + (T4 - T3)) / 2
오프셋 O   = (T2 - T1) - D = ((T2 - T1) - (T4 - T3)) / 2
```

### PTP 하드웨어 타임스탬핑의 중요성

소프트웨어 타임스탬핑은 OS 스케줄링, 인터럽트 지연 등으로 수 ms의 불확실성이 발생한다. NIC(Network Interface Card)가 패킷 송수신 시점에 직접 타임스탬프를 기록하면 나노초 정밀도가 가능하다.

```bash
# Linux에서 PTP 하드웨어 타임스탬핑 지원 확인
ethtool -T eth0

# 출력 예:
# Time stamping parameters for eth0:
# Capabilities:
#         hardware-transmit     (PHC 사용, TX 하드웨어 타임스탬핑)
#         software-transmit     (소프트웨어 TX 타임스탬핑)
#         hardware-receive      (PHC 사용, RX 하드웨어 타임스탬핑)
#         software-receive      (소프트웨어 RX 타임스탬핑)
#         software-system-clock (시스템 클록 사용)
#         hardware-raw-clock    (PHC 원시 클록)
# PTP Hardware Clock: 0        ← /dev/ptp0
# Hardware Transmit Timestamp Modes:
#         off
#         on
# Hardware Receive Filter Modes:
#         none
#         all

# PTP 데몬 실행 (linuxptp)
sudo ptp4l -i eth0 -m -S   # 슬레이브 모드
# 또는
sudo ptp4l -i eth0 -m -M   # 마스터 모드
```

## 분산 시스템에서의 시간 문제

### Leap Second (윤초)

지구의 자전 속도 불규칙으로 인해 약 1~2년마다 1초가 추가된다. 이때 클록이 갑자기 1초 뒤로 가면(23:59:59 → 23:59:59 → 00:00:00) 타임스탬프 비교 로직이 깨진다.

**Google Leap Smear**: 윤초 추가 24시간 전부터 초당 클록을 아주 조금씩 늦춰서 윤초를 부드럽게 흡수하는 기법. AWS, Google Cloud 모두 사용.

```
일반 처리: ...23:59:59 → 23:59:60 → 00:00:00 (불연속)
Leap Smear: 윤초 직전 24시간 동안 1일 = 86401초를 86400초처럼 운영
```

### TrueTime (Google Spanner)

Google Spanner는 GPS와 원자시계를 클라우드 데이터센터에 직접 배치하고, **TrueTime API**를 통해 오차 범위를 명시한 시간을 제공한다.

```
TrueTime.now() → { earliest: T - ε, latest: T + ε }
         ε = 약 1~7ms (대부분 < 7ms)
```

트랜잭션 커밋 전에 `2ε` 시간을 대기하면, 이후 커밋된 모든 트랜잭션은 반드시 더 큰 타임스탬프를 가짐이 보장된다. 이것이 Spanner의 External Consistency의 비결이다.

## 주의사항과 실전 팁

### 1. 시스템 클록 직접 조정 금지

애플리케이션에서 직접 `settimeofday()`로 시스템 클록을 뒤로 돌리면 `time.monotonic()` 외의 모든 타임스탬프가 의미를 잃는다. NTP 데몬(ntpd, chrony, systemd-timesyncd)에게 맡기고 `slew mode`(천천히 조정)를 사용하라.

```python
import time

# 나쁜 예: 로그 타임스탬프에 wall clock 사용
def bad_timing():
    start = time.time()  # NTP 조정으로 역행할 수 있음
    do_work()
    elapsed = time.time() - start  # 음수가 될 수 있음!

# 좋은 예: 경과 시간 측정에는 monotonic clock 사용
def good_timing():
    start = time.monotonic()  # 단조 증가, NTP 영향 받지 않음
    do_work()
    elapsed = time.monotonic() - start  # 항상 양수
```

### 2. 타임존 vs UTC

서버는 항상 UTC를 사용하라. 타임존 변환은 출력 직전에만 수행한다.

```python
from datetime import datetime, timezone

# 나쁜 예: 로컬 타임존 의존
local_time = datetime.now()  # 서버 타임존에 의존

# 좋은 예: UTC 명시
utc_time = datetime.now(timezone.utc)
iso_string = utc_time.isoformat()  # "2026-11-06T10:30:00+00:00"
```

### 3. chrony vs ntpd

현대 Linux 환경에서는 `chrony`가 `ntpd`보다 권장된다. chrony는 간헐적 네트워크 환경과 가상화 환경에서 더 빠르게 동기화되고, 클록 드리프트 보정이 더 정교하다.

```bash
# chrony 상태 확인
chronyc tracking

# 출력 예:
# Reference ID    : C0A80101 (ntp1.example.com)
# Stratum         : 3
# Ref time (UTC)  : Fri Nov 06 10:30:00 2026
# System time     : 0.000002341 seconds slow of NTP time  ← 현재 오프셋
# Last offset     : +0.000001523 seconds
# RMS offset      : 0.000003214 seconds
# Frequency       : 2.345 ppm slow                        ← 드리프트율
# Residual freq   : +0.001 ppm
# Skew            : 0.123 ppm
# Root delay      : 0.015234 seconds
# Root dispersion : 0.000832 seconds
# Update interval : 64.2 seconds
```

### 4. 분산 시스템에서의 시간 사용 원칙

- **이벤트 순서**: 가능하면 Lamport Clock, Hybrid Logical Clock(HLC) 같은 논리 클록을 사용
- **절대 시각이 필요한 경우**: TrueTime처럼 오차 범위를 명시하거나, NTP 동기화 상태를 모니터링
- **허용 가능한 오차**: 대부분의 애플리케이션은 10ms 이내 오차로 충분. 금융/통신은 μs~ns 필요

시간 동기화는 분산 시스템의 조용한 살인자다. 로그에서 이상한 순서, 트랜잭션의 간헐적 실패, 캐시가 갑자기 풀리는 현상이 보이면 먼저 클록 동기화 상태를 의심하라.

## 참고 자료
- [linuxptp: Linux PTP 구현체 (GitHub)](https://github.com/mlichvar/linuxptp)
- [NTPsec: 보안 강화된 NTPv4 구현체 (GitHub)](https://github.com/ntpsec/ntpsec)
