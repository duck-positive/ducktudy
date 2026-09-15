---
layout: post
title: "타원 곡선 암호화(ECC) 완전 정복: ECDH 키 교환과 ECDSA 서명의 수학적 원리"
date: 2026-09-15
categories: [cs, computer-science]
tags: [cryptography, ecc, ecdh, ecdsa, public-key, security, mathematics]
---

현대 인터넷 보안의 근간을 이루는 공개키 암호화는 오랫동안 RSA가 지배해 왔다. 그러나 오늘날 TLS 1.3, 비트코인, Signal 메신저 등 보안이 중요한 시스템들은 대부분 **타원 곡선 암호화(Elliptic Curve Cryptography, ECC)**를 채택하고 있다. ECC는 RSA보다 훨씬 짧은 키로 동등한 보안 강도를 제공하기 때문이다. 256비트 ECC 키는 3072비트 RSA 키와 맞먹는 보안성을 가진다. 이 글에서는 ECC의 수학적 토대부터 ECDH 키 교환, ECDSA 디지털 서명의 실제 구현까지 깊이 있게 탐구한다.

## 타원 곡선이란 무엇인가

암호학에서 사용하는 타원 곡선(Elliptic Curve)은 실수 위의 매끄러운 곡선이 아닌, **유한체(Finite Field) 위에서 정의된 대수 구조**다. 가장 흔히 사용되는 단-바이어슈트라스(short Weierstrass) 형태는 다음과 같다:

```
y² ≡ x³ + ax + b  (mod p)
```

여기서 `p`는 소수, `a`와 `b`는 곡선을 정의하는 상수다. 곡선이 특이점(cusp, node)을 갖지 않으려면 판별식 조건 `4a³ + 27b² ≢ 0 (mod p)`을 만족해야 한다.

타원 곡선 위의 점(point)들과 특수한 **무한원점(point at infinity, O)**의 집합은 **아벨 군(Abelian Group)**을 형성한다. 이 군의 연산인 **점 덧셈(Point Addition)**이 ECC 암호화의 핵심이다.

### 점 덧셈의 기하학적 의미

실수 위의 타원 곡선에서 두 점 P와 Q를 더하는 방법은 다음과 같다:
1. P와 Q를 잇는 직선이 곡선과 만나는 세 번째 점 R'을 구한다
2. R'을 x축에 대해 반전한 점 R = P + Q가 된다

P = Q인 경우(점 배가, Point Doubling)는 P에서의 접선을 이용한다. 무한체 위에서는 이 기하학적 직관이 모듈러 산술로 표현된다.

두 서로 다른 점 P(x₁, y₁)과 Q(x₂, y₂)에 대해:
```
λ = (y₂ - y₁) × (x₂ - x₁)⁻¹  (mod p)
x₃ = λ² - x₁ - x₂             (mod p)
y₃ = λ(x₁ - x₃) - y₁          (mod p)
```

P = Q인 점 배가의 경우:
```
λ = (3x₁² + a) × (2y₁)⁻¹  (mod p)
```

### 스칼라 곱셈과 ECDLP

ECC의 보안 기반은 **스칼라 곱셈(Scalar Multiplication)**이다. 점 G(생성점, Generator Point)를 정수 k번 더하는 연산을 `k·G = G + G + ... + G`로 표기한다. 이중 배가법(Double-and-Add)으로 O(log k) 시간에 계산할 수 있다.

하지만 역방향, 즉 Q = k·G일 때 Q와 G만 알고 k를 구하는 것은 현재의 계산 능력으로 불가능하다. 이것이 **타원 곡선 이산 로그 문제(ECDLP)**이며, ECC 보안의 근간이다.

## 왜 ECC인가

### RSA와의 보안 강도 비교

| 보안 강도 (비트) | RSA/DSA 키 크기 | ECC 키 크기 |
|:-:|:-:|:-:|
| 80 | 1024 | 160 |
| 112 | 2048 | 224 |
| 128 | 3072 | 256 |
| 192 | 7680 | 384 |
| 256 | 15360 | 521 |

더 짧은 키는 곧 더 빠른 연산, 더 작은 인증서, 더 적은 전력 소비를 의미한다. IoT 기기, 모바일 환경, TLS 핸드셰이크 성능이 중요한 상황에서 ECC는 필수적이다.

### 주요 표준 곡선

- **secp256k1**: 비트코인·이더리움이 사용, `p = 2²⁵⁶ - 2³² - 977`
- **P-256 (secp256r1)**: NIST 표준, TLS에서 가장 널리 사용
- **Curve25519**: D.J. Bernstein이 설계, Signal·WireGuard에서 사용, 단순성과 보안성으로 주목

## 코드 예제 1: 타원 곡선 점 연산 직접 구현 (Python)

```python
class EllipticCurve:
    """유한체 Fp 위의 단-바이어슈트라스 타원 곡선: y^2 = x^3 + ax + b (mod p)"""
    
    def __init__(self, a, b, p):
        self.a = a
        self.b = b
        self.p = p
        assert (4 * a**3 + 27 * b**2) % p != 0, "특이 곡선입니다"
    
    def is_on_curve(self, point):
        if point is None:  # 무한원점
            return True
        x, y = point
        return (y * y - x * x * x - self.a * x - self.b) % self.p == 0
    
    def point_add(self, P, Q):
        """두 점의 합 계산"""
        if P is None:
            return Q
        if Q is None:
            return P
        
        x1, y1 = P
        x2, y2 = Q
        
        if x1 == x2:
            if y1 != y2:  # P + (-P) = O
                return None
            # P == Q: 점 배가
            lam = (3 * x1 * x1 + self.a) * pow(2 * y1, -1, self.p) % self.p
        else:
            lam = (y2 - y1) * pow(x2 - x1, -1, self.p) % self.p
        
        x3 = (lam * lam - x1 - x2) % self.p
        y3 = (lam * (x1 - x3) - y1) % self.p
        return (x3, y3)
    
    def scalar_mult(self, k, P):
        """이중 배가법(Double-and-Add)으로 k*P 계산: O(log k)"""
        result = None  # 무한원점
        addend = P
        
        while k:
            if k & 1:  # 최하위 비트가 1이면 더하기
                result = self.point_add(result, addend)
            addend = self.point_add(addend, addend)  # 배가
            k >>= 1
        
        return result


# secp256k1 파라미터 (간소화된 작은 예시)
p = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEFFFFFC2F
a = 0
b = 7
Gx = 0x79BE667EF9DCBBAC55A06295CE870B07029BFCDB2DCE28D959F2815B16F81798
Gy = 0x483ADA7726A3C4655DA4FBFC0E1108A8FD17B448A68554199C47D08FFB10D4B8

curve = EllipticCurve(a, b, p)
G = (Gx, Gy)

# Alice의 개인키 생성 (랜덤 정수)
import secrets
private_key_alice = secrets.randbelow(p)

# 공개키 = 개인키 × 생성점
public_key_alice = curve.scalar_mult(private_key_alice, G)
print(f"Alice 공개키 x: {hex(public_key_alice[0])[:20]}...")

# 점이 곡선 위에 있는지 검증
assert curve.is_on_curve(public_key_alice), "곡선 위에 없습니다!"
print("공개키 검증 성공: 점이 secp256k1 곡선 위에 있습니다")
```

## 코드 예제 2: ECDH 키 교환과 ECDSA 서명 (Python cryptography 라이브러리)

```python
from cryptography.hazmat.primitives.asymmetric.ec import (
    ECDH, ECDSA, SECP256R1, generate_private_key
)
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.backends import default_backend
import hashlib

# ─── ECDH 키 교환 ───────────────────────────────────────────

def ecdh_key_exchange():
    """Alice와 Bob이 공유 비밀을 교환하는 ECDH 시연"""
    curve = SECP256R1()
    
    # 각자 개인키/공개키 쌍 생성
    alice_private = generate_private_key(curve, default_backend())
    bob_private   = generate_private_key(curve, default_backend())
    
    alice_public = alice_private.public_key()
    bob_public   = bob_private.public_key()
    
    # 공유 비밀 계산: Alice는 bob_public으로, Bob은 alice_public으로
    alice_shared = alice_private.exchange(ECDH(), bob_public)
    bob_shared   = bob_private.exchange(ECDH(), alice_public)
    
    # 두 공유 비밀은 동일해야 함
    assert alice_shared == bob_shared, "공유 비밀 불일치!"
    print(f"ECDH 공유 비밀 (hex): {alice_shared.hex()[:32]}...")
    print("Alice와 Bob의 공유 비밀이 일치합니다 ✓")
    
    return alice_shared  # 이 값을 AES-GCM 등 대칭키로 파생


# ─── ECDSA 서명 생성 및 검증 ────────────────────────────────

def ecdsa_sign_verify():
    """문서 서명과 검증 시연"""
    curve = SECP256R1()
    private_key = generate_private_key(curve, default_backend())
    public_key  = private_key.public_key()
    
    message = b"ECC is the future of cryptography."
    
    # 서명 생성 (ECDSA with SHA-256)
    signature = private_key.sign(message, ECDSA(hashes.SHA256()))
    print(f"서명 길이: {len(signature)} bytes (DER 인코딩)")
    
    # 서명 검증
    try:
        public_key.verify(signature, message, ECDSA(hashes.SHA256()))
        print("서명 검증 성공 ✓")
    except Exception as e:
        print(f"검증 실패: {e}")
    
    # 변조된 메시지로 검증 (실패해야 함)
    tampered = b"Tampered message."
    try:
        public_key.verify(signature, tampered, ECDSA(hashes.SHA256()))
    except Exception:
        print("변조 감지 성공 ✓ - 올바른 서명이 아닙니다")
    
    # 공개키 내보내기 (PEM 형식)
    pem = public_key.public_bytes(
        encoding=serialization.Encoding.PEM,
        format=serialization.PublicFormat.SubjectPublicKeyInfo
    )
    print(f"\n공개키 PEM:\n{pem.decode()}")


if __name__ == "__main__":
    shared_secret = ecdh_key_exchange()
    print()
    ecdsa_sign_verify()
```

## ECDSA 서명 알고리즘 내부 동작

ECDSA 서명은 다음 단계로 작동한다:

**서명 생성** (개인키 d, 생성점 G, 곡선 위수 n):
1. 임시 랜덤 정수 k (1 ≤ k ≤ n-1) 선택
2. R = k·G 계산, r = R.x mod n
3. s = k⁻¹ × (hash(m) + r·d) mod n
4. 서명 = (r, s)

**서명 검증** (공개키 Q = d·G):
1. u₁ = hash(m)·s⁻¹ mod n
2. u₂ = r·s⁻¹ mod n
3. X = u₁·G + u₂·Q
4. X.x mod n == r이면 유효

## 주의사항과 흔한 실수

### 1. k 재사용 취약점: Sony PS3 사건

ECDSA에서 **k는 절대로 재사용되어서는 안 된다**. Sony PS3 펌웨어 서명에서 k가 고정 상수로 사용된 것이 발견되어, 두 개의 서명만으로 개인키 전체가 복원되었다. 수학적으로 두 서명 (r, s₁)과 (r, s₂)에서 s₁ - s₂ = k⁻¹(h₁ - h₂)이므로 k를 계산할 수 있다.

해결책: RFC 6979의 **결정론적 k 생성** 사용 (개인키와 메시지를 HMAC-DRBG에 입력).

### 2. 안전하지 않은 곡선

모든 타원 곡선이 동등하게 안전하지 않다. **Anomalous 곡선** (위수가 p인 경우)이나 **Supersingular 곡선**은 MOV 공격에 취약하다. NIST P-256, P-384, Curve25519 같은 검증된 곡선만 사용해야 한다.

### 3. 점 검증 생략 위험

외부에서 받은 공개키가 실제로 올바른 곡선 위에 있는지 반드시 검증해야 한다. 검증을 생략하면 **소집합(small subgroup) 공격**에 노출된다.

```python
# 외부 공개키 검증 (항상 수행해야 함)
from cryptography.hazmat.primitives.asymmetric.ec import EllipticCurvePublicKey
from cryptography.exceptions import InvalidKey

def validate_public_key(key: EllipticCurvePublicKey) -> bool:
    try:
        # cryptography 라이브러리는 로드 시 자동 검증
        key.public_numbers()
        return True
    except (ValueError, InvalidKey):
        return False
```

### 4. 타이밍 공격

스칼라 곱셈 구현이 k의 비트 패턴에 따라 다른 시간을 소요하면 타이밍 공격이 가능하다. **Montgomery ladder** 또는 상수 시간(constant-time) 구현을 사용해야 한다.

## 정리

ECC는 더 작은 키, 더 빠른 연산, 더 낮은 전력 소비로 RSA를 대체하고 있다. TLS 1.3은 RSA 키 교환을 완전히 제거하고 ECDHE만 지원한다. 비트코인의 모든 지갑 주소는 secp256k1 위의 공개키에서 파생된다. Signal 프로토콜은 Curve25519 기반 X3DH로 E2E 암호화를 구현한다.

핵심은 ECDLP의 어려움이다: Q = k·G에서 k를 역산하는 것은 현재 기술로 불가능하다. 이 단방향성 위에 ECDH의 키 교환과 ECDSA의 디지털 서명이 구축된다.

## 참고 자료
- [Cloudflare: A Primer on Elliptic Curve Cryptography](https://blog.cloudflare.com/a-relatively-easy-to-understand-primer-on-elliptic-curve-cryptography/)
- [NIST FIPS 186-5: Digital Signature Standard](https://csrc.nist.gov/publications/detail/fips/186/5/final)
- [RFC 6979: Deterministic Usage of DSA and ECDSA](https://datatracker.ietf.org/doc/html/rfc6979)
- [SafeCurves: Choosing Safe Curves for Elliptic-Curve Cryptography](https://safecurves.cr.yp.to/)
