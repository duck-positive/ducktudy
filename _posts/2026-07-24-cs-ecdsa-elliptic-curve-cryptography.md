---
layout: post
title: "타원 곡선 암호화(ECC)와 ECDSA 완전 정복: Bitcoin·TLS·SSH가 선택한 현대 암호학의 핵심"
date: 2026-07-24
categories: [cs, computer-science]
tags: [cryptography, ecc, ecdsa, elliptic-curve, digital-signature, bitcoin, tls, public-key]
---

## 개념 설명

오늘날 인터넷 보안의 근간을 이루는 공개키 암호화에는 크게 두 가지 접근법이 있다. 하나는 1977년에 발표된 RSA, 다른 하나는 1985년에 Neal Koblitz와 Victor Miller가 독립적으로 제안한 **타원 곡선 암호화(Elliptic Curve Cryptography, ECC)**다. 현재 Bitcoin, TLS 1.3, SSH, Signal 프로토콜, FIDO2/WebAuthn이 모두 ECC를 기반으로 한다. 256비트 ECC 키는 3072비트 RSA 키와 동등한 보안 강도를 제공하면서도 훨씬 빠른 연산과 작은 키 크기를 가진다.

### 타원 곡선이란

타원 곡선(Elliptic Curve)은 다음 방정식을 만족하는 점들의 집합이다:

```
y² = x³ + ax + b (단, 4a³ + 27b² ≠ 0)
```

조건 `4a³ + 27b² ≠ 0`은 곡선이 자기 교차점(self-intersection)이나 뾰족한 점(cusp) 없이 매끄럽다는 것을 보장한다. 실수 위에서 그린 타원 곡선은 x축 대칭인 곡선 형태를 띤다.

암호학에서 중요한 것은 **유한체 위의 타원 곡선**이다. 소수 p에 대해 `y² ≡ x³ + ax + b (mod p)` 를 만족하는 정수 쌍 (x, y)의 집합이 타원 곡선 군(group)을 이룬다.

### 점 덧셈 연산

타원 곡선 암호학의 핵심은 **점 덧셈(point addition)** 연산이다. 두 점 P와 Q가 있을 때, P + Q는 기하학적으로 다음과 같이 정의된다:

1. P와 Q를 잇는 직선을 그린다
2. 이 직선이 곡선과 만나는 세 번째 점 R'을 찾는다
3. R'을 x축에 대해 대칭 이동한 점이 P + Q = R이다

이 연산의 대수적 공식은:

```
λ = (y₂ - y₁) / (x₂ - x₁) mod p  (P ≠ Q인 경우)
λ = (3x₁² + a) / (2y₁) mod p      (P = Q인 경우, 점 두 배)

x₃ = λ² - x₁ - x₂ mod p
y₃ = λ(x₁ - x₃) - y₁ mod p
```

### 스칼라 곱과 ECDLP

정수 k와 점 G에 대해 **스칼라 곱(scalar multiplication)** `k·G = G + G + ... + G (k번)`은 double-and-add 방법으로 효율적으로 계산할 수 있다. 그런데 반대로, 주어진 점 `P = k·G`에서 k를 구하는 것은 계산적으로 극도로 어렵다. 이것이 **타원 곡선 이산 로그 문제(ECDLP)**이며, ECC의 보안 기반이다.

---

## 왜 RSA 대신 ECC인가

RSA의 보안은 큰 수의 소인수분해의 어려움에 기반한다. ECDLP는 이보다 더 어려운 문제로 알려져 있어, 동일한 보안 수준을 훨씬 짧은 키로 달성할 수 있다.

| 보안 강도 (비트) | RSA 키 길이 | ECC 키 길이 |
|:---:|:---:|:---:|
| 80 | 1024 | 160 |
| 112 | 2048 | 224 |
| 128 | 3072 | 256 |
| 192 | 7680 | 384 |
| 256 | 15360 | 521 |

256비트 ECC가 3072비트 RSA와 동등하다는 것은 키 크기가 약 12배 작다는 의미다. 이는 TLS 핸드셰이크에서 전송할 데이터 양, 서명 생성/검증 시간, IoT 기기의 메모리 사용량 모두에 큰 영향을 미친다.

---

## 실제 구현 예제

### Python으로 ECC 점 연산 구현

```python
class EllipticCurve:
    """유한체 Fp 위의 Weierstrass 형식 타원 곡선: y² = x³ + ax + b mod p"""

    def __init__(self, a, b, p):
        self.a = a
        self.b = b
        self.p = p
        # 판별식 검사
        assert (4 * a**3 + 27 * b**2) % p != 0, "특이점(singular point)이 있는 곡선"

    def point_add(self, P, Q):
        """두 점 P, Q를 더한다. None은 무한원점(항등원)을 의미한다."""
        if P is None:
            return Q
        if Q is None:
            return P

        x1, y1 = P
        x2, y2 = Q

        # P + (-P) = 무한원점
        if x1 == x2 and (y1 + y2) % self.p == 0:
            return None

        if P == Q:
            # 점 두 배: 접선의 기울기 사용
            lam = (3 * x1 * x1 + self.a) * pow(2 * y1, -1, self.p) % self.p
        else:
            # 일반 덧셈: 두 점을 잇는 직선의 기울기
            lam = (y2 - y1) * pow(x2 - x1, -1, self.p) % self.p

        x3 = (lam * lam - x1 - x2) % self.p
        y3 = (lam * (x1 - x3) - y1) % self.p
        return (x3, y3)

    def scalar_mul(self, k, P):
        """스칼라 곱 k·P를 double-and-add로 계산 (O(log k))"""
        result = None  # 항등원(무한원점)
        addend = P
        while k:
            if k & 1:
                result = self.point_add(result, addend)
            addend = self.point_add(addend, addend)
            k >>= 1
        return result


# secp256k1 곡선 (Bitcoin이 사용하는 곡선)
# y² = x³ + 7 mod p
p = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEFFFFFC2F
a = 0
b = 7
n = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141

# 생성점 G
Gx = 0x79BE667EF9DCBBAC55A06295CE870B07029BFCDB2DCE28D959F2815B16F81798
Gy = 0x483ADA7726A3C4655DA4FBFC0E1108A8FD17B448A68554199C47D08FFB10D4B8
G = (Gx, Gy)

curve = EllipticCurve(a, b, p)

# 비밀키(private key): 임의의 정수 [1, n-1]
import secrets
private_key = secrets.randbelow(n - 1) + 1

# 공개키(public key) = 비밀키 × G
public_key = curve.scalar_mul(private_key, G)

print(f"비밀키: {private_key}")
print(f"공개키 x: {hex(public_key[0])}")
print(f"공개키 y: {hex(public_key[1])}")

# 검증: 공개키가 곡선 위에 있는지 확인
x, y = public_key
assert (y * y - x * x * x - b) % p == 0, "공개키가 곡선 위에 없음"
print("공개키 검증 성공: 곡선 위의 점임을 확인")
```

### ECDSA 서명과 검증

```python
import hashlib

def ecdsa_sign(message: bytes, private_key: int, curve, G, n) -> tuple:
    """ECDSA 서명 생성"""
    # 메시지 해시
    z = int.from_bytes(hashlib.sha256(message).digest(), 'big')

    while True:
        # 임시 비밀값 k 생성 (반드시 매번 새로운 값!)
        # 실제 구현에서는 RFC 6979 결정론적 방식 사용
        k = secrets.randbelow(n - 1) + 1

        # R = k·G, r = R.x mod n
        R = curve.scalar_mul(k, G)
        r = R[0] % n
        if r == 0:
            continue

        # s = k⁻¹(z + r·private_key) mod n
        k_inv = pow(k, -1, n)
        s = (k_inv * (z + r * private_key)) % n
        if s == 0:
            continue

        return (r, s)

def ecdsa_verify(message: bytes, signature: tuple, public_key, curve, G, n) -> bool:
    """ECDSA 서명 검증"""
    r, s = signature

    # 범위 검사
    if not (1 <= r < n and 1 <= s < n):
        return False

    z = int.from_bytes(hashlib.sha256(message).digest(), 'big')

    s_inv = pow(s, -1, n)
    u1 = (z * s_inv) % n
    u2 = (r * s_inv) % n

    # 검증 점: u1·G + u2·공개키
    point1 = curve.scalar_mul(u1, G)
    point2 = curve.scalar_mul(u2, public_key)
    verify_point = curve.point_add(point1, point2)

    if verify_point is None:
        return False

    return verify_point[0] % n == r

# 실제 서명/검증 테스트
message = b"Hello, Elliptic Curve!"
signature = ecdsa_sign(message, private_key, curve, G, n)
print(f"\n서명 r: {hex(signature[0])}")
print(f"서명 s: {hex(signature[1])}")

is_valid = ecdsa_verify(message, signature, public_key, curve, G, n)
print(f"서명 검증 결과: {'성공' if is_valid else '실패'}")

# 변조된 메시지로 검증
tampered = b"Hello, Elliptic Curve! (tampered)"
is_valid_tampered = ecdsa_verify(tampered, signature, public_key, curve, G, n)
print(f"변조된 메시지 검증 결과: {'성공' if is_valid_tampered else '실패 (올바른 동작)'}")
```

ECDSA 서명의 핵심은 임시 비밀값 k다. **k가 재사용되거나 예측 가능하면 비밀키가 노출된다.** 2010년 Sony PlayStation 3 해킹 사건은 k가 항상 같은 값으로 고정되어 있었기 때문에 발생했다. 현대 구현에서는 RFC 6979가 정의한 결정론적(deterministic) k 생성 방식을 사용한다.

---

## 주요 표준 곡선

실제 시스템에서 자주 사용되는 표준 곡선들이 있다:

**secp256k1** (Bitcoin, Ethereum): `y² = x³ + 7`, 소수체, 256비트. Satoshi Nakamoto가 선택한 곡선으로 NIST 표준에는 없지만 구조가 단순해 사이드채널 공격에 강하다.

**P-256 (secp256r1)** (TLS, FIDO2): NIST 표준 곡선. 무작위하게 선택된 것처럼 보이는 파라미터 때문에 일부에서 백도어 의혹을 제기했지만, 가장 광범위하게 지원되는 곡선이다.

**Curve25519** (Signal, WireGuard, SSH Ed25519): Daniel J. Bernstein이 설계. 몽고메리 형식(`y² = x³ + 486662x² + x`)을 사용하며, 상수 시간 구현이 용이해 사이드채널 공격에 강하다. Ed25519는 이 곡선의 Edwards 형식 변환이다.

---

## 주의사항과 실전 팁

**절대 직접 구현하지 말 것**: 위 코드는 교육 목적이다. 실제 애플리케이션에서는 반드시 검증된 라이브러리(Python의 `cryptography`, Java의 `Bouncy Castle`, Go의 `crypto/elliptic`)를 사용해야 한다. 타이밍 공격, 포인트 유효성 검사, 모듈러 역원 등에서 미묘한 실수가 치명적 취약점이 된다.

**k 재사용 금지**: ECDSA에서 동일한 k로 두 개의 다른 메시지에 서명하면 두 서명에서 비밀키를 수식으로 복구할 수 있다. 반드시 매번 암호학적으로 안전한 난수를 사용하거나 RFC 6979 결정론적 방식을 써야 한다.

**EdDSA vs ECDSA**: Ed25519(Edwards-curve Digital Signature Algorithm)는 ECDSA의 단점을 개선한 서명 방식이다. k가 결정론적으로 생성되고, 포인트 검증이 더 엄격하며, 상수 시간 구현이 더 쉽다. 새 시스템을 설계한다면 Ed25519를 권장한다.

**점 유효성 검사**: 외부로부터 받은 점이 실제 곡선 위에 있는지 반드시 검증해야 한다. 악의적인 점이 소집단(subgroup)에 속해 있으면 "소집단 공격(small subgroup attack)"으로 비밀키가 노출될 수 있다.

## 참고 자료

- [Elliptic-curve cryptography - Wikipedia](https://en.wikipedia.org/wiki/Elliptic-curve_cryptography)
- [Elliptic Curve Cryptography: a gentle introduction - Andrea Corbellini](https://andrea.corbellini.name/2015/05/17/elliptic-curve-cryptography-a-gentle-introduction/)
- [Elliptic Curve Cryptography: ECDH and ECDSA - Andrea Corbellini](https://andrea.corbellini.name/2015/05/30/elliptic-curve-cryptography-ecdh-and-ecdsa/)
- [RFC 6979: Deterministic Usage of the DSA and ECDSA](https://www.rfc-editor.org/rfc/rfc6979)
