---
layout: post
title: "패스워드 해싱의 과학: Argon2·bcrypt·scrypt 메모리 하드 함수 완전 정복"
date: 2026-10-23
categories: [cs, computer-science]
tags: [cryptography, password-hashing, argon2, bcrypt, scrypt, security, memory-hard-function]
---

## 개요

"패스워드를 MD5로 해시하면 안전하다"는 말이 통하던 시절이 있었습니다. 오늘날 초당 수십억 회의 해시를 계산하는 GPU·ASIC 앞에서 MD5는 물론이고 SHA-256조차도 패스워드 보호에는 부적합합니다. 이 글에서는 **메모리 하드(memory-hard) 함수**의 개념을 중심으로 bcrypt, scrypt, Argon2가 어떻게 공격자의 병렬 연산 능력을 무력화하는지 깊이 있게 살펴봅니다.

---

## 1. 왜 일반 암호학적 해시 함수로는 부족한가

암호학적 해시 함수(SHA-256, SHA-3, BLAKE3)는 원래 **파일 무결성 검증**이나 **디지털 서명**을 위해 설계되었습니다. 설계 목표가 `가능한 빠를 것`이므로, 패스워드 저장에 쓰면 공격자에게 큰 혜택을 줍니다.

### 오프라인 공격의 위협

1. 공격자가 해시 DB를 탈취합니다.
2. 각 해시에 대해 후보 패스워드 목록(사전, 레인보우 테이블)을 순차·병렬로 시도합니다.
3. GPU 하나로 SHA-256 단순 해시는 초당 **수십억 번** 시도 가능합니다.

레인보우 테이블 공격을 막기 위해 **솔트(salt)**를 도입하면 사전공격 재사용은 막을 수 있지만, 온라인 브루트포스의 속도 자체는 줄어들지 않습니다.

해결책은 해싱 자체를 **의도적으로 느리고 자원을 많이 쓰게** 만드는 것입니다. 이것이 bcrypt, scrypt, Argon2의 공통 목표입니다.

---

## 2. bcrypt: 시간 비용 기반의 선구자 (1999)

bcrypt는 1999년 Niels Provos와 David Mazières가 설계한 패스워드 해싱 함수로, Blowfish 블록 암호의 키 스케줄을 기반으로 합니다.

### 핵심 메커니즘: Eksblowfish

bcrypt는 내부적으로 `EksBlowfishSetup` 함수를 호출하며, 비용 파라미터(cost factor) `c`에 따라 키 스케줄을 **2^c 번** 반복합니다. `c=12`이면 4,096번 반복이고, `c=14`이면 16,384번 반복입니다. 하드웨어가 빨라지면 `c`를 높여 대응합니다.

```
bcrypt(password, salt, cost):
    state = EksBlowfishSetup(cost, salt, password)
    ciphertext = "OrpheanBeholderScryDoubt"   # 고정 64-bit × 3 블록
    for i in range(64):
        ciphertext = EncryptECB(state, ciphertext)
    return cost || salt || ciphertext
```

#### 한계

- **최대 72바이트 입력 제한**: 72바이트를 초과하는 패스워드는 잘립니다.
- **메모리 사용량 고정(~4 KB)**: GPU나 FPGA가 많은 병렬 인스턴스를 동시에 실행하는 것을 막지 못합니다. bcrypt는 *시간 하드(time-hard)*이지만 *메모리 하드(memory-hard)*는 아닙니다.
- **병렬화 가능**: 각 해시 계산이 독립적이므로 GPU 수천 코어가 동시에 돌릴 수 있습니다.

---

## 3. scrypt: 메모리 하드의 등장 (2009)

Colin Percival이 2009년 발표한 scrypt는 패스워드 해싱이 **메모리를 많이 쓰도록** 강제하여, 메모리가 비싼 하드웨어(특히 ASIC)의 병렬 연산을 억제합니다.

### 핵심: ROMix 알고리즘

scrypt의 핵심인 `ROMix`는 다음 단계를 거칩니다:

1. **Fill 단계**: `N`개의 블록을 순차적으로 생성해 메모리에 저장합니다. 각 블록은 이전 블록에 의존합니다.
2. **Mix 단계**: `N`번의 반복을 수행하면서, 매 반복마다 **무작위 위치**의 저장된 블록을 읽습니다.

무작위 접근 때문에 Fill 단계에서 생성한 `N`개 블록을 전부 메모리에 유지해야 합니다. 메모리를 절약하면 재계산 비용이 기하급수적으로 증가합니다(time-memory tradeoff).

```
scrypt(password, salt, N, r, p):
    # PBKDF2-SHA256으로 초기 블록 생성
    B = PBKDF2(password, salt, 1, p * 128 * r)

    # 각 블록에 ROMix 적용
    for i in range(p):
        B[i] = ROMix(B[i], N)

    # 최종 출력
    return PBKDF2(password, B, 1, dkLen)
```

파라미터:
- `N`: CPU/메모리 비용 (보통 2^14 ~ 2^20)
- `r`: 블록 크기 파라미터 (보통 8)
- `p`: 병렬화 파라미터

#### scrypt의 한계

- `N`, `r`, `p` 파라미터 간 상호작용이 복잡하여 튜닝이 어렵습니다.
- 데이터 의존적 메모리 접근 패턴이 **캐시 타이밍 사이드 채널**을 유발할 수 있습니다.
- 병렬 인스턴스(멀티스레드)를 허용하면서도 각 인스턴스의 메모리 사용을 분리하기 어렵습니다.

---

## 4. Argon2: 현대적 표준 (2015, RFC 9106)

Argon2는 2015년 Password Hashing Competition(PHC)의 우승작으로, 2021년 IETF RFC 9106으로 표준화되었습니다. Alex Biryukov, Daniel Dinu, Dmitry Khovratovich가 설계했습니다.

### 세 가지 변형

| 변형 | 설계 목적 |
|------|----------|
| **Argon2d** | GPU 공격 방어 최적화 (데이터 의존적 접근) |
| **Argon2i** | 사이드 채널 공격 방어 (데이터 독립적 접근) |
| **Argon2id** | 권장값: 첫 절반은 Argon2i, 나머지는 Argon2d |

RFC 9106은 일반 패스워드 해싱에는 **Argon2id**를 권장합니다.

### 파라미터 구성

```
Argon2id(password, salt, m, t, p, hashLen):
    # m: 메모리 크기 (KiB 단위, 권장 최소 19MiB = 19456)
    # t: 반복 횟수 (권장 최소 1)
    # p: 병렬도 (권장 1~4)
    # hashLen: 출력 길이 (보통 32)
```

RFC 9106 권장 파라미터 (2021 기준):
- **최소**: `m=19456`, `t=2`, `p=1`
- **고보안**: `m=65536`, `t=3`, `p=4`

### 내부 구조

Argon2는 메모리를 `p × q` 격자로 구성합니다. 각 레인(lane)은 `q`개의 1 KiB 블록으로 이루어지며, 블록 채우기는 다음 규칙을 따릅니다:

1. 초기 두 블록은 H(password, salt, ...) 함수로 채웁니다.
2. 이후 각 블록은 이전 블록과 참조 블록을 **블레이크2b 기반 압축 함수 G**로 혼합합니다.
3. Argon2id는 1패스에서 데이터 독립적, 2패스 이후 데이터 의존적 참조를 씁니다.

---

## 5. Python 구현 예제

### 예제 1: Argon2id로 패스워드 해시 및 검증

```python
from argon2 import PasswordHasher
from argon2.exceptions import VerifyMismatchError, VerificationError

# RFC 9106 권장 파라미터
ph = PasswordHasher(
    time_cost=2,       # t: 반복 횟수
    memory_cost=19456, # m: 메모리 사용량 (KiB)
    parallelism=1,     # p: 병렬도
    hash_len=32,       # 해시 출력 길이 (bytes)
    salt_len=16,       # 솔트 길이 (bytes)
)

# 해시 생성
password = "super_secret_password_123"
hashed = ph.hash(password)
print(f"해시: {hashed[:60]}...")
# $argon2id$v=19$m=19456,t=2,p=1$<salt>$<hash>

# 검증 성공
try:
    ph.verify(hashed, password)
    print("패스워드 일치!")
except VerifyMismatchError:
    print("패스워드 불일치")

# 파라미터 업그레이드 필요 여부 확인
if ph.check_needs_rehash(hashed):
    new_hash = ph.hash(password)
    print("해시 재생성 필요 (파라미터 업그레이드)")
```

### 예제 2: bcrypt vs Argon2 성능 비교

```python
import time
import bcrypt
from argon2 import PasswordHasher

password = b"benchmark_password"

# bcrypt (cost=12, ~250ms)
start = time.perf_counter()
salt = bcrypt.gensalt(rounds=12)
bcrypt_hash = bcrypt.hashpw(password, salt)
bcrypt_time = time.perf_counter() - start

# Argon2id (RFC 9106 최소 권장)
ph = PasswordHasher(time_cost=2, memory_cost=19456, parallelism=1)
start = time.perf_counter()
argon2_hash = ph.hash(password.decode())
argon2_time = time.perf_counter() - start

print(f"bcrypt (cost=12):  {bcrypt_time*1000:.1f}ms, 출력길이={len(bcrypt_hash)}bytes")
print(f"Argon2id (m=19MiB, t=2): {argon2_time*1000:.1f}ms, 출력길이={len(argon2_hash)}bytes")

# GPU 공격 비용 추정 (단순화)
gpu_sha256_per_sec = 10_000_000_000  # 10G hashes/sec
gpu_bcrypt_per_sec = gpu_sha256_per_sec / (2**12)  # bcrypt cost=12
print(f"\nGPU 추정 bcrypt 시도/초: {gpu_bcrypt_per_sec:,.0f}")
print(f"8자 영숫자 패스워드(62^8≈218조) 크랙 예상: {218e12/gpu_bcrypt_per_sec/3600:.0f}시간")
```

---

## 6. 주의사항 및 실전 팁

### 솔트는 반드시 암호학적으로 안전하게 생성

솔트는 해시마다 고유해야 하며, `os.urandom()` 또는 `secrets.token_bytes()`를 사용해야 합니다. 예측 가능한 솔트는 공격자가 사전을 재활용할 수 있게 합니다.

### 파라미터를 해시에 함께 저장

Argon2의 PHC 문자열 형식(`$argon2id$v=19$m=19456,t=2,p=1$...`)은 파라미터를 해시와 함께 인코딩합니다. 이를 통해 서버 업그레이드 시 이전 해시도 검증할 수 있습니다.

### 하드웨어가 빨라지면 파라미터를 높여라

OWASP 권장사항(2023):
- **Argon2id**: `m=19456`, `t=2`, `p=1` 최소; 가능하면 `m=65536`, `t=3`
- **bcrypt**: 최소 cost=10, 가능하면 12 이상
- **scrypt**: `N=32768`, `r=8`, `p=1` 최소

로그인 시 검증 성공 후 `check_needs_rehash()`를 호출해 이전 파라미터로 저장된 해시를 자동으로 업그레이드하세요.

### 72바이트 bcrypt 한계 우회

bcrypt 도입 시 긴 패스워드를 SHA-256으로 먼저 해시한 후 bcrypt에 입력하는 방식이 쓰이기도 합니다. 그러나 이는 해시의 해시로 서명/검증 체계를 복잡하게 만들 수 있으므로, 새 시스템에서는 Argon2id를 권장합니다.

### 패스워드 해시는 절대 비교 연산에 `==` 쓰지 말 것

타이밍 공격을 막으려면 `hmac.compare_digest(a, b)` 또는 라이브러리 내장 검증 함수를 사용해야 합니다.

---

## 참고 자료

- [RFC 9106 — Argon2 Memory-Hard Function for Password Hashing and Proof-of-Work Applications](https://www.rfc-editor.org/info/rfc9106)
- [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)
- [Password Hashing Competition — Argon2 Specification](https://password-hashing.net/argon2-specs.pdf)
- [RFC 7914 — The scrypt Password-Based Key Derivation Function](https://www.rfc-editor.org/info/rfc7914)
