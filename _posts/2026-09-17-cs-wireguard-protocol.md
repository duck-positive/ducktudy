---
layout: post
title: "WireGuard 프로토콜 완전 정복: 현대 VPN의 암호화 설계와 Linux 커널 통합 원리"
date: 2026-09-17
categories: [cs, computer-science]
tags: [wireguard, vpn, cryptography, networking, linux-kernel, noise-protocol, 네트워크보안]
---

## 개요

WireGuard는 2016년 Jason A. Donenfeld가 발표한 현대적 VPN 프로토콜이다. 기존 VPN 솔루션인 OpenVPN(약 10만 줄의 코드)이나 IPsec(수십 만 줄)과 달리, WireGuard의 Linux 커널 구현체는 약 **4,000줄**에 불과하다. 코드 수가 적다는 것은 단순히 편리함의 문제가 아니다 — 보안 코드는 적을수록 감사(audit)하기 쉽고 취약점이 줄어든다. 리누스 토르발스는 WireGuard를 "예술 작품"이라 평가하며 Linux 5.6(2020)에 메인라인으로 병합했다.

이 아티클에서는 WireGuard의 암호화 설계 원리, Noise 프로토콜 프레임워크, 그리고 Linux 커널 모듈로서의 통합 방식을 깊이 파고든다.

## 왜 WireGuard인가: 기존 VPN의 문제점

### OpenVPN의 복잡성

OpenVPN은 TLS 위에서 동작하며, SSL/TLS 협상 과정에서 수백 가지의 암호 스위트 협상이 가능하다. 이 유연성은 곧 복잡성이자 공격 면적(attack surface)이다. 실제로 OpenVPN의 `--tls-cipher` 옵션에는 취약한 암호 스위트도 포함될 수 있으며, 잘못된 설정이 보안 취약점으로 이어지는 사례가 많다.

### IPsec의 협상 오버헤드

IPsec는 IKEv2 협상 단계에서 수십 개의 메시지가 오가며, 정책 협상(SPD/SAD)이 복잡하다. 또한 ESP/AH 헤더, NAT 통과(NAT-T), 재키잉(rekeying) 등의 구현이 각 벤더마다 달라 상호 운용성 문제가 빈번하다.

### WireGuard의 접근법: 고정 암호 스위트 + 단순 설계

WireGuard는 협상(negotiation)을 완전히 제거했다. **하나의 암호 스위트만 사용한다:**

| 역할 | 알고리즘 |
|------|---------|
| 키 교환 | Curve25519 (ECDH) |
| 대칭 암호화 | ChaCha20 |
| 인증 | Poly1305 |
| 해시 | BLAKE2s |
| HKDF | HMAC-SHA256 |

이 선택은 현재까지 알려진 최강의 현대적 암호 원시를 고정적으로 사용한다. 만약 향후 ChaCha20이 취약해진다면, 프로토콜 전체를 새 버전으로 교체하는 방식을 취한다.

## Noise 프로토콜 프레임워크

WireGuard 핸드셰이크는 **Noise Protocol Framework**를 기반으로 한다. Noise는 암호화 통신 프로토콜을 정형화된 방법으로 구성하는 프레임워크로, 다양한 핸드셰이크 패턴을 정의한다.

### WireGuard의 핸드셰이크: Noise_IKpsk2

WireGuard는 Noise 프레임워크의 `IKpsk2` 패턴을 사용한다:
- **I**: Initiator의 정적 공개키를 즉시 전송 (암호화됨)
- **K**: Responder의 정적 공개키가 Initiator에게 이미 알려져 있음
- **psk2**: 두 번째 메시지에 사전 공유 키(Pre-Shared Key)를 선택적으로 통합

핸드셰이크는 단 **2개의 메시지**로 완료된다:

```
Initiator                        Responder
─────────────────────────────────────────────
msg1 = (type=1, sender_index,
        unencrypted_ephemeral,
        encrypted_static,
        encrypted_timestamp)  ────────────>

                              <──────────
                              msg2 = (type=2, sender_index, receiver_index,
                                      unencrypted_ephemeral,
                                      encrypted_nothing)

데이터 패킷 교환 시작
```

### 핸드셰이크 암호 상태 머신

```python
# Noise IKpsk2 핸드셰이크의 의사 코드 구현
import hashlib
import hmac
from cryptography.hazmat.primitives.asymmetric.x25519 import X25519PrivateKey
from cryptography.hazmat.primitives.ciphers.aead import ChaCha20Poly1305

CONSTRUCTION = b"Noise_IKpsk2_25519_ChaChaPoly_BLAKE2s"
IDENTIFIER   = b"WireGuard v1 zx2c4 Jason@zx2c4.com"
LABEL_MAC1   = b"mac1----"
LABEL_COOKIE = b"cookie--"

def blake2s(data: bytes) -> bytes:
    return hashlib.blake2s(data).digest()

def hkdf(key: bytes, input_data: bytes, n: int) -> tuple:
    """HKDF로 n개의 출력 도출"""
    prk = hmac.new(key, input_data, hashlib.sha256).digest()
    outputs = []
    T = b""
    for i in range(1, n + 1):
        T = hmac.new(prk, T + bytes([i]), hashlib.sha256).digest()
        outputs.append(T)
    return tuple(outputs)

class HandshakeState:
    """Noise 핸드셰이크 상태"""
    
    def __init__(self):
        # Chaining Key: 이전 핸드셰이크 재료를 누적
        self.ck = blake2s(CONSTRUCTION)
        # Hash: 핸드셰이크 전사록(transcript) 해시
        self.h  = blake2s(self.ck + IDENTIFIER)
        self.k  = b"\x00" * 32  # 현재 임시 암호화 키
        self.n  = 0              # nonce
    
    def mix_hash(self, data: bytes):
        """핸드셰이크 전사록에 데이터 추가"""
        self.h = blake2s(self.h + data)
    
    def mix_key(self, dh_result: bytes):
        """DH 결과를 Chaining Key와 혼합"""
        self.ck, self.k = hkdf(self.ck, dh_result, 2)
        self.n = 0
    
    def encrypt_and_hash(self, plaintext: bytes) -> bytes:
        """데이터를 암호화하고 전사록에 포함"""
        aead = ChaCha20Poly1305(self.k)
        nonce = b"\x00" * 4 + self.n.to_bytes(8, 'little')
        ciphertext = aead.encrypt(nonce, plaintext, self.h)
        self.mix_hash(ciphertext)
        self.n += 1
        return ciphertext
    
    def decrypt_and_hash(self, ciphertext: bytes) -> bytes:
        """데이터를 복호화하고 전사록에 포함"""
        aead = ChaCha20Poly1305(self.k)
        nonce = b"\x00" * 4 + self.n.to_bytes(8, 'little')
        plaintext = aead.decrypt(nonce, ciphertext, self.h)
        self.mix_hash(ciphertext)
        self.n += 1
        return plaintext

class WireGuardInitiator:
    def create_initiation(self, 
                          initiator_static_priv, initiator_static_pub,
                          responder_static_pub,
                          psk=None):
        state = HandshakeState()
        state.mix_hash(responder_static_pub)
        
        # 1. 임시 키 쌍 생성
        ephemeral_priv = X25519PrivateKey.generate()
        ephemeral_pub = ephemeral_priv.public_key().public_bytes_raw()
        
        # 2. 임시 공개키 전송 (평문)
        state.mix_hash(ephemeral_pub)
        state.mix_key(
            ephemeral_priv.exchange(responder_static_pub)  # DH1: e_i * S_r
        )
        
        # 3. Initiator 정적 공개키 암호화 전송
        encrypted_static = state.encrypt_and_hash(initiator_static_pub)
        state.mix_key(
            initiator_static_priv.exchange(responder_static_pub)  # DH2: S_i * S_r
        )
        
        # 4. 타임스탬프 암호화 (재전송 공격 방지)
        import time, struct
        timestamp = struct.pack(">Q", int(time.time() * 1e9))  # TAI64N
        encrypted_timestamp = state.encrypt_and_hash(timestamp)
        
        return {
            "unencrypted_ephemeral": ephemeral_pub,
            "encrypted_static": encrypted_static,
            "encrypted_timestamp": encrypted_timestamp,
            "_state": state,
            "_ephemeral_priv": ephemeral_priv,
        }
```

### Perfect Forward Secrecy와 Identity Hiding

WireGuard의 핸드셰이크 설계의 핵심 보안 속성:

1. **Perfect Forward Secrecy (PFS)**: 매 세션마다 새 임시 키(ephemeral key)를 사용. 장기 정적 키가 노출되어도 과거 세션의 복호화 불가
2. **Identity Hiding**: Initiator의 정적 공개키는 Responder의 정적 공개키로 암호화되어 전송. 도청자는 누가 연결을 시작했는지 알 수 없음
3. **Replay Protection**: TAI64N 타임스탬프를 암호화하여 재전송 공격 방지. Responder는 가장 최근에 수신한 타임스탬프보다 큰 값만 수락

## 데이터 패킷 처리

핸드셰이크 완료 후 두 개의 ChaCha20-Poly1305 세션 키가 생성된다:
- `send_key`: Initiator → Responder 방향 암호화
- `recv_key`: Responder → Initiator 방향 암호화

```
┌─────────────────────────────────────────────┐
│ WireGuard 데이터 패킷 구조                    │
├──────────┬────────────────┬─────────────────┤
│ type (1) │ receiver_index │    counter (8)   │
│  1바이트  │    4바이트      │    8바이트       │
├──────────┴────────────────┴─────────────────┤
│    encrypted_encapsulated_packet             │
│    (IP 패킷 + 16바이트 Poly1305 태그)         │
└─────────────────────────────────────────────┘
```

`counter`는 Anti-Replay 윈도우로 사용된다. 수신자는 비트마스크를 사용해 최근 2048개의 counter 값을 추적하며, 이미 수신한 counter가 있으면 패킷을 버린다.

## Linux 커널 통합: 네트워크 가상 인터페이스

WireGuard는 Linux에서 **가상 네트워크 인터페이스**(`wg0`, `wg1`, ...)로 노출된다. 커널 내부에서는 UDP 소켓을 사용해 암호화된 데이터를 전송하고, 복호화된 데이터는 일반 IP 스택으로 전달된다.

### 커널 데이터 경로

```
         사용자 공간         │         커널 공간
                            │
앱 → 소켓 → 라우팅 테이블   │  wg0 인터페이스 (tun 드라이버)
                    │       │        │
                    └───────┤───> WireGuard 모듈
                            │        │
                            │   암호화 (ChaCha20-Poly1305)
                            │        │
                            │   UDP 소켓 → 실제 NIC
```

### 라우팅: Allowed IPs와 크립토라우팅 테이블

WireGuard의 피어 구성에서 `AllowedIPs`는 단순한 라우팅 규칙이 아니라 **암호화 라우팅 테이블(Cryptokey Routing Table)**의 핵심이다.

```ini
# /etc/wireguard/wg0.conf

[Interface]
PrivateKey = [서버 개인키]
Address = 10.0.0.1/24
ListenPort = 51820

[Peer]
PublicKey = [피어1 공개키]
AllowedIPs = 10.0.0.2/32  # 이 IP에서 오는 패킷만 피어1의 키로 복호화
Endpoint = 203.0.113.1:51820

[Peer]
PublicKey = [피어2 공개키]
AllowedIPs = 10.0.0.3/32, 192.168.1.0/24  # 여러 서브넷도 가능
```

**송신 시**: 목적지 IP를 AllowedIPs 테이블에서 가장 긴 프리픽스 매칭으로 찾아 해당 피어의 키로 암호화
**수신 시**: 패킷을 복호화한 후 출발지 IP가 해당 피어의 AllowedIPs에 속하는지 검증

이 설계는 단순하지만 강력하다. 피어가 위조된 IP를 사용해도 복호화 후 검증 단계에서 걸러진다.

### 실제 WireGuard 설정과 사용

```bash
# 키 생성
wg genkey | tee server_private.key | wg pubkey > server_public.key
wg genkey | tee client_private.key | wg pubkey > client_public.key

# 서버 설정 (/etc/wireguard/wg0.conf)
cat > /etc/wireguard/wg0.conf << 'EOF'
[Interface]
PrivateKey = $(cat server_private.key)
Address = 10.8.0.1/24
ListenPort = 51820
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = $(cat client_public.key)
AllowedIPs = 10.8.0.2/32
EOF

# 클라이언트 설정
cat > /etc/wireguard/wg1.conf << 'EOF'
[Interface]
PrivateKey = $(cat client_private.key)
Address = 10.8.0.2/24
DNS = 1.1.1.1

[Peer]
PublicKey = $(cat server_public.key)
AllowedIPs = 0.0.0.0/0  # 모든 트래픽을 터널로
Endpoint = server_ip:51820
PersistentKeepalive = 25  # NAT 통과 유지
EOF

# 시작
wg-quick up wg0   # 서버
wg-quick up wg1   # 클라이언트

# 상태 확인
wg show
```

### Go 언어로 WireGuard 라이브러리 사용

```go
package main

import (
    "fmt"
    "log"
    "net"
    "os"

    "golang.zx2c4.com/wireguard/conn"
    "golang.zx2c4.com/wireguard/device"
    "golang.zx2c4.com/wireguard/ipc"
    "golang.zx2c4.com/wireguard/tun"
    "golang.zx2c4.com/wireguard/wgctrl"
    "golang.zx2c4.com/wireguard/wgctrl/wgtypes"
)

func createWireGuardInterface() error {
    // TUN 인터페이스 생성
    tunIface, err := tun.CreateTUN("wg0", device.DefaultMTU)
    if err != nil {
        return fmt.Errorf("TUN 생성 실패: %w", err)
    }

    // WireGuard 디바이스 생성
    logger := device.NewLogger(device.LogLevelVerbose, "(wg0) ")
    wgDevice := device.NewDevice(tunIface, conn.NewDefaultBind(), logger)

    // UAPI 소켓 설정 (wg 도구와 통신)
    uapiFile, err := ipc.UAPIOpen("wg0")
    if err != nil {
        return fmt.Errorf("UAPI 소켓 열기 실패: %w", err)
    }
    uapiServer, err := ipc.UAPIListen("wg0", uapiFile)
    if err != nil {
        return fmt.Errorf("UAPI 리슨 실패: %w", err)
    }
    go func() {
        for {
            conn, err := uapiServer.Accept()
            if err != nil {
                return
            }
            go wgDevice.IpcHandle(conn)
        }
    }()

    // wgctrl로 피어 설정
    client, err := wgctrl.New()
    if err != nil {
        return fmt.Errorf("wgctrl 클라이언트 생성 실패: %w", err)
    }
    defer client.Close()

    privateKey, err := wgtypes.GeneratePrivateKey()
    if err != nil {
        return err
    }

    port := 51820
    peerPubKeyStr := "피어_공개키_Base64_문자열"
    peerPubKey, _ := wgtypes.ParseKey(peerPubKeyStr)
    
    _, allowedNet, _ := net.ParseCIDR("10.0.0.0/24")

    cfg := wgtypes.Config{
        PrivateKey: &privateKey,
        ListenPort: &port,
        Peers: []wgtypes.PeerConfig{
            {
                PublicKey: peerPubKey,
                AllowedIPs: []net.IPNet{*allowedNet},
                Endpoint: &net.UDPAddr{
                    IP:   net.ParseIP("203.0.113.1"),
                    Port: 51820,
                },
            },
        },
    }

    return client.ConfigureDevice("wg0", cfg)
}

func main() {
    if err := createWireGuardInterface(); err != nil {
        log.Fatal(err)
    }
    fmt.Println("WireGuard 인터페이스 설정 완료")
    select {} // 실행 유지
}
```

## Roaming과 Connection Migration

WireGuard의 설계에서 인상적인 점 중 하나는 **IP 주소와 세션을 분리**한 것이다. 기존 VPN은 클라이언트 IP가 변경되면(WiFi → LTE 전환 등) 재연결이 필요하다. WireGuard는 피어의 **공개키**를 기준으로 피어를 식별하므로, 클라이언트 IP가 바뀌어도 서버는 최신 IP로 자동 업데이트하여 연결을 유지한다.

```
클라이언트: WiFi(1.2.3.4) → LTE(5.6.7.8)
                              │
                              ↓ 새 IP에서 패킷 수신
서버: "이 공개키는 알고 있는 피어!
      엔드포인트 주소를 5.6.7.8로 업데이트"
                              │
                              ↓ 즉시 연결 복구
```

## 성능: 왜 빠른가

WireGuard는 여러 설계 결정으로 성능을 극대화했다:

1. **커널 내 암호화**: 사용자 공간 복사 없이 커널에서 직접 암호화
2. **ChaCha20의 하드웨어 최적화**: ARM, x86 모두 어셈블리 최적화 구현
3. **멀티코어 병렬화**: 패킷 암복호화가 여러 CPU 코어에서 병렬 처리
4. **작은 코드 = 낮은 캐시 풋프린트**: L1 캐시에 코드가 다 들어감

```bash
# iperf3로 성능 측정 예시
# WireGuard: 보통 9~10 Gbps (10Gbps NIC 기준)
# OpenVPN(TCP): 보통 1~2 Gbps
# IPsec: 보통 5~8 Gbps (하드웨어 가속 포함)

iperf3 -s  # 서버
iperf3 -c 10.8.0.1 -P 4 -t 30  # 클라이언트 (4 병렬 스트림)
```

## 주의사항

### 메타데이터 보호의 한계

WireGuard는 패킷 내용을 암호화하지만, 패킷 크기, 타이밍, 목적지 IP(Responder 서버 IP)는 여전히 노출된다. 강력한 익명성이 필요하면 Tor 위에 WireGuard를 운용하거나, 커버 트래픽 기법을 사용해야 한다.

### 정적 키 관리

WireGuard에는 인증서 기반 PKI가 없다. 피어 공개키를 직접 배포하고 관리해야 한다. 대규모 배포에서는 **Headscale**(오픈소스 Tailscale 컨트롤 플레인)이나 별도의 키 관리 시스템이 필요하다.

### 양자 내성

현재 WireGuard는 Curve25519(ECDH)를 사용하며, 충분히 강력한 양자 컴퓨터가 등장하면 취약해진다. Post-Quantum WireGuard(PQ WireGuard) 연구가 진행 중이며, NIST PQC 표준화 완료 알고리즘(CRYSTALS-Kyber 등)과의 하이브리드 통합이 논의되고 있다.

## 참고 자료
- [WireGuard 공식 논문 (NDSSSymposium 2017)](https://www.wireguard.com/papers/wireguard.pdf)
- [Noise Protocol Framework 명세](https://noiseprotocol.org/noise.html)
- [WireGuard 공식 사이트](https://www.wireguard.com/)
- [wireguard-go (Go 구현체)](https://git.zx2c4.com/wireguard-go)
