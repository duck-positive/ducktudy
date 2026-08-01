---
layout: post
title: "BBR 혼잡 제어 알고리즘 완전 정복: 측정 기반 접근으로 인터넷을 빠르게 만든 방법"
date: 2026-08-01
categories: [cs, computer-science]
tags: [bbr, congestion-control, tcp, networking, bandwidth, latency, google, cubic, reno]
---

TCP 혼잡 제어는 인터넷의 안정성을 책임지는 핵심 메커니즘이다. 1980년대 개발된 Reno와 그 진화형인 CUBIC은 **패킷 손실**을 혼잡의 신호로 삼아 전송 속도를 조절한다. 이 방식은 40년 가까이 인터넷을 지탱해왔지만, 현대 네트워크 환경에서는 심각한 한계를 드러냈다. 2016년 Google이 발표한 **BBR(Bottleneck Bandwidth and Round-trip propagation time)**은 이 문제를 근본적으로 다른 방식으로 해결한다. 이 글에서는 BBR의 핵심 아이디어와 내부 동작 원리, 그리고 실제 효과를 심층 분석한다.

## 기존 손실 기반 혼잡 제어의 한계

### 버퍼블로트(Bufferbloat) 문제

현대 네트워크 장비(라우터, 스위치, 모뎀)는 패킷 손실을 줄이기 위해 큰 버퍼를 장착하고 있다. 이는 CUBIC 같은 손실 기반 알고리즘에 치명적인 부작용을 일으킨다.

```
CUBIC의 동작 (손실 기반):
1. 전송 속도 증가 → 라우터 버퍼 채워짐
2. 버퍼가 가득 차도 라우터는 버퍼를 비우기 위해 패킷을 버리지 않음
3. CUBIC은 손실을 감지하지 못해 계속 속도 증가
4. 버퍼가 완전히 꽉 찰 때까지 채움 → 결국 패킷 드롭
5. 손실 감지 → 속도 급감 → cwnd *= 0.7 (70% 감소)

결과:
- 버퍼에 수십~수백 ms의 패킷이 쌓임 → 대기 레이턴시 폭발
- 예: 100Mbps 링크, 버퍼 크기 1000패킷(1.5KB each)
  → 최악의 경우 추가 지연: 1000 × 1500바이트 / 100Mbps ≈ 120ms
```

이것이 버퍼블로트 문제다. 전송 속도는 빠른데 레이턴시가 수백 ms로 치솟는 현상. 게임, 화상 회의, 실시간 스트리밍이 끊기는 이유다.

### 랜덤 패킷 손실과의 혼동

무선 네트워크(WiFi, LTE, 5G)에서는 혼잡과 무관한 **랜덤 패킷 손실**이 자주 발생한다. CUBIC은 이를 혼잡으로 오인하여 불필요하게 전송 속도를 줄인다. 실제 예시: WiFi 환경에서 1%의 랜덤 패킷 손실률이 있을 경우, CUBIC의 실효 처리량은 이론 대역폭의 20%에 불과할 수 있다.

## BBR의 핵심 아이디어: 네트워크 모델 기반 제어

### 두 가지 핵심 측정값

BBR은 네트워크의 실제 상태를 나타내는 두 가지 물리량을 지속적으로 측정하여 전송 속도를 제어한다.

**1. BtlBw (Bottleneck Bandwidth, 병목 대역폭)**  
연결 경로에서 가장 느린 링크(병목 링크)의 실제 가용 대역폭.

**2. RTprop (Round-Trip propagation time, 왕복 전파 시간)**  
큐잉 지연이 없을 때의 순수한 왕복 시간. 즉, 빛의 속도와 미디어 특성에 의한 물리적 최솟값.

```
BBR의 핵심 목표:
- 전송 속도 = BtlBw (병목 대역폭만큼 정확히 전송)
- 파이프 내 데이터 = BtlBw × RTprop (BDP, Bandwidth-Delay Product)
- 초과 데이터 없음 → 큐 없음 → 버퍼블로트 없음
```

### BDP(Bandwidth-Delay Product)와 최적 동작점

네트워크 파이프에 "가득 차지만 넘치지 않는" 최적의 데이터 양은 BDP다.

```
BDP = BtlBw × RTprop

예: 
  BtlBw = 100 Mbps
  RTprop = 20 ms
  BDP = 100,000,000 bit/s × 0.020 s = 2,000,000 bit = 250 KB

→ 파이프에 항상 250KB의 데이터를 유지하면 최대 처리량과 최소 지연 달성
```

CUBIC이 이 값보다 훨씬 많은 데이터를 버퍼에 쌓는 것과 달리, BBR은 BDP에 근접한 양만 파이프에 유지하려 한다.

## BBR의 상태 기계

BBR은 4개의 상태를 순환하며 동작한다.

### 1. STARTUP (시작) 단계

초기에 대역폭을 빠르게 탐색한다. Slow Start와 유사하게 pacing_gain=2/ln2 ≈ 2.885를 사용해 지수적으로 전송 속도를 높인다. BtlBw 추정값이 3회 연속 증가하지 않으면 DRAIN으로 전환한다.

### 2. DRAIN (배출) 단계

STARTUP에서 과잉 생성된 큐를 빠르게 비운다. pacing_gain=ln2/2 ≈ 0.346으로 전송 속도를 낮춰 버퍼를 비운다. inflight 데이터 ≤ BDP에 도달하면 PROBE_BW로 전환한다.

### 3. PROBE_BW (대역폭 탐색) 단계 — 핵심

정상 동작의 대부분을 차지하는 단계. 8개의 RTT 사이클을 반복하며 대역폭을 추적한다.

```
8-RTT 사이클:
[1.25, 0.75, 1.0, 1.0, 1.0, 1.0, 1.0, 1.0]
  ^      ^
  증가   감소   안정
  
- 1개의 probe 슬롯 (gain=1.25): 대역폭 1.25배 시도 → 새 최대치 발견 시 업데이트
- 1개의 drain 슬롯 (gain=0.75): 과잉 큐 배출
- 6개의 cruise 슬롯 (gain=1.0): 추정 BtlBw로 안정적 전송
```

### 4. PROBE_RTT (RTT 탐색) 단계

RTprop 추정값이 10초 이상 갱신되지 않으면 진입한다. inflight를 4패킷으로 극단적으로 줄여 큐를 모두 비우고 순수 전파 지연을 측정한다. 200ms 경과 후 PROBE_BW로 복귀한다.

## BBR의 핵심 메커니즘: Pacing과 cwnd

### Pacing(패이싱): 버스트 없는 균일한 전송

CUBIC은 ACK를 받는 즉시 허용된 최대량을 버스트로 전송한다. BBR은 **pacing**으로 패킷을 시간에 고르게 분산시킨다.

```
CUBIC 전송 패턴 (버스트):
시간:  0     1ms   2ms   3ms   4ms
전송:  ████████                ████████ (ACK 오면 한꺼번에)

BBR 전송 패턴 (pacing):
시간:  0     1ms   2ms   3ms   4ms
전송:  ██    ██    ██    ██    ██  (BtlBw에 맞게 균일하게)
```

pacing_rate = BtlBw × pacing_gain

### cwnd(혼잡 윈도우): BDP 기반 상한선

BBR의 cwnd는 패킷 손실이 아닌 BDP로 결정된다.

```
cwnd = BtlBw × RTprop × cwnd_gain + 3 × BDP/BW
     ≈ 2 × BDP (PROBE_BW 단계의 일반적인 값)
```

## Linux 커널에서의 BBR 측정 코드 분석

BBR은 Linux 커널 4.9부터 포함되었다. 핵심 측정 로직을 분석한다.

```c
/* 패킷 ACK 수신 시 호출되는 BBR 측정 함수 */
static void bbr_update_bw(struct sock *sk, const struct rate_sample *rs)
{
    struct tcp_sock *tp = tcp_sk(sk);
    struct bbr *bbr = inet_csk_ca(sk);
    u64 bw;
    
    /* delivery_rate = 이 ACK까지 전달된 데이터량 / 소요 시간 */
    if (rs->delivered < 0 || rs->interval_us <= 0)
        return; /* 유효하지 않은 샘플 무시 */
    
    bw = (u64)rs->delivered * BW_UNIT;
    do_div(bw, rs->interval_us);
    
    /* 10회 RTT 슬라이딩 윈도우에서 최대 BW를 추적 */
    /* 오래된 측정값은 windowed_max로 자동 제거 */
    if (bbr_full_bw_reached(sk) || bw >= bbr_max_bw(sk)) {
        minmax_running_max(&bbr->bw, bbr_bw_rtts, bbr->rtt_cnt, bw);
    }
}

static void bbr_update_min_rtt(struct sock *sk, const struct rate_sample *rs)
{
    struct tcp_sock *tp = tcp_sk(sk);
    struct bbr *bbr = inet_csk_ca(sk);
    bool filter_expired;
    
    /* RTprop을 10초 슬라이딩 윈도우에서 최솟값으로 추적 */
    filter_expired = after(tcp_jiffies32,
                           bbr->min_rtt_stamp + bbr_min_rtt_win_sec * HZ);
    if (rs->rtt_us >= 0 &&
        (rs->rtt_us <= bbr->min_rtt_us || filter_expired)) {
        bbr->min_rtt_us = rs->rtt_us;   /* RTprop 업데이트 */
        bbr->min_rtt_stamp = tcp_jiffies32;
    }
    /* ... PROBE_RTT 진입 로직 ... */
}
```

이 코드에서 핵심은 `minmax_running_max`와 `minmax_running_min`이다. 각각 슬라이딩 윈도우에서 BtlBw의 최댓값과 RTprop의 최솟값을 O(1)에 추적한다.

## Python으로 BBR 동작 시뮬레이션

```python
from collections import deque
import random

class BBRSimulator:
    """BBR 혼잡 제어 시뮬레이터 (교육용 단순화 버전)"""
    
    def __init__(self, btl_bw_mbps: float, base_rtt_ms: float, buffer_size_kb: float):
        self.btl_bw = btl_bw_mbps * 1e6 / 8  # bytes/sec
        self.base_rtt = base_rtt_ms / 1000    # seconds
        self.buffer_size = buffer_size_kb * 1024  # bytes
        
        # BBR 상태
        self.state = "STARTUP"
        self.btl_bw_est = 1e3      # 초기 대역폭 추정값 (1KB/s)
        self.rtt_prop = float('inf')
        self.bw_window = deque(maxlen=10)  # 슬라이딩 윈도우
        
        # 통계
        self.queue_bytes = 0
        self.time = 0
        self.total_delivered = 0
    
    def _delivery_rate(self, sent_bytes: float, rtt: float) -> float:
        """실제 전달률 추정"""
        return sent_bytes / rtt if rtt > 0 else 0
    
    def _queue_delay(self) -> float:
        """현재 버퍼 큐잉 지연"""
        return self.queue_bytes / self.btl_bw
    
    def step(self, dt: float = 0.001):
        """dt초만큼 시뮬레이션 진행"""
        # 실제 RTT = 기본 RTT + 큐잉 지연
        actual_rtt = self.base_rtt + self._queue_delay()
        
        # 측정: 전달률 계산
        inflight = self.btl_bw_est * actual_rtt
        pacing_rate = self.btl_bw_est * self._pacing_gain()
        
        sent = min(pacing_rate * dt, self.btl_bw * dt)
        self.queue_bytes = max(0, self.queue_bytes + sent - self.btl_bw * dt)
        self.total_delivered += self.btl_bw * dt
        
        # BtlBw 추정 업데이트
        rate = self._delivery_rate(sent, actual_rtt)
        self.bw_window.append(rate)
        self.btl_bw_est = max(self.bw_window)
        
        # RTprop 업데이트 (최솟값 추적)
        if actual_rtt < self.rtt_prop:
            self.rtt_prop = actual_rtt
        
        self.time += dt
        return {
            "time": self.time,
            "throughput_mbps": self.btl_bw * 8 / 1e6,
            "rtt_ms": actual_rtt * 1000,
            "queue_ms": self._queue_delay() * 1000,
            "pacing_rate_mbps": pacing_rate * 8 / 1e6,
            "btl_bw_est_mbps": self.btl_bw_est * 8 / 1e6,
        }
    
    def _pacing_gain(self) -> float:
        if self.state == "STARTUP":
            return 2.885
        elif self.state == "DRAIN":
            return 0.346
        elif self.state == "PROBE_BW":
            cycle_idx = int(self.time / self.base_rtt) % 8
            gains = [1.25, 0.75, 1.0, 1.0, 1.0, 1.0, 1.0, 1.0]
            return gains[cycle_idx]
        return 1.0

# CUBIC vs BBR 비교 시뮬레이션
print("=== BBR 시뮬레이션: 100Mbps 링크, 버퍼 4MB ===")
bbr = BBRSimulator(btl_bw_mbps=100, base_rtt_ms=20, buffer_size_kb=4096)

for step in range(200):
    result = bbr.step(dt=0.005)  # 5ms 스텝
    if step % 20 == 0:
        print(f"T={result['time']*1000:.0f}ms | "
              f"RTT={result['rtt_ms']:.1f}ms | "
              f"큐={result['queue_ms']:.1f}ms | "
              f"추정BW={result['btl_bw_est_mbps']:.1f}Mbps")
```

## BBR의 실제 성능 개선 효과

Google이 유튜브에 BBR을 적용한 결과(2016년 발표):

| 지표 | CUBIC | BBR | 개선율 |
|------|-------|-----|--------|
| 처리량 (RTT 100ms, 1% 손실) | ~3 Mbps | ~36 Mbps | +1100% |
| 평균 RTT | 33ms | 23ms | -30% |
| 95th 퍼센타일 RTT | 600ms | 35ms | -94% |
| 재전송률 | 3.5% | 0.5% | -86% |

특히 **고지연 고손실 환경**(예: 위성 인터넷, 장거리 해저 케이블)에서 CUBIC 대비 압도적으로 우수한 성능을 보인다.

## BBR의 한계와 BBRv2, BBRv3

BBRv1에는 여러 문제가 발견되었다:

1. **CUBIC과의 공정성 문제**: BBR 흐름들이 같은 병목을 공유하는 CUBIC 흐름에 비해 대역폭을 더 많이 점유하는 경향.
2. **얕은 버퍼에서의 패킷 손실**: probe 단계의 1.25 gain이 얕은 버퍼 라우터에서 불필요한 손실을 일으킴.
3. **다수 BBR 흐름 간 불공정**: 같은 병목을 공유하는 여러 BBR 흐름들이 대역폭을 불균등하게 나누는 경우.

2023년 발표된 BBRv3는 이런 문제들을 해결했다:
- 패킷 손실에 대한 반응 추가 (완전히 무시하지 않음)
- Inflight 한도 계산 개선
- PROBE_RTT 지속 시간 조정

```bash
# Linux에서 BBR 설정 방법
# 현재 혼잡 제어 알고리즘 확인
sysctl net.ipv4.tcp_congestion_control
# net.ipv4.tcp_congestion_control = cubic

# BBR로 변경
sudo sysctl -w net.ipv4.tcp_congestion_control=bbr
sudo sysctl -w net.core.default_qdisc=fq  # BBR은 fq 큐 규율과 함께 사용 권장

# 영구 설정 (/etc/sysctl.conf에 추가)
# net.core.default_qdisc=fq
# net.ipv4.tcp_congestion_control=bbr

# 현재 활성화된 알고리즘 확인
cat /proc/sys/net/ipv4/tcp_congestion_control

# 사용 가능한 알고리즘 목록
cat /proc/sys/net/ipv4/tcp_available_congestion_control
# reno cubic bbr
```

## 주의사항과 팁

**팁 1**: BBR은 **fq(Fair Queueing)** 큐 규율과 함께 사용해야 최적 성능을 발휘한다. fq는 Pacing을 커널 수준에서 구현하여 애플리케이션 레벨의 버스트를 매끄럽게 만든다.

**팁 2**: 짧은 지연 LAN 환경(RTT < 1ms)에서는 CUBIC이 여전히 경쟁력 있다. BBR의 장점은 고지연 환경에서 두드러진다.

**팁 3**: BBRv3를 사용하려면 최신 커널(6.x+)이 필요하다. 커널 버전 확인: `uname -r`.

**팁 4**: BBR을 서버에 적용할 때는 반드시 트래픽 패턴을 모니터링하라. 공유 인프라에서는 BBR 흐름이 CUBIC 흐름보다 더 공격적으로 대역폭을 사용할 수 있다.

## 참고 자료

- [BBR IETF 드래프트 — 공식 표준 명세](https://datatracker.ietf.org/doc/draft-ietf-ccwg-bbr/)
- [Neal Cardwell et al., BBR: Congestion-Based Congestion Control, ACM Queue 2016](https://dl.acm.org/doi/10.1145/3012426.3022184)
- [Linux Kernel BBR 소스 코드](https://github.com/torvalds/linux/blob/master/net/ipv4/tcp_bbr.c)
- [Google IETF 101 BBR 발표 슬라이드](https://www.ietf.org/proceedings/101/slides/slides-101-iccrg-an-update-on-bbr-work-at-google-00.pdf)
