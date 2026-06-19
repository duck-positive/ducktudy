---
layout: post
title: "TLS 1.3 핸드셰이크 심화: 공개키 암호화부터 인증서 체인까지"
date: 2026-06-19
categories: [cs, computer-science]
tags: [tls, ssl, handshake, pki, certificate, x509, ecdhe, network-security, https]
---

## TLS란 무엇이고 왜 중요한가?

**TLS(Transport Layer Security)**는 인터넷에서 두 엔드포인트 사이의 통신을 **기밀성(Confidentiality)**, **무결성(Integrity)**, **인증(Authentication)**이라는 세 가지 목표로 보호하는 암호화 프로토콜입니다.

우리가 매일 사용하는 HTTPS는 HTTP + TLS이며, 뱅킹, 로그인, API 호출 등 민감한 데이터가 오가는 모든 곳에서 동작합니다. TLS를 이해하면 다음을 설계할 수 있습니다.

- 중간자 공격(MITM)을 방어하는 인증서 검증 로직
- Perfect Forward Secrecy(PFS)를 보장하는 키 교환 방식 선택
- mTLS(상호 TLS)를 활용한 서비스 간 인증

이 글에서는 TLS 1.3의 핸드셰이크 과정을 단계별로 분해하고, X.509 인증서 체인 검증 원리, 그리고 Python으로 TLS 서버/클라이언트를 직접 구현해봅니다.

---

## TLS 1.2 vs TLS 1.3: 무엇이 달라졌나?

| 항목 | TLS 1.2 | TLS 1.3 |
|---|---|---|
| 핸드셰이크 RTT | 2 RTT | 1 RTT (0-RTT 재연결 지원) |
| 키 교환 | RSA, DHE, ECDHE | ECDHE만 허용 (RSA 키 교환 제거) |
| 암호 스위트 수 | 수백 개 | 5개로 축소 |
| Forward Secrecy | 선택적 | 필수 |
| 핸드셰이크 암호화 | 일부 평문 | ServerHello 이후 전체 암호화 |

TLS 1.3은 불필요한 옵션을 제거하고 보안을 강화하여 구현 오류 표면을 크게 줄였습니다.

---

## TLS 1.3 핸드셰이크 단계별 분해

### 1단계: ClientHello

클라이언트가 서버에 첫 메시지를 보냅니다.

```
ClientHello:
  - ProtocolVersion: TLS 1.3
  - Random: 32바이트 난수
  - CipherSuites: [TLS_AES_128_GCM_SHA256, TLS_AES_256_GCM_SHA384, ...]
  - Extensions:
    - supported_groups: [x25519, secp256r1, ...]   # ECDHE 그룹
    - key_share: 클라이언트의 ECDHE 공개키 (미리 전송!)
    - server_name: "example.com"                   # SNI
    - signature_algorithms: [ecdsa_secp256r1_sha256, ...]
```

**TLS 1.3의 핵심**: 클라이언트는 ClientHello에 **ECDHE 공개키를 미리 포함**하여 1-RTT 핸드셰이크를 가능하게 합니다.

### 2단계: ServerHello + 키 도출

서버는 한 번의 응답으로 다음을 전송합니다:

```
ServerHello:
  - 선택된 CipherSuite: TLS_AES_256_GCM_SHA384
  - 서버의 ECDHE 공개키

EncryptedExtensions (이미 암호화됨):
  - ALPN, max_fragment_length 등

Certificate (암호화됨):
  - 서버 인증서 체인 (leaf → intermediate → root)

CertificateVerify (암호화됨):
  - 핸드셰이크 전체에 대한 서버 서명

Finished (암호화됨):
  - 핸드셰이크 MAC
```

서버는 ClientHello에 포함된 클라이언트 ECDHE 공개키를 받자마자 **공유 비밀(Shared Secret)**을 계산할 수 있습니다.

### 3단계: ECDHE 키 교환과 세션 키 도출

ECDHE(Elliptic Curve Diffie-Hellman Ephemeral) 과정:

```
클라이언트: 임시 개인키 a 생성 → 공개키 A = a * G 전송
서버:       임시 개인키 b 생성 → 공개키 B = b * G 전송

공유 비밀 = a * B = b * A = a * b * G  ← 수학적으로 동일
```

이 공유 비밀을 HKDF(HMAC-based Key Derivation Function)로 확장해 실제 암호화 키를 도출합니다.

**Ephemeral(임시)**: 매 세션마다 새로운 개인키를 생성하므로, 서버의 장기 개인키가 유출되더라도 과거 세션을 복호화할 수 없습니다 → **Perfect Forward Secrecy**.

---

## X.509 인증서와 인증서 체인

### 인증서 구조

X.509 인증서는 다음 핵심 필드를 포함합니다:

```
Certificate:
  Version: 3
  SerialNumber: 123456789
  Issuer: CN=DigiCert Global CA G2, O=DigiCert Inc, C=US
  Subject: CN=*.example.com, O=Example Inc, C=US
  Validity:
    Not Before: 2025-01-01
    Not After:  2026-01-01
  SubjectPublicKeyInfo:
    Algorithm: id-ecPublicKey (P-256)
    PublicKey: 04:xx:xx:xx...
  Extensions:
    SubjectAlternativeNames: [example.com, www.example.com]
    KeyUsage: digitalSignature
    BasicConstraints: CA:FALSE
  Signature:
    Algorithm: ecdsa-with-SHA256
    Value: 30:46:02:21:...  ← Issuer의 개인키로 서명
```

### 인증서 체인 검증 과정

```
[Root CA 인증서]          ← 브라우저/OS에 내장된 신뢰 앵커
      │ 서명
[Intermediate CA 인증서]  ← 서버가 핸드셰이크 때 전송
      │ 서명
[서버 인증서 (Leaf)]      ← example.com 인증서
```

클라이언트는 다음 순서로 검증합니다:

1. **체인 구성**: Leaf → Intermediate → Root를 연결
2. **서명 검증**: 각 인증서의 서명을 상위 CA 공개키로 검증
3. **유효 기간 확인**: 현재 시간이 Not Before ~ Not After 범위 내인지
4. **SANs 확인**: 접속하려는 호스트명이 인증서의 Subject Alternative Names에 포함되는지
5. **폐기 상태 확인**: OCSP(Online Certificate Status Protocol) 또는 CRL로 인증서 폐기 여부 확인

---

## 실제 구현 예제

### 예제 1: Python으로 TLS 서버와 클라이언트 구현

```python
# tls_server.py
import ssl
import socket
import threading


def create_tls_server(host: str = "localhost", port: int = 8443,
                      certfile: str = "server.crt", keyfile: str = "server.key"):
    """
    자체 서명 인증서를 사용하는 TLS 1.3 서버 (테스트용)
    사전 준비: openssl req -x509 -newkey ec -pkeyopt ec_paramgen_curve:P-256
               -keyout server.key -out server.crt -days 365 -nodes -subj "/CN=localhost"
    """
    # TLS 컨텍스트 설정
    context = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
    context.minimum_version = ssl.TLSVersion.TLSv1_3  # TLS 1.3 강제
    context.load_cert_chain(certfile=certfile, keyfile=keyfile)

    # 허용할 암호 스위트 (TLS 1.3은 자동 선택)
    # context.set_ciphers("TLS_AES_256_GCM_SHA384")

    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as sock:
        sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        sock.bind((host, port))
        sock.listen(5)
        print(f"[Server] TLS 1.3 서버 시작: {host}:{port}")

        with context.wrap_socket(sock, server_side=True) as tls_sock:
            while True:
                conn, addr = tls_sock.accept()
                threading.Thread(target=handle_client, args=(conn, addr)).start()


def handle_client(conn: ssl.SSLSocket, addr):
    with conn:
        # 핸드셰이크 완료 후 세션 정보 출력
        print(f"\n[Server] 클라이언트 연결: {addr}")
        print(f"[Server] TLS 버전: {conn.version()}")
        print(f"[Server] 암호 스위트: {conn.cipher()}")
        print(f"[Server] 서버 인증서 Subject: {conn.getpeercert()}")

        data = conn.recv(1024)
        print(f"[Server] 수신: {data.decode()}")
        conn.sendall(b"HTTP/1.1 200 OK\r\n\r\nHello, TLS 1.3!")


# tls_client.py
def create_tls_client(host: str = "localhost", port: int = 8443,
                      cafile: str = "server.crt"):
    """자체 서명 인증서를 신뢰 앵커로 사용하는 TLS 클라이언트"""
    context = ssl.SSLContext(ssl.PROTOCOL_TLS_CLIENT)
    context.minimum_version = ssl.TLSVersion.TLSv1_3
    context.load_verify_locations(cafile=cafile)  # Root CA 등록
    # context.check_hostname = False  # 테스트용으로만 비활성화

    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as sock:
        with context.wrap_socket(sock, server_hostname=host) as tls_sock:
            tls_sock.connect((host, port))

            print(f"\n[Client] 연결 성공!")
            print(f"[Client] TLS 버전: {tls_sock.version()}")
            print(f"[Client] 암호 스위트: {tls_sock.cipher()}")

            # 서버 인증서 검증 결과 출력
            cert = tls_sock.getpeercert()
            print(f"[Client] 서버 인증서 Subject: {cert.get('subject')}")
            print(f"[Client] 유효 기간: {cert.get('notBefore')} ~ {cert.get('notAfter')}")

            tls_sock.sendall(b"GET / HTTP/1.1\r\nHost: localhost\r\n\r\n")
            response = tls_sock.recv(4096)
            print(f"[Client] 응답: {response.decode()}")


if __name__ == "__main__":
    import sys
    if sys.argv[1] == "server":
        create_tls_server()
    else:
        create_tls_client()
```

---

### 예제 2: OpenSSL 커맨드로 인증서 체인 분석

```bash
# 1. 서버 인증서 체인 전체 다운로드 및 확인
echo | openssl s_client -connect example.com:443 -showcerts 2>/dev/null \
  | openssl x509 -noout -text | grep -A5 "Subject:\|Issuer:\|Validity"

# 2. 인증서의 상세 필드 확인 (Subject, SANs, 서명 알고리즘 등)
openssl s_client -connect example.com:443 2>/dev/null \
  | openssl x509 -noout -text

# 3. 인증서 체인 전체를 파일로 저장
echo | openssl s_client -connect example.com:443 -showcerts 2>/dev/null \
  | awk '/-----BEGIN CERTIFICATE-----/,/-----END CERTIFICATE-----/' \
  > chain.pem

# 4. 저장된 체인 검증 (Root CA 기준)
openssl verify -CAfile /etc/ssl/certs/ca-certificates.crt chain.pem

# 5. OCSP 응답으로 인증서 폐기 여부 확인
OCSP_URL=$(openssl x509 -in chain.pem -noout -ocsp_uri)
openssl ocsp -issuer chain.pem -cert chain.pem -url "$OCSP_URL" -resp_text

# 6. TLS 1.3 핸드셰이크 과정 전체 출력 (디버깅용)
openssl s_client -connect example.com:443 \
  -tls1_3 \
  -msg \          # 핸드셰이크 메시지 출력
  -debug \        # 원시 바이트 출력
  -state          # 상태 전환 출력

# 7. 자체 서명 테스트 인증서 생성 (EC P-256)
openssl req -x509 \
  -newkey ec \
  -pkeyopt ec_paramgen_curve:P-256 \
  -keyout server.key \
  -out server.crt \
  -days 365 \
  -nodes \
  -subj "/CN=localhost" \
  -addext "subjectAltName=DNS:localhost,IP:127.0.0.1"

echo "인증서 생성 완료"
openssl x509 -in server.crt -noout -text | grep -A3 "Subject Alternative Name"
```

---

## 주의사항과 팁

### 1. 인증서 만료 모니터링

TLS의 가장 흔한 장애 원인은 **인증서 만료**입니다. Let's Encrypt 인증서는 90일이 기본이며, 자동 갱신(certbot renew)이 실패하면 서비스 전체가 중단됩니다.

```bash
# 만료까지 남은 일수 확인
openssl s_client -connect your-domain.com:443 2>/dev/null \
  | openssl x509 -noout -enddate \
  | awk -F= '{print $2}' \
  | xargs -I{} date -d "{}" +%s \
  | xargs -I{} bash -c 'echo "남은 일수: $(( ({} - $(date +%s)) / 86400 ))일"'
```

### 2. SNI(Server Name Indication) 필수

하나의 IP에 여러 도메인을 호스팅(가상 호스팅)하는 경우, 서버는 ClientHello의 SNI 확장을 보고 어떤 인증서를 제공할지 결정합니다. SNI 없이 연결하면 기본 인증서가 제공되어 인증 실패가 발생할 수 있습니다.

### 3. 0-RTT는 Replay Attack에 취약

TLS 1.3의 0-RTT 재연결은 빠르지만, **재전송 공격(Replay Attack)**에 취약합니다. 0-RTT 데이터는 幂等(idempotent)한 요청(GET 등)에만 사용하고, 상태를 변경하는 요청(POST, 결제 등)에는 절대 사용하지 마세요.

### 4. 인증서 투명성(Certificate Transparency, CT)

최신 브라우저는 인증서가 공개된 CT 로그에 등록되어 있어야 신뢰합니다. Let's Encrypt는 자동으로 CT 로그에 제출하지만, 사설 CA로 발급한 인증서는 별도 제출이 필요합니다.

### 5. mTLS로 서비스 간 인증 강화

일반 TLS는 서버만 인증하지만, **mTLS(Mutual TLS)**는 클라이언트도 인증서를 제공하여 **양방향 인증**을 수행합니다. 마이크로서비스, gRPC, Kubernetes에서 서비스 간 통신 보안에 널리 활용됩니다.

---

## 참고 자료

- [The TLS Handshake Deep Dive — Medium](https://hosseinnejati.medium.com/the-tls-handshake-deep-dive-what-happens-before-a-single-byte-of-data-flows-ad8b7546b4dd)
- [TLS and mTLS Deep Dive — schartz.github.io](https://schartz.github.io/blog/tls-and-mtls-deep-dive/)
- [A Deep Dive into SSL/TLS — NashTech Blog](https://blog.nashtechglobal.com/a-deep-dive-into-ssl-tls-encryption-and-handshake-process/)
- [Part 17: TLS/SSL Deep Dive — PKI & Certificates](https://www.wasilzafar.com/pages/series/computing-systems-foundations/computing-systems-foundations-part17-tls-pki.html)
