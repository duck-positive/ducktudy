---
layout: post
title: "Double Ratchet 알고리즘 완전 정복: Signal·WhatsApp이 선택한 전방향 비밀성과 미래 비밀성의 수학"
date: 2026-10-22
categories: [cs, computer-science]
tags: [DoubleRatchet, Signal, 종단간암호화, E2EE, 전방향비밀성, X3DH, Diffie-Hellman, 메시지암호화]
---

WhatsApp, Signal, iMessage, Facebook Messenger의 비밀 대화 기능... 이들은 모두 "종단간 암호화(E2EE)"를 제공한다고 광고합니다. 그런데 이 암호화는 정확히 어떻게 동작할까요? 메시지마다 다른 키를 사용하고, 과거 메시지가 현재 키 유출로 복호화되지 않으며, 단기간의 키 유출에서도 빠르게 회복할 수 있는 마법. 이것이 바로 **Double Ratchet 알고리즘**이 제공하는 보장입니다. Trevor Perrin과 Moxie Marlinspike가 2013년 설계하고 Signal 프로토콜에 구현된 이 알고리즘은 현대 보안 메시징의 근간입니다.

## 배경: 종단간 암호화의 요구사항

보안 메시징 시스템이 달성해야 하는 핵심 보안 속성을 먼저 이해해야 합니다:

### 전방향 비밀성 (Forward Secrecy, FS)
**과거 메시지 보호**: 현재의 개인키가 유출되더라도, 과거에 교환된 메시지를 복호화할 수 없어야 합니다.

기존 TLS의 경우, 서버 인증서 개인키가 유출되면 과거 녹화된 모든 트래픽이 복호화될 수 있었습니다(TLS 1.3은 ECDHE를 필수로 하여 이를 해결). 메시징에서 이 문제는 더 심각합니다.

### 미래 비밀성 (Break-in Recovery / Future Secrecy, PCS)
**침해 후 회복**: 공격자가 잠시 개인키에 접근했더라도, 이후 메시지는 다시 안전해져야 합니다. Post-Compromise Security라고도 불립니다.

### 부인 가능성 (Deniability)
메시지 수신자가 나중에 "Alice가 이 메시지를 썼다"고 증명할 수 없어야 합니다. 디지털 서명은 이를 위반하므로 MAC을 사용합니다.

Double Ratchet는 이 세 가지를 동시에 달성합니다.

## 선행 기술: Diffie-Hellman 키 교환

Double Ratchet를 이해하려면 먼저 **ECDH(Elliptic Curve Diffie-Hellman)** 키 교환을 이해해야 합니다.

```
Alice                              Bob
IK_A = 개인키                      IK_B = 개인키
IK_A_pub = G * IK_A = 공개키      IK_B_pub = G * IK_B = 공개키

              IK_A_pub 전송
              ────────────►
              IK_B_pub 전송
              ◄────────────

shared_secret = IK_A * IK_B_pub    shared_secret = IK_B * IK_A_pub
              = IK_A * G * IK_B    = IK_B * G * IK_A
              ← 두 값이 같음 (타원곡선 연산의 교환법칙) →
```

그러나 정적 DH 키만으로는 전방향 비밀성을 제공하지 못합니다. Alice의 장기 개인키가 유출되면 모든 세션이 복호화됩니다.

## X3DH: 비동기 세션 초기화

Double Ratchet가 시작되기 전에, 두 사용자가 처음 통신할 때 **X3DH(Extended Triple Diffie-Hellman)** 프로토콜로 초기 공유 비밀을 수립합니다.

X3DH는 다음 키들을 사용합니다:

- **IK**: Identity Key (장기 서명 키쌍)
- **SPK**: Signed Prekey (중기 DH 키쌍, 주기적 교체)
- **OPK**: One-Time Prekey (1회용 DH 키쌍)
- **EK**: Ephemeral Key (매 세션마다 생성되는 임시 키쌍)

```
Bob이 서버에 사전 등록:
  - IK_B_pub
  - SPK_B_pub (IK_B로 서명)
  - OPK_B_pub[1...n] (선택적)

Alice가 Bob에게 처음 메시지 보낼 때:
  1. EK_A 임시 키쌍 생성
  2. DH1 = DH(IK_A, SPK_B_pub)
  3. DH2 = DH(EK_A, IK_B_pub)
  4. DH3 = DH(EK_A, SPK_B_pub)
  5. DH4 = DH(EK_A, OPK_B_pub) ← OPK 사용 시
  6. master_secret = KDF(DH1 || DH2 || DH3 || DH4)

Bob이 메시지 수신 시 같은 계산으로 master_secret 도출
```

X3DH는 비동기 통신(Bob이 오프라인이어도 Alice가 먼저 메시지 가능)을 지원하면서도 전방향 비밀성을 제공합니다.

## Double Ratchet의 두 래칫

이름에서 알 수 있듯, Double Ratchet는 **두 개의 래칫**을 결합합니다:

### 1. KDF Chain Ratchet (대칭 키 래칫)

**KDF(Key Derivation Function) 체인**은 연속적으로 새로운 키를 파생합니다. HKDF(HMAC-based Key Derivation Function)를 사용합니다:

```
Chain Key[0]
    │
    ▼ KDF
Chain Key[1] + Message Key[1]
    │
    ▼ KDF
Chain Key[2] + Message Key[2]
    │
    ▼ KDF
    ...

각 Message Key로 메시지 암호화 (AES-256-CBC 또는 AES-256-GCM)
```

특징:
- Chain Key는 앞으로만 진행 (래칫처럼 역방향 불가)
- 각 메시지는 고유한 Message Key 사용 후 폐기
- 과거 Message Key는 사용 후 즉시 메모리에서 삭제

이것이 **전방향 비밀성**을 제공합니다. Chain Key[10]이 유출되어도 Key[0-9]는 알 수 없습니다.

### 2. Diffie-Hellman Ratchet (비대칭 DH 래칫)

KDF 체인 단독으로는 "현재 상태 유출 → 미래 복호화" 문제를 해결하지 못합니다. 공격자가 현재 Chain Key를 가지면 이후 모든 메시지를 복호화할 수 있습니다.

**DH 래칫**이 이 문제를 해결합니다:

```
Alice                              Bob
────────────────────────────────────────────────────────
DH_A1 = 새 임시 키쌍              DH_B1 = 새 임시 키쌍

  DH_A1.pub 포함하여 메시지 전송
  ─────────────────────────────►
                                  DH_B2 = 새 임시 키쌍 생성!
                                  RK, CK_B = KDF_RK(RK, DH(DH_B2, DH_A1.pub))
  ◄─────────────────────────────
  DH_B2.pub 포함하여 메시지 수신

Alice: DH_A2 = 새 임시 키쌍 생성!
  RK, CK_A = KDF_RK(RK, DH(DH_A2, DH_B2.pub))
```

메시지를 수신할 때마다 새 DH 키쌍을 생성하고, 새 DH 교환 결과로 루트 키(RK)와 체인 키를 갱신합니다. 이렇게 하면 과거 상태가 유출되더라도, 새 DH 교환이 발생하면 공격자는 새 체인을 추적할 수 없습니다.

이것이 **미래 비밀성(Break-in Recovery)**입니다.

## Double Ratchet 완전 상태 모델

```python
# Double Ratchet 상태 구조 (Python 의사코드)
@dataclass
class State:
    # 자신의 DH 래칫 키쌍
    DHs: KeyPair  # 현재 DH 발신 키쌍
    DHr: PublicKey  # 상대방의 현재 DH 공개키

    # 루트 키 (DH 래칫 출력)
    RK: bytes  # 32 bytes

    # 체인 키 (KDF 체인 상태)
    CKs: bytes  # 발신 체인 키 (32 bytes)
    CKr: bytes  # 수신 체인 키 (32 bytes)

    # 메시지 카운터
    Ns: int  # 발신 메시지 수
    Nr: int  # 수신 메시지 수
    PN: int  # 이전 발신 체인의 메시지 수

    # 순서 어긋난 메시지를 위한 버퍼
    # key: (DH_pub, message_number) → message_key
    MKSKIPPED: dict
```

## 실제 구현: Python으로 Double Ratchet 구현

```python
# double_ratchet.py
# 교육 목적의 단순화된 Double Ratchet 구현
# 프로덕션에서는 libsignal 등 검증된 라이브러리를 사용하세요

import os
import hmac
import hashlib
from dataclasses import dataclass, field
from typing import Optional, Tuple
from cryptography.hazmat.primitives.asymmetric.x25519 import X25519PrivateKey, X25519PublicKey
from cryptography.hazmat.primitives.kdf.hkdf import HKDF
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.ciphers.aead import AESGCM

# 상수
MAX_SKIP = 1000  # 순서 어긋난 메시지 최대 건너뛰기 수

def generate_dh() -> X25519PrivateKey:
    """X25519 DH 키쌍 생성"""
    return X25519PrivateKey.generate()

def dh(priv: X25519PrivateKey, pub: X25519PublicKey) -> bytes:
    """X25519 DH 교환: 32바이트 공유 비밀 반환"""
    return priv.exchange(pub)

def kdf_rk(rk: bytes, dh_out: bytes) -> Tuple[bytes, bytes]:
    """루트 KDF: (새 루트 키, 새 체인 키) 반환"""
    # HKDF-SHA256 사용
    hkdf = HKDF(
        algorithm=hashes.SHA256(),
        length=64,
        salt=rk,
        info=b"WhisperRatchet",
    )
    output = hkdf.derive(dh_out)
    return output[:32], output[32:]

def kdf_ck(ck: bytes) -> Tuple[bytes, bytes]:
    """체인 KDF: (새 체인 키, 메시지 키) 반환"""
    # HMAC-SHA256 기반
    msg_key = hmac.new(ck, b'\x01', hashlib.sha256).digest()
    new_ck  = hmac.new(ck, b'\x02', hashlib.sha256).digest()
    return new_ck, msg_key

def encrypt(mk: bytes, plaintext: bytes, associated_data: bytes) -> bytes:
    """AES-256-GCM으로 메시지 암호화"""
    # 메시지 키에서 AES 키와 nonce 파생
    hkdf = HKDF(
        algorithm=hashes.SHA256(),
        length=80,  # 32 (AES key) + 32 (auth key) + 16 (nonce)
        salt=b'\x00' * 32,
        info=b"WhisperMessageKeys",
    )
    material = hkdf.derive(mk)
    enc_key, _auth_key, nonce = material[:32], material[32:64], material[64:]
    aesgcm = AESGCM(enc_key)
    return aesgcm.encrypt(nonce, plaintext, associated_data)

def decrypt(mk: bytes, ciphertext: bytes, associated_data: bytes) -> bytes:
    """AES-256-GCM으로 메시지 복호화"""
    hkdf = HKDF(
        algorithm=hashes.SHA256(),
        length=80,
        salt=b'\x00' * 32,
        info=b"WhisperMessageKeys",
    )
    material = hkdf.derive(mk)
    enc_key, _auth_key, nonce = material[:32], material[32:64], material[64:]
    aesgcm = AESGCM(enc_key)
    return aesgcm.decrypt(nonce, ciphertext, associated_data)

def pub_bytes(priv: X25519PrivateKey) -> bytes:
    return priv.public_key().public_bytes(
        serialization.Encoding.Raw,
        serialization.PublicFormat.Raw
    )

@dataclass
class RatchetState:
    DHs: X25519PrivateKey
    DHr: Optional[X25519PublicKey]
    RK: bytes
    CKs: Optional[bytes]
    CKr: Optional[bytes]
    Ns: int = 0
    Nr: int = 0
    PN: int = 0
    MKSKIPPED: dict = field(default_factory=dict)

def ratchet_init_alice(SK: bytes, bob_dh_pub: X25519PublicKey) -> RatchetState:
    """Alice 측 래칫 초기화 (Alice가 먼저 메시지 보냄)"""
    dh_pair = generate_dh()
    rk, cks = kdf_rk(SK, dh(dh_pair, bob_dh_pub))
    return RatchetState(
        DHs=dh_pair, DHr=bob_dh_pub,
        RK=rk, CKs=cks, CKr=None,
    )

def ratchet_init_bob(SK: bytes, bob_dh_pair: X25519PrivateKey) -> RatchetState:
    """Bob 측 래칫 초기화 (Bob은 Alice의 첫 메시지를 기다림)"""
    return RatchetState(
        DHs=bob_dh_pair, DHr=None,
        RK=SK, CKs=None, CKr=None,
    )

def ratchet_encrypt(state: RatchetState, plaintext: bytes, AD: bytes) -> Tuple[dict, bytes]:
    """메시지 암호화 및 헤더 생성"""
    state.CKs, mk = kdf_ck(state.CKs)
    header = {
        "dh": pub_bytes(state.DHs),
        "pn": state.PN,
        "n": state.Ns,
    }
    state.Ns += 1
    ciphertext = encrypt(mk, plaintext, AD + str(header).encode())
    return header, ciphertext

def ratchet_decrypt(state: RatchetState, header: dict, ciphertext: bytes, AD: bytes) -> bytes:
    """메시지 복호화 (순서 어긋난 메시지 처리 포함)"""
    # 1. 이미 저장된 건너뛴 키 확인
    mk = state.MKSKIPPED.pop((bytes(header["dh"]), header["n"]), None)
    if mk:
        return decrypt(mk, ciphertext, AD + str(header).encode())

    # 2. 새 DH 공개키인지 확인 (DH 래칫 단계)
    from cryptography.hazmat.primitives.asymmetric.x25519 import X25519PublicKey
    new_dh_pub = X25519PublicKey.from_public_bytes(bytes(header["dh"]))
    current_dh_bytes = state.DHr.public_bytes(
        serialization.Encoding.Raw, serialization.PublicFormat.Raw
    ) if state.DHr else None

    if bytes(header["dh"]) != current_dh_bytes:
        # 새 DH 공개키: 이전 체인의 건너뛴 메시지 저장 후 DH 래칫 수행
        _skip_message_keys(state, header["pn"])

        # DH 래칫: 새 수신 체인 키 계산
        state.PN = state.Ns
        state.Ns = 0
        state.Nr = 0
        state.DHr = new_dh_pub
        state.RK, state.CKr = kdf_rk(state.RK, dh(state.DHs, state.DHr))

        # 새 발신 DH 키쌍 생성
        state.DHs = generate_dh()
        state.RK, state.CKs = kdf_rk(state.RK, dh(state.DHs, state.DHr))

    # 3. 현재 체인에서 건너뛴 메시지 처리
    _skip_message_keys(state, header["n"])

    # 4. 현재 메시지 복호화
    state.CKr, mk = kdf_ck(state.CKr)
    state.Nr += 1
    return decrypt(mk, ciphertext, AD + str(header).encode())

def _skip_message_keys(state: RatchetState, until: int):
    """순서 어긋난 메시지를 위한 키 사전 계산 및 저장"""
    if state.Nr + MAX_SKIP < until:
        raise ValueError("Too many skipped messages")
    if state.CKr:
        while state.Nr < until:
            state.CKr, mk = kdf_ck(state.CKr)
            key = (state.DHr.public_bytes(
                serialization.Encoding.Raw, serialization.PublicFormat.Raw
            ), state.Nr)
            state.MKSKIPPED[key] = mk
            state.Nr += 1


# === 사용 예제 ===
def demo():
    # X3DH로 공유 비밀(SK)을 수립했다고 가정
    SK = os.urandom(32)
    bob_keypair = generate_dh()
    bob_pub = bob_keypair.public_key()

    # 래칫 초기화
    alice_state = ratchet_init_alice(SK, bob_pub)
    bob_state   = ratchet_init_bob(SK, bob_keypair)

    AD = b"Alice->Bob"  # Associated Data (인증 컨텍스트)

    # Alice → Bob 메시지
    header1, ct1 = ratchet_encrypt(alice_state, b"Hello, Bob!", AD)
    header2, ct2 = ratchet_encrypt(alice_state, b"How are you?", AD)

    pt1 = ratchet_decrypt(bob_state, header1, ct1, AD)
    pt2 = ratchet_decrypt(bob_state, header2, ct2, AD)
    print(f"Bob received: {pt1}, {pt2}")  # b"Hello, Bob!", b"How are you?"

    # Bob → Alice 메시지 (DH 래칫 발생!)
    header3, ct3 = ratchet_encrypt(bob_state, b"I'm fine, thanks!", AD)
    pt3 = ratchet_decrypt(alice_state, header3, ct3, AD)
    print(f"Alice received: {pt3}")  # b"I'm fine, thanks!"

    # 순서 어긋난 메시지 처리
    header4, ct4 = ratchet_encrypt(alice_state, b"First message", AD)
    header5, ct5 = ratchet_encrypt(alice_state, b"Second message", AD)

    # 나중에 보낸 메시지를 먼저 수신
    pt5 = ratchet_decrypt(bob_state, header5, ct5, AD)
    pt4 = ratchet_decrypt(bob_state, header4, ct4, AD)
    print(f"Out-of-order: {pt5}, {pt4}")  # Second message, First message

if __name__ == "__main__":
    demo()
```

## 보안 분석: 무엇을 보장하고 무엇을 보장하지 않는가

### 보장하는 것

**메시지별 독립 암호화 키**: 각 메시지는 한 번만 사용되는 고유 키로 암호화됩니다. 키가 유출되어도 해당 메시지 하나만 영향받습니다.

**전방향 비밀성**: 현재 상태가 유출되어도 과거 메시지는 안전합니다. 사용된 Message Key는 즉시 폐기되고 역방향 계산이 불가능합니다.

**미래 비밀성 (PCS)**: 각 DH 래칫 단계(새 DH 공개키 교환)는 새 루트 키를 생성합니다. 공격자가 현재 체인 키를 가지더라도, 다음 DH 교환 후 생성된 메시지는 복호화할 수 없습니다.

**순서 어긋난 메시지 지원**: 네트워크 패킷 순서 역전에 안전합니다. `MKSKIPPED`에 미리 계산된 키를 저장합니다.

### 보장하지 않는 것

**기기 침해**: 인증자(스마트폰)가 물리적으로 탈취되어 공격자가 현재 래칫 상태에 접근하면, 이후 메시지를 실시간으로 복호화할 수 있습니다. 이는 암호학이 아닌 기기 보안의 문제입니다.

**메타데이터**: Double Ratchet는 메시지 내용을 보호하지만, 누가 언제 누구에게 메시지를 보냈는지(메타데이터)는 보호하지 않습니다. Signal은 봉인된 발신자(Sealed Sender) 기능으로 이를 보완합니다.

**트래픽 분석**: 메시지 크기, 타이밍 패턴을 분석하면 통신 패턴을 파악할 수 있습니다.

## Double Ratchet가 Signal 이후 세계에 미친 영향

Double Ratchet는 단순히 Signal의 기술이 아닙니다. 오늘날 사실상 표준이 된 보안 메시징 프로토콜의 기반입니다:

- **WhatsApp (2016)**: Signal Protocol(X3DH + Double Ratchet) 전면 도입. 20억 명 이상의 사용자.
- **Google Messages (2021)**: RCS 기반 E2EE에 Signal Protocol 적용.
- **Skype (2018)**: "비공개 대화" 기능에 Signal Protocol.
- **Facebook Messenger (2016)**: "비밀 대화" 기능. (2023년 기본 E2EE 전환)
- **Wire**: 독자적으로 유사한 Axolotl Ratchet 구현.
- **Matrix/Element**: Double Ratchet 기반의 Megolm 프로토콜.

2013년 Moxie Marlinspike가 TextSecure(Signal의 전신)에서 처음 구현한 이 알고리즘은 10년 만에 수십억 명의 일상적인 통신을 보호하는 기술이 되었습니다. 암호학과 실용 공학의 아름다운 결합, Double Ratchet는 현대 보안 메시징의 초석입니다.

## 참고 자료
- [Double Ratchet Algorithm Specification - Signal.org](https://signal.org/docs/specifications/doubleratchet/)
- [X3DH Key Agreement Protocol - Signal.org](https://signal.org/docs/specifications/x3dh/)
- [The Double Ratchet: Security Notions, Proofs, and Modularization (Alwen et al. 2019)](https://eprint.iacr.org/2016/1013.pdf)
- [libsignal (Signal의 공식 구현 라이브러리)](https://github.com/signalapp/libsignal)
