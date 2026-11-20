---
layout: post
title: "TCP 혼잡 제어 알고리즘 완전 정복: Tahoe·Reno·CUBIC·BBR의 진화와 동작 원리"
date: 2026-11-20
categories: [cs, computer-science]
tags: [tcp, congestion-control, networking, bbr, cubic, reno, protocol, linux]
---

인터넷이 붕괴되지 않는 이유가 무엇일까? 1986년 Van Jacobson이 TCP 혼잡 붕괴(congestion collapse) 현상을 목격하고 고안한 혼잡 제어 알고리즘 덕분이다. 그 이후 수십 년간 Tahoe, Reno, CUBIC, BBR로 이어지는 진화를 통해 오늘날 인터넷은 수백억 개의 연결을 안정적으로 처리한다. 이 아티클에서는 각 알고리즘의 동작 원리와 성능 차이를 깊이 있게 살펴본다.

## 혼잡 제어란 무엇인가

TCP 혼잡 제어는 **송신자가 네트워크의 혼잡 정도를 감지하고 전송 속도를 조절하는 메커니즘**이다. 핵심 목표는 세 가지다:

1. **공정성(Fairness)**: 여러 연결이 대역폭을 공평하게 나눠 사용
2. **효율성(Efficiency)**: 네트워크 용량을 최대한 활용
3. **안정성(Stability)**: 혼잡 붕괴 없이 안정적 동작

혼잡 제어의 핵심 변수는 **혼잡 윈도우(CWND, Congestion Window)**다. 송신자는 `min(CWND, Receiver Window)` 크기만큼만 ACK 없이 전송할 수 있다.

### 혼잡 신호의 종류

- **패킷 손실**: 타임아웃 또는 중복 ACK 3개(Triple Duplicate ACK) 발생 → 혼잡 발생 신호
- **지연 증가**: 큐에 패킷이 쌓이면서 RTT가 증가 → 혼잡 조짐 신호
- **ECN(Explicit Congestion Notification)**: 라우터가 직접 혼잡 플래그를 설정 → 명시적 신호

## TCP Tahoe: 혼잡 제어의 시작 (1988)

Van Jacobson이 BSD에 처음 구현한 Tahoe는 세 가지 메커니즘을 도입했다.

### 1. 슬로 스타트 (Slow Start)

연결 초기에는 네트워크 용량을 모른다. CWND를 1 MSS(Maximum Segment Size, 보통 1460 bytes)에서 시작해 ACK가 올 때마다 1 MSS씩 증가 → 사실상 **지수적 증가 (RTT마다 2배)**.

### 2. 혼잡 회피 (Congestion Avoidance)

CWND가 임계값(ssthresh)에 도달하면 RTT마다 1 MSS씩 증가 → **선형 증가**로 전환.

### 3. 혼잡 감지 시 반응

- **타임아웃 발생**: `ssthresh = CWND/2`, `CWND = 1 MSS`, 슬로 스타트 재시작
- **Triple Duplicate ACK**: 동일하게 `ssthresh = CWND/2`, `CWND = 1 MSS`, 슬로 스타트

Tahoe의 한계: Triple Duplicate ACK에서도 CWND를 1로 떨어뜨려 성능이 급격히 저하.

## TCP Reno: 빠른 회복 (1990)

Reno는 Triple Duplicate ACK와 타임아웃을 구별한다.

### 빠른 재전송 + 빠른 회복 (Fast Retransmit + Fast Recovery)

```
Triple Duplicate ACK 발생 시:
  ssthresh = CWND / 2
  CWND = ssthresh + 3  (손실되지 않은 3개 패킷 반영)
  빠른 회복 모드 진입: 추가 중복 ACK마다 CWND += 1
  
새 ACK 수신 시:
  CWND = ssthresh  (빠른 회복 종료)
  혼잡 회피 모드로 전환
```

CWND를 1로 떨어뜨리는 대신 절반으로만 줄임으로써 성능이 크게 개선됐다.

### AIMD: Additive Increase, Multiplicative Decrease

Reno가 확립한 AIMD 원칙:
- **증가**: RTT마다 `CWND += 1 MSS` (Additive Increase)
- **감소**: 혼잡 감지 시 `CWND = CWND / 2` (Multiplicative Decrease)

이 비대칭적 조정이 여러 연결의 공정한 대역폭 수렴을 보장한다는 것이 수학적으로 증명됐다.

## Python으로 시뮬레이션: Tahoe vs Reno

```python
from dataclasses import dataclass
from typing import List, Tuple

@dataclass
class TCPState:
    cwnd: float = 1.0      # 혼잡 윈도우 (MSS 단위)
    ssthresh: float = 64.0  # 슬로 스타트 임계값
    mode: str = "slow_start"  # "slow_start" | "congestion_avoidance" | "fast_recovery"


def simulate_tahoe(events: List[str]) -> List[float]:
    """TCP Tahoe 시뮬레이션. events: 'ack' | 'timeout' | 'triple_dup'"""
    state = TCPState()
    history = [state.cwnd]

    for event in events:
        if state.mode == "slow_start":
            if event == "ack":
                state.cwnd += 1  # 지수 증가
                if state.cwnd >= state.ssthresh:
                    state.mode = "congestion_avoidance"
            elif event in ("timeout", "triple_dup"):
                state.ssthresh = max(state.cwnd / 2, 2)
                state.cwnd = 1
                state.mode = "slow_start"

        elif state.mode == "congestion_avoidance":
            if event == "ack":
                state.cwnd += 1 / state.cwnd  # 선형 증가 근사
            elif event in ("timeout", "triple_dup"):
                state.ssthresh = max(state.cwnd / 2, 2)
                state.cwnd = 1  # Tahoe는 항상 1로 리셋
                state.mode = "slow_start"

        history.append(round(state.cwnd, 2))

    return history


def simulate_reno(events: List[str]) -> List[float]:
    """TCP Reno 시뮬레이션"""
    state = TCPState()
    history = [state.cwnd]

    for event in events:
        if state.mode == "slow_start":
            if event == "ack":
                state.cwnd += 1
                if state.cwnd >= state.ssthresh:
                    state.mode = "congestion_avoidance"
            elif event == "timeout":
                state.ssthresh = max(state.cwnd / 2, 2)
                state.cwnd = 1
                state.mode = "slow_start"
            elif event == "triple_dup":
                state.ssthresh = max(state.cwnd / 2, 2)
                state.cwnd = state.ssthresh  # Reno: ssthresh로 설정 (1이 아님!)
                state.mode = "congestion_avoidance"

        elif state.mode == "congestion_avoidance":
            if event == "ack":
                state.cwnd += 1 / state.cwnd
            elif event == "timeout":
                state.ssthresh = max(state.cwnd / 2, 2)
                state.cwnd = 1
                state.mode = "slow_start"
            elif event == "triple_dup":
                state.ssthresh = max(state.cwnd / 2, 2)
                state.cwnd = state.ssthresh  # 절반으로만 감소
                state.mode = "congestion_avoidance"

        history.append(round(state.cwnd, 2))

    return history


# 시뮬레이션: 일반 ACK 10개 → Triple Dup → ACK 10개 → 타임아웃
events = (["ack"] * 10 + ["triple_dup"] + ["ack"] * 10 + ["timeout"] + ["ack"] * 5)

tahoe = simulate_tahoe(events)
reno = simulate_reno(events)

print("Round | Tahoe CWND | Reno CWND")
print("-" * 35)
for i, (t, r) in enumerate(zip(tahoe, reno)):
    marker = " ← triple_dup" if i == 11 else (" ← timeout" if i == 22 else "")
    print(f"{i:5} | {t:10.2f} | {r:9.2f}{marker}")
```

**출력 (일부):**
```
Round | Tahoe CWND | Reno CWND
-----------------------------------
    0 |       1.00 |      1.00
    5 |       6.00 |      6.00
   10 |      11.00 |     11.00
   11 |       1.00 |      5.50 ← triple_dup  (Reno가 훨씬 덜 감소)
   22 |       1.00 |      1.00 ← timeout
```

## TCP CUBIC: 고속 장거리 네트워크 대응 (2008)

Reno의 선형 증가는 **고대역폭-고지연(High-BDP: Bandwidth-Delay Product) 환경**에서 너무 느리다. 예를 들어 10Gbps 링크, RTT 100ms 환경에서 패킷 손실 후 Reno가 최대 속도를 회복하는 데 수십 분이 걸릴 수 있다.

CUBIC은 CWND 성장에 **3차 함수(cubic function)**를 사용한다:

```
W(t) = C × (t - K)³ + W_max

여기서:
  W_max = 마지막 혼잡 발생 시 CWND
  K = ³√(W_max × β / C)  (CWND가 W_max에 도달하는 시간)
  C = 0.4 (스케일링 상수)
  β = 0.7 (혼잡 감소 계수)
```

이 공식의 의미:
- 혼잡 직후: W_max보다 훨씬 아래 → 급격히 증가 (오목 구간)
- W_max 근처: 완만하게 증가 (안전 탐색)
- W_max 초과 후: 다시 급격히 증가 (새 대역폭 탐색)

```
CWND
 |          *
 |        *
W_max ----*---------
 |      *
 |    *   *
 |  *       *
 | *           *
 +------------------------→ 시간
   손실 발생  K  W_max 도달
```

CUBIC은 Linux 2.6.19 (2007)부터 기본 알고리즘이 됐고, RFC 9438(2023)로 표준화됐다.

## TCP BBR: 패러다임 전환 (Google, 2016)

Reno와 CUBIC의 근본적 한계는 **패킷 손실을 혼잡의 신호로 사용**한다는 점이다. 현대 네트워크에서 패킷 손실은 혼잡 외에 무선 오류, 패킷 재정렬 등 다양한 이유로 발생한다. 더 나쁜 것은 대용량 버퍼(Bufferbloat)가 있는 라우터에서 큐에 가득 차도 패킷 손실이 발생하지 않아 불필요하게 높은 지연을 유발한다.

BBR(Bottleneck Bandwidth and Round-trip propagation time)은 완전히 다른 접근을 취한다:

> **"네트워크 경로의 최대 처리량(BtlBw)과 최소 왕복 시간(RTprop)을 직접 모델링하고, 그 최적점에서 동작하라"**

### BBR의 핵심 메커니즘

```python
class BBRState:
    """BBR 동작 단계 개념 시뮬레이션"""

    def __init__(self):
        self.btl_bw = 0       # 병목 대역폭 추정 (bytes/sec)
        self.rt_prop = float('inf')  # 최소 RTT 추정 (seconds)
        self.phase = "STARTUP"
        self.pacing_gain = 2.89  # STARTUP: 지수 증가
        self.cwnd_gain = 2.89

    def update_model(self, delivered: int, rtt: float, elapsed: float):
        """각 ACK에서 BtlBw와 RTprop 갱신"""
        bw_sample = delivered / elapsed
        self.btl_bw = max(self.btl_bw, bw_sample)
        self.rt_prop = min(self.rt_prop, rtt)

    def compute_pacing_rate(self) -> float:
        return self.btl_bw * self.pacing_gain

    def compute_cwnd(self) -> float:
        bdp = self.btl_bw * self.rt_prop  # Bandwidth-Delay Product
        return bdp * self.cwnd_gain

    def transition_to_drain(self):
        """STARTUP에서 큐가 가득 찼을 때 DRAIN으로 전환"""
        self.phase = "DRAIN"
        self.pacing_gain = 1 / 2.89  # 역수로 큐 비우기

    def transition_to_probe_bw(self):
        """정상 상태: 8개 RTT 사이클로 대역폭 탐색"""
        self.phase = "PROBE_BW"
        # pacing_gain을 주기적으로 변경: 1.25 → 0.75 → 1.0 × 6 반복
```

BBR의 단계:
1. **STARTUP**: 지수 증가로 BtlBw 발견 (Reno의 슬로 스타트 유사)
2. **DRAIN**: 과도한 큐 비우기
3. **PROBE_BW**: 주기적으로 대역폭 탐색 (기본 상태)
4. **PROBE_RTT**: 최소 RTT 측정을 위해 CWND 최소화

### BBR의 장점

- **낮은 지연**: 버퍼를 가득 채우지 않아 Bufferbloat 없음
- **랜덤 패킷 손실 내성**: 손실이 아닌 RTT/BW로 혼잡 판단
- **고손실 환경 성능**: 위성 링크 등 1~2% 손실 환경에서도 Reno/CUBIC의 100배 이상 처리량

Google은 BBR을 YouTube, Google.com에 적용해 전 세계 평균 처리량 4% 향상, 지연 33% 감소를 보고했다.

## 알고리즘 비교 요약

| 알고리즘 | 혼잡 신호 | CWND 증가 | 주요 환경 | Linux 기본 |
|---------|---------|---------|---------|---------|
| Tahoe | 패킷 손실 | 선형 | 역사적 | 아니오 |
| Reno | 패킷 손실 | 선형 (AIMD) | LAN | 아니오 |
| CUBIC | 패킷 손실 | 3차 함수 | 고BDP WAN | 2.6.19~ |
| BBR v1 | RTT+BW 모델 | 모델 기반 | 고속/고손실 | 선택적 |

## Linux에서 혼잡 제어 알고리즘 변경

```bash
# 현재 사용 중인 알고리즘 확인
sysctl net.ipv4.tcp_congestion_control
# → net.ipv4.tcp_congestion_control = cubic

# 사용 가능한 알고리즘 목록
sysctl net.ipv4.tcp_available_congestion_control
# → net.ipv4.tcp_available_congestion_control = reno cubic bbr

# BBR로 변경 (임시)
sudo sysctl -w net.ipv4.tcp_congestion_control=bbr

# BBR 영구 적용
echo "net.ipv4.tcp_congestion_control=bbr" | sudo tee -a /etc/sysctl.conf
echo "net.core.default_qdisc=fq" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

BBR은 FQ(Fair Queue) 패킷 스케줄러와 함께 사용할 때 최적 성능을 발휘한다. BBR은 패킷 페이싱(pacing)을 사용하는데, FQ가 이를 커널 수준에서 정확하게 구현해주기 때문이다.

## 주의사항과 팁

1. **BBR과 CUBIC의 공존 문제**: 같은 병목 링크를 BBR과 CUBIC 연결이 공유하면 BBR이 불공평하게 더 많은 대역폭을 가져갈 수 있다. 모든 연결이 동일한 알고리즘을 사용하는 것이 이상적이다.

2. **BBRv1 vs BBRv2**: BBRv1은 얕은 버퍼 환경에서 패킷 손실을 과도하게 발생시킬 수 있다. Google이 개발 중인 BBRv2는 이 문제를 패킷 손실 신호와 혼합하여 개선했다.

3. **ECN(Explicit Congestion Notification) 활성화**: ECN은 라우터가 패킷 손실 없이 혼잡을 알릴 수 있게 해준다. CUBIC과 함께 활성화하면 성능 향상 효과가 있다.
   ```bash
   sudo sysctl -w net.ipv4.tcp_ecn=1
   ```

4. **측정이 먼저다**: 알고리즘을 바꾸기 전에 `ss -tin` 또는 `tc -s qdisc`로 현재 RTT, 혼잡 윈도우, 재전송률을 측정하라. 환경에 따라 최적 알고리즘이 다르다.

5. **QUIC과 HTTP/3**: QUIC은 UDP 위에서 혼잡 제어를 애플리케이션 레벨로 올려 더 빠른 알고리즘 혁신을 가능하게 한다. Google의 gQUIC, Cloudflare의 QUIC는 모두 BBR 변형을 기본으로 사용한다.

## 참고 자료
- [RFC 9438: CUBIC for Fast Long-Distance Networks (IETF)](https://datatracker.ietf.org/doc/html/rfc9438)
- [TCP Congestion Control - Wikipedia](https://en.wikipedia.org/wiki/TCP_congestion_control)
- [BBR: Congestion-Based Congestion Control - Google Research](https://research.google/pubs/bbr-congestion-based-congestion-control/)
- [RFC 9743: Considerations for the Use of a TCP-like Transport Protocol (IETF)](https://datatracker.ietf.org/doc/html/rfc9743)
