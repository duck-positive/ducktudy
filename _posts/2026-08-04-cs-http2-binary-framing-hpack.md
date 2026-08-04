---
layout: post
title: "HTTP/2 바이너리 프레이밍과 HPACK 헤더 압축 완전 정복: 멀티플렉싱으로 웹을 빠르게 만든 방법"
date: 2026-08-04
categories: [cs, computer-science]
tags: [http2, hpack, network, protocol, web, rfc7540, rfc7541, multiplexing]
---

HTTP/1.1은 1999년 RFC 2616으로 표준화된 이래 20년 이상 웹을 지탱해왔다. 그러나 현대 웹 페이지가 수십~수백 개의 리소스를 로드하면서 HTTP/1.1의 구조적 한계가 명백해졌다. HTTP/2(RFC 7540)는 이 문제들을 프로토콜 레벨에서 근본적으로 해결하기 위해 Google의 SPDY 프로토콜을 기반으로 2015년 표준화되었다.

## HTTP/1.1의 구조적 문제점

### 1. Head-of-Line (HOL) Blocking

HTTP/1.1은 하나의 TCP 연결에서 요청-응답이 순서대로(파이프라인) 처리된다. 앞의 요청이 지연되면 뒤의 모든 요청도 기다려야 한다. HTTP/1.1 파이프라이닝은 표준에 정의되어 있으나, 대부분의 서버와 프록시가 제대로 구현하지 않아 실질적으로 사용되지 않는다.

```
HTTP/1.1 연결:
[요청1] → [응답1] → [요청2] → [응답2] → [요청3] → [응답3]
          ^^^^^^^^ 응답1이 느리면 요청2는 대기
```

### 2. 헤더 중복과 비압축

HTTP/1.1 헤더는 평문 텍스트로 전송된다. 동일한 도메인에 100번의 요청을 보내면 `Cookie`, `User-Agent`, `Accept-Encoding`, `Authorization` 같은 수 KB짜리 헤더가 100번 반복된다. 실제로 헤더 크기가 응답 본문보다 큰 경우도 흔하다.

### 3. 도메인 샤딩의 한계

브라우저는 도메인당 6개의 TCP 연결만 허용하므로, 개발자들이 `static1.example.com`, `static2.example.com` 같은 여러 서브도메인으로 리소스를 분산하는 도메인 샤딩(domain sharding)을 사용했다. 이는 DNS 조회, TCP 핸드셰이크, TLS 협상 오버헤드를 증가시킨다.

## HTTP/2 핵심 개념: 연결, 스트림, 프레임

HTTP/2는 세 가지 계층적 개념을 도입한다.

- **연결(Connection)**: 하나의 TCP 연결. HTTP/2는 단일 TCP 연결로 모든 통신을 처리한다.
- **스트림(Stream)**: 연결 내의 독립적인 양방향 바이트 흐름. 여러 스트림이 동시에 인터리빙되어 실행된다.
- **프레임(Frame)**: HTTP/2 통신의 최소 단위. 모든 메시지는 하나 이상의 프레임으로 분할된다.

```
TCP 연결
├── 스트림 1 (요청/응답 쌍)
│   ├── HEADERS 프레임
│   └── DATA 프레임
├── 스트림 3 (동시 진행)
│   ├── HEADERS 프레임
│   └── DATA 프레임
└── 스트림 5 (동시 진행)
    └── HEADERS 프레임
```

클라이언트가 시작하는 스트림은 홀수 번호(1, 3, 5, ...), 서버 푸시는 짝수 번호(2, 4, 6, ...)를 사용한다.

## 바이너리 프레이밍 레이어

HTTP/1.1이 텍스트 기반이었다면, HTTP/2는 모든 통신을 **바이너리 프레임**으로 전달한다. 각 프레임은 9바이트 고정 헤더와 가변 길이 페이로드로 구성된다.

```
+-----------------------------------------------+
|                 Length (24 bits)               |
+---------------+---------------+---------------+
|   Type (8)    |   Flags (8)   |
+-+-------------+---------------+-------------------------------+
|R|                 Stream Identifier (31 bits)                  |
+=+=============================================================+
|                   Frame Payload (0...2^24-1 octets)           |
+---------------------------------------------------------------+
```

- **Length**: 페이로드 길이 (최대 2^24-1 바이트, 기본 최대값 16KB)
- **Type**: 프레임 유형 (0~9)
- **Flags**: 프레임 유형별 플래그 (예: END_STREAM, END_HEADERS)
- **R**: 예약 비트 (항상 0)
- **Stream ID**: 이 프레임이 속한 스트림 번호 (0이면 연결 레벨)

주요 프레임 타입:

| 타입 | 값 | 설명 |
|------|----|------|
| DATA | 0x0 | 실제 HTTP 본문 데이터 |
| HEADERS | 0x1 | HPACK 인코딩된 헤더 블록 |
| PRIORITY | 0x2 | 스트림 우선순위 설정 |
| RST_STREAM | 0x3 | 스트림 강제 종료 |
| SETTINGS | 0x4 | 연결 파라미터 협상 |
| PUSH_PROMISE | 0x5 | 서버 푸시 예고 |
| PING | 0x6 | 연결 상태 확인 |
| GOAWAY | 0x7 | 연결 종료 통지 |
| WINDOW_UPDATE | 0x8 | 흐름 제어 윈도우 갱신 |
| CONTINUATION | 0x9 | HEADERS 프레임 연속 |

## HPACK 헤더 압축 (RFC 7541)

HPACK은 HTTP/2에서 헤더를 압축하는 전용 알고리즘이다. SPDY에서 사용하던 DEFLATE 기반 압축이 CRIME 공격(Compression Ratio Info-leak Made Easy)에 취약하다는 것이 밝혀지면서, 보안을 최우선으로 설계된 새로운 방식이다.

### 1. 정적 테이블 (Static Table)

61개의 자주 쓰이는 헤더-값 쌍을 고정 인덱스로 제공한다. 단 1~2 바이트의 인덱스로 전체 헤더 항목을 표현할 수 있다.

| 인덱스 | 헤더 이름 | 값 |
|--------|------------|-----|
| 1 | `:authority` | |
| 2 | `:method` | `GET` |
| 3 | `:method` | `POST` |
| 4 | `:path` | `/` |
| 7 | `:scheme` | `https` |
| 8 | `:status` | `200` |
| 14 | `:status` | `404` |
| 37 | `if-modified-since` | |
| 46 | `content-type` | |

### 2. 동적 테이블 (Dynamic Table)

연결 중에 교환된 헤더를 FIFO 방식으로 캐싱한다. 클라이언트와 서버가 각각 독립적인 동적 테이블을 유지하며, 양측이 동일한 규칙으로 테이블을 업데이트하므로 별도 동기화 없이 같은 상태를 공유한다. 최대 테이블 크기는 SETTINGS_HEADER_TABLE_SIZE 파라미터로 협상한다 (기본값 4096 바이트).

```
연결 시작 시: 동적 테이블 비어 있음

첫 번째 요청 후 동적 테이블:
[62] :authority: api.example.com
[63] accept: application/json
[64] authorization: Bearer token123

두 번째 요청: 인덱스 62, 63, 64만으로 동일 헤더 참조 가능
```

### 3. 허프만 인코딩

문자열 리터럴을 HPACK 전용 허프만 코드로 압축한다. HTTP 헤더에서 통계적으로 자주 등장하는 ASCII 문자에 짧은 비트 코드를 배분한다. 예를 들어 `0` (숫자)은 `00000` (5비트)으로 표현되는 반면, 거의 쓰이지 않는 특수문자는 30비트 이상이 될 수 있다.

### 헤더 표현 형식

HPACK은 세 가지 헤더 표현 방식을 지원한다.

**1. 인덱스 참조 (Indexed Header Field)**: 첫 비트가 1, 나머지 7비트가 인덱스
```
1xxxxxxx
```

**2. 리터럴 + 인덱스 추가 (Literal with Incremental Indexing)**: 동적 테이블에 추가
```
01xxxxxx [이름] [값]
```

**3. Never Indexed**: 중간 노드(프록시)에서도 절대 압축하지 않음 — 패스워드 등 민감 데이터용
```
0001xxxx [이름] [값]
```

## 코드 예제 1: Python으로 HPACK 압축 효율 측정

```python
# pip install hpack
from hpack import Encoder, Decoder

def measure_hpack_efficiency():
    encoder = Encoder()

    # 첫 번째 요청
    headers1 = [
        (b':method', b'GET'),
        (b':path', b'/api/users'),
        (b':scheme', b'https'),
        (b':authority', b'api.example.com'),
        (b'user-agent', b'Mozilla/5.0 (compatible; MyApp/1.0)'),
        (b'accept', b'application/json'),
        (b'authorization', b'Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9'),
        (b'accept-encoding', b'gzip, deflate, br'),
    ]

    raw_size1 = sum(len(k) + len(v) + 32 for k, v in headers1)
    encoded1 = encoder.encode(headers1)
    print(f"=== 첫 번째 요청 ===")
    print(f"원본 헤더 크기: {raw_size1} bytes")
    print(f"HPACK 인코딩:   {len(encoded1)} bytes")
    print(f"압축률:         {100 * (1 - len(encoded1)/raw_size1):.1f}%")

    # 두 번째 요청 — 동적 테이블 활용
    headers2 = [
        (b':method', b'GET'),
        (b':path', b'/api/products'),   # path만 변경
        (b':scheme', b'https'),
        (b':authority', b'api.example.com'),   # 동적 테이블 히트
        (b'user-agent', b'Mozilla/5.0 (compatible; MyApp/1.0)'),  # 동적 테이블 히트
        (b'accept', b'application/json'),       # 동적 테이블 히트
        (b'authorization', b'Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9'),  # 히트
        (b'accept-encoding', b'gzip, deflate, br'),  # 히트
    ]

    raw_size2 = sum(len(k) + len(v) + 32 for k, v in headers2)
    encoded2 = encoder.encode(headers2)
    print(f"\n=== 두 번째 요청 (동적 테이블 활용) ===")
    print(f"원본 헤더 크기: {raw_size2} bytes")
    print(f"HPACK 인코딩:   {len(encoded2)} bytes")
    print(f"압축률:         {100 * (1 - len(encoded2)/raw_size2):.1f}%")

    # 디코더로 복원 확인
    decoder = Decoder()
    decoded = decoder.decode(encoded1)
    print(f"\n=== 디코딩 검증 ===")
    for name, value in decoded:
        print(f"  {name.decode()}: {value.decode()}")

if __name__ == '__main__':
    measure_hpack_efficiency()
```

실행 결과 예시:
```
=== 첫 번째 요청 ===
원본 헤더 크기: 312 bytes
HPACK 인코딩:   148 bytes
압축률:         52.6%

=== 두 번째 요청 (동적 테이블 활용) ===
원본 헤더 크기: 316 bytes
HPACK 인코딩:   23 bytes
압축률:         92.7%
```

두 번째 요청에서 동적 테이블 덕분에 압축률이 52%에서 92%로 크게 향상된다.

## 코드 예제 2: h2 라이브러리로 간단한 HTTP/2 클라이언트 구현

```python
# pip install h2 hyper
import socket
import ssl
import h2.connection
import h2.config
import h2.events

def send_http2_request(host, path='/'):
    """순수 h2 라이브러리로 HTTP/2 GET 요청 전송"""
    # TLS 컨텍스트 설정 — ALPN으로 h2 협상
    ctx = ssl.SSLContext(ssl.PROTOCOL_TLS_CLIENT)
    ctx.set_alpn_protocols(['h2'])
    ctx.load_default_certs()

    raw_sock = socket.create_connection((host, 443))
    sock = ctx.wrap_socket(raw_sock, server_hostname=host)

    # ALPN으로 HTTP/2 협상 확인
    negotiated = sock.selected_alpn_protocol()
    if negotiated != 'h2':
        raise RuntimeError(f"HTTP/2 협상 실패, 협상된 프로토콜: {negotiated}")
    print(f"프로토콜 협상: {negotiated}")

    # HTTP/2 연결 초기화
    config = h2.config.H2Configuration(client_side=True, header_encoding='utf-8')
    conn = h2.connection.H2Connection(config=config)
    conn.initiate_connection()
    sock.sendall(conn.data_to_send(65535))

    # HEADERS 프레임으로 GET 요청 전송 (스트림 1)
    headers = [
        (':method', 'GET'),
        (':path', path),
        (':scheme', 'https'),
        (':authority', host),
        ('user-agent', 'python-h2-demo/1.0'),
        ('accept', '*/*'),
    ]
    conn.send_headers(1, headers, end_stream=True)
    sock.sendall(conn.data_to_send(65535))

    # 응답 수신
    response_headers = {}
    response_body = b''
    stream_ended = False

    while not stream_ended:
        data = sock.recv(65535)
        if not data:
            break
        events = conn.receive_data(data)

        for event in events:
            if isinstance(event, h2.events.ResponseReceived):
                response_headers = dict(event.headers)
                print(f"응답 상태: {response_headers.get(':status')}")
                print(f"스트림 ID: {event.stream_id}")

            elif isinstance(event, h2.events.DataReceived):
                response_body += event.data
                # 흐름 제어: 수신한 데이터만큼 윈도우 업데이트
                conn.acknowledge_received_data(event.flow_controlled_length, event.stream_id)

            elif isinstance(event, h2.events.StreamEnded):
                stream_ended = True

            elif isinstance(event, h2.events.WindowUpdated):
                pass  # 서버가 우리 윈도우를 업데이트

        sock.sendall(conn.data_to_send(65535))

    conn.close_connection()
    sock.sendall(conn.data_to_send())
    sock.close()

    return response_headers, response_body

if __name__ == '__main__':
    headers, body = send_http2_request('httpbin.org', '/get')
    print(f"\n응답 헤더: {dict(headers)}")
    print(f"응답 본문 크기: {len(body)} bytes")
```

## 흐름 제어 (Flow Control)

HTTP/2는 TCP 흐름 제어와 별개로 **애플리케이션 레벨 흐름 제어**를 제공한다. 연결 레벨과 스트림 레벨, 두 단계로 작동한다.

각 스트림과 연결에 **수신 윈도우(receive window)**가 있다. 송신자는 윈도우 크기 이상의 DATA를 보낼 수 없다. 수신자가 데이터를 처리하면 WINDOW_UPDATE 프레임으로 윈도우를 보충한다. 기본 윈도우 크기는 65,535 바이트 (64KB - 1)다.

이 메커니즘은 느린 소비자(클라이언트)가 빠른 송신자(서버)에 의해 버퍼가 넘치는 것을 방지한다.

## 서버 푸시 (Server Push)

서버는 클라이언트가 요청하기 전에 PUSH_PROMISE 프레임으로 리소스를 미리 전송할 수 있다. `/index.html` 요청 시 서버가 `/style.css`와 `/app.js`를 즉시 푸시하면 클라이언트의 별도 왕복 지연(RTT)을 줄일 수 있다.

그러나 서버 푸시는 실제 효과가 제한적이다. 클라이언트가 이미 캐시에 해당 리소스를 갖고 있어도 서버는 알 수 없으므로 중복 전송이 발생한다. 이 한계로 인해 HTTP/3(RFC 9114)에서는 서버 푸시가 제거되었다.

## HTTP/2의 한계 — TCP HOL Blocking

HTTP/2가 애플리케이션 레벨의 HOL Blocking은 해결했지만, **TCP 레벨의 HOL Blocking**은 여전히 존재한다. TCP는 패킷을 순서대로 전달해야 하므로, 하나의 패킷이 손실되면 재전송될 때까지 같은 TCP 연결의 모든 스트림이 대기한다. 패킷 손실률이 2%일 때 HTTP/2가 HTTP/1.1보다 느려진다는 연구 결과도 있다.

이를 근본적으로 해결하기 위해 UDP 기반의 QUIC 프로토콜(HTTP/3)이 설계되었다. QUIC은 스트림별로 독립적인 신뢰성을 보장하므로 한 스트림의 패킷 손실이 다른 스트림에 영향을 주지 않는다.

## 주의사항 및 실전 팁

**HTTP/2는 실제로 TLS 없이도 동작하지만(h2c, cleartext), 모든 주요 브라우저가 TLS 위에서만 HTTP/2를 지원한다.** ALPN(Application Layer Protocol Negotiation) TLS 확장을 통해 TLS 핸드셰이크 중 `h2`를 협상한다.

**동적 테이블 크기 조정:** 서버가 SETTINGS_HEADER_TABLE_SIZE를 너무 크게 설정하면 연결 수만큼 메모리가 증가한다. 1만 개의 동시 연결이 있을 때 테이블 크기 1MB는 10GB 메모리를 의미한다.

**스트림 우선순위(Priority)는 현실에서 제대로 구현되지 않는다.** RFC 7540의 우선순위 트리는 복잡하고 구현이 어려워 대부분의 서버가 지원하지 않는다. RFC 9218에서 새로운 우선순위 스킴인 Extensible Priorities가 정의되었다.

**HTTP/2 업그레이드 협상:** 클라이언트가 HTTP/1.1 요청에 `Upgrade: h2c` 헤더를 포함해 HTTP/2로 업그레이드를 요청할 수 있다. 서버는 `101 Switching Protocols`로 응답한다. 그러나 TLS 기반 연결에서는 ALPN이 더 효율적이다.

## 참고 자료
- [RFC 7540 - HTTP/2 (IETF Datatracker)](https://datatracker.ietf.org/doc/html/rfc7540)
- [RFC 7541 - HPACK: Header Compression for HTTP/2](https://httpwg.org/specs/rfc7541.html)
- [HTTP/2 - High Performance Browser Networking (Ilya Grigorik)](https://hpbn.co/http2/)
- [RFC 9114 - HTTP/3](https://datatracker.ietf.org/doc/html/rfc9114)
