---
layout: post
title: "TCP TIME_WAIT 딥다이브: 연결 생명주기와 고성능 서버 최적화"
date: 2026-10-27
categories: [cs, computer-science]
tags: [tcp, network, time-wait, linux, socket, performance]
---

TCP는 신뢰성 있는 연결 지향 프로토콜로, 연결을 맺고 끊는 과정에서 엄격한 상태 관리를 수행한다. 그 중 **TIME_WAIT** 상태는 많은 개발자에게 골칫거리가 되는 개념이다. 초당 수만 건의 연결을 처리하는 고성능 서버를 운영하다 보면 "address already in use" 오류나 연결 지연 현상으로 이 상태와 맞닥뜨리게 된다. 이 글에서는 TCP 연결 생명주기의 전체 흐름과 TIME_WAIT가 왜 필요한지, 그리고 실무에서 어떻게 최적화하는지 깊이 있게 다룬다.

## TCP 연결 생명주기: 상태 다이어그램 이해

TCP 연결은 아래의 상태 머신을 거친다. 연결 수립(3-way handshake)과 연결 종료(4-way handshake) 과정이 모두 상태를 통해 관리된다.

```
CLOSED → LISTEN (서버, bind/listen 호출)
CLOSED → SYN_SENT (클라이언트, connect 호출)

[3-way Handshake]
클라이언트: CLOSED → SYN_SENT → ESTABLISHED
서버:       LISTEN → SYN_RCVD → ESTABLISHED

[4-way Handshake — 능동 종료 측]
ESTABLISHED → FIN_WAIT_1 → FIN_WAIT_2 → TIME_WAIT → CLOSED

[4-way Handshake — 수동 종료 측]
ESTABLISHED → CLOSE_WAIT → LAST_ACK → CLOSED
```

### 4-way Handshake 상세

연결 종료는 half-close 방식으로 동작한다. 각 방향의 스트림이 독립적으로 종료된다.

```
클라이언트(능동종료)          서버(수동종료)
     |                            |
     |--- FIN, seq=x ----------->|   FIN_WAIT_1 → FIN_WAIT_2
     |<-- ACK, ack=x+1 ----------|   CLOSE_WAIT
     |                            |   (서버가 남은 데이터 전송)
     |<-- FIN, seq=y ------------|   LAST_ACK
     |--- ACK, ack=y+1 ---------->|
     |                            |   CLOSED
     | (2MSL 대기)                |
     |                            |
  TIME_WAIT
  (→ CLOSED after 2MSL)
```

능동적으로 연결을 먼저 닫는 쪽(active closer)이 TIME_WAIT 상태로 진입하며, 여기서 **MSL(Maximum Segment Lifetime)**의 두 배 시간(기본 60초)을 대기한 후 CLOSED로 전이한다.

## TIME_WAIT가 반드시 필요한 이유

TIME_WAIT는 두 가지 핵심 목적을 위해 존재한다.

### 1. 지연 세그먼트(Delayed Segment) 처리

네트워크에서 패킷은 다양한 경로를 거쳐 늦게 도착할 수 있다. TCP는 (source IP, source port, dest IP, dest port)의 4-tuple로 연결을 식별한다. 같은 4-tuple로 새 연결이 즉시 맺어지면, 이전 연결의 지연 패킷이 새 연결에서 잘못 처리될 수 있다.

TIME_WAIT(2MSL) 기간 동안 동일한 4-tuple로 새 연결이 생성되는 것을 차단함으로써, 네트워크 상에 남아 있는 이전 연결의 모든 패킷이 소멸되도록 보장한다.

### 2. 마지막 ACK 재전송 보장

4-way handshake의 마지막 ACK가 손실될 경우, 수동 종료 측(서버)은 FIN을 재전송한다. 능동 종료 측이 이미 CLOSED 상태라면 이 FIN에 RST로 응답하게 되어 서버가 오류 상태로 종료된다. TIME_WAIT 상태에서는 FIN을 받으면 ACK를 재전송하고 타이머를 리셋하므로 정상 종료가 보장된다.

## 고성능 서버에서의 TIME_WAIT 문제

초당 10,000개의 HTTP 연결을 처리하고 TIME_WAIT가 60초 지속된다면, 최대 600,000개의 TIME_WAIT 소켓이 동시에 존재할 수 있다. Linux의 기본 로컬 포트 범위(`ip_local_port_range`)는 약 28,000개이므로, 포트 고갈(port exhaustion)이 발생한다.

```bash
# 현재 TCP 상태별 연결 수 확인
ss -ant | awk '{print $1}' | sort | uniq -c | sort -rn

# TIME_WAIT 상세 확인
ss -ant state time-wait

# 로컬 포트 범위 확인
cat /proc/sys/net/ipv4/ip_local_port_range
# 기본값: 32768 60999 (약 28,000개)
```

## 코드 예제 1: TIME_WAIT 소켓 재사용 설정 (C)

```c
#include <sys/socket.h>
#include <netinet/in.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>

int create_reusable_server_socket(int port) {
    int fd = socket(AF_INET, SOCK_STREAM, 0);
    if (fd < 0) {
        perror("socket");
        return -1;
    }

    // SO_REUSEADDR: TIME_WAIT 상태의 소켓과 동일한 주소로 bind 허용
    int opt = 1;
    if (setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt)) < 0) {
        perror("setsockopt SO_REUSEADDR");
        close(fd);
        return -1;
    }

    // SO_REUSEPORT: 동일 포트에 여러 소켓 bind 허용 (커널 4.x+)
    // 멀티 쓰레드 accept 성능 향상에 유용
    if (setsockopt(fd, SOL_SOCKET, SO_REUSEPORT, &opt, sizeof(opt)) < 0) {
        perror("setsockopt SO_REUSEPORT");
        // 선택적 기능이므로 오류를 무시할 수 있음
    }

    struct sockaddr_in addr;
    memset(&addr, 0, sizeof(addr));
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = INADDR_ANY;
    addr.sin_port = htons(port);

    if (bind(fd, (struct sockaddr*)&addr, sizeof(addr)) < 0) {
        perror("bind");
        close(fd);
        return -1;
    }

    if (listen(fd, SOMAXCONN) < 0) {
        perror("listen");
        close(fd);
        return -1;
    }

    printf("서버 소켓 생성 완료 (port=%d, SO_REUSEADDR 적용)\n", port);
    return fd;
}

int main() {
    int server_fd = create_reusable_server_socket(8080);
    if (server_fd < 0) return 1;
    // ... accept loop
    close(server_fd);
    return 0;
}
```

`SO_REUSEADDR`는 TIME_WAIT 상태의 소켓이 점유한 포트에 새로 bind하는 것을 허용한다. 단, 동일한 4-tuple(ip+port 양측)로의 연결은 여전히 커널이 차단하므로 보안상의 문제는 없다.

## 코드 예제 2: Linux 커널 파라미터 튜닝 (Python 자동화)

```python
import subprocess
import sys

SYSCTL_PARAMS = {
    # TIME_WAIT 상태 소켓 재사용 허용 (클라이언트측 아웃바운드에서 효과)
    "net.ipv4.tcp_tw_reuse": "1",
    
    # 로컬 포트 범위 확장 (기본: 32768~60999 → 1024~65535)
    "net.ipv4.ip_local_port_range": "1024 65535",
    
    # FIN_WAIT 타임아웃 단축 (기본: 60초)
    "net.ipv4.tcp_fin_timeout": "15",
    
    # TIME_WAIT 버킷 최대 개수 (메모리 허용 범위 내)
    "net.ipv4.tcp_max_tw_buckets": "262144",
    
    # TCP Keep-Alive 설정 (유휴 연결 조기 감지)
    "net.ipv4.tcp_keepalive_time": "300",
    "net.ipv4.tcp_keepalive_intvl": "30",
    "net.ipv4.tcp_keepalive_probes": "3",
}

def apply_sysctl(params: dict, dry_run: bool = False) -> None:
    for key, value in params.items():
        cmd = ["sysctl", "-w", f"{key}={value}"]
        if dry_run:
            print(f"[dry-run] {' '.join(cmd)}")
        else:
            result = subprocess.run(cmd, capture_output=True, text=True)
            if result.returncode == 0:
                print(f"✓ {key} = {value}")
            else:
                print(f"✗ {key}: {result.stderr.strip()}", file=sys.stderr)

def show_current_state() -> None:
    result = subprocess.run(
        ["ss", "-ant"],
        capture_output=True, text=True
    )
    lines = result.stdout.strip().split('\n')
    state_count: dict[str, int] = {}
    for line in lines[1:]:  # 헤더 제외
        parts = line.split()
        if parts:
            state = parts[0]
            state_count[state] = state_count.get(state, 0) + 1
    
    print("\n현재 TCP 상태 분포:")
    for state, count in sorted(state_count.items(), key=lambda x: -x[1]):
        bar = "█" * min(count // 100, 50)
        print(f"  {state:<20} {count:>6}  {bar}")

if __name__ == "__main__":
    show_current_state()
    dry_run = "--dry-run" in sys.argv
    print("\n커널 파라미터 적용 중...")
    apply_sysctl(SYSCTL_PARAMS, dry_run=dry_run)
    print("\n적용 후 상태:")
    show_current_state()
```

## 핵심 sysctl 파라미터 정리

| 파라미터 | 기본값 | 설명 | 권장값 |
|---------|--------|------|--------|
| `tcp_tw_reuse` | 0 | 아웃바운드에서 TIME_WAIT 소켓 재사용 | 1 (클라이언트) |
| `tcp_fin_timeout` | 60 | FIN_WAIT_2 최대 대기 시간(초) | 15~30 |
| `tcp_max_tw_buckets` | 131072 | TIME_WAIT 소켓 최대 개수 | 262144~1048576 |
| `ip_local_port_range` | 32768~60999 | 아웃바운드 포트 범위 | 1024~65535 |
| `tcp_keepalive_time` | 7200 | Keep-Alive 시작 시간(초) | 300 |

> **주의**: `tcp_tw_recycle`은 Linux 4.12에서 완전히 제거되었다. NAT 환경에서 심각한 패킷 드롭을 유발했기 때문이다. 절대 사용하지 말 것.

## 실무 아키텍처 패턴

### 연결 풀링 (Connection Pooling)

HTTP Keep-Alive를 활성화하면 연결을 재사용하여 TIME_WAIT 생성을 대폭 줄일 수 있다. Nginx, HAProxy, Go의 `http.Transport` 등은 기본으로 연결 풀을 관리한다.

```python
import requests
from requests.adapters import HTTPAdapter

# 연결 풀 최대 크기 지정
session = requests.Session()
adapter = HTTPAdapter(
    pool_connections=100,   # 연결 풀 개수
    pool_maxsize=100,       # 풀당 최대 연결 수
    max_retries=3
)
session.mount("http://", adapter)
session.mount("https://", adapter)

# Keep-Alive 연결 재사용 → TIME_WAIT 최소화
response = session.get("http://api.example.com/data")
```

### 서버가 먼저 닫도록 설계

HTTP/1.1에서 서버가 `Connection: close` 헤더를 보내면 서버가 먼저 종료하여 TIME_WAIT가 클라이언트에 발생한다. 대규모 서비스에서는 서버의 TIME_WAIT를 클라이언트로 이전하는 것이 효과적이다.

## 주의사항과 팁

1. **2MSL 값**: Linux에서 MSL은 기본 30초로 TIME_WAIT는 60초 지속된다. `/proc/sys/net/ipv4/tcp_fin_timeout`으로 `FIN_WAIT_2` 시간을 조절할 수 있지만, TCP 표준의 TIME_WAIT 2MSL 자체는 변경되지 않는다.

2. **SO_LINGER 사용 주의**: `l_onoff=1, l_linger=0` 설정으로 RST를 보내 TIME_WAIT를 우회할 수 있지만, 데이터 손실 위험이 있다. 프로덕션에서는 신중히 사용해야 한다.

3. **모니터링**: `ss -s`, `netstat -st`, `/proc/net/sockstat`으로 TIME_WAIT 수를 정기적으로 모니터링하라. 급증하는 경우 서버 설계나 Keep-Alive 설정을 재검토한다.

4. **ephemeral port**: 클라이언트가 서버에 연결할 때 OS가 임시 포트를 할당한다. `ip_local_port_range` 내 포트가 모두 TIME_WAIT로 고갈되면 `EADDRNOTAVAIL` 오류가 발생한다.

5. **IPv6에서의 차이**: IPv6 소켓에서도 동일한 TIME_WAIT 메커니즘이 적용된다. 다만 IPv4/IPv6 듀얼 스택 환경에서는 `IPV6_V6ONLY` 설정에 따라 소켓 공유 방식이 달라진다.

## 참고 자료
- [RFC 793 - Transmission Control Protocol](https://www.rfc-editor.org/rfc/rfc793)
- [Linux tcp(7) man page - tcp_tw_reuse, tcp_fin_timeout](https://man7.org/linux/man-pages/man7/tcp.7.html)
- [Vincent Bernat - Coping with the TCP TIME-WAIT state on busy Linux servers](https://vincent.bernat.ch/en/blog/2014-tcp-time-wait-state-linux)
- [RFC 1337 - TIME-WAIT Assassination Hazards in TCP](https://www.rfc-editor.org/rfc/rfc1337)
