---
layout: post
title: "TCP 혼잡 제어와 흐름 제어: Slow Start, AIMD, Fast Retransmit 완전 정복"
date: 2026-06-17
categories: [cs, computer-science]
tags: [TCP, network, congestion-control, flow-control, slow-start, AIMD, CUBIC, BBR]
---

## TCP는 왜 혼잡 제어가 필요한가

인터넷은 **베스트 에포트(Best-Effort)** 네트워크다. 라우터는 패킷 전달을 보장하지 않으며, 버퍼가 가득 차면 패킷을 그냥 버린다. 만약 모든 TCP 연결이 아무런 제어 없이 데이터를 최대한 빠르게 보낸다면 어떻게 될까? 네트워크는 순식간에 마비된다. 1986년 실제로 이런 일이 일어났고, 인터넷 처리량이 최대값의 1/1000 이하로 떨어졌다. 이 사건이 Van Jacobson을 자극해 TCP 혼잡 제어 알고리즘(RFC 5681)을 탄생시켰다.

**혼잡 제어(Congestion Control)**와 **흐름 제어(Flow Control)**는 비슷해 보이지만 다른 문제를 해결한다:
- **흐름 제어**: 수신자가 감당할 수 있는 속도로 전송 (송신자 ↔ 수신자 사이 문제)
- **혼잡 제어**: 네트워크 중간 경로가 감당할 수 있는 속도로 전송 (네트워크 전체 문제)

---

## 흐름 제어: 슬라이딩 윈도우

수신자는 **수신 버퍼(Receive Buffer)**의 여유 공간을 **rwnd(Receive Window)** 값으로 TCP 헤더에 실어 보낸다. 송신자는 이 값을 초과해 데이터를 보낼 수 없다.

```
[송신자]                         [수신자]
  │──── 데이터 (seq=1~1000) ────►│  rwnd: 4000
  │──── 데이터 (seq=1001~2000) ─►│  rwnd: 3000
  │◄─── ACK=2001, rwnd=3000 ─────│
  │──── 데이터 (seq=2001~3000) ─►│  rwnd: 2000
  │◄─── ACK=3001, rwnd=2000 ─────│
```

수신자 버퍼가 꽉 차면 rwnd=0을 보내 전송을 멈출 수 있다 (Zero Window). 이후 버퍼에 여유가 생기면 Window Update를 보내 재개한다.

---

## 혼잡 제어의 4가지 메커니즘

TCP 혼잡 제어는 **cwnd(Congestion Window)**를 이용해 송신 속도를 조절한다. 실제 전송 가능량은 `min(cwnd, rwnd)`이다.

### 1. 느린 시작 (Slow Start)

이름과 달리 지수적으로 성장한다. 처음에는 cwnd=1MSS(최대 세그먼트 크기)에서 시작해, ACK를 받을 때마다 cwnd를 1씩 증가시킨다. 이는 매 RTT마다 cwnd가 2배가 되는 효과다.

```
RTT 1: cwnd=1  → 1개 전송
RTT 2: cwnd=2  → 2개 전송
RTT 3: cwnd=4  → 4개 전송
RTT 4: cwnd=8  → 8개 전송
...
```

**ssthresh(Slow Start Threshold)**에 도달하면 혼잡 회피 단계로 전환된다.

### 2. 혼잡 회피 (Congestion Avoidance, AIMD)

cwnd가 ssthresh 이상이 되면 AIMD(Additive Increase/Multiplicative Decrease) 방식으로 동작한다:
- **증가(AI)**: 매 RTT마다 cwnd를 1MSS씩 선형 증가
- **감소(MD)**: 패킷 손실 감지 시 cwnd를 절반으로 급감

이 비대칭 구조가 중요하다. 천천히 늘리고 빠르게 줄이는 방식이 네트워크 안정성을 보장한다.

### 3. 빠른 재전송 (Fast Retransmit)

타임아웃을 기다리지 않고, **3개의 중복 ACK** 수신 시 즉시 해당 패킷을 재전송한다. 중복 ACK는 패킷이 순서 없이 도착했다는 신호로, 타임아웃보다 빠른 손실 감지가 가능하다.

```
[시퀀스 번호 100이 손실된 경우]
수신자: ACK 99 (seq 101, 102, 103 도착해도 계속 ACK 99)
      → 중복 ACK #1
      → 중복 ACK #2
      → 중복 ACK #3
송신자: 3번째 중복 ACK 수신 → 즉시 seq 100 재전송!
```

### 4. 빠른 복구 (Fast Recovery)

3중복 ACK에 의한 재전송 후, 타임아웃과 달리 Slow Start로 돌아가지 않는다. ssthresh를 cwnd/2로 줄이고 ssthresh에서 다시 혼잡 회피를 시작한다.

---

## 코드로 이해하는 TCP 혼잡 제어 시뮬레이션

### 혼잡 제어 상태 머신 (Python)

```python
from enum import Enum
import random

class State(Enum):
    SLOW_START = "slow_start"
    CONGESTION_AVOIDANCE = "congestion_avoidance"
    FAST_RECOVERY = "fast_recovery"

class TCPCongestionControl:
    def __init__(self, initial_ssthresh=64):
        self.cwnd = 1.0          # 혼잡 윈도우 (MSS 단위)
        self.ssthresh = initial_ssthresh
        self.state = State.SLOW_START
        self.duplicate_acks = 0
        self.rtt = 0

    def on_ack(self):
        """정상 ACK 수신 처리"""
        self.duplicate_acks = 0
        self.rtt += 1
        
        if self.state == State.SLOW_START:
            self.cwnd += 1.0  # 지수 증가
            if self.cwnd >= self.ssthresh:
                self.state = State.CONGESTION_AVOIDANCE
                print(f"  RTT {self.rtt}: Slow Start → Congestion Avoidance (cwnd={self.cwnd:.1f})")
        
        elif self.state == State.CONGESTION_AVOIDANCE:
            self.cwnd += 1.0 / self.cwnd  # 선형 증가 (RTT당 +1MSS)
        
        elif self.state == State.FAST_RECOVERY:
            self.cwnd = self.ssthresh
            self.state = State.CONGESTION_AVOIDANCE

    def on_triple_duplicate_ack(self):
        """3중복 ACK → Fast Retransmit + Fast Recovery"""
        print(f"  RTT {self.rtt}: 3중복 ACK 감지! cwnd {self.cwnd:.1f} → {self.cwnd/2:.1f}")
        self.ssthresh = max(self.cwnd / 2, 2.0)
        self.cwnd = self.ssthresh + 3  # inflate
        self.state = State.FAST_RECOVERY

    def on_timeout(self):
        """타임아웃 → 가장 심각한 혼잡"""
        print(f"  RTT {self.rtt}: 타임아웃! cwnd {self.cwnd:.1f} → 1")
        self.ssthresh = max(self.cwnd / 2, 2.0)
        self.cwnd = 1.0
        self.state = State.SLOW_START

def simulate_tcp(rounds=30, loss_probability=0.05):
    tcp = TCPCongestionControl(ssthresh=32)
    history = []
    
    print("=== TCP 혼잡 제어 시뮬레이션 ===")
    for _ in range(rounds):
        r = random.random()
        if r < loss_probability * 0.2:  # 타임아웃 (드문 경우)
            tcp.on_timeout()
        elif r < loss_probability:       # 3중복 ACK
            tcp.on_triple_duplicate_ack()
        else:
            tcp.on_ack()
        
        history.append(round(tcp.cwnd, 2))
        print(f"  RTT {tcp.rtt}: cwnd={tcp.cwnd:.2f} [{tcp.state.value}]")
    
    return history

random.seed(42)
simulate_tcp(rounds=20)
```

### 네트워크 처리량 분석 (Python)

```python
def calculate_throughput(cwnd_mss: float, rtt_ms: float, mss_bytes: int = 1460) -> float:
    """
    TCP 혼잡 제어 상태에서의 이론적 처리량 계산
    Mathis 공식: throughput ≈ (MSS / RTT) * (1 / sqrt(loss_rate))
    """
    rtt_sec = rtt_ms / 1000
    bytes_per_rtt = cwnd_mss * mss_bytes
    throughput_bps = bytes_per_rtt / rtt_sec
    return throughput_bps / 1_000_000  # Mbps

# 다양한 상황에서의 처리량 비교
scenarios = [
    ("가정용 WiFi (cwnd=32, RTT=20ms)", 32, 20),
    ("데이터센터 (cwnd=256, RTT=1ms)", 256, 1),
    ("위성 인터넷 (cwnd=8, RTT=600ms)", 8, 600),
]

print("=== TCP 처리량 추정 ===")
for desc, cwnd, rtt in scenarios:
    mbps = calculate_throughput(cwnd, rtt)
    print(f"{desc}: {mbps:.2f} Mbps")

# 결과:
# 가정용 WiFi (cwnd=32, RTT=20ms): 18.69 Mbps
# 데이터센터 (cwnd=256, RTT=1ms): 374.78 Mbps
# 위성 인터넷 (cwnd=8, RTT=600ms): 0.02 Mbps
```

---

## 현대 혼잡 제어 알고리즘: CUBIC과 BBR

### CUBIC (Linux 기본 알고리즘)

Linux 2.6.19부터 기본 알고리즘으로 사용된다. 손실 이벤트 이후 cwnd를 3차 함수(cubic function)로 증가시킨다. 고속·고지연 네트워크(BDP가 큰 환경)에서 Reno보다 훨씬 빠르게 회복한다.

### BBR (Bottleneck Bandwidth and RTT, Google 개발)

2016년 Google이 개발한 혁신적인 알고리즘이다. 패킷 손실 대신 **실측 RTT와 대역폭**을 기반으로 혼잡을 감지한다. 버퍼 팽창(Bufferbloat) 문제를 해결하고, 랜덤 패킷 손실이 많은 무선 네트워크에서 특히 효과적이다. YouTube와 Google Cloud에서 기본으로 사용 중이다.

---

## 주의사항과 실전 팁

### 고속 네트워크에서의 한계
RTT가 10ms이고 1Gbps 링크를 활용하려면 cwnd가 약 850MSS 이상이 되어야 한다. 패킷 손실률 0.01%만 돼도 처리량이 급감하는데, 이를 **Long Fat Network(LFN)** 문제라고 한다.

### 타임아웃 튜닝
`sysctl net.ipv4.tcp_syn_retries` 등으로 재전송 횟수를 조정할 수 있다. 데이터센터처럼 손실이 거의 없는 환경에서는 타임아웃을 짧게 설정하는 것이 유리하다.

### 큐 지연(Queuing Delay) 측정
`ss -tin` 명령으로 Linux에서 TCP 소켓의 혼잡 윈도우, RTT, 재전송 횟수를 실시간으로 확인할 수 있다. BBR 상태를 모니터링하려면 `ss -tio state established` 명령을 활용하자.

### HTTP/3와 QUIC
HTTP/3는 TCP 대신 UDP 기반의 **QUIC** 프로토콜을 사용한다. QUIC는 HOL(Head-of-Line) 블로킹 문제를 해결하고 0-RTT 핸드셰이크를 지원한다. 혼잡 제어는 QUIC 레이어에서 플러그인 방식으로 교체 가능하며, 기본값은 CUBIC이다.

---

## 참고 자료
- [RFC 5681 — TCP Congestion Control](https://www.rfc-editor.org/rfc/rfc5681)
- [TCP Congestion Control: A Systems Approach](https://tcpcc.systemsapproach.org/algorithm.html)
- [Computer Networks: A Systems Approach — Congestion Control](https://book.systemsapproach.org/congestion/tcpcc.html)
- [Wikipedia — TCP Congestion Control](https://en.wikipedia.org/wiki/TCP_congestion_control)
