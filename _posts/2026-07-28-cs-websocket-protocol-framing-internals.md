---
layout: post
title: "WebSocket 프로토콜 내부 구조 완전 정복: RFC 6455 핸드셰이크·프레이밍·마스킹의 모든 것"
date: 2026-07-28
categories: [cs, computer-science]
tags: [websocket, rfc6455, network, protocol, real-time, tcp, http-upgrade]
---

WebSocket은 단순한 "양방향 통신 기술"이 아닙니다. HTTP 업그레이드 메커니즘 위에서 동작하는 정밀하게 설계된 바이너리 프로토콜이며, 그 내부에는 프레이밍, 마스킹, 확장(Extension) 협상 등 수많은 설계 결정이 담겨 있습니다. 이 글에서는 RFC 6455를 기반으로 WebSocket의 작동 원리를 바이트 수준까지 파헤칩니다.

## 왜 WebSocket이 필요한가

HTTP는 기본적으로 단방향(클라이언트 → 서버) 요청-응답 모델입니다. 실시간 채팅, 주식 시세, 게임 등 서버가 클라이언트에게 먼저 데이터를 보내야 하는 시나리오에서 HTTP는 다음과 같은 우회책을 사용해왔습니다.

- **폴링(Polling)**: 클라이언트가 주기적으로 새 데이터를 요청 → 불필요한 요청 폭탄
- **롱 폴링(Long Polling)**: 서버가 응답을 지연시켜 데이터가 생기면 즉시 응답 → TCP 연결 낭비
- **SSE(Server-Sent Events)**: 서버→클라이언트 단방향 스트림 → 클라이언트→서버 불가

WebSocket은 이 모든 한계를 해결합니다. 단 하나의 TCP 연결 위에서 완전한 양방향(full-duplex) 통신을 실현하며, 오버헤드가 극도로 낮습니다. HTTP 요청이 수백 바이트의 헤더를 매 요청마다 전송하는 반면, WebSocket 프레임 헤더는 최소 **2바이트**입니다.

## RFC 6455 핵심 개념

RFC 6455는 2011년 IETF가 표준화한 WebSocket 프로토콜 명세입니다. 크게 세 부분으로 구성됩니다.

### 1. 오프닝 핸드셰이크 (Opening Handshake)

WebSocket 연결은 반드시 HTTP 업그레이드 요청으로 시작합니다. 클라이언트는 다음 요청을 보냅니다.

```
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

`Sec-WebSocket-Key`는 클라이언트가 생성한 16바이트 랜덤값을 Base64로 인코딩한 것입니다. 서버는 이 키에 매직 스트링 `258EAFA5-E914-47DA-95CA-C5AB0DC85B11`을 붙여 SHA-1 해시 후 Base64로 인코딩해 응답합니다.

```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

101 Switching Protocols를 수신하면 TCP 연결은 WebSocket 프레임 전용으로 전환됩니다. HTTP는 더 이상 사용되지 않습니다.

### 2. 데이터 프레이밍 (Data Framing)

WebSocket의 핵심은 프레임 구조입니다. 각 프레임은 다음 비트 레이아웃을 가집니다.

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
|N|V|V|V|       |S|             |   (if payload len==126/127)   |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - -+
|     Extended payload length continued, if payload len == 127  |
+ - - - - - - - - - - - - - - -+-------------------------------+
|                               |Masking-key, if MASK set to 1  |
+-------------------------------+-------------------------------+
| Masking-key (continued)       |          Payload Data         |
+-------------------------------- - - - - - - - - - - - - - - -+
:                     Payload Data continued ...                :
+---------------------------------------------------------------+
```

주요 필드를 해설하겠습니다.

- **FIN(1bit)**: 메시지의 마지막 프레임임을 나타냄. 큰 메시지는 여러 프레임으로 분할 전송 가능
- **RSV1~3(3bit)**: 확장용 예약 비트. 기본적으로 0. WebSocket 확장(예: permessage-deflate 압축)이 이 비트를 활용
- **Opcode(4bit)**: 프레임 종류 지정
  - `0x0`: Continuation frame
  - `0x1`: Text frame (UTF-8)
  - `0x2`: Binary frame
  - `0x8`: Connection close
  - `0x9`: Ping
  - `0xA`: Pong
- **MASK(1bit)**: 클라이언트→서버 방향은 반드시 1이어야 함
- **Payload len(7bit)**: 125 이하면 직접 길이. 126이면 다음 2바이트가 16-bit 길이. 127이면 다음 8바이트가 64-bit 길이

### 3. 마스킹 (Masking)

WebSocket의 독특한 특징 중 하나가 마스킹입니다. **클라이언트에서 서버로 보내는 모든 프레임은 반드시 마스킹**되어야 합니다. 서버에서 클라이언트로 가는 프레임은 마스킹하지 않습니다.

마스킹 알고리즘은 간단합니다. 4바이트 마스킹 키를 생성한 뒤, 페이로드의 각 바이트에 XOR 연산을 반복 적용합니다.

```
masked_byte[i] = original_byte[i] XOR masking_key[i % 4]
```

마스킹의 목적은 **캐시 포이즈닝 공격 방지**입니다. 중간 프록시 서버가 WebSocket 트래픽을 HTTP 응답으로 오인해 캐시에 저장하는 공격을 막기 위해, 임의의 마스킹 키를 사용해 페이로드가 예측 불가능하게 보이도록 합니다.

## 실제 구현 예제

### 예제 1: Python으로 WebSocket 핸드셰이크와 프레임 파싱 직접 구현

```python
import socket
import hashlib
import base64
import struct

MAGIC = "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"

def compute_accept_key(client_key: str) -> str:
    combined = client_key + MAGIC
    sha1 = hashlib.sha1(combined.encode()).digest()
    return base64.b64encode(sha1).decode()

def parse_handshake(raw: bytes) -> dict:
    headers = {}
    lines = raw.decode().split("\r\n")
    for line in lines[1:]:
        if ": " in line:
            k, v = line.split(": ", 1)
            headers[k] = v
    return headers

def send_handshake_response(conn: socket.socket, client_key: str):
    accept = compute_accept_key(client_key)
    response = (
        "HTTP/1.1 101 Switching Protocols\r\n"
        "Upgrade: websocket\r\n"
        "Connection: Upgrade\r\n"
        f"Sec-WebSocket-Accept: {accept}\r\n"
        "\r\n"
    )
    conn.sendall(response.encode())

def decode_frame(data: bytes) -> dict:
    if len(data) < 2:
        return None

    byte0 = data[0]
    byte1 = data[1]

    fin  = (byte0 >> 7) & 1
    rsv1 = (byte0 >> 6) & 1
    opcode = byte0 & 0x0F
    masked = (byte1 >> 7) & 1
    payload_len = byte1 & 0x7F

    offset = 2
    if payload_len == 126:
        payload_len = struct.unpack(">H", data[offset:offset+2])[0]
        offset += 2
    elif payload_len == 127:
        payload_len = struct.unpack(">Q", data[offset:offset+8])[0]
        offset += 8

    mask_key = None
    if masked:
        mask_key = data[offset:offset+4]
        offset += 4

    payload = bytearray(data[offset:offset+payload_len])
    if masked and mask_key:
        for i in range(len(payload)):
            payload[i] ^= mask_key[i % 4]

    opcodes = {0x1: "text", 0x2: "binary", 0x8: "close", 0x9: "ping", 0xA: "pong"}
    return {
        "fin": fin,
        "opcode": opcodes.get(opcode, f"0x{opcode:x}"),
        "masked": bool(masked),
        "payload": bytes(payload),
    }

def encode_frame(payload: bytes, opcode: int = 0x1) -> bytes:
    length = len(payload)
    header = bytearray()
    header.append(0x80 | opcode)  # FIN=1

    if length <= 125:
        header.append(length)
    elif length <= 65535:
        header.append(126)
        header.extend(struct.pack(">H", length))
    else:
        header.append(127)
        header.extend(struct.pack(">Q", length))

    return bytes(header) + payload

# 서버 실행 예시
def run_server():
    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server.bind(("0.0.0.0", 8765))
    server.listen(1)
    print("WebSocket 서버 대기 중: ws://localhost:8765")

    conn, addr = server.accept()
    raw = conn.recv(4096)
    headers = parse_handshake(raw)
    send_handshake_response(conn, headers["Sec-WebSocket-Key"])
    print(f"클라이언트 연결됨: {addr}")

    while True:
        data = conn.recv(4096)
        if not data:
            break
        frame = decode_frame(data)
        if frame["opcode"] == "close":
            break
        print(f"수신: {frame['payload'].decode()}")
        conn.sendall(encode_frame(b"Echo: " + frame["payload"]))

    conn.close()

if __name__ == "__main__":
    run_server()
```

이 구현은 실제 WebSocket 라이브러리 없이 RFC 6455를 직접 따른 것입니다. `decode_frame` 함수의 마스킹 해제 로직과 가변 길이 페이로드 처리가 핵심입니다.

### 예제 2: JavaScript WebSocket 클라이언트와 Ping/Pong 하트비트

```javascript
// Node.js 환경에서 WebSocket 저수준 동작 시뮬레이션
const net = require('net');
const crypto = require('crypto');

function generateKey() {
  return crypto.randomBytes(16).toString('base64');
}

function computeAccept(key) {
  const MAGIC = '258EAFA5-E914-47DA-95CA-C5AB0DC85B11';
  return crypto.createHash('sha1').update(key + MAGIC).digest('base64');
}

function buildHandshake(host, path, key) {
  return [
    `GET ${path} HTTP/1.1`,
    `Host: ${host}`,
    `Upgrade: websocket`,
    `Connection: Upgrade`,
    `Sec-WebSocket-Key: ${key}`,
    `Sec-WebSocket-Version: 13`,
    '',
    ''
  ].join('\r\n');
}

function maskPayload(payload, maskKey) {
  const result = Buffer.alloc(payload.length);
  for (let i = 0; i < payload.length; i++) {
    result[i] = payload[i] ^ maskKey[i % 4];
  }
  return result;
}

function buildFrame(payload, opcode = 0x1) {
  const maskKey = crypto.randomBytes(4);
  const masked = maskPayload(Buffer.from(payload), maskKey);
  const len = masked.length;

  let header;
  if (len <= 125) {
    header = Buffer.from([0x80 | opcode, 0x80 | len]);
  } else if (len <= 65535) {
    header = Buffer.alloc(4);
    header[0] = 0x80 | opcode;
    header[1] = 0x80 | 126;
    header.writeUInt16BE(len, 2);
  } else {
    header = Buffer.alloc(10);
    header[0] = 0x80 | opcode;
    header[1] = 0x80 | 127;
    header.writeBigUInt64BE(BigInt(len), 2);
  }

  return Buffer.concat([header, maskKey, masked]);
}

function parseFrame(buf) {
  const fin = (buf[0] >> 7) & 1;
  const opcode = buf[0] & 0x0F;
  const masked = (buf[1] >> 7) & 1;
  let payloadLen = buf[1] & 0x7F;
  let offset = 2;

  if (payloadLen === 126) {
    payloadLen = buf.readUInt16BE(2);
    offset = 4;
  } else if (payloadLen === 127) {
    payloadLen = Number(buf.readBigUInt64BE(2));
    offset = 10;
  }

  let payload = buf.slice(offset, offset + payloadLen);
  if (masked) {
    const key = buf.slice(offset, offset + 4);
    payload = buf.slice(offset + 4, offset + 4 + payloadLen);
    for (let i = 0; i < payload.length; i++) payload[i] ^= key[i % 4];
    offset += 4;
  }

  return { fin, opcode, payload };
}

// 연결 데모
const key = generateKey();
const socket = net.createConnection(8765, 'localhost', () => {
  socket.write(buildHandshake('localhost', '/', key));
});

socket.on('data', (data) => {
  const text = data.toString();
  if (text.includes('101 Switching Protocols')) {
    console.log('핸드셰이크 성공! 메시지 전송...');
    socket.write(buildFrame('안녕하세요, WebSocket!'));

    // 30초마다 Ping 전송 (opcode=0x9)
    setInterval(() => {
      console.log('Ping 전송...');
      socket.write(buildFrame('ping', 0x9));
    }, 30000);
  } else {
    const frame = parseFrame(data);
    const opcodeMap = { 1: 'text', 2: 'binary', 8: 'close', 9: 'ping', 10: 'pong' };
    console.log(`수신 [${opcodeMap[frame.opcode]}]: ${frame.payload.toString()}`);
  }
});
```

## 제어 프레임과 연결 종료

WebSocket에는 데이터 전송 외에도 세 가지 제어 프레임(Control Frame)이 있습니다.

**Close (0x8)**: 양측이 연결 종료에 합의하는 4단계 종료 절차를 거칩니다. Close 프레임을 받은 쪽은 반드시 Close 프레임으로 응답해야 합니다. 페이로드의 첫 2바이트는 상태 코드(1000=정상, 1001=클라이언트 이탈, 1011=서버 오류 등)입니다.

**Ping (0x9) / Pong (0xA)**: 연결 생존 여부를 확인하는 하트비트 메커니즘입니다. Ping을 받은 쪽은 동일한 페이로드로 Pong을 반환해야 합니다.

## 메시지 분할 (Fragmentation)

큰 메시지는 여러 프레임으로 나눠 전송할 수 있습니다.

- 첫 번째 프레임: `FIN=0`, `opcode=text/binary`
- 중간 프레임: `FIN=0`, `opcode=0x0 (continuation)`
- 마지막 프레임: `FIN=1`, `opcode=0x0 (continuation)`

이를 통해 서버는 전체 메시지가 완성되기 전에 부분 데이터를 스트리밍 방식으로 처리할 수 있습니다.

## 주의사항과 팁

**1. 마스킹 키는 매 프레임마다 랜덤하게 생성해야 합니다.** 예측 가능한 마스킹 키는 캐시 포이즈닝 방어 효과가 없습니다.

**2. permessage-deflate 확장 활용**: 대용량 텍스트 데이터(JSON 등)를 주고받을 때 압축 확장을 협상하면 페이로드 크기를 70~90% 줄일 수 있습니다. `Sec-WebSocket-Extensions: permessage-deflate` 헤더로 협상합니다.

**3. 메시지 크기 제한**: 페이로드 길이 필드가 64-bit이지만, 실제로는 메모리 사용량을 고려해 최대 메시지 크기를 반드시 제한해야 합니다. Nginx의 기본값은 64KB입니다.

**4. 프록시와의 호환성**: 일부 HTTP 프록시는 WebSocket 업그레이드를 지원하지 않아 연결이 끊어집니다. 이럴 때는 `wss://`(TLS 위의 WebSocket)를 사용하면 프록시가 암호화된 TCP 터널로 인식해 통과시킵니다.

**5. 자동 재연결 전략**: 네트워크 장애나 서버 재시작 시 클라이언트는 지수 백오프(Exponential Backoff)로 재연결을 시도해야 합니다. 모든 클라이언트가 동시에 재연결 시도하는 Thundering Herd 문제를 방지하기 위해 랜덤 지터(Jitter)를 추가하세요.

## 참고 자료
- [RFC 6455: The WebSocket Protocol](https://www.rfc-editor.org/rfc/rfc6455.html)
- [WebSocket Protocol Guide — websocket.org](https://websocket.org/guides/websocket-protocol/)
- [MDN Web Docs — Writing WebSocket servers](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API/Writing_WebSocket_servers)
- [IETF RFC 6455 원문](https://www.ietf.org/rfc/rfc6455)
