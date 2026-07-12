---
layout: post
title: "공개키 암호화와 RSA·Diffie-Hellman 완전 정복: 인터넷 보안의 수학적 토대"
date: 2026-07-12
categories: [cs, computer-science]
tags: [cryptography, RSA, Diffie-Hellman, public-key, asymmetric-encryption, modular-arithmetic, PKI]
---

## 왜 공개키 암호화가 필요한가?

1970년대 이전까지 모든 암호화는 **대칭키(Symmetric Key)** 방식이었습니다. 송신자와 수신자가 같은 키를 공유하고, 그 키로 암호화와 복호화를 모두 수행합니다. AES, DES가 대표적입니다.

하지만 대칭키는 치명적인 약점을 갖고 있습니다. **키 배포 문제(Key Distribution Problem)**입니다. "첫 번째 만남"에서 어떻게 안전하게 키를 나눌 것인가? 도청자가 있는 인터넷 채널을 통해 키를 전달하면, 도청자도 키를 알게 됩니다.

공개키 암호화(Public Key Cryptography, 비대칭 암호화)는 1976년 Whitfield Diffie와 Martin Hellman이 제안하면서 이 문제를 혁신적으로 해결했습니다. 핵심 아이디어는 이렇습니다.

> **"암호화에 사용하는 키와 복호화에 사용하는 키를 수학적으로 연결하되, 암호화 키(공개키)를 알아도 복호화 키(개인키)를 계산하는 것이 현실적으로 불가능하게 만들 수 있다."**

---

## 수학적 기초: 단방향 함수와 트랩도어

공개키 암호화의 핵심은 **단방향 함수(One-Way Function)**와 **트랩도어(Trapdoor)**입니다.

- **단방향 함수**: 계산은 쉽지만 역산은 어려운 함수
  - 예: `f(x) = x²`은 쉽지만 `x = √f(x)`는 음수 처리 등이 필요
- **트랩도어**: 추가 비밀 정보(개인키)가 있을 때만 역산이 쉬운 단방향 함수

RSA는 **정수 인수분해(Integer Factorization)** 문제의 어려움을 이용합니다.
- 쉬운 방향: 두 소수 p, q를 곱해 n = p × q 계산 (밀리초)
- 어려운 방향: n만 주어졌을 때 p, q를 찾기 (n이 2048비트면 우주의 나이보다 오래 걸림)

Diffie-Hellman은 **이산 로그 문제(Discrete Logarithm Problem)**를 이용합니다.
- 쉬운 방향: g^x mod p 계산 (빠른 거듭제곱)
- 어려운 방향: g^x mod p = y에서 x 찾기 (지수적 시간)

---

## RSA 알고리즘 완전 분석

### 키 생성 과정

RSA 키 생성은 수학적으로 매우 구체적인 절차를 따릅니다.

```
1. 두 큰 소수 p, q 선택 (실제: 각 1024비트 이상)
2. n = p × q 계산 (모듈러스, 공개)
3. φ(n) = (p-1)(q-1) 계산 (오일러 totient, 비밀)
4. e 선택: 1 < e < φ(n) 이고 gcd(e, φ(n)) = 1 (보통 e = 65537 사용)
5. d = e^(-1) mod φ(n) 계산 (모듈러 역원)
   → e × d ≡ 1 (mod φ(n))

공개키: (n, e)
개인키: (n, d) [p, q, φ(n)은 폐기]
```

### 암호화와 복호화

```
암호화: C = M^e mod n
복호화: M = C^d mod n
```

페르마의 소정리에 의해 M^(e×d) ≡ M (mod n)이 성립합니다.

---

## 실제 구현 예제

### 예제 1: RSA 알고리즘 처음부터 구현 (Python)

```python
import random
import math
from typing import Tuple

def is_prime(n: int, k: int = 10) -> bool:
    """Miller-Rabin 확률적 소수 판별 (k번 반복, 오류 확률 4^-k)"""
    if n < 2:
        return False
    if n == 2 or n == 3:
        return True
    if n % 2 == 0:
        return False

    # n-1 = 2^r * d 형태로 분해
    r, d = 0, n - 1
    while d % 2 == 0:
        r += 1
        d //= 2

    for _ in range(k):
        a = random.randrange(2, n - 1)
        x = pow(a, d, n)  # pow(base, exp, mod) — 파이썬 내장 빠른 거듭제곱

        if x == 1 or x == n - 1:
            continue
        for _ in range(r - 1):
            x = pow(x, 2, n)
            if x == n - 1:
                break
        else:
            return False  # 합성수
    return True

def generate_prime(bits: int) -> int:
    """지정 비트 크기의 소수 생성"""
    while True:
        # 최상위 비트와 최하위 비트를 1로 고정 (홀수 + 지정 크기 보장)
        p = random.getrandbits(bits) | (1 << bits - 1) | 1
        if is_prime(p):
            return p

def extended_gcd(a: int, b: int) -> Tuple[int, int, int]:
    """확장 유클리드 알고리즘: ax + by = gcd(a,b)의 해 반환"""
    if b == 0:
        return a, 1, 0
    g, x, y = extended_gcd(b, a % b)
    return g, y, x - (a // b) * y

def mod_inverse(e: int, phi: int) -> int:
    """e의 phi에 대한 모듈러 역원 계산"""
    g, x, _ = extended_gcd(e, phi)
    if g != 1:
        raise ValueError("모듈러 역원이 존재하지 않음 (gcd != 1)")
    return x % phi

def generate_rsa_keypair(bits: int = 512) -> Tuple[Tuple[int, int], Tuple[int, int]]:
    """RSA 키 쌍 생성 (교육용 — 실제는 최소 2048비트)"""
    print(f"소수 p 생성 중... ({bits}비트)")
    p = generate_prime(bits // 2)

    print(f"소수 q 생성 중... ({bits}비트)")
    while True:
        q = generate_prime(bits // 2)
        if q != p:
            break

    n = p * q
    phi_n = (p - 1) * (q - 1)

    # 공개 지수: 65537 = 2^16 + 1 (페르마 소수, 이진 표현에서 1이 2개뿐 → 빠른 연산)
    e = 65537
    assert math.gcd(e, phi_n) == 1, "e와 phi_n이 서로소가 아님"

    d = mod_inverse(e, phi_n)

    public_key = (n, e)
    private_key = (n, d)
    return public_key, private_key

def rsa_encrypt(message: int, public_key: Tuple[int, int]) -> int:
    n, e = public_key
    if message >= n:
        raise ValueError("메시지가 모듈러스보다 큽니다")
    return pow(message, e, n)  # 빠른 모듈러 거듭제곱 O(log e)

def rsa_decrypt(ciphertext: int, private_key: Tuple[int, int]) -> int:
    n, d = private_key
    return pow(ciphertext, d, n)

# 실행 예시
print("=== RSA 키 생성 ===")
pub_key, priv_key = generate_rsa_keypair(bits=512)
n, e = pub_key
_, d = priv_key
print(f"공개키 e: {e}")
print(f"모듈러스 n: {n}")
print(f"개인키 d: {d} (비밀!)")

# 정수 메시지 암호화/복호화
message = 42
print(f"\n원본 메시지: {message}")

encrypted = rsa_encrypt(message, pub_key)
print(f"암호화된 값: {encrypted}")

decrypted = rsa_decrypt(encrypted, priv_key)
print(f"복호화된 값: {decrypted}")
assert decrypted == message, "복호화 실패!"
print("복호화 성공!")

# 문자열 메시지 처리 (실제로는 OAEP 패딩 사용)
def encrypt_string(text: str, pub_key: Tuple[int, int]) -> list[int]:
    return [rsa_encrypt(ord(c), pub_key) for c in text]

def decrypt_string(ciphertext: list[int], priv_key: Tuple[int, int]) -> str:
    return ''.join(chr(rsa_decrypt(c, priv_key)) for c in ciphertext)

original = "Hello RSA"
enc = encrypt_string(original, pub_key)
dec = decrypt_string(enc, priv_key)
print(f"\n원본: '{original}'")
print(f"복호화: '{dec}'")
```

---

### 예제 2: Diffie-Hellman 키 교환 프로토콜 구현

```python
import secrets
import hashlib

class DHParty:
    """
    Diffie-Hellman 키 교환 참여자
    실제로는 ECDH(타원 곡선 DH)를 사용하지만, 원리 이해를 위해 고전 DH 구현
    """
    # RFC 3526 — 2048비트 MODP 그룹 소수 (교육용으로는 작은 값 사용)
    # 실제 운영 환경에서는 아래 작은 p를 절대 사용하지 마세요
    PRIME_P = (
        0xFFFFFFFFFFFFFFFFC90FDAA22168C234C4C6628B80DC1CD1
        + 0x29024E088A67CC74020BBEA63B139B22514A08798E3404DD
        # (실제 2048비트 소수는 훨씬 길지만 여기서는 생략)
        # 교육용으로 더 작은 안전 소수 사용
    )
    # 교육용 소수: 실제 사용 금지
    SAFE_PRIME = 23  # 실제: 2048비트 이상

    def __init__(self, name: str, p: int = None, g: int = 2):
        self.name = name
        self.p = p or self.SAFE_PRIME
        self.g = g
        # 개인 키: 비밀 난수 [2, p-2]
        self._private_key = secrets.randbelow(self.p - 2) + 2
        # 공개 키: g^private_key mod p
        self.public_key = pow(self.g, self._private_key, self.p)

    def compute_shared_secret(self, other_public_key: int) -> int:
        """
        상대방의 공개키로 공유 비밀 계산
        Alice: shared = Bob_pub^alice_priv mod p = (g^bob_priv)^alice_priv mod p
        Bob:   shared = Alice_pub^bob_priv mod p = (g^alice_priv)^bob_priv mod p
        두 값이 동일: g^(alice_priv * bob_priv) mod p
        """
        shared = pow(other_public_key, self._private_key, self.p)
        return shared

    def derive_aes_key(self, shared_secret: int) -> bytes:
        """공유 비밀에서 실제 사용할 대칭키 유도 (KDF: Key Derivation Function)"""
        secret_bytes = shared_secret.to_bytes(
            (shared_secret.bit_length() + 7) // 8, byteorder='big'
        )
        return hashlib.sha256(secret_bytes).digest()  # 256비트 AES 키


def simulate_dh_exchange():
    """
    Alice와 Bob이 도청자 Eve가 있는 채널에서 공유 비밀을 확립하는 시뮬레이션
    
    공개 정보 (Eve도 알 수 있음): p, g, Alice의 공개키, Bob의 공개키
    비밀 정보 (Eve는 모름): Alice의 개인키, Bob의 개인키, 공유 비밀
    """
    # 공개 파라미터 (안전 소수 p, 생성원 g)
    # 교육용: 실제로는 RFC 3526의 큰 소수 사용
    p = 0xFFFFFFFFFFFFFFFFC90FDAA22168C234  # 128비트 교육용 (실제: 2048비트)
    p = 9999991  # 더 단순한 교육용 소수 (7자리)
    g = 2

    print("=== Diffie-Hellman 키 교환 시뮬레이션 ===")
    print(f"공개 파라미터: p={p}, g={g}")
    print()

    # Alice와 Bob이 각자 개인키 생성
    alice = DHParty("Alice", p=p, g=g)
    bob = DHParty("Bob", p=p, g=g)

    print(f"[Alice] 개인키 (비밀): {alice._private_key}")
    print(f"[Alice] 공개키 (공개): {alice.public_key}")
    print()
    print(f"[Bob]   개인키 (비밀): {bob._private_key}")
    print(f"[Bob]   공개키 (공개): {bob.public_key}")
    print()

    # 공개키 교환 (Eve가 이것을 가로채도 공유 비밀을 계산할 수 없음)
    print("-- 공개키 교환 (네트워크 전송, Eve가 볼 수 있음) --")
    print(f"Alice → Bob: 공개키 {alice.public_key} 전송")
    print(f"Bob → Alice: 공개키 {bob.public_key} 전송")
    print()

    # 각자 공유 비밀 계산
    alice_shared = alice.compute_shared_secret(bob.public_key)
    bob_shared = bob.compute_shared_secret(alice.public_key)

    print(f"[Alice] 계산한 공유 비밀: {alice_shared}")
    print(f"[Bob]   계산한 공유 비밀: {bob_shared}")
    assert alice_shared == bob_shared, "공유 비밀이 다름!"
    print("✓ 공유 비밀 일치!")
    print()

    # Eve의 공격 시뮬레이션: 공개키만 알고 있는 상태에서 개인키 계산 시도
    print("-- Eve의 이산 로그 공격 시뮬레이션 --")
    print(f"Eve가 아는 정보: p={p}, g={g}, Alice 공개키={alice.public_key}")
    print("Eve가 Alice의 개인키를 찾으려면 g^x ≡ alice_pub (mod p)를 풀어야 함")

    # Baby-step Giant-step 알고리즘 (O(√p) 시간/공간)
    import math
    def baby_step_giant_step(g: int, h: int, p: int) -> int:
        """이산 로그 계산: g^x ≡ h (mod p) 에서 x 찾기 — O(√p)"""
        n = int(math.ceil(math.sqrt(p)))

        # Baby steps: 테이블 {g^j mod p: j}
        baby_steps = {}
        for j in range(n):
            baby_steps[pow(g, j, p)] = j

        # Giant step factor: g^(-n) mod p
        factor = pow(pow(g, n, p), p - 2, p)  # 페르마의 소정리로 역원 계산

        # Giant steps: h * factor^i mod p = g^(i*n - ?) mod p
        gamma = h
        for i in range(n):
            if gamma in baby_steps:
                x = i * n + baby_steps[gamma]
                if pow(g, x, p) == h:
                    return x
            gamma = gamma * factor % p
        return -1

    import time
    start = time.perf_counter()
    found_key = baby_step_giant_step(g, alice.public_key, p)
    elapsed = time.perf_counter() - start

    if found_key == alice._private_key:
        print(f"Eve가 개인키 발견: {found_key} (소요: {elapsed:.4f}초)")
        print(f"→ p={p}가 너무 작아서 공격 성공. 실제 p는 2048비트 이상이어야 함")
    else:
        print("Eve 공격 실패")

    # AES 키 유도
    aes_key = alice.derive_aes_key(alice_shared)
    print(f"\n[Alice/Bob] 공유된 AES-256 키: {aes_key.hex()}")

simulate_dh_exchange()
```

---

## RSA vs Diffie-Hellman: 언제 무엇을 쓰는가?

| 항목 | RSA | Diffie-Hellman |
|------|-----|----------------|
| 목적 | 암호화 + 디지털 서명 | 키 교환 전용 |
| 보안 기반 | 정수 인수분해 어려움 | 이산 로그 어려움 |
| 순방향 비밀성 | ❌ (세션키가 개인키에 종속) | ✅ (각 세션마다 새 키 생성 가능) |
| 연산 속도 | 느림 (특히 복호화) | 빠름 |
| 실용 쓰임새 | TLS 인증서, 서명 | TLS 키 교환 (ECDHE) |
| 현대 권장 방식 | RSA-OAEP, RSA-PSS | ECDH (타원 곡선 기반) |

### 현대 TLS 1.3에서의 실제 적용

```
TLS 1.3 핸드셰이크 (단순화):
1. Client Hello: 지원 ECDH 그룹 + 공개키 전송
2. Server Hello: ECDH 공개키 + 인증서(RSA/ECDSA) + 서명 전송
3. ECDHE로 세션키 유도 → 이후 AES-GCM으로 대칭 암호화
4. RSA는 서버 신원 인증에만 사용 (순방향 비밀성은 ECDHE가 담당)
```

---

## 주의사항 및 실전 팁

### 주의 1: 직접 구현 금지 — 검증된 라이브러리 사용

```python
# ❌ 직접 구현 (패딩 오류, 타이밍 공격 등 취약점 위험)
cipher = rsa_encrypt(message, pub_key)

# ✅ cryptography 라이브러리 사용
from cryptography.hazmat.primitives.asymmetric import rsa, padding
from cryptography.hazmat.primitives import hashes

private_key = rsa.generate_private_key(public_exponent=65537, key_size=2048)
public_key = private_key.public_key()

# OAEP 패딩으로 안전하게 암호화 (교과서 RSA는 결정론적 — 공격에 취약)
ciphertext = public_key.encrypt(
    b"secret message",
    padding.OAEP(
        mgf=padding.MGF1(algorithm=hashes.SHA256()),
        algorithm=hashes.SHA256(),
        label=None
    )
)
```

### 주의 2: 소수 크기와 권장 키 길이 (2026년 기준)

| RSA 키 길이 | 보안 수준 | 권장 만료 |
|-------------|-----------|-----------|
| 1024비트 | ~80비트 | 이미 안전하지 않음 |
| 2048비트 | ~112비트 | 2030년까지 |
| 3072비트 | ~128비트 | 2031년 이후 |
| 4096비트 | ~140비트 | 장기 권장 |

### 팁: ECDSA vs RSA 서명

```python
from cryptography.hazmat.primitives.asymmetric import ec

# EC P-256 키 쌍 (RSA 3072비트와 동등한 128비트 보안, 훨씬 빠름)
ec_private_key = ec.generate_private_key(ec.SECP256R1())
ec_public_key = ec_private_key.public_key()

signature = ec_private_key.sign(b"data to sign", ec.ECDSA(hashes.SHA256()))
ec_public_key.verify(signature, b"data to sign", ec.ECDSA(hashes.SHA256()))
print("ECDSA 서명 검증 성공")
```

### 주의 3: 양자 컴퓨터와 포스트 양자 암호화

Shor's Algorithm은 양자 컴퓨터에서 RSA와 DH를 다항 시간에 깰 수 있습니다. 2024년 NIST는 양자 내성 암호화 표준을 확정했습니다.

- **ML-KEM** (구 CRYSTALS-Kyber): 키 캡슐화 (KEM)
- **ML-DSA** (구 CRYSTALS-Dilithium): 디지털 서명
- **SLH-DSA** (구 SPHINCS+): 해시 기반 서명

현재 OpenSSL 3.2+와 Google Chrome은 X25519Kyber768(하이브리드 ECDH + ML-KEM)을 지원합니다.

---

## 마무리

RSA와 Diffie-Hellman은 현대 인터넷 보안의 수학적 토대입니다. HTTPS, SSH, VPN, 디지털 서명 — 이 모든 것이 이 두 알고리즘 위에 세워져 있습니다. 하지만 실제 구현은 반드시 검증된 암호화 라이브러리를 사용하고, 키 길이 권장 사항을 준수하며, 포스트 양자 암호화로의 전환도 미리 대비해야 합니다.

## 참고 자료

- [RSA cryptosystem - Wikipedia](https://en.wikipedia.org/wiki/RSA_cryptosystem)
- [Diffie-Hellman key exchange - Wikipedia](https://en.wikipedia.org/wiki/Diffie%E2%80%93Hellman_key_exchange)
- [The RSA Cryptosystem Concepts — Practical Cryptography for Developers](https://cryptobook.nakov.com/asymmetric-key-ciphers/the-rsa-cryptosystem-concepts)
- [Diffie-Hellman Key Exchange — Practical Cryptography for Developers](https://cryptobook.nakov.com/key-exchange/diffie-hellman-key-exchange)
