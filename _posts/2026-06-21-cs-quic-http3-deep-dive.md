---
layout: post
title: "QUIC 프로토콜 심화: HTTP/3와 UDP 기반 차세대 전송 계층 완전 정복"
date: 2026-06-21
categories: [cs, computer-science]
tags: [quic, http3, network, protocol, udp, tls, multiplexing, rfc9000]
---

2021년 5월 RFC 9000으로 표준화된 QUIC(Quick UDP Internet Connections)은 구글이 2012년 처음 실험적으로 도입한 뒤 IETF가 표준화한 차세대 전송 계층 프로토콜이다. HTTP/3의 기반 전송 계층으로 채택되어 2024년 기준 전 세계 상위 1000만 개 사이트 중 약 34%가 이미 HTTP/3를 지원하고, 크롬·파이어폭스·사파리 등 주요 브라우저의 95% 이상이 활성화하고 있다.

이 아티클에서는 QUIC이 TCP의 어떤 문제를 해결하는지, 핵심 메커니즘(0-RTT, 멀티플렉싱, 연결 마이그레이션)은 어떻게 동작하는지, 실제 Go 서버 구현 예제와 함께 완전히 살펴본다.

---

## TCP의 문제점: 왜 QUIC이 등장했는가?

### 문제 1: HOL 블로킹 (Head-of-Line Blocking)

HTTP/2는 하나의 TCP 연결 위에 여러 스트림을 다중화(Multiplexing)해 HTTP/1.1의 HOL 블로킹을 해결했다. 그러나 TCP 레이어에서 패킷 손실이 발생하면 손실된 세그먼트가 재전송될 때까지 **모든 스트림**이 멈춘다. TCP는 스트림 개념을 모르기 때문이다.

```
HTTP/2 위 TCP에서의 HOL 블로킹:
스트림A [데이터1] [데이터2] [X손실X] [데이터4]
스트림B [데이터1] [데이터2] [  ←  대기  →  ] ← 영향 없는데도 블록됨
스트림C [데이터1] [데이터2] [  ←  대기  →  ]
```

QUIC은 스트림을 전송 계층 자체에서 인지하므로, 한 스트림의 패킷 손실이 다른 스트림에 영향을 주지 않는다.

### 문제 2: 연결 수립 지연 (Handshake Latency)

TCP + TLS 1.3 조합에서도 첫 연결 수립에 최소 1-RTT(TCP) + 1-RTT(TLS 1.3) = **2 RTT**가 필요하다. 모바일 네트워크처럼 RTT가 50~100ms인 환경에서는 연결 수립만으로 100~200ms가 낭비된다.

QUIC은 TLS 1.3을 전송 계층에 내장하여 최초 연결도 **1-RTT**, 이전에 연결한 적 있는 서버라면 **0-RTT**로 데이터를 전송할 수 있다.

### 문제 3: 연결 마이그레이션 불가

TCP 연결은 4-튜플(src IP, src Port, dst IP, dst Port)로 식별된다. Wi-Fi에서 LTE로 전환되면 IP가 바뀌어 연결이 끊기고 재연결해야 한다.

QUIC은 **연결 ID(Connection ID)** 를 사용해 IP/포트가 바뀌어도 연결을 유지한다. 모바일 환경에서 끊김 없는 스트리밍이 가능한 이유다.

---

## QUIC 핵심 메커니즘

### 1. 0-RTT 연결 수립

```
최초 연결 (1-RTT):
클라이언트                    서버
    |── Initial (CRYPTO) ──→   |   ← TLS ClientHello 포함
    |← Handshake (CRYPTO) ──   |   ← TLS ServerHello + 인증서
    |── Handshake Complete ──→  |
    |──── 애플리케이션 데이터 ────→  |  ← 1-RTT

재연결 (0-RTT):
클라이언트                    서버
    |── Initial + 0-RTT 데이터 ──→  |  ← 즉시 데이터 전송
    |← (확인 응답)  ────────────   |  ← 0-RTT
```

0-RTT는 이전 세션의 **PSK(Pre-Shared Key)** 를 재사용하므로 리플레이 공격(Replay Attack)에 취약하다. 따라서 멱등성이 없는 요청(POST, 결제 등)에는 0-RTT를 사용해서는 안 된다.

### 2. 스트림 기반 멀티플렉싱

QUIC 스트림은 단방향/양방향, 클라이언트/서버 개시 여부에 따라 네 종류로 분류된다. 스트림 ID는 62비트로 최대 약 4.6 × 10^18개의 스트림을 지원한다.

| 스트림 ID 하위 2비트 | 종류 |
|---|---|
| 0x0 | 클라이언트 개시, 양방향 |
| 0x1 | 서버 개시, 양방향 |
| 0x2 | 클라이언트 개시, 단방향 |
| 0x3 | 서버 개시, 단방향 |

각 스트림은 독립적인 흐름 제어를 가지며, 한 스트림의 패킷 손실은 해당 스트림의 수신 버퍼만 블록한다.

### 3. 패킷 번호 공간 분리

QUIC은 세 개의 독립적인 패킷 번호 공간을 사용한다:

- **Initial 공간**: TLS 핸드셰이크 초기 메시지
- **Handshake 공간**: TLS 핸드셰이크 완료 메시지
- **Application Data 공간**: 실제 애플리케이션 데이터

각 공간에서 패킷 번호는 단조 증가하며, TCP처럼 ACK 번호가 모호해지는 문제가 없다. 재전송된 패킷도 새 번호를 받아 RTT 측정이 정확하다.

### 4. 내장 암호화 (Always-On Encryption)

QUIC은 TLS 1.3을 필수로 내장하므로 암호화를 선택적으로 끌 수 없다. 헤더 일부조차 암호화되어 미들박스(방화벽, 라우터)가 패킷 내용을 파악하기 어렵다. 이는 보안 강점이지만 기업 네트워크의 DPI(Deep Packet Inspection) 장비와 충돌하는 이유이기도 하다.

---

## 구현 예제 1: Go로 QUIC HTTP/3 서버 구축

```go
package main

import (
    "crypto/tls"
    "fmt"
    "log"
    "net/http"

    "github.com/quic-go/quic-go/http3"
)

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        proto := r.Proto
        fmt.Fprintf(w, "Hello via %s!\n", proto)

        // HTTP/3 Push 예시 (서버 푸시)
        if pusher, ok := w.(http.Pusher); ok {
            pusher.Push("/style.css", nil)
        }
    })
    mux.HandleFunc("/ping", func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "application/json")
        fmt.Fprint(w, `{"status":"ok","protocol":"HTTP/3"}`)
    })

    // TLS 인증서 로드 (실제 환경: Let's Encrypt 등 사용)
    tlsConfig := &tls.Config{
        MinVersion: tls.VersionTLS13, // QUIC은 TLS 1.3 필수
    }

    server := &http3.Server{
        Addr:      ":443",
        Handler:   mux,
        TLSConfig: tlsConfig,
    }

    log.Println("HTTP/3 서버 시작: https://localhost:443")

    // UDP :443에서 QUIC 수신
    if err := server.ListenAndServeTLS("server.crt", "server.key"); err != nil {
        log.Fatalf("서버 오류: %v", err)
    }
}
```

`quic-go` 라이브러리를 사용하면 Go에서 몇 십 줄로 HTTP/3 서버를 구축할 수 있다. 클라이언트가 HTTP/2로 최초 접속하면 서버는 `Alt-Svc: h3=":443"` 응답 헤더로 HTTP/3 지원을 광고하고, 브라우저는 다음 요청부터 QUIC으로 업그레이드한다.

---

## 구현 예제 2: Python으로 QUIC 연결 시뮬레이션 및 성능 측정

```python
import time
import asyncio
from aioquic.asyncio import connect
from aioquic.asyncio.protocol import QuicConnectionProtocol
from aioquic.quic.configuration import QuicConfiguration
from aioquic.h3.connection import H3_ALPN


class HttpClient(QuicConnectionProtocol):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self._response_data = b""
        self._stream_id = None

    async def get(self, path: str) -> bytes:
        # HTTP/3 GET 요청 전송
        self._stream_id = self._quic.get_next_available_stream_id()
        headers = [
            (b":method",    b"GET"),
            (b":scheme",    b"https"),
            (b":path",      path.encode()),
            (b":authority", b"example.com"),
        ]
        self._quic.send_stream_data(
            self._stream_id,
            b"",  # HTTP/3은 헤더와 바디를 QPACK으로 인코딩
            end_stream=True,
        )
        await asyncio.sleep(0.1)  # 응답 대기
        return self._response_data


async def benchmark_quic(host: str, port: int, paths: list[str]):
    config = QuicConfiguration(
        alpn_protocols=H3_ALPN,
        is_client=True,
        verify_mode=False,  # 테스트 환경
        max_datagram_size=1350,  # MTU - QUIC 오버헤드
    )

    results = []
    start_total = time.perf_counter()

    async with connect(host, port, configuration=config,
                       create_protocol=HttpClient) as client:
        # 모든 요청을 동시에 보냄 (HOL 블로킹 없음)
        tasks = [client.get(path) for path in paths]
        responses = await asyncio.gather(*tasks)

    elapsed = time.perf_counter() - start_total

    for path, resp in zip(paths, responses):
        results.append({
            "path":    path,
            "size":    len(resp),
        })

    print(f"총 {len(paths)}개 요청 완료: {elapsed*1000:.1f}ms")
    print(f"평균 요청당: {elapsed*1000/len(paths):.1f}ms")
    return results


if __name__ == "__main__":
    paths = ["/api/users", "/api/posts", "/api/comments",
             "/api/tags",  "/api/search"]
    asyncio.run(benchmark_quic("localhost", 4433, paths))
```

`aioquic` 라이브러리로 QUIC 클라이언트를 구현할 수 있다. 실제 환경에서 HTTP/2 대비 패킷 손실 1% 시나리오에서 QUIC은 약 20~30% 빠른 응답 시간을 보인다.

---

## QUIC 배포 시 주의사항 및 팁

### 1. UDP 블로킹 방화벽 주의
기업 방화벽은 UDP 443을 차단하는 경우가 많다. 브라우저는 QUIC 연결 실패 시 자동으로 TCP/TLS로 폴백하지만, 이를 위해 반드시 HTTP/2 폴백을 유지해야 한다. `Alt-Svc` 헤더가 그 역할을 한다.

### 2. 리플레이 공격 방지
0-RTT 데이터는 리플레이 공격에 취약하다. 서버는 0-RTT 요청에 대해:
- 멱등성이 있는 요청(GET, HEAD)만 처리
- 0-RTT 토큰에 만료 시간 설정 (`max_early_data` 제한)
- 중복 토큰 감지를 위한 서버 측 블룸 필터 활용

### 3. 혼잡 제어 알고리즘 선택
QUIC은 플러거블 혼잡 제어를 지원한다. 기본값은 New Reno이지만 BBR, CUBIC 등을 선택할 수 있다. 구글은 BBR v2를 사용해 장거리 고대역폭 환경에서 TCP 대비 최대 4배 처리량 향상을 보고했다.

### 4. CPU 오버헤드
UDP를 직접 처리하고 TLS 암호화를 유저스페이스에서 수행하므로 TCP+TLS 대비 CPU 사용량이 높다. 고트래픽 서버에서는 kernel bypass(DPDK, io_uring)와 함께 사용하거나 하드웨어 TLS 오프로딩을 고려해야 한다.

### 5. 로드밸런서 설정
QUIC의 연결 ID 기반 라우팅을 지원하는 로드밸런서가 필요하다. HAProxy 2.6+, nginx 1.25+, Caddy가 QUIC/HTTP3를 지원한다. 연결 ID를 일관성 해싱의 키로 사용하면 세션 유지(Sticky Session)가 가능하다.

---

## 참고 자료
- [RFC 9000 - QUIC: A UDP-Based Multiplexed and Secure Transport](https://datatracker.ietf.org/doc/html/rfc9000)
- [RFC 9114: HTTP/3 - RFC Editor](https://www.rfc-editor.org/rfc/rfc9114.html)
- [HTTP/3 - Wikipedia](https://en.wikipedia.org/wiki/HTTP/3)
- [IETF QUIC Working Group](https://quicwg.org/)
