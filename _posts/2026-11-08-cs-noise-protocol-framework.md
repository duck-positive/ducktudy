---
layout: post
title: "노이즈 프로토콜 프레임워크(Noise Protocol Framework) 완전 정복: Signal·WireGuard가 선택한 암호화 핸드셰이크 설계 도구"
date: 2026-11-08
categories: [cs, computer-science]
tags: [noise-protocol, cryptography, handshake, wireguard, signal, diffie-hellman, security, tls]
---

## 1. 노이즈 프로토콜 프레임워크란?

노이즈 프로토콜 프레임워크(Noise Protocol Framework)는 Trevor Perrin이 설계하고 2018년 Revision 34로 최종 명세화된 **암호화 핸드셰이크 프로토콜 설계 도구(toolkit)**이다. 하나의 단일 프로토콜이 아니라, 다양한 보안 요구사항을 갖는 핸드셰이크 프로토콜을 작은 조각들(pattern + DH function + cipher + hash)을 조합해 구성할 수 있는 프레임워크이다.

**Noise를 채택한 실제 시스템들:**
- **WireGuard**: Linux 커널 VPN — `Noise_IKpsk2_25519_ChaChaPoly_BLAKE2s`
- **Signal Protocol**: 안전 메시징의 표준 (Signal, WhatsApp, Facebook Messenger의 비밀 대화)
- **Lightning Network**: Bitcoin Layer 2 채널 개설 핸드셰이크
- **libp2p**: IPFS, Ethereum 2.0의 P2P 네트워킹 레이어

## 2. 왜 TLS 대신 Noise인가?

TLS 1.3은 훌륭한 프로토콜이지만 다음과 같은 한계가 있다:

**복잡성:** TLS는 수십 년에 걸쳐 쌓인 수백 개의 옵션과 협상 메커니즘을 가진다. 구현의 복잡성이 곧 보안 취약점의 원천이 된다(BEAST, POODLE, Heartbleed 모두 이 복잡성에서 기인).

**불필요한 PKI 의존성:** TLS는 인증서 기반 PKI(Public Key Infrastructure)를 가정한다. 그러나 P2P 네트워크, IoT, VPN 같은 환경에서는 사전에 공개 키를 알고 있거나, 익명 연결을 원하거나, 다양한 인증 모델이 필요하다.

**Noise의 장점:**
1. **명세가 단 44페이지**: 전체 프레임워크를 한 사람이 완전히 이해할 수 있다
2. **조합 가능성**: 필요한 보안 속성만 가진 프로토콜을 선택/설계할 수 있다
3. **형식 검증 가능**: ProVerif, Tamarin 등으로 보안 속성을 형식 증명할 수 있다
4. **고성능**: 불필요한 협상 메시지가 없어 RTT가 최소화된다

## 3. Noise의 구조: 프로토콜 이름 해부

Noise 프로토콜의 이름은 다음 형식을 따른다:

```
Noise_<HandshakePattern>_<DHFunction>_<CipherFunction>_<HashFunction>

예시:
Noise_XX_25519_AESGCM_SHA256
  ↑    ↑↑  ↑         ↑       ↑
  │    │└──┘         │       └─ 해시 함수 (SHA256, SHA512, BLAKE2s, BLAKE2b)
  │    │  DH 함수   │
  │    │  (25519,   └─ 대칭 암호 (AESGCM, ChaChaPoly)
  │    │   448)
  │    └─ 핸드셰이크 패턴 (N, K, X, NN, NK, NX, KN, KK, KX, XN, XK, XX, IX, IK, NNpsk0...)
  └─ "Noise" 접두사
```

**PSK(Pre-Shared Key) 변형:** WireGuard처럼 PSK를 추가로 혼합하고 싶다면 패턴 뒤에 `psk<N>`을 붙인다. 예: `Noise_IKpsk2_25519_ChaChaPoly_BLAKE2s` (IK 패턴에 2번째 메시지 처리 후 PSK를 혼합).

## 4. 핸드셰이크 패턴

핸드셰이크 패턴의 이름은 발신자와 수신자가 각자의 정적(static) 키를 언제 전송하는지를 나타낸다.

| 패턴 이름 | 의미 | 보안 속성 |
|-----------|------|-----------|
| `NN` | 양방향 모두 익명 | 도청 방지만 |
| `NK` | 발신자 익명, 수신자 사전 공개키 보유 | 수신자 인증 |
| `KK` | 양방향 모두 사전 공개키 보유 | 양방향 인증 |
| `XX` | 모두 핸드셰이크 중 공개키 교환 | 양방향 인증 + 공개키 기밀 |
| `IX` | 발신자가 즉시 공개키 전송 | 발신자 신원 노출 감수 |
| `IK` | 발신자 즉시 전송, 수신자 사전 공개키 보유 | 0.5-RTT 가능 |

가장 흔히 사용되는 `XX` 패턴을 살펴보자:

```
Initiator                   Responder
---------                   ---------
  → e                        (임시 공개키 전송)
  ← e, ee, s, es             (임시키 + DH, 정적키 + DH)
  → s, se                    (정적키 + DH)
  ← [응용 데이터]
```

화살표의 각 토큰:
- `e`: 임시(ephemeral) 키 쌍 생성 후 공개키 전송
- `s`: 정적(static) 공개키 전송 (암호화된 상태로)
- `ee`: `DH(e_I, e_R)` — 임시키끼리 DH
- `es`: `DH(e_I, s_R)` — 발신자 임시키 × 수신자 정적키 DH
- `se`: `DH(s_I, e_R)` — 발신자 정적키 × 수신자 임시키 DH

## 5. 핵심 상태 머신: HandshakeState 구현

```python
"""
Noise_XX_25519_AESGCM_SHA256 핸드셰이크 상태 머신 (교육용 단순화 버전)
"""
import os
import hashlib
import hmac
from cryptography.hazmat.primitives.asymmetric.x25519 import X25519PrivateKey
from cryptography.hazmat.primitives.ciphers.aead import AESGCM

DHLEN = 32  # X25519 출력 길이

class CipherState:
    """암호화 상태: 키(k)와 논스(n)를 관리"""
    def __init__(self):
        self.k = None   # 32-byte 키
        self.n = 0      # 64-bit 논스 카운터

    def initialize_key(self, key: bytes):
        self.k = key
        self.n = 0

    def encrypt_with_ad(self, ad: bytes, plaintext: bytes) -> bytes:
        if self.k is None:
            return plaintext
        nonce = self.n.to_bytes(12, 'little')  # 96-bit 논스 (IV)
        self.n += 1
        aead = AESGCM(self.k)
        return aead.encrypt(nonce, plaintext, ad)

    def decrypt_with_ad(self, ad: bytes, ciphertext: bytes) -> bytes:
        if self.k is None:
            return ciphertext
        nonce = self.n.to_bytes(12, 'little')
        self.n += 1
        aead = AESGCM(self.k)
        return aead.decrypt(nonce, ciphertext, ad)


class SymmetricState:
    """
    핸드셰이크 중 체인 키(ck)와 해시(h)를 관리.
    모든 핸드셰이크 메시지를 h에 혼합하여 무결성 보장.
    """
    def __init__(self, protocol_name: str):
        self.cipher = CipherState()
        name_bytes = protocol_name.encode('utf-8')
        if len(name_bytes) <= 32:
            self.h = name_bytes.ljust(32, b'\x00')
        else:
            self.h = hashlib.sha256(name_bytes).digest()
        self.ck = self.h  # 체인 키

    def _hkdf(self, ikm: bytes, num_outputs: int):
        """HKDF-SHA256: ck에서 새 키 파생"""
        prk = hmac.new(self.ck, ikm, hashlib.sha256).digest()
        output1 = hmac.new(prk, b'\x01', hashlib.sha256).digest()
        if num_outputs == 1:
            return (output1,)
        output2 = hmac.new(prk, output1 + b'\x02', hashlib.sha256).digest()
        if num_outputs == 2:
            return (output1, output2)
        output3 = hmac.new(prk, output2 + b'\x03', hashlib.sha256).digest()
        return (output1, output2, output3)

    def mix_key(self, input_key_material: bytes):
        """DH 결과를 ck와 k에 혼합"""
        self.ck, temp_k = self._hkdf(input_key_material, 2)
        self.cipher.initialize_key(temp_k[:32])

    def mix_hash(self, data: bytes):
        """데이터를 해시 h에 혼합 (전사적 무결성)"""
        self.h = hashlib.sha256(self.h + data).digest()

    def encrypt_and_hash(self, plaintext: bytes) -> bytes:
        ciphertext = self.cipher.encrypt_with_ad(self.h, plaintext)
        self.mix_hash(ciphertext)
        return ciphertext

    def decrypt_and_hash(self, ciphertext: bytes) -> bytes:
        plaintext = self.cipher.decrypt_with_ad(self.h, ciphertext)
        self.mix_hash(ciphertext)
        return plaintext

    def split(self):
        """핸드셰이크 완료 후 송신/수신 CipherState 분리"""
        temp_k1, temp_k2 = self._hkdf(b'', 2)
        c1, c2 = CipherState(), CipherState()
        c1.initialize_key(temp_k1)
        c2.initialize_key(temp_k2)
        return c1, c2


class HandshakeState:
    """Noise_XX 핸드셰이크 상태 머신"""
    def __init__(self, is_initiator: bool, s_priv: X25519PrivateKey, rs_pub=None):
        self.symmetric = SymmetricState("Noise_XX_25519_AESGCM_SHA256")
        self.is_initiator = is_initiator
        self.s = s_priv                   # 나의 정적 키 쌍
        self.e = X25519PrivateKey.generate()  # 임시 키 쌍
        self.rs = rs_pub                  # 상대방 정적 공개키
        self.re = None                    # 상대방 임시 공개키
        self.symmetric.mix_hash(b'')      # 프롤로그

    def _dh(self, priv, pub_bytes: bytes) -> bytes:
        from cryptography.hazmat.primitives.asymmetric.x25519 import X25519PublicKey
        pub = X25519PublicKey.from_public_bytes(pub_bytes)
        return priv.exchange(pub)

    def write_message_1(self) -> bytes:
        """→ e (발신자 임시 공개키 전송)"""
        e_pub = self.e.public_key().public_bytes_raw()
        self.symmetric.mix_hash(e_pub)
        return e_pub

    def read_message_1(self, msg: bytes):
        """발신자의 임시 공개키 수신"""
        self.re = msg[:32]
        self.symmetric.mix_hash(self.re)

    def write_message_2(self) -> bytes:
        """← e, ee, s, es"""
        buf = b''
        # e: 수신자 임시 공개키
        e_pub = self.e.public_key().public_bytes_raw()
        self.symmetric.mix_hash(e_pub)
        buf += e_pub
        # ee: DH(e_R, e_I)
        self.symmetric.mix_key(self._dh(self.e, self.re))
        # s: 수신자 정적 공개키 (암호화)
        s_pub = self.s.public_key().public_bytes_raw()
        buf += self.symmetric.encrypt_and_hash(s_pub)
        # es: DH(s_R, e_I)
        self.symmetric.mix_key(self._dh(self.s, self.re))
        return buf

    def split(self):
        return self.symmetric.split()


# 사용 예시
alice_key = X25519PrivateKey.generate()
bob_key   = X25519PrivateKey.generate()

alice_hs = HandshakeState(is_initiator=True, s_priv=alice_key)
bob_hs   = HandshakeState(is_initiator=False, s_priv=bob_key)

msg1 = alice_hs.write_message_1()      # → e
bob_hs.read_message_1(msg1)
msg2 = bob_hs.write_message_2()        # ← e, ee, s, es
# (msg3 단계 생략 — → s, se 완료 후 split)

print(f"핸드셰이크 메시지 1 크기: {len(msg1)} bytes")  # 32 bytes
print(f"핸드셰이크 메시지 2 크기: {len(msg2)} bytes")  # 32 + 32+16 + 32+16 = 128 bytes
```

## 6. WireGuard가 Noise를 사용하는 방식

WireGuard는 `Noise_IKpsk2_25519_ChaChaPoly_BLAKE2s`를 사용하며 1-RTT 핸드셰이크를 달성한다.

```
Initiator                           Responder
---------                           ---------
[Initiator Handshake Message]  →
  - Sender Index (4 bytes)
  - Encrypted Initiator Static Key (32+16 bytes) [es로 암호화]
  - Encrypted Timestamp (12+16 bytes)
  - MAC1 (16 bytes)
  - MAC2 (16 bytes, cookie 기반 DoS 방어)

                              ←  [Responder Handshake Message]
                                   - Sender Index (4 bytes)
                                   - Receiver Index (4 bytes)
                                   - Encrypted Empty (0+16 bytes) [ee, se, psk로 암호화]
                                   - MAC1 (16 bytes)

[데이터 패킷 즉시 전송 가능]  →
```

**IK 패턴의 의미:**
- `I` (Initiator): 발신자가 첫 메시지에서 즉시 정적 공개키 전송
- `K` (Known): 수신자의 정적 공개키를 발신자가 사전에 알고 있음 (WireGuard 설정 파일에 등록)

이 덕분에 발신자는 첫 메시지부터 `DH(e_I, s_R)` (발신자 임시 × 수신자 정적 DH)로 암호화할 수 있어 1-RTT 연결이 가능하다.

## 7. Noise의 보안 속성 분류

**전방향 비밀성(Forward Secrecy):** `ee` DH 토큰이 포함되면 임시 키는 핸드셰이크 후 삭제되므로, 나중에 정적 키가 유출되어도 과거 세션을 복호화할 수 없다.

**후방 비밀성(Future Secrecy):** 임시 키 교환이 세션마다 새로 이루어지므로 한 세션이 뚫려도 다른 세션은 안전하다.

**신원 숨김(Identity Hiding):** `XX` 패턴에서 정적 공개키는 암호화된 채 전송되므로 수동 도청자는 누가 누구와 통신하는지 알 수 없다.

**Zero-RTT 가능성:** PSK를 사전에 공유한 경우 첫 번째 메시지에 데이터를 실어 보낼 수 있다. 단 replay attack 위험이 있으므로 WireGuard는 타임스탬프로 방어한다.

## 8. 주의사항 및 함정

**패턴 선택의 중요성:** 잘못된 패턴 선택은 보안 속성 누락으로 이어진다. 예를 들어 `KN` 패턴에서 수신자는 발신자를 인증할 수 없다. 패턴 선택 전 Noise Explorer(formal verification tool)로 보안 속성을 반드시 확인하라.

**정적 키 재사용 금지:** Noise 세션 간에 정적 키를 공유하면 프로토콜 간 공격(cross-protocol attack)이 가능해질 수 있다. 각 프로토콜 목적마다 독립된 정적 키를 사용해야 한다.

**논스 단조 증가 필수:** CipherState의 논스가 재사용되면 AES-GCM/ChaCha20-Poly1305의 보안이 완전히 무너진다. 세션 간 논스 상태를 공유하거나 오버플로를 허용하면 안 된다 (Noise는 최대 2^64-1 메시지 후 세션 종료를 권고).

**구현 라이브러리 활용:** 직접 구현보다 검증된 라이브러리를 사용하라. `libsodium`, `snow`(Rust), `noise-c`, `noise-java` 등이 Noise 명세를 따른다.

## 참고 자료
- [Noise Protocol Framework 공식 명세 (Rev. 34)](https://noiseprotocol.org/noise.html)
- [Noise Explorer — 형식 검증 도구](https://noiseexplorer.com/)
- [WireGuard 논문 — Jason A. Donenfeld](https://www.wireguard.com/papers/wireguard.pdf)
- [snow — Rust Noise 구현 (GitHub)](https://github.com/mcginty/snow)
