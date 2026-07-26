---
layout: post
title: "AES 대칭키 암호화 내부 구조 완전 정복: SubBytes·ShiftRows·MixColumns의 수학"
date: 2026-07-26
categories: [cs, computer-science]
tags: [aes, cryptography, encryption, security, symmetric-key, galois-field, rijndael]
---

TLS 핸드셰이크에서 세션 키가 교환되는 순간, 이후 모든 데이터는 AES로 암호화된다. HTTPS로 보내는 신용카드 번호, WhatsApp 메시지, 파일 저장소의 데이터 — 현대 디지털 보안의 핵심은 AES(Advanced Encryption Standard)다. 이전 글에서 비대칭 암호화인 RSA와 ECC를 살펴봤다면, 이번에는 실제 대용량 데이터 암호화에 사용되는 AES의 내부 구조를 수학적으로 깊이 파고든다.

## AES란 무엇인가?

AES는 미국 NIST(국가표준기술연구소)가 2001년에 표준화한 대칭키 블록 암호다. 벨기에 암호학자 Joan Daemen과 Vincent Rijmen이 설계한 **Rijndael** 알고리즘이 AES로 채택되었다.

### 핵심 파라미터

| 파라미터 | AES-128 | AES-192 | AES-256 |
|---------|---------|---------|---------|
| 키 길이 | 128 비트 | 192 비트 | 256 비트 |
| 블록 크기 | 128 비트 | 128 비트 | 128 비트 |
| 라운드 수 | 10 | 12 | 14 |
| 보안 수준 | 128 비트 | 192 비트 | 256 비트 |

AES는 항상 128비트(16바이트) 블록 단위로 동작한다. 키는 128/192/256비트 중 선택한다. 더 긴 키는 더 많은 라운드를 수행한다.

### 왜 AES가 필요한가?

AES 이전의 표준인 DES(Data Encryption Standard)는 56비트 키를 사용했는데, 1999년에 단 22시간 만에 무차별 대입으로 깨졌다. 3DES는 DES를 세 번 적용해 보안을 높였지만 속도가 느렸다. AES는 이 두 문제를 동시에 해결했다. 충분한 키 길이와 하드웨어 가속(AES-NI 명령어셋)으로 현재까지도 가장 널리 쓰이는 암호 알고리즘이다.

## AES의 수학적 토대: 갈루아 체(GF(2⁸))

AES의 연산은 일반 산술이 아니라 **유한체(Finite Field)** 위에서 이루어진다. 구체적으로는 **GF(2⁸)**, 즉 2⁸ = 256개의 원소를 가진 갈루아 체다.

GF(2⁸)에서의 덧셈은 **XOR** 연산이고, 곱셈은 기약 다항식 `x⁸ + x⁴ + x³ + x + 1`(16진수로 `0x11b`)로 나눈 나머지다. 이 수학적 구조가 AES의 보안과 구현 효율성 모두를 보장한다.

```python
def gf_multiply(a: int, b: int, mod: int = 0x11b) -> int:
    """
    GF(2^8)에서의 곱셈 구현
    AES에서 사용하는 기약 다항식: x^8 + x^4 + x^3 + x + 1 (0x11b)
    """
    result = 0
    while b > 0:
        if b & 1:  # b의 최하위 비트가 1이면
            result ^= a  # XOR (GF에서의 덧셈)
        a <<= 1  # a를 x 만큼 곱함
        if a & 0x100:  # a가 8비트를 넘으면
            a ^= mod   # 기약 다항식으로 모듈러
        b >>= 1
    return result & 0xFF


# 검증
print(hex(gf_multiply(0x57, 0x83)))  # 0xc1 (AES 표준 예제)
print(hex(gf_multiply(0x02, 0x87)))  # xtime(0x87)
```

## AES의 상태(State)

AES는 16바이트 입력 블록을 **4×4 바이트 행렬(State)**로 취급한다. 이 행렬에 여러 변환을 순서대로 적용하는 것이 AES 암호화다.

```
입력: [b0, b1, b2, ..., b15]
State:
  b0  b4  b8  b12
  b1  b5  b9  b13
  b2  b6  b10 b14
  b3  b7  b11 b15
```

데이터는 열(column) 우선으로 채워진다는 점에 주의하자.

## 라운드 함수: 4가지 변환

AES의 각 라운드는 4가지 변환으로 구성된다. 마지막 라운드에서는 MixColumns가 생략된다.

### 1. SubBytes — 비선형 치환

16×16 크기의 S-박스(Substitution Box)를 이용해 각 바이트를 다른 바이트로 치환한다. S-박스는 **GF(2⁸)에서의 역원** 계산과 **GF(2)에서의 아핀 변환** 두 단계로 구성된다.

```python
# AES S-박스 (표준값)
SBOX = [
    0x63, 0x7c, 0x77, 0x7b, 0xf2, 0x6b, 0x6f, 0xc5,
    0x30, 0x01, 0x67, 0x2b, 0xfe, 0xd7, 0xab, 0x76,
    # ... (256개 값)
]

def sub_bytes(state: list) -> list:
    """SubBytes: 각 바이트를 S-박스로 치환"""
    return [[SBOX[b] for b in row] for row in state]
```

S-박스 설계의 핵심은 **선형 공격(linear cryptanalysis)**과 **차분 공격(differential cryptanalysis)**에 대한 저항성이다. 역원 연산은 최적의 비선형성을 제공한다.

### 2. ShiftRows — 행 순환

State의 각 행을 좌측으로 순환 이동시킨다:
- 0행: 이동 없음
- 1행: 1바이트 좌측 이동
- 2행: 2바이트 좌측 이동
- 3행: 3바이트 좌측 이동

```python
def shift_rows(state: list) -> list:
    """ShiftRows: 각 행을 좌측으로 순환 이동"""
    result = [row[:] for row in state]
    for i in range(4):
        result[i] = state[i][i:] + state[i][:i]
    return result
```

ShiftRows는 단독으로는 별로 복잡하지 않지만, MixColumns와 결합하면 강력한 **확산(diffusion)** 효과를 만든다. 한 열의 바이트들이 4개의 서로 다른 열로 분산된다.

### 3. MixColumns — 열 혼합

AES에서 가장 수학적으로 복잡한 단계다. 각 열(4바이트)을 GF(2⁸) 위의 4×4 행렬로 곱셈한다:

```
[2 3 1 1]   [s0]   [s'0]
[1 2 3 1] × [s1] = [s'1]
[1 1 2 3]   [s2]   [s'2]
[3 1 1 2]   [s3]   [s'3]
```

여기서 모든 연산은 GF(2⁸)에서 이루어진다. 이 행렬의 계수가 {1, 2, 3}으로 제한된 것은 하드웨어 구현을 위해서다. GF에서 2를 곱하는 것은 좌측 비트 쉬프트 + 조건부 XOR이므로 매우 빠르다.

```python
def xtime(a: int) -> int:
    """GF(2^8)에서 2를 곱하기 (좌측 쉬프트 + 조건부 XOR)"""
    result = (a << 1) & 0xFF
    if a & 0x80:  # MSB가 1이면 기약 다항식의 하위 바이트 XOR
        result ^= 0x1B
    return result

def mix_column(col: list) -> list:
    """MixColumns: 한 열에 대한 행렬 곱셈"""
    s0, s1, s2, s3 = col
    
    # GF에서: 2*s0 ^ 3*s1 ^ s2 ^ s3
    # 3*x = 2*x ^ x (GF에서 3 = 2+1 = 0b11)
    result = [
        xtime(s0) ^ (xtime(s1) ^ s1) ^ s2 ^ s3,
        s0 ^ xtime(s1) ^ (xtime(s2) ^ s2) ^ s3,
        s0 ^ s1 ^ xtime(s2) ^ (xtime(s3) ^ s3),
        (xtime(s0) ^ s0) ^ s1 ^ s2 ^ xtime(s3),
    ]
    return result

def mix_columns(state: list) -> list:
    """State의 모든 열에 MixColumns 적용"""
    # state를 열 단위로 재구성
    cols = [[state[r][c] for r in range(4)] for c in range(4)]
    mixed = [mix_column(col) for col in cols]
    # 다시 행 단위로 변환
    return [[mixed[c][r] for c in range(4)] for r in range(4)]
```

### 4. AddRoundKey — 라운드 키 XOR

각 라운드에서 파생된 서브 키를 State와 XOR한다. 이 단계가 없으면 AES는 암호화가 아니라 그저 복잡한 데이터 셔플링에 불과하다.

```python
def add_round_key(state: list, round_key: list) -> list:
    """AddRoundKey: State와 라운드 키를 XOR"""
    return [[state[r][c] ^ round_key[r][c] 
             for c in range(4)] 
            for r in range(4)]
```

## 키 스케줄(Key Schedule)

AES-128은 10라운드를 수행하므로 원래 키(128비트) 외에 10개의 라운드 키가 필요하다. 키 확장(Key Expansion)은 원래 키에서 총 11개의 128비트 라운드 키를 생성한다.

키 확장에는 `SubWord`(4바이트 워드에 S-박스 적용), `RotWord`(4바이트 순환 이동), `Rcon`(라운드 상수, GF에서 2의 거듭제곱) 등이 사용된다. Rcon을 사용하는 이유는 관련 키 공격(related-key attack)을 방지하기 위해서다.

## AES 운영 모드(Modes of Operation)

AES는 블록 암호이므로 여러 블록을 처리하는 방법을 별도로 정의해야 한다.

```python
import os

def aes_ecb_encrypt_demo(blocks: list, round_keys: list) -> list:
    """ECB 모드: 각 블록을 독립적으로 암호화 (보안 취약)"""
    # ECB는 동일한 평문 블록이 동일한 암호문 블록이 됨
    return [aes_encrypt_block(block, round_keys) for block in blocks]

def aes_cbc_encrypt_demo(blocks: list, round_keys: list, iv: bytes) -> list:
    """CBC 모드: 이전 암호문 블록과 XOR 후 암호화"""
    result = []
    prev = list(iv)  # 초기화 벡터
    
    for block in blocks:
        # 이전 암호문과 XOR
        xored = [block[i] ^ prev[i] for i in range(16)]
        encrypted = aes_encrypt_block(xored, round_keys)
        result.append(encrypted)
        prev = encrypted  # 다음 라운드의 이전 블록으로 사용
    
    return result
```

| 모드 | 특징 | 사용 사례 | 주의사항 |
|------|------|---------|---------|
| ECB | 블록 독립 암호화 | 절대 사용 금지 | 패턴 노출 |
| CBC | 이전 블록 체이닝 | 파일 암호화 | IV 재사용 금지 |
| CTR | 카운터 스트림 | 스트리밍 | 카운터 재사용 금지 |
| GCM | CTR + 인증 | TLS 1.3, HTTPS | 현재 표준 |

ECB 모드의 보안 취약점은 유명하다. 같은 평문 블록이 항상 같은 암호문 블록을 생성하기 때문에, 비트맵 이미지를 ECB로 암호화하면 원본의 윤곽이 그대로 드러난다. 오늘날 권장되는 모드는 **AES-256-GCM**이다.

## 완전한 AES-128 암호화 구현

```python
# 표준 AES S-박스
SBOX = bytes([
    0x63,0x7c,0x77,0x7b,0xf2,0x6b,0x6f,0xc5,0x30,0x01,0x67,0x2b,0xfe,0xd7,0xab,0x76,
    0xca,0x82,0xc9,0x7d,0xfa,0x59,0x47,0xf0,0xad,0xd4,0xa2,0xaf,0x9c,0xa4,0x72,0xc0,
    0xb7,0xfd,0x93,0x26,0x36,0x3f,0xf7,0xcc,0x34,0xa5,0xe5,0xf1,0x71,0xd8,0x31,0x15,
    0x04,0xc7,0x23,0xc3,0x18,0x96,0x05,0x9a,0x07,0x12,0x80,0xe2,0xeb,0x27,0xb2,0x75,
    0x09,0x83,0x2c,0x1a,0x1b,0x6e,0x5a,0xa0,0x52,0x3b,0xd6,0xb3,0x29,0xe3,0x2f,0x84,
    0x53,0xd1,0x00,0xed,0x20,0xfc,0xb1,0x5b,0x6a,0xcb,0xbe,0x39,0x4a,0x4c,0x58,0xcf,
    0xd0,0xef,0xaa,0xfb,0x43,0x4d,0x33,0x85,0x45,0xf9,0x02,0x7f,0x50,0x3c,0x9f,0xa8,
    0x51,0xa3,0x40,0x8f,0x92,0x9d,0x38,0xf5,0xbc,0xb6,0xda,0x21,0x10,0xff,0xf3,0xd2,
    0xcd,0x0c,0x13,0xec,0x5f,0x97,0x44,0x17,0xc4,0xa7,0x7e,0x3d,0x64,0x5d,0x19,0x73,
    0x60,0x81,0x4f,0xdc,0x22,0x2a,0x90,0x88,0x46,0xee,0xb8,0x14,0xde,0x5e,0x0b,0xdb,
    0xe0,0x32,0x3a,0x0a,0x49,0x06,0x24,0x5c,0xc2,0xd3,0xac,0x62,0x91,0x95,0xe4,0x79,
    0xe7,0xc8,0x37,0x6d,0x8d,0xd5,0x4e,0xa9,0x6c,0x56,0xf4,0xea,0x65,0x7a,0xae,0x08,
    0xba,0x78,0x25,0x2e,0x1c,0xa6,0xb4,0xc6,0xe8,0xdd,0x74,0x1f,0x4b,0xbd,0x8b,0x8a,
    0x70,0x3e,0xb5,0x66,0x48,0x03,0xf6,0x0e,0x61,0x35,0x57,0xb9,0x86,0xc1,0x1d,0x9e,
    0xe1,0xf8,0x98,0x11,0x69,0xd9,0x8e,0x94,0x9b,0x1e,0x87,0xe9,0xce,0x55,0x28,0xdf,
    0x8c,0xa1,0x89,0x0d,0xbf,0xe6,0x42,0x68,0x41,0x99,0x2d,0x0f,0xb0,0x54,0xbb,0x16
])

RCON = [0x01,0x02,0x04,0x08,0x10,0x20,0x40,0x80,0x1b,0x36]

def key_expansion(key: bytes) -> list:
    """AES-128 키 확장 → 11개의 라운드 키 (각 16바이트)"""
    assert len(key) == 16
    w = list(key)
    
    for i in range(4, 44):
        temp = w[(i-1)*4 : i*4]
        if i % 4 == 0:
            # RotWord + SubWord + Rcon
            temp = temp[1:] + temp[:1]  # RotWord
            temp = [SBOX[b] for b in temp]  # SubWord
            temp[0] ^= RCON[i//4 - 1]
        for j in range(4):
            w.append(w[(i-4)*4 + j] ^ temp[j])
    
    # 11개의 4×4 라운드 키로 변환
    round_keys = []
    for r in range(11):
        rk = [[w[r*16 + c*4 + row] for c in range(4)] for row in range(4)]
        round_keys.append(rk)
    return round_keys

# Python의 내장 AES 사용 예시 (실제 운영 코드용)
from cryptography.hazmat.primitives.ciphers.aead import AESGCM

def aes_gcm_encrypt(plaintext: bytes, key: bytes) -> tuple:
    """AES-256-GCM 암호화 (실제 사용 권장)"""
    assert len(key) == 32, "AES-256은 32바이트 키 필요"
    nonce = os.urandom(12)  # GCM은 12바이트 nonce 권장
    aesgcm = AESGCM(key)
    ciphertext = aesgcm.encrypt(nonce, plaintext, None)
    return nonce, ciphertext

def aes_gcm_decrypt(nonce: bytes, ciphertext: bytes, key: bytes) -> bytes:
    """AES-256-GCM 복호화 (인증 태그 자동 검증)"""
    aesgcm = AESGCM(key)
    return aesgcm.decrypt(nonce, ciphertext, None)

# 테스트
key = os.urandom(32)
message = b"Hello, AES-GCM! This is a secret."
nonce, encrypted = aes_gcm_encrypt(message, key)
decrypted = aes_gcm_decrypt(nonce, encrypted, key)
print(f"원본: {message}")
print(f"암호화: {encrypted.hex()[:32]}...")
print(f"복호화: {decrypted}")
print(f"일치: {message == decrypted}")
```

## 주의사항과 보안 팁

### 1. 절대 직접 구현하지 말 것

위의 코드는 교육 목적이다. 실제 서비스에서는 `cryptography` (Python), `javax.crypto` (Java), OpenSSL 등 검증된 라이브러리를 사용해야 한다. 직접 구현한 암호화 코드는 타이밍 공격(timing attack) 등 미묘한 취약점이 생기기 쉽다.

### 2. Nonce/IV 재사용 금지

GCM 모드에서 같은 키로 같은 nonce를 두 번 사용하면 **재앙적으로 보안이 붕괴**된다. 공격자는 두 암호문을 XOR해 평문과 인증 키를 복구할 수 있다. 항상 `os.urandom(12)`로 무작위 nonce를 생성하라.

### 3. AES-NI 하드웨어 가속

현대 CPU(Intel Westmere 이후, AMD Bulldozer 이후)는 AES 연산을 전용 명령어(AESENC, AESDEC 등)로 가속한다. 검증된 라이브러리는 자동으로 이를 활용하므로, 소프트웨어 구현 대비 수십 배 빠르다. Golang의 `crypto/aes`, Java의 `javax.crypto`, Python의 `cryptography` 패키지 모두 AES-NI를 사용한다.

### 4. 키 관리가 핵심

암호화 알고리즘 자체보다 **키 관리**가 더 어렵다. 하드코딩된 키, 소스코드에 커밋된 키, 환경 변수로만 관리되는 키 모두 위험하다. AWS KMS, HashiCorp Vault, GCP Cloud KMS 같은 전용 키 관리 서비스를 사용하라.

## 마치며

AES의 아름다움은 단순한 연산의 조합이 수학적으로 증명 가능한 강력한 보안을 제공한다는 점이다. SubBytes의 비선형성, ShiftRows와 MixColumns의 확산, AddRoundKey의 혼동이 결합하여 현재 알려진 어떤 방법으로도 2^126.1 미만의 연산으로는 AES-128을 깰 수 없다(관련 키 공격에서도). 양자 컴퓨터 시대에도 AES-256은 Grover 알고리즘에 대응하여 128비트의 보안 강도를 유지한다.

## 참고 자료
- [NIST FIPS 197 — Advanced Encryption Standard](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.197-upd1.pdf)
- [Advanced Encryption Standard — Wikipedia](https://en.wikipedia.org/wiki/Advanced_Encryption_Standard)
- [Rijndael MixColumns — Wikipedia](https://en.wikipedia.org/wiki/Rijndael_MixColumns)
- [The Design of Rijndael — Daemen & Rijmen](https://link.springer.com/book/10.1007/978-3-662-60769-5)
